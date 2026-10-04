# Angular — Services

Un **Service Angular** est une classe destinée à contenir une logique qui doit être séparée des Components et éventuellement réutilisée par plusieurs parties de l'application.

Dans une application Angular connectée à une API ASP.NET Core, on rencontre très souvent :

```text
Component
    ↓
Service
    ↓
HttpClient
    ↓
ASP.NET Core API
```

Le Service devient donc notamment la couche qui centralise les communications avec le backend.

---

# 1. Pourquoi utiliser un Service ?

On pourrait techniquement faire un appel HTTP directement dans un Component :

```typescript
export class HotelListComponent {

  constructor(private http: HttpClient) {}

  loadHotels() {
    return this.http.get<Hotel[]>('/api/hotels');
  }
}
```

Mais si plusieurs Components ont besoin des hôtels :

```text
HotelListComponent
HotelDetailsComponent
HotelEditComponent
```

on risque de répéter :

```typescript
this.http.get(...)
this.http.post(...)
this.http.put(...)
this.http.delete(...)
```

On préfère :

```text
HotelListComponent
        ↓
   HotelService
        ↓
    HttpClient
        ↓
       API
```

Le Service centralise donc la communication.

---

# 2. Créer un Service

On utilise généralement le décorateur :

```typescript
@Injectable({
  providedIn: 'root'
})
```

Exemple :

```typescript
import { Injectable } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class HotelService {

}
```

Le décorateur indique à Angular que la classe peut participer au système de Dependency Injection.

---

# 3. `providedIn: 'root'`

Exemple :

```typescript
@Injectable({
  providedIn: 'root'
})
```

Cela signifie que le Service est fourni au niveau racine de l'application.

Mental model :

```text
Application
     ↓
DI container Angular
     ↓
HotelService
```

Le Service peut alors être injecté dans différents Components ou autres Services.

Dans le cas classique de `providedIn: 'root'`, Angular fournit une instance partagée au niveau racine.

---

# 4. Injection du Service dans un Component

Exemple :

```typescript
export class HotelListComponent {

  constructor(
    private hotelService: HotelService
  ) {}

}
```

Le Component dit conceptuellement :

> J'ai besoin d'une instance de `HotelService`.

Angular résout cette dépendance.

Flux :

```text
Component
    ↓
"J'ai besoin de HotelService"
    ↓
Angular DI
    ↓
HotelService
```

C'est le même grand principe de Dependency Injection que celui que tu utilises en ASP.NET Core.

---

# 5. Comparaison avec ASP.NET Core

En .NET :

```csharp
builder.Services.AddScoped<IHotelService, HotelService>();
```

Puis :

```csharp
public HotelController(IHotelService hotelService)
{
    _hotelService = hotelService;
}
```

Angular :

```typescript
@Injectable({
  providedIn: 'root'
})
export class HotelService {
}
```

Puis :

```typescript
constructor(
  private hotelService: HotelService
) {}
```

Dans les deux cas :

```text
Classe A
   ↓
demande une dépendance
   ↓
système DI
   ↓
classe B
```

Les mécanismes internes ne sont pas identiques, mais le principe architectural est très proche.

---

# 6. Service avec HttpClient

Pour appeler une API, on utilise généralement `HttpClient`.

Exemple :

```typescript
import { HttpClient } from '@angular/common/http';

@Injectable({
  providedIn: 'root'
})
export class HotelService {

  constructor(
    private http: HttpClient
  ) {}

  getHotels() {
    return this.http.get<Hotel[]>('/api/hotels');
  }
}
```

Le Service devient :

```text
HotelService
    ↓
HttpClient
    ↓
HTTP
    ↓
ASP.NET Core
```

---

# 7. Pourquoi typer `HttpClient` ?

On peut écrire :

```typescript
this.http.get<Hotel[]>('/api/hotels');
```

Le :

```typescript
<Hotel[]>
```

indique le type attendu.

On obtient conceptuellement :

