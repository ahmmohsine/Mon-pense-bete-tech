# Architecture — Repository Pattern

## 1. Qu'est-ce que le Repository Pattern ?

Le **Repository Pattern** consiste à fournir une abstraction permettant au reste de l'application d'accéder aux données sans dépendre directement de leur mécanisme de stockage.

Mentalement :

```text
Application
    ↓
IHotelRepository
    ↓
HotelRepository
    ↓
EF Core
    ↓
SQL Server
```

L'application exprime :

> « J'ai besoin de récupérer ou de sauvegarder un hôtel. »

Elle ne devrait pas nécessairement avoir à connaître :

```text
DbContext
SQL
SQL Server
EF Core
```

---

# 2. Le problème que le Repository cherche à résoudre

Sans abstraction :

```csharp
public class HotelService
{
    private readonly AppDbContext _context;

    public HotelService(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Hotel?> GetAsync(int id)
    {
        return await _context.Hotels
            .FirstOrDefaultAsync(h => h.Id == id);
    }
}
```

Le service connaît directement :

```text
EF Core
DbContext
DbSet
LINQ
```

Avec un Repository :

```csharp
public class HotelService
{
    private readonly IHotelRepository _repository;

    public HotelService(IHotelRepository repository)
    {
        _repository = repository;
    }

    public Task<Hotel?> GetAsync(int id)
    {
        return _repository.GetByIdAsync(id);
    }
}
```

Le service dépend maintenant de :

```text
IHotelRepository
```

---

# 3. Interface du Repository

Exemple :

```csharp
public interface IHotelRepository
{
    Task<Hotel?> GetByIdAsync(
        int id,
        CancellationToken cancellationToken);

    Task AddAsync(
        Hotel hotel,
        CancellationToken cancellationToken);

    void Remove(Hotel hotel);
}
```

L'interface représente le **contrat**.

Elle répond à :

> « Quelles opérations de données sont disponibles ? »

Elle ne dit pas :

> « Comment ces opérations sont-elles techniquement réalisées ? »

---

# 4. Implémentation avec EF Core

Infrastructure peut ensuite implémenter le contrat :

```csharp
public class HotelRepository : IHotelRepository
{
    private readonly AppDbContext _context;

    public HotelRepository(AppDbContext context)
    {
        _context = context;
    }

    public async Task<Hotel?> GetByIdAsync(
        int id,
        CancellationToken cancellationToken)
    {
        return await _context.Hotels
            .FirstOrDefaultAsync(
                h => h.Id == id,
                cancellationToken);
    }

    public async Task AddAsync(
        Hotel hotel,
        CancellationToken cancellationToken)
    {
        await _context.Hotels.AddAsync(
            hotel,
            cancellationToken);
    }

    public void Remove(Hotel hotel)
    {
        _context.Hotels.Remove(hotel);
    }
}
```

Le Repository connaît EF Core.

Le reste de l'application peut rester dépendant de l'abstraction.

---

# 5. Enregistrement dans Dependency Injection

Dans `Program.cs` :

```csharp
builder.Services.AddScoped<
    IHotelRepository,
    HotelRepository>();
```

Cela signifie :

```text
Quelqu'un demande
IHotelRepository
        ↓
DI Container
        ↓
HotelRepository
```

Le service n'a donc pas besoin de faire :

```csharp
new HotelRepository(...)
```

Il reçoit sa dépendance par injection.

---

# 6. Repository et DbContext

C'est un point important.

EF Core possède déjà des mécanismes proches du Repository et du Unit of Work :

```csharp
_context.Hotels
```

ressemble à une collection d'accès aux données.

Et :

```csharp
_context.SaveChangesAsync()
```

regroupe les changements à persister.

On peut donc se demander :

> « Pourquoi créer un Repository au-dessus d'EF Core ? »

La réponse est :

> **Seulement si cette abstraction apporte une vraie valeur au projet.**

---

# 7. Repository Pattern n'est pas obligatoire avec EF Core

