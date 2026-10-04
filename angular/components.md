# Angular — Components

Le **Component** est l'unité fondamentale d'une application Angular.

Si tu comprends bien les Components, tu comprends une grande partie du fonctionnement d'Angular.

Un Component associe principalement :

```text
Classe TypeScript
       +
Template HTML
       +
Styles CSS
```

On peut le voir comme :

```text
Component
│
├── état
├── logique
├── template
└── style
```

---

# 1. Qu'est-ce qu'un Component ?

Exemple moderne Angular :

```typescript
import { Component } from '@angular/core';

@Component({
  selector: 'app-user',
  templateUrl: './user.component.html',
  styleUrl: './user.component.css'
})
export class UserComponent {

  name = 'Ahlame';

}
```

Le décorateur :

```typescript
@Component(...)
```

indique à Angular :

> Cette classe est un Component Angular.

Sans `@Component`, la classe est simplement une classe TypeScript normale.

---

# 2. Le décorateur `@Component`

`@Component` reçoit une configuration.

Exemple :

```typescript
@Component({
  selector: 'app-user',
  templateUrl: './user.component.html',
  styleUrl: './user.component.css'
})
```

Les propriétés les plus importantes sont notamment :

```text
selector
template
templateUrl
styles
styleUrl
imports
providers
```

Toutes ne sont pas obligatoires.

---

# 3. `selector`

Exemple :

```typescript
@Component({
  selector: 'app-user'
})
```

Le selector définit comment le component peut être utilisé dans le HTML.

Par exemple :

```html
<app-user></app-user>
```

Angular reconnaît alors :

```text
<app-user>
     ↓
UserComponent
```

Mental model :

```text
selector
   ↓
nom HTML du component
```

---

# 4. Pourquoi `app-` ?

On voit souvent :

```text
app-user
app-header
app-home
app-products
```

Le préfixe `app-` est une convention courante.

Il permet notamment d'éviter les collisions avec les éléments HTML existants.

Par exemple :

```html
<app-user>
```

est clairement un élément appartenant à l'application.

---

# 5. `template`

On peut écrire le HTML directement dans le component.

```typescript
@Component({
  selector: 'app-user',
  template: `
    <h1>{{ name }}</h1>
  `
})
export class UserComponent {

  name = 'Ahlame';

}
```

Ici le template est directement dans le fichier TypeScript.

---

# 6. `templateUrl`

Pour les templates plus importants, on utilise généralement un fichier HTML séparé.

```typescript
@Component({
  selector: 'app-user',
  templateUrl: './user.component.html'
})
```

Puis :

```html
<h1>{{ name }}</h1>
```

Organisation :

```text
user/
│
├── user.component.ts
├── user.component.html
└── user.component.css
```

Cela permet de séparer :

```text
logique
HTML
style
```

---

# 7. `styleUrl`

On peut associer un fichier CSS :

```typescript
@Component({
  styleUrl: './user.component.css'
})
```

Exemple :

```css
h1 {
  font-size: 24px;
}
```

Angular associe alors ce style au component.

---

# 8. Standalone Components

Les versions modernes d'Angular utilisent largement les **standalone components**.

Exemple :

```typescript
@Component({
  selector: 'app-user',
  standalone: true,
  imports: [],
  templateUrl: './user.component.html'
})
export class UserComponent {
}
```

Le principe est de permettre au component de déclarer directement les dépendances dont il a besoin.

Exemple :

```typescript
@Component({
  selector: 'app-user',
  standalone: true,
  imports: [CommonModule],
  templateUrl: './user.component.html'
})
```

Mental model :

```text
Component
   ↓
déclare ses imports
   ↓
peut utiliser les fonctionnalités nécessaires
```

---

# 9. Pourquoi les standalone components ?

Historiquement Angular utilisait beaucoup les `NgModule`.

On avait par exemple :

```text
AppModule
   ↓
declarations
   ↓
components
```

Avec les standalone components, la dépendance peut être déclarée directement au niveau du component.

Cela simplifie notamment :

```text
structure
configuration
lazy loading
compréhension des dépendances
```

