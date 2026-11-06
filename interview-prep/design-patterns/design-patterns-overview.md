← [Back to Index](../README.md)

# Design Patterns — Complete Reference with Java Examples

This is a comprehensive reference for the 23 Gang of Four (GoF) design patterns and their application in modern Java development. 

---

## Creational Patterns (5)

Creational design patterns deal with object creation mechanisms, trying to create objects in a manner suitable to the situation. 

### 1. Singleton
- **Problem it solves:** Ensures a class has only one instance, and provides a global point of access to it.
- **UML-style description:** A single class with a private constructor, a static private instance variable, and a public static getter method.

#### Implementation
- **Eager:** Instance created at class loading time. Thread-safe but creates the instance even if unused.
- **Lazy:** Instance created when first requested. Not thread-safe without synchronization.
- **Double-checked locking:** Uses `volatile` keyword to ensure that multiple threads handle the singleton instance correctly when it is being initialized.
- **Bill Pugh (Static Inner Class):** Relies on class initialization phase for thread safety. Highly efficient.
- **Enum:** Best way to create a singleton in Java since Java 1.5. Protects against serialization and reflection attacks.

#### Full Java Code (Bill Pugh + Enum)

```java
// Bill Pugh Singleton Approach
public class DatabaseConnection {
    
    private DatabaseConnection() {
        // Prevent reflection instantiation
        if (ConnectionHolder.INSTANCE != null) {
            throw new IllegalStateException("Already initialized");
        }
    }
    
    // Static inner class is not loaded until getInstance() is called
    private static class ConnectionHolder {
        private static final DatabaseConnection INSTANCE = new DatabaseConnection();
    }
    
    public static DatabaseConnection getInstance() {
        return ConnectionHolder.INSTANCE;
    }
    
    public void query(String sql) {
        System.out.println("Executing: " + sql);
    }
}

// Enum Singleton Approach
public enum ConfigurationManager {
    INSTANCE;
    
    private Properties properties = new Properties();
    
    private ConfigurationManager() {
        // Load properties
        properties.setProperty("version", "1.0");
    }
    
    public String getProperty(String key) {
        return properties.getProperty(key);
    }
}
```
- **Thread Safety:** Both approaches above are inherently thread-safe. Double-checked locking requires the instance variable to be `volatile` to prevent the JVM from reordering instructions which could result in returning a partially initialized object.
- **Spring Connection:** By default, all Spring beans are Singleton scoped. Spring manages the lifecycle and ensures only one instance exists in the ApplicationContext.

### 2. Factory Method
- **Problem:** Creates objects without exposing the instantiation logic to the client. Defers instantiation to subclasses.
- **When to use:** When a class cannot anticipate the type of objects it needs to create, or when you want subclasses to specify the objects they create.
- **Connection to SOLID:** Open/Closed Principle (open for extension, closed for modification).

#### Full Java Code

```java
public interface Notification {
    void notifyUser(String message);
}

public class EmailNotification implements Notification {
    @Override
    public void notifyUser(String message) {
        System.out.println("Sending an e-mail notification: " + message);
    }
}

public class SMSNotification implements Notification {
    @Override
    public void notifyUser(String message) {
        System.out.println("Sending an SMS notification: " + message);
    }
}

public abstract class NotificationFactory {
    // The Factory Method
    public abstract Notification createNotification();
    
    public void notify(String message) {
        Notification notification = createNotification();
        notification.notifyUser(message);
    }
}

public class EmailNotificationFactory extends NotificationFactory {
    @Override
    public Notification createNotification() {
        return new EmailNotification();
    }
}

public class SMSNotificationFactory extends NotificationFactory {
    @Override
    public Notification createNotification() {
        return new SMSNotification();
    }
}

// Usage
NotificationFactory factory = new EmailNotificationFactory();
factory.notify("Hello User!"); // Output: Sending an e-mail notification: Hello User!
```

### 3. Abstract Factory
- **Problem:** Provide an interface for creating families of related or dependent objects without specifying their concrete classes.
- **When to use:** When the system needs to be independent of how its products are created, composed, and represented, and configured with one of multiple families of products.
- **Difference from Factory Method:** Factory Method uses inheritance and relies on a subclass to handle object creation (one product). Abstract Factory uses object composition to create a family of related objects.

#### Full Java Code

