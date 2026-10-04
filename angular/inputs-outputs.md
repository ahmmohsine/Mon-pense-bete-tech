# Angular — Inputs & Outputs

`Input` et `Output` permettent aux Components Angular de communiquer entre eux.

La règle fondamentale est :

```text
Input
parent → enfant

Output
enfant → parent
```

Mental model :

```text
             DONNÉES
Parent ─────────────────→ Child
        Input

             ÉVÉNEMENT
Parent ←───────────────── Child
        Output
```

C'est l'un des mécanismes essentiels pour construire une application Angular propre.

---

# 1. Pourquoi Input et Output ?

Imagine :

```text
HotelListComponent
        ↓
HotelCardComponent
```

Le parent possède les données :

```typescript
hotels = [...]
```

Le component enfant doit afficher un hôtel précis.

Le parent peut lui transmettre :

```html
<app-hotel-card
  [hotel]="hotel">
</app-hotel-card>
```

C'est un `Input`.

Maintenant l'utilisateur clique sur :

```text
Delete
```

L'enfant doit prévenir le parent.

L'enfant émet :

```text
deleted
```

C'est un `Output`.

---

# 2. Input : parent → enfant

API classique :

```typescript
import { Input } from '@angular/core';

@Input()
hotel!: Hotel;
```

Parent :

```html
<app-hotel-card
  [hotel]="selectedHotel">
</app-hotel-card>
```

Flux :

```text
selectedHotel
     ↓
   [hotel]
     ↓
HotelCardComponent
```

L'enfant reçoit donc la valeur.

---

# 3. Pourquoi les crochets `[ ]` ?

Cette syntaxe :

```html
[hotel]="selectedHotel"
```

signifie :

> Évalue `selectedHotel` dans le contexte Angular et affecte le résultat à l'Input `hotel`.

Ce n'est pas la même chose que :

```html
hotel="selectedHotel"
```

Dans ce deuxième cas, on transmet une valeur littérale :

```text
"selectedHotel"
```

et non la valeur contenue dans la variable.

Mental model :

```text
[hotel]="selectedHotel"
       ↓
valeur de la propriété

hotel="selectedHotel"
       ↓
chaîne littérale
```

---

# 4. Exemple concret

Parent :

```typescript
selectedHotel = {
  id: 1,
  name: 'Hotel Royal'
};
```

Template parent :

```html
<app-hotel-card
  [hotel]="selectedHotel">
</app-hotel-card>
```

Enfant :

```typescript
export class HotelCardComponent {

  @Input()
  hotel!: Hotel;

}
```

Template enfant :

```html
<h2>{{ hotel.name }}</h2>
```

Résultat :

```text
Hotel Royal
```

---

# 5. Input moderne avec `input()`

Angular moderne permet également :

```typescript
import { input } from '@angular/core';

hotel = input<Hotel>();
```

La valeur est alors lue comme un Signal :

```typescript
this.hotel()
```

Dans le template :

```html
<h2>{{ hotel()?.name }}</h2>
```

Ou avec un Input obligatoire :

```typescript
hotel = input.required<Hotel>();
```

On obtient une intention plus explicite :

```text
input()
→ Input réactif

input.required()
→ Input obligatoire
```

---

# 6. Input obligatoire

API moderne :

```typescript
hotel = input.required<Hotel>();
```

Cela signifie que le component attend obligatoirement une valeur.

Parent :

```html
<app-hotel-card
  [hotel]="selectedHotel">
</app-hotel-card>
```

L'idée est d'éviter qu'un component soit utilisé sans une donnée indispensable.

Mental model :

```text
Component
   ↓
"J'ai besoin de Hotel"
   ↓
input.required<Hotel>()
```

---

# 7. Input avec une valeur par défaut

Un Input peut également avoir une valeur initiale.

Exemple avec l'API moderne :

```typescript
pageSize = input(10);
```

Si le parent ne fournit rien :

```text
pageSize = 10
```

Si le parent fournit :

```html
<app-hotel-list
  [pageSize]="20">
</app-hotel-list>
```

alors :

```text
pageSize = 20
```

---

# 8. Alias d'un Input

On peut exposer un nom différent dans le template.

Exemple :

```typescript
hotel = input<Hotel>({
  alias: 'item'
});
```

Le parent utilise :

```html
<app-hotel-card
  [item]="selectedHotel">
</app-hotel-card>
```

Alors que dans le component, la propriété reste :

