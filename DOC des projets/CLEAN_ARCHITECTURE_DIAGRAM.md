# Schéma Circulaire - Clean Architecture

## Le Cercle de Clean Architecture (Vue Générale)

```
                        ┌─────────────────────────────────────────────┐
                        │                                             │
                        │        FRAMEWORKS & DRIVERS                 │
                        │    (Web, DB, UI, External APIs)            │
                        │                                             │
                        │  ┌─────────────────────────────────────┐   │
                        │  │                                     │   │
                        │  │    INTERFACE ADAPTERS               │   │
                        │  │  (Controllers, Gateways, Presenters)│   │
                        │  │                                     │   │
                        │  │  ┌─────────────────────────────┐   │   │
                        │  │  │                             │   │   │
                        │  │  │   APPLICATION BUSINESS      │   │   │
                        │  │  │        RULES                │   │   │
                        │  │  │     (Use Cases)             │   │   │
                        │  │  │                             │   │   │
                        │  │  │  ┌─────────────────────┐   │   │   │
                        │  │  │  │                     │   │   │   │
                        │  │  │  │   ENTERPRISE        │   │   │   │
                        │  │  │  │    BUSINESS         │   │   │   │
                        │  │  │  │     RULES           │   │   │   │
                        │  │  │  │  (Entities)         │   │   │   │
                        │  │  │  │                     │   │   │   │
                        │  │  │  └─────────────────────┘   │   │   │
                        │  │  │                             │   │   │
                        │  │  └─────────────────────────────┘   │   │
                        │  │                                     │   │
                        │  └─────────────────────────────────────┘   │
                        │                                             │
                        └─────────────────────────────────────────────┘

                    ───────────────────────────────────────────────────
                              RÈGLE DE DÉPENDANCE
                        Les dépendances pointent vers l'intérieur
                    ───────────────────────────────────────────────────
```

---

## Les 4 Couches en Détail

### 🔴 Couche 1 (Centre) - ENTITIES (Domain)
**Règles métier de l'entreprise**

```
┌────────────────────────────────────────┐
│         ENTITIES (Domain)              │
│                                        │
│  • Product                             │
│  • Order                               │
│  • User                                │
│  • Cart                                │
│  • Category                            │
│  • Address                             │
│                                        │
│  ✓ Pur Java (zéro dépendance)         │
│  ✓ Logique métier pure                │
│  ✓ Indépendant de tout framework      │
│                                        │
└────────────────────────────────────────┘
```

**Exemple**:
```java
public class Product {
    void applyDiscount(Money discount) {
        if (discount.isGreaterThan(price)) {
            throw new InvalidDiscountException();
        }
        this.discountPrice = discount;
    }
}
```

---

### 🟠 Couche 2 - USE CASES (Application)
**Règles métier de l'application**

```
┌────────────────────────────────────────┐
│      USE CASES (Application)           │
│                                        │
│  • CreateProductUseCase                │
│  • CreateOrderUseCase                  │
│  • AddToCartUseCase                    │
│  • LoginUseCase                        │
│                                        │
│  → Dépend de: ENTITIES                 │
│  ✓ Orchestration de la logique        │
│  ✓ Indépendant de l'UI et DB          │
│                                        │
└────────────────────────────────────────┘
```

**Exemple**:
```java
public class CreateProductUseCase {
    public Product execute(CreateProductCommand cmd) {
        // Orchestration de la logique métier
        Category category = categoryRepo.findById(cmd.categoryId);
        Product product = new Product(...);
        return productRepo.save(product);
    }
}
```

---

### 🟡 Couche 3 - INTERFACE ADAPTERS (Presentation/Infrastructure)
**Adaptateurs entre les use cases et le monde extérieur**

```
┌────────────────────────────────────────┐
│     INTERFACE ADAPTERS                 │
│                                        │
│  Presentation:                         │
│  • Controllers (REST API)              │
│  • DTOs Request/Response               │
│  • Mappers Domain ↔ DTO                │
│                                        │
│  Infrastructure:                       │
│  • Repository Implementations          │
│  • JPA Entities                        │
│  • Mappers Domain ↔ JPA                │
│  • Email Adapter                       │
│  • Payment Adapter                     │
│                                        │
│  → Dépend de: USE CASES & ENTITIES     │
│  ✓ Conversion de formats               │
│                                        │
└────────────────────────────────────────┘
```

