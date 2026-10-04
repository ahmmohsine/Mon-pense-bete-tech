# Architecture — Clean Architecture

## 1. Qu'est-ce que la Clean Architecture ?

La **Clean Architecture** est une manière d'organiser une application pour protéger son cœur métier des détails techniques.

L'idée centrale est simple :

> **Le métier ne doit pas dépendre des détails techniques.**

Par détails techniques, on entend notamment :

```text
ASP.NET Core
EF Core
SQL Server
Azure
SMTP
Redis
API externes
```

Le métier, lui, représente :

```text
règles
entités
cas métier
invariants
```

---

# 2. Le principe des cercles

La représentation classique utilise plusieurs cercles concentriques :

```text
        +--------------------------------+
        |      Frameworks & Drivers      |
        |                                |
        |   +------------------------+   |
        |   |     Interface Adapters |   |
        |   |                        |   |
        |   |  +------------------+  |   |
        |   |  |   Application    |  |   |
        |   |  |                  |  |   |
        |   |  | +--------------+ |  |   |
        |   |  | |    Domain    | |  |   |
        |   |  | +--------------+ |  |   |
        |   |  +------------------+  |   |
        |   +------------------------+   |
        +--------------------------------+
```

Plus on va vers le centre :

```text
plus le code représente le métier
```

Plus on va vers l'extérieur :

```text
plus on trouve des détails techniques
```

---

# 3. La règle des dépendances

C'est le point le plus important.

La **Dependency Rule** peut être résumée ainsi :

> **Les dépendances du code source doivent pointer vers l'intérieur.**

Donc :

```text
Infrastructure
      ↓
Application
      ↓
Domain
```

est cohérent.

Mais :

```text
Domain
   ↓
Infrastructure
```

est contraire à l'objectif de la Clean Architecture.

Pourquoi ?

Parce que le domaine deviendrait dépendant d'un détail technique.

---

# 4. Exemple simple

Imagine une application de réservation d'hôtel.

Le métier dit :

> Une chambre ne peut pas être réservée si elle est déjà réservée pour la période demandée.

Cette règle appartient au domaine.

Elle ne dépend pas de :

```text
SQL Server
EF Core
ASP.NET Core
Azure
```

La règle pourrait être utilisée :

```text
API
Web App
Console
Tests
Background Worker
```

Le métier reste donc indépendant du moyen utilisé pour l'appeler.

---

# 5. Les couches dans un projet .NET

Une organisation fréquente est :

```text
HotelListing.Domain
HotelListing.Application
HotelListing.Infrastructure
HotelListing.API
```

On peut les représenter ainsi :

```text
        API
         ↓
   Application
         ↓
      Domain

Infrastructure
      ↓
Application / Domain
```

L'idée est que l'Infrastructure fournit des implémentations aux abstractions attendues par le cœur.

---

# 6. Domain

Le projet Domain contient le cœur métier.

On peut y trouver :

```text
Entities
Value Objects
Enums
Domain Services
Domain Events
Business Rules
```

Exemple :

```csharp
public class Reservation
{
    public DateTime StartDate { get; private set; }

    public DateTime EndDate { get; private set; }

    public void Cancel()
    {
        // règle métier
    }
}
```

Le domaine ne devrait pas avoir besoin de :

```csharp
ControllerBase
HttpContext
IActionResult
DbContext
```

pour exprimer cette règle.

---

# 7. Application

Application contient généralement les **cas d'utilisation**.

Exemples :

```text
CreateReservation
CancelReservation
GetReservation
UpdateHotel
RegisterUser
```

Elle orchestre les opérations nécessaires.

Exemple :

```csharp
public class CreateReservationService
{
    private readonly IReservationRepository _repository;

    public CreateReservationService(
        IReservationRepository repository)
    {
        _repository = repository;
    }

    public async Task CreateAsync(
        Reservation reservation,
        CancellationToken cancellationToken)
    {
        await _repository.AddAsync(
            reservation,
            cancellationToken);
    }
}
```

L'Application sait **quoi faire** pour réaliser un cas d'utilisation.

---

# 8. Infrastructure

Infrastructure contient les détails techniques.

Exemples :

```text
EF Core
SQL Server
SMTP
Azure Blob Storage
Redis
HttpClient
API externes
```

Exemple :

```csharp
public class ReservationRepository
    : IReservationRepository
{
    private readonly AppDbContext _context;

    public ReservationRepository(
        AppDbContext context)
    {
        _context = context;
    }

    public async Task AddAsync(
        Reservation reservation,
        CancellationToken cancellationToken)
    {
        _context.Reservations.Add(reservation);

        await _context.SaveChangesAsync(
            cancellationToken);
    }
}
```

