+++
title = "Setup DNSCrypt with NetworkManager"
date = "2021-04-20"
type = ["posts","post"]
[ author ]
  name = "Xavier Valcarce"
+++

To avoid your ISP eavesdropping on website you visit and other url you try to resolv, using privacy friendly DNS server is a necessity.  
`dnscrypt-proxy` supports the encryption of your DNS request over HTTPS, and makes use of the DNSCrypt protocol.
Here is a small guide to set this up alongside NetworkManager.

## Installation

Install the dnscrypt package provided by your distribution:  
	+ [dnscrypt-proxy](https://archlinux.org/packages/?name=dnscrypt-proxy) for ArchLinux  
	+ [dnscrypt-proxy](https://packages.debian.org/search?keywords=dnscrypt-proxy) for Debian-based distribution  

## Configuration

### NetworkManager

In its default config, NetworkManager overwrite `/etc/resolv.conf`. 
In order to stop this behaviour, create a configuration file under `/etc/NetworkManager/conf.d/`, e.g. `/etc/NetworkManager/conf.d/00-dns.conf`, containing
```
[main]
dns=none
```

Restart `NetworkManager.service`

```bash
> sudo systemctl restart NetworkManager.service
```

### resolv.conf

`/etc/resolv.conf` is now a dead symlink.
Remove the file and create a new one with nameserver pointing to your localhost.

```bash
> sudo rm /etc/resolv.conf
> sudoedit /etc/resolv.conf
```

```
nameserver ::1
nameserver 127.0.0.1
options edns0 single-request-reopen
```

### dnscrypt-proxy

Make sure no other service makes use of port `:53` by running
```bash
> ss -lp 'sport = :domain'
```
If the command output something more than one line (starting with `Netid`), a process is using the port 53 (common ones are `dnsmasq` and `systemd-resolv`).  

By default, `dnscrypt-proxy` will chose the fastest resolver from the servers listed in `/etc/dnscrypt-proxy/dnscrypt-proxy.toml` under `[sources]`. 
Optionally you can setup specific resolvers.
For an up-to-date list of server see the [upstream page](https://download.dnscrypt.info/resolvers-list/v3/public-resolvers.md).

Enable and start the `dnscrypt-proxy` service.
```bash
> sudo systemctl start dnscrypt-proxy.service
> sudo systemctl enable dnscrypt-proxy.service
```

## Test your config

NetworkManager still shows the DNS server reported by your router:
```bash
> nmcli dev show | grep IP4.DNS
```

However, using `nslookup` should show localhost on port 53 as the resolver:
```bash
> nslookup valcarce.fr
Server:         ::1
Address:        ::1#53

Non-authoritative answer:
Name:   valcarce.fr
Address: 51.15.121.4
Name:   valcarce.fr
Address: 2001:bc8:1824:e4e::1
```
  
Finally, you can test your DNS via dnsleak websites such as [dnsleaktest.com](https://dnsleaktest.com/) or [mullvad.net](https://mullvad.net/en/check/).  
Notice for Firefox user: by default (in the US) Firefox uses DNS-over-HTTPS (DoH) and send all your DNS requests to CloudFlare.
To prevent this, open Firefox `Preferences`, search for `Network Settings`, click on `Settings...` and disable `Enable DNS over HTTPS`.

