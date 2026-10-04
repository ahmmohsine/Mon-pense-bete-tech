# DbContext

Le `DbContext` est l'un des concepts centraux d'Entity Framework Core.

Pour bien le comprendre, il faut éviter de le réduire à :

```csharp
context.Hotels.ToListAsync();
```

Le `DbContext` joue plusieurs rôles : il permet d'accéder aux entités, construit et exécute les requêtes, suit l'état des entités et coordonne la persistance des changements.

---

# 1. Définition

Un `DbContext` représente principalement une **session de travail avec la base de données**.

Exemple :

```csharp
public class ApplicationDbContext : DbContext
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
Application
     |
     v
DbContext
     |
     +---- DbSet<Hotel>
     +---- DbSet<Room>
     +---- DbSet<User>
     |
     v
Database
```

Le `DbContext` fait donc partie de la couche qui relie ton application au système de persistance.

---

# 2. Les responsabilités du DbContext

Un `DbContext` s'occupe notamment de :

- fournir l'accès aux entités ;
- construire des requêtes ;
- transmettre les requêtes au provider EF Core ;
- suivre les entités ;
- détecter les modifications ;
- préparer les opérations `INSERT`, `UPDATE` et `DELETE` ;
- exécuter `SaveChanges()` ou `SaveChangesAsync()` ;
- appliquer la configuration du modèle.

On peut résumer :

```text
DbContext
   |
   +-- Query
   +-- Tracking
   +-- Model
   +-- Changes
   +-- Persistence
```

---

# 3. `DbSet<T>`

Un `DbSet<T>` représente le point d'accès EF Core à un type d'entité.

Exemple :

```csharp
public DbSet<Hotel> Hotels => Set<Hotel>();
```

On peut ensuite écrire :

```csharp
var hotels = await context.Hotels
    .ToListAsync();
```

Mais attention :

```csharp
context.Hotels
```

n'est pas une `List<Hotel>`.

Une `List<Hotel>` contient des objets déjà présents en mémoire.

Un `DbSet<Hotel>` permet notamment de construire une requête destinée au provider EF Core.

Mentalement :

```text
List<Hotel>
    |
    v
Objets en mémoire

DbSet<Hotel>
    |
    v
Point d'entrée EF Core
    |
    v
Requête potentiellement traduite en SQL
```

---

# 4. Le cycle d'une requête

Prenons :

```csharp
var hotels = await context.Hotels
    .Where(h => h.Stars >= 4)
    .ToListAsync();
```

Il faut visualiser plusieurs étapes :

```text
context.Hotels
      |
      v
DbSet<Hotel>
      |
      v
Where(...)
      |
      v
IQueryable<Hotel>
      |
      v
ToListAsync()
      |
      v
EF Core
      |
      v
SQL
      |
      v
Database
      |
      v
Résultats
      |
      v
Hotel objects
```

C'est l'un des modèles mentaux les plus importants d'EF Core.

---

# 5. Pourquoi `IQueryable` est important

Lorsque tu écris :

```csharp
var query = context.Hotels
    .Where(h => h.Stars >= 4);
```

tu n'as pas nécessairement récupéré les hôtels.

Tu as construit une représentation de requête.

Puis :

```csharp
var hotels = await query.ToListAsync();
```

demande les résultats.

On peut donc penser :

```text
Where
Select
OrderBy
Take
Skip
    |
    v
Construction de la requête

ToListAsync
FirstAsync
SingleAsync
AnyAsync
CountAsync
    |
    v
Exécution
```

---

# 6. `DbContext` et Change Tracker

Le `DbContext` possède un **Change Tracker**.

Son rôle est notamment de connaître l'état des entités qu'il suit.

Exemple :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == 1);

hotel.Name = "Hotel Central";

await context.SaveChangesAsync();
```

Le `DbContext` sait que l'objet `hotel` est associé à une entité de la base.

Après la modification :

```csharp
hotel.Name = "Hotel Central";
```

EF Core peut détecter que l'entité a changé.

Il peut alors préparer un `UPDATE`.

Mentalement :

```text
Database
   |
   v
DbContext
   |
   v
Hotel
   |
   v
Modification en C#
   |
   v
Change Tracker détecte
   |
   v
SaveChangesAsync()
   |
   v
UPDATE SQL
```

---

# 7. Les états d'une entité

Le Change Tracker peut notamment utiliser des états comme :

```text
Detached
Unchanged
Added
Modified
Deleted
```

## Detached

L'entité n'est pas suivie par ce `DbContext`.

```text
Objet C#
   X
