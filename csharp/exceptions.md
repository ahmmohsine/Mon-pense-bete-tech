# Exceptions en C#

Les exceptions permettent de gérer les situations anormales qui empêchent le programme de poursuivre normalement son traitement.

L'objectif n'est pas seulement de savoir écrire `try/catch`, mais de comprendre :

- ce qu'est réellement une exception ;
- comment elle remonte dans la pile d'appels ;
- quand utiliser `throw` ;
- quand créer une exception personnalisée ;
- pourquoi il ne faut pas attraper toutes les exceptions sans réfléchir ;
- comment gérer les exceptions correctement dans une application .NET / Web API.

---

# 1. Qu'est-ce qu'une exception ?

Une exception représente une situation exceptionnelle ou une erreur d'exécution.

Exemple :

```csharp
int result = 10 / 0;
```

Une division entière par zéro provoque une exception.

Autre exemple :

```csharp
var user = users.First();
```

Si `users` est vide, `First()` peut lever une exception.

On peut voir une exception comme :

```text
Le programme rencontre un problème
        ↓
Une exception est créée
        ↓
Elle est levée
        ↓
.NET cherche un gestionnaire
        ↓
catch
```

---

# 2. `try`, `catch`, `finally`

La structure classique :

```csharp
try
{
    // Code susceptible de provoquer une exception
}
catch
{
    // Gestion de l'exception
}
finally
{
    // Code exécuté à la fin
}
```

Exemple :

```csharp
try
{
    int result = 10 / 0;
}
catch
{
    Console.WriteLine("Une erreur est survenue.");
}
```

---

# 3. Comment fonctionne `catch` ?

On peut récupérer l'exception :

```csharp
try
{
    int result = 10 / 0;
}
catch (Exception ex)
{
    Console.WriteLine(ex.Message);
}
```

`ex` contient l'objet représentant l'exception.

On peut notamment avoir :

```csharp
ex.Message
ex.StackTrace
ex.InnerException
```

### Attention

Ne pas considérer `Exception` comme une simple chaîne de texte.

C'est un objet qui contient des informations sur l'erreur.

---

# 4. La hiérarchie des exceptions

Toutes les exceptions dérivent finalement de :

```csharp
System.Exception
```

Exemple conceptuel :

```text
Exception
   │
   ├── SystemException
   │      ├── ArgumentException
   │      │      └── ArgumentNullException
   │      ├── InvalidOperationException
   │      └── ...
   │
   └── Exceptions personnalisées
```

Cela permet d'attraper une catégorie d'erreurs.

Par exemple :

```csharp
catch (ArgumentException ex)
{
}
```

ou plus largement :

```csharp
catch (Exception ex)
{
}
```

---

# 5. L'ordre des `catch`

L'ordre est important.

Correct :

```csharp
try
{
}
catch (ArgumentNullException ex)
{
}
catch (ArgumentException ex)
{
}
catch (Exception ex)
{
}
```

Pourquoi ?

Parce que :

```text
ArgumentNullException
        ↓
ArgumentException
        ↓
Exception
```

Le type le plus spécifique doit être traité avant le type général.

Sinon le `catch (Exception)` intercepterait déjà l'exception.

---

# 6. `throw`

`throw` permet de lever une exception.

```csharp
if (age < 0)
{
    throw new ArgumentException("L'âge ne peut pas être négatif.");
}
```

Le programme indique :

> Cette situation n'est pas valide et doit être signalée comme une exception.

---

# 7. `throw` dans un `catch`

On peut intercepter une exception puis la relancer :

```csharp
try
{
    DoSomething();
}
catch (Exception ex)
{
    Log(ex);
    throw;
}
```

Le point important est :

```csharp
throw;
```

et non :

```csharp
throw ex;
```

---

# 8. `throw;` vs `throw ex;`

## `throw;`

```csharp
catch (Exception ex)
{
    Log(ex);
    throw;
}
```

Relance l'exception en conservant sa stack trace originale.

## `throw ex;`

```csharp
catch (Exception ex)
{
    Log(ex);
    throw ex;
}
```

Cette forme peut réinitialiser la stack trace à cet endroit.

### Règle mentale

