# EF Core — Performance

Optimiser EF Core ne consiste pas à appliquer systématiquement :

```csharp
AsNoTracking()
```

ou :

```csharp
Include(...)
```

La vraie question est :

> **Qu'est-ce qui coûte réellement du temps ou des ressources dans ma requête ?**

Le modèle mental à retenir :

```text
Code C#
   |
   v
LINQ
   |
   v
SQL généré
   |
   +-- Database
   |     +-- Index
   |     +-- JOIN
   |     +-- WHERE
   |     +-- ORDER BY
   |
   v
Résultats
   |
   v
Réseau
   |
   v
Matérialisation C#
   |
   v
Tracking éventuel
```

Une requête peut donc être lente à plusieurs niveaux.

---

# 1. Première règle : mesurer avant d'optimiser

Une mauvaise approche :

```text
La requête est lente
    |
    v
J'ajoute AsNoTracking()
```

Une meilleure approche :

```text
La requête est lente
    |
    v
J'observe
    |
    +-- SQL généré
    +-- nombre de requêtes
    +-- volume de données
    +-- durée SQL
    +-- index
    +-- tracking
    +-- matérialisation
    |
    v
J'identifie le vrai problème
    |
    v
J'optimise
    |
    v
Je mesure à nouveau
```

Mentalement :

> **Mesurer → comprendre → modifier → mesurer.**

---

# 2. Les principales causes de lenteur

Une requête EF Core peut être lente à cause de :

```text
1. Trop de données récupérées
2. Mauvais filtre
3. Mauvais index
4. Trop de JOIN
5. N+1 queries
6. Tracking inutile
7. Projection inefficace
8. Pagination absente
9. Requête SQL coûteuse
10. Trop de données transférées sur le réseau
11. Matérialisation importante
12. Appels séquentiels inutiles
```

Il faut donc éviter de réduire la performance à EF Core lui-même.

---

# 3. Projection avec `Select`

Une optimisation très importante consiste à ne demander que les données nécessaires.

Au lieu de :

```csharp
var hotels = await context.Hotels
    .ToListAsync();
```

puis :

```csharp
var result = hotels
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .ToList();
```

préférer :

```csharp
var result = await context.Hotels
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .ToListAsync();
```

Mentalement :

```text
Mauvais flux possible :

Database
   |
   v
Toutes les colonnes
   |
   v
C#
   |
   v
DTO


Meilleur objectif :

Database
   |
   v
Colonnes nécessaires
   |
   v
DTO
```

---

# 4. Pourquoi la projection peut être importante

Supposons :

```csharp
public class Hotel
{
    public int Id { get; set; }

    public string Name { get; set; } = string.Empty;

    public string Description { get; set; } = string.Empty;

    public string InternalNotes { get; set; } = string.Empty;

    public byte[] LargeImage { get; set; } = [];
}
```

Si l'API ne retourne que :

```csharp
public class HotelListDto
{
    public int Id { get; set; }

    public string Name { get; set; } = string.Empty;
}
```

il est inutile de récupérer systématiquement :

```text
Description
InternalNotes
LargeImage
```

La projection permet de limiter les données demandées.

---

# 5. `AsNoTracking`

Pour une lecture pure :

```csharp
var hotels = await context.Hotels
    .AsNoTracking()
    .ToListAsync();
```

Cela évite le tracking normal des entités.

C'est utile lorsque :

```text
Je lis
    +
Je ne vais pas modifier les entités
    +
Je ne vais pas appeler SaveChanges pour elles
```

---

# 6. Attention : `AsNoTracking()` n'est pas magique

Ne pense pas :

```text
AsNoTracking()
=
requête rapide
```

Il peut réduire le coût du Change Tracker, mais si le problème est :

```text
SELECT énorme
JOIN coûteux
pas d'index
N+1
réseau
pagination absente
```

`AsNoTracking()` ne réglera pas nécessairement le problème principal.

Mentalement :

