# Angular — Signals

Les **Signals** sont un mécanisme de gestion d'état réactif d'Angular.

L'idée fondamentale est simple :

```text
État
 ↓
Signal
 ↓
Angular sait qui dépend de cet état
 ↓
Mise à jour ciblée
```

Un Signal permet donc de représenter une valeur que l'application doit pouvoir observer lorsqu'elle change.

---

# 1. Pourquoi les Signals ?

Prenons un état classique :

```typescript
count = 0;
```

Puis :

```typescript
this.count++;
```

Avec une simple variable, Angular ne possède pas la même information explicite sur les dépendances réactives.

Avec un Signal :

```typescript
count = signal(0);
```

Angular dispose d'une abstraction réactive :

```text
count
 ↓
Signal
 ↓
dépendances connues
 ↓
réaction aux changements
```

Le Signal devient donc une **source d'état réactif**.

---

# 2. Créer un Signal

Il faut importer `signal` :

```typescript
import { signal } from '@angular/core';
```

Puis :

```typescript
count = signal(0);
```

Le type conceptuel est :

```text
WritableSignal<number>
```

Le Signal contient :

```text
valeur actuelle
+
mécanisme de notification des dépendances
```

---

# 3. Lire un Signal

Une erreur fréquente est d'écrire :

```typescript
count
```

pour obtenir la valeur.

Avec un Signal, on lit la valeur en l'appelant :

```typescript
count()
```

Exemple :

```typescript
console.log(this.count());
```

Mental model :

```text
count
  ↓
le Signal lui-même

count()
  ↓
la valeur contenue dans le Signal
```

C'est une distinction fondamentale.

---

# 4. Modifier avec `set()`

Pour remplacer complètement la valeur :

```typescript
this.count.set(10);
```

Avant :

```text
count = 0
```

Après :

```text
count = 10
```

`set()` signifie donc :

> Remplace la valeur actuelle par cette nouvelle valeur.

---

# 5. Modifier avec `update()`

Si la nouvelle valeur dépend de l'ancienne :

```typescript
this.count.update(value => value + 1);
```

Avant :

```text
10
```

Après :

```text
11
```

Mental model :

```text
set()
→ nouvelle valeur connue directement

update()
→ nouvelle valeur calculée à partir de l'ancienne
```

Exemple :

```typescript
this.count.set(100);

this.count.update(value => value + 1);
```

Résultat :

```text
101
```

---

# 6. Exemple complet

```typescript
import { Component, signal } from '@angular/core';

@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <p>Count : {{ count() }}</p>

    <button (click)="increment()">
      +
    </button>
  `
})
export class CounterComponent {

  count = signal(0);

  increment() {
    this.count.update(value => value + 1);
  }
}
```

Le fonctionnement :

```text
Initialisation
    ↓
count = 0
    ↓
Template lit count()
    ↓
Utilisateur clique
    ↓
increment()
    ↓
count.update(...)
    ↓
Signal change
    ↓
Angular sait que count a changé
    ↓
Template réévalué
```

---

# 7. Signal et template

Si le template contient :

```html
<p>{{ count() }}</p>
```

le template **lit** le Signal.

Angular peut alors enregistrer cette dépendance.

Mental model :

```text
Template
    │
    │ lit
    ↓
count()
    │
    ↓
Signal
```

Lorsque le Signal change :

```text
count.set(...)
       ↓
Signal change
       ↓
Angular sait que le template dépend de count
       ↓
mise à jour nécessaire
```

C'est l'un des intérêts majeurs du système réactif.

---

# 8. Signal avec un objet

Un Signal peut contenir un objet.

```typescript
user = signal({
  name: 'Ahlame',
  age: 42
});
```

Lecture :

```typescript
console.log(this.user());
```

Résultat :

```typescript
{
  name: 'Ahlame',
  age: 42
}
```

---

# 9. Modifier un objet avec `set`

On peut remplacer tout l'objet :

```typescript
this.user.set({
  name: 'Ahlame',
  age: 43
});
```

Le Signal contient maintenant une nouvelle valeur.

---

# 10. Modifier un objet avec `update`

On peut également partir de la valeur précédente :

```typescript
this.user.update(user => ({
  ...user,
  age: user.age + 1
}));
```

On utilise ici le spread :

```typescript
...user
```

pour conserver les autres propriétés.

Mental model :

```text
ancienne valeur
      ↓
