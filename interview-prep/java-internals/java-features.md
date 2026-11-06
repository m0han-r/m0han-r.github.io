← [Back to Index](../README.md)

# Java Features — Every LTS & Major Release

## Java 8 (LTS) — March 2014

### Lambda Expressions
Introduced functional programming concepts to Java, reducing boilerplate code.

**Before Java 8 (Anonymous Inner Class):**
```java
Runnable r = new Runnable() {
    @Override
    public void run() {
        System.out.println("Hello from inner class");
    }
};
```

**Java 8 (Lambda):**
```java
Runnable r = () -> System.out.println("Hello from lambda");
```

### Functional Interfaces
An interface with exactly one abstract method. Annotated with `@FunctionalInterface`.

* **Predicate<T>** (takes T, returns boolean):
  ```java
  Predicate<String> isLong = s -> s.length() > 5;
  System.out.println(isLong.test("Java 8")); // true
  ```
* **Function<T, R>** (takes T, returns R):
  ```java
  Function<String, Integer> lengthFunc = String::length;
  System.out.println(lengthFunc.apply("Hello")); // 5
  ```
* **Consumer<T>** (takes T, returns void):
  ```java
  Consumer<String> printer = System.out::println;
  printer.accept("Log message");
  ```
* **Supplier<T>** (takes nothing, returns T):
  ```java
  Supplier<Double> randomGen = Math::random;
  System.out.println(randomGen.get());
  ```

### Stream API
For functional-style operations on streams of elements.

* **map** (transform):
  ```java
  List<String> names = Arrays.asList("Alice", "Bob");
  List<Integer> lengths = names.stream().map(String::length).collect(Collectors.toList());
  ```
* **filter** (conditional extraction):
  ```java
  List<String> aNames = names.stream().filter(n -> n.startsWith("A")).collect(Collectors.toList());
  ```
* **reduce** (aggregate):
  ```java
  List<Integer> numbers = Arrays.asList(1, 2, 3);
  int sum = numbers.stream().reduce(0, Integer::sum);
  ```
* **collect** (terminal operation):
  ```java
  Set<String> set = names.stream().collect(Collectors.toSet());
  ```
* **flatMap** (flatten collections):
  ```java
  List<List<String>> nested = Arrays.asList(Arrays.asList("A"), Arrays.asList("B"));
  List<String> flat = nested.stream().flatMap(Collection::stream).collect(Collectors.toList());
  ```

### Optional
A container object which may or may not contain a non-null value, avoiding `NullPointerException`.

* **creation**:
  ```java
  Optional<String> emptyOpt = Optional.empty();
  Optional<String> opt = Optional.of("value"); // throws NPE if null
  Optional<String> nullableOpt = Optional.ofNullable(null); // safe
  ```
* **orElse / orElseGet / orElseThrow**:
  ```java
  String val1 = nullableOpt.orElse("default");
  String val2 = nullableOpt.orElseGet(() -> "expensive default");
  String val3 = nullableOpt.orElseThrow(() -> new IllegalArgumentException());
  ```
* **map / flatMap**:
  ```java
  Optional<Integer> length = opt.map(String::length);
  ```
* **ifPresent**:
  ```java
  opt.ifPresent(System.out::println);
  ```

### Default & Static methods in interfaces
Allows adding new methods to interfaces without breaking implementing classes.

```java
public interface Vehicle {
    void start();
    
    default void honk() {
        System.out.println("Beep beep!");
    }
    
    static void checkEngine() {
        System.out.println("Engine OK");
    }
}
```

### Method References
Shorthand syntax for lambdas calling a specific method.

1. **Static method**: `Math::max`
2. **Instance method of an object**: `System.out::println`
3. **Instance method of an arbitrary object of a particular type**: `String::length`
4. **Constructor reference**: `ArrayList::new`

### New Date/Time API
Immutable and thread-safe API (JSR-310).

