# Week 2: Spring MVC & REST APIs - Study Notes

## 🎯 Learning Objectives
- ✅ Master Spring MVC request processing flow
- ✅ Build production-ready REST APIs with CRUD operations
- ✅ Handle HTTP methods, status codes, and responses properly
- ✅ Implement proper API design patterns and best practices

---

## 📅 Daily Breakdown

### **Day 1-2: DispatcherServlet & MVC Architecture**

#### **Spring MVC Architecture Overview**

Spring MVC follows the Front Controller pattern with DispatcherServlet as the central entry point.

**Complete MVC Flow:**
```
HTTP Request → DispatcherServlet → HandlerMapping → Controller → Service → Repository → Database
                     ↓
HTTP Response ← ViewResolver ← ModelAndView ← Controller ← Service ← Repository ← Database
```

#### **DispatcherServlet - The Heart of Spring MVC**

**How DispatcherServlet Works:**

1. **Request Reception**: Receives all HTTP requests
2. **Handler Mapping**: Finds appropriate controller method
3. **Handler Adapter**: Invokes the controller method
4. **Model Processing**: Processes returned model and view
5. **View Resolution**: Resolves view name to actual view
6. **Response Generation**: Renders final response

**Detailed Flow Example:**
```java
// 1. DispatcherServlet receives request: GET /api/users/123

// 2. HandlerMapping finds the matching method
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping("/{id}") // This method matches the request
    public ResponseEntity<UserDTO> getUser(@PathVariable Long id) {
        // Controller logic
    }
}

// 3. HandlerAdapter invokes the method
// 4. Method executes and returns ResponseEntity
// 5. HttpMessageConverter converts response to JSON
// 6. DispatcherServlet sends HTTP response
```

**DispatcherServlet Configuration (Spring Boot handles this automatically):**
```java
// In traditional Spring, you'd configure like this:
@Configuration
@EnableWebMvc
public class WebMvcConfig implements WebMvcConfigurer {
    
    @Bean
    public DispatcherServlet dispatcherServlet() {
        return new DispatcherServlet();
    }
    
    // Spring Boot auto-configures this for you!
}
```

#### **Key Components in Detail**

**1. HandlerMapping:**
```java
// Maps URLs to handler methods
@RequestMapping("/api/products")
public class ProductController {
    
    @GetMapping // Maps GET /api/products
    public List<Product> getAllProducts() { }
    
    @GetMapping("/{id}") // Maps GET /api/products/{id}
    public Product getProduct(@PathVariable Long id) { }
    
    @PostMapping // Maps POST /api/products
    public Product createProduct(@RequestBody Product product) { }
}
```

**2. HandlerAdapter:**
```java
// Adapts different handler types (annotated controllers, HttpRequestHandler, etc.)
// Spring Boot configures RequestMappingHandlerAdapter automatically
```

**3. ViewResolver:**
```java
// For REST APIs, we use HttpMessageConverters instead of ViewResolvers
@Configuration
public class WebConfig {
    
    @Bean
    public HttpMessageConverter<Object> jsonConverter() {
        return new MappingJackson2HttpMessageConverter();
    }
}
```

#### **@Controller vs @RestController**

**@Controller (Traditional MVC):**
```java
@Controller
public class WebController {
    
    @RequestMapping("/home")
    public String home(Model model) {
        model.addAttribute("message", "Welcome!");
        return "home"; // Returns view name
    }
    
    @RequestMapping("/api/users")
    @ResponseBody // Need this for JSON response
    public List<User> getUsers() {
        return userService.getAllUsers(); // Returns data
    }
}
```

**@RestController (REST APIs):**
```java
@RestController // Combines @Controller + @ResponseBody
public class UserRestController {
    
    @GetMapping("/api/users")
    public List<User> getUsers() {
        return userService.getAllUsers(); // Automatically converts to JSON
    }
    
    @PostMapping("/api/users")
    public ResponseEntity<User> createUser(@RequestBody User user) {
        User created = userService.create(user);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }
}
```

**Key Differences:**

| Aspect | @Controller | @RestController |
|--------|-------------|----------------|
| **Purpose** | Web pages (MVC) | REST APIs (JSON/XML) |
| **Response** | View name | Data objects |
| **@ResponseBody** | Required for data | Automatic |
| **Content-Type** | text/html | application/json |

---

### **Day 3-4: CRUD REST APIs**

#### **Building Complete CRUD Operations**

**Entity Class:**
```java
@Entity
@Table(name = "products")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Product {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(nullable = false)
    private String name;
    
    @Column(nullable = false)
    private BigDecimal price;
    
    private String description;
    
    @Enumerated(EnumType.STRING)
    private ProductStatus status = ProductStatus.ACTIVE;
    
    @CreationTimestamp
    private LocalDateTime createdAt;
    
    @UpdateTimestamp
    private LocalDateTime updatedAt;
}

enum ProductStatus {
    ACTIVE, INACTIVE, DISCONTINUED
}
```

