# Cole LMS — Tổ Chức Code Backend (Spring Boot)

> Tài liệu này dành cho developer FE đang chuyển sang BE.  
> Mọi ví dụ code đều lấy từ thực tế trong dự án này.

---

## Mục Lục

1. [Tổng quan dự án](#1-tổng-quan-dự-án)
2. [Tech Stack](#2-tech-stack)
3. [Cấu trúc thư mục](#3-cấu-trúc-thư-mục)
4. [Kiến trúc phân tầng](#4-kiến-trúc-phân-tầng)
5. [Tầng Entity — Database Layer](#5-tầng-entity--database-layer)
6. [Tầng Repository — Data Access Layer](#6-tầng-repository--data-access-layer)
7. [Tầng Service — Business Logic Layer](#7-tầng-service--business-logic-layer)
8. [Tầng Controller — API Layer](#8-tầng-controller--api-layer)
9. [DTO — Data Transfer Object](#9-dto--data-transfer-object)
10. [Security & Authentication](#10-security--authentication)
11. [Configuration & Environment](#11-configuration--environment)
12. [Design Patterns trong dự án](#12-design-patterns-trong-dự-án)
13. [Luồng xử lý một API hoàn chỉnh](#13-luồng-xử-lý-một-api-hoàn-chỉnh)
14. [Tổng số file theo từng tầng](#14-tổng-số-file-theo-từng-tầng)

---

## 1. Tổng quan dự án

**Cole LMS** là hệ thống quản lý học tập (Learning Management System) cho trung tâm đào tạo.

| Thông tin | Chi tiết |
|---|---|
| Ngôn ngữ | Java 21 |
| Framework | Spring Boot 4.0.1 |
| Database | PostgreSQL |
| Cache | Redis (fallback: In-Memory) |
| Auth | JWT + Google OAuth2 |
| Base package | `vn.cole.lms` |
| Port mặc định | 8080 |

### Các domain chính

```
┌─────────────────────────────────────────────────────────┐
│                     Cole LMS Backend                    │
├──────────┬──────────┬──────────┬──────────┬─────────────┤
│  Course  │  Class   │   Exam   │  Auth    │ Integration │
│ (Khóa   │ (Lớp    │  (Thi    │ (Đăng   │ (Me/Misa/  │
│  học)   │  học)   │  cử)    │  nhập)  │  Payment)  │
└──────────┴──────────┴──────────┴──────────┴─────────────┘
```

---

## 2. Tech Stack

### Core Dependencies

| Dependency | Mục đích |
|---|---|
| `spring-boot-starter-web` | REST API (HTTP server) |
| `spring-boot-starter-data-jpa` | ORM — làm việc với database |
| `spring-boot-starter-security` | Authentication & Authorization |
| `spring-boot-starter-data-redis` | Cache với Redis |
| `spring-boot-starter-mail` | Gửi email |
| `spring-boot-starter-oauth2-client` | Google OAuth2 |
| `jjwt 0.12.5` | Tạo & validate JWT token |
| `postgresql` | Driver kết nối PostgreSQL |
| `lombok` | Giảm boilerplate (getter/setter/builder) |
| `springdoc-openapi` | Swagger UI documentation |
| `apache-poi` | Xuất file Excel |
| `pdfbox` | Xuất file PDF |

### So sánh với FE ecosystem

| FE (React) | BE (Spring Boot) |
|---|---|
| `express` / `fastify` | `spring-boot-starter-web` |
| `prisma` / `typeorm` | `spring-data-jpa` |
| `jsonwebtoken` | `jjwt` |
| `passport.js` | `spring-security` |
| `redis` client | `spring-data-redis` |
| `nodemailer` | `spring-boot-starter-mail` |
| `.env` | `application.properties` |

---

## 3. Cấu trúc thư mục

```
lms_be/
├── lms/
│   ├── src/main/java/vn/cole/lms/
│   │   ├── LmsApplication.java              ← Entry point (giống index.js)
│   │   │
│   │   ├── config/                          ← Cấu hình hệ thống
│   │   │   ├── SecurityConfig.java          ← Bảo mật, CORS, JWT filter
│   │   │   ├── JwtTokenProvider.java        ← Tạo & validate JWT
│   │   │   ├── JwtAuthenticationFilter.java ← Đọc token từ header
│   │   │   ├── OAuth2SuccessHandler.java    ← Xử lý sau login Google
│   │   │   ├── RedisConfig.java             ← Kết nối Redis
│   │   │   ├── OpenApiConfig.java           ← Swagger docs
│   │   │   └── JacksonConfig.java           ← JSON serialization
│   │   │
│   │   ├── controller/                      ← Nhận HTTP request (giống route handler)
│   │   │   ├── BaseController.java          ← Class cha chứa helper methods
│   │   │   ├── AuthenticationController.java
│   │   │   ├── ClassController.java
│   │   │   ├── CourseController.java
│   │   │   ├── ExamController.java
│   │   │   └── ... (33 controllers tổng)
│   │   │
│   │   ├── service/                         ← Business logic
│   │   │   ├── AuthenticationService.java
│   │   │   ├── courseclass/                 ← Service liên quan đến lớp học
│   │   │   │   ├── ClassService.java
│   │   │   │   ├── ClassSessionService.java
│   │   │   │   └── ...
│   │   │   ├── auth/                        ← Strategy pattern cho auth
│   │   │   │   ├── AuthStrategy.java        ← Interface
│   │   │   │   ├── LocalAuthStrategy.java   ← Đăng nhập email/phone
│   │   │   │   └── GoogleAuthStrategy.java  ← Đăng nhập Google
│   │   │   ├── cache/                       ← Cache abstraction
│   │   │   ├── otp/                         ← OTP (SMS/Email)
│   │   │   ├── payment/                     ← Thanh toán
│   │   │   ├── storage/                     ← Lưu trữ file
│   │   │   ├── integration/                 ← Me platform sync
│   │   │   └── misa/                        ← Misa accounting sync
│   │   │
│   │   ├── db/
│   │   │   ├── entity/                      ← Ánh xạ với bảng database
│   │   │   │   ├── User.java
│   │   │   │   ├── Role.java
│   │   │   │   ├── course/                  ← Entity khóa học
│   │   │   │   │   ├── Course.java
│   │   │   │   │   ├── CourseModule.java
│   │   │   │   │   ├── Lesson.java
│   │   │   │   │   └── ...
│   │   │   │   ├── classes/                 ← Entity lớp học
│   │   │   │   │   ├── CourseClass.java
│   │   │   │   │   ├── ClassSession.java
│   │   │   │   │   ├── StudentClass.java
│   │   │   │   │   └── ...
│   │   │   │   ├── exam/                    ← Entity bài thi
│   │   │   │   ├── integration/             ← Entity tích hợp bên ngoài
│   │   │   │   ├── lecturer/                ← Entity giảng viên
│   │   │   │   ├── learingpath/             ← Entity lộ trình học
│   │   │   │   └── misa/                    ← Entity Misa sync
│   │   │   │
│   │   │   ├── repository/                  ← Truy vấn database
│   │   │   │   ├── UserRepository.java
│   │   │   │   ├── ClassRepository.java
│   │   │   │   └── ... (80+ repositories)
│   │   │   │
│   │   │   ├── dto/
│   │   │   │   ├── request/                 ← Nhận data từ client
│   │   │   │   │   ├── RegisterRequest.java
│   │   │   │   │   ├── LoginRequest.java
│   │   │   │   │   ├── course/
│   │   │   │   │   ├── courseclass/
│   │   │   │   │   └── exam/
│   │   │   │   └── response/                ← Trả data về client
│   │   │   │       ├── ApiResponse.java     ← Wrapper chung cho mọi response
│   │   │   │       ├── AuthResponse.java
│   │   │   │       └── ...
│   │   │   │
│   │   │   ├── enums/                       ← Các kiểu Enum
│   │   │   │   ├── CourseStatus.java
│   │   │   │   ├── ClassStatus.java
│   │   │   │   ├── AuthType.java
│   │   │   │   └── ... (42 enums tổng)
│   │   │   │
│   │   │   └── projection/                  ← Interface cho native query
│   │   │
│   │   ├── mapper/                          ← Chuyển đổi Entity <-> DTO
│   │   ├── exception/                       ← Global error handler
│   │   └── util/                            ← Utility classes
│   │
│   └── src/main/resources/
│       └── application.properties           ← Cấu hình ứng dụng
│
├── Dockerfile
└── docker-compose.yml
```

---

## 4. Kiến trúc phân tầng

Mọi HTTP request đi qua đúng thứ tự sau:

```
HTTP Request
     │
     ▼
┌─────────────────────────────────────────┐
│         Security Filter Chain           │  ← Kiểm tra JWT trước
│   JwtAuthenticationFilter              │
└─────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────┐
│              Controller                 │  ← Nhận request, trả response
│   @RestController, @GetMapping, ...    │
└─────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────┐
│               Service                   │  ← Business logic
│   @Service, @Transactional             │
└─────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────┐
│             Repository                  │  ← Truy vấn database
│   JpaRepository, @Query                │
└─────────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────────┐
│             PostgreSQL DB               │
└─────────────────────────────────────────┘
     │
     ▼
HTTP Response (bọc trong ApiResponse<T>)
```

### So sánh với FE (Express.js)

```
FE (Express)              BE (Spring Boot)
──────────────────────    ─────────────────────────
middleware (auth check)   JwtAuthenticationFilter
router.get('/...')        @GetMapping("/...")
controller function       @RestController method
service/business logic    @Service class
db.query(...)             @Repository / JpaRepository
```

---

## 5. Tầng Entity — Database Layer

Entity là class Java ánh xạ 1-1 với bảng trong PostgreSQL.

### Ví dụ thực tế: `Course.java`

```java
@Entity                          // Đánh dấu đây là bảng DB
@Table(name = "courses")         // Tên bảng trong PostgreSQL
@Builder                         // Dùng builder pattern để tạo object
@Getter @Setter                  // Lombok: tự generate getter/setter
@NoArgsConstructor               // Constructor không tham số
@AllArgsConstructor              // Constructor đầy đủ tham số
public class Course {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;              // AUTO_INCREMENT

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "course_type_id")
    private CourseType courseType; // Khóa ngoại → bảng course_types

    @Column(name = "status")
    @Enumerated(EnumType.STRING)
    private CourseStatus status;  // Lưu dạng string: "DRAFT", "PUBLISHED"

    @OneToMany(mappedBy = "course", cascade = CascadeType.ALL)
    private Set<CourseModule> modules; // Một khóa học có nhiều module

    @Column(name = "deleted_at")
    private LocalDateTime deletedAt; // Soft delete — không xóa thật

    @PrePersist                  // Hook tự động chạy trước INSERT
    protected void onCreate() {
        createdAt = LocalDateTime.now();
    }

    @PreUpdate                   // Hook tự động chạy trước UPDATE
    protected void onUpdate() {
        updatedAt = LocalDateTime.now();
    }
}
```

### Các annotation Entity quan trọng

| Annotation | Ý nghĩa | So sánh TS |
|---|---|---|
| `@Entity` | Đây là bảng DB | `@Entity()` (TypeORM) |
| `@Table(name="...")` | Tên bảng | `@Entity('table_name')` |
| `@Id` | Primary key | `@PrimaryGeneratedColumn()` |
| `@Column` | Cột thường | `@Column()` |
| `@ManyToOne` | Nhiều record → 1 record | `@ManyToOne()` |
| `@OneToMany` | 1 record → nhiều record | `@OneToMany()` |
| `@ManyToMany` | Nhiều → nhiều | `@ManyToMany()` |
| `@Enumerated` | Lưu enum dạng STRING | — |
| `@PrePersist` | Hook trước INSERT | `@BeforeInsert()` |

### Soft Delete Pattern

Dự án này **không xóa dữ liệu thật** — chỉ set `deleted_at`:

```java
// Thay vì DELETE FROM courses WHERE id = 1
// Dự án này làm:
course.setDeletedAt(LocalDateTime.now()); // UPDATE courses SET deleted_at = now()
courseRepository.save(course);
```

---

## 6. Tầng Repository — Data Access Layer

Repository là interface để truy vấn database. Spring Data JPA **tự generate câu SQL** từ tên method.

### Ví dụ: `ClassRepository.java`

```java
@Repository  // Hoặc không cần vì extends JpaRepository
public interface ClassRepository extends JpaRepository<CourseClass, Long> {

    // Spring tự viết: SELECT * FROM course_class WHERE course_id = ?
    List<CourseClass> findByCourseId(Long courseId);

    // Spring tự viết: SELECT * FROM course_class WHERE status = ? AND deleted_at IS NULL
    List<CourseClass> findByStatusAndDeletedAtIsNull(ClassStatus status);

    // Tự viết JPQL (Java query - không phải SQL thuần)
    @Query("SELECT c FROM CourseClass c WHERE c.course.id = :courseId AND c.deletedAt IS NULL")
    List<CourseClass> findActiveByCourseId(@Param("courseId") Long courseId);

    // Native SQL (khi cần query phức tạp)
    @Query(value = "SELECT * FROM course_class WHERE ...", nativeQuery = true)
    List<Object[]> findComplexData();
}
```

### Các method có sẵn từ `JpaRepository`

```java
repository.findById(id)        // SELECT WHERE id = ?
repository.findAll()           // SELECT *
repository.save(entity)        // INSERT hoặc UPDATE
repository.delete(entity)      // DELETE
repository.existsById(id)      // SELECT EXISTS
repository.count()             // SELECT COUNT(*)
```

### Naming Convention cho query method

```
findBy + FieldName + Condition

findByEmail                    → WHERE email = ?
findByEmailAndPhone            → WHERE email = ? AND phone = ?
findByStatusIn                 → WHERE status IN (?, ?)
findByDeletedAtIsNull          → WHERE deleted_at IS NULL
findByCreatedAtBetween         → WHERE created_at BETWEEN ? AND ?
findByNameContaining           → WHERE name LIKE '%?%'
```

---

## 7. Tầng Service — Business Logic Layer

Service chứa toàn bộ logic nghiệp vụ. Controller **không được** chứa logic.

### Ví dụ: `AuthenticationService.java`

```java
@Service  // Spring quản lý lifecycle của class này
public class AuthenticationService {

    // Dependency Injection qua constructor (không dùng @Autowired trực tiếp)
    public AuthenticationService(
        UserRepository userRepository,
        PasswordEncoder passwordEncoder,
        JwtTokenProvider tokenProvider,
        ...
    ) {
        this.userRepository = userRepository;
        // ...
    }

    @Transactional  // Nếu có lỗi giữa chừng → rollback toàn bộ
    public AuthResponse register(RegisterRequest request) {
        // 1. Validate OTP đã verify chưa
        boolean isVerified = otpService.isIdentifierVerified(request.getIdentifier());
        if (!isVerified) throw new RuntimeException("OTP verification failed");

        // 2. Kiểm tra email/phone đã tồn tại chưa
        if (userRepository.existsByEmail(identifier)) {
            throw new RuntimeException("Email is already in use!");
        }

        // 3. Tạo User entity
        User user = User.builder()
            .email(request.getIdentifier())
            .password(passwordEncoder.encode(request.getPassword())) // Hash password
            .build();
        user = userRepository.save(user); // INSERT INTO users ...

        // 4. Tạo JWT token
        String jwt = tokenProvider.generateToken(authentication);

        // 5. Trả về DTO (không trả Entity thẳng)
        return AuthResponse.builder()
            .userId(user.getId())
            .accessToken(jwt)
            .build();
    }
}
```

### Rule quan trọng trong Service

- **Không bao giờ** trả về Entity thẳng — luôn convert sang DTO
- `@Transactional` bao quanh những method thay đổi nhiều bảng cùng lúc
- Throw `RuntimeException` khi có lỗi — Controller sẽ catch

---

## 8. Tầng Controller — API Layer

Controller là nơi duy nhất tiếp nhận HTTP request.

### Ví dụ: `AuthenticationController.java`

```java
@RestController                          // = @Controller + @ResponseBody
@RequestMapping("/api/auth")            // Prefix cho tất cả route trong class
@CrossOrigin(origins = "*")             // Cho phép CORS từ mọi origin
public class AuthenticationController extends BaseController {

    // Constructor injection — không @Autowired
    public AuthenticationController(
        AuthenticationService authenticationService,
        OtpService otpService
    ) { ... }

    @PostMapping("/register")            // POST /api/auth/register
    public ResponseEntity<ApiResponse<AuthResponse>> register(
        @Valid @RequestBody RegisterRequest request  // Validate + parse body
    ) {
        try {
            AuthResponse response = authenticationService.register(request);
            return sendSuccess("User registered successfully", response); // 200 OK
        } catch (RuntimeException e) {
            return sendError(e.getMessage()); // 400 Bad Request
        }
    }

    @GetMapping("/me")                   // GET /api/auth/me
    public ResponseEntity<ApiResponse<UserMeResponse>> getMe(
        Authentication authentication   // Spring tự inject từ JWT token
    ) {
        String identifier = authentication.getName(); // Lấy email/phone từ token
        UserMeResponse response = authenticationService.getMe(identifier);
        return sendSuccess("Success", response);
    }
}
```

### `BaseController` — Helper methods chung

```java
// Mọi controller extends BaseController để dùng các method này:
sendSuccess("message", data)     // 200 OK
sendSuccess("message")           // 200 OK, no data
sendError("message")             // 400 Bad Request
sendError("message", status)     // HTTP status tùy chọn
sendUnauthorized("message")      // 401 Unauthorized
sendConflict("message")          // 409 Conflict
```

### HTTP Method Annotations

| Annotation | HTTP Method | Dùng khi |
|---|---|---|
| `@GetMapping` | GET | Lấy dữ liệu |
| `@PostMapping` | POST | Tạo mới |
| `@PutMapping` | PUT | Cập nhật toàn bộ |
| `@PatchMapping` | PATCH | Cập nhật một phần |
| `@DeleteMapping` | DELETE | Xóa |

### Parameter Annotations

```java
@PathVariable Long id          // /api/classes/{id}
@RequestParam String status    // /api/classes?status=ONGOING
@RequestBody CreateRequest req // Body của POST request
Authentication auth            // Thông tin user đang đăng nhập (từ JWT)
```

---

## 9. DTO — Data Transfer Object

DTO là class chỉ dùng để **truyền data** — không có logic, không phải Entity.

### Tại sao cần DTO?

```
Entity (DB)     DTO (API)
────────────    ─────────────────────────
password        KHÔNG trả password ra ngoài
deletedAt       KHÔNG trả ra ngoài
internalCode    KHÔNG trả ra ngoài
...             Chỉ trả những field client cần
```

### ApiResponse — Wrapper chuẩn của dự án

```java
// Mọi response đều được bọc trong ApiResponse<T>
{
    "success": true,
    "message": "Login successful",
    "timestamp": "2026-05-21T10:30:00",
    "data": {
        "userId": 123,
        "accessToken": "eyJhbGci...",
        "typeToken": "Bearer"
    }
}
```

```java
// Khi lỗi:
{
    "success": false,
    "message": "Email is already in use!",
    "timestamp": "2026-05-21T10:30:00",
    "data": null
}
```

### Request DTO — Validate với annotation

```java
public class RegisterRequest {
    @NotBlank(message = "Identifier is required")
    private String identifier;    // email hoặc phone

    @NotBlank
    @Size(min = 8, message = "Password must be at least 8 characters")
    private String password;

    private String firstName;
    private String lastName;
    private String fullName;
}
```

---

## 10. Security & Authentication

### JWT Flow

```
1. Client gửi POST /api/auth/login { email, password }
2. Server kiểm tra password với BCrypt
3. Server tạo JWT token (có chứa email + roles)
4. Server trả về: { accessToken: "eyJ..." }

5. Client gửi request tiếp theo với header:
   Authorization: Bearer eyJ...

6. JwtAuthenticationFilter đọc token từ header
7. Validate token (chưa hết hạn, chữ ký đúng)
8. Lấy username (email/phone) từ token
9. Load user từ DB → set vào SecurityContext
10. Request được phép tiếp tục
```

### JwtTokenProvider

```java
// Tạo token — gắn username và roles vào payload
String jwt = Jwts.builder()
    .subject(username)           // email hoặc phone
    .claim("roles", roles)       // ["ADMIN", "LECTURER"]
    .issuedAt(new Date())
    .expiration(new Date(now + 86400000)) // hết hạn sau 24 giờ
    .signWith(secretKey)
    .compact();

// Validate token — xác minh chữ ký
boolean valid = validateToken(token); // throws exception nếu hết hạn

// Lấy username từ token
String username = getUsernameFromToken(token); // trả về email/phone
```

### SecurityConfig — Các endpoint public vs private

```java
.authorizeHttpRequests(auth -> auth
    // Public — không cần đăng nhập
    .requestMatchers("/api/auth/**").permitAll()
    .requestMatchers("/swagger-ui/**").permitAll()
    .requestMatchers("/payment/return").permitAll()

    // Private — mọi request khác đều cần JWT
    .anyRequest().authenticated()
)
```

### Google OAuth2 Flow

```
1. Client redirect → GET /oauth2/authorization/google
2. Google hiện form đăng nhập
3. Google callback → GET /login/oauth2/code/google?code=...
4. Spring Security đổi code lấy Google profile
5. OAuth2SuccessHandler chạy → tạo/tìm user → tạo JWT
6. Redirect về FE với JWT token
```

### Lấy user hiện tại trong Service

```java
// Trong bất kỳ Service nào cũng dùng được:
public Long getCurrentUserId() {
    Authentication authentication = SecurityContextHolder.getContext().getAuthentication();
    String username = authentication.getName();
    User user = userRepository.findByEmailOrPhone(username, username).orElseThrow();
    return user.getId();
}
```

---

## 11. Configuration & Environment

### `application.properties` — Cấu hình ứng dụng

```properties
# Database
spring.datasource.url=jdbc:postgresql://lms-db:5432/lms-db
spring.jpa.hibernate.ddl-auto=update   # Tự update schema khi app khởi động

# JWT — bắt buộc set qua biến môi trường
app.jwt.secret=${JWT_SECRET:}           # Đọc từ env var JWT_SECRET
app.jwt.expiration=${JWT_EXPIRATION:86400000}  # 24h mặc định

# Cache — redis hoặc memory
app.cache.type=${CACHE_TYPE:redis}

# Storage — LOCAL hoặc AWS
app.storage.provider=${STORAGE_PROVIDER:LOCAL}
```

### Pattern đọc environment variable

```java
// Trong class Java:
@Value("${app.jwt.secret}")
private String jwtSecret;

// Với giá trị mặc định:
@Value("${app.cache.type:memory}")
private String cacheType;
```

### Các biến môi trường cần set khi chạy

| Biến | Mô tả | Bắt buộc |
|---|---|---|
| `JWT_SECRET` | Secret key để ký JWT (min 32 ký tự) | Production |
| `GOOGLE_CLIENT_ID` | Google OAuth Client ID | Khi dùng Google login |
| `GOOGLE_CLIENT_SECRET` | Google OAuth Secret | Khi dùng Google login |
| `REDIS_HOST` | Host của Redis server | Khi dùng Redis cache |
| `REDIS_PORT` | Port Redis (default: 6378) | Khi dùng Redis cache |
| `PAYMENT_PROVIDER` | Nhà cung cấp thanh toán | Khi dùng payment |
| `ME_SYNC_KEY` | API key đồng bộ Me platform | Khi dùng Me sync |
| `MISA_URL` | URL Misa API | Khi dùng Misa |

---

## 12. Design Patterns trong dự án

### Strategy Pattern — Authentication

```
AuthStrategyFactory
├── LocalAuthStrategy   (đăng nhập email/phone + password)
└── GoogleAuthStrategy  (đăng nhập Google OAuth)

// Sử dụng:
AuthStrategy strategy = strategyFactory.getStrategy(request.getAuthType());
User user = strategy.authenticate(request);  // dùng chung interface
```

### Factory Pattern — Cache, Storage, Payment

```
CacheServiceFactory
├── RedisCacheService    (dùng Redis)
└── InMemoryCacheService (dùng RAM)

StorageProviderFactory
├── LocalStorageProvider (lưu file local)
└── AwsStorageProvider   (lưu AWS S3)

PaymentServiceFactory
├── OnePayProvider       (cổng OnePay)
└── MePaymentProvider    (Me platform)
```

### Builder Pattern — Entity & DTO

```java
// Tạo User với Builder (nhờ @Builder của Lombok):
User user = User.builder()
    .email("user@example.com")
    .password(encoded)
    .firstName("Trường")
    .isActive(true)
    .build();
```

### Template Method — BaseController

```java
// BaseController định nghĩa template response:
protected <T> ResponseEntity<ApiResponse<T>> sendSuccess(String message, T data)
protected <T> ResponseEntity<ApiResponse<T>> sendError(String message)

// Controller con gọi lại:
return sendSuccess("Login successful", response);
```

---

## 13. Luồng xử lý một API hoàn chỉnh

### Ví dụ: `POST /api/auth/login`

**Request từ FE:**
```json
{
  "identifier": "user@example.com",
  "password": "abc123456",
  "authType": "LOCAL"
}
```

**Bước 1 — JwtAuthenticationFilter:**
```
→ Endpoint /api/auth/** là public → bỏ qua filter, đi thẳng vào controller
```

**Bước 2 — AuthenticationController.login():**
```java
// File: controller/AuthenticationController.java:41
@PostMapping("/login")
public ResponseEntity<ApiResponse<AuthResponse>> login(@Valid @RequestBody LoginRequest request) {
    AuthResponse response = authenticationService.login(request);
    return sendSuccess("Login successful", response);
}
```

**Bước 3 — AuthenticationService.login():**
```java
// File: service/AuthenticationService.java:119
public AuthResponse login(LoginRequest request) {
    AuthStrategy strategy = strategyFactory.getStrategy(request.getAuthType()); // → LocalAuthStrategy
    User user = strategy.authenticate(request); // Kiểm tra password với BCrypt
    String jwt = tokenProvider.generateToken(authentication); // Tạo JWT
    return AuthResponse.builder()
        .userId(user.getId())
        .accessToken(jwt)
        .build();
}
```

**Bước 4 — LocalAuthStrategy.authenticate():**
```java
// Dùng Spring Security AuthenticationManager để verify password
authenticationManager.authenticate(
    new UsernamePasswordAuthenticationToken(email, password)
);
```

**Bước 5 — Response trả về FE:**
```json
{
  "success": true,
  "message": "Login successful",
  "timestamp": "2026-05-21T10:30:00",
  "data": {
    "userId": 123,
    "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
    "typeToken": "Bearer"
  }
}
```

---

### Ví dụ: `GET /api/auth/me` (cần JWT)

**Bước 1 — JwtAuthenticationFilter:**
```
→ Đọc header: Authorization: Bearer eyJ...
→ Validate token → OK
→ getUsernameFromToken() → "user@example.com"
→ Load user từ DB → Set vào SecurityContext
→ Tiếp tục vào Controller
```

**Bước 2 — Controller nhận `Authentication` từ Spring:**
```java
@GetMapping("/me")
public ResponseEntity<?> getMe(Authentication authentication) {
    String identifier = authentication.getName(); // "user@example.com"
    UserMeResponse response = authenticationService.getMe(identifier);
    return sendSuccess("Success", response);
}
```

---

## 14. Tổng số file theo từng tầng

| Tầng | Số lượng | Đường dẫn |
|---|---|---|
| Entity | 75 | `db/entity/` |
| Repository | 80+ | `db/repository/` |
| Service | 64 | `service/` |
| Controller | 33 | `controller/` |
| Request DTO | 99 | `db/dto/request/` |
| Response DTO | 90 | `db/dto/response/` |
| Enum | 42 | `db/enums/` |
| Config | 11 | `config/` |
| Mapper | 7 | `mapper/` |
| Projection | 11 | `db/projection/` |
| **Tổng** | **~512** | |

---

## Checklist học BE cho FE Developer

- [ ] Hiểu luồng: Controller → Service → Repository → DB
- [ ] Biết đọc một Entity và nhận ra quan hệ bảng
- [ ] Biết viết Repository method từ tên
- [ ] Hiểu `@Transactional` là gì và khi nào dùng
- [ ] Hiểu JWT flow: login → token → bearer header → filter
- [ ] Hiểu sự khác biệt Entity vs DTO
- [ ] Biết đọc `application.properties` và biến môi trường
- [ ] Hiểu Dependency Injection (DI) là gì
- [ ] Biết dùng `ApiResponse<T>` để wrap response

---

*Tài liệu này được tạo dựa trên codebase thực tế của dự án Cole LMS.*  
*Cập nhật lần cuối: 2026-05-21*