Il ne faut pas appliquer automatiquement :

```text
IHotelRepository
HotelRepository

IUserRepository
UserRepository

IOrderRepository
OrderRepository

IProductRepository
ProductRepository
```

à toutes les entités.

Cela peut simplement ajouter beaucoup de code autour d'EF Core sans résoudre de problème réel.

Une application simple peut parfaitement utiliser directement :

```csharp
AppDbContext
```

dans sa couche applicative, selon son architecture.

---

# 8. Le problème du Generic Repository

On rencontre souvent :

```csharp
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(int id);

    Task AddAsync(T entity);

    void Update(T entity);

    void Delete(T entity);
}
```

Puis :

```csharp
public class Repository<T> : IRepository<T>
{
    ...
}
```

L'idée est de réutiliser le même Repository pour toutes les entités.

Cela peut sembler pratique.

Mais EF Core possède déjà :

```csharp
DbSet<T>
```

qui fournit beaucoup de ces opérations.

Donc :

```text
Generic Repository
        ↓
DbSet<T>
        ↓
EF Core
```

peut parfois devenir une abstraction inutile.

---

# 9. Le problème du Repository CRUD générique

Un Repository générique peut conduire à une interface comme :

```csharp
GetById()
GetAll()
Add()
Update()
Delete()
```

Mais les besoins réels ne sont pas toujours génériques.

Par exemple :

```text
GetAvailableHotels()
GetReservationsForCustomer()
GetOrdersPendingPayment()
FindHotelsNearLocation()
```

Ces opérations représentent des besoins spécifiques.

Un Repository métier peut alors être plus expressif :

```csharp
public interface IHotelRepository
{
    Task<IReadOnlyList<Hotel>> GetAvailableHotelsAsync(
        DateOnly start,
        DateOnly end,
        CancellationToken cancellationToken);
}
```

Le contrat décrit directement le besoin.

---

# 10. Repository orienté métier

Un Repository n'a pas besoin de reproduire toutes les méthodes de `DbSet`.

Exemple :

```csharp
public interface IReservationRepository
{
    Task<Reservation?> GetByIdAsync(
        int id,
        CancellationToken cancellationToken);

    Task<bool> HasConflictAsync(
        int hotelId,
        DateTime start,
        DateTime end,
        CancellationToken cancellationToken);

    Task AddAsync(
        Reservation reservation,
        CancellationToken cancellationToken);
}
```

Cette interface est plus proche du métier que :

```csharp
IRepository<Reservation>
```

avec uniquement :

```text
Get
Add
Update
Delete
```

---

# 11. Repository vs Service

Ils n'ont pas le même rôle.

### Repository

Responsable de l'accès aux données.

```text
Lire
Ajouter
Supprimer
Rechercher
```

### Application Service

Responsable d'orchestrer un cas d'utilisation.

Exemple :

```text
CreateReservationService
```

peut :

```text
Vérifier la disponibilité
Créer la réservation
Appliquer une règle métier
Persister
Envoyer éventuellement un événement
```

Mentalement :

```text
Repository
    = Comment accéder aux données ?

Service / Use Case
    = Quelle action l'application doit-elle réaliser ?
```

---

# 12. Repository vs DbContext

Avec EF Core :

```text
DbContext
    = session de travail avec la base

DbSet
    = accès aux entités

Repository
    = abstraction personnalisée éventuelle
```

`DbContext` gère notamment :

- le Change Tracker ;
- les requêtes ;
- la persistance ;
- les transactions selon le scénario ;
- la relation avec le provider EF Core.

Un Repository ne remplace pas automatiquement toutes les responsabilités du `DbContext`.

---

# 13. Qui appelle `SaveChangesAsync()` ?

Il existe plusieurs stratégies.

### Stratégie A — Repository sauvegarde

```csharp
public async Task AddAsync(...)
{
    await _context.Hotels.AddAsync(...);

    await _context.SaveChangesAsync();
}
```

Simple, mais cela peut compliquer les opérations impliquant plusieurs repositories.

