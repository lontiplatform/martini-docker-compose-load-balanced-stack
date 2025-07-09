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

## TLS Certificate Generation (for Production Domains)

If you're using a real domain and need a trusted certificate from a Certificate Authority (e.g., GoDaddy, DigiCert, Let's Encrypt), follow these steps:

### Step 1: Generate a Private Key and CSR

Run these commands in your terminal (replace `yourdomain.com`):

**Private key:** 
```bash
openssl genpkey -algorithm RSA -out yourdomain.com.key -pkeyopt rsa_keygen_bits:2048
```

**CSR:**
```bash
openssl req -new -key test2.homemed.com.key -out test2.homemed.com.csr
```

* `yourdomain.com.key`: Your private key (keep this safe!)
* `yourdomain.com.csr`: Certificate Signing Request (send this to the CA)

### Step 2: Buy and Submit CSR to a Certificate Authority

* Go to your preferred CA (e.g., Sectigo, DigiCert, ZeroSSL)
* Choose the certificate you want (e.g., single-domain, wildcard)
* During purchase, you will be prompted to **upload the `.csr` file**
* Complete domain validation steps (email, DNS, or HTTP-based)

### Step 3: Receive Your Certificate from the CA

You'll typically receive one or more of the following:

* `yourdomain.com.crt` 
* `ca_bundle.crt` or `intermediate.crt` 

### Step 4: Create the `.pem` File for HAProxy

HAProxy expects a `.pem` file that **includes your cert and the chain**, in the following order:

```bash
cat yourdomain.com.crt ca_bundle.crt > certs/yourdomain.com.pem
```

Make sure the resulting file `certs/yourdomain.com.pem` contains:

1. Your certificate (`-----BEGIN CERTIFICATE-----`)
2. The full chain of intermediate certificates
3. (Optional) The root certificate, if provided

Ensure the corresponding private key file (`certs/yourdomain.com.key`) matches the certificate.

### Step 5: Mount Files into HAProxy Container

Make sure your `docker-compose.yml` includes the volume mount:

```yaml
  haproxy:
    ...
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
      - ./certs/yourdomain.com.pem:/usr/local/etc/haproxy/certs/yourdomain.com.pem:ro
      - ./certs/yourdomain.com.key:/usr/local/etc/haproxy/certs/yourdomain.com.key:ro
```

> **HAProxy uses these files to terminate SSL on port 443.**

## Self-Signed Certificate (for Development or Testing)

If you're testing locally or don't need a trusted certificate, you can generate a **self-signed certificate** to use with HAProxy.

> Browsers will show a **security warning** when using a self-signed cert. This is expected in development environments.

### Step 1: Generate the Certificate and Key

Run this command to generate a self-signed cert valid for 365 days:

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout certs/localhost.key -out certs/localhost.pem -days 365 \
  -subj "/C=US/ST=Dev/L=Local/O=LocalDev/CN=localhost"
```

This creates:

* `certs/localhost.key`: Private key
* `certs/localhost.pem`: Self-signed certificate

### Step 2: Mount Files into HAProxy Container

In your `docker-compose.yml`, make sure these files are mounted:

```yaml
  haproxy:
    ...
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro
      - ./certs/localhost.pem:/usr/local/etc/haproxy/certs/localhost.pem:ro
      - ./certs/localhost.key:/usr/local/etc/haproxy/certs/localhost.key:ro
```

Update the HAProxy config to use `localhost.pem` and `localhost.key`.

## Switching Between Certs

You can easily swap between production and self-signed certs by:

* Editing the filenames in `haproxy.cfg`
* Updating the volume mounts in `docker-compose.yml`
* Restarting the stack

Use the production `.pem` and `.key` for real domains, and the `localhost.pem` and `localhost.key` when testing locally.

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

