# 🚀 ANGULAR + FIREBASE — EXAM CHEAT SHEET

---

## 1. ARCHITECTURE — Ce que tu dois créer

```
app.config.ts          → Configurer Firebase
app.routes.ts          → Déclarer les routes
src/app/
├── services/
│   ├── auth.service.ts      → login, register, logout, user$
│   └── [item].service.ts    → getItems, addItem, deleteItem, updateItem
└── components/
    ├── login/               → Formulaire login
    ├── register/            → Formulaire register
    └── [item]-list/         → Page principale CRUD
```

---

## 2. app.config.ts — Configurer Firebase

```typescript
import { ApplicationConfig } from '@angular/core';
import { provideRouter } from '@angular/router';
import { provideFirebaseApp, initializeApp } from '@angular/fire/app';
import { provideFirestore, getFirestore, connectFirestoreEmulator } from '@angular/fire/firestore';
import { provideAuth, getAuth, connectAuthEmulator } from '@angular/fire/auth';
import { routes } from './app.routes';
import { environment } from '../environements';  // ⚠️ vérifier le nom du fichier

export const appConfig: ApplicationConfig = {
  providers: [
    provideRouter(routes),
    provideFirebaseApp(() => initializeApp(environment.firebase)),
    provideFirestore(() => {
      const firestore = getFirestore();
      connectFirestoreEmulator(firestore, 'localhost', 8080); // ← port FIXE
      return firestore;
    }),
    provideAuth(() => {
      const auth = getAuth();
      connectAuthEmulator(auth, 'http://localhost:9099'); // ← port FIXE
      return auth;
    }),
  ]
};
```

**Ports à retenir :**
- Firestore → **8080**
- Auth → **9099**

---

## 3. app.routes.ts — Les routes

```typescript
import { Routes } from '@angular/router';
import { Login } from './components/login/login';
import { Register } from './components/register/register';
import { ItemList } from './components/item-list/item-list';  // ← nom de CLASSE TypeScript

export const routes: Routes = [
  { path: '',         redirectTo: 'register', pathMatch: 'full' },
  { path: 'login',    component: Login },
  { path: 'register', component: Register },
  { path: 'items',    component: ItemList },   // ← 'items' = chemin URL (minuscule)
];
```

> ⚠️ `path: 'items'` (string URL) ≠ `component: ItemList` (classe TypeScript importée)

---

## 4. auth.service.ts — Authentication

```typescript
import { Injectable, inject } from '@angular/core';
import { Auth, createUserWithEmailAndPassword,
         signInWithEmailAndPassword, signOut, user, User } from '@angular/fire/auth';
import { Observable } from 'rxjs';

@Injectable({ providedIn: 'root' })
export class AuthService {

  private auth = inject(Auth);

  user$: Observable<User | null> = user(this.auth);  // ← émet l'user connecté en temps réel

  register(email: string, password: string) {
    return createUserWithEmailAndPassword(this.auth, email, password);  // retourne Promise
  }

  login(email: string, password: string) {
    return signInWithEmailAndPassword(this.auth, email, password);  // retourne Promise
  }

  logout() {
    return signOut(this.auth);  // retourne Promise
  }
}
```

---

## 5. [item].service.ts — CRUD Firestore

```typescript
import { Injectable, inject } from '@angular/core';
import { Firestore, collection, collectionData,
         addDoc, doc, deleteDoc, updateDoc,
         query, where } from '@angular/fire/firestore';
import { Observable } from 'rxjs';

export interface Item {
  id: string;
  title: string;
  // ... autres champs selon l'examen
  userId: string;
}

@Injectable({ providedIn: 'root' })
export class ItemService {

  private firestore = inject(Firestore);

  // LIRE — filtre par userId
  getItems(userId: string): Observable<Item[]> {
    const ref = collection(this.firestore, 'items');       // ← nom de la collection Firestore (minuscule)
    const q   = query(ref, where('userId', '==', userId)); // ← filtre par user
    return collectionData(q, { idField: 'id' }) as Observable<Item[]>;
  }

  // AJOUTER
  addItem(title: string, userId: string) {
    const ref = collection(this.firestore, 'items');
    return addDoc(ref, { title, userId });  // ← Firestore génère l'id automatiquement
  }

  // SUPPRIMER
  deleteItem(id: string) {
    return deleteDoc(doc(this.firestore, 'items', id));
  }

  // MODIFIER (Partial = tous les champs optionnels)
  updateItem(id: string, data: Partial<Item>) {
    return updateDoc(doc(this.firestore, 'items', id), data);
  }
}
```

---

## 6. [item]-list.ts — Composant principal

```typescript
import { Component, inject, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { Observable } from 'rxjs';
import { Router } from '@angular/router';
import { ItemService, Item } from '../../services/item.service';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-item-list',
  imports: [CommonModule, FormsModule],   // ← CommonModule pour *ngFor/*ngIf, FormsModule pour [(ngModel)]
  templateUrl: './item-list.html',
})
export class ItemList implements OnInit {

  private itemService = inject(ItemService);
  private authService = inject(AuthService);
  private router      = inject(Router);

  items$!: Observable<Item[]>;   // ← Observable = données temps réel
  newTitle = '';
  private userId = '';

  ngOnInit(): void {
    this.authService.user$.subscribe(user => {
      if (user) {
        this.userId = user.uid;                          // ← identifiant unique Firebase
        this.items$ = this.itemService.getItems(this.userId);
      } else {
        this.router.navigate(['/login']);                // ← sécurité : redirige si pas connecté
      }
    });
  }

  addItem() {
    if (this.newTitle.trim() === '') return;
    this.itemService.addItem(this.newTitle.trim(), this.userId);
    this.newTitle = '';                                  // ← reset l'input
  }

  deleteItem(id: string) {
    this.itemService.deleteItem(id);
  }

  logout() {
    this.authService.logout().then(() => this.router.navigate(['/login']));
  }
}
```

