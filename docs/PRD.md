# 트렌드 코리아 어드민 PRD

> 상태: 확정 | 작성일: 2026-03-09 | 최종 검증: 2026-03-09

---

## 1. 제품 정의 (WHY)

### 1.1 한 줄 정의

트렌드 코리아 서비스의 데이터 품질 관리, 커뮤니티 모더레이션, 파이프라인 모니터링을 위한 독립 어드민 풀스택 애플리케이션

### 1.2 해결하는 문제

- 사건/이슈/트리거 데이터를 관리하려면 DB에 직접 접근해야 한다
- 커뮤니티 신고 콘텐츠를 검토할 UI가 없다
- 뉴스 수집 파이프라인의 성공/실패를 코드 로그로만 확인할 수 있다
- 사용자 역할 변경, 정지 처리를 수동 SQL로 해야 한다

### 1.3 핵심 가치

| 가치 | 설명 | 측정 지표 |
|------|------|----------|
| 운영 효율 | 모든 데이터 관리를 하나의 UI에서 처리 | DB 직접 접근 횟수 → 0 |
| 데이터 품질 | 사건/이슈의 생성/수정/삭제를 즉시 반영 | 데이터 오류 수정 소요 시간 |
| 안정성 | 파이프라인 상태를 실시간 확인하고 수동 개입 가능 | 잡 실패 인지 시간 |

### 1.4 성공 지표

| 지표 | 목표 | 측정 방법 |
|------|------|----------|
| DB 직접 접근 빈도 | 0회/주 | 운영 로그 |
| 파이프라인 실패 인지 시간 | < 5분 | 대시보드 확인 주기 |
| 데이터 오류 수정 시간 | < 2분 | 어드민 UI 작업 시간 |

---

## 2. 페르소나

### 2.1 운영자 — 1인 개발자 겸 운영

- **맥락**: 사건/이슈 데이터 품질을 관리하고, 커뮤니티 모더레이션을 수행하며, 데이터 파이프라인을 모니터링한다
- **니즈**: 최소한의 노력으로 서비스 전체를 관리할 수 있는 어드민 도구
- **서비스 숙련도**: 고급
- **핵심 동기**: "자동화된 파이프라인으로 운영 부담을 최소화하고 싶다"

---

## 3. 메인 시나리오

### 시나리오 A: 일상 운영 점검

1. 운영자가 어드민 대시보드에 접속하여 오늘 등록된 사건 수, 활성 이슈 수, 가입 사용자 수를 확인한다
2. 최근 잡 실행 이력에서 실패한 잡이 없는지 확인한다
3. 신고된 콘텐츠가 있으면 검토하여 삭제 또는 유지를 결정한다

### 시나리오 B: 데이터 관리

1. 운영자가 사건 관리 페이지에서 새 사건을 생성한다 (제목, 요약, 중요도, 태그, 출처 입력)
2. 기존 이슈에 새 트리거를 추가한다
3. 잘못된 태그를 수정하고 불필요한 출처를 삭제한다

### 시나리오 C: 파이프라인 모니터링

1. 운영자가 파이프라인 모니터링 페이지에서 최근 뉴스 수집 결과를 확인한다
2. 실패한 잡의 에러 로그를 확인하고 원인을 파악한다
3. 필요 시 수동으로 뉴스 수집 사이클을 트리거한다

---

## 4. 아키텍처

### 4.1 시스템 위치

```
┌─────────────────────────────────────────────────────────────┐
│                    사용자 (브라우저)                          │
└────────────────┬────────────────────────┬───────────────────┘
                 │                        │
┌────────────────▼──────────┐  ┌─────────▼───────────────────┐
│  trend-korea-app          │  │  trend-korea-admin           │
│  Next.js 16 (사용자 FE)   │  │  Next.js Fullstack (어드민)  │
│  → trend-korea-api 호출   │  │  → Prisma로 DB 직접 접근     │
└────────────────┬──────────┘  │  → Route Handlers (자체 API) │
                 │              │  → JWT 검증 (발급 안 함)     │
┌────────────────▼──────────┐  └─────────┬───────────────────┘
│  trend-korea-api          │            │
│  FastAPI (백엔드)         │            │
│  → JWT 발급               │            │
│  → SQLAlchemy 2.0         │            │
└────────────────┬──────────┘            │
                 │                        │
          ┌──────▼────────────────────────▼──┐
          │        PostgreSQL 16 (공유 DB)    │
          └──────────────────────────────────┘
```

### 4.2 핵심 아키텍처 결정

| 항목 | 결정 | 근거 |
|------|------|------|
| 독립 프로젝트 | trend-korea-app에서 분리 | 어드민은 데스크톱 전용, 배포/빌드 독립 |
| 자체 API (Route Handlers) | trend-korea-api에 admin API 없음 | 어드민 전용 로직을 독립적으로 관리 |
| Prisma ORM | `prisma db pull`로 기존 DB 스키마 introspection | SQLAlchemy와 별도로 타입 안전한 DB 접근 |
| 공유 JWT | trend-korea-api가 발급한 JWT를 검증만 함 | 인증 체계 통일, 추가 로그인 시스템 불필요 |
| shadcn/ui | 어드민용 컴포넌트 라이브러리 | 데이터 테이블, 폼, 다이얼로그 등 관리 도구에 적합 |

### 4.3 프로젝트 구조

```
trend-korea-admin/
├── docs/
│   └── PRD.md                    # 이 문서
├── prisma/
│   └── schema.prisma             # DB introspection 결과
├── src/
│   ├── app/
│   │   ├── (auth)/
│   │   │   └── login/page.tsx    # 로그인
│   │   ├── (admin)/
│   │   │   ├── layout.tsx        # 어드민 레이아웃 (사이드바 + 헤더)
│   │   │   ├── page.tsx          # 대시보드 (/)
│   │   │   ├── events/
│   │   │   │   ├── page.tsx      # 사건 목록
│   │   │   │   ├── new/page.tsx  # 사건 생성
│   │   │   │   └── [id]/page.tsx # 사건 수정
│   │   │   ├── issues/
│   │   │   │   ├── page.tsx
│   │   │   │   ├── new/page.tsx
│   │   │   │   └── [id]/page.tsx
│   │   │   ├── triggers/
│   │   │   │   ├── page.tsx
│   │   │   │   ├── new/page.tsx
│   │   │   │   └── [id]/page.tsx
│   │   │   ├── tags/page.tsx
│   │   │   ├── sources/page.tsx
│   │   │   ├── users/
│   │   │   │   ├── page.tsx
│   │   │   │   └── [id]/page.tsx
│   │   │   ├── moderation/page.tsx
│   │   │   └── pipeline/page.tsx
│   │   ├── api/                  # Route Handlers
│   │   │   ├── auth/
│   │   │   │   └── me/route.ts
│   │   │   ├── events/
│   │   │   │   ├── route.ts
│   │   │   │   └── [id]/route.ts
│   │   │   ├── issues/
│   │   │   │   ├── route.ts
│   │   │   │   └── [id]/route.ts
│   │   │   ├── triggers/
│   │   │   │   ├── route.ts
│   │   │   │   └── [id]/route.ts
│   │   │   ├── tags/
│   │   │   │   ├── route.ts
│   │   │   │   └── [id]/route.ts
│   │   │   ├── sources/
│   │   │   │   ├── route.ts
│   │   │   │   └── [id]/route.ts
│   │   │   ├── users/
│   │   │   │   ├── route.ts
│   │   │   │   └── [id]/
│   │   │   │       ├── role/route.ts
│   │   │   │       └── status/route.ts
│   │   │   ├── posts/
│   │   │   │   ├── route.ts
│   │   │   │   └── [id]/route.ts
│   │   │   ├── comments/
│   │   │   │   └── [id]/route.ts
│   │   │   ├── dashboard/
│   │   │   │   ├── stats/route.ts
│   │   │   │   ├── recent-jobs/route.ts
│   │   │   │   └── reports/route.ts
│   │   │   └── pipeline/
│   │   │       ├── jobs/route.ts
│   │   │       ├── keywords/route.ts
│   │   │       └── trigger-collect/route.ts
│   │   └── layout.tsx            # 루트 레이아웃
│   ├── features/                 # 도메인 기능
│   │   ├── auth/                 # 로그인/JWT 검증
│   │   ├── dashboard/            # 대시보드
│   │   ├── events/               # 사건 관리
│   │   ├── issues/               # 이슈 관리
│   │   ├── triggers/             # 트리거 관리
│   │   ├── tags/                 # 태그 관리
│   │   ├── sources/              # 출처 관리
│   │   ├── users/                # 사용자 관리
│   │   ├── moderation/           # 커뮤니티 모더레이션
│   │   └── pipeline/             # 파이프라인 모니터링
│   ├── shared/
│   │   ├── lib/
│   │   │   ├── prisma.ts         # Prisma Client 싱글톤
│   │   │   ├── auth.ts           # JWT 검증 유틸
│   │   │   └── api.ts            # Route Handler 헬퍼
│   │   ├── ui/                   # shadcn/ui 컴포넌트
│   │   ├── layouts/              # 사이드바, 헤더
│   │   ├── types/                # 공유 타입
│   │   └── utils/                # 유틸리티
│   └── middleware.ts             # JWT 검증 미들웨어
├── .env.local                    # 환경변수 (DB URL, JWT 시크릿)
├── package.json
├── tsconfig.json
├── next.config.ts
├── tailwind.config.ts
└── CLAUDE.md
```