### Stratégie B — Unit of Work / couche applicative sauvegarde

```csharp
await _repository.AddAsync(...);

await _repository.AddReservationAsync(...);

await _unitOfWork.SaveChangesAsync();
```

Cela permet de regrouper plusieurs changements dans une même unité de travail.

Avec EF Core, le `DbContext` joue déjà largement ce rôle.

---

# 14. Repository et Unit of Work

On peut avoir :

```text
Application
    ↓
Repositories
    ↓
Unit of Work
    ↓
DbContext
```

Exemple :

```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(
        CancellationToken cancellationToken);
}
```

Puis :

```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    public UnitOfWork(AppDbContext context)
    {
        _context = context;
    }

    public Task<int> SaveChangesAsync(
        CancellationToken cancellationToken)
    {
        return _context.SaveChangesAsync(
            cancellationToken);
    }
}
```

Mais encore une fois :

> EF Core fournit déjà un comportement très proche du Unit of Work avec `DbContext`.

Il ne faut donc pas ajouter cette abstraction sans raison.

---

# 15. Repository et IQueryable

Attention à une abstraction qui expose directement :

```csharp
IQueryable<Hotel>
```

Exemple :

```csharp
IQueryable<Hotel> GetHotels();
```

Cela permet à la couche appelante de construire :

```csharp
repository
    .GetHotels()
    .Where(...)
    .OrderBy(...)
    .Select(...);
```

Mais le code appelant connaît alors indirectement le fonctionnement de la requête EF/LINQ.

On risque donc de déplacer la complexité sans réellement l'isoler.

Une alternative est d'exposer une opération plus claire :

```csharp
Task<IReadOnlyList<HotelListItemDto>> SearchAsync(
    HotelSearchCriteria criteria,
    CancellationToken cancellationToken);
```

Le bon choix dépend du niveau d'abstraction réellement recherché.

---

# 16. Repository et projections

Pour les lectures, il peut être préférable de faire directement une projection :

```csharp
return await _context.Hotels
    .AsNoTracking()
    .Where(h => h.City == city)
    .Select(h => new HotelListItemDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .ToListAsync(cancellationToken);
```

Cela évite de charger une entité complète lorsque l'application n'en a pas besoin.

C'est particulièrement intéressant dans une approche CQRS.

---

# 17. Repository et CQRS

CQRS permet de séparer :

```text
Command
    ↓
Write side

Query
    ↓
Read side
```

On peut alors choisir des abstractions différentes.

Par exemple :

```text
Command
  ↓
IReservationRepository
  ↓
Domain
```

et :

```text
Query
  ↓
EF Core projection
  ↓
DTO
```

Il n'est donc pas obligatoire de forcer toutes les lectures à passer par le même Repository que les écritures.

---

# 18. Exemple avec une Command

```csharp
public class CreateReservationHandler
{
    private readonly IReservationRepository _repository;

    public CreateReservationHandler(
        IReservationRepository repository)
    {
        _repository = repository;
    }

    public async Task HandleAsync(
        CreateReservationCommand command,
        CancellationToken cancellationToken)
    {
        var reservation = Reservation.Create(
            command.HotelId,
            command.StartDate,
            command.EndDate);

        await _repository.AddAsync(
            reservation,
            cancellationToken);
    }
}
```

Le Handler orchestre.

Le Repository persiste.

Le Domain applique les règles métier.

---

# 19. Repository et tests

Une raison possible d'utiliser un Repository est de pouvoir tester l'application sans dépendre directement d'une base.

Exemple :

```text
CreateReservationHandler
          ↓
IReservationRepository
          ↓
FakeReservationRepository
```

Le test peut contrôler les données sans SQL Server.

Mais attention :

> Un faux Repository ne remplace pas les tests d'intégration avec EF Core.

Il faut également tester le vrai comportement de persistance lorsque cela est important.

---

# 20. Faux Repository vs base de test

### Test unitaire

On peut utiliser :

```text
Fake
Mock
Stub
```

