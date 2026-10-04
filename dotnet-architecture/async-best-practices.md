# Gestion de l'Asynchronisme en .NET Core : Bonnes Pratiques & Pitfalls

Ce document détaille les règles d'or, les principes de fonctionnement interne et les pièges classiques de l'asynchronisme (`async`/`await`) dans les applications .NET Core modernes (Web API, Services backend).

---

# Comprendre l'asynchronisme en .NET Core
## `async`, `await`, `Task`, ThreadPool et I/O

> Objectif : comprendre **comment et pourquoi** l'asynchronisme fonctionne en .NET, plutôt que simplement mémoriser la syntaxe.

---

## 1. L'idée fondamentale

En .NET, `async/await` n'a pas pour objectif principal de rendre une opération individuelle plus rapide.

Son objectif est surtout d'améliorer :

- le **throughput** : nombre de requêtes traitées pendant une période donnée ;
- la **scalabilité** : capacité à gérer davantage de requêtes simultanément ;
- l'utilisation des threads du **ThreadPool**.

### Règle mentale

> **Async ne signifie pas "plus rapide".**
>
> Cela signifie surtout : **"ne bloque pas inutilement un thread pendant qu'on attend quelque chose."**

---

# 2. Deux types de travail : I/O-bound et CPU-bound

Avant de comprendre `async/await`, il faut distinguer deux catégories de travail.

## 2.1 I/O-bound

I/O signifie **Input/Output**.

Exemples :

- requête SQL ;
- appel HTTP vers une autre API ;
- lecture d'un fichier ;
- écriture dans un fichier ;
- accès réseau.

Dans ces situations, le programme passe souvent une grande partie de son temps à **attendre un autre système**.

Exemple :

```csharp
var user = await dbContext.Users
    .FirstOrDefaultAsync(u => u.Id == id);
```

Le CPU ne travaille pas continuellement pendant l'attente de la base de données.

### Idée à retenir

```text
Application
    |
    | "Donne-moi l'utilisateur 42"
    v
Base de données
    |
    | ... traitement ...
    | ... réponse ...
    v
Application
```

Pendant cette attente, garder un thread bloqué est généralement inutile.

C'est précisément là que l'asynchronisme est particulièrement intéressant.

---

## 2.2 CPU-bound

Une opération CPU-bound nécessite réellement du temps de calcul.

Exemples :

- calcul mathématique très lourd ;
- compression ;
- traitement d'image ;
- chiffrement ;
- analyse complexe de données.

Exemple :

```csharp
var result = CalculateSomethingVeryExpensive();
```

Ici, le processeur travaille réellement.

Mettre simplement `await` autour d'une opération CPU-bound ne la rend pas automatiquement plus rapide.

---

# 3. Le ThreadPool .NET

ASP.NET Core utilise largement le **ThreadPool .NET**.

On peut l'imaginer comme une réserve de threads réutilisables.

```text
             ThreadPool
        ┌───────────────────┐
        │ Thread 1           │
        │ Thread 2           │
        │ Thread 3           │
        │ Thread 4           │
        │ ...                │
        └───────────────────┘
```

Lorsqu'une requête HTTP arrive :

```text
Client
  |
  v
ASP.NET Core
  |
  v
ThreadPool
  |
  v
Exécution du code
```

Le problème apparaît lorsqu'un thread reste bloqué pendant une opération d'I/O.

---

# 4. Le problème du code synchrone

Prenons :

```csharp
public User GetUser(int id)
{
    return dbContext.Users
        .FirstOrDefault(u => u.Id == id);
}
```

La méthode effectue une requête SQL de manière synchrone.

Pendant que SQL Server travaille :

```text
Thread 1
   |
   |----> SQL Server
   |
   | ATTEND...
   |
   | ATTEND...
   |
   | <---- réponse
   |
continue
```

Le thread reste occupé à attendre.

Si beaucoup de requêtes arrivent :

