# LINQ en C#

LINQ signifie **Language Integrated Query**.

C'est un ensemble d'outils de C# permettant de rechercher, filtrer, transformer, trier, regrouper et agréger des données avec une syntaxe cohérente.

L'objectif n'est pas seulement de connaître `Where`, `Select` ou `OrderBy`, mais de comprendre ce qui se passe derrière ces méthodes.

---

## 1. Pourquoi LINQ ?

Sans LINQ, on écrit souvent des boucles pour parcourir une collection :

```csharp
var adults = new List<User>();

foreach (var user in users)
{
    if (user.Age >= 18)
    {
        adults.Add(user);
    }
}
```

Avec LINQ :

```csharp
var adults = users
    .Where(user => user.Age >= 18)
    .ToList();
```

Le code décrit davantage **ce que l'on veut obtenir** que **comment parcourir manuellement la collection**.

### À retenir

LINQ permet de transformer une collection en une autre représentation de manière déclarative.

---

# 2. Le modèle mental de LINQ

Imagine une chaîne :

```text
Collection
    ↓
Where
    ↓
Select
    ↓
OrderBy
    ↓
ToList
    ↓
Résultat final
```

Chaque opération construit une étape de traitement.

Exemple :

```csharp
var result = users
    .Where(u => u.Age >= 18)
    .Select(u => u.Name)
    .OrderBy(name => name)
    .ToList();
```

On peut le lire comme :

> Prends les utilisateurs adultes, récupère leur nom, trie les noms, puis crée une liste.

---

# 3. La syntaxe méthode

C'est la syntaxe la plus utilisée dans les projets modernes :

```csharp
var result = users
    .Where(u => u.Age >= 18)
    .Select(u => u.Name)
    .ToList();
```

Les méthodes comme `Where`, `Select` et `OrderBy` sont principalement des **méthodes d'extension**.

Elles sont disponibles grâce à :

```csharp
using System.Linq;
```

Dans les versions modernes de .NET, les imports implicites peuvent parfois éviter d'écrire explicitement ce `using`.

---

# 4. La syntaxe query

LINQ possède également une syntaxe proche du SQL :

```csharp
var result =
    from user in users
    where user.Age >= 18
    select user.Name;
```

Elle peut être utilisée dans certains cas, mais la syntaxe méthode est très courante :

```csharp
var result = users
    .Where(user => user.Age >= 18)
    .Select(user => user.Name);
```

### Règle pratique

Connaître les deux est utile.

Pour le code quotidien, tu rencontreras très souvent la syntaxe :

```csharp
.Where(...)
.Select(...)
.OrderBy(...)
```

---

# 5. `Where` : filtrer

`Where` conserve uniquement les éléments qui respectent une condition.

```csharp
var adults = users
    .Where(u => u.Age >= 18);
```

La lambda :

```csharp
u => u.Age >= 18
```

signifie :

> Pour chaque `u`, retourne `true` si `u.Age >= 18`.

Exemple :

```text
Alice  25  → true  → conservée
Bob    15  → false → supprimé
Sarah  32  → true  → conservée
```

---

# 6. `Select` : transformer

`Select` ne filtre pas.

Il transforme chaque élément.

```csharp
var names = users
    .Select(u => u.Name);
```

Si on a :

```text
User
 ├── Id
 ├── Name
 └── Age
```

`Select` peut transformer :

```text
User → string
```

Exemple :

```csharp
var names = users
    .Select(u => u.Name)
    .ToList();
```

Résultat :

```text
["Alice", "Bob", "Sarah"]
```

### Différence fondamentale

```csharp
Where
```

réduit ou conserve des éléments.

```csharp
Select
```

transforme chaque élément.

---

# 7. `Where` + `Select`

C'est une combinaison extrêmement fréquente :

```csharp
var adultNames = users
    .Where(u => u.Age >= 18)
    .Select(u => u.Name)
    .ToList();
```

Lecture :

