# Step-by-Step Rebuild Guide

[← 메인 README](../readme.md) · [주소표](addressing.md) · [문제 해결](troubleshooting.md) · [검증 결과](validation.md)

> **용도:** 빈 Packet Tracer 파일에서 최종 설계를 재구축하는 절차. 기존 파일의 주소 변경 명령을 다시 적용하는 Migration Guide가 아니다. 실제 기존 장비에 그대로 덮어쓰지 않는다.
>
> 최종 명령을 기존 실습 기록에서 정리했으며 이 문서 전체를 새 .pkt에서 다시 실행한 것은 아니다. 모델/버전에 따라 CLI가 다르면 `?`로 지원 형식을 확인한다. 비밀번호는 직접 정한 Lab 값으로 교체한다.

## 0. 준비와 진행 원칙

- Core 3560-24PS 1대, Access 2960 6대(HR/ENG/SM/IT/Server/Guest).
- Edge 2911 1대, Voice 2811 1대.
- Server-PT 3대(DHCP/DNS/Syslog/TFTP, 내부 Web, 외부 Web/DNS).
- 부서별 대표 Phone 7960 2대 및 PC, Printer. Guest 유선 PC와 Laptop.
- AccessPoint-PT 2대, Wireless Laptop 2대(Guest/Staff). Laptop 무선 모듈 WPC300N.
- 한 단계를 적용하고 확인한 다음 진행한다. `>`에서 enable, `#`에서 configure terminal, `(config)#`에서 설정 입력. `end`는 #로 돌아간다.

실제 목표 수량과 대표 배치는 다르다. [Architecture](architecture.md)와 [Addressing](addressing.md)을 먼저 읽는다.

## 1. 장비 배치와 배선

[포트 맵](architecture.md)을 따라 Core Fa0/1~6을 각 Access Gi0/1에 연결한다.
Core Gi0/1 ↔ Edge Gi0/0, Edge Gi0/1 ↔ External-Web.
Server Fa0/1 ↔ DHCP, Fa0/2 ↔ Web, Fa0/3 ↔ Voice Fa0/0.
Guest Fa0/3 ↔ Guest-AP Port 0, IT Fa0/4 ↔ Staff-AP Port 0.

부서 Fa0/1~2는 Phone의 Switch 포트, Phone의 PC 포트는 PC로 연결한다. Fa0/3은 Printer. 전화 전원은 모델의 PoE 지원 또는 Physical 탭 전원 어댑터로 공급한다. 자동 케이블 선택을 이용해도 된다.

## 2. Core VLAN, Trunk, Gateway 및 DHCP Relay

- VLAN: 부서/역할별 L2 Broadcast 구역.
- Trunk: 케이블 하나에 여러 VLAN 전달.
- SVI: VLAN의 L3 인터페이스이며 이번에는 단말 Gateway.
- ip routing: Core가 VLAN 간 패킷 전달.
- ip helper-address: DHCP Broadcast를 중앙 서버로 Relay.

Core CLI에 입력한다. 기존 실습의 Fa0/1~3 협상 Trunk 대신 재구축에서는 필요한 VLAN만 명시한다.

```text
enable
configure terminal
hostname Core-Switch
ip routing

vlan 10
 name HR-DATA
exit
vlan 110
 name HR-VOICE
exit
vlan 20
 name ENG-DATA
exit
vlan 120
 name ENG-VOICE
exit
vlan 30
 name SM-DATA
exit
vlan 130
 name SM-VOICE
exit
vlan 40
 name IT-DATA
exit
vlan 140
 name IT-VOICE
exit
vlan 50
 name SERVER
exit
vlan 99
 name GUEST
exit
vlan 60
 name STAFF-WIFI
exit

interface fastEthernet0/1
 description TO-HR-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 10,110
 no shutdown
exit
interface fastEthernet0/2
 description TO-ENG-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 20,120
 no shutdown
exit
interface fastEthernet0/3
 description TO-SM-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 30,130
 no shutdown
exit
interface fastEthernet0/4
 description TO-IT-SWITCH
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk allowed vlan 40,60,140
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

interface vlan 10
 ip address 192.168.0.193 255.255.255.224
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 110
 ip address 192.168.0.225 255.255.255.224
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 20
 ip address 192.168.0.1 255.255.255.192
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 120
 ip address 192.168.0.65 255.255.255.192
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 30
 ip address 192.168.1.65 255.255.255.192
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 130
 ip address 192.168.1.129 255.255.255.192
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 40
 ip address 192.168.1.17 255.255.255.240
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 140
 ip address 192.168.1.33 255.255.255.240
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 50
 ip address 192.168.1.1 255.255.255.240
 no shutdown
exit
interface vlan 99
 ip address 192.168.1.193 255.255.255.192
 ip helper-address 192.168.1.2
 no shutdown
exit
interface vlan 60
 ip address 192.168.2.1 255.255.255.128
 ip helper-address 192.168.1.2
 no shutdown
exit
end
```

