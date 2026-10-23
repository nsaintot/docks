## Description
A script to simplify the way to interact with dockers container

### Features
- restart a dead container in no time
- each container are aware of any containers (dns hosts)
- backup containers volume and configuration locally or upload them on s3
- light script to install anywhere

# Usage

```
$~> docks
usage: services start|stop|restart|status|build|update|list|log|self-update <services> <opts>
services: manage the containers

Options:

start|stop|restart <services> : Manipulations around services
	[--rm|-r]	: Erase the previously running container if so / rebuild the image
	[--no-cache|-n]	: Build the image without using the layer cache
	[--verbose|-v]	: Display the verbose output (behind the scenes)
	[--force-pull|-f]	: Always pull the image tag before commands
	[--dependency|-d]	: Check/Start dependents containers before
build <services>	: Build the service with docker of choosen service
push <services>	: Push the service of choosen service
reset <services>	: Stop the container, rebuild and start it
status <services>	: Sh ow the running services
enter  <services>	: Enter interactivly inside container
update 			: Update the containers resolve ip's
list [-c]		: List the availables services
log [--color|-c]	: Logging containers (need ccze)
secret <service> [list|set|rm] <NAME>: Manage the service secrets (set reads stdin, needs root)
self-update		: Check/Install latest version of Docks

offered apps: *registered containers listed there*
```

# Installation

###1 - Get

```bash
wget https://raw.githubusercontent.com/Amnacog/docks/master/docks.sh > docks.sh;chmod +x docks.sh
```

###2 - Script Configuration

at the first launch, it will create a `.docks-config` under the same directory.
Edit it as you want...

###3 - Containers configuration

> Containers dir tree:

```
containers folder  
│
└───┬─ prefix.apache
    │   │INFO
    │   │start.sh
    │
    ├─ prefix.nginx
    │   │Dockerfile
    │   │INFO
    │   │start.sh
    │   ├─ app
    │      │...
    │
    └───...
```

- `start.sh`:
	contains the docker run command with some injected environment variables    
	eg:

	```bash
	$~> cat start.sh
	docker run \
		-d \
		-h container.sample.com \
		--restart=always \
		-v $PWD/volume/data:/data \
		-v $LOGDIR:/logs \
		--name $NAME \
		$IMAGE
	```

 - `INFO`:
    contains the image configuration (build version run backup)
    eg:
    
    ```bash
	NAME="sample_container" # container to run
	IMAGE="nginx"           # image name to build
	VERSION="0.1"           # image tag to build
	DEPENDS="some_child"
	
	#backups
	VOLUME_DIR="app"        # folder to backup 
	BACKUP_DIRS="data"      #filter inside 'app' folder. unset if not wanted
    ```
 
 - `Dockerfile`:
    contains the docker script to build

### Secrets

Secrets are kept out of `start.sh` and out of the container configuration, so `docker inspect` and the Docker API do not show them.

| Item | Value |
|---|---|
| Enable | `SECRETS=true` in `INFO` (files owned by root), or `SECRETS="<uid>:<gid>"` when the container user is not root |
| Storage | `<datadir>/<prefix><service>/secrets/<NAME>`, directory `0500`, files `0400` |
| Mounts | `$SECRETS_OPTS`: the secrets directory at `/run/secrets` and the wrapper at `/run/docks/secrets-env`, both read-only |
| Wrapper | `/run/docks/secrets-env <command>` exports each file as `NAME=<content>`, then runs the command |

Commands (root only; `set` reads the value from stdin, or prompts without echo on a terminal):

```bash
docks secret <service>                 # list secret names
docks secret <service> set <NAME>      # create or replace
docks secret <service> rm <NAME>
openssl rand -base64 32 | docks secret mysql set MYSQL_ROOT_PASSWORD
```

`start.sh`, for an image that reads `*_FILE` variables:

```bash
docker run \
	-d \
	$SECRETS_OPTS \
	-e MYSQL_ROOT_PASSWORD_FILE=/run/secrets/MYSQL_ROOT_PASSWORD \
	--name $NAME \
	$IMAGE
```

`start.sh`, for any other image: put the wrapper in front of the image's original entrypoint and command (`docker image inspect -f '{{.Config.Entrypoint}} {{.Config.Cmd}}' <image>`):

```bash
docker run \
	-d \
	$SECRETS_OPTS \
	--entrypoint /run/docks/secrets-env \
	--name $NAME \
	$IMAGE /entrypoint.sh serve
```

Notes:
- The variables still exist in the environment of the container processes.
- The wrapper uses shell built-ins only. For an image without a shell, add a static one in its `Dockerfile`: `COPY --from=busybox:1.37.0-musl /bin/busybox /bin/sh`.
- The uid in `SECRETS` is also a host uid: a host user with the same uid can read the files.
- Restart the service after `docks secret set` (`docks restart <service> -r`).




###4 - Run !

> Example:

```
	$~> docks build nginx
	Building nginx image
	Configuring nginx image finished
	
	$~> docks start nginx
	Starting nginx container (new) finished
	Updating container's ip finished
	
	#wait, nginx is dead ??
	$~> docks status nginx
	[KO] nginx is not running.
	
	#ooh right.. Easy !
	$~> docks restart nginx
	Removing nginx container (dead) finished
	Starting nginx container (new) finished
	Updating container's ip finished
		
	$~> docks status nginx
	[OK] nginx is running on 172.17.0.1


	$~> docks enter nginx
	root@container.sample.com:/# ps x
	PID TTY      STAT    TIME COMMAND
     1 ?         S      0:00 nginx: master process nginx
     2 ?         Ss     0:00 bash
     3 ?         R+     0:00 ps x 
```


More to come..
