# Middleware en ASP.NET Core

## 1. Définition

Un **Middleware** est un composant du pipeline HTTP ASP.NET Core.

Il peut :

- recevoir une requête HTTP ;
- exécuter du code avant l'étape suivante ;
- appeler l'étape suivante ;
- exécuter du code après cette étape ;
- modifier la requête ;
- modifier la réponse ;
- arrêter le pipeline.

Mentalement :

```text
HTTP Request
     |
     v
Middleware A
     |
     v
Middleware B
     |
     v
Middleware C
     |
     v
Endpoint
     |
     v
HTTP Response
```

Le pipeline est donc une chaîne de composants.

---

# 2. Le principe fondamental

Un Middleware ressemble conceptuellement à ceci :

```csharp
public async Task InvokeAsync(HttpContext context)
{
    // Avant

    await _next(context);

    // Après
}
```

Le point essentiel est :

```csharp
await _next(context);
```

`_next` représente le composant suivant dans le pipeline.

Mentalement :

```text
Middleware
    |
    | code avant
    v
_next(context)
    |
    v
Middleware suivant
    |
    v
Endpoint
    |
    v
retour
    |
    v
code après
```

---

# 3. Le pipeline en oignon

Une excellente façon de visualiser les Middleware est de les imaginer comme des couches.

```text
             Middleware A
        +----------------------+
        |    Middleware B      |
        |   +--------------+   |
        |   | Middleware C |   |
        |   |   Endpoint   |   |
        |   +--------------+   |
        +----------------------+
```

La requête traverse les couches dans un sens :

```text
A -> B -> C -> Endpoint
```

La réponse remonte dans l'autre :

```text
Endpoint -> C -> B -> A
```

C'est pourquoi un Middleware peut faire :

```csharp
// avant
await _next(context);
// après
```

---

# 4. `HttpContext`

Le Middleware reçoit généralement :

```csharp
HttpContext
```

C'est l'objet qui représente le contexte HTTP courant.

Il donne notamment accès à :

```csharp
context.Request
context.Response
context.User
context.Items
context.RequestServices
context.RequestAborted
```

Exemple :

```csharp
var path = context.Request.Path;

var method = context.Request.Method;
```

---

# 5. Lire la requête

On peut accéder à :

```csharp
context.Request.Path
context.Request.Method
context.Request.Query
context.Request.Headers
context.Request.RouteValues
```

Exemple :

```csharp
public async Task InvokeAsync(HttpContext context)
{
    var path = context.Request.Path;

    Console.WriteLine(
        $"Request : {context.Request.Method} {path}");

    await _next(context);
}
```

---

# 6. Modifier la réponse

Un Middleware peut également modifier la réponse.

Exemple :

```csharp
public async Task InvokeAsync(HttpContext context)
{
    context.Response.Headers["X-App-Version"] = "1.0";

    await _next(context);
}
```

Toutes les réponses qui passent par ce Middleware peuvent recevoir ce header.

---

# 7. Arrêter le pipeline

Le Middleware n'est pas obligé d'appeler :

```csharp
await _next(context);
```

Exemple :

```csharp
public async Task InvokeAsync(HttpContext context)
{
    if (!IsAllowed(context))
    {
        context.Response.StatusCode = 403;
        return;
    }

    await _next(context);
}
```

Ici :

```text
Condition invalide
      |
      v
403
      |
      X
Pas de Middleware suivant
```

C'est ce qu'on appelle souvent **court-circuiter le pipeline**.

---

# 8. `RequestDelegate`

Le type :

```csharp
RequestDelegate
```

représente une fonction capable de traiter un :

```csharp
HttpContext
```

Conceptuellement :

```csharp
delegate Task RequestDelegate(HttpContext context);
```

Donc :

```csharp
_next(context)
```

signifie essentiellement :

> « Exécute le composant suivant avec le contexte HTTP actuel. »

---

# 9. Middleware avec classe

Exemple classique :

