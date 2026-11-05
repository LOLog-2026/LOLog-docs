# API 명세

주소 이름, 필드 이름, 상태 값은 구현하면서 바뀌어도 돼요.

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
| PATCH | /api/cards/{id} | 카드 고치기. 그 카드로 만화를 만드는 중이면 409 CARD\_LOCKED | speakers\[\], lines\[\] (고친 대사만 id·speaker·text), place (바꾼 것만). 대사의 화자를 바꾸려면 그 대사의 speaker를 바꿔요 | 카드 전체 | FR-EDIT-01 |
| DELETE | /api/cards/{id} | 카드 지우기. 그 카드로 만화를 만드는 중이면 409 CARD\_LOCKED | - | - | FR-LIB-03 |
| POST | /api/cards/{id}/comics | 만화 요청. 그 카드에 만드는 중인 만화가 있으면 409 COMIC\_IN\_PROGRESS, 3번을 넘으면 409 COMIC\_LIMIT(실패한 만화는 세지 않음) | - | 202, comicId, status | FR-COMIC-01 |
| GET | /api/comics | 내 만화 목록 | cursor, size | comics\[\], nextCursor | FR-LIB-01 |
| GET | /api/comics/{id} | 만화 상태·결과 | - | id, cardId, status, imageUrl, createdAt (cardId는 카드를 지웠으면 null) | FR-COMIC-03 |
| DELETE | /api/comics/{id} | 만화 지우기 | - | - | FR-LIB-03 |
| PUT | /api/devices/fcm-token | 푸시 토큰 등록 | token | - | FR-NOTI |
| DELETE | /api/devices/fcm-token | 이 폰의 푸시 토큰 지우기(로그아웃할 때) | token | - | FR-AUTH-04 |

**보내는 데이터 예시**

```json
// POST /api/recordings
{
  "clientRecordingId": "7f3c...",   // 앱이 만든 ID
  "partNo": 1,                    // 1부터 빠짐없이 1, 2, 3. 같은 ID·partNo면 한 번만 처리
  "isLast": false,                 // 녹음이 끝난 묶음이면 true
  "startedAt": "2026-11-10T12:30:00+09:00", // 녹음 전체의 시작 시각. 모든 묶음에 같은 값
  "durationSec": 3600,             // 이 묶음의 길이
  "sentences": [                   // 말소리 구간 단위. 시간은 녹음 시작부터
    { "startMs": 1200, "endMs": 3400, "text": "야 너 어제 그거 봤어" }
  ]
}

// GET /api/cards/{id}
{
  "id": 12, "kind": "웃김", "summary": "...", "place": null,
  "recordedAt": "2026-11-10T12:30:00+09:00",   // 녹음 날짜
  "startMs": 1200, "endMs": 95000,
  "speakers": [ { "key": "화자1", "name": null } ],
  "lines": [ { "id": 1, "speaker": "화자1", "text": "..." } ],
  "comics": [ { "id": 31, "status": "DONE" }, { "id": 35, "status": "DRAWING" } ],
  "createdAt": "..."
}
```

**POST /api/recordings 규칙**

| 항목 | 규칙 |
| --- | --- |
| 묶음 번호 | partNo는 녹음마다 1부터 빠짐없이 이어요. 앱은 앞 묶음의 202를 받은 뒤 다음 묶음을 올려요. 건너뛴 번호가 오면 409 PART\_OUT\_OF\_ORDER예요 |
| 빈 묶음 | 말이 없던 시간에도 묶음을 올리고 sentences는 빈 배열로 보내요. 마지막 묶음이 비어도 isLast: true로 올려야 서버가 녹음이 끝난 걸 알아요 |
| 재시도 | 같은 clientRecordingId·partNo가 다시 오면 저장하지 않고 처음 결과를 202로 돌려줘요. 내용이 달라도 처음 것을 유지해요 |
| 녹음 끝 | isLast: true를 받은 뒤 더 큰 partNo가 오면 409 RECORDING\_CLOSED예요 |
| 시간 값 | startedAt은 첫 묶음 값을 유지해요. durationSec는 0 이상의 정수예요. startMs, endMs는 녹음 시작부터의 밀리초이고, 0 ≤ startMs < endMs예요 |
| sentences | 이름과 달리 문장이 아니라 말소리 구간(VAD) 단위예요. 측정에서 한 구간이 최대 38.6초였고, 여러 문장이 들어 있고 문장부호가 거의 없어요. text는 비어 있으면 안 되고, 앱은 빈 인식 결과를 올리지 않아요 |
| 크기 제한 | 묶음당 5,000구간, 본문 1MB까지예요. 넘으면 413 PAYLOAD\_TOO\_LARGE예요 |
| 응답 | `202 { "recordingId": 3, "status": "RECEIVED" }`. status는 녹음의 상태예요 |
| 오류 | 형식이 틀리면 400 INVALID\_REQUEST, 토큰이 없거나 틀리면 401 UNAUTHORIZED예요. 앱은 네트워크 오류와 5xx만 다시 시도하고, 401은 토큰을 갱신한 뒤 다시 시도해요 |

