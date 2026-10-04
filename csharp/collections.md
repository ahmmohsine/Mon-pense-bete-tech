# Collections en C#

Les collections permettent de stocker et manipuler plusieurs valeurs dans une même structure.

En C#, le choix d'une collection dépend principalement de la manière dont on doit :

- accéder aux éléments ;
- rechercher des éléments ;
- ajouter ou supprimer des éléments ;
- garantir l'unicité ;
- respecter un ordre ;
- associer une clé à une valeur.

Le but n'est donc pas de mémoriser toutes les collections, mais de comprendre **pourquoi choisir une collection plutôt qu'une autre**.

---

## 1. Les principales collections

Les collections que l'on rencontre le plus souvent sont :

| Collection | Idée principale | Accès typique |
|---|---|---|
| `List<T>` | Liste ordonnée d'éléments | par index |
| `Dictionary<TKey,TValue>` | Association clé → valeur | par clé |
| `HashSet<T>` | Ensemble d'éléments uniques | recherche d'appartenance |
| `Queue<T>` | Premier entré, premier sorti | FIFO |
| `Stack<T>` | Dernier entré, premier sorti | LIFO |

La question à se poser est :

> **Comment vais-je utiliser mes données ?**

---

# 2. `List<T>`

## Définition

`List<T>` représente une collection ordonnée d'éléments accessibles par leur index.

```csharp
var users = new List<string>
{
    "Alice",
    "Bob",
    "Charlie"
};
```

Les index commencent à `0` :

```text
Index      0        1         2
           ↓        ↓         ↓
        Alice      Bob      Charlie
```

On peut récupérer un élément avec :

```csharp
var user = users[1];
```

Résultat :

```text
Bob
```

---

## Ajouter un élément

```csharp
users.Add("David");
```

Ajouter plusieurs éléments :

```csharp
users.AddRange(new[]
{
    "Emma",
    "Frank"
});
```

---

## Supprimer

```csharp
users.Remove("Bob");
```

Ou par index :

```csharp
users.RemoveAt(1);
```

---

## Rechercher

```csharp
bool exists = users.Contains("Alice");
```

On peut également utiliser LINQ :

```csharp
var user = users.FirstOrDefault(u => u == "Alice");
```

---

## Fonctionnement interne

`List<T>` est basée sur un tableau interne.

Conceptuellement :

```text
List<T>
   |
   v
[ A ][ B ][ C ][   ][   ][   ]
```

La liste possède une **Capacity**, c'est-à-dire la taille du tableau interne, et un **Count**, c'est-à-dire le nombre réel d'éléments.

```csharp
users.Count
users.Capacity
```

Exemple conceptuel :

```text
Capacity = 8
Count    = 3

[ A ][ B ][ C ][   ][   ][   ][   ][   ]
```

Si la capacité devient insuffisante, `List<T>` doit créer un tableau interne plus grand et copier les éléments.

C'est pourquoi l'ajout à la fin est généralement très efficace, même si certains ajouts peuvent provoquer une réallocation.

---

## Complexité courante

| Opération | Complexité moyenne |
|---|---:|
| Accès par index | O(1) |
| Ajout à la fin | O(1) amorti |
| Recherche par valeur | O(n) |
| Suppression par valeur | O(n) |
| Insertion au début | O(n) |

### À retenir

`List<T>` est un bon choix lorsque :

- l'ordre des éléments compte ;
- on veut accéder aux éléments par index ;
- on parcourt fréquemment la collection ;
- on ajoute principalement à la fin.

---

# 3. `Dictionary<TKey,TValue>`

## Définition

Un `Dictionary<TKey,TValue>` associe une **clé** à une **valeur**.

```csharp
var users = new Dictionary<int, string>
{
    [1] = "Alice",
    [2] = "Bob",
    [3] = "Charlie"
};
```

On peut ensuite récupérer un utilisateur avec sa clé :

```csharp
var user = users[2];
```

Résultat :

```text
Bob
```

Mentalement :

```text
Clé        Valeur

1    --->   Alice
2    --->   Bob
3    --->   Charlie
```

---

## Pourquoi utiliser un Dictionary ?

Supposons :

```csharp
var users = new List<User>();
```

Si tu veux trouver l'utilisateur ayant l'identifiant `5000`, tu peux devoir parcourir la liste :

```text
User 1
User 2
User 3
...
User 5000
```

Avec un `Dictionary` :

```csharp
usersById[5000]
```

