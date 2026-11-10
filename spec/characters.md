# 화자 그림 (기본 화자·프로필 캐릭터)

카드의 화자마다 그림을 하나 붙일 수 있어요. 그림이 붙은 화자는 4컷 모두 그 그림을 참조해서 같은 얼굴로 그려요. 그림을 붙이지 않은 화자는 지금처럼 콘티 LLM이 정한 외모 글로만 그려요(랜덤). 아무 화자에도 붙이지 않으면 지금과 완전히 같아요.

붙일 수 있는 그림은 두 가지예요.

- **기본 화자:** 서비스가 미리 넣어 둔 24명. 어린이·청년·중장년 × 여·남 × 4명, 그룹마다 2명은 안경
- **프로필 캐릭터:** 회원이 본인 사진 한 장으로 만든 본인 캐릭터. 사진은 한 번 웹툰 화풍 그림으로 바꾸고 원본은 바로 지워요. 만화에는 그림만 써요. 내 캐릭터와, 요청·수락으로 친구가 된 사람([친구](friends.md))의 캐릭터를 붙일 수 있어요

친구의 캐릭터는 내가 친구 사진으로 만들지 않아요. 친구가 직접 만든 프로필 캐릭터를 가져와 써요.

## 1. 화면 흐름

카드 자세히(또는 고치기) 화면에서 화자를 누르면 화자 고치기 팝업이 떠요. 팝업에는 기본 탭(기본 화자)과 친구 프로필 탭(나와 친구들)이 있어요. 그 화자가 누구인지에 따라 넷으로 나뉘어요.

| 화자가 누구인지 | 이름 | 그림 | 서버로 보내는 것 |
| --- | --- | --- | --- |
| ① 친구가 아닌 사람 | 직접 적어요 | 기본 화자 중에서 골라요. 안 고르면 AI가 외모를 정해요 | `name`, 골랐으면 `character { type: DEFAULT, id }` |
| ② 친구, 프로필 캐릭터 있음 | 친구 목록에서 고르면 그 친구의 닉네임이 들어가요 | 그 친구의 프로필 캐릭터가 같이 붙어요 | `name`, `character { type: PROFILE, id: 친구 회원 ID }` |
| ③ 친구, 프로필 캐릭터 없음 | 친구 목록에서 고르면 그 친구의 닉네임이 들어가요 | 붙일 그림이 없어요. ①처럼 기본 화자에서 고르거나 안 골라요 | `name`, 골랐으면 `character { type: DEFAULT, id }` |
| ④ 나 | 내 닉네임이 들어가요 | 내 프로필 캐릭터가 있으면 ②처럼 붙어요. 없으면 "프로필 캐릭터 만들기" 버튼이 보이고, 그동안은 ③처럼 해요 | ②·③과 같아요(회원 ID는 내 것) |

- 친구 프로필 탭의 친구 목록에는 모든 친구를 똑같이 보여요. 그림이 있는 친구는 그 그림, 없는 친구는 기본 프로필 그림으로 보여요(흐리게 하거나 숨기지 않아요)
- 그림이 없는 친구를 고르면 닉네임이 들어가고, 같은 팝업에서 기본 화자 목록과 "AI에게 맡기기" 버튼이 보여요. 안 고르고 닫아도 AI에게 맡긴 걸로 봐요. 이름을 직접 적은 사람(①)도 이 단계는 같아요
- 친구(또는 나)를 고르면 닉네임이 화자 이름으로 자동으로 들어가요. 들어간 이름은 고른 뒤에 고칠 수 있어요
- ②에서 친구의 프로필 캐릭터 대신 기본 화자를 쓰고 싶으면 기본 탭에서 다시 골라요. 이름은 그대로예요
- ③은 서버에 이름만 남아요. "이 화자가 그 친구"라는 건 남지 않아서, 그 친구가 나중에 프로필 캐릭터를 만들어도 저절로 붙지 않아요. 다시 골라야 해요
- 기본 화자는 나이대·성별로 거를 수 있어요

순서는 이래요.

1. 카드 자세히(또는 고치기) 화면에서 화자를 눌러요
2. 위 표대로 이름을 적거나 친구(또는 나)를 고르고, 필요하면 기본 화자를 골라요
3. 앱이 `PATCH /api/cards/{id}`로 그 화자의 이름과 그림을 보내요
4. 만화 만들기를 누르면 그림이 붙은 화자는 그 그림대로 그려요

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

그림 파일은 MinIO에만 둬요(api 레포에는 넣지 않아요). 기본 화자 목록은 백엔드가 DB로 관리해요. 그래야 나중에 기본 화자를 늘릴 때 그림을 올리고 목록에 더하기만 하면 돼요. 처음 24명은 이 폴더의 파일과 `characters.json`을 그대로 넣어요.

- **넣는 방법:** 메인 백엔드의 `PUT /internal/characters/defaults/{id}`(아래 /internal 표)로 한 명씩 넣어요. 같은 id로 다시 넣으면 덮어써요. lolog-api의 스크립트가 이 폴더의 24명을 차례로 넣어요. 배포한 뒤 한 번 돌리고, 기본 화자를 늘릴 때도 같은 API로 더해요
- **저장 위치:** 만화 이미지와 같은 버킷(`lolog-comics`)의 `defaults/characters/{id}.png`(768), `defaults/characters/{id}_thumb.webp`(256)

## 3. 프로필 캐릭터 만들기

