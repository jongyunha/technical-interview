# REST API에 대한 기술 면접 답변

## 주요 질문
"REST API의 개념과 설계 원칙(제약조건)에 대해 설명해주세요. RESTful한 API를 설계한 경험이 있다면 함께 설명해주세요."

## 1. REST 기본 개념

### 1.1 REST의 구성 요소
- 자원(Resource): URI
- 행위(Verb): HTTP Method
- 표현(Representation): JSON, XML 등

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }
    
    @PostMapping
    public ResponseEntity<User> createUser(@RequestBody UserDto userDto) {
        User user = userService.create(userDto);
        URI location = ServletUriComponentsBuilder
            .fromCurrentRequest()
            .path("/{id}")
            .buildAndExpand(user.getId())
            .toUri();
        return ResponseEntity.created(location).body(user);
    }
}
```

## 2. REST 6가지 원칙(제약조건)

### 2.1 Uniform Interface
```java
@GetMapping("/users")
public ResponseEntity<List<UserResponse>> getUsers() {
    // HATEOAS 적용
    List<UserResponse> users = userService.findAll().stream()
        .map(user -> UserResponse.builder()
            .id(user.getId())
            .name(user.getName())
            .links(Arrays.asList(
                new Link("/api/v1/users/" + user.getId(), "self"),
                new Link("/api/v1/users/" + user.getId() + "/orders", "orders")
            ))
            .build())
        .collect(Collectors.toList());
    
    return ResponseEntity.ok(users);
}
```

### 2.2 Stateless
```java
// Bad Practice - 세션 의존
@GetMapping("/cart")
public Cart getCart(HttpSession session) {
    return (Cart) session.getAttribute("cart");
}

// Good Practice - 상태 정보를 요청에 포함
@GetMapping("/cart")
public Cart getCart(@RequestHeader("Authorization") String token) {
    String userId = jwtUtil.extractUserId(token);
    return cartService.getCartByUserId(userId);
}
```

### 2.3 Cacheable
```java
@GetMapping("/products/{id}")
public ResponseEntity<Product> getProduct(@PathVariable Long id) {
    Product product = productService.findById(id);
    return ResponseEntity
        .ok()
        .cacheControl(CacheControl.maxAge(60, TimeUnit.MINUTES))
        .eTag(String.valueOf(product.getVersion()))
        .body(product);
}
```

## 3. HTTP Methods와 상태 코드

### 3.1 HTTP Methods 활용
```java
@RestController
@RequestMapping("/api/v1/orders")
public class OrderController {
    
    // 조회 - GET
    @GetMapping("/{id}")
    public Order getOrder(@PathVariable Long id) { ... }
    
    // 생성 - POST
    @PostMapping
    public ResponseEntity<Order> createOrder(@RequestBody OrderDto dto) { ... }
    
    // 수정 - PUT (전체 수정)
    @PutMapping("/{id}")
    public Order updateOrder(@PathVariable Long id, @RequestBody OrderDto dto) { ... }
    
    // 수정 - PATCH (부분 수정)
    @PatchMapping("/{id}")
    public Order partialUpdateOrder(@PathVariable Long id, 
                                  @RequestBody Map<String, Object> updates) { ... }
    
    // 삭제 - DELETE
    @DeleteMapping("/{id}")
    public ResponseEntity<?> deleteOrder(@PathVariable Long id) { ... }
}
```

### 3.2 상태 코드 활용
```java
@ExceptionHandler(value = {Exception.class})
public ResponseEntity<ErrorResponse> handleException(Exception e) {
    if (e instanceof ResourceNotFoundException) {
        return ResponseEntity
            .status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("Resource not found"));
    }
    
    if (e instanceof InvalidRequestException) {
        return ResponseEntity
            .status(HttpStatus.BAD_REQUEST)
            .body(new ErrorResponse("Invalid request"));
    }
    
    return ResponseEntity
        .status(HttpStatus.INTERNAL_SERVER_ERROR)
        .body(new ErrorResponse("Internal server error"));
}
```

## 4. API 버전 관리

### 4.1 URI 버전 관리
```java
@RestController
public class UserController {
    
    @GetMapping("/api/v1/users")
    public List<UserV1Response> getUsersV1() { ... }
    
    @GetMapping("/api/v2/users")
    public List<UserV2Response> getUsersV2() { ... }
}
```

### 4.2 Header 버전 관리
```java
@RestController
@RequestMapping("/api/users")
public class UserController {
    
    @GetMapping(headers = "X-API-VERSION=1")
    public List<UserV1Response> getUsersV1() { ... }
    
    @GetMapping(headers = "X-API-VERSION=2")
    public List<UserV2Response> getUsersV2() { ... }
}
```

## 근거 자료

### 1. Roy Fielding의 논문
- [Architectural Styles and the Design of Network-based Software Architectures](https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm)

### 2. REST API 설계 가이드
- [Microsoft REST API Guidelines](https://github.com/microsoft/api-guidelines)
- [Google Cloud API Design Guide](https://cloud.google.com/apis/design)

### 3. Best Practices
- [RESTful Web API Design](https://docs.microsoft.com/en-us/azure/architecture/best-practices/api-design)
- [HTTP API Development Tools](https://github.com/yosriady/api-development-tools)

## 실무 관련 추가 질문

1. "REST API 보안을 어떻게 구현하시나요?"

2. "API 응답 시간이 느릴 때 어떻게 최적화하시나요?"

3. "API 버전 관리는 어떤 방식을 선호하시나요?"

4. "REST API 문서화는 어떤 도구를 사용하시나요?"

## 실제 사용 예시

### API 문서화 (Swagger/OpenAPI)
```java
@Configuration
@EnableSwagger2
public class SwaggerConfig {
    @Bean
    public Docket api() {
        return new Docket(DocumentationType.SWAGGER_2)
            .select()
            .apis(RequestHandlerSelectors.basePackage("com.example.api"))
            .paths(PathSelectors.ant("/api/**"))
            .build()
            .apiInfo(apiInfo());
    }
    
    private ApiInfo apiInfo() {
        return new ApiInfoBuilder()
            .title("REST API Documentation")
            .description("REST API 명세서")
            .version("1.0.0")
            .build();
    }
}
```

### API 응답 형식 표준화
```java
@Getter
@Builder
public class ApiResponse<T> {
    private final String status;
    private final String message;
    private final T data;
    private final LocalDateTime timestamp;
    
    public static <T> ApiResponse<T> success(T data) {
        return ApiResponse.<T>builder()
            .status("success")
            .data(data)
            .timestamp(LocalDateTime.now())
            .build();
    }
    
    public static <T> ApiResponse<T> error(String message) {
        return ApiResponse.<T>builder()
            .status("error")
            .message(message)
            .timestamp(LocalDateTime.now())
            .build();
    }
}
```

실제 면접에서는 REST API 설계 원칙을 지키면서도 실무에서 마주치는 현실적인 제약사항들을 어떻게 해결했는지, 그리고 그 과정에서 배운 점들을 구체적으로 설명하는 것이 좋습니다.