```text
Performance EF Core
    =
Tracking
+
SQL
+
Database
+
Network
+
Materialization
+
Volume de données
```

---

# 7. Le problème N+1

Un des problèmes classiques.

Exemple :

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

Si tu as :

```text
100 hôtels
```

tu peux obtenir :

```text
1 requête Hotels
+
100 requêtes Rooms
=
101 requêtes
```

C'est le problème **N+1**.

---

# 8. Pourquoi N+1 est mauvais

Chaque requête peut entraîner :

```text
Application
    |
    v
Database
    |
    v
Résultat
    |
    v
Application
```

Répéter ce cycle des centaines ou milliers de fois coûte cher.

Le problème n'est pas seulement le temps d'exécution SQL.

Il y a aussi :

```text
Réseau
Latence
Connexions
Traitement
```

---

# 9. Éviter N+1 avec une requête adaptée

Au lieu de faire des requêtes dans une boucle, réfléchir à une requête globale.

Exemple :

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

Le but est de laisser EF Core construire une stratégie de requête adaptée plutôt que d'effectuer une requête par élément.

---

# 10. `Include()` et performance

Exemple :

```csharp
var hotels = await context.Hotels
    .Include(h => h.Rooms)
    .ToListAsync();
```

`Include()` peut être parfaitement approprié.

Mais :

```csharp
.Include(h => h.Rooms)
.Include(h => h.Bookings)
.Include(h => h.Reviews)
.Include(h => h.Users)
```

peut produire une requête très complexe ou beaucoup de données.

Il faut se demander :

> Est-ce que j'ai réellement besoin de toutes ces données ?

---

# 11. Projection vs Include

Pour une API :

```csharp
.Select(...)
```

peut souvent être plus précis que :

```csharp
.Include(...)
```

Exemple :

```csharp
var hotels = await context.Hotels
    .Select(h => new HotelSummaryDto
    {
        Id = h.Id,
        Name = h.Name,
        RoomsCount = h.Rooms.Count()
    })
    .ToListAsync();
```

Tu demandes :

```text
Id
Name
Nombre de Rooms
```

et non :

```text
toutes les Rooms
```

---

# 12. Pagination

Éviter de récupérer des milliers de lignes si l'API n'en affiche que 20.

Mauvais scénario :

```csharp
var hotels = await context.Hotels
    .ToListAsync();
```

puis :

```csharp
var page = hotels
    .Skip(1000)
    .Take(20)
    .ToList();
```

Les données ont déjà été chargées.

Préférer :

```csharp
var page = await context.Hotels
    .OrderBy(h => h.Id)
    .Skip(1000)
    .Take(20)
    .ToListAsync();
```

Le filtrage de pagination est alors demandé à la base.

---

# 13. Pagination et ordre stable

Une pagination doit généralement utiliser un ordre déterministe :

```csharp
.OrderBy(h => h.Id)
```

puis :

```csharp
.Skip(...)
.Take(...)
```

Sinon les résultats peuvent être difficiles à stabiliser lorsque les données évoluent.

---

# 14. Offset pagination

Avec :

```csharp
.Skip(100000)
.Take(20)
```

la base doit gérer un offset important.

Pour de très grandes tables, cela peut devenir coûteux.

Exemple :

```text
Page 1
Skip 0

Page 1000
Skip 19980

Page 5000
Skip 99980
```

Plus l'offset augmente, plus cette stratégie peut devenir problématique selon le moteur et la requête.

---

# 15. Keyset pagination

Une autre stratégie est la pagination basée sur une valeur de référence.

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
"Saute N lignes."

Keyset
    =
"Continue après cette clé."
```

La keyset pagination peut être particulièrement intéressante pour les gros volumes et les flux où l'utilisateur avance page après page.

---

# 16. Index

Un index est un mécanisme de la base de données permettant notamment d'accélérer certaines recherches.

Supposons :

```csharp
.Where(h => h.City == city)
```

Si la table contient des millions de lignes, l'absence d'un index adapté peut devenir un problème.

Conceptuellement :

```text
Sans index
    |
    v
