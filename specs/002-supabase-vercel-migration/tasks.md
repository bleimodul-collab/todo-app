---
description: "Task list for demodev Tasks — Supabase + Vercel 마이그레이션"
---

# Tasks: demodev Tasks — Supabase + Vercel 마이그레이션

**Input**: Design documents from `specs/002-supabase-vercel-migration/`
**Prerequisites**: plan.md, spec.md

**Tests**: 인증 플로우(OAuth)는 E2E 테스트 없이 수동 검증 허용.
Supabase 클라이언트 래퍼·데이터 변환 함수는 단위 테스트 권장 (Constitution 원칙 I 예외 적용).

**Organization**: Phase 1 Setup → Phase 2 Foundational → Phase 3 US1(로그인) →
Phase 4 US2(데이터 저장) → Phase 5 US3(로그아웃) → Phase 6 US4(배포) →
Phase 7 US5(다기기) → Phase 8 Polish

## Format: `[ID] [P?] [Story] Description`

- **[P]**: 병렬 가능 (다른 파일, 미완료 작업에 의존하지 않음)
- **[Story]**: 해당 user story (US1~US5). Setup/Foundational/Polish은 라벨 없음.
- 모든 작업에 정확한 파일 경로 포함

---

## Phase 1: Setup

**Purpose**: Supabase 프로젝트 생성, 패키지 설치, 환경 변수 설정

- [ ] T001 Supabase 프로젝트 생성 — 대시보드(supabase.com)에서 프로젝트 생성 후
  `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` 를 `.env.local` 에 저장
- [ ] T002 [P] Supabase 패키지 설치 — `npm install @supabase/supabase-js @supabase/ssr`
  실행 후 `package.json` 확인
- [ ] T003 [P] Supabase 대시보드 OAuth 설정 — Authentication → Providers에서
  Google·GitHub OAuth 앱 등록 및 Client ID/Secret 입력; Site URL = `http://localhost:3000`,
  Redirect URL = `http://localhost:3000/auth/callback`

---

## Phase 2: Foundational (Blocking Prerequisites)

**Purpose**: 모든 user story가 공유하는 인프라 — Supabase 클라이언트, DB 스키마,
RLS 정책, OAuth 콜백 라우트

**⚠️ CRITICAL**: 이 단계가 끝나기 전에는 어떤 user story도 시작할 수 없다.

- [ ] T004 [P] 브라우저용 Supabase 클라이언트 — `src/lib/supabase/client.ts`
  (`createBrowserClient` 사용, NEXT_PUBLIC_ 환경 변수 읽기)
- [ ] T005 [P] 서버용 Supabase 클라이언트 — `src/lib/supabase/server.ts`
  (`createServerClient` + cookies 설정, Server Component/Route Handler용)
- [ ] T006 [P] 미들웨어용 Supabase 클라이언트 — `src/lib/supabase/middleware.ts`
  (`createServerClient` + request/response cookie 갱신)
- [ ] T007 DB 스키마 마이그레이션 작성 — `supabase/migrations/001_initial_schema.sql`
  (tasks 테이블: id, user_id, title, due_date, priority_id, category_id, starred,
  done, note, created_at; subtasks 테이블: id, task_id, text, done; 인덱스 포함)
- [ ] T008 Row Level Security 정책 적용 — `supabase/migrations/001_initial_schema.sql`
  에 RLS enable + `user_id = auth.uid()` SELECT·INSERT·UPDATE·DELETE 정책 추가;
  Supabase 대시보드 또는 CLI로 마이그레이션 실행
- [ ] T009 OAuth 콜백 라우트 핸들러 — `src/app/auth/callback/route.ts`
  (`exchangeCodeForSession` 호출 후 `/main` 리다이렉트; 오류 시 `/` 리다이렉트)
- [ ] T010 Next.js 미들웨어 — `src/middleware.ts`
  (Supabase 세션 갱신 + `/main`,`/calendar`,`/stats` 접근 시 미인증 사용자를 `/`
  리다이렉트; `matcher` 패턴 설정)

