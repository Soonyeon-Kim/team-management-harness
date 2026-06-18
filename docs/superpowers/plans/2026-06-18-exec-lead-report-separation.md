# 상급자용/팀장용 보고 리포트 분리 구현 계획

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 멀티-팀 보고 HTML을 보고 대상별로 분리한다 — 상급자에겐 전사↔팀 양방향 묶음(`_reports/exec/`), 팀장에겐 자기 팀 단독 파일(`_reports/leads/`).

**Architecture:** 정적 자체완결 HTML이라 접근 제한은 "건네는 파일 단위(공유 경계)"로만 구현된다. 같은 팀 데이터를 두 경로에 저장하되 내용은 동일하고 네비게이션만 다르다(상급자용 팀 페이지에만 "← 전사 현황으로" back 링크). `template.html`에 `{{NAV_BACK}}` 토큰 1개를 추가해 상급자용은 채우고 팀장용은 빈값으로 둔다. 요청 표현으로 대상을 분기하며(기본 상급자), 멀티-팀 모드에서만 동작하고 레거시 단일 팀은 기존 동작을 보존한다.

**Tech Stack:** Claude 네이티브 툴(코드/빌드/테스트 러너 없음). 산출물 = 마크다운 스킬 + 인라인 CSS HTML 템플릿. 검증 = grep + 브라우저 육안.

## Global Constraints

- 자체완결 규칙: 산출 HTML에 외부 리소스 금지(`<script src>`, CDN `http(s)://` CSS/폰트/이미지/차트). 차트는 순수 CSS 막대만.
- 기존 토큰·`<style>`·구조·클래스는 보존하고 **추가만** 한다(트랙 A/B 토큰 계약 유지).
- 적용 범위는 **멀티-팀 모드(`team-data/teams/` 존재)** 전용. 레거시 단일 팀(`teams/` 없음)은 기존 경로 `_reports/{날짜}-exec-report.html` 유지, `{{NAV_BACK}}` 빈값, 폴더 분리 없음.
- back 링크 문구는 정확히 `← 전사 현황으로`. 경로는 상대경로 `../{YYYY-MM-DD}-org-summary.html`.
- 날짜 토큰은 세션 날짜(현재 작업: `2026-06-18`).
- 커밋 메시지는 conventional commits(`feat:`/`docs:`), 끝에 `Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>`.
- 선행 설계: `docs/superpowers/specs/2026-06-18-exec-lead-report-separation-design.md`.

---

### Task 1: `template.html` — back 링크 토큰 + `.backlink` CSS

**Files:**
- Modify: `.claude/skills/exec-report-web/template.html`

**Interfaces:**
- Produces: `{{NAV_BACK}}` 스칼라 토큰(`<body>` 최상단), `a.backlink` CSS 클래스(+ `@media print` 숨김). Task 2가 치환 규칙으로, Task 4가 산출물 주입으로 소비.

- [ ] **Step 1: `{{NAV_BACK}}` 토큰을 `<body>` 최상단에 삽입**

`<div class="page">` 직후·`<header class="report">` 직전에 토큰 한 줄을 넣는다. 아래 old→new로 Edit:

old:
```html
<div class="page">

  <header class="report">
```
new:
```html
<div class="page">

  {{NAV_BACK}}
  <header class="report">
```

- [ ] **Step 2: `a.backlink` 기본 CSS를 `footer.report` 규칙 바로 앞에 삽입**

`<style>` 블록 안, `  /* 푸터 */` 주석 줄(존재) 앞에 규칙을 추가한다. old→new:

old:
```css
  /* 푸터 */
  footer.report{
```
new:
```css
  /* 상급자 묶음 전용 복귀 링크 (팀장용/레거시에선 빈값 치환) */
  a.backlink{display:inline-flex; align-items:center; gap:6px; margin-bottom:14px; padding:7px 13px; text-decoration:none;
    background:var(--card); border:1px solid var(--line); border-radius:999px; color:var(--accent);
    font-size:12.5px; font-weight:650; box-shadow:var(--shadow); transition:border-color .15s;}
  a.backlink:hover{border-color:var(--accent);}
  /* 푸터 */
  footer.report{
```

