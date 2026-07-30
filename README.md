# auth

[![Test](https://github.com/benitogf/auth/actions/workflows/tests.yml/badge.svg)](https://github.com/benitogf/auth/actions/workflows/tests.yml)

JWT authentication library for the [ooo](https://github.com/benitogf/ooo) ecosystem.

## Features

- **JWT token authentication** with configurable expiry
- **User management** with registration and login
- **Gate middleware** for access control via `Router.Use()`
- **Compatible with ooo** server and filters

## Installation

```bash
go get github.com/benitogf/auth
```

## Usage

```go
package main

import (
    "log"
    "time"

    "github.com/benitogf/auth"
    "github.com/benitogf/ko"
    "github.com/benitogf/ooo"
    "github.com/benitogf/ooo/storage"
    "github.com/gorilla/mux"
)

func main() {
    // Auth storage (users), persisted with ko
    authStore := storage.New(storage.LayeredConfig{
        Memory:   storage.NewMemoryLayer(),
        Embedded: ko.NewEmbeddedStorage("./auth_data"),
    })
    authStore.Start(storage.Options{})
    go storage.WatchStorageNoop(authStore)

    // Create auth with JWT token expiry
    key := "your-secret-key"
    tokenAuth := auth.New(
        auth.NewJwtStore(key, time.Minute*10),
        authStore,
    )

    // Create server with static mode
    app := ooo.Server{Static: true}
    app.Router = mux.NewRouter()

    // Gate access with the auth middleware. ooo's Server.Audit hook was
    // removed in favor of Router.Use(): the middleware fans out to every
    // matched route, requiring a valid token for the data routes while
    // leaving the auth-managed routes (register, authorize, ...) open.
    // Pass extra open paths to exempt them.
    app.Router.Use(tokenAuth.Middleware("/open"))

    app.OpenFilter("open")   // Available without token
    app.OpenFilter("closed") // Requires valid token
    tokenAuth.Routes(&app)   // Add auth routes

    app.Start("localhost:8800")
    app.WaitClose()
}
```

## Auth Routes

| Method | Path | Description |
|--------|------|-------------|
| POST | `/register` | Register new user (open) |
| POST / PUT | `/authorize` | Login and get token / refresh an expired token |
| GET | `/available?account=` | Check if an account name is taken (open) |
| GET | `/profile` | Get the profile for the request token |
| POST | `/create` | Create a user (root/admin only) |
| GET | `/users` | List users (root/admin only) |
| GET / POST / DELETE | `/user/{account}` | Read, update or delete a user |
| PUT | `/password/{account}` | Update an account password |

## Related Projects

- [ooo](https://github.com/benitogf/ooo) - Main server library
- [ko](https://github.com/benitogf/ko) - Persistent storage adapter
- [ooo-client](https://github.com/benitogf/ooo-client) - JavaScript client
- [mono](https://github.com/benitogf/mono) - Full-stack boilerplate