pour isoler le cas d'utilisation.

### Test d'intégration

On teste réellement :

```text
Application
   ↓
EF Core
   ↓
Database
```

Le but est différent.

Mentalement :

```text
Unit test
→ Le code fonctionne-t-il ?

Integration test
→ Les composants fonctionnent-ils correctement ensemble ?
```

---

# 21. Le piège du Repository qui devient énorme

Mauvais exemple :

```csharp
public interface IHotelRepository
{
    Task<Hotel?> GetByIdAsync(...);
    Task<List<Hotel>> GetAllAsync(...);
    Task<List<Hotel>> GetByCityAsync(...);
    Task<List<Hotel>> GetByCountryAsync(...);
    Task<List<Hotel>> GetAvailableAsync(...);
    Task<List<Hotel>> GetPopularAsync(...);
    Task<List<Hotel>> GetCheapestAsync(...);
    Task<List<Hotel>> GetWithReviewsAsync(...);
    Task<List<Hotel>> GetWithRoomsAsync(...);
    ...
}
```

Le Repository devient une énorme collection de méthodes.

Cela peut être le signe que l'abstraction n'est plus bien définie.

---

# 22. Specification Pattern

Lorsque les recherches deviennent complexes, une autre approche possible est le **Specification Pattern**.

Conceptuellement :

```text
Specification
    =
règle de recherche réutilisable
```

Exemple :

```text
AvailableHotelsSpecification
```

qui décrit :

```text
Hôtel actif
+
Disponible pendant la période
+
Ville demandée
```

Puis le Repository peut utiliser cette spécification.

Mais comme pour CQRS et les autres patterns :

> Ne pas introduire Specification simplement parce qu'elle existe.

Elle devient intéressante lorsque la complexité des critères de recherche le justifie.

---

# 23. Repository et Domain

Le Repository est souvent associé à l'accès aux agrégats métier.

Exemple :

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(
        OrderId id,
        CancellationToken cancellationToken);

    Task AddAsync(
        Order order,
        CancellationToken cancellationToken);
}
```

Le Repository peut alors représenter une frontière entre :

```text
Domain
```

et :

```text
Persistence
```

---

# 24. Le Repository ne doit pas devenir un "God Object"

Il ne doit pas être responsable de :

```text
Validation métier
Envoi d'emails
Authentification
Calcul métier
Logging de toute l'application
Appels HTTP sans rapport
```

Son rôle principal reste lié à la persistance ou à la récupération des données.

---

# 25. Quand le Repository est pertinent ?

Il peut être pertinent lorsque :

- le domaine est suffisamment complexe ;
- on veut isoler la persistance ;
- les opérations de données sont métier-spécifiques ;
- plusieurs mécanismes de persistance doivent être supportés ;
- l'abstraction facilite réellement les tests ;
- les requêtes doivent être centralisées pour une raison claire.

---

# 26. Quand il peut être inutile ?

Il peut être inutile lorsque :

- l'application est un petit CRUD ;
- EF Core suffit parfaitement ;
- les repositories ne font que transférer chaque appel vers `DbSet` ;
- on ajoute beaucoup de code sans bénéfice ;
- l'abstraction cache simplement EF Core sans résoudre de problème.

Exemple :

```csharp
public Task<Hotel?> GetByIdAsync(int id)
{
    return _context.Hotels
        .FirstOrDefaultAsync(h => h.Id == id);
}
```

Si chaque méthode du Repository ressemble exactement à cela et n'apporte aucune vraie abstraction, il faut se demander pourquoi le Repository existe.

---

# 27. Repository vs directement EF Core

| Approche | Avantage | Risque |
|---|---|---|
| `DbContext` direct | Simple, puissant | Couplage à EF Core |
| Repository spécifique | Contrat clair, abstraction métier | Plus de code |
| Generic Repository | Réutilisable | Peut dupliquer `DbSet` |
| Repository + Unit of Work | Contrôle explicite | Peut dupliquer `DbContext` |
| Query directe optimisée | Très efficace pour les lectures | Moins d'abstraction |

Il n'existe donc pas une réponse universelle.

---

# 28. Une règle pratique

Avant de créer :

```text
IProductRepository
```

demande-toi :

> « Quel problème concret cette abstraction résout-elle ? »

Si la réponse est :

> « Parce que Clean Architecture dit qu'il faut un Repository. »

Ce n'est pas suffisant.

Une meilleure réponse serait :

> « Je veux isoler mes opérations métier de persistance et avoir un contrat spécifique pour les agrégats que mon application manipule. »

---

# 29. Mental model

Imagine une bibliothèque.

```text
Application
    ↓
