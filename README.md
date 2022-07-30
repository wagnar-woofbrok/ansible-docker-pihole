# ansible-docker-pihole

Generalized and env configured Ansible automation for setting up PiHole DNS.

The dockerized pihole and other services are installed to the `/opt/dns` directory. Pihole listens on port 53 for "external" requests and communicates resolutions to dnscrypt (if enabled) on port 5353.


## Architecture

This automation has been tested on the following architectures:

* `linux/arm/v7`
* `linux/amd64`


## Functionality

* PiHole and other DNS services are dockerized and run using docker-compose
* (Optional) DNSCrypt-Proxy v2 as a resolver to add encrypted DNS queries (Anonymous DNS, DoH, DoT, ODoH)
* (Optional) Configure `apt-transport-tor` to add authenticated and encrypted channel to improve security of updates (ideal would be tor+https)


## Requirements

A Linux machine to be the target server for installing PiHole and other services to be a DNS server.

Tested on:
* Raspbian (Debian) bullseye, armv7l
* Ubuntu 20.04, x86_64

Ansible also requires key authentication, so if that is not setup yet you will need to do that first. This can be done using `ssh-copy-id` working over password based SSH until the key is added to `authorized_keys` on the PiHole:

```
ssh-copy-id -i /path/to/ssh/key/for/pihole pi@PIHOLE_IPv4_ADDRESS
```

If you used the [Raspberry Pi Imager](https://www.raspberrypi.com/software/), you have the option of configuring either password or key based SSH authentication.


## Usage
### First-time setup

The first time the pi is setup, you will need to create and fill out a `.env` file similar to the provided example `.env.example`. These values are used to configure the templates, settings and services for the pi DNS.

### Reset the admin web password for PiHole

First, connect to the pi over SSH. Then run this command:
```
reset web admin password: $ docker exec -it pihole pihole -a -p
```


## Known Issues

1. Occasionally the bootstrap apt update files with the following error message:

```
fatal: [server]: FAILED! => {"changed": false, "msg": "Failed to update apt cache: unknown reason"}
```

Unless this is the first time setup run, this is not a problem and can be ignored.


## Thanks and Contributions

Thanks to Matthew Booe for [his docker + traefik blog](https://codecaptured.com/blog/self-hosting-pi-hole-with-docker-and-traefik/) which helped me configure the Traefik reverse proxy to work with pihole.


## TODO
* Add support/roles/tasks for installing on other Linux distributions; especially redhat which is the bare metal foundation for Ubiquiti's UDM Pro routers
* Add (option for) DNSCrypt-Proxy to work as PiHole resolver
* Add reverse proxy (nginx or traefik)
* Add support for configuring local LAN hosts (homekit, home-automation, etc)
* Improve configurability of dnscrypt and other service settings; include being able to reconfigure and restart/redeploy the dns service(s); include configuring dnscrypt-proxy modes such as anonymized DNS, DoH, DoT, ODoH, etc; include optional configurations such as a (tor) proxy for dnscrypt-proxy; also revamp README with tables for env vars and configurations
* Setup docker secret file for webpassword for pihole web admin
* Finish implementing optionally enabling `apt-transport-tor` and/or `apt-transport-https`
* Create a `reset` role for resetting the state of the raspberry pi to a default state
* Create a `generate_config` role for processing and using env vars to create the files needed to run the dockerized services without using Ansible for users that just want the configurable docker stuff alone
* Research using [arm-runner-action in GHA](https://github.com/pguyot/arm-runner-action) to create CI pipeline testing this setup on virtualized Raspberry Pi instances
* Handling systemd-resolved: can conflicts on port 53, is used by system itself, disabling can break DNS for the box; is it really best practice to disable it? Editing `/etc/systemd/resolved.conf` does not seem to work disabling stub resolver and setting DNS to 127.0.0.1