**DTO Classes (Data Transfer Objects):**
```java
// Request DTO
@Data
@NoArgsConstructor
@AllArgsConstructor
public class CreateProductRequest {
    
    @NotBlank(message = "Product name is required")
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    private String name;
    
    @NotNull(message = "Price is required")
    @DecimalMin(value = "0.01", message = "Price must be greater than 0")
    private BigDecimal price;
    
    @Size(max = 500, message = "Description cannot exceed 500 characters")
    private String description;
}

// Response DTO
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ProductResponse {
    private Long id;
    private String name;
    private BigDecimal price;
    private String description;
    private ProductStatus status;
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}

// Update DTO
@Data
@NoArgsConstructor
@AllArgsConstructor
public class UpdateProductRequest {
    
    @Size(min = 2, max = 100, message = "Name must be between 2 and 100 characters")
    private String name;
    
    @DecimalMin(value = "0.01", message = "Price must be greater than 0")
    private BigDecimal price;
    
    @Size(max = 500, message = "Description cannot exceed 500 characters")
    private String description;
    
    private ProductStatus status;
}
```

**Complete CRUD Controller:**
```java
@RestController
@RequestMapping("/api/v1/products")
@Validated
public class ProductController {
    
    private final ProductService productService;
    
    public ProductController(ProductService productService) {
        this.productService = productService;
    }
    
    // CREATE - POST /api/v1/products
    @PostMapping
    public ResponseEntity<ProductResponse> createProduct(
            @Valid @RequestBody CreateProductRequest request) {
        
        ProductResponse product = productService.createProduct(request);
        
        return ResponseEntity
                .status(HttpStatus.CREATED)
                .header("Location", "/api/v1/products/" + product.getId())
                .body(product);
    }
    
    // READ - GET /api/v1/products (Get all with pagination)
    @GetMapping
    public ResponseEntity<Page<ProductResponse>> getAllProducts(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "id") String sortBy,
            @RequestParam(defaultValue = "asc") String sortDir,
            @RequestParam(required = false) ProductStatus status) {
        
        Pageable pageable = PageRequest.of(page, size, 
            "desc".equalsIgnoreCase(sortDir) ? Sort.by(sortBy).descending() : Sort.by(sortBy));
        
        Page<ProductResponse> products = productService.getAllProducts(pageable, status);
        
        return ResponseEntity.ok(products);
    }
    
    // READ - GET /api/v1/products/{id} (Get by ID)
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProductById(@PathVariable Long id) {
        
        ProductResponse product = productService.getProductById(id);
        
        return ResponseEntity.ok(product);
    }
    
    // UPDATE - PUT /api/v1/products/{id} (Full update)
    @PutMapping("/{id}")
    public ResponseEntity<ProductResponse> updateProduct(
            @PathVariable Long id,
            @Valid @RequestBody UpdateProductRequest request) {
        
        ProductResponse updated = productService.updateProduct(id, request);
        
        return ResponseEntity.ok(updated);
    }
    
    // PARTIAL UPDATE - PATCH /api/v1/products/{id}
    @PatchMapping("/{id}")
    public ResponseEntity<ProductResponse> patchProduct(
            @PathVariable Long id,
            @RequestBody Map<String, Object> updates) {
        
        ProductResponse updated = productService.patchProduct(id, updates);
        
        return ResponseEntity.ok(updated);
    }
    
    // DELETE - DELETE /api/v1/products/{id}
    @DeleteMapping("/{id}")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        
        productService.deleteProduct(id);
        
        return ResponseEntity.noContent().build();
    }
    
    // BULK OPERATIONS
    @PostMapping("/bulk")
    public ResponseEntity<List<ProductResponse>> createProducts(
            @Valid @RequestBody List<CreateProductRequest> requests) {
        
        List<ProductResponse> products = productService.createProducts(requests);
        
        return ResponseEntity.status(HttpStatus.CREATED).body(products);
    }
    
    @DeleteMapping("/bulk")
    public ResponseEntity<Void> deleteProducts(@RequestBody List<Long> ids) {
        
        productService.deleteProducts(ids);
        
        return ResponseEntity.noContent().build();
    }
}
```

#### **HTTP Methods Deep Dive**

**GET - Retrieve Data:**
```java
@GetMapping("/products")
public ResponseEntity<List<Product>> getProducts(
        @RequestParam(required = false) String category,
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size) {
    
    // GET is:
    // - Safe: No side effects
    // - Idempotent: Same result on multiple calls
    // - Cacheable: Can be cached by browsers/proxies
    
    List<Product> products = productService.getProducts(category, page, size);
    return ResponseEntity.ok(products);
}
```

**POST - Create New Resource:**
```java
@PostMapping("/products")
public ResponseEntity<Product> createProduct(@RequestBody Product product) {
    
    // POST is:
    // - Not Safe: Creates side effects
    // - Not Idempotent: Multiple calls create multiple resources
    // - Not Cacheable
    
    Product created = productService.create(product);
    
    return ResponseEntity
        .status(HttpStatus.CREATED)
        .header("Location", "/products/" + created.getId())
        .body(created);
}
```

