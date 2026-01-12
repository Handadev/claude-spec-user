# Quickstart: Admin API Server

**Date**: 2026-01-12
**Feature**: 001-admin-api

## Prerequisites

- Java 21+
- Gradle 9.2.1+
- Docker & Docker Compose
- MariaDB 10.11+ (또는 Docker로 실행)

## 1. 로컬 환경 설정

### 1.1 애플리케이션 설정

**src/main/resources/application-local.properties**:
```properties
# Database
spring.datasource.url=jdbc:mariadb://localhost:3306/users
spring.datasource.username=handa
spring.datasource.password=handaDev!1

# JPA
spring.jpa.hibernate.ddl-auto=validate
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.format_sql=true

# Flyway
spring.flyway.enabled=true
spring.flyway.locations=classpath:db/migration

# JWT
jwt.secret=your-256-bit-secret-key-here-make-it-long-enough
jwt.expiration=3600000

# reCAPTCHA
recaptcha.site-key=your-site-key
recaptcha.secret-key=your-secret-key

# Logging
logging.level.org.hibernate.SQL=DEBUG
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### 1.3 빌드 및 실행

```bash
# 프로젝트 빌드
./gradlew build

# 로컬 프로파일로 실행
./gradlew bootRun --args='--spring.profiles.active=local'

# 또는 JAR 직접 실행
java -jar build/libs/spec-user-0.0.1-SNAPSHOT.jar --spring.profiles.active=local
```

## 2. 초기 데이터

Flyway 마이그레이션 실행 시 자동으로 생성되는 데이터:

**기본 권한:**
| ID | Name | Description |
|----|------|-------------|
| 1 | SUPER | 시스템 최고 관리자 |
| 2 | MANAGER | 관리자 |
| 3 | SUBMANAGER | 부관리자 |
| 4 | VIEWER | 조회 전용 |

**초기 관리자:**
| Email | Password | Role |
|-------|----------|------|
| admin@specuser.com | Admin123!@# | SUPER |

## 3. API 테스트
### 공통 응답 형식:
1. 기본 응답: data object 안에 key - value 형식
```json
{
  "message": "message",
  "data": {
    "key": " value"
  },
  "timestamp": "2026-01-12T10:00:00"
}
```
2. 목록 응답 형식: data object > content array 안에 key - value 형식
```json
{
  "message": "message",
  "data": {
    "content": [
      {
        "key": " value"    
      }
    ]
  },
  "timestamp": "2026-01-12T10:00:00"
}
```
3. 페이지네이션 응답 형식: data object > content array 안에 key - value 형식 \
data object > page(int), size(int), totalPages(int), totalElements(int), isLast(boolean) 존재
```json
{
  "message": "message",
  "data": {
    "content": [
      {
        "key": " value"    
      }
    ]
  },
  "page": 1,
  "size": 10,
  "totalPages": 2,
  "totalElements": 20,
  "isLast": false,
  "timestamp": "2026-01-12T10:00:00"
}
```

### 3.1 로그인

```bash
curl -X POST http://localhost:8080/api/v1/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "admin@specuser.com",
    "password": "Admin123!@#"
  }'
```

**응답:**
```json
{
  "success": true,
  "message": "로그인 성공",
  "data": {
    "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "tokenType": "Bearer",
    "expiresIn": 3600
  },
  "timestamp": "2026-01-12T10:00:00"
}
```

### 3.2 인증된 요청

```bash
# 환경 변수로 토큰 저장
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

# 현재 관리자 정보 조회
curl -X GET http://localhost:8080/api/v1/auth/me \
  -H "Authorization: Bearer $TOKEN"

# 관리자 목록 조회
curl -X GET http://localhost:8080/api/v1/admins \
  -H "Authorization: Bearer $TOKEN"

# 권한 목록 조회
curl -X GET http://localhost:8080/api/v1/roles \
  -H "Authorization: Bearer $TOKEN"

# 메뉴 목록 조회 (트리 구조)
curl -X GET http://localhost:8080/api/v1/menus \
  -H "Authorization: Bearer $TOKEN"

# 유저 목록 조회 (페이지네이션)
curl -X GET "http://localhost:8080/api/v1/users?page=0&size=20" \
  -H "Authorization: Bearer $TOKEN"
```

### 3.3 관리자 생성 (SUPER 권한)

```bash
curl -X POST http://localhost:8080/api/v1/admins \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "email": "manager@specuser.com",
    "password": "Manager123!@#",
    "name": "Manager User",
    "roleId": 2
  }'
```

### 3.4 메뉴 생성

```bash
# 상위 메뉴 생성
curl -X POST http://localhost:8080/api/v1/menus \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "관리자 관리",
    "url": "/admin",
    "sortOrder": 1
  }'

# 하위 메뉴 생성 (parentId 지정)
curl -X POST http://localhost:8080/api/v1/menus \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "관리자 목록",
    "url": "/admin/list",
    "sortOrder": 1,
    "parentId": 1
  }'
```

### 3.5 권한-메뉴 매핑

```bash
# MANAGER 권한에 메뉴 할당
curl -X PUT http://localhost:8080/api/v1/roles/2/menus \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "menuIds": [1, 2, 3]
  }'
```

### 3.6 유저 정지/해제

```bash
# 유저 정지
curl -X POST http://localhost:8080/api/v1/users/1/suspend \
  -H "Authorization: Bearer $TOKEN"

# 유저 정지 해제
curl -X POST http://localhost:8080/api/v1/users/1/activate \
  -H "Authorization: Bearer $TOKEN"
```

## 4. 테스트 실행

```bash
# 전체 테스트
./gradlew test

# 특정 테스트 클래스
./gradlew test --tests "org.specuser.service.AuthServiceTest"

# 통합 테스트만
./gradlew test --tests "*IntegrationTest"

# 테스트 커버리지
./gradlew check
```

## 5. API 문서

서버 실행 후 다음 URL에서 API 문서 확인:

- **Swagger UI**: http://localhost:8080/swagger-ui.html
- **OpenAPI JSON**: http://localhost:8080/v3/api-docs
- **OpenAPI YAML**: http://localhost:8080/v3/api-docs.yaml

## 6. 문제 해결

### 6.1 데이터베이스 연결 실패

```bash
# MariaDB 컨테이너 상태 확인
docker ps

# 로그 확인
docker logs mariadb

# 컨테이너 재시작
docker-compose restart mariadb
```

### 6.2 JWT 토큰 만료

```
{
  "message": "토큰이 만료되었습니다",
  "errorCode": "TOKEN_EXPIRED",
  "timestamp": "2026-01-12T10:00:00"
}
```

→ 다시 로그인하여 새 토큰 발급

### 6.3 권한 부족

```
{
  "message": "접근 권한이 없습니다",
  "errorCode": "ACCESS_DENIED",
  "timestamp": "2026-01-12T10:00:00"
}
```

→ SUPER 권한 계정으로 로그인 또는 권한 확인

## 7. 다음 단계

1. `/speckit.tasks` 명령으로 세부 태스크 생성
2. Constitution VII에 따라 `task_{YYYY-MM-DD}.md` 파일 생성
3. TDD 방식으로 테스트 먼저 작성 (Constitution II)
4. Red-Green-Refactor 사이클 진행
