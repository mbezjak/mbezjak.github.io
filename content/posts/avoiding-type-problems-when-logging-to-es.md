---
title: "Avoiding Type Problems When Logging to ElasticSearch"
date: 2022-11-09T13:38:22+01:00
draft: false
---

If you're sending your log events to
[ElasticSearch](https://www.elastic.co/elasticsearch/), you might notice that
ElasticSearch sometimes fails to index some events due to type problems. A
single ES index cannot contain two documents having the same key but different
value types. For example, `{"user": 1}` (integer) and `{"user":
"jane@example.com"}` (string) cannot be written to the same ES index.

[mulog](https://github.com/BrunoBonacci/mulog) is an excellent logging library
for Clojure. It can also
[send](https://github.com/BrunoBonacci/mulog/blob/master/doc/publishers/elasticsearch-publisher.md)
your events to ElasticSearch. How does it deal with type problems? By using name
mangling. Simply put, name mangling means that for each key, mulog will
[append](https://github.com/BrunoBonacci/mulog/blob/0.9.0/mulog-elasticsearch/src/com/brunobonacci/mulog/publishers/util.clj#L31)
a type suffix to avoid the clash in ES. `{"user": 1}` becomes `{"user.i": 1}`
and `{"user": "jane@example.com"}` becomes `{"user.s": "jane@example.com"}`.
This covers almost all type problems.

**Except** there is one other case to cover. Types being different inside of a
JSON arrays. Arrays themselves are mangled (having `.a` suffix), but the
primitive values inside the array are not. Consider the following two documents
`{:users [1 2 3]}` (array of integers) and `{:users ["jane@example.com"
"john@example.com"]}` (array of strings). ES will refuse to index the subsequent
document due to a type problem. Mulog might not be in the best place to solve
this problem because it doesn't know anything about how you intend to use this
data. So, how to solve this problem anyway? There are a few ways that come to
mind:

1. Rename the keys. E.g. use `:user-ids` and `:user-emails` instead of `:users`
2. Eliminate the array from being logged. E.g. `(dissoc event :users)`
3. `(str v)` all the values inside of an array to make sure that they have the
   same type.
4. Use a different mulog index. E.g. use a different `:index-pattern` for each
   event type.

Those solutions have different trade-offs to consider. I usually don't use
primitive array values in Kibana, but like having them be a part of the same
event because sometimes it simplifies debugging. So far, solution no. 3. worked
well for me. Here is what that could look like.

`mulog_init.clj`

```clojure
(ns mulog-init
  (:require
   [clojure.walk :as walk]
   [com.brunobonacci.mulog.utils :as mulog.utils]))

(defn- transform-elasticsearch-types [events]
  (walk/postwalk
   (fn [c]
     (cond
       ;; ES doesn't like it when array element type changes.
       (or (seq? c)
           (set? c)
           (and (vector? c) (not (map-entry? c))))
       (map (fn [x]
              (if (or (number? x) (boolean? x))
                (str x)
                x))
            c)

       ;; com.brunobonacci.mulog.publishers.util/type-mangle mangles only
       ;; Exceptions, not any Throwable. This creates type problems (oh the
       ;; irony) in ES:
       ;; 1. Exception is encoded as `{"exception": {"x": "exception stack trace"}}`
       ;; 2. Throwable is encoded as `{"exception": "exception stack trace"}`
       ;; This of course cannot work. "exception" must either be an object or a string.
       ;; https://github.com/BrunoBonacci/mulog/issues/98
       ;; Here until v0.10.0 comes out.
       ;; https://github.com/BrunoBonacci/mulog/blob/master/CHANGELOG.md
       (instance? Throwable c)
       (mulog.utils/exception-stacktrace c)

       :else c))
   events))
```

Using the transformer.

```clojure
(mulog/start-publisher!
 {:type :elasticsearch
  :transform mulog-init/transform-elasticsearch-types
  :url ,,,})
 ```