Quand tu veux simplement relancer l'exception :

```csharp
throw;
```

---

# 9. Propagation d'une exception

Supposons :

```csharp
void A()
{
    B();
}

void B()
{
    C();
}

void C()
{
    throw new Exception("Erreur");
}
```

L'exception est créée dans `C`.

Elle remonte :

```text
C()
 ↓
B()
 ↓
A()
 ↓
appelant
```

.NET cherche un `catch` compatible.

Si aucun gestionnaire n'est trouvé, l'exception finit par atteindre le niveau supérieur de l'application.

### Mental model

```text
throw
  ↓
remonte la call stack
  ↓
cherche un catch compatible
```

---

# 10. Pourquoi les exceptions remontent-elles ?

Imagine :

```csharp
public void ControllerAction()
{
    ServiceMethod();
}
```

Puis :

```csharp
public void ServiceMethod()
{
    RepositoryMethod();
}
```

Et :

```csharp
public void RepositoryMethod()
{
    throw new Exception();
}
```

Le repository n'est pas nécessairement le meilleur endroit pour décider comment présenter l'erreur à l'utilisateur.

L'exception peut donc remonter vers une couche supérieure.

Dans une Web API, une gestion globale peut alors transformer l'erreur en réponse HTTP adaptée.

---

# 11. `finally`

`finally` est exécuté lorsqu'on quitte le bloc `try/catch`, y compris lorsqu'une exception se produit.

```csharp
try
{
    DoSomething();
}
catch (Exception ex)
{
    Log(ex);
}
finally
{
    Cleanup();
}
```

Il est historiquement utilisé pour garantir du nettoyage.

Cependant, pour les ressources `IDisposable`, on préfère généralement utiliser :

```csharp
using
```

ou :

```csharp
await using
```

---

# 12. `using` et `IDisposable`

Exemple :

```csharp
using (var connection = new SqlConnection(connectionString))
{
    connection.Open();
}
```

Conceptuellement, `using` garantit que la ressource sera libérée.

Cela correspond à l'idée :

```text
Créer ressource
      ↓
Utiliser
      ↓
Dispose()
```

C'est souvent préférable à écrire manuellement :

```csharp
try
{
}
finally
{
    resource.Dispose();
}
```

---

# 13. Exceptions courantes

Quelques exceptions que tu rencontreras souvent en C# :

### `ArgumentException`

Un argument fourni à une méthode n'est pas valide.

```csharp
throw new ArgumentException("Valeur invalide.");
```

### `ArgumentNullException`

Un argument obligatoire est `null`.

```csharp
throw new ArgumentNullException(nameof(user));
```

### `ArgumentOutOfRangeException`

Une valeur est en dehors de la plage acceptée.

```csharp
throw new ArgumentOutOfRangeException(nameof(age));
```

### `InvalidOperationException`

L'opération n'est pas valide dans l'état actuel de l'objet.

Exemple conceptuel :

```csharp
var item = collection.First();
```

alors que la collection est vide.

### `NullReferenceException`

On tente généralement d'accéder à un membre sur une référence `null`.

```csharp
User? user = null;

Console.WriteLine(user.Name);
```

---

# 14. Ne pas utiliser une exception pour un flux normal

Une erreur fréquente consiste à utiliser les exceptions pour des situations normales.

Mauvais réflexe :

```csharp
try
{
    var user = users.Single(u => u.Id == id);
}
catch
{
    // L'utilisateur n'existe pas
}
```

Si l'absence d'utilisateur est une situation normale, il peut être plus clair d'utiliser :

```csharp
var user = users.SingleOrDefault(u => u.Id == id);
```

Puis :

```csharp
if (user is null)
{
    // Pas trouvé
}
```

### Règle

```text
Situation normale attendue
→ résultat explicite

Situation exceptionnelle
→ exception
```

---

# 15. Ne pas faire `catch (Exception)` partout

Exemple problématique :

```csharp
try
{
    DoSomething();
}
catch (Exception)
{
}
```

C'est un **swallowing exception** : l'exception est avalée.

Le programme continue sans signaler le problème.

Cela peut rendre les bugs extrêmement difficiles à diagnostiquer.

