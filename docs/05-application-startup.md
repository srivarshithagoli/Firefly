# 05 — Application Startup & Initialization

This document explains what happens when the application starts up, tracing execution from the main entry point through Spring Boot initialization, database setup, and component configuration.

---

## Startup Sequence

When you run `mvnw.cmd spring-boot:run` (or `./mvnw spring-boot:run`), the following sequence occurs:

1. **Main Entry Point (`UpiMeshApplication.java`)**:
   - The `main` method invokes `SpringApplication.run(UpiMeshApplication.class, args);`.
   - Spring Boot initializes the application context, scanning the package `com.demo.upimesh` for annotated beans (`@Component`, `@Service`, `@RestController`, `@Configuration`).

2. **Embedded Server & Web Container Initialization**:
   - Spring Boot auto-configures and starts an embedded Apache Tomcat web server listening on port `8080` (configured in `application.properties`).

3. **Database Initialization & DDL Auto (`application.properties`)**:
   - Configured with `spring.datasource.url=jdbc:h2:mem:upimesh`.
   - `spring.jpa.hibernate.ddl-auto=create-drop` instructs Hibernate to automatically create the database tables (`accounts`, `transactions`) when the application starts, and drop them when it shuts down.
   - The H2 web console is enabled at `/h2-console`.

4. **Cryptographic Key Generation (`ServerKeyHolder.java`)**:
   - Spring instantiates `ServerKeyHolder` as a Spring `@Component`.
   - The `@PostConstruct` annotated `init()` method executes automatically:
     ```java
     KeyPairGenerator gen = KeyPairGenerator.getInstance("RSA");
     gen.initialize(2048);
     this.keyPair = gen.generateKeyPair();
     ```
   - A fresh RSA 2048-bit public/private keypair is generated in memory. The public key fingerprint is logged to the console.

5. **Demo Data Seeding (`DemoService.java`)**:
   - Spring instantiates `DemoService`.
   - Its `@PostConstruct` method `seedAccounts()` checks if the database is empty (`accounts.count() == 0`). If so, it seeds four demo accounts (`alice@demo`, `bob@demo`, `carol@demo`, `dave@demo`) into the H2 database.

6. **Scheduled Tasks Configuration (`AppConfig.java`)**:
   - `@EnableScheduling` activates Spring's scheduling infrastructure, enabling background tasks such as `IdempotencyService.evictExpired()` which runs every 60 seconds (`@Scheduled(fixedDelay = 60_000)`).

7. **Ready State**:
   - Spring Boot logs `Started UpiMeshApplication in X.XXX seconds`. The server is ready to accept HTTP requests at `http://localhost:8080`.
