# Async / Await en C#

L'asynchronisme est un concept fondamental de .NET.

Le but n'est pas de rendre automatiquement le code plus rapide.

Le but principal est de permettre à une application de **ne pas bloquer inutilement un thread pendant qu'elle attend une opération**, notamment une opération d'entrée/sortie (I/O).

Exemples d'opérations I/O :

- appel HTTP ;
- requête SQL ;
- lecture d'un fichier ;
- écriture d'un fichier ;
- accès à une API externe ;
- communication réseau.

L'idée essentielle :

```text
Code synchrone
→ le thread attend

Code asynchrone
→ le thread peut être libéré pendant l'attente
```

---

# 1. Le problème du code synchrone

Supposons :

```csharp
var user = GetUserFromDatabase();
```

Si la base de données met 2 secondes à répondre, le thread qui exécute cette méthode reste bloqué pendant l'attente.

Conceptuellement :

```text
Thread
  ↓
envoie requête DB
  ↓
ATTEND 2 secondes
  ↓
continue
```

Dans une application serveur, beaucoup de requêtes simultanées peuvent donc immobiliser beaucoup de threads.

---

# 2. Code asynchrone

On peut écrire :

```csharp
var user = await GetUserFromDatabaseAsync();
```

Pendant que l'opération I/O attend :

```text
Request
  ↓
appel DB
  ↓
attente I/O
  ↓
thread libéré
  ↓
DB répond
  ↓
suite du traitement
```

Le but est surtout d'améliorer l'utilisation des ressources.

---

# 3. `Task`

Une méthode asynchrone retourne très souvent un :

```csharp
Task
```

ou :

```csharp
Task<T>
```

Exemple :

```csharp
Task<User> GetUserAsync();
```

Cela signifie conceptuellement :

> Une opération est en cours ou sera terminée plus tard et produira un `User`.

Important :

```text
Task<User>
≠
User
```

Un `Task<User>` représente l'opération.

Le `User` est le résultat futur.

---

# 4. `Task` vs `Task<T>`

## `Task`

Utilisé lorsqu'il n'y a pas de valeur de retour :

```csharp
public async Task SendEmailAsync()
{
    await ...
}
```

Conceptuellement :

```text
opération terminée
→ pas de résultat
```

## `Task<T>`

Lorsqu'il existe une valeur :

```csharp
public async Task<User> GetUserAsync()
{
    ...
}
```

Conceptuellement :

```text
opération
→ résultat User
```

---

# 5. Que fait `await` ?

Prenons :

```csharp
var user = await GetUserAsync();
```

`await` signifie essentiellement :

> Attends la fin de cette opération asynchrone avant de continuer cette méthode, sans bloquer inutilement le thread pendant l'attente.

Il ne signifie pas :

> Crée un nouveau thread.

C'est une confusion très fréquente.

---

# 6. `async` ne crée pas non plus un thread

Cette méthode :

```csharp
public async Task<User> GetUserAsync()
{
    ...
}
```

ne signifie pas :

```text
nouveau thread
```

`async` indique principalement que la méthode peut utiliser `await` et retourner une opération asynchrone.

Le mécanisme repose sur les `Task`, les continuations et la machine à états générée par le compilateur.

---

# 7. Ce qui se passe derrière `await`

Prenons :

```csharp
public async Task<User> GetUserAsync()
{
    var user = await repository.GetUserAsync();

    return user;
}
```

Conceptuellement :

```text
GetUserAsync()
    ↓
lance / obtient une Task
    ↓
Task pas terminée ?
    ↓
la méthode suspend sa continuation
    ↓
le thread peut faire autre chose
    ↓
Task terminée
    ↓
la continuation reprend
    ↓
return user
```

Le mot important est :

```text
continuation
```

La continuation représente la partie de la méthode qui doit s'exécuter après le `await`.

---

# 8. Pourquoi le compilateur intervient ?

Une méthode `async` contenant `await` est transformée par le compilateur en une structure permettant de conserver son état.

On parle de :

```text
async state machine
```

Conceptuellement :

```text
Avant await
     ↓
suspend
     ↓
état sauvegardé
     ↓
opération terminée
     ↓
reprendre
     ↓
après await
```

Tu n'écris pas cette machine à états manuellement.

Le compilateur génère le mécanisme nécessaire.

---

# 9. `await` ne bloque pas le thread comme `.Wait()`

Compare :

```csharp
var user = GetUserAsync().Result;
```

avec :

```csharp
var user = await GetUserAsync();
```