```typescript
hotel
```

Mental model :

```text
HTML public API
       ↓
     item
       ↓
propriété interne
       ↓
     hotel
```

L'alias peut être utile lorsque le nom public doit être différent du nom interne.

---

# 9. Input et changement de valeur

Un Input n'est pas uniquement une valeur initiale.

Le parent peut modifier la valeur :

```typescript
selectedHotel = signal<Hotel>(...);
```

Puis :

```typescript
selectedHotel.set(otherHotel);
```

Le child reçoit la nouvelle valeur.

Flux :

```text
Parent state
    ↓
Input
    ↓
Child
```

Si le child utilise l'API moderne :

```typescript
hotel = input.required<Hotel>();
```

il peut lire :

```typescript
this.hotel()
```

et les mécanismes réactifs peuvent suivre cette dépendance.

---

# 10. Output : enfant → parent

L'Output sert à transmettre un événement au parent.

API classique :

```typescript
@Output()
deleted = new EventEmitter<number>();
```

Puis :

```typescript
deleteHotel() {
  this.deleted.emit(this.hotel.id);
}
```

Parent :

```html
<app-hotel-card
  (deleted)="deleteHotel($event)">
</app-hotel-card>
```

Flux :

```text
Child
  ↓
emit(id)
  ↓
(deleted)
  ↓
Parent
  ↓
deleteHotel(id)
```

---

# 11. Pourquoi les parenthèses `( )` ?

Cette syntaxe :

```html
(deleted)="deleteHotel($event)"
```

indique un **event binding**.

On dit à Angular :

> Lorsque l'événement `deleted` est émis, exécute cette expression.

Mental model :

```text
(event)="handler(...)"
```

Donc :

```text
Input
[...]
↓
données

Output
(...)
↑
événement
```

---

# 12. `$event`

`$event` représente la donnée envoyée avec l'événement.

Si l'enfant fait :

```typescript
this.deleted.emit(this.hotel.id);
```

alors le parent reçoit :

```html
(deleted)="deleteHotel($event)"
```

et :

```typescript
deleteHotel(id: number) {
  console.log(id);
}
```

Si l'enfant émet :

```text
42
```

alors :

```text
$event = 42
```

---

# 13. Output moderne avec `output()`

Angular moderne propose également :

```typescript
import { output } from '@angular/core';

deleted = output<number>();
```

Puis :

```typescript
this.deleted.emit(this.hotel.id);
```

Le parent écoute toujours :

```html
(deleted)="deleteHotel($event)"
```

Mental model :

```text
@Output()
→ API classique

output()
→ API moderne
```

Les deux sont importants à savoir lire.

---

# 14. Output sans donnée

Un événement n'a pas forcément besoin de transporter une valeur.

Exemple :

```typescript
closed = output<void>();
```

Puis :

```typescript
this.closed.emit();
```

Parent :

```html
<app-dialog
  (closed)="closeDialog()">
</app-dialog>
```

Ici :

```text
event
↓
aucune donnée
```

---

# 15. Output avec un objet

On peut également envoyer un objet.

```typescript
saved = output<Hotel>();
```

Puis :

```typescript
this.saved.emit(this.hotel);
```

Parent :

```html
<app-hotel-form
  (saved)="onHotelSaved($event)">
</app-hotel-form>
```

TypeScript :

```typescript
onHotelSaved(hotel: Hotel) {
  console.log(hotel);
}
```

Flux :

```text
Hotel
 ↓
emit(Hotel)
 ↓
$event
 ↓
parent
```

---

# 16. Input + Output ensemble

C'est un cas extrêmement fréquent.

Exemple :

```text
Parent
   │
   │ [hotel]
   ↓
Child
   │
   │ (saved)
   ↓
Parent
```

Parent :

```html
<app-hotel-form
  [hotel]="selectedHotel"
  (saved)="onSaved($event)">
</app-hotel-form>
```

Le parent :

```text
fournit une donnée
+
écoute un événement
```

L'enfant :

```text
reçoit une donnée
+
émet un événement
```

---

# 17. Exemple complet

Child :

```typescript
@Component({
  selector: 'app-counter',
  standalone: true,
  template: `
    <p>{{ value }}</p>

    <button (click)="increment()">
      +
    </button>
  `
})
export class CounterComponent {

  value = input(0);

  changed = output<number>();

  increment() {
    const newValue = this.value() + 1;

    this.changed.emit(newValue);
  }
}
```

