# 상급자 보고 HTML 리포트 디자인 개편 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `exec-report-web` 스킬의 `template.html`을 모던 SaaS 디자인으로 재정의하고, SKILL.md 치환 계약을 맞춘 뒤 보고서를 재생성한다.

> **갱신(2026-06-18):** 멀티-팀원 가독성 항목을 본 계획에 흡수해 함께 실행함. Task 1(템플릿)에 **sticky `<thead>` 헤더**, **주의 팀원 행 강조(`tr.attn` + `{{M_ATTN_CLASS}}`)**, **위험 무음 절단 방지(`{{RISK_MORE_NOTE}}` "외 N건")** 추가. Task 2(SKILL)에 **팀원 행 심각도순 정렬**, **위험 노출 개수/절단 규칙**, **KPI note 편차·이상치 병기** 규칙 추가. 상세 설계는 `docs/superpowers/specs/2026-06-18-multiteam-report-design.md` §4 참조. 멀티-팀(트랙 B)은 동일 설계서 §5~7 + 별도 계획에서 진행.

**Architecture:** 코드 산출물 없는 Claude 네이티브 툴. `template.html`은 `{{TOKEN}}`/`REPEAT` 마커를 가진 **파일 산출물**이고, 스킬 지시문이 이를 채워 `team-data/_reports/`에 저장한다. 이번 작업은 **표현 계층(`<style>`+마크업)만** 교체하고 정보구조·KPI 산식·토큰 계약은 보존한다(단, 위험 아이콘→배지 1건 변경).

**Tech Stack:** 단일 자체완결 HTML5 + 인라인 CSS(외부 의존성 0). 테스트 러너 없음 → 검증은 토큰 잔여 grep + 브라우저 육안 확인.

## Global Constraints

- 자체완결: 외부 CDN·웹폰트·`<script src>`·이미지 URL 금지. 인라인 `<style>`만, 차트는 CSS 막대/CSS 점만.
- 토큰/REPEAT 계약 보존: `<!-- REPEAT_START:risk -->`·`<!-- REPEAT_START:member -->` 마커와 기존 토큰명 유지. 변경 허용은 **`{{RISK_ICON}}` 제거 + `{{RISK_PILL}}` 신규**뿐.
- 디자인 토큰(verbatim): `--ink:#0f172a --muted:#64748b --line:#e7ebf0 --bg:#eef1f6 --card:#fff --accent:#4f46e5 --ok:#16a34a --warn:#d97706 --bad:#dc2626 --info:#2563eb --ok-bg:#f0fdf4 --warn-bg:#fffbeb --bad-bg:#fef2f2 --info-bg:#eff6ff`.
- 폰트 스택: `"Segoe UI","Malgun Gothic","맑은 고딕",-apple-system,Roboto,sans-serif`.
- 위험 심각도 클래스명 유지: `sev-high`/`sev-mid`/`sev-low`. 배지 텍스트 매핑: sev-high=`즉시 조치`, sev-mid=`주의`, sev-low=`참고`.
- 빈/단일 데이터 견고성: 팀원 1명·투입 종료·1:1 없음에서도 깨지지 않음. `0%`/`null`은 `0`/`—`.

---

## File Structure

- `.claude/skills/exec-report-web/template.html` — **Modify(전면 교체)**: `<style>` 블록 + body 마크업. 토큰/REPEAT 마커 보존.
- `.claude/skills/exec-report-web/SKILL.md` — **Modify**: 위험 토큰 계약(RISK_ICON 제거·RISK_PILL 추가), 심각도 매핑 문구, KPI 색·헤더 키커 주석.
- `CLAUDE.md` (meta_harness 루트) — **Modify**: 변경 이력 표에 디자인 개편 행 추가.
- `team-data/_reports/2026-06-17-exec-report.html` — **Regenerate(overwrite)**: 새 템플릿으로 재생성(gitignore 대상, 출력물).

---

## Task 1: template.html 전면 교체 (디자인 적용)

**Files:**
- Modify: `.claude/skills/exec-report-web/template.html` (전체)