**Checkpoint**: Supabase 클라이언트·스키마·RLS·콜백·미들웨어 준비 완료 —
user story 구현 시작 가능.

---

## Phase 3: User Story 1 — 계정으로 로그인한다 (Priority: P1) 🎯 MVP

**Goal**: Google/GitHub OAuth 실제 로그인 → 메인 화면 이동, 미인증 시 보호 라우트 차단.

**Independent Test**: 로그인 화면에서 Google 버튼 클릭 → Google 동의 화면 → 메인
화면으로 이동. 메인 URL 직접 접근 시 로그인 화면으로 리다이렉트됨을 확인.

### Implementation for User Story 1

- [ ] T011 [US1] LoginScreen Supabase Auth 연결 — `src/components/login/LoginScreen.tsx`
  에서 Google·GitHub 버튼 onClick을 `supabase.auth.signInWithOAuth({provider, redirectTo})`
  호출로 교체; 이메일/비밀번호 핸들러는 제거하고 버튼은 시각적으로 유지
- [ ] T012 [US1] 로그인 페이지 서버 리다이렉트 — `src/app/page.tsx` 에서
  이미 로그인된 사용자가 `/` 접근 시 `/main` 으로 서버 사이드 리다이렉트 추가
- [ ] T013 [US1] OAuth 콜백 수동 검증 — `T009` 콜백 라우트에서 코드 교환이 정상
  동작하는지 브라우저로 로그인 흐름 전체 수동 검증; 성공 시 T011~T012 완료 확인

**Checkpoint**: US1 독립 동작 — 실제 Google/GitHub 로그인 및 보호 라우트 차단 완성.

---

## Phase 4: User Story 2 — 할 일 데이터가 클라우드에 저장된다 (Priority: P1)

**Goal**: 모든 할 일 CRUD가 Supabase PostgreSQL에 저장. 로그인 사용자 간 데이터 격리.
첫 로그인 시 샘플 데이터 시드.

**Independent Test**: 로그인 후 할 일 추가 → 브라우저 닫기 → 재접속 후 동일 데이터
표시. 다른 계정 로그인 시 이전 계정 데이터 보이지 않음.

### Implementation for User Story 2

- [ ] T014 [US2] Supabase 데이터 접근 유틸리티 — `src/lib/supabase/tasks-api.ts`
  (fetchTasks, createTask, updateTask, deleteTask, createSubtask, updateSubtask,
  deleteSubtask 함수 — 모두 `user_id` 필터 포함)
- [ ] T015 [P] [US2] 초기 시드 유틸리티 — `src/lib/supabase/seed.ts`
  (SAMPLE_TASKS를 Supabase tasks 테이블에 INSERT — 최초 로그인 시 데이터 없을 때만
  실행; `src/lib/store/sample-data.ts` 상수 재사용)
- [ ] T016 [US2] TasksProvider Supabase 교체 — `src/context/TasksProvider.tsx`
  를 Supabase 기반으로 교체: 마운트 시 `fetchTasks()` 호출, 각 action(add/update/
  delete/toggleDone/toggleStar/subtask CRUD)을 `tasks-api.ts` 로 위임, optimistic
  update 패턴 사용(UI 즉시 반영 → 서버 쓰기 → 실패 시 롤백)
- [ ] T017 [US2] 첫 로그인 시드 실행 — `src/context/TasksProvider.tsx` 에서
  `fetchTasks()` 결과가 빈 배열이면 `seed.ts` 의 시드 함수 호출
- [ ] T018 [P] [US2] 테마 설정 Supabase 저장 — `src/context/ThemeProvider.tsx` 에서
  테마 변경 시 `user_metadata` 또는 별도 `profiles` 테이블에 저장;
  로그인 시 저장된 테마 로드
- [ ] T019 [US2] 네트워크 오류 처리 — `src/context/TasksProvider.tsx` 에 API 오류
  발생 시 에러 상태 저장, `src/components/shared/ErrorBanner.tsx` 간단한 오류 배너
  컴포넌트 추가 (toast 또는 상단 배너 형태)

**Checkpoint**: US1 + US2 독립 동작 — 실제 클라우드 저장 및 사용자 데이터 격리 완성.

