# Changelog
All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [0.9.1] - 2020-06-07
### Added
- `dlan_nginx_status_port` option
- CHANGELOG.md

### Changed
- Disabled the nginx `stub_status` 1337 port by default. It's now configurable
- `ipv6only` option on nginx's `listen` directives has been removed
- Improved documentation, should now be more readable

## [0.9.0] - 2020-06-02
### Added
- Initial release of the project

[Unreleased]: https://github.com/mvitale1989/dlan/compare/v0.9.1...HEAD
[0.9.1]: https://github.com/mvitale1989/dlan/compare/v0.9.0...v0.9.1
[0.9.0]: https://github.com/mvitale1989/dlan/releases/tag/v0.9.0
