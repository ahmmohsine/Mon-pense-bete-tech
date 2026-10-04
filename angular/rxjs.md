# Angular — RxJS

**RxJS** est une bibliothèque de programmation réactive utilisée largement par Angular.

Elle fournit notamment :

```text
Observable
Subject
Operators
Subscription
```

Dans Angular, RxJS est particulièrement important pour :

```text
HTTP
événements
flux asynchrones
combinaisons de données
annulation
transformations
gestion d'erreurs
```

La première idée à retenir est :

```text
Observable
    ↓
représente un flux de valeurs dans le temps
```

---

# 1. Pourquoi RxJS ?

Prenons un appel HTTP :

```typescript
this.http.get<Hotel[]>('/api/hotels');
```

Le résultat n'est pas directement :

```typescript
Hotel[]
```

Il est généralement :

```typescript
Observable<Hotel[]>
```

Pourquoi ?

Parce que la réponse HTTP arrivera plus tard.

Mental model :

```text
Maintenant
  ↓
demande HTTP
  ↓
...
  ↓
réponse
  ↓
Hotel[]
```

RxJS fournit une abstraction pour représenter ce type de flux asynchrone.

---

# 2. Observable

Un Observable représente une source pouvant produire des valeurs dans le temps.

Exemple conceptuel :

```text
Observable
   ↓
10
   ↓
20
   ↓
30
   ↓
complete
```

Un Observable peut donc produire :

```text
0 valeur
1 valeur
plusieurs valeurs
```

et éventuellement terminer :

```text
complete
```

ou échouer :

```text
error
```

---

# 3. Les trois notifications principales

Un Observable peut communiquer avec son consommateur via :

```text
next
error
complete
```

Conceptuellement :

```text
Observable
    │
    ├── next(value)
    │
    ├── next(value)
    │
    ├── next(value)
    │
    └── complete()
```

Ou :

```text
Observable
    │
    ├── next(value)
    │
    └── error(error)
```

Après `error` ou `complete`, le flux est terminé.

---

# 4. Créer un Observable

Exemple :

```typescript
import { Observable } from 'rxjs';

const numbers$ = new Observable<number>(subscriber => {

  subscriber.next(1);
  subscriber.next(2);
  subscriber.next(3);

  subscriber.complete();
});
```

Convention très courante :

```text
numbers$
```

Le suffixe `$` indique généralement :

> Cette variable représente un Observable.

Ce n'est pas une règle imposée par TypeScript.

C'est une convention de nommage.

---

# 5. S'abonner avec `subscribe`

Un Observable ne signifie pas nécessairement :

> Exécute immédiatement tout le code.

Il faut généralement s'abonner pour consommer le flux.

Exemple :

```typescript
numbers$.subscribe(value => {
  console.log(value);
});
```

Résultat :

```text
1
2
3
```

Mental model :

```text
Observable
    ↓
subscribe()
    ↓
consommation du flux
```

---

# 6. `subscribe` avec les trois handlers

On peut écrire :

```typescript
numbers$.subscribe({
  next: value => {
    console.log(value);
  },

  error: error => {
    console.error(error);
  },

  complete: () => {
    console.log('Completed');
  }
});
```

Cela correspond à :

```text
next
→ nouvelle valeur

error
→ erreur

complete
→ fin du flux
```

---

# 7. Observable et HTTP

Exemple :

```typescript
getHotels() {
  return this.http.get<Hotel[]>('/api/hotels');
}
```

Le Component :

```typescript
this.hotelService.getHotels()
  .subscribe({
    next: hotels => {
      console.log(hotels);
    },

    error: error => {
      console.error(error);
    }
  });
```

Flux :

```text
HttpClient
   ↓
Observable<Hotel[]>
   ↓
subscribe()
   ↓
réponse HTTP
   ↓
hotels
```

---

# 8. Pourquoi l'Observable est utile pour HTTP ?

Un appel HTTP est asynchrone.

Le code ne peut pas supposer que la réponse est immédiatement disponible.

Mauvaise intuition :

```typescript
const hotels = this.http.get<Hotel[]>('/api/hotels');

console.log(hotels);
```

`hotels` est un :

