# meta_harness

## 하네스: 팀원 관리툴

**목표:** 팀장이 팀장 체크리스트를 기준으로 팀원의 1:1·업무·성장·휴가를 한곳에서 관리하도록 돕는 Claude 네이티브 툴.

**트리거:** 팀원 관리(1:1, 업무/태스크, 성장/개발, 휴가/근태, 주간 체크리스트, 팀 현황 리포트, 상급자 보고용 웹페이지/HTML 리포트) 관련 작업 요청 시 `team-management` 스킬을 사용하라. 최초 실행이면 `checklist-setup`으로 셋업을 먼저 안내한다. 단순 사실 질문은 직접 응답 가능.

**상태:** 모든 데이터는 프로젝트 루트 `team-data/`에 영속(파일 기반). 코드 산출물 없음 — 에이전트+스킬 자체가 툴이다.

**변경 이력:** (날짜는 항목을 기록하는 시점의 **오늘 날짜**로 기재한다)
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-06-17 | 초기 구성 (전문가 4 + 오케스트레이터, 하이브리드 실행, 파일 기반 저장소) | 전체 | - |
| 2026-06-17 | 실제 팀장 체크리스트(자격증·휴가·역량 3축) 반영: certification-manager 신규, leave/growth 스킬 보강, checklist.md 작성, task-tracker 보조 전환 | agents/certification-manager, skills/certification-management·leave-management·growth-coaching·checklist-setup·team-management·team-data-store, team-data/checklist.md | 팀장 보유 체크리스트 제공 |
| 2026-06-17 | 휴가비 신청 자격 규칙 추가(연차 10일 초과 사용 시에만 신청 가능) | skills/leave-management·team-data-store, agents/leave-manager | 사내 휴가비 규정 |
| 2026-06-17 | 휴가 종류(연차/보상/경조사/출산/육아휴직/기타)·단위(연차1·반차0.5·반반차0.25) 모델 추가, 연차휴가만 used 차감 | skills/leave-management·team-data-store, agents/leave-manager | 휴가 종류별 연차 차감 규정 |
| 2026-06-17 | assignment-manager(프로젝트 투입/가동률) 신규: assignments 데이터·스킬·에이전트 추가, leave/growth가 프로젝트 일정·역량 매칭에 참조, 체크리스트 프로젝트 항목 연결 | agents/assignment-manager, skills/assignment-tracking·team-data-store·leave-management·growth-coaching·team-management, team-data/checklist.md | 컨설팅 팀 프로젝트 투입 이력 관리 필요(팀장 지적) |
| 2026-06-17 | 상급자 보고용 자체완결 HTML 리포트 신규: exec-report-web 스킬(SKILL.md + template.html, 인라인 CSS·CSS 막대 차트, 외부 의존성 없음, @media print/PDF), team-management 리포트 흐름에 HTML 산출 5단계 연결(전문가 결과 재사용). 신규 에이전트 없음(프레젠테이션 스킬 1개) | skills/exec-report-web·team-management | 상급자에게 보고할 웹페이지 필요 |
| 2026-06-17 | 가동률 정의 재정립: 시점 스냅샷 → **기간 기반(가동일÷가용일×100, 기본 YTD)**. 가동=고객 대상 활동(프로젝트·교육·행사) 전부·종료 투입도 기간 집계, 가용일=영업일−승인휴가−공휴일. 옛 스냅샷은 '현재 동시 투입률'(스태핑)로 개명. 공휴일 데이터(calendar/holidays.json) 신규 | skills/team-data-store·assignment-tracking·exec-report-web, agents/assignment-manager, team-data/calendar/holidays.json | 단발 교육·종료 프로젝트가 0%로 잡히는 스냅샷의 한계, 컨설팅 가동률 표준 반영(팀장 논의) |
| 2026-06-18 | 트랙 A — exec 보고 HTML 모던 SaaS 개편 + 멀티-팀원 가독성: 라이트 헤더+액센트·KPI 상태색 카드·위험 CSS 색점+배지(RISK_ICON→RISK_PILL), sticky `<thead>`, 주의 팀원 행 강조(tr.attn·M_ATTN_CLASS), 위험 무음 절단 방지(RISK_MORE_NOTE "외 N건"), 팀원 심각도순 정렬·KPI 편차 병기 | skills/exec-report-web(template.html·SKILL.md) | 상급자 보고 가독성·다인원 견고성 개선 |
| 2026-06-18 | 트랙 B — **멀티-팀 지원**: 데이터 모델 `team-data/teams/<id>/` 도입(teams/ 유무로 레거시/멀티 판별, 하위호환), 전 도메인 스킬·에이전트에 "팀 루트 해석" 프리앰블(team-data/x→{team_root}/x, 공휴일만 루트 고정), team-management Phase0 팀 확정·팬아웃 team 전달·Phase5 조직 흐름, exec-report-web 팀 스코프 출력(_reports/<id>/)+신규 org-template.html(조직 KPI 집계·팀 비교표·팀명 태그 위험·상대경로 드릴다운), checklist-setup 멀티팀 초기화. DS팀→teams/ds/ 이행·데모 AI팀 추가로 E2E 검증 | skills/team-data-store·team-management·exec-report-web·checklist-setup + 6 도메인 스킬, agents/6, team-data/teams/ | 동격 팀 다수를 한 도구로 관리·조직 통합 보고 필요 |
| 2026-06-19 | 상급자용/팀장용 보고 분리: 산출물 폴더 2벌(`_reports/exec/`=org-summary+팀 페이지(전사 복귀 링크 `{{NAV_BACK}}`), `_reports/leads/`=팀장 단독 파일(전사/타팀 링크 없음)), `template.html` `.backlink`(print 숨김), 요청 표현으로 audience 분기(기본 상급자)·멀티-팀 모드 전용·레거시 하위호환. 2026-06-18 세트 재생성 | skills/exec-report-web(template.html·SKILL.md)·team-management | 상급자는 전사↔팀 양방향 필요, 팀장은 자기 팀만 접근 |
