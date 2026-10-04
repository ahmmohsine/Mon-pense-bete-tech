# EF Core — Querying

Les requêtes sont l'une des parties les plus importantes d'EF Core.

L'idée centrale est de comprendre que lorsque tu écris :

```csharp
context.Hotels
    .Where(...)
    .Select(...)
```

tu écris du **LINQ**, mais EF Core doit ensuite transformer cette requête en une opération compréhensible par la base de données.

Le modèle mental principal est :

```text
C# / LINQ
    |
    v
IQueryable
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

---

# 1. Une requête EF Core n'est pas une requête en mémoire

Prenons :

```csharp
var query = context.Hotels
    .Where(h => h.Stars >= 4);
```

Il est tentant de penser :

```text
context.Hotels
    =
liste des hôtels en mémoire
```

Mais ce n'est pas le cas.

EF Core construit généralement une requête représentée par un `IQueryable<T>`.

Mentalement :

```text
Where(...)
    |
    v
Construction de la requête
    |
    v
Pas encore nécessairement d'exécution
```

---

# 2. `IQueryable<T>`

Exemple :

```csharp
IQueryable<Hotel> query =
    context.Hotels
        .Where(h => h.Stars >= 4);
```

`IQueryable<T>` permet de composer une requête.

Tu peux continuer :

```csharp
query = query
    .OrderBy(h => h.Name)
    .Take(20);
```

Puis :

```csharp
var hotels = await query.ToListAsync();
```

Le principe :

```text
Where
   +
OrderBy
   +
Take
   |
   v
Requête construite
   |
   v
ToListAsync()
   |
   v
Exécution
```

---

# 3. Exécution différée

Ce comportement est souvent appelé **deferred execution**.

Exemple :

```csharp
var query = context.Hotels
    .Where(h => h.Stars >= 4);
```

À ce moment-là, tu construis la requête.

Puis :

```csharp
var hotels = await query.ToListAsync();
```

demande les résultats.

Mentalement :

```text
Construire
    |
    v
Attendre
    |
    v
Exécuter
```

---

# 4. Méthodes qui déclenchent généralement l'exécution

Des méthodes comme :

```csharp
ToListAsync()
ToArrayAsync()
FirstAsync()
FirstOrDefaultAsync()
SingleAsync()
SingleOrDefaultAsync()
AnyAsync()
CountAsync()
LongCountAsync()
SumAsync()
AverageAsync()
MinAsync()
MaxAsync()
```

nécessitent généralement l'exécution de la requête.

Exemple :

```csharp
var exists = await context.Hotels
    .AnyAsync(h => h.Id == id);
```

La base peut alors effectuer une opération adaptée pour répondre à :

```text
Existe-t-il au moins un hôtel correspondant ?
```

---

# 5. `Where`

`Where` filtre les résultats.

```csharp
var hotels = await context.Hotels
    .Where(h => h.Stars >= 4)
    .ToListAsync();
```

Conceptuellement :

```sql
SELECT ...
FROM Hotels
WHERE Stars >= 4;
```

Mentalement :

```text
Where
=
Filtrer côté serveur lorsque EF Core peut traduire l'expression.
```

---

# 6. `Select`

`Select` permet de faire une projection.

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

Au lieu de récupérer nécessairement toutes les colonnes de l'entité, on demande les données nécessaires au DTO.

Mentalement :

```text
Entity
   |
   v
Select
   |
   v
DTO
```

---

# 7. Pourquoi `Select` est important dans une API

Supposons :

```csharp
public class Hotel
{
    public int Id { get; set; }

    public string Name { get; set; } = string.Empty;

    public string InternalNotes { get; set; } = string.Empty;

    public DateTime CreatedAt { get; set; }
}
```

Tu n'as peut-être besoin que de :

```csharp
public class HotelDto
{
    public int Id { get; set; }

