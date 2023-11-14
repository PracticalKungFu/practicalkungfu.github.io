---
comments: true
date: 2023-11-15
categories:
  - Homelab
  - Hashi
  - Consul
authors:
  - tupcakes
---

# Consul and DNS
!!! note

    There is a lot here. I've done my best to make things correct, but I'm also still learning how a lot of nomad, consul, etc... work and their best practices. It's entirely possible I missed something or could have done something better.

In the last couple posts I described how to setup a small(ish) Hashi Nomad and Consul cluster. Using Consul for service discovery is something that took me a little while to figure out.

Lets start by spinning up the whoami job if you haven't already.

<!-- more -->

??? note "whoami job definition"

    ```
    job "whoami" {
      datacenters = ["dc1"]

      type = "service"

      group "demo" {
        count = 1

        network {
          port "http" {
            to = 80
          }
        }

        service {
          name = "whoami-demo"
          port = "http"
          provider = "consul"
        }

        task "server" {
          resources {
            cpu    = 10
            memory = 10
          }

          env {
            WHOAMI_PORT_NUMBER = "${NOMAD_PORT_http}"
          }

          driver = "docker"

          config {
            image = "traefik/whoami"
            force_pull = true
            ports = ["http"]
          }
        }
      }
    }
    ```

## The Problem
Normally nomad will spin up the job on a random node with a random port. You could then access the service by navigating to `http://10.28.0..41:31106/`. Well that url sucks... We want our users to be able to use something like http://whoami.practicalkungfu.net.

The high level overview is that Consul can respond to DNS requests on port 8600 by default. Lets keep using whoami as an example. We `whoami-demo` running on one of the nodes. You can point nslookup at one of your consul nodes, on port 8600, and it should respond with the A record of the host running the service.

```
nslookup -port=8600 whoami-demo.service.dc1.consul 10.28.0.19
# or
nslookup -port=8600 whoami-demo.service.consul 10.28.0.19
```

```
Non-authoritative answer:
Name:    whoami-demo.service.dc1.consul
Address:  10.28.0.41
```

## DNS forwarding
The first step is to create a conditional forwarder in DNS your internal dns environment for the domain `consul`. That domain should forward dns requests to some of your consul servers.

!!! info

    The steps for setting up a conditional forwarder varies by DNS server. Personally I use Technitium DNS, but bind, or Windows Server DNS would work.

In my case I have it forwarding to:

- 10.28.0.19:8600
- 10.28.0.20:8600

You should now be able to resolve the service by dns name.

```
nslookup whoami-demo.service.dc1.consul
# or for a more generalize service address
nslookup whoami-demo.service.consul
```

```
Non-authoritative answer:
Name:    whoami-demo.service.dc1.consul
Address:  10.28.0.41
```

In we web browser you should also be able to go to `http://whoami-demo.service.dc1.consul:31106`. Sadly the URL still sucks.


### CNAME for your Main Domain
We can make this a little better.

Create a CNAME record in your main zone (practicalkungfu.net for me) as follows.

```
whoami.practicalkungfu.net -> whoami-demo.service.dc1.consul
```

Now you should be able to access the web app at the following address `http://whoami.tupkers.net:31106`.

That address still sucks, but it's a little better.

## Wrappup
This can be made better with Traefik. Traefik allows you to do SSL offload and acts as an ingress for web apps in the cluster.

I was planning to add the Traefik setup to this post as well, but it was getting a little long. I'll seperate out Traefik into it's own post.