```java
// Abstract Products
public interface Button { void paint(); }
public interface Checkbox { void paint(); }

// Concrete Products for Mac
public class MacButton implements Button {
    public void paint() { System.out.println("Mac Button"); }
}
public class MacCheckbox implements Checkbox {
    public void paint() { System.out.println("Mac Checkbox"); }
}

// Concrete Products for Windows
public class WindowsButton implements Button {
    public void paint() { System.out.println("Windows Button"); }
}
public class WindowsCheckbox implements Checkbox {
    public void paint() { System.out.println("Windows Checkbox"); }
}

// Abstract Factory
public interface GUIFactory {
    Button createButton();
    Checkbox createCheckbox();
}

// Concrete Factories
public class MacFactory implements GUIFactory {
    public Button createButton() { return new MacButton(); }
    public Checkbox createCheckbox() { return new MacCheckbox(); }
}

public class WindowsFactory implements GUIFactory {
    public Button createButton() { return new WindowsButton(); }
    public Checkbox createCheckbox() { return new WindowsCheckbox(); }
}

// Usage
public class Application {
    private Button button;
    private Checkbox checkbox;

    public Application(GUIFactory factory) {
        button = factory.createButton();
        checkbox = factory.createCheckbox();
    }

    public void paint() {
        button.paint();
        checkbox.paint();
    }
}
```

### 4. Builder
- **Problem:** Separate the construction of a complex object from its representation.
- **When to use:** When you have many constructor parameters (telescoping constructor anti-pattern), especially optional ones, and you want immutable objects.
- **Effective Java Style:** Static inner class.
- **Lombok:** Can be generated via `@Builder`.

#### Full Java Code

```java
public class User {
    private final String firstName; // required
    private final String lastName;  // required
    private final int age;          // optional
    private final String phone;     // optional
    private final String address;   // optional

    private User(UserBuilder builder) {
        this.firstName = builder.firstName;
        this.lastName = builder.lastName;
        this.age = builder.age;
        this.phone = builder.phone;
        this.address = builder.address;
    }

    public static class UserBuilder {
        private final String firstName;
        private final String lastName;
        private int age;
        private String phone;
        private String address;

        public UserBuilder(String firstName, String lastName) {
            this.firstName = firstName;
            this.lastName = lastName;
        }

        public UserBuilder age(int age) {
            this.age = age;
            return this;
        }

        public UserBuilder phone(String phone) {
            this.phone = phone;
            return this;
        }

        public UserBuilder address(String address) {
            this.address = address;
            return this;
        }

        public User build() {
            return new User(this);
        }
    }
    
    @Override
    public String toString() {
        return "User: " + firstName + " " + lastName + ", Age: " + age;
    }
}

// Usage
User user = new User.UserBuilder("John", "Doe")
                    .age(30)
                    .phone("1234567890")
                    .build();
```

### 5. Prototype
- **Problem:** Specify the kinds of objects to create using a prototypical instance, and create new objects by copying this prototype.
- **When to use:** When creating an object is expensive (e.g., database calls) and we want to clone a pre-configured object.
- **Shallow vs Deep Copy:** Shallow copy duplicates references; deep copy duplicates the underlying objects. `Cloneable` is Java's built-in marker interface for this (though copy constructors or factories are generally preferred in modern Java).

#### Full Java Code

```java
import java.util.ArrayList;
import java.util.List;

public class EmployeeRecord implements Cloneable {
    private String id;
    private String name;
    private List<String> permissions;

    public EmployeeRecord(String id, String name, List<String> permissions) {
        this.id = id;
        this.name = name;
        this.permissions = permissions;
        // Assume simulating an expensive DB call here
    }

    // Deep Copy using Clone
    @Override
    public EmployeeRecord clone() {
        try {
            EmployeeRecord clone = (EmployeeRecord) super.clone();
            // Deep copy the list to ensure independence
            clone.permissions = new ArrayList<>(this.permissions);
            return clone;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }

    // Modern Java alternative: Copy Constructor
    public EmployeeRecord(EmployeeRecord target) {
        this.id = target.id;
        this.name = target.name;
        this.permissions = new ArrayList<>(target.permissions);
    }
}
```

---

## Structural Patterns (7)

Structural patterns deal with object composition or how classes and objects are composed to form larger structures.

### 6. Adapter
- **Problem:** Convert the interface of a class into another interface clients expect. Adapter lets classes work together that couldn't otherwise because of incompatible interfaces.
- **Class vs Object Adapter:** Class adapter uses multiple inheritance (not feasible in Java for classes, but possible with interfaces). Object adapter uses composition.

#### Full Java Code

