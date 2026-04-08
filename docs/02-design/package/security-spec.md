# Security Specification: 패키지 생산현황 조회 서비스

| 항목 | 내용 |
|------|------|
| Feature | package-production-viewer |
| 작성일 | 2026-04-08 |
| 작성자 | Security Architect Agent |
| 대상 | `web/index.html` (순수 HTML/JS SPA) + Supabase Auth/DB/Storage |
| 배포 | Vercel 정적 배포 |
| 레퍼런스 | `/Users/jack/dev/gabwoo/출판_생산 진행 현황/docs/02-design/security-spec.md` (검증된 패턴) |
| 초기 관리자 | `zephirus03@gmail.com` (사용자 확정) |

---

## 0. 한눈에 보기 (Executive Summary)

| 영역 | 결정 |
|------|------|
| Supabase 프로젝트 | **Option A 채택 — 출판 프로젝트와 동일 Supabase 재사용** (`btbqzbrtsmwoolurpqgx`) |
| 사용자 테이블 | `approved_users`, `signup_requests` **공유** (출판/패키지 모두 동일 계정으로 로그인) |
| 패키지 전용 테이블 | 없음. 데이터는 Storage(`package-data` 버킷 또는 `gabwoo-data/package/` 경로)에 `data.json`으로 저장 |
| 권한 강제 | 프론트엔드는 UI 표시 용도, 실제 권한은 **Supabase RLS/Storage Policy**에서 강제 |
| 관리자 판별 | DB `approved_users.is_admin` 플래그 + SECURITY DEFINER 함수 `public.is_admin_user()` |
| 비밀번호 정책 | 최소 8자 (출판의 6자 대비 강화) |
| 초기 관리자 보호 | `zephirus03@gmail.com`의 `is_admin`은 트리거로 `false` 변경 차단 |

이 스펙은 출판 프로젝트의 검증된 패턴을 그대로 재사용하면서, 해당 리뷰에서 드러난 **5가지 갭**을 함께 보완합니다 (섹션 12 참고).

---

## 1. Supabase 프로젝트 전략 — Option A vs Option B

### Option A: 기존 Supabase 프로젝트 재사용 (채택)

기존 출판 프로젝트의 Supabase (`btbqzbrtsmwoolurpqgx.supabase.co`)를 재사용.
- `approved_users`, `signup_requests` 테이블은 **공유** (SSO 효과)
- 패키지 데이터는 새로운 Storage 버킷 `package-data` (또는 기존 `gabwoo-data` 버킷에 `package/` 폴더)

### Option B: 패키지 전용 신규 Supabase 프로젝트

완전히 별도 프로젝트 생성. 사용자는 출판/패키지 각각 가입/로그인해야 함.

### 비교

| 기준 | Option A (공유) | Option B (분리) |
|------|---------------|----------------|
| **사용자 경험** | 한 번 가입/승인 → 두 앱 모두 사용 | 두 번 가입, 두 번 승인 (번거로움) |
| **관리자 부담** | 승인 1회 | 승인 2회, 2개 대시보드 관리 |
| **비용** | 무료(Free Tier) 내에서 해결 가능 (2개 앱 합쳐도 500MB DB/1GB Storage 여유) | 무료 프로젝트 2개 (문제없음), 다만 대시보드 컨텍스트 전환 |
| **보안 블라스트 반경** | 한 프로젝트 사고 → 두 앱 모두 영향 | 격리 |
| **운영 복잡성** | RLS/Policy 일원화, anon key 1개 | RLS/Policy 중복, anon key 2개 |
| **데이터 격리** | 테이블명 prefix/버킷 분리로 충분 | 물리적 분리 (자연스러움) |
| **장기 진화** | 조직 차원 SSO 기반 마련 (ERP 연결 시 유리) | 앱마다 재구축 필요 |

### 권장: **Option A (공유)**

**이유:**
1. 갑우문화사는 **단일 조직, 단일 사용자 모집단**. 영업사원·생산팀장 대부분이 출판과 패키지를 함께 다룸 (프로젝트 개요: "2개 사업부" 구조). 한 사람이 두 계정을 관리하는 것은 실사용 마찰.
2. 이미 출판 프로젝트에서 `zephirus03@gmail.com`이 초기 관리자. 동일 관리자가 패키지도 운영하므로 대시보드 분리 실익 없음.
3. 자체 ERP 개발 중(CLAUDE.md: "ERP 자체개발 중")이라는 맥락에서, **사용자 신원 시스템을 하나로 통일**해두면 추후 ERP와의 SSO 연결도 용이.
4. Supabase 무료 티어의 여유가 충분. 정적 데이터 JSON 2개(출판+패키지) 합산 수 MB 수준.
5. 블라스트 반경 우려는 RLS를 엄격히 가져가면 실질적으로 낮음. 공유되는 것은 "사용자 신원"뿐이며, 앱별 데이터는 버킷/경로로 분리됨.

**채택 시 주의사항:**
- 앱별 권한이 달라지는 경우(예: 출판은 되는데 패키지는 안 되는 사용자)를 위해 추후 `approved_users`에 `apps` 배열 컬럼 또는 별도 `user_app_access(user_email, app)` 테이블을 추가할 수 있도록 설계. **v1에서는 승인되면 양쪽 모두 접근 가능**이 기본값.
- 신규 앱을 Storage에 추가할 때 Policy를 앱별로 분리해 둘 것 (섹션 6 참조).

---

## 2. 인증 흐름 (Auth Flow)

출판 프로젝트와 동일한 "관리자 사전 승인" 모델을 재사용합니다. 공개 가입이지만 승인 전에는 로그인 불가.

### 2.1 가입 흐름

```
[사용자] 가입 폼 입력 (email + password)
   ↓
[frontend] signup_requests 에 기존 신청 존재 여부 확인
   ↓ (없으면)
[Supabase Auth] auth.signUp({ email, password })  → Auth 사용자 생성
   ↓
[frontend] signup_requests.insert({ email, status: 'pending' })
   ↓
[frontend] auth.signOut()   ← 즉시 로그아웃 (승인 전 사용 불가 강제)
   ↓
[UI] "가입 신청 완료. 관리자 승인 후 로그인 가능" 메시지
```

