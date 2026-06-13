# Structure Complète du Projet - Clean Architecture

Structure détaillée de votre projet TKD organisé en Clean Architecture.

---

## Vue d'ensemble des couches

```
src/main/java/com/tdk/backend_tdk/
│
├── domain/              ← COUCHE 1: Entités & Logique Métier (Cœur)
├── application/         ← COUCHE 2: Use Cases & Ports
├── infrastructure/      ← COUCHE 3: Adapters (DB, Email, Payment, etc.)
├── presentation/        ← COUCHE 3: Adapters (REST API)
└── config/             ← COUCHE 4: Configuration Spring Boot
```

---

## Structure Complète

```
src/
├── main/
│   ├── java/
│   │   └── com/
│   │       └── tdk/
│   │           └── backend_tdk/
│   │               │
│   │               ├── domain/                                    # ← COUCHE 1: DOMAIN
│   │               │   │
│   │               │   ├── entity/                               # Entités du domaine
│   │               │   │   ├── Product.java
│   │               │   │   ├── Order.java
│   │               │   │   ├── OrderItem.java
│   │               │   │   ├── User.java
│   │               │   │   ├── Cart.java
│   │               │   │   ├── CartItem.java
│   │               │   │   ├── Category.java
│   │               │   │   ├── Address.java
│   │               │   │   ├── Role.java
│   │               │   │   ├── ProductImage.java
│   │               │   │   └── Payment.java
│   │               │   │
│   │               │   ├── valueobject/                          # Value Objects
│   │               │   │   ├── Money.java
│   │               │   │   ├── Email.java
│   │               │   │   ├── PhoneNumber.java
│   │               │   │   ├── Password.java
│   │               │   │   ├── PersonalInfo.java
│   │               │   │   ├── ShippingInfo.java
│   │               │   │   ├── PasswordResetToken.java
│   │               │   │   ├── Username.java
│   │               │   │   │
│   │               │   │   ├── ids/                              # Identifiants (Value Objects)
│   │               │   │   │   ├── ProductId.java
│   │               │   │   │   ├── OrderId.java
│   │               │   │   │   ├── UserId.java
│   │               │   │   │   ├── CartId.java
│   │               │   │   │   ├── CategoryId.java
│   │               │   │   │   ├── AddressId.java
│   │               │   │   │   ├── RoleId.java
│   │               │   │   │   ├── ProductImageId.java
│   │               │   │   │   ├── PaymentId.java
│   │               │   │   │   ├── OrderItemId.java
│   │               │   │   │   └── CartItemId.java
│   │               │   │   │
│   │               │   │   └── enums/                            # Enums (Value Objects)
│   │               │   │       ├── OrderStatus.java
│   │               │   │       ├── PaymentStatus.java
│   │               │   │       ├── PaymentMethod.java
│   │               │   │       └── RoleEnum.java
│   │               │   │
│   │               │   ├── repository/                           # Ports Repository (Interfaces)
│   │               │   │   ├── ProductRepository.java
│   │               │   │   ├── OrderRepository.java
│   │               │   │   ├── UserRepository.java
│   │               │   │   ├── CartRepository.java
│   │               │   │   ├── CategoryRepository.java
│   │               │   │   ├── AddressRepository.java
│   │               │   │   ├── RoleRepository.java
│   │               │   │   ├── ProductImageRepository.java
│   │               │   │   ├── PaymentRepository.java
│   │               │   │   ├── OrderItemRepository.java
│   │               │   │   └── CartItemRepository.java
│   │               │   │
│   │               │   ├── service/                              # Services du domaine
│   │               │   │   ├── OrderValidationService.java
│   │               │   │   ├── PricingService.java
│   │               │   │   ├── StockManagementService.java
│   │               │   │   └── DiscountCalculationService.java
│   │               │   │
│   │               │   └── exception/                            # Exceptions métier
│   │               │       ├── ProductNotFoundException.java
│   │               │       ├── OrderNotFoundException.java
│   │               │       ├── UserNotFoundException.java
│   │               │       ├── CategoryNotFoundException.java
│   │               │       ├── AddressNotFoundException.java
│   │               │       ├── CartNotFoundException.java
│   │               │       ├── InvalidDiscountException.java
│   │               │       ├── InsufficientStockException.java
│   │               │       ├── EmptyCartException.java
│   │               │       ├── InvalidOrderException.java
│   │               │       ├── InvalidPriceException.java
│   │               │       ├── InvalidProductException.java
│   │               │       ├── PaymentFailedException.java
│   │               │       ├── OrderCannotBeCancelledException.java
│   │               │       ├── EmailAlreadyExistsException.java
│   │               │       ├── UsernameAlreadyExistsException.java
│   │               │       ├── InvalidPasswordException.java
│   │               │       └── UnauthorizedException.java
│   │               │
│   │               ├── application/                              # ← COUCHE 2: APPLICATION
│   │               │   │
│   │               │   ├── usecase/                              # Use Cases
│   │               │   │   │
│   │               │   │   ├── product/                          # Module Product
│   │               │   │   │   ├── CreateProductUseCase.java
│   │               │   │   │   ├── UpdateProductUseCase.java
│   │               │   │   │   ├── GetProductUseCase.java
│   │               │   │   │   ├── GetAllProductsUseCase.java
│   │               │   │   │   ├── GetActiveProductsUseCase.java
│   │               │   │   │   ├── GetProductsByCategoryUseCase.java
│   │               │   │   │   ├── SearchProductsUseCase.java
│   │               │   │   │   ├── GetNewProductsUseCase.java
│   │               │   │   │   ├── GetProductsOnSaleUseCase.java
│   │               │   │   │   ├── DeleteProductUseCase.java
│   │               │   │   │   ├── AddProductImageUseCase.java
│   │               │   │   │   ├── DeleteProductImageUseCase.java
│   │               │   │   │   ├── UpdateProductStockUseCase.java
│   │               │   │   │   ├── ActivateProductUseCase.java
│   │               │   │   │   ├── DeactivateProductUseCase.java
│   │               │   │   │   │
│   │               │   │   │   ├── command/                      # Commands
│   │               │   │   │   │   ├── CreateProductCommand.java
│   │               │   │   │   │   ├── UpdateProductCommand.java
│   │               │   │   │   │   ├── AddProductImageCommand.java
│   │               │   │   │   │   └── UpdateProductStockCommand.java
│   │               │   │   │   │
│   │               │   │   │   └── query/                        # Queries
│   │               │   │   │       ├── GetProductsByCategoryQuery.java
│   │               │   │   │       └── SearchProductsQuery.java
│   │               │   │   │
│   │               │   │   ├── order/                            # Module Order
│   │               │   │   │   ├── CreateOrderUseCase.java
│   │               │   │   │   ├── GetOrderUseCase.java
│   │               │   │   │   ├── GetOrderByNumberUseCase.java
│   │               │   │   │   ├── GetUserOrdersUseCase.java
│   │               │   │   │   ├── CancelOrderUseCase.java
│   │               │   │   │   ├── ConfirmOrderUseCase.java
│   │               │   │   │   ├── ShipOrderUseCase.java
│   │               │   │   │   ├── DeliverOrderUseCase.java
│   │               │   │   │   ├── GetAllOrdersUseCase.java
│   │               │   │   │   ├── UpdateOrderStatusUseCase.java
│   │               │   │   │   │
│   │               │   │   │   ├── command/
│   │               │   │   │   │   ├── CreateOrderCommand.java
│   │               │   │   │   │   ├── CancelOrderCommand.java
│   │               │   │   │   │   ├── ShipOrderCommand.java
│   │               │   │   │   │   └── UpdateOrderStatusCommand.java
│   │               │   │   │   │
│   │               │   │   │   └── query/
│   │               │   │   │       ├── GetOrderQuery.java
│   │               │   │   │       ├── GetOrderByNumberQuery.java
│   │               │   │   │       ├── GetUserOrdersQuery.java
│   │               │   │   │       └── GetAllOrdersQuery.java
│   │               │   │   │
│   │               │   │   ├── auth/                             # Module Auth
│   │               │   │   │   ├── RegisterUserUseCase.java
│   │               │   │   │   ├── LoginUseCase.java
│   │               │   │   │   ├── VerifyEmailUseCase.java
│   │               │   │   │   ├── ForgotPasswordUseCase.java
│   │               │   │   │   ├── ResetPasswordUseCase.java
│   │               │   │   │   ├── ResendVerificationEmailUseCase.java
│   │               │   │   │   ├── RefreshTokenUseCase.java
│   │               │   │   │   ├── LogoutUseCase.java
│   │               │   │   │   │
│   │               │   │   │   ├── command/
│   │               │   │   │   │   ├── RegisterUserCommand.java
│   │               │   │   │   │   ├── LoginCommand.java
│   │               │   │   │   │   ├── ForgotPasswordCommand.java
│   │               │   │   │   │   └── ResetPasswordCommand.java
│   │               │   │   │   │
│   │               │   │   │   └── result/
│   │               │   │   │       ├── AuthToken.java
│   │               │   │   │       ├── RegisterResult.java
│   │               │   │   │       └── MessageResult.java
│   │               │   │   │
│   │               │   │   ├── user/                             # Module User
│   │               │   │   │   ├── GetUserProfileUseCase.java
│   │               │   │   │   ├── UpdateUserProfileUseCase.java
│   │               │   │   │   ├── ChangePasswordUseCase.java
│   │               │   │   │   ├── UpdateAvatarUseCase.java
│   │               │   │   │   ├── DeleteUserAccountUseCase.java
│   │               │   │   │   ├── GetAllUsersUseCase.java
│   │               │   │   │   ├── ToggleUserStatusUseCase.java
│   │               │   │   │   ├── AssignRoleToUserUseCase.java
│   │               │   │   │   │
│   │               │   │   │   └── command/
│   │               │   │   │       ├── UpdateUserProfileCommand.java
│   │               │   │   │       ├── ChangePasswordCommand.java
│   │               │   │   │       ├── UpdateAvatarCommand.java
│   │               │   │   │       ├── DeleteUserAccountCommand.java
│   │               │   │   │       ├── ToggleUserStatusCommand.java
│   │               │   │   │       └── AssignRoleCommand.java
│   │               │   │   │
│   │               │   │   ├── cart/                             # Module Cart
│   │               │   │   │   ├── GetOrCreateCartUseCase.java
│   │               │   │   │   ├── AddToCartUseCase.java
│   │               │   │   │   ├── UpdateCartItemUseCase.java
│   │               │   │   │   ├── RemoveFromCartUseCase.java
│   │               │   │   │   ├── ClearCartUseCase.java
│   │               │   │   │   ├── GetCartTotalUseCase.java
│   │               │   │   │   │
│   │               │   │   │   └── command/
│   │               │   │   │       ├── AddToCartCommand.java
│   │               │   │   │       ├── UpdateCartItemCommand.java
│   │               │   │   │       └── RemoveFromCartCommand.java
│   │               │   │   │
│   │               │   │   ├── category/                         # Module Category
│   │               │   │   │   ├── CreateCategoryUseCase.java
│   │               │   │   │   ├── UpdateCategoryUseCase.java
│   │               │   │   │   ├── DeleteCategoryUseCase.java
│   │               │   │   │   ├── GetCategoryUseCase.java
│   │               │   │   │   ├── GetAllCategoriesUseCase.java
│   │               │   │   │   ├── GetRootCategoriesUseCase.java
│   │               │   │   │   ├── GetSubCategoriesUseCase.java
│   │               │   │   │   │
│   │               │   │   │   └── command/
│   │               │   │   │       ├── CreateCategoryCommand.java
│   │               │   │   │       └── UpdateCategoryCommand.java
│   │               │   │   │
│   │               │   │   ├── address/                          # Module Address
│   │               │   │   │   ├── CreateAddressUseCase.java
│   │               │   │   │   ├── UpdateAddressUseCase.java
│   │               │   │   │   ├── DeleteAddressUseCase.java
│   │               │   │   │   ├── GetAddressUseCase.java
│   │               │   │   │   ├── GetUserAddressesUseCase.java
│   │               │   │   │   ├── SetDefaultAddressUseCase.java
│   │               │   │   │   │
│   │               │   │   │   ├── command/
│   │               │   │   │   │   ├── CreateAddressCommand.java
│   │               │   │   │   │   ├── UpdateAddressCommand.java
│   │               │   │   │   │   ├── DeleteAddressCommand.java
│   │               │   │   │   │   └── SetDefaultAddressCommand.java
│   │               │   │   │   │
│   │               │   │   │   └── query/
│   │               │   │   │       └── GetAddressQuery.java
│   │               │   │   │
│   │               │   │   ├── payment/                          # Module Payment
│   │               │   │   │   ├── ProcessStripePaymentUseCase.java
│   │               │   │   │   ├── ProcessMtnPaymentUseCase.java
│   │               │   │   │   ├── ProcessOrangeMoneyPaymentUseCase.java
│   │               │   │   │   ├── CheckPaymentStatusUseCase.java
│   │               │   │   │   ├── RefundPaymentUseCase.java
│   │               │   │   │   │
│   │               │   │   │   └── command/
│   │               │   │   │       ├── ProcessStripePaymentCommand.java
│   │               │   │   │       ├── ProcessMtnPaymentCommand.java
│   │               │   │   │       ├── ProcessOrangeMoneyPaymentCommand.java
│   │               │   │   │       └── RefundPaymentCommand.java
│   │               │   │   │
│   │               │   │   └── admin/                            # Module Admin
│   │               │   │       ├── GetDashboardStatisticsUseCase.java
│   │               │   │       ├── GetRevenueStatisticsUseCase.java
│   │               │   │       ├── GetTopSellingProductsUseCase.java
│   │               │   │       ├── GetLowStockProductsUseCase.java
│   │               │   │       ├── GetUserGrowthStatisticsUseCase.java
│   │               │   │       │
│   │               │   │       ├── query/
│   │               │   │       │   ├── GetRevenueStatisticsQuery.java
│   │               │   │       │   └── GetUserGrowthQuery.java
│   │               │   │       │
│   │               │   │       └── result/
│   │               │   │           ├── DashboardStatistics.java
│   │               │   │           ├── RevenueStatistics.java
│   │               │   │           ├── ProductSales.java
│   │               │   │           └── UserGrowthStatistics.java
│   │               │   │
│   │               │   └── port/                                 # Ports (Interfaces)
│   │               │       │
│   │               │       ├── output/                           # Output Ports
│   │               │       │   ├── EmailPort.java
│   │               │       │   ├── FileStoragePort.java
│   │               │       │   ├── PaymentPort.java
│   │               │       │   ├── TokenPort.java
│   │               │       │   └── PasswordEncoderPort.java
│   │               │       │
│   │               │       └── input/                            # Input Ports (optionnel)
│   │               │           └── (interfaces de use cases si nécessaire)
│   │               │
│   │               ├── infrastructure/                           # ← COUCHE 3: INFRASTRUCTURE
│   │               │   │
│   │               │   ├── persistence/                          # Persistance (Database)
│   │               │   │   │
│   │               │   │   ├── entity/                           # JPA Entities
│   │               │   │   │   ├── ProductJpaEntity.java
│   │               │   │   │   ├── OrderJpaEntity.java
│   │               │   │   │   ├── OrderItemJpaEntity.java
│   │               │   │   │   ├── UserJpaEntity.java
│   │               │   │   │   ├── CartJpaEntity.java
│   │               │   │   │   ├── CartItemJpaEntity.java
│   │               │   │   │   ├── CategoryJpaEntity.java
│   │               │   │   │   ├── AddressJpaEntity.java
│   │               │   │   │   ├── RoleJpaEntity.java
│   │               │   │   │   ├── ProductImageJpaEntity.java
│   │               │   │   │   ├── PaymentJpaEntity.java
│   │               │   │   │   └── MobilePaymentJpaEntity.java
│   │               │   │   │
│   │               │   │   ├── repository/                       # Repository Implementations
│   │               │   │   │   │
│   │               │   │   │   ├── jpa/                          # Spring Data JPA Repositories
│   │               │   │   │   │   ├── ProductJpaRepository.java
│   │               │   │   │   │   ├── OrderJpaRepository.java
│   │               │   │   │   │   ├── OrderItemJpaRepository.java
│   │               │   │   │   │   ├── UserJpaRepository.java
│   │               │   │   │   │   ├── CartJpaRepository.java
│   │               │   │   │   │   ├── CartItemJpaRepository.java
│   │               │   │   │   │   ├── CategoryJpaRepository.java
│   │               │   │   │   │   ├── AddressJpaRepository.java
│   │               │   │   │   │   ├── RoleJpaRepository.java
│   │               │   │   │   │   ├── ProductImageJpaRepository.java
│   │               │   │   │   │   └── PaymentJpaRepository.java
│   │               │   │   │   │
│   │               │   │   │   └── impl/                         # Adapters (Implementations)
│   │               │   │   │       ├── ProductRepositoryImpl.java
│   │               │   │   │       ├── OrderRepositoryImpl.java
│   │               │   │   │       ├── OrderItemRepositoryImpl.java
│   │               │   │   │       ├── UserRepositoryImpl.java
│   │               │   │   │       ├── CartRepositoryImpl.java
│   │               │   │   │       ├── CartItemRepositoryImpl.java
│   │               │   │   │       ├── CategoryRepositoryImpl.java
│   │               │   │   │       ├── AddressRepositoryImpl.java
│   │               │   │   │       ├── RoleRepositoryImpl.java
│   │               │   │   │       ├── ProductImageRepositoryImpl.java
│   │               │   │   │       └── PaymentRepositoryImpl.java
│   │               │   │   │
│   │               │   │   └── mapper/                           # Mappers (Domain ↔ JPA)
│   │               │   │       ├── ProductMapper.java
│   │               │   │       ├── OrderMapper.java
│   │               │   │       ├── OrderItemMapper.java
│   │               │   │       ├── UserMapper.java
│   │               │   │       ├── CartMapper.java
│   │               │   │       ├── CartItemMapper.java
│   │               │   │       ├── CategoryMapper.java
│   │               │   │       ├── AddressMapper.java
│   │               │   │       ├── RoleMapper.java
│   │               │   │       ├── ProductImageMapper.java
│   │               │   │       └── PaymentMapper.java
│   │               │   │
│   │               │   ├── email/                                # Email Adapter
│   │               │   │   ├── EmailAdapter.java
│   │               │   │   └── template/                         # Email Templates
│   │               │   │       ├── WelcomeEmailTemplate.java
│   │               │   │       ├── PasswordResetEmailTemplate.java
│   │               │   │       ├── OrderConfirmationEmailTemplate.java
│   │               │   │       └── OrderShippedEmailTemplate.java
│   │               │   │
│   │               │   ├── payment/                              # Payment Adapters
│   │               │   │   ├── StripePaymentAdapter.java
│   │               │   │   ├── MtnMoneyAdapter.java
│   │               │   │   ├── OrangeMoneyAdapter.java
│   │               │   │   │
│   │               │   │   ├── dto/                              # DTOs spécifiques aux providers
│   │               │   │   │   ├── StripePaymentRequest.java
│   │               │   │   │   ├── MtnPaymentRequest.java
│   │               │   │   │   └── OrangePaymentRequest.java
│   │               │   │   │
│   │               │   │   └── mapper/
│   │               │   │       ├── StripePaymentMapper.java
│   │               │   │       ├── MtnPaymentMapper.java
│   │               │   │       └── OrangePaymentMapper.java
│   │               │   │
│   │               │   ├── storage/                              # File Storage
│   │               │   │   ├── LocalFileStorageAdapter.java
│   │               │   │   └── FileStorageProperties.java
│   │               │   │
│   │               │   └── security/                             # Security (JWT, Password)
│   │               │       ├── JwtTokenProvider.java
│   │               │       ├── JwtAuthenticationFilter.java
│   │               │       ├── JwtAuthenticationEntryPoint.java
│   │               │       ├── PasswordEncoderAdapter.java
│   │               │       └── CustomUserDetailsService.java
│   │               │
│   │               ├── presentation/                             # ← COUCHE 3: PRESENTATION (REST API)
│   │               │   │
│   │               │   ├── rest/                                 # Controllers REST
│   │               │   │   ├── ProductController.java
│   │               │   │   ├── OrderController.java
│   │               │   │   ├── AuthController.java
│   │               │   │   ├── UserController.java
│   │               │   │   ├── CartController.java
│   │               │   │   ├── CategoryController.java
│   │               │   │   ├── AddressController.java
│   │               │   │   ├── PaymentController.java
│   │               │   │   ├── MobilePaymentController.java
│   │               │   │   ├── AdminController.java
│   │               │   │   ├── AdminDashboardController.java
│   │               │   │   ├── FileUploadController.java
│   │               │   │   └── RoleController.java
│   │               │   │
│   │               │   ├── dto/                                  # DTOs REST
│   │               │   │   │
│   │               │   │   ├── request/                          # Request DTOs
│   │               │   │   │   │
│   │               │   │   │   ├── product/
│   │               │   │   │   │   ├── CreateProductRequest.java
│   │               │   │   │   │   └── UpdateProductRequest.java
│   │               │   │   │   │
│   │               │   │   │   ├── order/
│   │               │   │   │   │   ├── CreateOrderRequest.java
│   │               │   │   │   │   ├── UpdateOrderStatusRequest.java
│   │               │   │   │   │   └── CancelOrderRequest.java
│   │               │   │   │   │
│   │               │   │   │   ├── auth/
│   │               │   │   │   │   ├── RegisterRequest.java
│   │               │   │   │   │   ├── LoginRequest.java
│   │               │   │   │   │   ├── ForgotPasswordRequest.java
│   │               │   │   │   │   └── ResetPasswordRequest.java
│   │               │   │   │   │
│   │               │   │   │   ├── user/
│   │               │   │   │   │   ├── UpdateUserProfileRequest.java
│   │               │   │   │   │   └── ChangePasswordRequest.java
│   │               │   │   │   │
│   │               │   │   │   ├── cart/
│   │               │   │   │   │   ├── AddToCartRequest.java
│   │               │   │   │   │   └── UpdateCartItemRequest.java
│   │               │   │   │   │
│   │               │   │   │   ├── category/
│   │               │   │   │   │   ├── CreateCategoryRequest.java
│   │               │   │   │   │   └── UpdateCategoryRequest.java
│   │               │   │   │   │
│   │               │   │   │   ├── address/
│   │               │   │   │   │   ├── CreateAddressRequest.java
│   │               │   │   │   │   └── UpdateAddressRequest.java
│   │               │   │   │   │
│   │               │   │   │   └── payment/
│   │               │   │   │       ├── StripePaymentRequest.java
│   │               │   │   │       ├── MtnPaymentRequest.java
│   │               │   │   │       └── OrangeMoneyPaymentRequest.java
│   │               │   │   │
│   │               │   │   └── response/                         # Response DTOs
│   │               │   │       │
│   │               │   │       ├── product/
│   │               │   │       │   ├── ProductResponse.java
│   │               │   │       │   └── ProductImageResponse.java
│   │               │   │       │
│   │               │   │       ├── order/
│   │               │   │       │   ├── OrderResponse.java
│   │               │   │       │   └── OrderItemResponse.java
│   │               │   │       │
│   │               │   │       ├── auth/
│   │               │   │       │   ├── AuthResponse.java
│   │               │   │       │   └── MessageResponse.java
│   │               │   │       │
│   │               │   │       ├── user/
│   │               │   │       │   └── UserResponse.java
│   │               │   │       │
│   │               │   │       ├── cart/
│   │               │   │       │   ├── CartResponse.java
│   │               │   │       │   └── CartItemResponse.java
│   │               │   │       │
│   │               │   │       ├── category/
│   │               │   │       │   └── CategoryResponse.java
│   │               │   │       │
│   │               │   │       ├── address/
│   │               │   │       │   └── AddressResponse.java
│   │               │   │       │
│   │               │   │       ├── payment/
│   │               │   │       │   └── PaymentResponse.java
│   │               │   │       │
│   │               │   │       └── admin/
│   │               │   │           ├── DashboardStatsResponse.java
│   │               │   │           └── RevenueStatsResponse.java
│   │               │   │
│   │               │   ├── mapper/                               # Mappers REST
│   │               │   │   ├── ProductRestMapper.java
│   │               │   │   ├── OrderRestMapper.java
│   │               │   │   ├── UserRestMapper.java
│   │               │   │   ├── CartRestMapper.java
│   │               │   │   ├── CategoryRestMapper.java
│   │               │   │   ├── AddressRestMapper.java
│   │               │   │   └── PaymentRestMapper.java
│   │               │   │
│   │               │   └── exception/                            # Exception Handlers
│   │               │       ├── GlobalExceptionHandler.java
│   │               │       ├── ErrorResponse.java
│   │               │       └── ValidationErrorResponse.java
│   │               │
│   │               ├── config/                                   # ← COUCHE 4: CONFIGURATION
│   │               │   ├── SecurityConfig.java
│   │               │   ├── PersistenceConfig.java
│   │               │   ├── UseCaseConfig.java
│   │               │   ├── WebConfig.java
│   │               │   ├── CorsConfig.java
│   │               │   ├── AsyncConfig.java
│   │               │   ├── EmailConfig.java
│   │               │   ├── FileStorageConfig.java
│   │               │   ├── PaymentConfig.java
│   │               │   └── SwaggerConfig.java
│   │               │
│   │               └── BackendTdkApplication.java                # Main Application
│   │
│   └── resources/
│       ├── application.properties                                # Configuration principale
│       ├── application-dev.properties                            # Config développement
│       ├── application-prod.properties                           # Config production
│       ├── application-test.properties                           # Config tests
│       │
│       ├── db/
│       │   └── migration/                                        # Scripts Flyway/Liquibase
│       │       ├── V1__create_initial_schema.sql
│       │       ├── V2__add_product_images.sql
│       │       └── V3__add_payment_tables.sql
│       │
│       ├── static/                                               # Fichiers statiques
│       │   └── images/
│       │
│       └── templates/                                            # Templates email (Thymeleaf)
│           ├── email/
│           │   ├── welcome.html
│           │   ├── password-reset.html
│           │   ├── order-confirmation.html
│           │   └── order-shipped.html
│           └── error/
│               ├── 404.html
│               └── 500.html
│
└── test/
    └── java/
        └── com/
            └── tdk/
                └── backend_tdk/
                    │
                    ├── domain/                                   # Tests du Domain
                    │   ├── entity/
                    │   │   ├── ProductTest.java
                    │   │   ├── OrderTest.java
                    │   │   ├── CartTest.java
                    │   │   └── UserTest.java
                    │   │
                    │   ├── valueobject/
                    │   │   ├── MoneyTest.java
                    │   │   ├── EmailTest.java
                    │   │   └── PhoneNumberTest.java
                    │   │
                    │   └── service/
                    │       └── OrderValidationServiceTest.java
                    │
                    ├── application/                              # Tests des Use Cases
                    │   └── usecase/
                    │       ├── product/
                    │       │   ├── CreateProductUseCaseTest.java
                    │       │   ├── UpdateProductUseCaseTest.java
                    │       │   └── DeleteProductUseCaseTest.java
                    │       │
                    │       ├── order/
                    │       │   ├── CreateOrderUseCaseTest.java
                    │       │   └── CancelOrderUseCaseTest.java
                    │       │
                    │       ├── auth/
                    │       │   ├── RegisterUserUseCaseTest.java
                    │       │   └── LoginUseCaseTest.java
                    │       │
                    │       └── cart/
                    │           └── AddToCartUseCaseTest.java
                    │
                    ├── infrastructure/                           # Tests Infrastructure
                    │   ├── persistence/
                    │   │   ├── repository/
                    │   │   │   ├── ProductRepositoryImplTest.java
                    │   │   │   └── OrderRepositoryImplTest.java
                    │   │   │
                    │   │   └── mapper/
                    │   │       └── ProductMapperTest.java
                    │   │
                    │   ├── email/
                    │   │   └── EmailAdapterTest.java
                    │   │
                    │   └── payment/
                    │       └── StripePaymentAdapterTest.java
                    │
                    ├── presentation/                             # Tests Controllers
                    │   ├── rest/
                    │   │   ├── ProductControllerTest.java
                    │   │   ├── OrderControllerTest.java
                    │   │   └── AuthControllerTest.java
                    │   │
                    │   └── mapper/
                    │       └── ProductRestMapperTest.java
                    │
                    └── integration/                              # Tests d'intégration
                        ├── ProductIntegrationTest.java
                        ├── OrderIntegrationTest.java
                        ├── AuthIntegrationTest.java
                        └── CartIntegrationTest.java
```