**PUT - Update/Replace Resource:**
```java
@PutMapping("/products/{id}")
public ResponseEntity<Product> updateProduct(
        @PathVariable Long id, 
        @RequestBody Product product) {
    
    // PUT is:
    // - Not Safe: Modifies resource
    // - Idempotent: Same result on multiple calls
    // - Replaces entire resource
    
    Product updated = productService.update(id, product);
    return ResponseEntity.ok(updated);
}
```

**PATCH - Partial Update:**
```java
@PatchMapping("/products/{id}")
public ResponseEntity<Product> patchProduct(
        @PathVariable Long id,
        @RequestBody Map<String, Object> updates) {
    
    // PATCH is:
    // - Not Safe: Modifies resource  
    // - Can be Idempotent: Depends on implementation
    // - Updates only specified fields
    
    Product updated = productService.patch(id, updates);
    return ResponseEntity.ok(updated);
}
```

**DELETE - Remove Resource:**
```java
@DeleteMapping("/products/{id}")
public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
    
    // DELETE is:
    // - Not Safe: Removes resource
    // - Idempotent: Same result on multiple calls (404 after first success)
    // - No response body typically
    
    productService.delete(id);
    return ResponseEntity.noContent().build();
}
```

---

### **Day 5-6: Path Variables, Request Parameters & Response Handling**

#### **Path Variables Deep Dive**

**Basic Path Variables:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    // Single path variable
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(user);
    }
    
    // Multiple path variables
    @GetMapping("/{userId}/orders/{orderId}")
    public ResponseEntity<Order> getUserOrder(
            @PathVariable Long userId,
            @PathVariable Long orderId) {
        
        Order order = orderService.findByUserIdAndOrderId(userId, orderId);
        return ResponseEntity.ok(order);
    }
    
    // Path variable with different name
    @GetMapping("/email/{emailAddress}")
    public ResponseEntity<User> getUserByEmail(
            @PathVariable("emailAddress") String email) {
        
        User user = userService.findByEmail(email);
        return ResponseEntity.ok(user);
    }
    
    // Optional path variable (using method overloading)
    @GetMapping({"/search", "/search/{query}"})
    public ResponseEntity<List<User>> searchUsers(
            @PathVariable(required = false) String query) {
        
        List<User> users = query != null ? 
            userService.searchByQuery(query) : 
            userService.findAll();
            
        return ResponseEntity.ok(users);
    }
}
```

**Advanced Path Variable Patterns:**
```java
@RestController
@RequestMapping("/api")
public class AdvancedPathController {
    
    // Regular expression constraints
    @GetMapping("/users/{id:[0-9]+}") // Only numeric IDs
    public User getUserById(@PathVariable Long id) {
        return userService.findById(id);
    }
    
    @GetMapping("/users/{username:[a-zA-Z0-9]+}") // Alphanumeric usernames
    public User getUserByUsername(@PathVariable String username) {
        return userService.findByUsername(username);
    }
    
    // Complex path patterns
    @GetMapping("/files/{category}/{year:[0-9]{4}}/{month:[0-9]{2}}/{filename:.+}")
    public ResponseEntity<Resource> getFile(
            @PathVariable String category,
            @PathVariable int year,
            @PathVariable int month,
            @PathVariable String filename) {
        
        Resource file = fileService.getFile(category, year, month, filename);
        return ResponseEntity.ok(file);
    }
}
```

#### **Request Parameters Mastery**

**Query Parameter Handling:**
```java
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    // Basic query parameters
    @GetMapping
    public ResponseEntity<List<Product>> getProducts(
            @RequestParam(required = false) String category,
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "10") int size,
            @RequestParam(defaultValue = "name") String sortBy,
            @RequestParam(defaultValue = "asc") String sortDir) {
        
        // URL: /api/products?category=electronics&page=1&size=20&sortBy=price&sortDir=desc
        
        List<Product> products = productService.getProducts(category, page, size, sortBy, sortDir);
        return ResponseEntity.ok(products);
    }
    
    // Multiple values for same parameter
    @GetMapping("/by-categories")
    public ResponseEntity<List<Product>> getProductsByCategories(
            @RequestParam List<String> categories) {
        
        // URL: /api/products/by-categories?categories=electronics&categories=books&categories=clothing
        
        List<Product> products = productService.getProductsByCategories(categories);
        return ResponseEntity.ok(products);
    }
    
    // Map for dynamic parameters
    @GetMapping("/search")
    public ResponseEntity<List<Product>> searchProducts(
            @RequestParam Map<String, String> filters) {
        
        // URL: /api/products/search?name=laptop&minPrice=500&maxPrice=2000&brand=apple
        
        List<Product> products = productService.searchProducts(filters);
        return ResponseEntity.ok(products);
    }
    
    // Custom object from request parameters
    @GetMapping("/filter")
    public ResponseEntity<List<Product>> filterProducts(ProductFilter filter) {
        
        // Spring automatically binds request parameters to object fields
        // URL: /api/products/filter?name=laptop&minPrice=500&maxPrice=2000
        
        List<Product> products = productService.filterProducts(filter);
        return ResponseEntity.ok(products);
    }
}