Le premier cas bloque le thread en attendant.

Le deuxième permet au modèle asynchrone de suspendre la méthode sans bloquer de la même manière.

### Règle mentale

```text
await
→ attendre sans bloquer inutilement

Wait / Result
→ attendre en bloquant
```

---

# 10. Pourquoi `.Result` et `.Wait()` sont dangereux

Exemple :

```csharp
var result = GetDataAsync().Result;
```

ou :

```csharp
GetDataAsync().Wait();
```

Ces appels peuvent :

- bloquer un thread ;
- réduire les performances ;
- provoquer des problèmes de deadlock dans certains contextes ;
- casser la chaîne asynchrone.

Dans du code moderne .NET, on préfère généralement :

```csharp
var result = await GetDataAsync();
```

---

# 11. Le principe de propagation async

Si une méthode appelle une opération asynchrone :

```csharp
public async Task<User> GetUserAsync()
{
    return await repository.GetUserAsync();
}
```

la couche appelante devrait généralement rester asynchrone :

```csharp
public async Task<IActionResult> GetUser()
{
    var user = await service.GetUserAsync();

    return Ok(user);
}
```

On obtient :

```text
Controller
   ↓ await
Service
   ↓ await
Repository
   ↓ await
Database
```

C'est souvent appelé :

```text
async all the way
```

---

# 12. Pourquoi éviter de casser la chaîne ?

Imagine :

```text
Controller async
     ↓
Service async
     ↓
Repository async
     ↓
DB async
```

Puis le service fait :

```csharp
repository.GetAsync().Result
```

La chaîne asynchrone est cassée.

Le thread est bloqué alors que l'opération est conçue pour être asynchrone.

---

# 13. `Task.Run` : erreur fréquente

Beaucoup de développeurs pensent :

```csharp
await Task.Run(() => GetDataFromDatabase());
```

est nécessaire pour rendre une opération asynchrone.

Ce n'est généralement pas le bon modèle pour une I/O.

Si la bibliothèque fournit :

```csharp
await GetDataFromDatabaseAsync();
```

utilise directement cette API.

---

# 14. I/O-bound vs CPU-bound

C'est une distinction fondamentale.

## I/O-bound

Le programme attend principalement une ressource externe :

```text
SQL
HTTP
fichier
réseau
```

L'asynchronisme est particulièrement utile.

Exemple :

```csharp
await httpClient.GetAsync(url);
```

## CPU-bound

Le processeur doit réellement effectuer beaucoup de calcul :

```text
compression
calcul mathématique
traitement d'image
machine learning
```

Dans certains scénarios, `Task.Run` peut être pertinent pour déplacer le travail CPU vers le ThreadPool.

---

# 15. `Task.Run` et CPU-bound

Exemple :

```csharp
var result = await Task.Run(() =>
{
    return CalculateSomethingExpensive();
});
```

Ici, le calcul est exécuté sur un thread du ThreadPool.

Mais :

```text
Task.Run
≠
rend n'importe quelle opération plus performante
```

Pour une I/O déjà asynchrone, `Task.Run` ajoute généralement du travail inutile.

---

# 16. ThreadPool

ASP.NET Core utilise largement le ThreadPool pour traiter le travail.

L'objectif n'est pas d'avoir :

```text
1 requête = 1 thread bloqué pendant toute l'opération
```

Pour une I/O asynchrone, on cherche plutôt :

```text
thread
  ↓
lance I/O
  ↓
libéré pendant attente
  ↓
reprend lorsque nécessaire
```

Cela permet à un serveur de gérer davantage de travail concurrent avec un nombre limité de threads.

---

# 17. Async n'est pas parallèle

C'est une distinction essentielle.

### Asynchronisme

```text
ne pas rester bloqué pendant une attente
```

### Parallélisme

```text
plusieurs calculs exécutés simultanément
```

On peut avoir :

```text
async sans parallèle
```

et :

```text
parallèle sans async
```

Ce sont des concepts différents.

---

# 18. Exemple de deux appels séquentiels

```csharp
var user = await GetUserAsync();
var orders = await GetOrdersAsync();
```

Si les deux opérations sont indépendantes :

```text
GetUser
  ↓ attente
terminé
  ↓
GetOrders
  ↓ attente
terminé
```

La durée totale est approximativement :

```text
durée user + durée orders
```

---

# 19. `Task.WhenAll`

Si les opérations sont indépendantes :

```csharp
var userTask = GetUserAsync();
var ordersTask = GetOrdersAsync();

await Task.WhenAll(userTask, ordersTask);
```

