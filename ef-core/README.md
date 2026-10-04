# Entity Framework Core

Cette section regroupe les notions essentielles d'**Entity Framework Core (EF Core)** pour comprendre comment .NET communique avec une base de données relationnelle.

L'objectif n'est pas seulement de mémoriser des méthodes comme `Add()`, `SaveChangesAsync()` ou `Include()`, mais de comprendre ce qui se passe entre :

```text
Code C#
   |
   v
EF Core
   |
   v
SQL
   |
   v
Database
```

## Fiches

* [DbContext](dbcontext.md)
* [Tracking](tracking.md)
* [Relationships](relationships.md)
* [Migrations](migrations.md)
* [Querying](querying.md)
* [Performance](performance.md)
* [Seeding](seeding.md)

## Ordre conseillé d'apprentissage

Pour bien comprendre EF Core, je recommande cet ordre :

1. **DbContext**
2. **Relationships**
3. **Querying**
4. **Tracking**
5. **Migrations**
6. **Seeding**
7. **Performance**

Pourquoi ?

Parce qu'il faut d'abord comprendre comment EF Core représente la base et suit les entités avant d'aborder les optimisations.

---

# 1. Le modèle mental EF Core

EF Core est un **ORM** (*Object-Relational Mapper*).

Son objectif est de faire correspondre :

```text
Objets C#
      ↕
Tables SQL
```

Par exemple :

```csharp
public class Hotel
{
    public int Id { get; set; }

    public string Name { get; set; } = string.Empty;
}
```

peut correspondre à :

```sql
CREATE TABLE Hotels
(
    Id INT PRIMARY KEY,
    Name NVARCHAR(...)
);
```

EF Core fait le lien entre les deux mondes.

---

# 2. ORM

ORM signifie :

```text
Object
Relational
Mapping
```

### Object

Le monde C# :

```csharp
Hotel
User
Booking
```

### Relational

Le monde SQL :

```text
Tables
Rows
Columns
Foreign Keys
```

### Mapping

La correspondance entre les deux :

```text
Hotel.Id
    ↕
Hotels.Id

Hotel.Name
    ↕
Hotels.Name
```

---

# 3. `DbContext`

Le `DbContext` est l'un des objets les plus importants d'EF Core.

Il représente principalement une **session de travail avec la base de données**.

Exemple :

```csharp
public class ApplicationDbContext
    : DbContext
{
    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Hotel> Hotels => Set<Hotel>();
}
```

Mentalement :

```text
DbContext
   |
   +-- Hotels
   +-- Users
   +-- Bookings
```

Il coordonne notamment :

* les requêtes ;
* le suivi des entités ;
* la détection des modifications ;
* la génération des commandes SQL ;
* la sauvegarde des changements.

---

# 4. `DbSet<T>`

Un :

```csharp
DbSet<Hotel>
```

représente le point d'accès EF Core aux entités `Hotel`.

Exemple :

```csharp
var hotels = await context.Hotels.ToListAsync();
```

On demande à EF Core de récupérer les hôtels.

Il faut toutefois éviter de penser que :

```csharp
context.Hotels
```

est simplement une `List<Hotel>`.

C'est une abstraction permettant notamment de construire des requêtes qui pourront être traduites en SQL.

---

# 5. LINQ et EF Core

EF Core utilise fortement LINQ.

Exemple :

```csharp
var hotels = await context.Hotels
    .Where(h => h.City == "Mons")
    .ToListAsync();
```

Conceptuellement :

```text
LINQ
  |
  v
Expression
  |
  v
EF Core
  |
  v
SQL
  |
  v
Database
```

C'est une notion fondamentale.

---

# 6. `IQueryable<T>`

Une requête EF Core est généralement construite sous forme de :

```csharp
IQueryable<Hotel>
```

Cela permet à EF Core de conserver la requête et de la traduire vers le langage de la base.

Exemple :

```csharp
var query = context.Hotels
    .Where(h => h.Stars >= 4);
```

