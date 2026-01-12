# Research: Admin API Server

**Date**: 2026-01-12
**Feature**: 001-admin-api

## 1. Authentication Strategy

### Decision: JWT (JSON Web Token) 기반 Stateless 인증
- accessToken, refreshToken 을 사용한 관리
- accessToken(1시간), refreshToken(2달)로 관리
- accessToken 갱신시 refreshToken도 갱신하여 token table에 업데이트함
- accessToken 갱신시 refreshToken이 존재하는지 검증해야함
- 관리자 상태가 LOCKED, INACTIVE라면 인증 실패 응답 돌려주고, token table에서 삭제
- refreshToken에는 최소한의 정보만 들어있음

### Rationale
- Spring Boot 4.x에서 표준적으로 지원되는 인증 방식
- Stateless 특성으로 수평 확장(Scale-out)에 유리
- 프론트엔드(관리자 사이트)와의 API 통신에 적합
- 세션 저장소 불필요로 인프라 단순화

### Alternatives Considered
| 방식 | 장점 | 단점 | 선택 이유 |
|------|------|------|-----------|
| Session 기반 | 서버 측 세션 제어 용이 | Scale-out 시 세션 공유 필요 (Redis 등) | 인프라 복잡도 증가 |
| OAuth2 | 표준화된 인증 프로토콜 | 단순 관리자 인증에 과도한 복잡성 | 외부 IdP 연동 불필요 |
| **JWT** | Stateless, 확장성 우수 | 토큰 무효화 어려움 | 관리자 수 제한적, 적합 |

### Implementation Details
- Access Token: 1시간 유효
- Refresh Token: 7일 유효 (선택적 구현)
- 비밀번호: BCrypt 해시 (strength 12)
- JWT Library: jjwt-api 0.12.x

---

## 2. Password Security

### Decision: BCrypt with Strength 12

### Rationale
- Spring Security 기본 권장 알고리즘
- Strength 12는 보안성과 성능의 균형점 (약 250ms/hash)
- 무차별 대입 공격 방어에 효과적

### Alternatives Considered
| 알고리즘 | 장점 | 단점 | 선택 이유 |
|----------|------|------|-----------|
| SHA-256 | 빠름 | Rainbow table 공격 취약 | 비밀번호용 부적합 |
| Argon2 | 최신, 메모리 기반 | 라이브러리 추가 필요 | Spring Security 기본 미지원 |
| **BCrypt** | 표준, Spring 기본 지원 | 상대적으로 느림 | 업계 표준, 즉시 사용 가능 |

---

## 3. Login Failure Handling

### Decision: 3회 실패 시 CAPTCHA + 10회 실패 시 계정 잠금

### Rationale
- Brute Force 공격 방어와 사용자 편의성의 균형
- CAPTCHA로 자동화된 공격 차단
- 계정 잠금으로 지속적인 시도 차단

### Implementation Details
- 로그인 실패 카운트: Redis 또는 DB 저장 (IP + Email 기준)
- 성공 시 카운트 초기화
- CAPTCHA: Google reCAPTCHA v2 또는 hCaptcha
- 잠금 해제: SUPER 권한 관리자만 가능

---

## 4. Soft Delete Strategy

### Decision: INACTIVE 상태 전환 + 90일 후 Batch Hard Delete

### Rationale
- 실수로 삭제된 계정 복구 가능 (90일 이내)
- 감사 추적 유지
- 스토리지 효율성 (90일 후 정리)

### Implementation Details
- `deletedAt` 컬럼 추가 (null = 활성, not null = 삭제 시점)
- `status` = INACTIVE로 상태 전환
- 스케줄러: 매일 00:00 UTC에 90일 경과 레코드 Hard Delete
- Spring @Scheduled 또는 별도 배치 Job

---

## 5. Menu Hierarchy Implementation

### Decision: Self-referencing Foreign Key (Adjacency List)

### Rationale
- 2단계 계층만 지원하므로 단순한 구조로 충분
- JPA @ManyToOne 자기 참조로 구현
- 쿼리 단순화 (N+1 방지를 위해 fetch join 사용)
- children 호출진행하며 n+1 방지를 위해 메뉴를 findAll 로 가져온 다음 어플리케이션 단에서 로직 처리

