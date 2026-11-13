# API 명세

주소 이름, 필드 이름, 상태 값은 구현하면서 바뀌어도 돼요.

필드 단위 정의는 [API DTO](api-dto.md)에 있어요.

## 공통

- **서버 상태 확인:** `GET /auth/health`, `GET /api/health`는 토큰 없이 부르고 `200 { "status": "UP" }`를 돌려줘요. DB·Redis에 붙지 못하면 `503 { "status": "DOWN" }`이에요. 연결 시험용이에요.

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
| POST | /auth/email/code | 가입 인증번호 보내기. 늘 204예요. 가입 안 된 이메일(탈퇴한 계정 포함)에는 6자리 번호를, 이미 가입된 이메일(소셜 가입 포함)에는 번호 대신 "이미 가입된 계정" 안내를 메일로 보내요. 같은 이메일로 60초 안에 다시 부르면 429 TOO\_MANY\_REQUESTS | email | 204 - | FR-AUTH-01 |
| POST | /auth/signup | 이메일 가입. 인증번호를 먼저 확인해요: 틀리면 400 INVALID\_CODE, 없거나 만료(10분)됐거나 5번 틀렸으면 400 CODE\_EXPIRED. 번호가 맞고 이미 가입된 이메일이면(소셜 가입 포함) 409 ACCOUNT\_EXISTS. 탈퇴한 계정의 이메일로는 다시 가입할 수 있어요 | email, password, code | accessToken, refreshToken | FR-AUTH-01 |
| POST | /auth/login | 이메일 로그인 | email, password | accessToken, refreshToken | FR-AUTH-01 |
| POST | /auth/social/{provider} | 소셜 로그인 (provider = google, naver). 그 이메일로 이미 가입된 계정이 있으면 409 ACCOUNT\_EXISTS. 그 회사에 providerToken을 확인하지 못하면 401 INVALID\_TOKEN | providerToken | accessToken, refreshToken, isNewUser | FR-AUTH-02 |
| POST | /auth/refresh | 토큰 갱신 | refreshToken | accessToken, refreshToken | FR-AUTH-03 |
| POST | /auth/logout | 로그아웃. 없거나 이미 쓴 refreshToken이어도 204 | refreshToken | - | FR-AUTH-04 |
| DELETE | /auth/me | 탈퇴. 회원 정보는 소프트 삭제하고, 메인 백엔드가 그 사용자의 데이터를 모두 지워요(DELETE /internal/users/{id}). 메인 백엔드가 실패해도 204이고, 인증 서비스가 204가 올 때까지 다시 불러요. 탈퇴한 회원의 토큰으로 다시 부르면 401 INVALID\_TOKEN이에요 | - | - | FR-AUTH-05 |
| POST | /auth/me/social/{provider} | 로그인한 계정에 소셜 계정 연결. 다른 계정에 이미 연결돼 있거나, 내 계정에 같은 회사의 다른 계정이 이미 연결돼 있으면 409 SOCIAL\_ALREADY\_LINKED. 같은 계정을 다시 연결하면 그대로 200이에요. 탈퇴한 계정에 붙어 있던 소셜 계정은 다시 연결할 수 있어요 | providerToken | linkedProviders\[\] | FR-AUTH-06 |
| DELETE | /auth/me/social/{provider} | 내 계정에서 그 회사 연결 끊기. 연결돼 있지 않아도 200이에요. 비밀번호가 없는 계정의 마지막 소셜 연결이면 409 LAST\_LOGIN\_METHOD | - | linkedProviders\[\] | FR-AUTH-06 |
| GET | /auth/me | 내 정보와 연결된 소셜 계정, 비밀번호 유무 보기 | - | email, linkedProviders\[\], hasPassword, role(초안) | FR-AUTH-06, FR-AUTH-07, FR-ADMIN-01 |
| POST | /auth/password/reset/code | 비밀번호 찾기 인증번호 보내기. 늘 바로 204로 답하고, 메일은 답한 뒤에 보내요. 비밀번호가 있는 계정이면 6자리 번호를, 소셜로만 가입한 계정이면 번호 대신 "소셜로 가입한 계정" 안내를 메일로 보내요. 가입 안 된 이메일이면 메일을 보내지 않아요. 같은 이메일로 60초 안에 다시 부르면 429 TOO\_MANY\_REQUESTS | email | 204 - | FR-AUTH-07 |
| POST | /auth/password/reset | 인증번호로 새 비밀번호 정하기. 틀리면 400 INVALID\_CODE, 없거나 만료(10분)됐거나 5번 틀렸으면 400 CODE\_EXPIRED. 바꾸면 그 계정의 로그인(리프레시 토큰)을 모두 끊어요 | email, code, newPassword | 204 - | FR-AUTH-07 |
| GET | /auth/admin/users | (관리자) 계정 목록. 자세히는 [관리자](admin.md) (초안) | q, sort, cursor, size | users\[\], nextCursor | FR-ADMIN-04 |
| POST·DELETE | /auth/admin/users/{id}/suspend | (관리자) 정지 · 정지 풀기 | duration, reason · - | 204 | FR-ADMIN-02 |
| DELETE | /auth/admin/users/{id} | (관리자) 탈퇴시키기. 같은 이메일·소셜 계정으로 다시 가입하지 못하게 막아요 | - | 204 | FR-ADMIN-04 |
| POST | /auth/me/password | 로그인한 상태에서 비밀번호 바꾸기. 지금 비밀번호가 틀리면 401 INVALID\_CREDENTIALS, 비밀번호가 없는 계정(소셜로만 가입)이면 400 INVALID\_REQUEST(field `currentPassword`). 다른 기기의 로그인은 끊고 지금 기기에는 새 토큰을 줘요 | currentPassword, newPassword | accessToken, refreshToken | FR-AUTH-07 |

