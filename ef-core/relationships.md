# EF Core — Relationships

Les relations sont au cœur d'une base de données relationnelle.

Dans EF Core, il faut comprendre comment faire correspondre :

```text
Classes C#
    ↕
Relations entre objets
    ↕
Foreign Keys
    ↕
Relations SQL
```

L'objectif n'est pas seulement de savoir écrire `HasOne()` ou `HasMany()`, mais de comprendre ce que signifie réellement une relation entre deux entités.

---

# 1. Les trois relations principales

EF Core permet principalement de représenter :

```text
1 - 1    One-to-One
1 - N    One-to-Many
N - N    Many-to-Many
```

Exemples :

```text
User 1 ---- 1 UserProfile

Hotel 1 ---- N Room

Student N ---- N Course
```

---

# 2. One-to-Many

C'est probablement la relation que tu rencontreras le plus souvent.

Exemple :

```text
Hotel
  |
  +-- Room
  +-- Room
  +-- Room
```

Un hôtel possède plusieurs chambres.

Une chambre appartient à un hôtel.

Donc :

```text
Hotel 1 ---- N Room
```

---

# 3. Les classes C#

On peut représenter la relation ainsi :

```csharp
public class Hotel
{
    public int Id { get; set; }

    public string Name { get; set; } = string.Empty;

    public ICollection<Room> Rooms { get; set; }
        = new List<Room>();
}
```

Et :

```csharp
public class Room
{
    public int Id { get; set; }

    public string Number { get; set; } = string.Empty;

    public int HotelId { get; set; }

    public Hotel Hotel { get; set; } = null!;
}
```

On trouve donc deux notions :

```text
HotelId
    =
Foreign Key

Hotel
    =
Navigation Property
```

---

# 4. Foreign Key

La Foreign Key permet de relier une ligne à une autre.

Exemple SQL conceptuel :

```text
Hotels
+----+-------------+
| Id | Name        |
+----+-------------+
| 1  | Hotel A     |
| 2  | Hotel B     |
+----+-------------+

Rooms
+----+--------+---------+
| Id | Number | HotelId |
+----+--------+---------+
| 1  | 101    | 1       |
| 2  | 102    | 1       |
| 3  | 201    | 2       |
+----+--------+---------+
```

Ici :

```text
Room 101 -> HotelId = 1
Room 102 -> HotelId = 1
Room 201 -> HotelId = 2
```

Donc :

```text
Hotel 1
   |
   +-- Room 101
   +-- Room 102

Hotel 2
   |
   +-- Room 201
```

---

# 5. Navigation Property

Une navigation property permet de naviguer entre les objets C#.

Dans `Room` :

```csharp
public Hotel Hotel { get; set; } = null!;
```

Tu peux conceptuellement faire :

```csharp
room.Hotel
```

Dans `Hotel` :

```csharp
public ICollection<Room> Rooms { get; set; }
    = new List<Room>();
```

Tu peux alors faire :

```csharp
hotel.Rooms
```

Mentalement :

```text
room.Hotel
    =
"Donne-moi l'hôtel associé"

hotel.Rooms
    =
"Donne-moi les chambres associées"
```

---

# 6. Configurer la relation avec Fluent API

Dans `OnModelCreating()` :

```csharp
modelBuilder.Entity<Room>()
    .HasOne(r => r.Hotel)
    .WithMany(h => h.Rooms)
    .HasForeignKey(r => r.HotelId);
```

Lis cette expression comme une phrase :

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

Puis :

```text
Room.HotelId
      |
      v
Hotel.Id
```

---

# 7. Comprendre `HasOne`

```csharp
.HasOne(r => r.Hotel)
```

signifie :

> Une `Room` possède une navigation vers un `Hotel`.

Donc :

```text
Room ----> Hotel
```

---

# 8. Comprendre `WithMany`

```csharp
.WithMany(h => h.Rooms)
```

signifie :

> Un `Hotel` peut être associé à plusieurs `Room`.

Donc :

```text
Hotel
 |
 +-- Room
 +-- Room
 +-- Room
```

---

# 9. Comprendre `HasForeignKey`

```csharp
.HasForeignKey(r => r.HotelId)
```

indique quelle propriété porte la clé étrangère.

Donc :

```text
Room.HotelId
      |
      v
Hotel.Id
```

---

# 10. Règle mentale `HasOne / WithMany`

Pour :

```text
Hotel 1 ---- N Room
```

on peut retenir :

```csharp
modelBuilder.Entity<Room>()
    .HasOne(r => r.Hotel)
    .WithMany(h => h.Rooms)
    .HasForeignKey(r => r.HotelId);
```

