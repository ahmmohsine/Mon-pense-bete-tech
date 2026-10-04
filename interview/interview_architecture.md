# Entretien — Architecture .NET

Cette fiche rassemble les concepts d'architecture importants pour un entretien de développeuse C#/.NET.

L'objectif n'est pas de réciter des noms de patterns.

En entretien, il faut surtout savoir expliquer :

```text
Quel problème cherche-t-on à résoudre ?
        ↓
Quelle séparation met-on en place ?
        ↓
Quel est le bénéfice ?
        ↓
Quel est le coût / compromis ?
```

---

# 1. Pourquoi parler d'architecture ?

L'architecture définit notamment :

```text
où placer le code
qui dépend de qui
comment séparer les responsabilités
comment faire évoluer l'application
comment tester le système
```

Une architecture correcte cherche notamment à limiter :

```text
couplage excessif
responsabilités mélangées
dépendances difficiles à remplacer
logique métier dispersée
```

---

# 2. Couplage

Le couplage représente le niveau de dépendance entre deux composants.

Exemple fortement couplé :

```csharp
public class OrderService
{
    private readonly SqlPaymentService _payment;

    public OrderService()
    {
        _payment = new SqlPaymentService();
    }
}
```

`OrderService` connaît directement l'implémentation.

On préfère souvent :

```csharp
public class OrderService
{
    private readonly IPaymentService _payment;

    public OrderService(IPaymentService payment)
    {
        _payment = payment;
    }
}
```

Mental model :

```text
fort couplage
→ difficile à remplacer

faible couplage
→ plus facile à faire évoluer
```

---

# 3. Cohésion

La cohésion décrit à quel point les responsabilités d'un composant sont liées entre elles.

Une classe :

```text
UserService
```

qui contient :

```text
création utilisateur
envoi email
génération PDF
connexion database
paiement
logging
```

a probablement une cohésion faible.

Une classe avec une responsabilité clairement délimitée a généralement une meilleure cohésion.

Mental model :

```text
Cohésion
→ ce qui appartient ensemble

Couplage
→ ce qui dépend de quoi
```

Une bonne architecture cherche généralement :

```text
forte cohésion
+
faible couplage
```

---

# 4. SOLID

SOLID regroupe cinq principes :

```text
S
Single Responsibility Principle

O
Open/Closed Principle

L
Liskov Substitution Principle

I
Interface Segregation Principle

D
Dependency Inversion Principle
```

Ils ne sont pas des règles absolues.

Ils servent à guider la conception.

---

# 5. Single Responsibility Principle

SRP signifie :

> Une classe devrait avoir une responsabilité clairement définie et une seule raison principale de changer.

Mauvais exemple :

```csharp
public class UserService
{
    public void CreateUser()
    {
    }

    public void SendEmail()
    {
    }

    public void GeneratePdf()
    {
    }
}
```

Plusieurs responsabilités sont mélangées.

On peut séparer :

```text
UserService
EmailService
PdfService
```

Attention :

SRP ne signifie pas :

```text
"une classe = une méthode"
```

Il s'agit de responsabilité et de raison de changement.

---

# 6. Open/Closed Principle

Une entité logicielle devrait être :

```text
ouverte à l'extension
fermée à la modification
```

Exemple conceptuel :

```csharp
public interface IDiscount
{
    decimal Calculate(decimal amount);
}
```

Puis :

```csharp
public class RegularDiscount : IDiscount
{
    ...
}

public class VipDiscount : IDiscount
{
    ...
}
```

On peut ajouter un nouveau type de réduction sans modifier toute la logique existante.

L'objectif est de limiter les modifications risquées dans du code déjà stable.

---

# 7. Liskov Substitution Principle

Une implémentation dérivée doit pouvoir être utilisée là où son abstraction est attendue sans casser les garanties du programme.

Mental model :

```text
Base abstraction
      ↓
Derived implementation
      ↓
le comportement attendu reste valide
```

Un exemple classique de violation est une hiérarchie où une sous-classe doit systématiquement refuser les opérations promises par la classe de base.