Conceptuellement :

```text
GetUser ────────┐
                ├── WhenAll
GetOrders ──────┘
```

Les deux opérations peuvent progresser en parallèle du point de vue de l'attente.

Puis :

```csharp
var user = await userTask;
var orders = await ordersTask;
```

---

# 20. Pourquoi `WhenAll` peut être plus efficace ?

Supposons :

```text
GetUser = 500 ms
GetOrders = 700 ms
```

Séquentiel :

```text
500 + 700
≈ 1200 ms
```

Concurrent :

```text
max(500, 700)
≈ 700 ms
```

La durée réelle dépend du système et des ressources, mais le principe est :

```text
indépendant → possibilité de concurrence
```

---

# 21. Attention à `WhenAll`

Il ne faut pas utiliser :

```csharp
Task.WhenAll(...)
```

simplement parce que c'est asynchrone.

Les opérations doivent être réellement indépendantes.

Si la deuxième dépend du résultat de la première :

```csharp
var user = await GetUserAsync();
var orders = await GetOrdersAsync(user.Id);
```

elles doivent être séquentielles.

---

# 22. Exceptions avec `await`

Exemple :

```csharp
try
{
    var user = await GetUserAsync();
}
catch (Exception ex)
{
    Log(ex);
}
```

Une exception produite par l'opération asynchrone peut être observée lorsque le `await` reprend.

C'est l'une des raisons pour lesquelles :

```csharp
await
```

est préférable à :

```csharp
.Result
```

dans du code asynchrone.

---

# 23. CancellationToken

Une opération asynchrone peut également accepter :

```csharp
CancellationToken
```

Exemple :

```csharp
public async Task<User> GetUserAsync(
    int id,
    CancellationToken cancellationToken)
{
    return await repository.GetUserAsync(
        id,
        cancellationToken);
}
```

Le token permet de demander :

> Arrête l'opération si elle n'est plus nécessaire.

---

# 24. Pourquoi la cancellation est importante dans ASP.NET Core ?

Imaginons :

```text
Client
  ↓
requête HTTP
  ↓
SQL très long
```

Puis le client abandonne la requête.

Il est inutile de continuer certains traitements coûteux si l'opération n'a plus de consommateur.

Le `CancellationToken` permet de propager l'annulation :

```text
HTTP request
   ↓
Controller
   ↓
Service
   ↓
Repository
   ↓
Database
```

---

# 25. Propager le `CancellationToken`

Une bonne chaîne peut ressembler à :

```csharp
public async Task<User?> GetUserAsync(
    int id,
    CancellationToken cancellationToken)
{
    return await _context.Users
        .FirstOrDefaultAsync(
            u => u.Id == id,
            cancellationToken);
}
```

Puis :

```csharp
public async Task<User?> GetUserAsync(
    int id,
    CancellationToken cancellationToken)
{
    return await _repository.GetUserAsync(
        id,
        cancellationToken);
}
```

Le token voyage avec l'opération.

---

# 26. `Task` est un objet représentant une opération

Il est utile de distinguer :

```csharp
Task<User>
```

de :

```csharp
User
```

`Task<User>` représente :

```text
une opération asynchrone
+
un résultat futur
```

Une fois terminée :

```text
Task<User>
     ↓
User
```

`await` permet d'obtenir le résultat de manière asynchrone :

```csharp
User user = await task;
```

---

# 27. `async Task` vs `async void`

Préférer :

```csharp
async Task
```

ou :

```csharp
async Task<T>
```

Éviter :

```csharp
async void
```

sauf pour certains cas spécifiques, principalement les event handlers.

Pourquoi ?

Parce qu'un `Task` permet à l'appelant de :

- attendre ;
- gérer les exceptions ;
- composer plusieurs opérations ;
- propager l'état de l'opération.

Avec `async void`, l'appelant ne peut pas attendre l'opération de la même manière.

---

# 28. Exemple

Préférer :

```csharp
public async Task SaveAsync()
{
    await repository.SaveAsync();
}
```

plutôt que :

```csharp
public async void SaveAsync()
{
    await repository.SaveAsync();
}
```

L'appelant peut alors :

```csharp
await SaveAsync();
```

---

# 29. Async et retour direct d'une Task

On voit parfois :

```csharp
public Task<User> GetUserAsync()
{
    return repository.GetUserAsync();
}
```

Ici, il n'est pas forcément nécessaire d'écrire :

