← [Back to Index](../README.md)

# Spring Security for Senior Developers

An in-depth, production-oriented architectural guide to Spring Security (Spring Security 6.x / Spring Boot 3.x) for senior Java backend engineers. This document breaks down the filter chain internals, authentication and authorization pipelines, JWT implementations, OAuth2 flows, modern security changes, common pitfalls, and interview defense strategies.

---

## 1. Security Filter Chain Architecture

Spring Security operates entirely within the standard Servlet `Filter` pipeline. It bridges the gap between the Servlet container lifecycle and the Spring `ApplicationContext`.

### 1.1 The Bridge: `DelegatingFilterProxy` & `FilterChainProxy`

```
   HTTP Servlet Request
            │
            ▼
┌────────────────────────────────────────────────────────┐
│             Standard Servlet Filter Chain              │
│                                                        │
│  [LoggingFilter] -> [DelegatingFilterProxy] -> ...     │
└───────────────────────────┬────────────────────────────┘
                            │ Delegates to bean: "springSecurityFilterChain"
                            ▼
┌────────────────────────────────────────────────────────┐
│                   FilterChainProxy                     │
│                                                        │
│ Matches URI against list of SecurityFilterChain beans: │
│                                                        │
│ ┌────────────────────────────────────────────────────┐ │
│ │ SecurityFilterChain 1: /api/public/** (PermitAll)  │ │
│ └────────────────────────────────────────────────────┘ │
│ ┌────────────────────────────────────────────────────┐ │
│ │ SecurityFilterChain 2: /api/v1/** (JWT Filter Chain)│ │
│ │  1. SecurityContextHolderFilter                    │ │
│ │  2. CorsFilter                                     │ │
│ │  3. CsrfFilter (Disabled for Stateless API)        │ │
│ │  4. JwtAuthenticationFilter (Custom)               │ │
│ │  5. UsernamePasswordAuthenticationFilter           │ │
│ │  6. ExceptionTranslationFilter                     │ │
│ │  7. AuthorizationFilter                            │ │
│ └────────────────────────────────────────────────────┘ │
└───────────────────────────┬────────────────────────────┘
                            │ Target Controller Invocation
                            ▼
                   [DispatcherServlet]
                            │
                            ▼
                    [@RestController]
```

#### Key Components:
1. **`DelegatingFilterProxy`**:
   - Registered in the Servlet Container (Tomcat).
   - Knows nothing about security logic.
   - Its sole purpose is to locate a Spring Bean named `springSecurityFilterChain` inside the Spring `ApplicationContext` and delegate all `doFilter()` calls to it.
2. **`FilterChainProxy`**:
   - The actual Spring bean implementing `Filter`.
   - Manages one or more `SecurityFilterChain` instances.
   - Decides which filter chain executes for an incoming `HttpServletRequest` using `RequestMatcher`.
   - Clears the `SecurityContextHolder` after request completion to prevent ThreadLocal memory leaks.
3. **`SecurityFilterChain`**:
   - Defines a list of Spring Security filters ordered deterministically.

### 1.2 Core Filters & Execution Order in Spring Security 6

Filters execute in a strict sequential order:

| Position | Filter Class | Purpose |
| :--- | :--- | :--- |
| 1 | `ChannelProcessingFilter` | Enforces HTTPS redirects (if configured). |
| 2 | `SecurityContextHolderFilter` | Loads established security context from storage (replaces deprecated `SecurityContextPersistenceFilter`). |
| 3 | `ConcurrentSessionFilter` | Enforces session concurrency limits. |
| 4 | `CorsFilter` | Processes CORS preflight and request headers. |
| 5 | `CsrfFilter` | Validates CSRF tokens against mutating HTTP verbs (POST/PUT/DELETE). |
| 6 | `LogoutFilter` | Handles logout endpoint, invalidates tokens/sessions. |
| 7 | `AuthenticationFilter`s | `UsernamePasswordAuthenticationFilter`, `BearerTokenAuthenticationFilter`, or custom `JwtAuthenticationFilter`. |
| 8 | `RequestCacheAwareFilter` | Restores original requests saved prior to authentication redirection. |
| 9 | `SecurityContextHolderAwareRequestFilter` | Wraps request to implement `HttpServletRequest` security methods (`getUserPrincipal()`, `isUserInRole()`). |
| 10 | `AnonymousAuthenticationFilter` | Populates `SecurityContext` with an `AnonymousAuthenticationToken` if no authentication is present. |
| 11 | `ExceptionTranslationFilter` | Catches `AuthenticationException` (returns 401 Unauthorized) and `AccessDeniedException` (returns 403 Forbidden). |
| 12 | `AuthorizationFilter` | Final gatekeeper (replaces `FilterSecurityInterceptor`). Performs access checks via `AuthorizationManager`. |

