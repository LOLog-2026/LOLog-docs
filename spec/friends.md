# 친구

사용자끼리 친구를 맺는 기능이에요. 친구 코드로 요청을 보내고, 상대가 수락하면 친구가 돼요. 친구가 된 뒤에야 서로의 프로필 캐릭터를 화자에 불러올 수 있어요(프로필 캐릭터는 [화자 그림](characters.md)).

- 친구는 사용자끼리의 친구예요. `/api/friends`와 `FRIEND_` 이름은 이 기능의 것이에요
- 친구는 친구 코드로 찾아요. 닉네임은 화면에 보이는 이름으로만 쓰고, 겹쳐도 돼요
- 요청하고 수락해야 친구가 돼요. 모르는 사람과 친구가 되는 걸 막는 장치예요
- 친구는 100명까지예요
- 차단할 수 있어요

## 1. 화면 흐름

화면을 어디에 둘지는 앱 담당과 정해요.

1. 이메일 가입 화면에서 닉네임을 적을 수 있어요(선택). 입력 칸 안내는 "친구에게 보여질 별명을 입력해 주세요"이고, "나중에 설정에서 바꿀 수 있어요"를 같이 보여 줘요
2. 친구 화면 맨 위에 내 닉네임과 내 친구 코드가 보여요(코드는 복사·공유 버튼). 닉네임이 없는 사람(가입 때 비워 둔 사람, 소셜 가입, 이미 가입한 사람)에게는 닉네임 자리가 입력 칸으로 보이고, 정하기 전에는 친구 추가가 꺼져 있어요. 따로 띄우는 닉네임 화면은 없어요
3. "친구 추가"에서 상대의 친구 코드를 넣으면 요청이 가요
4. 상대는 친구 화면의 "받은 요청"에서 수락하거나 거절해요. 요청을 받으면 푸시가 와요
5. 수락하면 두 사람의 친구 목록에 서로가 보여요. 수락했다는 푸시는 보내지 않아요
6. 친구 목록에서 친구를 끊을 수 있어요. 끊으면 양쪽 목록에서 모두 빠져요

- 내가 요청을 보냈는데 상대도 이미 나에게 요청을 보낸 상태면, 수락 없이 바로 친구가 돼요
- 거절해도 상대에게 알리지 않아요. 상대 화면에서는 보낸 요청이 사라지기만 해요
- 거절·취소·친구 끊기를 한 뒤에도 나중에 다시 요청할 수 있어요
- 탈퇴하면 그 회원의 프로필, 친구 관계·요청·차단을 모두 지워요. 친구들 목록에서는 그 사람이 빠지기만 해요
- 친구 목록은 최근에 친구가 된 사람이 위에 와요. 친구 화면에 닉네임 검색 칸이 있고, 검색은 앱이 받은 목록 안에서 걸러요(서버 API 없음)
- 닉네임이 같은 친구는 닉네임 옆에 친구 코드를 작게 보여 구분해요. 받은 요청에도 코드를 같이 보여요. 친구의 프로필 캐릭터가 있으면 그 그림도 보이고, 없으면 기본 프로필 그림이 보여요
- 닉네임 바꾸기는 설정 화면에도 있어요
- 가입 직후 닉네임 저장이 실패해도(통신 끊김 등) 친구 화면의 같은 입력 칸에서 다시 정해요

### 닉네임

- 1~10자. 앞뒤 공백은 잘라 저장하고, 공백만 있으면 400 INVALID\_REQUEST예요
- 겹쳐도 돼요
- 메인 백엔드가 가지고 있어요. 인증 서비스의 회원 정보는 바꾸지 않아요
- 이메일 가입에서 닉네임을 적었으면, 앱이 가입(`POST /auth/signup`)에 성공한 뒤 받은 토큰으로 바로 `PATCH /api/me/profile`에 보내요

### 친구 코드

- 대문자와 숫자 8자리예요. 헷갈리는 글자(0·O·1·I·L)는 빼요. 예: `A7K3QX2M`
- 프로필이 처음 만들어질 때 서버가 발급해요. 겹치면 다시 뽑아요
- 한 번 받은 코드는 바뀌지 않아요

### 차단

- 친구 목록이나 받은 요청에서 차단해요. 차단하면 두 사람 사이의 친구 관계와 요청이 모두 지워져요
- 차단된 사람이 내 친구 코드로 요청하면 없는 코드와 같은 404 FRIEND\_CODE\_NOT\_FOUND를 받아요. 차단됐다는 걸 상대가 알 수 없게 해요
- 내가 차단한 사람에게 요청을 보내면 409 USER\_BLOCKED예요. 풀고 나서 보내요
- 차단을 풀어도 친구로 돌아가지 않아요. 다시 요청해야 해요
- 설정의 "차단한 사람" 목록에서 풀어요
- 차단할 수 있는 건 친구이거나 요청이 오간 사람이에요

## 2. API

모두 `/api`(메인 백엔드)이고 로그인이 필요해요.