    public string Name { get; set; } = string.Empty;
}
```

La projection :

```csharp
.Select(h => new HotelDto
{
    Id = h.Id,
    Name = h.Name
})
```

exprime précisément ce dont l'API a besoin.

---

# 8. `First` vs `Single`

Ces méthodes sont souvent confondues.

## `FirstAsync()`

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.City == "Mons");
```

Signifie :

> Donne-moi le premier résultat correspondant.

S'il n'y en a aucun, `FirstAsync()` lève une exception.

---

## `FirstOrDefaultAsync()`

```csharp
var hotel = await context.Hotels
    .FirstOrDefaultAsync(h => h.Id == id);
```

Signifie :

> Donne-moi le premier résultat ou une valeur par défaut s'il n'existe pas.

Pour une référence, la valeur par défaut est généralement :

```text
null
```

---

# 9. `Single`

```csharp
var hotel = await context.Hotels
    .SingleAsync(h => h.Id == id);
```

Signifie :

> Je m'attends à exactement un résultat.

Si :

```text
0 résultat
```

ou :

```text
plus d'un résultat
```

l'opération échoue.

C'est utile lorsque l'unicité fait réellement partie de la règle métier ou de la structure des données.

---

# 10. `SingleOrDefault`

```csharp
var hotel = await context.Hotels
    .SingleOrDefaultAsync(h => h.Id == id);
```

Signifie :

```text
0 résultat
    -> null

1 résultat
    -> entité

plusieurs résultats
    -> erreur
```

---

# 11. Règle mentale `First` / `Single`

```text
First
    =
"Un résultat m'intéresse, prends-en un."

Single
    =
"Il doit y en avoir exactement un."
```

Ne choisis pas `Single()` simplement parce que tu recherches un ID.

Réfléchis à la règle que tu veux exprimer.

---

# 12. `Any`

Pour vérifier l'existence :

```csharp
var exists = await context.Hotels
    .AnyAsync(h => h.Name == "Hotel Central");
```

Cela répond à :

```text
Existe-t-il au moins un résultat ?
```

C'est généralement préférable à :

```csharp
var count = await context.Hotels
    .CountAsync(h => h.Name == "Hotel Central");

var exists = count > 0;
```

si tu n'as besoin que d'un booléen.

---

# 13. `Count`

Si tu veux réellement connaître le nombre :

```csharp
var count = await context.Hotels
    .CountAsync(h => h.Stars >= 4);
```

Mentalement :

```text
Any
    =
Existe ?

Count
    =
Combien ?
```

---

# 14. `OrderBy`

```csharp
var hotels = await context.Hotels
    .OrderBy(h => h.Name)
    .ToListAsync();
```

Conceptuellement :

```sql
ORDER BY Name
```

Pour l'ordre descendant :

```csharp
.OrderByDescending(h => h.Name)
```

---

# 15. `ThenBy`

Pour plusieurs critères :

```csharp
var hotels = await context.Hotels
    .OrderBy(h => h.City)
    .ThenBy(h => h.Name)
    .ToListAsync();
```

Mentalement :

```text
1. Trier par City
2. À City identique, trier par Name
```

---

# 16. `Skip` et `Take`

Pour la pagination :

```csharp
var hotels = await context.Hotels
    .OrderBy(h => h.Id)
    .Skip(20)
    .Take(10)
    .ToListAsync();
```

Mentalement :

```text
Skip(20)
    =
ignorer les 20 premiers

Take(10)
    =
prendre les 10 suivants
```

---

# 17. Pourquoi l'ordre est important pour la pagination

Évite de paginer sans ordre stable :

```csharp
.Skip(20)
.Take(10)
```

Une pagination fiable doit généralement avoir un ordre déterministe :

```csharp
.OrderBy(h => h.Id)
.Skip(...)
.Take(...)
```

Sinon les résultats peuvent être moins prévisibles lorsque les données changent.

---

# 18. Pagination

Exemple :

```csharp
int page = 3;
int pageSize = 20;

var hotels = await context.Hotels
    .OrderBy(h => h.Id)
    .Skip((page - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();
```

