# 카드 영상 (초안)

카드로 30초 안팎의 세로 영상을 만드는 기능이에요(FR-EXT-02). 4컷만화는 그대로 두고 더하는 기능이에요. AI 쪽에서 쓴 제안이고, 백엔드·앱 담당과 맞춰서 확정해요.

- 대사는 화자 목소리의 TTS로 읽고, 화면에는 자막으로 넣어요. 말풍선은 없어요
- 카드 구간의 실제 음성([음성 저장](audio.md))은 영상에 쓰지 않아요
- 그림은 장면마다 FLUX로 그리고, 확대·이동과 장면 전환 효과는 워커가 코드로 넣어요
- 영상은 카드 주인만 봐요. 외부 공유는 쇼츠 화면의 "내보내기"로 해요([9장](#9-내보내기와-공유-설계_v3-34번-탭), 정함 11-12, 지원). 방에 올리기는 이번 범위가 아니에요. 일정 때문에 뺀 것이고, 방 글이 만화 그림 한 장 기준이라 영상을 붙이려면 방 글 형식과 재생 화면을 같이 바꿔야 해요. 나중에 붙일 수 있어요

## 1. 만드는 흐름

작업 하나가 Jetson 두 대를 차례로 거쳐요. TTS는 3호, 그림은 2호에 있어서예요. 두 대는 서로 직접 주고받지 않고 메인 백엔드를 거쳐요.

```
앱: POST /api/cards/{id}/videos                         → 영상 QUEUED, lolog:video 큐에 넣음
3호: GET  /internal/videos/{id}/source                  → VOICE
     영상 콘티(1호 LLM) → 대사마다 TTS → 한 파일로 이어 붙임
     POST /internal/videos/{id}/storyboard              (콘티 + 대사별 시작·끝 시각)
     POST /internal/videos/{id}/audio                   → DRAWING, lolog:comic 큐에 kind "video"로 넣음
2호: GET  /internal/videos/{id}/storyboard, .../audio
     장면 그림(FLUX) → 움직임·전환·자막 → mp4
     POST /internal/videos/{id}/video                   → DONE, 푸시
```

| 상태 | 뜻 | 바뀌는 때 |
| --- | --- | --- |
| `QUEUED` | 대기 | 요청을 받았을 때 |
| `VOICE` | 콘티를 짜고 목소리를 만드는 중 | 3호가 source를 가져갔을 때 |
| `DRAWING` | 그림을 그리고 영상으로 합치는 중 | audio를 받았을 때 |
| `DONE` | 완성 | video를 받았을 때. 푸시 `VIDEO_DONE` |
| `FAILED` | 실패 | fail을 받았을 때. 푸시 `VIDEO_FAILED` |

걸리는 시간은 20~25초 영상 기준으로 3호 약 3분(콘티 10초 안팎 + TTS 2분 30초), 2호 약 3~4분(그림 6장 + 합치기)을 예상해요. Jetson 3호 TTS는 실제 길이의 5.4배가 걸렸어요([측정](https://github.com/LOLog-2026/LOLog-ai/tree/develop/experiments/261111-video-tts)). 2호는 만화 컷 한 장 23초(참조 그림 1장이면 26초)로 짐작한 값이고, 영상 합치기는 아직 Jetson에서 재지 않았어요. 연구실 서버(3090)에서는 한 편에 약 1분이었어요.

## 2. 앱 API (/api)

만화 API와 같은 모양이에요. 로그인이 필요하고 내 것만 보여요.

| 메서드 | 주소 | 하는 일 | 응답 |
| --- | --- | --- | --- |
| POST | /api/cards/{id}/videos | 영상 요청. 그 카드에 만드는 중인 영상이 있으면 409 `VIDEO_IN_PROGRESS`, 한도(카드 하나에 실패가 아닌 영상 2개, 지운 영상도 셈)를 넘으면 409 `VIDEO_LIMIT`. 남은 수는 CardDetail의 `videoRemaining`(0~2)으로 봐요(정함 11-13, 지원) | 202 `{ videoId, status }` |
| GET | /api/videos | 내 영상 목록. 앱 컨텐츠 미디어 탭이 쓰고, 4컷 목록(`GET /api/comics`)과 같은 모양이에요. 쿼리 `cursor`, `size`(기본 20, 1~50), `sort`(`latest` 기본·`oldest`) | 200 `{ videos: List<VideoResponse>, nextCursor }` |
| GET | /api/videos/{id} | 영상 상태·결과 | VideoResponse |
| DELETE | /api/videos/{id} | 영상 지우기 | 204 |

**VideoResponse**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | |
| cardId | Long | null 가능 | 카드를 지웠으면 null |
| cardSummary | String | O | 영상을 요청할 때의 카드 요약(목록 제목). 카드를 지워도 남아요. ComicResponse의 `cardSummary`와 같아요 |
| status | String | O | `QUEUED`·`VOICE`·`DRAWING`·`DONE`·`FAILED` |
| videoUrl | String | null 가능 | `DONE`일 때만. 10분 동안 열리는 서명 주소(mp4) |
| thumbnailUrl | String | null 가능 | `DONE`일 때만. 첫 장면 그림(목록용) |
| durationMs | Long | null 가능 | `DONE`일 때만 |
| createdAt | String | O | |

- 카드 자세히(`GET /api/cards/{id}`)에 `videos[]`(id, status)를 더해요. `comics[]`와 같은 방식이에요
- 카드 자세히에 `videoRemaining`(Int, 0~2)도 더해요. 이 카드로 더 만들 수 있는 쇼츠 수이고 2 − 실패가 아닌 쇼츠 수(지운 쇼츠도 셈)예요. `comicRemaining`과 같은 방식이라 앱이 `videos[]`로 세지 않아요(정함 11-13, 지원)
- 목록의 `cursor`는 정렬과 상관없이 마지막으로 받은 영상 ID라서, 다음 페이지도 처음과 같은 `sort`로 불러요(만화 목록과 같음). 틀린 `sort`·`cursor`·`size`는 400(field는 그 쿼리 이름)
- 미디어 탭은 4컷 탭처럼 목록에 `thumbnailUrl`(DONE일 때만, 10분 서명 주소)과 `cardSummary`, 상태를 보여 주고, 누르면 `GET /api/videos/{id}`로 새 `videoUrl`을 받아 재생해요. 주소가 403이면 다시 조회해요
- 푸시: `VIDEO_DONE`·`VIDEO_FAILED`, 데이터는 `videoId`, `cardId`. 문구에 대사를 넣지 않아요(FR-NOTI-03)

## 3. 워커 API (/internal)

`X-Internal-Token`을 붙여요. GET이 아닌 것은 모두 204예요.

| 메서드 | 주소 | 누가 | 하는 일 | 본문 |
| --- | --- | --- | --- | --- |
| GET | /internal/videos/{id}/source | 3호 | 카드 내용 받기. 부르면 `VOICE` | → 200 VideoSource |
| POST | /internal/videos/{id}/storyboard | 3호 | 영상 콘티 보내기 | 영상 콘티 JSON(5장) |
| POST | /internal/videos/{id}/audio | 3호 | 대사 음성 올리기. 받으면 `DRAWING`, 2호 큐에 넣음 | multipart `file`(audio/wav) |
| GET | /internal/videos/{id}/storyboard | 2호 | 3호가 보낸 콘티 받기. 그림이 붙은 인물(`characters[].ref`가 있는 인물)에는 `imageUrl`(새로 서명한 10분 주소)을 붙여 줘요 | → 200 영상 콘티 JSON |
| GET | /internal/videos/{id}/audio | 2호 | 3호가 올린 음성 받기 | → 200 wav 바이트 |
| POST | /internal/videos/{id}/video | 2호 | 완성 영상 올리기. 받으면 `DONE`, 푸시 | multipart `file`(video/mp4), `thumbnail`(image/jpeg), `durationMs` |
| POST | /internal/jobs/{id}/fail | 둘 다 | 실패 알리기(이미 있는 API) | JobFailRequest |

- **VideoSource**는 ComicSource와 같은 모양이에요(`comicId` 자리에 `videoId`). 화자의 `character`(`ref`, `appearance`, `imageUrl`)에 `voice`를 하나 더해요. 프로필 캐릭터면 사용자가 고른 목소리(기본 화자 id 또는 `"self"`, 안 골랐으면 null), 기본 화자면 null이어도 돼요(워커가 `ref`로 알아요). [화자 그림](characters.md)의 `characterVoice`는 이 `character.voice`를 말해요
- **imageUrl**: 2호가 그림을 그릴 때 화자 그림을 참조로 넣어 만화처럼 같은 얼굴로 그려요. 3호가 source를 받은 때와 2호가 그리는 때가 몇 분 떨어져 있어 source의 주소는 만료돼요. 그래서 storyboard를 돌려줄 때 새로 서명해 붙여 주세요. 콘티 안의 `ref`(`default:young-f-02`, `profile:42`)로 찾으면 돼요. 없으면 외모 글로만 그려요
- 영상이 없거나 이미 끝났으면 404 `VIDEO_NOT_FOUND`예요. 워커는 다시 시도하지 않고 그 작업을 멈춰요(만화와 같아요)
- **파일 크기**: 음성은 wav 한 개(24kHz 모노, 60초면 약 2.9MB), 영상은 mp4 한 개(720×1280, 60초면 10MB 안팎)예요. `/internal`은 SSH 터널로 메인 백엔드에 바로 닿으니 Nginx 5MB 한도와는 상관없고, 메인 백엔드의 multipart 한도는 **파일 하나 20MB, 요청 전체 25MB**로 해 주세요. `/internal/videos/{id}/video`는 mp4와 썸네일을 한 요청에 올려서, 요청 전체 한도는 25MB예요(lolog-api `application.yml`의 `max-request-size`)
- 음성 파일은 영상이 `DONE`·`FAILED`가 되면 지워도 돼요. 영상을 다시 만들 때는 처음부터 다시 해요

## 4. 큐 메시지

| 스트림 | 읽는 곳 | 메시지 | 넣는 때 |
| --- | --- | --- | --- |
| `lolog:video` (새로 만듦) | 3호 | `{ jobId, videoId, cardId }` | 영상 요청을 받았을 때 |
| `lolog:comic` (있는 것) | 2호 | `{ jobId, kind: "video", videoId, cardId }` | audio를 받았을 때 |

- 2호는 지금도 `kind`로 comic과 avatar를 나눠요. `"video"`를 하나 더해요
- 2호에서 만화와 영상은 같은 큐(`lolog:comic`)에 먼저 온 순서로 서요. 영상 하나가 5~6분을 잡으니 만화가 그만큼 밀릴 수 있어요
- 두 단계의 `jobId`는 따로 줘요. 실패 횟수는 워커가 `jobId`로 세니, 단계마다 3번까지 다시 해 볼 수 있어요
- 컨슈머 그룹(`jetson-workers`), 다시 시도, 같은 메시지가 두 번 오는 경우는 만화와 같은 규칙이에요

## 5. 영상 콘티

3호가 만들어 보내고, 2호가 그대로 받아 그려요. 메인 백엔드는 저장했다가 돌려주기만 하면 돼요(검사는 `version`이 1, `videoId`·`cardId`가 주소와 같음 정도). 자세한 형식은 AI 레포에서 코드와 같이 정한 뒤 [콘티 형식](storyboard.md)처럼 따로 문서로 올려요. 대략 이런 모양이에요.

```json
{
  "version": 1, "videoId": 7, "cardId": 12,
  "size": { "width": 720, "height": 1280, "fps": 30 }, "durationMs": 28400,
  "place": { "text": "치킨집", "prompt": "a small fried chicken restaurant" },
  "characters": [ { "key": "화자1", "name": "민수", "appearance": "a young man ...", "ref": "default:young-m-01",
                    "role": "teller", "gender": "m", "age": "young", "voice": "young-m-01" } ],
  "scenes": [
    { "index": 1, "startMs": 0, "endMs": 5200, "flashback": true, "scene": "...", "cast": [ { "key": "화자1", "position": "left" } ],
      "motion": "zoom_in", "transition": "fade",
      "lines": [ { "speaker": "화자1", "text": "양념 반 후라이드 반", "startMs": 300, "endMs": 2400, "sourceLineIds": [101] } ] }
  ]
}
```

- 장면은 4~8개, 전환 효과와 움직임은 정해 둔 목록에서 LLM이 골라요
- `role`은 이야기를 들려주는 사람(`teller`)과 듣는 사람(`listener`), `flashback`은 들려주는 일을 그린 장면이에요. 회상 장면에는 듣는 사람을 그리지 않아요
- `voice`는 3호가 고른 목소리(기본 화자 id)예요. 백엔드가 저장할 필요는 없고, 콘티째 돌려주면 돼요
- 대사의 `startMs`·`endMs`는 음성 파일 안의 위치예요. 자막도 이 시각에 맞춰 나와요

## 6. 목소리

- 기본 화자 24명에게 목소리가 하나씩 있어요([AI 레포 #27](https://github.com/LOLog-2026/LOLog-ai/issues/27)). 화자에 기본 화자가 붙어 있으면(`ref`가 `default:young-f-02`) 그 목소리로 읽어요
- 화자에 그림이 안 붙어 있으면 콘티에서 정한 나이대·성별(`age`, `gender`)에 맞는 기본 목소리를 워커가 골라요. 카드 ID로 묶음의 4명 중 하나를 정해요
- **프로필 캐릭터(`profile:42`)**: 사진으로 새 목소리를 만들지 않고 기본 목소리 24개 중 하나나, 내 목소리(`"self"`)를 써요. `voice`가 `"self"`면 워커가 `GET /internal/users/{userId}/voice-clone/sample`(초안)로 그 사람의 원본 녹음을 받아 참조 음성으로 써요. 친구 쇼츠 동의(③)를 안 했으면 친구가 만드는 쇼츠에서는 기본 화자 목소리로 읽어요([내 목소리](characters.md#내-목소리-fr-char-05), 정함 11-14). 그래서 "그림으로 목소리 만들기" 요청은 필요 없어요. 프로필 캐릭터의 `appearance`에는 나이대·성별이 없어서(사진만 받기로 함), 아래 가로 정했어요

| 방법 | 백엔드·앱이 할 일 | 장단점 |
| --- | --- | --- |
| **가. 사용자가 고름 (정함)** | 프로필 캐릭터에 `voice`(기본 화자 id, null 가능) 필드. 앱이 24개를 미리 듣고 고르는 화면. SourceCharacter에 `voice`를 실어 보냄 | 본인 목소리 느낌을 직접 고르니 틀릴 일이 없어요. 화면이 하나 늘어요 |

- **프로필 캐릭터인데 목소리를 안 고른 경우(`voice` null)**: 백엔드는 null 그대로 보내면 돼요. 워커가 콘티의 나이대·성별 묶음에서 회원 ID로 하나를 골라요. 그래서 그 사람은 어느 카드에서나 같은 목소리예요. 사진만 받아 성별을 모르니 틀릴 수 있어서, 앱에서 목소리를 고르도록 안내해 주면 좋아요
- 미리 듣기용 인사 클립(wav 24개, 합계 3.1MB)은 기본 화자 그림처럼 저장소에 올려 두고 주소를 내려 주면 돼요([화자 그림](characters.md) `voiceUrl`). 파일은 AI 쪽에서 드려요. 정확도 평가([측정](https://github.com/LOLog-2026/LOLog-ai/tree/develop/experiments/261111-video-tts))에서 `kid-f-04`, `young-m-03` 두 개를 바꿨으니 바꾼 24개를 드려요

## 7. 저장과 삭제

| 무엇 | 객체 키(제안) | 언제 지우나 |
| --- | --- | --- |
| 완성 영상 | `users/{userId}/videos/{videoId}/video.mp4` | 영상을 지울 때, 탈퇴 |
| 목록용 그림 | `users/{userId}/videos/{videoId}/thumb.jpg` | 위와 같음 |
| 대사 음성(중간 파일) | `users/{userId}/videos/{videoId}/audio.wav` | `DONE`·`FAILED`가 됐을 때 |

- 만화와 같은 버킷(`lolog-comics`)의 `users/{userId}/` 아래라 탈퇴할 때 같이 지워져요
- 카드를 지워도 그 카드로 만든 영상은 남아요(만화와 같아요)

## 8. DB와 lolog-api 구현에서 정한 것 (Claude 초안, 검토 필요)

이 명세에 없던 것을 lolog-api를 만들면서 Claude가 정했어요. 사용자·팀이 아직 하나씩 고르지 않았으니 검토가 필요해요. 바꾸면 lolog-api도 같이 고쳐요. (lolog-api PR #67 이후)

### DB (lolog-api V14)

```sql
CREATE TABLE videos (
    id BIGINT NOT NULL GENERATED BY DEFAULT AS IDENTITY,
    card_id BIGINT,                      -- 카드를 지우면 NULL(ON DELETE SET NULL)
    user_id BIGINT NOT NULL,
    card_summary VARCHAR(30),            -- 요청할 때의 카드 요약(목록 제목)
    status VARCHAR(10) NOT NULL DEFAULT 'QUEUED',   -- QUEUED, VOICE, DRAWING, DONE, FAILED
    storyboard JSONB,                    -- 3호가 보낸 콘티 그대로
    audio_key VARCHAR(255),              -- 중간 음성. DONE·FAILED가 되면 지움
    video_key VARCHAR(255), thumb_key VARCHAR(255), duration_ms BIGINT,
    fail_reason VARCHAR(500),
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT videos_pk PRIMARY KEY (id)
);
CREATE UNIQUE INDEX uk_videos_card_id_in_progress ON videos (card_id) WHERE status IN ('QUEUED', 'VOICE', 'DRAWING');
ALTER TABLE cards ADD COLUMN video_count INTEGER NOT NULL DEFAULT 0;   -- 0~2
ALTER TABLE jobs ADD COLUMN video_id BIGINT;                           -- 영상을 지우면 작업도 지움(CASCADE)
-- jobs.job_type에 VIDEO_VOICE(3호), VIDEO_DRAW(2호)를 더함
```

### 정한 것

| 정하지 않았던 점 | 정한 것 |
| --- | --- |
| "카드 하나에 실패가 아닌 영상 2개"를 세는 법 | 만화처럼 카드의 횟수(`cards.video_count`). 실패하면 돌아오고, 지워도 돌아오지 않음. 앱에는 CardDetail `videoRemaining` = 2 − `video_count`로 줘요(정함 11-13, 지원) |
| 영상을 만드는 동안 카드 고치기·지우기 | 3호가 카드 내용을 쓰는 동안(`QUEUED`·`VOICE`)만 409 CARD\_LOCKED. 그림 단계(`DRAWING`)는 카드를 보지 않아서 막지 않음 |
| 만드는 중인 영상 지우기 | 지울 수 있음. 작업도 지워져 워커의 다음 요청은 404 VIDEO\_NOT\_FOUND(만화와 같음) |
| 콘티보다 음성이 먼저 옴 | 400 INVALID\_REQUEST("콘티를 먼저 보내 주세요") |
| `DRAWING`에서 음성이 또 옴 | 파일만 덮어쓰고 2호 작업은 새로 만들지 않음 |
| 끝난 영상에 워커가 보냄 | 받는 쪽(GET)은 404, 보내는 쪽(POST)은 204로 받고 무시(만화와 같음) |
| 2호에 주는 콘티의 `imageUrl` | 그림을 쓸 수 있는 인물에만 새로 서명해 넣고, 없으면 원래 있던 `imageUrl`을 뺌 |
| VideoSource의 기본 화자 `voice` | 목소리 파일이 있으면 그 기본 화자 id, 없으면 null |
| 완성 영상 검사 | mp4는 `ftyp` 머리말, 썸네일은 JPEG 머리말, `durationMs`는 꼭 있고 0보다 큼. 파일 하나 20MB |
| 영상 목록 | 만화 목록(`GET /api/comics`)과 같게 `sort`(`latest` 기본·`oldest`), `size` 기본 20(1~50), 커서는 마지막 영상 ID. 앱 컨텐츠 미디어 탭이 4컷 탭과 같은 모양으로 쓰게 |
| 목록 제목 | VideoResponse에 `cardSummary`(요청할 때의 카드 요약을 `videos.card_summary`에 복사, 카드를 지워도 남음). ComicResponse와 같음 |

## 9. 내보내기와 공유 (설계_v3 3·4번 탭)

"정함"이라고 쓴 것만 지원이 정한 내용이고, 나머지는 초안이에요. 서버(api) 일은 없어요. 앱 화면은 서현 담당이에요.

### 지금 상태 (11-12 코드 확인)

- 서버는 완성 mp4와 첫 장면 그림을 MinIO에 두고(`videos.video_key`·`thumb_key`), 앱에 10분짜리 서명 주소(`videoUrl`·`thumbnailUrl`)를 줘요. 앱은 그 주소로 바로 재생하고, 만료되면 쇼츠를 다시 받아 새 주소를 써요. "MinIO에 저장돼서 바로 볼 수 있게"는 이 동작으로 끝이에요(정함 11-12, 지원)
- 폰에 남는 건 "폰에 저장"을 눌렀을 때뿐이에요(`Movies/LOLog`)
- 앱 PR #93(11-12 머지): 쇼츠 화면에 "인스타그램 릴스에 공유" 버튼이 있어요. mp4를 `cacheDir/shared/lolog_shorts_{id}.mp4`로 받아 인스타그램 공유 화면만 열어요(`VideoExport.kt`). 카카오톡·공유 창은 아직 없어요

### 화면 (정함 11-12, 지원)

- 쇼츠 보기 화면을 만화 보기 화면에 맞춰요. 만화 화면은 그대로 둬요
- 지우기는 위 오른쪽 점 세 개(⋮) 메뉴에 둬요. 만드는 중·실패 화면도 만화와 같이 그래요
- 아래 버튼은 "내보내기" 하나만 둬요. 카카오톡 전용 버튼은 따로 두지 않아요
- 내보내기 시트는 만화 내보내기 시트와 같은 줄이에요: "폰에 저장", "카카오톡으로 보내기", "인스타그램", "다른 앱으로 공유"(줄 이름은 초안). PR #93의 "인스타그램 릴스에 공유"는 시트의 인스타그램 줄로 옮겨요
- 카카오톡·인스타그램 줄은 만화처럼 그 앱을 고른 공유(안드로이드 인텐트 `setPackage`)이고, 카카오 SDK는 안 써요

### 동작

- 앱이 `videoUrl`에서 mp4를 받은 뒤 `FileProvider` 주소로 안드로이드 공유(`ACTION_SEND`, `video/mp4`)를 열어요. 만화 공유(`ComicExport.kt`의 `share`, 지금은 `image/png`만)와 같은 방식이에요
- 카카오톡: 채팅방 고르기 → 보내기. 받은 사람은 카카오톡 안에서 동영상을 바로 봐요
- 인스타그램: 인스타그램 공유 화면에서 피드·스토리·릴스 등을 사용자가 골라요. 피드·릴스는 편집 화면 → 캡션 → "공유"를 눌러야 게시돼요(칸 이름과 개수는 폰에서 확인 필요)
- **공유한다고 폰(갤러리)에 남기지 않아요(정함 11-12, 지원).** 공유용 임시 파일(`cacheDir/shared`, 만화 공유와 같은 곳)로만 둬요. 남기려면 "폰에 저장"을 써요
- 임시 파일 지우기(초안): 다음에 쇼츠를 공유할 때 예전 공유 영상을 앱이 지워요. 보낸 뒤 바로 지우면 받는 앱이 아직 못 읽었을 수 있어서예요

### 바로 게시(서버가 사용자 계정으로 올리기)

- **정함(11-12, 지원): 우선 앱이 각 앱의 공유 화면을 여는 방식(위)을 구현하고, 서버가 바로 올리는 방식은 검토만 해요**
- 서버가 바로 올리려면 Instagram Graph API(Content Publishing, 비즈니스·크리에이터 계정만)나 YouTube Data API `videos.insert`를 써요. 둘 다 사용자 계정 연결과 그 회사의 앱 검수가 필요하고, 검수 전에는 다른 사람이 못 쓰거나(인스타그램) 비공개로만 올라가요(유튜브·틱톡)
- 검토(초안): 검수가 되면 유튜브부터 더하는 2단계로 봐요

### 참고: 카카오 SDK를 안 쓰는 이유

- 카카오톡 공유 API는 메시지 템플릿(그림·글·버튼 링크)을 보내요. 영상 파일을 메시지에 붙이는 방법이 없고, 링크가 열 공개 영상 페이지도 없어요(서명 주소는 10분 뒤 만료)
- 참고: [카카오톡 공유 Android](https://developers.kakao.com/docs/latest/ko/kakaotalk-share/android-link), [Sharing to Reels (Android)](https://developers.facebook.com/docs/android/sharing-to-reels-instagram), [Instagram Content Publishing](https://developers.facebook.com/docs/instagram-platform/instagram-graph-api/content-publishing), [YouTube videos.insert](https://developers.google.com/youtube/v3/docs/videos/insert)

### 앱 구현에서 정한 것 (Claude 초안, 검토 필요, 앱 PR #99)

| 정하지 않았던 점 | 정한 것 | 왜 |
| --- | --- | --- |
| 카카오톡·인스타그램이 폰에 없을 때 | 영상을 받지 않고 "카카오톡 앱이 없어요"·"인스타그램 앱이 없어요" 창 | PR #93의 인스타그램 창과 같게. 만화는 공유 창으로 넘겨서 다름 |
| 만드는 중·실패 화면의 아래 버튼 | 없앰(지우기는 ⋮ 메뉴만) | 만화 화면과 같게 |
| 예전 공유 파일 지우기 | 쇼츠를 공유할 때 `cacheDir/shared`의 다른 `lolog_shorts_*.mp4`를 지움. 만화 공유 파일은 그대로 | 위 "임시 파일 지우기(초안)" |
| 시트 제목 | "쇼츠 내보내기" | 만화 "만화 내보내기"와 같게 |

### 바꿀 곳

| 레포 | 할 일 |
| --- | --- |
| app | 쇼츠 화면을 만화 화면처럼: ⋮ 메뉴에 "쇼츠 지우기"(만드는 중·실패도), 아래는 "내보내기" 하나 → 만화와 같은 시트. PR #93 릴스 버튼은 시트의 인스타그램 줄로. 예전 공유 영상은 다음 공유 때 지움(초안). 화면은 서현 담당 |
| api | 없음 |
