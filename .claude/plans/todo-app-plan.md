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
- **성공기준**:
  - 조건: `TeamCreate("todo-app")` 호출
  - 결과: 팀 생성 완료, `TaskList` 조회 시 Backend/Frontend/QA 3명의 팀메이트가 `active` 상태로 표시

### Task 0-2: 테스트 인프라 설정 (QA 담당)
- 설치: `bun add -d vitest @testing-library/react @testing-library/jest-dom @testing-library/user-event @vitejs/plugin-react jsdom`
- 생성 파일:
  - `vitest.config.ts` — 테스트 러너 설정 (`@/*` alias, jsdom, coverage)
  - `vitest.setup.ts` — RTL cleanup, jest-dom matchers, Next.js 모킹
  - `__tests__/helpers/render.tsx` — Provider 래핑 커스텀 render
  - `__tests__/helpers/mock-supabase.ts` — Supabase 체이너블 쿼리 빌더 mock
  - `__tests__/fixtures/todo.ts` — 테스트 데이터 팩토리
- `package.json`에 스크립트 추가: `test`, `test:watch`, `test:coverage`
- **성공기준**:
  - 조건: `bun test` 실행
  - 결과: exit code 0, 콘솔에 "no test suites found" 또는 테스트 0개 통과 메시지 출력 (에러 없음)
  - 조건: `vitest.config.ts`에서 `@/*` alias resolve 확인
  - 결과: `resolve.alias`에 `@` → 프로젝트 루트 매핑 존재

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
- **성공기준**:
  - 조건: `npx supabase status` 실행
  - 결과: `API URL: http://127.0.0.1:54321` 포함, `DB URL`, `anon key`, `service_role key` 모두 출력
  - 조건: `cat .env.local` 실행
  - 결과: `NEXT_PUBLIC_SUPABASE_URL=http://127.0.0.1:54321`과 `NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJ...` (유효한 JWT) 존재
  - 조건: `cat .gitignore | grep supabase`
  - 결과: `.supabase/` 라인 존재

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
- **성공기준**:
  - 조건: `npx supabase db reset` 실행
  - 결과: exit code 0, "Applying migration" 로그에 `20260226000001_create_tables.sql` 포함, 에러 메시지 없음
  - 조건: Supabase Studio(`http://127.0.0.1:54323`)에서 테이블 목록 확인
  - 결과: `todos`, `categories`, `tags`, `todo_tags` 4개 테이블 존재, `priority` enum 타입 존재

### Task 1-3: RLS 정책
- `supabase/migrations/20260226000002_enable_rls.sql`:
  - todos/categories/tags: `auth.uid() = user_id` 기반 SELECT/INSERT/UPDATE/DELETE
  - todo_tags: 소유한 todo의 tag만 관리 가능 (서브쿼리)
- **성공기준**:
  - 조건: anon key로 `curl -H "apikey: <anon_key>" http://127.0.0.1:54321/rest/v1/todos` 요청 (Authorization 헤더 없음)
  - 결과: HTTP 200, 응답 body `[]` (빈 배열) — RLS가 미인증 요청을 차단
  - 조건: `npx supabase db reset` 실행
  - 결과: 두 마이그레이션 파일 모두 에러 없이 적용

### Task 1-4: Supabase 클라이언트 유틸리티
- `lib/supabase/server.ts` — 서버 클라이언트 (`cookies()` async 처리)
- `lib/supabase/client.ts` — 브라우저 클라이언트
- `lib/supabase/types.ts` — DB 타입 정의 (Todo, Category, Tag, TodoTag, Priority, Database)
- **성공기준**:
  - 조건: `bunx tsc --noEmit` 실행
  - 결과: `lib/supabase/server.ts`, `lib/supabase/client.ts`, `lib/supabase/types.ts` 관련 타입 에러 0건
  - 조건: `server.ts`에서 `createClient()` 반환 타입 확인
  - 결과: `SupabaseClient<Database>` 타입으로 추론됨

---

## Phase 2: Auth 구현 (Backend + Frontend 협업)

