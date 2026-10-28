# VoIP & Call Manager Express

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)

> 기준: 2026-10-27. 설정 절차와 실제 검증 상태를 구분한다.

## VoIP / CME — Eight Phones Across Four Departments

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

## 검증 상태와 부서별 주소

| 부서 | Voice VLAN | 전화기 DHCP 범위 |
| :--- | :--- | :--- |
| HR | 110 | 192.168.0.226~254 |
| ENG | 120 | 192.168.0.66~125 |
| S/M | 130 | 192.168.1.130~190 |
| IT | 140 | 192.168.1.34~46 |

8대 등록 및 HR↔ENG, S/M↔IT 통화는 사용자 완료 보고이다. HR 1001의 REGISTERED는 실제 출력으로 확인했다. 자동 번호 배정이며 부서별 번호 고정은 미구현이다.

[등록 문제 해결](troubleshooting.md) · [새로 구성하기](guide.md)


---

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)