---

## 5. 도메인 모델

### 5.1 핵심 엔티티 (Prisma introspection 대상)

> Prisma는 `prisma db pull` 명령으로 기존 PostgreSQL 스키마를 introspection한다.
> 아래는 어드민에서 사용하는 주요 테이블과 필드 정의다.

```
Event (사건) — events
├── id: String(36) PK
├── occurred_at: DateTime(tz)
├── title: String(50)
├── summary: Text
├── importance: Enum(low|medium|high)
├── verification_status: Enum(verified|unverified)
├── source_count: Integer
├── created_at: DateTime(tz)
└── updated_at: DateTime(tz)

Issue (이슈) — issues
├── id: String(36) PK
├── title: String(50)
├── description: Text
├── status: Enum(ongoing|closed|reignited|unverified)
├── tracker_count: Integer
├── latest_trigger_at: DateTime(tz)?
├── created_at: DateTime(tz)
└── updated_at: DateTime(tz)

Trigger (트리거) — triggers
├── id: String(36) PK
├── issue_id: FK → issues.id
├── occurred_at: DateTime(tz)
├── summary: Text
├── type: Enum(article|ruling|announcement|correction|status_change)
├── created_at: DateTime(tz)
└── updated_at: DateTime(tz)

User (사용자) — users
├── id: String(36) PK
├── nickname: String(50) UNIQUE
├── email: String(255) UNIQUE
├── password_hash: String(255)
├── profile_image: String(500)?
├── role: Enum(guest|member|admin)
├── is_active: Boolean
├── withdrawn_at: DateTime(tz)?
├── created_at: DateTime(tz)
└── updated_at: DateTime(tz)

Tag (태그) — tags
├── id: String(36) PK
├── name: String(50)
├── type: Enum(category|region)
├── slug: String(80) UNIQUE
└── updated_at: DateTime(tz)

Source (출처) — sources
├── id: String(36) PK
├── entity_type: Enum(event|issue|trigger)
├── entity_id: String(36)
├── url: String(500)
├── title: String(255)
├── publisher: String(100)
└── published_at: DateTime(tz)

Post (게시글) — posts
├── id: String(36) PK
├── author_id: FK → users.id
├── title: String(100)
├── content: Text
├── is_anonymous: Boolean
├── like_count: Integer
├── dislike_count: Integer
├── comment_count: Integer
├── created_at: DateTime(tz)
└── updated_at: DateTime(tz)

Comment (댓글) — comments
├── id: String(36) PK
├── post_id: FK → posts.id
├── parent_id: FK → comments.id?
├── author_id: FK → users.id
├── content: Text
├── like_count: Integer
├── created_at: DateTime(tz)
└── updated_at: DateTime(tz)
```

### 5.2 연관 테이블 (Many-to-Many)

| 테이블 | 관계 | 추가 컬럼 |
|--------|------|----------|
| `event_tags` | Event ↔ Tag | — |
| `issue_tags` | Issue ↔ Tag | — |
| `issue_events` | Issue ↔ Event | — |
| `post_tags` | Post ↔ Tag | — |

### 5.3 파이프라인 테이블 (조회 전용)

| 테이블 | 용도 | 어드민 사용 |
|--------|------|-----------|
| `job_runs` | 스케줄러 잡 실행 이력 | 대시보드 + 파이프라인 모니터링 |
| `issue_keyword_states` | 이슈-키워드 연결 상태 | 파이프라인 키워드 상태 조회 |
| `raw_articles` | 크롤링된 원본 뉴스 기사 | 파이프라인 수집 결과 집계 |
| `news_keyword_summaries` | OpenAI 요약 결과 | 파이프라인 요약 결과 집계 |
| `event_updates` | 사건 업데이트 분류 결과 | 파이프라인 분류 결과 집계 |

### 5.4 신규 테이블: 신고 시스템 (추가 구현 필요)

> 어드민 커뮤니티 모더레이션(FR-AD5)을 위해 추가해야 하는 테이블이다.
> Prisma 마이그레이션(`prisma migrate dev`)이 아닌 **Alembic 마이그레이션**(trend-korea-api 측)으로 생성한 뒤 `prisma db pull`로 반영한다.

```
Report (신고) — reports
├── id: String(36) PK
├── reporter_id: FK → users.id          # 신고자
├── target_type: Enum(post|comment)      # 신고 대상 유형
├── target_id: String(36)               # 신고 대상 ID
├── reason: Enum(spam|abuse|misinformation|inappropriate|other)
├── detail: Text?                        # 상세 사유 (선택)
├── status: Enum(pending|reviewed|dismissed)  # 처리 상태
├── reviewed_at: DateTime(tz)?           # 검토 완료 시각
├── reviewed_by: FK → users.id?          # 검토한 어드민
├── created_at: DateTime(tz)
└── updated_at: DateTime(tz)

인덱스:
- (target_type, target_id) — 대상별 신고 조회
- (status) — 미처리 신고 필터
- (reporter_id, target_type, target_id) UNIQUE — 중복 신고 방지
```

### 5.5 상태 정의

**이슈 상태 (IssueStatus)**

| 상태 | 코드 | 설명 |
|------|------|------|
| 진행중 | `ongoing` | 현재 진행 중인 이슈 |
| 종결 | `closed` | 마무리된 이슈 |
| 재점화 | `reignited` | 종결 후 재부상 |
| 확인필요 | `unverified` | 사실 확인 필요 |

**사건 중요도 (Importance)**

| 등급 | 코드 |
|------|------|
| 높음 | `high` |
| 중간 | `medium` |
| 낮음 | `low` |

**트리거 유형 (TriggerType)**

| 유형 | 코드 |
|------|------|
| 기사 | `article` |
| 판결 | `ruling` |
| 공식발표 | `announcement` |
| 정정 | `correction` |
| 상태변경 | `status_change` |

**신고 사유 (ReportReason)**

| 사유 | 코드 |
|------|------|
| 스팸 | `spam` |
| 욕설/비방 | `abuse` |
| 허위정보 | `misinformation` |
| 부적절 | `inappropriate` |
| 기타 | `other` |

**신고 처리 상태 (ReportStatus)**

| 상태 | 코드 |
|------|------|
| 대기중 | `pending` |
| 검토완료 | `reviewed` |
| 기각 | `dismissed` |

---

## 6. 인증/인가

### 6.1 인증 흐름

