← [Back to Index](../README.md)

# REST API Design Best Practices for Senior Developers

A comprehensive architectural and implementation guide to building robust, enterprise-grade RESTful APIs with Spring Boot 3.x and Java 17+. This guide covers RESTful foundations, versioning strategies, pagination performance, RFC 7807 error modeling, validation frameworks, HATEOAS, OpenAPI documentation, distributed rate limiting, idempotency keys, and DTO/entity separation via MapStruct.

---

## 1. RESTful Principles & Architecture

REST (Representational State Transfer) is an architectural style based on stateless, client-server communication using standard HTTP semantics.

### 1.1 The Richardson Maturity Model

```
Level 3: HATEOAS (Hypermedia As The Engine Of Application State)
         Resources emit dynamic hypermedia controls/links for next valid actions.
                             ▲
Level 2: HTTP Verbs & Status Codes
         Correct use of GET, POST, PUT, PATCH, DELETE and standard HTTP status codes.
                             ▲
Level 1: Resources
         Individual URIs represent distinct business nouns (/orders, /customers/42).
                             ▲
Level 0: Swamp of POX (Plain Old XML/JSON)
         Single URI (/api/service), single verb (POST), RPC-style payloads.
```

### 1.2 HTTP Methods: Safety & Idempotency Matrix

A crucial senior interview topic is the precise definition of **Safety** and **Idempotence**:
- **Safe**: Calling the method does not alter resource state on the server (read-only).
- **Idempotent**: Making $N > 0$ identical requests produces the exact same server side-effects as a single request ($f(f(x)) = f(x)$).

| HTTP Method | Safe? | Idempotent? | RFC Semantics & Usage |
| :--- | :--- | :--- | :--- |
| **GET** | **Yes** | **Yes** | Retrieve resource representation without side effects. |
| **HEAD** | **Yes** | **Yes** | Identical to GET but omits response body; used to check headers or resource existence. |
| **OPTIONS** | **Yes** | **Yes** | Discovers supported communication options (e.g., CORS preflights). |
| **POST** | **No** | **No** | Creates a new sub-resource or executes a non-idempotent operation (e.g., checkout). |
| **PUT** | **No** | **Yes** | Complete replacement of the target resource with payload (or creation at a client-chosen URI). |
| **PATCH** | **No** | **No\*** | Partial update of resource attributes. (\*RFC 5789 permits non-idempotent semantics, e.g., JSON Patch operations like `add`). |
| **DELETE** | **No** | **Yes** | Removes the resource. Deleting a deleted resource returns 404 or 204, but server state remains deleted. |

### 1.3 Resource Naming Conventions
- **Use Nouns, Not Verbs**: Resources represent domain entities, not remote procedure calls.
  - Good: `POST /api/v1/orders`
  - Bad: `POST /api/v1/createOrder`
- **Plural Naming**: Use plural collections for uniformity.
  - Good: `/api/v1/customers/102/orders`
  - Bad: `/api/v1/customer/102/order`
- **Kebab-Case URIs**: Standard in modern web architecture (`/api/v1/order-items`).
- **Hierarchical Relationships**:
  - `/api/v1/departments/{deptId}/employees/{empId}` (indicates ownership/scoping).
  - If a resource can exist independently, keep it top-level: `/api/v1/employees/{empId}`.

### 1.4 HTTP Status Codes: Practical Selection

| Code | Status | When to Use |
| :--- | :--- | :--- |
| **200** | `OK` | Successful GET, PUT, or PATCH returning payload. |
| **201** | `Created` | Successful POST creating a new resource. Must include `Location` header. |
| **202** | `Accepted` | Asynchronous processing accepted (e.g., batch jobs, message queues). |
| **204** | `No Content` | Successful DELETE or PUT returning no payload. |
| **304** | `Not Modified` | Conditional GET via `If-None-Match` (ETag) or `If-Modified-Since`. |
| **400** | `Bad Request` | Malformed JSON syntax, type mismatch, unparseable input. |
| **401** | `Unauthorized` | Authentication missing or invalid (no valid token/credentials). |
| **403** | `Forbidden` | Authenticated, but lacking permission/role to access resource. |
| **404** | `Not Found` | Target resource URI does not exist. |
| **409** | `Conflict` | State conflict (e.g., duplicate unique key, optimistic locking failure). |
| **422** | `Unprocessable Entity` | Syntactically valid JSON, but business validation failed. |
| **429** | `Too Many Requests` | Rate limit quota exceeded. |
| **500** | `Internal Server Error` | Unexpected server-side bug or unhandled exception. |
| **503** | `Service Unavailable` | Downstream dependency down, circuit breaker open, or maintenance mode. |

