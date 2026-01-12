# Data Model: Admin API Server

**Date**: 2026-01-12
**Feature**: 001-admin-api

## Entity Relationship Diagram

```
┌─────────────┐       ┌─────────────┐       ┌─────────────┐
│    Role     │       │   RoleMenu  │       │    Menu     │
├─────────────┤       ├─────────────┤       ├─────────────┤
│ id (PK)     │◄──────│ role_id(FK) │       │ id (PK)     │
│ name        │       │ menu_id(FK) │──────►│ name        │
│ description │       │ created_at  │       │ url         │
│ created_at  │       └─────────────┘       │ sort_order  │
│ updated_at  │                             │ parent_id   │──┐
└─────────────┘                             │ created_at  │  │
       │                                    │ updated_at  │  │
       │                                    └─────────────┘  │
       │                                           ▲         │
       │                                           └─────────┘
       │                                           (self-ref)
       ▼
┌─────────────┐
│    Admin    │
├─────────────┤
│ id (PK)     │
│ email       │
│ password    │
│ name        │
│ role_id(FK) │
│ status      │
│ login_fail  │
│ last_login  │
│ deleted_at  │
│ created_at  │
│ updated_at  │
└─────────────┘

┌─────────────┐
│    User     │
├─────────────┤
│ id (PK)     │
│ email       │
│ name        │
│ status      │
│ created_at  │
│ updated_at  │
└─────────────┘
```

## Entities

### 1. Role (권한)

관리자의 권한 유형을 정의합니다.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | 고유 식별자 |
| name | VARCHAR(50) | NOT NULL, UNIQUE | 권한 이름 (SUPER, MANAGER, SUBMANAGER, VIEWER) |
| description | VARCHAR(255) | NULLABLE | 권한 설명 |
| created_at | DATETIME | NOT NULL, DEFAULT NOW | 생성 일시 |
| updated_at | DATETIME | NOT NULL, DEFAULT NOW ON UPDATE | 수정 일시 |

**Initial Data:**
- SUPER: 모든 권한, 시스템 최고 관리자
- MANAGER: 관리자 생성(SUBMANAGER/VIEWER만), 메뉴/유저 관리
- SUBMANAGER: 유저 관리만 가능
- VIEWER: 조회만 가능

**Validation Rules:**
- name: 2-50자, 영문 대문자 + 언더스코어만 허용
- 기본 권한(SUPER, MANAGER, SUBMANAGER, VIEWER)은 삭제 불가

---

### 2. Admin (관리자)

시스템 관리자 계정을 정의합니다.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | 고유 식별자 |
| email | VARCHAR(255) | NOT NULL, UNIQUE | 로그인 이메일 |
| password | VARCHAR(255) | NOT NULL | BCrypt 해시된 비밀번호 |
| name | VARCHAR(100) | NOT NULL | 관리자 이름 |
| role_id | BIGINT | FK → Role(id), NOT NULL | 권한 참조 |
| status | ENUM | NOT NULL, DEFAULT 'ACTIVE' | 계정 상태 |
| login_fail_count | INT | NOT NULL, DEFAULT 0 | 로그인 실패 횟수 |
| last_login_at | DATETIME | NULLABLE | 마지막 로그인 일시 |
| deleted_at | DATETIME | NULLABLE | 삭제(비활성화) 일시 |
| created_at | DATETIME | NOT NULL, DEFAULT NOW | 생성 일시 |
| updated_at | DATETIME | NOT NULL, DEFAULT NOW ON UPDATE | 수정 일시 |

**AdminStatus Enum:**
- `ACTIVE`: 정상 활성 상태
- `LOCKED`: 잠김 상태 (30일 미로그인 또는 10회 로그인 실패)
- `INACTIVE`: 삭제됨 (Soft Delete)

**Validation Rules:**
- email: 유효한 이메일 형식, 255자 이내
- password: 최소 8자, 영문+숫자+특수문자 조합
- name: 2-100자

