# Syslog & Configuration Backup

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)

> 기준: 2026-10-27. 설정 절차와 실제 검증 상태를 구분한다.

## Monitoring Scope — Central Syslog Collection

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

## Manual Configuration Backup & Save

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

## 최종 파일 현황 보완

문서 분리 시 저장소에서 [fake-company-final.pkt](../fake-company-final.pkt)의 존재를 확인했다. 이전 기록의 '.pkt 파일 미수집'은 저장소 존재 확인으로 갱신한다. 다만 파일 내부 설정, TFTP 파일 목록, 재실행/복원 결과를 직접 검증한 것은 아니다.

백업 완료 보고와 실제 전송 출력은 [검증 결과](validation.md)에서 구분한다. 실제 VM의 가용성/성능 대시보드 및 장애 알림은 다음 Lab으로 남긴다.


---

[← 메인 README](../readme.md) · [재구축 가이드](guide.md) · [검증 결과](validation.md)