À ce stade, on n'a généralement pas encore récupéré tous les résultats.

Puis :

```csharp
var hotels = await query.ToListAsync();
```

déclenche l'exécution de la requête.

Mentalement :

```text
Construire la requête
        |
        v
IQueryable
        |
        v
ToListAsync()
        |
        v
SQL exécuté
```

---

# 7. `ToListAsync()` et exécution

Une erreur fréquente est de croire que chaque appel LINQ interroge immédiatement la base.

Exemple :

```csharp
var query = context.Hotels
    .Where(h => h.Stars >= 4)
    .OrderBy(h => h.Name);
```

On construit la requête.

Puis :

```csharp
var hotels = await query.ToListAsync();
```

on demande les résultats.

Mentalement :

```text
Where
OrderBy
Select
    |
    v
Construction de la requête

ToListAsync
    |
    v
Exécution
```

---

# 8. EF Core et SQL

Avec :

```csharp
var hotels = await context.Hotels
    .Where(h => h.Stars >= 4)
    .ToListAsync();
```

EF Core peut produire conceptuellement :

```sql
SELECT ...
FROM Hotels
WHERE Stars >= 4;
```

La syntaxe SQL exacte dépend notamment du provider utilisé.

Le point important est :

> LINQ ne signifie pas que la base reçoit du C#.

EF Core traduit l'expression vers le langage compris par la base.

---

# 9. `SaveChangesAsync()`

Pour les opérations de modification :

```csharp
context.Hotels.Add(hotel);

await context.SaveChangesAsync();
```

Il faut distinguer :

```text
Add()
```

et :

```text
SaveChangesAsync()
```

`Add()` indique au `DbContext` :

> « Cette entité doit être insérée. »

`SaveChangesAsync()` demande ensuite à EF Core de synchroniser les changements suivis avec la base.

Mentalement :

```text
Add
 |
 v
Change Tracker
 |
 v
SaveChangesAsync
 |
 v
SQL INSERT
 |
 v
Database
```

---

# 10. Change Tracker

EF Core possède un système appelé **Change Tracker**.

Il suit notamment l'état des entités :

```text
Detached
Unchanged
Added
Modified
Deleted
```

Exemple :

```csharp
var hotel = await context.Hotels.FindAsync(5);

hotel.Name = "New Name";

await context.SaveChangesAsync();
```

EF Core peut détecter que :

```text
Name a changé
```

et générer une commande SQL adaptée.

---

# 11. Tracking

Par défaut, les requêtes d'entités peuvent être suivies par le `DbContext`.

Exemple :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == 5);

hotel.Name = "New Name";

await context.SaveChangesAsync();
```

Le contexte connaît l'entité et peut détecter la modification.

Pour une lecture pure, on peut utiliser :

```csharp
var hotels = await context.Hotels
    .AsNoTracking()
    .ToListAsync();
```

Cela évite le tracking lorsque celui-ci n'est pas nécessaire.

---

# 12. Relationships

EF Core permet de représenter les relations entre entités.

Exemple :

```text
Hotel
  |
  +-- Rooms
```

En C# :

```csharp
public class Hotel
{
    public int Id { get; set; }

    public ICollection<Room> Rooms { get; set; }
        = new List<Room>();
}
```

Et :

```csharp
public class Room
{
    public int Id { get; set; }

    public int HotelId { get; set; }

    public Hotel Hotel { get; set; } = null!;
}
```

Cela correspond à une relation :

```text
Hotel 1 ---- N Room
```

---

# 13. Foreign Key

Dans :

```csharp
public int HotelId { get; set; }
```

`HotelId` représente la clé étrangère.

Conceptuellement :

```text
Rooms.HotelId
      |
      v
Hotels.Id
```

La relation est importante aussi bien côté C# que côté SQL.

---

# 14. `Include`

Pour charger une navigation :

```csharp
var hotels = await context.Hotels
    .Include(h => h.Rooms)
    .ToListAsync();
```

Mentalement :

```text
Hotel
  |
  +-- Rooms
