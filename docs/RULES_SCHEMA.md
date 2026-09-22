# RULES_SCHEMA — 규칙 JSON 스키마

`assets/rules/` 아래 규칙 JSON 3종의 구조, 각 파라미터의 의미와 허용 범위, 로더 검사 항목을 정한다.
이 문서는 JSON 실물과 함께 쓴다. JSON을 바꾸면 이 문서를 같은 커밋에서 맞춘다.

| 파일 | 상태 |
|---|---|
| `palette_v1.json` | **확정** (2026-09-22, 45색) — §2 |
| `harmony_rules_v1.json` | 미작성 — §3 |
| `face_constraints_v1.json` | 미작성 — §4 |

## 0. 공통 규약

- 인코딩 UTF-8, 줄바꿈 LF, 들여쓰기 2칸. 키는 camelCase.
- 모든 파일은 최상위에 `schema`(파일 종류 문자열)와 `version`(정수)을 가진다. 로더는 기대하는 `schema`·`version`과 다르면 `Failed(internal_error)`.
- 수치는 JSON number. 각도는 도(°) `[0, 360)`, 명도 L* `[0, 100]`, 채도 C* `≥ 0`, 비율 `[0, 1]`.
- 로더(`RulesLoader`)는 파일이 아니라 **문자열**을 받는다 (CONTRACT §3.1). 검증 실패는 첫 오류에서 멈추지 않고 모두 모아 `Failed(internal_error)`로 반환하며, 각 진단에 항목 경로(예: `palette.colors[3].lab`)와 진단 식별자를 붙인다 (CONTRACT §5.2).
- **튜닝 대상 값은 전부 JSON에 있다.** 코드에는 없다. 이 문서에서 "현재 값"이라 적힌 것은 JSON의 값을 옮겨 적은 것이며, 기준은 JSON이다.
- 파일 안의 배열 순서는 사람이 읽기 위한 것이다. 엔진은 순서에 의존하지 않고 `id` 등으로 정렬한다 (CONTRACT §2.4).

## 1. 명명 규칙

식별자는 세 종류다. 각각 한 줄 규칙을 가진다.

| 종류 | 규칙 | 상태 |
|---|---|---|
| **팔레트 색 id** | `^[a-z][a-z0-9]*(_[a-z0-9]+)*$`. 소문자 snake_case, 의류 유통에서 쓰는 영문 색 이름, 자연스러운 영어 어순(`light_grey`, `dusty_rose`, `sky_blue`), 고유 이름이 있으면 `_light/_dark` 접미사 대신 고유 이름(`charcoal`, `navy`), 숫자·버전 없음, 파일 안에서 유일 | **확정** |
| **규칙 ID** (harmony·face_constraints의 각 규칙) | — | harmony_rules 작성 시 확정 |
| **진단 식별자** (`Diagnostic.ruleId` 중 규칙이 아닌 것) | 잠정: `<영역>.<snake_case>` (예: `palette.hex_lab_mismatch`, `input.lab_range`). 영역은 `palette`, `harmony`, `constraints`, `input`, `engine` 중 하나 | 잠정. harmony_rules 작성 시 규칙 ID 규칙과 함께 확정 |
| **에러 코드** (`Failed.error.code`) | 잠정: 소문자 snake_case (`internal_error`, `no_candidate` — CONTRACT §2.5) | 잠정. 같은 시점에 확정 |

팔레트 검사(§2.4)의 진단 식별자는 잠정 규칙을 따른다. 규칙이 바뀌면 함께 고친다.

## 2. `palette_v1.json`

고정 팔레트. 추천 결과는 자유 RGB가 아니라 여기 있는 색 중에서만 나온다 (설계 원칙 3).

### 2.1 최상위 필드

| 키 | 타입 | 의미 | 현재 값 |
|---|---|---|---|
| `schema` | string | 파일 종류 | `"palette"` |
| `version` | int | 스키마 버전 | `1` |
| `labSource` | string | Lab 값의 출처. `"computed_from_hex"`만 허용. 별도 측정값이 아니라 hex에서 COLOR_SPEC §1.2로 계산한 값이라는 선언 | `"computed_from_hex"` |
| `colorSpec` | string | 계산 규약 참조 (사람용 메모) | COLOR_SPEC §1.2 |
| `tagVocabulary` | string[] | 허용 태그 목록. `colors[].tags`의 값은 이 안에 있어야 한다 | `["neutral"]` |
| `validation` | object | 로더 검사 파라미터 (§2.3) | |
| `colors` | object[] | 색 목록 (§2.2) | 45개 |

### 2.2 `colors[]` 항목

