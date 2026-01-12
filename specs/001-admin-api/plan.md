# Implementation Plan: Admin API Server

**Branch**: `001-admin-api` | **Date**: 2026-01-12 | **Spec**: [spec.md](./spec.md)
**Input**: Feature specification from `/specs/001-admin-api/spec.md`

## Summary

관리자 관리 사이트를 위한 RESTful API 서버 구현. 관리자 CRUD, 권한 관리(SUPER/MANAGER/SUBMANAGER/VIEWER), 2단계 계층 메뉴 관리, 유저 조회/정지 기능을 제공한다. Spring Boot 4.0.1 + Java 21 + MariaDB 기반으로 구현하며, JWT 토큰 인증과 Spring Security를 활용한 역할 기반 접근 제어를 적용한다.

## Technical Context

**Language/Version**: Java 21
**Primary Dependencies**: Spring Boot 4.0.1, Spring Security, Spring Data JPA, Lombok, JJWT
**Storage**: MariaDB (localhost:3306/users)
**Testing**: JUnit 5, Mockito, Spring Boot Test, Spring REST Docs
**Target Platform**: Linux server (Docker containerized)
**Project Type**: Single project (Backend API only)
**Performance Goals**: P99 < 200ms, 50+ concurrent admins, 10k+ users pagination < 3s
**Constraints**: P99 < 200ms, Stateless JWT authentication
**Scale/Scope**: 50+ admins, 10k+ users, 20+ menus

## Constitution Check

*GATE: Must pass before Phase 0 research. Re-check after Phase 1 design.*

| Principle | Gate Question | Status |
|-----------|---------------|--------|
| I. Code Quality | Does the design use Ubiquitous Language matching API/Domain terms? | ✅ |
| II. TDD | Is Red-Green-Refactor cycle planned for all features? | ✅ |
| III. Performance | Are SLA targets (P99 < 200ms) defined? N+1 detection planned? | ✅ |
| IV. Spec-First | Is OpenAPI/Proto schema frozen before implementation? | ✅ |
| V. Security | Is input validation and DTO-only response enforced? | ✅ |
| VI. DX | Is code generation from spec configured? Local env reproducible? | ✅ |
| VII. Task Mgmt | Is daily task file (`task_{YYYY-MM-DD}.md`) workflow established? | ✅ |

*Legend: ✅ Pass | ⬜ Pending | ❌ Violation (must justify in Complexity Tracking)*

### Constitution Compliance Notes

- **I. Code Quality**: Entity/DTO 명칭이 spec의 Key Entities와 1:1 매핑 (Admin, Role, Menu, RoleMenu, User)
- **II. TDD**: 각 User Story별 테스트 먼저 작성 후 구현 (Red-Green-Refactor)
- **III. Performance**: SC-002, SC-005, SC-007에 응답 시간 목표 명시됨. Hibernate Query 검증으로 N+1 탐지
- **IV. Spec-First**: OpenAPI 3.0 스키마를 contracts/ 디렉토리에 먼저 정의
- **V. Security**: 모든 요청은 DTO로 변환, @Valid 검증, Role 기반 접근 제어 적용
- **VI. DX**: docker-compose.yml로 MariaDB 로컬 환경 구성, Spring REST Docs로 API 문서화
- **VII. Task Mgmt**: todo.json 및 task_{YYYY-MM-DD}.md 워크플로우 적용

## Project Structure

### Documentation (this feature)

```text
specs/001-admin-api/
├── plan.md              # This file
├── research.md          # Phase 0 output
├── data-model.md        # Phase 1 output
├── quickstart.md        # Phase 1 output
├── contracts/           # Phase 1 output (OpenAPI specs)
│   └── openapi.yaml
└── tasks.md             # Phase 2 output (/speckit.tasks command)
```

### Source Code (repository root)

```text
src/main/java/org/specuser/
├── config/
│   ├── SecurityConfig.java
│   ├── JwtConfig.java
│   └── WebMvcConfig.java
├── controller/
│   ├── AuthController.java
│   ├── AdminController.java
│   ├── RoleController.java
│   ├── MenuController.java
│   └── UserController.java
├── dto/
│   ├── request/
│   │   ├── LoginRequest.java
│   │   ├── AdminCreateRequest.java
│   │   ├── AdminUpdateRequest.java
│   │   ├── RoleCreateRequest.java
│   │   ├── MenuCreateRequest.java
│   │   └── UserStatusUpdateRequest.java
│   └── response/
│       ├── LoginResponse.java
│       ├── AdminResponse.java
│       ├── RoleResponse.java
│       ├── MenuResponse.java
│       ├── UserResponse.java
│       └── ApiResponse.java
├── entity/
│   ├── Admin.java
│   ├── Role.java
│   ├── Menu.java
│   ├── RoleMenu.java
│   └── User.java
├── enums/
│   ├── AdminStatus.java
│   └── UserStatus.java
├── repository/
│   ├── AdminRepository.java
│   ├── RoleRepository.java
│   ├── MenuRepository.java
│   ├── RoleMenuRepository.java
│   └── UserRepository.java
├── service/
│   ├── AuthService.java
│   ├── AdminService.java
│   ├── RoleService.java
│   ├── MenuService.java
│   └── UserService.java
├── security/
│   ├── JwtTokenProvider.java
│   ├── JwtAuthenticationFilter.java
│   └── CustomUserDetailsService.java
├── exception/
│   ├── GlobalExceptionHandler.java
│   ├── BusinessException.java
│   ├── ErrorCode.java
│   └── CustomException.java
└── SpecUserApplication.java

src/main/resources/
├── application.properties
├── application-local.properties
└── db/migration/
    ├── V1__create_role_table.sql
    ├── V2__create_admin_table.sql
    ├── V3__create_menu_table.sql
    ├── V4__create_role_menu_table.sql
    ├── V5__create_user_table.sql
    └── V6__insert_initial_data.sql

src/test/java/org/specuser/
├── controller/
│   ├── AuthControllerTest.java
│   ├── AdminControllerTest.java
│   ├── RoleControllerTest.java
│   ├── MenuControllerTest.java
│   └── UserControllerTest.java
├── service/
│   ├── AuthServiceTest.java
│   ├── AdminServiceTest.java
│   ├── RoleServiceTest.java
│   ├── MenuServiceTest.java
│   └── UserServiceTest.java
└── integration/
    └── AdminApiIntegrationTest.java
```

**Structure Decision**: Spring Boot 표준 패키지 구조 사용. 기존 `org.specuser` 패키지 기반으로 controller, service, repository, entity, dto 계층 분리. Flyway 마이그레이션으로 DB 스키마 관리.

## Complexity Tracking

> **Fill ONLY if Constitution Check has violations that must be justified**

| Violation | Why Needed | Simpler Alternative Rejected Because |
|-----------|------------|-------------------------------------|
| N/A | 모든 원칙 준수 | - |
