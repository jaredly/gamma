# gamma

Gamma simplifies developing GLSL shaders for use with WebGL. It represents shaders as composable expressions, giving you the full power of Clojure to compose, abstract and manipulate them before finally compiling to an executable GLSL string. 

Using Gamma directly requires knowledge of GLSL. It can be used directly by applications, or by graphics library authors to create arbitrary abstractions, from composable shader libraries to high-level declarative graphics DSLs. 

Read the [rationale](https://github.com/kovasb/gamma/wiki/Gamma-Rationale).

# API

Gamma lets you do two things: a) construct a GLSL AST as clojure data, and b) compile it to a GLSL program string

## Constructing GLSL 

Gamma represents the GLSL AST as Clojure maps. 
The functions in gamma.api are convenience functions for constructing the maps:

```clojure
(require '[gamma.api :as g])
```

```clojure
(g/sin 1)
=>
{:tag :term, :head :sin, :id {:tag :id, :id 1}, :type :float,
  :body ({:tag :term, :head :literal, :value 1, :type :float, :id {:tag :id, :id 2}})}
```

The mapping from GLSL to Gamma's AST maps is in general very direct, although Gamma deviates in one important way. 

Gamma's AST is designed to represent pure functions operating on values. This vastly simplies the task of metaprogramming: no state, no side-effects, no variable names to juggle, no order-of-operations issues. 

Most importantly, it is referentially transparent, which means we can replace any AST fragment with code that generates it, and vice verse. This frees us to abstract the AST using any technique available in Clojure. 

As a simple example, consider all the ways we can factor the expression (g/+ 1 (g/+ 2 3)) using functions. 

```clojure
;; many equivalent forms of the following
(g/+ 1 (g/+ 2 3))

;; factor (g/+ 2 3) into reuseable function 
(defn my-fn [x] (g/+ x 3))

(g/+ 1 (my-fn 2))

;; input to my-fn can be another expression that computes to 2, not just a literal value 
(g/+ 1 (my-fn (g/+ 1 1)))

;; for that matter can generate a simple 2 with Clojure's arithmetic
(g/+ 1 (my-fn (+ 1 1)))

;; factor out the whole tree, and pass a constructor for the subtree   
(defn root-ast-fn [sub-ast-fn]
  (g/+ 1 (sub-ast-fn 2)))
  
(root-ast-fn my-fn)  
  
;; close over data 
(defn my-fn-2 [x] (fn [y] (g/+ x y)))

(root-ast-fn (my-fn-2 3))


```

Functions are an extremely powerful abstraction, making many composition patterns trivial that are difficult or impossible in GLSL. 

There is no limit to how you construct your ASTs, how you set up your composition, polymorphism, indirection, etc. 

```clojure
;; use arbitrary kinds of indirection or polymorphism
(g/+ 1 (some-AST-generating-fn))
(g/+ 1 (:val (map-returning-AST-generating-fn)))
(g/+ 1 (my-ast-returning-multimethod some-data))
(g/+ 1 (my-ast-returning-protocol some-type))
```

You can go to town with higher order functions:

```clojure
(reduce g/+ 0 [1 2 3 4])

(apply g/vec4 (map #(g/clamp % 0 1) [0 0.5 1 2]))
```

You can pass around AST fragments however you want, including inside datastructure:

```clojure

(def x {:color (g/vec3 0 0 1) :alpha 0.5})

(defn conj-alpha [c]
  (g/vec4 (:color c) (:alpha c)))
  
(conj-alpha x)  
```

If this looks like very basic programming, its because it is. However, these forms of abstraction are unavailable in raw GLSL, or in many of the languages and tools dedicated to wrapping it. 


### Differences from GLSL

#### No binding forms 

You don't create bindings or assignments within the AST.

If you want to pass an expression to multiple places, just do with Clojure:

```clojure
(defn my-abs [a b]
  (let [x (g/+ a b)]
    (g/if (g/< 0 x)
       (g/* -1 x) 
       x)))
```

#### Statements are expressions

Unlike ordinary GLSL, Gamma's AST is expression oriented. 

In particular, if-statements are expressions that can be nested in other expressions.

```clojure
(g/+ 1 (g/if conditional-expr 2 3))
```


This duplicates the (g/+ a b) AST fragment in multiple places, but Gamma's compiler will optimize that duplication away.

#### No for loops 

WebGL only supports for loops that are un-rollable. You can unroll these yourself trivially with higher-order functions.

For example, map a function across each element of a GLSL vector and produce a new GLSL vector:

```clojure
(defn map-over-vec4 [f v]
 (apply g/vec4 (for [i (range 4)] (f (g/part v i)))))
``` 

## Compiling to GLSL program strings






## License

Copyright © 2014 Kovas Boguta

Distributed under the Eclipse Public License either version 1.0 or (at
your option) any later version.
