# Configuration MailHog pour le Développement

## Pourquoi MailHog?

MailHog est un serveur SMTP de test qui intercepte tous les emails envoyés par votre application. Parfait pour le développement quand:
- Votre pare-feu bloque les ports SMTP (587/465)
- Vous ne voulez pas envoyer de vrais emails pendant les tests
- Vous voulez voir tous les emails dans une interface web

## Installation MailHog

### Option 1: Téléchargement direct (Le plus simple)

1. Allez sur: https://github.com/mailhog/MailHog/releases/latest
2. Téléchargez: `MailHog_windows_amd64.exe`
3. Renommez le fichier en `mailhog.exe`
4. Placez-le dans un dossier (ex: `C:\Tools\mailhog.exe`)
5. Double-cliquez pour lancer MailHog

### Option 2: Avec Chocolatey

```powershell
choco install mailhog
```

### Option 3: Avec Scoop

```powershell
scoop install mailhog
```

## Lancer MailHog

Double-cliquez sur `mailhog.exe` ou dans le terminal:

```bash
mailhog.exe
```

Vous verrez:
```
[HTTP] Binding to address: 0.0.0.0:8025
[SMTP] Binding to address: 0.0.0.0:1025
```

**Interface Web:** http://localhost:8025
**Serveur SMTP:** localhost:1025

Laissez cette fenêtre ouverte pendant le développement.

## Configuration de votre application

### Étape 1: Modifier votre fichier `.env`

Ouvrez `.env` et modifiez la section EMAIL:

```env
# ==============================================
# EMAIL CONFIGURATION (MailHog - Développement)
# ==============================================
MAIL_HOST=localhost
MAIL_PORT=1025
MAIL_USERNAME=
MAIL_PASSWORD=
```

**IMPORTANT:** Laissez `MAIL_USERNAME` et `MAIL_PASSWORD` vides pour MailHog

### Étape 2: Modifier `application.properties`

Ouvrez `src/main/resources/application.properties` et modifiez:

```properties
spring.mail.host=${MAIL_HOST:localhost}
spring.mail.port=${MAIL_PORT:1025}
spring.mail.username=${MAIL_USERNAME:}
spring.mail.password=${MAIL_PASSWORD:}

# Désactiver l'authentification pour MailHog
spring.mail.properties.mail.smtp.auth=false
spring.mail.properties.mail.smtp.ssl.enable=false
spring.mail.properties.mail.smtp.starttls.enable=false

# Augmenter les timeouts
spring.mail.properties.mail.smtp.connectiontimeout=30000
spring.mail.properties.mail.smtp.timeout=30000
spring.mail.properties.mail.smtp.writetimeout=30000
```

### Étape 3: Redémarrer votre application Spring Boot

Arrêtez et relancez votre application pour charger les nouvelles configurations.

## Utilisation

1. **Lancez MailHog**: `mailhog.exe`
2. **Lancez votre application Spring Boot**
3. **Testez "Mot de passe oublié"** depuis votre frontend
4. **Ouvrez MailHog**: http://localhost:8025
5. **Consultez l'email** reçu dans l'interface MailHog

## Interface MailHog

L'interface web (http://localhost:8025) affiche:
- Tous les emails interceptés
- L'expéditeur et le destinataire
- Le sujet de l'email
- Le contenu HTML et texte
- Les headers complets

Vous pouvez:
- Voir l'email en HTML ou texte brut
- Supprimer des emails
- Télécharger les emails

## Tester "Mot de passe oublié"

1. Dans votre frontend Angular, allez sur la page "Mot de passe oublié"
2. Entrez un email d'utilisateur existant
3. Cliquez sur "Envoyer"
4. Ouvrez MailHog (http://localhost:8025)
5. Vous verrez l'email de réinitialisation
6. Cliquez sur le lien dans l'email
7. Réinitialisez le mot de passe

## Revenir à Gmail (Production)

Quand vous voulez utiliser Gmail en production, modifiez votre `.env`:

```env
# ==============================================
# EMAIL CONFIGURATION (Gmail SMTP)
# ==============================================
MAIL_HOST=smtp.gmail.com
MAIL_PORT=465
MAIL_USERNAME=pauldanp416@gmail.com
MAIL_PASSWORD=votre-app-password-gmail
```

Et dans `application.properties`:

```properties
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.ssl.enable=true
```

## Avantages de MailHog

- ✅ Pas besoin de configuration Gmail
- ✅ Pas de blocage pare-feu
- ✅ Voir tous les emails dans une interface web
- ✅ Pas d'envoi accidentel d'emails aux utilisateurs
- ✅ Tester les templates d'email facilement
- ✅ Rapide et léger

## Dépannage

### MailHog ne démarre pas
- Vérifiez que le port 1025 n'est pas déjà utilisé
- Vérifiez que le port 8025 n'est pas déjà utilisé

### L'application ne se connecte pas à MailHog
- Vérifiez que MailHog est bien lancé
- Vérifiez que `MAIL_HOST=localhost` et `MAIL_PORT=1025` dans `.env`
- Redémarrez votre application Spring Boot

### Les emails n'apparaissent pas dans MailHog
- Vérifiez les logs de votre application Spring Boot
- Vérifiez que `spring.mail.properties.mail.smtp.auth=false` dans application.properties
- Vérifiez que `MAIL_USERNAME` et `MAIL_PASSWORD` sont vides dans `.env`

## Résumé des commandes

```bash
# Lancer MailHog
mailhog.exe

# Voir l'interface web
# Ouvrir http://localhost:8025 dans le navigateur

# Arrêter MailHog
# Ctrl+C dans la fenêtre du terminal
```

Voilà! Vous pouvez maintenant tester l'envoi d'emails sans blocage réseau.
