# Architecture — Vue d'ensemble

## 1. Pourquoi parler d'architecture ?

Quand une application devient plus grande, le problème n'est plus seulement :

> « Est-ce que mon code fonctionne ? »

Il devient aussi :

> « Est-ce que mon code restera compréhensible, testable et modifiable dans six mois ? »

L'architecture sert à organiser les responsabilités et les dépendances de l'application.

L'objectif n'est pas de créer le plus de projets ou de dossiers possible.

L'objectif est de rendre les **responsabilités claires** et les **dépendances maîtrisées**.

---

# 2. Le problème d'une application sans architecture

Imaginons un contrôleur ASP.NET Core qui fait tout :

```csharp
[HttpPost]
public async Task<IActionResult> Create(OrderDto dto)
{
    // Validation

    // Règles métier

    // Accès à EF Core

    // Calcul du prix

    // Envoi d'un email

    // Logging

    // Sauvegarde

    // Mapping

    return Ok();
}
```

Au début, cela peut fonctionner.

Mais le contrôleur devient progressivement responsable de :

```text
HTTP
Validation
Métier
Base de données
Email
Logging
Mapping
```

On obtient alors un composant difficile à :

- comprendre ;
- tester ;
- modifier ;
- réutiliser ;
- maintenir.

---

# 3. Le principe fondamental : séparer les responsabilités

Une architecture saine cherche à répondre à une question :

> « Qui est responsable de quoi ? »

Par exemple :

```text
Controller
    ↓
Use case / Service
    ↓
Domain
    ↓
Infrastructure
```

Chaque niveau a une responsabilité différente.

### Controller

Comprend le monde HTTP :

```text
Request
Response
Route
Status code
Model binding
```

### Application

Orchestre les cas d'utilisation :

```text
CreateOrder
GetOrder
CancelOrder
```

### Domain

Contient les règles métier importantes.

### Infrastructure

Contient les détails techniques :

```text
EF Core
SQL Server
Email
File system
API externes
```

---

# 4. Architecture en couches

Une architecture classique peut être représentée ainsi :

```text
Presentation
     ↓
Application
     ↓
Domain
     ↓
Infrastructure
```

Mais il faut faire attention au sens des dépendances.

Le principe important n'est pas simplement :

```text
Projet A appelle Projet B
```

mais :

> « Qui doit connaître qui ? »

---

# 5. Pourquoi les dépendances sont importantes ?

Supposons que le domaine dépende directement d'EF Core :

```text
Domain
  ↓
EF Core
  ↓
SQL Server
```

Le cœur métier connaît alors un détail technique.

Si demain on change :

```text
SQL Server
```

pour :

```text
PostgreSQL
```

on risque de propager ce changement dans plusieurs parties de l'application.

L'objectif d'une architecture propre est de protéger le cœur métier contre les détails techniques.

---

# 6. Clean Architecture

La Clean Architecture pousse ce principe plus loin.

Une représentation simplifiée :

```text
              +----------------------+
              |    Presentation     |
              +----------+-----------+
                         |
              +----------v-----------+
              |     Application      |
              +----------+-----------+
                         |
              +----------v-----------+
              |       Domain        |
              +----------------------+

        Infrastructure dépend du cœur
        via les abstractions nécessaires
```

L'idée centrale est :

> **Les dépendances doivent pointer vers le cœur de l'application.**

Le domaine ne doit pas dépendre d'ASP.NET Core, d'EF Core ou de SQL Server simplement parce que ces technologies sont utilisées autour de lui.

---

# 7. Les quatre grandes responsabilités

Une organisation fréquente en .NET :

```text
MyApp.Domain
MyApp.Application
MyApp.Infrastructure
MyApp.API
```

### Domain

Contient le métier :

```text
Entities
Value Objects
Domain Rules
Domain Events
Interfaces métier nécessaires
```

### Application

Contient les cas d'utilisation :

```text
Commands
Queries
DTOs
Handlers
Services applicatifs
Interfaces
```

### Infrastructure

Contient les implémentations techniques :

```text
DbContext
Repositories
Email
Storage
External APIs
```

### API

