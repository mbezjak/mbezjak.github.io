---
title: "What to Do With Evaluated REPL Expressions?"
date: 2024-05-20T12:31:45+02:00
draft: false
---
In Clojure, we've internalized, as Stuart Halloway said, to [send things to the REPL, not type into
the REPL](https://vimeo.com/223309989). The (sub)expressions we send to the REPL live in actual
files saved on disk, vs. being ephemeral and tied to a REPL connection. Sometimes, those
(sub)expressions come from existing project namespaces. Sometimes, those (sub)expressions don't fit
any particular namespace. In that case, Stu's suggestion is to simply append them to
`everything.clj` and evaluate from there. That works quite well! If you're not doing that, consider
starting.

Over the years of using an equivalent of `everything.clj`, I've observed a few things:
1. Certain (sub)expressions are reusable, or can be made reusable. Enough for me, but not
   necessarily enough for them to be included in `dev/user.clj`.
2. Sometimes, a (sub)expression I wrote a while back is almost (maybe with slight modification)
   exactly what I need now.
3. Coworkers might ask for how I do things. This leads to copy/pasting (sub)expressions to Slack,
   GitHub PRs, etc.

Based on above, I'd like to recommend an addition to Stu's suggestion.
> Commit those file(s) to project's git repository under your name [^1]

[^1]: Depending on if you use a monorepo or multirepo, this might mean multiple `everything.clj`
    files. In that case, `everything.clj` is not necessarily referring to a global "everything", but
    "everything" for that particular project.

For example:

```bash
$ tree repl/
repl/
└── miro
    ├── everything.clj
    └── reuse.clj
```

Every other developer, willing to partake, could put their stuff into `repl/$name` directory. This
might also be the place for those [debugging helpers](../clojure-debugging-helpers).

## Suggested rules for `repl/*`

1. Don't add `repl/*` to the classpath.
2. Don't review code in `repl/*`.
3. When changing `src`, there is no need to change `repl/*`. Especially someone else's REPL scripts!
4. There is no guarantee that the code in `repl/*` works.
5. Setup your editor/IDE to ignore searching text in `repl/*`.

The point of `everything.clj` is to send individual expressions to the REPL and not the whole file.
Adding those files to the classpath would send the wrong message. To use functions from `reuse.clj`
in `everything.clj` simply load the whole file with `clojure.core/load-file`.

The code in `repl/*` is mostly write-and-evaluate-once (with few exceptions). There is no need to
expand the effort into making those expressions work again after an incompatible change to `src`.
The effort to fix the expression should only be made when you (or someone else) have the **need** to
evaluate it again.

In fact, there is no guarantee that any particular expression will work. Some might; some might
require a tweak or two; some might be completely broken. The value of having them is that you don't
need to start from scratch. Feel free to delete the code that will obviously not be useful anymore.
Also feel free to share certain expressions with your coworkers. With the files committed to git,
you can simply give them the permalink of the expression.

Accept any change in the PR coming from `repl/*` as-is. It's not production code, it's not on the
classpath, and won't enter production in any way. It's safe to simply accept someone else's
experiments.

Finally, you might rely on certain tools or IDEs that search for a given text across all text files
in the project. Make sure to exclude `repl/*` files when searching to avoid superfluous lines.

## Appendix: an example

An example of how the top of `everything.clj` might look like. The top might be include something
you're using every now and again.

```clojure
(load-file "repl/miro/reuse.clj")

(reuse/start-unless-already-started)
(reuse/connect-flow-storm)

(reuse/start-quietly)
(reuse/restore-logging)

(reuse/stop-job-executors)
(reuse/run-job-now :integrity-check)

(reuse/run-all-migrations)
(reuse/execute-sql-across-all-dbs-and-schemas ["DELETE TABLE IF EXISTS recently_added"])
(reuse/simulate-time-passed 1-day)

(reuse/delete-all db)
(reuse/bootstrap-the-usually-things db)

(reuse/request-body-as-clojure "/tmp/captured.json")
```

The bottom might be a recent experiment. The code isn't meant to be production quality. Comments
added for the reader.

```clojure
;; What's the result?
(time-format/parse-local-date "2024-04-24T00:00:00.000Z")
(time-format/parse-local-date "2024-04-23T23:59:59.000Z")

;; Which formatters are available?
time-format/formatters
(time-format/show-formatters)

;; What's my local time zone?
(time/now)
(java.util.Date.)
(System/getProperty "user.timezone")
(java.util.TimeZone/getDefault)

;; Which formatter is the first that parses successfully?
(doseq [formatter-name (map #(get (set/map-invert time-format/formatters) %)
                            (vals time-format/formatters))]
  (try
    (println "Succeeded" formatter-name
             (time-format/parse-local-date (get time-format/formatters formatter-name)
                                           "2024-04-24T00:00:00.000Z"))
    (catch Exception e
      (println "Failed " formatter-name))))
```
