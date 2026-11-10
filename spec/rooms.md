# 친구와 같이 쓰는 방 (초안)

친구 몇 명과 방을 만들어 내 만화를 올리고, 좋아요와 댓글로 이야기하는 기능이에요. 방은 [친구](friends.md)끼리 만들어요.

> 이 문서는 초안이에요. 앱 PR(LOLog-app #53)의 화면·목업 규칙을 바탕으로 썼고, 정해지지 않은 것은 맨 아래 "아직 정할 것"에 모았어요.

- 방은 나를 포함해 2~6명이에요. 방을 만들 때 친구를 골라 넣어요
- 방에는 내 만화를 올려요. 올린 만화는 쌓이고(덮어쓰지 않음), 한 사람이 올리는 수에 제한이 없어요
- 올린 만화에 좋아요를 누르고 댓글을 달 수 있어요
- 새 글 푸시는 보내지 않아요. 방 목록에 받지 않은 글 수(unread)를 보여 줘요
- 음성은 방에 나가지 않아요. 만화 그림만 보여요([순간 다시 듣기](replay.md): 음성은 카드 주인만)

## 1. 화면 흐름

화면을 어디에 둘지는 앱 담당과 정해요. 지금 앱은 친구 탭 맨 위 "방" / "친구" 알약에서 들어가요.

1. **방 목록:** 마지막 소식이 최근인 방이 위에 와요. 방마다 이름, 멤버 그림, 마지막 소식, 받지 않은 글 수가 보여요
2. **방 만들기:** 방 이름을 적고 친구를 1~5명 골라요
3. **방 안 – 만화:** 올린 만화가 2열 격자로 보여요(최근에 올린 것이 앞). 누르면 크게 보고, 꾹 누르면 좋아요·댓글·저장·내리기가 나와요
4. **방 안 – 대화:** 오래된 줄이 위예요. "준님이 만화를 올렸어요", "서연님이 이 만화를 좋아해요" 같은 알림 줄과 댓글 말풍선이 섞여 있어요. 대화를 열면 그 방의 받지 않은 글 수가 0이 돼요
5. **나가기:** 방에서 나갈 수 있어요

### 규칙

- 방 이름은 앞뒤 공백을 빼고 1~20자예요
- 방을 만들 때 고르는 사람은 내 친구여야 해요(친구 1~5명, 나 포함 2~6명). 6명을 넘으면 409 ROOM\_MEMBER\_LIMIT
- 방을 만든 뒤에는 멤버를 더 초대할 수 없어요(이번 범위 밖)
- 같은 방 사람과 친구를 끊거나 차단해도 방에는 그대로 남아요. 방은 친구 관계와 따로 가요
- 올릴 수 있는 것은 내 만화 중 완성된 것(DONE)만이에요. 같은 만화를 같은 방에 두 번 올리면 409 ROOM\_POST\_EXISTS
- 같은 만화를 다른 방에는 올릴 수 있어요
- 올린 사람만 내릴 수 있어요. 내리면 그 만화의 좋아요와 그 만화에 단 댓글, 알림 줄도 같이 지워져요
- 보관함에서 만화를 지우면 그 만화를 올린 방에서도 내려가요(위와 같이 좋아요·댓글도 지움)
- 좋아요는 한 사람이 만화 하나에 한 번이에요. 다시 누르면 취소되고, 취소하면 그 좋아요 알림 줄도 빠져요. 내 만화에도 누를 수 있어요
- 댓글은 앞뒤 공백을 빼고 1~200자예요. 특정 만화에 단 댓글(postId 있음)과 방 전체에 단 댓글(postId 없음)이 있어요
- 마지막 한 명이 나가면 방과 올린 만화 기록, 대화를 모두 지워요(만화 자체는 각자 보관함에 그대로 있어요)
- 탈퇴하면 그 회원이 들어간 방에서 빠지고, 그 회원이 올린 만화(좋아요·댓글 포함)와 남긴 좋아요·댓글을 지워요. 그래서 마지막 한 명이 되면 그 방도 지워져요

### 받지 않은 글(unread)

- 멤버마다 대화를 마지막으로 본 시각을 둬요. 그 뒤에 **다른 사람이** 남긴 대화 줄(만화 올림, 좋아요, 댓글) 수가 unread예요
- `GET /api/rooms/{id}/messages`를 부르면 그 시각을 지금으로 바꿔 unread가 0이 돼요
- 방에 들어온 직후(방을 만든 때)에는 0이에요

## 2. API

모두 `/api`(메인 백엔드)이고 로그인이 필요해요. 방 멤버가 아니면 방이 없는 것과 같은 404 ROOM\_NOT\_FOUND예요.

| 메서드 | 주소 | 하는 일 | 요청 | 응답 |
| --- | --- | --- | --- | --- |
| GET | /api/rooms | 내 방 목록. 마지막 소식이 최근인 방이 위 | - | 200 `{ rooms: List<RoomItem> }` |
| POST | /api/rooms | 방 만들기 | RoomCreateRequest | 201 RoomItem |
| GET | /api/rooms/{id} | 방 하나 | - | 200 RoomItem |
| DELETE | /api/rooms/{id}/members/me | 방 나가기. 마지막 한 명이면 방을 지움 | - | 204 |
| GET | /api/rooms/{id}/posts | 올린 만화 목록. 최근에 올린 것이 앞 | - | 200 `{ posts: List<RoomPost> }` |
| POST | /api/rooms/{id}/posts | 내 만화 올리기 | `{ comicId }` | 201 RoomPost |
| DELETE | /api/rooms/{id}/posts/{postId} | 내가 올린 만화 내리기 | - | 204 |
| PUT | /api/rooms/{id}/posts/{postId}/like | 좋아요. 이미 눌렀어도 200 | - | 200 RoomPost |
| DELETE | /api/rooms/{id}/posts/{postId}/like | 좋아요 취소. 안 눌렀어도 200 | - | 200 RoomPost |
| GET | /api/rooms/{id}/messages | 대화. 오래된 줄이 위, 최근 200줄까지. 부르면 unread가 0 | - | 200 `{ messages: List<RoomMessage> }` |
| POST | /api/rooms/{id}/messages | 댓글 달기 | RoomCommentRequest | 201 RoomMessage |

- 목록은 방 인원과 만화 수가 작아 한 번에 다 줘요(cursor 없음). 대화만 최근 200줄로 잘라요
- 좋아요는 앱이 `likedByMe`를 보고 PUT이나 DELETE를 불러요. 두 번 눌려도 결과가 같아요

**RoomCreateRequest**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| name | String | O | 1~20자(앞뒤 공백을 뺀 길이) |
| memberIds | List\<Long\> | O | 나를 뺀 친구 회원 ID 1~5개. 같은 ID 두 번, 내 ID는 400 INVALID\_REQUEST. 친구가 아니면 404 USER\_NOT\_FOUND. 6개 이상은 409 ROOM\_MEMBER\_LIMIT |

**RoomItem**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | 방 ID |
| name | String | O | 방 이름 |
| members | List\<RoomMember\> | O | 나를 포함한 멤버. 들어온 순서 |
| lastActivity | RoomActivity | null 가능 | 마지막 소식. 아직 아무 일도 없으면 null |
| unread | Int | O | 받지 않은 글 수 |
| createdAt | DateTime | O | 방을 만든 시각 |

**RoomMember**: userId(Long), nickname(String), thumbnailUrl(String, null 가능. 그 사람의 프로필 캐릭터 256 그림 10분 서명 주소, 없으면 null), isMe(Boolean).

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
| ROOM_MEMBER_LIMIT | 409 | 방 만들기: 나 포함 6명을 넘음 |
| PROFILE_REQUIRED | 409 | 방 만들기: 닉네임을 아직 안 정함 |

## 3. DB

[DB](db.md) V8에 있어요. 표는 `rooms`, `room_members`, `room_posts`, `room_post_likes`, `room_messages`예요.

## 4. 아직 정할 것

- 방에서 나간 사람이 올린 만화와 남긴 좋아요·댓글을 남길지(지금 초안: 남김. 닉네임은 그대로 보임)
- 멤버 더 초대하기, 방 이름 바꾸기
- 대화 200줄 넘을 때 더 불러오기
- 새 글 푸시
- 앱에서 정한 "초대 링크 대신 친구 고르기", 들어가는 곳(친구 탭 안 "방" 알약)을 이대로 둘지
