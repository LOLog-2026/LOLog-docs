# 배포 설정

서버에 올릴 때 쓰는 주소, 포트, 환경 변수를 모았어요. 각자 코드를 이 값에 맞춰 두면 인프라는 환경 변수만 넣어 바로 띄워요. API 주소와 데이터 모양은 [API 명세](api-spec.md)와 [API DTO](api-dto.md)가 기준이고, 여기에는 그 밖의 배포 값만 적었어요.

- **상태:** "확정"은 정한 값이에요. "백엔드가 정함"은 백엔드에서 정하면 인프라가 이 문서와 서버 설정을 고쳐요. "미정"은 나중에 정해요.
- **비밀값:** 실제 값은 레포에 올리지 않아요. 인프라 담당이 서버 `.env`와 GitHub secrets에만 넣어요.

## 1. 구성

| 위치 | 돌아가는 것 |
| --- | --- |
| 교육기관 서버 | Nginx, 인증(lolog-auth), 메인 백엔드(lolog-api), PostgreSQL, Redis, SeaweedFS. 모두 Docker Compose 한 파일(lolog-api `deploy/compose.yaml`)로 띄워요 |
| Jetson | 추론 워커(Python), LLM(OpenAI 호환 서버), FLUX.2 klein + LoRA |
| 연구실 서버 | LoRA 학습만 해요. 가중치는 Jetson에 올려요 |

- 교육기관 서버는 여러 학생이 같이 써요. 80은 학교 Nginx가 쓰고, 학교 Nginx가 `lolog.java21.net`을 우리 Nginx 컨테이너의 개인 포트(`127.0.0.1:NGINX_PORT`)로 넘겨요. 도메인 등록은 학교 Nginx 관리 웹사이트에서 해요.
- 밖에서 들어오는 길은 학교 Nginx 80 하나예요. 지금은 HTTP만 써요.
- Jetson은 SSH 터널로 교육기관 서버의 Redis와 메인 백엔드에 붙어요(TR-10). `/internal`은 Nginx를 거치지 않아요.
- SeaweedFS는 MinIO 대신 쓰는 S3 호환 저장소예요. 완성된 만화 이미지를 둬요(TR-06).

## 2. 도메인과 주소

| 항목 | 값 | 상태 |
| --- | --- | --- |
| 도메인 | `lolog.java21.net` (학교에서 받음) | 확정 |
| 앱이 부르는 주소 | `http://lolog.java21.net` 하나. `/auth`, `/api`를 같은 주소로 불러요 | 확정 |
| HTTPS | 쓰지 않아요. 학교 Nginx가 HTTP만 받아요. 나중에 되면 주소만 `https://`로 바꿔요 | 확정 |

**Nginx 경로** (우리 Nginx 컨테이너)

| 경로 | 넘기는 곳 | 비고 |
| --- | --- | --- |
| `/auth/**` | `auth:8081` | 경로를 자르지 않고 그대로 넘겨요 |
| `/api/**` | `api:8080` | 경로를 자르지 않고 그대로 넘겨요 |
| `/lolog-comics/**` | `seaweedfs:8333` | 만화 이미지(imageUrl). GET만 허용 |
| `/internal/**` | 막음 | 밖에 열지 않아요 |

- 요청 본문은 Nginx에서 5MB까지 받아요. 녹음 본문 1MB 제한(413)은 메인 백엔드가 검사해요.
- 로그인·가입 주소에는 요청 횟수 제한을 걸어요.

## 3. 서버 안 서비스와 포트

Compose 안에서는 서비스 이름으로 서로 불러요. 서버에 여는 포트는 아래 3개뿐이고 모두 `127.0.0.1`에만 열어요. 포트 번호는 개인 포트 범위 안에서 골라 서버 `.env`에 넣어요.

| 서비스 이름 | 하는 일 | 컨테이너 포트 | 상태 |
| --- | --- | --- | --- |
| `nginx` | 경로 나누기 | 80 | 확정 |
| `api` | 메인 백엔드(lolog-api) | 8080 | 확정 |
| `auth` | 인증(lolog-auth) | 8081 | 확정 |
| `postgres` | PostgreSQL | 5432 | 확정 |
| `redis` | Redis(작업 큐, 리프레시 토큰) | 6379 | 확정 |
| `seaweedfs` | SeaweedFS(S3 8333) | 8333 | 확정 |

| 서버 포트 | 넘기는 곳 | 쓰는 곳 |
| --- | --- | --- |
| `127.0.0.1:NGINX_PORT` | `nginx:80` | 학교 Nginx |
| `127.0.0.1:TUNNEL_API_PORT` | `api:8080` | Jetson 터널(`/internal`) |
| `127.0.0.1:TUNNEL_REDIS_PORT` | `redis:6379` | Jetson 터널(작업 큐) |

