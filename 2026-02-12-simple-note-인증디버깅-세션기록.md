# simple-note 인증 디버깅 세션 기록

**날짜:** 2026-02-12  
**시간:** 약 5시간 소요  
**상태:** 🔴 진행 중 (미해결)

---

## 📋 현재 상황

### 구현된 기능
✅ Google OAuth 로그인 (클이언트 측)  
✅ 쿠키에 세션 저장 (Supabase 자동 처리)  
✅ Middleware에서 쿠키 존재 여부 확인  
❌ Server Actions에서 세션 읽기 실패  

---

## 🔍 시도한 방법들

### 1. `@supabase/ssr` 사용 (권장 방식)
```typescript
const supabase = createServerClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  { cookies: { getAll, setAll } }
);
const { data: { session } } = await supabase.auth.getSession();
```
**결과:** `session`이 항상 `null` 반환

### 2. 쿠키 직접 파싱 시도
```typescript
const authCookie = cookies.find(c => 
  c.name.startsWith('sb-') && c.name.endsWith('-auth-token')
);
const decoded = atob(authCookie.value); // 또는 Buffer.from
```
**결과:** 쿠키 값이 암호화되어 있어 디코딩 실패

### 3. Middleware에서 헤더로 전달
```typescript
// middleware.ts
response.headers.set('x-user-id', user.id);

// actions.ts
const userId = headers().get('x-user-id');
```
**결과:** 미들웨어에서도 세션 읽기 실패

### 4. PKCE flow 설정
```typescript
auth: {
  flowType: 'pkce',
  autoRefreshToken: true,
  persistSession: true,
}
```
**결과:** 변화 없음

---

## 🔴 핵심 문제

**Server Component / Server Action에서 Supabase 세션에 접근 불가**

| 위치 | 세션 읽기 | 상태 |
|------|----------|------|
| Client (Browser) | ✅ `supabase.auth.getSession()` | 성공 |
| Middleware | ❌ `@supabase/ssr` | 실패 |
| Server Action | ❌ `@supabase/ssr` | 실패 |
| Server Component | ❌ `@supabase/ssr` | 실패 |

---

## 🧪 디버깅 로그

### Middleware
```
[MIDDLEWARE] Has auth cookie: true
[MIDDLEWARE] Proceeding
```
→ 쿠키는 존재함

### Server Actions
```
[DEBUG] getCurrentUser called
[DEBUG] Session: null Error: none
[DEBUG] User: null Error: Auth session missing!
```
→ `@supabase/ssr`가 쿠키를 읽지 못함

---

## 💡 가능한 원인

### 1. 쿠키 SameSite 설정 문제
Supabase 쿠키가 `SameSite=Lax` 또는 `SameSite=Strict`로 설정되어 있어 서버 액션에서 접근 불가

### 2. `@supabase/ssr` 버전 호환성
Next.js 16.1.1 + Turbopack와 `@supabase/ssr`의 호환성 문제

### 3. 쿠키 도메인/경로 설정
쿠키가 `localhost`가 아닌 다른 도메인으로 설정됨

### 4. PKCE 설정 문제
Google OAuth로 인증 시 PKCE flow가 제대로 동작하지 않음

---

## 🛠 시도할 수 있는 해결책

### 1. `@supabase/auth-helpers-nextjs` 사용
```bash
npm install @supabase/auth-helpers-nextjs
```
→ 더 안정적인 Next.js 통합 제공

### 2. 쿠키 설정 직접 제어
```typescript
// supabaseClient.ts
const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
  {
    auth: {
      cookieOptions: {
        sameSite: 'lax',
        secure: false, // 개발 환경
      }
    }
  }
);
```

### 3. Route Handler 방식으로 전환
Server Actions 대신 `/api/notes/*` Route Handler 사용

### 4. Supabase Dashboard 설정 확인
- **URL Configuration**: `http://localhost:3000`
- **Redirect URLs**: `http://localhost:3000/auth/callback`
- **Cookie Settings**: SameSite 설정

### 5. 로컬 개발 환경에서 HTTPS 사용
```bash
npx local-ssl-proxy --source 3001 --target 3000
```
→ `https://localhost:3001`로 접속

---

## 📁 관련 파일

- `app/login/page.tsx` - 로그인 페이지
- `app/auth/callback/page.tsx` - OAuth 콜백
- `app/notes/actions.ts` - 서버 액션
- `middleware.ts` - 미들웨어
- `components/Header.tsx` - 헤더 컴포넌트
- `components/LogoutButton.tsx` - 로그아웃 버튼

---

## 📝 메모

- 이메일 인증은 rate limit으로 테스트 불가
- Google OAuth는 작동하나 세션 읽기 실패
- `@supabase/ssr` 쿠키 파싱이 제대로 되지 않음
- Next.js 16 + Turbopack 환경

---

## 🔄 다음 단계

### 옵션 A: 계속 디버깅
1. Supabase 공식 문서 다시 확인
2. GitHub Issues 검색 (supabase/supabase-js)
3. 쿠키 브라우저 개발자 도구에서 직접 확인

### 옵션 B: 대안 구현
1. Route Handler 방식으로 전환
2. JWT 직접 검증 (Supabase 없이)
3. 다른 인증 라이브러리 사용 (NextAuth.js 등)

### 옵션 C: 임시 비활성화
1. 개발 중 인증 체크 비활성화
2. 프로덕션 배포 전 다시 구현

---

**결론:** 현재 Next.js 16 + `@supabase/ssr` 조합에서 서버 측 세션 읽기가 작동하지 않음. 공식 예제나 다른 프로젝트 참고 필요.