- [ ] **Step 3: `@media print`에서 back 링크 숨김**

기존 인쇄 블록 첫 줄 뒤에 숨김 규칙을 추가한다. old→new:

old:
```css
  @media print{
    body{background:#fff;} .page{margin:0; max-width:none;}
```
new:
```css
  @media print{
    body{background:#fff;} .page{margin:0; max-width:none;}
    a.backlink{display:none;}
```

- [ ] **Step 4: 검증 (grep)**

Run:
```bash
grep -c "{{NAV_BACK}}" .claude/skills/exec-report-web/template.html
grep -c "a.backlink" .claude/skills/exec-report-web/template.html
grep -c "a.backlink{display:none;}" .claude/skills/exec-report-web/template.html
```
Expected: 1번째 `1`, 2번째 `3`(기본 규칙 2줄 매칭 + hover 1 + print 1 중 `a.backlink` 등장 횟수 ≥3), 3번째 `1`. 최소한 모두 ≥1.

- [ ] **Step 5: 자체완결 회귀 확인 (외부 의존성 0)**

Run:
```bash
grep -nE "https?://|<script src" .claude/skills/exec-report-web/template.html
```
Expected: 매칭 없음(0건).

- [ ] **Step 6: 커밋**

```bash
git add .claude/skills/exec-report-web/template.html
git commit -m "feat: template.html에 {{NAV_BACK}} back 링크 토큰·.backlink CSS(print 숨김) 추가

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 2: `exec-report-web/SKILL.md` — 폴더 2벌 경로 · 대상 분기 · 치환 규칙

**Files:**
- Modify: `.claude/skills/exec-report-web/SKILL.md`

**Interfaces:**
- Consumes: Task 1의 `{{NAV_BACK}}` 토큰.
- Produces: 저장 경로 규약(`_reports/exec/{team_id}/...`, `_reports/leads/{team_id}-{date}.html`, `_reports/exec/{date}-org-summary.html`), "보고 대상 분기" 절, `{{NAV_BACK}}` 치환 규칙. Task 3(오케스트레이터)와 Task 4(산출물)가 소비.

- [ ] **Step 1: 팀별 리포트 절차의 저장 경로(5번)를 대상 분기로 교체**

old:
```markdown
5. 저장 경로(팀 스코프):
   - 레거시 단일 팀 → `team-data/_reports/{YYYY-MM-DD}-exec-report.html`.
   - 멀티-팀 → `team-data/_reports/{team_id}/{YYYY-MM-DD}-exec-report.html` (`{team_id}/`가 없으면 만든다).
   같은 날 재실행이면 덮어쓴다.
```
new:
```markdown
5. 저장 경로(팀 스코프 · 보고 대상):
   - **레거시 단일 팀** → `team-data/_reports/{YYYY-MM-DD}-exec-report.html` (대상 분리 없음, `{{NAV_BACK}}` 빈값).
   - **멀티-팀 · 상급자용** → `team-data/_reports/exec/{team_id}/{YYYY-MM-DD}-exec-report.html` (`exec/{team_id}/`가 없으면 만든다). `{{NAV_BACK}}`에 전사 복귀 링크를 채운다(아래 치환 규칙).
   - **멀티-팀 · 팀장용** → `team-data/_reports/leads/{team_id}-{YYYY-MM-DD}.html` (`leads/`가 없으면 만든다). `{{NAV_BACK}}`는 **빈값**(전사/타팀 링크 없음).
   대상은 호출 인자/요청 표현으로 정한다(아래 "보고 대상 분기"). 같은 날·같은 대상 재실행이면 덮어쓴다.
```

- [ ] **Step 2: "보고 대상 분기" 절을 팀별 리포트 절차 블록쿼트 뒤에 신설**

블록쿼트(`> 팀원이 많아 컨텍스트가 부담되면 …`) 바로 뒤에 새 절을 삽입한다. old→new:

old:
```markdown
> 팀원이 많아 컨텍스트가 부담되면 종합 데이터를 요약해 `general-purpose` 서브 에이전트(`model: "opus"`)에 이 스킬과 함께 넘겨 렌더링을 위임해도 된다. 단독 1~수 명 규모는 인컨텍스트로 처리한다.

