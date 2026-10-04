# Architecture — CQRS

## 1. Qu'est-ce que CQRS ?

**CQRS** signifie :

> **Command Query Responsibility Segregation**

L'idée est de séparer :

```text
Command
= modifier l'état

Query
= lire l'état
```

Donc :

```text
Command → Write
Query   → Read
```

Le principe est simple :

> **Une opération qui modifie les données n'a pas exactement les mêmes responsabilités qu'une opération qui les lit.**

---

# 2. Pourquoi séparer lecture et écriture ?

Dans une application classique, on peut avoir un service qui fait tout :

```csharp
public class HotelService
{
    public Task<Hotel?> GetAsync(int id)
    {
        ...
    }

    public Task CreateAsync(CreateHotelRequest request)
    {
        ...
    }

    public Task UpdateAsync(...)
    {
        ...
    }

    public Task DeleteAsync(int id)
    {
        ...
    }
}
```

Cela fonctionne.

Mais lorsque l'application devient complexe, les opérations de lecture et d'écriture peuvent avoir des besoins très différents.

Une lecture peut chercher :

```text
DTO
pagination
filtres
tri
projection
jointures
```

Une écriture peut nécessiter :

```text
validation
règles métier
création d'entités
transaction
événements
```

CQRS permet de rendre cette différence explicite.

---

# 3. Command

Une **Command** représente une demande de modification de l'état.

Exemples :

```text
CreateHotel
UpdateHotel
DeleteHotel
CreateReservation
CancelReservation
ChangePassword
```

Une Command signifie :

> « Je demande au système d'effectuer une action. »

Exemple :

```csharp
public record CreateHotelCommand(
    string Name,
    string Address);
```

La commande contient généralement les informations nécessaires à l'exécution de l'action.

---

# 4. Query

Une **Query** représente une demande de lecture.

Exemples :

```text
GetHotelById
GetHotels
SearchHotels
GetReservations
GetCustomerDetails
```

Une Query signifie :

> « Je demande au système de me donner une information. »

Exemple :

```csharp
public record GetHotelByIdQuery(int Id);
```

Une query ne devrait pas avoir pour objectif de modifier l'état du système.

---

# 5. Le principe de séparation

On obtient :

```text
                 Application
                      |
             +--------+--------+
             |                 |
             v                 v
         Commands           Queries
             |                 |
             v                 v
          Handlers           Handlers
             |                 |
             v                 v
          Domain             Read model
             |                 |
             v                 v
        Persistence       Persistence
```

Les deux chemins peuvent donc être optimisés différemment.

---

# 6. Command Handler

Une Command est généralement traitée par un **Command Handler**.

Exemple :

```csharp
public class CreateHotelHandler
{
    private readonly IHotelRepository _repository;

    public CreateHotelHandler(
        IHotelRepository repository)
    {
        _repository = repository;
    }

    public async Task HandleAsync(
        CreateHotelCommand command,
        CancellationToken cancellationToken)
    {
        var hotel = new Hotel(
            command.Name,
            command.Address);

        await _repository.AddAsync(
            hotel,
            cancellationToken);
    }
}
```

Le Handler orchestre l'exécution de la commande.

Il ne faut pas automatiquement mettre toutes les règles métier dans le Handler.

Les règles appartenant au domaine doivent rester dans le domaine.

---

# 7. Query Handler

Une Query possède généralement son propre Handler.

Exemple :

```csharp
public class GetHotelByIdHandler
{
    private readonly AppDbContext _context;

    public GetHotelByIdHandler(
        AppDbContext context)
    {
        _context = context;
    }

    public async Task<HotelResponse?> HandleAsync(
        GetHotelByIdQuery query,
        CancellationToken cancellationToken)
    {
        return await _context.Hotels
            .AsNoTracking()
            .Where(h => h.Id == query.Id)
            .Select(h => new HotelResponse(
                h.Id,
                h.Name,
                h.Address))
            .FirstOrDefaultAsync(cancellationToken);
    }
}
```

Remarque importante :

La lecture peut directement faire une projection vers un DTO.

Il n'est pas toujours nécessaire de charger une entité complète.

---

# 8. Pourquoi les Queries sont souvent différentes ?

Prenons une page :

```text
Liste des hôtels
```

L'utilisateur veut :

```text
Id
Nom
Ville
Prix moyen
Nombre de chambres
Note
```

Il n'a pas forcément besoin de :

```text
Entity complète
Navigation properties
Tracking
Domain methods
```

La query peut donc faire :

```csharp
.Select(h => new HotelListItemDto
{
    Id = h.Id,
    Name = h.Name,
    City = h.City
})
```

EF Core peut alors générer un SQL qui ne sélectionne que les colonnes nécessaires.

---

# 9. Command vs Query