**Exemple**:
```java
@RestController
public class ProductController {
    public ResponseEntity<ProductResponse> create(
        @RequestBody CreateProductRequest request) {

        // Adapter: Request → Command
        CreateProductCommand cmd = mapper.toCommand(request);

        // Use Case
        Product product = createProductUseCase.execute(cmd);

        // Adapter: Domain → Response
        return ResponseEntity.ok(mapper.toResponse(product));
    }
}
```

---

### 🟢 Couche 4 (Extérieur) - FRAMEWORKS & DRIVERS
**Détails techniques et frameworks**

```
┌────────────────────────────────────────┐
│      FRAMEWORKS & DRIVERS              │
│                                        │
│  • Spring Boot                         │
│  • JPA/Hibernate                       │
│  • MySQL                               │
│  • Spring Security + JWT               │
│  • JavaMailSender                      │
│  • Stripe SDK                          │
│  • MTN/Orange Money APIs               │
│                                        │
│  → Dépend de: INTERFACE ADAPTERS       │
│  ✓ Technologies concrètes              │
│                                        │
└────────────────────────────────────────┘
```

---

## Flux de Données (Dependency Rule)

```
┌──────────────┐         ┌──────────────┐         ┌──────────────┐         ┌──────────────┐
│              │         │              │         │              │         │              │
│  FRAMEWORKS  │  ────→  │  ADAPTERS    │  ────→  │  USE CASES   │  ────→  │   ENTITIES   │
│   & DRIVERS  │         │ (Interface)  │         │ (Application)│         │   (Domain)   │
│              │         │              │         │              │         │              │
└──────────────┘         └──────────────┘         └──────────────┘         └──────────────┘
      ↓                        ↓                        ↓                        ↓
  Détails              Conversion               Orchestration            Logique Métier
 techniques            de formats              des opérations                Pure
```

**IMPORTANT**: Les flèches montrent la direction des dépendances. Le code de la couche externe dépend de la couche interne, jamais l'inverse.

---

## Schéma Appliqué à Votre Projet TKD

