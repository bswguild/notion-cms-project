# ROADMAP: 노션 레시피 웹/PDF 서비스 (MVP)

> 이 로드맵은 [`docs/PRD.md`](./PRD.md)의 MVP 범위(F001~F011)와 기술 설계 방향(§5~6), 리스크(§9)를 바탕으로 작성되었다. 각 Phase는 이전 Phase의 결과물에 의존하므로 순서대로 진행한다.

- 전체 예상 소요: **9~14일** (1인 개발자 기준)
- 전제: Phase 2 착수 전까지 Notion Integration 생성과 레시피 데이터베이스(§5.4 속성 설계)가 준비되어 있어야 한다.

## 전체 일정 개요

| Phase | 내용 | 예상 소요 | 관련 기능 |
|---|---|---|---|
| 1 | 프로젝트 초기 설정 | 1~2일 | 기반 작업 |
| 2 | 공통 모듈 개발 | 2~3일 | F007 일부 |
| 3 | 핵심 기능 개발 | 3~4일 | F001, F003, F010, F011 |
| 4 | 추가 기능 개발 | 2~3일 | F002, F004, F005, F006, F009 |
| 5 | 최적화 및 배포 | 1~2일 | F008, 비기능 요구사항(§7) |

---

## Phase 1: 프로젝트 초기 설정 (1~2일)

**목표**: Notion 연동과 캐싱 모델을 실제로 동작시킬 수 있는 최소 기반을 만든다. 이 저장소는 이미 레이아웃/다크모드/폼 검증이 갖춰진 스타터킷이므로, 이번 Phase는 "레이아웃 신규 제작"이 아니라 **Notion 연동 환경 구축**에 집중한다.

### 작업 항목
- [ ] Notion Integration 생성(Internal, Read content 권한만) 및 레시피 데이터베이스에 Connection으로 연결(PRD §5.2)
- [ ] 레시피 데이터베이스에 §5.4 속성(`Name`, `Slug`, `Published`, `PublishedAt`, `Category`, `Tags`, `Difficulty`, `CookTime`, `Servings`, `Cover`) 생성, `Cover`는 external 타입 운영 규칙 확정
- [ ] 신규 의존성 설치: `@notionhq/client`(**v5.0.0 이상**), `zustand` — 현재 `package.json`에 없음(PRD §9 "미설치 의존성" 리스크)
- [ ] `.env.local` 생성 후 `NOTION_API_KEY`, `NOTION_RECIPES_DATA_SOURCE_ID` 등록(`NEXT_PUBLIC_` 접두사 금지, PRD §5.3)
- [ ] `databases.retrieve` 1회 호출로 데이터소스 ID 확인 → 환경변수로 고정
- [ ] `lib/notion/` 디렉토리 뼈대 생성, 환경변수 zod 스키마로 1차 검증만 먼저 구현(클라이언트 본체는 Phase 2)
- [ ] `app/recipes/` 라우트 스켈레톤(빈 페이지)만 생성해 기존 `ui → common → layout → sections` 계층에 연결점 확보

### 완료 기준
- `databases.retrieve`가 로컬에서 성공적으로 응답하고 `data_sources` 배열에서 ID를 확인할 수 있다.
- `.env.local`이 git에 커밋되지 않음을 확인했다(`.gitignore`에 이미 포함).
- `npm run dev`로 빈 `/recipes` 페이지가 에러 없이 뜬다.

### 선행 리스크
- Integration을 Database에 연결하지 않으면 `object_not_found`류 오류가 날 수 있음(PRD §5.2) — 연결 여부부터 확인.

---

## Phase 2: 공통 모듈 개발 (2~3일)

**목표**: 모든 기능이 재사용할 Notion 데이터 접근 계층과 공통 UI 부품을 먼저 만들어 중복을 방지한다.