DbContext
```

## Unchanged

L'entité est suivie et aucune modification n'a été détectée.

```text
Hotel
 |
 v
DbContext
 |
 v
Unchanged
```

## Added

L'entité doit être insérée.

```csharp
context.Hotels.Add(hotel);
```

État :

```text
Added
```

Puis :

```csharp
await context.SaveChangesAsync();
```

peut produire :

```sql
INSERT ...
```

## Modified

L'entité existante a été modifiée.

```text
Modified
```

Puis :

```sql
UPDATE ...
```

## Deleted

L'entité est marquée pour suppression.

```csharp
context.Hotels.Remove(hotel);
```

Puis :

```sql
DELETE ...
```

---

# 8. `SaveChangesAsync()`

Une erreur fréquente consiste à penser que :

```csharp
hotel.Name = "New Name";
```

modifie immédiatement la base.

Ce n'est pas le principe.

Cette instruction modifie d'abord l'objet C#.

Puis EF Core détecte la modification.

Enfin :

```csharp
await context.SaveChangesAsync();
```

demande la persistance.

Mentalement :

```text
Modification C#
       |
       v
Change Tracker
       |
       v
SaveChangesAsync()
       |
       v
SQL
       |
       v
Database
```

---

# 9. Ajouter une entité

Exemple :

```csharp
var hotel = new Hotel
{
    Name = "Hotel Central"
};

context.Hotels.Add(hotel);

await context.SaveChangesAsync();
```

Le cycle est :

```text
new Hotel
   |
   v
Add()
   |
   v
Added
   |
   v
SaveChangesAsync()
   |
   v
INSERT
```

---

# 10. Modifier une entité

Exemple :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == id);

hotel.Name = "New Name";

await context.SaveChangesAsync();
```

Le contexte suit l'entité.

Après la modification :

```text
Unchanged
    |
    | Name modified
    v
Modified
    |
    | SaveChangesAsync
    v
UPDATE
```

---

# 11. Supprimer une entité

Exemple :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == id);

context.Hotels.Remove(hotel);

await context.SaveChangesAsync();
```

Mentalement :

```text
Hotel
  |
  v
Remove()
  |
  v
Deleted
  |
  v
SaveChangesAsync()
  |
  v
DELETE
```

---

# 12. `FindAsync()`

Pour rechercher une entité par clé primaire, tu peux utiliser :

```csharp
var hotel = await context.Hotels
    .FindAsync(id);
```

`FindAsync()` est particulièrement intéressant parce qu'EF Core peut d'abord vérifier si l'entité correspondante est déjà suivie par le contexte avant de chercher dans la base.

Conceptuellement :

```text
FindAsync(id)
     |
     v
Entité déjà suivie ?
   /         oui        non
 |           |
 v           v
retourner   DB
            |
            v
         entité
```

---

# 13. `FirstAsync()` vs `FindAsync()`

Ces deux méthodes ne répondent pas exactement au même besoin.

### `FindAsync`

```csharp
await context.Hotels.FindAsync(id);
```

À utiliser principalement pour une recherche par clé primaire.

### `FirstAsync`

```csharp
await context.Hotels
    .FirstAsync(h => h.Id == id);
```

Construit une requête selon la condition.

Mentalement :

```text
FindAsync
    =
Recherche par clé primaire avec comportement lié au tracking

FirstAsync
    =
Exécution d'une requête LINQ
```

---

# 14. Configuration du modèle

Le `DbContext` peut également configurer la manière dont les entités correspondent à la base.

La méthode importante est :

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);

    modelBuilder.Entity<Hotel>()
        .Property(h => h.Name)
        .IsRequired();
}
```

On peut y configurer :

- les clés ;
- les propriétés ;
- les longueurs ;
- les relations ;
- les clés étrangères ;
- les index ;
- les contraintes ;
- les conversions ;
- certaines données initiales.

Mentalement :

```text
C# Entities
     |
     v
OnModelCreating
     |
     v
EF Core Model
     |
     v
Database Mapping
```

---

# 15. Data Annotations vs Fluent API

Il existe notamment deux grandes manières de configurer le modèle.

## Data Annotations

Exemple :

```csharp
public class Hotel
{
    public int Id { get; set; }

    [Required]
    [MaxLength(200)]
    public string Name { get; set; } = string.Empty;
}
```

## Fluent API

Exemple :

```csharp
modelBuilder.Entity<Hotel>()
    .Property(h => h.Name)
    .IsRequired()
    .HasMaxLength(200);
```

