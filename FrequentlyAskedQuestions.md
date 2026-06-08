# ITSM-Pro — 100 Detailed FAQs: Technology Choices, Versions & Decisions

> Every technology decision explained: why it was chosen, what alternatives existed,
> what advantages it brings, and what its drawbacks are.
> Targeted at developers, architects, and IT managers evaluating or extending ITSM-Pro.

---

## SECTION 1: Java & Backend Language (Q1–Q10)

---

### Q1. Why was Java 21 LTS chosen? What other versions were available?

**Chosen:** Java 21 LTS (Long-Term Support)

**Other options available:**
| Version | Type | Support Until |
|---------|------|--------------|
| Java 11 LTS | Older LTS | September 2026 |
| Java 17 LTS | LTS | September 2029 |
| **Java 21 LTS** | **Current LTS** | **September 2031** ✅ |
| Java 22/23 | Short-term | 6 months only |

**Why Java 21 was selected:**
Java 21 is the most recent Long-Term Support release as of 2024. LTS versions receive security patches and bug fixes for years, making them suitable for enterprise production systems. Using a non-LTS version (like Java 22 or 23) means the version stops receiving updates after just 6 months.

**Advantages of Java 21:**
- **Virtual Threads (Project Loom)** — Lightweight threads that scale to millions of concurrent tasks without needing async code. Your `@Async` email threads can be massively scaled.
- **Pattern Matching for switch** — Cleaner code for multi-case type checking in service logic.
- **Record types** — Immutable data carriers perfect for DTOs (available since Java 16, matured in 21).
- **Text blocks** — Multi-line strings used in JPQL queries: `""" SELECT i FROM Incident i WHERE ... """`
- **Sequenced Collections** — `SequencedCollection` interface for ordered access to first/last elements.
- **Spring Boot 3.x requires Java 17 minimum** — Java 21 is fully compatible and recommended.
- **Support until 2031** — Enterprises can deploy today and not worry about Java upgrades for 7 years.

**Drawbacks of Java 21:**
- Slightly higher memory footprint than Java 11/17 out of the box (mitigated by container flags).
- Some older third-party libraries may have compatibility issues with newer Java versions.
- Virtual threads (preview features) require explicit enabling in earlier builds.
- Team members familiar only with Java 8/11 have a learning curve for newer language features.

**Why NOT Java 17?**
Java 17 was a valid choice (Spring Boot 3 minimum), but Java 21 brings Virtual Threads which significantly improves throughput for I/O-heavy applications like ITSM-Pro (HTTP requests, database calls, email sending) without adding async complexity.

**Why NOT Java 11?**
Spring Boot 3.x dropped support for Java 11. You would be stuck on Spring Boot 2.x which is now end-of-life (November 2024).

---

### Q2. Why Spring Boot 3.3.4 specifically? What about Spring Boot 2.x?

**Chosen:** Spring Boot 3.3.4

**Available versions at time of development:**
| Version | Java Requirement | Status |
|---------|----------------|--------|
| Spring Boot 2.7.x | Java 8+ | End of Life (Nov 2024) ❌ |
| Spring Boot 3.0.x | Java 17+ | Older 3.x — not recommended |
| Spring Boot 3.2.x | Java 17+ | Previous supported |
| **Spring Boot 3.3.4** | **Java 17+** | **Current stable** ✅ |
| Spring Boot 3.4.x | Java 17+ | Newer, less battle-tested |

**Why 3.3.4 was selected:**

**Advantages of Spring Boot 3.3.4:**
- **Jakarta EE 10 migration** — Spring Boot 3.x uses `jakarta.*` packages instead of deprecated `javax.*`. Future-proof.
- **Virtual Thread support** — Native integration with Java 21 virtual threads via `spring.threads.virtual.enabled=true`.
- **Improved Actuator** — Better health check endpoints (`/liveness`, `/readiness`) for Kubernetes probes.
- **Native compilation** — Spring Boot 3 supports GraalVM native image compilation (smaller, faster-starting images).
- **AOT Processing** — Ahead-of-time compilation hints for faster startup times.
- **Better observability** — Micrometer Tracing built in for distributed tracing.
- **Security by default** — Spring Security 6.x defaults are more secure than 5.x.
- **Active development** — 3.3.x is actively maintained with regular security patches.
- **.3.4 patch release** — Bug fixes since 3.3.0 without breaking API changes.

**Drawbacks of Spring Boot 3.3.4:**
- **Jakarta EE migration** — `javax.*` → `jakarta.*` broke many older libraries. Some third-party integrations still haven't migrated.
- **Minimum Java 17** — Cannot run on Java 8/11 environments (common in legacy enterprises).
- **@SpringBootTest changes** — Some testing patterns changed; teams upgrading from 2.x need test rewrites.
- **WebMvcConfigurer changes** — Some configuration methods were deprecated and replaced.

**Why NOT Spring Boot 2.7.x?**
Spring Boot 2.7.x reached End of Life in November 2024. No security patches means production deployments become vulnerable over time. Starting any new enterprise project on EOL software is a risk.

---

### Q3. Why was Spring Security chosen for authentication? What were the alternatives?

**Chosen:** Spring Security 6.3.x (bundled with Spring Boot 3.3.4)

**Alternatives considered:**
| Alternative | Type | Best For |
|------------|------|---------|
| **Spring Security** | Framework | Enterprise Java ✅ |
| Apache Shiro | Framework | Simpler apps, not Spring-native |
| Keycloak | Identity Server | Federated auth, multi-app SSO |
| Auth0 | SaaS | Outsourced auth, paid service |
| Okta | SaaS | Enterprise SSO, paid service |
| Custom JWT filter | DIY | Full control, high risk |

**Why Spring Security was selected:**

**Advantages:**
- **Native Spring integration** — Zero additional wiring needed. `@PreAuthorize`, `SecurityContextHolder`, `AuthenticationManager` all work out of the box.
- **Battle-tested** — Powers thousands of enterprise applications. Security vulnerabilities are found and patched rapidly.
- **BCrypt built in** — `BCryptPasswordEncoder` is part of the library, no extra dependency.
- **Method security** — `@PreAuthorize("hasRole('ADMIN')")` on individual methods, not just URL patterns.
- **Filter chain** — Pluggable filter chain means adding JWT validation is a single `addFilterBefore()` call.
- **CSRF, CORS, CSP** — Built-in support for web security headers with sensible defaults.
- **Free** — No licensing cost unlike Auth0/Okta for high user volumes.

