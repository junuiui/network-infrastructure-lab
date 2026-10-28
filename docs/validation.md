# Validation & Evidence

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)

## 결과를 읽는 기준

| 표시 | 의미 |
| :--- | :--- |
| 출력 확인 | 사용자가 보낸 CLI 텍스트/화면으로 해당 값 확인 |
| 완료 보고 | 사용자가 성공을 보고했으나 개별 원본 출력/화면은 미수집 |
| 확인 대기 | 설정 목표/안내만 있고 해당 결과는 확인하지 못함 |

문서를 재구성하면서 .pkt를 새로 실행한 것은 아니다. 아래 테스트는 기존 실습 증거이다. Ctrl+C로 중단한 Ping은 실제 송수신 수만 기록한다.

## Actual Verification Matrix

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

위 초기 표의 Guest 무선은 출력으로 확인됐고 Voice 전체는 아래 완료 보고 상태이다. S/M Web 첫 timeout 이후 응답은 확인됐으나 별도 연속 4회 성공 출력은 없다.

## Actual SSH Verification Results

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

## Routing, DNS, VoIP, Staff Wireless

| 기능 | 증거 / 결과 | 구분 |
| :--- | :--- | :--- |
| Core–Edge | Edge→Core 4/5, Connected/Local 및 두 내부 Static Route | 출력 확인 |
| IT PC→Edge, 외부 연결 | 성공 | 완료 보고 |
| 내부 DNS | DNS Off 원인 확인 후 직원 네 Data VLAN 이름 접속 성공 | 완료 보고 |
| Guest 외부 DNS/HTTP | 외부 이름 접속 성공, 내부 차단 유지 | 완료 보고 |
| HR Phone | .0.226, MAC 0060.3E43.7B36, 1001 REGISTERED | 출력 확인 |
| VoIP 전체 | 8대 등록, HR↔ENG 및 S/M↔IT 통화 | 완료 보고 |
| 직원 Wireless0 | .2.11; Gateway .2.1, 서버 .1.2, 외부 .113.10 각각 2/2 | 출력 확인 |
| 직원 DNS/관리 제한 | 내부 이름 접속 성공, SSH .1.17 차단 | 완료 보고 |

## Actual PAT Evidence

## Actual PAT Verification

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

## Syslog Receiver Evidence

Core show logging은 수신 서버 192.168.1.2, UDP 514, debugging, 전송 대상 2줄 기록을 표시했다. 서버 SYSLOG 화면에는 Source 192.168.1.1에서 온 CONFIG_I 2개와 LOGGINGHOST_STARTSTOP 1개가 표시됐다. 장비 시각은 실제 날짜와 동기화하지 않았다.

## Backup and Saved Artifacts

| 항목 | 상태 |
| :--- | :--- |
| IT-Switch startup-config | [OK] 출력 확인 |
| IT-Switch TFTP | 서버/파일명 안내 후 완료 보고 |
| Core/Edge/Voice TFTP | 개별 전송 결과/목록 확인 대기 |
| final .pkt | 저장소 [fake-company-final.pkt](../fake-company-final.pkt) 존재 확인 |
| .pkt 재실행 / TFTP 복원 | 미실행 |

## 최종 확인 체크리스트

- [ ] ENG-Data DHCP Max Users가 58인지 마지막 화면 확인.
- [ ] DHCP-Server NIC Mask /28, 일부 Printer/최종 config 추가 출력 확인.
- [ ] Core/Edge/Voice TFTP 사본 존재 확인.
- [ ] 변경 장비 startup-config 및 최종 .pkt 상태 확인.
- [ ] 필요 시 전화기 8대 등록 출력/부서별 실제 번호 수집.

위 미체크 항목은 기능 미동작 판정이 아니라 **미수집 증거/검증 범위**를 뜻한다.

## 재실행 절차

| 대상 | 명령 / 동작 | 기대 |
| :--- | :--- | :--- |
| 직원 PC | ipconfig /all; 내부 이름 HTTP | 부서 IP/Mask/GW/DNS, 페이지 열림 |
| Guest | Gateway Ping / 내부 서버 Ping | 성공 / 차단 |
| IT PC | SSH Core .1.17, IT .1.28, HR .0.220 | 로그인 |
| 비IT PC | 위 SSH | 차단 |
| Staff Laptop | 내부 DNS/HTTP, 외부 HTTP, Core SSH | 성공, 성공, 차단 |
| Edge | show ip route / show ip nat translations / statistics | 복귀 경로와 PAT 항목 |
| Voice | show ephone / 타 부서 내선 호출 | 등록 및 통화 |
| Core/서버 | 설정 이벤트 → SYSLOG | 새 이벤트 수신 |
| TFTP | 장비 copy 실행 → 서버 파일 목록 | 성공 및 파일 존재 |

---

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)
