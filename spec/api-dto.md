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
| POST | /auth/signup | SignupRequest | 201 TokenResponse |
| POST | /auth/login | LoginRequest | 200 TokenResponse |
| POST | /auth/social/{provider} | SocialLoginRequest | 200 TokenResponse(+isNewUser) |
| POST | /auth/refresh | RefreshRequest | 200 TokenResponse |
| POST | /auth/logout | LogoutRequest | 204 - |
| GET | /auth/me | - | 200 MeResponse |
| POST | /auth/me/social/{provider} | SocialLoginRequest | 200 `{ linkedProviders }` (MeResponse의 한 필드) |
| DELETE | /auth/me | - | 204 - |
| POST | /api/recordings | RecordingUploadRequest | 202 RecordingUploadResponse |
| GET | /api/recordings | 쿼리 cursor, size, status | 200 PageResponse\<RecordingListItem\> |
| GET | /api/recordings/{id} | - | 200 RecordingResponse |
| GET | /api/cards | 쿼리 cursor, size | 200 PageResponse\<CardSummary\> |
| GET | /api/cards/{id} | - | 200 CardDetail |
| PATCH | /api/cards/{id} | CardUpdateRequest | 200 CardDetail |
| DELETE | /api/cards/{id} | - | 204 - |
| POST | /api/cards/{id}/comics | - | 202 ComicCreateResponse |
| GET | /api/comics | 쿼리 cursor, size | 200 PageResponse\<ComicResponse\> |
| GET | /api/comics/{id} | - | 200 ComicResponse |
| DELETE | /api/comics/{id} | - | 204 - |
| PUT | /api/devices/fcm-token | FcmTokenRequest | 204 - |
| DELETE | /api/devices/fcm-token | FcmTokenRequest | 204 - |
| (푸시) | FCM data | - | 3장 표 |
| (큐) | lolog:detect | DetectMessage | - |
| (큐) | lolog:comic | ComicMessage | - |
| POST | /internal/recordings/{id}/cards | DetectResult | 204 - |
| GET | /internal/comics/{id}/source | - | 200 ComicSource |
| POST | /internal/comics/{id}/storyboard | 콘티 JSON(storyboard.md v1) | 204 - |
| POST | /internal/comics/{id}/image | multipart `file` | 204 - |
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

### 오류 코드

| 코드 | HTTP | 어디서 |
| --- | --- | --- |
| INVALID_REQUEST | 400 | 모든 API. 필드가 빠졌거나 타입·제한이 틀림 |
| UNKNOWN_SPEAKER | 400 | PATCH /api/cards/{id}. 카드에 없는 화자 키 |
| INVALID_CREDENTIALS | 401 | /auth/login. 이메일이나 비밀번호가 틀림 |
| TOKEN_EXPIRED | 401 | 액세스·리프레시 토큰 만료. /api도 만료된 액세스 토큰이면 이 코드(앱은 이때만 갱신) |
| INVALID_TOKEN | 401 | 토큰이 틀렸거나 서명이 안 맞음 |
| UNAUTHORIZED | 401 | /api에 토큰이 없거나 틀림(만료 말고), /internal에 내부 토큰이 없거나 틀림 |
| CARD_NOT_FOUND | 404 | 카드가 없거나 내 카드가 아님 |
| COMIC_NOT_FOUND | 404 | 만화가 없거나 내 만화가 아님 |
| RECORDING_NOT_FOUND | 404 | 녹음이 없거나 내 녹음이 아님 |
| ACCOUNT_EXISTS | 409 | 가입, 소셜 로그인 |
| SOCIAL_ALREADY_LINKED | 409 | 소셜 계정 연결 |
| PART_OUT_OF_ORDER | 409 | 녹음 올리기 |
| RECORDING_CLOSED | 409 | 녹음 올리기 |
| CARD_LOCKED | 409 | 카드 고치기·지우기 |
| COMIC_IN_PROGRESS | 409 | 만화 요청 |
| COMIC_LIMIT | 409 | 만화 요청 |
| PAYLOAD_TOO_LARGE | 413 | 녹음 올리기 |
| INTERNAL_ERROR | 500 | 서버 오류 |

