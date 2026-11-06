# my_maps Walkthrough

## 2026-11-06 — 저장된 GPS 핀 드래그로 위치 수정

### 배경

이미 저장된 핀도 GPS 오차 보정을 위해 지도에서 드래그해 옮길 수 있어야 함.

### 변경

- 저장된 핀 마커를 `draggable`로 표시
- `dragend` 시 Firestore `pins/{id}`의 lat/lng 업데이트 + 표 동기화
- 다른 기기 `onSnapshot` `modified`로 실시간 반영
- `firestore.rules`: pins `update` 허용 (이름·createdAt 변경은 불가, 삭제 불가)
- `APP_VERSION` 20261106-1

### 콘솔

Firebase Rules에 로컬 `firestore.rules`를 다시 게시해야 드래그 저장이 동작함.

## 2026-11-04 — 로그인 세션 재수정 (localStorage)

### 원인

1. 초기엔 `loadConfig` 전에 session 복원을 시도해 실패
2. 이후에도 `sessionStorage` 사용 → 모바일 탭 폐기/복원 시 자주 소실 → 다시 로그인 화면

### 수정 (`APP_VERSION` 20261104-6)

- 인증을 `localStorage`로 저장 (탭 닫았다 열어도 유지)
- 비밀번호 토큰(`wolmi_auth_pw`) 저장 → 비번 변경 시에만 재로그인
- 구 `sessionStorage` 세션은 localStorage로 이관
- `config.json`은 `cache: no-store` + 최대 3회 재시도

## 2026-11-04 — 새로고침 시 로그인 세션 풀림 수정

### 원인

`sessionStorage`에 `wolmi_auth`가 있어도, 부트 시점에 `PAGE_PASSWORD`가 아직 `null`이라 복원 분기가 항상 실패함.
`loadConfig()` 이후에도 세션 체크 없이 비밀번호 입력만 focus 했음.

### 수정

- 항상 `loadConfig()` 후 `wolmi_auth === '1'`이면 `unlock()` (`APP_VERSION` 20261104-5)

## 2026-11-04 — GPS 오차 보정(위치 미세 조정)

### 배경

현장 GPS는 수 미터 오차가 흔해, 찍힌 좌표를 저장 전에 손으로 맞출 수 있어야 함.

### 변경

- 「현재 위치 핀 찍기」→ 즉시 저장하지 않고 조정 모드 진입
- 드래그 / 지도 탭 / 동서남북 버튼(1·2·5·10m)으로 위치 이동
- 가운데 `GPS` 버튼으로 원위치 복귀
- 「이 위치로 저장」 시에만 Firestore 기록 (`APP_VERSION` 20261104-4)

## 2026-11-04 — 버전 폴링으로 모바일 자동 새로고침

### 배경

현장 모바일은 강력 새로고침이 어렵고, GitHub Pages 캐시 때문에 배포가 안 보이는 경우가 많음.

### 변경

- `version.json` + `index.html`의 `APP_VERSION` 동기화
- 10초마다(및 탭 복귀 시) `version.json` 조회, 다르면 `location.replace`로 자동 갱신
- 캐시 방지 메타 태그 추가
- 배포 시 둘 다 같은 값으로 bump 필요

## 2026-11-04 — 내 위치 실시간 표기 (+ 다른 작업자 공유)

### 배경

핀이 많아지면 지도가 넓게 맞춰져 현재 GPS 위치를 찾기 어려움.
여러 명이 동시에 움직일 때 서로의 현재 위치도 지도에 보여야 함.

### 변경

- `index.html`
  - 게이트 통과 후 `watchPosition`으로 내 GPS 연속 갱신 (파란 원)
  - `liveLocations/{clientId}`에 약 2초 간격으로 위치 기록
  - `onSnapshot`으로 다른 작업자 위치 표시 (주황 원)
  - 「내 위치로 이동」은 추적 시작 + 지도 중심 이동
- `firestore.rules`: `liveLocations` read/create/update 허용 추가

### 콘솔

Rules를 다시 게시해야 실시간 위치 쓰기가 동작함.

## 2026-11-04 — Firebase Firestore 핀 저장/불러오기 + 실시간 동기화

### 배경

