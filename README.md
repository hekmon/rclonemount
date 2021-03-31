# rclonemount

rclone mount is a rootless systemd integration allowing to seamlessly use one or several `rclone mount` commands as a systemd services with an optional directories and files structure cache warmup.

rootless means that root privileges will not be used during run (as they should not) and rclone mount execution will be compartimented on a dedicated user (`rclonemount`) while the files and directories can be mapped to specific and different UID/GID with valid linux permissions enforced by the kernel. In addition to avoid using root, this means that you can provide vfs to your services without giving them the ability to read your rclone backend credentials (as config file and execution will be compartimented on a dedicated user).

The directories and files structure cache warmup allows to fully scan the structure of your backend and have it in memory before the systemd unit is actually ready allowing subsequent services whichs depends on the mount and its VFS to start with the vfs structure already in memory.

- [rclonemount](#rclonemount)
  - [Installation](#installation)
    - [deb pkg (WIP)](#deb-pkg-wip)
    - [manually](#manually)
  - [Configuration](#configuration)
    - [Environment configuration](#environment-configuration)
      - [Command section](#command-section)
      - [Global section](#global-section)
      - [Mount section](#mount-section)
      - [RC section](#rc-section)
    - [systemd service unit template](#systemd-service-unit-template)
  - [Usage](#usage)
    - [First start](#first-start)
    - [Usual commands](#usual-commands)
    - [Checking logs](#checking-logs)
      - [Live logs](#live-logs)
      - [Full logs until now](#full-logs-until-now)
    - [Purge directories and files structure cache](#purge-directories-and-files-structure-cache)
    - [Unregister](#unregister)
    - [Binding to another service](#binding-to-another-service)

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

As both file contains sensitive informations (auth tokens and password) I recommend to always have them readable by `rclonemount` only. One time fix/enforce can be found below:

```bash
chown rclonemount: /etc/rclonemount/*
chmod 640 /etc/rclonemount/*
```

### Environment configuration

Using `example.env` as reference. Environment variables are used to configure RCLONE to avoid specifying these options on the command line which would lead us to edit the systemd unit service file and:

- force us to reload the unit into systemd `sytemctl daemon-reload` for every modification before issuing the actual service restart
- prevent us from mutualize the service unit file (`rclonemount@.service` is a [systemd service unit template](https://www.freedesktop.org/software/systemd/man/systemd.unit.html))

#### Command section

Used to configure not rclone directly but the command line in particular.

- `SOURCE` reference the rclone backend you want to mount (it must exist in the corresponding `.conf` file, here `example.conf`)
- `DESTINATION` is the mount point. It must be a valid directory path and the `rclonemount` user (or group) must be have the right permission ont it and its parent directory.
- `WARMUPCACHE` will trigger a directories and files structure warmup during startup, set to `false` to deactive cache warmup.

#### Global section

Used to set up global rclone options.

- `RCLONE_CHECKERS` I usually set up this to `2 * <nbCores>` (see [rclone documentation](https://rclone.org/docs/#checkers-n) for details)
- `RCLONE_FAST_LIST` not actually used during rclone vfs operations but is quite effective for cache warmup if enabled (see [rclone documentation](https://rclone.org/docs/#fast-list) for details)
- `RCLONE_LOG_LEVEL` controls the rclone verbosity (check the [logs section](#checking-logs) and the [rclone documentation](https://rclone.org/docs/#log-level-level) for details)
- `RCLONE_TRANSFERS` I usually set up this to `<nbCores>` (see [rclone documentation](https://rclone.org/docs/#transfers-n) for details)

#### Mount section

Used to set up mount rclone options.

- `RCLONE_ALLOW_OTHER` this is important to have it set to `true`, all virtual filesystem operations will be executed by rclone as `rclonemout` but the files ownership will actually be mapped to a different user/group for compartmentalization. By default FUSE only allow the mounting user to access the virtual filesystem. More on the different mapped user/group below.
- `RCLONE_ATTR_TIMEOUT` How often the kernel will refresh/ask rclone about file attributes. If the backend is not modified outside this mount, you can increase it to enhance performance (let's say `8760h`, see [rclone documentation](https://rclone.org/commands/rclone_mount/#attribute-caching) for details.
- `RCLONE_CACHE_DIR` where rclone will store its cache. Example use `/var/lib/rclone` as base directory but make sure you have enough space or put it elsewhere (check `RCLONE_VFS_CACHE_MAX_SIZE` option and see [rclone documentation](https://rclone.org/commands/rclone_mount/#vfs-file-caching)). Be carefull to target a dedicated sub folder for each configuration.
- `RCLONE_DEFAULT_PERMISSIONS` this is important to have it set to `true` as we allowed other users to access this FUSE mount, file security will be handled by kernel with regular rights (see [rclone documentation](https://rclone.org/commands/rclone_mount/#options) for details).
- `RCLONE_DIR_CACHE_TIME` directories structure cache (see [rclone documentation](https://rclone.org/commands/rclone_mount/#vfs-directory-cache) for details).
- `RCLONE_DIR_PERMS` default permissions for mounted directories, this is important as we have activated `allow-other` and `default-permissions`. Example value `0750` will allow the mapped UID to read/write/traverse and GID to read/traverse but will forbid any actions by others (see [rclone documentation](https://rclone.org/commands/rclone_mount/#options) for details).
- `RCLONE_FILE_PERMS` default permissions for mounted files, this is important as we have activated `allow-other` and `default-permissions`. Example value `0640` will allow the mapped UID to read/write and GID to read but will forbid any actions by others (see [rclone documentation](https://rclone.org/commands/rclone_mount/#options) for details).
- `RCLONE_GID` the GID to map the directories and files to. On this example, GID only has read right, so it could be a group (let's say `mediaservers` containing emby and plex users) which you want to access your files but deny them write (see [rclone documentation](https://rclone.org/commands/rclone_mount/#options) for details).
- `RCLONE_POLL_INTERVAL` request the backend for external changes (see [rclone documentation](https://rclone.org/commands/rclone_mount/#vfs-directory-cache) for details)
- `RCLONE_UID` the UID to map the directories and files to. On this example, UID has read/write rights, so it could be your own user in order for you to drop new files (see [rclone documentation](https://rclone.org/commands/rclone_mount/#options) for details).
- `RCLONE_UMASK` set the desired mask when creating new directories and files. Example value `027` is set accordingly with `RCLONE_DIR_PERMS` and `RCLONE_FILE_PERMS` (see [rclone documentation](https://rclone.org/commands/rclone_mount/#options) for details).
- `RCLONE_VFS_CACHE_MAX_AGE` trigger age of files to be purged of cache each time a cleanup is performed (check `RCLONE_VFS_CACHE_POLL_INTERVAL` and see [rclone documentation](https://rclone.org/commands/rclone_mount/#vfs-file-caching) for details).
- `RCLONE_VFS_CACHE_MODE` cache mod, see [rclone documentation](https://rclone.org/commands/rclone_mount/#vfs-file-caching) for details.
- `RCLONE_VFS_CACHE_POLL_INTERVAL` interval to cleanup the cache (see [rclone documentation](https://rclone.org/commands/rclone_mount/#vfs-file-caching) for details)
- `RCLONE_VFS_READ_AHEAD` disk buffer in addition to the kernel buffer during read (see [rclone documentation](https://rclone.org/commands/rclone_mount/#vfs-cache-mode-full) for details)
- `RCLONE_VFS_WRITE_BACK` how many time to wait before starting to upload a file (see [rclone documentation](https://rclone.org/commands/rclone_mount/#vfs-file-caching) for details)
- `RCLONE_WRITE_BACK_CACHE` send writes in batch to rclone, should increase performance especially if a lot of context switching is occuring (see [rclone documentation](https://rclone.org/commands/rclone_mount/#options) for details)

#### RC section

Only used for cache warmup at start. Be carefull to set a different port in `RCLONE_RC_ADDR` for each configuration. See `rclone_vfsrcwarmup` script for details.

### systemd service unit template

This normally does not need configuration but values here will impact all yours configurations. You can tune them before registering your first configuration.

- `Type=notify` as rclone supports systemd notify system, it will enhance systemd services scheduling. Should not be changed.
- `User=rclonemount` do not use root to run rclone
- `Nice=-5` increase system priority for rclone mounts. Reduce time others services using the mount will spend their CPU time in iowait by scheduling rclone mount more often in order for data to be ready for them.
- `TimeoutStartSec=infinity` as the cache warmup can be quite long if there is thousands of thousands of files, we don't want systemd to consider the unit stalling. Note that cache warmup script using rc has also timeout deactivated.
- `ExecReload=/bin/kill -SIGHUP $MAINPID` allow easy runtime dir cache purging, see [Purge directories and files structure cache](#purge-directories-and-files-structure-cache)
- `ExecStopPost=-+/bin/umount -f $DESTINATION` sometimes rclone will fail to unmount cleanly. In order to be able to start the unit again and not leave the mount in an awkward state we force an unmount just in case. Will fail when regular mount has succeeded (this is expected and indicated to systemd with `-`). Because a forced unmount needs root privilege, this command only will run as root as indicated by `+`.

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

Now that your virtual filesystem is perfectly integrated to your system as a systemd service unit, there is a big probability others services are meant to use it. It could be important to bind these services to it to prevent/delay their start if/when the mount has failed/is not ready.

For example for a plex web server:

```bash
~$ cat /etc/systemd/system/plexmediaserver.service.d/rclone.conf
[Unit]
After=rclonemount@example.service
BindsTo=rclonemount@example.service
~$ systemctl daemon-reload
~$ systemctl restart plexmediaserver.service
```

This way:

- `plexmediaserver.service` will only start when `rclonemount@example.service` is fully started/ready
- if you stop `rclonemount@example.service` systemd will FIRST stop `plexmediaserver.service` THEN stop `rclonemount@example.service`
- if you restart `rclonemount@example.service` and `plexmediaserver.service` has been stopped because of the dependency, `plexmediaserver.service` will be automatically (re)started
- if `rclonemount@example.service` fails to start, `plexmediaserver.service` won't start neither