1. 앱: 프로필 화면에서 본인 사진 한 장을 골라 올려요. 나이대·성별·안경은 고르지 않아요. "친구가 내 캐릭터를 만화에 쓸 수 있어요"라고 안내해요
2. 백엔드: 사진을 임시로 저장하고 캐릭터 상태를 `PROCESSING`으로 바꾼 뒤 `lolog:comic`에 avatar 메시지를 넣어요
3. 워커(2호 젯슨): 사진을 받아 기본 화자와 같은 화풍·구도로 그림을 그려 보내요. 외모 글도 같이 보내요
4. 백엔드: 그림과 외모 글을 저장하고 **원본 사진을 바로 지워요.** 상태가 `READY`가 되고 본인에게 푸시 `PROFILE_CHARACTER_READY`를 보내요
5. 실패(fail)하면 `FAILED`, 본인에게 푸시 `PROFILE_CHARACTER_FAILED`. 원본 사진은 이때도 바로 지워요. 사진을 다시 올려 다시 만들 수 있어요

- 프로필 캐릭터는 한 사람에 하나예요
- 다시 만들 수 있어요. 다시 만드는 동안과 실패했을 때는 예전 그림을 그대로 써요. 새 그림이 오면 그때 바꿔요
- 새 그림이 오면 이미 그 캐릭터를 붙여 둔 카드 화자(친구 카드 포함)도 새 그림을 써요. 화자는 그림 파일이 아니라 "그 사람의 캐릭터"를 가리키기 때문이에요. 이미 만든 만화는 그대로예요
- 만들기는 하루 3번까지예요(한국 시간 자정에 다시 세요). 사진을 올려 202를 받은 것을 세고, 실패(FAILED)한 것은 세지 않아요. 넘으면 409 CHARACTER\_LIMIT. 2호 젯슨 GPU를 만화와 같이 써서, 계속 다시 만들면 다른 사람의 만화가 늦어지기 때문이에요
- 앱은 사진을 올리기 전에 폰 안에서 얼굴이 한 명만 보이는지 확인해요(ML Kit 얼굴 감지). 얼굴이 없거나 여러 명이면 올리지 않고 "얼굴이 한 명만 보이는 사진을 골라 주세요"를 보여 줘요. 서버와 워커는 얼굴을 따로 검사하지 않아요
- 256 그림은 백엔드가 768 그림을 줄여 PNG로 만들어요
- 저장 위치: `users/{userId}/character/avatar.png`(768), `avatar_thumb.png`(256), 원본 사진 `users/{userId}/character/photo`. 모두 `users/{userId}/` 아래라 탈퇴할 때 함께 지워져요
- 지울 수 있어요. 지우면 그 캐릭터가 붙어 있던 카드 화자(내 카드와 친구들 카드)에서 그림이 빠져요(character가 null). 이미 만든 만화는 그대로예요
- 친구를 끊거나 차단하면 서로의 프로필 캐릭터가 상대 카드 화자에서 빠져요
- 빠지는 건 지우기·끊기·차단·탈퇴를 처리하는 그때 바로예요. 다시 친구가 되거나 캐릭터를 다시 만들어도 빠진 화자에 저절로 돌아오지 않아요(다시 골라야 해요)
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

기본 화자 썸네일은 앱에 넣어 두지 않고 서버 주소로 받아요. 앱은 받은 그림을 캐시해요. 서명 주소가 10분마다 바뀌니 캐시는 주소가 아니라 기본 화자 id 기준이에요.

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
| CHARACTER_LIMIT | 409 | 오늘 만들기 3번을 다 씀(실패한 것은 세지 않음) |

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
| PUT | /internal/characters/defaults/{id} | 기본 화자 넣기·바꾸기(스크립트용) | multipart `image`(PNG 768, 선택), `thumbnail`(WebP 256, 선택), `ageGroup`, `gender`, `glasses`, `label`, `appearance`, `sortOrder` | 204. 그림을 빼면 이미 저장소에 있다고 보고 목록만 넣어요(배포 MinIO에는 11/11에 미리 올림) |
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
                   "imageUrl": "https://lolog.java21.net/lolog-comics/..." } },
  { "key": "화자2", "name": null, "character": null }
]
```

## 5. 워커가 하는 일

- **콘티:** character가 있는 화자는 LLM이 외모를 정하지 않고 character.appearance를 그대로 써요. `characters[].ref`에 ref를 남겨요
- **그림:** 컷마다 그 컷에 나오는 인물 중 그림이 있는 사람의 그림을 384px로 줄여 FLUX.2 klein 참조 이미지로 넣어요. 프롬프트에서는 "the person from image 1 (외모 글)"로 불러요
- **시간(2호 젯슨 실측, 2026-11-10):** 컷 23초 → 참조 1장 26초, 2장 30초, 3장 37초. 참조 2명 만화 한 편은 약 90초 → 110초. 참조 3장까지 메모리 부족은 없었어요
- **프로필 캐릭터:** 사진을 512px 이하로 줄여 참조로 넣고 기본 화자와 같은 문장으로 768 초상화를 그려요. 사진은 메모리에서만 쓰고 파일로 남기지 않아요
- **시드:** 메시지의 `jobId`를 시드로 써요. 같은 사진으로 다시 만들어도 다른 그림이 나와요
- **외모 글:** 사진만 받기로 해서, 나이대·성별·안경으로 만들던 외모 글(지금 `friend_appearance`)은 AI 쪽에서 사진만으로 만드는 방법으로 바꿔요

## 6. 아직 정할 것

- 지금은 없어요. 처음 적어 둔 여섯 가지(기본 화자 올리는 방법, 다시 만드는 동안 예전 그림, 다시 만들기 횟수, 256 그림, 얼굴 사진 처리, 시드)는 위 2·3·5장에 정했어요
