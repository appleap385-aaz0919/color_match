# CONTRACT — 단계 계약, 모듈 경계, 이관 규칙

파이프라인의 모든 단계(Stage)가 따르는 결과 계약과 모듈 간 경계를 정한다.
코드와 문서가 어긋나면 문서를 먼저 고치고(사용자 승인) 코드를 맞춘다.
`[미결]` 표시는 값이나 정의가 아직 결정되지 않았다는 뜻이다. PLAN.md 미결 사항과 대응한다.

## 1. 적용 범위

파이프라인 7단계 전부에 적용된다. 1단계에서 구현하는 것은 Stage 5(HarmonyEngine), Stage 6(PaletteMap)과
Stage 4의 출력 형태(RelativeFeature)다. 나중 단계(Capture, FaceSegment, ColorExtract)도 같은 StageResult를 반환한다.

## 2. StageResult 계약

모든 단계는 `StageResult<T>`를 반환한다. 세 갈래다.

| 갈래 | 뜻 | 누가 해결하나 | 예 |
|---|---|---|---|
| `Success(value, confidence, diagnostics)` | 정상 출력 | — | 추천 3세트 산출 |
| `Rejected(reason, confidence, diagnostics)` | 입력이 처리 조건을 못 채움. **사용자가 재시도하면 해결될 수 있음** | 사용자 | 조명 부족, 얼굴 미검출, 피부–립 색이 구분 불가 |
| `Failed(error, diagnostics)` | **사용자 재시도로 해결되지 않음.** 코드 또는 규칙·팔레트를 고쳐야 함 | 개발자·튜닝 담당 | `internal_error`: 규칙 JSON 검증 실패, 불변식 위반 / `no_candidate`: 제약 조합으로 후보 0개 (§2.5) |

구분 기준: "사용자가 무엇을 바꾸면 되는가"를 말할 수 있으면 Rejected, 없으면 Failed.
`Rejected.reason`은 사용자에게 보여줄 메시지 코드와 문구를 가진다.
`Failed.error`는 개발자용 코드·메시지·원인이며, 사용자에게는 일반 오류 문구만 보인다.

### 2.1 confidence
- 범위 `0.0 ~ 1.0`, Double. 세 갈래 모두 가진다.
- Success: 출력의 신뢰도. Stage 5–6에서는 채택 세트의 점수가 판정 경계에 얼마나 가까운지로 정한다
  (산식은 RULES_SCHEMA에서, 값은 JSON에서). [미결]
- Rejected: 거절 판단의 확신도. 임계 근처면 낮다. UI가 "다시 시도" 안내 강도를 조절하는 데 쓴다. 정의는 [미결].
- Failed: 항상 `0.0`.

### 2.2 diagnostics
- `List<Diagnostic>`. 순서는 발생 순서이며 결정론적이다.
- `Diagnostic(stage, ruleId, level, message, values)`
  - `stage`: 단계 식별자 (예: `rules_load`, `harmony`, `palette_map`)
  - `ruleId`: 적용/위반한 규칙 ID. 규칙 JSON의 ID와 같은 문자열.
    규칙과 무관한 항목도 명시적 식별자를 쓴다 (예: `input.lab_range`, `palette.hex_lab_mismatch`)
  - `level`: `INFO` (적용됨), `WARN` (경계 근처·클램프 발생), `REJECT` (후보 탈락 사유), `FAIL` (검증 실패·내부 오류)
  - `message`: 사람이 읽는 문장. 수치는 문장에 박지 않고 `values`에 담는다
  - `values`: 이름 → 수치(Double). 삽입 순서 유지. 픽셀 데이터·이미지 바이트 금지
