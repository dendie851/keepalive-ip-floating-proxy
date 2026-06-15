# 📘 Keepalived + HAProxy: High Availability & Load Balancing Architecture

---

## 📌 Opening / Pembukaan

**"One server is not enough. One gateway is a single point of failure."**

In the modern digital world, websites and applications must be **always available**. If a server goes down, users lose access, and businesses lose money. To solve this problem, we need a system that can:

1. **Distribute traffic** across multiple servers so no single server gets overloaded.
2. **Automatically switch** to a backup server if the main server fails.

This project demonstrates a **production-ready architecture** using three powerful technologies:

| Technology | Role |
|------------|------|
| **Keepalived** | Provides a **floating (virtual) IP** that moves between servers when one fails |
| **HAProxy** | Acts as a **load balancer** that distributes requests across multiple web servers |
| **Nginx** | Serves as the **backend web server** nodes |

Together, they create a **highly available, scalable, and fault-tolerant** system.


---

## 1️⃣ Concept Introduction: IP Floating, Keepalived & HAProxy

### 🧠 What is a Floating IP?

A **floating IP** (or Virtual IP / VIP) is an IP address that is **not permanently attached** to one physical server. Instead, it "floats" between servers in a cluster. Only one server holds the VIP at any given time. When that server fails, another server automatically takes over the VIP.

Think of it like a **taxi stand**: Customers wait at a fixed location (the VIP). When one taxi (server) leaves, another taxi immediately pulls up to the same spot to serve the next customer.

### 🔁 What is Keepalived?

**Keepalived** is software that implements the **VRRP (Virtual Router Redundancy Protocol)**. It allows multiple servers to work together as a **high-availability cluster**.

Key components in Keepalived:

| Component | Description |
|-----------|-------------|
| **MASTER** | The primary server that currently holds the VIP |
| **BACKUP** | The standby server(s) ready to take over if MASTER fails |
| **VRRP Instance** | A group of servers that share a Virtual Router ID and compete for the VIP |
| **Priority** | A number (higher = more likely to be MASTER). MASTER=101, BACKUP=100 |
| **Advert Interval** | How often (in seconds) MASTER broadcasts "I'm alive" messages |
| **Virtual IPaddress** | The floating IP that moves between MASTER and BACKUP |
| **Track Script** | A script that checks if HAProxy is running. If not, priority decreases |

### ⚖️ What is HAProxy?

**HAProxy (High Availability Proxy)** is a fast, reliable **load balancer** and **reverse proxy**. It sits in front of backend servers and distributes incoming traffic.

Key HAProxy concepts in this project:

| Concept | Description |
|---------|-------------|
| **Frontend** | The entry point where clients connect (port 8080) |
| **Backend** | The pool of backend servers (Nginx nodes) |
| **Round Robin** | Traffic is distributed equally to each server in turn |
| **Health Check** | HAProxy regularly checks if each backend server is healthy |
| **Stats Dashboard** | A web UI to monitor traffic and server status (port 8404) |

---

## 2️⃣ Architecture Explanation (design/aristektur.png)

The architecture consists of **7 Docker containers** organized into 3 groups:

```
┌──────────────────────────────────────────────────────────┐
│                    FLOATING VIP                           │
│                  192.168.50.200                           │
└──────────────────────────────────────────────────────────┘
                              │
         ┌────────────────────┴────────────────────┐
         │                                         │
┌──────────────────┐                    ┌──────────────────┐
│ Keepalived-1     │                    │ Keepalived-2     │
│   (MASTER)       │◄──────────────────►│   (BACKUP)       │
│ Priority: 101    │    VRRP Protocol   │ Priority: 100    │
│ 10.10.0.10       │                    │ 10.10.0.20       │
├──────────────────┤                    ├──────────────────┤
│ HAProxy Master   │                    │ HAProxy Backup   │
│ Port: 8080       │                    │ Port: 8088       │
│ Stats: 8404      │                    │ Stats: 8405      │
└────────┬─────────┘                    └────────┬─────────┘
         │                                        │
         └──────────────┬─────────────────────────┘
                        │ Load Balances To
               ┌────────┴────────┐
               │                 │
        ┌──────┴──────┐  ┌──────┴──────┐
        │  Nginx       │  │  Nginx       │
        │  Node 1      │  │  Node 2      │
        │  Port 8081   │  │  Port 8082   │
        └─────────────┘  └─────────────┘
                                 │
                          ┌──────┴──────┐
                          │  Nginx       │
                          │  Node 3      │
                          │  Port 8083   │
                          └─────────────┘
```

