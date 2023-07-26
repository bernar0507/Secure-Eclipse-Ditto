# Securing Eclipse Ditto through Authetication and Encryption at Rest in MongoDB

This repo will contain the steps to setup this Authetication and Encryption at Rest in MongoDB.

## 1 - Prepare the Directories
You need to create the directory on your host machine that will be used for the Docker volume. 
Remember, you have to replace /path/on/host/to/mongodb-decrypted in your YAML file with this actual path. Once you've decided on a directory, create it using the mkdir command. 
For example:
```
mkdir -p /path/on/host/to/mongodb-decrypted
```
Replace /path/on/host/to/mongodb-decrypted with the actual directory path.

## 2 - Install and Configure eCryptfs
Install eCryptfs on your host system. 
On a Debian-based system, this would look like:
```
sudo apt-get update
sudo apt-get install ecryptfs-utils
```

## 3 - Set up an Encrypted Mount point
Once eCryptfs is installed, you will set up an encrypted mount point. 
This involves creating an additional directory for the encrypted data, which should not be the same as the one you are using for the Docker volume.
```
sudo mkdir -p /path/to/encrypted/data
```

## 4 - Create the Encrypted Mount point
```
sudo mount -t ecryptfs /path/to/encrypted/data /path/on/host/to/mongodb-decrypted
```



* Select key type to use for newly created files: 
```
 1) tspi
 2) passphrase
```

* Select option 2:
```Selection: 2```

* Write the passphrase:
```Passphrase: ```


* Select cipher:
``` 
 1) aes: blocksize = 16; min keysize = 16; max keysize = 32
 2) blowfish: blocksize = 8; min keysize = 16; max keysize = 56
 3) des3_ede: blocksize = 8; min keysize = 24; max keysize = 24
 4) twofish: blocksize = 16; min keysize = 16; max keysize = 32
 5) cast6: blocksize = 16; min keysize = 16; max keysize = 32
 6) cast5: blocksize = 8; min keysize = 5; max keysize = 16
```

* Select option 1:
```Selection [aes]: 1```

* Select key bytes:
```
 1) 16
 2) 32
 3) 24
```

* Select option 1:
```Selection [16]: 1```

* Enable plaintext passthrough (y/n) [n]: ```n```

* Enable filename encryption (y/n) [n]: ```n```

* Attempting to mount with the following options:
  ecryptfs_unlink_sigs
  ecryptfs_key_bytes=16
  ecryptfs_cipher=aes
  ecryptfs_sig=6d619b4401f1f943
WARNING: Based on the contents of [/root/.ecryptfs/sig-cache.txt],
it looks like you have never mounted with this key 
before. This could mean that you have typed your 
passphrase wrong.

* Would you like to proceed with the mount (yes/no)? :
```yes```

* Would you like to append sig [6d619b4401f1f943] to [/root/.ecryptfs/sig-cache.txt]  in order to avoid this warning in the future (yes/no)? :
```yes```

* You should receive this message:
```
Successfully appended new sig to user sig cache file

Mounted eCryptfs
```

## 5 - Change permissions so Mongo can access

* change owner
```
sudo chown -R $(whoami):$(whoami) /path/to/directory/mongodb-decrypted
```

* change permissions
```
sudo chmod -R 777 /path/to/directory/mongodb-decrypted
```

