{:title "Schemas & Transformations", :layout :post, :tags ["clojure" "schema"]}

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

You might want to add some data validation to the `POST /people/:id` route which
can easily be accomplished:

```clojure
(POST "/people/:id" request
  (or (maybe-invalid request)
      (handle request)))
```

So far, so good. It doesn't really matter how `maybe-invalid` produces an error
(e.g. 401) response since the problem I'll now leave you with arises somewhere
else: the `POST` route shall now no longer take JSON but will expect it's data
in `:form-params`.

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