```java
// Target Interface
public interface ModernPaymentGateway {
    void processModernPayment(String accountId, double amount);
}

// Adaptee (Legacy)
public class LegacyPaymentProcessor {
    public void doLegacyPay(int acctNum, float amt) {
        System.out.println("Legacy pay: " + amt + " for " + acctNum);
    }
}

// Object Adapter
public class PaymentAdapter implements ModernPaymentGateway {
    private final LegacyPaymentProcessor legacyProcessor;

    public PaymentAdapter(LegacyPaymentProcessor legacyProcessor) {
        this.legacyProcessor = legacyProcessor;
    }

    @Override
    public void processModernPayment(String accountId, double amount) {
        int acctNum = Integer.parseInt(accountId);
        float amt = (float) amount;
        legacyProcessor.doLegacyPay(acctNum, amt);
    }
}
```

### 7. Bridge
- **Problem:** Decouple an abstraction from its implementation so that the two can vary independently. Prefer composition over inheritance.
- **When to use:** When you have orthogonal dimensions (e.g., Shape and Color) and you want to prevent a Cartesian product explosion of classes (e.g., RedCircle, BlueCircle, RedSquare...).

#### Full Java Code

```java
// Implementor
public interface Color {
    void applyColor();
}

// Concrete Implementors
public class RedColor implements Color {
    public void applyColor() { System.out.println("red."); }
}

public class GreenColor implements Color {
    public void applyColor() { System.out.println("green."); }
}

// Abstraction
public abstract class Shape {
    protected Color color;
    
    public Shape(Color color) {
        this.color = color;
    }
    
    abstract public void applyColor();
}

// Refined Abstraction
public class Triangle extends Shape {
    public Triangle(Color color) {
        super(color);
    }
    
    @Override
    public void applyColor() {
        System.out.print("Triangle filled with color ");
        color.applyColor();
    }
}
```

### 8. Composite
- **Problem:** Compose objects into tree structures to represent part-whole hierarchies. Composite lets clients treat individual objects and compositions of objects uniformly.
- **When to use:** Menus, UI components, File Systems.

#### Full Java Code

```java
import java.util.ArrayList;
import java.util.List;

// Component
public interface FileSystemNode {
    void printName();
    int getSize();
}

// Leaf
public class File implements FileSystemNode {
    private String name;
    private int size;

    public File(String name, int size) {
        this.name = name;
        this.size = size;
    }

    @Override
    public void printName() { System.out.println("File: " + name); }

    @Override
    public int getSize() { return size; }
}

// Composite
public class Directory implements FileSystemNode {
    private String name;
    private List<FileSystemNode> children = new ArrayList<>();

    public Directory(String name) {
        this.name = name;
    }

    public void add(FileSystemNode node) { children.add(node); }

    @Override
    public void printName() {
        System.out.println("Directory: " + name);
        for (FileSystemNode child : children) {
            child.printName();
        }
    }

    @Override
    public int getSize() {
        return children.stream().mapToInt(FileSystemNode::getSize).sum();
    }
}
```

### 9. Decorator
- **Problem:** Attach additional responsibilities to an object dynamically. Decorators provide a flexible alternative to subclassing for extending functionality.
- **Difference from Inheritance:** Applied at runtime to individual objects, not classes. Prevents class explosion.
- **JDK Example:** `java.io.InputStream` (`BufferedInputStream`, `DataInputStream`).

#### Full Java Code

```java
// Component
public interface Coffee {
    double getCost();
    String getDescription();
}

// Concrete Component
public class SimpleCoffee implements Coffee {
    @Override
    public double getCost() { return 2.0; }
    
    @Override
    public String getDescription() { return "Simple Coffee"; }
}

// Base Decorator
public abstract class CoffeeDecorator implements Coffee {
    protected final Coffee decoratedCoffee;

    public CoffeeDecorator(Coffee coffee) {
        this.decoratedCoffee = coffee;
    }

    @Override
    public double getCost() { return decoratedCoffee.getCost(); }

    @Override
    public String getDescription() { return decoratedCoffee.getDescription(); }
}

// Concrete Decorator
public class MilkDecorator extends CoffeeDecorator {
    public MilkDecorator(Coffee coffee) { super(coffee); }

    @Override
    public double getCost() { return super.getCost() + 0.5; }

    @Override
    public String getDescription() { return super.getDescription() + ", Milk"; }
}

// Usage
Coffee coffee = new SimpleCoffee();
coffee = new MilkDecorator(coffee);
System.out.println(coffee.getDescription() + " $" + coffee.getCost());
```

### 10. Facade
- **Problem:** Provide a unified interface to a set of interfaces in a subsystem. Facade defines a higher-level interface that makes the subsystem easier to use.
- **When to use:** To simplify complex systems for clients, providing a clear entry point.

