---
layout: post
title: "Some trivial examples of using Clojure Reducers"
date: 2013-08-25 12:30
comments: true
categories: 
---

> TL;DR: **Reducers** are all about maximising performance.

The [Clojure][Clojure] [Reducers][Clojure Reducers Github] library
(**"reducers"**) was announced by [Clojure's][CLojure] author
[Rich Hickey][Rich Hickey] in May 2012 in
[this][Clojure Reducers Blog 1] blog post. He  followed up a week later with
a [second][CLojure Reducers Blog 2] post.

These posts were very well written and densely packed with volumes
 of information and both really benefit from being pored over and
multiple re-reading. Their approach was one of trying to convey an
_understanding_ of reducers together with some examples of how to
_use_ them.

Both posts covered a lot of ground, introducing the (new) **reducers** way
of thinking about _map, filter & reduce_ for collections, aided by new terminology to
explain the new ideas.

**Reducers** also use heavily Clojure's support for
dynamically generating functions (in fact, a function generating a
function that in turn generates a third function) so you need to have a clear grasp of
how that works as well. 

The second post  also explained how *reduce* had been changed (in v1.4) to allow collections to support directly reduction using the **CollReduce** protocol.

Quite a lot to take in and the mental jigsaw puzzle quite hard to
piece together at times.

There have been other posts explaining **reducers** (especially good is
this slideshare from [Leonardo Borges][LeonardoBorges]) but all have
followed the approach of, and examples from, Rich's original posts i.e.
_understand_ and then _use_.

Sometimes though is easier to consolidate your _understanding_ after
having _used_ something successfully. As I've been poring over Rich's
and other
posts, I been writing some examples for precisely that reason
and thought I'd present a  "narrative"  of some trivial example to illustrate the use
of  **reducers**.

I'm will be deliberately avoiding as much "theory" and new terminology  as I can
and just concentrating on some practical, if trivial, examples, labouring points sometimes
to ensure clarity.




<!-- more -->

# The Examples Collection #

Many of the following examples will use the same collection which
describes the population of a very (very!) small village.  The village
has four families, some families have two parents, others one; and
each family between one and three children.  One family lives in the North, and one each in the South, East and West.

_This is stylised, artificial and contrived data designed to be easily understandable to most and in no way intended to suggest any family structures, conventions or arrangements as socially preferable.  Just saying._

``` clojure
;; The Families in the Village

(def village
  [
   {:home :north :family "smith" :name "sue" :age 37 :sex :f :role :parent}
   {:home :north :family "smith" :name "stan" :age 35 :sex :m :role :parent}
   {:home :north :family "smith" :name "simon" :age 7 :sex :m :role :child}
   {:home :north :family "smith" :name "sadie" :age 5 :sex :f :role :child}
   
   {:home :south :family "jones" :name "jill" :age 45 :sex :f :role :parent}
   {:home :south :family "jones" :name "jeff" :age 45 :sex :m :role :parent}
   {:home :south :family "jones" :name "jackie" :age 19 :sex :f :role :child}
   {:home :south :family "jones" :name "jason" :age 16 :sex :f :role :child}
   {:home :south :family "jones" :name "june" :age 14 :sex :f :role :child}

   {:home :west :family "brown" :name "billie" :age 55 :sex :f :role :parent}
   {:home :west :family "brown" :name "brian" :age 23 :sex :m :role :child}
   {:home :west :family "brown" :name "bettie" :age 29 :sex :f :role :child}
   
   {:home :east :family "williams" :name "walter" :age 23 :sex :m :role :parent}
   {:home :east :family "williams" :name "wanda" :age 3 :sex :f :role :child}
   ])
```

# Code is on Github #

The code can be found [here][Clojure Reducers Code].

Note the **reducers** library is referred to as _r_:

```clojure
  [:require [clojure.core.reducers :as r]
            [clojure.string :as string]]
```

# Example 1 - how many children in the village? #

An obvious way to do this would be to _map_  each child to the value 1, else 0.  The total number of children is then the simple addition of the 1s & 0s results of the map operation.

## Example 1 - create the reducers map function to return 1 if a child, else 0

But, with **reducers**, you don't use _map_ in the same way as **core**
map because the **reducers** _map_ function (i.e. r/map below) returns
another function to be used together with a collection and
_reduce_.

```clojure
;; Example 1 - create the reducers map function to return 1 if a child, else 0
(def ex1-map-children-to-value-1 (r/map #(if (= :child (:role %)) 1 0)))
```

## Example 1 - use reduce to add up all the mapped values

The _reduce_ function has to be used to perform (action) the mapping and then add up the mapped 1 & 0 values.

```clojure
;; Example 1 - use reduce to add up all the mapped values
(r/reduce + 0 (ex1-map-children-to-value-1 village))
;;=>
8
```

An important point to emphasise here is  the **reducers** _map_
function only "prepares" the resulting collection to be consumed by
the _reduce_; the _map_ itself doesn't do anything off its own bat to the contents of
the collection and needs _reduce_ to "drive" the mapping.


# Example 2 - how many children in the Brown family? #

Ok, how about finding how many children just in the Brown family? Again,
an obvious way to do this would be to select ( _filter_) the members of the
Brown family, and use the same _map_ function as Example 1 
(_ex1-map-children-to-value-1_)  to count the children.

## Example 2 - select the members of the Brown family ##

Just like _map_, **reducers** has a _filter_ function that returns another function that can be used with _reduce_:

```clojure
;; Example 2 - select the members of the Brown family
(def ex2-select-the-brown-family (r/filter #(= "brown" (string/lower-case (:family %)))))
```

## Example 2 - compose a composite function to select the Brown family and map children to 1

The functions returned by **reducers** _map_ and _filter_ can be composed in the usual Clojure way to create a multi-step transformation. So we can create a transformation _pipeline_ function to perform first the _ex2-select-the-brown-family_ and then _ex1-map children to 1_ :

```clojure
;; compose a composite function to select the Brown family and map children to 1
(def ex2-pipeline (comp ex1-map-children-to-value-1 ex2-select-the-brown-family))
```

## Example 2 - using reduce to add up all the Brown children ##

The pipeline function (_ex2-pipeline_) then can be used with  _reduce_ to find (add) the number of children in the Brown family:

```clojure
;; Example 2 - using reduce to add up all the  Brown children
(r/reduce + 0 (ex2-pipeline village))
;;=>
2
```

Its worth noting that although this is a multi-step transformation
pipeline (i.e _filter_ then _map_), the _reduce_ does *not* need to create any
intermediate collections as would be the case when using **core** _map_
and _filter_.  This is a useful performance boost.


#  Example 3 - how many children's names start with J? #

We already know the answer:  just the 3 children in the _Jones_ family.

Algorithmically, this is a three step pipeline: _filter_ on children, a filter on
names beginning with "j" and then a count of how many children in the result.

## Example 3 - selecting (filtering) just the children ##

A very simple filter will select just the children:

```clojure
;; Example 3 - selecting (filtering) just the children
(def ex3-select-children (r/filter #(= :child (:role %))
```

## Example 3 - selecting names beginning with "j" ##

Again, this is a very simple filter:

```clojure
;; Example 3 - selecting names beginning with "j"
(def ex3-select-names-beginning-with-j (r/filter #(= "j" (string/lower-case (first (:name %))))))
```

## Example 3 - mapping the  entries in a collection to 1 ##

In Example 1 we created the _map_ function _ex1-map-children-to-value-1_ to
enable _reduce_ to count the number of children.

But the need to count the number
of entries in a collection using _reduce_, after a pipeline possibly involving many filters and mappers, is
a common one.  This is straightforward to do,  in the final stage of the pipeline use a  _map_ function to transform each entry to
value 1:

```clojure
;; Example 3 - mapping the  entries in a collection to 1
(def ex0-map-to-value-1 (r/map (fn [v] 1)))
```

## Example 3 - create the three step pipeline function ##

As in Example 2, the three-step pipeline can be composed simply:

```clojure
;; Example 3 - create the three step pipeline function
(def ex3-pipeline (comp ex0-map-to-value-1
                        ex3-select-names-beginning-with-j
                        ex3-select-children))
```

Its worth labouring the point that composing custom pipelines from
simply individual _filters_ and _mappers_ is very powerful.

## Example 3 - reduce the village with the pipeline function ##

The final reduction should now be familiar and the result will be the number of children whose names begin with "j":

```clojure
;; Example 3 - reduce the village with the pipeline function
(r/reduce + 0 (ex3-pipeline village))
;; =>
3
```


# Example 4 - making a collection of the children whose names start with J? #

Sometimes though you will want the actually collection itself.

## Example 4 - a pipeline to just filter children with names starting with "j"

Since we want the actual entries, rather than count them, we need a  pure filter pipeline, similar to Example 3 but one that **doesn't** use the _ex-map-to-value-1_ mapper:

```clojure
;; Example 4 - a pipeline to just filter children with names starting with "j"
(def ex4-pipeline (comp ex3-select-names-beginning-with-j
                        ex3-select-children)
```

## Example 4 - create a vector with the "j" children

Creating a vector a _vector_ of the children who's names start with J can be done simply using _into_.

Under the covers _into_ uses _reduce_ so making the vector is just a matter of applying the _ex4-pipeline_ function to _village_  and then using _into_ to build the vector.

```clojure
;; Example 4 - create a vector with the "j" children
(into [] (ex4-pipeline village))
;; =>
[{:age 19, :home :south, :name "jackie", :sex :f, :family "jones", :role :child}
 {:age 16, :home :south, :name "jason", :sex :f, :family "jones", :role :child}
 {:age 14, :home :south, :name "june", :sex :f, :family "jones", :role :child}]
```

# Example 5 - average age of children on or below the equator

A more involved, but still straightforward, example to finish this section:  what is the average age of the children who live on or below the equator? By equator I mean where _home_ is East, South or West.

To do this, the value of _home_ will be mapped to a latitude and longitude.  For example West will be _:lat 0 :lng -180_ and South is _:lat -90 :lng 0_.  

## Example 5 - select the children

We can re-use the _ex3-select-children_ filter again.

## Example 5 - map home to latitude and longitude

A simple mapper to add latitude and longitude depending on the value of _:home_:

```clojure
;; Example 5 - map :home to latitude and longitude
(def ex5-map-home-to-latitude-and-longitude
  (r/map
   (fn [v]
     (condp = (:home v)
       :north (assoc v :lat 90 :lng 0)
       :south (assoc v :lat -90 :lng 0)
       :west (assoc v :lat 0 :lng -180)
       :east (assoc v :lat 0 :lng 180)))))
```

## Example 5 - select people on or below the equator i.e. latitude <= 0

People on or below the equator have a latitude with maximum value of 0:

```clojure
;; Example 5 - select people on or below the equator i.e. latitude <= 0
(def ex5-select-on-or-below-equator (r/filter #(>= 0 (:lat %)))
```

## Example 5 - count the number of children on or below the equator

To find the average age, we need to know the number of children.

Note, rather than creating a composite pipeline function, in this example the individual stages of the pipeline have been used explicitly:

```clojure
;; Example 5 - count the number of children on or below the equator
(def ex5-no-children-on-or-below-the-equator
  (r/reduce + 0
          (ex0-map-to-value-1
           (ex5-select-people-on-or-below-equator
            (ex5-map-home-to-latitude-and-longitude
             (ex3-select-children village))))))
```

## Example 5 - sum the ages of children

To sum the ages require the _ex0-map-to-value-1_ to be replaced by a function to select the age value (_ex5-select-age_):

```clojure
;; Example 5 - sum the ages of children
(def ex5-select-age (r/map #(:age %)))

(def ex5-sum-of-ages-of-children-on-or-below-the-equator
  (r/reduce + 0
          (ex5-select-age
           (ex5-select-people-on-or-below-equator
            (ex5-map-home-to-latitude-and-longitude
             (ex3-select-children village))))))
```

## Example 5 - calculate the average age of children on or below the equator

For completeness, the final  step is the  division to calculate the average age:

```clojure
;; Example 5 - calculate the average age of children on or below the equator
(def ex5-averge-age-of-children-on-or-below-the-equator (float (/ ex5-sum-of-ages-of-children-on-or-below-the-equator ex5-no-children-on-or-below-the-equator )))
;; =>
17.3
```

# That was fun but why bother? #

The elimination of intermediate collections in **reducers** multi-step pipelines is a very useful performance boost, and 
the **reducers** way of thinking about and processing collections is an interesting mindset change, but are they worth the bother?

The answer lies in a peer function of _reduce_ called _fold_. When
_fold_ can be used, it will **automatically** parallelise the
processing of the collection, taking advantage of multiple CPU cores.

Where _reduce_ has been used above, _fold_ can be used instead. **NOTE THOUGH** _fold_ has slightly different arguments; at its simplest it does not have / need the initial value.

If needed, _fold_ will drop down to _reduce_ if it can't parallelise.

In fact, _fold_ breaks the collection down into chunks (currently
defaulting to 512) and processes (_reduces_) each chunk in parallel. When a chunk
has been processed, its results have to be __combined__ with the
results of the other processed chunks.

Bit of unavoidable theory, to maintain the order
of the results the _map_ and _filter_ functions used by _fold_ must
be [associative][AssociativeProperty]. Associativity means simply that how results
are "grouped" together is unimportant e.g. for arithmetic
addition (+) then 1 + (2 + 3) === (1 + 2) + 3 === (1 + 2 + 3). Not all operators are
associative though.

# Example 6 - comparing the performance of reduce and fold

Lets time the addition of the ages from Example 5 using _reduce_ and _fold_.

## Example 6 - time reduce adding up Example 5's ages

Lets time _reduce_ first:

```clojure
;; Example 6 - time reduce adding up Example 5's ages
(time (r/reduce +
                (ex5-select-age
                 (ex5-select-people-on-or-below-equator
                  (ex5-map-home-to-latitude-and-longitude
                   (ex3-select-children village))))))
;; =>
"Elapsed time: 0.091714 msecs"
```

# Example 6 - time fold adding up Example 5's ages

Now _fold_:

```clojure
;; Example 6 - time fold adding up Example 5's ages
(time (r/fold +
              (ex5-select-age
               (ex5-select-people-on-or-below-equator
                (ex5-map-home-to-latitude-and-longitude
                 (ex3-select-children village))))))
;; =>
"Elapsed time: 0.185448 msecs"
```

So _fold_ took twice as long as _reduce_ in this example because (I guess) the _fold_ _"red tape"_ (e.g. _chunking_ and _combining_) outweighed the benefit gained from performing the computation in parallel.  No free lunch.

# Example 7 - all the relatives visit the village!

Every year all the relatives of the families in the village visit and
the population of the village swells enormously. _Stick with me on
this :-)_

