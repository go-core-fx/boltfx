# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### New Features

- **Fx module** — `boltfx.Module()` provides an open `*bbolt.DB` to the Fx graph and closes it on the application stop hook, so consumers never manage the database file lifecycle themselves.
- **`bolt://` URL configuration** — `boltfx.Config.URL` locates the database: a URL with a host opens a path relative to the working directory (`bolt://data/app.db`), a URL with an empty host opens an absolute path (`bolt:///var/lib/app.db`).
- **Database file permission guard** — an existing database file that users other than the owner can read or write is rejected at startup; files created by the module use mode `0600`.
- **Bounded lock wait** — a database file locked by another process fails startup after 1 second instead of blocking.
- **Sentinel errors** — `ErrEmptyURL`, `ErrInvalidScheme`, `ErrOpenFailed` and `ErrInsecureFile` cover every failure mode and are matchable with `errors.Is`.