```csharp
public class RequestLoggingMiddleware
{
    private readonly RequestDelegate _next;

    public RequestLoggingMiddleware(RequestDelegate next)
    {
        _next = next;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        Console.WriteLine(
            $"Début : {context.Request.Path}");

        await _next(context);

        Console.WriteLine(
            $"Fin : {context.Response.StatusCode}");
    }
}
```

Enregistrement :

```csharp
app.UseMiddleware<RequestLoggingMiddleware>();
```

---

# 10. Version avec Primary Constructor

En C# moderne, on peut écrire :

```csharp
public class RequestLoggingMiddleware(
    RequestDelegate next,
    ILogger<RequestLoggingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        logger.LogInformation(
            "Request {Path}",
            context.Request.Path);

        await next(context);
    }
}
```

Le principe reste exactement le même :

```text
RequestDelegate
    |
    v
étape suivante
```

---

# 11. `app.Use`

`Use` permet d'ajouter un Middleware dans le pipeline.

Exemple :

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Avant");

    await next();

    Console.WriteLine("Après");
});
```

Le pipeline continue avec :

```csharp
await next();
```

---

# 12. `app.Run`

`Run` ajoute un Middleware terminal.

Exemple :

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello");
});
```

Il n'a pas de `next`.

Mentalement :

```text
A
 |
 v
B
 |
 v
Run
 |
 X
```

Le `Run` termine normalement le pipeline.

---

# 13. `app.Map`

`Map` permet de créer une branche du pipeline selon un chemin ou une condition.

Exemple :

```csharp
app.Map("/admin", adminApp =>
{
    adminApp.Run(async context =>
    {
        await context.Response.WriteAsync("Admin");
    });
});
```

Une requête :

```text
/admin
```

peut donc emprunter cette branche.

Mentalement :

```text
              +--> /admin -> Pipeline Admin
Request ------|
              +--> reste -> Pipeline principal
```

---

# 14. `Use` vs `Run` vs `Map`

| Méthode | Rôle |
|---|---|
| `Use` | Ajoute un Middleware pouvant continuer |
| `Run` | Ajoute un Middleware terminal |
| `Map` | Crée une branche du pipeline |

À retenir :

```text
Use -> continue
Run -> stop
Map -> branche
```

---

# 15. L'ordre des Middleware est crucial

Considérons :

```csharp
app.UseMiddleware<A>();
app.UseMiddleware<B>();
app.UseMiddleware<C>();
```

La requête passe :

```text
A -> B -> C
```

Mais après `await _next(context)` :

```text
C -> B -> A
```

On peut représenter :

```text
A avant
  |
  v
B avant
  |
  v
C avant
  |
  v
Endpoint
  |
  v
C après
  |
  v
B après
  |
  v
A après
```

L'ordre change donc le comportement de l'application.

---

# 16. Exemple avec logging

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("A avant");

    await next();

    Console.WriteLine("A après");
});

app.Use(async (context, next) =>
{
    Console.WriteLine("B avant");

    await next();

    Console.WriteLine("B après");
});
```

Résultat :

```text
A avant
B avant
...
B après
A après
```

Pourquoi ?

Parce que chaque Middleware reprend son exécution après le `await next()` lorsque le composant suivant a terminé.

---

# 17. Pourquoi l'ordre est important ?

Imagine :

```text
Exception Handling
Authentication
Authorization
Routing
Endpoint
```

Si tu places certains composants dans un ordre incorrect, le comportement peut devenir incorrect.

Exemple conceptuel :

```text
Authorization
    |
    v
Authentication
```

L'autorisation a besoin de connaître l'identité de l'utilisateur.

Donc l'ordre doit permettre à l'authentification de produire l'utilisateur avant que l'autorisation vérifie ses droits.

---

# 18. Routing et Middleware

Dans une API Controller moderne, on retrouve généralement :

```csharp
app.UseRouting();

app.UseAuthentication();

app.UseAuthorization();

app.MapControllers();
```

Mentalement :

```text
Request
   |
   v
Routing
   |
   v
Authentication
   |
   v
Authorization
   |
   v
