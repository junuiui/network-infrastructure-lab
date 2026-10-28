# Guest & Staff Wi-Fi

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)

> 기준: 2026-10-27. 설정 절차와 실제 검증 상태를 구분한다.


## 두 SSID의 역할

| 항목 | Guest | Staff |
| :--- | :--- | :--- |
| SSID | FakeCompany-Guest | FakeCompany-Staff |
| AP | Guest-AP | Staff-AP |
| 연결 | Guest-Switch Fa0/3 | IT-Switch Fa0/4 |
| VLAN | 99 | 60 |
| 대역 | 192.168.1.192/26 | 192.168.2.0/25 |
| Gateway | 192.168.1.193 | 192.168.2.1 |
| DNS | 203.0.113.10 | 192.168.1.2 |
| 내부 서비스 | 차단 | 접근 가능 |
| 장비 SSH | 차단 | IT 전용 VTY ACL로 차단 |
| 보안 / 채널 | WPA2-PSK AES / 6 | WPA2-PSK AES / 1 |

## Guest AP 연결과 단말 설정

Guest-Switch Fa0/3을 Access VLAN 99로 설정하고 AP Port 0에 연결한다. AP Config에서 유선 Port 0/무선 Port 1 On, SSID, WPA2-PSK, AES와 Lab 암호를 설정한다.

Laptop은 전원을 끈 뒤 WPC300N을 장착하고 다시 켠다. Desktop → PC Wireless에서 SSID/암호로 연결, IP Configuration의 Wireless0에서 DHCP 선택. 단일 SSID/VLAN AP 연결이므로 AP 포트는 Trunk가 아니다.

Guest는 DHCP .1.194~254를 수신해야 한다. 기존 문서의 .0.194~221은 변경 전 기록이며 사용하지 않는다.

## Staff Wi-Fi — Purpose and Implementation

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

## 테스트 해석

Guest Wireless0 .1.196, Staff Wireless0 .2.11과 각 Gateway/서버 Ping 출력이 수집됐다. Guest는 내부 Ping 실패, Staff는 내부 Ping 성공이 정상이다. Staff 내부 DNS 및 SSH 차단은 완료 보고 기준이다.

기업 무선에서 사용하는 802.1X, RADIUS, 로밍, 무선 용량/간섭 측정은 이번에 구현하지 않았다. 대표 Laptop 기능 테스트이며 AP 한 대의 실제 직원 수용 성능을 입증하지 않는다.

[실제 검증](validation.md) · [재구축 순서](guide.md)


---

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)