```
                    ╔═══════════════════════════════════════════════╗
                    ║                                               ║
                    ║   FRAMEWORKS & DRIVERS (Couche 4)            ║
                    ║                                               ║
                    ║   • Spring Boot 3.5.6                         ║
                    ║   • MySQL Database                            ║
                    ║   • Spring Security + JWT                     ║
                    ║   • JavaMailSender (Email)                    ║
                    ║   • Stripe API                                ║
                    ║   • MTN Mobile Money API                      ║
                    ║   • Orange Money API                          ║
                    ║                                               ║
                    ║  ╔════════════════════════════════════════╗   ║
                    ║  ║                                        ║   ║
                    ║  ║   INTERFACE ADAPTERS (Couche 3)       ║   ║
                    ║  ║                                        ║   ║
                    ║  ║   Presentation Layer:                 ║   ║
                    ║  ║   ┌─────────────────────────────┐     ║   ║
                    ║  ║   │ ProductController           │     ║   ║
                    ║  ║   │ OrderController             │     ║   ║
                    ║  ║   │ AuthController              │     ║   ║
                    ║  ║   │ CartController              │     ║   ║
                    ║  ║   │ ProductRestMapper           │     ║   ║
                    ║  ║   │ CreateProductRequest (DTO)  │     ║   ║
                    ║  ║   │ ProductResponse (DTO)       │     ║   ║
                    ║  ║   └─────────────────────────────┘     ║   ║
                    ║  ║                                        ║   ║
                    ║  ║   Infrastructure Layer:               ║   ║
                    ║  ║   ┌─────────────────────────────┐     ║   ║
                    ║  ║   │ ProductJpaEntity            │     ║   ║
                    ║  ║   │ ProductRepositoryImpl       │     ║   ║
                    ║  ║   │ ProductJpaRepository        │     ║   ║
                    ║  ║   │ ProductMapper (JPA↔Domain)  │     ║   ║
                    ║  ║   │ EmailAdapter                │     ║   ║
                    ║  ║   │ StripePaymentAdapter        │     ║   ║
                    ║  ║   │ LocalFileStorageAdapter     │     ║   ║
                    ║  ║   │ JwtTokenProvider            │     ║   ║
                    ║  ║   └─────────────────────────────┘     ║   ║
                    ║  ║                                        ║   ║
                    ║  ║  ╔══════════════════════════════════╗ ║   ║
                    ║  ║  ║                                  ║ ║   ║
                    ║  ║  ║  USE CASES (Couche 2)           ║ ║   ║
                    ║  ║  ║                                  ║ ║   ║
                    ║  ║  ║  Product Module:                ║ ║   ║
                    ║  ║  ║  • CreateProductUseCase         ║ ║   ║
                    ║  ║  ║  • UpdateProductUseCase         ║ ║   ║
                    ║  ║  ║  • GetProductUseCase            ║ ║   ║
                    ║  ║  ║  • DeleteProductUseCase         ║ ║   ║
                    ║  ║  ║                                  ║ ║   ║
                    ║  ║  ║  Order Module:                  ║ ║   ║
                    ║  ║  ║  • CreateOrderUseCase           ║ ║   ║
                    ║  ║  ║  • CancelOrderUseCase           ║ ║   ║
                    ║  ║  ║  • ShipOrderUseCase             ║ ║   ║
                    ║  ║  ║                                  ║ ║   ║
                    ║  ║  ║  Auth Module:                   ║ ║   ║
                    ║  ║  ║  • RegisterUserUseCase          ║ ║   ║
                    ║  ║  ║  • LoginUseCase                 ║ ║   ║
                    ║  ║  ║  • ResetPasswordUseCase         ║ ║   ║
                    ║  ║  ║                                  ║ ║   ║
                    ║  ║  ║  Ports (Interfaces):            ║ ║   ║
                    ║  ║  ║  • EmailPort                    ║ ║   ║
                    ║  ║  ║  • PaymentPort                  ║ ║   ║
                    ║  ║  ║  • FileStoragePort              ║ ║   ║
                    ║  ║  ║                                  ║ ║   ║
                    ║  ║  ║  ╔════════════════════════════╗ ║ ║   ║
                    ║  ║  ║  ║                            ║ ║ ║   ║
                    ║  ║  ║  ║  ENTITIES (Couche 1)      ║ ║ ║   ║
                    ║  ║  ║  ║                            ║ ║ ║   ║
                    ║  ║  ║  ║  Domain Entities:          ║ ║ ║   ║
                    ║  ║  ║  ║  • Product                 ║ ║ ║   ║
                    ║  ║  ║  ║  • Order                   ║ ║ ║   ║
                    ║  ║  ║  ║  • User                    ║ ║ ║   ║
                    ║  ║  ║  ║  • Cart                    ║ ║ ║   ║
                    ║  ║  ║  ║  • Category                ║ ║ ║   ║
                    ║  ║  ║  ║                            ║ ║ ║   ║
                    ║  ║  ║  ║  Value Objects:            ║ ║ ║   ║
                    ║  ║  ║  ║  • Money                   ║ ║ ║   ║
                    ║  ║  ║  ║  • Email                   ║ ║ ║   ║
                    ║  ║  ║  ║  • ProductId               ║ ║ ║   ║
                    ║  ║  ║  ║  • OrderStatus             ║ ║ ║   ║
                    ║  ║  ║  ║                            ║ ║ ║   ║
                    ║  ║  ║  ║  Repository Interfaces:    ║ ║ ║   ║
                    ║  ║  ║  ║  • ProductRepository       ║ ║ ║   ║
                    ║  ║  ║  ║  • OrderRepository         ║ ║ ║   ║
                    ║  ║  ║  ║  • UserRepository          ║ ║ ║   ║
                    ║  ║  ║  ║                            ║ ║ ║   ║
                    ║  ║  ║  ╚════════════════════════════╝ ║ ║   ║
                    ║  ║  ║                                  ║ ║   ║
                    ║  ║  ╚══════════════════════════════════╝ ║   ║
                    ║  ║                                        ║   ║
                    ║  ╚════════════════════════════════════════╝   ║
                    ║                                               ║
                    ╚═══════════════════════════════════════════════╝
```

---

## Exemple Concret: Flux "Créer un Produit"

### Étape par étape à travers les couches

