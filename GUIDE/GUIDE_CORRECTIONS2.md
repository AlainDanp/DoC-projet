# Guide des corrections Frontend — DANP Hotel

> Ce guide liste toutes les corrections à effectuer avant de démarrer le backend,
> dans l'ordre de priorité recommandé.

---

## Sommaire

1. [Corriger le HTML du carousel](#1-corriger-le-html-du-carousel)
2. [AuthService + AuthGuard](#2-authservice--authguard)
3. [Structure des Services Angular](#3-structure-des-services-angular)
4. [environment.ts — Config API](#4-environmentts--config-api)
5. [Intercepteur HTTP JWT](#5-intercepteur-http-jwt)
6. [routerLinkActive dans le header](#6-routerlinkactive-dans-le-header)
7. [Scroll to top au changement de route](#7-scroll-to-top-au-changement-de-route)
8. [Corriger les textes placeholder](#8-corriger-les-textes-placeholder)
9. [Responsive des pages restantes](#9-responsive-des-pages-restantes)
10. [Titres de pages dynamiques](#10-titres-de-pages-dynamiques)
11. [Page de détail chambre](#11-page-de-détail-chambre)

---

## 1. Corriger le HTML du carousel

**Fichier :** `src/app/component/carousel/carousel.component.html`

**Problème :** Les balises `<html>` et `<body>` sont dans le template Angular, ce qui est invalide. Un composant Angular ne doit contenir que du contenu HTML, jamais les balises racines.

**Correction :**

Supprimer les balises `<app-header>`, `<html>` et `<body>` du fichier. Le template doit commencer directement par `<div class="carousel">`.

```html
<!-- AVANT (incorrect) -->
<app-header></app-header>
<html>
<body>
  <div class="carousel" #carousel>
    ...
  </div>
</body>
</html>

<!-- APRÈS (correct) -->
<div class="carousel" #carousel>
  ...
</div>
```

---

## 2. AuthService + AuthGuard

### 2a. Créer le service d'authentification

**Créer le fichier :** `src/app/services/auth.service.ts`

```typescript
import { Injectable } from '@angular/core';
import { Router } from '@angular/router';

@Injectable({ providedIn: 'root' })
export class AuthService {

  private readonly TOKEN_KEY = 'danp_token';
  private readonly USER_KEY  = 'danp_user';

  constructor(private router: Router) {}

  // Simule une connexion (à remplacer par un appel HTTP)
  login(email: string, password: string): boolean {
    // TODO: remplacer par this.http.post('/api/auth/login', { email, password })
    if (email && password) {
      localStorage.setItem(this.TOKEN_KEY, 'mock-jwt-token');
      localStorage.setItem(this.USER_KEY, JSON.stringify({ email, firstName: 'Jean', lastName: 'Dupont' }));
      return true;
    }
    return false;
  }

  logout(): void {
    localStorage.removeItem(this.TOKEN_KEY);
    localStorage.removeItem(this.USER_KEY);
    this.router.navigate(['/login']);
  }

  isLoggedIn(): boolean {
    return !!localStorage.getItem(this.TOKEN_KEY);
  }

  getToken(): string | null {
    return localStorage.getItem(this.TOKEN_KEY);
  }

  getUser(): any {
    const user = localStorage.getItem(this.USER_KEY);
    return user ? JSON.parse(user) : null;
  }
}
```

### 2b. Créer le garde de route

**Créer le fichier :** `src/app/guards/auth.guard.ts`

```typescript
import { Injectable } from '@angular/core';
import { CanActivate, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

@Injectable({ providedIn: 'root' })
export class AuthGuard implements CanActivate {

  constructor(private authService: AuthService, private router: Router) {}

  canActivate(): boolean {
    if (this.authService.isLoggedIn()) {
      return true;
    }
    this.router.navigate(['/login']);
    return false;
  }
}
```

### 2c. Appliquer le guard sur les routes protégées

**Fichier :** `src/app/app-routing.module.ts`

```typescript
import { AuthGuard } from './guards/auth.guard';

const routes: Routes = [
  { path: '', redirectTo: 'login', pathMatch: 'full' },
  { path: 'login', component: LoginComponent },
  {
    path: '',
    component: LayoutComponent,
    canActivate: [AuthGuard],   // ← Toutes les pages enfants sont protégées
    children: [
      { path: 'Homepage',    component: HomeComponent },
      { path: 'Chamber',     component: ChamberComponent },
      { path: 'Reservation', component: ReservationComponent },
      { path: 'Contact',     component: ContactComponent },
      { path: 'About',       component: AboutComponent },
      { path: 'Services',    component: ServicePageComponent },
      { path: 'Profile',     component: ProfileComponent },
    ]
  },
  { path: '**', component: NotFoundComponent }
];
```

### 2d. Connecter le formulaire de login au AuthService

**Fichier :** `src/app/component/login/login.component.ts`

```typescript
import { AuthService } from '../../services/auth.service';

// Dans la méthode onLogin() :
onLogin(): void {
  const email    = /* valeur du champ email */;
  const password = /* valeur du champ password */;
  if (this.authService.login(email, password)) {
    this.router.navigate(['/Homepage']);
  }
}
```

---

## 3. Structure des Services Angular

Créer un fichier par service avec des données mock. Quand le backend sera prêt, seul le corps des méthodes changera (ajout de `HttpClient`).

### 3a. RoomService

**Créer :** `src/app/services/room.service.ts`

```typescript
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';

export interface Room {
  id: string;
  name: string;
  type: 'standard' | 'deluxe' | 'suite' | 'presidential';
  pricePerNight: number;
  capacity: number;
  description: string;
  image: string;
  amenities: string[];
  available: boolean;
}

@Injectable({ providedIn: 'root' })
export class RoomService {

  private rooms: Room[] = [
    {
      id: 'R001',
      name: 'Chambre Standard',
      type: 'standard',
      pricePerNight: 80,
      capacity: 2,
      description: 'Chambre confortable avec vue sur le jardin.',
      image: '/image/imgchambre1.jpg',
      amenities: ['Wi-Fi', 'Climatisation', 'TV', 'Minibar'],
      available: true
    },
    {
      id: 'R002',
      name: 'Chambre Deluxe',
      type: 'deluxe',
      pricePerNight: 150,
      capacity: 2,
      description: 'Chambre spacieuse avec vue panoramique.',
      image: '/image/imgchambre3.jpg',
      amenities: ['Wi-Fi', 'Climatisation', 'TV', 'Minibar', 'Jacuzzi'],
      available: true
    },
    {
      id: 'R003',
      name: 'Suite Junior',
      type: 'suite',
      pricePerNight: 250,
      capacity: 3,
      description: 'Suite élégante avec salon séparé.',
      image: '/image/imgchambre5.jpg',
      amenities: ['Wi-Fi', 'Climatisation', 'TV 4K', 'Minibar', 'Jacuzzi', 'Room Service'],
      available: true
    },
    {
      id: 'R004',
      name: 'Suite Présidentielle',
      type: 'presidential',
      pricePerNight: 500,
      capacity: 4,
      description: 'Le summum du luxe avec terrasse privée.',
      image: '/image/imgchambre7.jpg',
      amenities: ['Wi-Fi', 'Climatisation', 'TV 4K', 'Minibar', 'Jacuzzi', 'Room Service 24h', 'Butler', 'Terrasse'],
      available: true
    }
  ];

  // TODO: remplacer par this.http.get<Room[]>(`${environment.apiUrl}/rooms`)
  getRooms(): Observable<Room[]> {
    return of(this.rooms);
  }

  // TODO: remplacer par this.http.get<Room>(`${environment.apiUrl}/rooms/${id}`)
  getRoomById(id: string): Observable<Room | undefined> {
    return of(this.rooms.find(r => r.id === id));
  }
}
```

### 3b. ReservationService

**Créer :** `src/app/services/reservation.service.ts`

```typescript
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';

export interface ReservationPayload {
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  checkIn: string;
  checkOut: string;
  roomType: string;
  guests: number;
  services: string[];
  specialRequests?: string;
}

@Injectable({ providedIn: 'root' })
export class ReservationService {

  // TODO: remplacer par this.http.post(`${environment.apiUrl}/reservations`, payload)
  createReservation(payload: ReservationPayload): Observable<{ id: string; success: boolean }> {
    const mockId = 'RES-' + Date.now();
    return of({ id: mockId, success: true });
  }

  // TODO: remplacer par this.http.get(`${environment.apiUrl}/reservations/user/${userId}`)
  getUserReservations(userId: string): Observable<any[]> {
    return of([]);
  }
}
```

### 3c. UserService

**Créer :** `src/app/services/user.service.ts`

```typescript
import { Injectable } from '@angular/core';
import { Observable, of } from 'rxjs';

export interface UserProfile {
  id: string;
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  nationality: string;
  address: string;
  memberSince: string;
  loyaltyPoints: number;
  tier: 'Standard' | 'Gold' | 'Platinum';
}

@Injectable({ providedIn: 'root' })
export class UserService {

  // TODO: remplacer par this.http.get(`${environment.apiUrl}/users/me`)
  getProfile(): Observable<UserProfile> {
    return of({
      id: 'U001',
      firstName: 'Jean',
      lastName: 'Dupont',
      email: 'jean.dupont@email.com',
      phone: '+237 6 99 00 11 22',
      nationality: 'Camerounaise',
      address: 'Yaoundé, Cameroun',
      memberSince: '2022',
      loyaltyPoints: 4750,
      tier: 'Gold'
    });
  }

  // TODO: remplacer par this.http.put(`${environment.apiUrl}/users/me`, profile)
  updateProfile(profile: Partial<UserProfile>): Observable<UserProfile> {
    return of({ ...profile } as UserProfile);
  }
}
```

---

## 4. environment.ts — Config API

**Fichier :** `src/environments/environment.ts`

```typescript
export const environment = {
  production: false,
  apiUrl: 'http://localhost:8080/api'
};
```

**Fichier :** `src/environments/environment.prod.ts`

```typescript
export const environment = {
  production: true,
  apiUrl: 'https://ton-domaine.com/api'
};
```

> Si le dossier `environments` n'existe pas, le créer dans `src/`.
> Puis dans `angular.json`, dans la section `fileReplacements` de la config production, vérifier que le remplacement est bien configuré.

---

## 5. Intercepteur HTTP JWT

Quand le backend sera actif, cet intercepteur ajoutera automatiquement le token dans chaque requête.

**Créer :** `src/app/interceptors/auth.interceptor.ts`

```typescript
import { Injectable } from '@angular/core';
import { HttpInterceptor, HttpRequest, HttpHandler } from '@angular/common/http';
import { AuthService } from '../services/auth.service';

@Injectable()
export class AuthInterceptor implements HttpInterceptor {

  constructor(private authService: AuthService) {}

  intercept(req: HttpRequest<any>, next: HttpHandler) {
    const token = this.authService.getToken();
    if (token) {
      const cloned = req.clone({
        setHeaders: { Authorization: `Bearer ${token}` }
      });
      return next.handle(cloned);
    }
    return next.handle(req);
  }
}
```

**Enregistrer dans `app.module.ts` :**

```typescript
import { HTTP_INTERCEPTORS, HttpClientModule } from '@angular/common/http';
import { AuthInterceptor } from './interceptors/auth.interceptor';

// Dans imports[] :
HttpClientModule,

// Dans providers[] :
{ provide: HTTP_INTERCEPTORS, useClass: AuthInterceptor, multi: true }
```

---

## 6. routerLinkActive dans le header

**Fichier :** `src/app/component/header/header.component.html`

Ajouter `routerLinkActive="active-link"` sur chaque `<a>` du menu pour mettre le lien en surbrillance selon la route actuelle.

```html
<li>
  <a routerLink="/Homepage" routerLinkActive="active-link">
    <span class="material-icons">home</span>Home
  </a>
</li>
<li>
  <a routerLink="/Chamber" routerLinkActive="active-link">
    <span class="material-icons">hotel</span>Chamber
  </a>
</li>
<!-- Répéter pour tous les liens -->
```

**Dans `header.component.css`, ajouter :**

```css
.sidebar ul li a.active-link {
  background: #444;
  padding-left: 40px;
  border-left: 3px solid #efc708;
  color: #efc708;
}
```

---

## 7. Scroll to top au changement de route

Sans cette correction, Angular conserve la position de scroll quand on navigue entre les pages.

**Fichier :** `src/app/app.module.ts`

```typescript
import { RouterModule } from '@angular/router';

// Dans imports[] :
RouterModule.forRoot(routes, { scrollPositionRestoration: 'top' })
```

> **Note :** Cette option est déjà dans `AppRoutingModule`. Il faut l'ajouter là :

**Fichier :** `src/app/app-routing.module.ts`

```typescript
@NgModule({
  imports: [RouterModule.forRoot(routes, {
    scrollPositionRestoration: 'top'   // ← ajouter cette option
  })],
  exports: [RouterModule]
})
export class AppRoutingModule { }
```

---

## 8. Corriger les textes placeholder

Les textes suivants sont encore des placeholders à remplacer par le vrai contenu DANP Hotel.

### Dans `src/app/services/services.component.html`

| Texte actuel | Texte à mettre |
|---|---|
| "NOS DIFFERNET SERVICE" | "NOS SERVICES" |
| "Discover the most loved Destionations..." | "Découvrez ce que notre hôtel vous offre" |
| "Tradition and Futurism" (×3 cartes) | Noms réels des services |
| "New York City, USA" (×3) | Descriptions des services |
| "Loved By Over Thousand Hotels home" | Supprimer ou remplacer |
| "RAISING COMFOMRT TO THE HIGHEST LEVEL" | "L'EXCELLENCE À VOTRE SERVICE" |
| "Welcome to Luviana Hotel Resort" | "Bienvenue au DANP Hotel" |
| Lorem ipsum (×2 paragraphes) | Texte réel de l'hôtel |
| "Journey to the Sky Made Simple!" | Titre adapté à l'hôtel |
| "UNLEASH WANDERLUST WITH SKYWINGS" | Slogan DANP Hotel |
| "Discover THe workd From Above" | Section à renommer/supprimer |
| "Book A Hotel Now" (bouton) | Ajouter `routerLink="/Reservation"` |
| "NOS RETOURS" | "AVIS DE NOS CLIENTS" |

### Dans `src/app/component/footer/footer.component.html`

- Vérifier que l'email est correct : `alain.datou@gmail.com`

---

## 9. Responsive des pages restantes

Pour chaque page listée ci-dessous, ouvrir les DevTools (F12), passer en mode mobile (375px, 768px) et corriger les débordements.

### Pages à vérifier et corriger

#### `reservation.component.css`
Points critiques :
- Le formulaire en 2 colonnes doit passer en 1 colonne sous 640px
- Le résumé de réservation doit s'empiler sous le formulaire
- Les checkboxes services doivent rester lisibles

```css
@media (max-width: 640px) {
  /* Passer le form-grid en 1 colonne */
  .form-grid {
    grid-template-columns: 1fr;
  }
  /* Empiler résumé sous le formulaire */
  .reservation-layout {
    grid-template-columns: 1fr;
  }
}
```

#### `contact.component.css`
Points critiques :
- La section hero doit avoir un texte lisible sur mobile
- Les 4 cartes d'info doivent passer de 2×2 à 1 colonne sous 480px
- Le formulaire + carte Google Maps doit s'empiler

```css
@media (max-width: 640px) {
  .info-cards {
    grid-template-columns: 1fr;
  }
  .contact-layout {
    grid-template-columns: 1fr;
  }
}
```

#### `about.component.css`
Points critiques :
- La section story (images + texte côte à côte) doit s'empiler
- Les 4 cards team doivent passer en 2 colonnes puis 1
- Les awards en 2×2 puis 1 colonne

```css
@media (max-width: 768px) {
  .story-grid     { grid-template-columns: 1fr; }
  .team-grid      { grid-template-columns: 1fr 1fr; }
  .awards-grid    { grid-template-columns: 1fr 1fr; }
}
@media (max-width: 480px) {
  .team-grid      { grid-template-columns: 1fr; }
  .awards-grid    { grid-template-columns: 1fr; }
}
```

#### `chamber-galery.component.css`
Points critiques :
- La grille de chambres doit passer en 1 colonne sous 640px
- La chambre "wide" (Présidentielle) doit perdre son `grid-column: span 2`

```css
@media (max-width: 640px) {
  .rooms-grid {
    grid-template-columns: 1fr;
  }
  .room-card-wide {
    grid-column: unset;
  }
}
```

#### `service-page.component.css`
Points critiques :
- La grille masonry doit passer en 2 colonnes puis 1 sur mobile
- Les 8 cards de services en 2 colonnes puis 1

---

## 10. Titres de pages dynamiques

Chaque page devrait avoir son propre `<title>` dans l'onglet du navigateur.

**Méthode simple — Dans chaque composant `.ts` :**

```typescript
import { Title } from '@angular/platform-browser';

constructor(private title: Title) {
  this.title.setTitle('Chambres — DANP Hotel');
}
```

**Titres suggérés :**

| Page | Titre |
|---|---|
| Login | `Connexion — DANP Hotel` |
| Home | `Accueil — DANP Hotel` |
| Chamber | `Nos Chambres — DANP Hotel` |
| Reservation | `Réservation — DANP Hotel` |
| Services | `Nos Services — DANP Hotel` |
| About | `À propos — DANP Hotel` |
| Contact | `Contact — DANP Hotel` |
| Profile | `Mon Profil — DANP Hotel` |
| 404 | `Page introuvable — DANP Hotel` |

---

## 11. Page de détail chambre

Actuellement, cliquer sur "Réserver" dans le carousel envoie directement vers `/Reservation` sans passer les infos de la chambre sélectionnée. Il manque une page de détail.

### Étapes

**1. Créer le composant :**
```bash
ng generate component component/chamber-detail
```

**2. Ajouter la route dans `app-routing.module.ts` :**
```typescript
{ path: 'Chamber/:id', component: ChamberDetailComponent }
```

**3. Template minimal `chamber-detail.component.html` :**
```html
<div class="detail-page">
  <div *ngIf="room">
    <div class="detail-hero" [style.background-image]="'url(' + room.image + ')'">
      <div class="detail-hero-overlay">
        <h1>{{ room.name }}</h1>
        <p>{{ room.pricePerNight }} € / nuit</p>
      </div>
    </div>
    <div class="detail-body">
      <p>{{ room.description }}</p>
      <ul>
        <li *ngFor="let a of room.amenities">{{ a }}</li>
      </ul>
      <a [routerLink]="['/Reservation']" [queryParams]="{ room: room.type }" class="btn-reserve">
        Réserver cette chambre
      </a>
    </div>
  </div>
  <app-footer></app-footer>
</div>
```

**4. Composant `chamber-detail.component.ts` :**
```typescript
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute } from '@angular/router';
import { RoomService, Room } from '../../services/room.service';

@Component({
  selector: 'app-chamber-detail',
  templateUrl: './chamber-detail.component.html',
  styleUrls: ['./chamber-detail.component.css']
})
export class ChamberDetailComponent implements OnInit {
  room: Room | undefined;

  constructor(private route: ActivatedRoute, private roomService: RoomService) {}

  ngOnInit(): void {
    const id = this.route.snapshot.paramMap.get('id');
    if (id) {
      this.roomService.getRoomById(id).subscribe(r => this.room = r);
    }
  }
}
```

**5. Mettre à jour les boutons "Réserver" du carousel-chamber :**
```html
<!-- Remplacer le href par un routerLink avec l'id de la chambre -->
<a [routerLink]="['/Chamber', room.id]" class="btn-reserve">
  Voir les détails
</a>
```

---

## Récapitulatif des fichiers à créer/modifier

| Action | Fichier |
|---|---|
| ✏️ Modifier | `src/app/component/carousel/carousel.component.html` |
| ✏️ Modifier | `src/app/app-routing.module.ts` |
| ✏️ Modifier | `src/app/app.module.ts` |
| ✏️ Modifier | `src/app/component/header/header.component.html` |
| ✏️ Modifier | `src/app/component/header/header.component.css` |
| ✏️ Modifier | `src/app/services/services.component.html` |
| ✏️ Modifier | `src/app/component/reservation/reservation.component.css` |
| ✏️ Modifier | `src/app/component/contact/contact.component.css` |
| ✏️ Modifier | `src/app/component/about/about.component.css` |
| ✏️ Modifier | `src/app/component/chamber-galery/chamber-galery.component.css` |
| ➕ Créer | `src/app/services/auth.service.ts` |
| ➕ Créer | `src/app/services/room.service.ts` |
| ➕ Créer | `src/app/services/reservation.service.ts` |
| ➕ Créer | `src/app/services/user.service.ts` |
| ➕ Créer | `src/app/guards/auth.guard.ts` |
| ➕ Créer | `src/app/interceptors/auth.interceptor.ts` |
| ➕ Créer | `src/environments/environment.ts` |
| ➕ Créer | `src/environments/environment.prod.ts` |
| ➕ Créer | `src/app/component/chamber-detail/` (composant complet) |

---

*DANP Hotel Frontend — Guide de corrections pré-backend*
