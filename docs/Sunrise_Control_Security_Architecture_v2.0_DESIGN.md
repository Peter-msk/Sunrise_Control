# Sunrise Control Security Architecture v2.0

> 상태: **DESIGN / NOT IMPLEMENTED**  
> 기준일: 2026-09-28  
> 기존 운영 계약: `control.txt` schema v1 유지  
> 목표: Sunrise 제품의 중앙 Control 우회, 정책 위변조, 임의 재빌드/복제 비용을 높이고 서버 측에서 실질적인 실행권한을 통제한다.

## 1. 핵심 원칙

Sunrise Control v2의 목표는 "클라이언트 소스를 절대 볼 수 없게 만드는 것"이 아니다.

사용자가 PC와 실행파일을 완전히 통제하는 비관리형 Windows 환경에서는 숙련된 개발자가 로컬 검사코드를 패치할 가능성을 0으로 만들 수 없다. 따라서 다음 원칙을 사용한다.

> **로컬 검사는 방어층이고, 실질적인 강제력은 서버가 보유한 실행권한·키·핵심 서비스에 둔다.**

기존 `control.txt`와 각 제품 updater는 유지한다. v2는 그 위에 보안 계층을 추가한다.

## 2. 방어 대상

- `control.txt`를 임의 수정해 `allow`로 변경
- GitHub/네트워크/캐시에서 가짜 정책 공급
- ControlClient의 `block`/상태 검사를 제거
- 디컴파일 후 비공식 EXE 재빌드
- 공식 폴더/lease를 다른 PC로 복사
- 과거 allow 정책 또는 만료 전 lease 재생
- 시스템 시간 rollback
- 특정 빌드/장치/제품의 긴급 철회
- 정책/lease signing key 유출 시 피해확산 제한

## 3. 전체 구조

```text
                    [관리자]
                       │
              control.txt 수정
                       │
              정책 서명 도구/HSM
                       │
           ┌───────────┴───────────┐
           ▼                       ▼
     control.txt              control.txt.sig
           │                       │
           └──────── GitHub ───────┘
                       │
              HTTPS + 서명검증
                       │
          ┌────────────┴────────────┐
          ▼                         ▼
 [Sunrise Product]        [Authorization Server]
          │                         │
   Policy Verifier             정책 서명검증
   Native Guard                Product Registry
   Device Key                  Build Registry
   Lease Verifier              Device Registry
   Existing Updater            Revocation
          │                    Lease Issuer
          └────────────┬────────────┘
                       │
              signed Execution Lease
                       │
             핵심 기능/키/서비스 활성화
```

## 4. 역할 분리

| 구성요소 | 책임 |
|---|---|
| `control.txt` | 사람이 관리하는 운영 상태 원본 |
| `control.txt.sig` | 정책 진위, SHA, revision, 유효기간 증명 |
| Authorization Server | 제품/장치/빌드/철회 확인 후 실행권한 발급 |
| Execution Lease | 짧은 수명의 서명된 실행권한 |
| Device Key | 장치 복제/lease 복사 방어 |
| Native Guard | Python/JS에서 단순 조건 삭제만으로 우회하기 어렵게 하는 로컬 방어층 |
| Existing Updater | 실제 다운로드/검증/교체. 기존 wire contract 유지 |

## 5. 기존 v1 비침범

v2 적용으로 다음을 변경하지 않는다.

- 각 제품의 기존 `latest.json`, `notice.txt`
- 기존 다운로드 URL 계약
- SHA256 검증
- onefile / onedir / installer 방식
- 사용자 설정/원본 보존 계약
- 기존 updater의 공개 wire contract

`control.txt`는 계속 다음 운영값을 소유한다.

```text
operation = allow | maintenance | block
update    = allow | block | required
minimum_version
message
```

## 6. 신뢰의 뿌리

### Offline Root Key

최상위 keyset 서명용이다.

- 일반 개발 PC, CI, GitHub에 두지 않는다.
- 가능하면 HSM/하드웨어키/오프라인 보관.
- 일상 policy/lease 서명에는 직접 사용하지 않는다.

