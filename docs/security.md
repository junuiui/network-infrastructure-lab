# Guest ACL & SSH Management

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)

> 기준: 2026-10-27. 설정 절차와 실제 검증 상태를 구분한다.


## Guest 접근 제어

VLAN 99 inbound에 GUEST-IN을 적용한다. DHCP와 Guest Gateway Ping은 허용하고 사설망 목적지는 차단한다. 나머지 외부 목적지를 허용하되 Routing/PAT가 따로 필요하다.

```text
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
```

ACL은 위에서부터 처음 일치한 규칙을 적용한다. 같은 Guest VLAN 안의 통신 차단/무선 Client Isolation, Stateful Firewall은 구현하지 않았다. Guest DNS는 외부 203.0.113.10을 사용하며 내부 DNS 예외가 없다. NAT 대상 ACL은 접근 차단 ACL과 목적이 다르다.

## What / Why / Scope

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

## Management Address & Access Policy

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

## Configuration Procedure & Commands

장비의 직접 CLI에서 설정한다. 아래 명령은 안내한 구성의 재현 기록이며, 실행 전 기존 계정/VTY/RSA 설정을 확인한다. `<LAB_PASSWORD>`는 실제로 입력할 문자열이 아니라 사용자가 정한 Lab 전용 비밀번호를 뜻한다.

### Core-Switch

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

### IT-Switch

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

### HR-Switch

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

## Command Meaning & Design Understanding

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

## 직원 Wi-Fi의 관리 제한

직원 Wi-Fi .2.0/25는 SSH-IT-ONLY의 허용 범위 밖이다. 업무 서비스에 접근해도 장비 SSH는 차단되는 정책으로 테스트 완료 보고를 받았다.

비밀번호는 장비별로 독립적이다. 문서의 <LAB_PASSWORD>는 직접 정한 값을 대체 입력하는 표시이며 실제 비밀번호가 아니다. 설정 실패 사례와 ACL 카운터는 [Troubleshooting](troubleshooting.md), [Validation](validation.md)에 있다.


---

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)