```text
Requête 1 -> Thread 1 -> attend SQL
Requête 2 -> Thread 2 -> attend SQL
Requête 3 -> Thread 3 -> attend SQL
Requête 4 -> Thread 4 -> attend SQL
...
```

On peut finir par manquer de threads disponibles.

---

# 5. Ce que change `async/await`

Version asynchrone :

```csharp
public async Task<User?> GetUserAsync(int id)
{
    return await dbContext.Users
        .FirstOrDefaultAsync(u => u.Id == id);
}
```

Le point important est `await`.

Lorsque l'opération I/O n'est pas terminée, le thread n'a pas besoin de rester bloqué.

Conceptuellement :

```text
Thread 1
   |
   | démarre la requête SQL
   |
   | await
   |
   X ----> Thread libéré
            |
            v
        peut traiter
        une autre requête
```

Plus tard, lorsque l'opération est terminée :

```text
SQL Server
    |
    | réponse
    v
.NET
    |
    v
Continuation de la méthode
```

Un thread disponible peut alors reprendre l'exécution.

---

# 6. Attention : `await` ne crée pas un nouveau thread

C'est une erreur très fréquente.

On pourrait croire :

```csharp
await SomeOperationAsync();
```

signifie :

> "Crée un nouveau thread et exécute l'opération dessus."

Ce n'est pas le principe.

Pour une opération I/O asynchrone, l'idée est plutôt :

```text
Thread
  |
  | démarre I/O
  v
await
  |
  | thread libéré
  v
autre travail
```

Puis :

```text
I/O terminée
      |
      v
Continuation
      |
      v
suite de la méthode
```

### À retenir

> `await` permet surtout de **ne pas bloquer le thread pendant l'attente**.

---

# 7. Que représente réellement `Task` ?

Considérons :

```csharp
Task<User> GetUserAsync();
```

`Task<User>` ne signifie pas :

> "Voici un User."

Cela signifie :

> "Voici une opération qui produira éventuellement un User."

On peut voir `Task<T>` comme une promesse d'un résultat futur.

```text
Task<User>
     |
     | pas encore terminé
     v
   [ ... ]
     |
     | opération terminée
     v
   User
```

### Comparaison

Synchrone :

```csharp
User user = GetUser();
```

La méthode doit fournir le `User` avant de continuer.

Asynchrone :

```csharp
Task<User> task = GetUserAsync();
```

On possède immédiatement une représentation de l'opération en cours.

Puis :

```csharp
User user = await task;
```

On demande à la méthode :

> "Reprends-moi lorsque le résultat est disponible."

---

# 8. Pourquoi une méthode `async` retourne souvent `Task`

Exemple :

```csharp
public async Task<User?> GetUserAsync()
```

Il y a deux concepts différents :

```text
Task<User?>
   |
   +---- Task = opération asynchrone
   |
   +---- User? = résultat produit par l'opération
```

Pour une méthode qui ne retourne rien :

```csharp
public async Task DeleteUserAsync(int id)
```

Le résultat est simplement :

```text
Task
 |
 +-- l'opération sera terminée
```

---

# 9. Le fonctionnement conceptuel de `await`

Prenons :

```csharp
public async Task<User?> GetUserAsync(int id)
{
    var user = await dbContext.Users
        .FirstOrDefaultAsync(u => u.Id == id);

    return user;
}
```

Conceptuellement, on peut imaginer :

```text
1. Démarrer l'opération SQL
              |
              v
2. L'opération est-elle terminée ?
              |
       +------+------+
       |             |
      Oui            Non
       |             |
       v             v
Continuer       Suspendre la
la méthode       continuation
                     |
                     v
              libérer le thread
                     |
                     v
             SQL termine
                     |
                     v
             reprendre la suite
```

Le mot important ici est **continuation**.

La partie du code située après `await` constitue conceptuellement la suite qui devra être exécutée lorsque l'opération sera terminée.

