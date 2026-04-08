# 패키지 생산현황 조회 서비스 — Requirements Document

> **Summary**: 갑우문화사 패키지 사업부 생산현황을 모바일에서 조회하는 MVP 서비스의 요구사항 정의서
>
> **Author**: PM Agent
> **Created**: 2026-04-08
> **Last Modified**: 2026-04-08
> **Status**: Draft
> **Related Plan**: [package-production-viewer.plan.md](../01-plan/features/package-production-viewer.plan.md)

---

## 1. User Personas

### Persona A — 영업사원 (Primary)

| 항목 | 내용 |
|------|------|
| 역할 | 패키지 사업부 영업담당자 |
| 사용 환경 | 스마트폰 (외근 중, 이동 중 포함) |
| 핵심 니즈 | 거래처 전화 문의를 받았을 때 30초 내 진행 상황 답변 |
| 고통점 | 현재는 사무실 PC의 엑셀 3개 파일을 열어야 하며, 모바일에서는 NAS 접근 불가 |
| 성공 기준 | "오늘 인쇄 끝났나요?" 질문에 현장에서 즉답 가능 |

### Persona B — 생산팀장 (Secondary)

| 항목 | 내용 |
|------|------|
| 역할 | 생산 일정 및 인쇄기 운영 총괄 |
| 사용 환경 | 공장 현장 + 스마트폰 |
| 핵심 니즈 | 오늘 인쇄 예정 건, 출고 예정 건을 한눈에 파악 |
| 고통점 | 인쇄종합 파일이 18MB 대형 파일이라 모바일 열람 사실상 불가 |
| 성공 기준 | 아침 출근 시 오늘 작업 현황을 30초 안에 파악 |

### Persona C — 외부 거래처 담당자 (v2 Future)

| 항목 | 내용 |
|------|------|
| 역할 | 코스맥스, 에뛰드 등 패키지 발주 거래처 담당자 |
| 사용 환경 | 스마트폰 / PC |
| 핵심 니즈 | 자사 발주 건의 인쇄/납기 진행률 자체 확인 |
| 고통점 | 현재는 갑우 영업사원에게 개별 연락해야 함 |
| 성공 기준 | 24시간 언제든 발주 건 상태 자기 확인 가능 |
| v1 처리 | 이 페르소나의 요구사항은 v2에서 구현. v1 범위에서 제외. |

---

## 2. User Stories

### 검색 기능

**US-001**
> As a 영업사원, I want to search by 관리번호, 품명, or 거래처명 in a single search field, so that I can find any order without knowing which Excel file it's in.

**Acceptance Criteria:**
- Given I am on the dashboard
- When I type "코스맥스" in the search field
- Then all orders with "코스맥스" in client name are shown as result cards within 1 second

---

**US-002**
> As a 영업사원, I want to see a result card showing 공정 진행 현황 (which steps are done vs. pending), so that I can immediately describe where the order is in production.

**Acceptance Criteria:**
- Given search results are displayed
- When I look at a result card
- Then I see: 관리번호, 거래처, 품명, 수주량, 납기, and a progress bar showing 인쇄/코팅/박/실크/형압/톰슨/접착 steps with completed (filled) vs. pending (empty) state

---

**US-003**
> As a 영업사원, I want result cards to display contextual badges (e.g., "오늘 인쇄 중", "출고 대기"), so that I can convey status at a glance without reading all details.

**Acceptance Criteria:**
- Given an order is scheduled for printing today (from 인쇄종합 파일)
- When its result card is displayed
- Then a "오늘 인쇄 중" badge is shown on the card
- And if the order appears in today's 출고리스트, a "출고 대기" badge is also shown

---

### KPI 카드

**US-004**
> As a 생산팀장, I want to see 4 KPI cards at the top of the dashboard (진행중 / 오늘 인쇄 / 출고 예정 / 완료), so that I can get a production snapshot without any search.