하루이틀 현장용으로 새로고침·기기 전환 시에도 핀이 남도록 서버 저장이 필요함.
GitHub Pages는 정적 호스팅이라 쓰기 불가 → Firebase Firestore 연동.
여러 명이 동시에 찍을 때 서로 핀을 바로 볼 수 있어야 함 → `onSnapshot` 구독.

### 변경

- `index.html`: Firebase compat SDK + Firestore
  - 게이트 통과 후 `pins` 컬렉션 `onSnapshot` 구독 (추가분 실시간 반영)
  - 「현재 위치 핀 찍기」 시 Firestore 저장 후 UI 반영, `seenPinIds`로 스냅샷 중복 방지
- `firestore.rules`: 단기용 규칙 (read/create만, update/delete 금지)

### 콘솔에서 할 일

1. Firebase 콘솔 → Firestore Database **만들기** (아직 없으면)
2. Rules 탭에 `firestore.rules` 내용 게시
3. GitHub에 push하면 Pages에 반영 (또는 로컬에서 바로 테스트)

### 보안 메모

- 클라이언트 `apiKey` 노출은 Firebase 기본 동작
- 페이지 게이트와 Firestore Rules는 별개 → Rules가 실제 쓰기 방어선
- 작업 끝나면 Rules를 잠그거나 프로젝트/데이터를 정리하는 것을 권장

## 2026-06-27 — GAS 연동 실패 진단 및 index.html 수정

### 증상

핀을 찍으면 화면에는 "보안저장요청완료"로 표시되지만, Google Drive에 `my_gis_pins.csv`가 생성·갱신되지 않음.

### 진단 결과

**Google Apps Script 서버 자체는 정상 동작함** (원격 POST 테스트):

| 테스트 | 결과 |
|--------|------|
| `password: admin@123` | `{"result":"success"}` |
| 잘못된 비밀번호 | `{"result":"error","error":"Unauthorized"}` |
| GET (doGet 없음) | `doGet 함수를 찾을 수 없습니다` (POST 전용이면 정상) |

### 근본 원인 (프론트엔드)

1. **`mode: 'no-cors'` + `Content-Type: application/json`**
   - `application/json`은 브라우저가 CORS preflight(OPTIONS)를 보냄.
   - GAS에 `doOptions`가 없어 OPTIONS → **405 Method Not Allowed** → 실제 POST가 차단됨.
   - `no-cors`는 응답 본문을 읽을 수 없어 실패를 감지할 수 없음.

2. **항상 성공 처리**
   - `fetch().then()`에서 응답 검증 없이 `uploaded = true`로 설정 → 서버 실패·비밀번호 오류가 화면에 안 보임.

3. **비밀번호 불일치 가능성**
   - 페이지 접속 시 입력한 비밀번호가 GAS의 `MASTER_PASSWORD`(`admin@123`)와 다르면 서버는 `Unauthorized` 반환.

### index.html 수정 내용

- `no-cors` 제거 → 일반 CORS `fetch`로 응답 JSON 파싱.
- `Content-Type`을 `text/plain;charset=utf-8`로 변경 (preflight 없이 JSON 본문 전송, GAS `JSON.parse`와 호환).
- `result.result === 'success'`일 때만 저장 완료 표시, 실패 시 alert 및 "저장 실패" 배지.

### GAS 측 확인 체크리스트

Apps Script 편집기에서:

1. **배포** → **새 배포** → 유형: **웹 앱**
2. **다음 사용자 인증 정보로 실행**: **나**
3. **액세스 권한**: **모든 사용자** (익명 포함)
4. 코드 저장 후 **반드시 새 배포** (기존 URL 유지 시 "버전 관리"에서 새 버전 선택)
5. 배포 URL이 `index.html`의 `SCRIPT_URL`과 일치하는지 확인
6. **실행** 탭에서 POST 요청 로그·오류 확인

### GAS 권장 보완 코드 (선택)

`application/json`을 쓰려면 `doOptions` 추가:

```javascript
function doOptions() {
  return ContentService.createTextOutput('')
    .setMimeType(ContentService.MimeType.TEXT);
}
```

현재 프론트는 `text/plain`으로 JSON을내므로 `doOptions` 없이도 동작함.