# Update Ditto yaml file:
```
# Copyright (c) 2019 Contributors to the Eclipse Foundation
#
# See the NOTICE file(s) distributed with this work for additional
# information regarding copyright ownership.
#
# This program and the accompanying materials are made available under the
# terms of the Eclipse Public License 2.0 which is available at
# http://www.eclipse.org/legal/epl-2.0
#
# SPDX-License-Identifier: EPL-2.0
version: '2.4'

services:
  mongodb:
    image: docker.io/mongo:4.4
    mem_limit: 256m
    restart: always
    networks:
      default:
        aliases:
          - mongodb
    command: mongod --storageEngine wiredTiger --auth --dbpath /mongodb-decrypted
    user: mongodb
    ports:
      - 27017:27017
    volumes:
      - /home/bagao/project2/mongodb/script.js:/docker-entrypoint-initdb.d/script.js
      - /home/bagao/project2/mongodb-decrypted:/mongodb-decrypted
    environment:
       - MONGO_INITDB_ROOT_USERNAME=myuser
       - MONGO_INITDB_ROOT_PASSWORD=mypassword
       - TZ=Europe/Berlin

  policies:
    image: docker.io/eclipse/ditto-policies:${DITTO_VERSION:-latest}
    mem_limit: 512m
    restart: always
    networks:
      default:
        aliases:
          - ditto-cluster
    environment:
      - TZ=Europe/Berlin
      - BIND_HOSTNAME=0.0.0.0
      # Set additional configuration options here appending JAVA_TOOL_OPTIONS: -Dditto.policies...
      - JAVA_TOOL_OPTIONS=-XX:ActiveProcessorCount=2 -XX:+ExitOnOutOfMemoryError -XX:+UseContainerSupport -XX:+UseStringDeduplication -Xss512k -XX:MaxRAMPercentage=50 -XX:+UseG1GC -XX:MaxGCPauseMillis=150 -Dakka.coordinated-shutdown.exit-jvm=on -Dakka.cluster.shutdown-after-unsuccessful-join-seed-nodes=180s -Dakka.cluster.failure-detector.threshold=15.0 -Dakka.cluster.failure-detector.expected-response-after=10s -Dakka.cluster.failure-detector.acceptable-heartbeat-pause=20s -Dakka.cluster.downing-provider-class=
      - MONGO_DB_HOSTNAME=mongodb
      - MONGO_DB_AUTHENTICATION=myuser:mypassword@
      # in order to write logs into a file you can enable this by setting the following env variable
      # the log file(s) can be found in /var/log/ditto directory on the host machine
      # - DITTO_LOGGING_FILE_APPENDER=true
    # only needed if DITTO_LOGGING_FILE_APPENDER is set
    # volumes:
    #  - ditto_log_files:/var/log/ditto
    healthcheck:
      test: curl --fail `hostname`:8558/alive || exit 1
      interval: 30s
      timeout: 15s
      retries: 4
      start_period: 120s

  things:
    image: docker.io/eclipse/ditto-things:${DITTO_VERSION:-latest}
    mem_limit: 512m
    restart: always
    networks:
      default:
        aliases:
          - ditto-cluster
    depends_on:
      - policies
    environment:
      - TZ=Europe/Berlin
      - BIND_HOSTNAME=0.0.0.0
      # Set additional configuration options here appending JAVA_TOOL_OPTIONS: -Dditto.things...
      - JAVA_TOOL_OPTIONS=-XX:ActiveProcessorCount=2 -XX:+ExitOnOutOfMemoryError -XX:+UseContainerSupport -XX:+UseStringDeduplication -Xss512k -XX:MaxRAMPercentage=50 -XX:+UseG1GC -XX:MaxGCPauseMillis=150 -Dakka.coordinated-shutdown.exit-jvm=on -Dakka.cluster.shutdown-after-unsuccessful-join-seed-nodes=180s -Dakka.cluster.failure-detector.threshold=15.0 -Dakka.cluster.failure-detector.expected-response-after=10s -Dakka.cluster.failure-detector.acceptable-heartbeat-pause=20s -Dakka.cluster.downing-provider-class=
      - MONGO_DB_HOSTNAME=mongodb
      - MONGO_DB_AUTHENTICATION=myuser:mypassword@
      # in order to write logs into a file you can enable this by setting the following env variable
      # the log file(s) can be found in /var/log/ditto directory on the host machine
      # - DITTO_LOGGING_FILE_APPENDER=true
    # only needed if DITTO_LOGGING_FILE_APPENDER is set
    # volumes:
    #  - ditto_log_files:/var/log/ditto
    healthcheck:
      test: curl --fail `hostname`:8558/alive || exit 1
      interval: 30s
      timeout: 15s
      retries: 4
      start_period: 120s

  things-search:
    image: docker.io/eclipse/ditto-things-search:${DITTO_VERSION:-latest}
    mem_limit: 512m
    restart: always
    networks:
      default:
        aliases:
          - ditto-cluster
    depends_on:
      - policies
    environment:
      - TZ=Europe/Berlin
      - BIND_HOSTNAME=0.0.0.0
      # Set additional configuration options here appending JAVA_TOOL_OPTIONS: -Dditto.search...
      - JAVA_TOOL_OPTIONS=-XX:ActiveProcessorCount=2 -XX:+ExitOnOutOfMemoryError -XX:+UseContainerSupport -XX:+UseStringDeduplication -Xss512k -XX:MaxRAMPercentage=50 -XX:+UseG1GC -XX:MaxGCPauseMillis=150 -Dakka.coordinated-shutdown.exit-jvm=on -Dakka.cluster.shutdown-after-unsuccessful-join-seed-nodes=180s -Dakka.cluster.failure-detector.threshold=15.0 -Dakka.cluster.failure-detector.expected-response-after=10s -Dakka.cluster.failure-detector.acceptable-heartbeat-pause=20s -Dakka.cluster.downing-provider-class=
      - MONGO_DB_HOSTNAME=mongodb
      - MONGO_DB_AUTHENTICATION=myuser:mypassword@
      # in order to write logs into a file you can enable this by setting the following env variable
      # the log file(s) can be found in /var/log/ditto directory on the host machine
      # - DITTO_LOGGING_FILE_APPENDER=true
    # only needed if DITTO_LOGGING_FILE_APPENDER is set
    # volumes:
    #  - ditto_log_files:/var/log/ditto
    healthcheck:
      test: curl --fail `hostname`:8558/alive || exit 1
      interval: 30s
      timeout: 15s
      retries: 4
      start_period: 120s

  connectivity:
    image: docker.io/eclipse/ditto-connectivity:${DITTO_VERSION:-latest}
    mem_limit: 768m
    restart: always
    networks:
      default:
        aliases:
          - ditto-cluster
    depends_on:
      - policies
    environment:
      - TZ=Europe/Berlin
      - BIND_HOSTNAME=0.0.0.0
      # if connections to rabbitmq broker are used, you might want to disable ExitOnOutOfMemoryError, because the amqp-client has a bug throwing OOM exceptions and causing a restart loop
      # Set additional configuration options here appending JAVA_TOOL_OPTIONS: -Dditto.connectivity...
      - JAVA_TOOL_OPTIONS=-XX:ActiveProcessorCount=2 -XX:+ExitOnOutOfMemoryError -XX:+UseContainerSupport -XX:+UseStringDeduplication -Xss512k -XX:MaxRAMPercentage=50 -XX:+UseG1GC -XX:MaxGCPauseMillis=150 -Dakka.coordinated-shutdown.exit-jvm=on -Dakka.cluster.shutdown-after-unsuccessful-join-seed-nodes=180s -Dakka.cluster.failure-detector.threshold=15.0 -Dakka.cluster.failure-detector.expected-response-after=10s -Dakka.cluster.failure-detector.acceptable-heartbeat-pause=20s -Dakka.cluster.downing-provider-class=
      - MONGO_DB_HOSTNAME=mongodb
      - MONGO_DB_AUTHENTICATION=myuser:mypassword@
      # in order to write logs into a file you can enable this by setting the following env variable
      # the log file(s) can be found in /var/log/ditto directory on the host machine
      # - DITTO_LOGGING_FILE_APPENDER=true
    # only needed if DITTO_LOGGING_FILE_APPENDER is set
    #volumes:
    #  - ditto_log_files:/var/log/ditto
    healthcheck:
      test: curl --fail `hostname`:8558/alive || exit 1
      interval: 30s
      timeout: 15s
      retries: 4
      start_period: 120s

  gateway:
    image: docker.io/eclipse/ditto-gateway:${DITTO_VERSION:-latest}
    mem_limit: 512m
    restart: always
    networks:
      default:
        aliases:
          - ditto-cluster
    depends_on:
      - policies
    ports:
      - "8081:8080"
    environment:
      - TZ=Europe/Berlin
      - BIND_HOSTNAME=0.0.0.0
      - ENABLE_PRE_AUTHENTICATION=true
      # Set additional configuration options here appending JAVA_TOOL_OPTIONS: -Dditto.gateway.authentication.devops.password=foobar -Dditto.gateway...
      - JAVA_TOOL_OPTIONS=-XX:ActiveProcessorCount=2 -XX:+ExitOnOutOfMemoryError -XX:+UseContainerSupport -XX:+UseStringDeduplication -Xss512k -XX:MaxRAMPercentage=50 -XX:+UseG1GC -XX:MaxGCPauseMillis=150 -Dakka.coordinated-shutdown.exit-jvm=on -Dakka.cluster.shutdown-after-unsuccessful-join-seed-nodes=180s -Dakka.cluster.failure-detector.threshold=15.0 -Dakka.cluster.failure-detector.expected-response-after=10s -Dakka.cluster.failure-detector.acceptable-heartbeat-pause=20s -Dakka.cluster.downing-provider-class=
      # in order to write logs into a file you can enable this by setting the following env variable
      # the log file(s) can be found in /var/log/ditto directory on the host machine
      # - DITTO_LOGGING_FILE_APPENDER=true
      # You may use the environment for setting the devops password
      #- DEVOPS_PASSWORD=foobar
    # only needed if DITTO_LOGGING_FILE_APPENDER is set
    # volumes:
    #  - ditto_log_files:/var/log/ditto
    healthcheck:
      test: curl --fail `hostname`:8558/alive || exit 1
      interval: 30s
      timeout: 15s
      retries: 4
      start_period: 120s

  swagger-ui:
    image: docker.io/swaggerapi/swagger-ui:v4.14.3
    mem_limit: 32m
    restart: always
    environment:
      - QUERY_CONFIG_ENABLED=true
    volumes:
       - ../../documentation/src/main/resources/openapi:/usr/share/nginx/html/openapi:ro
       - ../../documentation/src/main/resources/images:/usr/share/nginx/html/images:ro
       - ./swagger3-index.html:/usr/share/nginx/html/index.html:ro
    command: nginx -g 'daemon off;'

  nginx:
    image: docker.io/nginx:1.21-alpine
    mem_limit: 32m
    restart: always
    volumes:
       - ./nginx.conf:/etc/nginx/nginx.conf:ro
       - ./nginx.htpasswd:/etc/nginx/nginx.htpasswd:ro
       - ./nginx-cors.conf:/etc/nginx/nginx-cors.conf:ro
       - ./mime.types:/etc/nginx/mime.types:ro
       - ./index.html:/etc/nginx/html/index.html:ro
       - ../../documentation/src/main/resources/images:/etc/nginx/html/images:ro
       - ../../documentation/src/main/resources/wot:/etc/nginx/html/wot:ro
       - ../../ui:/etc/nginx/html/ui:ro
    ports:
      - "${DITTO_EXTERNAL_PORT:-8080}:80"
    depends_on:
      - gateway
      - swagger-ui

volumes:
  ditto_log_files:
    driver: local
    driver_opts:
      type: none
      device: /var/log/ditto
      o: bind,uid=1000,gid=1000
```

We need to:
* Add these flags `--auth --dbpath /mongodb-decrypted` and remove `--noscripting`
* Add volumes section to mongodb service:
```
volumes:
      - /path/to/script.js:/docker-entrypoint-initdb.d/script.js
      - /path/to/mongodb-decrypted:/mongodb-decrypted
    environment:
       - MONGO_INITDB_ROOT_USERNAME=myuser
       - MONGO_INITDB_ROOT_PASSWORD=mypassword
       - TZ=Europe/Berlin
```
* Also need to create this `script.js`:
```
var user = 'myuser';
var pwd = 'mypassword';

var databases = ['ditto', 'things', 'search', 'connectivity', 'policies'];

for (var i = 0; i < databases.length; i++) {
  var db = db.getSiblingDB(databases[i]);
  db.createUser({
    user: user,
    pwd: pwd,
    roles: [
      {
        role: 'readWrite',
        db: databases[i],
      },
    ],
  });
}
```

Then we just need to do: `docker-compose up -d` and we will have encryption at rest and authetication in mongodb.