**토큰과 오류**

- 받는 토큰 모양은 [API DTO](api-dto.md)의 TokenResponse예요.
- provider는 주소에서는 소문자(`google`), linkedProviders에서는 대문자(`GOOGLE`)로 써요. 카카오는 이메일 동의항목에 비즈 앱 심사가 필요해 이번 범위에서 뺌(2026-11-09). 다시 넣을 때는 이메일 없이 받는 설계부터 정한다.
- 오류: `INVALID_CREDENTIALS` 401(이메일·비밀번호가 틀림), `TOKEN_EXPIRED` 401, `INVALID_TOKEN` 401, `INVALID_CODE` 400, `CODE_EXPIRED` 400, `TOO_MANY_REQUESTS` 429.
- **정지된 계정(초안):** 로그인(`/auth/login`, `/auth/social/{provider}`)이 403 `ACCOUNT_SUSPENDED`예요. 정지할 때 리프레시 토큰을 모두 지워서 토큰 갱신(`/auth/refresh`)은 401 `INVALID_TOKEN`이에요. 앱은 401을 받아 로그인 화면으로 가고, 로그인에서 403으로 정지를 알아요(정함 11-13, 지원). 관리자가 탈퇴시킨 계정의 이메일·소셜 계정으로 가입·소셜 로그인하거나 그 소셜 계정을 연결하면 409 `ACCOUNT_BANNED`(이름은 초안)예요. [관리자](admin.md)
- **가입 인증번호:** 6자리 숫자이고 10분 동안 써요. 다시 받으면 전 번호는 못 쓰고, 5번 틀리면 그 번호도 못 써요. 맞게 쓴 번호도 한 번만 써요. 인증 서비스가 Redis에 해시로 저장하고, 키에도 이메일 원문을 넣지 않아요. 메일은 SMTP(Gmail)로 보내요. 메일을 보내지 못하면 500 `INTERNAL_ERROR`이고, 바로 다시 받을 수 있어요.
- **비밀번호 찾기 인증번호:** 가입 인증번호와 규칙이 같아요(6자리, 10분, 5번 틀리면 못 씀, 한 번만, 60초에 한 번). 가입 번호와 따로 저장해서 서로 바꿔 쓸 수 없어요. 가입 여부가 응답 내용이나 응답 시간으로 드러나지 않게, 60초 간격만 확인하고 바로 204로 답한 뒤 메일을 보내요. 그래서 메일을 보내지 못해도 앱은 알 수 없고(서버 로그에만 남아요), 사용자는 60초 뒤에 다시 받으면 돼요. 비밀번호를 바꾸면(찾기·바꾸기 모두) 그 회원의 리프레시 토큰을 모두 지워요. 이미 받은 액세스 토큰은 만료(최대 30분)까지 그대로 써요.
- 로그인이 필요한 /auth 주소(`GET /auth/me`, `DELETE /auth/me`, `POST`·`DELETE /auth/me/social/{provider}`, `POST /auth/me/password`)는 `Authorization: Bearer <액세스 토큰>`을 보내요. 토큰이 없거나 틀리면 401 `INVALID_TOKEN`, 만료면 401 `TOKEN_EXPIRED`예요(/api의 UNAUTHORIZED와 달라요).
- **JWT:** HS256으로 서명해요. 비밀값은 환경 변수 `JWT_SECRET` 하나를 인증 서비스와 메인 백엔드가 같이 가져요. `JWT_SECRET`은 32바이트(256비트) 이상이어야 해요(짧으면 Spring이 거부해요).
- 액세스 토큰에는 `sub`(사용자 ID, 문자열), `typ: "access"`, `iat`, `exp`, `role`(`USER`·`ADMIN`, 초안)이 들어가요. 액세스 토큰은 30분, 리프레시 토큰은 14일 뒤 만료돼요. 리프레시 토큰은 JWT가 아닌 무작위 문자열이고 Redis에 해시로 저장해요.
- **탈퇴한 회원의 토큰:** 탈퇴하면 인증 서비스가 그 회원의 리프레시 토큰을 모두 지우고, Redis에 `lolog:auth:withdrawn:{사용자 ID}` 키를 31분 동안 둬요(액세스 토큰 30분 + 시계 오차 30초 + 여유 30초). 이 키가 있으면 아직 만료 전인 액세스 토큰도 받지 않아요. 정지된 회원도 같은 방법으로 `lolog:auth:suspended:{사용자 ID}` 키를 봐요(초안, [관리자](admin.md)). /auth 주소는 401 `INVALID_TOKEN`, /api 주소는 401 `UNAUTHORIZED`예요. 메인 백엔드가 Redis에 못 붙어 이 키를 확인하지 못하면 토큰을 통과시키지 않고 500 `INTERNAL_ERROR`로 답해요(401이면 앱이 로그인을 풀어 버려요). 이 확인은 2초 안에 끝나지 않으면 실패로 봐요. 다만 만료된 토큰은 Redis와 관계없이 401 `TOKEN_EXPIRED`로 답해 앱이 토큰을 갱신하게 해요. 앱은 탈퇴 204를 받으면 저장한 토큰을 지워요.
- 인증 서비스 없이 시험할 때는 메인 백엔드 레포의 로컬 전용 스크립트로 `JWT_SECRET`을 써서 테스트 토큰(사용자 1번, 1주일)을 만들어요.