**Interfaces:**
- Produces(토큰 계약, 후속 Task·스킬이 의존): 스칼라 `{{TEAM_NAME}}`,`{{TEAM_LEAD}}`,`{{REPORT_DATE}}`,`{{KPI_HEADCOUNT_VALUE}}`,`{{KPI_HEADCOUNT_NOTE}}`,`{{KPI_UTIL_VALUE}}`,`{{KPI_UTIL_NOTE}}`,`{{KPI_UTIL_STATUS}}`,`{{KPI_LEAVE_VALUE}}`,`{{KPI_LEAVE_NOTE}}`,`{{KPI_LEAVE_STATUS}}`,`{{KPI_CERT_VALUE}}`,`{{KPI_CERT_NOTE}}`,`{{KPI_1ON1_VALUE}}`,`{{KPI_1ON1_NOTE}}`,`{{KPI_1ON1_STATUS}}`,`{{GENERATED_AT}}`. REPEAT:risk `{{RISK_SEV_CLASS}}`,`{{RISK_TITLE}}`,`{{RISK_DETAIL}}`,`{{RISK_PILL}}`(신규, RISK_ICON 제거). REPEAT:member `{{M_NAME}}`,`{{M_ROLE}}`,`{{M_UTIL_PCT}}`,`{{M_UTIL_LABEL}}`,`{{M_UTIL_BARCLASS}}`,`{{M_LEAVE_PCT}}`,`{{M_LEAVE_LABEL}}`,`{{M_LEAVE_BARCLASS}}`,`{{M_CERT}}`,`{{M_LASTONE}}`,`{{M_GROWTH}}`. 리스트 `{{AREA_ASSIGN_LIST}}`,`{{AREA_LEAVE_LIST}}`,`{{AREA_CERT_LIST}}`,`{{AREA_GROWTH_LIST}}`,`{{AREA_1ON1_LIST}}`.

- [ ] **Step 1: 새 template.html 전체를 아래 내용으로 작성(덮어쓰기)**

