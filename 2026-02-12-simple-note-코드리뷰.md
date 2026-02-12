# simple-note 코드 리뷰

**검토일:** 2026-02-12  
**검토자:** OpenClaw Agent  
**대상:** [[simple-note]] 레포지토리  

---

## 프로젝트 개요

| 항목 | 내용 |
|------|------|
| **타입** | Next.js 16 + React 19 기반 노트 CRUD 앱 |
| **DB/백엔드** | Supabase (PostgreSQL) |
| **스타일** | Tailwind CSS 4 |
| **테스트** | Vitest + React Testing Library |
| **상태 관리** | React useState (로컬) |

---

## 잘된 점 ✅

### 최신 기술 스택
- Next.js 16, React 19, Tailwind CSS 4 적극 활용
- React Compiler 활성화 (`next.config.ts`)

### 타입스크립트 도입
- 기본적인 타입 정의 (`Note`, `NoteFormProps` 등)
- Path alias 설정 (`@/*`)으로 import 경로 깔끔함

### 테스트 인프라
- Vitest + jsdom + React Testing Library 설정 완료
- Mock 기반 단위 테스트 작성

### DB 마이그레이션
- Supabase 마이그레이션 파일로 스키마 버전 관리
- 인덱스 추가 (`notes_created_at_idx`)로 조회 성능 고려

### 데이터 백업 스크립트
- `export:notes` 스크립트로 JSON 백업 기능 제공

---

## 개선 필요사항 ⚠️

### 🔴 Critical (즉시 수정 권장)

| 항목 | 문제점 | 개선 방향 |
|------|--------|-----------|
| **보안 - RLS** | 모든 사용자가 모든 노트에 접근 가능 (`using (true)`) | 인증 기반 정책 적용: `auth.uid() = user_id` |
| **보안 - 클라이언트 키 노출** | `NEXT_PUBLIC_SUPABASE_ANON_KEY`로 클라이언트에 노출 | 서버 액션(Server Actions) 또는 Route Handler 사용 |

### 🟠 Major (중요)

| 항목 | 문제점 | 개선 방향 |
|------|--------|-----------|
| **에러 핸들링** | `alert()`만 사용, UX 미흡 | Toast/Modal 컴포넌트 도입, 에러 바욍리 |
| **타입 단언** | `as Note`로 강제 캐스팅 | zod 등으로 런타임 검증 + 타입 추론 |
| **낙관적 업데이트** | `router.refresh()`로 전체 재요청 | SWR/React Query + 낙관적 업데이트 |
| **폼 검증** | 빈 값만 체크, 길이/특수문자 제한 없음 | zod/yup 도입으로 스키마 검증 |
| **Edit 페이지 로딩** | `useEffect`로 데이터 페칭, 로딩 중 깜빡임 | `loading.tsx` 또는 스트리밍 활용 |

### 🟡 Minor (권장)

| 항목 | 문제점 | 개선 방향 |
|------|--------|-----------|
| **메타데이터** | 페이지별 title/description 없음 | `generateMetadata` 활용 |
| **Link 컴포넌트** | `layout.tsx`에서 `<a>` 사용 | Next.js `<Link>`로 변경 |
| **접근성** | `aria-label`, `role` 등 부재 | 접근성 속성 추가 |
| **콘텐츠 길이** | 제목/내용 무제한 입력 가능 | `maxLength` 또는 DB 제약 추가 |
| **날짜 포맷** | `toLocaleDateString` 클라이언트 의존 | `date-fns` 등 라이브러리로 통일 |

---

## 구체적인 개선 코드 예시

### 1. 보안 개선 - 서버 액션 도입

```typescript
// app/notes/actions.ts
'use server'
import { createClient } from '@supabase/supabase-js'

const supabase = createClient(
  process.env.SUPABASE_URL!,  // 서버 전용 env
  process.env.SUPABASE_SERVICE_ROLE_KEY!
)

export async function createNote(title: string, content: string) {
  // 서버에서만 실행되므로 anon key 노출 없음
  // + 사용자 인증 검증 가능
}
```

### 2. 타입 안전성 개선

```typescript
// types/note.ts
import { z } from 'zod'

export const NoteSchema = z.object({
  id: z.string().uuid(),
  title: z.string().min(1).max(200),
  content: z.string().max(10000),
  created_at: z.string().datetime(),
  updated_at: z.string().datetime(),
})

export type Note = z.infer<typeof NoteSchema>

// 사용 시
const { data, error } = await supabase.from("notes").select()
const notes = NoteSchema.array().parse(data) // 런타임 검증
```

### 3. 에러 핸들링 개선

```typescript
// lib/errors.ts
export class NoteError extends Error {
  constructor(message: string, public code: string) {
    super(message)
  }
}

// components/ErrorBoundary.tsx
// 또는 react-error-boundary 사용
```

---

## 종합 평가

| 영역 | 점수 | 코멘트 |
|------|------|--------|
| 기능 완성도 | ⭐⭐⭐⭐☆ | CRUD 기본 기능 충실 |
| 코드 품질 | ⭐⭐⭐☆☆ | 타입 사용하나 강제 캐스팅 많음 |
| 보안 | ⭐⭐☆☆☆ | RLS 개방적, 클라이언트 키 노출 |
| 테스트 | ⭐⭐⭐☆☆ | Mock 기반 단위 테스트 존재 |
| UX/성능 | ⭐⭐⭐☆☆ | 낙관적 업데이트 없음, 로딩 미흡 |
| 유지보수성 | ⭐⭐⭐⭐☆ | 구조 깔끔, 마이그레이션 관리 |

### 총평

학습용/포트폴리오용으로 좋은 구조입니다. 프로덕션 배포 전 **보안(RLS + 인증)**과 **에러 핸들링** 개선이 필수적입니다.

---

## 관련 링크

- 레포지토리: https://github.com/vobcodee/simple-note
- 마이그레이션: [[supabase/migrations]]
- 타입 정의: [[types/note]]
- DB 함수: [[lib/db/notes]]