---

# 16. Le mauvais `catch`

Autre problème :

```csharp
catch (Exception ex)
{
    Console.WriteLine("Erreur");
}
```

dans toutes les méthodes de l'application.

On finit avec :

```text
Repository → catch
Service    → catch
Controller → catch
Middleware → catch
```

La même erreur peut être capturée plusieurs fois sans apporter de valeur.

Il est généralement préférable de déterminer **quelle couche possède la responsabilité de gérer l'erreur**.

---

# 17. Exceptions et architecture .NET

Dans une architecture en couches :

```text
Controller
    ↓
Service
    ↓
Repository
    ↓
Database
```

Une exception peut remonter :

```text
Database
   ↓
Repository
   ↓
Service
   ↓
Controller / Middleware
```

La couche qui possède suffisamment de contexte peut décider quoi faire.

Dans une Web API, une gestion globale des exceptions permet souvent de centraliser la transformation :

```text
Exception
    ↓
Middleware global
    ↓
HTTP response
```

---

# 18. Exemple avec une Web API

Supposons un service :

```csharp
public async Task<User> GetUserAsync(int id)
{
    var user = await _repository.GetByIdAsync(id);

    if (user is null)
    {
        throw new KeyNotFoundException(
            $"User {id} not found.");
    }

    return user;
}
```

Le service signale :

```text
Utilisateur demandé mais inexistant
```

Une couche supérieure peut ensuite transformer cette situation en :

```http
404 Not Found
```

L'idée importante est de séparer :

```text
détection de la situation
        ≠
présentation HTTP de la situation
```

---

# 19. Exceptions personnalisées

On peut créer sa propre exception :

```csharp
public class UserNotFoundException : Exception
{
    public UserNotFoundException(string message)
        : base(message)
    {
    }
}
```

Puis :

```csharp
throw new UserNotFoundException(
    $"User {id} not found.");
```

Cela permet de distinguer une erreur métier particulière.

---

# 20. Quand créer une exception personnalisée ?

Une exception personnalisée peut être utile lorsque le type de situation a une vraie signification dans le domaine.

Exemple :

```text
OrderNotFoundException
InsufficientStockException
UserNotFoundException
```

Cela permet ensuite de traiter spécifiquement l'erreur.

Mais il ne faut pas créer une nouvelle classe d'exception pour chaque petit problème.

---

# 21. Exception et résultat métier

Toutes les situations métier ne nécessitent pas obligatoirement une exception.

Par exemple :

```text
Commande créée
```

ou :

```text
Stock insuffisant
```

peuvent, selon l'architecture choisie, être représentés par un objet de résultat :

```csharp
Result<T>
```

ou par une exception métier.

Le choix dépend du contexte et de l'architecture.

### Idée importante

Une exception est un mécanisme de contrôle d'erreur exceptionnel, pas automatiquement le mécanisme pour tous les résultats négatifs.

---

# 22. `InnerException`

Une exception peut contenir une autre exception :

```csharp
try
{
    DoSomething();
}
catch (Exception ex)
{
    Console.WriteLine(ex.InnerException);
}
```

Conceptuellement :

```text
Exception principale
       ↓
InnerException
       ↓
Cause originale
```

C'est particulièrement utile lorsque plusieurs couches encapsulent une erreur.

---

# 23. Wrapping d'une exception

On peut créer une nouvelle exception tout en conservant la cause :

```csharp
try
{
    repository.Save();
}
catch (Exception ex)
{
    throw new UserPersistenceException(
        "Impossible de sauvegarder l'utilisateur.",
        ex);
}
```

La cause originale est alors accessible via :

```csharp
ex.InnerException
```

C'est différent de simplement remplacer l'exception sans conserver la cause.

---

# 24. Exceptions asynchrones

Avec `async/await` :

```csharp
public async Task<User> GetUserAsync()
{
    return await repository.GetUserAsync();
}
```

Si l'opération asynchrone produit une exception, elle peut être propagée au code appelant.

On peut donc écrire :

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

### Règle importante

Éviter de transformer inutilement :

```csharp
await GetUserAsync();
```

en :

```csharp
GetUserAsync().Wait();
```

