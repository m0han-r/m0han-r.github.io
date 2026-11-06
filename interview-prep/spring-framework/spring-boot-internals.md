← [Back to Index](../README.md)

# Spring Boot Internals for Senior Developers

A deep dive into the inner workings of Spring Boot designed for senior Java engineers (8+ years experience). This guide covers auto-configuration mechanics, custom auto-configuration authoring, starter POM architecture, embedded servlet containers, Actuator and Micrometer internals, configuration precedence, the application lifecycle, and Spring Boot 3.x architectural shifts.

---

## 1. Auto-Configuration Mechanism

Spring Boot auto-configuration attempts to automatically configure your Spring application based on the jar dependencies detected on the classpath, defined beans, and configuration properties.

### 1.1 The `@SpringBootApplication` & `@EnableAutoConfiguration` Anchor

At the entry point of every application lies `@SpringBootApplication`, which is a meta-annotation composed of:
1. `@SpringBootConfiguration`: Specialized form of `@Configuration` designating the class as a configuration source.
2. `@ComponentScan`: Scans for `@Component`, `@Service`, `@Repository`, `@Controller` within the anchor class package and sub-packages.
3. `@EnableAutoConfiguration`: Enables Spring Boot's auto-configuration machinery.

```
                  @SpringBootApplication
                            │
       ┌────────────────────┼────────────────────┐
       ▼                    ▼                    ▼
@SpringBootConfiguration  @ComponentScan  @EnableAutoConfiguration
                                                 │
                                                 ▼
                                     @Import(AutoConfigurationImportSelector.class)
```

Inside `@EnableAutoConfiguration`:
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@AutoConfigurationPackage
@Import(AutoConfigurationImportSelector.class)
public @interface EnableAutoConfiguration {
    String ENABLED_OVERRIDE_PROPERTY = "spring.boot.enableautoconfiguration";
    Class<?>[] exclude() default {};
    String[] excludeName() default {};
}
```

### 1.2 `AutoConfigurationImportSelector` Workflow

The `AutoConfigurationImportSelector` implements `DeferredImportSelector`. Unlike standard `ImportSelector`s that run immediately during component scanning, a `DeferredImportSelector` runs **after all user-defined `@Configuration` beans have been registered and processed**. This deferred execution is vital: auto-configuration beans must evaluate conditions against existing user beans to back off when custom configurations are detected.

```
Step 1: Application starts -> SpringApplication.run()
        │
Step 2: Component Scanning registers all user-defined @Configuration and @Component beans.
        │
Step 3: DeferredImportSelector executes -> AutoConfigurationImportSelector.selectImports()
        │
Step 4: Load candidate configuration class names:
        - Spring Boot 2.x: META-INF/spring.factories (key: EnableAutoConfiguration)
        - Spring Boot 3.x: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
        │
Step 5: Deduplicate candidate class names and remove explicitly excluded configurations:
        (@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class}))
        │
Step 6: Filter candidates through Condition Evaluator (@ConditionalOnClass, @ConditionalOnMissingBean...)
        │
Step 7: Sort matching configurations based on @AutoConfigureOrder, @AutoConfigureBefore, @AutoConfigureAfter
        │
Step 8: Register surviving configurations into the BeanDefinitionRegistry.
```

### 1.3 `spring.factories` vs `AutoConfiguration.imports`

In **Spring Boot 1.x and 2.x**, auto-configuration classes were discovered via `META-INF/spring.factories` using `SpringFactoriesLoader`:
```properties
# Spring Boot 2.x: META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration,\
  org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
