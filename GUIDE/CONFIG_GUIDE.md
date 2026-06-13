# Guide de Configuration Simplifié

## Structure du projet

Votre projet utilise maintenant une configuration simple et claire:

```
├── .env                          # ← Toutes vos variables d'environnement
├── src/main/resources/
│   └── application.properties    # ← Configuration Spring Boot
└── pom.xml                       # ← Dépendance spring-dotenv
```

## Fichiers de configuration

### 1. `.env` (Valeurs spécifiques à votre environnement)

Le fichier `.env` contient **toutes vos valeurs sensibles** et spécifiques à votre environnement:
- Mots de passe de base de données
- Clés API (Stripe, Orange Money, MTN)
- Configuration email
- Secrets JWT

**IMPORTANT:** Ce fichier est dans `.gitignore` et ne sera jamais commité dans Git.

### 2. `application.properties` (Configuration Spring Boot)

Le fichier `application.properties` contient:
- Les noms des propriétés Spring Boot
- Les références aux variables d'environnement du `.env`
- Les valeurs par défaut pour le développement

Format: `propriété=${VARIABLE_ENV:valeur_par_défaut}`

## Configuration Email

### Développement (MailHog - Par défaut)

Le projet est configuré par défaut pour utiliser **MailHog**, un serveur SMTP de test local.

**Avantages:**
- ✅ Pas de blocage pare-feu/réseau
- ✅ Voir tous les emails dans une interface web
- ✅ Pas d'envoi réel d'emails
- ✅ Rapide et facile

**Configuration dans `.env`:**
```env
MAIL_HOST=localhost
MAIL_PORT=1025
MAIL_USERNAME=
MAIL_PASSWORD=
MAIL_SMTP_AUTH=false
MAIL_SMTP_SSL=false
MAIL_SMTP_STARTTLS=false
```

**Installation MailHog:**
1. Téléchargez: https://github.com/mailhog/MailHog/releases/latest
2. Fichier: `MailHog_windows_amd64.exe`
3. Renommez en `mailhog.exe`
4. Double-cliquez pour lancer
5. Ouvrez: http://localhost:8025

### Production (Gmail)

Pour utiliser Gmail en production:

1. **Créez un App Password Gmail:**
   - Allez sur: https://myaccount.google.com/apppasswords
   - Générez un mot de passe d'application
   - Copiez le mot de passe (16 caractères sans espaces)

2. **Modifiez `.env`:**
   ```env
   # Commentez la configuration MailHog
   # MAIL_HOST=localhost
   # MAIL_PORT=1025
   # MAIL_USERNAME=
   # MAIL_PASSWORD=
   # MAIL_SMTP_AUTH=false
   # MAIL_SMTP_SSL=false
   # MAIL_SMTP_STARTTLS=false

   # Décommentez la configuration Gmail
   MAIL_HOST=smtp.gmail.com
   MAIL_PORT=465
   MAIL_USERNAME=pauldanp416@gmail.com
   MAIL_PASSWORD=votre-app-password-ici
   MAIL_SMTP_AUTH=true
   MAIL_SMTP_SSL=true
   MAIL_SMTP_STARTTLS=false
   ```

3. **Redémarrez l'application**

## Utilisation

### Démarrer l'application

1. **Lancez MailHog** (pour le développement):
   ```bash
   mailhog.exe
   ```

2. **Lancez l'application Spring Boot**:
   ```bash
   mvnw.cmd spring-boot:run
   ```

### Tester "Mot de passe oublié"

1. Allez sur votre frontend Angular
2. Cliquez sur "Mot de passe oublié"
3. Entrez un email
4. **Avec MailHog:** Ouvrez http://localhost:8025 pour voir l'email
5. **Avec Gmail:** Vérifiez votre boîte email

## Modifier une configuration

### Changer une valeur:

1. Ouvrez le fichier `.env`
2. Modifiez la valeur de la variable
3. Redémarrez l'application

### Ajouter une nouvelle variable:

1. **Ajoutez dans `.env`:**
   ```env
   MA_NOUVELLE_VARIABLE=ma_valeur
   ```

2. **Utilisez dans `application.properties`:**
   ```properties
   ma.propriete=${MA_NOUVELLE_VARIABLE:valeur_par_defaut}
   ```

3. Redémarrez l'application

## Dépannage

### L'application n'utilise pas les valeurs du .env

**Solution:**
1. Vérifiez que `spring-dotenv` est dans `pom.xml`:
   ```xml
   <dependency>
       <groupId>me.paulschwarz</groupId>
       <artifactId>spring-dotenv</artifactId>
       <version>4.0.0</version>
   </dependency>
   ```

2. Installez les dépendances:
   ```bash
   mvnw.cmd clean install
   ```

3. Redémarrez l'application

### Les emails ne sont pas envoyés

**Avec MailHog:**
- Vérifiez que `mailhog.exe` est lancé
- Ouvrez http://localhost:8025 pour voir les emails
- Vérifiez que `MAIL_HOST=localhost` et `MAIL_PORT=1025` dans `.env`

**Avec Gmail:**
- Vérifiez que vous utilisez un App Password (pas votre mot de passe Gmail)
- Vérifiez que les ports 465 ou 587 ne sont pas bloqués par votre pare-feu
- Vérifiez que `MAIL_SMTP_AUTH=true` et `MAIL_SMTP_SSL=true` dans `.env`

### Erreur "Couldn't connect to host"

C'est un problème de pare-feu/réseau:
- **Pour Gmail:** Débloquez les ports 465/587 dans votre pare-feu
- **Solution rapide:** Utilisez MailHog pour le développement

## Résumé

- ✅ **1 fichier `.env`** → Toutes vos variables sensibles
- ✅ **1 fichier `application.properties`** → Configuration Spring Boot
- ✅ **MailHog par défaut** → Pas de problème de réseau
- ✅ **Facile à basculer** → Commentez/décommentez dans `.env`
- ✅ **Sécurisé** → `.env` dans `.gitignore`

C'est tout! Simple et efficace.
