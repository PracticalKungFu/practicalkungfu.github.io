---
comments: true
draft: false
date: 2023-11-13
categories:
  - Homelab
  - Hashi
  - Nomad
  - Consul
  - Gluster
authors:
  - tupcakes
---


# Nomad, Consul, Gluster...Oh My
!!! note

    There is a lot here. I've done my best to make things correct, but I'm also still learning how a lot of nomad, consul, etc... work and their best practices. It's entirely possible I missed something or could have done something better.
    
The goal here is to setup a 5 node nomad cluster (3 servers and 5 clients), a 5 node consul cluster, and a storage backend using a 3 node gluster cluster (I love saying gluster cluster). Gluster is mounted to a VIP provided by keepalived. I've found this gives a decent amount or resiliancy.

For the nodes in this cluster I use Debian 12, Armbian, or DietPi. Basically all Debian 12.

<!-- more -->

## Servers and Roles
nomad01 (10.28.0.19) - old laptop

- nomad server
- nomad client
- consul server
- consul client
- gluster storage
- keepalived

nomad02 (10.28.0.20) - old laptop

 - nomad server
 - nomad client
 - consul server
 - consul client
 - gluster storage
 - keepalived

nomad03 (10.28.0.41) - libre computer renegade

 - nomad server
 - nomad client
 - consul server
 - consul client
 - gluster arbiter

nomad04 (10.28.0.42) - rpi 3

 - nomad client
 - consul client

nomad05 (10.28.0.43) - rpi 3

 - nomad client
 - consul client


## Install Consul
Consul will be installed on all nodes in the cluster, because why not. It's very light weight.

### Scope
All servers

### Install
```
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install consul
```

### Create config
```
mv /etc/consul.d/consul.hcl /etc/consul.d/consul.hcl.orig
```

Create a new consul.hcl file and use the below configs on the related server.

Since I have kind of a random set of hardware I'm running I wanted to have as few unique configs as possible. I found out you can template the configs in regards to the bind_addr, which is what the `"{{ GetInterfaceIP \"eno1|eth0|end0\" }}"` is about.

#### Consul Server
Use the below config on the server hosts.
``` title="/etc/consul.d/consul.hcl"
datacenter  = "dc1"
data_dir    = "/opt/consul"
bind_addr   = "{{ GetInterfaceIP \"eno1|eth0|end0\" }}"
client_addr = "0.0.0.0"
retry_join  = ["10.28.0.19", "10.28.0.20", "10.28.0.41"]
ports {
  grpc = 8502
}
addresses {
  dns = "0.0.0.0"
}
ui               = true
server           = true
bootstrap_expect = 3
encrypt          = "ESTeHTyz2Agr8iSFlCLIYOJBSulSQ3XDVSHc1A5yODs="
connect {
  enabled = true
}
```

#### Consul Client
Use the below config on the client only hosts.
``` title="/etc/consul.d/consul.hcl"
datacenter  = "dc1"
data_dir    = "/opt/consul"
bind_addr   = "{{ GetInterfaceIP \"eno1|eth0|end0\" }}"
client_addr = "0.0.0.0"
retry_join  = ["10.28.0.19", "10.28.0.20", "10.28.0.41"]
ports {
  grpc = 8502
}
addresses {
}
encrypt = "ESTeHTyz2Agr8iSFlCLIYOJBSulSQ3XDVSHc1A5yODs="
connect {
  enabled = true
}
```

### Start Service
```
systemctl enable consul
systemctl start consul
```

### Web Interface
Any of the server hosts should have a web page available at: http://ip:8500/ui


## Install Nomad

### Scope
Nomad servers

- nomad01
- nomad02
- nomad03

Nomad clients

- all servers

