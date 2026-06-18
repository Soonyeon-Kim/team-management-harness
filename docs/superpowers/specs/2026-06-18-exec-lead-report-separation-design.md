# 상급자용 / 팀장용 보고 리포트 분리 설계

**작성일:** 2026-06-18
**대상:** `exec-report-web` 스킬(`SKILL.md`·`template.html`), `team-management` 오케스트레이터(Phase5 리포트 흐름), `team-data/_reports/` 산출물 레이아웃
**선행 문서:** `2026-06-18-multiteam-report-design.md`(멀티-팀 데이터 모델·조직 요약·드릴다운). 본 문서는 그 §11에서 명시적으로 "비범위(YAGNI)"였던 **보고 대상(audience) 분리**를 추가하는 후속 설계다.

---

## 1. 배경 / 문제

멀티-팀 보고는 현재 **단방향 드릴다운**만 된다.

- 조직 요약(`_reports/{date}-org-summary.html`)은 "팀별 상세 리포트" 섹션에서 각 팀 리포트로 내려가는 링크가 있다(`./{team_id}/{date}-exec-report.html`).
- 그러나 팀 상세 리포트에는 **전사 요약으로 돌아오는 링크가 전혀 없다**(원본 `template.html`에 복귀 요소·토큰 자체가 없음). 즉 org → team 단방향만 동작.

근본 원인은 보고 대상이 두 종류인데 산출물이 한 종류라는 점이다.

- **상급자**(팀장보다 윗 직급): 전사 현황을 보고, 팀으로 드릴다운하고, **다시 전사로 복귀**할 수 있어야 한다.
- **팀장**: **자기 팀 리포트만** 본다. 전사 요약·타팀 리포트에 접근하면 안 된다.

요구: 상급자용/팀장용 보고를 구분한다. 상급자에겐 양방향 네비게이션(돌아오기 포함), 팀장에겐 본인 팀으로 제한.

## 2. 핵심 전제 — 접근 제한의 의미

산출물은 **로그인·서버가 없는 정적 자체완결 HTML**이다(하네스 원칙: 코드 산출물 없음). 따라서 "팀장은 자기 팀만 본다"는 제한은 코드/인증으로 강제할 수 없고, **누구에게 어떤 파일을 건네느냐(공유 경계)**로만 구현된다.

→ 설계의 본질은 **공유 경계를 물리적으로 명확하게 만드는 폴더 구조 + 그 경계에 맞는 네비게이션 차등**이다. 권한 시스템을 만드는 것이 아니다.

## 3. 확정된 결정 (브레인스토밍)

| 결정 | 값 |
|------|-----|
| 산출물 구조 | **폴더 2벌 분리** — `_reports/exec/`(상급자: org-summary + 팀 페이지, 폴더째 전달) / `_reports/leads/`(팀장: 본인 1파일만 전달). |
| 내용 차등 | **내용 동일, 네비게이션만 차이** — 같은 팀 데이터, exec 팀 페이지에만 "← 전사 현황으로" back 링크 추가. 한 템플릿 재사용. |
| 생성 트리거 | **요청 표현으로 분기** — "상급자/경영진/전사 보고" → exec 묶음, "팀장용/내 팀/팀별 배포용" → leads 파일. 둘 다 요청 시 둘 다. **명시 없으면 상급자용**(기존 동작 유지). |
| 적용 범위 | **멀티-팀 모드 전용** — 레거시 단일 팀은 org-summary가 없어 분리가 무의미하므로 기존 동작 유지(§7). |

## 4. 산출물 구조 (폴더 2벌)

```
team-data/_reports/
├─ exec/                                  ← 상급자: 폴더(또는 ZIP)째 전달
│   ├─ {YYYY-MM-DD}-org-summary.html       (전사 허브 → 팀으로 드릴다운)
│   ├─ ds/{YYYY-MM-DD}-exec-report.html     (상단 "← 전사 현황으로" back 링크 O)
│   └─ ai/{YYYY-MM-DD}-exec-report.html     (상단 back 링크 O)
└─ leads/                                 ← 팀장: 본인 1파일만 전달
    ├─ ds-{YYYY-MM-DD}.html                 (전사/타팀/back 링크 전부 X)
    └─ ai-{YYYY-MM-DD}.html
```

**현행 대비 경로 변경**

| 산출물 | 현행 | 신규 |
|--------|------|------|
| 조직 요약 | `_reports/{date}-org-summary.html` | `_reports/exec/{date}-org-summary.html` |
| 팀 상세(상급자) | `_reports/{team_id}/{date}-exec-report.html` | `_reports/exec/{team_id}/{date}-exec-report.html` |
| 팀장용(신규) | — | `_reports/leads/{team_id}-{date}.html` |

- org-summary → 팀 드릴다운은 여전히 `./{team_id}/{date}-exec-report.html` **상대경로** → `exec/` 내부에서 그대로 동작(변경 없음).
- exec 팀 페이지의 back 링크는 `../{date}-org-summary.html`(예: `exec/ds/` → `exec/`로 한 단계 위).
- `exec/{team_id}/...`와 `leads/{team_id}-{date}.html`은 **본문 내용 동일**, exec 쪽만 back 링크 보유.

## 5. 네비게이션 — back 링크 사양

- **위치:** exec 팀 페이지 `<body>` 최상단(헤더 카드 위)에 복귀 링크 1개.
- **문구:** `← 전사 현황으로`.
- **경로:** `../{YYYY-MM-DD}-org-summary.html` (상대경로, 같은 exec 묶음 내).
- **print 동작:** `@media print`에서 **숨김**(`display:none`). 상급자가 묶음을 PDF로 출력할 때 클릭 불가한 링크가 지면에 남지 않도록.
- **leads 파일:** back 링크 토큰을 **빈값으로 치환** → 끊긴 링크 없음, 전사 요약의 존재가 노출되지 않음.

