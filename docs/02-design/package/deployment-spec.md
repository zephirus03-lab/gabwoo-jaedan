# Deployment Specification: 패키지 생산현황 조회 서비스

| 항목 | 내용 |
|---|---|
| Feature | package-production-viewer |
| 작성일 | 2026-04-08 |
| 작성자 | Infrastructure Architect Agent |
| 대상 | `web/` 정적 사이트 (HTML + JS + data.json) |
| 배포 플랫폼 | **Vercel (Free Tier)** |
| 레퍼런스 | `/Users/jack/dev/gabwoo/출판_생산 진행 현황/vercel.json` |
| 관련 문서 | `docs/02-design/security-spec.md` (RLS/Storage 정책), `docs/01-plan/features/package-production-viewer.plan.md` |

> ⚠️ **중요 전제**: 사용자가 본 프로젝트를 출판 프로젝트와 **단일 배포로 통합**하는 것을 검토 중입니다. 본 문서는 "독립 배포 가능한 최소 스펙"으로 작성되었으며, 통합이 확정되면 본 문서는 폐기되고 통합 배포 스펙으로 대체됩니다. 11장 참조.

---

## 1. Deployment Target

- **플랫폼**: Vercel Free Tier
- **배포 유형**: Static Site (빌드 없음, `web/` 디렉토리를 그대로 업로드)
- **리전**: Vercel 자동 Edge (전 세계 CDN, 별도 리전 지정 없음)
- **재사용 패턴**: 출판 프로젝트(`/Users/jack/dev/gabwoo/출판_생산 진행 현황/`)와 동일한 "정적 사이트 + 수동 변환" 패턴
- **서버 컴포넌트**: 없음 (Python `convert.py`는 로컬 전용, Vercel은 정적 파일만 서빙)

---

## 2. vercel.json 구성

출판 프로젝트의 `vercel.json`을 기반으로, security-spec.md의 보안 헤더 권장사항(I3)을 반영하여 확장합니다. 아래 내용을 리포지토리 루트의 `vercel.json`으로 사용:

```json
{
  "buildCommand": null,
  "outputDirectory": "web",
  "cleanUrls": true,
  "trailingSlash": false,
  "headers": [
    {
      "source": "/(.*)",
      "headers": [
        { "key": "X-Frame-Options", "value": "DENY" },
        { "key": "X-Content-Type-Options", "value": "nosniff" },
        { "key": "Referrer-Policy", "value": "strict-origin-when-cross-origin" },
        { "key": "Strict-Transport-Security", "value": "max-age=63072000; includeSubDomains; preload" },
        { "key": "X-XSS-Protection", "value": "1; mode=block" },
        {
          "key": "Content-Security-Policy",
          "value": "default-src 'self'; script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; style-src 'self' 'unsafe-inline'; connect-src 'self' https://btbqzbrtsmwoolurpqgx.supabase.co; img-src 'self' data:; font-src 'self' data:; frame-ancestors 'none'; base-uri 'self'; form-action 'self'"
        }
      ]
    },
    {
      "source": "/data.json",
      "headers": [
        { "key": "Cache-Control", "value": "no-cache, no-store, must-revalidate" }
      ]
    },
    {
      "source": "/index.html",
      "headers": [
        { "key": "Cache-Control", "value": "no-cache, no-store, must-revalidate" }
      ]
    },
    {
      "source": "/(.*)\\.(js|css|woff2|png|jpg|svg|ico)",
      "headers": [
        { "key": "Cache-Control", "value": "public, max-age=31536000, immutable" }
      ]
    }
  ]
}
```

### 설계 메모
- `buildCommand: null` — 빌드 단계 없음. Vercel은 `web/` 내용을 그대로 배포
- `cleanUrls: true` — `/index` 같은 확장자 없는 경로 허용 (MVP는 단일 페이지지만 향후 확장 대비)
- `CSP connect-src` — Supabase 프로젝트 도메인(`btbqzbrtsmwoolurpqgx.supabase.co`) 명시. 출판과 Supabase 공유(security-spec 섹션 1)
- `Cache-Control no-cache`가 `data.json`/`index.html`에 적용되어야 수동 변환 후 새 데이터가 즉시 반영됨
- 정적 에셋(js/css/이미지)은 파일명 해시 없이도 `immutable`로 설정 가능 (MVP는 번들러 미사용)

