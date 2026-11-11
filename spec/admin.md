# 관리자 (초안)

관리자 계정으로 공지사항을 쓰고, 계정을 정지·탈퇴시키고, 사용자가 보낸 오류 신고·의견을 보는 기능이에요. 사용자가 오류 신고·의견을 보내는 것도 여기 있어요. 공지사항을 사용자에게 보여 주는 것은 [알림 목록과 공지사항](inbox.md)에 있어요.

- 관리자 화면은 앱에서 관리자 계정으로 들어가면 보여요. 따로 웹 화면은 없어요
- 설계부터 하고, 앱 쪽과 공유한 뒤에 구현해요
- 이름이 "(초안)"인 주소·필드·코드는 구현하면서 바뀌어도 돼요

## 1. 관리자 구분

- 인증 서비스의 회원 정보에 `role`(`USER`·`ADMIN`)을 두고, 액세스 토큰에 `role` 클레임을 넣어요. 메인 백엔드는 DB를 보지 않고 토큰만으로 관리자 주소를 막아요
- 관리자에서 빼도 이미 받은 액세스 토큰은 만료(30분)될 때까지 관리자예요. 메인 백엔드(`/api/admin/**`)가 그래요. 인증 서비스(`/auth/admin/**`)는 토큰의 `role`이 아니라 DB를 봐서 바로 막혀요
- 관리자는 보통 회원처럼 가입한 뒤 DB에서 `role`을 직접 바꿔 정해요. 관리자를 지정하는 화면은 없어요
- 앱은 `GET /auth/me`의 `role`을 보고 관리자 화면을 열지 정해요(초안)
- 관리자 주소는 메인 백엔드 `/api/admin/**`, 인증 서비스 `/auth/admin/**`예요(초안). 관리자가 아니면 403 `FORBIDDEN`이에요. 액세스 토큰이 없거나 틀리면 먼저 401(`/auth`는 `INVALID_TOKEN`)이고, 관리자 확인이 요청 값 검사(400)보다 먼저예요

## 2. 계정 정지

- 관리자가 기한을 1개월·3개월·6개월·영구 중에서 골라요. 이유는 관리자가 직접 적어요(이유 목록 없음, 1~200자)
- 기한이 지나면 저절로 풀려요. 관리자가 먼저 풀 수도 있어요
- 기한은 정지한 때부터 달력으로 세요(11월 11일에 1개월이면 12월 11일까지). 이미 정지된 계정을 다시 정지하면 지금부터 새 기한·이유로 바꿔요
- 정지되지 않은 계정을 풀어도 204예요
- 신고 기능은 없어요
- 정지된 계정은 로그인(이메일·소셜)과 토큰 갱신이 403 `ACCOUNT_SUSPENDED`예요. 정지할 때 그 회원의 리프레시 토큰을 모두 지워요. 이메일 로그인은 비밀번호가 맞을 때만 이 코드이고, 틀리면 지금처럼 401 `INVALID_CREDENTIALS`예요(틀린 비밀번호로 정지 여부를 알 수 없게)
- 정지 전에 받은 액세스 토큰은 탈퇴와 같은 방법으로 막아요. 인증 서비스가 Redis에 `lolog:auth:suspended:{userId}`를 남기고, 메인 백엔드가 요청마다 탈퇴 키와 함께 봐요. 이 키가 있으면 /api는 401 `UNAUTHORIZED`, /auth는 401 `INVALID_TOKEN`이에요(탈퇴한 회원의 토큰과 같은 처리)
- 이 키는 탈퇴 키와 같이 31분 동안 둬요. 정지하면 새 토큰이 생기지 않으니, 정지 전에 받은 액세스 토큰이 만료될 때까지만 있으면 돼요. 정지를 풀면 바로 지워요
- 정지된 사람의 데이터는 그대로 둬요. 같이 있던 방에는 남고, 멤버 목록에 "정지된 계정"으로 보여요. 올린 글도 그대로예요
- 그래서 인증 서비스가 정지·해제할 때 메인 백엔드의 내부 주소를 불러 알려요(탈퇴 때 `DELETE /internal/users/{id}`를 부르는 것과 같은 방식)
- 메인 백엔드가 204로 답하지 않으면 정지·해제는 그대로 되고(관리자에게 204), 인증 서비스가 1분마다 204가 올 때까지 다시 불러요. 다시 부를 때는 그때의 상태를 알려요(정지했다가 바로 풀었으면 해제만). 그래서 메인 백엔드의 두 주소는 여러 번 불려도 같은 결과여야 해요
- 정지된 사람의 앱은 로그인 화면에서 "정지된 계정이에요"를 보여 줘요(초안)
- 관리자 자신과 다른 관리자는 정지하거나 탈퇴시킬 수 없어요(초안)
- 확인 순서: 액세스 토큰(401) → 관리자인지(403) → 요청 본문(400, field `duration`·`reason`) → 대상 회원이 있는지(404 `USER_NOT_FOUND`, 탈퇴한 회원 포함) → 대상이 관리자인지(403 `FORBIDDEN`)