ou :

```csharp
GetUserAsync().Result;
```

Cela peut provoquer des problèmes de blocage et casse généralement le modèle asynchrone.

---

# 25. `finally` avec `return`

Il faut connaître une règle importante :

```csharp
try
{
    return 10;
}
finally
{
    Console.WriteLine("finally");
}
```

Le `finally` est exécuté avant que la méthode ne termine réellement son retour.

Il ne faut cependant pas mettre un `return` dans `finally`.

Exemple à éviter :

```csharp
try
{
    return 10;
}
finally
{
    return 20;
}
```

Cela rend le comportement difficile à comprendre et peut masquer une exception.

---

# 26. `catch` avec filtre

C# permet de filtrer un `catch` :

```csharp
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound)
{
    // Traitement spécifique
}
```

Le `when` permet de préciser :

> Je veux ce `catch` uniquement si cette condition est vraie.

C'est utile pour éviter des blocs `catch` trop généraux.

---

# 27. Exceptions et logging

Lorsqu'une exception inattendue se produit, elle doit généralement être journalisée avec suffisamment de contexte.

Exemple :

```csharp
catch (Exception ex)
{
    _logger.LogError(
        ex,
        "Error while loading user {UserId}",
        userId);

    throw;
}
```

Point important :

```csharp
_logger.LogError(ex, ...);
```

permet au système de logging de conserver les informations de l'exception.

Il ne faut pas seulement faire :

```csharp
_logger.LogError(ex.Message);
```

car cela perd des informations importantes comme la stack trace.

---

# 28. Ne jamais exposer directement les détails internes

Dans une API, il faut éviter de retourner :

```csharp
return BadRequest(ex.ToString());
```

à un utilisateur.

Une exception peut contenir :

- noms de classes ;
- chemins ;
- requêtes SQL ;
- détails techniques ;
- informations internes.

En production, l'API doit généralement retourner une réponse contrôlée.

Conceptuellement :

```text
Exception interne
      ↓
Logging détaillé
      ↓
Réponse API contrôlée
```

---

# 29. Gestion globale des exceptions

Dans ASP.NET Core, on peut centraliser la gestion des exceptions avec un middleware ou le mécanisme global approprié.

Conceptuellement :

```text
Request
   ↓
Middleware
   ↓
Controller
   ↓
Service
   ↓
Exception
   ↑
Middleware
   ↓
HTTP response
```

Cela évite de répéter :

```csharp
try/catch
```

dans chaque contrôleur.

---

# 30. Exemple de middleware conceptuel

```csharp
public async Task InvokeAsync(HttpContext context)
{
    try
    {
        await _next(context);
    }
    catch (Exception ex)
    {
        _logger.LogError(ex, "Unhandled exception");

        context.Response.StatusCode = 500;
    }
}
```

Le middleware laisse l'application travailler normalement.

Si une exception non gérée remonte :

```text
Controller
    ↓
Service
    ↓
Exception
    ↓
Middleware
    ↓
500
```

---

# 31. Exceptions et HTTP

Il ne faut pas confondre :

```text
Exception C#
```

et :

```text
HTTP status code
```

Ce sont deux concepts différents.

Exemple :

```text
UserNotFoundException
        ↓
404 Not Found
```

L'exception appartient au monde .NET.

Le `404` appartient au protocole HTTP.

Une couche de l'application peut faire le mapping entre les deux.

---

# 32. `ProblemDetails`

Dans une Web API ASP.NET Core, `ProblemDetails` est couramment utilisé pour représenter proprement les erreurs HTTP.

Conceptuellement :

```json
{
  "status": 404,
  "title": "User not found",
  "detail": "The requested user does not exist."
}
```

Cela permet de ne pas exposer directement l'exception interne au client.

---

# 33. Exceptions et validation

Toutes les erreurs de validation ne doivent pas être transformées en exceptions.

Exemple :

```text
Email obligatoire
Password trop court
Name obligatoire
```

Ce sont souvent des erreurs de validation attendues.

ASP.NET Core peut les représenter directement dans la réponse HTTP.

### Mental model

```text
Données invalides
→ validation

Erreur inattendue / situation exceptionnelle
→ exception
```