### Policy Signing Key

`control.txt.sig` 생성용이다.

- Root Key와 분리.
- 노출 시 이 key만 revoke/rotation 가능.

### Lease Signing Key

Authorization Server의 Execution Lease 서명용이다.

- 서버 KMS/HSM 또는 비추출 키 저장소 사용.
- GitHub, 소스, 배포 ZIP/PFX에 포함하지 않는다.

### Code Signing Certificate

공식 EXE/DLL/Setup Authenticode 서명용이다.

### Device Key

가능한 Windows PC에서는 TPM-backed non-exportable key를 사용하고 서버에는 public key fingerprint만 등록한다.

## 7. Signed control policy

`control.txt` 원문은 그대로 유지하고 별도 `control.txt.sig`를 둔다.

```text
Sunrise_Control/
├─ README.md
├─ control.txt
├─ control.txt.sig
└─ docs/
```

권장 방식은 자체 암호 포맷 대신 표준 JWS를 사용한다.

JWS payload 예:

```json
{
  "type": "sunrise-control-policy",
  "payload_sha256": "<SHA256(control.txt exact UTF-8 bytes)>",
  "control_revision": 17,
  "issued_at": "2026-09-28T13:00:00+09:00",
  "not_after": "2026-10-05T13:00:00+09:00"
}
```

protected header 예:

```json
{
  "alg": "PS256",
  "kid": "scp-2026-01",
  "typ": "SUNRISE-CONTROL-SIG"
}
```

검증 순서:

```text
control.txt + control.txt.sig 다운로드
→ trusted keyset에서 kid 조회
→ JWS signature 검증
→ payload SHA256과 실제 파일 SHA256 비교
→ revision/issued_at/not_after 확인
→ last accepted revision보다 낮은 rollback 거부
→ 정책 수락 및 정상 cache 저장
```

정상 rollback이 필요한 경우에는 별도 signed recovery authorization을 사용한다.

## 8. Keyset rotation

클라이언트에는 Offline Root Public Key만 고정한다.

Root Key가 서명한 `keyset.jws`가 active policy/lease public key를 제공한다.

효과:

- policy/lease signing key 유출 시 제품 전체 재배포 없이 교체 가능
- Root Key는 일상 온라인 환경에서 분리
- key status를 active/revoked/retired로 관리 가능

## 9. Authorization Server

서버의 최소 책임:

- 서명된 `control.txt` 검증/캐시
- Product Registry
- 제품별 Security Profile
- minimum version
- Device Registry
- Official Build Registry
- Revocation
- challenge 발급
- device proof 검증
- Execution Lease 발급

서버는 사용자 PDF/HWP/DXF/GIS 문서, API Key, 업무파일 경로 등을 수집하지 않는다.

장치 ID는 motherboard serial 조합보다 Device Public Key fingerprint를 우선한다.

## 10. Execution Lease

ENFORCED/STRICT 제품은 유효한 Execution Lease가 있어야 핵심 기능을 활성화한다.

권장 형식: signed JWS/JWT.

최소 claims 예:

```json
{
  "iss": "sunrise-control",
  "aud": "sunrise-product",
  "jti": "<unique lease id>",
  "iat": 0,
  "nbf": 0,
  "exp": 0,
  "product_id": "sunrise_finder",
  "security_profile": "ENFORCED",
  "policy_revision": 17,
  "operation": "allow",
  "update": "allow",
  "minimum_version": "2.3.0",
  "device_key_fp": "<public key fingerprint>",
  "build_id": "<registered build id>"
}
```

Native Guard/제품 클라이언트는 signature, issuer, audience, expiry, product/device/profile/revision을 확인한다.

## 11. Device Binding

최초 등록:

```text
제품 최초 실행
→ TPM-backed non-exportable key 생성
→ public key + 선택적 key attestation 전송
→ 서버 device_id 발급
```

lease 발급:

```text
Client → /challenge
Server → nonce + nonce_id + server_time
Client → Device Private Key로 nonce 서명
Client → /lease + device_signature
Server → 검증
Server → signed Execution Lease
```