Phrase mentale :

> Une Room a un Hotel, et un Hotel a plusieurs Rooms.

---

# 11. One-to-One

Exemple :

```text
User 1 ---- 1 UserProfile
```

Une méthode classique :

```csharp
public class User
{
    public int Id { get; set; }

    public UserProfile Profile { get; set; } = null!;
}
```

```csharp
public class UserProfile
{
    public int Id { get; set; }

    public int UserId { get; set; }

    public User User { get; set; } = null!;
}
```

Configuration :

```csharp
modelBuilder.Entity<User>()
    .HasOne(u => u.Profile)
    .WithOne(p => p.User)
    .HasForeignKey<UserProfile>(p => p.UserId);
```

Lis-le :

```text
User
  |
  | HasOne
  v
Profile
  |
  | WithOne
  v
User
```

---

# 12. Pourquoi préciser `HasForeignKey<UserProfile>` ?

Dans une relation One-to-One, EF Core peut parfois avoir besoin de savoir quelle entité porte réellement la Foreign Key.

Ici :

```csharp
.HasForeignKey<UserProfile>(p => p.UserId)
```

signifie :

> La Foreign Key de cette relation se trouve dans `UserProfile`.

Donc :

```text
UserProfile.UserId
        |
        v
User.Id
```

---

# 13. Many-to-Many

Exemple :

```text
Student N ---- N Course
```

Un étudiant peut suivre plusieurs cours.

Un cours peut être suivi par plusieurs étudiants.

Conceptuellement :

```text
Student
  |   |    v   v
Course Course
```

En SQL relationnel, on utilise généralement une table de jointure.

```text
Students
    |
    v
StudentCourses
    ^
    |
Courses
```

---

# 14. Table de jointure

Exemple :

```text
StudentCourses
+-----------+----------+
| StudentId | CourseId |
+-----------+----------+
| 1         | 10       |
| 1         | 20       |
| 2         | 10       |
+-----------+----------+
```

Cela signifie :

```text
Student 1 -> Course 10
Student 1 -> Course 20
Student 2 -> Course 10
```

---

# 15. Many-to-Many avec EF Core

EF Core peut représenter directement une relation Many-to-Many.

Exemple :

```csharp
public class Student
{
    public int Id { get; set; }

    public ICollection<Course> Courses { get; set; }
        = new List<Course>();
}
```

```csharp
public class Course
{
    public int Id { get; set; }

    public ICollection<Student> Students { get; set; }
        = new List<Student>();
}
```

Puis :

```csharp
modelBuilder.Entity<Student>()
    .HasMany(s => s.Courses)
    .WithMany(c => c.Students);
```

EF Core peut gérer la table de jointure automatiquement.

---

# 16. Many-to-Many avec données supplémentaires

Supposons que la relation elle-même possède des informations :

```text
Student
Course
StudentCourse
```

Et :

```text
StudentCourse
    |
    +-- EnrollmentDate
    +-- Grade
```

Dans ce cas, il est souvent préférable de représenter explicitement la table de jointure.

```csharp
public class StudentCourse
{
    public int StudentId { get; set; }

    public int CourseId { get; set; }

    public DateTime EnrollmentDate { get; set; }

    public Student Student { get; set; } = null!;

    public Course Course { get; set; } = null!;
}
```

Mentalement :

```text
Student
   |
   v
StudentCourse
   |
   v
Course
```

La relation devient alors une vraie entité du modèle.

---

# 17. Required vs Optional

Une relation peut être :

```text
Required
Optional
```

Exemple obligatoire :

```csharp
public int HotelId { get; set; }
```

Une `Room` doit avoir un `HotelId`.

Exemple optionnel :

```csharp
public int? HotelId { get; set; }
```

Ici :

```text
HotelId
   |
   v
nullable
```

La chambre peut potentiellement ne pas avoir de relation vers un hôtel.

---

# 18. Required Navigation

Une navigation peut également exprimer une relation attendue :

```csharp
public Hotel Hotel { get; set; } = null!;
```

Le `null!` signifie principalement :

> « Je demande au compilateur de ne pas me signaler cette propriété comme potentiellement null ici. »

Il ne crée pas magiquement l'objet.

C'est important avec les Nullable Reference Types.

---

# 19. Optional Navigation

Pour une navigation réellement optionnelle :

```csharp
public Hotel? Hotel { get; set; }
```

Cela signifie :

```text
Hotel peut être null
```

C'est différent de :

```csharp
public Hotel Hotel { get; set; } = null!;
```

---

# 20. Convention EF Core

EF Core possède des conventions.

Par exemple :

```csharp
public int HotelId { get; set; }
```

