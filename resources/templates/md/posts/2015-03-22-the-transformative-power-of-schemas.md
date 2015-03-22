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

Custom wakers are wonderful for generic and mostly schema-independent
transformations. Localized processing, however, is a different beast.

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
on key names. Whenever we encounter `:x`, `:y` or `:z-limit` the value will be
capped:

```clojure
(defn cap-walker
  [schema]
  (let [walk (s/walker schema)]
    (if (and (instance? schema.core.MapEntry schema)
             (#{:x :y :z-index} (:kspec schema)))
      (fn [x]
        (let [[k v] (walk x)]
          [k (min v 1000)]))
      walk)))
```

This can be run on a value using `start-walker`, producing an `ErrorContainer`
if validation fails:

```clojure
((s/start-walker cap-walker point-schema)
 {:pos {:x 1001, :y 2}
  :z-index 2000
  :value 0})
;; => {:value 0, :z-index 1000, :pos {:y 2, :x 1000}}
```

We are now validating and transforming in one go but expressiveness is still
lacking - you can't look at the schema and immediately see that some elements
are treated differently from others. And if upper limits and capping behaviour
are not part of the data description - and thus schemas - what is?

### Solution #3: How I learned to stop worrying and love the Schema.