---

## 2. API Versioning Strategies

No public enterprise API survives without versioning. Four strategies are common:

```
1. URI Path Versioning:
   GET https://api.example.com/v1/orders
   GET https://api.example.com/v2/orders

2. Request Header Versioning:
   GET /orders HTTP/1.1
   X-API-VERSION: 2

3. Content Negotiation (Media Type / Accept Header):
   GET /orders HTTP/1.1
   Accept: application/vnd.company.v2+json

4. Query Parameter:
   GET /orders?version=2
```

### 2.1 Comparison of Versioning Strategies

| Strategy | Pros | Cons | Recommended Use Case |
| :--- | :--- | :--- | :--- |
| **URI Path** | Simple, highly visible, cache-friendly for CDNs, easily testable in browsers. | Violates pure REST URI permanence (resource identity changes with version). | **Default enterprise choice** for public APIs (Stripe, GitHub, AWS). |
| **Header** | Clean URIs, separates resource locator from routing metadata. | Harder to test in browser address bar, requires CDN cache variation configuration (`Vary`). | Internal microservice architectures. |
| **Media Type** | True REST compliant (HATEOAS), fine-grained representation control. | Complex client configurations, poor browser testability. | Enterprise architectures with strict REST adherence. |
| **Query Param** | Easy to default if omitted, browser-friendly. | Complicates query caching, mixes filtering with versioning. | Quick prototypes and legacy systems. |

### 2.2 Spring Boot Implementation for All 4 Strategies

```java
package com.example.controller;

import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/api")
public class VersioningController {

    // 1. URI Path Versioning
    @GetMapping("/v1/products")
    public String getProductV1Path() {
        return "Products V1 (URI Path)";
    }

    @GetMapping("/v2/products")
    public String getProductV2Path() {
        return "Products V2 (URI Path)";
    }

    // 2. Request Header Versioning
    @GetMapping(value = "/products", headers = "X-API-VERSION=1")
    public String getProductV1Header() {
        return "Products V1 (Custom Header)";
    }

    @GetMapping(value = "/products", headers = "X-API-VERSION=2")
    public String getProductV2Header() {
        return "Products V2 (Custom Header)";
    }

    // 3. Content Negotiation / Media Type Versioning
    @GetMapping(value = "/products", produces = "application/vnd.company.v1+json")
    public String getProductV1Produces() {
        return "Products V1 (Accept Header Content Negotiation)";
    }

    @GetMapping(value = "/products", produces = "application/vnd.company.v2+json")
    public String getProductV2Produces() {
        return "Products V2 (Accept Header Content Negotiation)";
    }

    // 4. Query Parameter Versioning
    @GetMapping(value = "/products", params = "version=1")
    public String getProductV1Param() {
        return "Products V1 (Query Param)";
    }

    @GetMapping(value = "/products", params = "version=2")
    public String getProductV2Param() {
        return "Products V2 (Query Param)";
    }
}
```

---

## 3. High-Performance Pagination & Sorting

### 3.1 Offset vs Cursor-Based Pagination

```
Offset-Based Pagination:
Client requests: GET /orders?page=10000&size=20
SQL: SELECT * FROM orders ORDER BY id LIMIT 20 OFFSET 200000;
      └─ Database MUST scan & discard 200,000 index entries! -> O(N) performance degradation!

Cursor-Based (Keyset) Pagination:
Client requests: GET /orders?after_id=200000&limit=20
SQL: SELECT * FROM orders WHERE id > 200000 ORDER BY id ASC LIMIT 20;
      └─ Direct B-Tree index seek! -> O(log N) constant time, regardless of depth!
```

### 3.2 Spring Data: `Page<T>` vs `Slice<T>` vs `List<T>`

A common interview trap:
- `Page<T>`: Triggers an **extra `SELECT COUNT(*)`** query to compute total pages. On large tables (millions of rows), this `COUNT(*)` kills database performance.
- `Slice<T>`: Fetches `size + 1` rows to know if a next page exists without ever running `COUNT(*)`. Perfect for infinite scrolling.
- `List<T>`: Pure data fetch with `LIMIT` only. Lowest database overhead.