### Task 2-1: Auth 테스트 작성 (QA 담당)
- `lib/validators/__tests__/auth.test.ts` — 이메일/비밀번호 검증 테스트
- `components/auth/__tests__/login-form.test.tsx` — 로그인 폼 렌더링/제출/에러 테스트
- `components/auth/__tests__/signup-form.test.tsx` — 회원가입 폼 테스트
- **성공기준**:
  - 조건: `bun test --run` 실행
  - 결과: auth 관련 테스트 파일 3개 실행, 전체 FAIL (구현체 미존재), exit code 1

### Task 2-2: 미들웨어 (Backend 담당)
- `middleware.ts` (프로젝트 루트):
  - 세션 리프레시 (`supabase.auth.getUser()`)
  - 라우트 보호: 미인증 → `/login` 리다이렉트
  - 인증 사용자의 `/login`, `/signup` 접근 → `/` 리다이렉트
  - matcher: 정적 자산 제외
- **성공기준**:
  - 조건: 미인증 상태에서 `curl -I http://localhost:3000/` 요청
  - 결과: HTTP 307, `Location: /login` 헤더 포함
  - 조건: 미인증 상태에서 `curl -I http://localhost:3000/login` 요청
  - 결과: HTTP 200 (리다이렉트 없음, 로그인 페이지 정상 접근)

### Task 2-3: Auth 서버 액션 (Backend 담당)
- `app/(auth)/actions.ts`: `signUp`, `signIn`, `signOut`
  - 서버 사이드 검증 (이메일 필수, 비밀번호 6자 이상)
  - `revalidatePath("/", "layout")` + `redirect()`
- `app/auth/callback/route.ts`: 이메일 확인 콜백
- **성공기준**:
  - 조건: `signUp` 호출 — `email: "test@example.com"`, `password: "password123"`
  - 결과: Supabase Local `auth.users` 테이블에 해당 이메일의 유저 레코드 생성
  - 조건: `signIn` 호출 — `email: "test@example.com"`, `password: "password123"`
  - 결과: 세션 쿠키 설정, `/`로 리다이렉트
  - 조건: `signUp` 호출 — `email: ""`, `password: "12"`
  - 결과: `{ error: "..." }` 반환, DB에 레코드 생성되지 않음

### Task 2-4: Auth 페이지 (Frontend 담당)
- `app/(auth)/layout.tsx` — 중앙 정렬 레이아웃
- `app/(auth)/login/page.tsx` + `login-form.tsx` — 로그인 폼 (useActionState)
- `app/(auth)/signup/page.tsx` + `signup-form.tsx` — 회원가입 폼
- 사용 컴포넌트: Card, Input, Label, Button (기존)
- **성공기준**:
  - 조건: `bun test --run` 실행 (QA의 auth 테스트)
  - 결과: `login-form.test.tsx`, `signup-form.test.tsx` 모든 케이스 PASS
  - 조건: 브라우저에서 `/signup` 접속 → 이메일/비밀번호 입력 후 제출
  - 결과: 회원가입 성공 후 `/login`으로 이동 또는 자동 로그인

### Task 2-5: Auth 통합 테스트 (QA 담당)
- `__tests__/integration/auth-flow.test.tsx`: 가입→로그인→로그아웃→라우트보호 시나리오
- **성공기준**:
  - 조건: `bun test __tests__/integration/auth-flow.test.tsx --run` 실행
  - 결과: 모든 테스트 케이스 PASS, exit code 0

**커밋**: "feat: implement email/password auth with Supabase Local"

---

## Phase 3: 레이아웃 + 사이드바 (Frontend 담당)

### Task 3-1: shadcn 추가 컴포넌트 설치
- shadcn MCP로 적합한 컴포넌트 탐색 후 설치:
  - checkbox, dialog, sheet, switch, tabs, popover, tooltip, skeleton, scroll-area, calendar, sidebar
- `bun add date-fns` (날짜 포맷팅)
- **성공기준**:
  - 조건: `ls components/ui/` 실행
  - 결과: `checkbox.tsx`, `dialog.tsx`, `sheet.tsx`, `switch.tsx`, `tabs.tsx`, `popover.tsx`, `tooltip.tsx`, `skeleton.tsx`, `scroll-area.tsx`, `calendar.tsx`, `sidebar.tsx` 파일 모두 존재
  - 조건: `bun run build` 실행
  - 결과: exit code 0, 빌드 성공

