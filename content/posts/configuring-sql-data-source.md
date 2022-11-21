---
title: "Configuring SQL Data Source"
date: 2022-11-21T10:00:27+01:00
draft: false
---

What are you using to configure an SQL data source? Clojure
[wrappers](https://github.com/tomekw/hikari-cp)? Own abstractions? Plain
[implementation](https://github.com/brettwooldridge/HikariCP)?

Perhaps you're seeing something weird in production and want to investigate? Do
questions such as these come up now and again:
- How many concurrent connections does the pool allow?
- Is this ok time to do RDS snapshot?
- What happens to existing connections if RDS restarts?
- How long is the connection timeout?
- How long is the statement timeout?
- How long is the lock timeout?
- etc.

To find the answer, do you have to read and understand:
- connection pool's documentation?
- JDBC driver's documentation?
- RDS engine defaults?
- wrapper's documentation?
- wrapper's source code?

Can easy-to-read data source configuration help answer those questions? I find
this to be one of those situations where readability and unambiguousness are
more important that avoiding the copy and paste of defaults, using wrappers or
own abstractions. It doesn't take much more to configure the data source.

```clojure
(ns db-init
  (:import
   (com.zaxxer.hikari HikariConfig HikariDataSource)))

(defn start [url password]
  ;; https://github.com/brettwooldridge/HikariCP
  (let [hc (doto (HikariConfig.)
             (.setJdbcUrl url)
             (.setPassword password)
             (.setAutoCommit true)
             (.setConnectionTimeout (* 30 1000))
             (.setIdleTimeout (* 10 60 1000))
             (.setMaxLifetime (* 30 60 1000))
             (.setMinimumIdle 3)
             (.setMaximumPoolSize 10)
             (.setConnectionInitSql nil)
             (.setValidationTimeout (* 5 1000))
             (.setLeakDetectionThreshold (* 60 1000))
             ;; Global postgres timeout settings. If a particular transaction
             ;; needs different timeouts just execute SQL statements such as:
             ;;    SET LOCAL statement_timout = '30s';
             ;;    SET LOCAL lock_timout = '30s';
             ;; https://jdbc.postgresql.org/documentation/use/
             ;; https://www.postgresql.org/docs/11/runtime-config-client.html
             ;; https://stackoverflow.com/questions/20963450/controlling-duration-of-postgresql-lock-waits
             (.addDataSourceProperty "options" "-c statement_timeout=60s -c lock_timeout=5s"))]
    (HikariDataSource. hc)))
```

Note that the actual configuration is entirely contextual and depends on the
service you're building. E.g. having a lock timeout or large statement timeout
might be good enough (even beneficial) for an internal admin service that
doesn't receive a lot of requests. A service with 100 requests per second might
need to configure the data source a bit differently.
