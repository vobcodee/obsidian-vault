# simple-note 로그인 구현 가이드

**작성일:** 2026-02-12  
**상태:** Phase 2 구현 대상  

---

## 🔐 현재 상황

Phase 1에서 서버 액션에 임시 `user_id`가 하드코딩되어 있습니다:

```typescript
// app/notes/actions.ts
async function getCurrentUser() {
  // TODO: Replace with actual auth.getUser() call
  return { id: process.env.TEST_USER_ID || '00000000-0000-0000-0000-000000000000' };
}
```

---

## 로그인 방식 선택

| 방식 | 장점 | 단점 | 추천도 |
|------|------|------|--------|
| **Magic Link** | 비밀번호 없음, 간편 | 이메일 확인 필요 | ⭐⭐⭐⭐⭐ |
| **Email/Password** | 전통적, 익숙함 | 비밀번호 관리 필요 | ⭐⭐⭐⭐ |
| **OAuth (Google)** | 클릭 한 번으로 로그인 | 설정 복잡함 | ⭐⭐⭐⭐ |

---

## 방법 1: Magic Link (권장)

### 1. Supabase 설정

Dashboard → Authentication → Providers → **Email** → **Enable**:
- ✅ `Confirm email` (이메일 확인 필요)
- ✅ `Enable Signup` (회원가입 활성화)
- ⬜ `Secure email change` (이메일 변경 보안)

### 2. 로그인/회원가입 페이지

```typescript
// app/login/page.tsx
'use client';

import { useState } from 'react';
import { createClient } from '@supabase/supabase-js';
import { toast } from 'sonner';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);

export default function LoginPage() {
  const [email, setEmail] = useState('');
  const [loading, setLoading] = useState(false);
  const [sent, setSent] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setLoading(true);

    const { error } = await supabase.auth.signInWithOtp({
      email,
      options: {
        emailRedirectTo: `${window.location.origin}/auth/callback`,
      },
    });

    if (error) {
      toast.error('로그인 링크 발송 실패: ' + error.message);
    } else {
      toast.success('이메일로 로그인 링크를 볃송했습니다!');
      setSent(true);
    }

    setLoading(false);
  };

  if (sent) {
    return (
      <div className="max-w-md mx-auto p-8 text-center">
        <h2 className="text-2xl font-bold mb-4">📧 확인하세요!</h2>
        <p className="text-neutral-600">
          {email}로 로그인 링크를 볃송했습니다.<br />
          이메일을 확인하고 링크를 클릭하면 로그인됩니다.
        </p>
      </div>
    );
  }

  return (
    <main className="max-w-md mx-auto p-8">
      <h2 className="text-2xl font-bold mb-6">로그인</h2>
      <form onSubmit={handleSubmit} className="space-y-4">
        <input
          type="email"
          placeholder="이메일 주소"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
          className="w-full border rounded-lg p-3"
        />
        <button
          type="submit"
          disabled={loading}
          className="w-full bg-black text-white rounded-lg p-3 disabled:opacity-50"
        >
          {loading ? '전송 중...' : '로그인 링크 받기'}
        </button>
      </form>
      <p className="mt-4 text-sm text-neutral-500 text-center">
        이메일로 로그인 링크가 발송됩니다. 비밀번호가 필요 없어요!
      </p>
    </main>
  );
}
```

### 3. 인증 콜백 핸들러

```typescript
// app/auth/callback/route.ts
import { createClient } from '@supabase/supabase-js';
import { NextResponse } from 'next/server';

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get('code');
  const next = searchParams.get('next') ?? '/';

  if (code) {
    const supabase = createClient(
      process.env.SUPABASE_URL!,
      process.env.SUPABASE_SERVICE_ROLE_KEY!
    );

    const { error } = await supabase.auth.exchangeCodeForSession(code);
    if (!error) {
      return NextResponse.redirect(`${origin}${next}`);
    }
  }

  return NextResponse.redirect(`${origin}/login?error=auth`);
}
```

### 4. Middleware로 세션 관리