- 세 갈래 모두 diagnostics를 가진다. Rejected/Failed는 최소 1개(사유)를 가져야 한다.
- 탈락한 후보는 전부 `REJECT` 항목을 남긴다. "왜 이 색이 안 나왔는가"에 답할 수 없는 출력은 계약 위반이다.
- `ruleId`, 진단 식별자, 에러 코드의 명명 규칙은 RULES_SCHEMA.md에서 세 종류 각각 정한다. 위 예시는 규칙이 정해지면 그에 맞춘다.

**기록 범위 ≠ 스냅샷 범위.** 위 원칙은 엔진 출력(`Recommendation`)에 대한 것이다. 골든 스냅샷은 사람이 diff를 읽을 수 있어야
하므로 범위를 좁힌다. 팔레트 60색이면 60 × 3전략 × 3역할 = 540건 이상이 되어 diff가 수천 줄이 되면 골든의 목적이 무너진다.

| 산출물 | 담는 것 |
|---|---|
| 엔진 출력 `Recommendation` | 채택 세트 + 탈락 후보 전부의 상세 사유(수치 포함) |
| `testdata/golden/snapshots/kotlin.json` (골든) | 채택 세트 + **탈락 요약**: 규칙 ID별 탈락 건수, 각 규칙에서 탈락한 색 id 목록 |
| `testdata/golden/snapshots/scratch.json` (스크래치) | 상세 사유 **전량**. 케이스 하나를 파고들 때 쓴다 |
| `golden-diff` 비교 대상 | 채택 세트 + 탈락 요약 |

### 2.3 예외와 조용한 실패
- core/는 예외를 호출자에게 던지지 않는다. 모든 오류 경로는 `Failed`로 끝난다.
- catch 후 빈 결과·기본값·null로 대체하는 것을 금지한다. 처리할 수 없으면 `Failed`, 입력 문제면 `Rejected`.
- 입력 검증은 단계 진입 시점에 한 번, 명시적으로 한다 (§5).

### 2.4 결정론
- 같은 입력(색값 + 규칙 JSON)에 항상 바이트 단위로 같은 출력.
- 정렬은 명시적 비교자를 쓰고, 마지막 동점 처리는 항상 팔레트 `id` 오름차순(문자열 비교).
- `HashMap`/`HashSet`의 순회 순서에 의존하지 않는다. 순서가 필요한 곳은 `List`, `LinkedHashMap`, 정렬된 컬렉션.
- 시간, 랜덤, 로케일, 환경 변수, 시스템 프로퍼티에 의존하지 않는다.
- 부동소수 계산 순서를 바꾸지 않는다(리팩터링 시 골든 diff로 확인). 출력 JSON의 소수 자릿수는 고정한다 (자릿수는 [미결]).
- LLM, 학습 모델, 확률적 요소를 Stage 5–6에 넣지 않는다.

### 2.5 Failed 에러 코드
Failed는 하나가 아니다. "누가 무엇을 고쳐야 하는가"가 다르므로 코드를 나눈다.

| 코드 | 뜻 | 고치는 것 |
|---|---|---|
| `internal_error` | 실제 내부 오류. 규칙 JSON 검증 실패, 불변식 위반, 예상 밖 상태 | 코드 (또는 JSON 문법·구조) |
| `no_candidate` | 규칙·팔레트 조합으로 어떤 전략·역할에서 후보가 0개. 코드는 정상 | 파라미터 또는 팔레트 구성 |

`no_candidate` 예: L*이 30인 어두운 피부에 ΔL 하한 15를 걸면 의상은 L* 45 이상이거나 15 이하여야 하는데,
팔레트 구성에 따라 중성형에서 후보가 남지 않을 수 있다. 재시도해도 같지만 개발자가 코드를 고칠 일도 아니다.

`no_candidate`의 diagnostics는 **이 진단만 보고 "무엇을 완화하면 후보가 생기는지"를 판단할 수 있어야 한다.**
튜닝 단계에서 가장 자주 보게 될 화면이다. 반드시 담는 것:
1. 어떤 전략, 어떤 역할에서 발생했는가
2. 각 제약이 몇 개의 후보를 걸렀는가 (규칙 ID별 건수, 적용 순서대로)
3. 마지막까지 남았다가 탈락한 후보 상위 K개와 각각의 탈락 사유·수치 (K는 [미결], 예: 5)

