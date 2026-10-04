# Entretien — Entity Framework Core

Cette fiche rassemble les notions EF Core importantes pour un entretien de développeuse .NET.

L'objectif est de comprendre ce qui se passe entre le code C# et la base de données :

```text
C# / LINQ
   ↓
EF Core
   ↓
Expression / Query
   ↓
SQL
   ↓
Database
```

---

# 1. Qu'est-ce qu'Entity Framework Core ?

Entity Framework Core, ou EF Core, est un ORM :

```text
Object-Relational Mapper
```

Il permet de travailler avec une base de données à travers des objets C#.

Sans ORM, on peut écrire directement :

```sql
SELECT *
FROM Users
WHERE Id = 42;
```

Avec EF Core :

```csharp
var user = await context.Users
    .FirstOrDefaultAsync(u => u.Id == 42);
```

EF Core peut traduire la requête LINQ en SQL selon le provider utilisé.

Mental model :

```text
C# objects
    ↕
EF Core
    ↕
Relational database
```

---

# 2. Pourquoi utiliser un ORM ?

Un ORM permet notamment de :

```text
réduire le SQL répétitif
travailler avec des objets C#
utiliser LINQ
gérer les relations
gérer les changements d'état
générer des migrations
```

Mais un ORM ne dispense pas de comprendre SQL.

Un développeur EF Core doit savoir raisonner sur :

```text
JOIN
INDEX
WHERE
ORDER BY
GROUP BY
pagination
transactions
performance
```

---

# 3. `DbContext`

Le `DbContext` est l'objet central d'EF Core.

Exemple :

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(
        DbContextOptions<AppDbContext> options)
        : base(options)
    {
    }

    public DbSet<User> Users => Set<User>();
    public DbSet<Order> Orders => Set<Order>();
}
```

Il représente notamment une session de travail avec la base de données.

Il fournit :

```text
DbSet
Change Tracker
configuration
requêtes
SaveChanges
transactions selon le contexte
```

Mental model :

```text
DbContext
   ↓
unit of work
   ↓
changes
   ↓
SaveChanges()
   ↓
database
```

---

# 4. `DbSet<T>`

Un `DbSet<T>` représente généralement l'ensemble des entités d'un type pouvant être interrogées ou persistées.

Exemple :

```csharp
public DbSet<User> Users => Set<User>();
```

On peut ensuite écrire :

```csharp
var users = await context.Users
    .ToListAsync();
```

---

# 5. Cycle de vie du `DbContext`

Dans ASP.NET Core, le `DbContext` est généralement enregistré en `Scoped`.

```csharp
builder.Services.AddDbContext<AppDbContext>(
    options =>
        options.UseSqlServer(connectionString));
```

Mental model typique :

```text
HTTP Request
    ↓
DbContext scoped
    ↓
queries / changes
    ↓
SaveChanges
    ↓
Request end
    ↓
DbContext disposed
```

Le `DbContext` n'est donc généralement pas un singleton.

---

# 6. Pourquoi éviter un `DbContext` Singleton ?

Le `DbContext` contient notamment un état de tracking et n'est pas conçu pour être partagé globalement entre de nombreuses requêtes concurrentes.

Un singleton pourrait provoquer des problèmes liés à :

```text
concurrence
état accumulé
tracking
durée de vie
```

La configuration scoped est le scénario classique pour ASP.NET Core.

---

# 7. Entity

Une entity représente généralement un objet métier/persisté possédant une identité.

Exemple :

```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}
```

EF Core utilise les propriétés et la configuration pour déterminer comment mapper l'entité vers la base.

---

# 8. Convention over configuration

EF Core possède des conventions.

Exemple :

```csharp
public int Id { get; set; }
```

peut être reconnu comme clé primaire selon les conventions.

On peut néanmoins modifier explicitement la configuration lorsque les conventions ne suffisent pas.

---

# 9. Data Annotations vs Fluent API

### Data Annotations

```csharp
[Required]
[MaxLength(100)]
public string Name { get; set; } = "";
```

### Fluent API

```csharp
modelBuilder.Entity<User>(entity =>
{
    entity.HasKey(u => u.Id);

    entity.Property(u => u.Name)
        .IsRequired()
        .HasMaxLength(100);
});
```

La Fluent API est généralement plus puissante et permet de centraliser la configuration EF.

---

# 10. `OnModelCreating`

La configuration Fluent API se fait généralement dans :

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    ...
}
```