La Fluent API permet généralement une configuration plus complète et centralisée.

---

# 16. `OnModelCreating`

Une configuration plus réaliste peut ressembler à :

```csharp
protected override void OnModelCreating(
    ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Hotel>(entity =>
    {
        entity.HasKey(h => h.Id);

        entity.Property(h => h.Name)
            .IsRequired()
            .HasMaxLength(200);
    });
}
```

Le bloc :

```csharp
entity => { ... }
```

configure spécifiquement l'entité `Hotel`.

---

# 17. Relations dans le DbContext

Exemple :

```csharp
modelBuilder.Entity<Room>()
    .HasOne(r => r.Hotel)
    .WithMany(h => h.Rooms)
    .HasForeignKey(r => r.HotelId);
```

Lis-le comme une phrase :

```text
Room
  |
  | HasOne
  v
Hotel
  |
  | WithMany
  v
Rooms
```

Donc :

```text
Hotel 1 ---- N Room
```

La clé étrangère est :

```csharp
HotelId
```

---

# 18. `DbContext` et Dependency Injection

Dans ASP.NET Core, on enregistre généralement le contexte dans le conteneur DI :

```csharp
builder.Services.AddDbContext<ApplicationDbContext>(
    options =>
        options.UseSqlServer(connectionString));
```

Puis une classe peut recevoir le contexte par injection :

```csharp
public class HotelService(
    ApplicationDbContext context)
{
}
```

Le framework fournit alors le `DbContext`.

Mentalement :

```text
DI Container
     |
     v
ApplicationDbContext
     |
     v
HotelService
```

---

# 19. Pourquoi `Scoped` ?

`AddDbContext()` enregistre généralement le contexte avec une durée de vie **Scoped**.

Dans une API :

```text
HTTP Request #1
      |
      v
DbContext #1

HTTP Request #2
      |
      v
DbContext #2
```

On évite ainsi de partager le même contexte entre toutes les requêtes HTTP.

Un `DbContext` n'est pas conçu pour être un Singleton global.

---

# 20. DbContext et thread-safety

Un `DbContext` n'est pas conçu pour être utilisé simultanément par plusieurs threads.

Il faut donc éviter des scénarios comme :

```csharp
await Task.WhenAll(
    context.Hotels.ToListAsync(),
    context.Rooms.ToListAsync()
);
```

avec le même `DbContext` lorsque les opérations sont exécutées simultanément.

Le message important :

> Un `DbContext` représente une unité de travail et n'est pas un objet partagé pour des opérations concurrentes.

---

# 21. Durée de vie du DbContext

Dans une application web, le modèle courant est :

```text
Request
   |
   +-- Controller
   |
   +-- Service
   |
   +-- DbContext
   |
   +-- SaveChanges
   |
   v
End Request
```

Le contexte est ensuite disposé.

Il faut éviter de garder un `DbContext` vivant beaucoup plus longtemps que nécessaire.

---

# 22. DbContext et Repository

Dans une architecture utilisant des repositories :

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
DbContext
    |
    v
Database
```

Le repository peut utiliser le `DbContext`.

Mais EF Core fournit déjà beaucoup d'abstractions d'accès aux données.

Il ne faut donc pas créer un repository générique uniquement par réflexe.

La vraie question est :

> Est-ce que cette abstraction apporte une valeur architecturale réelle au projet ?

---

# 23. `DbContext` n'est pas la base de données

Très important :

```text
DbContext != Database
```

Le `DbContext` est un objet de ton application.

La base de données est un système externe de persistance.

```text
Application
     |
     v
DbContext
     |
     v
EF Core Provider
     |
     v
Database
```

Par exemple, selon le provider, EF Core peut communiquer avec SQL Server, PostgreSQL, SQLite, etc.

---

# 24. Provider

EF Core utilise un **provider** pour communiquer avec un système de base de données particulier.

Exemple SQL Server :

```csharp
options.UseSqlServer(connectionString);
```

Le provider connaît les particularités du moteur cible.

Mentalement :

```text
EF Core
   |
   v
Provider
   |
   v
SQL Server / PostgreSQL / SQLite / ...
```

---

# 25. `SaveChanges()` vs `SaveChangesAsync()`

Version synchrone :

```csharp
context.SaveChanges();
```

Version asynchrone :

```csharp
await context.SaveChangesAsync();
```

Dans une application web moderne, les opérations I/O sont généralement écrites en async afin d'éviter de bloquer inutilement un thread pendant l'attente de la base.

---

# 26. CancellationToken

Une méthode peut propager un token :

```csharp
public async Task<List<Hotel>> GetHotelsAsync(
    CancellationToken cancellationToken)
{
    return await context.Hotels
        .ToListAsync(cancellationToken);
}
```

Le flux peut être :

```text
HTTP Request
     |
     v
