# Angular Routing

Le Routing permet de naviguer entre différentes vues d'une application Angular sans recharger toute l'application.

## Mental model

```text
URL
  ↓
Angular Router
  ↓
Route correspondante
  ↓
Component
```

Exemples :

```text
/hotels
/hotels/12
/login
/users
```

## Définir des routes

Exemple :

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

Angular associe alors une URL à un component.

## Navigation

Dans le template :

```html
<a routerLink="/hotels">
  Hotels
</a>
```

Ou avec le Router :

```typescript
this.router.navigate(['/hotels']);
```

## Paramètres de route

Une route peut contenir un paramètre :

```text
/hotels/:id
```

Pour :

```text
/hotels/12
```

`id` vaut `12`.

## Guards

Les guards permettent notamment de contrôler l'accès ou la navigation.

Mental model :

```text
Navigation
    ↓
Guard
    ↓
Autorisé ?
   /  Oui  Non
  |    |
  v    v
Route  Refus
```

## À retenir

> Routing = associer une URL à une vue et gérer la navigation dans l'application Angular.