```text
Observable<Hotel[]>
```

et non directement :

```text
Hotel[]
```

Il faut consommer le flux.

---

# 9. `pipe()`

RxJS permet de transformer un Observable grâce à des operators.

Syntaxe :

```typescript
observable.pipe(
  operator1(),
  operator2(),
  operator3()
);
```

Mental model :

```text
Observable
    ↓
operator
    ↓
operator
    ↓
operator
    ↓
nouvel Observable
```

Exemple :

```typescript
this.http.get<Hotel[]>('/api/hotels')
  .pipe(
    map(hotels => hotels.filter(h => h.active))
  );
```

---

# 10. `map`

`map` transforme chaque valeur émise.

Exemple :

```typescript
of(1, 2, 3).pipe(
  map(value => value * 2)
);
```

Le flux devient :

```text
1 → 2
2 → 4
3 → 6
```

Mental model :

```text
map
=
transformer chaque valeur
```

Attention :

```text
Array.map()
```

et :

```text
RxJS map()
```

ont une idée similaire mais travaillent sur des abstractions différentes.

```text
Array.map()
→ éléments d'un tableau

RxJS map()
→ valeurs émises par un Observable
```

---

# 11. `filter`

`filter` laisse passer uniquement les valeurs qui respectent une condition.

Exemple :

```typescript
of(1, 2, 3, 4).pipe(
  filter(value => value % 2 === 0)
);
```

Résultat :

```text
2
4
```

Mental model :

```text
filter
=
laisser passer ou bloquer
```

---

# 12. `tap`

`tap` permet d'effectuer une action secondaire sans modifier la valeur.

Exemple :

```typescript
this.hotelService.getHotels()
  .pipe(
    tap(hotels => {
      console.log(hotels);
    })
  );
```

La valeur continue de circuler.

```text
Observable
   ↓
tap
   ↓
même valeur
```

`tap` est utile notamment pour :

```text
logging
debug
side effects
```

Il ne doit pas devenir un endroit où l'on construit toute la logique métier.

---

# 13. `catchError`

`catchError` permet de gérer une erreur dans un flux.

Exemple :

```typescript
this.http.get<Hotel[]>('/api/hotels')
  .pipe(
    catchError(error => {
      console.error(error);

      return of([]);
    })
  );
```

Si la requête échoue :

```text
HTTP
 ↓
error
 ↓
catchError
 ↓
Observable de []
```

Attention :

```typescript
return of([]);
```

ne signifie pas :

> l'erreur n'existe plus.

Cela signifie :

> on transforme le flux d'erreur en une nouvelle valeur selon la stratégie choisie.

---

# 14. `finalize`

`finalize` permet d'exécuter une action lorsque le flux se termine, qu'il réussisse ou qu'il échoue.

Exemple :

```typescript
this.loading.set(true);

this.hotelService.getHotels()
  .pipe(
    finalize(() => {
      this.loading.set(false);
    })
  )
  .subscribe(...);
```

Mental model :

```text
start
  ↓
HTTP
  ↓
success OR error
  ↓
finalize
  ↓
cleanup
```

Très pratique pour :

```text
loading indicators
cleanup
```

---

# 15. `switchMap`

`switchMap` est un operator extrêmement important.

Il permet de lancer un nouvel Observable à partir d'une valeur tout en abandonnant l'intérêt pour le flux précédent lorsqu'une nouvelle valeur arrive.

Exemple classique :

```text
recherche utilisateur
```

L'utilisateur tape :

```text
h
ho
hot
hote
hotel
```

On ne veut pas nécessairement traiter toutes les anciennes recherches.

Conceptuellement :

```text
"h"
 ↓
requête A

"ho"
 ↓
requête B
 ↓
A devient obsolète

"hot"
 ↓
requête C
 ↓
B devient obsolète
```

`switchMap` permet cette stratégie.

Exemple :

```typescript
searchTerms$.pipe(
  switchMap(term =>
    this.hotelService.search(term)
  )
);
```

Mental model :

```text
nouvelle valeur
     ↓
nouvel Observable
     ↓
remplace le précédent
```

---

# 16. `mergeMap`

