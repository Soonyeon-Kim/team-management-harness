---
name: exec-report-web
description: 팀 현황을 상급자/경영진에게 보고하기 위한 자체완결 HTML 리포트(웹페이지)를 생성하는 스킬. "상급자 보고", "상급자에게 보고할 웹페이지", "경영진 리포트", "보고용 웹페이지", "팀 현황 HTML/웹 리포트", "보고서로 뽑아줘", "PDF로 보고" 같은 요청에 사용한다. team-management 오케스트레이터가 "팀 현황 종합 리포트" 흐름의 마지막 단계로 호출해 마크다운 종합과 함께 HTML을 산출한다. 후속 표현("보고서 다시 만들어", "HTML 업데이트", "리포트 새로 뽑아")도 포함. 단순 휴가/자격증/역량 단일 조회는 각 도메인 스킬을 쓰고 이 스킬은 종합 보고가 필요할 때만 사용한다.
---

# 상급자 보고용 HTML 리포트 생성

팀장이 상급자에게 보고할 수 있는 **경영진 요약 중심의 자체완결 HTML 페이지**를 만든다. 산출물은 외부 의존성이 없는 단일 `.html` 파일이라 더블클릭으로 브라우저에서 열고, `Ctrl+P`로 PDF 출력하고, 메일/메신저로 그대로 공유할 수 있다. 생성 시점의 **스냅샷**이다.

이 하네스는 코드 산출물이 없는 Claude 네이티브 툴이며, HTML은 `checklist.md`·`_reports/*.md`와 같은 **파일 산출물**일 뿐이다(새 서버·빌드·코드베이스 아님). 렌더링은 이 스킬의 지시문 + 번들 템플릿(`template.html`)으로 표현하고, 실행 코드(JS 등)는 만들지 않는다.

## 입력: 데이터는 어디서 오는가

이 스킬은 KPI를 **새로 발명하지 않는다.** 산식·규칙은 `team-data-store`와 각 도메인 스킬의 것을 그대로 따른다.

- **오케스트레이션 호출(기본):** `team-management`가 "팀 현황 종합 리포트" 흐름에서 5개 전문가(certification·leave·growth·oneonone·assignment)를 팬아웃해 받은 결과와 교차검증 결과를 이미 갖고 있다. 그 값을 그대로 채운다(재계산 금지).
- **단독 호출:** 사전 종합 없이 직접 요청받으면, 먼저 `team-management`의 리포트 흐름을 거치는 것이 원칙이다. 부득이 직접 읽어야 하면 `team-data-store/SKILL.md` 스키마에 따라 `team-data/`를 읽어 아래 산식으로 계산한다.

헤더용 메타(`team`, `lead`)는 `{team_root}/team.md`에서, 현재 날짜는 세션 날짜를 쓴다. **팀 루트(`team_root`)는 `team-data-store`의 "멀티-팀 모드" 규칙으로 해석**한다(레거시면 `team-data/`, 멀티-팀이면 `team-data/teams/{team_id}/`; 오케스트레이터가 `team_id`를 전달).

## 절차 (팀별 리포트)

1. `{team_root}/team.md`에서 팀명·팀장을 확보한다.
2. 종합 데이터(전문가 결과 또는 직접 읽은 값)로 아래 **KPI·위험·팀원행·영역요약**을 산출한다.
3. 이 스킬 디렉토리의 `template.html`을 읽는다.
4. 아래 **치환 규칙**대로 토큰·REPEAT 블록을 실제 값으로 채운다. `<style>`과 구조·클래스는 그대로 둔다.
5. 저장 경로(팀 스코프 · 보고 대상):
   - **레거시 단일 팀** → `team-data/_reports/{YYYY-MM-DD}-exec-report.html` (대상 분리 없음, `{{NAV_BACK}}` 빈값).
   - **멀티-팀 · 상급자용** → `team-data/_reports/exec/{team_id}/{YYYY-MM-DD}-exec-report.html` (`exec/{team_id}/`가 없으면 만든다). `{{NAV_BACK}}`에 전사 복귀 링크를 채운다(아래 치환 규칙).
   - **멀티-팀 · 팀장용** → `team-data/_reports/leads/{team_id}-{YYYY-MM-DD}.html` (`leads/`가 없으면 만든다). `{{NAV_BACK}}`는 **빈값**(전사/타팀 링크 없음).
   대상은 호출 인자/요청 표현으로 정한다(아래 "보고 대상 분기"). 같은 날·같은 대상 재실행이면 덮어쓴다.
6. 저장 경로를 보고하고, "브라우저로 열거나 Ctrl+P로 PDF 저장 가능"을 안내한다.

