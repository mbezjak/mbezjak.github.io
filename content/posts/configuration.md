---
title: "Configuration"
date: 2022-11-23T18:30:41+01:00
draft: false
---

Which library do you use for project configuration? There are so many to choose
from. Take a look at [The Clojure Toolbox](https://www.clojure-toolbox.com) under
"Configuration". Choosing the right one mostly comes down to that you want out
of the configuration library.

Here are my requirements:

- It should not maintain it's own state. So
  [environ](https://github.com/weavejester/environ) is out.
- It should allow the project to control configuration sources (e.g. from
  environmental variables, java properties, command line arguments, file (edn?)
  on the classpath, external file, etc.) and their order.
  [cprop](https://github.com/tolitius/cprop) seems simple enough, but there is
  one other requirement.
- It should help document the configuration that the project uses. After all,
  someone needs to know how to configure the service properly. Is it
  `DATABASE_URL`? `DB_URL`? `DB`? What else is there to configure?
  [aero](https://github.com/juxt/aero) seems to be on the right track.

However, if those are the only requirements then implementing your own
configuration "library" becomes really simple in Clojure.

Here is what worked quite well for me.

`config_init.clj`

```clojure
(ns config-init
  (:refer-clojure :exclude [load])
  (:require
   [clojure.edn :as edn]
   [clojure.java.io :as io]
   [my.company.map :as map]
   [my.company.text :as text]))

;; Default values are production quality safe! It's up to development-like
;; environments to relax the constrains a bit.
(def ^:private known-configs
  [;; required
   {:key :db-url
    :env "DB_URL"}
   {:key :db-password
    :env "DB_PASSWORD"}
   {:key :keycloak-issuer-url
    :env "KEYCLOAK_ISSUER_URL"}
   {:key :keycloak-client-id
    :env "KEYCLOAK_CLIENT_ID"}

   ;; optional
   {:key :elasticsearch-url
    :env "ELASTICSEARCH_URL"}
   {:key :kms-key-id
    :env "KMS_KEY_ID"}

   ;; usually different in development-like environments
   {:key :enable-debug-logging
    :default false
    :env "ENABLE_DEBUG_LOGGING"
    :env-transform text/try-as-boolean}

   ;; rarely needs to be redefined
   {:key :api-port
    :default 8800
    :env "API_PORT"
    :env-transform text/try-as-int}
   {:key :flyway-version
    :default "latest"
    :env "FLYWAY_VERSION"}

   ;; for deployed service: added via Dockerfile
   {:key :git-tag
    :env "GIT_TAG"
    :env-transform not-empty}
   {:key :git-commit-sha
    :env "GIT_COMMIT_SHA"}
   {:key :build-time
    :env "BUILD_TIME"}

   ;; local development
   {:key :developer-focused-logs
    :default false
    :env "DEVELOPER_FOCUSED_LOGS"
    :env-transform text/try-as-boolean}])

(defn- from-defaults []
  (->> known-configs
       (filter #(contains? % :default))
       (map (juxt :key :default))
       (into {})))

(defn- from-local-edn []
  (let [f (io/file "local/resources/config.edn")]
    (when (.exists f)
      (edn/read-string (slurp f)))))

(defn- from-env []
  (let [m (->> known-configs
               (filter :env)
               (map (fn [{:keys [key env env-transform]}]
                      (let [transform (or env-transform identity)]
                        [key (transform (System/getenv env))])))
               (into {}))]
    (map/remove-val m nil?)))

(defn load []
  (merge (from-defaults)
         (from-local-edn)
         (from-env)))
```

That is all there is to it. Refer to [clojure.core
extensions](../clojure-core-extensions) for some of the functions used here.

Configuration for local development can be something like the following.

`local/resources/config.edn`

```clojure
{:db-password "postgres"
 :db-url "jdbc:postgresql://localhost:5432/db?user=postgres"
 :keycloak-issuer-url "https://auth.development.company.com/realms/customer"
 :keycloak-client-id "mobile-app"
 :enable-debug-logging true
 :developer-focused-logs true
 :git-commit-sha "local-development"
 :build-time "1970-01-01T00:00:00+00:00"}
```

## Extensions

The nice thing when you have your own implementation is that adding stuff that
matters to your project becomes really simple.

For example, if you have a `/health` endpoint that returns service configuration
then you probably don't want it returning sensitive values. That seems simple to
solve. Just annotate the configuration as needed and redact the values before
returning them.

```clojure
(def ^:private known-configs
   ,,,
   {:key :db-password
    :sensitive? true
    :env "DB_PASSWORD"}
   ,,,)

(defn redact-sensitive-values [config]
  (reduce #(assoc %1 %2 "<redacted>")
          config
          (->> known-configs (filter :sensitive?) (map :key))))
```

Other ideas:
- add `:documentation` to those variables where the usage is not clear enough
- add `:used-by #{:admin :mobile}` to signal which services are using this
  variable