Exemple :

```csharp
modelBuilder.Entity<User>(entity =>
{
    entity.HasKey(u => u.Id);

    entity.Property(u => u.Name)
        .IsRequired()
        .HasMaxLength(100);
});
```

---

# 11. Change Tracker

EF Core suit l'état des entités attachées au `DbContext`.

États principaux :

```text
Detached
Unchanged
Added
Modified
Deleted
```

Exemple :

```csharp
var user = await context.Users
    .FirstAsync();

user.Name = "New name";

await context.SaveChangesAsync();
```

Le Change Tracker détecte que :

```text
Name
```

a changé.

EF Core peut alors générer un `UPDATE`.

Mental model :

```text
Database
   ↓
entity loaded
   ↓
Change Tracker remembers state
   ↓
property modified
   ↓
SaveChanges
   ↓
UPDATE
```

---

# 12. `SaveChanges()`

`SaveChanges()` ou `SaveChangesAsync()` demande à EF Core de persister les changements suivis.

Exemple :

```csharp
context.Users.Add(user);

await context.SaveChangesAsync();
```

Conceptuellement :

```text
Add
 ↓
Added
 ↓
SaveChanges
 ↓
INSERT
```

---

# 13. `SaveChangesAsync()`

Dans une application web, on préfère généralement :

```csharp
await context.SaveChangesAsync();
```

pour les opérations I/O asynchrones.

L'objectif est de ne pas bloquer inutilement le thread pendant l'attente de la base de données.

---

# 14. Tracking

Par défaut, une requête EF Core sur des entités peut être trackée.

Exemple :

```csharp
var user = await context.Users
    .FirstAsync(u => u.Id == id);

user.Name = "Ahlame";

await context.SaveChangesAsync();
```

Pas besoin de :

```csharp
context.Users.Update(user);
```

si l'entité est déjà trackée et que le changement est détecté.

---

# 15. `AsNoTracking()`

Pour une lecture seule :

```csharp
var users = await context.Users
    .AsNoTracking()
    .ToListAsync();
```

Cela désactive le tracking des entités retournées pour cette requête.

Avantages possibles :

```text
moins de travail du Change Tracker
moins de mémoire
lecture seule plus adaptée
```

Il ne faut cependant pas utiliser `AsNoTracking()` automatiquement partout.

Si on veut modifier directement les entités puis appeler `SaveChanges`, le tracking peut être nécessaire.

---

# 16. Tracking vs No Tracking

Mental model :

```text
Tracking
→ EF surveille les entités

NoTracking
→ EF récupère les données sans les suivre
```

Pour :

```text
GET /users
```

où les données sont seulement affichées :

```csharp
AsNoTracking()
```

peut être pertinent.

Pour :

```text
charger
modifier
SaveChanges
```

le tracking est souvent adapté.

---

# 17. LINQ avec EF Core

Exemple :

```csharp
var users = await context.Users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .ToListAsync();
```

EF Core peut traduire cette expression vers SQL.

Conceptuellement :

```text
Where
 ↓
SQL WHERE

OrderBy
 ↓
SQL ORDER BY

ToListAsync
 ↓
exécution
```

---

# 18. Deferred execution avec EF Core

Exemple :

```csharp
var query = context.Users
    .Where(u => u.IsActive);
```

La requête n'est généralement pas exécutée immédiatement.

Puis :

```csharp
var users = await query.ToListAsync();
```

déclenche l'exécution.

Mental model :

```text
IQueryable
 ↓
construction de la requête
 ↓
ToListAsync()
 ↓
SQL
 ↓
Database
```

---

# 19. `IQueryable`

Avec EF Core :

```csharp
IQueryable<User>
```

représente une requête composable.

Exemple :

```csharp
var query = context.Users
    .Where(u => u.IsActive);

query = query.Where(u => u.Age >= 18);

var users = await query.ToListAsync();
```

EF Core peut construire une requête SQL correspondant à l'expression.

---

# 20. Attention à `ToList()`

Exemple :