la collection est conçue pour effectuer une recherche par clé beaucoup plus efficacement.

---

## `TryGetValue`

Une erreur fréquente est d'utiliser directement :

```csharp
var user = users[100];
```

si la clé n'existe pas, une `KeyNotFoundException` est levée.

Pour rechercher sans exception :

```csharp
if (users.TryGetValue(100, out var user))
{
    Console.WriteLine(user);
}
```

C'est souvent la manière préférable lorsqu'on ne sait pas si la clé existe.

---

## Fonctionnement interne

Le `Dictionary` utilise une structure basée sur le **hachage**.

Conceptuellement :

```text
clé
 |
 v
GetHashCode()
 |
 v
emplacement interne
 |
 v
valeur
```

C'est pourquoi le type de clé doit avoir un comportement correct concernant l'égalité et le hash.

Le framework utilise notamment :

```csharp
GetHashCode()
Equals(...)
```

pour déterminer où chercher et si deux clés sont considérées comme identiques.

---

## Complexité courante

| Opération | Complexité moyenne |
|---|---:|
| Recherche par clé | O(1) |
| Ajout | O(1) |
| Suppression | O(1) |
| Recherche d'une valeur | O(n) |

Ces complexités sont des moyennes. Dans certaines situations particulières, les performances peuvent être moins bonnes.

### À retenir

Utilise un `Dictionary` lorsque ton besoin principal est :

> **clé → valeur**

Exemples :

```text
UserId -> User
ProductId -> Product
Code -> Description
Email -> User
```

---

# 4. `HashSet<T>`

## Définition

`HashSet<T>` représente un ensemble dans lequel les éléments sont uniques.

```csharp
var roles = new HashSet<string>();

roles.Add("Admin");
roles.Add("User");
roles.Add("Admin");
```

Même si `"Admin"` est ajouté deux fois :

```text
Admin
User
```

il n'y aura qu'un seul `"Admin"`.

---

## Pourquoi utiliser un HashSet ?

Supposons :

```csharp
var roles = new List<string>();
```

Tu dois vérifier qu'un rôle n'existe pas avant de l'ajouter :

```csharp
if (!roles.Contains("Admin"))
{
    roles.Add("Admin");
}
```

Avec un `HashSet` :

```csharp
roles.Add("Admin");
```

La collection gère directement l'unicité.

---

## Vérifier l'existence

```csharp
if (roles.Contains("Admin"))
{
    ...
}
```

Le `HashSet` est particulièrement adapté aux tests d'appartenance :

> "Est-ce que cet élément existe dans l'ensemble ?"

---

## Fonctionnement interne

Comme `Dictionary`, `HashSet` utilise une structure basée sur le hachage.

Conceptuellement :

```text
élément
   |
   v
GetHashCode()
   |
   v
structure de hachage
```

L'égalité est ensuite vérifiée lorsque cela est nécessaire.

Cela explique pourquoi `HashSet<T>` est généralement très efficace pour les recherches d'appartenance.

---

## Complexité courante

| Opération | Complexité moyenne |
|---|---:|
| `Add` | O(1) |
| `Contains` | O(1) |
| `Remove` | O(1) |

### À retenir

Utilise `HashSet<T>` lorsque ton besoin principal est :

> **"Je veux un ensemble d'éléments uniques et vérifier rapidement leur présence."**

---

# 5. `Queue<T>`

## Définition

`Queue<T>` fonctionne selon le principe :

> **FIFO — First In, First Out**

Le premier élément ajouté est le premier à sortir.

```csharp
var queue = new Queue<string>();

queue.Enqueue("Alice");
queue.Enqueue("Bob");
queue.Enqueue("Charlie");
```

Conceptuellement :

```text
Entrée

Alice -> Bob -> Charlie
  |
  v
sortie
```

On récupère le premier élément avec :

```csharp
var user = queue.Dequeue();
```

Résultat :

```text
Alice
```

La file devient :

```text
Bob -> Charlie
```

---

## `Peek()`

Si tu veux regarder le premier élément sans le supprimer :

```csharp
var user = queue.Peek();
```

---

## Exemple concret

Une file d'attente de tâches :

```text
Tâche 1
Tâche 2
Tâche 3
```

On traite :

```text
Tâche 1
   ↓
Tâche 2
   ↓
Tâche 3
```

C'est exactement le principe FIFO.

---

# 6. `Stack<T>`

## Définition

