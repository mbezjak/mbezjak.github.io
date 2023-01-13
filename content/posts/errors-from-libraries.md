---
title: "Errors From Libraries"
date: 2023-01-13T18:00:24+01:00
draft: false
---

At this point, we've [defined the error model](../error-model) and [know how to
use it](../using-the-error-model). Until now, we focused on *custom* validations
in the project. However, what if the errors originate in the libraries you might
be using for [some
validation](../what-is-a-validation#what-can-result-in-an-error)? Open-source
libraries cannot possibly know about your own error model. They have their way
of representing and returning errors. We hinted at the solution already -
[convert library errors into your own model](../error-model#how-to-use-it).
Let's do that here.

I'll use [malli](https://github.com/metosin/malli) as an example, but the same
idea can be applied to any other library you might be using.
- [Clojure spec](https://clojure.org/guides/spec)
- [Plumatic schema](https://github.com/plumatic/schema)
- [buddy.sign.jwt/unsign](https://github.com/funcool/buddy-sign/blob/3.4.333/src/buddy/sign/jwt.clj#L22)
- etc.

## Conversion

For clarity, let's reiterate what we want to achieve.
- We want to have one error representation - our own error model. This makes
  dealing with errors simple.
- We want errors to collect as much information as possible to make debugging
  easier. Remember that the error model is an in-memory model. It doesn't
  necessarily mean that everything you collect is going to be used as an HTTP
  response.
- We want to carry that information to the code that can decide what it's going
  to do with it. Perhaps it'll use a subset of that information to respond to an
  HTTP request, log the error, collect metrics, write to DB, etc.

How to do that for an external library? By wrapping. If some function returns
that library's error representation then convert it on the fly to our own error
model.

For completeness, let's just state the obvious. Wrapping every function in that
library is not the goal. The library is not the problem. "Non-standard" error
representation is. For example, `malli.core/validate` is not a good candidate
for wrapping. It's a reasonable function to call when you only want a yes or no
answer, but the boolean result is not something that can be usefully converted
to own error model. Keep those calls in your project as they are.

```clojure
(malli/validate [:string] "1")
=> true
(malli/validate [:string] 1)
=> false
```

The perfect candidate is `malli.core/explain`.

```clojure
(malli/explain [:map
                [:user [:map {:closed true}
                        [:first-name :string]
                        [:last-name :string]]]]
               {:user {:first-name "John"
                       :last-name "Doe"}})
=> nil

(malli/explain [:map
                [:user [:map {:closed true}
                        [:first-name :string]
                        [:last-name :string]]]]
               {:user {:name "John Doe"}})
=>
{:schema [:map [:user [:map {:closed true} [:first-name :string] [:last-name :string]]]]
 :value {:user {:name "John Doe"}}
 :errors '({:path [:user :first-name]
            :in [:user :first-name]
            :schema [:map {:closed true} [:first-name :string] [:last-name :string]]
            :value nil
            :type :malli.core/missing-key}
           {:path [:user :last-name]
            :in [:user :last-name]
            :schema [:map {:closed true} [:first-name :string] [:last-name :string]]
            :value nil
            :type :malli.core/missing-key}
           {:path [:user :name]
            :in [:user :name]
            :schema [:map {:closed true} [:first-name :string] [:last-name :string]]
            :value "John Doe"
            :type :malli.core/extra-key})}
```

The hash-map that `malli.core/explain` returns is what we would like to convert
to our [own error model](../error-model#model-that-works-well-enough).

That seems simple enough. It's just a transformation from one data type into
another. We just need to keep in mind that malli comes with a lot of [built-in
schemas](https://github.com/metosin/malli#built-in-schemas). Here is a snippet
of what that transformation would look like.

`malli_validator.clj`

```clojure
,,,

(def ^:private malli-code->error-code
  {'nil? :malli/null
   'some? :malli/some
   'boolean? :malli/boolean
   'true? :malli/true
   'false? :malli/false
   'number? :malli/number
   ::malli/missing-key :malli/required
   ::malli/extra-key :malli/extra-key
   ,,,})

(defn- malli->error [root-schema root-value malli-error]
  ,,,
  (assoc-breadcrumb-info
   (merge {::root-schema root-schema
           ::root-value root-value
           ::schema error-schema
           ::value error-value
           ::path (:path malli-error)
           ::in (:in malli-error)}
          (if-let [type (:type malli-error)]
            {:code (get malli-code->error-code type)}
            {:code (get malli-code->error-code schema-type)})
          ,,,)))

(defn validate [schema value]
  (when-let [{:keys [schema value errors]} (malli/explain schema value)]
    (->> errors
         (map #(malli->error schema value %))
         (errors/make))))
```

Given that we're referring to the error message via `:code`, we also need the
[error templates](../error-model#code-and-args).

`resources/error-templates.edn`

```clojure
{:malli/required "%s is required"
 :malli/extra-key "System doesn't recognize property %s"
 :malli/null "%s must be empty"
 :malli/some "%s must not be null"
 :malli/boolean "%s must be a boolean"
 :malli/true "%s must have a value of `true`"
 :malli/false "%s must have a value of `false`"
 :malli/number "%s must be a number"
 ,,,}
```

That is the core idea. It's relatively simple. To complete the above we'd need
to take a look at the malli's implementation because the documentation doesn't
describe everything we need to know to complete it.

Or you can just refer to the full implementation below.

## Malli validator

The full implementation with tests and error templates is here:
https://gist.github.com/mbezjak/a76b737cd6330e60b60c78b7e2c8fb9e

Notes regarding the implementation:
- The implementation is compatible with malli 0.9.2. Latest at the time of this
  writing.
- `malli-code->error-code` might be sensitive to malli's internal implementation
  (or breaking changes). E.g. new malli version might add or remove a schema. To
  guard against that, just call
  `malli-validator/check-compatibility-with-malli!` at the start of your
  integration tests (or system start). It's used to check if `malli-validator`
  is still in sync after you've updated malli.
- The implementation includes the automatic generation of `args`, human
  (developer) breadcrumbs, and error templates with certain wording. It's geared
  towards both HTTP service response as well as responding to a web application
  with the intent of displaying it to the customer (mostly as a global toast
  element vs. below the form field). Feel free to modify `malli-validator` and
  templates to suit your need.
- Refer to [clojure.core extensions](../clojure-core-extensions) for some of the
  functions used in the implementation.

## Alternative implementation

You might be using `malli.error/humanize` in your project.

```clojure
(me/humanize
 (malli/explain [:map
                 [:user [:map {:closed true}
                         [:first-name :string]
                         [:last-name :string]]]]
                {:user {:name "John Doe"}}))
=>
{:user {:first-name ["missing required key"]
        :last-name ["missing required key"]
        :name ["disallowed key"]}}
```

Can that be used instead of `malli.core/explain` to generate own error model?
Yes, it can! The only problem is that you'd be throwing a lot of information.
For example, `malli.core/explain` gives you very nested schemas and values that
failed, not just the root schema and value. Besides that, you'd be losing the
programmatic access to why an int or string failed the validation: is it due to
min/max constraints, is it null, type mismatch, etc? With `humanize` that
information is embedded inside of the error message itself, while with `explain`
it can be embedded both in the error message and as a specific `:code` (e.g.
`:malli/required`, `:malli/string`, `:malli/string-with-length-between`,
`:malli/string-with-length-max`, `:malli/string-with-length-min`). However, if
you don't care about that information (not even while debugging?) then you can
use the result from `humanize`. Just be aware that the result is not a simple
"property -> vector-of-messages" hash-map, but can be nested as indicated in the
example above.

You could even combine the results of `malli.core/explain` and
`malli.error/humanize` to collect all of the information from `explain`, but
also reuse the error messages from `humanize`. This could get rid of duplicated
templates between `error-template.edn` and `malli.error/default-errors`. Feel
free to consider if this is worthwhile for your project.

## What to do with the errors?

```clojure
(let [errors (malli-validator/validate schema value)]
  ,,,)
```

Now that we have the `errors`, what do we do with them? We are in the same
situation as before. Please see the [previous
suggestion](../using-the-error-model#what-to-do-with-the-errors).