@Data
public class ProductFilter {
    private String name;
    private BigDecimal minPrice;
    private BigDecimal maxPrice;
    private String category;
    private Boolean inStock;
    
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private LocalDate createdAfter;
    
    @DateTimeFormat(pattern = "yyyy-MM-dd")
    private LocalDate createdBefore;
}
```

#### **Request Body Handling**

**JSON Request Processing:**
```java
@RestController
@RequestMapping("/api/orders")
public class OrderController {
    
    // Simple JSON body
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody CreateOrderRequest request) {
        
        Order order = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }
    
    // Validated JSON body
    @PostMapping("/validated")
    public ResponseEntity<Order> createValidatedOrder(
            @Valid @RequestBody CreateOrderRequest request) {
        
        // @Valid triggers JSR-303 validation
        Order order = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }
    
    // Multiple request bodies (using wrapper)
    @PostMapping("/batch")
    public ResponseEntity<List<Order>> createOrders(
            @RequestBody BatchOrderRequest request) {
        
        List<Order> orders = orderService.createOrders(request.getOrders());
        return ResponseEntity.status(HttpStatus.CREATED).body(orders);
    }
}

@Data
public class CreateOrderRequest {
    @NotNull(message = "Customer ID is required")
    private Long customerId;
    
    @NotEmpty(message = "Order items cannot be empty")
    private List<OrderItemRequest> items;
    
    private String notes;
}

@Data
public class OrderItemRequest {
    @NotNull(message = "Product ID is required")
    private Long productId;
    
    @Min(value = 1, message = "Quantity must be at least 1")
    private Integer quantity;
}
```

#### **Response Handling & ResponseEntity**

**ResponseEntity Deep Dive:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    // Basic ResponseEntity usage
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        
        Optional<User> user = userService.findById(id);
        
        if (user.isPresent()) {
            return ResponseEntity.ok(user.get());
        } else {
            return ResponseEntity.notFound().build();
        }
    }
    
    // Custom headers
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody User user) {
        
        User created = userService.create(user);
        
        return ResponseEntity
            .status(HttpStatus.CREATED)
            .header("Location", "/api/users/" + created.getId())
            .header("X-Created-By", "user-service")
            .header("X-Request-ID", UUID.randomUUID().toString())
            .body(created);
    }
    
    // Conditional responses
    @PutMapping("/{id}")
    public ResponseEntity<User> updateUser(
            @PathVariable Long id, 
            @RequestBody User user,
            @RequestHeader(value = "If-Match", required = false) String ifMatch) {
        
        try {
            User updated = userService.updateWithETag(id, user, ifMatch);
            
            return ResponseEntity
                .ok()
                .eTag(updated.getVersion().toString())
                .body(updated);
                
        } catch (OptimisticLockException e) {
            return ResponseEntity.status(HttpStatus.CONFLICT).build();
        } catch (EntityNotFoundException e) {
            return ResponseEntity.notFound().build();
        }
    }
    
    // File download
    @GetMapping("/{id}/avatar")
    public ResponseEntity<Resource> getUserAvatar(@PathVariable Long id) {
        
        Resource avatar = userService.getUserAvatar(id);
        
        if (avatar != null && avatar.exists()) {
            return ResponseEntity
                .ok()
                .contentType(MediaType.IMAGE_JPEG)
                .header(HttpHeaders.CONTENT_DISPOSITION, "inline; filename=\"avatar.jpg\"")
                .body(avatar);
        } else {
            return ResponseEntity.notFound().build();
        }
    }
    
    // Async response
    @GetMapping("/{id}/report")
    public CompletableFuture<ResponseEntity<ReportData>> generateReport(@PathVariable Long id) {
        
        return userService.generateReportAsync(id)
            .thenApply(report -> ResponseEntity
                .ok()
                .header("X-Generation-Time", report.getGenerationTime().toString())
                .body(report))
            .exceptionally(ex -> ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .build());
    }
}
```

**Custom Response Wrappers:**
```java
// Standard API response wrapper
@Data
@AllArgsConstructor
public class ApiResponse<T> {
    private boolean success;
    private String message;
    private T data;
    private LocalDateTime timestamp;
    private String requestId;
    
    public static <T> ApiResponse<T> success(T data) {
        return new ApiResponse<>(true, "Success", data, LocalDateTime.now(), 
            UUID.randomUUID().toString());
    }
    
    public static <T> ApiResponse<T> error(String message) {
        return new ApiResponse<>(false, message, null, LocalDateTime.now(), 
            UUID.randomUUID().toString());
    }
}

// Controller using wrapper
@RestController
@RequestMapping("/api/products")
public class ProductController {
    
    @GetMapping
    public ResponseEntity<ApiResponse<List<Product>>> getProducts() {
        List<Product> products = productService.getAllProducts();
        return ResponseEntity.ok(ApiResponse.success(products));
    }
    
    @PostMapping
    public ResponseEntity<ApiResponse<Product>> createProduct(@RequestBody Product product) {
        try {
            Product created = productService.create(product);
            return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(ApiResponse.success(created));
        } catch (ValidationException e) {
            return ResponseEntity
                .badRequest()
                .body(ApiResponse.error(e.getMessage()));
        }
    }
}
```