---

## Détail des fichiers clés

### pom.xml (Racine du projet)

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.5.6</version>
    </parent>

    <groupId>com.tdk</groupId>
    <artifactId>backend-tdk</artifactId>
    <version>1.0.0</version>
    <name>TKD E-Commerce Backend</name>
    <description>E-commerce backend with Clean Architecture</description>

    <properties>
        <java.version>17</java.version>
    </properties>

    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-security</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-validation</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-mail</artifactId>
        </dependency>

        <!-- Database -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version>
        </dependency>

        <!-- JWT -->
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-api</artifactId>
            <version>0.11.5</version>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-impl</artifactId>
            <version>0.11.5</version>
            <scope>runtime</scope>
        </dependency>
        <dependency>
            <groupId>io.jsonwebtoken</groupId>
            <artifactId>jjwt-jackson</artifactId>
            <version>0.11.5</version>
            <scope>runtime</scope>
        </dependency>

        <!-- Stripe Payment -->
        <dependency>
            <groupId>com.stripe</groupId>
            <artifactId>stripe-java</artifactId>
            <version>24.0.0</version>
        </dependency>

        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>

        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.springframework.security</groupId>
            <artifactId>spring-security-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
            </plugin>
        </plugins>
    </build>
</project>
```

---

## application.properties

```properties
# Application
spring.application.name=TKD E-Commerce Backend
server.port=8080