```html
<!DOCTYPE html>
<!--
  이 파일은 템플릿이다. exec-report-web 스킬이 {{토큰}}을 실제 값으로 치환하고
  <!-- REPEAT_START:x --> ~ <!-- REPEAT_END:x --> 블록을 항목 수만큼 복제한 뒤
  team-data/_reports/{YYYY-MM-DD}-exec-report.html 로 저장한다.
  자체완결 규칙: 외부 CDN/스크립트/폰트/이미지 URL을 추가하지 마라. 모든 스타일은 인라인 <style>,
  차트는 순수 CSS 막대(div width %)·CSS 점만 사용한다(오프라인·사내망에서 그대로 열린다).
-->
<html lang="ko">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>{{TEAM_NAME}} 팀 현황 보고 — {{REPORT_DATE}}</title>
<style>
  :root{
    --ink:#0f172a; --muted:#64748b; --line:#e7ebf0; --bg:#eef1f6; --card:#ffffff;
    --accent:#4f46e5; --ok:#16a34a; --warn:#d97706; --bad:#dc2626; --info:#2563eb;
    --ok-bg:#f0fdf4; --warn-bg:#fffbeb; --bad-bg:#fef2f2; --info-bg:#eff6ff;
    --shadow:0 1px 2px rgba(15,23,42,.05),0 4px 12px rgba(15,23,42,.05);
  }
  *{box-sizing:border-box;}
  body{
    margin:0; background:var(--bg); color:var(--ink);
    font-family:"Segoe UI","Malgun Gothic","맑은 고딕",-apple-system,Roboto,sans-serif;
    line-height:1.55; font-size:14px;
  }
  .page{max-width:900px; margin:26px auto; padding:0 16px;}
  /* 헤더 (라이트 카드 + 좌측 액센트 바 + 키커) */
  header.report{
    position:relative; overflow:hidden;
    background:var(--card); border:1px solid var(--line); border-radius:14px;
    padding:20px 24px; display:flex; justify-content:space-between; align-items:flex-end; gap:16px;
    box-shadow:var(--shadow);
  }
  header.report::before{content:""; position:absolute; left:0; top:0; bottom:0; width:5px; background:var(--accent);}
  header.report .kicker{font-size:11px; font-weight:700; letter-spacing:.06em; color:var(--accent); text-transform:uppercase;}
  header.report h1{margin:4px 0; font-size:22px; letter-spacing:-.02em;}
  header.report .sub{color:var(--muted); font-size:12.5px;}
  .badge{
    background:#f1f5f9; border:1px solid var(--line); color:var(--muted);
    padding:5px 12px; border-radius:999px; font-size:11.5px; white-space:nowrap;
  }
  /* 섹션 공통 */
  section{margin-top:22px;}
  h2.sec{font-size:14px; margin:0 0 12px; display:flex; align-items:center; gap:8px; letter-spacing:-.01em;}
  h2.sec::before{content:""; width:4px; height:15px; background:var(--accent); border-radius:2px;}
  /* KPI 카드 (상태색 배경 + 컬러 숫자) */
  .kpi-row{display:grid; grid-template-columns:repeat(5,1fr); gap:12px;}
  .kpi{background:var(--card); border:1px solid var(--line); border-radius:14px; padding:15px 16px;}
  .kpi .label{font-size:11px; color:var(--muted); text-transform:uppercase; letter-spacing:.03em;}
  .kpi .value{font-size:27px; font-weight:760; letter-spacing:-.02em; margin-top:7px;}
  .kpi .note{font-size:11.5px; color:var(--muted); margin-top:3px;}
  .kpi.ok{background:var(--ok-bg); border-color:#bbf7d0;} .kpi.ok .value{color:var(--ok);}
  .kpi.warn{background:var(--warn-bg); border-color:#fde68a;} .kpi.warn .value{color:var(--warn);}
  .kpi.bad{background:var(--bad-bg); border-color:#fecaca;} .kpi.bad .value{color:var(--bad);}
  /* 위험 신호 (좌측 심각도 바 + CSS 색 점 + 배지) */
  .risk{display:flex; gap:12px; align-items:flex-start; background:var(--card);
        border:1px solid var(--line); border-left-width:4px; border-radius:10px; padding:13px 14px; margin-bottom:9px;}
  .risk .ico{flex:0 0 auto; width:22px; height:22px; display:flex; align-items:center; justify-content:center;}
  .risk .dot{width:10px; height:10px; border-radius:50%;}
  .risk .body{flex:1; min-width:0;}
  .risk .title{font-weight:650; font-size:14px;}
  .risk .detail{font-size:12.5px; color:var(--muted); margin-top:2px;}
  .risk .pill{flex:0 0 auto; align-self:center; font-size:10.5px; font-weight:700; padding:3px 9px; border-radius:999px; white-space:nowrap;}
  .risk.sev-high{border-left-color:var(--bad);}
  .risk.sev-high .dot{background:var(--bad); box-shadow:0 0 0 4px rgba(220,38,38,.14);}
  .risk.sev-high .pill{background:var(--bad-bg); color:var(--bad);}
  .risk.sev-mid{border-left-color:var(--warn);}
  .risk.sev-mid .dot{background:var(--warn); box-shadow:0 0 0 4px rgba(217,119,6,.14);}
  .risk.sev-mid .pill{background:var(--warn-bg); color:var(--warn);}
  .risk.sev-low{border-left-color:var(--info);}
  .risk.sev-low .dot{background:var(--info); box-shadow:0 0 0 4px rgba(37,99,235,.14);}
  .risk.sev-low .pill{background:var(--info-bg); color:var(--info);}
  .empty{color:var(--muted); font-size:13px; padding:10px 0;}
  /* 팀원 현황 표 (정돈된 표) */
  .table-wrap{overflow-x:auto;}
  table{border-collapse:separate; border-spacing:0; width:100%; min-width:680px; background:var(--card);
        border:1px solid var(--line); border-radius:12px; overflow:hidden; font-size:13px;}
  th{background:#f8fafc; color:var(--muted); font-weight:600; font-size:11px; text-transform:uppercase; letter-spacing:.03em; text-align:left; padding:11px 14px; border-bottom:1px solid var(--line);}
  td{padding:13px 14px; border-bottom:1px solid #f1f5f9; vertical-align:middle;}
  tr:last-child td{border-bottom:none;}
  tbody tr:hover{background:#fafbff;}
  .nm{font-weight:680; font-size:13.5px;}
  .bar{position:relative; height:7px; background:#e7ebf0; border-radius:999px; overflow:hidden; min-width:80px;}
  .bar > span{position:absolute; left:0; top:0; bottom:0; border-radius:999px; background:var(--accent);}
  .bar.ok > span{background:var(--ok);} .bar.warn > span{background:var(--warn);} .bar.bad > span{background:var(--bad);}
  .barlabel{font-size:11px; color:var(--muted); margin-top:4px; display:block;}
  /* 영역별 요약 (카드 + 아이콘 칩 + CSS 점 불릿) */
  .areas{display:grid; grid-template-columns:repeat(2,1fr); gap:14px;}
  .area{background:var(--card); border:1px solid var(--line); border-radius:14px; padding:16px 18px; box-shadow:var(--shadow);}
  .area.wide{grid-column:1 / -1;}
  .area h3{margin:0 0 10px; font-size:13.5px; display:flex; align-items:center; gap:8px;}
  .area .chip{width:26px; height:26px; border-radius:8px; display:flex; align-items:center; justify-content:center; font-size:14px; background:#eef2ff;}
  .area ul{margin:0; padding:0; list-style:none;}
  .area li{font-size:12.5px; color:#334155; padding:4px 0 4px 16px; position:relative;}
  .area li::before{content:""; position:absolute; left:0; top:10px; width:5px; height:5px; border-radius:50%; background:var(--accent);}
  .area li.empty{padding-left:0; color:var(--muted);} .area li.empty::before{display:none;}
  /* 푸터 */
  footer.report{margin-top:22px; padding-top:14px; border-top:1px solid var(--line); color:var(--muted); font-size:11.5px; display:flex; justify-content:space-between; gap:12px; flex-wrap:wrap;}
  /* 반응형 */
  @media (max-width:760px){
    .kpi-row{grid-template-columns:repeat(2,1fr);} .areas{grid-template-columns:1fr;}
    .area.wide{grid-column:auto;}
    header.report{flex-direction:column; align-items:flex-start;}
  }
  /* 인쇄(PDF) */
  @media print{
    body{background:#fff;} .page{margin:0; max-width:none;}
    header.report,.kpi,.risk,.area,table{box-shadow:none;}
    section,.area,.risk,tr{break-inside:avoid;}
    .kpi,.risk,.area,.bar > span,.risk .dot,header.report::before,h2.sec::before{
      -webkit-print-color-adjust:exact; print-color-adjust:exact;
    }
    @page{margin:14mm;}
  }
</style>
</head>
<body>
<div class="page">

  <header class="report">
    <div>
      <div class="kicker">팀 현황 보고</div>
      <h1>{{TEAM_NAME}}</h1>
      <div class="sub">팀장 {{TEAM_LEAD}} · 보고 기준일 {{REPORT_DATE}}</div>
    </div>
    <span class="badge">🔒 내부 보고용</span>
  </header>

  <!-- ① 핵심 KPI -->
  <section>
    <h2 class="sec">핵심 지표</h2>
    <div class="kpi-row">
      <div class="kpi">
        <div class="label">팀 인원 (active)</div>
        <div class="value">{{KPI_HEADCOUNT_VALUE}}</div>
        <div class="note">{{KPI_HEADCOUNT_NOTE}}</div>
      </div>
      <div class="kpi {{KPI_UTIL_STATUS}}">
        <div class="label">가동률 (YTD)</div>
        <div class="value">{{KPI_UTIL_VALUE}}</div>
        <div class="note">{{KPI_UTIL_NOTE}}</div>
      </div>
      <div class="kpi {{KPI_LEAVE_STATUS}}">
        <div class="label">평균 연차 누적률</div>
        <div class="value">{{KPI_LEAVE_VALUE}}</div>
        <div class="note">{{KPI_LEAVE_NOTE}}</div>
      </div>
      <div class="kpi">
        <div class="label">자격증 (보유/목표)</div>
        <div class="value">{{KPI_CERT_VALUE}}</div>
        <div class="note">{{KPI_CERT_NOTE}}</div>
      </div>
      <div class="kpi {{KPI_1ON1_STATUS}}">
        <div class="label">1:1 주기</div>
        <div class="value">{{KPI_1ON1_VALUE}}</div>
        <div class="note">{{KPI_1ON1_NOTE}}</div>
      </div>
    </div>
  </section>

  <!-- ② 핵심 위험 신호 (Top 3~5) -->
  <section>
    <h2 class="sec">핵심 위험 신호</h2>
    <!-- 위험이 없으면 아래 REPEAT 블록 전체를 <div class="empty">현재 우선 조치가 필요한 위험 신호가 없습니다.</div> 로 교체 -->
    <!-- REPEAT_START:risk -->
    <div class="risk {{RISK_SEV_CLASS}}">
      <div class="ico"><span class="dot"></span></div>
      <div class="body">
        <div class="title">{{RISK_TITLE}}</div>
        <div class="detail">{{RISK_DETAIL}}</div>
      </div>
      <span class="pill">{{RISK_PILL}}</span>
    </div>
    <!-- REPEAT_END:risk -->
  </section>

  <!-- ③ 팀원 현황 한눈에 -->
  <section>
    <h2 class="sec">팀원 현황</h2>
    <div class="table-wrap">
      <table>
        <thead>
          <tr>
            <th>팀원</th><th>가동률 (YTD)</th><th>연차 (잔여/누적률)</th><th>자격증</th><th>최근 1:1</th><th>역량 평가</th>
          </tr>
        </thead>
        <tbody>
          <!-- REPEAT_START:member -->
          <tr>
            <td><div class="nm">{{M_NAME}}</div><span class="barlabel">{{M_ROLE}}</span></td>
            <td><div class="bar {{M_UTIL_BARCLASS}}"><span style="width:{{M_UTIL_PCT}}%"></span></div><span class="barlabel">{{M_UTIL_LABEL}}</span></td>
            <td><div class="bar {{M_LEAVE_BARCLASS}}"><span style="width:{{M_LEAVE_PCT}}%"></span></div><span class="barlabel">{{M_LEAVE_LABEL}}</span></td>
            <td>{{M_CERT}}</td>
            <td>{{M_LASTONE}}</td>
            <td>{{M_GROWTH}}</td>
          </tr>
          <!-- REPEAT_END:member -->
        </tbody>
      </table>
    </div>
  </section>

  <!-- ④ 영역별 요약 -->
  <section>
    <h2 class="sec">영역별 요약</h2>
    <div class="areas">
      <div class="area"><h3><span class="chip">🗂️</span> 프로젝트 투입 · 가동률</h3><ul>{{AREA_ASSIGN_LIST}}</ul></div>
      <div class="area"><h3><span class="chip">🏖️</span> 휴가 · 연차</h3><ul>{{AREA_LEAVE_LIST}}</ul></div>
      <div class="area"><h3><span class="chip">📜</span> 자격증</h3><ul>{{AREA_CERT_LIST}}</ul></div>
      <div class="area"><h3><span class="chip">📈</span> 역량 (AS-IS⇒TO-BE)</h3><ul>{{AREA_GROWTH_LIST}}</ul></div>
      <div class="area wide"><h3><span class="chip">🤝</span> 1:1 미팅</h3><ul>{{AREA_1ON1_LIST}}</ul></div>
    </div>
  </section>

  <footer class="report">
    <span>생성: {{GENERATED_AT}} · 데이터 출처: team-data/ (파일 기반)</span>
    <span>meta_harness 팀원 관리툴 · 자동 생성 스냅샷</span>
  </footer>

</div>
</body>
</html>
```

