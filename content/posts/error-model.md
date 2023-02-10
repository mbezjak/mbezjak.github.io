---
title: "Error Model"
date: 2022-12-10T15:25:38+01:00
draft: false
---

In [part 1](../what-is-a-validation) we talked about the various tasks that can
result in an error. Libraries can help with those tasks. However, there is still
a whole bunch that the project will have to take care of. Let's continue from
there. Here we'll concentrate on the error model - the way the errors are
represented. What is that about?

Each library pulled in to help with the validation tasks represents errors in
its own way. The code you write in the project has even more ways to do the
same. Let's see a couple of examples of what that might look like.
- A function might throw a Java exception (e.g.
  `java.lang.IllegalArgumentException`) thus representing an error as a string
  message.
- A function might throw an `clojure.lang.ExceptionInfo` (via `ex-info`) thus
  representing an error as a string + hash map (via `ex-data`).
- A function might return `true`/`false` to indicate an error.
- A function might return an error message.
- A function might return a collection of error messages.
- `malli.core/explain` returns a hash map with `:errors` being a list of hash
  maps.
- `malli.error/humanize` returns a hash map of path -> vector of messages.
- `clojure.spec.alpha/explain-data` returns a hash map with `:problems` being a
  vector of hash maps.
- `buddy.sign.jwt/unsign` throws `ex-info` with `ex-data` having `:cause`. [^1]
- Calling an external service might return zero, one or many errors when it
  didn't succeed, but in what format? JSON (or something similar) might be
  preferable, but the service might respond in HTML as well.
- A function might return a pair `[error result]` with `error` being anything
  from above.
- etc.

Having to deal with a large subset of the above makes working on the project
unnecessarily hard. It's even harder when needing to collect multiple errors
from multiple sources all using their own error model.

What would make it easy is to have just one error model used throughout the
project. Preferably the one that best suits your project and that you can
control to make sure it will work best in the future as well. That seems easy
enough in Clojure. Just define your own error model and base everything on top
of it, wrapping libraries and external services if necessary.

## In search of a model

Pick a model that is generic enough to be able to represent any error your
project produces and can represent errors from the libraries you're using while
still not being so involved that it makes it cumbersome to use.

How might we represent an error?

Let's start simple. An error message as a string.

```clojure
(defn validate [country]
  (when-not (country-exists? country)
    (format "Country with code '%s' doesn't exist" country)))
```

A bit too simplistic? What if we want to attach `country` with the error? So,
perhaps a hash map?

```clojure
(defn validate [country]
  (when-not (country-exists? country)
    {:message (format "Country '%s' doesn't exist" country)
     :country country}))
```

A bit better, but what if we want to represent multiple errors, not just one?
Perhaps a collection of hash maps?

```clojure
(defn validate [country]
  (let [errors [(when-not (headquarters-in? country)
                  {:message (format "I'm sorry, we don't have company headquarters in %s" country)
                   :headquarters (available-headquarters)})
                (when (embargoed? country)
                  {:message (format "Sorry, by law we aren't allowed to conduct business in %s" country)
                   :country country})
                (when (too-far-away? country)
                  {:message "You are too far away from us"
                   :supported-time-zones (supported-time-zones)})]]
    (remove nil? errors)))
```

What about grouping errors together? Sure, if you need it. Continue on your own
from here. Just remember not to overcomplicate the model. You'll have to work
with it regularly.

A few notes to keep in mind:
- This post is about defining the model within your programming language.
  Clojure is used here, but the same idea can be applied elsewhere.
- It's not really about the HTTP response body. Although an identical model can
  be used for responses as well. Depending on the situation, the error hash map
  might be trimmed down to only the essentials (e.g. for public endpoints) or
  only slightly trimmed (e.g. for internal service-to-service communication).
- It's not really about the database model. It's mostly about the in-memory
  error model. Although, nothing is stopping you from saving errors to the DB if
  you have that requirement.
- When you decide on an error model, it can be used with exception throwing
  (`ex-info`) if you don't want to simply return errors.

## Model that works well enough

Here is one possible model that worked quite well for me for years.

The model:
- `error` is a hash map
- `errors` is a vector of `error`