Il est important de savoir lire les deux approches car les projets Angular existants peuvent encore utiliser les modules.

---

# 10. Classe du Component

Exemple :

```typescript
export class UserComponent {

  name = 'Ahlame';

  getName() {
    return this.name;
  }

}
```

La classe contient l'état et le comportement utilisés par le template.

Template :

```html
<h1>{{ name }}</h1>

<button (click)="getName()">
  Get name
</button>
```

Le template peut accéder aux membres exposés par le component.

---

# 11. Le flux Component → Template

Prenons :

```typescript
export class UserComponent {

  name = 'Ahlame';

}
```

Et :

```html
<h1>{{ name }}</h1>
```

Le flux conceptuel est :

```text
UserComponent
     │
     │ name
     ↓
Template
     │
     ↓
DOM
```

Angular maintient la relation entre l'état du component et l'interface.

---

# 12. Interpolation dans un Component

```typescript
name = 'Ahlame';
age = 42;
```

Template :

```html
<p>{{ name }}</p>
<p>{{ age }}</p>
```

Angular évalue les expressions.

On peut aussi écrire :

```html
<p>{{ name.toUpperCase() }}</p>
```

Mais il faut éviter de mettre des traitements lourds dans le template.

Une interface doit rester simple à lire et à maintenir.

---

# 13. Event binding

Le Component peut recevoir des événements du template.

```html
<button (click)="increment()">
  Increment
</button>
```

Component :

```typescript
count = 0;

increment() {
  this.count++;
}
```

Flux :

```text
Utilisateur
    ↓
click
    ↓
increment()
    ↓
count change
    ↓
Angular met à jour l'interface
```

---

# 14. Property binding

Exemple :

```typescript
imageUrl = '/images/hotel.jpg';
```

Template :

```html
<img [src]="imageUrl">
```

Le `[]` signifie que l'on fait un binding vers une propriété.

Autre exemple :

```html
<button [disabled]="isSaving">
  Save
</button>
```

Si :

```typescript
isSaving = true;
```

le bouton devient désactivé.

---

# 15. Attribute binding vs Property binding

Il faut distinguer :

```html
[attr.aria-label]="label"
```

et :

```html
[disabled]="isDisabled"
```

Le property binding travaille avec une propriété de l'objet DOM ou du composant.

L'attribute binding manipule un attribut HTML.

Mental model :

```text
[property]
    ↓
propriété

[attr.attribute]
    ↓
attribut
```

Cette distinction devient importante notamment avec les attributs ARIA et certains attributs HTML.

---

# 16. Inputs : parent vers enfant

Supposons un component enfant :

```typescript
@Component({
  selector: 'app-user-card',
  template: `
    <h2>{{ user.name }}</h2>
  `
})
export class UserCardComponent {

  @Input()
  user!: User;

}
```

Le parent :

```html
<app-user-card
  [user]="selectedUser">
</app-user-card>
```

Flux :

```text
Parent
  │
  │ selectedUser
  ↓
[ user ]
  ↓
Child
```

Donc :

> Input = données reçues par le component.

---

# 17. Input avec la nouvelle API

Angular moderne permet également une API basée sur `input()`.

Exemple :

```typescript
user = input.required<User>();
```

Le parent peut toujours transmettre :

```html
<app-user-card
  [user]="selectedUser">
</app-user-card>
```

Dans le component, la valeur est lue comme un Signal :

```typescript
this.user()
```

Mental model :

```text
@Input()
→ ancienne API / API classique

input()
→ nouvelle API réactive
```

Il est utile de connaître les deux syntaxes.

---

# 18. Outputs : enfant vers parent

Un enfant peut émettre un événement.

API classique :

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
  (deleted)="onUserDeleted($event)">
</app-user-card>
```

Flux :

```text
Child
  │
  │ emit(id)
  ↓
Output
  ↓
Parent
  │
  ↓
onUserDeleted(id)
```

---

# 19. Pourquoi utiliser Input/Output ?

Ils permettent de faire communiquer les components sans les rendre fortement dépendants les uns des autres.

Exemple :

```text
Parent
   │
   ├── UserCard
   ├── UserCard
   └── UserCard
