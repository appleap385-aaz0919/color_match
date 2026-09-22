# PLAN

## 전체 단계

- [ ] 0. 구조 설계 — 파일 트리, 루트 문서, 에이전트, CONTRACT/COLOR_SPEC (문서 작성 완료, 사용자 확인 대기)
- [ ] 1. 배색 규칙 엔진 + 뷰어 (순수 Kotlin/JVM)
- [ ] 2. 검수·튜닝 — 스타일리스트 검수, 골든 expected 확정, 파라미터 튜닝.
      **파라미터 확정은 이 단계에서 한다.** 1단계의 값은 임시값이며 결과의 좋고 나쁨은 여기서 처음 판단한다
- [ ] 3. 조명 실험 — 촬영 조명 변화에 대한 상대값 안정성 검증.
      촬영 사진의 보관 위치와 git 제외 규칙을 이 단계 착수 시 결정한다 (저장소 밖에 두는 방안 포함)
- [ ] 4. 안드로이드 이관 — core 모듈 그대로 포함, 앱 계층에서 assets 로딩
- [ ] 5. 카메라 — CaptureGate (조명/거리 검사)
- [ ] 6. 얼굴 인식 — FaceSegment, ColorExtract, RelativeFeature 연결
- [ ] 7. QA·출시

파이프라인: Capture(1) → FaceSegment(2) → ColorExtract(3) → RelativeFeature(4) → HarmonyEngine(5) → PaletteMap(6) → Present(7).
1단계는 Stage 5–6과 Stage 4의 출력 형태(RelativeFeature 계산)만 구현한다.

## 1단계 세부

진행 순서는 합의된 대로. 각 항목은 시작 전에 열린 질문(미결 사항)을 먼저 해소한다.

### 1-1. 규칙 JSON + docs/RULES_SCHEMA.md
- [ ] 질문 해소: 팔레트 색 수(40/60), 중성색 최소 N, hue 구간 표현(부호 유무), 채도 제약 방식과 값,
      서브/포인트 선정 규칙, 피부 색상대 회피 범위, 피부–립 구분 불가 임계
- [ ] assets/rules/harmony_rules_v1.json — 조화/부조화 구간, 전략별 목표 구역
- [ ] assets/rules/face_constraints_v1.json — ΔL 하한, 피부 색상대 회피(채도 조건부), 메인 채도 상한
- [ ] assets/rules/palette_v1.json — id, nameKo, hex, lab, tags. 팔레트 구성은 제안 후 승인
- [ ] docs/RULES_SCHEMA.md — JSON 실물과 함께 작성
- [ ] 식별자 명명 규칙을 RULES_SCHEMA.md에 한 줄씩 정한다: 규칙 JSON의 규칙 ID / 진단 식별자 / 에러 코드 (세 종류 각각).
      조화 구간값이 먼셀 기준 원 이론과 다른 CIELCh 초기 추정치라는 주의도 파라미터 설명에 적는다 (COLOR_SPEC §2)

### 1-2. Gradle 설정
- [ ] settings.gradle.kts, build.gradle.kts, gradle.properties, wrapper (최초 1회 배포판 다운로드)
- [ ] Kotlin 2.x, JVM target 17, kotlinx-serialization-json, kotlin-test(JUnit5)
- [ ] .gitattributes — 기본 설정(`* text=auto eol=lf`)은 첫 커밋에 포함 완료. wrapper 생성 시 gradlew 관련 규칙만 추가

### 1-3. core/contract
- [ ] StageResult, Diagnostic, RejectReason, FailureInfo
- [ ] 직렬화 형태 (스냅샷 JSON에 어떻게 실리는가)

### 1-4. core/color
- [ ] Srgb, Lab, Lch 값 타입 / ColorConvert / ColorDifference(ΔE76, ΔE00, 원형 ΔHue) / RelativeFeature
- [ ] 테스트: 기준값 왕복 변환, CIEDE2000 Sharma 테스트 쌍, 무채색 hue 처리, 경계값

### 1-5. core/rules
- [ ] model/ (JSON 대응 데이터 클래스) + RulesLoader (검증 항목은 CONTRACT §5.2)
- [ ] HarmonyJudge / ConstraintFilter / CandidateScorer / HarmonyEngine / PaletteMapper / Recommendation
- [ ] 테스트: 로더 검증 실패 케이스 각각, 판정 경계값, 제약 필터, 결정론(같은 입력 2회 → 동일 출력)
- [ ] reviewer 리뷰

### 1-6. 뷰어 (prototype/index.html)
- [ ] 전 케이스 격자 (케이스 × 3전략), 케이스 상세 (규칙ID·점수·탈락 요약; 스크래치 결과면 후보별 상세 사유)
- [ ] 후보 0개(`no_candidate`)는 빈 칸이 아니라 전략·역할·규칙별 탈락 건수·마지막 탈락 후보와 사유를 표시
- [ ] 스크래치 입력 도우미 (피커 2개 → JSON 조각 복사)
- [ ] 색 계산 없음. 자동 갱신 없음 (수동 새로고침)

### 1-7. 골든
- [ ] testdata/golden/cases.json — 15~20 케이스, 입력만 (id, skinLab, lipLab, 설명).
      피부 밝은~어두운, 립 레드/코랄/핑크/누드/버건디 등 고루
- [ ] testdata/golden/README.md — expected 부재 이유와 추가 시점
- [ ] testdata/scratch/cases.json
- [ ] GoldenSnapshotMain + gradle goldenSnapshot 태스크 (+ -t 연속 빌드 확인)
      kotlin.json = 채택 세트 + 탈락 요약(규칙 ID별 건수, 규칙별 탈락 색 id) / scratch.json = 상세 사유 전량
