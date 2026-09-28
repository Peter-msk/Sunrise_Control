# Sunrise Control

Sunrise Control은 **Sunrise Group 프로그램의 작동 여부와 업데이트 정책을 중앙에서 관리하기 위한 공통 제어 저장소**입니다.

이 저장소의 핵심 파일인 `control.txt`를 각 Sunrise 프로그램이 시작 시 또는 정해진 시점에 조회하여, 프로그램의 정상 실행 여부·점검 상태·사용 중지 여부·업데이트 허용 여부·최소 지원 버전 등을 판단합니다.

Sunrise Control은 개별 프로그램의 기능을 대신 실행하는 런처가 아니며, 각 제품의 기존 업데이트 시스템(`latest.json`, `notice.txt`, 패키지 정보, SHA256, onefile/onedir 교체 절차 등)을 대체하지 않습니다.  
**중앙 제어는 "이 프로그램을 지금 사용할 수 있는가 / 업데이트를 허용하거나 요구할 것인가"만 결정하고, 실제 업데이트는 각 제품의 기존 업데이트 계약이 담당합니다.**

---

## 1. 목적

Sunrise Group 제품이 여러 개로 늘어나면 프로그램마다 별도의 긴급 차단·점검·업데이트 제어 기능을 운영하기 어렵습니다.

Sunrise Control은 하나의 중앙 정책 파일로 다음 상황을 관리하기 위해 사용합니다.

- 특정 프로그램의 정상 실행 허용
- 특정 프로그램의 일시적인 점검 모드 전환
- 특정 프로그램의 긴급 사용 중지
- 제품별 업데이트 허용 또는 차단
- 특정 버전 이하의 업데이트 요구
- 프로그램별 사용자 안내 메시지 제공
- 전체 제품에 대한 공통 기본 정책 설정
- GitHub 또는 네트워크 장애 시 캐시·fallback 정책 제공

즉, Sunrise Control은 **Sunrise Group 제품군의 중앙 운영 스위치(Control Plane)** 역할을 합니다.

---

## 2. 저장소 구성

현재 기본 구조는 단순하게 유지합니다.

```text
Sunrise_Control/
├─ README.md
└─ control.txt
```

### `README.md`

Sunrise Control의 목적, 운영 방식, `control.txt` 계약과 프로그램 연동 원칙을 설명합니다.

### `control.txt`

실제 프로그램이 읽는 중앙 제어 정책 파일입니다.

운영 정책을 바꿀 때는 소스코드나 각 프로그램의 업데이트 파일을 수정하는 대신, 필요한 경우 이 파일의 해당 제품 영역만 변경합니다.

---

## 3. 기본 동작 구조

```text
                    Sunrise_Control
                         │
                    control.txt
                         │
          ┌──────────────┼──────────────┐
          │              │              │
          ▼              ▼              ▼
  Sunrise Finder   Sunrise Cleaner   Sunrise EIASS
          │              │              │
          └───── Control Client ─────────┘
                         │
               제품별 operation/update 판정
                         │
             ┌───────────┴───────────┐
             ▼                       ▼
        정상 프로그램 실행       점검/차단/업데이트 처리
```

각 프로그램은 자신의 고유 `control_id`를 사용하여 `control.txt`에서 자기 제품 영역만 확인합니다.

예:

```ini
[product.sunrise_finder]
display_name=Sunrise Finder
operation=allow
update=allow
minimum_version=
message=
```

---

## 4. 전역 설정

`[global]`은 개별 제품에 별도 설정이 없거나 중앙 제어 서버 연결에 실패했을 때 사용할 공통 정책을 정의합니다.

현재 기본 구조:

```ini
[global]
schema_version=1
control_revision=1
updated_at=2026-09-28T12:39:00+09:00

default_operation=allow
default_update=allow

fallback_operation=allow
fallback_update=allow

cache_ttl_seconds=21600
```

### 주요 항목

| 항목 | 설명 |
|---|---|
| `schema_version` | control.txt 형식 버전 |
| `control_revision` | 중앙 제어 정책 개정 번호 |
| `updated_at` | 마지막 정책 변경 시각 |
| `default_operation` | 제품별 항목이 없을 때 기본 실행 정책 |
| `default_update` | 제품별 항목이 없을 때 기본 업데이트 정책 |
| `fallback_operation` | 네트워크 및 캐시 실패 시 실행 정책 |
| `fallback_update` | 네트워크 및 캐시 실패 시 업데이트 정책 |
| `cache_ttl_seconds` | 마지막 정상 제어정보를 사용할 수 있는 캐시 유효시간 |

현재 캐시 유효시간 `21600`초는 **6시간**입니다.