`mergeMap` permet de souscrire à plusieurs Observables produits et de les laisser s'exécuter de manière concurrente.

Exemple conceptuel :

```text
A
 ↓
requête A

B
 ↓
requête B

C
 ↓
requête C
```

Les requêtes peuvent être actives simultanément.

Mental model :

```text
switchMap
→ nouvelle requête remplace l'ancienne

mergeMap
→ plusieurs requêtes peuvent rester actives
```

Le choix dépend du comportement souhaité.

---

# 17. `concatMap`

`concatMap` traite les Observables les uns après les autres.

Exemple :

```text
A
 ↓
requête A
 ↓
terminée

B
 ↓
requête B
 ↓
terminée

C
 ↓
requête C
```

Mental model :

```text
concatMap
=
file d'attente
```

Très utile lorsqu'il faut préserver un ordre d'exécution.

---

# 18. `exhaustMap`

`exhaustMap` ignore les nouvelles valeurs tant que le premier Observable n'est pas terminé.

Exemple classique :

```text
Bouton Submit
```

L'utilisateur clique plusieurs fois rapidement.

On veut :

```text
clic 1
 ↓
requête active
 ↓
clic 2 → ignoré
clic 3 → ignoré
clic 4 → ignoré
 ↓
requête terminée
```

Mental model :

```text
exhaustMap
=
"Je termine d'abord ce que je fais."
```

---

# 19. Les quatre opérateurs à mémoriser

```text
switchMap
→ dernier uniquement / précédent remplacé

mergeMap
→ concurrence

concatMap
→ séquence

exhaustMap
→ ignore pendant qu'une opération est active
```

Image mentale :

```text
switchMap
A ──X
 B ──X
  C ─────✓

mergeMap
A ─────✓
 B ───✓
  C ──✓

concatMap
A ──✓
    B ──✓
         C ──✓

exhaustMap
A ─────✓
 B → ignoré
 C → ignoré
```

---

# 20. `Subject`

Un `Subject` est à la fois :

```text
Observable
+
Observer
```

Cela signifie qu'on peut :

```typescript
subject.next(value);
```

et que plusieurs consommateurs peuvent s'abonner :

```typescript
subject.subscribe(...);
```

Mental model :

```text
Producteur
    ↓
 Subject
   ↙   ↘
Consumer A  Consumer B
```

---

# 21. Pourquoi utiliser un Subject ?

Exemple :

```typescript
private refreshSubject = new Subject<void>();
```

Puis :

```typescript
refresh() {
  this.refreshSubject.next();
}
```

D'autres parties peuvent écouter :

```typescript
this.refreshSubject.subscribe(() => {
  this.loadData();
});
```

Mais attention :

> Un Subject ne doit pas être utilisé automatiquement pour tout.

Avec Angular moderne, Signals sont souvent très adaptés à l'état local ou partagé.

---

# 22. `BehaviorSubject`

Un `BehaviorSubject` conserve une valeur actuelle et nécessite une valeur initiale.

Exemple :

```typescript
const count$ = new BehaviorSubject<number>(0);
```

Un nouvel abonné reçoit immédiatement la valeur actuelle.

```text
BehaviorSubject
      │
      ├── valeur actuelle = 10
      │
      └── nouvel abonné
              ↓
             10
```

Cela le différencie d'un `Subject` classique.

---

# 23. Subject vs BehaviorSubject

### Subject

```text
pas de valeur actuelle stockée
```

### BehaviorSubject

```text
possède une valeur actuelle
+
la fournit au nouvel abonné
```

Mental model :

```text
Subject
→ événement

BehaviorSubject
→ état actuel + flux de changements
```

Cependant, pour représenter de l'état Angular moderne, un Signal peut souvent être plus naturel.

---

# 24. Subscription

Quand on fait :

```typescript
const subscription =
  observable.subscribe(...);
```

on obtient une `Subscription`.

On peut éventuellement appeler :

```typescript
subscription.unsubscribe();
```

pour arrêter l'abonnement.

Mental model :

```text
subscribe()
    ↓
Subscription
    ↓
unsubscribe()
```

---

# 25. Attention aux subscriptions

