팀을 구성하여, Todo 앱 구현해줘.

## 팀 구성

```
       [Team Leader]
      /      |      \
[Backend] [Frontend] [QA]
```

- 팀 리더가 전체 task를 조율하고, 팀메이트 간 통신은 리더를 통해서만 진행
- Backend: Supabase Local, Auth, DB 스키마, RLS
- Frontend: 페이지, 컴포넌트, UI
- QA: TDD 테스트 작성, 검증

## 요구사항

1. Supabase Local로 Email/Password 기반 Auth 구현
2. Supabase RLS로 todo 관련 스키마 구성 (todos, categories, tags, todo_tags)
3. Todo List 주요 기능: 검색, 필터, 카테고리, 우선순위(높/중/낮), 마감일, 태그
4. Settings 페이지: 테마 토글, 이용약관, 이메일 확인, 비밀번호 변경
5. UI는 미니멀 Todoist 스타일 (사이드바 + 메인 리스트)
6. Shadcn MCP로 적합한 컴포넌트 탐색 및 적용
7. Vitest + React Testing Library로 TDD, 작업 단위마다 커밋