#### Full Java Code

```java
// Subsystem classes
class InventorySystem {
    public boolean checkStock(String item) { return true; }
}

class PaymentSystem {
    public boolean processPayment(String account, double amount) { return true; }
}

class ShippingSystem {
    public void shipItem(String item, String address) { 
        System.out.println("Shipping " + item + " to " + address); 
    }
}

// Facade
public class OrderFacade {
    private InventorySystem inventory;
    private PaymentSystem payment;
    private ShippingSystem shipping;

    public OrderFacade() {
        inventory = new InventorySystem();
        payment = new PaymentSystem();
        shipping = new ShippingSystem();
    }

    public boolean placeOrder(String item, double amount, String account, String address) {
        if (inventory.checkStock(item)) {
            if (payment.processPayment(account, amount)) {
                shipping.shipItem(item, address);
                return true;
            }
        }
        return false;
    }
}
```

### 11. Flyweight
- **Problem:** Use sharing to support large numbers of fine-grained objects efficiently.
- **When to use:** Memory optimization. When you have objects with Extrinsic state (varies) and Intrinsic state (shared).
- **JDK Example:** `String` pool, `Integer.valueOf()` cache.

#### Full Java Code

```java
import java.util.HashMap;
import java.util.Map;

// Flyweight
public interface CharacterFlyweight {
    void display(int x, int y); // x, y are extrinsic state
}

// Concrete Flyweight (Intrinsic state shared)
public class ConcreteCharacter implements CharacterFlyweight {
    private final char characterSymbol; // Intrinsic state
    private final String font;

    public ConcreteCharacter(char symbol, String font) {
        this.characterSymbol = symbol;
        this.font = font;
    }

    @Override
    public void display(int x, int y) {
        System.out.println("Character " + characterSymbol + " at (" + x + "," + y + ")");
    }
}

// Flyweight Factory
public class CharacterFactory {
    private Map<Character, CharacterFlyweight> characters = new HashMap<>();

    public CharacterFlyweight getCharacter(char key, String font) {
        return characters.computeIfAbsent(key, k -> new ConcreteCharacter(k, font));
    }
}
```

### 12. Proxy
- **Problem:** Provide a surrogate or placeholder for another object to control access to it.
- **Types:** Virtual (lazy initialization), Protection (access control), Remote, Caching.
- **Spring Connection:** Spring AOP extensively uses JDK Dynamic Proxies or CGLIB proxies.

#### Full Java Code

```java
// Subject
public interface Image {
    void display();
}

// Real Subject
public class RealImage implements Image {
    private String filename;

    public RealImage(String filename) {
        this.filename = filename;
        loadFromDisk(filename);
    }

    private void loadFromDisk(String filename) {
        System.out.println("Loading " + filename);
    }

    @Override
    public void display() {
        System.out.println("Displaying " + filename);
    }
}

// Proxy (Virtual Proxy for Lazy Loading)
public class ProxyImage implements Image {
    private RealImage realImage;
    private String filename;

    public ProxyImage(String filename) {
        this.filename = filename;
    }

    @Override
    public void display() {
        if (realImage == null) {
            realImage = new RealImage(filename);
        }
        realImage.display();
    }
}

/* JDK Dynamic Proxy Example snippet:
Image proxyInstance = (Image) Proxy.newProxyInstance(
    Image.class.getClassLoader(),
    new Class<?>[]{Image.class},
    (proxy, method, args) -> {
        System.out.println("Before method");
        Object result = method.invoke(new RealImage("test.jpg"), args);
        System.out.println("After method");
        return result;
    }
);
*/
```

---

## Behavioral Patterns (11)

Behavioral patterns are concerned with algorithms and the assignment of responsibilities between objects.

### 13. Chain of Responsibility
- **Problem:** Avoid coupling the sender of a request to its receiver by giving more than one object a chance to handle the request. Chain the receiving objects and pass the request along the chain.
- **Connection:** Servlet Filters, Spring Interceptors, Exception Handling.

#### Full Java Code

