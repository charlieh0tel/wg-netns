Simplified fork of [wg-netns](https://github.com/dadevel/wg-netns).

Notable changes:

* All use of sudo and shell have been eliminated to harden it.

* Packaged for Debian-based distros.


## Usage

First, create a configuration profile:

~~~ json
{
  "name": "ns-example",
  "interfaces": [
    {
      "name": "wg-example",
      "address": ["10.10.10.192/32", "fc00:dead:beef::192/128"],
      "private-key": "4bvaEZHI...",
      "peers": [
        {
          "public-key": "bELgMXGt...",
          "endpoint": "vpn.example.com:51820",
          "allowed-ips": ["0.0.0.0/0", "::/0"]
        }
      ]
    }
  ]
}
~~~

Now it's time to setup your new network namespace and all associated
wireguard interfaces.

~~~ bash
wg-netns up ./example.yaml
~~~

Profiles stored under `/etc/wireguard/` can be referenced by their
name.

~~~ bash
wg-netns up example
~~~

You can verify the success with a combination of `ip` and `wg`.

~~~ bash
ip netns exec ns-example wg show
~~~

You can also spawn a shell inside the netns.

~~~ bash
ip netns exec ns-example bash -i
~~~

### Systemd Service

A `wg-quick@.service` equivalent called `wg-netns@.service` is part of
this pacakge.

Place your profile in `/etc/wireguard/`, e.g. `example.json`, then
enable the service.

~~~ bash
systemctl enable --now wg-netns@example.service
~~~
