# Proxy

In cf-mysql-release, [Switchboard](https://github.com/cloudfoundry-incubator/switchboard) is used to proxy TCP connections to healthy MariaDB nodes.

A proxy is used to gracefully handle failure of MariaDB nodes. Use of a proxy permits very fast, unambiguous failover to other nodes within the cluster in the event of a node failure.

When a node becomes unhealthy, the proxy re-routes all subsequent connections to a healthy node. All existing connections to the unhealthy node are closed.

## Consistent Routing

At any given time, Switchboard will only route to one active node. That node will continue to be the only active node until it becomes unhealthy.

If multiple Switchboard proxies are used in parallel (ex: behind a load-balancer) there is no guarantee that the proxies will choose the same active node. This can result in deadlocks, wherein attempts to update the same row by multiple clients will result one commit succeeding and the other fails. This is a known issue, with exploration of mitigation options on the roadmap for this product. To avoid this problem, use a single proxy instance or an external failover system to direct traffic to one proxy instance at a time.

## Node Health

### Healthy

The proxy queries an HTTP healthcheck process, co-located on the database node, when determining where to route traffic. 

If the healthcheck process returns HTTP status code of 200, the node is added to the pool of healthy nodes. 

A resurrected node will not immediately receive connections. The proxy will continue to route all connections, new or existing, to the currently active node. In the case of failover, all healthy nodes will be considered as candidates for new connections. 

### Unhealthy

If the healthcheck returns HTTP status code 503, the node is considered unhealthy. 

This happens when a node becomes non-primary, as specified by the [cluster-behavior docs](cluster-behavior.md).

The proxy will sever all existing connections to newly unhealthy nodes. Clients are expected to handle reconnecting on connection failure. The proxy will route new connections to a healthy node, assuming such a node exists.

### Unresponsive

If node health cannot be determined due to an unreachable or unresponsive healthcheck endpoint, the proxy will consider the node unhealthy. This may happen if there is a network partition or if the VM containing the healthcheck and MariaDB node died.


## State Snapshot Transfer (SST)

When a new node is added to the cluster or rejoins the cluster, it synchronizes state with the primary component via a process called SST. A single node from the primary component is chosen to act as a state donor. By default Galera uses rsync to perform SST, which blocks for the duration of the transfer. However, cf-mysql-release is configured to use [Xtrabackup](http://www.percona.com/doc/percona-xtrabackup), which allows the donor node to continue to accept reads and writes.

## Proxy count

If the operator sets the total number of proxies to 0 hosts in their manifest, then applications will start routing connections directly to one healthy MariaDB node making that node a single point of failure for the cluster.

The recommended number of proxies are 2; this provides redundancy should one of the proxies fail.

## Removing the proxy as a SPOF

The proxy tier is responsible for routing connections from applications to healthy MariaDB cluster nodes, even in the event of node failure.

Bound applications are provided with a hostname or IP address to reach a database managed by the service. By default, the MySQL service will provide bound applications with the IP of the first instance in the proxy tier. Even if additional proxy instances are deployed, client connections will not be routed through them. This means the first proxy instance is a single point of failure.

**In order to eliminate the first proxy instance as a single point of failure, operators have two options:**
  * Configure [Consul](http://consul.io)<sup>[[1]](#configuring-consul)</sup> for service discovery.
  * Configure a load balancer<sup>[[2]](#configuring-load-balancer)</sup> to route client connections to all proxy IPs, and configure the MySQL service<sup>[[3]](#configuring-cf-mysql-release-to-give-applications-the-address-of-the-load-balancer)</sup> to give bound applications a hostname or IP address that resolves to the load balancer.

*Note: To use MySQL as a bindable service on Cloud Foundry the operator must configure a load balancer, Consul will not be acceessible from applications.*

### Configuring load balancer

Configure the load balancer to route traffic for TCP port 3306 to the IPs of all proxy instances on TCP port 3306.
Next, configure the load balancer's healthcheck to use the proxy health port.
This is TCP port 1936 by default to maintain backwards compatibility with previous releases, but this port can be configured by changing the following manifest property in the `property-overrides` stub:

```
property_overrides:
  proxy:
    health_port: <port>
```

### Configuring cf-mysql-release to give applications the address of the load balancer
To ensure that bound applications will use the load balancer to reach bound databases, the set `property_overrides.host` in the `property-overrides` stub:

```
property_overrides:
  host: <load balancer address>
```

If deploying on AWS, also add the ELB name (not the address) in the `iaas-settings` stub:

```
properties:
  template_only:
    aws:
      mysql_elb_names: [<load balancer name>]
```

### AWS Route 53

To set up a Round Robin DNS across multiple proxy IPs using AWS Route 53,
follow the following instructions:

1. Log in to AWS.
2. Click Route 53.
3. Click Hosted Zones.
4. Select the hosted zone that contains the domain name to apply round robin routing to.
5. Click 'Go to Record Sets'.
6. Select the record set containing the desired domain name.
7. In the value input, enter the IP addresses of each proxy VM, separated by a newline.

Finally, update the manifest property `properties.mysql_node.host` for the cf-mysql-broker job, as described above.

### Configuring Consul

The Consul server is deployed with [cf-release](https://github.com/cloudfoundry/cf-release), if your deployment does not include cf-release you can use [consul-release](https://github.com/cloudfoundry-incubator/consul-release) to deploy a standalone Consul cluster.

To configure service discovery for the proxies, you need to colocate the consul agent with each Proxy and specify some Consul-specific properties.

#### Colocate the Consul-Agent Job

To colocate the agent with the proxy, add a `job-overrides.yml` stub to your manifest generation command. An example can be found in `manifest-generation/examples/job-overrides-consul.yml`:

```
# job-overrides.yml

job_overrides:
  colocated_jobs:
    proxy_z1:
      additional_templates:
        - {release: cf, name: consul_agent}
    proxy_z2:
      additional_templates:
        - {release: cf, name: consul_agent}
  additional_releases:
  - name: cf
    version: (( release_versions.cf.version || "latest" ))
```

For a full bosh in general infrastructure:

```
./scripts/generate-deployment-manifest \
    -c <cf-manifest> \
    -p <property-overrides-stub> \
    -i <iaas-settings-stub> \
    -j <job-overrides-stub>
```

For BOSH-lite:

```
./scripts/generate-bosh-lite-manifest \
    -j <job-overrides-stub>
```

#### Specify Consul-specific Properties

To enable consul, provide the following properties in your cf-mysql-release manifest (for example, using the provided `property-overrides.yml` stub):

```
# manifest-generation/examples/property-overrides.yml

property_overrides:
  proxy:
    # ...
    consul_enabled: true
    consul_service_name: <your-service-name>
```

## API

The proxy hosts a JSON API at `proxy-<bosh job index>-p-mysql.<system domain>/v0/`.

The API provides the following route:

Request:
*  Method: GET
*  Path: `/v0/backends`
*  Params: ~
*  Headers: Basic Auth

Response:

```
[
  {
    "name": "mysql-0",
    "ip": "1.2.3.4",
    "healthy": true,
    "active": true,
    "currentSessionCount": 2
  },
  {
    "name": "mysql-1",
    "ip": "5.6.7.8",
    "healthy": false,
    "active": false,
    "currentSessionCount": 0
  },
  {
    "name": "mysql-2",
    "ip": "9.9.9.9",
    "healthy": true,
    "active": false,
    "currentSessionCount": 0
  }
]
```

## Dashboard

The proxy also provides a Dashboard UI to view the current status of the database nodes. This is hosted at `proxy-<bosh job index>-p-mysql.<system domain>`.
