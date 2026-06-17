# meta_harness

## 하네스: 팀원 관리툴

**목표:** 팀장이 팀장 체크리스트를 기준으로 팀원의 1:1·업무·성장·휴가를 한곳에서 관리하도록 돕는 Claude 네이티브 툴.

**트리거:** 팀원 관리(1:1, 업무/태스크, 성장/개발, 휴가/근태, 주간 체크리스트, 팀 현황 리포트) 관련 작업 요청 시 `team-management` 스킬을 사용하라. 최초 실행이면 `checklist-setup`으로 셋업을 먼저 안내한다. 단순 사실 질문은 직접 응답 가능.

**상태:** 모든 데이터는 프로젝트 루트 `team-data/`에 영속(파일 기반). 코드 산출물 없음 — 에이전트+스킬 자체가 툴이다.

**변경 이력:**
| 날짜 | 변경 내용 | 대상 | 사유 |
|------|----------|------|------|
| 2026-06-17 | 초기 구성 (전문가 4 + 오케스트레이터, 하이브리드 실행, 파일 기반 저장소) | 전체 | - |
| 2026-06-17 | 실제 팀장 체크리스트(자격증·휴가·역량 3축) 반영: certification-manager 신규, leave/growth 스킬 보강, checklist.md 작성, task-tracker 보조 전환 | agents/certification-manager, skills/certification-management·leave-management·growth-coaching·checklist-setup·team-management·team-data-store, team-data/checklist.md | 팀장 보유 체크리스트 제공 |
| 2026-06-17 | 휴가비 신청 자격 규칙 추가(연차 10일 초과 사용 시에만 신청 가능) | skills/leave-management·team-data-store, agents/leave-manager | 사내 휴가비 규정 |
| 2026-06-17 | 휴가 종류(연차/보상/경조사/출산/육아휴직/기타)·단위(연차1·반차0.5·반반차0.25) 모델 추가, 연차휴가만 used 차감 | skills/leave-management·team-data-store, agents/leave-manager | 휴가 종류별 연차 차감 규정 |