남의 카드·만화·녹음은 403이 아니라 404로 답해요. 있는지 없는지 알리지 않으려는 뜻이에요.

### PageResponse

목록 응답은 `{ "<목록 이름>": [...], "nextCursor": ... }` 모양이에요.

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| cards / comics / recordings | List | O | 최신순(id 내림차순, createdAt 순서와 같음). 없으면 빈 배열 |
| nextCursor | String | null 가능 | 다음 페이지를 부를 때 `?cursor=`에 넣는 값. 마지막 페이지면 null. 앱은 값의 뜻을 해석하지 않아요 |

요청 쿼리: `cursor`(String, 선택), `size`(Int, 선택, 기본 20, 1~50).

## 0. 서버 상태 확인

`GET /auth/health`, `GET /api/health`. 토큰 없이 불러요. 200 `{ "status": "UP" }`. DB·Redis에 붙지 못하면 503 `{ "status": "DOWN" }`이에요.

## 1. lolog-auth (/auth)

### SignupRequest · LoginRequest

`POST /auth/signup`, `POST /auth/login`

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| email | String | O | 이메일 형식, 254자 이하. 서버가 소문자로 바꿔 저장 |
| password | String | O | 8~64자, 영문·숫자 포함 (가입 때만 검사, 로그인은 검사 안 함) |

### SocialLoginRequest

`POST /auth/social/{provider}`, `POST /auth/me/social/{provider}`. provider는 주소에서 소문자 `kakao`, `google`, `naver`.

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| providerToken | String | O | 앱이 각 회사 SDK로 받은 액세스 토큰 |

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
| linkedProviders | List\<String\> | O | `KAKAO`, `GOOGLE`, `NAVER` 중 연결된 것. 없으면 빈 배열 |

`POST /auth/me/social/{provider}` 200은 `{ linkedProviders }`만 돌려줘요. 로그아웃·탈퇴는 204, 본문 없음.

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
| status | String | O | 위와 같음. 마지막 묶음까지 감지가 끝나면 DONE |
| cardCount | Int | O | 지금까지 이 녹음에서 나온 카드 수 |

### 2.2 카드

#### CardSummary

`GET /api/cards` 목록의 한 항목

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | |
| kind | String | O | `웃김`·`황당`·`추억` |
| summary | String | O | 한두 문장 요약 |
| recordedAt | DateTime | O | 녹음 시작 시각 |
| startMs | Long | O | 녹음 안에서 이야기가 시작한 때 |
| endMs | Long | O | 끝난 때 |
| comicCount | Int | O | 이 카드로 만든 만화 수(실패한 것 빼고, 만드는 중 포함) |
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
| speakers | List\<Speaker\> | O | 1개 이상, key 순서(화자1, 화자2…) |
| lines | List\<Line\> | O | 1개 이상, 대사 순서 |
| comics | List\<ComicRef\> | O | 이 카드로 만든 만화, 최신순. 없으면 빈 배열 |
| createdAt | DateTime | O | |

**Speaker**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| key | String | O | `화자1`, `화자2`… 바뀌지 않는 키 |
| name | String | null 가능 | 사용자가 붙인 이름. null이면 앱은 key를 보여 줌 |

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

`PATCH /api/cards/{id}`. 바꾼 것만 보내요. 셋 다 빠지면 400 INVALID_REQUEST.

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| speakers | List\<SpeakerUpdate\> | 선택 | 바꾼 화자만 |
| lines | List\<LineUpdate\> | 선택 | 바꾼 대사만 |
| place | String | 선택, null 가능 | 50자 이하. null을 보내면 장소를 지움 |

