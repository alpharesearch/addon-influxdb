# Home Assistant App: InfluxDB

[InfluxDB][influxdb] is an open source time series database optimized for
high-write-volume. It's useful for recording metrics, sensor data, events,
and performing analytics. It exposes an HTTP API for client interaction and is
often used in combination with Grafana to visualize the data.

This app comes with Chronograf & Kapacitor pre-installed as well. Which
gives you a nice InfluxDB admin interface for managing your users, databases,
data retention settings, and lets you peek inside the database using the
Data Explorer.

**Note**: _This app is a community fork. The Home Assistant Community Add-ons
project marked the original add-on end-of-life in August 2026 because
InfluxData end-of-lifed InfluxDB 1.x. See
[About this fork](#about-this-fork) for what that means for you._

## Installation

1. Add this repository to Home Assistant: **Settings → Apps → ⋮ (three dots)
   → Add repository**, and paste:
   `https://github.com/alpharesearch/addon-influxdb`

   Or click the button below, which does both steps at once.

   [![Open this app in your Home Assistant instance.][my-badge]][my]

1. Click the "Install" button to install the app.
1. Start the "InfluxDB" app.
1. Check the logs of the "InfluxDB" to see if everything went well.
1. Click the "OPEN WEB UI" button!

The app is shipped as a pre-built multi-arch image
(`ghcr.io/alpharesearch/influxdb`) for `amd64` and `aarch64`, so nothing is
compiled on your machine.

## Migrating from the community add-on

The community add-on and this app are two separate installations as far as the
Supervisor is concerned, even though they share the same slug. Installing this
app does **not** take over the data of an existing community add-on
installation, and a Supervisor backup of the old add-on cannot be restored
into this one.

Two facts shape the procedure:

- **The two cannot run at the same time with default settings.** Both publish
  `8086/tcp` and `8088/tcp` to the host, so whichever one starts second fails
  with `Port 8086/tcp is already in use`. (`80/tcp` is unmapped by default and
  the Ingress port is internal to each app, so neither of those clashes.)
- **The published `8088/tcp` port answers nothing.** InfluxDB 1.8 binds its
  backup and restore RPC service to `127.0.0.1:8088` inside the container, so
  there is nothing listening on the host mapping. Re-pointing that host port to
  something else does not help either.

### Recommended: hand the backup over `/share`

Both installations mount `share:rw`, so `/share` is the same directory in each
one. That allows a hand-off without either app ever needing the other's ports:

1. Take a full snapshot of Home Assistant first.
1. Open a shell on the Home Assistant host: `ha host login`, or SSH into Home
   Assistant OS as root. The Terminal & SSH add-on does not ship the InfluxDB
   client, so the commands below run inside the containers themselves. On a
   managed install, `docker exec` is a debugging escape hatch rather than a
   supported feature.
1. Find the two containers. Their names carry a per-repository hash prefix,
   which is why the old add-on and this app have different names:

   ```bash
   docker ps --format '{{.Names}}'
   ```

1. Back up from the old installation into the shared folder:

   ```bash
   docker exec -it <old-container> influxd backup -portable \
     -host 127.0.0.1:8088 /share/influx-migration
   ```

1. Stop **and uninstall** the old add-on. This is what frees `8086`/`8088`.
1. Install and start this app, then restore into it:

   ```bash
   docker exec -it <influxdb-container> influxd restore -portable \
     -host 127.0.0.1:8088 /share/influx-migration
   ```

1. Restart this app and check the Data Explorer shows your old data.

**Users are not part of a portable backup.** Recreate your `homeassistant` user
(and any others you had) under "InfluxDB Admin" → Users, and grant it access to
the restored databases, or Home Assistant will connect happily and write
nothing at all.

### Alternative: expose the RPC service during the migration

To run the commands from another machine instead, let the app bind the RPC
service to all interfaces through its own `envvars` option:

```yaml
envvars:
  - name: INFLUXDB_HTTP_RPC_BIND_ADDRESS
    value: "0.0.0.0:8088"
```

`influxd backup -host homeassistant.local:8088 …` then works from anywhere that
can reach the host. **That RPC service has no authentication at all**: anyone
who can reach the port can read and write every database. Turn it on only while
you migrate, do not publish `8088/tcp` beyond the host itself unless you are
sure you need it, and remove the `envvars` entry when you are done.

### Not migrating is a legitimate option too

There is no tested one-click upgrade path, and both installations occupy disk
while they co-exist, so check your free space first if your database is large.
If history matters less than simplicity, install this app fresh and let Home
Assistant write new data into it; the old add-on can simply be removed.

**Status of these instructions:** this app (6.0.0) has been installed and run on
Home Assistant OS 18.2, Core 2026.9.2, Supervisor 2026.09.0. The migration
procedure above is derived from InfluxDB's own backup/restore behaviour and the
app's configuration, and has not yet been walked end to end. If it does not
work exactly as written, please open an issue here rather than assuming you did
something wrong.

## Configuration

**Note**: _Remember to restart the app when the configuration is changed._

Example app configuration:

```yaml
log_level: info
auth: true
reporting: true
ssl: true
certfile: fullchain.pem
keyfile: privkey.pem
envvars:
  - name: INFLUXDB_HTTP_LOG_ENABLED
    value: "true"
```

**Note**: _This is just an example, don't copy and paste it! Create your own!_

### Option: `log_level`

The `log_level` option controls the level of log output by the app and can
be changed to be more or less verbose, which might be useful when you are
dealing with an unknown issue. Possible values are:

- `trace`: Show every detail, like all called internal functions.
- `debug`: Shows detailed debug information.
- `info`: Normal (usually) interesting events.
- `warning`: Exceptional occurrences that are not errors.
- `error`: Runtime errors that do not require immediate action.
- `fatal`: Something went terribly wrong. The app becomes unusable.

Please note that each level automatically includes log messages from a
more severe level, e.g., `debug` also shows `info` messages. By default,
the `log_level` is set to `info`, which is the recommended setting unless
you are troubleshooting.

### Option: `auth`

Enable or disable InfluxDB user authentication.

**Note**: _Turning this off is NOT recommended!_

### Option: `reporting`

This option allows you to disable the reporting of usage data to InfluxData.

**Note**: _No data from user databases is ever transmitted!_

### Option: `ssl`

Enables/Disables SSL (HTTPS) on the web interface.
Set it `true` to enable it, `false` otherwise.

**Note**: _This does NOT activate SSL for InfluxDB, just the web interface_

### Option: `certfile`

The certificate file to use for SSL.

**Note**: _The file MUST be stored in `/ssl/`, which is the default_

### Option: `keyfile`

The private key file to use for SSL.

**Note**: _The file MUST be stored in `/ssl/`, which is the default_

### Option: `envvars`

This allows the setting of Environment Variables to control InfluxDB
configuration as documented at:

<https://docs.influxdata.com/influxdb/v1.8/administration/config/#configuration-settings>

**Note**: _Changing these options can possibly cause issues with you instance.
USE AT YOUR OWN RISK!_

These are case sensitive.

#### Sub-option: `name`

The name of the environment variable to set which must start with `INFLUXDB_`

#### Sub-option: `value`

The value of the environment variable to set, set the Influx documentation for
full details. Values should always be entered as a string (even true/false values).

### Option: `leave_front_door_open`

Adding this option to the app configuration allows you to disable
authentication on the Web Terminal by setting it to `true` and leaving the
username and password empty.

**Note**: _We STRONGLY suggest, not to use this, even if this app is
only exposed to your internal network. USE AT YOUR OWN RISK!_

## Integrating into Home Assistant

The `influxdb` integration of Home Assistant makes it possible to transfer all
state changes to an InfluxDB database.

You need to do the following steps in order to get this working:

- Click on "OPEN WEB UI" to open the admin web-interface provided by this app.
- On the left menu click on the "InfluxDB Admin".
- Create a database for storing Home Assistant's data in, e.g., `homeassistant`.
- Go to the users tab and create a user for Home Assistant,
  e.g., `homeassistant`.
- Add "ALL" to "Permissions" of the created user, to allow writing to your
  database.

Now we've got this in place, add the following snippet to your Home Assistant
`configuration.yaml` file.

```yaml
influxdb:
  host: influxdb
  port: 8086
  database: homeassistant
  username: homeassistant
  password: <yourpassword>
  max_retries: 3
  default_measurement: state
```

Restart Home Assistant.

Apps are reachable from Home Assistant under their slug, which is why
`host: influxdb` works. If that does not resolve on your installation, the
`8086/tcp` port of this app is published to the host by default, so
`host: homeassistant.local` works as well. Older installations of the
community add-on used the host name `a0d7b954-influxdb`; that name belongs to
the old installation, not to this one.

You should now see the data flowing into InfluxDB by visiting the web-interface
and using the Data Explorer.

Full details of the Home Assistant integration can be found here:

<https://www.home-assistant.io/integrations/influxdb/>

## About this fork

The Home Assistant Community Add-ons project marked this add-on end-of-life in
August 2026 (see [upstream][upstream]), because it is built on InfluxDB 1.x,
which InfluxData no longer supports. This repository continues the work as a
Home Assistant **app**.

What this fork does: keeps the app building and running on current Home
Assistant versions, keeps the container base image patched, keeps the
packaging current, and takes care of the app-format migration.

What this fork cannot do: ship security fixes for InfluxDB 1.8.10, Chronograf
or Kapacitor, because their upstream vendors no longer release them for the
1.x line. InfluxDB 1.8.10 is the final 1.8 release. If none of this fits your
needs yet and you are starting from scratch today, consider a maintained time
series database instead.

## Known issues and limitations

- While the Chronograf interface supports SSL, currently, the app does
  not support having SSL on InfluxDB. This limitation is caused by
  Chronograf and we are still looking into a proper solution for this.
- The `armv7` architecture is no longer supported since version 6.0.0. The
  current app specification and base images cover `aarch64` and `amd64` only.
- While an installation of the community add-on still exists, this app cannot
  start at the same time: both publish host port `8086` by default. See
  [Migrating from the community add-on](#migrating-from-the-community-add-on).
- InfluxDB 1.x is end-of-life upstream; see [About this fork](#about-this-fork).

## Changelog & Releases

This repository keeps a change log using [GitHub's releases][releases]
functionality, and the app ships a [`CHANGELOG.md`][changelog] as well.

Releases are based on [Semantic Versioning][semver], and use the format
of `MAJOR.MINOR.PATCH`. In a nutshell, the version will be incremented
based on the following:

- `MAJOR`: Incompatible or major changes.
- `MINOR`: Backwards-compatible new features and enhancements.
- `PATCH`: Backwards-compatible bugfixes and package updates.

The version in `config.yaml` is also the tag of the published container image,
so releases have to be tagged with exactly that version, e.g., `6.0.0`.

## Support

Got questions?

You have several options to get them answered:

- The [Home Assistant Discord chat server][discord] for general Home
  Assistant discussions and questions.
- The Home Assistant [Community Forum][forum].
- Join the [Reddit subreddit][reddit] in [/r/homeassistant][reddit]

You could also [open an issue here][issue] GitHub. Please understand that this
is a small, best-effort fork: issues are welcome, guarantees are not.

## Authors & contributors

This repository is a fork of the Home Assistant Community Add-on, written and
maintained from 2018 to 2026 by [Franck Nijhof][frenck] and
[the contributors of that project][contributors]. The original setup of this
fork is by [Markus Schulz][maintainer].

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

[changelog]: CHANGELOG.md
[contributors]: https://github.com/hassio-addons/addon-influxdb/graphs/contributors
[discord]: https://discord.me/hassioaddons
[forum]: https://community.home-assistant.io/t/home-assistant-community-add-on-influxdb/54491
[frenck]: https://github.com/frenck
[influxdb]: https://www.influxdata.com/product/
[issue]: https://github.com/alpharesearch/addon-influxdb/issues
[maintainer]: https://github.com/alpharesearch
[my-badge]: https://my.home-assistant.io/badges/supervisor_app.svg
[my]: https://my.home-assistant.io/redirect/supervisor_app/?app=influxdb&repository_url=https%3A%2F%2Fgithub.com%2Falpharesearch%2Faddon-influxdb
[reddit]: https://reddit.com/r/homeassistant
[releases]: https://github.com/alpharesearch/addon-influxdb/releases
[semver]: https://semver.org/spec/v2.0.0.html
[upstream]: https://github.com/hassio-addons/addon-influxdb
