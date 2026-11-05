# API 명세

주소 이름, 필드 이름, 상태 값은 구현하면서 바뀌어도 돼요.

## 공통

- **주소:** Nginx가 `/auth`는 인증 서비스로, `/api`는 메인 백엔드로 넘겨요. `/internal`은 서버끼리만 부르는 주소라 밖에 열지 않아요. Jetson 워커는 SSH 터널로, 인증 서비스는 서버 안에서 불러요.
- **인증:** `/api`는 `Authorization: Bearer <액세스 토큰>`이 필요해요. `/internal`은 서버끼리 쓰는 토큰을 따로 써요.
- **오류:** `{ "code": "CARD_NOT_FOUND", "message": "..." }` 모양으로 돌려줘요.
- **목록:** `?cursor=...&size=20`으로 다음 페이지를 받아요. 응답에 `nextCursor`가 있어요.
- **오래 걸리는 일:** 이야기 감지와 만화 만들기는 바로 `202`로 답하고, 상태는 조회 API와 푸시로 알려요.
- **시각:** 실제 시각은 ISO-8601 문자열(`2026-11-10T12:30:00+09:00`)이에요. `startMs`·`endMs`·`carryFromMs`는 녹음 시작부터 잰 밀리초예요.
- **화자 키:** 카드와 콘티 모두 `"화자1"`, `"화자2"`…로 맞춰요. 콘티 스키마의 화자 키 패턴 변경은 AI 담당이에요.

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

**토큰과 오류**

- 받는 토큰은 `TokenResponse { accessToken, refreshToken, accessTokenExpiresIn }`이에요. accessTokenExpiresIn은 초 단위예요.
- provider는 주소에서는 소문자(`kakao`), linkedProviders에서는 대문자(`KAKAO`)로 써요.
- 오류: `INVALID_CREDENTIALS` 401(이메일·비밀번호가 틀림), `TOKEN_EXPIRED` 401, `INVALID_TOKEN` 401.
- **JWT:** HS256으로 서명해요. 비밀값은 환경 변수 `JWT_SECRET` 하나를 인증 서비스와 메인 백엔드가 같이 가져요. `JWT_SECRET`은 32바이트(256비트) 이상이어야 해요(짧으면 Spring이 거부해요).
- 액세스 토큰에는 `sub`(사용자 ID, 문자열), `typ: "access"`, `iat`, `exp`가 들어가요. 액세스 토큰은 30분, 리프레시 토큰은 14일 뒤 만료돼요. 리프레시 토큰은 JWT가 아닌 무작위 문자열이고 Redis에 해시로 저장해요.
- 인증 서비스 없이 시험할 때는 메인 백엔드 레포의 로컬 전용 스크립트로 `JWT_SECRET`을 써서 테스트 토큰(사용자 1번, 1주일)을 만들어요.

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

**응답 모양과 규칙**

