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

### Play it!!!

Now having understood the theoretical premise of what and how we are to achieve a centralized netdata, it worth taking that to a ride. The resources(self contained binaries) for the setup are availalble [here](https://github.com/samof76/writtings/tree/master/metric_hill_netdata_part1) at the `resources` directory. Now for the setup, we would require [`gorouter`](https://github.com/samof76/writtings/blob/master/metric_hill_netdata_part1/resources/gorouter?raw=true), [`gnatsd`](https://github.com/samof76/writtings/blob/master/metric_hill_netdata_part1/resources/gnatsd?raw=true) and [`nats-pub`](https://github.com/samof76/writtings/blob/master/metric_hill_netdata_part1/resources/nats-pub?raw=true), from that directory. This setup exercise is completely manual to understand the nuances. And all commands here are run on a Ubuntu 16.04 server.

#### Supervised Gorouter

Always using process manager to manage long running process is an ideal mechanism. For this setup we would be using [supervisord](http://supervisord.org), to manage both Gorouter and Gnatsd,same machine. So lets first download these resources on to the machine which would run Gorouter.

    $ # Download Gorouter
    $ sudo wget -O /usr/bin/gorouter https://github.com/samof76/writtings/blob/master/metric_hill_netdata_part1/resources/gorouter?raw=true
    $ sudo chmod +x /usr/bin/gorouter
    $ # Download Gnatsd
    $ sudo wget -O /usr/bin/gnatsd https://github.com/samof76/writtings/blob/master/metric_hill_netdata_part1/resources/gorouter?raw=true
    $ sudo chmod +x /usr/bin/gnatsd


Now we have Gorouter and Gnatsd in place on our machine, so off to setting up supervisord. Supervisord is program that manage multiple coworking or disparate long running processes, that you would like to put into the background. Supervisord also provide control center(CLI-based, `supervisorctl`), that allows you to manage the processes. Supervisord installation is simple.

    $ sudo apt-get install python-pip python-setuptools
    $ sudo pip install supervisor

To make supervisor aware of running both Gorouter and Gnatsd, we have to create `supervisord.conf` file, and place it in /etc, which one of the location where supervisord will pick up the configuration from. Our `supervisord.conf` looks like the following.

    [unix_http_server]
    file=/var/run/supervisor.sock

    [supervisord]
    logfile=/var/log/supervisord
    logfile_maxbytes=50MB
    logfile_backups=10
    loglevel=info
    pidfile=/var/run/supervisord.pid
    nodaemon=false
    minfds=1024
    minprocs=200

    [supervisorctl]
    serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket

    [rpcinterface:supervisor]
    supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

    [program:gnatsd]
    command=/usr/bin/gnatsd
    priority=400
    stdout_logfile=/var/log/gorouter/access.log
    stdout_logfile_maxbytes=10MB
    stdout_logfile_backups=10
    stderr_logfile=/var/log/gorouter/error.log
    stderr_logfile_maxbytes=10MB
    stderr_logfile_backups=10

    [program:gorouter]
    command=/usr/bin/gorouter -c /etc/gorouter.yml
    priority=500
    stdout_logfile=/var/log/gnatsd/access.log
    stdout_logfile_maxbytes=10MB
    stdout_logfile_backups=10
    stderr_logfile=/var/log/gnatsd/error.log
    stderr_logfile_maxbytes=10MB
    stderr_logfile_backups=10

Notice `/etc/gorouter.yml`, this is just a basic Gorouter configuration file, that looks like this.

    status:
        port: 8082
        user: admin
        pass: 5tr0ngp@55w0rd

    nats:
        - host: "localhost"
          port: 4222
          user:
          pass:


    port: 8081
    index: 0

    go_max_procs: 5

The Gorouter configuration, is actually not needed, yet needs to created if you have specified `-c` option on your `/etc/supervisord.conf` file. Also note the `priority` in the supervisord configuration, this actually ensures that `gorouter` is launched after `gnatsd`.

Now we can start the supervisord daemon.

    $ sudo supervisord -c /etc/supervisord.conf

This will start both `gnatsd` and `gorouter`, ensure they are start but making curl to the Gorouter's admin API.

    $ curl http://admin:5tr0ngp@55w0rd@localhost:8082/healthz

This should return __`ok`__. That means that you are all set to register your netdata URI. But one pit stop to setup Nginx proxy.

### Proxy thru Nginx

Remember our second diagram, we have to route `*.netdata.monitor.zapped.pigs` to `gorouter`. This is where that happens. Setup Nginx on your machine, I would do that on the same server thats running `gnatsd` and `gorouter`, but you are free to run it on a different server. Here is the configuration,


    upstream gorouter {
        # the netdata server
        server 127.0.0.1:8081;
        keepalive 64;
    }

    server {
        # nginx listens to this
        listen 80;

        # the virtual host name of this
        server_name *.netdata.monitor.zapped.pigs;

        location / {
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Server $host;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://gorouter;
            proxy_http_version 1.1;
            proxy_pass_request_headers on;
            proxy_set_header Connection "keep-alive";
            proxy_store off;
        }
    }

We take that and create `/etc/nginx/sites-available/star_netdata_monitor_zapped_pigs`, and then symlink to it with `/etc/nginx/sites-enabled/star_netdata_monitor_zapped_pigs`. Then we restart the `nginx` service. All we've left to do is point our domain `*.netdata.monitor.zapped.pigs` to our server on the DNS. One more thing we could do is point `admin.netdata.monitor.zapped.pigs`, to admin API of the Gorouter, that running on the server's `localhost:8082`. So we create `/etc/nginx/sites-available/admin_netdata_monitor_zapped_pigs`, and symlink to it with `/etc/nginx/sites-enabled/admin_netdata_monitor_zapped_pigs`. Here is its configuration.

    upstream gorouter-admin {
        # the netdata server
        server 127.0.0.1:8082;
        keepalive 64;
    }

    server {
        # nginx listens to this
        listen 80;

        # the virtual host name of this
        server_name admin.netdata.monitor.zapped.pigs;

        location / {
            proxy_set_header X-Forwarded-Host $host;
            proxy_set_header X-Forwarded-Server $host;
            proxy_set_header Host $host;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_pass http://gorouter-admin;
            proxy_http_version 1.1;
            proxy_pass_request_headers on;
            proxy_set_header Connection "keep-alive";
            proxy_store off;
        }
    }

Now we restart the `nginx` service again. We should able be to login from the browser into http://admin.netdata.monitor.zapped.pigs/healthz, using the `username-password` combination as provided in the `gorouter`'s `yml`.

### Time to Monitor

This is where we select a server to setup Netdata and register that server with, gorouter. 