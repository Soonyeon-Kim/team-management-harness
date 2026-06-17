---
name: team-data-store
description: 팀원 관리툴의 파일 기반 데이터 저장소(team-data/) 레이아웃과 스키마. 팀원 프로필·1:1 기록·업무·성장 계획·휴가 데이터를 읽거나 쓸 때 반드시 이 스키마를 따른다. oneonone-coach·task-tracker·growth-coach·leave-manager 전 에이전트가 공유하는 단일 데이터 표준. 팀원 데이터를 조회/생성/수정하기 전에 이 스킬을 먼저 확인하라.
---

# 팀 데이터 저장소 (team-data/)

이 하네스는 코드 산출물이 없는 Claude 네이티브 툴이다. 모든 상태는 **프로젝트 루트의 `team-data/` 디렉토리**에 마크다운/JSON 파일로 영속된다. 전문가 에이전트는 이 저장소를 단일 진실 공급원(single source of truth)으로 읽고 쓴다.

스키마를 통일하는 이유: 4명의 전문가가 같은 팀원을 서로 다른 파일에서 참조한다. `slug`가 어긋나면 오케스트레이터가 교차검증할 때 같은 사람을 다른 사람으로 오인한다. 따라서 아래 규약은 협상 대상이 아니다.

## 디렉토리 레이아웃

```
team-data/
├── team.md                 # 팀 메타 (팀명, 팀장, 케이던스)
├── checklist.md            # 팀장 체크리스트 (checklist-setup이 정의)
├── members/
│   └── {slug}.md           # 팀원 프로필 1인 1파일
├── oneonones/
│   └── {slug}/
│       └── YYYY-MM-DD.md   # 1:1 기록 1회 1파일
├── growth/
│   └── {slug}.md           # AS-IS⇒TO-BE 역량 개발 1인 1파일
├── certifications/
│   ├── certs.json          # 전체 자격증 배열
│   └── team-goals.md       # 팀 차원 자격증 목표·파트너십 혜택
├── tasks/
│   └── tasks.json          # 전체 업무 배열 (보조 도구, 체크리스트 외)
└── leave/
    ├── leave.json          # 휴가 사용 내역 배열
    └── balances.json       # 팀원별 연차 잔여·누적률·휴가비
```

저장소가 없으면(최초 실행) 디렉토리와 빈 `tasks.json`(`{"tasks": []}`), `leave.json`(`{"leave": []}`), `certs.json`(`{"certs": []}`), `balances.json`(`{"year": <올해>, "balances": []}`)을 생성한다.

## 공통 규약

- **slug**: 팀원의 영문 소문자 kebab-case 식별자(예: `cheolsu-kim`). 한 번 정하면 바꾸지 않는다. 모든 파일·JSON에서 팀원 참조는 이 slug로 한다. 표시용 한글 이름은 프로필 `name`에 둔다.
- **날짜**: 항상 `YYYY-MM-DD` (ISO). 세션의 현재 날짜를 사용한다.
- **ID**: 업무 `T-{3자리}`, 휴가 `L-{3자리}`, 자격증 `C-{3자리}`. 기존 최대값+1로 부여한다.
- **쓰기 안전**: 파일을 수정할 때는 먼저 읽고(read-modify-write), 기존 필드를 보존한다. 전체 덮어쓰기로 다른 필드를 날리지 마라. 이유: 여러 전문가가 같은 파일을 시점 차로 건드린다.
- **삭제 금지 원칙**: 상충/오래된 데이터라도 임의 삭제하지 않는다. `status`로 표시하거나 별도 메모로 남긴다. 이유: 감사 추적과 복구 가능성.

## 스키마

### members/{slug}.md
```markdown
---
slug: cheolsu-kim
name: 김철수
role: 백엔드 개발자
joined: 2024-03-01
status: active        # active | on-leave | offboarding
---
## 강점
## 성장 영역
## 메모
```

### oneonones/{slug}/YYYY-MM-DD.md
```markdown
---
member: cheolsu-kim
date: 2026-06-17
mood: green           # green | yellow | red (팀원 컨디션 신호)
---
## 논의 주제
## 팀원 발언 요약
## 액션 아이템
- [ ] (담당) 내용 (마감: YYYY-MM-DD)
## 팔로업
- 지난 1:1 액션 점검 결과
```

### growth/{slug}.md (AS-IS ⇒ TO-BE 역량 개발)
```markdown
---
member: cheolsu-kim
updated: 2026-06-17
next_review: 2026-09-30        # 분기별 역량 평가 예정일
---
## AS-IS (현재 상태)
- 기술 스택 / 역량 수준:
- 역량 갭 (프로젝트 중 발견):
- 강점 / 개선 영역:
- 업무 만족도 / 성장 욕구:
## TO-BE (목표)
- 6개월 목표:
- 1년 목표:
- 비즈니스 방향성 연계:
- 측정 가능한 성과 지표:
- 확보 리소스 / 예산:
## 실행 & 모니터링
- 월간 1:1 진척 (oneonone-coach 연동):
- 교육 프로그램 참여:
- 멘토링 / 코칭:
- 프로젝트 역량 매칭:
## 성과 평가 (분기)
- 분기 역량 향상도:
- 성공 사례:
- 추가 지원 방안:
- 차기 목표 수정:
```