Bibliothécaire
    ↓
Repository
    ↓
Entrepôt
```

L'application demande :

> « Donne-moi le livre X. »

Elle ne devrait pas nécessairement savoir :

```text
dans quelle étagère
dans quel entrepôt
avec quel système de stockage
```

Le Repository joue le rôle d'intermédiaire.

Mais si l'application a déjà un accès simple et parfaitement adapté à l'entrepôt, ajouter trois intermédiaires peut devenir inutile.

---

# 30. Checklist Repository

Avant d'ajouter un Repository :

- [ ] Quel problème concret résout-il ?
- [ ] L'abstraction est-elle réellement utile ?
- [ ] Est-elle orientée métier ou simplement CRUD ?
- [ ] Est-ce qu'elle duplique `DbSet` ?
- [ ] Qui est responsable de `SaveChangesAsync()` ?
- [ ] Les transactions sont-elles correctement gérées ?
- [ ] Est-ce que `IQueryable` fuit inutilement ?
- [ ] Les lectures pourraient-elles utiliser directement des projections ?
- [ ] Le Repository reste-t-il focalisé sur la persistance ?
- [ ] Les tests unitaires bénéficient-ils réellement de cette abstraction ?
- [ ] Des tests d'intégration vérifient-ils le vrai comportement EF Core ?

---

# 31. Questions d'entretien

### 1. Qu'est-ce que le Repository Pattern ?

C'est une abstraction qui encapsule l'accès aux données afin de réduire le couplage entre l'application et le mécanisme de persistance.

### 2. Est-ce obligatoire avec EF Core ?

Non. EF Core fournit déjà des fonctionnalités proches du Repository et du Unit of Work.

### 3. Pourquoi éviter un Generic Repository systématique ?

Parce qu'il peut simplement reproduire les fonctionnalités de `DbSet<T>` sans apporter une vraie valeur.

### 4. Repository et Service, quelle différence ?

Le Repository s'occupe principalement de l'accès aux données. Le Service applicatif orchestre un cas d'utilisation.

### 5. Pourquoi ne pas exposer systématiquement `IQueryable` ?

Parce que cela peut exposer les détails de construction des requêtes et faire fuiter la technologie de persistance vers la couche appelante.

### 6. Qui doit appeler `SaveChangesAsync()` ?

Cela dépend de l'architecture. Il peut être géré au niveau du Repository, d'une Unit of Work ou de la couche applicative. Avec EF Core, `DbContext` fournit déjà une unité de travail naturelle.

### 7. Un Repository remplace-t-il `DbContext` ?

Non. `DbContext` possède des responsabilités plus larges, notamment le Change Tracker et la coordination de la persistance.

---

# À retenir

```text
Repository
    =
abstraction d'accès aux données
```

Mais avec EF Core :

```text
DbSet<T>
    ≈ Repository

DbContext
    ≈ Unit of Work
```

Ce n'est pas une équivalence parfaite, mais c'est une bonne intuition pour comprendre pourquoi un Repository supplémentaire n'est pas toujours nécessaire.

La vraie question n'est pas :

> « Dois-je utiliser Repository ? »

Mais :

> « Quelle valeur cette abstraction apporte-t-elle à mon architecture ? »

# Phrase à mémoriser

> **Un Repository doit cacher une vraie complexité de persistance, pas simplement recopier les méthodes de `DbSet`.**
