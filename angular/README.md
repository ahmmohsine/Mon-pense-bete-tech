# Angular — Vue d'ensemble

Angular est un framework TypeScript permettant de construire des applications web, principalement des applications **Single Page Applications (SPA)**.

Pour bien comprendre Angular, il faut surtout comprendre comment plusieurs mécanismes travaillent ensemble :

```text
Application Angular
│
├── Components
│     ├── Template
│     ├── Class
│     └── Styles
│
├── Services
│
├── Dependency Injection
│
├── Signals / Reactive state
│
├── Inputs / Outputs
│
├── Routing
│
└── HTTP / APIs
```

Cette fiche donne la carte mentale générale. Les mécanismes importants seront détaillés dans les fichiers suivants.

---

# 1. Angular, c'est quoi exactement ?

Angular est un framework frontend basé sur **TypeScript**.

Une application Angular est composée de différents éléments qui collaborent pour afficher et manipuler l'interface utilisateur.

On peut simplifier :

```text
Utilisateur
    ↓
Interface Angular
    ↓
Component
    ↓
Service
    ↓
HTTP
    ↓
API ASP.NET Core
    ↓
Database
```

Dans ton contexte .NET, le schéma typique sera donc :

```text
Angular
   ↓ HTTP
ASP.NET Core Web API
   ↓
EF Core
   ↓
SQL Server
```

Angular s'occupe principalement du frontend.

ASP.NET Core s'occupe principalement du backend.

---

# 2. Component : l'élément central

Le **component** est l'une des notions les plus importantes d'Angular.

Un component regroupe généralement :

```text
Component
│
├── TypeScript
├── HTML
└── CSS
```

Exemple :

```typescript
@Component({
  selector: 'app-users',
  templateUrl: './users.component.html',
  styleUrl: './users.component.css'
})
export class UsersComponent {

  users = [
    'Alice',
    'Bob'
  ];
}
```

Template :

```html
<h1>Users</h1>

<ul>
  <li *ngFor="let user of users">
    {{ user }}
  </li>
</ul>
```

Le component contient donc la logique nécessaire à l'affichage et le template décrit l'interface.

---

# 3. Le template

Le template est le HTML associé au component.

Exemple :

```html
<h1>{{ title }}</h1>
```

Si le component contient :

```typescript
title = 'Hotel Listing';
```

Angular affiche :

```text
Hotel Listing
```

Le mécanisme :

```text
TypeScript
    ↓
state / données
    ↓
Template
    ↓
DOM
```

Angular synchronise l'interface avec l'état du component.

---

# 4. Interpolation

L'interpolation utilise :

```html
{{ expression }}
```

Exemple :

```typescript
name = 'Ahlame';
```

```html
<p>Bonjour {{ name }}</p>
```

Résultat :

```text
Bonjour Ahlame
```

On peut également utiliser des expressions simples :

```html
<p>{{ price * quantity }}</p>
```

Mais il vaut mieux éviter de mettre trop de logique dans le template.

---

# 5. Property binding

Le property binding permet de lier une propriété du DOM à une valeur Angular.

Syntaxe :

```html
<img [src]="imageUrl">
```

Ici :

```text
[src]
```

est une propriété du DOM.

Et :

```text
imageUrl
```

vient du component.

Exemple :

```typescript
imageUrl = '/images/hotel.jpg';
```

Angular établit le lien :

```text
imageUrl
   ↓
[src]
   ↓
<img>
```

---

# 6. Event binding

Le event binding permet de réagir aux événements utilisateur.

Syntaxe :

```html
<button (click)="save()">
  Save
</button>
```

Lorsqu'un utilisateur clique :

```text
click
  ↓
save()
  ↓
code TypeScript
```

Exemple :

```typescript
save() {
  console.log('Saved');
}
```

Mental model :

```text
[] = données → interface

() = interface → component
```

---

# 7. Two-way binding

Le two-way binding permet une synchronisation dans les deux directions.