```
1. 운영자가 trend-korea-app 또는 API를 통해 로그인한다
   → trend-korea-api가 JWT Access Token + Refresh Token 발급

2. 운영자가 trend-korea-admin 로그인 페이지에서 Access Token을 입력한다
   → 또는 동일 도메인 쿠키를 통해 자동 전달

3. trend-korea-admin의 middleware.ts가 JWT Access Token을 검증한다
   → jose 라이브러리로 서명 검증
   → payload에서 role 확인 → admin이 아니면 403

4. 모든 Route Handler에서 동일한 JWT 검증을 수행한다
```

### 6.2 JWT 검증 구현

```typescript
// shared/lib/auth.ts
import { jwtVerify } from "jose";

interface JWTPayload {
  sub: string;       // user.id
  role: string;      // "guest" | "member" | "admin"
  typ: string;       // "access" | "refresh"
  exp: number;
}

async function verifyAdminToken(token: string): Promise<JWTPayload> {
  const secret = new TextEncoder().encode(process.env.JWT_SECRET);
  const { payload } = await jwtVerify(token, secret);

  if (payload.typ !== "access") {
    throw new Error("Access token required");
  }

  if (payload.role !== "admin") {
    throw new Error("Admin access required");
  }

  return payload as unknown as JWTPayload;
}
```

### 6.3 미들웨어 보호 범위

| 경로 패턴 | 보호 수준 |
|-----------|----------|
| `/login` | 미인증 허용 |
| `/api/auth/me` | JWT 검증 (role 무관) |
| `/api/*` | JWT 검증 + admin role 필수 |
| `/(admin)/*` | JWT 검증 + admin role 필수 |

### 6.4 로그인 페이지

어드민 로그인은 trend-korea-api의 기존 로그인 API를 호출하여 JWT를 받는다.

| 단계 | 동작 |
|------|------|
| 1 | 운영자가 이메일 + 비밀번호를 입력한다 |
| 2 | 프론트엔드가 `POST {API_BASE_URL}/api/v1/auth/login`을 호출한다 |
| 3 | 응답으로 받은 Access Token을 httpOnly 쿠키에 저장한다 |
| 4 | JWT payload의 `role`이 `admin`이 아니면 "권한 없음" 메시지를 표시한다 |
| 5 | admin이면 대시보드(`/`)로 리다이렉트한다 |

---

## 7. 기능 요구사항

### REQ-AD1: 대시보드

#### Feature: 대시보드 (FR-AD1) — P1

- **사용자 스토리**: 어드민으로서 서비스 주요 지표와 최근 활동을 한눈에 확인할 수 있다

| Spec ID | 명세 | Route Handler | 데이터 소스 |
|---------|------|--------------|------------|
| S-AD1-1 | 어드민이 오늘 등록된 사건/트리거 수, 활성 이슈 수를 확인한다 | `GET /api/dashboard/stats` | `events`, `triggers`, `issues` 집계 쿼리 |
| S-AD1-2 | 어드민이 오늘 가입/활성 사용자 수를 확인한다 | `GET /api/dashboard/stats` | `users` WHERE `created_at >= today` 및 `is_active = true` 집계 |
| S-AD1-3 | 어드민이 최근 잡 실행 이력(성공/실패)을 확인한다 | `GET /api/dashboard/recent-jobs` | `job_runs` ORDER BY `started_at` DESC LIMIT 10 |
| S-AD1-4 | 어드민이 최근 신고된 게시글/댓글 목록을 확인한다 | `GET /api/dashboard/reports` | `reports` WHERE `status = 'pending'` ORDER BY `created_at` DESC LIMIT 10 |

**Userflow (대시보드)**:
1. 어드민이 `/` (대시보드)에 접속한다
2. 상단 카드 영역에서 오늘 사건 수, 트리거 수, 활성 이슈 수, 신규 가입자 수를 확인한다
3. 최근 잡 실행 이력 테이블에서 성공/실패 상태를 확인한다
4. 미처리 신고 목록에서 건수를 확인하고, 클릭하면 `/moderation`으로 이동한다

**API 응답 형태**:

```typescript
// GET /api/dashboard/stats
interface DashboardStats {
  todayEvents: number;
  todayTriggers: number;
  activeIssues: number;
  todayNewUsers: number;
  activeUsers: number;
  pendingReports: number;
}

// GET /api/dashboard/recent-jobs
interface RecentJob {
  id: string;
  jobName: string;
  status: "success" | "failed";
  detail: string | null;
  startedAt: string;
  finishedAt: string | null;
}

// GET /api/dashboard/reports
interface RecentReport {
  id: string;
  targetType: "post" | "comment";
  targetId: string;
  targetTitle: string;       // post.title 또는 comment.content 앞 50자
  reason: string;
  reporterNickname: string;
  createdAt: string;
}
```

---

### REQ-AD2: 사건/이슈/트리거 CRUD

#### Feature: 사건 관리 (FR-AD2-Event) — P0

- **사용자 스토리**: 어드민으로서 사건을 생성/수정/삭제하여 데이터를 관리할 수 있다

| Spec ID | 명세 | Route Handler |
|---------|------|--------------|
| S-AD2-1 | 어드민이 사건을 생성한다 (제목, 요약, 중요도, 발생일시, 검증상태, 태그, 출처) | `POST /api/events` |
| S-AD2-2 | 어드민이 사건을 수정한다 | `PATCH /api/events/[id]` |
| S-AD2-3 | 어드민이 사건을 삭제한다 | `DELETE /api/events/[id]` |

**Userflow (사건 생성)**:
1. 어드민이 `/events`에서 "새 사건" 버튼을 클릭한다
2. `/events/new`에서 폼을 작성한다:
   - 제목 (필수, 50자)
   - 요약 (필수, 텍스트)
   - 발생일시 (필수, DateTimePicker)
   - 중요도 (필수, select: low/medium/high)
   - 검증상태 (필수, select: verified/unverified)
   - 태그 (선택, 다중 선택)
   - 출처 (선택, URL + 제목 + 매체명 반복 입력)
3. "저장" 버튼을 클릭한다
4. Route Handler가 Prisma 트랜잭션으로 `events`, `event_tags`, `sources`를 생성한다
5. 성공 시 `/events`로 리다이렉트하고 성공 토스트를 표시한다

**Userflow (사건 목록)**:
1. 어드민이 `/events`에 접속한다
2. 사건 목록이 테이블 형태로 표시된다 (제목, 발생일, 중요도, 검증상태, 출처수, 생성일)
3. 검색, 중요도 필터, 검증상태 필터, 정렬(발생일/생성일)을 조합하여 조회한다
4. 페이지네이션 (page + limit, 기본 20건)
5. 행 클릭 시 `/events/[id]` 수정 페이지로 이동한다

**엣지 케이스**:
- 삭제 시 확인 다이얼로그 표시, 관련 `event_tags`, `sources`, `issue_events`, `user_saved_events` 연관 데이터도 삭제
- 제목 중복 허용 (같은 사건이 다른 관점으로 등록될 수 있음)
- 발생일시가 미래인 경우 경고 표시 (저장은 허용)

**API 요청/응답 형태**:

```typescript
// POST /api/events
interface CreateEventRequest {
  title: string;              // max 50
  summary: string;
  occurredAt: string;         // ISO 8601
  importance: "low" | "medium" | "high";
  verificationStatus: "verified" | "unverified";
  tagIds?: string[];
  sources?: {
    url: string;
    title: string;
    publisher: string;
    publishedAt?: string;
  }[];
}

// GET /api/events
interface EventListParams {
  page?: number;              // 기본 1
  limit?: number;             // 기본 20
  search?: string;            // 제목/요약 검색
  importance?: string;
  verificationStatus?: string;
  sort?: "occurred_at" | "created_at";
  order?: "asc" | "desc";
}

interface EventListResponse {
  items: EventListItem[];
  total: number;
  page: number;
  limit: number;
}

interface EventListItem {
  id: string;
  title: string;
  occurredAt: string;
  importance: string;
  verificationStatus: string;
  sourceCount: number;
  tags: { id: string; name: string }[];
  createdAt: string;
}

// GET /api/events/[id]
interface EventDetail extends EventListItem {
  summary: string;
  sources: {
    id: string;
    url: string;
    title: string;
    publisher: string;
    publishedAt: string | null;
  }[];
  relatedIssues: { id: string; title: string; status: string }[];
  updatedAt: string;
}

// PATCH /api/events/[id]
interface UpdateEventRequest {
  title?: string;
  summary?: string;
  occurredAt?: string;
  importance?: "low" | "medium" | "high";
  verificationStatus?: "verified" | "unverified";
  tagIds?: string[];
  sources?: {
    url: string;
    title: string;
    publisher: string;
    publishedAt?: string;
  }[];
}
```

