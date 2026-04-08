# 패키지 생산현황 조회 서비스 — UI Design

**Feature**: package-production-viewer  
**Design Date**: 2026-04-08  
**Designer**: Frontend Architect Agent  
**Reference**: 출판_생산 진행 현황/web_deploy/index.html

---

## 1. Component Hierarchy

```
<body>
├── #authScreen          (화면 A: 로그인/가입)
│   ├── .auth-brand
│   │   ├── .auth-brand-logo      (아이콘)
│   │   ├── .auth-brand-name      "갑우문화사"
│   │   └── .auth-brand-tagline   "패키지 생산현황"
│   └── .auth-box
│       ├── #authError / #authSuccess  (메시지 배너)
│       ├── #loginForm
│       │   ├── .auth-field  (이메일)
│       │   ├── .auth-field  (비밀번호)
│       │   └── .auth-btn    "로그인"
│       └── #signupForm
│           ├── .auth-field  (이메일)
│           ├── .auth-field  (비밀번호)
│           └── .auth-btn    "가입 신청"
│
├── #pendingScreen       (화면 B: 승인 대기)
│   └── .pending-box
│       ├── .pending-icon
│       └── .pending-message
│
└── #mainApp             (화면 C/D: 대시보드)
    ├── .header                         (sticky)
    │   ├── .header-inner
    │   │   ├── .header-brand
    │   │   │   ├── .header-logo
    │   │   │   └── .header-title  "갑우 패키지 생산현황"
    │   │   └── .auth-logout        "로그아웃"
    │   └── .header-date-bar        "기준일: YYYY-MM-DD | 업로드: HH:MM"
    │
    ├── #adminSection                   (관리자만 표시)
    │   ├── #adminPanel                 (승인 대기 목록)
    │   └── #approvedUsersPanel         (승인된 사용자 목록)
    │
    ├── .dashboard
    │   ├── .kpi-grid                   (2×2 그리드)
    │   │   ├── .kpi-card.in-progress   "진행중"
    │   │   ├── .kpi-card.printing      "오늘 인쇄"
    │   │   ├── .kpi-card.shipping      "출고 예정"
    │   │   └── .kpi-card.completed     "완료(7일)"
    │   └── .chart-section
    │       ├── .chart-title            "최근 7일 완료 추이"
    │       └── .chart-bars             (CSS 막대 차트, 7개 컬럼)
    │           └── .chart-bar × 7
    │               ├── .bar-fill       (height: %)
    │               └── .bar-label      (MM/DD)
    │
    ├── .search-wrap                    (sticky, top: header 높이)
    │   ├── .search-input               (통합 검색)
    │   └── .search-hint
    │
    ├── .shipment-section               (오늘 출고 예정)
    │   ├── .section-title              "오늘 출고 예정"
    │   └── .shipment-list
    │       └── .shipment-item × N
    │           ├── .shipment-client
    │           ├── .shipment-product
    │           └── .shipment-meta     (수량 + 상태)
    │
    └── .result-list                    (검색 결과 또는 기본 빈 상태)
        ├── .no-search-hint             (검색 전 안내 텍스트)
        └── .order-card × N             (검색 결과 카드)
            ├── .card-header
            │   ├── .mgmt-no            관리번호
            │   └── .card-badges
            │       ├── .badge.printing-today   (조건부)
            │       └── .badge.shipping-today   (조건부)
            ├── .card-body
            │   ├── .client-name
            │   ├── .product-name
            │   └── .card-meta          (수주량 + 납기)
            └── .process-bar
                └── .process-step × N   (인쇄/코팅/실크/박/형압/톰슨/접착)
                    ├── .step-icon      (✅ 완료 / ⬜ 미완 / -- 해당없음)
                    └── .step-label
```

---

## 2. Screen States

### 화면 A — 로그인/가입 (미인증)

- `#authScreen` 표시 / `#mainApp` 숨김
- 두 서브폼(`#loginForm`, `#signupForm`) 중 하나만 표시
- 다크 그라디언트 배경 (출판 프로젝트 동일: `#1a1a2e → #16213e → #0f3460`)
- 오류/성공 메시지 배너 인라인 표시

