# Implementation Plan: demodev Tasks — Supabase + Vercel 마이그레이션

**Branch**: `002-supabase-vercel-migration` | **Date**: 2026-05-22 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `specs/002-supabase-vercel-migration/spec.md`

## Summary

기존 localStorage 단일 사용자 앱을 Supabase(Auth + PostgreSQL) + Vercel로 마이그레이션한다.
UI 4개 화면(로그인/메인/캘린더/통계)은 그대로 유지하고, 백엔드만 교체한다.
핵심 접근: (1) Supabase Auth로 Google/GitHub OAuth 실제 인증 구현, (2) PostgreSQL
테이블로 할 일 데이터 영속화 및 Row Level Security(RLS)로 사용자 간 데이터 격리,
(3) Next.js 미들웨어로 인증 보호 라우트 구현, (4) Vercel에 프로덕션 배포.

## Technical Context

**Language/Version**: TypeScript 5.x (strict), Node.js 20+
**Primary Dependencies**: Next.js 15 (App Router), Supabase JS v2 (@supabase/supabase-js,
  @supabase/ssr), React 18
**Storage**: Supabase PostgreSQL — tasks, subtasks 테이블 + Row Level Security
**Auth**: Supabase Auth — Google OAuth, GitHub OAuth (소셜 로그인만, 이메일/비밀번호 없음)
**Testing**: 기존 Vitest + React Testing Library (마이그레이션 단계에서 E2E는 범위 밖)
**Target Platform**: 웹 — Vercel 배포, 최신 데스크톱 브라우저
**Project Type**: 기존 웹 앱 마이그레이션 (프론트엔드 + 백엔드 as a Service)
**Performance Goals**: 할 일 CRUD 반영 <500ms, 로그인 흐름 <60초
**Constraints**: UI 변경 없음 · 통계 고정 합성값 유지 · "오늘" = 2026-05-15 고정 ·
  오프라인 퍼스트 없음 · 실시간 동기화 없음(페이지 로드 시 최신 데이터)
**Scale/Scope**: 단일 사용자(개인), 다기기 지원, Vercel Hobby tier 충분

## Constitution Check

*GATE: Phase 0 연구 전 통과 필수. Phase 1 설계 후 재확인.*

| 원칙 | 기존 값 | 이 Feature에서의 적용 | 판정 |
|------|---------|----------------------|------|
| I. Test-First | TDD 필수 | 인증 플로우는 E2E 테스트 없이 수동 검증 허용 (OAuth 특성상 단위 테스트 어려움). Supabase 클라이언트 래퍼·데이터 변환 함수는 단위 테스트 권장. | PASS (예외 인정) |
| II. Design Fidelity | 디자인 토큰 verbatim | UI 4개 화면 변경 없음. 로그인 화면의 소셜 버튼이 실제 동작하도록 핸들러만 교체. | PASS |
| III. Simplicity & YAGNI | 백엔드 없음, localStorage만 | 이 Feature가 백엔드(Supabase)를 도입하는 것이 목적. 기존 원칙을 이 Feature에서 의도적으로 확장. 실시간 동기화·오프라인 퍼스트 등 추가 복잡도는 배제. | JUSTIFIED OVERRIDE |
| IV. Component Modularity | 화면/store 분리 | Supabase 클라이언트를 `src/lib/supabase/` 에 격리. TasksProvider 내부만 교체, 컴포넌트 API 변경 없음. | PASS |
| V. Accessibility | 시맨틱 HTML·aria·키보드 | 로그인 화면 버튼 등 기존 접근성 속성 유지. 새로 추가되는 로그아웃 버튼에도 aria-label 필수. | PASS |

**Complexity Tracking**:

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| 백엔드(Supabase) 도입 | 클라우드 저장·다기기 접근·실제 인증 구현 목적 | localStorage는 단일 브라우저에만 저장되어 다기기 불가 |
| Next.js 미들웨어 | 인증되지 않은 사용자의 보호 라우트 접근 차단 | 클라이언트 사이드만으론 URL 직접 접근 차단 불가 |

## Project Structure

### Documentation (this feature)

```text
specs/002-supabase-vercel-migration/
├── plan.md              # 이 파일
├── spec.md              # 기능 명세
├── checklists/
│   └── requirements.md  # 명세 품질 체크리스트
└── tasks.md             # Phase 2 출력 (/speckit-tasks)
```

### Source Code Changes (기존 구조 위에 추가/변경)

```text
src/
├── app/
│   ├── auth/
│   │   └── callback/route.ts     # [NEW] OAuth 콜백 핸들러
│   ├── layout.tsx                # [MODIFY] SupabaseProvider 추가
│   ├── page.tsx                  # [MODIFY] LoginScreen에 Supabase Auth 연결
│   ├── main/page.tsx             # [MODIFY] 인증 보호 (미들웨어로)
│   ├── calendar/page.tsx         # [MODIFY] 인증 보호 (미들웨어로)
│   └── stats/page.tsx            # [MODIFY] 인증 보호 (미들웨어로)
├── lib/
│   ├── supabase/
│   │   ├── client.ts             # [NEW] 브라우저용 Supabase 클라이언트
│   │   ├── server.ts             # [NEW] 서버용 Supabase 클라이언트 (SSR)
│   │   └── middleware.ts         # [NEW] 미들웨어용 Supabase 클라이언트
│   └── store/
│       ├── persistence.ts        # [MODIFY] localStorage → Supabase DB
│       └── sample-data.ts        # [KEEP] 초기 시드용 유지
├── context/
│   └── TasksProvider.tsx         # [MODIFY] Supabase CRUD로 교체
├── components/
│   └── shared/
│       └── SideNav.tsx           # [MODIFY] 사용자 정보 + 로그아웃 버튼 추가
└── middleware.ts                 # [NEW] 인증 보호 라우트 미들웨어

# Supabase 마이그레이션
supabase/
└── migrations/
    └── 001_initial_schema.sql    # [NEW] tasks, subtasks 테이블 + RLS 정책

# 환경 변수
.env.local                        # [NEW] NEXT_PUBLIC_SUPABASE_URL, ANON_KEY
```