## Example 7 - make some visitors

Lets define some functions to create an influx of visitors. _Note, no attempt has been made to ensure this randomly generated data makes any sort of real world sense - it could include e.g. a child of age 100._

```clojure
;; Example 7 - make some visitors

(def ex7-fn-random-name (fn [] (rand-nth ["chris" "jim" "mark" "jon" "lisa" "kate" "jay" "june" "julie" "laura"])))
(def ex7-fn-random-family (fn [] (rand-nth ["smith" "jones" "brown" "williams" "taylor" "davies"])))
(def ex7-fn-random-home (fn [] (rand-nth [:north :south :east :west])))
(def ex7-fn-random-sex (fn [] (rand-nth [:m :f])))
(def ex7-fn-random-role (fn [] (rand-nth [:child :parent])))
(def ex7-fn-random-age (fn [] (rand-int 100)))

(def ex7-visitor-template
  {:home ex7-fn-random-home
   :family ex7-fn-random-family
   :name ex7-fn-random-name
   :age ex7-fn-random-age
   :sex ex7-fn-random-sex
   :role ex7-fn-random-role})

(defn ex7-make-visitor [] (into {} (for [[k v] ex7-visitor-template] [k (v)])))

(defn ex7-make-visitors [n] (take n (repeatedly ex7-make-visitor)))

(def ex7-visitors (into [] (ex7-make-visitors 1000000)))
```