Contient l'exposition HTTP :

```text
Controllers
Middleware
Authentication
Authorization
HTTP configuration
```

---

# 8. Exemple de dépendances

Une structure possible :

```text
MyApp.Domain
      ↑
MyApp.Application
      ↑
MyApp.Infrastructure
      ↑
MyApp.API
```

Attention : la flèche représente ici la direction de dépendance.

Cela signifie :

```text
API
  → Application

Infrastructure
  → Application / Domain

Application
  → Domain
```

Le domaine reste indépendant des couches externes.

---

# 9. Le principe Dependency Inversion

C'est l'un des principes SOLID.

Sans inversion de dépendance :

```text
Application
     ↓
EF Core
```

L'application dépend directement d'une technologie.

Avec une abstraction :

```text
Application
     ↓
IOrderRepository
     ↑
OrderRepository
     ↓
EF Core
```

L'application connaît :

```csharp
IOrderRepository
```

mais pas nécessairement :

```csharp
OrderRepository
```

L'implémentation concrète peut être fournie par l'Infrastructure grâce à l'injection de dépendances.

---

# 10. Exemple concret

Dans Application :

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(
        int id,
        CancellationToken cancellationToken);
}
```

Dans Infrastructure :

```csharp
public class OrderRepository : IOrderRepository
{
    private readonly AppDbContext _context;

    public OrderRepository(AppDbContext context)
    {
        _context = context;
    }

    public Task<Order?> GetByIdAsync(
        int id,
        CancellationToken cancellationToken)
    {
        return _context.Orders
            .FirstOrDefaultAsync(
                o => o.Id == id,
                cancellationToken);
    }
}
```

Puis dans DI :

```csharp
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
```

Le code applicatif dépend de :

```text
IOrderRepository
```

et le conteneur DI fournit :

```text
OrderRepository
```

---

# 11. Interface ≠ abstraction magique

Une erreur fréquente est de penser :

> « Si je mets une interface partout, mon architecture est automatiquement propre. »

Non.

Une interface doit représenter une responsabilité utile.

Mauvais exemple :

```csharp
public interface IOrderService
{
}
```

sans aucune raison.

Ou :

```text
IUserService
    ↓
UserService
```

uniquement pour ajouter une couche supplémentaire sans bénéfice.

Le but n'est pas de multiplier les abstractions.

Le but est de contrôler les dépendances et de clarifier les responsabilités.

---

# 12. Application Layer

L'Application Layer représente généralement les **cas d'utilisation**.

Exemples :

```text
CreateOrder
CancelOrder
GetOrder
UpdateCustomer
RegisterUser
```

Elle orchestre les opérations.

Exemple simplifié :

```csharp
public class CreateOrderService
{
    private readonly IOrderRepository _repository;

    public CreateOrderService(IOrderRepository repository)
    {
        _repository = repository;
    }

    public async Task CreateAsync(...)
    {
        // orchestration du cas d'utilisation
    }
}
```

L'Application Layer ne devrait pas devenir une deuxième couche de domaine contenant toutes les règles métier.

---

# 13. Domain Layer

Le Domain représente les concepts métier.

Exemple :

```csharp
public class Order
{
    public decimal Total { get; private set; }

    public void Cancel()
    {
        // règle métier
    }
}
```

Le domaine peut protéger ses invariants.

Par exemple :

```csharp
public void Cancel()
{
    if (Status == OrderStatus.Paid)
    {
        throw new InvalidOperationException(
            "A paid order cannot be cancelled.");
    }

    Status = OrderStatus.Cancelled;
}
```

La règle :

> « Une commande payée ne peut pas être annulée »

est une règle métier.

Elle appartient naturellement au domaine plutôt qu'au contrôleur HTTP.

---

# 14. Infrastructure Layer

Infrastructure contient les détails techniques.

Exemples :

```text
EF Core
SQL Server
SMTP
Azure Blob Storage
HttpClient
Redis
Message broker
```

Exemple :

```text
Application
    ↓
IEmailSender
    ↑
