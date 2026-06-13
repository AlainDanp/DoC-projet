# Entités et Use Cases - Clean Architecture

Ce document liste toutes les entités du domaine et les use cases à créer pour votre migration vers Clean Architecture.

---

## Table des matières

1. [Entités du Domaine](#entités-du-domaine)
2. [Value Objects](#value-objects)
3. [Use Cases par Module](#use-cases-par-module)
4. [Ports (Interfaces)](#ports-interfaces)

---

## Entités du Domaine

### 1. Product (Produit)

**Localisation**: `domain/entity/Product.java`

**Attributs**:
- `ProductId id` - Identifiant unique
- `String name` - Nom du produit
- `String description` - Description
- `Money price` - Prix normal
- `Money discountPrice` - Prix avec réduction (optionnel)
- `int stockQuantity` - Quantité en stock
- `CategoryId categoryId` - Référence à la catégorie
- `String brand` - Marque
- `boolean active` - Produit actif ou non
- `List<ProductImage> images` - Liste des images
- `Date createdAt` - Date de création
- `Date updatedAt` - Date de modification

**Méthodes métier**:
```java
void applyDiscount(Money discountPrice)
void removeDiscount()
void increaseStock(int quantity)
void decreaseStock(int quantity)
boolean isInStock()
boolean hasDiscount()
Money getEffectivePrice()
void activate()
void deactivate()
void addImage(ProductImage image)
void removeImage(ProductImage image)
void setPrimaryImage(ProductImage image)
```

---

### 2. Order (Commande)

**Localisation**: `domain/entity/Order.java`

**Attributs**:
- `OrderId id` - Identifiant unique
- `UserId userId` - Référence à l'utilisateur
- `String orderNumber` - Numéro de commande unique
- `List<OrderItem> items` - Articles de la commande
- `Money totalAmount` - Montant total
- `OrderStatus status` - Statut (PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED)
- `PaymentMethod paymentMethod` - Méthode de paiement
- `PaymentStatus paymentStatus` - Statut du paiement
- `Address shippingAddress` - Adresse de livraison
- `ShippingInfo shippingInfo` - Informations de livraison
- `String notes` - Notes de la commande
- `String trackingNumber` - Numéro de suivi
- `Date createdAt` - Date de création
- `Date confirmedAt` - Date de confirmation
- `Date shippedAt` - Date d'expédition
- `Date deliveredAt` - Date de livraison

**Méthodes métier**:
```java
void confirm()
void ship(String trackingNumber)
void deliver()
void cancel()
boolean canBeCancelled()
boolean isDelivered()
boolean isPaid()
void markAsPaid(String transactionId)
int getTotalItems()
Money calculateTotal()
void addItem(OrderItem item)
void removeItem(OrderItem item)
static String generateOrderNumber()
```

---

### 3. OrderItem (Article de commande)

**Localisation**: `domain/entity/OrderItem.java`

**Attributs**:
- `OrderItemId id` - Identifiant unique
- `ProductId productId` - Référence au produit
- `String productName` - Nom du produit (snapshot)
- `int quantity` - Quantité commandée
- `Money unitPrice` - Prix unitaire au moment de la commande
- `Money discountPrice` - Prix réduit (optionnel)

**Méthodes métier**:
```java
Money getSubtotal()
Money getEffectiveUnitPrice()
boolean hasDiscount()
```

---

### 4. User (Utilisateur)

**Localisation**: `domain/entity/User.java`

**Attributs**:
- `UserId id` - Identifiant unique
- `Username username` - Nom d'utilisateur
- `Email email` - Email
- `Password password` - Mot de passe (hashé)
- `PersonalInfo personalInfo` - Informations personnelles
- `Set<Role> roles` - Rôles de l'utilisateur
- `boolean active` - Compte actif
- `boolean emailVerified` - Email vérifié
- `String emailVerificationToken` - Token de vérification email
- `PasswordResetToken passwordResetToken` - Token de reset (optionnel)
- `Date lastLogin` - Dernière connexion
- `Date createdAt` - Date de création

**Méthodes métier**:
```java
void activate()
void deactivate()
void verifyEmail()
boolean isActive()
boolean isEmailVerified()
void updatePassword(Password newPassword)
void initiatePasswordReset()
void resetPassword(Password newPassword, String token)
boolean hasRole(String roleName)
void addRole(Role role)
void removeRole(Role role)
```

---

### 5. Cart (Panier)

**Localisation**: `domain/entity/Cart.java`

**Attributs**:
- `CartId id` - Identifiant unique
- `UserId userId` - Référence à l'utilisateur
- `List<CartItem> items` - Articles du panier
- `Date createdAt` - Date de création
- `Date updatedAt` - Date de modification

**Méthodes métier**:
```java
void addItem(Product product, int quantity)
void removeItem(CartItem item)
void updateItemQuantity(CartItemId itemId, int quantity)
void clear()
boolean isEmpty()
Money getTotalAmount()
int getTotalItems()
CartItem findItemByProductId(ProductId productId)
boolean hasProduct(ProductId productId)
```

---

### 6. CartItem (Article du panier)

**Localisation**: `domain/entity/CartItem.java`

**Attributs**:
- `CartItemId id` - Identifiant unique
- `ProductId productId` - Référence au produit
- `Product product` - Produit (pour accès aux infos)
- `int quantity` - Quantité

**Méthodes métier**:
```java
Money getSubtotal()
void increaseQuantity(int amount)
void decreaseQuantity(int amount)
void setQuantity(int quantity)
```

---

### 7. Category (Catégorie)

**Localisation**: `domain/entity/Category.java`

**Attributs**:
- `CategoryId id` - Identifiant unique
- `String name` - Nom de la catégorie
- `String description` - Description
- `String imageUrl` - URL de l'image
- `CategoryId parentId` - Catégorie parente (optionnel)
- `List<CategoryId> subCategoryIds` - Sous-catégories
- `Date createdAt` - Date de création

**Méthodes métier**:
```java
boolean isRootCategory()
boolean hasSubCategories()
void setParent(CategoryId parentId)
```

---

### 8. Address (Adresse)

**Localisation**: `domain/entity/Address.java`

**Attributs**:
- `AddressId id` - Identifiant unique
- `UserId userId` - Référence à l'utilisateur
- `String fullName` - Nom complet
- `PhoneNumber phoneNumber` - Numéro de téléphone
- `String addressLine1` - Ligne d'adresse 1
- `String addressLine2` - Ligne d'adresse 2 (optionnel)
- `String city` - Ville
- `String state` - État/Province (optionnel)
- `String postalCode` - Code postal
- `String country` - Pays
- `boolean isDefault` - Adresse par défaut
- `Date createdAt` - Date de création

**Méthodes métier**:
```java
String getFormattedAddress()
void setAsDefault()
void unsetAsDefault()
boolean isComplete()
```

---

### 9. Role (Rôle)

**Localisation**: `domain/entity/Role.java`

**Attributs**:
- `RoleId id` - Identifiant unique
- `String name` - Nom du rôle (USER, ADMIN, MODERATOR)
- `String description` - Description du rôle
- `Date createdAt` - Date de création

**Méthodes métier**:
```java
boolean isAdmin()
boolean isUser()
```

---

### 10. ProductImage (Image de produit)

**Localisation**: `domain/entity/ProductImage.java`

**Attributs**:
- `ProductImageId id` - Identifiant unique
- `ProductId productId` - Référence au produit
- `String imageUrl` - URL de l'image
- `boolean isPrimary` - Image principale
- `int displayOrder` - Ordre d'affichage

**Méthodes métier**:
```java
void setAsPrimary()
void unsetAsPrimary()
```

---

### 11. Payment (Paiement)

**Localisation**: `domain/entity/Payment.java`

**Attributs**:
- `PaymentId id` - Identifiant unique
- `OrderId orderId` - Référence à la commande
- `Money amount` - Montant
- `PaymentMethod method` - Méthode de paiement
- `PaymentStatus status` - Statut du paiement
- `String transactionId` - ID de transaction externe
- `Date createdAt` - Date de création
- `Date completedAt` - Date de complétion

**Méthodes métier**:
```java
void markAsCompleted(String transactionId)
void markAsFailed(String reason)
boolean isCompleted()
boolean isFailed()
```

---

## Value Objects

Les Value Objects sont des objets immuables qui représentent des concepts du domaine.

### 1. Money (Argent)

**Localisation**: `domain/valueobject/Money.java`

```java
public class Money {
    private final double amount;
    private final String currency;

    Money add(Money other)
    Money subtract(Money other)
    Money multiply(int factor)
    boolean isGreaterThan(Money other)
    boolean isLessThan(Money other)
    boolean isZero()
}
```

### 2. Email

**Localisation**: `domain/valueobject/Email.java`

```java
public class Email {
    private final String value;

    // Validation dans le constructeur
    boolean isValid()
    String getDomain()
}
```

### 3. PhoneNumber

**Localisation**: `domain/valueobject/PhoneNumber.java`

```java
public class PhoneNumber {
    private final String value;

    // Validation format téléphone
    boolean isValid()
    String getFormattedValue()
}
```

### 4. Password

**Localisation**: `domain/valueobject/Password.java`

```java
public class Password {
    private final String hashedValue;

    static Password fromPlainText(String plainText, PasswordEncoder encoder)
    boolean matches(String plainText, PasswordEncoder encoder)
}
```

### 5. PersonalInfo

**Localisation**: `domain/valueobject/PersonalInfo.java`

```java
public class PersonalInfo {
    private final String firstName;
    private final String lastName;
    private final Date dateOfBirth;
    private final String avatar;

    String getFullName()
    int getAge()
}
```

### 6. ShippingInfo

**Localisation**: `domain/valueobject/ShippingInfo.java`

```java
public class ShippingInfo {
    private final String fullName;
    private final PhoneNumber phoneNumber;
    private final String addressText;
}
```

### 7. PasswordResetToken

**Localisation**: `domain/valueobject/PasswordResetToken.java`

```java
public class PasswordResetToken {
    private final String token;
    private final Date expiryDate;

    boolean isExpired()
    boolean isValid()
}
```

### 8. Identifiants (IDs)

Tous les IDs sont des Value Objects:

```java
// domain/valueobject/ProductId.java
public class ProductId {
    private final Long value;
}

// domain/valueobject/OrderId.java
public class OrderId {
    private final Long value;
}

// domain/valueobject/UserId.java
public class UserId {
    private final Long value;
}

// domain/valueobject/CategoryId.java
public class CategoryId {
    private final Long value;
}

// domain/valueobject/CartId.java
public class CartId {
    private final Long value;
}

// domain/valueobject/AddressId.java
public class AddressId {
    private final Long value;
}

// ... etc pour tous les IDs
```

### 9. Enums

Les enums sont aussi des Value Objects:

```java
// domain/valueobject/OrderStatus.java
public enum OrderStatus {
    PENDING, CONFIRMED, SHIPPED, DELIVERED, CANCELLED
}

// domain/valueobject/PaymentStatus.java
public enum PaymentStatus {
    PENDING, COMPLETED, FAILED, REFUNDED
}

// domain/valueobject/PaymentMethod.java
public enum PaymentMethod {
    STRIPE, MTN_MOBILE_MONEY, ORANGE_MONEY
}
```

---

## Use Cases par Module

### Module: Product

**Localisation**: `application/usecase/product/`

#### 1. CreateProductUseCase
**Responsabilité**: Créer un nouveau produit

**Input**: `CreateProductCommand`
```java
public class CreateProductCommand {
    String name;
    String description;
    Double price;
    Double discountPrice;
    Integer stockQuantity;
    Long categoryId;
    String brand;
}
```

**Output**: `Product`

**Dépendances**:
- `ProductRepository`
- `CategoryRepository`

---

#### 2. UpdateProductUseCase
**Responsabilité**: Mettre à jour un produit existant

**Input**: `UpdateProductCommand`
```java
public class UpdateProductCommand {
    Long productId;
    String name;
    String description;
    Double price;
    Double discountPrice;
    Integer stockQuantity;
    Long categoryId;
    String brand;
    Boolean isActive;
}
```

**Output**: `Product`

**Dépendances**:
- `ProductRepository`
- `CategoryRepository`

---

#### 3. GetProductUseCase
**Responsabilité**: Récupérer un produit par son ID

**Input**: `Long productId`

**Output**: `Product`

**Dépendances**:
- `ProductRepository`

---

#### 4. GetAllProductsUseCase
**Responsabilité**: Récupérer tous les produits (avec pagination)

**Input**: `Pageable`

**Output**: `Page<Product>`

**Dépendances**:
- `ProductRepository`

---

#### 5. GetActiveProductsUseCase
**Responsabilité**: Récupérer les produits actifs

**Input**: `Pageable`

**Output**: `Page<Product>`

**Dépendances**:
- `ProductRepository`

---

#### 6. GetProductsByCategoryUseCase
**Responsabilité**: Récupérer les produits d'une catégorie

**Input**: `GetProductsByCategoryQuery`
```java
public class GetProductsByCategoryQuery {
    Long categoryId;
    Pageable pageable;
}
```

**Output**: `Page<Product>`

**Dépendances**:
- `ProductRepository`

---

#### 7. SearchProductsUseCase
**Responsabilité**: Rechercher des produits par mot-clé

**Input**: `SearchProductsQuery`
```java
public class SearchProductsQuery {
    String keyword;
    Pageable pageable;
}
```

**Output**: `Page<Product>`

**Dépendances**:
- `ProductRepository`

---

#### 8. GetNewProductsUseCase
**Responsabilité**: Récupérer les nouveaux produits

**Input**: Aucun

**Output**: `List<Product>`

**Dépendances**:
- `ProductRepository`

---

#### 9. GetProductsOnSaleUseCase
**Responsabilité**: Récupérer les produits en promotion

**Input**: `Pageable`

**Output**: `Page<Product>`

**Dépendances**:
- `ProductRepository`

---

#### 10. DeleteProductUseCase
**Responsabilité**: Supprimer un produit

**Input**: `Long productId`

**Output**: `void`

**Dépendances**:
- `ProductRepository`
- `FileStoragePort`

---

#### 11. AddProductImageUseCase
**Responsabilité**: Ajouter une image à un produit

**Input**: `AddProductImageCommand`
```java
public class AddProductImageCommand {
    Long productId;
    MultipartFile file;
    Boolean isPrimary;
}
```

**Output**: `ProductImage`

**Dépendances**:
- `ProductRepository`
- `FileStoragePort`

---

#### 12. DeleteProductImageUseCase
**Responsabilité**: Supprimer une image de produit

**Input**: `Long imageId`

**Output**: `void`

**Dépendances**:
- `ProductRepository`
- `FileStoragePort`

---

#### 13. UpdateProductStockUseCase
**Responsabilité**: Mettre à jour le stock d'un produit

**Input**: `UpdateProductStockCommand`
```java
public class UpdateProductStockCommand {
    Long productId;
    Integer quantity;
    StockOperation operation; // ADD, SUBTRACT, SET
}
```

**Output**: `Product`

**Dépendances**:
- `ProductRepository`

---

### Module: Order

**Localisation**: `application/usecase/order/`

#### 1. CreateOrderUseCase
**Responsabilité**: Créer une commande à partir du panier

**Input**: `CreateOrderCommand`
```java
public class CreateOrderCommand {
    String userEmail;
    Long shippingAddressId;
    Long billingAddressId; // optionnel
    PaymentMethod paymentMethod;
    String notes;
}
```

**Output**: `Order`

**Dépendances**:
- `OrderRepository`
- `CartRepository`
- `ProductRepository`
- `UserRepository`
- `AddressRepository`
- `PaymentPort`
- `EmailPort`

---

#### 2. GetOrderUseCase
**Responsabilité**: Récupérer une commande par ID

**Input**: `GetOrderQuery`
```java
public class GetOrderQuery {
    String userEmail;
    Long orderId;
}
```

**Output**: `Order`

**Dépendances**:
- `OrderRepository`

---

#### 3. GetOrderByNumberUseCase
**Responsabilité**: Récupérer une commande par numéro

**Input**: `GetOrderByNumberQuery`
```java
public class GetOrderByNumberQuery {
    String userEmail;
    String orderNumber;
}
```

**Output**: `Order`

**Dépendances**:
- `OrderRepository`

---

#### 4. GetUserOrdersUseCase
**Responsabilité**: Récupérer les commandes d'un utilisateur

**Input**: `GetUserOrdersQuery`
```java
public class GetUserOrdersQuery {
    String userEmail;
    Pageable pageable;
}
```

**Output**: `Page<Order>`

**Dépendances**:
- `OrderRepository`
- `UserRepository`

---

#### 5. CancelOrderUseCase
**Responsabilité**: Annuler une commande

**Input**: `CancelOrderCommand`
```java
public class CancelOrderCommand {
    String userEmail;
    Long orderId;
    String cancellationReason;
}
```

**Output**: `Order`

**Dépendances**:
- `OrderRepository`
- `ProductRepository`
- `EmailPort`

---

#### 6. ConfirmOrderUseCase (Admin)
**Responsabilité**: Confirmer une commande

**Input**: `Long orderId`

**Output**: `Order`

**Dépendances**:
- `OrderRepository`
- `EmailPort`

---

#### 7. ShipOrderUseCase (Admin)
**Responsabilité**: Marquer une commande comme expédiée

**Input**: `ShipOrderCommand`
```java
public class ShipOrderCommand {
    Long orderId;
    String trackingNumber;
}
```

**Output**: `Order`

**Dépendances**:
- `OrderRepository`
- `EmailPort`

---

#### 8. DeliverOrderUseCase (Admin)
**Responsabilité**: Marquer une commande comme livrée

**Input**: `Long orderId`

**Output**: `Order`

**Dépendances**:
- `OrderRepository`
- `EmailPort`

---

#### 9. GetAllOrdersUseCase (Admin)
**Responsabilité**: Récupérer toutes les commandes (admin)

**Input**: `GetAllOrdersQuery`
```java
public class GetAllOrdersQuery {
    OrderStatus status; // optionnel
    Pageable pageable;
}
```

**Output**: `Page<Order>`

**Dépendances**:
- `OrderRepository`

---

### Module: User / Auth

**Localisation**: `application/usecase/auth/` et `application/usecase/user/`

#### 1. RegisterUserUseCase
**Responsabilité**: Enregistrer un nouvel utilisateur

**Input**: `RegisterUserCommand`
```java
public class RegisterUserCommand {
    String username;
    String email;
    String password;
    String confirmPassword;
    String firstName;
    String lastName;
    String phoneNumber;
    Set<String> roles;
}
```

**Output**: `RegisterResult`
```java
public class RegisterResult {
    String message;
    boolean success;
}
```

**Dépendances**:
- `UserRepository`
- `RoleRepository`
- `PasswordEncoder`
- `EmailPort`

---

#### 2. LoginUseCase
**Responsabilité**: Authentifier un utilisateur

**Input**: `LoginCommand`
```java
public class LoginCommand {
    String email;
    String password;
}
```

**Output**: `AuthToken`
```java
public class AuthToken {
    String token;
    Long userId;
    String username;
    String email;
    List<String> roles;
    Date expiresAt;
}
```

**Dépendances**:
- `UserRepository`
- `PasswordEncoder`
- `TokenPort` (JWT)

---

#### 3. VerifyEmailUseCase
**Responsabilité**: Vérifier l'email avec le token

**Input**: `String token`

**Output**: `VerificationResult`

**Dépendances**:
- `UserRepository`

---

#### 4. ForgotPasswordUseCase
**Responsabilité**: Initier la réinitialisation de mot de passe

**Input**: `ForgotPasswordCommand`
```java
public class ForgotPasswordCommand {
    String email;
}
```

**Output**: `MessageResult`

**Dépendances**:
- `UserRepository`
- `EmailPort`

---

#### 5. ResetPasswordUseCase
**Responsabilité**: Réinitialiser le mot de passe avec le token

**Input**: `ResetPasswordCommand`
```java
public class ResetPasswordCommand {
    String token;
    String newPassword;
    String confirmPassword;
}
```

**Output**: `MessageResult`

**Dépendances**:
- `UserRepository`
- `PasswordEncoder`

---

#### 6. ResendVerificationEmailUseCase
**Responsabilité**: Renvoyer l'email de vérification

**Input**: `String email`

**Output**: `MessageResult`

**Dépendances**:
- `UserRepository`
- `EmailPort`

---

#### 7. RefreshTokenUseCase
**Responsabilité**: Rafraîchir le token JWT

**Input**: `String token`

**Output**: `AuthToken`

**Dépendances**:
- `UserRepository`
- `TokenPort`

---

#### 8. GetUserProfileUseCase
**Responsabilité**: Récupérer le profil utilisateur

**Input**: `String userEmail`

**Output**: `User`

**Dépendances**:
- `UserRepository`

---

#### 9. UpdateUserProfileUseCase
**Responsabilité**: Mettre à jour le profil utilisateur

**Input**: `UpdateUserProfileCommand`
```java
public class UpdateUserProfileCommand {
    String userEmail;
    String firstName;
    String lastName;
    String phoneNumber;
    Date dateOfBirth;
    String city;
    String country;
}
```

**Output**: `User`

**Dépendances**:
- `UserRepository`

---

#### 10. ChangePasswordUseCase
**Responsabilité**: Changer le mot de passe (utilisateur connecté)

**Input**: `ChangePasswordCommand`
```java
public class ChangePasswordCommand {
    String userEmail;
    String currentPassword;
    String newPassword;
    String confirmPassword;
}
```

**Output**: `MessageResult`

**Dépendances**:
- `UserRepository`
- `PasswordEncoder`

---

#### 11. UpdateAvatarUseCase
**Responsabilité**: Mettre à jour l'avatar de l'utilisateur

**Input**: `UpdateAvatarCommand`
```java
public class UpdateAvatarCommand {
    String userEmail;
    MultipartFile avatarFile;
}
```

**Output**: `User`

**Dépendances**:
- `UserRepository`
- `FileStoragePort`

---

#### 12. DeleteUserAccountUseCase
**Responsabilité**: Supprimer le compte utilisateur

**Input**: `DeleteUserAccountCommand`
```java
public class DeleteUserAccountCommand {
    String userEmail;
    String password; // confirmation
}
```

**Output**: `MessageResult`

**Dépendances**:
- `UserRepository`
- `PasswordEncoder`

---

### Module: Cart

**Localisation**: `application/usecase/cart/`

#### 1. GetOrCreateCartUseCase
**Responsabilité**: Récupérer ou créer le panier de l'utilisateur

**Input**: `String userEmail`

**Output**: `Cart`

**Dépendances**:
- `CartRepository`
- `UserRepository`

---

#### 2. AddToCartUseCase
**Responsabilité**: Ajouter un produit au panier

**Input**: `AddToCartCommand`
```java
public class AddToCartCommand {
    String userEmail;
    Long productId;
    Integer quantity;
}
```

**Output**: `Cart`

**Dépendances**:
- `CartRepository`
- `ProductRepository`
- `UserRepository`

---

#### 3. UpdateCartItemUseCase
**Responsabilité**: Mettre à jour la quantité d'un article

**Input**: `UpdateCartItemCommand`
```java
public class UpdateCartItemCommand {
    String userEmail;
    Long cartItemId;
    Integer quantity;
}
```

**Output**: `Cart`

**Dépendances**:
- `CartRepository`
- `ProductRepository`

---

#### 4. RemoveFromCartUseCase
**Responsabilité**: Supprimer un article du panier

**Input**: `RemoveFromCartCommand`
```java
public class RemoveFromCartCommand {
    String userEmail;
    Long cartItemId;
}
```

**Output**: `Cart`

**Dépendances**:
- `CartRepository`

---

#### 5. ClearCartUseCase
**Responsabilité**: Vider le panier

**Input**: `String userEmail`

**Output**: `void`

**Dépendances**:
- `CartRepository`

---

#### 6. GetCartTotalUseCase
**Responsabilité**: Obtenir le total du panier

**Input**: `String userEmail`

**Output**: `Money`

**Dépendances**:
- `CartRepository`

---

### Module: Category

**Localisation**: `application/usecase/category/`

#### 1. CreateCategoryUseCase
**Responsabilité**: Créer une catégorie

**Input**: `CreateCategoryCommand`
```java
public class CreateCategoryCommand {
    String name;
    String description;
    String imageUrl;
    Long parentId; // optionnel
}
```

**Output**: `Category`

**Dépendances**:
- `CategoryRepository`

---

#### 2. UpdateCategoryUseCase
**Responsabilité**: Mettre à jour une catégorie

**Input**: `UpdateCategoryCommand`
```java
public class UpdateCategoryCommand {
    Long categoryId;
    String name;
    String description;
    String imageUrl;
    Long parentId;
}
```

**Output**: `Category`

**Dépendances**:
- `CategoryRepository`

---

#### 3. DeleteCategoryUseCase
**Responsabilité**: Supprimer une catégorie

**Input**: `Long categoryId`

**Output**: `void`

**Dépendances**:
- `CategoryRepository`
- `ProductRepository` (vérifier qu'aucun produit n'est lié)

---

#### 4. GetCategoryUseCase
**Responsabilité**: Récupérer une catégorie par ID

**Input**: `Long categoryId`

**Output**: `Category`

**Dépendances**:
- `CategoryRepository`

---

#### 5. GetAllCategoriesUseCase
**Responsabilité**: Récupérer toutes les catégories

**Input**: Aucun

**Output**: `List<Category>`

**Dépendances**:
- `CategoryRepository`

---

#### 6. GetRootCategoriesUseCase
**Responsabilité**: Récupérer les catégories racines

**Input**: Aucun

**Output**: `List<Category>`

**Dépendances**:
- `CategoryRepository`

---

#### 7. GetSubCategoriesUseCase
**Responsabilité**: Récupérer les sous-catégories d'une catégorie

**Input**: `Long parentCategoryId`

**Output**: `List<Category>`

**Dépendances**:
- `CategoryRepository`

---

### Module: Address

**Localisation**: `application/usecase/address/`

#### 1. CreateAddressUseCase
**Responsabilité**: Créer une nouvelle adresse

**Input**: `CreateAddressCommand`
```java
public class CreateAddressCommand {
    String userEmail;
    String fullName;
    String phoneNumber;
    String addressLine1;
    String addressLine2;
    String city;
    String state;
    String postalCode;
    String country;
    Boolean isDefault;
}
```

**Output**: `Address`

**Dépendances**:
- `AddressRepository`
- `UserRepository`

---

#### 2. UpdateAddressUseCase
**Responsabilité**: Mettre à jour une adresse

**Input**: `UpdateAddressCommand`
```java
public class UpdateAddressCommand {
    String userEmail;
    Long addressId;
    String fullName;
    String phoneNumber;
    String addressLine1;
    String addressLine2;
    String city;
    String state;
    String postalCode;
    String country;
    Boolean isDefault;
}
```

**Output**: `Address`

**Dépendances**:
- `AddressRepository`

---

#### 3. DeleteAddressUseCase
**Responsabilité**: Supprimer une adresse

**Input**: `DeleteAddressCommand`
```java
public class DeleteAddressCommand {
    String userEmail;
    Long addressId;
}
```

**Output**: `void`

**Dépendances**:
- `AddressRepository`

---

#### 4. GetAddressUseCase
**Responsabilité**: Récupérer une adresse par ID

**Input**: `GetAddressQuery`
```java
public class GetAddressQuery {
    String userEmail;
    Long addressId;
}
```

**Output**: `Address`

**Dépendances**:
- `AddressRepository`

---

#### 5. GetUserAddressesUseCase
**Responsabilité**: Récupérer toutes les adresses d'un utilisateur

**Input**: `String userEmail`

**Output**: `List<Address>`

**Dépendances**:
- `AddressRepository`
- `UserRepository`

---

#### 6. SetDefaultAddressUseCase
**Responsabilité**: Définir une adresse comme adresse par défaut

**Input**: `SetDefaultAddressCommand`
```java
public class SetDefaultAddressCommand {
    String userEmail;
    Long addressId;
}
```

**Output**: `Address`

**Dépendances**:
- `AddressRepository`

---

### Module: Payment

**Localisation**: `application/usecase/payment/`

#### 1. ProcessStripePaymentUseCase
**Responsabilité**: Traiter un paiement Stripe

**Input**: `ProcessStripePaymentCommand`
```java
public class ProcessStripePaymentCommand {
    Long orderId;
    String paymentMethodId;
    String userEmail;
}
```

**Output**: `PaymentResult`

**Dépendances**:
- `OrderRepository`
- `PaymentPort` (Stripe)

---

#### 2. ProcessMtnPaymentUseCase
**Responsabilité**: Traiter un paiement MTN Mobile Money

**Input**: `ProcessMtnPaymentCommand`
```java
public class ProcessMtnPaymentCommand {
    Long orderId;
    String phoneNumber;
    String userEmail;
}
```

**Output**: `PaymentResult`

**Dépendances**:
- `OrderRepository`
- `PaymentPort` (MTN)

---

#### 3. ProcessOrangeMoneyPaymentUseCase
**Responsabilité**: Traiter un paiement Orange Money

**Input**: `ProcessOrangeMoneyPaymentCommand`
```java
public class ProcessOrangeMoneyPaymentCommand {
    Long orderId;
    String phoneNumber;
    String userEmail;
}
```

**Output**: `PaymentResult`

**Dépendances**:
- `OrderRepository`
- `PaymentPort` (Orange Money)

---

#### 4. CheckPaymentStatusUseCase
**Responsabilité**: Vérifier le statut d'un paiement

**Input**: `String transactionId`

**Output**: `PaymentStatus`

**Dépendances**:
- `PaymentPort`

---

#### 5. RefundPaymentUseCase
**Responsabilité**: Rembourser un paiement

**Input**: `RefundPaymentCommand`
```java
public class RefundPaymentCommand {
    String transactionId;
    Double amount;
    String reason;
}
```

**Output**: `PaymentResult`

**Dépendances**:
- `PaymentPort`

---

### Module: Admin / Statistics

**Localisation**: `application/usecase/admin/`

#### 1. GetDashboardStatisticsUseCase
**Responsabilité**: Récupérer les statistiques du dashboard

**Input**: Aucun

**Output**: `DashboardStatistics`
```java
public class DashboardStatistics {
    long totalOrders;
    long pendingOrders;
    long completedOrders;
    Money totalRevenue;
    long totalUsers;
    long totalProducts;
    long lowStockProducts;
}
```

**Dépendances**:
- `OrderRepository`
- `UserRepository`
- `ProductRepository`

---

#### 2. GetRevenueStatisticsUseCase
**Responsabilité**: Récupérer les statistiques de revenus

**Input**: `GetRevenueStatisticsQuery`
```java
public class GetRevenueStatisticsQuery {
    LocalDate startDate;
    LocalDate endDate;
    String period; // DAILY, WEEKLY, MONTHLY
}
```

**Output**: `RevenueStatistics`

**Dépendances**:
- `OrderRepository`

---

#### 3. GetTopSellingProductsUseCase
**Responsabilité**: Récupérer les produits les plus vendus

**Input**: `int limit`

**Output**: `List<ProductSales>`

**Dépendances**:
- `OrderRepository`
- `ProductRepository`

---

#### 4. GetLowStockProductsUseCase
**Responsabilité**: Récupérer les produits en stock faible

**Input**: `int threshold`

**Output**: `List<Product>`

**Dépendances**:
- `ProductRepository`

---

#### 5. GetUserGrowthStatisticsUseCase
**Responsabilité**: Récupérer les statistiques de croissance des utilisateurs

**Input**: `GetUserGrowthQuery`
```java
public class GetUserGrowthQuery {
    LocalDate startDate;
    LocalDate endDate;
}
```

**Output**: `UserGrowthStatistics`

**Dépendances**:
- `UserRepository`

---

#### 6. GetAllUsersUseCase (Admin)
**Responsabilité**: Récupérer tous les utilisateurs (admin)

**Input**: `Pageable`

**Output**: `Page<User>`

**Dépendances**:
- `UserRepository`

---

#### 7. ToggleUserStatusUseCase (Admin)
**Responsabilité**: Activer/Désactiver un utilisateur

**Input**: `ToggleUserStatusCommand`
```java
public class ToggleUserStatusCommand {
    Long userId;
    Boolean active;
}
```

**Output**: `User`

**Dépendances**:
- `UserRepository`

---

#### 8. AssignRoleToUserUseCase (Admin)
**Responsabilité**: Assigner un rôle à un utilisateur

**Input**: `AssignRoleCommand`
```java
public class AssignRoleCommand {
    Long userId;
    String roleName;
}
```

**Output**: `User`

**Dépendances**:
- `UserRepository`
- `RoleRepository`

---

## Ports (Interfaces)

Les ports sont des interfaces qui définissent les contrats entre l'application et l'infrastructure.

### Output Ports (Infrastructure → Application)

**Localisation**: `application/port/output/`

#### 1. EmailPort

```java
public interface EmailPort {
    void sendEmail(String to, String subject, String body);
    void sendHtmlEmail(String to, String subject, String htmlContent);
    void sendWelcomeEmail(String to, String username);
    void sendPasswordResetEmail(String to, String resetToken);
    void sendOrderConfirmationEmail(String to, String orderNumber);
    void sendOrderShippedEmail(String to, String orderNumber, String trackingNumber);
    void sendOrderDeliveredEmail(String to, String orderNumber);
}
```

---

#### 2. FileStoragePort

```java
public interface FileStoragePort {
    String storeFile(MultipartFile file, String directory);
    void deleteFile(String filePath);
    boolean fileExists(String filePath);
    String getFileUrl(String filePath);
}
```

---

#### 3. PaymentPort

```java
public interface PaymentPort {
    PaymentResult processPayment(PaymentRequest request);
    PaymentStatus checkPaymentStatus(String transactionId);
    void refundPayment(String transactionId, double amount);

    // DTOs
    record PaymentRequest(
        double amount,
        String currency,
        String customerEmail,
        String orderId,
        PaymentMethod method,
        Map<String, String> metadata
    ) {}

    record PaymentResult(
        String transactionId,
        PaymentStatus status,
        String message
    ) {}
}
```

---

#### 4. TokenPort (JWT)

```java
public interface TokenPort {
    String generateToken(String email, List<String> roles);
    String extractUsername(String token);
    boolean validateToken(String token);
    Date getExpirationDate(String token);
}
```

---

#### 5. PasswordEncoderPort

```java
public interface PasswordEncoderPort {
    String encode(String rawPassword);
    boolean matches(String rawPassword, String encodedPassword);
}
```

---

## Résumé

### Statistiques du projet

- **Entités du domaine**: 11
  - Product, Order, OrderItem, User, Cart, CartItem, Category, Address, Role, ProductImage, Payment

- **Value Objects**: 15+
  - Money, Email, PhoneNumber, Password, PersonalInfo, ShippingInfo, PasswordResetToken
  - IDs: ProductId, OrderId, UserId, CartId, CategoryId, AddressId, etc.
  - Enums: OrderStatus, PaymentStatus, PaymentMethod

- **Use Cases**: 60+
  - Product: 13 use cases
  - Order: 9 use cases
  - User/Auth: 12 use cases
  - Cart: 6 use cases
  - Category: 7 use cases
  - Address: 6 use cases
  - Payment: 5 use cases
  - Admin: 8 use cases

- **Ports**: 5
  - EmailPort, FileStoragePort, PaymentPort, TokenPort, PasswordEncoderPort

---

## Ordre de migration recommandé

1. **Phase 1**: Category (simple, peu de dépendances)
2. **Phase 2**: Product (dépend de Category)
3. **Phase 3**: User & Role & Auth (critique)
4. **Phase 4**: Cart (dépend de Product et User)
5. **Phase 5**: Address (dépend de User)
6. **Phase 6**: Order (dépend de Product, User, Cart, Address)
7. **Phase 7**: Payment (dépend de Order)
8. **Phase 8**: Admin/Statistics (dépend de tout)

---

## Notes importantes

1. **Commands vs Queries**: Séparer les opérations d'écriture (Commands) des lectures (Queries) pour suivre le pattern CQRS si souhaité.

2. **Validation**: Toute la validation métier doit être dans les entités du domaine ou les use cases, pas dans les controllers.

3. **Transactions**: Les use cases doivent être annotés avec `@Transactional` là où nécessaire.

4. **Exceptions**: Créer des exceptions métier spécifiques dans `domain/exception/`.

5. **Mappers**: Créer des mappers pour:
   - Domain ↔ JPA Entity (`infrastructure/persistence/mapper/`)
   - Domain ↔ REST DTO (`presentation/mapper/`)
   - Request/Command ↔ Domain (`presentation/mapper/`)

6. **Tests**: Chaque use case doit avoir des tests unitaires avec des mocks.

7. **Documentation**: Documenter chaque use case avec JavaDoc expliquant son but, ses contraintes et ses effets de bord.

---

Bonne chance avec l'implémentation de votre Clean Architecture !