Syntaxe classique :

```html
<input [(ngModel)]="name">
```

Cela signifie conceptuellement :

```text
Component
   ↕
Input
```

Une modification dans le component peut modifier l'input.

Une modification dans l'input peut modifier la valeur du component.

Le symbole :

```text
[( )]
```

est souvent appelé **banana-in-a-box**.

---

# 8. Components parent et enfant

Une application Angular contient généralement une hiérarchie de components.

Exemple :

```text
AppComponent
│
├── HeaderComponent
├── HotelListComponent
│      │
│      ├── HotelCardComponent
│      ├── HotelCardComponent
│      └── HotelCardComponent
│
└── FooterComponent
```

Le parent peut transmettre des données à l'enfant.

L'enfant peut notifier le parent d'un événement.

C'est là qu'interviennent :

```text
Input
Output
```

---

# 9. Input

Un `Input` permet au parent de transmettre une valeur à un enfant.

Exemple :

Parent :

```html
<app-user-card
  [user]="selectedUser">
</app-user-card>
```

Enfant :

```typescript
@Input()
user!: User;
```

Mental model :

```text
Parent
   │
   │ données
   ↓
Child
```

Donc :

> `Input` = parent → enfant.

---

# 10. Output

Un `Output` permet à l'enfant d'émettre un événement vers le parent.

Exemple :

```typescript
@Output()
deleted = new EventEmitter<number>();
```

Puis :

```typescript
deleteUser() {
  this.deleted.emit(this.user.id);
}
```

Parent :

```html
<app-user-card
  (deleted)="deleteUser($event)">
</app-user-card>
```

Mental model :

```text
Parent
   ↑
   │ événement
Child
```

Donc :

> `Output` = enfant → parent.

---

# 11. Services

Un service contient généralement une logique qui ne devrait pas être directement dans le component.

Exemple :

```typescript
@Injectable({
  providedIn: 'root'
})
export class HotelService {

  getHotels() {
    // appel API
  }
}
```

Le component peut utiliser ce service.

```text
Component
    ↓
Service
    ↓
HTTP
    ↓
API
```

Cela permet de séparer :

```text
UI
```

et :

```text
Business / communication logic
```

---

# 12. Dependency Injection

Angular possède son propre système de Dependency Injection.

Exemple :

```typescript
constructor(private hotelService: HotelService) {}
```

Le component demande :

```text
HotelService
```

au système d'injection.

Angular fournit alors l'instance configurée.

Mental model :

```text
Component
   ↓
"J'ai besoin de HotelService"
   ↓
Angular DI
   ↓
HotelService
```

C'est très similaire dans l'esprit au système DI d'ASP.NET Core.

En .NET :

```csharp
builder.Services.AddScoped<IHotelService, HotelService>();
```

Puis :

```csharp
public HotelController(IHotelService service)
{
    _service = service;
}
```

Angular suit un principe comparable :

```text
Dependency Injection
```

mais avec les mécanismes propres à Angular/TypeScript.

---

# 13. Signals

Les Signals sont un mécanisme important dans Angular moderne pour gérer l'état réactif.

Exemple :

```typescript
count = signal(0);
```

Lire la valeur :

```typescript
count()
```

Modifier :

```typescript
count.set(10);
```

Ou :

```typescript
count.update(value => value + 1);
```

Mental model :

```text
signal
   ↓
state réactif
   ↓
Angular sait quand la valeur change
   ↓
UI peut être mise à jour
```

---

# 14. Computed

`computed()` permet de créer une valeur dérivée à partir de Signals.

Exemple :

```typescript
firstName = signal('Ahlame');
lastName = signal('Mohsine');

fullName = computed(() =>
  `${firstName()} ${lastName()}`
);
```

Ici :

```text
firstName
lastName
   ↓
computed
   ↓
fullName
```

`fullName` ne représente pas une nouvelle source de vérité.

C'est une valeur calculée à partir d'autres états.