```

**Why Spring Boot 3.x changed this:**
- `spring.factories` required reading and parsing properties across multiple jars into memory even when querying a single SPI.
- In **Spring Boot 3.0+**, auto-configuration registration shifted to a dedicated file:
  `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`
- Each line in this file contains the fully qualified class name of an auto-configuration class.
- Loaded using `ImportCandidates.load(AutoConfiguration.class, getBeanClassLoader())`, which avoids parsing unrelated factories and dramatically accelerates startup and Ahead-Of-Time (AOT) GraalVM native image analysis.

```text
# Spring Boot 3.x: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
org.springframework.boot.autoconfigure.orm.jpa.HibernateJpaAutoConfiguration
org.springframework.boot.autoconfigure.kafka.KafkaAutoConfiguration
```

### 1.4 The `@Conditional` Annotation Hierarchy

Auto-configuration relies on conditional evaluation via Spring's `Condition` SPI (`org.springframework.context.annotation.Condition`). Spring Boot provides pre-built conditional annotations:

| Annotation | Evaluation Trigger | Internal Implementation Class |
| :--- | :--- | :--- |
| `@ConditionalOnClass` | Matches if specific classes are present on the application classpath. | `OnClassCondition` |
| `@ConditionalOnMissingClass` | Matches only if specific classes are absent from the classpath. | `OnClassCondition` |
| `@ConditionalOnBean` | Matches if a bean of specified type/name already exists in the `BeanFactory`. | `OnBeanCondition` |
| `@ConditionalOnMissingBean` | Matches only if **no** bean of specified type/name exists in the context (enables user overrides). | `OnBeanCondition` |
| `@ConditionalOnProperty` | Matches based on the value or presence of a Spring Environment property. | `OnPropertyCondition` |
| `@ConditionalOnResource` | Matches if a designated resource (e.g., `schema.sql`) exists on the classpath. | `OnResourceCondition` |
| `@ConditionalOnWebApplication` | Matches if the application is a reactive or servlet-based web application. | `OnWebApplicationCondition` |
| `@ConditionalOnExpression` | Evaluates a SpEL (Spring Expression Language) expression. | `OnExpressionCondition` |

#### How `@ConditionalOnMissingBean` Works Internally:
```java
@AutoConfiguration
@ConditionalOnClass(DataSource.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(DataSource.class)
    public DataSource dataSource(DataSourceProperties properties) {
        return properties.initializeDataSourceBuilder().build();
    }
}
```
If a developer defines a `@Bean public DataSource myCustomDataSource()`, Spring Boot skips the auto-configured `DataSource` bean completely because `OnBeanCondition` detects the user-supplied definition during the deferred evaluation phase.

---

## 2. Starter POMs: Mechanics & Architecture

### 2.1 The Problem Starters Solve
Before Spring Boot, configuring an enterprise stack (e.g., Spring MVC + Hibernate + Jackson + Logback) required manually managing 20+ individual dependencies, ensuring binary compatibility, and resolving transitive dependency version conflicts (JAR Hell).

### 2.2 How Starter POMs Function
A Spring Boot Starter (e.g., `spring-boot-starter-web`) contains **no actual Java code**. It is an **aggregation POM** that bundles:
1. **Curated Transitive Dependencies**: Brings in compatible versions of libraries tested together by the Spring Boot team.
2. **Auto-Configuration Triggers**: Transitive dependencies place sentinel classes on the classpath (e.g., `DispatcherServlet`, `ObjectMapper`), activating corresponding auto-configurations.

```
spring-boot-starter-web
   │
   ├── spring-boot-starter (core boot, logging, auto-configure, YAML)
   ├── spring-boot-starter-json (Jackson Databind, JSR310 modules)
   ├── spring-boot-starter-tomcat (embedded Tomcat container)
   ├── spring-web (RestClient, HttpMessageConverters)
   └── spring-webmvc (DispatcherServlet, Controller mappings)
```

### 2.3 Dependency Management & BOM Pattern
Spring Boot utilizes Maven's `dependencyManagement` section via the **Bill of Materials (BOM)**:
```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>3.2.4</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```
The BOM locks compatible versions of hundreds of third-party dependencies (Hibernate, Jackson, Netty, Micrometer, Flyway) without adding them directly to the classpath. Developers specify `<groupId>` and `<artifactId>` without `<version>`.

---

## 3. Embedded Web Servers: Internals & Tuning

Spring Boot inverts the traditional deployment model: instead of packaging a WAR and deploying it to an external servlet container, the servlet container is packaged inside the executable JAR.

### 3.1 Container Lifecycle & Factories
When a web application boots:
1. `ServletWebServerApplicationContext` is instantiated.
2. During the context refresh phase (`onRefresh()`), it invokes `createWebServer()`.
3. It locates a `ServletWebServerFactory` bean in the `BeanFactory`:
   - `TomcatServletWebServerFactory` (default when `spring-boot-starter-tomcat` is present)
   - `JettyServletWebServerFactory` (when using `spring-boot-starter-jetty`)
   - `UndertowServletWebServerFactory` (when using `spring-boot-starter-undertow`)
4. The factory instantiates and configures the embedded instance (e.g., `org.apache.catalina.startup.Tomcat`).
5. It binds `DispatcherServlet` to a `ServletRegistrationBean`, mounts filters, and starts the listening socket on `server.port`.

```
ApplicationContext.refresh()
        │
        ▼