et :

```csharp
public Hotel Hotel { get; set; } = null!;
```

donnent suffisamment d'informations à EF Core dans beaucoup de cas pour comprendre la relation.

On peut donc parfois ne pas avoir besoin d'écrire explicitement :

```csharp
.HasForeignKey(...)
```

Mais la Fluent API reste utile lorsque le modèle devient plus complexe ou lorsque tu veux rendre la configuration explicite.

---

# 21. Pourquoi connaître les conventions ?

Il faut éviter deux extrêmes :

```text
"Je laisse EF tout deviner."
```

et :

```text
"Je configure absolument chaque propriété."
```

L'objectif est de savoir :

```text
Ce que EF peut déduire
        +
Ce que je dois configurer explicitement
```

---

# 22. `Include()` et relations

Une relation définie dans le modèle ne signifie pas automatiquement que toutes les données liées seront chargées dans chaque requête.

Exemple :

```csharp
var hotels = await context.Hotels
    .ToListAsync();
```

Cela ne signifie pas automatiquement :

```text
Hotel
  +
Rooms
  +
Users
  +
...
```

Pour demander explicitement des navigations :

```csharp
var hotels = await context.Hotels
    .Include(h => h.Rooms)
    .ToListAsync();
```

---

# 23. `ThenInclude()`

Pour naviguer plus profondément :

```csharp
var hotels = await context.Hotels
    .Include(h => h.Rooms)
        .ThenInclude(r => r.Beds)
    .ToListAsync();
```

Mentalement :

```text
Hotel
  |
  +-- Rooms
       |
       +-- Beds
```

`ThenInclude()` signifie :

> « Après cette navigation, charge également cette autre navigation. »

---

# 24. Include vs Projection

Deux approches :

### Include

```csharp
var hotels = await context.Hotels
    .Include(h => h.Rooms)
    .ToListAsync();
```

### Projection

```csharp
var hotels = await context.Hotels
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name,
        Rooms = h.Rooms
            .Select(r => new RoomDto
            {
                Id = r.Id,
                Number = r.Number
            })
            .ToList()
    })
    .ToListAsync();
```

Pour une API, la projection peut être particulièrement intéressante car elle permet de définir précisément le résultat attendu.

Mentalement :

```text
Include
    =
Je charge des navigations de l'entité

Select
    =
Je construis le résultat dont j'ai besoin
```

---

# 25. Lazy Loading

EF Core peut être configuré pour utiliser le **lazy loading** dans certains scénarios.

L'idée :

```csharp
hotel.Rooms
```

peut déclencher le chargement des Rooms lorsque la navigation est accédée.

Cela semble pratique, mais il faut être prudent.

Un code qui semble innocent :

```csharp
foreach (var hotel in hotels)
{
    Console.WriteLine(hotel.Rooms.Count);
}
```

peut potentiellement provoquer de nombreuses requêtes si les navigations ne sont pas déjà chargées.

C'est une cause classique de problèmes de type N+1.

---

# 26. Eager Loading

L'eager loading consiste à demander explicitement les données liées.

Exemple :

```csharp
var hotels = await context.Hotels
    .Include(h => h.Rooms)
    .ToListAsync();
```

Mentalement :

```text
Je sais avant l'exécution
que j'ai besoin des Rooms.
```

---

# 27. Explicit Loading

EF Core permet également un chargement explicite.

Conceptuellement :

```csharp
await context.Entry(hotel)
    .Collection(h => h.Rooms)
    .LoadAsync();
```

On dit alors explicitement :

> « Charge les Rooms de cet hôtel maintenant. »

C'est différent du lazy loading, qui déclenche le chargement à l'accès à la navigation.

---

# 28. Les trois stratégies de chargement

| Stratégie | Idée |
|---|---|
| Eager Loading | Je demande les relations dans la requête |
| Explicit Loading | Je demande ensuite explicitement une relation |
| Lazy Loading | L'accès à la navigation peut déclencher le chargement |

Mentalement :

```text
Eager
  =
Je sais déjà ce dont j'ai besoin.

Explicit
  =
Je demande plus tard.

Lazy
  =
L'accès peut déclencher le chargement.
```

---

# 29. N+1 et relations

Le problème N+1 est particulièrement important avec les relations.

Exemple :

```text
1 requête
   -> récupérer 100 hôtels

puis

100 requêtes
   -> récupérer les rooms de chaque hôtel
```

Total :

```text
101 requêtes
```

Il faut donc surveiller :

```text
Include
Lazy Loading
Boucles
Navigation Properties
```

et surtout vérifier le SQL réellement exécuté.

---

# 30. Cascade Delete