update()
      ↓
nouvel objet
      ↓
nouvelle valeur du Signal
```

---

# 11. Signal avec un tableau

Exemple :

```typescript
users = signal<User[]>([]);
```

Ajouter un élément :

```typescript
this.users.update(users => [
  ...users,
  newUser
]);
```

Supprimer :

```typescript
this.users.update(users =>
  users.filter(user => user.id !== id)
);
```

Le Signal contient toujours :

```text
User[]
```

mais Angular peut réagir au changement de valeur.

---

# 12. Ne pas modifier directement la valeur interne

Évite de faire :

```typescript
this.users().push(newUser);
```

Même si cela modifie le tableau JavaScript, tu ne demandes pas explicitement au Signal de changer.

Préférer :

```typescript
this.users.update(users => [
  ...users,
  newUser
]);
```

ou :

```typescript
this.users.set([
  ...this.users(),
  newUser
]);
```

Le principe important est :

> Modifie l'état à travers l'API du Signal.

---

# 13. Writable Signal

Un Signal créé avec :

```typescript
signal(...)
```

est modifiable.

On parle de :

```text
Writable Signal
```

Il expose notamment :

```typescript
set()
update()
```

Exemple :

```typescript
count = signal(0);

count.set(10);

count.update(value => value + 1);
```

---

# 14. Readonly Signal

Parfois, un component ou un service doit pouvoir **lire** un état sans avoir le droit de le modifier.

On peut exposer une interface en lecture seule.

L'idée est :

```text
interne
   ↓
Writable Signal

extérieur
   ↓
Readonly Signal
```

Cela permet d'encapsuler la modification de l'état.

Mental model :

```text
Le propriétaire du state
→ peut écrire

Les consommateurs
→ peuvent lire
```

C'est un principe important pour éviter que n'importe quelle partie de l'application puisse modifier directement un état partagé.

---

# 15. `computed()`

`computed()` permet de créer une valeur dérivée.

Exemple :

```typescript
firstName = signal('Ahlame');
lastName = signal('Mohsine');

fullName = computed(() =>
  `${this.firstName()} ${this.lastName()}`
);
```

Lecture :

```typescript
this.fullName();
```

Le résultat :

```text
Ahlame Mohsine
```

Mental model :

```text
firstName ──┐
            ├──→ fullName
lastName ───┘
```

---

# 16. `computed` n'est pas un nouveau state source

C'est une distinction essentielle.

Avec :

```typescript
count = signal(10);
```

`count` est une source d'état.

Avec :

```typescript
double = computed(() => this.count() * 2);
```

`double` est une valeur dérivée.

```text
count
 ↓
double
```

On ne doit donc pas considérer `double` comme une deuxième source de vérité.

---

# 17. Pourquoi `computed` est utile ?

Supposons :

```typescript
users = signal<User[]>([]);
```

On veut uniquement les utilisateurs actifs :

```typescript
activeUsers = computed(() =>
  this.users().filter(user => user.active)
);
```

On obtient :

```text
users
 ↓
computed
 ↓
activeUsers
```

Si `users` change, Angular sait que `activeUsers` dépend de `users`.

On n'a donc pas besoin de faire manuellement :

```typescript
this.activeUsers = ...
```

à chaque changement.

---

# 18. `computed` est en lecture seule

Un `computed` ne se modifie pas comme un Writable Signal.

On ne fait pas :

```typescript
this.activeUsers.set(...);
```

La valeur est calculée par :

```typescript
computed(() => ...)
```

La source doit être modifiée :

```typescript
this.users.update(...);
```

Puis la valeur dérivée est recalculée lorsque nécessaire.

Mental model :

```text
Source
 ↓
computed
 ↓
résultat
```

On modifie la source, pas le résultat calculé.

---

# 19. Dépendances dynamiques

Angular peut déterminer quelles valeurs sont réellement lues dans un `computed`.

Exemple :

```typescript
showDetails = signal(false);

