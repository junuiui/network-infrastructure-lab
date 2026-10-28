# Troubleshooting Journal

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)

> 실제 증상 → 진단 → 조치 → 검증 순서로 보존한다. 주소 부족이 설계에서 확인된 경우와 실제 서비스 장애를 구분한다.

## 초기 DHCP 장애 — 2026-10-26


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

### 당시 결과 기록
* 모든 SVI 게이트웨이가 올바르게 활성화된 후, 클라이언트 PC의 DHCP 요청 패킷이 L3 스위치를 거쳐 `192.168.1.2` DHCP 서버로 정확하게 Unicast Relay됨.
* 각 부서 PC에서 정상적으로 VLSM 대역별 IP, Subnet Mask, Gateway 주소를 할당받음 성공.

위 초기 결과는 당시 문서의 완료 기록이다. 변경 후 검증은 [Validation](validation.md)에 구분해 정리한다.

## Troubleshooting Details — Problem → Attempt → Verification

### Issue 1: Capacity calculation and DHCP/Subnet mismatch risk

- **문제 발견:** Server/IT의 /29는 사용 가능 6개 중 Gateway가 1개를 차지해 단말은 5대만 수용한다. DHCP Maximum Users만 늘려도 Subnet 용량 자체는 늘지 않는다.
- **첫 해결 시도:** 필요한 구역의 DHCP Mask와 Pool 범위를 확대하고 Core SVI를 같은 Mask/대역으로 변경한다.
- **추가 조치:** 겹치는 Subnet을 이동하고 Printer 고정 IP와 Pool을 분리한다. 서버 NIC Mask도 수정한다.
- **검증:** 아래 Test Matrix의 실제 출력으로 DHCP 재할당 및 대표 경로 통신을 확인했다.
- **한계:** 최대 단말 수까지 Lease를 소진하는 부하 테스트는 하지 않았다. 주소 용량은 산술 검토, 실제 동작은 대표 단말로 검증했다.

### Issue 2: Overlapping subnet rejected by Core

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

### Issue 3: Old DHCP address remains after migration

- **관찰:** Guest PC의 첫 ipconfig는 192.168.0.195/27, GW 192.168.0.193이었다. 이후 출력은 192.168.1.195/26, GW 192.168.1.193으로 변경됐다.
- **원인 해석:** 기존 Lease/단말 설정이 남아 있을 수 있다. 첫 출력만으로 새 DHCP Pool이 잘못됐다고 단정하지 않는다.
- **해결 절차:** DHCP Pool Save와 SVI 변경 후 단말에서 Static → DHCP로 재요청한다.
- **검증:** Guest 유선 .1.195 및 무선 .1.196이 새 /26과 Gateway를 수신했다.

### Issue 4: Guest gets DHCP but cannot Ping the DHCP Server

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

### Issue 5: ENG Printer static IP overlaps the DHCP range

- **기존 위험:** 초기 기록의 Printer .0.4가 DHCP .0.2~61 안에 있어 같은 주소를 다른 Client에 할당할 가능성이 있었다. 실제 중복 Lease 발생 출력은 수집하지 않았다.
- **해결 안내:** ENG-Data Maximum Users를 60→58로 줄여 .2~59만 할당하고 .60~62를 고정 IP용으로 분리한다. Printer를 .62/26, GW .1로 변경한다.
- **검증:** ENG PC에서 Printer .62에 3/3, DHCP Server .1.2에 2/2 응답.
- **남은 확인:** 마지막 Pool 화면은 변경 전 Max Users 60이었다. Printer 응답만으로 Pool이 58로 저장됐다고 입증할 수 없으므로 최종 화면 재확인이 필요하다.

### Issue 6: First Web Server Ping timeout and invalid test targets

- S/M PC→Web .1.3 테스트는 첫 요청 timeout, 이후 2개 응답 (3 sent / 2 received)이었다. 초기 ARP 과정일 가능성은 있으나 확인된 원인은 아니다. 후속 4회 연속 성공 출력은 미수집이다.
- Guest가 192.168.1.192에 Ping했을 때 .193에서 응답했다. .192는 /26의 Network Address이므로 단말 연결 검증 대상으로 사용하지 않는다.
- `ing`은 `ping` 오타이고 `show running config`는 `show running-config`로 수정했다. 이 오류는 네트워크 장애 원인이 아니다.

## Troubleshooting — Detailed Problem / Action / Result

