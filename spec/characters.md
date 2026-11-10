# 화자 그림 (기본 화자·친구)

카드의 화자마다 그림을 하나 붙일 수 있어요. 그림이 붙은 화자는 4컷 모두 그 그림을 참조해서 같은 얼굴로 그려요. 그림을 붙이지 않은 화자는 지금처럼 콘티 LLM이 정한 외모 글로만 그려요(랜덤). 아무 화자에도 붙이지 않으면 지금과 완전히 같아요.

붙일 수 있는 그림은 두 가지예요.

- **기본 화자:** 서비스가 미리 넣어 둔 24명. 어린이·청년·중장년 × 여·남 × 4명, 그룹마다 2명은 안경
- **친구:** 사용자가 사진으로 등록한 친구. 사진은 등록할 때 한 번 웹툰 화풍 아바타로 바꾸고 원본은 바로 지워요. 만화에는 아바타만 써요

## 1. 화면 흐름

1. 카드 자세히 화면에서 화자를 누르면 팝업이 떠요
2. 기본 탭에 기본 화자 24명이 보여요. 나이대·성별로 거를 수 있어요
3. 친구 탭에 내가 등록한 친구가 보여요. READY인 친구만 고를 수 있고, PROCESSING은 "만드는 중", FAILED는 "다시 올리기"로 보여요
4. 고르면 `PATCH /api/cards/{id}`로 그 화자에 그림을 붙여요. 친구를 고르면 앱이 화자 이름이 비어 있을 때 친구 이름도 같이 보내요(사용자가 다시 바꿀 수 있어요)
5. 만화 만들기를 누르면 그림이 붙은 화자는 그 그림대로 그려요

한 카드에서 같은 그림을 두 화자에 붙일 수 없어요(한 컷에 같은 얼굴이 둘 나와요).

## 2. 기본 화자

AI 쪽이 만화와 같은 모델(FLUX.2 klein 4B)·같은 화풍 문장으로 만들어 백엔드에 폴더 하나로 넘겨요.

| 파일 | 내용 |
| --- | --- |
| `<id>.png` | 768×768 원본 24장. 워커가 참조로 받는 그림 |
| `<id>_thumb.webp` | 256×256 썸네일 24장. 앱 팝업용 |
| `characters.json` | 아래 표의 필드 24개 |

**DefaultCharacter** (`characters.json`의 `characters[]` 한 항목)

| 필드 | 타입 | 설명 |
| --- | --- | --- |
| id | String | `kid-f-01`~`middle-m-04`. 바뀌지 않는 키 |
| ageGroup | String | `CHILD`·`YOUNG`·`MIDDLE` |
| gender | String | `F`·`M` |
| glasses | Boolean | 안경 여부 |
| label | String | 앱에 보일 한글 설명. 예: "동그란 안경 양갈래 소녀" |
| appearance | String | 그림 모델용 영어 외모 글. 앱에는 보이지 않아요 |
| seed | Int | 만든 시드(다시 만들 때용) |
| image, thumbnail | String | 파일 이름 |

백엔드는 처음 한 번 파일을 MinIO에 올리고 `characters.json`을 테이블에 넣어요(시드 데이터).

## 3. 친구 등록

1. 앱: 이름(1~10자), 나이대, 성별, 안경 여부를 고르고 사진 한 장을 올려요. "친구에게 사진 사용 동의를 받았어요"를 체크해야 올릴 수 있어요
2. 백엔드: 친구를 `PROCESSING`으로 만들고 사진을 저장한 뒤 `lolog:comic`에 avatar 메시지를 넣어요
3. 워커(2호 젯슨): 사진을 받아 기본 화자와 같은 화풍·구도로 아바타를 그려 보내요. 외모 글도 같이 보내요
4. 백엔드: 아바타와 외모 글을 저장하고 **원본 사진을 바로 지워요.** 친구는 `READY`가 되고 푸시 `FRIEND_READY`를 보내요
5. 실패(fail)하면 `FAILED`, 푸시 `FRIEND_FAILED`. 원본 사진은 이때도 바로 지워요. 사용자는 사진을 다시 올려 다시 만들 수 있어요

