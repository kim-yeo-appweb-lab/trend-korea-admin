# ROADMAP

> PRD 기반 개발 로드맵
> 생성일: 2026-03-09
> 기반 PRD: `docs/PRD.md`

## 개요

트렌드 코리아 서비스의 데이터 품질 관리, 커뮤니티 모더레이션, 파이프라인 모니터링을 위한 독립 어드민 풀스택 애플리케이션의 개발 로드맵.

### 기술 스택

| 레이어 | 기술 |
|--------|------|
| 프레임워크 | Next.js (App Router) — 풀스택 (페이지 + Route Handlers) |
| 언어 | TypeScript 5.9 (strict mode) |
| 스타일링 | Tailwind CSS 4 + shadcn/ui |
| ORM | Prisma (기존 PostgreSQL DB introspection) |
| 데이터 페칭 | TanStack Query (React Query) |
| 폼 검증 | Zod + React Hook Form |
| 테이블 | TanStack Table |
| JWT | jose (검증 전용, 발급은 trend-korea-api) |
| 패키지 매니저 | pnpm |
| 린터/포매터 | ESLint 9 (flat config) + Prettier |
| 테스트 | Vitest (단위/통합) + Playwright (E2E) |

### 테스트 전략

- **단위 테스트**: Route Handler 로직, 유틸 함수 (Vitest)
- **통합 테스트**: Route Handler + Prisma 연동 (Vitest + 테스트 DB)
- **E2E 테스트**: 주요 Userflow — CRUD, 로그인 (Playwright)
- **커버리지 기준**: Route Handler 80%+
- **테스트 트로피**: Unit 30% / Integration 40% / E2E 10% / Static Analysis 20%
- **Phase 1 기능 구현부터**: TDD 필수 (Red-Green-Refactor)

### 아키텍처 특징

- **독립 프로젝트**: trend-korea-app과 분리된 별도 Next.js 앱
- **Prisma DB introspection**: `prisma db pull`로 기존 PostgreSQL 스키마를 반영
- **공유 JWT**: trend-korea-api가 발급한 JWT를 검증만 수행 (자체 발급 없음)
- **Route Handlers**: 자체 API 계층 (30개 엔드포인트)

---

## Phase 요약

| Phase | 이름 | 설명 | 예상 기간 | TDD | 상태 |
|-------|------|------|-----------|-----|------|
| 1 | MVP | 프로젝트 초기화 + 사건/이슈/트리거 CRUD + 태그/출처 관리 (P0) | 3주 | 필수 | ⬜ |
| 2 | 운영 도구 | 대시보드 + 사용자 관리 + 파이프라인 모니터링 (P1) | 2주 | 필수 | ⬜ |
| 3 | 모더레이션 | 커뮤니티 모더레이션 + 신고 시스템 (P1) | 1.5주 | 필수 | ⬜ |

---

## Phase 1: MVP

**목표**: 프로젝트를 초기화하고, 사건/이슈/트리거/태그/출처 CRUD를 완성하여 DB 직접 접근 없이 데이터를 관리할 수 있게 한다
**예상 기간**: 3주
**선행 조건**: 없음
**TDD**: 필수 (Phase 1-2 골격/UI는 선택, Phase 1-3 기능 연동부터 필수)

### Phase 1-1: 프로젝트 골격

> 환경 설정, Prisma 연결, JWT 인증, 공통 레이아웃을 구성한다.

#### 태스크

- [ ] `P1-001` Next.js 프로젝트 초기화
  - **설명**: Next.js App Router 프로젝트를 생성하고 기본 설정을 구성한다
  - **영역**: 인프라
  - **구현 사항**:
    - `package.json` — Next.js, TypeScript, Tailwind CSS 4, ESLint 9, Prettier 의존성
    - `tsconfig.json` — strict mode, path alias (`@/`)
    - `next.config.ts` — 기본 설정
    - `tailwind.config.ts` — Tailwind CSS 4 설정
    - `.env.local.example` — 환경변수 템플릿 (`DATABASE_URL`, `JWT_SECRET`, `NEXT_PUBLIC_API_BASE_URL`)
    - `src/app/layout.tsx` — 루트 레이아웃
  - **예상 소요**: 1일
  - **의존성**: 없음

- [ ] `P1-002` shadcn/ui 초기화 및 공통 컴포넌트 설치
  - **설명**: shadcn/ui를 초기화하고 어드민에서 공통으로 사용할 컴포넌트를 설치한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `components.json` — shadcn/ui 설정
    - `src/shared/ui/` — Table, Form, Input, Select, Textarea, Button, Badge, Card, Dialog, AlertDialog, Tabs, Pagination, Sonner(Toast), DatePicker 설치
  - **예상 소요**: 0.5일
  - **의존성**: `P1-001`

- [ ] `P1-003` Prisma 설정 및 DB introspection
  - **설명**: Prisma를 설정하고 기존 PostgreSQL DB 스키마를 introspection하여 Client를 생성한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `prisma/schema.prisma` — `prisma db pull`로 생성
    - `src/shared/lib/prisma.ts` — Prisma Client 싱글톤 (개발 환경 hot reload 대응)
    - `package.json` scripts — `prisma db pull`, `prisma generate`, `prisma studio`
  - **예상 소요**: 1일
  - **의존성**: `P1-001`

