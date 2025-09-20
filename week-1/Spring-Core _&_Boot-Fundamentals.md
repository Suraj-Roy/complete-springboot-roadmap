# Week 1: Spring Core & Boot Fundamentals - Study Notes

## 🎯 Learning Objectives
- ✅ Understand Spring Framework, IoC, and DI mechanisms
- ✅ Master bean lifecycle and scopes
- ✅ Setup production-ready Spring Boot environment
- ✅ Apply best practices for enterprise development

---

## 📅 Daily Breakdown

### **Day 1-2: Spring Overview & IoC Container**

#### **What is Spring Framework?**
Spring is a comprehensive framework for enterprise Java development that provides:
- **Inversion of Control (IoC)** - Dependency management
- **Aspect-Oriented Programming (AOP)** - Cross-cutting concerns
- **Data Access** - Database integration
- **Web Framework** - MVC architecture
- **Testing Support** - Unit and integration testing

#### **Spring vs Spring Boot - The Key Differences**

| Aspect | Spring Framework | Spring Boot |
|--------|-----------------|-------------|
| **Configuration** | Extensive XML/Java config | Auto-configuration |
| **Dependencies** | Manual management | Starter dependencies |
| **Server** | External deployment | Embedded server |
| **Development Speed** | Slower setup | Rapid development |
| **Production Features** | Manual setup | Built-in actuator |

**Example Comparison:**

**Traditional Spring:**
```java
// Multiple config files: web.xml, applicationContext.xml, etc.
@Controller
public class HelloController {
    @RequestMapping("/hello")
    @ResponseBody
    public String hello() {
        return "Hello World";
    }
}
```

**Spring Boot:**
```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() {
        return "Hello World";
    }
}
```

#### **IoC (Inversion of Control) Deep Dive**

**Problem with Traditional Approach:**
```java
public class OrderService {
    private PaymentService paymentService;
    
    public OrderService() {
        // OrderService controls dependency creation - TIGHT COUPLING
        this.paymentService = new PaymentService();
    }
}
```

**IoC Solution:**
```java
public class OrderService {
    private final PaymentService paymentService;
    
    // IoC Container injects dependency - LOOSE COUPLING
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

**Benefits of IoC:**
- **Loose Coupling**: Easy to change implementations
- **Testability**: Easy to inject mock objects
- **Flexibility**: Runtime configuration of dependencies
- **Separation of Concerns**: Business logic separated from object creation

#### **Dependency Injection (DI) Types**

**1. Constructor Injection (Recommended):**
```java
@Service
public class OrderService {
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    
    public OrderService(PaymentService paymentService, 
                       NotificationService notificationService) {
        this.paymentService = paymentService;
        this.notificationService = notificationService;
    }
}
```

**Why Constructor Injection is Best:**
- ✅ **Immutability**: Fields can be final
- ✅ **Mandatory Dependencies**: Cannot create object without dependencies
- ✅ **Fail Fast**: Missing dependencies cause startup failure
- ✅ **Testing**: Easy to create objects in tests
- ✅ **Thread Safety**: Immutable objects are thread-safe

**2. Setter Injection (Optional Dependencies):**
```java
@Service
public class OrderService {
    private NotificationService notificationService;
    
    @Autowired(required = false)
    public void setNotificationService(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}
```

**3. Field Injection (❌ Not Recommended):**
```java
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService; // Hard to test, not immutable
}
```

**Why Field Injection is Bad:**
- ❌ Cannot make fields final
- ❌ Hard to test without Spring context
- ❌ Hidden dependencies
- ❌ Framework coupling

---

### **Day 3-4: Beans, Scopes, and Annotations**

#### **What is a Spring Bean?**
A Spring Bean is an object that is:
- Instantiated by Spring IoC container
- Assembled with dependencies
- Managed throughout its lifecycle
- Configured through metadata

#### **Bean Definition Methods**

**1. Component Stereotype Annotations:**
```java
@Component  // Generic component
public class DataProcessor { }

@Service    // Business logic layer
public class OrderService { }

@Repository // Data access layer (adds exception translation)
public class OrderRepository { }

@Controller // Web layer
public class OrderController { }
```

**2. @Bean Method Configuration:**
```java
@Configuration
public class AppConfig {
    
    @Bean
    public PaymentService paymentService() {
        return new StripePaymentService();
    }
    
