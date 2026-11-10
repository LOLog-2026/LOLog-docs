# API DTO

[API 명세](api-spec.md)의 요청·응답 본문을 필드 단위로 정해요. 앱(lolog-app), 인증(lolog-auth), 메인 백엔드(lolog-api), Jetson 워커가 이 모양 그대로 주고받아요. 이름은 Java·Kotlin 클래스 이름으로 써도 되게 붙였어요.

## 읽는 법

- **타입:** `Long`(64비트 정수, ID), `Int`(32비트 정수), `Boolean`, `String`, `DateTime`(ISO-8601 문자열, 시간대 포함. 예: `2026-11-10T12:30:00+09:00`), `List<X>`.
- **필수:** O는 늘 있어야 하고 null도 안 돼요. `null 가능`이라고 적힌 필드는 키는 늘 있고 값이 null일 수 있어요. 요청에서 `선택`인 필드는 키를 빼도 돼요.
- **JSON:** camelCase를 써요. 응답에 모르는 필드가 와도 받는 쪽은 무시해요(앱은 `ignoreUnknownKeys`, 서버는 `FAIL_ON_UNKNOWN_PROPERTIES=false`). 요청에 모르는 필드가 와도 서버는 무시해요.
- **시간:** `startMs`·`endMs`·`carryFromMs`는 녹음 시작부터 잰 밀리초(`Long`)예요.
- **열거 값:** 대문자 문자열이에요. 단, `kind`는 한국어 값 그대로예요.

## API별 DTO 찾기

API 하나마다 어떤 DTO를 쓰는지예요. 여러 API가 같은 DTO를 나눠 써서 DTO 수가 API 수보다 적어요. "-"는 본문이 없다는 뜻이에요.

