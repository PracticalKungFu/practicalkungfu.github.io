---
comments: true
date: 2023-11-16
categories:
  - Homelab
  - Hashi
  - Traefik
authors:
  - tupcakes
---


# Nomad and Traefik
!!! note

    There is a lot here. I've done my best to make things correct, but I'm also still learning how a lot of nomad, consul, etc... work and their best practices. It's entirely possible I missed something or could have done something better.

The goal here is to be able to have a web application available at the address `https://whoami.practicalkungfu.net`. To do this we'll use Traefik, which is a very nice reverse proxy/load balancer that has integrations with a number of container orchestration platforms.

<!-- more -->

## A few Things of Note in the Job

### Constraints
```hcl
      group "infra" {
        count = 2

        constraint {
          attribute = "${attr.cpu.arch}"
          operator  = "="
          value     = "amd64"
        }
```

`count = 2` simply means that it will create two instances of the traefik container.

The constraint stanza uses metadata of the nomad nodes to tell the jobs where they should run. In this case it will only run on nodes with an amd64 architecture.

### Tags
```hcl
        service {
          name = "${NOMAD_JOB_NAME}-https"
          provider = "consul"
          port = "https"
          tags = [
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls=true",
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls.certresolver=cloudflare",
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls.domains[0].main=practicalkungfu.net",
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls.domains[0].sans=*.practicalkungfu.net",
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls.certresolver=letsencrypt"
          ]
        }
```

The tags section of the service stanza contains tags that Traefik will look for.

`"traefik.http.routers.${NOMAD_JOB_NAME}.tls=true"` tells traefik that a tls cert is needed