name = signal('Ahlame');
age = signal(42);

details = computed(() => {
  if (!this.showDetails()) {
    return this.name();
  }

  return `${this.name()} - ${this.age()}`;
});
```

Le `computed` lit toujours :

```text
showDetails
name
```

Mais `age` n'est lu que lorsque la condition correspondante est exécutée.

Cela permet au système réactif de suivre les dépendances réellement utilisées.

---

# 20. `effect()`

`effect()` sert à exécuter un effet secondaire lorsqu'un Signal utilisé par cet effet change.

Exemple :

```typescript
effect(() => {
  console.log(this.count());
});
```

Le code lit :

```typescript
this.count()
```

Angular sait donc que l'effet dépend de `count`.

Si :

```typescript
this.count.set(10);
```

l'effet peut être exécuté à nouveau.

Mental model :

```text
Signal
   ↓
effect
   ↓
action secondaire
```

---

# 21. `computed` vs `effect`

Cette distinction est extrêmement importante.

### `computed`

Utilise-le lorsqu'on veut :

```text
calculer une valeur
```

Exemple :

```typescript
fullName = computed(() =>
  `${this.firstName()} ${this.lastName()}`
);
```

### `effect`

Utilise-le lorsqu'on veut :

```text
faire quelque chose à cause d'un changement
```

Exemple :

```typescript
effect(() => {
  console.log(this.count());
});
```

Mental model :

```text
computed
= "Quelle est la nouvelle valeur ?"

effect
= "Quelle action dois-je effectuer ?"
```

---

# 22. Mauvaise utilisation de `effect`

Supposons :

```typescript
count = signal(10);

double = signal(0);

effect(() => {
  this.double.set(this.count() * 2);
});
```

Cela fonctionne comme logique, mais on crée inutilement un deuxième état.

On préfère :

```typescript
count = signal(10);

double = computed(() =>
  this.count() * 2
);
```

Pourquoi ?

Parce que :

```text
double
```

est une valeur dérivée.

Il n'est pas nécessaire d'en faire une source de vérité indépendante.

---

# 23. Deux sources de vérité

Mauvaise architecture :

```typescript
price = signal(100);

priceWithTax = signal(121);
```

Il faut maintenant maintenir :

```text
price
+
priceWithTax
```

synchronisés.

Si :

```typescript
price.set(200);
```

il faut penser à modifier également :

```typescript
priceWithTax
```

On crée un risque d'incohérence.

Préférer :

```typescript
price = signal(100);

priceWithTax = computed(() =>
  this.price() * 1.21
);
```

Une seule source de vérité :

```text
price
 ↓
priceWithTax
```

---

# 24. Signals et `OnPush`

Les Signals fonctionnent particulièrement bien avec le modèle moderne de détection des changements d'Angular.

Lorsqu'un template lit un Signal :

```html
{{ count() }}
```

Angular sait que cette partie de l'interface dépend de ce Signal.

Lorsque le Signal change :

```typescript
count.set(10);
```

Angular dispose de cette information réactive.

Cela permet une approche plus précise de la gestion de l'état et des mises à jour de l'interface.

---

# 25. Signal dans un service

Les Signals ne sont pas limités aux components.

Exemple :

```typescript
@Injectable({
  providedIn: 'root'
})
export class CartService {

  private items = signal<CartItem[]>([]);

  readonly cartItems = this.items.asReadonly();

  add(item: CartItem) {
    this.items.update(items => [
      ...items,
      item
    ]);
  }
}
```

Ici :

```text
Service
│
├── items
│    └── Writable Signal privé
│
└── cartItems
     └── Readonly Signal public
```

Le service contrôle donc les modifications.

---

# 26. Pourquoi `asReadonly()` ?

Supposons que le service expose directement :

```typescript
items = signal<CartItem[]>([]);
```

Un autre objet pourrait potentiellement avoir accès à l'API d'écriture.

Avec :

```typescript
private items = signal<CartItem[]>([]);

readonly cartItems = this.items.asReadonly();
```

le service garde la responsabilité de modifier l'état.

Mental model :

```text
Service
  │
  ├── écrit
  │
  └── expose une vue en lecture
