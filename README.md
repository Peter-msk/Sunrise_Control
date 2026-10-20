# Sunrise Control

Sunrise Control은 Sunrise Group 프로그램별 운영 상태와 업데이트 기준을 정리하는 공통 저장소입니다. 제품별 설정과 안내 정보를 한곳에서 관리합니다.

## 저장소 구성

```text
Sunrise_Control/
├─ README.md
├─ control.txt
└─ products/
   └─ sunrise_file_manager/
      ├─ latest.json
      └─ notice.txt
```

- `control.txt`: 전체 기본값과 제품별 운영·업데이트 상태
- `products/<control_id>/latest.json`: 제품 버전 및 업데이트 안내 정보
- `products/<control_id>/notice.txt`: 제품별 공지사항

## 제품별 설정 항목

제품 설정은 `control.txt`의 `[product.<control_id>]` 영역에 작성합니다. 제품별로 사용하는 항목은 다음과 같습니다.

| 항목 | 설명 |
|---|---|
| `display_name` | 제품 표시 이름 |
| `operation` | 제품의 운영 상태 |
| `update` | 업데이트 처리 상태 |
| `minimum_version` | 업데이트가 필요한 최소 버전 |
| `message` | 제품별 안내 문구 |

### 운영 상태

| 값 | 의미 |
|---|---|
| `allow` | 정상 이용 |
| `maintenance` | 점검 안내 표시 |
| `block` | 이용 중지 안내 표시 |

### 업데이트 상태

| 값 | 의미 |
|---|---|
| `allow` | 제품의 업데이트 기능 이용 |
| `block` | 업데이트 일시 중지 |
| `required` | 설정된 최소 버전 기준으로 업데이트 안내 |

## 현재 등록된 제품

아래 목록은 현재 `control.txt`에 등록된 제품 영역을 기준으로 합니다.

| Control ID | 제품명 |
|---|---|
| `sunrise_control` | Sunrise Control |
| `sunrise_cleaner` | Sunrise Cleaner |
| `sunrise_pdf_pro` | Sunrise PDF PRO |
| `sunrise_pdf_editor` | Sunrise PDF Editor |
| `sunrise_file_manager` | Sunrise File Manager |
| `sunrise_finder` | Sunrise Finder |
| `sunrise_capture` | Sunrise Capture |
| `sunrise_eiass` | Sunrise EIASS |
| `sunrise_eia_manager` | Sunrise EIA Manager |
| `sunrise_aermod` | Sunrise Aermod |
| `sunrise_shorts_ai` | Sunrise Shorts AI |
| `agent_peter` | Agent Peter |
| `geoagent_ai` | GeoAgent AI |

## 정책 변경

- 기존 제품의 설정을 바꿀 때는 해당 제품 영역을 수정합니다.
- 전체 기본값을 바꿀 때는 `[global]` 영역을 수정합니다.
- 제품을 추가할 때는 `control.txt`에 새 제품 영역을 등록하고 이 README의 목록도 함께 갱신합니다.
- 공지사항이나 버전 안내가 필요한 제품은 `products/<control_id>/` 아래에 제품별 파일을 둡니다.

