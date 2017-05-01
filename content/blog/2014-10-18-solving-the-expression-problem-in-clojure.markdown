---
title: "Solving the Expression Problem in Clojure"
date: 2014-10-18
comments: false
categories: 
---

Last night, I was having a drink with a friend and he asked me what I
liked about Clojure. Immutable data structures are coming in vogue
[outside](https://github.com/facebook/immutable-js) Clojure, and they
don't need to be sold very hard. I don't know a lot about virtual
machine optimization, but I've always been swayed by the argument that
with the amount of dollars and intellectual effort spent on JVM
optimization in the past decades, it's pretty fast. Honestly, I just
find [parentheses](http://xkcd.com/297/) alluring.

Then I tried to say something about how protocols elegantly solve the
expression problem, my friend had no idea what I was talking about.

I started writing an email about what I meant, and it morphed into this post.

## The expression problem

To be honest, "a drink" is somewhat of an understatement, and I did a
poor job of explaining what the expression problem actually is.

The best explanation of the expression problem comes from
[c2.com](http://c2.com/cgi/wiki?ExpressionProblem), and I include it
almost in its entirety:

> The "expression problem" is a phrase used to describe a dual problem that neither ObjectOrientedProgramming nor FunctionalProgramming fully addresses.
> 
> The basic problem can be expressed in terms of a simple example: Suppose you wish to describe shapes, including rectangles and circles, and you wish to compute the areas.
> 
> In FunctionalProgramming, you would describe a data type such as:
> 
>     type Shape = Square of side
>                  | Circle of radius 
> 
>
> Then you would define a single area function:
> 
> 
>     define area = fun x -> case x of
>         Square of side => (side * side)
>         | Circle of radius => (3.14 *  radius * radius)
> 
>
> In ObjectOrientedProgramming, you would describe the data type such as:
> 
> 
>     class Shape <: Object
>       virtual fun area : () -> double
>    
>     class Square <: Shape
>       side : double
>       area() =  side * side
>      
>     class Circle <: Shape
>       radius : double
>       area() = 3.14 * radius * radius
> 
> 
>The 'ExpressionProblem' manifests when you wish to 'extend' the set of objects or functions.
> 
> - If you want to add a 'triangle' shape: 
>     - the ObjectOrientedProgramming approach makes it easy (because you can simply create a new class)
>     - but FunctionalProgramming makes it difficult (because you'll need to edit every function that accepts a 'Shape' parameter, including 'area') 
> - OTOH, if you want add a 'perimeter' function: 
>     - FunctionalProgramming makes it easy (simply add a new function 'perimeter')
>     - while ObjectOrientedProgramming makes it difficult (because you'll need to edit every class to add 'perimeter()' to the interface). 
>

## Defining some shapes

Clojure's [records](http://clojure.org/datatypes) serve the purpose of types and classes in the examples above. 

```clojure
;; We have circles with radii
(defrecord Circle [radius])

;; and squares with sides, oh my!
(defrecord Square [side])

;; Here's a circle
(def my-circle (Circle. 5))

;; And a square!
(def my-square (Square. 4))
```

## Adding a new Shape

As the quote above says, in functional programming languages, defining a perimeter function is easy, but adding a new shape is hard. If our area function looks like this:

```clojure
(defn area [shape]
  (cond 
   (instance? Circle shape) (* Math/PI (:radius shape) (:radius shape))
   (instance? Square shape) (* (:side shape) (:side shape))))
```

we can easily add a perimeter function in the same vein, but adding a
new shape is harder.

We can define a new record:

```clojure
(defrecord Triangle [a b c])
```

But now we're stuck rewriting the `area` function to include `Triangle` in the switch statement. If we were using a shape library that didn't have triangles, we would have a hard
time extending it without forking their code.

We need a better way to do polymorphism then a switch statement!

One choice is [multi-methods](http://clojure.org/multimethods), but I'm going to focus on Protocols.

## Protocols

[Protocols](http://clojure.org/protocols) are an abstract notion of
interfaces from OO land. A protocol is simply a set of methods and
their signatures. If a type participates in a protocol, there exists
an implementation for that method for that type.

```clojure
;; Things that are Areable have an area method
(defprotocol Areable
  (area [this]))
```

We use `extend-protocol` to define how `Square` and `Circle` implement the `Areable` protocol[^1].

```clojure
;; Circles and Squares are Areable
(extend-protocol Areable
  Circle
  (area [{radius :radius}] (* Math/PI radius radius))
  Square
  (area [{side :side}] (* side side)))
```

Protocol methods behave just like functions, with dispatch determined by the type of the argument:

```clojure
;; A circle with the radius 5
(def my-circle (Circle. 5))

(area my-circle)
; => 78.53981633974483

;; A square with the side 5
(def my-square (Square. 5))

(area my-square)
; => 25
```

With protocols we simple extend the Triangle type to support the Areable protocol:

```clojure
(extend-type Triangle
  Areable
  (area [this] ...))
```

## Adding a new function

The other part of the expression problem is defining a "perimeter"
method. Types can implement multiple protocols, and this implementation can be
written anywhere. You can define a new protocol, and then extend core
Java classes to participate in it. If you come from the Ruby world,
you might think that this is similar to monkey-patching, and it is.

Because we can extend the `Circle` and `Square` types to participate
in a new protocol, adding a perimeter is also straightforward:

We define a new protocol

```clojure
(defprotocol Perimeterable
  (perimeter [this]))
```

and extend it to support `Circle` and `Square`

```
(extend-protocol Perimeterable
  Circle
  (perimeter [{radius :radius}] (* 2 Math/PI radius))
  Square
  (perimeter [{side :side}] (* 4 side)))
```

## On "monkey-patching"

If you come from the Ruby world, the above code might remind you of
monkey-patching. Monkey-patching certainly deserves its infamy, but
it's problems are misunderstood. Being able to add functionality to
core types is not a bad thing! The issue is namespacing.

What bothers many people about monkey-patching in Ruby is that you can load
a library, and suddenly it injects all kinds of new methods into core
classes you didn't expect:

```ruby
"string".methods.count
# => 170
require 'rails'
"string".methods.count
# => 225
```

Now your code has 55 new methods in the String class, and who knows
how many of the existing String methods you know and love have been
redefined!

In Clojure, you can extend core types to support whatever protocol you
want, but participating in a protocol is just like defining some functions,
and those functions are scoped to a namespace:

```clojure
(def foo bar)
(in-ns 'monkeypatching)
;; We're now in the monkeypatching namespace

(defprotocol Monkey
  (patch [this]))
            
;; Let's let nil and String participate in this protocol
(extend-protocol Monkey
  nil
  (patch [this] "Check out this cool nil!")
  String
  (patch [this] "I'm an awesome string!"))

(patch nil)
; => "Check out this cool nil!"
(patch "this is some string")
; => "I'm an awesome string!"

;; Now we switch back to another namespace to do some hard work
(in-ns 'workin-hard)

(patch nil)
; => CompilerException java.lang.RuntimeException: Unable to resolve symbol: patch in this context, compiling:(NO_SOURCE_PATH:1:1)

;; We can access the "monkeypatched" methods by using their namespace
(monkeypatching/patch "a string")
; => "I'm an awesome string!"
```


## Brining it together

Let's add a `Triangle` type that support both `area` and `perimeter`.  We can even provide implementations for the protocols it
participates in directly inside of `defrecord`:

```clojure
(defrecord Triangle [a b c]
  Areable
  ;; The hardest part of solving the expression problem in Clojure is looking up highschool geometry
  ;; Heron's formula
  (area [{:keys [a b c]}] 
    (let [s (/ (+ a b c) 2)]
       (Math/sqrt (* s (- s a) (- s b) (- s c)))))
  Perimeterable
  (perimeter [{:keys [a b c]}] (+ a b c)))
```

## Footnotes

[^1]:
    The `[{radius :radius}]` bit may be confusing. It's using [map destructuring](http://clojure.org/special_forms#Special%20Forms--Binding%20Forms%20%28Destructuring%29-Map%20binding%20destructuring) to bind the `:radius` field of the Circle record to the symbol `radius`.
    Without it, the function would look like this:

        (area [this]
          (let [radius (:radius this)]
            (* Math/PI radius radius)))
