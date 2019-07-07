---
title: "Automatic Https"
date: 2017-08-20T16:57:56+05:30
draft: flase
---

To include automatic HTTPS for any website, we need to have two things in place:

1. A trusted **Certificate Authority** that support automatic verification and provisioning of SSL certificate.
2. A **web-server/reverse-proxy** that can automatically obtain the SSL certificate from above CA(s) and configure it.

[Let's Encrypt](https://letsencrypt.org/) is the perfect solution for the first requirement. Quoting from their official site:

> Let’s Encrypt is a free, automated, and open certificate authority (CA), run for the public’s benefit.

Let’s Encrypt is a Linux Foundation project and is supported by lot of big organisation like Mozilla, Chrome, Cisco, Facebook etc. Recently Let’s Encrypt crossed the milestone of issuing 100 Million certificates. Let’s Encrypt is also working on [IETF RFC](https://github.com/ietf-wg-acme/acme/) for standardising Automatic Certificate Management Environment (ACME) protocol.

The second requirement can be fulfilled by a variety of web servers. But here we’ll be only talking about two in particular - Caddy and Traefik. We also have a dedicated [module](https://github.com/icing/mod_md) for the Apache httpd server that serves the purspose.

There are two popular ways to complete this task:

- Traditional: Specifying domains in configuration for which HTTPS certificates needs to be configured and served.
- On-Demand TLS: Instead of specifying the domains in the config, we can make use of the On-Demand feature. It allows you to obtain a certificate during the first TLS handshake for a site/hostname that doesn't have a certificate as of yet, and reuse it afterwards.

### [Caddy](https://caddyserver.com/)

From the official page of Caddy:

> Fast, cross-platform HTTP/2 web server with automatic HTTPS.

Caddy has many [built in features](https://caddyserver.com/features) like static files, dynamic sites, simple configuration, zero-downtime reloads, extensible core, automatic TLS and a lot more. Caddy even received a [grant from Mozilla](https://blog.mozilla.org/blog/2016/06/22/mozilla-awards-385000-to-open-source-projects-as-part-of-moss-mission-partners-program/) to help them continue the awesome work that they do. A [cncf](https://www.cncf.io/) project, [CoreDNS](https://coredns.io/) that started out as [Caddy middleware](https://coredns.io/2017/02/23/history-of-coredns-in-four-posts/) shows it’s powerful extensibility. And to top it all, Caddy also has very friendly and active [community](https://caddy.community/).

Follow [this excellent](https://caddyserver.com/tutorial) guide to get started. Caddy is the first web server that does On-Demand TLS.

Creating a user for Caddy, a directory for storing SSL certificate and for Caddyfile, file for logging:

```sh
sudo useradd -md /var/log/caddy -s /sbin/nologin caddy
sudo mkdir -p /etc/ssl/caddy
sudo chown -R caddy. /etc/ssl/caddy
sudo mkdir /etc/caddy
sudo chown -R caddy. /etc/ssl/caddy
sudo touch /var/log/caddy/caddy.log
sudo chown caddy. /var/log/caddy/caddy.log
```

Save this Caddyfile as `/etc/caddy/Caddyfile` file.

```
:80 :443 {

  proxy / http://backend.example.com {
      transparent
  }
  # redirect to HTTPS
  redir 301 {
    if {scheme} not https
    / https://{host}{uri}
  }
  tls on
  tls user@example.com
  tls {
      max_certs 100
  }
      log / access.log "{combined}" {
            rotate_size     50
            rotate_age      90
            rotate_keep     20
            rotate_compress
      }
      prometheus 0.0.0.0:9180
}
```

This `Caddyfile` shows a configuration for using Caddy as a reverse proxy. It uses `proxy`, `tls`, `log`, `redir` and `prometheus` middlewares. Prometheus middleware doesn’t come integrated with Caddy binary and hence needs to be specified at the time of download. It will instruct Caddy to listen in on ports 80 and 443 for any incoming connection and forward all requests transparently to specified destination (`backend.example.com`) with redirection to HTTPS first if request is not HTTPS. `tls email` is responsible for enabling Let’s Encrypt support. log middleware enables logging of each (with /) or selected requests. `prometheus` middleware exposes Caddy metrics on configured port (9180) and it can be consumed by [Prometheus Server](https://prometheus.io/) for monitoring and alerting.

Use this systemd init script to manage caddy process by saving it as `/etc/systemd/system/caddy.service` file:

```
[Unit]
Description=Caddy HTTP/2 web server
Documentation=https://caddyserver.com/docs
After=network-online.target
Wants=network-online.target

[Service]
Restart=on-failure
StartLimitInterval=86400
StartLimitBurst=5

; User and group the process will run as.
User=caddy
Group=caddy
WorkingDirectory=/var/log/caddy

; Letsencrypt-issued certificates will be written to this directory.
Environment=CADDYPATH=/etc/ssl/caddy

; Always set "-root" to something safe in case it gets forgotten in the Caddyfile.
ExecStart=/usr/local/bin/caddy -log=/var/log/caddy/caddy.log -agree=true -conf=/etc/caddy/Caddyfile -root=/var/tmp
ExecReload=/bin/kill -USR1 $MAINPID

; Limit the number of file descriptors; see `man systemd.exec` for more limit settings.
LimitNOFILE=1048576
; Unmodified caddy is not expected to use more than that.
LimitNPROC=64
ReadWriteDirectories=/etc/ssl/caddy

[Install]
WantedBy=multi-user.target
```

Start Caddy server by:

```sh
sudo systemctl daemon-reload
sudo systemctl start caddy.service
```

### [Traefik](https://traefik.io/)

From the official page of [Traefik](https://traefik.io/):

> Træfik (pronounced like traffic) is a modern HTTP reverse proxy and load balancer made to deploy microservices with ease. It supports several backends (Docker, Swarm, Kubernetes, Marathon, Mesos, Consul, Etcd, Zookeeper, BoltDB, Eureka, Amazon DynamoDB, Rest API, file…) to manage its configuration automatically and dynamically.

Traefik supports both ways of [automatic HTTPS](https://docs.traefik.io/toml/#acme-lets-encrypt-configuration), Traditional and On-Demand TLS.

Traefik has excellent [starting guide](https://docs.traefik.io/basics/), [configuration guide](https://docs.traefik.io/toml/), [examples](https://docs.traefik.io/user-guide/examples/) and key-value database integration with [trafik configuration](https://docs.traefik.io/user-guide/kv-config/) for configuration on fly.

Traefik can be broken into three major parts:

1. Entrypoint: This defines the point where Traefik will continously listen in on and decide whether to perform an SSL termination or not.
2. Frontend: It consists of a set of rules that determine how incoming requests are forwarded from an entrypoint to a backend.
3. Backend: It is responsible for load-balancing the traffic that comes in from one or more frontends to a set of http servers.

We can directly use Traefik with a file backend to serve as a single point of source for Traefik configuarion and update it using API calls. We can also use any supported key-value database to store configuration information. The latter has an important advantage of being able to update a key and instaneously be reloaded by Traefik.

For now we will be using consul as key-value store for Traefik. Consul can be [download from here](https://www.consul.io/downloads.html). We are going to run Consul on a single node and hence it's fine to just use the `bootstrap` mode of Consul. Given below is the Consul configuartion, saved as a json file `consul.json`.

```
{
  "datacenter": "aws-west",
  "enable_syslog": true,
  "data_dir": "/opt/consul",
  "log_level": "INFO",
  "node_name": "node1",
  "server": true,
  "bootstrap": true,
  "advertise_addr": "IP_of_SERVER",
  "bind_addr": "0.0.0.0",
  "client_addr": "0.0.0.0"
}
```


Create a data directory for Consul:

```sh
sudo mkdir /opt/consul
```


Use this file to create systemd init script for Consul by saving this file as `/etc/systemd/system/consul.service`.

```
[Unit]
Description=consul agent
Requires=network-online.target
After=network-online.target

[Service]
ExecStart=/PATH/OF/BINARY/consul agent -ui -config-file=/PATH/OF/CONFIGURATION/consul.json
LimitNOFILE=60000
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

Start Consul server:

```sh
sudo systemctl daemon-reload
sudo systemctl start consul
```


Now, lets get back to Traefik. Create a directory for logs of traefik:

```sh
sudo mkdir -p /var/log/traefik
```

This is the main consul configuration. It defines logging, default entrypoints, ACME (Let’s Encrypt setup for automatic HTTPS), web (an endpoint for API calls and site) and Consul setup configuration. For `ACME`, note that `OnDemand` flips the switch for On-Demand SSL and `OnHostRule` can be used for serving HTTPS for selective domains determined by a frontend rule like `rule = "Host:https-site.example.com"` for enabling HTTPS for `https-site.exapmle.com`.


```
traefikLogsFile = "/var/log/traefik/traefik.log"
accessLogsFile = "/var/log/traefik/access.log"
logLevel = "INFO"
InsecureSkipVerify = true
defaultEntryPoints = ["http", "https"]

[entryPoints]
[entryPoints.http]
address = ":80"
[entryPoints.https]
address = ":443"
[entryPoints.https.tls]

[acme]
email = "EMAIL_ID"
storage = "traefik/acme/account"
entryPoint = "https"
acmeLogging = true
onDemand = false
OnHostRule = true

[web]
address = ":8080"
ReadOnly = false

[consul]
endpoint = "127.0.0.1:8500"
watch = true
prefix = "traefik"
```

Traefik can be started using systemd init script upon saving as `/etc/systemd/system/traefik.service` file:

```
[Unit]
Description=traefik load balancer
After=network.target auditd.service

[Service]
User=root
ExecStart=/PATH/TO/BINARY/traefik --configfile=/PATH/TO/traefik.toml
Restart=on-failure
LimitNOFILE=60000

[Install]
WantedBy=default.target
```

Start traefik:

```sh
sudo systemctl daemon-reload
sudo systemctl start traefik
```

The static Traefik configuration in a key-value store can be automatically created and updated, using the storeconfig subcommand. `sudo /PATH/TO/BINARY/traefik --configfile=/PATH/TO/traefik.toml` storeconfig

The API given below calls and sends key-value to Consul for storing it. Note that, all keys start with the `traefik` keyword as we have already configured Traefik to watch for any keys that start with this keyword.

```
curl -X PUT \
  http://CONSUL_IP:8500/v1/kv/traefik/backends/backend1/servers/server1/url \
  -d backend.example.com

curl -X PUT \
  http://CONSUL_IP:8500/v1/kv/traefik/frontends/frontend1/backend \
  -d backend1

curl -X PUT \
  http://CONSUL_IP:8500/v1/kv/traefik/frontends/frontend1/entrypoints \
  -d 'http,https'

curl -X PUT \
  http://CONSUL_IP:8500/v1/kv/traefik/frontends/frontend1/routes/route1/rule \
  -d Host:app.example.com

curl -X PUT \
  http://CONSUL_IP:8500/v1/kv/traefik/frontends/frontend1/passHostHeader \
  -d true

curl -X PUT \
  http://CONSUL_IP:8500/v1/kv/traefik/frontends/frontend1/priority \
  -d 10
```

This configures traefik to route any HTTP and HTTPS request for `app.example.com` to the backend server `backend.example.com`, automatically getting the HTTPS certificate from Let’s Encrypt and storing it on the specified key on Consul due to `OnHostRule` being set to true in the `ACME` configuration. We can verify this by querying Traefik API as:

```
curl -X GET \
  http://TRAEFIK_IP:8080/api

{
    "consul": {
        "backends": {
            "backend1": {
                "servers": {
                    "server1": {
                        "url": "backend.example.com",
                        "weight": 0
                    }
                },
                "loadBalancer": {
                    "method": "wrr"
                }
            }
        },
        "frontends": {
            "frontend1": {
                "entryPoints": [
                    "http",
                    "https"
                ],
                "backend": "backend1",
                "routes": {
                    "route1": {
                        "rule": "Host:app.example.com"
                    }
                },
                "passHostHeader": true,
                "priority": 10,
                "basicAuth": null
            }
        }
    }
}
```

> Gotchas: Traefik with Consul backend doesn't work on more than certain(~100) domains TLS certificate as it saves certificates by appending it to a Consul key and Consul has limit of 512KB on a value. Also due to a [bug](https://github.com/containous/traefik/issues/926) Let's Encrypt (Automatic HTTPS) doen't work on Traefik with etcd backend.