## 메인 백엔드 (/api)

| 메서드 | 주소 | 하는 일 | 보내는 것 | 받는 것 | 요구사항 |
| --- | --- | --- | --- | --- | --- |
| POST | /api/recordings | 녹음 글 올리기(녹음 중 1시간마다, 녹음이 끝날 때) | clientRecordingId, partNo, isLast, startedAt, durationSec, audioSaved, sentences\[\] | 202, recordingId, status | FR-UP-01, 02 |
| GET | /api/recordings | 내 녹음 목록. 앱을 다시 켰을 때 "분석 중"인 녹음을 찾는 데 써요. status로 거를 수 있어요(예: `?status=RECEIVED,DETECTING`) | cursor, size, status | recordings\[\], nextCursor | FR-CARD |
| GET | /api/recordings/{id} | 감지 상태 보기 | - | recordingId, status, cardCount | FR-CARD |
| PUT | /api/recordings/{id}/audio/{seq} | 음성 파일 올리기. 음성 저장을 켠 녹음만, 10분 단위 파일을 번호(seq) 순서대로 올려요. 같은 번호를 다시 올리면 덮어써요. 자세히는 [음성 저장](audio.md) | file (AAC, multipart), startMs, endMs | 204 | FR-EXT-01, FR-UP-03 |
| GET | /api/cards | 내 카드 목록. sort로 최신순·오래된순을 골라요 | cursor, size, sort | cards\[\] (hasAudio 포함), nextCursor | FR-LIB-01 |
| GET | /api/cards/{id} | 카드 자세히 | - | 카드 전체, 이 카드로 만든 comics\[\] (id, status), audio (status, url, durationMs. [순간 다시 듣기](replay.md)) | FR-LIB-01, FR-EXT-01 |
| PATCH | /api/cards/{id} | 카드 고치기. 그 카드로 만화를 만드는 중이면 409 CARD\_LOCKED | speakers\[\] (key와 name·character 중 바꾼 것. 다음 번호의 key를 보내면 새 화자), removeSpeakers\[\] (뺄 화자의 key), lines\[\] (고친 대사만 id·speaker·text), place (바꾼 것만). 대사의 화자를 바꾸려면 그 대사의 speaker를 바꿔요 | 카드 전체 | FR-EDIT-01 |
| DELETE | /api/cards/{id} | 카드 지우기. 그 카드로 만화를 만드는 중이면 409 CARD\_LOCKED | - | - | FR-LIB-03 |
| POST | /api/cards/{id}/comics | 만화 요청. 그 카드에 만드는 중인 만화가 있으면 409 COMIC\_IN\_PROGRESS, 3번을 넘으면 409 COMIC\_LIMIT(실패한 만화는 세지 않고, 지운 만화는 셈) | - | 202, comicId, status | FR-COMIC-01 |
| GET | /api/characters/defaults | 기본 화자 24명 목록 | - | characters\[\] (id, label, ageGroup, gender, glasses, thumbnailUrl, voiceUrl) | FR-CHAR-01, FR-CHAR-03 |
| PUT·DELETE | /api/me/profile/character | 본인 사진 한 장으로 프로필 캐릭터 만들기·다시 만들기 · 지우기. 자세히는 [화자 그림](characters.md) | photo · - | 202 status, thumbnailUrl, updatedAt · 204 | FR-CHAR-02 |
| PATCH | /api/me/profile/character/voice | 프로필 캐릭터 목소리 고르기(초안) | voice(기본 화자 id, null 가능) | 204 | FR-CHAR-04 |
| GET·PATCH | /api/me/profile | 내 프로필(닉네임, 친구 코드) 보기·닉네임 정하기. 처음 부르면 프로필을 만들고 친구 코드를 발급해요. 자세히는 [친구](friends.md) | - · nickname | nickname, friendCode | FR-FRIEND-02 |
| POST | /api/friend-requests | 친구 코드나 같은 방 멤버의 userId로 친구 요청 보내기 | friendCode 또는 userId | 201, 요청(id, direction, user, status, createdAt) | FR-FRIEND-01 |
| GET | /api/friend-requests | 받은 요청·보낸 요청 목록 | direction(`received`·`sent`) | requests\[\] | FR-FRIEND-01 |
| POST·DELETE | /api/friend-requests/{id}(/accept) | 받은 요청 수락 · 거절, 보낸 요청 취소 | - | 200 친구 · 204 | FR-FRIEND-01 |
| GET | /api/friends | 내 친구 목록(최근에 친구가 된 순) | - | friends\[\] (userId, nickname, friendCode, friendsSince, character(쓸 수 있는 그림이 있으면 `{ thumbnailUrl }`, 없으면 null. [친구](friends.md))) | FR-FRIEND-03 |
| DELETE | /api/friends/{userId} | 친구 끊기 | - | 204 | FR-FRIEND-03 |
| POST·GET | /api/blocks | 차단하기(회원이면 누구나. 친구를 차단하면 친구 관계도 끊어요) · 차단한 사람 목록 | userId, hideInRooms · - | 204 · blocks\[\] | FR-FRIEND-04 |
| PATCH·DELETE | /api/blocks/{userId} | 방에서 그 사람 글 숨기기 바꾸기(초안) · 차단 풀기 | hideInRooms · - | 204 | FR-FRIEND-04 |
| GET·POST | /api/rooms | 내 방 목록, 방 만들기 | RoomCreateRequest | 200·201 | FR-ROOM-01~06. [친구와 같이 쓰는 방](rooms.md) (초안) |
| GET·PATCH·DELETE | /api/rooms/{id} | 방 하나, (방장) 이름 바꾸기, (방장) 해산 | - · name · - | 200 · 200 · 204 | |
| DELETE | /api/rooms/{id}/members/me | 방 나가기. 방장이면 newOwnerId | newOwnerId | 204 | |
| DELETE | /api/rooms/{id}/members/{userId} | (방장) 내보내기 | - | 204 | |
| POST | /api/rooms/{id}/invites, DELETE /api/rooms/{id}/invites/{inviteId} | (방장) 초대 보내기 · 취소 | userIds | 201 · 204 | |
| GET | /api/room-invites | 받은 방 초대 | - | 200 | |
| POST·DELETE | /api/room-invites/{id}(/accept) | 초대 수락 · 거절 | - | 200 · 204 | |
| GET·POST | /api/rooms/{id}/posts | 올린 만화 목록, 내 만화 올리기 | `{ comicId }` | 200·201 | |
| DELETE | /api/rooms/{id}/posts/{postId} | 내 만화 내리기 | - | 204 | |
| PUT·DELETE | /api/rooms/{id}/posts/{postId}/like | 좋아요, 취소 | - | 200 | |
| GET·POST | /api/rooms/{id}/messages | 대화 보기(읽음 처리, 커서), 댓글 달기 | cursor · RoomCommentRequest | 200·201 | |
| GET | /api/inbox | 알림 목록·공지사항. 자세히는 [알림 목록과 공지사항](inbox.md) (초안) | tab, cursor | items\[\], nextCursor | FR-NOTI-04, 05 |
| GET | /api/inbox/unread | 종 버튼의 점 | - | hasUnread | FR-NOTI-04 |
| POST | /api/inbox/{kind}/{id}/read, /api/inbox/read | 하나 읽음, 모두 읽음 | - | 204 | FR-NOTI-04 |
| GET | /api/me/badges | 기능별 새 소식 있음·없음 | - | cards, comics, profileCharacter, friends, rooms, inbox | FR-NOTI-06 |
| POST | /api/me/badges/{kind}/read | 그 화면을 열었음 | - | 204 | FR-NOTI-06 |
| POST | /api/feedback | 오류 신고·의견 보내기. 하루 10번 넘으면 429 TOO\_MANY\_REQUESTS. 자세히는 [관리자](admin.md) (초안) | category, text, appVersion, deviceModel | 201 | FR-FEEDBACK-01 |
| POST·PATCH·DELETE | /api/admin/notices(/{id}) | (관리자) 공지사항 쓰기·고치기·지우기 | title, body | 201·200 Notice · 204 | FR-ADMIN-03 |
| GET | /api/admin/users/{id} | (관리자) 그 회원의 닉네임과 친구 코드 | - | nickname, friendCode | FR-ADMIN-04 |
| GET·PATCH | /api/admin/feedback(/{id}) | (관리자) 의견 목록 · 처리 표시 | status, cursor · status | items\[\], nextCursor · FeedbackItem | FR-ADMIN-05 |
| POST | /api/texts | 직접 쓴 글 올리기. 길이 상한을 넘으면 400 TEXT_TOO_LONG. 자세히는 [글로 만화·영상 만들기](texts.md) (초안) | title, body, place, situation | 202 textId, status | FR-TEXT-01 |
| GET | /api/texts/{id} | 글 상태와 원문(블로그 글이면 body 대신 link·blogTitle). status는 카드·작업으로 채움 | - | textId, source, status, cardId, title, body, place, situation, link, blogTitle, truncated, createdAt | FR-TEXT-01 |
| POST | /api/blogs/check | 링크만 받아 RSS 주소를 찾고 피드의 카테고리 목록 돌려주기(저장 안 함) | url | feedUrl, categories\[\] | FR-TEXT-02 |
| POST·PATCH | /api/blogs(/{id}) | 블로그 등록(같은 RSS 주소면 거절) · 가져올 카테고리 바꾸기 | url, categories\[\] · categories\[\] | 201·200 id, siteUrl, title, categories | FR-TEXT-02 |
| GET | /api/blogs | 연결한 블로그 목록(연결한 순서) | - | blogs\[\] (id, siteUrl, title, categories) | FR-TEXT-02 |
| DELETE | /api/blogs/{id} | 블로그 연결 끊기. 만든 카드는 남아요 | - | 204 | FR-TEXT-02 |
| GET | /api/blogs/{id}/posts | 가져온 글 목록(각각 바로 카드로 만들어짐). 최신 글부터 20개씩 | cursor | posts\[\], nextCursor | FR-TEXT-03 |
| GET | /api/comics | 내 만화 목록. sort로 최신순·오래된순을 골라요 | cursor, size, sort | comics\[\], nextCursor | FR-LIB-01 |
| GET | /api/comics/{id} | 만화 상태·결과 | - | id, cardId, status, imageUrl, createdAt (cardId는 카드를 지웠으면 null) | FR-COMIC-03 |
| DELETE | /api/comics/{id} | 만화 지우기 | - | - | FR-LIB-03 |
| POST | /api/cards/{id}/videos | 카드로 영상 요청. 409 VIDEO\_IN\_PROGRESS·VIDEO\_LIMIT. 자세히는 [카드 영상](video.md) (초안) | - | 202 videoId, status | FR-EXT-02 |
| GET | /api/videos | 내 영상 목록(만화 목록과 같은 모양) | cursor, size, sort | videos\[\], nextCursor | FR-EXT-02 |
| GET·DELETE | /api/videos/{id} | 영상 상태·결과 · 영상 지우기 | - | VideoResponse · 204 | FR-EXT-02 |
| PUT | /api/devices/fcm-token | 푸시 토큰 등록. 같은 token이 다른 사용자에게 있으면 지금 사용자로 옮겨요 | token | - | FR-NOTI |
| DELETE | /api/devices/fcm-token | 이 폰의 푸시 토큰 지우기(로그아웃할 때) | token | - | FR-AUTH-04 |