Les relations peuvent avoir des comportements de suppression en cascade.

Conceptuellement :

```text
Hotel
 |
 +-- Room
 +-- Room
```

Si le parent est supprimé, la base peut être configurée pour supprimer automatiquement les enfants selon la configuration de la relation.

Cela doit être réfléchi avec attention.

Une cascade mal configurée peut supprimer beaucoup plus de données que prévu.

---

# 31. Restrict / No Action

Une autre stratégie consiste à empêcher la suppression du parent lorsque des enfants existent.

Mentalement :

```text
Hotel
 |
 +-- Room

DELETE Hotel
     |
     X
Rooms existent
```

La base ou EF Core peut alors empêcher l'opération selon la configuration.

Le comportement exact dépend notamment du provider et de la configuration de la relation.

---

# 32. Relation et intégrité référentielle

Une Foreign Key permet à la base de garantir une relation valide.

Exemple :

```text
Rooms.HotelId = 999
```

alors que :

```text
Hotels.Id = 999
```

n'existe pas.

Une contrainte de Foreign Key peut empêcher cette situation.

C'est une protection importante au niveau de la base.

---

# 33. Fluent API complète

Exemple :

```csharp
modelBuilder.Entity<Room>(entity =>
{
    entity.HasKey(r => r.Id);

    entity.HasOne(r => r.Hotel)
        .WithMany(h => h.Rooms)
        .HasForeignKey(r => r.HotelId)
        .OnDelete(DeleteBehavior.Cascade);
});
```

Lis-la dans l'ordre :

```text
Room
 |
 | has one
 v
Hotel
 |
 | with many
 v
Rooms
 |
 | foreign key
 v
HotelId
 |
 | delete behavior
 v
Cascade
```

---

# 34. Relations et architecture

Dans une architecture propre, les entités du domaine peuvent représenter les relations métier, tandis que la configuration EF Core peut être placée dans Infrastructure.

Exemple :

```text
Domain
 |
 +-- Hotel
 +-- Room
 |
 v
Infrastructure
 |
 +-- EF Core configuration
 +-- DbContext
 |
 v
Database
```

L'objectif est de limiter la dépendance du domaine aux détails techniques de persistance lorsque l'architecture du projet le nécessite.

---

# 35. Relations et DTOs

Une API ne doit pas nécessairement exposer directement les graphes complets d'entités.

Éviter de transformer automatiquement :

```text
Hotel
  |
  +-- Rooms
       |
       +-- Hotel
            |
            +-- Rooms
                 |
                 ...
```

en réponse JSON.

Cela peut provoquer :

- des cycles ;
- des réponses énormes ;
- un couplage avec le modèle de persistance ;
- des performances médiocres.

Les DTOs permettent de contrôler le graphe exposé :

```text
Entity
   |
   v
Projection
   |
   v
DTO
   |
   v
JSON
```

---

# 36. Erreurs fréquentes

## Erreur 1 : croire que Navigation Property = données déjà chargées

Avoir :

```csharp
public ICollection<Room> Rooms { get; set; }
```

ne signifie pas que les rooms sont automatiquement chargées dans toutes les requêtes.

---

## Erreur 2 : utiliser `Include()` partout

`Include()` n'est pas une solution universelle.

Pour une API, une projection peut être plus efficace et plus claire.

---

## Erreur 3 : ignorer le N+1

Une boucle sur des navigations peut cacher beaucoup de requêtes.

Toujours penser :

```text
Combien de requêtes SQL vont réellement être exécutées ?
```

---

## Erreur 4 : confondre Foreign Key et Navigation Property

```csharp
public int HotelId { get; set; }
```

est la clé étrangère.

```csharp
public Hotel Hotel { get; set; } = null!;
```

est la navigation.

Elles sont liées, mais ce sont deux concepts différents.

---

## Erreur 5 : créer des relations circulaires dans les DTOs

Exemple :

```text
Hotel -> Rooms -> Hotel -> Rooms -> ...
```

Il faut contrôler le modèle exposé par l'API.

---

## Erreur 6 : mettre toutes les relations en cascade

La cascade doit être choisie selon les règles métier.

---

# 37. Tableau récapitulatif

| Concept | Signification |
|---|---|
| Foreign Key | Référence vers une autre ligne |
| Navigation Property | Permet de naviguer entre entités C# |
| `HasOne()` | Relation vers une entité |
| `HasMany()` | Relation vers plusieurs entités |
| `WithOne()` | L'autre côté possède une relation 1 |
| `WithMany()` | L'autre côté possède plusieurs relations |
| `HasForeignKey()` | Indique la propriété FK |
| `Include()` | Eager loading |
| `ThenInclude()` | Navigation supplémentaire |
| Explicit Loading | Chargement demandé explicitement |
| Lazy Loading | Chargement déclenché par accès |
| Cascade Delete | Suppression des dépendants selon configuration |
| DTO | Contrôle du graphe exposé par l'API |

