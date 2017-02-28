# MetricHill: Centralizing Netdata

### Netdata: The New Kid!

Netdata is great tool to see the realtime metrics of your system or systems. There are some great benefits of using netdata like...

* It's extremely no nonsense setup procedure
* It's out-of-the-box monitoring presets
* It's ability to stay low on resources
* It's great documentation wiki
* Extensible with plugins
* It's active response community

... but with a pinch of salt...

* There is no centralized look at all the Netdata nodes
* [Brendan Gregg](http://www.brendangregg.com/) takes dig at it on [Hacker News](https://news.ycombinator.com/item?id=11388196)
* It misses overlays, making difficult run live comparisons

### Setting the perspective

I am most interest in getting the first one of this fixed at the moment.

When I say "No centralized look at the Netdata nodes", I mean that netdata does not offer out-of-box mechanism to discover which nodes in your infrastructure have netdata installed and how they could individually addressable. There is of course the [Netdata registry](), but with serious drawbacks when nodes are across mulitple subnets.

So let me set the perspective of this writing more clearly. Say I have node `w1`, i want it to be addressable using some internal domain say `http://w1.netdata.monitor.zapped.pigs`, in other words I would want the `w1` to publish itself to central system so its netdata dashboard becomes accessible. Also it would be better for me to have central dashboard provide me with the list all registered nodes and their netdata accessible urls.

### Gorouter It

Though netdata can be setup behind a proxy like, [Nginx](https://github.com/firehol/netdata/wiki/Running-behind-nginx), [Apache2](https://github.com/firehol/netdata/wiki/Running-behind-apache) or [Caddy](https://github.com/firehol/netdata/wiki/Running-behind-caddy), but these are not robust, for dynamic registration in an environment where there are microservices. My work with Cloudfoundry led me look at [Gorouter](https://github.com/cloudfoundry/gorouter), which a pretty effective http(tcp) traffic router, that would allow you to add service endpoint at will.

A jist of how Gorouter works. Gorouter works along with [GNatsd](https://nats.io), subscribed to messages on the Nats for nodes publishing their signature(in this case `w1.netdata.monitor.zapped.pigs`, where `w1` is a hostname) and their ip address, and port(netdata port). As you can see in the following diagram.

![gorouter_nats_nodes](https://github.com/samof76/writtings/blob/master/metric_hill_netdata_part1/resources/gorouter_nats_nodes.png)

Gorouter sits behind a proxy like Nginx which serves say, *.netdata.monitor.zapped.pigs data, and any `w1.netdata.monitor.zapped.pigs` is sent to it(Gorouter) by the Nginx proxy. Once the Gorouter picks this message it updates its registry with the new node details its URI(`w1.netdata.monitor.zapped.pigs`). So now when Gorouter receives the `w1.netdata.monitor.zapped.pigs` through the Nginx proxy, it just routes the traffic to `w1` node's netdata port, which it picks from its registry. As you can see in the following diagram.

![nginx_gorouter_nodes](https://github.com/samof76/writtings/blob/master/metric_hill_netdata_part1/resources/nginx_gorouter_nodes.png)

### Advantage Gorouter

__The design__ of Gorouter makes it most suited for Dynamic environments where service nodes come up and go. So every node requires to ping its status to the Gorouter publishing its availability, otherwise the Gorouter will remove the node details from its registry, after a defined TTL(usually 2 minutes).

__The Admin API__ of Gorouter(usually on 8081), which is protected by some __basic authentication__, allows users to check the health of the Gorouter, get runtime variables and also the routes configured. The `/routes` is very helpful to see what netdata nodes are registered and URIs they are reachable at.

__Scalablility__ is one of the features of the Gorouter, as you could run multiple Gorouter instances behind the an Nginx proxy, allowing it load balance between the configured Gorouters. This capability is enabled by running _Gnatsd_ on a independepent instance, providing each Gorouter to pick the routes from _Gnatsd_ queue. Though this article will not detail the scalable setup, it is pretty obvious.

### Setup

