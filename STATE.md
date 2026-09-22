# STATE

- 마지막 작업 (2026-09-22): 1-1 팔레트 확정. assets/rules/palette_v1.json(45색, neutral 16), docs/RULES_SCHEMA.md 팔레트 절·명명 절, COLOR_SPEC §1.4를 태그 기준으로 수정, PLAN 미결 정리. 커밋 "feat: 팔레트 v1 (45색) + RULES_SCHEMA 팔레트 절" 후 푸시.
- 다음 할 일: 사용자 지시 후 harmony_rules_v1.json + RULES_SCHEMA §3. 시작 전에 PLAN 미결 "배색 규칙" 항목(hue 구간 부호 유무, 전략별 목표 구역, 채도 관계, 서브/포인트 규칙, 랭킹 가중치)과 규칙 ID·진단 식별자·에러 코드 명명 규칙을 한꺼번에 질문한다. 무채색 임계 C*<5는 이 파일에 넣는다.
- 막힌 것: 없음.
- 참고: face_constraints 단계로 이연된 값 — 저채도 임계(후보 C* 20), ΔL 하한, 피부 색상대 회피 범위. camel(neutral, C* 31, h° 74)이 피부 색상대 제약과 충돌할 수 있음을 그때 확인. Gradle 빌드는 1-2에서.
