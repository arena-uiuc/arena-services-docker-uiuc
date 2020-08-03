# Compose arena services

The [docker-compose.yaml](docker-compose.yaml) creates several containers with ARENA services:

* Web server for ARENA (Nginx)
* Database (MongoDB)
* Pubsub (mosquitto)
* Persistence service
* ARTS
* File Upload (Droppy)
* Certbot

We clone **ARENA-core** and **arena-persist** to create containers with these services. The **ARENA-core** files are copied into the web server container (called ```arena-web```) at build time (thus, updates to ARENA-core require a rebuild of the container).

Nginx and mosquitto are configured with TLS/SSL using certificates created by certbot (running as a service in a container), which will periodically attempt to renew the certificates. On the first execution, certbot must be initialized. See [Certbot Init](certbot-init) Section below.

## Quick Setup

1. We need [docker](https://docs.docker.com/get-docker/) and [docker-compose](https://docs.docker.com/compose/install/) installed.
2. Clone this repo (with ```--recurse-submodules``` to make sure you get the contents of the repositories added as submodules):

```bash
git clone git@github.com:conix-center/arena-services-docker.git --recurse-submodules
```

3. Modify configuration:

- Add domains and email addresses to **init-letsencrypt.sh**  (e.g. ```mr.andrew.cmu.edu```)
- Update ```conf/nginx-conf.d/arena-web.conf``` and ```conf/mosquitto.conf``` to reflect these changes (the server name and location of the certificates might need to be updated).
- Check hostname in the **arts** service config (```docker-compose.yaml```)

4. Run letsencrypt init script:

```bash
 ./init-letsencrypt.sh
```

* If you see no errors; you are good to go. For details, see [Certbot Init](certbot-init) Section below.

4. Create a user and password to protect the web server's ```/upload/``` area by opening the ```/upload``` URL (e.g. ```https://mr.andrew.cmu.edu/upload```)  in your browser. See details in the [Asset Upload](asset-upload) Section below.

## Certbot Init

Before starting services, we need to configure certbot with the right domains and then execute **init-letsencrypt.sh** (needs [openssl](https://www.openssl.org/)):

1. Modify configuration:

- Add domains and email addresses to **init-letsencrypt.sh**  (e.g. ```mr.andrew.cmu.edu```)
- Update ```conf/nginx-conf.d/arena-web.conf``` and ```conf/mosquitto.conf``` to reflect these changes (the server name and location of the certificates might need to be updated).
- Check hostname in the **arts** service config (```docker-compose.yaml```)

2. Run the init script:

```bash
 ./init-letsencrypt.sh
```

## Services Config

The host machine/domains are referenced in several config files. These need to be updated to reflect the machine/domain.

- ```conf/nginx-conf.d/arena-web.conf```:

```
...
ssl_certificate     /etc/letsencrypt/live/[change-to-reflect-hostname]/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/[change-to-reflect-hostname]/privkey.pem;
...
```

- ```conf/mosquitto.conf```:
```
...
listener 8083
protocol websockets
certfile /etc/letsencrypt/live/[change-to-reflect-hostname]/fullchain.pem
keyfile /etc/letsencrypt/live/[change-to-reflect-hostname]/privkey.pem

listener 8883
protocol mqtt
certfile /etc/letsencrypt/live/[change-to-reflect-hostname]/fullchain.pem
keyfile /etc/letsencrypt/live/[change-to-reflect-hostname]/privkey.pem
...
```
- ```conf/arts-settings.py```:
```
...
    'mqtt_server': { 'host': '[change-to-reflect-hostname]', 'port': 1883, 'ws_port': 9001, 'wss_port': 8083 },
...
```

## Assets Upload

The web server files under ```/assets``` (e.g. ```https://mr.andrew.cmu.edu/assets```) can be uploaded via a web interface available at ```/upload```  (e.g. ```https://mr.andrew.cmu.edu/upload```) . The uploads area is protected by a user and password that needs to be setup the first time we access it.

**Be sure to open the ```/upload``` URL on your browser and setup the user and password.**

## Update Submodules

To update the repositories added as submodules (**ARENA-core** and **arena-persist**), run:

```bash
./update-submodules.sh
```

After updating the submodules, to have the updates refected live, you will need to restart the services:

```bash
docker-compose down; docker-compose up -d --force-build
```

*  See [Compose Quick Reference](compose-quick-reference) for the description of these commands.

## Files/Folders Description

* **ARENA-core:**	Contents of the ARENA-core repository (submodule).
* **arena-persist:**	Contents of the arena-persist repository (submodule).
* **conf:** Configuration files for the services (e.g. certificates, mosquito, nginx, persistence). Some important files described below.
  * *mosquitto.conf*: configures listners on ports 8833 (mqtt), 9001 (mqtt-ws), 8083 (mqtt-wss) and 8883 (mqtt-tls); certificate files under ```/data/certbot/conf``` are mapped to ```/etc/letsencrypt``` in the container.
  * *nginx-conf.d/arena-web.conf*: configures the web server to serve a proxy to port 9001 under ```/mqtt/```, forwards requests to```/persist/``` to the **arena-persist** service and requests tp ```/upload``` to the **droppy** service;  certificate files under ```/data/certbot/conf``` are mapped to ```/etc/letsencrypt``` in the container.
  * *persist-config.json*: configures the mongodb uri to the container service name.
  * *arts-settings.py*: configuration of arts.
* **data:** Data files (e,g, certificates generated by certbot, mongodb database, uploaded files).
* **docker-compose.yaml:** Compose file that describes all services.
* **init-letsencrypt.sh:** Initialize certbot. See [Certbot Init](certbot-init) Section.
* **update-submodules.sh:** Run this to get the latest updates from the repositories added as submodules (**ARENA-core**, **arena-persist**). You will need to restart the services to have the changes live (see [Update Submodules](update-sybmodules)).

## Compose Quick Reference

**Start services and see their output/logs**

- ```docker-compose up``` (add ```--force-build  ``` to build containers after updating submodules)

**Start the services in "detached" (daemon) mode (-d)**

- ```docker-compose up -d``` (add ```--force-build  ``` to build containers after updating submodules)

**Start just a particular service**

- ```docker-compose start <service name in docker-compose.yml>```

**Stop services**

- ```docker-compose down```

**Restart the services**

- ```docker-compose restart```

**See logs**

- ```docker-compose logs```