```

Cela améliore l'encapsulation.

---

# 27. Signal et API HTTP

Exemple :

```typescript
hotels = signal<Hotel[]>([]);
loading = signal(false);
error = signal<string | null>(null);
```

Lors du chargement :

```typescript
loadHotels() {

  this.loading.set(true);
  this.error.set(null);

  this.hotelService.getHotels()
    .subscribe({
      next: hotels => {
        this.hotels.set(hotels);
        this.loading.set(false);
      },

      error: () => {
        this.error.set('Unable to load hotels');
        this.loading.set(false);
      }
    });
}
```

Le template peut alors utiliser :

```html
@if (loading()) {
  <p>Loading...</p>
}

@if (error(); as message) {
  <p>{{ message }}</p>
}

@for (hotel of hotels(); track hotel.id) {
  <p>{{ hotel.name }}</p>
}
```

On obtient trois états réactifs :

```text
loading
hotels
error
```

---

# 28. State local vs state partagé

Tous les états n'ont pas besoin d'être dans un service global.

### État local

Exemple :

```typescript
isMenuOpen = signal(false);
```

Si seul le component utilise cette information :

```text
Component
   ↓
Signal local
```

suffit.

### État partagé

Si plusieurs components doivent utiliser le même état :

```text
Component A
       \
        Service
       /
Component B
```

un service avec Signals peut être pertinent.

---

# 29. Signal vs Observable

Il ne faut pas penser :

```text
Signal = remplacement exact d'Observable
```

Ils ont des rôles différents.

### Signal

Très pratique pour :

```text
état actuel
valeur actuelle
état UI
valeurs dérivées
réactivité locale
```

### Observable

Très adapté à :

```text
flux asynchrones
HTTP
événements
streams
opérations RxJS
```

Exemple :

```text
HttpClient
   ↓
Observable
   ↓
Service
   ↓
Signal
   ↓
Component
```

Une application Angular peut parfaitement utiliser les deux.

---

# 30. Signal vs Promise

Une Promise représente principalement :

```text
une opération asynchrone
→ une valeur future
```

Un Signal représente :

```text
un état réactif actuel
→ qui peut évoluer
```

Exemple mental :

```text
Promise
"Donne-moi le résultat lorsqu'il sera disponible."

Signal
"Voici l'état actuel et préviens les consommateurs lorsqu'il évolue."
```

---

# 31. Piège : oublier les parenthèses

Avec :

```typescript
count = signal(10);
```

Mauvais :

```html
<p>{{ count }}</p>
```

Correct :

```html
<p>{{ count() }}</p>
```

Même chose en TypeScript :

```typescript
console.log(this.count());
```

et non :

```typescript
console.log(this.count);
```

---

# 32. Piège : utiliser `set` alors qu'on veut calculer

Si on a :

```typescript
count = signal(10);
```

On peut faire :

```typescript
count.set(20);
```

Mais si on veut :

```text
ancienne valeur + 1
```

préférer :

```typescript
count.update(value => value + 1);
```

Mental model :

```text
set(value)
→ "Voici la nouvelle valeur."

update(value => ...)
→ "Calcule la nouvelle valeur à partir de l'ancienne."
```

---

# 33. Piège : mettre une valeur dérivée dans un Signal mutable

Éviter :

```typescript
users = signal<User[]>([]);
activeUsers = signal<User[]>([]);
```

si `activeUsers` est uniquement calculé depuis `users`.

Préférer :

```typescript
users = signal<User[]>([]);