## Example 7 - count the visiting Brown children using reduce

Lets reprise Example 2 and time how long it takes to count the visiting
Brown children using **reducers** _reduce_:

```clojure
;; Example 7 - count the visiting Brown children using reduce
(time (r/reduce + 0 (ex2-pipeline ex7-visitors)))
;; =>
"Elapsed time: 238.448041 msecs"
```

## Example 7 - count the visiting Brown children using fold

Ok, lets try the same using _fold_:

```clojure
;; Example 7 - count the visiting Brown children using fold
;;(time (r/fold + (ex2-pipeline ex7-visitors)))
;; =>
"Elapsed time: 59.6259 msecs"
```

Nearly four times faster! (My workstation has four cores.)  _The exact timing values should be taken with a pinch of salt though as this wasn't a rigorous benchmark.  But the improvement is undeniable._

## Example 7 - count the visiting Brown children using core map, filter and reduce

How do the **reducers** fare against **core** map, filter and reduce?

```clojure
;; Example 7 - count the visiting Brown children using core map, filter and reduce
(time (reduce + 0
              (map #(if (= :child (:role %)) 1 0)
                   (filter #(= "brown" (string/lower-case (:family %))) ex7-visitors))))

;; =>
"Elapsed time: 223.55717 msecs"
```

The **core** _reduce_ and **reducers** _reduce_ timings are pretty much the same; as you might expect since only one intermediate collection would be created by the **core** versions and very little else has changed.