---

## 🔑 Key Interview Questions & Answers

### **Q1: How does DispatcherServlet work?**

**Answer:**
DispatcherServlet is the front controller in Spring MVC that handles all HTTP requests through this flow:

1. **Receives Request**: All requests come to DispatcherServlet first
2. **Handler Mapping**: Finds the appropriate controller method using URL patterns
3. **Handler Adapter**: Invokes the matched controller method
4. **Method Execution**: Controller method executes business logic
5. **Response Processing**: Converts response to appropriate format (JSON/XML)
6. **Return Response**: Sends HTTP response back to client

**Example:**
```java
// Request: GET /api/users/123
// 1. DispatcherServlet receives request
// 2. Finds UserController.getUser() method
// 3. Invokes method with id=123
// 4. Returns User object
// 5. Converts to JSON
// 6. Sends JSON response
```

### **Q2: Difference between @RestController and @Controller?**

**Answer:**

| Aspect | @Controller | @RestController |
|--------|-------------|----------------|
| **Purpose** | Web pages (MVC) | REST APIs |
| **Response** | View templates | JSON/XML data |
| **@ResponseBody** | Required for data | Automatic |
| **Content-Type** | text/html | application/json |

**Code Example:**
```java
@Controller
public class WebController {
    @GetMapping("/home")
    public String home(Model model) {
        return "home"; // Returns view name
    }
    
    @GetMapping("/api/data")
    @ResponseBody // Required for JSON
    public List<User> getData() {
        return users;
    }
}

@RestController // = @Controller + @ResponseBody
public class ApiController {
    @GetMapping("/api/users")
    public List<User> getUsers() {
        return users; // Automatically converts to JSON
    }
}
```

### **Q3: Explain GET vs POST vs PUT vs DELETE**

**Answer:**

| Method | Purpose | Safe | Idempotent | Request Body | Response Body |
|--------|---------|------|------------|--------------|---------------|
| **GET** | Retrieve data | ✅ | ✅ | ❌ | ✅ |
| **POST** | Create resource | ❌ | ❌ | ✅ | ✅ |
| **PUT** | Replace resource | ❌ | ✅ | ✅ | ✅ |
| **DELETE** | Remove resource | ❌ | ✅ | ❌ | ❌ |

**Safe**: No side effects
**Idempotent**: Same result on multiple calls

**Examples:**
```java
@GetMapping("/users/{id}")        // Safe, Idempotent
@PostMapping("/users")            // Not safe, Not idempotent  
@PutMapping("/users/{id}")        // Not safe, Idempotent
@DeleteMapping("/users/{id}")     // Not safe, Idempotent
```

### **Q4: What's ResponseEntity?**

**Answer:**
ResponseEntity represents the entire HTTP response including status code, headers, and body.

**Benefits:**
- Full control over HTTP response
- Custom status codes and headers
- Type-safe response handling

**Examples:**
```java
// Basic usage
return ResponseEntity.ok(user);                    // 200 OK
return ResponseEntity.notFound().build();          // 404 Not Found
return ResponseEntity.status(HttpStatus.CREATED)   // 201 Created
    .body(createdUser);

// With custom headers
return ResponseEntity.ok()
    .header("X-Total-Count", "100")
    .header("Cache-Control", "max-age=3600")
    .body(users);

// Conditional responses
if (user.isPresent()) {
    return ResponseEntity.ok(user.get());
} else {
    return ResponseEntity.notFound().build();
}
```

### **Q5: How do you pass query params in Spring Boot?**

**Answer:**
Multiple ways to handle query parameters:

**1. Individual Parameters:**
```java
@GetMapping("/products")
public List<Product> getProducts(
    @RequestParam(required = false) String category,
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size) {
    // URL: /products?category=electronics&page=1&size=20
}
```

**2. Multiple Values:**
```java
@GetMapping("/products")
public List<Product> getProducts(@RequestParam List<String> categories) {
    // URL: /products?categories=electronics&categories=books
}
```

**3. Map for Dynamic Params:**
```java
@GetMapping("/search")
public List<Product> search(@RequestParam Map<String, String> params) {
    // URL: /search?name=laptop&color=black&brand=apple
}
```

**4. Object Binding:**
```java
@GetMapping("/filter")
public List<Product> filter(ProductFilter filter) {
    // URL: /filter?name=laptop&minPrice=500&maxPrice=2000
    // Spring binds params to object fields automatically
}
```

### **Q6: How do you handle JSON input/output?**

**Answer:**
Spring Boot handles JSON automatically through HttpMessageConverters:

**Input (Deserialization):**
```java
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    // JSON in request body automatically converted to User object
    return userService.create(user);
}

// With validation
@PostMapping("/users")
public User createUser(@Valid @RequestBody CreateUserRequest request) {
    // Validates JSON data against annotations
}
```

**Output (Serialization):**
```java
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    User user = userService.findById(id);
    // User object automatically converted to JSON response
    return user;
}
```

**Custom JSON Configuration:**
```java
@Configuration
public class JacksonConfig {
    
    @Bean
    @Primary
    public ObjectMapper objectMapper() {
        return JsonMapper.builder()
            .addModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS)
            .disable(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES)
            .build();
    }
}
```

### **Q7: Explain content negotiation**

**Answer:**
Content negotiation allows the same endpoint to return different formats (JSON, XML, etc.) based on client preferences.

**Configuration:**
```java
@Configuration
public class WebConfig implements WebMvcConfigurer {
    
    @Override
    public void configureContentNegotiation(ContentNegotiationConfigurer configurer) {
        configurer
            .favorParameter(true)
            .parameterName("mediaType")
            .defaultContentType(MediaType.APPLICATION_JSON)
            .mediaType("json", MediaType.APPLICATION_JSON)
            .mediaType("xml", MediaType.APPLICATION_XML);
    }
}
```

**Usage:**
```java
@GetMapping(value = "/users/{id}", 
           produces = {MediaType.APPLICATION_JSON_VALUE, MediaType.APPLICATION_XML_VALUE})
public User getUser(@PathVariable Long id) {
    return userService.findById(id);
}

// Requests:
// GET /users/123?mediaType=json  → Returns JSON
// GET /users/123?mediaType=xml   → Returns XML
// GET /users/123 with Accept: application/json → Returns JSON
// GET /users/123 with Accept: application/xml  → Returns XML
```

### **Q8: What is idempotency in REST APIs?**

**Answer:**
**Idempotency** means multiple identical requests should have the same effect as a single request.

**Idempotent Methods:**
- **GET**: Reading data multiple times doesn't change anything
- **PUT**: Updating with same data produces same result
- **DELETE**: Deleting already deleted resource is same as first delete
- **HEAD, OPTIONS**: Safe operations

**Non-Idempotent:**
- **POST**: Creating multiple resources with same data creates duplicates

**Examples:**
```java
// Idempotent - PUT
@PutMapping("/users/{id}")
public User updateUser(@PathVariable Long id, @RequestBody User user) {
    // Calling this 10 times with same data = calling once
    return userService.update(id, user);
}

// Non-idempotent - POST  
@PostMapping("/users")
public User createUser(@RequestBody User user) {
    // Calling this 10 times = 10 different users created
    return userService.create(user);
}

// Making POST idempotent with client-generated ID
@PostMapping("/users")
public ResponseEntity<User> createUser(
        @RequestBody User user,
        @RequestHeader("Idempotency-Key") String idempotencyKey) {
    
    // Check if request with this key was already processed
    if (userService.isRequestProcessed(idempotencyKey)) {
        return ResponseEntity.ok(userService.getByIdempotencyKey(idempotencyKey));
    }
    
    User created = userService.create(user, idempotencyKey);
    return ResponseEntity.status(HttpStatus.CREATED).body(created);
}
```

### **Q9: How do you version REST APIs?**

**Answer:**
Several strategies for API versioning:

**1. URL Versioning (Most Common):**
```java
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    
    @GetMapping("/{id}")
    public UserV1 getUser(@PathVariable Long id) {
        return userService.getUserV1(id);
    }
}

@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    
    @GetMapping("/{id}")
    public UserV2 getUser(@PathVariable Long id) {
        return userService.getUserV2(id);
    }
}
```

**2. Header Versioning:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(value = "/{id}", headers = "X-API-VERSION=1")
    public UserV1 getUserV1(@PathVariable Long id) {
        return userService.getUserV1(id);
    }
    
    @GetMapping(value = "/{id}", headers = "X-API-VERSION=2")
    public UserV2 getUserV2(@PathVariable Long id) {
        return userService.getUserV2(id);
    }
}
```

**3. Accept Header Versioning:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(value = "/{id}", produces = "application/vnd.company.user-v1+json")
    public UserV1 getUserV1(@PathVariable Long id) {
        return userService.getUserV1(id);
    }
    
    @GetMapping(value = "/{id}", produces = "application/vnd.company.user-v2+json")
    public UserV2 getUserV2(@PathVariable Long id) {
        return userService.getUserV2(id);
    }
}
```