### Task 3-2: ThemeProvider
- `components/providers/theme-provider.tsx` — dark 클래스 토글, localStorage 지속
- `hooks/use-theme.ts` — theme/toggleTheme 훅
- `app/layout.tsx` 수정: ThemeProvider 래핑, metadata 업데이트
- **성공기준**:
  - 조건: 브라우저에서 `toggleTheme()` 호출 (또는 UI 토글 클릭)
  - 결과: `<html>` 태그에 `class="dark"` 추가/제거, 배경색 변경
  - 조건: 다크모드 상태에서 페이지 새로고침
  - 결과: `localStorage.getItem("theme")` === `"dark"`, `<html class="dark">` 유지

### Task 3-3: 사이드바 테스트 (QA 담당)
- `components/layout/__tests__/app-sidebar.test.tsx`: 네비게이션 렌더링, 활성 상태, 로그아웃 테스트
- **성공기준**:
  - 조건: `bun test components/layout/__tests__/app-sidebar.test.tsx --run` 실행
  - 결과: 전체 FAIL (구현체 미존재), exit code 1, 각 테스트 케이스별 실패 사유 출력

### Task 3-4: 사이드바 + 대시보드 레이아웃 (Frontend 담당)
- `components/layout/app-sidebar.tsx` — shadcn Sidebar 활용:
  - 헤더: 앱 이름 + 사용자 Avatar
  - 필터 뷰: All, Today, Upcoming, Completed (Phosphor 아이콘)
  - 카테고리 목록 (색상 dot)
  - 푸터: Settings 링크, Sign Out
- `components/layout/top-bar.tsx` — 모바일 햄버거 + 페이지 제목
- `app/(protected)/layout.tsx` — SidebarProvider + AppSidebar + SidebarInset
- **성공기준**:
  - 조건: `bun test components/layout/__tests__/app-sidebar.test.tsx --run` 실행
  - 결과: 모든 테스트 케이스 PASS, exit code 0
  - 조건: 브라우저 1024px 너비에서 `/` 접속 (로그인 상태)
  - 결과: 좌측에 사이드바 표시 — "All", "Today", "Upcoming", "Completed" 네비게이션 링크, "Settings", "Sign Out" 버튼 표시
  - 조건: 브라우저 375px 너비에서 `/` 접속
  - 결과: 사이드바 숨김, 상단에 햄버거 메뉴 아이콘 표시

**커밋**: "feat: add sidebar layout with navigation and theme toggle"

---

## Phase 4: Todo CRUD (Backend + Frontend + QA 협업)

### Task 4-1: Todo 서버 액션 (Backend 담당)
- `app/(protected)/actions.ts`: `getTodos`, `createTodo`, `updateTodo`, `toggleTodo`, `deleteTodo`
  - `getTodos`: nested select `todos` + `categories` + `todo_tags(tags(*))`
  - `user_id`는 서버에서 `getUser()`로 설정
- `app/(protected)/category-actions.ts`: `getCategories`, `createCategory`, `deleteCategory`
- `app/(protected)/tag-actions.ts`: `getTags`, `createTag`, `addTagToTodo`, `removeTagFromTodo`
- **성공기준**:
  - 조건: 로그인된 유저로 `createTodo({ title: "테스트 할일", priority: "high" })` 호출
  - 결과: `todos` 테이블에 `title="테스트 할일"`, `priority="high"`, `user_id=<현재유저>` 레코드 생성, 생성된 todo 객체 반환
  - 조건: `getTodos()` 호출
  - 결과: 현재 유저의 todo만 반환, 각 todo에 `categories`, `todo_tags.tags` 중첩 데이터 포함
  - 조건: `toggleTodo(todoId)` 호출 (기존 `completed: false`)
  - 결과: `completed: true`로 변경, `updated_at` 갱신
  - 조건: `deleteTodo(todoId)` 호출
  - 결과: 해당 todo 및 연관 `todo_tags` 레코드 삭제 (CASCADE 또는 명시적 삭제)
  - 조건: `createCategory({ name: "업무", color: "#FF0000" })` 호출
  - 결과: `categories` 테이블에 레코드 생성
  - 조건: 동일 유저가 `createCategory({ name: "업무", color: "#00FF00" })` 재호출
  - 결과: unique 제약 위반 에러 반환