```java
public interface OrderRepository extends JpaRepository<Order, Long> {

    // Heavy: Runs COUNT(*) query every time
    Page<Order> findByCustomerId(Long customerId, Pageable pageable);

    // Optimized: Fetches size + 1 records, NO COUNT(*) query
    Slice<Order> findSliceByCustomerId(Long customerId, Pageable pageable);

    // Keyset pagination (Cursor-based)
    @Query("SELECT o FROM Order o WHERE o.id > :afterId ORDER BY o.id ASC")
    List<Order> findNextBatch(@Param("afterId") Long afterId, Pageable pageable);
}
```

---

## 4. RFC 7807 Problem Details & Global Error Handling

Spring Boot 3 natively adopts **RFC 7807** (`application/problem+json`), standardizing error payloads across the internet.

### 4.1 RFC 7807 Specification Fields
- `type`: URI reference identifying the problem type.
- `title`: Short, human-readable summary of problem.
- `status`: HTTP status code.
- `detail`: Human-readable explanation specific to this occurrence.
- `instance`: URI reference identifying the specific occurrence (request path).
- `properties`: Custom extension attributes (e.g., field validation errors, timestamps, traceId).

### 4.2 Production-Grade `@RestControllerAdvice` Implementation

```java
package com.example.exception;

import org.springframework.http.HttpStatus;
import org.springframework.http.ProblemDetail;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.net.URI;
import java.time.Instant;
import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ProblemDetail handleResourceNotFound(ResourceNotFoundException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(HttpStatus.NOT_FOUND, ex.getMessage());
        problem.setType(URI.create("https://api.company.com/errors/not-found"));
        problem.setTitle("Resource Not Found");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ProblemDetail handleValidationErrors(MethodArgumentNotValidException ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.BAD_REQUEST, "One or more request parameters failed validation.");
        problem.setType(URI.create("https://api.company.com/errors/validation-failed"));
        problem.setTitle("Validation Failure");
        problem.setProperty("timestamp", Instant.now());

        Map<String, String> fieldErrors = new HashMap<>();
        for (FieldError error : ex.getBindingResult().getFieldErrors()) {
            fieldErrors.put(error.getField(), error.getDefaultMessage());
        }
        problem.setProperty("invalidFields", fieldErrors);

        return problem;
    }

    @ExceptionHandler(Exception.class)
    public ProblemDetail handleGeneralException(Exception ex) {
        ProblemDetail problem = ProblemDetail.forStatusAndDetail(
                HttpStatus.INTERNAL_SERVER_ERROR, "An unexpected internal system error occurred.");
        problem.setType(URI.create("https://api.company.com/errors/internal-error"));
        problem.setTitle("Internal Server Error");
        problem.setProperty("timestamp", Instant.now());
        return problem;
    }
}
```

Enable native Problem Details in `application.yml`:
```yaml
spring:
  mvc:
    problemdetails:
      enabled: true
```

---

## 5. Input Validation & Custom Constraints

### 5.1 `@Valid` vs `@Validated`
- **`@Valid`** (Standard Jakarta Bean Validation `jakarta.validation.Valid`):
  - Supported at method parameter level.
  - Required on nested object properties to trigger recursive cascading validation (`@Valid private Address address`).
  - Does not support validation groups.
- **`@Validated`** (Spring-specific `org.springframework.validation.annotation.Validated`):
  - Can be placed at the **class level** on `@Service` or `@RestController` to validate method arguments (`@Min`, `@NotNull` directly on parameters).
  - Supports **Validation Groups**.

### 5.2 Validation Groups Implementation

Validation groups allow applying different validation rules to the same DTO class during different lifecycle phases (e.g., Create vs Update).

```java
package com.example.dto;

public interface ValidationGroups {
    interface OnCreate {}
    interface OnUpdate {}
}
```

```java
package com.example.dto;

import jakarta.validation.constraints.Email;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import jakarta.validation.constraints.Null;

public record UserDto(
        @Null(groups = ValidationGroups.OnCreate.class, message = "ID must be null when creating")
        @NotNull(groups = ValidationGroups.OnUpdate.class, message = "ID is required when updating")
        Long id,

        @NotBlank(groups = {ValidationGroups.OnCreate.class, ValidationGroups.OnUpdate.class})
        String fullName,

        @NotBlank(groups = ValidationGroups.OnCreate.class)
        @Email(groups = {ValidationGroups.OnCreate.class, ValidationGroups.OnUpdate.class})
        String email
) {}
```

