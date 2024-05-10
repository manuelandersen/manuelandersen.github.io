---
title: Docker-Compose 
date: last-modified
description: Running Postgres and pgAdmin with Docker-Compose
---

## Terminology

- `docker-compose`: is a convenient way to run multiple related services with just one config file.

## Useful links

- docker compose [official page](https://docs.docker.com/compose/).

## What is Docker Compose?

`docker-compose` allows us to launch multiple containers using a single configuration file, so that we don't have to run multiple complex `docker run` commands separately.

First of all, we need to create a file called `docker-compose.yaml`:

```YAML
services:
  pgdatabase:
    image: postgres:13
    environment:
      - POSTGRES_USER=root
      - POSTGRES_PASSWORD=root
      - POSTGRES_DB=ny_taxi
    volumes:
      - "./ny_taxi_postgres_data:/var/lib/postgresql/data:rw"
    ports:
      - "5432:5432"
  pgadmin:
    image: dpage/pgadmin4
    environment:
      - PGADMIN_DEFAULT_EMAIL=admin@admin.com
      - PGADMIN_DEFAULT_PASSWORD=root
    volumes:
      - "./data_pgadmin:/var/lib/pgadmin"
    ports:
      - "8080:80"
```

here:

- `services` will be all the services we want in the compose, in our case we want the `pgdatabase` and `pgadmin`.
- `image` is the image we want for every service.
- `environment` are all the environment variables each services needs to run.
- `volumes` is for doing volume mapping in the form `hostPath:containerPath:mode`
- `ports` are for accesing the ports, in the form `hostPort:containerPort`

Now, before running the `docker-compose` make sure you stop every container by cheking with `docker ps`. Now for running the compose we do:

```bash
docker-compose up
```

a few notes:

- We don't have to specify a network because docker-compose takes care of it: every single container (or "service", as the file states) will run withing the same network and will be able to find each other according to their names (pgdatabase and pgadmin in this example).
- We've added a volume for pgAdmin to save its settings, so that you don't have to keep re-creating the connection to Postgres every time ypu rerun the container. Make sure you create a data_pgadmin directory in your work folder where you run docker-compose from.
- All other details from the docker run commands (environment variables, volumes and ports) are mentioned accordingly in the file following YAML syntax.

For shutting down, first you type `Ctrl+C` and then 

```bash
docker-compose down
```