```csharp
var users = await context.Users
    .ToListAsync();

var adults = users
    .Where(u => u.Age >= 18)
    .ToList();
```

Ici :

```text
Database
 ↓
tous les users
 ↓
mémoire
 ↓
Where en C#
```

Alors que :

```csharp
var adults = await context.Users
    .Where(u => u.Age >= 18)
    .ToListAsync();
```

permet au provider de traduire le filtre en SQL.

Mental model :

```text
Filtrer avant ToList
→ généralement côté database

Filtrer après ToList
→ côté mémoire
```

---

# 21. `Include`

Pour charger une relation :

```csharp
var orders = await context.Orders
    .Include(o => o.User)
    .ToListAsync();
```

`Include` indique à EF Core de charger les données liées selon la requête générée.

---

# 22. `ThenInclude`

Pour naviguer plus profondément :

```csharp
var orders = await context.Orders
    .Include(o => o.User)
        .ThenInclude(u => u.Address)
    .ToListAsync();
```

Mental model :

```text
Order
 ↓
User
 ↓
Address
```

---

# 23. Eager, Explicit et Lazy Loading

### Eager loading

Chargement explicite dans la requête :

```csharp
.Include(...)
```

### Explicit loading

On demande explicitement le chargement après coup.

### Lazy loading

La relation est chargée automatiquement lorsqu'on y accède, si le mécanisme est configuré.

Attention au lazy loading dans les APIs : il peut provoquer des requêtes inattendues et des problèmes de type N+1.

---

# 24. N+1 Query Problem

Exemple conceptuel :

```text
1 requête
→ récupérer 100 commandes

puis pour chaque commande :
100 requêtes supplémentaires
```

Total :

```text
101 requêtes
```

C'est le problème N+1.

Il peut être provoqué par une mauvaise utilisation des relations ou du lazy loading.

On peut notamment utiliser :

```text
projection
Include
requêtes adaptées
```

selon le cas.

---

# 25. Projection avec `Select`

Au lieu de charger toute l'entité :

```csharp
var users = await context.Users
    .Select(u => new UserDto
    {
        Id = u.Id,
        Name = u.Name
    })
    .ToListAsync();
```

On ne demande que les données nécessaires.

Mental model :

```text
Entity complète
→ beaucoup de colonnes

Projection
→ uniquement les données nécessaires
```

Cela peut améliorer :

```text
trafic DB
mémoire
temps de traitement
```

---

# 26. Relations

Les relations classiques :

```text
One-to-One
One-to-Many
Many-to-Many
```

Exemple :

```text
User
 ↓
Orders
```

Un utilisateur peut avoir plusieurs commandes.

---

# 27. One-to-Many

Exemple :

```csharp
public class User
{
    public int Id { get; set; }

    public ICollection<Order> Orders { get; set; }
        = new List<Order>();
}

public class Order
{
    public int Id { get; set; }

    public int UserId { get; set; }

    public User User { get; set; } = null!;
}
```

Conceptuellement :

```text
User 1
  ↓
  ↓
Order *
```

---

# 28. Foreign Key

Dans :

```csharp
public int UserId { get; set; }
```

`UserId` peut représenter la clé étrangère vers :

```text
User.Id
```

La relation devient :

```text
Users.Id
    ↑
    |
Orders.UserId
```

---

# 29. Navigation property

Exemple :

```csharp
public User User { get; set; } = null!;
```

est une navigation property.

Elle permet de naviguer depuis l'entité :

```csharp
order.User
```

La foreign key est généralement :

```csharp
order.UserId
```

Mental model :

```text
UserId
→ valeur de la relation

User
→ navigation vers l'objet lié
```

---

# 30. Cascade delete

Une relation peut définir ce qui arrive aux entités dépendantes lorsqu'une entité principale est supprimée.

Exemple conceptuel :

```text
User supprimé
   ↓
Orders ?
```

Selon la configuration :

```text
Cascade
Restrict
NoAction
SetNull
```

Il faut être particulièrement prudent avec les suppressions en cascade sur des modèles complexes.

---

# 31. Many-to-Many

Exemple :

```text
Student
   ↕
Enrollment
   ↕
Course
```

Une table de jointure permet de représenter :

```text
Student ↔ Course
```

