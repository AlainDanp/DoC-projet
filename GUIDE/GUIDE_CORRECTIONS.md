# Guide de corrections — Backend Hotel

Ce guide couvre **3 corrections prioritaires** à apporter avant la démo. Chaque section explique le problème, montre le code actuel défaillant, puis donne le code corrigé complet avec explications.

---

## Table des matières

1. [Correction 2 — Race condition des réservations + vérification des dates](#correction-2)
2. [Correction 3 — Envoi de l'email de vérification](#correction-3)
3. [Correction 4 — Ajout de @Valid sur les endpoints manquants](#correction-4)

---

## Correction 2 — Race condition des réservations + vérification des dates {#correction-2}

### Le problème

Actuellement `ReservationService.createReservation()` fait ceci :

```java
// ReservationService.java — lignes 49-53 (CODE ACTUEL BUGUÉ)
List<Room> availableRooms = roomRepository.findByTypeAndAvailable(roomType, true);
if (availableRooms.isEmpty()) {
    throw new BadRequestException("Aucune chambre disponible...");
}
Room room = availableRooms.get(0);
```

**Deux bugs critiques :**

1. **Race condition** : si deux utilisateurs envoient simultanément une demande pour une chambre SUITE, les deux passent le `if (availableRooms.isEmpty())` avant qu'aucun des deux ait sauvegardé. Résultat : deux réservations actives pour la même chambre.

2. **Pas de vérification de dates** : le flag `available = false` est permanent. Si une chambre est réservée du 1er au 5 juin, elle reste `available = false` pour toujours — même après le 5 juin. De plus, une chambre disponible peut être réservée deux fois sur des plages qui se chevauchent.

3. **Validation incomplète** : `checkOut` n'a pas `@Future` dans le DTO, et la capacité de la chambre n'est jamais comparée au nombre de guests.

---

### Étape 1 — Ajouter la requête de chevauchement dans `ReservationRepository`

**Fichier :** `src/main/java/com/danp/backend_hotel/repository/ReservationRepository.java`

Remplacez le contenu entier par :

```java
package com.danp.backend_hotel.repository;

import com.danp.backend_hotel.entity.Reservation;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.time.LocalDate;
import java.util.List;
import java.util.Optional;

@Repository
public interface ReservationRepository extends JpaRepository<Reservation, Long> {

    List<Reservation> findByUserId(Long userId);

    List<Reservation> findByStatus(Reservation.ReservationStatus status);

    Optional<Reservation> findByReservationCode(String reservationCode);

    List<Reservation> findByUserIdAndStatus(Long userId, Reservation.ReservationStatus status);

    /**
     * Vérifie s'il existe une réservation ACTIVE (non annulée) pour cette chambre
     * dont les dates chevauchent la plage demandée.
     *
     * La condition de chevauchement est : checkIn_existant < checkOut_demandé
     *                                 ET checkOut_existant > checkIn_demandé
     */
    @Query("""
        SELECT COUNT(r) > 0 FROM Reservation r
        WHERE r.room.id = :roomId
          AND r.status <> com.danp.backend_hotel.entity.Reservation.ReservationStatus.CANCELLED
          AND r.checkIn  < :checkOut
          AND r.checkOut > :checkIn
    """)
    boolean existsOverlappingReservation(
            @Param("roomId")   String    roomId,
            @Param("checkIn")  LocalDate checkIn,
            @Param("checkOut") LocalDate checkOut
    );
}
```

> **Pourquoi cette formule ?**
> Deux plages [A, B] et [C, D] se chevauchent si et seulement si `A < D ET C < B`.
> Si une réservation existante finit avant que la nouvelle commence (`checkOut_existant <= checkIn_demandé`), il n'y a pas de conflit. Idem si elle commence après la fin de la nouvelle.

---

### Étape 2 — Ajouter `@Version` sur l'entité `Room` (verrouillage optimiste)

**Fichier :** `src/main/java/com/danp/backend_hotel/entity/Room.java`

Ajoutez le champ `@Version` après la déclaration de `@Id` :

```java
// Ajout à faire dans Room.java, après la ligne : private String id;
@Version
private Long version;
```

Le bloc `@Id` + nouveau champ doit ressembler à :

```java
@Entity
@Table(name = "rooms")
@Data
@NoArgsConstructor
public class Room {

    @Id
    private String id;

    @Version                  // ← NOUVEAU : verrouillage optimiste
    private Long version;

    private String name;
    // ... reste inchangé
```

> **Pourquoi ?** Avec `@Version`, si deux transactions lisent la même chambre simultanément et qu'une la modifie en premier, la seconde recevra une `OptimisticLockException` au moment du `save()`. Spring la convertit en `409 Conflict` automatiquement.

---

### Étape 3 — Corriger la validation dans `ReservationRequest`

**Fichier :** `src/main/java/com/danp/backend_hotel/dto/ReservationRequest.java`

Ajoutez `@FutureOrPresent` sur `checkOut` (elle doit au minimum être aujourd'hui) :

```java
// Remplacez les deux champs de date par :

@NotNull
@Future
private LocalDate checkIn;

@NotNull
@FutureOrPresent   // ← NOUVEAU : checkOut ne peut pas être dans le passé
private LocalDate checkOut;
```

L'import à ajouter en tête de fichier :

```java
import jakarta.validation.constraints.FutureOrPresent;
```

---

### Étape 4 — Réécrire `ReservationService.createReservation()`

**Fichier :** `src/main/java/com/danp/backend_hotel/service/ReservationService.java`

Remplacez la méthode `createReservation` entière (lignes 33–74) par :

```java
public Reservation createReservation(ReservationRequest request, String userEmail) {

    // 1. Récupérer l'utilisateur
    User user = userRepository.findByEmail(userEmail)
            .orElseThrow(() -> new ResourceNotFoundException("Utilisateur non trouvé"));

    // 2. Validation des dates
    if (!request.getCheckOut().isAfter(request.getCheckIn())) {
        throw new BadRequestException("La date de départ doit être strictement après la date d'arrivée");
    }

    // 3. Résoudre le type de chambre
    Room.RoomType roomType;
    try {
        roomType = Room.RoomType.valueOf(request.getRoomType().toUpperCase());
    } catch (IllegalArgumentException e) {
        throw new BadRequestException("Type de chambre invalide : " + request.getRoomType()
                + ". Valeurs acceptées : STANDARD, DELUXE, SUITE, PRESIDENTIAL");
    }

    // 4. Trouver une chambre du bon type dont les dates sont libres
    //    On exclut les chambres qui ont déjà une réservation qui chevauche la plage demandée.
    List<Room> candidates = roomRepository.findByType(roomType);
    if (candidates.isEmpty()) {
        throw new BadRequestException("Aucune chambre de type " + request.getRoomType() + " n'existe");
    }

    Room room = candidates.stream()
            .filter(r -> !reservationRepository.existsOverlappingReservation(
                    r.getId(), request.getCheckIn(), request.getCheckOut()))
            .findFirst()
            .orElseThrow(() -> new BadRequestException(
                    "Aucune chambre disponible de type " + request.getRoomType()
                    + " pour les dates demandées"));

    // 5. Vérifier la capacité
    if (request.getGuests() != null && room.getCapacity() != null
            && request.getGuests() > room.getCapacity()) {
        throw new BadRequestException(
                "Cette chambre accepte au maximum " + room.getCapacity() + " personne(s)");
    }

    // 6. Calculer le montant total
    long nights = ChronoUnit.DAYS.between(request.getCheckIn(), request.getCheckOut());
    double totalAmount = room.getPricePerNight() * nights;

    // 7. Créer la réservation
    Reservation reservation = new Reservation();
    reservation.setUser(user);
    reservation.setRoom(room);
    reservation.setRoomType(room.getType().name());
    reservation.setCheckIn(request.getCheckIn());
    reservation.setCheckOut(request.getCheckOut());
    reservation.setGuests(request.getGuests());
    reservation.setTotalAmount(totalAmount);
    reservation.setServices(request.getServices());
    reservation.setSpecialRequests(request.getSpecialRequests());
    reservation.setStatus(Reservation.ReservationStatus.UPCOMING);

    // Note : on ne touche PLUS au flag available de la chambre.
    // La disponibilité est désormais calculée dynamiquement par les dates.

    return reservationRepository.save(reservation);
}
```

> **Changement important :** On ne modifie plus `room.setAvailable(false)`. La disponibilité est entièrement portée par les dates des réservations. Le flag `available` peut servir pour la maintenance (fermeture administrative), mais plus pour la logique de réservation.

---

### Étape 5 — Ajouter `findByType` dans `RoomRepository` (si absent)

**Fichier :** `src/main/java/com/danp/backend_hotel/repository/RoomRepository.java`

Vérifiez que cette méthode existe, sinon ajoutez-la :

```java
List<Room> findByType(Room.RoomType type);
```

---

### Résumé des fichiers modifiés (Correction 2)

| Fichier | Modification |
|---|---|
| `ReservationRepository.java` | + méthode `existsOverlappingReservation()` |
| `Room.java` | + champ `@Version private Long version` |
| `ReservationRequest.java` | + `@FutureOrPresent` sur `checkOut` |
| `ReservationService.java` | Réécriture de `createReservation()` |
| `RoomRepository.java` | Vérifier/ajouter `findByType()` |

---

---

## Correction 3 — Envoi de l'email de vérification {#correction-3}

### Le problème

Dans `AuthService.java` aux lignes 93–95 et 217–219 :

```java
// CODE ACTUEL — email de vérification jamais envoyé
try {
    emailService.sendWelcomeEmail(savedUser);
    // TODO: Envoyer email de vérification
    // emailService.sendEmailVerification(savedUser);  ← commenté, méthode inexistante
} catch (Exception e) {
    System.err.println("Failed to send welcome email: " + e.getMessage());
}
```

La méthode `sendEmailVerification()` n'existe pas dans `EmailService`. Il faut :
1. Créer le template HTML Thymeleaf
2. Ajouter la méthode dans `EmailService`
3. Décommenter les appels dans `AuthService`

---

### Étape 1 — Créer le template Thymeleaf

**Fichier à créer :** `src/main/resources/templates/email-verification.html`

```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org" lang="fr">
<head>
    <meta charset="UTF-8">
    <title>Vérification de votre email — DanP Hotel</title>
    <style>
        body { font-family: Arial, sans-serif; background-color: #f4f4f4; margin: 0; padding: 0; }
        .container { max-width: 600px; margin: 40px auto; background: #ffffff; border-radius: 8px; overflow: hidden; box-shadow: 0 2px 8px rgba(0,0,0,0.1); }
        .header { background-color: #1a1a2e; padding: 30px; text-align: center; }
        .header h1 { color: #ffffff; margin: 0; font-size: 24px; }
        .body { padding: 30px 40px; }
        .body p { color: #444444; line-height: 1.6; }
        .btn { display: inline-block; margin: 24px 0; padding: 14px 32px; background-color: #c9a84c; color: #ffffff; text-decoration: none; border-radius: 5px; font-weight: bold; font-size: 16px; }
        .footer { background-color: #f4f4f4; padding: 20px 40px; text-align: center; font-size: 12px; color: #888888; }
        .warning { color: #888888; font-size: 13px; }
    </style>
</head>
<body>
<div class="container">
    <div class="header">
        <h1>DanP Hotel</h1>
    </div>
    <div class="body">
        <p>Bonjour <strong th:text="${username}">Utilisateur</strong>,</p>
        <p>Merci de vous être inscrit sur <strong>DanP Hotel</strong>. Pour activer votre compte, veuillez vérifier votre adresse email en cliquant sur le bouton ci-dessous :</p>

        <div style="text-align: center;">
            <a th:href="${verificationUrl}" class="btn">Vérifier mon email</a>
        </div>

        <p class="warning">Ce lien expire dans <strong>24 heures</strong>.</p>
        <p class="warning">Si vous n'avez pas créé de compte sur DanP Hotel, ignorez cet email.</p>
        <p class="warning">
            Si le bouton ne fonctionne pas, copiez ce lien dans votre navigateur :<br>
            <span th:text="${verificationUrl}" style="word-break: break-all;"></span>
        </p>
    </div>
    <div class="footer">
        <p>&copy; 2025 DanP Hotel. Tous droits réservés.</p>
    </div>
</div>
</body>
</html>
```

---

### Étape 2 — Ajouter la méthode dans `EmailService`

**Fichier :** `src/main/java/com/danp/backend_hotel/service/EmailService.java`

Ajoutez cette méthode après `sendPasswordResetEmail()` (après la ligne 115) :

```java
@Async
public void sendEmailVerification(User user) {
    try {
        String verificationUrl = frontendUrl
                + "/verify-email?token="
                + user.getEmailVerificationToken();

        Map<String, Object> variables = new HashMap<>();
        variables.put("username", user.getUsername());
        variables.put("verificationUrl", verificationUrl);
        variables.put("frontendUrl", frontendUrl);

        EmailDTO email = EmailDTO.builder()
                .to(user.getEmail())
                .subject("Vérifiez votre adresse email — DanP Hotel")
                .template("email-verification")
                .variables(variables)
                .build();

        sendHtmlEmail(email);
    } catch (MessagingException | UnsupportedEncodingException e) {
        log.error("Erreur envoi email de vérification à {} : {}", user.getEmail(), e.getMessage(), e);
    }
}
```

---

### Étape 3 — Décommenter les appels dans `AuthService`

**Fichier :** `src/main/java/com/danp/backend_hotel/service/AuthService.java`

**Modification 1 — méthode `register()` (lignes 92–98) :**

Remplacez le bloc `try` par :

```java
try {
    emailService.sendWelcomeEmail(savedUser);
    emailService.sendEmailVerification(savedUser);   // ← décommenté
} catch (Exception e) {
    log.error("Erreur envoi email lors de l'inscription de {} : {}", savedUser.getEmail(), e.getMessage());
}
```

> Ajoutez l'import du logger en tête de classe si absent :
> ```java
> import org.slf4j.Logger;
> import org.slf4j.LoggerFactory;
> ```
> Et déclarez-le comme attribut de classe :
> ```java
> private static final Logger log = LoggerFactory.getLogger(AuthService.class);
> ```

**Modification 2 — méthode `resendVerificationEmail()` (lignes 217–222) :**

Remplacez le bloc `try` par :

```java
try {
    emailService.sendEmailVerification(user);   // ← décommenté
} catch (Exception e) {
    log.error("Erreur renvoi email de vérification à {} : {}", user.getEmail(), e.getMessage());
}
```

---

### Étape 4 — Vérifier la configuration `application.properties`

**Fichier :** `src/main/resources/application.properties`

Assurez-vous que ces propriétés sont présentes et correctes :

```properties
# URL du frontend (utilisée pour construire le lien de vérification)
app.frontend.url=${FRONTEND_URL:http://localhost:4200}

# Email expéditeur
app.email.from=${MAIL_USERNAME:noreply@danphotel.com}
app.email.from-name=DanP Hotel
```

> Si `app.frontend.url` n'existe pas encore, ajoutez-la. La notation `${VAR:valeur_par_defaut}` signifie : lire la variable d'environnement `FRONTEND_URL`, ou utiliser `http://localhost:4200` si absente.

---

### Test manuel de la correction 3

1. Lancez l'application et créez un compte via `POST /api/auth/register`
2. Vérifiez les logs : vous devez voir `Sending email to ...` sans erreur
3. Si vous utilisez un vrai compte Gmail configuré, l'email arrive avec le bouton de vérification
4. Cliquez le lien ou appelez `GET /api/auth/verify-email?token=<token>` — vous devez obtenir `{"message": "Email vérifié avec succès"}`
5. En base, le champ `is_email_verified` doit passer de `0` à `1`

---

### Résumé des fichiers modifiés (Correction 3)

| Fichier | Modification |
|---|---|
| `templates/email-verification.html` | Nouveau fichier — template HTML |
| `EmailService.java` | + méthode `sendEmailVerification()` |
| `AuthService.java` | Décommentage des 2 appels + remplacement `System.err` par logger |
| `application.properties` | Vérifier présence de `app.frontend.url` |

---

---

## Correction 4 — Ajout de `@Valid` sur les endpoints manquants {#correction-4}

### Le problème

Sans `@Valid`, Spring ne déclenche pas les annotations de validation (`@NotBlank`, `@Size`, `@Min`, etc.) sur les DTOs reçus. N'importe quelle valeur passe, y compris `null`, des prix négatifs, etc.

**Endpoints impactés :**

| Contrôleur | Méthode | Problème |
|---|---|---|
| `RoomController` | `POST /api/rooms` | Pas de `@Valid` — chambre créée sans prix ni nom |
| `RoomController` | `PUT /api/rooms/{id}` | Idem |
| `UserController` | `PUT /api/users/profile` | Pas de `@Valid` — mise à jour sans validation |

---

### Étape 1 — Ajouter des contraintes de validation sur `Room`

Avant d'ajouter `@Valid` dans le contrôleur, l'entité `Room` doit avoir des contraintes. Actuellement elle n'en a aucune.

**Fichier :** `src/main/java/com/danp/backend_hotel/entity/Room.java`

Ajoutez les imports suivants en tête du fichier :

```java
import jakarta.validation.constraints.DecimalMin;
import jakarta.validation.constraints.Min;
import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
```

Puis annotez les champs :

```java
@NotBlank(message = "Le nom de la chambre est obligatoire")
private String name;

@NotNull(message = "Le type de chambre est obligatoire")
@Enumerated(EnumType.STRING)
private RoomType type;

@NotNull(message = "Le prix par nuit est obligatoire")
@DecimalMin(value = "0.01", message = "Le prix doit être supérieur à 0")
private Double pricePerNight;

@NotNull(message = "La capacité est obligatoire")
@Min(value = 1, message = "La capacité doit être d'au moins 1 personne")
private Integer capacity;
```

> Les autres champs (`description`, `image`, `amenities`) restent optionnels.

---

### Étape 2 — Corriger `RoomController`

**Fichier :** `src/main/java/com/danp/backend_hotel/controller/RoomController.java`

Ajoutez l'import :

```java
import jakarta.validation.Valid;
```

Puis ajoutez `@Valid` sur les deux endpoints d'écriture :

```java
@PostMapping
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<Room> createRoom(@Valid @RequestBody Room room) {   // ← @Valid ajouté
    return ResponseEntity.status(HttpStatus.CREATED).body(roomService.createRoom(room));
}

@PutMapping("/{id}")
@PreAuthorize("hasRole('ADMIN')")
public ResponseEntity<Room> updateRoom(@PathVariable String id,
                                        @Valid @RequestBody Room room) {   // ← @Valid ajouté
    return ResponseEntity.ok(roomService.updateRoom(id, room));
}
```

---

### Étape 3 — Ajouter des contraintes sur `UserProfileDto`

**Fichier :** `src/main/java/com/danp/backend_hotel/dto/UserProfileDto.java`

Ajoutez les imports :

```java
import jakarta.validation.constraints.Pattern;
import jakarta.validation.constraints.Size;
```

Annotez les champs modifiables :

```java
@Size(max = 50, message = "Le prénom ne peut pas dépasser 50 caractères")
private String firstName;

@Size(max = 50, message = "Le nom ne peut pas dépasser 50 caractères")
private String lastName;

@Pattern(
    regexp = "^[+]?[0-9\\s\\-]{7,20}$",
    message = "Numéro de téléphone invalide"
)
private String phone;

@Size(max = 50, message = "La nationalité ne peut pas dépasser 50 caractères")
private String nationality;

@Size(max = 200, message = "L'adresse ne peut pas dépasser 200 caractères")
private String address;
```

> Les champs `id`, `email` et `createdAt` ne doivent pas être modifiables par l'utilisateur — ils sont en lecture seule côté contrôleur (on ne les applique pas depuis le DTO).

---

### Étape 4 — Corriger `UserController`

**Fichier :** `src/main/java/com/danp/backend_hotel/controller/UserController.java`

Ajoutez l'import si absent :

```java
import jakarta.validation.Valid;
```

Ajoutez `@Valid` sur `updateProfile` :

```java
@PutMapping("/profile")
public ResponseEntity<UserProfileDto> updateProfile(
        @AuthenticationPrincipal UserDetails userDetails,
        @Valid @RequestBody UserProfileDto dto) {   // ← @Valid ajouté
    // ... reste inchangé
}
```

---

### Étape 5 — Vérifier que le `GlobalExceptionHandler` gère bien `MethodArgumentNotValidException`

**Fichier :** `src/main/java/com/danp/backend_hotel/exception/GlobalExceptionHandler.java`

Ce handler doit déjà exister. Vérifiez qu'il ressemble bien à ceci (sinon complétez-le) :

```java
@ExceptionHandler(MethodArgumentNotValidException.class)
public ResponseEntity<Map<String, String>> handleValidationErrors(
        MethodArgumentNotValidException ex) {

    Map<String, String> errors = new LinkedHashMap<>();
    ex.getBindingResult().getFieldErrors()
            .forEach(err -> errors.put(err.getField(), err.getDefaultMessage()));

    return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(errors);
}
```

> Sans ce handler, Spring renverrait une page d'erreur HTML au lieu d'un JSON lisible.

---

### Test des validations

**Tester la création de chambre sans nom (doit échouer) :**

```http
POST /api/rooms
Authorization: Bearer <token_admin>
Content-Type: application/json

{
  "type": "SUITE",
  "pricePerNight": -50,
  "capacity": 0
}
```

**Réponse attendue (400 Bad Request) :**

```json
{
  "name": "Le nom de la chambre est obligatoire",
  "pricePerNight": "Le prix doit être supérieur à 0",
  "capacity": "La capacité doit être d'au moins 1 personne"
}
```

**Tester la mise à jour du profil avec un téléphone invalide :**

```http
PUT /api/users/profile
Authorization: Bearer <token>
Content-Type: application/json

{
  "phone": "abc-not-a-phone"
}
```

**Réponse attendue (400 Bad Request) :**

```json
{
  "phone": "Numéro de téléphone invalide"
}
```

---

### Résumé des fichiers modifiés (Correction 4)

| Fichier | Modification |
|---|---|
| `Room.java` | + annotations `@NotBlank`, `@NotNull`, `@DecimalMin`, `@Min` |
| `RoomController.java` | + `@Valid` sur `createRoom` et `updateRoom` |
| `UserProfileDto.java` | + annotations `@Size`, `@Pattern` |
| `UserController.java` | + `@Valid` sur `updateProfile` |
| `GlobalExceptionHandler.java` | Vérifier présence du handler `MethodArgumentNotValidException` |

---

## Récapitulatif global

| Correction | Fichiers touchés | Difficulté |
|---|---|---|
| **2** — Race condition réservations | 5 fichiers | Moyenne |
| **3** — Email de vérification | 4 fichiers + 1 nouveau template | Facile |
| **4** — Validation @Valid | 5 fichiers | Facile |

**Ordre recommandé :** Faire la 4 en premier (5 min), puis la 3 (20 min), puis la 2 (30 min).

La correction 2 est la plus importante fonctionnellement — sans elle, votre système de réservation peut produire des données incohérentes en conditions réelles.