### 작업 항목
- [ ] `lib/notion/client.ts`: `Notion-Version: 2025-09-03` 고정, `dataSources.query` 기반 클라이언트(PRD §5.1)
- [ ] `lib/notion/parser.ts`: Notion 응답(discriminated union)을 `any` 없이 타입가드로 파싱
- [ ] `lib/notion/schema.ts`: zod 스키마로 `RecipeSummary`, `RecipeDetail` 도메인 타입 검증(PRD §6 타입 안전성)
- [ ] `lib/notion/recipes.ts`: 캐시된 도메인 함수 `getRecipes()`(파라미터 없이 전체 공개 목록만), `getRecipeBySlug(slug)` — `"use cache"` + 커스텀 `cacheLife`(30~45분) + `cacheTag` 적용(PRD §6)
- [ ] `lib/notion/blocks.ts`: 상세 페이지용 블록 트리 재귀 조회 함수
- [ ] 공통 컴포넌트: `components/sections/recipes/recipe-card.tsx`, `components/common/`에 범용 Notion 블록 렌더러(불릿/번호 리스트 → 재료/조리순서 표시)
- [ ] 빈 상태/에러 상태 공통 UI(F006) — 기존 `components/common/empty-state.tsx` 확장 또는 재사용
- [ ] 공통 타입 정의: `types/recipe.ts`에 `RecipeSummary`, `RecipeDetail`, `Category`, `Tag`

### 완료 기준
- `getRecipes()`를 호출하면 실제 Notion DB의 `Published=true` 레시피만 zod 검증을 통과해 반환된다.
- 속성 값이 누락된 행이 있어도 해당 행만 건너뛰고 나머지는 정상 반환된다(§7 안정성 요구사항).
- 레시피 카드 컴포넌트가 스토리성 더미 데이터로 독립 렌더링된다.

### 선행 리스크
- `"use cache"` 스코프 내부에서 `searchParams`/`cookies()`를 쓸 수 없으므로, 이 단계에서 필터 파라미터를 넘기는 구조로 설계하지 않는다(PRD §9 "캐싱-필터 상호작용").

---

## Phase 3: 핵심 기능 개발 (3~4일)

**목표**: 방문자가 레시피를 찾아 상세 내용을 볼 수 있는, 서비스의 가장 기본적인 흐름을 완성한다.

### 작업 항목
- [ ] `/recipes` 목록 페이지: Phase 2의 `getRecipes()` 사용, `PublishedAt` 기준 최신순 정렬(F011)
- [ ] `/recipes/[slug]` 상세 페이지: 제목, 대표 이미지, 재료, 조리 순서, 조리시간/난이도/인분 표시(F003, F010)
- [ ] Notion 본문 블록(불릿/번호 리스트) → 재료·조리순서 렌더링 연결
- [ ] 목록/상세 페이지에 Phase 2의 빈 상태·에러 상태 UI 연결(F006): 레시피 없음, Notion 연동 실패 케이스
- [ ] 100개 초과 시 커서 기반 페이지네이션으로 전체 수집 확인(PRD §5.5)

### 완료 기준
- 운영자가 Notion에서 `Published`를 체크한 레시피가 목록/상세 페이지에 실제로 노출된다(PRD §8 성공 지표 1번).
- 초안(`Published=false`) 레시피는 어떤 경로로도 노출되지 않는다.
- 슬러그가 존재하지 않는 `/recipes/[slug]` 접근 시 깨지지 않고 에러 상태 UI가 뜬다.

### 선행 리스크
- Rate limit(초당 약 3회)에 걸리지 않도록 페이지네이션/상세 조회 시 재시도·캐싱이 이미 Phase 2에서 적용돼 있어야 한다.

---

## Phase 4: 추가 기능 개발 (2~3일)

**목표**: 핵심 열람 흐름 위에 탐색성(필터)과 MVP의 차별 기능(PDF, 이미지 안정화)을 추가한다.