**SpeakerUpdate**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| key | String | O | 카드에 있는 key. 없으면 400 UNKNOWN_SPEAKER |
| name | String | O, null 가능 | 1~20자. null이면 이름을 지움 |

**LineUpdate**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| id | Long | O | 이 카드의 대사 ID. 아니면 400 INVALID_REQUEST |
| speaker | String | 선택 | 카드에 있는 key. 없으면 400 UNKNOWN_SPEAKER |
| text | String | 선택 | 1~500자 |

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
| status | String | O | `QUEUED`·`STORYBOARD`·`DRAWING`·`DONE`·`FAILED` |
| imageUrl | String | null 가능 | DONE일 때만 값이 있음. 세로 4칸 PNG 한 장. 10분 동안 열리는 서명 주소라 저장하지 말고, 이미지를 못 받으면(403) 다시 조회 |
| createdAt | DateTime | O | 만화를 요청한 시각 |

실패 이유는 앱에 보내지 않아요. 썸네일은 따로 없어요.

### 2.4 푸시 토큰

#### FcmTokenRequest

`PUT /api/devices/fcm-token`, `DELETE /api/devices/fcm-token`. 둘 다 204.

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| token | String | O | FCM 등록 토큰, 512자 이하(DB device_tokens.token) |

## 3. 푸시 (lolog-api → 앱)

FCM **data** 메시지예요. FCM data는 값이 모두 문자열이라 숫자도 문자열로 와요.

| type | 나머지 키 | 앱 동작 |
| --- | --- | --- |
| CARDS_CREATED | recordingId, cardCount | 보관함 새로고침 |
| COMIC_DONE | comicId, cardId | 만화 화면 |
| COMIC_FAILED | comicId, cardId | "만들지 못했어요" |

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
| sentences | List\<Sentence\> | O | 앞 묶음에서 넘겨받은 부분 + 이 묶음. 1개 이상 |

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

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| jobId | Long | O | 받은 메시지의 jobId. 이미 받은 jobId면 204로 받고 무시 |
| partNo | Int | O | |
| cards | List\<DetectedCard\> | O | 없으면 빈 배열 |
| carryFromMs | Long | null 가능 | 다음 묶음으로 넘길 시작 시각. 마지막 묶음이면 null |

**DetectedCard**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| kind | String | O | `웃김`·`황당`·`추억` |
| summary | String | O | 1~200자 |
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

`GET /internal/comics/{id}/source` → 200. 부르면 만화가 STORYBOARD 상태가 돼요.

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| comicId | Long | O | |
| cardId | Long | O | |
| kind | String | O | |
| summary | String | O | |
| place | String | null 가능 | null이면 LLM이 정함 |
| speakers | List\<Speaker\> | O | 카드와 같은 모양 |
| lines | List\<Line\> | O | 카드와 같은 모양. id는 콘티의 sourceLineIds에 씀 |

### 콘티

`POST /internal/comics/{id}/storyboard` → 204. 본문은 [콘티 형식](storyboard.md) v1이에요. 화자 키(`characters[].key`, `cast[].key`, `dialogue[].speaker`)는 카드와 같이 `화자1`… 로 맞춰요(스키마 변경은 AI 담당). 받으면 DRAWING 상태가 돼요.

### 이미지

`POST /internal/comics/{id}/image` → 204. `multipart/form-data`, 파트 이름 `file`, `image/png`, 20MB 이하. 받으면 DONE이 되고 푸시를 보내요.

### JobFailRequest

`POST /internal/jobs/{id}/fail` → 204. 받으면 녹음 감지나 만화가 FAILED가 되고, 만화면 푸시를 보내요.

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| reason | String | O | 1~500자(DB fail_reason). 서버 기록용, 앱에 안 보냄 |

### 사용자 지우기

`DELETE /internal/users/{id}` → 204. 본문 없음. lolog-auth가 탈퇴 때 불러요. 지울 게 없어도 204예요.