**Drawbacks of Spring Security:**
- **Steep learning curve** — The filter chain, authentication providers, and security context can confuse beginners.
- **Configuration verbosity** — Even with lambda DSL in Spring Boot 3.x, complex rules require careful configuration.
- **Over-engineered for small apps** — A simple CRUD app may not need all this complexity.
- **Breaking changes in 6.x** — `antMatchers` removed, replaced with `requestMatchers`. Teams upgrading must rewrite security config.

**Why NOT Keycloak?**
Keycloak would be excellent for multi-application SSO (sign-in once, access 10 systems). For a standalone ITSM system, Keycloak adds operational overhead: another service to deploy, configure, and maintain. Spring Security handles our simpler single-application auth perfectly.

---

### Q4. Why BCrypt with strength 12? Why not Argon2 or PBKDF2?

**Chosen:** `BCryptPasswordEncoder(12)` from Spring Security

**Alternatives:**
| Algorithm | Speed (per hash) | Memory Hard? | Best For |
|-----------|-----------------|-------------|---------|
| MD5 | < 1ms | No | ❌ Never use for passwords |
| SHA-256 | < 1ms | No | ❌ Too fast, easily brute-forced |
| PBKDF2 | ~50ms | No | Legacy systems |
| **BCrypt (12)** | **~250ms** | **No** | **Most apps** ✅ |
| Argon2id | ~100-500ms | Yes | High-security systems |
| scrypt | ~100-500ms | Yes | Cryptocurrency wallets |

**Why BCrypt strength 12 was selected:**

**Advantages of BCrypt(12):**
- **Spring Security native** — `BCryptPasswordEncoder` is the default Spring Security password encoder. Zero additional dependency.
- **Automatic salt** — BCrypt generates and stores a unique random salt per hash. Two identical passwords produce different hashes.
- **Deliberate slowness** — 2^12 = 4,096 iterations. Checking one password takes ~250ms. Brute-forcing 1 million candidates takes 70+ hours at this speed.
- **Self-describing format** — The `$2a$12$...` format embeds the algorithm version and cost factor. You can upgrade cost later without breaking existing hashes.
- **Industry standard** — Used by GitHub, Dropbox, LastPass, and most major services.
- **Adaptive** — Can increase from 12 to 14 later when hardware gets faster, without changing existing passwords.
- **Good UX balance** — 250ms is imperceptible to users (login still feels instant) but expensive for attackers.

**Drawbacks of BCrypt(12):**
- **Not memory-hard** — BCrypt does NOT require large RAM during computation. Specialized ASICs can crack it faster than Argon2id. For extremely high-security applications (banking passwords), Argon2id is superior.
- **72-byte limit** — BCrypt only processes the first 72 bytes of a password. Passwords longer than 72 chars are truncated. In practice, no user types 72-character passwords.
- **CPU-bound** — Under login storms (DDoS), BCrypt verification can consume significant CPU. Mitigate with rate limiting.

**Why NOT Argon2id?**
Argon2id is technically superior (memory-hard, won the Password Hashing Competition in 2015). However, Spring Security requires `spring-security-crypto` extras, and for enterprise ITSM where password security is adequate with BCrypt(12), the additional complexity is not justified. Argon2 is the better choice for a cryptocurrency exchange or medical records system.

**Why NOT strength 10 or 14?**
- Strength 10 (~100ms): slightly too fast; recent GPU hardware can crack common passwords.
- Strength 14 (~1,000ms): noticeably slow for login, impacts UX, and unnecessary for ITSM.
- Strength 12 (~250ms): the sweet spot for 2024 hardware.

---

### Q5. Why use JWT for authentication instead of session-based auth?

**Chosen:** Stateless JWT (JSON Web Tokens)

**Alternative:** Stateful server-side sessions (traditional)

| Property | JWT (Stateless) | Sessions (Stateful) |
|----------|----------------|-------------------|
| Server memory | None needed | Stores session per user |
| Horizontal scaling | ✅ Easy — any server validates | ❌ Needs sticky sessions or shared Redis |
| Revocation | ❌ Difficult (need blocklist) | ✅ Easy (delete session) |
| Payload size | Slightly larger (Base64 token) | Small (just a session ID) |
| Database calls | One per request (user lookup) | One per request (session lookup) |
| Mobile clients | ✅ Works natively | ❌ Needs cookie management |

**Why JWT was selected:**

**Advantages:**
- **Stateless scalability** — Any backend instance can validate a JWT independently by verifying the HMAC signature. You can run 10 backend pods in Kubernetes with zero session synchronization.
- **No shared session store** — Traditional sessions require Redis or a database for sharing state between instances. JWT eliminates this dependency.
- **Self-contained** — The token carries username and role. The backend doesn't need a database lookup to know who the user is (though we do reload for `isActive` checks).
- **Mobile/SPA friendly** — Single-Page Applications and mobile apps work naturally with tokens in `Authorization` headers. Cookie-based sessions have SameSite and CORS complications.
- **Refresh token pattern** — Short-lived access tokens (15 min) minimize damage from stolen tokens. Refresh tokens (7 days) allow seamless UX without re-login.

**Drawbacks of JWT:**
- **Cannot revoke access tokens** — If an access token is stolen, it remains valid until it expires (15 minutes in our case). Sessions can be invalidated instantly.
- **Logout is client-side** — The server doesn't truly "log out" a user. The client just deletes the token.
- **Payload size** — JWTs are ~200-500 bytes vs a simple 32-byte session ID. Minor but measurable bandwidth overhead.
- **Secret key management** — If `JWT_SECRET` is compromised, all tokens become forgeable. Requires secure secret rotation procedures.

**Mitigation for revocation:** A Redis token blocklist can be added — store the `jti` (JWT ID) claim of revoked tokens with TTL = token remaining lifetime. The JWT filter checks the blocklist on each request.

