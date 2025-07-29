+++
title = "Setting up Metricbeat on a Docker Swarm Cluster"
date = "2021-12-25"
author = "Roman Böhringer"
cover = ""
tags = ["devops"]
keywords = ["docker", "docker-swarm", "elastic", "metricbeat"]
description = "How to use a Metricbeat sidecar container to monitor your Docker Swarm hosts and additional services."
showFullContent = false
+++

To monitor your Docker Swarm hosts (and containers) with Metricbeat, you can set up a global sidecar service. You need to mount the host's filesystem, `/sys/fs/cgroup`, `/proc`, and `/var/run/docker.sock` inside the container to do so. An example compose file for the sidecar looks like this:

{{< code language="yaml" title="docker-stack.yml" >}}
version: '3.7'

services:
  metricbeat:
    image: docker.elastic.co/beats/metricbeat:7.16.1
    user: root
    hostname: "{{.Node.Hostname}}-{{.Service.Name}}"
    configs:
      - source: metricbeat-config
        target: /usr/share/metricbeat/metricbeat.yml
    command:
      - -e
      - --strict.perms=false
      - --system.hostfs=/hostfs
    volumes:
      - type: bind
        source: /
        target: /hostfs
        read_only: true
      - type: bind
        source: /sys/fs/cgroup
        target: /hostfs/sys/fs/cgroup
        read_only: true
      - type: bind
        source: /proc
        target: /hostfs/proc
        read_only: true
      - type: bind
        source: /var/run/docker.sock
        target: /var/run/docker.sock
        read_only: true
    deploy:
      mode: global

configs:
  metricbeat-config:
    file: ./metricbeat.yml
{{< /code >}}

The `hostname` ensures that the sidecar reports metrics with the hostname of the Swarm host (and the service name) such that you can distinguish the reported metrics easily.
An example `metricbeat.yml` for monitoring the Swarm nodes is provided below:

{{< code language="yaml" title="metricbeat.yml" >}}
## Metricbeat configuration
## https://github.com/elastic/beats/blob/master/deploy/docker/metricbeat.docker.yml
#

metricbeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    # Reload module configs as they change:
    reload.enabled: false

metricbeat.autodiscover:
  providers:
    - type: docker
      hints.enabled: true

metricbeat.modules:
- module: docker
  metricsets:
    - container
    - cpu
    - diskio
    - healthcheck
    - info
    - memory
    - network
  hosts: ['unix:///var/run/docker.sock']
  period: 10s
  enabled: true


processors:
  - add_cloud_metadata: ~

output.elasticsearch:
  hosts: ["elastic-01:9200"]
  ...
{{< /code >}}

You can of course manually configure other modules in the file to monitor other services in your cluster. Thanks to the enabled `autodiscover` feature, this can also be done more easily by setting labels on the services you want to monitor, e.g. for Redis:
```yaml
    labels:
      co.elastic.metrics/module: "redis"
      co.elastic.metrics/hosts: "redis1:6379"
```