Beaucoup de lignes à examiner

Avec index adapté
    |
    v
Recherche potentiellement beaucoup plus efficace
```

Mais :

> Un index n'est pas automatiquement bénéfique pour toutes les colonnes.

---

# 17. Les indexes ont un coût

Un index accélère certaines lectures mais peut augmenter :

```text
Espace disque
Coût des INSERT
Coût des UPDATE
Coût des DELETE
```

Il faut donc choisir les indexes en fonction des requêtes réellement utilisées.

Mentalement :

```text
Index
    =
accélérateur de certaines recherches

mais
    =
coût supplémentaire à maintenir
```

---

# 18. Configurer un index avec EF Core

Exemple :

```csharp
modelBuilder.Entity<Hotel>()
    .HasIndex(h => h.City);
```

Pour plusieurs colonnes :

```csharp
modelBuilder.Entity<Hotel>()
    .HasIndex(h => new
    {
        h.City,
        h.Name
    });
```

La stratégie exacte dépend du moteur de base de données et des requêtes.

---

# 19. Index et `Where`

Une requête :

```csharp
.Where(h => h.City == "Mons")
```

peut bénéficier d'un index sur :

```text
City
```

Mais il faut considérer la requête complète :

```text
WHERE
ORDER BY
JOIN
SELECT
```

Un index doit être réfléchi par rapport aux accès réels aux données.

---

# 20. SQL généré

Quand une requête est lente, il est essentiel de regarder ce qu'EF Core génère.

Le code :

```csharp
var hotels = await context.Hotels
    .Where(h => h.Stars >= 4)
    .OrderBy(h => h.Name)
    .Take(20)
    .ToListAsync();
```

est une intention C#.

La base reçoit une représentation SQL correspondante.

Le SQL est ce qu'il faut finalement analyser côté base.

---

# 21. Pourquoi regarder le SQL ?

Parce qu'une requête LINQ peut sembler simple :

```csharp
.Include(...)
```

mais générer :

```text
JOIN
JOIN
JOIN
JOIN
```

avec énormément de lignes intermédiaires.

À l'inverse, une projection bien pensée peut générer une requête beaucoup plus ciblée.

Mentalement :

```text
Code LINQ
    =
Intention

SQL
    =
Exécution réelle côté base
```

---

# 22. Logging EF Core

En développement, le logging peut permettre d'observer les requêtes.

Selon la configuration de l'application, tu peux voir des informations comme :

```text
Executed DbCommand
SELECT ...
FROM ...
WHERE ...
```

Cela permet de repérer :

```text
N+1
requêtes inattendues
filtres absents
SELECT trop large
```

---

# 23. `ToQueryString()`

Pour certaines requêtes EF Core, tu peux inspecter le SQL généré avec :

```csharp
var query = context.Hotels
    .Where(h => h.Stars >= 4)
    .OrderBy(h => h.Name);

var sql = query.ToQueryString();
```

Cela permet de voir la représentation SQL produite par EF Core sans nécessairement exécuter la requête pour obtenir les données.

C'est particulièrement utile pour comprendre ce que ton LINQ devient.

---

# 24. Attention à `ToQueryString()`

`ToQueryString()` est un outil d'inspection.

Ce n'est pas :

```csharp
var result = await query.ToQueryString();
```

et ce n'est pas la méthode qui récupère les données.

Pour exécuter :

```csharp
var result = await query.ToListAsync();
```

Mentalement :

```text
ToQueryString()
    =
"Montre-moi le SQL."

ToListAsync()
    =
"Exécute la requête et donne-moi les résultats."
```

---

# 25. `Select` avant `ToList`

Une règle très utile :

```text
Filtrer
   |
   v
Projeter
   |
   v
Trier
   |
   v
Paginer
   |
   v
Matérialiser
```

Par exemple :

```csharp
var result = await context.Hotels
    .Where(h => h.Stars >= 4)
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .OrderBy(h => h.Name)
    .Take(20)
    .ToListAsync();
