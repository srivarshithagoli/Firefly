# 02 — Technology Stack

This document details every important technology and dependency used in `pom.xml` and the repository, explained for beginners.

---

## 1. Java 17
- **What it is**: A long-term support (LTS) release of the Java Development Kit (JDK).
- **Why it is used**: Provides modern language features like `record` (used in `MeshSimulatorService` and `BridgeIngestionService`), text blocks, pattern matching, and improved performance.
- **Where it appears**: Configured in `pom.xml` via `<java.version>17</java.version>`.
- **Key Concepts to Learn First**: Classes, methods, records, immutability, exception handling.

---

## 2. Spring Boot 3.3.5
- **What it is**: An opinionated framework built on top of Spring Framework that simplifies creating production-grade Java web applications with embedded servers and auto-configuration.
- **Why it is used**: Eliminates boilerplate XML configuration, embeds Tomcat so the app runs with a single command, and provides dependency injection out of the box.
- **Where it appears**: Specified as the parent in `pom.xml` (`spring-boot-starter-parent` v3.3.5) and initialized in `UpiMeshApplication.java`.
- **Key Concepts to Learn First**: `@SpringBootApplication`, `@Component`, `@Service`, `@Autowired`, Dependency Injection (IoC container).

---

## 3. Spring Web & Embedded Tomcat
- **What it is**: The module for building RESTful web applications and APIs using Spring MVC, running inside an embedded Apache Tomcat web server.
- **Why it is used**: Allows the server to listen on port `8080`, expose endpoints like `/api/bridge/ingest`, and serve JSON responses.
- **Where it appears**: Included via `spring-boot-starter-web` in `pom.xml`; controllers located in `com.demo.upimesh.controller` (`ApiController.java`, `DashboardController.java`).
- **KeyConcepts to Learn First**: `@RestController`, `@GetMapping`, `@PostMapping`, `@RequestBody`, `@RequestHeader`, `ResponseEntity`.

---

## 4. Thymeleaf
- **What it is**: A modern server-side Java template engine for HTML5.
- **Why it is used**: Renders the interactive dashboard UI (`dashboard.html`) directly from the server without requiring a separate single-page app (SPA) frontend build pipeline (like React or Vue).
- **Where it appears**: Included via `spring-boot-starter-thymeleaf` in `pom.xml`; template located at `src/main/resources/templates/dashboard.html`.
- **Key Concepts to Learn First**: HTML templates, server-side rendering, Thymeleaf namespace attributes (`th:`).

---

## 5. Spring Data JPA & Hibernate
- **What it is**: Spring Data JPA provides repository abstractions on top of JPA (Java Persistence API), while Hibernate is the underlying ORM (Object-Relational Mapping) framework.
- **Why it is used**: Maps Java classes (`Account.java`, `Transaction.java`) directly to SQL database tables without writing manual SQL queries for basic CRUD operations.
- **Where it appears**: Included via `spring-boot-starter-data-jpa` in `pom.xml`; repository interfaces (`AccountRepository.java`, `TransactionRepository.java`) in `com.demo.upimesh.model`.
- **Key Concepts to Learn First**: `@Entity`, `@Id`, `@Version`, JpaRepository interface methods (`save`, `findById`, `findAll`), `@Transactional`.

---

## 6. H2 Database
- **What it is**: A fast, lightweight relational database written in Java that can run entirely in-memory.
- **Why it is used**: Allows the application to run with zero external installation or database configuration (`spring.datasource.url=jdbc:h2:mem:upimesh` in `application.properties`). Also provides an in-browser admin console at `/h2-console`.
- **Where it appears**: Included via `com.h2database:h2` (scope: runtime) in `pom.xml` and configured in `application.properties`.
- **Key Concepts to Learn First**: In-memory databases, JDBC URLs, SQL tables, H2 console.

---

## 7. Spring Validation
- **What it is**: Integrates Jakarta Bean Validation into Spring Boot for validating object properties.
- **Why it is used**: Ensures incoming request payloads adhere to constraints (e.g., `@NotBlank`, `@Min`) before processing.
- **Where it appears**: Included via `spring-boot-starter-validation` in `pom.xml` and used on model fields in `MeshPacket.java`.
- **Key Concepts to Learn First**: Validation annotations (`@NotBlank`, `@NotNull`, `@Min`).

---

## 8. Jackson (Jackson Databind)
- **What it is**: Java library for parsing and generating JSON data.
- **Why it is used**: Automatically serializes Java objects to JSON responses and deserializes incoming HTTP JSON requests into Java objects (e.g., in `HybridCryptoService.java` and `ApiController.java`).
- **Where it appears**: Included via `jackson-databind` in `pom.xml` and used implicitly by Spring MVC and explicitly in `HybridCryptoService` (`ObjectMapper`).
- **Key Concepts to Learn First**: JSON serialization/deserialization, Object mapping.

---

## 9. Java Cryptography Architecture (JCA)
- **What it is**: Built-in Java security libraries (`javax.crypto`, `java.security`).
- **Why it is used**: Performs cryptographic operations including RSA-2048 keypair generation, RSA-OAEP encryption/decryption, AES-256-GCM authenticated encryption, and SHA-256 hashing.
- **Where it appears**: Implemented in `ServerKeyHolder.java` and `HybridCryptoService.java`.
- **Key Concepts to Learn First**: Public/private key cryptography, symmetric vs. asymmetric encryption, IV (Initialization Vector), authentication tags, cryptographic hashing (SHA-256).