```typescript
Observable<Hotel[]>
```

Cela permet à TypeScript et à l'IDE de mieux comprendre le résultat.

Attention :

> Le type TypeScript ne valide pas magiquement que le serveur renvoie réellement un `Hotel[]`.

Il décrit ce que ton code considère comme le format attendu.

La validation réelle des données venant du réseau doit être pensée séparément si elle est nécessaire.

---

# 8. Les méthodes HTTP dans un Service

Un Service API peut regrouper les opérations CRUD.

Exemple :

```typescript
getHotels() {
  return this.http.get<Hotel[]>('/api/hotels');
}
```

```typescript
getHotel(id: number) {
  return this.http.get<Hotel>(`/api/hotels/${id}`);
}
```

```typescript
createHotel(hotel: CreateHotelDto) {
  return this.http.post<Hotel>(
    '/api/hotels',
    hotel
  );
}
```

```typescript
updateHotel(id: number, hotel: UpdateHotelDto) {
  return this.http.put<Hotel>(
    `/api/hotels/${id}`,
    hotel
  );
}
```

```typescript
deleteHotel(id: number) {
  return this.http.delete<void>(
    `/api/hotels/${id}`
  );
}
```

On obtient :

```text
GET
→ récupérer

POST
→ créer

PUT
→ modifier

DELETE
→ supprimer
```

---

# 9. Service et DTO

Comme côté ASP.NET Core, il est souvent préférable de ne pas faire circuler n'importe quel objet interne.

Exemple :

```typescript
export interface CreateHotelDto {
  name: string;
  address: string;
}
```

Puis :

```typescript
createHotel(dto: CreateHotelDto) {
  return this.http.post<Hotel>(
    '/api/hotels',
    dto
  );
}
```

Mental model :

```text
Angular Form
    ↓
CreateHotelDto
    ↓
HTTP POST
    ↓
ASP.NET Core DTO
```

Cela crée un contrat clair entre frontend et backend.

---

# 10. Où placer l'URL de l'API ?

Évite de disperser :

```typescript
'http://localhost:5000/api/...'
```

dans tous les Services.

On utilise généralement la configuration d'environnement Angular.

Par exemple conceptuellement :

```typescript
export const environment = {
  apiUrl: 'https://localhost:7000'
};
```

Puis :

```typescript
this.http.get<Hotel[]>(
  `${environment.apiUrl}/api/hotels`
);
```

L'idée est :

```text
Development
→ API locale

Production
→ API Azure
```

sans modifier chaque Service.

---

# 11. Service et séparation des responsabilités

Un Component devrait principalement gérer :

```text
UI
état local
interaction utilisateur
```

Un Service peut gérer :

```text
communication HTTP
logique réutilisable
état partagé
transformation de données
```

Exemple :

```text
HotelListComponent
        ↓
HotelService
        ↓
HttpClient
        ↓
API
```

Cela permet d'éviter un Component énorme.

---

# 12. Un Service n'est pas forcément un Service HTTP

Important :

> Tous les Services Angular ne servent pas à appeler une API.

On peut avoir :

```text
AuthService
CartService
StorageService
NotificationService
ThemeService
HotelService
```

Par exemple :

```typescript
@Injectable({
  providedIn: 'root'
})
export class ThemeService {

  private darkMode = signal(false);

  toggle() {
    this.darkMode.update(value => !value);
  }

  isDarkMode() {
    return this.darkMode();
  }
}
```

Ce Service ne communique avec aucune API.

Il encapsule simplement une logique partagée.

---

# 13. Service avec état partagé

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

  remove(id: number) {
    this.items.update(items =>
      items.filter(item => item.id !== id)
    );
  }
}
```

Plusieurs Components peuvent utiliser le même Service :

```text
Header
   │
   └── CartService

ProductList
   │
   └── CartService

Checkout
   │
   └── CartService