Parent :

```html
<app-counter
  [value]="count()"
  (changed)="onCountChanged($event)">
</app-counter>
```

Parent :

```typescript
count = signal(0);

onCountChanged(value: number) {
  this.count.set(value);
}
```

Flux :

```text
Parent
  │
  │ count()
  ↓
[value]
  ↓
Child
  │
  │ changed.emit(...)
  ↓
(changed)
  ↓
Parent
  │
  ↓
count.set(...)
```

---

# 18. Le piège de la modification directe

Supposons :

```typescript
user = input.required<User>();
```

Il ne faut pas considérer l'Input comme un état appartenant à l'enfant.

Mentalement :

```text
Parent possède la donnée
        ↓
Child la reçoit
```

L'enfant ne devrait pas essayer de devenir le propriétaire de cette donnée.

Si l'enfant veut demander une modification au parent :

```text
Child
  ↓
Output
  ↓
Parent
  ↓
modifie son état
  ↓
nouvelle valeur
  ↓
Input
  ↓
Child
```

C'est une architecture beaucoup plus claire.

---

# 19. Un Input n'est pas un Output

Erreur fréquente :

```typescript
@Input()
deleted = ...
```

Un Input reçoit une donnée.

Un Output émet un événement.

```text
Input
→ recevoir

Output
→ émettre
```

Cette distinction doit devenir automatique.

---

# 20. Communication entre components frères

Supposons :

```text
Parent
├── SearchComponent
└── ListComponent
```

`SearchComponent` et `ListComponent` sont frères.

Il est déconseillé de faire communiquer directement :

```text
SearchComponent → ListComponent
```

On peut faire :

```text
SearchComponent
       ↓
     Output
       ↓
     Parent
       ↓
     Signal / state
       ↓
     Input
       ↓
ListComponent
```

Mental model :

```text
       Parent
      /      \
     ↓        ↓
 Search      List
```

Le parent coordonne les deux enfants.

---

# 21. Quand utiliser un service ?

Si plusieurs components éloignés partagent beaucoup d'état :

```text
Component A
       \
        \
       Service
        /
       /
Component B
```

un service peut devenir plus approprié.

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

Le service devient alors une source d'état partagée.

---

# 22. Input/Output vs Service

Utilise généralement :

```text
Input / Output
```

pour une communication locale dans une hiérarchie parent/enfant.

Utilise plutôt :

```text
Service + état partagé
```

lorsque plusieurs parties éloignées de l'application ont besoin du même état.

Exemple :

```text
Parent
 ├── Child A
 └── Child B
```

Input/Output est naturel.

Mais :

```text
Header
   │
   ├── Cart
   │
   ├── ProductList
   │
   └── Checkout
```

Si tous doivent accéder au même panier, un `CartService` peut être plus adapté.

---

# 23. Input transform

Angular permet également certaines transformations d'Input.

Exemple conceptuel :

```typescript
disabled = input(false, {
  transform: booleanAttribute
});
```

Cela permet notamment de transformer une valeur reçue en booléen.

Ce type de fonctionnalité est utile lorsqu'un component expose une API propre aux templates.

---

# 24. Alias et API publique

Un component peut être considéré comme ayant une petite API publique.

Par exemple :

```text
Inputs
   ↓
ce que le parent peut fournir

Outputs
   ↓
ce que l'enfant peut notifier
```

Exemple :

```typescript
hotel = input.required<Hotel>();

saved = output<Hotel>();

deleted = output<number>();
```

On peut lire cela comme :

```text
API du component :

Entrées :
    Hotel

Sorties :
    Hotel sauvegardé
    id supprimé
```

Cette façon de penser est très utile pour concevoir des components réutilisables.

---

# 25. Component réutilisable

Un bon component peut être utilisé plusieurs fois.

Exemple :

```html
<app-hotel-card
  [hotel]="hotel1">
</app-hotel-card>

<app-hotel-card
  [hotel]="hotel2">
</app-hotel-card>

<app-hotel-card
  [hotel]="hotel3">
</app-hotel-card>
```

Chaque instance possède son propre état local.

Mais la même API est utilisée :

```text
Input → hotel
Output → deleted
```

---

# 26. Exemple d'un component réutilisable