---

# 10. Exemple avec plusieurs requêtes HTTP

Supposons une API qui reçoit :

```text
100 requêtes HTTP
```

Chaque requête doit appeler une base de données.

### Approche bloquante

```text
Requête 1 -> Thread 1 -> attente DB
Requête 2 -> Thread 2 -> attente DB
Requête 3 -> Thread 3 -> attente DB
...
```

Les threads sont occupés pendant l'attente.

### Approche asynchrone

```text
Requête 1 -> démarre DB -> await -> thread disponible
Requête 2 -> démarre DB -> await -> thread disponible
Requête 3 -> démarre DB -> await -> thread disponible
...
```

Les threads peuvent être réutilisés pour d'autres travaux pendant les attentes I/O.

### Résultat

L'application peut généralement gérer davantage de requêtes concurrentes avec moins de threads bloqués.

---

# 11. `async` tout seul ne fait rien de magique

Cette méthode :

```csharp
public async Task<int> CalculateAsync()
{
    return 42;
}
```

n'apporte pas de bénéfice d'I/O.

Il n'y a aucune véritable attente asynchrone.

Le mot `async` permet notamment d'utiliser `await` et indique au compilateur que la méthode suit le modèle asynchrone.

### Règle mentale

> Ce n'est pas le mot `async` qui rend une opération asynchrone.
>
> C'est surtout l'opération appelée qui doit proposer une véritable API asynchrone.

Exemples :

```csharp
FirstOrDefaultAsync()
ToListAsync()
SaveChangesAsync()
ReadAsync()
SendAsync()
```

---

# 12. Pourquoi `Task.Run()` n'est généralement pas nécessaire dans une Web API

Erreur classique :

```csharp
public async Task<User> GetUserAsync()
{
    return await Task.Run(() =>
        dbContext.Users.FirstOrDefault());
}
```

Cela ne transforme pas une requête SQL synchrone en véritable I/O asynchrone.

On déplace simplement le travail vers un thread du ThreadPool.

```text
Thread HTTP
     |
     v
Task.Run()
     |
     v
autre ThreadPool thread
     |
     v
requête SQL bloquante
```

Le thread reste bloqué quelque part.

Avec Entity Framework Core, il vaut mieux utiliser directement :

```csharp
await dbContext.Users
    .FirstOrDefaultAsync();
```

### À retenir

> Pour de l'I/O : privilégier la vraie API asynchrone.
>
> `Task.Run()` n'est pas un bouton "rendre asynchrone".

---

# 13. Piège : oublier `await`

Supposons :

```csharp
public async Task<User?> GetUserAsync(int id)
{
    var user = dbContext.Users
        .FirstOrDefaultAsync(u => u.Id == id);

    return user;
}
```

Ici :

```csharp
var user = ...
```

contient un :

```text
Task<User?>
```

et non un :

```text
User?
```

Il faut :

```csharp
var user = await dbContext.Users
    .FirstOrDefaultAsync(u => u.Id == id);
```

### Question à se poser

Quand tu vois :

```csharp
Task<User>
```

demande-toi :

> "Est-ce que j'ai l'opération, ou le résultat de l'opération ?"

Réponse :

```text
Task<User> -> l'opération
User       -> le résultat
```

---

# 14. Piège : `.Result` et `.Wait()`

Éviter dans du code serveur :

```csharp
var user = GetUserAsync().Result;
```

ou :

```csharp
GetUserAsync().Wait();
```

Pourquoi ?

Parce qu'on transforme une opération asynchrone en attente bloquante.

```text
Thread
  |
  | démarre async
  |
  | .Result
  |
  X BLOQUÉ
```

Le bénéfice de l'asynchronisme est alors perdu.

Préférer :

```csharp
var user = await GetUserAsync();
```

### Règle simple

> **Async all the way** : lorsqu'une méthode est naturellement asynchrone, fais remonter l'asynchronisme jusqu'à l'appelant.

