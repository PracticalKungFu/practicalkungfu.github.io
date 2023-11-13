---
comments: true
date: 2023-11-14
categories:
  - Homelab
  - Hashi
  - Nomad
authors:
  - tupcakes
---


# Nomad Jobs
This is pretty easy once you get the hang of the job definitions. It's honestly not much diffent than docker-compose definitions.

<!-- more -->

Navigate to the web ui at http://10.28.0.19:4646. In the upper right corner there should be a "Run Job" button. Click the button. Then paste in the below job definition.

## Whoami Example
```hcl
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

After a few seconds it will spin up a simple http service on a random port. The random port will redirect to the conainer port 80. Click through the web interface and you should be able to find the IP and random port that the web applicaiton is listening on. It should be something like: `http://10.28.0.19:31106`.

Yeah that url sucks and no one is going to want to type that in every time. Also the port will change if the container gets spun up on a differnt node for some reason.

There are ways around this using consul for service discovery and an ingress service like Traefik. I'll go over those in the next articles.

Alternatively you can set the port to a `static` type and it will use port 80 on the host rather than a random port.

```hcl
    network {
      port "http" {
        static = 80
      }
    }
```

## Storage
If your application requires storage, you can do something like this in the config stanza. There are a lot of ways you can do storage, I just happen to like this method when using gluster.

```hcl
      config {
        image = "traefik/whoami"
        force_pull = true
        ports = ["http"]
        volumes = [
          "/mnt/nomad-vol/${NOMAD_JOB_NAME}/${NOMAD_TASK_NAME}:/containerpath"
        ]
      }
```

## Wrappup
There is a TON to the job definitions, too much to go over here. I highly recommend reading through the [job definition docs](https://developer.hashicorp.com/nomad/docs/job-specification) to get a feel for what you can do.