La question importante en entretien :

> "Est-ce que cette sous-classe respecte réellement le contrat de son parent ?"

---

# 8. Interface Segregation Principle

Il vaut mieux plusieurs petites interfaces ciblées qu'une énorme interface imposant des méthodes inutiles.

Mauvais :

```csharp
public interface IWorker
{
    void Work();
    void Print();
    void Scan();
    void Fax();
}
```

Un appareil qui ne sait pas scanner est obligé de gérer :

```text
Scan()
```

On peut séparer :

```csharp
public interface IPrinter
{
    void Print();
}

public interface IScanner
{
    void Scan();
}
```

Mental model :

```text
petits contrats spécialisés
→ moins de dépendances inutiles
```

---

# 9. Dependency Inversion Principle

Le DIP est particulièrement important en .NET.

Principe :

> Les modules de haut niveau ne doivent pas dépendre directement des détails de bas niveau. Les deux doivent dépendre d'abstractions.

Mauvais :

```csharp
public class OrderService
{
    private readonly SqlPaymentService _payment;
}
```

Meilleur :

```csharp
public class OrderService
{
    private readonly IPaymentService _payment;

    public OrderService(IPaymentService payment)
    {
        _payment = payment;
    }
}
```

Puis :

```csharp
builder.Services.AddScoped<
    IPaymentService,
    SqlPaymentService>();
```

Mental model :

```text
Application
    ↓
Interface
    ↑
Infrastructure
```

---

# 10. Dependency Inversion vs Dependency Injection

Ces deux concepts sont liés mais différents.

### Dependency Inversion

C'est un principe de conception.

Il dit notamment :

```text
dépendre d'abstractions
plutôt que de détails concrets
```

### Dependency Injection

C'est une technique permettant de fournir les dépendances à une classe.

Exemple :

```csharp
public UserService(IEmailService emailService)
```

La DI permet donc de mettre en pratique certains principes de conception, notamment le DIP.

---

# 11. Clean Architecture

Clean Architecture cherche à séparer les responsabilités et à contrôler la direction des dépendances.

Une représentation classique :

```text
        API
         ↓
    Application
         ↓
      Domain

Infrastructure
      ↑
      |
Application
```

La règle essentielle est :

```text
les détails externes ne doivent pas dicter les règles métier centrales
```

---

# 12. Domain

Le Domain contient les concepts métier centraux.

Exemples :

```text
Entities
Value Objects
Domain rules
Domain services
Domain events
```

Il devrait être aussi indépendant que possible de :

```text
HTTP
EF Core
SQL Server
ASP.NET Core
UI
```

Mental model :

```text
Domain
→ métier
```

---

# 13. Application

La couche Application représente généralement les cas d'utilisation.

Exemples :

```text
CreateUser
GetUser
UpdateOrder
CancelReservation
```

Elle orchestre le travail nécessaire pour réaliser un scénario applicatif.

Elle peut dépendre du Domain et d'abstractions nécessaires.

---

# 14. Infrastructure

Infrastructure contient les détails techniques.

Exemples :

```text
EF Core
SQL Server
Email provider
File storage
External APIs
Azure services
```

Mental model :

```text
Domain
→ règles métier

Application
→ cas d'utilisation

Infrastructure
→ détails techniques

API
→ transport HTTP
```

---

# 15. API / Presentation

La couche API gère généralement :

```text
HTTP
routing
authentication
authorization
DTO HTTP
status codes
```

Elle ne devrait pas contenir toute la logique métier.

Exemple :

```text
Controller
    ↓
Application Service
    ↓
Domain
    ↓
Infrastructure abstraction
```

---

# 16. Direction des dépendances

C'est un point essentiel.

Mauvais modèle :

```text
Domain
 ↓
EF Core
 ↓
SQL
```

Le métier devient dépendant de la technologie.

Architecture plus propre :

```text
Domain
 ↑
Application
 ↑
API

Infrastructure
 ↑
implémente les abstractions
```

L'idée n'est pas simplement de créer beaucoup de projets.

Le point important est :

```text
qui connaît qui ?
```