## 절차 (조직 요약 — 멀티-팀 "조직 전체/전사 보고")
```
new:
```markdown
> 팀원이 많아 컨텍스트가 부담되면 종합 데이터를 요약해 `general-purpose` 서브 에이전트(`model: "opus"`)에 이 스킬과 함께 넘겨 렌더링을 위임해도 된다. 단독 1~수 명 규모는 인컨텍스트로 처리한다.

## 보고 대상 분기 (상급자 / 팀장)

산출물은 로그인 없는 정적 HTML이라, 접근 제한은 **건네는 파일 단위(공유 경계)**로만 이뤄진다. 같은 팀 데이터를 두 대상으로 나눠 저장한다(내용 동일, 네비게이션만 차이).

- **상급자용** — 요청에 "상급자/경영진/전사 보고"가 있거나 **대상 표현이 없으면 기본**. `_reports/exec/{team_id}/...`에 저장하고 `{{NAV_BACK}}`에 `<a class="backlink" href="../{YYYY-MM-DD}-org-summary.html">← 전사 현황으로</a>`를 채운다. 상급자에겐 `exec/` 폴더(또는 ZIP)째 전달 → 전사↔팀 양방향.
- **팀장용** — 요청에 "팀장용/내 팀/팀별 배포용/각 팀장에게 줄". `_reports/leads/{team_id}-{YYYY-MM-DD}.html`에 저장하고 `{{NAV_BACK}}`는 **빈 문자열**로 치환(전사·타팀 링크 일절 없음). 각 팀장에게 본인 파일 하나만 전달.
- 둘 다 언급되면 둘 다 산출. **레거시 단일 팀은 분리하지 않는다**(org-summary가 없어 무의미; 기존 경로 유지, `{{NAV_BACK}}` 빈값).

