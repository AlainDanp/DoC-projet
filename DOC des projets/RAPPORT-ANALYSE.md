# Rapport d'Analyse Complet - TKD Frontend + Backend

**Date :** 2026-02-13
**Projet :** TKD E-commerce (Angular 19 + Spring Boot)
**Frontend :** `TKD_Front/` (Angular 19.2.0, NgModule, Tailwind CSS)
**Backend :** `backend_tdk/` (Spring Boot, JWT, Stripe, Mobile Money)

---

## Table des matieres

1. [Frontend - Problemes Critiques](#1-frontend---problemes-critiques)
2. [Frontend - Problemes Moderes](#2-frontend---problemes-moderes)
3. [Frontend - Problemes Mineurs](#3-frontend---problemes-mineurs)
4. [Backend - Securite Critique](#4-backend---securite-critique)
5. [Backend - Bugs Majeurs](#5-backend---bugs-majeurs)
6. [Backend - Problemes Moderes](#6-backend---problemes-moderes)
7. [Desalignement Frontend / Backend](#7-desalignement-frontend--backend)
8. [Resume](#8-resume)

---

## 1. Frontend - Problemes Critiques

### 1.1 Alias `@shared` non configure dans tsconfig

| | |
|---|---|
| **Fichier** | `src/app/shared/components/input/input.directive.ts:5` |
| **Severite** | CRITIQUE - Erreur de compilation |

```typescript
import { mergeClasses, transform } from '@shared/utils/merge-classes';
```

Le chemin `@shared` n'est defini nulle part dans `tsconfig.json` ou `tsconfig.app.json`. L'import echoue a la compilation.

**Correction :** Ajouter dans `tsconfig.json` :
```json
"paths": {
  "@shared/*": ["src/app/shared/*"]
}
```
Ou remplacer par un import relatif : `'../../utils/merge-classes'`

---

### 1.2 Typo dans le selector de ResetPassword

| | |
|---|---|
| **Fichier** | `src/app/components/reset-password/reset-password.ts:7` |
| **Severite** | CRITIQUE - Composant inaccessible par selector |

```typescript
selector: 'app-reset-passowrd'  // "passowrd" au lieu de "password"
```

---

### 1.3 Fuites memoire - Subscriptions jamais nettoyees

| | |
|---|---|
| **Severite** | CRITIQUE - Memory leaks en production |

Aucun des composants suivants n'implemente `OnDestroy` pour unsubscribe :

| Composant | Fichier | Nb subscriptions non nettoyees |
|-----------|---------|-------------------------------|
| OptionComponent | `src/app/components/option/option.component.ts` | 10+ |
| Checkout | `src/app/components/checkout/checkout.ts` | 3 |
| ResetPassword | `src/app/components/reset-password/reset-password.ts` | 1 (queryParams) |
| Login | `src/app/components/login/login.ts` | 2 |
| ForgotPassword | `src/app/components/forgot-password/forgot-password.ts` | 1 |

**Correction :** Ajouter `implements OnDestroy` et utiliser `takeUntil()` avec un `Subject` destroy :
```typescript
private destroy$ = new Subject<void>();

ngOnDestroy(): void {
  this.destroy$.next();
  this.destroy$.complete();
}

// Dans chaque subscribe :
this.service.getData().pipe(takeUntil(this.destroy$)).subscribe(...)
```

---

### 1.4 About ne declare pas `implements OnInit`

| | |
|---|---|
| **Fichier** | `src/app/components/about/about.ts:22` |
| **Severite** | CRITIQUE |

La classe `About` definit `ngOnInit()` (ligne 75) mais n'implemente pas l'interface `OnInit`.

---

### 1.5 Typos dans les modeles

| | |
|---|---|
| **Fichier** | `src/app/core/models/user.model.ts` |
| **Severite** | MODERE |

- Ligne 56 : `RestPasswordRequest` → devrait etre `ResetPasswordRequest`
- Ligne 52 : `ForgotPassWordRequest` → `W` majuscule inconsistant

---

### 1.6 Version Angular CLI incompatible

| | |
|---|---|
| **Fichier** | `package.json:34` |
| **Severite** | MODERE |

```json
"@angular/cli": "^21.1.2"
```

Angular 19.2.0 devrait utiliser Angular CLI 19.x. La version 21.x peut causer des incompatibilites.

---

## 2. Frontend - Problemes Moderes

### 2.1 Casing des routes inconsistant

| | |
|---|---|
| **Fichier** | `src/app/app-routing.module.ts` |

Mix de PascalCase et kebab-case dans les routes :

| Route actuelle | Convention recommandee |
|---------------|----------------------|
| `/Home` | `/home` |
| `/About` | `/about` |
| `/Shop` | `/shop` |
| `/FAQ` | `/faq` |
| `/contact-Support` | `/contact-support` |
| `/Option` | `/option` |
| `/Checkout` | `/checkout` |
| `/condition` | `/condition` (OK) |
| `/confidentiel` | `/confidentiel` (OK) |

---

### 2.2 Pas de PaymentService

| | |
|---|---|
| **Fichier** | `src/app/components/option/option.component.ts` |

Les methodes de paiement sont stockees uniquement en `localStorage` et jamais synchronisees avec le backend. Le backend a des endpoints de paiement (Stripe, Orange Money, MTN) mais aucun service frontend ne les consomme.

---

### 2.3 State management mixte dans AuthService

| | |
|---|---|
| **Fichier** | `src/app/core/services/auth.service.ts:23-27` |

```typescript
// Signal (Angular)
isLoggedIn = signal<boolean>(this.hasToken());

// BehaviorSubject (RxJS)
private currentUserSubject = new BehaviorSubject<User | null>(...);
public currentUser$ = this.currentUserSubject.asObservable();
```

Deux systemes de state management coexistent. Risque de desynchronisation.

---

### 2.4 Error interceptor incomplet

| | |
|---|---|
| **Fichier** | `src/app/core/interceptors/error-interceptor.ts` |

Seuls les codes 400, 401, 403, 404, 500 sont geres. Manquent : 502, 503, 504, timeout reseau.

---

## 3. Frontend - Problemes Mineurs

### 3.1 Code mort dans Shop

| | |
|---|---|
| **Fichier** | `src/app/components/shop/shop.ts:100-119` |

La methode `searchProducts()` est declaree mais jamais appelee. La recherche utilise `onSearchInput()` via debounce.

---

### 3.2 Switch sans default dans Checkout

| | |
|---|---|
| **Fichier** | `src/app/components/checkout/checkout.ts` |

Les methodes `getPaymentLabel()`, `getPaymentIcon()`, `getPaymentDesc()` n'ont pas de `default` case. TypeScript peut retourner `undefined` implicitement.

---

### 3.3 CartService - appel API silencieux dans le constructeur

| | |
|---|---|
| **Fichier** | `src/app/core/services/cart.service.ts:16-21` |

```typescript
constructor(private http: HttpClient) {
  if (localStorage.getItem('auth_token')) {
    this.loadCart();  // Erreurs avalees silencieusement
  }
}
```

---

## 4. Backend - Securite Critique

### 4.1 Credentials en dur dans application.properties

| | |
|---|---|
| **Fichier** | `src/main/resources/application.properties` |
| **Severite** | CRITIQUE |

```properties
# Ligne 6 - Mot de passe BDD expose
spring.datasource.password=Jeux@joneA56

# Ligne 36-37 - Cles Stripe exposees
stripe.api.key.secret=sk_test_51SaJkk...
stripe.api.key.publishable=pk_test_51SaJkk...

# Ligne 71 - JWT secret avec default previsible
jwt.secret=${JWT_SECRET:MyS3cr3tK3yF0rJWT!...}
```

**Correction :** Utiliser des variables d'environnement sans valeurs par defaut :
```properties
spring.datasource.password=${DB_PASSWORD}
stripe.api.key.secret=${STRIPE_SECRET_KEY}
jwt.secret=${JWT_SECRET}
```

---

### 4.2 Endpoint admin registration public

| | |
|---|---|
| **Fichier** | `controller/AdminRegistrationController.java:31` |
| **Severite** | CRITIQUE - Escalade de privileges |

```java
@PostMapping("/register-admin")  // sous /api/auth/** → permitAll !
```

L'endpoint `/api/auth/register-admin` est accessible sans authentification car `SecurityConfig` autorise `/api/auth/**`. **N'importe qui peut creer un compte admin.**

**Correction :** Deplacer sous `/api/admin/register` avec `@PreAuthorize("hasRole('ADMIN')")`, ou supprimer en production.

---

### 4.3 Console H2 exposee

| | |
|---|---|
| **Fichier** | `config/SecurityConfig.java:62` |
| **Severite** | CRITIQUE |

```java
.requestMatchers("/h2-console/**").permitAll()
```

Acces complet a la base de donnees sans authentification. Doit etre desactive en production.

**Correction :** Utiliser un profil Spring :
```java
// Seulement dans SecurityConfig-dev.java ou via @Profile("dev")
```

---

### 4.4 Webhook Stripe non implemente

| | |
|---|---|
| **Fichier** | `service/StripeService.java:234-237` |
| **Severite** | CRITIQUE |

```java
public void handleWebhook(String payload, String sigHeader) {
    // TODO: Implémenter la gestion des webhooks
}
```

Aucune validation de signature. Un attaquant peut forger des confirmations de paiement.

---

### 4.5 Risque de directory traversal

| | |
|---|---|
| **Fichier** | `controller/FileUploadController.java:24` |
| **Severite** | CRITIQUE |

```java
Resource resource = fileStorageService.LoadFileAsResource(folder + "/" + filename);
```

Concatenation directe d'inputs utilisateur. Un attaquant pourrait acceder a :
`/api/uploads/../../etc/passwd`

**Correction :** Valider et normaliser les chemins :
```java
Path filePath = uploadDir.resolve(folder).resolve(filename).normalize();
if (!filePath.startsWith(uploadDir)) {
    throw new BadRequestException("Invalid file path");
}
```

---

### 4.6 Typo content-type

| | |
|---|---|
| **Fichier** | `controller/FileUploadController.java:26` |

```java
String contentType = "appliction/octet-stream";  // "application" manque le 'a'
```

---

## 5. Backend - Bugs Majeurs

### 5.1 Enum MobilePaymentProvider dupliquee

| | |
|---|---|
| **Fichiers** | `model/MobilePaymentProvider.java` + `model/payement/MobilePaymentProvider.java` |
| **Severite** | MAJEUR |

Deux enums differentes avec le meme nom :
- `model/` → 4 valeurs : `ORANGE_MONEY, MTN_MOBILE_MONEY, STRIPE, CASH_ON_DELIVERY`
- `model/payement/` → 7 valeurs : ajoute `CREDIT_CARD, PAYPAL, BANK_TRANSFER`

Ambiguite selon l'import utilise.

**Correction :** Supprimer `model/MobilePaymentProvider.java` et garder uniquement celui dans `model/payement/`.

---

### 5.2 Collision de routes - Products

| | |
|---|---|
| **Fichier** | `controller/ProductController.java` |
| **Severite** | MAJEUR |

- Ligne 58 : `@GetMapping("/{id}")`
- Ligne 83 : `@GetMapping("/search")`

`/api/products/search` est interprete comme `/{id}` avec `id="search"` car `/{id}` est defini avant `/search`.

**Correction :** Placer `/search` AVANT `/{id}` dans le controller, ou utiliser `@GetMapping("/{id:\\d+}")` pour limiter aux numeriques.

---

### 5.3 Collision de routes - Addresses

| | |
|---|---|
| **Fichier** | `controller/AddressController.java` |
| **Severite** | MAJEUR |

Meme probleme : `@GetMapping("/default")` (ligne 96) est apres `@GetMapping("/{id}")` (ligne 44). `/api/addresses/default` est interprete comme `id="default"`.

---

### 5.4 Race condition sur le stock

| | |
|---|---|
| **Fichier** | `service/OrderService.java:86-106` |
| **Severite** | MAJEUR |

La verification du stock (ligne 90) et la deduction (ligne 100) ne sont pas atomiques. Deux commandes simultanees peuvent valider un stock insuffisant.

**Correction :** Utiliser un verrouillage pessimiste :
```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Product> findById(Long id);
```

---

### 5.5 Missing @Transactional sur ProductService

| | |
|---|---|
| **Fichier** | `service/ProductService.java` |
| **Severite** | MAJEUR |

La suppression d'images + produit peut echouer partiellement sans annotation `@Transactional`.

---

### 5.6 Typo MTN Mobile Money

| | |
|---|---|
| **Fichier** | `service/MtnMobileMoneyService.java:108` |

```java
headers.set("X-Target-Environment", "mtnc ameroon");  // devrait etre "mtncameroon"
```

---

### 5.7 Mauvais import BadRequestException

| | |
|---|---|
| **Fichiers** | `service/ProductService.java:12`, `service/UserService.java` |

```java
import org.apache.coyote.BadRequestException;  // FAUX
// Devrait etre :
import com.tdk.backend_tdk.exception.BadRequestException;
```

---

### 5.8 CORS configuree en double

| | |
|---|---|
| **Fichiers** | `config/SecurityConfig.java` + `webconfig/WebConfig.java` |

Deux configurations CORS avec des valeurs differentes. `WebConfig` hardcode `localhost:4200`, `SecurityConfig` utilise une variable configurable.

**Correction :** Supprimer la config CORS dans `WebConfig.java` et garder uniquement celle dans `SecurityConfig`.

---

### 5.9 Annulation de commande sans remboursement

| | |
|---|---|
| **Fichier** | `service/OrderService.java:164-181` |

`cancelOrder()` ne verifie pas si le paiement a ete effectue et ne declenche aucun remboursement. Un utilisateur pourrait annuler une commande payee sans etre rembourse.

---

### 5.10 Mauvais import dans les services

| | |
|---|---|
| **Fichier** | `service/ProductService.java:12` |

Utilise `org.apache.coyote.BadRequestException` au lieu de l'exception custom du projet.

---

## 6. Backend - Problemes Moderes

### 6.1 N+1 Query dans StatisticsService

| | |
|---|---|
| **Fichier** | `service/StatisticsService.java:160-189` |

`order.getUser().getEmail()` declenche N requetes supplementaires.

**Correction :** Utiliser `JOIN FETCH` dans les requetes repository.

---

### 6.2 System.err au lieu de SLF4J

| | |
|---|---|
| **Fichiers** | Tous les services |

`System.err.println()` utilise partout au lieu du logger SLF4J.

**Correction :** Ajouter `@Slf4j` (Lombok) et utiliser `log.error()`, `log.warn()`.

---

### 6.3 Email verification non implementee

| | |
|---|---|
| **Fichier** | `service/AuthService.java:93-94` |

```java
// TODO: Envoyer email de vérification
// emailService.sendEmailVerification(savedUser);
```

Les utilisateurs s'inscrivent sans verification d'email.

---

### 6.4 Pas de rate limiting sur les endpoints auth

| | |
|---|---|
| **Fichiers** | Tous les controllers auth |

Login, register, forgot-password n'ont aucun rate limiting. Vulnerable aux attaques brute force.

---

### 6.5 Integer pour le stock au lieu de int

| | |
|---|---|
| **Fichier** | `model/Product.java:36` |

```java
private Integer stockQuantity = 0;  // Peut etre null → NPE
```

**Correction :** Utiliser `int` (primitif) ou ajouter `@Column(nullable = false)`.

---

## 7. Desalignement Frontend / Backend

### 7.1 Path dashboard stats different (CRITIQUE)

| | |
|---|---|
| **Frontend** | `GET /admin/dashboard-stats` (admin.service.ts:31) |
| **Backend** | `GET /admin/dashboard/stats` (AdminDashboardController) |
| **Impact** | Erreur 404 garantie |

---

### 7.2 CategoryController manquant (CRITIQUE)

| | |
|---|---|
| **Frontend** | `CategoryService` avec CRUD complet (`GET/POST/PUT/DELETE /categories`) |
| **Backend** | Aucun `CategoryController` |
| **Impact** | Toutes les operations categories echouent |

---

### 7.3 Payment non integre cote frontend

| | |
|---|---|
| **Backend** | Stripe + Orange Money + MTN Mobile Money implementes |
| **Frontend** | Aucun service de paiement, methodes stockees en localStorage uniquement |

---

### 7.4 Email verification non utilisee cote frontend

| | |
|---|---|
| **Backend** | Endpoints `verify-email` et `resend-verification` existent |
| **Frontend** | N'appelle jamais ces endpoints |

---

### 7.5 Token refresh absent du frontend

| | |
|---|---|
| **Backend** | `POST /auth/refresh-token` existe |
| **Frontend** | Ne l'appelle jamais, le token expire apres 24h sans renouvellement |

---

### 7.6 Endpoints backend non utilises

| Endpoint Backend | Description |
|-----------------|-------------|
| `GET /products/new` | Nouveaux produits |
| `GET /products/sale` | Produits en promotion |
| `GET /products/category/{id}` | Produits par categorie |
| `DELETE /user/avatar` | Suppression d'avatar |
| `GET /orders/number/{orderNumber}` | Commande par numero |
| `GET /addresses/default` | Adresse par defaut |
| `GET /cart/total` | Total du panier |

---

## 8. Resume

### Par severite

| Severite | Nombre | Categories |
|----------|--------|------------|
| **CRITIQUE** | 8 | Securite backend (credentials, admin public, H2, webhook, traversal), compilation frontend (@shared), desalignement API |
| **MAJEUR** | 10 | Bugs backend (enums dupliques, collisions routes, race conditions, typos), memory leaks frontend |
| **MODERE** | 8 | Frontend (routes, state management, interceptor), backend (N+1, logging, email) |
| **MINEUR** | 4 | Code mort, switch sans default, typos modeles |
| **Total** | **~30** | |

### Actions immediates recommandees

1. **Securite backend** : Retirer les credentials en dur, proteger `/register-admin`, desactiver H2
2. **Compilation frontend** : Configurer l'alias `@shared` dans tsconfig
3. **Desalignement API** : Corriger le path dashboard-stats, creer le CategoryController
4. **Bugs backend** : Fusionner les enums dupliques, corriger les collisions de routes, ajouter `@Transactional`
5. **Memory leaks frontend** : Ajouter `OnDestroy` et `takeUntil()` dans tous les composants avec subscriptions
6. **Typos** : Corriger selector reset-password, content-type, MTN environment