# What else can you do with fold? #

Bit more theory.

I mentioned above that _fold_ breaks the collection into chunks, the default chunk size is 512, and each chunk is processed in parallel.

But this raises a question:  how does _fold_ know how to _combine_ the processed chunks to produce the final results?

Because _fold_ can take also the chunk size and  a _combiner_ function as arguments, as well as the usual _reducing_ function and collection.

To demonstrate, here is the  _fold_ from Example 7:

```clojure
;; A simple fold where both the reducer and combiner is +
(r/fold + (ex2-pipeline ex7-visitors))
```

In the above the **+** operator (function) was both the _reducer_ **and** _combiner_.  No chunk size was given.

A completely specified call to _fold_ could look like this:

```clojure
(r/fold the-chunk-size (r/monoid the-combiner-function the-init-function) the-reducer-function the-collection)
```

The reducer function could be **+** but is more likely to some custom logic.

The call to the _monoid_ function is a convenient way of creating a
properly formed combination function that _fold_ can use. _monoid_
takes two arguments, the first being your _combiner_ function, and the
second being the function that will create the **init** value when it
is called with no arguments. (In the previous examples the init value
to _reduce_ was 0)

Or, if your _combiner_ will generate the **init** value, it can be used directly:

```clojure
(r/fold the-chunk-size the-combiner-function-with-init the-reducer-function the-collection)
```