이 방식은 lease 파일을 다른 PC로 복사하는 공격을 방어한다.

### 경계

TPM key는 장치/키 신뢰를 높이지만 일반 비관리형 Windows 환경에서 "현재 실행 중인 사용자 모드 EXE 전체가 공식 binary다"를 단독으로 증명하는 것으로 간주하지 않는다.

## 12. Official Build Registry

서버 데이터:

```text
product_id
version
revision
build_id
artifact_kind
sha256
Authenticode signer identity
released_at
status = allowed | revoked | superseded
```

목적:

- 정식 release 식별
- 긴급 build revoke
- update 강제
- 이상 build/version 탐지

단, client가 자기 SHA/build_id를 보고하는 것만으로 hostile client에 대한 cryptographic attestation이라고 간주하지 않는다.

## 13. Security Profiles

Profile의 최종 권한은 서버 Product Registry가 가진다. Client가 자기 profile을 낮출 수 없다.

### STANDARD

대상: 무료/내부 유틸리티, 오프라인 가용성이 특히 중요한 도구.

- Authenticode
- signed control policy
- signed cache/fallback
- optional Device Key
- lease 필수 아님

정책 위변조에는 강하지만 완전 로컬 rebuilt binary 차단은 보장하지 않는다.

### ENFORCED — 기본 권장

일반 상용/업무용 Sunrise 제품의 기본 목표.

- STANDARD 전체
- Authorization Server 필수
- Device Key
- short-lived Execution Lease
- 핵심 기능 진입 전 lease 요구
- build/version/revocation 확인
- Python/Tauri는 Native Guard 권장

초안 기본값:

```text
lease_ttl      = 12h
refresh_after  = 6h
offline_grace  = 48h
```

서버 장애 시 유효 lease까지 동작하며 grace 종료 후 핵심 기능은 제한하고 업데이트/복구 UI만 허용할 수 있다.

### STRICT

대상: 중요 라이선스, 민감 핵심기능, 관리형 사내 PC.

- ENFORCED 전체
- TPM 필수 가능
- 짧은 lease
- offline grace 최소/없음
- server-held key/service
- 관리형 PC에서 App Control for Business
- 필요 시 TPM/Azure Attestation

초안:

```text
lease_ttl     = 1h
offline_grace = 0~4h
```

## 14. 서버가 보유해야 하는 실질 강제력

완전히 로컬에 존재하는 기능은 결국 local patch 가능성이 남는다.

따라서 보호 중요도가 높은 제품은 다음 중 하나를 사용한다.

1. 핵심 서비스 자체가 서버 API를 필요로 함.
2. 암호화된 핵심 리소스/모델/정책 bundle의 content key를 valid lease 후에만 서버가 제공.
3. 서버가 entitlement/capability를 직접 확인하는 기능 경로 사용.

복호화된 데이터가 메모리에 존재하면 숙련된 공격자가 덤프할 수 있으므로 이것도 절대 보호는 아니다. 목표는 공격비용 상승과 대량 무단복제를 어렵게 하는 것이다.

## 15. Native Guard

Python/PyInstaller, JavaScript/Tauri 제품에 공통 native module을 둘 수 있다.

예:

```text
sunrise_guard.dll
```

책임:

- Root/keyset 검증
- control signature 검증
- lease 검증
- TPM/Device challenge signing 호출
- expiry/monotonic 보조검사
- 핵심기능 enable decision

원칙:

- private key 내장 금지
- 전체 업무로직을 Guard에 몰아넣지 않음
- Guard 실패가 사용자 데이터 손상으로 이어지지 않음
- anti-debug는 호환성/오탐 때문에 보수적으로 적용

Native Guard는 우회비용을 높이는 계층이지 독립 Root of Trust가 아니다.

## 16. Time/Replay 방어

Policy:
- `control_revision` 단조 증가
- `not_after`
- last accepted revision 저장

Lease:
- unique `jti`
- 짧은 `exp`
- server time 기반
- 매 refresh마다 새로운 nonce