- 서버 밖에서는 안 보여요. 같은 서버의 다른 계정에서는 보이니 Redis 비밀번호와 `X-Internal-Token`을 꼭 써요.

## 4. 백엔드 (lolog-api, lolog-auth)

- 레포 루트에 `Dockerfile`을 둬요. Java 21로 빌드하고 `SPRING_PROFILES_ACTIVE=prod`로 띄워요.
- 컨트롤러 주소는 [API 명세](api-spec.md) 그대로 `/api/...`, `/auth/...`, `/internal/...`로 시작해요(Nginx가 경로를 자르지 않아요).
- 앞에 Nginx가 있어요. `server.forward-headers-strategy=framework`를 켜요.
- `GET /api/health`, `GET /auth/health`는 배포 헬스체크에 써요. 토큰 없이 200, DB·Redis에 못 붙으면 503이에요([API 명세](api-spec.md)).
- 로그는 표준 출력으로만 내고, 대화 텍스트·토큰·미리 서명한 주소는 남기지 않아요(TR-12).
- `application-prod.yml`은 아래 환경 변수 이름으로 값을 읽고, 값을 파일에 적지 않아요.

| 환경 변수 | 값(예) | api | auth | 상태 |
| --- | --- | --- | --- | --- |
| `SPRING_PROFILES_ACTIVE` | `prod` | O | O | 확정 |
| `JWT_SECRET` | 비밀값, 32바이트 이상 | O | O | 확정([API 명세](api-spec.md)) |
| `INTERNAL_TOKEN` | 비밀값. `X-Internal-Token` 헤더 값 | O | O | 확정([API 명세](api-spec.md)) |
| `DB_URL` | `jdbc:postgresql://postgres:5432/<DB 이름>?currentSchema=<스키마>` | O | O | 이름 확정, DB 이름·스키마는 백엔드가 정함 |
| `DB_USERNAME`, `DB_PASSWORD` | 비밀값 | O | O | 확정 |
| Redis 접속 | 예: `REDIS_HOST=redis`, `REDIS_PORT=6379`, `REDIS_PASSWORD` | O | O | 백엔드가 정함 |
| SeaweedFS 접속 | 5장 참고 | O |  | 백엔드가 정함 |
| `FCM_CREDENTIALS_PATH` | `/run/secrets/fcm.json` | O |  | 확정 |
| `API_INTERNAL_URL` | `http://api:8080`. 탈퇴 때 `DELETE /internal/users/{id}`를 부를 주소 |  | O | 확정 |
| 소셜 로그인 키 | 카카오·구글·네이버 |  | O | 미정 |

- 인증과 메인 백엔드는 같은 DB를 써요. Flyway 이력 테이블은 메인 백엔드가 `flyway_schema_history_api`이고, 인증 테이블 이름은 백엔드가 정해요.
- FCM 서비스 계정 키는 앱과 같은 Firebase 프로젝트에서 받아요. 파일은 서버에만 두고 컨테이너에 `/run/secrets/fcm.json`으로 넣어요.

## 5. SeaweedFS (만화 이미지)

| 항목 | 값(예) | 상태 |
| --- | --- | --- |
| 구성 | 컨테이너 하나로 `weed server -s3`, 복제 없음 | 확정 |
| 업로드·삭제 주소 | `http://seaweedfs:8333` (서버 안) | 확정 |
| imageUrl 서명 주소 | `http://lolog.java21.net` | 확정 |
| 버킷 | 예: `lolog-comics` | 백엔드가 정함 |
| 객체 키 | 예: `users/{userId}/comics/{comicId}.png` | 백엔드가 정함 |
| 환경 변수 | 예: `S3_ENDPOINT`, `S3_PUBLIC_ENDPOINT`, `S3_BUCKET`, `S3_REGION`, `S3_ACCESS_KEY`, `S3_SECRET_KEY` | 백엔드가 정함 |

- MinIO와 같은 S3 API라 AWS SDK S3 클라이언트를 그대로 써요. path-style을 켜요.
- 업로드용 클라이언트(서버 안 주소)와 imageUrl 서명용 클라이언트(공개 주소)를 따로 만들어요. 서명에 주소(Host)가 들어가서, 공개 주소로 서명해야 앱에서 열려요.
- imageUrl은 10분 뒤 만료돼요([API 명세](api-spec.md)). SDK가 요구하는 region은 `us-east-1` 같은 아무 값이면 돼요.
- 접근 키는 메인 백엔드 하나만 가져요. 버킷은 비공개예요.
- 만화를 지우거나 탈퇴하면 객체도 지워요(FR-LIB-03, FR-AUTH-05). 버킷 이름이 정해지면 Nginx 경로(2장)도 그 이름으로 바꿔요.