---

# 17. Projet typique Clean Architecture

Une solution peut ressembler à :

```text
MyApp.sln

src/
├── MyApp.Api
├── MyApp.Application
├── MyApp.Domain
└── MyApp.Infrastructure

tests/
├── MyApp.UnitTests
└── MyApp.IntegrationTests
```

Exemple de dépendances :

```text
Api
 ↓
Application
 ↓
Domain

Infrastructure
 ↓
Application / Domain
```

La direction exacte peut varier selon l'implémentation, mais le principe reste de protéger le cœur métier.

---

# 18. Pourquoi séparer en projets ?

Avantages :

```text
responsabilités claires
dépendances contrôlées
testabilité
maintenabilité
possibilité de remplacer une infrastructure
```

Mais cela apporte aussi un coût :

```text
plus de projets
plus de configuration
plus d'abstractions
plus de code
```

Une architecture n'est donc pas meilleure simplement parce qu'elle possède davantage de couches.

---

# 19. Overengineering

Une erreur fréquente consiste à appliquer tous les patterns partout.

Exemple :

```text
Controller
 ↓
Service
 ↓
Manager
 ↓
Handler
 ↓
Repository
 ↓
GenericRepository
 ↓
UnitOfWork
 ↓
DbContext
```

alors que l'application est minuscule.

Le résultat peut être une architecture difficile à comprendre sans bénéfice réel.

En entretien, une bonne réponse est :

> Je choisis l'abstraction en fonction de la complexité et des besoins du projet, plutôt que d'appliquer systématiquement un pattern.

---

# 20. CQRS

CQRS signifie :

```text
Command Query Responsibility Segregation
```

L'idée est de séparer :

```text
Command
→ modifier l'état

Query
→ lire l'état
```

Mental model :

```text
             ┌── Query → lecture
Request ─────┤
             └── Command → modification
```

---

# 21. Command

Une command représente une intention de modification.

Exemple :

```text
CreateUserCommand
UpdateUserCommand
DeleteUserCommand
```

Elle peut être traitée par un handler :

```csharp
public class CreateUserCommandHandler
{
    public async Task Handle(
        CreateUserCommand command)
    {
        ...
    }
}
```

---

# 22. Query

Une query représente une demande de lecture.

Exemple :

```text
GetUserQuery
GetOrdersQuery
SearchProductsQuery
```

Elle ne devrait normalement pas modifier l'état métier.

Mental model :

```text
Command
→ write

Query
→ read
```

---

# 23. CQRS ne signifie pas forcément deux databases

C'est un piège.

CQRS peut être implémenté avec :

```text
une seule database
```

et simplement séparer les modèles ou chemins de lecture/écriture.

Une architecture plus avancée peut utiliser :

```text
write database
read database
```

mais ce n'est pas obligatoire.

---

# 24. Quand CQRS est intéressant ?

CQRS peut être pertinent lorsque :

```text
modèle métier complexe
lectures très différentes des écritures
besoin de scaling indépendant
workflows complexes
besoin d'isoler les responsabilités
```

Il peut être inutile pour un simple CRUD.

---

# 25. Repository Pattern

Le Repository fournit une abstraction autour de l'accès aux données.

Exemple :

```csharp
public interface IUserRepository
{
    Task<User?> GetByIdAsync(int id);
    Task AddAsync(User user);
}
```

Implémentation :

```csharp
public class UserRepository : IUserRepository
{
    private readonly AppDbContext _context;

    public UserRepository(AppDbContext context)
    {
        _context = context;
    }
}
```

Mental model :

```text
Application
 ↓
IUserRepository
 ↓
Infrastructure
 ↓
EF Core
```

---

# 26. Pourquoi utiliser un Repository ?

Il peut permettre :

```text
abstraction de persistance
centralisation de requêtes complexes
isolation d'une infrastructure
facilitation de certains tests
```

Mais il faut éviter de créer un repository qui ne fait que recopier chaque méthode de `DbSet`.

---

# 27. Generic Repository

Exemple :

