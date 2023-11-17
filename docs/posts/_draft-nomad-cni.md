---
comments: true
draft: true
date: 1900-01-01
categories:
  - Homelab
  - Hashi
  - Nomad
authors:
  - tupcakes
---


# Nomad and CNI
In the [previous post](https://www.practicalkungfu.net/2023/11/13/nomad-consul-glusteroh-my/#install-cni-plugins) I went over the install steps to add the CNI plugins to nomad.

I'm not super familiar with the nuances of using the cni plugins. My personal use case is for when I have a service that needs (or would be easier to implement with) it's own IP address.

A good example of this is anything that listens on a static port like a load balancer or in the example I'll use, Unifi Controller.

The CNI [documentation](https://www.cni.dev/plugins/current/main/) is a good reference for the options available.

<!-- more -->

There are many ways to implement this, but the two I like are static assignment of an IP to a service or using a range of IPs that get assigned dynamically. You can also use your networks DHCP to have the containr request an IP address, however that appears to require a seperate daemon running.

The dynamicly assigned IP option can use a consul service associated with it to have an always updated dns record.

## ipvlan

### CNI Config
```json title="/opt/cni/config/ipvlan.conflist"
{
  "cniVersion": "0.4.0",
  "name": "ipvlan",
  "plugins": [
    {
      "type": "ipvlan",
      "ipam": {
        "type": "host-local",
        "ranges": [
          [
            {
              "subnet": "10.28.0.0/24",
              "rangeStart": "10.28.0.50",
              "rangeEnd": "10.28.0.75",
              "gateway": "10.28.0.1"
            }
          ]
        ],
        "routes": [
            {
                "dst": "0.0.0.0/0"
            }
        ],
        "dns": {
            "nameservers": [
                "10.28.0.19",
                "10.28.0.32"
            ],
            "domain": "tupkers.net",
            "search": [
                "tupkers.net"
            ]
        }
      }
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      },
      "snat": true
    }
  ]
}
```

```
systemctl restart nomad
```

## macvlan
### CNI Config