ServletWebServerApplicationContext.onRefresh()
        │
        ▼
ServletWebServerApplicationContext.createWebServer()
        │
        ├──> Finds ServletWebServerFactory bean (e.g., TomcatServletWebServerFactory)
        │
        ├──> Applies WebServerFactoryCustomizer beans (applies server.* properties)
        │
        └──> factory.getWebServer(servletContextInitializer)
                    │
                    ▼
             Tomcat.start() -> Connector binds to port -> Listening for HTTP
```

### 3.2 Programmatic Customization via `WebServerFactoryCustomizer`

```java
package com.example.config;

import org.apache.catalina.connector.Connector;
import org.apache.coyote.http11.AbstractHttp11Protocol;
import org.springframework.boot.web.embedded.tomcat.TomcatServletWebServerFactory;
import org.springframework.boot.web.server.WebServerFactoryCustomizer;
import org.springframework.context.annotation.Configuration;

import java.time.Duration;

@Configuration
public class CustomTomcatConfiguration implements WebServerFactoryCustomizer<TomcatServletWebServerFactory> {

    @Override
    public void customize(TomcatServletWebServerFactory factory) {
        factory.setPort(8080);
        factory.setSessionTimeout(Duration.ofMinutes(30));
        
        // Low-level Tomcat connector tuning
        factory.addConnectorCustomizers((Connector connector) -> {
            if (connector.getProtocolHandler() instanceof AbstractHttp11Protocol<?> protocol) {
                // Connection keep-alive and keep-alive timeout
                protocol.setKeepAliveTimeout(20000); // 20s
                protocol.setMaxKeepAliveRequests(100);
                // Buffer sizes
                protocol.setMaxHttpHeaderSize(16384); // 16 KB
            }
        });
    }
}
```

### 3.3 High-Throughput Tomcat Thread Pool Tuning (`application.yml`)

```yaml
server:
  port: 8080
  tomcat:
    threads:
      max: 400          # Maximum request processing threads (default: 200)
      min-spare: 50     # Initial / warm pool threads (default: 10)
    max-connections: 10000 # Max concurrent TCP connections before refusing
    accept-count: 200   # OS TCP accept queue depth when all threads are busy
    connection-timeout: 5000ms # Timeout before closing idle TCP connection
```

### 3.4 Virtual Threads (Project Loom) in Spring Boot 3.2+
Starting with Spring Boot 3.2 and Java 21, enabling virtual threads switches Tomcat and Spring MVC from a platform thread-per-request model to virtual threads:
```yaml
spring:
  threads:
    virtual:
      enabled: true
```
Under the hood, Tomcat configures an `Executors.newVirtualThreadPerTaskExecutor()` for request handling, allowing hundreds of thousands of concurrent connections with minimal memory overhead compared to 1MB OS stack platform threads.

---

## 4. Spring Boot Actuator & Observability (Micrometer)

Spring Boot Actuator provides production-ready monitoring, metrics, and health audits through JMX and HTTP endpoints.

### 4.1 Architecture of Actuator Endpoints
Actuator endpoints are declared using `@Endpoint`, `@WebEndpoint`, or `@JmxEndpoint`. They are decoupled from the transport protocol:
- `@ReadOperation`: Maps to HTTP `GET`.
- `@WriteOperation`: Maps to HTTP `POST`.
- `@DeleteOperation`: Maps to HTTP `DELETE`.

### 4.2 Building a Custom Production Actuator Endpoint

Here is a custom thread pool monitor endpoint exposing internal metrics and runtime management:

```java
package com.example.actuator;

import org.springframework.boot.actuate.endpoint.annotation.*;
import org.springframework.stereotype.Component;

import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.ThreadPoolExecutor;

@Component
@Endpoint(id = "threadpool-monitor")
public class ThreadPoolMonitorEndpoint {

    private final Map<String, ThreadPoolExecutor> registeredPools = new HashMap<>();

    public void registerExecutor(String poolName, ThreadPoolExecutor executor) {
        registeredPools.put(poolName, executor);
    }

