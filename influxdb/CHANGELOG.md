# Changelog

All notable changes to this Home Assistant app are documented in this file.
The format is based on [Keep a Changelog][keepachangelog] and this project
adheres to [Semantic Versioning][semver].

## 6.0.2

### Changed

- Documentation only; the packaged software is unchanged since 6.0.0.
- The integration instructions no longer put connection settings in
  `configuration.yaml`. Home Assistant's InfluxDB integration is configured
  through the UI: its manifest sets `config_flow` and `single_config_entry`,
  `async_setup` only imports YAML into a config entry, `issue.py` files a
  `deprecated_yaml` repair issue with `breaks_in_ha_version="2026.9.0"`, and
  `async_setup_entry` builds the client from `entry.data`. Editing `host:`,
  `username:` or `password:` there therefore changes nothing on Core 2026.9 and
  later, which presents as writes silently stopping while Grafana keeps showing
  the old data.
- Documents what `configuration.yaml` still controls -- `max_retries`,
  `default_measurement`, `override_measurement`, `precision`, `measurement_attr`,
  `include`, `exclude`, `tags`, `tags_attributes`, `ignore_attributes` and the
  `component_config` overrides -- since `async_setup_entry` reads those from the
  YAML file on every setup.
- The URL field of the UI flow takes a scheme, which is what selects TLS, and
  the connection is validated before the entry is stored.
- Grafana is mentioned by name as a second client whose data source URL has to
  change when the app's alias changes, with `homeassistant.local` given as the
  alternative that does not.

## 6.0.1

### Changed

- Documentation only; the packaged software is identical to 6.0.0.
- The migration procedure was carried out end to end for the first time
  (community add-on 5.0.2 to this app on Home Assistant OS 18.2/amd64, about
  15MB) and has been rewritten around what that run actually required.
- `influxd backup` and `influxd restore` examples no longer show credential
  flags. They never had any: in 1.8 both commands contain no HTTP support at all
  and talk only to the RPC port, so `auth: true` does not concern them.
- Restore flags are given as 1.8 spells them (`-newdb`, `-rp`, `-newrp`,
  `-metadir`, `-datadir`, `-online`); `-new-database` and `-overwrite` do not
  exist.
- Recreating the Home Assistant user is now an explicit step including its
  `GRANT`, because a portable backup contains no users and a user without
  privileges authenticates and then rejects every write.
- Getting a shell: `ha host login` does not exist. Use the console of the HAOS
  VM, or Advanced SSH & Web Terminal with protection mode disabled, which is
  root-equivalent until protection mode is switched back on.
- Backups must not be staged under HAOS `/tmp` or `/`, which share the ~254MB
  system partition and are normally full. Note also that `docker cp` reports
  success even when the file it copied was truncated.
- The InfluxDB host name to give Home Assistant is `<repository-id>-influxdb`,
  not `influxdb`: Supervisor registers `App.hostname`, which is the slug with
  underscores replaced by dashes, as the container alias.

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
