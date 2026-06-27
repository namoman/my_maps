# my_maps Walkthrough

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