1. Filtrer les adultes.
2. Transformer les utilisateurs en noms.
3. Matérialiser le résultat en liste.

---

# 8. Les lambdas derrière LINQ

Cette expression :

```csharp
u => u.Age >= 18
```

est une lambda.

Conceptuellement, elle représente une fonction :

```text
User → bool
```

Par exemple :

```csharp
bool IsAdult(User u)
{
    return u.Age >= 18;
}
```

On pourrait donc imaginer :

```csharp
users.Where(IsAdult);
```

La lambda permet d'écrire directement cette logique :

```csharp
users.Where(u => u.Age >= 18);
```

### À retenir

LINQ et les lambdas vont très souvent ensemble.

---

# 9. `OrderBy` et `ThenBy`

Pour trier :

```csharp
var users = users
    .OrderBy(u => u.Name)
    .ToList();
```

Tri décroissant :

```csharp
var users = users
    .OrderByDescending(u => u.Age)
    .ToList();
```

Tri sur plusieurs critères :

```csharp
var result = users
    .OrderBy(u => u.LastName)
    .ThenBy(u => u.FirstName)
    .ToList();
```

Lecture :

> Trie d'abord par nom, puis par prénom lorsque deux noms sont identiques.

Pour inverser le deuxième critère :

```csharp
.ThenByDescending(u => u.FirstName)
```

---

# 10. `First` et `FirstOrDefault`

## `First`

Retourne le premier élément correspondant.

```csharp
var user = users
    .First(u => u.Age >= 18);
```

Si aucun élément ne correspond, `First` lève une exception.

---

## `FirstOrDefault`

```csharp
var user = users
    .FirstOrDefault(u => u.Age >= 18);
```

S'il n'existe aucun élément correspondant, la méthode retourne la valeur par défaut.

Pour une référence :

```text
null
```

Depuis les versions modernes de C#, le type nullable doit être pris en compte :

```csharp
User? user = users.FirstOrDefault(u => u.Id == id);
```

### Règle mentale

```text
First        → "Je suis certain qu'il existe."
FirstOrDefault → "Il peut ne pas exister."
```

---

# 11. `Single` et `SingleOrDefault`

Ces méthodes ont une différence importante avec `First`.

`Single` signifie :

> Je veux exactement un élément.

```csharp
var user = users.Single(u => u.Id == id);
```

Une exception est levée si :

- aucun élément ne correspond ;
- plusieurs éléments correspondent.

`SingleOrDefault` accepte l'absence d'élément :

```csharp
var user = users.SingleOrDefault(u => u.Id == id);
```

Mais plusieurs résultats restent une erreur.

### Différence mentale

```text
First
    → donne-moi le premier

Single
    → donne-moi l'unique
```

Si la base garantit qu'un `Id` est unique, `Single` peut exprimer cette intention.

---

# 12. `Any` : vérifier l'existence

Très utile :

```csharp
if (users.Any(u => u.Age >= 18))
{
    // Au moins un adulte existe
}
```

`Any` répond à une question :

> Est-ce qu'au moins un élément existe ?

### Mauvaise habitude

```csharp
users.Count() > 0
```

Si l'objectif est uniquement de savoir si un élément existe, préfère :

```csharp
users.Any()
```

ou :

```csharp
users.Any(u => u.Age >= 18)
```

`Any` peut s'arrêter dès qu'il trouve un élément correspondant.

---

# 13. `All`

`All` vérifie que tous les éléments respectent une condition :

```csharp
bool allAdults = users.All(u => u.Age >= 18);
```

Lecture :

> Est-ce que tous les utilisateurs sont adultes ?

Attention : sur une séquence vide, `All(...)` retourne `true`.

C'est une conséquence logique importante à connaître.

---

# 14. `Count`

```csharp
int count = users.Count();
```

Avec une condition :

```csharp
int adults = users.Count(u => u.Age >= 18);
```

Attention à ne pas confondre :

```csharp
users.Count
```

et :

```csharp
users.Count()
```