Controller
     |
     v
Service
     |
     v
EF Core
     |
     v
Database
```

Si la requête HTTP est annulée, le token peut être propagé jusqu'à EF Core.

---

# 27. Tracking vs No Tracking

Pour une lecture destinée uniquement à afficher des données :

```csharp
var hotels = await context.Hotels
    .AsNoTracking()
    .ToListAsync();
```

Pour une entité que tu veux modifier et sauvegarder avec le même contexte :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == id);

hotel.Name = "New Name";

await context.SaveChangesAsync();
```

Mentalement :

```text
Lecture pure
    -> AsNoTracking()

Lecture + modification
    -> Tracking
```

Ce n'est pas une règle absolue, mais c'est une bonne première règle mentale.

---

# 28. Projection et DbContext

Dans une API, il est souvent préférable de ne pas récupérer toute l'entité si tu n'en as besoin que d'une partie.

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
Database
   |
   | seulement les colonnes nécessaires
   v
DTO
```

Cela peut réduire les données transférées et évite aussi de faire circuler inutilement des entités de persistance jusqu'à la couche API.

---

# 29. `DbContext` et transactions

`SaveChanges` gère la persistance des changements dans le contexte transactionnel approprié.

Pour des opérations nécessitant une transaction explicite :

```csharp
await using var transaction =
    await context.Database.BeginTransactionAsync();

