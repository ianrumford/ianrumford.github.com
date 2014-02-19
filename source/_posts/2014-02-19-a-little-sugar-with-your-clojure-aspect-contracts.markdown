---
title: "A little sugar with your Clojure Aspect Contracts"
date: 2014-02-19
layout: post
categories: clojure aspect contract sugar
published: true
comments: getting started with clojure-contracts-sugar
---

> TL;DR: clojure-contracts-sugar - some sugar  macros for clojure.core.contracts

# Introduction

Back in November 2012, in one of my first toe-dippings into
[Clojure][ClojureHome], I wrote a
[post][RumfordBlogClojureCoreContractsFirstTake] on my initial experiences
with [Michael Fogus's][FogusHome]
[clojure.core.contracts][ClojureCoreContractsGithub] (hereafter just
**CCC**).

The rest of this post assumes you have a passing familiarity with
**CCC**. If you aren't, you may find my 
[original post][RumfordBlogClojureCoreContractsFirstTake] a digestible introduction.

Although using **CCC** for
_[contracts programming][DesignByContractWikipedia]_ is self-evident,
it was the opportunity to use the [Clojure's][ClojureHome]
[:pre and :post assertions][FogusBlogClojurePreandPost] to support the
concept and potential of _aspects_ as defined originally by
[Gregor Kiczales][KiczalesHome] in his work on
[Aspect Oriented Programming][AOPWikipedia] (**AOP**).

In this post I'm following through the experiments and ideas of the
[original post][RumfordBlogClojureCoreContractsFirstTake], and
illustrating with some examples using a new library
I've written recently - [clojure-contracts-sugar][ClojureContractsSugarGithub] (**CHUGAR**).

In the [original post][RumfordBlogClojureCoreContractsFirstTake] I made the point that **CCC** could do with some
productivity aids - _sugar_ - to ease the rich usage of **CCC**. **CHUGAR**
attempts to supply that _sugar_ and you can think of this post as
partly a _getting started_ tutorial for the library.

In the simplest cases, **CHUGAR** reduces to **CCC** albeit with a slightly
more flexible syntax. If your contracts needs are straightforward, you're
likely better off using **CCC** or even [:pre and :post assertions][Fogusblogclojurepreandpost] 
directly.

That said, I would encourage you though to have a look at the sections
on _mnemonics_ below which offer an easy way to use (and re-use) rich
and flexible aspect contracts.

Note, the usual caveat,  **CHUGAR**  is a work in progress and really is
intended to be only a starting point (foundation) for  explorations of
contracts from [Clojure][ClojureHome].  The API is "settling" but may change.

> A quick note on terminology: in the [original
> post][RumfordBlogClojureCoreContractsFirstTake] I used **suck** to identify
> the (input) arguments of a function and **spit** the result (return value). That
> "convention" is used extensively both here and in the library's code.

<!-- more -->

# Aspect Oriented Programming

I first bumped into **AOP** reading the **Communications of the ACM's**
[October 2001 edition][CACMOct2001] special edition on AOP (paywall
for content). In their [introductory article][CACMArticleAOPIntro] in
that volume, _Elrad, Filman and Bader_ explains the need for AOP like
so:

> Any structural realization of a system will find that some concerns
> are neatly localized within a specific structural piece, while others
> cross multiple elements. AOP is focused on mechanisms for simplifying
> the realization of such crosscutting concerns.

and

> Separating the expressions of multiple concerns in programming
> systems promises simpler system evolution, more comprehensible
> systems, adaptability, customizability, and easier reuse. 

They go on to highlight the opportunity for a services paradigm for
common requirements such as authentication, logging, etc facilitating
clear separation of concerns: Leaving the application developers to
focus of their agenda whilst minimising the potential for "misuse",
whether by omission or commission, of the specialist subsystems e.g. authentication
which may be  supported by  other, third,  parties:

> Implicit invocation is a virtue in this age of increased software
> complexity, as domain experts for an application are unlikely to be
> familiar with intricacies of specialized algorithms for distribution,
> authentication, access control, synchronization, encryption,
> redundancy, and so forth, and cannot be trusted to always invoke them
> appropriately in their programs.

In [Ramnivas Laddad's][LaddadTwitter]  [book on AspectJ][LaddadBookAspectJinAction], he says:

> AOP is a new methodology that provides separation of crosscutting concerns
> by introducing a new unit of modularization—an aspect—that crosscuts other
> modules. With AOP you implement crosscutting concerns in aspects instead of
> fusing them in the core modules.

> The result is that AOP modularizes the cross-cutting concerns in a
> clear-cut fashion, yielding a system architecture that is easier to
> design, implement, and maintain.

In another take, in [Emerick][EmerickHome], [Carper][CarperHome] and [Grand's][GrandHome]  book [Clojure Programming][ClojureProgrammingBook], they say:

> Aspect-oriented programming (AOP) is a methodology that allows separation of cross-cutting
> concerns. In object-oriented code, a behavior or process is often repeated in
> multiple classes, or spread across multiple methods. AOP is a way to abstract this
> behavior and apply it to classes and methods without using inheritance.

# The Code

## Jar is on Clojars

The jar is on [Clojars][ClojarsClojureContractsSugar]:

[Leiningen][LeiningenHome] dependency information:

```clojure
[name.rumford/clojure-contracts-sugar "0.1.0"]
```

[Maven][MavenHome] dependency information:

```xml
<dependency>
  <groupId>name.rumford</groupId>
  <artifactId>clojure-contracts-sugar</artifactId>
  <version>0.1.0</version>
</dependency>
```

## Repo is on Github

The [repo][ClojureContractsSugarGithub] is  on [github][ClojureContractsSugarGithub].
As is common with Clojure code bases, its organised as a [Leiningen][LeiningenHome]
project so you'll need Leiningen [installed][LeiningenGithub] to work.

The project structure is Maven style but there is only Clojure today:
_./src/main/clojure_ and _./src/test/clojure_.

The code uses another of my other new libraries
[clojure-carp][ClojureCarpGithub] for some utility functions,  exceptions, diagnostics and
other miscellany.

## Overview

The repo's _./doc_ folder contains the source of this post: it is an
[emacs][emacshome] [org][orgmodehome] file
[tangled][orgmodemanualextractsourcecode] to generate the examples below
in a [Leiningen][LeiningenHome] project.

It also contains an (org and html) file _code-notes_ offering a brief
high-level overview.

> NEED TO WORTK ON THE CODE OVERVIEW SOME !!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!

## Tests

There are a number of tests providing reasonable code coverage that can be run from the repo:

```bash
lein test aspect-tests1
```

## Examples

The examples below can be found in the repo's examples folder
(specifically in _./examples/aspect\_examples_) and they can be run using
_lein_ in the usual way:

```bash
cd ./examples/aspect-examples
lein deps
lein run -m aspect-examples1
```

The examples use a couple of harness functions - _will-work_ and
_will-fail_ - to run tests.

_will-work_ takes as arguments the constrained
function and a list of arguments. 

_will-fail_ similarly takes just the constrained function and its arguments and
catches the **AssertionError** expected to be thrown.

```clojure
;; Helper for accessor examples expected to work.  Returns the expected result, else fails

(defn will-work
  [fn-constrained & fn-args]
  (let [actual-result (apply fn-constrained fn-args)]
    (println "will-work" "worked as expected" "actual-result" actual-result "fn-constrained" fn-constrained "fn-args" fn-args)
    actual-result))

;; Helper for accessor examples expected to fail.  Catches the expected AssertionError, else fails.
;; A nil return from the function is ok

(defn will-fail
  [fn-constrained & fn-args]
  (try
    (do
      (let [return-value (apply fn-constrained fn-args)]
        (if return-value (assert (println "will-fail" "DID NOT FAIL" "did not cause AssertionError" "fn-constrained" fn-constrained "fn-args" fn-args "RETURN-VALUE" (class return-value) return-value)))))
    (catch AssertionError e
      (println "will-fail" "failed as expected" "fn-constrained" fn-constrained "fn-args" fn-args))))
```

# Using Contract Aspects - Apply v Update

The libary has two main aspect contract  macros: _apply-contract-aspects_ and
_update-contract-aspects_.  

The majority of examples below  use _apply-contract-aspects_ but _update-contract-aspects_ could be used just as well.

A couple of very simple examples follow to give a _flavour_ of their usage and details will be expanded upon in the following sections.

## Using apply-contract-aspects

The first macro, _apply-contract-aspects_, applies one or more aspects to an
existing function and returns a **new** function.  

### Example - applying a built-in predicate

The below will create, from the original function
_any-fn_, a new constrained function _map-fn_ that will **only** suck a
map as its input argument.  (The return value will be unconstrained.)

```clojure
;; Example - applying a built-in predicate

;; any-fn is the "base" function

(defn any-fn [x] x)

;; map-fn is the new function constrained to suck a map

(def suck-map-fn1 (apply-contract-aspects any-fn map?))

;; This will work

(will-work suck-map-fn1 {:a 1 :b 2 :c 3})

;; But this will fail since suck-map-fn1 can only suck a map

(will-fail suck-map-fn1 [1 2 3])

;; The original function any-fn is unchanged and not constrained in any way

(will-work any-fn {:a 1 :b 2 :c 3})
(will-work any-fn [1 2 3])
(will-work any-fn :a)
(will-work any-fn 99)
```

> The map? predicate in the above call to _apply-contract-aspects_ is  the **Contract Definition**.

Under the covers, _apply-contract-aspects_ generates a **CCC** contract
similar to the  below
where the _ctx-aspect2721_ is the random, but unique, name (gensym) of
the contract function.

```clojure
;; Example - example of the generated clojure.core.contract call
(clojure.core.contracts/contract ctx-aspect2721 "\"ctx-aspect2721\"" [arg0] [(map? arg0)])
```

> Quick note on argument names: the arguments in a generated contract
> are given names _arg0_, _arg1_, etc. These names can be used to
>  refer explicitly to specific arguments. More on this later.

Similarly, to suck a vector:

```clojure
;; Example - suck a vector

(def suck-vector-fn1 (apply-contract-aspects (fn [x] x) vector?))

(will-work suck-vector-fn1 [1 2 3])

(will-fail suck-vector-fn1 99)
```

> Built-in mnemonics provide a simple way of applying the same  assertion to both the input argument and return value - see later.

### Example - applying your own custom predicate

You can of course create and  use your own **custom** predicate function, returning true or false as
decided.  You can constrain multiple input arguments and/or the return
value in a custom predicate.

A simple way to create a custom predicate would be to use [:pre and post assertions][FogusBlogClojurePreandPost] 
in an identity function.

```clojure
;; Example - applying your own custom predicate

;; The custom predicate ensures the argument is a map, its keys are keywords and values are numbers.

(defn is-map-with-keyword-keys-and-numeric-values?
  [x]
  {:pre [(map? x) (every? keyword? (keys x)) (every? number? (vals x))]}
  x)

(def map-keyword-keys-numeric-values-fn1 (apply-contract-aspects any-fn is-map-with-keyword-keys-and-numeric-values?))

;; This will work

(will-work map-keyword-keys-numeric-values-fn1 {:a 1 :b 2 :c 3})

;; But these will fail the contracts

(will-fail map-keyword-keys-numeric-values-fn1 {:a :x :b 2 :c 3})
(will-fail map-keyword-keys-numeric-values-fn1 {"x" 1 :b 2 :c 3})
(will-fail map-keyword-keys-numeric-values-fn1 [1 2 3])

;; As before the original function any-fn is unchanged and not constrained in any way

(will-work any-fn {:a 1 :b 2 :c 3})
(will-work any-fn [1 2 3])
(will-work any-fn :a)
(will-work any-fn 99)
```

## Using update-contract-aspects

The second macro, _update-contract-aspect_, "changes" (using
_alter-var-root_) an existing function.  

### Example - updating a function with a built-in predicate

Essentially the same example as above except the source function but _any-fn_ is "changed" to **only** suck a map.

```clojure
;; Example - updating a function with a built-in predicate

;; any-fn is "changed" to now only suck a map

(update-contract-aspects any-fn map?)

;; This will work

(will-work any-fn {:a 1 :b 2 :c 3})

;; But this will fail as any-fn can now only suck a map

(will-fail any-fn [1 2 3])
```

# Applying Contracts to Many Arguments and the Result

Many functions will have more than one (suck) argument, 
even different arities, each
likely requiring its own specific _assertions_ (constraints), and the (spit) result
maybe different assertion(s) again.

To support a rich definition of the assertions required by each argument and the return value, 
the contract definition can  be specified as a map with two keys: _:suck_
and _:spit_ where the value of the keys are the assertions to apply to
the input arguments and return values. An example should clarify.

## Example - suck a map and keyword and spit a vector

The below defines a two argument contract: the first argument
must be a map, the second a keyword; with a vector expected as the
result:

```clojure
{:suck [map? keyword?] :spit vector?}
```

```clojure
;; Example - suck a map and keyword and spit a vector

;; In this example, the assertion constrains the function to suck a map and keyword
;; and spit a vector.  

;; The function looks up the value of the keyword in the map.

(def suck-map-keyword-spit-vector-fn1 (apply-contract-aspects (fn [m k] (k m)) {:suck [map? keyword?] :spit vector?}))

;; This will work as key :c contains a vector

(will-work suck-map-keyword-spit-vector-fn1 {:a 1 :b 2 :c [1 2 3]} :c)

;; But these will fail

(will-fail suck-map-keyword-spit-vector-fn1 {:a 1 :b 2 :c 3} :c)
(will-fail suck-map-keyword-spit-vector-fn1 {:a 1 :b 2 :c 3} :d)
```

Some notes:

-   assertions are matched positionally to their arguments

The _map?_ constrains **only** the first argument (arg0) and the
_keyword?_ constrains **only** the second argument (arg1); and the returned value must be a _vector?_.

-   if there is only one argument, the enclosing vector is not needed

Just as the return value can be specified as just _vector?_ and not
_[vector?]_, if the function only sucked a map _:suck map?_ would be sufficient e.g. _{:suck map? :spit vector?}_.

## Example - suck a map - with keyword keys and numeric values - and keyword and spit a vector

To include additional assertions on the map in the previous example to
insist on keyword keys and numeric values, the assertion for the map
argument would be changed to a vector of constraints.  

Note the use of
_arg0_ to refer to the input map in the _every?_ clauses.

```clojure
{:suck [[map? (every? keyword? (keys arg0)) (every? number? (vals arg0))] keyword?] :spit vector?}
```

```clojure
;; Example - suck a map - with keyword keys and numeric values - and keyword and spit a vector

;; In this example, the contract constrains the function to suck a map and keyword, spit a number.

;; The map must have keywords keys and numeric values.

(def suck-map-keyword-spit-number-fn1 (apply-contract-aspects (fn [m k] (k m)) {:suck [[map? (every? keyword? (keys arg0)) (every? number? (vals arg0))] keyword?] :spit number?}))

;; This will work

(will-work suck-map-keyword-spit-number-fn1 {:a 1 :b 2 :c 3} :a)

;; But these will fail their contracts

(will-fail suck-map-keyword-spit-number-fn1 {:a :x :b 2 :c 3} :a)
(will-fail suck-map-keyword-spit-number-fn1 {:a 1 :b 2 :c 3} :d)
(will-fail suck-map-keyword-spit-number-fn1 {"x" 1 :b 2 :c 3} :c)
```

## Example - specifying argument order explicitly

Specifying the arguments' order implicitly by their position in the suck assertion list is
natural but there may be times when you want to explicitly define the
argument position and its assertions, irrespective of its position in the
assertion list.

You can do this by providing a map where the keys are the argument
positions and the values the assertion list to apply to that argument.

The example below is a variant of the map and keyword example above but the keyword
is the first argument (key 0) and the map the second (key 1). The map
must have  keyword keys and
numeric values.

> Note the use of _arg0_ to refer to the input map in the _every?_
> clauses **even though** the map is the second argument (and will
> therefore be _arg1_ in the contract).
> 
> That's because the _every?_ forms will be rewritten **automatically** to
> reflect the map's argument position i.e. its _arg1_. The point is that the
> map assertion list does not change no matter where the map appears in
> the argument order.
> 
> This is similar to when mnemonics are composed - see later.

```clojure
{:suck {0 :keyword 1 [:map (every? keyword? (keys arg0)) (every? number? (vals arg0))]} :spit :number}
```

```clojure
;; Example - specifying argument order explicitly

;; In this example, the arguments are specified by their explicit position in the argument order

(def explicit-argument-order-fn1 (apply-contract-aspects (fn [k m] (k m)) {:suck {0 :keyword 1 [:map (every? keyword? (keys arg0)) (every? number? (vals arg0))]} :spit :number}))

;; This will work

(will-work explicit-argument-order-fn1 :a {:a 1 :b 2 :c 3})

;; But these will fail their contracts

(will-fail explicit-argument-order-fn1 :a {:a :x :b 2 :c 3})
(will-fail explicit-argument-order-fn1 :d {:a 1 :b 2 :c 3})
(will-fail explicit-argument-order-fn1 :c {"x" 1 :b 2 :c 3})
```

BTW The contract looks like this:

```clojure
(clojure.core.contracts/contract ctx-aspect3000 "\"ctx-aspect3000\"" [arg0 arg1] [(keyword? arg0) (map? arg1) (every? keyword? (keys arg1)) (every? number? (vals arg1)) => (number? %)])
```

# Using CCC's contract definition form

For those familiar with **CCC**, you can also use **CCC's** contract specification format as well.
But note the signature vector (e.g. '[v]) and assertion vector (e.g.
'[map?]) must be inside a third  vector:

```clojure
[[v] [map?]]
```

## Example - Using CCC's format to suck a map and spit a vector

The assertion vector can have any assertions supported by **CCC**.  For example, here the constrained function
below sucks a map and spits a vector:

```clojure
;; Example - suck map and spit vector using CCC form

(def suck-map-spit-vector-fn1 (apply-contract-aspects (fn [m] (:c m)) [[v] [map? => vector?]]))

(will-work suck-map-spit-vector-fn1 {:a 1 :b 2 :c [1 2 3]})

(will-fail suck-map-spit-vector-fn1 {:a 1 :b 2 :c 1})
```

## Example - Using CCC's format to suck a map with keyword keys, and spit a vector

Or, additionally, to ensure the map's keys are all keywords:

```clojure
;; Example - suck map, spit vector but also all map keys are keywords

(def suck-map-keyword-keys-fn1 (apply-contract-aspects (fn [m] (:c m)) [[v] [map? (every? keyword? (keys v)) => vector?]]))

(will-work suck-map-keyword-keys-fn1 {:a 1 :b 2 :c [1 2 3]})

(will-fail suck-map-keyword-keys-fn1 {"x" 1 :b 2 :c 1})
```

The example below will fail becuase the keys of _test-map2_ are not keywords:

```clojure
;; Example - this will fail as test-map2's keys are not keywords

;;(suck-map-keyword-keys-fn1 test-map2)
```

## Example - using CCC's format with a rich assertions

**CCC**  supports the specification of rich
assertions. For a two argument function (map, keyword), where the map's
keys are keywords, the values numbers; and the return value
unconstrained, in CCC's format, the full contract would look like this:

```clojure
[[m k] [(map? m) (every? keyword (keys m)) (every? number? (vals m)) (keyword? k)]]
```

An example:

```clojure
;; Example - using CCC's format to specify multiple assertions

;; In this example, the assertion constrains the function to suck a map,
;; with keywords keys and numeric values, and a keyword.

;; The returned value is unconstrained

(def map-keyword-keys-numeric-vals-fn2 (apply-contract-aspects (fn [m k] (k m)) [[m k] [(map? m) (every? keyword (keys m)) (every? number? (vals m)) (keyword? k)]]))

;; This will work and return nil as the return value is not constrained

(will-work map-keyword-keys-numeric-vals-fn2 {:a 1 :b 2 :c 3} :d)

(will-fail map-keyword-keys-numeric-vals-fn2 {:a 1 :b 2 :c 3} "d")
(will-fail map-keyword-keys-numeric-vals-fn2 {:a :x :b 2 :c 3} :a)
(will-fail map-keyword-keys-numeric-vals-fn2 {"x" 1 :b 2 :c 3} :d)
```

## Example - using CCC's format in a suck definition

You can also use a **CCC** form in a suck definition. Likely confusing,
 notably because you have to be quite careful as to what assertions are
 applied to which arguments, but it works. The **CCC** form works as if
 it is a mnemonic (see later) in the same position.

Note in the example below the _map?_ assertion for the result in the
**CCC** form has been discarded because it is not a _suck_ assertion;
the _spit_ _:number_ assertion is applied to the result.

```clojure
;; Example - using CCC's format in a suck definition

;; Not the clearest way of specifying the contract

(def using-ccc-form-in-the-suck-definition-fn1 (apply-contract-aspects (fn [m k s] (k m)) {:suck [:map [[k s] [(keyword? k) (string? s) => map?]]] :spit :number} ))

(will-work using-ccc-form-in-the-suck-definition-fn1 {:a 1 :b 2 :c 3} :a "s2")
(will-fail using-ccc-form-in-the-suck-definition-fn1 {:a 1 :b 2 :c 3} "d" "s2")
(will-fail using-ccc-form-in-the-suck-definition-fn1 {:a :x :b 2 :c 3} :a 1 )
(will-fail using-ccc-form-in-the-suck-definition-fn1 {"x" 1 :b 2 :c 3} :d "s2")
```

# Using Mnemonics

At their simplest, **mnemonic** are (Clojure) keyword "short-hands" for a contract assertion(s).

## Using Mnemonics for Built-in Predicates

So far the assertions used have used Clojure's built-in predicates such as _map?_,
_number?_ and _vector?_ but we could have used their keyword mnemonics
_:map_, _:number_ or _:vector_.  In fact any predicate of the form
_name?_ can be replaced by its keyword form _:name_ (as long as the
symbol can be **resolved**).

### Example - using a built-in mnemonic

To repeat the example above using _map?_ but with _:map_:

> Note: using a built-in mnemonic as the full contract definition will apply the assertion(s) to both the input argument and also return value.

```clojure
;; Example - using a built-in mnemonic

;; This is a contrived example to show the symmetry when using a buit-in mnemonic.
;; BTW The function hard-codes a map as it return value so will always satisfy the spit constraint.

(def mnemonic-suck-and-spit-map-fn1 (apply-contract-aspects (fn [x] {:x 1 :y 2 :z 3}) :map))

;; This will work because the argument is a map and the (hard-coded) return value is a map

(will-work mnemonic-suck-and-spit-map-fn1 {:a 1 :b 2 :c 3})

;; But this fail sicne the argument is not a map

(will-fail mnemonic-suck-and-spit-map-fn1 [1 2 3])
```

### Example - applying built-in mnemonics to individual arguments and the result

Repeating one of the examples above sucking a map and keyword and
returning a vector, all that has changed is the
assertions now  use keywords.

> Note: built-in mnemonics in the map form of a contract
> definition apply the assertion only to the mnemonic's corresponding
> argument.

```clojure
;; Example - applying built-in mnemonics to individual arguments and the result

;; In this example, built-in mnemonics are used to constrains the
;; function to suck a map and keyword and spit a vector.

(def suck-map-keyword-spit-vector-fn1 (apply-contract-aspects (fn [m k] (k m)) {:suck [:map :keyword] :spit :vector}))

;; This will work as key :c contains a vector

(will-work suck-map-keyword-spit-vector-fn1 {:a 1 :b 2 :c [1 2 3]} :c)

;; But these will fail their contract

(will-fail suck-map-keyword-spit-vector-fn1 {:a 1 :b 2 :c 3} :c)
(will-fail suck-map-keyword-spit-vector-fn1 {:a 1 :b 2 :c 3} :d)
```

## Changing a Built-in Mnemonic Contract Definition

Replacing a built-in predicate with its keyword mnemonic is not a big win,
just saving a few characters in the assertion definition. 

The real power of
mnemonics comes from the opportunity to change the definition of an
existing mnemonic (or add custom ones - see later).

The  _configure-contracts-store_ macro manages mnemonics definitions.

### Example - redefining the :map built-in mnemonic

Say you wanted to re-define the built-in _:map_ mnemonic to check  **always** that a map's keys are keywords:

```clojure
;; Changing a Built-in Mnemonic Contract Definition

;; Change the built-in :map mnemonics to also check the keys are keywords

(configure-contracts-store aspect-mnemonic-definitions {:map {:suck [[map? (every? keyword? (keys arg0))]]}}) 
```

Using the updated mnemonic is exactly the same as before:

```clojure
;; Example - re-defining the :map built-in mnemonic

;; In this example, the :map built-in mnemonic has been changed to check the keys are keywords.

(def suck-map-keyword-spit-vector-fn1 (apply-contract-aspects (fn [m k] (k m)) {:suck [:map :keyword] :spit :vector}))

;; This will work as key :c contains a vector

(will-work suck-map-keyword-spit-vector-fn1 {:a 1 :b 2 :c [1 2 3]} :c)

;; But this will fail the contract as "x" is not a keyword.

(will-fail suck-map-keyword-spit-vector-fn1 {"x" 1 :b 2 :c 3} :c)
```

## Adding and Using Custom Mnemonics

Just as you can update the definition of a built-in mnemonic, you can
add / update your own **custom** mnemonics.

### Example - using a custom mnemonic

Say you wanted to define a custom mnemonic that "packages" the assertions
that a map's keys are keywords and all the values are numeric:

```clojure
;; Example - add a new mnemonic to the contracts store

;; The new mnemonic - :map-keyword-keys-numeric-vals - constrains an
;; argument to be a map with keyword keys and numeric values.

(configure-contracts-store
 aspect-mnemonic-definitions
 {:map-keyword-keys-numeric-vals {:suck [[map? (every? keyword? (keys arg0)) (every? number? (vals arg0))]]}}) 
```

To use the new mnemonic is straightforward.  Note the mnemonic appears
as the first value in the _:suck_ assertion vector, the other entry
being _:keyword_.

```clojure
;; Example - using a custom mnemonic

;; In this example, the assertion constrains the function to suck a map and keyword, spit a number.

;; The map must have keywords keys and numeric values.

(def mnemonic-suck-map-keyword-spit-number-fn1 (apply-contract-aspects (fn [m k] (k m)) {:suck [:map-keyword-keys-numeric-vals :keyword] :spit :number}))

;; This will work

(will-work mnemonic-suck-map-keyword-spit-number-fn1 {:a 1 :b 2 :c 3} :a)

;; But these will fail their contracts

(will-fail mnemonic-suck-map-keyword-spit-number-fn1 {:a :x :b 2 :c 3} :a)
(will-fail mnemonic-suck-map-keyword-spit-number-fn1 {:a 1 :b 2 :c 3} :d)
(will-fail mnemonic-suck-map-keyword-spit-number-fn1 {"x" 1 :b 2 :c 3} :c)
```

## Using a Custom Mnemonic to package multiple arguments

You can go a step farther from the previous example and add the assertion for the second
argument to be a keyword into the mnemonic as well:

```clojure
;; Using a Custom Mnemonic to package multiple arguments

;; The new mnemonic combines the assertions to ensure the first argument
;; is a map with keyword keys and numerics value and also the requirement
;; for the second argument to be a keyword.

(configure-contracts-store aspect-mnemonic-definitions {:suck-map-keyword-keys-numeric-vals-and-keyword {:suck [[map? (every? keyword? (keys arg0)) (every? number? (vals arg0))] keyword?]}}) 
```

### Example - using a custom multiple argument suck mnemonic

In this example a multiple argument mnemonic replaces the whole _:suck_ definition.

```clojure
;; Example - using a custom multiple argument suck mnemonic

;; In this example, the map assertion uses a mnemonic to ensure keywords keys and numeric values.

(def mnemonic-suck-map-keyword-spit-number-fn2 (apply-contract-aspects (fn [m k] (k m)) {:suck :suck-map-keyword-keys-numeric-vals-and-keyword :spit :number}))

;; Using the same tests as above

(will-work mnemonic-suck-map-keyword-spit-number-fn2 {:a 1 :b 2 :c 3} :a)
(will-fail mnemonic-suck-map-keyword-spit-number-fn2 {:a :x :b 2 :c 3} :a)
(will-fail mnemonic-suck-map-keyword-spit-number-fn2 {:a 1 :b 2 :c 3} :d)
(will-fail mnemonic-suck-map-keyword-spit-number-fn2 {"x" 1 :b 2 :c 3} :c)
```

## Using a Custom Mnemonic to package the complete contract

Its just a small step from the multi argument example to packaging
the whole contract in a custom mnemonic:

```clojure
;; Using a Custom Mnemonic to package the complete contract

;; The custom mnemonic combines the assertions to ensure the first
;; argument is a map with keyword keys and numerics value and also the
;; requirement for the second argument to be a keywork. It also includes
;; the requirement for the return value to be a number.

(configure-contracts-store
 aspect-mnemonic-definitions
 {:contract-suck-map-keyword-keys-numeric-vals-and-keyword-spit-number 
  {:suck [[map? (every? keyword? (keys arg0)) (every? number? (vals arg0))] keyword?] :spit :number}}) 
```

### Example - using a custom mnemonic to package the whole contract

In this example the complete contract mnemonic replaces the whole
contract map form.

```clojure
;; Example - using a custom mnemonic to package the whole contract

;; In this example, the a mnemonic packages the complete assertion

(def mnemonic-suck-map-keyword-spit-number-fn3 
  (apply-contract-aspects (fn [m k] (k m)) :contract-suck-map-keyword-keys-numeric-vals-and-keyword-spit-number))

;; Exactly the same tests as above

(will-work mnemonic-suck-map-keyword-spit-number-fn3 {:a 1 :b 2 :c 3} :a)
(will-fail mnemonic-suck-map-keyword-spit-number-fn3 {:a :x :b 2 :c 3} :a)
(will-fail mnemonic-suck-map-keyword-spit-number-fn3 {:a 1 :b 2 :c 3} :d)
(will-fail mnemonic-suck-map-keyword-spit-number-fn3 {"x" 1 :b 2 :c 3} :c)
```

## Using Mnemonics in Custom Mnemonics

You can use mnemonics in the **composition**  of other, richer mnemonics (although
beware the infinite recursion gotcha mentioned below).

For example, create a custom mnemonic - _:suck-map-special_ - to constrain a map to have
keyword keys and numeric values, and use that mnemonic in another
mnemonic - _:suck-map-special-and-keyword_ - to include the keyword as the second argument. And finally use the
second mnemonic to specify the full contract for a two argument function sucking
the constrained map and a
keyword, and also spitting a number - _:contract-suck-map-special-and-keyword-spit-number_.

```clojure
;; Using Mnemonics in Custom Mnemeonics

;; The first customer mnemonic constrains a map to have keyword keys and numeric values.

;; The second custome mnemonic speficiy the constrained map and a keyword as the second argument.

;; The third custom mnemonic uses the second mnemonic to build a
;; complete contract mnemonic for a two argument function sucking the
;; constrained map and a keyword, and spitting a number.

(configure-contracts-store
 aspect-mnemonic-definitions
 {:suck-map-special {:suck [[map? (every? keyword? (keys arg0)) (every? number? (vals arg0))]]}
  :suck-map-special-and-keyword {:suck [:suck-map-special :keyword]}
  :contract-suck-map-special-and-keyword-spit-number {:suck :suck-map-special-and-keyword :spit :number}}) 
```

### Example - using a mnemonic containing mnemonics

The example is exactly the same as the one above, but the use of "sub"
mnemonics is transparent.

```clojure
;; Example - using a mnemonic containing mnemonics

;; In this example, the three level mnemonic packages the complete assertion

(def mnemonic-suck-map-special-keyword-spit-number-fn1 (apply-contract-aspects (fn [m k] (k m)) :contract-suck-map-special-and-keyword-spit-number ))

;; Exactly the same tests as above

(will-work mnemonic-suck-map-special-keyword-spit-number-fn1 {:a 1 :b 2 :c 3} :a)
(will-fail mnemonic-suck-map-special-keyword-spit-number-fn1 {:a :x :b 2 :c 3} :a)
(will-fail mnemonic-suck-map-special-keyword-spit-number-fn1 {:a 1 :b 2 :c 3} :d)
(will-fail mnemonic-suck-map-special-keyword-spit-number-fn1 {"x" 1 :b 2 :c 3} :c)
```

## Composing Mnemonics - resolving arguments

In the examples above, mnemonics were always the first entry
in the value of a suck or spit key - see the three level composed
mnemonic immediately above.

Most the time the assertion (e.g. _:map_) did **not** need to include (specify) the
name (symbol)
of the argument the assertion would be applied to; the name was
deduced from the assertion's position in the value of the suck / spit key.

The only time an explicit argument name  appeared was  _arg0_
in the _every?_ assertion clauses because the map was the first
argument.  

But what if the map was not the first argument?

Lets recast the _:suck-map-special-and-keyword_
mnemonic to expect the _:keyword_ first and  the _:map-special_ second **but continue to use**
the _:suck-map-special_ mnemonic even though the latter expects (and
defines) the map
to be _arg0_:

```clojure
(configure-contracts-store
 aspect-mnemonic-definitions
 {:suck-keyword-and-map-special {:suck [:keyword :suck-map-special]}
  :contract-suck-keyword-and-map-special-spit-number {:suck :suck-keyword-and-map-special :spit :number}}) 
```

### Example - swapping the keyword and map in the three level composed mnemonics

An example using the swapped argument third level mnemonic _:contract-suck-keyword-and-map-special-spit-number_

```clojure
;; Example - swapping the keyword and map in the three level composed mnemonics

;; In this example, the keyword and map are swapped in the three level mnemonic

(def mnemonic-suck-keyword-map-special-spit-number-fn1 (apply-contract-aspects (fn [k m] (k m)) :contract-suck-keyword-and-map-special-spit-number ))

;; The same tests as above but the arguments swapped

(will-work mnemonic-suck-keyword-map-special-spit-number-fn1 :a {:a 1 :b 2 :c 3})
(will-fail mnemonic-suck-keyword-map-special-spit-number-fn1 :a {:a :x :b 2 :c 3})
(will-fail mnemonic-suck-keyword-map-special-spit-number-fn1 :d {:a 1 :b 2 :c 3})
(will-fail mnemonic-suck-keyword-map-special-spit-number-fn1 :c {"x" 1 :b 2 :c 3})
```

The behaviour is  as expected but the generated contract look similar to
this:

```clojure
;; Example - swapping the keyword and map in the three level composed mnemonics
(clojure.core.contracts/contract ctx-aspect2879 "\"ctx-aspect2879\"" [arg0 arg1] [(keyword? arg0) (map? arg1) (every? keyword? (keys arg1)) (every? number? (vals arg1)) => (number? %)])
```

Some notes:

-   The _arg0_ in the canonical definition of the :map-special mnemonic has been automatically rewritten in the final contract to be _arg1_ i.e. the second argument. _argo_ refers to the :keyword (first) argument.

-   More generally, explicitly specified arguments in a mnemonic are automatically **shifted right** to whatever position the mnemonic has in the assertion clause. This applies recursively for composed mnemonics.

-   So when creating mnemonics, if you need to use explicit argument names (arg0, arg1, arg2, etc), name them relative to the mnemonic's argument order and they can be composed successfully.

### Example - using absolute arguments in mnemonics

Much (most?) of the time relative arguments names suffice.  But
there may be times when using composed mnemonics when you need to
specify (refer to) absolute argument names. 

A rather contrived scenario: say you needed to define
mnemonics with relative arguments but use an absolute argument inside the
relative mnemonic. Concretely:  e.g. if the first argument is a map but
the third (relative) argument must be a keyword that is a key in the
map.  

Note the _:keyword-in-first-argument-map_ below uses _arg0_ to refer
to itself (i.e. the keyword) but _abs-arg0_ to refer to  the first argument (i.e. the map).

```clojure
(configure-contracts-store
 aspect-mnemonic-definitions
 {:keyword-in-first-argument-map {:suck [[:keyword (contains? abs-arg0 arg0)]]}}) 
```

The example follows the familiar format:

```clojure
;; Example - using absolute arguments in mnemonics

;; This function takes a map, string and keyword, and returns a number.

;; The map must have keyword keys and numberic values.

;; The keyword must exist in the map

(def absolute-argument-mnemonic-fn1 (apply-contract-aspects (fn [m s k] (k m)) {:suck [:suck-map-special :string :keyword-in-first-argument-map] :spit :number}))

;; The same tests as above but the arguments swapped

(will-work absolute-argument-mnemonic-fn1 {:a 1 :b 2 :c 3} "s1" :a)
(will-fail absolute-argument-mnemonic-fn1 {:a :x :b 2 :c 3} "s1" :a)
(will-fail absolute-argument-mnemonic-fn1 {:a 1 :b 2 :c 3} "s1" :d)
(will-fail absolute-argument-mnemonic-fn1 {"x" 1 :b 2 :c 3} "s1" :c)
```

For reference, the contract looks like the below, the _abs-arg0_ has
been rewritten to _arg0_ while the _arg0_ in the
_:keyword-in-first-argument-map_ mnemonic has been rewritten to _arg2_.

```clojure
(clojure.core.contracts/contract ctx-aspect2879 "\"ctx-aspect2879\"" [arg0 arg1 arg2] [(map? arg0) (every? keyword? (keys arg0)) (every? number? (vals arg0)) (string? arg1) (keyword? arg2) (contains? arg0 arg2) => (number? %)])
```

# Beware mnemonic gotchas

The code tries to be as aggressive as possible to catch
inconsistencies and ensure your get what
you want. But there are some things to be aware of

#### Beware mnemonic gotchas - infinite recursion

Because mnemonics can use other mnemonic in their definition there is the
ability to create an infinite loop if a "downstream" mnemonic refers to
an "upstream" one.

_It would be possible to "remember" used mnemonics during evaluation
but not done so yet._

#### Beware mnemonic gotchas - incompatible argument assertions

If a custom mnemonic's argument assertions conflict with an explicit predicate,
built-in mnemonic (e.g. :map) or another custom mnemonic, the contract
will include more than one, but potentially different,
assertions for the same argument.  Which will fail miserably.

Note though that duplicate assertions for the same argument will be **distinct**-ified and cause no issue. 

#### Beware mnemonic gotchas - unexpected arguments

If a custom mnemonic with two arguments is applied to a function expecting
e.g. only one argument, an error will occur at run time.

# Contracts with Multiple Arities

**CCC** supports contracts for functions with multiple arities.

**CHUGAR** supports multiple arities, just put them
all in a vector on the call to e.g. _apply-contract-aspects_.

**CHUGAR** raises an error if it identifies  contracts with the same
arity for the same function in the same call to the macro (e.g. _apply-contract-aspects_).

## Example - two arities (map => number) and (map,keyword => vector)

This example of a multiple arities contract defines one arity for a single argument
function that suck a map and returns a vector; and a second arity for
a two argument function that sucks a map and keyword and spits a
vector.

```clojure
;; Example - two arities (map => number) and (map,keyword => number)

;; This is the target function with two arities

(defn two-arity-fn1
  ([m] (:a m))
  ([m k] (k m)))

;; The constrained function

(def constrained-two-arity-fn1 (apply-contract-aspects two-arity-fn1 [{:suck :map :spit :number} {:suck [:map :keyword] :spit :vector}]))

;; First Arity Tests

;; This will works as value of key :a is a number

(will-work constrained-two-arity-fn1 {:a 1 :b 2 :c [1 2 3]})

; This will fail as value of key :a is not a number

(will-fail constrained-two-arity-fn1 {:a "x"})

;; Second Arity Tests

;; This will work as value of key :c is a vector

(will-work constrained-two-arity-fn1 {:a 1 :b 2 :c [1 2 3]} :c)

; This will fail as value of key :d is not a vector (its nil)

(will-fail constrained-two-arity-fn1 {:a "x"} :d)
```

## Example - multiple arities using mixed CCC form and map form

The definition of the contract for each arity can be either CCC form
or map form; they can be mixed as well.

```clojure
;; Example - multiple arities using mixed CCC form and map form 

;; The same multiple arity example as above but using a mixed contract definition with CCC form and map form.

(def constrained-two-arity-fn1 (apply-contract-aspects two-arity-fn1 [[[m] [map? => number?]]  {:suck [:map :keyword] :spit :vector}]))

;; First Arity Tests

;; This will works as value of key :a is a number

(will-work constrained-two-arity-fn1 {:a 1 :b 2 :c [1 2 3]})

; This will fail as value of key :a is not a number

(will-fail constrained-two-arity-fn1 {:a "x"})

;; Second Arity Tests

;; This will work as value of key :c is a vector

(will-work constrained-two-arity-fn1 {:a 1 :b 2 :c [1 2 3]} :c)

; This will fail as value of key :d is not a vector (its nil)

(will-fail constrained-two-arity-fn1 {:a "x"} :d)
```

# Final Words

**CHUGAR** is part of a larger project (other parts to be published soon). 

Writing the project has taught me a lot about 
[Clojure][ClojureHome] (notably macros and protocols) and its
ecosystem (testing, profiling, Clojars and suchlike)  but I still have lots to learn.

I'm sure more experienced Clojurians will have some head-scratching moments if they
looked at the code.  As I tweeted recently, I think the biggest challenge
to learning a new language is to design idiomatically and well in it.
All advice on that subject gratefully received and acknowledged.

The whole point of **CHUGAR** was/is to make using the rich features of
**CCC** as easy  as possible.  I hope it (begins to) succeed on that
criterion and I believe  _mnemonics_ offers an original contribution
and productivity aid for defining, re-using and composing contract aspects.

I already have another project article in the works on the practical use of
**CHUGAR** to apply aspect contracts to the value of map keys.
Coming soon!

# Final Final Words

The overall project is the first "serious" (as opposed to dabbling)
[Clojure][ClojureHome] code I've written.  And the first serious
code in a functional language. 

> In all my time writing software, I
> can't ever remember learning a new language that just gets out of the
> way when I'm rattling along, but gets in the way when I get stuck and
> need some help overcoming an implementation or design issue.

# HOLD

This may seems a bit dry, even academic ("Where the beef?",
"What does this offer over Core Contracts?" and "Why would I
bother?"), so I'll follow up with posts giving practical examples of
the library's usage (one of which will be about the need that made
me build out **CHUGAR** in the first place).




[ClojureHome]: http:///clojure.org
[JavaHome]: http://www.java.com
[LeiningenHome]: http://leiningen.org/
[LeiningenGithub]: https://github.com/technomancy/leiningen
[MavenHome]: http://maven.apache.org/
[ClojarsHome]: http://clojars.org
[ClojarsClojureContractsSUgar]: https://clojars.org/name.rumford/clojure-contracts-sugar
[ClojureCoreContractsGithub]: https://github.com/clojure/core.contracts
[ClojureContractsSugarGithub]: https://github.com/ianrumford/clojure-contracts-sugar
[ClojureCarpGithub]: https://github.com/ianrumford/clojure-carp
[RumfordBlogClojureCoreContractsFirstTake]: http://ianrumford.github.io/blog/2012/11/17/first-take-on-contracts-in-clojure/
[FogusHome]: http://blog.fogus.me
[FogusBlogClojurePreandPost]: http://blog.fogus.me/2009/12/21/clojures-pre-and-post/
[Eiffel Design by Contract]: http://en.wikipedia.org/wiki/Eiffel_(programming_language)#Design_by_Contract
[DesignByContractWikipedia]: http://en.wikipedia.org/wiki/Design_by_contract
[CACMOct2001]:  http://dl.acm.org/citation.cfm?id=383845
[CACMArticleAOPIntro]: http://dl.acm.org/citation.cfm?id=383845.383853&coll=portal&dl=ACM
[AOPWikipedia]: http://en.wikipedia.org/wiki/Aspect-oriented_programming
[LaddadBookAspectJinAction]: http://www.manning.com/laddad/
[LaddadTwitter]: https://twitter.com/ramnivas
[KiczalesHome]: http://people.cs.ubc.ca/~gregor/
[EmerickHome]: http://cemerick.com/
[GrandHome]: http://clj-me.cgrand.net/
[CarperHome]: http://briancarper.net/
[ClojureProgrammingBook]: http://www.clojurebook.com/
[emacshome]: http://www.gnu.org/software/emacs/
[orgmodehome]: http://orgmode.org/
[orgmodemanualextractsourcecode]: http://orgmode.org/org.html#Extracting-source-code
