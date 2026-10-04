# CancellationToken en .NET

## 1. Définition

`CancellationToken` permet de demander à une opération en cours de s'arrêter proprement.

Il faut bien comprendre le mot **demander**.

Un `CancellationToken` ne force généralement pas une méthode à s'arrêter brutalement.

Il fournit un mécanisme coopératif :

```text
Un composant demande l'annulation
             |
             v
     CancellationToken
             |
             v
L'opération vérifie le token
             |
             v
Elle s'arrête proprement
```

L'idée fondamentale est :

> CancellationToken permet aux différentes parties d'une application de coopérer pour annuler une opération longue.

---

# 2. Pourquoi l'annulation est nécessaire ?

Supposons une API qui démarre une opération longue :

```text
Client
  |
  v
GET /api/report
  |
  v
Génération du rapport
  |
  v
Base de données
  |
  v
Calcul
```

Mais le client ferme son navigateur avant la fin.

Continuer inutilement le traitement peut consommer :

- CPU ;
- mémoire ;
- connexions ;
- ressources de base de données ;
- appels vers d'autres services.

L'annulation permet donc de dire :

> « Cette opération n'est plus nécessaire, arrête-la si tu peux le faire proprement. »

---

# 3. `CancellationTokenSource`

Le composant qui demande l'annulation est généralement :

```csharp
CancellationTokenSource
```

Exemple :

```csharp
using var cts = new CancellationTokenSource();

CancellationToken token = cts.Token;

cts.Cancel();
```

Mentalement :

```text
CancellationTokenSource
          |
          +-- possède l'état d'annulation
          |
          +-- expose un CancellationToken
```

Le `CancellationToken` est principalement le moyen de transmettre cet état aux méthodes qui travaillent.

---

# 4. `CancellationToken` vs `CancellationTokenSource`

C'est une distinction importante.

## `CancellationTokenSource`

Il permet de **déclencher** l'annulation.

```csharp
cts.Cancel();
```

## `CancellationToken`

Il permet de **recevoir/observer** la demande d'annulation.

```csharp
token.IsCancellationRequested
```

Mental model :

```text
Source
  |
  | Cancel()
  v
Token
  |
  | IsCancellationRequested
  v
Opération
```

### Phrase à retenir

> `CancellationTokenSource` demande l'annulation ; `CancellationToken` transporte cette demande.

---

# 5. Vérifier `IsCancellationRequested`

Une méthode peut vérifier le token :

```csharp
public async Task ProcessAsync(
    CancellationToken cancellationToken)
{
    for (int i = 0; i < 100; i++)
    {
        if (cancellationToken.IsCancellationRequested)
        {
            return;
        }

        await Task.Delay(100);
    }
}
```

Lorsque l'annulation est demandée :

```csharp
cancellationToken.IsCancellationRequested
```

devient :

```text
true
```

La méthode décide alors de s'arrêter.

---

# 6. `ThrowIfCancellationRequested`

Une autre possibilité est :

```csharp
cancellationToken.ThrowIfCancellationRequested();
```

Si l'annulation n'a pas été demandée :

```text
continuer
```

Si elle a été demandée :

```text
OperationCanceledException
```

est généralement levée.

Exemple :

```csharp
public async Task ProcessAsync(
    CancellationToken cancellationToken)
{
    for (int i = 0; i < 100; i++)
    {
        cancellationToken.ThrowIfCancellationRequested();

        await Task.Delay(100, cancellationToken);
    }
}
```

Cette approche est souvent préférable lorsque l'annulation doit remonter naturellement dans la chaîne d'appels.

---

# 7. Annulation coopérative

Le point le plus important :

```csharp
cancellationToken.Cancel();
```

ne tue pas une méthode de force.

Il indique simplement :

```text
Annulation demandée
```

C'est ensuite à l'opération de respecter cette demande.

Exemple :

```csharp
while (true)
{
    if (token.IsCancellationRequested)
    {
        break;
    }

    // Travail
}
```

Si la méthode ne vérifie jamais le token et n'appelle aucune API qui le respecte, l'annulation peut ne produire aucun effet.

### À retenir

> CancellationToken = coopération, pas interruption forcée.

---

# 8. `Task.Delay` avec CancellationToken

Exemple :

```csharp
await Task.Delay(
    5000,
    cancellationToken);
```

Sans token :

```csharp
await Task.Delay(5000);
```

la méthode attend cinq secondes.

Avec le token :

```csharp
await Task.Delay(
    5000,
    cancellationToken);
```

l'attente peut être annulée.

Cela permet d'éviter d'attendre inutilement lorsque l'opération n'est plus nécessaire.