activeUsers = computed(() =>
  this.users().filter(user => user.active)
);
```

Cela réduit les risques d'état incohérent.

---

# 34. Piège : utiliser `effect` pour calculer

Éviter :

```typescript
effect(() => {
  this.total.set(
    this.price() * this.quantity()
  );
});
```

si `total` est uniquement une valeur dérivée.

Préférer :

```typescript
total = computed(() =>
  this.price() * this.quantity()
);
```

---

# 35. Piège : trop de Signals

Les Signals sont utiles, mais il ne faut pas transformer chaque variable en Signal sans raison.

Question à se poser :

> Cette valeur doit-elle être réactive et observée par l'interface ou d'autres dépendances ?

Si non, une variable normale peut suffire.

Exemple :

```typescript
const taxRate = 0.21;
```

n'a pas forcément besoin d'être :

```typescript
taxRate = signal(0.21);
```

---

# 36. Architecture mentale

Imagine une application :

```text
                USER ACTION
                     │
                     ↓
              Component method
                     │
                     ↓
                  Signal
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
      Template              computed
          │                     │
          ↓                     ↓
         UI                  valeur dérivée
                                │
                                ↓
                              effect
                                │
                                ↓
                         action secondaire
```

Cette représentation permet de comprendre le rôle de chaque mécanisme.

---

# 37. Exemple complet

```typescript
import {
  Component,
  computed,
  effect,
  signal
} from '@angular/core';

@Component({
  selector: 'app-cart',
  standalone: true,
  template: `
    <p>Items : {{ items().length }}</p>
    <p>Total : {{ total() }}</p>

    <button (click)="addItem()">
      Add item
    </button>
  `
})
export class CartComponent {

  items = signal<number[]>([]);

  total = computed(() =>
    this.items().reduce(
      (sum, price) => sum + price,
      0
    )
  );

  constructor() {
    effect(() => {
      console.log('Cart total:', this.total());
    });
  }

  addItem() {
    this.items.update(items => [
      ...items,
      10
    ]);
  }
}
```

Flux :

```text
items
 ↓
computed(total)
 ↓
template

items
 ↓
computed(total)
 ↓
effect
```

Lorsqu'on clique :

```text
click
 ↓
addItem()
 ↓
items.update()
 ↓
items change
 ↓
total dépend de items
 ↓
total change
 ↓
template peut être mis à jour
 ↓
effect peut réagir
```

---

# 38. Questions d'entretien

### Qu'est-ce qu'un Signal ?

> Un Signal est une abstraction réactive qui représente une valeur et permet à Angular de suivre les dépendances qui la lisent afin de réagir lorsqu'elle change.

### Comment lit-on un Signal ?

```typescript
count()
```

### Comment modifie-t-on un Writable Signal ?

```typescript
count.set(10);
```

ou :

```typescript
count.update(value => value + 1);
```

### Quelle différence entre `set()` et `update()` ?

> `set()` remplace directement la valeur. `update()` calcule une nouvelle valeur à partir de la valeur actuelle.

### À quoi sert `computed()` ?

> À créer une valeur dérivée à partir d'autres Signals.

### À quoi sert `effect()` ?

> À exécuter un effet secondaire en réaction aux Signals lus dans l'effet.

### Pourquoi préférer `computed()` à `effect()` pour une valeur dérivée ?

> Parce qu'une valeur dérivée doit rester une conséquence des données sources et ne doit pas devenir une deuxième source de vérité mutable.

### Signal ou Observable ?

> Un Signal est particulièrement adapté à l'état réactif actuel et aux valeurs dérivées. Un Observable est particulièrement adapté aux flux asynchrones et à l'écosystème RxJS. Les deux peuvent être utilisés ensemble.

---

# À retenir

```text
signal()
→ crée un état réactif modifiable

signal()
→ lire la valeur

set()
→ remplacer la valeur

update()
→ calculer une nouvelle valeur à partir de l'ancienne

computed()
→ valeur dérivée

effect()
→ effet secondaire

asReadonly()
→ exposer un Signal sans exposer son écriture

Signal
→ état actuel réactif

Observable
→ flux de valeurs
```

## Règle d'or

```text
SOURCE DE VÉRITÉ
       ↓
     signal
       ↓
   computed
       ↓
     UI
```

Et si une action secondaire doit être déclenchée :

```text
Signal
  ↓
effect
  ↓
action secondaire
```

## Phrase à mémoriser

> **Un Signal représente mon état réactif, `set()` le remplace, `update()` le transforme, `computed()` dérive une nouvelle valeur sans créer une deuxième source de vérité, et `effect()` déclenche une action secondaire lorsqu'une dépendance change.**