#### Feature: 이슈 관리 (FR-AD2-Issue) — P0

- **사용자 스토리**: 어드민으로서 이슈를 생성/수정/삭제하여 데이터를 관리할 수 있다

| Spec ID | 명세 | Route Handler |
|---------|------|--------------|
| S-AD2-4 | 어드민이 이슈를 생성한다 (제목, 설명, 상태, 태그, 관련 사건) | `POST /api/issues` |
| S-AD2-5 | 어드민이 이슈를 수정한다 | `PATCH /api/issues/[id]` |
| S-AD2-6 | 어드민이 이슈를 삭제한다 | `DELETE /api/issues/[id]` |

**Userflow (이슈 생성)**:
1. 어드민이 `/issues/new`에서 폼을 작성한다:
   - 제목 (필수, 50자)
   - 설명 (필수, 텍스트)
   - 상태 (필수, select: ongoing/closed/reignited/unverified)
   - 태그 (선택, 다중 선택)
   - 관련 사건 (선택, 사건 검색/다중 선택)
2. Route Handler가 `issues`, `issue_tags`, `issue_events`를 트랜잭션으로 생성한다
3. 성공 시 `/issues`로 리다이렉트한다

**Userflow (이슈 목록)**:
1. 테이블: 제목, 상태, 추적자 수(`tracker_count`), 최근 트리거 일시, 태그, 생성일
2. 상태 필터, 검색, 정렬(최근 트리거순/생성일순)
3. 페이지네이션 (기본 20건)

**엣지 케이스**:
- 이슈 삭제 시 관련 `triggers`, `issue_tags`, `issue_events`, `user_tracked_issues` 연관 데이터도 삭제
- 삭제 전 트리거 수가 0이 아니면 경고 표시

**API 요청/응답 형태**:

```typescript
// POST /api/issues
interface CreateIssueRequest {
  title: string;
  description: string;
  status: "ongoing" | "closed" | "reignited" | "unverified";
  tagIds?: string[];
  eventIds?: string[];
}

// GET /api/issues
interface IssueListParams {
  page?: number;
  limit?: number;
  search?: string;
  status?: string;
  sort?: "latest_trigger_at" | "created_at" | "tracker_count";
  order?: "asc" | "desc";
}

interface IssueListItem {
  id: string;
  title: string;
  status: string;
  trackerCount: number;
  latestTriggerAt: string | null;
  tags: { id: string; name: string }[];
  createdAt: string;
}

// GET /api/issues/[id]
interface IssueDetail extends IssueListItem {
  description: string;
  triggers: {
    id: string;
    occurredAt: string;
    summary: string;
    type: string;
  }[];
  relatedEvents: { id: string; title: string; occurredAt: string }[];
  updatedAt: string;
}
```

#### Feature: 트리거 관리 (FR-AD2-Trigger) — P0

- **사용자 스토리**: 어드민으로서 트리거를 생성/수정/삭제하여 이슈 업데이트를 관리할 수 있다

| Spec ID | 명세 | Route Handler |
|---------|------|--------------|
| S-AD2-7 | 어드민이 트리거를 생성한다 (이슈 선택, 요약, 유형, 발생일시, 출처) | `POST /api/triggers` |
| S-AD2-8 | 어드민이 트리거를 수정한다 | `PATCH /api/triggers/[id]` |
| S-AD2-9 | 어드민이 트리거를 삭제한다 | `DELETE /api/triggers/[id]` |

**Userflow (트리거 생성)**:
1. 어드민이 `/triggers/new`에서 폼을 작성한다:
   - 이슈 (필수, 이슈 검색/선택)
   - 요약 (필수, 텍스트)
   - 유형 (필수, select: article/ruling/announcement/correction/status_change)
   - 발생일시 (필수, DateTimePicker)
   - 출처 (선택, URL + 제목 + 매체명 반복 입력)
2. Route Handler가 트랜잭션으로 `triggers`, `sources`를 생성하고 `issues.latest_trigger_at`을 갱신한다
3. 성공 시 `/triggers`로 리다이렉트한다

**Userflow (트리거 목록)**:
1. 테이블: 요약, 유형, 이슈 제목, 발생일, 생성일
2. 이슈별 필터, 유형 필터, 검색, 정렬(발생일/생성일)
3. 페이지네이션 (기본 20건)

**부수 효과**:
- 트리거 생성 시: `issues.latest_trigger_at`을 해당 트리거의 `occurred_at`으로 갱신 (더 최신인 경우만)
- 트리거 삭제 시: `issues.latest_trigger_at`을 남은 트리거 중 최신값으로 재계산

**API 요청/응답 형태**:

```typescript
// POST /api/triggers
interface CreateTriggerRequest {
  issueId: string;
  summary: string;
  type: "article" | "ruling" | "announcement" | "correction" | "status_change";
  occurredAt: string;
  sources?: {
    url: string;
    title: string;
    publisher: string;
    publishedAt?: string;
  }[];
}

// GET /api/triggers
interface TriggerListParams {
  page?: number;
  limit?: number;
  issueId?: string;
  type?: string;
  search?: string;
  sort?: "occurred_at" | "created_at";
  order?: "asc" | "desc";
}

interface TriggerListItem {
  id: string;
  summary: string;
  type: string;
  occurredAt: string;
  issue: { id: string; title: string };
  createdAt: string;
}
```

---

### REQ-AD3: 태그/출처 관리

#### Feature: 태그 관리 (FR-AD3-Tag) — P0

- **사용자 스토리**: 어드민으로서 태그를 생성/수정/삭제하여 분류 체계를 관리할 수 있다

| Spec ID | 명세 | Route Handler |
|---------|------|--------------|
| S-AD3-1 | 어드민이 태그를 생성한다 (이름, 유형, slug) | `POST /api/tags` |
| S-AD3-2 | 어드민이 태그를 수정한다 | `PATCH /api/tags/[id]` |
| S-AD3-3 | 어드민이 태그를 삭제한다 | `DELETE /api/tags/[id]` |

**Userflow (태그 관리)**:
1. `/tags` 페이지에 태그 목록이 테이블 형태로 표시된다 (이름, 유형, slug, 사용 수)
2. 인라인 추가 또는 다이얼로그로 새 태그를 생성한다
3. 행 클릭 시 인라인 편집 또는 다이얼로그로 수정한다
4. 삭제 시 사용 중인 엔티티 수를 표시하고 확인을 요청한다

**엣지 케이스**:
- slug 중복 시 에러 메시지 표시
- 사용 중인 태그 삭제 시: `event_tags`, `issue_tags`, `post_tags` 연관 레코드도 삭제 (확인 다이얼로그에 영향 범위 표시)

**API 요청/응답 형태**:

```typescript
// POST /api/tags
interface CreateTagRequest {
  name: string;           // max 50
  type: "category" | "region";
  slug: string;           // max 80, unique
}

// GET /api/tags
interface TagListItem {
  id: string;
  name: string;
  type: string;
  slug: string;
  usageCount: number;     // event_tags + issue_tags + post_tags 합산
  updatedAt: string;
}
```

#### Feature: 출처 관리 (FR-AD3-Source) — P0

- **사용자 스토리**: 어드민으로서 출처를 추가/삭제하여 데이터의 신뢰도를 관리할 수 있다