```

Le Service peut donc devenir une source d'état partagé.

---

# 14. Pourquoi `private` ?

Dans :

```typescript
private items = signal<CartItem[]>([]);
```

on indique que l'état interne ne doit pas être modifié directement depuis l'extérieur.

Puis :

```typescript
readonly cartItems = this.items.asReadonly();
```

permet de l'exposer en lecture.

On obtient :

```text
extérieur
   ↓
cartItems
   ↓
lecture

Service
   ↓
items
   ↓
écriture
```

Le Service reste propriétaire de son état.

---

# 15. Service comme façade

Un Service peut également servir de façade devant plusieurs opérations.

Par exemple :

```text
Component
    ↓
AuthService
    ├── HttpClient
    ├── Token handling
    ├── User state
    └── Logout
```

Le Component n'a pas besoin de connaître tous les détails.

Il peut simplement faire :

```typescript
this.authService.login(credentials);
```

Le Service s'occupe du reste.

Mental model :

```text
Component
    ↓
API simple
    ↓
Service
    ↓
complexité interne
```

---

# 16. Service et RxJS

`HttpClient` retourne généralement des Observables.

Exemple :

```typescript
getHotels() {
  return this.http.get<Hotel[]>('/api/hotels');
}
```

Le Component peut s'abonner :

```typescript
this.hotelService.getHotels()
  .subscribe(hotels => {
    console.log(hotels);
  });
```

Mais un Service peut aussi utiliser les opérateurs RxJS.

Exemple :

```typescript
getActiveHotels() {
  return this.http.get<Hotel[]>('/api/hotels')
    .pipe(
      map(hotels =>
        hotels.filter(hotel => hotel.active)
      )
    );
}
```

Le Component reçoit alors directement le résultat dont il a besoin.

---

# 17. Où mettre une transformation de données ?

Cela dépend de la responsabilité.

Si la transformation est spécifique à l'affichage :

```text
Component
```

peut être approprié.

Si elle est réutilisée ou liée aux données :

```text
Service
```

peut être plus adapté.

Exemple :

```text
API
 ↓
HotelService
 ↓
transformation
 ↓
Component
```

Mais il faut éviter de transformer le Service en une classe contenant toute la logique métier de l'application.

---

# 18. Service et business logic

Angular est principalement le frontend.

La logique métier critique appartient généralement au backend si elle doit être sécurisée ou faire autorité.

Exemple :

```text
Angular
→ affiche "prix = 100"

ASP.NET Core
→ vérifie réellement le prix autorisé
```

Ne jamais considérer une validation frontend comme une protection de sécurité suffisante.

Le frontend peut améliorer l'expérience utilisateur.

Le backend doit faire respecter les règles de sécurité et les règles métier qui doivent être fiables.

---

# 19. Service et sécurité

Exemple :

```typescript
authService.isLoggedIn()
```

peut être utile pour l'interface.

Mais :

```text
Angular Guard
```

ne remplace pas l'autorisation backend.

Un utilisateur peut toujours contourner le frontend.

Donc :

```text
Angular
→ UX / navigation / affichage

ASP.NET Core
→ véritable autorisation
```

C'est particulièrement important pour une application avec JWT ou authentification.

---

# 20. Injection d'un Service dans un autre Service

Les Services peuvent eux-mêmes dépendre d'autres Services.

Exemple :

```typescript
@Injectable({
  providedIn: 'root'
})
export class HotelService {

  constructor(
    private http: HttpClient,
    private authService: AuthService
  ) {}

}
```

Flux :

```text
HotelService
    ├── HttpClient
    └── AuthService
```

Angular résout récursivement les dépendances.

Mental model :

```text
Component
   ↓
HotelService
   ├── HttpClient
   └── AuthService
```

---

# 21. Injection moderne avec `inject()`

Angular moderne permet aussi :

```typescript
import { inject } from '@angular/core';

@Injectable({
  providedIn: 'root'
})
export class HotelService {