---

## 3. Build & Deploy Flow

```
[NAS: 3개 원본 엑셀]
   ↓ (수동 복사, 담당자 작업)
[Local: 생산진행현황/*.xlsx]
   ↓
[Local: cp web/data.json web/data.json.bak]   ← 롤백용 백업
   ↓
[Local: python3 convert.py]
   ↓
[Local: web/data.json 갱신]
   ↓
[Local: 샘플 10건 수동 대조 검증 (plan.md 리스크 R5)]
   ↓
[Local: git add web/data.json && git commit -m "data: YYYY-MM-DD update"]
   ↓
[Local: git push origin main]
   ↓
[GitHub: main 브랜치 업데이트 webhook]
   ↓
[Vercel: 자동 빌드 트리거 (빌드 스킵, 정적 업로드)]
   ↓
[Vercel Edge: 전 세계 CDN 배포 (30초 내 완료)]
   ↓
[모바일 브라우저: fetch('data.json') → 최신 데이터 반영]
```

### 배포 빈도 가정
- MVP: 담당자가 필요할 때 수동 갱신 (하루 1~3회 예상)
- 각 갱신 = 1커밋 = 1배포
- Vercel Free Tier는 무제한 배포 허용

---

## 4. Environment Variables

정적 사이트이므로 **서버 사이드 시크릿이 없습니다.** 모든 값은 클라이언트 JS에 직접 인라인됩니다(출판 프로젝트와 동일 패턴).

| 값 | 위치 | 공개 여부 | 비고 |
|---|---|---|---|
| `SUPABASE_URL` | `index.html` 내 `<script>` 변수 | 공개 | `https://btbqzbrtsmwoolurpqgx.supabase.co` |
| `SUPABASE_ANON_KEY` | `index.html` 내 `<script>` 변수 | 공개 | Supabase 설계상 공개 키. RLS가 실제 방어 (security-spec M1 참조) |
| `ADMIN_EMAIL` | `index.html` 상수 | 공개 | UI 표시 용도, 실제 권한은 RLS (security-spec H3) |

**Vercel Environment Variables 미사용.** 정적 사이트는 Vercel 환경변수를 참조할 방법이 없으며, 도입 시 빌드 단계 추가로 복잡도만 올라감.

**service_role 키는 절대 커밋하지 않음** (security-spec 섹션 8.2). 향후 convert.py가 Supabase Storage에 자동 업로드하게 되면 `.env.local`에 보관하고 `.gitignore`에 포함.

---

## 5. Git Repository Setup

### 5.1 원격 저장소
- **호스팅**: GitHub
- **저장소명 제안**: `gabwoo-package-production-viewer` (단, 통합 결정 시 기존 출판 리포에 흡수될 수 있음)
- **브랜치**: `main` 단일 브랜치
- **권한**: 단일 개발자 MVP → Branch Protection 규칙 불필요, PR 워크플로 생략

### 5.2 .gitignore

리포지토리 루트 `.gitignore`:

```gitignore
# 원본 엑셀 — 절대 커밋 금지 (CLAUDE.md 규칙 + 파일 용량/저작권)
생산진행현황/*.xlsx
생산진행현황/~$*.xlsx

# 변환 백업본
web/data.json.bak

# Secrets (향후 service_role 키 사용 시)
.env
.env.*
!.env.example
*.pem
*.key

# Supabase CLI
.supabase/

# Python
__pycache__/
*.pyc
*.pyo
.venv/
venv/

# OS / Editor
.DS_Store
.vscode/
.idea/
Thumbs.db

# Vercel
.vercel/
```

### 5.3 git remote 설정 (초기 1회)

```bash
cd /Users/jack/dev/gabwoo/패키지_생산진행현황
git init
git add .
git commit -m "chore: initial commit"
git branch -M main
git remote add origin git@github.com:<owner>/gabwoo-package-production-viewer.git
git push -u origin main
```

