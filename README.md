moodle-dev-compose
------------------

Compose file for Docker based local Moodle dev environment.

## Requirements

You will need:

* Docker and Docker-compose installed.
* Clone of Moodle repo that you want to work on.
* Make sure you don't run local webserver that is using port 80 (if you need,
  change it to different port or use different port in compose file).
* Make sure you don't run local postgres server that is using port 5432 (if
  you need, change it to different port or use different port in compose file)

### Volumes

For data persitense (so that you won't lose your Moodle data or databases) you
need to create volumes. Those volumes are then mounted to containers according
to your compose file configuration. By default, you need at least two volumes:

```bash
> docker volume create pgdata11
pgdata11
> docker volume create moodledata
moodledata
```


### Moodle config.php

Your Moodle config.php should contain at least:

* Database host and credentials pointing to exising database container
  (you can use container name as hostname).
* `$CFG->dataroot  = '/var/www/moodledata';`
* `$CFG->wwwroot   = 'http://moodle.local';`

### Hostname

Compose file is defining proxy container that handles hostname mapping to web
container, this makes things simpler as you may add more web containers (e.g.
for different repos, php versions or DBs) using a different hostname for each.

By default, the only web container is called `moodle.local`.

To make possible accessing container from your host machine by using
hostname, you need a record in your `/etc/hosts` file pointing to localhost.

```
127.0.0.1 moodle.local
```

## Running environment


First, define env variable with location to your Moodle code:

```bash
export MOODLE_DOCKER_WWWROOT=/home/ruslan/git/moodle
```

This env var is used for Moodle codebase mount inside the web container.

To start the environment, use docker-compose command. You need to execute it
from this repo directory:

```bash
> docker-compose up
```

You can keep the terminal open to see the log, alernatively you may run it in
deaemon mode:

```bash
> docker-compose up -d
```

You can see the status of containers using:

```bash
> docker-compose ps
              Name                             Command               State           Ports
---------------------------------------------------------------------------------------------------
moodle-dev-compose_moodle_1         docker-php-entrypoint apac ...   Up      80/tcp
moodle-dev-compose_postgres_1       /docker-entrypoint.sh postgres   Up      0.0.0.0:5432->5432/tcp
nginx-proxy                         /app/docker-entrypoint.sh  ...   Up      0.0.0.0:80->80/tcp
```

Above command amond other things is showing ports mapping from local interface to containers.

Notice that names of containers are resolved to their IPs on any container in
the set, e.g. you may use `moodle-dev-compose_postgres_1` in your moodle
`config.php` as DB hostname.

You may stop the containers using `docker-compose stop` or destroy them using
`docker-compose down`. It is safe to desroy as you are using volumes, so next
time you start them, all data will be in place within newly created
containers.

### Database

In this setup offical Postgres 11 docker image is used. The data is localted on
the volume, so recreating container will not cause data loss.

On the first run, you will probably need to create database that you will use.

Accessing server using DB client is possible on `localhost:5432`, as container
propagates this port to the host machine. Alternatively, you can access it by
executing shell on running DB container and using `psql` client:

```bash
> docker exec -it -u postgres moodle-dev-compose_postgres_1 bash
postgres@6f001003d4c4:/$ createuser -DPRS moodlepg
Enter password for new role:
Enter it again:
postgres@6f001003d4c4:/$ createdb -O moodlepg -E UTF-8 moodle-master-db
postgres@6f001003d4c4:/$ psql
psql (11.5)
Type "help" for help.

postgres=# \l
                                         List of databases
           Name            |  Owner   | Encoding |  Collate   |   Ctype    |   Access privileges
---------------------------+----------+----------+------------+------------+-----------------------
 moodle-master-db          | moodlepg | UTF8     | en_US.utf8 | en_US.utf8 |
 postgres                  | postgres | UTF8     | en_US.utf8 | en_US.utf8 |
 template0                 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
                           |          |          |            |            | postgres=CTc/postgres
 template1                 | postgres | UTF8     | en_US.utf8 | en_US.utf8 | =c/postgres          +
                           |          |          |            |            | postgres=CTc/postgres
(4 rows)

postgres=#

```

#### Upgrading DB

I suggest to use https://github.com/tianon/docker-postgres-upgrade image to
perform major release upgrade. For 9.4 to 11 upgrade I createed a new volume
called `pgdata11`, then stop database gracefully by logging in into container
and executing:

```
docker exec -it -u postgres moodle-dev-compose_postgres_9.4_1 bash
postgres@ea13edb262fb:/$ pg_ctl stop
```

Now, when service is gracefully stopped, execute:
```
docker run --rm -v pgdata:/var/lib/postgresql/9.4/data  -v pgdata11:/var/lib/postgresql/11/data tianon/postgres-upgrade:9.4-to-11
```

This will perform upgrade. Finally, shut down your dev suit, make necessary changes in
compose files and start again (reflect volume changes as in [1f266d](https://github.com/kabalin/moodle-dev-compose/commit/1f266df29c4d6b0c33eddbc0faad4af47db9fa1f)).

### Accessing Moodle

Now when you have database created and config.php configured, you may enter
`http://moodle.local` in your browser on host machine. This will point to your
localhost according to `/etc/hosts` record we created, which will then be
handled by `nginx-proxy` container (notice the port mapping mentioned above).
`nginx-proxy` will recognise hostname `moodle.local` and redirect request to
`moodle-dev-compose_moodle_1` container that will show you Moodle setup screen
in the browser.

If you need `php` command line (e.g. for setting up Moodle or cron run), you
may access it from web container:

```bash
> docker exec -it -u www-data moodle-dev-compose_moodle_1 bash
www-data@0c46f3f2037a:~/html$
www-data@0c46f3f2037a:~/html$ php admin/cli/upgrade.php
No upgrade needed for the installed version 3.7.2 (Build: 20190909) (2019052002). Thanks for coming anyway!
```

### More containers

Suppose you need another container with a different php version, but for the
same Moodle instance. You can easily add another container either directly to
existing compose file, or by using a seaparate file:

Create a new file called docker-compose.local.yml containing:

```
version: '3'
services:
    moodle73:
        image: moodlehq/moodle-php-apache:7.3
        volumes:
          - moodledata:/var/www/moodledata
          - $MOODLE_DOCKER_WWWROOT:/var/www/html
        depends_on:
          - postgres
        environment:
          - VIRTUAL_HOST=moodle73.local
        networks:
          - devbox
```

Now, stop the existing containers and start them using slightly different compose command:

```bash
> docker-compose -f docker-compose.yml -f docker-compose.local.yml up
```

This will bring a new container into play:

```bash
> docker-compose -f docker-compose.yml -f docker-compose.local.yml ps
              Name                             Command               State           Ports
---------------------------------------------------------------------------------------------------
moodle-dev-compose_moodle_1         docker-php-entrypoint apac ...   Up      80/tcp
moodle-dev-compose_moodle73_1       docker-php-entrypoint apac ...   Up      80/tcp
moodle-dev-compose_postgres_1       /docker-entrypoint.sh postgres   Up      0.0.0.0:5432->5432/tcp
nginx-proxy                         /app/docker-entrypoint.sh  ...   Up      0.0.0.0:80->80/tcp
```

Add the new host to your `/etc/hosts` and you can start using it on `http://moodle73.local`.

```
127.0.0.1 moodle.local moodle73.local
```

### Even more contnainers

With docker you can do anything you need. Try adding memcached to make your
instance faster:

Your docker-compose.local.yml may look like:

```
version: '3'
services:
    moodle73:
        image: moodlehq/moodle-php-apache:7.3
        volumes:
          - moodledata:/var/www/moodledata
          - $MOODLE_DOCKER_WWWROOT:/var/www/html
        depends_on:
          - postgres
        environment:
          - VIRTUAL_HOST=moodle73.local
        networks:
          - devbox
    memcached0:
        image: memcached:1.4.33
        networks:
          - devbox
```

This will add another container:

```bash
> docker-compose -f docker-compose.yml -f docker-compose.local.yml ps
              Name                             Command               State           Ports
---------------------------------------------------------------------------------------------------
moodle-dev-compose_memcached0_1     docker-entrypoint.sh memcached   Up      11211/tcp
moodle-dev-compose_moodle_1         docker-php-entrypoint apac ...   Up      80/tcp
moodle-dev-compose_moodle73_1       docker-php-entrypoint apac ...   Up      80/tcp
moodle-dev-compose_postgres_1       /docker-entrypoint.sh postgres   Up      0.0.0.0:5432->5432/tcp
nginx-proxy                         /app/docker-entrypoint.sh  ...   Up      0.0.0.0:80->80/tcp
```

In Moodle, navigate to cache configuration and create instance using
`moodle-dev-compose_memcached0_1` as hostname and `11211` as port.

Enjoy!


