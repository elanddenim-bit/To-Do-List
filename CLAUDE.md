# to-do-list — 데일리 업무 체크 (출근·퇴근 TO-DO / 일정 / 기록, Supabase 동기화)

## 작업 규칙 (Claude)
- 소유자에게 질문하지 말고 가장 합리적인 해석으로 바로 수정 → main에 커밋·푸시 → 변경 요약만 짧게 보고.
- 예외(먼저 확인): KV/D1/R2 데이터 삭제·스키마 파괴적 변경, 도메인·Worker 이름 변경, 시크릿 추가 필요, 비용 발생. 이 앱에서는 `WORKSPACE` 값·Supabase 프로젝트 변경, `DB` 구조 파괴적 변경(필드 이름 변경·삭제)도 먼저 확인 — 소유자의 실제 업무 기록이 들어 있음.
- 푸시 = 자동 배포. 문법 오류는 곧 서비스 장애이므로 푸시 전 반드시 검증(아래 "검증").
- 단일 파일 구조 유지. 프레임워크·빌드 도구·npm 의존성 추가 금지.
- 커밋 메시지는 한국어 한 줄.
- 스크립트는 ES5 스타일(`var`, `function`, 전역 `window.actXxx` 핸들러 + 인라인 `onclick`)로 작성돼 있음 — 같은 스타일 유지.
- 저장 로직(`loadAll`/`saveAll`/`LOADED` 가드)을 바꿀 때는 "서버를 못 읽은 상태에서 빈 DB 로 덮어쓰기" 가 절대 발생하지 않게 유지.

## 개요
- 지사장 개인 일일 업무 관리 PWA. 탭 4개: 출근(`tab-in`, 오늘 TO-DO·진행 현황) / 퇴근(`tab-out`, 오늘 업무 결과·완료 멘트·익일 TO-DO·오늘 메모) / 일정(`tab-sched`, 일정 등록·월 캘린더) / 기록(`tab-history`, 주간 요약·지난 기록·백업/이전).
- 기능: 우선순위(high/mid/low)·시간 지정, 완료 멘트(doneNote), 날짜 이동, 익일 이월(개별/전체), 반복 업무(매주 요일/매월 일자 자동 등록), 자연어 일정 파싱("9/24~27 충칭 출장"), 구글 캘린더 링크·ICS 내보내기(`업무일정.ics`), 주간 요약 복사, JSON 텍스트 내보내기/가져오기(병합).

## 배포
- 방식: 정적 파일. wrangler 설정 없음 → Cloudflare Pages(GitHub `elanddenim-bit/to-do-list` main) 정적 배포로 추정. 미확인.
- URL/도메인: 미확인.
- 바인딩: 없음. 서비스워커 없음.
- 외부 서비스: Supabase `SUPA_URL=https://emoblnrvyqezitvymwbr.supabase.co`, `SUPA_KEY=sb_publishable_…`(공개키) — index.html 상수. `WORKSPACE`(데이터 행 id = 비밀 코드)는 2026-10-09부터 코드에 없음: 기기마다 처음 한 번 입력 → localStorage `todo.ws`.
- 시크릿: 없음. GitHub 저장소는 private(2026-10-09 전환). 공개키는 vendor-directory와 같은 Supabase 프로젝트·같은 `workdata` 테이블을 씀.

## 파일 구조
- `index.html` — 전체 앱(CSS, HTML, `<script>` 204행~). `LOGO_WHITE` base64 로고.
- `manifest.json` — "데일리 업무 체크"/"업무체크", 테마 #1A2A4A.
- `favicon.png`, `apple-touch-icon.png`, `icon-192.png`, `icon-512.png`.

## 아키텍처 / API
- 앱 로드: `loadAll()` → `GET {SUPA_URL}/rest/v1/workdata?id=eq.<WORKSPACE>&select=data`. 서버가 비어 있으면 `migrateFromClaude()`(Claude 아티팩트 `window.storage` 키 `wapp`)로 자동 이전.
- 저장: `saveAll()` → `POST /rest/v1/workdata` (Prefer: resolution=merge-duplicates) body `[{id:WORKSPACE, data:DB, updated_at}]`. 전체 DB 를 통째로 upsert.
- `LOADED=false`(서버 미확인) 상태에서 저장 요청 시 먼저 재로드·병합, 그래도 실패하면 저장 보류 + 배너.
- 요일별 순환 백업 `snapshotOnce()`: 세션당 1회 `id=<WORKSPACE>-bak-<0..6>` 행에 upsert(최근 7일 복구용).
- `configured()`: URL https, KEY 30자 초과, WORKSPACE 10자 이상일 때만 클라우드 사용. 코드가 없으면 `boot()`가 코드 입력 화면(`showWsGate`)을 띄움.
- localStorage: `todo.ws`(공간 코드) 하나만. 모든 요청에 헤더 `x-ws: <코드>`를 붙임 → Supabase RLS가 `id = x-ws` 또는 `id LIKE x-ws||'-bak-%'` 행만 허용(RLS 적용은 대시보드에서, 저장소에 SQL 없음).
- 공간 코드 관리(기록 탭): 입력 시 서버에 그 id가 있어야 통과(없는 코드로 빈 공간 생성 방지). '새 코드로 바꾸기' = 새 id에 DB 저장·확인 → 옛 행은 데이터 유지 + `movedAt` 표시(옛 코드를 기억한 기기는 코드 입력 화면으로) → 새 코드 표시. '이 기기에서 잊기'.
- 렌더: `render()` 가 `S.mode` 에 따라 `#view` innerHTML 재생성.

