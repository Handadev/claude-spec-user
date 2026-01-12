# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Spring Boot 4.0.1 RESTful web application for user management, using Java 21 with Gradle 9.2.1 build system.

## Essential Commands

### Build and Run
```bash
# Build the project
./gradlew build

# Run the application
./gradlew bootRun

# Clean and rebuild
./gradlew clean build
```

### Testing
```bash
# Run all tests
./gradlew test

# Run specific test class
./gradlew test --tests "org.specuser.*TestClassName"

# Run with test coverage
./gradlew check
```

### Development
```bash
# View dependencies
./gradlew dependencies

# Create executable JAR
./gradlew bootJar

# Build Docker image
./gradlew bootBuildImage
```

## Architecture

### Package Structure
- **Base Package:** `org.specuser`
- **Entry Point:** `org.specuser.SpecUserApplication`

### Technology Stack
- **Framework:** Spring Boot with Spring Data JPA, Spring Security, Spring Web MVC
- **Database:** MariaDB (localhost:3306/users)
- **Connection:** Configured in `src/main/resources/application.properties`
- **Key Libraries:** Lombok for code generation, HikariCP for connection pooling

### Spring Boot Conventions
The project follows standard Spring Boot patterns:
- `@SpringBootApplication` main class for application startup
- `application.properties` for configuration
- Dependency injection via constructor injection (preferred) or `@Autowired`
- JPA entities with `@Entity`, repositories extending `JpaRepository`
- Service layer with `@Service` for business logic
- REST controllers with `@RestController` and `@RequestMapping`

### Database Configuration
Database connection is configured in `application.properties`:
- URL: `jdbc:mariadb://localhost:3306/users`
- Driver: MariaDB JDBC driver
- JPA configured with Hibernate as the provider

## Development Notes

### Current State
This is a skeleton Spring Boot project with database configuration but no domain implementation. When implementing features:
1. Create entities in a `model` or `entity` package
2. Create repository interfaces extending `JpaRepository` in a `repository` package
3. Implement service layer in a `service` package
4. Create REST controllers in a `controller` package
5. Add appropriate tests for each layer

### Testing Approach
- Unit tests for services using Mockito
- Integration tests for repositories using `@DataJpaTest`
- Web layer tests using `@WebMvcTest` for controllers
- Full integration tests using `@SpringBootTest`

## Active Technologies
- Java 21 + Spring Boot 4.0.1, Spring Security, Spring Data JPA, Lombok, JJWT (001-admin-api)

## Recent Changes
- 001-admin-api: Added Java 21 + Spring Boot 4.0.1, Spring Security, Spring Data JPA, Lombok, JJWT