---

## 2. Authentication Architecture

Authentication is the mechanism verifying that a principal is who they claim to be.

```
                      [HttpServletRequest]
                               │
                               ▼
                   [AuthenticationFilter]
           (Extracts credentials, creates unauthenticated
              UsernamePasswordAuthenticationToken)
                               │
                               ▼
                     [AuthenticationManager]
                    (Usually ProviderManager)
                               │
            ┌──────────────────┴──────────────────┐
            ▼                                     ▼
 [DaoAuthenticationProvider]            [CustomJwtAuthProvider]
            │
            ├──> UserDetailsService.loadUserByUsername()
            │         └──> Database / Identity Provider
            │
            └──> PasswordEncoder.matches(raw, encoded)
                        │
      (Success: returns authenticated Authentication token)
                        │
                        ▼
            [SecurityContextHolder]
        SecurityContextHolder.getContext().setAuthentication(authResult)
```

### 2.1 The Core Authentication SPIs

1. **`Authentication`**:
   Represents the user token.
   - `getPrincipal()`: Identity of the user (e.g., `UserDetails` or username string).
   - `getCredentials()`: Password or secret (often wiped after authentication).
   - `getAuthorities()`: Collection of `GrantedAuthority` (roles or permissions).
   - `isAuthenticated()`: Boolean flag.

2. **`AuthenticationManager` / `ProviderManager`**:
   The primary entry point. `ProviderManager` delegates to a list of configured `AuthenticationProvider`s.

3. **`AuthenticationProvider`**:
   Carries out authentication for specific token types.
   - `Authentication authenticate(Authentication authentication)`
   - `boolean supports(Class<?> authentication)`

4. **`SecurityContextHolder` & ThreadLocal Strategy**:
   Stores security context details. By default, it uses `MODE_THREADLOCAL`.
   - In standard multi-threaded web applications, each HTTP worker thread gets an isolated `SecurityContext`.
   - **Crucial**: In asynchronous executions (`@Async`, `CompletableFuture`), ThreadLocal does **not** automatically propagate unless `MODE_INHERITABLETHREADLOCAL` is set or an `DelegatingSecurityContextAsyncTaskExecutor` is configured.

---

## 3. Authorization (Access Control)

### 3.1 Roles vs Authorities
- **`GrantedAuthority`**: An arbitrary permission string (e.g., `READ_PRIVILEGES`, `ORDER_DELETE`).
- **Role**: By convention in Spring Security, a role is simply an authority prefixed with `ROLE_` (e.g., `ROLE_ADMIN`).
  - `hasRole("ADMIN")` checks for the authority `ROLE_ADMIN`.
  - `hasAuthority("ROLE_ADMIN")` or `hasAuthority("ADMIN")` checks the exact raw string without prefix manipulation.

### 3.2 Method-Level Security (`@EnableMethodSecurity`)

In Spring Security 6, `@EnableMethodSecurity` replaces the legacy `@EnableGlobalMethodSecurity(prePostEnabled = true)`.

```java
@Configuration
@EnableMethodSecurity(
    prePostEnabled = true, // Enables @PreAuthorize & @PostAuthorize (default: true)
    securedEnabled = true, // Enables @Secured
    jsr250Enabled = true   // Enables @RolesAllowed
)
public class MethodSecurityConfig {
}
```

#### Annotation Comparison:
- **`@PreAuthorize`**: Evaluates SpEL before method invocation. Can inspect method arguments.
- **`@PostAuthorize`**: Evaluates SpEL after method execution. Can inspect the return object (`returnObject`).
- **`@PreFilter`**: Filters a collection argument before method execution.
- **`@PostFilter`**: Filters a returned collection based on a SpEL rule.