Mental model :

```text
Signal = donnée source

Computed = donnée dérivée
```

---

# 15. `effect`

`effect()` sert à exécuter du code lorsqu'un Signal lu par l'effet change.

Exemple :

```typescript
effect(() => {
  console.log(this.count());
});
```

Si :

```typescript
count.set(10);
```

l'effet peut être réexécuté car il dépend de `count()`.

Il faut cependant éviter d'utiliser `effect()` pour tout.

Pour une valeur dérivée :

```typescript
computed()
```

est généralement plus approprié.

Mental model :

```text
computed = calculer une valeur

effect = effectuer une action secondaire
```

---

# 16. Routing

Angular permet de naviguer entre différentes vues sans recharger toute l'application.

Exemple :

```text
/hotels
/hotels/12
/login
/users
```

Configuration conceptuelle :

```typescript
const routes: Routes = [
  {
    path: 'hotels',
    component: HotelListComponent
  },
  {
    path: 'hotels/:id',
    component: HotelDetailsComponent
  }
];
```

Navigation :

```text
URL
 ↓
Angular Router
 ↓
Route correspondante
 ↓
Component
```

C'est l'un des mécanismes fondamentaux d'une SPA.

---

# 17. HttpClient

Angular communique généralement avec une API grâce à `HttpClient`.

Exemple :

```typescript
this.http.get<Hotel[]>('/api/hotels');
```

Le flux devient :

```text
Angular
   ↓ HTTP GET
ASP.NET Core API
   ↓
Controller
   ↓
Service
   ↓
EF Core
   ↓
Database
```

Puis :

```text
Database
   ↓
EF Core
   ↓
API
   ↓ JSON
Angular
   ↓
Component
   ↓
Template
```

---

# 18. Observables et RxJS

Angular utilise largement **RxJS** pour la programmation réactive.

Exemple :

```typescript
this.http.get<Hotel[]>('/api/hotels')
```

retourne généralement un :

```typescript
Observable<Hotel[]>
```

On peut ensuite utiliser des opérateurs RxJS :

```typescript
pipe(
  map(...),
  filter(...),
  catchError(...)
)
```

Mental model :

```text
Observable
    ↓
flux de valeurs
    ↓
operators
    ↓
résultat
```

Il faut bien distinguer :

```text
Promise
```

et :

```text
Observable
```

Ils ne répondent pas exactement aux mêmes besoins.

---

# 19. Angular et ASP.NET Core : architecture complète

Dans ton contexte .NET, tu peux retenir cette architecture :

```text
┌───────────────────────────┐
│          Angular          │
│                           │
│ Components                │
│ Services                  │
│ Signals                   │
│ Router                    │
│ HttpClient                │
└─────────────┬─────────────┘
              │ HTTP
              │ JSON
              ↓
┌───────────────────────────┐
│      ASP.NET Core API     │
│                           │
│ Controllers               │
│ Services                  │
│ DTOs                      │
│ Authentication            │
│ Authorization             │
└─────────────┬─────────────┘
              ↓
┌───────────────────────────┐
│         EF Core           │
│                           │
│ DbContext                 │
│ Entities                  │
│ LINQ                      │
└─────────────┬─────────────┘
              ↓
┌───────────────────────────┐
│         Database          │
└───────────────────────────┘
```

Cette séparation est extrêmement importante.

---

# 20. Angular ne remplace pas ASP.NET Core

Angular et ASP.NET Core n'ont pas le même rôle.

```text
Angular
= frontend
= interface utilisateur
= interaction utilisateur
= état UI
= appels HTTP
```

```text
ASP.NET Core
= backend
= API
= logique métier
= sécurité
= accès aux données
```

Exemple :

```text
Angular demande :

GET /api/hotels
```

ASP.NET Core répond :

```json
[
  {
    "id": 1,
    "name": "Hotel Royal"
  }
]
```

Angular transforme ensuite ces données en interface.

---