Controller endpoint
```

La configuration exacte peut varier selon l'application et les versions de .NET, mais le principe reste le même :

> Les composants doivent être placés dans un ordre cohérent avec leurs dépendances.

---

# 19. Middleware d'exception

Un cas très important est la gestion globale des exceptions.

Exemple :

```csharp
app.Use(async (context, next) =>
{
    try
    {
        await next();
    }
    catch (Exception ex)
    {
        // Log
        // Transformer en réponse HTTP
    }
});
```

Le Middleware se place autour du reste du pipeline :

```text
Exception Middleware
       |
       v
   Application
       |
       v
   Exception
       |
       v
Exception Middleware
       |
       v
HTTP 500
```

Cela permet d'avoir un point central de traitement.

---

# 20. Pourquoi éviter les `try/catch` partout ?

Mauvaise approche :

```csharp
public IActionResult Get()
{
    try
    {
        // ...
    }
    catch (Exception ex)
    {
        return StatusCode(500);
    }
}
```

répétée dans chaque action.

Cela produit :

```text
Controller A -> try/catch
Controller B -> try/catch
Controller C -> try/catch
```

Un Middleware global permet de centraliser ce comportement.

```text
Toutes les requêtes
        |
        v
Exception Middleware
```

---

# 21. Middleware et logging

Un Middleware est également adapté à certains logs HTTP.

Exemple :

```csharp
public async Task InvokeAsync(HttpContext context)
{
    var stopwatch = Stopwatch.StartNew();

    await _next(context);

    stopwatch.Stop();

    _logger.LogInformation(
        "{Method} {Path} -> {StatusCode} in {Elapsed} ms",
        context.Request.Method,
        context.Request.Path,
        context.Response.StatusCode,
        stopwatch.ElapsedMilliseconds);
}
```

On peut ainsi observer :

```text
GET /api/hotels -> 200 -> 35 ms
```

---

# 22. Middleware et Correlation ID

Un Middleware est un bon endroit pour gérer un identifiant de corrélation.

Exemple :

```csharp
public async Task InvokeAsync(HttpContext context)
{
    var correlationId =
        context.Request.Headers["X-Correlation-Id"]
        .FirstOrDefault()
        ?? Guid.NewGuid().ToString();

    context.Response.Headers["X-Correlation-Id"]
        = correlationId;

    await _next(context);
}
```

L'identifiant peut ensuite être utilisé pour relier plusieurs logs appartenant à la même requête.

Mentalement :

```text
Request
   |
   v
Correlation ID
   |
   +--> Service
   +--> Repository
   +--> Logs
```

---

# 23. `HttpContext.Items`

`HttpContext.Items` permet de stocker des données pour la durée de la requête.

Exemple :

```csharp
context.Items["CorrelationId"] = correlationId;
```

Puis plus loin :

```csharp
var correlationId =
    context.Items["CorrelationId"];
```

C'est une donnée temporaire liée au `HttpContext`.

Il faut éviter d'en faire un mécanisme général de communication entre toutes les couches de l'application.

---

# 24. `RequestAborted`

ASP.NET Core fournit un token :

```csharp
context.RequestAborted
```

Il représente l'annulation associée à la requête HTTP.

On peut le transmettre :

```csharp
await service.GetHotelsAsync(
    context.RequestAborted);
```

Puis :

```csharp
await repository.GetHotelsAsync(
    cancellationToken);
```

Le flux devient :

```text
Client
  |
  | annule la requête
  v
RequestAborted
  |
  v
Service
  |
  v