SmtpEmailSender
```

L'application demande :

```csharp
await _emailSender.SendAsync(...);
```

Elle n'a pas besoin de connaître le protocole SMTP.

---

# 15. Presentation Layer

Dans une API ASP.NET Core, Presentation est généralement la couche HTTP.

Exemple :

```csharp
[ApiController]
[Route("api/orders")]
public class OrdersController : ControllerBase
{
    [HttpPost]
    public async Task<IActionResult> Create(CreateOrderRequest request)
    {
        // appeler le cas d'utilisation

        return Ok();
    }
}
```

Le contrôleur doit idéalement rester mince.

Son travail est notamment de :

```text
Recevoir HTTP
    ↓
Valider / récupérer les données
    ↓
Appeler l'application
    ↓
Transformer le résultat en réponse HTTP
```

---

# 16. Pourquoi les contrôleurs doivent rester minces ?

Imagine :

```csharp
[HttpPost]
public async Task<IActionResult> Create(OrderDto dto)
{
    // 200 lignes de logique métier
}
```

Cela devient difficile à tester.

À l'inverse :

```csharp
[HttpPost]
public async Task<IActionResult> Create(OrderDto dto)
{
    var result = await _createOrder.ExecuteAsync(dto);

    return Ok(result);
}
```

Le contrôleur est alors principalement un adaptateur entre :

```text
HTTP
```

et :

```text
Application
```

---

# 17. DTOs dans l'architecture

Les DTOs permettent de contrôler ce qui entre et sort de l'API.

Exemple :

```csharp
public record CreateOrderRequest(
    int CustomerId,
    List<int> ProductIds);
```

On évite de recevoir directement une entité EF Core :

```csharp
public async Task<IActionResult> Create(Order order)
```

Pourquoi ?

Parce que l'entité représente le modèle métier/persistance, alors que le DTO représente le contrat HTTP.

Mentalement :

```text
HTTP
 ↓
DTO
 ↓
Application
 ↓
Domain
 ↓
Infrastructure
```

---

# 18. Où placer EF Core ?

EF Core est généralement considéré comme un détail d'infrastructure.

Donc :

```text
Infrastructure
    └── Data
         ├── AppDbContext
         ├── Configurations
         └── Repositories
```

L'API ne devrait pas contenir toute la logique EF Core.

Cela ne signifie pas qu'il faut absolument interdire tout accès à EF Core dans chaque projet ; le niveau d'abstraction doit rester proportionné au projet.

---

# 19. Attention au Repository Pattern

EF Core implémente déjà beaucoup de fonctionnalités de :

```text
Repository
Unit of Work
```

Par exemple :

```csharp
_context.Products
```

ressemble déjà à une collection d'accès aux données.

Et :

```csharp
_context.SaveChangesAsync()
```

regroupe les changements à persister.

Il ne faut donc pas créer automatiquement :

```text
IProductRepository
ProductRepository
IProductUnitOfWork
ProductUnitOfWork
```

pour chaque entité sans besoin réel.

Le Repository Pattern peut être utile lorsqu'il apporte une vraie abstraction métier ou technique, mais ce n'est pas une obligation.

---

# 20. Architecture ≠ nombre de projets

Une mauvaise architecture peut avoir :

```text
20 projets
```

et rester difficile à maintenir.

Une bonne architecture peut parfois tenir dans :

```text
1 projet
```

ou quelques projets bien structurés.

Le nombre de projets n'est pas la mesure de la qualité.

La vraie question est :

> « Les responsabilités et les dépendances sont-elles cohérentes ? »

---

# 21. Architecture et testabilité

Une architecture avec des responsabilités séparées facilite les tests.

Exemple :

```text
CreateOrderHandler
       ↓
IOrderRepository
```

Pendant un test, on peut fournir une implémentation simulée.

Conceptuellement :

```text
Application
    ↓
Mock IOrderRepository
```

au lieu de :

```text
Application
    ↓
SQL Server réel
```

Cela permet de tester le comportement sans dépendre systématiquement de la base.

---

# 22. Architecture et changement de technologie

Un bon découpage réduit le coût de certains changements.

Exemple :

```text
Application
      ↓
IEmailSender
      ↑
      |