```java
public abstract class Logger {
    public static int INFO = 1;
    public static int DEBUG = 2;
    public static int ERROR = 3;

    protected int level;
    protected Logger nextLogger;

    public void setNextLogger(Logger nextLogger) {
        this.nextLogger = nextLogger;
    }

    public void logMessage(int level, String message) {
        if (this.level <= level) {
            write(message);
        }
        if (nextLogger != null) {
            nextLogger.logMessage(level, message);
        }
    }

    abstract protected void write(String message);
}

public class ConsoleLogger extends Logger {
    public ConsoleLogger(int level) { this.level = level; }
    @Override protected void write(String message) { System.out.println("Console: " + message); }
}

public class ErrorLogger extends Logger {
    public ErrorLogger(int level) { this.level = level; }
    @Override protected void write(String message) { System.out.println("Error: " + message); }
}

// Usage
Logger errorLogger = new ErrorLogger(Logger.ERROR);
Logger consoleLogger = new ConsoleLogger(Logger.INFO);
errorLogger.setNextLogger(consoleLogger);
errorLogger.logMessage(Logger.ERROR, "System Crash");
```

### 14. Command
- **Problem:** Encapsulate a request as an object, thereby letting you parameterize clients with different requests, queue or log requests, and support undoable operations.
- **Macro Commands:** A Command that executes a list of Commands.

#### Full Java Code

```java
// Command
public interface Command {
    void execute();
    void undo();
}

// Receiver
public class TextEditor {
    private StringBuilder text = new StringBuilder();
    public void append(String s) { text.append(s); }
    public void deleteLast(int length) { text.delete(text.length() - length, text.length()); }
    public String getText() { return text.toString(); }
}

// Concrete Command
public class AppendTextCommand implements Command {
    private TextEditor editor;
    private String textToAppend;

    public AppendTextCommand(TextEditor editor, String text) {
        this.editor = editor;
        this.textToAppend = text;
    }

    @Override
    public void execute() { editor.append(textToAppend); }

    @Override
    public void undo() { editor.deleteLast(textToAppend.length()); }
}

// Invoker
import java.util.Stack;
public class CommandInvoker {
    private Stack<Command> history = new Stack<>();

    public void executeCommand(Command cmd) {
        cmd.execute();
        history.push(cmd);
    }

    public void undo() {
        if (!history.isEmpty()) {
            history.pop().undo();
        }
    }
}
```

### 15. Iterator
- **Problem:** Provide a way to access the elements of an aggregate object sequentially without exposing its underlying representation.
- **JDK Connection:** `java.util.Iterator`, `Iterable` interface, enhanced for-loop.

#### Full Java Code

```java
import java.util.Iterator;
import java.util.NoSuchElementException;

// Aggregate
public class NameRepository implements Iterable<String> {
    private String[] names = {"John", "Jane", "Bob"};

    @Override
    public Iterator<String> iterator() {
        return new NameIterator();
    }

    // Concrete Iterator
    private class NameIterator implements Iterator<String> {
        private int index = 0;

        @Override
        public boolean hasNext() {
            return index < names.length;
        }

        @Override
        public String next() {
            if (this.hasNext()) {
                return names[index++];
            }
            throw new NoSuchElementException();
        }
    }
}
```

### 16. Mediator
- **Problem:** Define an object that encapsulates how a set of objects interact. Mediator promotes loose coupling by keeping objects from referring to each other explicitly.
- **When to use:** Chat rooms, UI control coordination.

#### Full Java Code

```java
import java.util.ArrayList;
import java.util.List;

// Mediator
public interface ChatMediator {
    void sendMessage(String msg, User user);
    void addUser(User user);
}

public class ChatMediatorImpl implements ChatMediator {
    private List<User> users = new ArrayList<>();

    @Override
    public void addUser(User user) { this.users.add(user); }

    @Override
    public void sendMessage(String msg, User user) {
        for (User u : users) {
            if (u != user) { // don't send to self
                u.receive(msg);
            }
        }
    }
}

public abstract class User {
    protected ChatMediator mediator;
    protected String name;
    
    public User(ChatMediator med, String name) {
        this.mediator = med;
        this.name = name;
    }
    public abstract void send(String msg);
    public abstract void receive(String msg);
}

public class UserImpl extends User {
    public UserImpl(ChatMediator med, String name) { super(med, name); }

    @Override
    public void send(String msg) {
        System.out.println(this.name + ": Sending Message=" + msg);
        mediator.sendMessage(msg, this);
    }

    @Override
    public void receive(String msg) {
        System.out.println(this.name + ": Received Message=" + msg);
    }
}
```

### 17. Memento
- **Problem:** Without violating encapsulation, capture and externalize an object's internal state so that the object can be restored to this state later.
- **When to use:** Undo mechanisms, checkpoints.

#### Full Java Code