## 절차 (조직 요약 — 멀티-팀 "조직 전체/전사 보고")
```

- [ ] **Step 3: 조직 요약 절차의 저장 경로·공유 안내(4·5번)를 `exec/` 기준으로 갱신**

old:
```markdown
4. `team-data/_reports/{YYYY-MM-DD}-org-summary.html`로 저장한다. 팀 바로가기는 `./{team_id}/{YYYY-MM-DD}-exec-report.html` **상대경로**로 건다(같은 `_reports/` 폴더 기준).
5. 조직 요약 + 각 팀 리포트 경로를 보고한다. 자체완결이므로 "폴더(또는 ZIP) 단위로 전체 공유, 팀 하나만 따로 공유 가능"을 안내한다.
```
new:
```markdown
4. `team-data/_reports/exec/{YYYY-MM-DD}-org-summary.html`로 저장한다(상급자 묶음 안). 팀 바로가기는 `./{team_id}/{YYYY-MM-DD}-exec-report.html` **상대경로**로 건다(같은 `exec/` 폴더 기준). 이 드릴다운과 각 팀 페이지의 `{{NAV_BACK}}` 복귀 링크가 상급자용 양방향 네비게이션을 이룬다.
5. 조직 요약 + 각 팀 리포트 경로를 보고한다. **상급자에겐 `exec/` 폴더(또는 ZIP)째** 전달(전사↔팀 양방향), **팀장에겐 `leads/`의 본인 파일 하나만** 전달하도록 안내한다.
```

- [ ] **Step 4: 치환 규칙 표에 `{{NAV_BACK}}` 행 추가**

스칼라 토큰 표의 헤더 행 바로 뒤에 추가한다. old→new:

old:
```markdown
| 토큰 | 내용 |
|---|---|
| `{{TEAM_NAME}}` `{{TEAM_LEAD}}` `{{REPORT_DATE}}` | 헤더. REPORT_DATE=세션 날짜 |
```
new:
```markdown
| 토큰 | 내용 |
|---|---|
| `{{NAV_BACK}}` | 상급자용 팀 페이지: `<a class="backlink" href="../{REPORT_DATE}-org-summary.html">← 전사 현황으로</a>`. 팀장용·레거시: 빈 문자열(제거) |
| `{{TEAM_NAME}}` `{{TEAM_LEAD}}` `{{REPORT_DATE}}` | 헤더. REPORT_DATE=세션 날짜 |
```

- [ ] **Step 5: 조직 요약 치환 규칙의 `{{TL_HREF}}` 설명을 `exec/` 기준으로 갱신**

old:
```markdown
- `teamlink`(드릴다운): `{{TL_NAME}}` `{{TL_LEAD}}` `{{TL_SUMMARY}}`(한 줄 요약, 예 "가동 75% · 인원 4") `{{TL_HREF}}`(**상대경로** `./{team-id}/{YYYY-MM-DD}-exec-report.html`). 각 팀 리포트가 같은 `_reports/` 폴더 하위에 있어야 링크가 열린다.
```
new:
```markdown
- `teamlink`(드릴다운): `{{TL_NAME}}` `{{TL_LEAD}}` `{{TL_SUMMARY}}`(한 줄 요약, 예 "가동 75% · 인원 4") `{{TL_HREF}}`(**상대경로** `./{team-id}/{YYYY-MM-DD}-exec-report.html`). 각 팀 리포트가 같은 `exec/` 폴더 하위(`exec/{team-id}/...`)에 있어야 링크가 열린다(조직 요약도 `exec/`에 있음).
```

- [ ] **Step 6: 검증 (grep)**

Run:
```bash
grep -nE "_reports/exec/|_reports/leads/|보고 대상 분기|\{\{NAV_BACK\}\}" .claude/skills/exec-report-web/SKILL.md
```
Expected: `_reports/exec/`(팀별·조직 요약·치환규칙), `_reports/leads/`, `보고 대상 분기`, `{{NAV_BACK}}` 각각 ≥1건 매칭.

- [ ] **Step 7: 커밋**

```bash
git add .claude/skills/exec-report-web/SKILL.md
git commit -m "feat: exec-report-web에 상급자/팀장 대상 분기·폴더 2벌 경로·{{NAV_BACK}} 치환 규칙

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 3: `team-management/SKILL.md` — Phase5 대상 분기 + 조직 요약 경로

**Files:**
- Modify: `.claude/skills/team-management/SKILL.md`

**Interfaces:**
- Consumes: Task 2의 경로 규약·대상 분기.
- Produces: 오케스트레이터 리포트 흐름의 대상 분기 안내. 사용자 대면 동작.

- [ ] **Step 1: 팬아웃/팬인 5번(상급자 보고용 HTML 산출)의 경로 분기 갱신**

old:
```markdown
5. **상급자 보고용 HTML 산출** (팀 현황 종합 리포트 / 상급자·경영진 보고 / 보고용 웹페이지 요청일 때): 위에서 종합·교차검증한 **동일 데이터**로 `exec-report-web` 스킬을 적용해 자체완결 HTML을 생성한다(전문가를 다시 호출하지 말고 이미 받은 결과를 재사용). 출력 경로는 팀 스코프에 따른다:
   - **레거시 단일 팀** → `team-data/_reports/{날짜}-exec-report.html` (기존).
   - **멀티-팀 단일 팀** → `team-data/_reports/{team_id}/{날짜}-exec-report.html`.
   마크다운 종합과 HTML 파일 경로를 함께 보고하고, "브라우저로 열거나 Ctrl+P로 PDF 저장" 안내를 덧붙인다. 마크다운 종합이 크면 `team-data/_reports/[{team_id}/]{날짜}-team-report.md`에도 저장한다.
```
new:
```markdown
5. **보고용 HTML 산출** (팀 현황 종합 리포트 / 상급자·경영진 보고 / 보고용 웹페이지 / 팀장용 리포트 요청일 때): 위에서 종합·교차검증한 **동일 데이터**로 `exec-report-web` 스킬을 적용해 자체완결 HTML을 생성한다(전문가를 다시 호출하지 말고 이미 받은 결과를 재사용). 출력 경로는 팀 스코프 + **보고 대상**에 따른다(상세 규칙은 `exec-report-web`의 "보고 대상 분기"):
   - **레거시 단일 팀** → `team-data/_reports/{날짜}-exec-report.html` (대상 분리 없음).
   - **멀티-팀 · 상급자용**(요청에 "상급자/경영진/전사" 또는 대상 표현 없음=기본) → `team-data/_reports/exec/{team_id}/{날짜}-exec-report.html` (전사 복귀 링크 포함).
   - **멀티-팀 · 팀장용**(요청에 "팀장용/내 팀/팀별 배포용") → `team-data/_reports/leads/{team_id}-{날짜}.html` (전사/타팀 링크 없음).
   마크다운 종합과 HTML 파일 경로를 함께 보고하고, "브라우저로 열거나 Ctrl+P로 PDF 저장" 안내를 덧붙인다. 마크다운 종합이 크면 `team-data/_reports/[exec/{team_id}/]{날짜}-team-report.md`에도 저장한다.
```