```java
LocalDate date = LocalDate.now(); // 2024-05-15
LocalTime time = LocalTime.now(); // 14:30:00
LocalDateTime dateTime = LocalDateTime.now();
ZonedDateTime zdt = ZonedDateTime.now(ZoneId.of("Europe/Paris"));

Period tenDays = Period.ofDays(10);
LocalDate future = date.plus(tenDays);

Duration twoHours = Duration.ofHours(2);
LocalTime futureTime = time.plus(twoHours);
```

### CompletableFuture basics
Enhancement over Future for asynchronous programming.

```java
CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
    // long running task
    return "Result";
}).thenApply(String::toUpperCase);

System.out.println(future.join()); // RESULT
```

### Nashorn JavaScript Engine
Allowed executing JS from Java (removed in Java 15).

---

## Java 9 — September 2017

### Module System (JPMS)
Project Jigsaw. Introduces `module-info.java` to explicitly declare dependencies and exports.

```java
// module-info.java
module com.myapp {
    requires java.sql;
    requires transitive com.mylib; // implies dependents of myapp also get mylib
    exports com.myapp.api; // makes package available
    opens com.myapp.internal to com.testing.framework; // for reflection
}
```

### JShell (REPL)
Interactive Read-Eval-Print Loop for testing Java code quickly via terminal `jshell`.

### Collection Factory Methods
Immutable collection creation.

```java
List<String> list = List.of("A", "B", "C"); // Immutable
Set<String> set = Set.of("X", "Y");
Map<String, Integer> map = Map.of("K1", 1, "K2", 2);
```

### Stream improvements

* **takeWhile / dropWhile**:
  ```java
  Stream.of(1, 2, 3, 4, 5).takeWhile(i -> i < 3).forEach(System.out::print); // 12
  Stream.of(1, 2, 3, 4, 5).dropWhile(i -> i < 3).forEach(System.out::print); // 345
  ```
* **ofNullable**:
  ```java
  Stream.ofNullable(null).count(); // 0
  ```
* **iterate with predicate**:
  ```java
  Stream.iterate(1, i -> i < 10, i -> i + 2).forEach(System.out::print); // 13579
  ```

### Optional improvements

* **ifPresentOrElse**:
  ```java
  Optional.ofNullable(val).ifPresentOrElse(
      v -> System.out.println("Found: " + v),
      () -> System.out.println("Not found")
  );
  ```
* **or**:
  ```java
  Optional<String> res = opt.or(() -> Optional.of("backup"));
  ```
* **stream**:
  ```java
  List<String> vals = listOfOptionals.stream()
      .flatMap(Optional::stream)
      .collect(Collectors.toList());
  ```

### Private methods in interfaces
Used to share code between default methods.

```java
public interface Logger {
    default void logInfo(String msg) { log("INFO", msg); }
    default void logError(String msg) { log("ERROR", msg); }
    
    private void log(String level, String msg) {
        System.out.println(level + ": " + msg);
    }
}
```

### Try-with-resources enhancement
Allows using effectively final variables in try-with-resources.

```java
BufferedReader reader = new BufferedReader(new FileReader("file.txt"));
// Java 9+ syntax
try (reader) {
    System.out.println(reader.readLine());
} // reader closed automatically
```

### Process API improvements
Easier interaction with OS processes.

```java
ProcessHandle current = ProcessHandle.current();
System.out.println("PID: " + current.pid());
```

### Reactive Streams (Flow API)
Standardized interfaces for reactive programming (`Flow.Publisher`, `Flow.Subscriber`, `Flow.Subscription`, `Flow.Processor`).

---

## Java 10 — March 2018

### Local Variable Type Inference (`var`)
Reduces boilerplate. Compiler infers type. Only for local variables!

**When to use:**
```java
// Good: clear from RHS
var list = new ArrayList<String>(); 
var stream = list.stream();
```

**When NOT to use:**
```java
// Bad: poor readability
var result = processData(); // What type is this?

// Not allowed:
// var emptyList; // Must be initialized
// var lambda = () -> System.out.println("hi"); // Cannot infer lambda
```