Pour :

```text
page = 3
pageSize = 20
```

on obtient :

```text
Skip(40)
Take(20)
```

---

# 19. Offset pagination vs Keyset pagination

La pagination avec :

```csharp
Skip()
Take()
```

est appelée couramment **offset pagination**.

Elle est simple, mais pour de très gros volumes, de grands offsets peuvent devenir coûteux.

Une autre approche est la **keyset pagination**.

Exemple :

```csharp
var hotels = await context.Hotels
    .Where(h => h.Id > lastId)
    .OrderBy(h => h.Id)
    .Take(20)
    .ToListAsync();
```

Mentalement :

```text
Offset
    =
"Ignore les N premiers."

Keyset
    =
"Continue après cette clé."
```

---

# 20. `Contains`

Exemple :

```csharp
var ids = new[] { 1, 5, 8 };

var hotels = await context.Hotels
    .Where(h => ids.Contains(h.Id))
    .ToListAsync();
```

EF Core peut traduire ce type de condition vers une forme SQL adaptée, souvent proche de :

```sql
WHERE Id IN (1, 5, 8)
```

---

# 21. `Any` avec une relation

Exemple :

```csharp
var hotels = await context.Hotels
    .Where(h => h.Rooms.Any())
    .ToListAsync();
```

Cela signifie :

> Sélectionne les hôtels qui possèdent au moins une chambre.

On peut aussi filtrer :

```csharp
var hotels = await context.Hotels
    .Where(h => h.Rooms.Any(r => r.IsAvailable))
    .ToListAsync();
```

---

# 22. `All`

Exemple :

```csharp
var valid = await context.Hotels
    .AllAsync(h => h.Stars >= 3);
```

Cela signifie :

> Toutes les lignes satisfont-elles la condition ?

Il faut être attentif à la logique métier de `All()` et au cas d'une séquence vide.

---

# 23. `GroupBy`

Exemple :

```csharp
var result = await context.Hotels
    .GroupBy(h => h.City)
    .Select(g => new
    {
        City = g.Key,
        Count = g.Count()
    })
    .ToListAsync();
```

Mentalement :

```text
Hotels
   |
   v
GroupBy(City)
   |
   v
Groupes
   |
   v
Select
   |
   v
Résultat agrégé
```

EF Core doit traduire la partie correspondante vers le langage de la base.

---

# 24. `Join`

On peut effectuer des jointures avec LINQ :

```csharp
var result = await context.Rooms
    .Join(
        context.Hotels,
        room => room.HotelId,
        hotel => hotel.Id,
        (room, hotel) => new
        {
            Room = room.Number,
            Hotel = hotel.Name
        })
    .ToListAsync();
```

Mais dans EF Core, une requête exprimée avec les navigations peut souvent être plus naturelle :

```csharp
var result = await context.Rooms
    .Select(r => new
    {
        Room = r.Number,
        Hotel = r.Hotel.Name
    })
    .ToListAsync();
```

---

# 25. Navigations et requêtes

Avec :

```csharp
var rooms = await context.Rooms
    .Where(r => r.Hotel.City == "Mons")
    .ToListAsync();
```

tu peux filtrer à travers une navigation.

Mentalement :

```text
Room
  |
  v
Hotel
  |
  v
City
```

EF Core peut traduire cette relation en SQL.

---

# 26. `Include` dans une requête

Exemple :

```csharp
var hotels = await context.Hotels
    .Include(h => h.Rooms)
    .ToListAsync();
```

Cela demande de charger les navigations.

Mais si ton objectif est simplement de produire un DTO, une projection peut être préférable :

```csharp
var hotels = await context.Hotels
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name,
        RoomsCount = h.Rooms.Count()
    })
    .ToListAsync();
```

Ici, tu ne demandes pas nécessairement toutes les chambres.

Tu demandes seulement :

