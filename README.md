# postgres-docker-multiple-databases

Multiple databases in Postgres DB using Docker compose.

# The need
Postgres offers the ability to specify only one database when used in docker. This is done via the `POSTGRES_DB` environment variable. However, Postgres supports having multiple databases. 

# The solution
Mount the folder that contains a script into the `docker-entrypoint-initdb.d` folder inside the Postgres container. This script will be executed once the container is started. 

The script contains a for loop that reads the list of database names (supplied via environment variable) and creates a database with that name, and grants it the adequate permissions.

# The deep dive
This is the docker-compose.yml file: 
```
version: "3.9"

services:
  db:
    image: postgres:14.2
    container_name: postgres-db
    restart: always
    environment: 
      - POSTGRES_USER=${DB_USERNAME}
      - POSTGRES_PASSWORD=${DB_PASSWORD}
      - POSTGRES_MULTIPLE_DATABASES=${DATABASE_NAMES}
    ports:
      - 5432:5432
    volumes:
      - data-volume:/var/lib/postgresql/data
      - ./postgres-init-scripts:/docker-entrypoint-initdb.d

volumes:
  data-volume: 
```

## Breakdown:

There is just one service (`db`) that is the actual Postgres container. 

There are two volumes of which one of them is defined at the bottom (`data-volume`) and it holds (persists) the DB data. 

Let's focus on the `db` service.

- Image is `postgres:14.2`. Test it with a different version if needed. 
- container name is irrelevant (change it to your liking)
- environment variables are read from the `.env` file by default: NEVER PUSH YOUR `.env` FILE TO GIT. KEEP IT ON YOUR SERVER WITH ALL ITS SECRETS! The `.env-example` is provided as the example on how the `.env` should look like.
- port is irrelevant, the default one is used
- volumes are already explained: the first one is for the data persistence, and the second one mounts a directory with a script into the initdb

### Environment variables
You can notice that we did not define the `POSTGRES_DB` at all. That's because we pass on multiple database names using the `POSTGRES_MULTIPLE_DATABASES` environment variable.

The insides of the `.env` file should look like this: 
```
DB_USERNAME=postgres
DB_PASSWORD=postgres
DATABASE_NAMES=app1,app2
```

This means that our username will be `postgres`, our password would be `postgres` and we would create two databases - one named `app1`, and another named `app2`. You can have as many database names as you want. Just separate them with a comma `,`.

### The script
This is the script that actually creates the databases: 
```
#!/bin/bash

set -e
set -u

function create_user_and_database() {
	local database=$1
	echo "  Creating user and database '$database'"
	psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" <<-EOSQL
	    CREATE USER $database;
	    CREATE DATABASE $database;
	    GRANT ALL PRIVILEGES ON DATABASE $database TO $database;
EOSQL
}

if [ -n "$POSTGRES_MULTIPLE_DATABASES" ]; then
	echo "Multiple database creation requested: $POSTGRES_MULTIPLE_DATABASES"
	for db in $(echo $POSTGRES_MULTIPLE_DATABASES | tr ',' ' '); do
		create_user_and_database $db
	done
	echo "Multiple databases created"
fi

```

As you can see, there is a for loops through the list of databases provided in the `POSTGRES_MULTIPLE_DATABASES` environment variable. 

For each element in the list, the `create_user_and_database` function is called. 

This function creates the database (name of the database is the entry passed into the function; the `$db` variable). It also grants all privileges on that database. 

This script gets executed automatically when your container starts.


# Conclusion
And there you have it. Multiple databases in Postgres using docker-compose.

# Sources
- https://hub.docker.com/_/postgres 
- https://dev.to/bgord/multiple-postgres-databases-in-a-single-docker-container-417l 
- https://github.com/mrts/docker-postgresql-multiple-databases/blob/master/create-multiple-postgresql-databases.sh 