- [ ] **Step 2: 조직 요약 흐름(2·3·4번)의 경로·공유 안내 갱신**

old:
```markdown
2. **각 팀별로** 위 팬아웃/팬인(1~5단계)을 수행해 팀별 자체완결 리포트(`_reports/{team_id}/{날짜}-exec-report.html`)를 생성한다. (팀이 많으면 팀 단위로 순차/병렬 처리)
3. 팀별 종합값(KPI·위험·인원 등)을 모아 `exec-report-web`의 **조직 요약**(`org-template.html`)으로 통합 요약을 `team-data/_reports/{날짜}-org-summary.html`에 생성한다. 조직 KPI는 팀별 값의 집계(인원 합·가동률/연차 평균·자격증 합·1:1 due 합)이고, 각 팀 리포트로 상대경로 링크를 건다.
4. 조직 요약 경로 + 각 팀 리포트 경로를 함께 보고한다("폴더 단위로 공유하거나 각 팀 파일만 따로 공유 가능").
```
new:
```markdown
2. **각 팀별로** 위 팬아웃/팬인(1~5단계)을 상급자용으로 수행해 팀별 자체완결 리포트(`_reports/exec/{team_id}/{날짜}-exec-report.html`, 전사 복귀 링크 포함)를 생성한다. (팀이 많으면 팀 단위로 순차/병렬 처리)
3. 팀별 종합값(KPI·위험·인원 등)을 모아 `exec-report-web`의 **조직 요약**(`org-template.html`)으로 통합 요약을 `team-data/_reports/exec/{날짜}-org-summary.html`에 생성한다. 조직 KPI는 팀별 값의 집계(인원 합·가동률/연차 평균·자격증 합·1:1 due 합)이고, 각 팀 리포트로 상대경로 링크를 건다.
4. 조직 요약 경로 + 각 팀 리포트 경로를 함께 보고한다. **상급자에겐 `exec/` 폴더(또는 ZIP)째** 전달(전사↔팀 양방향), 팀장에게 줄 거면 "팀장용"으로 따로 요청 시 `leads/` 단독 파일을 산출한다고 안내한다.
```

- [ ] **Step 3: 검증 (grep)**

Run:
```bash
grep -nE "_reports/exec/|_reports/leads/|팀장용" .claude/skills/team-management/SKILL.md
```
Expected: `_reports/exec/`(5번·조직 2·3), `_reports/leads/`(5번), `팀장용` 각각 ≥1건.

- [ ] **Step 4: 커밋**