Controller usage:
```java
@PostMapping
public ResponseEntity<Void> createUser(
        @Validated(ValidationGroups.OnCreate.class) @RequestBody UserDto dto) {
    // ID validated as null, email and fullName validated
    return ResponseEntity.status(HttpStatus.CREATED).build();
}

@PutMapping("/{id}")
public ResponseEntity<Void> updateUser(
        @PathVariable Long id,
        @Validated(ValidationGroups.OnUpdate.class) @RequestBody UserDto dto) {
    // ID must be present
    return ResponseEntity.ok().build();
}
```

### 5.3 Custom Constraint Validator (e.g., Phone Number Validation)

```java
package com.example.validation;

import jakarta.validation.Constraint;
import jakarta.validation.Payload;
import java.lang.annotation.*;

@Documented
@Constraint(validatedBy = PhoneNumberValidator.class)
@Target({ElementType.FIELD, ElementType.PARAMETER})
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidPhoneNumber {
    String message() default "Invalid international phone number format";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}
```

```java
package com.example.validation;

import jakarta.validation.ConstraintValidator;
import jakarta.validation.ConstraintValidatorContext;

public class PhoneNumberValidator implements ConstraintValidator<ValidPhoneNumber, String> {

    private static final String E164_REGEX = "^\\+[1-9]\\d{1,14}$";

    @Override
    public boolean isValid(String value, ConstraintValidatorContext context) {
        if (value == null) {
            return true; // Use @NotNull separately if required
        }
        return value.matches(E164_REGEX);
    }
}
```

---

## 6. HATEOAS (Hypermedia As The Engine Of Application State)

Spring HATEOAS enables returning hypermedia links that tell the client what state transitions are valid from the current state.

```java
package com.example.controller;

import com.example.model.Order;
import com.example.model.OrderStatus;
import org.springframework.hateoas.EntityModel;
import org.springframework.hateoas.IanaLinkRelations;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import static org.springframework.hateoas.server.mvc.WebMvcLinkBuilder.*;

@RestController
@RequestMapping("/api/v1/orders")
public class OrderHateoasController {

    @GetMapping("/{id}")
    public ResponseEntity<EntityModel<Order>> getOrderById(@PathVariable Long id) {
        Order order = findOrder(id);

        EntityModel<Order> orderModel = EntityModel.of(order,
                linkTo(methodOn(OrderHateoasController.class).getOrderById(id)).withSelfRel(),
                linkTo(methodOn(OrderHateoasController.class).getAllOrders()).withRel(IanaLinkRelations.COLLECTION)
        );

        // State machine driven dynamic link emission
        if (order.getStatus() == OrderStatus.PENDING_PAYMENT) {
            orderModel.add(linkTo(methodOn(OrderHateoasController.class).cancelOrder(id)).withRel("cancel"));
            orderModel.add(linkTo(methodOn(OrderHateoasController.class).processPayment(id)).withRel("payment"));
        } else if (order.getStatus() == OrderStatus.PAID) {
            orderModel.add(linkTo(methodOn(OrderHateoasController.class).shipOrder(id)).withRel("ship"));
        }

        return ResponseEntity.ok(orderModel);
    }

    // Dummy helper endpoints for methodOn reference
    @GetMapping public ResponseEntity<?> getAllOrders() { return null; }
    @PutMapping("/{id}/cancel") public ResponseEntity<?> cancelOrder(@PathVariable Long id) { return null; }
    @PostMapping("/{id}/payment") public ResponseEntity<?> processPayment(@PathVariable Long id) { return null; }
    @PostMapping("/{id}/ship") public ResponseEntity<?> shipOrder(@PathVariable Long id) { return null; }
    private Order findOrder(Long id) { return new Order(id, OrderStatus.PENDING_PAYMENT, 99.99); }
}
```