## 6. 앱 (lolog-app)

- 서버 주소는 `http://lolog.java21.net` 하나예요. 로컬과 배포 주소를 빌드 설정(`BuildConfig`)으로 바꿀 수 있게 둬요.
- 서버가 HTTP라 배포 빌드도 `lolog.java21.net`에만 평문 통신(cleartext)을 허용해요(network security config의 domain-config). 다른 주소는 막아 둬요.
- imageUrl도 같은 도메인으로 와요. 저장하지 않고 403이면 다시 조회해요([API 명세](api-spec.md)).
- FCM은 서버와 같은 Firebase 프로젝트를 써요.

## 7. Jetson 워커

- 워커는 SSH 터널 덕에 모두 `localhost`로 불러요. 서버 주소를 코드에 적지 않아요.
- 터널은 `autossh`를 systemd로 띄워, 끊기거나 재부팅해도 다시 붙어요. 서버 SSH는 `s2.java21.net:8822`이고, Jetson의 `localhost:6379`·`localhost:18080`을 서버의 `127.0.0.1:TUNNEL_REDIS_PORT`·`127.0.0.1:TUNNEL_API_PORT`로 이어요(`-L 6379:127.0.0.1:<TUNNEL_REDIS_PORT> -L 18080:127.0.0.1:<TUNNEL_API_PORT>`).
- `/internal`을 부를 때 `X-Internal-Token` 헤더를 붙여요.
- 작업이 끝나면 로컬 임시 파일(컷 그림 등)을 바로 지워요.

| 환경 변수 | 값(예) | 상태 |
| --- | --- | --- |
| `REDIS_URL` | `redis://:<REDIS_PASSWORD>@localhost:6379/0` | 확정 |
| `API_INTERNAL_URL` | `http://localhost:18080` | 확정 |
| `INTERNAL_TOKEN` | 비밀값 | 확정 |
| `LLM_BASE_URL` | `http://localhost:8000/v1` (Jetson의 OpenAI 호환 서버) | 확정 |
| `WORKER_NAME` | `jetson-1` (컨슈머 이름) | 확정 |

## 8. 배포 흐름

1. `main`에 머지하면 GitHub Actions가 테스트하고 이미지를 빌드해 GHCR에 올려요. 태그는 커밋 SHA예요.
2. Actions가 SSH로 서버에 들어가 `deploy/scripts/deploy.sh <api|auth> <SHA>`를 실행해요. 서버 `.env`의 이미지 태그를 바꾸고 그 서비스만 다시 띄운 뒤 헬스체크를 해요. 2분 안에 안 뜨면 이전 태그로 되돌려요.
3. 손으로 되돌릴 때는 서버에서 `deploy/scripts/deploy.sh api <이전 SHA>`를 실행해요.

- 배포 워크플로는 `main` push에서만 돌아요(레포가 public이라 PR에서는 배포하지 않아요).
- `main`에는 시연해도 되는 것만 올려요.
- 새 환경 변수가 필요하면 먼저 인프라 담당에게 알려 서버 `.env`와 secrets에 넣은 뒤 머지해요.
- 데이터는 Docker 볼륨에만 있고 백업은 하지 않아요. 볼륨을 지우는 `docker compose down -v`는 쓰지 않아요.

## 9. 비밀값

| 비밀값 | 가지는 곳 |
| --- | --- |
| `JWT_SECRET` | 인증, 메인 백엔드 |
| `INTERNAL_TOKEN` | 인증, 메인 백엔드, Jetson 워커 |
| DB 비밀번호 | 인증, 메인 백엔드, PostgreSQL |
| Redis 비밀번호 | 인증, 메인 백엔드, Redis, Jetson 워커 |
| SeaweedFS 접근 키 | 메인 백엔드, SeaweedFS |
| FCM 서비스 계정 키 | 메인 백엔드 |
| 소셜 로그인 키 | 인증 |
| 배포용 SSH 키 | GitHub Actions secrets |
| Jetson 터널 SSH 키 | Jetson |

## 10. 남은 것

- **백엔드가 정함:** DB 이름·스키마, 인증 Flyway 이력 테이블 이름, Redis·SeaweedFS 환경 변수 이름, 버킷 이름과 객체 키. 정해지면 인프라가 이 문서와 서버 설정을 고쳐요.
- **미정:** 소셜 로그인 키(나중에 정해요).