```

`Include` demande à EF Core d'inclure les données de navigation dans le résultat selon la stratégie de requête utilisée.

Il ne faut toutefois pas utiliser `Include` automatiquement partout.

Pour certaines lectures, une projection est préférable :

```csharp
var hotels = await context.Hotels
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .ToListAsync();
```

---

# 15. Projection

La projection permet de demander uniquement les données nécessaires.

Exemple :

```csharp
var hotels = await context.Hotels
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .ToListAsync();
```

Mentalement :

```text
Entity complète
      |
      v
Select
      |
      v
DTO avec seulement les données nécessaires
```

C'est particulièrement important pour les APIs.

---

# 16. Migrations

Les migrations permettent de faire évoluer le schéma de la base en fonction des changements du modèle.

Exemple :

```text
Version 1
    Hotels(Id, Name)

Version 2
    Hotels(Id, Name, Stars)
```

Une migration représente le changement :

```text
Ajouter Stars
```

Le principe :

```text
C# Model
   |
   v
Migration
   |
   v
Database Schema
```

---

# 17. Seeding

Le **seeding** consiste à initialiser des données.

Exemples :

```text
Admin
Roles
Categories
Default configuration
```

Exemple conceptuel :

```csharp
modelBuilder.Entity<Role>().HasData(
    new Role
    {
        Id = 1,
        Name = "Admin"
    });
```

Le choix de la stratégie de seeding dépend du type de données et du projet.

---

# 18. Performance

EF Core peut produire du SQL efficace, mais il faut comprendre ce qui est réellement exécuté.

Points importants :

```text
Projection
AsNoTracking
Pagination
Indexes
Avoid N+1
Queries SQL
Tracking
```

Exemple :

Éviter de récupérer toute une table si on ne veut que quelques lignes :

```csharp
var hotels = await context.Hotels
    .Where(h => h.City == city)
    .Take(20)
    .ToListAsync();
```

---

# 19. N+1 Query Problem

Un problème classique :

```text
1 requête pour récupérer les hôtels
+
1 requête pour chaque hôtel pour récupérer les chambres
```

Avec 100 hôtels :

```text
1 + 100 = 101 requêtes
```

C'est potentiellement très coûteux.

Il faut analyser les requêtes générées et choisir une stratégie adaptée :

```text
Include
Projection
Split Queries
Requête adaptée
```

---

# 20. EF Core et architecture

Dans une architecture propre, EF Core appartient généralement à la partie infrastructure.

Exemple :

```text
API
 |
 v
Application
 |
 v
Domain
 ^
 |
Infrastructure
 |
 v
EF Core
 |
 v
Database
```

L'objectif est d'éviter que toute l'application dépende directement des détails de persistance.

---

# 21. Repository et EF Core

EF Core fournit déjà des abstractions comme :

```csharp
DbSet<T>
```

et un `DbContext` qui joue un rôle important dans l'accès aux données et le Unit of Work.

Il ne faut donc pas créer automatiquement :

```text
Repository générique
Repository de Repository
UnitOfWork autour de UnitOfWork
```

sans raison.

Le choix dépend de l'architecture du projet et des besoins réels.

---

# 22. `DbContext` et Dependency Injection

Dans ASP.NET Core, le `DbContext` est généralement enregistré via :

```csharp
builder.Services.AddDbContext<ApplicationDbContext>(
    options =>
    {
        options.UseSqlServer(connectionString);
    });
```

Le `DbContext` est généralement utilisé avec une durée de vie **Scoped** dans une application web.

Mentalement :

```text
HTTP Request
     |
     v
DbContext
     |
     +-- Query
     +-- Tracking
     +-- Changes
     |
     v