### Install docker
This step is pretty much straight out of the Docker [documentation](https://docs.docker.com/engine/install/debian/).
```
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do sudo apt-get remove $pkg; done

sudo apt-get update
sudo apt-get install ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo \
  "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### Install the required packages.
```
sudo apt-get update && \
  sudo apt-get install wget gpg coreutils
```

### Add the HashiCorp GPG key.
```
wget -O- https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
```

### Add the official HashiCorp Linux repository.
```
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
```

### Update and install.
```
sudo apt-get update && sudo apt-get install nomad
```

### Install CNI plugins
This allows network namespaces to use bridge mode. This must be done on all nodes in the cluster.
```
curl -L -o cni-plugins.tgz "https://github.com/containernetworking/plugins/releases/download/v1.0.0/cni-plugins-linux-$( [ $(uname -m) = aarch64 ] && echo arm64 || echo amd64)"-v1.0.0.tgz && \
  sudo mkdir -p /opt/cni/bin && \
  sudo tar -C /opt/cni/bin -xzf cni-plugins.tgz
```

#### Allow containers through bridge network
```
echo 1 | sudo tee /proc/sys/net/bridge/bridge-nf-call-arptables && \
echo 1 | sudo tee /proc/sys/net/bridge/bridge-nf-call-ip6tables && \
echo 1 | sudo tee /proc/sys/net/bridge/bridge-nf-call-iptables
```

``` title="/etc/sysctl.d/bridge.conf"
net.bridge.bridge-nf-call-arptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
```

### Create config
```
mv /etc/nomad.d/nomad.hcl /etc/nomad.d/nomad.hcl.orig
```

#### Nomad Server Config
The server config also allows the host to be used when scheduling containers (the client stanza below). Use this config on the hosts you want to be nomad servers.

``` title="/etc/nomad.d/nomad.hcl"
datacenter = "dc1"
data_dir   = "/opt/nomad"
advertise {
  http = "{{ GetInterfaceIP \"eno1|eth0|end0\" }}"
  rpc  = "{{ GetInterfaceIP \"eno1|eth0|end0\" }}"
  serf = "{{ GetInterfaceIP \"eno1|eth0|end0\" }}"
}
server {
  enabled          = true
  bootstrap_expect = 3
  server_join {
    retry_join = ["10.28.0.19", "10.28.0.20", "10.28.0.41"]
  }
  encrypt = "ESTeHTyz2Agr8iSFlCLIYOJBSulSQ3XDVSHc1A5yODs="
}
client {
  enabled = true
  server_join {
    retry_join = ["10.28.0.19", "10.28.0.20", "10.28.0.41"]
  }
}
plugin "docker" {
  config {
    allow_privileged = true
    allow_caps = ["all"]
    volumes {
      enabled = true
    }
  }
}
```

#### Nomad Client Config
The nomad client config below will only schedule jobs, but not be considered nomad servers.
``` title="/etc/nomad.d/nomad.hcl"
datacenter = "dc1"
data_dir   = "/opt/nomad"
advertise {
  http = "{{ GetInterfaceIP \"eno1|eth0|end0\" }}"
  rpc  = "{{ GetInterfaceIP \"eno1|eth0|end0\" }}"
  serf = "{{ GetInterfaceIP \"eno1|eth0|end0\" }}"
}
client {
  enabled = true
  server_join {
    retry_join = ["10.28.0.19", "10.28.0.20", "10.28.0.41"]
  }
}
plugin "docker" {
  config {
    allow_privileged = true
    allow_caps = ["all"]
    volumes {
      enabled = true
    }
  }
}
```

### Start Service
```
systemctl enable nomad
systemctl start nomad
```

### Web Interface
Any of the server hosts should have a web page available at: http://ip:4646/ui


## Install Keepalived
We use keepalived to provide a single IP to use when mounting gluster volumes. The IP can float between 2 (or more) hosts if one goes down. Since only the first two nomad hosts are also gluster server with storage, we'll only configure keepalived for 2 hosts.

!!! info

    [There is an alternative to using keepalived for HA that I've read about.](https://www.practicalkungfu.net/2023/11/13/nomad-consul-glusteroh-my/#mount-at-boot)

### Scope
- nomad01
- nomad02

### Install
apt -y install keepalived

### Master Config
Note the `state MASTER` and `priority 255` lines.

``` title="/etc/keepalived/keepalived.conf"
vrrp_instance VI_1 {
        state MASTER
        interface eno1
        virtual_router_id 51
        priority 255
        advert_int 1
        authentication {
              auth_type PASS
              auth_pass 12345
        }
        virtual_ipaddress {
              10.28.0.50/24
        }
}
```

Restart keepalived and you should see the VIP on the master host.
```
systemctl restart keepalived
ip -brief addr
```

### Backup Config
!!! note
    Note the `state BACKUP` and `priority 254` lines.

``` title="/etc/keepalived/keepalived.conf"
vrrp_instance VI_1 {
        state BACKUP
        interface eno1
        virtual_router_id 51
        priority 254
        advert_int 1
        authentication {
              auth_type PASS
              auth_pass 12345
        }
        virtual_ipaddress {
              10.28.0.50/24
        }
}
```

```
systemctl restart keepalived
```

If you want to test it, you can stop keepalived on the MASTER host. The VIP should automaticaly move over to the BACKUP host.


## Install Gluster
Gluster server should be installed on at least 3 servers. In my setup I have two nodes that host the volume and a 3rd Raspberry Pi 3 that is an arbiter.

The arbiter is a member of the gluster pool that doesn't save any file data. It'll still have the directory struction in the brick.

### Scope
- nomad01
- nomad02
- nomad03

### Install
```
apt install glusterfs-server
systemctl enable glusterd
systemctl start glusterd
```

### Create Gluster Volume
Make the gluster file system location. Run on all gluster nodes including the arbiter.
```
mkdir -p /glusterfs/bricks/brick
```

Create the volume with 2 replica and 1 arbiter.
```
gluster volume create nomad-vol replica 2 arbiter 1 transport tcp 10.28.0.19:/glusterfs/bricks/brick 10.28.0.20:/glusterfs/bricks/brick 10.28.0.41:/glusterfs/bricks/brick force
```

### Start the Volume
Start the volume and check status.
```
gluster volume start nomad-vol
gluster volume info
```

!!! warning "Do not write directly to the brick location"
    One thing of note with gluster is that you should NOT write directly to the brick location `/glusterfs/bricks/brick`. Doing so will not replicate the written data accross the other nodes. You need to mount the gluster volume to a mount point (described in the next section) and use that when reading/writing data.


## Mount Gluster Volume

### Scope
All servers

### Install Client
The gluster client needs to be installed on all nodes in the nomad cluster.
```
apt install glusterfs-client -y
```

This assumes you have setup keepalived. Since nomad01 and nomad02 are both gluster hosts, we can use the keepalived VIP. Regardless if nomad01 goes down the VIP will just move over to nomad02 and the hosts in the nomad cluster will still be able to connect to the gluster storage.

### Create a mount point on all the hosts.
```
mkdir /mnt/nomad-vol
```

### Mount at Boot
Add a mount line to the fstab of all the hosts so it mounts automatically.
``` title="/etc/fstab"
10.28.0.50:/nomad-vol /mnt/nomad-vol/ glusterfs  defaults,_netdev,noauto,x-systemd.automount 0 0
```

!!! info 

    I found out you can also get HA without keepalived using `backup-volfile-servers=<volfile_server2>:<volfile_server3>`.
    ```
    10.28.0.19:/nomad-vol /mnt/nomad-vol/ glusterfs  defaults,_netdev,noauto,x-systemd.automount,backup-volfile-servers=10.28.0.20 0 0
    ```


Mount gluster.
```
mount -a
```

## Wrappup
The whole hashi stack may seem complex, but for me is much simpler than somthing like Kubernetes, which I used to run before Nomad.

The only thing from the hashi stack that I didn't go over in this guide is Vault, which is a secret store. Honestly I haven't looked that much at how it works, and will save that for a future article.

In the next part of this series I'll go over how you can use Consul with DNS for service location.



## Edits
- Boot mounting wasn't working for some reason. Changed to mount on demand.
- Option for HA without keepalived