Le repository concret connaît EF Core.

L'Application, elle, peut ne connaître que :

```csharp
IReservationRepository
```

---

# 9. API / Presentation

API représente le monde extérieur HTTP.

Elle contient généralement :

```text
Controllers
Middleware
Authentication
Authorization
Program.cs
HTTP configuration
```

Exemple :

```csharp
[HttpPost]
public async Task<IActionResult> Create(
    CreateReservationRequest request,
    CancellationToken cancellationToken)
{
    await _service.CreateAsync(
        request,
        cancellationToken);

    return Ok();
}
```

Le contrôleur ne devrait pas contenir toute la logique métier.

Il adapte :

```text
HTTP
 ↓
Application
```

---

# 10. Le problème des dépendances

Supposons :

```text
Application
   ↓
IReservationRepository
```

L'interface peut être définie dans Application :

```csharp
public interface IReservationRepository
{
    Task AddAsync(
        Reservation reservation,
        CancellationToken cancellationToken);
}
```

Puis Infrastructure l'implémente :

```csharp
public class ReservationRepository
    : IReservationRepository
{
    ...
}
```

La dépendance devient :

```text
Application
    ↑
Infrastructure
```

L'Application définit le contrat.

Infrastructure fournit l'implémentation.

---

# 11. Dependency Inversion

C'est ici que le principe de **Dependency Inversion** devient concret.

Sans inversion :

```text
Application
     ↓
EF Core
```

Avec inversion :

```text
Application
     ↓
IReservationRepository
     ↑
ReservationRepository
     ↓
EF Core
```

Le code métier ne dépend donc plus directement de la technologie de persistance.

---

# 12. Le rôle de l'injection de dépendances

L'interface ne crée pas l'implémentation.

C'est le conteneur DI qui fait le lien.

Dans `Program.cs` :

```csharp
builder.Services.AddScoped<
    IReservationRepository,
    ReservationRepository>();
```

Cela signifie :

> « Lorsque quelqu'un demande `IReservationRepository`, fournis `ReservationRepository`. »

Ainsi :

```text
Application
    demande
IReservationRepository
        ↓
DI Container
        ↓
ReservationRepository
```

---

# 13. Le piège : croire que Clean Architecture signifie Repository partout

Non.

La Clean Architecture ne dit pas :

> « Chaque entité doit avoir son Repository. »

Elle dit principalement :

> « Les dépendances vers les détails doivent être contrôlées. »

Avec EF Core, certaines applications peuvent fonctionner très bien sans créer un repository par entité.

Exemple :

```csharp
public class CreateReservationService
{
    private readonly AppDbContext _context;

    ...
}
```

Cela peut être acceptable selon l'architecture et le niveau de complexité du projet.

Le Repository Pattern doit être utilisé lorsqu'il apporte une vraie valeur.

---

# 14. Domain Service

Certaines règles métier ne correspondent naturellement à aucune entité.

On peut alors utiliser un **Domain Service**.

Exemple conceptuel :

```text
PricingService
```

qui calcule un prix selon plusieurs concepts métier.

Mais il ne faut pas transformer tous les comportements en services.

Si une règle appartient naturellement à une entité :

```csharp
reservation.Cancel();
```

est souvent préférable à :

```csharp
reservationService.Cancel(reservation);
```

La question à se poser est :

> « À quel objet cette responsabilité appartient-elle naturellement ? »

---

# 15. Entity vs DTO

Une Clean Architecture distingue généralement :

```text
Entity
```

et :

```text
DTO
```

Une Entity représente le métier.

Un DTO représente un contrat de communication.

Exemple :

```csharp
public class Hotel
{
    public int Id { get; private set; }

    public string Name { get; private set; }
        = null!;
}
```

DTO :

```csharp
public record HotelResponse(
    int Id,
    string Name);
```

Le DTO peut évoluer en fonction de l'API sans modifier nécessairement le modèle métier.

---

# 16. Pourquoi ne pas exposer directement les Entities ?

Supposons :

```csharp
[HttpGet]
public async Task<Hotel> GetHotel(int id)
{
    ...
}
```

Cela peut créer un couplage entre :

```text
API
```

et :

```text
Domain
```

Cela peut aussi exposer accidentellement :

- des propriétés internes ;
- des données sensibles ;
- des relations EF Core ;
- des détails de persistance.

Un DTO permet de contrôler explicitement le contrat :