```bash
git add .claude/skills/team-management/SKILL.md
git commit -m "feat: team-management 리포트 흐름에 상급자/팀장 대상 분기·exec 폴더 경로 반영

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 4: 2026-06-18 산출물 재생성 (`exec/` + `leads/`) · 구버전 정리

**Files:**
- Create: `team-data/_reports/exec/2026-06-18-org-summary.html`
- Create: `team-data/_reports/exec/ds/2026-06-18-exec-report.html`
- Create: `team-data/_reports/exec/ai/2026-06-18-exec-report.html`
- Create: `team-data/_reports/leads/ds-2026-06-18.html`
- Create: `team-data/_reports/leads/ai-2026-06-18.html`
- Delete: `team-data/_reports/2026-06-18-org-summary.html`, `team-data/_reports/ds/2026-06-18-exec-report.html`, `team-data/_reports/ai/2026-06-18-exec-report.html` (및 빈 `ds/`·`ai/` 디렉터리)

**Interfaces:**
- Consumes: Task 1의 `.backlink`/`{{NAV_BACK}}` 설계. 현행 생성물의 데이터(이미 합성됨).
- 변환 원리: 현행 팀 리포트 본문 = 팀장용 본문과 동일. 상급자용 팀 페이지 = 현행 팀 리포트 + back 링크 + `.backlink` CSS. org-summary는 내용 불변(드릴다운 `./ds/...`·`./ai/...`가 `exec/` 안에서 그대로 동작).

- [ ] **Step 1: 새 폴더 생성 + 팀장용 파일(현행 팀 리포트 그대로 복사)**

Run:
```bash
cd "C:/Users/user/Desktop/Claude/Project/meta_harness/team-data/_reports"
mkdir -p exec/ds exec/ai leads
cp ds/2026-06-18-exec-report.html leads/ds-2026-06-18.html
cp ai/2026-06-18-exec-report.html leads/ai-2026-06-18.html
```
Expected: 에러 없음. `leads/ds-2026-06-18.html`·`leads/ai-2026-06-18.html` 생성(현행 팀 리포트와 동일 = 팀장용, back/전사 링크 없음).

- [ ] **Step 2: 상급자용 팀 페이지로 이동 + org-summary 이동**

Run:
```bash
cd "C:/Users/user/Desktop/Claude/Project/meta_harness/team-data/_reports"
git mv ds/2026-06-18-exec-report.html exec/ds/2026-06-18-exec-report.html
git mv ai/2026-06-18-exec-report.html exec/ai/2026-06-18-exec-report.html
git mv 2026-06-18-org-summary.html exec/2026-06-18-org-summary.html
```
Expected: 3개 파일이 `exec/` 하위로 이동. 빈 `ds/`·`ai/` 디렉터리는 git에선 자동 사라짐(미추적 잔존 시 Step 6에서 확인).

- [ ] **Step 3: `exec/ds/2026-06-18-exec-report.html`에 `.backlink` CSS 주입**

생성물은 CSS 주석이 제거된 상태이므로 `footer.report{` 규칙을 앵커로 삼는다. old→new (해당 파일 `<style>` 내):

old:
```css
  footer.report{margin-top:22px;
```
new:
```css
  a.backlink{display:inline-flex; align-items:center; gap:6px; margin-bottom:14px; padding:7px 13px; text-decoration:none; background:var(--card); border:1px solid var(--line); border-radius:999px; color:var(--accent); font-size:12.5px; font-weight:650; box-shadow:var(--shadow); transition:border-color .15s;}
  a.backlink:hover{border-color:var(--accent);}
  footer.report{margin-top:22px;
```

> 주의: 위 old `footer.report{margin-top:22px;`가 정확히 매칭되지 않으면, 파일을 Read해 `footer.report{`로 시작하는 실제 줄을 확인하고 그 줄 앞에 두 규칙을 삽입한다.

- [ ] **Step 4: `exec/ds/...`의 `@media print`에 숨김 + back 링크 요소 삽입**

print 숨김 (old→new):

old:
```css
@media print{ body{background:#fff;}
```
new:
```css
@media print{ body{background:#fff;} a.backlink{display:none;}
```

> 주의: 생성물의 print 블록은 한 줄로 압축돼 있을 수 있다. Read로 `@media print{` 실제 형태를 확인해 `body{background:#fff;}` 직후에 `a.backlink{display:none;}`를 넣는다.

back 링크 요소 삽입 (old→new):

old:
```html
<div class="page">

  <header class="report">
```
new:
```html
<div class="page">

  <a class="backlink" href="../2026-06-18-org-summary.html">← 전사 현황으로</a>
  <header class="report">
```

- [ ] **Step 5: `exec/ai/2026-06-18-exec-report.html`에 동일 주입(Step 3·4 반복)**

Step 3의 CSS 주입, Step 4의 print 숨김 + back 링크 요소 삽입을 `exec/ai/2026-06-18-exec-report.html`에 동일하게 적용한다. back 링크 href도 동일하게 `../2026-06-18-org-summary.html`(같은 `exec/` 기준).

- [ ] **Step 6: 검증 (grep + 디렉터리)**

Run:
```bash
cd "C:/Users/user/Desktop/Claude/Project/meta_harness/team-data/_reports"
ls exec exec/ds exec/ai leads
echo "--- exec 팀 페이지 back 링크(각 1):"
grep -c 'class="backlink" href="../2026-06-18-org-summary.html"' exec/ds/2026-06-18-exec-report.html exec/ai/2026-06-18-exec-report.html
echo "--- exec 팀 페이지 .backlink CSS(각 ≥2):"
grep -c "a.backlink" exec/ds/2026-06-18-exec-report.html exec/ai/2026-06-18-exec-report.html
echo "--- leads 파일에 전사/back 링크 없음(각 0):"
grep -cE "backlink|org-summary|전사 현황으로" leads/ds-2026-06-18.html leads/ai-2026-06-18.html
echo "--- org-summary 드릴다운 대상 존재:"
grep -o './ds/2026-06-18-exec-report.html\|./ai/2026-06-18-exec-report.html' exec/2026-06-18-org-summary.html
echo "--- 외부 의존성 0:"
grep -clE "https?://|<script src" exec/2026-06-18-org-summary.html exec/ds/2026-06-18-exec-report.html exec/ai/2026-06-18-exec-report.html leads/ds-2026-06-18.html leads/ai-2026-06-18.html
echo "--- 구버전 경로 제거 확인(없어야 함):"
ls 2026-06-18-org-summary.html ds/2026-06-18-exec-report.html ai/2026-06-18-exec-report.html 2>/dev/null || echo "OLD-PATHS-GONE-OK"
```
Expected: exec 팀 페이지 back 링크 각 `1`, `.backlink` CSS 각 ≥`2`, leads 파일 전사/back 매칭 각 `0`, org-summary 드릴다운 2개 출력, 외부 의존성 모두 `0`, 마지막 `OLD-PATHS-GONE-OK`.

- [ ] **Step 7: 육안 확인 (브라우저)**

`exec/2026-06-18-org-summary.html`을 브라우저로 연다 → "팀별 상세 리포트"에서 DS팀 클릭 → `exec/ds/...` 열림 → 상단 "← 전사 현황으로" 클릭 → org-summary 복귀(왕복 동작). `leads/ds-2026-06-18.html`을 열어 상단에 back 링크가 **없음** 확인.

- [ ] **Step 8: 커밋**

```bash
cd "C:/Users/user/Desktop/Claude/Project/meta_harness"
git add team-data/_reports
git commit -m "feat: 2026-06-18 보고 산출물을 상급자(exec/)·팀장(leads/) 2벌로 재생성

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

### Task 5: `CLAUDE.md` 변경 이력 + 최종 E2E 검증

**Files:**
- Modify: `meta_harness/CLAUDE.md`

**Interfaces:**
- Consumes: Task 1~4 결과.
- Produces: 변경 이력 1행(하네스 문서 동기화).

- [ ] **Step 1: 변경 이력 표 마지막 행 뒤에 신규 행 추가**

`meta_harness/CLAUDE.md`의 변경 이력 표에서 마지막 행(2026-06-18 트랙 B 행) 뒤에 추가한다. old→new:

old:
```markdown
| 2026-06-18 | 트랙 B — **멀티-팀 지원**:
```
> (위 줄로 시작하는 마지막 행 **뒤**에 아래 줄을 삽입. 마지막 행 전체를 그대로 두고 그 다음 줄에 신규 행을 붙인다.)

추가할 행:
```markdown
| 2026-06-18 | 상급자용/팀장용 보고 분리: 산출물 폴더 2벌(`_reports/exec/`=org-summary+팀 페이지(전사 복귀 링크 `{{NAV_BACK}}`), `_reports/leads/`=팀장 단독 파일(전사/타팀 링크 없음)), `template.html` `.backlink`(print 숨김), 요청 표현으로 audience 분기(기본 상급자)·멀티-팀 모드 전용·레거시 하위호환. 2026-06-18 세트 재생성 | skills/exec-report-web(template.html·SKILL.md)·team-management | 상급자는 전사↔팀 양방향 필요, 팀장은 자기 팀만 접근 |
```

- [ ] **Step 2: 최종 E2E 검증 (grep 스윕)**

Run:
```bash
cd "C:/Users/user/Desktop/Claude/Project/meta_harness"
echo "--- 스킬/템플릿 일관성:"
grep -c "{{NAV_BACK}}" .claude/skills/exec-report-web/template.html .claude/skills/exec-report-web/SKILL.md
echo "--- 산출물 구조:"
ls team-data/_reports/exec team-data/_reports/leads
echo "--- 전 산출물 외부 의존성 0:"
grep -rlE "https?://|<script src" team-data/_reports/exec team-data/_reports/leads || echo "ALL-SELF-CONTAINED-OK"
echo "--- 변경 이력 반영:"
grep -c "상급자용/팀장용 보고 분리" meta_harness/CLAUDE.md
```
Expected: `{{NAV_BACK}}` template 1·SKILL ≥1, `exec/`·`leads/` 디렉터리 내용 출력, `ALL-SELF-CONTAINED-OK`, 변경 이력 `1`.

- [ ] **Step 3: 커밋**

```bash
cd "C:/Users/user/Desktop/Claude/Project/meta_harness"
git add meta_harness/CLAUDE.md
git commit -m "docs: CLAUDE.md 변경 이력에 상급자/팀장 보고 분리 추가

Co-Authored-By: Claude Opus 4.8 (1M context) <noreply@anthropic.com>"
```

---

## Self-Review

**Spec coverage (설계 §별 대응):**
- §4 폴더 2벌 구조 → Task 4(산출물), Task 2·3(경로 규약).
- §5 back 링크 사양(위치/문구/경로/print/leads 빈값) → Task 1(템플릿), Task 4(산출물 주입).
- §6 템플릿 `{{NAV_BACK}}`+`.backlink` → Task 1.
- §7 레거시 처리 → Task 2 Step 1·2, Task 3 Step 1(레거시 경로 보존 명문화).
- §8 트리거/라우팅 → Task 2 Step 2, Task 3 Step 1.
- §9 견고성/엣지 → Task 2 "보고 대상 분기"(레거시·둘 다·팀장만), Task 4 Step 6 검증.
- §10 기존 산출물 이행 → Task 4.
- §11 검증 → 각 Task의 grep/육안 Step + Task 5 최종 스윕.
- §12 산출물 → Task 1~5가 1:1 대응.

**Placeholder scan:** "TBD/TODO/적절히/나중에" 없음. 모든 편집 step에 정확한 old→new 또는 명령·기대출력 포함. Task 4 Step 3·4는 생성물 압축 가능성에 대비한 fallback 지시 포함.

**Type/이름 일관성:** 토큰 `{{NAV_BACK}}`, 클래스 `a.backlink`, 문구 `← 전사 현황으로`, 경로 `_reports/exec/{team_id}/`, `_reports/leads/{team_id}-{날짜}.html`, back href `../{날짜}-org-summary.html` — 전 Task에서 동일하게 사용.

## 비범위 (YAGNI)
실제 인증/권한·비밀번호, 개인용(팀원) 3번째 대상, 3단계+ 조직 롤업, 상급자/팀장 내용 차등 제외. 코드 산출물 없음(Claude 네이티브 툴).