```

L'ordre exact peut varier selon la logique souhaitée, mais l'idée centrale est :

> Faire le plus possible côté base avant de matérialiser.

---

# 26. Ne pas charger une table entière sans raison

Éviter :

```csharp
var hotels = await context.Hotels
    .ToListAsync();
```

si tu sais que tu veux seulement :

```text
les hôtels de Mons
les 20 premiers
seulement Id + Name
```

Préférer une requête exprimant directement le besoin.

---

# 27. `Any` plutôt que `Count > 0`

Pour vérifier l'existence :

```csharp
var exists = await context.Hotels
    .AnyAsync(h => h.City == "Mons");
```

plutôt que :

```csharp
var count = await context.Hotels
    .CountAsync(h => h.City == "Mons");

var exists = count > 0;
```

Pourquoi ?

Parce que tu demandes directement :

```text
Existe-t-il au moins un élément ?
```

La base peut alors utiliser une stratégie adaptée à cette intention.

---

# 28. `First` plutôt que `ToList()[0]`

Éviter :

```csharp
var hotels = await context.Hotels
    .Where(h => h.Id == id)
    .ToListAsync();

var hotel = hotels[0];
```

Préférer :

```csharp
var hotel = await context.Hotels
    .FirstOrDefaultAsync(h => h.Id == id);
```

Pourquoi ?

Parce que tu demandes directement ce dont tu as besoin.

---

# 29. `FindAsync` pour une clé primaire

Pour une recherche par clé primaire :

```csharp
var hotel = await context.Hotels
    .FindAsync(id);
```

C'est généralement plus adapté que de charger une liste entière.

---

# 30. Éviter les appels séquentiels inutiles

Supposons :

```csharp
var hotels = await service.GetHotelsAsync();
var rooms = await service.GetRoomsAsync();
```

Si les deux opérations sont réellement indépendantes et peuvent être exécutées en parallèle, on peut envisager :

```csharp
var hotelsTask = service.GetHotelsAsync();
var roomsTask = service.GetRoomsAsync();

await Task.WhenAll(hotelsTask, roomsTask);
```

Mais attention avec EF Core :

> Ne lance pas plusieurs opérations simultanées sur le même `DbContext`.

Si les opérations utilisent le même contexte, il faut respecter sa contrainte de non-concurrence.

La parallélisation doit donc être pensée avec la durée de vie et les instances de contexte.

---

# 31. `DbContext` et performance

Un `DbContext` trop long peut accumuler beaucoup d'entités suivies.

Cela peut augmenter :

```text
Mémoire
Travail du Change Tracker
Complexité du contexte
```

Dans ASP.NET Core, le modèle Scoped permet généralement de garder une portée raisonnable :

```text
HTTP Request
   |
   v
DbContext
   |
   v
End Request
```

---

# 32. Split Query

Avec plusieurs collections :

```csharp
var result = await context.Hotels
    .Include(h => h.Rooms)
    .Include(h => h.Reviews)
    .ToListAsync();
```

une seule requête peut générer un grand résultat avec plusieurs combinaisons de lignes.

Dans certains scénarios, une requête séparée peut être préférable.

EF Core permet notamment :

```csharp
var result = await context.Hotels
    .Include(h => h.Rooms)
    .Include(h => h.Reviews)
    .AsSplitQuery()
    .ToListAsync();
```

Mentalement :

```text
Single Query
    =
une requête principale avec les relations

Split Query
    =
plusieurs requêtes coordonnées
```

Il faut comparer les résultats et le coût réel.

---

# 33. Attention au cartesian explosion

Avec plusieurs collections, une requête SQL avec plusieurs jointures peut produire beaucoup de lignes intermédiaires.

Exemple :

```text
Hotel
  |
  +-- 10 Rooms
  |
  +-- 20 Reviews
