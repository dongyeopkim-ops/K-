# 리니지M K팀 길드 관리 웹앱 (출석앱)

> 리니지M 길드(K팀) 운영용 웹앱. 출석체크·통계·혈원현황·사다리·경매·출석요청·주요맵.
> 바닐라 JS + Google Apps Script(GAS) + Firebase(Firestore + Hosting). 빌드 과정 없음.

---

## 📁 파일 구성 (이 3개가 전부)

| 파일 | 역할 |
|---|---|
| `index.html` | 프론트엔드 **단일 파일**. 바닐라 JS, Firebase SDK(CDN), 상태객체 `S` + `render()` 재렌더 방식. Firebase 호스팅으로 배포. |
| `Code.gs` | **Google Apps Script** 백엔드. 구글시트 ↔ Firestore 동기화, 각종 관리 함수. GAS 웹앱으로 배포. |
| `sw.js` | 서비스워커(PWA 캐시). **수정 시 `VERSION` 숫자 반드시 올릴 것** (안 올리면 캐시 때문에 변경 반영 안 됨). 현재 `v96`. |

이미지(예배당.jpg, 보라방.jpg, 녹방.jpg, 파랑방.jpg, 노랑방.jpg, 빨간방.jpg)는 이 저장소(`dongyeopkim-ops/K-`)에 있고, index.html이 `raw.githubusercontent.com/dongyeopkim-ops/K-/main/<파일>`로 참조. **파일명/경로 바꾸지 말 것** (바꾸면 URL도 수정 필요).

---

## 🚀 배포 (중요)

1. **프론트(index.html, sw.js)**: `firebase deploy`
   - index.html을 고쳤으면 **반드시 `sw.js`의 `VERSION`을 올리고 같이 배포** (캐시 무효화). 안 그러면 사용자 화면에 반영 안 됨.
2. **백엔드(Code.gs)**: GAS 편집기에 붙여넣고 **"배포 → 새 버전으로 웹앱 재배포"**.
   - ⚠️ 저장만 하면 기존 배포 URL에 반영 안 됨. 반드시 새 버전 재배포.
   - GAS 편집기 "실행" 버튼은 **함수 인자를 못 넘김** → 인자 필요한 함수는 무인자 래퍼 사용(`audit_2026_08`, `syncNow`, `previewDedupe_2026_08` 등).

---

## 🗄️ 데이터 소스 원칙 (가장 중요)

- **구글시트 = 정답(source of truth)**, Firestore는 시트를 따라감.
- 앱은 **출석을 Firestore(`attendance` 컬렉션)에서 실시간(onSnapshot)으로 읽음**. 시트를 직접 안 읽음.
- 그래서 **시트에서 행을 지워도** Firestore가 정리돼야 앱에 반영됨 → `onSheetChange` 트리거/`syncNow`/6시간 자동동기화가 그 역할.

### attendance 문서 ID 규칙 (절대 지킬 것)
- 문서 ID = **원문ID** `날짜_보스_시간_캐릭` (예: `2026-09-08_오만 1부_12:25_채찍`).
- **MD5 ID 쓰지 말 것.** 과거에 프론트(원문ID)와 GAS(MD5ID)가 달라서 **같은 출석이 2벌로 중복 저장**되는 버그 있었음. 지금은 양쪽 다 원문ID로 통일.
- GAS에서 Firestore 경로 만들 때 한글/공백/`:` 때문에 **반드시 `encodeURIComponent`** 할 것 (안 하면 삭제가 조용히 실패함 — 실제로 겪은 버그).

### 통계 집계 키
- `calcStats`/`loadAttByMonth`/저장 모두 키가 `날짜_보스_시간` 로 **정확히 일치**해야 함.
- 보스명·시간은 `SCHEDULE`(index.html)에서 나옴. 출석요청의 보스목록(`getTodayBosses`)도 같은 SCHEDULE 사용 → 키 일치 보장.

---

## 🔑 Firestore 컬렉션

| 컬렉션 | 내용 | 비고 |
|---|---|---|
| `members/{캐릭명}` | 로그인계정: id, password(해시), status, admin, proxy[] | 문서ID=캐릭명. 비번은 `gasHashPw_`(=프론트 `hashPw`와 동일 알고리즘) |
| `attendance/{원문ID}` | date, boss, time, charName | 출석기록. 통계 소스 |
| `attendanceRequests/{id}` | date, boss, time, charName, status, requestedBy, processedBy, processedAt, approvedDate | 출석요청(누락분 신청) |
| `memberInfo/{캐릭명}` | 스펙(서버/Lv/클래스/변신/집행/신화갑옷/메모) | 혈원현황 스펙표시 |
| `auctions/{id}` + `/bids/{id}` | 경매 + 입찰 | 실시간 경매 |
| `ladder/current` | 사다리(phase: recruiting/drawn) | 실시간 사다리 |
| `config/guildMembers` | 혈원현황 캐시 | |

멤버 시트 열: **A=캐릭터명, B=아이디, C=PW, D=권한, E=변경전캐릭터명, F=통계제외**
- 비밀번호는 동기화가 **절대 덮어쓰지 않음**(updateMask로 id/admin/proxy만). C열(PW)에 값 넣고 동기화할 때만 비번 재설정.
- F열에 아무 값이나 있으면 그 멤버는 **통계에서 제외**.

