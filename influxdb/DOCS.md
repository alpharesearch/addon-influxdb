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

If the community add-on is still installed, read this before you start. Two
things block a copy-paste migration, and neither is obvious:

- **Ports collide.** Both installations publish `8086/tcp` and `8088/tcp`, and
  Supervisor does not warn you in advance. While the community add-on still
  runs, give this app other host ports -- `8087` and `8089` are free -- and
  switch them back after you uninstall the old one. The containers listen on
  their own `8086` regardless, so this affects only how you and Home Assistant
  address them: during the migration point the InfluxDB **integration entry**
  (**Settings → Integrations → InfluxDB → ⋮ → Configure**) at this app, URL
  `http://<repository-id>-influxdb:8087`, and put it back afterwards. Nothing in
  this app's own configuration needs to change.
- **`influxd backup` reaches the daemon over port 8088, inside the container.**
  That service binds `127.0.0.1:8088` and, in InfluxDB 1.x, it cannot be told
  to listen elsewhere: the default `influxdb.conf` shipped by both the community
  add-on and this app writes no `rpc-bind-address` key, and 1.8.10 does not
  recognise the setting -- check `influxd help` before trusting a
  `rpc-bind-address` line, because it is silently ignored rather than rejected.
  So a command like `influxd backup -host homeassistant.local:8088` cannot
  succeed while the RPC port stays on loopback, which is its default.

### Get a shell that can run `influxd`

The Terminal & SSH add-on is a dead end here: it ships no InfluxDB client, and
its `ha` CLI has no host-shell command -- `ha host` only offers `info`, `logs`,
`options`, `reboot`, `reload`, `shutdown` and `disks`. Pick one of these:

1. **The console of the HAOS VM**, if Home Assistant OS runs as a guest on
   Proxmox or another hypervisor. Log in as root there and the `docker` CLI is
   available. Best option: nothing extra to install, nothing privileged left
   running afterwards.
1. **Advanced SSH & Web Terminal with Protection mode disabled.** With
   protection mode off that app gets access to the host's Docker instance.
   Understand the trade-off first: while protection mode is off, that app is
   root-equivalent on your installation, so switch it back on or uninstall it
   as soon as you are done.
1. **No host access at all**, by running the backup client from another machine
   against a temporarily exposed RPC port. See the last subsection.

### Back up and restore through `/share`

Walked end to end in September 2026 from community add-on 5.0.2 to this app
6.0.0 on Home Assistant OS 18.2 (amd64), 15 MB of data. Both installations
mount `share:rw`, so `/share` is the same directory in each of them and nothing
has to be copied between containers.

Find the container names first. They carry a per-repository hash prefix, which
is why the old add-on and this app differ even though both end in `influxdb`:

```console
$ docker ps | grep influx
2f117c38a17f  ghcr.io/alpharesearch/influxdb:6.0.0  ...  app_dd4ddeab_influxdb
d482e29956eb  ghcr.io/hassio-addons/influxdb/amd64:5.0.2  ...  app_a0d7b954_influxdb
```

Back up from the old container straight into `/share`:

```console
# docker exec app_a0d7b954_influxdb sh -c 'influxd backup -portable \
#   -db homeassistant -host 127.0.0.1:8088 /share/influx-migration'
```

Then restore it inside this app. Both installations can still be running while
you do: the command only ever touches the container's own loopback RPC port.

```console
# docker exec app_dd4ddeab_influxdb sh -c 'influxd restore -portable \
#   -host 127.0.0.1:8088 /share/influx-migration'
```

Four things about those two commands that cost somebody an afternoon:

- **Quote the command you hand to `docker exec`.** With `docker exec c du -sh
/data/*` the _host_ shell expands the glob before Docker runs anything, and
  `/data` does not exist there. Wrap the whole thing in `sh -c '...'`.
- **Neither command takes credentials**, and `auth: true` does not affect them:
  InfluxDB 1.8 `influxd backup` and `influxd restore` have no `net/http` and no
  username handling at all -- they speak only to the RPC port, which is exactly
  why that port is loopback-bound. There is no `-username` flag to pass.
