# ansible-docker-pihole

Generalized and env configured Ansible automation for setting up PiHole DNS.

The dockerized pihole and other services are installed to the `/opt/dns` directory. Pihole listens on port 53 for "external" requests and communicates resolutions to dnscrypt (if enabled) on port 5300.

This automation offers the following functionality:

* PiHole and other DNS services are dockerized and run using docker-compose
* (Optional) DNSCrypt-Proxy v2 as a resolver to add encrypted DNS queries (Anonymous DNS, DoH, DoT, ODoH)


## Architecture

This automation has been tested on the following architectures:

* `Raspberry Pi OS - linux/arm/v7`
* `Ubuntu 20.04 - linux/amd64`
* `Raspbian OS Buster - linux/arm`


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

The first time the pi is setup, you will need to create and fill out a `.env` file similar to the provided example `.env.example`. These values are used to configure the templates, settings and services for the pi DNS. Then, run the script that wraps the Ansible playbook in a shell environment with the Env Vars from `.env`:

```
$ ./path/to/ansible-pihole/ansible-playbook.sh
```

After the Ansible playbook runs for the first time setup, it can take up to 10 minutes depending on the target host for pihole and other services to finish setting up. The web admin page and DNS services won't be available until these complete their setup process.

You can check the logs for the services using the following commands:

```
$ docker-compose -f /opt/dns/docker-compose.yml logs
$ docker-compose -f /opt/dns/docker-compose.yml logs pihole
$ docker-compose -f /opt/dns/docker-compose.yml logs dnscrypt
```

#### Anonymized DNS

If you're running this with DNSCrpyt-proxy configured for Anonymized DNS (the default configuration), then you'll need to perform a couple extra steps to get the firewall rules correct.

Anonymized DNS operates (simplified) by using encryption around the DNS queries and separating the domain name answering server from your client/dnscrypt by using a relay. Relays are (usually) listed as `sdns://...` which is a DNS stamp.

For each of the Anonymized relays you configure in your `.env` you'll need to go [here](https://dnscrypt.info/stamps/) and use the stamp to parse the IP address. Then, for each of those IPv4 addresses, add a line like this to the list of UFW rules in `roles/harden/tasks/ufw.yml`, assuming your interface is `eth0` which is the default ethernet interface for raspberry pis:

```
- name: Set UFW firewalls rules
  become: true
  ansible.builtin.shell: |
    ufw default deny incoming
    ufw default deny outgoing
    ufw allow 22/tcp
    ufw allow 80/tcp
    ufw allow 443/tcp
    ufw allow 53/tcp
    ufw allow 53/udp
    ufw allow out on eth0 to 1.2.3.4 (or w/e the relay ip address is)
```

If you need to see what interfaces you're working with (on Linux), run this command and look for an assigned IP address in the subnet of the LAN the PiHole host is on:

```
$ ip addr
```


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

2. If the logs indicate dnscrypt-proxy cannot start because its port binding is already in use, try checking for what is using that port with this command:

```
lsof -i:PORT (e.g., lsof -i:5353)
```

This happens in older versions of this project when host network_mode is used, because Debian linux distros have a daemon called "avahi" already running on UDP port 5353.

3. The PiHole and DNSCrypt containers are running and the logs indicate they're OK, but DNS requests fail. If you try to dig/nslookup on the pihoe host and get messages about the connection/request timing out or no name server being found, the problem is probably the firewall (UFW) rules. 

If you have any issues where the PiHole and DNSCrpy containers and services seem to be fine and should be working, but are not, the problem is likely either the `/etc/dhcpcd.conf` name server configuration on the pihole host or the UFW firewall rules. You can tail or cat the UFW logs at `/var/log/ufw.log` to look; if you see lines like this it's telling you the firewall is blocking outgoing traffic (possibly to a relay):

```
[UFW BLOCK] IN= OUT=eth0 SRC=192.168.1.111 DST=1.2.3.4 ... PROTO=UDP SPT=53335 DPT=443 LEN=612 
```

For reference, `SPT` is "source port" and `DPT` is "destination port" for the (blocked) outgoing DNS resolution packet attempt. This means the containers/services are working but the firewall won't let the DNS requests go, so check your relay-server configurations and make sure you have UFW rules for allowing outgoing connections to relay servers. Alternatively, you can either:

1. simply disable the UFW firewall on the PiHole host `$ sudo ufw disable`, but this is bad security, or
2. allow outgoing connections in general (strongly suggest you don't allow incoming generally though) with the command `$ sudo ufw allow incoming`

Generally, the stricter your UFW rules, the better your firewall security, but the more likely you might have an issue if IP addresses change, but security and convenience are always a trade off.


## Thanks and Contributions

Thanks to Matthew Booe for [his docker + traefik blog](https://codecaptured.com/blog/self-hosting-pi-hole-with-docker-and-traefik/) which helped me configure the Traefik reverse proxy to work with pihole.


## TODO
* Add support/roles/tasks for installing on other Linux distributions; especially redhat which is the bare metal foundation for Ubiquiti's UDM Pro routers
* Add (option for) DNSCrypt-Proxy to work as PiHole resolver
* Add reverse proxy (nginx or traefik)
* Add and configure `fail2ban` for at least SSH and possibly web routes such as pihole admin
* Restrict UFW rules - 80 and 443 should by default be only accessible from the same LAN/subnet as the pihole host (unless configuring something like a road warrior incoming connection)
* Can the SDNS/IP address lookup be automated? (ttps://dnscrypt.info/stamps)
* Add support for configuring local LAN hosts (homekit, home-automation, etc)
* Improve configurability of dnscrypt and other service settings; include being able to reconfigure and restart/redeploy the dns service(s); include configuring dnscrypt-proxy modes such as anonymized DNS, DoH, DoT, ODoH, etc; include optional configurations such as a (tor) proxy for dnscrypt-proxy; also revamp README with tables for env vars and configurations
* Setup docker secret file for webpassword for pihole web admin
* Finish implementing optionally enabling `apt-transport-tor` and/or `apt-transport-https`
* Create a `reset` role for resetting the state of the raspberry pi to a default state
* Create a `generate_config` role for processing and using env vars to create the files needed to run the dockerized services without using Ansible for users that just want the configurable docker stuff alone
* Research using [arm-runner-action in GHA](https://github.com/pguyot/arm-runner-action) to create CI pipeline testing this setup on virtualized Raspberry Pi instances
* Handling systemd-resolved: can conflicts on port 53, is used by system itself, disabling can break DNS for the box; is it really best practice to disable it? Editing `/etc/systemd/resolved.conf` does not seem to work disabling stub resolver and setting DNS to 127.0.0.1