**확인:** `show vlan brief`, `show ip interface brief`, `show interfaces trunk`.
아직 해당 VLAN의 활성 연결이 없으면 SVI가 down일 수 있다. 모든 단말을 연결한 후 필요한 SVI up/up을 확인한다.

## 3. 부서 Access Switch 설정

Access/Data VLAN은 PC, Voice VLAN은 전화기에 사용한다. Printer 포트에는 Voice VLAN을 추가할 필요가 없다.

### 3.1 HR-Switch
```text
enable
configure terminal
hostname HR-Switch
vlan 10
exit
vlan 110
exit
interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 10
 switchport voice vlan 110
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/3
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
exit
interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 10,110
 no shutdown
exit
end
```

### 3.2 ENG-Switch
```text
enable
configure terminal
hostname ENG-Switch
vlan 20
exit
vlan 120
exit
interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 20
 switchport voice vlan 120
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/3
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
exit
interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 20,120
 no shutdown
exit
end
```

### 3.3 SM-Switch
```text
enable
configure terminal
hostname SM-Switch
vlan 30
exit
vlan 130
exit
interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 30
 switchport voice vlan 130
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/3
 switchport mode access
 switchport access vlan 30
 spanning-tree portfast
 no shutdown
exit
interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 30,130
 no shutdown
exit
end
```

### 3.4 IT-Switch
```text
enable
configure terminal
hostname IT-Switch
vlan 40
exit
vlan 140
exit
vlan 60
exit
interface range fastEthernet0/1 - 2
 switchport mode access
 switchport access vlan 40
 switchport voice vlan 140
 spanning-tree portfast
 no shutdown
exit
interface fastEthernet0/3
 switchport mode access
 switchport access vlan 40
 spanning-tree portfast
 no shutdown
exit
interface gigabitEthernet0/1
 switchport mode trunk
 switchport trunk allowed vlan 40,60,140
 no shutdown
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

### 3.5 Server-Switch
```text
enable
configure terminal
hostname Server-Switch
vlan 50
exit
interface range fastEthernet0/1 - 3
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
```

### 3.6 Guest-Switch
```text
enable
configure terminal
hostname Guest-Switch
vlan 99
exit
interface range fastEthernet0/1 - 3
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
```

**확인:** 각 Switch에서 show vlan brief / show interfaces trunk. Phone 연결 Port는 Data/Voice VLAN을 `show interfaces fastEthernet0/1 switchport`로 확인한다.

## 4. 서버와 Printer 고정 IP

Server의 Desktop → IP Configuration → Static에서 설정한다.

| 장비 | IP | Mask | Gateway |
| :--- | :--- | :--- | :--- |
| DHCP-Server | 192.168.1.2 | 255.255.255.240 | 192.168.1.1 |
| Web-Server | 192.168.1.3 | 255.255.255.240 | 192.168.1.1 |
| HR Printer | 192.168.0.221 | 255.255.255.224 | 192.168.0.193 |
| ENG Printer | 192.168.0.62 | 255.255.255.192 | 192.168.0.1 |
| S/M Printer | 192.168.1.126 | 255.255.255.192 | 192.168.1.65 |
| IT Printer | 192.168.1.30 | 255.255.255.240 | 192.168.1.17 |

Printer는 Config의 해당 Ethernet/Settings 화면에서 입력한다. .1.6은 기본 serverPool 전용으로 비워두고 고정 서버에 할당하지 않는다. HR .220과 IT .28도 나중 관리 SVI용으로 비워둔다.

Web-Server → Services → HTTP → On. DNS를 쓰기 전에 IP로 페이지가 열리는지 확인한다.

## 5. 중앙 DHCP Pool

DHCP-Server → Services → DHCP → On.
[최종 Pool 표](addressing.md)의 Start/Mask/Gateway/Max Users를 그대로 입력하고 신규 Pool은 Add, 기존 Pool 변경은 Save한다.

- 직원 HR/ENG/S/M/IT Data 및 Staff-WiFi DNS: **192.168.1.2**.
- Guests DNS: **203.0.113.10**.
- 네 Voice Pool TFTP Server: **192.168.1.4**.
- 기본 serverPool은 제거 가능하면 제거. 이 Lab과 동일하게 유지한다면 Start .1.6, Mask /28, GW .1.1, Max 1로 제한한다.
- ENG-Data는 Max **58** (.2~59)로 설정해 Printer .62와 분리한다.
- Staff-WiFi는 Start .2.10, Mask /25, GW .2.1, Max **110**.

PC Desktop → IP Configuration → DHCP. 기존 설정이 남았으면 Static을 선택했다가 DHCP를 선택한다.
부서별 PC가 [주소표](addressing.md)의 IP/Mask/GW를 받는지, 자신의 Gateway와 .1.2에 Ping 되는지 확인한다.

## 6. Guest ACL

Core에서 적용한다. DHCP 허용이 내부 차단보다 앞에 있어야 한다.

```text
configure terminal
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
exit
end
```

Guest는 .1.193 Ping 성공, .1.2/.1.3 Ping 실패가 정상. Core show access-lists에서 deny 카운터를 확인한다. 실패만 보고 ACL 동작으로 단정하지 말고 Gateway 정상과 카운터를 함께 본다.

## 7. Core–Edge Routing 및 모의 외부 서버

Core는 외부로 기본 경로, Edge는 내부로 복귀 경로가 필요하다. Core Gi0/1은 Trunk가 아니라 Routed Port로 만든다.

### Core
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

### Edge
```text
enable
configure terminal
hostname Edge-Router
interface gigabitEthernet0/0
 description TO-CORE-SWITCH
 ip address 10.0.0.2 255.255.255.252
 no shutdown