> 팀원이 많아 컨텍스트가 부담되면 종합 데이터를 요약해 `general-purpose` 서브 에이전트(`model: "opus"`)에 이 스킬과 함께 넘겨 렌더링을 위임해도 된다. 단독 1~수 명 규모는 인컨텍스트로 처리한다.

## 보고 대상 분기 (상급자 / 팀장)

산출물은 로그인 없는 정적 HTML이라, 접근 제한은 **건네는 파일 단위(공유 경계)**로만 이뤄진다. 같은 팀 데이터를 두 대상으로 나눠 저장한다(내용 동일, 네비게이션만 차이).

- **상급자용** — 요청에 "상급자/경영진/전사 보고"가 있거나 **대상 표현이 없으면 기본**. `_reports/exec/{team_id}/...`에 저장하고 `{{NAV_BACK}}`에 `<a class="backlink" href="../{YYYY-MM-DD}-org-summary.html">← 전사 현황으로</a>`를 채운다. 상급자에겐 `exec/` 폴더(또는 ZIP)째 전달 → 전사↔팀 양방향.
- **팀장용** — 요청에 "팀장용/내 팀/팀별 배포용/각 팀장에게 줄". `_reports/leads/{team_id}-{YYYY-MM-DD}.html`에 저장하고 `{{NAV_BACK}}`는 **빈 문자열**로 치환(전사·타팀 링크 일절 없음). 각 팀장에게 본인 파일 하나만 전달.
- 둘 다 언급되면 둘 다 산출. **레거시 단일 팀은 분리하지 않는다**(org-summary가 없어 무의미; 기존 경로 유지, `{{NAV_BACK}}` 빈값).

## 절차 (조직 요약 — 멀티-팀 "조직 전체/전사 보고")

여러 동격 팀을 한 장으로 요약하고 각 팀 리포트로 드릴다운하는 흐름. **각 팀 리포트는 위 "팀별 리포트" 절차로 먼저 생성**되어 있어야 한다(오케스트레이터가 팀별 팬아웃 후 호출).

1. `team-data/teams/` 아래 활성 팀 목록과 각 팀의 종합값(팀명·팀장·KPI·위험·인원)을 받는다(재계산 금지, 팀별 리포트 산출값 재사용).
2. 이 스킬 디렉토리의 `org-template.html`을 읽는다.
3. 아래 **조직 요약 치환 규칙**대로 채운다: 조직 KPI = 팀별 값 집계(인원 **합**, 가동률·연차 누적률 **평균**, 자격증 보유/목표 **합**, 1:1 due **합**), 팀 비교 미니표(`REPEAT_START:team`), 조직 Top 위험(팀명 태그·"외 N건"), 팀 바로가기 링크.
4. `team-data/_reports/exec/{YYYY-MM-DD}-org-summary.html`로 저장한다(상급자 묶음 안). 팀 바로가기는 `./{team_id}/{YYYY-MM-DD}-exec-report.html` **상대경로**로 건다(같은 `exec/` 폴더 기준). 이 드릴다운과 각 팀 페이지의 `{{NAV_BACK}}` 복귀 링크가 상급자용 양방향 네비게이션을 이룬다.
5. 조직 요약 + 각 팀 리포트 경로를 보고한다. **상급자에겐 `exec/` 폴더(또는 ZIP)째** 전달(전사↔팀 양방향), **팀장에겐 `leads/`의 본인 파일 하나만** 전달하도록 안내한다.

## KPI 산식 (출처: team-data-store + 도메인 스킬)

| KPI | 산식 | 출처 |
|---|---|---|
| 팀 인원(active) | `members/` 중 `status == active` 수 (전체 인원 대비 표기) | members |
| 가동률 (YTD) | 팀원별 **(기간 가동일 ÷ 가용일 × 100)**의 팀 평균. 가동일=고객 대상 투입(프로젝트·교육·행사) 실근무일×allocation(종료 투입도 기간이면 포함), 가용일=영업일−승인 휴가−공휴일(`calendar/holidays.json`) | assignment-tracking + leave + holidays |
| 현재 동시 투입률 | `status==active` 투입 allocation 합(스냅샷). 0%=현재 무투입(스태핑) — 가동률과 구분해 위험/메모로만 다룬다 | assignment-tracking |
| 평균 연차 누적률 | 팀원별 (잔여연차 / 잔여개월 × 100)의 평균. 잔여연차=`granted+carryover-used`, 잔여개월=현재월~연말 | leave-management |
| 자격증(보유/목표) | `status==acquired` 수 / `status==target`(+in-progress) 수 | certification-management |
| 1:1 주기 | cadence 대비 due(기한 초과) 팀원 수 표기(예 "1명 due", 기록 없으면 "기록 없음") | oneonone-workflow |

