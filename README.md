# 🧭 1:1 자기관리 코치

할 일을 정해 놓고도 제대로 못 하는 사람이 많습니다.
이 앱은 **사람마다 1:1 코치처럼** 기상부터 취침까지 하루 루틴(식사, 약, 공부, 운동, 취침)을 챙겨 주는 투두 앱입니다.

- 처음에 **기상/취침 시간**만 입력하면 코치가 **기본 루틴**을 만들어 주고, 사용자는 고치기만 합니다.
- 시간이 지났는데 체크가 없으면 코치가 말을 겁니다. (예: "💊 '아침 약' 시간이 10분 지났어요!")
- 맨 위 **"지금 할 일" 카드**에 남은 시간과 진행 막대, 다음 일정을 보여 줍니다.
- 식사는 **먹은 음식**, 운동은 **몇 분 했는지** 기록합니다. (다이어트 기록)
- 앱이 열려 있으면 루틴 시간에 **종소리 + 배너**로 알려 줍니다.

자세한 요구사항은 [PRD.md](PRD.md), 표 설계는 [DB.md](DB.md), 배포 준비물은 [DEPLOY.md](DEPLOY.md)를 보세요.

## 진행 상황

| 단계 | 내용 | 상태 |
|---|---|---|
| 길 A | 로컬 SQLite(`todo.db`)에 저장, CRUD, 코치, 화면 | ✅ 완료 |
| 길 B | Supabase로 저장소 전환 + 이메일 로그인 + RLS | ✅ 완료 (구글 로그인은 예정) |
| 배포 | Render 배포 준비물 정리 | ✅ [DEPLOY.md](DEPLOY.md) (실제 배포는 선택) |

## 실행 방법

```bash
npm install
cp .env.example .env.local   # 값 채우기 (아래)
npm run dev                  # http://localhost:3000
```

`.env.local`의 `DB_PROVIDER`로 저장소를 고릅니다. **화면 코드는 그대로이고 저장소만 바뀝니다.**

| `DB_PROVIDER` | 저장 위치 | 로그인 | 준비 |
|---|---|---|---|
| `sqlite` (길 A) | 내 컴퓨터의 `todo.db` | 없음 | 없음. 처음 실행하면 표가 자동으로 만들어진다 |
| `supabase` (길 B) | Supabase PostgreSQL | 이메일 + 비밀번호 | ① Supabase 프로젝트 생성 ② SQL Editor에서 [supabase/schema.sql](supabase/schema.sql) 실행 ③ `SUPABASE_URL`, `SUPABASE_PUBLISHABLE_KEY` 입력 |

- 개발 중에는 터미널에 실행되는 SQL이 출력됩니다(길 A). 배포 환경에서는 사용자 기록이 로그에 남지 않도록 꺼집니다.
- `npm run dev`: **이 컴퓨터에서만** 접속됩니다 (`127.0.0.1`). 로그인이 없는 길 A의 데이터를 같은 와이파이의 다른 사람이 보거나 바꾸지 못하게 막아 둔 것입니다.
- `npm run dev:phone`: 같은 와이파이의 폰에서 확인할 때만 씁니다. 믿을 수 있는 네트워크에서만 켜세요.
- `npm run test:rls`: 두 계정으로 RLS를 확인합니다 (아래 "확인 기록 (길 B)").

## 기술

| 역할 | 기술 |
|---|---|
| 프레임워크 | Next.js 16 (App Router, Server Actions, proxy) + TypeScript |
| 스타일 | CSS Modules, CSS 변수로 다크모드 (`prefers-color-scheme`) |
| DB (길 A) | SQLite 파일 + Drizzle ORM + libsql 드라이버 |
| DB (길 B) | Supabase(PostgreSQL) + supabase-js + `@supabase/ssr`, RLS |
| 로그인 (길 B) | Supabase Auth (이메일 + 비밀번호), JWT를 쿠키에 저장 |

## 구조와 설계 원칙