    @ReadOperation
    public Map<String, Object> getPoolStatistics() {
        Map<String, Object> stats = new HashMap<>();
        registeredPools.forEach((name, executor) -> {
            Map<String, Object> details = new HashMap<>();
            details.put("activeCount", executor.getActiveCount());
            details.put("poolSize", executor.getPoolSize());
            details.put("corePoolSize", executor.getCorePoolSize());
            details.put("maximumPoolSize", executor.getMaximumPoolSize());
            details.put("queueSize", executor.getQueue().size());
            details.put("queueRemainingCapacity", executor.getQueue().remainingCapacity());
            details.put("completedTaskCount", executor.getCompletedTaskCount());
            stats.put(name, details);
        });
        return stats;
    }

    @WriteOperation
    public String adjustCorePoolSize(String poolName, int newCoreSize) {
        ThreadPoolExecutor executor = registeredPools.get(poolName);
        if (executor == null) {
            return "Pool not found: " + poolName;
        }
        executor.setCorePoolSize(newCoreSize);
        return "Updated " + poolName + " corePoolSize to " + newCoreSize;
    }
}
```

Expose and access this endpoint:
```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,prometheus,threadpool-monitor
  server:
    port: 9090 # Isolated management port for security
```
- GET: `http://localhost:9090/actuator/threadpool-monitor`
- POST: `http://localhost:9090/actuator/threadpool-monitor` with JSON body `{"poolName": "orderProcessor", "newCoreSize": 25}`

### 4.3 Custom HealthIndicator Implementation

```java
package com.example.actuator;

import org.springframework.boot.actuate.health.Health;
import org.springframework.boot.actuate.health.HealthIndicator;
import org.springframework.stereotype.Component;

import java.net.Socket;

@Component("paymentGatewayHealth")
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final String gatewayHost = "payment.external-provider.com";
    private final int gatewayPort = 443;

    @Override
    public Health health() {
        try (Socket socket = new Socket(gatewayHost, gatewayPort)) {
            return Health.up()
                    .withDetail("gatewayHost", gatewayHost)
                    .withDetail("latencyMs", 12)
                    .build();
        } catch (Exception ex) {
            return Health.down(ex)
                    .withDetail("gatewayHost", gatewayHost)
                    .withDetail("failureReason", "TCP Handshake timeout")
                    .build();
        }
    }
}
```

### 4.4 Micrometer Metrics & Prometheus Integration

Micrometer acts as the "SLF4J for application metrics". Spring Boot integrates Micrometer out-of-the-box.

```java
package com.example.service;

import io.micrometer.core.instrument.Counter;
import io.micrometer.core.instrument.MeterRegistry;
import io.micrometer.core.instrument.Timer;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class OrderProcessingService {

    private final Counter orderCounter;
    private final Timer orderExecutionTimer;

    public OrderProcessingService(MeterRegistry registry) {
        // Counter for throughput and error rate
        this.orderCounter = Counter.builder("business.orders.placed")
                .description("Total number of orders placed")
                .tag("region", "us-east-1")
                .register(registry);

        // Timer for latency percentiles and SLA monitoring
        this.orderExecutionTimer = Timer.builder("business.orders.processing.time")
                .description("Time taken to process an order")
                .publishPercentiles(0.5, 0.95, 0.99) // p50, p95, p99
                .publishPercentileHistogram()
                .register(registry);
    }

    public void processOrder(String orderId) {
        long start = System.nanoTime();
        try {
            // Business execution logic
            Thread.sleep(15);
            orderCounter.increment();
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        } finally {
            orderExecutionTimer.record(System.nanoTime() - start, TimeUnit.NANOSECONDS);
        }
    }
}
```

---

## 5. Configuration Precedence & `@ConfigurationProperties`

Spring Boot uses a carefully ordered `PropertySource` chain to resolve configuration values. When the same key is defined in multiple sources, higher precedence sources override lower ones.

### 5.1 Externalized Configuration Precedence Order (Top 17)

Highest priority wins (values at the top override values below):

1. **Devtools global settings** properties on `$HOME/.config/spring-boot` (when devtools is active).
2. **`@TestPropertySource`** annotations on test classes.
3. **`@SpringBootTest#properties`** annotation attribute.
4. **Command line arguments** (e.g., `--server.port=9090`).
5. **Properties from `SPRING_APPLICATION_JSON`** (inline JSON embedded in environment variable or system property).
6. **ServletConfig init parameters**.
7. **ServletContext init parameters**.
8. **JNDI attributes** from `java:comp/env`.
9. **Java System properties** (`System.getProperties()`, e.g., `-Dserver.port=8081`).
10. **OS Environment variables** (e.g., `SERVER_PORT=8081`).
11. **`RandomValuePropertySource`** (e.g., `random.int`, `random.uuid`).
12. **Profile-specific application properties outside packaged JAR** (`application-{profile}.properties` or `.yml`).
13. **Profile-specific application properties packaged inside JAR** (`application-{profile}.properties` or `.yml`).
14. **Application properties outside packaged JAR** (`application.properties` or `.yml`).
15. **Application properties packaged inside JAR** (`application.properties` or `.yml`).
16. **`@PropertySource`** annotations on `@Configuration` classes.
17. **Default properties** specified via `SpringApplication.setDefaultProperties`.