Un abonnement long vivant peut maintenir des références et continuer à exécuter du code.

Il faut donc réfléchir au cycle de vie.

Exemple problématique :

```typescript
ngOnInit() {
  this.someObservable.subscribe(value => {
    ...
  });
}
```

si l'Observable ne se termine jamais et que l'abonnement n'est jamais nettoyé.

Cela peut provoquer des comportements indésirables et potentiellement des fuites de ressources.

---

# 26. `takeUntilDestroyed`

Angular moderne fournit des mécanismes permettant de lier automatiquement la durée d'un abonnement au cycle de vie du Component.

Exemple :

```typescript
this.someObservable
  .pipe(
    takeUntilDestroyed(this.destroyRef)
  )
  .subscribe(value => {
    console.log(value);
  });
```

L'idée :

```text
Component vivant
    ↓
subscription active

Component détruit
    ↓
subscription nettoyée
```

Cela évite de gérer manuellement certaines subscriptions.

---

# 27. HTTP et nettoyage

Les Observables HTTP d'Angular sont généralement des flux courts :

```text
request
 ↓
response
 ↓
complete
```

Ils se terminent normalement après la réponse.

Le problème des subscriptions longues concerne davantage :

```text
events
WebSocket
Subjects
intervals
streams continus
```

Il faut donc connaître la durée de vie du flux avant de décider comment le gérer.

---

# 28. `async` pipe

Angular fournit également le pipe :

```html
{{ hotels$ | async }}
```

Il permet au template de consommer un Observable.

Exemple :

```typescript
hotels$ = this.hotelService.getHotels();
```

Template :

```html
@for (hotel of (hotels$ | async); track hotel.id) {
  <p>{{ hotel.name }}</p>
}
```

L'`async` pipe peut notamment gérer l'abonnement et le nettoyage associé au cycle de vie du template.

---

# 29. `async` pipe vs `subscribe`

Approche manuelle :

```typescript
this.hotelService.getHotels()
  .subscribe(hotels => {
    this.hotels = hotels;
  });
```

Approche template :

```typescript
hotels$ = this.hotelService.getHotels();
```

Puis :

```html
@for (hotel of (hotels$ | async); track hotel.id) {
  <p>{{ hotel.name }}</p>
}
```

Le choix dépend du besoin.

Si tu dois effectuer une action impérative :

```text
save
navigation
notification
modification d'un autre état
```

un `subscribe` peut être approprié.

Si tu veux simplement afficher le flux :

```text
Observable → template
```

l'`async` pipe peut être très pratique.

---

# 30. RxJS et Signals ensemble

Angular moderne permet parfaitement d'utiliser les deux.

Architecture possible :

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
    ↓
Template
```

Par exemple :

```typescript
hotels = signal<Hotel[]>([]);

loadHotels() {
  this.hotelService.getHotels()
    .subscribe(hotels => {
      this.hotels.set(hotels);
    });
}
```

Ici :

```text
HTTP
→ Observable

UI state
→ Signal
```

---

# 31. Observable → Signal

Angular fournit des outils permettant de convertir un Observable en Signal.

Conceptuellement :

```text
Observable
    ↓
toSignal()
    ↓
Signal
```

Exemple :

```typescript
hotels = toSignal(
  this.hotelService.getHotels(),
  {
    initialValue: []
  }
);
```

On peut ensuite utiliser :

```typescript
this.hotels()
```

dans le code et :

```html
@for (hotel of hotels(); track hotel.id) {
  <p>{{ hotel.name }}</p>
}
```

Cette approche peut être très pratique lorsque la source est naturellement un Observable mais que le Component veut travailler avec un état réactif sous forme de Signal.

---

# 32. Signal → Observable

L'inverse est également possible avec les outils interop d'Angular.

Conceptuellement :

```text
Signal
   ↓
toObservable()
   ↓
Observable
```

C'est utile lorsqu'une partie de ton application ou une bibliothèque attend un Observable.

---

# 33. Pourquoi ne pas choisir un seul des deux ?

Parce qu'ils excellent dans des domaines différents.

```text
Signal
→ état réactif

Observable
→ flux / événements / asynchronisme / RxJS
```

Exemple :

```text
HTTP
 ↓