- [ ] **Step 2: 토큰/마커 보존 검증 (grep)**

Run:
```bash
cd "C:/Users/user/Desktop/Claude/Project/meta_harness"
grep -c "REPEAT_START:risk\|REPEAT_START:member\|{{RISK_PILL}}\|{{KPI_UTIL_STATUS}}\|{{AREA_1ON1_LIST}}" .claude/skills/exec-report-web/template.html
grep -c "RISK_ICON" .claude/skills/exec-report-web/template.html
```
Expected: 첫 줄 ≥5 (마커·신규/기존 토큰 존재), 둘째 줄 `0` (RISK_ICON 완전 제거).

- [ ] **Step 3: 외부 의존성 0 검증 (grep)**

Run:
```bash
grep -nE "https?://|<script|cdn|fonts.googleapis" .claude/skills/exec-report-web/template.html || echo "CLEAN"
```
Expected: `CLEAN` (외부 URL/스크립트/폰트 없음).

- [ ] **Step 4: 커밋**

```bash
git add .claude/skills/exec-report-web/template.html
git commit -m "feat: exec-report 템플릿 모던 SaaS 디자인 적용"
```

---

## Task 2: SKILL.md 치환 계약 동기화

**Files:**
- Modify: `.claude/skills/exec-report-web/SKILL.md`