| 메서드 | 주소 | 하는 일 | 요청 | 응답 |
| --- | --- | --- | --- | --- |
| GET | /api/me/profile | 내 프로필 보기. 처음 부르면 프로필을 만들고 친구 코드를 발급해요 | - | 200 ProfileResponse |
| PATCH | /api/me/profile | 닉네임 정하기·바꾸기. 프로필이 없으면 만들어요 | ProfileUpdateRequest | 200 ProfileResponse |
| POST | /api/friend-requests | 친구 코드로 요청 보내기. 상대가 이미 나에게 요청했으면 바로 친구가 돼요 | FriendRequestCreate | 201 FriendRequestResponse |
| GET | /api/friend-requests | 받은 요청·보낸 요청 목록(최신순) | 쿼리 `direction`(`received`·`sent`, 필수) | 200 `{ requests: List<FriendRequestResponse> }` |
| POST | /api/friend-requests/{id}/accept | 받은 요청 수락 | - | 200 FriendItem |
| DELETE | /api/friend-requests/{id} | 받은 요청 거절, 보낸 요청 취소 | - | 204 |
| GET | /api/friends | 내 친구 목록. 최근에 친구가 된 사람이 위 | - | 200 `{ friends: List<FriendItem> }` |
| DELETE | /api/friends/{userId} | 친구 끊기. 친구가 아니어도 204 | - | 204 |
| POST | /api/blocks | 차단하기 | `{ userId }` | 204 |
| GET | /api/blocks | 차단한 사람 목록(최신순) | - | 200 `{ blocks: List<BlockItem> }` |
| DELETE | /api/blocks/{userId} | 차단 풀기. 차단한 적 없어도 204 | - | 204 |

- 목록은 친구가 100명까지라 한 번에 다 줘요(cursor 없음)
- 닉네임과 친구 코드는 친구이거나 요청이 오간 사람에게만 보여요. 이메일은 누구에게도 주지 않아요

**ProfileResponse**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| nickname | String | null 가능 | 아직 안 정했으면 null |
| friendCode | String | O | 8자리 친구 코드 |
| character | ProfileCharacterResponse | null 가능 | 내 프로필 캐릭터. 만든 적이 없으면 null. [화자 그림](characters.md#4-api) |

**ProfileUpdateRequest**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| nickname | String | O | 1~10자(앞뒤 공백을 뺀 길이) |

**FriendRequestCreate**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| friendCode | String | O | 8자. 서버는 대문자로 바꿔 찾아요 |

**FriendRequestResponse**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | 요청 ID |
| direction | String | O | `RECEIVED`(내가 받음)·`SENT`(내가 보냄) |
| user | FriendUser | O | 상대 |
| status | String | O | `PENDING`. POST에서 바로 친구가 됐으면 `ACCEPTED` |
| createdAt | DateTime | O | 요청한 시각 |

**FriendUser**: userId(Long), nickname(String, null 가능. 상대가 아직 안 정했으면 null), friendCode(String).

**FriendItem**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| userId | Long | O | 친구의 회원 ID |
| nickname | String | O | |
| friendCode | String | O | 닉네임이 같은 친구 구분용 |
| friendsSince | DateTime | O | 친구가 된 시각 |
| character | Object | null 가능 | 친구에게 쓸 수 있는 그림이 있으면 `{ thumbnailUrl }`(256 그림, 10분 서명 주소), 없으면 null |

**BlockItem**: userId(Long), nickname(String, null 가능), friendCode(String), blockedAt(DateTime).

### 오류 코드

| 코드 | HTTP | 언제 |
| --- | --- | --- |
| INVALID_REQUEST | 400 | 닉네임이 비었거나 10자 초과, friendCode가 빠짐, direction이 틀림 |
| CANNOT_FRIEND_SELF | 400 | 내 코드를 넣음, 나를 차단하려 함 |
| FRIEND_CODE_NOT_FOUND | 404 | 없는 친구 코드. 상대가 나를 차단했을 때도 이 코드 |
| FRIEND_REQUEST_NOT_FOUND | 404 | 없는 요청, 내 요청이 아님, 보낸 사람이 수락하려 함 |
| USER_NOT_FOUND | 404 | 차단: 친구도 아니고 요청도 오간 적 없는 userId |
| PROFILE_REQUIRED | 409 | 닉네임을 정하지 않고 요청을 보내거나 수락함 |
| ALREADY_FRIENDS | 409 | 이미 친구인 사람에게 요청 |
| FRIEND_REQUEST_EXISTS | 409 | 같은 사람에게 이미 보낸 요청이 있음 |
| FRIEND_LIMIT | 409 | 나나 상대의 친구가 이미 100명(요청 보내기·수락) |
| USER_BLOCKED | 409 | 내가 차단한 사람에게 요청 |
| TOO_MANY_REQUESTS | 429 | 친구 코드 입력을 짧은 시간에 너무 많이 함(코드 맞히기 막기. 1분에 10번) |

## 3. 푸시

| type | 나머지 키 | 보내는 때 | 앱 동작 |
| --- | --- | --- | --- |
| FRIEND_REQUEST | requestId | 친구 요청을 받았을 때 | 친구 화면의 받은 요청 |

- 문구에 닉네임만 넣어요. 예: "민수님이 친구 요청을 보냈어요"
- 요청이 바로 친구가 된 경우(상대도 나에게 요청해 둔 경우)는 보내지 않아요
- 수락했을 때는 보내지 않아요
- 푸시를 못 받아도 받은 요청은 친구 화면에 들어올 때 조회하니 빠지지 않아요