JSON response received by client:
```json
{
  "id": 101,
  "status": "PENDING_PAYMENT",
  "amount": 99.99,
  "_links": {
    "self": { "href": "http://localhost:8080/api/v1/orders/101" },
    "collection": { "href": "http://localhost:8080/api/v1/orders" },
    "cancel": { "href": "http://localhost:8080/api/v1/orders/101/cancel" },
    "payment": { "href": "http://localhost:8080/api/v1/orders/101/payment" }
  }
}
```

---

## 7. Distributed Rate Limiting (Bucket4j / Redis)

Rate limiting protects APIs against denial-of-service, brute force, and runaway client loops.

### 7.1 The Token Bucket Algorithm
- A bucket holds a maximum capacity of tokens $C$.
- Tokens refill at a steady rate $r$ tokens/second.
- Each incoming request consumes 1 token.
- If tokens $> 0$, request proceeds. If tokens $= 0$, HTTP 429 Too Many Requests is returned.

### 7.2 Rate Limiter Filter Implementation

```java
package com.example.filter;

import io.github.bucket4j.Bandwidth;
import io.github.bucket4j.Bucket;
import io.github.bucket4j.Refill;
import jakarta.servlet.*;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.http.HttpStatus;
import org.springframework.stereotype.Component;

import java.io.IOException;
import java.time.Duration;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class RateLimitingFilter implements Filter {

    // In production, back this with Redis via bucket4j-redis
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    private Bucket createNewBucket() {
        // 50 requests per minute with refill capacity
        Bandwidth limit = Bandwidth.classic(50, Refill.greedy(50, Duration.ofMinutes(1)));
        return Bucket.builder().addLimit(limit).build();
    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        HttpServletRequest httpRequest = (HttpServletRequest) request;
        HttpServletResponse httpResponse = (HttpServletResponse) response;

        // Partition limits by API key or IP address
        String apiKey = httpRequest.getHeader("X-API-KEY");
        String key = (apiKey != null) ? apiKey : httpRequest.getRemoteAddr();

        Bucket bucket = buckets.computeIfAbsent(key, k -> createNewBucket());

        if (bucket.tryConsume(1)) {
            // Standard rate limit response headers
            httpResponse.setHeader("X-RateLimit-Limit", "50");
            httpResponse.setHeader("X-RateLimit-Remaining", String.valueOf(bucket.getAvailableTokens()));
            chain.doFilter(request, response);
        } else {
            httpResponse.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            httpResponse.setHeader("Retry-After", "60");
            httpResponse.setContentType("application/json");
            httpResponse.getWriter().write("{\"error\": \"Too many requests. Please wait before retrying.\"}");
        }
    }
}
```

---

## 8. Idempotency Key Pattern for Safe Retries

When a network timeout occurs during a payment creation `POST /api/v1/payments`, the client cannot tell whether the server processed the transaction or never received it.

```
Client                                                  Server / Redis
  │                                                           │
  ├─── POST /payments (Header: Idempotency-Key: abc-123) ────>│
  │                                                           ├── Look up key "idemp:abc-123"
  │                                                           ├── Key NOT found: Set state = IN_PROGRESS (TTL 120s)
  │                                                           ├── Execute Payment DB Transaction
  │                                                           ├── Store Result in Redis & set state = COMPLETED
  │<── 201 Created (Payment Confirmation) ────────────────────┤
  │                                                           │
 (Network Drops / Client Retries with SAME Idempotency-Key):  │
  ├─── POST /payments (Header: Idempotency-Key: abc-123) ────>│
  │                                                           ├── Look up key "idemp:abc-123"
  │                                                           ├── Key EXISTS with state COMPLETED
  │<── 201 Created (Cached Payment Confirmation from Redis) ──┤ (NO duplicate DB insert!)
```

### Key Implementation Guidelines:
1. Client generates UUIDv4 for each unique business action.
2. Server validates `Idempotency-Key` header via a Servlet Filter or Spring Interceptor.
3. Uses Redis distributed lock (`SET NX PX`) to guard concurrent in-flight requests.
4. Caches the exact HTTP response status code and JSON body for replay.

---

## 9. DTOs vs JPA Entities & MapStruct Integration

### 9.1 Why Never Expose JPA `@Entity` in REST Endpoints

