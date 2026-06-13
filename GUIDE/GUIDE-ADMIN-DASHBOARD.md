  # Guide : Dashboard Admin TKD

## Architecture globale

Le dashboard admin est un **module separe** accessible uniquement aux utilisateurs avec le role `ADMIN`.
Il utilise un layout distinct (sidebar + contenu) et ses propres composants, services et models.

```
src/app/
  admin/
    admin.module.ts              ← NgModule admin
    admin-routing.module.ts      ← Routes admin (lazy-loaded)
    admin-layout/                ← Layout avec sidebar
      admin-layout.ts
      admin-layout.html
      admin-layout.css
    dashboard/                   ← Page d'accueil admin
      dashboard.ts
      dashboard.html
      dashboard.css
    users/                       ← Gestion utilisateurs
      users.ts
      users.html
      users.css
    products/                    ← Gestion produits
      products.ts
      products.html
      products.css
    product-form/                ← Formulaire creation/edition produit
      product-form.ts
      product-form.html
      product-form.css
    orders/                      ← Gestion commandes (CRUD complet)
      orders.ts
      orders.html
      orders.css
    services/
      admin.service.ts           ← Service API admin
    models/
      admin.model.ts             ← Interfaces admin (DTOs)
```

---

## Etape 1 : Models admin

**Fichier : `src/app/admin/models/admin.model.ts`**

```typescript
// === Dashboard Stats ===
export interface DashboardStats {
  totalOrders: number;
  totalUsers: number;
  totalProducts: number;
  totalRevenue: number;

  ordersThisMonth: number;
  newUsersThisMonth: number;
  revenueThisMonth: number;

  ordersThisWeek: number;
  revenueThisWeek: number;

  ordersToday: number;
  revenueToday: number;

  pendingOrders: number;
  confirmedOrders: number;
  shippedOrders: number;
  deliveredOrders: number;
  cancelledOrders: number;

  activeProducts: number;
  outOfStockProducts: number;

  conversionRate: number;
}

// === Graphique des ventes ===
export interface SalesChart {
  labels: string[];       // ["Jan","Fev","Mar",...]
  revenues: number[];     // Revenus par mois
  orderCounts: number[];  // Nb commandes par mois
}

// === Produits les plus vendus ===
export interface TopProduct {
  productId: number;
  productName: string;
  productImage: string;
  totalSold: number;
  totalRevenue: number;
  averageRating: number;
}

// === Activites recentes ===
export interface RecentActivity {
  type: string;          // "ORDER" | "USER"
  description: string;
  timestamp: Date;
  icon: string;          // "shopping-cart" | "user-plus"
  color: string;         // "#3b82f6" | "#10b981"
}

// === Stats utilisateurs ===
export interface UserStats {
  totalUsers: number;
  activeUsers: number;
  verifiedUsers: number;
  newUsersThisWeek: number;
}

// === Role ===
export interface Role {
  id: number;
  name: string;
  description: string;
}

// === Requete de creation produit ===
export interface ProductCreateRequest {
  name: string;
  description?: string;
  price: number;
  discountPrice?: number;
  stockQuantity: number;
  categoryId: number;
  brand?: string;
  isActive: boolean;
}

// === Commandes admin ===
export interface OrderDTO {
  id: number;
  orderNumber: string;
  userId: number;
  userEmail: string;
  totalAmount: number;
  totalItems: number;
  status: OrderStatus;
  paymentMethod: string;
  paymentStatus: string;
  shippingFullName: string;
  shippingPhone: string;
  shippingAddress: string;
  notes: string;
  trackingNumber: string;
  createdAt: Date;
  updatedAt: Date;
  confirmedAt: Date;
  shippedAt: Date;
  deliveredAt: Date;
  items: OrderItemDTO[];
}

export interface OrderItemDTO {
  id: number;
  productId: number;
  productName: string;
  productImage: string;
  quantity: number;
  price: number;
  subtotal: number;
}

export type OrderStatus = 'PENDING' | 'CONFIRMED' | 'PROCESSING' | 'SHIPPED' | 'DELIVERED' | 'CANCELLED';

export interface OrderPage {
  content: OrderDTO[];
  totalElements: number;
  totalPages: number;
  number: number;
  size: number;
}
```

---

## Etape 2 : Service admin

**Fichier : `src/app/admin/services/admin.service.ts`**