# 21. Où placer la logique ?

Une erreur fréquente est de mettre toute la logique dans les components.

Mauvaise organisation :

```text
Component
 ├── affichage
 ├── appels HTTP
 ├── business logic
 ├── transformation des données
 └── gestion complexe de l'état
```

On préfère généralement :

```text
Component
    ↓
Service
    ↓
API
```

avec une séparation claire des responsabilités.

Le component doit principalement gérer l'interface et l'état lié à cette interface.

---

# 22. Mental model général Angular

Quand tu regardes une application Angular, pense :

```text
COMPONENT
    ↓
affiche et réagit

SERVICE
    ↓
centralise la logique / communication

SIGNAL
    ↓
gère un état réactif

INPUT
    ↓
parent → enfant

OUTPUT
    ↓
enfant → parent

ROUTER
    ↓
URL → component

HTTP
    ↓
Angular → API

RXJS
    ↓
flux asynchrones
```

---

# 23. Ordre conseillé pour apprendre Angular

Pour éviter d'apprendre les concepts dans le désordre :

```text
1. Components
        ↓
2. Templates
        ↓
3. Data binding
        ↓
4. Inputs / Outputs
        ↓
5. Services
        ↓
6. Dependency Injection
        ↓
7. Routing
        ↓
8. HttpClient
        ↓
9. RxJS
        ↓
10. Signals
        ↓
11. Forms
        ↓
12. Authentication
```

Une fois ces bases comprises, les concepts plus avancés deviennent beaucoup plus faciles.

---

# 24. Correspondances Angular / .NET

Comme tu viens principalement de .NET, certaines analogies peuvent aider.

| Angular | .NET / ASP.NET Core |
|---|---|
| Component | Classe orientée UI |
| Service | Service applicatif |
| Dependency Injection | Dependency Injection |
| HttpClient | HttpClient |
| Observable | Abstraction de flux asynchrone |
| Router | Routing ASP.NET Core |
| Guard | Logique de contrôle d'accès/navigation |
| Interceptor | Middleware / pipeline conceptuellement |
| Signal | État réactif |
| Template | Vue / interface |
| TypeScript | Langage côté frontend |
| ASP.NET Core API | Backend |
| EF Core | Accès aux données |

Attention : ces correspondances servent uniquement à construire une intuition. Les mécanismes internes ne sont pas identiques.

---

# 25. Questions d'entretien

### Quelle est la différence entre un component et un service ?

> Un component gère principalement une partie de l'interface utilisateur et son comportement. Un service permet de centraliser une logique réutilisable, par exemple des appels HTTP ou certaines opérations métier côté frontend.

### À quoi sert un Input ?

> À transmettre une donnée d'un component parent vers un component enfant.

### À quoi sert un Output ?

> À permettre à un component enfant d'émettre un événement que le parent peut écouter.

### À quoi sert un Signal ?

> À représenter un état réactif qu'Angular peut suivre afin de réagir aux changements de valeur.

### À quoi sert `computed()` ?

> À représenter une valeur dérivée calculée à partir d'autres Signals.

### Quel est le rôle de HttpClient ?

> À effectuer des communications HTTP avec des services externes, notamment une API ASP.NET Core.

### Pourquoi utiliser des services ?

> Pour séparer la logique de l'interface et éviter de concentrer toute la responsabilité dans les components.

---

# À retenir

```text
Component
→ interface + comportement UI

Service
→ logique réutilisable / communication

DI
→ fournit les dépendances

Input
→ parent → enfant

Output
→ enfant → parent

Signal
→ état réactif

Computed
→ état dérivé

Router
→ URL → component

HttpClient
→ Angular → API

RxJS
→ gestion des flux asynchrones
```

## Phrase à mémoriser

> **Angular construit l'interface avec des components, centralise les services avec la Dependency Injection, gère l'état avec des mécanismes réactifs comme les Signals, et communique avec mon API ASP.NET Core via HTTP.**
