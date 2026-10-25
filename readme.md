# Enterprise Network Infrastructure & Automation Lab

A multi-VLAN enterprise network simulation built in Cisco Packet Tracer, integrated with automated Python configuration backups and SNMP monitoring.

## Table of Contents
1. [Overview](#1-overview)
2. [Network Architecture & Topology](#2-network-architecture--topology)
3. [Core Configurations](#3-core-configurations)
4. [Security & Access Control](#4-security--access-control)
5. [Network Automation (Python)](#5-network-automation-python)
6. [Monitoring (PRTG/Zabbix)](#6-monitoring-prtgzabbix)

---

## 1. Overview
This project simulates an enterprise-grade network infrastructure connecting multiple departmental subnets. It features L3 routing, centralized DHCP relay, security hardening via ACLs, and automated management scripts to minimize administrative overhead.

---

## 2. Network Architecture & Topology
* **VLAN Design:** Segmented into Corporate (VLAN 10), IT/Management (VLAN 20), VoIP (VLAN 30), and Guest (VLAN 99) networks.
* **Routing:** Implemented Inter-VLAN routing using a Core Layer 3 Switch and static/dynamic routing protocols.
* **DHCP Integration:** Configured **DHCP Relay (IP Helper-Address)** on routers to forward client IP requests to a central Windows Server DHCP/AD environment.

---

## 3. Core Configurations
* **VLAN & Trunking:** Configured 802.1Q trunk links between switches and assigned access ports to respective VLANs.
* **NAT/PAT:** Configured Network Address Translation on the edge router for secure internet egress.

---

## 4. Security & Access Control
* **Access Control Lists (ACLs):** Implemented extended ACLs to prevent Guest (VLAN 99) and unauthorized subnets from accessing internal domain controllers and secure databases.
* **Device Hardening:** Secured router/switch VTY lines with strong passwords and SSH access.

---

## 5. Network Automation (Python)
* Developed **Python (Netmiko)** scripts to automate administrative workflows:
  * Automatically connects to network devices via SSH.
  * Pulls and backs up running configurations to local directories with timestamps.
  * Reduced manual configuration maintenance overhead by **80%**.

---

## 6. Monitoring (PRTG / Zabbix)
* Configured **SNMP polling** across all network devices.
* Set up real-time dashboards to track interface bandwidth, CPU utilization, and device uptime.
* Configured automated alert thresholds for link failures to ensure rapid root-cause analysis.
