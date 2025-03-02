
This project demonstrates how to set up an Nginx server as both a reverse proxy and load balancer. It distributes incoming HTTP requests between two backend servers (with hidden IP addresses) and provides a monitoring endpoint and custom logging to track which backend server handled each request.

## Table of Contents

- Overview
- Prerequisites
- Project Structure
- Nginx Basics
- Configuration Details
    - Global Configuration (`nginx.conf`)
    - Site Configuration (`default` in sites-enabled)
- Commands & Testing
- Monitoring & Logs
- Summary

## Overview

This setup allows a local Ubuntu server to act as a gateway for incoming requests by:

- **Reverse Proxying:** Accepting client requests and forwarding them to backend servers.
- **Load Balancing:** Distributing requests between multiple backend servers using a round-robin mechanism.
- **Monitoring:** Using the `stub_status` module to monitor active connections and performance metrics.
- **Custom Logging:** Recording which backend server (using hidden IP addresses) handled each request via a custom log format.

## Prerequisites

- Ubuntu server with Nginx installed.
- Two backend servers running a web server (Nginx, Apache, etc.) serving your HTML page.
- Basic knowledge of Linux command-line operations.

## Project Structure

The key Nginx configuration files used are:

```bash
/etc/nginx/
├── nginx.conf            # Global configuration
├── sites-available/
│   └── default           # Site-specific configuration for reverse proxy/load balancing
└── sites-enabled/
    └── default           # Symlink to sites-available/default
```

## Nginx Basics

Nginx is a powerful web server used for:

- Serving static and dynamic content.
- Acting as a reverse proxy to hide backend server details.
- Load balancing incoming requests to multiple servers.
- Monitoring performance via modules like `stub_status`.

### Key Directives

- **upstream:** Groups backend servers for load balancing.
- **server:** Defines a virtual host.
- **location:** Specifies how to handle requests for specific URIs.
- **proxy_pass:** Forwards requests to the backend.
- **stub_status:** Provides basic server status information for monitoring.

## Configuration Details

### Install Nginx

If Nginx is not already installed, you can install it using:

```bash
sudo apt update
sudo apt install nginx
```

### Global Configuration (`nginx.conf`)

This file sets up general server settings, including a custom log format that logs which backend handled each request.

```bash
sudo nano /etc/nginx/nginx.conf
```

```bash
user www-data;
worker_processes auto;
pid /run/nginx.pid;
error_log /var/log/nginx/error.log;
include /etc/nginx/modules-enabled/*.conf;

events {
    worker_connections 768;
}

http {
    # Basic Settings
    sendfile on;
    tcp_nopush on;
    types_hash_max_size 2048;
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # SSL Settings (adjust as needed)
    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    # Custom Logging Settings
    log_format upstreamlog '$remote_addr - $remote_user [$time_local] '
                           '"$request" $status $body_bytes_sent '
                           '"$http_referer" "$http_user_agent" '
                           'upstream: "$upstream_addr" '
                           'request_time: $request_time '
                           'upstream_response_time: $upstream_response_time';
    access_log /var/log/nginx/access.log upstreamlog;

    # Gzip Compression
    gzip on;

    # Include Additional Configuration Files
    include /etc/nginx/conf.d/*.conf;
    include /etc/nginx/sites-enabled/*;
}
```

### Site Configuration (`/etc/nginx/sites-enabled/default`)

This file defines the reverse proxy and load balancing configuration along with a monitoring endpoint. The backend IP addresses are hidden (represented as placeholders).

```bash
sudo nano /etc/nginx/sites-enabled/default
```

```nginx
# Define the backend servers (hidden IPs)
upstream backend {
    server 192.0.2.1;  # Backend Server 1 (hidden IP)
    server 192.0.2.2;  # Backend Server 2 (hidden IP)
}

server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;

    # Monitoring endpoint using stub_status
    location /nginx_status {
        stub_status;
        allow 127.0.0.1;  # Restrict access to local machine; adjust as needed
        deny all;
    }

    # Reverse Proxy and Load Balancing
    location / {
        proxy_pass http://backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## Commands & Testing

### 1. Test Nginx Configuration

Before reloading Nginx, check for syntax errors:

```bash
sudo nginx -t
```

### 2. Reload Nginx

If the configuration test passes, reload Nginx to apply the changes:

```bash
sudo systemctl reload nginx
```

### 3. Generate a New Request

To ensure your requests are not cached (which may skip the proxy logic), use:

```bash
curl -H "Cache-Control: no-cache" http://<YOUR_LOCAL_SERVER_IP>/
```

## Monitoring & Logs

### View Nginx Status

Access the monitoring endpoint from the local machine:

```bash
curl http://127.0.0.1/nginx_status
```

This will display active connections and other metrics provided by Nginx.

### Check the Logs

To monitor which backend server handled each request, tail the access log:

```bash
sudo tail -f /var/log/nginx/access.log
```

![Screenshot 2025-03-02 033258](https://github.com/user-attachments/assets/1d5586b1-45fc-45b7-a1f4-e07d5b491557)


In the log entries, look for the `upstream: "..."` field which shows the backend IP (e.g., `"192.0.2.1"` or `"192.0.2.2"`).

## Summary

- **Reverse Proxy & Load Balancing:**  
    Incoming requests to your local server are forwarded to one of the backend servers (hidden IPs) using the `upstream` block. Load balancing is done using a round-robin mechanism.
    
- **Monitoring:**  
    The `/nginx_status` endpoint provides real-time metrics about Nginx performance.
    
- **Custom Logging:**  
    The custom log format in `nginx.conf` includes the `$upstream_addr` variable so you can see which backend server processed each request.
    
- **Testing & Deployment:**  
    Use the provided commands (`nginx -t`, `systemctl reload nginx`, `curl`, and `tail`) to test and monitor your configuration.