| Command | Query |
|---|---|
| Modifie l'état | Lit l'état |
| Représente une action | Représente une question |
| Peut déclencher des règles métier | Devrait rester orientée lecture |
| Peut utiliser une transaction | Généralement lecture seule |
| `CreateHotel` | `GetHotel` |
| `CancelReservation` | `GetReservation` |

Mentalement :

```text
Command
→ Fais quelque chose.

Query
→ Donne-moi quelque chose.
```

---

# 10. CQRS n'impose pas deux bases de données

C'est une confusion très fréquente.

CQRS ne signifie pas obligatoirement :

```text
Write Database
       +
Read Database
```

On peut commencer avec :

```text
                 CQRS
                  |
          +-------+-------+
          |               |
        Write            Read
          |               |
          +-------+-------+
                  |
             SQL Server
```

La même base peut être utilisée pour les deux.

---

# 11. CQRS simple

Une application peut utiliser :

```text
Commands
   ↓
Write Model
   ↓
Database

Queries
   ↓
Read Model
   ↓
Database
```

avec une seule base.

C'est souvent une bonne manière d'introduire CQRS sans ajouter immédiatement beaucoup de complexité.

---

# 12. CQRS avec bases séparées

Dans une architecture plus avancée :

```text
                 Application
                /           \
               /             \
          Commands          Queries
              |                |
              v                v
        Write Database     Read Database
              |                ^
              |                |
              +--> Events -----+
```

Les changements effectués dans la base d'écriture peuvent être propagés vers une base optimisée pour les lectures.

On peut alors avoir :

```text
SQL Server
   +
Read replica
   +
ElasticSearch
   +
NoSQL
```

selon les besoins.

Mais cette architecture ajoute de la complexité.

---

# 13. CQRS et Eventual Consistency

Lorsque lecture et écriture utilisent des systèmes séparés, les données peuvent ne pas être immédiatement synchronisées.

Exemple :

```text
Command
  ↓
Write DB
  ↓
Event
  ↓
Read DB
```

Pendant un court instant :

```text
Write DB = nouvelle valeur
Read DB  = ancienne valeur
```

On parle d'**eventual consistency**.

La lecture finit par recevoir la nouvelle information, mais pas forcément immédiatement.

---

# 14. CQRS n'est pas Event Sourcing

Deux concepts sont souvent mélangés.

### CQRS

Sépare :

```text
Command
Query
```

### Event Sourcing

Stocke l'évolution de l'état sous forme d'événements.

Exemple :

```text
AccountCreated
MoneyDeposited
MoneyWithdrawn
```

L'état peut être reconstruit à partir de ces événements.

On peut avoir :

```text
CQRS sans Event Sourcing
```

et :

```text
Event Sourcing sans CQRS complet
```

Ils sont souvent associés, mais ils ne sont pas synonymes.

---

# 15. CQRS et Clean Architecture

Les deux peuvent très bien fonctionner ensemble.

Exemple :

```text
API
 |
 +-------------------+
 |                   |
Command             Query
 |                   |
CommandHandler      QueryHandler
 |                   |
Domain              EF Core projection
 |
Infrastructure
```

Clean Architecture répond plutôt à :

> « Comment organiser les dépendances et protéger le métier ? »

CQRS répond plutôt à :

> « Comment séparer les opérations de lecture et d'écriture ? »

Ce sont donc deux problématiques différentes.

---

# 16. CQRS et MediatR

MediatR est souvent utilisé avec CQRS.

Exemple :

```csharp
public record CreateHotelCommand(
    string Name,
    string Address)
    : IRequest<int>;
```

Handler :

```csharp
public class CreateHotelHandler
    : IRequestHandler<CreateHotelCommand, int>
{
    public async Task<int> Handle(
        CreateHotelCommand request,
        CancellationToken cancellationToken)
    {
        // traitement
    }
}
```

Puis depuis le controller :

```csharp
var id = await _mediator.Send(
    command,
    cancellationToken);
```

Le controller ne connaît pas directement le Handler.

---

# 17. Attention : MediatR n'est pas CQRS

On pourrait faire du CQRS sans MediatR :

```csharp
var result =
    await _createHotelHandler.HandleAsync(
        command,
        cancellationToken);
```

Et on peut utiliser MediatR sans appliquer un CQRS particulièrement sophistiqué.

Donc :

```text
CQRS
= architecture / principe de séparation

MediatR
= bibliothèque de médiation
```

MediatR est un moyen possible d'implémenter cette organisation, pas sa définition.

---

# 18. Exemple ASP.NET Core

Command :

```csharp
public record CreateHotelCommand(
    string Name,
    string City);
```

Controller :