### Issue 1: SSH activation log shows 1.99 after configuring v2

- **관찰:** `ip ssh version 2` 입력 시 "%SSH-5-ENABLED: SSH 1.99 has been enabled" 로그가 나왔다.
- **조치:** 로그 문자열만으로 최종 버전을 판단하지 않고 `show ip ssh`로 확인했다.
- **실제 결과:**
```text
SSH Enabled - version 2.0
Authentication timeout: 120 secs; Authentication retries: 3
```
- **판정:** Core의 SSH v2 활성화 확인. 이 출력만으로 계정 로그인과 ACL 정책까지 검증된 것은 아니므로 별도 접속 테스트를 진행했다.

### Issue 2: Core SSH reaches Password prompt but authentication fails

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

### Issue 3: Core SSH Connection refused due to wrong allowed subnet

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

### Issue 4: IT-Switch local authentication failure

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

### Issue 5: Why HR-Switch needs another VLAN 10 address

- **질문:** Core에 VLAN 10 IP를 설정했는데 왜 HR-Switch에도 다시 설정하는가?
- **설명:** Core .0.193은 라우팅 Gateway, HR-Switch .0.220은 해당 L2 장비의 관리 목적지이다. 같은 VLAN의 서로 다른 호스트 주소이므로 중복 설정이 아니다.
- **결과:** .0.220 SSH 로그인 후 HR-Switch# 도달. ACL permit 2 matches / deny 15 matches 수집.

## DNS — IP 접속은 되지만 이름 조회 실패

- **증상:** 서버 Ping과 IP 기반 HTTP는 정상, 이름으로 접속하면 Hostname unresolved.
- **진단:** PC DNS 주소가 맞아도 서버 DNS 서비스가 실제로 켜졌는지 확인해야 한다.
- **원인:** DHCP-Server의 DNS Service가 Off.
- **조치:** Services → DNS → On.
- **검증:** 이름 기반 내부 HTTP 성공 보고. 이후 직원 Data Pool 전체에 DNS 배포 완료 보고.
- **교훈:** 네트워크 도달성, 서비스 활성 상태, 레코드 존재를 별도로 확인한다.

## CME 미지원 — 2911에서 telephony-service 거절

- 기존 Edge-Router에서 % Unrecognized command.
- Edge Routing/PAT를 유지하고 2811을 추가.
- 새 2811에서 telephony-service ? → <cr>를 확인하여 지원 여부 검증.
- 특정 Packet Tracer 모델/IOS의 제약이며 실제 2911 전체의 기능 제한으로 일반화하지 않는다.

## 전화기 — Configuring CM List에서 대기

- **증상:** 전화기 등록 실패, show ephone 비어 있음.
- **출력 확인:** Voice-Router IP가 Web Server와 같은 .1.3이고 CME ip source-address 누락.
- **조치:** Fa0/0를 .1.4/28로 수정, telephony-service에서 ip source-address 192.168.1.4 port 2000, create cnf-files.
- **후속:** 네 Voice DHCP Pool의 TFTP .1.4 확인 후 전화기 재시작.
- **검증:** HR .0.226 / 1001 REGISTERED 실제 출력, 이후 8대 및 부서 간 통화 성공 보고.
- **교훈:** IP 중복, 구성 서버 전달, CME 등록 주소를 각각 확인한다.

## Syslog — 심각도 입력 형식 오류

- logging trap informational 및 logging trap 6이 실제 CLI에서 거절됐다.
- logging ?에는 trap이 있으므로 명령 전체 미지원으로 단정할 수 없었다.
- logging trap debugging 적용 후 show logging에서 debugging 및 수신 서버 .1.2 확인.
- 수신 서버 화면에서 CONFIG_I 및 LOGGINGHOST_STARTSTOP 세 메시지 확인.
- 초기 도움말 확인 전에 숫자 6 대안을 제안한 안내는 이 환경에서 맞지 않았다. 작동하지 않은 명령은 최종 가이드에서 제거했다.

## 문서 정리 시의 상태 보완

- Guest 무선 검증은 최종적으로 완료됐다.
- 직원 Wi-Fi와 Syslog는 구현됐다.
- PAT 실제 변환표도 수집됐다.
- 최종 .pkt는 저장소에서 존재 확인했다.
- ENG Pool 58 최종 화면, 일부 백업 파일과 복구 테스트는 확인 대기이다.

---

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)
