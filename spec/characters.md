# 화자 그림 (기본 화자·프로필 캐릭터)

카드의 화자마다 그림을 하나 붙일 수 있어요. 그림이 붙은 화자는 4컷 모두 그 그림을 참조해서 같은 얼굴로 그려요. 그림을 붙이지 않은 화자는 지금처럼 콘티 LLM이 정한 외모 글로만 그려요(랜덤). 아무 화자에도 붙이지 않으면 지금과 완전히 같아요.

붙일 수 있는 그림은 두 가지예요.

- **기본 화자:** 서비스가 미리 넣어 둔 24명. 어린이·청년·중장년 × 여·남 × 4명, 그룹마다 2명은 안경
- **프로필 캐릭터:** 회원이 본인 사진 한 장으로 만든 본인 캐릭터. 사진은 한 번 웹툰 화풍 그림으로 바꾸고 원본은 바로 지워요. 만화에는 그림만 써요. 내 캐릭터와, 요청·수락으로 친구가 된 사람([친구](friends.md))의 캐릭터를 붙일 수 있어요

친구의 캐릭터는 내가 친구 사진으로 만들지 않아요. 친구가 직접 만든 프로필 캐릭터를 가져와 써요.

## 1. 화면 흐름

1. 카드 자세히 화면에서 화자를 누르면 팝업이 떠요
2. 기본 탭에 기본 화자 24명이 보여요. 나이대·성별로 거를 수 있어요
3. 프로필 탭에 내 프로필 캐릭터와 친구들의 프로필 캐릭터가 보여요. 쓸 수 있는 그림이 있는 사람만 고를 수 있어요. 그림이 없는 친구는 보이지 않거나 고를 수 없게 보여요(앱이 정해요)
4. 고르면 `PATCH /api/cards/{id}`로 그 화자에 그림을 붙여요. 프로필 캐릭터를 고르면 앱이 화자 이름이 비어 있을 때 그 사람의 닉네임도 같이 보내요(사용자가 다시 바꿀 수 있어요)
5. 만화 만들기를 누르면 그림이 붙은 화자는 그 그림대로 그려요

한 카드에서 같은 그림을 두 화자에 붙일 수 없어요(한 컷에 같은 얼굴이 둘 나와요).

## 2. 기본 화자

AI 쪽이 만화와 같은 모델(FLUX.2 klein 4B)·같은 화풍 문장으로 만들어 [`assets/default-characters/`](../assets/default-characters/)에 올려 뒀어요. 백엔드는 이 폴더를 그대로 써요.

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

## 3. 프로필 캐릭터 만들기

1. 앱: 프로필 화면에서 본인 사진 한 장을 골라 올려요. 나이대·성별·안경은 고르지 않아요. "친구가 내 캐릭터를 만화에 쓸 수 있어요"라고 안내해요
2. 백엔드: 사진을 임시로 저장하고 캐릭터 상태를 `PROCESSING`으로 바꾼 뒤 `lolog:comic`에 avatar 메시지를 넣어요
3. 워커(2호 젯슨): 사진을 받아 기본 화자와 같은 화풍·구도로 그림을 그려 보내요. 외모 글도 같이 보내요
4. 백엔드: 그림과 외모 글을 저장하고 **원본 사진을 바로 지워요.** 상태가 `READY`가 되고 본인에게 푸시 `PROFILE_CHARACTER_READY`를 보내요
5. 실패(fail)하면 `FAILED`, 본인에게 푸시 `PROFILE_CHARACTER_FAILED`. 원본 사진은 이때도 바로 지워요. 사진을 다시 올려 다시 만들 수 있어요

- 프로필 캐릭터는 한 사람에 하나예요
- 다시 만들 수 있어요. 다시 만드는 동안과 실패했을 때는 예전 그림을 그대로 써요. 새 그림이 오면 그때 바꿔요
- 지울 수 있어요. 지우면 그 캐릭터가 붙어 있던 카드 화자(내 카드와 친구들 카드)에서 그림이 빠져요(character가 null). 이미 만든 만화는 그대로예요
- 친구를 끊거나 차단하면 서로의 프로필 캐릭터가 상대 카드 화자에서 빠져요
- 탈퇴(`DELETE /internal/users/{id}`)하면 프로필 캐릭터도 지워요
- 원본 사진은 워커가 그림을 보내거나 실패할 때까지만 있어요. 로그에 남기지 않아요

## 4. API

### /api (앱 → 메인 백엔드)

