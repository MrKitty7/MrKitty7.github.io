---
title: Ship VM With Docker
date: 2022-06-25
categories: [homelab, machines, virtual-machines]
tags: [servers, ubuntu, docker, grafana]
---

# Making the PERFECT Ship VM with Docker and Grafana
We'll be making a Ship VM, meaning it will store all our Docker containers, and also, we'll setup Grafana on it to monitor all our Docker containers. 
Let's get started!


## Let's begin by making our VM in Proxmox
Note: You can use any hypervisor you choose.

![I'll choose the name Ship.](/mrkitty7.github.io/assets/img/creating-vm1.png "img1")

![Next, I recommend you use Ubuntu Server 20.04 for your OS.](/mrkitty7.github.io/assets/img/creating-vm2.png "img2")

![Here, just copy my settings.](/mrkitty7.github.io/assets/img/creating-vm3.png "img3")

![Here you can give your VM the amount of storage it can use. I gave it 200GB. ](/mrkitty7.github.io/assets/img/creating-vm4.png "img4")

![Here, choose how much CPU cores and sockets you want to give your VM.](/mrkitty7.github.io/assets/img/creating-vm5.png "img5")

![Here, choose how much RAM you want to give your VM.](/mrkitty7.github.io/assets/img/creating-vm6.png "img6")

On the Networking section, just click next and finally finish. 

### After your VM has been created, start it up. Click on the console and install Ubuntu Server.

After it's done, reboot.

## Next, run these commands:
```bash
sudo apt update && sudo apt upgrade
```
After you update your machine, install some essential tools:
```bash
sudo apt install docker.io docker-compose net-tools htop python3 qemu-guest-agent -y
```
After that's done, we can create a Portainer volume:
```bash
docker volume create portainer_data
```
Now we can install Portainer:
```bash
sudo docker run -d -p 8000:8000 -p 9443:9443 --name portainer \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:2.9.3
```
Check if the container is running:
```bash
sudo docker ps
```
Now head over to Portainer's web interface:
```url
https://<ipaddress>:9443
```


Let's start by making a folder called docker_volumes in which we'll have all our volumes for containers.

## Now let's get Grafana LOKI set-up.
First, in docker_volume folder run these commands:
```bash
mkdir grafana
mkdir loki
mkdir promtail
```

Next, log in to your Portainer interface and head over to Stacks and click Add Stack. Name it 'loki'.


```yaml
version: "3"
networks:
  loki:
services:
  loki:
    image: grafana/loki:2.4.0
    volumes:
      - /home/serveradmin/docker_volumes/loki:/etc/loki
    ports:
      - "3100:3100"
    restart: unless-stopped
    command: -config.file=/etc/loki/loki-config.yml
    networks:
      - loki
  promtail:
    image: grafana/promtail:2.4.0
    volumes:
      - /var/log:/var/log
      - /home/serveradmin/docker_volumes/promtail:/etc/promtail
    # ports:
    #   - "1514:1514" # this is only needed if you are going to send syslogs
    restart: unless-stopped
    command: -config.file=/etc/promtail/promtail-config.yml
    networks:
      - loki
  grafana:
    image: grafana/grafana:latest
    user: "1000"
    volumes:
    - /home/serveradmin/docker_volumes/grafana:/var/lib/grafana
    ports:
      - "3000:3000"
    restart: unless-stopped
    networks:
      - loki
```
Make sure to replace the user 'serveradmin' with your user aswell as the user id. But do not run the stack yet.
```bash
nano loki/loki-config.yml
```
Put the following config into the file:
```yaml
auth_enabled: false

server:
  http_listen_port: 3100
  grpc_listen_port: 9096

common:
  path_prefix: /tmp/loki
  storage:
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1
  ring:
    instance_addr: 127.0.0.1
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2020-10-24
      store: boltdb-shipper
      object_store: filesystem
      schema: v11
      index:
        prefix: index_
        period: 24h

ruler:
  alertmanager_url: http://localhost:9093
```
```bash
nano promtail/promtail-config.yml
```
Put the following config into the file:
```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions:
  filename: /tmp/positions.yaml

clients:
  - url: http://loki:3100/loki/api/v1/push

#scrape_configs:

# local machine logs

#- job_name: local
 # static_configs:
 # - targets:
#      - localhost
 #   labels:
#      job: varlogs
 #     __path__: /var/log/*log
  
# docker logs
scrape_configs:
  - job_name: docker 
    pipeline_stages:
      - docker: {}
    static_configs:
      - labels:
          job: docker
          __path__: /var/lib/docker/containers/*/*-json.log

# syslog target

#- job_name: syslog
#  syslog:
#    listen_address: 0.0.0.0:1514 # make sure you also expose this port on the container
#    idle_timeout: 60s
#    label_structured_data: yes
#    labels:
#      job: "syslog"
#  relabel_configs:
#    - source_labels: ['__syslog_message_hostname']
#      target_label: 'host'
```

Now run the stack.

Install the docker plugin:
```bash
docker plugin install grafana/loki-docker-driver:latest --alias loki --grant-all-permissions
```

Edit the docker daemon json:
```bash
sudo nano /etc/docker/daemon.json
```
```json
{
    "log-driver": "loki",
    "log-opts": {
        "loki-url": "http://localhost:3100/loki/api/v1/push",
        "loki-batch-size": "400"
    }
}
```
```bash
sudo systemctl restart docker
```

Now, let's head to the Grafana interface.


Now let's add a new container.
Uptime Kuma:

```yaml
---
version: "3.1"

services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    volumes:
      - /home/serveradmin/docker_volumes/uptime-kuma/data:/app/data
    ports:
      - 3001:3001
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
```

Visit the interface at:
```url
http://192.168.1.5:3001
```
to be continued