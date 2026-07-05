# High-Availability & Core Infrastructure Labs

A comprehensive, production-grade infrastructure lab simulating scalable, resilient, and clustered network services using Docker, Nginx, HAProxy, Keepalived, and BIND9.

This repository demonstrates edge routing, high availability (HA) at multiple layers, failover automation, and master-slave zone replication.

---

## 🏗️ Architecture Overview

The entire lab is containerized and orchestrated via a highly resilient network topology:

1. **DNS Layer (BIND9)**: Master-Slave DNS architecture with automatic zone transfers (AXFR).
2. **Edge Load-Balancing (HAProxy + Keepalived)**: Active-Backup ingress nodes handling SSL Termination and routing requests to the web cluster via a shared Virtual IP (VIP).
3. **Web Application Layer (Nginx + Keepalived)**: Master-Master (Active-Active) web cluster processing requests simultaneously with independent health checking.

---

## 📁 Repository Structure

```text
infrastructure-labs/
├── docker-compose.yml         # Core multi-container orchestration
├── ex1/                       # Exercise 1: Nginx Active-Active Cluster
│   └── configs/               # Keepalived Node A/B & Nginx configurations
├── ex2/                       # Exercise 2: HAProxy SSL Termination
│   ├── certs/                 # Unified SSL certificates (.pem)
│   ├── configs/               # HAProxy frontend/backend configuration
│   └── generate-certs.sh      # Automated self-signed SSL generator
├── ex3/                       # Exercise 3: HAProxy Active-Backup Cluster
│   ├── configs/               # Keepalived configurations for HAProxy
│   └── Dockerfile.haproxy_ha  # Custom co-located Alpine image (HAProxy + Keepalived)
└── ex4/                       # Exercise 4: BIND9 DNS Master-Slave
    └── configs/               # Named configurations and zone files

```

---

## 🛠️ Lab Detailed Exercises

### Exercise 1: Nginx Master-Master Cluster via Keepalived

* **Implementation**: Deployed two independent Nginx nodes sharing two Virtual IPs (VIPs).
* **Core Strategy**: Avoided split-brain and allowed non-local binding by tuning core kernel parameters (`net.ipv4.ip_nonlocal_bind = 1`).
* **Failover**: Automated via health-check scripts monitoring the Nginx PID lifecycle.

### Exercise 2: SSL Termination using HAProxy

* **Implementation**: Terminated incoming TLS/SSL traffic (Port 443) at the HAProxy layer.
* **Mechanism**: Combined private keys and certificates into unified `.pem` blocks. Traffic is safely decrypted at edge and passed down as plain HTTP (Port 80) to backend servers to eliminate crypto-overhead on applications.
* **Traffic Safety**: Enforced strict HTTP-to-HTTPS redirection (301 Moved Permanently).

### Exercise 3: HAProxy Active-Backup Infrastructure

* **Implementation**: Mitigated single-point-of-failure (SPOF) at the load-balancer layer.
* **Advanced Architecture**: To bypass kernel namespace initialization race conditions inside standard containers, a custom co-located Alpine image was developed, running both HAProxy and Keepalived daemons natively inside the same network namespace.

### Exercise 4: DNS Master-Slave Zone Replication (BIND9)

* **Implementation**: Deployed an authoritative Master BIND9 DNS and a Slave DNS.
* **Automation**: Enabled instantaneous `also-notify` triggers and secure `allow-transfer` restrictions. Zone files utilize dynamic serial versions (`YYYYMMDDNN`) to enforce atomic updates across nodes.

---

## 🚦 Verification & Testing

### 1. Provisioning the entire Lab

Bring up the multi-layered infrastructure with a single atomic command:

```bash
docker compose up --build -d

```

### 2. Validating SSL Termination & Redirection

```bash
# Check 301 Redirect
curl -kI http://localhost

# Check HTTPS Payload Distribution
curl -k https://localhost

```

### 3. Testing Load-Balancer Failover

Simulate a catastrophic hardware or daemon failure on the active edge node:

```bash
docker stop haproxy-active-a

```

*Result: Keepalived immediately re-assigns the Virtual IP (`172.20.0.100`) to `haproxy-backup-b`. Traffic continues to flow with zero downtime.*

### 4. Verifying DNS Zone Transfer

Execute a direct query targeting the Slave DNS engine to confirm successful AXFR replication:

```bash
docker run --rm --network infrastructure-labs_cluster_net alpine apk add --no-cache bind-tools && dig @172.20.0.201 www.mysite.local +short

```

---

## 🔑 Key Engineering Takeaways

* **Race Conditions**: Solved container networking runtime failures (`OCI runtime create failed`) by migrating multi-daemon dependencies into unified namespaces.
* **Real-World Parity**: Realized full cluster topology on local virtualized sandboxes using Docker capabilities (`NET_ADMIN`) for low-level virtual network interface management.

```

```