| 메서드 | 주소 | 하는 일 | 요청 | 응답 |
| --- | --- | --- | --- | --- |
| GET | /api/characters/defaults | 기본 화자 목록 | - | 200 `{ characters: List<DefaultCharacterItem> }` |
| PUT | /api/me/profile/character | 사진을 올려 프로필 캐릭터 만들기·다시 만들기 | multipart `photo` | 202 ProfileCharacterResponse(`PROCESSING`) |
| DELETE | /api/me/profile/character | 프로필 캐릭터 지우기. 없어도 204 | - | 204 |
| GET | /api/me/profile | (친구 명세) 응답에 `character` 추가 | - | 200 ProfileResponse |
| GET | /api/friends | (친구 명세) 친구마다 `character` 추가 | - | 200 `{ friends: List<FriendItem> }` |
| PATCH | /api/cards/{id} | 화자에 그림 붙이기·떼기 | SpeakerUpdate에 `character` | 200 CardDetail |

**DefaultCharacterItem**: id, label, ageGroup, gender, glasses, thumbnailUrl(10분 서명 주소). appearance는 앱에 보내지 않아요.

**PUT /api/me/profile/character** (multipart)

| 파트 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| photo | 파일 | O | JPEG·PNG, 10MB 이하. 넘으면 413 PAYLOAD_TOO_LARGE |

만드는 중(`PROCESSING`)에 또 올리면 409 CHARACTER_IN_PROGRESS예요. 닉네임은 없어도 만들 수 있어요.

**ProfileCharacterResponse** (PUT의 응답이자 ProfileResponse의 `character`. 만든 적이 없으면 null)

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| status | String | O | 가장 최근에 올린 사진의 상태. `PROCESSING`·`READY`·`FAILED` |
| thumbnailUrl | String | null 가능 | 쓸 수 있는 그림이 있으면 256 그림의 10분 서명 주소. 처음 만드는 중이거나 처음 만들기에 실패했으면 null |
| updatedAt | DateTime | O | 마지막으로 사진을 올렸거나 상태가 바뀐 시각 |

다시 만드는 중이면 status는 `PROCESSING`이고 thumbnailUrl은 예전 그림이에요.

**FriendItem의 character**: 친구에게 쓸 수 있는 그림이 있으면 `{ thumbnailUrl }`, 없으면 null이에요. 친구의 status는 보내지 않아요.

**CharacterRef** (카드의 Speaker와 SpeakerUpdate에 쓰는 모양)

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| type | String | O | `DEFAULT`·`PROFILE` |
| id | String | O | 기본 화자 id(`kid-f-01`) 또는 회원 ID(문자열, 예: `"42"`). 내 회원 ID나 친구의 회원 ID |

카드 응답의 Speaker에는 `character`가 늘 있고 값은 null이거나 `{ type, id, label, thumbnailUrl }`이에요(label은 기본 화자 label 또는 그 사람의 닉네임. 닉네임이 없으면 null).

`PATCH /api/cards/{id}`의 SpeakerUpdate는 `name`과 `character`가 둘 다 선택이고, 둘 중 하나는 있어야 해요(새 화자를 만들 때는 둘 다 없어도 돼요). `character`를 빼면 그대로, `null`이면 떼요.

```json
{ "speakers": [ { "key": "화자1", "name": "민수", "character": { "type": "PROFILE", "id": "42" } },
                { "key": "화자2", "character": { "type": "DEFAULT", "id": "young-f-02" } } ] }
```

| 오류 코드 | HTTP | 언제 |
| --- | --- | --- |
| INVALID_REQUEST | 400 | PUT에 photo가 없거나 JPEG·PNG가 아님 |
| UNKNOWN_CHARACTER | 400 | 없는 기본 화자 id, 나도 아니고 친구도 아닌 회원 ID |
| DUPLICATE_CHARACTER | 400 | 한 카드에서 같은 그림을 두 화자에 붙임 |
| PAYLOAD_TOO_LARGE | 413 | 사진이 10MB 초과 |
| CHARACTER_NOT_READY | 409 | 쓸 수 있는 그림이 없는 사람(처음 만드는 중, 처음 만들기 실패, 지움)의 캐릭터를 붙임 |
| CHARACTER_IN_PROGRESS | 409 | 만드는 중에 사진을 또 올림 |

만화를 요청할 때(`POST /api/cards/{id}/comics`) 붙어 있던 프로필 캐릭터가 그 사이 지워졌거나 친구가 끊겼으면 그 화자는 그림 없이 만들어요(오류 아님).

### 푸시

본인에게만 보내요.