| Spec ID | 명세 | Route Handler |
|---------|------|--------------|
| S-AD3-4 | 어드민이 출처를 추가한다 (엔티티 연결, URL, 제목, 매체명) | `POST /api/sources` |
| S-AD3-5 | 어드민이 출처를 삭제한다 | `DELETE /api/sources/[id]` |

**Userflow (출처 관리)**:
1. `/sources` 페이지에 출처 목록이 테이블 형태로 표시된다 (제목, URL, 매체명, 대상 유형, 대상 제목, 발행일)
2. 엔티티 유형(event/issue/trigger) 필터로 조회한다
3. 새 출처 추가 시: 엔티티 유형 선택 → 엔티티 검색/선택 → URL, 제목, 매체명, 발행일 입력
4. 삭제 시: 연결된 엔티티의 `source_count` 비정규화 필드 갱신 (events 테이블)

**부수 효과**:
- 출처 생성/삭제 시 `entity_type = 'event'`이면 해당 `events.source_count`를 재계산

**API 요청/응답 형태**:

```typescript
// POST /api/sources
interface CreateSourceRequest {
  entityType: "event" | "issue" | "trigger";
  entityId: string;
  url: string;            // max 500
  title: string;          // max 255
  publisher: string;      // max 100
  publishedAt?: string;
}

// GET /api/sources
interface SourceListParams {
  page?: number;
  limit?: number;
  entityType?: string;
  entityId?: string;
  search?: string;
}

interface SourceListItem {
  id: string;
  entityType: string;
  entityId: string;
  entityTitle: string;    // JOIN으로 가져온 대상 엔티티 제목
  url: string;
  title: string;
  publisher: string;
  publishedAt: string | null;
}
```

---

### REQ-AD4: 사용자 관리

#### Feature: 사용자 관리 (FR-AD4) — P1

- **사용자 스토리**: 어드민으로서 사용자 목록을 조회하고 역할 변경, 정지/복원을 수행할 수 있다

| Spec ID | 명세 | Route Handler |
|---------|------|--------------|
| S-AD4-1 | 어드민이 사용자 목록을 조회한다 (닉네임, 이메일, 역할, 상태, 가입일) | `GET /api/users` |
| S-AD4-2 | 어드민이 사용자 역할을 변경한다 (member ↔ admin) | `PATCH /api/users/[id]/role` |
| S-AD4-3 | 어드민이 사용자를 정지/복원한다 (`is_active` 토글) | `PATCH /api/users/[id]/status` |

**Userflow (사용자 관리)**:
1. `/users`에서 사용자 목록을 테이블로 확인한다 (닉네임, 이메일, 역할, 활성 상태, 가입일, 최근 활동)
2. 닉네임/이메일 검색, 역할 필터(guest/member/admin), 활성 상태 필터
3. 행 클릭 시 `/users/[id]` 상세 페이지로 이동한다
4. 상세 페이지에서: 프로필 정보, 작성 게시글/댓글 수, 역할 변경 버튼, 정지/복원 버튼

**엣지 케이스**:
- 자기 자신의 역할을 변경할 수 없다
- 자기 자신을 정지할 수 없다
- guest 역할은 admin으로 직접 승격 불가 (member를 거쳐야 함)
- 정지된 사용자는 목록에서 비활성 뱃지로 구분한다

**API 요청/응답 형태**:

```typescript
// GET /api/users
interface UserListParams {
  page?: number;
  limit?: number;
  search?: string;         // 닉네임 or 이메일
  role?: string;
  isActive?: boolean;
  sort?: "created_at" | "nickname";
  order?: "asc" | "desc";
}

interface UserListItem {
  id: string;
  nickname: string;
  email: string;
  role: string;
  isActive: boolean;
  createdAt: string;
  postCount: number;
  commentCount: number;
}

// GET /api/users/[id]
interface UserDetail extends UserListItem {
  profileImage: string | null;
  withdrawnAt: string | null;
  updatedAt: string;
}

// PATCH /api/users/[id]/role
interface UpdateRoleRequest {
  role: "member" | "admin";
}

// PATCH /api/users/[id]/status
interface UpdateStatusRequest {
  isActive: boolean;
}
```

---

### REQ-AD5: 커뮤니티 모더레이션

#### Feature: 커뮤니티 모더레이션 (FR-AD5) — P1

- **사용자 스토리**: 어드민으로서 게시글/댓글을 삭제하고, 신고된 콘텐츠를 검토할 수 있다

| Spec ID | 명세 | Route Handler |
|---------|------|--------------|
| S-AD5-1 | 어드민이 게시글을 삭제한다 (작성자와 무관하게) | `DELETE /api/posts/[id]` |
| S-AD5-2 | 어드민이 댓글을 삭제한다 | `DELETE /api/comments/[id]` |
| S-AD5-3 | 어드민이 신고된 콘텐츠를 검토한다 (승인=콘텐츠 삭제/기각) | `PATCH /api/reports/[id]` (추가 Route Handler 필요) |

**Userflow (모더레이션)**:
1. `/moderation`에서 세 개 탭으로 구성한다: 신고 대기 | 게시글 | 댓글
2. **신고 대기 탭**: `status = 'pending'`인 신고 목록 표시
   - 신고 사유, 대상 콘텐츠 미리보기, 신고자, 신고일
   - "삭제" 버튼: 대상 콘텐츠 삭제 + 신고 `status = 'reviewed'`로 변경
   - "기각" 버튼: 신고 `status = 'dismissed'`로 변경
3. **게시글 탭**: 전체 게시글 목록 (신고 건수 포함)
   - 제목, 작성자, 추천수, 댓글수, 신고수, 작성일
   - "삭제" 버튼으로 게시글 삭제
4. **댓글 탭**: 전체 댓글 목록 (신고 건수 포함)
   - 내용(앞 100자), 작성자, 좋아요수, 신고수, 작성일
   - "삭제" 버튼으로 댓글 삭제

**부수 효과**:
- 게시글 삭제 시: 해당 게시글의 모든 댓글, `post_tags`, `post_votes`, 관련 `reports`도 삭제
- 댓글 삭제 시: 하위 대댓글, `comment_likes`, 관련 `reports`도 삭제. `posts.comment_count` 재계산

**추가 Route Handler (신고 처리)**:

```
GET  /api/reports          — 신고 목록 (status 필터, 페이지네이션)
PATCH /api/reports/[id]    — 신고 처리 (reviewed/dismissed)
```

**API 요청/응답 형태**:

```typescript
// GET /api/reports
interface ReportListParams {
  page?: number;
  limit?: number;
  status?: "pending" | "reviewed" | "dismissed";
  targetType?: "post" | "comment";
}

interface ReportListItem {
  id: string;
  targetType: string;
  targetId: string;
  targetPreview: string;     // 대상 콘텐츠 미리보기 (100자)
  targetAuthor: string;      // 대상 작성자 닉네임
  reason: string;
  detail: string | null;
  reporterNickname: string;
  status: string;
  createdAt: string;
}

// PATCH /api/reports/[id]
interface UpdateReportRequest {
  status: "reviewed" | "dismissed";
  deleteTarget?: boolean;    // true면 대상 콘텐츠도 삭제
}

// GET /api/posts (어드민용)
interface AdminPostListItem {
  id: string;
  title: string;
  authorNickname: string;
  isAnonymous: boolean;
  likeCount: number;
  commentCount: number;
  reportCount: number;       // reports 테이블 집계
  createdAt: string;
}

// GET /api/comments (어드민용)
interface AdminCommentListItem {
  id: string;
  content: string;           // 앞 100자
  authorNickname: string;
  postTitle: string;
  likeCount: number;
  reportCount: number;
  createdAt: string;
}
```

---

### REQ-AD6: 데이터 파이프라인 모니터링

#### Feature: 파이프라인 모니터링 (FR-AD6) — P1

- **사용자 스토리**: 어드민으로서 뉴스 수집 파이프라인의 실행 이력과 키워드 상태를 모니터링하고, 수동 수집을 트리거할 수 있다

