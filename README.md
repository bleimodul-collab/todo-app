# demodev Tasks — 한국어 Todo 웹앱

> **강의 실습용 리포지토리**

## 수강생 안내

### 시작하기

```bash
git clone https://github.com/jocoding-ax-partners/ax-academy-1.git
cd ax-academy-1
npm install
npm run dev   # http://localhost:3000
```

### 브랜치 구조

| 브랜치 / 태그 | 내용 |
|---|---|
| `master` | 수강생 시작점 (이 코드) |
| `v0-starter` | master 동결 태그 — 언제든 되돌릴 수 있는 기준점 |
| `lectures` | 강의 진행 브랜치 — 강사가 단계별로 커밋 추가 |
| `lecture-N` | N강 종료 시점 태그 |

특정 강의 결과물을 보고 싶을 때:

```bash
git fetch --all --tags
git checkout lecture-3   # 예: 3강 종료 상태
```

처음으로 되돌아오고 싶을 때:

```bash
git checkout master
```

## 기술 스택

- **Next.js 15** App Router + TypeScript (strict)
- **React 18** + Context + useReducer
- **Vitest** + React Testing Library
- **localStorage** 영속화 (백엔드 없음)

## 화면 구성

| 경로 | 화면 |
|---|---|
| `/` | 로그인 |
| `/main` | 메인 (할 일 목록 + 상세 패널) |
| `/calendar` | 캘린더 |
| `/stats` | 통계 |

## 개발 명령어

```bash
npm run dev        # 개발 서버
npm test           # 전체 테스트 (122개)
npm run typecheck  # 타입 체크
npm run build      # 프로덕션 빌드
```
