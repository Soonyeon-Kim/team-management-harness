# 멀티-팀 상급자 보고 (트랙 B) 구현 계획 — 실행 완료 기록

> 설계: `docs/superpowers/specs/2026-06-18-multiteam-report-design.md`. 트랙 A(멀티-팀원 가독성)는 `2026-06-17-exec-report-redesign.md`에 흡수·실행됨. 본 문서는 트랙 B(멀티-팀)의 실행 계획이자 완료 기록이다.

**Goal:** `team-data/`를 단일 팀 전제에서 **여러 동격 팀**을 담는 구조로 확장하고, 팀별 자체완결 리포트 + 조직 요약(드릴다운)을 산출한다. 레거시 단일-팀 동작은 하위 호환으로 보존.

## Phase B1 — 데이터 모델 + 팀 루트 해석
- `team-data-store/SKILL.md`에 `teams/<id>/` 레이아웃 + "멀티-팀 모드 (팀 루트 해석)" 규칙 명문화.
- 모드 판별: `teams/` 없으면 `team_root=team-data/`(레거시), 있으면 `team-data/teams/{team_id}/`.
- 전사 공통(항상 루트): `calendar/holidays.json`. 체크리스트: `{team_root}/checklist.md` 우선, 루트 fallback.

## Phase B2 — 도메인 스킬·에이전트 팀 인식
- 6 스킬(leave/cert/assignment/growth/oneonone/task) + 6 에이전트에 "팀 루트 해석" 프리앰블 추가.
- 본문 `team-data/<x>` → `{team_root}/<x>`로 해석. 공휴일만 루트 고정. 에이전트 입력에 `team` 인자 추가.

## Phase B3 — 오케스트레이터(team-management)
- Phase 0에 팀 확정 분기(teams/ 유무 → 활성 팀 수 → 명시/되묻기/조직전체).
- 팬아웃 전원에 동일 `team_id` 전달. Phase 5에 팀별 출력 경로 + 조직 요약 흐름 추가.

## Phase B4 — exec-report-web 팀 스코프 + 조직 요약
- 팀별 리포트: `team_root`에서 team.md 읽고 `_reports/<team-id>/{날짜}-exec-report.html` 출력(레거시는 기존 경로).
- 신규 `org-template.html`: 조직 KPI(팀별 집계)·팀 비교 미니표·조직 Top 위험(팀명 태그)·팀 바로가기(상대경로 `./<id>/...`). 출력 `_reports/{날짜}-org-summary.html`.

## Phase B5 — 이행 + 데모팀 + 검증
- DS팀 데이터를 `team-data/teams/ds/`로 이행(calendar/_reports 루트 유지).
- 데모 `team-data/teams/ai/`(가상 3명) 추가 — 멀티-팀 E2E.
- `checklist-setup` 멀티팀 초기화/팀 추가 보강.
- E2E 산출: `_reports/ds/...`, `_reports/ai/...`, `_reports/2026-06-18-org-summary.html`(상대링크 드릴다운).

## 검증 결과 (2026-06-18)
- 이행 후 루트 직속 도메인 폴더 부재(MIGRATED-OK), `teams/ds`·`teams/ai` 구조 정상.
- 팀별 리포트 2 + 조직 요약 1 생성, 미치환 토큰 0, 외부 의존성 0(CLEAN), 상대링크 2.
- 데모 팀 JSON 5종 유효. 12개 스킬/에이전트 team_root 프리앰블 확인.

## 비범위(YAGNI)
권한제어, 팀원 팀 이동 이력, 3단계+ 조직 롤업 제외. 코드 산출물 없음(Claude 네이티브 툴).