```csharp
public async Task<User> GetUserAsync()
{
    return await repository.GetUserAsync();
}
```

Les deux peuvent être corrects.

Si la méthode ne fait qu'exposer directement la Task, le premier peut être plus simple.

---

# 30. Quand `async/await` est utile malgré tout ?

Si on doit faire quelque chose autour du résultat :

```csharp
public async Task<User> GetUserAsync()
{
    var user = await repository.GetUserAsync();

    Log(user);

    return user;
}
```

ou gérer des exceptions :

```csharp
public async Task<User> GetUserAsync()
{
    try
    {
        return await repository.GetUserAsync();
    }
    catch (Exception ex)
    {
        Log(ex);
        throw;
    }
}
```

alors `async/await` apporte une vraie valeur.

---

# 31. Éviter `async` inutile

Exemple :

```csharp
public async Task<User> GetUserAsync()
{
    return await _repository.GetUserAsync();
}
```

si aucune logique n'est ajoutée.

On peut souvent écrire :

```csharp
public Task<User> GetUserAsync()
{
    return _repository.GetUserAsync();
}
```

Mais il faut aussi tenir compte de la lisibilité, du besoin de `try/catch`, du `using`, du `finally`, de la transformation du résultat ou de la propagation de cancellation.

---

# 32. Async et `IAsyncEnumerable<T>`

Pour produire des données progressivement, C# propose :

```csharp
IAsyncEnumerable<T>
```

Exemple :

```csharp
await foreach (var item in GetItemsAsync())
{
    Console.WriteLine(item);
}
```

Cela permet de consommer une séquence de manière asynchrone.

Mental model :

```text
IEnumerable<T>
→ séquence synchrone

IAsyncEnumerable<T>
→ séquence consommée de manière asynchrone
```

---

# 33. `await foreach`

Exemple :

```csharp
await foreach (var item in GetItemsAsync())
{
    Process(item);
}
```

Le consommateur peut recevoir les éléments au fur et à mesure sans attendre que toute la séquence soit disponible.

C'est particulièrement intéressant pour certaines sources réseau, fichiers ou flux de données.

---

# 34. Async dans ASP.NET Core

Un contrôleur moderne utilise généralement :

```csharp
[HttpGet("{id}")]
public async Task<IActionResult> Get(int id)
{
    var user = await _service.GetUserAsync(id);

    if (user is null)
    {
        return NotFound();
    }

    return Ok(user);
}
```

La chaîne complète peut être :

```text
HTTP request
   ↓
Controller async
   ↓
Service async
   ↓
EF Core async
   ↓
Database
```

Le but est d'éviter de bloquer inutilement les threads pendant les I/O.

---

# 35. EF Core et async

EF Core fournit des méthodes comme :

```csharp
ToListAsync()
FirstOrDefaultAsync()
SingleAsync()
AnyAsync()
SaveChangesAsync()
```

Exemple :

```csharp
var users = await _context.Users
    .Where(u => u.IsActive)
    .ToListAsync();
```

Ici :

```text
LINQ
   ↓
requête SQL
   ↓
I/O database asynchrone
   ↓
await
```

---

# 36. HTTP et async

Avec `HttpClient` :

```csharp
var response = await _httpClient.GetAsync(url);
```

L'opération attend principalement le réseau.

C'est un cas typique où l'asynchronisme est approprié.

---

# 37. Ce que `await` ne garantit pas

`await` ne garantit pas :

```text
plus rapide
```

Il ne garantit pas :

```text
nouveau thread
```

Il ne garantit pas :

```text
parallélisme
```

Il permet principalement :

```text
attendre une opération asynchrone sans bloquer inutilement le thread
```

---

# 38. Erreurs fréquentes

## Erreur 1 : croire que `async` crée un thread

Faux.

---

## Erreur 2 : utiliser `Task.Run` pour toutes les I/O

Une I/O déjà asynchrone n'a généralement pas besoin d'être enveloppée dans `Task.Run`.

---

## Erreur 3 : utiliser `.Result`

```csharp
var result = GetAsync().Result;
```

Cela bloque.

Préférer :

```csharp
var result = await GetAsync();
```

---

## Erreur 4 : utiliser `.Wait()`

Même problème :

```csharp
GetAsync().Wait();
```

---

## Erreur 5 : faire `async void`

Préférer :

```csharp
Task
```

sauf pour certains event handlers.

---

## Erreur 6 : lancer des opérations dépendantes avec `WhenAll`

Si B dépend de A :

```text
A → B
```

elles ne sont pas indépendantes.

---