---

### Q6. Why JJWT 0.12.6? What other JWT libraries exist for Java?

**Chosen:** JJWT (Java JWT) version 0.12.6

**Java JWT libraries compared:**
| Library | Version | Maintainer | Stars | Pros |
|---------|---------|-----------|-------|------|
| **JJWT** | **0.12.6** | **Stormpath/okta** | 11k+ | **Most popular, fluent API** ✅ |
| Nimbus JOSE JWT | 9.x | Connect2id | 3k | Standards-compliant, JWE support |
| Auth0 java-jwt | 4.x | Auth0 | 5k | Simple API, good docs |
| Spring Security OAuth | Built-in | Spring | N/A | Complex, overkill for simple JWT |

**Why JJWT 0.12.6 was selected:**

**Advantages:**
- **Most popular Java JWT library** — 11,000+ GitHub stars, extensive community support.
- **Fluent builder API** — `Jwts.builder().subject(username).expiration(date).signWith(key).compact()` reads like English.
- **Three-jar system** — `jjwt-api` (compile), `jjwt-impl` (runtime), `jjwt-jackson` (JSON). Only the API is needed at compile time, implementation swappable.
- **Actively maintained** — Regular security updates, Java 21 compatible.
- **0.12.x API** — Major API cleanup vs 0.11.x. `Jwts.parser().verifyWith(key)` is cleaner than the older deprecated methods.
- **Algorithm safety** — Refuses to verify `alg: none` tokens (a common JWT vulnerability in older libraries).

**Drawbacks of JJWT 0.12.6:**
- **Breaking API changes from 0.11.x** — Code written for JJWT 0.11.x requires rewriting for 0.12.x. The builder and parser APIs changed significantly.
- **No JWE support** — JJWT focuses on JWS (signed) tokens, not JWE (encrypted) tokens. Nimbus handles both.
- **Three artifacts** — Slightly more Maven config than single-jar alternatives.

**Why NOT Nimbus JOSE JWT?**
Nimbus is more standards-complete (JWE, JWK, OIDC support) and is used by Spring Security OAuth internally. However, for a simple signed JWT use case (which is all ITSM-Pro needs), Nimbus is over-engineered. JJWT's API is more readable.

---

### Q7. Why Maven 3.9.6 over Gradle? Was Gradle considered?

**Chosen:** Apache Maven 3.9.6

**Build tools compared:**
| Tool | Config Language | Spring Boot Support | Learning Curve |
|------|----------------|--------------------|----|
| **Maven** | XML (pom.xml) | First-class | Low |
| Gradle Groovy | Groovy DSL | First-class | Medium |
| Gradle Kotlin DSL | Kotlin | First-class | High |
| Ant | XML + Java | Manual | High |
| Bazel | Starlark | Manual | Very High |

**Why Maven was selected:**

**Advantages of Maven:**
- **Enterprise standard** — The vast majority of enterprise Java projects use Maven. New team members already know it.
- **Explicit pom.xml** — XML is verbose but unambiguous. Every dependency and plugin configuration is visible.
- **Spring Initializr default** — https://start.spring.io generates Maven projects by default, making documentation and tutorials align with your project.
- **Maven BOM (Bill of Materials)** — Spring Boot's parent POM manages all dependency versions via BOM. `spring-boot-starter-parent` sets Hibernate, Jackson, Tomcat, Flyway versions automatically — eliminates version conflicts.
- **Mature ecosystem** — Maven plugins for every task: Surefire (tests), Flyway, Docker, etc.
- **Reproducible builds** — `mvn clean package` produces identical JARs given identical inputs. Gradle has more footguns around cache invalidation.
- **Maven Wrapper (mvnw)** — The wrapper script downloads Maven automatically, so CI/CD environments don't need Maven pre-installed.

**Drawbacks of Maven:**
- **Verbose XML** — `pom.xml` files become very long for complex projects. Gradle's DSL is more concise.
- **Slower than Gradle** — Gradle has more sophisticated incremental build caching and parallel execution. Large multi-module Maven builds can be slower.
- **Inflexible** — Custom build logic requires writing Maven plugins or ugly `exec-maven-plugin` hacks. Gradle scripts can do anything.
- **No build cache server** — Gradle Enterprise offers shared remote build caches. Maven's equivalent (Develocity) is commercial.

**Why NOT Gradle?**
Gradle is technically more powerful, but for a team starting a new ITSM project, Maven's predictability and lower learning curve outweigh Gradle's speed advantages. At the scale of this project (one JAR, ~25 Java files), build time differences are seconds, not minutes.

---

### Q8. Why use Lombok? Is it really necessary?

**Chosen:** Lombok 1.18.34

**Why Lombok is used in ITSM-Pro:**
Every JPA entity needs getters, setters, equals, hashCode, toString, and often a builder. For the `Incident` entity alone, that's ~150 lines of boilerplate. Lombok generates all of it from four annotations.

**Without Lombok:**
```java
public class Incident {
    private Long id;
    private String title;
    // ... 20+ fields
    
    // 20+ getters
    public Long getId() { return id; }
    public String getTitle() { return title; }
    
    // 20+ setters
    public void setId(Long id) { this.id = id; }
    
    // equals() - 50 lines
    // hashCode() - 30 lines
    // toString() - 20 lines
    // Builder class - 80 lines
    // TOTAL: ~400 lines of noise
}
```

**With Lombok:**
```java
@Data        // getters + setters + equals + hashCode + toString
@Builder     // builder pattern
@NoArgsConstructor
@AllArgsConstructor
public class Incident {
    private Long id;
    private String title;
    // ... 20+ fields
    // TOTAL: ~30 lines of signal
}
```

**Advantages of Lombok:**
- **Eliminates ~80% of boilerplate** — Entities, DTOs, services with `@Slf4j` logger all become dramatically shorter.
- **`@Slf4j`** — Generates `private static final Logger log = LoggerFactory.getLogger(...)` automatically. Used extensively for debugging.
- **`@Builder`** — Fluent builder pattern for creating entities: `Incident.builder().title("...").priority(P1).build()`
- **`@RequiredArgsConstructor`** — Generates constructors for all `final` fields. Used with Spring's constructor injection (replaces `@Autowired`).
- **Compile-time** — Lombok runs during compilation. Zero runtime overhead — the generated code is plain Java bytecode.