`"traefik.http.routers.${NOMAD_JOB_NAME}.tls.certresolver=cloudflare"` tells traefik to use cloudflare for verification. There are quite a few ways of doing this depending on the resolver. Or you could even use http verification (personally I don't like using that. DNS for the win).


`"traefik.http.routers.${NOMAD_JOB_NAME}.tls.domains[0].main=practicalkungfu.net"` and `"traefik.http.routers.${NOMAD_JOB_NAME}.tls.domains[0].sans=*.practicalkungfu.net"` specify the SANs to put on the cert.

`"traefik.http.routers.${NOMAD_JOB_NAME}.tls.certresolver=letsencrypt"` tells Traefik to use letsencrypt for cert generation.


### Envinronment Variables
```hcl
          env {
            CF_API_EMAIL = "email@domain.com"
            CF_API_KEY = "APIKEY"
          }
```

the two environment variables here are unique to cloudflare. You will need to populate them with your email and api key. Or lookup the correct settings for the DNS verification mechanisim you are using.


### Testing Cert Generation
```
"--certificatesresolvers.letsencrypt.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory",
#"--certificatesresolvers.letsencrypt.acme.caserver=https://acme-v02.api.letsencrypt.org/directory",
```

The above lines are for using either the staging server for letsencrypt or the prod server. It's always good to use staging first until you are sure everything is working. Otherwise the prod server might block you for a time.


### Complete Traefik Job
??? note "Job Definition"

    ```hcl title="traefik.hcl"
    job "traefik" {
      datacenters = ["dc1"]
      type        = "service"

      group "infra" {
        count = 2

        constraint {
          attribute = "${attr.cpu.arch}"
          operator  = "="
          value     = "amd64"
        }

        network {
          mode = "host"
          
          port  "http"{
            static = 80
          }
          port  "https"{
            static = 443
          }
          port  "admin"{
            static = 8080
          }
        }

        service {
          name = "${NOMAD_JOB_NAME}-http"
          provider = "consul"
          port = "http"
        }
        service {
          name = "${NOMAD_JOB_NAME}-https"
          provider = "consul"
          port = "https"
          tags = [
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls=true",
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls.certresolver=cloudflare",
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls.domains[0].main=practicalkungfu.net",
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls.domains[0].sans=*.practicalkungfu.net",
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls.certresolver=letsencrypt"
          ]
        }

        task "server" {
          driver = "docker"
          env {
            CF_API_EMAIL = "email@domain.com"
            CF_API_KEY = "APIKEY"
          }

          config {
            image = "traefik:v2.10"
            force_pull = true
            ports = ["admin", "http", "https"]
            args = [
              "--global.checkNewVersion=true",
              "--global.sendAnonymousUsage=false",

              "--api.dashboard=true",
              "--api.insecure=true",

              "--entrypoints.web.address=:${NOMAD_PORT_http}",
              "--entrypoints.websecure.address=:${NOMAD_PORT_https}",
              "--entrypoints.traefik.address=:${NOMAD_PORT_admin}",
              "--entrypoints.web.http.redirections.entryPoint.to=websecure",
              "--entrypoints.web.http.redirections.entryPoint.scheme=https",

              "--certificatesresolvers.letsencrypt.acme.caserver=https://acme-staging-v02.api.letsencrypt.org/directory",
              #"--certificatesresolvers.letsencrypt.acme.caserver=https://acme-v02.api.letsencrypt.org/directory",
              "--certificatesresolvers.letsencrypt.acme.storage=/etc/traefik/acme.json",
              "--certificatesresolvers.letsencrypt.acme.dnschallenge=true",
              "--certificatesresolvers.letsencrypt.acme.dnschallenge.resolvers=1.1.1.1:53",
              "--certificatesresolvers.letsencrypt.acme.dnschallenge.provider=cloudflare",

              "--providers.file.directory=/etc/traefik/configs",
              "--providers.file.watch=true",

              "--providers.consulcatalog=true",
              "--providers.consulcatalog.exposedByDefault=false",
              "--providers.consulcatalog.endpoint.address=consul.service.dc1.consul:8500",

              "--log.level=info",
              "--log.filepath=/etc/traefik/log/traefik.log",
              "--accessLog.filepath=/etc/traefik/log/access.log"
            ]
            volumes = [
              "/mnt/nomad-vol/${NOMAD_JOB_NAME}/${NOMAD_TASK_NAME}:/etc/traefik"
            ]
          }
        }
      }
    }
    ```

Now run the job and watch the logs. It should generate the cert for the traefik job. It can take a minute or two because of the way DNS verification works.

!!! warning

    Do not switch over to to the proc cert server until cert generation is working correctly.

You can now create a CNAME in DNS that points traefik.practicalkungfu.net to traefik-https.service.dc1.consul. The Traefik web page should be accessible at `https://traefik.practicalkungfu.net`. If everything looks good with the cert (you might still get a cert warning since its a staging cert) then move the comment off the prod server to the staging cert server. Then redeploy the job.

!!! info

    You might need to delete the acme.json file before redeploying the job with the prod server.

## Whoami Job Changes
??? info "whoami.hcl"

    ```
    job "whoami" {
      datacenters = ["dc1"]

      type = "service"

      group "demo" {
        count = 2

        network {
          port "http" {
            to = 80
          }
        }

        service {
          name = "${NOMAD_JOB_NAME}-http"
          port = "http"
          provider = "consul"

          tags = [
            "traefik.enable=true",
            "traefik.http.routers.${NOMAD_JOB_NAME}.rule=Host(`whoami.practicalkungfu.net`)",
            "traefik.http.routers.${NOMAD_JOB_NAME}.entrypoints=web,websecure",
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls.certresolver=letsencrypt"
          ]
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

### Adding Tags
```hcl
        service {
          name = "${NOMAD_JOB_NAME}-http"
          port = "http"
          provider = "consul"

          tags = [
            "traefik.enable=true",
            "traefik.http.routers.${NOMAD_JOB_NAME}.rule=Host(`whoami.practicalkungfu.net`)",
            "traefik.http.routers.${NOMAD_JOB_NAME}.entrypoints=web,websecure",
            "traefik.http.routers.${NOMAD_JOB_NAME}.tls.certresolver=letsencrypt"
          ]
        }
```

Again the traefik tags need to be set. The rest of the job is the same as before.

!!! info

    Remember cert generation can take a couple minutes.

### DNS Change
Next we need to change the DNS entry for whoami. Orginally we pointed the CNAME at the consul alias for the whoami service. However now we want users to go through Traefik so we need change the CNAME to:

`whoami.practicalkungfu.net` -> `whoami.service.dc1.consul`

Now the whoami app should be accessible at `https://whoami.practicalkungfu.net`.