### Task 4-2: Todo 컴포넌트 테스트 (QA 담당)
- `components/todo/__tests__/todo-item.test.tsx`: 렌더링, 체크박스, 우선순위 뱃지, 삭제 확인
- `components/todo/__tests__/todo-list.test.tsx`: 리스트 렌더링, 빈 상태, 로딩 스켈레톤
- `components/todo/__tests__/todo-create-dialog.test.tsx`: 폼 검증, 제출
- **성공기준**:
  - 조건: `bun test components/todo/__tests__/ --run` 실행
  - 결과: 3개 테스트 파일 실행, 전체 FAIL (구현체 미존재), exit code 1

### Task 4-3: TodoItem 컴포넌트 (Frontend 담당)
- `components/todo/todo-item.tsx`:
  - Checkbox + 제목 (완료 시 line-through) + Priority Badge + Category Badge + 마감일
  - DropdownMenu: Edit, Delete (destructive)
  - 클릭 시 설명 확장
- `components/todo/todo-empty-state.tsx` — 빈 상태 메시지
- `components/todo/todo-skeleton.tsx` — 로딩 스켈레톤
- **성공기준**:
  - 조건: `bun test components/todo/__tests__/todo-item.test.tsx --run` 실행
  - 결과: 모든 테스트 PASS
  - 조건: `{ title: "할일", completed: true, priority: "high" }` 데이터로 TodoItem 렌더링
  - 결과: 제목에 `line-through` 스타일, 체크박스 checked, "high" 우선순위 뱃지 표시

### Task 4-4: TodoList + 검색/필터 (Frontend 담당)
- `components/todo/todo-list.tsx` — 필터/정렬된 아이템 리스트, ScrollArea
- `components/todo/todo-search-bar.tsx` — InputGroup + 디바운스 검색 (300ms)
- `components/todo/todo-filter-chips.tsx` — 상태/우선순위/카테고리 필터 칩
- `components/todo/todo-sort-select.tsx` — 정렬 (마감일/우선순위/생성일)
- **성공기준**:
  - 조건: `bun test components/todo/__tests__/todo-list.test.tsx --run` 실행
  - 결과: 모든 테스트 PASS
  - 조건: 검색바에 "장보기" 입력 후 300ms 대기
  - 결과: 제목에 "장보기"를 포함하는 todo만 리스트에 표시
  - 조건: 우선순위 필터에서 "high" 선택
  - 결과: `priority: "high"`인 todo만 표시
  - 조건: 정렬을 "마감일순"으로 변경
  - 결과: `due_date` 오름차순 정렬 (null은 맨 뒤)

### Task 4-5: Todo 생성/수정/삭제 (Frontend 담당)
- `components/todo/date-picker.tsx` — Popover + Calendar
- `components/todo/todo-create-dialog.tsx` — Dialog 기반 생성 폼 (title, description, priority, due_date, category, tags)
- `components/todo/todo-edit-dialog.tsx` — 기존 데이터 프리필 수정 폼
- `components/todo/todo-delete-dialog.tsx` — AlertDialog 삭제 확인
- **성공기준**:
  - 조건: `bun test components/todo/__tests__/todo-create-dialog.test.tsx --run` 실행
  - 결과: 모든 테스트 PASS
  - 조건: 생성 다이얼로그에서 `title: ""` (빈 제목)으로 제출 시도
  - 결과: 제출 차단, "제목을 입력해주세요" 등 검증 에러 메시지 표시
  - 조건: 수정 다이얼로그 열기 — 기존 todo `{ title: "기존 할일", priority: "medium" }`
  - 결과: 폼 필드에 기존 값 프리필 — title 입력란에 "기존 할일", priority에 "medium" 선택 상태
  - 조건: 삭제 다이얼로그에서 "삭제" 버튼 클릭
  - 결과: 해당 todo가 리스트에서 제거, DB에서도 삭제

### Task 4-6: 대시보드 메인 페이지 조립 (Frontend 담당)
- `app/(protected)/page.tsx`:
  - TopBar (검색 + 생성 버튼)
  - TodoFilterChips + TodoSortSelect
  - TodoList
  - TodoCreateDialog