EF Core moderne peut également gérer certaines relations many-to-many sans nécessiter une entité de jointure explicite dans les modèles simples.

---

# 32. Migrations

Les migrations permettent de faire évoluer le schéma de la base de données à partir du modèle EF Core.

Commandes courantes :

```bash
dotnet ef migrations add InitialCreate
```

Puis :

```bash
dotnet ef database update
```

Mental model :

```text
Model C#
 ↓
migration
 ↓
SQL / schema changes
 ↓
Database
```

---

# 33. Une migration contient quoi ?

Une migration décrit les changements nécessaires entre deux versions du modèle.

Exemple conceptuel :

```text
CREATE TABLE Users
ADD COLUMN Email
CREATE INDEX ...
```

EF Core génère le code de migration, mais le développeur doit comprendre ce que la migration va faire à la base.

---

# 34. Pourquoi ne pas simplement recréer la database ?

En développement, on peut parfois repartir de zéro.

En production, on doit conserver les données.

Les migrations permettent donc d'appliquer progressivement les changements :

```text
Version 1
 ↓
Migration 1
 ↓
Version 2
 ↓
Migration 2
 ↓
Version 3
```

---

# 35. Migration destructive

Attention aux changements comme :

```text
supprimer une colonne
renommer une colonne
changer un type
```

Une migration mal conçue peut entraîner une perte de données.

Avant une migration destructive, il faut réfléchir à la stratégie :

```text
backup
migration progressive
copie des données
dépréciation
déploiement en plusieurs étapes
```

---

# 36. Seeding

Le seeding permet d'initialiser des données.

Exemple :

```csharp
modelBuilder.Entity<User>().HasData(
    new User
    {
        Id = 1,
        Name = "Admin"
    });
```

Cela peut être utile pour des données de référence ou initiales.

---

# 37. Seed data vs données applicatives

Il faut distinguer :

```text
reference data
```

de :

```text
business data
```

Exemples de données de référence :

```text
roles
countries
statuses
categories
```

Créer des données utilisateurs réelles via un seeder statique peut être beaucoup moins adapté.

---

# 38. Tracking et `Update()`

Piège :

```csharp
context.Users.Update(user);
```

ne signifie pas simplement :

```text
"EF sait exactement quelle propriété j'ai modifiée"
```

Avec une entité détachée, `Update()` peut marquer l'entité et son graphe comme modifiés selon le contexte.

Cela peut générer plus d'UPDATE que nécessaire.

Si l'entité est déjà trackée :

```csharp
var user = await context.Users
    .FirstAsync(u => u.Id == id);

user.Name = "New name";

await context.SaveChangesAsync();
```

est souvent préférable.

---

# 39. `FindAsync`

Exemple :

```csharp
var user = await context.Users
    .FindAsync(id);
```

`Find` est conçu pour rechercher une entité par sa clé primaire et peut notamment tirer parti du Change Tracker avant d'interroger la base.

C'est différent d'une requête LINQ générale.

---

# 40. `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`

### `First`

Retourne le premier élément.

Lève une exception si aucun élément n'est trouvé.

### `FirstOrDefault`

Retourne le premier élément ou la valeur par défaut.

### `Single`

Attend exactement un élément.

Lève une exception si :

```text
0 résultat
ou
plusieurs résultats
```

### `SingleOrDefault`

Accepte :

```text
0 ou 1 résultat
```

mais lève une exception s'il y en a plusieurs.

Mental model :

```text
First
→ le premier

Single
→ exactement un
```

---

# 41. `Any()` vs `Count()`

Pour vérifier l'existence :

```csharp
await context.Users
    .AnyAsync(u => u.Email == email);
```

est généralement plus adapté que :

```csharp
await context.Users
    .CountAsync(u => u.Email == email) > 0;
```

L'intention est :

```text
Any
→ existe-t-il au moins un élément ?
```

---

# 42. Pagination

Mauvais scénario :

```csharp
var users = await context.Users
    .ToListAsync();
```

si la table contient des millions de lignes.

Pagination classique :

```csharp
var users = await context.Users
    .OrderBy(u => u.Id)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

Mental model :

```text
page
 ↓
Skip
 ↓
Take
 ↓
