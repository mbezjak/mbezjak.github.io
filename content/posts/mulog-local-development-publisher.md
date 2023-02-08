---
title: "Mulog Publisher for Nicer Local Development"
date: 2022-11-09T12:22:17+01:00
draft: false
---

[mulog](https://github.com/BrunoBonacci/mulog) has a couple of console
publishers ready for use:

- [Simple console
  publisher](https://github.com/BrunoBonacci/mulog/blob/master/doc/publishers/simple-console-publisher.md):
  output in EDN format
- [Advanced console
  publisher](https://github.com/BrunoBonacci/mulog/blob/master/doc/publishers/advanced-console-publisher.md):
  output in JSON format

Both can be used as publishers for **local development** (either running locally
or [executing tests](../make-mulog-play-nice-with-kaocha). The problem is that
their output is not something pleasant to look at. Even if the output is pretty
printed. Here is a contrived example, with just 5 events.

```
{:mulog/event-name :company.system.slf4j-init/slf4j,
 :mulog/timestamp 1668079374515,
 :mulog/trace-id #mulog/flake "4mNpLawLr6vG7UiZo44Uxi4U-MgZ1ugS",
 :mulog/namespace "company.system.slf4j-init",
 :exception nil,
 :level "INFO",
 :logger "com.zaxxer.hikari.HikariDataSource",
 :message "HikariPool-1 - Starting...",
 :service-name "admin",
 :thread-name "nREPL-session-28c2f6be-4dc4-4da5-ac67-b85d3f311112"}

{:mulog/event-name :company.api.middleware/http-request,
 :mulog/timestamp 1668080116247,
 :mulog/trace-id #mulog/flake "4mNq0m69wO_Nirx1NblMWLocruE7ep44",
 :mulog/namespace "company.api.middleware",
 :content-length nil,
 :content-type nil,
 :endpoint "GET /",
 :headers
 {"user-agent"
  "Mozilla/5.0 (X11; Linux x86_64; rv:106.0) Gecko/20100101 Firefox/106.0",
  "accept"
  "text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8",
  "accept-language" "en-US,en;q=0.5"},
 :ip-address nil,
 :method "GET",
 :params {},
 :protocol "HTTP/1.1",
 :request-body nil,
 :request-id #uuid "e6572d57-665f-43e0-b68b-fc442b7e4930",
 :scheme :http,
 :server-name "localhost",
 :server-port 8800,
 :service-name "admin",
 :thread-name "qtp1042791362-127",
 :uri "/",
 :user-agent
 "Mozilla/5.0 (X11; Linux x86_64; rv:106.0) Gecko/20100101 Firefox/106.0"}

{:mulog/event-name :company.api.middleware/http-response,
 :mulog/timestamp 1668080116248,
 :mulog/trace-id #mulog/flake "4mNq0m6DvqK-odmeeuyzY44gDL2GnbWh",
 :mulog/namespace "company.api.middleware",
 :duration-ms 1,
 :endpoint "GET /",
 :headers {"Location" "/admin-ui/"},
 :ip-address nil,
 :method "GET",
 :request-id #uuid "e6572d57-665f-43e0-b68b-fc442b7e4930",
 :server-name "localhost",
 :service-name "admin",
 :status 302,
 :thread-name "qtp1042791362-127",
 :uri "/",
 :user-agent
 "Mozilla/5.0 (X11; Linux x86_64; rv:106.0) Gecko/20100101 Firefox/106.0"}

{:mulog/event-name :company.api.middleware/http-request,
 :mulog/timestamp 1668080116584,
 :mulog/trace-id #mulog/flake "4mNq0nMJVdAaywZpVcKPxGtJQ46ovt2D",
 :mulog/namespace "company.api.middleware",
 :content-length nil,
 :content-type nil,
 :endpoint "GET /api/v1/ui-configuration",
 :headers
 {"user-agent"
  "Mozilla/5.0 (X11; Linux x86_64; rv:106.0) Gecko/20100101 Firefox/106.0",
  "accept" "application/json, text/plain, */*",
  "accept-language" "en-US,en;q=0.5"},
 :ip-address nil,
 :method "GET",
 :params {},
 :protocol "HTTP/1.1",
 :request-body nil,
 :request-id #uuid "97ac32d0-f848-4d91-8245-f3b310154b8a",
 :scheme :http,
 :server-name "localhost",
 :server-port 8800,
 :service-name "admin",
 :thread-name "qtp1042791362-122",
 :uri "/api/v1/ui-configuration",
 :user-agent
 "Mozilla/5.0 (X11; Linux x86_64; rv:106.0) Gecko/20100101 Firefox/106.0"}

{:mulog/event-name :company.api.middleware/http-response,
 :mulog/timestamp 1668080116586,
 :mulog/trace-id #mulog/flake "4mNq0nMm1AQX9ZpqW-NMe7r324ub74Od",
 :mulog/namespace "company.api.middleware",
 :duration-ms 0,
 :endpoint "GET /api/v1/ui-configuration",
 :headers nil,
 :ip-address nil,
 :method "GET",
 :request-id #uuid "97ac32d0-f848-4d91-8245-f3b310154b8a",
 :server-name "localhost",
 :service-name "admin",
 :status 200,
 :thread-name "qtp1042791362-122",
 :uri "/api/v1/ui-configuration",
 :user-agent
 "Mozilla/5.0 (X11; Linux x86_64; rv:106.0) Gecko/20100101 Firefox/106.0"}
```

What I'd like is to just glance at the output and let my eyes immediately jump
to an interesting part of the log without thinking too much. Standard Java
logging libraries are on the right track here (for local development). They
print a few key facts in column format. That way it's easier to ignore all
events (and their info) and focus on the *right* exception message (or whatever
is deemed important). Here are the same 5 events from above but presented a bit
nicer.

```
12:15:40.504 :company.system.slf4j-init/slf4j INFO [com.zaxxer.hikari.HikariDataSource] HikariPool-1 - Starting...
12:36:57.791 :company.api.middleware/http-request GET /admin-ui
{:params {}}
12:36:57.791 :company.api.middleware/http-response GET /admin-ui HTTP 200 in 1 ms
12:36:57.908 :company.api.middleware/http-request GET /api/v1/ui-configuration
{:params {}}
12:36:57.908 :company.api.middleware/http-response GET /api/v1/ui-configuration HTTP 200 in 0 ms
```

How can we get the output above? By writing a new publisher. This is not a big
deal with mulog.

A few notes for the code below:

- Add your own keys (and their values) that you don't consider worth looking at
  during development.
- Change the time formatting as you see fit. I don't consider date to be
  important enough for local development.
- `(:endpoint event)` is something like `"POST /api/v1/users"`. This is useful
  not only here, but also in [Kibana](https://www.elastic.co/kibana/).

`mulog_init.clj`

```clojure
(ns mulog-init
  (:require
   [clojure.pprint :as pprint]
   [com.brunobonacci.mulog.buffer :as buffer]
   [com.brunobonacci.mulog.publisher :as publisher]
   [java-time.api :as time])
  (:import
   (com.brunobonacci.mulog.publisher PPublisher)))

(defn- ignore-uninteresting [event]
  (dissoc event
          :mulog/timestamp :mulog/event-name :mulog/trace-id :mulog/namespace
          :service-name :request-id :method :uri :user-agent :ip-address
          :server-name :thread-name))

(deftype LocalDevelopmentPublisher [buffer]
  PPublisher
  (agent-buffer [_]
    buffer)

  (publish-delay [_]
    200)

  (publish [_ buffer]
    ;; events are pairs [offset <event>]
    (doseq [event (map second (buffer/items buffer))]
      (let [ts (time/format
                (time/formatter "HH:mm:ss.SSS")
                (time/local-date-time (time/instant (:mulog/timestamp event)) (time/zone-id "Europe/Vienna")))
            en (:mulog/event-name event)]
        ;; We do this only for select few and very common events because we want
        ;; a good balance between good looking logs vs. having to maintain this.
        (condp = (:mulog/event-name event)
          :slf4j-init/slf4j
          (if-let [exception (:exception event)]
            (printf "%s %s %s [%s] %s %s\n" ts en (:level event) (:logger event) (:message event) (pr-str exception))
            (printf "%s %s %s [%s] %s\n" ts en (:level event) (:logger event) (:message event)))

          :middleware/http-request
          (printf "%s %s %s\n%s" ts en (:endpoint event)
                  (with-out-str (pprint/pprint (dissoc (ignore-uninteresting event)
                                                       :headers :protocol :scheme :thread-name
                                                       :server-port :endpoint))))

          :middleware/http-response
          (printf "%s %s %s HTTP %s in %s ms\n" ts en (:endpoint event) (:status event) (:duration-ms event))

          (printf "%s %s\n%s" ts en (with-out-str (pprint/pprint (ignore-uninteresting event)))))))
    (flush)
    (buffer/clear buffer)))

(defn local-development-publisher []
  (LocalDevelopmentPublisher. (buffer/agent-buffer 10000)))
```

Then you can [use](https://github.com/BrunoBonacci/mulog#publishers) the
publisher by using [inline
type](https://github.com/BrunoBonacci/mulog/blob/master/doc/publishers/inline-publishers.md):

```clojure
(mulog/start-publisher!
 {:type :inline :publisher (mulog-init/local-development-publisher)})
```