`Count` est une propriété de nombreuses collections :

```csharp
List<User> users;

int count = users.Count;
```

`Count()` est une méthode LINQ :

```csharp
IEnumerable<User> users;

int count = users.Count();
```

---

# 15. Agrégations

LINQ permet également de calculer des valeurs.

## `Sum`

```csharp
int total = products.Sum(p => p.Price);
```

## `Average`

```csharp
double average = products.Average(p => p.Price);
```

## `Min`

```csharp
decimal min = products.Min(p => p.Price);
```

## `Max`

```csharp
decimal max = products.Max(p => p.Price);
```

## `Aggregate`

`Aggregate` permet de définir une accumulation personnalisée :

```csharp
int result = numbers.Aggregate(
    0,
    (total, number) => total + number
);
```

Conceptuellement :

```text
accumulateur + élément → nouvel accumulateur
```

`Aggregate` est puissant, mais il ne faut pas l'utiliser simplement pour rendre un code plus compliqué.

---

# 16. `Distinct`

Supprimer les doublons :

```csharp
var categories = products
    .Select(p => p.Category)
    .Distinct()
    .ToList();
```

Exemple :

```text
Phone
Laptop
Phone
Tablet
Laptop
```

devient :

```text
Phone
Laptop
Tablet
```

La notion d'égalité utilisée par `Distinct` est importante : elle dépend de la manière dont les éléments définissent leur égalité et, selon la surcharge utilisée, du comparateur fourni.

---

# 17. `SelectMany`

`SelectMany` est l'une des méthodes LINQ les plus importantes à comprendre.

Supposons :

```csharp
class User
{
    public string Name { get; set; }
    public List<string> Roles { get; set; }
}
```

On veut récupérer tous les rôles :

```csharp
var roles = users
    .SelectMany(u => u.Roles)
    .ToList();
```

Sans `SelectMany`, on obtient conceptuellement :

```text
User 1 → ["Admin", "User"]
User 2 → ["User"]
User 3 → ["Manager", "User"]
```

Avec `SelectMany` :

```text
["Admin", "User", "User", "Manager", "User"]
```

### Règle mentale

```text
Select    → 1 élément → 1 résultat
SelectMany → 1 élément → plusieurs résultats aplatis
```

C'est le concept de **flattening**.

---

# 18. `GroupBy`

Regrouper les éléments :

```csharp
var groups = products
    .GroupBy(p => p.Category);
```

Conceptuellement :

```text
Electronics
    → Product A
    → Product B

Books
    → Product C
    → Product D
```

On peut ensuite transformer les groupes :

```csharp
var result = products
    .GroupBy(p => p.Category)
    .Select(g => new
    {
        Category = g.Key,
        Count = g.Count()
    })
    .ToList();
```

Résultat conceptuel :

```text
Electronics → 2
Books       → 2
```

---

# 19. `Join`

`Join` permet de combiner deux séquences à partir d'une clé.

Exemple :

```csharp
var result = users.Join(
    orders,
    user => user.Id,
    order => order.UserId,
    (user, order) => new
    {
        UserName = user.Name,
        OrderId = order.Id
    });
```

Conceptuellement :

```text
users
    User.Id

orders
    Order.UserId

       ↓ Join

User + Order
```

Cela ressemble conceptuellement à un `INNER JOIN` SQL.

---

# 20. `ToList`, `ToArray`, `ToDictionary`

Ces méthodes sont importantes parce qu'elles **matérialisent** le résultat.

Exemple :

```csharp
var query = users
    .Where(u => u.Age >= 18);
```

Ici, on peut avoir une requête différée.

Avec :

```csharp
var list = query.ToList();
```

le résultat est exécuté et stocké dans une `List<User>`.

Autres matérialisations :

```csharp
.ToArray()
.ToDictionary(...)
.ToHashSet()
```

### Règle mentale

```text
LINQ peut construire une requête
        ↓
ToList / ToArray / ToDictionary
        ↓
exécution + stockage du résultat
```