### 2.2 로그인 흐름

```
[사용자] 이메일/비밀번호 입력
   ↓
[Supabase Auth] signInWithPassword → JWT 발급
   ↓ (성공 시)
[frontend] approved_users 에서 본인 이메일 조회 (RLS: self_read)
   ↓
  ├─ 존재 O → enterApp()  → 패키지 대시보드 진입 + data.json 로드
  └─ 존재 X → auth.signOut() + "관리자 승인 대기 중" 메시지
```

### 2.3 세션 유지

- Supabase JS SDK의 기본값 `persistSession: true` 사용 → `localStorage`에 `sb-<project>-auth-token` 저장
- 앱 초기화 시 `sb.auth.getSession()`으로 복원 → 이미 로그인된 세션이면 자동으로 `enterApp()`
- **"로그인 유지" 체크박스**: 출판 프로젝트 커밋 `680e944` 참고. 체크 해제 시 `storage: sessionStorage`로 클라이언트 재설정
- 로그아웃 버튼: 헤더 우측에 배치 (`handleLogout()` → `auth.signOut()` + 화면 전환)

### 2.4 초기 관리자

- `zephirus03@gmail.com`을 **DB 최초 INSERT 및 트리거로 보호** (섹션 7의 migration SQL)
- 프론트엔드에서의 하드코딩은 UI 표시 용도로만 사용 (관리자 섹션 노출 여부)
- **실제 권한 enforcement는 RLS의 `public.is_admin_user()` 함수**가 담당

---

## 3. 데이터베이스 스키마 (공유 테이블)

출판 프로젝트가 이미 운영 중인 테이블을 **그대로 재사용**. 패키지 전용 테이블은 추가하지 않습니다 (정적 JSON 저장소 기반).

### 3.1 `approved_users`

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `email` | `text` | PRIMARY KEY | 사용자 이메일 (Auth와 1:1 매칭) |
| `approved_at` | `timestamptz` | DEFAULT `now()` | 승인 시각 |
| `is_admin` | `boolean` | DEFAULT `false`, NOT NULL | 관리자 여부 |

### 3.2 `signup_requests`

| 컬럼 | 타입 | 제약 | 설명 |
|------|------|------|------|
| `email` | `text` | PRIMARY KEY | 신청자 이메일 |
| `status` | `text` | NOT NULL, CHECK (`status` IN ('pending','approved','rejected')), DEFAULT 'pending' | 처리 상태 |
| `requested_at` | `timestamptz` | DEFAULT `now()` | 신청 시각 |

### 3.3 설계 메모

- **`approved_users`가 signup 흐름의 단일 권한 소스**. `signup_requests.status`는 감사/이력용.
- **`pending_signups` 별도 테이블은 만들지 않음** — `signup_requests`가 그 역할을 겸함. 새로 만들면 출판 프로젝트와 스키마가 달라져서 공유 효익 훼손.
- Supabase `auth.users`와 `approved_users.email`는 애플리케이션 레벨 1:1 관계. DB FK는 걸지 않음 (Supabase auth 스키마 교차 참조 복잡도 회피).

---

## 4. RLS 정책 (완성판 SQL)

출판 프로젝트의 정책을 기반으로, **섹션 12에서 식별한 갭**(초기 관리자 이외 사용자가 `is_admin=true`가 되어도 실제 권한이 안 붙는 버그)을 수정한 버전입니다.

### 4.1 권한 판별 함수 (핵심)

```sql
-- =========================================
-- SECURITY DEFINER 함수: RLS 재귀 없이 관리자 여부 조회
-- =========================================
CREATE OR REPLACE FUNCTION public.is_admin_user()
RETURNS boolean
LANGUAGE sql
SECURITY DEFINER
STABLE
AS $$
  SELECT COALESCE(
    (SELECT is_admin
       FROM public.approved_users
      WHERE email = auth.jwt() ->> 'email'),
    false
  );
$$;

-- 함수는 authenticated 역할에서 호출 가능해야 함
REVOKE ALL ON FUNCTION public.is_admin_user() FROM PUBLIC;
GRANT EXECUTE ON FUNCTION public.is_admin_user() TO authenticated;
```

**왜 SECURITY DEFINER인가?** 일반 RLS 정책에서 `approved_users`를 참조하면 그 참조 자체가 다시 RLS를 타서 `self_read` 정책(=내 행만 보임)에 걸립니다. 다른 사용자의 `is_admin`을 조회하는 관리자 정책이 작동하지 않게 됨. 함수를 `SECURITY DEFINER`로 정의하면 RLS를 우회하고 오너 권한(보통 `postgres`)으로 읽어, 재귀 없이 내 `is_admin`을 확인할 수 있습니다.

### 4.2 RLS 활성화

```sql
ALTER TABLE public.approved_users  ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.signup_requests ENABLE ROW LEVEL SECURITY;
```

### 4.3 `approved_users` 정책

```sql
-- 기존 정책 정리
DROP POLICY IF EXISTS "self_read_approved"        ON public.approved_users;
DROP POLICY IF EXISTS "admin_insert_approved"     ON public.approved_users;
DROP POLICY IF EXISTS "admin_update_approved"     ON public.approved_users;
DROP POLICY IF EXISTS "admin_delete_approved"     ON public.approved_users;
DROP POLICY IF EXISTS "admin_read_all_approved"   ON public.approved_users;

-- SELECT: 본인 레코드 or 관리자는 전체
CREATE POLICY "approved_users_select"
  ON public.approved_users
  FOR SELECT
  USING (
    email = auth.jwt() ->> 'email'
    OR public.is_admin_user()
  );

-- INSERT: 관리자만 (승인 처리)
CREATE POLICY "approved_users_insert"
  ON public.approved_users
  FOR INSERT
  WITH CHECK (public.is_admin_user());

-- UPDATE: 관리자만 (is_admin 토글 등)
CREATE POLICY "approved_users_update"
  ON public.approved_users
  FOR UPDATE
  USING (public.is_admin_user())
  WITH CHECK (public.is_admin_user());

-- DELETE: 관리자만
CREATE POLICY "approved_users_delete"
  ON public.approved_users
  FOR DELETE
  USING (public.is_admin_user());
```