| 메서드 | 주소 | 요청 DTO | 응답 DTO |
| --- | --- | --- | --- |
| GET | /auth/health, /api/health | - | 200 `{ status }` (0장) |
| POST | /auth/email/code | EmailCodeRequest | 204 - |
| POST | /auth/signup | SignupRequest | 201 TokenResponse |
| POST | /auth/login | LoginRequest | 200 TokenResponse |
| POST | /auth/social/{provider} | SocialLoginRequest | 200 TokenResponse(+isNewUser) |
| POST | /auth/refresh | RefreshRequest | 200 TokenResponse |
| POST | /auth/logout | LogoutRequest | 204 - |
| GET | /auth/me | - | 200 MeResponse |
| POST | /auth/me/social/{provider} | SocialLoginRequest | 200 `{ linkedProviders }` (MeResponse의 한 필드) |
| DELETE | /auth/me/social/{provider} | - | 200 `{ linkedProviders }` |
| DELETE | /auth/me | - | 204 - |
| POST | /auth/password/reset/code | PasswordResetCodeRequest | 204 - |
| POST | /auth/password/reset | PasswordResetRequest | 204 - |
| POST | /auth/me/password | PasswordChangeRequest | 200 TokenResponse |
| POST | /api/recordings | RecordingUploadRequest | 202 RecordingUploadResponse |
| GET | /api/recordings | 쿼리 cursor, size, status | 200 PageResponse\<RecordingListItem\> |
| GET | /api/recordings/{id} | - | 200 RecordingResponse |
| GET | /api/cards | 쿼리 cursor, size, sort | 200 PageResponse\<CardSummary\> |
| GET | /api/cards/{id} | - | 200 CardDetail |
| PATCH | /api/cards/{id} | CardUpdateRequest | 200 CardDetail |
| DELETE | /api/cards/{id} | - | 204 - |
| POST | /api/cards/{id}/comics | - | 202 ComicCreateResponse |
| GET | /api/characters/defaults | - | 200 `{ characters }` ([화자 그림](characters.md#4-api)) |
| GET | /api/me/profile | - | 200 ProfileResponse ([친구](friends.md#2-api)) |
| PATCH | /api/me/profile | ProfileUpdateRequest | 200 ProfileResponse |
| PUT | /api/me/profile/character | multipart `photo` | 202 ProfileCharacterResponse ([화자 그림](characters.md#4-api)) |
| DELETE | /api/me/profile/character | - | 204 - |
| POST | /api/friend-requests | FriendRequestCreate | 201 FriendRequestResponse |
| GET | /api/friend-requests | 쿼리 direction | 200 `{ requests }` |
| POST | /api/friend-requests/{id}/accept | - | 200 FriendItem |
| DELETE | /api/friend-requests/{id} | - | 204 - |
| GET | /api/friends | - | 200 `{ friends }` |
| DELETE | /api/friends/{userId} | - | 204 - |
| POST | /api/blocks | `{ userId }` | 204 - |
| GET | /api/blocks | - | 200 `{ blocks }` |
| DELETE | /api/blocks/{userId} | - | 204 - |
| GET | /api/comics | 쿼리 cursor, size, sort | 200 PageResponse\<ComicResponse\> |
| GET | /api/comics/{id} | - | 200 ComicResponse |
| DELETE | /api/comics/{id} | - | 204 - |
| PUT | /api/devices/fcm-token | FcmTokenRequest | 204 - |
| DELETE | /api/devices/fcm-token | FcmTokenRequest | 204 - |
| (푸시) | FCM data | - | 3장 표 |
| (큐) | lolog:detect | DetectMessage | - |
| (큐) | lolog:comic | ComicMessage, AvatarMessage([화자 그림](characters.md#큐-메인-백엔드--워커)) | - |
| POST | /internal/recordings/{id}/cards | DetectResult | 204 - |
| GET | /internal/comics/{id}/source | - | 200 ComicSource |
| POST | /internal/comics/{id}/storyboard | 콘티 JSON(storyboard.md v1) | 204 - |
| POST | /internal/comics/{id}/image | multipart `file` | 204 - |
| GET | /internal/users/{userId}/character/photo | - | 200 사진 바이트 ([화자 그림](characters.md#internal-워커--메인-백엔드)) |
| POST | /internal/users/{userId}/character/avatar | multipart `file`, `appearance` | 204 - |
| POST | /internal/jobs/{id}/fail | JobFailRequest | 204 - |
| DELETE | /internal/users/{id} | - | 204 - |

오류는 모든 API가 ErrorResponse 하나로 돌려줘요.

## 공통

### ErrorResponse

모든 4xx·5xx 응답의 본문이에요.

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| code | String | O | 오류 코드. 아래 표 |
| message | String | O | 개발자용 설명. 앱 화면에 그대로 띄우지 않아요 |
| field | String | null 가능 | 400 INVALID\_REQUEST일 때 틀린 칸의 경로(예: `lines[2].text`). 다른 오류는 null |

### 오류 코드

| 코드 | HTTP | 어디서 |
| --- | --- | --- |
| INVALID_REQUEST | 400 | 모든 API. 필드가 빠졌거나 타입·제한이 틀림 |
| INVALID_CODE | 400 | /auth/signup, /auth/password/reset. 인증번호가 틀림(5번까지) |
| CODE_EXPIRED | 400 | /auth/signup, /auth/password/reset. 인증번호가 없거나, 만료됐거나, 5번 틀려서 못 씀. 앱은 번호를 다시 받게 해요 |
| UNKNOWN_SPEAKER | 400 | PATCH /api/cards/{id}. 카드에 없는 화자 키 |
| UNKNOWN_CHARACTER | 400 | PATCH /api/cards/{id}. 없는 기본 화자 id, 나도 친구도 아닌 회원 ID |
| DUPLICATE_CHARACTER | 400 | PATCH /api/cards/{id}. 한 카드에서 같은 그림을 두 화자에 붙임 |
| CHARACTER_NOT_READY | 409 | PATCH /api/cards/{id}. 쓸 수 있는 그림이 없는 프로필 캐릭터 |
| CHARACTER_IN_PROGRESS | 409 | PUT /api/me/profile/character. 만드는 중에 사진을 또 올림 |
| CHARACTER_NOT_FOUND | 404 | /internal/users/{userId}/character/…. 받을 사진이 없음(이미 지워짐, 회원 없음). 워커는 그 메시지를 버림([화자 그림](characters.md)) |
| CANNOT_FRIEND_SELF | 400 | 친구 요청에 내 코드를 넣음, 나를 차단하려 함([친구](friends.md#오류-코드)) |
| FRIEND_CODE_NOT_FOUND | 404 | POST /api/friend-requests. 없는 친구 코드, 상대가 나를 차단함 |
| FRIEND_REQUEST_NOT_FOUND | 404 | /api/friend-requests/{id}. 없거나 내 요청이 아님, 보낸 사람이 수락하려 함 |
| USER_NOT_FOUND | 404 | POST /api/blocks. 친구도 아니고 요청도 오간 적 없는 사람 |
| PROFILE_REQUIRED | 409 | 친구 요청 보내기·수락. 닉네임을 아직 안 정함 |
| ALREADY_FRIENDS | 409 | 친구 요청. 이미 친구 |
| FRIEND_REQUEST_EXISTS | 409 | 친구 요청. 같은 사람에게 이미 보낸 요청이 있음 |
| FRIEND_LIMIT | 409 | 친구 요청 보내기·수락. 나나 상대의 친구가 이미 100명 |
| USER_BLOCKED | 409 | 친구 요청. 내가 차단한 사람 |
| INVALID_CREDENTIALS | 401 | /auth/login. 이메일이나 비밀번호가 틀림. /auth/me/password. 지금 비밀번호가 틀림(앱은 로그아웃하지 않아요) |
| TOKEN_EXPIRED | 401 | 액세스·리프레시 토큰 만료. /api도 만료된 액세스 토큰이면 이 코드(앱은 이때만 갱신) |
| INVALID_TOKEN | 401 | 토큰이 틀렸거나 서명이 안 맞음. 로그인이 필요한 /auth 주소(/auth/me 등)에 액세스 토큰이 없거나 틀릴 때, 소셜 로그인·연결에서 그 회사가 providerToken을 확인해 주지 않을 때, 소셜 로그인에서 그 회사가 이메일을 주지 않았거나 구글이 인증하지 않은 이메일(`email_verified`가 false)일 때도 이 코드 |
| UNAUTHORIZED | 401 | /api에 토큰이 없거나 틀림(만료 말고), /internal에 내부 토큰이 없거나 틀림 |
| CARD_NOT_FOUND | 404 | 카드가 없거나 내 카드가 아님 |
| COMIC_NOT_FOUND | 404 | 만화가 없거나 내 만화가 아님. /internal은 만화가 없거나 이미 끝남(source) |
| RECORDING_NOT_FOUND | 404 | 녹음이 없거나 내 녹음이 아님 |
| ACCOUNT_EXISTS | 409 | 가입, 소셜 로그인 |
| SOCIAL_ALREADY_LINKED | 409 | 소셜 계정 연결. 그 소셜 계정이 다른 계정에 붙어 있거나, 내 계정에 같은 회사의 다른 계정이 이미 붙어 있음 |
| LAST_LOGIN_METHOD | 409 | 소셜 계정 연결 끊기. 비밀번호가 없는 계정의 마지막 소셜 연결이라 끊으면 다시 로그인할 수 없음 |
| PART_OUT_OF_ORDER | 409 | 녹음 올리기 |
| RECORDING_CLOSED | 409 | 녹음 올리기 |
| CARD_LOCKED | 409 | 카드 고치기·지우기 |
| COMIC_IN_PROGRESS | 409 | 만화 요청. COMIC_LIMIT과 둘 다 해당하면 이 코드 |
| COMIC_LIMIT | 409 | 만화 요청. 성공·진행 중인 만화가 3번. 지운 만화도 셈 |
| PAYLOAD_TOO_LARGE | 413 | 녹음 올리기, 완성 이미지 올리기(/internal) |
| TOO_MANY_REQUESTS | 429 | /auth/email/code, /auth/password/reset/code. 같은 이메일로 60초 안에 다시 부름. POST /api/friend-requests. 1분에 10번 넘게 부름 |
| INTERNAL_ERROR | 500 | 서버 오류 |

남의 카드·만화·녹음은 403이 아니라 404로 답해요. 있는지 없는지 알리지 않으려는 뜻이에요.

### PageResponse

목록 응답은 `{ "<목록 이름>": [...], "nextCursor": ... }` 모양이에요.

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| cards / comics / recordings | List | O | 기본은 최신순(id 내림차순, createdAt 순서와 같음). cards·comics는 `sort=oldest`면 오래된순(id 오름차순). 없으면 빈 배열 |
| nextCursor | String | null 가능 | 다음 페이지를 부를 때 `?cursor=`에 넣는 값. 마지막 페이지면 null. 앱은 값의 뜻을 해석하지 않아요 |

요청 쿼리: `cursor`(String, 선택), `size`(Int, 선택, 기본 20, 1~50).

`GET /api/cards`, `GET /api/comics`는 `sort`(String, 선택)도 받아요.

| sort | 순서 |
| --- | --- |
| `latest` (빼면 이것) | 최신순(id 내림차순) |
| `oldest` | 오래된순(id 오름차순) |

- 다음 페이지를 부를 때도 처음과 같은 `sort`를 보내요. cursor는 마지막으로 받은 항목의 ID라서 sort가 다르면 순서가 섞여요.
- 정렬을 바꾸면 cursor 없이 처음부터 다시 불러요.
- 다른 값이면 400 `INVALID_REQUEST`, `field`는 `sort`예요.

## 0. 서버 상태 확인

`GET /auth/health`, `GET /api/health`. 토큰 없이 불러요. 200 `{ "status": "UP" }`. DB·Redis에 붙지 못하면 503 `{ "status": "DOWN" }`이에요.

## 1. lolog-auth (/auth)

### EmailCodeRequest

`POST /auth/email/code`

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| email | String | O | 이메일 형식, 254자 이하. 서버가 소문자로 바꿔 다뤄요 |

### SignupRequest · LoginRequest

`POST /auth/signup`, `POST /auth/login`

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| email | String | O | 이메일 형식, 254자 이하. 서버가 소문자로 바꿔 저장 |
| password | String | O | 8~64자, 영문·숫자 포함, UTF-8로 72바이트 이하(BCrypt 한도. 한글은 한 글자 3바이트라 64자 안이어도 넘을 수 있어요) (가입 때만 검사, 로그인은 검사 안 함) |
| code | String | 가입 때만 O | 메일로 받은 6자리 숫자(`"048213"`처럼 앞의 0도 그대로). 로그인에는 없어요 |

### PasswordResetCodeRequest

`POST /auth/password/reset/code`

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| email | String | O | 이메일 형식, 254자 이하. 서버가 소문자로 바꿔 다뤄요 |

### PasswordResetRequest

`POST /auth/password/reset` 204

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| email | String | O | 이메일 형식, 254자 이하 |
| code | String | O | 메일로 받은 6자리 숫자(앞의 0도 그대로) |
| newPassword | String | O | 가입의 password와 같은 규칙(8~64자, 영문·숫자 포함, UTF-8로 72바이트 이하) |

### PasswordChangeRequest

`POST /auth/me/password` 200 TokenResponse(새 토큰 한 벌. 앱은 저장한 토큰을 바꿔요)

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| currentPassword | String | O | 지금 비밀번호. 규칙은 검사하지 않아요 |
| newPassword | String | O | 가입의 password와 같은 규칙 |

### SocialLoginRequest

`POST /auth/social/{provider}`, `POST /auth/me/social/{provider}`. provider는 주소에서 소문자 `google`, `naver`.

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| providerToken | String | O | 앱이 각 회사 SDK로 받은 토큰. 네이버는 액세스 토큰, 구글은 ID 토큰(서버가 웹 클라이언트 ID로 `aud`를 확인) |

- provider가 `google`, `naver`가 아니면(대문자, `kakao` 포함) 400 `INVALID_REQUEST`, field는 `provider`예요.
- 계정은 그 회사의 사용자 ID(구글 `sub`, 네이버 `id`)로 찾아요. 이메일은 처음 가입할 때 계정 이메일로 저장하고, 같은 이메일로 가입된 계정이 있는지 볼 때만 써요. 그 회사에서 이메일을 바꿔도 같은 계정으로 들어가요.
- 네이버가 주는 이메일은 사용자가 적어 둔 연락처 이메일이라 계정마다 고유하지 않아요.
- 계정 연결은 이메일을 쓰지 않아서, 그 회사가 이메일을 주지 않아도 연결돼요.
- 그 회사 서버가 응답하지 않거나 오류면 500 `INTERNAL_ERROR`예요(토큰이 틀린 것과 구분해요).

### RefreshRequest · LogoutRequest

`POST /auth/refresh`, `POST /auth/logout`

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| refreshToken | String | O | 로그인 때 받은 리프레시 토큰 |

### TokenResponse

가입 201, 로그인·갱신 200. 소셜 로그인은 `isNewUser`가 더 붙어요.

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| accessToken | String | O | JWT. 아래 "토큰" |
| refreshToken | String | O | 무작위 문자열. 갱신하면 새 값으로 바뀌고 옛 값은 못 써요 |
| accessTokenExpiresIn | Int | O | 액세스 토큰 남은 시간(초). 지금은 1800 |
| isNewUser | Boolean | 소셜만 O | 이번에 처음 가입했으면 true |

### MeResponse

`GET /auth/me` 200

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| email | String | O | |
| linkedProviders | List\<String\> | O | `GOOGLE`, `NAVER` 중 연결된 것. 없으면 빈 배열 |

`POST`·`DELETE /auth/me/social/{provider}` 200은 `{ linkedProviders }`만 돌려줘요. 끊은 뒤에도 남은 연결 목록이에요. 로그아웃·탈퇴는 204, 본문 없음. 로그아웃은 없거나 만료됐거나 이미 쓴 refreshToken이어도 204예요(이미 로그아웃된 것과 같아요).

### 토큰

- 서명 HS256. 비밀값은 환경 변수 `JWT_SECRET`(32바이트 이상)이고 lolog-auth와 lolog-api가 같은 값을 가져요.
- 액세스 토큰 내용(claims):

| 이름 | 타입 | 설명 |
| --- | --- | --- |
| sub | String | 사용자 ID(숫자를 문자열로, 예: `"1"`) |
| typ | String | `"access"` |
| iat | Long | 만든 시각(초) |
| exp | Long | 만료 시각(초). iat + 1800 |

- 리프레시 토큰은 JWT가 아니고 14일 동안 써요. lolog-auth가 Redis에 해시로 둬요.

## 2. lolog-api (/api)

모든 요청에 `Authorization: Bearer <accessToken>`. 서버는 토큰의 `sub`로 사용자를 정해요. 토큰이 만료됐으면 401 TOKEN_EXPIRED, 없거나 틀리면 401 UNAUTHORIZED예요.

### 2.1 녹음

#### RecordingUploadRequest

`POST /api/recordings`. 규칙은 API 명세 "POST /api/recordings 규칙".

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| clientRecordingId | String | O | 앱이 만든 UUID(36자). 녹음 하나에 하나 |
| partNo | Int | O | 1부터 빠짐없이 |
| isLast | Boolean | O | 녹음이 끝난 묶음이면 true |
| startedAt | DateTime | O | 녹음 전체 시작 시각. 묶음마다 같은 값 |
| durationSec | Int | O | 이 묶음이 덮는 시간(초), 0 이상 86,400 이하 |
| sentences | List\<Sentence\> | O | 말이 없었으면 빈 배열. 5,000개를 넘거나 본문이 1MB를 넘으면 413 PAYLOAD_TOO_LARGE |

**Sentence**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| startMs | Long | O | 0 이상 |
| endMs | Long | O | startMs보다 큼 |
| text | String | O | 1자 이상 |

#### RecordingUploadResponse

202

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| recordingId | Long | O | 서버 녹음 ID. 묶음이 몇 개든 같은 녹음이면 같은 값. "분석 중" 화면은 이 값을 씀 |
| status | String | O | 녹음 상태 `RECEIVED`·`DETECTING`·`DONE`·`FAILED` |

#### RecordingListItem

`GET /api/recordings` 목록의 한 항목. 쿼리 `status`(선택)는 녹음 상태를 쉼표로 이어 써요(예: `RECEIVED,DETECTING`). 빼면 전부예요. 최신순(id 내림차순)이에요.

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| recordingId | Long | O | |
| clientRecordingId | String | O | 앱이 만든 ID. 앱이 폰에 남은 녹음과 맞춰 볼 때 씀 |
| status | String | O | 녹음 상태 |
| startedAt | DateTime | O | 녹음 시작 시각 |
| cardCount | Int | O | 지금까지 나온 카드 수 |

응답은 `{ "recordings": [...], "nextCursor": ... }`예요.

#### RecordingResponse

`GET /api/recordings/{id}` 200

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| recordingId | Long | O | |
| status | String | O | 위와 같음. 마지막 묶음까지 감지가 끝나면 DONE. 감지한 묶음이 모두 실패했으면 FAILED(하나라도 성공했으면 DONE). 그 전에는 다음 묶음을 기다리는 동안에도 DETECTING |
| cardCount | Int | O | 지금까지 이 녹음에서 나온 카드 수 |

### 2.2 카드

#### CardSummary

`GET /api/cards` 목록의 한 항목

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | |
| kind | String | O | `웃김`·`황당`·`추억` |
| summary | String | O | 제목으로 쓰는 한 문장 요약, 30자 이하 |
| recordedAt | DateTime | O | 녹음 시작 시각 |
| startMs | Long | O | 녹음 안에서 이야기가 시작한 때 |
| endMs | Long | O | 끝난 때 |
| place | String | null 가능 | 사용자가 적은 장소 |
| lineCount | Int | O | 대사 수 |
| comicCount | Int | O | 이 카드로 만든 만화 중 지금 남아 있는 수(실패한 것과 지운 것 빼고, 만드는 중 포함). 만들기 횟수는 CardDetail의 comicRemaining으로 봐요 |
| createdAt | DateTime | O | 카드가 만들어진 시각 |

#### CardDetail

`GET /api/cards/{id}` 200, `PATCH /api/cards/{id}` 200

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | |
| recordingId | Long | null 가능 | 녹음 기록이 지워졌으면 null |
| kind | String | O | |
| summary | String | O | |
| place | String | null 가능 | 사용자가 적은 장소 |
| recordedAt | DateTime | O | |
| startMs | Long | O | |
| endMs | Long | O | |
| speakers | List\<Speaker\> | O | 1~4개, key 순서(화자1, 화자2…) |
| lines | List\<Line\> | O | 1개 이상, 대사 순서 |
| comics | List\<ComicRef\> | O | 이 카드로 만든 만화, 최신순. 없으면 빈 배열 |
| comicRemaining | Int | O | 이 카드로 더 만들 수 있는 횟수(0~3). 3 − 실패 아닌 만화 수(지운 만화도 셈) |
| createdAt | DateTime | O | |

**Speaker**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| key | String | O | `화자1`, `화자2`… 바뀌지 않는 키 |
| name | String | null 가능 | 사용자가 붙인 이름. 카드가 생길 때는 null이고, null이면 앱은 key를 보여 줌. 예: 처음 `{ "key": "화자1", "name": null }` → "화자1", 고친 뒤 `{ "key": "화자1", "name": "민수" }` → "민수" |
| character | Object | null 가능 | 이 화자에 붙인 그림(기본 화자·프로필 캐릭터). 없으면 null. `{ type, id, label, thumbnailUrl }`, [화자 그림](characters.md#4-api) |

**Line**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | 대사 ID |
| speaker | String | O | speakers의 key 중 하나 |
| text | String | O | |

**ComicRef**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | 만화 ID |
| status | String | O | 만화 상태 |

#### CardUpdateRequest

`PATCH /api/cards/{id}`. 바꾼 것만 보내요. 셋 다 빠지면 400 INVALID_REQUEST. 빈 배열은 빠진 것과 같아요. speakers에 같은 key, lines에 같은 id가 두 번 오면 400 INVALID_REQUEST예요.

화자 이름·장소·대사는 서버가 앞뒤 공백을 빼고 검사해요. 빼고 0자면(빈 값, 띄어쓰기만) 400 INVALID_REQUEST예요. 이름·장소를 지우려면 null을 보내요.

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| speakers | List\<SpeakerUpdate\> | 선택 | 바꾼 화자만 |
| lines | List\<LineUpdate\> | 선택 | 바꾼 대사만 |
| place | String | 선택, null 가능 | 20자 이하. null을 보내면 장소를 지움 |

**SpeakerUpdate**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| key | String | O | 카드에 있는 key. 없으면 400 UNKNOWN_SPEAKER |
| name | String | 선택, null 가능 | 1~10자. null이면 이름을 지움 |
| character | CharacterRef | 선택, null 가능 | `{ type: DEFAULT·PROFILE, id }`. null이면 그림을 뗌. name과 character 중 하나는 있어야 해요(둘 다 빠지면 400 INVALID_REQUEST). [화자 그림](characters.md#4-api) |

**LineUpdate**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| id | Long | O | 이 카드의 대사 ID. 아니면 400 INVALID_REQUEST |
| speaker | String | 선택 | 카드에 있는 key. 없으면 400 UNKNOWN_SPEAKER |
| text | String | 선택 | 1~200자. 고쳐 보내는 대사에만 검사하고, 고치지 않은 대사(LLM이 만든 원래 대사)는 200자를 넘어도 그대로 둠 |

화자를 새로 만들거나 대사를 더하거나 빼는 건 이 API로 하지 않아요.

### 2.3 만화

#### ComicCreateResponse

`POST /api/cards/{id}/comics` 202. 요청 본문 없음.

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| comicId | Long | O | |
| status | String | O | 처음엔 `QUEUED` |

#### ComicResponse

`GET /api/comics/{id}` 200, `GET /api/comics` 목록의 한 항목

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | |
| cardId | Long | null 가능 | 카드를 지웠으면 null |
| cardSummary | String | O | 만화를 요청할 때 카드의 summary를 복사해 둔 값. 카드를 지워도 남아요 |
| status | String | O | `QUEUED`·`STORYBOARD`·`DRAWING`·`DONE`·`FAILED` |
| imageUrl | String | null 가능 | DONE일 때만 값이 있음. 세로 4칸 PNG 한 장, 800×3152px([storyboard.md](storyboard.md#정한-것)). 10분 동안 열리는 서명 주소라 저장하지 말고, 이미지를 못 받으면(403) 다시 조회 |
| createdAt | DateTime | O | 만화를 요청한 시각 |

실패 이유는 앱에 보내지 않아요. 썸네일은 따로 없어요.

#### 만화 지우기

`DELETE /api/comics/{id}` → 204. 만화와 이미지를 바로 지워요. 남의 만화나 없는 만화는 404 COMIC\_NOT\_FOUND예요. 만드는 중(QUEUED·STORYBOARD·DRAWING)인 만화도 지울 수 있고, 그 작업도 함께 지워져 워커의 다음 요청은 404가 돼요. 지운 만화도 만들기 횟수에 세요(comicRemaining은 늘지 않아요).

### 2.4 푸시 토큰

#### FcmTokenRequest

`PUT /api/devices/fcm-token`, `DELETE /api/devices/fcm-token`. 둘 다 204.

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| token | String | O | FCM 등록 토큰, 512자 이하(DB device_tokens.token) |

PUT에서 같은 token이 다른 사용자에게 등록돼 있으면 지금 사용자로 옮겨요(같은 폰에서 계정을 바꾼 경우). 같은 사용자가 같은 token을 다시 보내도 204예요(앱은 로그인할 때와 토큰이 바뀔 때마다 보내요). DELETE는 지금 사용자의 token일 때만 지우고, 없거나 다른 사용자의 token이면 아무것도 하지 않고 204예요.

## 3. 푸시 (lolog-api → 앱)

FCM **data** 메시지예요. FCM data는 값이 모두 문자열이라 숫자도 문자열로 와요. Android 우선순위(`android.priority`)는 `high`로 보내요. data 메시지라 알림은 앱이 직접 띄우는데, normal이면 절전(Doze) 중에 늦게 받을 수 있어서예요. collapse_key와 TTL은 정하지 않고 FCM 기본값을 써요(푸시가 늦거나 빠져도 앱은 조회로 상태를 알아요). 서버는 사용자의 모든 폰에 한 번씩만 보내고, 보내기가 실패하면 다시 보내지 않아요. FCM이 토큰이 무효라고 답하면(UNREGISTERED 등) 그 토큰을 지워요.

| type | 나머지 키 | 보내는 때 | 앱 동작 |
| --- | --- | --- | --- |
| CARDS_CREATED | recordingId, cardCount(이번 묶음에서 새로 생긴 카드 수) | 묶음마다 감지가 끝나고 새 카드가 1개 이상일 때 한 번 | 보관함 새로고침 |
| COMIC_DONE | comicId, cardId | 완성 이미지를 받았을 때 | 만화 화면 |
| COMIC_FAILED | comicId, cardId | 워커의 fail을 받았을 때 바로 | "만들지 못했어요" |
| FRIEND_REQUEST | requestId | 친구 요청을 받았을 때([친구](friends.md#3-푸시)) | 친구 화면의 받은 요청 |
| PROFILE_CHARACTER_READY | - | 내 프로필 캐릭터 그림을 받았을 때([화자 그림](characters.md#푸시)) | 프로필 새로고침 |
| PROFILE_CHARACTER_FAILED | - | 내 프로필 캐릭터 avatar 작업 fail을 받았을 때 | "캐릭터를 만들지 못했어요" |

예: `{ "type": "COMIC_DONE", "comicId": "35", "cardId": "12" }`. 카드를 지웠으면 cardId는 빈 문자열이에요. 푸시가 붙기 전에는 앱이 5초마다 조회해요.

## 4. 큐 메시지 (lolog-api → Jetson 워커)

Redis Streams. 스트림 `lolog:detect`·`lolog:comic`, 메시지 필드 `payload` 하나, 값은 아래 JSON 문자열. 컨슈머 그룹 `jetson-workers`.

### DetectMessage

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| jobId | Long | O | |
| recordingId | Long | O | |
| partNo | Int | O | |
| isLast | Boolean | O | |
| sentences | List\<Sentence\> | O | 앞 묶음에서 넘겨받은 부분 + 이 묶음. 1개 이상. startMs 순으로 정렬돼 있고 구간끼리 겹치지 않아요 |

### ComicMessage

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| jobId | Long | O | |
| comicId | Long | O | |
| cardId | Long | O | |

## 5. /internal (워커·인증 → lolog-api)

헤더 `X-Internal-Token: <INTERNAL_TOKEN 값>`. 없거나 틀리면 401 UNAUTHORIZED.

### DetectResult

`POST /internal/recordings/{id}/cards` → 204

같은 녹음에 이미 있는 카드와 시간 범위(startMs~endMs)가 겹치는 카드는 저장하지 않아요(기능 명세 "묶음 경계" ③). 앞 묶음에서 넘긴 글에 이미 카드가 된 이야기가 섞여 다시 감지된 경우예요. 버린 카드는 새 카드 알림의 cardCount에 세지 않아요.

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| jobId | Long | O | 받은 메시지의 jobId. 이미 결과나 fail을 받은 jobId면 204로 받고 무시 |
| partNo | Int | O | |
| cards | List\<DetectedCard\> | O | 없으면 빈 배열 |
| carryFromMs | Long | null 가능 | 다음 묶음으로 넘길 시작 시각. 마지막 묶음(isLast: true로 받은 메시지)이면 null. 값을 보내도 서버는 무시해요 |

주소의 녹음이 없으면 404 RECORDING\_NOT\_FOUND, jobId가 없거나 이 녹음의 감지 작업이 아니거나 partNo가 그 작업의 묶음과 다르면 400 INVALID\_REQUEST예요.

**DetectedCard**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| kind | String | O | `웃김`·`황당`·`추억` |
| summary | String | O | 1~30자. 제목으로 쓰는 한 문장 |
| startMs | Long | O | |
| endMs | Long | O | startMs보다 큼 |
| speakers | List\<String\> | O | `화자1`부터 빠짐없이, 1~4개 |
| lines | List\<DetectedLine\> | O | 1개 이상 |

**DetectedLine**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| lineNo | Int | O | 1부터 대사 순서 |
| speaker | String | O | speakers 중 하나 |
| text | String | O | |
| startMs | Long | null 가능 | 모르면 null |
| endMs | Long | null 가능 | 모르면 null |

### ComicSource

`GET /internal/comics/{id}/source` → 200. 부르면 만화가 STORYBOARD 상태가 돼요. 만화가 없거나 이미 끝났으면(DONE·FAILED) 404 COMIC\_NOT\_FOUND예요. 워커는 그 작업을 멈춰요.

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| comicId | Long | O | |
| cardId | Long | O | |
| kind | String | O | |
| summary | String | O | |
| place | String | null 가능 | null이면 LLM이 정함 |
| speakers | List\<Speaker\> | O | key·name은 카드와 같음. character는 카드와 달리 `{ ref, appearance, imageUrl }`(SourceCharacter, [화자 그림](characters.md#internal-워커--메인-백엔드)) 또는 null |
| lines | List\<Line\> | O | 카드와 같은 모양. id는 콘티의 sourceLineIds에 씀 |

### 콘티

`POST /internal/comics/{id}/storyboard` → 204. 본문은 [콘티 형식](storyboard.md) v1이에요. 화자 키(`characters[].key`, `cast[].key`, `dialogue[].speaker`)는 카드와 같이 `화자1`… 로 맞춰요(스키마 변경은 AI 담당). 받으면 DRAWING 상태가 돼요.

서버는 기본만 검사해요: JSON 객체, `version`이 1, `comicId`·`cardId`가 주소의 만화와 같음, `panels`가 4개. 나머지 검사 규칙([콘티 형식](storyboard.md#검사-규칙))은 워커가 해요. 틀리면 400 INVALID\_REQUEST예요.

### 이미지

`POST /internal/comics/{id}/image` → 204. `multipart/form-data`, 파트 이름 `file`, `image/png`, 800×3152px, 20MB 이하. 받으면 DONE이 되고 푸시를 보내요. PNG가 아니거나 크기가 800×3152px가 아니면 400 INVALID\_REQUEST(field `file`), 20MB를 넘으면 413 PAYLOAD\_TOO\_LARGE예요.

이미 끝난(DONE·FAILED) 만화에 콘티나 이미지가 오면 204로 받고 아무것도 바꾸지 않아요. 만화가 없으면 404 COMIC\_NOT\_FOUND예요.

### JobFailRequest

`POST /internal/jobs/{id}/fail` → 204. 받으면 녹음 감지나 만화가 FAILED가 되고, 만화면 푸시를 보내요. avatar 작업이면 프로필 캐릭터가 FAILED가 되고 원본 사진을 지우고 푸시를 보내요([화자 그림](characters.md)). 이미 결과나 fail을 받은 작업이면 204로 받고 무시해요. 작업이 없으면 400 INVALID\_REQUEST예요.

감지 작업이 실패하면 그 묶음의 글과 앞에서 넘겨받은 글을 바로 지우고, 다음 묶음은 넘겨받을 것 없이 처리해요. 늦게 온 묶음의 넘겨 둔 부분만 처리하던 작업이 실패하면 넘겨 둔 글만 지우고 묶음은 FAILED로 바꾸지 않아요.

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| reason | String | O | 1~500자(DB fail_reason). 서버 기록용, 앱에 안 보냄 |

### 사용자 지우기

`DELETE /internal/users/{id}` → 204. 본문 없음. lolog-auth가 탈퇴 때 불러요. 친구 기능의 프로필(프로필 캐릭터 그림 포함), 친구 관계·요청, 차단도 함께 지워요([친구](friends.md)). 지울 게 없어도 204예요. 같은 요청을 여러 번 불러도 돼요.

만화 이미지를 지우지 못하면 500 INTERNAL\_ERROR예요(DB의 데이터는 이미 지워졌어요). lolog-auth는 204가 올 때까지 다시 불러요. 다시 부르면 남은 이미지만 마저 지워요. 진행 중인 감지·만화 작업도 함께 지워지므로 워커의 다음 요청은 404·400이 돼요.