**Drawbacks of Lombok:**
- **IDE dependency** — IntelliJ IDEA requires the Lombok plugin. Without it, the IDE shows false errors. Can confuse developers new to the project.
- **Annotation processor ordering** — Lombok MUST run before MapStruct in the annotation processor chain (hence the explicit ordering in `pom.xml`).
- **Hidden complexity** — `@Data` on JPA entities is controversial — it generates `equals()` based on ALL fields, which can cause issues with Hibernate's lazy loading and circular references. Best practice: use `@Getter @Setter` on entities instead of `@Data`.
- **Debugging** — Stack traces go through generated code, which can be harder to read.
- **Team dependency** — New developers must understand which methods are generated vs written.

**Why NOT use records (Java 21)?**
Java records are immutable. JPA entities MUST be mutable (Hibernate modifies them via setters). Records also cannot have no-args constructors easily, which JPA requires. Lombok's `@Data` is the correct choice for JPA entities. Records are excellent for DTOs (response objects) where immutability is a feature.

---

### Q9. Why MapStruct over other mapping approaches?

**Chosen:** MapStruct 1.5.5.Final

**Mapping approaches compared:**
| Approach | Speed | Type Safety | Boilerplate | Notes |
|----------|-------|-------------|-------------|-------|
| **MapStruct** | **Compile-time** | **✅ Full** | **✅ Minimal** | **Code generation** ✅ |
| ModelMapper | Runtime reflection | Partial | ✅ Minimal | Slow, runtime errors |
| Dozer | Runtime reflection | Partial | ✅ Minimal | Largely unmaintained |
| Manual mapping | N/A | ✅ Full | ❌ High | Hundreds of lines |
| Jackson ObjectMapper | Runtime | Partial | ✅ Minimal | Only for same-shape objects |

**Why MapStruct was selected:**

**Advantages:**
- **Compile-time generation** — MapStruct reads `@Mapper` interfaces and generates `IncidentMapperImpl.java` at build time. No reflection at runtime.
- **10-100x faster than ModelMapper** — Generated code is equivalent to hand-written assignments: `dto.setTitle(entity.getTitle())`. Benchmark: MapStruct maps 1M objects in ~50ms, ModelMapper takes ~5,000ms.
- **Type safety** — Compilation fails if a source field doesn't exist. ModelMapper silently ignores unknown fields at runtime, causing hard-to-debug bugs.
- **Clear errors** — If a field mapping is ambiguous or missing, MapStruct reports it at compile time with a specific error message.
- **Spring integration** — `componentModel = "spring"` generates `@Component` mappers that are injectable via `@Autowired`.
- **`@AfterMapping`** — Allows computed fields after mapping (e.g., calculating `slaMinutesRemaining` after the main entity-to-DTO mapping).

**Drawbacks of MapStruct:**
- **Build complexity** — Annotation processor ordering with Lombok is critical. If Lombok runs after MapStruct, generated getters/setters aren't visible to MapStruct.
- **More verbose setup** — Requires `@Mapper` interface and `pom.xml` annotation processor configuration.
- **Generated code in version control** — Some teams commit the generated `*MapperImpl.java` files; others don't. This can cause confusion.
- **Learning curve** — `@Mapping`, `@BeanMapping`, `@AfterMapping`, `@Named` annotations require documentation reading.

**Why NOT ModelMapper?**
ModelMapper uses Java reflection to map fields at runtime. While easier to start with, it:
1. Cannot catch mapping errors at compile time (runtime NullPointerExceptions instead).
2. Is significantly slower (10-100x) for large object graphs.
3. Has surprising behaviour with complex type hierarchies.
In a production system mapping thousands of incidents per minute, MapStruct's performance advantage is significant.

---

### Q10. Why constructor injection over field injection in Spring services?

**Pattern used:** Constructor injection via Lombok `@RequiredArgsConstructor`

```java
// ✅ ITSM-Pro uses this:
@Service
@RequiredArgsConstructor
public class IncidentService {
    private final IncidentRepository incidentRepository;  // final!
    private final UserRepository userRepository;
    // ...
}

// ❌ Field injection (common but problematic):
@Service
public class IncidentService {
    @Autowired
    private IncidentRepository incidentRepository;  // not final
}
```

**Why constructor injection was selected:**

**Advantages:**
- **Immutability** — `final` fields cannot be reassigned. Accidental reassignment of a service dependency is impossible.
- **Testability** — Constructor injection allows creating service instances without Spring context in unit tests: `new IncidentService(mockRepo, mockMapper, ...)`. Field injection requires `@InjectMocks` + reflection.
- **Null safety** — If a dependency is missing, Spring throws an error at startup. Field injection can leave fields null until first use.
- **Spring recommended** — The Spring team explicitly recommends constructor injection over field injection in their official documentation.
- **`@RequiredArgsConstructor`** — Lombok generates the constructor automatically from all `final` fields. No boilerplate needed.
- **Dependency visibility** — Looking at the constructor tells you exactly what a class depends on. Field injection hides dependencies.

**Drawbacks:**
- **Circular dependencies** — Constructor injection makes circular dependencies fail at startup (actually a feature — it forces you to fix the design). Field injection allows circular dependencies to work (a smell).
- **Long constructors** — Services with 8+ dependencies have verbose constructors. Solution: refactor the class into smaller units.

---

## SECTION 2: Spring Data & Database (Q11–Q20)

---

### Q11. Why SQL Server 2022 over PostgreSQL, MySQL, or Oracle?

**Chosen:** Microsoft SQL Server 2022 (Developer Edition for dev, Standard/Enterprise for prod)

**Databases compared:**
| Database | License | Windows | Linux | JSON | Full-text | Enterprise Feature |
|----------|---------|---------|-------|------|-----------|-------------------|
| **SQL Server 2022** | Commercial | ✅ | ✅ | ✅ | ✅ | **Most enterprises** ✅ |
| PostgreSQL 16 | Free | ✅ | ✅ | ✅ | ✅ | Growing enterprise adoption |
| MySQL 8.0 | Free/Commercial | ✅ | ✅ | ✅ | Limited | Web apps |
| Oracle 21c | Expensive | ✅ | ✅ | ✅ | ✅ | Legacy enterprise |
| MariaDB 11 | Free | ✅ | ✅ | Limited | Limited | MySQL alternative |