### 4.4 `signup_requests` 정책

```sql
-- 기존 정책 정리
DROP POLICY IF EXISTS "self_insert_signup"        ON public.signup_requests;
DROP POLICY IF EXISTS "self_or_admin_read_signup" ON public.signup_requests;
DROP POLICY IF EXISTS "admin_update_signup"       ON public.signup_requests;
DROP POLICY IF EXISTS "admin_delete_signup"       ON public.signup_requests;
DROP POLICY IF EXISTS "public_self_read_signup"   ON public.signup_requests;

-- INSERT: 본인 이메일로만 가입 신청 가능 (가입 직후 JWT 상태)
CREATE POLICY "signup_requests_insert"
  ON public.signup_requests
  FOR INSERT
  WITH CHECK (email = auth.jwt() ->> 'email');

-- SELECT: 본인 신청 확인 or 관리자는 전체 (승인 대기 목록)
CREATE POLICY "signup_requests_select"
  ON public.signup_requests
  FOR SELECT
  USING (
    email = auth.jwt() ->> 'email'
    OR public.is_admin_user()
  );

-- UPDATE: 관리자만 (status: pending → approved/rejected)
CREATE POLICY "signup_requests_update"
  ON public.signup_requests
  FOR UPDATE
  USING (public.is_admin_user())
  WITH CHECK (public.is_admin_user());

-- DELETE: 관리자만
CREATE POLICY "signup_requests_delete"
  ON public.signup_requests
  FOR DELETE
  USING (public.is_admin_user());
```

### 4.5 초기 관리자 강등 방지 트리거

출판 스펙에서 빠진 보호 장치. 초기 관리자(`zephirus03@gmail.com`)의 `is_admin`을 실수로 해제하면 관리자 0명이 되어 시스템이 잠기는 사고를 방지합니다.

```sql
CREATE OR REPLACE FUNCTION public.protect_initial_admin()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  -- 초기 관리자 is_admin 강등 방지
  IF OLD.email = 'zephirus03@gmail.com'
     AND OLD.is_admin = true
     AND NEW.is_admin = false THEN
    RAISE EXCEPTION '초기 관리자의 권한은 해제할 수 없습니다';
  END IF;

  -- 초기 관리자 행 삭제 방지는 별도 DELETE 트리거에서 처리
  RETURN NEW;
END;
$$;

DROP TRIGGER IF EXISTS trg_protect_initial_admin_update ON public.approved_users;
CREATE TRIGGER trg_protect_initial_admin_update
  BEFORE UPDATE ON public.approved_users
  FOR EACH ROW
  EXECUTE FUNCTION public.protect_initial_admin();

CREATE OR REPLACE FUNCTION public.protect_initial_admin_delete()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  IF OLD.email = 'zephirus03@gmail.com' THEN
    RAISE EXCEPTION '초기 관리자 계정은 삭제할 수 없습니다';
  END IF;
  RETURN OLD;
END;
$$;

DROP TRIGGER IF EXISTS trg_protect_initial_admin_delete ON public.approved_users;
CREATE TRIGGER trg_protect_initial_admin_delete
  BEFORE DELETE ON public.approved_users
  FOR EACH ROW
  EXECUTE FUNCTION public.protect_initial_admin_delete();
```

---

## 5. 관리자 패널 로직 (설계 수준)

> 구현 코드는 이 문서의 범위 외. 의사코드와 UI 요구사항만 명시.

### 5.1 진입 조건

```
enterApp() 호출 시:
  isAdmin = await public.is_admin_user() 를 RPC로 호출
  if (isAdmin) → #adminSection 표시
               → loadPendingRequests()
               → loadApprovedUsers()
```

실제로는 출판 프로젝트처럼 `approved_users.select('is_admin').eq('email', me).single()`로도 동일 효과를 얻을 수 있으나, **is_admin_user() RPC 호출을 권장** (프론트엔드에서 한 가지 방법으로 통일 + SELECT 정책 변경 영향 없음).

### 5.2 승인 대기 목록 (`loadPendingRequests`)

- 조회: `signup_requests.select('*').eq('status','pending').order('requested_at', desc)`
- 각 행마다 **승인** / **거절** 버튼
- 승인 클릭:
  1. `signup_requests.update({ status: 'approved' }).eq('email', X)`
  2. `approved_users.insert({ email: X, is_admin: false })`
  3. 목록 새로고침
- 거절 클릭:
  1. `signup_requests.update({ status: 'rejected' }).eq('email', X)`
  2. 목록 새로고침

### 5.3 승인 사용자 목록 (`loadApprovedUsers`)

- 조회: `approved_users.select('*').order('approved_at', desc)`
- 각 행에 관리자 배지 표시
- 본인 행(`email === myEmail`)에는 "나" 표시, 버튼 없음 (자기 자신 강등 금지)
- 초기 관리자(`zephirus03@gmail.com`) 행에는 버튼 자체를 렌더링하지 않음 (프론트 가드) — 서버 측 트리거가 최종 방어선
- 기타 사용자는 "관리자 지정" / "관리자 해제" 토글:
  - `approved_users.update({ is_admin: true|false }).eq('email', X)`
  - 에러 응답 시 트리거 메시지를 그대로 노출하지 않고 "권한 변경 실패" 일반 메시지로 표시

### 5.4 UI 복구

출판 프로젝트 `index.html` 라인 ~1115–1213 의 관리자 패널 구조를 그대로 따릅니다. 패키지 화면에서는 엑셀 업로드 UI는 필요 시 추가하되 v1 범위에 없다면 생략 (plan.md "수동 변환" 방향과 일치).

---

## 6. Storage 및 데이터 파일 전략

### 6.1 구조 결정

패키지 `data.json`의 저장 위치는 두 가지 옵션:

| 방안 | 설명 | 권장도 |
|------|------|--------|
| **A. 신규 버킷 `package-data`** | 버킷 단위 Policy 분리, 명확한 격리 | ✅ 권장 |
| B. 기존 `gabwoo-data` 버킷에 `package/data.json` | Storage 1개로 통합, 파일 경로로 구분 | 대안 |

**권장: A. 신규 버킷 `package-data` 생성** — Policy 관리가 단순하고, 실수로 출판 `data.json`을 덮어쓸 위험 없음.

### 6.2 Storage 정책 (패키지 전용 버킷)

```sql
-- =========================================
-- Storage: package-data 버킷 정책
-- =========================================

-- 다운로드 (SELECT): 승인된 사용자만
CREATE POLICY "package_data_download"
  ON storage.objects
  FOR SELECT
  USING (
    bucket_id = 'package-data'
    AND EXISTS (
      SELECT 1 FROM public.approved_users
      WHERE email = auth.jwt() ->> 'email'
    )
  );

-- 업로드 (INSERT): 관리자만
CREATE POLICY "package_data_upload"
  ON storage.objects
  FOR INSERT
  WITH CHECK (
    bucket_id = 'package-data'
    AND public.is_admin_user()
  );

-- 덮어쓰기 (UPDATE): 관리자만
CREATE POLICY "package_data_update"
  ON storage.objects
  FOR UPDATE
  USING (
    bucket_id = 'package-data'
    AND public.is_admin_user()
  );

-- 삭제 (DELETE): 관리자만 (v1에서는 불필요하지만 방어 심층)
CREATE POLICY "package_data_delete"
  ON storage.objects
  FOR DELETE
  USING (
    bucket_id = 'package-data'
    AND public.is_admin_user()
  );
```

### 6.3 버킷 생성 설정

- 버킷 이름: `package-data`
- Public: **OFF** (private)
- File size limit: 10 MB (data.json 예상 1.5MB, 여유 포함)
- Allowed MIME types: `application/json`

### 6.4 v1의 수동 변환 시나리오

plan.md 기준 v1은 **로컬에서 `python3 convert.py` 실행 → `web/data.json` 생성 → git push → Vercel 배포** 흐름.
이 경우 Storage가 아니라 Vercel 정적 자산으로 배포되는 `data.json`이 실제 사용되므로, Storage Policy는 **v2 자동화 또는 웹 업로드를 위해 미리 준비해두는 선언적 기반**에 가깝습니다.

⚠️ **중요**: v1이 정적 자산으로 `data.json`을 배포하면, **data.json은 공개 URL로 노출**됩니다. 로그인을 우회하여 URL만 알면 누구나 읽을 수 있음.

**v1에서 이 문제를 어떻게 처리할 것인가** — 두 가지 옵션:

| 옵션 | 설명 | 특징 |
|------|------|------|
| **V1a. Vercel 정적 `data.json` + 로그인 게이트는 UI만** | 가장 간단, 출판 v1과 동일. 보안은 "URL 난독화" 수준. | 내부용 MVP로 수용 가능하지만 진짜 보호는 아님 |
| **V1b. data.json을 Storage에 두고 frontend에서 `sb.storage.download`로 로드** | 승인된 사용자만 JSON 접근 가능 (RLS). | 약간의 수작업 추가 (`python3 convert.py && supabase CLI로 업로드`), 실질적 보안 획득 |

**권장: V1b.** 출판 프로젝트가 이미 이 패턴을 운영 중(`web_deploy/index.html:671` — `sb.storage.from('gabwoo-data').download('data.json')`)이므로, 패키지도 동일하게 맞추면 코드/운영 일관성 + 실질적 접근 제어 획득. Plan.md의 "수동 변환" 범위를 유지하면서 `convert.py`가 로컬 파일 생성 후 한 줄 업로드 단계만 추가됩니다.

만약 V1a를 선택한다면 문서에 "로그인은 UI 게이트일 뿐 data.json 자체는 URL만 알면 조회 가능"이라고 **명시적으로 기록**하고, 영업/생산 내부 유출 리스크가 허용되는지 사용자 확인을 받을 것.

---

## 7. Threat Model

OWASP Top 10 2021 기준, 정적 SPA + Supabase 아키텍처에 해당하는 위협만 선별.

### T1. 무권한 데이터 접근 (A01 Broken Access Control)

| 항목 | 내용 |
|------|------|
| 시나리오 | 브라우저 콘솔에서 직접 `sb.from('approved_users').insert(...)` 호출로 자가 승인 |
| 영향 | Critical — 전체 생산 데이터 노출 |
| 완화 | RLS 정책이 `is_admin_user() = true`가 아니면 INSERT 차단. 섹션 4 참조. |

### T2. SQL/NoSQL Injection (A03 Injection)

| 항목 | 내용 |
|------|------|
| 시나리오 | 검색 입력을 DB 쿼리에 직접 interpolate |
| 영향 | Low — Supabase JS SDK는 파라미터 바인딩. 클라이언트 측 검색은 메모리 내 JS 필터링이라 DB 관계 없음 |
| 완화 | SDK 호출 시 `.eq('email', userInput)` 형태 유지. 문자열 concat 금지. |

### T3. Cross-Site Scripting (A03 / 전통적 XSS)