---

# 21. Deferred Execution

C'est un concept fondamental de LINQ.

Exemple :

```csharp
var query = users.Where(u => u.Age >= 18);
```

On n'a pas forcément immédiatement parcouru toute la collection.

Puis :

```csharp
foreach (var user in query)
{
    Console.WriteLine(user.Name);
}
```

C'est à ce moment que la séquence est parcourue.

Ou :

```csharp
var list = query.ToList();
```

qui force la matérialisation.

---

# 22. Pourquoi la deferred execution est importante ?

Considérons :

```csharp
var query = users.Where(u => u.Age >= 18);

users.Add(new User
{
    Name = "Sarah",
    Age = 25
});

var result = query.ToList();
```

Selon le type de source, la nouvelle donnée peut être prise en compte au moment de l'énumération.

Si on avait fait :

```csharp
var result = users
    .Where(u => u.Age >= 18)
    .ToList();
```

le résultat aurait déjà été matérialisé.

Puis :

```csharp
users.Add(...);
```

n'ajouterait pas automatiquement l'utilisateur à `result`.

### Mental model

```text
IEnumerable + LINQ
        ↓
"requête à exécuter"
        ↓
énumération / ToList
        ↓
résultat
```

---

# 23. `IEnumerable<T>` et LINQ

Beaucoup de méthodes LINQ travaillent sur :

```csharp
IEnumerable<T>
```

Par exemple :

```csharp
IEnumerable<User> adults =
    users.Where(u => u.Age >= 18);
```

Cela signifie :

> une séquence d'utilisateurs que l'on peut parcourir.

`IEnumerable<T>` est particulièrement associé au traitement **en mémoire**.

---

# 24. `IQueryable<T>` et Entity Framework Core

Avec Entity Framework Core, on rencontre :

```csharp
IQueryable<T>
```

Exemple :

```csharp
var users = dbContext.Users
    .Where(u => u.Age >= 18)
    .OrderBy(u => u.Name)
    .ToListAsync();
```

Ici, LINQ ne signifie pas nécessairement :

> charge tous les utilisateurs puis filtre en C#.

EF Core peut traduire l'expression en SQL.

Conceptuellement :

```text
LINQ
  ↓
Expression
  ↓
EF Core
  ↓
SQL
  ↓
Database
```

Par exemple, quelque chose ressemblant à :

```sql
SELECT ...
FROM Users
WHERE Age >= 18
ORDER BY Name
```

C'est une différence fondamentale avec une collection déjà en mémoire.

---

# 25. `IEnumerable` vs `IQueryable`

### `IEnumerable`

```csharp
IEnumerable<User>
```

Le traitement se fait généralement côté application, sur des objets déjà disponibles en mémoire.

### `IQueryable`

```csharp
IQueryable<User>
```

La requête peut être construite puis traduite par le fournisseur, par exemple EF Core vers SQL.

### Mental model

```text
IEnumerable
→ "travaille sur mes objets"

IQueryable
→ "construis une requête que mon fournisseur peut traduire"
```

---

# 26. `Func` et `Expression<Func<...>>`

C'est un point très important avec EF Core.

Une lambda peut être utilisée comme délégué :

```csharp
Func<User, bool> predicate =
    u => u.Age >= 18;
```

Elle représente du code exécutable en mémoire.

Mais avec `IQueryable`, on peut rencontrer :

```csharp
Expression<Func<User, bool>> predicate =
    u => u.Age >= 18;
```

Ici, l'expression peut être représentée sous forme d'un **arbre d'expression**.

EF Core peut analyser cet arbre pour essayer de produire du SQL.

Conceptuellement :

```text
Func<User, bool>
        ↓
code à exécuter

Expression<Func<User, bool>>
        ↓
description du code
        ↓
analyse par EF Core
        ↓
SQL
```

C'est une des raisons pour lesquelles une même syntaxe lambda peut avoir des comportements différents selon qu'elle travaille sur `IEnumerable` ou `IQueryable`.

