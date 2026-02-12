# 2026-02-12 개발 세션 - simple-note 인증 구현

**시간:** 10:14 - 17:18 (약 7시간)  
**참여자:** Seongjin Kim, OpenClaw Agent  
**브랜치:** `feat/login-magic-link`  
**상태:** ⚠️ 개발 중 (인증 일부 해결, 개발용 임시 방편 적용)

---

## 📋 세션 목표

1. ✅ GitHub 연동 확인
2. ✅ Phase 1 (보안 개선) 완료 및 머지
3. 🔴 Phase 2 (인증 구현) - 진행 중, 기술적 문제로 임시 해결책 적용
4. ⚠️ Vercel 자동 배포 설정 확인

---

## ✅ 완료된 작업

### 1. GitHub 접근 확인
- GitHub CLI 인증 상태 확인 완료
- 계정: `vobcodee`
- 토큰 스코프: `repo`, `read:org`, `gist`, `workflow`

### 2. Phase 1: 보안 및 코드 품질 개선
**완료 및 PR #1 머지 완료**

| 항목 | 상태 |
|------|------|
| Zod 스키마 추가 | ✅ |
| 서버 액션 생성 | ✅ |
| Supabase 마이그레이션 (003, 004) | ✅ |
| RLS 정책 업데이트 | ✅ |
| Toast 알림 적용 | ✅ |

**생성된 파일:**
- `schemas/note.ts`
- `app/notes/actions.ts`
- `supabase/migrations/003_add_user_id_to_notes.sql`
- `supabase/migrations/004_update_rls_policies_auth.sql`

### 3. Phase 2: 인증 시스템 구현 (부분 완료)

#### 시도한 방법들 (5시간+ 디버깅)

**A. @supabase/ssr 사용 (권장 방식)**
```typescript
const supabase = createServerClient(...);
const { data: { session } } = await supabase.auth.getSession();
```
**결과:** Server Component/Action에서 `session`이 항상 `null`

**B. 쿠키 직접 파싱**
```typescript
const decoded = Buffer.from(authCookie.value, 'base64').toString('utf-8');
const parsed = JSON.parse(decoded);
```
**결과:** 쿠키 값이 암호화되어 디코딩 불가

**C. Middleware → 헤더 전달**
```typescript
// middleware.ts
response.headers.set('x-user-id', user.id);

// actions.ts  
const userId = headers().get('x-user-id');
```
**결과:** Middleware에서도 세션 읽기 실패

**D. Route Handler 방식**
- `/api/notes/*` API 라우트 생성
- `@supabase/ssr`로 인증 시도
**결과:** 동일한 문제 발생

#### 최종 해결책 (임시)

**개발용 인증 우회 방식 적용:**

1. **`/dev-auth` 페이지 생성**
   - Supabase Dashboard에서 조회한 User ID 입력
   - localStorage에 저장

2. **API에서 헤더로 User ID 수신**
   ```typescript
   const userId = request.headers.get('x-dev-user-id');
   ```

3. **클이언트에서 헤더로 User ID 전송**
   ```typescript
   headers['x-dev-user-id'] = localStorage.getItem('dev_user_id');
   ```

**⚠️ 주의:** 이 방식은 개발 중에만 사용. 프로덕션 배포 전 반드시 실제 인증으로 교체 필요.

---

## 🔍 문제 원인 분석

### 원인 1: Next.js 16 + Turbopack + @supabase/ssr 호환성
- Next.js 16.1.1 (Turbopack) 환경에서 `@supabase/ssr`의 쿠키 처리에 문제
- `cookies()`로 읽은 쿠키를 `@supabase/ssr`가 제대로 파싱하지 못함

### 원인 2: PKCE Flow 설정 문제
- Google OAuth로 인증 시 PKCE 설정이 제대로 동작하지 않을 수 있음
- Supabase Dashboard에서 PKCE 옵션 UI가 보이지 않음

### 원인 3: 쿠키 암호화
- Supabase가 쿠키를 추가로 암호화하여 직접 파싱 불가
- `@supabase/ssr` 없이는 복호화 불가능

---

## 📁 생성/수정된 파일

