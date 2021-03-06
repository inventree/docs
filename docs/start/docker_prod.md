---
title: Docker Production Server
---

## Docker Production Server

Using the [InvenTree docker image](./docker.md) streamlines the setup process for an InvenTree production server.

## Docker Compose

It is strongly recommended that you use a [docker-compose](https://docs.docker.com/compose/) script to manage your InvenTree docker image.

An example docker compose script is provided below, which provides a robust "out of the box" setup for running InvenTree.

Firstly, here is the complete `docker-compose.yml` file which can be used "as is" or as a starting point for a custom setup:

``` yaml
{% include 'docker-compose.yml' %}
```

### Containers

The following containers are created:

| Container | Description |
| --- | --- |
| inventree-db | PostgreSQL database |
| inventree-server | Gunicorn web server |
| invenrtee-worker | django-q background worker |
| inventree-proxy | nginx proxy |

#### PostgreSQL Database

A postgresql database container which creates a postgres user:password combination (which can be changed). This uses the official [PostgreSQL image](https://hub.docker.com/_/postgres).

*__Note__: An empty database must be manually created as part of the setup (below)*.

#### Web Server

Runs an InvenTree web server instance, powered by a Gunicorn web server. In the default configuration, the web server listens on port `8000`.

#### Background Worker

Runs the InvenTree background worker process. This spins up a second instance of the *inventree* container, with a different entrypoint command.

#### Nginx

Nginx working as a reverse proxy, separating requests for static and media files, and directing everything else to Gunicorn.

This container uses the official [nginx image](https://hub.docker.com/_/nginx).

An nginx configuration file must be provided to the image. Use the example configuration below as a starting point:

```
{% include 'nginx.conf' %}
```

*__Note__: You must save this conf file in the same directory as your docker-compose.yml file*

!!! info "Proxy Pass"
    If you change the name (or port) of the InvenTree web server container, you will need to also adjust the `proxy_pass` setting in the nginx.conf file!

### Data Volume

InvenTree stores data which is meant to be persistent (e.g. uploaded media files, database data, etc) in a volume which is mapped to a local system directory.

!!! info "Data Directory"
    Make sure you change the path to the local directory where you want persistent data to be stored.

The InvenTree docker server will manage the following directories and files within the 'data' volume:

| Path | Description |
| --- | --- |
| ./config.yaml | InvenTree server configuration file |
| ./secret_key.txt | Secret key file |
| ./media | Directory for storing uploaded media files |
| ./static | Directory for storing static files |

## Production Setup

With the docker-compose recipe above, follow the instructions below to initialize a complete production server for InvenTree.

### Required Files

The following files are required on your local machine (use the examples above, or edit as required):

- [docker-compose.yml](https://github.com/inventree/InvenTree/blob/master/docker/docker-compose.yml)
- [nginx.conf](https://github.com/inventree/InvenTree/blob/master/docker/nginx.conf)

!!! info "Command Directory"
    It is assumed that all commands will be run from the directory where `docker-compose.yml` is located. 

### Configure Compose File

Save and edit the `docker-compose.yml` file as required.

The only **required** change is to ensure that the `/path/to/data` entry (at the end of the file) points to the correct directory on your local file system, where you want InvenTree data to be stored.

!!! info "Database Credentials"
    You may also wish to change the default postgresql username and password!

### Launch Database Container

Before we can create the database, we need to launch the database server container:

```
docker-compose up -d inventree-db
```

This starts the database container.

### Create Database

As this is the first time we are interacting with the docker containers, the InvenTree database has not yet been created.

Run the following command to open a shell session for the database:

```
docker-compose run inventree-server pgcli -h inventree-db -p 5432 -u pguser
```

!!! info "User"
    If you have changed the `POSTGRES_USER` variable in the compose file, replace `pguser` with the different user.

You will be prompted to enter the database user password (default="pgpassword", unless altered in the compose file).

Once logged in, run the following command in the database shell:

```
create database inventree;
```

Then exit the shell with <kbd>Ctrl</kbd>+<kbd>d</kbd>

### Perform Database Migrations

The database has now been created, but it is empty! We need to perform the initial database migrations:

```
docker-compose run inventree-server invoke migrate
```

This will perform the required schema updates to create the required database tables.

### Collect Static Files

On first run, the required static files must be collected into the `static` volume:

```
docker-compose run inventree-server invoke static
```

### Create Admin Account

You need to create an admin (superuser) account for the database. Run the command below, and follow the prompts:

```
docker-compose run inventree-server invoke superuser
```

### Configure InvenTree Options

By default, all required InvenTree settings are specified in the docker compose file, with the `INVENTREE_DB_` prefix.

You are free to skip this step, if these InvenTree settings meet your requirements.

If you wish to tweak the InvenTree configuration options, you can either:

#### Environment Variables

Alter (or add) environment variables into the docker-compose `environment` section

#### Configuration File

A configuration file `config.yaml` has been created in the data volume (at the location specified on your local disk).

Edit this file (as per the [configuration guidelines](./config.md)).

### Run Web Server

Now that the database has been created, migrations applied, and you have created an admin account, we are ready to launch the web server:

```
docker-compose up -d
```

This command launches the remaining containers:

- `inventree-server` - InvenTree web server
- `inventree-worker` - Background worker
- `inventree-nginx` - Nginx reverse proxy

!!! success "Up and Running!"
    You should now be able to view the InvenTree login screen at [http://localhost:1337](http://localhost:1337)

## Updating InvenTree

To update your InvenTree installation to the latest version, follow these steps:

### Stop Containers

Stop all running containers as below:

```
docker-compose down
```

### Update Images

Pull down the latest version of the InvenTree docker image

```
docker-compose pull
```

This ensures that the InvenTree containers will be running the latest version of the InvenTree source code.

### Start Containers

Now restart the containers.

As part of the server initialization process, data migrations and static file updates will be performed automatically.

```
docker-compose up -d
``` 

## Data Backup

Database and media files are stored external to the container, in the volume location specified in the `docker-compose.yml` file. It is strongly recommended that a backup of the files in this volume is performed on a regular basis.

### Exporting Database as JSON

To export the database to an agnostic JSON file, perform the following command:

```
docker-compose run inventree-server invoke export-records /home/inventree/data/data.json
```

This will export database records to the file `data.json` in your mounted volume directory.