| Spec ID | 명세 | Route Handler | 데이터 소스 |
|---------|------|--------------|------------|
| S-AD6-1 | 어드민이 최근 뉴스 수집 결과를 확인한다 (수집 기사 수, 요약 수, 분류 결과) | `GET /api/pipeline/jobs` | `job_runs` 테이블 |
| S-AD6-2 | 어드민이 실패한 잡과 에러 로그를 확인한다 | `GET /api/pipeline/jobs?status=failed` | `job_runs` WHERE `status='failed'` |
| S-AD6-3 | 어드민이 키워드 상태(active/cooldown/closed)를 확인한다 | `GET /api/pipeline/keywords` | `issue_keyword_states` |
| S-AD6-4 | 어드민이 수동으로 뉴스 수집 사이클을 트리거한다 | `POST /api/pipeline/trigger-collect` | trend-korea-api 내부 호출 또는 DB 기반 플래그 |

**Userflow (파이프라인 모니터링)**:
1. `/pipeline`에서 두 개 섹션으로 구성한다: 잡 실행 이력 | 키워드 상태
2. **잡 실행 이력 섹션**:
   - 테이블: 잡 이름, 상태(성공/실패 뱃지), 상세, 시작시각, 종료시각, 소요시간
   - 잡 이름 필터 (news_collect, keyword_state_cleanup, search_rankings 등)
   - 상태 필터 (전체/성공/실패)
   - 기본 정렬: 시작시각 DESC
3. **키워드 상태 섹션**:
   - 테이블: 키워드, 이슈 제목, 상태(active/cooldown/closed), 최근 확인 시각, 생성일
   - 상태별 필터
4. **수동 수집 트리거**:
   - "뉴스 수집 실행" 버튼 클릭
   - 확인 다이얼로그 표시
   - 실행 후 잡 실행 이력에 새 레코드가 추가된다

**S-AD6-4 수동 수집 구현 방식**:

> trend-korea-api의 스케줄러에 HTTP 트리거 엔드포인트가 없으므로, 다음 두 가지 방식 중 하나를 선택한다.
>
> **방식 A (권장)**: trend-korea-api에 `POST /api/v1/admin/trigger-collect` 엔드포인트를 추가하고, 어드민 Route Handler에서 이를 호출한다.
>
> **방식 B (대안)**: `job_runs` 테이블에 `status='pending'`, `job_name='manual_news_collect'` 레코드를 삽입하고, 스케줄러가 이를 폴링하여 실행한다.
>
> 이 결정은 구현 시점에 trend-korea-api 측과 협의한다. (Ask First 항목)

**API 요청/응답 형태**:

```typescript
// GET /api/pipeline/jobs
interface PipelineJobListParams {
  page?: number;
  limit?: number;
  jobName?: string;
  status?: "success" | "failed";
}

interface PipelineJobListItem {
  id: string;
  jobName: string;
  status: string;
  detail: string | null;     // JSON 문자열 (수집 기사 수 등 포함)
  startedAt: string;
  finishedAt: string | null;
  durationSeconds: number | null;
}

// GET /api/pipeline/keywords
interface KeywordStateListParams {
  page?: number;
  limit?: number;
  status?: "active" | "cooldown" | "closed";
}

interface KeywordStateListItem {
  id: string;
  keyword: string;
  issueTitle: string | null;
  status: string;
  lastSeenAt: string;
  createdAt: string;
}

// POST /api/pipeline/trigger-collect
// 요청 본문 없음
// 응답: { message: "수집이 시작되었습니다", jobRunId: string }
```

---

## 8. 유즈케이스 (심화)

### UC-AD1: 사건 생성 (태그 + 출처 포함)

- **선행 조건**: 어드민이 로그인되어 있고 admin role을 갖고 있다
- **정상 흐름**:
  1. 어드민이 `/events/new`로 이동한다
  2. 폼 필드를 입력한다 (제목, 요약, 발생일시, 중요도, 검증상태)
  3. 태그를 다중 선택한다 (기존 태그 목록에서 검색)
  4. 출처를 1개 이상 추가한다 (URL, 제목, 매체명, 발행일)
  5. "저장" 버튼을 클릭한다
  6. Route Handler가 Prisma 트랜잭션으로 다음을 수행한다:
     - `events` 레코드 생성 (UUID v4 생성)
     - `event_tags` 연관 레코드 생성
     - `sources` 레코드 생성 (`entity_type = 'event'`)
     - `events.source_count` 갱신
  7. 성공 시 `/events`로 리다이렉트하고 "사건이 생성되었습니다" 토스트를 표시한다
- **예외 흐름**:
  - 필수 필드 누락 → 클라이언트 측 폼 검증 에러 (Zod)
  - slug 중복 등 DB 제약 위반 → 400 에러 + 구체적 메시지
  - JWT 만료 → 401 에러 → 로그인 페이지로 리다이렉트
- **성공 조건**: `events` 테이블에 새 레코드가 생성되고, 관련 태그/출처가 정확히 연결된다

### UC-AD2: 신고 콘텐츠 검토 후 삭제

- **선행 조건**: `reports` 테이블에 `status = 'pending'`인 신고가 1건 이상 존재한다
- **정상 흐름**:
  1. 어드민이 `/moderation`에서 "신고 대기" 탭을 확인한다
  2. 신고된 게시글의 내용을 미리보기로 확인한다
  3. "삭제" 버튼을 클릭한다
  4. 확인 다이얼로그: "이 게시글을 삭제하시겠습니까? 관련 댓글 N개도 함께 삭제됩니다."
  5. Route Handler가 트랜잭션으로 다음을 수행한다:
     - `reports` 레코드 `status = 'reviewed'`, `reviewed_at = now()`, `reviewed_by = 현재 어드민 ID` 갱신
     - 대상 게시글의 모든 댓글/대댓글 삭제
     - `comment_likes`, `post_votes`, `post_tags` 연관 레코드 삭제
     - 게시글 삭제
     - 해당 게시글/댓글에 대한 다른 `pending` 신고도 `reviewed`로 일괄 변경
  6. 목록에서 해당 신고가 사라지고 "처리 완료" 토스트를 표시한다
- **예외 흐름**:
  - 대상 콘텐츠가 이미 삭제된 경우 → "이미 삭제된 콘텐츠입니다" 메시지 + 신고 `status = 'reviewed'`로 자동 변경
- **성공 조건**: 신고 상태가 `reviewed`로 변경되고, 대상 콘텐츠와 연관 데이터가 삭제된다

### UC-AD3: 사용자 정지

- **선행 조건**: 대상 사용자가 `is_active = true`이고, 어드민 자신이 아니다
- **정상 흐름**:
  1. 어드민이 `/users/[id]`에서 "사용자 정지" 버튼을 클릭한다
  2. 확인 다이얼로그: "이 사용자를 정지하시겠습니까?"
  3. Route Handler가 `users.is_active = false`로 갱신한다
  4. "사용자가 정지되었습니다" 토스트를 표시한다
- **예외 흐름**:
  - 자기 자신을 정지 시도 → "자기 자신은 정지할 수 없습니다" 에러
  - 이미 정지된 사용자 → "이미 정지된 사용자입니다" 에러
- **성공 조건**: `users.is_active = false`로 갱신. 정지된 사용자는 trend-korea-app에서 로그인 불가

---

## 9. 화면 구성

### 9.1 정보 구조 (IA)

```
/login                    # 로그인 (인증 전 유일한 접근 가능 페이지)
/                         # 대시보드 (FR-AD1)
/events                   # 사건 목록 (FR-AD2)
/events/new               # 사건 생성
/events/[id]              # 사건 수정
/issues                   # 이슈 목록 (FR-AD2)
/issues/new               # 이슈 생성
/issues/[id]              # 이슈 수정
/triggers                 # 트리거 목록 (FR-AD2)
/triggers/new             # 트리거 생성
/triggers/[id]            # 트리거 수정
/tags                     # 태그 관리 (FR-AD3)
/sources                  # 출처 관리 (FR-AD3)
/users                    # 사용자 목록 (FR-AD4)
/users/[id]               # 사용자 상세
/moderation               # 커뮤니티 모더레이션 (FR-AD5)
/pipeline                 # 파이프라인 모니터링 (FR-AD6)
```

