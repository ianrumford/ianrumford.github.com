---
layout: post
title: "A first take on contracts in Clojure"
date: 2012-11-17 19:20
comments: true
categories: 
---

I guess I've always written pretty defensive code (*trust nothing*),
insisting that e.g. passed arguments are as expected and needed.  Anybody who's
seen my recent Ruby code will have found it littered with many, many
assertions.  Usually 
my methods check both the input arguments for validity and also end with a call to a final *guard*  to ensure
the returned value is also valid.

During my time writing Perl (rather a long time ago), I  used  my own
variants of the standard  *carp*, *croak* and *confess* functions
heavily, and followed received wisdom
to *fail early and often*. 

After moving to Ruby as my primary language, my Perl style evolved  into the implementation of *mustbe* method e.g. *mustbe-hash-or-croak*
to ensure the argument is as expected.  My *mustbe* methods are
dynamically defined in a mixin
using standard Ruby's metaprogramming techniques and, although they commonly 
test whether the argument is an instance of a given class, they can
apply (assert)
arbitrary constraints.

In time I realised I'd fallen into a contracts style of programming.  I'd not sure
how or where I first came across contracts formalism but it may have
been when reading about
[Eiffel's Design by Contract][Eiffel Design by Contract].

Eiffel's DBC asserts *pre* and *post* constraints on a routine's execution; in my own
informal way, I'd become  strongly attracted to contract assertion, not only
on entry to and exit from  routines (methods, functions), but also in their  bodies.

My recent excursions into writing [Clojure][Clojure] has had me  reaching
for a way (technique) to apply contracts. Hence this post.


<!-- more -->

# Contracts in Clojure #

I first stumbled on contracts in Clojure whilst reading the *superb*
[Joy of Clojure](http://www.manning.com/fogus/) (*JoC*) written 
by [Michael Fogus](http://blog.fogus.me/) (hereafter just *Fogus*) and
[Chris Houser](http://old.n01se.net/chouser/).

Fogus had already
[blogged](http://blog.fogus.me/2009/12/21/clojures-pre-and-post/)
about Clojure's direct support for the enforcement (assertion) of a function's entry (*pre*) and
exit (*post*) constraints.  For example, the following function ensures
the passed argument is a positive number and a map (hash) is returned.

``` clojure
(defn my-constrained-function
  "This function constrains its input and output
  It ensures the passed argument is a positive number and returned value a map"
  [n]
  {:pre [(instance? Number n) (> n 0]
   :post [(map? %)]}
  {:return n})
```

Note both *:pre* and *:post*  apply an  *and-ed* sequence of
constraints;   *pre*  has two while *post* has one  in the above.

Note also, the return value is available in a *:post* constraint using
the  *%* familiar. 

To me, one of the most interesting, powerful and exciting capabilities of
Clojure's support for contracts is the ability (opportunity) to decouple the
constraints from the function the former are applied to.  Consider this example:

<script src="https://gist.github.com/4060196.js?file=blog_car1.clj"></script>

The call to *make-a-red-car* will succeed but the call to
*make-an-eco-car* will fail because the *fuel* given (*petrol*) is not one of the
acceptable values in the *post* condition.

The take away here is that the more specific constraints (i.e. red or eco)
can be external to the base *make-a-car* function. JoC
observes (section 7.1.5) on this *aspect*-like nature:

> By pulling out the assertions into a wrapper function, we’ve detached some domain-
specific requirements from a potentially globally useful function and isolated them in
aspects (Laddad 2003). By detaching pre- and postconditions from the functions them-
selves, you can mix in any implementation that you please, knowing that as long as it
fulfills the contract (Meyer 1991), its interposition is transparent. 

# Contracts Libraries for Clojure #

Fogus followed up the JoC's treatment of constraints with a 
[blog](http://blog.fogus.me/2010/05/25/trammel-contracts-programming-for-clojure/)
announcing his implementation of a contracts library for Clojure: [Trammel](https://github.com/fogus/trammel).

Also in the frame has been Dmitri Naumov's
[clojure-contracts](https://github.com/dnaumov/clojure-contracts).

Earlier this year Trammel and clojure-contracts were brought together
into
[clojure.core.contracts](https://github.com/clojure/core.contracts) (*CCC*)
which the Github page warns is a work in progress.  So, assuming the
core library has a more assured future than the earlier two, 
this post explores using *CCC*.  Documentation for *CCC* is less well
developed than e.g. Trammel:  I found myself looking at both Trammel's examples
and also *CCC*'s code to get my head fully around what was going on.


# Exploring clojure.core.contracts #

The core idea behind *CCC* is both simple and elegant.  Contracts are
applied to  a target function by "wrapping" the latter in anonymous
functions which apply the *pre* and *post* constraints.
Multiple contracts are applied by recursive definitions of
wrapper functions (so each wrapper sees all the input arguments and
their signature must be appropriate).

Much of the heavy lifting of *CCC* is accomplished by Clojure macros.

## Defining an *aspect* contract ##

The following code generates two contracts.  The first insists on the
argument being a map, and the second on the return value being a number:

<script src="https://gist.github.com/4060881.js?file=blog_contract1.clj"></script>

Notes:

* the *contract* macro generate an *anonymous* contract *but* still must
   be supplied with a name (e.g. *aspect-suck-a-map-cx*) and
  description (e.g. *enforce a map input argument*).
  
* To use the anonymous contract subsequently, the result of the *contract* macro
  must be assigned to a symbol (e.g. *aspect-suck-a-map*)
  
* in the input a map contract (*aspect-suck-a-map*), a single *pre* condition has been
  specified *(map? m)* 
  
* in the output a number contract (*aspect-spit-a-number*) , a single *post* condition has been
  specified *(instance Number %)*
  
    *post* conditions appear *after* the *=>*
    
    The return value is available in *post* conditions as *%*
    
* the [arity](http://en.wikipedia.org/wiki/Arity) of the contracts must be compatible to whatever subsequent
  function(s) they will be applied to (see later for more on this)

## Defining a constrained function ##

The *with-constraints* function applies 0, 1 or more *aspect* contracts
to an existing function, creating a new function that enforces all
the contracts.  For example, in the following example contracts are
used to ensure  the input argument to the
*guaranteed-suck-a-map-and-spit-a-number* function is definitely a map,
and the return value definitely a number.  The unconstrained function
(*suck-a-map-and-spit-a-number*) offers  no such guarantees.

<script src="https://gist.github.com/4061893.js?file=blog_with_constraints1.clj"></script>

## Applying contracts to existing functions ##

A refinement of *with-constraints* is  *provide* which "updates" an existing
function with contracts.  Under the covers, *provide* uses
*with-constraints* together with *alter-var-root* to update the
binding of the target function to the (new) function generated by
*with-constraints*.  For example:

<script src="https://gist.github.com/4062046.js?file=blog_provide1.clj"></script>

Notes:

* *provide* can take a 0, 1 or more vectors, in this example only one is
supplied:

* *provide* offers an easy way to add (or subtract) contracts to a
   existing function, without making any other changes to the code.
   
* contracts are applied in the order given.

# A note on contract signatures and arities #

Even though an output contract may only be interested in the return
value (*%*), it must still have a signature that is compatible with
every other contract used in the same  *with-constraints* call.  

This militates
somewhat against composing multiple argument functions using  generic
aspect contracts as the "composite"  signature and arity of the composed function must
be consistent with *all* of the  *pre* and *post* conditions / contracts.

The following code illustrates the point:

<script src="https://gist.github.com/4065672.js?file=blog_contract_arity1.clj"></script>

It would help some, especially for pure output contracts, if *CCC* supported *varargs* i.e the ability to have
optional arguments after  the *%* in the signature (e.g. [a b & c]).
This currently fails because the *build-contract-body* function does
not notice the *%* and includes it in an *apply*.  However a one line
modification to that function seems to do the job with no untoward
side effects:

```clojure
          '?ARGS       (vec (list* (remove #(= "&" (name %)) args)))
```

Some of the below relies on the use of varargs and hence the modification.

# Contracts Sugar #

So far, so good.  However, it would be *cumbrous* to define the
equivalent rich (large)
portfolio of contracts /  constrained functions I use in Ruby using the available
*CCC*  functions and macros.  More of an exercise in writing Clojure macros,
it is straightforward to implement a few productivity aids.

## Generating many contracts easily ##

The code below shows a macro (*make-contracts*) that does much of the leg-work to
define new contracts.  Like *provide*, it accepts 0, 1 or more
vectors.  Note, these examples are for one argument contracts.

<script src="https://gist.github.com/4066462.js?file=blog_make_contracts1.clj"></script>

## Generating *mustbe* functions ##

My Ruby *mustbe* methods mostly take one argument and test it, usually
to confirm whether it is an instance of a class or not.

The follow code includes a macro (*make-mustbe-functions*) to do the
necessary, leveraging *make-contracts* to define the *aspect* contracts.
Note, the "target" function is always *identity*.

<script src="https://gist.github.com/4079359.js?file=blog_make_mustbe_functions1.clj"></script>

## Generating contract functions for inline use ##

Sometimes, in "open code", you will also want to apply one or more
contracts to an value of an arbitrary expression.  Again, a macro
(*make-inline-contract-functions*) can help with the legwork.  Note this example
require my minor mod to support varags.

<script src="https://gist.github.com/4079419.js?file=blog_make_inline_functions1.clj"></script>


# Overloading  *pre* and *post* conditions support #

In order to satisfy a constraint in a *:pre* or *:post* condition, the
test function needs only to be (return) *true*: You can write any old
condition function that accepts the
arguments, does any processing it wants to, and as long as it
return true, the constraint will pass.  *:pre* and *:post* therefore
allow arbitrary logic to be applied at a function's entry and exit,
not just performing argument checking.

For example, you may want to know (e.g. *println*) the return value from a
function whilst testing, but turn  off the println  when in
production.  Using *provide* to update the definition of a target
function  makes its
trivially easy to enable and disable such *pseudo-contracts*, with no
run-time performance penalty either
(since
*provide* is a compile-time macro).  

Another interesting possibility
would be to use pseudo-contracts to support out-of-band auditing a function's usage, perhaps adding authorisation checks.

Creating a
pseudo-contract with an arbitrary code body, and probably without any actual
conditions,  that can be
used with *with-constraints* and / or *provide*
is not possible with the *contract*
macro. But it is possible to create a new macro, and hack the  other macros and
functions (*build-contract-body* and *build-contract-fn-body*) to
do the necessary.  It works but the code feels messy and should
probably be a feature request, together with varargs support, to Fogus
for comment.


# Final Words #

*CCC* is a great addition to the tools you can use to build quality
 Clojure code.  That said, support for rich usage (portfolio) of contracts feels a
 bit primitive right now.  But, as my own simple attempts have
 demonstrated, it wouldn't take much to add some  productivity features.


[Clojure]: http:///clojure.org
[Leiningen]: https://github.com/technomancy/leiningen
[Eiffel Design by Contract]: http://en.wikipedia.org/wiki/Eiffel_(programming_language)#Design_by_Contract