### Unmodifiable Collections
```java
List<String> copy = List.copyOf(originalList); // Immutable copy
Set<String> set = stream.collect(Collectors.toUnmodifiableSet());
```

### Application Class-Data Sharing (AppCDS)
Improves startup time and reduces footprint by sharing class metadata across JVMs.

### G1 parallel full GC
Improves worst-case latencies of G1 by making Full GC parallel.

---

## Java 11 (LTS) — September 2018

### `var` in lambda parameters
Allows adding annotations to lambda parameters.

```java
BiFunction<Integer, Integer, Integer> adder = 
    (@Nonnull var a, @Nonnull var b) -> a + b;
```

### New String methods
```java
" ".isBlank(); // true
" foo ".strip(); // "foo" (Unicode aware trim)
"a\nb".lines().forEach(System.out::println);
"A".repeat(3); // "AAA"
```

### Files.readString() and writeString()
```java
Path path = Path.of("test.txt");
Files.writeString(path, "Hello Java 11");
String content = Files.readString(path);
```

### HttpClient API (standard)
Replaces `HttpURLConnection` with a modern, asynchronous API.

**GET:**
```java
HttpClient client = HttpClient.newHttpClient();
HttpRequest request = HttpRequest.newBuilder(URI.create("https://api.github.com")).build();
HttpResponse<String> response = client.send(request, HttpResponse.BodyHandlers.ofString());
System.out.println(response.body());
```

**POST Async:**
```java
HttpRequest postRequest = HttpRequest.newBuilder(URI.create("https://httpbin.org/post"))
    .POST(HttpRequest.BodyPublishers.ofString("data"))
    .build();
client.sendAsync(postRequest, HttpResponse.BodyHandlers.ofString())
    .thenApply(HttpResponse::body)
    .thenAccept(System.out::println);
```

### Optional.isEmpty()
```java
Optional<String> opt = Optional.empty();
if (opt.isEmpty()) { ... } // Inverse of isPresent()
```

### Collection.toArray(IntFunction)
```java
List<String> list = List.of("A", "B");
String[] arr = list.toArray(String[]::new);
```

### Running single-file source code
```bash
java HelloWorld.java
```
Directly compiles and runs in memory.

### Nest-based access control
JVM-level support for nested classes accessing each other's private members without synthetic bridge methods.