로컬:
- server_time과 wall clock 차이 기록
- 같은 세션에서는 monotonic clock 병행
- 큰 시간 역행 감지 시 online refresh 요구

## 17. Offline 정책

운영 policy fallback과 보안 lease를 분리한다.

```text
GitHub policy 장애
→ signed cached policy

Authorization Server 장애
→ 아직 유효한 lease

lease 만료
→ offline_grace

grace 종료
→ profile에 따른 제한모드
```

따라서 v1의 `fallback_operation=allow`가 ENFORCED/STRICT에서 무기한 보안우회 수단이 되지 않는다.

## 18. Startup 권장 흐름

```text
Process Start
→ product identity/version
→ Authenticode/자가무결성 보조검사
→ Root-signed keyset 검증
→ signed control policy
→ operation 판정
   ├─ maintenance/block → 업데이트/복구 UI
   └─ allow
→ server security profile
   ├─ STANDARD → 정상진입
   ├─ ENFORCED → device proof + lease → 정상진입
   └─ STRICT → attestation/lease/server key/service → 정상진입
```

## 19. 기존 updater 연결

```text
policy/lease: update=required
→ current_version < minimum_version
→ 핵심 업무 기능 제한
→ 기존 product updater 실행
→ 기존 HTTPS/SHA256/package/rollback 계약
→ 새 공식 build 시작
→ 새 lease 획득
```

기존 제품에서 "업데이트 창을 닫으면 프로그램 종료" 계약이 있다면 그 제품 현행계약을 우선한다.

## 20. 관리형 Windows 강화

회사에서 관리하는 Windows 단말은 별도 OS 계층으로 App Control for Business를 사용할 수 있다.

- Sunrise signer/hash만 실행 허용
- 비공식 재빌드 실행 차단
- signed App Control policy + Secure Boot
- 적용 전 Audit Mode/대표 PC 검증

일반 개인 PC에 제품이 임의로 이 정책을 강제 설치하지 않는다. 조직 단말 관리용 옵션이다.

## 21. TPM/Attestation 경계

TPM Key Attestation:
- private key가 TPM에 생성/관리됨
- non-exportable 성질
- TPM-backed credential 신뢰

TPM/Measured Boot Attestation:
- boot integrity/platform state 검증

일반적인 TPM attestation만으로 특정 Sunrise 사용자 모드 EXE 전체 코드가 공식 binary라는 사실까지 자동 증명한다고 가정하지 않는다.

## 22. 서버 데이터모델 초안

### Product

```text
product_id
security_profile
status
minimum_version
offline_grace_seconds
device_binding_policy
build_policy
```

### Build

```text
build_id
product_id
version
revision
artifact_kind
sha256
signer
status
released_at
```

### Device

```text
device_id
public_key_fingerprint
key_type
attestation_state
status
registered_at
last_seen_at
```

### Revocation

```text
type = product | build | device | key | lease
id
reason_code
revoked_at
```

## 23. API 초안

### `POST /v1/device/register`

입력:
- product_id
- device_public_key
- optional key attestation

출력:
- device_id
- registration status

### `POST /v1/challenge`

출력:
- nonce_id
- nonce
- server_time
- expires_at

### `POST /v1/lease`

입력:
- product_id
- version
- build_id
- device_id
- nonce_id
- device_signature
- last_policy_revision

서버 확인:
- product/security profile
- minimum version
- build/device status
- revocation
- current signed control policy

출력:
- signed Execution Lease
- optional maintenance/update message

### `GET /v1/keyset`

출력:
- Offline Root signed keyset

## 24. 주요 reason codes

```text
ALLOW
MAINTENANCE
PRODUCT_BLOCKED
UPDATE_REQUIRED
LEASE_EXPIRED
DEVICE_NOT_REGISTERED
DEVICE_REVOKED
BUILD_REVOKED
POLICY_SIGNATURE_INVALID
POLICY_ROLLBACK_REJECTED
KEYSET_INVALID
ATTESTATION_REQUIRED
AUTH_SERVER_UNAVAILABLE
```

