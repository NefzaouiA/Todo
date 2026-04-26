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

---

## 13. LOGIN & REGISTER — Détail complet côte à côte

> Les deux composants sont quasi-identiques. Voici les différences exactes.

| | `login.ts` | `register.ts` |
|--|-----------|--------------|
| Méthode | `login()` | `register()` |
| Appel service | `authService.login(email, password)` | `authService.register(email, password)` |
| Redirection | `navigate(['/items'])` | `navigate(['/items'])` ou `navigate(['/login'])` |
| Message erreur | `'Email ou mot de passe incorrect'` | `'Email déjà utilisé ou invalide'` |
| Lien bas de page | `routerLink="/register"` | `routerLink="/login"` |

### login.ts complet
```typescript
import { Component, inject } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';
import { Router, RouterLink } from '@angular/router';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-login',
  imports: [FormsModule, CommonModule, RouterLink],
  templateUrl: './login.html',
})
export class Login {
  email = '';
  password = '';
  errorMessage = '';

  private authService = inject(AuthService);
  private router = inject(Router);

  login() {
    if (!this.email || !this.password) {
      this.errorMessage = 'Veuillez remplir tous les champs';
      return;
    }
    this.authService.login(this.email, this.password)
      .then(() => this.router.navigate(['/items']))
      .catch(() => this.errorMessage = 'Email ou mot de passe incorrect');
  }
}
```

### register.ts complet
```typescript
import { Component, inject } from '@angular/core';
import { FormsModule } from '@angular/forms';
import { CommonModule } from '@angular/common';
import { Router, RouterLink } from '@angular/router';
import { AuthService } from '../../services/auth.service';

@Component({
  selector: 'app-register',
  imports: [FormsModule, CommonModule, RouterLink],
  templateUrl: './register.html',
})
export class Register {
  email = '';
  password = '';
  errorMessage = '';

  private authService = inject(AuthService);
  private router = inject(Router);

  register() {
    if (!this.email || !this.password) {
      this.errorMessage = 'Veuillez remplir tous les champs';
      return;
    }
    this.authService.register(this.email, this.password)
      .then(() => this.router.navigate(['/items']))
      .catch(() => this.errorMessage = 'Email déjà utilisé ou invalide');
  }
}
```

### login.html complet
```html
<form (ngSubmit)="login()">
  <h1>Connexion</h1>
  <input type="email"    name="email"    [(ngModel)]="email"    placeholder="Email" />
  <input type="password" name="password" [(ngModel)]="password" placeholder="Mot de passe" />
  <button type="submit">Se connecter</button>
  <p>Pas de compte ? <a routerLink="/register">Créer un compte</a></p>
  <div *ngIf="errorMessage.length > 0">{{ errorMessage }}</div>
</form>
```

### register.html complet
```html
<form (ngSubmit)="register()">
  <h1>Créer un compte</h1>
  <input type="email"    name="email"    [(ngModel)]="email"    placeholder="Email" />
  <input type="password" name="password" [(ngModel)]="password" placeholder="Mot de passe" />
  <button type="submit">S'inscrire</button>
  <p>Déjà un compte ? <a routerLink="/login">Se connecter</a></p>
  <div *ngIf="errorMessage.length > 0">{{ errorMessage }}</div>
</form>
```

---

## 14. COMMANDES ANGULAR CLI — Les essentielles

### Démarrer l'application
```bash
ng serve                          # Lance le serveur de dev → http://localhost:4200
ng serve --open                   # Lance ET ouvre le navigateur automatiquement
ng serve --port 4201              # Changer le port si 4200 est déjà utilisé
```

### Générer des fichiers (évite les erreurs de boilerplate)
```bash
# Générer un composant
ng generate component components/login
ng g c components/login            # version courte
# → crée login.ts, login.html, login.css, login.spec.ts

# Générer un service
ng generate service services/auth
ng g s services/auth               # version courte
# → crée auth.service.ts, auth.service.spec.ts

# Générer une interface (le modèle de données)
ng generate interface models/item
ng g i models/item

# Générer un guard (protection de routes)
ng generate guard guards/auth
ng g g guards/auth
```

### Options utiles pour la génération
```bash
ng g c components/login --skip-tests    # Sans le fichier .spec.ts
ng g s services/task --skip-tests       # Service sans tests
```

### Vérification et build
```bash
ng build                          # Build de production (dossier dist/)
ng build --watch                  # Build en mode watch (recompile à chaque save)
ng lint                           # Vérifier les erreurs de style (ESLint)
ng test                           # Lancer les tests unitaires
ng version                        # Voir la version d'Angular installée
ng --version                      # Vérifier Angular CLI + Node installés
```

> ⚠️ **À l'examen** : utilise `ng g c` et `ng g s` pour ne pas créer les fichiers manuellement avec des erreurs de nommage.

---

## 15. COMMANDES NPM — Installer les dépendances