**Interfaces:**
- Consumes: Task 1의 토큰 계약(특히 `{{RISK_PILL}}` 신규, `{{RISK_ICON}}` 제거).

- [ ] **Step 1: 심각도 매핑 문구 교체 (line 59 부근)**

기존:
```
심각도 매핑: `sev-high`(🔴 즉시 조치) / `sev-mid`(🟠 주의) / `sev-low`(🔵 참고). 위험이 없으면 REPEAT 블록을 `<div class="empty">현재 우선 조치가 필요한 위험 신호가 없습니다.</div>`로 교체한다.
```
교체:
```
심각도 매핑(클래스→배지 텍스트, 색 점은 클래스로 자동): `sev-high`→배지 `즉시 조치`(적색 점) / `sev-mid`→`주의`(주황 점) / `sev-low`→`참고`(청색 점). 아이콘은 이모지가 아니라 CSS 색 점이라 별도 토큰이 없다. 위험이 없으면 REPEAT 블록을 `<div class="empty">현재 우선 조치가 필요한 위험 신호가 없습니다.</div>`로 교체한다.
```

- [ ] **Step 2: risk 블록 토큰 줄 교체 (line 77 부근)**

기존:
```
- `risk` 블록: `{{RISK_SEV_CLASS}}`(sev-high|sev-mid|sev-low) `{{RISK_ICON}}`(🔴/🟠/🔵) `{{RISK_TITLE}}` `{{RISK_DETAIL}}`
```
교체:
```
- `risk` 블록: `{{RISK_SEV_CLASS}}`(sev-high|sev-mid|sev-low) `{{RISK_TITLE}}` `{{RISK_DETAIL}}` `{{RISK_PILL}}`(심각도 배지 텍스트: sev-high=즉시 조치/sev-mid=주의/sev-low=참고). ※ 색 점은 `{{RISK_SEV_CLASS}}`로 자동 렌더되므로 아이콘 토큰 없음.
```