```
src/
├─ proxy.ts               (길 B) 모든 요청의 문지기: JWT 갱신, 로그인 안 했으면 /login으로
├─ app/
│  ├─ page.tsx            오늘 화면 (서버에서 DB 읽기)
│  ├─ routines/page.tsx   루틴 관리 화면
│  ├─ login/              (길 B) 로그인/회원가입/로그아웃
│  └─ actions.ts          서버 액션 = 백엔드 API (입력 검사 후 저장소 호출)
├─ components/            화면 부품 (브라우저에서 실행)
├─ lib/
│  ├─ repo/               저장소: DB에 접근하는 유일한 곳 (server-only)
│  │  ├─ types.ts         저장소 약속(interface)
│  │  ├─ sqlite.ts        길 A 구현 (Drizzle)
│  │  ├─ supabase.ts      길 B 구현 (supabase-js, 사용자 JWT로 접속)
│  │  └─ index.ts         DB_PROVIDER로 저장소 선택, 길 B는 로그인한 사용자 확인
│  ├─ supabase/           (길 B) 서버용 Supabase 클라이언트, 설정값
│  ├─ coach.ts            규칙 기반 코치 (순수 함수, 나중에 AI 코치로 교체 가능)
│  ├─ defaultRoutines.ts  기상/취침 시간 → 기본 루틴
│  ├─ dayItems.ts         시간표, "지금 할 일", 알람 대상 계산
│  ├─ chime.ts            알람 종소리 (Web Audio, 소리 파일 없음)
│  └─ time.ts             날짜/시간 계산 (한국 시간 기준)
└─ db/                    길 A 표 정의와 생성 SQL
supabase/schema.sql       길 B 표 + 권한 + RLS 정책
scripts/rls-test.mjs      길 B RLS 확인 스크립트
```

- **화면은 백엔드에만, DB에는 백엔드만**: 화면 컴포넌트는 서버 액션만 부르고, DB 코드는 `server-only`로 막아 브라우저 번들에 들어가지 못합니다. Supabase 키도 `NEXT_PUBLIC_` 없이 서버에서만 읽습니다.
- **내 것만 (이중 방어)**: 저장소는 만들 때 주인(owner)이 정해지고, 모든 쿼리에 `user_id = owner` 조건이 붙습니다. 길 B에서는 여기에 더해 **DB의 RLS**가 JWT의 사용자 id(`auth.uid()`)로 한 번 더 막습니다. 코드에서 조건을 빠뜨려도 남의 행은 돌아오지 않습니다.
- **관리자 키를 쓰지 않음**: 길 B는 공개용(publishable) 키 + 로그인한 사용자의 JWT로만 접속합니다. RLS를 건너뛰는 secret(service_role) 키는 앱 어디에도 없습니다.
- **로그인 확인은 서버에서**: `getUser()`가 JWT를 Supabase Auth 서버에 다시 확인하므로 위조된 쿠키를 믿지 않습니다. 비밀번호는 우리 DB가 아니라 Supabase Auth가 해시로 보관합니다.
- **입력은 서버에서 다시 검사**: 브라우저에서 온 값은 조작될 수 있으므로 `actions.ts`에서 형식, 길이, 시간 순서를 검사합니다. DB에도 CHECK/UNIQUE 제약을 걸어 두 번 막습니다.
- **SQL 주입 방지**: Drizzle과 supabase-js가 값을 SQL 문장과 따로 전달합니다(파라미터화).
- **비밀값**: `.env*`와 `todo.db`는 `.gitignore`에 있어 올라가지 않습니다.
- **XSS**: 사용자가 쓴 글은 React가 글자로만 그려서 코드로 실행되지 않습니다 (`dangerouslySetInnerHTML` 사용 안 함).
- **로그**: SQL 로그(값 포함)는 개발 환경에서만 켭니다.
- **네트워크**: 로그인이 없는 길 A의 개발 서버는 `127.0.0.1`에서만 열립니다.

## 확인 기록 (길 A)

2026-10-29, 로컬 개발 서버에서 직접 확인했습니다.

- [x] 기상 07:00 / 취침 23:00 입력 → 기본 루틴 8개 생성
- [x] 운동 루틴 체크(40분, 걷기) → "이번 주 운동 총 40분" 코치 메시지
- [x] 할 일 추가 → 완료 → 삭제
- [x] 끝 시간이 시작보다 빠르면 "끝 시간은 시작 시간보다 늦어야 해요." 오류
- [x] 루틴 수정(시간, 요일 월~금) → 토요일 화면에서 빠짐, 미래 날짜는 체크 불가
- [x] **개발 서버를 껐다 켠 뒤에도** 체크 기록과 수정한 루틴이 그대로 남음
- [x] 폰 화면 크기(375px) + 다크모드 확인
- [x] 루틴 알람: 소리 알림 켜기 → 2분 뒤 시작하는 테스트 루틴을 만들고 기다림 → 그 분 정각에 "⏰ 지금 '알람 테스트' 시간이에요!" 배너 표시 → 확인으로 닫힘 (테스트 루틴은 삭제)

## 확인 기록 (길 B: Supabase + RLS)

2026-10-29, Supabase 프로젝트에 [supabase/schema.sql](supabase/schema.sql)을 실행하고 `DB_PROVIDER=supabase`로 확인했습니다.