### Phase 1 (완료)
```
schemas/note.ts
app/notes/actions.ts (초기 버전)
supabase/migrations/003_add_user_id_to_notes.sql
supabase/migrations/004_update_rls_policies_auth.sql
.env.example
```

### Phase 2 (진행 중)
```
app/login/page.tsx - Google OAuth + Magic Link UI
app/auth/callback/page.tsx - OAuth 콜백 처리
middleware.ts - 인증 체크 (현재 비활성화)
components/Header.tsx - 인증 상태 표시
components/LogoutButton.tsx - 로그아웃 버튼
app/api/notes/route.ts - API 라우트 (GET, POST)
app/api/notes/[id]/route.ts - API 라우트 (GET, PUT, DELETE)
app/dev-auth/page.tsx - 개발용 User ID 입력
app/notes/page.tsx - 클라이언트에서 API 호출
app/notes/new/page.tsx - 새 노트 작성
app/notes/[id]/edit/page.tsx - 노트 수정
components/NoteCard.tsx - 노트 카드 (삭제 기능)
```

---

## 🛠 프로덕션 배포 전 필수 작업

### 1. 인증 시스템 재구현
**옵션 A: NextAuth.js**
- 가장 안정적인 Next.js 인증 라이브러리
- Supabase Adapter 사용 가능

**옵션 B: Supabase Auth Helpers (공식)**
- `@supabase/auth-helpers-nextjs` 패키지
- Next.js 15+ 지원 확인 필요

**옵션 C: Route Handler + JWT 직접 검증**
- jose 라이브러리 사용
- Supabase JWT 서명 검증

### 2. 환경 변수 설정
```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=
NEXT_PUBLIC_SUPABASE_ANON_KEY=
SUPABASE_URL=
SUPABASE_SERVICE_ROLE_KEY=
# TEST_USER_ID 제거 (개발용)
```

### 3. 미들웨어 재활성화
```typescript
// middleware.ts
export const config = {
  matcher: ['/((?!api|_next/static|...).*)'],
};
```

### 4. Supabase 설정 확인
- [ ] Dashboard → URL Configuration
- [ ] Google OAuth Provider 설정
- [ ] RLS 정책 활성화 확인
- [ ] Cookie Settings (SameSite, Secure)

---

## 📊 시간 투자 분석

| 작업 | 예상 시간 | 실제 시간 | 비고 |
|------|----------|----------|------|
| Phase 1 (보안 개선) | 2-3시간 | 2시간 | ✅ 예상 내 |
| Phase 2 인증 구현 | 3-5시간 | 5시간+ | 🔴 예상 초과 |
| 디버깅 (@supabase/ssr) | - | 5시간+ | 원인 불명 |
| 임시 해결책 구현 | 30분 | 1시간 | ✅ 완료 |

---

## 💡 교훈 및 인사이트

### 기술적 교훈
1. **Next.js 16 + Turbopack + @supabase/ssr** 조합에서 서버 측 인증은 아직 안정적이지 않을 수 있음
2. **쿠키 기반 인증**은 클라이언트/서버 간 쿠키 동기화가 핵심
3. **Route Handler**가 Server Actions보다 디버깅이 용이

### 프로세스 교훈
1. 복잡한 기술 스택(Next.js 16 + Supabase)은 사전 검증 필요
2. 공식 문서의 예제가 항상 최신 버전과 일치하지 않음
3. 오래 디버깅할 경우 대안을 빨리 모색하는 것이 효율적

---

## 📝 관련 링크

- GitHub PR: https://github.com/vobcodee/simple-note/pull/3
- Supabase Auth Docs: https://supabase.com/docs/guides/auth
- Next.js Middleware: https://nextjs.org/docs/app/building-your-application/routing/middleware
- @supabase/ssr: https://github.com/supabase/ssr

---

## 다음 세션에서 할 일

1. [ ] 실제 인증 시스템 재구현 (NextAuth.js 권장)
2. [ ] Supabase RLS 정책 최종 확인
3. [ ] Vercel 배포 테스트
4. [ ] E2E 테스트 작성

---

**기록:** 2026-02-12  
**옵시디언 파일명:** `2026-02-12-개발세션-simple-note-인증구현.md`
