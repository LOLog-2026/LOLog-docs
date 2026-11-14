# 글로 만화·영상 만들기 (초안)

사용자가 직접 쓴 글(일기·일지·일화)과 블로그 글로도 카드를 만들어, 지금처럼 만화·쇼츠를 만드는 기능이에요(FR-TEXT-01~03). 녹음에서 카드가 생기는 길 옆에 "글 → 카드" 길을 하나 더 내요. 카드가 생긴 뒤(카드 고치기·화자 캐릭터·만화·쇼츠)는 지금과 같아요.

> 이 문서는 업데이트 설계_v3의 1-1(블로그 글)·1-2(직접 쓴 글) 탭을 옮긴 초안이에요. 구현은 lolog-api PR #73(직접 쓴 글)·#74(블로그), 앱 PR #98에 있어요(11-13, 머지 전). 구현하면서 정한 것은 [10장](#10-구현에서-정한-것-claude-초안-검토-필요)에 있어요.
> - **정함**: 지원이 정한 것
> - **임시 정함**: 지원이 정했지만 나중에 바뀔 수 있는 것
> - **제안**: 지원이 고른 안이고, AI 담당(민주)이 확정해야 하는 것
> - 이름 옆 "(초안)"인 주소·필드·열은 구현하면서 바뀌어도 돼요

## 1. 지금 상태 (코드 확인, 11-12)

