# docker-rclone

Docker image for [rclone](https://rclone.org/) mount, with

- Ubuntu 20.04
- pooling filesystem (a choice of mergerfs or unionfs)
- some useful scripts

## Usage

```yaml
version: '3'

services:
  rclone:
    container_name: rclone
    image: slink42/rclone
    restart: always
    network_mode: "bridge"
    volumes:
      - ${DOCKER_ROOT}/rclone/config:/config
      - ${DOCKER_ROOT}/rclone/log:/log
      - ${DOCKER_ROOT}/rclone/cache:/cache
      - /your/mounting/points/parent/directory:/mnt:shared
    devices:
      - /dev/fuse
    cap_add:
      - SYS_ADMIN
    security_opt:
      - apparmor:unconfined
    environment:
      - PUID=${PUID}
      - PGID=${PGID}
      - TZ=Australia/Melbourne
      - RCLONE_REMOTE_PATH=remote_name:path/to/mount
      - MERGED_DEST=/mnt/data
      - PLEXDRIVE_REMOTE=plexdrive_remote_name
```

equivalently,

```bash
docker run -d \
    --name=rclone \
    --cap-add SYS_ADMIN \
    --device /dev/fuse \
    --security-opt apparmor=unconfined \
    -v ${DOCKER_ROOT}/rclone/config:/config \
    -v ${DOCKER_ROOT}/rclone/log:/log \
    -v ${DOCKER_ROOT}/rclone/cache:/cache \
    -v /your/mounting/points/parent/directory:/mnt:shared \
    -e PUID=${PUID} \
    -e PGID=${PGID} \
    -e TZ=Australia/Melbourne \
    -e RCLONE_REMOTE_PATH=remote_name:path/to/mount \
    -e MERGED_DEST=/mnt/data \
    -e PLEXDRIVE_REMOTE=plexdrive_remote_name \
    slink42/rclone
```

First, you need to prepare an rclone configuration file in ```/config/rclone.conf```. It can be done manually (copy yourself) or by running a built-in script below

```bash
docker-compose exec <service_name> rclone_setup
```

Then, up and run your container with a proper environment variable ```RCLONE_REMOTE_PATH``` which specifies an rclone remote path you want to mount. In the initialization process of every container start, it will check 1) existance of ```rclone.conf``` and 2) validation of ```RCLONE_REMOTE_PATH``` whether it actually exists in ```rclone.conf```. If there is any problem, please check container log by

```bash
docker logs <container name or sha1, e.g. rclone>
```

### rclone mount

Here is the internal command for rclone mount.

```bash
rclone mount ${RCLONE_REMOTE_PATH} ${rclone_mountpoint} \
    --uid=${PUID:-911} \
    --gid=${PGID:-911} \
    --cache-dir=/cache \
    --use-mmap \
    --allow-other \
    --umask=002 \
    --rc \
    --rc-no-auth \
    --rc-addr=:5574 \
    ${RCLONE_MOUNT_USER_OPTS}
```

Please note that variables only with capital letters are configurable by environment variables. Also, be aware that rc is enabled by default.

| ENV  | Description  | Default  |
|---|---|---|
| ```PUID``` / ```PGID```  | uid and gid for running an app  | ```911``` / ```911```  |
| ```TZ```  | timezone, required for correct timestamp in log  |   |
| ```RCLONE_REMOTE_PATH```  | this should be in ```rclone.conf```  |   |
| ```RCLONE_CONFIG```  | path to ```rclone.conf```  |  ```/config/rclone.conf``` |
| ```RCLONE_LOG_LEVEL```  | log level for rclone runtime  | ```NOTICE```  |
| ```RCLONE_LOG_FILE```  | to redirect logging to file  |   |
| ```RCLONE_MOUNT_USER_OPTS```  | additioanl arguments will be appended to the basic options in the above command  |   |

## [plexdrive](https://github.com/plexdrive/plexdrive) mount (optional)