---

# 15. Propagation de l'asynchronisme

Exemple :

```csharp
Controller
    |
    v
Service
    |
    v
Repository
    |
    v
Database
```

Si la base de données possède une API asynchrone :

```csharp
Repository
    -> ToListAsync()
```

alors on peut conserver l'asynchronisme :

```text
Controller
    |
    | await
    v
Service
    |
    | await
    v
Repository
    |
    | await
    v
Database
```

Exemple :

```csharp
public async Task<List<User>> GetUsersAsync()
{
    return await _repository.GetUsersAsync();
}
```

Puis :

```csharp
public async Task<IActionResult> GetUsers()
{
    var users = await _service.GetUsersAsync();

    return Ok(users);
}
```

---

# 16. Pourquoi c'est important dans ASP.NET Core

Une Web API peut recevoir énormément de requêtes concurrentes.

Le problème n'est pas seulement :

> "Combien de temps prend une requête ?"

Mais également :

> "Combien de requêtes puis-je gérer simultanément sans épuiser mes ressources ?"

C'est une question de **scalabilité**.

```text
                 API
                  |
       +----------+----------+
       |          |          |
    Request    Request    Request
       |          |          |
      I/O        I/O        I/O
       |          |          |
     await      await      await
       |          |          |
       +----------+----------+
                  |
        threads réutilisés
```

---

# 17. `async/await` ne signifie pas "exécution parallèle"

Autre confusion fréquente.

Ce code :

```csharp
var user = await GetUserAsync();
var orders = await GetOrdersAsync();
```

effectue les opérations l'une après l'autre.

```text
GetUser
   |
   v
terminé
   |
   v
GetOrders
   |
   v
terminé
```

Si les deux opérations sont indépendantes, on peut parfois les démarrer ensemble :

```csharp
var userTask = GetUserAsync();
var ordersTask = GetOrdersAsync();

var user = await userTask;
var orders = await ordersTask;
```

Conceptuellement :

```text
GetUser   ────────────────┐
                          |
GetOrders ────────────────┤
                          v
                    résultats
```

Ou avec `Task.WhenAll()` lorsque cela correspond réellement au besoin :

```csharp
await Task.WhenAll(userTask, ordersTask);
```

### Attention

Paralléliser des opérations n'est pas automatiquement meilleur.

Il faut tenir compte :

- de la charge de la base de données ;
- des limites de l'API externe ;
- des connexions disponibles ;
- des dépendances entre opérations.

---

# 18. Une analogie pour mémoriser

Imagine un serveur comme un restaurant.

### Synchrone bloquant

Un serveur prend une commande :

```text
Client -> "Je veux un plat"
```

Le serveur va en cuisine et reste devant le four jusqu'à ce que le plat soit prêt.

Pendant ce temps :

```text
Serveur = bloqué
```

Il ne peut pas servir les autres clients.

### Asynchrone

Le serveur donne la commande à la cuisine :

```text
Serveur -> Cuisine
```

Puis il retourne servir d'autres clients.

Quand la cuisine termine :

```text
Cuisine -> "Commande prête"
```

Le serveur revient terminer le service.

### Correspondance .NET

```text
Serveur       = Thread
Cuisine       = système externe / DB / réseau
Commande      = opération I/O
"Attendre"    = await
Client suivant = autre requête
```

### À retenir

> `await` revient conceptuellement à dire :
>
> **"Je reviendrai quand l'opération sera terminée ; en attendant, utilise ce thread pour autre chose."**

---

# 19. Les erreurs classiques à reconnaître

## Erreur 1 — Bloquer avec `.Result`

```csharp
var result = GetSomethingAsync().Result;
```

### Problème

Une opération asynchrone est transformée en attente bloquante.

### Préférer

```csharp
var result = await GetSomethingAsync();
```

---

## Erreur 2 — Utiliser `.Wait()`

