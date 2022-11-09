---
title: "Mulog Publisher for Nicer Local Development"
date: 2022-11-09T12:22:17+01:00
draft: true
---

[mulog](https://github.com/BrunoBonacci/mulog) has a couple of console
publishers already baked in:

- [Simple console
  publisher](https://github.com/BrunoBonacci/mulog/blob/master/doc/publishers/simple-console-publisher.md):
  output in EDN format
- [Advanced console
  publisher](https://github.com/BrunoBonacci/mulog/blob/master/doc/publishers/advanced-console-publisher.md):
  output in JSON format

Both can be used as publishers for **local development** (either running locally
or [executing tests]({{<ref "/posts/make-mulog-play-nice-with-kaocha.md">}})).
The problem is that their output is not something pleasant to look at. Even if
the output is pretty printed. What I'd like is to just glance at the output and
let my eyes immediately jump to an interesting part of the log without thinking
too much. Standard Java logging libraries are on the right track here (for local
development). They print a few key facts in column format. That way it's easier
to ignore all events (and their info) and focus on the *right* exception message
(or whatever is deemed important).

So why not just write your own publisher? Here is what worked for me.

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

(defn- local-development-publisher []
  (LocalDevelopmentPublisher. (buffer/agent-buffer 10000)))
```

Then you can [use](https://github.com/BrunoBonacci/mulog#publishers) the
publisher by using [inline
type](https://github.com/BrunoBonacci/mulog/blob/master/doc/publishers/inline-publishers.md):

```clojure
(mulog/start-publisher!
 {:type :inline :publisher (mulog-init/local-development-publisher)})
```