```text
Id
Name
Nombre de Rooms
```

---

# 27. Projection de collections

Tu peux également projeter une collection :

```csharp
var hotels = await context.Hotels
    .Select(h => new HotelDetailsDto
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

Cela permet de construire directement la forme nécessaire à l'API.

---

# 28. `AsNoTracking`

Pour une lecture :

```csharp
var hotels = await context.Hotels
    .AsNoTracking()
    .ToListAsync();
```

Le tracking n'est pas demandé.

Pour une projection DTO :

```csharp
var hotels = await context.Hotels
    .AsNoTracking()
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .ToListAsync();
```

Cela peut être pertinent dans une lecture pure.

---

# 29. `AsNoTracking` n'est pas un accélérateur magique

Il ne faut pas penser :

```text
AsNoTracking()
=
toujours beaucoup plus rapide
```

Son intérêt dépend du scénario.

Il réduit notamment le travail de tracking, mais le coût principal peut parfois venir de :

```text
SQL
Index
Jointures
Volume de données
Réseau
Matérialisation
```

Il faut donc optimiser la requête dans son ensemble.

---

# 30. `ToList()` trop tôt

Erreur fréquente :

```csharp
var hotels = await context.Hotels
    .ToListAsync();

var result = hotels
    .Where(h => h.Stars >= 4)
    .OrderBy(h => h.Name)
    .Take(20)
    .ToList();
```

Ici :

```text
ToListAsync()
    |
    v
tous les hôtels sont chargés
    |
    v
Where/OrderBy/Take en mémoire
```

Souvent, il vaut mieux :

```csharp
var result = await context.Hotels
    .Where(h => h.Stars >= 4)
    .OrderBy(h => h.Name)
    .Take(20)
    .ToListAsync();
```

Mentalement :

```text
Filtrer avant d'exécuter.
```

---

# 31. `IEnumerable` vs `IQueryable`

Très important.

## `IQueryable<T>`

```csharp
IQueryable<Hotel>
```

représente généralement une requête composable que le provider peut traduire.

## `IEnumerable<T>`

```csharp
IEnumerable<Hotel>
```

représente une séquence que tu peux parcourir.

Si tu matérialises :

```csharp
var hotels = await context.Hotels
    .ToListAsync();
```

tu as maintenant une collection en mémoire.

Puis :

```csharp
hotels.Where(...)
```

travaille en mémoire.

---

# 32. Le piège du `AsEnumerable()`

Exemple :

```csharp
var result = context.Hotels
    .Where(h => h.Stars >= 4)
    .AsEnumerable()
    .Where(h => SomeMethod(h));
```

Après :

```csharp
AsEnumerable()
```

la suite peut être exécutée côté .NET plutôt que traduite en SQL.

Mentalement :

```text
IQueryable
   |
   | AsEnumerable()
   v
IEnumerable
   |
   v
Traitement côté application
```

Cela peut être volontaire, mais il faut savoir que l'on change de monde.

---

# 33. `Func` vs `Expression`

Une autre notion importante :

```csharp
Func<Hotel, bool>
```

et :

```csharp
Expression<Func<Hotel, bool>>
```

ne représentent pas exactement la même chose.

`Func<Hotel, bool>` représente une fonction exécutable par .NET.

`Expression<Func<Hotel, bool>>` représente une structure décrivant cette expression, que le provider peut analyser et éventuellement traduire.

C'est une raison importante pour laquelle EF Core peut comprendre :

```csharp
h => h.Stars >= 4
```

et la traduire.

---

# 34. Attention aux méthodes non traduisibles

Exemple :

```csharp
var hotels = await context.Hotels
    .Where(h => MyCustomMethod(h.Name))
    .ToListAsync();
```

Selon l'expression et la version/provider, EF Core peut ne pas pouvoir traduire la méthode personnalisée vers SQL.

Le principe important :

```text
Expression C#
     |
     v
EF Core doit pouvoir la traduire
     |
     v