**상태 값**

- 녹음: `RECEIVED` → `DETECTING` → `DONE` / `FAILED`
- 만화: `QUEUED` → `STORYBOARD` → `DRAWING` → `DONE` / `FAILED`

## 워커 전용 (/internal)

Jetson 워커가 SSH 터널로 부르는 주소예요. 사용자 데이터 지우기만 인증 서비스가 불러요.

| 메서드 | 주소 | 하는 일 | 보내는 것 |
| --- | --- | --- | --- |
| POST | /internal/recordings/{id}/cards | 감지 결과(카드) 보내기 | cards\[\] (없으면 빈 배열), carryFromMs(다음 묶음으로 넘길 시작 시각, 마지막 묶음이면 없음) |
| GET | /internal/comics/{id}/source | 만화 만들 카드 내용 받기 | - |
| POST | /internal/comics/{id}/storyboard | 콘티 보내기 | 콘티 JSON: characters\[\], panels\[4\] (장면, 인물 위치, 대사). 형식은 [콘티 형식](storyboard.md) |
| POST | /internal/comics/{id}/image | 완성 4컷 올리기 | PNG (multipart) |
| POST | /internal/jobs/{id}/fail | 실패 알리기 | reason |
| DELETE | /internal/users/{id} | 탈퇴한 사용자의 데이터 모두 지우기. 인증 서비스가 불러요 | - |

## 큐 메시지 (Redis Streams)

detect 메시지에는 대화 텍스트 묶음을 같이 넣어요. sentences\[\]에는 앞 묶음에서 넘겨받은 부분(carryFromMs 이후)을 앞에 붙여 넣어요(기능 명세의 "묶음 경계" 참고). 처리가 끝나면 워커가 완료 표시(XACK)를 하고 메시지를 지워요(XDEL). 꺼내 가도 스트림에 남기 때문에, 대화 글이 큐에 남지 않게 하려는 뜻이에요. comic 메시지는 ID만 넣고, 카드 내용은 워커가 `/internal`에서 받아 가요.

- **보내는 곳:** 스트림 이름은 `lolog:detect`, `lolog:comic`이에요. 메시지 필드는 `payload` 하나이고 값은 아래 모양의 JSON 문자열이에요.
- **detect:** `{ jobId, recordingId, partNo, isLast, sentences[] }`
  - 앞 묶음의 carryFromMs를 알아야 하므로, 앞 묶음의 감지가 끝난 뒤에 다음 묶음을 큐에 넣어요. 앞 묶음이 실패했으면 넘겨받을 것 없이 넣어요.
  - 새 글도 넘겨받을 글도 없는 묶음은 큐에 넣지 않고 바로 끝난 것으로 처리해요.
- **comic:** `{ jobId, comicId, cardId }`

## 담당과 정할 것

- **컷 배치:** 세로 4칸으로 정했어요. 컷을 하나씩 그려 세로로 합쳐요. 사용자가 고르는 옵션은 없어요.
- **다시 만들기:** 같은 주소(POST /api/cards/{id}/comics)를 다시 부르면 새 만화가 하나 더 생겨요. 한 카드에 3번까지이고(실패한 만화는 세지 않아요), 넘으면 메인 백엔드가 409 COMIC\_LIMIT로 막아요. 그 카드에 만드는 중인 만화가 있으면 409 COMIC\_IN\_PROGRESS로 막아요.
- **콘티 모양:** [콘티 형식](storyboard.md)으로 정했어요. 스키마는 `spec/storyboard.schema.json`이에요.
- **추가 구현(P1·P2):** 녹음 파일 업로드, 카드 구간 재생·조정, 피드 업로드 API는 그때 추가해요. 공유는 P0이고 GET /api/comics/{id}의 imageUrl로 해요.