```
┌─────────────────────────────────────────────────────────────────────┐
│  1. CLIENT (Frontend Angular)                                      │
│     POST /api/products                                              │
│     Body: { name: "T-shirt", price: 15000, categoryId: 1 }        │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│  2. COUCHE 4: Framework (Spring Boot)                              │
│     • Reçoit la requête HTTP                                        │
│     • Validation @Valid                                             │
│     • Sérialisation JSON → Java                                     │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│  3. COUCHE 3: ProductController (Presentation)                     │
│     @PostMapping                                                    │
│     public ResponseEntity<ProductResponse> create(                 │
│         @RequestBody CreateProductRequest request) {               │
│                                                                     │
│         // Convertir Request → Command                             │
│         CreateProductCommand cmd = mapper.toCommand(request);      │
│                                                                     │
│         // Appeler Use Case                                        │
│         Product product = createProductUseCase.execute(cmd);       │
│                                                                     │
│         // Convertir Domain → Response                             │
│         return ResponseEntity.ok(mapper.toResponse(product));      │
│     }                                                               │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│  4. COUCHE 2: CreateProductUseCase (Application)                   │
│     @Service                                                        │
│     public class CreateProductUseCase {                            │
│         public Product execute(CreateProductCommand cmd) {         │
│                                                                     │
│             // Vérifier catégorie existe                           │
│             Category category = categoryRepo.findById(             │
│                 CategoryId.of(cmd.getCategoryId())                 │
│             );                                                      │
│                                                                     │
│             // Créer l'entité du domaine                           │
│             Product product = new Product(                         │
│                 null,                                               │
│                 cmd.getName(),                                      │
│                 new Money(cmd.getPrice(), "XAF"),                  │
│                 ...                                                 │
│             );                                                      │
│                                                                     │
│             // Sauvegarder via repository                          │
│             return productRepository.save(product);                │
│         }                                                           │
│     }                                                               │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│  5. COUCHE 1: Product (Domain Entity)                              │
│     public class Product {                                         │
│         private ProductId id;                                      │
│         private String name;                                       │
│         private Money price;                                       │
│         private CategoryId categoryId;                             │
│                                                                     │
│         // Constructeur avec validation métier                     │
│         public Product(...) {                                      │
│             if (name == null || name.isBlank()) {                  │
│                 throw new InvalidProductException(...);            │
│             }                                                       │
│             if (price.isNegativeOrZero()) {                        │
│                 throw new InvalidPriceException(...);              │
│             }                                                       │
│             // ... logique métier                                  │
│         }                                                           │
│     }                                                               │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│  6. COUCHE 3: ProductRepositoryImpl (Infrastructure)               │
│     @Repository                                                     │
│     public class ProductRepositoryImpl                             │
│             implements ProductRepository {                         │
│                                                                     │
│         public Product save(Product product) {                     │
│             // Convertir Domain → JPA Entity                       │
│             ProductJpaEntity entity =                              │
│                 mapper.toJpaEntity(product);                       │
│                                                                     │
│             // Sauvegarder via JPA                                 │
│             ProductJpaEntity saved =                               │
│                 jpaRepository.save(entity);                        │
│                                                                     │
│             // Convertir JPA Entity → Domain                       │
│             return mapper.toDomain(saved);                         │
│         }                                                           │
│     }                                                               │
└──────────────────────────────┬──────────────────────────────────────┘
                               │
                               ↓
┌─────────────────────────────────────────────────────────────────────┐
│  7. COUCHE 4: JPA/Hibernate + MySQL                                │
│     INSERT INTO products (name, price, category_id, ...)           │
│     VALUES ('T-shirt', 15000, 1, ...)                              │
└─────────────────────────────────────────────────────────────────────┘
```

---

## La Règle de Dépendance (Dependency Rule)

### ✅ AUTORISÉ (Dépendances vers l'intérieur)

```
Controller ────────→ UseCase ────────→ Entity
   (3)                (2)                (1)

Infrastructure ────→ UseCase ────────→ Entity
     (3)               (2)                (1)
```

### ❌ INTERDIT (Dépendances vers l'extérieur)

```
Entity ──────X────→ UseCase
 (1)                  (2)

UseCase ─────X────→ Controller
  (2)                  (3)

Entity ──────X────→ JPA Annotations
 (1)                  (4)
```

---

## Pattern Ports & Adapters (Hexagonal Architecture)

```
                            ┌─────────────────┐
                            │                 │
         ┌─────────────────►│   USE CASES     │◄─────────────────┐
         │                  │  (Application)  │                  │
         │                  │                 │                  │
         │                  └────────┬────────┘                  │
         │                           │                           │
         │                           ↓                           │
         │                  ┌─────────────────┐                  │
         │                  │                 │                  │
         │                  │    ENTITIES     │                  │
         │                  │    (Domain)     │                  │
         │                  │                 │                  │
         │                  └─────────────────┘                  │
         │                                                        │
         │                                                        │
    ┌────┴────┐                                            ┌─────┴────┐
    │         │                                            │          │
    │  REST   │  (Input Adapter)              (Output     │ Database │
    │   API   │   ProductController            Adapter)   │  MySQL   │
    │         │                                            │          │
    └─────────┘                                            └──────────┘

    ┌─────────┐                                            ┌──────────┐
    │         │                                            │          │
    │  Email  │                    PORT                    │  Stripe  │
    │ Service │  ◄──────────────  EmailPort  ────────────► │   API    │
    │         │      (Interface)                           │          │
    └─────────┘                                            └──────────┘
```