`Stack<T>` fonctionne selon le principe :

> **LIFO — Last In, First Out**

Le dernier élément ajouté est le premier à sortir.

```csharp
var stack = new Stack<string>();

stack.Push("A");
stack.Push("B");
stack.Push("C");
```

Conceptuellement :

```text
   C  <- dernier ajouté
   B
   A
```

Si on fait :

```csharp
var value = stack.Pop();
```

on obtient :

```text
C
```

---

## `Peek()`

Regarder le sommet sans le supprimer :

```csharp
var value = stack.Peek();
```

---

## Exemple concret

Imagine une pile d'assiettes :

```text
Assiette C
Assiette B
Assiette A
```

Tu retires naturellement l'assiette du dessus.

C'est le principe LIFO.

---

# 7. Comparaison rapide

| Collection | Question à laquelle elle répond |
|---|---|
| `List<T>` | "Je veux une liste ordonnée et accéder par index." |
| `Dictionary<TKey,TValue>` | "Je connais une clé et je veux retrouver une valeur." |
| `HashSet<T>` | "Cet élément existe-t-il déjà ?" |
| `Queue<T>` | "Qui est arrivé en premier ?" |
| `Stack<T>` | "Quel est le dernier élément ajouté ?" |

---

# 8. Comment choisir ?

Ne mémorise pas seulement les noms.

Pars du besoin.

### Besoin 1

> Je veux une liste de produits et accéder au produit numéro 5.

```csharp
List<Product>
```

---

### Besoin 2

> Je connais l'ID du produit et je veux retrouver rapidement le produit.

```csharp
Dictionary<int, Product>
```

---

### Besoin 3

> Je veux garantir qu'un utilisateur ne possède pas deux fois le même rôle.

```csharp
HashSet<string>
```

---

### Besoin 4

> Je dois traiter les demandes dans l'ordre d'arrivée.

```csharp
Queue<Request>
```

---

### Besoin 5

> Je dois toujours traiter le dernier élément ajouté en premier.

```csharp
Stack<T>
```

---

# 9. Collection ou tableau ?

On peut également utiliser un tableau :

```csharp
var numbers = new int[5];
```

Un tableau possède une taille fixe.

```text
[ ][ ][ ][ ][ ]
```

Une fois créé :

```csharp
numbers.Length
```

reste fixe.

À l'inverse :

```csharp
var numbers = new List<int>();
```

peut grandir et réallouer son stockage interne lorsque nécessaire.

### Règle simple

Utilise généralement un tableau lorsque :

- la taille est connue et fixe ;
- tu as un besoin spécifique de tableau.

Utilise généralement `List<T>` lorsque :

- la collection doit pouvoir grandir ou rétrécir ;
- tu veux une API de collection plus pratique.

---

# 10. `IEnumerable<T>` n'est pas une collection concrète

Cette distinction est importante.

```csharp
IEnumerable<User>
```

ne signifie pas nécessairement :

```text
List<User>
```

`IEnumerable<T>` représente principalement quelque chose que l'on peut **parcourir**.

Exemple :

```csharp
IEnumerable<User> users = GetUsers();
```

On peut faire :

```csharp
foreach (var user in users)
{
    Console.WriteLine(user.Name);
}
```

Mais on ne possède pas nécessairement les opérations propres à `List<T>` :

```csharp
users.Add(...)
users.RemoveAt(...)
```

---

## Règle mentale

```text
IEnumerable<T>
    =
"Je peux parcourir ces éléments."

List<T>
    =
"Je possède une liste concrète et modifiable."
```

Cette distinction devient particulièrement importante avec LINQ et Entity Framework Core.

---

# 11. `IEnumerable<T>` vs `IQueryable<T>`

Dans une application .NET utilisant EF Core, cette distinction devient encore plus importante.

```csharp
IQueryable<User>
```

représente une requête qui peut être traduite par un provider, par exemple en SQL.

Exemple :

```csharp
var query = dbContext.Users
    .Where(u => u.IsActive);
```

Avec EF Core, cette expression peut être traduite en SQL.

À l'inverse, une fois les données chargées en mémoire :

```csharp
var users = await dbContext.Users.ToListAsync();
```

on possède :

```csharp
List<User>
```

et les opérations LINQ suivantes s'exécutent en mémoire.

### À retenir

```text
IQueryable
    ↓
requête potentiellement traduite vers la base

ToListAsync()
    ↓
données chargées en mémoire

List<T>
    ↓
traitement en mémoire
```