**Why SQL Server 2022 was selected:**

**Advantages:**
- **Enterprise ITSM market** — Most enterprises that need ITSM software (banks, government, healthcare) already run SQL Server as their standard database platform. ITSM-Pro fits naturally into this ecosystem.
- **SQL Server Management Studio (SSMS)** — Best-in-class GUI for database administration. DBAs already trained on it.
- **Microsoft ecosystem** — Azure SQL Database is direct SQL Server cloud hosting. Easy lift-and-shift to Azure when needed.
- **IDENTITY columns** — Auto-increment primary keys (`IDENTITY(1,1)`) behave predictably, unlike MySQL's edge cases.
- **DATETIME2** — High-precision timestamps (100-nanosecond accuracy) for audit logs and SLA calculations.
- **UPDATE...OUTPUT clause** — Used in `TicketSequenceService` for atomic ticket numbering. Equivalent in PostgreSQL is `UPDATE...RETURNING` but SQL Server's version works cleanly with JDBC.
- **Row-level locking** — `WITH (ROWLOCK, UPDLOCK)` hints give explicit concurrency control for the ticket sequence table.
- **Developer Edition is free** — Full-featured, free for development and testing. Only licensing cost is production.
- **SQL Server 2022 new features** — Azure Synapse Link, ledger tables for immutable audit trails, JSON performance improvements.

**Drawbacks of SQL Server:**
- **Cost** — SQL Server Standard is ~$1,000/core/year. Enterprise is ~$7,000/core/year. PostgreSQL is free.
- **Windows dependency historically** — SQL Server on Linux (since 2017) works well, but the tooling (SSMS) is still primarily Windows.
- **Vendor lock-in** — Some SQL Server-specific syntax (OUTPUT clause, top-N queries) doesn't port to PostgreSQL without rewriting.
- **Resource hungry** — Minimum 2GB RAM for SQL Server. PostgreSQL runs on 256MB.

**Why NOT PostgreSQL?**
PostgreSQL is technically superior in many ways (better standards compliance, JSONB, window functions, better open-source ecosystem). For a pure green-field project with no enterprise constraints, PostgreSQL would be an excellent choice. SQL Server was chosen because ITSM-Pro targets enterprises that typically already license SQL Server.

**Switching to PostgreSQL:** Change the `pom.xml` JDBC driver from `mssql-jdbc` to `postgresql`, update Flyway to `flyway-database-postgresql`, and adjust dialect in `application.properties`. The JPQL queries are database-agnostic; only native SQL in repositories needs review.

---

### Q12. Why Flyway over Liquibase for database migrations?

**Chosen:** Flyway 10.x (bundled with Spring Boot 3.3.4)

**Migration tools compared:**
| Tool | Config Format | SQL-first | XML/YAML | Learning Curve |
|------|-------------|-----------|----------|----------------|
| **Flyway** | SQL + Java | **✅ Primary** | Optional | **Low** ✅ |
| Liquibase | XML/YAML/SQL | Optional | **Primary** | Medium |
| Django migrations | Python | Generated | N/A | Low (Python-specific) |
| Hibernate DDL | Properties | Generated | N/A | Very Low (dangerous!) |

**Why Flyway was selected:**

**Advantages:**
- **SQL-first** — Migration files are plain SQL (`V1__Initial_Schema.sql`). Any DBA can read, modify, and execute them. Liquibase's XML format is harder for DBAs unfamiliar with it.
- **Simple versioning** — `V1__`, `V2__`, `V3__` naming makes execution order obvious. No complex dependency graphs.
- **Spring Boot auto-configuration** — `spring.flyway.enabled=true` in `application.properties` — Flyway runs automatically on startup. Zero explicit configuration needed.
- **SQL Server support** — `flyway-sqlserver` artifact provides SQL Server-specific dialect support.
- **Checksum validation** — Flyway stores checksums of applied scripts. If a script is modified post-deployment, startup fails immediately rather than silently applying wrong changes.
- **Baseline on migrate** — `spring.flyway.baseline-on-migrate=true` allows Flyway to work with pre-existing databases (useful when introducing Flyway to an existing schema).
- **Wide enterprise adoption** — Trusted by thousands of production systems.

**Drawbacks of Flyway:**
- **Irreversible by default** — Flyway's paid tier has undo migrations. The free version doesn't. Rolling back a migration requires writing a new `V4__Undo_Something.sql`.
- **No automatic rollback** — If a migration fails halfway, the database may be in an inconsistent state. Most SQL DDL is not transactional on SQL Server.
- **Checksum brittleness** — A trailing space, BOM character, or CRLF line ending change will break the checksum and prevent startup. Developers must never edit applied migrations.

**Why NOT Liquibase?**
Liquibase's XML changeset format is more feature-rich (rollback support, tagged deployments, diff generation). However, for most teams, Flyway's simplicity wins. Plain SQL files that any database person can read are more maintainable than XML describing SQL operations.

---

### Q13. Why Spring Data JPA instead of raw JDBC or an alternative ORM?

**Chosen:** Spring Data JPA 3.3.4 (with Hibernate 6.5.x)

**Data access options:**
| Approach | Abstraction | Type Safety | Boilerplate | SQL Control |
|----------|-------------|-------------|-------------|-------------|
| **Spring Data JPA** | High | Partial | **Very Low** | JPQL + @Query |
| Plain JDBC | None | Low | High | Full |
| JdbcTemplate | Low | Low | Medium | Full |
| jOOQ | Code-gen | **Very High** | Low | Full (fluent) |
| JDBI | Low | Medium | Low | SQL strings |
| MyBatis | Medium | Medium | Medium | Full XML |

**Why Spring Data JPA was selected:**