```

Une combinaison de jointures peut produire potentiellement :

```text
10 x 20 = 200
```

lignes intermédiaires pour un seul hôtel dans certaines formes de requêtes.

C'est l'une des raisons pour lesquelles les requêtes complexes doivent être analysées.

---

# 34. Pagination des relations

Une requête comme :

```csharp
.Include(h => h.Rooms)
```

peut charger toutes les rooms d'un hôtel.

Mais peut-être que l'API veut seulement :

```text
20 rooms
```

Dans ce cas, une projection dédiée peut être plus adaptée qu'un `Include` général.

Le principe :

> Charger uniquement le graphe réellement nécessaire.

---

# 35. Compter sans charger

Si tu veux :

```text
Nombre de rooms
```

ne fais pas :

```csharp
var rooms = await context.Rooms
    .Where(r => r.HotelId == hotelId)
    .ToListAsync();

var count = rooms.Count;
```

Préférer :

```csharp
var count = await context.Rooms
    .CountAsync(r => r.HotelId == hotelId);
```

Mentalement :

```text
Je veux un nombre
    |
    v
Demander un nombre à la base
```

---

# 36. Agréger côté base

Même principe pour :

```csharp
SumAsync()
AverageAsync()
MinAsync()
MaxAsync()
CountAsync()
```

Exemple :

```csharp
var averageStars = await context.Hotels
    .AverageAsync(h => h.Stars);
```

Il est généralement inutile de charger toutes les lignes en mémoire juste pour calculer une moyenne.

---

# 37. Filtrer tôt

Supposons :

```csharp
var result = await context.Hotels
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .Where(h => h.Name.Contains("Central"))
    .ToListAsync();
```

Selon le cas, il peut être plus clair de filtrer avant la projection :

```csharp
var result = await context.Hotels
    .Where(h => h.Name.Contains("Central"))
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .ToListAsync();
```

Le point important est de garder la requête composable et de laisser EF Core traduire correctement les opérations.

---

# 38. Attention aux méthodes C# personnalisées

Exemple :

```csharp
bool IsInteresting(string name)
{
    return name.Length > 10;
}
```

Puis :

```csharp
context.Hotels
    .Where(h => IsInteresting(h.Name));
```

Il faut vérifier si cette expression est traduisible par le provider.

Une méthode C# arbitraire n'est pas automatiquement transformable en SQL.

Une approche consiste à exprimer directement la condition traduisible :

```csharp
.Where(h => h.Name.Length > 10)
```

si le provider prend en charge cette traduction.

---

# 39. Compiled Queries

EF Core possède également des mécanismes de **compiled queries** pour certains scénarios très spécifiques où la même forme de requête est exécutée très fréquemment.

Le concept :

```text
Requête LINQ
    |
    v
Compilation/résolution répétée
```

peut être optimisé dans certains cas.

Mais ce n'est pas la première optimisation à appliquer.

Avant cela :

```text
SQL
Index
N+1
Projection
Pagination
Tracking
```

doivent être correctement traités.

---

# 40. Raw SQL

EF Core permet aussi d'utiliser du SQL directement dans certains scénarios.

Exemple conceptuel :

```csharp
var hotels = await context.Hotels
    .FromSql(...)
    .ToListAsync();
```

Cela peut être utile lorsque :

```text
requête SQL spécialisée
fonctionnalité spécifique du moteur
requête difficile à exprimer en LINQ
```

Mais utiliser du SQL brut ne rend pas automatiquement une requête plus performante.

Il faut comparer les résultats et les plans d'exécution.

---

# 41. Attention aux injections SQL

Ne concatène pas directement des entrées utilisateur dans une chaîne SQL.

Mauvais principe :

```csharp
$"SELECT * FROM Hotels WHERE Name = '{name}'"
```

Le SQL paramétré doit être utilisé.

Avec les APIs EF Core appropriées, les valeurs peuvent être paramétrées.

Mentalement :

```text
Donnée utilisateur
    |
    v
Paramètre SQL
    |
    v
Database
```

et non :

```text
Donnée utilisateur
    |
    v
