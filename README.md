# rclonemount

rclone mount is a rootless systemd integration allowing to seamlessly use one or several `rclone mount` commands as a systemd services with an optionnal custom startup cache warmup script.

## Installation

### deb pkg

Just install it and go straight to configuration.

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

All configurations are stored in `/etc/rclonemount` in order for the service unit template to find them.

### Environment configuration

Using `example.env` as reference. Environment variables are used to configure RCLONE to avoid specifying these options on the command line which would lead us to edit the systemd unit service file and:

- force us to reload the unit into systemd `sytemctl daemon-reload`
- prevent us from mutualize the service unit file (`rclonemount@.service` is a [systemd service unit template](https://www.freedesktop.org/software/systemd/man/systemd.unit.html))

#### Command section

Used to configuration not rclone directly but the command line in particular.

- `SOURCE` reference the rclone backend you want to mount (it must exist in the corresponding `.conf` file, here `example.conf`)
- `DESTINATION` is the mount point. It must be a valid directory path and the `rclonemount` user (or group) must be have the right permission ont it and its parent directory.
- `WARMUPCACHE` will trigger a directories and files structure warmup during startup, set to `false` to deactive cache warmup.

#### Global

Used to set up global rclone options.

- ``