```typescript
// middleware.ts (루트에 생성)
import { createServerClient } from '@supabase/ssr';
import { NextResponse, type NextRequest } from 'next/server';

export async function middleware(request: NextRequest) {
  let supabaseResponse = NextResponse.next({
    request,
  });

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        getAll() {
          return request.cookies.getAll();
        },
        setAll(cookiesToSet) {
          cookiesToSet.forEach(({ name, value }) => request.cookies.set(name, value));
          supabaseResponse = NextResponse.next({
            request,
          });
          cookiesToSet.forEach(({ name, value, options }) =>
            supabaseResponse.cookies.set(name, value, options)
          );
        },
      },
    }
  );

  const {
    data: { user },
  } = await supabase.auth.getUser();

  // 보호된 라우트 체크
  if (request.nextUrl.pathname.startsWith('/notes') && !user) {
    return NextResponse.redirect(new URL('/login', request.url));
  }

  return supabaseResponse;
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)'],
};
```

### 5. 서버 액션 업데이트

```typescript
// app/notes/actions.ts
import { createClient } from '@supabase/supabase-js';
import { cookies } from 'next/headers';

async function getCurrentUser() {
  const cookieStore = cookies();
  
  const supabase = createClient(
    process.env.SUPABASE_URL!,
    process.env.SUPABASE_SERVICE_ROLE_KEY!,
    {
      auth: {
        autoRefreshToken: false,
        persistSession: false,
      },
      global: {
        headers: {
          cookie: cookieStore.toString(),
        },
      },
    }
  );

  const { data: { user }, error } = await supabase.auth.getUser();
  
  if (error || !user) {
    throw new Error('인증이 필요합니다.');
  }

  return { id: user.id };
}
```

### 6. 로그아웃 버튼

```typescript
// components/LogoutButton.tsx
'use client';

import { createClient } from '@supabase/supabase-js';
import { useRouter } from 'next/navigation';
import { toast } from 'sonner';

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);

export function LogoutButton() {
  const router = useRouter();

  const handleLogout = async () => {
    await supabase.auth.signOut();
    toast.success('로그아웃되었습니다.');
    router.push('/login');
    router.refresh();
  };

  return (
    <button
      onClick={handleLogout}
      className="text-sm text-neutral-500 hover:text-red-500 transition-colors"
    >
      로그아웃
    </button>
  );
}
```

### 7. Header에 로그아웃 추가

```typescript
// app/layout.tsx (Header 부분 수정)
<header className="flex flex-wrap justify-between items-center mb-8 gap-4">
  <h1 className="text-xl font-bold">Simple Notes</h1>
  <nav className="flex gap-4 text-sm items-center">
    <a href="/notes" className="hover:text-accent transition-colors">노트 목록</a>
    <a href="/notes/new" className="text-accent font-medium hover:underline">새 노트</a>
    <LogoutButton />
  </nav>
</header>
```

---

## 패키지 설치

```bash
npm install @supabase/ssr
```

---

## 환경 변수 업데이트

```bash
# .env.local
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key

SUPABASE_URL=https://your-project.supabase.co
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key
```

---

## Supabase Dashboard 설정

### Site URL 설정

Authentication → URL Configuration:
- **Site URL**: `http://localhost:3000` (개발)
- **Redirect URLs**: `http://localhost:3000/auth/callback`

### Email 템플릿 (선택)

Authentication → Email Templates → **Magic Link**:

```html
<h2>Magic Link</h2>
<p>Follow this link to login:</p>
<p><a href="{{ .ConfirmationURL }}">Log In</a></p>
```

---

## 테스트 순서

1. `/login` 접속
2. 이메일 입력 → "로그인 링크 받기"
3. 이메일 확인 (스팸함도 체크)
4. 링크 클릭 → 자동 로그인
5. `/notes` 접속 확인
6. 로그아웃 → `/login` 리다이렉트 확인

---

## 참고

- [[2026-02-12-simple-note-개선계획|전체 개선 계획]]
- [Supabase Auth Docs](https://supabase.com/docs/guides/auth)
- [Next.js Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware)