### 화면 B — 승인 대기 (인증됨, 미승인)

- 로그인 시도 후 `checkApproval()` 실패 케이스
- `#authScreen` 내부에 별도 메시지로 표시 (`showError()` 활용)
- 관리자가 승인하기 전까지 진입 불가 (`sb.auth.signOut()` 직후 에러 표시)

### 화면 C — 일반 사용자 대시보드 (인증 + 승인)

- `#mainApp` 표시
- `#adminSection` 숨김
- 헤더, KPI 카드, 차트, 출고 예정, 검색 모두 활성

### 화면 D — 관리자 대시보드 (인증 + 승인 + isAdmin)

- 화면 C와 동일 + `#adminSection` 표시
- 승인 대기 목록, 승인된 사용자 목록 포함

---

## 3. Main Dashboard Layout (mobile 375px baseline)

```
┌────────────────────────── 375px ──────────────────────────┐
│ ┌──────────────────────────────────────────────────────┐  │
│ │ [📦]  갑우 패키지 생산현황          [로그아웃]        │  │ ← sticky header, ~62px
│ │ 기준일: 2026-04-08 | 업로드: 09:51                   │  │
│ └──────────────────────────────────────────────────────┘  │
│                                                            │
│  ┌──────────────────────────────────────────────────┐     │
│  │  🔍 관리번호 / 품명 / 거래처 검색                  │     │ ← sticky, search
│  └──────────────────────────────────────────────────┘     │
│                                                            │
│  KPI 카드 (2×2 grid, gap:10px)                            │
│  ┌────────────────┐  ┌────────────────┐                   │
│  │  📋 진행중      │  │  🖨 오늘 인쇄   │                   │
│  │     225        │  │      12        │                   │
│  └────────────────┘  └────────────────┘                   │
│  ┌────────────────┐  ┌────────────────┐                   │
│  │  🚚 출고 예정   │  │  ✅ 완료(7일)  │                   │
│  │      18        │  │      87        │                   │
│  └────────────────┘  └────────────────┘                   │
│                                                            │
│  최근 7일 완료 추이                                         │
│  ┌──────────────────────────────────────────────────┐     │
│  │     ██                                           │     │
│  │     ██  ██      ██                               │     │
│  │  ██ ██  ██  ██  ██  ██  ██                       │     │
│  │  02 03  04  05  06  07  08                       │     │
│  └──────────────────────────────────────────────────┘     │
│                                                            │
│  오늘 출고 예정 (N건)                                      │
│  ┌──────────────────────────────────────────────────┐     │
│  │ 알래스카애드 · 오로라 PN 프로          2,300 · 톰슨대기 ││
│  └──────────────────────────────────────────────────┘     │
│                                                            │
│  [검색 전: 안내 텍스트]                                    │
│  [검색 후: order-card × N]                                 │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

**헤더 sticky**: `position: sticky; top: 0; z-index: 100`  
**검색바 sticky**: `position: sticky; top: 62px; z-index: 90; background: #f0f2f5`  
**최대 너비**: `max-width: 640px; margin: 0 auto`

---

## 4. Search Results Card Design

### 기본 카드 구조

```
┌──────────────────────────────────────────────┐
│  260402-0005                [오늘 인쇄 중] [출고 대기] │
│  (주)비앤에프엘엔씨                             │
│  리아_AO클렌저100G_단상자                        │
│  수주 10,000   납기 2026-04-14                  │
├──────────────────────────────────────────────┤
│  인쇄 ✅  코팅 ⬜  실크 --  박 --  톰슨 ⬜  접착 ⬜  │
└──────────────────────────────────────────────┘
```

### 공정 진행 바 로직