## 3. 관리자가 탈퇴시키기

- 본인 탈퇴(`DELETE /auth/me`)와 같은 처리를 해요. 회원 정보는 소프트 삭제하고, 메인 백엔드 데이터는 모두 지워요
- 본인 탈퇴와 다른 점: 그 계정에 연결된 이메일과 소셜 계정(구글·네이버)으로는 다시 가입할 수 없어요. 인증 서비스가 그 이메일과 소셜 계정 번호를 따로 적어 두고 가입할 때 걸러요
- 다른 이메일이나 다른 소셜 계정으로 가입하는 것은 막을 수 없어요
- 본인이 탈퇴한 계정은 지금처럼 같은 이메일로 다시 가입할 수 있어요

## 4. 공지사항 관리

- 모든 사용자에게 같은 글이 보여요. 제목(1~100자)과 본문이 있어요
- 새 공지사항은 푸시로 알리지 않아요

## 5. 오류 신고와 의견

- 사용자는 설정 화면에서 보내요. 종류는 오류 신고(`BUG`)와 의견(`OPINION`) 두 가지예요
- 한 사람이 하루에 10번까지 보낼 수 있어요. 넘으면 429 `TOO_MANY_REQUESTS`예요
- 관리자는 관리자 화면에서 목록을 보고 처리했다고 표시해요(`OPEN`·`DONE`)
- 보낸 사람이 탈퇴해도 글은 남겨요. 보낸 사람만 비워서 "탈퇴한 사용자"로 보여요. 글과 앱 버전, 기기 이름은 남아요. 탈퇴하면 데이터를 모두 지우는 원칙(FR-AUTH-05)에서 이것만 예외예요
- 탈퇴해도 보낸 의견은 남는다는 걸 의견 보내기 화면에 적어요(초안)

## 6. 계정 목록

- 회원의 이메일과 가입일은 인증 DB에, 닉네임과 친구 코드는 메인 백엔드 DB에 있어요. 그래서 목록에는 이메일만 보여 주고, 한 사람을 누르면 메인 백엔드에 물어 닉네임과 친구 코드를 받아요
- 목록은 이메일 일부로 찾을 수 있고, 정렬을 고를 수 있어요(가입일 최신순·오래된순·이메일순, 기본은 최신순. 정렬 종류는 초안)
- 찾기는 대소문자를 가리지 않고, `%`·`_`도 글자 그대로 찾아요
- 탈퇴한 계정(본인 탈퇴, 관리자가 탈퇴시킴)은 목록에 없어요
- 한 번에 `size`개(1~50, 기본 20)씩 받아요. `cursor`는 앞 응답의 `nextCursor`를 그대로 넣고, 앱은 값의 뜻을 해석하지 않아요. 다음 페이지도 같은 `sort`·`q`로 불러요

## 7. API

### 인증 서비스 (/auth)

| 메서드 | 주소 | 하는 일 | 요청 | 응답 |
| --- | --- | --- | --- | --- |
| GET | /auth/me | 응답에 `role` 추가 | - | 200 MeResponse |
| POST | /auth/login, /auth/social/{provider}, /auth/refresh | 정지된 계정이면 403 ACCOUNT\_SUSPENDED | - | - |
| GET | /auth/admin/users | 계정 목록 | 쿼리 `q`(이메일 일부, 254자 이하), `sort`(`NEWEST`·`OLDEST`·`EMAIL`, 초안), `cursor`, `size`(1~50, 기본 20) | 200 `{ users: List<AdminUserItem>, nextCursor }` |
| POST | /auth/admin/users/{id}/suspend | 정지 | SuspendRequest | 204 |
| DELETE | /auth/admin/users/{id}/suspend | 정지 풀기 | - | 204 |
| DELETE | /auth/admin/users/{id} | 탈퇴시키기. 같은 이메일·소셜 계정으로 다시 가입하지 못하게 막음 | - | 204 |