  private http = inject(HttpClient);

}
```

Au lieu de :

```typescript
constructor(
  private http: HttpClient
) {}
```

Les deux approches sont importantes à connaître.

`inject()` est particulièrement pratique dans certaines configurations modernes et pour les fonctions qui ont accès à un contexte d'injection.

---

# 22. `constructor` vs `inject()`

Classique :

```typescript
constructor(
  private http: HttpClient
) {}
```

Moderne :

```typescript
private http = inject(HttpClient);
```

Les deux signifient conceptuellement :

```text
"Donne-moi une instance de HttpClient."
```

Ce qui change principalement est la syntaxe et le contexte dans lequel l'injection peut être utilisée.

---

# 23. Singleton : attention au vocabulaire

Avec :

```typescript
@Injectable({
  providedIn: 'root'
})
```

on obtient généralement une instance partagée au niveau racine.

On parle souvent de :

```text
singleton
```

Mais il faut comprendre le contexte :

> L'instance est partagée selon le scope du provider.

Ce n'est donc pas une règle absolue disant qu'il n'existera jamais aucune autre instance dans toute situation Angular.

La configuration des providers influence le scope.

---

# 24. Provider au niveau d'un Component

Un Service peut également être fourni dans un Component.

Exemple :

```typescript
@Component({
  selector: 'app-counter',
  providers: [CounterService]
})
export class CounterComponent {
}
```

Dans ce cas, le Service possède un scope lié à l'injecteur du Component.

Cela peut permettre d'avoir des instances séparées.

Exemple :

```text
CounterComponent A
    ↓
CounterService A

CounterComponent B
    ↓
CounterService B
```

Ce comportement est différent d'un Service fourni au niveau root.

---

# 25. Pourquoi le scope est important ?

Supposons un panier :

```text
CartService
```

Si tu veux un panier partagé dans toute l'application :

```typescript
providedIn: 'root'
```

est généralement logique.

Mais si tu veux un état isolé pour chaque instance d'un component :

```typescript
providers: [CartService]
```

peut être approprié.

Mental model :

```text
root provider
→ partage global selon l'injecteur racine

component provider
→ état isolé selon le scope du component
```

---

# 26. Service et testabilité

Un gros avantage des Services est la testabilité.

Au lieu de tester toute l'interface :

```text
Component
 + HTTP
 + logique
 + état
```

on peut tester séparément :

```text
HotelService
```

et :

```text
HotelListComponent
```

Cela permet des tests plus ciblés.

---

# 27. Exemple d'architecture

Pour une application Hotel Listing :

```text
Angular
│
├── Components
│   ├── HotelListComponent
│   ├── HotelDetailsComponent
│   └── HotelFormComponent
│
├── Services
│   ├── HotelService
│   ├── AuthService
│   └── NotificationService
│
├── Models / DTOs
│
└── Routing
        │
        ↓
ASP.NET Core API
        │
        ↓
Services
        │
        ↓
EF Core
        │
        ↓
SQL Server
```

Cette séparation correspond bien à une architecture frontend/backend moderne.

---

# 28. Exemple complet

Service :

```typescript
@Injectable({
  providedIn: 'root'
})
export class HotelService {

  private http = inject(HttpClient);

  getHotels() {
    return this.http.get<Hotel[]>('/api/hotels');
  }

  getHotel(id: number) {
    return this.http.get<Hotel>(
      `/api/hotels/${id}`
    );
  }

  createHotel(dto: CreateHotelDto) {
    return this.http.post<Hotel>(
      '/api/hotels',
      dto
    );
  }

  deleteHotel(id: number) {
    return this.http.delete<void>(
      `/api/hotels/${id}`
    );
  }
}
```

Component :

```typescript
@Component({
  selector: 'app-hotel-list',
  standalone: true,
  templateUrl: './hotel-list.component.html'
})
export class HotelListComponent {

  private hotelService = inject(HotelService);

  hotels = signal<Hotel[]>([]);
  loading = signal(false);

