# Quant-BE

Multi-Agent 기반 맞춤형 퀀트 투자·리포팅 서비스의 백엔드(Spring Boot)입니다.
인증, 투자 성향 분류, 포트폴리오·시장 리포트 조회, 가상 매매(리스크 검사) API를 제공합니다.

## 기술 스택
Java 21 · Spring Boot 4.0.6 (Web, Security, Data JPA, Data Redis, Validation) · Flyway · JJWT 0.12.6 · springdoc-openapi 2.8.8 · PostgreSQL 16 · Redis 7 · Gradle

## 디렉토리 구조
```text
src/main
├── java/com/example/quantserver
│   ├── global/       # 공통 (config, jwt, exception, response)
│   ├── auth/         # 회원가입·로그인·토큰
│   ├── user/         # 사용자 엔티티
│   ├── investment/   # 투자 성향 설문·분류
│   ├── dashboard/    # 대시보드 집계
│   ├── portfolio/    # 포트폴리오 추천 조회·월간 생성
│   ├── report/       # 시장 리포트 조회·일일 생성
│   ├── order/        # 가상 매매·리스크 검사·일별 스냅샷
│   ├── ai/           # AI 서버 클라이언트·에이전트 활동 로그
│   └── admin/        # 운영용 API
└── resources
    ├── application.yml
    ├── application-local.yml
    └── db/migration/ # Flyway V1~V9
```
각 도메인은 `controller / service / repository / entity / dto / enums`로 나뉩니다.

## 실행 방법

### 요구 사항
- JDK 21
- PostgreSQL 16, Redis 7 (직접 설치하거나 Docker로 실행)

### 1. 데이터베이스와 Redis 준비
`quant_db` 데이터베이스를 만들고 Redis를 실행합니다. Docker를 쓴다면 예시는 다음과 같습니다.
```bash
docker run -d -p 5432:5432 \
  -e POSTGRES_DB=quant_db \
  -e POSTGRES_USER=<DB 사용자> \
  -e POSTGRES_PASSWORD=<DB 비밀번호> \
  postgres:16-alpine

docker run -d -p 6379:6379 redis:7-alpine
```

### 2. 설정
애플리케이션은 환경 변수로 설정하며, 비밀 값은 저장소에 커밋하지 않습니다.

| 환경 변수 | 설명 |
|---|---|
| `JWT_SECRET` | JWT 서명 키 (임의의 긴 문자열) |
| `DB_URL`, `DB_USERNAME`, `DB_PASSWORD` | PostgreSQL 접속 정보 |
| `REDIS_HOST`, `REDIS_PORT` | Redis 접속 정보 |
| `AI_SERVER_URL` | AI 서버 주소 (선택) |

`local` 프로필(기본값)은 `application-local.yml`의 DB·Redis 접속 정보를 사용하므로, 이 파일을 본인 환경에 맞게 수정하면 `JWT_SECRET`만 지정해도 실행됩니다. 환경 변수로 모두 지정하려면 프로필을 비우고 실행합니다.

### 3. 서버 실행

**Gradle**
```bash
export JWT_SECRET=<임의의 시크릿 키>
./gradlew bootRun
```

**IntelliJ**
1. `QuantServerApplication`을 실행 대상으로 선택합니다.
2. Run Configuration의 Environment variables에 위 환경 변수를 등록합니다.
3. Run을 누릅니다.

**JAR**
```bash
./gradlew build -x test
export JWT_SECRET=<임의의 시크릿 키>
java -jar build/libs/*.jar
```

**환경 변수로 모두 지정할 때**
```bash
export SPRING_PROFILES_ACTIVE=
export DB_URL=jdbc:postgresql://<호스트>:5432/quant_db
export DB_USERNAME=<DB 사용자>
export DB_PASSWORD=<DB 비밀번호>
export REDIS_HOST=<호스트>
export REDIS_PORT=6379
export JWT_SECRET=<임의의 시크릿 키>
./gradlew bootRun
```