An example may help to clarify.


# Example 8 - find the average age of visiting children with names beginning "j"

In this example, we'll calculate the average age of all visiting children with names beginning "j".  But in just one fold, not like the two-step calculation of average age in Example 5.

## Example 8 - create a collection of visiting children with name beginning with "j"

To select  visiting children with names starting with "j" we can use exactly the same pipeline from Example 4, apply it to the visitors and store the collection _into_ a vector:

```clojure
;; Example 8 - create a collection of vistsors with name beginning with "j"
(def ex8-visitors-collection (into [] (ex4-pipeline (ex7-make-visitors 1000000))))
```

_I don't understand why but fold seems to need a "real" collection else it doesn't seem to call the combiner.  Answers on a postcard please._

## Example 8 - create the reducer to sum the ages and number of people in that chunk

The _reducer_ function adds up how many people have been seen in that chunk and also sums their age:

```clojure
;; Example 8 - create the reducer to sum the ages and number of people in that chunk
(def ex8-reducer
  (fn [s v]
    {:total-people (+ (get s :total-people 0) 1)    ;; add 1 to number of people seen
     :total-age (+ (get s :age 0) (get v :age))}))  ;; and add their ages
```

## Example 8 - create the combiner to calculate the average age

The _combiner_ takes the results of two processed chunks and calculates the average age.

