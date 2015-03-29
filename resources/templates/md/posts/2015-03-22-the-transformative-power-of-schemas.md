{:title "The Transformative Power of Schemas",
 :layout :post,
 :tags ["clojure" "schema"]}

You have probably encountered Prismatic's [schema library](https://github.com/prismatic/schema)
and maybe you already used it to make sure data flowing through your system has
the right shape and format. You might even have seen the following paragraph in
the library's [README](https://github.com/Prismatic/schema/blob/619707064e5ff96bb16eccfc713edd1dc2660c24/README.md#transformations-and-coercion):

> As of version 0.2.0, Schema also supports schema-driven data transformations,
  with coercion being the main application fleshed out thus far. Coercion is
  like validation, except a schema-dependent transformation can be applied to
  the input data before validation.

Coercion works by creating a [custom walker](https://github.com/Prismatic/schema/wiki/Writing-Custom-Transformations)
that wraps each subschema with an optional preprocessing step, converting
between value types if possible and necessary. The available coercions are
injected via a function that generates a coercer based on the desired output
schema (or a map that does the same thing):

```
(def simple-matcher
  {s/Str str
   s/Int #(Long/parseLong %)})

(def coerce-and-check
  (schema.coerce/coercer
    {:a s/Str, :b s/Int}
    simple-matcher))

(coerce-and-check {:a 1, :b "2"})
;; => {:a "1", :b 2}
```

Custom walkers are wonderful for generic and schema-independent transformations.
And I want to know how useful they are for localized ones.

### The Problem

Let's say you have a value matching the following simple schema:

```clojure
(def point-schema
  {:pos     {:x s/Int, :y s/Int}
   :z-index s/Int
   :value   s/Int})
```

All of `:x`, `:y` and `:z-index` shall be capped at `1000` - meaning that if
they exceed that range they will be set to the maximum value.

There are multiple ways of handling this requirement.

### Solution #1: Go with the Flow

Adding an additional processing step within the data flow might produce the
most intuitive solution:

```clojure
(defn cap-value
  [path data]
  (update-in data path #(min % 1000)))

(defn process-point
  [point]
  (->> point
       (s/validate point-schema)
       (cap-value [:z-index])
       (cap-value [:pos :x])
       (cap-value [:pos :y])
       (do-something)))
```

There are problems with this approach, mainly maintenance-wise: What if there
are `:x`, `:y` and `:z-limit` values across different schemas? What if they need
different limits? You'll basically write the same functions over and over again,
and you'll have to adjust all of them if something changes.

### Solution #2: Walk the Walk

To illustrate how a custom walker could solve this problem we'll tackle it based
on a marker schema defined as follows:

```clojure
(defrecord Transform [schema f]
  s/Schema
  (walker [_]
    (s/walker schema))
  (explain [_]
    (s/explain schema)))
```

Whenever we encounter such a value we'll validate it and then apply the stored
function:

```clojure
(defn transform-walker
  [schema]
  (if (instance? Transform schema)
    (let [walk (s/walker schema)
          f (:f schema)]
      (fn [x]
        (let [result (walk x)]
          (if-not (schema.utils/error? result)
            (f result)
            result))))
    (s/walker schema)))
```

This allows us to create a schema representing a generic upper limit:

```clojure
(defn capped
  [max-value]
  (->Transform s/Int #(min % max-value)))
```

And our point schema ends up being:

```clojure
(def pos-schema     (capped 1000))
(def z-index-schema (capped 1000))

(def point-schema
  {:pos     {:x pos-schema, :y pos-schema}
   :z-index z-index-schema
   :value   s/Int})
```

The great advantage of this approach is that we can utilize different walkers:
If we only want to validate input data, a call to `s/validate` will do so; if we
want to coerce we can use the built-in coercion walker; and if we desire to
apply our custom transformations we will prevail:

```clojure
((s/start-walker transform-walker point-schema)
 {:pos {:x 1001, :y 2}
  :z-index 2000
  :value 0})
;; => {:value 0, :z-index 1000, :pos {:y 2, :x 1000}}
```

But walkers are not easily composable since they are created based on the schema
of the original input data. If one of the walkers changes the shape of the data
(and this is what we're talking about here) subsequent ones might fail when
attempting validation.

### Solution #3: How I learned to stop worrying and love the Schema.

I'd argue that within a fixed processing pipeline the flexibility of being able
to choose between different walkers is not a necessary requirement - but being
able to easily compose different processing steps is. By definition.

As we saw above, it is possible to attach processing logic to a schema - this
time, however, the schema will do the processing itself:

```clojure
(defrecord Fn [f input output explain-name explain-args]
  s/Schema
  (walker [_]
    (let [iwalk (s/subschema-walker input)
          owalk (s/subschema-walker output)]
      (fn [value]
        (let [in (iwalk value)]
          (if-not (schema.utils/error? in)
            (owalk (f in))
            in)))))
  (explain [_]
    (list* explain-name
           [(s/explain input) '=> (s/explain output)]
           explain-args)))
```

This opens the doors for concise descriptions of processing units. (I'd even
argue that this is what `schema.core/fn` et al should produce - maybe
implementing the necessary interfaces to behave like a normal function - since
it allows for a declarative way of generating localized transformations.) Note
that the actual processing needs the following convenience function:

```clojure
(defn run
  [schema value]
  ((s/start-walker s/walker schema) value))
```

But back to our omnipresent and all-important capping example:

```clojure
(defn capped
  [max-value]
  (->Fn #(min % max-value) s/Int s/Int 'limit [max-value]))
```

Bow, limit-ignoring values!

```clojure
(run (capped 1000) 1500)
;; => 1000

(run
  {:pos {:x (capped 200), :y (capped 100)}}
  {:pos {:x 500, :y 500}})
;; => {:pos {:x 200, :y 100}}
```

And schemas are composable, as we know:

```clojure
(run (s/both (capped 500) (capped 200)) 1000)
;; => 200
```

(Although it would probably be better to create an `FnComp` schema instead of
relying on `s/both` sequentially running the schemas.)

Processing within a schema and processing in a custom walker are orthogonal
concepts. This means that there is e.g. no reason why coercion could not be used
with the above `Fn` schema - just coerce the input values, call the function and
finally coerce its output.

### Discussion

Okay, here's the thing: While certainly possible, I'm not sure this is what
schemas were made for.