**Advantages:**
- **Zero-boilerplate repositories** — `interface IncidentRepository extends JpaRepository<Incident, Long>` gives `save()`, `findById()`, `findAll()`, `delete()`, `count()` automatically. No implementation needed.
- **Query derivation** — `findByStatusAndPriority(IncidentStatus status, Priority priority)` generates SQL automatically from the method name.
- **Pagination built-in** — `findAll(Pageable pageable)` returns `Page<T>` with `totalElements`, `totalPages`, `content`. Every list API in ITSM-Pro uses this.
- **Audit columns** — `@CreationTimestamp`, `@UpdateTimestamp` automatically populate `created_at` and `updated_at` without explicit code.
- **Relationships** — `@ManyToOne`, `@OneToMany`, `@ManyToMany` map complex relationships without writing JOIN SQL.
- **Integration** — Works directly with `@Transactional` for transaction management.

**Drawbacks of Spring Data JPA:**
- **N+1 query problem** — Fetching a list of incidents, then accessing `incident.getAssignedTo()` for each, triggers N additional queries. Must use `JOIN FETCH` or `EAGER` fetching strategically.
- **JPQL limitations** — Complex aggregations, window functions, or database-specific features require native SQL `@Query(nativeQuery=true)`.
- **Hidden complexity** — `session.flush()`, dirty checking, and `LazyInitializationException` can surprise developers not deeply familiar with JPA.
- **Performance for bulk operations** — Hibernate processes entities one at a time by default. Bulk inserts require `@Modifying @Query` or `JdbcTemplate` for best performance.

---

### Q14. Why HikariCP for connection pooling? What are the settings?

**Chosen:** HikariCP (bundled with Spring Boot)

**Connection pool options:**
| Pool | Bundled with Spring Boot | Speed | Enterprise Use |
|------|------------------------|-------|---------------|
| **HikariCP** | ✅ Default since Boot 2.0 | **Fastest** | ✅ |
| Apache DBCP2 | ❌ Manual add | Slower | Legacy |
| c3p0 | ❌ Manual add | Older | Legacy |
| Tomcat JDBC | ❌ Manual add | Medium | Replaced by HikariCP |

**Why HikariCP was selected:**
HikariCP is the Spring Boot default connection pool since version 2.0, and for good reason. Benchmarks consistently show HikariCP as 2-3x faster than competitors for connection acquisition under high concurrency.

**ITSM-Pro HikariCP configuration:**
```properties
spring.datasource.hikari.maximum-pool-size=20      # max concurrent DB connections
spring.datasource.hikari.minimum-idle=5             # always keep 5 connections ready
spring.datasource.hikari.idle-timeout=300000        # close idle connections after 5 min
spring.datasource.hikari.connection-timeout=20000   # fail after 20s if no connection available
spring.datasource.hikari.max-lifetime=1800000       # replace connections every 30 min
```

**Why maximum-pool-size=20?**
SQL Server can handle 32,767 concurrent connections, but keeping 20 per application instance is the sweet spot. Each connection consumes ~5MB of SQL Server memory. 20 connections × 5MB = 100MB, manageable for a typical server.

**Advantages of HikariCP:**
- **Zero-overhead design** — Uses `AtomicInteger` and custom collections instead of synchronization, minimizing CPU overhead.
- **Connection validation** — Validates connections before lending them. Stale connections from SQL Server restarts are detected and replaced.
- **Leak detection** — `leakDetectionThreshold` setting reports connections held longer than expected (catching unreturned connections in code bugs).

**Drawbacks of HikariCP:**
- **Max connections limit** — If all 20 connections are in use and a 21st request arrives, it waits up to `connection-timeout` milliseconds. Under extreme load, requests queue up.
- **Connection size** — `maximum-pool-size` needs tuning per environment. Too small = timeouts, too large = database resource exhaustion.

---

### Q15. Why is `spring.jpa.hibernate.ddl-auto=validate` used instead of `create-drop`?

**DDL auto options:**
| Setting | Behaviour | Use Case |
|---------|-----------|----------|
| `none` | Does nothing | Production (Flyway manages schema) |
| `validate` | Checks entities match DB schema, fails if not | **ITSM-Pro production** ✅ |
| `update` | Adds missing columns/tables | ❌ Never production — can't DROP |
| `create` | Drops and recreates everything on startup | ❌ Loses all data |
| `create-drop` | Creates on start, drops on shutdown | ❌ Test only |

**Why `validate` was selected:**

**Advantages:**
- **Safety net** — If a developer adds a field to a JPA entity but forgets to write a Flyway migration, `validate` catches the mismatch at startup and prevents the app from starting with an inconsistent schema.
- **Flyway owns the schema** — Flyway runs first and creates/alters tables. Hibernate then validates them. Clear separation of responsibilities.
- **No surprise changes** — Hibernate never alters the database structure. Only Flyway SQL files change the schema.

**Why NOT `update`?**
`ddl-auto=update` is extremely dangerous for production because:
1. It adds columns but NEVER removes them (orphaned columns accumulate).
2. It can't rename columns (adds new, leaves old).
3. Behavior is unpredictable with complex entity graphs.
4. Many DBAs consider it a data loss risk — it's been known to drop and recreate foreign keys in edge cases.

---

### Q16. Why use `@Transactional(readOnly = true)` on read operations?

**Used in:** `IncidentService.findAll()`, `DashboardService.getDashboardData()`, all repository lookups.

```java
@Transactional(readOnly = true)  // ← this annotation
public PagedResponse<IncidentResponse> findAll(...) {
    Page<Incident> page = incidentRepository.findWithFilters(...);
    // ...
}
```

**Why readOnly=true:**

**Performance advantages:**
1. **Hibernate skips dirty-checking** — In a normal `@Transactional`, Hibernate tracks all entity state changes (dirty checking) so it can flush changes to the DB on commit. `readOnly=true` disables this tracking entirely. For a query returning 100 incidents, this saves 100 entity comparisons.
2. **Hibernate flush mode set to MANUAL** — No accidental writes to the database even if an entity is modified.
3. **Database driver optimization** — Some JDBC drivers (including SQL Server) send a read-only hint to the database, allowing the database to use read-only replicas in a clustered setup.
4. **Connection pool hint** — HikariCP can route read-only transactions to read replicas in Azure SQL or Always On Availability Groups.