exit
interface gigabitEthernet0/1
 description TO-EXTERNAL-WEB
 ip address 203.0.113.1 255.255.255.0
 no shutdown
exit
ip route 192.168.0.0 255.255.255.0 10.0.0.1
ip route 192.168.1.0 255.255.255.0 10.0.0.1
ip route 192.168.2.0 255.255.255.128 10.0.0.1
end
```

External-Web → Desktop → IP Configuration: .113.10 /24, Gateway .113.1 (전체 주소 203.0.113.x).
Services → HTTP → On.

**검증:** Edge ping 10.0.0.1 및 203.0.113.10; IT PC ping 10.0.0.2. show ip route에서 C/L/S 확인. Guest의 10.0.0.2 Ping은 ACL 차단 대상이다.

## 8. PAT

여러 내부 단말이 Edge의 외부 주소 하나를 공유하도록 한다. 이 ACL은 허용/차단 정책이 아니라 변환할 Source 선택이다.

```text
configure terminal
ip access-list standard NAT-INTERNAL
 permit 192.168.0.0 0.0.0.255
 permit 192.168.1.0 0.0.0.255
 permit 192.168.2.0 0.0.0.127
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

직원/Guest Browser에서 http://203.0.113.10을 열고 Edge에서 바로 확인한다.
```text
show ip nat translations
show ip nat statistics
```

Inside local은 내부 주소, Inside global은 203.0.113.1이어야 한다. 외부 페이지 성공만으로 변환을 입증하지 말고 표까지 확인한다.

## 9. 내부 및 외부 DNS

각 서버 Services → DNS → **On**, Type A Record 입력 후 **Add**한다. 입력칸에만 남아 있으면 등록된 것이 아니다.

| 서버 | Name | Address |
| :--- | :--- | :--- |
| DHCP-Server .1.2 | intranet.fakecompany.test | 192.168.1.3 |
| External-Web .113.10 | www.external.test | 203.0.113.10 |

직원/Guest DHCP 갱신 후 ipconfig /all로 DNS 확인. 직원은 내부 이름 HTTP, Guest는 외부 이름 HTTP 성공 및 내부 서버 Ping 차단을 검증한다.
```text
nslookup intranet.fakecompany.test
```
이 명령은 직원 PC에서 실행한다. Guest에서는 www.external.test를 조회한다. IP HTTP는 되는데 이름만 실패하면 DNS 서비스 On/서버 주소/레코드를 차례로 확인한다.


