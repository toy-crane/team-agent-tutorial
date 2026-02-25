# Todo App 구현 계획

## Context

Next.js 16 + shadcn/ui 기반 프로젝트에 Supabase Local을 활용한 Todo 앱을 구현한다. 팀 에이전트(Leader + Backend + Frontend + QA) 구조로 작업을 분배하며, TDD 워크플로우를 따른다.

## 팀 구성 및 역할

```
         [Team Leader]
        /      |      \
  [Backend]  [Frontend]  [QA]
```

- **Leader**: TeamCreate로 팀 생성, TaskCreate로 작업 분배, SendMessage로 조율, 커밋 관리
- **Backend**: Supabase 설정, DB 스키마, RLS, Auth, 서버 액션
- **Frontend**: 페이지, 컴포넌트, 레이아웃, UI 구현
- **QA**: 테스트 인프라 설정, TDD 테스트 작성, 검증

---

## Phase 0: 팀 생성 및 인프라 (Leader)

### Task 0-1: 팀 생성
- TeamCreate로 "todo-app" 팀 생성
- Backend, Frontend, QA 팀메이트 스폰
- **성공기준**: `TeamCreate` 완료, 3명의 팀메이트가 활성화

### Task 0-2: 테스트 인프라 설정 (QA 담당)
- 설치: `bun add -d vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event @vitejs/plugin-react jsdom`
- 생성 파일:
  - `vitest.config.ts` — 테스트 러너 설정 (`@/*` alias, jsdom, coverage)
  - `vitest.setup.ts` — RTL cleanup, jest-dom matchers, Next.js 모킹
  - `__tests__/helpers/render.tsx` — Provider 래핑 커스텀 render
  - `__tests__/helpers/mock-supabase.ts` — Supabase 체이너블 쿼리 빌더 mock
  - `__tests__/fixtures/todo.ts` — 테스트 데이터 팩토리
- `package.json`에 스크립트 추가: `test`, `test:watch`, `test:coverage`
- **성공기준**: `bun test` 실행 시 에러 없이 "no tests found" 출력

---

## Phase 1: Supabase Local + DB 스키마 (Backend 담당)

### Task 1-1: Supabase 초기 설정
- 설치: `bun add @supabase/supabase-js @supabase/ssr`
- `npx supabase init` 실행
- `supabase/config.toml`에서 `enable_confirmations = false` 설정 (로컬 개발용)
- `npx supabase start` 실행
- `.env.local` 생성 (URL + anon key)
- `.env.example` 생성 (커밋용 템플릿)
- `.gitignore`에 `.supabase/` 추가
- **성공기준**: `npx supabase status`로 로컬 서비스 확인, `.env.local`에 유효한 키 존재

### Task 1-2: DB 스키마 마이그레이션
- `supabase/migrations/20260226000001_create_tables.sql`:
  - `priority` enum (high, medium, low)
  - `categories` 테이블 (id, name, color, user_id, created_at)
  - `todos` 테이블 (id, title, description, completed, priority, due_date, category_id, user_id, created_at, updated_at)
  - `tags` 테이블 (id, name, user_id, created_at)
  - `todo_tags` 테이블 (todo_id, tag_id, composite PK)
  - 인덱스: user_id, category_id, due_date, (user_id, completed)
  - Unique 제약: (user_id, name) on categories, tags
  - `updated_at` 자동 갱신 트리거
- **성공기준**: `npx supabase db reset` 에러 없이 완료

### Task 1-3: RLS 정책
- `supabase/migrations/20260226000002_enable_rls.sql`:
  - todos/categories/tags: `auth.uid() = user_id` 기반 SELECT/INSERT/UPDATE/DELETE
  - todo_tags: 소유한 todo의 tag만 관리 가능 (서브쿼리)
- **성공기준**: Supabase Studio에서 RLS 정책 확인, 미인증 요청 시 빈 결과 반환

### Task 1-4: Supabase 클라이언트 유틸리티
- `lib/supabase/server.ts` — 서버 클라이언트 (`cookies()` async 처리)
- `lib/supabase/client.ts` — 브라우저 클라이언트
- `lib/supabase/types.ts` — DB 타입 정의 (Todo, Category, Tag, TodoTag, Priority, Database)
- **성공기준**: 서버/클라이언트 컴포넌트에서 import 후 타입 에러 없음