- 친구를 지우면 아바타도 바로 지우고, 그 친구가 붙어 있던 카드 화자의 그림은 빠져요(character가 null)
- 탈퇴(`DELETE /internal/users/{id}`)하면 친구와 아바타도 모두 지워요
- 원본 사진은 워커가 아바타를 보내거나 실패할 때까지만 있어요. 로그에 남기지 않아요

## 4. API

### /api (앱 → 메인 백엔드)

| 메서드 | 주소 | 하는 일 | 요청 | 응답 |
| --- | --- | --- | --- | --- |
| GET | /api/characters/defaults | 기본 화자 목록 | - | 200 `{ characters: List<DefaultCharacterItem> }` |
| GET | /api/friends | 내 친구 목록(만든 순) | - | 200 `{ friends: List<FriendResponse> }` |
| POST | /api/friends | 친구 등록 | multipart FriendCreateRequest | 202 FriendResponse(`PROCESSING`) |
| PUT | /api/friends/{id}/photo | 사진 다시 올려 아바타 다시 만들기 | multipart `photo` | 202 FriendResponse(`PROCESSING`) |
| PATCH | /api/friends/{id} | 이름 바꾸기 | `{ name }` | 200 FriendResponse |
| DELETE | /api/friends/{id} | 친구와 아바타 지우기 | - | 204 |
| PATCH | /api/cards/{id} | 화자에 그림 붙이기·떼기 | SpeakerUpdate에 `character` | 200 CardDetail |

**DefaultCharacterItem**: id, label, ageGroup, gender, glasses, thumbnailUrl(10분 서명 주소). appearance는 앱에 보내지 않아요.

**FriendCreateRequest** (multipart)

| 파트 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| name | String | O | 1~10자 |
| ageGroup | String | O | `CHILD`·`YOUNG`·`MIDDLE` |
| gender | String | O | `F`·`M` |
| glasses | Boolean | O | |
| consent | Boolean | O | true가 아니면 400 INVALID_REQUEST |
| photo | 파일 | O | JPEG·PNG, 10MB 이하. 넘으면 413 PAYLOAD_TOO_LARGE |

친구는 사용자당 20명까지예요. 넘으면 409 FRIEND_LIMIT.

**FriendResponse**: id(Long), name, ageGroup, gender, glasses, status(`PROCESSING`·`READY`·`FAILED`), thumbnailUrl(READY일 때만, 10분 서명 주소, 아니면 null), createdAt.

**CharacterRef** (카드의 Speaker와 SpeakerUpdate에 쓰는 모양)

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| type | String | O | `DEFAULT`·`FRIEND` |
| id | String | O | 기본 화자 id(`kid-f-01`) 또는 친구 id(문자열, 예: `"42"`) |

카드 응답의 Speaker에는 `character`가 늘 있고 값은 null이거나 `{ type, id, label, thumbnailUrl }`이에요(label은 기본 화자 label 또는 친구 이름).

`PATCH /api/cards/{id}`의 SpeakerUpdate는 `name`과 `character`가 둘 다 선택이고, 둘 중 하나는 있어야 해요. `character`를 빼면 그대로, `null`이면 떼요.

```json
{ "speakers": [ { "key": "화자1", "name": "민수", "character": { "type": "FRIEND", "id": "42" } },
                { "key": "화자2", "character": { "type": "DEFAULT", "id": "young-f-02" } } ] }
```

| 오류 코드 | HTTP | 언제 |
| --- | --- | --- |
| UNKNOWN_CHARACTER | 400 | 없는 기본 화자 id, 없거나 남의 친구 |
| DUPLICATE_CHARACTER | 400 | 한 카드에서 같은 그림을 두 화자에 붙임 |
| CHARACTER_NOT_READY | 409 | PROCESSING·FAILED 친구를 붙임 |
| FRIEND_NOT_FOUND | 404 | 없거나 남의 친구 (/api/friends/{id}) |
| FRIEND_LIMIT | 409 | 친구 20명 초과 |