### 🏗️ Architecture Flow

1. **Client** accesses the application at `http://192.168.50.200:8080` (Floating IP)
2. The request reaches either **Keepalived-1 (MASTER)** or **Keepalived-2 (BACKUP)**, depending on who currently holds the VIP
3. The **HAProxy** running inside that container receives the request
4. HAProxy applies the **Round Robin** algorithm to forward the request to one of the **3 Nginx nodes**
5. HAProxy **monitors health** of each node every 2 seconds. If a node fails, it is removed from the pool
6. The Nginx node responds with a custom HTML page (e.g., "This is Web Server from NODE 1")

### 📋 Key Configuration Details

| Component | Parameter | Value | Purpose |
|-----------|-----------|-------|---------|
| **Keepalived MASTER** | state | MASTER | Primary server for VIP |
| | priority | 101 | Higher = more likely to be MASTER |
| **Keepalived BACKUP** | state | BACKUP | Standby server |
| | priority | 100 | Lower = backup |
| **VIP** | virtual_ipaddress | 192.168.50.200/24 | The floating IP |
| **VRRP** | virtual_router_id | 51 | Unique group ID |
| | advert_int | 1 | Advertise every 1 second |
| **Health Check** | interval | 2 seconds | Checks HAProxy every 2s |
| | weight | 2 | Priority adjustment on failure |
| **HAProxy** | balance | roundrobin | Even distribution |
| | httpchk | GET / | HTTP health check |
| **Nginx Nodes** | 3 servers | 8081, 8082, 8083 | Backend web servers |

---

## 3️⃣ Evidence Screenshots Explanation (ss/ folder)

### 📸 Screenshot 1: `ss/1-deploy-node.png`

This screenshot shows the **deployment of the 3 Nginx backend nodes**:

```
> docker-compose -f .\node/docker-compose.yaml up -d
```

Containers created:
- `nginx_node1` → port **8081**
- `nginx_node2` → port **8082**
- `nginx_node3` → port **8083**

Each container runs Nginx with Alpine Linux, and automatically creates a custom HTML page identifying which node it is.

### 📸 Screenshot 2: `ss/2-deploy-node-test.png`

This screenshot shows **testing each Nginx node individually** using `curl`:

- `curl http://localhost:8081` → Response: "This is Web Server from NODE 1" ✅
- `curl http://localhost:8082` → Response: "This is Web Server from NODE 2" ✅
- `curl http://localhost:8083` → Response: "This is Web Server from NODE 3" ✅

All three backend servers are running and responding correctly.

### 📸 Screenshot 3: `ss/3-deploy-node-keepalived-haproxy.png`

This screenshot shows the **deployment of HAProxy + Keepalived cluster**:

```
> docker-compose -f .\keepalived-1\docker-compose.yaml up -d
> docker-compose -f .\keepalived-2\docker-compose.yaml up -d
```

Containers created:
- `keepalived_master` + `haproxy_master` (IP: 10.10.0.10 & 10.10.0.11)
- `keepalived_backup` + `haproxy_backup` (IP: 10.10.0.20 & 10.10.0.21)

After deployment, accessing the VIP `192.168.50.200:8080` routes traffic through HAProxy, which load balances across the 3 Nginx nodes.

---

## 4️⃣ Benefits for Scalability: Failover & Load Balancing

### 🛡️ Failover (High Availability)

**What it is:** The ability to automatically switch to a backup system when the primary system fails.

**How this project implements it:**

```
If MASTER (Keepalived-1) fails:
  ├── BACKUP (Keepalived-2) detects MASTER is gone (no VRRP advertisement)
  ├── BACKUP promotes itself to MASTER
  ├── VIP 192.168.50.200 moves to the BACKUP server
  └── HAProxy Backup starts serving traffic ─── No downtime! ✨
```

