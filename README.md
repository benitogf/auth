# auth

[![Test](https://github.com/benitogf/auth/actions/workflows/tests.yml/badge.svg)](https://github.com/benitogf/auth/actions/workflows/tests.yml)

library to add jwt authentication to a katamari server

# creating rules and audits

    Define ad lib filters to send and receive criteria using key glob patterns, audit middleware

Using the default open setting is usefull while prototyping, but maybe not ideal to deploy as a public service.

jwt auth enabled with static routing server example:

```golang
package main

import (
  "net/http"
  "github.com/gorilla/mux"
  "github.com/benitogf/katamari"
  "github.com/benitogf/auth"
  "github.com/benitogf/level"
)

func main() {
  // auth storage (users)
	authStore := &level.Storage{Path: "/data/auth"}
	err := authStore.Start([]string{}, nil)
	if err != nil {
		log.Fatal(err)
  }
  // noop to capture the storage channel feed
  go katamari.WatchStorageNoop(authStore)
  // set the JWT tokens expiry
	auth := auth.New(
		auth.NewJwtStore(*key, time.Minute*10),
		authStore,
  )

  app := katamari.Server{}
  // set the server static mode (only defined filters and routes available)
  app.Static = true
  // perform audits on the request path/headers/referer
  // if the function returns false the request will return
  // status 401
	app.Audit = func(r *http.Request, auth *auth.TokenAuth) bool {
    if r.URL.Path == "/open" {
      return true
    }

    return false
  }
  app.Router = mux.NewRouter()
  katamari.OpenFilter(app, "open") // available withour token
  katamari.OpenFilter(app, "closed") // valid token required
  auth.Router(app)
  app.Start("localhost:8800")
  app.WaitClose()
}
```