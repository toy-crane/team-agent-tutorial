팀을 구성하여 대시보드 페이지를 구현해줘.

## 팀 구성

```
        [Team Leader]
       /             \
[UI Developer]   [Page Developer]
```

- Team Leader가 전체 작업을 조율하고, 팀메이트 간 통신은 리더를 통해서만 진행
- UI Developer: shadcn 컴포넌트 탐색 및 추가, 재사용 가능한 UI 컴포넌트 구현
- Page Developer: 페이지 라우팅, 레이아웃 구성, 컴포넌트 조합

## 요구사항

### 대시보드 페이지 (`/dashboard`)

1. **사이드바 네비게이션**
   - 로고 영역
   - 메뉴 항목: Dashboard, Projects, Tasks, Settings
   - Phosphor Icons 사용
   - 현재 페이지 활성 상태 표시

2. **통계 카드 섹션** (상단 4개 카드 그리드)
   - 총 프로젝트 수, 완료된 태스크, 진행 중, 팀 멤버 수
   - 각 카드에 아이콘과 수치 표시
   - shadcn Card 컴포넌트 활용

3. **최근 활동 리스트**
   - 더미 데이터 5개 항목
   - 활동 유형별 Badge 표시 (완료/진행중/대기)
   - 날짜, 설명, 담당자 정보 포함

4. **빠른 작업 영역**
   - "새 프로젝트 추가" 버튼 (AlertDialog로 확인)
   - 검색 입력 필드 (Input + MagnifyingGlassIcon)

## 기술 제약

- 기존 `components/ui/` 컴포넌트 최대한 활용
- 부족한 shadcn 컴포넌트는 shadcn MCP로 탐색 후 추가
- `cn()` 유틸리티로 조건부 스타일링
- 모든 컴포넌트는 `data-slot` 패턴 준수
- 반응형 레이아웃 (모바일: 1열, 데스크톱: 사이드바 + 메인)

## 작업 완료 기준

- `bun run build` 에러 없이 통과
- `/dashboard` 경로 접근 시 정상 렌더링
- 각 팀메이트는 작업 단위마다 커밋