사용자 UI에는 내부정보를 과다 노출하지 않고 이해 가능한 안내문만 표시한다.

## 25. 위협별 방어

| 위협 | 주요 방어 | 잔여 한계 |
|---|---|---|
| control 수정 | JWS signature + SHA + revision | 검증코드 자체 패치 가능 |
| GitHub 변조 | policy signature | signing key 유출 시 위험 |
| Control 분기 삭제 | Native Guard + lease | 완전 로컬 기능 patch 가능 |
| 비공식 rebuild | server lease/server-held capability | unmanaged binary attestation 한계 |
| lease 복사 | TPM Device Key binding | 같은 장치 내 patch 공격 잔존 |
| 정책 replay | revision + not_after | 로컬검사 제거 가능 |
| 시간 rollback | server time + short lease | local-only 강제력 한계 |
| build revoke | Build Registry + server revocation | self-report build identity 한계 |
| 사내 PC 비공식 실행 | App Control + Secure Boot | 관리형 단말 필요 |

## 26. 구현 단계

### P1 — Signed Policy Foundation

- policy signing key 운영
- `control.txt.sig`
- Root-signed keyset
- verifier
- anti-rollback cache

완료:
- control 1 byte 수정 시 거부
- old revision 거부
- 정상 signed cache 사용

### P2 — Authorization Server + Lease

- Product Registry
- challenge/lease API
- lease signing key
- STANDARD/ENFORCED
- revocation

완료:
- ENFORCED 제품은 valid lease 없이 핵심 기능 진입 불가

### P3 — Device Binding

- TPM-backed device key
- register/challenge proof
- DPAPI fallback 정책
- device revoke

완료:
- lease 다른 PC 복사 실패

### P4 — Native Guard

- `sunrise_guard.dll`
- Python/Tauri binding
- policy/lease verify API

완료:
- Python/JS의 단일 조건문 삭제만으로 전체 보안체계가 무력화되지 않음

### P5 — STRICT / Managed Device

선택:
- App Control for Business
- Secure Boot 운영절차
- TPM/Azure Attestation PoC
- server-held content key/service

## 27. 최소 검증 게이트

Signed Policy:
- 정상 signature PASS
- 1 byte 수정 FAIL
- 잘못된 kid FAIL
- old revision FAIL
- expired signature FAIL

Lease:
- 정상 PASS
- signature 변조 FAIL
- expired FAIL
- product/device mismatch FAIL
- revoked device/build FAIL

Device:
- 다른 PC 복사 FAIL
- nonce replay FAIL
- expired nonce FAIL

Regression:
- 기존 latest/notice/SHA/package 계약 동일
- required update 정상
- 최종 packaged startup 정상

## 28. 기본 적용 제안

일반 Sunrise 데스크톱 제품의 목표 기본값:

```text
Security Profile : ENFORCED
Policy           : signed
Device Binding   : TPM preferred
Lease            : required
Lease TTL        : 12h
Offline Grace    : 48h
Build Registry   : enabled
Native Guard     : Python/Tauri 제품 우선
Updater          : existing contract preserved
```

실제 lease/offline 값은 제품별 오프라인 사용 요구를 확인한 뒤 확정한다.

## 29. 설계 결정 요약

1. `control.txt v1`은 폐기하지 않는다.
2. `control.txt`에 private key/원격명령을 넣지 않는다.
3. `control.txt.sig`로 정책을 서명한다.
4. Offline Root 기반 keyset으로 signing key를 교체할 수 있게 한다.
5. ENFORCED/STRICT는 short-lived Execution Lease가 필요하다.
6. Device Private Key는 가능한 경우 TPM에 non-exportable로 생성한다.
7. Build Registry를 unmanaged endpoint의 절대 binary attestation으로 과장하지 않는다.
8. 비공식 rebuild를 실질적으로 제한해야 하는 핵심기능은 server capability/key/service에 의존한다.
9. 관리형 PC에서는 App Control for Business를 별도 강제층으로 사용한다.
10. 기존 제품 updater/wire/package 계약은 유지한다.