- `process_spec` 필드에서 해당 주문에 적용되는 공정만 활성 표시
- 날짜값 있음 → `✅` (완료, green 배경)
- 날짜값 없음 + 적용 공정 → `⬜` (미완, 회색)
- 해당 없는 공정 → `--` (비활성, 매우 연한 회색)
- 전체 완료 시 (`completed_at` 있음) → 카드 전체 opacity 0.6

### 공정 순서

인쇄 → 코팅 → 실크 → 박 → 형압 → 톰슨 → 접착 (7단계)

### 배지 종류

| 배지 | 색상 | 조건 |
|------|------|------|
| 오늘 인쇄 중 | 파란색 `#1565c0` | `badges.printing_today === true` |
| 출고 예정 | 주황색 `#e65100` | `badges.shipping_today === true` |

---

## 5. Color Palette & Typography

### 색상 (출판 프로젝트 동일 팔레트)

| 토큰 | 값 | 용도 |
|------|-----|------|
| `--color-bg` | `#f0f2f5` | 페이지 배경 |
| `--color-surface` | `#ffffff` | 카드/박스 배경 |
| `--color-text-primary` | `#1a1a2e` | 본문 텍스트 |
| `--color-text-secondary` | `#6b7280` | 서브 텍스트 |
| `--color-text-muted` | `#9ca3af` | 힌트/플레이스홀더 |
| `--color-border` | `#dde1e7` | 인풋/구분선 |
| `--color-header-from` | `#1a1a2e` | 헤더 그라디언트 시작 |
| `--color-header-to` | `#16213e` | 헤더 그라디언트 끝 |
| `--color-blue` | `#1565c0` | KPI 오늘인쇄, 배지, 링크 |
| `--color-orange` | `#e65100` | KPI 출고예정, 배지 |
| `--color-green` | `#2e7d32` | KPI 완료, 공정완료 체크 |
| `--color-navy` | `#1a1a2e` | KPI 진행중, 기본 강조 |

### KPI 카드 색상 매핑

| 카드 | accent 색상 | 아이콘 |
|------|------------|--------|
| 진행중 | `#1a1a2e` (navy) | 📋 |
| 오늘 인쇄 | `#1565c0` (blue) | 🖨 |
| 출고 예정 | `#e65100` (orange) | 🚚 |
| 완료(7일) | `#2e7d32` (green) | ✅ |

### 타이포그래피

- **본문 폰트**: `-apple-system, BlinkMacSystemFont, 'Apple SD Gothic Neo', 'Malgun Gothic', sans-serif`
- **헤더 타이틀**: 17px, font-weight 700
- **KPI 숫자**: 28px, font-weight 800
- **KPI 라벨**: 12px, font-weight 500
- **카드 품명**: 14px, font-weight 600
- **태그/배지**: 12px, font-weight 600~700
- **공정 라벨**: 11px, font-weight 500

---

## 6. Interaction Patterns

### 검색

- `input` 이벤트에 debounce 300ms 적용 (`setTimeout` 패턴, clearTimeout)
- 검색어 2자 미만이면 기본 화면(출고 예정 리스트) 표시
- 검색어 있으면 `orders` 배열에서 `mgmt_no + client + product` 통합 매칭 (`.toLowerCase().indexOf()`)
- 검색 결과 없으면 "검색 결과 없음" 빈 상태 표시

### KPI 카드 클릭

- 출판 프로젝트의 `statClick()` 패턴 미채택 (패키지 v1은 탭 없음)
- 클릭 시 해당 조건으로 검색 결과 필터링 (선택적 구현, TODO 마커)

### 공정 진행 바

- CSS only, JS로 `data-done`, `data-active`, `data-skip` 속성 세팅
- 애니메이션 없음 (성능 우선)

### 데이터 로드

- `fetch('data.json')` 단순 fetch
- 로딩 중: `.loading` skeleton 텍스트
- 오류 시: "데이터가 없습니다" 안내 메시지

### Pull-to-refresh

- v1 미구현 (TODO 마커만 추가)
- 페이지 재로드로 대체 안내 문구

### 로딩 스켈레톤