- [ ] tools/golden-diff.mjs — 비교 대상은 채택 세트 + 탈락 요약
- [ ] 첫 baseline.json 확정 (사용자 검토 후 커밋)

### 1단계 완료 조건

기준은 "엔진이 조작 가능한가"다. "결과가 좋은가"는 2단계(스타일리스트 검수)의 몫이다.
1단계의 초기 파라미터는 임시값이며, 2단계 검수 전까지 결과의 좋고 나쁨을 판단하지 않는다.

- [ ] 골든 케이스 전부에서 3전략 × 1세트가 나오거나, 후보 0개인 곳은 `no_candidate` 진단이 사유와 함께 뷰어에 표시된다
- [ ] 규칙 JSON 값을 바꿨을 때 diff가 바뀐 케이스만 보여준다
- [ ] 파라미터를 바꿨을 때 결과가 의도한 방향으로 움직인다. 최소 3개 파라미터에 대해 **양방향**으로 확인한다.
      올렸을 때 줄고 내렸을 때 느는 것까지 본다. 한 방향만 보면 "규칙이 아예 안 걸리는 상태"와 구분되지 않는다
      (예: ΔL 하한을 올리면 피부와 명도가 가까운 의상 후보가 탈락하고, 내리면 다시 들어온다.
      메인 채도 상한을 낮추면 선명한 색이 탈락하고, 높이면 다시 들어온다)

## 확정 사항

| 항목 | 결정 |
|---|---|
| 빌드 | Gradle Kotlin/JVM 멀티모듈, Android 플러그인 없음, JVM target 17 |
| 패키지 루트 | com.colormatch |
| 모듈 | core/contract (신설), core/color, core/rules |
| JSON | kotlinx.serialization |
| 엔진 | Kotlin 단일. JS 엔진 없음. 뷰어는 Kotlin이 떨군 결과 JSON을 렌더링만 |
| 뷰어 열기 | python -m http.server, 수동 새로고침 |
| 스크래치 입력 위치 | testdata/scratch/ (prototype/은 렌더링 전용 유지) |
| 골든 검증 | baseline.json 대비 diff. expected/tolerance는 2단계 검수 이후 |
| 스냅샷 범위 | 기록은 전부, 스냅샷은 요약만. kotlin.json = 채택 세트 + 탈락 요약, scratch.json = 상세 전량. diff 비교 대상 = 채택 세트 + 탈락 요약 (CONTRACT §2.2) |
| Failed 코드 | `internal_error`(내부 오류) / `no_candidate`(제약 조합으로 후보 0개) 구분 (CONTRACT §2.5) |
| 1단계 완료 기준 | "엔진이 조작 가능한가". 결과 품질 판단과 파라미터 확정은 2단계 |
| java.* 금지 | 1차 목적 안드로이드 이관, KMP 전환 가능성은 부수 이득 (CONTRACT §6) |
| 자동 갱신 | 이번 단계 제외 (검증 대상을 늘리지 않기 위해) |

## 미결 사항 (결정 전까지 값을 채우지 않는다)

배색 규칙
- 의상–피부 ΔL 하한 15의 적정성. 어두운 피부에서 어두운 의상이 전부 탈락하지 않는지
- hue 구간을 부호 없는 |ΔHue| 0~180으로 두는가, 부호 있는 −180~180으로 두는가
  (매치형 ±25~45의 양쪽 처리 방식과 연결)
- 메인/서브 채도 상한: 립 채도 대비 비율인가 절대 차이인가, 값은
- 서브 색 선정 규칙: 메인과의 관계(유사/명도 대비/중성)
- 포인트 색: 립 색상 계열 범위(±몇 도), 채도 허용 상한(립과 같은 채도까지)
- 피부 색상대 회피: 색상 범위(오렌지~옐로우의 각도), 회피가 발동하는 채도 임계
- 중성형 "저채도" 판정 채도 임계, 무채색 판정 임계
- 대비형: 100~180 구간 안에서 우선순위(보색 180°에 가까울수록 높은 점수인지)
- 후보 랭킹 점수 구성(가중치), 동점 처리 규칙

계약·수치
- Stage 5 Rejected 조건: 피부–립 ΔE 최소값, 립 채도 최소값 필요 여부
- 팔레트 hex↔lab 일치 허용 오차(ΔE)
- 스냅샷 JSON 출력 소수 자릿수 (제안: 3자리)
- ΔE 공식 사용처: 지각 비교는 ΔE00, 로더 검증은 ΔE76 (제안)
- Rejected의 confidence 정의
- `no_candidate` 진단에 싣는 "마지막까지 남았다가 탈락한 후보" 개수 K (예: 5)
- 규칙 JSON 검증 실패를 `internal_error`에 두는가, `rules_invalid`로 따로 떼는가

팔레트
- 색 개수 40 vs 60
- 중성색/저채도 최소 개수 N
- 계열 태그 목록(뉴트럴/웜/쿨/…)과 태그 부여 기준
- Lab 값 출처(hex에서 계산해 채우는가, 별도 측정값인가)

골든
- 케이스 수(15~20)와 피부톤 × 립 분포
- ~~diff 비교 대상 범위~~ → 해소: 채택 세트 + 탈락 요약 (확정 사항 표)

## v2 이연

- 뷰어 자동 갱신(결과 JSON 폴링)
- 단일 파일 HTML 인라인 빌드(공유용)
- KMP JS 타깃(실시간 피커가 필요해질 때)
- JSON Schema 파일 기반 검증(현재는 로더 코드 검증)
- 골든 expected/tolerance (2단계 검수 이후)
- 립 질감(매트/글로시) 반영
- 사용자 선호 반영(결정론을 깨지 않는 범위에서 필터로만)
- 색 이름 다국어
