title: Friend Authorization Checks and Compojure Routing
date: 2019-01-24
tags: tech clojure

Despite several good online resources, it's not necessarily obvious
how [friend](https://github.com/cemerick/friend)'s `wrap-authorize` 
interacts with [Compojure](https://github.com/weavejester/compojure) routing.

This set of routes handles `/4` incorrectly:

```clojure
(defroutes app-routes
  (GET "/1" [] (site-page 1))
  (GET "/2" [] (site-page 2))
  (friend/wrap-authorize (GET "/3" [] (site-page 3)) #{::user})
  (GET "/4" [] (site-page 4)))
```

Any attempt to route to `/4` for a user that doesn't have the `::user`
role will fail with the same error you would expect to (and do) get
from an unauthorized attempt to route to `/3`. The reason this happens
is that Compojure considers the four routes in the sequence in which
they are listed and `wrap-authorize` works by `throw`-ing out if there
is an authorization error (and aborting the routing entirely).

So, even though the code looks like the authorization check is
associated with `/3`, it's really associated with the point in
evaluation after `/2` is considered, but before `/3` or `/4`. So for
an unauthorized user of `/3`, Compojure never considers either the the
`/3` or `/4` routes. `/4` (and anything that might follow it) is hidden
behind the same security as `/3`.

This is what's meant when the documentation says to do the
authorization check *after* the routing and not *before*. Let the
route decide if the authorization check gets run and then your other
routes won't be impacted by authorization checks that don't apply.

What that looks like in code is this (with the `friend/authorize`
check *inside* the body of the route):

```clojure
(defroutes app-routes
  (GET "/1" [] (site-page 1))
  (GET "/2" [] (site-page 2))
  (GET "/3" [] (friend/authorize #{::user} (site-page 3)))
  (GET "/4" [] (site-page 4)))
```

The documentation does mention the use of `context` to help solve this
problem. Where that plays a role is when a set of routes need to be
hidden behind the same authorization check. But the essential point is
to check and enforce authorization only after you know you need to do it.