---

# 34. Performance

Les exceptions sont conçues pour gérer des situations exceptionnelles.

Il ne faut pas construire une logique normale comme :

```csharp
for (...)
{
    try
    {
        ...
    }
    catch
    {
        ...
    }
}
```

si l'exception est utilisée comme mécanisme normal de contrôle du flux.

Préférer une logique explicite lorsque la situation est prévisible.

---

# 35. Exemple : mauvais vs meilleur

### Mauvais

```csharp
try
{
    var user = users.Single(u => u.Id == id);
}
catch (InvalidOperationException)
{
    return null;
}
```

### Plus explicite

```csharp
var user = users.SingleOrDefault(u => u.Id == id);

if (user is null)
{
    return null;
}
```

Le deuxième code exprime directement que l'absence est une situation possible.

---

# 36. Règle de propagation

Une règle très utile :

> Une méthode doit généralement gérer une exception lorsqu'elle sait réellement quoi en faire.

Exemple :

```text
Repository
→ détecte un problème technique

Service
→ connaît éventuellement le contexte métier

Middleware / API
→ sait comment produire une réponse HTTP
```

Il ne faut donc pas automatiquement mettre un `catch` partout.

---

# 37. Mental model complet

Quand une exception apparaît, pense :

```text
1. Où l'exception est-elle créée ?
        ↓
2. Qui la `throw` ?
        ↓
3. Quelle est la call stack ?
        ↓
4. Où est le premier catch compatible ?
        ↓
5. Cette couche sait-elle quoi en faire ?
        ↓
6. Faut-il logger ?
        ↓
7. Faut-il relancer ?
        ↓
8. Comment l'erreur devient-elle une réponse API ?
```

---

# 38. À retenir

1. Une exception représente une situation anormale ou exceptionnelle.
2. `throw` lève une exception.
3. `catch` permet de la gérer.
4. `finally` sert notamment au nettoyage garanti.
5. `throw;` conserve la stack trace lors d'une relance.
6. Une exception remonte la call stack jusqu'à trouver un `catch` compatible.
7. Ne pas utiliser les exceptions pour tous les flux normaux.
8. Éviter les `catch (Exception)` qui avalent les erreurs.
9. Une exception personnalisée peut représenter une vraie situation métier.
10. `InnerException` permet de conserver une cause interne.
11. `using` simplifie la gestion des ressources `IDisposable`.
12. Avec `async/await`, laisser les exceptions se propager naturellement lorsque c'est approprié.
13. Dans une Web API, séparer l'exception .NET de la réponse HTTP.
14. Une gestion globale évite de répéter des `try/catch` dans tous les contrôleurs.
15. Logger l'exception complète, pas uniquement `ex.Message`.
16. Ne jamais exposer les détails techniques d'une exception au client en production.

---

# 39. Questions d'entretien

### Que fait `throw` ?

Il lève une exception.

---

### Quelle est la différence entre `throw` et `throw ex` ?

`throw;` relance l'exception en conservant sa stack trace. `throw ex;` peut modifier la stack trace et doit généralement être évité pour une simple relance.

---

### Qu'est-ce que la propagation d'une exception ?

Une exception non gérée remonte la pile d'appels jusqu'à trouver un `catch` compatible.

---

### Pourquoi éviter `catch (Exception)` partout ?

Parce que cela peut masquer les erreurs, dupliquer la gestion et rendre le diagnostic plus difficile.

---

### Quand créer une exception personnalisée ?

Lorsqu'une situation possède une signification suffisamment spécifique pour être traitée différemment.

---

### Exception ou validation ?

Une validation concerne généralement une erreur attendue dans les données fournies. Une exception représente plutôt une situation exceptionnelle ou inattendue.

---

### Comment gérer les exceptions dans une Web API ?

Une approche courante consiste à laisser les exceptions remonter puis à les traiter dans une couche globale, comme un middleware, qui journalise l'erreur et produit une réponse HTTP contrôlée.

---

# 40. La phrase à mémoriser

> **Une exception remonte la call stack jusqu'à une couche capable de décider quoi en faire ; on ne la capture pas simplement pour la faire disparaître.**
