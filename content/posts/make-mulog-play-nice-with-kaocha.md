---
title: "Make Mulog Play Nice With Kaocha"
date: 2022-11-04T17:16:10+01:00
draft: false
---

[mulog](https://github.com/BrunoBonacci/mulog) is an excellent logging library
for Clojure. It uses agents to process events asynchronously. This makes sense
for deployed services.

But what about executing (integration, functional, etc.) tests? Some tests
runners (e.g. [Kaocha](https://github.com/lambdaisland/kaocha)) try to capture
the test output while the test is executing and provide that output in case the
test failed. Do those two libraries play well together? Not really. The output
captured cannot be relied upon:

- either the test output was not captured during the test execution at all (due
  to publish delay)
- or the output of previous test (or several of them) was captured

Not really what we want when we're trying to understand why the test failed.
Worse still, there doesn't seem to be any configuration option to improve the
situation. And the [test
code](https://github.com/BrunoBonacci/mulog/blob/0.9.0/mulog-core/test/com/brunobonacci/mulog/test_publisher.clj#L65)
that mulog itself uses doesn't seem to be in a good direction for most projects
because it makes all the tests wait for the next publish delay to be sure the
events were processed.

Luckily not all is lost. We can use Clojure's late binding to change mulog.
After a bit of looking around in mulog's source code and a dash of thinking,
this is what I came to use. It works reliably and I didn't have any problems
with it so far.

`mulog_init.clj`

```clojure
(ns mulog-init
  (:require
   [com.brunobonacci.mulog.buffer :as buffer]
   [com.brunobonacci.mulog.core :as mulog-core]
   [com.brunobonacci.mulog.publisher :as publisher]))

(defn make-mulog-sychronous []
  ;; What?
  ;; Mulog handles events in asynchronous nature. This makes sense. Event
  ;; processing and send off is done in another thread so that the main thread
  ;; is not blocked. This code makes mulog handles events synchronously.
  ;;
  ;; Why?
  ;; Test runners (e.g. kaocha) capture test output while the test is running.
  ;; Capturing doesn't work at all if the event processing is asynchronous. What
  ;; usually happens is that:
  ;; - test B captures output from previously executed test A
  ;; - and/or output is completely lost
  ;; To have correct output capture, the event processing must be synchronous.
  ;;
  ;; How?
  ;; The hack below is exploiting the fact that all `mulog/log` calls go through
  ;; `mulog-core/enqueue!`. From there the event processing is roughly (from
  ;; reading of the source code):
  ;; 1. added to an atom (for performance reasons)
  ;; 2. (in a new thread) picked up events from an atom and send to each
  ;;    publisher's agent-buffer
  ;; 3. (possibly in yet another thread) picked up events from agent-buffer and
  ;;    process them by executing each publisher independently
  ;; The hack is consisting of hijacking 1. and and immediately giving the event
  ;; to each publisher.
  ;;
  ;; Note!
  ;; Because this is a hack, the implementation needs to be checked each time
  ;; mulog dependency gets updated.
  (let [f (fn [_ value]
            ;; we are even using a private function in `mulog-core`
            (let [event (apply @#'mulog-core/merge-pairs value)]
              (doseq [publisher (map :publisher (map second @mulog-core/publishers))]
                (publisher/publish publisher (buffer/enqueue (buffer/ring-buffer 1) event)))))]
    (alter-var-root #'mulog-core/enqueue! (constantly f))))
```

Above code can be used as part of system startup in [Kaocha hooks]({{<ref
 "/posts/make-mulog-play-nice-with-kaocha.md" >}}).
