# 🍃 Spring Boot — Backend Interview Prep

> Comprehensive Spring & Spring Boot notes for backend interview prep. Covers Core Spring, Boot, MVC, JPA/Hibernate, WebFlux, Microservices, Exception Handling. Includes interview Q&A and production-style code samples.

---

## 📑 Table of Contents

1. [Spring Ecosystem Overview](#spring-ecosystem-overview)
2. [Spring Core — IoC & DI](#spring-core--ioc--di)
3. [Bean Scopes](#bean-scopes)
4. [Configuration & Profiles](#configuration--profiles)
5. [Spring Boot Auto-Configuration](#spring-boot-auto-configuration)
6. [REST Controllers & ResponseEntity](#rest-controllers--responseentity)
7. [Validation](#validation)
8. [Exception Handling with @ControllerAdvice](#exception-handling-with-controlleradvice)
9. [Spring Data JPA & Hibernate](#spring-data-jpa--hibernate)
10. [Inheritance Strategies in JPA](#inheritance-strategies-in-jpa)
11. [Transactions](#transactions)
12. [Spring WebFlux & Reactive](#spring-webflux--reactive)
13. [WebClient — Calling Downstream Services](#webclient--calling-downstream-services)
14. [Microservices Patterns](#microservices-patterns)
15. [Service Discovery (Eureka)](#service-discovery-eureka)
16. [Security Quick Reference](#security-quick-reference)
17. [Top 50 Interview Q&A](#top-50-interview-qa)

---

## Spring Ecosystem Overview

| Module | Purpose |
|--------|---------|
| **Spring Core** | IoC container, DI, beans, AOP |
| **Spring Boot** | Auto-config, embedded server, starters, opinionated defaults |
| **Spring MVC** | Servlet-based REST/web layer |
| **Spring Data** | Repository abstraction over JPA, MongoDB, Redis, etc. |
| **Spring REST** | RESTful web services |
| **Spring Security** | Auth (basic, JWT, OAuth2), authz, CSRF, CORS |
| **Spring Kafka / Redis / Docker / Cloud** | Integrations & cloud-native tooling |
| **Spring WebFlux** | Reactive non-blocking stack (Reactor) |
| **Microservices** | Eureka, Config Server, Gateway, Resilience4j |

---

## Spring Core — IoC & DI

### Inversion of Control (IoC)

Object creation and lifecycle managed by the **Spring container** instead of the application code. The container injects dependencies where needed.

### Dependency Injection (DI) — Three Types

```java
// 1. Constructor injection (PREFERRED — testable, immutable, fails fast on missing deps)
@Component
public class OrderService {
    private final InventoryService inventory;

    public OrderService(InventoryService inventory) {
        this.inventory = inventory;
    }
}

// 2. Setter injection
@Component
public class OrderService {
    private InventoryService inventory;
    @Autowired
    public void setInventory(InventoryService inventory) { this.inventory = inventory; }
}

// 3. Field injection (avoid — hides dependencies, harder to test)
@Component
public class OrderService {
    @Autowired private InventoryService inventory;
}
```

### Stereotype Annotations

| Annotation | Used on |
|------------|---------|
| `@Component` | Generic Spring-managed bean |
| `@Service` | Service / business-logic layer |
| `@Repository` | Data-access layer (also enables exception translation) |
| `@Controller` | Spring MVC controller (returns view) |
| `@RestController` | `@Controller` + `@ResponseBody` (returns JSON) |
| `@Configuration` | Class containing `@Bean` definitions |

### `@Component` vs `@ComponentScan`

- `@Component` tells Spring "create an instance of this class" — auto-scan picks it up.
- `@ComponentScan` specifies which packages Spring should scan for `@Component`, `@Service`, `@Configuration`.

```java
@SpringBootApplication
@ComponentScan(basePackages = {"com.example.app", "com.example.shared"})
public class Application { }
```

### `@SpringBootApplication` Decomposed

`@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan(basePackages = current package)`

### `@Primary` & `@Qualifier`

When multiple beans of the same type exist, Spring throws `UnsatisfiedDependencyException`. Resolve via:

```java
// Mark one as default
@Service @Primary
public class OracleDatabaseService implements DatabaseService { }

// Or pick by name
@Autowired
public OrderService(@Qualifier("oracleDatabaseService") DatabaseService db) { }
```

---

## Bean Scopes

```java
@Component
@Scope("prototype")
public class MyBean { }
```

| Scope | Lifecycle |
|-------|-----------|
| `singleton` (**default**) | One instance per Spring container |
| `prototype` | New instance every time bean is requested |
| `request` | One instance per HTTP request (web only) |
| `session` | One instance per HTTP session (web only) |
| `application` | One instance per ServletContext |

**Verification:**

```java
@Scope("singleton")
// container.getBean(MyBean.class) returns SAME object each time

@Scope("prototype")
// container.getBean(MyBean.class) returns NEW object each time
```

### `@Lookup` for Dynamic Prototype Creation

When a singleton bean depends on a prototype, Spring would inject the prototype only **once** (defeating the point). `@Lookup` overrides a method to fetch a fresh prototype each call.

```java
@Component
public abstract class SingletonBean {
    @Lookup
    public abstract PrototypeBean getPrototypeBean();

    public void doWork() {
        PrototypeBean fresh = getPrototypeBean(); // new instance each call
    }
}
```

### Lifecycle Hooks

```java
@Component
public class MyBean {
    @PostConstruct
    public void init() { /* runs after DI complete */ }

    @PreDestroy
    public void cleanup() { /* runs before bean destroyed */ }
}
```

---

## Configuration & Profiles

### `application.yml` / `application.properties`

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: ${DB_USER}
    password: ${DB_PASSWORD}
  jpa:
    hibernate:
      ddl-auto: validate
    show-sql: false
```

### `@Value` Injection

```java
@Component
public class AppInfo {
    @Value("${app.name}")
    private String appName;

    @Value("${app.version:1.0}") // default value
    private String version;
}
```

### `@ConfigurationProperties` (preferred for grouped configs)

```yaml
app:
  name: my-service
  version: 2.3
  features:
    - auth
    - billing
```

```java
@Component
@ConfigurationProperties(prefix = "app")
public class AppConfig {
    private String name;
    private String version;
    private List<String> features;
    // getters/setters required (or use records in Java 17+)
}
```

Enable on app class:
```java
@SpringBootApplication
@EnableConfigurationProperties(AppConfig.class)
public class Application { }
```

### Profiles — Environment-Specific Beans

```java
@Service
@Profile("dev")
public class MockEmailService implements EmailService { }

@Service
@Profile("prod")
public class SendGridEmailService implements EmailService { }
```

```yaml
spring:
  profiles:
    active: dev
```

Or via env: `SPRING_PROFILES_ACTIVE=prod`

### `@PropertySource` — Load External Config Files

```java
@Configuration
@PropertySource("classpath:custom.properties")
public class AppConfig {
    @Value("${app.name}")
    private String appName;
}
```

---

## Spring Boot Auto-Configuration

Spring Boot inspects the classpath, beans, and properties to **auto-configure** the application.

**How it works:**
1. `@EnableAutoConfiguration` triggers loading of `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`.
2. Each auto-config class is gated by `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`.
3. Example: if `DataSource` is on classpath and a JDBC URL is configured → `DataSourceAutoConfiguration` kicks in.

**Disable specific auto-config:**
```java
@SpringBootApplication(exclude = { DataSourceAutoConfiguration.class })
public class Application { }
```

---

## REST Controllers & ResponseEntity

### Basic REST Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService service;

    public UserController(UserService service) { this.service = service; }

    @GetMapping("/{id}")
    public User getUser(@PathVariable int id) {
        return service.findById(id);
    }

    @PostMapping
    public User create(@RequestBody @Valid UserDto dto) {
        return service.create(dto);
    }

    @GetMapping
    public List<User> list(@RequestParam(defaultValue = "0") int page) {
        return service.list(page);
    }
}
```

### Common Annotations

| Annotation | Purpose |
|------------|---------|
| `@RestController` | Class-level: marks REST controller |
| `@RequestMapping` | Class or method-level URL prefix/path |
| `@GetMapping` / `@PostMapping` / `@PutMapping` / `@DeleteMapping` / `@PatchMapping` | HTTP method shortcuts |
| `@PathVariable` | Bind URL path segment to method param |
| `@RequestParam` | Bind query string param |
| `@RequestBody` | Bind request body (JSON) to object |
| `@RequestHeader` | Bind HTTP header to param |
| `@ResponseStatus(HttpStatus.CREATED)` | Set response status code |

### Returning Custom Status with `ResponseEntity`

```java
@GetMapping("/{id}")
public ResponseEntity<User> getUser(@PathVariable int id) {
    User user = service.findById(id);
    return ResponseEntity.ok(user);                    // 200
}

// 200 with custom headers
return ResponseEntity.ok()
    .headers(headers)
    .body(user);

// 201 Created with Location header
return ResponseEntity.created(URI.create("/users/" + id)).body(user);

// 204 No Content
return ResponseEntity.noContent().build();

// 404 Not Found
return ResponseEntity.notFound().build();

// 500
return ResponseEntity.internalServerError().body("error");

// Generic — set any status
return ResponseEntity.status(HttpStatus.UNPROCESSABLE_ENTITY).body(errorBody);
```

### Pagination Pattern

```java
@GetMapping
public Page<User> list(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "20") int size,
        @RequestParam(defaultValue = "id,asc") String sort) {

    Sort sortObj = Sort.by(sort.split(",")[0])
                       .ascending(); // or descending()
    Pageable pageable = PageRequest.of(page, size, sortObj);
    return userRepository.findAll(pageable);
}
```

---

## Validation

### Bean Validation (JSR 380) — `@Valid` + Annotations

```java
public record UserDto(
    @NotBlank @Size(min = 2, max = 50) String name,
    @Email String email,
    @Min(18) @Max(120) int age,
    @NotNull @Pattern(regexp = "^[+]?[0-9]{10,15}$") String phone
) {}

@PostMapping
public User create(@RequestBody @Valid UserDto dto) {
    return service.create(dto);
}
```

### Common Validation Annotations

| Annotation | Use |
|------------|-----|
| `@NotNull` | Not null |
| `@NotBlank` | Not null + not empty + not whitespace (String) |
| `@NotEmpty` | Not null + size > 0 (String, Collection) |
| `@Size(min,max)` | Length/size constraint |
| `@Min` / `@Max` | Numeric range |
| `@Email` | Valid email format |
| `@Pattern(regexp=...)` | Regex match |
| `@Past`, `@Future` | Date constraints |
| `@Positive`, `@Negative` | Sign constraints |

**Validation failure** → throws `MethodArgumentNotValidException`. Handle globally:

---

## Exception Handling with @ControllerAdvice

> ⭐ **Critical interview topic — covers production maturity.**

### Why Centralize Exception Handling?

- Avoid try/catch clutter in every controller method.
- Consistent error response shape across the API.
- Map domain exceptions to proper HTTP status codes.
- Log + propagate trace IDs uniformly.

### Standard Error Response Shape

```java
public record ErrorResponse(
    String code,
    String message,
    Instant timestamp,
    String path,
    List<FieldError> errors
) {}

public record FieldError(String field, String message) {}
```

### Production-Style `@ControllerAdvice`

```java
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // 400 — Bean validation failures
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest req) {

        List<FieldError> fields = ex.getBindingResult().getFieldErrors().stream()
            .map(fe -> new FieldError(fe.getField(), fe.getDefaultMessage()))
            .toList();

        ErrorResponse body = new ErrorResponse(
            "VALIDATION_FAILED", "Request validation failed",
            Instant.now(), req.getRequestURI(), fields
        );
        return ResponseEntity.badRequest().body(body);
    }

    // 400 — bad input from path/query
    @ExceptionHandler({ ConstraintViolationException.class,
                       MethodArgumentTypeMismatchException.class })
    public ResponseEntity<ErrorResponse> handleBadRequest(Exception ex, HttpServletRequest req) {
        return buildResponse(HttpStatus.BAD_REQUEST, "BAD_REQUEST", ex.getMessage(), req);
    }

    // 404 — domain not-found
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex, HttpServletRequest req) {
        return buildResponse(HttpStatus.NOT_FOUND, "NOT_FOUND", ex.getMessage(), req);
    }

    // 409 — state conflicts (duplicate, optimistic lock)
    @ExceptionHandler({ DataIntegrityViolationException.class,
                        OptimisticLockingFailureException.class })
    public ResponseEntity<ErrorResponse> handleConflict(Exception ex, HttpServletRequest req) {
        log.warn("Conflict: {}", ex.getMessage());
        return buildResponse(HttpStatus.CONFLICT, "CONFLICT", "Resource conflict", req);
    }

    // 401 / 403 — auth issues
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleForbidden(AccessDeniedException ex, HttpServletRequest req) {
        return buildResponse(HttpStatus.FORBIDDEN, "FORBIDDEN", "Access denied", req);
    }

    // 503 — downstream failure
    @ExceptionHandler({ WebClientResponseException.class,
                        ResourceAccessException.class })
    public ResponseEntity<ErrorResponse> handleDownstream(Exception ex, HttpServletRequest req) {
        log.error("Downstream failure", ex);
        return buildResponse(HttpStatus.SERVICE_UNAVAILABLE,
            "DOWNSTREAM_UNAVAILABLE", "Downstream service unavailable", req);
    }

    // 500 — fallback (log full trace, never leak details)
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAny(Exception ex, HttpServletRequest req) {
        log.error("Unhandled exception", ex);
        return buildResponse(HttpStatus.INTERNAL_SERVER_ERROR,
            "INTERNAL_ERROR", "Unexpected error occurred", req);
    }

    private ResponseEntity<ErrorResponse> buildResponse(
            HttpStatus status, String code, String message, HttpServletRequest req) {
        ErrorResponse body = new ErrorResponse(
            code, message, Instant.now(), req.getRequestURI(), List.of()
        );
        return ResponseEntity.status(status).body(body);
    }
}
```

### Custom Domain Exception

```java
@ResponseStatus(HttpStatus.NOT_FOUND)
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, Object id) {
        super(resource + " not found with id: " + id);
    }
}

// In service
public User findById(int id) {
    return repo.findById(id)
        .orElseThrow(() -> new ResourceNotFoundException("User", id));
}
```

### Demo Scenario — Validation + NotFound

```bash
# Request: POST /api/users with missing name
{ "email": "alice@example.com", "age": 30 }

# Response 400
{
  "code": "VALIDATION_FAILED",
  "message": "Request validation failed",
  "timestamp": "2026-04-26T10:15:30Z",
  "path": "/api/users",
  "errors": [
    { "field": "name", "message": "must not be blank" }
  ]
}

# Request: GET /api/users/99 (no such user)
# Response 404
{
  "code": "NOT_FOUND",
  "message": "User not found with id: 99",
  "timestamp": "2026-04-26T10:16:00Z",
  "path": "/api/users/99",
  "errors": []
}
```

### `@ControllerAdvice` vs `@RestControllerAdvice`

`@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody` → handler return values are serialized to JSON.

### Scoping `@ControllerAdvice`

```java
// Apply only to specific package
@RestControllerAdvice(basePackages = "com.example.api.v1")

// Apply only to specific annotation
@RestControllerAdvice(annotations = RestController.class)

// Apply only to specific controllers
@RestControllerAdvice(assignableTypes = { UserController.class })
```

---

## Spring Data JPA & Hibernate

### Entity Basics

```java
@Entity
@Table(name = "authors")
public class Author {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(unique = true)
    private String email;

    @CreationTimestamp
    private Instant createdAt;

    @UpdateTimestamp
    private Instant updatedAt;

    // getters / setters / equals / hashCode (use id only)
}
```

### `@GeneratedValue` Strategies

| Strategy | How |
|----------|-----|
| `AUTO` | JPA picks based on DB |
| `IDENTITY` | DB auto-increment column |
| `SEQUENCE` | DB sequence (Postgres, Oracle) |
| `TABLE` | Separate table to track values |

```java
// SEQUENCE example
@Id
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "author_seq")
@SequenceGenerator(name = "author_seq", sequenceName = "author_sequence", allocationSize = 1)
private Long id;

// TABLE example
@Id
@GeneratedValue(strategy = GenerationType.TABLE, generator = "author_id_gen")
@TableGenerator(
    name = "author_id_gen",
    table = "id_generator",
    pkColumnName = "id_name",
    valueColumnName = "id_value",
    allocationSize = 1
)
private Long id;
```

### Relationships

```java
// One-to-Many
@OneToMany(mappedBy = "author", cascade = CascadeType.ALL, orphanRemoval = true)
private List<Book> books = new ArrayList<>();

// Many-to-One (owning side)
@ManyToOne(fetch = FetchType.LAZY)
@JoinColumn(name = "author_id")
private Author author;

// Many-to-Many
@ManyToMany
@JoinTable(name = "student_course",
    joinColumns = @JoinColumn(name = "student_id"),
    inverseJoinColumns = @JoinColumn(name = "course_id"))
private Set<Course> courses = new HashSet<>();
```

> ⚠️ **Always use `LAZY` fetching** by default. `EAGER` causes N+1 issues and unintended joins.

### Spring Data Repository

```java
public interface UserRepository extends JpaRepository<User, Long> {

    // Derived query method
    Optional<User> findByEmail(String email);
    List<User> findByAgeGreaterThanAndStatus(int age, Status status);

    // JPQL
    @Query("SELECT u FROM User u WHERE u.email LIKE %:domain")
    List<User> findByEmailDomain(@Param("domain") String domain);

    // Native SQL
    @Query(value = "SELECT * FROM users WHERE created_at > ?1", nativeQuery = true)
    List<User> findRecent(Instant since);

    // Modifying
    @Modifying
    @Query("UPDATE User u SET u.status = :status WHERE u.id = :id")
    int updateStatus(@Param("id") Long id, @Param("status") Status status);
}
```

### N+1 Problem

**The problem:** Loading 1 list of 100 authors then for each loading their books = 1 + 100 queries.

**Solutions:**

```java
// 1. Fetch join in JPQL
@Query("SELECT a FROM Author a JOIN FETCH a.books")
List<Author> findAllWithBooks();

// 2. EntityGraph
@EntityGraph(attributePaths = {"books"})
List<Author> findAll();

// 3. Batch fetching (in entity)
@OneToMany(mappedBy = "author")
@BatchSize(size = 50)
private List<Book> books;
```

### JPA First-Level vs Second-Level Cache

- **L1 (Persistence Context / Session)** — per-transaction, automatic, can't disable.
- **L2 (SessionFactory / EMF)** — across sessions, explicit (Ehcache, Hazelcast, Caffeine via Hibernate).
- **Query Cache** — caches query results; requires L2.

---

## Inheritance Strategies in JPA

### `@Inheritance` Options

```java
@Entity
@Inheritance(strategy = InheritanceType.SINGLE_TABLE)
@DiscriminatorColumn(name = "resource_type")
public abstract class Resource {
    @Id @GeneratedValue private Long id;
    private String name;
}

@Entity
@DiscriminatorValue("F")
public class File extends Resource { }

@Entity
@DiscriminatorValue("V")
public class Video extends Resource { }

@Entity
@DiscriminatorValue("T")
public class Text extends Resource { }
```

### Strategies Compared

| Strategy | Tables | Pros | Cons |
|----------|--------|------|------|
| `SINGLE_TABLE` (default) | 1 table for parent + all children, with discriminator column | Fast (no joins), simple | Many nullable columns, can't `NOT NULL` on subclass fields |
| `JOINED` | One table per class; joined via PK | Normalized, no nulls | Slow on read (multiple joins) |
| `TABLE_PER_CLASS` | Separate table for each concrete class | No joins for single class | Polymorphic queries need UNIONs |
| `MAPPED_SUPERCLASS` (not `@Inheritance`) | No table for parent; fields inherited | Pure code reuse | Parent isn't queryable |

> 💡 **In your notes:** with `SINGLE_TABLE`, all File/Video/Text rows live in `Resource` table. The default discriminator column is `dtype`. Use `@DiscriminatorColumn(name = "resource_type")` to customize, and `@DiscriminatorValue("T")` on each subclass.

---

## Transactions

### `@Transactional`

```java
@Service
public class TransferService {

    @Transactional
    public void transfer(Long from, Long to, BigDecimal amount) {
        accountRepo.debit(from, amount);
        accountRepo.credit(to, amount);
    }
}
```

### Propagation

| Propagation | Behavior |
|-------------|----------|
| `REQUIRED` (default) | Use existing or start new |
| `REQUIRES_NEW` | Always start new (suspends existing) |
| `SUPPORTS` | Use existing if any |
| `MANDATORY` | Throw if no transaction |
| `NEVER` | Throw if transaction exists |
| `NESTED` | Savepoint within outer transaction |

### Isolation

| Level | Prevents |
|-------|----------|
| `READ_UNCOMMITTED` | Nothing |
| `READ_COMMITTED` (most DBs default) | Dirty reads |
| `REPEATABLE_READ` | Dirty + non-repeatable reads |
| `SERIALIZABLE` | All anomalies (slowest) |

### Rollback Rules

By default, only **runtime exceptions** trigger rollback. For checked exceptions:

```java
@Transactional(rollbackFor = { IOException.class })
public void process() throws IOException { ... }
```

### Common Pitfalls

- **Self-invocation** — `this.foo()` doesn't go through proxy; transaction won't apply. Inject the bean into itself or restructure.
- **`@Transactional` on private methods** — also bypasses the proxy. Always public.
- **Long-running transactions** — hold locks too long. Keep them tight.

---

## Spring WebFlux & Reactive

### Why Reactive?

- **Non-blocking I/O** with **backpressure**.
- Threads aren't blocked waiting on downstream calls → high throughput on small thread pools.
- Suited for: streaming, high-concurrency I/O-bound microservices, SSE, server-side fan-out.

> 🔄 With **Java 21 virtual threads**, plain blocking code now scales similarly for many use cases. Reactive still wins for **streaming**, **backpressure**, and **functional composition**.

### Mono vs Flux

| Type | Emits |
|------|-------|
| `Mono<T>` | 0 or 1 element |
| `Flux<T>` | 0 to N elements |

```java
// Creating
Mono<String> a = Mono.just("hello");
Mono<String> empty = Mono.empty();
Flux<Integer> b = Flux.range(1, 5);
Flux<String> c = Flux.fromIterable(List.of("a","b","c"));

// Subscribing
a.subscribe(System.out::println, Throwable::printStackTrace, () -> System.out.println("done"));
```

### WebFlux Controller

```java
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public Mono<User> getUser(@PathVariable String id) {
        return userService.findById(id);
    }

    @GetMapping
    public Flux<User> listUsers() {
        return userService.findAll();
    }

    @GetMapping(value = "/stream", produces = MediaType.TEXT_EVENT_STREAM_VALUE)
    public Flux<Event> stream() {
        return eventService.streamEvents();
    }
}
```

### Common Operators

| Operator | Purpose |
|----------|---------|
| `map(Function)` | Transform each element |
| `flatMap(Function)` | Async transform — returns `Mono`/`Flux` |
| `filter(Predicate)` | Keep matching |
| `zip(...)` | Combine multiple sources |
| `merge(...)` | Interleave streams |
| `concat(...)` | Sequence streams |
| `timeout(Duration)` | Fail if no signal within window |
| `retry(n)` / `retryWhen(...)` | Retry on error |
| `onErrorResume(fn)` | Fallback on error |
| `onErrorReturn(value)` | Fixed fallback value |
| `defaultIfEmpty(v)` | Value if no element emitted |
| `subscribeOn(Scheduler)` | Where the source runs |
| `publishOn(Scheduler)` | Where downstream runs |

---

## WebClient — Calling Downstream Services

### Bean Configuration

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        HttpClient httpClient = HttpClient.create()
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 2000)
            .responseTimeout(Duration.ofSeconds(5));

        return WebClient.builder()
            .baseUrl("https://api.example.com")
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}
```

### Single Downstream Call

```java
public Mono<User> getUser(String id) {
    return webClient.get()
        .uri("/users/{id}", id)
        .retrieve()
        .bodyToMono(User.class)
        .timeout(Duration.ofSeconds(2))
        .onErrorResume(ex -> {
            log.warn("Falling back for user {}", id, ex);
            return Mono.just(new User(id, "default-user"));
        });
}
```

### ⭐ Parallel Calls — `Mono.zip` (Production Pattern)

> **Top interview question.** This pattern shows up in nearly every backend interview at Capital One, Microsoft, and similar.

```java
public Mono<CombinedResponse> getCombinedData(String userId) {

    Mono<User> userMono = getUser(userId);
    Mono<Account> accountMono = getAccount(userId);
    Mono<Profile> profileMono = getProfile(userId);
    Mono<Preference> prefMono = getPreference(userId);

    return Mono.zip(userMono, accountMono, profileMono, prefMono)
        .map(tuple -> new CombinedResponse(
            tuple.getT1(),  // User
            tuple.getT2(),  // Account
            tuple.getT3(),  // Profile
            tuple.getT4()   // Preference
        ));
}
```

### Adding Timeout + Fallback (Real-World)

```java
private Mono<User> getUser(String userId) {
    return webClient.get()
        .uri("/user/{id}", userId)
        .retrieve()
        .bodyToMono(User.class)
        .timeout(Duration.ofSeconds(2))
        .onErrorResume(ex -> Mono.just(new User("default-user")));
}
```

### Adding Retry (Transient Failure Handling)

```java
private Mono<Account> getAccount(String userId) {
    return webClient.get()
        .uri("/account/{id}", userId)
        .retrieve()
        .bodyToMono(Account.class)
        .retryWhen(
            Retry.backoff(2, Duration.ofMillis(200))
                 .filter(ex -> ex instanceof IOException
                            || ex instanceof TimeoutException)
                 .onRetryExhaustedThrow((spec, signal) -> signal.failure())
        );
}
```

### Final Production-Style Aggregation

```java
public Mono<CombinedResponse> getCombinedData(String userId) {
    return Mono.zip(
        getUser(userId),
        getAccount(userId),
        getProfile(userId),
        getPreference(userId)
    ).map(tuple -> new CombinedResponse(
        tuple.getT1(), tuple.getT2(), tuple.getT3(), tuple.getT4()
    ));
}
```

### 💼 What to Say in Interview

> "We used Spring WebClient to execute multiple downstream calls **concurrently** with `Mono.zip`, added **timeouts** to prevent thread/resource starvation, **retries with exponential backoff** for transient failures, and **fallbacks** to maintain partial availability. This significantly improved both **latency** (parallel vs serial) and **resilience** of the service."

### One Level Deeper (When Pushed)

| Why? | Answer |
|------|--------|
| Why timeout? | Prevent thread/resource starvation when downstream is slow |
| Why retry? | Handle transient network glitches automatically |
| Why fallback? | Graceful degradation — return partial data instead of failing |
| Why `Mono.zip`? | Parallel execution + aggregation in one operator |
| Why exponential backoff? | Avoid hammering a recovering service |
| Why filter retries on `IOException`? | Don't retry 4xx (client errors) — only network/transient |

### Error Handling on `bodyToMono`

```java
.retrieve()
.onStatus(HttpStatus::is4xxClientError, resp ->
    resp.bodyToMono(ErrorBody.class)
        .flatMap(err -> Mono.error(new ClientException(err.message()))))
.onStatus(HttpStatus::is5xxServerError, resp ->
    Mono.error(new DownstreamException("5xx from upstream")))
.bodyToMono(User.class);
```

### `RestTemplate` vs `WebClient`

| Aspect | `RestTemplate` | `WebClient` |
|--------|----------------|-------------|
| Style | Blocking | Non-blocking |
| Status | Maintenance mode (since Spring 5) | Recommended |
| Use today | Legacy code only | New code |

---

## Microservices Patterns

### Typical Architecture

```
       ┌──────────────┐
       │ API Gateway  │   (Spring Cloud Gateway / Zuul)
       └──────┬───────┘
              │
   ┌──────────┼──────────┐
   ▼          ▼          ▼
[Service A] [Service B] [Service C]
              │
              └──> Eureka (Service Discovery)
              └──> Config Server
              └──> Distributed Tracing (Sleuth/Zipkin)
              └──> Circuit Breaker (Resilience4j)
```

### Key Spring Cloud Components

| Component | Purpose |
|-----------|---------|
| **Eureka Server** | Service registry & discovery |
| **Config Server** | Centralized config (Git-backed) |
| **Spring Cloud Gateway** | API gateway, routing, filtering |
| **Resilience4j** | Circuit breaker, retry, bulkhead, rate limiter |
| **Sleuth / Micrometer Tracing** | Distributed tracing |
| **OpenFeign** | Declarative HTTP client |

---

## Service Discovery (Eureka)

> Note: Netflix has stopped active development of Eureka — many teams use **HashiCorp Consul** or **Kubernetes-native** discovery now. Eureka still appears in interviews and Spring Cloud projects.

### Why Service Discovery?

- Each microservice can run on **multiple instances on different ports**.
- Hard-coded URLs don't scale.
- Eureka acts as a phone book: services register themselves; consumers look up by **service name**.

```
                    Eureka Server
                  /      |       \
              Service1  Service2  Service3
                  ↑       ↑         ↑
                  └── ask Eureka for other service addresses
```

### Set Up Eureka Server

**1. Dependencies:**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-config</artifactId>
</dependency>
```

**2. Main class:**
```java
@SpringBootApplication
@EnableEurekaServer
public class DiscoveryServerApp {
    public static void main(String[] args) { SpringApplication.run(DiscoveryServerApp.class, args); }
}
```

**3. `application.yml`:**
```yaml
spring:
  application:
    name: discovery-service
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```

### Register a Service with Eureka

**1. Dependencies:**
```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

**2. `application.yml`:**
```yaml
spring:
  application:
    name: order-service
  config:
    import: optional:configserver:http://localhost:8888
server:
  port: 8081
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka
```

**3. Discovery via WebClient (load-balanced):**
```java
@Bean
@LoadBalanced
public WebClient.Builder loadBalancedBuilder() {
    return WebClient.builder();
}

// Then call by service name (Eureka resolves to instance):
webClient.get()
    .uri("http://payment-service/api/payments/{id}", id)
    .retrieve();
```

### API Gateway (Spring Cloud Gateway)

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: order-service
          uri: lb://order-service     # lb:// = load-balanced via Eureka
          predicates:
            - Path=/api/orders/**
          filters:
            - StripPrefix=1
```

### Circuit Breaker with Resilience4j

```java
@Service
public class PaymentClient {

    @CircuitBreaker(name = "paymentService", fallbackMethod = "fallback")
    @Retry(name = "paymentService")
    public Payment getPayment(String id) {
        return webClient.get().uri("/payments/{id}", id)
            .retrieve().bodyToMono(Payment.class).block();
    }

    public Payment fallback(String id, Throwable t) {
        return new Payment(id, "FALLBACK", BigDecimal.ZERO);
    }
}
```

```yaml
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 5s
        permitted-number-of-calls-in-half-open-state: 3
```

---

## Security Quick Reference

### Basic Setup

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(csrf -> csrf.disable())
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .oauth2ResourceServer(oauth -> oauth.jwt(Customizer.withDefaults()));
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### Method-Level Security

```java
@PreAuthorize("hasRole('ADMIN')")
public void deleteUser(Long id) { ... }

@PreAuthorize("#userId == authentication.principal.id")
public User updateUser(Long userId, UserDto dto) { ... }
```

---

## Top 50 Interview Q&A

### 🟢 Spring Core

**Q1. What is IoC and DI?**
IoC = container manages object creation/lifecycle. DI = container injects dependencies. Together they enable loose coupling and testability.

**Q2. Constructor vs setter vs field injection — which to use?**
Constructor for required, immutable deps (preferred). Setter for optional. Field injection — avoid (hides deps, hard to test).

**Q3. `@Component` vs `@Service` vs `@Repository` vs `@Controller`.**
All `@Component` variants — Spring scans them as beans. `@Repository` adds DAO exception translation. `@Controller` for MVC. `@Service` is semantic.

**Q4. What does `@SpringBootApplication` do?**
= `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan` (current package).

**Q5. How does auto-configuration work?**
Spring Boot scans `AutoConfiguration.imports`, applies classes guarded by conditional annotations (`@ConditionalOnClass`, `@ConditionalOnMissingBean`, etc.).

**Q6. Bean scopes — what are they?**
`singleton` (default), `prototype`, `request`, `session`, `application`. Singleton = one per container.

**Q7. `@Primary` vs `@Qualifier`.**
Both resolve ambiguity when multiple beans of same type. `@Primary` marks default; `@Qualifier` selects by name.

**Q8. What is `@Lookup` for?**
Inject a fresh prototype bean into a singleton on each method call.

**Q9. What does `@Scheduled` need?**
`@EnableScheduling` on a config class. Then `@Scheduled(fixedDelay=5000)`, `@Scheduled(cron="...")`.

**Q10. How are profiles used?**
`@Profile("dev")` on beans/classes; activate via `spring.profiles.active=prod`.

### 🟢 REST & Validation

**Q11. `@RestController` vs `@Controller`.**
`@RestController` = `@Controller` + `@ResponseBody`; auto-serializes return values to JSON.

**Q12. How do you return a 201 Created with Location header?**
```java
return ResponseEntity.created(URI.create("/users/" + id)).body(user);
```

**Q13. How does `@Valid` work?**
Spring uses Jakarta Bean Validation (Hibernate Validator). On failure throws `MethodArgumentNotValidException` → handle globally.

**Q14. Difference between `@RequestParam` and `@PathVariable`.**
`@PathVariable` binds URI template segment. `@RequestParam` binds query/form param.

**Q15. How do you handle CORS?**
```java
@Bean
WebMvcConfigurer cors() {
    return new WebMvcConfigurer() {
        public void addCorsMappings(CorsRegistry r) {
            r.addMapping("/**").allowedOrigins("https://app.example.com");
        }
    };
}
```

### 🟢 Exception Handling

**Q16. How would you implement global exception handling in Spring Boot?**
Use `@RestControllerAdvice` with `@ExceptionHandler` methods returning `ResponseEntity<ErrorResponse>`. Map domain exceptions to HTTP statuses, return a consistent error shape.

**Q17. How do you handle validation errors specifically?**
`@ExceptionHandler(MethodArgumentNotValidException.class)`, extract `BindingResult.getFieldErrors()`, build a 400 response with field-level details.

**Q18. `@ControllerAdvice` vs `@RestControllerAdvice`.**
`@RestControllerAdvice` adds `@ResponseBody` so handler return values are serialized to JSON.

**Q19. How do you handle WebClient errors?**
Use `.onStatus(...)` or `onErrorResume`. For global handling, throw a typed exception (`DownstreamException`) and handle it in `@RestControllerAdvice`.

**Q20. Where do you log exceptions?**
- Expected exceptions (4xx) — `WARN` with context (no stack trace).
- Unexpected (5xx) — `ERROR` with full stack trace.
- Always include trace/correlation ID (MDC).

### 🟢 JPA & Hibernate

**Q21. `EAGER` vs `LAZY` fetch — which default?**
Default `LAZY` for `@OneToMany`/`@ManyToMany`, default `EAGER` for `@ManyToOne`/`@OneToOne`. Always **prefer LAZY** explicitly.

**Q22. What is the N+1 problem?**
Loading 1 parent then N children one-by-one. Solution: `JOIN FETCH`, `@EntityGraph`, or `@BatchSize`.

**Q23. `save()` vs `saveAndFlush()`.**
`save()` schedules write — flushed at transaction end. `saveAndFlush()` flushes immediately.

**Q24. JPA inheritance strategies?**
`SINGLE_TABLE` (one table + discriminator), `JOINED` (parent + child tables linked), `TABLE_PER_CLASS` (one table per concrete class).

**Q25. What is `@DiscriminatorColumn` / `@DiscriminatorValue`?**
For `SINGLE_TABLE`: column tells JPA which subclass each row maps to. `@DiscriminatorValue("T")` on a subclass sets its discriminator.

**Q26. Optimistic vs pessimistic locking?**
- **Optimistic** — `@Version` field, retry on conflict. Best for low-contention.
- **Pessimistic** — `SELECT ... FOR UPDATE`. Best for hot rows.

**Q27. `@Transactional` propagation — when use `REQUIRES_NEW`?**
When you need a child operation to commit/rollback independently — e.g., audit logging that must persist even if parent rolls back.

**Q28. Why doesn't `@Transactional` work on self-invocation?**
`@Transactional` works via Spring proxy. `this.foo()` bypasses the proxy. Inject the bean into itself or restructure.

**Q29. What happens when an unchecked exception is thrown in a `@Transactional` method?**
Rollback by default. Checked exceptions don't trigger rollback unless `rollbackFor` is specified.

**Q30. How do you avoid Lazy initialization exceptions?**
- Fetch eagerly in repo query (`JOIN FETCH`).
- Use `@EntityGraph`.
- Open Session in View (anti-pattern, avoid).
- Map to DTO inside transaction.

### 🟢 WebFlux & WebClient

**Q31. Mono vs Flux.**
`Mono<T>` = 0 or 1 element. `Flux<T>` = 0..N elements. Both are publishers in Project Reactor.

**Q32. WebClient vs RestTemplate.**
WebClient is non-blocking, supports reactive streams. RestTemplate is blocking and in maintenance.

**Q33. How do you call multiple downstream services in parallel with WebClient?**
```java
Mono.zip(getUser(id), getAccount(id), getProfile(id))
    .map(t -> new Combined(t.getT1(), t.getT2(), t.getT3()));
```

**Q34. How do you add retry with backoff?**
```java
.retryWhen(Retry.backoff(2, Duration.ofMillis(200))
    .filter(ex -> ex instanceof IOException))
```

**Q35. Difference between `subscribeOn` and `publishOn`?**
`subscribeOn` — sets the scheduler for the source (where data is produced).
`publishOn` — switches the scheduler for downstream operators.

**Q36. What is backpressure?**
Mechanism where consumer signals demand to producer, preventing overflow. Built into Reactive Streams `request(n)`.

**Q37. WebFlux vs virtual threads (Java 21) — which to use?**
- **WebFlux** — streaming, SSE, backpressure, true reactive composition.
- **Virtual threads** — simpler imperative code, similar throughput for typical request/response I/O. Many teams prefer virtual threads for simplicity.

**Q38. How do you test reactive code?**
```java
StepVerifier.create(myFlux)
    .expectNext("a", "b")
    .expectComplete()
    .verify();
```

### 🟢 Microservices

**Q39. What does an API Gateway do?**
Routing, auth, rate limiting, request/response transformation, SSL termination. Single entry point.

**Q40. What is service discovery?**
Services register with a registry (Eureka, Consul). Consumers look up addresses by service name instead of hardcoding URLs.

**Q41. Circuit breaker — how does it work?**
States: **CLOSED** (normal) → **OPEN** (failing, short-circuit) → **HALF_OPEN** (test if recovered) → CLOSED. Implemented by Resilience4j.

**Q42. How do you propagate trace IDs across services?**
Use Micrometer Tracing / Sleuth — auto-injects `traceId`/`spanId` into headers (`B3`, `W3C Trace Context`) and MDC.

**Q43. How do you centralize config?**
Spring Cloud Config Server backed by Git. Services pull config at startup with `spring.config.import: configserver:...`.

**Q44. Synchronous vs asynchronous communication between services?**
- **Sync** (REST/gRPC) — simpler, but tightly coupled, latency cascades.
- **Async** (Kafka, RabbitMQ) — decoupled, resilient, eventual consistency.

**Q45. SAGA pattern — what is it?**
Distributed transaction pattern: each service performs its local transaction and publishes an event. Compensating actions roll back on failure. Two flavors: **choreography** (events) or **orchestration** (central coordinator).

### 🟢 Production & Operations

**Q46. How do you health-check a Spring Boot app?**
Spring Boot Actuator → `/actuator/health`. Customize with `HealthIndicator` beans for downstream checks.

**Q47. How do you monitor metrics?**
Micrometer (built-in) → Prometheus → Grafana. Custom metrics via `MeterRegistry.counter("orders.placed").increment()`.

**Q48. How do you handle graceful shutdown?**
```yaml
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s
```
This stops accepting new requests, completes in-flight ones, then shuts down.

**Q49. How do you secure secrets in Spring Boot?**
- Don't commit `application.yml` with secrets.
- Use environment variables, AWS Secrets Manager, HashiCorp Vault, or Spring Cloud Config with encryption.
- For dev: `.env` files (gitignored).

**Q50. Caching in Spring — how?**
```java
@EnableCaching
class Config {}

@Service
class UserService {
    @Cacheable("users")
    public User findById(Long id) { ... }

    @CacheEvict(value = "users", key = "#user.id")
    public void update(User user) { ... }
}
```
Backed by Caffeine, Redis, Ehcache.

---

## 🎯 Final Tips

- **Always tie answers to production impact.** "We added timeouts → reduced p99 latency from 5s to 800ms."
- **Trade-offs first.** WebFlux vs virtual threads, optimistic vs pessimistic locking, sync vs async — show you've thought about it.
- **Resilience patterns are interview gold.** Timeout + retry + circuit breaker + fallback + bulkhead.
- **Know your `@Transactional` traps** — self-invocation, private methods, checked exceptions.
- **Be ready to draw a microservice architecture** on a whiteboard with API gateway, service registry, config server, tracing, and circuit breakers labeled.

---

*Notes compiled from personal handwritten study material + interview prep additions. Last updated April 2026.*