**Acceptance Criteria:**
- Given I open the dashboard
- When the page loads
- Then 4 KPI cards are displayed showing: count of in-progress orders, today's print count, today's shipping count, and completed orders
- And each card shows a number prominently readable on mobile

---

**US-005**
> As a 생산팀장, I want the KPI cards to reflect data as of the last manual update, so that I trust the numbers represent the latest committed data.

**Acceptance Criteria:**
- Given the data was last updated on a specific date
- When I view the dashboard
- Then the last update timestamp ("업데이트: YYYY-MM-DD HH:MM") is visible at the top of the page

---

### 출고 예정 리스트

**US-006**
> As a 생산팀장, I want to see today's scheduled shipment list (오늘 출고 예정), so that I can prepare logistics without opening the shipment Excel file.

**Acceptance Criteria:**
- Given the 출고리스트 파일 has a sheet for today's date
- When I view the dashboard
- Then a "오늘 출고 예정" section lists: 거래처, 품명, 출고수량, 현재상태
- And if no shipments are scheduled today, the section displays "오늘 출고 예정 없음"

---

### 완료 추이 차트

**US-007**
> As a 생산팀장, I want to see a bar chart of completed orders for the last 7 days, so that I can gauge production throughput trends at a glance.

**Acceptance Criteria:**
- Given orders have completion dates recorded (생산일 column)
- When I view the dashboard
- Then a bar chart shows completed order counts for each of the last 7 calendar days
- And each bar is labeled with the date (MM/DD format)

---

### 인증 및 접근 제어

**US-008**
> As a 영업사원, I want to sign up with my email and password, so that I can request access to the production dashboard.

**Acceptance Criteria:**
- Given I am not logged in
- When I visit the service URL
- Then I see a login/signup screen
- And when I submit a signup form with email and password, my request is recorded as "pending"
- And I see a message: "관리자 승인 후 이용하실 수 있습니다"

---

**US-009**
> As the admin (zephirus03@gmail.com), I want to see pending signup requests and approve or reject them, so that I control who accesses the production data.

**Acceptance Criteria:**
- Given I am logged in as admin
- When I view the admin section
- Then I see a list of pending signup requests with applicant email and timestamp
- And when I click "승인", the user is added to approved_users and can access the dashboard
- And when I click "거절", the request is removed and the user cannot access

---

**US-010**
> As an approved user, I want to stay logged in across browser sessions (when I check "로그인 유지"), so that I don't have to log in every time I open the app on my phone.

**Acceptance Criteria:**
- Given I am an approved user
- When I log in with "로그인 유지" checked
- Then my session persists even after closing and reopening the browser
- And when "로그인 유지" is not checked, the session expires when the browser is closed

---

**US-011**
> As a pending or unapproved user, I want to see a clear message explaining my status, so that I know why I cannot access the dashboard.

**Acceptance Criteria:**
- Given I have signed up but not yet been approved
- When I log in with valid credentials
- Then I see "승인 대기 중입니다. 관리자에게 문의하세요." and cannot access the dashboard content

---

### 데이터 신뢰성

**US-012**
> As a 영업사원, I want the production data to match the source Excel files exactly, so that I can give accurate answers to clients without risking misinformation.

**Acceptance Criteria:**
- Given the convert.py script has been run on the latest Excel files
- When I compare 10 randomly sampled orders between the web dashboard and the source Excel
- Then all fields (관리번호, 품명, 수주량, 납기, 공정 완료 날짜) match exactly

---

---

## 3. Functional Requirements

### FR-001: Single-Page Dashboard Layout
The service must be a single HTML page with no tab navigation. All sections (search, KPI, chart, shipment list) coexist on one scrollable page.

### FR-002: Unified Search Field
A single text search field must accept 관리번호, 품명, and 거래처명 as search terms. Partial matching (substring) is required. Search must execute client-side on the pre-loaded data.json without any server round-trip.