---

# 9. Propager le token

Supposons :

```csharp
public async Task CreateOrderAsync(
    CancellationToken cancellationToken)
{
    await _repository.SaveAsync(cancellationToken);
}
```

Le contrôleur appelle :

```csharp
await _service.CreateOrderAsync(cancellationToken);
```

Le token traverse donc les couches :

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
Database
```

Chaque couche reçoit le même signal d'annulation.

### Règle mentale

> Quand une méthode reçoit un CancellationToken et appelle une autre opération asynchrone qui accepte un token, il faut généralement le transmettre.

---

# 10. ASP.NET Core et `RequestAborted`

Dans ASP.NET Core, le contexte HTTP expose :

```csharp
HttpContext.RequestAborted
```

Ce token représente l'annulation liée à la requête HTTP.

Exemple :

```csharp
[HttpGet]
public async Task<IActionResult> Get(
    CancellationToken cancellationToken)
{
    var result =
        await _service.GetAsync(cancellationToken);

    return Ok(result);
}
```

ASP.NET Core peut fournir le token correspondant à la requête.

On peut ensuite le transmettre :

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
Repository
      |
      v
EF Core
```

---

# 11. Pourquoi propager le token dans une API ?

Imagine :

```text
Client
  |
  v
API
  |
  v
Service
  |
  v
Database
```

Le client abandonne la requête.

Si toutes les couches respectent le token :

```text
Client disconnect
       |
       v
RequestAborted
       |
       v
Service
       |
       v
Database operation cancelled
```

Les ressources peuvent être libérées plus tôt.

---

# 12. CancellationToken avec EF Core

EF Core fournit de nombreuses méthodes asynchrones acceptant un token.

Exemple :

```csharp
var users = await _context.Users
    .ToListAsync(cancellationToken);
```

Le token peut donc être transmis jusqu'à la couche de données.

Autres exemples :

```csharp
await _context.SaveChangesAsync(
    cancellationToken);
```

ou :

```csharp
await _context.Users
    .FirstOrDefaultAsync(
        u => u.Id == id,
        cancellationToken);
```

Le principe est :

```text
Application
    |
    v
CancellationToken
    |
    v
EF Core
    |
    v
Database provider
```

La possibilité réelle d'interrompre l'opération dépend ensuite du provider et de l'opération exécutée.

---

# 13. CancellationToken avec `HttpClient`

Un appel HTTP peut également accepter un token.

```csharp
var response =
    await _httpClient.GetAsync(
        url,
        cancellationToken);
```

Si l'opération est annulée :

```text
Request HTTP externe
        |
        X
   cancellation
```

Cela évite de laisser des appels externes continuer inutilement lorsque la requête initiale n'est plus pertinente.

---

# 14. Timeout et CancellationToken

On peut utiliser une source avec un délai :

```csharp
using var cts =
    new CancellationTokenSource(
        TimeSpan.FromSeconds(10));
```

Après dix secondes, l'annulation est demandée.

Mentalement :

```text
Start
  |
  v
10 secondes
  |
  v
Cancel()
  |
  v
OperationCanceledException
```

Le timeout devient donc une forme de demande d'annulation.

---

# 15. `CancelAfter`

On peut également faire :

```csharp
using var cts = new CancellationTokenSource();

cts.CancelAfter(TimeSpan.FromSeconds(10));
```

L'annulation sera automatiquement demandée après le délai.

---

# 16. Lier plusieurs tokens

Il peut exister plusieurs raisons d'annuler une opération.

Par exemple :

```text
Client déconnecté
        +
Timeout
        +
Arrêt de l'application
```

On peut créer un token lié :

```csharp
using var linkedCts =
    CancellationTokenSource.CreateLinkedTokenSource(
        requestToken,
        timeoutToken);
```

L'opération reçoit :

```csharp
linkedCts.Token
```

Si l'un des tokens sources est annulé, le token lié est annulé.

Mentalement :

```text
RequestToken ----\
                  \
TimeoutToken ------> LinkedToken ---> Operation
                  /
ShutdownToken ---/
```

---

# 17. `OperationCanceledException`

Lorsqu'une opération asynchrone respecte un token et est annulée, elle peut produire :

```csharp
OperationCanceledException
```

Exemple :

```csharp
try
{
    await service.ExecuteAsync(cancellationToken);
}
catch (OperationCanceledException)
{
    // Opération annulée
}
```

Il faut distinguer :

```text
Exception normale
```

de :

```text
Annulation demandée
```

Une annulation n'est généralement pas une panne inattendue.

---

# 18. `TaskCanceledException`