---

## Phase 2: Auth 구현 (Backend + Frontend 협업)

### Task 2-1: Auth 테스트 작성 (QA 담당)
- `lib/validators/__tests__/auth.test.ts` — 이메일/비밀번호 검증 테스트
- `components/auth/__tests__/login-form.test.tsx` — 로그인 폼 렌더링/제출/에러 테스트
- `components/auth/__tests__/signup-form.test.tsx` — 회원가입 폼 테스트
- **성공기준**: 모든 테스트 RED (실패) 상태

### Task 2-2: 미들웨어 (Backend 담당)
- `middleware.ts` (프로젝트 루트):
  - 세션 리프레시 (`supabase.auth.getUser()`)
  - 라우트 보호: 미인증 → `/login` 리다이렉트
  - 인증 사용자의 `/login`, `/signup` 접근 → `/` 리다이렉트
  - matcher: 정적 자산 제외
- **성공기준**: 미인증으로 `/` 접근 시 `/login`으로 리다이렉트

### Task 2-3: Auth 서버 액션 (Backend 담당)
- `app/(auth)/actions.ts`: `signUp`, `signIn`, `signOut`
  - 서버 사이드 검증 (이메일 필수, 비밀번호 6자 이상)
  - `revalidatePath("/", "layout")` + `redirect()`
- `app/auth/callback/route.ts`: 이메일 확인 콜백
- **성공기준**: Supabase Local에서 회원가입/로그인/로그아웃 동작 확인

### Task 2-4: Auth 페이지 (Frontend 담당)
- `app/(auth)/layout.tsx` — 중앙 정렬 레이아웃
- `app/(auth)/login/page.tsx` + `login-form.tsx` — 로그인 폼 (useActionState)
- `app/(auth)/signup/page.tsx` + `signup-form.tsx` — 회원가입 폼
- 사용 컴포넌트: Card, Input, Label, Button (기존)
- **성공기준**: QA 테스트 통과 (GREEN), 브라우저에서 회원가입→로그인 플로우 동작

### Task 2-5: Auth 통합 테스트 (QA 담당)
- `__tests__/integration/auth-flow.test.tsx`: 가입→로그인→로그아웃→라우트보호 시나리오
- **성공기준**: 통합 테스트 통과

**커밋**: "feat: implement email/password auth with Supabase Local"

---

## Phase 3: 레이아웃 + 사이드바 (Frontend 담당)

### Task 3-1: shadcn 추가 컴포넌트 설치
- shadcn MCP로 적합한 컴포넌트 탐색 후 설치:
  - checkbox, dialog, sheet, switch, tabs, popover, tooltip, skeleton, scroll-area, calendar, sidebar
- `bun add date-fns` (날짜 포맷팅)
- **성공기준**: 모든 컴포넌트 `components/ui/`에 존재, `bun run build` 성공

### Task 3-2: ThemeProvider
- `components/providers/theme-provider.tsx` — dark 클래스 토글, localStorage 지속
- `hooks/use-theme.ts` — theme/toggleTheme 훅
- `app/layout.tsx` 수정: ThemeProvider 래핑, metadata 업데이트
- **성공기준**: 다크모드 토글 동작, 새로고침 후 유지

### Task 3-3: 사이드바 테스트 (QA 담당)
- `components/layout/__tests__/app-sidebar.test.tsx`: 네비게이션 렌더링, 활성 상태, 로그아웃 테스트
- **성공기준**: 테스트 RED 상태

### Task 3-4: 사이드바 + 대시보드 레이아웃 (Frontend 담당)
- `components/layout/app-sidebar.tsx` — shadcn Sidebar 활용:
  - 헤더: 앱 이름 + 사용자 Avatar
  - 필터 뷰: All, Today, Upcoming, Completed (Phosphor 아이콘)
  - 카테고리 목록 (색상 dot)
  - 푸터: Settings 링크, Sign Out
- `components/layout/top-bar.tsx` — 모바일 햄버거 + 페이지 제목
- `app/(protected)/layout.tsx` — SidebarProvider + AppSidebar + SidebarInset
- **성공기준**: QA 사이드바 테스트 통과, 사이드바 렌더링 및 모바일 반응형 동작

**커밋**: "feat: add sidebar layout with navigation and theme toggle"

---

