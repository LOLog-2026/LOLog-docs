# 카드 영상 (초안)

카드로 30초 안팎의 세로 영상을 만드는 기능이에요(FR-EXT-02). 4컷만화는 그대로 두고 더하는 기능이에요. AI 쪽에서 쓴 제안이고, 백엔드·앱 담당과 맞춰서 확정해요.

- 대사는 화자 목소리의 TTS로 읽고, 화면에는 자막으로 넣어요. 말풍선은 없어요
- 카드 구간의 실제 음성([음성 저장](audio.md))은 영상에 쓰지 않아요
- 그림은 장면마다 FLUX로 그리고, 확대·이동과 장면 전환 효과는 워커가 코드로 넣어요
- 영상은 카드 주인만 봐요. 방에 올리기와 외부 공유는 이번 범위가 아니에요(공유는 폰에 저장한 파일로 해요)

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

걸리는 시간은 30초 영상 기준으로 3호 약 4분(콘티 1분 + TTS 3분), 2호 약 5~6분(그림 6장 + 합치기)을 예상해요. Jetson 3호 TTS는 실제 길이의 5.4배가 걸렸어요([측정](https://github.com/LOLog-2026/LOLog-ai/tree/develop/experiments/261111-video-tts)). 2호 시간은 아직 재지 않았어요.

## 2. 앱 API (/api)

만화 API와 같은 모양이에요. 로그인이 필요하고 내 것만 보여요.

| 메서드 | 주소 | 하는 일 | 응답 |
| --- | --- | --- | --- |
| POST | /api/cards/{id}/videos | 영상 요청. 그 카드에 만드는 중인 영상이 있으면 409 `VIDEO_IN_PROGRESS`, 한도(카드 하나에 실패가 아닌 영상 2개)를 넘으면 409 `VIDEO_LIMIT` | 202 `{ videoId, status }` |
| GET | /api/videos | 내 영상 목록 | `videos[]`, `nextCursor` |
| GET | /api/videos/{id} | 영상 상태·결과 | VideoResponse |
| DELETE | /api/videos/{id} | 영상 지우기 | 204 |

**VideoResponse**

| 필드 | 타입 | 필수 | 설명 |
| --- | --- | --- | --- |
| id | Long | O | |
| cardId | Long | null 가능 | 카드를 지웠으면 null |
| status | String | O | `QUEUED`·`VOICE`·`DRAWING`·`DONE`·`FAILED` |
| videoUrl | String | null 가능 | `DONE`일 때만. 10분 동안 열리는 서명 주소(mp4) |
| thumbnailUrl | String | null 가능 | `DONE`일 때만. 첫 장면 그림(목록용) |
| durationMs | Long | null 가능 | `DONE`일 때만 |
| createdAt | String | O | |

- 카드 자세히(`GET /api/cards/{id}`)에 `videos[]`(id, status)를 더해요. `comics[]`와 같은 방식이에요
- 푸시: `VIDEO_DONE`·`VIDEO_FAILED`, 데이터는 `videoId`, `cardId`. 문구에 대사를 넣지 않아요(FR-NOTI-03)

## 3. 워커 API (/internal)

`X-Internal-Token`을 붙여요. GET이 아닌 것은 모두 204예요.

| 메서드 | 주소 | 누가 | 하는 일 | 본문 |
| --- | --- | --- | --- | --- |
| GET | /internal/videos/{id}/source | 3호 | 카드 내용 받기. 부르면 `VOICE` | → 200 VideoSource |
| POST | /internal/videos/{id}/storyboard | 3호 | 영상 콘티 보내기 | 영상 콘티 JSON(5장) |
| POST | /internal/videos/{id}/audio | 3호 | 대사 음성 올리기. 받으면 `DRAWING`, 2호 큐에 넣음 | multipart `file`(audio/wav) |
| GET | /internal/videos/{id}/storyboard | 2호 | 3호가 보낸 콘티 받기 | → 200 영상 콘티 JSON |
| GET | /internal/videos/{id}/audio | 2호 | 3호가 올린 음성 받기 | → 200 wav 바이트 |
| POST | /internal/videos/{id}/video | 2호 | 완성 영상 올리기. 받으면 `DONE`, 푸시 | multipart `file`(video/mp4), `thumbnail`(image/jpeg), `durationMs` |
| POST | /internal/jobs/{id}/fail | 둘 다 | 실패 알리기(이미 있는 API) | JobFailRequest |

- **VideoSource**는 ComicSource와 같은 모양이에요(`comicId` 자리에 `videoId`). 화자의 `character`(`ref`, `appearance`, `imageUrl`)도 그대로 받아요
- 영상이 없거나 이미 끝났으면 404 `VIDEO_NOT_FOUND`예요. 워커는 다시 시도하지 않고 그 작업을 멈춰요(만화와 같아요)
- **파일 크기**: 음성은 wav 한 개(24kHz 모노, 60초면 약 2.9MB), 영상은 mp4 한 개(720×1280, 60초면 10MB 안팎)예요. `/internal`은 SSH 터널로 메인 백엔드에 바로 닿으니 Nginx 5MB 한도와는 상관없고, 메인 백엔드의 multipart 한도를 **영상 20MB**까지 받게 해 주세요
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
  "characters": [ { "key": "화자1", "name": "민수", "appearance": "a young man ...", "ref": "default:young-m-01", "voice": "young-m-01" } ],
  "scenes": [
    { "index": 1, "startMs": 0, "endMs": 5200, "scene": "...", "cast": [ { "key": "화자1", "position": "left" } ],
      "motion": "zoom_in", "transition": "fade",
      "lines": [ { "speaker": "화자1", "text": "양념 반 후라이드 반", "startMs": 300, "endMs": 2400, "sourceLineIds": [101] } ] }
  ]
}
```

- 장면은 4~8개, 전환 효과와 움직임은 정해 둔 목록에서 LLM이 골라요
- 대사의 `startMs`·`endMs`는 음성 파일 안의 위치예요. 자막도 이 시각에 맞춰 나와요

## 6. 목소리

- 기본 화자 24명에게 목소리가 하나씩 있어요([AI 레포 #27](https://github.com/LOLog-2026/LOLog-ai/issues/27)). 화자에 기본 화자가 붙어 있으면(`ref`가 `default:young-f-02`) 그 목소리로 읽어요
- 화자에 그림이 안 붙어 있으면 콘티에서 정한 외모(나이대·성별)에 맞는 기본 목소리를 워커가 골라요
- **프로필 캐릭터(`profile:42`)**: 사진으로 새 목소리를 만들지 않고 기본 목소리 24개 중 하나를 써요. 그래서 "그림으로 목소리 만들기" 요청은 필요 없어요. 프로필 캐릭터의 `appearance`에는 나이대·성별이 없어서(사진만 받기로 함), 아래 가로 정했어요

| 방법 | 백엔드·앱이 할 일 | 장단점 |
| --- | --- | --- |
| **가. 사용자가 고름 (정함)** | 프로필 캐릭터에 `voice`(기본 화자 id, null 가능) 필드. 앱이 24개를 미리 듣고 고르는 화면. SourceCharacter에 `voice`를 실어 보냄 | 본인 목소리 느낌을 직접 고르니 틀릴 일이 없어요. 화면이 하나 늘어요 |

- 미리 듣기용 인사 클립(wav 24개, 합계 3.1MB)은 기본 화자 그림처럼 저장소에 올려 두고 주소를 내려 주면 돼요. 파일은 AI 쪽에서 드려요

## 7. 저장과 삭제

| 무엇 | 객체 키(제안) | 언제 지우나 |
| --- | --- | --- |
| 완성 영상 | `users/{userId}/videos/{videoId}/video.mp4` | 영상을 지울 때, 탈퇴 |
| 목록용 그림 | `users/{userId}/videos/{videoId}/thumb.jpg` | 위와 같음 |
| 대사 음성(중간 파일) | `users/{userId}/videos/{videoId}/audio.wav` | `DONE`·`FAILED`가 됐을 때 |

- 만화와 같은 버킷(`lolog-comics`)의 `users/{userId}/` 아래라 탈퇴할 때 같이 지워져요
- 카드를 지워도 그 카드로 만든 영상은 남아요(만화와 같아요)
