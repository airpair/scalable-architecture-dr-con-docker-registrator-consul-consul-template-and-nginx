Docker is great fun when you start building things by plugging useful containers together. Recently I have been playing with [Consul](https://www.consul.io/) and trying to plug things together to make a truly horizontally scalable web application architecture. Consul is a **Service Discovery and Configuration** application, made by [HashiCorp](https://hashicorp.com/) the people who brought us [Vagrant](http://www.maori.geek.nz/post/vagrant_with_docker_how_to_set_up_postgres_elasticsearch_and_redis_on_mac_os_x).

Previously I experimented using Consul by using SRV records ([described here](http://www.maori.geek.nz/post/docker_web_services_with_consul)) to create a scalable architecture, but I found this approach a little complicated, and I am all about simple. Then I found [Consul Template](https://hashicorp.com/blog/introducing-consul-template.html) which links to Consul to update configurations and restart application when services come up or go down. 

In this post I will describe how to use Docker to plug together Consul, Consul Template, Registrator and Nginx into a truly scalable architecture that I am calling **DR CoN**. Once all plugged together, DR CoN lets you add and remove services from the architecture without having to rewrite **any** configuration or restart **any** services, and everything just works!

<!-- FOLD -->

# Docker

Docker is an API wrapper around LXC (Linux containers) so will only run on Linux. Since I am on OSX (as many of you probably are) I have written a post about [how to get Docker running in OSX using boot2docker](http://www.maori.geek.nz/post/boot_2_docker_how_to_set_up_postgres_elasticsearch_and_redis_on_mac_os_x). This is briefly described below:

```
brew install boot2docker
boot2docker init  
boot2docker up
```

This will start a virtual machine running a Docker daemon inside an Ubuntu machine. To attach to the daemon you can run:

```
export DOCKER_IP=`boot2docker ip`  
export DOCKER_HOST=`boot2docker socket` 
```

You can test Docker is correctly installed using:

```
docker ps
```

# Build a very simple Web Service with Docker

To test the Dr CoN architecture we will need a service. For this, let create the simplest service that I know how (further described [here](http://www.maori.geek.nz/post/the_smallest_docker_web_service_that_could)). Create a file called `Dockerfile` with the contents:

```
FROM  python:3  
EXPOSE  80  
CMD ["python", "-m", "http.server"] 
```

In the same directory as this file execute:

```
docker build -t python/server .
```

This will build the docker container and call it `python/server`, which can be run with:

```
docker run -it \
-p 8000:80 python/server
```

To test that it is running we can call the service with `curl`:

```
curl $DOCKER_IP:8000
```

# Consul

Consul is best described as a service that has a DNS and a HTTP API. It also has many other features like health checking services, clustering across multiple machines and acting as a key-value store. To run Consul in a Docker container execute:


```
docker run -it -h node \
 -p 8500:8500 \
 -p 8600:53/udp \
 progrium/consul \
 -server \
 -bootstrap \
 -advertise $DOCKER_IP \
 -log-level debug
```

If you browse to `$DOCKER_IP:8500` there is a dashboard to see the services that are registered in Consul.

To register a service in Consul's web API we can use `curl`:

```
curl -XPUT \
$DOCKER_IP:8500/v1/agent/service/register \
-d '{
 "ID": "simple_instance_1",
 "Name":"simple",
 "Port": 8000, 
 "tags": ["tag"]
}'
```

Then we can query Consuls DNS API for the service using `dig`:

```
dig @$DOCKER_IP -p 8600 simple.service.consul
```

```
; <<>> DiG 9.8.3-P1 <<>> simple.service.consul
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 39614
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 0

;; QUESTION SECTION:
;simple.service.consul.		IN	A

;; ANSWER SECTION:
simple.service.consul.	0	IN	A	192.168.59.103

;; Query time: 1 msec
;; SERVER: 192.168.59.103#53(192.168.59.103)
;; WHEN: Mon Jan 12 15:35:01 2015
;; MSG SIZE  rcvd: 76
```

Hold on, **there is a problem**, *where is the port of the service?* Unfortunately DNS A records do not return the port of a service, to get that we must check SRV records:

```
dig @$DOCKER_IP -p 8600 SRV simple.service.consul
``` 

```
; <<>> DiG 9.8.3-P1 <<>> SRV simple.service.consul
;; global options: +cmd
;; Got answer:
;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 3613
;; flags: qr aa rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

;; QUESTION SECTION:
;simple.service.consul.		IN	SRV

;; ANSWER SECTION:
simple.service.consul.	0	IN	SRV	1 1 8000 node.node.dc1.consul.

;; ADDITIONAL SECTION:
node.node.dc1.consul.	0	IN	A	192.168.59.103

;; Query time: 1 msec
;; SERVER: 192.168.59.103#53(192.168.59.103)
;; WHEN: Mon Jan 12 15:36:54 2015
;; MSG SIZE  rcvd: 136
```

SRV records are difficult to use because they are not supported by many technologies.

The container [srv-router](https://github.com/vlipco/srv-router) can be 
used with Consul and nginx to route incoming calls to the correct services, as [described here](http://www.maori.geek.nz/post/docker_web_services_with_consul). However there is an easier way than that to use nginx to route to services.

# Registrator

Registrator takes environment variables defined when a Docker container is started and automatically registers it with Consul. For example:

```
docker run -it \
-v /var/run/docker.sock:/tmp/docker.sock \
-h $DOCKER_IP progrium/registrator \
consul://$DOCKER_IP:8500
```

Starting a service with:
 
```
docker run -it \
-e "SERVICE_NAME=simple" \
-p 8000:80 python/server
```

Will automatically add the service to Consul, and stopping it will remove it. This is the first part to plugin to DR CoN as it will mean no more having to manually register services with Consul.

#Consul Template

[Consul Template](https://hashicorp.com/blog/introducing-consul-template.html) uses Consul to update files and execute commands when it detects the services in Consul have changed. 

For example, it can rewrite an nginx.conf file to include all the routing information of the services then reload the nginx configuration to load-balance many similar services or provide a single end-point to multiple services.

**I modified the Docker container from https://github.com/bellycard/docker-loadbalancer for this example**

```
FROM nginx:1.7

#Install Curl
RUN apt-get update -qq && apt-get -y install curl

#Download and Install Consul Template
ENV CT_URL http://bit.ly/15uhv24
RUN curl -L $CT_URL | \
tar -C /usr/local/bin --strip-components 1 -zxf -

#Setup Consul Template Files
RUN mkdir /etc/consul-templates
ENV CT_FILE /etc/consul-templates/nginx.conf

#Setup Nginx File
ENV NX_FILE /etc/nginx/conf.d/app.conf

#Default Variables
ENV CONSUL consul:8500
ENV SERVICE consul-8500

# Command will
# 1. Write Consul Template File
# 2. Start Nginx
# 3. Start Consul Template

CMD echo "upstream app {                 \n\
  least_conn;                            \n\
  {{range service \"$SERVICE\"}}         \n\
  server  {{.Address}}:{{.Port}};        \n\
  {{else}}server 127.0.0.1:65535;{{end}} \n\
}                                        \n\
server {                                 \n\
  listen 80 default_server;              \n\
  location / {                           \n\
    proxy_pass http://app;               \n\
  }                                      \n\
}" > $CT_FILE; \
/usr/sbin/nginx -c /etc/nginx/nginx.conf \
& CONSUL_TEMPLATE_LOG=debug consul-template \
  -consul=$CONSUL \
  -template "$CT_FILE:$NX_FILE:/usr/sbin/nginx -s reload";
```
**The repository for this file is [here](https://github.com/grahamjenson/DR-CoN).**

*NOTE: the `\n\` adds a new line and escapes the newline for Docker multiline command*

This Docker container will run both Consul Template and nginx, and when the services change it will rewrite the nginx `app.conf` file, then reload nginx. 

This container can be built with:

```
docker build -t drcon .
```

and run with:

```
docker run -it \
-e "CONSUL=$DOCKER_IP:8500" \
-e "SERVICE=simple" \
-p 80:80 drcon
```

`SERVICE` is query used to select which services to include from Consul. So this DR CoN container will now load balance across all services names `simple`.

#All Together

Lets now plug everything together!

Run Consul

```
docker run -it -h node \
 -p 8500:8500 \
 -p 53:53/udp \
 progrium/consul \
 -server \
 -bootstrap \
 -advertise $DOCKER_IP
```

Run Registrator

```
docker run -it \
-v /var/run/docker.sock:/tmp/docker.sock \
-h $DOCKER_IP progrium/registrator \
consul://$DOCKER_IP:8500
```

Run DR CoN

```
docker run -it \
-e "CONSUL=$DOCKER_IP:8500" \
-e "SERVICE=simple" \
-p 80:80 drcon
```

Running `curl $DOCKER_IP:80` will return:

```
curl: (52) Empty reply from server
```

Now start a service named `simple`

```
docker run -it \
-e "SERVICE_NAME=simple" \
-p 8000:80 python/server
```

This will cause:

1. Registrator to register the service with Consul
2. Consul Template to rewrite the nginx.conf then reload the configuration

Now `curl $DOCKER_IP:80` will be routed successfully to the service.

If we then start another simple service on a different port with:

```
docker run -it \
-e "SERVICE_NAME=simple" \
-p 8001:80 python/server
```

Requests will now be load balances across the two services.

A fun thing to do is to run `while true; do curl $DOCKER_IP:80; sleep 1; done` while killing and starting simple services and see that this all happens so fast no requests get dropped.

# Conclusion

Architectures like DR CoN are much easier to describe, distribute and implement using Docker and are impossible without good tools like Consul. Plugging things together and playing with Docker's ever more powerful tools fun and useful. Now I can create a horizontally scalable architecture and have everything just work.

# Further Reading

[The Docker Book](http://www.amazon.com/gp/product/B00LRROTI4/ref=as_li_qf_sp_asin_il_tl?ie=UTF8&camp=1789&creative=9325&creativeASIN=B00LRROTI4&linkCode=as2&tag=maor01-20&linkId=2QKZTS7EW7H2VZRM)

[Original Post](http://www.maori.geek.nz/post/scalable_architecture_dr_con_docker_registrator_consul_nginx)