```bash
# Installer toutes les dépendances (TOUJOURS faire ça en premier sur un projet cloné)
npm install
npm i                             # version courte

# Installer une dépendance spécifique
npm install @angular/fire         # Firebase pour Angular
npm install firebase              # SDK Firebase core
npm install -g @angular/cli       # Installer Angular CLI globalement

# Vérifier ce qui est installé
npm list --depth=0                # Voir toutes les dépendances installées

# Supprimer node_modules et réinstaller (si bug bizarre)
Remove-Item -Recurse -Force node_modules; npm install   # PowerShell Windows
```

> ⚠️ **À l'examen** : si le projet squelette est fourni, fait toujours `npm install` **en PREMIER** avant de toucher au code.

---

## 16. COMMANDES FIREBASE — Lancer les émulateurs

> L'examen utilise des **émulateurs locaux** (pas le vrai Firebase cloud).

### Lancer les émulateurs
```bash
firebase emulators:start                           # Démarre tous les émulateurs configurés
firebase emulators:start --only auth,firestore     # Uniquement Auth et Firestore
firebase emulators:start --import=./firebase-data  # Avec données pré-chargées (si fournies)
```

### Ports des émulateurs (FIXES à l'examen)
```
Auth      → http://localhost:9099
Firestore → http://localhost:8080
UI admin  → http://localhost:4000   ← interface pour voir/modifier les données
Angular   → http://localhost:4200   ← ton application
```

### Gérer les projets Firebase
```bash
firebase login                    # Se connecter
firebase login --no-localhost     # Si environnement restreint (pas de navigateur)
firebase projects:list            # Voir les projets disponibles
firebase use <project-id>         # Sélectionner un projet
firebase use --add                # Ajouter un projet à .firebaserc
```

### Déployer (si demandé)
```bash
firebase deploy                   # Tout déployer
firebase deploy --only hosting    # Seulement le frontend
firebase deploy --only firestore  # Seulement les règles Firestore
```

### Fichiers Firebase importants
```bash
.firebaserc          # → quel projet est actif
firebase.json        # → config émulateurs + hosting
firestore.rules      # → règles de sécurité
src/environements.ts # → clés Firebase (⚠️ vérifier l'orthographe !)
```

### Ordre de démarrage COMPLET à l'examen
```bash
# ÉTAPE 1 — Dans le dossier du projet
npm install

# ÉTAPE 2 — Terminal 1 : Firebase
firebase emulators:start

# ÉTAPE 3 — Terminal 2 : Angular
ng serve --open

# ÉTAPE 4 — Vérifier
# http://localhost:4200  → ton app Angular
# http://localhost:4000  → UI Firebase (voir les users/données en temps réel)
```

---

## 17. TYPESCRIPT — Syntaxes utiles à l'examen

```typescript
// Interface — modèle de données
export interface Item {
  id: string;
  title: string;
  done?: boolean;          // ? = champ optionnel
  userId: string;
}

// Partial — tous les champs deviennent optionnels (pour update)
function update(data: Partial<Item>) { }
// Permet : update({ title: 'x' })  sans fournir tous les champs

// Non-null assertion — "je garantis que ce n'est pas null"
items$!: Observable<Item[]>;     // le ! = sera initialisé dans ngOnInit

// Type assertion — forcer le type retourné
return collectionData(q) as Observable<Item[]>;

// Optional chaining — éviter les erreurs sur null/undefined
const uid = user?.uid;           // ne plante pas si user est null

// Nullish coalescing — valeur par défaut
const name = user?.displayName ?? 'Anonyme';

// Type union
type Status = 'active' | 'done' | 'deleted';
```

---

## 18. DIAGNOSTIC RAPIDE — Que faire si ça ne marche pas

| Symptôme | Cause probable | Solution |
|----------|---------------|----------|
| `Cannot find module '...'` | Import manquant ou mauvais chemin | Vérifier `../../services/...` |
| `NullInjectorError: No provider for X` | `@Injectable` manquant | Ajouter `@Injectable({ providedIn: 'root' })` |
| Page blanche / erreur CORS | Émulateurs pas lancés | `firebase emulators:start` |
| `*ngFor` ne s'affiche pas | `| async` manquant | `*ngFor="let x of list$ | async"` |
| `[(ngModel)]` ne fonctionne pas | `FormsModule` manquant | Ajouter dans `imports: [FormsModule]` |
| `routerLink` ne fonctionne pas | `RouterLink` manquant | Ajouter dans `imports: [RouterLink]` |
| `*ngIf` / `*ngFor` inconnu | `CommonModule` manquant | Ajouter dans `imports: [CommonModule]` |
| Données pas filtrées par user | `where()` manquant | Vérifier le `query()` dans le service |
| App ne démarre pas | `node_modules` absent | `npm install` |
| Port 4200 déjà utilisé | Autre app en cours | `ng serve --port 4201` |
| `user.uid` undefined | Hors du `.subscribe()` | Lire `user.uid` dans le callback subscribe |
| Rien dans Firestore UI | Collection mal nommée | `'tasks'` pas `'Tasks'` (minuscule!) |

---

*💡 Adapte les noms : `Item` → `Movie` / `Task` / `Expense` selon le thème de l'examen.*