```csharp
[HttpPost]
public async Task<IActionResult> Create(
    CreateHotelRequest request,
    CancellationToken cancellationToken)
{
    var command = new CreateHotelCommand(
        request.Name,
        request.City);

    await _handler.HandleAsync(
        command,
        cancellationToken);

    return Ok();
}
```

Query :

```csharp
public record GetHotelQuery(int Id);
```

Controller :

```csharp
[HttpGet("{id:int}")]
public async Task<IActionResult> Get(
    int id,
    CancellationToken cancellationToken)
{
    var query = new GetHotelQuery(id);

    var result = await _handler.HandleAsync(
        query,
        cancellationToken);

    if (result is null)
    {
        return NotFound();
    }

    return Ok(result);
}
```

On obtient deux chemins clairement séparés.

---

# 19. Où placer les règles métier ?

C'est une question importante.

Ne pas transformer :

```text
CommandHandler
```

en :

```text
God Object
```

Exemple mauvais :

```csharp
public async Task Handle(...)
{
    // 300 lignes
    // toutes les règles métier
    // calculs
    // validation
    // persistance
}
```

Le Handler doit principalement **orchestrer**.

Les règles importantes doivent rester dans le domaine lorsqu'elles appartiennent réellement au domaine.

Exemple :

```csharp
reservation.Cancel();
```

plutôt que :

```csharp
handler.CancelReservationWithTwentyConditions(...);
```

---

# 20. Validation d'une Command

Il peut être utile de distinguer :

```text
Validation de forme
```

et :

```text
Règle métier
```

Exemple :

```text
Name obligatoire
Name maximum 200 caractères
```

peuvent être des validations de données.

Alors que :

```text
Une réservation ne peut être annulée après le check-in
```

est une règle métier.

Le fait d'utiliser CQRS ne change pas cette distinction.

---

# 21. Transaction dans une Command

Une Command peut modifier plusieurs éléments :

```text
Créer commande
Créer lignes
Mettre à jour stock
Créer paiement
```

Ces opérations peuvent devoir être atomiques.

Conceptuellement :

```text
BEGIN TRANSACTION

Create Order
Create OrderLines
Update Stock

COMMIT
```

Si une opération critique échoue :

```text
ROLLBACK
```

Le Command Handler ou la couche appropriée orchestre ce processus selon l'architecture choisie.

---

# 22. Pourquoi les Queries peuvent être très simples ?

Une Query peut souvent être optimisée directement pour le besoin d'affichage.

Exemple :

```csharp
var hotels = await _context.Hotels
    .AsNoTracking()
    .Where(h => h.City == city)
    .OrderBy(h => h.Name)
    .Select(h => new HotelListItemDto
    {
        Id = h.Id,
        Name = h.Name,
        City = h.City
    })
    .ToListAsync(cancellationToken);
```

Ici :

```text
AsNoTracking
+
Where
+
OrderBy
+
Select
```

sont directement adaptés à une lecture.

Il n'est pas nécessaire de charger toute l'entité puis de la transformer si la page n'a besoin que de quelques colonnes.

---

# 23. CQRS et performance

CQRS peut faciliter l'optimisation des lectures.

Par exemple :

```text
Write model
    → entités riches
    → règles métier
    → tracking
```

et :

```text
Read model
    → DTO
    → projection
    → AsNoTracking
    → requête spécialisée
```

Cela permet de ne pas imposer les mêmes contraintes aux deux côtés.

Mais attention :

> CQRS n'améliore pas automatiquement les performances.

Une mauvaise query reste une mauvaise query.

---

# 24. CQRS et scalabilité

Dans des systèmes importants, la séparation peut permettre de faire évoluer différemment :

```text
Write side
    → peu d'opérations mais critiques

Read side
    → énormément de lectures
```

On peut alors dimensionner les deux parties différemment.

Exemple :

```text
                 API
                  |
        +---------+---------+
        |                   |
      Writes              Reads
        |                   |
     Service              Read API
        |                   |
    SQL Server         Read DB / Cache
```

Cela devient intéressant lorsque les besoins réels le justifient.

---

# 25. CQRS et cache

Les Queries sont également de bonnes candidates pour certaines stratégies de cache.

Exemple :

```text
GET /api/hotels
        ↓
Cache
   ↓         ↓
Hit         Miss
 ↓            ↓
Return     Database
```

Une Command peut ensuite invalider ou mettre à jour le cache.

CQRS ne fournit pas le cache lui-même, mais la séparation rend ce type de stratégie plus facile à raisonner.

---

# 26. Les inconvénients de CQRS

CQRS ajoute de la structure.

Au lieu de :

```text
HotelService
```

on peut avoir :

```text
CreateHotelCommand
CreateHotelHandler

UpdateHotelCommand
UpdateHotelHandler

DeleteHotelCommand
DeleteHotelHandler

GetHotelQuery
GetHotelHandler

GetHotelsQuery
GetHotelsHandler
```

