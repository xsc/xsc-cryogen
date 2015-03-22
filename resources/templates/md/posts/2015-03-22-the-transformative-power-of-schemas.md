{:title "The Transformative Power of Schemas",
 :layout :post,
 :tags ["clojure" "schema"]}

### Preface

You have probably encountered Prismatic's [schema library](https://github.com/prismatic/schema)
and maybe you already used it to make sure data flowing through your system has
the right shape and format. You might even have seen the following paragraph in
the library's README:

> As of version 0.2.0, Schema also supports schema-driven data transformations,
  with coercion being the main application fleshed out thus far. Coercion is
  like validation, except a schema-dependent transformation can be applied to
  the input data before validation.
  ([source](https://github.com/Prismatic/schema/blob/619707064e5ff96bb16eccfc713edd1dc2660c24/README.md#transformations-and-coercion))

And, alas, there is documentation on
[how to write custom transformations](https://github.com/Prismatic/schema/wiki/Writing-Custom-Transformations)
but it focuses on custom walkers, thus covering generic and mostly
schema-independent transformations while neglecting the usefulness of
__custom schema types__ and the localized transformations they represent.