---

# 27. Attention au `ToList()` trop tôt

Supposons :

```csharp
var users = dbContext.Users
    .ToList();

var adults = users
    .Where(u => u.Age >= 18)
    .ToList();
```

Le premier `ToList()` force le chargement des utilisateurs.

Le filtrage est ensuite effectué en mémoire.

Souvent, il vaut mieux :

```csharp
var adults = await dbContext.Users
    .Where(u => u.Age >= 18)
    .ToListAsync();
```

Le filtre peut alors être envoyé à la base de données.

### Règle

Avec EF Core :

```text
Construire la requête
        ↓
Filtrer / projeter
        ↓
Matérialiser le plus tard possible
```

Cela permet souvent d'éviter de récupérer des données inutiles.

---

# 28. Projection avec `Select` en EF Core

Éviter :

```csharp
var users = await dbContext.Users
    .ToListAsync();
```

si on n'a besoin que du nom.

On peut projeter :

```csharp
var names = await dbContext.Users
    .Select(u => u.Name)
    .ToListAsync();
```

Conceptuellement :

```text
Mauvais réflexe :
Database → toutes les colonnes → application

Meilleur réflexe si nécessaire :
Database → seulement Name → application
```

Avec un DTO :

```csharp
var result = await dbContext.Users
    .Select(u => new UserDto
    {
        Id = u.Id,
        Name = u.Name
    })
    .ToListAsync();
```

La projection est particulièrement importante dans les Web APIs.

---

# 29. `Any()` plutôt que `Count() > 0`

À éviter quand on veut simplement vérifier l'existence :

```csharp
if (users.Count() > 0)
{
}
```

Préférer :

```csharp
if (users.Any())
{
}
```

Avec EF Core :

```csharp
await dbContext.Users.AnyAsync(u => u.Email == email);
```

L'intention est claire :

> Existe-t-il au moins un utilisateur correspondant ?

---

# 30. `First` vs `Single`

Erreur fréquente :

```csharp
var user = users.First(u => u.Email == email);
```

si `Email` est censé être unique.

`First` signifie seulement :

> donne-moi le premier.

`Single` exprime davantage la contrainte :

```csharp
var user = users.Single(u => u.Email == email);
```

Cela permet de détecter le cas où plusieurs utilisateurs existent alors qu'un seul était attendu.

Dans une application réelle, le choix dépend aussi des contraintes de la base de données et de la manière dont l'absence de résultat doit être gérée.

---

# 31. Attention aux multiples énumérations

Exemple :

```csharp
var query = users.Where(u => u.Age >= 18);

var count = query.Count();
var first = query.First();
```

La séquence peut être parcourue plusieurs fois.

Avec une source coûteuse, cela peut devenir problématique.

Il faut donc réfléchir à :

```csharp
var list = query.ToList();
```

mais seulement si la matérialisation est réellement utile.

### Attention

`ToList()` n'est pas une solution magique.

Elle consomme de la mémoire et peut déclencher une requête complète avec EF Core.

---

# 32. `Select` ne modifie généralement pas la collection originale

Exemple :

```csharp
var names = users
    .Select(u => u.Name)
    .ToList();
```

Cela crée une nouvelle séquence de résultats.

La collection `users` n'est pas automatiquement transformée en liste de noms.

Même principe :

```csharp
var adults = users.Where(u => u.Age >= 18);
```

Ne signifie pas :

> supprime les mineurs de `users`.

Cela signifie :

> crée une séquence représentant les utilisateurs adultes.

---

# 33. LINQ et immutabilité

Les opérateurs LINQ classiques ne modifient généralement pas la source.

Par exemple :

```csharp
var sorted = users
    .OrderBy(u => u.Name)
    .ToList();
```

On obtient un résultat trié, mais la collection source n'est pas réordonnée par `OrderBy`.

Mentalement :