---

## 5. 프로그램 작동 정책

제품별 `operation` 값은 다음 세 가지를 사용합니다.

| 값 | 의미 |
|---|---|
| `allow` | 정상 실행 허용 |
| `maintenance` | 점검 상태. 일반 업무 화면 진입 제한 |
| `block` | 프로그램 사용 중지 |

### 정상 실행

```ini
operation=allow
```

프로그램은 평상시와 동일하게 실행합니다.

### 점검 모드

```ini
operation=maintenance
message=현재 시스템 점검 중입니다. 잠시 후 다시 실행해 주세요.
```

프로그램은 일반 업무 화면으로 진입하지 않고 점검 안내를 표시합니다.

### 사용 중지

```ini
operation=block
message=현재 이 버전의 프로그램 사용이 중지되었습니다.
```

프로그램의 정상 사용을 차단합니다.

---

## 6. 업데이트 정책

제품별 `update` 값은 다음 세 가지를 사용합니다.

| 값 | 의미 |
|---|---|
| `allow` | 기존 업데이트 기능 사용 허용 |
| `block` | 업데이트 조회 또는 적용 비활성 |
| `required` | 최소 버전 조건을 적용하여 업데이트 요구 |

### 일반 업데이트 허용

```ini
update=allow
```

각 제품이 원래 가지고 있는 업데이트 기능을 정상 사용합니다.

### 업데이트 중지

```ini
update=block
```

중앙 정책에 따라 업데이트 기능을 일시적으로 비활성화합니다.

### 최소 버전 요구

```ini
update=required
minimum_version=2.3.1
message=안정성 개선을 위해 최신 버전 업데이트가 필요합니다.
```

현재 실행 중인 제품 버전이 `minimum_version`보다 낮으면 해당 제품은 자신의 **기존 업데이트 계약**을 사용하여 업데이트를 진행해야 합니다.

Sunrise Control 자체는 EXE·ZIP 다운로드 주소나 SHA256을 제공하지 않습니다.

---

## 7. 기존 업데이트 시스템과의 관계

Sunrise Control은 각 제품의 기존 업데이트 파일을 대체하지 않습니다.

예를 들어:

```text
control.txt
    │
    └─ 업데이트를 허용하거나 요구할지 결정
             │
             ▼
각 제품의 기존 updater
    │
    ├─ latest.json 조회
    ├─ 최신 버전 확인
    ├─ 다운로드 URL 확인
    ├─ SHA256 검증
    ├─ onefile / onedir / installer 판단
    └─ 기존 제품 계약대로 교체
```

따라서 다음 정보는 원칙적으로 `control.txt`에 넣지 않습니다.

- 실제 EXE 다운로드 URL
- 실제 ZIP 다운로드 URL
- 배포파일 SHA256
- 제품별 `latest.json` 전체 내용
- onedir `package_info.json`
- 설치파일 교체 절차
- 제품별 기존 wire contract를 대체하는 필드

이 분리를 통해 중앙 제어 정책 변경이 각 제품의 업데이트 호환계약을 깨뜨리지 않도록 합니다.

---

## 8. 장애 및 오프라인 처리

Sunrise Control 서버 또는 GitHub에 일시적인 장애가 발생했다고 해서 모든 Sunrise 프로그램이 동시에 실행되지 않는 구조를 만들지 않습니다.

권장 처리 순서는 다음과 같습니다.

```text
1. 최신 control.txt 조회
        │
        ├─ 성공 → 검증 후 적용 + 로컬 캐시 저장
        │
        └─ 실패
             │
             ▼
2. 마지막 정상 캐시 확인
        │
        ├─ cache_ttl_seconds 이내 → 캐시 사용
        │
        └─ 유효 캐시 없음
             │
             ▼
3. fallback_operation / fallback_update 적용
```

현재 기본값:

```ini
fallback_operation=allow
fallback_update=allow
```

즉 일반적인 네트워크 장애만으로 프로그램 전체가 차단되지 않는 **fail-open 기본 정책**을 사용합니다.

특정 제품이 향후 별도의 온라인 필수 검증 정책을 가져야 한다면 그 제품 계약에서 별도로 정의하고, 그룹 전체 fallback 정책을 임의로 변경하지 않습니다.

---

## 9. Control Client 기본 요구사항

각 Sunrise 프로그램에 연결되는 Control Client는 최소한 다음 기능을 가져야 합니다.