| 키 | 타입 | 의미 | 제약 |
|---|---|---|---|
| `id` | string | 식별자. 진단·스냅샷·뷰어가 이 값으로 색을 가리킨다 | §1 팔레트 id 규칙, 유일 |
| `nameKo` | string | 사용자에게 보이는 한국어 색 이름. 기준: **일반 사용자가 옷을 사러 갈 때 검색할 이름** (국내 의류 유통 통용 표기) | 비어 있지 않음 |
| `hex` | string | 표시용 sRGB | `^#[0-9A-F]{6}$` (대문자) |
| `lab` | `{L, a, b}` | CIELAB(D65). hex에서 계산해 **소수 2자리**로 저장 | `L ∈ [0,100]`, `a,b ∈ [-128,127]`, hex와 일치(§2.4-4) |
| `tags` | string[] | 규칙이 소비하는 태그. 빈 배열 허용 | 값은 `tagVocabulary` 안 |

**태그 `neutral`** — 중성형 전략의 메인/서브 후보 풀. 매치·대비형의 서브 후보로도 쓸 수 있다.
부여 기준: 패션에서 "베이직/무난한 색"으로 통용되어 어떤 립 색과도 색상 충돌을 일으키지 않는 색. 현재 무채색 전부, 베이지·브라운 전부, 네이비, 슬레이트(16색).
채도만으로는 정의할 수 없다: camel(C* 31), navy(C* 18)은 뉴트럴이지만 sage(C* 15), blush(C* 14)는 아니다. 그래서 태그다.
태그를 하나만 둔 이유: 규칙이 실제로 소비하는 태그만 넣는다. 웜/쿨은 h°로, 다크/라이트는 L*로 계산되고, 색상 계열 표시는 Kotlin이 결과 JSON에 h° 구간 라벨을 실어 보내면 된다.

### 2.3 `validation` 파라미터

| 키 | 타입 | 의미 | 허용 범위 | 현재 값 | 근거 |
|---|---|---|---|---|---|
| `hexLabToleranceDe76` | number | hex→Lab 계산값과 선언 `lab`의 ΔE76 허용 오차 | `(0, 1]` | `0.05` | 소수 2자리 반올림의 최대 ΔE76 = 0.0078. hex 한 채널 ±1의 최소 ΔE76 = 0.140. 0.05는 반올림의 6배 이상, 가장 작은 수기 실수의 1/3 미만 |
| `minNeutralCount` | int | `neutral` 태그 색의 최소 개수. 중성형 후보 풀 소멸 방지 | `≥ 2` | `12` | 현재 16개의 75%. 실수로 지우거나 태그를 떼는 것을 잡는 안전망. 명도대별 보장은 런타임 `no_candidate`의 역할이므로 여기서 하지 않는다 |
| `neutralMaxChroma` | number | `neutral` 태그 색의 C* 상한. 오태그 방지 | `> 0` | `35.0` | 현재 최대 camel 31.3. `red`(C* 75)를 neutral로 잘못 태그하는 식의 실수는 런타임에서 오류로 잡히지 않고 이상한 결과만 나오므로 로더에서 막는다 |

### 2.4 로더 검사 항목

모두 `Failed(internal_error)`. 진단에는 항목 경로와 아래 식별자를 붙인다.

| # | 검사 | 진단 식별자 |
|---|---|---|
| 1 | `schema == "palette"`, `version == 1`, 필수 필드·타입 | `palette.schema` |
| 2 | `labSource == "computed_from_hex"` | `palette.lab_source` |
| 3 | `id`가 §1 규칙에 맞는가 | `palette.id_format` |
| 4 | `id` 유일 | `palette.id_duplicate` |
| 5 | `hex` 형식 `^#[0-9A-F]{6}$` | `palette.hex_format` |
| 6 | `nameKo` 비어 있지 않음 | `palette.name_empty` |
| 7 | `lab` 범위 | `palette.lab_range` |
| 8 | hex→Lab 계산값과 `lab`의 ΔE76 ≤ `hexLabToleranceDe76`. 진단 `values`에 계산값 L/a/b를 실어, 튜너가 그 값을 그대로 복사해 고칠 수 있게 한다 | `palette.hex_lab_mismatch` |
| 9 | `tags`의 모든 값이 `tagVocabulary` 안 | `palette.tag_unknown` |
| 10 | `neutral` 태그 색 개수 ≥ `minNeutralCount` | `palette.neutral_count_below_min` |
| 11 | `neutral` 태그 색의 C* ≤ `neutralMaxChroma` | `palette.neutral_chroma_exceeds_max` |