### 작업 항목
- [ ] 카테고리/태그 필터링(F002): Zustand 스토어로 UI 상태 관리 + URL 쿼리스트링 동기화, 캐시된 전체 목록을 받은 **이후** 클라이언트/캐시 바깥 레이어에서 필터링(PRD §6)
- [ ] 이미지 안정적 표시(F004): `Cover`가 external URL인 케이스를 1차 기본 경로로 구현. Notion 네이티브 업로드 폴백이 필요하면 `app/api/recipe-image/route.ts`를 "요청 시점 재조회" 방식으로 구현(단순 중계 금지, PRD §6)
- [ ] PDF 다운로드(F005): 상세 페이지에 `window.print()` 버튼 + `@media print` 전용 스타일
- [ ] 다크모드 대응(F009): 신규 레시피 컴포넌트 전체에 기존 `ThemeProvider`(class 전략)와 통일된 `dark:` variant 적용
- [ ] 온디맨드 갱신용 `/api/revalidate` 라우트: 시크릿 검증 + `revalidateTag(tag, profile)` (PRD §6, 레거시 라우트 세그먼트 설정 사용 금지)

### 완료 기준
- 카테고리/태그를 변경하면 URL이 갱신되고, 새로고침해도 같은 필터 상태가 유지된다.
- 상세 페이지에서 PDF 다운로드(인쇄 다이얼로그)가 한글 포함 레시피에서도 정상 동작한다(PRD §8 성공 지표 2번).
- 대표 이미지가 캐시 만료 전/후 모두 깨지지 않는다(PRD §8 성공 지표 4번).
- 다크모드 전환 시 신규 컴포넌트가 기존 페이지들과 시각적으로 통일된다.

### 선행 리스크
- `window.print()` 기반 PDF는 브라우저별 인쇄 스타일 차이가 있을 수 있음(PRD §9) — MVP에서는 허용 범위로 간주.

---

## Phase 5: 최적화 및 배포 (1~2일)

**목표**: 기능이 완성된 상태를 실제 운영 환경에서도 깨지지 않게 다듬고 배포한다.

### 작업 항목
- [ ] 반응형 디자인 최종 점검(F008): 모바일/태블릿/데스크톱에서 목록/상세/필터 UI 레이아웃 확인
- [ ] 성능 최적화: 커스텀 `cacheLife` 프로필(30~45분) 값 튜닝, Notion API 호출 횟수 실측
- [ ] 배포 타깃 확정(Vercel 등 서버리스 여부) 후, 서버리스 환경에서 캐시가 재검증 사이에 유지되지 않는 문제(PRD §9)가 실제로 발생하는지 점검
- [ ] 신규 Route Handler(`/api/revalidate`, 이미지 프록시)가 `cacheComponents: true`와 충돌하는 레거시 설정을 쓰지 않는지 최종 확인
- [ ] 환경변수(`NOTION_API_KEY`, `NOTION_RECIPES_DATA_SOURCE_ID`, revalidate 시크릿)를 배포 플랫폼에 등록
- [ ] Vercel(또는 선택한 플랫폼)에 배포, 프로덕션 URL에서 목록→상세→PDF 흐름 재검증

### 완료 기준
- 프로덕션 환경에서 모바일/데스크톱 레이아웃이 깨지지 않는다(PRD §8 성공 지표 3번).
- 배포 후 Notion에서 레시피를 추가/수정해도 코드 수정이나 재배포 없이 캐시 주기 내에 반영된다(PRD §8 성공 지표 1번, §4 운영자 플로우).
- 프로덕션에서 PDF 다운로드와 이미지 표시가 로컬과 동일하게 동작한다.

### 선행 리스크
- 서버리스 콜드스타트로 캐시가 기대만큼 지속되지 않으면(PRD §9), 캐싱 주기나 동시성 제한을 재조정해야 한다.

---

## Phase 이후 (MVP 제외 범위, 참고용)

PRD §3 "제외" 항목은 이번 로드맵에 포함하지 않는다: 로그인/즐겨찾기/댓글/평점, 키워드 전문 검색, 관리자 어드민 UI, 재료 기반 구조화 검색, 서버 생성 PDF(`@react-pdf/renderer`), 다국어, 웹훅 기반 실시간 반영. 필요 시 PRD §10 오픈 질문을 먼저 확정한 뒤 별도 로드맵으로 분리한다.
