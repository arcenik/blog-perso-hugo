+++
date = '2026-04-08T10:20:42+02:00'
title = 'Extracting MongoDB from the Unifi container'
+++

For many years, I've ran my Unifi network controller with Docker Compose using included mongodb server.

But now it is time to change this. To externalize and upgrade it.

## Previous situation

Until now, Unifi was deployed using the included MongoDB server.

```yaml
  unifi:
    image: goofball222/unifi:10.0.160-ubuntu
    hostname: unifi
    user: unifi
    restart: always
    ports:
      - 3478:3478/udp # STUN connection
      - 6789:6789 # throughput measurement from Android/iOS app
      - 8080:8080 # UAP/USW/USG to inform controller
      - 8443:8443 # controller GUI / API
      - 8880:8880 # HTTP portal redirect
      - 8843:8843 # HTTPS portal redirect
      - 10001:10001/udp # UBNT discovery broadcasts
    environment:
      - DB_MONGO_LOCAL=true
    volumes:
      - /srv/unifi/data:/usr/lib/unifi/data
      - /srv/unifi/log:/usr/lib/unifi/log
      - /srv/unifi/cert:/usr/lib/unifi/cert
```

## Migration plan

1. Check Unifi backup
2. Stop the Unifi container
3. Copy/move the MongoDB folder to the new location
4. Start the MongoDB container, using the same version
5. Update Unifi Docker Compose to use the external MongoDB
6. Start the Unifi container

### Check Unifi backup

```sh
# cd /srv
# ls -lh unifi/data/backup/
```
### Stop the Unifi container

```sh
$ docker compose down unifi
```

### Copy the data

```sh
# cd /srv
# cp -rp unifi/data/db mongounifi
```

### New mongounifi service

Using the same version as in the Unifi container, to be upgraded later.

```yaml
services:
  mongounifi:
    image: mongo:3.6.23
    hostname: mongounifi
    restart: always
    volumes:
      - /srv/mongounifi:/data/db
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongo mongodb://localhost --quiet
      interval: 5s
      timeout: 1s
      retries: 5
```

### New unifi service

Now upgrade the unifi service with non-local MongoDB database and a new dependency.

```yaml
  unifi:
    image: goofball222/unifi:10.0.160-alpine
    hostname: unifi
    depends_on:
      mongounifi:
        condition: service_healthy
        restart: true
    user: unifi
    restart: always
    ports:
      - 3478:3478/udp # STUN connection
      - 6789:6789 # throughput measurement from Android/iOS app
      - 8080:8080 # UAP/USW/USG to inform controller
      - 8443:8443 # controller GUI / API
      - 8880:8880 # HTTP portal redirect
      - 8843:8843 # HTTPS portal redirect
      - 10001:10001/udp # UBNT discovery broadcasts
environment:
      - DB_MONGO_LOCAL=false
      - DB_MONGO_URI=mongodb://mongounifi:27017/ace
      - STATDB_MONGO_URI=mongodb://mongounifi:27017/ace_stat
      - TZ=UTC
      - UNIFI_DB_NAME=ace
    volumes:
      - /srv/unifi/data:/usr/lib/unifi/data
      - /srv/unifi/log:/usr/lib/unifi/log
      - /srv/unifi/cert:/usr/lib/unifi/cert
```

### Start the Unifi container

```sh
$ docker compose up -d unifi
$ docker compose ps
$ docker compose logs -f unifi
```

## MongoDB upgrade plan

Looping over each major versions:
1. Upgrade the MongoDB version in the Docker Compose file
2. Restart the MongoDB container
3. Upgrade the MongoDB database

### Upgrade to v4.0

Change the mongounifi tag and restart the service. Then connect to it to upgrade the database.

```sh
$ docker compose exec mongounifi mongo
MongoDB shell version v4.0.28
connecting to: mongodb://127.0.0.1:27017/?gssapiServiceName=mongodb

> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
{ "featureCompatibilityVersion" : { "version" : "3.6" }, "ok" : 1 }

> db.adminCommand({ setFeatureCompatibilityVersion: "4.0" })
{ "ok" : 1 }

> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
{ "featureCompatibilityVersion" : { "version" : "4.0" }, "ok" : 1 }
```

### Upgrade to v4.2

Change the mongounifi tag and restart the service. Then connect to it to upgrade the database.

```sh
$ docker compose exec mongounifi mongo
MongoDB shell version v4.2.24
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb

> db.adminCommand({ setFeatureCompatibilityVersion: "4.2" })
{ "ok" : 1 }

> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
{ "featureCompatibilityVersion" : { "version" : "4.2" }, "ok" : 1 }
```

### Upgrade to v4.4

Change the mongounifi tag and restart the service. Then connect to it to upgrade the database.

```sh
$ docker compose exec mongounifi mongo
MongoDB shell version v4.4.30
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb

> db.adminCommand({ setFeatureCompatibilityVersion: "4.4" })
{ "ok" : 1 }

> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
{ "featureCompatibilityVersion" : { "version" : "4.4" }, "ok" : 1 }
```

### Upgrade to v5.0

Change the mongounifi tag and restart the service. Then connect to it to upgrade the database.

```sh
$ docker compose exec mongounifi mongo
MongoDB shell version v5.0.32
connecting to: mongodb://127.0.0.1:27017/?compressors=disabled&gssapiServiceName=mongodb

> db.adminCommand({ setFeatureCompatibilityVersion: "5.0" })
{ "ok" : 1 }
> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
{ "featureCompatibilityVersion" : { "version" : "5.0" }, "ok" : 1 }
```

### Upgrade to v6.0

Change the mongounifi tag and healthcheck then restart the service. Then connect to it to upgrade the database.

With version 6, the CLI command is now **monbosh**.

```yaml
healthcheck:
  test: echo 'db.runCommand("ping").ok' | mongosh mongodb://localhost --quiet
  interval: 5s
  timeout: 1s
  retries: 5
```

```sh
$ docker compose exec mongounifi mongosh
Current Mongosh Log ID: 69d640ccd62f8e25e48de665
Connecting to:          mongodb://127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.5.10
Using MongoDB:          6.0.27
Using Mongosh:          2.5.10

test> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
{ featureCompatibilityVersion: { version: '6.0' }, ok: 1 }

test> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
{ featureCompatibilityVersion: { version: '6.0' }, ok: 1 }
```

### Upgrade to v7.0

Change the mongounifi tag then restart the service. Then connect to it to upgrade the database.

```sh
$ docker compose exec mongounifi mongosh
Current Mongosh Log ID: 69d641c2454b05bed544ba88
Connecting to:          mongodb://127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.8.2
Using MongoDB:          7.0.31
Using Mongosh:          2.8.2

test> db.adminCommand({ setFeatureCompatibilityVersion: "7.0", confirm: true })
{ ok: 1 }

test> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
{ featureCompatibilityVersion: { version: '7.0' }, ok: 1 }
```

### Upgrade to v8.0

Change the mongounifi tag then restart the service. Then connect to it to upgrade the database.

```sh
$ docker compose exec mongounifi mongosh
Current Mongosh Log ID: 69d642536ab5313bd244ba88
Connecting to:          mongodb://127.0.0.1:27017/?directConnection=true&serverSelectionTimeoutMS=2000&appName=mongosh+2.8.2
Using MongoDB:          8.0.20
Using Mongosh:          2.8.2

test> db.adminCommand({ setFeatureCompatibilityVersion: "8.0", confirm: true })
{ ok: 1 }

test> db.adminCommand( { getParameter: 1, featureCompatibilityVersion: 1 } )
{ featureCompatibilityVersion: { version: '8.0' }, ok: 1 }
```