### Alternatives Considered
| 패턴 | 장점 | 단점 | 선택 이유 |
|------|------|------|-----------|
| Nested Set | 조회 빠름 | 삽입/삭제 복잡 | 2단계에 과도함 |
| Materialized Path | 경로 조회 용이 | 문자열 파싱 필요 | 불필요한 복잡성 |
| **Adjacency List** | 단순, JPA 호환 | 깊은 계층 조회 느림 | 2단계만 지원, 적합 |

### Implementation Details
```java
@Entity
public class Menu {
    @Id
    private Long id;
    private String name;
    private String url;
    private Integer sortOrder;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "parent_id")
    private Menu parent;

    @OneToMany(mappedBy = "parent", cascade = CascadeType.ALL)
    private List<Menu> children;
}
```

---

## 6. Role-Menu Access Control

### Decision: Many-to-Many with RoleMenu Junction Table

### Rationale
- 권한과 메뉴 간의 다대다 관계 명확히 표현
- 추가 속성 확장 가능 (읽기/쓰기 권한 등)
- JPA @ManyToMany 직접 사용보다 유연함

### Implementation Details
- RoleMenu 엔티티로 관계 관리
- 메뉴 조회 시 현재 사용자 권한으로 필터링
- Spring Security @PreAuthorize 활용

---

## 7. Pagination Strategy

### Decision: Spring Data JPA Pageable + Cursor 기반 하이브리드

### Rationale
- 기본은 Offset 기반 Pageable (관리자 목록 - 소규모)
- 유저 목록 (10k+)은 성능상 Cursor 기반 고려

### Implementation Details
- 관리자/권한/메뉴: Pageable (Page, Size, Sort)
- 유저 목록: Pageable + 인덱스 최적화
- 기본 페이지 크기: 20
- 최대 페이지 크기: 100

---

## 8. API Response Format

### Decision: 표준화된 ApiResponse Wrapper

### Rationale
- 일관된 응답 형식으로 프론트엔드 개발 편의성 향상
- 에러 처리 표준화
- Constitution V. Security 원칙 준수 (DTO만 반환)

### Implementation Details
```java
public class ApiResponse<T> {
    private String message;
    private T data;
    private LocalDateTime timestamp;
    private String errorCode; // 실패 시
}
```

---

## 9. N+1 Query Prevention

### Decision: EntityGraph + Fetch Join + 쿼리 로깅 검증

### Rationale
- Constitution III. Performance 원칙 준수
- Hibernate 쿼리 카운트 검증으로 개발 단계에서 탐지

### Implementation Details
- `@EntityGraph` 사용하여 연관 엔티티 즉시 로딩
- 테스트에서 쿼리 카운트 assertion
- application-local.properties에 SQL 로깅 활성화

---

## 10. CAPTCHA Integration

### Decision: Google reCAPTCHA v2

### Rationale
- 무료, 널리 사용되는 솔루션
- 서버 측 검증 API 제공
- 봇 방어에 효과적

### Alternatives Considered
| 솔루션 | 장점 | 단점 | 선택 이유 |
|--------|------|------|-----------|
| reCAPTCHA v3 | 사용자 개입 없음 | 점수 기반 불확실성 | 명확한 차단 필요 |
| **reCAPTCHA v2** | 명확한 검증 | 사용자 클릭 필요 | 보안 요구사항에 적합 |
| hCaptcha | 프라이버시 우선 | 덜 알려짐 | 대안으로 고려 가능 |

### Implementation Details
- Site Key / Secret Key 환경 변수로 관리
- 3회 실패 후 로그인 요청에 captchaToken 필수
- 서버에서 reCAPTCHA API로 토큰 검증

---

## Summary

| 영역 | 결정 | 핵심 이유 |
|------|------|-----------|
| 인증 | JWT | Stateless, 확장성 |
| 비밀번호 | BCrypt (strength 12) | Spring 기본, 업계 표준 |
| 로그인 실패 | 3회 CAPTCHA + 10회 잠금 | 보안과 편의성 균형 |
| 삭제 전략 | Soft Delete + 90일 Hard Delete | 복구 가능성 + 스토리지 효율 |
| 메뉴 계층 | Adjacency List | 2단계 지원에 단순함 |
| 권한-메뉴 | RoleMenu Junction Table | 유연한 다대다 관계 |
| 페이지네이션 | Spring Pageable | 표준, 충분한 성능 |
| 응답 형식 | ApiResponse Wrapper | 일관성, 프론트엔드 편의 |
| N+1 방지 | EntityGraph + 쿼리 검증 | 성능 원칙 준수 |
| CAPTCHA | reCAPTCHA v2 | 무료, 효과적 |