    @Bean
    public OrderService orderService(PaymentService paymentService) {
        return new OrderService(paymentService);
    }
}
```

#### **@Component vs @Bean**

| Aspect | @Component | @Bean |
|--------|------------|-------|
| **Level** | Class-level | Method-level |
| **Control** | Limited | Full instantiation control |
| **Third-party** | Cannot annotate | Can create beans |
| **Multiple Instances** | One per class | Multiple per method |

**When to use @Bean:**
```java
@Configuration
public class DatabaseConfig {
    
    // Cannot add @Component to third-party DataSource
    @Bean
    public DataSource dataSource() {
        HikariDataSource dataSource = new HikariDataSource();
        dataSource.setJdbcUrl("jdbc:postgresql://localhost:5432/mydb");
        dataSource.setUsername("user");
        dataSource.setPassword("password");
        return dataSource;
    }
}
```

#### **Bean Scopes**

**1. Singleton (Default):**
```java
@Component
@Scope("singleton") // Default - annotation not required
public class DatabaseConnection {
    // Same instance shared across application
}
```

**2. Prototype:**
```java
@Component
@Scope("prototype")
public class OrderProcessor {
    // New instance created each time requested
}
```

**3. Web Scopes:**
```java
// Request scope - new instance per HTTP request
@Component
@Scope(value = WebApplicationContext.SCOPE_REQUEST, 
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext { }

// Session scope - new instance per HTTP session
@Component
@Scope(value = WebApplicationContext.SCOPE_SESSION, 
       proxyMode = ScopedProxyMode.TARGET_CLASS)
public class UserSession { }
```

**Why Proxy Mode for Web Scopes:**
```java
@Service // Singleton
public class OrderService {
    private final RequestContext requestContext; // Request scope
    
    // Spring creates proxy that delegates to current request's instance
    public OrderService(RequestContext requestContext) {
        this.requestContext = requestContext; // This is actually a proxy
    }
}
```

#### **Bean Lifecycle**
```java
@Component
public class LifecycleBean implements InitializingBean, DisposableBean {
    
    // 1. Constructor
    public LifecycleBean() {
        System.out.println("1. Constructor called");
    }
    
    // 2. Dependency Injection
    @Autowired
    public void setDependency(SomeDependency dependency) {
        System.out.println("2. Dependencies injected");
    }
    
    // 3. @PostConstruct
    @PostConstruct
    public void postConstruct() {
        System.out.println("3. @PostConstruct - initialization logic");
    }
    
    // 4. InitializingBean.afterPropertiesSet()
    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("4. afterPropertiesSet() called");
    }
    
    // Bean ready for use
    
    // 5. @PreDestroy
    @PreDestroy
    public void preDestroy() {
        System.out.println("5. @PreDestroy - cleanup logic");
    }
    
    // 6. DisposableBean.destroy()
    @Override
    public void destroy() throws Exception {
        System.out.println("6. destroy() called");
    }
}
```

---

### **Day 5-6: Spring Boot Setup & Configuration**

#### **Spring Boot Project Structure**
```
src/
├── main/
│   ├── java/
│   │   └── com/company/app/
│   │       ├── Application.java              # Main class
│   │       ├── config/                       # Configuration
│   │       │   ├── DatabaseConfig.java
│   │       │   └── SecurityConfig.java
│   │       ├── controller/                   # REST controllers
│   │       │   └── OrderController.java
│   │       ├── service/                      # Business logic
│   │       │   └── OrderService.java
│   │       ├── repository/                   # Data access
│   │       │   └── OrderRepository.java
│   │       └── model/                        # Domain entities
│   │           └── Order.java
│   └── resources/
│       ├── application.yml                   # Configuration
│       ├── application-dev.yml               # Dev profile
│       ├── application-prod.yml              # Prod profile
│       └── static/                           # Static resources
└── test/                                     # Test classes
```

#### **Configuration Management**

**application.properties vs application.yml:**

**Properties Format:**
```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/orders
spring.datasource.username=user
spring.datasource.password=password
server.port=8080
logging.level.com.company.app=DEBUG
```

**YAML Format (Preferred):**
```yaml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/orders
    username: user
    password: password

server:
  port: 8080

logging:
  level:
    com.company.app: DEBUG
```

**Why YAML is Better:**
- ✅ **Hierarchical structure** - No repetition
- ✅ **Better readability** - Clean format
- ✅ **Complex data support** - Lists, maps, nested objects
- ✅ **Comments support**
- ✅ **Multi-document** - Multiple profiles in one file

#### **Profile-Based Configuration**

**Multi-Profile YAML:**
```yaml
spring:
  profiles:
    active: dev