Concaténation SQL
```

---

# 42. Performance et `SaveChanges`

Une autre question de performance concerne les écritures.

Éviter de faire inutilement :

```csharp
foreach (var hotel in hotels)
{
    context.Hotels.Add(hotel);
    await context.SaveChangesAsync();
}
```

Cela peut provoquer un grand nombre d'opérations de persistance.

Selon le scénario, il peut être préférable de préparer les changements puis d'effectuer une sauvegarde adaptée.

Exemple :

```csharp
foreach (var hotel in hotels)
{
    context.Hotels.Add(hotel);
}

await context.SaveChangesAsync();
```

Le comportement exact et les performances dépendent du volume, du provider et de la stratégie utilisée.

---

# 43. Performance et Change Tracker

Lors de grosses opérations, le Change Tracker peut devenir une partie importante du coût.

Il faut donc réfléchir à :

```text
Volume d'entités
Durée de vie du contexte
Tracking
Batching
SaveChanges
```

Mais il ne faut pas désactiver des mécanismes au hasard.

---

# 44. Architecture et performance

Une mauvaise architecture peut également créer des problèmes.

Exemple :

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
Repository
    |
    v
DbContext
```

avec plusieurs couches qui déclenchent chacune des requêtes peut rendre difficile la compréhension du nombre réel de requêtes.

Le code doit permettre de répondre à :

> « Combien de requêtes SQL cette opération métier déclenche-t-elle ? »

---

# 45. Checklist de diagnostic

Lorsqu'une requête EF Core est lente :

```text
[ ] Combien de requêtes SQL sont exécutées ?
[ ] Y a-t-il un N+1 ?
[ ] Quel SQL est généré ?
[ ] Quels champs sont réellement récupérés ?
[ ] Ai-je besoin du tracking ?
[ ] Ai-je besoin d'un Include ?
[ ] Une projection serait-elle meilleure ?
[ ] Ai-je une pagination ?
[ ] Les filtres sont-ils exécutés côté DB ?
[ ] Les colonnes filtrées sont-elles correctement indexées ?
[ ] Y a-t-il trop de JOIN ?
[ ] Y a-t-il une cartesian explosion ?
[ ] Le volume de données est-il raisonnable ?
[ ] Puis-je agréger côté base ?
[ ] Le DbContext a-t-il une durée de vie correcte ?
[ ] Le SQL est-il réellement le goulot d'étranglement ?
```

---

# 46. Règle mentale

Quand une requête est lente, ne pense pas immédiatement :

```text
"EF Core est lent."
```

Pense :

```text
Mon application
      |
      v
LINQ
      |
      v
SQL
      |
      v
Database
      |
      +-- Index ?
      +-- JOIN ?
      +-- Volume ?
      +-- Plan ?
      |
      v
Résultats
      |
      v
Réseau
      |
      v
Matérialisation
      |
      v
Tracking
```

Cherche le vrai goulot d'étranglement.

---

# 47. À retenir

1. Il faut mesurer avant d'optimiser.
2. `Select()` permet de limiter les données récupérées.
3. `AsNoTracking()` peut réduire le coût du tracking pour les lectures pures.
4. `AsNoTracking()` ne règle pas tous les problèmes de performance.
5. Le N+1 est un problème majeur à surveiller.
6. Les `Include()` multiples peuvent produire des requêtes complexes.
7. Les projections sont souvent très utiles dans les APIs.
8. La pagination évite de charger inutilement de gros volumes.
9. `Skip/Take` peut devenir coûteux avec de gros offsets.
10. La keyset pagination peut être intéressante sur de très gros volumes.
11. Les indexes doivent être conçus à partir des requêtes réelles.
12. Les indexes ont aussi un coût sur les écritures et l'espace disque.
13. `ToQueryString()` permet d'inspecter le SQL généré.
14. Le SQL généré doit être analysé lorsque la performance est problématique.
15. `AnyAsync()` est adapté à une vérification d'existence.
16. `CountAsync()` est adapté lorsqu'on veut réellement un nombre.
17. Les agrégations doivent généralement être réalisées côté base lorsque c'est possible.
18. Il faut éviter de matérialiser les données trop tôt.
19. `AsSplitQuery()` peut être utile dans certains graphes complexes.
20. Une cartesian explosion peut apparaître avec plusieurs collections et jointures.
21. Le `DbContext` doit avoir une durée de vie raisonnable.
22. Les requêtes concurrentes ne doivent pas utiliser simultanément le même `DbContext`.
23. Les compiled queries sont une optimisation spécialisée, pas le premier réflexe.
24. Le SQL brut n'est pas automatiquement plus performant.
25. Les données utilisateur ne doivent pas être concaténées directement dans du SQL.
26. La vraie optimisation commence par comprendre le SQL et le volume de données.