---

## Phase 5: User Story 3 — 로그아웃한다 (Priority: P2)

**Goal**: SideNav에 로그아웃 버튼 추가, 클릭 시 세션 종료 및 로그인 화면으로 이동.

**Independent Test**: 로그인 후 SideNav 로그아웃 버튼 클릭 → 로그인 화면 이동 →
메인 URL 직접 접근해도 로그인 화면으로 리다이렉트됨.

### Implementation for User Story 3

- [ ] T020 [US3] 로그아웃 버튼 — `src/components/shared/SideNav.tsx` footer 영역에
  로그아웃 버튼 추가 (`aria-label="로그아웃"`, 기존 테마 토글 위 또는 아래 배치);
  클릭 시 `supabase.auth.signOut()` 호출 후 `router.push('/')` 리다이렉트
- [ ] T021 [US3] 사용자 정보 표시 — `src/components/shared/SideNav.tsx` 에
  로그인한 사용자의 표시 이름 또는 이메일을 상단 프로필 영역에 표시
  (`useAuth` 또는 컨텍스트에서 user 객체 읽기)

**Checkpoint**: US3 독립 동작 — 로그아웃 및 세션 보안 완성.

---

## Phase 6: User Story 4 — 공개 URL로 앱에 접근한다 (Priority: P2)

**Goal**: Vercel에 배포하여 공개 URL로 앱 접근 가능. OAuth 리다이렉트 URL 업데이트.

**Independent Test**: 배포 URL에서 로그인 → 4개 화면 모두 정상 동작.

### Implementation for User Story 4

- [ ] T022 [US4] Vercel 프로젝트 연결 — Vercel 대시보드에서 GitHub 저장소 연결 또는
  `vercel` CLI로 프로젝트 초기화; `next.config.ts` 는 기존 그대로 사용
- [ ] T023 [US4] Vercel 환경 변수 설정 — Vercel 대시보드 Settings → Environment
  Variables에 `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY` 추가
- [ ] T024 [US4] OAuth 리다이렉트 URL 업데이트 — Supabase Auth 대시보드 및 Google
  Cloud Console · GitHub OAuth App에 Vercel 배포 URL (`https://<project>.vercel.app`)
  과 콜백 URL (`https://<project>.vercel.app/auth/callback`) 추가
- [ ] T025 [US4] 첫 배포 및 검증 — `git push` 또는 Vercel CLI로 배포 후 공개 URL에서
  Google/GitHub 로그인, 할 일 CRUD, 4개 화면 탐색 수동 검증

**Checkpoint**: US4 동작 — 공개 URL 배포 완성.

---

## Phase 7: User Story 5 — 다기기 데이터 접근 (Priority: P3)

**Goal**: 다른 기기에서 같은 계정 로그인 시 페이지 로드 후 최신 데이터 표시.
(실시간 동기화 없음 — 새로고침 필요)

**Independent Test**: 기기 A에서 할 일 추가 → 기기 B에서 동일 계정 로그인 후
페이지 로드 → 추가한 할 일 표시됨.

### Implementation for User Story 5

- [ ] T026 [US5] 다기기 접근 수동 검증 — US2(T016)의 `fetchTasks()` 가 마운트 시
  서버에서 최신 데이터를 가져오는지 두 기기(또는 두 브라우저)로 검증;
  추가 코드 변경 없이 US2 구현으로 달성됨을 확인

**Checkpoint**: US5 동작 — 다기기 데이터 접근 확인 완성.

---

## Phase 8: Polish & Cross-Cutting Concerns

**Purpose**: 전체 마이그레이션 검증, 접근성, 시각 충실도

- [ ] T027 [P] 로딩 상태 UX — `src/context/TasksProvider.tsx` 의 API 호출 중 로딩
  상태(isLoading)를 `src/components/main/TaskList.tsx` 에서 스켈레톤 또는
  스피너로 표시 (기존 UI 최소 변경)
- [ ] T028 [P] 환경 변수 타입 안전성 — `src/lib/supabase/client.ts` 에 환경 변수
  누락 시 명확한 오류 메시지 출력 (빌드 시점 체크)