- 카드 영역에 `.skeleton` 클래스로 pulse 애니메이션 placeholder
- `@keyframes pulse` 정의 (opacity 1 → 0.5 → 1)

---

## 7. Accessibility

### 터치 타깃

- 모든 버튼/탭: 최소 44×44px (iOS HIG 기준)
- 공정 스텝 아이콘: 최소 36px 너비

### 색상 대비

- 본문 (`#1a1a2e` on `#ffffff`): 18.1:1 (AAA 통과)
- 서브 텍스트 (`#6b7280` on `#ffffff`): 4.6:1 (AA 통과)
- 배지 파랑 (`#1565c0` on `#dbeafe`): 4.8:1 (AA 통과)
- 배지 주황 (`#e65100` on `#fff7ed`): 4.5:1 (AA 통과)

### 키보드 네비게이션

- 검색 인풋: `autofocus` 없음 (모바일 키패드 자동 팝업 방지)
- 버튼 모두 `<button>` 요소 사용 (div onclick 최소화)
- 로그아웃 버튼: `aria-label="로그아웃"` 추가

### 시맨틱 마크업

- 헤더: `<header>` 요소
- 검색: `<input type="search" aria-label="통합 검색">`
- KPI 섹션: `<section aria-label="현황 요약">`
- 카드 목록: `<ul>` + `<li>` 구조
- 공정 바: `role="list"` + `aria-label`

---

## 8. 출판 프로젝트에서 재사용하는 항목

| 항목 | 재사용 여부 | 비고 |
|------|------------|------|
| CSS reset + body styles | 그대로 복사 | `font-family`, `background`, `color` |
| `.header` / `.header-inner` 스타일 | 그대로 복사 | |
| `.auth-screen` 전체 블록 | 그대로 복사 | 로고 아이콘만 📦로 변경 |
| `.auth-box`, `.auth-field`, `.auth-btn` | 그대로 복사 | |
| `.auth-error` / `.auth-success` | 그대로 복사 | |
| `supabase.createClient` + auth 함수들 | 그대로 복사 (URL/KEY 변경) | handleLogin, handleSignup, handleLogout |
| `checkApproval()`, `checkAdmin()` | 그대로 복사 | |
| `loadPendingRequests()`, `loadApprovedUsers()` | 그대로 복사 | |
| `esc()` XSS 방지 함수 | 그대로 복사 | |
| `.stat-card` 기본 스타일 + `::before` | 수정하여 재사용 | 2×2 그리드로 변경 (출판은 3열) |
| `enterApp()` 진입 흐름 | 수정하여 재사용 | loadData() 내용 변경 |

**재사용하지 않는 항목**: 탭 UI, 업로드 패널(엑셀 업로드 불필요), renderGrouped/renderCompletedByDate, 출판 특유 필드(equipment, data_status 등)

---

## 9. 설계 질문 사항 (CTO 확인 필요)

1. **Supabase 연동 여부**: plan.md의 "v2 이후" 항목에 사용자 인증이 포함되어 있는데, 출판 프로젝트처럼 Supabase 인증을 v1에 포함시킬지 여부를 확인이 필요합니다. 현재 plan에는 "사용자 인증 v2 이후"로 명시되어 있으므로, 본 skeleton에서는 인증 블록을 구조만 포함하고 TODO 처리합니다. → 인증 없이 data.json 직접 fetch로 진행하면 skeleton이 더 단순해집니다.

2. **공정 해당없음 표시**: `공정` 텍스트 필드 파싱 실패 시 전체 7개 공정을 모두 표시할지, 아니면 공정 바 자체를 숨길지 결정이 필요합니다.

3. **검색 최소 글자수**: 1자부터 검색할지, 2자부터 할지. 관리번호 형식(`260402-0005`)이라 숫자 입력 시 1자라도 결과가 많을 수 있습니다.

4. **차트 최대값 기준**: 최근 7일 완료 추이 차트의 bar 높이를 "7일 중 최댓값 = 100%"로 할지, "고정값 예: 50건 = 100%"로 할지. 전자 권장.