**상태 클래스**(KPI 카드/막대 색): `ok`(양호) / `warn`(주의) / `bad`(위험), 비우면 기본(중립).
- 가동률(YTD): 적정(약 70~90%) → `ok`, 낮음(<60%, 비가동 과다) → `warn`, 과투입(>100%) → `bad`. ※ 현재 동시 투입 0%(무투입)는 가동률 색이 아니라 별도 위험 신호로 표기(YTD 가동률은 높아도 지금 놀고 있을 수 있음).
- 누적률: 100~150% → `ok`, 150% 초과(미소진) 또는 100% 미만(과소진) → `warn`, 마이너스 연차 발생 → `bad`.
- 1:1: due 없음 → `ok`, due 있음 → `warn`, 장기 미실시(8주+)·기록 없음 → `bad` 또는 중립(판단해 표기).

**표기 메모:** KPI 카드는 `{{KPI_*_STATUS}}`(ok/warn/bad) 클래스로 배경색·숫자색이 함께 칠해진다(중립은 빈값→흰 카드·검정 숫자). 팀원이 여럿이면 평균/합산 KPI의 `_NOTE`에 **편차·이상치**를 병기한다(예: "평균 78% · 최저 42% 박민호"). 헤더는 고정 키커 "팀 현황 보고" + `{{TEAM_NAME}}`(팀명) 구조다.

**휴가비 주의:** `used <= 10`이면 휴가비 미활용을 위험/독려로 띄우지 마라(신청 자격 미달이라 오인 유발). 자격(`used>10`) 충족 시에만 활용도를 다룬다.

## 위험 신호 (Top 3~5)

`team-management` 교차검증 항목을 심각도순으로 최대 5개 고른다. 후보:
- 과투입(YTD>100%) / 현재 무투입(동시 투입 0% — 신규 배치 검토) — assignment
- 자격증 시험일·신청 마감 임박, 유지보수(maintenance_due) 도래
- 휴가-프로젝트(active 투입) 일정 충돌 → 인수인계 필요
- 마이너스 연차 / 신입 연차 소멸 임박(3/31)
- 역량 정체(next_review 경과 / 8주+ 진척 없음) + 1:1 장기 미실시

심각도 매핑(클래스→배지 텍스트, 색 점은 클래스로 자동): `sev-high`→배지 `즉시 조치`(적색 점) / `sev-mid`→`주의`(주황 점) / `sev-low`→`참고`(청색 점). 아이콘은 이모지가 아니라 CSS 색 점이라 별도 토큰이 없다. 위험이 없으면 REPEAT 블록을 `<div class="empty">현재 우선 조치가 필요한 위험 신호가 없습니다.</div>`로 교체한다.

**노출 개수 / 무음 절단 금지:** 기본 Top 3~5를 심각도순으로 노출한다. 팀원이 많아 후보가 많으면 7~8개까지 올려도 된다. 그래도 남으면 **조용히 자르지 말고** `{{RISK_MORE_NOTE}}`에 `외 N건(요약: …)`을 채운다(잘린 게 없으면 빈값). 멀티-팀원 시 위험 제목에 **해당 팀원명을 명시**한다(예: "박민호 — 저가동(42%)").

## 치환 규칙

**스칼라 토큰** `{{TOKEN}}` — 한 값으로 교체. 값은 HTML 이스케이프(`< > &`)한다.

| 토큰 | 내용 |
|---|---|
| `{{NAV_BACK}}` | 상급자용 팀 페이지: `<a class="backlink" href="../{REPORT_DATE}-org-summary.html">← 전사 현황으로</a>`. 팀장용·레거시: 빈 문자열(제거) |
| `{{TEAM_NAME}}` `{{TEAM_LEAD}}` `{{REPORT_DATE}}` | 헤더. REPORT_DATE=세션 날짜 |
| `{{KPI_HEADCOUNT_VALUE}}` `{{KPI_HEADCOUNT_NOTE}}` | 예: "1명" / "전원 active" |
| `{{KPI_UTIL_VALUE}}` `{{KPI_UTIL_NOTE}}` `{{KPI_UTIL_STATUS}}` | YTD 가동률. 예: "75%" / "YTD · 현재 무투입(신규 배치 검토)" / `ok` |
| `{{KPI_LEAVE_VALUE}}` `{{KPI_LEAVE_NOTE}}` `{{KPI_LEAVE_STATUS}}` | 예: "112%" / "적정 범위" / `ok` |
| `{{KPI_CERT_VALUE}}` `{{KPI_CERT_NOTE}}` | 예: "4 / 1" / "목표 1건 추진 중" |
| `{{KPI_1ON1_VALUE}}` `{{KPI_1ON1_NOTE}}` `{{KPI_1ON1_STATUS}}` | 예: "기록 없음" / "1:1 데이터 미입력" / `bad` |
| `{{GENERATED_AT}}` | 생성 일시(세션 날짜, 분 단위는 생략 가능) |

