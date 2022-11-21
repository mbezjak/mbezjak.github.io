---
title: "Integration Tests for SQL Statement and Lock Timeout"
date: 2022-11-21T11:20:57+01:00
draft: false
---

If you're using (and [have configured](../configuring-sql-data-source)) SQL
statement and lock timeout, it would be good to verify how the service behaves
when those timeouts happen.

Integration tests can help us verify the behavior. However, the problem with
integration tests is that they make most sense in the context of your project.
Without the supporting code they are devoid of information that makes them tick.
Let's try it out anyway. Here are the assumptions for the tests below:

- A service executing the tests is a
  [Ring](https://github.com/ring-clojure/ring) based service.
- An integration test will execute (integrate) all the code below the top-level
  ring handler (usually called `app`). This is the function given to `run-jetty`.
- `call-app` (below) will simply run the `app` with the given mock request.
- `db` (below) is an SQL data source using a test DB.
- It's hard to provoke statement and lock timeouts in normal handlers. So we'll
  cheat and replace the real handler with our own code that provokes them in a
  clear manner.

## Statement timeout

This is relatively straight forward.

```clojure
(deftest db-statement-timeout
  (let [f (fn [_]
            (jdbc/with-transaction [tx db]
              (jdbc/execute! tx ["SET LOCAL statement_timeout = '10ms'"])
              (jdbc/execute! tx ["SELECT pg_sleep(0.1)"])))]
    (with-redefs [user-handlers/create f]
      (let [response (call-app (mock/request :post "/api/v1/users"))]
        ;; assert response has the appropriate error message
        ))))
```

## Lock timeout

Lock timeouts usually happen when two concurrent requests try to modify the same
DB resource at the same time. That's what we want to simulate. This means
coordinating threads in a deterministic manner.

```clojure
(deftest db-lock-timeout
  (jdbc/execute! db ["INSERT INTO user (email) VALUES (?)" "john@example.com"])
  (let [f (fn [_]
            (let [row-locked (CyclicBarrier. 2)
                  test-done (CountDownLatch. 1)]
              (future
                ;; Simulating another HTTP thread locked the same row
                (jdbc/with-transaction [tx db]
                  (jdbc/execute! tx ["UPDATE user SET email = ?" "john@company-1.com"])
                  (.await row-locked)
                  (.await test-done)))
              (.await row-locked)
              (try
                (jdbc/with-transaction [tx db]
                  (jdbc/execute! tx ["SET LOCAL lock_timeout = '10ms'"])
                  ;; Fails because another transaction locked this row
                  (jdbc/execute! tx ["UPDATE user SET email = ?" "john@company-2.com"]))
                (finally
                  (.countDown test-done)))))]
    (with-redefs [user-handlers/create f]
      (let [response (call-app (mock/request :post "/api/v1/users"))]
        ;; assert response has the appropriate error message
        )))
  ;; cleanup after the test, perhaps in try/finally. e.g. deleting the user
  )
```