---

# Questions d'entretien

### 1. Quelle est la première chose à faire lorsqu'une requête EF Core est lente ?

La mesurer et l'analyser : SQL généré, nombre de requêtes, volume de données, indexes, jointures et temps d'exécution.

### 2. Pourquoi une projection peut-elle améliorer les performances ?

Parce qu'elle permet de récupérer uniquement les colonnes nécessaires au résultat au lieu de matérialiser une entité complète.

### 3. Qu'est-ce que le problème N+1 ?

Une requête récupère une collection puis une requête supplémentaire est exécutée pour chaque élément de cette collection.

### 4. Pourquoi `AsNoTracking()` peut-il améliorer une lecture ?

Parce qu'EF Core n'a pas besoin de conserver les entités dans le Change Tracker pour une lecture qui ne sera pas modifiée via ce contexte.

### 5. Pourquoi `AsNoTracking()` ne suffit-il pas toujours ?

Parce que la lenteur peut provenir du SQL, des indexes, des jointures, du volume de données, du réseau ou du nombre de requêtes.

### 6. Comment voir le SQL généré par une requête ?

On peut notamment utiliser le logging EF Core ou :

```csharp
query.ToQueryString();
```

pour inspecter la représentation SQL.

### 7. Pourquoi les indexes sont-ils importants ?

Ils peuvent accélérer certaines recherches, mais ils ont également un coût en stockage et lors des écritures.

### 8. Quelle différence entre offset pagination et keyset pagination ?

L'offset utilise notamment `Skip/Take`, tandis que la keyset continue à partir d'une valeur de référence, par exemple une clé.

### 9. Pourquoi plusieurs `Include()` peuvent-ils être problématiques ?

Ils peuvent produire de grosses requêtes avec de nombreuses jointures et beaucoup de lignes intermédiaires.

### 10. Qu'est-ce que la cartesian explosion ?

C'est une augmentation importante du nombre de lignes intermédiaires pouvant résulter de plusieurs collections jointes dans une même requête.

### 11. Quand `AsSplitQuery()` peut-il être intéressant ?

Lorsqu'une requête charge plusieurs collections et qu'une seule requête avec toutes les jointures produit un résultat excessivement volumineux ou coûteux.

### 12. Pourquoi faut-il éviter `ToListAsync()` trop tôt ?

Parce qu'il matérialise les données immédiatement et peut empêcher la base d'effectuer efficacement les filtres, tris, projections ou limites.

### 13. Pourquoi ne faut-il pas paralléliser deux requêtes avec le même DbContext ?

Parce qu'une même instance de `DbContext` n'est pas conçue pour exécuter plusieurs opérations simultanément.

### 14. Pourquoi le SQL brut n'est-il pas automatiquement plus performant ?

Parce que la performance dépend de la requête réelle, des indexes, du plan d'exécution, du volume de données et du moteur de base, pas simplement du fait que le SQL soit écrit manuellement.

---

# Phrase à retenir

> **Pour optimiser EF Core, je ne devine pas : je mesure, j'inspecte le SQL, je réduis les données et les requêtes inutiles, puis je vérifie à nouveau le résultat.**