---

## 🧩 기능(탭) 목록

프론트 `render()`가 `S.tab`에 따라 분기:
`attend`(출석) · `guild`(혈원현황) · `myattend`(내출석) · `attreq`(출석요청) · `myinfo`(내정보) · `tower`(신념의탑) · `maps`(주요맵) · `ladder`(사다리) · `auction`(경매) · `stats`(통계·관리자) · `payout`(혈비·관리자) · `settings`

- **출석/사다리/경매/출석요청** 등은 전체 공개, **등록·생성·마감·승인·삭제 같은 조작 버튼만 관리자(`S.user.admin`)**.
- **사다리**: 관리자 모집시작 → 혈원 참가신청(중복방지, 트랜잭션) → 관리자가 결과입력 후 생성 → 빨간선 따라 뽑기 애니메이션.
- **경매**: 실시간 최고가 낙찰, +10/+100/+500, 입찰 히스토리, 상한가(차단/즉시낙찰), 마감60초전 20초연장, **입찰은 `runTransaction`으로 동시성 처리**. 아이템은 앱 폼 + `경매목록` 시트 드롭다운 검색.
- **출석요청**: 당일 진행된 보스만 신청(대리출석 가능 `canProxy`), 관리자 승인 시 `saveAtt`로 시트+FS 기록. 미처리 요청은 날짜 지나도 유지. 관리자용 "오늘 승인 내역"(다음날 자동리셋, `approvedDate` 기준). **시트에서 출석행 삭제 시 승인된 요청 이력도 함께 삭제됨**(`cleanupOrphanRequestsMonth_`).
- **주요맵 > 예배당**: `special:'chapel'`. 제사장 위치 클릭 핫스팟 → 방 사진 모달 + **빨간 선 따라가는 경로 애니메이션**. 경로는 `CHAPEL_HOTSPOTS[].path = [[x,y],...]`(이미지 대비 %)로 지정. 파랑방만 시작점이 가운데 아님.

---

## 🔄 동기화/정비 함수 (Code.gs)

- `syncNow()` — 최근 2개월 시트→Firestore 즉시 동기화(수동). 어긋날 때 제일 먼저 실행.
- `onSheetChange(e)` — **설치형 onChange 트리거**. 출석기록 시트 행 삭제 시 자동으로 FS 정리(attendance + 승인된 요청 이력). Lock으로 연속삭제 코얼레싱. **`installSheetChangeSync()` 한 번만 실행하면 이후 자동**.
- `installAttendanceAutoSync()` — 6시간마다 최근2개월 자동동기화(백업). 한 번만 설치.
- `mirrorAllAttendanceToFirestore()` — 전체 미러링(재개형, `MIRROR_FROM` 이후). `resetAttendanceMirror()`로 진행상황 초기화.
- `mirrorAttendanceMonthToFirestore(년, 월0based)` — 특정 월 시트↔FS 정합(중복/고아 삭제 + 누락 추가 + 요청이력 정리).
- `syncMembersFromSheet()` / `previewMembersSync()` — 멤버 A열 기준 동기화(비번 미변경, 공란 아이디 보존, A열에 없는 승인멤버만 삭제).
- `resetPasswordsTo1234()` — 전원 비번 1234로 초기화(응급).
- `dedupeAttendance_2026_08()` 등 — 시트 중복행 정리.
- `audit_2026_08()` / `debugChar_2026_08()` — 시트 vs FS 불일치 진단.

> 월 인자는 **0-based** (8월=7, 9월=8). `getAttendanceByMonth(year, month0)`.

---

## ⚠️ 겪은 버그 / 주의 (재발 방지)

1. **sw 버전 안 올림** → 변경이 앱에 반영 안 됨. index.html 고치면 sw VERSION 필수 증가.
2. **문서ID 불일치(원문 vs MD5)** → 출석 2배 중복 → 참여율 부풀림. 원문ID로 통일 유지.
3. **URL 인코딩 누락** → 한글/공백/`:` 포함 문서 삭제 실패. `encodeURIComponent` 필수.
4. **GAS 실행버튼 인자 전달 안 됨** → 무인자 래퍼 사용.
5. **전체 render() 남발 → 깜빡임**. 경매는 `paintAuctionDetail`/`paintAuctionPage`로 부분 렌더. 가능하면 영역 단위 갱신.
6. **출석 폴링(60초) 제거** → `onSnapshot` 실시간 구독으로 전환(변경 시에만 갱신).
7. **Firestore 쿼리는 단일 필드**로(복합 색인 회피). 복합조건은 한쪽만 쿼리하고 나머지는 코드에서 필터.
8. **PITR 켜져 있음** — 7일 내 복구 가능. 단, 사고 "이전에" 켜져 있어야 효과.

---

## 🛠 작업 방식 (협업)

- 변경은 **핵심만 최소 수정**, 큰 기능 추가 시엔 전체 파일 교체 가능.
- 코드 수정 후 **문법 검증**(node --check 등) 습관화.
- Firestore 삭제/대량 변경 전엔 **preview/audit 먼저**.
- 응답은 한국어, 간결하게.