뷰어는 `no_candidate`를 빈 칸으로 두지 않고 위 세 가지를 그대로 표시한다.
[미결] 규칙 JSON 검증 실패를 `internal_error`에 두는가, `rules_invalid`로 따로 떼는가.

## 3. 모듈 경계

의존 방향 (화살표 = 의존한다):
```
core/rules ──▶ core/contract
    │
    └────────▶ core/color
```
데이터 흐름:
```
assets/rules/*.json ─(문자열)─▶ RulesLoader ─▶ RuleSet
testdata/golden/cases.json  ─▶ GoldenSnapshotMain ─▶ Recommend.run ─▶ snapshots/kotlin.json  (요약)  ─▶ prototype/
testdata/scratch/cases.json ─▶ GoldenSnapshotMain ─▶ Recommend.run ─▶ snapshots/scratch.json (전량)  ─▶ prototype/
```

| 모듈 | 역할 | 의존 | 금지 |
|---|---|---|---|
| `core/contract` | StageResult, Diagnostic, RejectReason, FailureInfo | 없음 | 도메인 로직 |
| `core/color` | sRGB↔XYZ↔Lab↔LCh, ΔE76/ΔE00, 원형 ΔHue, RelativeFeature | 없음 | 규칙·팔레트 지식, 임계값 |
| `core/rules` | 규칙 JSON 로딩·검증, 조화 판정, 얼굴 제약, 후보 평가, 세트 조합·랭킹 | contract, color, kotlinx-serialization-json | 파일·네트워크·안드로이드·java.*, 하드코딩 파라미터, println |
| `assets/rules/` | 규칙 JSON 3종. 튜닝 값의 단일 진실 | — | 코드에서 경로로 직접 열기 |
| `testdata/` | golden(입력·스냅샷), scratch(임시 입력) | — | 프로덕션 코드가 참조 |
| `prototype/` | 렌더링 전용. `index.html` = 결과 뷰어, `palette.html` = 팔레트 스와치 (별개 파일) | 스냅샷 JSON, 팔레트 JSON | 색 계산, 규칙 로직, 입력 데이터 보관. 유일한 예외: 저장된 Lab에서 C* = √(a²+b²), h° = atan2(b, a)를 표시용으로 계산하는 것. 색 공간 변환은 금지 |
| `tools/` | 스냅샷 diff 등 개발 도구 | core 가능 | core가 tools를 의존 |