Pour un petit CRUD, cela peut devenir beaucoup de code.

Les coûts possibles sont :

- plus de fichiers ;
- plus de classes ;
- plus de concepts ;
- plus de configuration ;
- courbe d'apprentissage ;
- risque d'abstraction inutile.

---

# 27. Quand CQRS est-il intéressant ?

CQRS devient particulièrement intéressant lorsque :

- les lectures et écritures sont très différentes ;
- les règles métier sont complexes ;
- les lectures nécessitent des projections spécialisées ;
- l'application devient importante ;
- différentes stratégies de scaling sont nécessaires ;
- le domaine est complexe ;
- l'équipe bénéficie d'une séparation claire des responsabilités.

---

# 28. Quand éviter CQRS ?

Pour un petit projet :

```text
Todo API
```

avec :

```text
Create
Get
Update
Delete
```

un simple service peut être largement suffisant.

Il n'est pas nécessaire d'avoir :

```text
20 Commands
20 Queries
40 Handlers
MediatR
Events
Read database
Write database
```

simplement pour respecter une architecture à la mode.

---

# 29. Le niveau de CQRS

CQRS n'est pas forcément :

```text
CQRS simple
    ↓
CQRS avancé
    ↓
CQRS + Event Bus
    ↓
CQRS + Event Sourcing
    ↓
microservices
    ↓
deux bases
```

On peut adopter uniquement le principe de base :

```text
Commands ≠ Queries
```

sans introduire toute la complexité supplémentaire.

---

# 30. Exemple mental

Imagine un restaurant.

### Command

```text
Client :
« Je veux commander une pizza. »
```

Le système doit :

```text
Créer commande
Vérifier disponibilité
Calculer prix
Enregistrer commande
```

### Query

```text
Client :
« Quelles pizzas sont disponibles ? »
```

Le système doit :

```text
Lire les pizzas
Filtrer
Trier
Retourner les informations
```

Les deux demandes sont différentes.

C'est exactement l'idée de CQRS :

```text
Command
→ Fais quelque chose.

Query
→ Donne-moi quelque chose.
```

---

# 31. Checklist CQRS

Avant d'introduire CQRS :

- [ ] Les lectures et écritures ont-elles réellement des besoins différents ?
- [ ] Les règles métier justifient-elles la séparation ?
- [ ] Les Queries peuvent-elles bénéficier de projections spécifiques ?
- [ ] Les Commands ont-elles des workflows complexes ?
- [ ] Le projet est-il suffisamment complexe pour supporter cette abstraction ?
- [ ] Est-ce que CQRS apporte une vraie valeur ?
- [ ] Les Handlers restent-ils focalisés sur un cas d'utilisation ?
- [ ] Les règles métier importantes restent-elles dans le domaine ?
- [ ] Ai-je évité d'ajouter une deuxième base sans besoin réel ?
- [ ] Ai-je évité de confondre CQRS avec MediatR ou Event Sourcing ?

---

# 32. Questions d'entretien

### 1. Que signifie CQRS ?

**Command Query Responsibility Segregation.**

Il s'agit de séparer les opérations de modification de l'état des opérations de lecture.

---

### 2. Quelle est la différence entre Command et Query ?

Une Command modifie l'état du système. Une Query lit des informations sans avoir pour objectif de modifier l'état.

---

### 3. CQRS nécessite-t-il deux bases de données ?

Non.

On peut utiliser une seule base pour les lectures et les écritures.

---

### 4. CQRS nécessite-t-il MediatR ?

Non.

MediatR est une bibliothèque qui peut faciliter l'implémentation d'une architecture basée sur Commands et Queries.

---

### 5. CQRS est-il la même chose qu'Event Sourcing ?

Non.

CQRS sépare les lectures et les écritures. Event Sourcing stocke l'évolution de l'état sous forme d'événements.

---

### 6. Pourquoi utiliser CQRS avec EF Core ?

Notamment pour pouvoir avoir des lectures optimisées avec des projections et des requêtes adaptées au besoin, tout en gardant les écritures centrées sur les règles métier.

---

### 7. Quel est le principal inconvénient de CQRS ?

La complexité supplémentaire. Pour une petite application CRUD, CQRS peut être inutilement lourd.

---

# À retenir

```text
COMMAND
    ↓
Change l'état

QUERY
    ↓
Lit l'état
```

CQRS permet ensuite d'aller plus loin :

```text
Write side
    → règles métier
    → transactions
    → entités

Read side
    → projections
    → DTOs
    → performances de lecture
```

Mais :

```text
CQRS ≠ deux bases
CQRS ≠ MediatR
CQRS ≠ Event Sourcing
```

# Phrase à mémoriser

> **CQRS sépare ce qui demande au système de changer de ce qui demande au système de lire.**