```typescript
import { Injectable } from '@angular/core';
import { HttpClient, HttpParams } from '@angular/common/http';
import { Observable } from 'rxjs';
import { environment } from '../../../environments/environment';
import {
  DashboardStats, SalesChart, TopProduct,
  RecentActivity, UserStats, Role, ProductCreateRequest,
  OrderDTO, OrderPage
} from '../models/admin.model';
import { User } from '../../core/models/user.model';
import { Product, ProductPage } from '../../core/models/product.model';
import { ApiResponse } from '../../core/models/api-response.model';

// Interface pour les reponses paginées d'utilisateurs
export interface UserPage {
  content: User[];
  totalElements: number;
  totalPages: number;
  number: number;
  size: number;
}

@Injectable({
  providedIn: 'root'
})
export class AdminService {
  private apiUrl = environment.apiUrl;

  constructor(private http: HttpClient) {}

  // ==================== DASHBOARD ====================

  getDashboardStats(): Observable<DashboardStats> {
    return this.http.get<DashboardStats>(`${this.apiUrl}/admin/dashboard/stats`);
  }

  getSalesChart(): Observable<SalesChart> {
    return this.http.get<SalesChart>(`${this.apiUrl}/admin/dashboard/sales-chart`);
  }

  getTopProducts(limit: number = 10): Observable<TopProduct[]> {
    return this.http.get<TopProduct[]>(
      `${this.apiUrl}/admin/dashboard/top-products`,
      { params: { limit: limit.toString() } }
    );
  }

  getRecentActivities(limit: number = 20): Observable<RecentActivity[]> {
    return this.http.get<RecentActivity[]>(
      `${this.apiUrl}/admin/dashboard/recent-activities`,
      { params: { limit: limit.toString() } }
    );
  }

  getRevenue(period: 'week' | 'month' | 'year' = 'month'): Observable<Record<string, number>> {
    return this.http.get<Record<string, number>>(
      `${this.apiUrl}/admin/dashboard/revenue`,
      { params: { period } }
    );
  }

  // ==================== UTILISATEURS ====================

  getUsers(page = 0, size = 10, sortBy = 'createdAt', sortDir = 'desc'): Observable<UserPage> {
    const params = new HttpParams()
      .set('page', page.toString())
      .set('size', size.toString())
      .set('sortBy', sortBy)
      .set('sortDir', sortDir);
    return this.http.get<UserPage>(`${this.apiUrl}/admin/users`, { params });
  }

  searchUsers(query: string, page = 0, size = 10): Observable<UserPage> {
    const params = new HttpParams()
      .set('query', query)
      .set('page', page.toString())
      .set('size', size.toString());
    return this.http.get<UserPage>(`${this.apiUrl}/admin/users/search`, { params });
  }

  getUserById(id: number): Observable<User> {
    return this.http.get<User>(`${this.apiUrl}/admin/users/${id}`);
  }

  toggleUserStatus(id: number): Observable<ApiResponse> {
    return this.http.put<ApiResponse>(`${this.apiUrl}/admin/users/${id}/toggle-status`, {});
  }

  deleteUser(id: number): Observable<ApiResponse> {
    return this.http.delete<ApiResponse>(`${this.apiUrl}/admin/users/${id}`);
  }

  getUserStats(): Observable<UserStats> {
    return this.http.get<UserStats>(`${this.apiUrl}/admin/statistics/users`);
  }

  // ==================== PRODUITS ====================
  // Utilise les endpoints publics /api/products mais avec token ADMIN

  getProducts(page = 0, size = 20): Observable<ProductPage> {
    const params = new HttpParams()
      .set('page', page.toString())
      .set('size', size.toString());
    return this.http.get<ProductPage>(`${this.apiUrl}/products`, { params });
  }

  createProduct(product: ProductCreateRequest): Observable<Product> {
    return this.http.post<Product>(`${this.apiUrl}/products`, product);
  }

  updateProduct(id: number, product: ProductCreateRequest): Observable<Product> {
    return this.http.put<Product>(`${this.apiUrl}/products/${id}`, product);
  }

  deleteProduct(id: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/products/${id}`);
  }

  uploadProductImage(productId: number, file: File, isPrimary = false): Observable<{ imageUrl: string; message: string }> {
    const formData = new FormData();
    formData.append('file', file);
    return this.http.post<{ imageUrl: string; message: string }>(
      `${this.apiUrl}/products/${productId}/images`,
      formData,
      { params: { isPrimary: isPrimary.toString() } }
    );
  }

  deleteProductImage(imageId: number): Observable<void> {
    return this.http.delete<void>(`${this.apiUrl}/products/images/${imageId}`);
  }

  // ==================== COMMANDES ====================

  getOrders(page = 0, size = 10, status?: string, sortBy = 'createdAt', sortDir = 'desc'): Observable<OrderPage> {
    let params = new HttpParams()
      .set('page', page.toString())
      .set('size', size.toString())
      .set('sortBy', sortBy)
      .set('sortDir', sortDir);
    if (status) params = params.set('status', status);
    return this.http.get<OrderPage>(`${this.apiUrl}/admin/orders`, { params });
  }

  getOrderById(id: number): Observable<OrderDTO> {
    return this.http.get<OrderDTO>(`${this.apiUrl}/admin/orders/${id}`);
  }

  updateOrderStatus(id: number, status: string): Observable<OrderDTO> {
    return this.http.put<OrderDTO>(
      `${this.apiUrl}/admin/orders/${id}/status`,
      null,
      { params: { status } }
    );
  }

  setTrackingNumber(id: number, trackingNumber: string): Observable<OrderDTO> {
    return this.http.put<OrderDTO>(
      `${this.apiUrl}/admin/orders/${id}/tracking`,
      null,
      { params: { trackingNumber } }
    );
  }

  // ==================== ROLES ====================

  getRoles(): Observable<Role[]> {
    return this.http.get<Role[]>(`${this.apiUrl}/admin/roles`);
  }

  createRole(name: string, description?: string): Observable<Role> {
    let params = new HttpParams().set('name', name);
    if (description) params = params.set('description', description);
    return this.http.post<Role>(`${this.apiUrl}/admin/roles`, null, { params });
  }

  deleteRole(id: number): Observable<ApiResponse> {
    return this.http.delete<ApiResponse>(`${this.apiUrl}/admin/roles/${id}`);
  }
}
```

---

## Etape 3 : Module et routing admin

### 3.1 - Module admin

**Fichier : `src/app/admin/admin.module.ts`**

```typescript
import { NgModule } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { AdminRoutingModule } from './admin-routing.module';

import { AdminLayout } from './admin-layout/admin-layout';
import { Dashboard } from './dashboard/dashboard';
import { AdminUsers } from './users/users';
import { AdminProducts } from './products/products';
import { AdminProductForm } from './product-form/product-form';
import { AdminOrders } from './orders/orders';

@NgModule({
  declarations: [
    AdminLayout,
    Dashboard,
    AdminUsers,
    AdminProducts,
    AdminProductForm,
    AdminOrders
  ],
  imports: [
    CommonModule,
    FormsModule,
    AdminRoutingModule
  ]
})
export class AdminModule {}
```

### 3.2 - Routing admin

**Fichier : `src/app/admin/admin-routing.module.ts`**

```typescript
import { NgModule } from '@angular/core';
import { RouterModule, Routes } from '@angular/router';
import { adminGuard } from '../core/guards/auth-guard';

import { AdminLayout } from './admin-layout/admin-layout';
import { Dashboard } from './dashboard/dashboard';
import { AdminUsers } from './users/users';
import { AdminProducts } from './products/products';
import { AdminProductForm } from './product-form/product-form';
import { AdminOrders } from './orders/orders';

const routes: Routes = [
  {
    path: '',
    component: AdminLayout,
    canActivate: [adminGuard],
    children: [
      { path: '', redirectTo: 'dashboard', pathMatch: 'full' },
      { path: 'dashboard', component: Dashboard },
      { path: 'users', component: AdminUsers },
      { path: 'products', component: AdminProducts },
      { path: 'products/new', component: AdminProductForm },
      { path: 'products/edit/:id', component: AdminProductForm },
      { path: 'orders', component: AdminOrders }
    ]
  }
];

@NgModule({
  imports: [RouterModule.forChild(routes)],
  exports: [RouterModule]
})
export class AdminRoutingModule {}
```

### 3.3 - Ajouter la route admin dans app-routing.module.ts

```typescript
// Dans le tableau routes, AVANT le wildcard :
{
  path: 'admin',
  loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
},
```

> Le `adminGuard` existe deja dans `src/app/core/guards/auth-guard.ts` et verifie `isLoggedIn() && isAdmin()`.

---

## Etape 4 : Layout admin (sidebar)

### 4.1 - admin-layout.ts

**Fichier : `src/app/admin/admin-layout/admin-layout.ts`**

```typescript
import { Component } from '@angular/core';
import { Router } from '@angular/router';
import { AuthService } from '../../core/services/auth.service';

@Component({
  selector: 'app-admin-layout',
  standalone: false,
  templateUrl: './admin-layout.html',
  styleUrl: './admin-layout.css'
})
export class AdminLayout {
  sidebarOpen = true;

  menuItems = [
    { label: 'Dashboard',   icon: 'bx bxs-dashboard',    route: '/admin/dashboard' },
    { label: 'Utilisateurs', icon: 'bx bxs-user-detail',  route: '/admin/users' },
    { label: 'Produits',     icon: 'bx bxs-package',      route: '/admin/products' },
    { label: 'Commandes',    icon: 'bx bxs-cart',         route: '/admin/orders' },
  ];

  constructor(
    private authService: AuthService,
    private router: Router
  ) {}

  toggleSidebar(): void {
    this.sidebarOpen = !this.sidebarOpen;
  }

  logout(): void {
    this.authService.logout();
  }

