# ansible-docker-pihole

Generalized and env configured Ansible automation for setting up PiHole DNS.

The dockerized pihole and other services are installed to the `/opt/dns` directory.

NOTE: Raspbian/Raspberry Pi OS is a debian-based Linux distro, which by default use systemd and resolvconf to manage domain name resolution. The systemd stub resolver is disabled and resolvconf is configured to use the PiHole as the resolver to force the dns service created to be used as the only domain name resolver.


## Functionality

* PiHole and other DNS services are dockerized and run using docker-compose
* (Optional) DNSCrypt-Proxy v2 as a resolver to add encrypted DNS queries (Anonymous DNS, DoH, DoT, ODoH)
* (Optional) Configure `apt-transport-tor` to add authenticated and encrypted channel to improve security of updates (ideal would be tor+https)


## Requirements

A Raspberry PI (or debian-based distro server box) to be the target server for installation.

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
reset web admin password: $ docker exec -it pihole -a -p
```


## TODO
* Differentiate and allow either "bare metal" or dockerized setups for piholes
* Add (option for) DNSCrypt-Proxy to work as PiHole resolver
* Add reverse proxy (nginx or traefik)
* Improve configurability of dnscrypt and other service settings; include being able to reconfigure and restart/redeploy the dns service(s)
* Setup docker secret file for webpassword for pihole web admin
* Create a `reset` role for resetting the state of the raspberry pi to a default state
* Research using [arm-runner-action in GHA](https://github.com/pguyot/arm-runner-action) to create CI pipeline testing this setup on virtualized Raspberry Pi instances