> **Senior Interview Tip**: Environment variables map to relaxed names. `SPRING_DATASOURCE_URL` maps to `spring.datasource.url` or `spring.datasource.URL`. Dot notation is converted to underscores in bash.

### 5.2 Immutable `@ConfigurationProperties` with Records (Spring Boot 3.x)

In modern Spring Boot, configuration properties should be modeled as immutable Java records with validation:

```java
package com.example.config;

import jakarta.validation.constraints.Max;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.boot.context.properties.bind.DefaultValue;
import org.springframework.validation.annotation.Validated;

import java.time.Duration;

@ConfigurationProperties(prefix = "app.payment-client")
@Validated
public record PaymentClientProperties(
        @NotBlank
        String baseUrl,

        @Min(100) @Max(60000)
        @DefaultValue("5000")
        int connectTimeoutMs,

        @DefaultValue("10s")
        Duration readTimeout,

        RetryConfig retry
) {
    public record RetryConfig(
            @Min(0) @Max(5) int maxAttempts,
            Duration backoffDelay
    ) {}
}
```

Enable scanning via `@ConfigurationPropertiesScan` or `@EnableConfigurationProperties(PaymentClientProperties.class)`:
```java
@Configuration
@EnableConfigurationProperties(PaymentClientProperties.class)
public class PaymentClientConfiguration {

    @Bean
    public RestClient paymentRestClient(PaymentClientProperties props) {
        return RestClient.builder()
                .baseUrl(props.baseUrl())
                .build();
    }
}
```

---

## 6. Spring Boot Lifecycle & Extension Points

Understanding how the `ApplicationContext` bootstraps, runs, and terminates is critical for senior architecture and debugging startup latency.

### 6.1 Application Startup Lifecycle Sequence

```
1. main() invoked
   │
2. SpringApplication instantiated
   ├── Detects web application type (SERVLET, REACTIVE, NONE)
   ├── Loads ApplicationContextInitializer from spring.factories / imports
   └── Loads ApplicationListener implementations
   │
3. SpringApplication.run(String... args)
   │
   ├── Start stopwatch
   ├── Prepare Environment (StandardEnvironment / StandardServletEnvironment)
   │     └── Publish ApplicationEnvironmentPreparedEvent
   │
   ├── Print Banner
   │
   ├── Instantiate ApplicationContext
   │     └── AnnotationConfigServletWebServerApplicationContext (for Servlet web)
   │
   ├── Prepare Context
   │     ├── Apply ApplicationContextInitializers
   │     ├── Publish ApplicationContextInitializedEvent
   │     └── Load BeanDefinitions from primary source (@SpringBootApplication)
   │     └── Publish ApplicationPreparedEvent
   │
   ├── Refresh Context (Core Spring Framework ApplicationContext.refresh())
   │     ├── Prepare BeanFactory
   │     ├── Invoke BeanFactoryPostProcessors (processes @Configuration, @ComponentScan)
   │     ├── Register BeanPostProcessors
   │     ├── Initialize MessageSource & ApplicationEventMulticaster
   │     ├── onRefresh() -> Web server (Tomcat) initialized and started
   │     ├── Instantiate all remaining non-lazy Singleton Beans
   │     └── Finish Refresh (Publish ContextRefreshedEvent)
   │
   ├── Publish ApplicationStartedEvent
   │
   ├── Execute Runners:
   │     ├── ApplicationRunner.run(ApplicationArguments args)
   │     └── CommandLineRunner.run(String... args)
   │
   └── Publish ApplicationReadyEvent
```

### 6.2 The Complete Spring Bean Lifecycle

