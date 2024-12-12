---
comments: true
draft: false
date: 2024-12-12
categories:
  - python
  - docker
  - containers
  - dns
authors:
  - tupcakes
---


# Auto Update ControlD Folder Rules from https://github.com/hagezi/dns-blocklists

## The Issue
When recently I found the amazing repo at https://github.com/hagezi/dns-blocklists, which maintains various block lists in different formats. Including some for ControlD folder rule json exports.

## Project Repo
https://github.com/tupcakes/controld-updater

<!-- more -->

This a small python scritp I made to update controld folder rules based on the lists maintained at: https://github.com/hagezi/dns-blocklists/tree/main/controld. I personally use the badware and referral lists, and was sick of having to keep track of when they needed to be refreshed in controld. I have the script setup to run every 24 hours for each list I use using kubernetes cronjobs. The script deletes the old list then re-imports it from the hagezi/dns-blocklist repo nightly.


## Building
Clone the repo.
```shell
git clone https://github.com/tupcakes/controld-updater.git
```

Build the image.
```shell
podman build -t controld-updater .
```

## Run the container
*only needed if not building from source*
```shell
podman pull ghcr.io/tupcakes/controld-updater
```
*Image is built to support both amd64 and arm64*
```shell
podman run controld-updater \
    -a "api.abcd..." \
    -p "2342342kjhkj" \
    -g "Referral" \
    -b "https://raw.githubusercontent.com/hagezi/dns-blocklists/refs/heads/main/controld/referral-allow-folder.json"
```

The value for "-g" option shown above, should match the group name in the corresponding json file.
```json
{
  "group": {
    "group": "Referral",
    "action": {
      "do": 1,
      "status": 1
    }
  },
  "rules": [
```

## Kubernetest Manifest Example
This is currently how I run the container on a schedule. I just create a manifest for each cronjob and list.
```yaml
---
apiVersion: batch/v1
kind: CronJob
metadata:
  name: controld-updater-referral
spec:
  schedule: "0 1 * * *"
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: controld-updater
            image: ghcr.io/tupcakes/controld-updater:latest
            args:
            - -a
            - "api.asdfasdf"
            - -p
            - "123412asdfas"
            - -g
            - "Referral"
            - -b
            - "https://raw.githubusercontent.com/hagezi/dns-blocklists/refs/heads/main/controld/referral-allow-folder.json"
          restartPolicy: OnFailure
```