SQL
```

Si ce n'est pas possible, la requête peut échouer plutôt que de faire silencieusement tout le travail côté client dans de nombreux scénarios modernes.

---

# 35. `Contains` et collections en mémoire

Exemple :

```csharp
var ids = new List<int>
{
    1, 2, 3
};

var hotels = await context.Hotels
    .Where(h => ids.Contains(h.Id))
    .ToListAsync();
```

EF Core peut traduire ce type de logique vers une condition adaptée à la base.

Mais la taille de la collection doit être prise en compte.

Une liste contenant des milliers ou des millions d'éléments peut devenir un problème de conception.

---

# 36. Agrégations

EF Core permet des opérations comme :

```csharp
var count = await context.Hotels
    .CountAsync();

var average = await context.Hotels
    .AverageAsync(h => h.Stars);
```

Selon la requête, ces opérations peuvent être effectuées côté base.

Mentalement :

```text
Database
   |
   | calcul
   v
résultat unique
   |
   v
Application
```

Il est généralement préférable de ne pas charger toutes les lignes simplement pour calculer une valeur.

---

# 37. Ne pas faire ceci

Éviter :

```csharp
var hotels = await context.Hotels
    .ToListAsync();

var count = hotels.Count;
```

si tu as seulement besoin du nombre.

Préférer :

```csharp
var count = await context.Hotels
    .CountAsync();
```

La base peut alors effectuer le calcul directement.

---

# 38. Recherche par clé primaire

Pour une clé primaire :

```csharp
var hotel = await context.Hotels
    .FindAsync(id);
```

`FindAsync()` possède un comportement particulier lié au Change Tracker.

Si l'entité correspondante est déjà suivie, EF Core peut la retourner sans refaire une requête identique vers la base.

---

# 39. `FindAsync` vs `FirstOrDefaultAsync`

```csharp
FindAsync(id)
```

est adapté à :

```text
Recherche par clé primaire
```

Alors que :

```csharp
FirstOrDefaultAsync(h => h.Name == name)
```

est une requête selon une condition.

Mentalement :

```text
Find
    =
clé primaire

FirstOrDefault
    =
condition LINQ
```

---

# 40. Requêtes async

Dans une application web, utiliser les méthodes async pour les opérations I/O :

```csharp
ToListAsync()
FirstAsync()
FirstOrDefaultAsync()
SingleAsync()
AnyAsync()
CountAsync()
```

Exemple :

```csharp
public async Task<List<Hotel>> GetHotelsAsync(
    CancellationToken cancellationToken)
{
    return await context.Hotels
        .ToListAsync(cancellationToken);
}
```

---

# 41. CancellationToken dans les requêtes

Exemple :

```csharp
var hotels = await context.Hotels
    .Where(h => h.Stars >= 4)
    .ToListAsync(cancellationToken);
```

Cela permet de propager une annulation :

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

---

# 42. Requêtes et performance

Quand une requête est lente, ne commence pas automatiquement par ajouter :

```csharp
AsNoTracking()
```

Analyse plutôt :

```text
1. SQL généré
2. WHERE
3. JOIN
4. SELECT
5. Index
6. Volume de données
7. Pagination
8. Tracking
9. Nombre de requêtes
```

Le problème peut être totalement ailleurs.

---

# 43. Projection avant matérialisation

Bonne pratique :

```csharp
var hotels = await context.Hotels
    .Where(h => h.Stars >= 4)
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .ToListAsync();
```

Mentalement :

```text
Filter
   |
   v
Projection
   |
   v
Database
   |
   v
DTO
```

plutôt que :

```text
Database
   |
   v
Toutes les entités
   |
   v
Filtre en mémoire
   |
   v
Projection en mémoire
```

---

# 44. Requêtes et N+1

Attention aux boucles.

Mauvais scénario potentiel :

```csharp
var hotels = await context.Hotels
    .ToListAsync();