공통 금지 (core/* 전부): 얼굴 이미지·픽셀 데이터를 저장하거나 diagnostics에 넣지 않는다.
Stage 2–3이 들어와도 core는 영역별 대표색(Lab)만 다룬다.

### 3.1 규칙 로딩 경계
`RulesLoader`는 JSON **문자열** 세 개를 받는다. 파일을 여는 쪽은 호출자다.
- JVM 테스트/도구: `java.io.File` → String
- 안드로이드: `AssetManager.open()` → String
core/rules는 어느 경우인지 모른다.

## 4. Stage 5–6 인터페이스 형태 (초안)

구현 중 바뀔 수 있다. 바꾸기 전에 보고하고 이 절을 갱신한다.

```kotlin
// core/color
data class RelativeFeature(
    val deltaHue: Double?,   // 립 hue − 피부 hue, 부호 있는 원형 차. 한쪽이 무채색이면 null (COLOR_SPEC §2)
    val deltaL: Double,
    val deltaC: Double,
    val skin: Lch,
    val lip: Lch,
)

// core/rules
RulesLoader.load(harmonyJson: String, constraintsJson: String, paletteJson: String): StageResult<RuleSet>

HarmonyEngine.evaluate(skin: Lab, lip: Lab, rules: RuleSet): StageResult<CandidateEvaluation>
    // Stage 5: 전략별 목표 구역 산출 → 팔레트 각 색을 전략×역할별로 평가 (점수, 적용 규칙 ID, 탈락 사유)

PaletteMapper.assemble(evaluation: CandidateEvaluation, rules: RuleSet): StageResult<Recommendation>
    // Stage 6: 통과 후보로 메인/서브/포인트 세트 조합 → 전략별 1세트 랭킹

Recommend.run(skin: Lab, lip: Lab, rules: RuleSet): StageResult<Recommendation>   // 5 → 6 연결

Recommendation(
    input: InputEcho,                        // skin/lip의 Lab·LCh·hex, RelativeFeature (뷰어 표시용)
    strategies: List<StrategyResult>,        // MATCH, CONTRAST, NEUTRAL 순서 고정
    rejected: List<RejectedCandidate>,       // 색 id, 전략, 역할, 규칙 ID, 사유
)
StrategyResult(strategy, set: ColorSet?, score: Double, appliedRules: List<String>, runnerUps: List<ColorSet>)
ColorSet(main: PaletteColor, sub: PaletteColor, point: PaletteColor, areaRatio: AreaRatio /* 60/30/10, JSON 값 */)
```

원칙: 뷰어가 표시할 값은 모두 여기서 계산해 실어 보낸다 (hex, Lab, LCh, Δ값, 점수, 규칙 ID, 사유).
뷰어는 계산하지 않는다.

## 5. 입력 검증

### 5.1 색 입력 (Stage 5 진입)
- Lab 범위: `L ∈ [0,100]`, `a, b ∈ [-128,127]`. 벗어나면 `Failed(input.lab_range)`.
  상위 단계의 버그이므로 Rejected가 아니다.
- 피부–립 구분 불가 (ΔE가 임계 미만) → `Rejected(lip_not_distinct)`. 임계값은 face_constraints JSON. [미결]
- 립 채도가 임계 미만일 때 Rejected로 할지는 [미결].

### 5.2 규칙 JSON (RulesLoader)
검증 실패는 전부 `Failed`이며, diagnostics에 **항목 경로**(예: `harmony_rules.hue_bands[2].max`)와 **이유**를 남긴다.
첫 오류에서 멈추지 않고 발견한 오류를 모두 모아 반환한다. 튜닝 단계에서 JSON을 손으로 고치므로,
잘못된 값이 조용히 통과하면 원인 추적이 불가능해진다.

최소 검사 항목:
1. 필수 필드 존재, 타입, `version` 일치
2. hue 구간이 정의 구간 전체를 빈틈없이 덮는가. 구멍이 있으면 판정 불가 구간이 생긴다.
   (부호 없는 |ΔHue|면 0~180, 부호 있는 표현이면 −180~180. 어느 쪽인지는 RULES_SCHEMA에서 확정 [미결])
3. hue 구간끼리 겹치지 않는가. 경계는 반개구간 `[min, max)`로 통일한다
4. 팔레트 각 색의 `hex`와 `lab`이 일치하는가 — hex→Lab 계산값과 선언 Lab의 ΔE가 허용 오차 이내.
   수기 입력 시 어긋나기 쉽다. 허용 오차는 [미결]
5. 팔레트에 중성색/저채도 색이 최소 N개 이상인가 — 중성형 전략에서 후보 0개가 되는 것을 방지.
   N은 [미결, 팔레트 구성과 함께 제안 후 승인]
6. 팔레트 `id` 유일, `hex` 형식 `#RRGGBB`, 태그가 허용 목록 안에 있는가
7. 수치 범위: 비율 `0~1`, 각도 `0~360`, L `0~100`, 채도 `≥ 0`
8. 참조 무결성: 규칙이 가리키는 태그·전략·역할 이름이 정의돼 있는가