## Erreur 7 : oublier `CancellationToken`

Dans les opérations longues ou côté serveur, la cancellation peut être importante.

---

# 39. Comparaison rapide

| Concept | Signification |
|---|---|
| `Task` | opération asynchrone sans résultat |
| `Task<T>` | opération asynchrone avec résultat |
| `async` | permet notamment l'utilisation de `await` |
| `await` | attend une Task sans bloquer inutilement le thread |
| `Task.Run` | déplace du travail vers le ThreadPool |
| `WhenAll` | attend plusieurs Tasks |
| `CancellationToken` | permet de demander l'annulation |
| `IAsyncEnumerable<T>` | séquence consommée de manière asynchrone |

---

# 40. Mental model complet

Quand tu vois :

```csharp
await repository.GetUserAsync();
```

pense :

```text
1. Je démarre/obtiens une opération I/O
2. La Task représente cette opération
3. L'opération n'est peut-être pas terminée
4. Ma méthode peut suspendre sa continuation
5. Le thread peut faire autre chose
6. L'opération se termine
7. La continuation reprend
8. Je récupère le résultat
```

Et surtout :

```text
async/await
≠
nouveau thread

async/await
→
modèle d'attente non bloquante
```

---

# 41. À retenir

1. `Task` représente une opération asynchrone.
2. `Task<T>` représente une opération asynchrone qui produira un `T`.
3. `await` permet d'attendre une Task sans bloquer inutilement le thread.
4. `async` ne crée pas automatiquement un thread.
5. L'asynchronisme est particulièrement utile pour les I/O.
6. I/O-bound et CPU-bound sont deux problèmes différents.
7. `Task.Run` est surtout pertinent pour certains travaux CPU-bound.
8. Il faut éviter `.Result` et `.Wait()` dans une chaîne asynchrone.
9. Il faut généralement propager l'asynchronisme de bout en bout.
10. `Task.WhenAll` permet de faire progresser plusieurs opérations indépendantes ensemble.
11. `CancellationToken` permet de propager une demande d'annulation.
12. `async Task` est généralement préférable à `async void`.
13. `IAsyncEnumerable<T>` permet de traiter une séquence progressivement de manière asynchrone.
14. EF Core et `HttpClient` fournissent des APIs asynchrones adaptées aux I/O.
15. Async ne signifie ni "plus rapide", ni "parallèle", ni "nouveau thread".

---

# 42. Questions d'entretien

### Est-ce que `async` crée un nouveau thread ?

Non. `async` permet notamment d'utiliser `await` et de représenter une opération asynchrone. Le mécanisme ne signifie pas automatiquement création d'un thread.

---

### Quelle est la différence entre `Task` et `Task<T>` ?

`Task` représente une opération sans résultat.

`Task<T>` représente une opération qui produira une valeur de type `T`.

---

### Que fait `await` ?

Il attend la fin d'une opération asynchrone tout en permettant à la méthode de se suspendre sans bloquer inutilement le thread pendant l'attente.

---

### Pourquoi éviter `.Result` et `.Wait()` ?

Parce qu'ils bloquent le thread et peuvent casser les bénéfices de l'asynchronisme, avec des risques de problèmes de blocage dans certains contextes.

---

### Quand utiliser `Task.Run` ?

Principalement lorsque l'on souhaite déplacer un travail CPU-bound vers le ThreadPool dans un contexte où cela est approprié. Il n'est généralement pas nécessaire pour une I/O déjà asynchrone.

---

### Quelle est la différence entre asynchronisme et parallélisme ?

L'asynchronisme concerne principalement l'attente non bloquante d'opérations.

Le parallélisme concerne l'exécution simultanée de plusieurs travaux.

---

### Pourquoi utiliser `Task.WhenAll` ?

Pour attendre plusieurs opérations indépendantes et permettre qu'elles progressent concurremment plutôt que de les attendre séquentiellement.

---

### Pourquoi utiliser `CancellationToken` ?

Pour permettre à une opération et aux couches qu'elle appelle de répondre à une demande d'annulation.

---

### Pourquoi éviter `async void` ?

Parce que l'appelant ne peut pas attendre l'opération via une `Task` et la gestion des exceptions devient plus difficile. `async void` est principalement réservé à certains event handlers.

---

# 43. La phrase à mémoriser

> **`async/await` ne sert pas à créer des threads : il sert principalement à attendre les opérations I/O sans bloquer inutilement les threads, en permettant à l'exécution de reprendre lorsque l'opération est terminée.**