## Phase 4: Todo CRUD (Backend + Frontend + QA 협업)

### Task 4-1: Todo 서버 액션 (Backend 담당)
- `app/(protected)/actions.ts`: `getTodos`, `createTodo`, `updateTodo`, `toggleTodo`, `deleteTodo`
  - `getTodos`: nested select `todos` + `categories` + `todo_tags(tags(*))`
  - `user_id`는 서버에서 `getUser()`로 설정
- `app/(protected)/category-actions.ts`: `getCategories`, `createCategory`, `deleteCategory`
- `app/(protected)/tag-actions.ts`: `getTags`, `createTag`, `addTagToTodo`, `removeTagFromTodo`
- **성공기준**: 각 액션이 Supabase DB에 올바르게 CRUD 수행

### Task 4-2: Todo 컴포넌트 테스트 (QA 담당)
- `components/todo/__tests__/todo-item.test.tsx`: 렌더링, 체크박스, 우선순위 뱃지, 삭제 확인
- `components/todo/__tests__/todo-list.test.tsx`: 리스트 렌더링, 빈 상태, 로딩 스켈레톤
- `components/todo/__tests__/todo-create-dialog.test.tsx`: 폼 검증, 제출
- **성공기준**: 모든 테스트 RED 상태

### Task 4-3: TodoItem 컴포넌트 (Frontend 담당)
- `components/todo/todo-item.tsx`:
  - Checkbox + 제목 (완료 시 line-through) + Priority Badge + Category Badge + 마감일
  - DropdownMenu: Edit, Delete (destructive)
  - 클릭 시 설명 확장
- `components/todo/todo-empty-state.tsx` — 빈 상태 메시지
- `components/todo/todo-skeleton.tsx` — 로딩 스켈레톤
- **성공기준**: todo-item 테스트 통과

### Task 4-4: TodoList + 검색/필터 (Frontend 담당)
- `components/todo/todo-list.tsx` — 필터/정렬된 아이템 리스트, ScrollArea
- `components/todo/todo-search-bar.tsx` — InputGroup + 디바운스 검색 (300ms)
- `components/todo/todo-filter-chips.tsx` — 상태/우선순위/카테고리 필터 칩
- `components/todo/todo-sort-select.tsx` — 정렬 (마감일/우선순위/생성일)
- **성공기준**: todo-list 테스트 통과, 검색/필터/정렬 동작

### Task 4-5: Todo 생성/수정/삭제 (Frontend 담당)
- `components/todo/date-picker.tsx` — Popover + Calendar
- `components/todo/todo-create-dialog.tsx` — Dialog 기반 생성 폼 (title, description, priority, due_date, category, tags)
- `components/todo/todo-edit-dialog.tsx` — 기존 데이터 프리필 수정 폼
- `components/todo/todo-delete-dialog.tsx` — AlertDialog 삭제 확인
- **성공기준**: 생성/수정/삭제 테스트 통과

### Task 4-6: 대시보드 메인 페이지 조립 (Frontend 담당)
- `app/(protected)/page.tsx`:
  - TopBar (검색 + 생성 버튼)
  - TodoFilterChips + TodoSortSelect
  - TodoList
  - TodoCreateDialog
- **성공기준**: 전체 Todo CRUD 플로우가 브라우저에서 동작

### Task 4-7: Todo CRUD 통합 테스트 (QA 담당)
- `__tests__/integration/todo-crud.test.tsx`: 생성→조회→수정→삭제 시나리오
- `__tests__/integration/filter-search.test.tsx`: 필터+검색 조합 테스트
- **성공기준**: 모든 통합 테스트 통과

**커밋**: "feat: implement todo CRUD with search, filter, sort"

---

## Phase 5: Settings 페이지 (Frontend + QA)

### Task 5-1: Settings 테스트 (QA 담당)
- `components/settings/__tests__/theme-toggle.test.tsx`: 테마 토글 + 지속성
- `components/settings/__tests__/password-form.test.tsx`: 비밀번호 변경 검증
- **성공기준**: 테스트 RED 상태

### Task 5-2: Settings 컴포넌트 (Frontend 담당)
- `components/settings/theme-toggle.tsx` — Switch + SunIcon/MoonIcon
- `components/settings/password-change-form.tsx` — 현재/새/확인 비밀번호 (Card + FieldGroup)
- `components/settings/email-verification-status.tsx` — 이메일 + 인증 상태 Badge
- `components/settings/terms-of-service.tsx` — ScrollArea 기반 약관 표시
- **성공기준**: Settings 테스트 통과