**Les Ports** = Interfaces définies dans la couche Application
**Les Adapters** = Implémentations dans la couche Infrastructure

---

## Avantages de cette Architecture

### 1. Indépendance des Frameworks
```
┌──────────────┐
│  Domain      │  ← Pas de dépendance à Spring, JPA, etc.
│  Entities    │  ← Pur Java
└──────────────┘  ← Testable sans infrastructure
```

### 2. Testabilité
```
Test Unitaire Domain:
  Product product = new Product(...);
  product.applyDiscount(new Money(5000, "XAF"));
  → Pas besoin de Spring, DB, HTTP

Test Unitaire Use Case:
  @Mock ProductRepository repository;
  CreateProductUseCase useCase = new CreateProductUseCase(repository);
  → Mocks simples
```

### 3. Indépendance de la Base de Données
```
ProductRepository (Interface dans Domain)
       ↑
       │ implements
       │
ProductRepositoryImpl (Infrastructure)
       ↓
  JpaRepository / MongoDB / Redis / InMemory
```
→ Facile de changer de MySQL vers PostgreSQL, MongoDB, etc.

### 4. Indépendance de l'UI
```
Use Cases ← appelés par:
  • REST API (Spring MVC)
  • GraphQL
  • gRPC
  • CLI
  • Job batch
```

---

## Structure des Packages dans Votre Projet

```
src/main/java/com/tdk/backend_tdk/

├── domain/                           ← COUCHE 1 (Entities)
│   ├── entity/
│   │   ├── Product.java
│   │   ├── Order.java
│   │   └── User.java
│   ├── valueobject/
│   │   ├── Money.java
│   │   ├── Email.java
│   │   └── ProductId.java
│   ├── repository/                   ← Interfaces (Ports)
│   │   ├── ProductRepository.java
│   │   └── OrderRepository.java
│   └── exception/
│       └── ProductNotFoundException.java
│
├── application/                      ← COUCHE 2 (Use Cases)
│   ├── usecase/
│   │   ├── product/
│   │   │   ├── CreateProductUseCase.java
│   │   │   └── GetProductUseCase.java
│   │   └── order/
│   │       └── CreateOrderUseCase.java
│   └── port/                         ← Ports Output
│       └── output/
│           ├── EmailPort.java
│           ├── PaymentPort.java
│           └── FileStoragePort.java
│
├── infrastructure/                   ← COUCHE 3 (Adapters)
│   ├── persistence/
│   │   ├── entity/
│   │   │   └── ProductJpaEntity.java
│   │   ├── repository/
│   │   │   ├── ProductJpaRepository.java
│   │   │   └── ProductRepositoryImpl.java
│   │   └── mapper/
│   │       └── ProductMapper.java
│   ├── email/
│   │   └── EmailAdapter.java
│   └── payment/
│       └── StripePaymentAdapter.java
│
├── presentation/                     ← COUCHE 3 (Adapters)
│   ├── rest/
│   │   └── ProductController.java
│   ├── dto/
│   │   ├── request/
│   │   └── response/
│   └── mapper/
│       └── ProductRestMapper.java
│
└── config/                           ← COUCHE 4 (Configuration)
    ├── SecurityConfig.java
    └── PersistenceConfig.java
```

---

## Résumé Visuel des Dépendances

```
┌─────────────────────────────────────────────────────────────────────┐
│                                                                     │
│  ProductController  ──────┐                                         │
│                           │                                         │
│  ProductRepositoryImpl ───┼──→  CreateProductUseCase               │
│                           │            ↓                            │
│  EmailAdapter  ───────────┘    ┌──────────────┐                    │
│                                │  Product     │                    │
│  StripeAdapter ────────────────│  (Entity)    │                    │
│                                └──────────────┘                    │
│                                                                     │
│  TOUTES les dépendances pointent vers le DOMAIN                    │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Conclusion

La Clean Architecture garantit:

✅ **Indépendance** - Le domaine ne dépend de rien
✅ **Testabilité** - Tests unitaires rapides sans infrastructure
✅ **Maintenabilité** - Changements isolés dans chaque couche
✅ **Flexibilité** - Changement facile de frameworks/technologies
✅ **Clarté** - Séparation claire des responsabilités

**Règle d'Or**: Les dépendances vont toujours **vers l'intérieur**, vers le domaine métier.
