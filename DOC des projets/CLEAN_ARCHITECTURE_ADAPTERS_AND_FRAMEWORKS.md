# Interface Adapters & Frameworks & Drivers

Guide complet des couches externes de Clean Architecture pour votre projet TKD.

---

## Table des matières

1. [Vue d'ensemble](#vue-densemble)
2. [Couche 3: Interface Adapters](#couche-3-interface-adapters)
3. [Couche 4: Frameworks & Drivers](#couche-4-frameworks--drivers)
4. [Exemples complets](#exemples-complets)

---

## Vue d'ensemble

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  COUCHE 4: FRAMEWORKS & DRIVERS                            │
│  • Spring Boot, JPA, MySQL, Spring Security                │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │                                                         │ │
│  │  COUCHE 3: INTERFACE ADAPTERS                          │ │
│  │                                                         │ │
│  │  ┌─────────────────────┐  ┌─────────────────────────┐ │ │
│  │  │   PRESENTATION      │  │   INFRASTRUCTURE        │ │ │
│  │  │                     │  │                         │ │ │
│  │  │  • Controllers      │  │  • Repositories (Impl)  │ │ │
│  │  │  • DTOs             │  │  • JPA Entities         │ │ │
│  │  │  • Mappers          │  │  • Email Adapter        │ │ │
│  │  │  • Exception        │  │  • Payment Adapter      │ │ │
│  │  │    Handlers         │  │  • File Storage         │ │ │
│  │  │                     │  │  • Security (JWT)       │ │ │
│  │  └─────────────────────┘  └─────────────────────────┘ │ │
│  │                                                         │ │
│  │         Dépend de ↓                                    │ │
│  │                                                         │ │
│  │  ┌───────────────────────────────────────────────────┐ │ │
│  │  │  COUCHE 2: USE CASES                              │ │ │
│  │  │  COUCHE 1: ENTITIES                               │ │ │
│  │  └───────────────────────────────────────────────────┘ │ │
│  │                                                         │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

## Couche 3: Interface Adapters

Cette couche contient les **adaptateurs** qui convertissent les données entre le format des use cases et le format du monde extérieur.

### 3.1 Presentation Layer (REST API)

#### A. Controllers

Les controllers gèrent les requêtes HTTP et délèguent au use cases.

**Structure des packages:**
```
presentation/
├── rest/
│   ├── ProductController.java
│   ├── OrderController.java
│   ├── AuthController.java
│   ├── CartController.java
│   └── UserController.java
├── dto/
│   ├── request/
│   └── response/
├── mapper/
│   └── ProductRestMapper.java
└── exception/
    └── GlobalExceptionHandler.java
```

---

#### ProductController - Exemple Complet

**Localisation**: `presentation/rest/ProductController.java`

```java
package com.tdk.backend_tdk.presentation.rest;

import com.tdk.backend_tdk.application.usecase.product.*;
import com.tdk.backend_tdk.domain.entity.Product;
import com.tdk.backend_tdk.presentation.dto.request.CreateProductRequest;
import com.tdk.backend_tdk.presentation.dto.request.UpdateProductRequest;
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
import org.springframework.web.multipart.MultipartFile;

/**
 * REST Controller pour la gestion des produits.
 *
 * Responsabilités:
 * - Recevoir les requêtes HTTP
 * - Valider les inputs
 * - Convertir Request → Command
 * - Appeler les Use Cases
 * - Convertir Domain → Response
 * - Retourner les réponses HTTP
 */
@RestController
@RequestMapping("/api/products")
@RequiredArgsConstructor
public class ProductController {

    // Injection des Use Cases (pas de service monolithique)
    private final CreateProductUseCase createProductUseCase;
    private final UpdateProductUseCase updateProductUseCase;
    private final GetProductUseCase getProductUseCase;
    private final GetAllProductsUseCase getAllProductsUseCase;
    private final GetActiveProductsUseCase getActiveProductsUseCase;
    private final GetProductsByCategoryUseCase getProductsByCategoryUseCase;
    private final SearchProductsUseCase searchProductsUseCase;
    private final GetNewProductsUseCase getNewProductsUseCase;
    private final GetProductsOnSaleUseCase getProductsOnSaleUseCase;
    private final DeleteProductUseCase deleteProductUseCase;
    private final AddProductImageUseCase addProductImageUseCase;
    private final DeleteProductImageUseCase deleteProductImageUseCase;

    // Mapper pour conversions
    private final ProductRestMapper mapper;

    /**
     * Créer un nouveau produit (ADMIN uniquement)
     * POST /api/products
     */
    @PostMapping
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductResponse> createProduct(
            @RequestBody @Valid CreateProductRequest request) {

        // 1. Convertir Request → Command
        CreateProductCommand command = mapper.toCommand(request);

        // 2. Exécuter le Use Case
        Product product = createProductUseCase.execute(command);

        // 3. Convertir Domain → Response
        ProductResponse response = mapper.toResponse(product);

        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }

    /**
     * Obtenir un produit par ID
     * GET /api/products/{id}
     */
    @GetMapping("/{id}")
    public ResponseEntity<ProductResponse> getProduct(@PathVariable Long id) {
        Product product = getProductUseCase.execute(id);
        return ResponseEntity.ok(mapper.toResponse(product));
    }

    /**
     * Obtenir tous les produits avec pagination
     * GET /api/products?page=0&size=20
     */
    @GetMapping
    public ResponseEntity<Page<ProductResponse>> getAllProducts(Pageable pageable) {
        Page<Product> products = getAllProductsUseCase.execute(pageable);
        Page<ProductResponse> response = products.map(mapper::toResponse);
        return ResponseEntity.ok(response);
    }

    /**
     * Obtenir les produits actifs
     * GET /api/products/active
     */
    @GetMapping("/active")
    public ResponseEntity<Page<ProductResponse>> getActiveProducts(Pageable pageable) {
        Page<Product> products = getActiveProductsUseCase.execute(pageable);
        Page<ProductResponse> response = products.map(mapper::toResponse);
        return ResponseEntity.ok(response);
    }

    /**
     * Rechercher des produits
     * GET /api/products/search?keyword=shirt
     */
    @GetMapping("/search")
    public ResponseEntity<Page<ProductResponse>> searchProducts(
            @RequestParam String keyword,
            Pageable pageable) {

        SearchProductsQuery query = new SearchProductsQuery(keyword, pageable);
        Page<Product> products = searchProductsUseCase.execute(query);
        Page<ProductResponse> response = products.map(mapper::toResponse);
        return ResponseEntity.ok(response);
    }

    /**
     * Obtenir les produits d'une catégorie
     * GET /api/products/category/1
     */
    @GetMapping("/category/{categoryId}")
    public ResponseEntity<Page<ProductResponse>> getProductsByCategory(
            @PathVariable Long categoryId,
            Pageable pageable) {

        GetProductsByCategoryQuery query = new GetProductsByCategoryQuery(categoryId, pageable);
        Page<Product> products = getProductsByCategoryUseCase.execute(query);
        Page<ProductResponse> response = products.map(mapper::toResponse);
        return ResponseEntity.ok(response);
    }

    /**
     * Obtenir les nouveaux produits
     * GET /api/products/new
     */
    @GetMapping("/new")
    public ResponseEntity<List<ProductResponse>> getNewProducts() {
        List<Product> products = getNewProductsUseCase.execute();
        List<ProductResponse> response = products.stream()
                .map(mapper::toResponse)
                .collect(Collectors.toList());
        return ResponseEntity.ok(response);
    }

    /**
     * Obtenir les produits en promotion
     * GET /api/products/sale
     */
    @GetMapping("/sale")
    public ResponseEntity<Page<ProductResponse>> getProductsOnSale(Pageable pageable) {
        Page<Product> products = getProductsOnSaleUseCase.execute(pageable);
        Page<ProductResponse> response = products.map(mapper::toResponse);
        return ResponseEntity.ok(response);
    }

    /**
     * Mettre à jour un produit (ADMIN uniquement)
     * PUT /api/products/{id}
     */
    @PutMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductResponse> updateProduct(
            @PathVariable Long id,
            @RequestBody @Valid UpdateProductRequest request) {

        UpdateProductCommand command = mapper.toUpdateCommand(id, request);
        Product product = updateProductUseCase.execute(command);
        return ResponseEntity.ok(mapper.toResponse(product));
    }

    /**
     * Supprimer un produit (ADMIN uniquement)
     * DELETE /api/products/{id}
     */
    @DeleteMapping("/{id}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteProduct(@PathVariable Long id) {
        deleteProductUseCase.execute(id);
        return ResponseEntity.noContent().build();
    }

    /**
     * Ajouter une image à un produit (ADMIN uniquement)
     * POST /api/products/{id}/images
     */
    @PostMapping("/{id}/images")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<ProductImageResponse> addProductImage(
            @PathVariable Long id,
            @RequestParam("file") MultipartFile file,
            @RequestParam(value = "isPrimary", defaultValue = "false") Boolean isPrimary) {

        AddProductImageCommand command = new AddProductImageCommand(id, file, isPrimary);
        ProductImage image = addProductImageUseCase.execute(command);
        return ResponseEntity.status(HttpStatus.CREATED)
                .body(mapper.toImageResponse(image));
    }

    /**
     * Supprimer une image (ADMIN uniquement)
     * DELETE /api/products/images/{imageId}
     */
    @DeleteMapping("/images/{imageId}")
    @PreAuthorize("hasRole('ADMIN')")
    public ResponseEntity<Void> deleteProductImage(@PathVariable Long imageId) {
        deleteProductImageUseCase.execute(imageId);
        return ResponseEntity.noContent().build();
    }
}
```

---

#### B. Request DTOs

**Localisation**: `presentation/dto/request/CreateProductRequest.java`

```java
package com.tdk.backend_tdk.presentation.dto.request;

import jakarta.validation.constraints.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

/**
 * DTO pour la création d'un produit.
 * Utilisé pour recevoir les données du client (Frontend).
 *
 * Responsabilités:
 * - Définir le contrat de l'API REST
 * - Validation des inputs (annotations Jakarta)
 * - Sérialisation JSON
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
public class CreateProductRequest {

    @NotBlank(message = "Le nom du produit est requis")
    @Size(min = 3, max = 100, message = "Le nom doit contenir entre 3 et 100 caractères")
    private String name;

    @NotBlank(message = "La description est requise")
    @Size(max = 2000, message = "La description ne peut pas dépasser 2000 caractères")
    private String description;

    @NotNull(message = "Le prix est requis")
    @Positive(message = "Le prix doit être positif")
    @DecimalMin(value = "0.01", message = "Le prix minimum est 0.01")
    private Double price;

    @Positive(message = "Le prix réduit doit être positif")
    private Double discountPrice;

    @NotNull(message = "La quantité en stock est requise")
    @Min(value = 0, message = "La quantité ne peut pas être négative")
    private Integer stockQuantity;

    @NotNull(message = "L'ID de la catégorie est requis")
    @Positive(message = "L'ID de la catégorie doit être positif")
    private Long categoryId;

    @Size(max = 50, message = "La marque ne peut pas dépasser 50 caractères")
    private String brand;

    private Boolean isActive = true;
}
```

**Localisation**: `presentation/dto/request/UpdateProductRequest.java`

```java
package com.tdk.backend_tdk.presentation.dto.request;

import jakarta.validation.constraints.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class UpdateProductRequest {

    @NotBlank(message = "Le nom du produit est requis")
    @Size(min = 3, max = 100)
    private String name;

    @NotBlank(message = "La description est requise")
    @Size(max = 2000)
    private String description;

    @NotNull(message = "Le prix est requis")
    @Positive(message = "Le prix doit être positif")
    private Double price;

    @Positive(message = "Le prix réduit doit être positif")
    private Double discountPrice;

    @NotNull(message = "La quantité en stock est requise")
    @Min(value = 0)
    private Integer stockQuantity;

    @NotNull(message = "L'ID de la catégorie est requis")
    @Positive
    private Long categoryId;

    @Size(max = 50)
    private String brand;

    private Boolean isActive;
}
```

---

#### C. Response DTOs

**Localisation**: `presentation/dto/response/ProductResponse.java`

```java
package com.tdk.backend_tdk.presentation.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;
import java.util.List;

/**
 * DTO pour la réponse d'un produit.
 * Envoyé au client (Frontend).
 *
 * Responsabilités:
 * - Définir le format de réponse de l'API
 * - Cacher les détails internes (pas d'entités JPA)
 * - Sérialisation JSON
 */
@Data
@Builder
@NoArgsConstructor
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
    private String brand;
    private Boolean isActive;

    // Champs calculés
    private Boolean inStock;
    private Boolean hasDiscount;
    private Double effectivePrice;
    private Double discountPercentage;

    // Images
    private List<String> imageUrls;
    private String primaryImageUrl;

    // Métadonnées
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;
}
```

**Localisation**: `presentation/dto/response/ProductImageResponse.java`

```java
package com.tdk.backend_tdk.presentation.dto.response;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ProductImageResponse {
    private Long id;
    private String imageUrl;
    private Boolean isPrimary;
    private Integer displayOrder;
}
```

---

#### D. Mappers (Presentation)

**Localisation**: `presentation/mapper/ProductRestMapper.java`

```java
package com.tdk.backend_tdk.presentation.mapper;

import com.tdk.backend_tdk.application.usecase.product.CreateProductCommand;
import com.tdk.backend_tdk.application.usecase.product.UpdateProductCommand;
import com.tdk.backend_tdk.domain.entity.Product;
import com.tdk.backend_tdk.domain.entity.ProductImage;
import com.tdk.backend_tdk.presentation.dto.request.CreateProductRequest;
import com.tdk.backend_tdk.presentation.dto.request.UpdateProductRequest;
import com.tdk.backend_tdk.presentation.dto.response.ProductResponse;
import com.tdk.backend_tdk.presentation.dto.response.ProductImageResponse;
import org.springframework.stereotype.Component;

import java.time.ZoneId;
import java.util.stream.Collectors;

/**
 * Mapper pour convertir entre les DTOs REST et le Domain.
 *
 * Responsabilités:
 * - Request → Command (Application)
 * - Domain → Response (Presentation)
 */
@Component
public class ProductRestMapper {

    /**
     * Convertir CreateProductRequest → CreateProductCommand
     */
    public CreateProductCommand toCommand(CreateProductRequest request) {
        return new CreateProductCommand(
            request.getName(),
            request.getDescription(),
            request.getPrice(),
            request.getDiscountPrice(),
            request.getStockQuantity(),
            request.getCategoryId(),
            request.getBrand()
        );
    }

    /**
     * Convertir UpdateProductRequest → UpdateProductCommand
     */
    public UpdateProductCommand toUpdateCommand(Long productId, UpdateProductRequest request) {
        return new UpdateProductCommand(
            productId,
            request.getName(),
            request.getDescription(),
            request.getPrice(),
            request.getDiscountPrice(),
            request.getStockQuantity(),
            request.getCategoryId(),
            request.getBrand(),
            request.getIsActive()
        );
    }

    /**
     * Convertir Product (Domain) → ProductResponse
     */
    public ProductResponse toResponse(Product product) {
        return ProductResponse.builder()
            .id(product.getId() != null ? product.getId().getValue() : null)
            .name(product.getName())
            .description(product.getDescription())
            .price(product.getPrice().getAmount())
            .discountPrice(product.getDiscountPrice() != null ?
                product.getDiscountPrice().getAmount() : null)
            .stockQuantity(product.getStockQuantity())
            .categoryId(product.getCategoryId().getValue())
            .categoryName(null) // À enrichir si nécessaire
            .brand(product.getBrand())
            .isActive(product.isActive())

            // Champs calculés
            .inStock(product.isInStock())
            .hasDiscount(product.hasDiscount())
            .effectivePrice(product.getEffectivePrice().getAmount())
            .discountPercentage(calculateDiscountPercentage(product))

            // Images
            .imageUrls(product.getImages().stream()
                .map(ProductImage::getImageUrl)
                .collect(Collectors.toList()))
            .primaryImageUrl(product.getImages().stream()
                .filter(ProductImage::isPrimary)
                .findFirst()
                .map(ProductImage::getImageUrl)
                .orElse(null))

            // Métadonnées
            .createdAt(product.getCreatedAt() != null ?
                product.getCreatedAt().toInstant()
                    .atZone(ZoneId.systemDefault())
                    .toLocalDateTime() : null)
            .updatedAt(product.getUpdatedAt() != null ?
                product.getUpdatedAt().toInstant()
                    .atZone(ZoneId.systemDefault())
                    .toLocalDateTime() : null)
            .build();
    }

    /**
     * Convertir ProductImage → ProductImageResponse
     */
    public ProductImageResponse toImageResponse(ProductImage image) {
        return ProductImageResponse.builder()
            .id(image.getId() != null ? image.getId().getValue() : null)
            .imageUrl(image.getImageUrl())
            .isPrimary(image.isPrimary())
            .displayOrder(image.getDisplayOrder())
            .build();
    }

    /**
     * Calculer le pourcentage de réduction
     */
    private Double calculateDiscountPercentage(Product product) {
        if (!product.hasDiscount()) {
            return 0.0;
        }
        double originalPrice = product.getPrice().getAmount();
        double discountPrice = product.getDiscountPrice().getAmount();
        return ((originalPrice - discountPrice) / originalPrice) * 100;
    }
}
```

---

#### E. Exception Handler

**Localisation**: `presentation/exception/GlobalExceptionHandler.java`

```java
package com.tdk.backend_tdk.presentation.exception;

import com.tdk.backend_tdk.domain.exception.*;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.security.access.AccessDeniedException;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;
import org.springframework.web.context.request.WebRequest;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

/**
 * Gestion globale des exceptions pour l'API REST.
 *
 * Responsabilités:
 * - Intercepter les exceptions
 * - Convertir en réponses HTTP appropriées
 * - Formater les messages d'erreur
 */
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * Exception: Ressource non trouvée
     */
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleResourceNotFound(
            ResourceNotFoundException ex, WebRequest request) {

        ErrorResponse error = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.NOT_FOUND.value())
            .error("Not Found")
            .message(ex.getMessage())
            .path(request.getDescription(false).replace("uri=", ""))
            .build();

        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    /**
     * Exception: Validation des inputs
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ValidationErrorResponse> handleValidationErrors(
            MethodArgumentNotValidException ex, WebRequest request) {

        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach((error) -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });

        ValidationErrorResponse errorResponse = ValidationErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Validation Failed")
            .message("Les données fournies sont invalides")
            .path(request.getDescription(false).replace("uri=", ""))
            .validationErrors(errors)
            .build();

        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errorResponse);
    }

    /**
     * Exception: Accès refusé
     */
    @ExceptionHandler(AccessDeniedException.class)
    public ResponseEntity<ErrorResponse> handleAccessDenied(
            AccessDeniedException ex, WebRequest request) {

        ErrorResponse error = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.FORBIDDEN.value())
            .error("Forbidden")
            .message("Vous n'avez pas les permissions nécessaires")
            .path(request.getDescription(false).replace("uri=", ""))
            .build();

        return ResponseEntity.status(HttpStatus.FORBIDDEN).body(error);
    }

    /**
     * Exception: Logique métier (BadRequest)
     */
    @ExceptionHandler({
        InvalidDiscountException.class,
        InsufficientStockException.class,
        EmptyCartException.class
    })
    public ResponseEntity<ErrorResponse> handleBusinessLogicException(
            RuntimeException ex, WebRequest request) {

        ErrorResponse error = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.BAD_REQUEST.value())
            .error("Business Logic Error")
            .message(ex.getMessage())
            .path(request.getDescription(false).replace("uri=", ""))
            .build();

        return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(error);
    }

    /**
     * Exception générique
     */
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGlobalException(
            Exception ex, WebRequest request) {

        ErrorResponse error = ErrorResponse.builder()
            .timestamp(LocalDateTime.now())
            .status(HttpStatus.INTERNAL_SERVER_ERROR.value())
            .error("Internal Server Error")
            .message("Une erreur inattendue s'est produite")
            .path(request.getDescription(false).replace("uri=", ""))
            .build();

        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}
```

**DTOs pour les erreurs:**

```java
package com.tdk.backend_tdk.presentation.exception;

import lombok.AllArgsConstructor;
import lombok.Builder;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.time.LocalDateTime;

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
}

@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class ValidationErrorResponse {
    private LocalDateTime timestamp;
    private int status;
    private String error;
    private String message;
    private String path;
    private Map<String, String> validationErrors;
}
```

---

### 3.2 Infrastructure Layer

#### A. Repositories (Implémentations)

**Structure:**
```
infrastructure/
├── persistence/
│   ├── entity/
│   │   ├── ProductJpaEntity.java
│   │   ├── OrderJpaEntity.java
│   │   └── UserJpaEntity.java
│   ├── repository/
│   │   ├── ProductJpaRepository.java
│   │   └── ProductRepositoryImpl.java
│   └── mapper/
│       └── ProductMapper.java
├── email/
│   └── EmailAdapter.java
├── payment/
│   ├── StripePaymentAdapter.java
│   ├── MtnMoneyAdapter.java
│   └── OrangeMoneyAdapter.java
├── storage/
│   └── LocalFileStorageAdapter.java
└── security/
    ├── JwtTokenProvider.java
    └── JwtAuthenticationFilter.java
```

---

#### JPA Entity

**Localisation**: `infrastructure/persistence/entity/ProductJpaEntity.java`

```java
package com.tdk.backend_tdk.infrastructure.persistence.entity;

import jakarta.persistence.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.Date;
import java.util.List;

/**
 * Entité JPA pour la persistance des produits.
 *
 * IMPORTANT: Cette classe est UNIQUEMENT pour la base de données.
 * Elle ne contient AUCUNE logique métier.
 *
 * Responsabilités:
 * - Mapping avec la table MySQL
 * - Gestion des relations JPA
 * - Aucune logique métier
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

    @Column(nullable = false, length = 100)
    private String name;

    @Column(columnDefinition = "TEXT")
    private String description;

    @Column(nullable = false, precision = 10, scale = 2)
    private BigDecimal price;

    @Column(name = "discount_price", precision = 10, scale = 2)
    private BigDecimal discountPrice;

    @Column(name = "stock_quantity", nullable = false)
    private Integer stockQuantity = 0;

    @Column(name = "category_id", nullable = false)
    private Long categoryId;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "category_id", insertable = false, updatable = false)
    private CategoryJpaEntity category;

    @Column(length = 50)
    private String brand;

    @Column(name = "is_active", nullable = false)
    private Boolean isActive = true;

    @OneToMany(mappedBy = "product", cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    private List<ProductImageJpaEntity> images = new ArrayList<>();

    @Column(name = "created_at", nullable = false, updatable = false)
    @Temporal(TemporalType.TIMESTAMP)
    private Date createdAt = new Date();

    @Column(name = "updated_at")
    @Temporal(TemporalType.TIMESTAMP)
    private Date updatedAt = new Date();

    @PreUpdate
    public void setLastUpdate() {
        this.updatedAt = new Date();
    }
}
```

---

#### JPA Repository Interface

**Localisation**: `infrastructure/persistence/repository/ProductJpaRepository.java`

```java
package com.tdk.backend_tdk.infrastructure.persistence.repository;

import com.tdk.backend_tdk.infrastructure.persistence.entity.ProductJpaEntity;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.util.List;

/**
 * Repository JPA Spring Data pour les produits.
 *
 * Responsabilités:
 * - Définir les queries JPA
 * - Étendre JpaRepository pour CRUD de base
 */
public interface ProductJpaRepository extends JpaRepository<ProductJpaEntity, Long> {

    // Queries dérivées du nom de méthode
    Page<ProductJpaEntity> findByIsActive(Boolean isActive, Pageable pageable);

    Page<ProductJpaEntity> findByCategoryId(Long categoryId, Pageable pageable);

    Page<ProductJpaEntity> findByNameContainingIgnoreCaseOrDescriptionContainingIgnoreCase(
        String name, String description, Pageable pageable);

    List<ProductJpaEntity> findTop10ByOrderByCreatedAtDesc();

    // Query personnalisée avec @Query
    @Query("SELECT p FROM ProductJpaEntity p WHERE p.isActive = true AND " +
           "(p.discountPrice IS NOT NULL AND p.discountPrice < p.price)")
    Page<ProductJpaEntity> findProductsOnSale(Pageable pageable);

    @Query("SELECT p FROM ProductJpaEntity p WHERE p.stockQuantity <= :threshold")
    List<ProductJpaEntity> findLowStockProducts(@Param("threshold") int threshold);
}
```

---

#### Repository Implementation (Adapter)

**Localisation**: `infrastructure/persistence/repository/ProductRepositoryImpl.java`

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

import java.util.List;
import java.util.Optional;
import java.util.stream.Collectors;

/**
 * Implémentation du port ProductRepository.
 *
 * Cette classe est un ADAPTER qui adapte JPA au Domain.
 *
 * Responsabilités:
 * - Implémenter l'interface ProductRepository (Domain)
 * - Utiliser ProductJpaRepository (Spring Data)
 * - Convertir Domain ↔ JPA via ProductMapper
 */
@Repository
@RequiredArgsConstructor
public class ProductRepositoryImpl implements ProductRepository {

    private final ProductJpaRepository jpaRepository;
    private final ProductMapper mapper;

    @Override
    public Product save(Product product) {
        // Convertir Domain → JPA
        ProductJpaEntity entity = mapper.toJpaEntity(product);

        // Sauvegarder via JPA
        ProductJpaEntity saved = jpaRepository.save(entity);

        // Convertir JPA → Domain
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
    public Page<Product> findActiveProducts(Pageable pageable) {
        return jpaRepository.findByIsActive(true, pageable)
            .map(mapper::toDomain);
    }

    @Override
    public Page<Product> findByCategory(CategoryId categoryId, Pageable pageable) {
        return jpaRepository.findByCategoryId(categoryId.getValue(), pageable)
            .map(mapper::toDomain);
    }

    @Override
    public Page<Product> searchByKeyword(String keyword, Pageable pageable) {
        return jpaRepository.findByNameContainingIgnoreCaseOrDescriptionContainingIgnoreCase(
            keyword, keyword, pageable)
            .map(mapper::toDomain);
    }

    @Override
    public List<Product> findNewProducts() {
        return jpaRepository.findTop10ByOrderByCreatedAtDesc()
            .stream()
            .map(mapper::toDomain)
            .collect(Collectors.toList());
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

    @Override
    public List<Product> findLowStockProducts(int threshold) {
        return jpaRepository.findLowStockProducts(threshold)
            .stream()
            .map(mapper::toDomain)
            .collect(Collectors.toList());
    }
}
```

---

#### Mapper (Infrastructure)

**Localisation**: `infrastructure/persistence/mapper/ProductMapper.java`

```java
package com.tdk.backend_tdk.infrastructure.persistence.mapper;

import com.tdk.backend_tdk.domain.entity.Product;
import com.tdk.backend_tdk.domain.entity.ProductImage;
import com.tdk.backend_tdk.domain.valueobject.*;
import com.tdk.backend_tdk.infrastructure.persistence.entity.ProductJpaEntity;
import com.tdk.backend_tdk.infrastructure.persistence.entity.ProductImageJpaEntity;
import org.springframework.stereotype.Component;

import java.math.BigDecimal;
import java.util.stream.Collectors;

/**
 * Mapper bidirectionnel entre Domain et JPA.
 *
 * Responsabilités:
 * - Convertir Domain Entity → JPA Entity
 * - Convertir JPA Entity → Domain Entity
 * - Gérer les Value Objects
 */
@Component
public class ProductMapper {

    /**
     * Convertir JPA Entity → Domain Entity
     */
    public Product toDomain(ProductJpaEntity entity) {
        if (entity == null) {
            return null;
        }

        return new Product(
            // ID
            entity.getId() != null ? ProductId.of(entity.getId()) : null,

            // Informations de base
            entity.getName(),
            entity.getDescription(),
            entity.getBrand(),

            // Prix (Value Object Money)
            new Money(entity.getPrice().doubleValue(), "XAF"),
            entity.getDiscountPrice() != null ?
                new Money(entity.getDiscountPrice().doubleValue(), "XAF") : null,

            // Stock
            entity.getStockQuantity(),

            // Catégorie (Value Object)
            CategoryId.of(entity.getCategoryId()),

            // Statut
            entity.getIsActive(),

            // Images
            entity.getImages().stream()
                .map(this::toProductImageDomain)
                .collect(Collectors.toList()),

            // Dates
            entity.getCreatedAt(),
            entity.getUpdatedAt()
        );
    }

    /**
     * Convertir Domain Entity → JPA Entity
     */
    public ProductJpaEntity toJpaEntity(Product domain) {
        if (domain == null) {
            return null;
        }

        ProductJpaEntity entity = new ProductJpaEntity();

        // ID
        if (domain.getId() != null) {
            entity.setId(domain.getId().getValue());
        }

        // Informations de base
        entity.setName(domain.getName());
        entity.setDescription(domain.getDescription());
        entity.setBrand(domain.getBrand());

        // Prix (Money → BigDecimal)
        entity.setPrice(BigDecimal.valueOf(domain.getPrice().getAmount()));
        if (domain.getDiscountPrice() != null) {
            entity.setDiscountPrice(BigDecimal.valueOf(domain.getDiscountPrice().getAmount()));
        }

        // Stock
        entity.setStockQuantity(domain.getStockQuantity());

        // Catégorie
        entity.setCategoryId(domain.getCategoryId().getValue());

        // Statut
        entity.setIsActive(domain.isActive());

        // Images
        entity.setImages(domain.getImages().stream()
            .map(this::toProductImageJpaEntity)
            .collect(Collectors.toList()));

        // Dates
        entity.setCreatedAt(domain.getCreatedAt());
        entity.setUpdatedAt(domain.getUpdatedAt());

        return entity;
    }

    /**
     * Mapper pour ProductImage
     */
    private ProductImage toProductImageDomain(ProductImageJpaEntity entity) {
        return new ProductImage(
            entity.getId() != null ? ProductImageId.of(entity.getId()) : null,
            ProductId.of(entity.getProductId()),
            entity.getImageUrl(),
            entity.getIsPrimary(),
            entity.getDisplayOrder()
        );
    }

    private ProductImageJpaEntity toProductImageJpaEntity(ProductImage domain) {
        ProductImageJpaEntity entity = new ProductImageJpaEntity();

        if (domain.getId() != null) {
            entity.setId(domain.getId().getValue());
        }

        entity.setProductId(domain.getProductId().getValue());
        entity.setImageUrl(domain.getImageUrl());
        entity.setIsPrimary(domain.isPrimary());
        entity.setDisplayOrder(domain.getDisplayOrder());

        return entity;
    }
}
```

---

#### B. Email Adapter

**Localisation**: `infrastructure/email/EmailAdapter.java`

```java
package com.tdk.backend_tdk.infrastructure.email;

import com.tdk.backend_tdk.application.port.output.EmailPort;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.mail.SimpleMailMessage;
import org.springframework.mail.javamail.JavaMailSender;
import org.springframework.mail.javamail.MimeMessageHelper;
import org.springframework.stereotype.Service;

import jakarta.mail.MessagingException;
import jakarta.mail.internet.MimeMessage;

/**
 * Adaptateur pour l'envoi d'emails.
 * Implémente le port EmailPort défini dans la couche Application.
 *
 * Responsabilités:
 * - Implémenter EmailPort
 * - Utiliser JavaMailSender (Spring)
 * - Gérer les templates HTML
 */
@Service
@RequiredArgsConstructor
@Slf4j
public class EmailAdapter implements EmailPort {

    private final JavaMailSender mailSender;

    @Value("${spring.mail.username}")
    private String fromEmail;

    @Value("${app.frontend.url}")
    private String frontendUrl;

    @Override
    public void sendEmail(String to, String subject, String body) {
        try {
            SimpleMailMessage message = new SimpleMailMessage();
            message.setFrom(fromEmail);
            message.setTo(to);
            message.setSubject(subject);
            message.setText(body);

            mailSender.send(message);
            log.info("Email envoyé avec succès à: {}", to);

        } catch (Exception e) {
            log.error("Erreur lors de l'envoi de l'email à: {}", to, e);
            throw new RuntimeException("Échec de l'envoi de l'email", e);
        }
    }

    @Override
    public void sendHtmlEmail(String to, String subject, String htmlContent) {
        try {
            MimeMessage message = mailSender.createMimeMessage();
            MimeMessageHelper helper = new MimeMessageHelper(message, true, "UTF-8");

            helper.setFrom(fromEmail);
            helper.setTo(to);
            helper.setSubject(subject);
            helper.setText(htmlContent, true);

            mailSender.send(message);
            log.info("Email HTML envoyé avec succès à: {}", to);

        } catch (MessagingException e) {
            log.error("Erreur lors de l'envoi de l'email HTML à: {}", to, e);
            throw new RuntimeException("Échec de l'envoi de l'email HTML", e);
        }
    }

    @Override
    public void sendWelcomeEmail(String to, String username) {
        String subject = "Bienvenue sur TKD !";
        String htmlContent = buildWelcomeEmailTemplate(username);
        sendHtmlEmail(to, subject, htmlContent);
    }

    @Override
    public void sendPasswordResetEmail(String to, String resetToken) {
        String subject = "Réinitialisation de votre mot de passe";
        String htmlContent = buildPasswordResetEmailTemplate(resetToken);
        sendHtmlEmail(to, subject, htmlContent);
    }

    @Override
    public void sendOrderConfirmationEmail(String to, String orderNumber) {
        String subject = "Confirmation de commande - " + orderNumber;
        String htmlContent = buildOrderConfirmationTemplate(orderNumber);
        sendHtmlEmail(to, subject, htmlContent);
    }

    @Override
    public void sendOrderShippedEmail(String to, String orderNumber, String trackingNumber) {
        String subject = "Votre commande a été expédiée - " + orderNumber;
        String htmlContent = buildOrderShippedTemplate(orderNumber, trackingNumber);
        sendHtmlEmail(to, subject, htmlContent);
    }

    @Override
    public void sendOrderDeliveredEmail(String to, String orderNumber) {
        String subject = "Votre commande a été livrée - " + orderNumber;
        String htmlContent = buildOrderDeliveredTemplate(orderNumber);
        sendHtmlEmail(to, subject, htmlContent);
    }

    // Templates HTML privés
    private String buildWelcomeEmailTemplate(String username) {
        return """
            <!DOCTYPE html>
            <html>
            <head>
                <style>
                    body { font-family: Arial, sans-serif; }
                    .container { max-width: 600px; margin: 0 auto; padding: 20px; }
                    .header { background-color: #4CAF50; color: white; padding: 20px; text-align: center; }
                    .content { padding: 20px; }
                    .button {
                        background-color: #4CAF50;
                        color: white;
                        padding: 12px 24px;
                        text-decoration: none;
                        display: inline-block;
                        border-radius: 4px;
                    }
                </style>
            </head>
            <body>
                <div class="container">
                    <div class="header">
                        <h1>Bienvenue sur TKD !</h1>
                    </div>
                    <div class="content">
                        <h2>Bonjour %s,</h2>
                        <p>Merci de vous être inscrit sur notre plateforme e-commerce.</p>
                        <p>Nous sommes ravis de vous compter parmi nous !</p>
                        <p>
                            <a href="%s" class="button">Commencer vos achats</a>
                        </p>
                        <p>Cordialement,<br>L'équipe TKD</p>
                    </div>
                </div>
            </body>
            </html>
            """.formatted(username, frontendUrl);
    }

    private String buildPasswordResetEmailTemplate(String resetToken) {
        String resetLink = frontendUrl + "/reset-password?token=" + resetToken;
        return """
            <!DOCTYPE html>
            <html>
            <head>
                <style>
                    body { font-family: Arial, sans-serif; }
                    .container { max-width: 600px; margin: 0 auto; padding: 20px; }
                    .button {
                        background-color: #f44336;
                        color: white;
                        padding: 12px 24px;
                        text-decoration: none;
                        display: inline-block;
                        border-radius: 4px;
                    }
                </style>
            </head>
            <body>
                <div class="container">
                    <h2>Réinitialisation de mot de passe</h2>
                    <p>Vous avez demandé à réinitialiser votre mot de passe.</p>
                    <p>Cliquez sur le bouton ci-dessous pour procéder :</p>
                    <p>
                        <a href="%s" class="button">Réinitialiser le mot de passe</a>
                    </p>
                    <p>Ce lien expire dans 1 heure.</p>
                    <p>Si vous n'avez pas demandé cette réinitialisation, ignorez cet email.</p>
                </div>
            </body>
            </html>
            """.formatted(resetLink);
    }

    private String buildOrderConfirmationTemplate(String orderNumber) {
        String orderLink = frontendUrl + "/orders/" + orderNumber;
        return """
            <!DOCTYPE html>
            <html>
            <body>
                <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
                    <h2>Confirmation de commande</h2>
                    <p>Votre commande <strong>%s</strong> a été confirmée avec succès.</p>
                    <p>Nous préparons votre commande et vous tiendrons informé de son expédition.</p>
                    <p><a href="%s">Suivre ma commande</a></p>
                    <p>Merci pour votre achat !<br>L'équipe TKD</p>
                </div>
            </body>
            </html>
            """.formatted(orderNumber, orderLink);
    }

    private String buildOrderShippedTemplate(String orderNumber, String trackingNumber) {
        return """
            <!DOCTYPE html>
            <html>
            <body>
                <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
                    <h2>Votre commande a été expédiée !</h2>
                    <p>Bonne nouvelle ! Votre commande <strong>%s</strong> est en route.</p>
                    <p>Numéro de suivi: <strong>%s</strong></p>
                    <p>Vous devriez recevoir votre colis sous 3-5 jours ouvrables.</p>
                    <p>Merci de votre confiance !<br>L'équipe TKD</p>
                </div>
            </body>
            </html>
            """.formatted(orderNumber, trackingNumber);
    }

    private String buildOrderDeliveredTemplate(String orderNumber) {
        return """
            <!DOCTYPE html>
            <html>
            <body>
                <div style="font-family: Arial, sans-serif; max-width: 600px; margin: 0 auto;">
                    <h2>Votre commande a été livrée !</h2>
                    <p>Votre commande <strong>%s</strong> a été livrée avec succès.</p>
                    <p>Nous espérons que vous êtes satisfait de votre achat.</p>
                    <p>N'hésitez pas à nous laisser un avis !</p>
                    <p>À bientôt !<br>L'équipe TKD</p>
                </div>
            </body>
            </html>
            """.formatted(orderNumber);
    }
}
```

---

### Continuer avec Payment Adapter et autres?

Le document est déjà très long. Voulez-vous que je continue avec:
1. Payment Adapters (Stripe, MTN, Orange Money)
2. File Storage Adapter
3. Security (JWT)
4. Couche 4 (Frameworks & Drivers - Configuration Spring)

Ou préférez-vous que je crée un deuxième document pour la suite?
