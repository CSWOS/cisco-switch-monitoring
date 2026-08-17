# Cisco Switch Monitoring System (Hybrid)
**Model-Driven Telemetry (Push) + SNMP + Blackbox + Prometheus + Grafana**

Designed for **Ubuntu 24.04 LTS**.

### What it monitors

| Metric | MDT Push (modern) | SNMP (legacy) | ICMP |
|--------|-------------------|---------------|------|
| Uptime / System | Yes | Yes | - |
| Interfaces | Yes | Yes | - |
| Bandwidth / Errors | Yes | Yes | - |
| CPU / Memory | Yes | Limited | - |
| Reachability | - | - | Yes |

---

## Architecture

```
Modern Cisco (IOS-XE)  -- push (gRPC) --> Telegraf (:57000)
                                              |
Legacy Cisco ---------- SNMP -------------> Prometheus (:9090) --> Grafana (:3000)
                                              |
All devices ----------- ICMP -------------> Blackbox (:9115)
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

## 2. Configure the Cisco switch (Dial-Out / Push)

On the switch, create telemetry subscriptions that push data to your Ubuntu server.

```cisco
configure terminal

! Interface statistics
telemetry ietf subscription 100
 encoding encode-kvgpb
 filter xpath /interfaces-ios-xe-oper:interfaces/interface/statistics
 source-address <SWITCH_MGMT_IP>
 stream yang-push
 update-policy periodic 3000
 receiver ip address <UBUNTU_SERVER_IP> 57000 protocol grpc-tcp

! CPU utilization
telemetry ietf subscription 101
 encoding encode-kvgpb
 filter xpath /process-cpu-ios-xe-oper:cpu-usage/cpu-utilization
 source-address <SWITCH_MGMT_IP>
 stream yang-push
 update-policy periodic 5000
 receiver ip address <UBUNTU_SERVER_IP> 57000 protocol grpc-tcp

end
write memory
```

Replace:
- `<SWITCH_MGMT_IP>` with the switch management IP
- `<UBUNTU_SERVER_IP>` with the IP of this monitoring server

Check status on the switch:
```cisco
show telemetry ietf subscription all
show telemetry ietf subscription 100 receiver
```

---

## 3. Configure the project

- **Telegraf** is already configured for dial-out (port 57000)
- **SNMP** (`snmp_exporter/snmp.yml`) → change community string if needed
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
├── telegraf/telegraf.conf          ← now listens for push on :57000
├── snmp_exporter/snmp.yml
├── blackbox_exporter/blackbox.yml
├── prometheus/prometheus.yml
├── grafana/provisioning/
├── scripts/setup-ubuntu.sh
└── README.md
```
