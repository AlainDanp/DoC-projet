# Guide — Implémentation & Test de l'Authentification sur le Frontend Angular

## Prérequis

- Backend démarré sur `http://localhost:8081`
- Angular CLI installé (`npm install -g @angular/cli`)
- Projet Angular sur `http://localhost:4200`

---

## Étape 1 — Créer le service d'authentification

```bash
ng generate service services/auth
```

**`src/app/services/auth.service.ts`**

```typescript
import { Injectable } from '@angular/core';
import { HttpClient } from '@angular/common/http';
import { Router } from '@angular/router';
import { Observable, tap } from 'rxjs';

export interface LoginRequest {
  email: string;
  password: string;
}

export interface RegisterRequest {
  firstName: string;
  lastName: string;
  email: string;
  password: string;
  phone?: string;
  nationality?: string;
  address?: string;
}

export interface AuthResponse {
  token: string;
  type: string;
  email: string;
  firstName: string;
  lastName: string;
}

@Injectable({ providedIn: 'root' })
export class AuthService {

  private readonly API = 'http://localhost:8081/api/auth';

  constructor(private http: HttpClient, private router: Router) {}

  register(data: RegisterRequest): Observable<AuthResponse> {
    return this.http.post<AuthResponse>(`${this.API}/register`, data).pipe(
      tap(res => this.saveToken(res))
    );
  }

  login(data: LoginRequest): Observable<AuthResponse> {
    return this.http.post<AuthResponse>(`${this.API}/login`, data).pipe(
      tap(res => this.saveToken(res))
    );
  }

  logout(): void {
    localStorage.removeItem('token');
    localStorage.removeItem('user');
    this.router.navigate(['/login']);
  }

  isLoggedIn(): boolean {
    return !!localStorage.getItem('token');
  }

  getToken(): string | null {
    return localStorage.getItem('token');
  }

  getUser(): AuthResponse | null {
    const u = localStorage.getItem('user');
    return u ? JSON.parse(u) : null;
  }

  private saveToken(res: AuthResponse): void {
    localStorage.setItem('token', res.token);
    localStorage.setItem('user', JSON.stringify(res));
  }
}
```

---

## Étape 2 — Créer l'intercepteur JWT

Toutes les requêtes vers le backend auront automatiquement le token.

```bash
ng generate interceptor interceptors/jwt
```

**`src/app/interceptors/jwt.interceptor.ts`**

```typescript
import { HttpInterceptorFn } from '@angular/common/http';
import { inject } from '@angular/core';
import { AuthService } from '../services/auth.service';

export const jwtInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).getToken();

  if (token) {
    req = req.clone({
      setHeaders: { Authorization: `Bearer ${token}` }
    });
  }

  return next(req);
};
```

Enregistrer dans **`app.config.ts`** :

```typescript
import { provideHttpClient, withInterceptors } from '@angular/common/http';
import { jwtInterceptor } from './interceptors/jwt.interceptor';

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideHttpClient(withInterceptors([jwtInterceptor]))
  ]
};
```

---

## Étape 3 — Créer l'Auth Guard

```bash
ng generate guard guards/auth
```

**`src/app/guards/auth.guard.ts`**

```typescript
import { inject } from '@angular/core';
import { CanActivateFn, Router } from '@angular/router';
import { AuthService } from '../services/auth.service';

export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);

  if (auth.isLoggedIn()) return true;

  router.navigate(['/login']);
  return false;
};
```

---

## Étape 4 — Composant Login

```bash
ng generate component pages/login
```

**`src/app/pages/login/login.component.ts`**

```typescript
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { Router, RouterLink } from '@angular/router';
import { CommonModule } from '@angular/common';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-login',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, RouterLink],
  templateUrl: './login.component.html'
})
export class LoginComponent {

  form: FormGroup;
  errorMessage = '';
  loading = false;

  constructor(private fb: FormBuilder, private auth: AuthService, private router: Router) {
    this.form = this.fb.group({
      email:    ['', [Validators.required, Validators.email]],
      password: ['', Validators.required]
    });
  }

  submit(): void {
    if (this.form.invalid) return;
    this.loading = true;
    this.errorMessage = '';

    this.auth.login(this.form.value).subscribe({
      next: () => this.router.navigate(['/dashboard']),
      error: err => {
        this.errorMessage = err.status === 401
          ? 'Email ou mot de passe incorrect'
          : 'Erreur serveur, réessayez';
        this.loading = false;
      }
    });
  }
}
```

**`src/app/pages/login/login.component.html`**

```html
<div class="login-container">
  <h2>Connexion</h2>

  <form [formGroup]="form" (ngSubmit)="submit()">

    <div>
      <label>Email</label>
      <input formControlName="email" type="email" placeholder="votre@email.com" />
      <span *ngIf="form.get('email')?.invalid && form.get('email')?.touched">
        Email invalide
      </span>
    </div>

    <div>
      <label>Mot de passe</label>
      <input formControlName="password" type="password" placeholder="••••••••" />
    </div>

    <p *ngIf="errorMessage" class="error">{{ errorMessage }}</p>

    <button type="submit" [disabled]="loading">
      {{ loading ? 'Connexion...' : 'Se connecter' }}
    </button>

  </form>

  <p>Pas de compte ? <a routerLink="/register">S'inscrire</a></p>
</div>
```

