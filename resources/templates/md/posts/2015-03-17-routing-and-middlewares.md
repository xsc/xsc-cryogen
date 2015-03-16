{:title "Sandwiches, Routing & Middlewares", :layout :post,
 :tags ["clojure" "ronda" "routing" "library"]}

> __TL;DR:__ [ronda/routing](https://github.com/xsc/ronda-routing) promotes
  loosely coupled middlewares and is also delicious.

### Preface

Let's say you have a classic Clojure web service, consuming and producing JSON
data. If you've built it in a
[Ring](https://github.com/ring-clojure/ring)-compliant way, its handler part
might look something like the following:

```clojure
(def app
  (-> (compojure/routes
        (GET  "/people/:id" ...)
        (GET  "/people" ...)
        (POST "/people/:id" ...))
      (wrap-json)
      (wrap-something-else)))
```

So far, so good, but a problem arises: you're told that the `POST` data shall
actually be delivered via form parameters since there are clients that seem to
have problems with generating "that JSON thingy".

Your stack doesn't really account for this, crap. And who changes requirements
at this point in time? Well, whatever, let's get to it:

```clojure
(def app
  (-> (compojure/routes
        (wrap-json
          (compojure/routes
            (GET  "/people/:id" ...)
            (GET  "/people" ...)))
        (wrap-form-params
          (POST "/people/:id" ...)))
      (wrap-something-else)))
```

Ah, but we should add an ETag to the `GET /people/:id` endpoint, so it can be
cached (not the `GET /people` one, though, since it changes way too frequently).
But this one has to be applied after the response JSON was created, so there is
really no way around the following disaster:

```clojure
(def app
  (-> (compojure/routes
        (-> (GET "/people/:id" ...)
            (wrap-json)
            (wrap-etag))
        (wrap-json
          (GET "/people" ...))
        (wrap-form-params
          (POST "/people/:id" ...)))
      (wrap-something-else)))
```

I exaggeratingly call it a disaster, yet it's still quite manageable in small
applications (and proper namespacing goes a long way). However:

- the `wrap-json` middleware is duplicated across multiple handlers,
- you thus have to exchange it in multiple places if you want to switch
  implementations,
- you better pray that there are no more requirements that increase the
  heterogenity of your service.

Maybe it's just me but we should be doing better.

### Sandwiches

When my girlfriend asked me about my day I wanted to tell her about this thing I
was working on. But she doesn't care much for the technobabble-esque sprout of
words I'd produce were I to present it, for example, at work - thus, an analogy
was needed! And since it was over dinner, a delicious one at that!

> Imagine a sandwich dispenser with buttons delivering different variants of
  awesome sandwiches; let's say a French Ham and Cheese Sandwich or a Deviled Ham
  with Pickled Jalapeños one.

Disclaimer: I don't know a lot about sandwiches and they are also not that
common in Germany, so I basically just googled `sandwich recipes` and
random-walked across the first few results.

> Now, what happens exactly when you press the button? One possible outcome
  could certainly be that a small door opens somewhere deep inside the
  machine and your ready-made sandwich slides down into your arms.<br />
<br />
  In some other cases a tiny conveyor belt could be activated, a slice of bread
  is put onto it, then some slices of ham (French or Deviled), cheese and spicy
  stuff afterwards - but only if the sandwich requires it. <br />
<br />
  Basically, the moment you press the button, the sandwich recipe is sent to the
  conveyor belt and only the needed ingredients are added.

I then explained the maintenance advantages.

> If, for example, it turns out that sandwiches with ham on top of the chesse
sell better than the other way around you don't have to rearrange all the
ready-made stacks, just the order of the ingredients on the belt. Same goes if
you prefer a certain kind of cheese over the current one.

She nodded, smiled and we had carrot cake; and if you haven't caught on: The ham
is a middleware.

### What about the Cheese?

Also a middleware.

### Conditional Middlewares

Many existing middlewares are already triggered by some piece of information in
the request or response maps -
[ring-json](https://github.com/ring-clojure/ring-json), for example, only runs
if the `content-type` header is set appropriately. But there are much more
powerful triggers.

__Route Early ...__

If the routing decision is made at the very top of the stack and the actual
request processing happens on the very bottom, all middlewares inbetween have
access to where the current request _will be routed_:

```clojure
{:request-method :get
 :uri            "/people/123"
 :headers        {}
 :route-id       :person
 :route-params   {:id 123}}
```

This means that the available routes have to be decoupled from the handlers and
routing has to produce an identifier that can then be mapped back.
Libraries like [bidi](https://github.com/juxt/bidi) and
[clout](https://github.com/weavejester/clout) already make this possible.

__... Decide Later ...__

A middleware can then easily be wrapped to only run for certain routes:

```clojure
(defn endpoint-middleware
  [handler wrap-fn route-ids]
  (let [run? (comp (set route-ids) :route-id)
        wrapped-handler (wrap-fn handler)]
    (fn [request]
      (if (run? request)
        (wrapped-handler request)
        (handler request)))))
```

And your stack becomes not unlike the aforementioned sandwich conveyor belt:

```clojure
(def app
  (-> (dispatch-by-route-id
        {:person (fn [request] ...)
         ...})
      (endpoint-middleware wrap-json [:person :people])
      (endpoint-middleware wrap-form-params [:save-person])
      (endpoint-middleware wrap-etag [:person])
      (attach-route-id
        {"/person/:id" :person
         ...)))
```

There is exactly one place for each middleware where endpoints have to register
themselves. This makes it easy to activate them but can make it hard to grasp
which are actually active for a specific route.

__... or Let Someone Else Decide.__

If we are already able to make an independent routing decision there is no
reason why we shouldn't be able to attach more routing-based information to a
request - say, middleware IDs?

```clojure
(defn attach-middlewares
  [handler middlewares]
  {:pre [(map? middlewares)]}
  (let [lookup (comp set middlewares :route-id)]
    (fn [request]
      (-> request
          (assoc :route-middlewares (lookup request))
          (handler)))))
```

And middlewares can react:

```clojure
(defn routed-middleware
  [handler middleware-id wrap-fn]
  (let [run? #(contains? (:route-middlewares %) middleware-id)
        wrapped-handler (wrap-fn handler)]
    (fn [request]
      (if (run? request)
        (wrapped-handler request)
        (handler request)))))
```

Which leaves our stack with a clear separation of concerns:

```clojure
(def app
  (-> (dispatch-by-route-id
        {:person (fn [request] ...), ...})
      (routed-middleware :json        wrap-json)
      (routed-middleware :form-params wrap-form-params)
      (routed-middleware :etag        wrap-etag)
      (attach-middlewares
        {:person [:json :etag], ...})
      (attach-route-id
        {"/people/:id" :person})))
```

Note that it's still possible to use `endpoint-middleware` or wrap handlers
directly. Even `compojure/routes` can still have a place in this, although it
means making the routing decision twice.

### Introducing `ronda/routing`

__[ronda/routing](https://github.com/xsc/ronda-routing)__ is the library that
captures the concepts presented here. The main difference is that it
encapsulates all routing information (and arbitrary metadata) in a special data
structure - the so-called `RouteDescriptor` - which gets (together with the
routing decision) injected into incoming requests and can be used further down
the line for e.g.:

- __documentation__: the `RouteDescriptor` can contain descriptions/schemas/...
  which can be rendered in a user-friendly way,
- __path generation__: You want to link to some other endpoint? The descriptor
  knows how.
- __path matching__: You want to use references like `/people/123` as
 identifiers? The descriptor can tell you what data they represent.

There are implementations for [bidi](https://github.com/xsc/ronda-routing-bidi)
and [clout](https://github.com/xsc/ronda-routing-clout) and usage is very
similar to the previous examples:

```clojure
(require '[ronda.routing :as routing]
         '[ronda.routing.clout :as clout])

(def routes
  (-> (clout/descriptor
        {:person "/person/:id", ...})
      (routing/enable-middlewares
        :person [:json :etag]
        ...)))

(def app
  (-> (routing/compile-endpoints
        {:person (fn [request] ...), ...})
      (routing/routed-middlewares :json        wrap-json)
      (routing/routed-middlewares :form-params wrap-form-params)
      (routing/routed-middlewares :etag        wrap-etag)
      (routing/wrap-routing routes)))
```

Much more functionality remains to be explored (and I might do so in future
posts) but for now I'd refer anyone interested to the
[README](https://github.com/xsc/ronda-routing/blob/master/README.md) and the
[auto-generated documentation](https://xsc.github.io/ronda-routing/).

Also, for a related project, see
[ronda/schema](https://github.com/xsc/ronda-schema) - you might already know how
to properly integrate it. :)