## 데이터
- Supabase 테이블 `workdata(id text PK, data jsonb, updated_at)`. 생성 SQL 은 저장소에 없음.
- `DB = {days:{"YYYY-MM-DD":{todos:[…], note:""}}, sched:[{id,title,start,end}], recur:[{id,text,priority,type:'w'|'m',n}], recurSeed:{"<recurId>:<date>":1}}`.
- todo: `{id, text, priority:'high'|'mid'|'low', done, time?:"HH:MM", doneNote?, recur?:true, src?:'brief'}`. `src:'brief'`는 요약 붙여넣기로 추가된 항목. 정렬 `ORDER={high:0,mid:1,low:2}` 후 시간순.
- 반복 업무 `seedRecur()`: 오늘·내일 날짜에 매칭되면 1회만 TO-DO 추가(`recurSeed` 로 중복 방지). `type:'w'` 는 `n`=요일(0=일), `'m'` 은 `n`=일자.
- 가져오기 병합 규칙: 없는 날짜 추가, 있는 날짜는 text 중복 아닌 todo 만 추가, note 는 비었을 때만, sched 는 title+start+end 중복 제외.
- 세션 상태 `S = {mode:'in'|'out'|'sched'|'history', tKey, tmKey, today, tomorrow, sched, index, viewDay, weekOff, calY, calM, moveFor, timeFor, noteFor, …}`.

## 요약 붙여넣기 (2026-10-08)
- 출근 탭 "메일·위챗 요약 붙여넣기" 카드: 아침 메일 업무표(🔴→high, 🟡→mid, ⚪·제외 무시)나 기업위챗 요약(①③→high, ②→mid, ④ 무시)을 붙여넣으면 `parseBrief()`가 후보를 만들고, 체크한 항목만 오늘 TO-DO에 추가. 앞에 [메일]/[위챗] 표시, 이미 있는 항목(앞 40자 공백 제거 비교)은 기본 해제.
- 메일 업무표는 별도 예약 작업이 사용자 PC의 Outlook 내보내기 파일로 매일 06:50에 만든다(이 저장소 밖).

## 도메인 규칙
- 하루 흐름: 출근 시 오늘 TO-DO 확인 → 퇴근 시 결과 체크·완료 멘트·익일 TO-DO 작성·미완료 이월.
- 날짜 키는 로컬 시간 `YYYY-MM-DD`(fmtKey). 요일 표기 `WD=["일","월",…,"토"]`.
- 구글 종일 일정 종료일은 배타(+1일). 자연어 날짜 파서는 연도 생략 시 올해, 범위 `~`, `-`, `–` 지원.

## UI 규칙
- E·LAND CI: `--red:#D51030`, `--navy:#1A2A4A`, `--grey:#969BA5`, `--hair:#E1E4EA`, `--bg:#F6F7F9`. 네이비 헤더 + 흰 로고 + 레드 라인, 카드형 섹션(`.card`, 강조 헤더 `.card-h.accent`).
- 폰트 Noto Sans KR / Malgun Gothic / Apple SD Gothic Neo, 본문 15px, 최대폭 720px.
- 한국어 UI. 푸터 "E·LAND GUANGZHOU · 宇旭贸易（上海）有限公司 广州深圳分公司".

## 주의사항 / 알려진 이슈
- 2026-10-09 확인: RLS가 없어 공개키만으로 `workdata` 전체 id 목록·데이터 조회 가능했음(vendor-directory 행 포함). 헤더 기반 RLS 적용 전까지는 코드를 숨겨도 목록 조회로 노출됨.
- 저장이 전체 DB 통째 upsert(last write wins) — 두 기기 동시 편집 시 한쪽 변경 유실 가능.
- 서비스워커 없음 → 오프라인에서 새로 열 수 없음. 서버 미연결 시 입력은 세션 동안만 유지.
- 중국 본토에서 `*.supabase.co` 접속 가능 여부 미확인.

## 검증
```bash
# 저장소 루트에서 실행
node -e "const h=require('fs').readFileSync('index.html','utf8');let i=0;for(const m of h.matchAll(/<script(?![^>]*src=)[^>]*>([\s\S]*?)<\/script>/g)){i++;new (require('vm').Script)(m[1])}console.log('script OK',i)"
node -e "JSON.parse(require('fs').readFileSync('manifest.json','utf8'));console.log('manifest OK')"
# 인라인 onclick 이 참조하는 act* 함수가 모두 정의돼 있는지
node -e "const h=require('fs').readFileSync('index.html','utf8');const def=new Set([...h.matchAll(/window\.(act\w+)\s*=/g)].map(m=>m[1]));const used=new Set([...h.matchAll(/\b(act[A-Z]\w*)\(/g)].map(m=>m[1]));const miss=[...used].filter(x=>!def.has(x));console.log(miss.length?'MISSING '+miss:'handlers OK')"
```
