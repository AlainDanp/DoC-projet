# Guide de Configuration Gmail App Password

## Pourquoi App Password?

Gmail a désactivé l'accès "moins sécurisé" pour les applications tierces. Vous devez maintenant utiliser un **App Password** (mot de passe d'application) pour l'authentification SMTP.

## Étapes pour créer un App Password Gmail

### 1. Activer la validation en 2 étapes (si pas déjà fait)

1. Allez sur: https://myaccount.google.com/security
2. Dans la section "Comment vous connecter à Google", cliquez sur **Validation en 2 étapes**
3. Suivez les instructions pour l'activer (SMS, application Google Authenticator, etc.)

### 2. Générer un App Password

1. Une fois la validation en 2 étapes activée, retournez sur: https://myaccount.google.com/security
2. Cliquez sur **Validation en 2 étapes**
3. Faites défiler jusqu'en bas et cliquez sur **Mots de passe d'application** (App passwords)
   - Lien direct: https://myaccount.google.com/apppasswords
4. Connectez-vous si nécessaire
5. Dans le champ "Sélectionner l'application", choisissez **Autre (nom personnalisé)**
6. Entrez un nom comme: **TKD Backend Spring Boot**
7. Cliquez sur **Générer**
8. Gmail affichera un mot de passe de 16 caractères comme: `abcd efgh ijkl mnop`
9. **COPIEZ CE MOT DE PASSE IMMÉDIATEMENT** (vous ne pourrez plus le voir après)

### 3. Configurer le fichier .env

1. Ouvrez le fichier `.env` à la racine de votre projet
2. Remplacez la ligne:
   ```
   MAIL_PASSWORD=REMPLACER_PAR_VOTRE_APP_PASSWORD
   ```
   Par:
   ```
   MAIL_PASSWORD=abcdefghijklmnop
   ```
   **IMPORTANT:** Supprimez les espaces du mot de passe Gmail (16 caractères sans espace)

### 4. Exemple de configuration complète

```env
MAIL_HOST=smtp.gmail.com
MAIL_PORT=587
MAIL_USERNAME=pauldanp416@gmail.com
MAIL_PASSWORD=abcdefghijklmnop
```

### 5. Redémarrer l'application

Après avoir configuré le `.env`:

```bash
# Arrêter l'application (Ctrl+C)
# Puis redémarrer
mvn spring-boot:run
```

Ou si vous utilisez votre IDE, arrêtez et redémarrez l'application.

## Vérification

Testez en créant un nouvel utilisateur. Vous devriez recevoir l'email de bienvenue sans erreur d'authentification.

## Dépannage

### Erreur: "Application-specific password required"
- Vous n'avez pas créé d'App Password ou utilisez votre mot de passe normal
- Solution: Créez un App Password comme décrit ci-dessus

### Erreur: "Username and Password not accepted"
- L'App Password est incorrect
- Vérifiez qu'il n'y a pas d'espaces dans le mot de passe
- Régénérez un nouveau App Password

### L'application ne charge pas le fichier .env
- Vérifiez que la dépendance `spring-dotenv` est dans votre `pom.xml`
- Exécutez: `mvn clean install`
- Redémarrez l'application

### Le fichier .env est-il bien placé?
Le fichier `.env` doit être à la racine du projet:
```
backend_tdk/
├── .env           ← ICI
├── pom.xml
├── src/
└── ...
```

## Sécurité

- Le fichier `.env` est déjà dans `.gitignore`
- Ne partagez JAMAIS votre App Password
- Si compromis, révoquez-le et créez-en un nouveau sur: https://myaccount.google.com/apppasswords