```

Le parent possède les données.

Chaque enfant reçoit uniquement ce dont il a besoin.

Cela favorise :

```text
réutilisabilité
testabilité
séparation des responsabilités
```

---

# 20. Services vs Component

Question importante :

> Pourquoi ne pas mettre l'appel HTTP directement dans le Component ?

On pourrait écrire :

```typescript
export class HotelComponent {

  constructor(private http: HttpClient) {}

  loadHotels() {
    return this.http.get('/api/hotels');
  }
}
```

Mais si plusieurs components doivent accéder aux hôtels :

```text
HotelListComponent
HotelDetailsComponent
HotelEditComponent
```

on risque de dupliquer la logique.

On préfère :

```text
HotelComponent
      ↓
HotelService
      ↓
HttpClient
      ↓
API
```

---

# 21. Cycle de vie d'un Component

Un Component possède un cycle de vie.

Quelques hooks importants :

```text
constructor
   ↓
ngOnChanges
   ↓
ngOnInit
   ↓
ngAfterContentInit
   ↓
ngAfterViewInit
   ↓
ngOnDestroy
```

Tous les hooks ne sont pas toujours utilisés.

---

# 22. `constructor`

Le constructeur est principalement utilisé pour l'initialisation de la classe et, historiquement, l'injection des dépendances.

Exemple :

```typescript
constructor(
  private hotelService: HotelService
) {}
```

Le constructeur ne devrait pas devenir un endroit où l'on met toute la logique d'initialisation.

---

# 23. `ngOnInit`

`ngOnInit` est exécuté pendant l'initialisation du component.

Exemple :

```typescript
export class HotelListComponent implements OnInit {

  ngOnInit() {
    this.loadHotels();
  }

}
```

On peut y effectuer une initialisation dépendant du component.

---

# 24. `ngOnChanges`

`ngOnChanges` est utile lorsqu'un component reçoit des Inputs et qu'on veut réagir à leurs changements.

Exemple conceptuel :

```text
Parent
   ↓
Input change
   ↓
ngOnChanges
```

C'est particulièrement important lorsqu'un enfant doit recalculer quelque chose à partir d'une valeur reçue du parent.

---

# 25. `ngOnDestroy`

`ngOnDestroy` est exécuté avant la destruction du component.

Il peut servir à effectuer du nettoyage.

Par exemple :

```text
subscriptions
timers
listeners
ressources
```

Dans Angular moderne, certaines opérations peuvent être gérées avec des mécanismes comme `DestroyRef` et `takeUntilDestroyed`.

L'objectif reste le même :

> ne pas laisser des ressources continuer à vivre après la destruction du component.

---

# 26. Parent / enfant : mental model

Imagine :

```text
HotelPageComponent
        │
        ├── HotelListComponent
        │       │
        │       ├── HotelCardComponent
        │       ├── HotelCardComponent
        │       └── HotelCardComponent
        │
        └── FilterComponent
```

Les données peuvent descendre :

```text
HotelPage
   ↓
HotelList
   ↓
HotelCard
```

Les événements peuvent remonter :

```text
HotelCard
   ↑
HotelList
   ↑
HotelPage
```

Donc :

```text
DATA
↓
↓
↓

EVENT
↑
↑
↑
```

C'est une règle mentale extrêmement utile.

---

# 27. Quand utiliser un Service plutôt qu'un Output ?

Si la communication concerne simplement :

```text
parent ↔ enfant
```

`Input` / `Output` est souvent adapté.

Mais si beaucoup de components éloignés doivent partager un état :

```text
Component A
     │
     ├──────────┐
     │          │
Component B   Component C
     │          │
     └──────────┘
          ↓
       Service
```

un service ou une solution de gestion d'état peut être plus approprié.

Évite de créer une chaîne artificielle :

```text
A → B → C → D
```

uniquement pour transmettre une donnée.

---

# 28. Component et Signal

Exemple :

```typescript
count = signal(0);
```

Template :

```html
<p>{{ count() }}</p>

<button (click)="count.update(v => v + 1)">
  +
</button>
```

Le flux :

```text
count
  ↓
