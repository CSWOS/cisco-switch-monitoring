# Cisco Switch Monitoring System (Hybrid)
**gNMI (Telegraf) + SNMP + Blackbox + Prometheus + Grafana**

Designed for **Ubuntu 24.04 LTS**.

### What it monitors

| Metric | gNMI (modern) | SNMP (legacy) | ICMP |
|--------|---------------|---------------|------|
| Uptime / System | Yes | Yes | - |
| Interfaces | Yes (rich) | Yes | - |
| Bandwidth / Errors | Yes | Yes | - |
| BGP neighbors | Yes (OpenConfig) | Possible | - |
| OSPF | Yes (OpenConfig) | Possible | - |
| CPU / Memory | Yes | Limited | - |
| Reachability | - | - | Yes |

---

## Architecture

```
Modern Cisco (IOS-XE / NX-OS) -- gNMI --> Telegraf (:9273)
                                         |
Legacy Cisco ---------------- SNMP ----> Prometheus (:9090) --> Grafana (:3000)
                                         |
All devices ----------------- ICMP ----> Blackbox (:9115)
```

---

## 1. Prerequisites (Ubuntu 24.04)

```bash
cd /opt/cisco-switch-monitoring
chmod +x scripts/setup-ubuntu.sh
./scripts/setup-ubuntu.sh

# Then log out and log back in (or: newgrp docker)
```

---

## 2. Enable gNMI on Cisco devices

> **Note:** On modern IOS-XE the old `gnmi-yang` command has been replaced by the unified **`gnxi`** command set.

### IOS-XE (Catalyst 9000 / modern platforms)

**Lab / Non-secure mode (easiest for testing):**
```cisco
configure terminal
gnxi
gnxi server
! Optional: change port (default is usually 50052)
! gnxi port 50051
username monitoring privilege 15 secret YourStrongPassword
end
write memory
```

**Secure mode (recommended for production):**
```cisco
configure terminal
gnxi
gnxi secure-server
! or for self-signed:
! gnxi secure-init
username monitoring privilege 15 secret YourStrongPassword
end
write memory
```

Check status:
```cisco
show gnxi state
show gnxi state detail
```

### NX-OS (Nexus)

```cisco
configure terminal
feature gnmi
feature grpc
username monitoring password YourStrongPassword role network-admin
end
copy running-config startup-config
```

---

## 3. Configure the project

- **Telegraf** (`telegraf/telegraf.conf`) → put real switch IPs + credentials  
  (use the correct port: 50051, 50052 or 9339 depending on your config)
- **SNMP** (`snmp_exporter/snmp.yml`) → change community string
- **Prometheus** (`prometheus/prometheus.yml`) → put switch IPs for SNMP + ICMP jobs

---

## 4. Start the stack

```bash
docker compose up -d
```

### Access

| Service | URL | Credentials |
|---------|-----|-------------|
| Grafana | http://SERVER_IP:3000 | admin / admin |
| Prometheus | http://SERVER_IP:9090 | - |
| Telegraf metrics | http://SERVER_IP:9273/metrics | - |

---

## Project structure

```
cisco-switch-monitoring/
├── docker-compose.yml
├── telegraf/telegraf.conf
├── snmp_exporter/snmp.yml
├── blackbox_exporter/blackbox.yml
├── prometheus/prometheus.yml
├── grafana/provisioning/
├── scripts/setup-ubuntu.sh
└── README.md
```
