# 4컷 콘티 형식 (v1)

LLM이 카드로 만든 4컷 콘티의 모양이에요. Jetson 워커가 만들어 `POST /internal/comics/{id}/storyboard`로 메인 백엔드에 보내고, 그림 단계의 입력으로도 써요.

- 스키마: [`storyboard.schema.json`](storyboard.schema.json) (JSON Schema 2020-12)
- 예시: [`storyboard.example.json`](storyboard.example.json)
- 워커의 검사 코드: [ai 레포 `src/lolog_ai/storyboard.py`](https://github.com/LOLog-2026/ai/blob/develop/src/lolog_ai/storyboard.py) (`validate(storyboard, card)`)

형식을 바꿀 때는 이 문서, 스키마, 워커의 검사 코드를 같이 고쳐요.

## 어디에 쓰이나

```
카드(+사용자가 고친 화자·이름·대사·장소)
  → LLM 콘티 → 검사 ─(실패: 한 번 다시 묻고, 또 실패하면 FAILED)
  → POST /internal/comics/{id}/storyboard  ← 이 JSON이 본문
  → 그림: 컷 4개를 하나씩 FLUX.2 klein(+LoRA)로 그림 (글자 없이)
  → 말풍선·대사: 워커가 컷마다 dialogue를 한글 폰트로 붙임 (TR-07)
  → 합치기: 세로 4칸 한 장으로
  → 4컷 PNG → POST /internal/comics/{id}/image
```

## 정한 것

ai 레포 실험([ai#6](https://github.com/LOLog-2026/ai/issues/6))을 보고 정했어요.

| 항목 | 정한 것 | 근거 |
|---|---|---|
| 컷 배치 | **세로 4칸** | 폰 화면에서 컷이 화면 전체 너비라 대사가 길게 들어가요. 2×2는 컷이 화면 절반이라 말풍선 하나에 7–14자밖에 안 들어가요 |
| 그리는 방식 | **컷마다 따로 그려 코드로 합침** | 한 장에 4컷을 그리면 같은 인물이 둘로 복제되는 컷이 나왔어요(3/3). 컷마다 그리면 인물 수와 외모가 맞았어요. 시간은 4배(3090: 2.8초 → 6.3초) |
| 말풍선 대사 | **하나에 25자(공백 포함), 컷당 최대 2개** | 세로 4칸에서 폰 기준 읽을 수 있는 크기(약 12sp)로 최소 27자가 들어갔어요 |
| 대사 넣는 방식 | **워커가 코드로** (TR-07) | 그림 모델에 한글 대사를 쓰게 하면 33번 중 한 번도 정확하지 않았어요(`배고파` → `배고마`). 코드로 넣으면 18/18 정확했어요 |

## 필드

### 맨 위

| 필드 | 타입 | 설명 |
|---|---|---|
| `version` | `1` | 형식 버전 |
| `comicId`, `cardId` | int | 만화와 카드 ID |
| `layout` | `"vertical4"` | 컷 배치. 지금은 세로 4칸만 써요(사용자가 고르는 옵션은 없어요). 스키마에는 예비 값 `2x2`도 있어요 |
| `place.text` | string | 장소(한국어). 사용자가 적은 값이 있으면 그대로, 없으면 LLM이 정해요 |
| `place.prompt` | string | 그림 모델용 배경 설명(**영어**) |
| `characters[]` | array (1–4) | 등장인물 |
| `panels[]` | array (**정확히 4**) | 컷 |

### `characters[]`

| 필드 | 설명 |
|---|---|
| `key` | 카드의 `speakers[].key`와 같은 값(`A`, `B`…) |
| `name` | 사용자가 붙인 별명. 없으면 `null`. **그림 설명에는 쓰지 않아요** |
| `appearance` | 모든 컷에 똑같이 넣을 외모 설명(**영어**). 인물 일관성을 위한 필드예요 |

### `panels[]`

| 필드 | 설명 |
|---|---|
| `index` | 1–4 |
| `beat` | 1 `buildup` → 2 `development` → 3 `punchline` → 4 `reaction` |
| `scene` | 그림 모델용 장면 설명(**영어**). 이름이나 내부 키 없이 "the woman", "the man"처럼 써요. 컷마다 따로 그리므로 **이 컷만 보고 그릴 수 있게** 써요("again", "the same cake"처럼 앞 컷에 기대는 표현은 피해요) |
| `cast[]` | 이 컷에 나오는 인물: `key`, `position`(`left`/`center`/`right`, 한 컷 안에서 겹치면 안 됨), `expression`(선택, 영어). 말풍선을 어느 인물에게 붙일지 정할 때 써요 |
| `dialogue[]` | 말풍선(최대 2개, 읽는 순서). `speaker`(이 컷 `cast`에 있는 key), `text`(한국어, 최대 25자), `sourceLineIds`(선택, 근거가 된 카드 대사 id) |

그림 설명(`appearance`, `scene`, `place.prompt`)은 영어로 써요. 실험에서 영어 설명이 안정적이었고, 한글이 섞이면 그림 모델이 그림 안에 엉뚱한 글자를 그려 넣을 수 있어요.

## 검사 규칙

기능 명세 FR-COMIC의 콘티 검사를 워커가 이 규칙으로 해요.

| 규칙 | 이유 |
|---|---|
| 정확히 4컷, `beat` 순서 고정 | 4컷 구성 |
| 그림 설명에 `화자1`, `speaker A` 같은 내부 이름 없음 | 내부 이름이 그림에 섞이지 않게 |
| 그림 설명에 별명 없음 | 그림 모델은 이름을 모르니 외모 설명으로 그려야 해요 |
| 그림 설명에 한글 없음 | 위 설명 참고 |
| 말풍선 최대 2개, 대사 최대 25자, 빈 대사 없음 | 세로 4칸 실측(최소 27자). 말풍선은 2개까지만 확인했어요 |
| 대사의 화자가 그 컷 `cast`에 있음, 한 컷에서 `position`이 겹치지 않음 | 말풍선과 화자를 짝짓기 위해 |
| `characters`의 key가 카드의 화자에 있음 | 카드와 일관성 |

실명·주소 빼기(FR-COMIC-06)는 LLM 프롬프트에서 지시해요. 이 검사로는 "내부 이름"만 잡고, 실명 탐지는 하지 않아요.

## 아직 정할 것

- **나레이션 칸(예: "다음 날")**: 필요하면 `panels[].caption`을 더해요
