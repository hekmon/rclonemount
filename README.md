# rclonemount

rclone mount is a rootless systemd integration allowing to seamlessly use one or several `rclone mount` commands as a systemd services with an optionnal custom startup cache warmup script.

## Installation

### deb pkg (WIP)

Just install it and go straight to configuration. It will require that you have installed [rclone as a deb pkg](https://rclone.org/downloads/) too.

### manually

- Copy `rclone_vfsrcwarmup` to `/usr/local/bin`
- Copy `rclonemount@.service` to `/etc/systemd/system`
- Change the path of the `ExecStartPost=` entry in `rclonemount@.service` from `/usr/bin/rclone_vfsrcwarmup` to `/usr/local/bin/rclone_vfsrcwarmup`
- Create the `/etc/rclonemount` dir
- Copy `example.conf` and `example.env` files to the `/etc/rclonemount` dir
- Execute the following commands as root for a future rootless usage:

```bash
useradd --home-dir /var/lib/rclonemount --create-home --system --shell /usr/sbin/nologin
chown rclonemount: /etc/rclonemount
chmod 750 /etc/rclonemount /var/lib/rclonemount
chmod 640 /etc/rclonemount/*
```

## Configuration

Using the example provided, a single configuration is composed of 2 files:

- the `.conf` file which corresponds to the rclone backend vanilla configuration (nothing new here)
- the `.env` file which allows to configure this particular configuration as a systemd service

All configurations are stored in `/etc/rclonemount` in order for the service unit template to find them on activation.

As both file contains sensitive informations (auth tokens and password) I recommend to always have them readable by `rclonemout` only. One time fix/enforce can be found below:

```bash
chown rclonemount: /etc/rclonemount/*
chmod 640 /etc/rclonemount/*
```

### Environment configuration

Using `example.env` as reference. Environment variables are used to configure RCLONE to avoid specifying these options on the command line which would lead us to edit the systemd unit service file and:

- force us to reload the unit into systemd `sytemctl daemon-reload` for every modification before issuing the actual service restart
- prevent us from mutualize the service unit file (`rclonemount@.service` is a [systemd service unit template](https://www.freedesktop.org/software/systemd/man/systemd.unit.html))

#### Command section

Used to configuration not rclone directly but the command line in particular.

- `SOURCE` reference the rclone backend you want to mount (it must exist in the corresponding `.conf` file, here `example.conf`)
- `DESTINATION` is the mount point. It must be a valid directory path and the `rclonemount` user (or group) must be have the right permission ont it and its parent directory.
- `WARMUPCACHE` will trigger a directories and files structure warmup during startup, set to `false` to deactive cache warmup.

#### Global

Used to set up global rclone options.

- `RCLONE_CHECKERS` I usually set up this to `2 * <nbCores>` (see [rclone documentation](https://rclone.org/docs/#checkers-n) for more details)
- `RCLONE_FAST_LIST` not actually used as part of rclone `vfs` but is quite effective for cache warmup if enabled (see [rclone documentation](https://rclone.org/docs/#fast-list) for more details)
- `RCLONE_LOG_LEVEL` controls the rclone verbosity (check the [logs section]() and the [rclone documentation](https://rclone.org/docs/#log-level-level) for more details)
- `RCLONE_TRANSFERS` I usually set up this to `<nbCores>` (see [rclone documentation](https://rclone.org/docs/#transfers-n) for more details)

## Usage

Once a configuration is ready, you can start manipulating the unit. Following examples will use `anotherconf` as configuration name, which means `anotherconf.conf` and `anotherconf.env` files both exist within `/etc/rclonemount` and are valid.

### First start

```bash
systemctl enable --now rclonemount@anotherconf.service
```

### Usual commands

```bash
systemctl start rclonemount@anotherconf.service
systemctl stop rclonemount@anotherconf.service
systemctl restart rclonemount@anotherconf.service
```

### Checking logs

#### Live logs

```bash
journalctl -f -u rclonemount@anotherconf.service
```

#### Full logs until now

```bash
journalctl -u rclonemount@anotherconf.service
```

### Purge directories and files structure cache

```bash
systemctl reload rclonemount@anotherconf.service
```

### Unregister

```bash
systemctl disable --now rclonemount@anotherconf.service
```

### Binding to another service