```java
@Service
public class OrderService {

    // Pre-authorization based on role
    @PreAuthorize("hasRole('ADMIN') or hasAuthority('SCOPE_order:write')")
    public void deleteOrder(Long orderId) {
        // ...
    }

    // Checking if current authenticated user owns the resource
    @PreAuthorize("#userId == authentication.principal.id or hasRole('SUPER_ADMIN')")
    public UserProfile getUserProfile(Long userId) {
        return userRepository.findById(userId).orElseThrow();
    }

    // Inspecting method return value
    @PostAuthorize("returnObject.creatorUsername == authentication.name")
    public Order getOrderDetails(Long orderId) {
        return orderRepository.findById(orderId).orElseThrow();
    }

    // Filtering outgoing collection
    @PostFilter("filterObject.department == authentication.principal.department")
    public List<Document> getDepartmentDocuments() {
        return documentRepository.findAll();
    }
}
```

---

## 4. Production-Ready JWT Authentication (Complete Implementation)

Here is a complete, production-grade JWT implementation for **Spring Boot 3.x / Spring Security 6.x** using standard JJWT (`io.jsonwebtoken:jjwt-api:0.12.5`).

### 4.1 Maven Dependencies
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <!-- JJWT Dependencies -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.5</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.5</version>
        <scope>runtime</scope>
    </dependency>
</dependencies>
```

### 4.2 `JwtTokenProvider`
Handles signing, generation, parsing, and validation of JSON Web Tokens.

```java
package com.example.security.jwt;

import io.jsonwebtoken.Claims;
import io.jsonwebtoken.ExpiredJwtException;
import io.jsonwebtoken.JwtException;
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.security.Keys;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.security.core.GrantedAuthority;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.stereotype.Component;

import javax.crypto.SecretKey;
import java.nio.charset.StandardCharsets;
import java.util.Date;
import java.util.List;
import java.util.function.Function;

@Component
public class JwtTokenProvider {

    private static final Logger log = LoggerFactory.getLogger(JwtTokenProvider.class);

    private final SecretKey key;
    private final long jwtExpirationMs;

    public JwtTokenProvider(
            @Value("${app.jwt.secret:defaultSecretKeyWithAtLeast256BitsLengthForHmacSHA256}") String secret,
            @Value("${app.jwt.expiration-ms:3600000}") long jwtExpirationMs) {
        this.key = Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
        this.jwtExpirationMs = jwtExpirationMs;
    }

    public String generateToken(UserDetails userDetails) {
        List<String> roles = userDetails.getAuthorities().stream()
                .map(GrantedAuthority::getAuthority)
                .toList();

        return Jwts.builder()
                .subject(userDetails.getUsername())
                .claim("roles", roles)
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + jwtExpirationMs))
                .signWith(key)
                .compact();
    }

    public String extractUsername(String token) {
        return extractClaim(token, Claims::getSubject);
    }

    public <T> T extractClaim(String token, Function<Claims, T> claimsResolver) {
        Claims claims = extractAllClaims(token);
        return claimsResolver.apply(claims);
    }

    public boolean validateToken(String token) {
        try {
            extractAllClaims(token);
            return true;
        } catch (ExpiredJwtException ex) {
            log.error("JWT expired: {}", ex.getMessage());
        } catch (JwtException | IllegalArgumentException ex) {
            log.error("Invalid JWT token: {}", ex.getMessage());
        }
        return false;
    }

    private Claims extractAllClaims(String token) {
        return Jwts.parser()
                .verifyWith(key)
                .build()
                .parseSignedClaims(token)
                .getPayload();
    }
}
```

### 4.3 `JwtAuthenticationFilter`
Intercepts every request once, extracts Bearer token, validates it, and sets `SecurityContext`.

```java
package com.example.security.jwt;

import jakarta.servlet.FilterChain;
import jakarta.servlet.ServletException;
import jakarta.servlet.http.HttpServletRequest;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.lang.NonNull;
import org.springframework.security.authentication.UsernamePasswordAuthenticationToken;
import org.springframework.security.core.context.SecurityContextHolder;
import org.springframework.security.core.userdetails.UserDetails;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.web.authentication.WebAuthenticationDetailsSource;
import org.springframework.stereotype.Component;
import org.springframework.util.StringUtils;
import org.springframework.web.filter.OncePerRequestFilter;

import java.io.IOException;

