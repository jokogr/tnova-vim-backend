# Deploying T-NOVA VIM Monitoring Back-end

For the convenience of the end-users, an official Docker image is provided.
[Docker](https://www.docker.com) is an open-source project that automates the
deployment of applications inside software containers, by providing an
additional layer of abstraction and automation of operating-system-level
virtualisation on Linux [^1].

In order to deploy the VIM monitoring back-end, be sure to deploy the
[necessary Docker containers](#running-docker-containers) and [to configure
accordingly the monitoring agents](#configuring-the-monitoring-agents).

## Running Docker containers

### InfluxDB

The monitoring back-end requires an [InfluxDB](https://influxdata.com/)
instance to host the monitoring data. The [official Docker
image](https://hub.docker.com/_/influxdb/) is used in the T-NOVA testing
infrastructure and is highly recommended.

Run it with the following command:

``` command-line
docker run -d --name influxdb \
    --restart always \
    -p 8083:8083 -p 8086:8086 -p 25826:25826/udp \
    -v $PWD/influxdata:/var/lib/influxdb \
    -v $PWD/types.db:/usr/share/collectd/types.db \
    -e INFLUXDB_REPORTING_DISABLED=true \
    -e INFLUXDB_COLLECTD_ENABLED=true \
    -e INFLUXDB_COLLECTD_BIND_ADDRESS=":25826" \
    -e INFLUXDB_COLLECTD_DATABASE="statsdb" \
    -e INFLUXDB_COLLECTD_TYPESDB="/usr/share/collectd/types.db" \
    influxdb:alpine
```

The `docker run` command above had the following options:

* `-d`: This is to start the container in detached mode.
* `--name influxdb`: This is an identifier of the Docker container.
* `--restart=always`: Always restart the container regardless of the exit
  status. This is to ensure starting the container during the Docker daemon
  start, in case the host restarts.
* `--volume` option: The only data volume is used here to persist the database
  files.
* `-p` options: The published port 8083 provides an HTTP user interface,
  port 8096 an HTTP API and port 25826 the collectd interface.
* `--env` options: This is to create the database (*statsdb* in the example)
  that is going to be used for storing collectd data, and to set the listening
  port for collectd connections.

### Monitoring Back-End

Run the following command:

``` command-line
docker run --name monitoring_backend -d --restart=always \
    --env 'CEILOMETER_ENABLED=true' \
    --env 'CEILOMETER_HOST=localhost' --env 'CEILOMETER_PORT=8777' \
    --env 'POLLING_INTERVAL=600000' \
    --env 'NOVA_HOST=localhost' --env 'NOVA_PORT=8774' \
    --env 'IDENTITY_HOST=localhost' --env 'IDENTITY_PORT=5000' \
    --env 'IDENTITY_TENANT=tenant' \
    --env 'IDENTITY_USERNAME=username' --env 'IDENTITY_PASSWORD=pass' \
    --env 'ALARM_POLLING_INTERVAL=10000' \
    --volume /srv/docker/tnova_vim/subscriptions.json:/subscriptions.json \
    --link influxdb:influxdb \
    --publish 8080:3000 \
    spacehellas/tnova-vim-backend:latest
```

The `docker run` command above had the following options:

* `--name monitoring_backend`: This is an identifier of the Docker container.
* `-d`: This is to start the container in detached mode.
* `--restart=always`: Always restart the container regardless of the exit
  status. This is to ensure starting the container during the Docker daemon
  start, in case the host restarts.
* `--env` options: The environment variables are explained in the [next
  section](#environment-variables).
* `--volume` option: The only data volume is used here to persist the
  `/subscriptions.json` file, so that subscriptions are not lost between
  restarts.
* `--link influxdb:influxdb`: This option links the back-end container with the
  InfluxDB one. Docker bridges this way the two containers automatically and
  the back-end container can detect which ports InfluxDB listens to.
* `--publish 8080:3000`: The back-end application listens to port 3000 inside
  the container.

#### Environment Variables

Name | Description
-----|------------|
**CEILOMETER_ENABLED** | Defines if the monitoring backend polls the OpenStack Ceilometer service (default: true)
**CEILOMETER_HOST** | Defines the host of the OpenStack Ceilometer service
**CEILOMETER_PORT** | Defines the port of the OpenStack Ceilometer service
**POLLING_INTERVAL** | Defines the polling interval to the OpenStack Ceilometer service (in milliseconds)
**NOVA_HOST** | Defines the host of the OpenStack Nova service
**NOVA_PORT** | Defines the port of the OpenStack Nova service
**IDENTITY_HOST** | Defines the host of the OpenStack Identity (Keystone) service
**IDENTITY_PORT** | Defines the port of the OpenStack Identity (Keystone) service
**IDENTITY_TENANT** | Defines the OpenStack tenant's name
**IDENTITY_USERNAME** | Defines the OpenStack username
**IDENTITY_PASSWORD** | Defines the OpenStack password
**ALARM_POLLING_INTERVAL** | Defines the polling interval for alarm triggering (in milliseconds)

### Grafana (optional)

Feel free to use [Grafana](http://grafana.org/) for visualising the available
monitoring data:

``` command-line
docker run --name grafana -d --restart=always \
    --publish 3000:3000 \
    grafana/grafana:latest
```

## Configuring the monitoring agents

The VIM Monitoring Back-End uses collectd to collect the VNF instance
monitoring data. Most importantly, it requires [the collectd network
plugin](https://collectd.org/wiki/index.php/Plugin:Network) to be setup
against the InfluxDB in order to send monitoring data:

```
<Plugin network>
  Server "<influxdb_host>" "<influxdb_port>"
  ReportStats false
</Plugin>
```

where *<influxdb_host>* and *<influxdb_port>* are the hostname and the port
that are set previously for the InfluxDB Docker container.

For a more complete collectd configuration file, please refer to [the
recommended collectd configuration
template](../contrib/collectd.conf.template).

## How the Back-End is dockerised

[The official Docker
image](https://hub.docker.com/r/spacehellas/tnova-vim-backend/)
is based on [a minimal Node.js Docker
image](https://github.com/mhart/alpine-node).  It was selected due to its small
size: it does not exceed 40 MB. This image is built on [Alpine
Linux](https://alpinelinux.org/) and contains additionally just the node as a
static binary with no npm.

As such, it is required to run `npm install` locally *before* running `docker
build`, so that Docker just copies all the necessary files and the image will
not need `npm` installed in.

### Building the Docker image

Having ensured that `docker` and `npm` are working, run the following commands
in the project's root directory:

``` command-line
npm install
docker build --tag=spacehellas/tnova-vim-backend .
```

[^1]: O'Gara, Maureen (26 July 2013). "[Ben Golub, Who Sold Gluster to Red Hat,
Now Running dotCloud](http://maureenogara.sys-con.com/node/2747331)". SYS-CON
Media. Retrieved 2016-01-15.