- [ ] **Step 3: KPI 색·헤더 키커 주석 추가 (상태 클래스 설명 끝, line 48 부근에 한 줄 추가)**

`**휴가비 주의:**` 줄 바로 앞에 추가:
```
**표기 메모:** KPI 카드는 `{{KPI_*_STATUS}}`(ok/warn/bad) 클래스로 배경색과 숫자색이 함께 칠해진다(중립은 빈 값→흰 카드·검정 숫자). 헤더는 고정 키커 "팀 현황 보고" + `{{TEAM_NAME}}`(팀명) 구조다.
```

- [ ] **Step 4: 검증 (grep)**

Run:
```bash
grep -c "RISK_ICON" .claude/skills/exec-report-web/SKILL.md
grep -c "RISK_PILL" .claude/skills/exec-report-web/SKILL.md
```
Expected: 첫 줄 `0`, 둘째 줄 ≥1.

- [ ] **Step 5: 커밋**

```bash
git add .claude/skills/exec-report-web/SKILL.md
git commit -m "docs: exec-report SKILL 치환 계약 동기화 (RISK_PILL, KPI 색)"
```

---

## Task 3: CLAUDE.md 변경 이력 추가

**Files:**
- Modify: `CLAUDE.md` (meta_harness 루트)

- [ ] **Step 1: 변경 이력 표 마지막 행 다음에 추가**

```
| 2026-06-17 | exec 보고 HTML UI/UX 개편(모던 SaaS): 라이트 헤더+액센트, KPI 상태색 카드, 위험 CSS 색점+배지, 정돈 표, 인디고 액센트. RISK_ICON→RISK_PILL 계약 변경 | skills/exec-report-web(template.html·SKILL.md) | 상급자 보고서 가독성·완성도 개선 |
```