- **성공기준**:
  - 조건: 로그인 후 `/` 접속
  - 결과: 상단에 검색바 + "새 할일" 버튼, 그 아래 필터 칩과 정렬 셀렉트, 메인 영역에 TodoList 렌더링
  - 조건: "새 할일" 버튼 클릭 → 폼 입력 → 제출
  - 결과: 다이얼로그 닫힘, TodoList에 새 할일 즉시 표시
  - 조건: todo 항목의 체크박스 클릭
  - 결과: 완료 상태 토글, 제목에 line-through 스타일 적용/해제

### Task 4-7: Todo CRUD 통합 테스트 (QA 담당)
- `__tests__/integration/todo-crud.test.tsx`: 생성→조회→수정→삭제 시나리오
- `__tests__/integration/filter-search.test.tsx`: 필터+검색 조합 테스트
- **성공기준**:
  - 조건: `bun test __tests__/integration/ --run` 실행
  - 결과: `todo-crud.test.tsx`, `filter-search.test.tsx` 모든 테스트 PASS, exit code 0

**커밋**: "feat: implement todo CRUD with search, filter, sort"

---

## Phase 5: Settings 페이지 (Frontend + QA)

### Task 5-1: Settings 테스트 (QA 담당)
- `components/settings/__tests__/theme-toggle.test.tsx`: 테마 토글 + 지속성
- `components/settings/__tests__/password-form.test.tsx`: 비밀번호 변경 검증
- **성공기준**:
  - 조건: `bun test components/settings/__tests__/ --run` 실행
  - 결과: 2개 테스트 파일 실행, 전체 FAIL (구현체 미존재), exit code 1

### Task 5-2: Settings 컴포넌트 (Frontend 담당)
- `components/settings/theme-toggle.tsx` — Switch + SunIcon/MoonIcon
- `components/settings/password-change-form.tsx` — 현재/새/확인 비밀번호 (Card + FieldGroup)
- `components/settings/email-verification-status.tsx` — 이메일 + 인증 상태 Badge
- `components/settings/terms-of-service.tsx` — ScrollArea 기반 약관 표시
- **성공기준**:
  - 조건: `bun test components/settings/__tests__/ --run` 실행
  - 결과: 모든 테스트 PASS, exit code 0
  - 조건: 테마 토글 Switch 클릭
  - 결과: `<html>` 클래스 `dark` 추가/제거, Switch 상태 반영 (on=dark, off=light)
  - 조건: 비밀번호 변경 폼에서 새 비밀번호 `"ab"` (2자) 입력 후 제출
  - 결과: 제출 차단, "비밀번호는 6자 이상이어야 합니다" 에러 메시지 표시
  - 조건: 새 비밀번호와 확인 비밀번호 불일치 상태에서 제출
  - 결과: "비밀번호가 일치하지 않습니다" 에러 메시지 표시

### Task 5-3: Settings 페이지 조립 (Frontend 담당)
- `app/(protected)/settings/page.tsx`:
  - Tabs: Appearance (테마 토글) | Account (비밀번호 변경 + 이메일 확인) | Legal (이용약관)
- **성공기준**:
  - 조건: 브라우저에서 `/settings` 접속
  - 결과: 3개 탭("Appearance", "Account", "Legal") 표시, 기본 선택 탭은 "Appearance"
  - 조건: "Account" 탭 클릭
  - 결과: 비밀번호 변경 폼 + 이메일 인증 상태 Badge 표시
  - 조건: "Legal" 탭 클릭
  - 결과: ScrollArea 내에 이용약관 텍스트 표시

### Task 5-4: Settings 통합 테스트 (QA 담당)
- `__tests__/integration/settings-updates.test.tsx`: 테마 변경, 비밀번호 변경 시나리오
- **성공기준**:
  - 조건: `bun test __tests__/integration/settings-updates.test.tsx --run` 실행
  - 결과: 모든 테스트 PASS, exit code 0

**커밋**: "feat: add settings page with theme, password, terms"

---

## Phase 6: 마무리 (Leader 조율)

### Task 6-1: 정리
- 예제 파일 삭제: `components/component-example.tsx`, `components/example.tsx`
- `app/page.tsx`를 `(protected)` 그룹으로 이동 또는 리다이렉트
- **성공기준**:
  - 조건: `ls components/component-example.tsx components/example.tsx 2>&1`
  - 결과: "No such file or directory" — 예제 파일 삭제 확인
  - 조건: 미인증 상태에서 `/` 접근
  - 결과: `/login`으로 리다이렉트 (protected 라우트 정상 동작)