---

# 38. Règles mentales

Pour :

```text
Hotel 1 ---- N Room
```

pense :

```csharp
Room
    .HasOne(Hotel)
    .WithMany(Rooms)
```

Phrase :

> Une Room a un Hotel, un Hotel a plusieurs Rooms.

Pour :

```text
User 1 ---- 1 Profile
```

pense :

```csharp
User
    .HasOne(Profile)
    .WithOne(User)
```

Pour :

```text
Student N ---- N Course
```

pense :

```csharp
Student
    .HasMany(Courses)
    .WithMany(Students)
```

---

# 39. À retenir

1. EF Core représente les relations SQL avec des relations entre entités C#.
2. Les trois grandes cardinalités sont 1-1, 1-N et N-N.
3. Une Foreign Key relie une ligne à une autre.
4. Une navigation property permet de naviguer entre objets C#.
5. `HasOne()` signifie « possède une relation vers un élément ».
6. `HasMany()` signifie « possède une relation vers plusieurs éléments ».
7. `WithOne()` représente le côté opposé en relation 1-1.
8. `WithMany()` représente le côté opposé en relation 1-N.
9. `HasForeignKey()` indique la propriété utilisée comme clé étrangère.
10. EF Core possède des conventions permettant de déduire certaines relations.
11. `Include()` permet de demander le chargement de navigations.
12. `ThenInclude()` permet de poursuivre le graphe de navigation.
13. Eager, Explicit et Lazy Loading sont trois stratégies différentes.
14. Lazy Loading peut facilement provoquer des problèmes N+1.
15. Une relation définie dans le modèle ne signifie pas que toutes les données liées sont toujours chargées.
16. Les projections avec `Select()` permettent de contrôler précisément les données récupérées.
17. Les DTOs permettent de contrôler le graphe exposé par une API.
18. Une Foreign Key contribue à l'intégrité référentielle.
19. Cascade Delete doit être configuré avec attention.
20. Il faut toujours réfléchir au nombre de requêtes SQL générées.

---

# Questions d'entretien

### 1. Quelle différence entre Foreign Key et Navigation Property ?

La Foreign Key est une valeur permettant de référencer une autre ligne, tandis que la navigation property permet de naviguer entre les entités C#.

### 2. Explique `HasOne().WithMany()`.

Cela représente généralement une relation One-to-Many.

Par exemple :

```text
Room -> HasOne -> Hotel
Hotel -> WithMany -> Rooms
```

Donc :

```text
Hotel 1 ---- N Room
```

### 3. Qu'est-ce qu'une relation Many-to-Many ?

C'est une relation où plusieurs instances du premier type peuvent être liées à plusieurs instances du second type. En SQL, elle passe généralement par une table de jointure.

### 4. Quelle différence entre Include et Select ?

`Include()` charge des navigations liées à une entité ; `Select()` permet de projeter le résultat dans la forme souhaitée.

### 5. Quels sont les trois types de chargement des relations ?

```text
Eager Loading
Explicit Loading
Lazy Loading
```

### 6. Pourquoi Lazy Loading peut-il poser problème ?

Parce qu'un simple accès à une navigation peut déclencher une requête supplémentaire, ce qui peut créer un problème N+1.

### 7. Pourquoi utiliser des DTOs avec des relations ?

Pour contrôler précisément les données exposées, éviter les cycles, réduire la taille des réponses et découpler l'API du modèle de persistance.

### 8. Qu'est-ce que Cascade Delete ?

C'est un comportement selon lequel la suppression d'une entité principale peut entraîner la suppression des entités dépendantes selon la configuration de la relation.

### 9. Pourquoi les relations peuvent-elles être déduites par EF Core ?

EF Core possède des conventions basées notamment sur les noms des propriétés, les types et les navigation properties.

### 10. Pourquoi faut-il vérifier le SQL généré ?

Parce qu'une relation ou un `Include()` apparemment simple peut produire plusieurs jointures, beaucoup de données ou plusieurs requêtes. Le code C# ne suffit pas pour juger le coût réel.

---

# Phrase à retenir

> **Une relation EF Core relie des entités C# comme une Foreign Key relie des lignes SQL : `HasOne/HasMany` décrivent la cardinalité, les navigation properties permettent de naviguer, et je dois toujours contrôler comment les relations sont réellement chargées.**