try
{
    // opérations

    await context.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

Une transaction permet de raisonner en :

```text
Tout réussit
    -> Commit

Une étape échoue
    -> Rollback
```

Il ne faut cependant pas ajouter des transactions explicites partout sans comprendre le besoin.

---

# 30. Erreurs fréquentes

## 1. Créer un DbContext avec `new` dans chaque service

Mauvaise approche :

```csharp
var context = new ApplicationDbContext(...);
```

Dans ASP.NET Core, il est généralement préférable de laisser le conteneur DI gérer sa création et sa durée de vie.

---

## 2. Enregistrer DbContext en Singleton

À éviter :

```csharp
services.AddSingleton<ApplicationDbContext>();
```

Le `DbContext` n'est pas conçu pour être partagé globalement.

---

## 3. Utiliser un DbContext en parallèle

Éviter les opérations concurrentes sur la même instance.

```text
Même DbContext
      |
      +-- Query A
      |
      +-- Query B simultanée
```

Ce n'est pas le modèle d'utilisation prévu.

---

## 4. Penser que modifier un objet modifie automatiquement SQL

```csharp
hotel.Name = "New Name";
```

ne suffit pas à persister la modification.

Il faut que l'entité soit suivie de manière appropriée et que :

```csharp
await context.SaveChangesAsync();
```

soit exécuté.

---

## 5. Récupérer toute la table inutilement

Éviter :

```csharp
var hotels = await context.Hotels
    .ToListAsync();

var filtered = hotels
    .Where(h => h.Stars >= 4)
    .ToList();
```

si le filtrage peut être réalisé par la base :

```csharp
var hotels = await context.Hotels
    .Where(h => h.Stars >= 4)
    .ToListAsync();
```

---

# 31. Comparaison des rôles

| Élément | Rôle |
|---|---|
| `DbContext` | Coordonne le travail EF Core |
| `DbSet<T>` | Point d'accès à une entité |
| LINQ | Construit les requêtes |
| `IQueryable<T>` | Représente une requête composable |
| Change Tracker | Suit l'état des entités |
| `SaveChangesAsync()` | Persiste les changements |
| `Include()` | Charge des navigations |
| `Select()` | Projette les données |
| Migration | Fait évoluer le schéma |
| Provider | Communique avec le moteur de base |

---

# 32. Règle mentale complète

Quand tu vois :

```csharp
context.Hotels
```

pense :

> « Je commence depuis le DbSet des hôtels. »

Quand tu vois :

```csharp
.Where(...)
```

pense :

> « Je construis une requête. »

Quand tu vois :

```csharp
.Select(...)
```

pense :

> « Je choisis ce que je veux récupérer. »

Quand tu vois :

```csharp
.ToListAsync()
```

pense :

> « J'exécute la requête. »

Quand tu vois :

```csharp
.AsNoTracking()
```

pense :

> « Je n'ai pas besoin que le contexte suive les entités. »

Quand tu vois :

```csharp
.Add(...)
```

pense :

> « Je marque une nouvelle entité comme Added. »

Quand tu vois :

```csharp
.Remove(...)
```

pense :

> « Je marque l'entité comme Deleted. »

Quand tu vois :

```csharp
.SaveChangesAsync()
```

pense :

> « Je demande à EF Core de persister les changements suivis. »

---

# 33. À retenir

1. `DbContext` représente principalement une session/unité de travail avec la base.
2. Il coordonne les requêtes, le tracking et la persistance.
3. `DbSet<T>` est le point d'accès EF Core à une entité.
4. `DbSet<T>` n'est pas une `List<T>`.
5. LINQ permet de construire des requêtes.
6. `IQueryable<T>` permet de composer une requête destinée au provider.
7. `ToListAsync()` déclenche généralement l'exécution de la requête.
8. Le Change Tracker suit l'état des entités.
9. `Add()` marque une entité comme `Added`.
10. `Remove()` marque une entité comme `Deleted`.
11. Une modification d'une entité suivie peut être détectée automatiquement.
12. `SaveChangesAsync()` demande la persistance des changements.
13. `FindAsync()` est adapté aux recherches par clé primaire.
14. `OnModelCreating()` permet de configurer le modèle.
15. `AddDbContext()` intègre généralement le contexte avec la DI.
16. Un `DbContext` est généralement Scoped dans ASP.NET Core.
17. Un `DbContext` n'est pas thread-safe.
18. Il ne faut pas exécuter plusieurs opérations concurrentes sur le même contexte.
19. `AsNoTracking()` est utile pour les lectures qui n'ont pas besoin de tracking.
20. Une projection avec `Select()` peut être préférable à la récupération d'entités complètes.
21. Le provider EF Core permet de cibler différents moteurs de base de données.
22. Le `DbContext` n'est pas la base de données.
23. Il faut réfléchir à la durée de vie du contexte.
24. Il faut comprendre ce que le LINQ devient réellement côté SQL.

---

# Questions d'entretien

### 1. Qu'est-ce qu'un `DbContext` ?

C'est le principal objet de session de travail d'EF Core. Il coordonne les requêtes, le suivi des entités et la persistance des changements.

### 2. Pourquoi le `DbContext` est-il généralement Scoped ?

Parce qu'il est généralement associé à une unité de travail correspondant à une requête HTTP et qu'il n'est pas conçu pour être partagé globalement.

### 3. Un `DbContext` est-il thread-safe ?

Non. Une même instance ne doit pas être utilisée simultanément par plusieurs opérations concurrentes.

### 4. Quelle est la différence entre `DbSet<T>` et `List<T>` ?

`List<T>` représente des objets en mémoire. `DbSet<T>` représente un point d'accès EF Core permettant notamment de construire des requêtes destinées à la base.

### 5. Quel est le rôle du Change Tracker ?

Il suit l'état des entités afin qu'EF Core puisse détecter les changements et préparer les opérations de persistance.

### 6. Que fait `SaveChangesAsync()` ?

Il demande à EF Core de persister dans la base les changements suivis par le `DbContext`.

### 7. Pourquoi `AsNoTracking()` ?

Pour éviter le suivi des entités lorsque celui-ci n'est pas nécessaire, notamment dans certaines lectures.

### 8. Quelle différence entre `FindAsync()` et `FirstAsync()` ?

`FindAsync()` est spécialement adapté à une recherche par clé primaire et peut exploiter une entité déjà suivie par le contexte. `FirstAsync()` construit et exécute une requête selon une condition.

### 9. Où configure-t-on les relations EF Core ?

Notamment dans `OnModelCreating()` avec la Fluent API.

### 10. Pourquoi ne faut-il pas mettre `DbContext` en Singleton ?

Parce qu'il possède un état de tracking lié à son unité de travail et qu'il n'est pas conçu pour être partagé entre toutes les requêtes ou utilisé simultanément.

### 11. Pourquoi utiliser `Select()` dans une API ?

Pour projeter uniquement les données nécessaires, par exemple vers un DTO, plutôt que de récupérer inutilement une entité complète.

### 12. Quel est le lien entre EF Core et SQL ?

EF Core reçoit notamment des expressions LINQ, les traduit via son provider vers le langage SQL approprié, puis communique avec la base.

---

# Phrase à retenir

> **Le DbContext est ma session de travail EF Core : il construit les requêtes, suit mes entités et, avec SaveChangesAsync(), synchronise les changements avec la base.**
