# Keycloak Docker image with failover support for MariaDB

Keycloak Server Docker image taken from 
[jboss-dockerfile/keycloak](https://github.com/jboss-dockerfiles/keycloak)

This image can be used like the original except for the use of MariaDB.

If `DB_VENDOR` value is set to `mariadb`, the environment variables `DB_ADDR` and `DB_PORT` will be
skipped. Instead `DB_DESCR` value must be specified with hosts and ports of the cluster members,
e. g. `galera_node_1:3306,galera_node_2:3306,galera_node_3:3306`.

The connection uri pattern is `jdbc:mariadb:failover://${env.DB_DESCR:mariadb:3306}/${env.DB_DATABASE:keycloak}${env.JDBC_PARAMS:}`


## Usage


To boot in standalone mode

    docker run bremersee/keycloak-mariadb-failover



## Expose on localhost

To be able to open Keycloak on localhost map port 8080 locally

   docker run -p 8080:8080 bremersee/keycloak-mariadb-failover



## Creating admin account

By default there is no admin user created so you won't be able to login to the admin console. To create an admin account
you need to use environment variables to pass in an initial username and password. This is done by running:

    docker run -e KEYCLOAK_USER=<USERNAME> -e KEYCLOAK_PASSWORD=<PASSWORD> bremersee/keycloak-mariadb-failover

You can also create an account on an already running container by running:

    docker exec <CONTAINER> keycloak/bin/add-user-keycloak.sh -u <USERNAME> -p <PASSWORD>

Then restarting the container:

    docker restart <CONTAINER>



## Importing a realm

To create an admin account and import a previously exported realm run:

    docker run -e KEYCLOAK_USER=<USERNAME> -e KEYCLOAK_PASSWORD=<PASSWORD> \
        -e KEYCLOAK_IMPORT=/tmp/example-realm.json -v /tmp/example-realm.json:/tmp/example-realm.json bremersee/keycloak-mariadb-failover



## Database

This image supports using H2, MySQL, PostgreSQL or MariaDB as the database.

You can specify the DB vendor directly with the `DB_VENDOR` environment variable. Supported values are:

- `h2` for the embedded H2 database,
- `postgres` for the Postgres database,
- `mysql` for the MySql database.
- `mariadb` for the MariaDB database.

If `DB_VENDOR` value is not specified the image will try to detect the DB vendor based on the following logic:

- Is the default host name for the DB set using `getent hosts` (`postgres`, `mysql`, `mariadb`). This works if you are
using a user defined network and the default names as specified below.

If the DB can't be detected it will default to the embedded H2 database.

### Environment variables for all besides mariadb

Generic variable names can be used to configure any Database type, defaults may vary depending on the Database.

- `DB_ADDR`: Specify hostname of the database (optional)
- `DB_PORT`: Specify port of the database (optional, default is DB vendor default port)
- `DB_DATABASE`: Specify name of the database to use (optional, default is `keycloak`).
- `DB_USER`: Specify user to use to authenticate to the database (optional, default is `keycloak`).
- `DB_PASSWORD`: Specify user's password to use to authenticate to the database (optional, default is `password`).

### Environment variables for mariadb

Generic variable names can be used to configure any Database type, defaults may vary depending on the Database.

- `DB_DESCR`: Specify hostnames and ports of the databases in the cluster
- `DB_DATABASE`: Specify name of the database to use (optional, default is `keycloak`).
- `DB_USER`: Specify user to use to authenticate to the database (optional, default is `keycloak`).
- `DB_PASSWORD`: Specify user's password to use to authenticate to the database (optional, default is `password`).


### Specify JDBC parameters

When connecting Keycloak instance to the database, you can specify the JDBC parameters. Details on JDBC parameters can be
found here:

* [PostgreSQL](https://jdbc.postgresql.org/documentation/head/connect.html)
* [MySQL](https://dev.mysql.com/doc/connector-j/5.1/en/connector-j-reference-configuration-properties.html)
* [MariaDB](https://mariadb.com/kb/en/library/about-mariadb-connector-j/#optional-url-parameters)

#### Example

    docker run --name keycloak -e DB_VENDOR=postgres -e JDBC_PARAMS='connectTimeout=30' bremersee/keycloak-mariadb-failover



## Adding a custom theme

To add a custom theme extend the Keycloak image add the theme to the `/opt/jboss/keycloak/themes` directory.

To set the welcome theme, use the following environment value :

* `KEYCLOAK_WELCOME_THEME`: Specify the theme to use for welcome page (must be non empty and must match an existing theme name)


## Adding a custom provider

To add a custom provider extend the Keycloak image and add the provider to the `/opt/jboss/keycloak/standalone/deployments/`
directory.


## Misc

### Specify hostname

To set a fixed hostname for Keycloak use the following environment value. This is highly recommended in production.

* `KEYCLOAK_HOSTNAME`: Specify hostname for Keycloak (optional, default is retrieved from request, recommended in production)

### Specify ports

To set fixed ports for http and https for Keycloak use the following environment values.

* `KEYCLOAK_HTTP_PORT`: Specify the http port for Keycloak (optional, default is retrieved from request)
* `KEYCLOAK_HTTPS_PORT`: Specify the https port for Keycloak (optional, default is retrieved from request)

### Specify log level

There are two environment variables available to control the log level for Keycloak:

* `KEYCLOAK_LOGLEVEL`: Specify log level for Keycloak (optional, default is `INFO`)
* `ROOT_LOGLEVEL`: Specify log level for underlying container (optional, default is `INFO`)

Supported log levels are `ALL`, `DEBUG`, `ERROR`, `FATAL`, `INFO`, `OFF`, `TRACE` and `WARN`.

Log level can also be changed at runtime, for example (assuming docker exec access):

    ./keycloak/bin/jboss-cli.sh --connect --command='/subsystem=logging/root-logger=ROOT:change-root-log-level(level=DEBUG)'
    ./keycloak/bin/jboss-cli.sh --connect --command='/subsystem=logging/logger=org.keycloak:write-attribute(name=level,value=DEBUG)'

### Enabling proxy address forwarding

When running Keycloak behind a proxy, you will need to enable proxy address forwarding.

    docker run -e PROXY_ADDRESS_FORWARDING=true bremersee/keycloak-mariadb-failover



### Setting up TLS(SSL)

Keycloak image allows you to specify both a private key and a certificate for serving HTTPS. In that case you need to provide two files:

* tls.crt - a certificate
* tls.key - a private key

Those files need to be mounted in `/etc/x509/https` directory. The image will automatically convert them into a Java keystore and reconfigure Wildfly to use it.

It is also possible to provide an additional CA bundle and setup Mutual TLS this way. In that case, you need to mount an additional volume to the image
containing a `crt` file and point `X509_CA_BUNDLE` environmental variable to that file.

NOTE: See `openshift-examples` directory for an out of the box setup for OpenShift.



## Other details

This image extends the [`jboss/base-jdk`](https://github.com/JBoss-Dockerfiles/base-jdk) image which adds the OpenJDK
distribution on top of the [`jboss/base`](https://github.com/JBoss-Dockerfiles/base) image. Please refer to the README.md
for selected images for more info.