### FR-003: Search Result Card
Each search result must display a card containing:
- 관리번호
- 거래처명
- 품명
- 수주량 (formatted with thousand separators)
- 납기예정일 (YYYY-MM-DD)
- 공정 진행 바 (only applicable processes shown per 공정 field in source data)
- Contextual badges (오늘 인쇄 중 / 출고 대기)

### FR-004: KPI Cards (4 items)
The dashboard must display 4 KPI summary cards:
1. **진행중**: Count of orders with at least one process started but 생산일 empty
2. **오늘 인쇄**: Count of orders where 계획일 = today (from 인쇄종합 인쇄일정관리 sheet)
3. **출고 예정**: Count of rows in today's 출고리스트 sheet
4. **완료**: Count of orders with 생산일 populated

### FR-005: Last 7 Days Completion Chart
A bar chart must show completed order counts per day for the 7 calendar days ending today (inclusive). Source: 생산일 column in 수주관리. Days with zero completions must still appear as zero-height bars.

### FR-006: Today's Shipment List
A summary list must display all rows from the most recent date sheet in the 출고리스트 file. Each row shows: 업체명, 제품명, 출고수량, 비고(상태). If no sheet matches today's date, display "오늘 출고 예정 없음".

### FR-007: Data Timestamp Display
The page header must display the data generation timestamp from data.json's `generated_at` field, formatted as "업데이트: YYYY-MM-DD HH:MM".

### FR-008: Data Conversion Script
A Python script (`convert.py`) must:
- Read 수주관리 all monthly sheets (pattern: "수주관리 2026. NN월")
- Read 인쇄종합 only the "인쇄일정관리" sheet (skip all other sheets)
- Read 출고리스트 only the most recent MMDD-pattern sheet
- Merge all data into a single `web/data.json`
- Never modify the source Excel files (read-only access)
- Print a clear error message and exit non-zero on parse failure (preserving existing data.json)

### FR-009: Process Progress Bar
The progress bar in search result cards must:
- Only display processes applicable to the specific order (derived from the 공정 text field)
- Show completed processes (date present in column) as filled/checked
- Show pending processes as empty/unchecked
- Default to showing all 7 processes if 공정 field cannot be parsed

### FR-010: Badge Logic
- "오늘 인쇄 중" badge: shown when the order's 수주번호 matches a row in 인쇄일정관리 where 계획일 = today
- "출고 대기" badge: shown when the order's 품명 fuzzy-matches a row in today's 출고리스트. If fuzzy match confidence is below threshold, badge is omitted rather than shown incorrectly.

### FR-011: Supabase Authentication — Signup Flow
New users must go through a two-step flow:
1. Submit signup form → Supabase Auth account created + row inserted into `signup_requests` table with status "pending"
2. Admin approves → row inserted into `approved_users` table
3. User can then log in and access the dashboard

### FR-012: Supabase Authentication — Login Flow
- Login screen shown to all unauthenticated visitors
- "로그인 유지" checkbox: when checked, session stored in localStorage (persistent); when unchecked, session stored in sessionStorage (tab-session only)
- After login, check `approved_users` table for the user's email
- If not approved: show pending message, do not render dashboard

### FR-013: Admin Panel
- Admin UI visible only to zephirus03@gmail.com
- Admin can see list of pending signup requests
- Admin can approve (INSERT to approved_users) or reject (DELETE from signup_requests) each request
- All admin actions must be enforced by Supabase RLS policies (server-side), not just frontend UI hiding