  loadHotels() {

    this.loading.set(true);

    this.hotelService.getHotels()
      .subscribe({
        next: hotels => {
          this.hotels.set(hotels);
          this.loading.set(false);
        },
        error: () => {
          this.loading.set(false);
        }
      });
  }
}
```

La séparation est claire :

```text
Component
→ état UI + interaction

HotelService
→ appels API

HttpClient
→ HTTP

ASP.NET Core
→ backend
```

---

# 29. Ce qu'il faut éviter

Évite un Service gigantesque :

```text
HotelService
├── hotels
├── users
├── authentication
├── payments
├── notifications
├── products
├── orders
└── everything else
```

Mieux :

```text
HotelService
UserService
AuthService
PaymentService
NotificationService
ProductService
OrderService
```

Un Service doit avoir une responsabilité compréhensible.

---

# 30. Service vs Component

| Component | Service |
|---|---|
| UI | logique réutilisable |
| Template | pas de template |
| événements UI | opérations applicatives |
| état local UI | état partagé possible |
| interaction utilisateur | communication API |
| affichage | réutilisation |

La frontière n'est pas absolue, mais c'est une excellente règle de départ.

---

# 31. Service vs Repository

Dans ton backend .NET, tu peux avoir :

```text
Controller
   ↓
Application Service
   ↓
Repository
   ↓
EF Core
```

Dans Angular :

```text
Component
   ↓
Angular Service
   ↓
HttpClient
   ↓
API
```

Il ne faut pas automatiquement appeler un Service Angular un "Repository".

Le Service Angular communique généralement avec une API.

Le Repository backend est une abstraction d'accès aux données.

Ce sont deux responsabilités différentes.

---

# 32. Questions d'entretien

### À quoi sert un Service Angular ?

> À encapsuler une logique réutilisable ou partagée et à séparer cette logique des Components. Il est notamment utilisé pour centraliser les appels HTTP.

### Pourquoi `providedIn: 'root'` ?

> Pour enregistrer le Service dans l'injecteur racine et permettre son injection dans l'application avec un scope racine.

### Comment injecter un Service ?

Classiquement :

```typescript
constructor(
  private hotelService: HotelService
) {}
```

Ou avec l'API moderne :

```typescript
private hotelService = inject(HotelService);
```

### Pourquoi ne pas faire tous les appels HTTP dans les Components ?

> Pour éviter la duplication et séparer la responsabilité de l'interface de la communication avec le backend.

### Un Service Angular est-il forcément un singleton ?

> Pas absolument. `providedIn: 'root'` donne généralement une instance partagée dans le scope racine, mais des providers à d'autres niveaux peuvent créer des instances distinctes.

### Quelle différence entre Service Angular et Repository .NET ?

> Le Service Angular est généralement utilisé côté frontend pour encapsuler de la logique et communiquer avec l'API. Le Repository est généralement une abstraction backend destinée à l'accès aux données.

---

# À retenir

```text
@Injectable
    ↓
rend une classe injectable

providedIn: 'root'
    ↓
provider au niveau racine

Component
    ↓
Service
    ↓
HttpClient
    ↓
API
```

Injection classique :

```typescript
constructor(
  private service: MyService
) {}
```

Injection moderne :

```typescript
private service = inject(MyService);
```

Service HTTP :

```typescript
getHotels() {
  return this.http.get<Hotel[]>('/api/hotels');
}
```

État partagé :

```typescript
private state = signal(...);

readonly state$ = state.asReadonly();
```

Le Service peut être :

```text
API client
+
logique réutilisable
+
état partagé
+
façade
```

Mais il ne doit pas devenir une classe "fourre-tout".

## Phrase à mémoriser

> **Un Service Angular encapsule une logique réutilisable ou partagée ; grâce à la Dependency Injection, les Components peuvent le consommer sans créer eux-mêmes ses dépendances, et dans une application .NET il sert très souvent de couche entre le Component et l'API ASP.NET Core.**