---

## Étape 5 — Composant Register

```bash
ng generate component pages/register
```

**`src/app/pages/register/register.component.ts`**

```typescript
import { Component } from '@angular/core';
import { FormBuilder, FormGroup, Validators, ReactiveFormsModule } from '@angular/forms';
import { Router, RouterLink } from '@angular/router';
import { CommonModule } from '@angular/common';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-register',
  standalone: true,
  imports: [CommonModule, ReactiveFormsModule, RouterLink],
  templateUrl: './register.component.html'
})
export class RegisterComponent {

  form: FormGroup;
  errorMessage = '';
  loading = false;

  constructor(private fb: FormBuilder, private auth: AuthService, private router: Router) {
    this.form = this.fb.group({
      firstName: ['', Validators.required],
      lastName:  ['', Validators.required],
      email:     ['', [Validators.required, Validators.email]],
      password:  ['', [Validators.required, Validators.minLength(6)]],
      phone:     ['']
    });
  }

  submit(): void {
    if (this.form.invalid) return;
    this.loading = true;
    this.errorMessage = '';

    this.auth.register(this.form.value).subscribe({
      next: () => this.router.navigate(['/dashboard']),
      error: err => {
        this.errorMessage = err.status === 400
          ? 'Email déjà utilisé'
          : 'Erreur serveur, réessayez';
        this.loading = false;
      }
    });
  }
}
```

**`src/app/pages/register/register.component.html`**

```html
<div class="register-container">
  <h2>Créer un compte</h2>

  <form [formGroup]="form" (ngSubmit)="submit()">

    <input formControlName="firstName" placeholder="Prénom" />
    <input formControlName="lastName"  placeholder="Nom" />
    <input formControlName="email"     type="email"    placeholder="Email" />
    <input formControlName="password"  type="password" placeholder="Mot de passe (6+ caractères)" />
    <input formControlName="phone"     placeholder="Téléphone (optionnel)" />

    <p *ngIf="errorMessage" class="error">{{ errorMessage }}</p>

    <button type="submit" [disabled]="loading">
      {{ loading ? 'Création...' : "S'inscrire" }}
    </button>

  </form>

  <p>Déjà un compte ? <a routerLink="/login">Se connecter</a></p>
</div>
```

---

## Étape 6 — Configurer les routes

**`src/app/app.routes.ts`**

```typescript
import { Routes } from '@angular/router';
import { authGuard } from './guards/auth.guard';

export const routes: Routes = [
  { path: '',          redirectTo: 'login', pathMatch: 'full' },
  { path: 'login',     loadComponent: () => import('./pages/login/login.component').then(m => m.LoginComponent) },
  { path: 'register',  loadComponent: () => import('./pages/register/register.component').then(m => m.RegisterComponent) },
  { path: 'dashboard', canActivate: [authGuard],
    loadComponent: () => import('./pages/dashboard/dashboard.component').then(m => m.DashboardComponent) },
];
```

---

## Étape 7 — Tester

### Test manuel avec le navigateur

1. Démarrer le backend : `./mvnw spring-boot:run`
2. Démarrer le frontend : `ng serve`
3. Aller sur `http://localhost:4200/register`

### Test avec curl

**Register**

```bash
curl -X POST http://localhost:8081/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "firstName": "Jean",
    "lastName": "Dupont",
    "email": "jean@hotel.com",
    "password": "secret123"
  }'
```

**Login**

```bash
curl -X POST http://localhost:8081/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "jean@hotel.com",
    "password": "secret123"
  }'
```

**Réponse attendue**

```json
{
  "token": "eyJhbGciOiJIUzI1NiJ9...",
  "type": "Bearer",
  "email": "jean@hotel.com",
  "firstName": "Jean",
  "lastName": "Dupont"
}
```

**Tester un endpoint protégé**

```bash
curl -X GET http://localhost:8081/api/rooms \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiJ9..."
```

---

## Résumé du flux

```
[Register/Login Form]
        ↓  POST /api/auth/register  ou  /login
[Backend Spring Boot :8081]
        ↓  retourne { token, email, firstName, lastName }
[AuthService Angular]
        ↓  stocke token dans localStorage
[JwtInterceptor]
        ↓  ajoute "Authorization: Bearer <token>" à chaque requête
[Endpoints protégés → /api/rooms, /api/reservations, ...]
```

---

## Erreurs fréquentes

| Erreur | Cause | Solution |
|--------|-------|----------|
| `CORS error` | Backend pas démarré ou mauvais port | Vérifier que le backend tourne sur `:8081` |
| `401 Unauthorized` | Token absent ou expiré | Se reconnecter pour obtenir un nouveau token |
| `400 Bad Request` au register | Email déjà utilisé | Utiliser un autre email |
| `Cannot POST /api/auth/login` | Backend non démarré | Lancer `./mvnw spring-boot:run` |
| Token non envoyé | Intercepteur non enregistré | Vérifier `app.config.ts` |
