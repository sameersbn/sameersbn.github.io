---
layout: post
title: "Deploying a DNS Server using Docker"
date: 2015-04-28 13:16:21 +0530
comments: true
sharing: true
donating: true
published: true
categories:
  - howto
  - docker
---

***This is the first part of a series of how-to's where I describe setting up and using various docker containers for home and production use.***

To start off this series we will use the [sameersbn/bind](https://github.com/sameersbn/docker-bind) [docker](https://www.docker.com/) image to setup a DNS server in production and host only environments.

[BIND](https://www.isc.org/downloads/bind/), developed by the [Internet Systems Consortium](http://www.isc.org/), is a production-grade and by far the most popular and widely used opensource DNS server software available.

<!-- more -->

# Introduction

The [Domain Name System](http://en.wikipedia.org/wiki/Domain_Name_System) (DNS) server takes a fully qualified domain name (FQDN) such as `www.example.com` and returns the corresponding IP address such as `93.184.216.34`.

***A couple of reasons to set up a DNS server.***

By setting up your own local DNS server you don't rely on your ISP's DNS servers which are often bogged down by incoming traffic which makes responses to DNS queries take longer to get serviced.

Besides performing domain name resolutions, a DNS server also acts as a cache. This means that DNS queries could get serviced from the local cache as long as the cached DNS entry has not timed out. Overall the cache speeds up DNS responses.

Some ISP's block access to websites by DNS spoofing. Setting up your own DNS server can help you get around this. However, a more effective way to circumvent this type of censorship is by using the [tor browser](https://www.torproject.org/projects/torbrowser.html.en) which can be installed using the [sameersbn/browser-box](https://github.com/sameersbn/docker-browser-box) image.

Finally and most importantly, a local DNS server will enable you to define a domain for your local network. This will allow you to address machines/services on the network with a name rather than the IP address. When setting up web services whether you do it using docker or otherwise, installing a DNS server makes the setup much simpler and easier to deal with.

# Setting up the image

Begin by fetching the image from the docker hub.

<img src="/images/20150428130638-docker-pull-bind.gif" class="window-generic">

```bash
docker pull sameersbn/bind:latest
```

Now lets "boot" the image...

<img src="/images/20150428130816-docker-run-bind.gif" class="window-generic">

```bash
docker run -d --name=bind --dns=127.0.0.1 \
  --publish=172.17.42.1:53:53/udp --publish=172.17.42.1:10000:10000 \
  --volume=/srv/docker/bind:/data \
  --env='ROOT_PASSWORD=SecretPassword' \
  sameersbn/bind:latest
```

*Here is the gist of the equivalent configuration in [docker-compose.yml](https://gist.github.com/sameersbn/ea4692d9a8ee7accd6b3) form.*

- `-d` detaches, runs the container in the background
- `--name='bind'` assigns the name `bind` to the container
- `--dns=127.0.0.1` configures the dns of the container to `127.0.0.1`
- `--publish=172.17.42.1:53:53/udp` makes the DNS server accessible on `172.17.42.1:53`
- `--publish=172.17.42.1:10000:10000` makes webmin accessible at https://172.17.42.1:10000
- `--volume=/srv/docker/bind:/data` mounts `/srv/docker/bind` as a volume for persistence
- `--env='ROOT_PASSWORD=SecretPassword'` sets the root password to `SecretPassword`


In the above command the DNS server will only be accessible to the host and other containers over the docker bridge interface (host only). If you want the DNS server to be accessible over the network you should replace `--publish=172.17.42.1:53:53/udp` to `--publish=53:53/udp` (all interfaces) or something like `--publish=192.168.1.1:53:53/udp` (specific interface).

***From this point on `172.17.42.1` will refer to our local DNS server. Replace it with the appropriate address depending on your setup.***

The [sameersbn/bind](https://github.com/sameersbn/docker-bind) image includes webmin, a web-based interface for system administration, so that you can quickly and easily configure BIND. Webmin is launched automatically when the image is started.

If you prefer configuring BIND by hand, you can turn off webmin by setting `--env='WEBMIN_ENABLED=false'` in the docker run command. The BIND specific configuration will be available at `/srv/docker/bind/bind`. To apply your configuration send the `HUP` signal to the container using `docker kill -s HUP bind`

Lastly, when `--env='ROOT_PASSWORD=SecretPassword'` is not specified in the run command, a random password is generated and assigned for the root user which can be retrieved with `docker logs bind 2>&1 | grep '^User: ' | tail -n1`. This password is used while logging in to the webmin interface.

# Test the DNS server

Before we go any further, lets check if our local DNS server is able to resolve addresses using the unix `host` command.

<img src="/images/20150428131001-dns-server-test.gif" class="window-generic">

```bash
host www.google.com 172.17.42.1
```

- `www.google.com` the address to resolve
- `172.17.42.1` the DNS server to be used for the resolution

If everything works as expected the `host` command should execute successfully and should return the IP address of `www.google.com`.

# Using the DNS server

If you have setup a DNS server for your local network, you can configure your DHCP server to give out the address of the DNS server in the lease. If you do not have a DHCP server running on your network you will have to manually configure it in the operating systems network settings.

<img src="/images/20150428131129-dns-network-configuration.gif" class="window-generic">

Alternately, on linux distributions that use [Network Manager](https://wiki.gnome.org/Projects/NetworkManager) (virtually every linux distribution), you can add a [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html) configuration (`/etc/NetworkManager/dnsmasq.d/dnsmasq.conf`) such that the local DNS server would be used for specific addresses, while the default DNS servers would be used otherwise.

```
domain-needed
all-servers
cache-size=5000
strict-order

server=/example.com/google.com/172.17.42.1
```

In the above example, regardless of the primary DNS configuration the DNS server at `172.17.42.1` will be used to resolve `example.com` and `google.com` addresses. This is particularly useful in host only configurations when you setup a domain to address various services on the local host without having to manually change the DNS configuration everytime you connect to a different network.

After performing the `dnsmasq` configuration the network manager needs to be restarted for the changes to take effect. On Ubuntu, this is achieved using the command `restart network-manager`

Lastly, we can configure docker such that the containers are automatically configured to use our DNS server. This is done by adding `--dns 172.17.42.1` to the docker daemon command. On Ubuntu, this is done at `/etc/default/docker`. The docker daemon needs to be restarted for these changes to take effect.

# Creating a domain using webmin

Point your web browser to https://172.17.42.1:10000 and login to webmin as user `root` and password `SecretPassword`. Once logged in click on **Servers** and select **BIND DNS Server**.

<img src="/images/20150428131223-webmin-login.gif" class="window-firefox">

This is where we will perform the DNS configuration. Changes to the configuration can be applied using the **Apply Configuration** link in the top right corner of the page. We will create a domain named `example.com` for demonstration purposes.

We will start off by creating the reverse zone `172.17.42.1`. This is ***optional*** and required only if you want to be able to do reverse DNS (rDNS) lookups. A rDNS lookup returns the domain name that is associated with a given IP address. To create the zone select **Create master zone** and in the **Create new zone** dialog set the **Zone type** to **Reverse**, the **Network address** to your interface IP address `172.17.42.1`, the **Master server** to`ns.example.com` and finally set **Email address** to the domain administrator's email address and select **Create**.

<img src="/images/20150428131314-webmin-bind-reverse-zone.gif" class="window-firefox">

Next, we will the forward zone `example.com`. To create the zone select **Create master zone** and in the **Create new zone** dialog set the **Zone type** to **Forward**, the **Domain Name** to `example.com`, the **Master server** to `ns.example.com` and set **Email address** to the domain administrator's email address and select **Create**. Next, create the DNS entry for `ns.example.com` pointing to `172.17.42.1` and apply the configuration.

<img src="/images/20150428131339-webmin-bind-forward-zone.gif" class="window-firefox">

To complete this tutorial we will create a address (A) entry for `webserver.example.com` and then add a domain name alias (CNAME) entry `www.example.com` which will point to `webserver.example.com`.

To create a A entry, select the zone `example.com` and then select the **Address** option. Set the **Name** to `webserver` and the **Address** to `192.168.1.1`. To create a CNAME entry, select the zone `example.com` and then select the **Name Alias** option. Set the **Name** to `www` and the **Real Name** to `webserver` and apply the configuration.

<img src="/images/20150428131424-webmin-bind-sample-entries.gif" class="window-firefox">

And now, the moment of truth...

```bash
host webserver.example.com 172.17.42.1
host www.example.com 172.17.42.1
```

These commands should return the DNS addresses as per our configuration. Time to find out.

<img src="/images/20150428131518-dns-server-test2.gif" class="window-generic">

An there you have it. A local DNS server with a local domain named `example.com`.