**State Transitions:**
```
ACTIVE ──(30일 미로그인)──► LOCKED
ACTIVE ──(10회 로그인 실패)──► LOCKED
LOCKED ──(SUPER가 해제)──► ACTIVE
ACTIVE ──(삭제 요청)──► INACTIVE
INACTIVE ──(90일 경과)──► [Hard Delete]
```

**Indexes:**
- `idx_admin_email`: email (로그인 조회)
- `idx_admin_status`: status (활성 관리자 필터링)
- `idx_admin_role_id`: role_id (권한별 조회)
- `idx_admin_deleted_at`: deleted_at (배치 삭제용)

---

### 3. Menu (메뉴)

관리자 화면의 메뉴 항목을 정의합니다. 2단계 계층 구조를 지원합니다.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | 고유 식별자 |
| name | VARCHAR(100) | NOT NULL | 메뉴 이름 |
| url | VARCHAR(255) | NOT NULL | 메뉴 URL 경로 |
| sort_order | INT | NOT NULL, DEFAULT 0 | 같은 레벨 내 순서 |
| parent_id | BIGINT | FK → Menu(id), NULLABLE | 상위 메뉴 참조 (NULL = 상위 메뉴) |
| created_at | DATETIME | NOT NULL, DEFAULT NOW | 생성 일시 |
| updated_at | DATETIME | NOT NULL, DEFAULT NOW ON UPDATE | 수정 일시 |

**Validation Rules:**
- name: 2-100자
- url: 유효한 URL 경로 형식, /로 시작
- sort_order: 0 이상 정수
- parent_id: 존재하는 상위 메뉴만 참조 가능, 순환 참조 불가

**Hierarchy Rules:**
- 최대 2단계 (상위 메뉴 → 하위 메뉴)
- 상위 메뉴 삭제 시 하위 메뉴도 함께 삭제 (CASCADE)
- 하위 메뉴가 없는 상위 메뉴도 허용

**Indexes:**
- `idx_menu_parent_id`: parent_id (계층 조회)
- `idx_menu_sort_order`: parent_id, sort_order (정렬된 목록 조회)

---

### 4. RoleMenu (권한-메뉴 매핑)

권한과 메뉴 간의 접근 관계를 정의합니다.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| role_id | BIGINT | FK → Role(id), PK | 권한 참조 |
| menu_id | BIGINT | FK → Menu(id), PK | 메뉴 참조 |
| created_at | DATETIME | NOT NULL, DEFAULT NOW | 생성 일시 |

**Composite Primary Key:** (role_id, menu_id)

**Cascade Rules:**
- Role 삭제 시: RoleMenu 매핑도 삭제
- Menu 삭제 시: RoleMenu 매핑도 삭제

**Indexes:**
- `idx_role_menu_role_id`: role_id (권한별 메뉴 조회)
- `idx_role_menu_menu_id`: menu_id (메뉴별 권한 조회)

---

### 5. User (유저)

서비스에 가입한 사용자를 정의합니다.

| Field | Type | Constraints | Description |
|-------|------|-------------|-------------|
| id | BIGINT | PK, AUTO_INCREMENT | 고유 식별자 |
| email | VARCHAR(255) | NOT NULL, UNIQUE | 사용자 이메일 |
| name | VARCHAR(100) | NOT NULL | 사용자 이름 |
| status | ENUM | NOT NULL, DEFAULT 'ACTIVE' | 계정 상태 |
| created_at | DATETIME | NOT NULL, DEFAULT NOW | 가입 일시 |
| updated_at | DATETIME | NOT NULL, DEFAULT NOW ON UPDATE | 수정 일시 |

**UserStatus Enum:**
- `ACTIVE`: 정상 활성 상태
- `SUSPENDED`: 정지 상태

**Validation Rules:**
- email: 유효한 이메일 형식, 255자 이내
- name: 2-100자

**Indexes:**
- `idx_user_email`: email (이메일 검색)
- `idx_user_status`: status (상태별 필터링)
- `idx_user_created_at`: created_at (가입일 정렬)

---

## Database Schema (DDL)

