# DNS configuration

rkt can automatically prepare `/etc/resolv.conf` and `/etc/hosts` for the apps in the pod. 
They can either be generated at runtime, or the host's configuration can be used.

## `/etc/resolv.conf`

Four options affect how this file is created:

* `--dns` : Specify either a DNS server, or one of the "magic" values `host` or `none`
* `--dns-domain` : The resolv.conf `domain` parameter
* `--dns-opt` : One or more resolv.conf `option` parameters
* `--dns-search` : One or more domains for the search list

The simplest configuration is:

```sh
$ sudo rkt run --dns=8.8.8.8 pod.aci
```

Other parameters can be given:

```sh
$ sudo rkt run \
	--dns=8.8.8.8 --dns=4.2.2.2 \
	--dns-domain=example.org \
	--dns-opt=debug --dns-opt=rotate \
	--dns-search=example.com --dns-search=example.gov \
	pod.aci
```

This will generate the following `/etc/resolv.conf` for the applications:

```
# Generated by rkt run

search example.com example.gov
nameserver 8.8.8.8
nameserver 4.2.2.2
options debug rotate
domain example.org
```

### "Magic" parameters

#### `host`
The magic parameter `host` will bind-mount the host's `/etc/resolv.conf` in to the applications.
This will be a read-only mount.

#### `none`
The magic parameter `none` will ignore any DNS configuration from CNI. This will ensure that
the image's `/etc/resolv.conf` has precedence.

### Precedence
`resolv.conf` can be generated by multiple components. The order of precedence is:

1. If `--dns`, et al. are passed to `rkt run`
2. If a CNI plugin returns DNS information, unless `--dns=none` is passed
3. If a volume is mounted on `/etc/resolv.conf`
4. If the application container includes `/etc/resolv.conf`

![resolv-conf-logic](resolv-conf-logic.png)

## `/etc/hosts`
`rkt run` provides one option with two modes:

* `--hosts-entry <IP>=<HOST>`  
* `--hosts-entry host`

Passing `--hosts-entry=host` will bind-mount (read-only) the hosts's `/etc/hosts`
in to every application.

When passing IP=HOST pairs:

```sh
$ rkt run ... --hosts-entry 198.51.100.0=host1,198.51.100.1=host2 --hosts-entry 198.51.100.0=host3
```

rkt will take some [standard defaults](../../stage1/net/rootfs/etc/hosts-fallback)
and append the requested entries.

```
< the default entries >

198.51.100.0 host1 host3
198.51.100.1 host2
```


### Precedence
`/etc/hosts` can be generated by multiple components. The order of precedence is:

1. If `--hosts-entry` is passed to `rkt run`
2. If a volume is mounted on `/etc/hosts`
3. If the app image includes `/etc/hosts`
4. Otherwise, a fallback stub `/etc/hosts` is created



## Example
The following example shows that the DNS options allow the pod to resolve names successfully:

```
$ sudo rkt run --net=host --dns=8.8.8.8 quay.io/coreos/alpine-sh --exec=/bin/ping --interactive -- -c 1 coreos.com
...

PING coreos.com (104.20.47.236): 56 data bytes
64 bytes from 104.20.47.236: seq=0 ttl=63 time=5.421 ms

--- coreos.com ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 5.421/5.421/5.421 ms
```