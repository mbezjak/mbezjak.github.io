---
title: "Routing Slf4j Events To Mulog"
date: 2022-11-10T14:09:01+01:00
draft: false
---

[mulog](https://github.com/BrunoBonacci/mulog) plays to Clojure's strengths such
as declaring, using and manipulating data vs. composing and parsing strings.
That's why I like using it as a logging library. However, backend services might
use other Java libraries that only use what's available in the JVM ecosystem.
Best case scenario is that backend ends up using at least two logging libraries:
mulog and [slf4j](https://www.slf4j.org/). Each one has to be configured to work
properly. Even if they are, some questions still remain:

- Will the application output have structured events (EDN, JSON, etc.)
  intermixed with unstructured lines (slf4j)?
- Will slf4j lines be send to a central location? E.g. ElasticSearch or
  CloudWatch?
- How will events from both of those libraries be consumed? Do slf4j lines have
  to be parsed?

What if we took a different approach? Instead of configuring both libraries to
use their own appenders/publishers, how about we forward slf4j events to mulog?
That way we're still dealing with slf4j events as data (not strings) and we only
need to configure mulog:

- where to send events
- in what format should events be encoded

How does one go about doing that? I'll concentrate here only on routing slf4j
events to mulog. [Configuring](https://github.com/BrunoBonacci/mulog#publishers)
mulog or [bridging](https://www.slf4j.org/legacy.html) Java logging libaries
over slf4j (log4j, JCL, JUL over slf4j) is out of scope for this post.

What does it take to route slf4j events to mulog?

1. Make sure slf4j and logback are added to the project dependencies. In
   [Leiningen](https://leiningen.org/) that would look like:

   ```clojure
   :dependencies [,,,
                  [org.slf4j/slf4j-api "2.0.3"]
                  [ch.qos.logback/logback-classic "1.4.4"]
                  ,,,]
   ```

2. Remove any `logback.xml` files you might have on the classpath (e.g.
   `resources` directory). It's not needed and will be done programatically.

3. Write the appender that will forward all events to mulog and setup the
   routing.

   ```clojure
   (ns slf4j-init
     (:require
      [com.brunobonacci.mulog :as mulog])
     (:import
      (ch.qos.logback.classic Level Logger)
      (ch.qos.logback.classic.spi ILoggingEvent ThrowableProxy)
      (ch.qos.logback.core Appender AppenderBase)
      (org.slf4j LoggerFactory)))

   (defn- new-slf4j-to-mulog-appender ^Appender []
     (proxy [AppenderBase] []
       (append ^void [^ILoggingEvent event]
         (mulog/log ::slf4j
                    :message (.getFormattedMessage event)
                    :logger (.getLoggerName event)
                    :level (str (.getLevel event))
                    :thread-name (.getThreadName event)
                    :exception (when-let [ex-proxy (.getThrowableProxy event)]
                                 (.getThrowable ^ThrowableProxy ex-proxy))))))

   (defn setup-slf4j-to-mulog []
     (let [appender (new-slf4j-to-mulog-appender)
           logger-context (LoggerFactory/getILoggerFactory)
           ^Logger root-logger (LoggerFactory/getLogger Logger/ROOT_LOGGER_NAME)]
       ;; Remove the default appenders that logback installs by default when no
       ;; logback.xml is present. See:
       ;; https://logback.qos.ch/manual/configuration.html#auto_configuration
       (.detachAndStopAllAppenders root-logger)
       ;; Set the default logging level for all loggers
       (.setLevel root-logger Level/INFO)
       ;; Adding appender that forwards everything to mulog.
       (.setContext appender logger-context)
       (.start appender)
       (.addAppender root-logger appender)))

   (defn stop-slf4j-to-mulog []
     (let [^Logger root-logger (LoggerFactory/getLogger Logger/ROOT_LOGGER_NAME)]
       (.detachAndStopAllAppenders root-logger)))
   ```

4. Execute the setup on backend service start, REPL start or [before running
   the tests](../kaocha-start-system-hook).

   ```clojure
   (slf4j-init/setup-slf4j-to-mulog)
   ```

Note that there is a different approach to this. One that doesn't use
`logback-classic` as a dependency. I don't mind `logback-classic`, but if you
do, take a look at this approach: https://gitlab.com/nonseldiha/slf4j-mulog