## 10. 대표 장비 SSH

먼저 직접 CLI에서 설정한다. RSA 생성 전에 hostname/domain-name이 필요하다. <LAB_PASSWORD>는 장비별 Lab 암호로 바꾼다.

### 10.1 관리 IP

Core는 기존 VLAN 40 .1.17을 사용한다.

IT-Switch:
```text
configure terminal
interface vlan 40
 ip address 192.168.1.28 255.255.255.240
 no shutdown
exit
ip default-gateway 192.168.1.17
end
```

HR-Switch:
```text
configure terminal
interface vlan 10
 ip address 192.168.0.220 255.255.255.224
 no shutdown
exit
ip default-gateway 192.168.0.193
end
```

L2 Switch의 관리 SVI는 스위치 자체의 주소이며 PC Gateway를 대체하지 않는다. default-gateway는 다른 Subnet의 IT 관리자에게 응답하는 경로이다.

### 10.2 각 장비 계정과 SSH

Core/IT/HR 각각 실행한다. hostname은 앞 단계에서 설정했다.

```text
configure terminal
ip domain-name fakecompany.lab
username netadmin privilege 15 secret <LAB_PASSWORD>
ip access-list standard SSH-IT-ONLY
 permit 192.168.1.16 0.0.0.15
 deny any
exit
crypto key generate rsa
```

크기를 물으면 2048 입력. 기존 키가 있다면 재생성 전 확인한다. 이 가이드는 새 장비 기준이다.

IT/HR:
```text
ip ssh version 2
line vty 0 15
 login local
 transport input ssh
 access-class SSH-IT-ONLY in
 exec-timeout 5 0
exit
end
```

Core에서는 아래와 같이 수집된 구간인 **line vty 0 4**에 적용한다.

```text
ip ssh version 2
line vty 0 4
 login local
 transport input ssh
 access-class SSH-IT-ONLY in
 exec-timeout 5 0
exit
end
```

Core show running-config에 추가 VTY 구간이 있으면 그 구간에도 동일 설정을 적용한다.

- login local은 해당 장비의 계정 인증.
- transport input ssh는 SSH만 허용.
- VTY는 원격 접속 자리이며 0~15는 16개 라인이다.
- ACL Source는 **192.168.1.16/28**이다. .0/28을 넣으면 IT PC가 차단된다.

검증:
```text
show ip ssh
show access-lists
```
IT PC에서 ssh -l netadmin 192.168.1.17, .1.28, .0.220 로그인. HR/S/M/Staff에서 같은 접속은 차단. 비밀번호는 화면에 표시되지 않는다. 실제 암호/해시를 증거 문서에 붙이지 않는다.

## 11. Guest 및 Staff Wi-Fi

AccessPoint-PT 두 대의 Config에서 Port 0 On, 무선 Port 1 On으로 설정한다.

| 설정 | Guest | Staff |
| :--- | :--- | :--- |
| SSID | FakeCompany-Guest | FakeCompany-Staff |
| Channel | 6 | 1 |
| Authentication / Encryption | WPA2-PSK / AES | WPA2-PSK / AES |
| Password | Lab 전용 암호 | Guest와 다른 Lab 암호 |
| 연결 포트 | Guest Fa0/3 Access 99 | IT Fa0/4 Access 60 |

위 VLAN/Trunk 설정은 단계 2~3에 포함했다. 기존 파일에 Staff VLAN을 추가할 때는 Core Fa0/4와 IT Gi0/1 양쪽에 switchport trunk allowed vlan **add 60**을 사용해 기존 40,140을 보존한다.

Laptop Physical에서 전원 Off → 유선 모듈 제거 → WPC300N 장착 → On.
Desktop → PC Wireless → Connect/Refresh → 해당 SSID → 암호 입력.
Desktop → IP Configuration → Wireless0 → DHCP.

| 검증 | Guest | Staff |
| :--- | :--- | :--- |
| 주소 | .1.194~254 /26 | .2.10~119 /25 |
| Gateway | .1.193 | .2.1 |
| DNS | 203.0.113.10 | 192.168.1.2 |
| 내부 .1.3 Ping | 차단 | 성공 |
| 외부 HTTP | 성공 | 성공 |
| Core SSH | 차단 | 차단 |