**Benefits:**
| Benefit | Description |
|---------|-------------|
| **Zero Downtime** | Users never experience interruption |
| **Automatic** | No manual intervention needed |
| **Fast Recovery** | Failover happens in seconds (advert_int = 1s) |
| **Transparent** | Users connect to the same IP, unaware of any failure |
| **Health Monitoring** | Keepalived also checks if HAProxy is alive. If HAProxy dies, VIP moves |

### ⚖️ Load Balancing (Scalability)

**What it is:** Distributing incoming traffic across multiple backend servers to prevent any single server from becoming overloaded.

**How this project implements it:**

```
Client Request → HAProxy → Round Robin → Nginx Node 1 (33%)
                                         → Nginx Node 2 (33%)
                                         → Nginx Node 3 (33%)
```

**Benefits:**
| Benefit | Description |
|---------|-------------|
| **Even Distribution** | Round Robin ensures each server gets equal traffic |
| **Horizontal Scaling** | Add more Nginx nodes easily (just add new lines in haproxy.cfg) |
| **Health Checks** | Unhealthy servers are automatically removed from the pool |
| **Increased Capacity** | Can handle more concurrent users by adding more nodes |
| **Flexibility** | Different load balancing algorithms available (roundrobin, leastconn, source) |

### 📊 Comparison: Without vs With This Architecture

| Scenario | Without Architecture | With This Architecture |
|----------|--------------------|-----------------------|
| **Server Failure** | ❌ Website down for hours | ✅ Automatic failover in seconds |
| **Traffic Spike** | ❌ Server overload, slow response | ✅ Traffic distributed across nodes |
| **Maintenance** | ❌ Need downtime to update | ✅ Update nodes one at a time (rolling update) |
| **Scalability** | ❌ Need bigger server (vertical scaling) | ✅ Just add more nodes (horizontal scaling) |
| **Single Point of Failure** | ❌ One server = one point of failure | ✅ Redundant HAProxy + Keepalived |

---

## 5️⃣ Quick Reference: Complete Infrastructure List

| # | Component | Container Name | IP Address | Port | Role |
|---|-----------|---------------|------------|------|------|
| 1 | **Keepalived-1** | keepalived_master | 10.10.0.10 | - | VRRP MASTER (priority 101) |
| 2 | **HAProxy-1** | haproxy_master | 10.10.0.11 | 8080, 8404 | Load Balancer Primary |
| 3 | **Keepalived-2** | keepalived_backup | 10.10.0.20 | - | VRRP BACKUP (priority 100) |
| 4 | **HAProxy-2** | haproxy_backup | 10.10.0.21 | 8088, 8405 | Load Balancer Backup |
| 5 | **Nginx Node 1** | nginx_node1 | - | 8081 | Backend Web Server |
| 6 | **Nginx Node 2** | nginx_node2 | - | 8082 | Backend Web Server |
| 7 | **Nginx Node 3** | nginx_node3 | - | 8083 | Backend Web Server |
| - | **Floating VIP** | - | **192.168.50.200** | 8080 | Virtual IP for Clients |

### 🔐 Authentication
| Parameter | Value |
|-----------|-------|
| VRRP Auth Type | PASS |
| VRRP Auth Password | RahasiaC |

### 📈 Monitoring
| Service | URL | Description |
|---------|-----|-------------|
| HAProxy Stats (Master) | http://localhost:8404 | Real-time traffic dashboard |
| HAProxy Stats (Backup) | http://localhost:8405 | Backup dashboard |

### 🚀 Deployment Commands
```bash
# Step 1: Deploy backend Nginx nodes
docker-compose -f node/docker-compose.yaml up -d

# Step 2: Deploy Keepalived-1 + HAProxy Master
docker-compose -f keepalived-1/docker-compose.yaml up -d

# Step 3: Deploy Keepalived-2 + HAProxy Backup
docker-compose -f keepalived-2/docker-compose.yaml up -d

# Access the application
curl http://192.168.50.200:8080
```

---

## 🎯 Conclusion

This project demonstrates a **production-grade architecture** that combines:
- **Keepalived** for **high availability** through floating IP and automatic failover
- **HAProxy** for **load balancing** across multiple backend servers
- **Nginx** as lightweight **web server nodes**

The result is a system that is **resilient** (survives server failures), **scalable** (add more nodes easily), and **reliable** (always available to users). This architecture is widely used in production environments for critical applications requiring **99.99% uptime**.