**4. Query Parameter Versioning:**
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(value = "/{id}", params = "version=1")
    public UserV1 getUserV1(@PathVariable Long id) {
        return userService.getUserV1(id);
    }
    
    @GetMapping(value = "/{id}", params = "version=2")
    public UserV2 getUserV2(@PathVariable Long id) {
        return userService.getUserV2(id);
    }
}
```

**Best Practices:**
- **URL versioning** is most explicit and cache-friendly
- **Semantic versioning** (v1.0, v1.1, v2.0)
- **Deprecation strategy** with timeline
- **Backward compatibility** when possible

### **Q10: How do you test a REST API in Spring Boot?**

**Answer:**
Multiple testing approaches for Spring Boot REST APIs:

**1. @WebMvcTest (Web Layer Only):**
```java
@WebMvcTest(UserController.class)
public class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @MockBean
    private UserService userService;
    
    @Test
    public void testGetUser() throws Exception {
        User user = new User(1L, "John", "john@example.com");
        when(userService.findById(1L)).thenReturn(user);
        
        mockMvc.perform(get("/api/users/1"))
            .andExpect(status().isOk())
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("John"))
            .andExpect(jsonPath("$.email").value("john@example.com"));
    }
    
    @Test
    public void testCreateUser() throws Exception {
        User user = new User(1L, "John", "john@example.com");
        when(userService.create(any(User.class))).thenReturn(user);
        
        mockMvc.perform(post("/api/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content("{\"name\":\"John\",\"email\":\"john@example.com\"}"))
            .andExpect(status().isCreated())
            .andExpect(header().exists("Location"))
            .andExpect(jsonPath("$.id").value(1));
    }
    
    @Test
    public void testGetUserNotFound() throws Exception {
        when(userService.findById(999L)).thenThrow(new UserNotFoundException("User not found"));
        
        mockMvc.perform(get("/api/users/999"))
            .andExpect(status().isNotFound());
    }
}
```

**2. @SpringBootTest (Integration Test):**
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
@TestMethodOrder(OrderAnnotation.class)
public class UserControllerIntegrationTest {
    
    @Autowired
    private TestRestTemplate restTemplate;
    
    @Autowired
    private UserRepository userRepository;
    
    @Test
    @Order(1)
    public void testCreateUser() {
        User user = new User("John", "john@example.com");
        
        ResponseEntity<User> response = restTemplate.postForEntity("/api/users", user, User.class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().getName()).isEqualTo("John");
        assertThat(userRepository.count()).isEqualTo(1);
    }
    
    @Test
    @Order(2)
    public void testGetAllUsers() {
        ResponseEntity<User[]> response = restTemplate.getForEntity("/api/users", User[].class);
        
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody()).hasSize(1);
    }
}
```

**3. WebTestClient (Reactive Testing):**
```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class UserControllerWebTestClientTest {
    
    @Autowired
    private WebTestClient webTestClient;
    
    @Test
    public void testGetUser() {
        webTestClient.get()
            .uri("/api/users/1")
            .accept(MediaType.APPLICATION_JSON)
            .exchange()
            .expectStatus().isOk()
            .expectBody()
            .jsonPath("$.name").isEqualTo("John")
            .jsonPath("$.email").isEqualTo("john@example.com");
    }
    
    @Test
    public void testCreateUser() {
        User user = new User("Jane", "jane@example.com");
        
        webTestClient.post()
            .uri("/api/users")
            .contentType(MediaType.APPLICATION_JSON)
            .bodyValue(user)
            .exchange()
            .expectStatus().isCreated()
            .expectHeader().exists("Location")
            .expectBody(User.class)
            .value(created -> {
                assertThat(created.getName()).isEqualTo("Jane");
                assertThat(created.getId()).isNotNull();
            });
    }
}
```

**4. Custom Test Utilities:**
```java
@TestConfiguration
public class TestConfig {
    
    @Bean
    @Primary
    public Clock testClock() {
        return Clock.fixed(Instant.parse("2023-01-01T00:00:00Z"), ZoneOffset.UTC);
    }
}

@Component
public class ApiTestHelper {
    
    @Autowired
    private MockMvc mockMvc;
    
    public ResultActions performGet(String url, Object... urlVars) throws Exception {
        return mockMvc.perform(get(url, urlVars)
            .contentType(MediaType.APPLICATION_JSON));
    }
    
    public ResultActions performPost(String url, Object body) throws Exception {
        return mockMvc.perform(post(url)
            .contentType(MediaType.APPLICATION_JSON)
            .content(asJsonString(body)));
    }
    
    private String asJsonString(Object obj) throws JsonProcessingException {
        return new ObjectMapper().writeValueAsString(obj);
    }
}
```

**5. Test Data Builders:**
```java
public class UserTestDataBuilder {
    private String name = "Default Name";
    private String email = "default@example.com";
    private UserRole role = UserRole.USER;
    
    public UserTestDataBuilder withName(String name) {
        this.name = name;
        return this;
    }
    
    public UserTestDataBuilder withEmail(String email) {
        this.email = email;
        return this;
    }
    
    public UserTestDataBuilder withRole(UserRole role) {
        this.role = role;
        return this;
    }
    
    public User build() {
        User user = new User();
        user.setName(name);
        user.setEmail(email);
        user.setRole(role);
        return user;
    }
    
    // Usage in tests
    public static UserTestDataBuilder aUser() {
        return new UserTestDataBuilder();
    }
}

// In test class
@Test
public void testCreateAdminUser() {
    User admin = aUser()
        .withName("Admin User")
        .withEmail("admin@example.com")
        .withRole(UserRole.ADMIN)
        .build();
        
    // Test with admin user
}
```

