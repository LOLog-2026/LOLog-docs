# LOLog Docs

LOLog 기능·API 명세 문서예요. 문서는 PR로 고쳐요.

## 범위와 기준

아키텍처 v4와 기획\_v3에서 정한 내용을 기준으로 썼어요. 문서는 세 개예요. 요구사항 문서는 사용자와 서비스가 무엇을 할 수 있어야 하는지와 다 됐는지 판단할 확인 기준(기능 요구사항), 어떤 기술로 만드는지(기술 요구사항)예요. 기능 명세 문서는 기능마다 화면, 입력, 처리, 출력, 예외, API 명세 문서는 앱과 서버가 주고받는 주소와 데이터예요.

- **우선순위:** P0는 2주 안에 만드는 기본 범위예요. 추가 구현은 P1과 P2로 나눠 적었어요.
- **미정:** 아직 정하지 않은 것은 표에 "미정"으로 표시하고, 요구사항 문서의 "미정 항목"에 모았어요.
- **근거 문서:** [기획\_v3](https://claude.ai/artifact/MDmwPswUnAWxSA9HiRfSD2) · [아키텍처 v4](https://claude.ai/artifact/MKLtqnLEBD58h3AEHKKKew) · 추가 구현 구조는 [아키텍처 v5](https://claude.ai/artifact/UUzS1Cxfqv3aaTUAFj1SEj)

## 문서

| 문서 | 내용 |
| --- | --- |
| [요구사항](spec/requirements.md) | 기능 요구사항(확인 기준·우선순위), 기술 요구사항, 미정 항목 |
| [기능 명세](spec/functional-spec.md) | P0 기능마다 화면, 입력, 처리, 출력, 예외, 관련 API |
| [API 명세](spec/api-spec.md) | 인증·메인 백엔드·워커 전용 API, 큐 메시지 |