**Drawbacks:**
- **LazyInitializationException risk** — If a read-only service method loads an entity and then the response DTO tries to access a lazy-loaded collection after the session closes, you get `LazyInitializationException`. Solved with `JOIN FETCH` in the JPQL query.

---

### Q17. Why was the repository `findWithFilters()` method designed with null-safe parameters?

**Pattern used:**
```sql
WHERE (:status IS NULL OR i.status = :status)
  AND (:priority IS NULL OR i.priority = :priority)
  AND (:search IS NULL OR LOWER(i.title) LIKE ...)
```

**Alternatives:**
1. **Separate repository methods** — `findAll()`, `findByStatus()`, `findByStatusAndPriority()`, etc. Results in 8 methods for 3 filters with combinations.
2. **Spring Data Specification (Criteria API)** — Type-safe dynamic queries. More code, more flexible.
3. **Querydsl** — Code-generation based, very flexible. Additional build step required.
4. **Null-safe JPQL** — The pattern used in ITSM-Pro.

**Why null-safe JPQL was selected:**

**Advantages:**
- **Single method handles all filter combinations** — One repository method instead of 2^N combinations of optional filters.
- **Readable JPQL** — Plain SQL-like syntax that any developer can understand immediately.
- **Database efficiency** — Modern query optimizers recognize `(:x IS NULL OR col = :x)` as equivalent to no filter when `:x` is null. SQL Server's query optimizer handles this correctly.

**Drawbacks:**
- **Performance at scale** — For millions of rows with complex dynamic queries, JPA Specification API or Querydsl produces better-optimized queries. The null-safe pattern can prevent optimal index usage in some edge cases.
- **Limited flexibility** — Adding a 4th filter requires editing the `@Query` string. Specification pattern is more extensible.

---

### Q18. Why is `spring.jpa.open-in-view=false` set explicitly?

**Default (Spring Boot):** `true` (bad for production)
**ITSM-Pro setting:** `false`

**What `open-in-view=true` does:**
Keeps the Hibernate session open for the entire HTTP request lifecycle, including the view rendering phase. This allows lazy loading of JPA entities in Thymeleaf templates or Jackson serialization.

**Why it's dangerous:**
1. **Hidden N+1 queries** — Serializing an incident to JSON might silently trigger 20 extra database queries to load lazy associations (assignedTo, assignmentGroup, createdBy...). These queries run during response serialization, invisible in service-layer logs.
2. **Long-held connections** — The database connection is held for the full request duration, including JSON serialization time. Reduces effective connection pool capacity.
3. **False sense of correctness** — Tests pass, but production is hammered by hundreds of extra queries.

**Why `false` was selected:**
With `open-in-view=false`, lazy associations not loaded in the service layer throw `LazyInitializationException` immediately. This forces correct `JOIN FETCH` usage in repository queries and explicit DTO construction in service methods — exactly the pattern used throughout ITSM-Pro.

---

### Q19. Why use `@CreationTimestamp` and `@UpdateTimestamp` instead of `@PrePersist`?

**Pattern used:**
```java
@CreationTimestamp
@Column(name = "created_at", nullable = false, updatable = false)
private LocalDateTime createdAt;

@UpdateTimestamp
@Column(name = "updated_at")
private LocalDateTime updatedAt;
```

**Alternatives:**
1. `@PrePersist` + `@PreUpdate` lifecycle callbacks — Manual Java code.
2. `@EntityListeners(AuditingEntityListener.class)` + Spring Data Auditing — More features (createdBy, lastModifiedBy), more configuration.
3. Database defaults (`DEFAULT GETUTCDATE()`) — DB-side, no Java involvement.
4. `@CreationTimestamp` / `@UpdateTimestamp` — **Hibernate-native** (chosen).

**Why Hibernate annotations were selected:**

**Advantages:**
- **Zero code** — Hibernate populates these automatically on every `save()`. No `@PrePersist` method needed.
- **`updatable = false` on createdAt** — Prevents accidentally overwriting the creation timestamp on updates. Enforced at Hibernate level.
- **LocalDateTime** — Returns Java's modern `LocalDateTime` (not old `java.util.Date`). No conversion needed.
- **Simple** — Just two annotations. `@EntityListeners` requires a separate configuration class and `@EnableJpaAuditing` on the application class.

**Drawbacks:**
- **Hibernate-specific** — Not a JPA standard annotation (JPA uses `@PrePersist`). Couples entities to Hibernate implementation.
- **No `createdBy`** — If you need to auto-populate "who created this record," you need Spring Data Auditing with `@CreatedBy`. Hibernate's annotations only handle timestamps.

---

### Q20. Why use `@Index` annotations on entities instead of only Flyway SQL indexes?

**Pattern used:**
```java
@Table(name = "incidents", indexes = {
    @Index(name = "idx_incidents_ticket_number", columnList = "ticket_number", unique = true),
    @Index(name = "idx_incidents_status",        columnList = "status")
})
```

**Why both are used:**

**Flyway SQL indexes (V1__Initial_Schema.sql):** Definitive source of truth for the database schema. Always created correctly.

**`@Index` annotations on entities:** Documentation — they tell developers reading the entity class what indexes exist. During `ddl-auto=validate`, Hibernate checks that indexes defined in `@Index` exist in the database.

**Advantages of both:**
- **Self-documenting entities** — A developer reading `Incidents.java` immediately sees which columns are indexed without opening SSMS or reading the SQL migration.
- **Validation** — If a Flyway migration accidentally drops an index, `ddl-auto=validate` catches the mismatch.
- **No duplication risk** — Since `ddl-auto=validate` (not `create`), Hibernate never creates indexes — it only validates them.

---

## SECTION 3: API & Documentation (Q21–Q25)

---

### Q21. Why SpringDoc OpenAPI 2.6.0 instead of Springfox Swagger?

**Chosen:** SpringDoc OpenAPI 2.6.0

**Options compared:**
| Library | Spring Boot 3 Support | OpenAPI Version | Maintained |
|---------|----------------------|-----------------|-----------|
| **SpringDoc** | ✅ Full | OpenAPI 3.1 | ✅ Active |
| Springfox | ❌ Broken on Boot 3.x | OpenAPI 2.0 | ❌ Abandoned |
| Swagger Core | Manual config | OpenAPI 3.0 | ✅ Active |