SaveChanges
```

---

# 23. Attention : DbContext n'est pas une connexion SQL permanente

Il est important de ne pas imaginer :

```text
DbContext = connexion SQL ouverte en permanence
```

Le `DbContext` représente plutôt une unité de travail et coordonne les opérations avec la base.

La gestion réelle des connexions est assurée par les mécanismes de provider et de connexion sous-jacents.

---

# 24. EF Core et transactions

`SaveChanges()` regroupe les modifications nécessaires dans une opération cohérente selon les mécanismes transactionnels d'EF Core.

Pour plusieurs opérations métier nécessitant une transaction explicite, il peut être nécessaire d'utiliser une transaction.

Mentalement :

```text
Operation A
Operation B
Operation C
    |
    v
Transaction
    |
    +---- success -> Commit
    |
    +---- error ---> Rollback
```

---

# 25. EF Core et async

Pour les opérations I/O :

Préférer :

```csharp
await context.Hotels
    .ToListAsync();
```

plutôt que de bloquer inutilement avec :

```csharp
context.Hotels
    .ToList();
```

dans une méthode asynchrone.

Le même principe s'applique à :

```text
FirstAsync
SingleAsync
AnyAsync
CountAsync
SaveChangesAsync
FindAsync
```

selon le besoin.

---

# 26. EF Core et CancellationToken

Les méthodes asynchrones acceptent souvent un `CancellationToken`.

Exemple :

```csharp
await context.Hotels
    .ToListAsync(cancellationToken);
```

Le token peut venir de :

```csharp
HttpContext.RequestAborted
```

et être propagé depuis :

```text
Controller
   |
   v
Service
   |
   v
Repository
   |
   v
EF Core
```

---

# 27. Erreurs fréquentes

## Erreur 1 : penser que `DbSet` est une `List`

```csharp
context.Hotels
```

n'est pas simplement une collection déjà chargée en mémoire.

Il peut représenter une requête construite pour la base.

---

## Erreur 2 : appeler `ToList()` trop tôt

Exemple :

```csharp
var hotels = context.Hotels
    .ToList();

var result = hotels
    .Where(h => h.Stars >= 4)
    .ToList();
```

Le filtrage final se fait alors en mémoire.

Il est généralement préférable de garder la requête côté serveur :

```csharp
var result = await context.Hotels
    .Where(h => h.Stars >= 4)
    .ToListAsync();
```

---

## Erreur 3 : utiliser `Include` partout

`Include` n'est pas une solution universelle.

Pour une API, une projection peut être plus adaptée :

```csharp
.Select(...)
```

---

## Erreur 4 : ignorer le tracking

Pour une lecture pure :

```csharp
AsNoTracking()
```

peut réduire le travail de tracking.

---

## Erreur 5 : ignorer les requêtes générées

Toujours garder en tête :

```text
C# LINQ
   |
   v
