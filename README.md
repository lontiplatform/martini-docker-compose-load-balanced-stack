# Martini Docker Compose Load Balanced Stack

This repository provides a pre-configured [Docker Compose](https://docs.docker.com/compose/) setup to run **Martini Runtime** containers behind an **HAProxy** load balancer. It enables scalable, SSL-secured deployment with centralized access and health-checking.

## Stack Overview

### Components

- **Martini Runtime**
  - Docker image: `lontiplatform/martini-server-runtime:latest`
  - Default: 2 replicas
  - Mounted volumes for logs, packages, and database connection pools
  - Requires the `MR_LICENSE` environment variable

- **HAProxy**
  - Docker image: `haproxy:2.9`
  - Load balances traffic to Martini instances over HTTP/HTTPS
  - Provides a stats dashboard on port `8404`
  - SSL termination enabled using `.pem` certificate and key

## File Structure

```plaintext
├── docker-compose.yml          # Main Compose file to spin up the stack
├── haproxy.cfg                 # HAProxy configuration
├── martini_data/               # Local persistent data for Martini
│   ├── db_pool/
│   ├── logs/
│   └── packages/
└── certs/                      # TLS certificate and key
├── yourdomain.com.pem
└── yourdomain.com.key
```

## Usage

### 1. Prerequisites

- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)
- Valid `.pem` and `.key` files in the `certs/` directory
- Set the `MR_LICENSE` environment variable

### 2. Start the Stack

```bash
docker compose up -d 
```

Or if you want your Martini service to scale more than 2, you can use the command:

```bash
docker compose up -d --scale martini=<number of martini instances>
```

This will:

* Start 2 Martini containers with 2 vCPUs and 4GB RAM each
* Start 1 HAProxy container with 1 vCPU and 2GB RAM
* Expose the following ports:

  * `80`: HTTP (redirects to HTTPS)
  * `443`: HTTPS
  * `8404`: HAProxy stats interface

### 3. Access Martini

Navigate to `https://localhost` (or the hostname you configured in your certificate).

## HAProxy Monitoring

Access HAProxy statistics dashboard:

```
http://localhost:8404/stats
Username: admin
Password: admin
```

## Configuration Details

### docker-compose.yml

* Uses Docker **bridge** networking under `martini_network`
* Martini volumes:

  * `db_pool`: `/data/conf/db-pool`
  * `logs`: `/data/logs`
  * `packages`: `/data/packages`
* HAProxy mounts:

  * `haproxy.cfg` (read-only)
  * TLS cert and key (read-only)

### haproxy.cfg

* SSL enabled on port 443
* HTTP redirect to HTTPS
* Uses `server-template` with dynamic resolution of Martini containers
* Built-in health checking (`httpchk GET /`, expect 200 OK)


## TLS Setup

Ensure the following files are placed in the `certs/` directory:

* `yourdomain.com.pem`: TLS certificate (chain if necessary)
* `yourdomain.com.key`: Private key

HAProxy will use these to terminate HTTPS traffic.

## Cleanup

To stop and remove all containers:

```bash
docker compose down
```