만화를 요청할 때(`POST /api/cards/{id}/comics`) 붙어 있던 친구가 그 사이 FAILED·삭제됐으면 그 화자는 그림 없이 만들어요(오류 아님).

### 푸시

| type | 나머지 키 | 보내는 때 | 앱 동작 |
| --- | --- | --- | --- |
| FRIEND_READY | friendId | 아바타를 받았을 때 | 친구 목록 새로고침 |
| FRIEND_FAILED | friendId | avatar 작업 fail을 받았을 때 | "아바타를 만들지 못했어요" |

### 큐 (메인 백엔드 → 워커)

같은 스트림 `lolog:comic`을 써요. 2호 젯슨 GPU 하나에서 만화와 차례대로 돌려야 메모리가 모자라지 않아요.

**AvatarMessage**: `{ "jobId": 77, "kind": "avatar", "friendId": 42, "ageGroup": "YOUNG", "gender": "F", "glasses": true }`. `kind`가 없으면 지금과 같은 ComicMessage예요.

fail은 기존 `POST /internal/jobs/{id}/fail`을 써요. avatar 작업이면 친구를 FAILED로 바꾸고 사진을 지워요.

### /internal (워커 → 메인 백엔드)

| 메서드 | 주소 | 하는 일 | 본문 | 응답 |
| --- | --- | --- | --- | --- |
| GET | /internal/friends/{id}/photo | 원본 사진 받기 | - | 200 사진 바이트(Content-Type 원본대로). 사진이 이미 지워졌거나 친구가 없으면 404 FRIEND_NOT_FOUND(워커는 그 메시지를 버림) |
| POST | /internal/friends/{id}/avatar | 아바타 보내기 | multipart `file`(image/png, 768×768) + `appearance`(영어, 200자 이하) | 204. 받으면 READY, 원본 사진 삭제, 푸시 |
| GET | /internal/comics/{id}/source | (기존) | - | speakers[]에 `character` 추가 |

ComicSource의 Speaker:

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| key, name | | | 카드와 같음 |
| character | SourceCharacter | null 가능 | 그림이 안 붙었으면 null |

**SourceCharacter**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| ref | String | O | `default:young-f-02` 또는 `friend:42`. 콘티 `characters[].ref`에 그대로 남겨요 |
| appearance | String | O | 기본 화자는 characters.json의 appearance, 친구는 아바타와 함께 받은 appearance |
| imageUrl | String | O | 768 원본 PNG의 10분 서명 주소. 워커는 콘티를 만든 뒤 바로 받아요 |

```json
"speakers": [
  { "key": "화자1", "name": "민수",
    "character": { "ref": "friend:42", "appearance": "a young man in his twenties, wearing glasses",
                   "imageUrl": "https://lolog.java21.net/lolog-characters/..." } },
  { "key": "화자2", "name": null, "character": null }
]
```

## 5. 워커가 하는 일

- **콘티:** character가 있는 화자는 LLM이 외모를 정하지 않고 character.appearance를 그대로 써요. `characters[].ref`에 ref를 남겨요
- **그림:** 컷마다 그 컷에 나오는 인물 중 그림이 있는 사람의 그림을 384px로 줄여 FLUX.2 klein 참조 이미지로 넣어요. 프롬프트에서는 "the person from image 1 (외모 글)"로 불러요
- **시간(2호 젯슨 실측, 2026-11-10):** 컷 23초 → 참조 1장 26초, 2장 30초, 3장 37초. 참조 2명 만화 한 편은 약 90초 → 110초. 참조 3장까지 메모리 부족은 없었어요
- **아바타:** 사진을 512px 이하로 줄여 참조로 넣고 기본 화자와 같은 문장으로 768 초상화를 그려요. 사진은 메모리에서만 쓰고 파일로 남기지 않아요

## 6. 아직 정할 것

- 기본 화자 그림을 앱에 넣어 둘지(오프라인), 서버 썸네일만 쓸지
- 친구 20명 제한이 맞는지
- 아바타가 사진과 덜 닮았을 때 다시 만들기 횟수 제한