- [x] 로그인하지 않고 `/`에 들어가면 `/login`으로 보내짐
- [x] 계정 A, B를 각각 회원가입 → 각자 처음 시작 화면(기상/취침)에서 기본 루틴 생성
- [x] 계정 B로 로그인하면 A의 루틴과 기록이 보이지 않음 (B는 빈 상태에서 시작)
- [x] Supabase Table Editor에서 같은 `routines` 표에 `user_id`가 두 종류(A, B)로 섞여 저장된 것 확인
- [x] 할 일 추가 → 완료 → 삭제, 새로고침(F5) 후에도 루틴이 남는 것 확인
- [x] `npm run test:rls` (앱 화면을 거치지 않고 Supabase에 직접 요청) 결과:

| 확인 | 결과 | 내용 |
|---|---|---|
| B가 A의 할 일 조회 | ✅ 통과 | 돌려받은 행 0개 |
| B가 A의 할 일 수정 | ✅ 통과 | 수정된 행 0개, A가 다시 본 제목 그대로 |
| B가 A의 할 일 삭제 | ✅ 통과 | 삭제된 행 0개, A의 할 일 남아 있음 |
| B가 A 이름(user_id)으로 할 일 만들기 | ✅ 통과 | 거절됨 (42501, RLS 위반) |
| B가 A의 루틴에 체크 만들기 | ✅ 통과 | 거절됨 (42501, RLS 위반) |
| 로그인 없이 조회 | ✅ 통과 | 거절됨 (42501, 권한 없음) |
| A가 자기 할 일 조회 | ✅ 통과 | 돌려받은 행 1개 (정상 동작) |

## 스크린샷

### 길 A (로컬 SQLite)
| 개발 서버를 여러 번 껐다 켠 뒤에도 남아 있는 운동 기록(40분 · 걷기) |
|---|
| ![로컬 재시작 후](docs/screenshots/a1-local-after-restart.jpg) |

### 길 B (Supabase)
| 할 일 추가 | 완료 | 삭제 |
|---|---|---|
| ![추가](docs/screenshots/b1-supabase-add.png) | ![완료](docs/screenshots/b3-supabase-complete.png) | ![삭제](docs/screenshots/b3-supabase-delete.png) |

| 새로고침(F5) 후에도 남은 루틴 (계정 A, 취침 12:00) | 계정 B로 로그인한 화면 (취침 23:00, A의 루틴이 보이지 않음) |
|---|---|
| ![새로고침 후](docs/screenshots/b2-supabase-after-refresh.png) | ![계정 B](docs/screenshots/b-rls-account-b-view.png) |

**Supabase Table Editor (관리자 화면)**: 한 `routines` 표 안에 `user_id`가 `8c6808bd…`(A)와 `067e5a4e…`(B) 두 종류로 함께 저장되어 있습니다. 관리자 화면(Role postgres)에서는 모두 보이지만, 앱에서는 RLS 때문에 각자 자기 행만 받습니다.

![Table Editor](docs/screenshots/b4-supabase-table-editor.png)

> 테스트 계정은 Supabase의 **Confirm email**(가입 확인 메일)이 켜진 상태에서 메일 링크로 확인을 마친 뒤 사용했습니다. 무료 메일 발송 한도가 작아서, 계정을 많이 만들 때는 개발 중에만 끄는 것을 권장합니다.

## 디버깅 기록

개발하면서 실제로 만난 문제를 **증상 → 원인 → 해결 → 확인** 순서로 정리했습니다. (2026-10-29)

### 1. 첫 화면에서 `new Date()` 오류 (Next.js 16 `cacheComponents`)
- **증상**: 첫 브라우저 테스트 때 화면에 "1 Issue" 배지가 떴고, 콘솔에 오류가 났습니다.
  `Error: Route "/": Next.js encountered the unstable value new Date() while prerendering.` (`src/lib/time.ts`의 `nowInAppTz`)
- **원인**: `cacheComponents`가 켜져 있어서, Next.js가 **현재 시각과 DB를 읽는 페이지를 미리 만들어 두려고** 했습니다. 이 페이지는 요청할 때마다 달라야 합니다.
- **해결**: [page.tsx](src/app/page.tsx)와 [routines/page.tsx](src/app/routines/page.tsx)에서 `await connection()`을 불러 "요청할 때마다 만든다"고 알렸습니다.
- **확인**: 새로고침한 뒤 이슈 배지가 사라졌습니다. → 그런데 바로 2번 오류가 이어졌습니다.

### 2. `connection()`이 `<Suspense>` 밖에 있다는 오류
- **증상**: 서버 터미널에 오류가 났습니다.
  `Error: Route "/": Next.js encountered uncached data during prerendering. fetch(...) or connection() accessed outside of <Suspense> ...`