```text
Entity
   ↓
Mapping
   ↓
DTO
   ↓
HTTP response
```

---

# 17. Où mettre le mapping ?

Cela dépend de l'architecture.

On peut avoir :

```text
Application
    └── Mapping
```

ou utiliser un outil comme :

```text
AutoMapper
Mapster
```

Mais le mapping manuel reste parfaitement valable :

```csharp
var response = new HotelResponse(
    hotel.Id,
    hotel.Name);
```

Il ne faut pas ajouter un outil uniquement parce qu'un outil existe.

---

# 18. Clean Architecture et EF Core

Une question fréquente :

> « Si mon Domain doit être indépendant d'EF Core, comment EF Core connaît-il mes entités ? »

Infrastructure peut configurer EF Core.

Exemple :

```csharp
public class HotelConfiguration
    : IEntityTypeConfiguration<Hotel>
{
    public void Configure(
        EntityTypeBuilder<Hotel> builder)
    {
        builder.HasKey(h => h.Id);

        builder.Property(h => h.Name)
            .IsRequired()
            .HasMaxLength(200);
    }
}
```

Puis :

```csharp
modelBuilder.ApplyConfigurationsFromAssembly(
    typeof(AppDbContext).Assembly);
```

Ainsi, les détails de persistance peuvent rester dans Infrastructure.

---

# 19. Exemple de structure détaillée

```text
HotelListing.Domain
├── Entities
│   ├── Hotel.cs
│   └── Reservation.cs
├── ValueObjects
├── Enums
└── Exceptions

HotelListing.Application
├── DTOs
├── Interfaces
├── Services
├── Commands
└── Queries

HotelListing.Infrastructure
├── Data
│   ├── AppDbContext.cs
│   └── Configurations
├── Repositories
└── Services

HotelListing.API
├── Controllers
├── Middleware
├── Extensions
└── Program.cs
```

Ce n'est qu'un exemple.

Il faut adapter la structure au projet réel.

---

# 20. Flux d'une requête

Imaginons :

```http
POST /api/reservations
```

Le flux peut être :

```text
HTTP Request
     ↓
Controller
     ↓
DTO
     ↓
Application Use Case
     ↓
Domain
     ↓
Repository abstraction
     ↓
Infrastructure
     ↓
EF Core
     ↓
SQL Server
```

Puis retour :

```text
SQL Server
     ↓
EF Core
     ↓
Infrastructure
     ↓
Application
     ↓
DTO
     ↓
Controller
     ↓
HTTP Response
```

---

# 21. Ce que la Clean Architecture protège réellement

Elle cherche notamment à protéger :

### Le métier

Contre :

```text
EF Core
ASP.NET Core
SQL Server
Azure
```

### L'application

Contre un couplage excessif aux détails d'infrastructure.

### Les tests

En permettant de remplacer certaines dépendances.

### L'évolution

En facilitant certains changements techniques.

---

# 22. Ce que Clean Architecture ne garantit pas

Elle ne garantit pas :

```text
Code automatiquement propre
Performance
Sécurité automatique
Absence de bugs
Bon design métier
```

Une mauvaise architecture peut être écrite avec quatre couches parfaitement nommées.

Exemple :

```text
Controller
    ↓
Service
    ↓
Repository
    ↓
DbContext
```

Ce n'est pas automatiquement de la Clean Architecture.

La vraie question est :

> « Les responsabilités et les dépendances sont-elles correctement organisées ? »

---

# 23. Clean Architecture vs Architecture en couches classique

### Architecture en couches classique

```text
UI
 ↓
Business
 ↓
Data Access
 ↓
Database
```

Les couches dépendent souvent directement de celles situées en dessous.

### Clean Architecture

```text
        Domain
       ↑      ↑
Application   |
       ↑      |
Infrastructure
       ↑
      API
```

L'objectif est d'inverser certaines dépendances afin que le cœur reste indépendant des détails.

---

# 24. Clean Architecture et CQRS

Clean Architecture et CQRS sont deux concepts différents.

Clean Architecture :

```text
Comment organiser les responsabilités
et les dépendances ?
```

CQRS :

```text
Comment séparer les opérations de lecture
et de modification ?
```

On peut donc avoir :

```text
Clean Architecture
       +
      CQRS
```

mais CQRS n'est pas obligatoire pour utiliser Clean Architecture.

---

# 25. Clean Architecture et MediatR

Même principe.

MediatR est un outil permettant notamment d'implémenter une approche basée sur :

```text
Commands
Queries
Handlers
```