## 6. 템플릿 변경 (`template.html`)

기존 토큰·구조·`<style>`은 보존하고 **추가만** 한다.

- `<body>` 최상단에 back 링크용 토큰 1개 추가: `{{NAV_BACK}}`.
  - exec 팀 페이지: `<a class="backlink" href="../{date}-org-summary.html">← 전사 현황으로</a>`로 치환.
  - leads 파일: 빈 문자열로 치환.
- `.backlink` CSS 추가(헤더 위 여백·링크 스타일), `@media print { .backlink{display:none;} }` 포함.
- `org-template.html`은 **구조 변경 없음**(저장 위치만 `exec/`로 이동).

## 7. 단일 팀(레거시 `team-data/`) 처리

- `teams/`가 없는 레거시 단일-팀 모드는 **org-summary 자체가 없으므로** exec/leads 구분이 무의미하다.
- 레거시에서는 기존 경로 `_reports/{date}-exec-report.html`를 그대로 유지하고 폴더 분리·back 링크를 적용하지 않는다.
- 폴더 2벌 분리와 back 링크는 **멀티-팀 모드(`teams/` 존재)에서만** 동작 → 하위호환 보존.

## 8. 트리거 / 라우팅 (`team-management` + `exec-report-web`)

요청 표현으로 산출 세트를 분기한다.

| 요청 표현 | 산출 |
|-----------|------|
| "상급자 보고", "경영진 리포트", "전사 보고", (표현 없음=기본) | **exec 묶음**: `exec/{date}-org-summary.html` + `exec/{team_id}/{date}-exec-report.html`(back 링크) |
| "팀장용", "내 팀", "팀별 배포용", "각 팀장에게 줄 리포트" | **leads 파일**: `leads/{team_id}-{date}.html`(back 링크 없음) |
| 둘 다 언급 | 둘 다 |

- `exec-report-web/SKILL.md` 절차에 **대상(audience) 분기**를 명문화: 팀별 렌더 결과(동일 데이터)를 exec 경로(back 링크 채움)와 leads 경로(back 링크 빈값) 중 요청에 따라 저장.
- `team-management/SKILL.md` Phase5(리포트 흐름)에 분기 안내 추가(전문가 팬아웃 결과 재사용은 그대로).
- exec 묶음은 org-summary가 먼저/함께 생성되어야 back 링크 대상이 존재한다.

## 9. 견고성 / 엣지 케이스

- **팀장용만 요청 + 멀티-팀:** org-summary 없이 leads 파일만 생성. back 링크 없으므로 dangling 없음.
- **상급자용 요청인데 팀 1개:** exec 묶음에 org-summary(미니표 1행) + 팀 페이지 1개(back 링크는 1행짜리 org로 복귀). 정상.
- **레거시 단일 팀:** §7대로 기존 경로·무분리. exec/leads 폴더를 만들지 않음.
- **같은 날 재실행:** 해당 세트 경로 덮어쓰기(기존 규칙 유지).
- **자체완결 불변:** back 링크는 상대경로 `<a href>`뿐 — 외부 리소스 0 규칙 유지.

## 10. 기존 2026-06-18 산출물 이행

- 이미 생성된 `_reports/ds/`·`_reports/ai/`·`_reports/{date}-org-summary.html`을 새 구조 `_reports/exec/`로 재생성하고, `_reports/leads/` 파일도 새로 산출해 E2E 확인.
- 현행 위치의 구버전 파일 정리(이동/삭제)는 구현 계획에서 처리.

## 11. 검증 (테스트 러너 없음 → grep + 육안)

- `template.html`에 `{{NAV_BACK}}` 토큰·`.backlink` CSS·`@media print` 숨김 존재(grep).
- 멀티-팀 모드에서 "상급자 보고" → `exec/{date}-org-summary.html` + `exec/{team_id}/{date}-exec-report.html` 생성, 팀 페이지 상단 back 링크 존재·org→team 드릴다운 링크 정상(육안: 클릭 왕복).
- "팀장용" → `leads/{team_id}-{date}.html` 생성, back/전사/타팀 링크 **부재**(grep: `org-summary`·`전사` 0건).
- 모든 산출물 외부 의존성 0(grep: `http`, `<script src`, CDN).
- 레거시 단일-팀 호출이 기존 경로 `_reports/{date}-exec-report.html`로 정상 산출(하위호환).

## 12. 산출물 (이번 작업 범위)

1. `.claude/skills/exec-report-web/template.html` — `{{NAV_BACK}}` 토큰 + `.backlink` CSS(print 숨김).
2. `.claude/skills/exec-report-web/SKILL.md` — 폴더 2벌 경로·audience 분기·치환 규칙(`{{NAV_BACK}}`) 명문화.
3. `.claude/skills/team-management/SKILL.md` — Phase5 분기 안내.
4. `meta_harness/CLAUDE.md` — 변경 이력 1행.
5. 산출물 재생성: `team-data/_reports/exec/`·`leads/`(2026-06-18 세트), 구버전 정리.

## 13. 비범위 (YAGNI)

- 실제 인증/권한 시스템, 비밀번호 보호, 암호화. (정적 파일 + 공유 경계로만 처리)
- 팀장이 자기 팀 내 개별 팀원에게 주는 3번째 대상(개인용) 리포트.
- 조직 계층 3단계+ 롤업(본부→실→팀). 현재는 상급자/팀장 2단계 평면.
- 상급자/팀장 간 **내용** 차등(요약 vs 상세). 이번엔 네비게이션만 차이.
- 실행 코드(JS)·서버·빌드 없음(코드 산출물 없는 Claude 네이티브 툴 원칙 유지).