1. **Security (Mass Assignment Vulnerability)**: A client sending `{"id": 10, "isAdmin": true, "balance": 999999}` can bind directly to internal entity fields if bound directly.
2. **Circular Reference & Serialization Errors**: Bidirectional relationships (`@OneToMany` and `@ManyToOne`) cause Jackson `StackOverflowError` or lazy-loading `LazyInitializationException` outside transactions.
3. **Database Schema Leaks**: Any database column change directly breaks API contracts.
4. **Over-fetching**: Returns unnecessary sensitive columns (passwords, internal audit metadata).

### 9.2 High-Performance DTO Mapping with MapStruct

Unlike runtime reflection-based mappers (ModelMapper), **MapStruct** generates compile-time, zero-overhead Java bytecode.

#### Entity & DTO:
```java
package com.example.model;

import jakarta.persistence.*;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Entity
@Table(name = "customers")
public class CustomerEntity {
    @Id @GeneratedValue private Long id;
    private String firstName;
    private String lastName;
    private String email;
    private String passwordHash;
    private BigDecimal accountBalance;
    private LocalDateTime createdAt;
    // Getters and Setters
}
```

```java
package com.example.dto;

import java.math.BigDecimal;

public record CustomerResponseDto(
        Long id,
        String fullName,
        String email,
        BigDecimal balance
) {}
```

#### MapStruct Mapper Interface:
```java
package com.example.mapper;

import com.example.dto.CustomerResponseDto;
import com.example.model.CustomerEntity;
import org.mapstruct.Mapper;
import org.mapstruct.Mapping;
import org.mapstruct.Named;

@Mapper(componentModel = "spring")
public interface CustomerMapper {

    @Mapping(target = "fullName", expression = "java(entity.getFirstName() + ' ' + entity.getLastName())")
    @Mapping(source = "accountBalance", target = "balance")
    CustomerResponseDto toDto(CustomerEntity entity);
}
```

MapStruct generates pure Java code at compile time:
```java
// Target generated code:
public CustomerResponseDto toDto(CustomerEntity entity) {
    if (entity == null) return null;
    Long id = entity.getId();
    String fullName = entity.getFirstName() + " " + entity.getLastName();
    String email = entity.getEmail();
    BigDecimal balance = entity.getAccountBalance();
    return new CustomerResponseDto(id, fullName, email, balance);
}
```

---

## 10. API Documentation: SpringDoc OpenAPI 3

SpringDoc OpenAPI dynamically generates OpenAPI 3.0 specifications and Swagger UI for Spring Boot 3 applications.

### 10.1 Dependency
```xml
<dependency>
    <groupId>org.springdoc</groupId>
    <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
    <version>2.5.0</version>
</dependency>
```

### 10.2 Production OpenAPI Configuration & Endpoint Annotations

```java
package com.example.config;

import io.swagger.v3.oas.models.OpenAPI;
import io.swagger.v3.oas.models.info.Contact;
import io.swagger.v3.oas.models.info.Info;
import io.swagger.v3.oas.models.info.License;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
                .info(new Info()
                        .title("Enterprise Order Management API")
                        .version("v1.0.0")
                        .description("High-throughput microservice for processing and tracking global orders.")
                        .contact(new Contact().name("API Platform Team").email("api-support@company.com"))
                        .license(new License().name("Apache 2.0").url("https://springdoc.org")));
    }
}
```

```java
package com.example.controller;

import com.example.dto.CustomerResponseDto;
import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.Parameter;
import io.swagger.v3.oas.annotations.media.Content;
import io.swagger.v3.oas.annotations.media.Schema;
import io.swagger.v3.oas.annotations.responses.ApiResponse;
import io.swagger.v3.oas.annotations.responses.ApiResponses;
import io.swagger.v3.oas.annotations.tags.Tag;
import org.springframework.http.ProblemDetail;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/v1/customers")
@Tag(name = "Customer Management", description = "Endpoints for creating, retrieving, and updating customer profiles")
public class CustomerController {

    @Operation(summary = "Get customer profile by unique ID", description = "Retrieves customer profile and account balance information.")
    @ApiResponses(value = {
            @ApiResponse(responseCode = "200", description = "Customer located successfully",
                    content = @Content(schema = @Schema(implementation = CustomerResponseDto.class))),
            @ApiResponse(responseCode = "404", description = "Customer not found",
                    content = @Content(schema = @Schema(implementation = ProblemDetail.class))),
            @ApiResponse(responseCode = "429", description = "Rate limit exceeded")
    })
    @GetMapping("/{id}")
    public ResponseEntity<CustomerResponseDto> getCustomer(
            @Parameter(description = "Customer ID", example = "42", required = true)
            @PathVariable Long id) {
        // Business logic
        return ResponseEntity.ok(new CustomerResponseDto(id, "Jane Doe", "jane@company.com", null));
    }
}
```

