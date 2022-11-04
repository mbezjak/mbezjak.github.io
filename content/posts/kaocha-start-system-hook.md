---
title: "Start the System Before Executing Integration Tests in Kaocha"
date: 2022-11-04T15:59:25+01:00
draft: false
---

Kaocha is a very good tests runner for Clojure. The
[documentation](https://cljdoc.org/d/lambdaisland/kaocha/1.71.1119/) is really
good.

What I wanted to do recently is to start (and stop) the system before (and
after) running the integration tests. How to do that with hooks? If the project
has unit tests separated from integration tests then this solution should work.

`tests.edn`

```clojure
#kaocha/v1
 {:tests [{:id :unit
           :test-paths ["test/unit"]}
          {:id :integration
           :test-paths ["test/integration"]}]
  :plugins [:hooks]
  :kaocha.hooks/wrap-run [kaocha-hooks/wrap-tests]}
```

`test/kaocha_hooks.clj`

```clojure
(ns kaocha-hooks
  (:require
   [kaocha.hierarchy :as hierarchy]))

(defmulti start identity)
(defmulti stop identity)

(defmethod start :unit [_])
(defmethod stop :unit [_])

(defmethod start :integration [_]
  (println "Starting system to run integration tests")
  ,,,)

(defmethod stop :integration [_]
  (println "Stopping system to run integration tests")
  ,,,)

(defn wrap-tests [run]
  (fn [testable test-plan]
    (if (hierarchy/suite? testable)
      (try
        (start (:kaocha.testable/id testable))
        (run testable test-plan)
        (finally
          (stop (:kaocha.testable/id testable))))
      (run testable test-plan))))
```

And if your project needs to separate the integration tests as well (e.g. per
service) then it's just a matter of specifying that in `tests.edn`.

```clojure
#kaocha/v1
 {:tests [{:id :unit
           :test-paths ["test/unit"]}
          {:id :integration-service-a
           :test-paths ["test/service-a"]}
          {:id :integration-service-b
           :test-paths ["test/it/service-b"]}]
  :plugins [:kaocha.plugin/hooks]
  :kaocha.hooks/wrap-run [kaocha-hooks/wrap-tests]}
```