1. HTTPS를 통해 `control.txt` 조회
2. UTF-8 파싱
3. `schema_version` 지원 여부 확인
4. 자신의 `control_id` 영역 조회
5. `operation` 값 검증
6. `update` 값 검증
7. `minimum_version` Semantic Version 비교
8. 알 수 없는 값은 임의 실행하지 않고 안전한 기본 정책으로 정규화
9. 마지막 정상 응답 로컬 캐시
10. 캐시 유효시간 확인
11. 네트워크 timeout 적용
12. UI thread에서 네트워크 호출을 직접 장시간 수행하지 않음
13. 사용자에게 내부 오류나 개발용 식별자를 그대로 노출하지 않음
14. 중앙 제어 오류와 제품 자체 오류를 분리하여 기록
15. API 키·사용자 문서·개인정보를 Sunrise Control로 전송하지 않음

---

## 10. 프로그램 시작 시 권장 처리 순서

```text
프로그램 시작
    │
    ▼
로컬 기본정보 로드
    │
    ▼
Sunrise Control 조회
    │
    ▼
control.txt 형식 검증
    │
    ▼
자기 control_id 확인
    │
    ├─ 없음 → global default 적용
    │
    ▼
operation 판정
    │
    ├─ allow ───────────────┐
    │                       │
    ├─ maintenance → 안내   │
    │                       │
    └─ block → 차단         │
                            ▼
                      update 정책 확인
                            │
             ┌──────────────┼──────────────┐
             ▼              ▼              ▼
           allow          block         required
             │              │              │
             ▼              ▼              ▼
       기존 updater 허용   비활성      최소버전 비교
                                           │
                                           ▼
                                    기존 updater 사용
```

---

## 11. 운영 예시

### Sunrise Finder 점검

```ini
[product.sunrise_finder]
display_name=Sunrise Finder
operation=maintenance
update=allow
minimum_version=
message=현재 시스템 점검 중입니다. 잠시 후 다시 실행해 주세요.
```

### Sunrise Cleaner 정상 운영

```ini
[product.sunrise_cleaner]
display_name=Sunrise Cleaner
operation=allow
update=allow
minimum_version=
message=
```

### 특정 버전 이하 업데이트 요구

```ini
[product.sunrise_pdf_editor]
display_name=Sunrise PDF Editor
operation=allow
update=required
minimum_version=4.4.2
message=안정성 개선을 위해 프로그램 업데이트가 필요합니다.
```

### 긴급 사용 중지

```ini
[product.sunrise_eiass]
display_name=Sunrise EIASS
operation=block
update=allow
minimum_version=
message=현재 버전의 사용이 일시 중지되었습니다. 업데이트 또는 공지사항을 확인해 주세요.
```

---

## 12. 현재 등록된 제품

현재 `control.txt`에는 다음 제품 영역이 등록되어 있습니다.

| Control ID | 제품명 |
|---|---|
| `sunrise_control` | Sunrise Control |
| `sunrise_cleaner` | Sunrise Cleaner |
| `sunrise_pdf_pro` | Sunrise PDF PRO |
| `sunrise_pdf_editor` | Sunrise PDF Editor |
| `sunrise_finder` | Sunrise Finder |
| `sunrise_capture` | Sunrise Capture |
| `sunrise_eiass` | Sunrise EIASS |
| `sunrise_eia_manager` | Sunrise EIA Manager |
| `sunrise_aermod` | Sunrise Aermod |
| `sunrise_shorts_ai` | Sunrise Shorts AI |
| `agent_peter` | Agent Peter |
| `geoagent_ai` | GeoAgent AI |

제품이 추가될 경우 기존 제품 영역을 수정하지 않고 새 `[product.<control_id>]` 영역을 추가합니다.

---

## 13. 운영 변경 원칙

`control.txt` 수정 시 다음 원칙을 따릅니다.

### 제품별 변경은 해당 제품 영역에서만 수행

예를 들어 Sunrise Finder의 점검 상태를 변경하기 위해 `[global]` 기본값이나 다른 제품의 설정을 함께 변경하지 않습니다.

### 전체 정책 변경만 global에서 수행

모든 제품에 실제로 동일하게 적용해야 하는 경우에만 `[global]`을 수정합니다.

### revision 증가

실제 운영 정책을 변경했다면 `control_revision`을 증가시키고 `updated_at`을 변경합니다.

### 알 수 없는 상태값 사용 금지

```text
operation: allow / maintenance / block
update:    allow / block / required
```

외 임의 문자열을 운영 값으로 추가하려면 먼저 schema를 개정하고 Control Client 호환성을 검토합니다.

---

## 14. 보안 원칙

Sunrise Control은 중앙 운영 정책 파일이지 원격 명령 실행 시스템이 아닙니다.

다음 기능은 `control.txt`에 포함하지 않습니다.

