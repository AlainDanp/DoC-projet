# 📗 Guide Spring Boot — Annotations & Tests

> Guide de référence : comprendre les termes/annotations propres à Spring, puis maîtriser tous les types de tests.
> Version cible : **Spring Boot 3.x** / **Java 17+**

---

## Table des matières

### Partie A — Les annotations Spring
1. [Le cœur de Spring : IoC et Injection de dépendances](#1-le-cœur-de-spring--ioc-et-injection-de-dépendances)
2. [Annotations de déclaration de beans](#2-annotations-de-déclaration-de-beans)
3. [Annotations d'injection](#3-annotations-dinjection)
4. [Annotations de configuration](#4-annotations-de-configuration)
5. [Annotations Web / REST](#5-annotations-web--rest)
6. [Annotations JPA / Base de données](#6-annotations-jpa--base-de-données)
7. [Annotations Lombok](#7-annotations-lombok-data-etc)
8. [Annotations de validation](#8-annotations-de-validation)
9. [Annotations diverses (async, cache, transactions...)](#9-annotations-diverses)

### Partie B — Les tests
10. [Vue d'ensemble : la pyramide des tests](#10-la-pyramide-des-tests)
11. [Tests unitaires (JUnit 5 + Mockito)](#11-tests-unitaires)
12. [Le mock en détail (Mockito)](#12-le-mock-en-détail)
13. [Tests d'intégration (@SpringBootTest, @MockitoBean)](#13-tests-dintégration)
14. [Tests de la couche Web + JSON (@WebMvcTest, @JsonTest)](#14-tests-web--json)
15. [Tests avec base de données (@DataJpaTest, Testcontainers)](#15-tests-avec-base-de-données)
16. [Tests de charge (JMeter, Gatling)](#16-tests-de-charge)
17. [BDD — Behavior Driven Development (Cucumber)](#17-bdd--cucumber)
18. [Récapitulatif : quelle annotation de test pour quel besoin ?](#18-récapitulatif)

---

# PARTIE A — LES ANNOTATIONS SPRING

## 1. Le cœur de Spring : IoC et Injection de dépendances

Avant les annotations, il faut comprendre **2 concepts fondamentaux** :

### IoC — Inversion of Control (Inversion de contrôle)

Sans Spring, c'est **toi** qui crées les objets :

```java
// ❌ Sans Spring : couplage fort, difficile à tester
public class ProductService {
    private ProductRepository repository = new ProductRepositoryImpl(); // créé à la main
}
```

Avec Spring, c'est le **conteneur Spring (ApplicationContext)** qui crée et gère les objets. Ces objets gérés par Spring s'appellent des **beans**.

### DI — Dependency Injection (Injection de dépendances)

Spring **fournit** (injecte) automatiquement les dépendances dont une classe a besoin :

```java
// ✅ Avec Spring : la dépendance est injectée, testable facilement
@Service
public class ProductService {
    private final ProductRepository repository;

    public ProductService(ProductRepository repository) { // Spring injecte ici
        this.repository = repository;
    }
}
```

> 🧠 **À retenir** : un **bean** = un objet créé, configuré et géré par Spring. Toutes les annotations de la section suivante servent à dire à Spring « crée un bean à partir de cette classe ».

---

## 2. Annotations de déclaration de beans

Ces annotations, posées **sur une classe**, demandent à Spring de créer un bean.

| Annotation | Rôle | Où l'utiliser |
|---|---|---|
| `@Component` | Bean générique | Toute classe technique (adaptateurs, helpers) |
| `@Service` | Bean de logique métier | Couche application/service |
| `@Repository` | Bean d'accès aux données + traduction des exceptions SQL en exceptions Spring | Couche persistence |
| `@Controller` | Bean web qui retourne des vues (HTML) | Applications MVC classiques |
| `@RestController` | `@Controller` + `@ResponseBody` : retourne du JSON | API REST (ton cas) |
| `@Configuration` | Classe contenant des définitions de beans (`@Bean`) | Configuration |

> 💡 `@Service`, `@Repository`, `@Controller` sont des **spécialisations** de `@Component`. Techniquement équivalents, mais ils documentent le rôle de la classe et activent quelques comportements spécifiques.

### `@Bean` — déclarer un bean manuellement

`@Component` fonctionne sur **tes** classes. Mais pour des classes **externes** (librairies) que tu ne peux pas annoter, utilise `@Bean` dans une classe `@Configuration` :

```java
@Configuration
public class BeanConfig {

    @Bean   // Spring exécute cette méthode et enregistre le résultat comme bean
    public RestClient restClient() {
        return RestClient.builder()
            .baseUrl("https://api.exemple.com")
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();  // classe de Spring Security, pas la tienne
    }
}
```

**`@Component` vs `@Bean` :**

| | `@Component` | `@Bean` |
|---|---|---|
| Se pose sur | une classe (la tienne) | une méthode dans `@Configuration` |
| Utilisation | tes propres classes | classes de librairies externes, ou construction complexe |
| Détection | scan automatique du package | exécution de la méthode |

### `@SpringBootApplication`

Posée sur la classe principale, elle combine 3 annotations :

```java
@SpringBootApplication
// = @Configuration          (la classe peut définir des beans)
// + @EnableAutoConfiguration (Spring configure tout seul selon les dépendances du pom)
// + @ComponentScan           (scanne le package courant et ses sous-packages pour trouver les @Component)
public class MonApplication {
    public static void main(String[] args) {
        SpringApplication.run(MonApplication.class, args);
    }
}
```

> ⚠️ **Piège classique** : `@ComponentScan` ne scanne que le package de la classe principale **et ses sous-packages**. Si un bean est ailleurs, Spring ne le trouvera pas (`NoSuchBeanDefinitionException`).

---

## 3. Annotations d'injection

### `@Autowired` — l'injection automatique

Demande à Spring d'injecter un bean. **3 façons de l'utiliser :**

```java
@Service
public class ProductService {

    // ❌ 1. Injection par champ — À ÉVITER
    @Autowired
    private ProductRepository repository;
    // Problèmes : impossible de mettre 'final', difficile à tester sans Spring,
    // masque les dépendances trop nombreuses

    // ⚠️ 2. Injection par setter — cas rares (dépendances optionnelles)
    @Autowired
    public void setRepository(ProductRepository repository) {
        this.repository = repository;
    }

    // ✅ 3. Injection par CONSTRUCTEUR — LA BONNE PRATIQUE
    private final ProductRepository repository;   // final = immuable

    public ProductService(ProductRepository repository) {
        this.repository = repository;
    }
    // Bonus : depuis Spring 4.3, si la classe n'a QU'UN constructeur,
    // @Autowired est même INUTILE — Spring injecte automatiquement !
}
```

> 🏆 **Règle d'or** : injection par constructeur + champs `final`. Avec Lombok, `@RequiredArgsConstructor` génère ce constructeur pour toi (cf. §7).

### `@Qualifier` — choisir entre plusieurs implémentations

Si 2 beans implémentent la même interface, Spring ne sait pas lequel injecter :

```java
@Component("twilioSender")
public class TwilioSmsSender implements SmsSender { ... }

@Component("vonageSender")
public class VonageSmsSender implements SmsSender { ... }

@Service
public class OtpService {
    public OtpService(@Qualifier("twilioSender") SmsSender smsSender) { ... }
}
```

### `@Primary` — implémentation par défaut

```java
@Component
@Primary   // sera choisi par défaut si aucun @Qualifier n'est précisé
public class TwilioSmsSender implements SmsSender { ... }
```

### `@Value` — injecter une propriété de configuration

```java
@Component
public class JwtService {

    @Value("${app.jwt.secret}")            // depuis application.yml
    private String secret;

    @Value("${app.jwt.expiration-ms:86400000}")  // avec valeur par défaut après ':'
    private long expirationMs;
}
```

### `@ConfigurationProperties` — injecter un groupe de propriétés (préférable à @Value)

```java
// application.yml :
// app:
//   otp:
//     length: 6
//     expiration-minutes: 5

@ConfigurationProperties(prefix = "app.otp")
public record OtpProperties(int length, int expirationMinutes) {}

// À activer sur la classe principale :
@EnableConfigurationProperties(OtpProperties.class)
```

---

## 4. Annotations de configuration

| Annotation | Rôle |
|---|---|
| `@Configuration` | Classe de configuration contenant des `@Bean` |
| `@Profile("dev")` | Le bean n'existe que si le profil est actif (`spring.profiles.active=dev`) |
| `@ConditionalOnProperty` | Le bean n'existe que si une propriété a une certaine valeur |
| `@EnableAsync` | Active le support de `@Async` |
| `@EnableScheduling` | Active le support de `@Scheduled` |
| `@EnableFeignClients` | Active les clients Feign |
| `@EnableWebSecurity` | Active la configuration Spring Security |

### Exemple `@Profile` — très utile pour l'OTP en dev

```java
@Component
@Profile("dev")    // en dev : loggue le code au lieu d'envoyer un vrai SMS
public class FakeSmsSender implements SmsSender {
    public void send(String phone, String message) {
        log.info("SMS SIMULÉ vers {} : {}", phone, message);
    }
}

@Component
@Profile("prod")   // en prod : vrai envoi Twilio
public class TwilioSmsSender implements SmsSender { ... }
```

---

## 5. Annotations Web / REST

| Annotation | Rôle | Exemple |
|---|---|---|
| `@RestController` | Classe exposant des endpoints JSON | sur la classe |
| `@RequestMapping("/api/v1/products")` | Préfixe d'URL commun | sur la classe |
| `@GetMapping`, `@PostMapping`, `@PutMapping`, `@PatchMapping`, `@DeleteMapping` | Mappe une méthode HTTP + chemin | `@GetMapping("/{id}")` |
| `@PathVariable` | Récupère une variable de l'URL | `/products/5` → `@PathVariable Long id` |
| `@RequestParam` | Récupère un paramètre de requête | `/products?page=2` → `@RequestParam int page` |
| `@RequestBody` | Désérialise le corps JSON en objet Java | `@RequestBody ProductRequest req` |
| `@ResponseBody` | Sérialise le retour en JSON (inclus dans `@RestController`) | — |
| `@RequestHeader` | Récupère un header HTTP | `@RequestHeader("Authorization") String auth` |
| `@ResponseStatus(HttpStatus.CREATED)` | Force le code HTTP de réponse | sur méthode ou exception |
| `@Valid` | Déclenche la validation du DTO | `@Valid @RequestBody ProductRequest req` |
| `@RestControllerAdvice` | Handler global d'exceptions pour tous les controllers | cf. guide 1, §5 |
| `@ExceptionHandler` | Traite un type d'exception précis | dans le `@RestControllerAdvice` |
| `@CrossOrigin` | Autorise les appels CORS (front sur un autre domaine) | sur classe ou méthode |

### Exemple synthétique

```java
@RestController
@RequestMapping("/api/v1/products")
public class ProductController {

    // GET /api/v1/products/5
    @GetMapping("/{id}")
    public ProductResponse getById(@PathVariable Long id) { ... }

    // GET /api/v1/products/search?name=tele&page=0
    @GetMapping("/search")
    public List<ProductResponse> search(@RequestParam String name,
                                        @RequestParam(defaultValue = "0") int page) { ... }

    // POST /api/v1/products  (corps JSON)
    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public ProductResponse create(@Valid @RequestBody ProductRequest request) { ... }
}
```

---

## 6. Annotations JPA / Base de données

| Annotation | Rôle |
|---|---|
| `@Entity` | La classe est mappée à une table |
| `@Table(name = "products")` | Nom de la table (sinon = nom de la classe) |
| `@Id` | Clé primaire |
| `@GeneratedValue(strategy = GenerationType.IDENTITY)` | Auto-incrément géré par la BDD |
| `@Column(name = "...", nullable = false, unique = true, length = 100)` | Configuration de la colonne |
| `@Transient` | Champ NON persisté en base |
| `@Enumerated(EnumType.STRING)` | Stocke un enum comme texte (toujours STRING, jamais ORDINAL !) |
| `@CreationTimestamp` / `@UpdateTimestamp` | Dates auto-remplies (Hibernate) |
| `@OneToMany`, `@ManyToOne`, `@OneToOne`, `@ManyToMany` | Relations entre entités |
| `@JoinColumn(name = "user_id")` | Colonne de clé étrangère |
| `@Query("SELECT ...")` | Requête JPQL personnalisée dans un repository |
| `@Modifying` | Requête `@Query` de type UPDATE/DELETE |
| `@Transactional` | La méthode s'exécute dans une transaction (rollback si exception) |

### Exemple de relation

```java
@Entity
public class Order {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)   // LAZY = chargé seulement si accédé (recommandé)
    @JoinColumn(name = "customer_id", nullable = false)
    private Customer customer;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<OrderLine> lines = new ArrayList<>();
}
```

> ⚠️ **Pièges classiques** : `FetchType.EAGER` sur les collections (chargement massif inutile) ; `@ManyToMany` sans table de jointure explicite ; oublier `@Transactional` sur une méthode qui modifie plusieurs entités.

---

## 7. Annotations Lombok (@Data, etc.)

**Lombok n'est PAS Spring** : c'est une librairie qui **génère du code à la compilation** pour éviter le boilerplate (getters, setters, constructeurs...).

| Annotation | Génère |
|---|---|
| `@Getter` / `@Setter` | Getters / setters pour tous les champs |
| `@ToString` | Méthode `toString()` |
| `@EqualsAndHashCode` | `equals()` et `hashCode()` |
| `@NoArgsConstructor` | Constructeur vide |
| `@AllArgsConstructor` | Constructeur avec TOUS les champs |
| `@RequiredArgsConstructor` | Constructeur avec les champs `final` uniquement ⭐ |
| `@Data` | `@Getter` + `@Setter` + `@ToString` + `@EqualsAndHashCode` + `@RequiredArgsConstructor` |
| `@Builder` | Pattern builder : `Product.builder().name("TV").price(499).build()` |
| `@Slf4j` | Un logger : `log.info("...")` |
| `@Value` (Lombok) | Classe immuable (⚠️ à ne pas confondre avec `@Value` de Spring !) |

### Le duo gagnant avec Spring

```java
@Service
@RequiredArgsConstructor   // génère le constructeur des champs final → Spring injecte dedans
@Slf4j                     // fournit 'log'
public class ProductService {

    private final ProductRepository repository;    // injecté automatiquement
    private final NotificationPort notification;   // injecté automatiquement

    public Product create(Product p) {
        log.info("Création du produit {}", p.getName());
        return repository.save(p);
    }
}
```

### ⚠️ Précautions avec `@Data`

- **Sur les entités JPA : éviter `@Data`** — `@ToString` et `@EqualsAndHashCode` sur des relations LAZY peuvent déclencher des chargements involontaires ou des boucles infinies. Préfère `@Getter @Setter` explicites.
- Sur les DTO : `@Data` est parfait — ou mieux, utilise des **records Java** qui font le travail nativement :

```java
public record ProductResponse(Long id, String name, BigDecimal price) {}
// = immuable + constructeur + getters + equals/hashCode/toString, sans Lombok
```

---

## 8. Annotations de validation

(Voir aussi guide 1, §5.) Package `jakarta.validation` :

```java
public record UserRequest(
    @NotBlank(message = "Le nom est requis")
    @Size(min = 2, max = 50)
    String name,

    @NotBlank @Email(message = "Email invalide")
    String email,

    @NotNull @Min(18) @Max(120)
    Integer age,

    @Pattern(regexp = "^\\+[1-9]\\d{7,14}$", message = "Téléphone au format international")
    String phone,

    @Past(message = "La date de naissance doit être dans le passé")
    LocalDate birthDate
) {}
```

Déclenchée par `@Valid` dans le controller. Les erreurs sont capturées par le `GlobalExceptionHandler` (`MethodArgumentNotValidException`).

---

## 9. Annotations diverses

| Annotation | Rôle | Prérequis |
|---|---|---|
| `@Async` | Exécute la méthode dans un thread séparé (ex: envoi de mail) | `@EnableAsync` |
| `@Scheduled(cron = "0 0 8 * * *")` | Tâche planifiée (ici : tous les jours à 8h) | `@EnableScheduling` |
| `@Transactional` | Transaction BDD avec rollback automatique sur exception | — |
| `@Transactional(readOnly = true)` | Optimisation pour les lectures seules | — |
| `@Cacheable("products")` | Met en cache le résultat de la méthode | `@EnableCaching` |
| `@CacheEvict("products")` | Invalide le cache (après un update/delete) | `@EnableCaching` |
| `@Retryable` | Réessaie automatiquement en cas d'échec | spring-retry |
| `@PostConstruct` | Méthode exécutée juste après la création du bean | — |
| `@PreDestroy` | Méthode exécutée avant la destruction du bean | — |

```java
@Service
public class ReportService {

    @Scheduled(cron = "0 0 8 * * MON")   // chaque lundi 8h
    public void sendWeeklyReport() { ... }

    @Async
    public void sendConfirmationEmail(String to) { ... }   // ne bloque pas l'appelant

    @Transactional   // tout réussit ou tout est annulé
    public void transferMoney(Long fromId, Long toId, BigDecimal amount) {
        accountService.debit(fromId, amount);
        accountService.credit(toId, amount);   // si exception ici → le débit est annulé
    }
}
```

---

# PARTIE B — LES TESTS

## 10. La pyramide des tests

```
            ▲  Lents, coûteux, peu nombreux
            │
        ┌───────┐
        │  E2E  │          Tests bout-en-bout (toute l'app + vraie BDD)
        ├───────┴───┐
        │Intégration│      Plusieurs composants ensemble (@SpringBootTest)
        ├───────────┴──┐
        │  Unitaires   │   Une classe isolée, dépendances mockées (JUnit + Mockito)
        └──────────────┘
            │
            ▼  Rapides, ciblés, très nombreux (la base !)
```

| Type | Ce qu'on teste | Vitesse | Contexte Spring ? |
|---|---|---|---|
| **Unitaire** | Une classe seule, dépendances mockées | ⚡ ms | ❌ Non |
| **Intégration (tranche)** | Une couche (web OU jpa) | 🚶 s | ✅ Partiel |
| **Intégration (complet)** | Toute l'application | 🐢 s-min | ✅ Complet |
| **Charge** | Performance sous trafic | 🐢🐢 | App déployée |

### Dépendance de test (déjà incluse dans `spring-boot-starter-test`)

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
<!-- Inclut : JUnit 5, Mockito, AssertJ, MockMvc, JsonPath, Hamcrest -->
```

---

## 11. Tests unitaires

**Objectif** : tester **UNE classe** (souvent un service) en isolation totale, sans Spring, sans BDD, sans réseau. Les dépendances sont remplacées par des **mocks**.

### Anatomie d'un test : le pattern AAA (Arrange / Act / Assert)

```java
// src/test/java/.../application/service/ProductServiceTest.java

@ExtendWith(MockitoExtension.class)   // active Mockito avec JUnit 5 — PAS de Spring ici !
class ProductServiceTest {

    @Mock                             // crée un FAUX ProductRepositoryPort
    private ProductRepositoryPort repository;

    @InjectMocks                      // crée un VRAI ProductService et lui injecte les @Mock
    private ProductService productService;

    @Test
    @DisplayName("getById retourne le produit quand il existe")
    void getById_shouldReturnProduct_whenExists() {
        // ─── ARRANGE (préparer) ───
        Product product = new Product(1L, "TV", "4K", new BigDecimal("499"));
        when(repository.findById(1L)).thenReturn(Optional.of(product));

        // ─── ACT (exécuter) ───
        Product result = productService.getById(1L);

        // ─── ASSERT (vérifier) ───
        assertThat(result.getName()).isEqualTo("TV");                 // AssertJ
        verify(repository).findById(1L);                              // le mock a bien été appelé
    }

    @Test
    @DisplayName("getById lève une exception quand le produit n'existe pas")
    void getById_shouldThrow_whenNotFound() {
        when(repository.findById(99L)).thenReturn(Optional.empty());

        assertThatThrownBy(() -> productService.getById(99L))
            .isInstanceOf(ProductNotFoundException.class)
            .hasMessageContaining("99");
    }
}
```

### Annotations JUnit 5 essentielles

| Annotation | Rôle |
|---|---|
| `@Test` | Marque une méthode de test |
| `@DisplayName("...")` | Nom lisible du test dans les rapports |
| `@BeforeEach` / `@AfterEach` | Exécuté avant/après CHAQUE test |
| `@BeforeAll` / `@AfterAll` | Exécuté une fois avant/après TOUS les tests (méthode static) |
| `@ParameterizedTest` + `@ValueSource` / `@CsvSource` | Même test, plusieurs jeux de données |
| `@Disabled("raison")` | Désactive temporairement un test |
| `@Nested` | Regroupe des tests en sous-classes |

### Test paramétré (très utile)

```java
@ParameterizedTest
@CsvSource({
    "10, 90.00",      // remise 10% sur 100 → 90
    "50, 50.00",
    "100, 0.00"
})
void applyDiscount_shouldComputeCorrectPrice(int percent, BigDecimal expected) {
    Product p = new Product(1L, "TV", "", new BigDecimal("100"));
    p.applyDiscount(percent);
    assertThat(p.getPrice()).isEqualByComparingTo(expected);
}
```

> 🏛️ **Lien avec la Clean Architecture** : c'est ici qu'elle paie ! Ton domaine et tes services ne dépendent que d'interfaces (ports) → ils se testent en pur unitaire, ultra rapide, sans démarrer Spring.

---

## 12. Le mock en détail

### Qu'est-ce qu'un mock ?

Un **mock** est un **faux objet** qui remplace une vraie dépendance (BDD, API externe, envoi de SMS...). Tu **programmes son comportement** et tu **vérifies comment il a été utilisé**.

**Pourquoi mocker ?**
- ⚡ Rapidité : pas de vraie BDD/réseau
- 🎯 Isolation : si le test échoue, le bug est dans LA classe testée
- 🎭 Scénarios impossibles à reproduire : simuler une panne, un timeout, une réponse d'erreur
- 💸 Pas d'effets de bord : on n'envoie pas de vrais SMS/emails pendant les tests !

### Les 3 opérations Mockito à connaître

#### 1️⃣ Stubbing — programmer le comportement (`when...thenReturn`)

```java
// "Quand on appellera findById(1L), retourne ce produit"
when(repository.findById(1L)).thenReturn(Optional.of(product));

// Retourner selon l'argument (matchers) :
when(repository.findById(anyLong())).thenReturn(Optional.of(product));

// Simuler une exception (panne de la BDD, API externe down...) :
when(repository.findById(1L)).thenThrow(new RuntimeException("BDD indisponible"));

// Pour les méthodes void :
doNothing().when(smsSender).send(anyString(), anyString());
doThrow(new SmsException()).when(smsSender).send(eq("+33600000000"), anyString());
```

#### 2️⃣ Verification — vérifier les interactions (`verify`)

```java
verify(smsSender).send("+33612345678", "Votre code : 123456");  // appelé exactement 1 fois
verify(smsSender, times(2)).send(anyString(), anyString());     // appelé 2 fois
verify(smsSender, never()).send(anyString(), anyString());      // JAMAIS appelé
verifyNoInteractions(mailSender);                               // aucun appel du tout
```

#### 3️⃣ Capture — inspecter les arguments passés (`ArgumentCaptor`)

```java
@Captor
private ArgumentCaptor<String> messageCaptor;

@Test
void requestOtp_shouldSendSixDigitCode() {
    otpService.requestOtp("+33612345678");

    verify(smsSender).send(eq("+33612345678"), messageCaptor.capture());
    assertThat(messageCaptor.getValue()).matches(".*\\d{6}.*");  // contient bien 6 chiffres
}
```

### Mock vs Spy

| | `@Mock` | `@Spy` |
|---|---|---|
| Objet | 100% faux | VRAI objet, partiellement remplacé |
| Comportement par défaut | retourne null/0/vide | exécute le vrai code |
| Usage | dépendances externes | rare — surcharger 1 méthode d'un vrai objet |

```java
@Spy
private List<String> realList = new ArrayList<>();  // vraie liste, mais espionnée

realList.add("a");                    // vrai add
verify(realList).add("a");            // mais on peut vérifier les appels
```

> ⚠️ **Règle** : ne mocke jamais l'objet que tu testes, uniquement ses **dépendances**. Et ne teste pas les mocks eux-mêmes (tester que le mock retourne ce que tu lui as dit de retourner ne prouve rien).

---

## 13. Tests d'intégration

**Objectif** : tester que **plusieurs composants fonctionnent ensemble**, avec un vrai contexte Spring.

### `@SpringBootTest` — le contexte complet

```java
@SpringBootTest   // démarre TOUT le contexte Spring (tous les beans)
class ProductIntegrationTest {

    @Autowired    // ici on injecte les VRAIS beans
    private ProductUseCase productUseCase;

    @Test
    void create_thenGetById_shouldReturnSameProduct() {
        Product created = productUseCase.create(
            new Product(null, "Laptop", "16Go", new BigDecimal("999")));

        Product found = productUseCase.getById(created.getId());

        assertThat(found.getName()).isEqualTo("Laptop");
    }
}
```

### `@MockitoBean` — mocker UN bean dans le contexte Spring ⭐

> 📌 **Important** : depuis **Spring Boot 3.4**, `@MockBean` est **déprécié** et remplacé par **`@MockitoBean`** (package `org.springframework.test.context.bean.override.mockito`). Même principe, nouveau nom. Si tu vois `@MockBean` dans d'anciens tutos, c'est l'ancêtre direct.

**Différence fondamentale avec `@Mock` :**

| | `@Mock` (Mockito pur) | `@MockitoBean` (Spring) |
|---|---|---|
| Contexte Spring | ❌ aucun | ✅ requis (`@SpringBootTest`, `@WebMvcTest`...) |
| Effet | crée un mock local au test | **remplace le vrai bean dans le conteneur Spring** |
| Usage | tests unitaires | tests d'intégration où l'on veut neutraliser 1 dépendance |

**Cas d'usage typique** : test d'intégration complet, mais on ne veut PAS envoyer de vrais SMS :

```java
@SpringBootTest
class OtpAuthIntegrationTest {

    @Autowired
    private OtpAuthUseCase otpAuthUseCase;      // vrai service, vraie BDD, vrai contexte

    @MockitoBean                                 // MAIS le sender SMS est remplacé par un mock
    private OtpSenderPort otpSender;             // → aucun vrai SMS envoyé

    @Test
    void requestOtp_shouldStoreCodeAndSendSms() {
        otpAuthUseCase.requestOtp("+33612345678");

        verify(otpSender).sendOtp(eq("+33612345678"), anyString());
    }
}
```

Il existe aussi **`@MockitoSpyBean`** (remplace `@SpyBean`) : le vrai bean est conservé mais espionné.

### Tester l'API de bout en bout avec un vrai serveur HTTP

```java
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class ProductApiE2ETest {

    @Autowired
    private TestRestTemplate restTemplate;   // vrai client HTTP fourni par Spring

    @Test
    void postProduct_shouldReturn201() {
        ProductRequest request = new ProductRequest("TV", "4K", new BigDecimal("499"));

        ResponseEntity<ProductResponse> response =
            restTemplate.postForEntity("/api/v1/products", request, ProductResponse.class);

        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.CREATED);
        assertThat(response.getBody().id()).isNotNull();
    }
}
```

### `@ActiveProfiles` et configuration de test

```java
@SpringBootTest
@ActiveProfiles("test")   // utilise application-test.yml (ex: BDD H2 en mémoire)
class MyIntegrationTest { ... }
```

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb
    driver-class-name: org.h2.Driver
  jpa:
    hibernate:
      ddl-auto: create-drop
```

---

## 14. Tests Web + JSON

### `@WebMvcTest` — tester UNIQUEMENT la couche web

Charge seulement les controllers (pas les services, pas la BDD) → rapide. Les services sont mockés avec `@MockitoBean`.

```java
@WebMvcTest(ProductController.class)          // seulement CE controller
@AutoConfigureMockMvc(addFilters = false)     // désactive les filtres de sécurité si besoin
class ProductControllerTest {

    @Autowired
    private MockMvc mockMvc;                  // simule des requêtes HTTP SANS vrai serveur

    @MockitoBean
    private ProductUseCase productUseCase;    // le service est mocké

    @MockitoBean
    private ProductWebMapper mapper;

    @Autowired
    private ObjectMapper objectMapper;        // Java <-> JSON (Jackson)

    @Test
    void getById_shouldReturn200_withJsonBody() throws Exception {
        Product product = new Product(1L, "TV", "4K", new BigDecimal("499"));
        when(productUseCase.getById(1L)).thenReturn(product);
        when(mapper.toResponse(product))
            .thenReturn(new ProductResponse(1L, "TV", "4K", new BigDecimal("499"), Instant.now()));

        mockMvc.perform(get("/api/v1/products/1"))
            .andExpect(status().isOk())
            .andExpect(content().contentType(MediaType.APPLICATION_JSON))
            // ─── Assertions sur le JSON avec JsonPath ───
            .andExpect(jsonPath("$.id").value(1))
            .andExpect(jsonPath("$.name").value("TV"))
            .andExpect(jsonPath("$.price").value(499));
    }

    @Test
    void create_shouldReturn400_whenNameIsBlank() throws Exception {
        ProductRequest invalid = new ProductRequest("", "desc", new BigDecimal("10"));

        mockMvc.perform(post("/api/v1/products")
                .contentType(MediaType.APPLICATION_JSON)
                .content(objectMapper.writeValueAsString(invalid)))   // objet Java → JSON
            .andExpect(status().isBadRequest())
            .andExpect(jsonPath("$.details.name").exists());          // le détail de validation
    }

    @Test
    void getById_shouldReturn404_whenNotFound() throws Exception {
        when(productUseCase.getById(99L)).thenThrow(new ProductNotFoundException(99L));

        mockMvc.perform(get("/api/v1/products/99"))
            .andExpect(status().isNotFound())
            .andExpect(jsonPath("$.message").value(containsString("99")));
    }
}
```

### JsonPath — le langage d'interrogation du JSON

| Expression | Signifie |
|---|---|
| `$.name` | champ `name` à la racine |
| `$.address.city` | champ imbriqué |
| `$[0].name` | premier élément d'un tableau |
| `$.items.length()` | taille d'un tableau |
| `$.items[?(@.price > 100)]` | filtrage |

### `@JsonTest` — tester la sérialisation JSON seule

Utile pour vérifier tes DTO, formats de dates, champs ignorés :

```java
@JsonTest
class ProductResponseJsonTest {

    @Autowired
    private JacksonTester<ProductResponse> json;

    @Test
    void serialize_shouldProduceExpectedJson() throws Exception {
        ProductResponse dto = new ProductResponse(1L, "TV", "4K",
            new BigDecimal("499"), Instant.parse("2026-01-15T10:00:00Z"));

        assertThat(json.write(dto)).extractingJsonPathStringValue("$.name").isEqualTo("TV");
        assertThat(json.write(dto)).extractingJsonPathNumberValue("$.price").isEqualTo(499);
    }

    @Test
    void deserialize_shouldMapJsonToObject() throws Exception {
        String content = """
            {"id":1,"name":"TV","description":"4K","price":499,"createdAt":"2026-01-15T10:00:00Z"}
            """;

        assertThat(json.parseObject(content).name()).isEqualTo("TV");
    }
}
```

---

## 15. Tests avec base de données

### `@DataJpaTest` — tester UNIQUEMENT la couche JPA

Charge seulement les entités + repositories, avec une **BDD H2 en mémoire** par défaut. Chaque test est **transactionnel avec rollback automatique** (la BDD est propre entre les tests).

```java
@DataJpaTest
class ProductJpaRepositoryTest {

    @Autowired
    private ProductJpaRepository repository;

    @Autowired
    private TestEntityManager entityManager;   // pour préparer les données

    @Test
    void findByNameContainingIgnoreCase_shouldReturnMatches() {
        // Arrange : insérer des données de test
        entityManager.persist(ProductEntity.builder().name("Television").price(new BigDecimal("499")).build());
        entityManager.persist(ProductEntity.builder().name("Radio").price(new BigDecimal("49")).build());
        entityManager.flush();

        // Act
        List<ProductEntity> result = repository.findByNameContainingIgnoreCase("tele");

        // Assert
        assertThat(result).hasSize(1);
        assertThat(result.get(0).getName()).isEqualTo("Television");
    }

    @Test
    void findByPriceRange_shouldFilterCorrectly() {
        entityManager.persist(ProductEntity.builder().name("A").price(new BigDecimal("10")).build());
        entityManager.persist(ProductEntity.builder().name("B").price(new BigDecimal("100")).build());
        entityManager.persist(ProductEntity.builder().name("C").price(new BigDecimal("1000")).build());
        entityManager.flush();

        List<ProductEntity> result = repository.findByPriceRange(
            new BigDecimal("50"), new BigDecimal("500"));

        assertThat(result).extracting(ProductEntity::getName).containsExactly("B");
    }
}
```

### Testcontainers — tester avec une VRAIE BDD PostgreSQL 🐳

H2 ne se comporte pas exactement comme PostgreSQL (types, fonctions SQL...). **Testcontainers** lance un vrai PostgreSQL dans Docker le temps du test :

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-testcontainers</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>postgresql</artifactId>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>org.testcontainers</groupId>
    <artifactId>junit-jupiter</artifactId>
    <scope>test</scope>
</dependency>
```

```java
@DataJpaTest
@AutoConfigureTestDatabase(replace = AutoConfigureTestDatabase.Replace.NONE) // pas de H2 !
@Testcontainers
class ProductRepositoryPostgresTest {

    @Container
    @ServiceConnection    // Spring Boot 3.1+ : configure automatiquement la datasource !
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16");

    @Autowired
    private ProductJpaRepository repository;

    @Test
    void save_shouldPersistInRealPostgres() {
        ProductEntity saved = repository.save(
            ProductEntity.builder().name("TV").price(new BigDecimal("499")).build());

        assertThat(saved.getId()).isNotNull();
        assertThat(repository.findById(saved.getId())).isPresent();
    }
}
```

> 🏆 **Recommandation** : `@DataJpaTest` + H2 pour les tests rapides du quotidien, Testcontainers pour valider les requêtes SQL spécifiques et dans la CI avant la mise en production.

### `@Sql` — préparer la BDD avec des scripts

```java
@Test
@Sql("/sql/insert-products.sql")            // exécuté avant le test
@Sql(scripts = "/sql/cleanup.sql", executionPhase = Sql.ExecutionPhase.AFTER_TEST_METHOD)
void findAll_shouldReturnSeededData() {
    assertThat(repository.findAll()).hasSize(5);
}
```

---

## 16. Tests de charge

**Objectif** : mesurer le comportement de l'API sous trafic (temps de réponse, débit, taux d'erreur) et trouver le point de rupture. Ils s'exécutent contre une **application déployée** (pas dans JUnit).

### Vocabulaire

| Terme | Définition |
|---|---|
| **Load test** | Trafic attendu normal (ex: 100 utilisateurs simultanés) |
| **Stress test** | Monter jusqu'au point de rupture |
| **Spike test** | Pic brutal de trafic |
| **Soak test** | Charge modérée sur longue durée (fuites mémoire) |
| **Latence p95/p99** | 95%/99% des requêtes répondent en moins de X ms |
| **Throughput** | Requêtes traitées par seconde (req/s) |

### Option 1 — JMeter (interface graphique)

1. Télécharger [Apache JMeter](https://jmeter.apache.org)
2. Créer un **Thread Group** : 100 threads (utilisateurs), ramp-up 10s, boucle 50 fois
3. Ajouter un **HTTP Request** : `GET http://localhost:8080/api/v1/products`
4. Ajouter un **HTTP Header Manager** si JWT : `Authorization: Bearer ...`
5. Ajouter des **Listeners** : Summary Report, Aggregate Report
6. Lancer en ligne de commande pour les vrais tests (le mode GUI fausse les mesures) :

```bash
jmeter -n -t mon-plan.jmx -l resultats.jtl -e -o rapport-html/
```

### Option 2 — Gatling (code Java, recommandé pour les devs)

```xml
<dependency>
    <groupId>io.gatling.highcharts</groupId>
    <artifactId>gatling-charts-highcharts</artifactId>
    <version>3.13.1</version>
    <scope>test</scope>
</dependency>
```

```java
public class ProductLoadSimulation extends Simulation {

    HttpProtocolBuilder httpProtocol = http
        .baseUrl("http://localhost:8080")
        .acceptHeader("application/json");

    ScenarioBuilder scn = scenario("Charge sur les produits")
        .exec(http("GET produits")
            .get("/api/v1/products")
            .check(status().is(200)))
        .pause(1)
        .exec(http("POST produit")
            .post("/api/v1/products")
            .header("Content-Type", "application/json")
            .body(StringBody("""
                {"name":"TV","description":"4K","price":499}
                """))
            .check(status().is(201)));

    {
        setUp(
            scn.injectOpen(
                rampUsers(100).during(Duration.ofSeconds(30)),        // montée progressive
                constantUsersPerSec(50).during(Duration.ofMinutes(2)) // charge constante
            )
        ).protocols(httpProtocol)
         .assertions(
            global().responseTime().percentile(95).lt(500),   // p95 < 500ms
            global().successfulRequests().percent().gt(99.0)  // > 99% de succès
         );
    }
}
```

Gatling génère un **rapport HTML** détaillé (latences, percentiles, req/s).

### Bonnes pratiques

- Tester sur un environnement **proche de la prod** (pas ton laptop avec la BDD locale)
- Définir des **objectifs chiffrés avant** (ex: p95 < 300ms à 200 req/s)
- Surveiller côté serveur pendant le test : CPU, mémoire, pool de connexions BDD (Actuator + Prometheus/Grafana)
- Ne jamais lancer un test de charge sur la production sans autorisation !

---

## 17. BDD — Cucumber

> ℹ️ Ne pas confondre : **BDD** = base de données (§15) vs **BDD** = *Behavior Driven Development*, une méthode où les tests sont écrits en langage naturel compréhensible par les non-développeurs. Cette section couvre le second.

### Principe

Les scénarios sont écrits en **Gherkin** (Étant donné / Quand / Alors), puis reliés à du code Java :

```gherkin
# src/test/resources/features/otp.feature
# language: fr
Fonctionnalité: Authentification par OTP

  Scénario: Connexion réussie avec un code valide
    Étant donné un utilisateur avec le numéro "+33612345678"
    Quand il demande un code OTP
    Et il saisit le code reçu
    Alors il reçoit un token JWT valide

  Scénario: Échec avec un code invalide
    Étant donné un utilisateur avec le numéro "+33612345678"
    Quand il demande un code OTP
    Et il saisit le code "000000"
    Alors il reçoit une erreur "Code invalide"
```

### Mise en place

```xml
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-java</artifactId>
    <version>7.20.1</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-spring</artifactId>
    <version>7.20.1</version>
    <scope>test</scope>
</dependency>
<dependency>
    <groupId>io.cucumber</groupId>
    <artifactId>cucumber-junit-platform-engine</artifactId>
    <version>7.20.1</version>
    <scope>test</scope>
</dependency>
```

```java
// Le "glue code" relie le Gherkin au Java :
@CucumberContextConfiguration
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class OtpStepDefinitions {

    @Autowired private TestRestTemplate restTemplate;
    @MockitoBean private OtpSenderPort otpSender;

    private String phone;
    private ResponseEntity<Map> lastResponse;

    @Étantdonné("un utilisateur avec le numéro {string}")
    public void unUtilisateurAvecLeNumero(String phone) {
        this.phone = phone;
    }

    @Quand("il demande un code OTP")
    public void ilDemandeUnCodeOtp() {
        lastResponse = restTemplate.postForEntity("/api/v1/auth/otp/request",
            Map.of("phone", phone), Map.class);
    }

    @Alors("il reçoit un token JWT valide")
    public void ilRecoitUnTokenJwt() {
        assertThat(lastResponse.getBody()).containsKey("accessToken");
    }
}
```

**Quand utiliser Cucumber ?** Quand les scénarios métier doivent être lus/validés par des non-techniques (Product Owner, métier). Sinon, des tests d'intégration classiques bien nommés suffisent.

---

## 18. Récapitulatif

### Quelle annotation de test pour quel besoin ?

| Je veux tester... | Annotation | BDD ? | Beans chargés | Vitesse |
|---|---|---|---|---|
| Une classe métier isolée | `@ExtendWith(MockitoExtension.class)` | ❌ | aucun | ⚡⚡⚡ |
| La sérialisation JSON d'un DTO | `@JsonTest` | ❌ | Jackson | ⚡⚡⚡ |
| Un controller + validation + codes HTTP | `@WebMvcTest(X.class)` + `MockMvc` | ❌ | couche web | ⚡⚡ |
| Un repository + requêtes SQL | `@DataJpaTest` | ✅ H2 | couche JPA | ⚡⚡ |
| Requêtes SQL sur vraie BDD | `@DataJpaTest` + Testcontainers | ✅ Docker | couche JPA | ⚡ |
| Plusieurs couches ensemble | `@SpringBootTest` | ✅ | TOUT | 🐢 |
| L'API en HTTP réel de bout en bout | `@SpringBootTest(RANDOM_PORT)` + `TestRestTemplate` | ✅ | TOUT | 🐢 |
| La performance sous charge | JMeter / Gatling | ✅ réelle | app déployée | 🐢🐢 |

### Les mocks : quel outil dans quel contexte ?

| Contexte | Outil |
|---|---|
| Test unitaire pur (sans Spring) | `@Mock` + `@InjectMocks` |
| Test avec contexte Spring, remplacer un bean | `@MockitoBean` (ex-`@MockBean`, déprécié depuis Spring Boot 3.4) |
| Test avec contexte Spring, espionner un vrai bean | `@MockitoSpyBean` (ex-`@SpyBean`) |

### Règles d'or

1. **Beaucoup d'unitaires, peu d'intégration, encore moins d'E2E** (pyramide)
2. Un test = **un comportement** vérifié, nommé clairement (`methode_resultatAttendu_quandCondition`)
3. Pattern **AAA** : Arrange / Act / Assert, toujours
4. Ne jamais dépendre de l'ordre d'exécution des tests
5. Mock les dépendances **externes** (SMS, mail, API), jamais la classe testée
6. Les tests doivent tourner **dans la CI** à chaque push
7. Vise la couverture des **comportements critiques**, pas un % de couverture aveugle

---

*Guide complémentaire du guide « API, Clean Architecture & Microservices ». À consulter à chaque projet Spring Boot.*