### 9.2 공통 레이아웃

```
┌─────────────────────────────────────────────────────┐
│ Header: 로고 | 현재 페이지 | 알림 | 프로필 드롭다운 │
├───────────┬─────────────────────────────────────────┤
│           │                                         │
│ Sidebar   │  Main Content                           │
│           │                                         │
│ 대시보드  │  (각 페이지 내용)                        │
│ 사건      │                                         │
│ 이슈      │                                         │
│ 트리거    │                                         │
│ 태그      │                                         │
│ 출처      │                                         │
│ ────────  │                                         │
│ 사용자    │                                         │
│ 모더레이션│                                         │
│ ────────  │                                         │
│ 파이프라인│                                         │
│           │                                         │
└───────────┴─────────────────────────────────────────┘
```

### 9.3 핵심 UI 패턴

| 패턴 | 사용 위치 | shadcn/ui 컴포넌트 |
|------|----------|-------------------|
| 데이터 테이블 | 모든 목록 페이지 | `Table` + `DataTable` (TanStack Table) |
| 폼 | 생성/수정 페이지 | `Form` + `Input` + `Select` + `Textarea` |
| 다이얼로그 | 삭제 확인, 인라인 편집 | `AlertDialog`, `Dialog` |
| 토스트 | 성공/에러 알림 | `Sonner` (toast) |
| 뱃지 | 상태 표시 | `Badge` |
| 카드 | 대시보드 지표 | `Card` |
| 탭 | 모더레이션 탭 전환 | `Tabs` |
| 페이지네이션 | 모든 목록 | `Pagination` |
| 검색 | 목록 필터링 | `Input` (debounce 300ms) |
| 날짜 선택 | 발생일시 입력 | `DatePicker` (date-fns) |

### 9.4 반응형

- **데스크톱 우선** (1280px+): 사이드바 + 메인 콘텐츠 2컬럼
- **태블릿** (768px~1279px): 사이드바 접기 (아이콘만), 메인 콘텐츠 전체 너비
- **모바일**: 지원하지 않음 (관리 도구 특성상 데스크톱/태블릿만 지원)

---

## 10. 기술 스택

| 레이어 | 기술 | 선택 근거 |
|--------|------|----------|
| 프레임워크 | Next.js (App Router) | 풀스택 (페이지 + Route Handlers), RSC 지원 |
| 언어 | TypeScript 5.9 (strict mode) | 타입 안전성 |
| 스타일링 | Tailwind CSS 4 | 유틸리티 퍼스트, shadcn/ui와 통합 |
| UI | shadcn/ui | 어드민에 적합한 컴포넌트 (테이블, 폼, 다이얼로그) |
| ORM | Prisma | 기존 DB introspection, 타입 안전한 쿼리, 트랜잭션 |
| 데이터 페칭 | React Query (TanStack Query) | 캐시, 낙관적 업데이트, 뮤테이션 |
| 폼 검증 | Zod + React Hook Form | 서버/클라이언트 양방향 검증 |
| 테이블 | TanStack Table | 정렬, 필터, 페이지네이션 |
| JWT | jose | JWT 검증 (Node.js 호환) |
| 패키지 매니저 | pnpm | 빠른 설치, 디스크 효율 |
| Node.js | v24.13.0 | 최신 LTS |
| 린팅 | ESLint 9 (flat config) + Prettier | 코드 품질 |

### 환경변수

```env
# .env.local
DATABASE_URL="postgresql://user:password@localhost:5432/trend_korea"
JWT_SECRET="trend-korea-api와 동일한 시크릿"
NEXT_PUBLIC_API_BASE_URL="http://localhost:8000"   # 로그인 시 trend-korea-api 호출
```

---

## 11. Route Handlers 전체 목록

### 인증

| 메서드 | 경로 | 설명 | 비고 |
|--------|------|------|------|
| `GET` | `/api/auth/me` | 현재 로그인한 어드민 정보 | JWT에서 user_id 추출 → Prisma 조회 |

### 사건 (Events)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/events` | 목록 (필터/정렬/페이지네이션) |
| `GET` | `/api/events/[id]` | 상세 (태그, 출처, 관련 이슈 포함) |
| `POST` | `/api/events` | 생성 (트랜잭션: events + event_tags + sources) |
| `PATCH` | `/api/events/[id]` | 수정 (트랜잭션: events + event_tags + sources 교체) |
| `DELETE` | `/api/events/[id]` | 삭제 (트랜잭션: 연관 데이터 포함) |

### 이슈 (Issues)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/issues` | 목록 |
| `GET` | `/api/issues/[id]` | 상세 (트리거, 관련 사건 포함) |
| `POST` | `/api/issues` | 생성 (트랜잭션: issues + issue_tags + issue_events) |
| `PATCH` | `/api/issues/[id]` | 수정 |
| `DELETE` | `/api/issues/[id]` | 삭제 (트랜잭션: 연관 데이터 포함) |

### 트리거 (Triggers)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/triggers` | 목록 (issue_id 필터) |
| `POST` | `/api/triggers` | 생성 (트랜잭션: triggers + sources + issues.latest_trigger_at 갱신) |
| `PATCH` | `/api/triggers/[id]` | 수정 |
| `DELETE` | `/api/triggers/[id]` | 삭제 (issues.latest_trigger_at 재계산) |

### 태그 (Tags)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/tags` | 목록 (사용 수 포함) |
| `POST` | `/api/tags` | 생성 |
| `PATCH` | `/api/tags/[id]` | 수정 |
| `DELETE` | `/api/tags/[id]` | 삭제 (연관 레코드 포함) |

### 출처 (Sources)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/sources` | 목록 (entity_type/entity_id 필터) |
| `POST` | `/api/sources` | 생성 (events.source_count 갱신) |
| `DELETE` | `/api/sources/[id]` | 삭제 (events.source_count 갱신) |

### 사용자 (Users)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/users` | 목록 (검색, 역할 필터) |
| `GET` | `/api/users/[id]` | 상세 |
| `PATCH` | `/api/users/[id]/role` | 역할 변경 |
| `PATCH` | `/api/users/[id]/status` | 활성/정지 토글 |

### 커뮤니티 (Posts/Comments)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/posts` | 게시글 목록 (신고 건수 포함) |
| `DELETE` | `/api/posts/[id]` | 게시글 삭제 (연관 데이터 포함) |
| `GET` | `/api/comments` | 댓글 목록 (신고 건수 포함) |
| `DELETE` | `/api/comments/[id]` | 댓글 삭제 (연관 데이터 포함) |

### 신고 (Reports)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/reports` | 신고 목록 (status/targetType 필터) |
| `PATCH` | `/api/reports/[id]` | 신고 처리 (reviewed/dismissed, 선택적 콘텐츠 삭제) |

### 대시보드 (Dashboard)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/dashboard/stats` | 주요 지표 집계 |
| `GET` | `/api/dashboard/recent-jobs` | 최근 잡 실행 이력 10건 |
| `GET` | `/api/dashboard/reports` | 미처리 신고 10건 |

### 파이프라인 (Pipeline)

| 메서드 | 경로 | 설명 |
|--------|------|------|
| `GET` | `/api/pipeline/jobs` | 잡 실행 이력 (필터/페이지네이션) |
| `GET` | `/api/pipeline/keywords` | 키워드 상태 목록 |
| `POST` | `/api/pipeline/trigger-collect` | 수동 뉴스 수집 트리거 |

**총 Route Handler 메서드: 38개**

---

## 12. 비기능 요구사항

### 12.1 성능

| 지표 | 목표 | 비고 |
|------|------|------|
| 페이지 초기 로드 | < 3s | 관리 도구이므로 사용자 서비스보다 관대 |
| Route Handler 응답 (P95) | < 1s | Prisma 쿼리 최적화 |
| 대시보드 집계 쿼리 | < 2s | 인덱스 활용 |

