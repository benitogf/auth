# auth

[![Test](https://github.com/benitogf/auth/actions/workflows/tests.yml/badge.svg)](https://github.com/benitogf/auth/actions/workflows/tests.yml)

JWT authentication library for the [ooo](https://github.com/benitogf/ooo) ecosystem.

## Features

- **JWT token authentication** with configurable expiry
- **User management** with registration and login
- **Audit middleware** for access control
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
    "net/http"
    "time"

    "github.com/gorilla/mux"
    "github.com/benitogf/auth"
    "github.com/benitogf/ko"
    "github.com/benitogf/ooo"
)

func main() {
    // Auth storage (users)
    authStore := &ko.Storage{Path: "/data/auth"}
    err := authStore.Start([]string{}, nil)
    if err != nil {
        log.Fatal(err)
    }
    go ooo.WatchStorageNoop(authStore)

    // Create auth with JWT token expiry
    key := "your-secret-key"
    tokenAuth := auth.New(
        auth.NewJwtStore(key, time.Minute*10),
        authStore,
    )

    // Create server with static mode
    app := ooo.Server{Static: true}

    // Audit middleware for access control
    app.Audit = func(r *http.Request) bool {
        if r.URL.Path == "/open" {
            return true
        }
        return tokenAuth.Verify(r) // Require valid token
    }

    app.Router = mux.NewRouter()
    app.OpenFilter("open")   // Available without token
    app.OpenFilter("closed") // Requires valid token
    tokenAuth.Router(&app)   // Add auth routes

    app.Start("localhost:8800")
    app.WaitClose()
}
```

## Auth Routes

| Method | Path | Description |
|--------|------|-------------|
| POST | `/register` | Register new user |
| POST | `/authorize` | Login and get token |
| GET | `/verify` | Verify token validity |

## Related Projects

- [ooo](https://github.com/benitogf/ooo) - Main server library
- [ko](https://github.com/benitogf/ko) - Persistent storage adapter
- [ooo-client](https://github.com/benitogf/ooo-client) - JavaScript client
- [mono](https://github.com/benitogf/mono) - Full-stack boilerplate