---

# 12. Erreur fréquente : choisir une collection sans regarder le besoin

Exemple :

```csharp
var users = new List<User>();
```

puis :

```csharp
var user = users.FirstOrDefault(u => u.Id == id);
```

Si cette recherche est effectuée très souvent sur une très grande collection, `List<T>` n'est peut-être pas la meilleure structure.

On peut parfois avoir :

```csharp
var users = new Dictionary<int, User>();
```

puis :

```csharp
users.TryGetValue(id, out var user);
```

La bonne structure dépend donc du **pattern d'accès aux données**.

---

# 13. Erreur fréquente : penser qu'une collection est toujours plus rapide qu'une autre

Il n'existe pas de collection universellement "meilleure".

Par exemple :

```text
List<T>
```

est excellente pour l'accès par index :

```csharp
list[500]
```

mais une recherche par valeur :

```csharp
list.Contains(value)
```

est généralement O(n).

Un :

```text
HashSet<T>
```

est excellent pour :

```csharp
set.Contains(value)
```

mais ne fournit pas le même modèle d'accès par index.

### Règle mentale

> Une collection est optimisée pour certains usages.

Le choix doit venir du besoin, pas de la préférence personnelle.

---

# 14. Résumé des complexités

Pour les opérations les plus courantes :

| Collection | Accès / recherche | Ajout | Suppression |
|---|---:|---:|---:|
| `List<T>` par index | O(1) | O(1) amorti à la fin | O(n) selon le cas |
| `Dictionary<TKey,TValue>` par clé | O(1) moyen | O(1) moyen | O(1) moyen |
| `HashSet<T>` par valeur | O(1) moyen | O(1) moyen | O(1) moyen |
| `Queue<T>` | selon opération | O(1) | O(1) |
| `Stack<T>` | sommet | O(1) | O(1) |

Les complexités indiquées pour les structures basées sur le hachage sont des complexités moyennes et peuvent dépendre de la qualité de la distribution des hash.

---

# 15. Règle mentale finale

Quand tu dois choisir une collection, pose-toi cette question :

```text
Quel est mon besoin principal ?
```

Puis :

```text
Index ?
    ↓
List<T>

Clé → valeur ?
    ↓
Dictionary<TKey,TValue>

Unicité / appartenance ?
    ↓
HashSet<T>

Premier arrivé → premier traité ?
    ↓
Queue<T>

Dernier arrivé → premier traité ?
    ↓
Stack<T>
```

---

# 16. À retenir

- `List<T>` : liste ordonnée avec accès par index.
- `Dictionary<TKey,TValue>` : association clé → valeur.
- `HashSet<T>` : éléments uniques et recherche d'appartenance efficace.
- `Queue<T>` : FIFO.
- `Stack<T>` : LIFO.
- Les collections n'ont pas toutes les mêmes performances.
- `List<T>` utilise un tableau interne redimensionnable.
- `Dictionary<TKey,TValue>` et `HashSet<T>` utilisent le hachage.
- `IEnumerable<T>` décrit surtout la possibilité de parcourir une séquence.
- `IQueryable<T>` permet notamment à EF Core de construire une requête pouvant être traduite vers la base de données.
- Le meilleur choix dépend toujours de la manière dont les données seront utilisées.

---

# Questions d'entretien

Avant de considérer cette fiche comme acquise, tu devrais pouvoir répondre sans regarder la réponse :

1. Quelle est la différence fondamentale entre `List<T>` et `Dictionary<TKey,TValue>` ?
2. Pourquoi une recherche dans une `List<T>` est-elle généralement O(n) ?
3. Pourquoi `Dictionary<TKey,TValue>` permet-il généralement une recherche en O(1) moyen ?
4. Quelle est la différence entre `Dictionary` et `HashSet` ?
5. Que signifient FIFO et LIFO ?
6. Quelle collection utiliserais-tu pour gérer une file d'attente de commandes ?
7. Pourquoi `List<T>` peut-elle avoir une `Capacity` supérieure à son `Count` ?
8. Quelle différence entre `IEnumerable<T>` et `List<T>` ?
9. Pourquoi la distinction `IEnumerable<T>` / `IQueryable<T>` est-elle importante avec EF Core ?
10. Si tu dois faire des milliers de recherches par `Id`, pourquoi un `Dictionary<int, User>` peut-il être préférable à une `List<User>` ?
