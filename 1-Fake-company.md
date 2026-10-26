# Corporate Network Setup & Troubleshooting Guide

## 1. Overview & Department Information

### Departments
| Department | # People | # Machines | Core Devices & Details |
| :--- | :--- | :--- | :--- |
| **HR/Finance** | 8 | 18 | 8 PCs, 8 IP Phones, 2 Network Printers |
| **Engineering** | 35 | 72 | 35 PCs/Laptops, 35 IP Phones, 2 Test Workstations |
| **Sales/Marketing** | 22 | 45 | 22 Laptops/PCs, 22 IP Phones, 1 Network Printer |
| **IT Support/Admin** | 5 | 11 | 5 Management PCs, 5 IP Phones, 1 Admin Terminal |
| **Server Farm** | - | 6 | Active Directory/DNS, DHCP, Web, FTP, Database, Syslog |
| **Guests** | - | approx. 20 | Temporary Wireless Devices (Smartphones, Laptops) |
| **Total** | **70** | **172** | - |

---

### VLAN Identification Table
| Department / Role | Data VLAN ID | Voice VLAN ID |
| :--- | :--- | :--- |
| **HR / Finance** | VLAN 10 | VLAN 110 |
| **Engineering** | VLAN 20 | VLAN 120 |
| **Sales / Marketing** | VLAN 30 | VLAN 130 |
| **IT Support / Admin** | VLAN 40 | VLAN 140 |
| **Server Farm** | VLAN 50 | - |
| **Guests** | VLAN 99 | - |

---

### VLSM Table
> **IP Assigning Strategy:** Total required IPs are 172. Using `192.168.0.0/24` and `192.168.1.0/24` network blocks, VLSM is applied to isolate broadcast domains per department and maximize address utilization.

| Subnet Name | VLAN ID | Required Hosts | Subnet Mask | Network Address | Usable IP Range | Gateway IP |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **ENG-Data** | 20 | 37 | /26 (255.255.255.192) | 192.168.0.0 | 192.168.0.1 ~ 192.168.0.62 | 192.168.0.1 |
| **ENG-Voice** | 120 | 35 | /26 (255.255.255.192) | 192.168.0.64 | 192.168.0.65 ~ 192.168.0.126 | 192.168.0.65 |
| **S/M-Data** | 30 | 23 | /27 (255.255.255.224) | 192.168.0.128 | 192.168.0.129 ~ 192.168.0.158 | 192.168.0.129 |
| **S/M-Voice** | 130 | 22 | /27 (255.255.255.224) | 192.168.0.160 | 192.168.0.161 ~ 192.168.0.190 | 192.168.0.161 |
| **Guests** | 99 | 20 | /27 (255.255.255.224) | 192.168.0.192 | 192.168.0.193 ~ 192.168.0.222 | 192.168.0.193 |
| **HR-Data** | 10 | 10 | /28 (255.255.255.240) | 192.168.0.224 | 192.168.0.225 ~ 192.168.0.238 | 192.168.0.225 |
| **HR-Voice** | 110 | 8 | /28 (255.255.255.240) | 192.168.0.240 | 192.168.0.241 ~ 192.168.0.254 | 192.168.0.241 |
| **Server Farm** | 50 | 6 | /29 (255.255.255.248) | 192.168.1.0 | 192.168.1.1 ~ 192.168.1.6 | 192.168.1.1 |
| **IT-Data** | 40 | 6 | /29 (255.255.255.248) | 192.168.1.8 | 192.168.1.9 ~ 192.168.1.14 | 192.168.1.9 |
| **IT-Voice** | 140 | 5 | /29 (255.255.255.248) | 192.168.1.16 | 192.168.1.17 ~ 192.168.1.22 | 192.168.1.17 |

---

## 2. Network Topology & Implementation Steps

1. **상단 (인터넷 영역):** Router (2911) 1대배치 (외부 ISP 연결용) 및 External Server 1대
2. **중앙 (백본 컨트롤러):** Multilayer Switch (3560) 1대 (전체 Inter-VLAN 라우팅 및 DHCP Relay 담당)
3. **하단 (액세스 계층):** L2 Switch (2960) 각 부서별 배치 (HR, ENG, S/M, IT 등)
4. **단말 장비:** 각 부서 스위치 하단에 PC 및 IP Phone 연결

---

## 3. Core Device Configurations

### A. Access Switch Configuration (e.g., HR-Switch 2960)
```text
enable
configure terminal

! VLAN Creation
vlan 10
 name HR-Data
exit

vlan 110
 name HR-Voice
exit

! Access Port Configuration (PC & IP Phone)
interface range fa0/1 - 2
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 110
exit

! Printer Port Configuration
interface fa0/3
 switchport mode access
 switchport access vlan 10
exit

end
write memory
```

---

### B. Multilayer Switch Configuration (Core 3560)

#### 1. VLAN Creation
```text
enable
configure terminal

vlan 10
 name HR-Data
vlan 110
 name HR-Voice
vlan 20
 name ENG-Data
vlan 120
 name ENG-Voice
vlan 30
 name S/M-Data
vlan 130
 name S/M-Voice
vlan 40
 name IT-Data
vlan 140
 name IT-Voice
vlan 50
 name Server-Farm
vlan 99
 name Guests
exit
```