- 원격 Python/PowerShell/CMD 코드
- 임의 명령 실행
- 원격 스크립트 다운로드 후 실행
- 인증정보 또는 API Key 전달
- 사용자 파일 업로드 명령
- 사용자 파일 삭제 명령
- 관리자 권한 자동상승 명령
- 서비스·시작프로그램·예약작업 등록 명령

즉 중앙에서는 **정해진 상태값과 정책값만 전달**하고, 프로그램은 미리 구현·검증된 동작만 수행합니다.

---

## 15. 설계 원칙

Sunrise Control은 다음 원칙을 유지합니다.

### 중앙 정책, 로컬 실행

중앙 서버는 정책만 제공합니다. 실제 프로그램 동작과 업데이트 방식은 각 제품이 소유합니다.

### 기존 제품 계약 보호

각 제품이 이미 사용하고 있는 업데이트 JSON·공지·파일명·SHA256·패키징 계약을 Sunrise Control 때문에 변경하지 않습니다.

### 단일 장애점 방지

네트워크 또는 GitHub 장애가 전체 프로그램 장애로 확대되지 않도록 캐시와 fallback을 사용합니다.

### 최소 권한

Control Client는 제어정보를 읽기 위해 필요한 최소 네트워크 기능만 사용합니다.

### 변경 영향 격리

한 제품의 정책 변경이 다른 제품 정책을 자동 변경하지 않도록 제품별 영역을 분리합니다.

### 단순한 형식

사람이 GitHub에서 직접 읽고 긴급 수정할 수 있도록 `control.txt`는 단순한 UTF-8 INI 스타일 텍스트 형식을 유지합니다.

---

## 16. 향후 확장 방향

필요성이 확인될 경우 다음 기능을 단계적으로 검토할 수 있습니다.

- 제품 그룹별 정책
- 점검 시작·종료 시각
- 공지 식별자
- 특정 버전 범위 차단
- Control Client 공통 라이브러리
- Sunrise Control 관리 UI
- 정책 변경 전 문법 검증
- 변경 이력 표시
- 긴급 차단 시 운영자 확인 절차

다만 현재 사용하지 않는 기능을 미리 대량 추가하지 않고 실제 운영 요구가 발생할 때 schema를 확장합니다.

---

## 17. 중요 주의사항

`control.txt`를 수정하면 이를 조회하는 여러 Sunrise 프로그램의 동작에 영향을 줄 수 있습니다.

특히 다음 항목을 변경할 때는 주의해야 합니다.

- `default_operation`
- `fallback_operation`
- `operation=block`
- `operation=maintenance`
- `update=required`
- `minimum_version`

운영 변경 전에는 대상 제품과 의도를 확인하고, 가능하면 **한 제품 영역만 최소 변경**합니다.

---

## 18. 요약

**Sunrise Control은 Sunrise Group 프로그램을 중앙에서 운영하기 위한 제어 페이지입니다.**

핵심 역할은 다음과 같습니다.

```text
프로그램 작동 허용
프로그램 점검 전환
프로그램 사용 차단
업데이트 허용
업데이트 차단
최소 버전 강제
사용자 안내 메시지
장애 시 캐시/fallback
```

실제 프로그램 기능이나 업데이트 파일을 중앙에서 대신 실행하는 시스템이 아니라, **각 Sunrise 프로그램이 따라야 할 운영 상태를 전달하는 안전하고 단순한 중앙 Control Plane**을 목표로 합니다.

---

**Repository:** Peter-msk/Sunrise_Control  
**Primary policy file:** `control.txt`  
**Control schema:** v1

---

## 19. Security Architecture v2 설계

Sunrise Control의 중앙 정책을 다른 개발자가 클라이언트 패치·임의 재빌드로 단순 우회하기 어렵게 하기 위한 **Security Architecture v2** 설계를 별도 문서로 관리합니다.

설계 핵심:

- `control.txt`는 기존 운영정책 원본으로 유지
- `control.txt.sig`를 통한 정책 전자서명
- Offline Root 기반 keyset/키 교체
- Authorization Server
- short-lived Execution Lease
- TPM-backed Device Key
- Official Build Registry 및 Revocation
- STANDARD / ENFORCED / STRICT 보안 프로필
- Python/Tauri 제품용 Native Guard
- 관리형 Windows 환경의 App Control for Business 연동
- 기존 updater / latest.json / SHA256 / package contract 비침범

현재 문서는 **설계 상태이며 아직 런타임에 구현되지 않았습니다.**

문서: [Sunrise Control Security Architecture v2.0 DESIGN](docs/Sunrise_Control_Security_Architecture_v2.0_DESIGN.md)