  goToSite(): void {
    this.router.navigate(['/Home']);
  }
}
```

### 4.2 - admin-layout.html

```html
<div class="admin-wrapper" [class.sidebar-collapsed]="!sidebarOpen">

  <!-- Sidebar -->
  <aside class="sidebar">
    <div class="sidebar-header">
      <h2 class="sidebar-logo">TKD Admin</h2>
      <button class="sidebar-toggle" (click)="toggleSidebar()">
        <i class='bx bx-menu'></i>
      </button>
    </div>

    <nav class="sidebar-nav">
      <a *ngFor="let item of menuItems"
         [routerLink]="item.route"
         routerLinkActive="active"
         class="sidebar-link">
        <i [class]="item.icon"></i>
        <span class="link-text">{{ item.label }}</span>
      </a>
    </nav>

    <div class="sidebar-footer">
      <a class="sidebar-link" (click)="goToSite()">
        <i class='bx bx-store'></i>
        <span class="link-text">Voir le site</span>
      </a>
      <a class="sidebar-link logout" (click)="logout()">
        <i class='bx bx-log-out'></i>
        <span class="link-text">Deconnexion</span>
      </a>
    </div>
  </aside>

  <!-- Main Content -->
  <main class="admin-main">
    <header class="admin-header">
      <button class="header-toggle" (click)="toggleSidebar()">
        <i class='bx bx-menu'></i>
      </button>
      <h1 class="header-title">Administration</h1>
    </header>

    <div class="admin-content">
      <router-outlet></router-outlet>
    </div>
  </main>
</div>
```

### 4.3 - admin-layout.css

```css
.admin-wrapper {
  display: flex;
  min-height: 100vh;
  background: #f1f5f9;
}

/* Sidebar */
.sidebar {
  width: 260px;
  background: #1e293b;
  color: #e2e8f0;
  display: flex;
  flex-direction: column;
  transition: width 0.3s ease;
  position: fixed;
  top: 0;
  left: 0;
  bottom: 0;
  z-index: 100;
}

.sidebar-collapsed .sidebar {
  width: 70px;
}

.sidebar-collapsed .link-text,
.sidebar-collapsed .sidebar-logo {
  display: none;
}

.sidebar-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 20px 16px;
  border-bottom: 1px solid #334155;
}

.sidebar-logo {
  font-size: 1.3rem;
  font-weight: 700;
  color: #60a5fa;
}

.sidebar-toggle {
  background: none;
  border: none;
  color: #94a3b8;
  font-size: 1.4rem;
  cursor: pointer;
}

.sidebar-nav {
  flex: 1;
  padding: 16px 0;
  overflow-y: auto;
}

.sidebar-link {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 20px;
  color: #cbd5e1;
  text-decoration: none;
  cursor: pointer;
  transition: all 0.2s;
  border-left: 3px solid transparent;
}

.sidebar-link:hover {
  background: #334155;
  color: #f8fafc;
}

.sidebar-link.active {
  background: #334155;
  color: #60a5fa;
  border-left-color: #60a5fa;
}

.sidebar-link i {
  font-size: 1.3rem;
  min-width: 24px;
  text-align: center;
}

.sidebar-footer {
  border-top: 1px solid #334155;
  padding: 8px 0;
}

.sidebar-link.logout:hover {
  background: #7f1d1d;
  color: #fca5a5;
}

/* Main content */
.admin-main {
  flex: 1;
  margin-left: 260px;
  transition: margin-left 0.3s ease;
}

.sidebar-collapsed .admin-main {
  margin-left: 70px;
}

.admin-header {
  background: white;
  padding: 16px 24px;
  display: flex;
  align-items: center;
  gap: 16px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.1);
  position: sticky;
  top: 0;
  z-index: 50;
}

.header-toggle {
  background: none;
  border: none;
  font-size: 1.4rem;
  color: #475569;
  cursor: pointer;
}

.header-title {
  font-size: 1.2rem;
  font-weight: 600;
  color: #1e293b;
}

.admin-content {
  padding: 24px;
}

/* Responsive */
@media (max-width: 768px) {
  .sidebar {
    width: 70px;
  }
  .link-text, .sidebar-logo {
    display: none;
  }
  .admin-main {
    margin-left: 70px;
  }
}
```

---

## Etape 5 : Page Dashboard

### 5.1 - dashboard.ts

**Fichier : `src/app/admin/dashboard/dashboard.ts`**

```typescript
import { Component, OnInit } from '@angular/core';
import { AdminService } from '../services/admin.service';
import {
  DashboardStats, SalesChart,
  TopProduct, RecentActivity
} from '../models/admin.model';
import { environment } from '../../../environments/environment';

@Component({
  selector: 'app-admin-dashboard',
  standalone: false,
  templateUrl: './dashboard.html',
  styleUrl: './dashboard.css'
})
export class Dashboard implements OnInit {
  stats: DashboardStats | null = null;
  salesChart: SalesChart | null = null;
  topProducts: TopProduct[] = [];
  recentActivities: RecentActivity[] = [];
  isLoading = true;

  constructor(private adminService: AdminService) {}

  ngOnInit(): void {
    this.loadDashboard();
  }

  loadDashboard(): void {
    this.isLoading = true;

    // Charger les stats
    this.adminService.getDashboardStats().subscribe({
      next: (data) => {
        this.stats = data;
        this.isLoading = false;
      },
      error: () => this.isLoading = false
    });

    // Charger le graphique
    this.adminService.getSalesChart().subscribe({
      next: (data) => this.salesChart = data
    });

    // Top produits
    this.adminService.getTopProducts(5).subscribe({
      next: (data) => this.topProducts = data
    });

    // Activites recentes
    this.adminService.getRecentActivities(10).subscribe({
      next: (data) => this.recentActivities = data
    });
  }

  getProductImageUrl(image: string): string {
    if (!image) return 'assets/default-product.jpg';
    if (image.startsWith('http')) return image;
    return `${environment.apiUrl}/uploads/${image}`;
  }