SmtpEmailSender
```

Demain :

```text
AzureEmailSender
```

peut remplacer :

```text
SmtpEmailSender
```

sans réécrire le cas d'utilisation.

Attention :

> Une abstraction ne rend pas automatiquement un changement gratuit.

Elle réduit surtout le couplage lorsqu'elle correspond à une vraie frontière.

---

# 23. Architecture et Dependency Injection

L'injection de dépendances permet de connecter les couches.

Exemple :

```csharp
builder.Services.AddScoped<
    IOrderRepository,
    OrderRepository>();
```

Le code applicatif demande :

```csharp
IOrderRepository
```

Le conteneur DI fournit :

```text
OrderRepository
```

On obtient :

```text
Abstraction
     ↑
Implementation
```

sans que l'Application Layer instancie elle-même l'Infrastructure.

---

# 24. Architecture et middleware

Le middleware appartient généralement à la couche HTTP/ASP.NET Core.

Exemples :

```text
Exception handling
Authentication
Authorization
Logging
Correlation ID
Rate limiting
```

Le pipeline peut être vu ainsi :

```text
Request
  ↓
Middleware
  ↓
Routing
  ↓
Authentication
  ↓
Authorization
  ↓
Controller
  ↓
Application
  ↓
Domain / Infrastructure
```

Le middleware est donc principalement un mécanisme transversal autour du traitement HTTP.

---

# 25. Architecture et sécurité

La séparation des responsabilités aide aussi à la sécurité.

Par exemple :

```text
Authentication
    → Qui es-tu ?

Authorization
    → As-tu le droit ?

Application
    → Que doit faire le cas d'utilisation ?

Domain
    → Quelles règles métier doivent toujours être respectées ?
```

Il ne faut pas considérer l'autorisation HTTP comme le seul endroit où une règle métier doit être protégée.

---

# 26. Les erreurs classiques d'architecture

### Erreur 1 — mettre toute la logique dans les controllers

```text
Controller = tout faire
```

Résultat :

```text
Controller énorme
```

---

### Erreur 2 — transformer Application en simple wrapper EF Core

Exemple :

```csharp
public Task<Product?> GetProduct(int id)
{
    return _context.Products.FindAsync(id).AsTask();
}
```

Si chaque service ne fait que transmettre des appels EF Core sans réelle responsabilité, l'architecture peut devenir artificiellement complexe.

---

### Erreur 3 — utiliser des interfaces partout

```text
IService
IRepository
IFactory
IManager
IHelper
```

sans justification.

Une abstraction doit résoudre un problème.

---

### Erreur 4 — faire dépendre le Domain d'ASP.NET Core

Le domaine ne devrait pas connaître :

```text
ControllerBase
HttpContext
IActionResult
ModelState
```

Ces éléments appartiennent au monde HTTP.

---

### Erreur 5 — faire dépendre le Domain d'EF Core sans nécessité

Le domaine ne devrait pas devenir une extension de `DbContext`.

---

### Erreur 6 — confondre architecture et dossiers

Avoir :

```text
Services/
Repositories/
Models/
Helpers/
Utils/
```

ne garantit pas une architecture propre.

---

# 27. Comment raisonner devant un nouveau projet ?

Avant de créer des dossiers, pose-toi ces questions :

### Question 1

Quel est le métier de l'application ?

```text
Hotel booking
E-commerce
Task management
```

### Question 2

Quelles sont les règles métier importantes ?

```text
Une réservation ne peut pas être double.
Une commande payée ne peut pas être annulée.
```

### Question 3

Quels sont les détails techniques ?

```text
SQL Server
EF Core
SMTP
Azure
Redis
```

### Question 4

Quelles dépendances doivent être inversées ?

```text
Application
    ↓
Interface

Infrastructure
    ↓
Implementation
```

### Question 5

Quelle complexité est réellement justifiée ?

Ne pas appliquer Clean Architecture comme une recette mécanique.

---

# 28. Petit exemple complet

Imaginons :

```text
HotelListing
```

Une organisation possible :

```text
HotelListing.Domain
├── Entities
│   └── Hotel.cs
└── Enums

HotelListing.Application
├── DTOs
├── Interfaces
└── Services

