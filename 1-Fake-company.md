# Corporate Network Setup & Troubleshooting Guide

> **Last updated: 2026-10-26.** 최신 작업 상태, 명령어, Guest Wi-Fi 절차와 남은 작업은 [Section 5](#5-implementation-log--2026-10-26) 및 [Section 6](#6-remaining-work--next-session)을 참고한다. Section 1의 수량은 최종 설계 목표이며, Section 3~4는 초기 구성 및 장애 해결 기록이다.

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