SQL
```

Une requête LINQ élégante peut malgré tout produire un SQL coûteux.

---

# 28. Règle mentale

Quand tu vois :

```csharp
context.Hotels.Where(...)
```

pense :

> « Je construis une requête que EF Core pourra traduire en SQL. »

Quand tu vois :

```csharp
ToListAsync()
```

pense :

> « J'exécute la requête et je récupère les résultats. »

Quand tu vois :

```csharp
SaveChangesAsync()
```

pense :

> « EF Core synchronise les changements suivis avec la base. »

Quand tu vois :

```csharp
AsNoTracking()
```

pense :

> « Je demande une lecture sans suivi des entités. »

Quand tu vois :

```csharp
Include(...)
```

pense :

> « Je demande des données de navigation associées. »

Quand tu vois :

```csharp
Select(...)
```

pense :

> « Je choisis précisément les données à récupérer/projeter. »

---

# 29. À retenir

1. EF Core est un ORM pour .NET.
2. Il fait le lien entre les objets C# et les données relationnelles.
3. `DbContext` représente une unité de travail avec la base.
4. `DbSet<T>` représente un point d'accès aux entités.
5. LINQ permet de construire des requêtes.
6. `IQueryable<T>` permet à EF Core de traduire la requête vers le provider.
7. `ToListAsync()` déclenche généralement l'exécution de la requête.
8. `SaveChangesAsync()` persiste les modifications suivies.
9. Le Change Tracker suit l'état des entités.
10. `AsNoTracking()` est utile pour les lectures qui ne nécessitent pas de tracking.
11. `Include()` permet de charger des navigations.
12. `Select()` permet de faire des projections.
13. Les relations EF Core correspondent notamment aux clés étrangères SQL.
14. Les migrations permettent de faire évoluer le schéma.
15. Le seeding initialise certaines données.
16. Il faut surveiller le problème N+1.
17. Il faut comprendre le SQL généré, pas seulement le LINQ écrit.
18. Les projections sont souvent utiles dans les APIs.
19. `DbContext` est généralement Scoped dans ASP.NET Core.
20. Les opérations I/O doivent généralement utiliser les API async d'EF Core.
21. `CancellationToken` peut être propagé jusqu'à EF Core.
22. EF Core appartient généralement à la couche Infrastructure dans une Clean Architecture.
23. Un Repository supplémentaire n'est pas automatiquement nécessaire simplement parce qu'EF Core existe.
24. Une requête LINQ n'est pas forcément exécutée au moment où elle est écrite.

---

# Questions d'entretien

### 1. Qu'est-ce qu'EF Core ?

Un ORM .NET permettant de travailler avec une base de données relationnelle à travers des objets C# et LINQ.

### 2. Quel est le rôle du `DbContext` ?

Il représente principalement une unité de travail avec la base et coordonne les requêtes, le tracking et la persistance des changements.

### 3. Quelle différence entre `DbSet<T>` et `List<T>` ?

`DbSet<T>` représente une abstraction EF Core permettant notamment de construire des requêtes vers la base ; `List<T>` contient des objets déjà en mémoire.

### 4. Quand une requête LINQ EF Core est-elle exécutée ?

Généralement lorsqu'on utilise un opérateur terminal qui nécessite les résultats, comme :

```csharp
ToListAsync()
FirstAsync()
SingleAsync()
CountAsync()
AnyAsync()
```

### 5. Quelle différence entre `IEnumerable<T>` et `IQueryable<T>` dans EF Core ?

`IEnumerable<T>` travaille généralement sur des données déjà en mémoire, tandis que `IQueryable<T>` permet de construire une expression que le provider EF Core peut traduire vers la base.

### 6. À quoi sert le Change Tracker ?

Il suit l'état des entités afin qu'EF Core puisse détecter les changements et générer les opérations de persistance nécessaires.

### 7. Pourquoi utiliser `AsNoTracking()` ?

Pour les lectures qui ne nécessitent pas de modifier ou persister les entités, afin d'éviter le travail de tracking.

### 8. Quelle différence entre `Include()` et `Select()` ?

`Include()` demande des données de navigation associées ; `Select()` permet de choisir et projeter précisément les données nécessaires.

### 9. Qu'est-ce que le problème N+1 ?

Un scénario où une première requête récupère une liste puis une requête supplémentaire est exécutée pour chaque élément, produisant potentiellement un très grand nombre de requêtes.

### 10. Pourquoi faut-il comprendre le SQL généré par EF Core ?

Parce que le code LINQ ne représente pas directement le coût réel de la requête. Le SQL généré détermine notamment ce qui est exécuté par la base.

### 11. Pourquoi `SaveChangesAsync()` est-il important ?

Parce que les changements suivis par le `DbContext` ne sont pas simplement persistés par le fait de modifier un objet C#. `SaveChangesAsync()` demande à EF Core de synchroniser ces changements avec la base.

### 12. Pourquoi le `DbContext` est-il généralement Scoped dans ASP.NET Core ?

Parce qu'il est généralement associé à une unité de travail correspondant à une requête HTTP et qu'il n'est pas conçu pour être partagé comme un Singleton entre toutes les requêtes.

---

# Phrase à retenir

> **EF Core fait le pont entre C# et SQL : je construis une requête avec LINQ, EF Core la traduit en SQL, le DbContext suit les entités et SaveChangesAsync synchronise les modifications avec la base.**
