# Home Assistant App: InfluxDB

[![GitHub Release][releases-shield]][releases]
![Project Stage][project-stage-shield]
[![License][license-shield]](LICENSE.md)

![Supports aarch64 Architecture][aarch64-shield]
![Supports amd64 Architecture][amd64-shield]

[![Github Actions][github-actions-shield]][github-actions]
![Project Maintenance][maintenance-shield]
[![GitHub Activity][commits-shield]][commits]

[![Discord][discord-shield]][discord]
[![Community Forum][forum-shield]][forum]

Scalable datastore for metrics, events, and real-time analytics.

[![Open this app in your Home Assistant instance.][my-badge]][my]

## About

[InfluxDB][influxdb] is an open source time series database optimized for
high-write-volume. It's useful for recording metrics, sensor data, events,
and performing analytics. It exposes an HTTP API for client interaction and is
often used in combination with Grafana to visualize the data.

![Chronograf in the Home Assistant Frontend](images/screenshot.png)

This app comes with Chronograf & Kapacitor pre-installed. These provide a
nice InfluxDB admin interface for managing your users, databases, data
retention settings, and let you peek inside the database using the Data
Explorer.

[:books: Read the full app documentation][docs]

## About this fork

The Home Assistant Community Add-ons project
([hassio-addons/addon-influxdb][upstream]) marked this add-on end-of-life in
August 2026 and removed it from their store, because it is built on
InfluxDB 1.x, which InfluxData has end-of-lifed. This repository continues
that work:

- Same app, same slug (`influxdb`), same configuration options.
- Published as pre-built multi-arch images at `ghcr.io/alpharesearch/influxdb`
  for `amd64` and `aarch64`, instead of being compiled on your machine.
- Home Assistant **app** format: add this repository under
  **Settings → Apps → ⋮ → Add repository**.

**Be aware:** InfluxDB 1.8.10, Chronograf and Kapacitor are upstream
end-of-life products. This fork keeps them installable, packaged and running
on current Home Assistant versions; it cannot provide security fixes that
InfluxData no longer ships. If you are starting from scratch today, weigh
[this app][upstream-announce] against a supported time series database.

## Installation

1. Add this repository to Home Assistant: **Settings → Apps → ⋮ (three dots)
   → Add repository**, and paste
   `https://github.com/alpharesearch/addon-influxdb`.
1. Install and start the **InfluxDB** app.
1. Check the logs, then click **OPEN WEB UI**.

Coming from the community add-on? See
[Migrating from the community add-on][migration] in the documentation.

## Support

Got questions?

You have several options to get them answered:

- The [Home Assistant Discord chat server][discord] for general Home
  Assistant discussions and questions.
- The Home Assistant [Community Forum][forum].
- Join the [Reddit subreddit][reddit] in [/r/homeassistant][reddit]

You could also [open an issue here][issue] GitHub. Please understand that this
is a small, best-effort fork: issues are welcome, guarantees are not.

## Contributing

This is an active open-source project. We are always open to people who want to
use the code or contribute to it.

We have set up a separate document containing our
[contribution guidelines](.github/CONTRIBUTING.md).

Thank you for being involved! :heart_eyes:

## Authors & contributors

This repository is a fork of the Home Assistant Community Add-on, written and
maintained from 2018 to 2026 by [Franck Nijhof][frenck] and
[the contributors of that project][contributors]. The original setup of this
fork is by [Markus Schulz][maintainer].

If you use this app and want to say thanks, sponsoring the upstream author is
a fine way to do it: [GitHub Sponsors][github-sponsors].

## License

MIT License

Copyright (c) 2018-2026 Franck Nijhof
Copyright (c) 2026 Markus Schulz (fork modifications)

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

[aarch64-shield]: https://img.shields.io/badge/aarch64-yes-green.svg
[amd64-shield]: https://img.shields.io/badge/amd64-yes-green.svg
[commits-shield]: https://img.shields.io/github/commit-activity/y/alpharesearch/addon-influxdb.svg
[commits]: https://github.com/alpharesearch/addon-influxdb/commits/main
[contributors]: https://github.com/hassio-addons/addon-influxdb/graphs/contributors
[discord-shield]: https://img.shields.io/discord/478094546522079232.svg
[discord]: https://discord.me/hassioaddons
[docs]: https://github.com/alpharesearch/addon-influxdb/blob/main/influxdb/DOCS.md
[forum-shield]: https://img.shields.io/badge/community-forum-brightgreen.svg
[forum]: https://community.home-assistant.io/t/home-assistant-community-add-on-influxdb/54491
[frenck]: https://github.com/frenck
[github-actions-shield]: https://github.com/alpharesearch/addon-influxdb/workflows/CI/badge.svg
[github-actions]: https://github.com/alpharesearch/addon-influxdb/actions
[github-sponsors]: https://github.com/sponsors/frenck
[influxdb]: https://www.influxdata.com/product/
[issue]: https://github.com/alpharesearch/addon-influxdb/issues
[license-shield]: https://img.shields.io/github/license/alpharesearch/addon-influxdb.svg
[maintenance-shield]: https://img.shields.io/maintenance/yes/2026.svg
[maintainer]: https://github.com/alpharesearch
[my-badge]: https://my.home-assistant.io/badges/supervisor_app.svg
[my]: https://my.home-assistant.io/redirect/supervisor_app/?app=influxdb&repository_url=https%3A%2F%2Fgithub.com%2Falpharesearch%2Faddon-influxdb
[migration]: https://github.com/alpharesearch/addon-influxdb/blob/main/influxdb/DOCS.md#migrating-from-the-community-add-on
[project-stage-shield]: https://img.shields.io/badge/project%20stage-production%20ready-brightgreen.svg
[reddit]: https://reddit.com/r/homeassistant
[releases-shield]: https://img.shields.io/github/release/alpharesearch/addon-influxdb.svg
[releases]: https://github.com/alpharesearch/addon-influxdb/releases
[upstream-announce]: https://github.com/hassio-addons/addon-influxdb
[upstream]: https://github.com/hassio-addons/addon-influxdb