| 항목 | 내용 |
|------|------|
| 시나리오 | 관리자 패널에서 이메일 문자열 렌더링 시 HTML 주입, 검색 결과 카드의 거래처/품명 렌더링 |
| 영향 | High — 관리자 세션에서 XSS 발화 시 권한 남용 |
| 완화 | ① 모든 외부 입력/DB 값은 `esc()`로 이스케이핑. ② `esc()`는 `&`, `<`, `>`, `"`, `'`, `` ` `` 모두 이스케이핑 (출판 프로젝트의 누락 보완). ③ `onclick="handleApprove('...')"` 형태의 인라인 핸들러 대신 `data-email` 속성 + `addEventListener`로 전환 권장 (v1에서는 `esc()` 강화로 타협 가능). ④ CSP 헤더로 인라인 스크립트 범위 제한 (섹션 9). |

개선된 `esc()` 참조 구현:

```javascript
function esc(s) {
  if (s == null) return '';
  return String(s)
    .replace(/&/g, '&amp;')
    .replace(/</g, '&lt;')
    .replace(/>/g, '&gt;')
    .replace(/"/g, '&quot;')
    .replace(/'/g, '&#x27;')
    .replace(/`/g, '&#x60;');
}
```

### T4. CSRF (A01)

| 항목 | 내용 |
|------|------|
| 시나리오 | 악성 사이트가 사용자 세션을 탈취해 API 호출 |
| 영향 | Low |
| 완화 | Supabase JS SDK는 Bearer 토큰 기반 (Authorization 헤더). 쿠키 기반이 아니므로 전통적 CSRF 취약 아님. 추가 조치 불필요. |

### T5. 세션 하이재킹 (A07)

| 항목 | 내용 |
|------|------|
| 시나리오 | `localStorage`의 Supabase 토큰 탈취 (XSS 경유) |
| 영향 | High — 탈취 시 사용자 권한 그대로 획득 |
| 완화 | ① XSS 완화(T3)가 1차 방어선. ② Supabase Dashboard > Authentication > Settings에서 JWT 만료 시간을 적정 값(예: 1시간)으로 유지. ③ 프로덕션은 HTTPS 강제 (Vercel 기본). ④ 민감 동작(관리자 액션) 전 재인증 고려 — v2 범위. |

### T6. Credential Stuffing / 약한 비밀번호 (A07)

| 항목 | 내용 |
|------|------|
| 시나리오 | 유출 비밀번호 데이터베이스로 자동화된 로그인 시도 |
| 영향 | Medium — 승인 전 사용자면 Auth만 통과, `approved_users` 미등록으로 앱 진입 차단. 승인 사용자라면 전체 데이터 접근 가능 |
| 완화 | ① 비밀번호 최소 **8자** (출판 6자 → 강화). Supabase Dashboard > Authentication > Settings에서 Minimum password length = 8. ② Rate limit: Supabase는 기본 IP당 로그인 시도 제한을 적용함 (무료 티어에서도 제공). ③ 장기: MFA 도입 고려 (v2). |

### T7. 가입 스팸 / 봇 (A04)

| 항목 | 내용 |
|------|------|
| 시나리오 | 봇이 대량으로 가입 시도 → signup_requests 테이블 오염 |
| 영향 | Low — 승인 모델이라 실제 접근은 불가. 다만 관리자 알림 피로 |
| 완화 | Supabase Auth 기본 rate limit + 관리자 거절 버튼. v2에서 CAPTCHA 고려. |

### T8. 에러 메시지 정보 유출 (A04/A09)

| 항목 | 내용 |
|------|------|
| 시나리오 | Supabase 에러 `res.error.message`를 그대로 UI에 노출 → 내부 스키마, 존재 여부 누설 |
| 영향 | Low-Medium |
| 완화 | 프론트엔드는 일반화된 메시지만 표시. 상세는 `console.error()`로만. 예시: <br>`showError('가입에 실패했습니다. 잠시 후 다시 시도해주세요.')` |

### T9. SSRF (A10)

| 항목 | 내용 |
|------|------|
| 해당 없음 | 정적 SPA, 서버 사이드 요청 없음 |

### T10. 취약한 의존성 (A06)

| 항목 | 내용 |
|------|------|
| 시나리오 | CDN의 `supabase-js`, `xlsx.full.min.js` 버전이 취약 |
| 영향 | Medium |
| 완화 | ① 버전을 명시적으로 고정 (`@supabase/supabase-js@2`는 major pinning, 가능하면 minor까지). ② SRI(Subresource Integrity) 해시 추가 권장 (v1 후속). ③ 월 1회 CDN 라이브러리 취약성 체크 (`npm audit` 또는 Snyk). |

---

## 8. Secrets 관리

### 8.1 공개(Public) — 저장소 커밋 허용

| 항목 | 값 | 이유 |
|------|-----|------|
| `SUPABASE_URL` | `https://btbqzbrtsmwoolurpqgx.supabase.co` | 브라우저에서 사용, 공개 설계 |
| `SUPABASE_ANON_KEY` | anon 역할 JWT | **RLS로 보호되는 전제 하에** 공개 가능. Supabase 공식 설계 |

프론트엔드 `index.html`의 `<script>` 블록에 하드코딩 가능. 출판 프로젝트와 동일.

### 8.2 절대 비공개(Secret) — 커밋 금지

| 항목 | 유출 시 영향 |
|------|------------|
| `SUPABASE_SERVICE_ROLE_KEY` | RLS 무시, DB 전체 읽기/쓰기 — **즉시 키 로테이션 필요** |
| Supabase DB Password | DB 직접 접근 |
| Personal Access Token | Supabase Dashboard API 접근 |

**본 프로젝트는 service_role 키를 사용하지 않습니다.** 정적 SPA는 anon 키만 사용. service_role이 필요한 상황(예: convert.py가 Storage에 업로드)에서는:
- 로컬 개발자 머신의 환경 변수로만 보관 (`.env.local`)
- `.gitignore`에 `.env*` 추가
- CI/CD에서 사용할 경우 GitHub Actions Secrets 또는 Vercel Environment Variables(서버 전용)에만 저장

### 8.3 `.gitignore` 권장

```gitignore
# Secrets
.env
.env.*
!.env.example
*.pem
*.key

# Supabase CLI
.supabase/

# 원본 엑셀 (CLAUDE.md 규칙: 수정 금지이지만 커밋에서 제외하여 용량 관리)
# 필요 시 주석 해제
# 생산진행현황/*.xlsx

# 빌드 아티팩트
node_modules/
.vercel/
```

### 8.4 키 로테이션 체크리스트

anon 키 로테이션은 일반적으로 불필요하지만, service_role 키나 DB 비밀번호 유출 시:

1. Supabase Dashboard > Settings > API에서 키 재발급
2. 영향받는 환경 변수 업데이트 (로컬, Vercel, CI)
3. 이전 키를 사용 중인 배포 중단 확인
4. 유출 원인 파악 및 재발 방지

---

## 9. 보안 헤더 (Vercel 배포)

`vercel.json` 권장 설정. 출판 프로젝트 I3 권고를 패키지 도메인에 맞춰 수정.

```json
{
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Strict-Transport-Security", "value": "max-age=63072000; includeSubDomains" },
        { "key": "Permissions-Policy", "value": "camera=(), microphone=(), geolocation=()" },
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline'; connect-src 'self' https://btbqzbrtsmwoolurpqgx.supabase.co wss://btbqzbrtsmwoolurpqgx.supabase.co; img-src 'self' data:; font-src 'self' data:; frame-ancestors 'none'; base-uri 'self'; form-action 'self'"
        }
      ]
    }
  ]
}
```

**주의점:**
- `X-XSS-Protection`은 deprecated. 제외.
- `'unsafe-inline'` (script/style)는 현재 인라인 스크립트·스타일 사용 때문에 필요. 장기적으로는 해시 기반 CSP로 전환.
- Supabase Realtime 사용 가능성을 위해 `wss:` 포함.
- `frame-ancestors 'none'` 은 `X-Frame-Options: DENY`와 중복이지만 현대 브라우저에서 우선 적용됨.

---

## 10. 마이그레이션 / 셋업 실행 순서

> **인프라 담당자(Jack) 실행용 체크리스트**. 각 단계는 Supabase Dashboard와 로컬 저장소에서 번갈아 수행.

### Step 0. 사전 점검

- [ ] 출판 프로젝트 Supabase (`btbqzbrtsmwoolurpqgx`)의 현재 `approved_users`, `signup_requests` 테이블 백업 (Dashboard > Database > Backups 확인)
- [ ] 출판 프로젝트가 현재 사용 중인 RLS 정책을 Dashboard에서 확인하여 변경 영향 평가
- [ ] 변경 작업 중 출판 앱 일시 중단 필요 여부 판단 (정책 교체 순간만 짧게, 섹션 4.3/4.4의 DROP+CREATE는 원자성 확보를 위해 트랜잭션으로 감쌀 것)

### Step 1. SQL 마이그레이션 실행 (Supabase Dashboard > SQL Editor)

아래 SQL을 **트랜잭션으로 한 번에** 실행. 실패 시 전체 롤백:

```sql
BEGIN;

-- ─────────────────────────────────────────
-- 1. SECURITY DEFINER 관리자 판별 함수
-- ─────────────────────────────────────────
CREATE OR REPLACE FUNCTION public.is_admin_user()
RETURNS boolean
LANGUAGE sql
SECURITY DEFINER
STABLE
AS $$
  SELECT COALESCE(
    (SELECT is_admin
       FROM public.approved_users
      WHERE email = auth.jwt() ->> 'email'),
    false
  );
$$;

REVOKE ALL ON FUNCTION public.is_admin_user() FROM PUBLIC;
GRANT EXECUTE ON FUNCTION public.is_admin_user() TO authenticated;

-- ─────────────────────────────────────────
-- 2. 테이블 보장 (이미 존재하면 NOP)
-- ─────────────────────────────────────────
CREATE TABLE IF NOT EXISTS public.approved_users (
  email       text PRIMARY KEY,
  approved_at timestamptz NOT NULL DEFAULT now(),
  is_admin    boolean     NOT NULL DEFAULT false
);

CREATE TABLE IF NOT EXISTS public.signup_requests (
  email        text PRIMARY KEY,
  status       text NOT NULL DEFAULT 'pending'
                 CHECK (status IN ('pending','approved','rejected')),
  requested_at timestamptz NOT NULL DEFAULT now()
);

-- ─────────────────────────────────────────
-- 3. RLS 활성화
-- ─────────────────────────────────────────
ALTER TABLE public.approved_users  ENABLE ROW LEVEL SECURITY;
ALTER TABLE public.signup_requests ENABLE ROW LEVEL SECURITY;

-- ─────────────────────────────────────────
-- 4. 기존 정책 정리 (이름 기반 idempotent)
-- ─────────────────────────────────────────
DROP POLICY IF EXISTS "self_read_approved"        ON public.approved_users;
DROP POLICY IF EXISTS "admin_insert_approved"     ON public.approved_users;
DROP POLICY IF EXISTS "approved_users_select"     ON public.approved_users;
DROP POLICY IF EXISTS "approved_users_insert"     ON public.approved_users;
DROP POLICY IF EXISTS "approved_users_update"     ON public.approved_users;
DROP POLICY IF EXISTS "approved_users_delete"     ON public.approved_users;

DROP POLICY IF EXISTS "self_insert_signup"        ON public.signup_requests;
DROP POLICY IF EXISTS "self_or_admin_read_signup" ON public.signup_requests;
DROP POLICY IF EXISTS "admin_update_signup"       ON public.signup_requests;
DROP POLICY IF EXISTS "admin_delete_signup"       ON public.signup_requests;
DROP POLICY IF EXISTS "signup_requests_insert"    ON public.signup_requests;
DROP POLICY IF EXISTS "signup_requests_select"    ON public.signup_requests;
DROP POLICY IF EXISTS "signup_requests_update"    ON public.signup_requests;
DROP POLICY IF EXISTS "signup_requests_delete"    ON public.signup_requests;

-- ─────────────────────────────────────────
-- 5. approved_users 신규 정책
-- ─────────────────────────────────────────
CREATE POLICY "approved_users_select"
  ON public.approved_users
  FOR SELECT
  USING (
    email = auth.jwt() ->> 'email'
    OR public.is_admin_user()
  );

CREATE POLICY "approved_users_insert"
  ON public.approved_users
  FOR INSERT
  WITH CHECK (public.is_admin_user());

CREATE POLICY "approved_users_update"
  ON public.approved_users
  FOR UPDATE
  USING (public.is_admin_user())
  WITH CHECK (public.is_admin_user());

CREATE POLICY "approved_users_delete"
  ON public.approved_users
  FOR DELETE
  USING (public.is_admin_user());

-- ─────────────────────────────────────────
-- 6. signup_requests 신규 정책
-- ─────────────────────────────────────────
CREATE POLICY "signup_requests_insert"
  ON public.signup_requests
  FOR INSERT
  WITH CHECK (email = auth.jwt() ->> 'email');

CREATE POLICY "signup_requests_select"
  ON public.signup_requests
  FOR SELECT
  USING (
    email = auth.jwt() ->> 'email'
    OR public.is_admin_user()
  );

CREATE POLICY "signup_requests_update"
  ON public.signup_requests
  FOR UPDATE
  USING (public.is_admin_user())
  WITH CHECK (public.is_admin_user());

CREATE POLICY "signup_requests_delete"
  ON public.signup_requests
  FOR DELETE
  USING (public.is_admin_user());

-- ─────────────────────────────────────────
-- 7. 초기 관리자 보호 트리거
-- ─────────────────────────────────────────
CREATE OR REPLACE FUNCTION public.protect_initial_admin()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  IF OLD.email = 'zephirus03@gmail.com'
     AND OLD.is_admin = true
     AND NEW.is_admin = false THEN
    RAISE EXCEPTION '초기 관리자의 권한은 해제할 수 없습니다';
  END IF;
  RETURN NEW;
END;
$$;

DROP TRIGGER IF EXISTS trg_protect_initial_admin_update ON public.approved_users;
CREATE TRIGGER trg_protect_initial_admin_update
  BEFORE UPDATE ON public.approved_users
  FOR EACH ROW
  EXECUTE FUNCTION public.protect_initial_admin();

CREATE OR REPLACE FUNCTION public.protect_initial_admin_delete()
RETURNS trigger LANGUAGE plpgsql AS $$
BEGIN
  IF OLD.email = 'zephirus03@gmail.com' THEN
    RAISE EXCEPTION '초기 관리자 계정은 삭제할 수 없습니다';
  END IF;
  RETURN OLD;
END;
$$;

DROP TRIGGER IF EXISTS trg_protect_initial_admin_delete ON public.approved_users;
CREATE TRIGGER trg_protect_initial_admin_delete
  BEFORE DELETE ON public.approved_users
  FOR EACH ROW
  EXECUTE FUNCTION public.protect_initial_admin_delete();

-- ─────────────────────────────────────────
-- 8. 초기 관리자 시드 (이미 존재하면 admin 플래그만 보강)
-- ─────────────────────────────────────────
INSERT INTO public.approved_users (email, is_admin, approved_at)
VALUES ('zephirus03@gmail.com', true, NOW())
ON CONFLICT (email) DO UPDATE
  SET is_admin = true;

COMMIT;
```

### Step 2. Storage 버킷 생성 (Dashboard > Storage)

- [ ] **신규 버킷 생성**: 이름 `package-data`, Public OFF, File size limit 10MB, MIME `application/json`
- [ ] 섹션 6.2의 Storage 정책 4개를 SQL Editor에서 실행

### Step 3. Auth 설정 (Dashboard > Authentication > Settings)

- [ ] Minimum password length: **8**
- [ ] Site URL: Vercel 배포 도메인 (예: `https://package-gabwoo.vercel.app`)
- [ ] Redirect URLs: 배포 도메인 추가
- [ ] Email confirmations: 비활성 (관리자 승인 모델이므로 이메일 인증은 생략 가능 — 출판과 일관)

### Step 4. 초기 관리자 계정 생성 (Auth 사용자 생성)

초기 관리자의 Supabase Auth 계정이 아직 없다면:

옵션 A) Dashboard > Authentication > Users > "Add user" 로 `zephirus03@gmail.com` + 비밀번호 직접 생성
옵션 B) 프론트엔드 가입 폼으로 자가 가입 후, SQL Editor에서 `signup_requests` 상태 강제 업데이트

Step 1의 시드 INSERT는 이미 `approved_users`에 행을 넣었으므로, Auth 계정만 있으면 로그인 즉시 관리자 권한 획득.

### Step 5. 프론트엔드 설정

- [ ] `web/index.html`에 Supabase 상수 하드코딩:
  ```
  SUPABASE_URL        = 'https://btbqzbrtsmwoolurpqgx.supabase.co'
  SUPABASE_ANON_KEY   = (출판 프로젝트와 동일 anon key)
  ```
- [ ] 출판 프로젝트의 로그인/가입/관리자 패널 UI 코드를 복사해 패키지 화면 구조에 맞춰 통합
- [ ] `esc()` 함수에 single-quote, backtick 이스케이핑 추가 (섹션 7 T3 참조)
- [ ] 비밀번호 검증 6 → 8자로 수정
- [ ] 에러 메시지 일반화

### Step 6. 검증

- [ ] 관리자(`zephirus03@gmail.com`)로 로그인 → 관리자 패널 보임
- [ ] 관리자로 `is_admin=false`인 테스트 사용자 승인 → 테스트 사용자 로그인 성공
- [ ] 테스트 사용자로 브라우저 콘솔에서 `sb.from('approved_users').insert({email:'x@x.com',is_admin:true})` 시도 → **RLS 거부 확인**
- [ ] 테스트 사용자로 `sb.from('approved_users').select('*')` → 본인 레코드만 반환 확인
- [ ] 초기 관리자 행에 대해 `update({is_admin:false})` 시도 → **트리거 에러 확인**
- [ ] 승인 전 가입 신청자가 로그인 시도 → "관리자 승인 대기 중" 메시지 + auto signOut
- [ ] 로그아웃 → 재방문 시 `localStorage` 유지 확인, 로그아웃 후에는 인증 화면 표시
- [ ] Storage `package-data/data.json` 다운로드: 승인 사용자 성공, 비승인/비로그인 실패

### Step 7. Vercel 배포

- [ ] `vercel.json`에 섹션 9의 보안 헤더 추가
- [ ] `.gitignore` 섹션 8.3 반영
- [ ] `git push` → Vercel 자동 배포
- [ ] 프로덕션 도메인에서 DevTools > Security 탭에서 HTTPS/TLS 확인
- [ ] `curl -I https://<domain>`로 응답 헤더에 보안 헤더들이 실제 적용되었는지 확인

---

## 11. OWASP Top 10 적용 체크리스트

| # | 카테고리 | 상태 | 근거 |
|---|----------|------|------|
| A01 | Broken Access Control | ✅ | RLS + Storage Policy로 서버 측 강제 |
| A02 | Cryptographic Failures | ✅ | HTTPS 강제(Vercel), JWT 만료 관리, service_role 보호 |
| A03 | Injection | ✅ | Supabase SDK 파라미터 바인딩, XSS는 `esc()` |
| A04 | Insecure Design | ✅ | 승인 기반 가입, 기본 deny RLS |
| A05 | Security Misconfiguration | ⚠️ | 보안 헤더 설정 필수(섹션 9), Storage public OFF |
| A06 | Vulnerable Components | ⚠️ | CDN 버전 고정 + SRI 권장, 월간 점검 |
| A07 | Identification & Auth Failures | ✅ | 최소 8자 비밀번호, Supabase rate limit, 관리자 승인 |
| A08 | Software & Data Integrity | ➖ | 정적 사이트, 외부 CDN은 SRI로 보완 (미구현 = 잔여 위험) |
| A09 | Logging & Monitoring | ⚠️ | Supabase Logs 기본 활용, 앱 레벨 감사 로그 없음 (v2 고려) |
| A10 | SSRF | ➖ | 해당 없음 (서버 사이드 요청 없음) |

---

## 12. 출판 프로젝트 대비 개선 사항 (갭 분석)

출판 프로젝트의 현재 상태를 점검한 결과, 아래 갭을 **패키지 스펙에 선제 반영**했습니다. 같은 Supabase를 공유하므로 패키지 마이그레이션을 통해 **출판 프로젝트에도 자동으로 갭이 메워지는** 부수 효과가 있습니다 (Jack에게 사전 공유 필요).

| # | 출판의 갭 | 패키지 스펙에서의 조치 |
|---|-----------|----------------------|
| **G1** | 관리자 판별이 `auth.jwt() ->> 'email' = 'zephirus03@gmail.com'` 하드코딩 — `is_admin=true` 플래그가 실제 RLS 권한을 부여하지 않음 | `public.is_admin_user()` SECURITY DEFINER 함수로 `is_admin` 플래그 기반 판별. 관리자 추가가 실제로 동작. (섹션 4.1) |
| **G2** | 초기 관리자 강등/삭제 시 안전장치 없음 — 실수로 시스템 잠금 가능 | `protect_initial_admin` 트리거 2종 추가. (섹션 4.5) |
| **G3** | `esc()`가 `'`, `` ` `` 이스케이핑 누락 | 6개 문자 모두 이스케이핑. (섹션 7 T3) |
| **G4** | 비밀번호 최소 6자 | 8자로 상향 + Supabase 대시보드 설정 동기화 (섹션 10 Step 3) |
| **G5** | 에러 메시지에 Supabase 내부 메시지 그대로 노출 | 일반화된 사용자 메시지 + `console.error`로만 상세 기록 (섹션 7 T8) |

**Jack에게 알림 필요**: 섹션 10의 SQL은 출판 프로젝트의 기존 정책을 **덮어쓰기** 합니다 (`DROP POLICY IF EXISTS`). 기존 출판 정책과 새 정책은 의미상 호환(더 엄격·더 정확)이지만, 실행 순간에 출판 앱이 5초 정도 장애를 겪을 수 있습니다. **점심시간 또는 저녁에 실행 권장**.

---

## 13. 잔여 위험 (Accepted Risks)

| ID | 위험 | 이유 | 재평가 시점 |
|----|------|------|------------|
| AR1 | 관리자 이메일 프론트 하드코딩 (UI용) | RLS가 실제 방어이므로 정보 노출 수준 | 관리자 이메일 변경 시 |
| AR2 | anon key 프론트 노출 | Supabase 공식 설계. RLS로 방어 | RLS 변경 시 |
| AR3 | 인라인 스크립트 + `'unsafe-inline'` CSP | v1 HTML 규모상 외부 JS 분리 비용 큼 | v2 리팩토링 시 해시 기반 CSP로 전환 |
| AR4 | CDN SRI 미적용 | v1 속도 우선. CDN 버전 고정으로 일부 방어 | 월간 점검에서 취약성 발견 시 즉시 적용 |
| AR5 | 앱 레벨 감사 로그 없음 | Supabase Postgres 로그로 대체 (무료 티어 retention 7일) | v2에서 audit_log 테이블 고려 |
| AR6 | MFA 미적용 | 사용자 규모 및 운영 복잡도 고려 v1 제외 | 승인 사용자 수 20명 초과 시 |

---

## 14. 부록 — 참고 파일 절대 경로

| 목적 | 경로 |
|------|------|
| 이 스펙의 구현 대상 | `/Users/jack/dev/gabwoo/패키지_생산진행현황/web/index.html` (생성 예정) |
| Plan 문서 | `/Users/jack/dev/gabwoo/패키지_생산진행현황/docs/01-plan/features/package-production-viewer.plan.md` |
| 출판 보안 스펙 (레퍼런스) | `/Users/jack/dev/gabwoo/출판_생산 진행 현황/docs/02-design/security-spec.md` |
| 출판 운영 코드 (레퍼런스) | `/Users/jack/dev/gabwoo/출판_생산 진행 현황/web_deploy/index.html` |
| 프로젝트 규칙 | `/Users/jack/dev/gabwoo/CLAUDE.md` |

---

**끝.** 이 스펙은 "출판에서 검증된 패턴 + 발견된 5개 갭 개선"을 최소 변경으로 패키지에 적용합니다. 다음 단계는 Design 단계에서의 convert.py/index.html 구조 설계, 그리고 이 스펙 섹션 10의 마이그레이션 실행입니다.
