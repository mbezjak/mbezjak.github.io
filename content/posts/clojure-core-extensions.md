---
title: "Clojure Core Extensions"
date: 2022-11-21T17:20:18+01:00
draft: false
---

There are plenty of high quality libraries that add *missing* functions from
`clojure.core`. Just take a look at [the clojure
toolbox](https://www.clojure-toolbox.com) under "Misc. Functions" or "String
Manipulation". Which ones are you using?

I tried many of them, but mostly don't use them in my projects. Instead, I
usually write my own `clojure.core` extensions. Here are some of the benefits as
I see them:

- IMO it's better [^1] to be thinking "what patterns do I see in the project"
  than "what can this library do for me".
- If you have your own `clojure.core` extensions, you've likely given it some
  thought how you'll structure them. If you didn't then it might be the case
  that you have a bunch of files with less than ideal names. You've probably
  seen them already. They are sometimes called `a_utils.clj`, `b_utils.clj`,
  `util.clj`, `common.clj`, `extra.clj`, `helper.clj`.
- "Where is that function defined?" Given `clojure.core`, 3 different libraries
  and your own *util* namespaces, where is it? This becomes somewhat easier if
  you've written your own extensions. If it's not in `clojure.core` then it's in
  a dedicated namespace you wrote.
- They are not hard to maintain because in my experience they don't change at
  all.
- You won't need to write a lot of new function. `clojure.core` is pretty
  impressive and is getting even
  [better](https://clojure.org/news/2022/03/22/clojure-1-11-0).
- They are fun to write and each one is a learning opportunity.

[^1]: That holds up to a point of course. `clojure.core` extensions are usually
    quite simple. Security related algorithms are another matter.

With that out of the way, I'd like to share the namespaces and functions that I
have so far. Feel free to use them and adapt to your situation. For tests see
https://gist.github.com/mbezjak/baa6622a6edfa40e61aeff27041266dc.

{{< gist mbezjak baa6622a6edfa40e61aeff27041266dc coll.clj >}}
{{< gist mbezjak baa6622a6edfa40e61aeff27041266dc map.clj >}}
{{< gist mbezjak baa6622a6edfa40e61aeff27041266dc number.clj >}}
{{< gist mbezjak baa6622a6edfa40e61aeff27041266dc kw.clj >}}
{{< gist mbezjak baa6622a6edfa40e61aeff27041266dc text.clj >}}
{{< gist mbezjak baa6622a6edfa40e61aeff27041266dc uuid.clj >}}
{{< gist mbezjak baa6622a6edfa40e61aeff27041266dc inout.clj >}}
{{< gist mbezjak baa6622a6edfa40e61aeff27041266dc pair.clj >}}
{{< gist mbezjak baa6622a6edfa40e61aeff27041266dc pairs.clj >}}