현재 파일은 11개 검사를 모두 통과한다 (최대 ΔE76 0.0078, neutral 16개, neutral 최대 C* 31.3).

### 2.5 색 추가·변경 절차

1. hex를 정한다. 실제 의류에 존재하는 색이어야 한다.
2. `lab`은 COLOR_SPEC §1.2로 계산해 소수 2자리로 넣는다. 도구가 없으면 임의 값을 넣고 로더를 돌리면 §2.4-8 진단이 계산값을 알려준다.
3. `nameKo`는 §2.2 기준(검색할 이름)으로 정한다.
4. 팔레트 변경은 승인 대상이다 (CLAUDE.md). 변경 후 골든 스냅샷을 다시 만들고 diff를 검토받는다.
5. 파일 안 순서는 계열별(무채색 → 베이지·브라운 → 블루 계 기본색 → 레드 → 핑크 → 퍼플 → 블루·틸 → 그린 → 옐로우·오렌지), 계열 안에서는 L* 내림차순으로 둔다. 읽기 편의일 뿐 엔진은 순서를 쓰지 않는다.

### 2.6 결정 기록 (2026-09-22)

| 항목 | 결정 | 근거 요약 |
|---|---|---|
| 색 개수 | 45 (neutral 16, 유채색 29) | 40 근처로 시작. 버킷이 커서 추출 오차에 안정적, 검수 부담 적음. 부족한 자리는 `no_candidate` 진단이 보여주는 곳에 추가 |
| 명도 분포 | 계열당 3~5단계. 피부 L* 30~78 어느 가정에서도 \|ΔL\| ≥ 15 생존 neutral ≥ 9, 유채색 ≥ 14 | 어두운 피부에서 후보가 남아야 함 (임계 15는 미결, 조회용 계산) |
| `orchid` 추가 | `#B07BA8` (L* 58.2, C* 32.1, h° 331) | 퍼플 L* 50대 공백. 핑크·모브 립(h° 340~360)의 매치 구간이 퍼플 쪽이라 중간 명도가 비면 후보가 얇아짐 |
| `red` 유지 | C* 75로 메인 채도 조건을 거의 못 넘지만 유지 | "레드는 왜 안 나오나"에 진단으로 답한다. 채도 상한을 올렸을 때 들어오는지로 양방향 방향성 확인에 쓴다 |
| 옐로우·오렌지 계열 포함 | butter, peach, mustard, coral, terracotta, rust | 얼굴 제약(피부 색상대 고채도 회피)이 실제로 걸러낼 대상이 있어야 방향성 확인이 가능 |
| Lab 출처 | hex에서 계산, 소수 2자리 | 측정값이 아님. PLAN 미결 "Lab 값 출처" 해소 |
| 무채색 임계 | C* < 5 — 값은 **harmony_rules**에 둔다 (hue 판정 파라미터) | C*=5에서 a/b 1단위 오차가 hue 11°를 흔든다. 팔레트는 C* 1.5와 8.1 사이가 비어 2~8 어디든 분류 동일 |
| 저채도 임계 | 이 파일에 두지 않음. **face_constraints**로 이연 | 소비자가 "피부 색상대 고채도 회피, 저채도 허용"뿐. 그 맥락에서 정한다. 후보 C* 20 (현재 팔레트의 자연 경계 18.8/20.2/22.7) |
| 명도대 보장 검사 | 넣지 않음 | 런타임 `no_candidate`와 역할 중복. "L* ≥ 85 하나"의 근거도 ΔL 하한이 정해져야 나옴 |
| nameKo 변경 | chambray 라이트 블루 / grape 퍼플 / aubergine 딥 퍼플 / fuchsia 핫핑크 | 국내 쇼핑몰에서 검색되는 이름. 오트밀·토프·더스티 로즈는 이미 통용되므로 유지 |

## 3. `harmony_rules_v1.json` — 미작성

조화/부조화 hue 구간, 전략(MATCH/CONTRAST/NEUTRAL)별 목표 구역, 채도 관계, 무채색 임계(C* < 5), 랭킹 가중치가 들어갈 예정.
구간값은 Moon-Spencer 조화론을 착안점으로 한 CIELCh 각도의 초기 추정치이며 먼셀 기준 원 이론의 수치와 일치하지 않는다 (COLOR_SPEC §2). 확정은 2단계 검수.

## 4. `face_constraints_v1.json` — 미작성

의상–피부 ΔL 하한, 피부 색상대 회피 범위와 그 발동 채도 임계, 메인/서브 채도 상한, 피부–립 구분 불가(Rejected) 임계, **저채도 판정 임계**(§2.6에서 이연)가 들어갈 예정.