```text
Source
   │
   ├── LINQ
   ↓
Nouvelle séquence
```

---

# 34. Erreurs fréquentes

## Erreur 1 : confondre `Where` et `Select`

```csharp
.Where(u => u.Age >= 18)
```

→ filtre.

```csharp
.Select(u => u.Name)
```

→ transforme.

---

## Erreur 2 : utiliser `Count() > 0`

Préférer :

```csharp
.Any()
```

---

## Erreur 3 : utiliser `First()` alors que l'élément peut être absent

Utiliser éventuellement :

```csharp
FirstOrDefault()
```

selon le comportement attendu.

---

## Erreur 4 : utiliser `Single()` sans comprendre sa contrainte

`Single()` exige exactement un résultat.

---

## Erreur 5 : appeler `ToList()` trop tôt avec EF Core

Cela peut faire passer le traitement de la base vers la mémoire.

---

## Erreur 6 : oublier la deferred execution

Une requête LINQ n'est pas forcément exécutée au moment où elle est créée.

---

## Erreur 7 : utiliser `Select` alors qu'on veut aplatir une collection

Si chaque élément contient plusieurs éléments :

```csharp
SelectMany()
```

est souvent la bonne solution.

---

# 35. Méthodes essentielles à connaître

| Méthode | Rôle |
|---|---|
| `Where` | Filtrer |
| `Select` | Transformer |
| `SelectMany` | Transformer + aplatir |
| `OrderBy` | Trier |
| `ThenBy` | Deuxième critère de tri |
| `First` | Premier élément |
| `FirstOrDefault` | Premier ou valeur par défaut |
| `Single` | Exactement un |
| `SingleOrDefault` | Zéro ou un |
| `Any` | Au moins un ? |
| `All` | Tous ? |
| `Count` | Compter |
| `Sum` | Additionner |
| `Average` | Moyenne |
| `Min` | Minimum |
| `Max` | Maximum |
| `Distinct` | Supprimer les doublons |
| `GroupBy` | Regrouper |
| `Join` | Joindre deux séquences |
| `ToList` | Matérialiser en liste |
| `ToArray` | Matérialiser en tableau |
| `ToDictionary` | Matérialiser en dictionnaire |

---

# 36. Comment lire une chaîne LINQ

Prenons :

```csharp
var result = orders
    .Where(o => o.Total > 100)
    .Select(o => new OrderDto
    {
        Id = o.Id,
        Total = o.Total
    })
    .OrderByDescending(o => o.Total)
    .ToList();
```

Lis-la étape par étape :

### Étape 1

```csharp
.Where(o => o.Total > 100)
```

Garde uniquement les commandes de plus de 100 €.

### Étape 2

```csharp
.Select(...)
```

Transforme chaque commande en `OrderDto`.

### Étape 3

```csharp
.OrderByDescending(o => o.Total)
```

Trie par montant décroissant.

### Étape 4

```csharp
.ToList()
```

Exécute/matérialise le résultat en liste.

### Mental model

```text
Source
 ↓
Filtrer
 ↓
Transformer
 ↓
Trier
 ↓
Matérialiser
```

---

# 37. LINQ dans une Web API .NET

LINQ est omniprésent dans les APIs .NET.

Exemple avec EF Core :

```csharp
public async Task<List<UserDto>> GetUsersAsync()
{
    return await _context.Users
        .Where(u => u.IsActive)
        .OrderBy(u => u.Name)
        .Select(u => new UserDto
        {
            Id = u.Id,
            Name = u.Name
        })
        .ToListAsync();
}
```

On peut lire :

```text
DbSet<User>
    ↓
Where
    ↓
OrderBy
    ↓
Select DTO
    ↓
ToListAsync
    ↓
Database
```

Cette façon de penser est très utile pour comprendre les repositories, services et handlers dans une architecture .NET.

---

# 38. Pourquoi `Select` est important pour les DTOs

Dans une API, on ne veut généralement pas retourner directement toutes les propriétés de l'entité.