| type | 나머지 키 | 보내는 때 | 앱 동작 |
| --- | --- | --- | --- |
| PROFILE_CHARACTER_READY | - | 그림을 받았을 때 | 프로필 새로고침 |
| PROFILE_CHARACTER_FAILED | - | avatar 작업 fail을 받았을 때 | "캐릭터를 만들지 못했어요" |

### 큐 (메인 백엔드 → 워커)

같은 스트림 `lolog:comic`을 써요. 2호 젯슨 GPU 하나에서 만화와 차례대로 돌려야 메모리가 모자라지 않아요.

**AvatarMessage**: `{ "jobId": 77, "kind": "avatar", "userId": 42 }`. `kind`가 없으면 지금과 같은 ComicMessage예요.

fail은 기존 `POST /internal/jobs/{id}/fail`을 써요. avatar 작업이면 캐릭터를 FAILED로 바꾸고(예전 그림이 있으면 그 그림은 그대로) 사진을 지워요.

### /internal (워커 → 메인 백엔드)

| 메서드 | 주소 | 하는 일 | 본문 | 응답 |
| --- | --- | --- | --- | --- |
| GET | /internal/users/{userId}/character/photo | 원본 사진 받기 | - | 200 사진 바이트(Content-Type 원본대로). 사진이 이미 지워졌거나 회원이 없으면 404 CHARACTER_NOT_FOUND(워커는 그 메시지를 버림) |
| POST | /internal/users/{userId}/character/avatar | 그림 보내기 | multipart `file`(image/png, 768×768) + `appearance`(영어, 200자 이하) | 204. 받으면 READY, 원본 사진 삭제, 256 그림 만들기, 푸시. 받을 사진이 없으면 404 CHARACTER_NOT_FOUND |
| GET | /internal/comics/{id}/source | (기존) | - | speakers[]에 `character` 추가 |

ComicSource의 Speaker:

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| key, name | | | 카드와 같음 |
| character | SourceCharacter | null 가능 | 그림이 안 붙었으면 null |

**SourceCharacter**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| ref | String | O | `default:young-f-02` 또는 `profile:42`(42는 회원 ID). 콘티 `characters[].ref`에 그대로 남겨요 |
| appearance | String | O | 기본 화자는 characters.json의 appearance, 프로필 캐릭터는 그림과 함께 받은 appearance |
| imageUrl | String | O | 768 원본 PNG의 10분 서명 주소. 워커는 콘티를 만든 뒤 바로 받아요 |

```json
"speakers": [
  { "key": "화자1", "name": "민수",
    "character": { "ref": "profile:42", "appearance": "a young man in his twenties, wearing glasses",
                   "imageUrl": "https://lolog.java21.net/lolog-characters/..." } },
  { "key": "화자2", "name": null, "character": null }
]
```

## 5. 워커가 하는 일

- **콘티:** character가 있는 화자는 LLM이 외모를 정하지 않고 character.appearance를 그대로 써요. `characters[].ref`에 ref를 남겨요
- **그림:** 컷마다 그 컷에 나오는 인물 중 그림이 있는 사람의 그림을 384px로 줄여 FLUX.2 klein 참조 이미지로 넣어요. 프롬프트에서는 "the person from image 1 (외모 글)"로 불러요
- **시간(2호 젯슨 실측, 2026-11-10):** 컷 23초 → 참조 1장 26초, 2장 30초, 3장 37초. 참조 2명 만화 한 편은 약 90초 → 110초. 참조 3장까지 메모리 부족은 없었어요
- **프로필 캐릭터:** 사진을 512px 이하로 줄여 참조로 넣고 기본 화자와 같은 문장으로 768 초상화를 그려요. 사진은 메모리에서만 쓰고 파일로 남기지 않아요
- **외모 글:** 사진만 받기로 해서, 나이대·성별·안경으로 만들던 외모 글(지금 `friend_appearance`)은 AI 쪽에서 사진만으로 만드는 방법으로 바꿔요

## 6. 아직 정할 것

- 기본 화자 그림을 앱에 넣어 둘지(오프라인), 서버 썸네일만 쓸지
- 다시 만드는 동안 예전 그림을 계속 쓸지(지금은 계속 씀)
- 다시 만들기 횟수 제한
- 캐릭터를 지우거나 친구를 끊었을 때 카드 화자에서 그림을 뺄지(지금은 뺌, 만든 만화는 그대로)
- 256 그림을 누가 만들지(지금은 백엔드가 768을 줄임)
- 얼굴이 없거나 여러 명이 찍힌 사진 처리와 사진 안내 문구
- 시드: 지금은 대상 ID로 고정이라 같은 사진으로 다시 만들면 같은 그림이 나올 수 있어요. jobId로 바꿀지
