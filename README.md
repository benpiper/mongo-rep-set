# Dockerized MongoDB Replica Set

This MongoDB Docker container is intended to be used to set up a 3 node replica set.

Mongo version:  **3.4.0**

## About

A MongoDB [replica set](https://docs.mongodb.org/v3.4/replication/) consists of at least 3 Mongo instances. In this case, they will be a primary, secondary, and a tertiary. To use this project as a replica set, you simply launch three instances of this container across three separate host servers and the primary will configure your users and replica set.  Also note that each server must be able to access the others (discovery must work in both directions).

## Setup

You will need to create your own custom build of this image to generate a unique [keyfile](https://docs.mongodb.com/v3.4/tutorial/enforce-keyfile-access-control-in-existing-replica-set/) that Mongo uses for [internal authentication](https://docs.mongodb.org/v3.4/tutorial/enable-internal-authentication/) between replica set members. Each of your replica set members needs to have the same key, so be sure to use the same image in each location. Once using your build in production do NOT publish your Docker image to a public repo because it will container your private key.

#### Build

```sh
docker build -t yourname/mongo-rep-set:latest .
```

## Launch

Now you're ready to start launching containers.  You need to launch the secondary and tertiary first so they're ready for the primary to configure them when it starts.

#### Secondary and tertiary

```sh
docker run --name db2 -d -p 27017:27017 yourname/mongo-rep-set:latest
```

```sh
docker run --name db3 -d -p 27017:27017 yourname/mongo-rep-set:latest
```

#### Primary

The primary is responsible for setting up users and configuring the replica set, so this is where all of the configuration happens. Once your secondary and tertiary are up and running, launch your primary with:

```sh
docker run -d
  --name db1
  -p 27017:27017 \
  -e MONGO_ROLE="primary" \
  -e MONGO_SECONDARY="hostname or IP of secondary" \
  -e MONGO_TERTIARY="hostname or IP of tertiary" \
  -e MONGO_ROOT_USER="myRootUser" \
  -e MONGO_ROOT_PASSWORD="myRootUserPassword" \
  -e MONGO_APP_USER="myAppUser" \
  -e MONGO_APP_PASSWORD="myAppUserPassword" \
  -e MONGO_APP_DATABASE="myAppDatabase" \
  yourname/mongo-rep-set:latest
```

The primary will start up, configure your root and app users, shut down, and then start up once more and configure the replica set.  Assuming the secondary and tertiary are reachable, the server will now be ready for authenticated connections.  You can use the standard two server Mongo URL to connect to the primary/secondary like this:

#### Connect

Note that the following connection url is using default env var values (more info on those below), so it should work if you haven't overwritten any of the variables yourself.

```sh

mongodb://myAppUser:myAppUserPassword@db1:27017,db2:27017,db3:27017/myAppDatabase?replicaSet=rs0
```

## Environment Variables

Here are all of the available environment variables and their defaults.  Note that if you set `MONGO_REP_SET`, you must set it to the same value on all 3 containers.

```sh
# mongod config
MONGO_STORAGE_ENGINE wiredTiger
MONGO_JOURNALING true
MONGO_REP_SET rs0
MONGO_AUTH true
MONGO_OPLOG_SIZE # not set, but you can override the default
MONGO_SECONDARY db2:27017
MONGO_TERTIARY db3:27017
MONGO_DB_PATH /data/db

# mongo root user
MONGO_ROOT_USER root
MONGO_ROOT_PASSWORD Coast2018

# mongo app user + database (user is given oplog access)
MONGO_APP_USER appuser
MONGO_APP_PASSWORD App2018
MONGO_APP_DATABASE appdb
```

## Testing

From the worker running db1:

docker exec db1 mongo --host db2 --port 27017
docker exec db1 mongo --host db3 --port 27017
