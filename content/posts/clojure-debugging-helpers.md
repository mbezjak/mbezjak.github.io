---
title: "Clojure Debugging Helpers"
date: 2022-11-11T10:53:47+01:00
draft: false
---

I highly recommend [Repl Driven Development by Stuart
Halloway](https://vimeo.com/223309989) if you haven't watched it already. It's a
real eye opener when it comes to the topic of REPL and how to use it to your
advantage. Two sentence summary of that presentation could easily be:

> Don't type into REPL! Send things to the REPL!

As soon as you begin to understand that message, a whole new world opens up
where some things become so incredibly simple. For example, how to debug a
misbehaving [ring handler](https://github.com/ring-clojure/ring/wiki/Concepts)?

```clojure
(defn handler [request]
  ,,,)
```

Just make a binding that can be manipulated. Then reevaluate the handler (send
updated `defn` to the REPL).

```clojure
(defn handler [request]
  (def request request)
  ,,,)
```

When the new handler executes, `request` becomes available for prodding as a
top-level *var*. What can you do with it? Well, what do you need to investigate
the problem? Send more things to the REPL. Here are a couple of examples:

```clojure
(-> request :params :user-id)
(-> request :body :what :does :not :look :right?)
(get-in request [:headers "user-agent"])
(reitit-ring/get-match request)
(-> request (reitit-ring/get-match) :data (get (:request-method request)))
(db/get-user (-> request :params :user-id))
```

This technique is so general that it can be used almost everywhere.

- with `defn-`, `let`, `letfn`, `if-let`, `when-let`, `binding`, `with-redefs`,
  ...
- in `src` or `test`

It does get a bit tedious sometimes. E.g. when needing to litter the code
(temporarily) with a bunch of `def` expressions.

```clojure
(defn handler [{:keys [params body] :as request}]
  (def request request)
  (def params params)
  (def body body)
  (let [db (:db/admin request)
        user-id (:user-id params)
        user (db/get-user db user-id)]
    (def db db)
    (def user-id user-id)
    (def user user)
    (do-something user body)))
```

Wouldn't it be nice if I could specify that I want to add `def` expressions for
every binding without having to type (or
[expand](https://github.com/mbezjak/dotfiles/blob/main/emacs.d/snippets/clojure-mode/dd))
too much? How about something like the following?

```clojure
(defn/d handler [{:keys [params body] :as request}]
  (let/d [db (:db/admin request)
          user-id (:user-id params)
          user (db/get-user db user-id)]
    (do-something user body)))
```

Notice the variants `defn/d` and `let/d` instead of normal `clojure.core/defn`
and `clojure.core/let`. What would it take for this to work? Here is what I
ended up with and am quite happy using it.

The two helper namespaces are in the `repl` directory.

`repl/defn.clj`

```clojure
(ns defn)

(defn symbols [args]
  (mapcat
   (fn [a]
     (cond
       (map? a) (:keys a)
       (vector? a) (symbols a)
       :else [a]))
   args))

(defmacro d [fn-name & fdecl]
  (let [[args body] (if (string? (first fdecl))
                      [(second fdecl) (not-empty (drop 2 fdecl))]
                      [(first fdecl) (not-empty (rest fdecl))])
        defs (map (fn [s] `(def ~s ~s)) (symbols args))]
    `(defn ~fn-name ~args
       ~@defs
       ~@body)))

(comment
  (defn foo [])
  (defn foo [a b] (+ a b))
  (defn foo [{:keys [a b]}] (+ a b))

  (d foo "doc" [])
  (d foo [])
  (d foo [a b] (+ a b))
  (d foo [{:keys [a b]}] (+ a b))
  (d foo [& nums] (apply + nums))
  (d foo [[a b]] (+ a b))
  (d foo [[{:keys [a b]}]] (+ a b))
  (d foo [& {:keys [a b]}] (+ a b))
  (foo)
  (foo 1 2)
  (foo {:a 1 :b 2})
  (foo [1 2])
  (foo [{:a 1 :b 2}])
  (foo :a 1 :b 2))
```

`repl/let.clj`

```clojure
(ns let)

(defmacro d [bindings & body]
  (let [bindings-with-defs (->> bindings
                                (destructure)
                                (partition 2)
                                (mapcat (fn [[s code]]
                                          [s code
                                           (gensym "not-used") `(def ~s ~s)])))]
    `(let [~@bindings-with-defs]
       ~@body)))

(comment
  (let [a (+ 1 2)
        b (inc a)
        {:keys [c d]} {:c 1 :d 5}]
    (+ a b c d))

  (destructure '[a (+ 1 2)
                 b (inc a)
                 {:keys [c d]} {:c 1 :d 5}])

  (d [a (+ 1 2)
      b (inc a)
      {:keys [c e]} {:c 1 :e 5}]
     (+ a b c e))

  (d [a (+ 1 2)
      b (throw (ex-info "Can you see a?" {}))]
     :not-relevant))
```

For the helpers to be usable elsewhere in the project I made sure the following
was [evaluated every time I started the
project](https://github.com/mbezjak/dotfiles/blob/99acf8c8354a242a4035dd064122b1f1a531e787/emacs.d/lisp/my-functions.el#L254)
in the REPL:

```clojure
(load-file "repl/let.clj")
(load-file "repl/defn.clj")
```

That's it! Now I can use those helpers so I don't have to write a bunch of
`def` expressions anymore.

```clojure
(defn/d handler [{:keys [params body] :as request}]
  (let/d [db (:db/admin request)
          user-id (:user-id params)
          user (db/get-user db user-id)]
    (do-something user body)))
```

## Adapting the helpers to your own situation

- If you don't want to use `load-file`, you can put the helpers on the classpath
  and `require` them.
- If you don't have monorepo then you need to decide where it makes the most
  sense to put the helpers, but `load-file` might still be your best way of
  including the code in the project.
- If you don't like that the helpers are in a separate namespace then you can
  add them to `clojure.core` and use them like any other `clojure.core` fn.

  ```clojure
  (do
    (in-ns 'clojure.core)
    (defmacro defnd ,,,)
    (defmacro letd ,,,))
  ```

- The helpers above are not perfect, but are quite adequate. For example, I
  didn't have a strong need to create macros to cover `defn-`, `letfn`,
  `if-let`, `when-let`, `binding`, `with-redefs`, or `defn` with additional
  metadata. Those usages are rare enough that I can add `def` expressions
  manually. Feel free to develop your own helpers that match the situation
  you're in.