**Why SpringDoc was selected:**

**Advantages:**
- **Spring Boot 3 support** — Springfox was not updated for Spring Boot 3 and the `javax.*` → `jakarta.*` migration. It's effectively abandoned for Spring Boot 3.x projects.
- **Auto-configuration** — SpringDoc scans `@RestController` classes and generates OpenAPI 3.1 spec automatically. No manual configuration required.
- **Swagger UI included** — Bundled UI available at `/swagger-ui.html`. No separate hosting needed.
- **Try-it-out** — Developers can test every API endpoint directly from the browser. JWT authentication works via the BearerAuth scheme.
- **OpenAPI 3.1** — Latest spec version, supports JSON Schema, webhooks, and better null handling.

**Drawbacks:**
- **Auto-generation can be too verbose** — Generated spec includes every endpoint including actuator endpoints unless explicitly excluded.
- **Production security** — Swagger UI must be disabled in production (`springdoc.api-docs.enabled=false` in `application-prod.properties`).

---

### Q22. Why use `@Operation` and `@Tag` Swagger annotations everywhere?

**Why explicit Swagger annotations instead of relying on auto-generation:**

```java
@GetMapping
@Operation(
    summary = "List incidents with optional filtering and pagination",
    description = "Supports status, priority, search text filtering"
)
@Tag(name = "Incident Management")
public ResponseEntity<PagedResponse<IncidentResponse>> getAllIncidents(...) {
```

**Advantages:**
- **Human-readable documentation** — Auto-generated names like `getAllIncidents` are cryptic. `"List incidents with optional filtering and pagination"` is clear to non-developers (product managers, business analysts reading the API docs).
- **Grouping** — `@Tag(name = "Incident Management")` groups all incident endpoints under one section in Swagger UI, separate from changes and auth endpoints.
- **Parameter documentation** — `@Parameter(description = "...")` explains what each query param does, especially important for enum values like `status` and `priority`.

**Drawbacks:**
- **Maintenance overhead** — When you rename an endpoint or change behaviour, you must update the annotations too.
- **Annotation noise** — `@Operation` annotations add 3-5 lines per method, increasing visual noise in controllers.

---

### Q23. Why is the REST API designed with `PagedResponse<T>` wrapper instead of returning plain lists?

**Pattern used:**
```json
{
  "content": [...],
  "page": 0,
  "size": 20,
  "totalElements": 142,
  "totalPages": 8,
  "last": false
}
```

**Why not just return `List<IncidentResponse>`?**

**Advantages of `PagedResponse<T>`:**
- **Client knows pagination state** — `totalPages` tells the React frontend exactly when to stop fetching. `last: true` means "this is the final page."
- **Generic reuse** — `PagedResponse<T>` works for incidents, changes, users, and any future entity. One DTO, many uses.
- **`@Builder` construction** — `PagedResponse.builder().content(...).totalElements(...).build()` is clean and explicit.
- **Frontend React Query compatibility** — `usePendingApprovals()` hook checks `data.last` to decide whether to show a "Load More" button.

**Drawbacks:**
- **Nesting** — The response body has one extra level of nesting. Clients must access `data.content` not just `data`.
- **Over-fetching metadata** — If a client only needs the items and doesn't use pagination, it receives unnecessary metadata fields.

---

### Q24. Why use `@PreAuthorize` on methods instead of only URL-level security in `SecurityConfig`?

**Dual approach used:**
```java
// URL-level in SecurityConfig.java:
.requestMatchers("/api/admin/**").hasRole("ADMIN")

// Method-level in controller:
@DeleteMapping("/{id}")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<Void> cancelIncident(...) {
```

**Why both levels are used:**

**URL-level advantages:** Fast — Spring Security checks before the request reaches the controller. Good for broad rules (all `/api/admin/**` requires admin).

**Method-level advantages:**
- **Granularity** — `DELETE /api/incidents/{id}` (admin only) vs `PATCH /api/incidents/{id}/status` (all users) cannot be distinguished by URL pattern alone.
- **Code proximity** — The permission requirement is visible next to the method that needs it. No need to cross-reference `SecurityConfig`.
- **Testable** — `@PreAuthorize` can be tested with `@WithMockUser(roles = "MANAGER")` in Spring Security tests.
- **Dynamic expressions** — `@PreAuthorize("@incidentSecurity.canAccess(#id, principal)")` calls a Spring bean for complex business logic.

**Drawbacks:**
- **Double configuration** — Same access rule potentially defined in two places. Can cause confusion if they conflict.
- **AOP proxy requirement** — `@EnableMethodSecurity` requires Spring AOP proxies. Internal method calls (same class) bypass `@PreAuthorize`.

---

### Q25. Why use `ResponseEntity<T>` as the controller return type instead of plain `T`?

**Pattern used:**
```java
// ITSM-Pro (with ResponseEntity):
public ResponseEntity<IncidentResponse> createIncident(...) {
    return ResponseEntity.status(HttpStatus.CREATED).body(response);
}

// Alternative (plain return):
public IncidentResponse createIncident(...) {
    return response;  // always returns 200 OK
}
```

**Why `ResponseEntity<T>` was selected:**

**Advantages:**
- **Explicit HTTP status codes** — `201 Created` for POST, `204 No Content` for DELETE, `200 OK` for GET. Without `ResponseEntity`, Spring always returns 200.
- **Header control** — Can add `Location` header pointing to the new resource URL: `ResponseEntity.created(URI.create("/api/incidents/" + id)).body(response)`
- **Clear intent** — Readers immediately see what HTTP status is returned.
- **REST compliance** — Proper REST APIs use 201 for resource creation, 204 for deletion. Returning 200 for POST is technically incorrect.

**Drawbacks:**
- **Verbosity** — `ResponseEntity.status(HttpStatus.CREATED).body(response)` is more to type than `return response`.
- **Generic complexity** — `ResponseEntity<PagedResponse<IncidentResponse>>` is a deeply nested generic type.
