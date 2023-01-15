---
title: "What Is a Validation?"
date: 2022-11-30T10:35:44+01:00
draft: false
---

> Validate this

> Write a validation for that

> Once validated, it should ...

What does it mean "to validate"? What we usually mean is to make sure that *only
valid data get accepted by the system*. For example, a "create user" endpoint
should not be able to receive a PDF invoice as input and create a valid user as
a result [^1].

In this post, let's focus on the steps involved to get to *valid data*. Can we
break down "validate" into constituent parts? Into simpler tasks? What is their
order?

To answer that, let's do a thought experiment. Suppose we have a web service
accepting a JSON [^2] `input` that does something useful with it. Before doing
the work, it hands off the `input` to generic validation function that will
validate `input` against some `rules`.

```clojure
(defn validate [input rules]
  ,,,)
```

The actual arguments, return value or implementation is of no concern here. It's
very likely that your project will have multiple functions doing the work
instead of just one, but that detail doesn't matter here. What we want to know
is: what can result in an error while attempting to validate `input` against
some amorphous `rules`?

## What can result in an error?

1. **Not consumable**. If `input` is an input stream then attempting to read the
   whole stream [^3] into a byte array can fail. E.g if a client disconnected
   because they lost their network connection.
1. **Encoding**. If `input` is a byte array then creating a string out of it can
   result in encoding problems. E.g. if the client sent
   [CP1250](https://en.wikipedia.org/wiki/Windows-1250), but the service is
   expecting UTF-8.
1. **Serialization format**. If `input` is a string then trying to decode it as
   JSON can fail. E.g. when a client sent XML instead of JSON. [^4]
1. **Shape**. Top-level form might not match the expected shape. E.g. if the
   client sent JSON array instead of an object.
1. **Required**. The client might not have sent required properties.
1. **Not nullable**. Required properties only mandate that they are present in a
   JSON object. They could have `null` value. That might not be acceptable.
1. **Strictness**. Properties unknown to the system might be rejected. [^5]
1. **Primitive value type**. The property type might not match the expected
   primitive type (boolean, number, string). E.g. the client sent `{"datetime":
   1669637798}`, a `number`, but the system is expecting a `string`.
1. **Basic constraints**. E.g. number should be greater than 0, string should
   not be blank, string must have at least 3 characters, a regex doesn't match.
1. **Coercion**. Attempt to convert a primitive JSON type into your programming
   language's type. E.g. convert to `UUID`, `Date`, `DateTime`, `YearMonth`.
1. **Descend**. At this point, top-level form was verified correctly. However,
   nested arrays or objects might not be correct. Descending deeper in the JSON
   structure might yield the same errors from *4. Shape* onward.
1. **Property relationships**. Now that each individual property in a JSON data
   structure makes sense (at any depth), make sure that the relationship between
   properties holds. For example:
   - Shipping information cannot be `null` if shipping type is `:via-post` [^6]
   - Product validity dates should satisfy `$from < $to`
   - Invoice total does not match the sum of all item amounts

1. **Basic DB constraints**. The ones that can be easily checked or even
   implemented generically. For example:
   - Country doesn't exist [^6]
   - Unknown product type
   - Customer with that email already exists

1. **Business constraints**. You know, the thing that makes the service be
   useful and not a simple API around the database. For example:
   - Not enough funds in the bank account for money transfer [^6]
   - Cannot schedule the meeting because the person is out of office at the
     time
   - Pregnancy tests are only available to our female customers
   - Order cannot be canceled because it's already shipped and on its way to you
   - That product has been recalled in the meantime due to safety concern
   - Cannot create a new reference to a note that has been deleted
   - Referenced file no longer exists in the system
   - Please load the currency rates for today before attempting to book an
     entry
   - Generating this PDF would result in more than 1000 pages. Please constrain
     your dataset
   - Cannot create a histogram chart from the dataset because it doesn't contain
     field with timestamp type
   - That flight has been cancelled in the meantime
   - In order to import from GMail you need to log into your Google Account and
     allow access to our *EasyImportApp*

The above is not meant to be complete. Depending on the problem you're facing,
you might increase the granularity even more or reorder the tasks. It can also
be simplified up to a point.

## Why think about it?

Ok, "to validate" might mean many things. So what?

We can use that to improve the system. For example, suppose the system is
responding with *"Product not found"*, but that doesn't make any sense to you.
So you start debugging, spend some time on it, and find the root cause. The
error was supposed to be *"Cannot convert product.id='abc' into UUID"*! Did the
service implementation went from 3. directly to 12. and skip everything in
between? To improve the system, simply add one or more of the tasks needed from
above.

Equally important is to think about the libraries that help out with the
validation. Sometimes a library is used for absolutely everything without
regards for what that library was designed to do. You know how it goes. *Oh I
see [malli](https://clojars.org/metosin/malli) can handle a large subset of the
tasks above, so let's also beat it into submission and make it handle business
constraints.* It's important to acknowledge their limits. It's unlikely that any
open source library will help you with your business constraints. You need to
make room in your project to handle them properly. More on that in the [next
blog post](../error-model).

## Appendix

If you happen to implement your system such that it recognizes all the relevant
subtasks above then a nice side benefit is that it's really easy to explore and
course-correct using such a system. Event without any documentation. For
example, you can keep executing `curl -X POST https://api.company.com/users -H
'...' -d '...'` and learn something new.

1. Oh I see I need to send something
1. Oh it needs to be a JSON
1. Oh a JSON object
1. Oh `email` and `dateOfBirth` are required
1. Oh `dateOfBirth` needs to be a string
1. Oh `email` needs to match a regex
1. Oh `dateOfBirth` needs to be in ISO8601 format
1. Does it accept time information? No!
1. Oh `john@example.com` already exists
1. ... and so on

If you've ever used a system that guided you towards a solution, you know how
lovely it is to use it. On the other hand, if you've done that on the service
side, you know that careful error handling usually corresponds with proportional
reduction in time consumed by debugging.

[^1]: Usually anyway. I don't know your project. :)
[^2]: We'll use JSON because it's so common. The thought experiment should work
    equally well with EDN, XML, protocol buffers, or any other serialization
    format.
[^3]: We won't concern ourselves with optimizations such as performance,
    latency, or memory requirements. Nor about authentication or authorization.
[^4]: Have you seen input that looks like JSON, but is not a valid JSON? One has
    to wonder how was that generated? By concatenating strings together?
[^5]: This might make sense if you control both the backend and client code and
    can easily control how they get updated. If you cannot control your clients
    then it probably makes more sense to skip this validation to make sure you
    adhere to "open for extension" principle.
[^6]: Error messages here aren't necessarily optimized for humans. Even for
    other services, they might not contain the relevant details that would be
    helpful for debugging (what country? what amount? which bank account?).