---
# Development profile
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver

---
# Production profile
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://${DB_HOST}:${DB_PORT}/${DB_NAME}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

**Programmatic Profile Configuration:**
```java
@Configuration
@Profile("prod")
public class ProductionConfig {
    
    @Bean
    public NotificationService emailNotificationService() {
        return new EmailNotificationService();
    }
}

@Configuration
@Profile("dev")
public class DevelopmentConfig {
    
    @Bean
    public NotificationService consoleNotificationService() {
        return new ConsoleNotificationService();
    }
}
```

#### **Type-Safe Configuration Properties**

**Why Use @ConfigurationProperties:**
- ✅ **Type Safety** - Compile-time checking
- ✅ **IDE Support** - Auto-completion
- ✅ **Validation** - JSR-303 bean validation
- ✅ **Nested Properties** - Complex object mapping

```java
@ConfigurationProperties(prefix = "app")
@Validated
@Data
public class AppProperties {
    
    @NotNull
    private String name;
    
    private Database database = new Database();
    private Security security = new Security();
    
    @Data
    public static class Database {
        private int maxConnections = 20;
        private int timeout = 5000;
    }
    
    @Data
    public static class Security {
        @NotNull
        private String jwtSecret;
        private long jwtExpiration = 86400000;
    }
}
```

**Enable and Use:**
```java
@SpringBootApplication
@EnableConfigurationProperties(AppProperties.class)
public class Application { }

@Service
public class OrderService {
    private final AppProperties appProperties;
    
    public OrderService(AppProperties appProperties) {
        this.appProperties = appProperties;
    }
}
```

**Configuration:**
```yaml
app:
  name: Order Service
  database:
    max-connections: 50
    timeout: 10000
  security:
    jwt-secret: ${JWT_SECRET:default-secret}
    jwt-expiration: 3600000
```

---

## 🔑 Key Interview Questions & Answers

### **Q1: Difference between Spring and Spring Boot?**

**Answer:**
Spring Boot builds on Spring Framework to simplify development:

**Key Differences:**
- **Configuration**: Spring requires extensive XML/Java config; Boot uses auto-configuration
- **Dependencies**: Spring needs manual version management; Boot provides curated starter dependencies
- **Deployment**: Spring requires external servers; Boot has embedded servers
- **Development**: Spring has slower setup; Boot enables rapid development

**Example:**
Spring Boot reduces hundreds of lines of configuration to a single `@SpringBootApplication` annotation.

### **Q2: What is IoC? How does DI work?**

**Answer:**
**IoC (Inversion of Control)** inverts the control of object creation from application code to the Spring container.

**Traditional Approach (Bad):**
```java
public class OrderService {
    private PaymentService paymentService = new PaymentService(); // Tight coupling
}
```

**IoC Approach (Good):**
```java
public class OrderService {
    private final PaymentService paymentService;
    
    public OrderService(PaymentService paymentService) { // Dependency injected
        this.paymentService = paymentService;
    }
}
```

**DI** is the mechanism that implements IoC through constructor, setter, or field injection.

### **Q3: Explain Bean Lifecycle**

**Answer:**
Spring Bean lifecycle has these phases:
1. **Constructor** - Instance creation
2. **Dependency Injection** - Properties set
3. **@PostConstruct** - Custom initialization
4. **InitializingBean.afterPropertiesSet()** - Interface callback
5. **Bean Ready** - Available for use
6. **@PreDestroy** - Cleanup before destruction
7. **DisposableBean.destroy()** - Interface cleanup

**Practical Use:**
- `@PostConstruct` for database connections, cache warming
- `@PreDestroy` for resource cleanup, connection closing

### **Q4: What are Bean Scopes?**

**Answer:**
Bean scope determines lifecycle and visibility:

- **Singleton** (default): One instance per container
- **Prototype**: New instance each time requested
- **Request**: New instance per HTTP request
- **Session**: New instance per HTTP session

**Example:**
```java
@Component
@Scope("prototype")
public class OrderProcessor {
    // New instance for each order to avoid state conflicts
}
```

Web scopes need `proxyMode = ScopedProxyMode.TARGET_CLASS` to inject into singleton beans.

### **Q5: How does @Component differ from @Bean?**

**Answer:**

| Aspect | @Component | @Bean |
|--------|------------|-------|
| **Level** | Class-level | Method-level |
| **Control** | Limited | Full instantiation control |
| **Third-party** | Can't annotate | Can create beans |