**AdminUserItem**: userId(Long), email(String), createdAt(DateTime), role(String), suspendedAt(DateTime, null 가능), suspendedUntil(DateTime, null 가능. 영구 정지면 null), suspendReason(String, null 가능). 정지 세 칸은 지금 정지 중일 때만 채우고, 정지가 아니거나 기한이 지났으면 모두 null이에요. 틀린 `sort`·`size`·`cursor`·`q`는 400 `INVALID_REQUEST`이고 field는 그 쿼리 이름이에요.

**SuspendRequest**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| duration | String | O | `MONTH_1`·`MONTH_3`·`MONTH_6`·`PERMANENT`(이름은 초안) |
| reason | String | O | 1~200자(앞뒤 공백을 뺀 길이) |

### 메인 백엔드 (/api)

| 메서드 | 주소 | 하는 일 | 요청 | 응답 |
| --- | --- | --- | --- | --- |
| POST | /api/admin/notices | 공지사항 쓰기 | `{ title, body }` | 201 Notice |
| PATCH | /api/admin/notices/{id} | 고치기 | `{ title, body }`(바꾼 것만) | 200 Notice |
| DELETE | /api/admin/notices/{id} | 지우기 | - | 204 |
| GET | /api/admin/users/{id} | 그 회원의 닉네임과 친구 코드 | - | 200 `{ nickname, friendCode }`(정하기 전이면 null) |
| POST | /api/feedback | 오류 신고·의견 보내기(관리자 아니어도 됨) | FeedbackRequest | 201 |
| GET | /api/admin/feedback | 의견 목록. 최근 것이 위 | 쿼리 `status`(`OPEN`·`DONE`, 없으면 전부), `cursor` | 200 `{ items: List<FeedbackItem>, nextCursor }` |
| PATCH | /api/admin/feedback/{id} | 처리 표시 | `{ status }` | 200 FeedbackItem |

**Notice**: id(Long), title(String), body(String), createdAt(DateTime), updatedAt(DateTime).

**FeedbackRequest**

| 필드 | 타입 | 필수 | 제한 |
| --- | --- | --- | --- |
| category | String | O | `BUG`·`OPINION` |
| text | String | O | 1~1000자(앞뒤 공백을 뺀 길이) |
| appVersion | String | O | 앱 버전 |
| deviceModel | String | O | 기기 이름 |

**FeedbackItem**: id(Long), userId(Long, null 가능. 보낸 사람이 탈퇴했으면 null), category, text, appVersion, deviceModel, status(`OPEN`·`DONE`), createdAt(DateTime).

### /internal (인증 서비스 → 메인 백엔드, 초안)

| 메서드 | 주소 | 하는 일 |
| --- | --- | --- |
| PUT | /internal/users/{id}/suspension | 정지됐다고 알림. 본문 `{ suspendedUntil }`(영구면 null). 방 멤버 목록의 "정지된 계정" 표시에 써요. 기한이 지나면 메인 백엔드가 스스로 풀린 것으로 봐요. 204 |
| DELETE | /internal/users/{id}/suspension | 관리자가 먼저 풀었다고 알림. 204 |

### 오류 코드

| 코드 | HTTP | 언제 |
| --- | --- | --- |
| INVALID_REQUEST | 400 | 제목·본문·이유·의견 길이, 틀린 duration·category·status |
| FORBIDDEN | 403 | 관리자가 아닌데 관리자 주소를 부름. 관리자를 정지·탈퇴시키려 함(초안) |
| ACCOUNT_SUSPENDED | 403 | 정지된 계정으로 로그인·소셜 로그인·토큰 갱신 |
| ACCOUNT_BANNED | 409 | 관리자가 탈퇴시킨 계정의 이메일·소셜 계정으로 가입·소셜 로그인(이름은 초안) |
| USER_NOT_FOUND | 404 | 없는 회원 |
| NOTICE_NOT_FOUND | 404 | 없는 공지사항 |
| FEEDBACK_NOT_FOUND | 404 | 없는 의견 |
| TOO_MANY_REQUESTS | 429 | 의견을 하루에 10번 넘게 보냄 |

## 8. DB

[DB](db.md)에 있어요. 인증 서비스 V4(`users`에 `role`·정지 열, 다시 가입을 막는 표), 메인 백엔드 V9(`notices`, `feedback`, 정지 표시)예요.
