# 📘 Guide de Développement Spring Boot — API, Clean Architecture & Microservices

> Guide de référence réutilisable pour tous tes projets Spring Boot.
> Version cible : **Spring Boot 3.x** / **Java 17+**

---

## Table des matières

1. [Prérequis & création du projet](#1-prérequis--création-du-projet)
2. [Clean Architecture — structure du projet](#2-clean-architecture--structure-du-projet)
3. [Configuration (application.yml & profils)](#3-configuration)
4. [CRUD générique — GET / POST / PUT / DELETE](#4-crud-générique)
5. [Gestion globale des erreurs & validation](#5-gestion-globale-des-erreurs--validation)
6. [Envoi de mails](#6-envoi-de-mails)
7. [Authentification OTP par téléphone (+ JWT)](#7-authentification-otp-par-téléphone)
8. [Appeler des API externes (+ client généré via Swagger)](#8-appeler-des-api-externes)
9. [Documentation Swagger / OpenAPI](#9-documentation-swagger--openapi)
10. [Bases des microservices](#10-bases-des-microservices)
11. [Checklist de démarrage d'un nouveau projet](#11-checklist)

---

## 1. Prérequis & création du projet

### Outils nécessaires
- **Java 17+** (LTS)
- **Maven** ou Gradle
- **IDE** : IntelliJ IDEA (recommandé)
- **Docker** (pour la BDD et les microservices)
- **Postman** ou Swagger UI pour tester

### Créer le projet
Va sur [https://start.spring.io](https://start.spring.io) et sélectionne :

| Champ | Valeur |
|---|---|
| Project | Maven |
| Language | Java |
| Spring Boot | 3.x |
| Packaging | Jar |
| Java | 17 ou 21 |

### Dépendances de base (pom.xml)

```xml
<dependencies>
    <!-- Web / REST -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- JPA / Base de données -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>

    <!-- Validation -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-validation</artifactId>
    </dependency>

    <!-- Sécurité -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-security</artifactId>
    </dependency>

    <!-- Mail -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-mail</artifactId>
    </dependency>

    <!-- Swagger / OpenAPI -->
    <dependency>
        <groupId>org.springdoc</groupId>
        <artifactId>springdoc-openapi-starter-webmvc-ui</artifactId>
        <version>2.6.0</version>
    </dependency>

    <!-- Lombok (moins de code boilerplate) -->
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>

    <!-- MapStruct (mapping Entity <-> DTO) -->
    <dependency>
        <groupId>org.mapstruct</groupId>
        <artifactId>mapstruct</artifactId>
        <version>1.6.2</version>
    </dependency>

    <!-- JWT -->
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-api</artifactId>
        <version>0.12.6</version>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-impl</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>io.jsonwebtoken</groupId>
        <artifactId>jjwt-jackson</artifactId>
        <version>0.12.6</version>
        <scope>runtime</scope>
    </dependency>

    <!-- Tests -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>
```

---

## 2. Clean Architecture — structure du projet

### Principe

La Clean Architecture sépare le code en couches. **Règle d'or : les dépendances pointent toujours vers l'intérieur** (le domaine ne dépend de rien).

```
        ┌─────────────────────────────────────┐
        │  Infrastructure (Web, BDD, Mail...) │  ← détails techniques
        │   ┌─────────────────────────────┐   │
        │   │   Application (Use Cases)   │   │  ← orchestration métier
        │   │   ┌─────────────────────┐   │   │
        │   │   │   Domain (Entités,  │   │   │  ← cœur métier PUR
        │   │   │   Ports/Interfaces) │   │   │     (aucune dépendance
        │   │   └─────────────────────┘   │   │      à Spring/JPA)
        │   └─────────────────────────────┘   │
        └─────────────────────────────────────┘
```

### Structure de packages (template générique à copier)

```
com.monentreprise.monprojet
│
├── domain/                          # ❤️ CŒUR MÉTIER — Java pur
│   ├── model/                       # Entités métier (pas d'annotations JPA)
│   │   └── Product.java
│   ├── port/
│   │   ├── in/                      # Ports d'entrée (interfaces des use cases)
│   │   │   └── ProductUseCase.java
│   │   └── out/                     # Ports de sortie (interfaces vers l'extérieur)
│   │       ├── ProductRepositoryPort.java
│   │       ├── NotificationPort.java
│   │       └── OtpSenderPort.java
│   └── exception/
│       └── ProductNotFoundException.java
│
├── application/                     # 🧠 CAS D'UTILISATION
│   └── service/
│       └── ProductService.java      # implémente ProductUseCase
│
├── infrastructure/                  # 🔌 DÉTAILS TECHNIQUES
│   ├── web/                         # Adaptateurs d'entrée (REST)
│   │   ├── controller/
│   │   │   └── ProductController.java
│   │   ├── dto/
│   │   │   ├── request/ProductRequest.java
│   │   │   └── response/ProductResponse.java
│   │   └── mapper/
│   │       └── ProductWebMapper.java
│   ├── persistence/                 # Adaptateurs de sortie (BDD)
│   │   ├── entity/
│   │   │   └── ProductEntity.java   # annotations JPA ici
│   │   ├── repository/
│   │   │   └── ProductJpaRepository.java
│   │   ├── adapter/
│   │   │   └── ProductRepositoryAdapter.java  # implémente le port
│   │   └── mapper/
│   │       └── ProductPersistenceMapper.java
│   ├── mail/
│   │   └── EmailNotificationAdapter.java
│   ├── external/                    # Clients d'API externes
│   │   └── SmsProviderAdapter.java
│   ├── security/
│   │   ├── SecurityConfig.java
│   │   ├── JwtService.java
│   │   └── JwtAuthFilter.java
│   └── config/
│       ├── OpenApiConfig.java
│       └── BeanConfig.java
│
└── MonProjetApplication.java
```

### Pourquoi cette structure ?

| Couche | Contenu | Dépend de |
|---|---|---|
| **domain** | Modèles métier, interfaces (ports), exceptions métier | Rien (Java pur) |
| **application** | Services qui implémentent la logique métier | domain uniquement |
| **infrastructure** | Controllers REST, JPA, Mail, SMS, Sécurité | domain + application |

✅ **Avantages** : tu peux changer de BDD, de fournisseur SMS ou de framework web **sans toucher au métier**. Les tests unitaires du domaine ne nécessitent ni Spring ni BDD.

---

## 3. Configuration

### `application.yml` (config de base)

```yaml
spring:
  application:
    name: mon-projet-service

  datasource:
    url: jdbc:postgresql://localhost:5432/mondb
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
    driver-class-name: org.postgresql.Driver

  jpa:
    hibernate:
      ddl-auto: update        # 'validate' en production + Flyway/Liquibase
    show-sql: true
    properties:
      hibernate:
        format_sql: true

  mail:
    host: smtp.gmail.com
    port: 587
    username: ${MAIL_USER}
    password: ${MAIL_PASSWORD}    # mot de passe d'application, jamais en dur !
    properties:
      mail:
        smtp:
          auth: true
          starttls:
            enable: true

server:
  port: 8080

# Propriétés personnalisées
app:
  jwt:
    secret: ${JWT_SECRET}
    expiration-ms: 86400000     # 24h
  otp:
    expiration-minutes: 5
    length: 6

springdoc:
  swagger-ui:
    path: /swagger-ui.html
  api-docs:
    path: /v3/api-docs
```

### Profils d'environnement

Crée `application-dev.yml`, `application-prod.yml`, puis lance avec :

```bash
java -jar app.jar --spring.profiles.active=prod
```

> ⚠️ **Règle absolue** : jamais de secrets (mots de passe, clés API) en dur dans le code ou le yml versionné. Utilise des variables d'environnement `${...}` ou un coffre (Vault, AWS Secrets Manager).

### Lire les propriétés personnalisées

```java
@ConfigurationProperties(prefix = "app.otp")
public record OtpProperties(int expirationMinutes, int length) {}
```

Active-le avec `@EnableConfigurationProperties(OtpProperties.class)` sur la classe principale.

---

## 4. CRUD générique

Exemple complet avec une entité `Product` — **remplace `Product` par n'importe quelle ressource de ton projet** (User, Order, Client...), le pattern est identique.

### 4.1 Domain — Modèle métier (Java pur)

```java
// domain/model/Product.java
public class Product {
    private Long id;
    private String name;
    private String description;
    private BigDecimal price;
    private Instant createdAt;

    // constructeurs, getters, setters
    // + logique métier ici, ex :
    public void applyDiscount(int percent) {
        if (percent < 0 || percent > 100)
            throw new IllegalArgumentException("Pourcentage invalide");
        this.price = price.multiply(BigDecimal.valueOf(100 - percent))
                          .divide(BigDecimal.valueOf(100));
    }
}
```

### 4.2 Domain — Ports (interfaces)

```java
// domain/port/in/ProductUseCase.java
public interface ProductUseCase {
    Product create(Product product);
    Product getById(Long id);
    List<Product> getAll();
    Product update(Long id, Product product);
    void delete(Long id);
}

// domain/port/out/ProductRepositoryPort.java
public interface ProductRepositoryPort {
    Product save(Product product);
    Optional<Product> findById(Long id);
    List<Product> findAll();
    void deleteById(Long id);
    boolean existsById(Long id);
}
```

### 4.3 Application — Service (use case)

```java
// application/service/ProductService.java
@Service
@RequiredArgsConstructor
public class ProductService implements ProductUseCase {

    private final ProductRepositoryPort repository;

    @Override
    public Product create(Product product) {
        return repository.save(product);
    }

    @Override
    public Product getById(Long id) {
        return repository.findById(id)
            .orElseThrow(() -> new ProductNotFoundException(id));
    }

    @Override
    public List<Product> getAll() {
        return repository.findAll();
    }

    @Override
    public Product update(Long id, Product product) {
        Product existing = getById(id);      // lève 404 si absent
        existing.setName(product.getName());
        existing.setDescription(product.getDescription());
        existing.setPrice(product.getPrice());
        return repository.save(existing);
    }

    @Override
    public void delete(Long id) {
        if (!repository.existsById(id)) throw new ProductNotFoundException(id);
        repository.deleteById(id);
    }
}
```

### 4.4 Infrastructure — Persistence (JPA)

```java
// infrastructure/persistence/entity/ProductEntity.java
@Entity
@Table(name = "products")
@Getter @Setter @NoArgsConstructor @AllArgsConstructor @Builder
public class ProductEntity {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    private String description;

    @Column(nullable = false)
    private BigDecimal price;

    @CreationTimestamp
    private Instant createdAt;
}

// infrastructure/persistence/repository/ProductJpaRepository.java
public interface ProductJpaRepository extends JpaRepository<ProductEntity, Long> {
    // requêtes dérivées automatiques :
    List<ProductEntity> findByNameContainingIgnoreCase(String name);
    // ou JPQL personnalisé :
    @Query("SELECT p FROM ProductEntity p WHERE p.price BETWEEN :min AND :max")
    List<ProductEntity> findByPriceRange(BigDecimal min, BigDecimal max);
}

// infrastructure/persistence/mapper/ProductPersistenceMapper.java
@Mapper(componentModel = "spring")
public interface ProductPersistenceMapper {
    Product toDomain(ProductEntity entity);
    ProductEntity toEntity(Product domain);
}

// infrastructure/persistence/adapter/ProductRepositoryAdapter.java
@Component
@RequiredArgsConstructor
public class ProductRepositoryAdapter implements ProductRepositoryPort {

    private final ProductJpaRepository jpaRepository;
    private final ProductPersistenceMapper mapper;

    @Override
    public Product save(Product product) {
        return mapper.toDomain(jpaRepository.save(mapper.toEntity(product)));
    }

    @Override
    public Optional<Product> findById(Long id) {
        return jpaRepository.findById(id).map(mapper::toDomain);
    }

    @Override
    public List<Product> findAll() {
        return jpaRepository.findAll().stream().map(mapper::toDomain).toList();
    }

    @Override
    public void deleteById(Long id) { jpaRepository.deleteById(id); }

    @Override
    public boolean existsById(Long id) { return jpaRepository.existsById(id); }
}
```

### 4.5 Infrastructure — DTO + Controller REST

```java
// infrastructure/web/dto/request/ProductRequest.java
public record ProductRequest(
    @NotBlank(message = "Le nom est obligatoire") String name,
    String description,
    @NotNull @Positive(message = "Le prix doit être positif") BigDecimal price
) {}

// infrastructure/web/dto/response/ProductResponse.java
public record ProductResponse(Long id, String name, String description,
                              BigDecimal price, Instant createdAt) {}

// infrastructure/web/mapper/ProductWebMapper.java
@Mapper(componentModel = "spring")
public interface ProductWebMapper {
    Product toDomain(ProductRequest request);
    ProductResponse toResponse(Product product);
}
```

```java
// infrastructure/web/controller/ProductController.java
@RestController
@RequestMapping("/api/v1/products")
@RequiredArgsConstructor
@Tag(name = "Products", description = "Gestion des produits")
public class ProductController {

    private final ProductUseCase productUseCase;
    private final ProductWebMapper mapper;

    @PostMapping
    @Operation(summary = "Créer un produit")
    public ResponseEntity<ProductResponse> create(@Valid @RequestBody ProductRequest request) {
        Product created = productUseCase.create(mapper.toDomain(request));
        return ResponseEntity
            .created(URI.create("/api/v1/products/" + created.getId()))
            .body(mapper.toResponse(created));
    }

    @GetMapping("/{id}")
    @Operation(summary = "Récupérer un produit par ID")
    public ResponseEntity<ProductResponse> getById(@PathVariable Long id) {
        return ResponseEntity.ok(mapper.toResponse(productUseCase.getById(id)));
    }

    @GetMapping
    @Operation(summary = "Lister tous les produits")
    public ResponseEntity<List<ProductResponse>> getAll() {
        return ResponseEntity.ok(
            productUseCase.getAll().stream().map(mapper::toResponse).toList());
    }

    @PutMapping("/{id}")
    @Operation(summary = "Mettre à jour un produit")
    public ResponseEntity<ProductResponse> update(@PathVariable Long id,
                                                  @Valid @RequestBody ProductRequest request) {
        Product updated = productUseCase.update(id, mapper.toDomain(request));
        return ResponseEntity.ok(mapper.toResponse(updated));
    }

    @DeleteMapping("/{id}")
    @Operation(summary = "Supprimer un produit")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        productUseCase.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### 4.6 Récapitulatif des codes HTTP à utiliser

| Opération | Méthode | Succès | Erreurs courantes |
|---|---|---|---|
| Créer | POST | 201 Created | 400 (validation), 409 (conflit) |
| Lire | GET | 200 OK | 404 Not Found |
| Modifier (complet) | PUT | 200 OK | 400, 404 |
| Modifier (partiel) | PATCH | 200 OK | 400, 404 |
| Supprimer | DELETE | 204 No Content | 404 |

### 4.7 Pagination (bonus indispensable)

```java
@GetMapping("/paged")
public ResponseEntity<Page<ProductResponse>> getPaged(
        @RequestParam(defaultValue = "0") int page,
        @RequestParam(defaultValue = "10") int size,
        @RequestParam(defaultValue = "id,desc") String sort) {
    String[] s = sort.split(",");
    Pageable pageable = PageRequest.of(page, size,
        Sort.by(Sort.Direction.fromString(s[1]), s[0]));
    // le service retourne une Page<Product> via le port
    return ResponseEntity.ok(productUseCase.getPaged(pageable).map(mapper::toResponse));
}
```

---

## 5. Gestion globale des erreurs & validation

### Exception métier

```java
// domain/exception/ProductNotFoundException.java
public class ProductNotFoundException extends RuntimeException {
    public ProductNotFoundException(Long id) {
        super("Produit introuvable avec l'id : " + id);
    }
}
```

### Handler global (`@RestControllerAdvice`)

```java
// infrastructure/web/GlobalExceptionHandler.java
@RestControllerAdvice
public class GlobalExceptionHandler {

    record ErrorResponse(int status, String message, Instant timestamp, Map<String, String> details) {}

    @ExceptionHandler(ProductNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(RuntimeException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(404, ex.getMessage(), Instant.now(), null));
    }

    // Erreurs de validation @Valid → 400 avec le détail par champ
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors()
          .forEach(e -> errors.put(e.getField(), e.getDefaultMessage()));
        return ResponseEntity.badRequest()
            .body(new ErrorResponse(400, "Erreur de validation", Instant.now(), errors));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        // logguer l'exception ici, ne jamais exposer la stacktrace au client
        return ResponseEntity.internalServerError()
            .body(new ErrorResponse(500, "Erreur interne du serveur", Instant.now(), null));
    }
}
```

### Annotations de validation les plus utiles

| Annotation | Usage |
|---|---|
| `@NotNull` / `@NotBlank` / `@NotEmpty` | Champ requis |
| `@Size(min=, max=)` | Longueur chaîne / collection |
| `@Email` | Format email |
| `@Positive` / `@Min` / `@Max` | Nombres |
| `@Pattern(regexp=)` | Regex (ex: téléphone) |
| `@Past` / `@Future` | Dates |

---

## 6. Envoi de mails

### 6.1 Port (domaine) + Adaptateur (infrastructure)

```java
// domain/port/out/NotificationPort.java
public interface NotificationPort {
    void sendEmail(String to, String subject, String body);
    void sendHtmlEmail(String to, String subject, String htmlBody);
}
```

```java
// infrastructure/mail/EmailNotificationAdapter.java
@Component
@RequiredArgsConstructor
@Slf4j
public class EmailNotificationAdapter implements NotificationPort {

    private final JavaMailSender mailSender;

    @Value("${spring.mail.username}")
    private String from;

    @Override
    public void sendEmail(String to, String subject, String body) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setFrom(from);
        message.setTo(to);
        message.setSubject(subject);
        message.setText(body);
        mailSender.send(message);
        log.info("Email envoyé à {}", to);
    }

    @Override
    public void sendHtmlEmail(String to, String subject, String htmlBody) {
        try {
            MimeMessage mime = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(mime, true, "UTF-8");
            helper.setFrom(from);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(htmlBody, true); // true = HTML
            // Pièce jointe (optionnel) :
            // helper.addAttachment("facture.pdf", new File("facture.pdf"));
            mailSender.send(mime);
        } catch (MessagingException e) {
            throw new IllegalStateException("Échec d'envoi de l'email", e);
        }
    }
}
```

### 6.2 API REST pour envoyer un mail

```java
public record EmailRequest(
    @NotBlank @Email String to,
    @NotBlank String subject,
    @NotBlank String body
) {}

@RestController
@RequestMapping("/api/v1/emails")
@RequiredArgsConstructor
@Tag(name = "Emails")
public class EmailController {

    private final NotificationPort notificationPort;

    @PostMapping
    @Operation(summary = "Envoyer un email")
    public ResponseEntity<Map<String, String>> send(@Valid @RequestBody EmailRequest request) {
        notificationPort.sendEmail(request.to(), request.subject(), request.body());
        return ResponseEntity.ok(Map.of("message", "Email envoyé avec succès"));
    }
}
```

### 6.3 Envoi asynchrone (recommandé)

L'envoi de mail est lent (1-3s) : ne bloque jamais la requête HTTP.

```java
// Sur la classe principale :
@EnableAsync

// Sur la méthode d'envoi :
@Async
@Override
public void sendEmail(String to, String subject, String body) { ... }
```

### 6.4 Conseils

- **Gmail** : active la validation en 2 étapes et génère un **mot de passe d'application** (jamais ton vrai mot de passe).
- En production, préfère un service dédié : **SendGrid, Mailgun, Amazon SES, Brevo**.
- Pour des emails HTML propres, utilise des **templates Thymeleaf** (`spring-boot-starter-thymeleaf`) :

```java
Context context = new Context();
context.setVariable("username", "Alice");
context.setVariable("otpCode", "483920");
String html = templateEngine.process("email/otp-template", context); // templates/email/otp-template.html
notificationPort.sendHtmlEmail(to, "Votre code OTP", html);
```

---

## 7. Authentification OTP par téléphone

### Principe du flux

```
1. POST /auth/otp/request   { phone: "+33612345678" }
   → génère un code à 6 chiffres, le stocke (hashé, avec expiration), l'envoie par SMS

2. POST /auth/otp/verify    { phone: "+33612345678", code: "483920" }
   → vérifie le code → si OK : renvoie un token JWT

3. Les appels suivants portent : Authorization: Bearer <jwt>
```

### 7.1 Domain

```java
// domain/model/OtpCode.java
public class OtpCode {
    private String phone;
    private String hashedCode;
    private Instant expiresAt;
    private int attempts;

    public boolean isExpired() { return Instant.now().isAfter(expiresAt); }
    public boolean maxAttemptsReached() { return attempts >= 3; }
    public void incrementAttempts() { this.attempts++; }
    // getters/setters/constructeurs
}

// domain/port/out/OtpSenderPort.java  (interface vers le fournisseur SMS)
public interface OtpSenderPort {
    void sendOtp(String phone, String code);
}

// domain/port/out/OtpRepositoryPort.java
public interface OtpRepositoryPort {
    void save(OtpCode otp);
    Optional<OtpCode> findByPhone(String phone);
    void deleteByPhone(String phone);
}

// domain/port/in/OtpAuthUseCase.java
public interface OtpAuthUseCase {
    void requestOtp(String phone);
    String verifyOtp(String phone, String code); // retourne le JWT
}
```

### 7.2 Application — Service OTP

```java
// application/service/OtpAuthService.java
@Service
@RequiredArgsConstructor
public class OtpAuthService implements OtpAuthUseCase {

    private final OtpRepositoryPort otpRepository;
    private final OtpSenderPort otpSender;
    private final JwtService jwtService;
    private final PasswordEncoder passwordEncoder; // BCrypt
    private final OtpProperties otpProperties;

    private static final SecureRandom RANDOM = new SecureRandom();

    @Override
    public void requestOtp(String phone) {
        String code = generateCode(otpProperties.length());

        OtpCode otp = new OtpCode();
        otp.setPhone(phone);
        otp.setHashedCode(passwordEncoder.encode(code)); // jamais en clair !
        otp.setExpiresAt(Instant.now().plus(otpProperties.expirationMinutes(), ChronoUnit.MINUTES));
        otp.setAttempts(0);

        otpRepository.save(otp);
        otpSender.sendOtp(phone, code);
    }

    @Override
    public String verifyOtp(String phone, String code) {
        OtpCode otp = otpRepository.findByPhone(phone)
            .orElseThrow(() -> new InvalidOtpException("Aucun code demandé pour ce numéro"));

        if (otp.isExpired()) {
            otpRepository.deleteByPhone(phone);
            throw new InvalidOtpException("Code expiré, veuillez en redemander un");
        }
        if (otp.maxAttemptsReached()) {
            otpRepository.deleteByPhone(phone);
            throw new InvalidOtpException("Trop de tentatives, veuillez redemander un code");
        }
        if (!passwordEncoder.matches(code, otp.getHashedCode())) {
            otp.incrementAttempts();
            otpRepository.save(otp);
            throw new InvalidOtpException("Code invalide");
        }

        otpRepository.deleteByPhone(phone); // usage unique
        return jwtService.generateToken(phone);
    }

    private String generateCode(int length) {
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < length; i++) sb.append(RANDOM.nextInt(10));
        return sb.toString();
    }
}
```

### 7.3 Infrastructure — Envoi du SMS (exemple Twilio)

```xml
<dependency>
    <groupId>com.twilio.sdk</groupId>
    <artifactId>twilio</artifactId>
    <version>10.6.4</version>
</dependency>
```

```java
// infrastructure/external/TwilioOtpSenderAdapter.java
@Component
@Slf4j
public class TwilioOtpSenderAdapter implements OtpSenderPort {

    @Value("${twilio.account-sid}") private String accountSid;
    @Value("${twilio.auth-token}")  private String authToken;
    @Value("${twilio.from-number}") private String fromNumber;

    @PostConstruct
    void init() { Twilio.init(accountSid, authToken); }

    @Override
    public void sendOtp(String phone, String code) {
        Message.creator(
            new PhoneNumber(phone),
            new PhoneNumber(fromNumber),
            "Votre code de vérification : " + code + " (valide 5 min)"
        ).create();
        log.info("OTP envoyé au {}", phone);
    }
}
```

> 💡 Grâce au port `OtpSenderPort`, tu peux remplacer Twilio par **Vonage, AWS SNS, Orange SMS API**, ou même un envoi par email, sans toucher au service métier. En dev, crée un `FakeOtpSenderAdapter` (`@Profile("dev")`) qui loggue simplement le code dans la console.

### 7.4 Infrastructure — JWT

```java
// infrastructure/security/JwtService.java
@Service
public class JwtService {

    @Value("${app.jwt.secret}") private String secret;          // min 256 bits (32 chars)
    @Value("${app.jwt.expiration-ms}") private long expirationMs;

    private SecretKey key() {
        return Keys.hmacShaKeyFor(secret.getBytes(StandardCharsets.UTF_8));
    }

    public String generateToken(String subject) {
        return Jwts.builder()
            .subject(subject)
            .issuedAt(new Date())
            .expiration(new Date(System.currentTimeMillis() + expirationMs))
            .signWith(key())
            .compact();
    }

    public String extractSubject(String token) {
        return Jwts.parser().verifyWith(key()).build()
            .parseSignedClaims(token).getPayload().getSubject();
    }

    public boolean isValid(String token) {
        try {
            Jwts.parser().verifyWith(key()).build().parseSignedClaims(token);
            return true;
        } catch (JwtException e) {
            return false;
        }
    }
}
```

```java
// infrastructure/security/JwtAuthFilter.java
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {

    private final JwtService jwtService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String header = request.getHeader("Authorization");
        if (header != null && header.startsWith("Bearer ")) {
            String token = header.substring(7);
            if (jwtService.isValid(token)) {
                String phone = jwtService.extractSubject(token);
                var auth = new UsernamePasswordAuthenticationToken(
                    phone, null, List.of(new SimpleGrantedAuthority("ROLE_USER")));
                SecurityContextHolder.getContext().setAuthentication(auth);
            }
        }
        chain.doFilter(request, response);
    }
}
```

```java
// infrastructure/security/SecurityConfig.java
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {

    private final JwtAuthFilter jwtAuthFilter;

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http
            .csrf(AbstractHttpConfigurer::disable)
            .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/auth/**",
                                 "/swagger-ui/**", "/swagger-ui.html",
                                 "/v3/api-docs/**").permitAll()
                .anyRequest().authenticated())
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() { return new BCryptPasswordEncoder(); }
}
```

### 7.5 Controller d'authentification

```java
public record OtpRequestDto(
    @NotBlank
    @Pattern(regexp = "^\\+[1-9]\\d{7,14}$", message = "Format international requis, ex: +33612345678")
    String phone
) {}

public record OtpVerifyDto(
    @NotBlank @Pattern(regexp = "^\\+[1-9]\\d{7,14}$") String phone,
    @NotBlank @Size(min = 6, max = 6) String code
) {}

@RestController
@RequestMapping("/api/v1/auth/otp")
@RequiredArgsConstructor
@Tag(name = "Authentification OTP")
public class OtpAuthController {

    private final OtpAuthUseCase otpAuthUseCase;

    @PostMapping("/request")
    @Operation(summary = "Demander un code OTP par SMS")
    public ResponseEntity<Map<String, String>> requestOtp(@Valid @RequestBody OtpRequestDto dto) {
        otpAuthUseCase.requestOtp(dto.phone());
        return ResponseEntity.ok(Map.of("message", "Code envoyé par SMS"));
    }

    @PostMapping("/verify")
    @Operation(summary = "Vérifier le code OTP et obtenir un JWT")
    public ResponseEntity<Map<String, String>> verifyOtp(@Valid @RequestBody OtpVerifyDto dto) {
        String token = otpAuthUseCase.verifyOtp(dto.phone(), dto.code());
        return ResponseEntity.ok(Map.of("accessToken", token, "tokenType", "Bearer"));
    }
}
```

### 7.6 Règles de sécurité OTP (à respecter absolument)

- ✅ Code **hashé** en base (BCrypt), jamais en clair
- ✅ **Expiration courte** (5 min max)
- ✅ **Limite de tentatives** (3 essais) puis invalidation
- ✅ **Usage unique** : supprimé après vérification réussie
- ✅ **Rate limiting** sur `/request` (ex: 1 demande / minute / numéro) pour éviter le SMS bombing — Bucket4j ou Redis
- ✅ Réponse identique que le numéro existe ou non (pas de fuite d'information)

---

## 8. Appeler des API externes

Trois approches selon le besoin. **`RestClient` est le choix par défaut en Spring Boot 3.2+.**

### 8.1 RestClient (synchrone, moderne — recommandé)

```java
// infrastructure/config/BeanConfig.java
@Configuration
public class BeanConfig {
    @Bean
    public RestClient restClient() {
        return RestClient.builder()
            .baseUrl("https://api.exemple.com")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}
```

```java
// infrastructure/external/WeatherApiAdapter.java
@Component
@RequiredArgsConstructor
public class WeatherApiAdapter implements WeatherPort {   // le port est défini dans domain/port/out

    private final RestClient restClient;

    // GET avec paramètres
    @Override
    public WeatherDto getWeather(String city) {
        return restClient.get()
            .uri(uriBuilder -> uriBuilder
                .path("/v1/weather")
                .queryParam("city", city)
                .build())
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError, (req, res) -> {
                throw new ExternalApiException("Ville introuvable : " + city);
            })
            .body(WeatherDto.class);
    }

    // POST avec corps + header d'authentification
    public PaymentResponse createPayment(PaymentRequest request, String apiKey) {
        return restClient.post()
            .uri("/v1/payments")
            .header("Authorization", "Bearer " + apiKey)
            .body(request)
            .retrieve()
            .body(PaymentResponse.class);
    }
}
```

### 8.2 WebClient (réactif / non-bloquant)

À utiliser pour de la haute concurrence ou du streaming. Nécessite `spring-boot-starter-webflux`.

```java
WebClient webClient = WebClient.builder().baseUrl("https://api.exemple.com").build();

Mono<WeatherDto> result = webClient.get()
    .uri("/v1/weather?city={city}", city)
    .retrieve()
    .bodyToMono(WeatherDto.class)
    .timeout(Duration.ofSeconds(5))
    .retryWhen(Retry.backoff(3, Duration.ofMillis(500)));
```

### 8.3 OpenFeign (déclaratif — idéal en microservices)

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-openfeign</artifactId>
</dependency>
```

```java
// Sur la classe principale :
@EnableFeignClients

// Le client = une simple interface :
@FeignClient(name = "payment-service", url = "${external.payment.url}")
public interface PaymentClient {

    @GetMapping("/api/v1/payments/{id}")
    PaymentDto getPayment(@PathVariable("id") Long id);

    @PostMapping("/api/v1/payments")
    PaymentDto createPayment(@RequestBody PaymentRequest request);
}
```

### 8.4 Consommer une API externe à partir de son Swagger (génération de client)

Quand une API externe fournit sa **spec OpenAPI/Swagger** (fichier `swagger.json` ou URL `/v3/api-docs`), tu peux **générer automatiquement le client Java** au lieu de l'écrire à la main.

#### Étape 1 — Récupérer la spec

```bash
curl https://api-externe.com/v3/api-docs -o src/main/resources/openapi/external-api.json
```

#### Étape 2 — Plugin Maven `openapi-generator`

```xml
<plugin>
    <groupId>org.openapitools</groupId>
    <artifactId>openapi-generator-maven-plugin</artifactId>
    <version>7.8.0</version>
    <executions>
        <execution>
            <goals><goal>generate</goal></goals>
            <configuration>
                <inputSpec>${project.basedir}/src/main/resources/openapi/external-api.json</inputSpec>
                <generatorName>java</generatorName>
                <library>restclient</library>   <!-- ou webclient / resttemplate / feign -->
                <apiPackage>com.monprojet.external.api</apiPackage>
                <modelPackage>com.monprojet.external.model</modelPackage>
                <configOptions>
                    <useJakartaEe>true</useJakartaEe>
                    <openApiNullable>false</openApiNullable>
                </configOptions>
            </configuration>
        </execution>
    </executions>
</plugin>
```

#### Étape 3 — Générer et utiliser

```bash
mvn clean compile   # les classes sont générées dans target/generated-sources
```

```java
@Configuration
public class ExternalApiConfig {
    @Bean
    public UsersApi usersApi() {                       // classe générée automatiiquement
        ApiClient apiClient = new ApiClient();
        apiClient.setBasePath("https://api-externe.com");
        apiClient.setApiKey("ma-cle");                 // selon l'auth de l'API
        return new UsersApi(apiClient);
    }
}

// Utilisation dans un adaptateur :
@Component
@RequiredArgsConstructor
public class ExternalUserAdapter implements ExternalUserPort {
    private final UsersApi usersApi;

    @Override
    public UserDto getUser(Long id) {
        return usersApi.getUserById(id);   // méthode typée générée depuis le swagger
    }
}
```

✅ **Avantages** : client typé, à jour avec la spec, zéro code manuel. Il suffit de régénérer si l'API évolue.

> 🏛️ **Clean Architecture** : le client généré reste dans `infrastructure/external`. Le domaine ne connaît que le **port** (interface). Ne laisse jamais les modèles générés remonter dans le domaine — mappe-les vers tes modèles métier.

### 8.5 Bonnes pratiques pour les appels externes

- **Timeout** systématique (connexion + lecture) — jamais d'appel sans limite
- **Retry avec backoff** pour les erreurs transitoires (5xx, timeout)
- **Circuit breaker** (Resilience4j) en microservices :

```java
@CircuitBreaker(name = "paymentService", fallbackMethod = "fallbackPayment")
public PaymentDto getPayment(Long id) { ... }

public PaymentDto fallbackPayment(Long id, Throwable t) {
    return PaymentDto.unavailable(); // réponse dégradée
}
```

---

## 9. Documentation Swagger / OpenAPI

Avec la dépendance `springdoc-openapi-starter-webmvc-ui`, tout est automatique :

| URL | Contenu |
|---|---|
| `http://localhost:8080/swagger-ui.html` | Interface interactive pour tester les API |
| `http://localhost:8080/v3/api-docs` | Spec OpenAPI JSON (à partager avec les consommateurs) |

### Configuration personnalisée

```java
// infrastructure/config/OpenApiConfig.java
@Configuration
public class OpenApiConfig {

    @Bean
    public OpenAPI customOpenAPI() {
        return new OpenAPI()
            .info(new Info()
                .title("Mon Projet API")
                .version("1.0")
                .description("API REST — Clean Architecture")
                .contact(new Contact().name("Mon Équipe").email("dev@monentreprise.com")))
            // Bouton "Authorize" pour tester avec un JWT :
            .addSecurityItem(new SecurityRequirement().addList("bearerAuth"))
            .components(new Components().addSecuritySchemes("bearerAuth",
                new SecurityScheme()
                    .type(SecurityScheme.Type.HTTP)
                    .scheme("bearer")
                    .bearerFormat("JWT")));
    }
}
```

### Annotations utiles sur les controllers

```java
@Tag(name = "Products", description = "Gestion des produits")     // sur la classe
@Operation(summary = "Créer un produit", description = "...")     // sur la méthode
@ApiResponses({
    @ApiResponse(responseCode = "201", description = "Créé"),
    @ApiResponse(responseCode = "400", description = "Validation échouée")
})
@Parameter(description = "Identifiant du produit")                // sur un paramètre
@Schema(description = "Prix en euros", example = "19.99")         // sur un champ de DTO
```

> 💡 **Workflow d'équipe** : ton `/v3/api-docs` est la spec que les autres équipes (front, autres microservices) utilisent pour **générer leurs clients** (cf. section 8.4). C'est le contrat de ton API.

---

## 10. Bases des microservices

### Architecture type

```
                        ┌──────────────┐
   Client ────────────▶ │  API Gateway │ (Spring Cloud Gateway) :8080
                        └──────┬───────┘
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
      ┌──────────────┐ ┌──────────────┐ ┌──────────────┐
      │ user-service │ │order-service │ │notif-service │
      │    :8081     │ │    :8082     │ │    :8083     │
      └──────┬───────┘ └──────┬───────┘ └──────┬───────┘
             ▼                ▼                ▼
         BDD users        BDD orders      (mail / SMS)
                                                
      ┌─────────────────┐   ┌──────────────────┐
      │ Eureka Server   │   │  Config Server   │
      │ (découverte)    │   │ (config central.)│
      └─────────────────┘   └──────────────────┘
```

**Règles fondamentales :**
- 1 microservice = 1 responsabilité métier = **1 base de données à lui**
- Chaque microservice garde sa Clean Architecture interne (sections 2 à 9)
- Communication : REST/Feign (synchrone) ou messages Kafka/RabbitMQ (asynchrone)

### 10.1 Service Discovery — Eureka

**Serveur** (projet dédié) :

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication { ... }
```

```yaml
# application.yml du serveur Eureka
server:
  port: 8761
eureka:
  client:
    register-with-eureka: false
    fetch-registry: false
```

**Chaque microservice (client)** :

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-netflix-eureka-client</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: user-service        # nom utilisé pour la découverte
eureka:
  client:
    service-url:
      defaultZone: http://localhost:8761/eureka/
```

### 10.2 API Gateway — Spring Cloud Gateway

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-gateway</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: api-gateway
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://user-service          # lb:// = load balancing via Eureka
          predicates:
            - Path=/api/v1/users/**
        - id: order-service
          uri: lb://order-service
          predicates:
            - Path=/api/v1/orders/**
```

> 🔐 C'est généralement à la **gateway** qu'on valide le JWT (filtre global), puis on propage l'identité aux services internes via un header.

### 10.3 Communication entre services (Feign + Eureka)

```java
// Dans order-service, appeler user-service par son NOM (pas d'URL en dur) :
@FeignClient(name = "user-service")
public interface UserClient {
    @GetMapping("/api/v1/users/{id}")
    UserDto getUser(@PathVariable("id") Long id);
}
```

Eureka résout automatiquement l'adresse + load balancing.

### 10.4 Configuration centralisée — Config Server

```xml
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-config-server</artifactId>
</dependency>
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { ... }
```

```yaml
# Le config server lit les fichiers depuis un dépôt Git :
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/mon-org/config-repo
```

Chaque service pointe vers lui :

```yaml
spring:
  config:
    import: optional:configserver:http://localhost:8888
```

### 10.5 Docker Compose de développement (BDD + services)

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_DB: mondb
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data

  eureka-server:
    build: ./eureka-server
    ports:
      - "8761:8761"

  api-gateway:
    build: ./api-gateway
    ports:
      - "8080:8080"
    depends_on:
      - eureka-server

  user-service:
    build: ./user-service
    depends_on:
      - postgres
      - eureka-server

volumes:
  pgdata:
```

### 10.6 Ordre de démarrage d'un écosystème microservices

1. **Config Server** (8888)
2. **Eureka Server** (8761)
3. **Microservices métier** (8081, 8082, …)
4. **API Gateway** (8080)

---

## 11. Checklist

À chaque nouveau projet Spring Boot, suis cet ordre :

- [ ] **1.** Créer le projet sur start.spring.io avec les dépendances (§1)
- [ ] **2.** Mettre en place la structure Clean Architecture (§2)
- [ ] **3.** Configurer `application.yml` + profils + variables d'environnement (§3)
- [ ] **4.** Créer la première ressource CRUD complète : Model → Ports → Service → Entity → Adapter → DTO → Controller (§4)
- [ ] **5.** Ajouter le `GlobalExceptionHandler` + validation (§5)
- [ ] **6.** Brancher Swagger et vérifier `http://localhost:8080/swagger-ui.html` (§9)
- [ ] **7.** Ajouter la sécurité : OTP + JWT si besoin (§7)
- [ ] **8.** Intégrer les services externes (mail §6, SMS §7.3, API externes §8) **toujours derrière un port**
- [ ] **9.** Si microservices : Eureka → Gateway → Feign → Config Server (§10)
- [ ] **10.** Écrire les tests (unitaires sur les services du domaine, `@WebMvcTest` sur les controllers, Testcontainers pour la BDD)

### Rappels transverses

| Règle | Pourquoi |
|---|---|
| Le domaine ne dépend de rien | Testable, portable, pérenne |
| Toute dépendance externe = un port + un adaptateur | Remplaçable sans toucher au métier |
| DTO ≠ Entity ≠ Model métier | Chaque couche a ses objets, mappés entre eux |
| Secrets en variables d'environnement | Sécurité |
| Versionner les API (`/api/v1/...`) | Évolutivité sans casser les clients |
| Codes HTTP corrects + erreurs structurées | API prévisible et professionnelle |
| Swagger = contrat de ton API | Génération de clients, doc vivante |

---

*Guide généré pour servir de référence permanente à tes projets Spring Boot.*