#### 2. Enable Routing & SVI Gateway Setup
```text
configure terminal
ip routing

! ENG-Data & Voice
interface vlan 20
 ip address 192.168.0.1 255.255.255.192
 no shutdown
exit
interface vlan 120
 ip address 192.168.0.65 255.255.255.192
 no shutdown
exit

! S/M-Data & Voice
interface vlan 30
 ip address 192.168.0.129 255.255.255.224
 no shutdown
exit
interface vlan 130
 ip address 192.168.0.161 255.255.255.224
 no shutdown
exit

! Guests
interface vlan 99
 ip address 192.168.0.193 255.255.255.224
 no shutdown
exit

! HR-Data & Voice
interface vlan 10
 ip address 192.168.0.225 255.255.255.240
 no shutdown
exit
interface vlan 110
 ip address 192.168.0.241 255.255.255.240
 no shutdown
exit

! IT-Data & Voice
interface vlan 40
 ip address 192.168.1.9 255.255.255.248
 no shutdown
exit
interface vlan 140
 ip address 192.168.1.17 255.255.255.248
 no shutdown
exit

! Server Farm Gateway
interface vlan 50
 ip address 192.168.1.1 255.255.255.248
 no shutdown
exit
```

#### 3. Access Port for DHCP Server Interface
```text
configure terminal
interface gigabitEthernet 0/2
 switchport mode access
 switchport access vlan 50
 no shutdown
exit
```

#### 4. DHCP Helper Address Configuration (Centralized DHCP Relay)
```text
configure terminal

interface vlan 10
 ip helper-address 192.168.1.2
interface vlan 110
 ip helper-address 192.168.1.2
interface vlan 20
 ip helper-address 192.168.1.2
interface vlan 120
 ip helper-address 192.168.1.2
interface vlan 30
 ip helper-address 192.168.1.2
interface vlan 130
 ip helper-address 192.168.1.2
interface vlan 40
 ip helper-address 192.168.1.2
interface vlan 140
 ip helper-address 192.168.1.2
interface vlan 50
 ip helper-address 192.168.1.2
interface vlan 99
 ip helper-address 192.168.1.2

end
write memory
```

---

## 4. Problem Solving & Troubleshooting Log

### Issue Summary
단말 PC에서 DHCP로 IP 주소를 받아오지 못하고 `Failed. APIPA (169.254.x.x)` 주소가 할당되거나 무한 대기 상태에 빠짐.

---

### Problem 1: Device Context Confusion
* **증상:** CLI 명령어를 입력했으나 적용되지 않거나 이상 동작 발생.
* **원인:** L3 멀티레이어 스위치(3560)가 아닌 L2 액세스 스위치(`hr-switch`) 터미널에 접속하여 `ip routing` 및 `ip helper-address` 설정을 입력함.
* **해결:** 액세스 스위치 단말을 닫고 중앙의 멀티레이어 스위치(3560) CLI에 접속하여 L3 전용 설정 적용.

---

### Problem 2: SVI Network Address Misconfiguration
* **증상:** 게이트웨이 IP 설정 후 통신 불능.
* **원인:** VLAN 99와 VLAN 10 SVI 설정 시, 가용 IP가 아닌 네트워크 대표 주소(Network Address)를 설정함.
  * *잘못된 설정:* `Vlan 99` -> `192.168.0.192`, `Vlan 10` -> `192.168.0.224`
* **해결:** VLSM 표 기준 첫 번째 Usable IP로 수정.
  * *수정 후:* `Vlan 99` -> `192.168.0.193/27`, `Vlan 10` -> `192.168.0.225/28`

---

### Problem 3: Port VLAN Mismatch for DHCP Server
* **증상:** DHCP 패킷이 서버로 전달되지 않음.
* **원인:** DHCP 서버가 물려 있는 멀티레이어 스위치 포트(`GigabitEthernet0/2`)가 VLAN 50(Server Farm)이 아닌 **VLAN 10(HR-Data)**으로 할당되어 있었음.
* **해결:** 해당 포트의 Access VLAN을 `vlan 50`으로 재배정.

---

### Problem 4: Missing Gateways & Unassigned SVIs (Root Cause)
* **증상:** 시뮬레이션 모드에서 DHCP Discover 브로드캐스트 패킷이 중앙 서버로 Unicast 릴레이되지 않고 전체 스위치로 Flooding 발생.
* **원인:** `show ip interface brief` 확인 결과, `Vlan 50 (Server Farm)`, `Vlan 40`, `Vlan 140`의 SVI IP가 `unassigned` 상태였음. Server Farm의 게이트웨이(`192.168.1.1`)가 비활성화되어 라우터/릴레이 경로가 차단됨.
* **해결:** L3 스위치에서 Vlan 50, Vlan 40, Vlan 140 SVI에 IP 주소를 할당하고 `no shutdown` 실행.
  ```text
  interface vlan 50
   ip address 192.168.1.1 255.255.255.248
   no shutdown
  ```

---

### Final Result
* 모든 SVI 게이트웨이가 올바르게 활성화된 후, 클라이언트 PC의 DHCP 요청 패킷이 L3 스위치를 거쳐 `192.168.1.2` DHCP 서버로 정확하게 Unicast Relay됨.
* 각 부서 PC에서 정상적으로 VLSM 대역별 IP, Subnet Mask, Gateway 주소를 할당받음 성공.