Observable
 ↓
transformation RxJS
 ↓
Signal
 ↓
UI
```

C'est une combinaison parfaitement valide.

---

# 34. `debounceTime`

Très utile pour les champs de recherche.

Sans debounce :

```text
h
→ requête

ho
→ requête

hot
→ requête

hotel
→ requête
```

Avec :

```typescript
debounceTime(300)
```

on attend 300 ms sans nouvelle valeur avant de continuer.

Conceptuellement :

```text
h
  ↓
ho
  ↓
hot
  ↓
hotel
  ↓
300ms sans frappe
  ↓
requête
```

Cela réduit les appels inutiles.

---

# 35. `distinctUntilChanged`

Permet d'éviter de traiter deux valeurs consécutives identiques.

Exemple :

```text
A
A
B
B
C
```

Avec :

```typescript
distinctUntilChanged()
```

on obtient conceptuellement :

```text
A
B
C
```

Très utile pour :

```text
search
filters
form values
```

---

# 36. Exemple recherche Angular

```typescript
searchControl.valueChanges
  .pipe(
    debounceTime(300),
    distinctUntilChanged(),
    switchMap(term =>
      this.hotelService.search(term)
    )
  )
  .subscribe(hotels => {
    this.hotels.set(hotels);
  });
```

Flux :

```text
Input utilisateur
      ↓
valueChanges
      ↓
debounceTime
      ↓
distinctUntilChanged
      ↓
switchMap
      ↓
HTTP
      ↓
hotels
```

Chaque operator possède donc une responsabilité claire.

---

# 37. `combineLatest`

`combineLatest` permet de combiner plusieurs flux.

Exemple :

```text
filters$
sort$
```

On veut recalculer lorsque l'un des deux change.

Conceptuellement :

```text
filters$ ──────┐
               ├──→ combineLatest
sort$ ─────────┘
                    ↓
                 résultat
```

Il émet lorsque les sources ont fourni au moins une valeur et qu'une valeur ultérieure change.

---

# 38. `forkJoin`

`forkJoin` est particulièrement utile lorsqu'on veut attendre que plusieurs Observables terminent et récupérer leurs dernières valeurs.

Exemple :

```typescript
forkJoin({
  hotels: this.hotelService.getHotels(),
  cities: this.cityService.getCities()
});
```

Mental model :

```text
Hotels ───────✓
Cities ────────✓
               ↓
          forkJoin
               ↓
        résultat final
```

Pour des requêtes HTTP indépendantes qui se terminent, c'est un cas fréquent.

---

# 39. `switchMap` vs `forkJoin`

Ils répondent à des problèmes différents.

### `switchMap`

```text
Une nouvelle valeur arrive
→ lancer un nouveau flux
→ abandonner l'intérêt pour l'ancien
```

### `forkJoin`

```text
J'ai plusieurs opérations indépendantes
→ attends qu'elles terminent toutes
→ récupère le résultat final
```

Mental model :

```text
switchMap
→ changement de contexte

forkJoin
→ attendre plusieurs opérations
```

---

# 40. Les operators : penser en pipeline

Exemple :

```typescript
source$.pipe(
  debounceTime(300),
  distinctUntilChanged(),
  filter(term => term.length >= 2),
  switchMap(term => this.search(term)),
  catchError(() => of([]))
);
```

Lis-le de haut en bas :

```text
source
  ↓
attendre
  ↓
ignorer les doublons
  ↓
filtrer
  ↓
lancer la recherche
  ↓
gérer l'erreur
```

C'est la meilleure façon de lire un pipeline RxJS.

---

# 41. RxJS et LINQ : comparaison utile pour .NET

Comme tu connais C#, une comparaison peut aider.

Côté C# :

```csharp
users
    .Where(u => u.Active)
    .Select(u => u.Name);
```

Côté RxJS :

```typescript
users$.pipe(
  filter(u => u.active),
  map(u => u.name)
);
```

L'idée est similaire :

```text
Where
≈ filter

Select
≈ map
```

Mais attention :

```text
LINQ
→ collections / requêtes

