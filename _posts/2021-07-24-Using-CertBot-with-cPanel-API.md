---
layout: post
title: "Using Certbot with Interserver's cPanel API"
date: 2021-07-24 18:00 -0700
categories: [Tutorials, APIs]
tags: [cpanel, letsencrypt, certbot, interserver]
---

## API calls against cPanel at InterServer Webhosting

I need to remotely maintain DNS entries at my InterServer webhosting instance, and I want to use the `cPanel API v2` for that, because that's basically all InterServer, my web hosting provider, is providing access to.

The `cPanel` documentation can be found at: https://documentation.cpanel.net/display/DD/cPanel+API+2+Modules+-+ZoneEdit

### Current situation and difficulties

My main goal was to find a way of implementing this with `acme.sh` or `certbot` to enable automatic wildcard certificate renewals with [Let's Encrypt](https://letsencrypt.org/) in a reverse proxy container. The reason for this (likely overly complex) setup is that I have a Synology DiskStation behind my home router, that serves some applications available publicly (e.g. a photo gallery). This device isn't the only device that's hosting externally accessible resources, but until now that wasn't much of an issue: DSM, the OS that's running Synology devices, comes with a reverse proxy package, that can serve other applications too, additionally to those installed directly on the DiskStation.

Recently I've switched to automated installations of my `RaspberryPi`s and other stuff, including applications hosted in containers on these `RaspberryPi`s and the DiskStation. So now the main problem is, that in order to get external services and more complex setups covered by the Synology reverse proxy, it's configuration needs customization that needs to be carefully built, maintained and backed up, because it's unsupoported and gets overwritten with any update of the DSM. The whole process is hacky and highly annoying and unreliable.

### First steps toward a solution

#### Docker container

The very first thing to do was setting up a `Docker` container that takes over the role of the reverse proxy. I've used a `linuxserver/swag` container for that, because it comes with `nginx` and `certbot` already. Docker is available through a package, and besides some more configuration hacking inside the DiskStation (that's still way easier to replicate, if needed, and doesn't need maintenance) it is a breeze to set up with the help of `docker-compose`. I'll describe that setup in another post later...

#### Nginx as reverse proxy

When this container is up and running it needs to be configured to serve as reverse proxy for all desired applications. The setup of `nginx` is relatively easy to do for people not that much into such technical details, and also is pretty well documented. I may also write about my setup later, if I think it is special enough to be of value for anyone else but me.

#### Certbot for certificate auto-renewal

The next component in this setup is `certbot` which is responsible for the autmatic requests and renewals of SSL certificates. The easiest way for me to do this so far was letting the DSM do this. **BUT:** this implementation doesn't support wildcard certificates, and that meant I needed a different solution now. I no longer wanted to go through the hassle of maintaining a dozen different certificates or have multiple unrelated servernames hosted by one certificate. So for me the obvious solution for a centralized home-network reverse proxy is a wildcard certificate.

In a company's network, or in bigger installations I probably wouldn't use one wildcard certificate for everything, but - **it's my stuff, and I want one, so I'll get one!...**

#### DNS

The last piece of this puzzle is access to DNS, because the automated implementations of [Let's Encrypt](https://letsencrypt.org/) auto-renewal of wildcard certificates works only with `TXT records` in `DNS`. Bots like `acme.sh` and `certbot` come with a host of plugins for several `DNS` providers - but InterServer is not amongst them. Fortunately they provide access to the `cPanel` API, so this makes things a lot easier...right?

Well, not exactly.

### Extra hurdles put up by shared webhosting

#### No password?

The price of a webhosting package can be higher than what's on the regular invoice. In the case of InterServer it is the lack of access to `cPanel` functionality in their shared Webhosting. Specifically the missing password to my own account. The way the InterServer webhosting is logging me into `cPanel` is by creating a child session in `cPanel` from out of my current session in the parent portal. I'll then get forwarded to said child session and can administer my package from there. Except: I can not change my password, and I can not create additional `cPanel` users (only FTP, Email etc.).

#### Use an API key instead

Luckily the system does allow for creating an API key, and after studying the `cPanel` documentation I also found out how to use it.
So here are some example calls that one can use to either play around with it, put them into scripts for recurring tasks, or wrap a `Python` or `PowerShell` script around it for more complex usage.

BTW, the reason for using the outdated version 2 of the API is that there are no equivalent commands available in `cPanel`s newer `UAPI`.

### Manual API calls

I've came up with the following calls against the API to test the communication with the API, as well as the creation and removal of `TXT` (and `CNAME`) records, which is necessary to get wildcard certificates issued by [Let's Encrypt](https://letsencrypt.org/).

#### Simple domain lookup

This is an easy way to verify general functionality, authentication, access permission etc.

```bash
curl -H'Authorization: cpanel USER:API_KEY' 'https://my_domain.com:2083/execute/DNS/lookup?domain=my_domain.com'
```

#### Get entire zone file

This dumps the entire zone file as is.

```bash
curl -H'Authorization: cpanel USER:API_KEY' 'https://my_domain.com:2083/json-api/cpanel?cpanel_jsonapi_user=user&cpanel_jsonapi_apiversion=2&cpanel_jsonapi_module=ZoneEdit&cpanel_jsonapi_func=fetchzone&domain=my_domain.com'
```

#### Add CNAME record

The record name in this example is `cname-test`.

```bash
curl -H'Authorization: cpanel USER:API_KEY' 'https://my_domain.com:2083/json-api/cpanel?cpanel_jsonapi_user=user&cpanel_jsonapi_apiversion=2&cpanel_jsonapi_module=ZoneEdit&cpanel_jsonapi_func=add_zone_record&domain=my_domain.com&name=cname-test&type=CNAME&cname=www.other_domain.com'
```

#### Add TXT record

The record name in this example is `le-test`.

```bash
curl -H'Authorization: cpanel USER:API_KEY' 'https://my_domain.com:2083/json-api/cpanel?cpanel_jsonapi_user=user&cpanel_jsonapi_apiversion=2&cpanel_jsonapi_module=ZoneEdit&cpanel_jsonapi_func=add_zone_record&domain=my_domain.com&name=le-test&type=TXT&txtdata=1234567890ABCDEF'
```

#### Get line number for specific name in zone file

In order to remove an entry we need to know first where that entry is located in the file.

Note: the name in the zone file is an FQDN including a trailing dot! So here the name to look up is `le-test.my_domain.com.`.

```bash
curl -H'Authorization: cpanel USER:API_KEY' 'https://my_domain.com:2083/json-api/cpanel?cpanel_jsonapi_user=user&cpanel_jsonapi_apiversion=2&cpanel_jsonapi_module=ZoneEdit&cpanel_jsonapi_func=fetchzone_records&domain=my_domain.com&name=le-test.my_domain.com.&type=TXT'
```

#### Remove TXT record

When we know the number, the respective entry can be removed from the zone file.

```bash
curl -H'Authorization: cpanel USER:API_KEY' 'https://my_domain.com:2083/json-api/cpanel?cpanel_jsonapi_user=user&cpanel_jsonapi_apiversion=2&cpanel_jsonapi_module=ZoneEdit&cpanel_jsonapi_func=remove_zone_record&domain=my_domain.com&line=59'
```