### tasks/tasks.json
```json
{
  "tasks": [
    {
      "id": "T-001",
      "title": "결제 모듈 리팩터링",
      "assignee": "cheolsu-kim",
      "status": "in-progress",
      "priority": "high",
      "due": "2026-06-20",
      "created": "2026-06-17",
      "updated": "2026-06-17",
      "notes": ""
    }
  ]
}
```
`status`: `todo` | `in-progress` | `blocked` | `done`. `priority`: `high` | `medium` | `low`.

### leave/leave.json
```json
{
  "leave": [
    {
      "id": "L-001",
      "member": "cheolsu-kim",
      "type": "annual",
      "unit": "full",
      "days": 1.0,
      "start": "2026-06-23",
      "end": "2026-06-23",
      "status": "approved",
      "note": ""
    }
  ]
}
```
- `type` (휴가 종류): `annual`(연차휴가) | `comp`(보상휴가) | `family-event`(경조사 휴가) | `maternity`(출산휴가) | `parental`(육아휴직) | `other`(기타).
- `unit` (단위): `full`(연차 1일=1.0) | `half`(반차=0.5) | `quarter`(반반차=0.25). 다일 휴가는 `full` + 기간으로 `days`를 산정한다.
- `days`: 소비 일수(숫자). 단일일이면 unit에 따라 1.0/0.5/0.25, 다일이면 기간 일수.
- `status`: `requested` | `approved` | `rejected`.
- **연차 차감 규칙(중요)**: `balances.json`의 `used`에는 **`type == "annual"` 이고 `status == "approved"`인 항목의 `days` 합만** 더한다. 보상·경조사·출산·육아휴직·기타는 연차에서 차감하지 않는다(누적률·마이너스 연차·휴가비 자격 계산에도 미포함).

### leave/balances.json (팀원별 연차·휴가비)
```json
{
  "year": 2026,
  "balances": [
    {
      "member": "cheolsu-kim",
      "granted": 15,             // 연간 부여 연차(일)
      "carryover": 0,            // 이월 연차(일)
      "used": 5,                 // 사용 연차(일, half-day는 0.5)
      "is_newcomer": false,      // 입사 1년 미만 여부
      "newcomer_expiry": null,   // 신입 월차 소멸 기준일 (해당 시 "YYYY-03-31")
      "vacation_fund_used": 300000,  // 휴가비 사용액(원), 한도 1,000,000
      "notes": ""
    }
  ]
}
```
- **잔여연차** = `granted + carryover - used`. (`used`는 연차휴가(type=annual)·승인 건의 `days` 합만 반영 — leave.json 차감 규칙 참조.)
- **누적률** = `잔여연차 / 잔여개월 × 100` (잔여개월 = 현재월~연말까지 남은 개월수). 목표 100~150%.
- **마이너스 연차** = `used > granted + carryover` (잔여 < 0).
- **휴가비 신청 자격** = `used > 10` (올해 연차 10일 초과 사용 시에만 연 100만원 휴가비 신청 가능). `used <= 10`이면 미활용을 독려 대상으로 띄우지 마라.
- 휴가 사용액·일수 갱신은 `leave.json`의 approved 휴가를 근거로 한다. `used`/`vacation_fund_used`를 임의 추정하지 마라.

### certifications/certs.json
```json
{
  "certs": [
    {
      "id": "C-001",
      "member": "cheolsu-kim",
      "name": "Salesforce Certified Administrator",
      "vendor": "Salesforce",
      "status": "in-progress",        // target | in-progress | acquired | expired
      "relevance": "고객사 SF 운영 프로젝트",  // 업무 연관성
      "exam_date": "2026-07-15",      // 시험일 (없으면 null)
      "application_deadline": "2026-07-01",  // 신청 마감일
      "acquired_date": null,
      "expiry_date": null,            // 자격 만료일
      "maintenance_due": null,        // 유지보수 활동 마감 (Salesforce 등)
      "reimbursed": false,            // 합격 환급 처리 여부
      "reimburse_amount": null,
      "notes": ""
    }
  ]
}
```

### certifications/team-goals.md
```markdown
---
updated: 2026-06-17
---
## 팀 차원 자격증 목표 (분기)
## 활용 가능한 파트너십 혜택 (벤더 — 혜택 — 활용 방안)
## 교육 세션 / 환급 정책 메모
```

### team.md
```markdown
---
team: 플랫폼팀
lead: 박팀장
cadence: weekly         # 체크리스트 실행 주기
---
## 팀 메모
```

## 무결성 규칙

- `assignee`/`member` 값은 반드시 `members/`에 존재하는 slug여야 한다. 없으면 작업 전에 오케스트레이터에 보고하라(임의로 팀원을 만들지 마라).
- 한 팀원을 offboarding 처리해도 그의 1:1·업무·휴가 기록은 보존한다(삭제 금지 원칙).