### 5.4 Vercel 연결
1. Vercel Dashboard → New Project → Import Git Repository
2. GitHub 저장소 선택
3. Framework Preset: **Other** (정적)
4. Root Directory: `.` (루트)
5. Build and Output Settings → `vercel.json`이 자동 적용됨
6. Deploy 버튼 → 첫 배포 확인

---

## 6. Domain & SSL

### MVP
- 무료 `.vercel.app` 서브도메인 사용 (예: `gabwoo-package.vercel.app`)
- Vercel 자동 TLS (Let's Encrypt, 자동 갱신)
- 별도 DNS 작업 불필요

### 향후 옵션
- **Option 1**: 독자 커스텀 도메인 (예: `package.gabwoo.co.kr`) — 도메인 보유 시 Vercel에서 DNS 레코드 1개 추가로 연결
- **Option 2**: 기존 출판 도메인 하위 경로 (예: `gabwoo.vercel.app/package`) — **통합 배포 결정 시** (11장 참조)
- **Option 3**: 독자 Vercel 서브도메인 유지 — 가장 단순, MVP 권장

---

## 7. Monitoring (최소 수준)

| 항목 | 도구 | 비용 |
|---|---|---|
| 페이지뷰/트래픽 | Vercel Analytics (Free Tier) | $0 |
| 배포 성공/실패 | Vercel Dashboard + GitHub Actions 알림 | $0 |
| 인증/승인 지표 | Supabase Dashboard > Authentication | $0 |
| Storage 사용량 | Supabase Dashboard > Storage | $0 |
| 에러 로깅 | **MVP 범위 제외** — `console.error` + security-spec M3 일반 에러 메시지로 충분 |

### Vercel Analytics 활성화 방법
1. Vercel Dashboard > Project > Analytics 탭
2. "Enable Web Analytics" 클릭 (Free Tier는 월 2,500 이벤트 포함)
3. `index.html` 별도 스크립트 주입 불필요 (Vercel이 자동 주입)

### 모니터링 관찰 지표 (운영 후)
- 일별 순사용자수 (영업/생산 직원 20~40명 예상)
- 배포 실패율 (0에 가까워야 함)
- Supabase `signup_requests` pending 개수 (관리자 알림 트리거)

---

## 8. Rollback Strategy

### 8.1 코드/설정 롤백 (index.html, vercel.json 등)
```bash
git revert HEAD        # 직전 커밋만 되돌리기
git push origin main   # Vercel 자동 재배포 (~30초)
```
또는 **Vercel Dashboard > Deployments**에서 이전 배포를 "Promote to Production" 클릭 (즉시 복구, DNS 전파 없음).

### 8.2 데이터 롤백 (data.json 오류 시)
```bash
cp web/data.json.bak web/data.json   # 로컬 백업 복원
git add web/data.json
git commit -m "data: rollback to previous version"
git push origin main
```

### 8.3 재해 시나리오 대응

| 상황 | 조치 |
|---|---|
| convert.py가 잘못된 data.json 생성 | 백업(.bak) 복원 → push |
| index.html 버그로 화이트스크린 | Vercel Dashboard에서 직전 배포 Promote |
| Supabase Auth 장애 | 복구 대기 (Supabase Status Page 확인) |
| GitHub 장애 | Vercel CLI(`vercel deploy --prod`)로 수동 배포 가능 |

### 8.4 백업 정책
- `web/data.json.bak`: 매 `convert.py` 실행 전 자동 생성 (convert.py 또는 Makefile에 포함)
- Git history 자체가 data.json의 완전한 버전 관리 (무제한 과거 복원 가능)
- Supabase DB/Storage: Supabase 자체 일일 백업(Free Tier 7일 보존)

---

## 9. Performance Targets

| 지표 | 목표 | 근거 |
|---|---|---|
| 초기 로드 시간 (4G) | < 2초 | 영업 현장 모바일 사용, 3초 내 정보 도달 목표 (plan.md Q3) |
| `data.json` 크기 | 500KB ~ 1.5MB | plan.md 섹션 4.5 예상치 (연 1,000건 기준) |
| `index.html` + JS | < 200KB | 번들러 없음, CDN 라이브러리 최소 사용 |
| LCP (Largest Contentful Paint) | < 2.5초 | Vercel Edge CDN + gzip 자동 |
| Vercel 월 대역폭 사용 | < 1GB | 내부 사용 20~40명, Free Tier 100GB의 1% 수준 |

### 최적화 포인트
- `data.json`은 gzip 압축되어 전송됨 (Vercel 자동) — 1.5MB JSON → 실제 전송 ~300KB 예상
- 정적 에셋 `immutable` 캐시로 재방문 로드 시간 거의 0
- 모바일 뷰포트 우선 HTML/CSS (plan.md 섹션 4.2)

---

## 10. Cost

| 항목 | 서비스 | 월 비용 |
|---|---|---|
| 호스팅 & CDN | Vercel Hobby (Free) | $0 |
| 대역폭 | Vercel 100GB/월 포함 | $0 |
| 인증/DB/Storage | Supabase Free (출판과 공유) | $0 |
| 도메인 | `.vercel.app` | $0 |
| SSL 인증서 | Vercel 자동 | $0 |
| 모니터링 | Vercel Analytics Free + Supabase Dashboard | $0 |
| **합계** | | **$0 / 월** |

### Free Tier 한도 대비 예상 사용량

| 리소스 | Free 한도 | 예상 사용 | 여유 |
|---|---|---|---|
| Vercel 대역폭 | 100 GB/월 | < 1 GB | 99% |
| Vercel 빌드 시간 | 6,000 분/월 | < 5 분 (빌드 없음) | 99% |
| Supabase DB 용량 | 500 MB | 공유, 현재 < 10 MB | 98% |
| Supabase Storage | 1 GB | 공유, < 10 MB | 99% |
| Supabase MAU | 50,000 | < 50명 | 99% |

**결론**: MVP는 Free Tier 한도 대비 1% 미만 사용. 유료 전환 불필요.

---

## 11. Potential Merger with 출판 Project

사용자는 본 패키지 프로젝트를 **출판 프로젝트와 하나의 Vercel 배포로 통합하는 방안**을 검토 중입니다.

### 통합 시 예상 형태
- **단일 Vercel 프로젝트**: 기존 출판 리포지토리에 `web/package/` 서브디렉토리 추가 또는 top-level 토글 UI
- **공유 인증**: Supabase는 이미 공유(security-spec Option A 채택) → 사용자는 한 번 로그인으로 두 뷰 모두 접근
- **단일 도메인**: `gabwoo.vercel.app`에서 상단 토글 또는 `/package`, `/publication` 경로로 분기
- **두 개의 data.json**: `web/data.json`(출판) + `web/package/data.json` 또는 `web/data.package.json`

### 본 독립 배포 스펙에 미치는 영향

| 항목 | 통합 시 |
|---|---|
| 본 `vercel.json` | 폐기, 출판의 vercel.json에 CSP `connect-src` 유지, 캐시 정책 병합 |
| 본 Git 저장소 | 폐기 또는 아카이브, 코드는 출판 리포로 이관 |
| 도메인 | `.vercel.app` 2개 → 1개로 축소 |
| 배포 파이프라인 | 단일 리포 = 단일 webhook = 단일 배포 |
| 비용 | 여전히 $0 (Vercel 프로젝트 2개도 무료이지만 관리 단순화) |

### 권장 사항

> ⚠️ **통합 결정 타이밍이 중요합니다.** 본 독립 배포를 먼저 수행한 뒤 통합하면 마이그레이션 작업이 이중으로 발생합니다.
>
> **Option 1 (권장)**: 통합 여부를 지금 결정 → 통합이 확정이면 본 독립 배포 **스킵**, 출판 리포에 직접 패키지 섹션 추가하여 1회 배포.
>
> **Option 2**: 통합 결정이 지연된다면 본 스펙대로 독립 배포 → 실사용 피드백 수집 → 2주 내 통합 여부 재검토 → 통합 시 본 리포 아카이브.

본 문서는 **통합 아키텍처를 설계하지 않습니다.** 통합이 결정되면 별도 아키텍처 논의(`/pdca design integrated-deployment` 수준의 설계 세션)가 필요합니다.

---

## 12. First Deployment Checklist

### 12.1 사전 준비 (Pre-deploy)

- [ ] **Supabase RLS/Storage 정책 적용** — security-spec.md 섹션 10 "운영 체크리스트" SQL 실행
- [ ] **`approved_users`에 초기 관리자 INSERT** — `zephirus03@gmail.com` + `is_admin=true`
- [ ] **Supabase Auth 설정** — 이메일 인증 ON, 비밀번호 최소 8자 설정
- [ ] **원본 엑셀 3개 파일 NAS → 로컬 `생산진행현황/` 복사**
- [ ] **`python3 convert.py` 최초 실행** → `web/data.json` 생성 확인
- [ ] **샘플 10건 수동 대조 검증 통과** (plan.md 리스크 R5, 데이터 정확도 100%)
- [ ] **`.gitignore` 섹션 5.2 반영** — 원본 엑셀이 커밋되지 않는지 `git status`로 확인
- [ ] **`vercel.json` 섹션 2 내용 배치**
- [ ] **`index.html`에 SUPABASE_URL / ANON_KEY 삽입 확인**

### 12.2 배포 (Deploy)

- [ ] `git add . && git commit -m "feat: initial package viewer release"`
- [ ] `git push origin main`
- [ ] **Vercel Dashboard**: 첫 Import 설정 (섹션 5.4)
- [ ] Deploy 버튼 클릭 → 빌드 로그 확인 (빌드 없음, 업로드만)
- [ ] 배포 URL 확인 (`https://gabwoo-package.vercel.app` 등)

### 12.3 스모크 테스트 (Post-deploy)

- [ ] **익명 접근**: 로그인 페이지가 표시되는가?
- [ ] **신규 가입**: 이메일 인증 → `signup_requests`에 INSERT 확인
- [ ] **관리자 승인**: `zephirus03@gmail.com`이 승인 버튼으로 사용자 승인 가능
- [ ] **대시보드 로드**: `data.json` 다운로드, KPI 4개 표시, 차트 렌더링
- [ ] **검색 동작**: 관리번호/품명/거래처로 검색 → 결과 카드 정상
- [ ] **공정 진행 바**: 실제 데이터 몇 건 확인 (인쇄/코팅/톰슨 등)
- [ ] **모바일 실기기**: iPhone/Android 실기기에서 2초 내 로드 확인
- [ ] **보안 헤더 검증**: `curl -I https://gabwoo-package.vercel.app` → CSP, HSTS, X-Frame-Options 응답 확인
- [ ] **캐시 정책 검증**: `data.json` 응답 헤더에 `no-cache` 확인
- [ ] **data.json URL 직접 접근 시**: 로그인 우회 가능한지 확인 → 보안 헤더는 적용되지만 URL만 알면 읽힘 (V1a 선택의 알려진 제약, security-spec 섹션 6.3 V1a/V1b 논의 참고)

### 12.4 첫 운영 주간

- [ ] 영업/생산팀 파일럿 사용자 3~5명에 URL 공유 및 가입 유도
- [ ] 3일 후: Vercel Analytics 방문자 수, Supabase `approved_users` 승인 건수 점검
- [ ] 1주 후: 통합 배포 여부 사용자 재결정 (섹션 11)

---

## 13. 참고 파일

| 경로 | 용도 |
|---|---|
| `/Users/jack/dev/gabwoo/출판_생산 진행 현황/vercel.json` | 기준이 된 출판 프로젝트 vercel.json |
| `/Users/jack/dev/gabwoo/출판_생산 진행 현황/docs/02-design/security-spec.md` | 출판 보안 스펙 (I3 보안 헤더 권고) |
| `/Users/jack/dev/gabwoo/패키지_생산진행현황/docs/02-design/security-spec.md` | 패키지 보안 스펙 (Supabase 공유, Storage 정책) |
| `/Users/jack/dev/gabwoo/패키지_생산진행현황/docs/01-plan/features/package-production-viewer.plan.md` | MVP Plan (범위, 리스크) |