- 같은 녹음이면 묶음이 몇 개든 같은 recordingId를 돌려줘요. 앱의 "분석 중" 화면은 이 recordingId로 상태를 봐요.
- GET /api/cards 목록 항목: `{ id, kind, summary, recordedAt, startMs, endMs, comicCount, createdAt }`
- GET /api/cards/{id}에는 `recordingId`도 들어가요(null 가능).
- GET /api/comics 목록 항목은 GET /api/comics/{id}와 같은 모양이에요.
- PATCH /api/cards/{id}에서 카드에 없는 화자 키를 쓰면 400 UNKNOWN\_SPEAKER예요.
- **imageUrl:** SeaweedFS의 미리 서명한 주소이고 10분 뒤 만료돼요. 만료 시간은 응답에 넣지 않아요. 앱은 이미지를 못 받으면(403) GET /api/comics/{id}를 다시 불러 새 주소를 받고, 주소를 저장하지 않아요.
- 실패 이유는 앱에 보내지 않아요. 앱은 FAILED면 "만들지 못했어요"만 보여 줘요. 썸네일은 따로 없고 목록도 imageUrl을 써요.
- **푸시:** FCM data 메시지이고 값은 모두 문자열이에요. `{ type: "CARDS_CREATED", recordingId, cardCount }`, `{ type: "COMIC_DONE", comicId, cardId }`, `{ type: "COMIC_FAILED", comicId, cardId }`
- 푸시가 붙기 전에는 앱이 GET /api/recordings/{id}·GET /api/comics/{id}를 5초마다 부르고, DONE이나 FAILED가 되면 멈춰요.

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
  "recordedAt": "2026-11-10T12:30:00+09:00",   // 녹음 날짜
  "startMs": 1200, "endMs": 95000,
  "speakers": [ { "key": "화자1", "name": null } ],
  "lines": [ { "id": 1, "speaker": "화자1", "text": "..." } ],
  "comics": [ { "id": 31, "status": "DONE" }, { "id": 35, "status": "DRAWING" } ],
  "createdAt": "..."
}
```

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

- **인증:** 헤더 `X-Internal-Token: <비밀값>`을 붙여요. 값은 환경 변수 `INTERNAL_TOKEN`이고 메인 백엔드·워커·인증 서비스가 같은 값을 가져요. 틀리면 401이에요.
- **응답:** GET /internal/comics/{id}/source는 200 `ComicSource`, 나머지는 모두 204예요. 이미지는 multipart의 `file`(image/png)로 보내요.
- **DetectResult** (감지 결과 보내기): `{ jobId, partNo, cards[], carryFromMs }`. 카드가 없으면 빈 배열, 마지막 묶음이면 carryFromMs는 null이에요.
  - 카드 한 장: `{ kind, summary, startMs, endMs, speakers["화자1", …], lines[{ lineNo, speaker, text, startMs, endMs }] }`. lines의 startMs·endMs는 null이어도 돼요.
- **ComicSource** (카드 내용 받기): `{ comicId, cardId, kind, summary, place, speakers[{ key, name }], lines[{ id, speaker, text }] }`. lines의 id는 콘티의 `sourceLineIds`에 써요.
- **만화 상태가 바뀌는 때:** source를 가져가면 STORYBOARD, storyboard를 받으면 DRAWING, image를 받으면 DONE과 푸시, fail을 받으면 FAILED와 푸시예요.

## 큐 메시지 (Redis Streams)

detect 메시지에는 대화 텍스트 묶음을 같이 넣어요. sentences\[\]에는 앞 묶음에서 넘겨받은 부분(carryFromMs 이후)을 앞에 붙여 넣어요(기능 명세의 "묶음 경계" 참고). 처리가 끝나면 워커가 완료 표시(XACK)를 하고 메시지를 지워요(XDEL). 꺼내 가도 스트림에 남기 때문에, 대화 글이 큐에 남지 않게 하려는 뜻이에요. comic 메시지는 ID만 넣고, 카드 내용은 워커가 `/internal`에서 받아 가요.

- **detect:** `{ jobId, recordingId, partNo, isLast, sentences[] }`
- **comic:** `{ jobId, comicId, cardId }`
- **컨슈머 그룹:** `jetson-workers`이고, 컨슈머 이름은 워커마다 `jetson-1`처럼 붙여요. 그룹은 워커가 켜질 때 `XGROUP CREATE ... MKSTREAM`으로 만들고, 이미 있으면 넘어가요.
- **다시 시도:** 워커가 죽어 XACK하지 못한 메시지는 워커가 다시 켜질 때 `XAUTOCLAIM`으로 10분 넘게 멈춘 것을 가져와 다시 처리해요. 같은 jobId가 3번 실패하면 fail로 알려요. 메인 백엔드는 같은 jobId의 결과가 두 번 오면 두 번째는 204로 받고 무시해요.

## 담당과 정할 것

- **컷 배치:** 세로 4칸으로 정했어요. 컷을 하나씩 그려 세로로 합쳐요. 사용자가 고르는 옵션은 없어요.
- **다시 만들기:** 같은 주소(POST /api/cards/{id}/comics)를 다시 부르면 새 만화가 하나 더 생겨요. 한 카드에 3번까지이고(실패한 만화는 세지 않아요), 넘으면 메인 백엔드가 409 COMIC\_LIMIT로 막아요. 그 카드에 만드는 중인 만화가 있으면 409 COMIC\_IN\_PROGRESS로 막아요.
- **콘티 모양:** [콘티 형식](storyboard.md)으로 정했어요. 스키마는 `spec/storyboard.schema.json`이에요.
- **추가 구현(P1·P2):** 녹음 파일 업로드, 카드 구간 재생·조정, 피드 업로드 API는 그때 추가해요. 공유는 P0이고 GET /api/comics/{id}의 imageUrl로 해요.