### 4. 확인
- 서버 포트는 8080입니다. 이미 사용 중이면 기존 프로세스를 종료한 뒤 실행하세요.
- 스키마는 기동 시 Flyway가 자동으로 생성합니다.
- Swagger UI: http://localhost:8080/swagger-ui.html (Authorize에 액세스 토큰 입력)

### 관리자 계정
가입한 사용자는 모두 `ROLE_USER`입니다. 관리자 API를 쓰려면 DB에서 권한을 올린 뒤 다시 로그인합니다.
```sql
UPDATE users SET role = 'ROLE_ADMIN' WHERE email = '<이메일>';
```

## 모듈별 매뉴얼

| 모듈 | 주요 API | 설명 |
|---|---|---|
| auth | `POST /api/auth/signup`, `login`, `logout`, `refresh`, `DELETE /withdraw` | 액세스 토큰 1시간, 리프레시 토큰 7일. 비밀번호는 BCrypt, 리프레시 토큰은 Redis `RT:{userId}`, 로그아웃 토큰은 `BL:{토큰}`에 저장 |
| investment | `GET·POST·PUT /api/profile` | 설문(리스크 허용도, 투자 기간, 투자 가능 금액)을 점수화해 STABLE / NEUTRAL / AGGRESSIVE로 분류. 수정 시 이전 프로필은 이력으로 보존 |
| dashboard | `GET /api/dashboard` | 총자산, 손익, 보유 종목, 오늘의 리포트를 한 번에 조회 |
| portfolio | `GET /api/portfolio/latest` | 최신 추천 조회. 매월 1일 스케줄러가 사용자별 추천을 생성해 저장 |
| report | `GET /api/reports`, `/{id}` | 시장 리포트 조회. 평일 08:00에 생성해 저장 |
| order | `POST·GET /api/orders`, `/stats`, `/stocks` | 가상 매매. 계좌 잠금과 리스크 검사 후 체결을 요청 |
| ai | `GET /api/agent-logs` | AI 서버 호출 클라이언트와 에이전트 활동 로그(최근 20건) |
| admin | `/api/admin/**` | 활동 로그 조회, 포트폴리오·리포트 수동 생성 (`ROLE_ADMIN`) |

**리스크 규칙** (주문 시 순서대로 검사)

| 규칙 | 차단 조건 | 오류 코드 |
|---|---|---|
| 손실 한도 | 총자산이 초기 잔고 대비 15% 초과 손실 | `T002` |
| 일일 거래 횟수 | 당일 체결 10건 이상 | `T003` |
| 단일 종목 집중도 | 매수 후 종목 비중 20% 초과 | `T004` |

**스케줄러** (Asia/Seoul)

| 작업 | 주기 |
|---|---|
| 시장 리포트 생성 | 평일 08:00 |
| 포트폴리오 추천 생성 | 매월 1일 09:00 |
| 일별 총자산 스냅샷 | 평일 19:00 |

## 데이터베이스

스키마는 Flyway(`db/migration`)로 관리하며, JPA는 `ddl-auto: validate`입니다.

| 영역 | 테이블 |
|---|---|
| 사용자·투자 성향 | `users`, `investment_profiles` |
| 포트폴리오 추천 | `portfolio_recommendations`, `portfolio_items` |
| 거래 | `portfolios`, `holdings`, `orders`, `portfolio_snapshots`, `stocks` |
| 시장 데이터·운영 | `news_articles`, `news_sentiments`, `market_reports`, `agent_activity_logs`, `collection_failures` |

- `portfolios`, `holdings`, `orders`, `portfolio_snapshots`는 `user_id`로 `users`를 논리적으로 참조합니다(FK 제약 없음). `holdings`는 `(user_id, stock_code)`, `portfolio_snapshots`는 `(user_id, date)`가 유일합니다.
- Redis 키: `RT:{userId}`(리프레시 토큰), `BL:{액세스 토큰}`(로그아웃 차단), `price:{종목코드}`(현재가 캐시)

## API 문서
- 요청·응답 명세: [`docs/API_SPEC.md`](docs/API_SPEC.md)
- 모든 응답은 `{ success, code, message, data }` 형식입니다.
