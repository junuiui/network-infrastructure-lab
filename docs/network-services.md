# Routing, DHCP, PAT & DNS

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)

> 기준: 2026-10-27. 설정 절차와 실제 검증 상태를 구분한다.


## 역할과 통신 흐름

Core는 SVI Gateway와 DHCP Relay, Edge는 외부 경로와 PAT를 담당한다. DHCP Server .1.2는 주소 배포와 내부 DNS를 제공한다. 외부 서버는 HTTP/DNS를 제공하는 모의 목적지이다.

DHCP Relay는 각 SVI의 ip helper-address 192.168.1.2로 설정한다. 서버와 같은 VLAN 50에는 Relay가 필요하지 않다. 기존 Core에는 helper가 남아 있었지만 재구축 가이드는 생략한다. Pool 값은 [주소표](addressing.md)를 사용한다.

## Core–Edge Routed Link & Static Routes

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

## PAT & Simulated External Access

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
- **초기 결과:** 전체 성공 보고 후 아래 Actual PAT Evidence의 변환표/통계 출력을 추가 수집했다.
- 단순 외부 접속 성공만으로 PAT 변환을 입증하지 않는다. 이 구조는 왕복 Routing만으로도 통신 가능하므로 변환표 증거를 추가 수집해야 한다.
- 203.0.113.0/24는 문서용 주소를 이용한 모의 외부망이다. 실제 ISP/Internet 또는 Google 접속은 구현하지 않았다.

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

## Internal and Guest DNS

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

## 현재 검증 상태

위 이전 기록의 PAT 출력 미수집 상태는 **현재 해소**됐다. 직원 무선 .2.11과 IT 유선 .1.19의 실제 변환표를 수집했다. DNS 조회/HTTP 성공은 사용자 완료 보고 기준이다. 상세 상태는 [검증 결과](validation.md)를 참고한다.

[처음부터 설정하기](guide.md) · [오류 해결](troubleshooting.md)


---

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)