RxJS
→ flux de valeurs dans le temps
```

Cette différence est essentielle.

---

# 42. Erreur fréquente : confondre `map` et `subscribe`

`map` transforme le flux :

```typescript
observable.pipe(
  map(value => transform(value))
);
```

`subscribe` consomme le flux :

```typescript
observable.subscribe(value => {
  ...
});
```

Mental model :

```text
map
→ transformation

subscribe
→ consommation
```

---

# 43. Erreur fréquente : imbriquer les subscriptions

À éviter lorsque possible :

```typescript
this.service.getUser().subscribe(user => {

  this.service.getOrders(user.id)
    .subscribe(orders => {

      ...
    });

});
```

Cela crée un code difficile à gérer.

On peut souvent utiliser :

```typescript
switchMap
```

ou un autre operator adapté.

Exemple :

```typescript
this.service.getUser()
  .pipe(
    switchMap(user =>
      this.service.getOrders(user.id)
    )
  )
  .subscribe(orders => {
    ...
  });
```

Mental model :

```text
Observable
   ↓
operator
   ↓
Observable
   ↓
subscribe une fois
```

---

# 44. Choisir le bon flattening operator

Question à se poser :

### Je veux uniquement la dernière demande ?

```text
switchMap
```

### Je veux exécuter plusieurs opérations en parallèle ?

```text
mergeMap
```

### Je dois respecter l'ordre ?

```text
concatMap
```

### Je veux ignorer les nouveaux événements pendant une opération ?

```text
exhaustMap
```

C'est une question très fréquente en entretien Angular.

---

# 45. Questions d'entretien

### Qu'est-ce qu'un Observable ?

> Un Observable représente un flux de valeurs qui peut évoluer dans le temps et notifier ses consommateurs avec `next`, `error` et `complete`.

### À quoi sert `subscribe()` ?

> À consommer un Observable et à définir ce qu'on fait avec les valeurs, les erreurs ou la fin du flux.

### À quoi sert `pipe()` ?

> À composer des operators RxJS pour transformer ou contrôler un flux.

### Quelle différence entre `map` et `subscribe` ?

> `map` transforme les valeurs du flux. `subscribe` consomme le flux.

### Quelle différence entre `switchMap` et `mergeMap` ?

> `switchMap` abandonne l'ancien flux interne lorsqu'une nouvelle valeur arrive. `mergeMap` permet à plusieurs flux internes de s'exécuter en parallèle.

### Quelle différence entre `Subject` et `BehaviorSubject` ?

> Un `Subject` ne fournit pas automatiquement une valeur actuelle aux nouveaux abonnés. Un `BehaviorSubject` conserve une valeur courante et la fournit immédiatement à un nouvel abonné.

### Signal ou Observable ?

> Un Signal représente très bien un état réactif actuel, tandis qu'un Observable représente particulièrement bien un flux de valeurs, notamment les opérations asynchrones et l'écosystème RxJS. Ils peuvent être utilisés ensemble.

### Pourquoi éviter les subscriptions imbriquées ?

> Elles rendent le code difficile à lire et à gérer. Les operators comme `switchMap`, `concatMap` ou `mergeMap` permettent de composer les flux proprement.

---

# À retenir

```text
Observable
→ flux de valeurs dans le temps

subscribe()
→ consommer le flux

pipe()
→ construire un pipeline

map()
→ transformer

filter()
→ filtrer

tap()
→ effet secondaire sans transformer la valeur

catchError()
→ gérer une erreur

finalize()
→ nettoyage après fin du flux

switchMap()
→ dernier flux

mergeMap()
→ concurrence

concatMap()
→ séquence

exhaustMap()
→ ignorer pendant une opération

Subject
→ Observable + possibilité d'émettre

BehaviorSubject
→ Subject + valeur actuelle

async pipe
→ consommer un Observable dans le template

Signal
→ état réactif actuel
```

## Phrase à mémoriser

> **RxJS me permet de manipuler des flux de valeurs dans le temps : `pipe()` construit le traitement, les operators transforment ou contrôlent le flux, et `subscribe()` le consomme ; avec Angular moderne, je peux utiliser les Observables pour les flux asynchrones et les Signals pour l'état réactif de l'interface.**