- 카드는 녹음 텍스트를 Jetson 1호가 감지해서만 생겨요(`POST /internal/recordings/{id}/cards`). 녹음 없이 카드를 만드는 길이 없어요
- 만화(`POST /api/cards/{id}/comics`)와 쇼츠(`POST /api/cards/{id}/videos`)는 카드에서만 만들어요
- 글을 쓰거나 붙여넣는 화면·API, RSS를 읽는 코드는 어느 레포에도 없었어요. 11-13에는 lolog-api develop에 블로그 RSS 코드(`FeedLocator`·`FeedParser`·`BlogFetcher`·`BlogPoller`)가 있어요. 플랫폼별로 나누는 건 아래 [플랫폼별 구현](#플랫폼별-구현-정함-11-1314-지원)이에요

## 2. 공통: 글 → 카드

- 블로그 글과 직접 쓴 글은 **한 표(`texts`)**를 써요(정함 11-12, 지원). 워커 길·메시지 모양도 하나예요(제안, AI 담당 확정 필요)
- **글 하나 = 이야기 하나 = 카드 하나**예요(임시 제안 11-12, 지원, AI 담당 확정 필요, 나중에 바뀔 수 있음). 블로그 글이 길어도 카드는 하나예요
- 글은 **이야기 감지 없이 바로 화자 구분 → 카드**로 가요(직접 쓴 글은 임시 제안, 블로그 글은 제안. 둘 다 AI 담당 확정 필요). 이야기 감지는 긴 녹음에서 이야기 구간을 찾는 일인데, 글은 쓴 사람이 이미 한 주제로 묶어 둔 것이라서예요. 블로그에서 이야기 아닌 글 걸러내기는 사용자가 카테고리를 고르는 것으로 대신해요
- 글에서 화자를 어떻게 나눌지(서술이 섞인 글을 "화자: 대사"로)와 글용 프롬프트는 AI 담당이 정해요
- 글 카드에는 음성이 없어서 순간 다시 듣기 버튼은 안 보여요(`hasAudio` false. 앱에서 확인 필요)
- 블로그에서 만든 카드, 직접 쓴 글 카드도 카드마다 만화 3번 제한을 그대로 둬요(정함 11-12, 지원)

### 글 길이 상한 (정함 11-12, 지원)

- 상한 숫자는 AI 담당이 정하고, api가 큐에 넣기 전에 지켜요. 숫자가 정해지기 전까지 백엔드는 일단 2,000자로 막아요(임시 정함 11-12, 지원)
- 직접 쓴 글은 상한을 넘으면 올릴 때 거절해요(`TEXT_TOO_LONG`). 사용자가 줄여서 다시 올려요
- 블로그 글은 사용자가 고칠 수 없으니 앞부분만 잘라 보내요
- 잘랐다는 건 푸시 없이, `texts.truncated`를 두고 카드 화면에 "글 앞부분으로 만들었어요" 정도로 보여 줘요(임시 정함 11-12, 지원)

### 글 본문 남기기 (정함 11-12, 지원)

- 직접 쓴 글은 카드가 생겨도 기록으로 남기고, 카드 자세히 화면의 "원문 보기"로 다시 봐요(화면은 서현 담당). 녹음 글을 지우는 건 녹음에 남의 말이 섞여서이고(FR-CARD-04), 직접 쓴 글은 본인 기록이라 남겨요
- 블로그 글은 카드가 생기면 본문(`texts.body`)을 지우고 링크(`blog_posts.link`)만 남겨요. 본문은 블로그에 이미 있고 남의 블로그일 수도 있어서예요
- 카드를 지우면 그 글(`texts`)도 같이 지워요. `blog_posts` 줄은 같은 글을 다시 가져오지 않게 남겨요
- 탈퇴하면 그 회원의 `texts`·`blog_sources`·`blog_posts`를 모두 지워요

### 그 밖에 정한 것 (정함 11-12, 지원)

- **글 카드의 날짜:** `cards.recorded_at`(비울 수 없음)은 직접 쓴 글이면 올린 시각(`texts.created_at`), 블로그 글이면 글을 쓴 시각(RSS `pubDate`, `blog_posts.published_at`)으로 채워요. pubDate가 없는 글은 가져온 시각(`blog_posts.created_at`)을 써요
- **푸시:** 직접 쓴 글은 카드가 생기면 푸시해요(`CARDS_CREATED`에 `textId`를 담아서). 블로그 글은 30분마다 여러 장 생길 수 있어 푸시 없이 보관함 새 소식 점(FR-NOTI-06)만 켜요
- **상태 보기:** `POST /api/texts`는 202 `{ textId, status }`를 돌려주고, `GET /api/texts/{id}`로 `status`(`QUEUED`·`DONE`·`FAILED`)와 `cardId`를 봐요. 실패하면 앱은 "만들지 못했어요"만 보여 줘요. 블로그 글은 사용자가 기다리지 않아서 가져온 글 목록(`GET /api/blogs/{id}/posts`)에서만 봐요
- **상태 채우기:** `texts`에 상태 칸을 두지 않고 물어볼 때마다 채워요. 그 글로 만든 카드가 있으면 `DONE`, 그 글의 `TEXT` 작업이 실패면 `FAILED`, 그 밖에는 `QUEUED`예요
- **카드를 지운 블로그 글:** 카드를 지우면 그 글(`texts`)도 지워져서, 가져온 글 목록(`GET /api/blogs/{id}/posts`)의 그 글은 `cardId`·`status`가 둘 다 null이에요. 앱은 "에피소드 지움"으로 보여 줘요(정함 11-12, 지원)
- **작업 종류:** `jobs.job_type`에 `TEXT`를 더해요(이름은 초안)

## 3. 직접 쓴 글

### 보여 주는 방식 (정함 11-12, 지원)

- 인터넷 글 붙여넣기는 드러내지 않고, 사용자가 직접 쓴 글·일지·일기·일화를 콘텐츠로 만들어 주는 기능으로 보여 줘요. 11-11 저작권 검토에서 남의 글로 만화·쇼츠를 만드는 건 2차적저작물이라 허락이 필요하다고 봤어요
- 안내 문구: "직접 겪은 일을 적어 보세요. 일기, 일지, 기억나는 일화를 만화와 쇼츠로 만들어 드려요."

### 입력 칸 (정함 11-12, 지원)

일기를 쓸 때 제목·장소·상황을 같이 적게 해요. 제목은 사용자가 그 기록을 알아보는 데도 도움이 돼서 필수예요. 장소·상황은 선택이고, 하나로 합치지 않고 따로 둬요.

| 칸 | 필드 | 필수 | 의미와 역할 |
| --- | --- | --- | --- |
| 제목 | `title` | 필수 | 글의 이름. 사용자가 쓴 글을 알아보는 데 써요. 카드에는 제목 칸이 없어서(요약이 제목 역할) LLM이 요약을 만들 때 참고해요 |
| 본문 | `body` | 필수 | 카드로 바꿀 글 자체. LLM이 여기서 화자를 나눠 "화자: 대사"로 바꿔요 |
| 장소 | `place` | 선택 | 이야기가 일어난 곳. 적으면 적은 그대로 카드 장소(`cards.place`)가 되고 만화 콘티도 그 장소를 써요. 비면 녹음 카드처럼 AI가 정해요. 카드 고치기에서 바꿀 수 있어요 |
| 상황 | `situation` | 선택 | 언제·누구와·왜 같은 배경. 카드에는 저장하지 않고 LLM에 배경으로만 넘겨요. 비면 메시지에서 빠지고 LLM은 본문만 봐요 |

- 앱은 제목·본문이 비면 보내기를 막고, 서버도 빈 값·공백만 있는 값을 거절해요(기존 규칙과 같음)
- 본문 칸 이름은 DB·메시지 모두 `body`예요(정함 11-12, 지원). `kind: "text"`·`textId`와, 녹음 문장·카드 대사의 한 줄 `text`와 헷갈리지 않게 하려는 것이에요

## 4. 블로그 글

### 연결 방식 (정함 11-12, 지원)

- 기존 플랫폼 RSS가 우선이에요. 따로 맞춰 구현하는 플랫폼은 네이버 블로그·티스토리·velog·Medium·WordPress 5개예요(정함 11-14, 지원, 아래 플랫폼별 구현). 우리가 전에 만든 블로그와 연결할지는 추후 선택사항이에요
- 사용자는 **블로그 링크**와 **가져올 카테고리 목록** 두 가지만 줘요
- 서버는 링크로 플랫폼을 고르고, 그 플랫폼 방식으로 RSS 주소를 찾아 주기적으로 읽어요. 주요 플랫폼 5개는 플랫폼마다 따로 구현하고, 나머지는 Other(자동 찾기)로 읽어요(정함 11-13·14, 지원)
- 읽은 글 중 카테고리가 고른 카테고리에 맞는 글만 저장해요. 카테고리를 하나도 안 고르면 전부 받아요. 카테고리를 읽는 법은 플랫폼마다 따로 둬서, 태그가 카테고리에 섞이지 않게 해요(정함 11-13, 지원). 태그는 따로 보지 않아요
- 고른 카테고리 글이 없으면(플랫폼이 카테고리를 안 싣는 경우 포함) 아무 글도 안 들어오고, 따로 처리하지 않아요
- 블로그를 사용자당 여러 개 등록할 수 있어요. RSS 하나는 블로그 하나라, 네이버·티스토리를 같이 쓰는 사람은 둘 다 등록해요
- 같은 RSS 주소(`feed_url`)를 두 번 등록하면 거절해요
- 플랫폼은 연결할 때 한 번 골라 `blog_sources.platform`에 저장해요. 서버 안에서만 쓰고 앱 응답에는 넣지 않아요(정함 11-14, 지원)
- 블로그 이름은 RSS channel의 `title`을 저장해요
- 블로그 주소로 RSS를 찾지 못했거나 내부 주소면 `INVALID_BLOG`(400), 없는 블로그면 `BLOG_NOT_FOUND`예요. 주소(`url`)가 빠졌거나 비었으면 다른 빠진 칸처럼 400 `INVALID_REQUEST`(field `url`)예요(Claude 판단, 확인 필요)

### 플랫폼별 구현 (정함 11-13·14, 지원)

RSS는 블로그 플랫폼마다 주소 규칙, 본문이 있는 칸, 카테고리를 싣는 법이 달라요. 이걸 if문으로 잇지 않고 플랫폼마다 구현을 따로 두는 인터페이스로 나눠요.

- **주요 구현 5개:** 네이버 블로그, 티스토리, velog, Medium, WordPress. 나머지(브런치·Blogger 등)는 Other(지금의 RSS 구현: 주소 그대로 + `<link rel="alternate">` 찾기)로 읽어요(정함 11-14, 지원)
- **카테고리:** 플랫폼마다 카테고리를 제대로 읽어(태그 말고) 선택지로 보여 줘요(정함 11-13, 지원). 플랫폼별로는 아래 표예요
- **본문:** Other도 `content:encoded`가 있으면 그것, 없으면 `description`이에요(정함 11-14, 지원)
- **본문이 일부만 오는 플랫폼:** 글 페이지 크롤링은 하지 않아요(정함 11-14, 지원)

**플랫폼별 카테고리 (11-14 공개 피드로 확인)**

| 플랫폼 | RSS의 `<category>` | 카테고리 읽는 법 |
| --- | --- | --- |
| 네이버 블로그 | 카테고리. 태그는 `<tag>` 칸에 따로 와요 | `<category>`만 봐요 |
| 티스토리 | 카테고리. 하위 카테고리는 `상위/하위` 모양이에요. 태그는 오지 않아요 | 선택지에 `상위`와 `상위/하위`를 모두 보여 주고, `상위`를 고르면 그 아래 하위 카테고리 글도 가져와요(정함 11-14, 지원). 하위인지는 `상위/`로 시작하는지로 봐요. 앞글자만 보면 `여행`이 `여행기`까지 잡혀요(AI 담당 지적) |
| WordPress | 카테고리와 태그가 섞여 와요(예: `Development`, `Releases`, 태그 `7.2`, `development`) | WordPress REST API로 진짜 카테고리 이름 목록을 받아 그 이름만 카테고리로 봐요(정함 11-14, 지원). 이 목록은 최근 글에 없는 카테고리까지 다 있어서 선택지로 그대로 보여 줘요 |
| Medium | 모두 태그예요. Medium에는 카테고리가 없어요 | 카테고리를 고를 수 없고 글을 전부 받아요(정함 11-14, 지원) |
| velog | 칸이 없어요. velog에는 카테고리가 없어요(태그·시리즈만) | 카테고리를 고를 수 없고 글을 전부 받아요(정함 11-14, 지원) |
| Other | 그대로 | `<category>`를 그대로 봐요 |

- WordPress REST(초안): 자체 설치는 RSS channel의 `link` 뒤에 `/wp-json/wp/v2/categories?per_page=100&orderby=count&order=desc&_fields=name`, wordpress.com 블로그는 `https://public-api.wordpress.com/wp/v2/sites/{호스트}/categories`(같은 조건)예요. 많이 쓴 카테고리부터 100개까지예요. 받지 못하면(REST를 끈 사이트 등) RSS의 `<category>`를 그대로 써요. 이때는 태그가 섞일 수 있어요. 이 주소도 블로그 주소처럼 내부 주소는 거절해요
- Medium·velog에 `categories`를 보내면 400 `INVALID_REQUEST`(field `categories`, "이 블로그는 카테고리를 고를 수 없어요")예요(초안). `POST /api/blogs/check`의 `categories`는 빈 배열이에요
- 카테고리 전체 목록은 WordPress만 받아요. 네이버·티스토리는 공개된 목록 API가 없어 페이지를 긁어야 해서 넣지 않아요(티스토리 Open API는 종료됨). 그래서 이 둘은 최근 글(네이버 50개, 티스토리 보통 10~50개)의 카테고리만 선택지에 나오고, 나머지는 직접 적어요

**플랫폼 고르기 (초안)**

| 플랫폼 | 주소로 알기 | RSS 주소 후보 | 피드 `generator`로 알기(자체 도메인) |
| --- | --- | --- | --- |
| 네이버 블로그 | `blog.naver.com`, `m.blog.naver.com` | `https://rss.blog.naver.com/{id}.xml` | `Naver Blog` |
| 티스토리 | `*.tistory.com` | `https://{호스트}/rss` | `TISTORY` |
| velog | `velog.io/@{id}` | `https://v2.velog.io/rss/@{id}` | - |
| Medium | `medium.com/@{id}`, `medium.com/{출판물}`, `*.medium.com` | `https://medium.com/feed/@{id}`, `https://medium.com/feed/{출판물}`, `https://{호스트}/feed` | `Medium` |
| WordPress | `*.wordpress.com` | `https://{호스트}/feed/` | `wordpress.org`·`wordpress.com`이 들어 있음 |
| Other | 나머지 | (없음) | - |

- 주소로 고른 플랫폼의 후보부터 읽고, 안 되면 넣은 주소 그대로 + `<link rel="alternate">` 찾기(지금 구현)로 읽어요
- 주소로 Other인데 찾은 피드의 `generator`가 위 값이면 그 플랫폼으로 봐요. 자기 도메인을 붙인 티스토리·WordPress·Medium 블로그를 위해서예요

인터페이스(초안):

```java
public interface BlogPlatform {
    Platform platform();                                // NAVER, TISTORY, VELOG, MEDIUM, WORDPRESS, OTHER
    boolean supports(URI siteUrl);                      // 이 플랫폼 주소인가
    boolean supportsFeed(Feed feed);                    // 주소로 모를 때 피드 generator로 알아보기
    List<String> feedCandidates(URI siteUrl);           // RSS 주소 후보(순서대로 시도)
    String body(Item item);                             // 본문 칸. 기본은 content:encoded, 없으면 description
    Categories categories(URI siteUrl, Feed feed);      // 카테고리 읽는 법(태그 빼기, 하위 카테고리, 고를 수 없음)
}
```

- 구현(초안): `NaverPlatform`, `TistoryPlatform`, `VelogPlatform`, `MediumPlatform`, `WordPressPlatform`, 그리고 맨 마지막 `OtherPlatform`
- 고르는 법(초안): Spring이 모든 구현을 `List<BlogPlatform>`으로 넣어 주고, 앞에서부터 `supports`가 처음 true인 것을 써요. `OtherPlatform`은 `@Order(LOWEST_PRECEDENCE)`에 `supports`가 늘 true예요
- `categories`가 사이트 단위인 이유: WordPress는 글 하나만 봐서는 카테고리와 태그를 못 가려서, 피드를 읽을 때 한 번 REST로 목록을 받아 와야 해요

### 앱 안내 (정함 11-13·14, 지원)

- 블로그 연결 화면에서 따로 구현한 플랫폼을 안내하고, 다른 플랫폼은 연결이 안 되거나 최적화되지 않을 수 있다고 알려요(정함 11-13, 지원)
- 안내 문구(정함 11-14, 지원): "네이버 블로그, 티스토리, velog, Medium, WordPress 블로그를 잘 가져와요. 다른 블로그는 연결되지 않거나 글이 일부만 들어올 수 있어요." 화면은 서현 담당
- 안내할 플랫폼 목록은 앱에 박아 둬요. 목록을 주는 API는 두지 않아요(정함 11-14, 지원)
- Other로 잡힌 블로그면 응답에 `optimized: false`를 넣어, 앱이 "이 블로그는 최적화되지 않았어요"를 한 번 더 보여 줄 수 있게 해요(정함 11-14, 지원). 따로 구현한 5개 플랫폼이면 `true`예요. 플랫폼 이름은 여전히 응답에 넣지 않아요
  - 넣는 곳(초안): `POST /api/blogs/check` 응답과 블로그 응답(`POST`·`PATCH`·`GET /api/blogs`)
- 최근 글에 없는 카테고리: WordPress만 전체 목록을 받아요(위 플랫폼별 카테고리). 나머지는 직접 적어요

### 카테고리 고르기 (정함 11-12, 지원)

- **피드에서 모아 보여 주기:** 링크를 넣으면 서버가 그 블로그 피드를 한 번 읽어, 그 플랫폼 방식으로 읽은 카테고리를 목록으로 보여 줘요(태그는 빼요, 정함 11-13, 지원). 선택지를 미리 만들지 않아서 블로그마다·플랫폼마다 달라도 돼요
- **직접 치기:** 최근 글에 없어 목록에 안 보이는 카테고리는 사용자가 이름을 적고, 서버는 그 플랫폼 방식으로 읽은 카테고리 글자와 같은지만 봐요
- 카테고리 이름은 1~30자예요. 이미 있는 이름(대소문자 무시)은 더하지 못해요
- **화면 흐름:** 블로그 링크를 넣고 확인을 누르면, 그 밑에 카테고리 선택(피드에서 나온 목록 + 직접 치기)을 바로 띄워요. 그래서 링크만 받아 RSS를 찾고 카테고리 목록을 돌려주는 API가 있어요(`POST /api/blogs/check`)

### 가져오는 때 (정함 11-12, 지원)

- 서버가 등록한 블로그를 주기적으로 읽고(간격은 초안 30분), 고른 카테고리의 새 글은 바로 카드로 만들어요
- 카테고리로 이미 걸러지니 사용자가 글을 하나씩 고르지 않고, 피드에서 밀려난 글도 놓치지 않아요
- 블로그를 연결하면 고른 카테고리마다 그 카테고리의 최근 글 3개만 바로 불러와 카드로 만들어요(카테고리를 안 고르면 블로그 전체에서 최근 3개, 정함 11-14, 지원). 그보다 옛 글은 불러오지 않고 가져온 글 목록에도 안 나와요. 다음에 읽을 때 새 글로 잡히지 않게 서버는 그 글의 `guid`만 `blog_posts`에 적어 둬요(초안). 카테고리를 더하면(PATCH) 새로 고른 카테고리의 피드 글은 다음에 읽을 때 카드로 만들어요
- 같은 글은 `blog_posts`의 `(user_id, guid)`로 한 번만 가져와요. `guid`에는 RSS `guid` / Atom `id` / 둘 다 없으면 `link`를 넣어요

### 연결 끊기 (정함 11-12, 지원)

- `DELETE /api/blogs/{id}`로 끊으면 `blog_sources` 줄만 지워요. 이미 만든 카드는 남아요
- `blog_posts` 줄은 남긴 채 `source_id`만 비워요. 그래서 다시 연결해도 같은 글을 다시 가져오지 않고, 남은 카드의 원문 링크·블로그 이름(`blog_posts.blog_title`)도 유지돼요
- 다시 연결한 블로그의 가져온 글 목록에는 예전 글이 안 보여요. 그대로 둬요(카드는 보관함에 남아 있음)

### 피드 모양과 본문 (Claude 판단, 확인 필요)

블로그 피드는 플랫폼마다 RSS 2.0이나 Atom 중 하나예요. 둘 다 XML이고, 서버는 둘을 같은 칸으로 읽어요.

| 쓰는 곳 | RSS 2.0 | Atom |
| --- | --- | --- |
| 글 하나 | `channel` 안의 `item` | `feed` 안의 `entry` |
| 블로그 이름(`blog_sources.title`) | `channel`의 `title` | `feed`의 `title` |
| 글 제목 | `title` | `title` |
| 원문 링크 | `link` | `link`의 `href` |
| 본문 | `content:encoded`가 있으면 그것, 없으면 `description`(본문 일부나 전체, HTML) | `content` 또는 `summary` |
| 쓴 시각(`published_at`) | `pubDate`(없을 수 있음) | `published`, 없으면 `updated` |
| 같은 글 구분(`guid`) | `guid` | `id` |
| 카테고리 | `category`(여러 개일 수 있음) | `category`의 `term` |

- 칸이 플랫폼마다 다 차 있지는 않아요. 날짜·guid가 없을 때 쓰는 값은 위 "글 카드의 날짜"와 "가져오는 때"를 따라요
- 본문 칸에 글 전체가 들어오는지는 플랫폼·설정마다 달라서, 플랫폼마다 확인이 필요해요
- AI 쪽에는 XML 그대로가 아니라, 서버가 HTML 태그를 벗긴 글로 넘겨요. 블로그 글은 RSS 제목을 `title`로 쓰고, `place`·`situation`은 비어요
- 본문은 피드에 있는 칸만 써요. `content:encoded`가 있으면 그것, 없으면 `description`이에요. Other도 같아요(정함 11-14, 지원). 링크를 열어 본문을 긁지는 않아요(크롤링 안 함, 정함 11-14, 지원. 아래 저작권)

### 주소와 보안 (정함 11-12, 지원)

- 사용자가 아무 주소나 넣게 하면 서버가 내부 주소(Redis, DB 등)를 대신 부르는 문제(SSRF)가 생겨요. 그래서 내부 IP는 거절하고, 리다이렉트도 다시 확인해요
- api는 Docker 컨테이너 안에서 돌아서, 그 안의 `localhost`는 컨테이너 자신이에요. 같은 서버의 블로그를 부를 일이 생기면 `host.docker.internal:포트`로 불러요(우리 블로그 연결은 추후 선택사항)

### RSS와 저작권

법률 자문이 아니라 11-11 저작권 검토 기준의 판단이에요.

- RSS는 구독하라고 연 통로예요. 피드 리더처럼 글을 받아 보여 주는 건 블로그가 의도한 쓰임이라, 11-11에 위험하다고 본 크롤링(HTML 긁기)과는 달라요
- 하지만 RSS가 저작권 허락은 아니에요. 글을 읽어 가도 된다는 거지, 그 글로 만화·쇼츠(2차적저작물)를 만들어도 된다는 허락은 아니에요
- 그래서 "누구 블로그냐"가 핵심이에요. 사용자 본인의 블로그는 본인이 저작권자라 문제없어요. 걸리는 건 사용자가 남의 블로그 링크를 넣는 경우뿐이에요

## 5. AI 쪽에 넘기는 모양

### 큐 메시지 (제안 11-12, 지원, AI 담당 확정 필요)

- 따로 큐를 두지 않고 Jetson 1호 LLM 스트림 `lolog:detect`에 같이 넣어요. `kind: "text"`를 더해요. 지금 detect 메시지에는 `kind`가 없어요(`{ jobId, recordingId, partNo, isLast, sentences }`)
- 워커는 `kind`가 `text`면 이야기 감지를 건너뛰고 화자 구분 → 카드 하나만 만들어요
- 글은 녹음 감지처럼 메시지에 바로 넣어요. 블로그 글과 직접 쓴 글이 같은 모양이라 워커는 하나만 알면 돼요
- **본문 나누기:** api는 `body`를 통째로 보내고, 워커(LLM)가 문맥을 보고 카드 대사(`lines[]`)로 나눠요. 녹음은 api가 이미 구간(`sentences`)으로 나눠 보내지만, 글을 기계적으로 미리 자르면 대사 만들기에 방해가 돼요. AI 워커가 지금 문장을 받아 프롬프트를 만드는 방식은 ai 레포에서 확인하지 않았어요

```json
{
  "jobId": 101,
  "kind": "text",
  "textId": 8,
  "title": "동생이랑 마트 간 날",
  "body": "오늘 동생이랑 마트에 갔는데...",
  "place": "마트",
  "situation": "주말 저녁, 장 보러 감"
}
```

- `body`: 블로그 글이면 HTML 태그를 벗긴 본문(길이 상한을 넘으면 앞부분), 직접 쓴 글이면 사용자가 쓴 글 그대로
- `place`·`situation`: 비면 메시지에서 빠져요

### 결과 (제안 11-12, 지원, AI 담당 확정 필요)

- 주소는 `POST /internal/texts/{id}/cards`예요. 녹음 결과 주소(`/internal/recordings/{id}/cards`)는 녹음 상태·넘겨 둔 글(`carryFromMs`)·음성까지 다뤄서 따로 둬요
- 글 결과는 `{ jobId, cards: [카드 1장] }`이에요. 카드 한 장의 모양(`kind`, `summary`, `startMs`, `endMs`, `speakers`, `lines[]`)은 녹음 카드(DetectResult)와 같고 필드를 빼지 않아요
- 글 카드는 카드·대사의 `startMs`·`endMs`를 null로 보내요. 카드 밖 칸인 `partNo`·`carryFromMs`는 녹음 1시간 묶음에만 뜻이 있어서 글 결과에는 두지 않아요
- 실패는 지금처럼 `POST /internal/jobs/{id}/fail`이에요
- 워커가 결과를 보내기 전에 카드·글이 지워졌으면(사용자가 지움, 탈퇴) `404 TEXT_NOT_FOUND`예요. 워커는 녹음의 `RECORDING_NOT_FOUND`처럼 그 메시지를 버려요(Claude 판단, 확인 필요)

```json
// POST /internal/texts/{id}/cards
{
  "jobId": 101,
  "cards": [
    {
      "kind": "웃김", "summary": "동생이 마트에서 생긴 일",
      "startMs": null, "endMs": null,
      "speakers": ["화자1", "화자2"],
      "lines": [
        { "lineNo": 1, "speaker": "화자1", "text": "이거 사도 돼?", "startMs": null, "endMs": null }
      ]
    }
  ]
}
```

## 6. API (초안)

### 앱 (/api)

| 메서드 | 주소 | 하는 일 | 보내는 것 | 받는 것 |
| --- | --- | --- | --- | --- |
| POST | /api/texts | 직접 쓴 글 올리기 → `texts` 저장 → 큐. 길이 상한을 넘으면 400 `TEXT_TOO_LONG` | title, body, place, situation | 202 textId, status |
| GET | /api/texts/{id} | 글 상태와 원문. 블로그 글이면 body 대신 link와 blogTitle. 없는 글이면 `TEXT_NOT_FOUND` | - | textId, source, status, cardId, title, body, place, situation, link, blogTitle, truncated, createdAt |
| POST | /api/blogs/check | 링크 확인 버튼용. 링크만 받아 RSS 주소를 찾고 피드에 나온 카테고리 목록을 돌려줘요(저장 안 함) | url | feedUrl, categories\[\], optimized |
| POST | /api/blogs | 블로그 등록. 서버가 RSS 주소를 찾아요. 같은 RSS 주소를 이미 등록했으면 거절 | url, categories\[\] | 201 블로그 |
| GET | /api/blogs | 연결한 블로그 목록(연결한 순서) | - | blogs\[\] |
| PATCH | /api/blogs/{id} | 가져올 카테고리 목록 바꾸기 | categories\[\] | 200 블로그 |
| DELETE | /api/blogs/{id} | 블로그 연결 끊기. 이미 만든 카드는 남아요 | - | 204 |
| GET | /api/blogs/{id}/posts | 가져온 글 목록(고른 카테고리만, 각각 바로 카드로 만들어짐). 최신 글부터 20개씩 | cursor, size(기본 20, 1~50. 다른 목록과 같음) | posts\[\] (title, link, category, publishedAt, cardId, status), nextCursor |

- **블로그 응답 모양(정함 11-12, 지원):** `{ id, siteUrl, title, categories, optimized }`(`optimized`는 11-14에 더함). `POST /api/blogs`·`PATCH /api/blogs/{id}`·`GET /api/blogs`가 이 모양을 돌려줘요. `title`은 블로그 이름(RSS channel의 title)이에요
- **오류 코드(정함 11-12, 지원):** `TEXT_TOO_LONG`(400) 직접 쓴 글 본문이 길이 상한을 넘을 때, `TEXT_NOT_FOUND` 없는 글, `INVALID_BLOG`(400) 블로그 주소로 RSS를 찾지 못했거나 내부 주소일 때, `BLOG_NOT_FOUND` 없는 블로그. 같은 RSS 주소를 다시 등록하면 409 `BLOG_EXISTS`(정함 11-12, 지원)
- 블로그 글은 서버가 가져올 때 바로 `texts` 저장 → 큐로 가서 `POST /api/texts`를 안 거쳐요
- 카드 응답(CardSummary·CardDetail)에 `source`(`RECORDING`·`WRITTEN`·`BLOG`, 녹음 카드도 값을 적어요 — 정함 11-12, 지원)와 `textId`(글 카드만)를 더해요. `source`는 DB 열이 아니라 `recording_id`·`text_id`와 `texts.source`로 채워요. CardDetail에는 `truncated`도 더해요(초안). 글 카드는 `startMs`·`endMs`가 null이고 `recordedAt`은 위 "글 카드의 날짜"를 따라요

### 워커 (/internal)

| 메서드 | 주소 | 하는 일 | 보내는 것 |
| --- | --- | --- | --- |
| POST | /internal/texts/{id}/cards | 글 결과(카드 1장) 보내기 | jobId, cards\[1\] |

### 푸시

| type | 나머지 키 | 보내는 때 |
| --- | --- | --- |
| CARDS_CREATED | textId, cardCount(1) | 직접 쓴 글의 카드가 생겼을 때. 녹음과 같은 type이고 recordingId 대신 textId가 와요. 블로그 글은 보내지 않아요 |

## 7. DB (초안)

```sql
-- 등록한 블로그
CREATE TABLE blog_sources (
    id BIGINT NOT NULL GENERATED BY DEFAULT AS IDENTITY,
    user_id BIGINT NOT NULL,
    site_url VARCHAR(500) NOT NULL,     -- 사용자가 넣은 블로그 주소
    title VARCHAR(200),                 -- 블로그 이름(RSS channel의 title)
    feed_url VARCHAR(500) NOT NULL,     -- 서버가 찾은 RSS 주소
    platform VARCHAR(20) NOT NULL DEFAULT 'OTHER',  -- 연결할 때 고른 플랫폼(NAVER·TISTORY·VELOG·MEDIUM·WORDPRESS·OTHER). 서버 안에서만 씀(정함 11-14, 지원. 타입·기본값은 초안)
    categories JSONB NOT NULL DEFAULT '[]',  -- 고른 카테고리(문자열 배열). 비면 전체. 목록은 기존 Flyway(V11·V14)처럼 JSONB(정함 11-12, 지원)
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT blog_sources_pk PRIMARY KEY (id),
    CONSTRAINT uk_blog_sources_user_id_feed_url UNIQUE (user_id, feed_url)  -- 같은 RSS 주소 두 번 등록 막기
);
CREATE INDEX idx_blog_sources_user_id ON blog_sources (user_id ASC);

-- platform 열은 lolog-api V17로 나중에 더했어요(#82). 그보다 먼저 연결한 블로그는 주소로 채우고, 자기 도메인 블로그는 OTHER로 남겨요(초안)

-- 가져온 블로그 글. 같은 글을 다시 가져오지 않게 카드를 지워도, 연결을 끊어도 남김
CREATE TABLE blog_posts (
    id BIGINT NOT NULL GENERATED BY DEFAULT AS IDENTITY,
    source_id BIGINT,                   -- 연결을 끊으면 NULL
    user_id BIGINT NOT NULL,
    guid VARCHAR(500) NOT NULL,
    title VARCHAR(300) NOT NULL,
    link VARCHAR(500) NOT NULL,
    category VARCHAR(100),
    published_at TIMESTAMPTZ,           -- RSS pubDate. 없으면 NULL(카드 날짜는 created_at)
    blog_title VARCHAR(200),            -- 가져올 때 블로그 이름을 복사(연결을 끊어도 남게)
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,  -- 가져온 시각
    CONSTRAINT blog_posts_pk PRIMARY KEY (id),
    CONSTRAINT uk_blog_posts_user_id_guid UNIQUE (user_id, guid)  -- guid: RSS guid / Atom id / 없으면 link
);
ALTER TABLE blog_posts ADD CONSTRAINT fk_blog_posts_blog_sources FOREIGN KEY (source_id) REFERENCES blog_sources (id) ON DELETE SET NULL;

-- 카드로 만들 글(블로그 글·직접 쓴 글)
CREATE TABLE texts (
    id BIGINT NOT NULL GENERATED BY DEFAULT AS IDENTITY,
    user_id BIGINT NOT NULL,
    source VARCHAR(10) NOT NULL,        -- BLOG, WRITTEN
    blog_post_id BIGINT,                -- 블로그 글이면
    title VARCHAR(300) NOT NULL,
    body TEXT,                          -- 블로그 글은 카드가 생기면 NULL로 비움
    place VARCHAR(200),
    situation VARCHAR(500),
    truncated BOOLEAN NOT NULL DEFAULT false,  -- 길어서 앞부분만 썼는지. 직접 쓴 글은 늘 false
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
    CONSTRAINT texts_pk PRIMARY KEY (id),
    CONSTRAINT ck_texts_source CHECK (source IN ('BLOG', 'WRITTEN')),
    CONSTRAINT ck_texts_blog_post CHECK ((source = 'BLOG') = (blog_post_id IS NOT NULL))
);
-- (정함 11-12, 지원) blog_posts는 탈퇴 때만 지워지고 그때 texts를 먼저 지워요. SET NULL이면 ck_texts_blog_post와 부딪혀서 기본(NO ACTION)으로 둬요
ALTER TABLE texts ADD CONSTRAINT fk_texts_blog_posts FOREIGN KEY (blog_post_id) REFERENCES blog_posts (id);
CREATE INDEX idx_texts_user_id ON texts (user_id ASC);

-- 카드: 녹음 카드는 recording_id, 글 카드는 text_id. 글 카드는 시간 값이 없음
ALTER TABLE cards ADD COLUMN text_id BIGINT;
ALTER TABLE cards ALTER COLUMN start_ms DROP NOT NULL;
ALTER TABLE cards ALTER COLUMN end_ms DROP NOT NULL;
CREATE INDEX idx_cards_text_id ON cards (text_id);
ALTER TABLE cards ADD CONSTRAINT fk_cards_texts FOREIGN KEY (text_id) REFERENCES texts (id) ON DELETE SET NULL;  -- 글이 지워지면 카드는 남고 text_id만 비움, recording_id와 같음(정함 11-12, 지원)
-- 설계: recording_id 또는 text_id 중 하나. 녹음과 글을 동시에 가리키지 않게. 녹음을 지워 recording_id가 NULL이 된 카드도 통과해요
ALTER TABLE cards ADD CONSTRAINT ck_cards_origin CHECK (recording_id IS NULL OR text_id IS NULL);
-- 카드를 지우면 api가 그 texts 줄도 지움(정함). card_lines.start_ms·end_ms는 지금도 NULL 가능

-- 작업: job_type에 TEXT를 더하고 text_id로 가리킴. ck_jobs_type·ck_jobs_target을 같이 고침
ALTER TABLE jobs ADD COLUMN text_id BIGINT;
ALTER TABLE jobs ADD CONSTRAINT fk_jobs_texts FOREIGN KEY (text_id) REFERENCES texts (id) ON DELETE CASCADE;
```

- 녹음 카드인지 글 카드인지는 `cards.recording_id`·`text_id` 중 무엇이 있는지로, 블로그 글인지 직접 쓴 글인지는 `texts.source`로 알아요. 카드에 출처 칸을 따로 두지 않아요
- 서로 가리키는 칸은 `texts.blog_post_id` 하나만 둬요(정함 11-12, 지원)
- `cards.recording_id`는 지금 녹음을 지우면 NULL이 돼요(ON DELETE SET NULL). 그래서 "둘 중 하나는 꼭 있다"는 검사는 못 걸고, "둘 다 차지는 않는다"(`ck_cards_origin`)만 걸어요(설계 "recording_id 또는 text_id 중 하나")
- 글 상태(`QUEUED`·`DONE`·`FAILED`)는 칸으로 두지 않고 카드·`jobs`로 채워요(정함 11-12, 지원, 2장 "상태 채우기")
- 탈퇴(`UserDataService`)에 더할 것: 그 회원의 `texts`·`blog_posts`·`blog_sources`를 지워요. `blog_posts`는 연결을 끊어도 남게 SET NULL이라 외래 키로 함께 지워지지 않아서 `user_id`로 따로 지워요
- 마이그레이션 번호는 구현할 때 정해요(지금 docs 기준 V14 영상 다음)

## 8. 바꿀 곳

| 레포 | 할 일 |
| --- | --- |
| app | 글쓰기 화면(제목·본문·장소·상황), 카드 자세히의 "원문 보기"와 잘린 글 표시, 블로그 링크 넣기·확인 버튼과 그 밑 카테고리 선택(피드 목록 + 직접 치기), 카테고리 바꾸기, 연결한 블로그 목록·연결 끊기, 가져온 글 목록. 화면은 서현 담당 |
| api | `blog_sources`(+`platform`)·`blog_posts`·`texts` 표, 블로그 플랫폼 인터페이스(`BlogPlatform`, 구현 5개 + Other), `cards`·`jobs` 바꾸기, 글 API·블로그 API, 링크로 RSS 찾기(SSRF 막기), 30분마다 피드 읽기·카테고리 걸러내기·HTML 벗기기·길이 상한, 큐 넣기, 결과 주소, 푸시 |
| ai | `kind: text` 분기와 글용 프롬프트(민주). 블로그 글과 직접 쓴 글이 같은 길이라 따로 할 일은 없어요 |

## 9. AI 담당이 확정할 것

- 큐(`lolog:detect`에 같이)와 메시지 모양(글을 메시지에 바로)
- 글 하나 = 카드 하나, 이야기 감지 없이 바로 화자 구분
- 본문을 통째로 보내고 워커가 대사로 나누기
- 결과 모양 `{ jobId, cards: [카드 1장] }`
- 글에서 화자를 나누는 방법(서술이 섞인 글도 같은 프롬프트로 되는지), 글용 프롬프트
- 글 길이 상한 숫자

## 10. 구현에서 정한 것 (Claude 초안, 검토 필요)

명세에 없어서 구현하면서 Claude가 정한 것이에요. 지원이 정한 것이 아니에요.

### lolog-api (PR #73·#74)

| 정하지 않았던 점 | 정한 것 | 왜 |
| --- | --- | --- |
| 직접 쓴 글의 제목·장소·상황 길이 | 제목 300자·상황 500자(DB 열 길이), 장소 20자 | 장소는 그대로 카드 장소가 되어 카드 고치기의 장소 한도(20자)와 맞춤 |
| 공백만 있는 장소·상황 | 비운 것으로 봄(메시지에서 빠짐) | 제목·본문의 "공백만 있는 값 거절"과 같은 기준 |
| 글자 수 세는 법 | 글자(코드포인트) 수. 앱도 같게 셈 | 이모지가 2자로 세지지 않게 |
| 결과의 `startMs`·`endMs`에 값이 옴 | 거절하지 않고 null로 저장 | 워커 쪽 실수로 카드가 버려지지 않게 |
| 마이그레이션 번호 | `V15__texts.sql`(texts, cards·jobs), `V16__blogs.sql`(blog_sources·blog_posts, texts 외래 키) | 직접 쓴 글과 블로그를 PR 둘로 나눠서 |
| 링크 확인 카테고리 목록 | 30자가 넘는 이름은 뺌 | 고를 수 없는 이름(1~30자)이라서 |
| 카테고리 맞추기 | 앞뒤 공백을 뺀 뒤 글자가 같은지만 봄 | "같은지만 봐요"(4장) |
| 가져온 글 목록 순서 | 가져온 순서 내림차순. 한 번에 가져온 글은 쓴 시각 순서로 넣어서 결과는 최신 글부터 | 커서를 다른 목록처럼 마지막 ID로 두려고 |
| 저장할 수 없는 글 | 링크·guid가 없거나 500자를 넘으면 건너뜀, 제목이 없으면 "제목 없음", 본문이 비면 제목을 본문으로 | 열 길이·NOT NULL |
| `blog_posts.category` | 고른 카테고리 중 맞은 첫 이름(전부 받을 때는 글의 첫 카테고리) | 글 하나에 카테고리가 여럿일 수 있어서 |
| pubDate 읽기 | 요일은 무시하고 읽음. 못 읽으면 없는 것으로 봄(카드 날짜는 가져온 시각) | 요일이 날짜와 안 맞는 피드가 있어서 |
| RSS 찾기 | 네이버 `blog.naver.com/{id}`·`PostList.naver?blogId=` → `rss.blog.naver.com/{id}.xml`, 티스토리 → `/rss`, 그 밖은 그 주소가 피드인지 보고 HTML의 `<link rel="alternate">` | 4장 "플랫폼별 규칙 + 자동 찾기" |
| 내부 주소 범위 | 루프백·사설·링크 로컬(169.254 포함)·CGNAT(100.64/10)·IPv6 ULA·멀티캐스트·0.0.0.0/8 거절, 리다이렉트는 5번까지 하나씩 다시 확인 | 4장 "주소와 보안". DNS 답이 확인 뒤 바뀌는 경우(DNS rebinding)는 막지 못함 |
| 피드 받기 한도 | 3MB, 응답 10초, DOCTYPE 있는 XML 거절(XXE) | 큰 응답·느린 서버·XML 공격 |
| 처음 읽는 때 | 서버가 켜지고 5분 뒤, 그 뒤 30분마다 | 배포 직후 몰리지 않게 |
| 블로그 글 카드의 새 소식 점 | 알림 목록에 줄을 남기지 않고 `GET /api/me/badges`의 `cards`만 켬 | "푸시 없이 보관함 새 소식 점만"(2장) |
| `GET /api/blogs` 응답 | `{ blogs: [블로그 응답] }` | 다른 목록 응답 모양과 같게 |

### 앱 (PR #98)

| 정하지 않았던 점 | 정한 것 | 왜 |
| --- | --- | --- |
| 플랫폼 표시(N·T 동그라미) | 서버가 주지 않아서 앱이 블로그 주소로 정함 | 시안 디자인을 그대로 두려고 |
| 본문이 2,000자를 넘을 때 | 몰래 자르지 않고 글자 수를 빨갛게, 보내기를 막음 | "넘으면 거절하고 사용자가 줄여요"(2장) |
| 직접 쓴 글 카드 푸시 문구 | "쓴 글로 에피소드를 만들었어요" | 녹음 문구와 구분 |