**REPEAT 블록** `<!-- REPEAT_START:x -->` ~ `<!-- REPEAT_END:x -->` — 내부를 항목 수만큼 복제하고 각 복제본의 토큰을 채운 뒤 **마커 주석은 제거**한다. 항목이 없으면 위 "empty" 처리.

- `risk` 블록: `{{RISK_SEV_CLASS}}`(sev-high|sev-mid|sev-low) `{{RISK_TITLE}}` `{{RISK_DETAIL}}` `{{RISK_PILL}}`(배지 텍스트: sev-high=즉시 조치/sev-mid=주의/sev-low=참고). ※ 색 점은 `{{RISK_SEV_CLASS}}`로 자동 렌더되므로 아이콘 토큰 없음.
- `{{RISK_MORE_NOTE}}`(스칼라, REPEAT 블록 밖): 노출 한도를 넘긴 위험 요약 "외 N건…". 잘린 게 없으면 빈값.
- `member` 블록(팀원 1행): **주의 심각도순으로 정렬**(고심각 팀원을 상단에)한 뒤 채운다.
  - `{{M_ATTN_CLASS}}`(행 강조): 고심각(sev-high)에 해당하는 위험을 가진 팀원이면 `attn`(좌측 적색 바+옅은 배경), 아니면 빈값.
  - `{{M_NAME}}` `{{M_ROLE}}`
  - `{{M_UTIL_PCT}}`(YTD 가동률, 막대 너비 0~100 정수; 100 초과 시 100으로 클램프) `{{M_UTIL_LABEL}}`(실제값 표기, 예 "74.9% · YTD") `{{M_UTIL_BARCLASS}}`(ok|warn|bad|빈값)
  - `{{M_LEAVE_PCT}}`(누적률을 0~100으로 클램프) `{{M_LEAVE_LABEL}}`(예 "9.5일 · 누적률 112%") `{{M_LEAVE_BARCLASS}}`
  - `{{M_CERT}}`(예 "4 보유 / 1 목표") `{{M_LASTONE}}`(예 "기록 없음" 또는 "12일 전") `{{M_GROWTH}}`(예 "분기평가 09-30 예정")

**리스트 토큰** `{{AREA_*_LIST}}` — `<li>...</li>` 항목들의 HTML로 채운다(영역별 2~4줄 핵심 요약). 데이터 없으면 `<li class="empty">기록 없음</li>` 한 줄.
- `{{AREA_ASSIGN_LIST}}` 투입/가동률(YTD 가동률·진행/종료 요약·과투입/현재 무투입)
- `{{AREA_LEAVE_LIST}}` 휴가/연차(잔여·누적률·휴가비 자격·향후 2주 계획·충돌)
- `{{AREA_CERT_LIST}}` 자격증(보유·목표·마감 임박·유지보수·환급)
- `{{AREA_GROWTH_LIST}}` 역량(TO-BE 설정 여부·진척·분기평가 due·정체)
- `{{AREA_1ON1_LIST}}` 1:1(최근 경과·미완료 액션·주기 준수)

## 조직 요약 치환 규칙 (org-template.html)

조직 전체 보고 시 `org-template.html`에 채운다. KPI는 **팀별 산출값의 집계**(재계산 금지).

**스칼라 토큰**
| 토큰 | 내용 |
|---|---|
| `{{REPORT_DATE}}` `{{GENERATED_AT}}` `{{ORG_TEAMCOUNT}}` | 헤더(기준일·생성일시·팀 수) |
| `{{ORG_KPI_HEADCOUNT_VALUE}}` `{{ORG_KPI_HEADCOUNT_NOTE}}` | 총 active 인원 = 팀별 인원 **합** |
| `{{ORG_KPI_UTIL_VALUE}}` `{{ORG_KPI_UTIL_NOTE}}` `{{ORG_KPI_UTIL_STATUS}}` | 평균 가동률 = 팀별 가동률 **평균**. note에 이상 팀 병기(예 "최저 52% 컨설팀") |
| `{{ORG_KPI_LEAVE_VALUE}}` `{{ORG_KPI_LEAVE_NOTE}}` `{{ORG_KPI_LEAVE_STATUS}}` | 평균 연차 누적률 = 팀별 **평균** |
| `{{ORG_KPI_CERT_VALUE}}` `{{ORG_KPI_CERT_NOTE}}` | 자격증 보유/목표 = 팀별 **합** (예 "18 / 9") |
| `{{ORG_KPI_1ON1_VALUE}}` `{{ORG_KPI_1ON1_NOTE}}` `{{ORG_KPI_1ON1_STATUS}}` | 1:1 due = 팀별 due 팀원 수 **합** |
| `{{RISK_MORE_NOTE}}` | 조직 Top 위험 노출 한도 초과분 "외 N건…"(없으면 빈값) |