On peut également rencontrer :

```csharp
TaskCanceledException
```

Elle dérive de :

```text
OperationCanceledException
```

Mentalement :

```text
OperationCanceledException
        |
        +-- TaskCanceledException
```

Il est généralement préférable de raisonner au niveau de :

```csharp
OperationCanceledException
```

lorsqu'on veut gérer l'annulation de manière générale.

---

# 19. Ne pas logger une annulation comme une erreur grave

Mauvais raisonnement :

```csharp
catch (Exception ex)
{
    _logger.LogError(ex, "Everything failed");
}
```

Si l'exception correspond simplement à une annulation normale d'une requête, cela peut polluer les logs.

Il faut distinguer :

```text
Erreur réelle
```

et :

```text
Opération annulée
```

Selon l'architecture, une annulation peut être traitée séparément.

---

# 20. CancellationToken dans une méthode

Bonne signature :

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

Le token fait partie du contrat de la méthode.

Cela indique :

> Cette opération peut être annulée.

---

# 21. Token optionnel

On rencontre souvent :

```csharp
CancellationToken cancellationToken = default
```

Exemple :

```csharp
public async Task<User?> GetUserAsync(
    int id,
    CancellationToken cancellationToken = default)
{
    // ...
}
```

`default` correspond ici à un token qui n'a pas de source d'annulation active.

Cela permet d'appeler :

```csharp
await GetUserAsync(10);
```

ou :

```csharp
await GetUserAsync(10, cancellationToken);
```

---

# 22. `CancellationToken.None`

On peut explicitement indiquer qu'aucune annulation n'est demandée :

```csharp
CancellationToken.None
```

Conceptuellement :

```text
CancellationToken.None
    =
aucune demande d'annulation
```

---

# 23. Attention à `Task.Run`

On voit parfois :

```csharp
await Task.Run(
    () => DoSomething(),
    cancellationToken);
```

Le token ne signifie pas automatiquement que le code CPU s'arrêtera brutalement.

Pour un travail CPU long, le code lui-même doit coopérer :

```csharp
void DoSomething(CancellationToken token)
{
    for (int i = 0; i < 1000000; i++)
    {
        token.ThrowIfCancellationRequested();

        // calcul
    }
}
```

`Task.Run` et `CancellationToken` répondent donc à deux problèmes différents :

```text
Task.Run
    -> où exécuter le travail

CancellationToken
    -> comment demander son annulation
```

---

# 24. CancellationToken et `async/await`

Ils sont souvent utilisés ensemble mais ne sont pas la même chose.

`async/await` permet principalement de gérer efficacement les opérations asynchrones.

`CancellationToken` permet de demander l'arrêt d'une opération.

Exemple :

```csharp
await repository.GetAsync(
    cancellationToken);
```

On a :

```text
async/await
    +
CancellationToken
```

mais leurs rôles sont différents.

### À retenir

> `async/await` concerne le modèle d'exécution asynchrone ; `CancellationToken` concerne l'annulation coopérative.

---

# 25. Propagation à travers une architecture

Une architecture API classique :

```text
Controller
    |
    v
Application Service
    |
    v
Repository
    |
    v
EF Core
```

On veut généralement :

```csharp
Controller
    -> token

Service
    -> token

Repository
    -> token

EF Core
    -> token
```

Exemple :

```csharp
public async Task<User?> GetUserAsync(
    int id,
    CancellationToken token)
{
    return await _repository
        .GetUserAsync(id, token);
}
```

Puis :

```csharp
public async Task<User?> GetUserAsync(
    int id,
    CancellationToken token)
{
    return await _context.Users
        .FirstOrDefaultAsync(
            u => u.Id == id,
            token);
}
```

Le signal traverse toute la chaîne.

---

# 26. Annulation d'une collection asynchrone

Avec `IAsyncEnumerable<T>`, on peut également transmettre un token.

Exemple :

```csharp
await foreach (
    var item in GetItemsAsync()
        .WithCancellation(cancellationToken))
{
    // ...
}
```

Cela permet d'annuler une consommation asynchrone.

---

# 27. Erreurs fréquentes

## Erreur 1 : recevoir le token mais ne pas le transmettre

```csharp
public async Task ExecuteAsync(
    CancellationToken token)
{
    await _repository.SaveAsync();
}
```

Le token reçu n'est jamais utilisé.

Si `SaveAsync` accepte un token :

```csharp
await _repository.SaveAsync(token);
```

est généralement préférable.

---

## Erreur 2 : penser que `Cancel()` tue immédiatement le thread

Faux.

```csharp
cts.Cancel();
```