- **Restoring needs the target flag names of 1.8**: `-host`, `-portable`, `-db`,
  `-newdb`, `-rp`, `-newrp`, `-metadir`, `-datadir`, `-online`. `-new-database`
  and `-overwrite` do not exist.
- **Never stage backups under `/tmp` or `/` on Home Assistant OS.** Those live
  on the ~254MB system partition, which is normally sitting at 100%. `/share` is
  on the data partition. A truncated copy is not detected: `docker cp` reported
  `Successfully copied 14.9MB` for a backup set whose last file it had cut off
  mid-write, so prefer `/share` or a direct pipe (`docker exec old tar -C /data
-cf - dir | docker exec -i new tar -C /data -xf -`) over copying.

Then recreate the user Home Assistant logs in with. **A portable backup never
contains users** -- restore in 1.8 has no concept of them -- and this app only
ever creates `chronograf` and `kapacitor`, from the secret in `/data/secret`,
which it derives from your Supervisor token. A fresh install therefore has no
`homeassistant` user and HA's writes fail with `unauthorized` until you make
one. Leave `/data/secret` alone.

```console
# docker exec -it app_dd4ddeab_influxdb sh -c 'influx -username chronograf \
#   -password "$(cat /data/secret)" \
#   -execute "CREATE USER homeassistant WITH PASSWORD '''<your-password>'''"'
# docker exec -it app_dd4ddeab_influxdb sh -c 'influx -username chronograf \
#   -password "$(cat /data/secret)" \
#   -execute "GRANT READ, WRITE ON homeassistant TO homeassistant"'
```

The `GRANT` is not optional: a user without privileges authenticates fine and
then fails every write. Verify before you remove anything:

```console
# docker exec -it app_dd4ddeab_influxdb sh -c 'influx -username chronograf \
#   -password "$(cat /data/secret)" -database homeassistant \
#   -execute "SHOW MEASUREMENTS" | head -20'
```

Once the measurements are there and Home Assistant's log is quiet about
InfluxDB: stop and uninstall the community add-on, set this app's Network ports
back to `8086` and `8088`, restart, and re-point the InfluxDB integration entry
at this app -- URL `http://<repository-id>-influxdb:8086`. Do not leave the URL
of the old installation in place: that alias belongs to the old container and
vanishes with it, and the failure is quiet rather than loud -- Grafana goes on
plotting the restored history while nothing new arrives. Then delete
`/share/influx-migration` -- a portable backup is an unauthenticated plaintext
copy of your entire history, and every app with `share` access can read it.

If Home Assistant was already writing into this app before the restore, do not
mix the two timelines; restore beside them and compare:

```console
# docker exec app_dd4ddeab_influxdb sh -c 'influxd restore -portable \
#   -host 127.0.0.1:8088 -db homeassistant -newdb homeassistant_old \
#   /share/influx-migration'
```

### Without a host shell

`influxd backup` and `influxd restore` also work as clients against a remote RPC
endpoint. Fetch the same version the app ships (1.8.10) and use the `influxd`
binary inside:

- amd64: <https://dl.influxdata.com/influxdb/releases/influxdb-1.8.10_linux_amd64.tar.gz>
- arm64: <https://dl.influxdata.com/influxdb/releases/influxdb-1.8.10_linux_arm64.tar.gz>

Expose the RPC service of each installation in turn through the app's `envvars`
option:

```yaml
envvars:
  - name: INFLUXDB_HTTP_RPC_BIND_ADDRESS
    value: "0.0.0.0:8088"
```

Only one of the two installations can hold host port `8088` at a time, so leave
it on the old add-on while both exist and give this app `8089` or nothing.

**That RPC service has no authentication at all**: anyone who can reach the port
can read and write every database. Enable it only while you migrate, do not
publish `8088/tcp` beyond the host itself unless you are sure you need it, and
remove the `envvars` entry when you are done.

### Not migrating is a legitimate option too

There is no tested one-click upgrade path, and both installations occupy disk
while they co-exist, so check your free space first if your database is large.
If history matters less than simplicity, install this app fresh and let Home
Assistant write new data into it; the old add-on can simply be removed.