On peut avoir :

```text
API
 ↓
MediatR
 ↓
CommandHandler
 ↓
Domain
```

Mais :

> Clean Architecture ≠ MediatR.

MediatR est un outil/une technique.

Clean Architecture est une approche d'organisation et de dépendances.

---

# 26. Quand utiliser Clean Architecture ?

Elle devient particulièrement intéressante lorsque l'application possède :

- beaucoup de règles métier ;
- plusieurs cas d'utilisation ;
- plusieurs intégrations externes ;
- une durée de vie importante ;
- plusieurs développeurs ;
- un besoin important de testabilité ;
- un domaine complexe.

Pour un petit CRUD extrêmement simple, elle peut être disproportionnée.

---

# 27. Le piège de l'overengineering

Imagine :

```text
Todo API
```

avec seulement :

```text
Create
Read
Update
Delete
```

Construire immédiatement :

```text
Domain
Application
Infrastructure
API
CQRS
MediatR
Repository
UnitOfWork
Factory
Domain Events
Specifications
...
```

peut rendre le projet plus difficile à comprendre.

Une architecture doit répondre à la complexité réelle du système.

---

# 28. Comment reconnaître une bonne frontière ?

Une bonne frontière possède généralement :

```text
Responsabilité claire
        +
Faible couplage
        +
Contrat explicite
        +
Possibilité de tester
```

Exemple :

```text
Application
    ↓
IEmailSender
```

L'Application dit :

> « J'ai besoin d'envoyer un email. »

Elle ne dit pas :

> « Utilise SMTP avec telle librairie et tel serveur. »

Cette information appartient à Infrastructure.

---

# 29. Mental model

Imagine un restaurant.

```text
Client
  ↓
Serveur
  ↓
Cuisine
  ↓
Recette
  ↓
Fournisseurs
```

Dans cette analogie :

```text
API
    = serveur

Application
    = organisation de la commande

Domain
    = recettes et règles métier

Infrastructure
    = fournisseurs et outils techniques
```

Le restaurant ne change pas ses recettes simplement parce qu'il change de fournisseur de tomates.

De la même manière :

> Le métier ne devrait pas être réécrit simplement parce qu'on change de base de données.

---

# 30. Checklist Clean Architecture

Avant de valider une architecture :

- [ ] Le Domain est indépendant des détails techniques.
- [ ] Les règles métier importantes sont dans le bon endroit.
- [ ] Les controllers ne contiennent pas toute la logique métier.
- [ ] Les DTOs représentent les contrats externes.
- [ ] Les dépendances vont vers le cœur.
- [ ] Infrastructure implémente les détails techniques.
- [ ] Les abstractions sont justifiées.
- [ ] EF Core n'envahit pas inutilement le domaine.
- [ ] L'architecture reste compréhensible.
- [ ] La complexité de l'architecture est proportionnelle au projet.

---

# 31. Questions d'entretien

### 1. Quelle est la règle principale de la Clean Architecture ?

Les dépendances du code source doivent pointer vers l'intérieur, vers les couches plus proches du cœur métier.

### 2. Pourquoi le Domain ne devrait-il pas dépendre d'EF Core ?

Parce qu'EF Core est un détail de persistance. Le métier doit rester indépendant de la technologie utilisée pour stocker les données.

### 3. Où placer `DbContext` ?

Généralement dans Infrastructure, car il représente un détail de persistance.

### 4. Pourquoi utiliser une interface de repository ?

Lorsqu'elle apporte une vraie abstraction utile et permet notamment au cœur de l'application de ne pas dépendre directement de l'implémentation de persistance.

### 5. Clean Architecture impose-t-elle CQRS ?

Non. CQRS est une autre approche qui peut être combinée avec Clean Architecture.

### 6. Clean Architecture impose-t-elle MediatR ?

Non. MediatR est un outil qui peut être utilisé pour implémenter certains patterns, mais il n'est pas obligatoire.

### 7. Pourquoi éviter les controllers trop gros ?

Parce qu'ils mélangent HTTP, orchestration, logique métier et parfois persistance, ce qui augmente le couplage et rend le code plus difficile à tester.

---

# À retenir

```text
Domain
    = métier

Application
    = cas d'utilisation

Infrastructure
    = détails techniques

API
    = monde extérieur
```

La règle centrale :

```text
Les détails dépendent du cœur.
Le cœur ne dépend pas des détails.
```

# Phrase à mémoriser

> **Clean Architecture = protéger le métier en faisant pointer les dépendances vers le cœur.**