@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtTokenProvider jwtTokenProvider;
    private final UserDetailsService userDetailsService;

    public JwtAuthenticationFilter(JwtTokenProvider jwtTokenProvider, UserDetailsService userDetailsService) {
        this.jwtTokenProvider = jwtTokenProvider;
        this.userDetailsService = userDetailsService;
    }

    @Override
    protected void doFilterInternal(
            @NonNull HttpServletRequest request,
            @NonNull HttpServletResponse response,
            @NonNull FilterChain filterChain) throws ServletException, IOException {

        String token = parseJwt(request);

        if (token != null && jwtTokenProvider.validateToken(token)) {
            String username = jwtTokenProvider.extractUsername(token);

            if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
                UserDetails userDetails = userDetailsService.loadUserByUsername(username);

                UsernamePasswordAuthenticationToken authentication =
                        new UsernamePasswordAuthenticationToken(userDetails, null, userDetails.getAuthorities());
                authentication.setDetails(new WebAuthenticationDetailsSource().buildDetails(request));

                SecurityContextHolder.getContext().setAuthentication(authentication);
            }
        }

        filterChain.doFilter(request, response);
    }

    private String parseJwt(HttpServletRequest request) {
        String headerAuth = request.getHeader("Authorization");
        if (StringUtils.hasText(headerAuth) && headerAuth.startsWith("Bearer ")) {
            return headerAuth.substring(7);
        }
        return null;
    }
}
```

### 4.4 Modern `SecurityConfig` (Spring Security 6.x)
Using Component-based configuration, Lambda DSL, and strict stateless session management.

```java
package com.example.security.config;

import com.example.security.jwt.JwtAuthenticationFilter;
import jakarta.servlet.http.HttpServletResponse;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.HttpMethod;
import org.springframework.security.authentication.AuthenticationManager;
import org.springframework.security.authentication.AuthenticationProvider;
import org.springframework.security.authentication.dao.DaoAuthenticationProvider;
import org.springframework.security.config.Customizer;
import org.springframework.security.config.annotation.authentication.configuration.AuthenticationConfiguration;
import org.springframework.security.config.annotation.method.configuration.EnableMethodSecurity;
import org.springframework.security.config.annotation.web.builders.HttpSecurity;
import org.springframework.security.config.annotation.web.configuration.EnableWebSecurity;
import org.springframework.security.config.annotation.web.configurers.AbstractHttpConfigurer;
import org.springframework.security.config.http.SessionCreationPolicy;
import org.springframework.security.core.userdetails.UserDetailsService;
import org.springframework.security.crypto.bcrypt.BCryptPasswordEncoder;
import org.springframework.security.crypto.password.PasswordEncoder;
import org.springframework.security.web.SecurityFilterChain;
import org.springframework.security.web.authentication.UsernamePasswordAuthenticationFilter;
import org.springframework.web.cors.CorsConfiguration;
import org.springframework.web.cors.CorsConfigurationSource;
import org.springframework.web.cors.UrlBasedCorsConfigurationSource;

import java.util.List;

@Configuration
@EnableWebSecurity
@EnableMethodSecurity
public class SecurityConfig {

    private final JwtAuthenticationFilter jwtAuthFilter;
    private final UserDetailsService userDetailsService;

    public SecurityConfig(JwtAuthenticationFilter jwtAuthFilter, UserDetailsService userDetailsService) {
        this.jwtAuthFilter = jwtAuthFilter;
        this.userDetailsService = userDetailsService;
    }

    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            // 1. Disable CSRF (Stateless REST API using Bearer Tokens)
            .csrf(AbstractHttpConfigurer::disable)
            
            // 2. Enable CORS with dedicated configuration source
            .cors(cors -> cors.configurationSource(corsConfigurationSource()))
            
            // 3. Exception Handling (Clean 401 and 403 JSON responses)
            .exceptionHandling(ex -> ex
                .authenticationEntryPoint((request, response, authException) -> {
                    response.setContentType("application/json");
                    response.setStatus(HttpServletResponse.SC_UNAUTHORIZED);
                    response.getWriter().write("{\"error\": \"Unauthorized\", \"message\": \"" + authException.getMessage() + "\"}");
                })
                .accessDeniedHandler((request, response, accessDeniedException) -> {
                    response.setContentType("application/json");
                    response.setStatus(HttpServletResponse.SC_FORBIDDEN);
                    response.getWriter().write("{\"error\": \"Forbidden\", \"message\": \"" + accessDeniedException.getMessage() + "\"}");
                })
            )
            
