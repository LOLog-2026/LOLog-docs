# API 명세

초안이에요. 주소 이름, 필드 이름, 상태 값은 제안이고 구현하면서 바뀌어도 돼요.

## 공통

- **주소:** Nginx가 `/auth`는 인증 서비스로, `/api`는 메인 백엔드로 넘겨요. `/internal`은 서버끼리만 부르는 주소라 밖에 열지 않아요. Jetson 워커는 SSH 터널로, 인증 서비스는 서버 안에서 불러요.
- **인증:** `/api`는 `Authorization: Bearer <액세스 토큰>`이 필요해요. `/internal`은 서버끼리 쓰는 토큰을 따로 써요.
- **오류:** `{ "code": "CARD_NOT_FOUND", "message": "..." }` 모양으로 돌려줘요.
- **목록:** `?cursor=...&size=20`으로 다음 페이지를 받아요. 응답에 `nextCursor`가 있어요.
- **오래 걸리는 일:** 이야기 감지와 만화 만들기는 바로 `202`로 답하고, 상태는 조회 API와 푸시로 알려요.

## 인증 서비스 (/auth)

| 메서드 | 주소 | 하는 일 | 보내는 것 | 받는 것 | 요구사항 |
| --- | --- | --- | --- | --- | --- |
| POST | /auth/signup | 이메일 가입. 이미 가입된 이메일이면(소셜 가입 포함) 409 ACCOUNT\_EXISTS | email, password | accessToken, refreshToken | FR-AUTH-01 |
| POST | /auth/login | 이메일 로그인 | email, password | accessToken, refreshToken | FR-AUTH-01 |
| POST | /auth/social/{provider} | 소셜 로그인 (provider = kakao, google, naver). 그 이메일로 이미 가입된 계정이 있으면 409 ACCOUNT\_EXISTS | providerToken | accessToken, refreshToken, isNewUser | FR-AUTH-02 |
| POST | /auth/refresh | 토큰 갱신 | refreshToken | accessToken, refreshToken | FR-AUTH-03 |
| POST | /auth/logout | 로그아웃 | refreshToken | - | FR-AUTH-04 |
| DELETE | /auth/me | 탈퇴. 회원 정보는 소프트 삭제하고, 메인 백엔드가 그 사용자의 데이터를 모두 지워요(DELETE /internal/users/{id}) | - | - | FR-AUTH-05 |
| POST | /auth/me/social/{provider} | 로그인한 계정에 소셜 계정 연결. 다른 계정에 이미 연결돼 있으면 409 SOCIAL\_ALREADY\_LINKED | providerToken | linkedProviders\[\] | FR-AUTH-06 |
| GET | /auth/me | 내 정보와 연결된 소셜 계정 보기 | - | email, linkedProviders\[\] | FR-AUTH-06 |

## 메인 백엔드 (/api)

| 메서드 | 주소 | 하는 일 | 보내는 것 | 받는 것 | 요구사항 |
| --- | --- | --- | --- | --- | --- |
| POST | /api/recordings | 녹음 글 올리기(녹음 중 1시간마다, 녹음이 끝날 때) | clientRecordingId, partNo, isLast, startedAt, durationSec, sentences\[\] | 202, recordingId, status | FR-UP-01, 02 |
| GET | /api/recordings/{id} | 감지 상태 보기 | - | status, cardCount | FR-CARD |
| GET | /api/cards | 내 카드 목록 | cursor, size | cards\[\], nextCursor | FR-LIB-01 |
| GET | /api/cards/{id} | 카드 자세히 | - | 카드 전체, 이 카드로 만든 comics\[\] (id, status) | FR-LIB-01 |
| PATCH | /api/cards/{id} | 카드 고치기 | speakers\[\], lines\[\] (대사마다 speaker·text), place (바꾼 것만). 대사의 화자를 바꾸려면 그 대사의 speaker를 바꿔요 | 카드 전체 | FR-EDIT-01 |
| DELETE | /api/cards/{id} | 카드 지우기 | - | - | FR-LIB-03 |
| POST | /api/cards/{id}/comics | 만화 요청 | - | 202, comicId, status | FR-COMIC-01 |
| GET | /api/comics | 내 만화 목록 | cursor, size | comics\[\], nextCursor | FR-LIB-01 |
| GET | /api/comics/{id} | 만화 상태·결과 | - | status, imageUrl | FR-COMIC-03 |
| DELETE | /api/comics/{id} | 만화 지우기 | - | - | FR-LIB-03 |
| PUT | /api/devices/fcm-token | 푸시 토큰 등록 | token | - | FR-NOTI |
| DELETE | /api/devices/fcm-token | 이 폰의 푸시 토큰 지우기(로그아웃할 때) | token | - | FR-AUTH-04 |