---

## 11. Common Interview Questions & Answers

### Q1: What is the semantic difference between `PUT` and `PATCH`, and how should partial updates be handled?
**Answer**:
- **`PUT`**: Represents a complete resource replacement. The client sends a representation of the entire resource. Missing fields are considered cleared or set to defaults. It is strictly idempotent.
- **`PATCH`**: Represents a partial update. Only the fields intended to change are submitted.
- **Handling Partial Updates in Spring Boot**:
  1. *Merge Patch (RFC 7396)*: Submit a partial JSON object. Null fields mean "no update", or an explicit `JsonNullable<T>` is used to distinguish between "omitted" vs "set to null".
  2. *JSON Patch (RFC 6902)*: Submit an array of mutation instructions (`[{"op": "replace", "path": "/email", "value": "new@mail.com"}]`). Spring MVC supports this using `JsonPatch`.

### Q2: Why is `Page<T>` dangerous on large enterprise tables in Spring Data JPA?
**Answer**:
`Page<T>` automatically issues a secondary `COUNT(*)` query (`SELECT count(id) FROM table WHERE ...`) alongside the page limit query. On a table with tens of millions of rows, `COUNT(*)` forces a massive index or table scan that often takes seconds, saturating database IOPS.
**Alternative**: Use `Slice<T>`, which requests `PageRequest.of(page, size)` and issues `LIMIT size + 1` behind the scenes. If $size + 1$ records are returned, `hasNext()` is true. No `COUNT(*)` is executed. Alternatively, implement cursor-based (keyset) pagination using `WHERE id > :lastSeenId ORDER BY id ASC LIMIT :size`.

### Q3: How do you achieve content negotiation in Spring Boot?
**Answer**:
Spring Boot evaluates the client's `Accept` request header against registered `HttpMessageConverter`s:
1. `MappingJackson2HttpMessageConverter` handles `application/json`.
2. `MappingJackson2XmlHttpMessageConverter` handles `application/xml` (if Jackson XML is on classpath).
Spring inspects the `@GetMapping(produces = ...)` or `@RequestMapping(produces = ...)` attributes to match supported media types, returning HTTP 406 Not Acceptable if no converter can satisfy the client's requested format.

### Q4: How do you prevent Mass Assignment / Parameter Tampering vulnerabilities in REST controllers?
**Answer**:
Never accept raw domain or JPA entities directly in `@RequestBody` parameters!
Always use dedicated request DTO records containing only the exact fields permitted for mutation by that specific user role. Use Jakarta Validation annotations (`@NotNull`, `@Pattern`, `@Size`) and MapStruct to project vetted DTO data onto the domain entities within service boundaries.

---

## 12. Gotchas & Production Pitfalls

1. **Returning HTTP 200 OK for Errors with a Status Field**:
   - *Anti-Pattern*: Returning `HTTP 200 OK` with body `{"status": "FAILURE", "errorMessage": "Invalid account"}`.
   - *Impact*: Breaks HTTP caching proxies, API gateways, circuit breakers, and monitoring dashboards that rely on standard HTTP 4xx/5xx status codes.

2. **Exposing Internal Auto-Increment IDs**:
   - Exposing sequential numerical IDs (`/api/v1/orders/1`, `/api/v1/orders/2`) allows attackers to enumerate the entire database or scrape competitive business volumes (Insecure Direct Object Reference - IDOR). Use UUIDv7 or NanoIDs for public-facing identifiers.

3. **Missing ETag / Conditional Updates (Lost Update Problem)**:
   - When two clients read a resource and submit concurrent `PUT` requests, the last write overwrites the first without warning.
   - *Fix*: Emit `ETag` headers on GET responses and require `If-Match: <etag>` on PUT/PATCH requests (HTTP 412 Precondition Failed if stale).

4. **Ignoring HTTP Method Semantics in Caching**:
   - Placing mutating logic inside a `GET` request violates the safety contract of HTTP. CDNs, browsers, and web crawlers pre-fetch GET requests, potentially corrupting server data.
