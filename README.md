# AI-NOC Lab: ISP MPLS L3VPN with Zabbix Monitoring

A production-style ISP core network simulation built in GNS3, featuring a full MPLS Layer 3 VPN, FortiGate-secured customer edges, and a centralized NOC monitoring stack (Zabbix, Syslog, SNMP). Built as a side project to learn and practice ISP-grade networking, monitoring, and security tools.

## Table of Contents
- [Overview](#overview)
- [Topology](#topology)
- [IP Addressing](#ip-addressing)
- [What's Working](#whats-working)
- [Build Log & Troubleshooting Highlights](#build-log--troubleshooting-highlights)
- [Security Hardening](#security-hardening)
- [Known Limitations](#known-limitations)
- [Future Work](#future-work)

## Overview

This lab simulates a small ISP core (MPLS L3VPN) connecting two customer sites — an HQ and a Branch — through a provider network, with each site secured by its own FortiGate firewall. A centralized Zabbix server monitors the network via SNMP and Syslog, giving visibility into device health and security events, the foundation for the AI-driven anomaly detection planned in later phases.

**Stack:** GNS3 (via VMware Workstation) · Cisco IOSv/IOSvL2 · FortiGate VM 7.0.5 · Ubuntu Server 20.04 · Zabbix 6.0 LTS · rsyslog · Net-SNMP

## Topology

```
                         ISP-EDGE (AS 6500)
                              |
                             P1  (MPLS Core, OSPF Area 0)
                            /   \
                         PE1     PE2
                    (HQ-CUSTOMER) (BRANCHE-CUSTOMER VRF)
                          |            |
                        CE-HQ        CE-BR
                     (AS 65200)    (AS 65200)
                          |            |
                     FortiGate-HQ  FortiGate-BR
                          |            |
                       SW-HQ        SW-BR
                       /   \            \
                 PC-HQ   Zabbix       PC-BR
                (VLAN10) (VLAN20)    (VLAN10)
```

- **Core:** OSPF underlay + MPLS/LDP + MP-BGP VPNv4 between PE1 and PE2, with per-site VRFs (`HQ-CUSTOMER`, `BRANCHE-CUSTOMER`) joined by matching route-targets.
- **Edge:** Each site's CE router peers with its PE via eBGP and connects downstream to a FortiGate firewall, which handles VLAN segmentation and LAN security.
- **Monitoring:** A Zabbix server sits on the HQ Servers VLAN, polling devices via SNMPv2c and collecting Syslog centrally.

## IP Addressing

| Segment | Range |
|---|---|
| P2P core links | `10.255.0.0/24` (/30 subnets) |
| Loopbacks | `1.1.1.1` – `6.6.6.6` |
| HQ VLAN 10 (Users) | `192.168.10.0/24` |
| HQ VLAN 20 (Servers) | `192.168.20.0/24` |
| HQ VLAN 99 (Mgmt) | `192.168.99.0/24` |
| Branch VLAN 10 (Users) | `192.168.110.0/24` |
| Branch VLAN 99 (Mgmt) | `192.168.199.0/24` |
| Zabbix server | `192.168.20.10` |

Branch has no Servers VLAN by design — HQ is the only site hosting shared infrastructure.

## What's Working

✅ **MPLS L3VPN core** — OSPF underlay, LDP label distribution with confirmed PHP, MP-BGP VPNv4 exchange between PE1/PE2, correct VRF route-target import/export across `HQ-CUSTOMER` and `BRANCHE-CUSTOMER`
✅ **PE–CE eBGP** — full route exchange, `as-override` correctly applied
✅ **FortiGate-HQ firewall edge** — VLAN-aware routing, tightened firewall policies
✅ **SSH/AAA/NTP** — configured across all core devices
✅ **Zabbix server** — Ubuntu 20.04 + Zabbix 6.0 LTS + MySQL + Apache, web UI live
✅ **Centralized Syslog** — FortiGate-HQ logs (UTM, traffic, SSH, SSL events) flowing into a dedicated log file on the Zabbix server via rsyslog
✅ **SNMP monitoring** — FortiGate-HQ, CE-HQ, and CE-BR all polled successfully and visible in Zabbix's Latest Data
✅ **Security hardening** — FortiGate policies restricted from `ALL` services to only what's needed (PING/HTTPS/SSH/SNMP/SYSLOG); management-plane ACLs restricting router VTY access to the management subnet, applied across CE-HQ, PE1, PE2, CE-BR, and P1

## Build Log & Troubleshooting Highlights

A few of the real problems hit and solved during this build (kept here because the debugging process is arguably the most valuable part of the project):

- **FortiOS 7.4.2 dataplane forwarding bug** under GNS3/QEMU — resolved by downgrading to FortiOS 7.0.5.
- **Zabbix 7.0's Ubuntu 20.04 repo had a broken package index** (`zabbix-server-mysql` and `zabbix-frontend-php` missing entirely from the repo's package list) — worked around by switching to Zabbix 6.0 LTS, which installed cleanly.
- **Host-to-lab browser access** required understanding GNS3's actual architecture: GNS3 runs inside a VMware-hosted "GNS3 VM," and lab devices run as QEMU VMs orchestrated by it. `VMnet1` is reserved for GNS3's own control channel (bridging a Cloud node to it disconnects the GNS3 GUI), and `VMnet8` (VMware's default NAT network) silently dropped bridged traffic due to its NAT behavior. Fixed by creating a dedicated custom Host-Only network (`VMnet3`, DHCP disabled) bridged via a GNS3 Cloud node directly into the Servers VLAN.
- **VRF-vs-global-table routing gap** — provider loopbacks (PE1/PE2) live in the router's *global* routing table, while the PE–CE customer-facing interfaces live inside a VRF. This means a monitoring server on the customer LAN cannot reach the provider's own loopbacks without deliberate route leaking — a real, common ISP management-plane design question, not a misconfiguration. Documented as a known limitation rather than patched under deadline pressure (see below).
- **FortiGate VM license expiration** required full node rebuilds mid-project — handled by capturing `show full-configuration` backups beforehand and reconstructing working configs from the verified IP design.

## Security Hardening

- FortiGate-HQ firewall policies restricted from `service ALL` to the minimum required set: `PING`, `HTTPS`, `SSH`, `SNMP`, `SYSLOG`.
- Management ACL (`MGMT-ACCESS`) applied to VTY lines on CE-HQ, PE1, PE2, CE-BR, and P1, permitting SSH/Telnet access only from the management subnet (`192.168.20.0/24`) and logging all denied attempts.

## Known Limitations

- **PE1 / PE2 / P1 are not SNMP-monitored.** Their loopbacks live in the global routing table, which has no return route into the customer VRFs. Fixing this requires deliberate route leaking (e.g., `redistribute connected` into the VRF address-family plus a global-table static route back, using VRF-aware next-hop syntax) — a real technique, just not implemented yet.
- **SW-HQ is not SNMP-monitored.** A management SVI was added (`192.168.99.2`) but SNMP polling currently times out; not yet diagnosed.
- **Branch-side FortiGate is not fully rebuilt** after its license expired mid-project — Branch connectivity and monitoring are pending.
- **No HA, QoS, or automated remediation** implemented yet — see Future Work.

## Future Work

Ideas for extending this lab further:

- Route-leaking fix for full core SNMP visibility
- High-availability / redundancy testing
- QoS policy implementation
- **Telemetry pipeline:** Python (pysnmp, netmiko, pandas) pulling structured metrics from Zabbix/devices
- **Automated remediation:** Ansible/Netmiko closed-loop response to detected faults, measuring MTTR improvement over manual intervention
- **Visualization:** Grafana dashboards on top of Zabbix data
- **Security inspection:** Suricata/Scapy for deeper traffic analysis

---

*Built by Jwamer, Information System Engineering student at Erbil Polytechnic University, as a side project to figure out more stuff and learn new protocols and tools.*