EF Core
```

Cela permet d'éviter de continuer inutilement un travail dont le client n'a plus besoin.

---

# 25. Middleware et Dependency Injection

Un Middleware peut utiliser des dépendances.

Exemple :

```csharp
public class LoggingMiddleware(
    RequestDelegate next,
    ILogger<LoggingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        logger.LogInformation(
            "Request received");

        await next(context);
    }
}
```

Le Middleware peut donc participer au système DI.

---

# 26. Attention aux lifetimes dans les Middleware

Les Middleware conventionnels sont généralement créés et réutilisés selon le fonctionnement du pipeline.

Il faut donc être prudent lorsqu'on injecte directement des dépendances ayant une durée de vie `Scoped`.

Par exemple, une dépendance `DbContext` ne doit pas être capturée de manière incorrecte dans une instance Middleware réutilisée.

Une approche courante consiste à obtenir les dépendances scoped dans le contexte d'exécution approprié.

Exemple :

```csharp
public async Task InvokeAsync(
    HttpContext context,
    IMyScopedService service)
{
    await service.DoSomethingAsync();

    await _next(context);
}
```

Le framework peut résoudre le paramètre `service` au moment de l'appel de `InvokeAsync`.

Règle importante :

> Comprendre le lifetime de la dépendance avant de l'injecter dans un composant long-lived.

---

# 27. Middleware vs Filter

C'est une distinction fondamentale.

### Middleware

```text
Pipeline HTTP
```

Il peut traiter :

```text
toute requête
```

### Filter

```text
Pipeline MVC
```

Il connaît davantage :

```text
Controller
Action
ActionArguments
ModelState
```

Règle mentale :

```text
Middleware = HTTP

Filter = MVC
```

---

# 28. Middleware vs Controller

Le Controller représente un endpoint ou un ensemble d'actions.

Exemple :

```csharp
[HttpGet]
public async Task<IActionResult> GetHotels()
{
    var hotels = await _service.GetAllAsync();

    return Ok(hotels);
}
```

Le Middleware ne doit pas devenir un Controller géant.

Le Middleware sert principalement aux comportements transversaux.

---

# 29. Middleware vs Service

Le Service contient généralement la logique applicative.

Exemple :

```csharp
public async Task<Hotel> CreateAsync(
    CreateHotelDto dto)
{
    // logique applicative
}
```

Le Middleware ne devrait pas contenir :

```text
règles métier
calculs métier
accès métier complexe aux données
```

Il doit rester centré sur le pipeline HTTP.

---

# 30. Exemple complet

Middleware :

```csharp
public class RequestTimingMiddleware(
    RequestDelegate next,
    ILogger<RequestTimingMiddleware> logger)
{
    public async Task InvokeAsync(HttpContext context)
    {
        var stopwatch = Stopwatch.StartNew();

        try
        {
            await next(context);
        }
        finally
        {
            stopwatch.Stop();

            logger.LogInformation(
                "{Method} {Path} -> {StatusCode} in {Elapsed} ms",
                context.Request.Method,
                context.Request.Path,
                context.Response.StatusCode,
                stopwatch.ElapsedMilliseconds);
        }
    }
}
```

Enregistrement :

```csharp
app.UseMiddleware<RequestTimingMiddleware>();
```

Flux :

```text
HTTP Request
     |
     v
RequestTimingMiddleware
     |
     | Start
     v
Middleware suivants
     |
     v
Controller
     |
     v
Service
     |
     v
Response
     |
     v
RequestTimingMiddleware
     |
     | Stop + Log
     v
Client
```

---

# 31. `finally` dans un Middleware

Pourquoi utiliser :

```csharp
finally
```

pour certaines opérations après le pipeline ?

Parce qu'on veut souvent exécuter le code même lorsqu'une exception se produit.

Exemple :

```csharp
try
{
    await _next(context);
}
finally
{
    stopwatch.Stop();
}
```

Le chronomètre sera arrêté même si l'étape suivante lève une exception.

---

# 32. Court-circuiter : cas pratique

Exemple :

```csharp
app.Use(async (context, next) =>
{
    if (!context.Request.Headers.ContainsKey("X-Api-Key"))
    {
        context.Response.StatusCode = 401;
        return;
    }

    await next();
});
```

Flux :

```text
Request
   |
   v
API Key Middleware
   |
   +---- absente ---> 401
   |
   +---- présente --> next()
                         |
                         v
                    Application
