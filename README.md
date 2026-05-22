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
# Cole LMS — Tài Liệu Database

> Mô tả toàn bộ schema database PostgreSQL của dự án.  
> Mỗi bảng được trích xuất trực tiếp từ Entity class trong code.

---

## Mục Lục

1. [Tổng quan Database](#1-tổng-quan-database)
2. [Sơ đồ quan hệ tổng thể](#2-sơ-đồ-quan-hệ-tổng-thể)
3. [Nhóm 1 — User & Auth](#3-nhóm-1--user--auth)
4. [Nhóm 2 — Course (Khóa học)](#4-nhóm-2--course-khóa-học)
5. [Nhóm 3 — Class (Lớp học)](#5-nhóm-3--class-lớp-học)
6. [Nhóm 4 — Exam (Kiểm tra)](#6-nhóm-4--exam-kiểm-tra)
7. [Nhóm 5 — Integration (Tích hợp)](#7-nhóm-5--integration-tích-hợp)
8. [Nhóm 6 — Lecturer (Giảng viên)](#8-nhóm-6--lecturer-giảng-viên)
9. [Nhóm 7 — Learning Path (Lộ trình)](#9-nhóm-7--learning-path-lộ-trình)
10. [Nhóm 8 — Misa (Kế toán)](#10-nhóm-8--misa-kế-toán)
11. [Nhóm 9 — System (Hệ thống)](#11-nhóm-9--system-hệ-thống)
12. [Design Patterns trong Database](#12-design-patterns-trong-database)
13. [Enum toàn dự án](#13-enum-toàn-dự-án)
14. [Index & Constraint](#14-index--constraint)

---

## 1. Tổng quan Database

| Thông tin | Chi tiết |
|---|---|
| Loại DB | PostgreSQL |
| Host (Docker) | `lms-db:5432` |
| Database name | `lms-db` |
| ORM | Hibernate (Spring Data JPA) |
| DDL mode | `update` — tự update schema khi app restart |
| Naming | snake_case (camelCase trong Java → snake_case trong DB) |
| Tổng số bảng | ~75 bảng |

### Convention đặt tên bảng

```
Java Class Name      →   DB Table Name
─────────────────────────────────────────
User                 →   users
CourseClass          →   classes
ClassSession         →   class_sessions
StudentClass         →   student_classes
ExamSession          →   exam_session
QuestionTagBinding   →   question_tag_binding
```

---

## 2. Sơ đồ quan hệ tổng thể

```
                        ┌──────────┐
                        │  users   │
                        └────┬─────┘
              ┌──────────────┼──────────────┐
              │              │              │
         ┌────▼────┐   ┌─────▼────┐   ┌────▼────────┐
         │lecturers│   │ students │   │  role_data  │
         └────┬────┘   └────┬─────┘   └──────┬──────┘
              │             │                │
     ┌────────┴───┐    ┌────┴──────┐    ┌────▼────┐
     │   classes  │    │  student  │    │  role   │
     │(CourseClass│◄───│  _classes │    └─────────┘
     └──────┬─────┘    └───────────┘
            │
   ┌────────┼────────────────────┐
   │        │                    │
   ▼        ▼                    ▼
courses  class_sessions    exam_session
   │        │                    │
   ▼        ▼                    ▼
course_  class_session_      exams
modules  attendance           │
   │                          ▼
   ▼                       questions
lessons                        │
   │                           ▼
   ▼                      question_options
lesson_
resources
```

---

## 3. Nhóm 1 — User & Auth

### Bảng: `users`

> Tài khoản hệ thống — bao gồm cả admin, giảng viên, học viên.

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK, AUTO_INCREMENT | |
| `email` | VARCHAR | UNIQUE, NULLABLE | Email đăng nhập |
| `phone` | VARCHAR | UNIQUE, NULLABLE | Số điện thoại đăng nhập |
| `password` | VARCHAR | NULLABLE | BCrypt hashed (null nếu OAuth) |
| `auth_provider` | VARCHAR | | `LOCAL` hoặc `GOOGLE` |
| `email_verified` | BOOLEAN | | Đã xác thực email chưa |
| `phone_verified` | BOOLEAN | | Đã xác thực phone chưa |
| `is_active` | BOOLEAN | DEFAULT true | Tài khoản có đang hoạt động |
| `first_name` | VARCHAR | NOT NULL | |
| `last_name` | VARCHAR | NOT NULL | |
| `full_name` | VARCHAR | NOT NULL | |
| `last_login_timestamp` | TIMESTAMP | | Lần đăng nhập cuối |
| `created_at` | TIMESTAMP | | |
| `updated_at` | TIMESTAMP | | |

**Lưu ý:** Một user có thể đăng nhập bằng email **hoặc** phone — không bắt buộc cả hai.

---

### Bảng: `role`

> Danh sách vai trò trong hệ thống.

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `role_name` | VARCHAR | | `ADMIN`, `LECTURER`, `STUDENT` |
| `description` | VARCHAR | | |
| `created_at` | TIMESTAMP | | |
| `updated_at` | TIMESTAMP | | |
| `deleted_at` | TIMESTAMP | NULLABLE | Soft delete |

---

### Bảng: `role_data`

> Bảng junction: gán Role cho User (Many-to-Many có soft delete).

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `user_id` | BIGINT | FK → users.id | |
| `role_id` | BIGINT | FK → role.id | |
| `created_at` | TIMESTAMP | | |
| `updated_at` | TIMESTAMP | | |
| `deleted_at` | TIMESTAMP | NULLABLE | Soft delete — thu hồi role |

```
users (1) ──── (*) role_data (*) ──── (1) role
```

---

### Bảng: `otp_log`

> Lưu lịch sử OTP đã gửi (SMS/Email).

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `identifier` | VARCHAR | Email hoặc phone nhận OTP |
| `otp` | VARCHAR | Mã OTP |
| `expired_at` | TIMESTAMP | Thời hạn OTP |
| `is_verified` | BOOLEAN | Đã verify chưa |
| `created_at` | TIMESTAMP | |

---

## 4. Nhóm 2 — Course (Khóa học)

### Sơ đồ quan hệ nhóm Course

```
course_catalogs ──┐
course_types    ──┼──► courses ──► course_modules ──► lessons ──► lesson_resources
me_products     ──┘       │
                          ├──► course_config
                          ├──► course_lecturers
                          ├──► course_property
                          ├──► course_workflow
                          └──► course_combos
```

---

### Bảng: `courses`

> Khóa học — đơn vị nội dung chính.

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `me_product_id` | BIGINT | FK → me_products.id | Liên kết sản phẩm Me platform |
| `course_type_id` | BIGINT | FK → course_types.id | Loại khóa học |
| `course_catalog_id` | BIGINT | FK → course_catalogs.id | Danh mục |
| `description` | TEXT | | |
| `avatar_url` | VARCHAR | | Ảnh thumbnail |
| `intro_video_url` | VARCHAR | | Video giới thiệu |
| `status` | VARCHAR | | Enum `CourseStatus` |
| `is_draft` | BOOLEAN | | Đang soạn thảo |
| `is_visible` | BOOLEAN | DEFAULT false | Hiển thị công khai |
| `created_id` | BIGINT | FK → users.id | Người tạo |
| `updated_id` | BIGINT | FK → users.id | Người cập nhật |
| `created_at` | TIMESTAMP | | |
| `updated_at` | TIMESTAMP | | |
| `deleted_at` | TIMESTAMP | NULLABLE | Soft delete |

**CourseStatus enum:** `DRAFT` → `PENDING_REVIEW` → `APPROVED` / `REJECTED`

---

### Bảng: `course_types`

> Phân loại khóa học (VD: "Tiếng Anh", "Lập trình"...).

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `name` | VARCHAR | Tên loại |
| `description` | TEXT | |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |
| `deleted_at` | TIMESTAMP | Soft delete |

---

### Bảng: `course_catalogs`

> Danh mục / ngành học (VD: "Công nghệ thông tin").

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `name` | VARCHAR | |
| `description` | TEXT | |
| `created_at` | TIMESTAMP | |
| `updated_at` | TIMESTAMP | |
| `deleted_at` | TIMESTAMP | Soft delete |

---

### Bảng: `course_modules`

> Module/chương trong một khóa học.

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `course_id` | BIGINT | FK → courses.id | |
| `order_index` | INT | | Thứ tự hiển thị |
| `name` | VARCHAR | | Tên module |
| `description` | TEXT | | |
| `created_id` | BIGINT | FK → users.id | |
| `updated_id` | BIGINT | FK → users.id | |
| `created_at` | TIMESTAMP | | |
| `updated_at` | TIMESTAMP | | |
| `deleted_at` | TIMESTAMP | | Soft delete |

```
courses (1) ──── (*) course_modules (1) ──── (*) lessons
```

---

### Bảng: `lessons`

> Bài học trong một module.

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `course_module_id` | BIGINT | FK → course_modules.id | |
| `name` | VARCHAR | | Tên bài học |
| `description` | TEXT | | |
| `is_trial` | BOOLEAN | | Học thử miễn phí |
| `lesson_type` | VARCHAR | | Enum `LessonType` |
| `order_index` | INT | | Thứ tự |
| `created_id` | BIGINT | FK → users.id | |
| `updated_id` | BIGINT | FK → users.id | |
| `created_at` | TIMESTAMP | | |
| `updated_at` | TIMESTAMP | | |
| `deleted_at` | TIMESTAMP | | Soft delete |

**LessonType enum:** `VIDEO`, `DOCUMENT`, `EXERCISE`, `QUIZ`

---

### Bảng: `lesson_resources`

> Tài nguyên đính kèm bài học (file, video, link...).

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `lesson_id` | BIGINT | FK → lessons.id |
| `name` | VARCHAR | |
| `url` | VARCHAR | URL tài nguyên |
| `type` | VARCHAR | Loại tài nguyên |
| `order_index` | INT | |
| `created_at` | TIMESTAMP | |
| `deleted_at` | TIMESTAMP | Soft delete |

---

### Bảng: `course_config`

> Cấu hình riêng của từng khóa học (1-1 với courses).

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `course_id` | BIGINT | FK → courses.id (UNIQUE) |
| (các cột config) | | Cấu hình tùy chọn |

```
courses (1) ──── (1) course_config
```

---

### Bảng: `course_lecturers`

> Giảng viên phụ trách một khóa học (Many-to-Many).

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `course_id` | BIGINT | FK → courses.id |
| `lecturer_id` | BIGINT | FK → lecturers.id |
| `created_at` | TIMESTAMP | |

---

### Bảng: `course_workflow` / `course_workflow_steps` / `course_workflow_reviewers`

> Quy trình phê duyệt khóa học.

```
courses ──► course_workflow ──► course_workflow_steps ──► course_workflow_reviewers
```

**WorkflowStatus:** `PENDING` → `IN_REVIEW` → `APPROVED` / `REJECTED`

---

### Bảng: `course_property` / `course_property_options` / `course_property_selections`

> Thuộc tính động của khóa học (VD: "Trình độ", "Ngôn ngữ"...).

```
course_property (định nghĩa thuộc tính)
    └──► course_property_options (các lựa chọn)
              └──► course_property_selections (học viên đã chọn)
```

---

## 5. Nhóm 3 — Class (Lớp học)

### Sơ đồ quan hệ nhóm Class

```
courses ──► classes (CourseClass)
               │
    ┌──────────┼──────────────────────┐
    │          │                      │
    ▼          ▼                      ▼
student_   class_sessions         class_schedules
_classes       │
(enrollment)   ├──► class_session_attendance
               ├──► class_session_files
               ├──► class_session_assignment
               └──► class_session_override
```

---

### Bảng: `classes`

> Lớp học — một khóa học được mở thành nhiều lớp.

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `name` | VARCHAR | UNIQUE | Tên lớp |
| `code` | VARCHAR | UNIQUE | Mã lớp |
| `course_id` | BIGINT | FK → courses.id | |
| `main_lecturer_id` | BIGINT | FK → lecturers.id | Giảng viên chính |
| `assistant_id` | BIGINT | FK → lecturers.id | Trợ giảng |
| `student_count` | INT | | Số học viên hiện tại |
| `status` | VARCHAR | | Enum `ClassStatus` |
| `room_link` | VARCHAR | | Link phòng học online |
| `start_date` | DATE | | Ngày khai giảng |
| `duration_minutes` | INT | | Thời lượng mỗi buổi (phút) |
| `note` | TEXT | | |
| `created_at` | TIMESTAMP | | |
| `updated_at` | TIMESTAMP | | |

**ClassStatus:** `NOT_STARTED` → `ONGOING` → `COMPLETED` / `CANCELLED`

---

### Bảng: `student_classes`

> Học viên đăng ký vào lớp học (Many-to-Many).

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `student_id` | BIGINT | FK → students.id | |
| `class_id` | BIGINT | FK → classes.id | |
| `joined_at` | TIMESTAMP | | Ngày nhập học |
| `status` | VARCHAR | | Enum `ClassStudentStatus` |

**UNIQUE:** (`student_id`, `class_id`) — một học viên chỉ vào một lớp một lần.

**ClassStudentStatus:** `MAIN`, `RESERVE`, `TRANSFERRED`, `DROPPED`

---

### Bảng: `class_sessions`

> Buổi học cụ thể (ngày, giờ, giảng viên dạy).

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `class_id` | BIGINT | FK → classes.id | |
| `lesson_id` | BIGINT | FK → lessons.id, NULLABLE | Bài học được dạy buổi này |
| `order_index` | INT | | Số thứ tự buổi |
| `title` | VARCHAR | | Tiêu đề buổi học |
| `session_date` | DATE | | Ngày dạy |
| `start_time` | TIME | | Giờ bắt đầu |
| `end_time` | TIME | | Giờ kết thúc |
| `lecturer_id` | BIGINT | | Giảng viên dạy buổi này |
| `is_extra` | BOOLEAN | | Buổi học bù |
| `status` | VARCHAR | | Enum `ClassSessionStatus` |

**Index:** `idx_class_session_class` (class_id), `idx_class_session_date` (session_date)

**ClassSessionStatus:** `NOT_COMPLETED` → `COMPLETED` / `CANCELLED`

---

### Bảng: `class_session_attendance`

> Điểm danh học viên trong từng buổi học.

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `student_id` | BIGINT | FK → students.id | |
| `class_session_id` | BIGINT | FK → class_sessions.id | |
| `status` | VARCHAR | | Enum `AttendanceStatus` |
| `checked_at` | TIMESTAMP | | Thời gian điểm danh |
| `note` | TEXT | | |

**UNIQUE:** (`student_id`, `class_session_id`)

**AttendanceStatus:** `PRESENT`, `ABSENT`, `LATE`, `EXCUSED`

---

### Bảng: `class_schedules`

> Lịch học định kỳ của lớp (VD: Thứ 2-4-6, 18:00-20:00).

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `class_id` | BIGINT | FK → classes.id |
| `day_of_week` | INT | 1=Thứ 2, ..., 7=Chủ nhật |
| `start_time` | TIME | |
| `end_time` | TIME | |

---

### Bảng: `class_session_files`

> Tài liệu giáo viên upload trong buổi học.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `class_session_id` | BIGINT | FK → class_sessions.id |
| `file_id` | BIGINT | FK → files.id |
| `type` | VARCHAR | Enum `SessionFileType` |

---

### Bảng: `class_session_assignment`

> Bài tập được giao trong buổi học.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `class_session_id` | BIGINT | FK → class_sessions.id |
| `title` | VARCHAR | |
| `description` | TEXT | |
| `due_date` | TIMESTAMP | Deadline nộp bài |

---

### Bảng: `student_assignments`

> Bài nộp của học viên.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `assignment_id` | BIGINT | FK → class_session_assignment.id |
| `student_id` | BIGINT | FK → students.id |
| `submitted_at` | TIMESTAMP | |
| `status` | VARCHAR | Enum `StudentAssignmentStatus` |
| `score` | DECIMAL | Điểm chấm |
| `feedback` | TEXT | |

---

### Bảng: `class_score_config`

> Cấu hình thang điểm của lớp học (1-1 với classes).

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `class_id` | BIGINT | FK → classes.id (UNIQUE) |
| `score_type` | VARCHAR | Enum `ScoreType` |
| (các cột config điểm) | | |

---

### Bảng: `class_lecturers`

> Giảng viên tham gia dạy lớp (có thể nhiều giảng viên luân phiên).

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `class_id` | BIGINT | FK → classes.id |
| `lecturer_id` | BIGINT | FK → lecturers.id |
| `role` | VARCHAR | `MAIN`, `ASSISTANT` |

---

### Bảng: `class_session_override`

> Ghi đè thông tin buổi học (VD: đổi giờ, đổi giảng viên đột xuất).

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `class_session_id` | BIGINT | FK → class_sessions.id |
| `override_type` | VARCHAR | Enum `OverrideType` |
| `new_value` | TEXT | Giá trị mới |
| `reason` | TEXT | Lý do thay đổi |

---

### Nhóm Task trong Class

```
class_sessions ──► lesson_tasks (Nhiệm vụ học tập)
                       │
          ┌────────────┼────────────┐
          │            │            │
          ▼            ▼            ▼
   student_tasks  lesson_task_  lesson_task_
   (học viên      tickets       messages
   làm task)      (hỗ trợ)      (chat)
```

---

### Nhóm Post trong Class

> Bảng thảo luận / bài đăng trong lớp học.

```
class_posts ──► class_post_attachments
    │
    └──► class_post_comments
```

| Bảng | Mô tả |
|---|---|
| `class_posts` | Bài đăng trong lớp |
| `class_post_attachments` | File đính kèm bài đăng |
| `class_post_comments` | Bình luận |

---

## 6. Nhóm 4 — Exam (Kiểm tra)

### Sơ đồ quan hệ nhóm Exam

```
exams ──────────────────────────────────────┐
  │                                          │
  ├──► exam_questions                        │
  │       └──► exam_question_snapshots       │
  │               └──► exam_question_option_snapshots
  │                                          │
  ├──► exam_random_question_types            │
  ├──► exam_random_tags                      │
  └──► exam_random_difficulty_ratios         │
                                             │
exam_session ◄──────────────────────────────┘
  │  (class_id FK → classes)
  └──► exam_session_type
```

---

### Bảng: `exams`

> Đề thi — bộ câu hỏi được cấu hình.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `name` | VARCHAR | Tên đề thi |
| `mode` | VARCHAR | Enum `ExamMode`: `MANUAL` hoặc `RANDOM` |
| `total_questions` | INT | Tổng số câu hỏi |
| `duration_minutes` | INT | Thời gian làm bài |
| `total_score` | INT | Điểm tối đa |
| `is_shuffle_question` | BOOLEAN | Xáo trộn thứ tự câu hỏi |
| `is_shuffle_answer` | BOOLEAN | Xáo trộn đáp án |
| `is_show_result` | BOOLEAN | Hiển thị kết quả sau khi nộp |
| `attempt_limit` | INT | Số lần làm tối đa (null = không giới hạn) |
| `status` | VARCHAR | Enum `ExamStatus` |
| `is_draft` | BOOLEAN | |
| `created_id` | BIGINT | FK → users.id |
| `created_at` | TIMESTAMP | |
| `deleted_at` | TIMESTAMP | Soft delete |

**ExamMode:** `MANUAL` (chọn câu thủ công) | `RANDOM` (hệ thống random theo tiêu chí)

---

### Bảng: `questions`

> Ngân hàng câu hỏi — dùng chung cho nhiều đề thi.

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `question_type_id` | BIGINT | FK → question_types.id | |
| `course_catalog_id` | BIGINT | FK → course_catalogs.id | Thuộc ngành nào |
| `difficulty` | VARCHAR | | Enum `Difficulty` |
| `content` | TEXT | | Nội dung câu hỏi |
| `explanation` | TEXT | | Giải thích đáp án |
| `is_multiple_answer` | BOOLEAN | | Nhiều đáp án đúng |
| `is_active` | BOOLEAN | | Câu hỏi đang active |
| `allow_in_exam` | BOOLEAN | | Cho phép dùng trong thi |
| `created_id` | BIGINT | | |
| `created_at` | TIMESTAMP | | |
| `deleted_at` | TIMESTAMP | | Soft delete |

**Difficulty:** `EASY`, `MEDIUM`, `HARD`

---

### Bảng: `question_options`

> Các đáp án của câu hỏi trắc nghiệm.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `question_id` | BIGINT | FK → questions.id |
| `content` | TEXT | Nội dung đáp án |
| `is_correct` | BOOLEAN | Đây có phải đáp án đúng không |
| `order_index` | INT | Thứ tự hiển thị |

---

### Bảng: `question_types`

> Loại câu hỏi (VD: Trắc nghiệm, Tự luận, Điền vào chỗ trống...).

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `name` | VARCHAR | |
| `code` | VARCHAR | |

---

### Bảng: `question_tags` / `question_tag_binding`

> Tag/nhãn phân loại câu hỏi theo chủ đề.

```
questions (*) ──── (*) question_tags
     (qua bảng question_tag_binding)
```

---

### Bảng: `exam_session`

> Lần thi của một lớp học — gắn đề thi với lớp và thời gian mở thi.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `name` | VARCHAR | |
| `class_id` | BIGINT | FK → classes.id |
| `exam_id` | BIGINT | FK → exams.id |
| `exam_session_type_id` | BIGINT | FK → exam_session_types.id |
| `exam_session_name` | VARCHAR | |
| `description` | TEXT | |
| `start_time` | TIMESTAMP | Mở thi lúc |
| `end_time` | TIMESTAMP | Đóng thi lúc |
| `is_show_countdown` | BOOLEAN | Hiển thị đếm ngược |
| `is_auto_submit` | BOOLEAN | Tự nộp khi hết giờ |
| `created_id` | BIGINT | |
| `created_at` | TIMESTAMP | |

---

### Bảng: `exam_questions` / `exam_question_snapshots`

> Snapshot câu hỏi tại thời điểm thi — tránh bị ảnh hưởng khi câu hỏi gốc bị sửa.

```
exams ──► exam_questions (FK → questions)
               └──► exam_question_snapshots (bản sao tại thời điểm thi)
                         └──► exam_question_option_snapshots (bản sao đáp án)
```

---

## 7. Nhóm 5 — Integration (Tích hợp)

> Dữ liệu đồng bộ từ hệ thống ngoài: **Me e-learning platform**.

### Sơ đồ

```
me_products ──► courses (liên kết sản phẩm)

orders ──────────────────────────────────────────┐
  │                                               │
  ├──► order_items (FK → me_products)             │
  ├──► payment_schedules (lịch trả góp)           │
  └──► student_guardians ────► guardians          │
                                                  │
payment_transactions (lịch sử thanh toán) ────────┘
```

---

### Bảng: `students`

> Profile học viên (mở rộng từ users).

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `user_id` | BIGINT | FK → users.id, UNIQUE | 1-1 với users |
| `date_of_birth` | DATE | | |
| `gender` | VARCHAR | | Enum `Gender` |
| `address` | TEXT | | |
| `organization` | VARCHAR | | Tổ chức/trường |
| `education` | VARCHAR | | Trình độ học vấn |
| `created_at` | TIMESTAMP | | |
| `updated_at` | TIMESTAMP | | |

```
users (1) ──── (1) students
```

---

### Bảng: `orders`

> Đơn hàng mua khóa học (sync từ Me platform).

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `me_order_id` | VARCHAR | ID đơn hàng bên Me (UNIQUE) |
| `status` | VARCHAR | Enum `OrderStatus` |
| `total_amount` | DECIMAL | Tổng tiền |
| `paid_amount` | DECIMAL | Đã thanh toán |
| `remaining_amount` | DECIMAL | Còn phải trả |
| `installment_count` | INT | Số kỳ trả góp |
| `cancelled_at` | TIMESTAMP | |
| `created_at` | TIMESTAMP | |

**OrderStatus:** `PENDING` → `PAID` / `CANCELLED` / `PARTIAL`

---

### Bảng: `order_items`

> Chi tiết sản phẩm trong đơn hàng.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `order_id` | BIGINT | FK → orders.id |
| `me_product_id` | BIGINT | FK → me_products.id |
| `quantity` | INT | |
| `unit_price` | DECIMAL | |

---

### Bảng: `payment_schedules`

> Lịch trả góp của đơn hàng.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `order_id` | BIGINT | FK → orders.id |
| `due_date` | DATE | Ngày đến hạn |
| `amount` | DECIMAL | Số tiền kỳ này |
| `status` | VARCHAR | Enum `PaymentScheduleStatus` |

---

### Bảng: `payment_transactions`

> Lịch sử giao dịch thanh toán thực tế.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `order_id` | BIGINT | FK → orders.id |
| `amount` | DECIMAL | |
| `method` | VARCHAR | Enum `PaymentMethod` |
| `provider` | VARCHAR | Enum `PaymentProviderType` |
| `status` | VARCHAR | Enum `PaymentStatus` |
| `transaction_ref` | VARCHAR | Mã giao dịch từ cổng thanh toán |
| `paid_at` | TIMESTAMP | |

---

### Bảng: `guardians` / `student_guardians`

> Phụ huynh / người giám hộ của học viên.

```
students (*) ──── (*) guardians
     (qua bảng student_guardians)
```

---

### Bảng: `me_products`

> Sản phẩm đồng bộ từ Me e-learning platform.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `me_product_id` | VARCHAR | ID bên Me (UNIQUE) |
| `name` | VARCHAR | |
| `price` | DECIMAL | |
| `created_at` | TIMESTAMP | |

---

## 8. Nhóm 6 — Lecturer (Giảng viên)

### Bảng: `lecturers`

> Profile giảng viên (mở rộng từ users).

| Cột | Kiểu | Constraint | Mô tả |
|---|---|---|---|
| `id` | BIGINT | PK | |
| `user_id` | BIGINT | FK → users.id, UNIQUE | 1-1 với users |
| `code` | VARCHAR | NOT NULL, UNIQUE | Mã giảng viên |
| `date_of_birth` | DATE | | |
| `avatar_url` | VARCHAR | | |
| `degree` | VARCHAR | | Bằng cấp |
| `expertise` | VARCHAR | | Chuyên môn |
| `address` | TEXT | | |
| `gender` | VARCHAR | | Enum `Gender` |
| `education_level` | VARCHAR | | Trình độ học vấn |
| `teaching_role` | VARCHAR | | Enum `TeachingRole` |
| `identification_number` | VARCHAR | | CCCD/CMND |
| `experience` | TEXT | | Kinh nghiệm |
| `is_active` | BOOLEAN | DEFAULT true | |
| `note` | TEXT | | |
| `created_at` | TIMESTAMP | | |
| `updated_at` | TIMESTAMP | | |
| `deleted_at` | TIMESTAMP | | Soft delete |

**TeachingRole:** `LECTURER`, `ASSISTANT`, `TUTOR`

```
users (1) ──── (1) lecturers
```

---

### Bảng: `lecturer_contracts`

> Hợp đồng giảng dạy của giảng viên.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `lecturer_id` | BIGINT | FK → lecturers.id |
| `contract_type` | VARCHAR | Enum `LecturerContractType` |
| `status` | VARCHAR | Enum `ContractStatus` |
| `start_date` | DATE | |
| `end_date` | DATE | |
| `salary` | DECIMAL | |

---

## 9. Nhóm 7 — Learning Path (Lộ trình)

### Bảng: `learning_paths` / `learning_path_courses`

> Lộ trình học — tập hợp nhiều khóa học theo thứ tự.

```
learning_paths (1) ──── (*) learning_path_courses (*) ──── (1) courses
```

| Bảng | Mô tả |
|---|---|
| `learning_paths` | Lộ trình tổng thể |
| `learning_path_courses` | Khóa học trong lộ trình (có order_index) |

---

## 10. Nhóm 8 — Misa (Kế toán)

> Dữ liệu đồng bộ từ phần mềm kế toán **Misa**.

| Bảng | Mô tả |
|---|---|
| `misa_account_objects` | Đối tượng kế toán |
| `misa_inventory_items` | Mặt hàng tồn kho |
| `misa_organization_units` | Đơn vị tổ chức |
| `misa_units` | Đơn vị tính |
| `sync_metadata` | Metadata theo dõi lần sync cuối |

---

## 11. Nhóm 9 — System (Hệ thống)

### Bảng: `files`

> Quản lý file upload.

| Cột | Kiểu | Mô tả |
|---|---|---|
| `id` | BIGINT | PK |
| `name` | VARCHAR | Tên file gốc |
| `storage_key` | VARCHAR | Đường dẫn lưu trong storage |
| `mime_type` | VARCHAR | `image/png`, `application/pdf`... |
| `size` | BIGINT | Kích thước (bytes) |
| `status` | VARCHAR | Enum `FileStatus` |
| `created_at` | TIMESTAMP | |

---

### Bảng: `search_keywords` / `course_search_keywords`

> Index từ khóa tìm kiếm.

---

## 12. Design Patterns trong Database

### Pattern 1: Soft Delete

> **Không xóa dữ liệu thật** — chỉ set `deleted_at`.

```sql
-- Thay vì DELETE:
UPDATE courses SET deleted_at = NOW() WHERE id = 1;

-- Query luôn filter:
SELECT * FROM courses WHERE deleted_at IS NULL;
```

Các bảng có soft delete: `users` (không có), `role`, `role_data`, `courses`, `course_modules`, `lessons`, `lecturers`, `exams`, `questions`...

---

### Pattern 2: Audit Fields

> Mọi bảng có `created_at` / `updated_at`, một số bảng có `created_id` / `updated_id`.

```java
@PrePersist
protected void onCreate() {
    createdAt = LocalDateTime.now();  // Tự set khi INSERT
    updatedAt = LocalDateTime.now();
}

@PreUpdate
protected void onUpdate() {
    updatedAt = LocalDateTime.now();  // Tự set khi UPDATE
}
```

---

### Pattern 3: Snapshot Pattern (Exam)

> Sao chép câu hỏi tại thời điểm thi — tránh bị ảnh hưởng khi câu hỏi gốc bị sửa/xóa.

```
questions (ngân hàng gốc)
    └──► exam_questions (câu hỏi được chọn vào đề)
              └──► exam_question_snapshots (BẢN SAO khi mở thi)
                        └──► exam_question_option_snapshots
```

Khi học viên làm bài, hệ thống đọc từ `exam_question_snapshots` thay vì `questions`.

---

### Pattern 4: User Profile Extension

> `users` là bảng gốc — mở rộng bằng bảng riêng theo vai trò.

```
users (1) ──── (1) students    [thêm: ngày sinh, giới tính, địa chỉ]
users (1) ──── (1) lecturers   [thêm: mã GV, bằng cấp, chuyên môn]
```

Không dùng inheritance trong DB — dùng quan hệ 1-1.

---

### Pattern 5: Junction Table (Many-to-Many)

| Junction Table | Nối hai bảng | Có soft delete? |
|---|---|---|
| `role_data` | `users` ↔ `role` | Có (`deleted_at`) |
| `student_classes` | `students` ↔ `classes` | Không (có `status`) |
| `course_lecturers` | `courses` ↔ `lecturers` | Không |
| `class_lecturers` | `classes` ↔ `lecturers` | Không |
| `question_tag_binding` | `questions` ↔ `question_tags` | Không |
| `learning_path_courses` | `learning_paths` ↔ `courses` | Không |

---

### Pattern 6: Configurable via Enum stored as String

> Tất cả enum lưu dạng `VARCHAR` trong DB (không lưu số).

```java
@Enumerated(EnumType.STRING)  // Lưu "ONGOING" thay vì 1
private ClassStatus status;
```

**Ưu điểm:** Dễ đọc khi query thẳng DB, không bị ảnh hưởng khi thêm/xóa enum value.

---

## 13. Enum toàn dự án

| Enum | Giá trị | Dùng ở bảng |
|---|---|---|
| `AuthType` | `LOCAL`, `GOOGLE` | `users.auth_provider` |
| `CourseStatus` | `DRAFT`, `PENDING_REVIEW`, `APPROVED`, `REJECTED` | `courses.status` |
| `ClassStatus` | `NOT_STARTED`, `ONGOING`, `COMPLETED`, `CANCELLED` | `classes.status` |
| `ClassSessionStatus` | `NOT_COMPLETED`, `COMPLETED`, `CANCELLED` | `class_sessions.status` |
| `ClassStudentStatus` | `MAIN`, `RESERVE`, `TRANSFERRED`, `DROPPED` | `student_classes.status` |
| `AttendanceStatus` | `PRESENT`, `ABSENT`, `LATE`, `EXCUSED` | `class_session_attendance.status` |
| `ExamMode` | `MANUAL`, `RANDOM` | `exams.mode` |
| `ExamStatus` | `DRAFT`, `ACTIVE`, `CLOSED` | `exams.status` |
| `Difficulty` | `EASY`, `MEDIUM`, `HARD` | `questions.difficulty` |
| `LessonType` | `VIDEO`, `DOCUMENT`, `EXERCISE`, `QUIZ` | `lessons.lesson_type` |
| `OrderStatus` | `PENDING`, `PAID`, `CANCELLED`, `PARTIAL` | `orders.status` |
| `PaymentStatus` | `PENDING`, `SUCCESS`, `FAILED` | `payment_transactions.status` |
| `PaymentMethod` | `CASH`, `TRANSFER`, `ONEPAY` | `payment_transactions.method` |
| `Gender` | `MALE`, `FEMALE`, `OTHER` | `students.gender`, `lecturers.gender` |
| `RoleType` | `ADMIN`, `LECTURER`, `STUDENT` | `role.role_name` |
| `TeachingRole` | `LECTURER`, `ASSISTANT`, `TUTOR` | `lecturers.teaching_role` |
| `WorkflowStatus` | `PENDING`, `IN_REVIEW`, `APPROVED`, `REJECTED` | `course_workflow.status` |
| `FileStatus` | `ACTIVE`, `DELETED` | `files.status` |
| `StorageProviderType` | `LOCAL`, `AWS` | config |
| `PaymentProviderType` | `ONEPAY`, `ME` | config |

---

## 14. Index & Constraint

### Index quan trọng

```sql
-- class_sessions: query theo lớp và ngày học (thường xuyên dùng)
CREATE INDEX idx_class_session_class ON class_sessions(class_id);
CREATE INDEX idx_class_session_date  ON class_sessions(session_date);
```

### Unique Constraints

```sql
-- users
UNIQUE (email)
UNIQUE (phone)

-- classes
UNIQUE (name)
UNIQUE (code)

-- lecturers
UNIQUE (code)

-- student_classes: 1 học viên chỉ vào 1 lớp 1 lần
UNIQUE (student_id, class_id)

-- class_session_attendance: 1 học viên điểm danh 1 lần/buổi
UNIQUE (student_id, class_session_id)

-- orders: đồng bộ Me
UNIQUE (me_order_id)

-- me_products
UNIQUE (me_product_id)
```

---

## Tóm tắt số lượng bảng theo nhóm

| Nhóm | Số bảng | Chức năng |
|---|---|---|
| User & Auth | 4 | Đăng nhập, phân quyền, OTP |
| Course | 16 | Khóa học, module, bài học, duyệt bài |
| Class | 16 | Lớp học, lịch học, điểm danh, bài tập |
| Exam | 12 | Đề thi, câu hỏi, lần thi |
| Integration | 7 | Học viên, đơn hàng, thanh toán |
| Lecturer | 2 | Giảng viên, hợp đồng |
| Learning Path | 2 | Lộ trình học |
| Misa | 5 | Kế toán |
| System | 4 | File, từ khóa, sync |
| **Tổng** | **~68** | |

---

*Tài liệu này được tạo dựa trên Entity classes trong codebase Cole LMS.*  
*Cập nhật lần cuối: 2026-05-21*