Staff는 내부 이름 HTTP도 확인한다. 양쪽의 DHCP/Wireless0 전체 출력과 AP 설정은 별도로 수집한다.

## 12. Voice-Router / 전화 8대

Edge 2911에서 전화 기능을 강제로 설정하지 않는다. 별도 2811에서 telephony-service ?가 <cr>을 반환하는지 확인한다.

Voice-Router:
```text
enable
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

- CME는 전화 등록/제어.
- ip source-address는 전화기가 접속할 제어 주소, port 2000은 SCCP.
- ephone-dn은 내선 항목, number는 번호.
- auto assign은 등록 순서에 따라 자동 배정하므로 부서와 번호를 고정하지 않는다.
- create cnf-files는 전화기 구성 파일 생성.

네 Voice Pool의 **TFTP Server .1.4**를 다시 확인한다. Phone 전원을 재시작해 새 DHCP/구성을 받는다.
```text
show ephone
```

8대 REGISTERED와 부서별 Voice IP 확인. HR↔ENG 및 S/M↔IT의 **실제로 표시된 내선**을 호출하고 상대가 수화기를 들어 연결/종료한다.
Configuring CM List에서 멈추면 .1.4가 아닌 Web .1.3 중복, CME source-address 누락, DHCP TFTP 값부터 확인한다.

## 13. 중앙 Syslog

DHCP-Server → Services → SYSLOG → On.

Core:
```text
configure terminal
service timestamps log datetime msec
logging 192.168.1.2
logging trap debugging
logging on
end
show logging
```

이 환경에서 informational 및 숫자 6 입력은 거절됐고 debugging은 작동했다.
debugging(7)은 0~7 수준을 전송하며 debug 진단을 실행하는 명령은 아니다.

테스트:
```text
configure terminal
end
```
서버 SYSLOG 화면에 새 CONFIG_I 이벤트가 들어오는지 확인한다. Core show logging의 전송 기록만으로 수신까지 확정하지 않는다.
이벤트 수집 범위이며 성능 그래프/자동 알림이 아니다. 장비 시각이 실제 날짜와 다를 수 있다.

## 14. 저장과 TFTP 백업

DHCP-Server → Services → TFTP → On.
변경한 각 Switch/Router에서:
```text
copy running-config startup-config
```

대표 장비 Core/Edge/Voice/IT에서:
```text
copy running-config tftp:
```
Remote host는 192.168.1.2. 파일명은 장비별로 구분한다.

| 장비 | 파일명 |
| :--- | :--- |
| Core | Core-Switch-final.cfg |
| Edge | Edge-Router-final.cfg |
| Voice | Voice-Router-final.cfg |
| IT | IT-Switch-final.cfg |

전송 완료 메시지와 서버 TFTP 파일 목록을 확인한다. 내부 저장 성공과 원격 전송 성공은 별개이다. 실제 복구/덮어쓰기는 이 가이드의 필수 단계가 아니다.

File → Save As로 최종 .pkt 저장. 저장소의 기존 최종 파일명은 [fake-company-final.pkt](../fake-company-final.pkt)이다. 새 실습은 별도 이름으로 저장해 원본을 보존한다.

## 15. 최종 검증 및 보관

1. 모든 대표 단말의 IP/Mask/GW/DNS와 Printer 중복 방지 확인.
2. 직원은 내부 HTTP 성공, Guest는 외부 HTTP 성공/내부 차단.
3. IT SSH 허용, 비IT SSH 차단 및 ACL 카운터 확인.
4. Edge 변환표에서 IT/Staff 주소의 PAT 확인.
5. 전화 8대 등록 및 부서 간 통화 확인.
6. Syslog 서버의 새 이벤트 수신 확인.
7. TFTP 파일 존재, startup-config, 최종 .pkt 저장 확인.
8. 실제 출력/화면을 [검증 표](validation.md)와 비교해 기록.

이 문서의 기대 결과는 새 재구축의 합격 기준이다. 기존 프로젝트의 완료 증거와 동일시하지 않는다.
기존 주소 변경 및 실패 이력은 [Addressing](addressing.md)과 [Troubleshooting](troubleshooting.md)에 있다.

---

[← 메인 README](../readme.md) · [구성도](architecture.md) · [검증 결과](validation.md)