# Database
spring.datasource.url=jdbc:mysql://localhost:3306/tdk_ecommerce?useSSL=false&serverTimezone=UTC
spring.datasource.username=root
spring.datasource.password=your_password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA/Hibernate
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.MySQL8Dialect
spring.jpa.properties.hibernate.format_sql=true

# JWT
jwt.secret=your-secret-key-here-make-it-very-long-and-secure
jwt.expiration=86400000

# Email (MailHog pour développement)
spring.mail.host=localhost
spring.mail.port=1025
spring.mail.username=
spring.mail.password=
spring.mail.properties.mail.smtp.auth=false
spring.mail.properties.mail.smtp.starttls.enable=false

# File Upload
spring.servlet.multipart.enabled=true
spring.servlet.multipart.max-file-size=10MB
spring.servlet.multipart.max-request-size=10MB
file.upload-dir=uploads/

# Frontend URL
app.frontend.url=http://localhost:4200

# Stripe
stripe.api.key=sk_test_your_stripe_key

# MTN Mobile Money
mtn.api.url=https://sandbox.momodeveloper.mtn.com
mtn.api.key=your_mtn_key
mtn.subscription.key=your_subscription_key

# Orange Money
orange.api.url=https://api.orange.com/orange-money-webpay/dev/v1
orange.merchant.key=your_orange_merchant_key
orange.merchant.secret=your_orange_merchant_secret