foreach (var hotel in hotels)
{
    var rooms = await context.Rooms
        .Where(r => r.HotelId == hotel.Id)
        .ToListAsync();
}
```

Si 100 hôtels sont récupérés :

```text
1 requête pour les hôtels
+
100 requêtes pour les rooms
=
101 requêtes
```

C'est le problème N+1.

---

# 45. Solution : penser en requête globale

Au lieu de demander les données une par une, réfléchis à ce que la base doit retourner.

Par exemple :

```csharp
var hotels = await context.Hotels
    .Select(h => new HotelDetailsDto
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

L'objectif est :

```text
Décrire le résultat attendu
        |
        v
Laisser EF Core construire une stratégie SQL adaptée
```

---

# 46. Query Splitting

Les relations complexes avec plusieurs collections peuvent produire de grosses requêtes avec beaucoup de jointures.

EF Core propose notamment des stratégies de requêtes séparées.

Conceptuellement :

```text
Single Query
    =
une requête avec plusieurs jointures

Split Query
    =
plusieurs requêtes coordonnées par EF Core
```

Le choix dépend du scénario.

Il ne faut donc pas appliquer automatiquement une stratégie sans mesurer le résultat.

---

# 47. Logging du SQL

Pour comprendre une requête, il est important de pouvoir observer le SQL généré.

En développement, la configuration du logging EF Core peut aider à voir :

```text
SELECT ...
FROM ...
WHERE ...
```

Cela permet de vérifier que ton LINQ produit bien ce que tu imagines.

Mentalement :

```text
C# LINQ
   |
   v
"Qu'est-ce que j'ai demandé ?"

SQL
   |
   v
"Qu'est-ce que la base va réellement exécuter ?"
```

---

# 48. Une requête n'est pas seulement du C#

Une ligne comme :

```csharp
context.Hotels
    .Where(h => h.City == "Mons")
    .OrderBy(h => h.Name)
    .Take(20)
```

doit être lue sur deux niveaux.

### Niveau C#

```text
Where
OrderBy
Take
```

### Niveau base

```text
WHERE
ORDER BY
TOP / LIMIT / équivalent
```

C'est cette double lecture qui permet de devenir bon avec EF Core.

---

# 49. Règle mentale

Quand tu écris :

```csharp
.Where(...)
```

pense :

> « Je filtre. »

Quand tu écris :

```csharp
.Select(...)
```

pense :

> « Je projette. »

Quand tu écris :

```csharp
.OrderBy(...)
```

pense :

> « Je trie. »

Quand tu écris :

```csharp
.Skip(...)
```

pense :

> « J'ignore une partie. »

Quand tu écris :

```csharp
.Take(...)
```

pense :

> « Je limite le résultat. »

Quand tu écris :

```csharp
.AnyAsync()
```

pense :

> « Existe-t-il au moins un résultat ? »

Quand tu écris :

```csharp
.CountAsync()
```

pense :

> « Combien y en a-t-il ? »

Quand tu écris :

```csharp
.ToListAsync()
```

pense :

> « Exécute la requête et matérialise les résultats. »

---

# 50. À retenir

1. EF Core utilise fortement LINQ pour construire les requêtes.
2. `IQueryable<T>` permet de composer une requête destinée au provider.
3. Une requête n'est généralement pas exécutée au moment où elle est simplement construite.
4. `ToListAsync()`, `FirstAsync()`, `AnyAsync()`, `CountAsync()` et d'autres opérateurs déclenchent généralement l'exécution.
5. `Where()` sert à filtrer.
6. `Select()` sert à projeter.
7. `OrderBy()` et `ThenBy()` servent à trier.
8. `Skip()` et `Take()` permettent notamment la pagination.
9. `First()` signifie « prends le premier ».
10. `Single()` signifie « il doit y en avoir exactement un ».
11. `Any()` sert à tester l'existence.
12. `Count()` sert à obtenir un nombre.
13. Il vaut mieux filtrer et projeter avant de matérialiser les données.
14. `ToList()` ou `ToListAsync()` trop tôt peut déplacer inutilement du travail en mémoire.
15. `IQueryable` et `IEnumerable` ne représentent pas le même niveau d'exécution.
16. `AsEnumerable()` peut faire passer la suite du traitement côté application.
17. Toutes les expressions C# ne sont pas forcément traduisibles en SQL.
18. `Include()` sert notamment au chargement des navigations.
19. Une projection peut être préférable à `Include()` pour une API.
20. Le problème N+1 doit être surveillé.
21. `AsNoTracking()` peut être utile pour les lectures pures, mais ce n'est pas un accélérateur magique.
22. La pagination avec `Skip/Take` est simple mais peut devenir coûteuse sur de gros offsets.
23. La keyset pagination peut être intéressante pour certains gros volumes.
24. Il faut toujours réfléchir au SQL réellement généré.
25. Les requêtes async permettent de ne pas bloquer inutilement pendant les opérations I/O.
26. `CancellationToken` doit pouvoir être propagé jusqu'à EF Core.

---

# Questions d'entretien

### 1. Qu'est-ce que `IQueryable<T>` ?

C'est une abstraction représentant une requête composable que le provider peut analyser et traduire vers la source de données.

### 2. Qu'est-ce que l'exécution différée ?

C'est le fait qu'une requête peut être construite sans être immédiatement exécutée. L'exécution intervient généralement lorsqu'un opérateur terminal demande les résultats.

### 3. Pourquoi éviter `ToList()` trop tôt ?

Parce que cela matérialise les données et peut faire exécuter la suite des opérations en mémoire au lieu de laisser la base effectuer le filtrage, tri ou calcul.

### 4. Quelle différence entre `First()` et `Single()` ?

`First()` demande un premier résultat ; `Single()` exprime l'attente d'exactement un résultat et échoue si plusieurs résultats existent.

### 5. Pourquoi utiliser `Any()` plutôt que `Count() > 0` ?

Lorsque l'objectif est uniquement de savoir si une ligne existe, `Any()` exprime directement cette intention et permet à la base d'utiliser une stratégie adaptée à l'existence.

### 6. Pourquoi utiliser `Select()` avec un DTO ?

Pour récupérer et construire uniquement la forme de données nécessaire à l'API.

### 7. Qu'est-ce que le problème N+1 ?

Une requête initiale récupère une collection puis une requête supplémentaire est exécutée pour chaque élément, créant potentiellement un grand nombre de requêtes.

### 8. Quelle différence entre `IEnumerable<T>` et `IQueryable<T>` ?

`IEnumerable<T>` représente une séquence parcourue côté .NET ; `IQueryable<T>` permet de construire une expression que le provider peut traduire et exécuter sur la source de données.

### 9. Pourquoi faut-il comprendre le SQL généré ?

Parce que le coût réel de la requête dépend du SQL exécuté, des jointures, des index, du volume de données et du nombre de requêtes.

### 10. Qu'est-ce que la keyset pagination ?

Une pagination basée sur une valeur de référence, souvent une clé ou un ordre stable, plutôt que de demander à la base d'ignorer un grand nombre de lignes avec `Skip()`.

### 11. Est-ce que `AsNoTracking()` rend toujours une requête rapide ?

Non. Il réduit le coût du tracking, mais la lenteur peut venir du SQL, des jointures, des index, du volume de données, du réseau ou d'autres facteurs.

### 12. Pourquoi certaines méthodes C# ne peuvent-elles pas être utilisées directement dans une requête EF Core ?

Parce que le provider doit pouvoir traduire l'expression en une opération comprise par la base de données.

---

# Phrase à retenir

> **Avec EF Core, je construis ma requête en LINQ, EF Core la traduit vers le SQL du provider, puis un opérateur terminal comme `ToListAsync()` l'exécute ; je dois donc toujours penser à la fois en C# et en SQL.**