That's it. It's simple, quite readable, and very extensible. The project
recognizes a couple of known error keys such as `:message`, `:code`, `:args`,
etc. However, at any point, for any reason, it allows the code to attach other
keys to carry additional information. Presumably with the intent that some
upstream code will use it. How exactly is up to you and the project you're
working on:
- respond with it to an HTTP request
- handle it in an exception middleware (ring)
- save it to the DB
- add it to the queue
- log the event to ElasticSearch
- collect as metric
- etc.

Here are the project-wide keys that are used most often.

### message

```clojure
{:message "Country with code 'XA' doesn't exist"}
```

This is the simplest variant that's the easiest to understand. It simply encodes
an error message indented for a human. That doesn't necessarily mean a customer.
Perhaps you're the one that's going to be reading it from logs. Or perhaps it's
indented for another person within your organization.

I don't use it much. I find `:code` and `:args` much more advantageous.

### code and args

Instead of embedding the error message directly, we can let some other code deal
with constructing the message, but at the point of creating an error, we simply
want to refer to the eventual message. It looks like this:

```clojure
{:code :country/not-found
 :args [country-code2]}
```

Where `country-code2` is probably in a `let` binding. This has a couple of
advantages:
- It facilitates service-to-service communication. If service A calls B and B
  responds with errors, it's easier to deal with the error code rather than a
  message intended for a human. Perhaps A needs to execute a different code path
  depending on if B responds with a specific error code. Information is easier
  to deal with than fishing out the relevant part with `re-find`.
- You can let error messages live in an externalized EDN file (or Java
  properties, etc.). That way, all error messages for the project are in one
  place instead of being spread all over the code base. It also frees the code
  base from needing to concatenate strings together. Instead, a ring middleware
  might simply execute `(apply format template args)` once it reads the message
  template. An example EDN file is just a hash map from the error code to the
  error template.

  `resources/error-templates.edn`

  ```clojure
  {:country/not-found "Country with code '%s' doesn't exist"
   :transaction/insufficient-funds "Not enough funds in the bank account (%s) for money transfer. Available amount: %s. Needed amount: %s"}
  ```

- It aids in translating error messages into other natural languages (German,
  Spanish, etc.). The solution might be similar to `error-templates.edn` or it
  could be much more involved. It depends on your project and what you need to
  solve. What's important is for the error to carry the information until some
  code higher up can decide what to do with it. Perhaps it needs
  `Accept-Language` from HTTP headers?

### breadcrumbs

Path to the source that caused the error. For example, for a deeply nested
structure that might be:

```clojure
{:code :malli/required
 :breadcrumbs [:customer :address :country :code]}
```

### suggested-http-code

How to influence the HTTP response status code? Most of the time the default
would be your favorite HTTP status code when an error occurs. What are you
using: 400, 409, 412, 422? Sometimes, however, you want to explicitly define the
status code. For example:

```clojure
{:code :jwt/expired
 :suggested-http-code 401}
```

Theoretically this can be ambiguous if `errors` are:

```clojure
[{:code :jwt/expired
  :suggested-http-code 401}
 {:code :produt/not-found
  :suggested-http-code 404}]
```

However, I haven't seen that ambiguity in practice. Mostly because the example
above doesn't make sense. If authentication fails then the middleware responds
to an HTTP request immediately instead of executing more code. Same with other
errors that want to suggest an HTTP code.

### Other ideas

Here are some more ideas for you to think about and maybe try.

- Add `:severity` to describe how important is this error. Values might be
  `:fatal`, `:warning`, `:note`, ...
- Add `:module` and make error messages like "Amount is required" a bit less
  ambiguous. What module produced this error? Was it: transaction, cart, order,
  warehouse, ...?
- Most of the time, I let exceptions flow all the way to the ring exception
  middleware because it has the most information on how to react. Sometimes,
  however, it's nice to be able to catch it and return it as an error. Perhaps
  you want to attach the cause as `:exception`?
- Add `:response` if service A called service B and B failed to do its job
  properly. Your REPL experience will be much nicer if you can see and interact
  with that information instead of just throwing it away.
- If any error occurs you'll likely want to trigger a DB rollback. Sometimes you
  might not want to do this. Add `:transaction-rollback? false` to indicate it
  to the higher-up code.
- Add `:contact-support? true` to let UI know and present a special screen to
  the customer. The error is such that the customer can do nothing about it
  except contact customer support.