---

## 📋 Best Practices

### **1. HTTP Status Codes**
```java
// Use appropriate status codes
@PostMapping("/users")
public ResponseEntity<User> createUser(@RequestBody User user) {
    User created = userService.create(user);
    return ResponseEntity.status(HttpStatus.CREATED).body(created); // 201
}

@GetMapping("/users/{id}")
public ResponseEntity<User> getUser(@PathVariable Long id) {
    return userService.findById(id)
        .map(user -> ResponseEntity.ok(user))              // 200
        .orElse(ResponseEntity.notFound().build());        // 404
}

@PutMapping("/users/{id}")
public ResponseEntity<User> updateUser(@PathVariable Long id, @RequestBody User user) {
    try {
        User updated = userService.update(id, user);
        return ResponseEntity.ok(updated);                 // 200
    } catch (OptimisticLockException e) {
        return ResponseEntity.status(HttpStatus.CONFLICT).build(); // 409
    }
}
```

### **2. DTO Pattern (Separate DTOs from Entities)**
```java
// Entity (Internal representation)
@Entity
public class User {
    private Long id;
    private String password;  // Sensitive data
    private String email;
    private LocalDateTime createdAt;
    // ... other internal fields
}

// Response DTO (External representation)
public class UserResponseDTO {
    private Long id;
    private String email;
    private LocalDateTime createdAt;
    // No password field - security!
}

// Request DTO (Input validation)
public class CreateUserRequestDTO {
    @NotBlank
    @Email
    private String email;
    
    @NotBlank
    @Size(min = 8)
    private String password;
}

// Mapper
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserResponseDTO toResponseDTO(User user);
    User fromCreateRequest(CreateUserRequestDTO dto);
}
```

### **3. Keep Controllers Thin, Services Thick**
```java
// ❌ Fat Controller (Bad)
@RestController
public class BadUserController {
    
    @PostMapping("/users")
    public ResponseEntity<User> createUser(@RequestBody User user) {
        // Validation logic
        if (user.getEmail() == null || !user.getEmail().contains("@")) {
            throw new BadRequestException("Invalid email");
        }
        
        // Business logic
        if (userRepository.findByEmail(user.getEmail()).isPresent()) {
            throw new ConflictException("User already exists");
        }
        
        // Password encoding
        user.setPassword(passwordEncoder.encode(user.getPassword()));
        
        // Audit fields
        user.setCreatedAt(LocalDateTime.now());
        user.setCreatedBy("system");
        
        // Save
        User saved = userRepository.save(user);
        
        // Send notification
        emailService.sendWelcomeEmail(saved.getEmail());
        
        return ResponseEntity.status(HttpStatus.CREATED).body(saved);
    }
}

// ✅ Thin Controller (Good)
@RestController
public class UserController {
    
    private final UserService userService;
    
    @PostMapping("/users")
    public ResponseEntity<UserResponseDTO> createUser(@Valid @RequestBody CreateUserRequestDTO request) {
        UserResponseDTO user = userService.createUser(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
}

// ✅ Thick Service (Good)
@Service
public class UserService {
    
    public UserResponseDTO createUser(CreateUserRequestDTO request) {
        // All business logic here
        validateUserRequest(request);
        checkUserExists(request.getEmail());
        
        User user = userMapper.fromCreateRequest(request);
        user.setPassword(passwordEncoder.encode(user.getPassword()));
        setAuditFields(user);
        
        User saved = userRepository.save(user);
        
        notificationService.sendWelcomeEmail(saved.getEmail());
        
        return userMapper.toResponseDTO(saved);
    }
}
```

---

## 🧪 Self-Check Questions

1. **What happens if you don't use @ResponseBody with @Controller?**
2. **How do you handle file uploads in Spring MVC?**
3. **What's the difference between @PathVariable and @RequestParam?**
4. **How do you implement pagination in REST APIs?**
5. **What's the role of HandlerInterceptor in Spring MVC?**
6. **How do you handle CORS in Spring Boot?**
7. **What's the difference between @RequestMapping and specific annotations like @GetMapping?**
8. **How do you implement content compression in Spring Boot?**
9. **What's the purpose of @CrossOrigin annotation?**
10. **How do you implement custom HTTP message converters?**

---

## 🎯 Week 2 Summary

By the end of Week 2, you should be able to:

✅ **Understand Spring MVC flow** completely from request to response  
✅ **Build production-ready REST APIs** with proper CRUD operations  
✅ **Handle all HTTP methods** correctly with appropriate status codes  
✅ **Implement request/response handling** with path variables, query params, and JSON  
✅ **Apply REST API best practices** including DTOs, proper error handling, and testing  
✅ **Version APIs properly** and understand idempotency concepts  
✅ **Write comprehensive tests** for your REST APIs

You're now ready to build robust, scalable REST APIs that follow industry standards and best practices!