### Task 6-2: 모바일 반응형 확인
- 사이드바 → Sheet로 축소 (768px 이하)
- Todo 생성 버튼 → 플로팅 FAB
- 필터 칩 → 수평 스크롤
- **성공기준**:
  - 조건: 브라우저 뷰포트 375×667 (iPhone SE) 설정
  - 결과: 사이드바 숨김 + 햄버거 메뉴로 접근, 우하단 FAB으로 todo 생성, 필터 칩 영역 수평 스크롤 가능
  - 조건: 375px 뷰포트에서 todo 생성 → 수정 → 삭제 플로우 수행
  - 결과: 모든 다이얼로그가 화면 내에 표시, 잘림 없이 정상 동작

### Task 6-3: 최종 검증
- `bun test` — 모든 테스트 통과
- `bun run build` — 빌드 성공
- `bun run lint` — 린트 에러 없음
- **성공기준**:
  - 조건: `bun test --run` 실행
  - 결과: 전체 테스트 PASS, exit code 0, 실패 0건
  - 조건: `bun run build` 실행
  - 결과: "✓ Compiled successfully" 또는 "Build completed", exit code 0
  - 조건: `bun run lint` 실행
  - 결과: 경고/에러 0건, exit code 0

**커밋**: "chore: cleanup example code and verify build"

---

## Phase 7: E2E 테스트 — agent-browser (Leader 조율)

모든 구현이 완료된 후, `agent-browser` 스킬을 사용하여 실제 브라우저에서 전체 앱의 E2E 테스트를 수행한다.

