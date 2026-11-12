<a id="readme-top"></a>

[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![Apache 2.0 License][license-shield]][license-url]

<br />
<div align="center">
  <h1>boltfx</h1>
  <p>
    Uber Fx module for bbolt (BoltDB) — opens an embedded key/value database from a <code>bolt://</code> URL and ties its lifetime to the Fx application lifecycle.
  </p>
  <a href="https://github.com/go-core-fx/boltfx/issues/new">Report Bug</a>
  &middot;
  <a href="https://github.com/go-core-fx/boltfx/issues/new">Request Feature</a>
</div>

## Table of Contents
- [Table of Contents](#table-of-contents)
- [About The Project](#about-the-project)
	- [Features](#features)
	- [Built With](#built-with)
- [Getting Started](#getting-started)
	- [Prerequisites](#prerequisites)
	- [Installation](#installation)
- [Usage](#usage)
	- [Module Setup](#module-setup)
	- [Standalone Use](#standalone-use)
	- [Errors](#errors)
- [Configuration](#configuration)
	- [Database location](#database-location)
	- [Environment variables](#environment-variables)
- [Development](#development)
	- [Make Targets](#make-targets)
	- [Continuous Integration](#continuous-integration)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)

---

## About The Project

`boltfx` is the [go-core-fx](https://github.com/go-core-fx) module for [bbolt](https://github.com/etcd-io/bbolt) (BoltDB), the embedded transactional key/value store. It turns a `bolt://` URL into an open `*bbolt.DB`, hands the handle to the rest of the Fx graph, and closes it when the application stops — so consumers never have to manage the database file lifecycle themselves.

- **One-file module** — `boltfx.Module()` returns an `fx.Option` that provides the `*bbolt.DB` and registers the close hook
- **URL-based configuration** — the database location is a `bolt://` URL: relative paths (`bolt://data/app.db`) and absolute paths (`bolt:///var/lib/app.db`) both work
- **Fail-fast startup** — an insecure database file, a foreign URL scheme, a missing path or a file locked by another process is rejected before the application starts serving
- **Named logger** — the module decorates the injected `*zap.Logger` with `boltfx`, so its log lines are distinguishable from other modules

This is a library, not an application: it reads **no environment variables** of its own. `DEBUG` is read by the [`go-core-fx/logger`](https://github.com/go-core-fx/logger) dependency, and `TZ`/`DEBUG` are set for the development server by `make air` — see [Configuration](#configuration).

### Features

- **Fx lifecycle management** — `boltfx.Module()` provides `*bbolt.DB` to the graph and closes it on the Fx stop hook, with start/stop log lines carrying the database path
- **`bolt://` URL parsing** — a URL with a host yields a path relative to the working directory; a URL with an empty host yields an absolute path
- **Database file permission guard** — an existing database file that users other than the owner can read or write is rejected; files the module creates are mode `0600`
- **Bounded lock wait** — a database file locked by another process fails after 1 second instead of blocking startup
- **Sentinel errors** — every failure wraps one of four exported errors, so callers can classify with `errors.Is`
- **Strict golangci-lint** — the same golden config used across the go-core-fx modules

### Built With

- [![Go](https://img.shields.io/badge/Go-00ADD8?style=for-the-badge&logo=go&logoColor=white)](https://go.dev/)
- [![Uber Fx](https://img.shields.io/badge/Uber%20Fx-000000?style=for-the-badge)](https://uber-go.github.io/fx/)
- [![bbolt](https://img.shields.io/badge/go.etcd.io%2Fbbolt-417186?style=for-the-badge)](https://github.com/etcd-io/bbolt)
- [![zap](https://img.shields.io/badge/go.uber.org%2Fzap-00ADD8?style=for-the-badge)](https://github.com/uber-go/zap)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

---

## Getting Started

### Prerequisites

- [Go](https://go.dev/dl/) 1.26 or newer (`go.mod` requires `go 1.26.0`)
- [golangci-lint](https://golangci-lint.run/) v2 (the config targets v2.14.0) for `make lint` and `make fmt`
- GNU Make

### Installation

```bash
go get github.com/go-core-fx/boltfx
```

The module has no tagged release yet, so `go get` resolves a pseudo-version from the default branch until the first tag is cut.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

---

## Usage

### Module Setup

Add `boltfx.Module()` to your Fx options and provide a `boltfx.Config` anywhere in the graph — the module opens the database from it and closes it on stop:

```go
package main

import (
	"github.com/go-core-fx/boltfx"
	"github.com/go-core-fx/logger"
	bolt "go.etcd.io/bbolt"
	"go.uber.org/fx"
)

func main() {
	fx.New(
		logger.Module(),
		fx.Provide(func() boltfx.Config {
			return boltfx.Config{URL: "bolt://data/app.db"}
		}),
		boltfx.Module(),
		fx.Invoke(func(db *bolt.DB) {
			// use the database; it is closed when the application stops
		}),
	).Run()
}
```

Inside the module, `logger.WithNamedLogger("boltfx")` decorates the `*zap.Logger`, so the open, start and shutdown log lines are tagged with the module name:

```go
func Module() fx.Option {
	return fx.Module(
		"boltfx",
		logger.WithNamedLogger("boltfx"),
		fx.Provide(New),
		fx.Invoke(registerLifecycle),
	)
}
```

The parent directory of the database file is **not** created — it must exist before the application starts.

### Standalone Use

Outside an Fx application, `boltfx.New` opens the database without tying it to a lifecycle. The caller owns the returned handle and must close it:

```go
db, err := boltfx.New(boltfx.Config{URL: "bolt:///var/lib/app.db"}, logger)
if err != nil {
	return err
}
defer db.Close()
```

### Errors

Every failure wraps one of these exported sentinel errors — classify with `errors.Is`:

| Error              | Returned when                                                                            |
| ------------------ | ---------------------------------------------------------------------------------------- |
| `ErrEmptyURL`      | `Config.URL` is empty or carries no path (`bolt://`, `bolt:///`)                         |
| `ErrInvalidScheme` | `Config.URL` cannot be parsed or its scheme is not `bolt`                                |
| `ErrOpenFailed`    | The database file cannot be opened — locked by another process, or the parent is missing |
| `ErrInsecureFile`  | The database file exists with permissions that let users other than the owner access it  |

<p align="right">(<a href="#readme-top">back to top</a>)</p>

---

## Configuration

### Database location

The module's only setting is `boltfx.Config.URL`:

| URL form                       | Resolves to                                      |
| ------------------------------ | ------------------------------------------------ |
| `bolt://data/app.db`           | `data/app.db`, relative to the working directory |
| `bolt:///var/lib/app.db`       | `/var/lib/app.db`, absolute                      |
| empty, `bolt://` or `bolt:///` | rejected with `ErrEmptyURL`                      |
| anything else                  | rejected with `ErrInvalidScheme`                 |

Behavior worth knowing before production use:

- The database file is created with mode `0600`; an existing file with any other-user access is rejected with `ErrInsecureFile`
- A file locked by another process fails after a 1-second timeout with `ErrOpenFailed`
- The parent directory is not created

### Environment variables

This module itself reads no environment variables. The table below lists everything the repository and its direct dependency consume:

| Variable        | Read by                                       | Purpose                                                                                             | Format                                                                     | Default                                        | Example              |
| --------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------- | -------------------- |
| `DEBUG`         | `github.com/go-core-fx/logger` (`logger.New`) | Selects the zap development logger (console encoding, debug level) instead of the production logger | Boolean accepted by `strconv.ParseBool` (`1`, `t`, `T`, `TRUE`, `true`, …) | unset → treated as `false` → production logger | `1`                  |
| `TZ`            | `make air` (Makefile)                         | Timezone of the development server process                                                          | IANA timezone name                                                         | system local timezone                          | `UTC`                |
| `CODECOV_TOKEN` | Codecov GitHub Action (CI only)               | Uploads coverage reports from the `test` workflow job                                               | Opaque token, stored as a GitHub Actions secret                            | not set                                        | `your-codecov-token` |

<p align="right">(<a href="#readme-top">back to top</a>)</p>

---

## Development

### Make Targets

| Target           | Description                                                            |
| ---------------- | ---------------------------------------------------------------------- |
| `make all`       | Run all checks: `fmt`, `lint`, `coverage`                              |
| `make version`   | Display the current version (from `git describe`, defaults to `0.0.0`) |
| `make fmt`       | Run `go generate ./...`, then `golangci-lint fmt`                      |
| `make lint`      | Run `golangci-lint run --timeout=5m`                                   |
| `make test`      | Run tests with race detector, shuffle and atomic coverage              |
| `make coverage`  | Produce `coverage.out` and an HTML report (`coverage.html`)            |
| `make benchmark` | Run benchmarks into `benchmark.txt`                                    |
| `make air`       | Run the development server with air (`TZ=UTC DEBUG=1`)                 |
| `make deps`      | Download module dependencies                                           |
| `make gen`       | Run `go generate ./...`                                                |
| `make clean`     | Remove coverage output, benchmarks, `dist/` and `bin/`                 |
| `make help`      | List all available targets                                             |

### Continuous Integration

The [Go workflow](.github/workflows/go.yml) runs on pushes to `master` and on pull requests (path-filtered to Go sources, `go.mod`/`go.sum`, the lint config and the workflows themselves):

- **Lint** — golangci-lint with the repository config
- **Test** — race-enabled, shuffled tests with coverage uploaded to [Codecov](https://codecov.io/)
- **Benchmark** — runs `go test -bench`, keeps history in the Actions cache, uploads results when benchmarks exist and fails the job on a performance alert

A second workflow marks issues and pull requests stale after 7 days of inactivity and closes them 7 days later.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

---

## Contributing

Contributions are what make the open source community such an amazing place to learn, inspire, and create. Any contributions you make are **greatly appreciated**.

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

Please run `make all` before opening the pull request — it formats, lints and tests the change.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

---

## License

Distributed under the Apache License 2.0. See `LICENSE` for more information.

<p align="right">(<a href="#readme-top">back to top</a>)</p>

---

## Acknowledgments

- [Uber Fx](https://uber-go.github.io/fx/) — dependency injection framework
- [bbolt](https://github.com/etcd-io/bbolt) — embedded transactional key/value store
- [go-core-fx/logger](https://github.com/go-core-fx/logger) — zap logger module boltfx decorates
- [golangci-lint](https://golangci-lint.run/) — linter suite behind `make lint`
- [Best-README-Template](https://github.com/othneildrew/Best-README-Template) — README structure

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- MARKDOWN LINKS & IMAGES -->
[contributors-shield]: https://img.shields.io/github/contributors/go-core-fx/boltfx.svg?style=for-the-badge
[contributors-url]: https://github.com/go-core-fx/boltfx/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/go-core-fx/boltfx.svg?style=for-the-badge
[forks-url]: https://github.com/go-core-fx/boltfx/network/members
[stars-shield]: https://img.shields.io/github/stars/go-core-fx/boltfx.svg?style=for-the-badge
[stars-url]: https://github.com/go-core-fx/boltfx/stargazers
[issues-shield]: https://img.shields.io/github/issues/go-core-fx/boltfx.svg?style=for-the-badge
[issues-url]: https://github.com/go-core-fx/boltfx/issues
[license-shield]: https://img.shields.io/github/license/go-core-fx/boltfx.svg?style=for-the-badge
[license-url]: https://github.com/go-core-fx/boltfx/blob/master/LICENSE