```clojure
;; Example 8 - create the combiner to calculate the average age
(defn ex8-combiner
  ([] {}) ;; note this is the init value for each reduced chunk
  ([a b]
     (doall (println (format "EX8 CMB FN2 a >%s< b >%s<" a  b )))
     (let [total-people (+ (get a :total-people) (get b :total-people))
           total-age (+ (get a :total-age) (get b :total-age))
           average-age (float (/ total-age total-people))
           ]
       {:total-people total-people
        :total-age total-age
        :average-age average-age})))
```

Note, when called with no arguments, it returns the *init* value for the reduction of each chunk (an empty hash-map here).


## Example 8 - run the fold to perform the average calculation

The average age can now be calculated by running the _fold_:

```clojure
;; Example 8 - run the fold to perform the average calculation
(r/fold ex8-combiner-fn2 ex8-reducer-fn2  ex8-visitors-collection)
;; =>
{:total-people 28, :total-age 1314, :average-age 46.92857}
```

Note, the calculation of the average age in the _combiner_ rather than
the _reducer_ was an artificial distinction for illustration, the
processing could have been performed in one function. One reason why,
when doing arithmetic, a separate _combiner_ often isn't
necessary. However, one can conceive of some "final logic" that could and should
only be performed in a _combiner_.

# Final Words #

Some very trivial examples that only scratch the surface of the potential
of **reducers**.  

Once you _get_ **reducers** you realise they are
[simple if perhaps not easy at first][SimpleNotEasy].  But once you get
the hang of them, they feel as comfortable and accessible to use as
their **core** equivalents.

The performance benefits from being able to easily parallelise the
processing of collections is undeniable; for very little reworking of
the code, performance can be improved
significantly.

I particularly like how sophisticated, multi-step transformation
pipelines can be composed from individual _mappers_ and _filters_ and used with very little overhead (no need
to create intermediate collections.)


[Clojure]: http:///clojure.org
[Clojure Reducers Github]: https://github.com/clojure/clojure/blob/master/src/clj/clojure/core/reducers.clj
[Leiningen]: https://github.com/technomancy/leiningen
[Clojure Reducers Blog 1]: http://clojure.com/blog/2012/05/08/reducers-a-library-and-model-for-collection-processing.html
[Clojure Reducers Blog 2]: http://clojure.com/blog/2012/05/15/anatomy-of-reducer.html
[Clojure Reducers Code]: https://gist.github.com/ianrumford/6333358
[Rich Hickey]: https://twitter.com/richhickey
[LeonardoBorges]: http://www.slideshare.net/borgesleonardo/clojure-reducers-cljsyd-aug-2012?ref=http://www.leonardoborges.com/writings/presentations/
[AssociativeProperty]: http://en.wikipedia.org/wiki/Associative_property
[SimpleNotEasy]: http://www.infoq.com/presentations/Simple-Made-Easy