```java
// Memento
public class EditorState {
    private final String content;
    public EditorState(String content) { this.content = content; }
    public String getContent() { return content; }
}

// Originator
public class Editor {
    private String content;

    public void setContent(String content) { this.content = content; }
    public String getContent() { return content; }

    public EditorState createState() {
        return new EditorState(content);
    }

    public void restore(EditorState state) {
        this.content = state.getContent();
    }
}

// Caretaker
import java.util.Stack;
public class History {
    private Stack<EditorState> states = new Stack<>();

    public void push(EditorState state) { states.push(state); }
    public EditorState pop() { return states.pop(); }
}
```

### 18. Observer
- **Problem:** Define a one-to-many dependency between objects so that when one object changes state, all its dependents are notified and updated automatically.
- **Spring Connection:** `ApplicationEventPublisher` and `@EventListener`. Java's built-in `Observer`/`Observable` is deprecated in Java 9.

#### Full Java Code

```java
import java.util.ArrayList;
import java.util.List;

// Subject
public class NewsAgency {
    private String news;
    private List<Channel> channels = new ArrayList<>();

    public void addObserver(Channel channel) { this.channels.add(channel); }
    public void removeObserver(Channel channel) { this.channels.remove(channel); }

    public void setNews(String news) {
        this.news = news;
        for (Channel channel : this.channels) {
            channel.update(this.news);
        }
    }
}

// Observer
public interface Channel {
    void update(String news);
}

public class NewsChannel implements Channel {
    private String name;
    public NewsChannel(String name) { this.name = name; }

    @Override
    public void update(String news) {
        System.out.println(name + " received breaking news: " + news);
    }
}
```

### 19. State
- **Problem:** Allow an object to alter its behavior when its internal state changes. The object will appear to change its class.
- **When to use:** State machines (e.g., Orders, Vending Machines, Media Players).

#### Full Java Code

```java
// State
public interface VendingMachineState {
    void insertCoin();
    void dispenseItem();
}

// Context
public class VendingMachine {
    private VendingMachineState hasCoinState;
    private VendingMachineState noCoinState;
    private VendingMachineState currentState;

    public VendingMachine() {
        hasCoinState = new HasCoinState(this);
        noCoinState = new NoCoinState(this);
        currentState = noCoinState;
    }
    
    public void setState(VendingMachineState state) { this.currentState = state; }
    public VendingMachineState getHasCoinState() { return hasCoinState; }
    public VendingMachineState getNoCoinState() { return noCoinState; }

    public void insertCoin() { currentState.insertCoin(); }
    public void dispenseItem() { currentState.dispenseItem(); }
}

public class NoCoinState implements VendingMachineState {
    private VendingMachine machine;
    public NoCoinState(VendingMachine machine) { this.machine = machine; }

    @Override
    public void insertCoin() {
        System.out.println("Coin inserted");
        machine.setState(machine.getHasCoinState());
    }
    @Override
    public void dispenseItem() { System.out.println("Insert coin first!"); }
}

public class HasCoinState implements VendingMachineState {
    private VendingMachine machine;
    public HasCoinState(VendingMachine machine) { this.machine = machine; }

    @Override
    public void insertCoin() { System.out.println("Already has coin!"); }
    @Override
    public void dispenseItem() {
        System.out.println("Item dispensed");
        machine.setState(machine.getNoCoinState());
    }
}
```

### 20. Strategy
- **Problem:** Define a family of algorithms, encapsulate each one, and make them interchangeable. Strategy lets the algorithm vary independently from clients that use it.
- **Java 8+:** Often implemented using Lambdas instead of explicit classes. `Comparator` is a classic example.

#### Full Java Code

```java
// Strategy Interface
public interface PaymentStrategy {
    void pay(int amount);
}

// Concrete Strategies
public class CreditCardStrategy implements PaymentStrategy {
    private String name;
    private String cardNumber;

    public CreditCardStrategy(String name, String ccNum) {
        this.name = name;
        this.cardNumber = ccNum;
    }
    @Override
    public void pay(int amount) {
        System.out.println(amount + " paid with credit card.");
    }
}

public class PayPalStrategy implements PaymentStrategy {
    private String email;

    public PayPalStrategy(String email) { this.email = email; }
    @Override
    public void pay(int amount) {
        System.out.println(amount + " paid using PayPal.");
    }
}

// Context
public class ShoppingCart {
    public void checkout(int amount, PaymentStrategy paymentMethod) {
        paymentMethod.pay(amount);
    }
}

// Usage with Java 8 Lambdas
// ShoppingCart cart = new ShoppingCart();
// cart.checkout(100, amount -> System.out.println(amount + " paid with Crypto"));
```

