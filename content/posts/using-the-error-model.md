---
title: "Using the Error Model"
date: 2023-01-01T23:05:14+01:00
draft: false
---

Let's see how to use the [error model we defined in part 2](../error-model). The
one that works well enough:

- `error` is a hash map
- `errors` is a vector of `error`

Before we get started:
- As before, we'll concentrate on the functions that [validate
  something](../what-is-a-validation).
- Given the error model above, we'll want every validation function to return
  `errors`.
- We'll build on top of the [implementation and existing
  operations](https://gist.github.com/mbezjak/c1baeece563b8ed734692938e6d1a36f).
- The focus here won't be on the individual error - what goes in the hash map.
- The focus here will be on creating and combining multiple errors.

Ready? Let's go.

## Cutting the boilerplate

Let's start simple and work our way up. A function that returns a single error.

```clojure
(defn validate [,,,]
  (errors/make
   [{:code :account/destination-not-found}]))
```

`defn`, a function name, and arguments are an unneeded distraction so we'll drop
them from the following code snippets.

Returning two or more errors is equally straightforward.

```clojure
(errors/make
 [{:code :account/destination-not-found}
  {:code :account/insufficient-amount}])
```

Shouldn't all that be behind a conditional? Let's try.

```clojure
(errors/make
 [(when-not dest-account {:code :account/destination-not-found})
  (when (neg? remaining-amount) {:code :account/insufficient-amount})])
```

Hmm. That doesn't look right. We might get `[nil nil]` as an argument to
`errors/make`. `nil` doesn't represent an error. According to the model, an
error must be a hash map. So let's take `nil` out. Let's also return `nil` (vs.
`[]`) to indicate no errors.

```clojure
(->> [(when-not dest-account {:code :account/destination-not-found})
      (when (neg? remaining-amount) {:code :account/insufficient-amount})]
     (remove nil?)
     (errors/make)
     (not-empty))
```

Sometimes it's nice to introduce new bindings close to the error vs. at the top
of the function.

```clojure
(->> [(when-not dest-account {:code :account/destination-not-found})
      (when (neg? remaining-amount) {:code :account/insufficient-amount})
      (when-let [invalid (invalid-characters transaction-description)]
        {:code :transaction/description-invalid-characters
         :args [invalid]
         :allowed (allowed-sepa-characters)})]
     (remove nil?)
     (errors/make)
     (not-empty))
```

However, what when a `let` needs to return multiple errors? Use `flatten`.

```clojure
(->> [(when-not dest-account {:code :account/destination-not-found})
      (when (neg? remaining-amount) {:code :account/insufficient-amount})
      (when-let [invalid (invalid-characters transaction-description)]
        {:code :transaction/description-invalid-characters
         :args [invalid]
         :allowed (allowed-sepa-characters)})
      (let [sepa-countries (sepa-participating-country-codes)
            can-be-local? (local-capable-transport? transaction)
            can-be-sepa? (contains? sepa-countries destination-country-code)
            must-be-swift? (and (not can-be-local?) (not can-be-sepa?))
            user-chose-local? (user-chose-local-transport? transaction)
            user-chose-swift? (user-chose-swift-transport? transaction)]
        [(when (and can-be-local? (not user-chose-local?))
           {:code :exchange/local-exchange-is-cheaper})
         (when (and can-be-sepa? user-chose-swift?)
           {:code :exchange/sepa-exchange-is-cheaper})
         (when (and can-be-sepa? user-chose-local?)
           {:code :exchange/must-use-sepa-transport})
         (when (and must-be-swift? (not user-chose-swift?))
           {:code :exchange/must-use-swift-transport})])]
     (flatten)
     (remove nil?)
     (errors/make)
     (not-empty))
```

This also works nicely with `for`.

```clojure
(->> [(when-not (:source-account monthly-salaries) {:code :account/source-not-found})
      (when (neg? remaining-amount) {:code :account/insufficient-amount})
      (when (empty? (:transactions monthly-salaries)) {:code :monthly-salaries/empty})
      (for [salary-transaction (:transactions monthly-salaries)]
        [(validate-transaction salary-transaction)
         (when (business-account? salary-transaction)
           {:code :monthly-salaries/contracting-work-should-be-separate})])]
     (flatten)
     (remove nil?)
     (errors/make)
     (not-empty))
```

What about when wanting to reuse existing functions?

```clojure
(->> [(validate-source-account (:source-account transaction))
      (validate-destination-account (:destination-account transaction))
      (validate-currency (:currency transaction))
      (validate-amount (:amount transaction) (:currency transaction))
      (validate-description (:description transaction))
      (validate-possible-fraud transaction)
      ,,,]
     (flatten)
     (remove nil?)
     (errors/make)
     (not-empty))
```

So far so good. There is some boilerplate that we need to write to ensure the
consistency of the error model. However, that's easy to get rid of. We just need
a new function in `errors.clj`:

```clojure
(defn validate [& results]
  (->> results
       (flatten)
       (remove nil?)
       (make)
       (not-empty)))
```

Then the boilerplate is replaced with a single `errors/validate` call. This acts
as s signal to the reader reading the code that the code is about to concatenate
all the errors.

```clojure
(errors/validate
 (validate-source-account (:source-account transaction))
 (validate-destination-account (:destination-account transaction))
 (validate-currency (:currency transaction))
 (validate-amount (:amount transaction) (:currency transaction))
 (validate-description (:description transaction))
 (validate-possible-fraud transaction)
 ,,,)
```

## Combining validations conditionally

What gets hairy is conditionally running validation depending on if the previous
function call returned errors or not. For example, something like this: [^1]

```clojure
(if-let [lexical-errors (lexical-analysis program-representation)]
  lexical-errors
  (if-let [syntactic-errors (syntax-analysis program-representation)]
    syntactic-errors
    (if-let [semantic-errors (semantic-analysis program-representation)]
      semantic-errors
      (errors/validate
       (validate-output-for-x86 program-representation)
       (validate-output-for-arm program-representation)
       (validate-output-for-risc-v program-representation)
       (validate-output-for-js program-representation)))))
```

In my experience:
- "Don't validate if previous validation failed" is surprisingly common.
- Groups of validations that run together can be arbitrary in size.
- Ideally, validations should have as little [accidental
  complexity](https://github.com/papers-we-love/papers-we-love/blob/master/design/out-of-the-tar-pit.pdf)
  as possible.

This means we need a little help to make the code a bit more readable and
understandable. How about something like this?

```clojure
(errors/validate-groups
 (lexical-analysis program-representation)
 errors/continue-if-above-success
 (syntax-analysis program-representation)
 errors/continue-if-above-success
 (semantic-analysis program-representation)
 errors/continue-if-above-success
 (validate-output-for-x86 program-representation)
 (validate-output-for-arm program-representation)
 (validate-output-for-risc-v program-representation)
 (validate-output-for-js program-representation))
```

That's a bit nicer. The reader can clearly see which errors are going to be
concatenated together and what will be executed separately if the code above
doesn't yield errors. See the appendix for the implementation of this macro.

## Functions that want to return more than just errors

What about when we have a function that wants to return either a successful
value (not simply `nil`) or `errors`? E.g. `parse-int`?

Here is one way to handle them. Break the function into two parts:
- a success-or-nil function [^2]
- a validation function

For example:

```clojure
(defn parse-int [text]
  (try
    (Integer/parseInt text)
    (catch NumberFormatException _
      nil)))
```

With a validation.

```clojure
(defn validate-int [text]
  (let [x (parse-int text)]
    (errors/validate
     (when-not x
       {:code :core/not-a-number
        :args [text]}))))
```

Be aware that this does have drawbacks.

`parse-int` will be called twice: once for the validation and once after all the
validations succeeded to get the integer and do stuff with it. This has negative
performance characteristics that you might not be ok with.

This technique works for simple cases but breaks down for more complex ones.
E.g. `call-external-service` might not want to return `nil` because that means
losing a lot of information needed for debugging and potentially error
reporting. Information like `request`, `response`, exception, etc. In cases such
as those, it's better to seek [alternative
implementation](#alternative-implementations) or simply [rethink how to
implement](#what-to-do-with-the-errors) `call-external-service`.

## Alternative implementations

Depending on your project, `errors/validate` and `errors/validate-groups` can be
good enough for most use cases. If not then here are two alternatives to
consider: either and returning a pair. Feel free to consider your own
alternatives. The goal should be to make it easy to deal with errors in your
project.

If you have [some](http://learnyouahaskell.com)
[experience](https://wiki.haskell.org/Typeclassopedia)
[with](https://scalaz.github.io/7/) [monads](https://github.com/funcool/cats),
you might already know that a biased either monad can model a return of a
successful value or `errors`. I
[played](https://github.com/mbezjak/playground/tree/572f8f424189dc6358326412ad5cfe678b24bffe/clojure/cats)
around with the monads before, but so far I haven't needed them in Clojure.
However, for this particular use case, either monad might be worthwhile.

Here is what that would look like.

```clojure
(defn parse-int [text]
  (try
    (either/success (Integer/parseInt text))
    (catch NumberFormatException e
      (either/failure-1 {:code :core/not-a-number
                         :args [text]
                         :exception e}))))
```

See the appendix below for the implementation of a biased either monad.

Monads are used not by constantly peaking inside, but by using higher-order
functions to let the monad handle what it was designed to do. E.g. this doesn't
work so nicely because both `x` and `y` are monads, not integers.

```clojure
(let [x (parse-int ,,,)
      y (parse-int ,,,)]
  (+ x y))
```

A sufficiently powerful macro can help here.

```clojure
(either/mlet [x (parse-int ,,,)
              y (parse-int ,,,)]
  (+ x y))
```

The problem with a monad is that it "infects" everything else. That is, once a
monad is produced, the way to interact with it is to use `either/mlet` or
`either/map-success` to produce another one. Every code dealing with a monad
needs to do the same. All the way to the top. That doesn't look so nice in
Clojure. On the other hand, if your project has a very shallow call stack then
there aren't many places to consider. The trade-off might be worth it in your
project.

What about if you have a function that wants to return both `errors` and a
(partial) success value? E.g. `parse-amounts` might want to return successfully
parsed amounts as well as `errors` about the amounts that failed to parse. This
is a case for a product type, not a sum type. You might want to create a new
`deftype`. Or you can simply represent them as pairs in Clojure:
`[successful-value errors]` [^3].

## What to do with the errors?

However you end up collecting errors, the final question remains: what to do
with them once you have them? The code that generated them is rarely the best
place to deal with them. Most of the time they have to be carried to the
higher-up code for it to decide what to do with them. How to do that?

If you've collected as much as you can then at some point they will simply get
in the way: [^4]
- you have to pass `errors` along, and/or constantly check them with `if`
- or if using `either` then you constantly have to be using
  `either/map-success`, `either/mlet`, or similar

What I saw works well in practice is to simply throw them as an exception (via
`ex-info`). Yes, the boring Java `throw`. It's a simple solution to the problem
of letting [^5] some higher-up code know that it has to deal with the produced
errors. Perhaps it might also trigger a DB rollback which might be desirable.

Here is what that might look like.

```clojure
(defn transfer-money [account-from account-to amount]
  (errors/throw! (validate ,,,))
  (initiate-transfer ,,,))
```

Same for the previously mentioned` call-external-service`.

```clojure
(defn call-external-service [,,,]
  (let [request ,,,
        response (http-client/request request)]
    (when-not (expected-response? response)
      (errors/throw! (errors/make-1 {:code :external-service/failed
                                     :args [(:uri request)]
                                     :request request
                                     :response response})))
    (:body response)))
```

Who can then deal with those exceptions? In an HTTP service, a ring middleware
can. We saw already what that might look like in [part
2](../error-model#appendix). Here is the same snippet, a bit expanded.

```clojure
(defn wrap-exception [handler]
  (fn [request]
    (try
      (handler request)
      (catch Exception ex
        (cond
          (errors/validation-exception? ex)
          (let [errors (->> ex (errors/unwrap-exception) (errors/with-message (get-message-tpls)))]
            {:status (or (errors/suggested-http-code errors) 400)
             :body (->errors-body errors)})

          :else
          {:status 500
           :body (exception->errors-body ex)})))))
```

## Appendix: Additions to errors.clj

Refer to [clojure.core extensions](../clojure-core-extensions) for some of the
functions used here. For tests, please see the full gist:
https://gist.github.com/mbezjak/1112a321d12c7aaf41a2d7140f2a535a

{{< gist mbezjak 1112a321d12c7aaf41a2d7140f2a535a errors.clj >}}

## Appendix: Biased either monad

The code is not fully complete, but should be enough to get you started. For
tests, please see the full gist:
https://gist.github.com/mbezjak/193845228a71b826f58a95a9b2e7194e

{{< gist mbezjak 193845228a71b826f58a95a9b2e7194e either.clj >}}

[^1]: This is of course a contrived example. In reality, a compiler doesn't
    merely validate a `program-representation` but also takes the input and
    produces intermediate results such as AST.
[^2]: [Clojure 1.11](https://clojure.org/news/2022/03/22/clojure-1-11-0) added a
    couple of variants of success-or-nil functions.
[^3]: Should it be `[errors successful-value]`? See the problem? Try to find a
    way not to confuse yourself or your fellow programmers.
[^4]: Perhaps not. Perhaps returning them to where they are needed is the
    simplest (and most performant?) solution that works well enough in your
    project. You need to decide on the trade-offs.
[^5]: "Don't use exceptions for control flow" is an objection sometimes raised.
    Depending on what you consider *exceptional* it might be valid or not.
    However, exceptions are well understood on the JVM and might be the best
    no-additional-boilerplate GOTO statement that does the job correctly.

    There is also the issue of performance. If inside of a tight loop then
    throwing an exception might not be the best solution. Don't optimize too
    early, though. Verify if not throwing would give you much needed performance
    improvement.

    Finally, you might wonder why go through the pain of defining the error
    model if you end up throwing errors as exceptions anyway? The error model is
    here to have just one representation of errors, while the supporting
    functions are here to help you collect as many of them as possible. Throwing
    errors comes at a point when you cannot meaningfully deal with them in the
    code where you have them, but have to let some higher-up code know about
    them.
