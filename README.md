moodle-dev-compose
------------------

Compose file for Docker based local Moodle dev environment.

## Requirements

You will need:

* [Docker](https://docs.docker.com/get-docker/). If you don't need GUI, installing only [Docker engine](https://docs.docker.com/engine/install/) is sufficient, but you need [Docker Compose](https://docs.docker.com/compose/install/) installed separately.
* Clone of Moodle repo that you want to work on.
* Make sure you don't run local webserver that is using port 80 (if you need,
  change it to different port or use different port in compose file).
* Make sure you don't run local mysql/mariadb server that is using port 3306 (if
  you need, change it to different port or use different port in compose file)

### Moodle config.php

Your Moodle config.php should contain (as minimum):

```
<?php  // Moodle configuration file

unset($CFG);
global $CFG;
$CFG = new stdClass();

$CFG->dbtype    = 'mariadb';
$CFG->dblibrary = 'native';
$CFG->dbhost    = 'mariadb';
$CFG->dbname    = 'moodle-main';
$CFG->dbuser    = 'moodle';
$CFG->dbpass    = 'moodle';
$CFG->prefix    = 'mdl_';
$CFG->dboptions = array (
  'dbpersist' => 0,
  'dbport' => '',
  'dbsocket' => '',
  'dbcollation' => 'utf8mb4_general_ci',
);

$CFG->wwwroot   = 'http://moodle.local';
$CFG->dataroot  = '/var/www/moodledata';
$CFG->admin     = 'admin';

$CFG->directorypermissions = 0777;

require_once(__DIR__ . '/lib/setup.php');
```

You may add subdirectory to `dataroot` if you wish, e.g. for using different
dir for different databases when you switch between Moodle versions.

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

Define env variable with location to your Moodle code:

```bash
export MOODLE_DOCKER_WWWROOT=/home/ruslan/git/moodle
```

This env var is used for Moodle codebase mount inside the web container.

Define env variable with PHP version to be used:

```bash
export MOODLE_DOCKER_PHP_VERSION=8.1
```

This is used to select `moodlehq/moodle-php-apache` image version to use.

To start the environment, use `docker compose` command. You need to execute it
from this repo directory:

```bash
> docker compose up
```

You can keep the terminal open to see the log, alernatively you may run it in
deaemon mode:

```bash
> docker compose up -d
```

You can see the status of containers using:

```bash
> docker compose ps
```

Above command amond other things is showing ports mapping from local interface to containers.

Notice that names of containers specified in compose file are resolved to
their IPs on any container in the set, e.g. you may use `mariadb` in your
moodle `config.php` as DB hostname.

You may stop the containers using `docker compose stop` or destroy them using
`docker compose down`. It is safe to desroy as you are using volumes, so next
time you start them, all data will be in place within newly created
containers.

### Database

In this setup MariaDB docker image is used. The data is located on
the volume, so recreating container will not cause data loss.

On the first run, you will probably need to create database that you will use.
You can do that using phpmyadmin included in compose file, and accessible at
`http://moodle.local:8081`

Accessing server using DB client is also possible on `localhost:3306`, as container
propagates this port to the host machine. Alternatively, you can access it by
executing shell on running DB container and using `mysql` client:

```bash
> docker exec -u mysql -it moodle-dev-compose-mariadb-1 bash
mysql@35e794676852:/$ mysql -u root -p
Enter password:
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 5
Server version: 10.6.16-MariaDB-1:10.6.16+maria~ubu2004 mariadb.org binary distribution

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

### Accessing Moodle

Now when you have database created and config.php configured, you may enter
`http://moodle.local` in your browser on host machine. This will point to your
localhost according to `/etc/hosts` record we created, which will then be
handled by `nginx-proxy` container (notice the port mapping mentioned above).
`nginx-proxy` will recognise hostname `moodle.local` and redirect request to
`moodle-dev-compose-moodle-1` container that will show you Moodle setup screen
in the browser.

If you need `php` command line (e.g. for setting up Moodle or cron run), you
may access it from web container:

```bash
> docker exec -it -u www-data moodle-dev-compose-moodle-1 bash
www-data@0c46f3f2037a:~/html$
www-data@1d18bc646bf4:~/html$ php admin/cli/upgrade.php
No upgrade needed for the installed version 4.4dev (Build: 20240215) (2024021500). Thanks for coming anyway!
```

### Receiving mail

We run Mailpit container by default that allows to receive email and provides
interface to view it.

In Moodle configuration file add:
```
$CFG->smtphosts = 'mail:1025'
```

Web interface to view emails is available at `http://moodle.local:8025`


## Adding PHP extensions
### Profiling

You can add Xhprof extension and use Moodle built-in profiling tool.

```bash
> docker exec -it moodle-dev-compose-moodle-1 bash
root@62ba9a041e0f:/var/www/html#  pecl install xhprof
WARNING: channel "pecl.php.net" has updated its protocols, use "pecl channel-update pecl.php.net" to update
downloading xhprof-2.2.0.tgz ...
...
root@62ba9a041e0f:/var/www/html# docker-php-ext-enable xhprof
root@62ba9a041e0f:/var/www/html# php -i | grep xhprof
/usr/local/etc/php/conf.d/docker-php-ext-xhprof.ini,
xhprof
xhprof support => enabled
xhprof.collect_additional_info => 0 => 0
xhprof.output_dir => no value => no value
xhprof.sampling_depth => 0x7fffffff => 0x7fffffff
xhprof.sampling_interval => 100000 => 100000
```
Xhprof extension has been installed and enabled, restart the web container.

```bash
> docker restart moodle-dev-compose-moodle-1
```

A this point you can navigate to Site administration > Development >
Profiling, enable it and trigger for the content you need to profile.