**보내는 데이터 예시**

```json
// POST /api/recordings
{
  "clientRecordingId": "7f3c...",   // 앱이 만든 ID
  "partNo": 1,                    // 1시간마다 1, 2, 3. 같은 ID·partNo면 한 번만 처리
  "isLast": false,                 // 녹음이 끝난 묶음이면 true
  "startedAt": "2026-11-10T12:30:00+09:00",
  "durationSec": 3600,
  "sentences": [
    { "startMs": 1200, "endMs": 3400, "text": "야 너 어제 그거 봤어?" }
  ]
}

// GET /api/cards/{id}
{
  "id": 12, "kind": "웃김", "summary": "...", "place": null,
  "startMs": 1200, "endMs": 95000,
  "speakers": [ { "key": "A", "name": null } ],
  "lines": [ { "id": 1, "speaker": "A", "text": "..." } ],
  "comics": [ { "id": 31, "status": "DONE" }, { "id": 35, "status": "DRAWING" } ],
  "createdAt": "..."
}
```

**상태 값 (제안)**

- 녹음: `RECEIVED` → `DETECTING` → `DONE` / `FAILED`
- 만화: `QUEUED` → `STORYBOARD` → `DRAWING` → `DONE` / `FAILED`

## 워커 전용 (/internal)

Jetson 워커가 SSH 터널로 부르는 주소예요. 사용자 데이터 지우기만 인증 서비스가 불러요.

| 메서드 | 주소 | 하는 일 | 보내는 것 |
| --- | --- | --- | --- |
| POST | /internal/recordings/{id}/cards | 감지 결과(카드) 보내기 | cards\[\] (없으면 빈 배열), carryFromMs(다음 묶음으로 넘길 시작 시각, 마지막 묶음이면 없음) |
| GET | /internal/comics/{id}/source | 만화 만들 카드 내용 받기 | - |
| POST | /internal/comics/{id}/storyboard | 콘티 보내기 | panels\[4\] (장면, 인물, 대사) |
| POST | /internal/comics/{id}/image | 완성 4컷 올리기 | PNG (multipart) |
| POST | /internal/jobs/{id}/fail | 실패 알리기 | reason |
| DELETE | /internal/users/{id} | 탈퇴한 사용자의 데이터 모두 지우기. 인증 서비스가 불러요 | - |

## 큐 메시지 (Redis Streams)

detect 메시지에는 대화 텍스트 묶음을 같이 넣어요. sentences\[\]에는 앞 묶음에서 넘겨받은 부분(carryFromMs 이후)을 앞에 붙여 넣어요(기능 명세의 "묶음 경계" 참고). 처리가 끝나면 워커가 완료 표시(XACK)를 하고 메시지를 지워요(XDEL). 꺼내 가도 스트림에 남기 때문에, 대화 글이 큐에 남지 않게 하려는 뜻이에요. comic 메시지는 ID만 넣고, 카드 내용은 워커가 `/internal`에서 받아 가요.

- **detect:** `{ jobId, recordingId, partNo, isLast, sentences[] }`
- **comic:** `{ jobId, comicId, cardId }`

## 담당과 정할 것

- **컷 배치:** 2×2인지 세로 4칸인지 담당과 정해요. 사용자가 고르는 옵션은 없어요.
- **다시 만들기:** 같은 주소(POST /api/cards/{id}/comics)를 다시 부르면 새 만화가 하나 더 생겨요. 한 카드에 3번까지이고, 넘으면 메인 백엔드가 409 COMIC\_LIMIT로 막아요.
- **콘티 모양:** 칸마다 어떤 필드를 그림 모델에 넘길지는 그림 모델 담당과 맞춰요.
- **추가 구현(P1·P2):** 녹음 파일 업로드, 카드 구간 재생·조정, 피드 업로드 API는 그때 추가해요. 공유는 P0이고 GET /api/comics/{id}의 imageUrl로 해요.