  formatCurrency(amount: number): string {
    return new Intl.NumberFormat('fr-FR').format(amount) + ' FCFA';
  }
}
```

### 5.2 - dashboard.html

```html
<div class="dashboard">

  <!-- Spinner de chargement -->
  <div class="loading" *ngIf="isLoading">
    <i class='bx bx-loader-alt bx-spin'></i>
    <p>Chargement du dashboard...</p>
  </div>

  <!-- KPI Cards -->
  <div class="kpi-grid" *ngIf="stats">
    <div class="kpi-card blue">
      <div class="kpi-icon"><i class='bx bx-cart'></i></div>
      <div class="kpi-info">
        <span class="kpi-value">{{ stats.totalOrders }}</span>
        <span class="kpi-label">Commandes</span>
        <span class="kpi-sub">{{ stats.ordersToday }} aujourd'hui</span>
      </div>
    </div>

    <div class="kpi-card green">
      <div class="kpi-icon"><i class='bx bx-money'></i></div>
      <div class="kpi-info">
        <span class="kpi-value">{{ formatCurrency(stats.totalRevenue) }}</span>
        <span class="kpi-label">Revenus totaux</span>
        <span class="kpi-sub">{{ formatCurrency(stats.revenueThisMonth) }} ce mois</span>
      </div>
    </div>

    <div class="kpi-card purple">
      <div class="kpi-icon"><i class='bx bx-user'></i></div>
      <div class="kpi-info">
        <span class="kpi-value">{{ stats.totalUsers }}</span>
        <span class="kpi-label">Utilisateurs</span>
        <span class="kpi-sub">{{ stats.newUsersThisMonth }} ce mois</span>
      </div>
    </div>

    <div class="kpi-card orange">
      <div class="kpi-icon"><i class='bx bx-package'></i></div>
      <div class="kpi-info">
        <span class="kpi-value">{{ stats.totalProducts }}</span>
        <span class="kpi-label">Produits</span>
        <span class="kpi-sub">{{ stats.outOfStockProducts }} en rupture</span>
      </div>
    </div>
  </div>

  <!-- Order Status Bar -->
  <div class="status-bar" *ngIf="stats">
    <h3 class="section-title">Statut des commandes</h3>
    <div class="status-grid">
      <div class="status-item pending">
        <span class="status-count">{{ stats.pendingOrders }}</span>
        <span class="status-label">En attente</span>
      </div>
      <div class="status-item confirmed">
        <span class="status-count">{{ stats.confirmedOrders }}</span>
        <span class="status-label">Confirmees</span>
      </div>
      <div class="status-item shipped">
        <span class="status-count">{{ stats.shippedOrders }}</span>
        <span class="status-label">Expediees</span>
      </div>
      <div class="status-item delivered">
        <span class="status-count">{{ stats.deliveredOrders }}</span>
        <span class="status-label">Livrees</span>
      </div>
      <div class="status-item cancelled">
        <span class="status-count">{{ stats.cancelledOrders }}</span>
        <span class="status-label">Annulees</span>
      </div>
    </div>
  </div>

  <div class="dashboard-grid">

    <!-- Top Produits -->
    <div class="card">
      <h3 class="card-title"><i class='bx bx-trending-up'></i> Top produits</h3>
      <div class="top-product" *ngFor="let p of topProducts; let i = index">
        <span class="rank">#{{ i + 1 }}</span>
        <img [src]="getProductImageUrl(p.productImage)" [alt]="p.productName" class="product-thumb">
        <div class="product-info">
          <span class="product-name">{{ p.productName }}</span>
          <span class="product-sold">{{ p.totalSold }} vendus</span>
        </div>
        <span class="product-revenue">{{ formatCurrency(p.totalRevenue) }}</span>
      </div>
      <p *ngIf="topProducts.length === 0" class="empty">Aucune vente</p>
    </div>

    <!-- Activites recentes -->
    <div class="card">
      <h3 class="card-title"><i class='bx bx-time-five'></i> Activites recentes</h3>
      <div class="activity" *ngFor="let a of recentActivities">
        <div class="activity-icon" [style.background]="a.color + '20'" [style.color]="a.color">
          <i class='bx bx-{{ a.icon }}'></i>
        </div>
        <div class="activity-info">
          <span class="activity-desc">{{ a.description }}</span>
          <span class="activity-time">{{ a.timestamp | date:'dd/MM/yyyy HH:mm' }}</span>
        </div>
      </div>
      <p *ngIf="recentActivities.length === 0" class="empty">Aucune activite</p>
    </div>

  </div>
</div>
```

### 5.3 - dashboard.css (principaux styles)

```css
.dashboard { max-width: 1400px; }

.kpi-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: 20px;
  margin-bottom: 24px;
}

.kpi-card {
  background: white;
  border-radius: 12px;
  padding: 20px;
  display: flex;
  align-items: center;
  gap: 16px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.08);
}

.kpi-icon {
  width: 56px;
  height: 56px;
  border-radius: 12px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.6rem;
}

