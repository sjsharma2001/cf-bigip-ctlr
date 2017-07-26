F5 BIG-IP Controller for Cloud Foundry
======================================

The F5 BIG-IP Controller for [Cloud Foundry](http://cloudfoundry.org) makes the F5 BIG-IP
[Local Traffic Manager](<https://f5.com/products/big-ip/local-traffic-manager-ltm)
services available to applications running in the Cloud Foundry platform.

Documentation
-------------

For instructions on how to use this component, use the
[F5 BIG-IP Controller for Cloud Foundry docs](http://clouddocs.f5.com/products/connectors/cf-bigip-ctlr/latest/).

For guides on this and other solutions for Cloud Foundry, see the
[F5 Solution Guides for Cloud Foundry](http://clouddocs.f5.com/containers/latest/cloudfoundry).

Running
-------

The official docker image is `f5networks/cf-bigip-ctlr`.

Usually, the controller is deployed in Cloud Foundry. However, the controller can be run locally for development testing.
The controller requires a running NATS server, without a valid connection the controller will not start. The controller
can either be run against a gnatsd server or standalone against a Cloud Foundry installation.

Option gnatsd:
- Go should be installed and in the PATH
- GOPATH should be set as described in http://golang.org/doc/code.html
- Pip install the python/cf-runtime-requirements.txt into a virtualenv of your choice
```bash
workon cf-bigip-ctlr
pip install -r python/cf-runtime-requirements.txt
```
- [gnatds](https://github.com/nats-io/gnatds) installed and in the PATH
```
go get github.com/nats-io/gnatsd
gnatsd &
```
- Optionally run unit tests
```
go get github.com/onsi/ginkgo
go get github.com/onsi/gomega
ginkgo -keepGoing -trace -p -progress -r -failOnPending -randomizeAllSpecs -race
```
- Build and install controller from a cloned [cf-bigip-ctlr](https://github.com/F5Networks/cf-bigip-ctlr)
```
go install
```
- Update configuration file or BIGIP_CTLR_CFG environment variable for your specific environment
  as described in "Configuration"
- Run the controller
```
cf-bigip-ctlr -c [CONFIG_FILE]
```

Option standalone:
- Go should be installed and in the PATH
- GOPATH should be set as described in http://golang.org/doc/code.html
- Pip install the python/cf-runtime-requirements.txt into a virtualenv of your choice
```bash
workon cf-bigip-ctlr
pip install -r python/cf-runtime-requirements.txt
```
- Optionally run unit tests
```
go get github.com/onsi/ginkgo
go get github.com/onsi/gomega
ginkgo -keepGoing -trace -p -progress -r -failOnPending -randomizeAllSpecs -race
```
- Build and install controller from a cloned [cf-bigip-ctlr](https://github.com/F5Networks/cf-bigip-ctlr)
```
go install
```
- Update configuration file or BIGIP_CTLR_CFG environment variable for your specific environment
  as described in "Configuration"
- Run the controller
```
cf-bigip-ctlr -c [CONFIG_FILE]
```

Building
--------

The official images are built using docker, but standard go build tools can be used for development
purposes as described above.

### Official Build

Prerequisites:
- Docker

```bash
git clone https://github.com/F5Networks/cf-bigip-ctlr.git
cd cf-bigip-ctlr

# Use docker to build the release artifacts into a local "_docker_workspace" directory and push into docker images
make prod
```

### Alternate, unofficial build

A normal go and godep toolchain can be used as well

Prerequisites:
- go 1.7
- GOPATH pointing at a valid go workspace
- godep (Only needed to modify vendor's packages)
- python
- virtualenv

```bash
mkdir -p $GOPATH/src/github.com/F5Networks
cf $GOPATH/src/github.com/F5Networks
git clone https://github.com/F5Networks/cf-bigip-ctlr.git
cd cf-bigip-ctlr

# Building all packages, and run unit tests
make prod
```

Configuration
-------------

When pushing the controller into a Cloud Foundry environment a configuration must be passed
via the application manifest. An example manifest is located in the example_config directory.

Update required sections for environment:
- nats: leave empty for gnatsd otherwise update with CF installed NATS information
- bigip: leave empty if no BigIP is required otherwise update with BigIP information
- routing_api: only required if routing API access is required
- oauth: only required if routing API access is required

Development
-----------

**Note**: This repository should be imported as `github.com/F5Networks/cf-bigip-ctlr`.

## Dynamic Routing Table

The controller's routing table is updated dynamically via the NATS message bus.
NATS can be deployed via BOSH with
([cf-release](https://github.com/cloudfoundry/cf-release)) or standalone using
[nats-release](https://github.com/cloudfoundry/nats-release).

To add or remove a record from the routing table, a NATS client must send
register or unregister messages. Records in the routing table have a maximum
TTL of 120 seconds, so clients must heartbeat registration messages
periodically; we recommend every 20s. [Route
Registrar](https://github.com/cloudfoundry/route-registrar) is a BOSH job that
comes with [Routing
Release](https://github.com/cloudfoundry-incubator/routing-release) that
automates this process.

When deployed with Cloud Foundry, registration of routes for apps pushed to CF
occurs automatically without user involvement. For details, see [Routes and
Domains](https://docs.cloudfoundry.org/devguide/deploy-apps/routes-domains.html).

### Registering Routes via NATS

When the controller starts, it sends a `router.start` message to NATS. This
message contains an interval that other components should then send
`router.register` on, `minimumRegisterIntervalInSeconds`. It is recommended
that clients should send `router.register` messages on this interval. This
`minimumRegisterIntervalInSeconds` value is configured through the
`start_response_delay_interval` configuration property. The controller will prune
routes that it considers to be stale based upon a seperate "staleness" value,
`droplet_stale_threshold`, which defaults to 120 seconds. The controller will check
if routes have become stale on an interval defined by
  `prune_stale_droplets_interval`, which defaults to 30 seconds. All of these
  values are represented in seconds and will always be integers.

The format of the `router.start` message is as follows:

```json
{
  "id": "some-router-id",
  "hosts": ["1.2.3.4"],
  "minimumRegisterIntervalInSeconds": 20,
  "prunteThresholdInSeconds": 120,
}
```

After a `router.start` message is received by a client, the client should send
`router.register` messages. This ensures that the new controller can update its
routing table.

If a component comes online after the controller, it must make a NATS request
called `router.greet` in order to determine the interval. The response to this
message will be the same format as `router.start`.

The format of the `router.register` message is as follows:

```json
{
  "host": "127.0.0.1",
  "port": 4567,
  "uris": [
    "my_first_url.vcap.me",
    "my_second_url.vcap.me"
  ],
  "tags": {
    "another_key": "another_value",
    "some_key": "some_value"
  },
  "app": "some_app_guid",
  "stale_threshold_in_seconds": 120,
  "private_instance_id": "some_app_instance_id",
  "router_group_guid": "some_router_group_guid"
}
```

`stale_threshold_in_seconds` is the custom staleness threshold for the route
being registered. If this value is not sent, it will default to the controller's
default staleness threshold.

`app` is a unique identifier for an application that the endpoint is registered
for.

`private_instance_id` is a unique identifier for an instance associated with
the app identified by the `app` field.

`router_group_guid` determines which controllers will register route. Only
controllers configured with the matching router group will register the route. If
a value is not provided, the route will be registered by all controllers that
have not be configured with a router group.

Such a message can be sent to both the `router.register` subject to register
URIs, and to the `router.unregister` subject to unregister URIs, respectively.

**Note:** In order to use `nats-pub` to register a route, you must run the command on the NATS VM. If you are using [`cf-deployment`](https://github.com/cloudfoundry/cf-deployment), you can run `nats-pub` from any VM.

## Healthchecking

The controller has a health endpoint `/health` that returns a 200 OK which indicates
the controller instance is healthy; any other response indicates unhealthy.
This port can be configured via the `status.port` property in the application configuration for development purposes,
but will override to the Diego PORT value provided to the container environment.


```bash
curl -v http://10.0.32.15/health
*   Trying 10.0.32.15..
* Connected to 10.0.32.15 (10.0.32.15) port 80 (#0)
> GET /health HTTP/1.1
> Host: 10.0.32.15
> User-Agent: curl/7.43.0
> Accept: */*
>
< HTTP/1.1 200 OK
< Cache-Control: private, max-age=0
< Expires: 0
< Date: Thu, 22 Sep 2016 00:13:54 GMT
< Content-Length: 3
< Content-Type: text/plain; charset=utf-8
<
ok
* Connection #0 to host 10.0.32.15 left intact
```

## Instrumentation

### The Routing Table

The `/routes` endpoint returns the entire routing table as JSON. This endpoint
requires basic authentication. Each route has an associated array of host:port entries.

```bash
curl "http://someuser:somepass@10.0.32.15/routes"
{"api.catwoman.cf-app.com":[{"address":"10.244.0.138:9022","ttl":0,"tags":{"component":"CloudController"}}],"dora-dora.catwoman.cf-app.com":[{"address":"10.244.16.4:60035","ttl":0,"tags":{"component":"route-emitter"}},{"address":"10.244.16.4:60060","ttl":0,"tags":{"component":"route-emitter"}}]}
```

Because of the nature of the data present in `/routes`, it require http basic
authentication credentials. These credentials can be found the application
configuration:

```
status:
  password: zed292_bevesselled
  user: paronymy61-polaric
```

## Logs

The controller's logging is specified in its YAML configuration file, or application manifest. It supports the following log levels:

* `fatal` - A fatal error has occurred that makes the controller unable to execute.
* `error` - An unexpected error has occurred.
* `info`, `debug` - An expected event has occurred.

Sample log message.

`[2017-02-01 22:54:08+0000] {"log_level":0,"timestamp":1485989648.0895808,"message":"endpoint-registered","source":"vcap.cf-bigip-ctlr.registry","data":{"uri":"0-*.login.bosh-lite.com","backend":"10.123.0.134:8080","modification_tag":{"guid":"","index":0}}}
`

- `log_level`: This represents logging level of the message
- `timestamp`: Epoch time of the log
- `message`: Content of the log line
- `source`: The function within the controller that initiated the log message
- `data`: Additional information that varies based on the message