### FR-014: Supabase RLS Enforcement
The following RLS policies must be applied (mirroring the 출판 project's security-spec.md):
- `approved_users`: SELECT restricted to own email; INSERT restricted to admin only
- `signup_requests`: INSERT restricted to own email; SELECT for own or admin; UPDATE/DELETE for admin only
- Storage (`gabwoo-data` bucket or equivalent): download restricted to approved users; upload/update restricted to admin

### FR-015: Static Deployment
The service must deploy as a static site on Vercel with no server-side rendering. `vercel.json` must be present and must include security headers (X-Frame-Options, X-Content-Type-Options, Strict-Transport-Security).

### FR-016: Mobile-First Layout
All UI components must be usable on a 375px-wide viewport without horizontal scrolling. Touch targets must be at minimum 44x44px.

---

## 4. Non-Functional Requirements

### NFR-001: Performance — Initial Load
The page must be fully usable (dashboard visible, KPI cards rendered) within 3 seconds on a 4G mobile connection. data.json must be under 2MB.

### NFR-002: Performance — Search Response
Search results must render within 500ms of the user stopping typing (debounce 300ms). All filtering is client-side; no network request is made at search time.

### NFR-003: Data Accuracy
Data displayed in the dashboard must match the source Excel files with 100% accuracy for the following fields: 관리번호, 거래처명, 품명, 수주량, 납기예정일, and all process completion dates. Verification: manual spot-check of 10 randomly selected orders before each production deployment.

### NFR-004: Mobile-First
The primary design target is a 375px-wide smartphone screen. The layout must be usable one-handed. Text must be readable without zooming.

### NFR-005: Offline Graceful Degradation
If the network is unavailable after initial load, the previously loaded dashboard data must remain visible. A "오프라인" indicator may be shown, but the page must not break.

### NFR-006: Browser Support
Must function on Chrome for Android (latest), Safari for iOS 15+, and Chrome/Safari desktop.

### NFR-007: Data Freshness Transparency
The data generation timestamp must always be visible so users know how current the data is. There is no automatic refresh in v1; users must be informed that data is updated manually.

### NFR-008: No Server-Side Code
v1 is a pure static site. No backend API, no server-side functions. All logic runs in the browser or in the local Python conversion script.

### NFR-009: Source File Integrity
The convert.py script must open all Excel files in read-only mode. It must never write to, move, rename, or delete any file in the `생산진행현황/` directory.

### NFR-010: Password Security
User passwords must be a minimum of 8 characters. This must be enforced both in the frontend form validation and in Supabase Dashboard authentication settings.

---

## 5. Approval Flow Requirements

### 5.1 Flow Overview

```
[신규 사용자]
    │
    ▼
[회원가입 폼 제출]
    │  이메일 + 비밀번호 입력
    ▼
[Supabase Auth 계정 생성]
    │
    ▼
[signup_requests 테이블에 INSERT]
    │  status: "pending"
    ▼
[사용자에게 안내: "관리자 승인 후 이용 가능"]
    │
    ▼ (관리자 검토)
[관리자 (zephirus03@gmail.com) 로그인]
    │
    ▼
[Admin 패널: 대기 신청 목록 확인]
    │
    ├─── 승인 클릭 ──▶ [approved_users INSERT] ──▶ [사용자 대시보드 접근 허용]
    │
    └─── 거절 클릭 ──▶ [signup_requests DELETE] ──▶ [사용자 재신청 필요]
```

### 5.2 Supabase Tables Required

| 테이블 | 컬럼 | 설명 |
|--------|------|------|
| `signup_requests` | email, name, requested_at, status | 가입 신청 목록 |
| `approved_users` | email, approved_at, approved_by | 승인된 사용자 목록 |

### 5.3 Initial Admin

- **Admin email**: `zephirus03@gmail.com` (hardcoded in both frontend UI and Supabase RLS policies)
- Admin is pre-inserted into `approved_users` table at setup time
- Admin account is created directly in Supabase Auth (not through signup flow)

### 5.4 RLS Policy Summary

All permission enforcement happens server-side in Supabase. Frontend UI hiding is supplementary only.

| 테이블 | 작업 | 허용 대상 |
|--------|------|-----------|
| `approved_users` | SELECT | 본인 이메일만 |
| `approved_users` | INSERT | 관리자(zephirus03@gmail.com)만 |
| `signup_requests` | INSERT | 본인 이메일만 |
| `signup_requests` | SELECT | 본인 또는 관리자 |
| `signup_requests` | UPDATE | 관리자만 |
| `signup_requests` | DELETE | 관리자만 |
| Storage (data bucket) | SELECT (다운로드) | approved_users에 있는 사용자만 |
| Storage (data bucket) | INSERT/UPDATE (업로드) | 관리자만 |

### 5.5 Reference Implementation

The auth pattern is identical to the 출판 project. Reference:
`/Users/jack/dev/gabwoo/출판_생산 진행 현황/docs/02-design/security-spec.md`

The Design phase must reuse this RLS SQL verbatim, substituting the Supabase project ID and storage bucket name.

---

## 6. Priority Matrix (MoSCoW)

### Must Have (v1 배포 필수)

| ID | 요구사항 |
|----|----------|
| FR-001 | 단일 페이지 대시보드 |
| FR-002 | 통합 검색 필드 |
| FR-003 | 검색 결과 카드 (공정 진행 바 포함) |
| FR-004 | KPI 카드 4개 |
| FR-006 | 오늘 출고 예정 리스트 |
| FR-007 | 데이터 업데이트 타임스탬프 |
| FR-008 | 엑셀 → JSON 변환 스크립트 |
| FR-009 | 공정 진행 바 로직 |
| FR-011 | 가입 → 대기 → 승인 인증 플로우 |
| FR-012 | 로그인 / 로그인 유지 기능 |
| FR-013 | 관리자 패널 |
| FR-014 | Supabase RLS 정책 |
| FR-015 | Vercel 정적 배포 |
| FR-016 | 모바일 퍼스트 레이아웃 |
| NFR-003 | 데이터 정확도 100% |
| NFR-009 | 원본 엑셀 읽기 전용 접근 |

### Should Have (v1 내 포함 권장)

| ID | 요구사항 |
|----|----------|
| FR-005 | 최근 7일 완료 추이 차트 |
| FR-010 | 배지 로직 (오늘 인쇄 중 / 출고 대기) |
| NFR-001 | 3초 이내 초기 로드 |
| NFR-002 | 검색 500ms 응답 |
| NFR-005 | 오프라인 상태 유지 |
| NFR-010 | 비밀번호 최소 8자 |

### Could Have (시간 여유 시 포함)

| ID | 요구사항 |
|----|----------|
| - | 에러 메시지 일반화 (Supabase 내부 메시지 노출 방지) |
| - | XSS 방어 강화 (esc() 함수 작은따옴표 이스케이핑) |
| - | 사용자 삭제 기능 (관리자 패널) |
| - | 검색어 하이라이팅 |

### Won't Have (v1 명시적 제외)

| 항목 | 사유 |
|------|------|
| 거래처별 TOP 10 대시보드 | 사용자가 명시적으로 제외 요청 |
| 개별 탭 분리 UI (수주/인쇄/출고) | 미니멀 단일 페이지 방향 확정 |
| 구글 드라이브 자동 동기화 | v2 범위 |
| 웹 업로드 버튼 | 정적 배포와 충돌, v2 범위 |
| 생산성 통계 파일 활용 | 실시간 참조 아님 |
| PDF / 엑셀 내보내기 | v2 범위 |
| 외부 거래처 전용 화면 | v2 범위 (Persona C) |
| 이력 추적 / 알림 / 푸시 알림 | v2 범위 |

---

## 7. Out of Scope (v1 명시적 제외)

아래 항목은 v1에서 구현하지 않으며, 향후 버전에서 검토합니다.

1. **원본 엑셀 파일 수정/이동/삭제**: 변환 스크립트는 읽기 전용으로만 접근 (CLAUDE.md 최우선 규칙)
2. **생산성 통계 파일 (인쇄종합의 기타 17개 시트)**: 화면 노출 불필요, 변환 시 무시
3. **거래처별 진행 현황 TOP 10**: 사용자 명시 제외
4. **개별 탭 (수주/인쇄/출고 분리)**: 단일 검색으로 대체
5. **자동 데이터 갱신 (Google Drive API)**: v2 범위
6. **웹 브라우저 내 파일 업로드**: 정적 배포 구조와 충돌
7. **알림 / 푸시 알림**: v2 범위
8. **PDF 다운로드 / 엑셀 내보내기**: v2 범위
9. **외부 거래처 포털 (Persona C)**: v2 범위
10. **다국어 지원**: 한국어 단일 언어로 고정

---

## 8. Open Questions — CTO Clarification Needed

다음 항목은 Plan 문서에서 명확하지 않거나 이 요구사항 문서 작성 중 발견된 모호성입니다.

### AQ-001: 인증 포함 여부 (Priority: High)
**상황**: Plan 문서 섹션 6 "Out of Scope"에 "사용자 인증"이 명시적으로 제외되어 있습니다. 그러나 이 요구사항 문서 작성 지침은 Supabase 인증(가입→승인→접근 플로우)을 v1에 포함하도록 지시합니다.

**질문**: v1에 인증을 포함합니까? 포함한다면, 출판 프로젝트와 동일한 RLS 구조를 그대로 재사용합니까?

**권장**: 출판 프로젝트 인증이 이미 검증되었으므로 재사용 비용이 낮습니다. v1에 포함을 권장합니다.

---

### AQ-002: 출고리스트 날짜 시트 매칭 규칙 (Priority: Medium)
**상황**: Plan에 "MMDD 패턴 중 가장 최근 날짜 시트"를 사용한다고 명시되어 있습니다. 그러나 실제 파일명이 `출고품목리스트 (4월).xlsx`이고 시트명 패턴이 불규칙할 수 있습니다.

**질문**: "오늘 출고 예정"은 항상 당일 날짜 시트입니까, 아니면 "가장 최근 시트"입니까? 주말/휴일에 시트가 없으면 어떻게 처리합니까?

---

### AQ-003: 품명 Fuzzy 매칭 임계값 (Priority: Medium)
**상황**: 출고리스트에 관리번호가 없어 품명 기반 fuzzy 매칭으로 배지를 붙입니다. Plan에 "신뢰도 낮으면 배지 생략"이라고만 명시되어 있습니다.

**질문**: 구체적인 신뢰도 임계값은 누가 결정합니까? 배지 오표시 vs. 누락 중 어느 쪽이 더 큰 문제입니까? (영업사원에게 잘못된 정보 제공 vs. 배지 미표시)

---

### AQ-004: 데이터 갱신 주기 및 배포 권한 (Priority: Medium)
**상황**: convert.py 실행 후 git commit & push → Vercel 자동 배포 구조입니다. Plan에 이 작업을 누가, 얼마나 자주 수행하는지 명시되어 있지 않습니다.

**질문**: 데이터 갱신은 누가 담당합니까? (IT 담당자? 생산팀장?) 갱신 주기는 매일 아침입니까? Vercel git 접근 권한 설정이 필요합니다.

---

### AQ-005: "완료" KPI 카드의 기간 범위 (Priority: Low)
**상황**: FR-004에서 "완료" KPI는 생산일이 있는 건의 count라고 정의했습니다. 그러나 이것이 "올해 전체 누적"인지 "최근 30일"인지 명확하지 않습니다. 연간 누적은 숫자가 너무 커질 수 있습니다.

**질문**: "완료" KPI는 어떤 기간의 완료 건수를 보여야 합니까?

---

## Version History

| Version | Date | Changes | Author |
|---------|------|---------|--------|
| 1.0 | 2026-04-08 | Initial draft | PM Agent |
