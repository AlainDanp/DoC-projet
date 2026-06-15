# Guide de Recréation du Projet TKD

> Ce guide contient toutes les commandes nécessaires pour recréer le projet Angular TKD depuis zéro.

---

## Table des Matières

1. [Création du Projet](#1-création-du-projet-angular)
2. [Installation des Dépendances](#2-installation-des-dépendances-npm)
3. [Génération des Composants](#3-génération-des-composants)
4. [Génération des Services](#4-génération-des-services)
5. [Génération des Guards](#5-génération-des-guards)
6. [Génération des Interceptors](#6-génération-des-interceptors)
7. [Création des Models](#7-création-des-models-interfaces)
8. [Génération de la Directive](#8-génération-de-la-directive-input)
9. [Création des Utilitaires](#9-création-des-fichiers-utilitaires)
10. [Configuration des Environments](#10-configuration-des-environments)
11. [Fichier ScrollReveal](#11-fichier-de-déclaration-typescript-pour-scrollreveal)
12. [Configuration angular.json](#12-configuration-de-angularjson)
13. [Configuration index.html](#13-configuration-de-indexhtml)
14. [Configuration app.module.ts](#14-configuration-de-appmodulets)
15. [Configuration du Routing](#15-configuration-du-routing)
16. [Script Complet](#16-script-complet-bash)

---

## 1. Création du Projet Angular

```bash
# Créer le nouveau projet Angular
ng new TKD --routing --style=css --ssr=false --skip-tests=false

# Se placer dans le dossier
cd TKD

# Configurer les schematics pour désactiver standalone par défaut
ng config projects.TKD.schematics.@schematics/angular:component.standalone false
ng config projects.TKD.schematics.@schematics/angular:directive.standalone false
ng config projects.TKD.schematics.@schematics/angular:pipe.standalone false
```

---

## 2. Installation des Dépendances NPM

### Dépendances Principales

```bash
npm install ngx-toastr@19.0.0
npm install swiper@11.2.10
npm install scrollreveal@4.0.9
npm install class-variance-authority@0.7.1
npm install clsx@2.1.1
npm install tailwind-merge@3.3.1
npm install tw-animate-css@1.4.0
npm install lucide-react@0.545.0
```

### Installation en une seule commande

```bash
npm install ngx-toastr@19.0.0 swiper@11.2.10 scrollreveal@4.0.9 class-variance-authority@0.7.1 clsx@2.1.1 tailwind-merge@3.3.1 tw-animate-css@1.4.0 lucide-react@0.545.0
```

### Résumé des Dépendances

| Package | Version | Description |
|---------|---------|-------------|
| `ngx-toastr` | 19.0.0 | Notifications toast |
| `swiper` | 11.2.10 | Carousel/Slider |
| `scrollreveal` | 4.0.9 | Animations au scroll |
| `class-variance-authority` | 0.7.1 | Gestion des variants CSS |
| `clsx` | 2.1.1 | Utilitaire pour classes CSS |
| `tailwind-merge` | 3.3.1 | Fusion des classes Tailwind |
| `tw-animate-css` | 1.4.0 | Animations Tailwind |
| `lucide-react` | 0.545.0 | Icônes |

---

## 3. Génération des Composants

### Composants de Pages (14 au total)

```bash
# Page d'accueil
ng generate component components/home

# Authentification
ng generate component components/login
ng generate component components/forgot-password
ng generate component components/reset-passowrd

# Layout
ng generate component components/layouts
ng generate component components/navbar
ng generate component components/footer

# Pages principales
ng generate component components/about
ng generate component components/shop
ng generate component components/option

# Pages secondaires
ng generate component components/faq
ng generate component components/support
ng generate component components/condition
ng generate component components/confidentiel
```

### Version abrégée

```bash
ng g c components/home
ng g c components/login
ng g c components/forgot-password
ng g c components/reset-passowrd
ng g c components/layouts
ng g c components/navbar
ng g c components/footer
ng g c components/about
ng g c components/shop
ng g c components/option
ng g c components/faq
ng g c components/support
ng g c components/condition
ng g c components/confidentiel
```

### Tableau Récapitulatif des Composants

| Composant | Chemin | Description |
|-----------|--------|-------------|
| HomeComponent | `components/home` | Page d'accueil avec Swiper et ScrollReveal |
| LoginComponent | `components/login` | Authentification (login & inscription) |
| ForgotPasswordComponent | `components/forgot-password` | Mot de passe oublié |
| ResetPassowrdComponent | `components/reset-passowrd` | Réinitialisation mot de passe |
| LayoutsComponent | `components/layouts` | Layout wrapper pour pages authentifiées |
| NavbarComponent | `components/navbar` | Barre de navigation |
| FooterComponent | `components/footer` | Pied de page |
| AboutComponent | `components/about` | Page À propos |
| ShopComponent | `components/shop` | Boutique e-commerce |
| OptionComponent | `components/option` | Dashboard utilisateur |
| FaqComponent | `components/faq` | Page FAQ |
| SupportComponent | `components/support` | Page de contact/support |
| ConditionComponent | `components/condition` | Conditions d'utilisation |
| ConfidentielComponent | `components/confidentiel` | Politique de confidentialité |

---

## 4. Génération des Services

### Services (7 au total)

```bash
ng generate service core/services/auth
ng generate service core/services/user
ng generate service core/services/product
ng generate service core/services/cart
ng generate service core/services/order
ng generate service core/services/category
ng generate service core/services/loading
```

### Version abrégée

```bash
ng g s core/services/auth
ng g s core/services/user
ng g s core/services/product
ng g s core/services/cart
ng g s core/services/order
ng g s core/services/category
ng g s core/services/loading
```

### Tableau Récapitulatif des Services

| Service | Chemin | Responsabilités |
|---------|--------|-----------------|
| AuthService | `core/services/auth` | Login, register, logout, gestion tokens |
| UserService | `core/services/user` | Profil, avatar, adresses |
| ProductService | `core/services/product` | CRUD produits, recherche |
| CartService | `core/services/cart` | Gestion du panier |
| OrderService | `core/services/order` | Gestion des commandes |
| CategoryService | `core/services/category` | CRUD catégories |
| LoadingService | `core/services/loading` | État de chargement global |

---

## 5. Génération des Guards

### Guards (3 guards dans un seul fichier)

```bash
ng generate guard core/guards/auth --functional
```

### Version abrégée

```bash
ng g guard core/guards/auth --functional
```

### Guards Disponibles

| Guard | Type | Description |
|-------|------|-------------|
| `authGuard` | CanActivateFn | Protège les routes authentifiées |
| `adminGuard` | CanActivateFn | Protège les routes admin |
| `publicGuard` | CanActivateFn | Protège les pages publiques (redirige si connecté) |

---

## 6. Génération des Interceptors

### Interceptors HTTP (3 au total)

```bash
ng generate interceptor core/interceptors/auth --functional
ng generate interceptor core/interceptors/error --functional
ng generate interceptor core/interceptors/loading --functional
```

### Version abrégée

```bash
ng g interceptor core/interceptors/auth --functional
ng g interceptor core/interceptors/error --functional
ng g interceptor core/interceptors/loading --functional
```

### Tableau Récapitulatif des Interceptors

| Interceptor | Chemin | Responsabilités |
|-------------|--------|-----------------|
| authInterceptor | `core/interceptors/auth` | Ajoute le token JWT aux requêtes |
| errorInterceptor | `core/interceptors/error` | Gestion globale des erreurs HTTP |
| loadingInterceptor | `core/interceptors/loading` | Gestion de l'état de chargement |

---

## 7. Création des Models (Interfaces)

### Création du dossier

```bash
mkdir -p src/app/core/models
```

### Fichiers à créer manuellement

| Fichier | Interfaces |
|---------|------------|
| `user.model.ts` | User, LoginRequest, RegisterRequest, LoginResponse, UpdateProfileRequest, ForgotPassWordRequest, RestPasswordRequest |
| `product.model.ts` | Product, ProductImage, Category, ProductSearchParams, ProductPage |
| `cart.model.ts` | Cart, CartItem, AddToCartRequest, UpdateCartItemRequest |
| `order.model.ts` | Order, OrderItem, CreateOrderRequest, OrderStatus (enum), PaymentMethod (enum) |
| `address.model.ts` | Address, CreateAddressRequest |
| `api-response.model.ts` | ApiResponse\<T\>, MessageResponse, ErrorResponse |

### Commandes de création des fichiers (Windows PowerShell)

```powershell
New-Item -ItemType File -Path "src/app/core/models/user.model.ts" -Force
New-Item -ItemType File -Path "src/app/core/models/product.model.ts" -Force
New-Item -ItemType File -Path "src/app/core/models/cart.model.ts" -Force
New-Item -ItemType File -Path "src/app/core/models/order.model.ts" -Force
New-Item -ItemType File -Path "src/app/core/models/address.model.ts" -Force
New-Item -ItemType File -Path "src/app/core/models/api-response.model.ts" -Force
```

### Commandes de création des fichiers (Bash/Linux/Mac)

```bash
touch src/app/core/models/user.model.ts
touch src/app/core/models/product.model.ts
touch src/app/core/models/cart.model.ts
touch src/app/core/models/order.model.ts
touch src/app/core/models/address.model.ts
touch src/app/core/models/api-response.model.ts
```

---

## 8. Génération de la Directive Input

### Directive

```bash
ng generate directive shared/components/input/input
```

### Version abrégée

```bash
ng g d shared/components/input/input
```

### Fichier variants à créer manuellement

```bash
# Windows PowerShell
New-Item -ItemType File -Path "src/app/shared/components/input/input.variants.ts" -Force

# Bash/Linux/Mac
touch src/app/shared/components/input/input.variants.ts
```

---

## 9. Création des Fichiers Utilitaires

### Création du dossier

```bash
mkdir -p src/app/shared/utils
```

### Fichiers à créer

| Fichier | Exports |
|---------|---------|
| `cn.ts` | `cn()` - Fusion des classes Tailwind |
| `merge-classes.ts` | `mergeClasses()`, `transform()`, `generateId()` |
| `number.ts` | `clamp()`, `roundToStep()`, `convertValueToPercentage()` |

### Commandes de création (Windows PowerShell)

```powershell
New-Item -ItemType File -Path "src/app/shared/utils/cn.ts" -Force
New-Item -ItemType File -Path "src/app/shared/utils/merge-classes.ts" -Force
New-Item -ItemType File -Path "src/app/shared/utils/number.ts" -Force
```

### Commandes de création (Bash/Linux/Mac)

```bash
touch src/app/shared/utils/cn.ts
touch src/app/shared/utils/merge-classes.ts
touch src/app/shared/utils/number.ts
```

---

## 10. Configuration des Environments

### Création du dossier

```bash
mkdir -p src/environments
```

### Fichiers à créer

```bash
# Windows PowerShell
New-Item -ItemType File -Path "src/environments/environment.ts" -Force
New-Item -ItemType File -Path "src/environments/environment.development.ts" -Force

# Bash/Linux/Mac
touch src/environments/environment.ts
touch src/environments/environment.development.ts
```

### Contenu de `environment.ts`

```typescript
export const environment = {
  production: true,
  apiUrl: 'http://localhost:9090/api'
};
```

### Contenu de `environment.development.ts`

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:9090/api'
};
```

---

## 11. Fichier de Déclaration TypeScript pour ScrollReveal

### Création du fichier

```bash
# Windows PowerShell
New-Item -ItemType File -Path "src/scrollreveal.d.ts" -Force

# Bash/Linux/Mac
touch src/scrollreveal.d.ts
```

### Contenu de `scrollreveal.d.ts`

```typescript
declare module 'scrollreveal' {
  export default function ScrollReveal(options?: any): any;
}
```

---

## 12. Configuration de angular.json

### Modifications à apporter

Dans `angular.json`, modifier la section `projects.TKD.architect.build.options.styles` :

```json
{
  "projects": {
    "TKD": {
      "architect": {
        "build": {
          "options": {
            "styles": [
              "node_modules/swiper/swiper-bundle.min.css",
              "src/styles.css"
            ]
          }
        }
      }
    }
  }
}
```

---

## 13. Configuration de index.html

### Liens CDN à ajouter dans `<head>`

```html
<!doctype html>
<html lang="fr">
<head>
  <meta charset="utf-8">
  <title>TKD</title>
  <base href="/">
  <meta name="viewport" content="width=device-width, initial-scale=1">
  <link rel="icon" type="image/x-icon" href="favicon.ico">

  <!-- Boxicons -->
  <link href='https://unpkg.com/boxicons@2.1.4/css/boxicons.min.css' rel='stylesheet'>

  <!-- Font Awesome -->
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.0/css/all.min.css">

  <!-- Swiper CSS (si non inclus via angular.json) -->
  <!-- <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/swiper@11/swiper-bundle.min.css"> -->

  <!-- Remix Icon -->
  <link href="https://cdn.jsdelivr.net/npm/remixicon@4.2.0/fonts/remixicon.css" rel="stylesheet">
</head>
<body>
  <app-root></app-root>
</body>
</html>
```

---

## 14. Configuration de app.module.ts

### Contenu complet

```typescript
import { NgModule } from '@angular/core';
import { BrowserModule } from '@angular/platform-browser';
import { BrowserAnimationsModule } from '@angular/platform-browser/animations';
import { FormsModule } from '@angular/forms';
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { ToastrModule } from 'ngx-toastr';

import { AppRoutingModule } from './app-routing.module';
import { AppComponent } from './app.component';

// Components
import { HomeComponent } from './components/home/home.component';
import { LoginComponent } from './components/login/login.component';
import { LayoutsComponent } from './components/layouts/layouts.component';
import { NavbarComponent } from './components/navbar/navbar.component';
import { FooterComponent } from './components/footer/footer.component';
import { AboutComponent } from './components/about/about.component';
import { ShopComponent } from './components/shop/shop.component';
import { OptionComponent } from './components/option/option.component';
import { FaqComponent } from './components/faq/faq.component';
import { SupportComponent } from './components/support/support.component';
import { ConditionComponent } from './components/condition/condition.component';
import { ConfidentielComponent } from './components/confidentiel/confidentiel.component';
import { ForgotPasswordComponent } from './components/forgot-password/forgot-password.component';
import { ResetPassowrdComponent } from './components/reset-passowrd/reset-passowrd.component';

// Interceptors
import { authInterceptor } from './core/interceptors/auth.interceptor';
import { errorInterceptor } from './core/interceptors/error.interceptor';
import { loadingInterceptor } from './core/interceptors/loading.interceptor';

@NgModule({
  declarations: [
    AppComponent,
    HomeComponent,
    LoginComponent,
    LayoutsComponent,
    NavbarComponent,
    FooterComponent,
    AboutComponent,
    ShopComponent,
    OptionComponent,
    FaqComponent,
    SupportComponent,
    ConditionComponent,
    ConfidentielComponent,
    ForgotPasswordComponent,
    ResetPassowrdComponent
  ],
  imports: [
    BrowserModule,
    BrowserAnimationsModule,
    AppRoutingModule,
    FormsModule,
    ToastrModule.forRoot({
      timeOut: 3000,
      positionClass: 'toast-top-right',
      preventDuplicates: true
    })
  ],
  providers: [
    provideHttpClient(withInterceptors([
      authInterceptor,
      loadingInterceptor,
      errorInterceptor
    ]))
  ],
  bootstrap: [AppComponent]
})
export class AppModule { }
```

---

## 15. Configuration du Routing

### Contenu de `app-routing.module.ts`

```typescript
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';

// Components
import { HomeComponent } from './components/home/home.component';
import { LoginComponent } from './components/login/login.component';
import { LayoutsComponent } from './components/layouts/layouts.component';
import { AboutComponent } from './components/about/about.component';
import { ShopComponent } from './components/shop/shop.component';
import { OptionComponent } from './components/option/option.component';
import { FaqComponent } from './components/faq/faq.component';
import { SupportComponent } from './components/support/support.component';
import { ConditionComponent } from './components/condition/condition.component';
import { ConfidentielComponent } from './components/confidentiel/confidentiel.component';
import { ForgotPasswordComponent } from './components/forgot-password/forgot-password.component';
import { ResetPassowrdComponent } from './components/reset-passowrd/reset-passowrd.component';

// Guards
import { authGuard, publicGuard } from './core/guards/auth.guard';

const routes: Routes = [
  // Redirection par défaut
  { path: '', redirectTo: 'login', pathMatch: 'full' },

  // Routes publiques (accessibles uniquement si NON connecté)
  { path: 'login', component: LoginComponent, canActivate: [publicGuard] },
  { path: 'forgot-password', component: ForgotPasswordComponent, canActivate: [publicGuard] },
  { path: 'reset-password', component: ResetPassowrdComponent, canActivate: [publicGuard] },

  // Routes protégées (accessibles uniquement si connecté)
  {
    path: '',
    component: LayoutsComponent,
    canActivate: [authGuard],
    canActivateChild: [authGuard],
    children: [
      { path: 'Home', component: HomeComponent },
      { path: 'About', component: AboutComponent },
      { path: 'Shop', component: ShopComponent },
      {
        path: 'Option',
        component: OptionComponent,
        children: [
          { path: '', redirectTo: 'FAQ', pathMatch: 'full' },
          { path: 'FAQ', component: FaqComponent },
          { path: 'contact-Support', component: SupportComponent }
        ]
      }
    ]
  },

  // Pages légales (peuvent être ajoutées aux routes protégées ou publiques selon besoin)
  { path: 'conditions', component: ConditionComponent },
  { path: 'confidentialite', component: ConfidentielComponent }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

---

## 16. Script Complet (Bash)

### Script pour Linux/Mac

```bash
#!/bin/bash

echo "=== Création du projet TKD ==="

# 1. Création du projet
ng new TKD --routing --style=css --ssr=false
cd TKD

# 2. Configuration schematics (désactiver standalone)
ng config projects.TKD.schematics.@schematics/angular:component.standalone false
ng config projects.TKD.schematics.@schematics/angular:directive.standalone false
ng config projects.TKD.schematics.@schematics/angular:pipe.standalone false

echo "=== Installation des dépendances ==="

# 3. Dépendances NPM
npm install ngx-toastr@19.0.0 swiper@11.2.10 scrollreveal@4.0.9 class-variance-authority@0.7.1 clsx@2.1.1 tailwind-merge@3.3.1 tw-animate-css@1.4.0 lucide-react@0.545.0

echo "=== Génération des composants ==="

# 4. Composants
ng g c components/home
ng g c components/login
ng g c components/forgot-password
ng g c components/reset-passowrd
ng g c components/layouts
ng g c components/navbar
ng g c components/footer
ng g c components/about
ng g c components/shop
ng g c components/option
ng g c components/faq
ng g c components/support
ng g c components/condition
ng g c components/confidentiel

echo "=== Génération des services ==="

# 5. Services
ng g s core/services/auth
ng g s core/services/user
ng g s core/services/product
ng g s core/services/cart
ng g s core/services/order
ng g s core/services/category
ng g s core/services/loading

echo "=== Génération des guards ==="

# 6. Guards
ng g guard core/guards/auth --functional

echo "=== Génération des interceptors ==="

# 7. Interceptors
ng g interceptor core/interceptors/auth --functional
ng g interceptor core/interceptors/error --functional
ng g interceptor core/interceptors/loading --functional

echo "=== Génération de la directive ==="

# 8. Directive
ng g d shared/components/input/input

echo "=== Création des dossiers et fichiers ==="

# 9. Models
mkdir -p src/app/core/models
touch src/app/core/models/user.model.ts
touch src/app/core/models/product.model.ts
touch src/app/core/models/cart.model.ts
touch src/app/core/models/order.model.ts
touch src/app/core/models/address.model.ts
touch src/app/core/models/api-response.model.ts

# 10. Utils
mkdir -p src/app/shared/utils
touch src/app/shared/utils/cn.ts
touch src/app/shared/utils/merge-classes.ts
touch src/app/shared/utils/number.ts

# 11. Input variants
touch src/app/shared/components/input/input.variants.ts

# 12. Environments
mkdir -p src/environments
touch src/environments/environment.ts
touch src/environments/environment.development.ts

# 13. ScrollReveal declaration
touch src/scrollreveal.d.ts

echo "=== Projet TKD créé avec succès ! ==="
echo "N'oubliez pas de :"
echo "  1. Copier le contenu des fichiers depuis le projet original"
echo "  2. Configurer angular.json pour Swiper CSS"
echo "  3. Ajouter les CDN dans index.html"
echo "  4. Configurer app.module.ts et app-routing.module.ts"
```

---

## 17. Script Complet (Windows PowerShell)

```powershell
Write-Host "=== Création du projet TKD ===" -ForegroundColor Green

# 1. Création du projet
ng new TKD --routing --style=css --ssr=false
Set-Location TKD

# 2. Configuration schematics
ng config projects.TKD.schematics.@schematics/angular:component.standalone false
ng config projects.TKD.schematics.@schematics/angular:directive.standalone false
ng config projects.TKD.schematics.@schematics/angular:pipe.standalone false

Write-Host "=== Installation des dépendances ===" -ForegroundColor Green

# 3. Dépendances NPM
npm install ngx-toastr@19.0.0 swiper@11.2.10 scrollreveal@4.0.9 class-variance-authority@0.7.1 clsx@2.1.1 tailwind-merge@3.3.1 tw-animate-css@1.4.0 lucide-react@0.545.0

Write-Host "=== Génération des composants ===" -ForegroundColor Green

# 4. Composants
ng g c components/home
ng g c components/login
ng g c components/forgot-password
ng g c components/reset-passowrd
ng g c components/layouts
ng g c components/navbar
ng g c components/footer
ng g c components/about
ng g c components/shop
ng g c components/option
ng g c components/faq
ng g c components/support
ng g c components/condition
ng g c components/confidentiel

Write-Host "=== Génération des services ===" -ForegroundColor Green

# 5. Services
ng g s core/services/auth
ng g s core/services/user
ng g s core/services/product
ng g s core/services/cart
ng g s core/services/order
ng g s core/services/category
ng g s core/services/loading

Write-Host "=== Génération des guards ===" -ForegroundColor Green

# 6. Guards
ng g guard core/guards/auth --functional

Write-Host "=== Génération des interceptors ===" -ForegroundColor Green

# 7. Interceptors
ng g interceptor core/interceptors/auth --functional
ng g interceptor core/interceptors/error --functional
ng g interceptor core/interceptors/loading --functional

Write-Host "=== Génération de la directive ===" -ForegroundColor Green

# 8. Directive
ng g d shared/components/input/input

Write-Host "=== Création des dossiers et fichiers ===" -ForegroundColor Green

# 9. Models
New-Item -ItemType Directory -Path "src/app/core/models" -Force
New-Item -ItemType File -Path "src/app/core/models/user.model.ts" -Force
New-Item -ItemType File -Path "src/app/core/models/product.model.ts" -Force
New-Item -ItemType File -Path "src/app/core/models/cart.model.ts" -Force
New-Item -ItemType File -Path "src/app/core/models/order.model.ts" -Force
New-Item -ItemType File -Path "src/app/core/models/address.model.ts" -Force
New-Item -ItemType File -Path "src/app/core/models/api-response.model.ts" -Force

# 10. Utils
New-Item -ItemType Directory -Path "src/app/shared/utils" -Force
New-Item -ItemType File -Path "src/app/shared/utils/cn.ts" -Force
New-Item -ItemType File -Path "src/app/shared/utils/merge-classes.ts" -Force
New-Item -ItemType File -Path "src/app/shared/utils/number.ts" -Force

# 11. Input variants
New-Item -ItemType File -Path "src/app/shared/components/input/input.variants.ts" -Force

# 12. Environments
New-Item -ItemType Directory -Path "src/environments" -Force
New-Item -ItemType File -Path "src/environments/environment.ts" -Force
New-Item -ItemType File -Path "src/environments/environment.development.ts" -Force

# 13. ScrollReveal declaration
New-Item -ItemType File -Path "src/scrollreveal.d.ts" -Force

Write-Host "=== Projet TKD créé avec succès ! ===" -ForegroundColor Green
Write-Host "N'oubliez pas de :" -ForegroundColor Yellow
Write-Host "  1. Copier le contenu des fichiers depuis le projet original"
Write-Host "  2. Configurer angular.json pour Swiper CSS"
Write-Host "  3. Ajouter les CDN dans index.html"
Write-Host "  4. Configurer app.module.ts et app-routing.module.ts"
```

---

## 18. Structure Finale du Projet

```
TKD/
├── src/
│   ├── app/
│   │   ├── app.component.ts|html|css
│   │   ├── app.module.ts
│   │   ├── app-routing.module.ts
│   │   ├── components/
│   │   │   ├── home/
│   │   │   ├── login/
│   │   │   ├── layouts/
│   │   │   ├── navbar/
│   │   │   ├── footer/
│   │   │   ├── about/
│   │   │   ├── shop/
│   │   │   ├── option/
│   │   │   ├── faq/
│   │   │   ├── support/
│   │   │   ├── condition/
│   │   │   ├── confidentiel/
│   │   │   ├── forgot-password/
│   │   │   └── reset-passowrd/
│   │   ├── core/
│   │   │   ├── guards/
│   │   │   │   └── auth.guard.ts
│   │   │   ├── interceptors/
│   │   │   │   ├── auth.interceptor.ts
│   │   │   │   ├── error.interceptor.ts
│   │   │   │   └── loading.interceptor.ts
│   │   │   ├── models/
│   │   │   │   ├── user.model.ts
│   │   │   │   ├── product.model.ts
│   │   │   │   ├── cart.model.ts
│   │   │   │   ├── order.model.ts
│   │   │   │   ├── address.model.ts
│   │   │   │   └── api-response.model.ts
│   │   │   └── services/
│   │   │       ├── auth.service.ts
│   │   │       ├── user.service.ts
│   │   │       ├── product.service.ts
│   │   │       ├── cart.service.ts
│   │   │       ├── order.service.ts
│   │   │       ├── category.service.ts
│   │   │       └── loading.service.ts
│   │   └── shared/
│   │       ├── components/
│   │       │   └── input/
│   │       │       ├── input.directive.ts
│   │       │       └── input.variants.ts
│   │       └── utils/
│   │           ├── cn.ts
│   │           ├── merge-classes.ts
│   │           └── number.ts
│   ├── environments/
│   │   ├── environment.ts
│   │   └── environment.development.ts
│   ├── assets/
│   ├── index.html
│   ├── main.ts
│   ├── styles.css
│   └── scrollreveal.d.ts
├── angular.json
├── package.json
├── tsconfig.json
├── tsconfig.app.json
└── tsconfig.spec.json
```

---

## Notes Importantes

1. **Après exécution du script**, vous devrez copier le contenu de chaque fichier depuis le projet original
2. **Les fichiers de configuration** (`angular.json`, `tsconfig.json`) peuvent nécessiter des ajustements manuels
3. **Tailwind CSS** : Si vous utilisez Tailwind, n'oubliez pas de l'initialiser séparément avec `npx tailwindcss init`
4. **Tests** : Les fichiers `.spec.ts` sont générés automatiquement avec les composants/services

---

*Guide généré automatiquement à partir de l'analyse du projet TKD*