### 21. Template Method
- **Problem:** Define the skeleton of an algorithm in an operation, deferring some steps to subclasses.
- **Hollywood Principle:** "Don't call us, we'll call you." The superclass controls the flow.

#### Full Java Code

```java
public abstract class DataProcessor {
    // Template Method - Final so subclasses can't override the workflow
    public final void processData() {
        readData();
        process();
        saveData();
    }

    protected abstract void readData();
    protected abstract void process();
    
    // Default implementation (hook)
    protected void saveData() {
        System.out.println("Saving data to database...");
    }
}

public class CsvDataProcessor extends DataProcessor {
    @Override
    protected void readData() { System.out.println("Reading CSV file"); }

    @Override
    protected void process() { System.out.println("Processing CSV data"); }
}

public class JsonDataProcessor extends DataProcessor {
    @Override
    protected void readData() { System.out.println("Reading JSON file"); }

    @Override
    protected void process() { System.out.println("Processing JSON data"); }
}
```

### 22. Visitor
- **Problem:** Represent an operation to be performed on the elements of an object structure. Visitor lets you define a new operation without changing the classes of the elements on which it operates.
- **Concept:** Uses Double Dispatch.

#### Full Java Code

```java
// Element
public interface ItemElement {
    int accept(ShoppingCartVisitor visitor);
}

// Concrete Elements
public class Book implements ItemElement {
    private int price;
    public Book(int price) { this.price = price; }
    public int getPrice() { return price; }
    
    @Override
    public int accept(ShoppingCartVisitor visitor) { return visitor.visit(this); }
}

public class Fruit implements ItemElement {
    private int weight;
    public Fruit(int weight) { this.weight = weight; }
    public int getWeight() { return weight; }
    
    @Override
    public int accept(ShoppingCartVisitor visitor) { return visitor.visit(this); }
}

// Visitor
public interface ShoppingCartVisitor {
    int visit(Book book);
    int visit(Fruit fruit);
}

public class ShoppingCartVisitorImpl implements ShoppingCartVisitor {
    @Override
    public int visit(Book book) {
        // can apply logic like $5 discount for books > $50
        return book.getPrice();
    }
    @Override
    public int visit(Fruit fruit) {
        return fruit.getWeight() * 2; // $2 per kg
    }
}
```

### 23. Interpreter
- **Problem:** Given a language, define a representation for its grammar along with an interpreter that uses the representation to interpret sentences in the language.
- **Usage:** RegEx parsing, mathematical expressions.

#### Full Java Code

```java
public interface Expression {
    boolean interpret(String context);
}

public class TerminalExpression implements Expression {
    private String data;
    public TerminalExpression(String data) { this.data = data; }

    @Override
    public boolean interpret(String context) {
        return context.contains(data);
    }
}

public class OrExpression implements Expression {
    private Expression expr1;
    private Expression expr2;

    public OrExpression(Expression expr1, Expression expr2) {
        this.expr1 = expr1;
        this.expr2 = expr2;
    }

    @Override
    public boolean interpret(String context) {
        return expr1.interpret(context) || expr2.interpret(context);
    }
}

// Usage
// Expression isJava = new OrExpression(new TerminalExpression("Java"), new TerminalExpression("JVM"));
// System.out.println(isJava.interpret("I love Java")); // true
```

---

## Patterns in Spring Framework

| Spring Feature/Module | Design Pattern Used | Description |
|---|---|---|
| **Dependency Injection** | Factory / Builder | `BeanFactory` and `ApplicationContext` act as Factories. |
| **Bean Scopes** | Singleton / Prototype | Beans are Singleton by default. Can be set to Prototype. |
| **Spring MVC (DispatcherServlet)** | Front Controller | Single servlet delegates requests to other controllers. |
| **Spring AOP / Security** | Proxy | Wraps beans in JDK dynamic or CGLIB proxies to inject cross-cutting concerns. |
| **JdbcTemplate / RestTemplate** | Template Method | Boilerplate code handled by template, devs provide callbacks/queries. |
| **ViewResolver** | Strategy | Resolves views based on different strategies (JSP, Thymeleaf). |
| **Spring Events** | Observer | `ApplicationEvent` and `@EventListener` mechanisms. |
| **HandlerInterceptor** | Chain of Responsibility | Intercepts HTTP requests sequentially. |

---

## Quick Reference Table

