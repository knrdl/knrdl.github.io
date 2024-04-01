---
title: "Scan all Docker images on a host for vulnerabilities"
date: "2024-04-01"
tags: [docker, trivy]
lang: "en"
---

Aquasec Trivy is an open-source vulnerability scanner for dependencies. It can be used to get a quick overview of all (tagged) Docker images on a host and their respective vulnerabilities. This can be done with the following Compose setup:

```yaml
services:
  secscan-reports:
    image: caddy:alpine
    command: caddy file-server --root /reports --browse --listen :8080
    volumes:
      - ./reports:/reports:ro
    ports:
      - 127.0.0.1:8080:8080  # exposed at http://localhost:8080/?sort=time&order=desc

  trivy:
    image: aquasec/trivy
    environment:
      DOCKER_HOST: tcp://docker-socket-protector:2375
    entrypoint: [ "/bin/sh", "-c" ]
    command:
      - |
        sleep 10m
        apk add docker-cli jq
        while true; do
          for imagename in $(docker ps -a --format="{{.Image}}" | uniq); do 
            echo "Scanning image: $$imagename"
            filename=$(echo -n "$$imagename" | jq -sRr '@uri')
            rm "/reports/$$filename.html"
            nice -n 19 trivy image --no-progress --scanners vuln,secret,misconfig --severity MEDIUM,HIGH,CRITICAL --format template --template "@contrib/html.tpl" -o "/reports/$$filename.html" "$$imagename"
          done
          sleep 24h
        done
    volumes:
      - ./reports:/reports
    cpus: 0.1
    depends_on:
      - docker-socket-protector
    networks:
      - internet
      - docker_socket

  docker-socket-protector:
    image: knrdl/docker-socket-protector
    hostname: docker-socket-protector
    restart: always
    read_only: true
    cap_drop: [ all ]
    environment:
      PROFILE: "trivy"  # read only access to only docker images
      LOG_REQUESTS: "false"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    networks:
      - docker_socket
    mem_limit: 100mb

networks:
  internet:
    internal: false
  docker_socket:
    internal: true
    attachable: false
```