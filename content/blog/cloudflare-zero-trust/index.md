
+++
title = "Homelab Setup"
description = "Homelab on laptop"
date = 2023-09-28
draft = false

[taxonomies]
tags = ["docker", "docker-compose", "homelab"]
categories = ["Homelab"]

[extra]
toc = true
comment = true
+++


 &emsp; &emsp; After using my trusty old laptop for nearly 5 to 6 years, I noticed it had started making fan noises, even right after a fresh boot-up. Rather than retiring it or letting it gather dust, I decided to give it a new  life by transforming it into a personal homelab server. Leveraging the power of [Cloudflare Zero Trust](https://developers.cloudflare.com/cloudflare-one/) and a collection of Docker containers, I embarked on a journey to create a versatile homelab environment.

&emsp; &emsp; Within this homelab, I've set up a range of essential services that make daily life more enjoyable and efficient. These include a media player for movies, a music and audiobook player, a book reader, server monitoring tool, webpage monitoring capabilities, and a [nice looking homepage](https://homepage.pawanchhetri.com.np) accessible from virtually any device. What makes these services truly exceptional is their ability to accommodate multiple users, allowing family and friends to join in on the experience from anywhere.

&emsp; &emsp; To make your homelab setup as smooth as possible, I've  provided all the Docker Compose files you'll need for each of these services. With them, you'll be up and running in no time, ready to unlock the full potential of your homelab server. So, let's dive in and see how each services  with respective docker-compose stack looks like.

- [Homepage](#homepage)
- [Grafana, Prometheus, nodeexporter, cadvisor](#grafana-prometheus-nodeexporter-cadvisor)
- [Portainer](#portainer)
- [uptime-kuma](#uptime-kuma)
- [Jellyfin](#jellyfin)
- [Qbittorrent](#qbittorrent)
- [Navidrome](#navidrome)
- [Audiobookshelf](#audiobookshelf)
- [Kavita](#kavita)
- [Setup Requirements](#setup-requirements-and-best-practices)


<br />

#### [Homepage](https://homepage.pawanchhetri.com.np)
Homelab bookmarks tab, access all the self hosted services and more from this page. Allows fancy graphics and stuff too but I've gone only text based with minimal bookmarks.yml file configuration.

![](/images/2023-10-07-08-07-50.png)

```
version: "3.3"
services:
  homepage:
    image: ghcr.io/benphelps/homepage:latest
    container_name: homepage
    ports:
      - 8921:3000
    volumes:
      - ./config:/app/config # Make sure your local config directory exists
      - /var/run/docker.sock:/var/run/docker.sock:ro # (optional) For docker integrations
    restart: unless-stopped

```
`$ cat bookmarks.yaml`
```
---

- Monitoring:
    - Portainer:
        - abbr: Pr
          href: https://portainer.pawanchhetri.com.np/
    - Grafana:
        - abbr: gf
          href: https://grafana.pawanchhetri.com.np/
    - Uptime-kuma:
        - abbr: Up
          href: https://uptimekuma.pawanchhetri.com.np/
    - SSH:
       - abbr: sh
         href: https://ssh.pawanchhetri.com.np/

- Media:
   - Music:
       - abbr: Mu
         href: https://music.pawanchhetri.com.np
   - Jellyfin:
       - abbr: Jf
         href: https://jellyfin.pawanchhetri.com.np
   - Audiobook:
       - abbr: AS
         href: https://audiobook.pawanchhetri.com.np

- Reading:
    - Blog:
       - abbr: Bl
         href: https://pawanchhetri.com.np/
    - Books:
        - abbr: KV
          href: https://books.pawanchhetri.com.np

- Social:
    - Reddit:
        - abbr: RE
          href: https://reddit.com/
    - YouTube:
        - abbr: YT
          href: https://youtube.com/
    - LinkendIn:
        - abbr: LN
          href: https://www.linkedin.com/
    - Github:
        - abbr: GH
          href: https://github.com/Pawan-666/
    - Feedly:
        - abbr: rss
          href: https://feedly.com/
```
<br />
<br />



#### [Grafana Prometheus Nodeexporter Cadvisor](https://grafana.pawanchhetri.com.np)
For server and docker containers monitoring I've used Grafana. Helps to monitor the server CPU,Memory,Network status.
![](/images/2023-10-07-09-13-33.png)
![](/images/2023-10-07-09-12-52.png)
```
version: '3.3'
services:
``    node-exporter:
      # ports localhost:9100/metrics
        network_mode: host
        pid: host
        volumes:
            - '/:/host:ro,rslave'
        image: 'quay.io/prometheus/node-exporter:latest'

    prometheus:
        container_name: prometheus
        ports:
            - '9091:9090' #modify 9091 to your setup needs
        volumes:
            - './Configs/Prometheus/prometheus.yml:/etc/prometheus/prometheus.yml' #modify the path for your install location
        image: prom/prometheus

    cadvisor:
      image: gcr.io/cadvisor/cadvisor
      container_name: cadvisor
      ports:
        - 8080:8080
      volumes:
        - /:/rootfs:ro
        - /var/run:/var/run:rw
        - /sys:/sys:ro
        - /var/lib/docker:/var/lib/docker:ro

    grafana:            # admin/admin default username/password
        container_name: grafana
        ports:
            - '3457:3000' #modify 3457 to your setup needs
        image: grafana/grafana

```

`$ cat prometheus.yml`

```
global:
  scrape_interval: 5s
  external_labels:
    monitor: 'node'
scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['192.168.1.68:9090'] ## IP Address of the localhost. Match the port to your container port
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['192.168.1.68:9100'] ## IP Address of the localhost

  - job_name: 'docker-metrics'
    scrape_interval: 5s
    static_configs:
      - targets: ['192.168.1.68:9323']

  - job_name: 'cAvisor'
    static_configs:
      - targets: ['192.168.1.68:8080']
```

<br />
<br />

#### [Portainer](https://portainer.pawanchhetri.com.np)
I prefer and use terminal to deploy containers but portainer does provide nice gui to deploy docker stuffs with nice visualizations, bells and whistles.
![](/images/2023-10-07-08-17-59.png)
```
version: "3.7"

services:
  portainer:
    image: portainer/portainer-ce:2.11.0-alpine # Replace 2.11.0 with the latest version.
    command: -H unix:///var/run/docker.sock
    restart: always
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /volume1/docker/config/Portainer:/data
    ports:
      - 9443:9443
    healthcheck:
      test: ['CMD', 'wget', '--spider', 'localhost:9000']
      interval: 15s # Amount of time after it starts checking for readiness

```
<br />
<br />

#### [Uptime-kuma](https://uptimekuma.pawanchhetri.com.np)
Fantastic to monitor websites(up or not), latency and ssl cert expiry date, You can even share the status page with clients.

![](/images/2023-10-07-08-20-54.png)
```
version: '3'
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1.23.2
    container_name: uptime-kuma
    restart: always
    ports:
      - "3001:3001"
    volumes:
      - ./data:/app/data
```

<br />
<br />

#### [Jellyfin](https://jellyfin.pawanchhetri.com.np)
Your netflix at home. Also has plugin for subtitle download.

![](/images/2023-10-07-08-24-30.png)
![](/images/2023-10-07-08-27-57.png)
```
version: '3.3'
services:
    jellyfin:
        volumes:
            - './config:/config'
            - './cache:/cache'
            - '/home/pawan/Storage/movies/:/media'
        ports:
            - '8096:8096'
        container_name: jellyfin
        restart: unless-stopped
        image: jellyfin/jellyfin
```
<br />
<br />

#### [Qbittorrent](https://qbittorrent.pawanchhetri.com.np/)
This is a basic requirement to add movies and series to your jellyfin server. I don't use it and don't recommend it either !.

![](/images/2023-10-07-09-17-25.png)
```
---
version: "2.1"
services:
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=Etc/UTC
      - WEBUI_PORT=8081
    volumes:
      - ./config:/config
      - /home/pawan/Storage/movies/torrent:/downloads

    ports:
      - 8081:8081
      - 6881:6881
      - 6881:6881/udp
    restart: unless-stopped
```


<br />
<br />

#### [Navidrome](https://music.pawanchhetri.com.np)
Nice music player, especially great for those who prefer listening music with full albums like I do. It has android clients too.

![](/images/2023-10-07-08-29-53.png)
```
version: "3"
services:
  navidrome:
    image: deluan/navidrome:latest
    # user: 1000:1000 # should be owner of volumes
    ports:
      - "4533:4533"
    restart: unless-stopped
    environment:
      # Optional: put your config options customization here. Examples:
      ND_SCANSCHEDULE: 1h
      ND_LOGLEVEL: info  
      ND_SESSIONTIMEOUT: 24h
      ND_BASEURL: ""
    volumes:
      - "./data:/data"
      - "/home/pawan/Music/:/music:ro"
```

<br />
<br />

#### [Audiobookshelf](https://audiobook.pawanchhetri.com.np)
Same as the music player above but for audiobooks and podcasts.

![](/images/2023-10-07-08-35-29.png)
![](/images/2023-10-07-09-57-01.png)

```
version: "3.7"
services:
  audiobookshelf:
    image: ghcr.io/advplyr/audiobookshelf:latest
    container_name: audiobookshelf
    ports:
      - 13378:80
    volumes:
      - /home/pawan/Storage/movies/AudioBooks:/audiobooks
      - /home/pawan/Storage/movies/podcast:/podcasts
      - ./config/:/config
      - ./metadata/:/metadata
```
<br />
<br />

#### [Kavita](https://books.pawanchhetri.com.np)

An online book reader. I don't like reading softcopy books tho. Ctrl+c/v code snippets from technical books is useful at times and it helps to have a softcopy even if you have a hardcopy for this purpose only.
![](/images/2023-10-07-08-39-05.png)

```
version: '3.9'
services:
    kavita:
        image: kizaing/kavita:latest
        volumes:
            - /home/pawan/Storage/Bokks/Books:/manga
            - ./data:/kavita/config
        ports:
            - "5000:5000"
        restart: unless-stopped
```



<br />
<br />

#### Setup requirements and best practices
- Machine with a GNU/linux os 
    - You can run a linux vm with bridge networking set up
- Setting up static ip on your server for ip persistence
- Knowledge of how docker containers work and how to deploy them using docker-compose
- A domain name 
- Setting domains in Cloudflare
- Using [cloudflare zero trust](https://developers.cloudflare.com/cloudflare-one/) to map subdomains to the docker containers
- Desire to learn and troubleshoot, part of the process
- Setting strong passwords for each service
- Monitoring your server so that it doesn't burn!

<br />
I've also set up Cloudflare Access to ssh into my machine from the browser itself. A directory is set for each docker-compose stack. Helps to keep configuration easy and clean as they should.

![](/images/2023-10-07-08-56-38.png)
<br />
<br />

__Surf the subreddit [selfhosted](https://www.reddit.com/r/selfhosted/) for more awesomeness.__