- [ ] T029 전체 게이트 — `npm run typecheck` · `npm run build` 모두 통과 확인;
  기존 Vitest 단위 테스트(`npm test`) 도 통과 (TasksProvider mock 업데이트 필요 시)
- [ ] T030 [P] 접근성 패스 — 새로 추가된 로그아웃 버튼·사용자 정보·에러 배너의
  aria-label·시맨틱 요소 확인 (spec FR-014 기준)
- [ ] T031 [P] 시각 충실도 패스 — 4개 화면을 라이트·다크에서 기존 디자인과 대조;
  SideNav 사용자 정보·로그아웃 버튼이 기존 레이아웃과 어울리는지 확인

---

## Dependencies & Execution Order

### Phase Dependencies

- **Setup (Phase 1)**: 의존성 없음 — 즉시 시작.
- **Foundational (Phase 2)**: Setup(T001~T003) 완료 후 — 모든 user story BLOCK.
- **US1 (Phase 3)**: Foundational 완료 후 시작.
- **US2 (Phase 4)**: Foundational 완료 후 시작 — US1과 병렬 가능(다른 파일).
  실용적으론 US1 로그인 후 테스트하므로 US1 이후 권장.
- **US3 (Phase 5)**: Foundational + US2 완료 후 (로그아웃은 데이터 저장 완성 후 의미 있음).
- **US4 (Phase 6)**: US1 + US2 + US3 완료 후 배포.
- **US5 (Phase 7)**: US2 완료 후 (fetchTasks 구현 전제).
- **Polish (Phase 8)**: 원하는 모든 user story 완료 후.

### User Story Dependencies

- **US1 (P1)**: Foundational만 의존. MVP 시작점.
- **US2 (P1)**: Foundational만 의존. US1과 병렬 가능.
- **US3 (P2)**: US2에 의존 (데이터 컨텍스트에서 로그아웃 배선).
- **US4 (P2)**: US1+US2+US3 의존 (완전한 앱을 배포).
- **US5 (P3)**: US2 의존 (fetchTasks 구현 전제).

### Parallel Opportunities

- Phase 1: T002·T003 병렬 가능 (T001 이후).
- Phase 2: T004·T005·T006 병렬 가능; T007·T008은 순차 (스키마 → RLS).
- Phase 4: T015·T018 병렬 가능.
- Phase 8: T027·T028·T030·T031 병렬 가능.

---

## Implementation Strategy

### MVP First (US1 + US2 Only)

1. Phase 1: Setup 완료 (Supabase 프로젝트·패키지·OAuth 앱 등록)
2. Phase 2: Foundational 완료 (클라이언트·스키마·RLS·콜백·미들웨어)
3. Phase 3: US1 완료 (실제 Google/GitHub 로그인)
4. Phase 4: US2 완료 (클라우드 데이터 저장)
5. **STOP & VALIDATE**: 로그인 → 할 일 CRUD → 브라우저 닫기 → 재접속 후 데이터 유지 확인

### Incremental Delivery

1. Setup + Foundational → 인프라 준비
2. US1 → 실제 로그인 동작 → 검증
3. US2 → 클라우드 저장 → 검증 (새로고침 후 유지)
4. US3 → 로그아웃 → 검증
5. US4 → Vercel 배포 → 공개 URL 검증
6. US5 → 다기기 접근 확인
7. Phase 8: 전체 게이트·접근성·시각 충실도

---

## Notes

- `[P]` = 다른 파일, 의존성 없음.
- `[Story]` 라벨로 작업↔user story 추적.
- OAuth 설정(T001, T003, T013, T024)은 외부 대시보드 작업 포함 — 시간 여유 확보.
- Supabase CLI 선택 사항: 로컬 개발은 `npx supabase start` 로 로컬 Supabase 실행 가능.
  이 tasks는 대시보드 방식 기준으로 작성됨.
- Vercel 배포(T022~T025)는 GitHub 저장소 연결이 필요 — 저장소가 공개/비공개인지 확인.
