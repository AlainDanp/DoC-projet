# Guide de Migration vers Clean Architecture

## Table des matières
1. [Principes de Clean Architecture](#principes-de-clean-architecture)
2. [Structure cible](#structure-cible)
3. [Plan de migration](#plan-de-migration)
4. [Exemples de transformation](#exemples-de-transformation)
5. [Bonnes pratiques](#bonnes-pratiques)

---

## Principes de Clean Architecture

### Les 4 couches principales

```
┌─────────────────────────────────────────┐
│   Presentation Layer (Controllers)      │  ← Frameworks & Drivers
├─────────────────────────────────────────┤
│   Application Layer (Use Cases)         │  ← Interface Adapters
├─────────────────────────────────────────┤
│   Domain Layer (Entities + Logic)       │  ← Enterprise Business Rules
├─────────────────────────────────────────┤
│   Infrastructure (DB, Email, APIs)      │  ← Frameworks & Drivers
└─────────────────────────────────────────┘
```

### Règle de dépendance
**Les dépendances vont toujours vers l'intérieur** :
- Presentation → Application → Domain
- Infrastructure → Application → Domain
- Le Domain ne dépend de rien (pur Java)

---

## Structure cible

### Organisation des packages

```
src/main/java/com/tdk/backend_tdk/
├── domain/                          # ← Couche Domain (Cœur métier)
│   ├── entity/                      # Entités métier pures
│   │   ├── Product.java
│   │   ├── Order.java
│   │   ├── User.java
│   │   └── ...
│   ├── valueobject/                 # Value Objects
│   │   ├── Email.java
│   │   ├── Money.java
│   │   ├── Address.java
│   │   └── OrderStatus.java
│   ├── repository/                  # Interfaces de repository (ports)
│   │   ├── ProductRepository.java
│   │   ├── OrderRepository.java
│   │   └── UserRepository.java
│   ├── service/                     # Services du domaine (logique métier)
│   │   ├── PricingService.java
│   │   └── OrderValidationService.java
│   └── exception/                   # Exceptions métier
│       ├── ProductNotFoundException.java
│       └── InsufficientStockException.java
│
├── application/                     # ← Couche Application (Use Cases)
│   ├── usecase/
│   │   ├── product/
│   │   │   ├── CreateProductUseCase.java
│   │   │   ├── UpdateProductUseCase.java
│   │   │   └── GetProductUseCase.java
│   │   ├── order/
│   │   │   ├── CreateOrderUseCase.java
│   │   │   ├── CancelOrderUseCase.java
│   │   │   └── GetOrderDetailsUseCase.java
│   │   └── auth/
│   │       ├── RegisterUserUseCase.java
│   │       └── LoginUseCase.java
│   ├── port/                        # Ports (interfaces pour infrastructure)
│   │   ├── output/
│   │   │   ├── EmailPort.java
│   │   │   ├── FileStoragePort.java
│   │   │   └── PaymentPort.java
│   │   └── input/
│   │       └── # Use case interfaces si besoin
│   └── dto/                         # DTOs pour use cases
│       ├── CreateProductCommand.java
│       └── ProductResponse.java
│
├── infrastructure/                  # ← Couche Infrastructure (Implémentations)
│   ├── persistence/                 # Implémentation JPA
│   │   ├── entity/                  # Entités JPA
│   │   │   ├── ProductJpaEntity.java
│   │   │   ├── OrderJpaEntity.java
│   │   │   └── UserJpaEntity.java
│   │   ├── repository/              # Repositories JPA
│   │   │   ├── ProductJpaRepository.java
│   │   │   └── ProductRepositoryImpl.java
│   │   └── mapper/                  # Mappers Domain ↔ JPA
│   │       ├── ProductMapper.java
│   │       └── OrderMapper.java
│   ├── email/                       # Implémentation email
│   │   └── EmailAdapter.java
│   ├── storage/                     # Implémentation stockage fichiers
│   │   └── LocalFileStorageAdapter.java
│   ├── payment/                     # Implémentation paiements
│   │   ├── StripePaymentAdapter.java
│   │   ├── MtnMoneyAdapter.java
│   │   └── OrangeMoneyAdapter.java
│   └── security/                    # Sécurité (JWT, etc.)
│       ├── JwtTokenProvider.java
│       └── JwtAuthenticationFilter.java
│
├── presentation/                    # ← Couche Presentation (API REST)
│   ├── rest/
│   │   ├── ProductController.java
│   │   ├── OrderController.java
│   │   └── AuthController.java
│   ├── dto/                         # DTOs pour API REST
│   │   ├── request/
│   │   │   └── CreateProductRequest.java
│   │   └── response/
│   │       └── ProductResponse.java
│   ├── mapper/                      # Mappers Request/Response ↔ Domain
│   │   └── ProductRestMapper.java
│   └── exception/                   # Exception handlers
│       └── GlobalExceptionHandler.java
│
└── config/                          # Configuration Spring
    ├── SecurityConfig.java
    ├── PersistenceConfig.java
    └── UseCaseConfig.java           # Beans pour use cases
```

---

## Plan de migration

### Phase 1: Préparation (Sans casser le code existant)

#### Étape 1.1: Créer la structure de packages
```bash
mkdir -p src/main/java/com/tdk/backend_tdk/domain/entity
mkdir -p src/main/java/com/tdk/backend_tdk/domain/valueobject
mkdir -p src/main/java/com/tdk/backend_tdk/domain/repository
mkdir -p src/main/java/com/tdk/backend_tdk/domain/service
mkdir -p src/main/java/com/tdk/backend_tdk/application/usecase
mkdir -p src/main/java/com/tdk/backend_tdk/application/port/output
mkdir -p src/main/java/com/tdk/backend_tdk/infrastructure/persistence/entity
mkdir -p src/main/java/com/tdk/backend_tdk/infrastructure/persistence/repository
mkdir -p src/main/java/com/tdk/backend_tdk/infrastructure/persistence/mapper
mkdir -p src/main/java/com/tdk/backend_tdk/presentation/rest
mkdir -p src/main/java/com/tdk/backend_tdk/presentation/dto/request
mkdir -p src/main/java/com/tdk/backend_tdk/presentation/dto/response
```

#### Étape 1.2: Identifier un module pilote
Choisissez un module simple pour commencer (exemple: **Product**).

---

### Phase 2: Migration d'un module (Product)

#### Étape 2.1: Créer l'entité du domaine

**Avant** (`model/Product.java` - JPA):
```java
@Entity
@Table(name = "products")
@Data
public class Product {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(nullable = false)
    private Double price;

    @ManyToOne
    @JoinColumn(name = "category_id")
    private Category category;
}
```

**Après** (`domain/entity/Product.java` - Pure):
```java
package com.tdk.backend_tdk.domain.entity;

import com.tdk.backend_tdk.domain.valueobject.Money;
import com.tdk.backend_tdk.domain.valueobject.ProductId;
import lombok.AllArgsConstructor;
import lombok.Getter;

/**
 * Entité du domaine - AUCUNE annotation JPA
 */
@Getter
@AllArgsConstructor
public class Product {
    private final ProductId id;
    private String name;
    private String description;
    private Money price;
    private Money discountPrice;
    private Integer stockQuantity;
    private CategoryId categoryId;
    private boolean active;

    // Logique métier
    public void applyDiscount(Money discountPrice) {
        if (discountPrice.isGreaterThan(price)) {
            throw new IllegalArgumentException("Discount price cannot exceed original price");
        }
        this.discountPrice = discountPrice;
    }

    public void increaseStock(int quantity) {
        if (quantity <= 0) {
            throw new IllegalArgumentException("Quantity must be positive");
        }
        this.stockQuantity += quantity;
    }

    public void decreaseStock(int quantity) {
        if (quantity > stockQuantity) {
            throw new InsufficientStockException("Not enough stock available");
        }
        this.stockQuantity -= quantity;
    }

    public boolean isInStock() {
        return stockQuantity > 0;
    }

    public boolean hasDiscount() {
        return discountPrice != null && discountPrice.isLessThan(price);
    }
}
```

#### Étape 2.2: Créer les Value Objects

**`domain/valueobject/Money.java`**:
```java
package com.tdk.backend_tdk.domain.valueobject;

import lombok.EqualsAndHashCode;
import lombok.Getter;

@Getter
@EqualsAndHashCode
public class Money {
    private final double amount;
    private final String currency;

    public Money(double amount, String currency) {
        if (amount < 0) {
            throw new IllegalArgumentException("Amount cannot be negative");
        }
        if (currency == null || currency.isBlank()) {
            throw new IllegalArgumentException("Currency is required");
        }
        this.amount = amount;
        this.currency = currency;
    }

    public Money add(Money other) {
        validateSameCurrency(other);
        return new Money(this.amount + other.amount, this.currency);
    }

    public Money subtract(Money other) {
        validateSameCurrency(other);
        return new Money(this.amount - other.amount, this.currency);
    }

    public boolean isGreaterThan(Money other) {
        validateSameCurrency(other);
        return this.amount > other.amount;
    }

    public boolean isLessThan(Money other) {
        validateSameCurrency(other);
        return this.amount < other.amount;
    }

    private void validateSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Cannot compare money with different currencies");
        }
    }
}
```

**`domain/valueobject/ProductId.java`**:
```java
package com.tdk.backend_tdk.domain.valueobject;

import lombok.EqualsAndHashCode;
import lombok.Getter;

@Getter
@EqualsAndHashCode
public class ProductId {
    private final Long value;

    public ProductId(Long value) {
        if (value == null || value <= 0) {
            throw new IllegalArgumentException("Product ID must be a positive number");
        }
        this.value = value;
    }

    public static ProductId of(Long value) {
        return new ProductId(value);
    }
}
```

#### Étape 2.3: Créer l'interface du repository (Port)

**`domain/repository/ProductRepository.java`**:
```java
package com.tdk.backend_tdk.domain.repository;

import com.tdk.backend_tdk.domain.entity.Product;
import com.tdk.backend_tdk.domain.valueobject.ProductId;
import com.tdk.backend_tdk.domain.valueobject.CategoryId;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;

import java.util.Optional;

/**
 * Port du repository - Interface dans le domaine
 * L'implémentation sera dans l'infrastructure
 */
public interface ProductRepository {
    Product save(Product product);
    Optional<Product> findById(ProductId id);
    Page<Product> findAll(Pageable pageable);
    Page<Product> findByCategory(CategoryId categoryId, Pageable pageable);
    Page<Product> findActiveProducts(Pageable pageable);
    Page<Product> findProductsOnSale(Pageable pageable);
    void deleteById(ProductId id);
    boolean existsById(ProductId id);
}
```

#### Étape 2.4: Créer les Use Cases

**`application/usecase/product/CreateProductUseCase.java`**:
```java
package com.tdk.backend_tdk.application.usecase.product;

import com.tdk.backend_tdk.application.port.output.FileStoragePort;
import com.tdk.backend_tdk.domain.entity.Product;
import com.tdk.backend_tdk.domain.repository.ProductRepository;
import com.tdk.backend_tdk.domain.repository.CategoryRepository;
import com.tdk.backend_tdk.domain.valueobject.*;
import com.tdk.backend_tdk.domain.exception.CategoryNotFoundException;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class CreateProductUseCase {

    private final ProductRepository productRepository;
    private final CategoryRepository categoryRepository;

    @Transactional
    public Product execute(CreateProductCommand command) {
        // 1. Valider que la catégorie existe
        CategoryId categoryId = CategoryId.of(command.getCategoryId());
        if (!categoryRepository.existsById(categoryId)) {
            throw new CategoryNotFoundException("Category not found with id: " + categoryId.getValue());
        }

        // 2. Créer l'entité du domaine
        Product product = new Product(
            null, // ID sera généré par le repository
            command.getName(),
            command.getDescription(),
            new Money(command.getPrice(), "XAF"),
            command.getDiscountPrice() != null ? new Money(command.getDiscountPrice(), "XAF") : null,
            command.getStockQuantity(),
            categoryId,
            true
        );

        // 3. Valider et sauvegarder
        return productRepository.save(product);
    }
}
```

**`application/usecase/product/CreateProductCommand.java`**:
```java
package com.tdk.backend_tdk.application.usecase.product;

import lombok.AllArgsConstructor;
import lombok.Getter;

@Getter
@AllArgsConstructor
public class CreateProductCommand {
    private final String name;
    private final String description;
    private final Double price;
    private final Double discountPrice;
    private final Integer stockQuantity;
    private final Long categoryId;
}
```

**`application/usecase/product/GetProductUseCase.java`**:
```java
package com.tdk.backend_tdk.application.usecase.product;

import com.tdk.backend_tdk.domain.entity.Product;
import com.tdk.backend_tdk.domain.repository.ProductRepository;
import com.tdk.backend_tdk.domain.valueobject.ProductId;
import com.tdk.backend_tdk.domain.exception.ProductNotFoundException;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
@RequiredArgsConstructor
public class GetProductUseCase {

    private final ProductRepository productRepository;

    @Transactional(readOnly = true)
    public Product execute(Long productId) {
        return productRepository.findById(ProductId.of(productId))
            .orElseThrow(() -> new ProductNotFoundException("Product not found with id: " + productId));
    }
}
```

#### Étape 2.5: Implémenter le Repository (Infrastructure)

**`infrastructure/persistence/entity/ProductJpaEntity.java`**:
```java
package com.tdk.backend_tdk.infrastructure.persistence.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * Entité JPA - Utilisée uniquement pour la persistance
 */
@Entity
@Table(name = "products")
@Data
@NoArgsConstructor
@AllArgsConstructor
public class ProductJpaEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false)
    private Double price;

    private Double discountPrice;

    @Column(nullable = false)
    private Integer stockQuantity;

    @Column(name = "category_id", nullable = false)
    private Long categoryId;

    @Column(nullable = false)
    private Boolean isActive = true;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", insertable = false, updatable = false)
    private CategoryJpaEntity category;
}
```

**`infrastructure/persistence/repository/ProductJpaRepository.java`**:
```java
package com.tdk.backend_tdk.infrastructure.persistence.repository;

import com.tdk.backend_tdk.infrastructure.persistence.entity.ProductJpaEntity;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;

/**
 * Repository Spring Data JPA
 */
public interface ProductJpaRepository extends JpaRepository<ProductJpaEntity, Long> {
    Page<ProductJpaEntity> findByIsActive(Boolean isActive, Pageable pageable);
    Page<ProductJpaEntity> findByCategoryId(Long categoryId, Pageable pageable);

    @Query("SELECT p FROM ProductJpaEntity p WHERE p.isActive = true AND " +
           "(p.discountPrice IS NOT NULL AND p.discountPrice < p.price)")
    Page<ProductJpaEntity> findProductsOnSale(Pageable pageable);
}
```

**`infrastructure/persistence/repository/ProductRepositoryImpl.java`**:
```java
package com.tdk.backend_tdk.infrastructure.persistence.repository;

import com.tdk.backend_tdk.domain.entity.Product;
import com.tdk.backend_tdk.domain.repository.ProductRepository;
import com.tdk.backend_tdk.domain.valueobject.CategoryId;
import com.tdk.backend_tdk.domain.valueobject.ProductId;
import com.tdk.backend_tdk.infrastructure.persistence.entity.ProductJpaEntity;
import com.tdk.backend_tdk.infrastructure.persistence.mapper.ProductMapper;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Repository;

import java.util.Optional;

/**
 * Implémentation du port ProductRepository
 * Adapte JPA au domaine
 */
@Repository
@RequiredArgsConstructor
public class ProductRepositoryImpl implements ProductRepository {

    private final ProductJpaRepository jpaRepository;
    private final ProductMapper mapper;

    @Override
    public Product save(Product product) {
        ProductJpaEntity entity = mapper.toJpaEntity(product);
        ProductJpaEntity saved = jpaRepository.save(entity);
        return mapper.toDomain(saved);
    }

    @Override
    public Optional<Product> findById(ProductId id) {
        return jpaRepository.findById(id.getValue())
            .map(mapper::toDomain);
    }

    @Override
    public Page<Product> findAll(Pageable pageable) {
        return jpaRepository.findAll(pageable)
            .map(mapper::toDomain);
    }

    @Override
    public Page<Product> findByCategory(CategoryId categoryId, Pageable pageable) {
        return jpaRepository.findByCategoryId(categoryId.getValue(), pageable)
            .map(mapper::toDomain);
    }

    @Override
    public Page<Product> findActiveProducts(Pageable pageable) {
        return jpaRepository.findByIsActive(true, pageable)
            .map(mapper::toDomain);
    }

    @Override
    public Page<Product> findProductsOnSale(Pageable pageable) {
        return jpaRepository.findProductsOnSale(pageable)
            .map(mapper::toDomain);
    }

    @Override
    public void deleteById(ProductId id) {
        jpaRepository.deleteById(id.getValue());
    }

    @Override
    public boolean existsById(ProductId id) {
        return jpaRepository.existsById(id.getValue());
    }
}
```

**`infrastructure/persistence/mapper/ProductMapper.java`**:
```java
package com.tdk.backend_tdk.infrastructure.persistence.mapper;

import com.tdk.backend_tdk.domain.entity.Product;
import com.tdk.backend_tdk.domain.valueobject.*;
import com.tdk.backend_tdk.infrastructure.persistence.entity.ProductJpaEntity;
import org.springframework.stereotype.Component;

/**
 * Mapper bidirectionnel entre entité du domaine et entité JPA
 */
@Component
public class ProductMapper {

    public Product toDomain(ProductJpaEntity entity) {
        if (entity == null) return null;

        return new Product(
            entity.getId() != null ? ProductId.of(entity.getId()) : null,
            entity.getName(),
            entity.getDescription(),
            new Money(entity.getPrice(), "XAF"),
            entity.getDiscountPrice() != null ? new Money(entity.getDiscountPrice(), "XAF") : null,
            entity.getStockQuantity(),
            CategoryId.of(entity.getCategoryId()),
            entity.getIsActive()
        );
    }

    public ProductJpaEntity toJpaEntity(Product domain) {
        if (domain == null) return null;

        return new ProductJpaEntity(
            domain.getId() != null ? domain.getId().getValue() : null,
            domain.getName(),
            domain.getDescription(),
            domain.getPrice().getAmount(),
            domain.getDiscountPrice() != null ? domain.getDiscountPrice().getAmount() : null,
            domain.getStockQuantity(),
            domain.getCategoryId().getValue(),
            domain.isActive(),
            null // category relation
        );
    }
}
```

#### Étape 2.6: Créer les Ports pour l'infrastructure

**`application/port/output/FileStoragePort.java`**:
```java
package com.tdk.backend_tdk.application.port.output;

import org.springframework.web.multipart.MultipartFile;

/**
 * Port pour le stockage de fichiers
 * L'implémentation sera dans l'infrastructure
 */
public interface FileStoragePort {
    String storeFile(MultipartFile file, String directory);
    void deleteFile(String filePath);
    boolean fileExists(String filePath);
}
```

**`infrastructure/storage/LocalFileStorageAdapter.java`**:
```java
package com.tdk.backend_tdk.infrastructure.storage;

import com.tdk.backend_tdk.application.port.output.FileStoragePort;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;
import org.springframework.web.multipart.MultipartFile;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.util.UUID;

@Service
public class LocalFileStorageAdapter implements FileStoragePort {

    @Value("${file.upload-dir}")
    private String uploadDir;

    @Override
    public String storeFile(MultipartFile file, String directory) {
        try {
            Path uploadPath = Paths.get(uploadDir, directory);
            if (!Files.exists(uploadPath)) {
                Files.createDirectories(uploadPath);
            }

            String filename = UUID.randomUUID() + "_" + file.getOriginalFilename();
            Path filePath = uploadPath.resolve(filename);
            Files.copy(file.getInputStream(), filePath);

            return directory + "/" + filename;
        } catch (IOException e) {
            throw new RuntimeException("Failed to store file", e);
        }
    }

    @Override
    public void deleteFile(String filePath) {
        try {
            Path path = Paths.get(uploadDir, filePath);
            Files.deleteIfExists(path);
        } catch (IOException e) {
            throw new RuntimeException("Failed to delete file", e);
        }
    }

    @Override
    public boolean fileExists(String filePath) {
        Path path = Paths.get(uploadDir, filePath);
        return Files.exists(path);
    }
}
```

#### Étape 2.7: Adapter le Controller

**`presentation/rest/ProductController.java`**:
```java
package com.tdk.backend_tdk.presentation.rest;

import com.tdk.backend_tdk.application.usecase.product.*;
import com.tdk.backend_tdk.domain.entity.Product;
import com.tdk.backend_tdk.presentation.dto.request.CreateProductRequest;
import com.tdk.backend_tdk.presentation.dto.response.ProductResponse;
import com.tdk.backend_tdk.presentation.mapper.ProductRestMapper;
import jakarta.validation.Valid;
import lombok.RequiredArgsConstructor;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.prepost.PreAuthorize;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    private final CreateProductUseCase createProductUseCase;
    private final GetProductUseCase getProductUseCase;
    private final UpdateProductUseCase updateProductUseCase;
    private final DeleteProductUseCase deleteProductUseCase;
    private final GetAllProductsUseCase getAllProductsUseCase;
    private final ProductRestMapper mapper;

    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductResponse> createProduct(
            @RequestBody @Valid CreateProductRequest request) {

        CreateProductCommand command = mapper.toCommand(request);
        Product product = createProductUseCase.execute(command);
        ProductResponse response = mapper.toResponse(product);

        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable Long id) {
        Product product = getProductUseCase.execute(id);
        return ResponseEntity.ok(mapper.toResponse(product));
    }

    @GetMapping
    public ResponseEntity<Page<ProductResponse>> getAllProducts(Pageable pageable) {
        Page<Product> products = getAllProductsUseCase.execute(pageable);
        Page<ProductResponse> response = products.map(mapper::toResponse);
        return ResponseEntity.ok(response);
    }

    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductResponse> updateProduct(
            @PathVariable Long id,
            @RequestBody @Valid UpdateProductRequest request) {

        UpdateProductCommand command = mapper.toUpdateCommand(id, request);
        Product product = updateProductUseCase.execute(command);
        return ResponseEntity.ok(mapper.toResponse(product));
    }

    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        deleteProductUseCase.execute(id);
        return ResponseEntity.noContent().build();
    }
}
```

**`presentation/dto/request/CreateProductRequest.java`**:
```java
package com.tdk.backend_tdk.presentation.dto.request;

import jakarta.validation.constraints.*;
import lombok.Data;

@Data
public class CreateProductRequest {

    @NotBlank(message = "Product name is required")
    @Size(min = 3, max = 100, message = "Name must be between 3 and 100 characters")
    private String name;

    @NotBlank(message = "Description is required")
    @Size(max = 2000, message = "Description cannot exceed 2000 characters")
    private String description;

    @NotNull(message = "Price is required")
    @Positive(message = "Price must be positive")
    private Double price;

    @Positive(message = "Discount price must be positive")
    private Double discountPrice;

    @NotNull(message = "Stock quantity is required")
    @Min(value = 0, message = "Stock quantity cannot be negative")
    private Integer stockQuantity;

    @NotNull(message = "Category ID is required")
    @Positive(message = "Category ID must be positive")
    private Long categoryId;
}
```

**`presentation/dto/response/ProductResponse.java`**:
```java
package com.tdk.backend_tdk.presentation.dto.response;

import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class ProductResponse {
    private Long id;
    private String name;
    private String description;
    private Double price;
    private Double discountPrice;
    private Integer stockQuantity;
    private Long categoryId;
    private String categoryName;
    private Boolean isActive;
    private Boolean hasDiscount;
}
```

**`presentation/mapper/ProductRestMapper.java`**:
```java
package com.tdk.backend_tdk.presentation.mapper;

import com.tdk.backend_tdk.application.usecase.product.CreateProductCommand;
import com.tdk.backend_tdk.application.usecase.product.UpdateProductCommand;
import com.tdk.backend_tdk.domain.entity.Product;
import com.tdk.backend_tdk.presentation.dto.request.CreateProductRequest;
import com.tdk.backend_tdk.presentation.dto.request.UpdateProductRequest;
import com.tdk.backend_tdk.presentation.dto.response.ProductResponse;
import org.springframework.stereotype.Component;

@Component
public class ProductRestMapper {

    public CreateProductCommand toCommand(CreateProductRequest request) {
        return new CreateProductCommand(
            request.getName(),
            request.getDescription(),
            request.getPrice(),
            request.getDiscountPrice(),
            request.getStockQuantity(),
            request.getCategoryId()
        );
    }

    public UpdateProductCommand toUpdateCommand(Long productId, UpdateProductRequest request) {
        return new UpdateProductCommand(
            productId,
            request.getName(),
            request.getDescription(),
            request.getPrice(),
            request.getDiscountPrice(),
            request.getStockQuantity(),
            request.getCategoryId()
        );
    }

    public ProductResponse toResponse(Product product) {
        return new ProductResponse(
            product.getId() != null ? product.getId().getValue() : null,
            product.getName(),
            product.getDescription(),
            product.getPrice().getAmount(),
            product.getDiscountPrice() != null ? product.getDiscountPrice().getAmount() : null,
            product.getStockQuantity(),
            product.getCategoryId().getValue(),
            null, // categoryName - à récupérer si nécessaire
            product.isActive(),
            product.hasDiscount()
        );
    }
}
```

---

### Phase 3: Migration des autres modules

Une fois le module Product migré avec succès, répétez le processus pour:

1. **Order** (commandes)
   - Domain: Order, OrderItem entities
   - Use Cases: CreateOrderUseCase, CancelOrderUseCase, UpdateOrderStatusUseCase
   - Ports: PaymentPort (pour Stripe, MTN, Orange Money)

2. **User & Auth** (authentification)
   - Domain: User, Role entities
   - Use Cases: RegisterUserUseCase, LoginUseCase, ResetPasswordUseCase
   - Ports: EmailPort, TokenPort

3. **Cart** (panier)
   - Domain: Cart, CartItem entities
   - Use Cases: AddToCartUseCase, RemoveFromCartUseCase, ClearCartUseCase

4. **Category**
5. **Address**
6. **Payment**

---

### Phase 4: Migration des services transversaux

#### EmailPort et son adapter

**`application/port/output/EmailPort.java`**:
```java
package com.tdk.backend_tdk.application.port.output;

public interface EmailPort {
    void sendEmail(String to, String subject, String body);
    void sendHtmlEmail(String to, String subject, String htmlContent);
    void sendPasswordResetEmail(String to, String resetToken);
    void sendOrderConfirmationEmail(String to, String orderNumber);
}
```

**`infrastructure/email/EmailAdapter.java`**:
```java
package com.tdk.backend_tdk.infrastructure.email;

import com.tdk.backend_tdk.application.port.output.EmailPort;
import lombok.RequiredArgsConstructor;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.stereotype.Service;

import jakarta.mail.MessagingException;
import jakarta.mail.internet.MimeMessage;

@Service
@RequiredArgsConstructor
public class EmailAdapter implements EmailPort {

    private final JavaMailSender mailSender;

    @Override
    public void sendEmail(String to, String subject, String body) {
        SimpleMailMessage message = new SimpleMailMessage();
        message.setTo(to);
        message.setSubject(subject);
        message.setText(body);
        mailSender.send(message);
    }

    @Override
    public void sendHtmlEmail(String to, String subject, String htmlContent) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(htmlContent, true);
            mailSender.send(message);
        } catch (MessagingException e) {
            throw new RuntimeException("Failed to send email", e);
        }
    }

    @Override
    public void sendPasswordResetEmail(String to, String resetToken) {
        String subject = "Password Reset Request";
        String htmlContent = buildPasswordResetEmailTemplate(resetToken);
        sendHtmlEmail(to, subject, htmlContent);
    }

    @Override
    public void sendOrderConfirmationEmail(String to, String orderNumber) {
        String subject = "Order Confirmation - " + orderNumber;
        String htmlContent = buildOrderConfirmationTemplate(orderNumber);
        sendHtmlEmail(to, subject, htmlContent);
    }

    private String buildPasswordResetEmailTemplate(String resetToken) {
        return """
            <html>
            <body>
                <h2>Password Reset Request</h2>
                <p>Click the link below to reset your password:</p>
                <a href="http://localhost:4200/reset-password?token=%s">Reset Password</a>
                <p>This link will expire in 1 hour.</p>
            </body>
            </html>
            """.formatted(resetToken);
    }

    private String buildOrderConfirmationTemplate(String orderNumber) {
        return """
            <html>
            <body>
                <h2>Order Confirmation</h2>
                <p>Your order %s has been confirmed.</p>
                <p>Thank you for your purchase!</p>
            </body>
            </html>
            """.formatted(orderNumber);
    }
}
```

#### PaymentPort et ses adapters

**`application/port/output/PaymentPort.java`**:
```java
package com.tdk.backend_tdk.application.port.output;

public interface PaymentPort {
    PaymentResult processPayment(PaymentRequest request);
    PaymentStatus checkPaymentStatus(String transactionId);
    void refundPayment(String transactionId, double amount);
}

// DTOs pour le port
public record PaymentRequest(
    double amount,
    String currency,
    String customerEmail,
    String orderId,
    PaymentMethod method
) {}

public record PaymentResult(
    String transactionId,
    PaymentStatus status,
    String message
) {}

public enum PaymentStatus {
    SUCCESS, PENDING, FAILED, REFUNDED
}

public enum PaymentMethod {
    STRIPE, MTN_MONEY, ORANGE_MONEY
}
```

**`infrastructure/payment/StripePaymentAdapter.java`**:
```java
package com.tdk.backend_tdk.infrastructure.payment;

import com.stripe.Stripe;
import com.stripe.exception.StripeException;
import com.stripe.model.PaymentIntent;
import com.stripe.param.PaymentIntentCreateParams;
import com.tdk.backend_tdk.application.port.output.*;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

@Service
public class StripePaymentAdapter implements PaymentPort {

    public StripePaymentAdapter(@Value("${stripe.api.key}") String apiKey) {
        Stripe.apiKey = apiKey;
    }

    @Override
    public PaymentResult processPayment(PaymentRequest request) {
        try {
            PaymentIntentCreateParams params = PaymentIntentCreateParams.builder()
                .setAmount((long) (request.amount() * 100)) // Convert to cents
                .setCurrency(request.currency())
                .putMetadata("orderId", request.orderId())
                .putMetadata("customerEmail", request.customerEmail())
                .build();

            PaymentIntent intent = PaymentIntent.create(params);

            return new PaymentResult(
                intent.getId(),
                mapStripeStatus(intent.getStatus()),
                "Payment processed successfully"
            );
        } catch (StripeException e) {
            return new PaymentResult(
                null,
                PaymentStatus.FAILED,
                "Payment failed: " + e.getMessage()
            );
        }
    }

    @Override
    public PaymentStatus checkPaymentStatus(String transactionId) {
        try {
            PaymentIntent intent = PaymentIntent.retrieve(transactionId);
            return mapStripeStatus(intent.getStatus());
        } catch (StripeException e) {
            throw new RuntimeException("Failed to check payment status", e);
        }
    }

    @Override
    public void refundPayment(String transactionId, double amount) {
        // Implémentation du remboursement Stripe
    }

    private PaymentStatus mapStripeStatus(String stripeStatus) {
        return switch (stripeStatus) {
            case "succeeded" -> PaymentStatus.SUCCESS;
            case "processing", "requires_payment_method" -> PaymentStatus.PENDING;
            default -> PaymentStatus.FAILED;
        };
    }
}
```

---

### Phase 5: Configuration Spring

**`config/UseCaseConfig.java`**:
```java
package com.tdk.backend_tdk.config;

import org.springframework.context.annotation.ComponentScan;
import org.springframework.context.annotation.Configuration;

@Configuration
@ComponentScan(basePackages = {
    "com.tdk.backend_tdk.application.usecase",
    "com.tdk.backend_tdk.domain.service"
})
public class UseCaseConfig {
    // Les use cases seront automatiquement détectés comme des @Service
}
```

**`config/PersistenceConfig.java`**:
```java
package com.tdk.backend_tdk.config;

import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;

@Configuration
@EnableJpaRepositories(basePackages = "com.tdk.backend_tdk.infrastructure.persistence.repository")
@EntityScan(basePackages = "com.tdk.backend_tdk.infrastructure.persistence.entity")
public class PersistenceConfig {
}
```

---

## Exemples de transformation

### Exemple complet: Use Case CreateOrder

**`application/usecase/order/CreateOrderUseCase.java`**:
```java
package com.tdk.backend_tdk.application.usecase.order;

import com.tdk.backend_tdk.application.port.output.EmailPort;
import com.tdk.backend_tdk.application.port.output.PaymentPort;
import com.tdk.backend_tdk.domain.entity.*;
import com.tdk.backend_tdk.domain.repository.*;
import com.tdk.backend_tdk.domain.service.OrderValidationService;
import com.tdk.backend_tdk.domain.valueobject.*;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.List;
import java.util.stream.Collectors;

@Service
@RequiredArgsConstructor
public class CreateOrderUseCase {

    private final OrderRepository orderRepository;
    private final CartRepository cartRepository;
    private final ProductRepository productRepository;
    private final UserRepository userRepository;
    private final AddressRepository addressRepository;
    private final PaymentPort paymentPort;
    private final EmailPort emailPort;
    private final OrderValidationService orderValidationService;

    @Transactional
    public Order execute(CreateOrderCommand command) {
        // 1. Récupérer l'utilisateur
        User user = userRepository.findByEmail(command.getUserEmail())
            .orElseThrow(() -> new UserNotFoundException("User not found"));

        // 2. Récupérer le panier
        Cart cart = cartRepository.findByUserId(user.getId())
            .orElseThrow(() -> new CartNotFoundException("Cart not found"));

        if (cart.isEmpty()) {
            throw new EmptyCartException("Cannot create order from empty cart");
        }

        // 3. Récupérer l'adresse de livraison
        Address shippingAddress = addressRepository.findById(AddressId.of(command.getShippingAddressId()))
            .orElseThrow(() -> new AddressNotFoundException("Shipping address not found"));

        // 4. Créer les items de commande à partir du panier
        List<OrderItem> orderItems = cart.getItems().stream()
            .map(cartItem -> {
                Product product = productRepository.findById(cartItem.getProductId())
                    .orElseThrow(() -> new ProductNotFoundException("Product not found"));

                // Vérifier le stock
                if (!product.isInStock() || product.getStockQuantity() < cartItem.getQuantity()) {
                    throw new InsufficientStockException(
                        "Insufficient stock for product: " + product.getName()
                    );
                }

                return new OrderItem(
                    null,
                    product.getId(),
                    product.getName(),
                    cartItem.getQuantity(),
                    product.getPrice(),
                    product.getDiscountPrice()
                );
            })
            .collect(Collectors.toList());

        // 5. Créer la commande
        Order order = new Order(
            null,
            user.getId(),
            orderItems,
            shippingAddress,
            command.getBillingAddressId() != null
                ? addressRepository.findById(AddressId.of(command.getBillingAddressId())).orElse(null)
                : shippingAddress,
            OrderStatus.PENDING
        );

        // 6. Valider la commande (logique métier du domaine)
        orderValidationService.validate(order);

        // 7. Traiter le paiement
        PaymentRequest paymentRequest = new PaymentRequest(
            order.getTotalAmount().getAmount(),
            "XAF",
            user.getEmail().getValue(),
            order.getId() != null ? order.getId().getValue().toString() : "pending",
            command.getPaymentMethod()
        );

        PaymentResult paymentResult = paymentPort.processPayment(paymentRequest);

        if (paymentResult.status() != PaymentStatus.SUCCESS) {
            throw new PaymentFailedException("Payment failed: " + paymentResult.message());
        }

        order.markAsPaid(paymentResult.transactionId());

        // 8. Réduire le stock des produits
        orderItems.forEach(item -> {
            Product product = productRepository.findById(item.getProductId())
                .orElseThrow();
            product.decreaseStock(item.getQuantity());
            productRepository.save(product);
        });

        // 9. Sauvegarder la commande
        Order savedOrder = orderRepository.save(order);

        // 10. Vider le panier
        cart.clear();
        cartRepository.save(cart);

        // 11. Envoyer email de confirmation
        emailPort.sendOrderConfirmationEmail(
            user.getEmail().getValue(),
            savedOrder.getOrderNumber()
        );

        return savedOrder;
    }
}
```

**`domain/service/OrderValidationService.java`**:
```java
package com.tdk.backend_tdk.domain.service;

import com.tdk.backend_tdk.domain.entity.Order;
import com.tdk.backend_tdk.domain.exception.InvalidOrderException;
import org.springframework.stereotype.Service;

/**
 * Service du domaine - Contient la logique métier complexe
 */
@Service
public class OrderValidationService {

    private static final double MIN_ORDER_AMOUNT = 1000.0; // 1000 XAF
    private static final double MAX_ORDER_AMOUNT = 10000000.0; // 10M XAF

    public void validate(Order order) {
        validateOrderAmount(order);
        validateOrderItems(order);
        validateAddresses(order);
    }

    private void validateOrderAmount(Order order) {
        double totalAmount = order.getTotalAmount().getAmount();

        if (totalAmount < MIN_ORDER_AMOUNT) {
            throw new InvalidOrderException(
                "Order amount must be at least " + MIN_ORDER_AMOUNT + " XAF"
            );
        }

        if (totalAmount > MAX_ORDER_AMOUNT) {
            throw new InvalidOrderException(
                "Order amount cannot exceed " + MAX_ORDER_AMOUNT + " XAF"
            );
        }
    }

    private void validateOrderItems(Order order) {
        if (order.getItems().isEmpty()) {
            throw new InvalidOrderException("Order must contain at least one item");
        }

        // Valider que tous les items ont des quantités positives
        order.getItems().forEach(item -> {
            if (item.getQuantity() <= 0) {
                throw new InvalidOrderException("Order item quantity must be positive");
            }
        });
    }

    private void validateAddresses(Order order) {
        if (order.getShippingAddress() == null) {
            throw new InvalidOrderException("Shipping address is required");
        }

        if (order.getBillingAddress() == null) {
            throw new InvalidOrderException("Billing address is required");
        }
    }
}
```

---

## Bonnes pratiques

### 1. Principes SOLID

- **Single Responsibility**: Chaque use case fait une seule chose
- **Open/Closed**: Extensions via nouveaux use cases, pas modification
- **Liskov Substitution**: Les implémentations des ports sont interchangeables
- **Interface Segregation**: Ports spécifiques et ciblés
- **Dependency Inversion**: Dépendances vers abstractions (ports)

### 2. Tests

```java
// Test unitaire du domaine (aucune dépendance externe)
class ProductTest {
    @Test
    void shouldApplyDiscount() {
        Money price = new Money(10000, "XAF");
        Money discount = new Money(8000, "XAF");

        Product product = new Product(
            ProductId.of(1L),
            "Test Product",
            "Description",
            price,
            null,
            10,
            CategoryId.of(1L),
            true
        );

        product.applyDiscount(discount);

        assertEquals(discount, product.getDiscountPrice());
        assertTrue(product.hasDiscount());
    }

    @Test
    void shouldThrowExceptionWhenDiscountExceedsPrice() {
        Money price = new Money(10000, "XAF");
        Money discount = new Money(15000, "XAF");

        Product product = new Product(
            ProductId.of(1L),
            "Test Product",
            "Description",
            price,
            null,
            10,
            CategoryId.of(1L),
            true
        );

        assertThrows(IllegalArgumentException.class, () -> {
            product.applyDiscount(discount);
        });
    }
}

// Test du use case (avec mocks)
@ExtendWith(MockitoExtension.class)
class CreateProductUseCaseTest {

    @Mock
    private ProductRepository productRepository;

    @Mock
    private CategoryRepository categoryRepository;

    @InjectMocks
    private CreateProductUseCase useCase;

    @Test
    void shouldCreateProduct() {
        // Given
        CreateProductCommand command = new CreateProductCommand(
            "Test Product",
            "Description",
            10000.0,
            null,
            10,
            1L
        );

        when(categoryRepository.existsById(any())).thenReturn(true);
        when(productRepository.save(any())).thenAnswer(i -> i.getArgument(0));

        // When
        Product result = useCase.execute(command);

        // Then
        assertNotNull(result);
        assertEquals("Test Product", result.getName());
        verify(productRepository).save(any());
    }

    @Test
    void shouldThrowExceptionWhenCategoryNotFound() {
        // Given
        CreateProductCommand command = new CreateProductCommand(
            "Test Product",
            "Description",
            10000.0,
            null,
            10,
            999L
        );

        when(categoryRepository.existsById(any())).thenReturn(false);

        // When & Then
        assertThrows(CategoryNotFoundException.class, () -> {
            useCase.execute(command);
        });

        verify(productRepository, never()).save(any());
    }
}
```

### 3. Documentation

- Documenter chaque couche avec des commentaires JavaDoc
- Expliquer les choix architecturaux dans les README
- Maintenir des diagrammes à jour (architecture, flux)

### 4. Migration progressive

- **Ne pas tout refactorer d'un coup**
- Commencer par un module simple (Product)
- Tester en profondeur avant de continuer
- Garder l'ancien code en parallèle pendant la transition
- Utiliser des feature flags si nécessaire

### 5. Éviter les pièges

❌ **À éviter**:
- Logique métier dans les controllers
- Annotations JPA dans les entités du domaine
- Dépendances du domaine vers l'infrastructure
- Use cases qui dépendent d'autres use cases directement

✅ **À faire**:
- Garder le domaine pur (aucune annotation framework)
- Utiliser des value objects pour la validation
- Encapsuler la logique métier complexe dans les services du domaine
- Tester le domaine sans Spring

---

## Résumé

### Avantages de Clean Architecture

1. **Testabilité**: Domain et use cases testables sans frameworks
2. **Maintenabilité**: Séparation claire des responsabilités
3. **Flexibilité**: Changement facile de frameworks/technologies
4. **Scalabilité**: Structure qui supporte la croissance du projet
5. **Compréhension**: Architecture explicite et documentée

### Ordre de migration recommandé

1. ✅ Product (Simple, peu de dépendances)
2. ✅ Category (Très simple)
3. ✅ User & Auth (Important, modérément complexe)
4. ✅ Cart (Dépend de Product et User)
5. ✅ Order (Complexe, dépend de tout)
6. ✅ Payment (Intégrations externes)
7. ✅ Statistics (Peut rester en dernier)

### Temps estimé

- Module simple (Category): 2-4 heures
- Module moyen (Product, User): 1-2 jours
- Module complexe (Order): 2-3 jours
- **Total pour migration complète**: 2-3 semaines

---

## Ressources

- [Clean Architecture - Robert C. Martin](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
- [Hexagonal Architecture](https://alistair.cockburn.us/hexagonal-architecture/)
- [DDD - Domain-Driven Design](https://www.domainlanguage.com/ddd/)
- [Spring Boot + Clean Architecture](https://medium.com/codex/clean-architecture-for-dummies-df6561d42c94)

---

Bonne chance avec la migration ! N'hésitez pas à procéder étape par étape et à tester régulièrement.