```csharp
GetSomethingAsync().Wait();
```

Même problème :

```text
async
  +
Wait()
  =
attente bloquante
```

---

## Erreur 3 — Utiliser `Task.Run()` pour une requête SQL

```csharp
await Task.Run(() => db.Users.ToList());
```

Ce n'est pas la bonne manière de rendre une I/O réellement asynchrone.

Préférer :

```csharp
await db.Users.ToListAsync();
```

---

## Erreur 4 — Croire que `async` crée un thread

```csharp
async Task ...
```

ne signifie pas :

```text
"Créer un thread."
```

Le modèle async est principalement conçu pour permettre de ne pas bloquer pendant les opérations asynchrones.

---

## Erreur 5 — Utiliser `async` partout sans raison

Ne pas écrire automatiquement :

```csharp
async
```

sur toutes les méthodes.

Une méthode purement synchrone et très simple n'a pas besoin de devenir artificiellement asynchrone.

---

# 20. Schéma mental complet

Quand tu rencontres :

```csharp
var data = await service.GetDataAsync();
```

pense :

```text
                    API
                     |
                     v
              GetDataAsync()
                     |
                     v
               démarre I/O
                     |
                     v
                   await
                     |
             +-------+-------+
             |               |
        I/O terminée      I/O en attente
             |               |
             v               v
         continuer      thread libéré
                             |
                             v
                       autres requêtes
                             |
                             v
                       I/O terminée
                             |
                             v
                       continuation
                             |
                             v
                       suite du code
```

---

# 21. Checklist de développeur .NET

Quand tu écris une Web API, pose-toi ces questions :

### 1. Est-ce une opération I/O ?

```text
SQL ?
HTTP ?
Fichier ?
Réseau ?
```

Si oui, cherche l'API asynchrone.

---

### 2. Existe-t-il une version `Async` ?

Exemples :

```csharp
ToListAsync()
FirstOrDefaultAsync()
SaveChangesAsync()
ReadAsync()
SendAsync()
```

Si oui, utilise-la lorsque le contexte s'y prête.

---

### 3. Est-ce que je bloque avec `.Result` ou `.Wait()` ?

Si oui :

```text
STOP
```

Cherche à propager `async/await`.

---

### 4. Est-ce que j'utilise `Task.Run()` pour de l'I/O ?

Si oui, vérifie pourquoi.

Dans une Web API, ce n'est généralement pas la bonne solution pour les I/O.

---

### 5. Mes opérations sont-elles indépendantes ?

Si oui, il peut être possible de les lancer de manière concurrente avec :

```csharp
Task.WhenAll(...)
```

Mais seulement si cela apporte réellement un bénéfice.

---

# 22. Résumé : les 7 règles à retenir

1. **`async/await` améliore surtout la scalabilité, pas la vitesse d'une opération individuelle.**

2. **L'asynchronisme est particulièrement utile pour les opérations I/O-bound.**

3. **`await` évite de garder un thread bloqué pendant l'attente d'une opération asynchrone.**

4. **`Task<T>` représente une opération qui produira un `T`.**

5. **`await Task<T>` permet d'obtenir le `T`.**

6. **Évite `.Result` et `.Wait()` dans les chemins asynchrones.**

7. **Pour l'I/O, utilise les vraies API asynchrones (`ToListAsync`, `SaveChangesAsync`, etc.) plutôt que `Task.Run()`.**

---

# 23. La phrase à mémoriser

> **Async ne rend pas forcément mon travail plus rapide.**
>
> **Il évite surtout de gaspiller un thread pendant que mon application attend une ressource externe.**

C'est cette idée qui permet ensuite de comprendre naturellement :

- `Task`
- `async`
- `await`
- `Task.WhenAll`
- ThreadPool
- Entity Framework Core async
- appels HTTP async
- scalabilité ASP.NET Core
- cancellation avec `CancellationToken`
- concurrence vs parallélisme
