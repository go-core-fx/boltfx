<a id="readme-top"></a>

[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![Apache 2.0 License][license-shield]][license-url]

<br />
<div align="center">
  <h1>template</h1>
  <p>
    Uber Fx module template — the starting point for a new <code>go-core-fx</code> library module, with a named logger, strict lint config, Makefile and CI workflow out of the box.
  </p>
  <a href="https://github.com/go-core-fx/template/issues/new">Report Bug</a>
  &middot;
  <a href="https://github.com/go-core-fx/template/issues/new">Request Feature</a>
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
	- [Starting a New Module](#starting-a-new-module)
	- [Module Setup](#module-setup)
	- [Adding Your Providers](#adding-your-providers)
- [Configuration](#configuration)
- [Development](#development)
	- [Make Targets](#make-targets)
	- [Continuous Integration](#continuous-integration)
- [Contributing](#contributing)
- [License](#license)
- [Acknowledgments](#acknowledgments)


---

## About The Project

`template` is the starter repository for a new library module in the [go-core-fx](https://github.com/go-core-fx) ecosystem. It contains a single `module.go` that registers an [Uber Fx](https://uber-go.github.io/fx/) module named after the module constant and decorates the injected `*zap.Logger` with that name, plus the tooling the sibling modules share:

- **One-file module skeleton** — `Module()` returns an `fx.Option` built with `fx.Module`, ready for your `fx.Provide` and `fx.Invoke` entries
- **Named logger** — every logger handed to code inside the module is `logger.Named(ModuleName)`, so log lines from different modules are distinguishable
- **Strict lint config** — a v2.14.0 golangci-lint configuration with a large strict-but-practical linter set
- **Makefile + CI** — formatting, linting, race-enabled tests, coverage, benchmarks, and a GitHub Actions workflow that runs them

This is a library, not an application: it reads **no environment variables** of its own and has no configuration file. `DEBUG` is read by the [`go-core-fx/logger`](https://github.com/go-core-fx/logger) dependency, and `TZ`/`DEBUG` are set for the development server by `make air` — see [Configuration](#configuration).

### Features

- **Uber Fx module** — `template.Module()` registers module `"template"` and decorates the `*zap.Logger` with the module name
- **Works with `logger.Module()`** — the decorator composes with `go-core-fx/logger`'s provider, or with any `*zap.Logger` your application provides
- **Zero application code** — the template ships only `module.go`; you add domain code, providers and options
- **Strict golangci-lint** — the same golden config used across the go-core-fx modules
- **CI pipeline** — lint, race-enabled tests with coverage upload to Codecov, and benchmark tracking with alerting

### Built With

- [![Go](https://img.shields.io/badge/Go-00ADD8?style=for-the-badge&logo=go&logoColor=white)](https://go.dev/)
- [![Uber Fx](https://img.shields.io/badge/Uber%20Fx-000000?style=for-the-badge)](https://uber-go.github.io/fx/)
- [![zap](https://img.shields.io/badge/go.uber.org%2Fzap-00ADD8?style=for-the-badge)](https://github.com/uber-go/zap)

<p align="right">(<a href="#readme-top">back to top</a>)</p>

---

## Getting Started

### Prerequisites

- [Go](https://go.dev/dl/) 1.26 or newer (`go.mod` requires `go 1.26.0`)
- [golangci-lint](https://golangci-lint.run/) v2 (the config targets v2.14.0) for `make lint` and `make fmt`
- GNU Make

### Installation

Clone this repository as the root of your new module:

```bash
git clone https://github.com/go-core-fx/template.git mymodule
cd mymodule
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

---

## Usage

### Starting a New Module

Rename the module path and the module constant, then drop the template remote and point the repository at your own:

```bash
go mod edit -module github.com/you/mymodule
git remote set-url origin git@github.com:you/mymodule.git
```

Then edit `module.go`: change the package name, `ModuleName` (it must match your directory name), and the import of `github.com/go-core-fx/logger` if you keep it.

### Module Setup

`Module()` composes with the logger module and any providers you add. A minimal application wiring:

```go
package main

import (
	"github.com/go-core-fx/logger"
	"github.com/go-core-fx/template"
	"go.uber.org/fx"
)

func main() {
	fx.New(
		logger.Module(),
		template.Module(),
	).Run()
}
```

Inside the module, `logger.WithNamedLogger(ModuleName)` decorates the `*zap.Logger`, so any constructor provided by the module logs under the module name:

```go
const ModuleName = "template"

func Module() fx.Option {
	return fx.Module(
		ModuleName,
		logger.WithNamedLogger(ModuleName),
		// fx.Provide(New),
	)
}
```

### Adding Your Providers

Replace the commented line with your own constructors:

```go
func Module() fx.Option {
	return fx.Module(
		ModuleName,
		logger.WithNamedLogger(ModuleName),
		fx.Provide(New),
		fx.Invoke(Register),
	)
}
```

<p align="right">(<a href="#readme-top">back to top</a>)</p>

---

## Configuration

This module itself reads no environment variables. The table below lists everything the repository and its direct dependency consume:

| Variable        | Read by                                       | Purpose                                                                                             | Format                                                                     | Default                                        | Example              |
| --------------- | --------------------------------------------- | --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- | ---------------------------------------------- | -------------------- |
| `DEBUG`         | `github.com/go-core-fx/logger` (`logger.New`) | Selects the zap development logger (console encoding, debug level) instead of the production logger | Boolean accepted by `strconv.ParseBool` (`1`, `t`, `T`, `TRUE`, `true`, …) | unset → treated as `false` → production logger | `1`                  |
| `TZ`            | `make air` (Makefile)                         | Timezone of the development server process                                                          | IANA timezone name                                                         | system local timezone                          | `UTC`                |
| `CODECOV_TOKEN` | Codecov GitHub Action (CI only)               | Uploads coverage reports from the `test` workflow job                                               | Opaque token, stored as a GitHub Actions secret                            | not set                                        | `your-codecov-token` |

See [`.env.example`](.env.example) for the commented copy-paste version.

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
- [go-core-fx/logger](https://github.com/go-core-fx/logger) — zap logger module the template decorates
- [golangci-lint](https://golangci-lint.run/) — linter suite behind `make lint`
- [Best-README-Template](https://github.com/othneildrew/Best-README-Template) — README structure

<p align="right">(<a href="#readme-top">back to top</a>)</p>

<!-- MARKDOWN LINKS & IMAGES -->
[contributors-shield]: https://img.shields.io/github/contributors/go-core-fx/template.svg?style=for-the-badge
[contributors-url]: https://github.com/go-core-fx/template/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/go-core-fx/template.svg?style=for-the-badge
[forks-url]: https://github.com/go-core-fx/template/network/members
[stars-shield]: https://img.shields.io/github/stars/go-core-fx/template.svg?style=for-the-badge
[stars-url]: https://github.com/go-core-fx/template/stargazers
[issues-shield]: https://img.shields.io/github/issues/go-core-fx/template.svg?style=for-the-badge
[issues-url]: https://github.com/go-core-fx/template/issues
[license-shield]: https://img.shields.io/github/license/go-core-fx/template.svg?style=for-the-badge
[license-url]: https://github.com/go-core-fx/template/blob/master/LICENSE