| Category | Pattern | Intent | Real-World Java Example |
|---|---|---|---|
| Creational | Singleton | One instance globally | `Runtime.getRuntime()`, Spring Beans |
| Creational | Factory Method | Subclass decides creation | `Calendar.getInstance()` |
| Creational | Abstract Factory | Families of objects | UI toolkits (Windows/Mac) |
| Creational | Builder | Construct complex object | `StringBuilder`, Lombok `@Builder` |
| Creational | Prototype | Clone objects | `Object.clone()` |
| Structural | Adapter | Incompatible interfaces | Legacy API wrapper |
| Structural | Bridge | Separate abstraction/impl | JDBC Driver Manager |
| Structural | Composite | Tree structures | UI Components, File System |
| Structural | Decorator | Add features dynamically | `java.io.InputStream` subclasses |
| Structural | Facade | Simple UI to subsystem | SLF4J, complex service wrapper |
| Structural | Flyweight | Share to save memory | `String` pool, `Integer.valueOf()` |
| Structural | Proxy | Placeholder/Access control | Spring AOP, Lazy initialization |
| Behavioral | Chain of Resp. | Pass request along chain | Servlet Filters, `try-catch` |
| Behavioral | Command | Request as object | `Runnable`, UI actions (Undo/Redo) |
| Behavioral | Iterator | Sequential traversal | `java.util.Iterator` |
| Behavioral | Mediator | Centralized communication | Enterprise Service Bus (ESB) |
| Behavioral | Memento | Externalize state (undo) | `java.io.Serializable` state save |
| Behavioral | Observer | Publish/Subscribe | Event listeners, GUI clicks |
| Behavioral | State | Alter behavior on state | Process pipelines, Vending machine |
| Behavioral | Strategy | Interchangeable algorithms | `Comparator.compare()`, Sorting |
| Behavioral | Template Method | Algorithm skeleton | `AbstractList`, `JdbcTemplate` |
| Behavioral | Visitor | Operations on elements | ASM bytecode manipulation |
| Behavioral | Interpreter | Grammar interpretation | `java.util.regex.Pattern` |

---

## Interview Questions (Top 10 for Senior Devs)

1. **Q: How is Abstract Factory different from Factory Method?**
   *A:* Factory Method creates one product via inheritance (a subclass overrides a method to return a product). Abstract Factory creates families of related products using object composition.

2. **Q: What is the best way to implement a Singleton in Java?**
   *A:* The `enum` approach is safest against Serialization and Reflection attacks. If lazy loading is strictly required, the Bill Pugh static inner class approach is best because it's thread-safe without explicit synchronization overhead.

3. **Q: How does Spring use the Proxy pattern?**
   *A:* Spring AOP uses Proxies to apply cross-cutting concerns (Transactions, Security, Logging). If a class implements an interface, Spring uses JDK Dynamic Proxies. If not, it uses CGLIB to create a subclass proxy.

4. **Q: What's the difference between Decorator and Proxy?**
   *A:* Intent. A Proxy controls access to an object (e.g., lazy loading, security). A Decorator dynamically adds behavior/responsibilities to an object (e.g., adding milk to coffee).

5. **Q: In standard Java I/O (`java.io`), which design pattern is heavily used and why?**
   *A:* The Decorator pattern. Classes like `BufferedReader` or `DataInputStream` wrap underlying streams (`FileInputStream`) to add buffering or primitive reading capabilities dynamically, preventing class explosion.

6. **Q: What is the Template Method pattern and how does Spring use it?**
   *A:* Template Method defines an algorithm skeleton in a base class but lets subclasses override specific steps. Spring uses this in `JdbcTemplate` or `RestTemplate` where resource opening/closing is handled by the template, and developers only provide SQL/callbacks.

7. **Q: Can you explain Double Dispatch and which pattern uses it?**
   *A:* The Visitor pattern uses double dispatch. The execution depends on the type of two objects: the Visitor type and the Element type being visited. Java only natively supports single dispatch (method overriding).

8. **Q: When would you use a Strategy pattern over a State pattern?**
   *A:* Although structurally similar, Strategy is used to switch algorithms (e.g., sorting algorithms, payment methods) usually chosen by the client. State is used to encapsulate behavior associated with a state; the state transitions itself internally.

9. **Q: Explain the Open/Closed Principle with respect to Design Patterns.**
   *A:* A class should be open for extension but closed for modification. Patterns like Factory Method, Strategy, and Observer excel at this. For example, you can add a new Payment Strategy without touching existing context code.

10. **Q: How can you prevent a Singleton from being instantiated multiple times via Reflection?**
    *A:* By throwing an exception in the private constructor if the static instance variable is already non-null. Alternatively, use an `enum` which the JVM natively protects from reflective instantiation.