signifie :

```text
annulation demandée
```

Le code doit coopérer.

---

## Erreur 3 : traiter toute annulation comme une erreur

Une requête abandonnée par un client peut être un événement normal.

---

## Erreur 4 : ne pas annuler les appels externes

Si une API appelle :

```text
API externe
Database
File system
```

il faut vérifier si ces opérations acceptent un `CancellationToken`.

---

## Erreur 5 : oublier le token dans EF Core

Préférer :

```csharp
await query.ToListAsync(token);
```

lorsque l'opération doit être annulable.

---

# 28. Schéma global

```text
                    Cancel()
                       |
                       v
            CancellationTokenSource
                       |
                       v
              CancellationToken
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
   Controller       Service       External API
        |              |
        v              v
   Repository       EF Core
        |
        v
     Database
```

L'annulation est un signal qui peut traverser les différentes couches.

---

# 29. Règle mentale

Quand tu vois :

```csharp
CancellationTokenSource
```

pense :

> « Celui qui peut demander l'annulation. »

Quand tu vois :

```csharp
CancellationToken
```

pense :

> « Le signal d'annulation transmis à l'opération. »

Quand tu vois :

```csharp
token.IsCancellationRequested
```

pense :

> « Est-ce qu'on m'a demandé de m'arrêter ? »

Quand tu vois :

```csharp
token.ThrowIfCancellationRequested()
```

pense :

> « Si on m'a demandé de m'arrêter, je remonte l'annulation. »

Quand tu vois :

```csharp
await operation(token)
```

pense :

> « Je transmets la possibilité d'annuler cette opération. »

---

# 30. À retenir

1. `CancellationToken` permet une annulation coopérative.
2. `CancellationTokenSource` déclenche l'annulation.
3. `CancellationToken` transporte le signal.
4. `Cancel()` ne tue pas brutalement une opération.
5. Une opération doit respecter le token.
6. `IsCancellationRequested` permet de vérifier l'état.
7. `ThrowIfCancellationRequested()` permet de propager l'annulation sous forme d'exception.
8. `Task.Delay`, EF Core et `HttpClient` peuvent accepter un token.
9. ASP.NET Core fournit le token de requête via `RequestAborted`.
10. Il faut généralement propager le token à travers les différentes couches.
11. `OperationCanceledException` représente généralement une annulation coopérative.
12. `TaskCanceledException` dérive de `OperationCanceledException`.
13. Un timeout peut être réalisé avec `CancelAfter`.
14. Plusieurs tokens peuvent être combinés avec `CreateLinkedTokenSource`.
15. `async/await` et `CancellationToken` ont des rôles différents.
16. Une annulation n'est pas nécessairement une erreur.
17. Le token doit être transmis jusqu'aux opérations qui peuvent réellement s'arrêter.

---

# Questions d'entretien

### 1. Quelle différence entre `CancellationTokenSource` et `CancellationToken` ?

`CancellationTokenSource` déclenche l'annulation ; `CancellationToken` est transmis aux opérations pour qu'elles puissent observer et respecter cette demande.

### 2. Est-ce que `CancellationToken` tue un thread ?

Non. Il fournit un mécanisme coopératif. Le code doit vérifier et respecter le token.

### 3. Pourquoi transmettre le token jusqu'à EF Core ?

Pour permettre à une opération de base de données longue d'être annulée lorsque la requête ou l'opération qui l'a déclenchée n'est plus nécessaire.

### 4. Qu'est-ce que `RequestAborted` ?

C'est le `CancellationToken` associé à l'abandon d'une requête HTTP ASP.NET Core.

### 5. Quelle différence entre `async/await` et `CancellationToken` ?

`async/await` permet de gérer les opérations asynchrones ; `CancellationToken` permet de demander leur annulation.

### 6. Que fait `ThrowIfCancellationRequested()` ?

Si l'annulation a été demandée, il lève une `OperationCanceledException`.

### 7. Comment créer un timeout avec `CancellationTokenSource` ?

Par exemple :

```csharp
using var cts =
    new CancellationTokenSource(
        TimeSpan.FromSeconds(10));
```

ou :

```csharp
cts.CancelAfter(TimeSpan.FromSeconds(10));
```

### 8. Pourquoi ne faut-il pas logger systématiquement une annulation comme une erreur ?

Parce qu'une annulation peut être un comportement normal, par exemple lorsqu'un client abandonne une requête.

---

# Phrase à retenir

> **CancellationToken ne force pas l'arrêt d'une opération : il transmet une demande d'annulation que chaque couche doit respecter et propager jusqu'aux opérations réellement annulables.**