```csharp
public interface IRepository<T>
{
    Task<T?> GetByIdAsync(int id);
    Task AddAsync(T entity);
    void Remove(T entity);
}
```

L'idée est de factoriser des opérations communes.

Mais un generic repository peut devenir trop générique.

Exemple :

```text
Get
Add
Update
Delete
```

ne suffit pas toujours à représenter correctement les besoins métier et les requêtes spécifiques.

---

# 28. Unit of Work

Une Unit of Work regroupe plusieurs changements et les persiste comme une unité.

EF Core possède déjà une abstraction très proche avec :

```csharp
DbContext
```

et :

```csharp
SaveChanges()
SaveChangesAsync()
```

Mental model :

```text
Operation A
Operation B
Operation C
     ↓
SaveChanges
     ↓
commit des changements
```

---

# 29. Repository + Unit of Work

Architecture classique :

```text
Service
   ↓
Repository
   ↓
Unit of Work
   ↓
Database
```

Mais avec EF Core :

```text
Service
   ↓
DbContext
   ↓
Database
```

peut déjà fournir une grande partie de ce comportement.

Il faut donc justifier l'ajout d'une abstraction supplémentaire.

---

# 30. Service Layer

Un service applicatif peut orchestrer un cas d'utilisation.

Exemple :

```csharp
public class OrderService
{
    public async Task CreateOrderAsync(...)
    {
        // validation métier
        // création
        // persistence
        // événements
    }
}
```

Le controller devient alors plus léger :

```csharp
[HttpPost]
public async Task<IActionResult> Create(
    CreateOrderDto dto)
{
    await _orderService.CreateAsync(dto);

    return Ok();
}
```

---

# 31. Domain Service

Un Domain Service peut contenir une règle métier qui ne trouve pas naturellement sa place dans une seule entité.

Exemple conceptuel :

```text
PricingService
```

pour calculer une règle complexe impliquant plusieurs concepts métier.

Attention à ne pas appeler tous les services "Domain Services" simplement parce qu'ils s'appellent `Service`.

---

# 32. Entity vs Value Object

### Entity

Possède une identité.

Exemple :

```text
User Id = 42
```

Même si son nom change :

```text
Ahlame
→ Ahlame Mohsine
```

cela reste le même utilisateur.

### Value Object

Est défini par sa valeur.

Exemple :

```text
Address
Money
EmailAddress
```

Mental model :

```text
Entity
→ identité

Value Object
→ valeur
```

---

# 33. Immutabilité

Une valeur immutable ne change pas après sa création.

Exemple :

```csharp
public record Money(
    decimal Amount,
    string Currency);
```

Pour modifier la valeur, on crée une nouvelle instance.

Avantages :

```text
moins d'effets de bord
raisonnement plus simple
sécurité en concurrence
modèle métier plus prévisible
```

---

# 34. Architecture et testabilité

Une architecture bien séparée permet différents types de tests :

```text
Unit tests
Integration tests
End-to-end tests
```

Exemple :

```text
Domain
→ unit tests

Application
→ unit tests

Infrastructure
→ integration tests

API
→ integration / end-to-end
```

---

# 35. Unit Test vs Integration Test

### Unit test

Teste une unité de code isolée.

Exemple :

```text
OrderCalculator
```

sans vraie database.

### Integration test

Teste plusieurs composants ensemble.

Exemple :

```text
API
+
EF Core
+
database
```

Mental model :

```text
Unit
→ isolé

Integration
→ composants réels ensemble
```

---

# 36. Architecture hexagonale

Architecture hexagonale, ou Ports and Adapters :

```text
          Adapter
             ↓
Adapter → Core ← Adapter
             ↑
          Adapter
```

Le cœur contient le métier.

Les adapters permettent de communiquer avec l'extérieur.

Exemples :

```text
HTTP adapter
Database adapter
Messaging adapter
```

---

# 37. Clean Architecture vs Hexagonal

Elles partagent une idée importante :

```text
protéger le cœur métier
+
inverser les dépendances
+
isoler les détails externes
```

Les noms et structures peuvent différer.

Il ne faut pas se focaliser uniquement sur le nom du pattern.