.kpi-card.blue .kpi-icon  { background: #dbeafe; color: #2563eb; }
.kpi-card.green .kpi-icon { background: #d1fae5; color: #059669; }
.kpi-card.purple .kpi-icon { background: #ede9fe; color: #7c3aed; }
.kpi-card.orange .kpi-icon { background: #ffedd5; color: #ea580c; }

.kpi-info { display: flex; flex-direction: column; }
.kpi-value { font-size: 1.5rem; font-weight: 700; color: #1e293b; }
.kpi-label { font-size: 0.85rem; color: #64748b; }
.kpi-sub { font-size: 0.75rem; color: #94a3b8; margin-top: 4px; }

/* Status bar */
.status-bar {
  background: white;
  border-radius: 12px;
  padding: 20px;
  margin-bottom: 24px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.08);
}

.status-grid {
  display: flex;
  gap: 16px;
  flex-wrap: wrap;
}

.status-item {
  flex: 1;
  min-width: 100px;
  text-align: center;
  padding: 12px;
  border-radius: 8px;
}

.status-item.pending   { background: #fef3c7; }
.status-item.confirmed { background: #dbeafe; }
.status-item.shipped   { background: #e0e7ff; }
.status-item.delivered  { background: #d1fae5; }
.status-item.cancelled  { background: #fee2e2; }

.status-count { display: block; font-size: 1.5rem; font-weight: 700; }
.status-label { font-size: 0.8rem; color: #475569; }

/* Grid 2 colonnes */
.dashboard-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 24px;
}

@media (max-width: 900px) {
  .dashboard-grid { grid-template-columns: 1fr; }
}

.card {
  background: white;
  border-radius: 12px;
  padding: 20px;
  box-shadow: 0 1px 3px rgba(0,0,0,0.08);
}

.card-title {
  font-size: 1rem;
  font-weight: 600;
  color: #1e293b;
  margin-bottom: 16px;
  display: flex;
  align-items: center;
  gap: 8px;
}

/* Top products */
.top-product {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 0;
  border-bottom: 1px solid #f1f5f9;
}

.rank { font-weight: 700; color: #64748b; width: 24px; }
.product-thumb { width: 40px; height: 40px; border-radius: 8px; object-fit: cover; }
.product-info { flex: 1; display: flex; flex-direction: column; }
.product-name { font-weight: 500; font-size: 0.9rem; }
.product-sold { font-size: 0.75rem; color: #94a3b8; }
.product-revenue { font-weight: 600; color: #059669; font-size: 0.85rem; }

/* Activities */
.activity {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 10px 0;
  border-bottom: 1px solid #f1f5f9;
}

.activity-icon {
  width: 36px;
  height: 36px;
  border-radius: 8px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 1.1rem;
}

.activity-info { flex: 1; display: flex; flex-direction: column; }
.activity-desc { font-size: 0.85rem; color: #334155; }
.activity-time { font-size: 0.75rem; color: #94a3b8; }

.empty { text-align: center; color: #94a3b8; padding: 20px; }
.loading { text-align: center; padding: 60px; color: #64748b; font-size: 1.5rem; }
.loading i { font-size: 2rem; }

.section-title {
  font-size: 1rem;
  font-weight: 600;
  margin-bottom: 12px;
  color: #1e293b;
}
```

---

## Etape 6 : Page Gestion Utilisateurs

### 6.1 - users.ts

```typescript
import { Component, OnInit } from '@angular/core';
import { ToastrService } from 'ngx-toastr';
import { AdminService, UserPage } from '../services/admin.service';
import { User } from '../../core/models/user.model';
import { environment } from '../../../environments/environment';

@Component({
  selector: 'app-admin-users',
  standalone: false,
  templateUrl: './users.html',
  styleUrl: './users.css'
})
export class AdminUsers implements OnInit {
  users: User[] = [];
  currentPage = 0;
  totalPages = 0;
  totalElements = 0;
  searchQuery = '';
  isLoading = false;

  constructor(
    private adminService: AdminService,
    private toastr: ToastrService
  ) {}

  ngOnInit(): void {
    this.loadUsers();
  }

  loadUsers(page = 0): void {
    this.isLoading = true;
    const obs = this.searchQuery
      ? this.adminService.searchUsers(this.searchQuery, page)
      : this.adminService.getUsers(page);

    obs.subscribe({
      next: (data: UserPage) => {
        this.users = data.content;
        this.currentPage = data.number;
        this.totalPages = data.totalPages;
        this.totalElements = data.totalElements;
        this.isLoading = false;
      },
      error: () => {
        this.toastr.error('Erreur chargement utilisateurs');
        this.isLoading = false;
      }
    });
  }

  search(): void {
    this.loadUsers(0);
  }

  toggleStatus(user: User): void {
    this.adminService.toggleUserStatus(user.id).subscribe({
      next: () => {
        user.isActive = !user.isActive;
        this.toastr.success(`Utilisateur ${user.isActive ? 'active' : 'desactive'}`);
      },
      error: () => this.toastr.error('Erreur lors de la mise a jour')
    });
  }

  deleteUser(user: User): void {
    if (!confirm(`Supprimer l'utilisateur ${user.username} ?`)) return;

    this.adminService.deleteUser(user.id).subscribe({
      next: () => {
        this.toastr.success('Utilisateur supprime');
        this.loadUsers(this.currentPage);
      },
      error: () => this.toastr.error('Erreur lors de la suppression')
    });
  }

  getAvatarUrl(avatar?: string): string {
    if (!avatar) return 'assets/default-avatar.png';
    if (avatar.startsWith('http')) return avatar;
    return `${environment.apiUrl}/uploads/${avatar}`;
  }

  prevPage(): void {
    if (this.currentPage > 0) this.loadUsers(this.currentPage - 1);
  }

  nextPage(): void {
    if (this.currentPage < this.totalPages - 1) this.loadUsers(this.currentPage + 1);
  }
}
```

### 6.2 - users.html

```html
<div class="users-page">
  <div class="page-header">
    <h2>Gestion des utilisateurs</h2>
    <span class="total-badge">{{ totalElements }} utilisateurs</span>
  </div>

  <!-- Barre de recherche -->
  <div class="search-bar">
    <input type="search" placeholder="Rechercher par nom, email..."
           [(ngModel)]="searchQuery" (input)="search()" class="search-input">
    <i class='bx bx-search'></i>
  </div>

  <!-- Table -->
  <div class="table-wrapper">
    <table class="data-table">
      <thead>
        <tr>
          <th>Utilisateur</th>
          <th>Email</th>
          <th>Role</th>
          <th>Statut</th>
          <th>Inscription</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let user of users">
          <td class="user-cell">
            <img [src]="getAvatarUrl(user.avatar)" class="avatar-sm">
            <span>{{ user.username }}</span>
          </td>
          <td>{{ user.email }}</td>
          <td>
            <span class="role-badge" [class.admin]="user.roles.includes('ADMIN')">
              {{ user.roles[0] || 'USER' }}
            </span>
          </td>
          <td>
            <span class="status-badge" [class.active]="user.isActive" [class.inactive]="!user.isActive">
              {{ user.isActive ? 'Actif' : 'Inactif' }}
            </span>
          </td>
          <td>{{ user.createdAt | date:'dd/MM/yyyy' }}</td>
          <td class="actions">
            <button class="btn-sm toggle" (click)="toggleStatus(user)"
                    [title]="user.isActive ? 'Desactiver' : 'Activer'">
              <i [class]="user.isActive ? 'bx bx-block' : 'bx bx-check'"></i>
            </button>
            <button class="btn-sm delete" (click)="deleteUser(user)" title="Supprimer">
              <i class='bx bx-trash'></i>
            </button>
          </td>
        </tr>
      </tbody>
    </table>

    <p *ngIf="users.length === 0 && !isLoading" class="empty">Aucun utilisateur trouve</p>
  </div>

  <!-- Pagination -->
  <div class="pagination" *ngIf="totalPages > 1">
    <button (click)="prevPage()" [disabled]="currentPage === 0">Precedent</button>
    <span>Page {{ currentPage + 1 }} / {{ totalPages }}</span>
    <button (click)="nextPage()" [disabled]="currentPage >= totalPages - 1">Suivant</button>
  </div>
</div>
```

---

## Etape 7 : Page Gestion Produits

### 7.1 - products.ts

```typescript
import { Component, OnInit } from '@angular/core';
import { Router } from '@angular/router';
import { ToastrService } from 'ngx-toastr';
import { AdminService } from '../services/admin.service';
import { Product, ProductPage } from '../../core/models/product.model';
import { environment } from '../../../environments/environment';

@Component({
  selector: 'app-admin-products',
  standalone: false,
  templateUrl: './products.html',
  styleUrl: './products.css'
})
export class AdminProducts implements OnInit {
  products: Product[] = [];
  currentPage = 0;
  totalPages = 0;
  isLoading = false;

  constructor(
    private adminService: AdminService,
    private router: Router,
    private toastr: ToastrService
  ) {}

  ngOnInit(): void {
    this.loadProducts();
  }

  loadProducts(page = 0): void {
    this.isLoading = true;
    this.adminService.getProducts(page, 20).subscribe({
      next: (data: ProductPage) => {
        this.products = data.content;
        this.currentPage = data.number;
        this.totalPages = data.totalPages;
        this.isLoading = false;
      },
      error: () => {
        this.toastr.error('Erreur chargement produits');
        this.isLoading = false;
      }
    });
  }

  addProduct(): void {
    this.router.navigate(['/admin/products/new']);
  }

  editProduct(id: number): void {
    this.router.navigate(['/admin/products/edit', id]);
  }

  deleteProduct(product: Product): void {
    if (!confirm(`Supprimer "${product.name}" ?`)) return;

    this.adminService.deleteProduct(product.id).subscribe({
      next: () => {
        this.toastr.success('Produit supprime');
        this.loadProducts(this.currentPage);
      },
      error: () => this.toastr.error('Erreur suppression')
    });
  }

  getImageUrl(product: Product): string {
    const path = product.primaryImage ?? product.imageUrls?.[0];
    if (!path) return 'assets/default-product.jpg';
    if (path.startsWith('http')) return path;
    return `${environment.apiUrl}/uploads/${path}`;
  }

  prevPage(): void {
    if (this.currentPage > 0) this.loadProducts(this.currentPage - 1);
  }

  nextPage(): void {
    if (this.currentPage < this.totalPages - 1) this.loadProducts(this.currentPage + 1);
  }
}
```

### 7.2 - products.html

```html
<div class="products-page">
  <div class="page-header">
    <h2>Gestion des produits</h2>
    <button class="btn-primary" (click)="addProduct()">
      <i class='bx bx-plus'></i> Nouveau produit
    </button>
  </div>

  <div class="table-wrapper">
    <table class="data-table">
      <thead>
        <tr>
          <th>Image</th>
          <th>Nom</th>
          <th>Prix</th>
          <th>Stock</th>
          <th>Categorie</th>
          <th>Statut</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let p of products">
          <td><img [src]="getImageUrl(p)" class="product-thumb"></td>
          <td>{{ p.name }}</td>
          <td>{{ p.price | number:'1.0-0' }} FCFA</td>
          <td>
            <span [class.stock-low]="p.stockQuantity <= 5"
                  [class.stock-out]="p.stockQuantity === 0">
              {{ p.stockQuantity }}
            </span>
          </td>
          <td>{{ p.categoryName || '-' }}</td>
          <td>
            <span class="status-badge" [class.active]="p.isActive">
              {{ p.isActive ? 'Actif' : 'Inactif' }}
            </span>
          </td>
          <td class="actions">
            <button class="btn-sm edit" (click)="editProduct(p.id)" title="Modifier">
              <i class='bx bx-edit'></i>
            </button>
            <button class="btn-sm delete" (click)="deleteProduct(p)" title="Supprimer">
              <i class='bx bx-trash'></i>
            </button>
          </td>
        </tr>
      </tbody>
    </table>
  </div>

  <div class="pagination" *ngIf="totalPages > 1">
    <button (click)="prevPage()" [disabled]="currentPage === 0">Precedent</button>
    <span>Page {{ currentPage + 1 }} / {{ totalPages }}</span>
    <button (click)="nextPage()" [disabled]="currentPage >= totalPages - 1">Suivant</button>
  </div>
</div>
```

---

## Etape 8 : Formulaire Produit (Creation / Edition)

### 8.1 - product-form.ts

```typescript
import { Component, OnInit } from '@angular/core';
import { ActivatedRoute, Router } from '@angular/router';
import { ToastrService } from 'ngx-toastr';
import { AdminService } from '../services/admin.service';
import { ProductCreateRequest } from '../models/admin.model';
import { CategoryService } from '../../core/services/category.service';
import { Category } from '../../core/models/product.model';
import { ProductService } from '../../core/services/product.service';

@Component({
  selector: 'app-admin-product-form',
  standalone: false,
  templateUrl: './product-form.html',
  styleUrl: './product-form.css'
})
export class AdminProductForm implements OnInit {
  isEditMode = false;
  productId: number | null = null;
  categories: Category[] = [];
  isSaving = false;

  form: ProductCreateRequest = {
    name: '',
    description: '',
    price: 0,
    discountPrice: undefined,
    stockQuantity: 0,
    categoryId: 0,
    brand: '',
    isActive: true
  };

  // Upload image
  selectedFile: File | null = null;

  constructor(
    private route: ActivatedRoute,
    private router: Router,
    private adminService: AdminService,
    private productService: ProductService,
    private categoryService: CategoryService,
    private toastr: ToastrService
  ) {}

  ngOnInit(): void {
    // Charger les categories
    this.categoryService.getAllCategories().subscribe({
      next: (cats) => this.categories = cats
    });

    // Mode edition ?
    const idParam = this.route.snapshot.paramMap.get('id');
    if (idParam) {
      this.isEditMode = true;
      this.productId = +idParam;
      this.loadProduct(this.productId);
    }
  }

  loadProduct(id: number): void {
    this.productService.getProductById(id).subscribe({
      next: (product) => {
        this.form = {
          name: product.name,
          description: product.description,
          price: product.price,
          discountPrice: product.discountPrice,
          stockQuantity: product.stockQuantity,
          categoryId: product.categoryId || 0,
          brand: product.brand || '',
          isActive: product.isActive
        };
      },
      error: () => {
        this.toastr.error('Produit introuvable');
        this.router.navigate(['/admin/products']);
      }
    });
  }

  onSubmit(): void {
    if (!this.form.name || !this.form.price || !this.form.categoryId) {
      this.toastr.warning('Veuillez remplir les champs obligatoires');
      return;
    }

    this.isSaving = true;

    const obs = this.isEditMode && this.productId
      ? this.adminService.updateProduct(this.productId, this.form)
      : this.adminService.createProduct(this.form);

    obs.subscribe({
      next: (product) => {
        // Upload image si fichier selectionne
        if (this.selectedFile) {
          this.adminService.uploadProductImage(product.id, this.selectedFile, true).subscribe({
            next: () => {
              this.toastr.success(this.isEditMode ? 'Produit mis a jour' : 'Produit cree');
              this.router.navigate(['/admin/products']);
            },
            error: () => {
              this.toastr.warning('Produit sauvegarde mais erreur upload image');
              this.router.navigate(['/admin/products']);
            }
          });
        } else {
          this.toastr.success(this.isEditMode ? 'Produit mis a jour' : 'Produit cree');
          this.router.navigate(['/admin/products']);
        }
        this.isSaving = false;
      },
      error: () => {
        this.toastr.error('Erreur lors de la sauvegarde');
        this.isSaving = false;
      }
    });
  }

  onFileSelected(event: Event): void {
    const input = event.target as HTMLInputElement;
    this.selectedFile = input.files?.[0] || null;
  }

  cancel(): void {
    this.router.navigate(['/admin/products']);
  }
}
```

### 8.2 - product-form.html

```html
<div class="product-form-page">
  <div class="page-header">
    <h2>{{ isEditMode ? 'Modifier le produit' : 'Nouveau produit' }}</h2>
  </div>

  <div class="form-card">
    <form (ngSubmit)="onSubmit()">

      <div class="form-row">
        <div class="form-group">
          <label>Nom du produit *</label>
          <input type="text" [(ngModel)]="form.name" name="name" required placeholder="Nom du produit">
        </div>
        <div class="form-group">
          <label>Marque</label>
          <input type="text" [(ngModel)]="form.brand" name="brand" placeholder="Marque">
        </div>
      </div>

      <div class="form-group">
        <label>Description</label>
        <textarea [(ngModel)]="form.description" name="description" rows="4" placeholder="Description du produit"></textarea>
      </div>

      <div class="form-row">
        <div class="form-group">
          <label>Prix (FCFA) *</label>
          <input type="number" [(ngModel)]="form.price" name="price" required min="1">
        </div>
        <div class="form-group">
          <label>Prix promo (FCFA)</label>
          <input type="number" [(ngModel)]="form.discountPrice" name="discountPrice" min="0">
        </div>
      </div>

      <div class="form-row">
        <div class="form-group">
          <label>Stock *</label>
          <input type="number" [(ngModel)]="form.stockQuantity" name="stockQuantity" required min="0">
        </div>
        <div class="form-group">
          <label>Categorie *</label>
          <select [(ngModel)]="form.categoryId" name="categoryId" required>
            <option [ngValue]="0" disabled>Choisir une categorie</option>
            <option *ngFor="let cat of categories" [ngValue]="cat.id">{{ cat.name }}</option>
          </select>
        </div>
      </div>

      <div class="form-group">
        <label>Image du produit</label>
        <input type="file" accept="image/*" (change)="onFileSelected($event)">
      </div>

      <div class="form-group">
        <label class="checkbox-label">
          <input type="checkbox" [(ngModel)]="form.isActive" name="isActive">
          Produit actif (visible dans la boutique)
        </label>
      </div>

      <div class="form-actions">
        <button type="button" class="btn-cancel" (click)="cancel()">Annuler</button>
        <button type="submit" class="btn-primary" [disabled]="isSaving">
          <i class='bx bx-loader-alt bx-spin' *ngIf="isSaving"></i>
          {{ isEditMode ? 'Mettre a jour' : 'Creer le produit' }}
        </button>
      </div>
    </form>
  </div>
</div>
```

---

## Etape 9 : Page Gestion Commandes

> Les endpoints backend sont en place :
> - `GET /api/admin/orders` (pagination, filtre par statut, tri)
> - `GET /api/admin/orders/{id}`
> - `PUT /api/admin/orders/{id}/status?status=`
> - `PUT /api/admin/orders/{id}/tracking?trackingNumber=`

### 9.1 - orders.ts

```typescript
import { Component, OnInit } from '@angular/core';
import { ToastrService } from 'ngx-toastr';
import { AdminService } from '../services/admin.service';
import { OrderDTO, OrderStatus } from '../models/admin.model';

@Component({
  selector: 'app-admin-orders',
  standalone: false,
  templateUrl: './orders.html',
  styleUrl: './orders.css'
})
export class AdminOrders implements OnInit {
  orders: OrderDTO[] = [];
  currentPage = 0;
  totalPages = 0;
  totalElements = 0;
  isLoading = false;
  statusFilter = '';
  selectedOrder: OrderDTO | null = null;
  trackingInput = '';

  statuses: { value: string; label: string }[] = [
    { value: '', label: 'Tous' },
    { value: 'PENDING', label: 'En attente' },
    { value: 'CONFIRMED', label: 'Confirmees' },
    { value: 'PROCESSING', label: 'En traitement' },
    { value: 'SHIPPED', label: 'Expediees' },
    { value: 'DELIVERED', label: 'Livrees' },
    { value: 'CANCELLED', label: 'Annulees' }
  ];

  constructor(
    private adminService: AdminService,
    private toastr: ToastrService
  ) {}

  ngOnInit(): void {
    this.loadOrders();
  }

  loadOrders(page = 0): void {
    this.isLoading = true;
    this.adminService.getOrders(page, 10, this.statusFilter || undefined).subscribe({
      next: (data) => {
        this.orders = data.content;
        this.currentPage = data.number;
        this.totalPages = data.totalPages;
        this.totalElements = data.totalElements;
        this.isLoading = false;
      },
      error: () => {
        this.toastr.error('Erreur chargement commandes');
        this.isLoading = false;
      }
    });
  }

  filterByStatus(): void {
    this.loadOrders(0);
  }

  viewOrder(order: OrderDTO): void {
    this.selectedOrder = order;
    this.trackingInput = order.trackingNumber || '';
  }

  closeDetail(): void {
    this.selectedOrder = null;
  }

  updateStatus(orderId: number, newStatus: string): void {
    this.adminService.updateOrderStatus(orderId, newStatus).subscribe({
      next: (updated) => {
        this.toastr.success(`Statut mis a jour : ${newStatus}`);
        // Mettre a jour dans la liste
        const idx = this.orders.findIndex(o => o.id === orderId);
        if (idx >= 0) this.orders[idx] = updated;
        if (this.selectedOrder?.id === orderId) this.selectedOrder = updated;
      },
      error: (err) => {
        const msg = err.error?.message || 'Transition de statut invalide';
        this.toastr.error(msg);
      }
    });
  }

  setTracking(orderId: number): void {
    if (!this.trackingInput.trim()) {
      this.toastr.warning('Veuillez saisir un numero de suivi');
      return;
    }
    this.adminService.setTrackingNumber(orderId, this.trackingInput.trim()).subscribe({
      next: (updated) => {
        this.toastr.success('Numero de suivi enregistre');
        const idx = this.orders.findIndex(o => o.id === orderId);
        if (idx >= 0) this.orders[idx] = updated;
        if (this.selectedOrder?.id === orderId) this.selectedOrder = updated;
      },
      error: () => this.toastr.error('Erreur enregistrement tracking')
    });
  }

  getStatusLabel(status: string): string {
    return this.statuses.find(s => s.value === status)?.label || status;
  }

  getStatusClass(status: string): string {
    return status.toLowerCase();
  }

  getNextStatuses(currentStatus: string): string[] {
    // Transitions autorisees (voir backend validateStatusTransition)
    switch (currentStatus) {
      case 'PENDING':    return ['CONFIRMED', 'CANCELLED'];
      case 'CONFIRMED':  return ['PROCESSING', 'SHIPPED', 'CANCELLED'];
      case 'PROCESSING': return ['SHIPPED'];
      case 'SHIPPED':    return ['DELIVERED'];
      default:           return [];
    }
  }

  formatCurrency(amount: number): string {
    return new Intl.NumberFormat('fr-FR').format(amount) + ' FCFA';
  }

  prevPage(): void {
    if (this.currentPage > 0) this.loadOrders(this.currentPage - 1);
  }

  nextPage(): void {
    if (this.currentPage < this.totalPages - 1) this.loadOrders(this.currentPage + 1);
  }
}
```

### 9.2 - orders.html

```html
<div class="orders-page">
  <div class="page-header">
    <h2>Gestion des commandes</h2>
    <span class="total-badge">{{ totalElements }} commandes</span>
  </div>

  <!-- Filtre par statut -->
  <div class="filter-bar">
    <label>Filtrer par statut :</label>
    <select [(ngModel)]="statusFilter" (change)="filterByStatus()">
      <option *ngFor="let s of statuses" [value]="s.value">{{ s.label }}</option>
    </select>
  </div>

  <!-- Table des commandes -->
  <div class="table-wrapper">
    <table class="data-table">
      <thead>
        <tr>
          <th>N° Commande</th>
          <th>Client</th>
          <th>Total</th>
          <th>Articles</th>
          <th>Statut</th>
          <th>Paiement</th>
          <th>Date</th>
          <th>Actions</th>
        </tr>
      </thead>
      <tbody>
        <tr *ngFor="let order of orders">
          <td class="order-number">{{ order.orderNumber }}</td>
          <td>{{ order.userEmail }}</td>
          <td>{{ formatCurrency(order.totalAmount) }}</td>
          <td>{{ order.totalItems }}</td>
          <td>
            <span class="status-badge" [ngClass]="getStatusClass(order.status)">
              {{ getStatusLabel(order.status) }}
            </span>
          </td>
          <td>{{ order.paymentMethod }}</td>
          <td>{{ order.createdAt | date:'dd/MM/yyyy HH:mm' }}</td>
          <td class="actions">
            <button class="btn-sm view" (click)="viewOrder(order)" title="Details">
              <i class='bx bx-show'></i>
            </button>
            <button *ngFor="let next of getNextStatuses(order.status)"
                    class="btn-sm status-action" [ngClass]="getStatusClass(next)"
                    (click)="updateStatus(order.id, next)"
                    [title]="getStatusLabel(next)">
              {{ getStatusLabel(next) }}
            </button>
          </td>
        </tr>
      </tbody>
    </table>

    <p *ngIf="orders.length === 0 && !isLoading" class="empty">Aucune commande trouvee</p>
  </div>

  <!-- Pagination -->
  <div class="pagination" *ngIf="totalPages > 1">
    <button (click)="prevPage()" [disabled]="currentPage === 0">Precedent</button>
    <span>Page {{ currentPage + 1 }} / {{ totalPages }}</span>
    <button (click)="nextPage()" [disabled]="currentPage >= totalPages - 1">Suivant</button>
  </div>

  <!-- Detail commande (modale) -->
  <div class="modal-overlay" *ngIf="selectedOrder" (click)="closeDetail()">
    <div class="modal-content" (click)="$event.stopPropagation()">
      <div class="modal-header">
        <h3>Commande {{ selectedOrder.orderNumber }}</h3>
        <button class="close-btn" (click)="closeDetail()"><i class='bx bx-x'></i></button>
      </div>

      <div class="order-detail">
        <div class="detail-row">
          <span class="label">Client :</span>
          <span>{{ selectedOrder.userEmail }}</span>
        </div>
        <div class="detail-row">
          <span class="label">Statut :</span>
          <span class="status-badge" [ngClass]="getStatusClass(selectedOrder.status)">
            {{ getStatusLabel(selectedOrder.status) }}
          </span>
        </div>
        <div class="detail-row">
          <span class="label">Total :</span>
          <span class="total">{{ formatCurrency(selectedOrder.totalAmount) }}</span>
        </div>
        <div class="detail-row">
          <span class="label">Paiement :</span>
          <span>{{ selectedOrder.paymentMethod }} ({{ selectedOrder.paymentStatus }})</span>
        </div>
        <div class="detail-row">
          <span class="label">Livraison :</span>
          <span>{{ selectedOrder.shippingFullName }} — {{ selectedOrder.shippingAddress }}</span>
        </div>
        <div class="detail-row" *ngIf="selectedOrder.shippingPhone">
          <span class="label">Telephone :</span>
          <span>{{ selectedOrder.shippingPhone }}</span>
        </div>
        <div class="detail-row" *ngIf="selectedOrder.notes">
          <span class="label">Notes :</span>
          <span>{{ selectedOrder.notes }}</span>
        </div>

        <!-- Tracking -->
        <div class="tracking-section">
          <label>Numero de suivi :</label>
          <div class="tracking-input">
            <input type="text" [(ngModel)]="trackingInput" placeholder="Ex: TRACK-12345">
            <button class="btn-primary" (click)="setTracking(selectedOrder.id)">Enregistrer</button>
          </div>
          <p class="tracking-hint" *ngIf="selectedOrder.status === 'CONFIRMED' || selectedOrder.status === 'PROCESSING'">
            Ajouter un tracking passera automatiquement la commande en "Expediee".
          </p>
        </div>

        <!-- Articles -->
        <h4>Articles ({{ selectedOrder.items.length }})</h4>
        <div class="items-list">
          <div class="item-row" *ngFor="let item of selectedOrder.items">
            <span class="item-name">{{ item.productName }}</span>
            <span class="item-qty">x{{ item.quantity }}</span>
            <span class="item-price">{{ formatCurrency(item.subtotal) }}</span>
          </div>
        </div>

        <!-- Actions de statut -->
        <div class="status-actions" *ngIf="getNextStatuses(selectedOrder.status).length > 0">
          <h4>Changer le statut</h4>
          <div class="action-buttons">
            <button *ngFor="let next of getNextStatuses(selectedOrder.status)"
                    class="btn-status" [ngClass]="getStatusClass(next)"
                    (click)="updateStatus(selectedOrder.id, next)">
              {{ getStatusLabel(next) }}
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>
```

---

## Etape 10 : Integration dans app-routing.module.ts

```typescript
// app-routing.module.ts — ajouter AVANT le bloc Layouts :

export const routes: Routes = [
  { path: '', redirectTo: 'login', pathMatch: 'full' },
  { path: 'login', component: Login, canActivate: [publicGuard] },
  { path: 'forgot-password', component: ForgotPassword, canActivate: [publicGuard] },
  { path: 'reset-password', component: ResetPassword },

  // ===== ADMIN (lazy-loaded) =====
  {
    path: 'admin',
    loadChildren: () => import('./admin/admin.module').then(m => m.AdminModule)
  },

  // ===== SITE PUBLIC (apres login) =====
  {
    path: '',
    component: Layouts,
    canActivate: [authGuard],
    canActivateChild: [authGuard],
    children: [
      { path: 'Home', component: Home },
      // ... autres routes
    ]
  },

  // ===== 404 =====
  { path: '**', redirectTo: 'Home' }
];
```

---

## Etape 11 : Lien vers l'admin dans la navbar

Dans `navbar.html`, ajouter un lien conditionnel pour les admins :

```html
<!-- Dans ul.nav__links, avant DECONNEXION -->
<li *ngIf="isAdmin">
  <a routerLink="/admin" routerLinkActive="active">ADMIN</a>
</li>
```

Dans `navbar.ts` :

```typescript
import { AuthService } from '../../core/services/auth.service';

// Dans le constructeur, auth est deja injecte

get isAdmin(): boolean {
  return this.auth.isAdmin();
}
```

---

## Resume des fichiers a creer

| # | Fichier | Type |
|---|---------|------|
| 1 | `src/app/admin/models/admin.model.ts` | Models/Interfaces |
| 2 | `src/app/admin/services/admin.service.ts` | Service HTTP |
| 3 | `src/app/admin/admin.module.ts` | NgModule |
| 4 | `src/app/admin/admin-routing.module.ts` | Routes |
| 5 | `src/app/admin/admin-layout/admin-layout.ts` | Composant |
| 6 | `src/app/admin/admin-layout/admin-layout.html` | Template |
| 7 | `src/app/admin/admin-layout/admin-layout.css` | Styles |
| 8 | `src/app/admin/dashboard/dashboard.ts` | Composant |
| 9 | `src/app/admin/dashboard/dashboard.html` | Template |
| 10 | `src/app/admin/dashboard/dashboard.css` | Styles |
| 11 | `src/app/admin/users/users.ts` | Composant |
| 12 | `src/app/admin/users/users.html` | Template |
| 13 | `src/app/admin/users/users.css` | Styles |
| 14 | `src/app/admin/products/products.ts` | Composant |
| 15 | `src/app/admin/products/products.html` | Template |
| 16 | `src/app/admin/products/products.css` | Styles |
| 17 | `src/app/admin/product-form/product-form.ts` | Composant |
| 18 | `src/app/admin/product-form/product-form.html` | Template |
| 19 | `src/app/admin/product-form/product-form.css` | Styles |
| 20 | `src/app/admin/orders/orders.ts` | Composant |
| 21 | `src/app/admin/orders/orders.html` | Template |
| 22 | `src/app/admin/orders/orders.css` | Styles |

**Fichiers modifies :**
- `src/app/app-routing.module.ts` (ajouter route admin + wildcard)
- `src/app/components/navbar/navbar.ts` (ajouter `isAdmin`)
- `src/app/components/navbar/navbar.html` (ajouter lien ADMIN)

---

## Endpoints backend utilises

| Endpoint | Utilise dans |
|----------|-------------|
| `GET /api/admin/dashboard/stats` | Dashboard |
| `GET /api/admin/dashboard/sales-chart` | Dashboard |
| `GET /api/admin/dashboard/top-products` | Dashboard |
| `GET /api/admin/dashboard/recent-activities` | Dashboard |
| `GET /api/admin/dashboard/revenue` | Dashboard |
| `GET /api/admin/users` | Users |
| `GET /api/admin/users/search` | Users |
| `PUT /api/admin/users/{id}/toggle-status` | Users |
| `DELETE /api/admin/users/{id}` | Users |
| `GET /api/products` | Products |
| `POST /api/products` | Product Form |
| `PUT /api/products/{id}` | Product Form |
| `DELETE /api/products/{id}` | Products |
| `POST /api/products/{id}/images` | Product Form |
| `GET /api/categories` | Product Form |
| `GET /api/admin/orders` | Orders (pagination, filtre statut, tri) |
| `GET /api/admin/orders/{id}` | Orders (detail) |
| `PUT /api/admin/orders/{id}/status` | Orders (changement statut) |
| `PUT /api/admin/orders/{id}/tracking` | Orders (numero de suivi) |

## Lacunes backend a combler plus tard

| Fonctionnalite manquante | Endpoint necessaire |
|--------------------------|---------------------|
| Securiser l'inscription admin | Proteger `POST /api/auth/register-admin` |
