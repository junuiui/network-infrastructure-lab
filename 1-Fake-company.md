# Corporate Network Setup & Troubleshooting Guide

> **Last updated: 2026-10-27.** 최종 직원 Wi-Fi, Syslog 수신, PAT 실측 및 백업 상태는 [Section 10](#10-staff-wi-fi-syslog-pat-evidence--final-backup--2026-10-27)을 우선한다. Router/PAT/DNS/VoIP의 최신 상태와 증거 검토, 남은 범위는 [Section 9](#9-edge-routing-pat-dns--voip--2026-10-27)을 참고한다. SSH 구성 범위·접근 정책·인증 및 ACL Troubleshooting은 [Section 8](#8-ssh-remote-management--troubleshooting--2026-10-27)을 참고한다. 현재 주소와 변경 전/후 비교, DHCP 설정, 상세 Troubleshooting 및 실측 결과는 [Section 7](#7-address-capacity-expansion--troubleshooting--2026-10-27)을 우선한다. Section 1의 기존 주소표와 Section 3~6은 변경 전 이력으로 보존한다.
>
> **Previous update: 2026-10-26.** 최신 작업 상태, 명령어, Guest Wi-Fi 절차와 남은 작업은 [Section 5](#5-implementation-log--2026-10-26) 및 [Section 6](#6-remaining-work--next-session)을 참고한다. Section 1의 수량은 최종 설계 목표이며, Section 3~4는 초기 구성 및 장애 해결 기록이다.

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

---

## 5. Implementation Log — 2026-10-26

### A. Progress & Evidence

- **완료 보고:** IT 부서, Server Farm, Guest 유선 네트워크 추가 및 이전 안내 작업 완료.
- **확인 기준:** 사용자의 완료 보고 기준이다. 최신 `.pkt`와 CLI 출력은 아직 다시 검토하지 않았다.
- **Guest Wi-Fi 완료 보고:** 사용자가 2026-10-26에 무선 설정 작업 완료를 알렸다. 개별 테스트 출력과 최신 `.pkt`는 아직 수집하지 않아 아래 세부 검증 결과는 미확인으로 유지한다.
- 아래 명령은 오늘 안내한 구성의 재현용 기록이다. 실제 입력 이력이나 CLI 출력의 원본을 수집한 것은 아니다.
- 앞부분의 부서별 장비 수는 **최종 설계 목표**이며, 해당 수량을 모두 배치한 상태를 뜻하지 않는다.

### B. What / Why / How

| 작업 | 왜 했는가 | 어떻게 구성했는가 |
| :--- | :--- | :--- |
| Core Trunk 명시 설정 | 기존 부서 VLAN을 중앙 스위치까지 전달 | Core Fa0/1~3에 802.1Q Trunk 설정 |
| IT 부서 추가 | 관리용 단말을 다른 부서와 분리 | VLAN 40, IT-Switch, PC 2대, Printer 1대 연결 |
| IT Voice VLAN 준비 | 이후 IP Phone 추가를 위한 VLAN 준비 | VLAN 140 생성 및 IT Trunk에 허용; 통화 구성은 미완료 |
| Server Farm 확장 | 서버를 별도 스위치와 VLAN으로 구성 | VLAN 50에 기존 DHCP Server와 Web Server 연결 |
| 중앙 DHCP 유지 | 각 VLAN에 개별 DHCP Server를 두지 않고 주소 관리 | SVI의 ip helper-address로 192.168.1.2에 요청 전달 |
| Guest 유선 구역 추가 | Guest 접속을 내부 부서와 분리 | VLAN 99와 Guest-Switch 구성 |
| Guest ACL 적용 | 방문자의 내부 서버·부서 접근 차단 | DHCP와 Gateway Ping을 먼저 허용하고 사설망 목적지 차단 |
| DHCP 범위 정리 | 고정 IP 충돌 및 Broadcast 주소 할당 방지 | Server Pool 정리, HR Pool 사용자 수 수정 |
| Guest Wi-Fi 추가 — 사용자 완료 보고 | 무선 방문자도 기존 Guest 정책 사용 | AP를 Guest-Switch의 VLAN 99 Access Port에 연결 |

### C. Current Port Map

| Core Port | 연결 장비 | 연결 Port | 역할 |
| :--- | :--- | :--- | :--- |
| Fa0/1 | HR-Switch | Gi0/1 | 기존 부서 Trunk |
| Fa0/2 | ENG-Switch | Gi0/1 | 기존 부서 Trunk |
| Fa0/3 | S/M-Switch | Gi0/1 | 기존 부서 Trunk |
| Fa0/4 | IT-Switch | Gi0/1 | VLAN 40,140 Trunk |
| Fa0/5 | Server-Switch | Gi0/1 | VLAN 50 Trunk |
| Fa0/6 | Guest-Switch | Gi0/1 | VLAN 99 Trunk |
| Gi0/1 | Router0 | Gi0/0 | 기존 물리 연결; 외부망 Routing/NAT 미완료 |
| Gi0/2 | 현재 미사용 | - | 기존 DHCP Server 직접 연결을 Server-Switch로 이동 |

| Access Switch | Port | 단말 / 용도 |
| :--- | :--- | :--- |
| IT-Switch | Fa0/1~2 | IT-PC-1, IT-PC-2 / VLAN 40 |
| IT-Switch | Fa0/3 | IT-Printer / VLAN 40 |
| Server-Switch | Fa0/1 | DHCP-Server / VLAN 50 |
| Server-Switch | Fa0/2 | Web-Server / VLAN 50 |
| Guest-Switch | Fa0/1~2 | Guest-PC-1, Guest-PC-2 / VLAN 99 |
| Guest-Switch | Fa0/3 | Guest-AP / VLAN 99 — 사용자 완료 보고 |

> Section 3.B.3의 Core Gi0/2 직접 연결은 초기 구성 기록이다. 현재 구성은 DHCP Server를 Server-Switch Fa0/1에 연결한다. 두 위치에 동시에 연결하지 않는다.

### D. Commands — Core Switch 3560

```text
enable
configure terminal
ip routing

vlan 40
 name IT-Data
exit
vlan 140
 name IT-Voice
exit
vlan 50
 name Server-Farm
exit
vlan 99
 name Guest
exit

interface range fastEthernet0/1 - 3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 no shutdown
exit

interface fastEthernet0/4
 description TO-IT-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 40,140
 no shutdown
exit

interface fastEthernet0/5
 description TO-SERVER-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 50
 no shutdown
exit

interface fastEthernet0/6
 description TO-GUEST-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 99
 no shutdown
exit

interface vlan 40
 ip address 192.168.1.9 255.255.255.248
 ip helper-address 192.168.1.2
 no shutdown
exit

interface vlan 50
 ip address 192.168.1.1 255.255.255.248
 no shutdown
exit

interface vlan 99
 ip address 192.168.0.193 255.255.255.224
 ip helper-address 192.168.1.2
 no shutdown
exit

end
copy running-config startup-config
```

서버와 VLAN 50 Gateway는 같은 Subnet이므로 VLAN 50 자체의 DHCP Relay는 필요하지 않다. 초기 설정에 남아 있는 Helper는 이번 단계에서 통신에 필요한 항목이 아니다.

### E. Commands — New Access Switches 2960

#### IT-Switch

```text
enable
configure terminal
hostname IT-Switch

vlan 40
 name IT-Data
exit
vlan 140
 name IT-Voice
exit

interface range fastEthernet0/1 - 3
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
exit

interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 40,140
 no shutdown
exit

end
copy running-config startup-config
```

PC는 DHCP를 사용한다. IT-Printer는 DHCP 범위 밖의 `192.168.1.14/29`, Gateway `192.168.1.9`를 사용한다.
이번 PC 직접 연결 구성에는 Voice Access Port 설정이 없다. 전화기를 붙일 때 해당 Port에 `switchport voice vlan 140`을 추가해야 한다.

#### Server-Switch

```text
enable
configure terminal
hostname Server-Switch

vlan 50
 name Server-Farm
exit

interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 no shutdown
exit

interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 50
 no shutdown
exit

end
copy running-config startup-config
```

서버는 Desktop → IP Configuration → Static에서 설정한다.

| 장비 | IP | Mask | Gateway | Service |
| :--- | :--- | :--- | :--- | :--- |
| DHCP-Server | 192.168.1.2 | 255.255.255.248 | 192.168.1.1 | Services → DHCP → On |
| Web-Server | 192.168.1.3 | 255.255.255.248 | 192.168.1.1 | Services → HTTP → On |

#### Guest-Switch — Wired Clients

```text
enable
configure terminal
hostname Guest-Switch

vlan 99
 name Guest
exit

interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 99
 spanning-tree portfast
 no shutdown
exit

interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 99
 no shutdown
exit

end
copy running-config startup-config
```

### F. DHCP Pool Corrections — GUI

DHCP-Server → Services → DHCP에서 기존 Pool을 선택하고 수정한 뒤 **Save**한다.

| Pool | Start IP | Mask | Gateway | Maximum Users |
| :--- | :--- | :--- | :--- | ---: |
| IT-Data | 192.168.1.10 | 255.255.255.248 | 192.168.1.9 | 4 |
| IT-Voice | 192.168.1.18 | 255.255.255.248 | 192.168.1.17 | 4 |
| Guests | 192.168.0.194 | 255.255.255.224 | 192.168.0.193 | 28 |
| HR-Data | 192.168.0.226 | 255.255.255.240 | 192.168.0.225 | 13 |
| HR-Voice | 192.168.0.242 | 255.255.255.240 | 192.168.0.241 | 13 |

- HR-Data의 `.239`, HR-Voice의 `.255`는 Broadcast 주소이므로 할당하지 않는다.
- 서버는 Static IP를 사용하므로 `Server-Farm` Pool은 제거한다.
- 기본 `serverPool`도 제거 가능한 경우 제거한다.
- 기본 Pool 제거가 불가능한 경우 Start IP `192.168.1.6`, Mask `255.255.255.248`, Gateway `192.168.1.1`, Maximum Users `1`로 제한한다. 이 경우 `.6`은 고정 서버 주소로 사용하지 않는다.

### G. Commands — Guest Access Control

Core Switch에서 적용한다. 같은 이름의 ACL이 이미 있으면 아래 항목을 다시 추가하지 말고 `show access-lists`로 먼저 확인한다.

```text
enable
configure terminal

ip access-list extended GUEST-IN
 permit udp any eq 68 any eq 67
 permit icmp 192.168.0.192 0.0.0.31 host 192.168.0.193 echo
 deny ip any 10.0.0.0 0.255.255.255
 deny ip any 172.16.0.0 0.15.255.255
 deny ip any 192.168.0.0 0.0.255.255
 permit ip 192.168.0.192 0.0.0.31 any
exit

interface vlan 99
 ip access-group GUEST-IN in
exit

end
copy running-config startup-config
```

- DHCP Client의 UDP 68 → Server UDP 67 요청을 먼저 허용한다.
- Guest Gateway에 대한 Ping을 허용해 연결 상태를 확인한다.
- 이후 사설망 목적지를 차단하므로 내부 Web Server와 다른 부서에 접근할 수 없다.
- 마지막 Permit은 외부 목적지에 대한 정책일 뿐, Internet 연결을 생성하지 않는다.
- 이 ACL은 Guest에서 들어오는 Routed Traffic을 제한한다. 같은 VLAN의 Guest 간 통신 차단이나 양방향 Stateful Firewall을 구현한 것은 아니다.
- 현재 내부 DNS 예외는 없다. 이후 Guest DNS 설계 시 허용 정책을 별도로 검토한다.

### H. Guest Wi-Fi — Completed per User Report / Test Evidence Pending

#### Devices & Connection

1. Network Devices → Wireless Devices에서 **AccessPoint-PT**를 배치하고 이름을 `Guest-AP`로 변경한다. LAP-PT와 WLC는 이번 구성에 사용하지 않는다.
2. End Devices에서 **Laptop-PT**를 배치하고 `Guest-Laptop-1`로 이름을 변경한다.
3. Copper Straight-Through로 **Guest-Switch Fa0/3 ↔ Guest-AP Port 0**를 연결한다. AP 모델 표시가 Ethernet인 경우 유선 Ethernet Port를 선택한다.
4. AP와 Laptop을 가까이 배치한다. Laptop에는 유선 케이블을 연결하지 않는다.

단일 SSID를 Guest VLAN 하나에 연결하므로 AP 연결은 Trunk 대신 **Access VLAN 99**를 사용한다. AP는 무선 접속을 제공하고 DHCP와 Routing은 기존 장비가 담당한다.

#### Commands — Guest-Switch AP Port

```text
enable
configure terminal

interface fastEthernet0/3
 description TO-GUEST-AP
 switchport mode access
 switchport access vlan 99
 spanning-tree portfast
 no shutdown
exit

end
copy running-config startup-config
```

Core의 VLAN 99, DHCP Relay, GUEST-IN ACL이 기존대로 적용돼 있으면 Core 추가 명령은 필요 없다.

#### AP GUI Settings

Guest-AP → Config → 유선 Port 0의 Port Status가 On인지 확인한다.
무선 **Port 1**에서 다음 값을 설정한다.

| 설정 | 값 |
| :--- | :--- |
| Port Status | On |
| SSID | FakeCompany-Guest |
| Channel | 6 |
| Authentication | WPA2-PSK |
| Encryption | AES |
| Pass Phrase | 직접 정한 Lab 전용 값; 8~63 ASCII 문자 |

Pass Phrase의 실제 값은 문서에 기록하지 않는다. 단일 AP Lab에서 Channel 6은 간단한 고정값이며, 실제 환경의 Channel은 주변 간섭과 Coverage를 확인해서 정한다.
AP 설정은 GUI에서 수행하며 별도 IOS CLI 명령은 없다.

#### Laptop Wireless Adapter & Connection

1. Laptop → Physical에서 전원을 끈다.
2. 기존 Ethernet Module을 Slot 밖으로 드래그해 제거한다.
3. **WPC300N** Module을 빈 Slot에 넣고 전원을 켠다. 이미 적절한 무선 Adapter가 있으면 교체하지 않는다.
4. Desktop → PC Wireless → Connect에서 Refresh한다.
5. `FakeCompany-Guest`를 선택하고 Connect한다.
6. WPA2-Personal/PSK 항목에 AP와 같은 Pass Phrase를 입력한다.
7. Desktop → IP Configuration에서 Wireless0를 선택할 수 있으면 선택하고 DHCP를 누른다.
8. PC Wireless 메뉴가 다르면 Config → Wireless0에서 SSID, WPA2-PSK, AES, 같은 Pass Phrase와 DHCP를 설정한다.

#### Expected Address & Tests

| 항목 | 예상 값 |
| :--- | :--- |
| IP Address | 192.168.0.194~221 중 미사용 주소 |
| Subnet Mask | 255.255.255.224 |
| Default Gateway | 192.168.0.193 |

Laptop → Desktop → Command Prompt에서:

```text
ipconfig /all
ping 192.168.0.193
ping 192.168.1.2
ping 192.168.1.3
```

Core Switch에서:

```text
show access-lists
show ip interface brief
show interfaces trunk
```

Guest-Switch에서:

```text
show vlan brief
show interfaces fastEthernet0/3 switchport
show mac address-table
```

| 검증 항목 | 기대 결과 | 실제 결과 |
| :--- | :--- | :--- |
| SSID 검색 및 올바른 PSK로 연결 | 연결 성공 | 미확인 |
| DHCP 주소 할당 | Guest 범위 IP/Mask/Gateway 수신 | 미확인 |
| Gateway 192.168.0.193 Ping | 성공 | 미확인 |
| DHCP Server 192.168.1.2 Ping | 차단 | 미확인 |
| Web Server 192.168.1.3 Ping | 차단 | 미확인 |
| Browser에서 http://192.168.1.3 | 접속 차단 | 미확인 |
| DHCP 재요청 | Guest 범위 주소 재수신 | 미확인 |
| 잘못된 PSK로 신규 연결 | 인증 실패 | 미확인 |

PSK 실패 테스트 후에는 올바른 설정으로 복구한다. Internet 접속은 외부망 및 NAT/PAT 미완료로 이번 완료 기준에서 제외한다.

#### Troubleshooting

| 증상 | 확인할 항목 |
| :--- | :--- |
| SSID가 보이지 않음 | AP Port 1 On, 거리, Laptop 전원과 2.4GHz Adapter |
| SSID는 보이나 연결 실패 | SSID 철자, WPA2 설정, AES, Pass Phrase 일치 |
| Wireless0 또는 PC Wireless가 없음 | Laptop 전원 Off 후 WPC300N 장착 확인 |
| 연결됐으나 169.254.x.x 수신 | Fa0/3 Access VLAN 99, Trunk VLAN 99, Helper, Guests Pool, ACL DHCP 허용 |
| 내부 서버에 접근 가능 | Core Vlan99의 GUEST-IN inbound 적용 여부 및 ACL 순서 |
| Internet 접속 실패 | 현재 미구현 기능이므로 이 단계에서는 정상적인 한계 |

#### Save & Resume

- Guest-Switch에서 `copy running-config startup-config` 실행.
- Packet Tracer에서 **Ctrl+S**로 AP GUI 설정과 전체 Topology 저장.
- 실제 테스트 결과를 위 표에 기록하고 최신 `.pkt` 및 필요한 Screenshot을 저장소에 반영.
- 다음 작업 시작 시 최신 `.pkt`를 열어 Wireless DHCP와 ACL부터 다시 확인.

---

## 6. Remaining Work & Next Session

| 순서 | 작업 | 이유 / 완료 기준 |
| :--- | :--- | :--- |
| 1 | Wi-Fi 실제 결과 기록 | 위 테스트 표를 실측 결과로 채우고 최신 .pkt 저장 |
| 2 | Address Plan 재검토 | Gateway를 포함한 주소 용량 및 고정 IP/DHCP 중복 제거 |
| 3 | SSH 원격 관리 및 장비 보안 | IT PC에서 Core/Access Switch/Router에 SSH 접속; 관리 IP와 접근 정책 구성 |
| 4 | 모의 ISP / External Server / NAT-PAT | 내부망에서 모의 외부 서버 접속, Guest 내부망 차단 유지 |
| 5 | DNS 서비스 및 정책 | 이름으로 내부 Web 접근; Guest용 DNS와 예외 정책 결정 |
| 6 | VoIP 구성 | Voice VLAN만 있는 상태에서 전화 등록, 번호 할당, 실제 통화까지 확장 |
| 7 | 추가 Server 서비스 | FTP, Database, Syslog 등은 학습 목적과 Packet Tracer 지원 범위에 맞춰 선택 |
| 8 | Configuration Backup 자동화 | Packet Tracer 외부 연결 한계를 먼저 확인하고 실행 가능한 Lab 환경 선택 |
| 9 | SNMP / Syslog Monitoring | 시뮬레이터 지원 기능과 외부 도구 연동 가능 범위를 확인하고 검증 |
| 10 | 포트폴리오 증거 및 문서 정리 | Topology, Test Matrix, 장애 원인/해결, 실제 명령 출력 첨부 |

### Address Plan Follow-up

- **Server Farm /29:** Usable IP는 6개이며 Gateway가 1개를 사용한다. 별도 IP가 필요한 서버 6대를 모두 배치하려면 최소 7개 주소가 필요하므로 현재 목표와 맞지 않는다.
- **IT-Data /29:** Gateway를 제외하면 단말용 IP는 5개다. 목표인 Management PC 5대와 Admin Terminal 1대를 모두 연결하면 부족하다.
- **IT-Voice /29:** Gateway와 Phone 5대가 전체 Usable IP를 사용한다. 현재 DHCP Pool은 4대분이므로 목표 수량 적용 전 Pool도 조정해야 한다.
- 향후 Subnet을 확장할 때는 기존 192.168.1.0/29, .8/29, .16/29와 겹치지 않도록 함께 재설계한다.
- 초기 파일의 ENG Printer 고정 IP는 `192.168.0.4`이며 ENG-Data Pool `.2~61` 안에 있다. 변경하지 않았다면 충돌 가능성이 있으므로 DHCP 범위 밖의 예약 주소로 이동하거나 Pool을 재설계한다.

### Implementation Boundaries

- Packet Tracer의 Server-PT는 실제 Windows Server / Active Directory가 아니다. AD DS와 Domain Join은 별도 Windows Server Lab에서 검증한다.
- Packet Tracer의 모의 외부망은 실제 Internet 연결을 뜻하지 않는다.
- Netmiko, PRTG, Zabbix를 외부에서 Packet Tracer 장비에 연결할 수 있다고 가정하지 않는다. 필요하면 별도 실제/가상 네트워크 Lab을 사용한다.
- Root readme.md의 VLAN 설명은 실제 구성(HR 10, ENG 20, S/M 30, IT 40, Server 50, Guest 99)에 맞춰 후속 수정한다.
- NAT, SSH, 자동 백업, 모니터링은 검증되기 전까지 Planned로 표시한다. 측정하지 않은 “80% 감소” 수치는 사용하지 않는다.

### References

- [Cisco Packet Tracer Help — Access Point Configuration](https://tutorials.ptnetacad.net/help/default/config_others.htm)
- [Cisco Packet Tracer Help — Laptop Modules](https://tutorials.ptnetacad.net/help/default/devicesAndModules_endDevices.htm)


---

## 7. Address Capacity Expansion & Troubleshooting — 2026-10-27

### A. Scope, Problem & Evidence

**목적:** 현재 단말 수뿐 아니라 최소 50%의 추가 단말을 수용하도록 주소 용량을 재검토하고, DHCP Pool, Core SVI, 고정 IP 및 Guest ACL을 일치시킨다. 실제 배치는 부서별 대표 단말 2~3대로 기능을 검증하며, 회사 전체 목표 수량을 모두 배치한 것은 아니다.

- **Problem:** Server Farm과 IT Data의 기존 /29는 Gateway를 포함하면 원래 목표 단말 수도 수용하지 못한다. S/M Data/Voice, Guest, HR Data, IT Voice는 새로 정한 50% 성장 여유 기준을 충족하지 못한다.
- 이는 모든 VLAN에서 실제 DHCP 고갈이 관찰됐다는 뜻은 아니다. 주소표에서 확인한 **설계상 용량 부족 / 성장 여유 부족**이다.
- 산정 기준: `ceil(현재 필요한 단말 수 × 1.5) + Gateway 등 인프라 주소`.
- /28은 전체 16개, Network/Broadcast 제외 14개이며 Gateway 1개 제외 시 단말 13대를 수용한다. HR Voice 8대는 원래 /28로도 50% 성장 여유를 충족한다.
- HR Voice는 사용자의 추가 확장 결정에 따라 /27로 변경했다. 실제 주소 부족으로 인한 변경으로 기록하지 않는다.
- 증거: 사용자가 제공한 변경 전 Core running-config / show ip interface brief / ACL 출력, 변경 후 DHCP Pool 화면, 대표 PC·Server·무선 Laptop의 ipconfig 및 Ping 출력.
- 최신 .pkt 자체를 직접 열어 검증하지 않았으며, 문서의 명령은 설정 절차 기록이다. 실제 장비별 전체 입력 이력과 동일하다고 단정하지 않는다.
- 직원 Wi-Fi 추가는 사용자 요청으로 연기했다.

### B. Original vs New IP Address Plan

**Section 1의 기존 VLSM 표는 변경 전 기록으로 그대로 보존한다. 현재 작업 기준은 아래 신규 표이다.**
Usable IP Range는 Gateway도 포함하며, DHCP 범위와는 다르다.

| Subnet | VLAN | 기존 Network / Prefix | 기존 Gateway | 신규 Network / Prefix | 신규 Gateway | 신규 Mask | 신규 Usable IP Range |
| :--- | ---: | :--- | :--- | :--- | :--- | :--- | :--- |
| ENG-Data | 20 | 192.168.0.0/26 | 192.168.0.1 | 192.168.0.0/26 | 192.168.0.1 | 255.255.255.192 | 192.168.0.1~62 |
| ENG-Voice | 120 | 192.168.0.64/26 | 192.168.0.65 | 192.168.0.64/26 | 192.168.0.65 | 255.255.255.192 | 192.168.0.65~126 |
| S/M-Data | 30 | 192.168.0.128/27 | 192.168.0.129 | 192.168.1.64/26 | 192.168.1.65 | 255.255.255.192 | 192.168.1.65~126 |
| S/M-Voice | 130 | 192.168.0.160/27 | 192.168.0.161 | 192.168.1.128/26 | 192.168.1.129 | 255.255.255.192 | 192.168.1.129~190 |
| Guests | 99 | 192.168.0.192/27 | 192.168.0.193 | 192.168.1.192/26 | 192.168.1.193 | 255.255.255.192 | 192.168.1.193~254 |
| HR-Data | 10 | 192.168.0.224/28 | 192.168.0.225 | 192.168.0.192/27 | 192.168.0.193 | 255.255.255.224 | 192.168.0.193~222 |
| HR-Voice | 110 | 192.168.0.240/28 | 192.168.0.241 | 192.168.0.224/27 | 192.168.0.225 | 255.255.255.224 | 192.168.0.225~254 |
| Server Farm | 50 | 192.168.1.0/29 | 192.168.1.1 | 192.168.1.0/28 | 192.168.1.1 | 255.255.255.240 | 192.168.1.1~14 |
| IT-Data | 40 | 192.168.1.8/29 | 192.168.1.9 | 192.168.1.16/28 | 192.168.1.17 | 255.255.255.240 | 192.168.1.17~30 |
| IT-Voice | 140 | 192.168.1.16/29 | 192.168.1.17 | 192.168.1.32/28 | 192.168.1.33 | 255.255.255.240 | 192.168.1.33~46 |

신규 대역끼리는 겹치지 않는다. 192.168.0.128/26은 S/M 이동 후 설계상 미사용으로 남긴다. 기존 주소를 가진 단말의 잔여 설정까지 모두 제거됐다는 뜻은 아니다.

### C. DHCP Pools & Static IP Separation

Pool 수정 위치: **DHCP-Server → Services → DHCP → 기존 Pool 선택 → 수정 → Save**. 기존 이름을 유지하며 중복 Pool을 새로 만들지 않는다.

| Pool | Gateway | DHCP Start | Mask | Max Users | DHCP Range | 확인 상태 |
| :--- | :--- | :--- | :--- | ---: | :--- | :--- |
| ENG-Data | 192.168.0.1 | 192.168.0.2 | 255.255.255.192 | 58 | 192.168.0.2~59 | 58로 축소 안내, 최종 Pool 화면 재확인 필요 |
| ENG-Voice | 192.168.0.65 | 192.168.0.66 | 255.255.255.192 | 60 | 192.168.0.66~125 | DHCP 화면 확인 |
| S/M-Data | 192.168.1.65 | 192.168.1.66 | 255.255.255.192 | 60 | 192.168.1.66~125 | 화면 및 대표 PC 확인 |
| S/M-Voice | 192.168.1.129 | 192.168.1.130 | 255.255.255.192 | 61 | 192.168.1.130~190 | 화면 확인, 전화기 검증 대기 |
| Guests | 192.168.1.193 | 192.168.1.194 | 255.255.255.192 | 61 | 192.168.1.194~254 | 화면 및 유선/무선 단말 확인 |
| HR-Data | 192.168.0.193 | 192.168.0.194 | 255.255.255.224 | 26 | 192.168.0.194~219 | 화면 및 대표 PC 확인 |
| HR-Voice | 192.168.0.225 | 192.168.0.226 | 255.255.255.224 | 29 | 192.168.0.226~254 | 화면 확인, 전화기 검증 대기 |
| IT-Data | 192.168.1.17 | 192.168.1.18 | 255.255.255.240 | 10 | 192.168.1.18~27 | 화면 및 대표 PC 확인 |
| IT-Voice | 192.168.1.33 | 192.168.1.34 | 255.255.255.240 | 13 | 192.168.1.34~46 | 화면 확인, 전화기 검증 대기 |
| serverPool | 192.168.1.1 | 192.168.1.6 | 255.255.255.240 | 1 | 192.168.1.6만 | DHCP 화면 확인 |

- 위 표의 serverPool은 **192.168.1.6 한 개**만 할당하며, 고정 서버에는 사용하지 않는다.
- DNS Server는 화면상 모두 `0.0.0.0`. DNS 미구성 상태로 이름 해석은 검증하지 않았다.
- 처음에는 serverPool을 .14로 이동하도록 안내했지만 실제 최종 화면은 **.6 /28, 1대분**이다. 실제 화면을 우선하며 .6을 예약한다.
- 서버는 Static IP를 사용한다. 기본 serverPool을 남겨두는 것이 전체 서버를 DHCP로 관리한다는 뜻은 아니다.

| 장비 / 예약 | 기존 값 | 신규 값 | 비고 |
| :--- | :--- | :--- | :--- |
| DHCP Server | 192.168.1.2/29, GW .1 | 192.168.1.2/28, GW .1 | Pool 화면은 확인, NIC Mask 출력 별도 확인 필요 |
| Web Server | 192.168.1.3/29, GW .1 | 192.168.1.3/28, GW .1 | ipconfig /all 확인 |
| IT Printer | 192.168.1.14/29, GW .9 | 192.168.1.30/28, GW .17 | Ping 응답 확인 |
| S/M Printer | 기존 값 미수집 | 192.168.1.126/26, GW 192.168.1.65 | 설정 안내, 실제 결과 미수집 |
| HR Printer | 기존 값 미수집 | 192.168.0.221/27, GW .193 | Ping 응답 확인 |
| HR 추가 Printer 예약 | 기존 값 미수집 | 192.168.0.222/27, GW .193 | 실제 배치/통신 미확인 |
| ENG Printer | 초기 기록 192.168.0.4 | 192.168.0.62/26, GW .1 | .62 Ping 응답 확인 |
| ENG 고정 IP 예약 | DHCP .2~61과 중복 가능 | 192.168.0.60~62 | ENG Pool 58 적용 확인 필요 |

### D. Migration Method & Core Configuration

**첫 해결 방법:** DHCP Pool의 Mask, Gateway, Start IP, Maximum Users를 수정하고 Core SVI도 변경한다. Mask만 바꾸는 것으로는 충분하지 않다. 이동 대상의 고정 IP, Guest ACL, 기존 DHCP Lease도 함께 정리한다.

**순서의 이유:**
1. IT Data/Voice의 기존 SVI 주소를 먼저 제거한 뒤 새 대역으로 이동한다. 새 IT Data /28은 기존 IT Voice .17과 겹친다.
2. IT 이동 후 Server Farm을 /28로 확장한다. Server를 먼저 확장하면 기존 IT Data .8/29와 겹친다.
3. S/M을 192.168.1.64/26 및 .128/26으로 이동한다.
4. Guest를 192.168.1.192/26으로 이동하고 ACL을 변경한다.
5. Guest 이동 후 HR Data를 192.168.0.192/27로 이동한다.
6. HR Data 이동 후 HR Voice를 192.168.0.224/27로 확장한다. 기존 HR Data Gateway .225와의 충돌을 피한다.

아래는 대화에서 안내한 변경 명령을 순서대로 통합한 기록이다. **이미 적용한 장비에 다시 실행할 필요는 없다.** VLAN 번호, Access Port VLAN, 기존 DHCP Helper 192.168.1.2는 유지한다.

```text
enable
configure terminal

! 1. Move IT first
interface vlan 40
 no ip address
exit
interface vlan 140
 no ip address
exit
interface vlan 40
 ip address 192.168.1.17 255.255.255.240
 no shutdown
exit
interface vlan 140
 ip address 192.168.1.33 255.255.255.240
 no shutdown
exit

! 2. Expand Server Farm
interface vlan 50
 ip address 192.168.1.1 255.255.255.240
exit

! 3. Move Sales/Marketing
interface vlan 30
 no ip address
exit
interface vlan 130
 no ip address
exit
interface vlan 30
 ip address 192.168.1.65 255.255.255.192
 no shutdown
exit
interface vlan 130
 ip address 192.168.1.129 255.255.255.192
 no shutdown
exit

! 4. Move Guest and replace ACL while SVI is down
interface vlan 99
 shutdown
 no ip address
 ip address 192.168.1.193 255.255.255.192
exit
no ip access-list extended GUEST-IN
ip access-list extended GUEST-IN
 permit udp any eq 68 any eq 67
 permit icmp 192.168.1.192 0.0.0.63 host 192.168.1.193 echo
 deny ip any 10.0.0.0 0.255.255.255
 deny ip any 172.16.0.0 0.15.255.255
 deny ip any 192.168.0.0 0.0.255.255
 permit ip 192.168.1.192 0.0.0.63 any
exit
interface vlan 99
 ip access-group GUEST-IN in
 no shutdown
exit

! 5. Move HR Data before expanding HR Voice
interface vlan 10
 no ip address
 ip address 192.168.0.193 255.255.255.224
 no shutdown
exit
interface vlan 110
 no ip address
 ip address 192.168.0.225 255.255.255.224
 no shutdown
exit

end
show ip interface brief
show access-lists
```

- 서버 NIC 변경 위치: Desktop → IP Configuration → Static. DHCP/Web Server의 IP와 Gateway는 유지하고 Mask를 255.255.255.240으로 변경한다.
- DHCP 단말 갱신: Desktop → IP Configuration → Static 선택 후 DHCP 재선택. 무선 Laptop은 Wireless0를 선택한다.
- Printer의 IP/Mask는 Config → FastEthernet0, Gateway는 Config → Settings 등 해당 모델의 설정 화면에서 변경한다.
- 최종 저장: Core에서 `copy running-config startup-config`, Packet Tracer에서 Ctrl+S. 저장 명령 실행 완료 증거는 별도 수집하지 않았다.

### E. Troubleshooting Details — Problem → Attempt → Verification

#### Issue 1: Capacity calculation and DHCP/Subnet mismatch risk

- **문제 발견:** Server/IT의 /29는 사용 가능 6개 중 Gateway가 1개를 차지해 단말은 5대만 수용한다. DHCP Maximum Users만 늘려도 Subnet 용량 자체는 늘지 않는다.
- **첫 해결 시도:** 필요한 구역의 DHCP Mask와 Pool 범위를 확대하고 Core SVI를 같은 Mask/대역으로 변경한다.
- **추가 조치:** 겹치는 Subnet을 이동하고 Printer 고정 IP와 Pool을 분리한다. 서버 NIC Mask도 수정한다.
- **검증:** 아래 Test Matrix의 실제 출력으로 DHCP 재할당 및 대표 경로 통신을 확인했다.
- **한계:** 최대 단말 수까지 Lease를 소진하는 부하 테스트는 하지 않았다. 주소 용량은 산술 검토, 실제 동작은 대표 단말로 검증했다.

#### Issue 2: Overlapping subnet rejected by Core

사용자가 S/M 주소를 192.168.0.65/26으로 설정하려 했을 때 실제 오류:

```text
Switch(config-if)#no ip address
Switch(config-if)#ip address 192.168.0.65 255.255.255.192
% 192.168.0.64 overlaps with Vlan120
```

- **원인:** 192.168.0.65는 기존 ENG-Voice 192.168.0.64/26에 속한다. ENG의 .2/.66은 DHCP 시작 주소이며, 실제 예약된 Subnet 전체는 .0~63 및 .64~127이다.
- **영향:** `no ip address`는 적용되고 다음 IP 명령은 거절될 수 있어 해당 SVI Gateway가 제거된 중간 상태가 될 수 있다.
- **추가 검토:** S/M Data .0.129와 Voice .0.162도 /26에서는 같은 .0.128~191 대역이므로 분리할 수 없다. 다음 .0.192/26은 당시 Guest/HR과 겹쳤다.
- **해결:** S/M Data는 192.168.1.64/26 (GW .1.65), Voice는 192.168.1.128/26 (GW .1.129)으로 확정. 두 SVI를 명시적으로 지정해 주소를 재설정했다.
- **검증:** S/M PC가 192.168.1.67/26, GW 192.168.1.65를 수신하고 Gateway/DHCP Server/Web Server에서 응답을 받았다.
- **교훈:** 마지막 Octet이나 DHCP 시작 주소만 비교하지 말고 Network~Broadcast 전체 범위를 비교한다. 설정 전 변경 대상 SVI와 명령 순서를 확인한다.

#### Issue 3: Old DHCP address remains after migration

- **관찰:** Guest PC의 첫 ipconfig는 192.168.0.195/27, GW 192.168.0.193이었다. 이후 출력은 192.168.1.195/26, GW 192.168.1.193으로 변경됐다.
- **원인 해석:** 기존 Lease/단말 설정이 남아 있을 수 있다. 첫 출력만으로 새 DHCP Pool이 잘못됐다고 단정하지 않는다.
- **해결 절차:** DHCP Pool Save와 SVI 변경 후 단말에서 Static → DHCP로 재요청한다.
- **검증:** Guest 유선 .1.195 및 무선 .1.196이 새 /26과 Gateway를 수신했다.

#### Issue 4: Guest gets DHCP but cannot Ping the DHCP Server

- **관찰:** Guest Gateway Ping은 성공했지만 .1.2/.1.3은 Gateway가 Destination host unreachable로 응답했다.
- **의도:** DHCP UDP 68→67은 허용하고 내부 IP 통신은 차단하는 정책이다. DHCP 주소 수신과 서버 Ping은 별개이다.
- **검증:** 유선 서버 Ping 총 7회 실패와 ACL의 `deny ip any 192.168.0.0 0.0.255.255 (7 match(es))`가 일치했다.
- **결론:** 유선 테스트에서는 ACL 차단 카운터로 정책 동작을 확인했다. 무선도 새 주소 수신/Gateway 성공/서버 실패를 확인했으나 무선 테스트 후 추가 ACL 카운터는 수집하지 않았다.
- **주의:** 마지막 permit의 1 match는 Internet/NAT 성공 증거가 아니다. 외부망은 아직 미구현이다.

실제 수집한 ACL 카운터:

```text
10 permit udp any eq bootpc any eq bootps (13 match(es))
20 permit icmp 192.168.1.192 0.0.0.63 host 192.168.1.193 echo (2 match(es))
30 deny ip any 10.0.0.0 0.255.255.255
40 deny ip any 172.16.0.0 0.15.255.255
50 deny ip any 192.168.0.0 0.0.255.255 (7 match(es))
60 permit ip 192.168.1.192 0.0.0.63 any (1 match(es))
```

#### Issue 5: ENG Printer static IP overlaps the DHCP range

- **기존 위험:** 초기 기록의 Printer .0.4가 DHCP .0.2~61 안에 있어 같은 주소를 다른 Client에 할당할 가능성이 있었다. 실제 중복 Lease 발생 출력은 수집하지 않았다.
- **해결 안내:** ENG-Data Maximum Users를 60→58로 줄여 .2~59만 할당하고 .60~62를 고정 IP용으로 분리한다. Printer를 .62/26, GW .1로 변경한다.
- **검증:** ENG PC에서 Printer .62에 3/3, DHCP Server .1.2에 2/2 응답.
- **남은 확인:** 마지막 Pool 화면은 변경 전 Max Users 60이었다. Printer 응답만으로 Pool이 58로 저장됐다고 입증할 수 없으므로 최종 화면 재확인이 필요하다.

#### Issue 6: First Web Server Ping timeout and invalid test targets

- S/M PC→Web .1.3 테스트는 첫 요청 timeout, 이후 2개 응답 (3 sent / 2 received)이었다. 초기 ARP 과정일 가능성은 있으나 확인된 원인은 아니다. 후속 4회 연속 성공 출력은 미수집이다.
- Guest가 192.168.1.192에 Ping했을 때 .193에서 응답했다. .192는 /26의 Network Address이므로 단말 연결 검증 대상으로 사용하지 않는다.
- `ing`은 `ping` 오타이고 `show running config`는 `show running-config`로 수정했다. 이 오류는 네트워크 장애 원인이 아니다.

### F. Actual Verification Matrix

아래 수치는 사용자가 제공한 출력 그대로이며, 여러 테스트는 Ctrl+C로 조기 종료했다. 전부 4회씩 수행한 것으로 기록하지 않는다.

| 출발 장비 | 주소/설정 또는 목적지 | 실제 결과 | 판정 |
| :--- | :--- | :--- | :--- |
| IT PC | .1.19 /28, GW .1.17, DHCP .1.2 | ipconfig 확인 | DHCP 정상 |
| IT PC | .1.17 / .1.2 / Printer .1.30 | 2/2, 2/2, 4/4 응답 | 성공 |
| Web Server | .1.3 /28, GW .1.1 | ipconfig 확인 | NIC 설정 확인 |
| Web Server | IT PC .1.19 | 2/2 응답 | VLAN 간 통신 성공 |
| S/M PC | .1.67 /26, GW .1.65, DHCP .1.2 | ipconfig 확인 | DHCP 정상 |
| S/M PC | .1.65 / .1.2 | 2/2, 3/3 응답 | 성공 |
| S/M PC | Web .1.3 | 3회 중 2회 응답, 첫 timeout | 응답 확인, 연속 성공 재검증 대기 |
| Guest wired PC | .1.195 /26, GW .1.193 | ipconfig 확인 | DHCP 정상 |
| Guest wired PC | Gateway .1.193 | 2/2 응답 | 성공 |
| Guest wired PC | .1.2 / .1.3 | 0/4, 0/3; unreachable | 예상 차단, ACL 7 matches 확인 |
| HR PC | .0.195 /27, GW .0.193 | ipconfig 확인 | DHCP 정상 |
| HR PC | Gateway .0.193 / .1.2 / .1.3 / Printer .0.221 | 2/2, 3/3, 2/2, 2/2 응답 | 성공 |
| ENG PC | Printer .0.62 / DHCP .1.2 | 3/3, 2/2 응답 | 성공 |
| Guest wireless Laptop | Wireless0 .1.196 /26, GW .1.193 | ipconfig 확인 | 무선 DHCP 정상 |
| Guest wireless Laptop | Gateway .1.193 | 2/2 응답, 23~35 ms | 성공 |
| Guest wireless Laptop | .1.2 / .1.3 | 각각 0/2; unreachable | 예상 차단 |
| 모든 Voice VLAN | 전화기 주소 수신, 등록, 실제 통화 | 출력 미수집 | 검증 대기 |

**재현용 검증 명령:** 단말의 역할에 맞춰 실행한다.

```text
! IT PC
ipconfig /all
ping 192.168.1.17
ping 192.168.1.2
ping 192.168.1.30

! S/M PC
ipconfig /all
ping 192.168.1.65
ping 192.168.1.2
ping 192.168.1.3
ping 192.168.1.126

! HR PC
ipconfig /all
ping 192.168.0.193
ping 192.168.1.2
ping 192.168.1.3
ping 192.168.0.221

! ENG PC
ping 192.168.0.62
ping 192.168.1.2

! Guest wired / wireless: Gateway success, internal targets blocked
ipconfig /all
ping 192.168.1.193
ping 192.168.1.2
ping 192.168.1.3

! Core
show ip interface brief
show running-config
show access-lists
show interfaces trunk
```

### G. Remaining Checks & Next Work

1. ENG-Data Maximum Users **58** 최종 저장 확인. 기존 .60~62 Lease가 있으면 재할당 후 고정 장비에 사용.
2. DHCP Server NIC의 /28 Mask, 최종 Core SVI/Helper 전체 출력, S/M Printer 설정/통신 확인.
3. S/M→Web Ping을 중단 없이 다시 실행해 연속 응답 확인.
4. Voice VLAN의 Gateway/Pool은 구성했으나 전화기 DHCP, 등록, 통화는 별도 검증.
5. 최신 .pkt와 장비 startup-config 저장 완료 및 저장소 업로드 여부 확인.
6. Core Fa0/1~3은 변경 전 running-config에서 명시적 Trunk 설정이 없었다. 실제 Trunk 동작은 show interfaces trunk로 확인 필요.
7. 직원 Wi-Fi는 연기. SSH 관리, 모의 외부망/NAT-PAT, DNS, VoIP, 지원 가능한 백업/로그 및 최종 포트폴리오 문서 작업을 이어간다.
8. Section 3~6의 예전 IP/미확인 상태는 당시 기록이다. 현재 주소 및 오늘 확인된 결과는 Section 7을 우선한다. Root README의 구현 상태 정합성은 후속 작업이다.



---

## 8. SSH Remote Management & Troubleshooting — 2026-10-27

### A. What / Why / Scope

**목적:** IT 관리 PC에서 암호화된 SSH로 장비를 관리하고, IT Data 대역 이외의 원격 관리 접속을 제한한다. 기존 Data/Voice VLAN과 부서 간 일반 통신은 유지한다.

| 장비 | SSH 접속 주소 | 구현 및 검증 상태 |
| :--- | :--- | :--- |
| Core-Switch | 192.168.1.17 | SSH v2 활성화 출력, 로그인 성공, 접근 ACL 허용/차단 확인 |
| IT-Switch | 192.168.1.28 | 관리 SVI 추가, 로그인 성공, 접근 ACL 허용/차단 확인 |
| HR-Switch | 192.168.0.220 | 관리 SVI 추가, 로그인 성공, 접근 ACL 허용/차단 확인 |
| ENG-Switch | 미배정 | 동일한 Access Switch 설정 반복이므로 이번 실습에서는 의도적으로 생략 |
| S/M-Switch | 미배정 | 동일한 반복 작업으로 생략 |
| Server-Switch | 미배정 | 이번 SSH 구현 범위에 포함하지 않음 |
| Guest-Switch | 미배정 | 이번 SSH 구현 범위에 포함하지 않음 |
| Router0 | 미확인 | 이번 단계에서 SSH 구성/검증하지 않음 |

사용자는 Core 및 대표 Access Switch에서 기능과 접근 정책을 검증한 뒤, ENG/S/M 등으로의 반복 적용을 생략하기로 결정했다. 미구성 장비를 설정 실패 또는 구현 완료로 표시하지 않는다. 전체 장비의 원격 관리가 구축된 상태는 아니다.

### B. Management Address & Access Policy

| 항목 | Core-Switch | IT-Switch | HR-Switch |
| :--- | :--- | :--- | :--- |
| 관리에 사용하는 VLAN | 40 | 40 | 10 |
| SVI 주소 | 192.168.1.17/28 | 192.168.1.28/28 | 192.168.0.220/27 |
| 역할 | 기존 IT Gateway를 SSH 목적지로 사용 | L2 Switch 자체의 관리 IP | L2 Switch 자체의 관리 IP |
| L2 관리 Default Gateway | 해당 없음: Core는 ip routing 사용 | 192.168.1.17 | 192.168.0.193 |
| DHCP와의 분리 | 기존 Gateway | DHCP .1.18~27 밖 | DHCP .0.194~219 밖 |

- HR의 .0.220은 이제 HR-Switch 관리용이다. Printer에는 .0.221~222를 사용하며 .220을 중복 사용하지 않는다.
- **허용 Source:** IT Data 192.168.1.16/28. 일반적인 단말 주소는 .18~27이나 ACL은 IT Subnet 전체를 허용한다.
- **정책 적용 위치:** VTY의 `access-class SSH-IT-ONLY in`. 부서 간 Routed Traffic을 제한하는 인터페이스 ACL과는 별개이다.
- Guest는 기존 GUEST-IN에서도 내부망 접근이 차단된다. 이번 비IT SSH 차단 검증은 대화에서 요청한 HR/S/M 테스트 결과를 기준으로 하며, 제공된 PC 출력에는 출발지 IP가 별도로 포함되지 않았다.
- 별도 Management VLAN을 새로 구성하지 않았다. 기존 부서 VLAN의 SVI를 관리에 사용하고 VTY ACL로 접속 출발지를 제한한다.
- 실제 비밀번호 및 해시값은 문서에 기록하지 않는다.

### C. Configuration Procedure & Commands

장비의 직접 CLI에서 설정한다. 아래 명령은 안내한 구성의 재현 기록이며, 실행 전 기존 계정/VTY/RSA 설정을 확인한다. `<LAB_PASSWORD>`는 실제로 입력할 문자열이 아니라 사용자가 정한 Lab 전용 비밀번호를 뜻한다.

#### Core-Switch

```text
enable
configure terminal
hostname Core-Switch
ip domain-name fakecompany.lab

username netadmin privilege 15 secret <LAB_PASSWORD>

ip access-list standard SSH-IT-ONLY
 permit 192.168.1.16 0.0.0.15
 deny any
exit

crypto key generate rsa
! When prompted for modulus size: 2048

ip ssh version 2

line vty 0 4
 login local
 transport input ssh
 access-class SSH-IT-ONLY in
 exec-timeout 5 0
exit

end
show ip ssh
show access-lists
```

제공된 Core VTY 출력은 0~4 구간이다. 다른 VTY 구간이 존재한다면 해당 구간도 동일 정책을 적용해야 하며, 이번 출력만으로 미제공 구간까지 확인했다고 기록하지 않는다. RSA 2048을 안내했지만 실제 키 크기 확인 출력은 수집하지 않았다.

#### IT-Switch

```text
enable
configure terminal

interface vlan 40
 ip address 192.168.1.28 255.255.255.240
 no shutdown
exit
ip default-gateway 192.168.1.17

ip domain-name fakecompany.lab
username netadmin privilege 15 secret <LAB_PASSWORD>

ip access-list standard SSH-IT-ONLY
 permit 192.168.1.16 0.0.0.15
 deny any
exit

crypto key generate rsa
! When prompted for modulus size: 2048
ip ssh version 2

line vty 0 15
 login local
 transport input ssh
 access-class SSH-IT-ONLY in
 exec-timeout 5 0
exit

end
```

IT-Switch의 기존 Trunk는 VLAN 40,140을 허용하므로 관리 SVI 추가를 위해 Trunk를 변경할 필요는 없었다.

#### HR-Switch

```text
enable
configure terminal
hostname HR-Switch

interface vlan 10
 ip address 192.168.0.220 255.255.255.224
 no shutdown
exit
ip default-gateway 192.168.0.193

ip domain-name fakecompany.lab
username netadmin privilege 15 secret <LAB_PASSWORD>

ip access-list standard SSH-IT-ONLY
 permit 192.168.1.16 0.0.0.15
 deny any
exit

crypto key generate rsa
! When prompted for modulus size: 2048
ip ssh version 2

line vty 0 15
 login local
 transport input ssh
 access-class SSH-IT-ONLY in
 exec-timeout 5 0
exit

end
```

### D. Command Meaning & Design Understanding

| 명령 / 개념 | 의미 |
| :--- | :--- |
| enable | 사용자 모드 >에서 관리자 EXEC 모드 #로 전환 |
| configure terminal | 전체 설정 모드 (config)# 진입 |
| username netadmin | 현재 장비 안에 로컬 계정 생성 |
| privilege 15 | 최고 권한 수준 부여; 성공 시 바로 #로 진입 |
| secret | 비밀번호를 해시 형태로 저장 |
| line vty 0 15 | 가상 원격 터미널 0~15, 총 16개 라인을 선택해 같은 설정 적용 |
| login local | 해당 장비의 로컬 계정으로 인증 |
| transport input ssh | 해당 VTY의 원격 접속을 SSH로 제한 |
| access-class ... in | VTY에 들어오는 연결의 Source IP 제한 |
| exec-timeout 5 0 | 5분 0초 동안 입력이 없으면 세션 종료 |
| exit / end | 설정 계층 한 단계 종료 / 관리자 EXEC 모드까지 종료 |

- VTY 0~15는 Ethernet Port나 VLAN 번호가 아니다. VTY를 새로 생성하는 명령도 아니다.
- SSH 세션은 보통 VTY 하나를 사용한다. 16개 라인을 선택했다는 것은 장비의 지원/제한 범위에서 16개 접속 자리에 설정을 적용하는 것이다.
- `privilege 15`의 15는 권한, `line vty 0 15`의 15는 라인 번호이므로 의미가 다르다.
- Core와 Access Switch에 같은 VLAN 번호가 있어도 SVI IP는 장비별로 별개이다. Core VLAN 10의 .193은 HR Gateway이고 HR-Switch VLAN 10의 .220은 그 스위치 자체의 관리 주소이다.
- L2 Switch는 관리 IP 없이도 일반 Ethernet Frame을 전달할 수 있지만, IP 기반 SSH 접속을 받으려면 관리 IP가 필요하다.
- 계정과 비밀번호는 각 장비에 따로 저장된다. Core에서 계정을 만들었다고 Access Switch에 자동 생성되지 않는다.

### E. Troubleshooting — Detailed Problem / Action / Result

#### Issue 1: SSH activation log shows 1.99 after configuring v2

- **관찰:** `ip ssh version 2` 입력 시 "%SSH-5-ENABLED: SSH 1.99 has been enabled" 로그가 나왔다.
- **조치:** 로그 문자열만으로 최종 버전을 판단하지 않고 `show ip ssh`로 확인했다.
- **실제 결과:**
```text
SSH Enabled - version 2.0
Authentication timeout: 120 secs; Authentication retries: 3
```
- **판정:** Core의 SSH v2 활성화 확인. 이 출력만으로 계정 로그인과 ACL 정책까지 검증된 것은 아니므로 별도 접속 테스트를 진행했다.

#### Issue 2: Core SSH reaches Password prompt but authentication fails

- **증상:** Password 프롬프트가 반복된 뒤 "Connection ... closed by foreign host"가 발생했다.
- **판단:** SSH 인증 단계까지 도달했지만 인증이 실패했다. 원래 비밀번호 불일치/입력 오류의 구체적인 원인은 확인하지 않았다.
- **조치:** Core 직접 CLI에서 netadmin 계정의 secret을 다시 지정하고 VTY의 login local을 적용했다.
```text
configure terminal
username netadmin privilege 15 secret <NEW_LAB_PASSWORD>
line vty 0 4
 login local
exit
end
```
- **후속:** 이후 Connection refused 문제가 별도로 발생해 ACL을 점검했다. 처음의 인증 실패와 이후 ACL 차단을 하나의 원인으로 단정하지 않는다.

#### Issue 3: Core SSH Connection refused due to wrong allowed subnet

- **증상:** 재시도 시 Password 프롬프트 전에 "% Connection refused by remote host"가 발생했다.
- **진단:** show ip ssh는 v2 활성, show users는 콘솔만 사용 중, VTY는 login local / transport input ssh / access-class가 설정돼 있었다.
- **실제 오류 설정:**
```text
Standard IP access list SSH-IT-ONLY
 10 permit 192.168.1.0 0.0.0.15
 20 deny any (10 match(es))
```
- **원인:** .0/28 (주소 .0~15)을 허용해 실제 IT Data .16/28 (단말 .18~27)가 deny any에 걸렸다.
- **해결:** 잘못된 permit과 deny를 제거하고 올바른 permit을 먼저, deny를 나중에 다시 추가했다.
```text
configure terminal
ip access-list standard SSH-IT-ONLY
 no permit 192.168.1.0 0.0.0.15
 no deny any
 permit 192.168.1.16 0.0.0.15
 deny any
exit
end
```
- **순서의 이유:** ACL은 위에서부터 평가하므로 올바른 permit이 deny any 뒤에 있으면 허용되지 않는다.
- **재검증:** IT PC에서 .1.17에 netadmin 로그인 후 Core-Switch# 도달. 비IT 테스트에서 Connection refused, 이후 올바른 ACL의 permit 2 matches / deny 5 matches 확인.
- **교훈:** Subnet 변경 후 관리 ACL도 실제 Source Network에 맞춰야 한다. Connection refused 자체만으로 원인을 단정하지 말고 SSH 상태·VTY·ACL 카운터를 함께 확인한다.

#### Issue 4: IT-Switch local authentication failure

- **증상:** .1.28 SSH에서 Password 반복 후 연결 종료.
- **조치:** IT-Switch의 직접 CLI에서 netadmin secret을 재설정하고 VTY 0~15에 login local을 적용했다.
```text
configure terminal
username netadmin privilege 15 secret <NEW_LAB_PASSWORD>
line vty 0 15
 login local
exit
end
```
- **실제 결과:** 재시도 후 IT-Switch# 진입 및 exit 정상.
- **원인 범위:** 계정 재설정 후 정상화됐지만 최초 비밀번호 불일치의 구체적 이유는 확인하지 않았다. 장비별 로컬 계정이 별개라는 점을 설명했다.
- **정책 검증:** IT-Switch ACL에서 permit 16 matches / deny 16 matches 수집.

#### Issue 5: Why HR-Switch needs another VLAN 10 address

- **질문:** Core에 VLAN 10 IP를 설정했는데 왜 HR-Switch에도 다시 설정하는가?
- **설명:** Core .0.193은 라우팅 Gateway, HR-Switch .0.220은 해당 L2 장비의 관리 목적지이다. 같은 VLAN의 서로 다른 호스트 주소이므로 중복 설정이 아니다.
- **결과:** .0.220 SSH 로그인 후 HR-Switch# 도달. ACL permit 2 matches / deny 15 matches 수집.

### F. Actual SSH Verification Results

| 대상 | 허용 접속 결과 | 차단 테스트 / ACL 결과 | 판정 |
| :--- | :--- | :--- | :--- |
| Core .1.17 | netadmin 로그인 후 Core-Switch# | 비IT 시도 Connection refused; permit 2 / deny 5 | 로그인 및 접근 제한 확인 |
| IT-Switch .1.28 | 계정 재설정 후 IT-Switch# | permit 16 / deny 16 | 로그인 및 ACL 허용/차단 확인 |
| HR-Switch .0.220 | 로그인 후 HR-Switch# | permit 2 / deny 15 | 로그인 및 ACL 허용/차단 확인 |

- ACL match는 일치한 패킷 수이며 접속자 수, 세션 수 또는 실패한 비밀번호 횟수가 아니다.
- PC 로그인 출력 자체에는 Source IP가 표시되지 않아, 테스트 출발지는 대화의 IT/HR/S/M 절차 맥락에 기반한다.
- IT/HR Switch의 별도 show ip ssh 출력 및 전체 최종 running-config는 미수집이다. SSH 로그인은 확인했지만 SSH v2/전체 VTY 최종 상태는 추가 출력으로 확인할 수 있다.
- Telnet 차단은 transport input ssh로 설정하도록 안내했지만 별도 Telnet 접속 테스트는 하지 않았다.
- 장비별 비밀번호 값은 이 문서에 포함하지 않는다.

### G. Save, Deferred Work & Next Step

- Core, IT-Switch, HR-Switch에서 `copy running-config startup-config`, Packet Tracer Ctrl+S를 안내했다. 저장 완료 출력과 최신 .pkt 업로드는 별도 확인 필요.
- 기존 Section 6/7의 SSH 예정 항목은 이번 Section 8의 **Core/IT/HR 부분 완료** 상태로 갱신해 읽는다.
- ENG/S/M 등 Access Switch로 SSH 반복 적용은 사용자 결정으로 생략했다.
- 직원 Wi-Fi는 계속 연기한다.
- 다음 작업은 Router0의 show ip interface brief / show running-config를 확인한 뒤 **모의 외부망, Routing, NAT/PAT** 구성이다. 아직 구현 또는 외부 접속 성공으로 기록하지 않는다.


---

## 9. Edge Routing, PAT, DNS & VoIP — 2026-10-27

### Evidence and Current Scope

이 절은 SSH 이후 진행한 작업을 정리한다. **직접 제공된 CLI 출력**과 **사용자의 "다 됐어" 완료 보고**를 구분한다. 최신 .pkt를 직접 실행하거나 미제공 Screenshot/변환표/통화 결과를 새로 검증하지 않았다. Syslog, SNMP, 직원 Wi-Fi, 설정 백업은 아직 미구현이다. Python automation은 사용자의 요청으로 이번 프로젝트 범위에서 제외한다.

### A. Core–Edge Routed Link & Static Routes

기존 Core Gi0/1 Trunk를 Routed Port로 변경하고 기존 2911을 Edge-Router로 사용했다.

| 장비 | Port | IP / Prefix | 용도 |
| :--- | :--- | :--- | :--- |
| Core-Switch | Gi0/1 | 10.0.0.1/30 | Edge로 연결 |
| Edge-Router | Gi0/0 | 10.0.0.2/30 | 내부 방향 |
| Edge-Router | Gi0/1 | 203.0.113.1/24 | 모의 외부 서버 방향 |
| External-Web | FastEthernet0 | 203.0.113.10/24, GW 203.0.113.1 | HTTP 및 외부 DNS |

**왜:** Core는 부서 간 Routing, Edge는 외부 연결/PAT를 담당한다. /30은 두 장비용 주소 2개를 제공한다. 목적지로 가는 길과 응답의 복귀 경로가 모두 필요하다.

Core 변경 기록:
```text
configure terminal
interface gigabitEthernet0/1
 description TO-EDGE-ROUTER
 no switchport
 ip address 10.0.0.1 255.255.255.252
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 10.0.0.2
end
```

Edge 변경 기록:
```text
configure terminal
hostname Edge-Router
interface gigabitEthernet0/0
 description TO-CORE-SWITCH
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit
ip route 192.168.0.0 255.255.255.0 10.0.0.1
ip route 192.168.1.0 255.255.255.0 10.0.0.1

interface gigabitEthernet0/1
 description TO-EXTERNAL-WEB
 ip address 203.0.113.1 255.255.255.0
 no shutdown
exit
end
```

- no switchport: L2 Trunk에서 L3 IP Port로 전환.
- Default Route: 더 구체적인 경로가 없는 목적지를 Edge로 전달.
- Edge의 두 /24 Static Route: 내부 Subnet들을 묶어 Core로 돌려보내는 요약 경로. 단말 Mask를 /24로 바꾸는 명령이 아니다.
- 외부 서버는 Edge에 직접 연결되어 있어 이 모의 외부 대역에 도달하기 위한 Edge Default Route는 필요하지 않다.
- 실제 Edge 출력: C 10.0.0.0/30, L 10.0.0.2/32 및 S 192.168.0.0/24, S 192.168.1.0/24 via 10.0.0.1 확인.
- Edge→Core Ping은 .!!!!, 4/5 응답. 초기 ARP 가능성은 있으나 원인 미확정, 재실행 5/5 출력은 미수집.
- IT PC→Edge와 외부 서버 연결 성공은 사용자 완료 보고 기준이다.
- Guest는 사설 10.0.0.0/8이 ACL로 차단되므로 10.0.0.2 Ping을 Guest 성공 기준으로 삼지 않는다.

### B. PAT & Simulated External Access

External-Web을 Edge Gi0/1에 연결하고 Desktop → IP Configuration에서 위 주소를 설정했다. Services → HTTP → On.

```text
configure terminal
ip access-list standard NAT-INTERNAL
 permit 192.168.0.0 0.0.0.255
 permit 192.168.1.0 0.0.0.255
exit
interface gigabitEthernet0/0
 ip nat inside
exit
interface gigabitEthernet0/1
 ip nat outside
exit
ip nat inside source list NAT-INTERNAL interface gigabitEthernet0/1 overload
end
```

- NAT-INTERNAL은 변환할 Source를 고르는 ACL이며 인터페이스 접근 차단용으로 적용하지 않는다.
- inside/outside는 내부/외부 방향 지정.
- overload는 여러 단말이 외부 포트 주소 203.0.113.1을 공유하도록 한다.
- Inside local은 내부 원래 주소, Inside global은 외부에 보이는 변환 주소, Outside global은 외부 서버 주소.
- 검증 안내: IT 및 Guest에서 ping 203.0.113.10, Browser http://203.0.113.10, Guest→192.168.1.3 차단 유지. 직후 Edge에서 show ip nat translations / show ip nat statistics.
- **결과:** 사용자가 전체 성공을 보고했다. 실제 PAT 변환표 및 통계 출력은 미수집.
- 단순 외부 접속 성공만으로 PAT 변환을 입증하지 않는다. 이 구조는 왕복 Routing만으로도 통신 가능하므로 변환표 증거를 추가 수집해야 한다.
- 203.0.113.0/24는 문서용 주소를 이용한 모의 외부망이다. 실제 ISP/Internet 또는 Google 접속은 구현하지 않았다.

### C. Internal and Guest DNS

DNS는 이름을 IP로 변환한다. A Record는 이름과 IPv4 주소의 연결이며, DHCP DNS Server 값은 단말이 조회할 DNS 서버 위치를 배포한다.

| 구분 | DNS 제공 장비 | DNS Record | DHCP 적용 |
| :--- | :--- | :--- | :--- |
| 직원 내부 | DHCP-Server 192.168.1.2 | intranet.fakecompany.test → 192.168.1.3 | HR/ENG/S/M/IT Data DNS = 192.168.1.2 |
| Guest 외부 | External-Web 203.0.113.10 | www.external.test → 203.0.113.10 | Guests DNS = 203.0.113.10 |

GUI:
1. 해당 Server → Services → DNS → **On**.
2. Name (http:// 없이), Type A Record, Address 입력 후 **Add**하여 목록 확인.
3. DHCP-Server → Services → DHCP → 대상 Pool DNS Server 수정 후 Save.
4. PC/Laptop에서 Static→DHCP 재할당. ipconfig /all로 DNS 주소 확인.
5. nslookup 이름 및 Browser의 전체 http://이름으로 검증.

**Troubleshooting: Hostname unresolved**
- 증상: IT PC가 서버 .1.2에 Ping 가능, DNS 주소도 맞고 http://192.168.1.3 접속은 되지만 이름 접속 실패.
- 범위 좁히기: 기본 IP 통신/HTTP는 동작하므로 DNS 조회 단계 확인. Ping만으로 DNS Service 동작을 입증할 수 없다.
- 실제 원인: 사용자가 DNS Service **Off**였음을 확인.
- 해결: DNS On 전환.
- 결과: 내부 이름 접속 성공 보고. 이후 네 직원 Data Pool과 Guest 외부 DNS/웹 접속, Guest 내부 차단 유지도 완료 보고.
- 실제 nslookup 및 부서별 브라우저 Screenshot은 미수집.
- Guest는 내부 DNS 예외 없이 외부 DNS 사용. 등록된 테스트 이름을 제공하며 실제 공용 DNS 재귀 조회를 구현한 것으로 설명하지 않는다.

### D. VoIP / CME — Eight Phones Across Four Departments

기존 배치는 HR/ENG/S/M/IT 각 7960 전화기 2대, 총 8대이며 Switch→Phone→PC 구조이다. PC Data와 전화 Voice는 같은 물리 연결에서 VLAN으로 구분한다.

| 장비 | 연결 | 최종 IP / Gateway | 역할 |
| :--- | :--- | :--- | :--- |
| Voice-Router 2811 | Fa0/0 ↔ Server-Switch Fa0/3 | 192.168.1.4/28, 기본 경로 192.168.1.1 | CME/SCCP 및 전화 구성 TFTP |

**지원 문제:**
- Edge 2911에서 telephony-service ? 실행 시 % Unrecognized command.
- 실제 사용 중인 Packet Tracer 모델/IOS에서 명령 미지원으로 판단. 실제 2911 전체가 음성 기능 미지원이라는 뜻은 아니다.
- Edge Routing/PAT는 유지하고 별도 2811 추가.
- 새 2811에서 telephony-service ? → <cr> 확인. Enter로 명령 실행 가능한 상태로 지원 여부 검증.

Server-Switch:
```text
configure terminal
interface fastEthernet0/3
 description TO-VOICE-ROUTER
 switchport mode access
 switchport access vlan 50
 spanning-tree portfast
 no shutdown
exit
end
```

Voice-Router의 최종 목표 구성:
```text
configure terminal
hostname Voice-Router
interface fastEthernet0/0
 description TO-SERVER-SWITCH
 ip address 192.168.1.4 255.255.255.240
 no shutdown
exit
ip route 0.0.0.0 0.0.0.0 192.168.1.1

telephony-service
 max-ephones 8
 max-dn 8
 ip source-address 192.168.1.4 port 2000
 auto assign 1 to 8
exit
ephone-dn 1
 number 1001
exit
ephone-dn 2
 number 1002
exit
ephone-dn 3
 number 1003
exit
ephone-dn 4
 number 1004
exit
ephone-dn 5
 number 1005
exit
ephone-dn 6
 number 1006
exit
ephone-dn 7
 number 1007
exit
ephone-dn 8
 number 1008
exit
telephony-service
 create cnf-files
exit
end
```

- CME: 전화기 등록 및 통화 제어 기능.
- max-ephones: 등록 가능한 전화기 수. max-dn: 내선 항목 수.
- ephone-dn / number: 내선 항목과 실제 번호 설정.
- SCCP: 전화기와 CME의 등록/통화 제어 프로토콜. port 2000은 이 제어 연결에 사용.
- create cnf-files: 전화기가 TFTP로 받을 구성 파일 생성.
- 네 Voice Pool 모두 **TFTP Server = 192.168.1.4**로 Save. DHCP Option 150으로 전화기에 구성 서버를 알리는 목적이며 DNS Server 칸과 다르다.
- 기존 DHCP Server의 TFTP Service를 전화용으로 켜는 단계가 아니다. 구성 서버는 Voice-Router .4이다.
- auto assign은 등록 순서대로 배정한다. 1001~1008을 특정 부서/MAC에 고정한 상태는 아니다.

**Troubleshooting: Configuring CM List에서 정지**
1. 증상: 전화 화면이 Configuring CM List에서 대기, show ephone 출력 비어 있음.
2. 진단 증거: Voice-Router Fa0/0가 **192.168.1.3/28**로 설정되어 Web Server와 중복. telephony-service에 **ip source-address 누락**.
3. 해결: Fa0/0를 .4로 수정하고 source-address .4 port 2000 설정, create cnf-files 재실행. 네 Voice DHCP Pool TFTP .4 확인 후 대표 전화기 재시작.
4. 결과: 실제 show ephone에서 HR 전화 등록 확인:
```text
ephone-1 Mac:0060.3E43.7B36
REGISTERED in SCCP ver 12 and Server in ver 8
IP:192.168.0.226
button 1: dn 1 number 1001 CH1 IDLE
```
5. REGISTERED는 CME 등록, IDLE은 통화 대기 상태. 이것만으로 실제 통화 성공을 뜻하지 않으므로 별도로 통화 테스트.
6. HR↔ENG 통화 성공, 나머지 총 8대 등록 및 S/M↔IT 통화 성공은 사용자 완료 보고. 8대 전체 show ephone/전화 화면은 아직 미수집.

### E. Consolidated Evidence Review & Remaining Verification

| 기능 | 직접 수집한 증거 | 완료 보고 / 남은 증거 |
| :--- | :--- | :--- |
| IP 재설계 / Guest ACL | Section 7 DHCP 화면, 단말 IP/Ping, ACL | ENG Pool 58 최종 화면 등 일부 대기 |
| SSH | Section 8 로그인 프롬프트 및 ACL | 일부 최종 config/저장 대기 |
| Core–Edge | Router 경로표, 4/5 Ping | IT PC 성공 보고, 재실행 Ping 출력 대기 |
| 외부망/PAT | 구성 절차 기록 | 성공 보고, 변환표·통계·HTTP 화면 대기 |
| 내부/Guest DNS | DNS Off 원인에 대한 사용자 확인 | 이름 접속 성공 보고, nslookup/화면 대기 |
| VoIP | 2811 명령 지원, 최초 오류 config, HR REGISTERED 출력 | 8대·부서 간 통화 성공 보고, 전체 출력 대기 |
| Syslog/SNMP | 없음 | 미구현 |
| 직원 Wi-Fi | 없음 | 계획 |
| TFTP 설정 백업 | 없음 | 맨 마지막 진행 예정 |

이 표는 현재 증거의 문서 검토 결과이며 새로운 실장비/시뮬레이터 테스트를 대신하지 않는다. 사용자 요청의 최종 검증·증거 수집은 기존 자료 정리까지 수행했고, 추가 실행 및 화면 수집은 최신 .pkt 또는 실행 환경 접근이 필요하다.

후속 수집 명령:
```text
! Core / Edge
show ip interface brief
show ip route
! Edge, generate client traffic first
show ip nat translations
show ip nat statistics
! Voice-Router
show ephone
! Switches
show interfaces trunk
show access-lists
! Clients
ipconfig /all
nslookup intranet.fakecompany.test
! Guest
nslookup www.external.test
```

Syslog는 아직 구성하지 않았으며 수신 로그를 생성/검증했다고 주장하지 않는다. .pkt 저장, 장비 startup-config 저장, 최종 Screenshot 업로드도 별도 확인이 필요하다.

### F. Updated Remaining Scope

- **실제 Internet/google.ca:** Packet Tracer의 일반 시뮬레이션 PC/Router에서 공용 Internet으로 연결하는 방식은 지원하지 않는다. 기존 External-Web은 모의 외부망이다. 실제 접속은 별도 실제/가상 네트워크 환경 과제로 분리해야 한다.
- **직원 Wi-Fi:** VLAN 60, 192.168.2.0/25는 이전 제안일 뿐 아직 미적용. 적용한다면 Core/Trunk/DHCP/접근 정책 외에 Edge 복귀 경로와 NAT 대상에도 새 대역 추가가 필요하다.
- **Monitoring:** Packet Tracer 안에서 Syslog 수신 및 지원 가능한 SNMP 검증부터 범위를 정한다. 외부 PRTG/Zabbix 연동 완료로 표기하지 않는다.
- **설정 백업:** 마지막에 대표 장비 TFTP 수동 백업과 파일 존재 확인.
- **Python automation:** 프로젝트 완료 범위에서 제외.
- 다음 단계마다 용어, 목적, 명령 의미와 실제 검증 기준을 함께 설명한다.

References:
- [Cisco Community — Packet Tracer accessing Internet](https://community.cisco.com/t5/cisco-software-discussions/packet-tracer-accessing-internet/td-p/4722181)


---

## 10. Staff Wi-Fi, Syslog, PAT Evidence & Final Backup — 2026-10-27

**최종 기준:** 이 절은 Section 7~9 이후 변경 사항이다. 이전 표의 직원 Wi-Fi/Syslog 미구현, PAT 출력 대기 상태는 아래 결과로 갱신한다. 기존 주소표와 장애 기록은 이력으로 보존한다.

### A. Staff Wi-Fi — Purpose and Implementation

직원 무선은 내부 업무 서비스와 모의 외부망에 접근하고, Guest는 내부망 접근을 차단한다. 장비 SSH 관리는 기존 IT 유선 대역에만 허용한다.

| 항목 | 최종 구성 |
| :--- | :--- |
| AP / Client | Staff-AP / Staff-Laptop-1 |
| SSID | FakeCompany-Staff |
| 보안 | WPA2-PSK / AES, Guest와 별도 Lab 암호 (실제 값 미기록) |
| Channel | 1 설정 안내 |
| 연결 | IT-Switch Fa0/4 ↔ Staff-AP Port 0 |
| VLAN / Network | 60 / 192.168.2.0/25 |
| Gateway / DNS | 192.168.2.1 / 192.168.1.2 |
| DHCP Pool | Staff-WiFi, Start 192.168.2.10, Mask 255.255.255.128, Max Users 110 |
| DHCP Range | 192.168.2.10~119 |
| 관리 접근 | 기존 SSH-IT-ONLY는 .1.16/28만 허용하므로 직원 Wi-Fi SSH 차단 |

- AP가 IT-Switch에 연결돼도 직원 무선은 VLAN 60이고 IT Data VLAN 40과 별개이다. 물리 스위치 이름보다 포트 VLAN이 논리적 소속을 결정한다.
- AP 포트는 VLAN 60 하나를 전달하므로 Access. Core–IT 연결은 VLAN 40,60,140을 전달하므로 Trunk.
- /25는 Usable 126개이며 단말 70대의 50% 성장 목표와 Gateway를 수용한다. 한 AP가 실제 105명의 사용자를 수용한다는 성능 검증은 아니다.
- 처음 설명에 100대 Pool 예시가 있었으나 실제 안내 최종값은 **110대**이다.
- 사용자별 802.1X/RADIUS 인증, 부서별 무선 동적 VLAN은 미구현이다.
- 직원망의 내부 서비스 접근은 기존 Routing에 기반한다. 별도 세부 서비스 Allow-list ACL을 새로 구축한 것은 아니다.

Core:
```text
configure terminal
vlan 60
 name STAFF-WIFI
exit
interface vlan 60
 description STAFF-WIFI-GATEWAY
 ip address 192.168.2.1 255.255.255.128
 ip helper-address 192.168.1.2
 no shutdown
exit
interface fastEthernet0/4
 switchport trunk allowed vlan add 60
exit
end
```

IT-Switch:
```text
configure terminal
vlan 60
 name STAFF-WIFI
exit
interface gigabitEthernet0/1
 switchport trunk allowed vlan add 60
exit
interface fastEthernet0/4
 description TO-STAFF-AP
 switchport mode access
 switchport access vlan 60
 spanning-tree portfast
 no shutdown
exit
end
```

Edge-Router:
```text
configure terminal
ip route 192.168.2.0 255.255.255.128 10.0.0.1
ip access-list standard NAT-INTERNAL
 permit 192.168.2.0 0.0.0.127
exit
end
```

**왜 Edge도 변경하는가:** 새 .2.x 대역은 기존 .0.x/.1.x 경로와 NAT 대상 밖이므로 복귀 Static Route와 PAT Source 대역을 추가해야 한다. Trunk는 add를 사용해 기존 허용 VLAN을 보존한다.

검증:
- 실제 Wireless0 IP **192.168.2.11** 확인. 제공 ipconfig가 중간에서 끝나 Wireless0 Mask/Gateway/DNS 전체 필드는 직접 수집하지 못했다. Bluetooth 항목의 DNS를 Wireless0 증거로 간주하지 않는다.
- Gateway .2.1, 내부 서버 .1.2, 외부 서버 203.0.113.10에 각각 **2/2 응답**.
- 내부 이름 intranet.fakecompany.test 접속 및 SSH .1.17 차단은 사용자 완료 보고.
- 직원 무선 외부 HTTP의 PAT 변환은 아래 실제 출력으로 추가 확인했다.

### B. Monitoring Scope — Central Syslog Collection

**구현:** Core 이벤트를 DHCP-Server 192.168.1.2의 Services → SYSLOG → On으로 수집한다. 상태 대시보드, SNMP 성능 그래프, 주기적 가용성 검사, 자동 장애 알림 시스템은 이번 범위에 포함하지 않는다.

Core 설정 기록:
```text
configure terminal
service timestamps log datetime msec
logging 192.168.1.2
logging trap debugging
logging on
end
show logging
```

- timestamps는 이벤트 시각 표시, logging IP는 수신 서버 지정.
- debugging 심각도 7은 0~7 로그를 포함한다. 이 설정 자체가 debug 진단 명령을 실행하는 것은 아니다.
- DHCP Lease/주소 할당 실패, 모든 SSH 인증 실패, 모든 ACL 차단을 자동 수집하도록 구성한 상태는 아니다. 서비스별 로그 설정과 지원 여부는 별도이다.
- SYS-5-CONFIG_I는 설정 작업 이벤트이며 실행한 명령/변경 전후 값을 모두 기록하는 감사 이력은 아니다.

**Troubleshooting — logging severity syntax**
1. logging trap informational 입력 시 Invalid input.
2. logging ?에서 trap 하위 명령 존재 확인.
3. 대안으로 안내한 logging trap 6도 실제 CLI에서 거절됨. 숫자 6이 적용됐다고 기록하지 않는다.
4. 사용자가 logging trap debugging을 적용했고 show logging에서 정상 확인.
5. 결론: trap 전체 미지원이 아니라, 이 환경에서 시도한 두 심각도 입력은 실패하고 debugging 형식은 성공했다. 미확인 형식을 지원한다고 단정하지 않는다.

실제 Core 출력:
```text
Syslog logging: enabled
Trap logging: level debugging
Logging to 192.168.1.2 (udp port 514 ... link up)
2 message lines logged
```

**서버 수신 검증:** 제공된 SYSLOG 화면에서 Service On 및 3개 메시지 확인.
- Source/HostName: **192.168.1.1** (Core Server VLAN SVI).
- SYS-5-CONFIG_I 설정 이벤트 2개.
- SYS-6-LOGGINGHOST_STARTSTOP 전송 시작 이벤트 1개.
- 설정 모드 진입 후 end로 나온 다음 새 CONFIG_I 메시지가 서버에 수신됐다.
- 장비 전송 설정만이 아니라 **서버 측 실제 수신**까지 확인했다.
- 서버 화면 날짜는 03.01.1993, Core 출력은 Mar 01로 실제 작업 날짜와 다르다. 시각 동기화는 검증하지 않았다.
- 빈 포트 shutdown 테스트는 안내했으나 실제 실행 증거는 없다. 최종 증거는 설정 이벤트로 수집했다.

### C. Actual PAT Verification

직원 Laptop에서 외부 HTTP 트래픽을 생성한 뒤 사용자가 제공한 Edge 출력:

```text
Pro Inside global      Inside local        Outside local      Outside global
tcp 203.0.113.1:1027   192.168.2.11:1027   203.0.113.10:80    203.0.113.10:80
tcp 203.0.113.1:1031   192.168.1.19:1031   203.0.113.10:80    203.0.113.10:80

Total translations: 2 (0 static, 2 dynamic, 2 extended)
Outside Interfaces: GigabitEthernet0/1
Inside Interfaces: GigabitEthernet0/0
Hits: 20  Misses: 8
Expired translations: 6
```

- 직원 Wi-Fi .2.11 및 IT 유선 .1.19가 외부 IP **203.0.113.1 하나를 공유**하며 HTTP 서버 :80에 접속함을 확인.
- Inside local은 원래 내부 주소, Inside global은 변환 주소.
- Misses 8을 실패 연결 8개 또는 패킷 손실로 해석하지 않는다. 변환 조회의 miss 수치이다.
- 두 연결의 동시 변환표로 PAT 동작을 입증했다. 실제 공용 Internet/Google 접속 증거는 아니다.

### D. Manual Configuration Backup & Save

**목적:** startup-config는 재시작용 내부 저장, TFTP 사본은 설정 복구용 별도 파일이다. 자동화나 실제 복구 완료를 뜻하지 않는다.

백업 서버: **DHCP-Server 192.168.1.2 → Services → TFTP → On**.
전화기 구성 서버 Voice-Router .4의 TFTP와 역할이 다르다.

```text
enable
copy running-config startup-config
copy running-config tftp:
! Remote host: 192.168.1.2
! Destination filename: use device-specific name below
```

| 장비 | 백업 파일명 / 계획 | 확인 상태 |
| :--- | :--- | :--- |
| Core-Switch | Core-Switch-final.cfg | 백업 안내, 개별 전송/파일 출력 미수집 |
| Edge-Router | Edge-Router-final.cfg | 백업 안내, 개별 전송/파일 출력 미수집 |
| Voice-Router | Voice-Router-final.cfg | 백업 안내, 개별 전송/파일 출력 미수집 |
| IT-Switch | IT-Switch-final.cfg | startup-config [OK] 직접 확인, TFTP 대상/파일 안내 후 사용자 완료 보고 |
| HR-Switch / Server-Switch | startup-config 저장 안내 | 개별 완료 출력 미수집 |

- 마지막 "다 됐어"는 직전 IT-Switch TFTP 백업 안내에 대한 완료 보고로 기록한다. 이를 근거로 세 Router/Core 파일도 모두 존재한다고 추정하지 않는다.
- TFTP 서버 파일 목록 Screenshot, 나머지 장비별 전송 성공 출력은 제공되지 않았다.
- 복구 명령 실행 및 복원 후 동작 검증은 하지 않았다.
- Packet Tracer File → Save As → Network-Infrastructure-Final.pkt 저장을 안내했으나 실제 저장 파일/최종 Topology Screenshot은 아직 미수집.
- 같은 .pkt 안의 TFTP 사본은 실습상 별도 장비 백업이다. 호스트 장애에도 살아남는 독립 외부 저장소 백업을 구현한 것은 아니다.
- TFTP는 암호화 전송이 아니므로 실제 운영에서는 백업 경로/저장소 보안과 설정 파일 내 인증 정보 보호를 별도로 고려해야 한다. 문서에는 사용자 비밀번호를 기록하지 않는다.

### E. Final Project Scope & Handoff

**완성 범위:** 대표 단말 기반 기업망 시뮬레이션 — 부서별 Data/Voice VLAN, 주소 용량 재설계, 중앙 DHCP/Relay, 고정 IP 분리, Core Routing, Guest 유선/무선 ACL, 대표 장비 SSH, Edge Static Routing/PAT, 내부/외부 DNS, 8대 VoIP 완료 보고, 직원 Wi-Fi, Core 중앙 Syslog 수신, IT-Switch 수동 TFTP 백업 완료 보고.

**미구현/제외:** 실제 Internet, AD/Windows Server 기능, 모든 스위치의 SSH 반복 적용, Python automation, 외부 PRTG/Zabbix 연동, SNMP 대시보드/알림, 자동 백업, 복구 검증. 실제 VM 모니터링은 향후 별도 Lab으로 분리한다.

**문서 정리 완료와 구분할 마지막 파일 확인:**
- Core/Edge/Voice TFTP 백업을 수행했다면 서버 목록에서 파일 존재 확인.
- 변경한 장비 startup-config 저장 및 최종 .pkt 저장 확인.
- ENG-Data Pool Max Users 58, DHCP Server NIC /28, 필요 시 최종 전체 SVI/Trunk 상태 확인.
- 사진/출력이 없는 성공 항목은 사용자 완료 보고 상태를 유지한다.

사용자에게 추가 기능 확장을 요구하지 않고, 남은 항목은 최종 보관/증거 확인으로 남긴다.