---

# 38. Dependency Rule

Question importante :

> Dans quelle direction les dépendances doivent-elles aller ?

Réponse générale :

```text
vers les abstractions / vers le cœur métier
```

Les détails techniques ne doivent pas imposer leur modèle au cœur métier.

Exemple :

```text
SQL Server
EF Core
Azure
SMTP
```

sont des détails externes.

Le métier ne devrait pas être construit autour de ces détails.

---

# 39. Anti-Corruption Layer

Lorsqu'un système externe possède un modèle très différent du nôtre, on peut créer une couche qui protège notre modèle.

Mental model :

```text
External System
      ↓
Adapter / ACL
      ↓
Our Domain
```

Cela évite de propager les concepts étrangers dans toute l'application.

---

# 40. Domain Events

Un Domain Event représente quelque chose qui s'est produit dans le domaine.

Exemple :

```text
OrderPlaced
UserRegistered
PaymentCompleted
```

L'événement peut être consommé par d'autres composants.

Mental model :

```text
Domain action
    ↓
Event
    ↓
handlers / consumers
```

Il permet notamment de découpler certaines réactions secondaires.

---

# 41. Synchronous vs Asynchronous communication

### Synchrone

```text
Service A
   ↓ request
Service B
   ↓ response
Service A
```

### Asynchrone

```text
Service A
   ↓ message
Queue
   ↓
Service B
```

L'asynchrone peut améliorer le découplage, mais apporte davantage de complexité :

```text
retries
ordering
duplicate messages
monitoring
eventual consistency
```

---

# 42. Monolith vs Microservices

### Monolithe

Une application déployée comme une unité.

Avantages :

```text
simple
déploiement plus facile
debug plus simple
transactions plus faciles
```

### Microservices

Plusieurs services indépendants.

Avantages possibles :

```text
déploiement indépendant
scaling indépendant
équipes autonomes
```

Mais coûts importants :

```text
réseau
observabilité
déploiement
résilience
consistance distribuée
```

---

# 43. Microservices ne signifie pas automatiquement meilleure architecture

Question d'entretien classique :

> "Pourquoi ne pas utiliser des microservices ?"

Réponse :

> Les microservices apportent des avantages dans certains contextes, mais aussi une forte complexité opérationnelle. Pour une application suffisamment simple, un monolithe modulaire peut être une meilleure solution.

---

# 44. Modular Monolith

Un monolithe peut être organisé en modules fortement séparés :

```text
Monolith
├── Users
├── Orders
├── Payments
└── Catalog
```

Chaque module peut avoir :

```text
API interne
Application
Domain
Infrastructure
```

On peut obtenir une bonne séparation sans introduire immédiatement la complexité des microservices.

---

# 45. Architecture pragmatique

Une bonne architecture répond au problème réel.

Il faut équilibrer :

```text
maintenabilité
simplicité
performance
testabilité
évolutivité
coût
```

Il n'existe pas une architecture parfaite pour tous les projets.

---

# 46. Question d'entretien : Clean Architecture

### Recruteur

**Pourquoi utiliser Clean Architecture ?**

### Réponse

> Pour séparer les responsabilités et protéger le cœur métier des détails techniques. L'objectif est notamment de rendre l'application plus testable et de permettre de faire évoluer ou remplacer certaines infrastructures sans modifier les règles métier centrales.

---

# 47. Question d'entretien : pourquoi Dependency Inversion ?

### Réponse

> Pour éviter que la logique métier dépende directement d'implémentations concrètes. Je fais dépendre le code de haut niveau d'abstractions, puis l'infrastructure fournit les implémentations concrètes.

---

# 48. Question d'entretien : Repository avec EF Core ?

### Réponse

> EF Core fournit déjà `DbSet` et `DbContext`, qui couvrent une grande partie des concepts Repository et Unit of Work. Je n'ajoute donc pas systématiquement une couche repository générique ; je l'utilise lorsqu'elle apporte une vraie abstraction ou une valeur architecturale.

---

# 49. Question d'entretien : CQRS ?

### Réponse