- **원인**: 1번에서 넣은 `connection()`이 `<Suspense>` 경계 **밖**에 있었습니다.
- **해결**: 페이지를 "껍데기 + 데이터 컴포넌트"로 나눴습니다. 껍데기가 `<Suspense fallback={<Loading />}>`로 데이터 컴포넌트를 감싸고, `connection()`은 그 **안**에서 부릅니다. `src/app/loading.tsx`는 지우고 [Loading.tsx](src/components/Loading.tsx)로 옮겼습니다.
- **추가 시도**: 처음에는 콘솔에 남은 옛 메시지일 수도 있다고 보았습니다. 그래서 새로고침한 뒤 **서버 터미널을 다시 확인**해서, 진짜 오류인지부터 가렸습니다.
- **확인**: `tsc`와 `lint`가 통과했고 `npm run build`가 성공했습니다. `/`와 `/routines`는 ◐(Partial Prerender)로 표시됐습니다.
- 중간에 `Module not found: Can't resolve '@/components/Loading'`이 한 번 떴습니다. `Loading.tsx`를 만들기 직전에 개발 서버가 먼저 컴파일한 것으로 **추정**됩니다. 파일을 만든 뒤 `✓ Compiled`가 나와서 따로 고치지 않았습니다.

### 3. 체크하거나 추가한 뒤 "지금" 시계가 멈춤
- **증상**: 화면의 "지금 18:12" 시계가 새 시각으로 바뀌지 않았습니다.
- **원인**: [DayView.tsx](src/components/day/DayView.tsx)가 `useState(props.initialNowMinutes)`로 **첫 값만** 썼습니다. 그래서 서버가 새 시각을 보내도 반영하지 않았습니다.
- **해결**: `serverNow` 상태를 두고, 서버에서 받은 값이 바뀌면 시계를 다시 맞추게 했습니다.

### 4. 보안: 같은 와이파이의 다른 기기가 앱에 들어올 수 있었음 (직접 점검해서 발견)
- **증상**: 길 A에는 로그인이 없는데, `next dev`가 와이파이 주소(192.168.x.x:3000)로도 열려 있었습니다.
- **해결**: `npm run dev`를 `next dev -H 127.0.0.1`로 바꾸고, 폰에서 확인할 때만 쓰는 `npm run dev:phone`을 따로 만들었습니다. (커밋 111abf0)
- **확인**: `netstat`에서 127.0.0.1만 LISTENING이었습니다. localhost는 200을 돌려줬고, 와이파이 주소는 연결이 거부됐습니다.

### 5. 보안: 터미널 SQL 로그에 개인 기록이 찍힘
- **증상**: Drizzle 로그의 `params`에 식사 메모 같은 값이 그대로 나왔습니다.
- **해결**: [sqlite.ts](src/lib/repo/sqlite.ts)에서 `logger: process.env.NODE_ENV === "development"`로 바꿔, 개발할 때만 로그를 켭니다. (커밋 111abf0)

### 6. `.env.local`에 Secret key를 붙여 넣음 (저장 전에 발견)
- **증상**: `NEXT_PUBLIC_...` 키 자리의 값이 `sb_secret_`으로 시작했습니다.
- **원인**: Supabase API Keys 화면에서 Publishable key 대신 **Secret(관리자) key**를 복사했습니다.
- **해결**: 저장하지 않고 지운 다음 `sb_publishable_` 키를 넣었습니다.
- **확인**: 값을 가리고 읽어서 `sb_publishable_`로 시작하는 것을 확인했습니다. Secret key는 앱에 쓰지 않습니다.

### 7. RLS 테스트 환경변수에 안내문 예시 글자가 그대로 들어감
- **증상**: `RLS_TEST_A_EMAIL=가입한 A 이메일`처럼 예시 문구가 값으로 들어갔고, 빈 줄도 중복됐습니다.
- **원인**: 안내문이 헷갈리게 쓰여 있었습니다.
- **해결**: 중복된 줄을 지우고 실제 값을 다시 넣었습니다.
- **확인**: `npm run test:rls`가 7개 중 7개 통과했습니다. (위 "확인 기록 (길 B)" 표)

### 8. 회원가입하자 확인 메일이 옴 / 취침 시간이 낮 12:00으로 들어감 (버그 아님)
- **확인 메일**: Supabase의 **Confirm email**이 켜져 있었기 때문입니다. 코드는 그대로 두고, 메일 링크로 확인한 뒤 로그인했습니다.
- **취침 12:00**: 시간 입력칸의 "오후 12:00"은 **낮 12시**입니다(자정은 "오전 12:00"). 그래서 점심·운동·저녁 루틴이 빠졌습니다. 루틴 관리에서 고칠 수 있고, 이 차이는 A·B 계정 화면을 구분하는 증거로도 썼습니다.

### 배운 점
- 오류 메시지가 보이면 **옛 메시지인지 지금 메시지인지**부터 확인합니다(새로고침 → 서버 터미널 다시 보기).
- 오류를 하나 고치면 **다음 오류가 이어질 수 있습니다**(1번 → 2번). 고친 뒤에는 `build`까지 돌려 봅니다.
- 키와 환경변수는 **값을 가리고 앞부분만** 확인합니다.
