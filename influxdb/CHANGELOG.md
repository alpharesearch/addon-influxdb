# Changelog

All notable changes to this Home Assistant app are documented in this file.
The format is based on [Keep a Changelog][keepachangelog] and this project
adheres to [Semantic Versioning][semver].

## 6.0.0

### Changed

- This app is now published as a Home Assistant **app** (the new name for
  Home Assistant add-ons) from its own repository, with pre-built multi-arch
  container images at `ghcr.io/alpharesearch/influxdb`. Before this version,
  the app source came from the Home Assistant Community Add-ons project,
  which marked the add-on end-of-life in August 2026.
- Dropped the `armv7` architecture. Supported architectures are now
  `aarch64` and `amd64`, which is what the current app specification and
  base images cover.
- Base image updated from `ghcr.io/hassio-addons/debian-base` 7.7.1 to
  7.8.2, and referenced as a single multi-arch reference instead of one
  reference per architecture.

### Added

- Option descriptions shown in the app configuration UI, via
  `translations/en.yaml`.

### Fixed

- The image was no longer buildable: the pinned Debian packages
  (`nginx=1.22.1-9`, `libnginx-mod-http-lua=1:0.10.23-1`) had been superseded
  by their `+deb12uN` security revisions in the Debian 12 archive and could no
  longer be resolved. Pins updated to `nginx=1.22.1-9+deb12u9` and
  `libnginx-mod-http-lua=1:0.10.23-1+deb12u1`, and Renovate is now pointed at
  `debian_12` (the base image is Debian 12 "bookworm", the pinning config still
  said `debian_11`).

## Upstream

This fork is based on version `5.0.2` of the Home Assistant Community Add-on
`influxdb`, the last release published before the upstream project marked the
add-on end-of-life. InfluxDB 1.8.10, Chronograf 1.10.2 and Kapacitor 1.5.9
are unchanged from that release.

[keepachangelog]: https://keepachangelog.com/en/1.1.0/
[semver]: https://semver.org/spec/v2.0.0.html