            // 4. Stateless session management
            .sessionManagement(session -> session.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            
            // 5. Authorize HTTP requests with requestMatchers (Spring Security 6)
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**", "/actuator/health").permitAll()
                .requestMatchers(HttpMethod.GET, "/api/v1/products/**").permitAll()
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()
            )
            
            // 6. Register authentication provider and JWT filter
            .authenticationProvider(authenticationProvider())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);

        return http.build();
    }

    @Bean
    public CorsConfigurationSource corsConfigurationSource() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://trusted-client.com", "http://localhost:3000"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE", "PATCH", "OPTIONS"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type", "X-Requested-With"));
        config.setExposedHeaders(List.of("Authorization"));
        config.setAllowCredentials(true);
        config.setMaxAge(3600L); // Preflight cache 1 hour

        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/**", config);
        return source;
    }

    @Bean
    public AuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider authProvider = new DaoAuthenticationProvider();
        authProvider.setUserDetailsService(userDetailsService);
        authProvider.setPasswordEncoder(passwordEncoder());
        return authProvider;
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12); // Work factor 12
    }

    @Bean
    public AuthenticationManager authenticationManager(AuthenticationConfiguration config) throws Exception {
        return config.getAuthenticationManager();
    }
}
```

---

## 5. OAuth2 and OpenID Connect (OIDC)

### 5.1 The Four OAuth2 Roles
1. **Resource Owner**: The end-user granting access to their protected resources.
2. **Client**: The application requesting access tokens on behalf of the resource owner (e.g., SPA, mobile app, microservice).
3. **Authorization Server**: Issues tokens upon authenticating the Resource Owner (e.g., Keycloak, Okta, Auth0, Spring Authorization Server).
4. **Resource Server**: Hosts the protected APIs and validates access tokens.

### 5.2 Authorization Code Flow with PKCE (Proof Key for Code Exchange)
The gold standard for Single Page Applications (SPAs) and mobile clients:
1. Client generates `code_verifier` (cryptographic random secret) and computes `code_challenge = SHA256(code_verifier)`.
2. Client redirects user to Authorization Server with `code_challenge` & `code_challenge_method=S256`.
3. User authenticates and grants consent.
4. Authorization Server redirects back with an authorization `code`.
5. Client makes back-channel POST request with `code` and the original plain `code_verifier`.
6. Authorization server hashes `code_verifier` and matches with saved `code_challenge`. If valid, issues Access Token and Refresh Token.

### 5.3 Configuring Spring Boot as an OAuth2 Resource Server

Add dependency:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-resource-server</artifactId>
</dependency>
```

`application.yml`:
```yaml
spring:
  security:
    oauth2:
      resourceserver:
        jwt:
          issuer-uri: https://auth.company.com/realms/enterprise
          jwk-set-uri: https://auth.company.com/realms/enterprise/protocol/openid-connect/certs
```

Resource Server FilterChain with JWT Claim-to-Role Mapping:
```java
@Configuration
@EnableWebSecurity
public class OAuth2ResourceServerConfig {

    @Bean
    public SecurityFilterChain resourceServerFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/public/**").permitAll()
                .requestMatchers("/api/orders/**").hasAuthority("SCOPE_orders.read")
                .anyRequest().authenticated()
            )
            .oauth2ResourceServer(oauth2 -> oauth2
                .jwt(jwt -> jwt.jwtAuthenticationConverter(jwtAuthenticationConverter()))
            );
        return http.build();
    }

    @Bean
    public JwtAuthenticationConverter jwtAuthenticationConverter() {
        JwtGrantedAuthoritiesConverter grantedAuthoritiesConverter = new JwtGrantedAuthoritiesConverter();
        grantedAuthoritiesConverter.setAuthorityPrefix("ROLE_");
        grantedAuthoritiesConverter.setAuthoritiesClaimName("roles");

        JwtAuthenticationConverter jwtConverter = new JwtAuthenticationConverter();
        jwtConverter.setJwtGrantedAuthoritiesConverter(grantedAuthoritiesConverter);
        return jwtConverter;
    }
}
```

---

## 6. CSRF & CORS Deep Dive

### 6.1 CSRF (Cross-Site Request Forgery)
- **Mechanism**: Attacker tricks a user's browser into executing unwanted actions on a trusted site where the user is currently authenticated via cookies.
- **Why Disable CSRF in Stateless REST APIs**:
  - CSRF vulnerabilities exist because browsers **automatically attach ambient credentials (cookies, HTTP basic auth headers)** to cross-site requests.
  - When an API uses standard Bearer JWT tokens stored in JavaScript memory or web storage, the browser does **not** automatically attach the `Authorization: Bearer <token>` header across cross-origin links or forms. Hence, CSRF attacks are technically impossible against pure header-based Bearer token APIs.
- **When CSRF MUST be enabled**: If your application stores JWTs or session IDs inside `HttpOnly` cookies, CSRF protection is strictly mandatory! Use Spring's `CookieCsrfTokenRepository.withHttpOnlyFalse()`.

### 6.2 CORS (Cross-Origin Resource Sharing)
- CORS is a browser-enforced security mechanism preventing malicious web pages from reading data from another origin without permission.
- Preflight Requests: For requests modifying data or using custom headers (`Authorization`), the browser first issues an `OPTIONS` request.
- **Crucial Rule**: `CorsFilter` must execute **before** Spring Security's authorization filters. If authorization runs first, the unauthenticated `OPTIONS` preflight request will be rejected with HTTP 401 or 403, and the browser will report a CORS failure!

---

## 7. Password Encoding: BCrypt Internals

```java
PasswordEncoder encoder = new BCryptPasswordEncoder(12);
String hash = encoder.encode("password123");
// Output format: $2a$12$e8uq4fMhC7O/gqM6fUq90.eU76j7ZkWd2L.e3k5K1m2r5s6t7u8v9
```

### Breakdown of a BCrypt Hash:
```
 $2a$  12$  e8uq4fMhC7O/gqM6fUq90.  eU76j7ZkWd2L.e3k5K1m2r5s6t7u8v9
 └──┬─┘ └┬┘  └──────────┬───────────┘  └──────────────┬──────────────┘
    │    │              │                             │
    │    │              │                             └── 31 characters of 184-bit hash
    │    │              └─────────────────────────────── 22 characters of 128-bit Salt
    │    └────────────────────────────────────────────── Cost factor (2^12 = 4096 iterations)
    └─────────────────────────────────────────────────── BCrypt algorithm version
```

### Why Salt Doesn't Need a Separate Database Column
The salt is generated randomly per encryption and embedded directly in the resulting output string. When `matches("password123", hash)` is called, BCrypt extracts the salt and cost factor from the stored hash, hashes the candidate password using those exact parameters, and checks for equality using a constant-time comparison to prevent timing attacks.

---

## 8. Major Architectural Shifts in Spring Security 6.x

For senior candidates, interviewers deliberately test awareness of modernization from Spring Security 5 to 6 (Spring Boot 3):

```
+───────────────────────────────────────────────────────────────────────────────+
|               Spring Security 5.x               Spring Security 6.x           |
+───────────────────────────────────────────────────────────────────────────────+
| WebSecurityConfigurerAdapter (extends)     --> Component-based @Bean          |
| antMatchers(), regexMatchers()             --> requestMatchers()              |
| authorizeRequests()                        --> authorizeHttpRequests()        |
| Cascading chaining (and())                 --> Lambda DSL configurers         |
| AccessDecisionManager / Voters             --> AuthorizationManager           |
| SecurityContextPersistenceFilter           --> SecurityContextHolderFilter    |
+───────────────────────────────────────────────────────────────────────────────+
```

### 1. Goodbye `WebSecurityConfigurerAdapter`
Configurations must no longer extend base adapter classes. Instead, register a `SecurityFilterChain` bean.

### 2. Lambda DSL Enforcement
Spring Security 6 removed the `.and()` chaining syntax completely. All configuration must be expressed using functional lambda DSLs:
```java
// Spring Security 5:
http.csrf().disable().and().authorizeRequests().antMatchers("/public").permitAll();

// Spring Security 6:
http.csrf(AbstractHttpConfigurer::disable)
    .authorizeHttpRequests(auth -> auth.requestMatchers("/public").permitAll());
```

### 3. Unified `requestMatchers()`
`antMatchers()`, `mvcMatchers()`, and `regexMatchers()` have been collapsed into `requestMatchers()`. Under the hood, Spring Security automatically uses Spring MVC's `HandlerMappingIntrospector` if MVC is present, preventing past CVEs caused by pattern matching discrepancies between Servlet container and Spring MVC.

---

## 9. Common Interview Questions & Answers

### Q1: Why did Spring Security replace `SecurityContextPersistenceFilter` with `SecurityContextHolderFilter`?
**Answer**:
`SecurityContextPersistenceFilter` eagerly loaded the `SecurityContext` on every request and saved it at the end of the request automatically into the `HttpSession`.
In modern microservices and high-throughput architectures, this automatic and eager persistence caused unnecessary overhead and race conditions across concurrent requests.
`SecurityContextHolderFilter` defers loading and requires explicit persistence via `SecurityContextRepository`. It eliminates implicit session writes and integrates seamlessly with stateless or reactive execution.

### Q2: What is the purpose of `OncePerRequestFilter` and when would a filter run multiple times without it?
**Answer**:
A standard Servlet `Filter` can be invoked multiple times within a single client request cycle if request dispatching occurs (e.g., `REQUEST`, `FORWARD`, `INCLUDE`, `ASYNC`, or `ERROR`).
For example, when an uncaught exception is thrown and dispatched to `/error`, a standard filter executes a second time. `OncePerRequestFilter` guarantees that its `doFilterInternal()` logic runs **exactly once per request lifecycle** by checking and setting a unique request attribute.

### Q3: How do you handle authentication in an asynchronous `@Async` method?
**Answer**:
By default, `SecurityContextHolder` uses a standard `ThreadLocal`. When a method executes in a separate thread via `@Async` or an `ExecutorService`, the child thread starts with an empty context.
**Solutions**:
1. Configure `DelegatingSecurityContextAsyncTaskExecutor`:
```java
@Bean
public AsyncTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.initialize();
    return new DelegatingSecurityContextAsyncTaskExecutor(executor);
}
```
2. Wrap `Runnable`/`Callable` with `DelegatingSecurityContextRunnable(task)`.
3. Set the JVM property `-Dspring.security.strategy=MODE_INHERITABLETHREADLOCAL` (caution: risky in thread pools where threads are reused across different users).

### Q4: How does Spring Security protect against Timing Attacks?
**Answer**:
1. In `PasswordEncoder.matches()`, comparisons are performed using constant-time string comparisons (`MessageDigest.isEqual()`). A naive `String.equals()` terminates early on the first mismatched byte, allowing an attacker to deduce characters by measuring response latency.
2. In `DaoAuthenticationProvider`, if a username does not exist in the database, Spring Security still performs a dummy password check using a pre-computed salt (`userNotFoundEncodedPassword`) to equalize response time between "User Not Found" and "Invalid Password".

---

## 10. Gotchas & Production Pitfalls

1. **Incorrect Filter Ordering with Custom Filters**:
   - Registering a custom JWT filter using `.addFilter()` without specifying `.addFilterBefore()` or `.addFilterAfter()` throws an `IllegalArgumentException` because Spring Security requires deterministic positioning relative to known standard filters.

2. **Permitting URLs in Security but Blocking in Controllers**:
   - If `/api/v1/auth/**` is marked `permitAll()` in `SecurityFilterChain`, but `@PreAuthorize("hasRole('ADMIN')")` is placed on the controller method, method security will reject the request with HTTP 403 Forbidden! Security filter authorization and method-level authorization are two independent layers of security.

3. **CORS Configuration Overwritten by Spring MVC Interceptors**:
   - Configuring CORS solely in `@CrossOrigin` on controllers or via `WebMvcConfigurer.addCorsMappings()` is often insufficient for security endpoints. Unauthenticated preflight `OPTIONS` requests can be blocked by Spring Security filters before ever reaching Spring MVC. Always define `CorsConfigurationSource` inside the `SecurityFilterChain`.

4. **Storing JWT Tokens in LocalStorage**:
   - Storing sensitive authentication tokens in browser `localStorage` leaves them completely exposed to Cross-Site Scripting (XSS) attacks. For browser-based SPAs, store tokens in memory, or use secure, `SameSite=Strict`, `HttpOnly` cookies combined with CSRF protection.