```
[1. Class Loading & Reflection Discovery]
                    │
[2. Instantiation (Constructor Invocation)]
                    │
[3. Populate Bean Properties (Dependency Injection / Field & Setter Injection)]
                    │
[4. BeanNameAware.setBeanName(String name)]
                    │
[5. BeanClassLoaderAware.setBeanClassLoader(ClassLoader classLoader)]
                    │
[6. BeanFactoryAware.setBeanFactory(BeanFactory beanFactory)]
                    │
[7. EnvironmentAware / ApplicationContextAware.setApplicationContext(...)]
                    │
[8. BeanPostProcessor: postProcessBeforeInitialization()]
                    │
[9. @PostConstruct annotated initialization methods]
                    │
[10. InitializingBean.afterPropertiesSet()]
                    │
[11. Custom @Bean(initMethod = "customInit")]
                    │
[12. BeanPostProcessor: postProcessAfterInitialization()] (AOP Proxies created here!)
                    │
================ [BEAN IS FULLY INITIALIZED & IN SERVICE] ================
                    │
[13. ApplicationContext shutdown begins]
                    │
[14. @PreDestroy annotated destruction methods]
                    │
[15. DisposableBean.destroy()]
                    │
[16. Custom @Bean(destroyMethod = "customDestroy")]
```

> **Senior Gotcha**: AOP proxies (CGLIB/JDK) are generated during step 12 (`postProcessAfterInitialization`). If you call an annotated `@Transactional` or `@Async` method from inside `@PostConstruct` (step 9), the advice will **NOT** execute because the proxy has not been wrapped yet!

### 6.3 `ApplicationRunner` vs `CommandLineRunner`

Both execute after the `ApplicationContext` is fully refreshed and ready, but before `SpringApplication.run()` completes.

| Feature | `CommandLineRunner` | `ApplicationRunner` |
| :--- | :--- | :--- |
| Method Signature | `void run(String... args)` | `void run(ApplicationArguments args)` |
| Argument Parsing | Raw array of strings | Structured: distinguishes option arguments (`--key=value`) from non-option arguments |
| Usage | Simple scripts | Production services requiring flags and parameters |

```java
package com.example.runner;

import org.springframework.boot.ApplicationArguments;
import org.springframework.boot.ApplicationRunner;
import org.springframework.core.annotation.Order;
import org.springframework.stereotype.Component;

@Component
@Order(1) // Controls execution order relative to other runners
public class DataBootstrapRunner implements ApplicationRunner {

    @Override
    public void run(ApplicationArguments args) {
        // Access parsed options: --env=staging --migrate=true
        if (args.containsOption("migrate")) {
            System.out.println("Running migration with values: " + args.getOptionValues("migrate"));
        }
        // Access raw non-option arguments
        System.out.println("Non-option args: " + args.getNonOptionArgs());
    }
}
```

---

## 7. Step-by-Step Custom Auto-Configuration (Production Example)

Let's build a production-grade custom auto-configuration for a distributed **Rate Limiter SDK**.

### Step 1: Define the Service Interface & Implementation
```java
package com.example.ratelimiter;

public interface RateLimiterService {
    boolean allowRequest(String apiKey);
}
```

```java
package com.example.ratelimiter;

import java.util.concurrent.ConcurrentHashMap;
import java.util.concurrent.atomic.AtomicInteger;

public class InMemoryRateLimiterService implements RateLimiterService {

    private final int limitPerMinute;
    private final ConcurrentHashMap<String, AtomicInteger> requestCounts = new ConcurrentHashMap<>();

    public InMemoryRateLimiterService(int limitPerMinute) {
        this.limitPerMinute = limitPerMinute;
    }

    @Override
    public boolean allowRequest(String apiKey) {
        AtomicInteger count = requestCounts.computeIfAbsent(apiKey, k -> new AtomicInteger(0));
        return count.incrementAndGet() <= limitPerMinute;
    }
}
```

### Step 2: Create Configuration Properties
```java
package com.example.ratelimiter.autoconfigure;

import org.springframework.boot.context.properties.ConfigurationProperties;

@ConfigurationProperties(prefix = "app.rate-limiter")
public class RateLimiterProperties {
    private boolean enabled = true;
    private int limitPerMinute = 100;

    // Getters and Setters
    public boolean isEnabled() { return enabled; }
    public void setEnabled(boolean enabled) { this.enabled = enabled; }
    public int getLimitPerMinute() { return limitPerMinute; }
    public void setLimitPerMinute(int limitPerMinute) { this.limitPerMinute = limitPerMinute; }
}
```