### Task 5-3: Settings 페이지 조립 (Frontend 담당)
- `app/(protected)/settings/page.tsx`:
  - Tabs: Appearance (테마 토글) | Account (비밀번호 변경 + 이메일 확인) | Legal (이용약관)
- **성공기준**: 3개 탭 전환 동작, 테마 토글 end-to-end 동작

### Task 5-4: Settings 통합 테스트 (QA 담당)
- `__tests__/integration/settings-updates.test.tsx`: 테마 변경, 비밀번호 변경 시나리오
- **성공기준**: 통합 테스트 통과

**커밋**: "feat: add settings page with theme, password, terms"

---

## Phase 6: 마무리 (Leader 조율)

### Task 6-1: 정리
- 예제 파일 삭제: `components/component-example.tsx`, `components/example.tsx`
- `app/page.tsx`를 `(protected)` 그룹으로 이동 또는 리다이렉트
- **성공기준**: 불필요한 예제 코드 없음, 모든 라우트 정상

### Task 6-2: 모바일 반응형 확인
- 사이드바 → Sheet로 축소 (768px 이하)
- Todo 생성 버튼 → 플로팅 FAB
- 필터 칩 → 수평 스크롤
- **성공기준**: 375px 뷰포트에서 모든 기능 사용 가능

### Task 6-3: 최종 검증
- `bun test` — 모든 테스트 통과
- `bun run build` — 빌드 성공
- `bun run lint` — 린트 에러 없음
- **성공기준**: 3개 명령어 모두 에러 없이 완료

**커밋**: "chore: cleanup example code and verify build"

---

## 주요 파일 목록

### 새로 생성 (~40개)

```
# Supabase
.env.local, .env.example
supabase/migrations/20260226000001_create_tables.sql
supabase/migrations/20260226000002_enable_rls.sql
lib/supabase/server.ts, lib/supabase/client.ts, lib/supabase/types.ts
middleware.ts

# Auth
app/(auth)/layout.tsx, app/(auth)/actions.ts
app/(auth)/login/page.tsx, app/(auth)/login/login-form.tsx
app/(auth)/signup/page.tsx, app/(auth)/signup/signup-form.tsx
app/auth/callback/route.ts

# Dashboard
app/(protected)/layout.tsx, app/(protected)/page.tsx
app/(protected)/actions.ts, app/(protected)/category-actions.ts, app/(protected)/tag-actions.ts
app/(protected)/settings/page.tsx

# Components
components/providers/theme-provider.tsx
components/layout/app-sidebar.tsx, components/layout/top-bar.tsx
components/todo/todo-item.tsx, todo-list.tsx, todo-create-dialog.tsx
components/todo/todo-edit-dialog.tsx, todo-delete-dialog.tsx, todo-search-bar.tsx
components/todo/todo-filter-chips.tsx, todo-sort-select.tsx, todo-empty-state.tsx
components/todo/todo-skeleton.tsx, date-picker.tsx
components/settings/theme-toggle.tsx, password-change-form.tsx
components/settings/email-verification-status.tsx, terms-of-service.tsx

# Hooks
hooks/use-theme.ts, hooks/use-debounce.ts

# Testing
vitest.config.ts, vitest.setup.ts
__tests__/helpers/render.tsx, __tests__/helpers/mock-supabase.ts
__tests__/fixtures/todo.ts
```

### 수정 파일
- `app/layout.tsx` — ThemeProvider 추가
- `app/page.tsx` — protected 라우트로 이동
- `package.json` — 테스트 스크립트 추가
- `.gitignore` — `.supabase/` 추가

---

## 검증 방법

1. **Supabase**: `npx supabase status` → 서비스 실행 확인
2. **Auth**: 브라우저에서 회원가입 → 로그인 → 로그아웃 플로우
3. **Todo CRUD**: 생성 → 수정 → 완료 토글 → 삭제 → 검색/필터
4. **Settings**: 테마 토글 → 새로고침 유지, 비밀번호 변경 폼 검증
5. **테스트**: `bun test` 전체 통과
6. **빌드**: `bun run build && bun run lint` 에러 없음