**Status of these instructions:** this app has been installed and run on Home
Assistant OS 18.2, Core 2026.9.2, Supervisor 2026.09.0, and the migration above
was performed on a real installation of that combination -- community add-on
5.0.2 to this app, about 15MB of data, confirmed afterwards in Grafana. Every
statement in this document was checked against the software itself rather than
from memory, so if your run diverges, open an issue here instead of assuming
you did something wrong.

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

Create the database and the user first, in this app's admin interface:

- Click on "OPEN WEB UI" to open the admin web-interface provided by this app.
- On the left menu click on the "InfluxDB Admin".
- Create a database for storing Home Assistant's data in, e.g., `homeassistant`.
- Go to the users tab and create a user for Home Assistant, e.g.,
  `homeassistant`, and add "ALL" to its "Permissions". Skipping that grant is a
  classic mistake: such a user authenticates happily and then rejects every
  single write.

The connection itself belongs to the **integration UI**, not to
`configuration.yaml`. Go to **Settings → Integrations → Add Integration →
InfluxDB**, choose **InfluxDB v1**, and fill in:

- **URL**: `http://dd4ddeab-influxdb:8086`. The scheme decides TLS, so use
  `https://` when the app's `ssl` option is on.
- **Verify SSL**: off unless Home Assistant trusts the certificate.
- **Database**: `homeassistant`.
- **Username** / **Password**: the user created above.

Home Assistant tests the connection before it stores the entry, so a wrong host
or password fails visibly there instead of quietly later.

**Why not YAML for the connection?** In
`homeassistant/components/influxdb`, the manifest declares `config_flow: true`
and `single_config_entry: true`, `async_setup` exists only to import YAML into a
config entry, `issue.py` files a `deprecated_yaml` repair issue carrying
`breaks_in_ha_version="2026.9.0"`, and `async_setup_entry` builds the client
from `entry.data`. On Core 2026.9 and later that means editing `host:`,
`username:` or `password:` under `influxdb:` in `configuration.yaml` changes
nothing whatsoever -- the stored integration entry is what counts. When moving an
existing installation to a new InfluxDB host, change the URL under
**Settings → Integrations → InfluxDB → ⋮ → Configure**, then delete those keys
from `configuration.yaml` to close the repair issue. Symptom of getting this
wrong: Home Assistant's own dashboards stop updating while Grafana keeps
plotting the old history, because the old host name simply no longer resolves.

What `configuration.yaml` still controls is the _content_ of what is written.
`async_setup_entry` reads these keys from the YAML file on every setup, because
the UI has no equivalent for them:

```yaml
influxdb:
  max_retries: 3
  default_measurement: state
  include:
    entities:
      - sensor.radon_level
      - switch.radon_fan
```

The same applies to `precision`, `measurement_attr`, `override_measurement`,
`exclude`, `tags`, `tags_attributes`, `ignore_attributes` and the
`component_config` overrides. Restart Home Assistant after changing any of them:
they are read when the integration is set up, not live.

The host name in that URL is **`<repository-id>-influxdb`**, _not_ `influxdb`.
Home Assistant Core and this app share the internal `hassio` Docker network, so
this hop involves no published ports at all, and Supervisor registers
`App.hostname` -- the app slug with underscores replaced by dashes
(`supervisor/apps/model.py`) -- as both the container hostname and its DNS alias
(`supervisor/docker/app.py`). An app installed from a custom repository gets that
repository's id as the first half of its slug, the prefix you also see in
`docker ps` as `app_dd4ddeab_influxdb`. That name changes if you uninstall this
app and install it from a different repository URL, so confirm it with
`docker ps` rather than copying a value from a guide; `a0d7b954-influxdb`, used
by community add-on installations, belongs to that installation and stops
resolving once it is removed. If the alias does not resolve on your installation,
`8086/tcp` is published to the host by default and
`http://homeassistant.local:8086` works instead -- the sturdier choice for other
clients such as Grafana, whose data source URL otherwise has to be edited every
time this app is reinstalled.

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