```sql
-- V1__create_role_table.sql
CREATE TABLE role (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL UNIQUE,
    description VARCHAR(255),
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- V2__create_admin_table.sql
CREATE TABLE admin (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL,
    name VARCHAR(100) NOT NULL,
    role_id BIGINT NOT NULL,
    status ENUM('ACTIVE', 'LOCKED', 'INACTIVE') NOT NULL DEFAULT 'ACTIVE',
    login_fail_count INT NOT NULL DEFAULT 0,
    last_login_at DATETIME,
    deleted_at DATETIME,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_admin_role FOREIGN KEY (role_id) REFERENCES role(id)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE INDEX idx_admin_email ON admin(email);
CREATE INDEX idx_admin_status ON admin(status);
CREATE INDEX idx_admin_role_id ON admin(role_id);
CREATE INDEX idx_admin_deleted_at ON admin(deleted_at);

-- V3__create_menu_table.sql
CREATE TABLE menu (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    url VARCHAR(255) NOT NULL,
    sort_order INT NOT NULL DEFAULT 0,
    parent_id BIGINT,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    CONSTRAINT fk_menu_parent FOREIGN KEY (parent_id) REFERENCES menu(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE INDEX idx_menu_parent_id ON menu(parent_id);
CREATE INDEX idx_menu_sort_order ON menu(parent_id, sort_order);

-- V4__create_role_menu_table.sql
CREATE TABLE role_menu (
    role_id BIGINT NOT NULL,
    menu_id BIGINT NOT NULL,
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    PRIMARY KEY (role_id, menu_id),
    CONSTRAINT fk_role_menu_role FOREIGN KEY (role_id) REFERENCES role(id) ON DELETE CASCADE,
    CONSTRAINT fk_role_menu_menu FOREIGN KEY (menu_id) REFERENCES menu(id) ON DELETE CASCADE
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE INDEX idx_role_menu_role_id ON role_menu(role_id);
CREATE INDEX idx_role_menu_menu_id ON role_menu(menu_id);

-- V5__create_user_table.sql
CREATE TABLE user (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    name VARCHAR(100) NOT NULL,
    status ENUM('ACTIVE', 'SUSPENDED') NOT NULL DEFAULT 'ACTIVE',
    created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

CREATE INDEX idx_user_email ON user(email);
CREATE INDEX idx_user_status ON user(status);
CREATE INDEX idx_user_created_at ON user(created_at);

-- V6__insert_initial_data.sql
INSERT INTO role (name, description) VALUES
    ('SUPER', '시스템 최고 관리자. 모든 기능 접근 가능'),
    ('MANAGER', '관리자. SUBMANAGER/VIEWER 생성 가능, 메뉴/유저 관리'),
    ('SUBMANAGER', '부관리자. 유저 관리만 가능'),
    ('VIEWER', '조회 전용. 데이터 수정 불가');

-- 초기 SUPER 관리자 (password: Admin123!@#)
INSERT INTO admin (email, password, name, role_id, status) VALUES
    ('admin@specuser.com', '$2a$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/X4.VUO1T9FP6hHO5i', 'System Admin', 1, 'ACTIVE');
```

---

## Query Patterns

### 1. 관리자 로그인 조회
```sql
SELECT a.*, r.name as role_name
FROM admin a
JOIN role r ON a.role_id = r.id
WHERE a.email = ? AND a.status = 'ACTIVE'
```

### 2. 권한별 메뉴 트리 조회
```sql
SELECT m.*
FROM menu m
JOIN role_menu rm ON m.id = rm.menu_id
WHERE rm.role_id = ?
ORDER BY m.parent_id NULLS FIRST, m.sort_order
```

### 3. 유저 목록 페이지네이션
```sql
SELECT * FROM user
WHERE status = ?  -- optional filter
ORDER BY created_at DESC
LIMIT ? OFFSET ?
```

### 4. 90일 경과 비활성 관리자 조회 (배치용)
```sql
SELECT id FROM admin
WHERE status = 'INACTIVE'
AND deleted_at < DATE_SUB(NOW(), INTERVAL 90 DAY)
```
