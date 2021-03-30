# rclonemount

rclone mount is a rootless systemd integration allowing to seamlessly use one or several `rclone mount` commands as a systemd services with an optional directories and files structure cache warmup.

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

#### Global section

Used to set up global rclone options.

- `RCLONE_CHECKERS` I usually set up this to `2 * <nbCores>` (see [rclone documentation](https://rclone.org/docs/#checkers-n) for more details)
- `RCLONE_FAST_LIST` not actually used during rclone vfs operations but is quite effective for cache warmup if enabled (see [rclone documentation](https://rclone.org/docs/#fast-list) for more details)
- `RCLONE_LOG_LEVEL` controls the rclone verbosity (check the [logs section](#checking-logs) and the [rclone documentation](https://rclone.org/docs/#log-level-level) for more details)
- `RCLONE_TRANSFERS` I usually set up this to `<nbCores>` (see [rclone documentation](https://rclone.org/docs/#transfers-n) for more details)

#### Mount section

Used to set up mount rclone options.

- `RCLONE_ALLOW_OTHER` this is important to have it set to `true`, all virtual filesystem operations will be executed by rclone as `rclonemout` but the files ownership will actually be mapped to a different user/group for compartmentalization. By default FUSE only allow the mounting user to access the virtual filesystem. More on the different mapped user/group below.
- `RCLONE_ATTR_TIMEOUT` How often the kernel will refresh/ask rclone about file attributes. If the backend is not modified outside this mount, you can increase it to enhance performance (let's say `8760h`, see [rclone documentation](https://rclone.org/commands/rclone_mount/#attribute-caching) for more details.
- `RCLONE_CACHE_DIR` where rclone will store its cache. Example use `/var/lib/rclone` as base directory but make sure you have enough space or put it elsewhere (check [cache size]() option and see [rclone documentation](https://rclone.org/commands/rclone_mount/#vfs-file-caching)). Be carefull to target a dedicated sub folder for each configuration.
- `RCLONE_DEFAULT_PERMISSIONS` this is important to have it set to `true` as we allowed other users to access this FUSE mount, file security will be handled by kernel with regular rights (see [rclone documentation](https://rclone.org/commands/rclone_mount/#options) for more details).
- 

### systemd service unit template

This normally does not need configuration but values here will impact all yours configurations. You can tune them before registering your first configuration.

TODO

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
