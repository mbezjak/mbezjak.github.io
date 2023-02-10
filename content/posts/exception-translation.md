---
title: "Exception Translation"
date: 2023-02-10T12:01:31+01:00
draft: false
---

Can a typical "Internal Server Error" be improved a bit? Let's find out.

In fact, let's consider more than just a
[response](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/500) that is
typically associated with *internal server errors*. Those types of errors
usually originate in the outermost `try/catch` expression. There we might find
code to:
- respond with HTTP status code 500
- increment a metric that an unknown exception occurred
- log the event [^1]
- save the exception to the database [^2]
- etc.

We'll try to improve everything in the `catch` block. Just by a little bit.

It might be a bit easier if we have an example of what we'll target for
improvement. Here it is.

```clojure
(defn wrap-exception [handler]
  (fn [request]
    (try
      (handler request)
      (catch Exception ex
        (increment-metric-unknown-exception)
        (log-unknown-exception ex request)
        (save-to-db ex request)
        (respond-with-http-500)))))
```

The code above is an example of a [ring
middleware](https://github.com/ring-clojure/ring/wiki/Concepts#middleware). It
doesn't have to be ring, or even HTTP service specific. Any outermost
`try/catch` expression is relevant for this post:
- when using `java.util.concurrent.ExecutorService`,
- or spawning new threads in general,
- or using `java.lang.Thread.UncaughtExceptionHandler`,
- or a `try/catch` in `-main` for command line applications,
- etc.

## What is there to improve?

The outermost `try/catch` expression is all about handling an **unknown**
exception. That is completely fine. After all, it's not possible to anticipate
and/or handle every unforeseen situation.

However, the way the `catch` block is usually implemented does have some
drawbacks. At the same time, it's both eliminating too much information and not
accumulating enough information.
- Responding with HTTP 500 is usually collapsing a rich source of information
  (stack trace, exception message, any other data the exception is carrying)
  into a single error message "Something went wrong". That's perhaps too much
  reduction of information. There is a lot of middle ground between responding
  with a complete stack trace and responding with a generic error message.
- On the other hand, simply dumping all that information as an unknown exception
  log event, saving it to DB, or incrementing the same metric is not ideal. That
  groups too much into a single category making it harder to deal with.
  Extracting a bit more information about the exception might help with
  categorization.

The above is still a bit too abstract. Let's see how those problems might
manifest in the real world: losing users' confidence and/or wasting time. What
happens when *internal server errors* happen a bit too often?

How would you feel if you saw "Internal Server Error" too frequently in an app
you need to use? Apart from "something didn't work", the error doesn't tell you
much. You'd probably be a bit annoyed. Then you'd start questioning if the last
thing you did was saved successfully or not. The same thing can happen to any
user of your service: a paying customer, an employee (for internal apps), a
member of the QA team, or other developers if they are consuming your service.
They all can become disheartened because they don't have the necessary
information to conclude something about the error. Is it a real bug or simply
something transient?

If you're lucky, someone might report a generic error they saw while using the
app. That issue needs to be groomed, planned, and eventually, someone in the
development team needs to spend time debugging it. If the end result is "That
was just a DB lock timeout. They should have tried again." then a lot of people
wasted a bunch of time for no good reason. In fact, to "save time", someone
might try to shoot down similar issues early on (e.g. in grooming) without doing
the investigation to confirm their assumption. Are you sure that's not a real
bug that was just ignored?

Even if no new issues enter the Backlog, there is a similar waste happening when
viewing the logs for recent unknown exceptions. Are you mentally filtering out
the "I've seen those exceptions already multiple times" from the list of all
logged unknown exceptions? Or maybe you don't feel like looking at the list
anymore because it's mostly stuff you've seen before.

This is what we want to avoid. We want the metrics, logs, and errors that the
users see to be a bit more relevant so that when the actual bugs happen,
everyone sees them clearly.

## Exception examples

Let's see a couple of exceptions that might end up being caught in the outermost
`try/catch` expression. If you have access to the production logs, you could
also extract the exception from the last 7 days and consider them as well.

1. `java.lang.NullPointerException`
2. `java.lang.IllegalArgumentException`
3. `java.lang.ClassCastException: class java.lang.Long cannot be cast to class clojure.lang.IFn`
4. `java.net.SocketException: Network is unreachable`
5. `java.net.SocketTimeoutException: Connect timed out`
6. `java.net.SocketTimeoutException: Read timed out`
7. `java.net.ConnectException: Connection refused`
8. `java.net.UnknownHostException: external-service.com: Name or service not known`
9. `java.io.FileNotFoundException: /external-service/file-to-import.json (No such file or directory)`
10. `java.io.FileNotFoundException: /external-service/file-to-import.json (Permission denied)`
11. `java.io.IOException: Attempted read on closed stream.`
12. `java.io.IOException: Read error`
13. `org.postgresql.util.PSQLException: ERROR: canceling statement due to statement timeout`
14. `org.postgresql.util.PSQLException: ERROR: canceling statement due to lock timeout
     Where: while updating tuple (0,1) in relation "users"`
15. `org.postgresql.util.PSQLException: ERROR: deadlock detected
     Detail: Process 186 waits for ShareLock on transaction 1140; blocked by process 196.
     Process 196 waits for ShareLock on transaction 1139; blocked by process 186.
     Hint: See server log for query details.
     Where: while updating tuple (0,1) in relation "users"`
16. `org.postgresql.util.PSQLException: ERROR: new row for relation "users" violates check constraint "users_email_check"
     Detail: Failing row contains (...).`
17. `org.postgresql.util.PSQLException: ERROR: duplicate key value violates unique constraint "users_pkey"
     Detail: Key (id)=(...) already exists.`
18. `org.postgresql.util.PSQLException: FATAL: sorry, too many clients already`
19. `org.postgresql.util.PSQLException: Ran out of memory retrieving query results.`
20. etc.

Are those exceptions all the same? Should they all be handled as unknown
exceptions? Let's reason through a couple of examples and ask ourselves two
questions:
1. How unexpected is this exception?
2. How frequently do we expect it to occur?

Answering the questions above:
- Exceptions 1. - 3. and 11. are usually completely unexpected. They represent a
  bug in the system. Perhaps a missing [validation](../what-is-a-validation) if
  not an actual defect. They should not occur frequently. Best if they don't
  occur at all.
- Exceptions 4. - 8. might happen if the network becomes unreliable for some
  reason. Perhaps someone updated or is currently updating the cloud
  infrastructure? How often does someone break the development (or other)
  environment in your team?
- Exceptions 9. and 10.: in the old days, files were shared via FTP / network
  drive by one service writing the files, and another service reading them.
  Perhaps that is still the case in your project? Perhaps someone (from a
  different company) updated the user the external service is running under?
  This is an indication that another company changed something that broke your
  service and didn't notify your team of that change. How often does that
  happen?
- Exception 12.: this might happen when the other side went away for some
  reason. Backend usually has solid networking so you shouldn't expect too many
  of those. On the other hand, this might indicate that a mobile client
  connecting to the Backend abruptly disconnected. Mobile client disconnecting
  can be much more frequent. Especially if mobile clients are on a cellular
  network.
- Exceptions 13. - 15. are to be expected from time to time and usually become
  more frequent with increasing load on the RDS. The good news is that they are
  usually transient.
- Try to reason through the rest of the exceptions on your own. What else are
  you experiencing frequently in your project?

It does not look like all of the exceptions above should be grouped in the same
category: an unknown exception occurred. Yes, those are all unrecoverable
exceptions (from the point of view of the outermost `try/catch` expression).
However, that does **not** mean that nothing can be done about reporting the
error.

## How to improve?

To improve the situation we can derive a bit more information about the
exception so we can act more meaningfully. It will no longer be an unknown
exception. In other words, we translate an exception (and the data it might be
carrying) into something more useful. E.g. an error `code` and an error
`message` intended for a human.

Assuming you're using the [error model](../error-model), here is a seed
implementation to get you started.

`exception_translation.clj`

```clojure
(ns my.company.exception-translation
  (:require
   [my.company.errors :as errors])
  (:import
   (java.sql SQLException)))

(defn translate
  "Tries to translate `ex` into something a human might understand.

  Returns `errors` if it can or `nil` otherwise."
  [ex]
  (cond
    (instance? SQLException ex)
    (condp = (.getSQLState ^SQLException ex)
      ;; https://www.postgresql.org/docs/11/errcodes-appendix.html
      "55P03" (errors/make-1 {:code :db/lock-timeout})
      "57014" (errors/make-1 {:code :db/statement-timeout})
      nil)

    ,,,
    ;; Write more conditions when needed

    :else nil))

(defn translatable? [ex]
  (some? (translate ex)))
```

Given that we're referring to the error message via `:code`, we also need the
[error templates](../error-model#code-and-args).

`resources/error-templates.edn`

```clojure
:db/lock-timeout "Another database transaction is in progress. Please try again later."
:db/statement-timeout "Database query took too long. This might happen if the system is overloaded. Please try again later."
```

Finally, connecting it all in an updated exception middleware. See [another
post](../using-the-error-model#what-to-do-with-the-errors) to make sense of the
`errors` namespace used here.

```clojure
(defn wrap-exception [handler]
  (fn [request]
    (try
      (handler request)
      (catch Exception ex
        (cond
          ,,,
          ;; Do you have project specific exceptions above?

          (exception-translation/translatable? ex)
          (let [errors (->> ex
                            (exception-translation/translate)
                            (errors/with-message (get-message-tpls)))]
            (increment-metric-translated-exception errors)
            (log-translated-exception errors request)
            (save-translated-exception-to-db errors request)
            (respond-to-translated-exception errors))

          :else
          (do
            (increment-metric-unknown-exception)
            (log-unknown-exception ex request)
            (save-to-db ex request)
            (respond-with-http-500)))))))
```

That's all it takes. The user of the service should no longer see "Internal
Server Error" when a database lock timeout occurs, but rather a more meaningful
error message. We can implement logging, metrics, and saving to DB differently
too. That means that those exceptions will no longer crowd the list of real
unknown exceptions.

Note that the error messages above should be good enough for other developers,
QA, maybe other employees, or service-to-service communication. The messages are
probably not good enough for your customers. They won't know what "database
transaction" or "database query" means. If the UI uses the response to show an
error to the customer then you might want to change the response message
accordingly.

`exception-translation` namespace should be highly specific to your project.
Don't just copy/paste what you see above. Use it as an inspiration. If you're
not using the [error model](../error-model) then rewrite the code to something
that makes sense in your project. Take a look at the exceptions that happen
regularly in production and add a line or two to handle them specially.

The goal is not to handle every exception you see in the logs! Certainly not
something like `NullPointerException` or `ClassCastException`. Just the ones
that you see frequently and the ones that make sense to your project. A
successful application of this principle could be to start with 1-2 exceptions
that you see in the logs and expand to a few more cases over time (perhaps
months). I have seen from experience that I don't need to handle a lot of cases,
but the ones I do have helped a lot. They especially help with trimming down the
noise in the logs.

When it comes to implementing `exception-translation` namespace any information
you can extract is fair game:
- exception type
- regex matching against the exception message
- anything that the exception carries: SQL error code, `ex-data`
- stack frame
- nested exceptions

That might look very frail, given the JVM differences, OS differences, etc.
However, it works quite well in practice. Once written, I didn't find I needed
to update that code constantly to keep it up to date. Quite the contrary.

If you want to avoid some of the particularly nasty regexes or looking up the
stack frames then a simple trick of wrapping an exception with additional data
works quite well. E.g. somewhere in the low-level code.

```clojure
(try
  ...
  (catch Exception ex
    (throw (ex-info "Failed to call service A"
                    {:calling-service-a? true
                     :request request}
                    ex))))
```

Of course, if you come to depend on this feature then it pays to write tests for
those special cases. You don't want unit tests here. You want to see how the
whole system reacts to a particular exception. So integration or functional
tests would be much more useful. For example, take a look at the [tests for sql
statement and lock
timeout](../integration-tests-for-sql-statement-and-lock-timeout).

[^1]: I'll assume there is a central place for the logs. It doesn't matter where
    that is: ElasticSearch, Datadog, Sentry, CloudWatch, etc.
[^2]: Perhaps due to a regulatory reason?
