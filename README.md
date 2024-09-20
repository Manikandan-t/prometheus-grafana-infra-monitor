# Infrastructure Monitoring with Prometheus, Grafana, and Exporters

This repository contains the setup to monitor infrastructure using Prometheus, Grafana, and several exporters for collecting and visualizing metrics. All services are deployed as Docker containers managed by Docker Compose.

## Table of Contents

- [Introduction](#introduction)
- [Components](#components)
- [Prerequisites](#prerequisites)
- [Setup and Installation](#setup-and-installation)
  - [1. Prometheus](#1-prometheus)
  - [2. Grafana](#2-grafana)
  - [3. Node Exporter](#3-node-exporter)
  - [4. DCGM Exporter](#4-dcgm-exporter)
  - [5. NVIDIA-SMI Exporter](#5-nvidia-smi-exporter)
  - [6. cAdvisor](#6-cadvisor)
  - [7. mtail](#7-mtail)
  - [8. Nginx](#8-nginx)
- [Customizing the Setup](#customizing-the-setup)
- [Monitoring Dashboards](#monitoring-dashboards)
- [License](#license)

## Introduction

This repository provides a complete solution for monitoring infrastructure using the following components:

- **Prometheus**: For collecting and querying metrics.
- **Grafana**: For building customizable dashboards and visualizations.
- **Node Exporter**: To collect hardware and OS metrics.
- **cAdvisor**: For monitoring Docker containers.
- **DCGM Exporter & NVIDIA-SMI Exporter**: To monitor GPU stats from NVIDIA GPUs.
- **mtail**: For extracting metrics from log files.
- **Nginx**: Used as a reverse proxy for specific endpoints.

## Components

1. **Prometheus**: Metrics collection and storage.
2. **Grafana**: Dashboard and visualization platform.
3. **Node Exporter**: Hardware and OS-level metrics (e.g., CPU, memory).
4. **cAdvisor**: Real-time monitoring of running containers.
5. **DCGM Exporter**: NVIDIA GPU metrics via DCGM.
6. **NVIDIA-SMI Exporter**: Metrics from NVIDIA GPUs via nvidia-smi.
7. **mtail**: For generating custom metrics from logs, such as Nginx logs.
8. **Nginx**: Acts as a reverse proxy to route traffic and logs requests.

## Prerequisites

- Docker and Docker Compose installed on your system.
- NVIDIA drivers and CUDA installed if you are monitoring GPU stats.
- An external Docker network (`monitor-net`) configured for container communication.

## Setup and Installation

To set up this monitoring stack, follow these steps:

### 1. Prometheus

Prometheus is used to scrape metrics from different services.

#### Configuration:

Prometheus is configured to scrape metrics from the following targets:
- Node Exporter
- DCGM Exporter
- NVIDIA-SMI Exporter
- mtail (for logs)
- Prometheus itself
- cAdvisor (for Docker containers)

#### Docker Compose Setup:

```yaml
version: '3.7'

services:
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
      - ./prometheus_data:/prometheus
    ports:
      - "9090:9090"
    networks:
      monitor-net:
        ipv4_address: 172.20.0.56
    restart: unless-stopped
```

### 2. Grafana

Grafana is used to visualize metrics collected by Prometheus.

#### Docker Compose Setup:

```yaml
version: '3.7'

services:
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    ports:
      - "3000:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin
    volumes:
      - ./grafana_data:/var/lib/grafana
    networks:
      monitor-net:
        ipv4_address: 172.20.0.55
    restart: unless-stopped
```

### 3. Node Exporter

Node Exporter collects hardware and OS-level metrics.

#### Docker Compose Setup:

```yaml
version: '3.7'

services:
  node_exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    ports:
      - "9100:9100"
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    networks:
      monitor-net:
        ipv4_address: 172.20.0.57
    restart: unless-stopped
```

### 4. DCGM Exporter

The DCGM exporter collects GPU metrics using the NVIDIA Data Center GPU Manager (DCGM).

#### Docker Compose Setup:

```yaml
version: '3.7'

services:
  dcgm_exporter:
    image: nvidia/dcgm-exporter:latest
    container_name: dcgm_exporter
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
    ports:
      - "9400:9400"
    networks:
      monitor-net:
        ipv4_address: 172.20.0.58
    restart: unless-stopped
```

### 5. NVIDIA-SMI Exporter

This exporter collects metrics from the `nvidia-smi` command.

#### Docker Compose Setup:

```yaml
version: '3.7'

services:
  nvidia_smi_exporter:
    image: utkuozdemir/nvidia_gpu_exporter:1.1.0
    container_name: nvidia_smi_exporter
    ports:
      - "9835:9835"
    volumes:
      - /usr/lib/x86_64-linux-gnu/libnvidia-ml.so:/usr/lib/x86_64-linux-gnu/libnvidia-ml.so
      - /usr/bin/nvidia-smi:/usr/bin/nvidia-smi
    networks:
      monitor-net:
        ipv4_address: 172.20.0.59
    restart: unless-stopped
```

### 6. cAdvisor

cAdvisor collects Docker container metrics.

#### Docker Compose Setup:

```yaml
version: '3.7'

services:
  cadvisor:
    image: google/cadvisor:latest
    container_name: cadvisor
    ports:
      - "8080:8080"
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    networks:
      monitor-net:
        ipv4_address: 172.20.0.64
    restart: unless-stopped
```

### 7. mtail

mtail reads logs (e.g., Nginx access logs) and generates custom metrics.

#### Docker Compose Setup:

```yaml
version: '3.7'

services:
  mtail:
    image: dylanmei/mtail:latest
    container_name: mtail
    ports:
      - "3903:3903"
    volumes:
      - ./nginx_logs:/var/log/nginx
      - ./prometheus:/mtail
    command: [ "-logs", "/var/log/nginx/access.log", "-progs", "/mtail" ]
    networks:
      monitor-net:
        ipv4_address: 172.20.0.63
    restart: unless-stopped
```

### 8. Nginx

Nginx is used to proxy requests and generate logs for mtail to parse.

#### Docker Compose Setup:

```yaml
version: '3.7'

services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "80:80"
    volumes:
      - ./prometheus/nginx.conf:/etc/nginx/nginx.conf
      - ./nginx_logs:/var/log/nginx
    networks:
      monitor-net:
        ipv4_address: 172.20.0.61
    restart: unless-stopped
```

## Customizing the Setup

You can modify:
- **Docker Compose** to suit your environment and needs.
- **Prometheus scrape intervals** or add more targets in `prometheus.yml`.
- **Grafana dashboards** to include additional visualizations.

Here’s the additional information for the README file:

---

## Viewing Prometheus and Grafana

- **Prometheus**: Once Prometheus is running, you can view its dashboard by navigating to `http://localhost:9090` in your browser. Here, you can query metrics and check the status of your configured targets.
  
- **Grafana**: After starting Grafana, you can access the Grafana dashboard at `http://localhost:3000`. The default login credentials are:
  - Username: `admin`
  - Password: `admin` (you can change this in the environment settings)

### Adding Data Sources in Grafana

1. Open Grafana (`http://localhost:3000`).
2. Log in with the admin credentials.
3. Go to **Configuration** > **Data Sources**.
4. Add Prometheus as a data source by entering the URL: `http://prometheus:9090`.
5. Save and test the connection.

You can now create dashboards and use Grafana’s powerful visualization tools to monitor your infrastructure metrics.

---