```

Le Middleware joue donc le rôle de garde devant la suite du pipeline.

Attention : dans une vraie API, il vaut mieux utiliser les mécanismes d'authentification/autorisation adaptés plutôt que recréer soi-même un système de sécurité fragile.

---

# 33. `UseWhen`

ASP.NET Core permet également de conditionner l'utilisation d'une branche de Middleware.

Conceptuellement :

```csharp
app.UseWhen(
    context => context.Request.Path.StartsWithSegments("/admin"),
    branch =>
    {
        branch.UseMiddleware<AdminMiddleware>();
    });
```

Mentalement :

```text
Request
   |
   +---- /admin ---> Middleware Admin
   |
   +---- autre ----> Pipeline normal
```

C'est utile lorsque le comportement doit dépendre d'une condition.

---

# 34. `MapWhen`

`MapWhen` permet également de créer une branche basée sur une condition.

La différence principale à retenir est que `Map` travaille typiquement avec un chemin tandis que `MapWhen` permet une condition plus générale.

Exemple conceptuel :

```csharp
app.MapWhen(
    context => context.Request.Method == "POST",
    branch =>
    {
        branch.UseMiddleware<PostMiddleware>();
    });
```

---

# 35. Middleware et performance

Un Middleware est exécuté sur les requêtes qui traversent sa position dans le pipeline.

Il faut donc éviter d'y faire inutilement :

```text
requêtes DB lourdes
calculs coûteux
allocations inutiles
logs excessifs
```

Un comportement transversal est utile, mais il doit rester efficace.

Exemple :

```text
Logging HTTP
```

est raisonnable.

Mais faire une requête SQL complexe dans chaque Middleware pour chaque requête peut devenir très coûteux.

---

# 36. Middleware et sécurité

Les Middleware jouent un rôle important dans la sécurité :

```text
HTTPS
Authentication
Authorization
Headers
Rate Limiting
Exception handling
```

Mais chaque mécanisme doit utiliser l'abstraction adaptée du framework.

Il faut éviter de créer manuellement des mécanismes de sécurité lorsque ASP.NET Core fournit déjà une solution robuste.

---

# 37. Erreurs fréquentes

## Erreur 1 : oublier `_next`

```csharp
public async Task InvokeAsync(HttpContext context)
{
    Console.WriteLine("Hello");
}
```

Si ce Middleware est censé laisser passer la requête mais n'appelle jamais :

```csharp
await _next(context);
```

il peut bloquer la suite du pipeline.

---

## Erreur 2 : mauvais ordre

Deux Middleware identiques peuvent produire des résultats différents selon leur ordre.

```csharp
app.UseA();
app.UseB();
```

n'est pas nécessairement équivalent à :

```csharp
app.UseB();
app.UseA();
```

---

## Erreur 3 : mettre la logique métier dans Middleware

Éviter :

```text
Middleware
 -> calcule le prix
 -> vérifie le stock
 -> crée une commande
```

Cette logique appartient à la couche applicative/métier.

---

## Erreur 4 : capturer une dépendance Scoped incorrectement

Un Middleware réutilisé ne doit pas conserver une instance scoped de manière incorrecte.

Il faut respecter les lifetimes DI.

---

## Erreur 5 : exposer les exceptions internes

Éviter de retourner :

```csharp
ex.StackTrace
```

ou :

```csharp
ex.ToString()
```

directement au client en production.

Logger côté serveur et retourner une réponse contrôlée.

---

# 38. Schéma global ASP.NET Core

```text
                         HTTP REQUEST
                              |
                              v
                    +-------------------+
                    |    Middleware 1   |
                    +-------------------+
                              |
                              v
                    +-------------------+
                    |    Middleware 2   |
                    +-------------------+
                              |
                              v
                         Routing
                              |
                              v
                      Authentication
                              |
                              v
                       Authorization
                              |
                              v
                    +-------------------+
                    |       MVC         |
                    +-------------------+
                              |
                              v
                           Filters
                              |
                              v
                       Model Binding
                              |
                              v
                         Validation
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

Le retour traverse les couches dans le sens inverse pour les Middleware qui attendent `next`.

---

# 39. Règle mentale

Quand tu vois :