### 12.2 보안

- 모든 페이지/API에 admin role guard 적용 (middleware.ts)
- CSRF: Next.js 기본 CSRF 보호 + SameSite 쿠키
- SQL injection: Prisma ORM 파라미터 바인딩으로 방지
- XSS: React의 기본 이스케이핑 + 사용자 입력 sanitize
- 환경변수: `.env.local`에 시크릿 저장, 코드에 하드코딩 금지

### 12.3 데이터 정합성

- trend-korea-api와 동일 DB를 공유하므로 **Prisma 트랜잭션** 필수
- 비정규화 필드(`source_count`, `comment_count`, `tracker_count`) 갱신 시 트랜잭션 내에서 처리
- 삭제 시 연관 데이터 cascade 삭제를 명시적으로 수행 (DB FK cascade에 의존하지 않고 Prisma 코드로 제어)

### 12.4 접근성

- shadcn/ui 기본 접근성 수준 유지
- 키보드 네비게이션 (Tab, Enter, Escape)

---

## 13. AI 에이전트 Boundaries

### Always (항상 지킬 것)

- shadcn/ui 컴포넌트 사용, HTML 요소 직접 사용 금지
- `features -> shared` 의존 방향 준수, feature 간 직접 import 금지
- named export 우선, `import { type Foo }` 인라인 형식
- 모든 Route Handler에 JWT admin role 검증 포함
- DB 쓰기 작업은 Prisma 트랜잭션 사용
- 비정규화 필드 갱신을 트랜잭션 내에서 처리
- Zod 스키마로 요청 본문 검증 (Route Handler 진입부)
- ESLint + Prettier 린트 통과
- TypeScript strict mode
- 도메인 용어: 코드 식별자는 영문 용어 사전 준수

### Ask First (먼저 확인할 것)

- DB 스키마 변경 (Alembic 마이그레이션은 trend-korea-api 측에서 수행)
- 수동 뉴스 수집 트리거 구현 방식 (방식 A vs 방식 B)
- 새 외부 API/서비스 의존성 추가
- 인증 흐름 변경 (JWT 시크릿, 쿠키 정책 등)
- 새 feature 모듈 추가
- Prisma 스키마 수동 편집 (`prisma db pull`이 아닌 경우)

### Never (절대 하지 말 것)

- 프로덕션 DB 직접 접근 또는 데이터 삭제
- `.env`, 시크릿 키, API 키를 코드에 하드코딩
- JWT 검증 우회 또는 role guard 생략
- `prisma migrate dev`로 직접 마이그레이션 생성 (Alembic이 주관)
- feature 간 직접 import (반드시 `index.ts` 경유)
- Prisma Client를 Route Handler에서 직접 사용 (shared/lib/prisma.ts 싱글톤 경유)
- 테스트 없이 비즈니스 로직 변경

### Commands

```bash
# 개발
pnpm dev                    # 개발 서버 (:3200)
pnpm lint                   # ESLint
pnpm type:check             # TypeScript 타입 체크
pnpm format:check           # Prettier
pnpm build                  # 프로덕션 빌드

# Prisma
pnpm prisma db pull         # DB → schema.prisma 동기화
pnpm prisma generate        # Prisma Client 생성
pnpm prisma studio          # DB 브라우저 (개발용)

# 테스트
pnpm test                   # 단위 테스트
pnpm test:e2e               # E2E 테스트 (Playwright)
```

### Testing

| 레벨 | 대상 | 도구 |
|------|------|------|
| 단위 | Route Handler 로직, 유틸 함수 | Vitest |
| 통합 | Route Handler + Prisma | Vitest + 테스트 DB |
| E2E | 주요 Userflow (CRUD, 로그인) | Playwright |
| 커버리지 기준 | Route Handler 80%+ | — |

### Git Workflow

- 브랜치: `main` (프로덕션), `feature/*`, `fix/*`
- 커밋 메시지: 한국어, Conventional Commits 스타일
- PR: 기능 단위, 린트/테스트 통과 필수

---

## 14. 구현 로드맵

### Phase 1: MVP (2~3주)

- [ ] 프로젝트 초기화 (Next.js + Prisma + shadcn/ui + Tailwind CSS 4)
- [ ] Prisma DB introspection (`prisma db pull`) 및 Client 생성
- [ ] JWT 인증 미들웨어 + 로그인 페이지
- [ ] 공통 레이아웃 (사이드바 + 헤더)
- [ ] **사건 CRUD** (S-AD2-1 ~ S-AD2-3): 목록/생성/수정/삭제
- [ ] **이슈 CRUD** (S-AD2-4 ~ S-AD2-6): 목록/생성/수정/삭제
- [ ] **트리거 CRUD** (S-AD2-7 ~ S-AD2-9): 목록/생성/수정/삭제
- [ ] **태그 CRUD** (S-AD3-1 ~ S-AD3-3): 목록/생성/수정/삭제
- [ ] **출처 관리** (S-AD3-4 ~ S-AD3-5): 목록/추가/삭제

### Phase 2: 운영 도구 (1~2주)

- [ ] **대시보드** (S-AD1-1 ~ S-AD1-3): 지표 카드, 잡 이력
- [ ] **사용자 관리** (S-AD4-1 ~ S-AD4-3): 목록/역할 변경/정지
- [ ] **파이프라인 모니터링** (S-AD6-1 ~ S-AD6-3): 잡 이력, 키워드 상태
- [ ] reports 테이블 추가 (Alembic 마이그레이션, trend-korea-api 측)

### Phase 3: 모더레이션 + 수동 트리거 (1주)

- [ ] **커뮤니티 모더레이션** (S-AD5-1 ~ S-AD5-3): 게시글/댓글 삭제, 신고 검토
- [ ] **대시보드 신고** (S-AD1-4): 신고 목록 위젯
- [ ] **수동 뉴스 수집** (S-AD6-4): 트리거 구현

---

## 15. 미결 사항 / 추후 결정

| 항목 | 상태 | 비고 |
|------|------|------|
| 수동 뉴스 수집 트리거 구현 방식 | 미결정 | 방식 A (API 추가) vs 방식 B (DB 플래그) |
| reports 테이블 생성 | 미구현 | Phase 2에서 Alembic 마이그레이션으로 추가 |
| 어드민 로그인 UX | 미결정 | API 직접 호출 vs 공유 쿠키 |
| 배포 환경 | 미결정 | Vercel / 자체 서버 |
| 로그 조회 상세 | 미결정 | `job_runs.detail` JSON 파싱 범위 |
| 배치 작업 (일괄 삭제/태그 변경) | Phase 3+ | 데이터 양이 늘어나면 필요 |
| 감사 로그 (audit log) | Phase 3+ | 어드민 작업 이력 추적 |

---

## 16. 용어 사전

| 한글 | 영문 | 코드 | DB 테이블 | 설명 |
|------|------|------|----------|------|
| 사건 | Event | `Event` | `events` | 특정 일자에 발생한 단일 이벤트 |
| 이슈 | Issue | `Issue` | `issues` | 언론/SNS에서 지속 추적되는 주제 |
| 트리거 | Trigger | `Trigger` | `triggers` | 이슈에 대한 새로운 업데이트 |
| 태그 | Tag | `Tag` | `tags` | 사건/이슈 분류 라벨 |
| 출처 | Source | `Source` | `sources` | 뉴스 기사, 공식 발표 등 참고 자료 |
| 게시글 | Post | `Post` | `posts` | 커뮤니티 게시글 |
| 댓글 | Comment | `Comment` | `comments` | 게시글에 대한 댓글/대댓글 |
| 사용자 | User | `User` | `users` | 서비스 사용자 |
| 신고 | Report | `Report` | `reports` | 부적절 콘텐츠 신고 (신규) |
| 잡 실행 | Job Run | `JobRun` | `job_runs` | 스케줄러 잡 실행 이력 |

---

> **완성도**: 확정 (100) — 코드베이스 실사 및 통합 PRD 기반 상세 명세 (2026-03-09)
