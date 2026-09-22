# Color Match — 립 색 기반 의상 배색 추천

립스틱을 바른 얼굴 사진에서 피부색과 립 색을 추출해, 어울리는 의상 색 조합(메인/서브/포인트)을
고정 팔레트에서 추천하는 안드로이드 온디바이스 앱. 서버 없음. 퍼스널 컬러 진단과 무관.

**현재 단계: 1단계 — 배색 규칙 엔진(순수 Kotlin/JVM) + 렌더링 전용 뷰어.**
카메라·안드로이드·얼굴 인식은 이번 단계에 없다. 진행 상황은 PLAN.md, 인수인계는 STATE.md.

## 최우선 규칙: 의사결정은 항상 사용자에게 묻는다

임의로 결정하고 진행하지 않는다. 판단이 서지 않으면 묻는 쪽을 택한다.

반드시 사전에 묻고 승인을 받는 것:
- 설계 방향, 아키텍처, 모듈 구성의 변경
- 파일/디렉토리 구조의 추가·삭제·이동
- 라이브러리나 의존성 추가
- 규칙 파라미터 값의 결정이나 변경
- 팔레트 색 구성, 임계값 등 튜닝 대상 값
- 기존에 합의된 내용과 다르게 가야 한다고 판단될 때
- 스펙이 모호하거나 해석이 갈릴 때

묻지 않고 진행해도 되는 것:
- 이미 승인된 범위 안에서의 구현
- 변수명, 함수 분리 등 되돌리기 쉬운 내부 결정
- 오타나 명백한 버그 수정

하지 않는 것:
- "일단 이렇게 해두고 나중에 바꾸자"
- 모르는 값을 추정해서 채우기. 비워두고 질문한다.
- 선택지가 여럿일 때 하나를 골라 진행하기. 비용 비교와 추천안을 내고 사용자가 결정한다.

## 절대 규칙 (core/)

1. core/에는 안드로이드 API와 `java.*` API를 쓰지 않는다. Kotlin 표준 라이브러리와 kotlinx.serialization만.
   이유는 docs/CONTRACT.md §6 (1차 목적은 안드로이드 이관).
2. 규칙 파라미터를 코드에 하드코딩하지 않는다. 모든 값은 assets/rules/*.json에서 읽는다.
3. 색 계산은 CIELAB/LCh에서 한다. RGB/HSV 공간에서 거리·차이를 계산하지 않는다.
4. 판정은 절대 색값이 아닌 상대값(ΔHue, ΔL, ΔC)으로 한다.
5. 결정론을 유지한다. 같은 입력 → 바이트 단위로 같은 출력. 랜덤·시간·로케일·HashMap 순회 순서 의존 금지.
6. 얼굴 이미지를 저장하지 않는다. 진단(diagnostics)에도 픽셀 데이터를 넣지 않는다.
7. core/는 파일 경로·네트워크를 모른다. 규칙 JSON은 문자열로 받는다.
8. 엔진은 하나다. 뷰어(prototype/)나 도구에 색 계산·규칙 로직을 다시 구현하지 않는다.

## 작업 전 읽을 문서

순서대로: STATE.md → PLAN.md → docs/CONTRACT.md → docs/COLOR_SPEC.md.
규칙 JSON이나 core/rules를 다루면 docs/RULES_SCHEMA.md도 읽는다.

## 구조

```
core/contract   StageResult(Success/Rejected/Failed), Diagnostic        의존성 없음
core/color      sRGB↔Lab↔LCh, 색차(ΔE), 상대값(RelativeFeature)          의존성 없음
core/rules      규칙 로딩·검증, 조화 판정, 제약 필터, 후보 평가, 세트 조합·랭킹
                → contract, color, kotlinx-serialization-json
assets/rules/   규칙 JSON 3종 (단일 진실). 튜닝은 여기서만
testdata/       golden/cases.json(입력), golden/snapshots/(결과), scratch/(임시 입력)
prototype/      렌더링 전용 뷰어. 색 계산 없음. 입력 데이터 두지 않음
tools/          golden-diff.mjs (스냅샷 비교)
```

패키지 루트 `com.colormatch`. 모듈 경로 = Gradle 경로 (`:core:contract`, `:core:color`, `:core:rules`).

## 주요 명령어

```
gradlew test                            전체 유닛 테스트
gradlew :core:rules:goldenSnapshot      골든+스크래치 케이스 실행 → testdata/golden/snapshots/
                                        kotlin.json = 채택 세트 + 탈락 요약 / scratch.json = 상세 사유 전량
gradlew -t :core:rules:goldenSnapshot   연속 빌드. 규칙 JSON·케이스·소스 변경 시 자동 재실행
node tools/golden-diff.mjs testdata/golden/snapshots/baseline.json testdata/golden/snapshots/kotlin.json
                                        baseline 대비 변경 항목 출력 (pass/fail 아님)
python -m http.server 8000              루트에서 실행 → http://localhost:8000/prototype/
                                        뷰어는 수동 새로고침 (자동 갱신 없음)
```

Gradle 빌드 파일은 1-2 단계에서 만든다. 그 전까지 gradlew 명령은 동작하지 않는다.

## 금지 사항

- 확신 없는 파라미터를 임의로 정하지 않는다. 질문한다.
- 테스트를 통과시키려고 기댓값·baseline을 고치지 않는다. baseline 갱신은 diff를 검토받고 승인 후에만.
- 골든 케이스에 expected/tolerance를 넣지 않는다 (2단계 스타일리스트 검수 이후). testdata/golden/README.md 참고.
- core/에서 println·로그 출력 대신 diagnostics를 쓴다. 예외를 밖으로 던지지 않고 Failed로 반환한다.
- 조용한 실패 금지. catch 후 빈 결과 반환, 기본값 대체 금지.

## 서브에이전트 (.claude/agents/)

- rules-engine: core/rules 구현 (sonnet)
- test-golden: 테스트·testdata (sonnet). 프로덕션 코드 수정 금지
- reviewer: 읽기 전용 리뷰 (opus). 코드 변경 후 반드시 호출

모든 에이전트는 docs/CONTRACT.md를 먼저 읽는다. 결정이 필요한 사항은 에이전트가 정하지 않고 보고한다.

## 세션 종료 시

STATE.md를 갱신한다: 마지막 작업 / 다음 할 일 / 막힌 것. 3~5줄.
합의된 결정이나 미결 사항이 바뀌었으면 PLAN.md도 갱신한다.