```csharp
await _next(context);
```

pense :

> « Je donne la requête au composant suivant, puis mon Middleware reprend lorsque ce composant a terminé. »

Quand tu vois :

```csharp
app.Use(...)
```

pense :

> « J'ajoute une couche dans le pipeline. »

Quand tu vois :

```csharp
app.Run(...)
```

pense :

> « Je termine le pipeline. »

Quand tu vois :

```csharp
app.Map(...)
```

pense :

> « Je crée une branche du pipeline. »

Quand tu hésites entre Middleware et Filter :

```text
HTTP global -> Middleware
MVC / Action -> Filter
```

---

# 40. À retenir

1. Un Middleware est un composant du pipeline HTTP.
2. `HttpContext` représente le contexte de la requête HTTP.
3. `RequestDelegate` représente l'étape suivante.
4. `_next(context)` permet de continuer le pipeline.
5. Le code avant `_next` s'exécute à l'aller.
6. Le code après `_next` s'exécute au retour.
7. Un Middleware peut court-circuiter le pipeline.
8. `Use` ajoute un Middleware qui peut continuer.
9. `Run` ajoute un Middleware terminal.
10. `Map` crée une branche du pipeline.
11. L'ordre des Middleware est fondamental.
12. Les Middleware sont adaptés aux comportements transversaux HTTP.
13. Les Middleware ne doivent pas contenir toute la logique métier.
14. `RequestAborted` permet de propager l'annulation de la requête.
15. `HttpContext.Items` permet de partager temporairement des données pendant une requête.
16. Les lifetimes DI doivent être respectés dans les Middleware.
17. Middleware et Filter ne travaillent pas au même niveau.
18. Le Middleware est souvent adapté à la gestion globale des exceptions.
19. Le code après `await _next(context)` s'exécute lors du retour du pipeline.
20. Comprendre le pipeline est plus important que mémoriser les méthodes `Use`, `Run` et `Map`.

---

# Questions d'entretien

### 1. Qu'est-ce qu'un Middleware ?

Un composant du pipeline HTTP ASP.NET Core qui peut traiter une requête, appeler le composant suivant et éventuellement traiter la réponse au retour.

### 2. Que représente `RequestDelegate` ?

Une fonction représentant le prochain composant du pipeline :

```csharp
Task(HttpContext)
```

### 3. Pourquoi `await _next(context)` est-il important ?

Il permet de transmettre la requête au composant suivant et de reprendre l'exécution lorsque celui-ci a terminé.

### 4. Que se passe-t-il si un Middleware n'appelle pas `_next` ?

La requête peut être arrêtée à cet endroit : le pipeline est court-circuité.

### 5. Quelle différence entre `Use`, `Run` et `Map` ?

```text
Use -> ajoute un Middleware qui peut continuer
Run -> Middleware terminal
Map -> branche le pipeline
```

### 6. Pourquoi l'ordre des Middleware est-il important ?

Parce que chaque Middleware peut dépendre du travail effectué par ceux qui le précèdent.

### 7. Pourquoi mettre un gestionnaire global d'exceptions dans un Middleware ?

Parce qu'il peut entourer une grande partie du pipeline HTTP et centraliser la transformation des exceptions en réponses HTTP.

### 8. Quelle différence entre Middleware et Filter ?

Le Middleware travaille au niveau du pipeline HTTP ; le Filter travaille dans le pipeline MVC et peut accéder au contexte de l'action.

### 9. À quoi sert `RequestAborted` ?

À détecter et propager l'annulation associée à la requête HTTP.

### 10. Pourquoi ne faut-il pas injecter n'importe quelle dépendance Scoped directement dans un Middleware long-lived ?

Parce qu'une instance du Middleware peut être réutilisée et conserver une dépendance au-delà de sa portée prévue. Il faut respecter les lifetimes du conteneur DI.

---

# Phrase à retenir

> **Un Middleware est une couche du pipeline HTTP : il peut agir avant `_next`, laisser passer la requête avec `_next(context)`, puis agir à nouveau au retour.**