### Step 3: Author the `@AutoConfiguration` Class
```java
package com.example.ratelimiter.autoconfigure;

import com.example.ratelimiter.InMemoryRateLimiterService;
import com.example.ratelimiter.RateLimiterService;
import org.springframework.boot.autoconfigure.AutoConfiguration;
import org.springframework.boot.autoconfigure.condition.ConditionalOnClass;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.boot.context.properties.EnableConfigurationProperties;
import org.springframework.context.annotation.Bean;

@AutoConfiguration
@ConditionalOnClass(RateLimiterService.class)
@EnableConfigurationProperties(RateLimiterProperties.class)
@ConditionalOnProperty(prefix = "app.rate-limiter", name = "enabled", havingValue = "true", matchIfMissing = true)
public class RateLimiterAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean(RateLimiterService.class)
    public RateLimiterService rateLimiterService(RateLimiterProperties properties) {
        return new InMemoryRateLimiterService(properties.getLimitPerMinute());
    }
}
```

### Step 4: Register in `AutoConfiguration.imports` (Spring Boot 3.x)
Create the file:
`src/main/resources/META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports`

Content:
```text
com.example.ratelimiter.autoconfigure.RateLimiterAutoConfiguration
```

*(For Spring Boot 2.x backward compatibility, also register it under `META-INF/spring.factories` under `org.springframework.boot.autoconfigure.EnableAutoConfiguration`)*.

---

## 8. Spring Boot 3.x Architectural Changes

Spring Boot 3.0 represents the largest evolutionary leap in the framework's history since Spring Boot 1.0.

```
+─────────────────────────────────────────────────────────────────────────────+
|                          Spring Boot 3.x Architecture                       |
+─────────────────────────────────────────────────────────────────────────────+
| Java 17 Baseline (Records, Pattern Matching, Sealed Classes, Text Blocks)   |
| Jakarta EE 10 Migration (javax.* -> jakarta.*)                              |
| GraalVM Native Image Support & Ahead-of-Time (AOT) Compilation              |
| Observability Engine: Micrometer Observation API (replaces Cloud Sleuth)    |
| HTTP Interfaces: Declarative HTTP Clients via @HttpExchange                 |
| Project Loom Virtual Threads Integration (Java 21)                          |
+─────────────────────────────────────────────────────────────────────────────+
```

### 8.1 Jakarta EE 10 Package Migration
All dependencies moved from `javax.*` to `jakarta.*`:
- `javax.servlet.*` -> `jakarta.servlet.*`
- `javax.persistence.*` -> `jakarta.persistence.*`
- `javax.validation.*` -> `jakarta.validation.*`
- `javax.annotation.*` -> `jakarta.annotation.*`

### 8.2 Ahead-of-Time (AOT) & GraalVM Native Images
Standard JVM applications rely on dynamic class loading, reflection, and runtime CGLIB proxy generation. GraalVM Native Image compilation relies on a **Closed World Assumption**: all reachable classes, reflections, and dynamic proxies must be known at compile time.

Spring Boot 3 introduces AOT processing:
1. At build time, `spring-boot-maven-plugin:aot-compile` evaluates `@Configuration` and condition evaluations ahead of time.
2. It generates direct Java code substituting reflection and dynamic proxies.
3. Native binaries boot in **under 50 milliseconds** with **under 50MB RAM footprint**.

#### Providing Runtime Hints for AOT:
When using custom reflection or serialization not detected by AOT:
```java
package com.example.aot;

import org.springframework.aot.hint.RuntimeHints;
import org.springframework.aot.hint.RuntimeHintsRegistrar;

public class CustomPayloadHints implements RuntimeHintsRegistrar {
    @Override
    public void registerHints(RuntimeHints hints, ClassLoader classLoader) {
        // Register reflection hints for GraalVM
        hints.reflection().registerType(PaymentPayload.class, 
            MemberCategory.INVOKE_DECLARED_CONSTRUCTORS, 
            MemberCategory.INVOKE_DECLARED_METHODS);
    }
}
```

---

## 9. Common Interview Questions & Answers

### Q1: What is the exact difference between `BeanFactoryPostProcessor` and `BeanPostProcessor`?
**Answer**:
- `BeanFactoryPostProcessor` (e.g., `ConfigurationClassPostProcessor`, `PropertySourcesPlaceholderConfigurer`): Operates on **bean definitions (metadata)** before any bean instances are created. It can modify bean definitions, add properties, or change scopes.
- `BeanPostProcessor` (e.g., `AutowiredAnnotationBeanPostProcessor`): Operates on **actual bean instances** after the JVM has instantiated the object. It intercepts lifecycle hooks before and after initialization (`postProcessBeforeInitialization` / `postProcessAfterInitialization`), enabling dependency injection, security checks, and AOP proxy generation.