```typescript
@Component({
  selector: 'app-user-card',
  standalone: true,
  template: `
    <article>
      <h2>{{ user().name }}</h2>

      <button (click)="delete()">
        Delete
      </button>
    </article>
  `
})
export class UserCardComponent {

  user = input.required<User>();

  deleted = output<number>();

  delete() {
    this.deleted.emit(this.user().id);
  }
}
```

Parent :

```html
@for (user of users(); track user.id) {

  <app-user-card
    [user]="user"
    (deleted)="deleteUser($event)">
  </app-user-card>

}
```

L'enfant ne sait pas comment l'utilisateur sera réellement supprimé.

Il dit simplement :

```text
"Utilisateur X doit être supprimé."
```

Le parent décide quoi faire.

C'est une bonne séparation des responsabilités.

---

# 27. Pourquoi l'enfant ne devrait pas appeler directement le parent ?

Un component enfant ne devrait généralement pas connaître directement son parent.

Mauvaise idée conceptuelle :

```text
Child
 ↓
connaît Parent
 ↓
appelle directement une méthode
```

Meilleure architecture :

```text
Child
 ↓
Output
 ↓
Parent
```

Cela réduit le couplage.

---

# 28. Un Output représente un événement, pas une commande

Un Output devrait généralement exprimer un événement :

```text
saved
deleted
closed
selected
submitted
```

plutôt qu'une commande très spécifique au parent.

Par exemple :

```text
userDeleted
```

peut être raisonnable.

Mais un événement comme :

```text
deleteUserFromDatabaseAndRefreshParent
```

révèle trop de détails de l'implémentation du parent.

L'enfant doit rester générique.

---

# 29. Flux unidirectionnel

Input/Output favorise un flux clair :

```text
Parent state
     ↓
    Input
     ↓
   Child
     ↓
   Output
     ↓
Parent event handler
     ↓
Parent state change
```

Puis le nouveau state redescend :

```text
Parent
  ↓
Input
  ↓
Child
```

Cela donne un cycle prévisible.

---

# 30. Analogie avec .NET

Tu peux comparer conceptuellement :

```text
Angular Input
≈ donnée fournie à une dépendance/composant

Angular Output
≈ événement publié par le composant

Angular Service
≈ service injecté via DI
```

Mais attention :

`Input/Output` n'est pas l'équivalent exact d'un événement C#.

C'est une analogie pour comprendre le flux de communication.

---

# 31. Questions d'entretien

### Quelle est la différence entre Input et Output ?

> Input permet au parent de transmettre une donnée à l'enfant. Output permet à l'enfant d'émettre un événement vers le parent.

### À quoi sert `$event` ?

> `$event` contient la valeur émise par l'événement.

### Quelle différence entre `input()` et `@Input()` ?

> Ce sont deux APIs permettant de déclarer des Inputs. `input()` correspond à l'API moderne et fournit un Input sous forme de Signal, tandis que `@Input()` correspond à l'API classique.

### Quelle différence entre `output()` et `@Output()` ?

> `output()` est l'API moderne de déclaration des événements d'un component. `@Output()` avec `EventEmitter` correspond à l'approche classique.

### Comment faire communiquer deux components frères ?

> Généralement via leur parent : l'un émet un événement avec Output, le parent met à jour son état, puis transmet la nouvelle valeur à l'autre avec Input.

### Quand utiliser un service plutôt qu'Input/Output ?

> Lorsqu'un état ou une logique doit être partagé entre plusieurs components qui ne sont pas dans une relation parent/enfant simple.

### Pourquoi éviter que l'enfant modifie directement l'état du parent ?

> Pour conserver un flux de données prévisible et réduire le couplage entre les components.

---

# À retenir

```text
INPUT
Parent
  ↓
Child

OUTPUT
Child
  ↓
Parent
```

API classique :

```typescript
@Input()
@Output()
new EventEmitter<T>()
```

API moderne :

```typescript
input<T>()
input.required<T>()

output<T>()
```

Lecture d'un Input moderne :

```typescript
this.user()
```

Émission d'un Output moderne :

```typescript
this.deleted.emit(id);
```

Dans le parent :

```html
[hotel]="hotel"
```

signifie :

```text
donnée → enfant
```

Et :

```html
(deleted)="deleteUser($event)"
```

signifie :

```text
événement ← enfant
```

## Phrase à mémoriser

> **En Angular, les données descendent avec Input et les événements remontent avec Output : le parent possède l'état, l'enfant reçoit ce dont il a besoin et demande les changements en émettant des événements.**
