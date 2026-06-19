# 포크 후 실데이터 사용 가이드

이 저장소를 **포크해서 Claude Code로 팀 데이터를 현행화하며 운영**하려는 사람을 위한 안내다.

## 이 저장소는 무엇인가
- **Claude 네이티브 팀원 관리 하네스** — 코드 산출물 없이 에이전트 + 스킬 자체가 도구다.
  - `.claude/agents/`(전문가 6), `.claude/skills/`(워크플로우 10: `team-management` 오케스트레이터 + 도메인 6 + `team-data-store`·`exec-report-web`·`checklist-setup`)
  - `CLAUDE.md`(하네스 개요·변경 이력), `docs/`(설계·계획 문서)
- `team-data/`의 데이터는 **전부 가상(데모)** 이다 — 실제 인사정보가 아니다. 가상 3팀(`consulting`·`ai`·`platform`, 10명)과 데모 리포트가 들어 있다.

---

## ⚠️ 0단계 (가장 먼저) — PII / gitignore 안전

**이 저장소는 데모 공유를 위해 `team-data/teams/`를 git에 추적하도록 `.gitignore`가 설정돼 있다.** 포크에 **실제 인사정보**를 넣고 커밋·푸시하면 그대로 원격에 올라간다 — **포크가 public이면 전 세계에 공개된다.**

포크 직후 아래 중 하나를 반드시 선택한다:

- **(권장) 실데이터는 로컬 전용으로 되돌리기** — `.gitignore`의 team-data 블록을 다음 한 줄로 교체:
  ```gitignore
  # 실제 인사정보 — 로컬 전용 (커밋 금지)
  team-data/
  ```
  가장 단순하고 안전하다. (데모/공유용 리포트도 함께 추적 대상에서 빠진다 — 리포트는 데이터로 언제든 재생성 가능하니 무방.)
- 또는 포크를 **private**으로 유지하고 의도한 것만 커밋한다.

**확인 방법** (출력이 나오면 = ignore 중 = 안전):
```bash
git check-ignore team-data/teams/<팀id>/members/<누구>.md
```

> 주의: 현재 `.gitignore`의 데모 리포트 화이트리스트(`!team-data/_reports/...2026-06-19...`)는 **날짜 고정**이라 본인이 새로 만든 리포트는 자동 추적되지 않는다(그건 안전). 하지만 `team-data/teams/`는 추적되므로 위 조치가 필요하다.

---

## 1단계 — 데모 데이터 비우고 본인 팀 초기화
1. 가상 데이터 삭제:
   - `team-data/teams/consulting`, `team-data/teams/ai`, `team-data/teams/platform` 폴더 삭제
   - 데모 리포트 삭제: `team-data/_reports/exec/`, `team-data/_reports/leads/`
2. Claude Code에서 **`checklist-setup`** 실행 → 팀장 체크리스트 정의 + 본인 팀(멤버 명단) 등록. 여러 팀이면 `team-data/teams/<id>/` 구조로 초기화된다.
3. `team-data/calendar/holidays.json`(공휴일)·`team-data/checklist.md`(조직 기본 체크리스트)는 그대로 쓰거나 본인 회사 기준으로 수정한다.

## 2단계 — Claude Code로 데이터 현행화
일상 운영은 자연어 요청으로 한다(`team-management` 오케스트레이터가 도메인 전문가에 위임):
- "이번 주 팀 체크리스트 돌려줘", "OOO 1:1 기록", "휴가 등록", "프로젝트 투입/가동률", "자격증 현황", "팀 현황 리포트", "상급자 보고용 웹페이지"
- 데이터 변경은 `team-data-store` 스키마(slug·ID·날짜·연차 차감 규칙)를 강제하므로, **수기 편집보다 도구를 통한 갱신을 권장**한다(정합성 자동 유지).

## 3단계 — 하네스 업데이트 받기 (upstream sync)
데이터가 도구와 같은 저장소에 있어, 원본(upstream)의 데모 데이터 변경과 본인 데이터가 충돌할 수 있다. **도구만 선별 동기화**한다:
```bash
git remote add upstream https://github.com/Soonyeon-Kim/team-management-harness.git
git fetch upstream
# 도구(.claude·docs·CLAUDE.md)만 가져오고 본인 team-data는 그대로 둔다
git checkout upstream/main -- .claude docs CLAUDE.md
```
`team-data/`는 본인 운영 데이터이므로 upstream에서 덮어쓰지 않는다.

## 이식성 메모
- 스킬·에이전트는 **상대경로(`team-data/...`)만** 사용 → OS/경로 무관하게 동작(하드코딩 절대경로 없음).
- 사용자 메모리(`~/.claude`)는 포크에 포함되지 않는다 — 개인 작업 선호는 새 환경에서 새로 쌓인다.
- 파일 기반이라 별도 서버·DB·빌드가 필요 없다.

---

## 한눈 요약
| 단계 | 할 일 |
|---|---|
| 0 | **gitignore로 team-data 다시 제외(또는 private 유지)** — 실데이터 노출 방지 |
| 1 | 데모 데이터 삭제 → `checklist-setup`으로 본인 팀 초기화 |
| 2 | Claude Code 자연어 요청으로 데이터 현행화 |
| 3 | 하네스 업데이트는 `.claude`·`docs`·`CLAUDE.md`만 선별 sync |