### Q2: Why is `DeferredImportSelector` used instead of a standard `ImportSelector` for auto-configurations?
**Answer**:
A standard `ImportSelector` executes immediately during component scanning when the annotation is encountered. If auto-configurations ran immediately, `@ConditionalOnMissingBean` wouldn't know whether the user defined a custom bean later in the scanning process.
`DeferredImportSelector` guarantees that auto-configuration imports are deferred until **all other user `@Configuration` classes, component scans, and bean definitions have been fully registered**. This allows condition evaluations (such as `@ConditionalOnMissingBean`) to be 100% accurate.

### Q3: What happens when two auto-configurations conflict or have cyclical ordering dependencies?
**Answer**:
Spring Boot resolves auto-configuration ordering using:
1. `@AutoConfigureOrder(int)`: Integer precedence.
2. `@AutoConfigureBefore(Class<?>[] / String[])`
3. `@AutoConfigureAfter(Class<?>[] / String[])`

Spring internally constructs a directed graph of auto-configuration classes and performs a **topological sort**. If a circular dependency is detected (e.g., A before B, and B before A), Spring Boot's dependency sorter throws an `IllegalStateException` during context startup detailing the cycle.

### Q4: How does Spring Boot auto-configure an H2 In-Memory DataSource vs a HikariCP Production DataSource?
**Answer**:
`DataSourceAutoConfiguration` checks for the existence of `DataSource.class` on the classpath. It then delegates to `PooledDataSourceConfiguration` and `EmbeddedDatabaseConfiguration`:
1. If H2 or Derby is on the classpath and no `spring.datasource.url` is specified, `EmbeddedDatabaseConfiguration` boots an in-memory embedded database.
2. If `spring.datasource.url` is provided, `DataSourceConfiguration.Hikari` executes. It evaluates `@ConditionalOnClass(HikariDataSource.class)` and `@ConditionalOnProperty`. Since HikariCP is Spring Boot's default connection pool dependency, it instantiates `HikariDataSource` using the mapped properties.

### Q5: Can you run a Spring Boot Web application without an embedded Tomcat server?
**Answer**:
Yes, in multiple ways:
1. **Exclusion**: Exclude `spring-boot-starter-tomcat` from `spring-boot-starter-web` and add `spring-boot-starter-jetty` or `spring-boot-starter-undertow`.
2. **Reactive**: Use `spring-boot-starter-webflux`, which boots an embedded Netty server by default.
3. **Non-web**: Set `spring.main.web-application-type=none` or omit the web starter entirely; the application will start as a standard command-line or background worker app without any embedded HTTP server.

---

## 10. Gotchas & Production Pitfalls

1. **Calling `@Transactional` or `@Async` in `@PostConstruct`**:
   - *Problem*: In the bean lifecycle, `@PostConstruct` runs before `BeanPostProcessor.postProcessAfterInitialization()` creates the AOP proxy. The method call will execute on the raw unproxied instance—transactions and async execution will silently fail.
   - *Fix*: Use `ApplicationReadyEvent` or an `ApplicationRunner` to trigger post-initialization transactional workflows.

2. **Accidental Circular Dependencies with `@Lazy` Workarounds**:
   - Starting with Spring Boot 2.6+, circular dependencies between beans are forbidden by default (`spring.main.allow-circular-references=false`). Relying on `@Lazy` masks architectural design flaws where two services violate single-responsibility principles. Decouple services using events (`ApplicationEventPublisher`) or intermediary mediator beans.

3. **Memory Leaks from Misconfigured `@ConfigurationProperties`**:
   - When binding large nested dynamic structures (e.g., `Map<String, Object>`), unvalidated inputs from external config servers can lead to high Metaspace and Heap consumption if classes are dynamically generated.

4. **Slow Startup Times Due to Excessive Classpath Scanning**:
   - Defining `@ComponentScan` on top-level packages (e.g., `@ComponentScan("com")`) forces Spring to scan every single third-party library JAR on the classpath, resulting in multi-minute startup delays and massive CPU consumption. Always anchor scans to specific, granular domain packages.