HotelListing.Infrastructure
├── Data
│   ├── AppDbContext.cs
│   └── Configurations
└── Repositories

HotelListing.API
├── Controllers
├── Middleware
├── Program.cs
└── DTOs
```

Flux :

```text
POST /api/hotels
        ↓
HotelsController
        ↓
CreateHotelService
        ↓
Hotel
        ↓
IHotelRepository
        ↑
HotelRepository
        ↓
AppDbContext
        ↓
SQL Server
```

Chaque partie joue un rôle différent.

---

# 29. Le bon niveau d'architecture

Il existe un piège important :

> **Overengineering**

On peut construire une architecture extrêmement sophistiquée pour une application très simple.

Par exemple, pour un petit CRUD :

```text
Controller
Application Service
Command
CommandHandler
Repository
UnitOfWork
Factory
Domain Service
Mapper
Mediator
...
```

Cela peut parfois ajouter plus de complexité que de valeur.

La bonne architecture est celle qui :

- protège les règles importantes ;
- limite le couplage ;
- facilite les tests ;
- facilite les changements ;
- reste compréhensible.

---

# 30. Mental model à retenir

Pense à l'application comme à une entreprise.

```text
API
=
Accueil

Application
=
Employés qui exécutent les demandes

Domain
=
Règles de l'entreprise

Infrastructure
=
Fournisseurs et outils techniques
```

Exemple :

```text
Client HTTP
   ↓
Accueil
   ↓
Cas d'utilisation
   ↓
Règles métier
   ↓
Outils techniques
   ↓
Base de données
```

Cela permet de comprendre rapidement le rôle de chaque couche.

---

# 31. Checklist architecture

Avant de considérer une architecture comme saine :

- [ ] Les responsabilités sont clairement séparées.
- [ ] Le domaine ne dépend pas du HTTP.
- [ ] Le domaine ne dépend pas inutilement d'EF Core.
- [ ] Les contrôleurs restent relativement minces.
- [ ] Les DTOs séparent les contrats HTTP des entités.
- [ ] Les détails techniques restent dans Infrastructure lorsque cela est pertinent.
- [ ] Les abstractions sont utilisées lorsqu'elles apportent une vraie valeur.
- [ ] Les dépendances vont dans la bonne direction.
- [ ] L'application est testable.
- [ ] Les règles métier importantes sont protégées par le domaine.
- [ ] L'architecture n'est pas plus complexe que nécessaire.

---

# 32. Questions d'entretien

### 1. Qu'est-ce que la Clean Architecture ?

C'est une approche architecturale qui cherche notamment à isoler le cœur métier des détails techniques et à contrôler la direction des dépendances.

### 2. Pourquoi utiliser Dependency Inversion ?

Pour éviter que le cœur de l'application dépende directement des implémentations techniques et pour permettre de remplacer plus facilement certaines implémentations.

### 3. Pourquoi un controller doit-il rester mince ?

Parce qu'il doit principalement adapter HTTP vers les cas d'utilisation. La logique métier importante doit être placée ailleurs.

### 4. Quelle est la différence entre Domain et Infrastructure ?

Le Domain contient les règles et concepts métier. Infrastructure contient les détails techniques nécessaires pour communiquer avec l'extérieur.

### 5. Faut-il toujours utiliser Repository Pattern avec EF Core ?

Non. EF Core fournit déjà des mécanismes proches du Repository et du Unit of Work. Il faut ajouter une abstraction uniquement lorsqu'elle apporte une vraie valeur.

### 6. Une architecture avec beaucoup de projets est-elle forcément meilleure ?

Non. La qualité dépend de la séparation des responsabilités et de la direction des dépendances, pas du nombre de projets.

---

# À retenir

```text
Domain
  = règles métier

Application
  = cas d'utilisation

Infrastructure
  = détails techniques

API / Presentation
  = exposition HTTP
```

Et surtout :

```text
Le cœur de l'application ne doit pas être prisonnier
des détails techniques.
```

# Phrase à mémoriser

> **Une bonne architecture protège le métier, contrôle les dépendances et évite de mélanger les responsabilités.**