- [ ] **Step 2: 커밋**

```bash
git add CLAUDE.md
git commit -m "docs: CLAUDE 변경 이력에 exec-report 디자인 개편 기록"
```

---

## Task 4: 보고서 재생성 + 검증

**Files:**
- Regenerate(overwrite): `team-data/_reports/2026-06-17-exec-report.html` (gitignore 대상)

**Interfaces:**
- Consumes: 새 `template.html`, 기존 `2026-06-17-exec-report.html`의 산출 값(데이터 무변경 → 재계산 대신 동일 값 재사용, 단 `RISK_PILL` 추가).

- [ ] **Step 1: 기존 보고서에서 산출 값 확보**

기존 `team-data/_reports/2026-06-17-exec-report.html`를 읽어 KPI 값/노트/상태, 위험 항목(제목·상세·심각도), 팀원 행 값, 영역 리스트 항목을 추출한다. 데이터는 16:18 이후 변동 없음 → 값 그대로 재사용(표현만 교체). 위험 각 항목에 심각도→배지 텍스트(`즉시 조치`/`주의`/`참고`)를 새로 부여한다.

- [ ] **Step 2: 새 template.html을 채워 동일 경로에 덮어쓰기**

`template.html`을 읽고 치환 규칙대로 토큰·REPEAT를 채운 뒤 `team-data/_reports/2026-06-17-exec-report.html`로 저장(마커 주석 제거). `GENERATED_AT`/`REPORT_DATE`는 세션 날짜(2026-06-17).

- [ ] **Step 3: 토큰 잔여·구조 검증 (grep)**

Run:
```bash
cd "C:/Users/user/Desktop/Claude/Project/meta_harness"
grep -c "{{" team-data/_reports/2026-06-17-exec-report.html
grep -c "REPEAT_START\|RISK_ICON" team-data/_reports/2026-06-17-exec-report.html
grep -c "kicker\|class=\"pill\"\|class=\"dot\"" team-data/_reports/2026-06-17-exec-report.html
```
Expected: 첫 줄 `0`(미치환 토큰 없음), 둘째 줄 `0`(마커·구토큰 제거됨), 셋째 줄 ≥3(새 구조 적용됨).

- [ ] **Step 4: 브라우저 육안 확인**

Run:
```bash
start "" "C:/Users/user/Desktop/Claude/Project/meta_harness/team-data/_reports/2026-06-17-exec-report.html"
```
Expected: 합의한 디자인(라이트 헤더+액센트 바, 상태색 KPI, 좌측 바+색 점 위험 배지, 정돈 표, 아이콘 칩 영역 카드)이 깨짐 없이 렌더. 팀원 1명·1:1 없음에서도 레이아웃 정상, 미치환 `{{...}}` 없음.

- [ ] **Step 5: (출력물은 gitignore → 커밋 없음) 완료 보고**

`team-data/_reports/2026-06-17-exec-report.html` 갱신을 사용자에게 보고하고 "브라우저로 열거나 Ctrl+P로 PDF 저장 가능"을 안내한다.

---

## Self-Review

**1. Spec coverage:**
- 디자인 방향/토큰/컴포넌트 6종 → Task 1 (template) ✓
- 치환 계약 변경(헤더 키커·RISK_ICON 제거·RISK_PILL·KPI 색) → Task 1(템플릿) + Task 2(SKILL.md) ✓
- @media print 라이트 헤더 대응 → Task 1 `<style>` print 블록 ✓
- 반응형(KPI 5→2, areas 2→1) → Task 1 print/미디어쿼리 ✓
- CLAUDE.md 변경 이력 → Task 3 ✓
- 빈/단일 데이터 견고성·재생성 → Task 4 (1명·1:1 없음 확인) ✓
- 갭: 없음.

**2. Placeholder scan:** 모든 코드 블록은 실제 내용. "적절히/TBD" 없음. ✓

**3. Type consistency:** 토큰명이 spec·SKILL.md·template에서 일치(`RISK_SEV_CLASS`,`RISK_PILL`,`M_UTIL_BARCLASS` 등). sev 클래스명 `sev-high/mid/low` 3곳 통일. ✓

---

## Execution Handoff

(상위 세션에서 실행 방식 선택)