---

## 7. login.ts + register.ts — Auth Composants

```typescript
// ✅ PATTERN IDENTIQUE pour login.ts et register.ts

@Component({
  selector: 'app-login',
  imports: [FormsModule, CommonModule, RouterLink],  // ← RouterLink pour <a routerLink="...">
  templateUrl: './login.html',
})
export class Login {
  email = '';
  password = '';
  errorMessage = '';

  private authService = inject(AuthService);
  private router      = inject(Router);

  login() {               // → register() pour register.ts
    if (!this.email || !this.password) {
      this.errorMessage = 'Champs manquants';
      return;
    }
    this.authService.login(this.email, this.password)  // → .register() pour register.ts
      .then(() => this.router.navigate(['/items']))
      .catch(() => this.errorMessage = 'Identifiants incorrects');
  }
}
```

---

## 8. Templates HTML — Syntaxe de référence

### login.html / register.html
```html
<form (ngSubmit)="login()">                          <!-- (ngSubmit) = Entrée + clic -->
  <input type="email"    name="email"    [(ngModel)]="email"    />
  <input type="password" name="password" [(ngModel)]="password" />
  <button type="submit">Se connecter</button>
  <p>Pas de compte ? <a routerLink="/register">S'inscrire</a></p>
  <div *ngIf="errorMessage.length > 0">{{ errorMessage }}</div>
</form>
```

### item-list.html
```html
<!-- Ajout -->
<input [(ngModel)]="newTitle" placeholder="Nouveau..." />
<button (click)="addItem()">Ajouter</button>

<!-- Liste temps réel via | async -->
<div *ngFor="let item of items$ | async">
  <span>{{ item.title }}</span>
  <button (click)="deleteItem(item.id)">🗑️</button>
</div>

<!-- Logout -->
<button (click)="logout()">Se déconnecter</button>
```

### Checkbox (toggle done/undone)
```html
<input type="checkbox"
       [checked]="item.done"
       (change)="toggleDone(item)" />
```

---

## 9. LES 4 BINDINGS — Résumé express

| Syntaxe | Sens | Usage |
|---------|------|-------|
| `{{ value }}` | TS → HTML | Afficher du texte |
| `[property]="value"` | TS → HTML | Lier une prop (ex: `[checked]`) |
| `(event)="method()"` | HTML → TS | Réagir à un event (click, change…) |
| `[(ngModel)]="field"` | TS ↔ HTML | Input formulaire (two-way) |

---

## 10. DIRECTIVES STRUCTURELLES

```html
<!-- Boucle -->
<div *ngFor="let item of items">{{ item.name }}</div>

<!-- Boucle sur Observable (ne pas oublier | async) -->
<div *ngFor="let item of items$ | async">{{ item.name }}</div>

<!-- Condition -->
<div *ngIf="condition">Visible si vrai</div>
<div *ngIf="errorMessage.length > 0">{{ errorMessage }}</div>
```

---

## 11. PIÈGES CLASSIQUES ⚠️

| Piège | ❌ Erreur | ✅ Correct |
|-------|----------|----------|
| Nom de collection | `'Items'` | `'items'` (minuscule) |
| Route vs Composant | `component: 'items'` | `component: ItemList` |
| Redirect dans route | `component: Login` | `redirectTo: 'login'` |
| Checkbox | `[(ngModel)]="item.done"` | `[checked] + (change)` |
| Lien interne | `href="/login"` | `routerLink="/login"` |
| Nouvelle tâche | `done: true` | `done: false` |
| Update partiel | `data: Item` | `data: Partial<Item>` |
| Router import | `Router` dans imports[] | Jamais — c'est un service injectable |

---

## 12. IMPORTS RAPIDES — Copier-coller

### Dans le @Component `imports: []`
```typescript
imports: [CommonModule, FormsModule]                    // composants avec *ngFor, *ngIf, [(ngModel)]
imports: [CommonModule, FormsModule, RouterLink]        // + liens routerLink dans le HTML
```

### En haut du fichier service
```typescript
import { Injectable, inject } from '@angular/core';
import { Firestore, collection, collectionData, addDoc, doc, deleteDoc, updateDoc, query, where } from '@angular/fire/firestore';
import { Observable } from 'rxjs';
```

### En haut du fichier composant
```typescript
import { Component, inject, OnInit } from '@angular/core';
import { CommonModule } from '@angular/common';
import { FormsModule } from '@angular/forms';
import { Observable } from 'rxjs';
import { Router, RouterLink } from '@angular/router';
```

---

*💡 Adapte les noms : `Item` → `Movie` / `Task` / `Expense` selon le thème de l'examen.*