### Epsilon GC & ZGC (experimental)
Epsilon is a no-op GC (allocates but doesn't collect) for testing.

---

## Java 12-16 (Key Features)

* **Java 12: Switch Expressions (preview)**
  ```java
  int numLetters = switch (day) {
      case MONDAY, FRIDAY, SUNDAY -> 6;
      case TUESDAY                -> 7;
      default      -> {
          int k = day.toString().length();
          yield k;
      }
  };
  ```

* **Java 13: Text Blocks (preview)**
  ```java
  String html = """
                <html>
                    <body>
                        <p>Hello, world</p>
                    </body>
                </html>
                """;
  ```

* **Java 14: Helpful NullPointerExceptions**
  JVM points exactly to what was null: `Cannot invoke "String.length()" because "name" is null`

* **Java 16: Stream.toList()**
  ```java
  List<String> list = stream.toList(); // Immutable, shorter than collect(Collectors.toList())
  ```

---

## Java 17 (LTS) — September 2021

### Sealed Classes
Restricts which classes can extend or implement them, enabling domain modeling and exhaustive pattern matching.

```java
public sealed interface Shape permits Circle, Square, Rectangle {}

// Must be final, sealed, or non-sealed
public final class Circle implements Shape {}
public non-sealed class Square implements Shape {}
public sealed class Rectangle implements Shape permits TransparentRectangle {}
public final class TransparentRectangle extends Rectangle {}
```

### Pattern Matching for instanceof
Removes need for explicit casting.

```java
Object obj = "Hello";
if (obj instanceof String s) { // s is in scope here
    System.out.println(s.length());
}
```

### Enhanced switch with pattern matching (preview in 17, standard later)
Allows switching on types.

```java
String format(Object obj) {
    return switch (obj) {
        case Integer i -> String.format("int %d", i);
        case Long l    -> String.format("long %d", l);
        case Double d  -> String.format("double %f", d);
        case String s  -> String.format("String %s", s);
        default        -> obj.toString();
    };
}
```

### Text Blocks (Standard)
Multi-line strings, avoiding escaping nightmares (e.g., JSON, SQL).

```java
String json = """
    {
        "name": "Java",
        "version": 17
    }
    """;
```

### Records (Standard)
Transparent carriers for immutable data. Generates constructor, getters, `equals()`, `hashCode()`, `toString()`.

```java
public record User(String name, int age) {
    // Compact constructor for validation
    public User {
        if (age < 0) throw new IllegalArgumentException("Age cannot be negative");
    }
    
    // Custom method
    public boolean isAdult() {
        return age >= 18;
    }
}

// Usage
User u = new User("Alice", 30);
System.out.println(u.name()); // "Alice"
```

### Strong encapsulation of JDK internals
`--illegal-access` is removed. You can no longer forcefully reflect into JDK internal APIs (e.g., `sun.misc.Unsafe`) without explicit `--add-opens` flags.

---

## Java 18-20 (Key Features)

* **Java 18: Simple Web Server** (`jwebserver` tool) for static files. Default charset is now **UTF-8** natively!
* **Java 19 & 20: Previews** of Virtual Threads, Structured Concurrency, Scoped Values, Record Patterns.

---

## Java 21 (LTS) — September 2023

### Virtual Threads (Project Loom)
Lightweight threads managed by the JVM, not mapped 1:1 to OS threads. Solves the blocking I/O bottleneck.

**When to use:** IO-bound workloads (web servers, database queries).
**When NOT to use:** CPU-bound tasks (use parallel streams instead). DO NOT pool virtual threads.

```java
// Creating virtual threads
Thread vThread = Thread.ofVirtual().start(() -> {
    System.out.println("Running in virtual thread: " + Thread.currentThread());
});

// Using Executors
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    IntStream.range(0, 10_000).forEach(i -> {
        executor.submit(() -> {
            Thread.sleep(Duration.ofSeconds(1)); // Does not block an OS thread!
            return i;
        });
    });
} // Waits for all to finish
```

### Pattern Matching for switch (Standard)
Compiler checks exhaustiveness for sealed classes/enums.

```java
sealed interface Response permits Success, Error {}
record Success(String data) implements Response {}
record Error(int code, String msg) implements Response {}

String handle(Response r) {
    // Exhaustive, no default needed
    return switch (r) {
        case Success(var data) -> "Got: " + data;
        case Error(int code, String msg) when code == 404 -> "Not found: " + msg; // Guard clause (when)
        case Error e -> "Other error: " + e.code();
    };
}
```

### Record Patterns (Standard)
Destructuring records directly in `instanceof` or `switch`.

```java
record Point(int x, int y) {}

void print(Object obj) {
    if (obj instanceof Point(int x, int y)) { // Destructuring
        System.out.println(x + ", " + y);
    }
}
```

### Sequenced Collections
Unifies collections that have a defined encounter order (first/last elements, reversing).

```java
SequencedCollection<String> list = new ArrayList<>();
list.addFirst("First");
list.addLast("Last");
String first = list.getFirst();
SequencedCollection<String> reversed = list.reversed();

// Also SequencedSet and SequencedMap
SequencedMap<String, String> lhm = new LinkedHashMap<>();
lhm.putFirst("A", "1");
```

### String Templates (Preview)
Safe and clean string interpolation.

```java
String name = "Joan";
int age = 25;
// STR processor (interpolates variables)
String info = STR."My name is \{name} and I am \{age} years old.";

// FMT processor (formats)
String table = FMT."%-10s\{name} | %04d\{age}"; 
```

### Structured Concurrency (Preview)
Treats multiple concurrent tasks running in different threads as a single unit of work (clean cancellation, error handling).

```java
<T> T process() throws Exception {
    try (var scope = new StructuredTaskScope.ShutdownOnFailure()) {
        Subtask<String> user = scope.fork(() -> fetchUser());
        Subtask<Integer> order = scope.fork(() -> fetchOrder());

        scope.join();          // Wait for both
        scope.throwIfFailed(); // If one fails, the other is cancelled

        return new Result(user.get(), order.get());
    }
}
```

### Scoped Values (Preview)
Modern replacement for `ThreadLocal`. Immutable and scoped. Perfect for Virtual Threads.

```java
public static final ScopedValue<String> USERNAME = ScopedValue.newInstance();

ScopedValue.where(USERNAME, "admin").run(() -> {
    // Inside this block, and any nested method/thread, USERNAME is "admin"
    processRequest();
});

void processRequest() {
    System.out.println(USERNAME.get());
}
```

### Unnamed Patterns and Variables `_`
Using `_` to discard variables.

```java
// Unnamed pattern
if (obj instanceof Point(int x, _)) { ... }

// Unnamed variable
try {
    int i = Integer.parseInt(s);
} catch (NumberFormatException _) { // Exception ignored
    System.out.println("Invalid");
}
```

### Unnamed Classes and Instance Main Methods (Preview)
Simplifies entry point for beginners.

```java
// No class declaration needed!
void main() {
    System.out.println("Hello World");
}
```

---

## Java 22-25 (Latest & Future LTS)

* **Java 22:** Unnamed Variables (standard). Statements before `super(...)` (preview) allows logic before calling super constructor.
* **Java 23:** Primitive Types in Patterns (preview), Markdown doc comments (`///` instead of `/** */`).
* **Java 24:** Stream Gatherers (standard) — allows custom intermediate stream operations (e.g., `windowFixed`, `fold`). Flexible Constructor Bodies (standard).
* **Java 25 (Expected LTS - Sept 2025):** Will likely finalize Structured Concurrency, Scoped Values, and Stable Value.

---

## Garbage Collection Evolution

### Timeline
* **Serial GC:** Single-threaded, stop-the-world. Good for tiny heaps (<100MB).
* **Parallel GC:** Multi-threaded throughput collector. Default until Java 8. High throughput, higher latency.
* **CMS (Concurrent Mark Sweep):** Low latency, deprecated in Java 9, removed in 14. Prone to fragmentation.
* **G1 GC (Garbage-First):** Default since Java 9. Balances latency and throughput. Partitions heap into regions.
* **ZGC (Z Garbage Collector):** Sub-millisecond pauses. Scales to Terabytes. Production ready in 15.
* **Shenandoah GC:** Red Hat's ultra-low pause GC. Similar goals to ZGC.

### G1 GC Tuning
G1 tries to meet a pause time target.
```bash
# Enable G1 (default in 9+)
-XX:+UseG1GC 
# Set target pause time (default is 200ms)
-XX:MaxGCPauseMillis=50
# Set heap size
-Xms4G -Xmx4G
```

### ZGC
Performs all expensive work concurrently, without stopping application threads for more than a millisecond.
```bash
# Enable ZGC
-XX:+UseZGC
# Generational ZGC (Java 21 default for ZGC)
-XX:+ZGenerational
```

---

## Version Selection Guide

| Java Version | Status | When to use / Migration considerations |
|--------------|--------|----------------------------------------|
| **Java 8** | LTS | Legacy apps only. Migrate off due to lack of modern features and performance improvements. |
| **Java 11** | LTS | Older stable systems. Solid module system and GC. Still widely used. |
| **Java 17** | LTS | **Standard baseline** for most modern enterprise apps. Gets you Records, Text Blocks, Switch Expressions. Spring Boot 3 requires Java 17+. |
| **Java 21** | LTS | **Highly Recommended**. Huge upgrade due to Virtual Threads, Sequenced Collections, and full Pattern Matching. Best for heavy IO applications. |
| **Java 25** | Future LTS | Look forward to it in Sept 2025 for final Structured Concurrency and Value object previews. |