In additon to setting up an rclone mount, a plexdrive mount can be configured. This only occours when both plexdrive config files are found at /config/config.json and /config/token.json or the envrionment variable PLEXDRIVE_REMOTE is set. When PLEXDRIVE_REMOTE is supplied, the rclone remote matching the name of the variable value is used to generate the plexdrive config files. Internally, it will execute the following command to mount plexdrive:

```bash
plexdrive \
  mount ${pd_mountpoint} $(echo $pd_basic_opts) ${pd_user_opts[@]}
```

using the defaults this would equate to:

```bash
plexdrive \
  mount /mnt/plexdrive \
    --config /config/ \
    --uid=${PUID:-911} \
    --gid=${PGID:-911} \
    --umask=0100775 \
    -o allow_other
```

## [mergerfs](https://github.com/trapexit/mergerfs) or unionfs (optional)

Along with the rclone folder, you can specify one local directory to be mergerfs with by ```POOLING_FS=mergerfs```. Internally, it will execute a following command

```bash
mergerfs \
    -o uid=${PUID:-911},gid=${PGID:-911},umask=022,allow_other \
    -o ${MFS_USER_OPTS} ${MFS_BRANCHES} ${MERGED_DEST}
```

where a default value of ```MFS_USER_OPTS``` is

```bash
MFS_USER_OPTS="rw,use_ino,func.getattr=newest,category.action=all,category.create=ff,cache.files=auto-full,dropcacheonclose=true"
```

and a default value of ```MFS_BRANCHES``` is
```bash
MFS_BRANCHES="/mnt/local=RW:/mnt/cloud=NC"
```


If you want unionfs instead of mergerfs, set ```POOLING_FS=unionfs```, which will apply

```bash
unionfs \
    -o uid=${PUID:-911},gid=${PGID:-911},umask=022,allow_other \
    -o ${UFS_USER_OPTS} ${UFS_BRANCHES} ${MERGED_DEST}
```

where a default value of ```UFS_USER_OPTS``` is

```bash
UFS_USER_OPTS="cow,direct_io,nonempty,auto_cache,sync_read"
```

and a default value of ```UFS_BRANCHES``` is

```bash
UFS_BRANCHES="/mnt/local=RW:/mnt/cloud=RO"
```

### Built-in scripts

Two scripts performing basic rclone operations such as copy and move between ```/mnt/local``` and ```/mnt/cloud``` are prepared for your conveinence. Since they are from local to cloud directories, it is meaningful only when you mount an additional ```/mnt/local``` directory.

#### copy_local

You can make a copy of files in ```/mnt/local``` to ```/mnt/cloud``` by

```bash
docker exec -it <container name or sha1, e.g. rclone> copy_local
```

If you want to exclude a certain folder from copy, just put an empty ```.nocopy``` file on the folder root. Then, the script will ignore the sub-tree from the operation.

#### move_local

In contrast to ```copy_local```, ```move_local``` consists of three consecutive sub-operations. First, it will move old files. If ```MOVE_LOCAL_AFTER_DAYS``` is set, files older than that days will be moved. Then, it will move files exceed size of ```MOVE_LOCAL_EXCEEDS_GB``` by the amount of ```MOVE_LOCAL_FREEUP_GB```. Finally, it will move the rest of files in ```/mnt/local``` only if ```MOVE_LOCAL_ALL=true```. The command and the way to exclude subfolders are almost the same as for ```copy_local```.

#### cron - disabled by default

After making sure that a single execution of scripts is okay, you can add cron jobs of these operations by setting environment variables.

| ENV  | Description  | Default  |
|---|---|---|
| ```COPY_LOCAL_SCHEDULE```  | cron schedule for copy_local  |  |
| ```MOVE_LOCAL_SCHEDULE```  | cron schedule for move_local  |  |

## Credit

- [docker-rclone](https://github.com/wiserain/docker-rclone)
- [docker-plexdrive](https://github.com/wiserain/docker-plexdrive)
- [cloud-media-scripts](https://github.com/madslundt/docker-cloud-media-scripts)
