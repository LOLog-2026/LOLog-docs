# 친구와 같이 쓰는 방 (초안)

친구 몇 명과 방을 만들어 내 만화를 올리고, 좋아요와 댓글로 이야기하는 기능이에요(FR-ROOM-01~06). 방은 [친구](friends.md)끼리 만들어요.

> 이 문서는 초안이에요. 앱 PR(LOLog-app #53)의 화면·목업 규칙을 바탕으로 썼고, 그 뒤 업데이트 설계에서 정한 것(방장, 초대, 해산, 나간 멤버, 차단, 방에서 친구 요청)을 더했어요. 더한 것은 아직 구현 전이에요. 이름이 "(초안)"인 주소·필드·코드는 구현하면서 바뀌어도 돼요.

- 방은 나를 포함해 2~6명이에요. 방장이 친구를 초대하고, 초대받은 친구가 수락하면 들어와요
- 한 사람이 들어갈 수 있는 방은 20개예요
- 방에는 내 만화를 올려요. 올린 만화는 쌓이고(덮어쓰지 않음), 한 사람이 올리는 수에 제한이 없어요
- 올린 만화에 좋아요를 누르고 댓글을 달 수 있어요
- 초대, 내 만화에 달린 댓글, 방 해산은 푸시로 알려요. 그 밖의 새 글은 푸시 없이 방 목록에 받지 않은 글 수(unread)로 보여 줘요
- 음성은 방에 나가지 않아요. 만화 그림만 보여요([순간 다시 듣기](replay.md): 음성은 카드 주인만)

## 1. 화면 흐름

화면을 어디에 둘지는 앱 담당과 정해요. 지금 앱은 친구 탭 맨 위 "방" / "친구" 알약에서 들어가요.

1. **방 목록:** 마지막 소식이 최근인 방이 위에 와요. 방마다 이름, 멤버 그림, 마지막 소식, 받지 않은 글 수가 보여요. 받은 초대도 여기서 수락하거나 거절해요
2. **방 만들기:** 방 이름을 적고 친구를 1~5명 골라요. 고른 친구에게 초대가 가요
3. **방 안 – 만화:** 올린 만화가 2열 격자로 보여요(최근에 올린 것이 앞). 누르면 크게 보고, 꾹 누르면 좋아요·댓글·저장·내리기가 나와요
   - **만화 댓글 창(정함 11-12, 지원, 앱 PR #91):** 크게 보기의 댓글 버튼과 꾹 누르기 "댓글"이 그 만화 댓글 창을 열어요. 창은 새 주소 없이 방 대화(`GET /api/rooms/{id}/messages`)에서 그 만화 댓글(`postId`가 같은 줄)만 골라 보여 줘요. 받아 둔 댓글이 `commentCount`보다 적고 앞 대화가 더 있으면 "이전 댓글 더 보기"로 대화를 한 쪽(50줄, 초안) 더 받아요. 댓글 달기는 지금처럼 `POST /api/rooms/{id}/messages`에 `postId`를 넣어요. 만화 댓글은 "대화" 탭에도 지금처럼 섞여 보여요
4. **방 안 – 대화:** 오래된 줄이 위예요. "준님이 만화를 올렸어요", "서연님이 이 만화를 좋아해요" 같은 알림 줄과 댓글 말풍선이 섞여 있어요. 대화를 열면 그 방의 받지 않은 글 수가 0이 돼요
5. **멤버 목록:** 멤버마다 닉네임과 프로필 캐릭터 그림, 친구 추가 버튼이 보여요. 방장은 여기서 친구를 더 초대하고, 멤버를 내보내요
6. **나가기:** 방에서 나갈 수 있어요. 방장은 나가기 전에 새 방장을 골라요
7. **방 설정(방장):** 방 이름 바꾸기, 방 해산

### 규칙

- 방 이름은 앞뒤 공백을 빼고 1~20자예요
- 닉네임을 안 정했으면 방을 못 만들어요(409 PROFILE\_REQUIRED)
- 한 사람이 들어갈 수 있는 방은 20개예요. 넘으면 방을 만들거나 초대를 수락할 수 없어요(409 ROOM\_LIMIT, 초안)
- 같은 방 사람과 친구를 끊거나 차단해도 저절로 방에서 빠지지는 않아요. 방은 만든 뒤에는 친구 관계와 따로 가요(차단은 아래 "차단한 사람이 있는 방")
- 올릴 수 있는 것은 내 만화 중 완성된 것(DONE)만이에요. 같은 만화를 같은 방에 두 번 올리면 409 ROOM\_POST\_EXISTS
- 같은 만화를 다른 방에는 올릴 수 있어요
- 올린 사람만 내릴 수 있어요. 내리면 그 만화의 좋아요와 그 만화에 단 댓글, 알림 줄도 같이 지워져요
- 보관함에서 만화를 지우면 그 만화를 올린 방에서도 내려가요(위와 같이 좋아요·댓글도 지움)
- 좋아요는 한 사람이 만화 하나에 한 번이에요. 다시 누르면 취소되고, 취소하면 그 좋아요 알림 줄도 빠져요. 내 만화에도 누를 수 있어요
- 댓글은 앞뒤 공백을 빼고 1~200자예요. 특정 만화에 단 댓글(postId 있음)과 방 전체에 단 댓글(postId 없음)이 있어요
- 나가도 내가 올린 만화와 남긴 글은 남고, 닉네임 대신 "나간 멤버"로 보여요. 마지막 한 명이 나가면 방과 올린 만화 기록, 대화를 모두 지워요(만화 자체는 각자 보관함에 그대로 있어요)
- 탈퇴하면 그 회원이 들어간 방에서 빠지고, 그 회원이 올린 만화(좋아요·댓글 포함)와 남긴 좋아요·댓글을 지워요. 그래서 마지막 한 명이 되면 그 방도 지워져요
- 정지된 계정은 방에 그대로 남고 멤버 목록에 "정지된 계정"으로 보여요. 올린 글도 그대로예요([관리자](admin.md#2-계정-정지))
- 방 멤버끼리는 친구가 아니어도 닉네임·프로필 캐릭터 그림이 보여요. 친구가 아닌 멤버의 캐릭터는 화자에 붙일 수 없어요

### 방장과 초대

- 만든 사람이 처음 방장이에요. 방장은 바뀔 수 있어서 만든 사람과 따로 둬요
- 방장만 친구를 초대해요. 방을 만들 때도 고른 친구에게 초대를 보내요(바로 넣지 않아요). 초대는 푸시로 알려요
- 초대받은 사람이 수락하면 들어오고, 거절하면 초대가 없어져요. 거절해도 방장에게 알리지 않아요
- 방장은 답을 기다리는 초대를 취소할 수 있어요. 거절한 사람도 다시 초대할 수 있어요
- 인원은 멤버와 답을 기다리는 초대를 합쳐서 세요(나 포함 6명까지)
- 방장은 방 이름 바꾸기, 멤버 내보내기, 방 해산을 할 수 있어요
- 내보낸 사람의 글은 "나간 멤버"로 남아요. 내보낸 사람도 다시 초대할 수 있어요
- 방장이 나갈 때는 꼭 새 방장을 골라야 해요. 혼자 남은 방장이 나가면 방이 지워져요
- 해산하면 방의 글·대화를 모두 지우고(각자 보관함의 만화는 남음), 멤버에게 푸시로 알려요

### 차단한 사람이 있는 방

- 같은 방 사람을 차단하면 "방에서 나갈까요?"를 물어요. 남으면 그 사람 글을 숨길지 그대로 볼지 골라요
- 선택은 차단한 사람마다 하나이고, 설정의 차단한 사람 화면에서 언제든 바꿀 수 있어요([친구](friends.md#차단))
- 숨긴 글 자리는 "숨긴 글"로 보여요. 차단당한 사람에게는 알리지 않아요
- 차단한 사람이 있는 방에 초대받으면 수락 전에 알려 주고, 들어가면 같은 설정을 따라요

### 방에서 친구 요청

- 방 멤버 목록에서 사람마다 친구 추가 버튼을 눌러 친구 요청을 보낼 수 있어요
- 이미 친구인 사람은 버튼이 꺼져 있어요. 내가 이미 요청을 보낸 사람은 "요청 보냄"으로 꺼지고, 상대가 나에게 보낸 요청이 있으면 "수락"으로 바뀌어요. 내가 차단한 사람과 나 자신에는 버튼이 없어요
- 친구 코드 대신 그 멤버의 userId로 보내요. 서버는 지금 같은 방에 있는 사람인지 확인하고, 아니면 404예요. 방 밖의 사람에게는 지금처럼 친구 코드로만 보내요([친구](friends.md#2-api))
- 요청을 받은 쪽은 지금 친구 요청과 같이 푸시와 [알림 목록](inbox.md)으로 받아요

### 받지 않은 글(unread)

- 멤버마다 대화를 마지막으로 본 시각을 둬요. 그 뒤에 **다른 사람이** 남긴 대화 줄(만화 올림, 좋아요, 댓글) 수가 unread예요
- `GET /api/rooms/{id}/messages`를 부르면 그 시각을 지금으로 바꿔 unread가 0이 돼요
- 방에 들어온 직후에는 0이에요

## 2. API

모두 `/api`(메인 백엔드)이고 로그인이 필요해요. 방 멤버가 아니면 방이 없는 것과 같은 404 ROOM\_NOT\_FOUND예요.

| 메서드 | 주소 | 하는 일 | 요청 | 응답 |
| --- | --- | --- | --- | --- |
| GET | /api/rooms | 내 방 목록. 마지막 소식이 최근인 방이 위 | - | 200 `{ rooms: List<RoomItem> }` |
| POST | /api/rooms | 방 만들기. 고른 친구에게 초대를 보냄 | RoomCreateRequest | 201 RoomItem |
| GET | /api/rooms/{id} | 방 하나 | - | 200 RoomItem |
| PATCH | /api/rooms/{id} | 방 이름 바꾸기(방장, 초안) | `{ name }` | 200 RoomItem |
| DELETE | /api/rooms/{id} | 방 해산(방장, 초안). 멤버에게 푸시 | - | 204 |
| DELETE | /api/rooms/{id}/members/me | 방 나가기. 방장은 `newOwnerId`를 꼭 보냄(초안). 마지막 한 명이면 방을 지움 | 쿼리 `newOwnerId` | 204 |
| DELETE | /api/rooms/{id}/members/{userId} | 멤버 내보내기(방장, 초안) | - | 204 |
| POST | /api/rooms/{id}/invites | 친구 초대(방장, 초안) | `{ userIds }` | 201 `{ invites: List<RoomInvite> }` |
| DELETE | /api/rooms/{id}/invites/{inviteId} | 초대 취소(방장, 초안) | - | 204 |
| GET | /api/room-invites | 내가 받은 초대 목록(초안) | - | 200 `{ invites: List<RoomInvite> }` |
| POST | /api/room-invites/{inviteId}/accept | 초대 수락(초안) | - | 200 RoomItem |
| DELETE | /api/room-invites/{inviteId} | 초대 거절(초안). 방장에게 알리지 않음 | - | 204 |
| GET | /api/rooms/{id}/posts | 올린 만화 목록. 최근에 올린 것이 앞 | - | 200 `{ posts: List<RoomPost> }` |
| POST | /api/rooms/{id}/posts | 내 만화 올리기 | `{ comicId }` | 201 RoomPost |
| DELETE | /api/rooms/{id}/posts/{postId} | 내가 올린 만화 내리기 | - | 204 |
| PUT | /api/rooms/{id}/posts/{postId}/like | 좋아요. 이미 눌렀어도 200 | - | 200 RoomPost |
| DELETE | /api/rooms/{id}/posts/{postId}/like | 좋아요 취소. 안 눌렀어도 200 | - | 200 RoomPost |
| GET | /api/rooms/{id}/messages | 대화. 오래된 줄이 위. 처음엔 최근 줄부터 주고, 위로 올리면 커서로 더 불러옴(한 번에 50줄은 초안). 부르면 unread가 0 | 쿼리 `cursor` | 200 `{ messages: List<RoomMessage>, nextCursor }` |
| POST | /api/rooms/{id}/messages | 댓글 달기 | RoomCommentRequest | 201 RoomMessage |

- 목록은 방 인원과 만화 수가 작아 한 번에 다 줘요(cursor 없음). 대화만 커서로 나눠 줘요
- 좋아요는 앱이 `likedByMe`를 보고 PUT이나 DELETE를 불러요. 두 번 눌려도 결과가 같아요

**RoomCreateRequest**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| name | String | O | 1~20자(앞뒤 공백을 뺀 길이) |
| memberIds | List\<Long\> | O | 초대할 친구 회원 ID 1~5개(나를 뺌). 같은 ID 두 번, 내 ID는 400 INVALID\_REQUEST. 친구가 아니면 404 USER\_NOT\_FOUND. 6개 이상은 409 ROOM\_MEMBER\_LIMIT |

**RoomItem**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | 방 ID |
| name | String | O | 방 이름 |
| ownerId | Long | O | 방장 회원 ID(초안) |
| members | List\<RoomMember\> | O | 나를 포함한 멤버. 들어온 순서 |
| invites | List\<RoomInvite\> | O | 답을 기다리는 초대(초안). 방장이 아니면 빈 배열 |
| lastActivity | RoomActivity | null 가능 | 마지막 소식. 아직 아무 일도 없으면 null |
| unread | Int | O | 받지 않은 글 수 |
| createdAt | DateTime | O | 방을 만든 시각 |

**RoomMember**: userId(Long), nickname(String), thumbnailUrl(String, null 가능. 그 사람의 프로필 캐릭터 256 그림 10분 서명 주소, 없으면 null), isMe(Boolean), isOwner(Boolean, 초안), suspended(Boolean, 정지된 계정이면 true. 초안), friendState(String, 초안. `FRIEND`·`REQUESTED`(내가 보냄)·`RECEIVED`(상대가 보냄)·`NONE`·`BLOCKED`(내가 차단)·`ME`. 친구 추가 버튼에 써요).

대화와 올린 만화의 사람(`author`, `member`)이 방에서 나간 사람이면 `userId`·`nickname`·`thumbnailUrl`이 null이고 `left`가 true예요(초안). 앱은 "나간 멤버"로 보여요. 내가 숨기기로 한 차단한 사람의 줄은 `hidden`이 true이고 내용(`text`, `post`)이 null이에요(초안). 앱은 "숨긴 글"로 보여요.

**RoomInvite**(초안): id(Long), room(`{ id, name }`), invitedUser(RoomMember, 방장이 보는 초대), invitedBy(RoomMember, 내가 받은 초대), hasBlockedMember(Boolean, 내가 차단한 사람이 그 방에 있으면 true. 수락 전에 알려 줄 때 써요), createdAt(DateTime).

**RoomActivity**: type(String, `POSTED`·`LIKED`·`COMMENTED`), nickname(String, 한 사람의 닉네임), isMe(Boolean), at(DateTime). 문장("준님이 만화를 올렸어요", 나는 "내가 …")은 앱이 만들어요. 푸시 문구를 앱이 만드는 것과 같은 방식이에요.

**RoomPost**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | 올린 만화 ID(만화 ID와 다름) |
| roomId | Long | O | |
| comicId | Long | O | 만화 ID |
| title | String | O | 만화를 만들 때의 카드 요약(ComicResponse의 cardSummary와 같음) |
| imageUrl | String | O | 완성 이미지의 10분 서명 주소. 403이면 목록을 다시 불러요 |
| author | RoomMember | O | 올린 사람 |
| postedAt | DateTime | O | 올린 시각 |
| likeCount | Int | O | |
| likedByMe | Boolean | O | |
| commentCount | Int | O | 이 만화에 단 댓글 수 |

**RoomCommentRequest**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| text | String | O | 1~200자(앞뒤 공백을 뺀 길이) |
| postId | Long | 선택 | 이 방에 올린 만화에 다는 댓글이면 그 RoomPost id. 없거나 내려간 만화면 404 ROOM\_POST\_NOT\_FOUND |

**RoomMessage** (대화 한 줄)

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | |
| type | String | O | `POSTED`(만화를 올림)·`LIKED`(좋아요)·`COMMENT`(댓글) |
| member | RoomMember | O | 한 사람 |
| at | DateTime | O | |
| postId | Long | null 가능 | 어느 만화에 대한 줄인지. 방 전체 댓글이면 null |
| post | RoomPostSummary | null 가능 | postId가 있을 때 `{ id, title, imageUrl }`. 앱이 말풍선 위 캡션에 써요 |
| text | String | null 가능 | COMMENT일 때만 |

### 오류 코드

| 코드 | HTTP | 언제 |
| --- | --- | --- |
| INVALID_REQUEST | 400 | 방 이름·댓글 길이, memberIds가 비었거나 같은 ID·내 ID |
| USER_NOT_FOUND | 404 | 방 만들기: 친구가 아닌 회원 ID |
| ROOM_NOT_FOUND | 404 | 없는 방, 내가 멤버가 아닌 방 |
| ROOM_POST_NOT_FOUND | 404 | 없거나 내려간 만화 |
| COMIC_NOT_FOUND | 404 | 올리기: 내 만화가 아니거나 완성되지 않은 만화 |
| ROOM_POST_NOT_MINE | 403 | 남이 올린 만화를 내리려 함 |
| ROOM_POST_EXISTS | 409 | 같은 만화를 같은 방에 또 올림 |
| ROOM_MEMBER_LIMIT | 409 | 방 만들기·초대: 멤버와 답을 기다리는 초대를 합쳐 나 포함 6명을 넘음 |
| ROOM_LIMIT | 409 | 방 만들기·초대 수락: 이미 방 20개에 들어가 있음(초안) |
| ROOM_OWNER_ONLY | 403 | 방장만 할 수 있는 일(초대·취소·이름 바꾸기·내보내기·해산)을 방장이 아닌 사람이 함(초안) |
| NEW_OWNER_REQUIRED | 409 | 방장이 다른 멤버가 있는데 새 방장을 고르지 않고 나감(초안) |
| ROOM_INVITE_NOT_FOUND | 404 | 없거나 이미 답한 초대, 내 초대가 아님(초안) |
| PROFILE_REQUIRED | 409 | 방 만들기: 닉네임을 아직 안 정함 |

## 3. 푸시 (초안)

| type | 나머지 키 | 보내는 때 | 앱 동작 |
| --- | --- | --- | --- |
| ROOM_INVITE | inviteId, roomId, roomName | 방에 초대받았을 때(방을 만들 때 포함) | 받은 초대 |
| ROOM_COMMENT | roomId, postId, nickname(단 사람 닉네임) | 내가 올린 만화에 다른 사람이 댓글을 달았을 때. 올린 사람에게만 | 그 방의 대화 |
| ROOM_DISBANDED | roomName | 방장이 방을 해산했을 때. 방장 말고 멤버에게 | 방 목록 |

- 문구에 댓글 내용은 넣지 않아요. 수락·거절, 방 대화, 좋아요는 푸시가 없어요
- 이 푸시도 [알림 목록](inbox.md)에 남아요

## 4. DB

[DB](db.md) V8에 있어요. 표는 `rooms`, `room_members`, `room_posts`, `room_post_likes`, `room_messages`예요. 방장·초대는 V9(초안)에 있어요. 나간 멤버는 표를 바꾸지 않고, 글을 남긴 사람이 지금 멤버가 아니면 "나간 멤버"로 보여 줘요([DB](db.md) V9 초안 아래 설명).

## 5. 아직 정할 것

- 들어가는 곳(친구 탭 안 "방" 알약)을 이대로 둘지
- 만화별 댓글 주소(`GET /api/rooms/{id}/posts/{postId}/comments`)를 더할지. 지금은 방 대화를 한 쪽씩 받아 골라서, 오래된 만화는 "이전 댓글 더 보기"를 여러 번 눌러야 해요(Claude 의견: 친구 방은 작아서 나중에, 정하지 않음)

## 6. lolog-api 구현에서 정한 것 (Claude 초안, 검토 필요)

이 명세에 없던 것을 lolog-api를 만들면서 Claude가 정했어요. 사용자·팀이 아직 하나씩 고르지 않았으니 검토가 필요해요. 바꾸면 lolog-api도 같이 고쳐요. (lolog-api PR #59·#60·#61)

| 정하지 않았던 점 | 정한 것 |
| --- | --- |
| 이미 멤버인 사람을 초대 | 400 INVALID\_REQUEST(field `userIds[i]`) |
| 답을 기다리는 초대가 있는 사람을 또 초대 | 새로 만들지 않고 그 초대를 돌려줌. 푸시도 다시 보내지 않음 |
| 초대할 수 있는 사람 | 방장의 친구만(방 만들 때와 같이 404 USER\_NOT\_FOUND) |
| 초대를 수락하기 전에 생긴 대화 줄 | 받지 않은 글에 세지 않음(들어온 시각 뒤의 줄만 셈) |
| 방장이 고른 `newOwnerId`가 이 방의 다른 멤버가 아님 | 400 INVALID\_REQUEST(field `newOwnerId`) |
| 내보내기에서 나를 고르거나 멤버가 아닌 사람 | 나는 400, 멤버가 아니면 404 USER\_NOT\_FOUND |
| 마지막 소식(`lastActivity`)의 사람이 나간 멤버 | `nickname` null |
| ROOM\_COMMENT를 받는 사람 | 올린 사람이 지금 그 방 멤버일 때만 |
| 대화 커서 | 받은 것 중 가장 오래된 줄의 ID. 커서를 줘도 읽음 처리함 |
| 숨긴 사람이 올린 만화(RoomPost) | `hidden` true, `title`·`imageUrl` null(대화 줄과 같게) |
| 나간 멤버의 `friendState` | null |