- [ ] `P1-004` JWT 인증 미들웨어 구현
  - **설명**: jose 라이브러리로 JWT Access Token을 검증하고, admin role이 아닌 접근을 차단하는 미들웨어를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/shared/lib/auth.ts` — `verifyAdminToken()` 함수 (jose `jwtVerify`)
    - `src/middleware.ts` — Next.js 미들웨어 (경로별 보호 수준 적용)
    - 보호 범위: `/login` 미인증 허용, `/api/auth/me` JWT 검증(role 무관), `/api/*` 및 `/(admin)/*` admin role 필수
  - **테스트 (TDD)**:
    - 🔴 유효하지 않은 JWT로 `/api/events` 호출 시 401 반환 테스트
    - 🔴 `role=member`인 JWT로 `/api/events` 호출 시 403 반환 테스트
    - 🟢 jose `jwtVerify` + role 체크 로직 구현
    - 🔵 에러 메시지 표준화, 토큰 만료 시 리프레시 로직 검토
  - **예상 소요**: 1.5일
  - **의존성**: `P1-001`

- [ ] `P1-005` 로그인 페이지 구현
  - **설명**: trend-korea-api의 로그인 API를 호출하여 JWT를 받고, httpOnly 쿠키에 저장하는 로그인 페이지를 구현한다
  - **영역**: 프론트엔드 + 백엔드
  - **구현 사항**:
    - `src/app/(auth)/login/page.tsx` — 로그인 폼 (이메일 + 비밀번호)
    - `src/app/api/auth/me/route.ts` — 현재 로그인 어드민 정보 조회 Route Handler
    - `src/features/auth/` — 로그인 feature 모듈 (폼, 타입, API 호출)
    - 로그인 성공 시 `role=admin` 확인 후 대시보드(`/`)로 리다이렉트
    - `role != admin`이면 "권한 없음" 메시지 표시
  - **테스트 (TDD)**:
    - 🔴 로그인 성공 후 쿠키에 Access Token이 저장되는지 테스트
    - 🔴 `role != admin`일 때 "권한 없음" 메시지 렌더링 테스트
    - 🟢 로그인 폼 + API 호출 + 쿠키 저장 구현
    - 🔵 에러 핸들링 (네트워크 오류, 잘못된 자격 증명) 개선
  - **예상 소요**: 1.5일
  - **의존성**: `P1-004`

- [ ] `P1-006` 공통 레이아웃 (사이드바 + 헤더) 구현
  - **설명**: 어드민 전용 레이아웃으로 사이드바 네비게이션과 헤더를 구현한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/layout.tsx` — 어드민 레이아웃 (사이드바 + 헤더 + 메인 콘텐츠)
    - `src/shared/layouts/AdminSidebar.tsx` — 사이드바 (대시보드, 사건, 이슈, 트리거, 태그, 출처 메뉴 + 구분선 + 사용자, 모더레이션, 파이프라인)
    - `src/shared/layouts/AdminHeader.tsx` — 헤더 (로고, 현재 페이지명, 프로필 드롭다운)
    - 반응형: 데스크톱(1280px+) 사이드바 펼침, 태블릿(768px~) 사이드바 접기(아이콘만)
  - **예상 소요**: 1.5일
  - **의존성**: `P1-002`, `P1-004`

- [ ] `P1-007` Route Handler 헬퍼 및 공통 유틸
  - **설명**: Route Handler에서 반복 사용하는 JWT 검증, 에러 응답, Zod 검증 헬퍼를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/shared/lib/api.ts` — `withAuth()` 래퍼 (JWT 검증 + admin 체크), `apiError()` 표준 에러 응답, `parseBody()` Zod 검증 헬퍼
    - `src/shared/types/api.ts` — 공통 API 응답 타입 (`PaginatedResponse<T>`, `ApiError`)
    - `src/shared/utils/pagination.ts` — 페이지네이션 파라미터 파싱 유틸
  - **테스트 (TDD)**:
    - 🔴 `withAuth()` 래퍼가 JWT 없을 때 401 응답을 반환하는지 테스트
    - 🔴 `parseBody()`에 잘못된 데이터 전달 시 400 에러 반환 테스트
    - 🟢 헬퍼 함수 구현
    - 🔵 에러 코드 체계 표준화
  - **예상 소요**: 1일
  - **의존성**: `P1-004`

#### 완료 기준
- Next.js 프로젝트가 `pnpm dev`로 실행된다 (포트 3200)
- `prisma db pull`로 기존 DB 스키마가 반영된다
- `/login`에서 로그인 후 `/(admin)` 레이아웃이 렌더링된다
- `role != admin` 사용자는 어드민 페이지 접근이 차단된다
- 사이드바에서 각 관리 페이지로 이동할 수 있다
- `pnpm lint`, `pnpm type:check` 통과

---

### Phase 1-2: CRUD UI/UX 레이아웃

> 더미 데이터로 사건/이슈/트리거/태그/출처 관리 화면을 구성하여 UI/UX를 검증한다.
> 이 단계의 모든 UI는 더미/하드코딩 데이터로 동작한다. 실제 API 연동은 Phase 1-3에서 수행한다.

#### 태스크

- [ ] `P1-008` 사건 목록 페이지 UI
  - **설명**: 사건 목록을 데이터 테이블(TanStack Table)로 표시하고, 검색/필터/페이지네이션 UI를 구성한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/events/page.tsx` — 사건 목록 페이지
    - `src/features/events/ui/EventListPage.tsx` — 목록 페이지 컴포넌트
    - `src/features/events/ui/EventTable.tsx` — TanStack Table 기반 데이터 테이블 (제목, 발생일, 중요도, 검증상태, 출처수, 태그, 생성일)
    - `src/features/events/ui/EventFilters.tsx` — 검색 Input(debounce 300ms), 중요도 Select, 검증상태 Select, 정렬 Select
  - **더미 데이터**: `{ id, title, occurredAt, importance, verificationStatus, sourceCount, tags[], createdAt }` 10건
  - **예상 소요**: 2일
  - **의존성**: `P1-006`

- [ ] `P1-009` 사건 생성/수정 폼 UI
  - **설명**: 사건 생성 및 수정을 위한 폼 UI를 구현한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/events/new/page.tsx` — 사건 생성 페이지
    - `src/app/(admin)/events/[id]/page.tsx` — 사건 수정 페이지
    - `src/features/events/ui/EventForm.tsx` — 폼 컴포넌트 (React Hook Form + Zod)
    - `src/features/events/model/schemas/eventSchema.ts` — Zod 검증 스키마
    - 필드: 제목(50자), 요약(Textarea), 발생일시(DatePicker), 중요도(Select), 검증상태(Select), 태그(다중 선택), 출처(동적 반복 입력 — URL, 제목, 매체명, 발행일)
    - 발생일시가 미래인 경우 경고 표시 (저장은 허용)
  - **더미 데이터**: 수정 모드에서 기존 사건 데이터 하드코딩
  - **예상 소요**: 2일
  - **의존성**: `P1-008`

- [ ] `P1-010` 이슈 목록 페이지 UI
  - **설명**: 이슈 목록을 데이터 테이블로 표시한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/issues/page.tsx` — 이슈 목록 페이지
    - `src/features/issues/ui/IssueListPage.tsx` — 목록 페이지 컴포넌트
    - `src/features/issues/ui/IssueTable.tsx` — 데이터 테이블 (제목, 상태, 추적자 수, 최근 트리거 일시, 태그, 생성일)
    - 필터: 상태 Select, 검색, 정렬(최근 트리거순/생성일순/추적자 수)
  - **더미 데이터**: `{ id, title, status, trackerCount, latestTriggerAt, tags[], createdAt }` 10건
  - **예상 소요**: 1.5일
  - **의존성**: `P1-006`

- [ ] `P1-011` 이슈 생성/수정 폼 UI
  - **설명**: 이슈 생성 및 수정 폼을 구현한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/issues/new/page.tsx` — 이슈 생성 페이지
    - `src/app/(admin)/issues/[id]/page.tsx` — 이슈 수정 페이지 (트리거 목록 포함)
    - `src/features/issues/ui/IssueForm.tsx` — 폼 컴포넌트
    - `src/features/issues/model/schemas/issueSchema.ts` — Zod 검증 스키마
    - 필드: 제목(50자), 설명(Textarea), 상태(Select: ongoing/closed/reignited/unverified), 태그(다중 선택), 관련 사건(검색/다중 선택)
    - 수정 페이지에서 해당 이슈의 트리거 목록도 표시
  - **더미 데이터**: 수정 모드에서 기존 이슈 + 트리거 목록 하드코딩
  - **예상 소요**: 2일
  - **의존성**: `P1-010`

- [ ] `P1-012` 트리거 목록 페이지 UI
  - **설명**: 트리거 목록을 데이터 테이블로 표시한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/triggers/page.tsx` — 트리거 목록 페이지
    - `src/features/triggers/ui/TriggerListPage.tsx` — 목록 페이지 컴포넌트
    - `src/features/triggers/ui/TriggerTable.tsx` — 데이터 테이블 (요약, 유형, 이슈 제목, 발생일, 생성일)
    - 필터: 이슈별 필터, 유형 Select, 검색, 정렬(발생일/생성일)
  - **더미 데이터**: `{ id, summary, type, occurredAt, issue: { id, title }, createdAt }` 10건
  - **예상 소요**: 1.5일
  - **의존성**: `P1-006`

- [ ] `P1-013` 트리거 생성/수정 폼 UI
  - **설명**: 트리거 생성 및 수정 폼을 구현한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/triggers/new/page.tsx` — 트리거 생성 페이지
    - `src/app/(admin)/triggers/[id]/page.tsx` — 트리거 수정 페이지
    - `src/features/triggers/ui/TriggerForm.tsx` — 폼 컴포넌트
    - `src/features/triggers/model/schemas/triggerSchema.ts` — Zod 검증 스키마
    - 필드: 이슈(필수, 이슈 검색/선택), 요약(Textarea), 유형(Select: article/ruling/announcement/correction/status_change), 발생일시(DatePicker), 출처(동적 반복 입력)
  - **더미 데이터**: 수정 모드에서 기존 트리거 데이터 하드코딩
  - **예상 소요**: 1.5일
  - **의존성**: `P1-012`

- [ ] `P1-014` 태그 관리 페이지 UI
  - **설명**: 태그 목록, 생성, 수정, 삭제 UI를 구현한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/tags/page.tsx` — 태그 관리 페이지
    - `src/features/tags/ui/TagListPage.tsx` — 태그 목록 (이름, 유형, slug, 사용 수)
    - `src/features/tags/ui/TagFormDialog.tsx` — 태그 생성/수정 다이얼로그
    - `src/features/tags/model/schemas/tagSchema.ts` — Zod 검증 스키마
    - 필드: 이름(50자), 유형(Select: category/region), slug(80자, unique)
  - **더미 데이터**: `{ id, name, type, slug, usageCount, updatedAt }` 10건
  - **예상 소요**: 1일
  - **의존성**: `P1-006`

- [ ] `P1-015` 출처 관리 페이지 UI
  - **설명**: 출처 목록 및 추가/삭제 UI를 구현한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/sources/page.tsx` — 출처 관리 페이지
    - `src/features/sources/ui/SourceListPage.tsx` — 출처 목록 (제목, URL, 매체명, 대상 유형, 대상 제목, 발행일)
    - `src/features/sources/ui/SourceFormDialog.tsx` — 출처 추가 다이얼로그
    - 필드: 엔티티 유형(Select: event/issue/trigger), 엔티티(검색/선택), URL(500자), 제목(255자), 매체명(100자), 발행일(DatePicker)
    - 엔티티 유형별 필터
  - **더미 데이터**: `{ id, entityType, entityId, entityTitle, url, title, publisher, publishedAt }` 10건
  - **예상 소요**: 1일
  - **의존성**: `P1-006`

- [ ] `P1-016` 삭제 확인 다이얼로그 공통 컴포넌트
  - **설명**: 모든 CRUD 삭제 시 사용할 확인 다이얼로그를 공통 컴포넌트로 구현한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/shared/ui/DeleteConfirmDialog.tsx` — shadcn/ui AlertDialog 활용, 영향 범위 메시지 표시
  - **예상 소요**: 0.5일
  - **의존성**: `P1-002`

#### 완료 기준
- 모든 CRUD 화면이 더미 데이터로 렌더링된다
- 폼 입력 시 Zod 클라이언트 검증이 동작한다
- TanStack Table 기반 정렬/필터/페이지네이션 UI가 동작한다
- 반응형 레이아웃이 데스크톱/태블릿에서 동작한다
- `pnpm lint`, `pnpm type:check` 통과

---

### Phase 1-3: CRUD 기능 연동

> Phase 1-2에서 구성한 UI를 Route Handler + Prisma를 통해 실제 DB와 연동한다.
> **TDD 필수**: 모든 태스크는 Red-Green-Refactor 사이클을 따른다.

#### 태스크

- [ ] `P1-017` 사건 CRUD Route Handlers
  - **설명**: 사건 목록/상세/생성/수정/삭제 Route Handler를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/app/api/events/route.ts` — `GET` (목록: 필터/정렬/페이지네이션), `POST` (생성: events + event_tags + sources 트랜잭션)
    - `src/app/api/events/[id]/route.ts` — `GET` (상세: 태그, 출처, 관련 이슈 포함), `PATCH` (수정: 트랜잭션), `DELETE` (삭제: 연관 데이터 cascade)
    - Zod 요청 검증 스키마 적용
    - 생성/삭제 시 `events.source_count` 재계산
  - **테스트 (TDD)**:
    - 🔴 `POST /api/events` — 유효한 데이터로 사건 생성 시 201 반환 + DB에 레코드 존재 테스트
    - 🔴 `POST /api/events` — 필수 필드 누락 시 400 반환 테스트
    - 🔴 `DELETE /api/events/[id]` — 삭제 후 연관 event_tags, sources도 삭제되는지 테스트
    - 🟢 Prisma 트랜잭션 기반 CRUD 구현
    - 🔵 쿼리 최적화 (include/select 최소화), 에러 핸들링 표준화
  - **예상 소요**: 2일
  - **의존성**: `P1-007`, `P1-009`

- [ ] `P1-018` 이슈 CRUD Route Handlers
  - **설명**: 이슈 목록/상세/생성/수정/삭제 Route Handler를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/app/api/issues/route.ts` — `GET` (목록), `POST` (생성: issues + issue_tags + issue_events 트랜잭션)
    - `src/app/api/issues/[id]/route.ts` — `GET` (상세: 트리거, 관련 사건 포함), `PATCH` (수정), `DELETE` (삭제: triggers, issue_tags, issue_events, user_tracked_issues cascade)
    - 삭제 전 트리거 수가 0이 아니면 경고 응답 (확인 파라미터 필요)
  - **테스트 (TDD)**:
    - 🔴 `POST /api/issues` — 관련 사건 연결이 issue_events 테이블에 생성되는지 테스트
    - 🔴 `DELETE /api/issues/[id]` — 트리거가 존재할 때 confirm 파라미터 없이 호출 시 경고 응답 테스트
    - 🟢 Prisma 트랜잭션 기반 CRUD 구현
    - 🔵 이슈-사건 연결 로직 정리, 상태 전이 검증
  - **예상 소요**: 2일
  - **의존성**: `P1-007`, `P1-011`

- [ ] `P1-019` 트리거 CRUD Route Handlers
  - **설명**: 트리거 목록/생성/수정/삭제 Route Handler를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/app/api/triggers/route.ts` — `GET` (목록: issue_id 필터), `POST` (생성: triggers + sources + issues.latest_trigger_at 갱신 트랜잭션)
    - `src/app/api/triggers/[id]/route.ts` — `PATCH` (수정), `DELETE` (삭제: issues.latest_trigger_at 재계산)
  - **테스트 (TDD)**:
    - 🔴 `POST /api/triggers` — 생성 후 해당 이슈의 `latest_trigger_at`이 갱신되는지 테스트
    - 🔴 `DELETE /api/triggers/[id]` — 삭제 후 `latest_trigger_at`이 남은 트리거 중 최신값으로 재계산되는지 테스트
    - 🟢 Prisma 트랜잭션 + latest_trigger_at 갱신 로직 구현
    - 🔵 부수 효과 로직을 서비스 레이어로 분리 검토
  - **예상 소요**: 2일
  - **의존성**: `P1-007`, `P1-013`

- [ ] `P1-020` 태그 CRUD Route Handlers
  - **설명**: 태그 목록/생성/수정/삭제 Route Handler를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/app/api/tags/route.ts` — `GET` (목록: usageCount 집계 포함), `POST` (생성)
    - `src/app/api/tags/[id]/route.ts` — `PATCH` (수정), `DELETE` (삭제: event_tags, issue_tags, post_tags cascade)
    - slug 중복 시 409 Conflict 응답
  - **테스트 (TDD)**:
    - 🔴 `POST /api/tags` — slug 중복 시 409 반환 테스트
    - 🔴 `GET /api/tags` — usageCount가 event_tags + issue_tags + post_tags 합산값인지 테스트
    - 🟢 Prisma CRUD + 집계 쿼리 구현
    - 🔵 태그 캐시 전략 검토
  - **예상 소요**: 1.5일
  - **의존성**: `P1-007`, `P1-014`

- [ ] `P1-021` 출처 CRUD Route Handlers
  - **설명**: 출처 목록/생성/삭제 Route Handler를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/app/api/sources/route.ts` — `GET` (목록: entityType/entityId 필터, JOIN으로 대상 엔티티 제목 포함), `POST` (생성: events.source_count 갱신)
    - `src/app/api/sources/[id]/route.ts` — `DELETE` (삭제: events.source_count 갱신)
  - **테스트 (TDD)**:
    - 🔴 `POST /api/sources` — entity_type이 event일 때 해당 events.source_count가 증가하는지 테스트
    - 🔴 `DELETE /api/sources/[id]` — 삭제 후 events.source_count가 감소하는지 테스트
    - 🟢 Prisma CRUD + source_count 갱신 트랜잭션 구현
    - 🔵 entity_type별 JOIN 로직 최적화
  - **예상 소요**: 1.5일
  - **의존성**: `P1-007`, `P1-015`

- [ ] `P1-022` UI-API 연동 (사건/이슈/트리거)
  - **설명**: 사건/이슈/트리거 목록 및 폼 UI를 실제 Route Handler와 연동한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/features/events/model/api/` — 사건 CRUD React Query 훅 (`useEvents`, `useEvent`, `useCreateEvent`, `useUpdateEvent`, `useDeleteEvent`)
    - `src/features/issues/model/api/` — 이슈 CRUD React Query 훅
    - `src/features/triggers/model/api/` — 트리거 CRUD React Query 훅
    - 각 목록 페이지: 더미 데이터를 `useQuery` 호출로 교체
    - 각 폼: 제출을 `useMutation` 호출로 교체
    - 삭제: `DeleteConfirmDialog` + `useMutation` 연동
  - **테스트 (TDD)**:
    - 🔴 사건 생성 폼 제출 시 `POST /api/events` 호출 후 목록이 갱신되는지 테스트
    - 🔴 이슈 삭제 시 확인 다이얼로그 후 API 호출 및 목록에서 제거되는지 테스트
    - 🟢 React Query 훅 + UI 연동 구현
    - 🔵 캐시 무효화 전략 최적화 (`invalidateQueries`)
  - **예상 소요**: 2일
  - **의존성**: `P1-017`, `P1-018`, `P1-019`

- [ ] `P1-023` UI-API 연동 (태그/출처)
  - **설명**: 태그/출처 목록 및 폼 UI를 실제 Route Handler와 연동한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/features/tags/model/api/` — 태그 CRUD React Query 훅
    - `src/features/sources/model/api/` — 출처 CRUD React Query 훅
    - 태그: 다이얼로그 기반 생성/수정 + 삭제 시 영향 범위 표시
    - 출처: 다이얼로그 기반 추가 + 삭제
  - **테스트 (TDD)**:
    - 🔴 태그 생성 시 slug 중복 에러가 폼에 표시되는지 테스트
    - 🟢 React Query 훅 + UI 연동 구현
    - 🔵 태그 캐시 공유 (사건/이슈 폼의 태그 선택에서도 참조)
  - **예상 소요**: 1.5일
  - **의존성**: `P1-020`, `P1-021`

- [ ] `P1-024` 토스트 알림 및 에러 핸들링
  - **설명**: CRUD 작업 성공/실패 시 사용자 피드백을 제공한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - Sonner(Toast) 기반 성공/에러 알림 통합
    - 성공: "사건이 생성되었습니다", "태그가 삭제되었습니다" 등
    - 실패: API 에러 메시지 표시
    - 401: 로그인 페이지 리다이렉트
    - 403: "권한이 부족합니다" 메시지
    - 409: 충돌 에러 (slug 중복 등) 구체적 메시지
  - **테스트 (TDD)**:
    - 🔴 API 401 응답 시 로그인 페이지로 리다이렉트되는지 테스트
    - 🔴 CRUD 성공 시 적절한 토스트 메시지가 표시되는지 테스트
    - 🟢 에러 핸들러 + 토스트 통합 구현
    - 🔵 에러 타입별 메시지 매핑 정리
  - **예상 소요**: 1일
  - **의존성**: `P1-022`

#### 완료 기준
- 사건/이슈/트리거/태그/출처 CRUD가 실제 DB와 연동된다
- 모든 CRUD 작업에 성공/실패 피드백(토스트)이 제공된다
- Prisma 트랜잭션으로 데이터 정합성이 보장된다 (source_count, latest_trigger_at 등)
- Route Handler 테스트 커버리지 80%+
- `pnpm lint`, `pnpm type:check` 통과

---

## Phase 2: 운영 도구

**목표**: 대시보드, 사용자 관리, 파이프라인 모니터링을 구현하여 서비스 운영 효율을 높인다
**예상 기간**: 2주
**선행 조건**: Phase 1
**TDD**: 필수 (Red-Green-Refactor)

### Phase 2-1: 대시보드 (FR-AD1)

#### 태스크

- [ ] `P2-001` 대시보드 통계 Route Handler
  - **설명**: 어드민 대시보드에 필요한 통계 집계 Route Handler를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/app/api/dashboard/stats/route.ts` — `GET` 오늘 등록 사건/트리거 수, 활성 이슈 수, 오늘 신규 가입자 수, 활성 사용자 수, 미처리 신고 수 집계
    - `src/app/api/dashboard/recent-jobs/route.ts` — `GET` 최근 잡 실행 이력 10건 (job_runs 테이블)
    - Prisma 집계 쿼리 (`_count`, `where`, `gte`)
  - **테스트 (TDD)**:
    - 🔴 `GET /api/dashboard/stats` — 오늘 생성된 사건만 카운트되는지 테스트
    - 🔴 `GET /api/dashboard/recent-jobs` — 최신순 10건만 반환되는지 테스트
    - 🟢 Prisma 집계 쿼리 구현
    - 🔵 쿼리 성능 최적화 (인덱스 활용)
  - **예상 소요**: 1.5일
  - **의존성**: `P1-024`

- [ ] `P2-002` 대시보드 UI 구현
  - **설명**: 대시보드 페이지를 구현한다 (통계 카드 + 최근 잡 이력 테이블)
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/page.tsx` — 대시보드 페이지 (리다이렉트 대신 실제 대시보드)
    - `src/features/dashboard/ui/DashboardPage.tsx` — 대시보드 페이지 컴포넌트
    - `src/features/dashboard/ui/StatsCards.tsx` — 통계 카드 (오늘 사건, 트리거, 활성 이슈, 신규 가입자, 미처리 신고)
    - `src/features/dashboard/ui/RecentJobsTable.tsx` — 최근 잡 실행 테이블 (잡 이름, 상태 뱃지, 상세, 시작/종료 시각)
    - `src/features/dashboard/model/api/` — 대시보드 React Query 훅
    - 자동 갱신: 60초 polling (`refetchInterval`)
  - **테스트 (TDD)**:
    - 🔴 통계 API 응답이 올바르게 카드에 렌더링되는지 테스트
    - 🟢 API 훅 + UI 컴포넌트 구현
    - 🔵 로딩/에러 상태 스켈레톤 UI 추가
  - **예상 소요**: 2일
  - **의존성**: `P2-001`

### Phase 2-2: 사용자 관리 (FR-AD4)

#### 태스크

- [ ] `P2-003` 사용자 관리 Route Handlers
  - **설명**: 사용자 목록 조회, 역할 변경, 정지/복원 Route Handler를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/app/api/users/route.ts` — `GET` 사용자 목록 (닉네임/이메일 검색, 역할 필터, 활성 상태 필터, 페이지네이션)
    - `src/app/api/users/[id]/route.ts` — `GET` 사용자 상세 (프로필, 게시글/댓글 수)
    - `src/app/api/users/[id]/role/route.ts` — `PATCH` 역할 변경 (자기 자신 변경 불가, guest->admin 직접 승격 불가)
    - `src/app/api/users/[id]/status/route.ts` — `PATCH` 활성/정지 토글 (자기 자신 정지 불가)
  - **테스트 (TDD)**:
    - 🔴 자기 자신의 역할 변경 시도 시 403 반환 테스트
    - 🔴 guest -> admin 직접 승격 시 400 반환 테스트
    - 🔴 자기 자신 정지 시도 시 403 반환 테스트
    - 🟢 Prisma CRUD + 비즈니스 규칙 검증 구현
    - 🔵 감사 로그 추가 검토 (향후)
  - **예상 소요**: 2일
  - **의존성**: `P1-024`

- [ ] `P2-004` 사용자 관리 UI 구현
  - **설명**: 사용자 목록, 상세, 역할 변경, 정지/복원 UI를 구현한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/users/page.tsx` — 사용자 목록 페이지
    - `src/app/(admin)/users/[id]/page.tsx` — 사용자 상세 페이지
    - `src/features/users/ui/UserListPage.tsx` — 목록 (닉네임, 이메일, 역할 뱃지, 활성 상태 뱃지, 가입일, 게시글/댓글 수)
    - `src/features/users/ui/UserDetailPage.tsx` — 상세 (프로필, 역할 변경 버튼, 정지/복원 버튼)
    - `src/features/users/ui/RoleChangeDialog.tsx` — 역할 변경 확인 다이얼로그
    - `src/features/users/model/api/` — 사용자 관리 React Query 훅
    - 사이드바에 "사용자" 메뉴 활성화
  - **테스트 (TDD)**:
    - 🔴 역할 변경 다이얼로그에서 확인 클릭 시 API 호출 + 목록 갱신 테스트
    - 🔴 정지된 사용자가 목록에서 비활성 뱃지로 표시되는지 테스트
    - 🟢 UI 컴포넌트 + API 훅 연동
    - 🔵 사용자 검색/필터 UX 개선
  - **예상 소요**: 2일
  - **의존성**: `P2-003`

### Phase 2-3: 파이프라인 모니터링 (FR-AD6)

#### 태스크

- [ ] `P2-005` 파이프라인 모니터링 Route Handlers
  - **설명**: 잡 실행 이력, 키워드 상태 조회 Route Handler를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/app/api/pipeline/jobs/route.ts` — `GET` 잡 실행 이력 (잡 이름/상태 필터, 페이지네이션, 소요시간 계산)
    - `src/app/api/pipeline/keywords/route.ts` — `GET` 키워드 상태 목록 (issue_keyword_states + issues JOIN, 상태 필터)
    - `src/app/api/pipeline/trigger-collect/route.ts` — `POST` 수동 뉴스 수집 트리거 [PRD 확인 필요: 방식 A vs 방식 B]
  - **테스트 (TDD)**:
    - 🔴 `GET /api/pipeline/jobs?status=failed` — 실패한 잡만 필터되는지 테스트
    - 🔴 `GET /api/pipeline/keywords` — 이슈 제목이 JOIN으로 포함되는지 테스트
    - 🟢 Prisma 쿼리 구현
    - 🔵 잡 상세 JSON 파싱 로직 추가
  - **예상 소요**: 2일
  - **의존성**: `P1-024`

- [ ] `P2-006` 파이프라인 모니터링 UI 구현
  - **설명**: 파이프라인 모니터링 페이지를 구현한다 (잡 이력 + 키워드 상태 + 수동 수집 트리거)
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/pipeline/page.tsx` — 파이프라인 모니터링 페이지
    - `src/features/pipeline/ui/PipelinePage.tsx` — 페이지 컴포넌트 (두 섹션)
    - `src/features/pipeline/ui/JobHistoryTable.tsx` — 잡 이력 테이블 (잡 이름, 상태 뱃지, 상세, 시작/종료 시각, 소요시간)
    - `src/features/pipeline/ui/KeywordStateTable.tsx` — 키워드 상태 테이블 (키워드, 이슈 제목, 상태 뱃지, 최근 확인 시각)
    - `src/features/pipeline/ui/ManualTriggerButton.tsx` — "뉴스 수집 실행" 버튼 + 확인 다이얼로그
    - `src/features/pipeline/model/api/` — 파이프라인 React Query 훅
    - 사이드바에 "파이프라인" 메뉴 활성화
  - **테스트 (TDD)**:
    - 🔴 수동 수집 트리거 버튼 클릭 시 확인 다이얼로그 표시 후 API 호출 테스트
    - 🔴 잡 상태별 뱃지 색상이 올바르게 렌더링되는지 테스트
    - 🟢 UI 컴포넌트 + API 연동 구현
    - 🔵 자동 갱신 (30초 polling) 적용
  - **예상 소요**: 2일
  - **의존성**: `P2-005`

#### 완료 기준
- 대시보드에서 주요 지표(오늘 사건/트리거/이슈/사용자/신고)를 확인할 수 있다
- 사용자 역할 변경 및 정지/복원이 동작한다 (비즈니스 규칙 준수)
- 파이프라인 잡 실행 이력과 키워드 상태를 확인할 수 있다
- 모든 Route Handler 테스트 통과
- `pnpm lint`, `pnpm type:check` 통과

---

## Phase 3: 모더레이션

**목표**: 커뮤니티 모더레이션 기능과 신고 시스템을 구현하여 콘텐츠 품질을 관리할 수 있게 한다
**예상 기간**: 1.5주
**선행 조건**: Phase 2 (대시보드 신고 위젯 연동 위해)
**TDD**: 필수 (Red-Green-Refactor)

> reports 테이블은 Alembic 마이그레이션(trend-korea-api 측)으로 생성한 뒤 `prisma db pull`로 반영한다.

### Phase 3-1: 신고 시스템 기반

#### 태스크

- [ ] `P3-001` reports 테이블 Alembic 마이그레이션 (trend-korea-api 측)
  - **설명**: 신고 시스템을 위한 reports 테이블을 trend-korea-api 측 Alembic 마이그레이션으로 생성한다
  - **영역**: 백엔드 (trend-korea-api)
  - **구현 사항**:
    - `Report` 모델: id(UUID), reporter_id(FK users), target_type(enum: post/comment), target_id, reason(enum: spam/abuse/misinformation/inappropriate/other), detail(nullable), status(enum: pending/reviewed/dismissed), reviewed_at(nullable), reviewed_by(FK users, nullable), created_at, updated_at
    - 인덱스: (target_type, target_id), (status), (reporter_id, target_type, target_id) UNIQUE
    - Alembic 마이그레이션 파일 생성 및 적용
  - **테스트 (TDD)**:
    - 🔴 마이그레이션 적용 후 reports 테이블이 존재하는지 테스트
    - 🟢 Alembic 마이그레이션 작성 및 적용
    - 🔵 인덱스 최적화 검토
  - **예상 소요**: 1일
  - **의존성**: 없음

- [ ] `P3-002` Prisma 스키마 갱신 (reports 테이블 반영)
  - **설명**: reports 테이블 생성 후 `prisma db pull`로 Prisma 스키마를 갱신한다
  - **영역**: 인프라
  - **구현 사항**:
    - `prisma db pull` 실행
    - `prisma generate` 실행
    - Prisma Client에 Report 모델이 포함되는지 확인
  - **예상 소요**: 0.5일
  - **의존성**: `P3-001`

### Phase 3-2: 신고/게시글/댓글 관리 Route Handlers

#### 태스크

- [ ] `P3-003` 신고 관리 Route Handlers
  - **설명**: 신고 목록 조회 및 처리(승인/기각) Route Handler를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/app/api/reports/route.ts` — `GET` 신고 목록 (status/targetType 필터, 대상 콘텐츠 미리보기, 신고자 닉네임 JOIN, 페이지네이션)
    - `src/app/api/reports/[id]/route.ts` — `PATCH` 신고 처리: reviewed(status 변경 + reviewed_at/reviewed_by 기록) 또는 dismissed
    - `deleteTarget=true`일 때: 대상 콘텐츠 삭제 + 해당 대상의 다른 pending 신고도 reviewed로 일괄 변경
    - 대상이 이미 삭제된 경우: "이미 삭제된 콘텐츠" 메시지 + 신고 자동 reviewed 처리
  - **테스트 (TDD)**:
    - 🔴 `PATCH /api/reports/[id]` — deleteTarget=true일 때 대상 게시글과 관련 댓글이 삭제되는지 테스트
    - 🔴 `PATCH /api/reports/[id]` — 동일 대상의 다른 pending 신고도 reviewed로 변경되는지 테스트
    - 🔴 이미 삭제된 대상에 대한 신고 처리 시 자동 reviewed 처리 테스트
    - 🟢 Prisma 트랜잭션 기반 신고 처리 구현
    - 🔵 신고 처리 로직을 서비스 레이어로 분리
  - **예상 소요**: 2일
  - **의존성**: `P3-002`

- [ ] `P3-004` 게시글/댓글 관리 Route Handlers
  - **설명**: 어드민용 게시글/댓글 목록 조회 및 삭제 Route Handler를 구현한다
  - **영역**: 백엔드
  - **구현 사항**:
    - `src/app/api/posts/route.ts` — `GET` 게시글 목록 (신고 건수 집계 포함, 페이지네이션)
    - `src/app/api/posts/[id]/route.ts` — `DELETE` 게시글 삭제 (트랜잭션: 댓글, comment_likes, post_votes, post_tags, reports cascade + posts.comment_count 무관)
    - `src/app/api/comments/[id]/route.ts` — `DELETE` 댓글 삭제 (트랜잭션: 하위 대댓글, comment_likes, reports cascade + posts.comment_count 재계산)
  - **테스트 (TDD)**:
    - 🔴 `DELETE /api/posts/[id]` — 게시글 삭제 시 관련 댓글/투표/태그/신고가 모두 삭제되는지 테스트
    - 🔴 `DELETE /api/comments/[id]` — 댓글 삭제 시 하위 대댓글도 삭제되고 posts.comment_count가 재계산되는지 테스트
    - 🟢 Prisma 트랜잭션 기반 cascade 삭제 구현
    - 🔵 삭제 성능 최적화 (대량 연관 데이터 시)
  - **예상 소요**: 2일
  - **의존성**: `P3-002`

### Phase 3-3: 모더레이션 UI

#### 태스크

- [ ] `P3-005` 모더레이션 페이지 UI 구현
  - **설명**: 모더레이션 페이지를 세 개 탭(신고 대기, 게시글, 댓글)으로 구현한다
  - **영역**: 프론트엔드
  - **구현 사항**:
    - `src/app/(admin)/moderation/page.tsx` — 모더레이션 페이지
    - `src/features/moderation/ui/ModerationPage.tsx` — 탭 기반 페이지 컴포넌트
    - `src/features/moderation/ui/PendingReportsTab.tsx` — 신고 대기 탭 (신고 사유, 대상 미리보기, 신고자, 신고일, "삭제"/"기각" 버튼)
    - `src/features/moderation/ui/PostsTab.tsx` — 게시글 탭 (제목, 작성자, 추천수, 댓글수, 신고수, 작성일, "삭제" 버튼)
    - `src/features/moderation/ui/CommentsTab.tsx` — 댓글 탭 (내용 100자, 작성자, 게시글 제목, 좋아요수, 신고수, 작성일, "삭제" 버튼)
    - `src/features/moderation/ui/ReportActionDialog.tsx` — 신고 처리 확인 다이얼로그 ("이 게시글을 삭제하시겠습니까? 관련 댓글 N개도 함께 삭제됩니다.")
    - `src/features/moderation/model/api/` — 모더레이션 React Query 훅
    - 사이드바에 "모더레이션" 메뉴 활성화 (미처리 신고 건수 뱃지)
  - **테스트 (TDD)**:
    - 🔴 신고 "삭제" 버튼 클릭 시 확인 다이얼로그에 관련 댓글 수가 표시되는지 테스트
    - 🔴 신고 처리 후 목록에서 해당 신고가 사라지는지 테스트
    - 🔴 사이드바 뱃지에 미처리 신고 건수가 표시되는지 테스트
    - 🟢 탭 UI + API 훅 + 삭제/기각 로직 구현
    - 🔵 신고 상태 필터링 UX 개선
  - **예상 소요**: 2.5일
  - **의존성**: `P3-003`, `P3-004`

- [ ] `P3-006` 대시보드 신고 위젯 연동 (S-AD1-4)
  - **설명**: 대시보드에 미처리 신고 목록 위젯을 추가한다
  - **영역**: 프론트엔드 + 백엔드
  - **구현 사항**:
    - `src/app/api/dashboard/reports/route.ts` — `GET` 미처리 신고 10건 (target_type, targetTitle, reason, reporterNickname, createdAt)
    - `src/features/dashboard/ui/RecentReportsTable.tsx` — 미처리 신고 위젯 (클릭 시 `/moderation`으로 이동)
    - 대시보드 페이지에 위젯 추가
  - **테스트 (TDD)**:
    - 🔴 대시보드 신고 위젯이 미처리 신고만 표시하는지 테스트
    - 🔴 위젯 클릭 시 `/moderation`으로 이동하는지 테스트
    - 🟢 Route Handler + UI 위젯 구현
    - 🔵 대시보드 로딩 최적화 (병렬 API 호출)
  - **예상 소요**: 1일
  - **의존성**: `P3-005`, `P2-002`

#### 완료 기준
- reports 테이블이 생성되고 Prisma 스키마에 반영된다
- 신고된 콘텐츠를 검토하고 삭제/기각 처리할 수 있다
- 게시글/댓글을 어드민 권한으로 삭제할 수 있다 (연관 데이터 cascade)
- 대시보드에 미처리 신고 위젯이 표시된다
- 사이드바에 미처리 신고 건수 뱃지가 표시된다
- 모든 Route Handler 테스트 통과
- `pnpm lint`, `pnpm type:check` 통과

---

## 의존성 그래프

```
Phase 1-1 (프로젝트 골격)
  ├── P1-001 (Next.js 초기화)
  │     ├── P1-002 (shadcn/ui) ────────────────── P1-016 (삭제 다이얼로그)
  │     ├── P1-003 (Prisma)
  │     └── P1-004 (JWT 미들웨어)
  │           ├── P1-005 (로그인)
  │           ├── P1-006 (레이아웃) ─── [P1-002 필요]
  │           └── P1-007 (Route Handler 헬퍼)
  │
  └── Phase 1-2 (UI/UX 레이아웃) ─── P1-006 선행
        ├── P1-008 (사건 목록) → P1-009 (사건 폼)
        ├── P1-010 (이슈 목록) → P1-011 (이슈 폼)
        ├── P1-012 (트리거 목록) → P1-013 (트리거 폼)
        ├── P1-014 (태그 UI)
        └── P1-015 (출처 UI)
              │
              └── Phase 1-3 (기능 연동) ─── P1-007 선행
                    ├── P1-017 (사건 API) ─┐
                    ├── P1-018 (이슈 API) ─┤→ P1-022 (사건/이슈/트리거 연동)
                    ├── P1-019 (트리거 API)┘
                    ├── P1-020 (태그 API) ─┐→ P1-023 (태그/출처 연동)
                    ├── P1-021 (출처 API) ─┘
                    └── P1-024 (토스트/에러) ─── MVP 완성

Phase 2 (운영 도구) ─── Phase 1 완료 후
  ├── P2-1: 대시보드 (P2-001 → P2-002)
  ├── P2-2: 사용자 관리 (P2-003 → P2-004)
  └── P2-3: 파이프라인 (P2-005 → P2-006)

Phase 3 (모더레이션) ─── Phase 2 완료 후
  ├── P3-001 (reports 마이그레이션) → P3-002 (Prisma 갱신)
  │     ├── P3-003 (신고 API)
  │     └── P3-004 (게시글/댓글 API)
  │           └── P3-005 (모더레이션 UI)
  └── P3-006 (대시보드 신고 위젯) ─── P3-005 + P2-002 선행
```

---

## 리스크 및 고려사항

| 리스크 | 영향도 | 대응 방안 |
|--------|--------|----------|
| Prisma와 SQLAlchemy가 동일 DB를 공유하여 스키마 충돌 가능 | 높음 | Prisma는 `db pull` (읽기 전용)만 사용, 스키마 변경은 Alembic(trend-korea-api 측)에서만 수행 |
| 수동 뉴스 수집 트리거 구현 방식 미결정 | 중간 | 구현 시점에 trend-korea-api 측과 협의 (방식 A: API 추가 vs 방식 B: DB 플래그) |
| reports 테이블이 아직 없음 | 중간 | Phase 3 시작 전 Alembic 마이그레이션으로 생성 (Phase 3-1의 첫 태스크) |
| 어드민 로그인 UX 미결정 (API 직접 호출 vs 공유 쿠키) | 낮음 | MVP에서는 API 직접 호출 방식으로 구현, 추후 공유 쿠키 방식 검토 |
| 배포 환경 미결정 (Vercel vs 자체 서버) | 중간 | Phase 1 완료 후 배포 환경 결정, Prisma + DB 직접 접근이므로 자체 서버/Cloud Run 권장 |
| 비정규화 필드 정합성 (source_count, comment_count, tracker_count) | 높음 | 모든 관련 CUD 작업에서 Prisma 트랜잭션 내 재계산 필수 |
| TanStack Table 초기 설정 복잡도 | 낮음 | Phase 1-2에서 사건 테이블을 기준으로 패턴을 확립하고 이후 테이블에 재사용 |
| trend-korea-api 스케줄러 잡 상세(detail) JSON 구조 미명세 | 낮음 | 파이프라인 모니터링(Phase 2-3) 구현 시 실제 데이터 기반으로 파싱 로직 작성 |

---

## PRD 추적성 매트릭스

| PRD 요구사항 | Spec ID | 태스크 ID | Phase | 상태 |
|-------------|---------|----------|-------|------|
| 프로젝트 초기화 (Next.js + Prisma + JWT) | — | `P1-001` ~ `P1-007` | 1-1 | ⬜ |
| 공통 레이아웃 (사이드바 + 헤더) | — | `P1-006` | 1-1 | ⬜ |
| 사건 CRUD | S-AD2-1 ~ S-AD2-3 | `P1-008`, `P1-009`, `P1-017`, `P1-022` | 1-2, 1-3 | ⬜ |
| 이슈 CRUD | S-AD2-4 ~ S-AD2-6 | `P1-010`, `P1-011`, `P1-018`, `P1-022` | 1-2, 1-3 | ⬜ |
| 트리거 CRUD | S-AD2-7 ~ S-AD2-9 | `P1-012`, `P1-013`, `P1-019`, `P1-022` | 1-2, 1-3 | ⬜ |
| 태그 CRUD | S-AD3-1 ~ S-AD3-3 | `P1-014`, `P1-020`, `P1-023` | 1-2, 1-3 | ⬜ |
| 출처 관리 | S-AD3-4 ~ S-AD3-5 | `P1-015`, `P1-021`, `P1-023` | 1-2, 1-3 | ⬜ |
| 대시보드 — 통계/잡 이력 | S-AD1-1 ~ S-AD1-3 | `P2-001`, `P2-002` | 2-1 | ⬜ |
| 대시보드 — 신고 위젯 | S-AD1-4 | `P3-006` | 3-3 | ⬜ |
| 사용자 관리 | S-AD4-1 ~ S-AD4-3 | `P2-003`, `P2-004` | 2-2 | ⬜ |
| 커뮤니티 모더레이션 — 게시글/댓글 삭제 | S-AD5-1 ~ S-AD5-2 | `P3-004`, `P3-005` | 3-2, 3-3 | ⬜ |
| 커뮤니티 모더레이션 — 신고 검토 | S-AD5-3 | `P3-003`, `P3-005` | 3-2, 3-3 | ⬜ |
| 파이프라인 모니터링 — 잡 이력 | S-AD6-1 ~ S-AD6-2 | `P2-005`, `P2-006` | 2-3 | ⬜ |
| 파이프라인 모니터링 — 키워드 상태 | S-AD6-3 | `P2-005`, `P2-006` | 2-3 | ⬜ |
| 파이프라인 모니터링 — 수동 수집 트리거 | S-AD6-4 | `P2-005`, `P2-006` | 2-3 | ⬜ |
| 로그인 | 6.4 | `P1-005` | 1-1 | ⬜ |
| JWT 인증/인가 | 6.1 ~ 6.3 | `P1-004` | 1-1 | ⬜ |
| reports 테이블 | 5.4 | `P3-001`, `P3-002` | 3-1 | ⬜ |