이 목록은 RULES_SCHEMA.md와 함께 늘어난다. 스키마에 항목이 추가되면 검사도 추가한다.

## 6. 안드로이드 이관 구조 규칙

목표: `core/contract`, `core/color`, `core/rules` 디렉토리를 안드로이드 프로젝트에 **소스 수정 없이** 복사해
`:core:contract`, `:core:color`, `:core:rules` 모듈로 쓴다.

1. **디렉토리 = Gradle 경로.** `core/<module>/build.gradle.kts`,
   `core/<module>/src/main/kotlin/com/colormatch/core/<module>/`, 테스트는 `src/test/kotlin/…`.
2. **모듈은 `kotlin("jvm")`을 유지한다.** 안드로이드 앱 모듈은 순수 JVM 모듈을 그대로 의존할 수 있다.
   `com.android.library`로 바꿀 필요가 없다. JVM target 17 (AGP 8 기준).
3. **안드로이드 API를 쓰지 않는다.** `android.*`, `androidx.*` 금지. 이 모듈들은 안드로이드 없이 유닛테스트가 돌아야 한다.
4. **`java.*` API를 쓰지 않는다.** 1차 목적은 안드로이드 이관이다.
   `java.*` 중 상당수가 안드로이드에서 동작이 다르거나 API 레벨 제약이 있다:
   - `String.format` — 기본 로케일에 따라 소수점·숫자 표기가 달라져 결정론을 깨뜨린다
   - `java.time` — 최소 API 레벨 제약 (26 미만은 desugaring 필요)
   - `java.text.DecimalFormat` 등 포맷 계열 — 플랫폼별 부동소수 표기 차이
   - `java.awt.*`, `javax.*` — 안드로이드에 없다
   허용: Kotlin 표준 라이브러리(`kotlin.math`, `kotlin.text`, `kotlin.collections`), `kotlinx.serialization`.
   부수적 이득으로, 이 규칙을 지키면 나중에 Kotlin Multiplatform으로 전환할 때 `src/main` → `src/commonMain`
   이동만으로 끝난다. 단, KMP를 하지 않기로 결정해도 이 규칙은 그대로 유효하다. 근거는 안드로이드다.
   예외가 필요하면 사전 승인을 받고 이 절에 기록한다. 테스트·도구 코드는 이 규칙의 대상이 아니다.
5. **자산 로딩은 앱 계층.** 규칙 JSON은 안드로이드 `assets/rules/`에 복사되고 `AssetManager`로 읽어
   문자열로 넘긴다. core는 §3.1대로 문자열만 받는다.
6. **로그 출력 금지.** core에서 `println`, `System.err`를 쓰지 않는다. 진단은 diagnostics로만. 앱 계층이 로그로 옮긴다.
7. **얼굴 이미지 저장 금지.** 파이프라인은 이미지 바이트를 디스크·캐시에 쓰지 않는다.
   저장 가능한 것은 색값(Lab)과 진단 수치만. Stage 2–3 구현 시 CaptureGate/FaceSegment 계약에 다시 명시한다.
8. **직렬화 형식 고정.** 스냅샷과 앱 디버그 출력은 같은 `Recommendation` JSON 형식에서 파생한다.
   요약(골든)과 전량(스크래치·앱 디버그) 두 뷰만 존재하며 (§2.2), 뷰어는 이 두 형식만 안다.

## 7. 변경 절차

- 이 문서의 변경은 사용자 승인 후 문서 → 코드 순서로 한다.
- 공개 시그니처(§4) 변경은 구현 전에 보고한다.
- `testdata/golden/snapshots/baseline.json` 갱신은 `golden-diff` 출력을 검토받고 승인 후에만 한다.
  갱신 커밋 메시지에 바뀐 규칙과 이유를 쓴다.
- 규칙 JSON 값 변경도 승인 대상이다 (CLAUDE.md 최우선 규칙).