**응답 모양과 규칙**

- 같은 녹음이면 묶음이 몇 개든 같은 recordingId를 돌려줘요. 앱의 "분석 중" 화면은 이 recordingId로 상태를 봐요.
- 요청·응답 모양은 [API DTO](api-dto.md)의 2장(RecordingUploadRequest, CardSummary, CardDetail, ComicResponse 등)에 표로 있어요.
- PATCH /api/cards/{id}에서 카드에 없는 화자 키를 쓰면 400 UNKNOWN\_SPEAKER예요. 다음 번호의 key로 새 화자를 만들 때는 받아요. 화자 더하기·빼기 규칙은 [API DTO](api-dto.md)의 CardUpdateRequest에 있어요.
- **imageUrl:** 교육기관 MinIO(S3 호환)의 미리 서명한 주소이고 10분 뒤 만료돼요. 주소는 `https://lolog.java21.net/lolog-comics/...`이고 우리 Nginx가 MinIO로 넘겨요. 만료 시간은 응답에 넣지 않아요. 앱은 이미지를 못 받으면(403) GET /api/comics/{id}를 다시 불러 새 주소를 받고, 주소를 저장하지 않아요.
- 실패 이유는 앱에 보내지 않아요. 앱은 FAILED면 "만들지 못했어요"만 보여 줘요. 썸네일은 따로 없고 목록도 imageUrl을 써요.
- **푸시:** FCM data 메시지이고 값은 모두 문자열이에요. Android 우선순위는 high로 보내요(앱이 백그라운드·절전 중이어도 바로 받게). 아래 표는 카드·만화·영상 푸시만 적었어요. 프로필 캐릭터(`PROFILE_CHARACTER_READY`·`PROFILE_CHARACTER_FAILED`), 친구(`FRIEND_REQUEST`), 방(`ROOM_INVITE`·`ROOM_COMMENT`·`ROOM_DISBANDED`)까지 모든 type은 [API DTO](api-dto.md#3-푸시-lolog-api--앱) 3장에 있어요.

| type | 나머지 키 | 보내는 때 | 앱 동작 |
| --- | --- | --- | --- |
| CARDS_CREATED | recordingId, cardCount(이번 묶음에서 새로 생긴 카드 수) | 묶음마다 감지가 끝나고 새 카드가 1개 이상일 때 한 번 | 보관함 새로고침 |
| CARDS_CREATED (글) | textId, cardCount(1) | 직접 쓴 글의 카드가 생겼을 때. 블로그 글은 보내지 않아요([글로 만화·영상 만들기](texts.md), 초안) | 보관함 새로고침 |
| COMIC_DONE | comicId, cardId | 완성 이미지를 받았을 때 | 만화 화면 |
| COMIC_FAILED | comicId, cardId | 워커의 fail을 받았을 때 바로 | "만들지 못했어요" |
| VIDEO_DONE · VIDEO_FAILED | videoId, cardId | 완성 영상을 받았을 때 · 워커의 fail을 받았을 때 | [카드 영상](video.md#2-앱-api-api) 참고 |

- 푸시가 붙기 전에는 앱이 GET /api/recordings/{id}·GET /api/comics/{id}를 5초마다 부르고, DONE이나 FAILED가 되면 멈춰요.

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
  "id": 12, "recordingId": 3,                 // 녹음 기록이 지워졌으면 null
  "kind": "웃김", "summary": "...", "place": null,
  "recordedAt": "2026-11-10T12:30:00+09:00",   // 녹음 날짜
  "startMs": 1200, "endMs": 95000,
  "speakers": [ { "key": "화자1", "name": null } ],
  "lines": [ { "id": 1, "speaker": "화자1", "text": "..." } ],
  "comics": [ { "id": 31, "status": "DONE" }, { "id": 35, "status": "DRAWING" } ],
  "audio": { "status": "READY", "url": "https://...", "durationMs": 94800 },   // PENDING·NONE이면 url·durationMs는 null
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
| 늦게 온 묶음 | 다음 묶음이 1시간 10분 안에 안 오면 서버는 넘겨 둔 부분만 따로 처리해요(기능 명세 "묶음 경계" ⑦). 1시간 10분은 앞 묶음을 받은 때부터 재요. 이때도 녹음은 닫지 않아요(RECORDING\_CLOSED는 isLast 뒤에만). 그 뒤에 온 다음 묶음은 정상으로 받아 넘겨받을 것 없이 처리해요. 큐 메시지 모양은 아래 "큐 메시지"의 detect에 있어요 |
| 시간 값 | startedAt은 첫 묶음 값을 유지해요. durationSec는 0 이상 86,400 이하(하루)의 정수예요. startMs, endMs는 녹음 시작부터의 밀리초이고, 0 ≤ startMs < endMs예요 |
| sentences | 이름과 달리 문장이 아니라 말소리 구간(VAD) 단위예요. 앱이 20초가 넘는 구간을 강제로 잘라서, 한 구간은 최대 약 20초예요(FR-REC-03). 한 구간에 여러 문장이 들어 있고 문장부호가 거의 없어요. text는 비어 있으면 안 되고, 앱은 빈 인식 결과를 올리지 않아요 |
| 크기 제한 | 묶음당 5,000구간, 본문 1MB까지예요. 넘으면 413 PAYLOAD\_TOO\_LARGE예요 |
| 음성 저장 | audioSaved는 이 녹음에서 음성 저장을 켰는지예요(true·false). 없으면 false로 봐요(예전 앱). 녹음 중에는 못 바꾸니 묶음마다 같은 값을 보내요. 음성 파일은 따로 올려요([음성 저장](audio.md)) |
| 응답 | `202 { "recordingId": 3, "status": "RECEIVED" }`. status는 녹음의 상태예요 |
| 오류 | 형식이 틀리면 400 INVALID\_REQUEST, 토큰이 만료됐으면 401 TOKEN\_EXPIRED, 없거나 틀리면 401 UNAUTHORIZED예요. 앱은 네트워크 오류와 5xx만 다시 시도해요. TOKEN\_EXPIRED면 토큰을 갱신한 뒤 다시 시도하고, UNAUTHORIZED면 다시 로그인해요 |

**상태 값**

- 녹음: `RECEIVED` → `DETECTING` → `DONE` / `FAILED`
- 만화: `QUEUED` → `STORYBOARD` → `DRAWING` → `DONE` / `FAILED`

## 워커 전용 (/internal)

Jetson 워커가 SSH 터널로 부르는 주소예요. 사용자 데이터 지우기(`DELETE /internal/users/{id}`)와 정지 알림(`PUT`·`DELETE /internal/users/{id}/suspension`)은 인증 서비스가 불러요.

| 메서드 | 주소 | 하는 일 | 보내는 것 |
| --- | --- | --- | --- |
| POST | /internal/recordings/{id}/cards | 감지 결과(카드) 보내기 | cards\[\] (없으면 빈 배열), carryFromMs(다음 묶음으로 넘길 시작 시각, 마지막 묶음이면 null) |
| GET | /internal/comics/{id}/source | 만화 만들 카드 내용 받기 | - |
| POST | /internal/comics/{id}/storyboard | 콘티 보내기 | 콘티 JSON: characters\[\], panels\[4\] (장면, 인물 위치, 대사). 형식은 [콘티 형식](storyboard.md) |
| POST | /internal/comics/{id}/image | 완성 4컷 올리기 | PNG (multipart) |
| GET·POST | /internal/videos/{id}/source, storyboard, audio, video | 카드 영상 만들기(카드 내용 받기, 콘티·음성·완성 영상 주고받기). 자세히는 [카드 영상](video.md#3-워커-api-internal) (초안) | 영상 콘티 JSON, wav, mp4+썸네일 (multipart) |
| GET | /internal/users/{userId}/character/photo | 프로필 캐릭터 원본 사진 받기(avatar 작업) | - |
| POST | /internal/users/{userId}/character/avatar | 프로필 캐릭터 그림 보내기. 받으면 원본 사진 삭제 | PNG (multipart `file`) + `appearance` |
| POST | /internal/texts/{id}/cards | 글 결과(카드 1장) 보내기. 카드 모양은 DetectResult와 같고 startMs·endMs는 null([글로 만화·영상 만들기](texts.md), 제안) | jobId, cards\[1\] |
| POST | /internal/jobs/{id}/fail | 실패 알리기 | reason |
| DELETE | /internal/users/{id} | 탈퇴한 사용자의 데이터 모두 지우기. 인증 서비스가 불러요 | - |
| PUT·DELETE | /internal/users/{id}/suspension | 정지됐다·풀렸다고 알리기(방 멤버 목록의 "정지된 계정" 표시). 인증 서비스가 불러요(초안, [관리자](admin.md)) | PUT `{ suspendedUntil }`(영구면 null) · DELETE - |
| PUT | /internal/characters/defaults/{id} | 기본 화자 넣기·바꾸기(스크립트용). 자세히는 [화자 그림](characters.md#internal-워커--메인-백엔드) | multipart `image`, `thumbnail`, `voice`(모두 선택)와 목록 칸 |

- **인증:** 헤더 `X-Internal-Token: <비밀값>`을 붙여요. 값은 환경 변수 `INTERNAL_TOKEN`이고 메인 백엔드·워커·인증 서비스가 같은 값을 가져요. 틀리면 401이에요.
- **응답:** GET 주소는 200이에요. `GET /internal/comics/{id}/source`는 `ComicSource`, `GET /internal/users/{userId}/character/photo`는 사진 바이트, `GET /internal/videos/{id}/source`·`storyboard`·`audio`는 VideoSource·영상 콘티·wav예요([카드 영상](video.md#3-워커-api-internal)). GET이 아닌 것은 모두 204예요. 이미지는 multipart의 `file`(image/png)로 보내요.
- **본문 모양:** DetectResult, ComicSource, JobFailRequest는 [API DTO](api-dto.md)의 5장에 표로 있어요.

```json
// POST /internal/recordings/{id}/cards (DetectResult)
{
  "jobId": 7, "partNo": 1,
  "carryFromMs": 3300000,
  "cards": [
    {
      "kind": "웃김", "summary": "치킨 주문하다 생긴 일",
      "startMs": 1200, "endMs": 95000,
      "speakers": ["화자1", "화자2"],
      "lines": [
        { "lineNo": 1, "speaker": "화자1", "text": "양념 반 후라이드 반", "startMs": 1200, "endMs": 3400 },
        { "lineNo": 2, "speaker": "화자2", "text": "또 반반이야?", "startMs": null, "endMs": null }
      ]
    }
  ]
}

// GET /internal/comics/{id}/source (ComicSource)
{
  "comicId": 35, "cardId": 12, "kind": "웃김", "summary": "치킨 주문하다 생긴 일",
  "place": null,
  "speakers": [
    { "key": "화자1", "name": "민수",
      "character": { "ref": "default:young-m-01", "appearance": "a young man in his twenties, ...", "imageUrl": "https://..." } },
    { "key": "화자2", "name": null, "character": null }
  ],
  "lines": [ { "id": 101, "speaker": "화자1", "text": "양념 반 후라이드 반" } ]
}
```

- **만화 상태가 바뀌는 때:** source를 가져가면 STORYBOARD, storyboard를 받으면 DRAWING, image를 받으면 DONE과 푸시, fail을 받으면 FAILED와 푸시예요.

## 큐 메시지 (Redis Streams)

detect 메시지에는 대화 텍스트 묶음을 같이 넣어요. sentences\[\]에는 앞 묶음에서 넘겨받은 부분(carryFromMs 이후)을 앞에 붙여 넣어요(기능 명세의 "묶음 경계" 참고). 처리가 끝나면 워커가 완료 표시(XACK)를 하고 메시지를 지워요(XDEL). 꺼내 가도 스트림에 남기 때문에, 대화 글이 큐에 남지 않게 하려는 뜻이에요. comic 메시지는 ID만 넣고, 카드 내용은 워커가 `/internal`에서 받아 가요.

- **보내는 곳:** 스트림 이름은 `lolog:detect`, `lolog:comic`, `lolog:video`예요. 메시지 필드는 `payload` 하나이고 값은 아래 모양의 JSON 문자열이에요.
- **detect:** `{ jobId, recordingId, partNo, isLast, sentences[] }`
  - 앞 묶음의 carryFromMs를 알아야 하므로, 앞 묶음의 감지가 끝난 뒤에 다음 묶음을 큐에 넣어요. 앞 묶음이 실패했으면 넘겨받을 것 없이 넣어요.
  - 새 글도 넘겨받을 글도 없는 묶음은 큐에 넣지 않고 바로 끝난 것으로 처리해요.
  - 늦게 온 묶음(위 규칙 표): 넘겨 둔 부분만 처리할 때는 새 jobId로 `{ jobId, recordingId, partNo: 넘겨 둔 묶음의 번호, isLast: true, sentences: 넘겨 둔 부분 }`을 넣어요. 워커는 마지막 묶음처럼 남은 이야기를 모두 카드로 만들고 carryFromMs는 null로 보내요.
  - 감지가 실패한 묶음(fail)은 넘길 것이 없어요. 그 묶음과 앞에서 넘겨받은 글은 바로 지우고, 다음 묶음은 넘겨받을 것 없이 넣어요.
- **comic:** `{ jobId, comicId, cardId }`
- **글(같은 `lolog:detect` 스트림, 제안):** `{ jobId, kind: "text", textId, title, body, place?, situation? }`. kind가 없으면 녹음 detect예요. 워커는 이야기 감지 없이 화자 구분 → 카드 한 장을 만들어 `/internal/texts/{id}/cards`로 보내요. [글로 만화·영상 만들기](texts.md)
- **avatar(같은 `lolog:comic` 스트림):** `{ jobId, kind: "avatar", userId }`. kind가 없으면 comic이에요. [화자 그림](characters.md)
- **video:** 목소리 단계는 `lolog:video`에 `{ jobId, videoId, cardId }`, 그림 단계는 같은 `lolog:comic`에 `{ jobId, kind: "video", videoId, cardId }`예요. [카드 영상](video.md#4-큐-메시지)
- **컨슈머 그룹:** `jetson-workers`이고, 컨슈머 이름은 워커마다 `jetson-1`처럼 붙여요. 그룹은 워커가 켜질 때 `XGROUP CREATE ... MKSTREAM`으로 만들고, 이미 있으면 넘어가요.
- **다시 시도:** 워커가 죽어 XACK하지 못한 메시지는 워커가 다시 켜질 때 `XAUTOCLAIM`으로 10분 넘게 멈춘 것을 가져와 다시 처리해요. 같은 jobId가 3번 실패하면 fail로 알려요. 메인 백엔드는 같은 jobId의 결과가 두 번 오면 두 번째는 204로 받고 무시해요.
- **작업이 지워졌을 때:** 사용자가 만드는 중인 만화를 지우거나 탈퇴하면 작업도 지워져요. `/internal`이 404 `RECORDING_NOT_FOUND`·`COMIC_NOT_FOUND`·`CHARACTER_NOT_FOUND`·`VIDEO_NOT_FOUND`·`TEXT_NOT_FOUND`(글 결과 주소, [글로 만화·영상 만들기](texts.md))를 주면 워커는 다시 시도하지 않고 fail도 보내지 않고 그 메시지를 XACK·XDEL해요. fail이 400(작업 없음)이어도 XACK·XDEL해요.
- **같은 메시지가 두 번 올 수 있음:** 메인 백엔드는 DB에 작업을 저장한 뒤 큐에 넣고, 넣지 못한 작업은 1분 뒤 다시 넣어요. 그래서 드물게 같은 jobId 메시지가 두 번 들어와요. 워커는 jobId로 이미 처리한 메시지를 건너뛰어도 되고, 그냥 처리해서 결과를 두 번 보내도 돼요(두 번째 결과는 무시돼요).

## 담당과 정할 것

- **컷 배치:** 세로 4칸으로 정했어요. 컷을 하나씩 그려 세로로 합쳐요. 사용자가 고르는 옵션은 없어요.
- **다시 만들기:** 같은 주소(POST /api/cards/{id}/comics)를 다시 부르면 새 만화가 하나 더 생겨요. 한 카드에 3번까지이고(실패한 만화는 세지 않고, 만화를 지워도 횟수는 돌아오지 않아요), 넘으면 메인 백엔드가 409 COMIC\_LIMIT로 막아요. 그 카드에 만드는 중인 만화가 있으면 409 COMIC\_IN\_PROGRESS로 막아요. 둘 다 해당하면(세 번째 만화를 만드는 중) COMIC\_IN\_PROGRESS예요. 그 만화가 실패하면 다시 만들 수 있기 때문이에요.
- **콘티 모양:** [콘티 형식](storyboard.md)으로 정했어요. 스키마는 `spec/storyboard.schema.json`이에요.
- **추가 구현(P1·P2):** 카드 구간 조정, 피드 업로드 API는 그때 추가해요. 녹음 파일(음성) 업로드는 [음성 저장](audio.md)에, 카드 구간 재생은 [순간 다시 듣기](replay.md)에 정했어요. 비밀번호 찾기·바꾸기(FR-AUTH-07)는 인증 서비스 표에 있어요. 공유는 P0이고 GET /api/comics/{id}의 imageUrl로 해요.