### 사전 조건
- `bun dev` 실행 중 (http://localhost:3000)
- `npx supabase status` — Supabase Local 서비스 실행 중
- Chrome 브라우저에 Claude in Chrome 확장 프로그램 활성화

### Task 7-1: Auth E2E 테스트
agent-browser를 통해 실제 브라우저에서 인증 플로우를 검증한다.

- **테스트 시나리오 A: 회원가입**
  - 조건: `http://localhost:3000/signup` 페이지 접속
  - 행동: 이메일 `e2e-test@example.com`, 비밀번호 `testpass123` 입력 후 회원가입 버튼 클릭
  - 결과: 페이지가 `/login` 또는 `/`로 이동, 에러 메시지 없음

- **테스트 시나리오 B: 로그인**
  - 조건: `http://localhost:3000/login` 페이지 접속
  - 행동: 이메일 `e2e-test@example.com`, 비밀번호 `testpass123` 입력 후 로그인 버튼 클릭
  - 결과: `/` (대시보드)로 이동, 사이드바에 사용자 정보 표시

- **테스트 시나리오 C: 미인증 접근 차단**
  - 조건: 로그아웃 상태에서 `http://localhost:3000/` 접속
  - 행동: URL 직접 입력
  - 결과: `/login` 페이지로 리다이렉트

- **테스트 시나리오 D: 로그인 검증 오류**
  - 조건: `/login` 페이지에서 이메일 빈칸, 비밀번호 `"ab"` 입력 후 제출
  - 행동: 잘못된 자격증명으로 로그인 시도
  - 결과: 페이지 이동 없음, 에러 메시지 표시

### Task 7-2: Todo CRUD E2E 테스트
로그인 상태에서 Todo 생성/조회/수정/삭제를 검증한다.

- **테스트 시나리오 A: Todo 생성**
  - 조건: 대시보드(`/`) 로그인 상태
  - 행동: "새 할일" 버튼 클릭 → title `"E2E 테스트 할일"`, priority `"high"`, description `"자동 테스트"` 입력 → 제출
  - 결과: 다이얼로그 닫힘, 리스트에 "E2E 테스트 할일" 항목 표시, "high" 우선순위 뱃지 표시

- **테스트 시나리오 B: Todo 완료 토글**
  - 조건: "E2E 테스트 할일" 항목이 리스트에 존재
  - 행동: 해당 항목의 체크박스 클릭
  - 결과: 제목에 line-through 스타일 적용, 체크박스 checked 상태

- **테스트 시나리오 C: Todo 수정**
  - 조건: "E2E 테스트 할일" 항목의 DropdownMenu 열기
  - 행동: "Edit" 클릭 → title을 `"수정된 할일"`로 변경 → 제출
  - 결과: 리스트에 "수정된 할일"로 업데이트, "E2E 테스트 할일" 제목 사라짐

- **테스트 시나리오 D: Todo 삭제**
  - 조건: "수정된 할일" 항목의 DropdownMenu 열기
  - 행동: "Delete" 클릭 → 확인 다이얼로그에서 "삭제" 클릭
  - 결과: 해당 항목 리스트에서 사라짐

- **테스트 시나리오 E: 빈 상태 확인**
  - 조건: 모든 todo 삭제 후
  - 행동: 대시보드 확인
  - 결과: 빈 상태 메시지 표시 (예: "할일이 없습니다")

### Task 7-3: 검색/필터/정렬 E2E 테스트
여러 todo를 생성한 뒤 검색, 필터, 정렬 기능을 검증한다.

- **테스트 데이터 준비**:
  - todo 3개 생성: `"장보기"` (priority: low), `"보고서 작성"` (priority: high, due_date: 오늘), `"운동하기"` (priority: medium)

- **테스트 시나리오 A: 검색**
  - 조건: 3개 todo 생성 완료
  - 행동: 검색바에 `"장보기"` 입력
  - 결과: "장보기" 항목만 리스트에 표시, "보고서 작성"과 "운동하기"는 숨김

- **테스트 시나리오 B: 우선순위 필터**
  - 조건: 검색바 비움
  - 행동: 우선순위 필터에서 "high" 선택
  - 결과: "보고서 작성"만 표시

- **테스트 시나리오 C: 필터 해제**
  - 조건: "high" 필터 활성 상태
  - 행동: 필터 해제 (칩 클릭 또는 "전체" 선택)
  - 결과: 3개 todo 모두 다시 표시

### Task 7-4: Settings E2E 테스트

- **테스트 시나리오 A: Settings 페이지 접근**
  - 조건: 로그인 상태에서 사이드바의 "Settings" 클릭
  - 행동: Settings 링크 클릭
  - 결과: `/settings` 페이지 이동, "Appearance" 탭 기본 표시

- **테스트 시나리오 B: 다크모드 토글**
  - 조건: `/settings` 페이지, "Appearance" 탭
  - 행동: 테마 토글 Switch 클릭
  - 결과: 페이지 전체 다크모드 전환, 배경색 어두워짐

- **테스트 시나리오 C: 탭 전환**
  - 조건: `/settings` 페이지
  - 행동: "Account" 탭 클릭 → "Legal" 탭 클릭
  - 결과: 각 탭에 해당하는 콘텐츠 표시 — Account에 비밀번호 폼, Legal에 이용약관

- **테스트 시나리오 D: 비밀번호 변경 검증**
  - 조건: "Account" 탭
  - 행동: 새 비밀번호 `"ab"` 입력 후 제출
  - 결과: 검증 에러 메시지 표시, 비밀번호 변경되지 않음

### Task 7-5: 로그아웃 E2E 테스트

- **테스트 시나리오**:
  - 조건: 로그인 상태, 대시보드 페이지
  - 행동: 사이드바의 "Sign Out" 버튼 클릭
  - 결과: `/login` 페이지로 이동, 이후 `/` 접근 시 다시 `/login`으로 리다이렉트

### E2E 최종 성공기준
- 조건: Task 7-1 ~ 7-5 모든 시나리오 실행
- 결과: 전체 시나리오 통과, 실패 0건, 각 시나리오별 스크린샷 또는 GIF 기록

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

1. **Supabase**: `npx supabase status` → `API URL`, `DB URL`, `anon key` 출력 확인
2. **Auth**: 브라우저에서 회원가입 → 로그인 → 로그아웃 플로우 — 각 단계에서 기대 URL 이동 확인
3. **Todo CRUD**: 생성 → 수정 → 완료 토글 → 삭제 → 검색/필터 — 각 동작 후 리스트 상태 확인
4. **Settings**: 테마 토글 → 새로고침 후 `localStorage` 및 `<html>` 클래스 유지, 비밀번호 폼 검증 에러 표시
5. **단위 테스트**: `bun test --run` → 전체 PASS, exit code 0
6. **빌드/린트**: `bun run build && bun run lint` → 둘 다 exit code 0
7. **E2E**: agent-browser로 Task 7-1 ~ 7-5 전체 시나리오 통과