**Use @Component** for your own classes:
```java
@Service
public class OrderService { }
```

**Use @Bean** for third-party libraries or complex creation:
```java
@Bean
public DataSource dataSource() {
    // Complex configuration logic
    return new HikariDataSource();
}
```

### **Q6: What is Auto-Configuration?**

**Answer:**
Auto-configuration automatically configures beans based on:
- **Classpath content** - What libraries are present
- **Existing beans** - What's already configured
- **Properties** - Configuration values

**How it works:**
1. `@EnableAutoConfiguration` triggers scanning
2. `spring.factories` lists auto-configuration classes
3. Conditional annotations decide what to configure

**Example:**
If `DataSource` is on classpath but no `DataSource` bean exists, Spring Boot auto-configures one.

### **Q7: application.properties vs application.yml?**

**Answer:**
**YAML is preferred** for complex configurations:

**Properties:**
```properties
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
```

**YAML:**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      minimum-idle: 5
```

**YAML Benefits:**
- Hierarchical structure (no repetition)
- Better readability
- Native support for lists and maps
- Multi-document support

### **Q8: When would you use @Configuration?**

**Answer:**
Use `@Configuration` for:

**1. Third-party integration:**
```java
@Configuration
public class DatabaseConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource(); // Can't add @Component to third-party class
    }
}
```

**2. Complex bean creation:**
```java
@Configuration
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) {
        return http
            .authorizeRequests(auth -> auth.anyRequest().authenticated())
            .build();
    }
}
```

**3. Environment-specific configuration:**
```java
@Configuration
@Profile("prod")
public class ProductionConfig { }
```

### **Q9: Explain Lazy Initialization**

**Answer:**
**Lazy Initialization** defers bean creation until first use instead of application startup.

**Benefits:**
- ✅ Faster application startup
- ✅ Lower memory usage for unused beans
- ✅ Resource conservation

**Drawbacks:**
- ❌ Configuration errors discovered at runtime
- ❌ First-use latency
- ❌ Harder debugging

**Usage:**
```java
@Component
@Lazy
public class ExpensiveService {
    // Created only when first accessed
}

# Global lazy initialization
spring.main.lazy-initialization=true
```

### **Q10: What's the role of Spring Boot Starters?**

**Answer:**
**Starters** are dependency descriptors that provide curated, compatible dependencies for specific features.

**Problem Solved:**
Instead of managing dozens of dependencies with potential version conflicts:

**Without Starter:**
```xml
<dependency>
    <groupId>org.springframework</groupId>
    <artifactId>spring-webmvc</artifactId>
    <version>6.0.11</version>
</dependency>
<!-- Many more dependencies... -->
```

**With Starter:**
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```

**Benefits:**
- Curated, compatible versions
- Reduced configuration
- Auto-configuration included
- Simplified dependency management

---

## 📋 Best Practices

### **1. Dependency Injection**
- ✅ **Use constructor injection** for mandatory dependencies
- ✅ **Make fields final** when using constructor injection
- ❌ **Avoid field injection** (hard to test, not immutable)

### **2. Configuration**
- ✅ **Externalize configuration** using properties/YAML
- ✅ **Use @ConfigurationProperties** for type safety
- ✅ **Use profiles** for environment-specific settings
- ❌ **Don't hardcode values** in Java classes

### **3. Bean Management**
- ✅ **Use @Service, @Repository, @Controller** for semantic clarity
- ✅ **Use @Bean** for third-party libraries
- ✅ **Keep singleton beans stateless**
- ❌ **Don't use prototype scope unnecessarily**

### **4. Project Structure**
- ✅ **Organize by feature/domain**, not by layer
- ✅ **Keep configuration classes separate**
- ✅ **Use meaningful package names**
- ✅ **Separate test configurations**

---

## 🧪 Self-Check Questions

1. Can you explain why field injection makes testing difficult?
2. What happens if you inject a prototype-scoped bean into a singleton?
3. How does Spring resolve circular dependencies?
4. What's the difference between @Component and @Service functionally?
5. When would you choose lazy initialization?
6. How do you handle sensitive configuration data?
7. What's the order of property source precedence in Spring Boot?
8. How do you create custom conditional annotations?
9. What's the difference between @ConfigurationProperties and @Value?
10. How do you exclude specific auto-configurations?

---

This completes Week 1 fundamentals! You should now have solid understanding of Spring Core concepts and Spring Boot basics, ready to build more complex applications in the following weeks.