- Add `:increment-metric :rare-errors` to let middleware know that it should
  record that something bad happened. Again!
- Mark error as `:probable-hacking-attempt? true`. For example, if you know that
  a customer only has permission to use tenant A, but is attempting to use
  tenant B. You are also sure that the UI would never navigate the customer in
  such a way. Someone is likely trying something. E.g. using `curl` against your
  API.
- Attach `:query-too-slow "SELECT ..."` to help you debug a query that took way
  too long to execute.
- Perhaps you cache something first, then check for costly constraints? If you
  find out that there is an error, you might want to let the middleware know to
  invalidate the cache in case of an error: `:invalidate-cache :cart`.
- Add `:origin :service-b` to make it clear that service A called B and B
  returned errors. The code in A is now dealing with errors from `:service-b`.
  Alternatively, add `:origin-chain [:service-x :service-y :service-z]` to
  indicate a bunch of services failed before reaching `:service-a`.
- Add `:at-fault` to indicate who's at fault. Was it: `:client`
  (web/mobile/desktop application) or `:server`? This information can be used to
  influence the default HTTP status code. E.g. 4xx vs. 500.
- Mark error as `:log? false` to indicate that logging middleware should not log
  this error. Perhaps because it contains sensitive information. E.g. user
  credentials.
- In a [later blog post](../errors-from-libraries), we'll inject a bunch of
  information that's available when using malli to validate the input.

It's all about carrying the information until some code higher up can decide
what to do with it. This is one of the ways to get to immutable core, imperative
shell you might have heard about.

Defining the model is one thing. How might we implement it? Does the model have
any useful operations? Well, yes. See the appendix below for one possible
implementation. Feel free to use it and adapt it to your situation.

## How to use it?

Now that we have the error model, implementation, and some useful operations
(see appendix below), how do we use it? I need a whole new blog post for this.
Stay tuned for [part 3](../using-the-error-model).

For now, just remember that the goal is for the project to use only `errors` (or
however you've defined your model). If something external (library, service,
etc.) is representing an error then convert it first into your own model.

For the most part, leave exceptional situations (e.g. IOException,
SocketException, SQLException) to some other code to deal with (e.g. ring
middleware). I'm not suggesting you start wrapping every function or Java method
that can throw an exception. This is about representing validation errors and
the like. Mostly about what we thought about in [part
1](../what-is-a-validation).

## Summary

Don't let someone else define your error model. It's too important. You know
what your project needs, so tailor it to your situation. Write your own and base
everything on top of it.

You want to be in the business of creating information. Some other code is then
going to decide to do something based on that information + a lot more from the
surrounding context. E.g. HTTP request, response, caught exception, project
configuration, etc.

## Appendix

{{< gist mbezjak c1baeece563b8ed734692938e6d1a36f error.clj >}}
{{< gist mbezjak c1baeece563b8ed734692938e6d1a36f errors.clj >}}

We'll see how to use this properly in [part 3](../using-the-error-model).
However, here are a few (crude) usage examples.

```clojure
;; creating errors
(errors/make-1 {:code :maintenance/in-progress})

(errors/make-1 {:code :country/not-found
                :args [country]})

(errors/make [{:code :headquarters/not-found
               :args [country]
               :headquarters (available-headquarters)}
              {:code :country/embargoed
               :args [country]
               :country country}
              {:code :country/too-far-away
               :supported-time-zones (supported-time-zones)
               :increment-metric [:need-to-expand-in country]}])


;; in authentication middleware
(let [errors (validate-authentication request)]
  (errors/set-suggested-http-code 401 errors))


;; in exception middleware (very simplified), assuming:
;; - `unwrap-ex` creates `errors` from any exception
;; - `get-message-tpls` returns message templates. See `resources/error-templates.edn` above
;; - `->errors-body` creates HTTP response body from `errors`
(defn wrap-exception [handler]
  (fn [request]
    (try
      (handler request)
      (catch Exception ex
        (let [errors (->> ex (unwrap-ex) (errors/with-message (get-message-tpls)))]
          {:status (or (errors/suggested-http-code errors) 400)
           :body (->errors-body errors)})))))
```

[^1]: https://github.com/funcool/buddy-sign/blob/3.4.333/src/buddy/sign/jwt.clj#L22
