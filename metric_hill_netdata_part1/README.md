# Metric Hills: Netdata

Some tools are too attractive to be just shunned away as mere eye-candy. The attractiveness of Netdata stems not just from its beauty, but also for the following reasons,

* It's extremely no nonsense setup procedure
* It's out-of-the-box monitoring presets
* It's ability to stay low on resources
* It's great documentation wiki
* Extensible with plugins
* It's active response community

Having said that is it not definitely a "Goldy Locks" of Monitoring, it does have some pitfalls which obviously hinders its popularity and adoption.

* There is no centralized look at all the Netdata nodes
* [Brendan Gregg](http://www.brendangregg.com/) takes dig at it on [Hacker News](https://news.ycombinator.com/item?id=11388196)
* It misses overlays, making difficult run live comparisons

But somehow personally, I still feel that netdata offers much of promise for it to be ignored. My primary objectives are...

[] To make netdata nodes available behind a proxy
[] To make netdata nodes publish themselves and be discoverable

Netdata already has some documentation on how it could run behind [Nginx](https://github.com/firehol/netdata/wiki/Running-behind-nginx), [Apache2](https://github.com/firehol/netdata/wiki/Running-behind-apache) or [Caddy](https://github.com/firehol/netdata/wiki/Running-behind-caddy). The problem with each of these is that you will have to reconfigure them every time a new nodes comes into play, and restart the services. Moreover Nginx and Apache were never meant for Microservices environments.

So the need for is have a dynamic http proxy or router, that would help the my netdata nodes register with it dynamically and this router will my nodes be addressable. Since my work with Cloudfoundry I have known the Goruter to be quite desirable to situations where dynamic routing of traffic based on hostname is done.

[Gorouter](https://github.com/cloudfoundry/gorouter) works along with [Gnatsd](https://nats.io), where Gorouter subscribes to Gnatsd for receiving messages(in JSON) from the service nodes, and services publish messages(in JSON) to Gnatsd for Gorouter to pick to those messages. The message payload, from the service, to the Gorouter includes, `host` and `port` where the service is running, and `uris` by which the service addressable(or accessible) through the Gorouter. 

[Consul](https://consul.io) or [Etcd](https://coreos.com/etcd/) could be used for service discovery, but may be not this time around, as I would like to keep the process and procedure simple for now(or rather call it my way). But gorouter servers two purposes, one, it provides a registry where all the nodes could register their routes, and two, it works as a HTTP router to router traffic to appropriate nodes.

So for my setup, I have chosen the following the stack elements.

* Gorouter: Dynamic routing framework specifically developed for Microservices environments.
* GNatsd: A Messaging framework which receives updates from the nodes, and where Gorouter gets data from.
* Nginx(Optional): SSL Termination, Load balance between Gorouters(in case of many)

## The Setup

For this blog's sake we will keep the setup fairly simple. This includes two nodes,

1. One node for NGinx webserver
2. One node for Gorouter and GNatsd

### Gorouter and GNatsd node

For this setup we use an Ubuntu 16.04 server on AWS. Once the server is launched we move the precompiled Gorouter and GNatsd, binaries to the server. And copy them over to /usr/local/bin


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

I am most interest in getting the first one of this fixed at the moment.

When I say "No centralized look at the Netdata nodes", I mean that netdata does not offer out-of-box mechanism to discover which nodes in your infrastructure have netdata installed and how they could individually addressable. There is of course the [Netdata registry](), but with serious drawbacks when nodes are across mulitple subnets.

So let me set the perspective of this writing more clearly. Say I have node `w1`, i want it to be addressable using some internal domain say `http://w1.netdata.monitor.zapped.pigs`, in other words I would want the `w1` to publish itself to central system so its netdata dashboard becomes accessible. Also it would be better for me to have central dashboard provide me with the list all registered nodes and their netdata accessible urls.

Though netdata can be setup behind a proxy like, [Nginx](https://github.com/firehol/netdata/wiki/Running-behind-nginx), [Apache2](https://github.com/firehol/netdata/wiki/Running-behind-apache) or [Caddy](https://github.com/firehol/netdata/wiki/Running-behind-caddy), but these are not robust, for dynamic registration in an environment where there are microservices. My work with Cloudfoundry led me look at [Gorouter](https://github.com/cloudfoundry/gorouter), which a pretty effective http(tcp) traffic router, that would allow you to add service endpoint at will.

A jist of how Gorouter works. Gorouter works along with [GNatsd](https://nats.io), subscribed to messages on the Nats for nodes publishing their signature(in this case `w1.netdata.monitor.zapped.pigs`, where `w1` is a hostname) and their ip address, and port(netdata port). As you can see in the following diagram.

![gorouter_nats_nodes](https://github.com/samof76/writtings/blob/master/metric_hill_netdata_part1/resources/gorouter_nats_nodes.png)

Gorouter sits behind a proxy like Nginx which serves say, *.netdata.monitor.zapped.pigs data, and any `w1.netdata.monitor.zapped.pigs` is sent to it(Gorouter) by the Nginx proxy. Once the Gorouter picks this message it updates its registry with the new node details its URI(`w1.netdata.monitor.zapped.pigs`). So now when Gorouter receives the `w1.netdata.monitor.zapped.pigs` through the Nginx proxy, it just routes the traffic to `w1` node's netdata port, which it picks from its registry. As you can see in the following diagram.

![nginx_gorouter_nodes](https://github.com/samof76/writtings/blob/master/metric_hill_netdata_part1/resources/nginx_gorouter_nodes.png)

The design of Gorouter makes it most suited for Dynamic environments where service nodes come up and go. So every node requires to ping its status to the Gorouter publishing its availability, otherwise the Gorouter will remove the node details from its registry, after a defined TTL(usually 2 minutes).
 