**REPEAT 블록**
- `team`(팀 비교 1행): `{{T_ATTN_CLASS}}`(이상 팀이면 `attn`, 아니면 빈값) `{{T_NAME}}` `{{T_LEAD}}` `{{T_HEADCOUNT}}` `{{T_UTIL_PCT}}`(0~100 클램프) `{{T_UTIL_LABEL}}` `{{T_UTIL_BARCLASS}}`(ok|warn|bad) `{{T_LEAVE_PCT}}` `{{T_LEAVE_LABEL}}` `{{T_LEAVE_BARCLASS}}` `{{T_CERT}}` `{{T_1ON1}}`. 팀 행은 **이상치(과/저가동·누적률 이탈) 우선 정렬**.
- `risk`(조직 Top 위험): 팀별 리포트 형식과 동일하되 **제목에 팀명을 태그**(예 "[컨설팀] 저가동 52%"). `{{RISK_SEV_CLASS}}` `{{RISK_TITLE}}` `{{RISK_DETAIL}}` `{{RISK_PILL}}`. 심각도순 Top, 초과분은 `{{RISK_MORE_NOTE}}`.
- `teamlink`(드릴다운): `{{TL_NAME}}` `{{TL_LEAD}}` `{{TL_SUMMARY}}`(한 줄 요약, 예 "가동 75% · 인원 4") `{{TL_HREF}}`(**상대경로** `./{team-id}/{YYYY-MM-DD}-exec-report.html`). 각 팀 리포트가 같은 `exec/` 폴더 하위(`exec/{team-id}/...`)에 있어야 링크가 열린다(조직 요약도 `exec/`에 있음).

## 자체완결 규칙 (필수)

산출 HTML에 **외부 리소스를 넣지 마라.** `<script src=...>`, CDN(`http(s)://...`)으로 불러오는 CSS/폰트/이미지/차트 라이브러리 금지. 차트는 템플릿의 순수 CSS 막대(`<div class="bar"><span style="width:N%"></span></div>`)만 쓴다. 이유: 상급자가 사내망·오프라인에서 열어도 그대로 렌더되어야 하고, 단일 파일로 공유·보관·PDF화가 가능해야 한다.

## 빈/단일 데이터 견고성

팀원 1명, 투입 전부 ended(현재 무투입이나 YTD 가동률은 기간 집계로 0%가 아님), 1:1 기록 없음 같은 상태에서도 깨지지 말아야 한다. 빈 섹션은 "기록 없음"/empty로 표시하고, "데이터 미입력"을 위험으로 과장하지 마라(실제 위험과 구분). 0%·null은 0 또는 "—"로 안전하게 표기한다.

## 에러 핸들링

- 전문가 일부가 누락된 채 호출되면, 누락 영역은 해당 KPI/영역에 "수집 실패 — 재시도 필요"로 표기하고 나머지는 정상 산출한다(전체 중단 금지).
- 존재하지 않는 팀원 참조 등 데이터 무결성 문제는 임의 보정하지 말고 오케스트레이터에 보고한다.

## 테스트 시나리오

**정상:** 팀장 "이번 팀 현황 상급자 보고서 만들어줘" → team-management가 5개 전문가 종합 → 이 스킬이 `template.html`로 KPI·위험·팀원행·영역요약을 채워 `team-data/_reports/2026-06-17-exec-report.html` 생성 → 경로 보고 + PDF 안내.

**엣지(현재 데이터):** 팀원 1명·투입 전부 ended·1:1 없음 → 가동률 KPI는 **YTD 74.9%(ok)**(종료 투입도 기간 집계되므로 0% 아님), "현재 무투입"은 별도 위험 신호로, 1:1 "기록 없음", 휴가비 미달(used≤10)은 위험에서 제외, 빈 영역은 "기록 없음" — 레이아웃 정상.