Exemple :

```csharp
.Select(u => new UserDto
{
    Id = u.Id,
    Name = u.Name
})
```

Cela permet de contrôler les données exposées.

C'est donc à la fois :

- une technique LINQ ;
- une technique de projection ;
- une bonne pratique API.

---

# 39. Performance : ce qu'il faut comprendre

Il ne faut pas apprendre une règle du type :

> LINQ est toujours lent.

Ce n'est pas aussi simple.

Il faut comprendre **où le traitement se fait**.

### Collection en mémoire

```csharp
users.Where(...)
```

Le traitement est effectué par .NET.

### EF Core

```csharp
_context.Users.Where(...)
```

Le fournisseur peut traduire la requête en SQL.

Donc :

```text
Même syntaxe LINQ
        ↓
Sources différentes
        ↓
Comportements d'exécution différents
```

C'est l'une des idées les plus importantes de LINQ.

---

# 40. Règle mentale à retenir

Quand tu vois :

```csharp
.Where(...)
.Select(...)
.OrderBy(...)
.ToList()
```

pense :

```text
WHERE
→ filtrer

SELECT
→ transformer / projeter

ORDER BY
→ trier

ToList
→ exécuter / matérialiser
```

Et quand tu vois :

```csharp
IQueryable
```

pense :

```text
"Ma requête peut encore être traduite par le fournisseur."
```

Quand tu vois :

```csharp
IEnumerable
```

pense :

```text
"Je travaille sur une séquence que .NET peut énumérer."
```

---

# 41. À retenir

1. `Where` filtre.
2. `Select` transforme.
3. `SelectMany` aplatit.
4. `OrderBy` trie.
5. `First` prend le premier.
6. `Single` exige exactement un résultat.
7. `Any` vérifie l'existence.
8. `All` vérifie une condition sur tous.
9. `ToList()` matérialise le résultat.
10. LINQ utilise énormément les lambdas.
11. LINQ peut utiliser l'exécution différée.
12. `IEnumerable` et `IQueryable` ne fonctionnent pas de la même manière.
13. Avec EF Core, une requête LINQ peut être traduite en SQL.
14. Éviter de faire `ToList()` trop tôt.
15. `Select` est essentiel pour projeter vers des DTOs.

---

# 42. Questions d'entretien

### Quelle est la différence entre `Where` et `Select` ?

`Where` filtre les éléments.

`Select` transforme les éléments.

---

### Quelle est la différence entre `First` et `Single` ?

`First` retourne le premier élément correspondant.

`Single` exige qu'il y ait exactement un élément correspondant.

---

### Pourquoi utiliser `Any()` plutôt que `Count() > 0` ?

Parce que `Any()` exprime directement l'intention et peut s'arrêter dès qu'un élément est trouvé.

---

### Qu'est-ce que la deferred execution ?

C'est le fait qu'une requête LINQ puisse être construite sans être immédiatement exécutée. L'exécution intervient généralement lors de l'énumération ou d'une matérialisation comme `ToList()`.

---

### Quelle est la différence entre `IEnumerable` et `IQueryable` ?

`IEnumerable` représente une séquence énumérable, généralement traitée en mémoire.

`IQueryable` permet de construire une requête pouvant être interprétée et traduite par un fournisseur comme Entity Framework Core.

---

### Pourquoi éviter `ToList()` trop tôt avec EF Core ?

Parce qu'il peut provoquer l'exécution immédiate de la requête et faire revenir les données en mémoire avant que les autres opérations LINQ soient appliquées.

---

### Quelle est la différence entre `Select` et `SelectMany` ?

`Select` produit généralement un résultat par élément.

`SelectMany` permet de produire plusieurs résultats par élément et d'aplatir les séquences.

---

# 43. La phrase à mémoriser

> **LINQ est une manière de décrire une transformation de données ; les opérateurs construisent cette transformation, et l'exécution dépend de la source et du moment où la séquence est énumérée ou matérialisée.**