> CQRS sépare les responsabilités de lecture et d'écriture. Une Command modifie l'état, tandis qu'une Query récupère des données. CQRS peut être utile dans des domaines complexes, mais n'est pas nécessaire pour tous les CRUD.

---

# 50. Question d'entretien : Monolithe ou microservices ?

### Réponse

> Je choisirais en fonction des besoins. Un monolithe est souvent plus simple à développer, tester et déployer. Les microservices peuvent être pertinents lorsqu'il existe un vrai besoin de déploiement ou de scaling indépendant, mais ils ajoutent de la complexité distribuée.

---

# 51. Question d'entretien : comment éviter une architecture trop complexe ?

### Réponse

> Je commence par les besoins réels et je n'introduis une abstraction que lorsqu'elle résout un problème. Je privilégie une architecture simple, modulaire et évolutive plutôt qu'une accumulation de patterns sans justification.

---

# 52. Mini simulation

### Recruteur

**Montre-moi comment tu structurerais une API de réservation.**

### Réponse possible

```text
Booking.Api
    ↓
Booking.Application
    ↓
Booking.Domain
    ↑
Booking.Infrastructure
```

Puis :

```text
HTTP Request
    ↓
BookingController
    ↓
CreateBookingUseCase
    ↓
Domain rules
    ↓
IBookingRepository
    ↓
Infrastructure
    ↓
EF Core
    ↓
Database
```

Le controller ne connaît pas les détails SQL.

Le domaine ne connaît pas ASP.NET Core.

L'infrastructure implémente les abstractions nécessaires.

---

# 53. Mini simulation : changement de database

### Recruteur

**Que se passe-t-il si demain on remplace SQL Server par PostgreSQL ?**

Une architecture fortement couplée pourrait nécessiter des changements importants.

Avec une bonne séparation :

```text
Domain
→ aucun changement métier

Application
→ peu ou pas de changement

Infrastructure
→ adaptation du provider / implémentation
```

Il ne faut toutefois pas promettre :

```text
"aucun changement"
```

car les différences entre bases peuvent nécessiter des adaptations.

L'objectif est de limiter l'impact du changement.

---

# 54. Checklist Architecture

```text
[ ] Couplage
[ ] Cohésion
[ ] SOLID
[ ] SRP
[ ] OCP
[ ] LSP
[ ] ISP
[ ] DIP
[ ] Dependency Injection
[ ] Clean Architecture
[ ] Domain
[ ] Application
[ ] Infrastructure
[ ] API / Presentation
[ ] Direction des dépendances
[ ] CQRS
[ ] Command
[ ] Query
[ ] Repository
[ ] Generic Repository
[ ] Unit of Work
[ ] Service Layer
[ ] Domain Service
[ ] Entity
[ ] Value Object
[ ] Immutabilité
[ ] Unit Tests
[ ] Integration Tests
[ ] Hexagonal Architecture
[ ] Ports & Adapters
[ ] Domain Events
[ ] Communication synchrone/asynchrone
[ ] Monolithe
[ ] Microservices
[ ] Modular Monolith
[ ] Overengineering
```

# À retenir

```text
Architecture
→ organiser les responsabilités et dépendances

Couplage
→ niveau de dépendance entre composants

Cohésion
→ cohérence interne d'un composant

SOLID
→ principes de conception

DIP
→ dépendre d'abstractions

DI
→ fournir les dépendances

Clean Architecture
→ protéger le cœur métier

Domain
→ règles métier

Application
→ cas d'utilisation

Infrastructure
→ détails techniques

CQRS
→ séparer lecture et écriture

Repository
→ abstraction de persistance

Unit of Work
→ regrouper les changements

Monolith
→ simplicité

Microservices
→ indépendance avec coût distribué

Modular Monolith
→ séparation forte sans distribuer immédiatement le système
```

## Phrase à mémoriser

> **Une bonne architecture ne consiste pas à empiler des patterns : elle consiste à organiser les responsabilités et la direction des dépendances afin que le métier reste compréhensible, testable et le moins dépendant possible des détails techniques.**