Signal
  ↓
Template lit count()
  ↓
Utilisateur clique
  ↓
Signal update
  ↓
Angular sait que l'état a changé
  ↓
UI mise à jour
```

Le point important est que le Signal représente un état réactif.

---

# 29. `computed` dans un Component

Exemple :

```typescript
firstName = signal('Ahlame');
lastName = signal('Mohsine');

fullName = computed(() =>
  `${this.firstName()} ${this.lastName()}`
);
```

Template :

```html
<p>{{ fullName() }}</p>
```

Ici :

```text
firstName
       \
        → fullName
       /
lastName
```

`fullName` est une donnée dérivée.

Il n'est pas nécessaire de maintenir manuellement :

```typescript
fullName = ...
```

à chaque modification.

---

# 30. Ce qu'il faut éviter dans un Component

Un component qui contient :

```text
1000 lignes
```

avec :

```text
HTTP
SQL
business logic
validation
transformation
état
UI
logging
```

est généralement difficile à maintenir.

On cherche plutôt :

```text
Component
    ↓
Service
    ↓
API
```

avec des responsabilités bien séparées.

---

# 31. Exemple de Component propre

```typescript
@Component({
  selector: 'app-hotel-list',
  standalone: true,
  templateUrl: './hotel-list.component.html'
})
export class HotelListComponent {

  hotels = signal<Hotel[]>([]);
  isLoading = signal(false);

  constructor(
    private hotelService: HotelService
  ) {}

  loadHotels() {
    this.isLoading.set(true);

    this.hotelService.getHotels()
      .subscribe({
        next: hotels => {
          this.hotels.set(hotels);
          this.isLoading.set(false);
        },
        error: () => {
          this.isLoading.set(false);
        }
      });
  }
}
```

Le component orchestre l'interface.

Le service s'occupe de la communication avec l'API.

---

# 32. Analogie avec ASP.NET Core

En ASP.NET Core :

```text
Controller
    ↓
Service
    ↓
Repository / EF Core
```

En Angular :

```text
Component
    ↓
Service
    ↓
HttpClient
```

Attention : ce ne sont pas exactement les mêmes responsabilités.

Mais pour construire une première intuition :

```text
Component ≈ couche UI

Service ≈ logique réutilisable / communication

HttpClient ≈ communication externe
```

---

# 33. Questions d'entretien

### Qu'est-ce qu'un Component Angular ?

> C'est une unité d'interface composée d'une classe TypeScript, d'un template et éventuellement de styles. Le décorateur `@Component` fournit les métadonnées nécessaires à Angular.

### À quoi sert `selector` ?

> Il définit le sélecteur permettant d'utiliser le component dans un template.

### Quelle différence entre Input et Output ?

> Input transmet des données du parent vers l'enfant. Output permet à l'enfant d'émettre un événement vers le parent.

### Pourquoi utiliser un service ?

> Pour centraliser une logique réutilisable et séparer la logique applicative ou les appels HTTP du code d'interface.

### À quoi sert `ngOnInit` ?

> À effectuer certaines opérations d'initialisation du component après sa création et l'initialisation de ses propriétés liées au cycle Angular.

### Pourquoi éviter de mettre toute la logique dans le component ?

> Parce qu'un component trop chargé devient difficile à tester, maintenir et réutiliser. La logique peut être déplacée dans des services ou d'autres abstractions adaptées.

---

# À retenir

```text
@Component
    ↓
déclare un Component

selector
    ↓
nom utilisé dans le HTML

template
    ↓
interface

class
    ↓
état + comportement

Input
    ↓
parent → enfant

Output
    ↓
enfant → parent

Service
    ↓
logique réutilisable / communication

Signal
    ↓
état réactif

computed
    ↓
valeur dérivée

ngOnInit
    ↓
initialisation

ngOnDestroy
    ↓
nettoyage
```

## Phrase à mémoriser

> **Un Component Angular représente une partie de l'interface : sa classe contient l'état et le comportement, son template décrit l'affichage, les Inputs font descendre les données, les Outputs font remonter les événements, et les Services permettent de sortir la logique réutilisable du Component.**