Notice, if you destroy container, you will need to repeat above steps on the
new one.

## More containers

Suppose you need another container with a different php version, but for the
same Moodle instance. You can easily add another container either directly to
existing compose file, or by using a seaparate file:

Create a new file called `compose.local.yaml` containing:

```
services:
    moodle83:
        image: moodlehq/moodle-php-apache:8.3
        volumes:
          - moodledata:/var/www/moodledata
          - $MOODLE_DOCKER_WWWROOT:/var/www/html
        depends_on:
          - mariadb
        environment:
          - VIRTUAL_HOST=moodle83.local
        networks:
          - devbox
```

Now, stop the existing containers and modify `compose.yaml`, add [`include`](https://docs.docker.com/compose/multiple-compose-files/include/) section in top of file:

```
include:
  - compose.local.yaml
```

This will bring a new container into play on the next `docker compose up` run.

Note: if you prefer not to use `include` section in `compose.yaml`, alternative way to use many config
files is:
```bash
> docker compose -f compose.yaml -f compose.local.yaml up
```

Add the new host to your `/etc/hosts` and you can start using it on `http://moodle83.local`.

```
127.0.0.1 moodle.local moodle83.local
```

## Using SSL

If you need access by `https`, create self-signed certificates named
`localhost`:

```
$ mkdir certs
$ openssl req -x509 -nodes -newkey rsa:2048 -keyout certs/localhost.key -out certs/localhost.crt
```

Then use them as part of nginx-proxy configuration:

```
diff --git a/compose.yaml b/compose.yaml
index 0bc7fb0..934cc58 100644
--- a/compose.yaml
+++ b/compose.yaml
@@ -5,9 +5,11 @@ services:
         container_name: nginx-proxy
         ports:
           - "80:80"
+          - "443:443"
         volumes:
           - /var/run/docker.sock:/tmp/docker.sock:ro
           - ./nginx_proxy.conf:/etc/nginx/conf.d/nginx_proxy.conf:ro
+          - ./certs:/etc/nginx/certs
         networks:
           - devbox
     moodle:
```

Also in Moodle config make sure `wwwroot` reflects correct protocol:
```
$CFG->wwwroot   = 'https://moodle.local';
```

## Other services

Using docker you can add any service you need. Examples below assume you are
adding configuration to `compose.local.yaml`.

### Memcached

Adding memcached will make your instance running faster.

Your `compose.local.yaml` should contain:

```
services:
    memcached0:
        image: memcached:1.4.33
        networks:
          - devbox
```

In Moodle, navigate to cache configuration and create instance using
`memcached0` as hostname and `11211` as port.

### SAML2

In order to deploy SAML2 IdP, we will be using [kristophjunge/docker-test-saml-idp](https://github.com/kristophjunge/docker-test-saml-idp) docker image.

You need [auth_saml2](https://github.com/catalyst/moodle-auth_saml2) plugin to be installed in Moodle, it will act as SAML2 service provider (SP).

Your `compose.local.yaml` should contain:

```
services:
    samlidp:
        image: kristophjunge/test-saml-idp
        environment:
          VIRTUAL_HOST: samlidp.local
          SIMPLESAMLPHP_SP_ENTITY_ID: http://moodle.local/auth/saml2/sp/metadata.php
          SIMPLESAMLPHP_SP_ASSERTION_CONSUMER_SERVICE: http://moodle.local/auth/saml2/sp/saml2-acs.php/moodle.local
          SIMPLESAMLPHP_SP_SINGLE_LOGOUT_SERVICE: http://moodle.local/auth/saml2/sp/saml2-logout.php/moodle.local
          SIMPLESAMLPHP_ADMIN_PASSWORD: admin1
          SIMPLESAMLPHP_SECRET_SALT: salt
        ports:
        - "8080:8080"
        - "8443:8443"
        volumes:
        - ./saml2/authsources.php:/var/www/simplesamlphp/config/authsources.php
        networks:
          devbox:
            aliases:
              - samlidp.local
```

Those `SIMPLESAMLPHP_SP_*` values can be identified from SP metadata output (`http://moodle.local/auth/saml2/sp/metadata.php`).

You also need `saml2/authsources.php` containing user accounts, which is mounted
in container, use example below as starting point:

```
<?php

$config = [

    'admin' => [
        'core:AdminPassword',
    ],

    'example-userpass' => [
        'exampleauth:UserPass',
        'samlu1:samlu1pass' => [
            'uid' => ['samlu1'],
            'eduPersonAffiliation' => ['group1'],
            'email' => 'samluser1@example.com',
            'firstName' => 'Saml',
            'lastName' => 'User 1',
            'customOrg' => 'Company 1',
        ],
        'samlu2:samlu2pass' => [
            'uid' => ['samlu2'],
            'eduPersonAffiliation' => ['group2'],
            'email' => 'samluser2@example.com',
            'firstName' => 'Saml',
            'lastName' => 'User 2',
            'customOrg' => 'Company 2',
        ],
    ],
];
```

When you start containers, navigate to
`http://samlidp.local:8080/simplesaml/module.php/core/frontpage_federation.php`,
you will find metadata link you can use in `auth_saml2` `idpmetadata` setting
to complete setup (or you can use metadata XML that you can retrieve on the
same page). If you get `Invalid metadata at
http://samlidp.local:8080/simplesaml/saml2/idp/metadata.php` error, most
likely CURL security settings do not permit URL access, check
`curlsecurityblockedhosts` and `curlsecurityallowedport` config settings in
Moodle.

Once configured (you need to add field mapping as well, for example snippet
above you may need to map `firstName`, `lastName` and `email`), you should be
able to login to Moodle using one of accounts defined in `authsources.php`.