# CORS
cors.allowed.origins=http://localhost:4200
```

---

## Statistiques de la structure

### Répartition par couche:

| Couche | Packages | Classes estimées |
|--------|----------|------------------|
| **Domain** | 7 | ~60 (11 entités + 15 VOs + 11 repos + exceptions + services) |
| **Application** | 10 | ~80 (60+ use cases + commands/queries) |
| **Infrastructure** | 8 | ~50 (JPA entities + repos impl + adapters) |
| **Presentation** | 5 | ~40 (controllers + DTOs + mappers) |
| **Configuration** | 1 | ~10 (configs Spring) |
| **Tests** | 10 | ~100 (tests unitaires + intégration) |
| **TOTAL** | **41** | **~340 fichiers** |

---

## Flux de dépendances

```
Presentation (Controllers)
    ↓ dépend de
Application (Use Cases)
    ↓ dépend de
Domain (Entities + Repositories interfaces)
    ↑ implémenté par
Infrastructure (Repository Impl, Adapters)
```

**Règle d'or**: Les dépendances pointent toujours vers le domaine (vers l'intérieur).

---

## Prochaines étapes pour migration

1. **Créer la structure des packages** (vide)
2. **Migrer Category** (module le plus simple)
3. **Migrer Product** (dépend de Category)
4. **Migrer User & Auth** (critique)
5. **Migrer Cart** (dépend de Product et User)
6. **Migrer Address** (dépend de User)
7. **Migrer Order** (dépend de tout)
8. **Migrer Payment** (dépend de Order)
9. **Migrer Admin/Statistics** (en dernier)

---

## Commandes pour créer la structure

### Linux/Mac:
```bash
# Depuis la racine src/main/java/com/tdk/backend_tdk/
mkdir -p domain/{entity,valueobject/{ids,enums},repository,service,exception}
mkdir -p application/{usecase/{product/{command,query},order/{command,query},auth/{command,result},user/command,cart/command,category/command,address/{command,query},payment/command,admin/{query,result}},port/{output,input}}
mkdir -p infrastructure/{persistence/{entity,repository/{jpa,impl},mapper},email/template,payment/{dto,mapper},storage,security}
mkdir -p presentation/{rest,dto/{request/{product,order,auth,user,cart,category,address,payment},response/{product,order,auth,user,cart,category,address,payment,admin}},mapper,exception}
mkdir -p config
```

### Windows (PowerShell):
```powershell
# Depuis src\main\java\com\tdk\backend_tdk\
New-Item -ItemType Directory -Force -Path domain\entity
New-Item -ItemType Directory -Force -Path domain\valueobject\ids
New-Item -ItemType Directory -Force -Path domain\valueobject\enums
New-Item -ItemType Directory -Force -Path domain\repository
New-Item -ItemType Directory -Force -Path domain\service
New-Item -ItemType Directory -Force -Path domain\exception
New-Item -ItemType Directory -Force -Path application\usecase\product\command
New-Item -ItemType Directory -Force -Path application\usecase\product\query
New-Item -ItemType Directory -Force -Path application\usecase\order\command
New-Item -ItemType Directory -Force -Path application\usecase\order\query
New-Item -ItemType Directory -Force -Path application\usecase\auth\command
New-Item -ItemType Directory -Force -Path application\usecase\auth\result
New-Item -ItemType Directory -Force -Path application\port\output
New-Item -ItemType Directory -Force -Path infrastructure\persistence\entity
New-Item -ItemType Directory -Force -Path infrastructure\persistence\repository\jpa
New-Item -ItemType Directory -Force -Path infrastructure\persistence\repository\impl
New-Item -ItemType Directory -Force -Path infrastructure\persistence\mapper
New-Item -ItemType Directory -Force -Path infrastructure\email
New-Item -ItemType Directory -Force -Path infrastructure\payment
New-Item -ItemType Directory -Force -Path infrastructure\storage
New-Item -ItemType Directory -Force -Path infrastructure\security
New-Item -ItemType Directory -Force -Path presentation\rest
New-Item -ItemType Directory -Force -Path presentation\dto\request
New-Item -ItemType Directory -Force -Path presentation\dto\response
New-Item -ItemType Directory -Force -Path presentation\mapper
New-Item -ItemType Directory -Force -Path presentation\exception
New-Item -ItemType Directory -Force -Path config
```

---

Vous avez maintenant la structure complète de votre projet Clean Architecture avec **340+ fichiers** organisés en **41 packages** !