SQL
```

Pour de très gros volumes, la keyset pagination peut être plus performante que `Skip/Take` selon le cas.

---

# 43. Index

Un index permet notamment d'accélérer certaines recherches.

Exemple conceptuel :

```text
Users
----------------
Id
Email
Name

Index sur Email
```

Une requête :

```sql
WHERE Email = '...'
```

peut alors bénéficier de l'index.

Mais les index ont aussi un coût :

```text
stockage
INSERT
UPDATE
DELETE
maintenance
```

Il faut donc choisir les index selon les requêtes réelles.

---

# 44. Transactions

Une transaction permet de regrouper plusieurs opérations dans une unité atomique.

Mental model :

```text
BEGIN
 ↓
operation A
 ↓
operation B
 ↓
COMMIT
```

ou :

```text
ROLLBACK
```

si quelque chose échoue.

Avec EF Core :

```csharp
await using var transaction =
    await context.Database.BeginTransactionAsync();

try
{
    ...
    await context.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

---

# 45. `SaveChanges` et transaction

EF Core fournit déjà des garanties transactionnelles autour de certaines opérations de `SaveChanges`.

Il ne faut donc pas créer manuellement une transaction pour chaque appel sans raison.

Les transactions explicites deviennent pertinentes lorsque plusieurs opérations doivent être coordonnées selon un même périmètre transactionnel.

---

# 46. Concurrency

Deux utilisateurs peuvent modifier la même donnée.

Exemple :

```text
User A lit Product
User B lit Product

User A modifie
User B modifie

Qui gagne ?
```

On peut utiliser notamment la concurrence optimiste.

Un mécanisme courant est un token de concurrence.

L'idée :

```text
version A
 ↓
modification
 ↓
vérifier que la version n'a pas changé
```

Si elle a changé :

```text
concurrency conflict
```

---

# 47. AsNoTracking et mise à jour

Question d'entretien :

**Pourquoi `AsNoTracking()` peut poser problème si je veux modifier l'entité ?**

Parce que l'entité n'est pas suivie normalement par le Change Tracker.

Si on récupère :

```csharp
var user = await context.Users
    .AsNoTracking()
    .FirstAsync();
```

puis :

```csharp
user.Name = "Ahlame";
```

EF Core ne suit pas automatiquement cette modification comme une entité trackée.

Il faudrait alors explicitement gérer l'état ou recharger/attacher l'entité selon le scénario.

---

# 48. Async EF Core

Préférer les méthodes asynchrones dans les applications web :

```csharp
ToListAsync()
FirstAsync()
FirstOrDefaultAsync()
SingleAsync()
AnyAsync()
SaveChangesAsync()
```

Cela est particulièrement important lorsque l'opération attend la base de données.

---

# 49. Attention au faux async

Écrire :

```csharp
Task.Run(() =>
    context.Users.ToList());
```

dans une API pour "rendre EF asynchrone" n'est pas la bonne approche.

Il faut utiliser les APIs asynchrones fournies par EF Core :

```csharp
await context.Users.ToListAsync();
```

Mental model :

```text
ToListAsync
→ vrai mécanisme async du provider

Task.Run(ToList)
→ déplacer du travail synchrone sur un thread
```

---

# 50. Performance : règle générale

Avant d'optimiser :

```text
mesurer
 ↓
identifier le problème
 ↓
optimiser
 ↓
mesurer à nouveau
```

Les problèmes fréquents :

```text
N+1
trop de colonnes
absence d'index
pagination incorrecte
requêtes répétées
chargement de gros graphes
matérialisation trop tôt
```

---

# 51. Exemple de mauvaise requête

```csharp
var users = await context.Users
    .ToListAsync();

var result = users
    .Where(u => u.IsActive)
    .Select(u => new UserDto
    {
        Id = u.Id,
        Name = u.Name
    })
    .ToList();
```

Si la table contient beaucoup de données, on a chargé inutilement tous les utilisateurs.

Préférer :

```csharp
var result = await context.Users
    .Where(u => u.IsActive)
    .Select(u => new UserDto
    {
        Id = u.Id,
        Name = u.Name
    })
    .ToListAsync();
```

Le filtrage et la projection peuvent alors être traduits côté database.

---

# 52. Question d'entretien : pourquoi `IQueryable` peut être dangereux ?

Parce qu'une requête peut rester composable très longtemps et être exécutée plus tard.

Une méthode qui retourne :

```csharp
IQueryable<User>
```

expose potentiellement la construction de la requête à son appelant.

Il faut donc décider consciemment où doit se situer la frontière de requêtage.

Une méthode qui retourne :

```csharp
Task<List<UserDto>>
```

matérialise clairement le résultat.

Le bon choix dépend de l'architecture.

---

# 53. `Include` vs projection

### Include

```csharp
.Include(u => u.Orders)
```

Utile lorsqu'on a réellement besoin des entités et de leurs relations.

### Projection

```csharp
.Select(u => new UserDto
{
    Id = u.Id,
    OrderCount = u.Orders.Count
})
```

Souvent intéressante pour une API qui ne nécessite qu'un DTO.

Mental model :

```text
Include
→ charger des entités liées

Select
→ demander précisément les données nécessaires
```

---

# 54. Architecture avec EF Core

Une architecture peut ressembler à :

```text
API
 ↓
Application
 ↓
Domain
 ↓
Infrastructure
 ↓
EF Core
 ↓
Database
```

L'idée est d'éviter que le domaine soit fortement dépendant d'EF Core.

EF Core est généralement considéré comme une préoccupation d'infrastructure/persistance.

---

# 55. Repository Pattern et EF Core

EF Core fournit déjà :

```text
DbSet
DbContext
Unit of Work-like behavior
```

Créer systématiquement un repository générique du type :

```csharp
IRepository<T>
```

peut parfois simplement reproduire les abstractions déjà fournies par EF Core.

Le Repository Pattern peut toutefois être pertinent lorsqu'il représente une vraie abstraction métier ou de persistance.

La bonne réponse en entretien n'est donc pas :

> "Il faut toujours utiliser Repository."

Mais plutôt :

> "Je l'utilise lorsque l'abstraction apporte une vraie valeur ; EF Core fournit déjà beaucoup de fonctionnalités de repository et d'unit of work."

---

# 56. Questions d'entretien

### Qu'est-ce qu'EF Core ?

> Un ORM .NET permettant de mapper des objets C# vers une base relationnelle et de construire des requêtes avec LINQ.

### À quoi sert `DbContext` ?

> Il représente notamment une unité de travail avec la base, fournit les `DbSet`, suit les changements et permet de persister les modifications.

### Pourquoi `DbContext` est généralement Scoped ?

> Parce qu'il représente généralement une unité de travail liée à une requête et qu'il n'est pas conçu pour être partagé comme un singleton entre plusieurs requêtes concurrentes.

### Qu'est-ce que le Change Tracker ?

> C'est le mécanisme qui suit l'état des entités attachées au contexte afin qu'EF Core puisse déterminer les changements à persister.

### Pourquoi utiliser `AsNoTracking()` ?

> Pour les scénarios de lecture seule lorsque le suivi des entités n'est pas nécessaire, afin de réduire le travail du Change Tracker.

### Qu'est-ce qu'une migration ?

> Une représentation versionnée des changements du modèle qui permet de faire évoluer le schéma de la base de données.

### Qu'est-ce que N+1 ?

> Une situation où une requête initiale est suivie d'une requête supplémentaire pour chaque élément récupéré, entraînant potentiellement un grand nombre de requêtes.

### Pourquoi utiliser une projection ?

> Pour récupérer uniquement les données nécessaires plutôt que de charger une entité ou un graphe complet.

### `Any()` ou `Count() > 0` pour tester l'existence ?

> `Any()` exprime directement l'intention et permet généralement au provider de générer une requête adaptée à un simple test d'existence.

### Pourquoi `ToListAsync()` ?

> Pour matérialiser les résultats de la requête de manière asynchrone.

---

# 57. Questions pièges

## "AsNoTracking est toujours plus performant"

Pas forcément.

Il réduit le travail du Change Tracker, mais le bénéfice dépend du scénario.

---

## "Include est toujours la meilleure solution"

Non.

Pour une API, une projection vers un DTO peut être plus efficace et plus claire si on ne veut que quelques données.

---

## "EF Core évite de connaître SQL"

Faux.

Il faut comprendre SQL pour raisonner sur :

```text
JOIN
index
performance
transactions
cardinalités
```

---

## "ToList() ne fait rien de spécial"

Faux.

Dans une requête `IQueryable`, `ToList()` matérialise le résultat et déclenche généralement l'exécution de la requête.

---

## "DbContext est thread-safe"

Non.

Il ne faut pas utiliser simultanément une même instance `DbContext` depuis plusieurs threads.

---

## "Repository Pattern est obligatoire avec EF Core"

Non.

Il faut justifier son utilisation selon l'architecture et la valeur apportée.

---

# 58. Mini simulation d'entretien

### Recruteur

**Explique-moi ce qui se passe avec ce code :**

```csharp
var users = await context.Users
    .Where(u => u.IsActive)
    .Select(u => new UserDto
    {
        Id = u.Id,
        Name = u.Name
    })
    .ToListAsync();
```

### Réponse

> `context.Users` fournit une requête `IQueryable`. `Where` ajoute une condition à l'expression de requête et `Select` définit une projection vers `UserDto`. Comme la requête n'est pas encore matérialisée, EF Core peut construire la requête SQL correspondante. `ToListAsync()` déclenche ensuite l'exécution auprès de la base et retourne les résultats de manière asynchrone.

---

### Recruteur

**Pourquoi utiliser `AsNoTracking()` sur un GET ?**

### Réponse

> Si je fais uniquement de la lecture et que je ne compte pas modifier les entités dans le même contexte, je n'ai pas besoin que le Change Tracker les suive. `AsNoTracking()` peut donc réduire le coût du tracking.

---

### Recruteur

**Qu'est-ce que le problème N+1 ?**

### Réponse

> C'est lorsqu'une requête principale récupère une collection puis qu'une requête supplémentaire est exécutée pour chaque élément afin de récupérer ses relations. Avec 100 éléments, on peut ainsi se retrouver avec 101 requêtes. Je peux notamment utiliser une projection ou une stratégie de chargement adaptée pour éviter cela.

---

### Recruteur

**Pourquoi ne pas faire `ToListAsync()` immédiatement ?**

### Réponse

> Parce que cela matérialise immédiatement les résultats. Si j'ajoute ensuite des filtres ou une projection en mémoire, je risque de charger beaucoup plus de données que nécessaire. Je préfère généralement construire la requête puis la matérialiser une fois qu'elle représente réellement les données dont j'ai besoin.

---

# 59. Checklist EF Core

```text
[ ] ORM
[ ] DbContext
[ ] DbSet
[ ] Entity
[ ] Conventions
[ ] Fluent API
[ ] Data Annotations
[ ] OnModelCreating
[ ] Change Tracker
[ ] EntityState
[ ] SaveChanges
[ ] SaveChangesAsync
[ ] Tracking
[ ] AsNoTracking
[ ] IQueryable
[ ] LINQ translation
[ ] Deferred execution
[ ] Include
[ ] ThenInclude
[ ] Eager loading
[ ] Explicit loading
[ ] Lazy loading
[ ] N+1
[ ] Projection
[ ] Relationships
[ ] Foreign keys
[ ] Navigation properties
[ ] Cascade delete
[ ] Many-to-many
[ ] Migrations
[ ] Seeding
[ ] Transactions
[ ] Concurrency
[ ] Pagination
[ ] Indexes
[ ] Async queries
[ ] Repository Pattern
```

# À retenir

```text
DbContext
→ unité de travail EF Core

DbSet
→ ensemble d'entités

Change Tracker
→ suit les changements

SaveChanges
→ persiste les changements

AsNoTracking
→ lecture sans tracking

IQueryable
→ requête composable

ToListAsync
→ matérialise / exécute

Include
→ charge des relations

Select
→ projection vers les données nécessaires

N+1
→ trop de requêtes liées

Migration
→ évolution versionnée du schéma

Transaction
→ opérations atomiques

Concurrency
→ gérer les modifications concurrentes

Index
→ accélérer certaines recherches

DbContext
→ généralement Scoped
```

## Phrase à mémoriser

> **EF Core traduit mes expressions LINQ en opérations de persistance ; je dois surtout maîtriser le cycle de vie du DbContext, le Change Tracker, la différence entre requête et matérialisation, le chargement des relations et les conséquences SQL de mon code C#.**
