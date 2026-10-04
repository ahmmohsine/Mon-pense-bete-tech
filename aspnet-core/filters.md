# Filters en ASP.NET Core

## 1. Définition

Les **Filters** d'ASP.NET Core permettent d'exécuter du code autour de l'exécution d'une action de contrôleur.

Ils sont particulièrement utiles pour ajouter des comportements transversaux liés au fonctionnement des actions MVC/API.

Exemples :

```text
Autorisation
Validation supplémentaire
Logging
Gestion de résultats
Action hooks
Exception handling
```

Mentalement :

```text
Request
   |
   v
Middleware
   |
   v
Routing
   |
   v
Controller
   |
   +--> Filter
   |       |
   |       v
   |    Action
   |
   v
Response
```

Le point important est que les Filters interviennent dans le pipeline MVC/controller, alors que les Middleware interviennent plus largement dans le pipeline HTTP.

---

# 2. Pourquoi utiliser un Filter ?

Supposons que plusieurs contrôleurs doivent effectuer la même opération avant leurs actions.

Sans Filter :

```csharp
[HttpGet]
public IActionResult Get()
{
    LogRequest();

    // logique
}

[HttpPost]
public IActionResult Create()
{
    LogRequest();

    // logique
}
```

On répète :

```csharp
LogRequest();
```

Un Filter permet de centraliser ce comportement.

Mentalement :

> « Je veux appliquer automatiquement un comportement à certaines actions ou à certains contrôleurs. »

---

# 3. Les principaux types de Filters

ASP.NET Core possède plusieurs catégories importantes :

```text
Authorization Filters
Resource Filters
Action Filters
Exception Filters
Result Filters
```

Il existe également les **Endpoint Filters**, notamment dans le contexte des Minimal APIs.

---

# 4. Authorization Filters

Les Authorization Filters concernent l'autorisation.

Exemple conceptuel :

```csharp
[Authorize]
public IActionResult GetPrivateData()
{
    return Ok();
}
```

L'idée est :

```text
Utilisateur
    |
    v
Est-il authentifié ?
    |
    v
A-t-il les droits ?
    |
    +---- non ----> accès refusé
    |
    v
Action
```

Dans une API moderne, `[Authorize]` s'appuie sur le système d'authentification/autorisation ASP.NET Core.

---

# 5. Action Filters

Les Action Filters permettent d'exécuter du code avant et/ou après une action.

Exemple :

```csharp
public class LoggingActionFilter : IActionFilter
{
    public void OnActionExecuting(
        ActionExecutingContext context)
    {
        Console.WriteLine("Avant l'action");
    }

    public void OnActionExecuted(
        ActionExecutedContext context)
    {
        Console.WriteLine("Après l'action");
    }
}
```

On obtient conceptuellement :

```text
Request
   |
   v
OnActionExecuting
   |
   v
Controller Action
   |
   v
OnActionExecuted
   |
   v
Response
```

---

# 6. `IAsyncActionFilter`

Pour du code asynchrone :

```csharp
public class LoggingActionFilter : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        Console.WriteLine("Avant");

        var result = await next();

        Console.WriteLine("Après");
    }
}
```

Ici :

```csharp
await next();
```

signifie :

> « Continue l'exécution du pipeline jusqu'à l'action et reviens ensuite ici. »

C'est très proche mentalement de :

```csharp
await _next(context);
```

dans un Middleware.

Mais le contexte n'est pas le même.

---

# 7. Avant et après l'action

Avec :

```csharp
public async Task OnActionExecutionAsync(
    ActionExecutingContext context,
    ActionExecutionDelegate next)
{
    // AVANT
    var result = await next();

    // APRÈS
}
```

le flux est :

```text
Filter
  |
  | avant
  v
Action
  |
  | résultat
  v
Filter
  |
  | après
  v
Response
```

C'est l'un des concepts les plus importants des Action Filters.

---

# 8. Interrompre l'action

Un Filter peut parfois décider de ne pas continuer.

Exemple conceptuel :

```csharp
public void OnActionExecuting(
    ActionExecutingContext context)
{
    if (/* condition */)
    {
        context.Result = new UnauthorizedResult();
    }
}
```

En définissant :

```csharp
context.Result
```

le pipeline peut produire directement le résultat sans exécuter l'action.

Mentalement :

```text
Filter
  |
  +---- condition incorrecte
  |
  v
Result
```

au lieu de :

```text
Filter
  |
  v
Action
  |
  v
Result
```

---

# 9. `ActionExecutingContext`

Le contexte donne accès à des informations sur l'exécution.

On peut notamment accéder à :

```csharp
context.HttpContext
context.ActionArguments
context.Controller
context.ModelState
```

Par exemple :

```csharp
var arguments = context.ActionArguments;
```

permet d'inspecter les paramètres de l'action.

---

# 10. `ActionArguments`

Supposons :

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id)
{
    return Ok();
}
```

Le Filter peut accéder aux arguments via :

```csharp
context.ActionArguments
```

On peut conceptuellement obtenir :

```text
id -> 5
```

Cela peut être utile pour certains comportements transversaux.

---

# 11. Créer un Filter avec une classe

Exemple :

```csharp
public class LoggingActionFilter : IActionFilter
{
    public void OnActionExecuting(
        ActionExecutingContext context)
    {
        Console.WriteLine(
            $"Action : {context.ActionDescriptor.DisplayName}");
    }

    public void OnActionExecuted(
        ActionExecutedContext context)
    {
        Console.WriteLine("Action terminée");
    }
}
```

---

# 12. Appliquer un Filter à une action

On peut utiliser :

```csharp
[ServiceFilter(typeof(LoggingActionFilter))]
public IActionResult Get()
{
    return Ok();
}
```

Le Filter sera appliqué à cette action.

---

# 13. Enregistrer le Filter dans la DI

Si on utilise `ServiceFilter`, il faut enregistrer le Filter :

```csharp
builder.Services.AddScoped<LoggingActionFilter>();
```

Puis :

```csharp
[ServiceFilter(typeof(LoggingActionFilter))]
```

Le Filter est alors obtenu depuis le conteneur DI.

---

# 14. `TypeFilter`

Une autre possibilité est :

```csharp
[TypeFilter(typeof(LoggingActionFilter))]
```

`TypeFilter` permet notamment de construire le Filter en utilisant les dépendances nécessaires.

Il est utile lorsqu'on souhaite appliquer un Filter avec injection de dépendances sans enregistrer exactement le type de la même manière qu'avec `ServiceFilter`.

---

# 15. Filter avec dépendances

Exemple :

```csharp
public class LoggingActionFilter(
    ILogger<LoggingActionFilter> logger)
    : IActionFilter
{
    public void OnActionExecuting(
        ActionExecutingContext context)
    {
        logger.LogInformation(
            "Action en cours");
    }

    public void OnActionExecuted(
        ActionExecutedContext context)
    {
        logger.LogInformation(
            "Action terminée");
    }
}
```

Le Filter bénéficie de la Dependency Injection.

---

# 16. Filter global

On peut également appliquer un Filter globalement.

Exemple :

```csharp
builder.Services.AddControllers(options =>
{
    options.Filters.Add<LoggingActionFilter>();
});
```

Le Filter s'applique alors aux actions concernées par cette configuration MVC.

Mentalement :

```text
Global
   |
   +--> Controller A
   +--> Controller B
   +--> Controller C
```

---

# 17. Filter sur un Controller

On peut également appliquer un Filter à un contrôleur entier.

Exemple :

```csharp
[ServiceFilter(typeof(LoggingActionFilter))]
public class HotelsController : ControllerBase
{
}
```

Le Filter concerne alors les actions du contrôleur.

On peut donc avoir trois niveaux conceptuels :

```text
Global
   |
Controller
   |
Action
```

---

# 18. Exception Filters

Les Exception Filters permettent de traiter certaines exceptions produites pendant l'exécution MVC.

Exemple conceptuel :

```csharp
public class GlobalExceptionFilter
    : IExceptionFilter
{
    public void OnException(ExceptionContext context)
    {
        context.Result =
            new ObjectResult(new
            {
                message = "Une erreur est survenue."
            })
            {
                StatusCode = 500
            };
    }
}
```

Cependant, pour une API ASP.NET Core moderne, un **Middleware global d'exception** est souvent plus adapté lorsqu'on souhaite gérer les exceptions à l'échelle de toute la pipeline HTTP.

---

# 19. Result Filters

Les Result Filters interviennent autour de l'exécution du résultat produit par l'action.

Mentalement :

```text
Action
   |
   v
Action Result
   |
   v
Result Filter
   |
   v
HTTP Response
```

Ils sont utiles lorsqu'un comportement doit concerner le résultat MVC plutôt que l'exécution de l'action elle-même.

---

# 20. Resource Filters

Les Resource Filters interviennent plus tôt dans le pipeline MVC que les Action Filters.

Ils peuvent être utiles lorsqu'on veut agir autour d'une grande partie de l'exécution MVC.

Mentalement :

```text
Request
   |
   v
Resource Filter
   |
   v
Model Binding / autres étapes MVC
   |
   v
Action Filter
   |
   v
Action
```

Ils sont plus spécialisés et moins utilisés dans les cas simples.

---

# 21. Ordre général des Filters

Il ne faut pas mémoriser uniquement une liste ; il faut comprendre l'idée.

Le pipeline MVC possède plusieurs niveaux :

```text
Authorization
      |
      v
Resource
      |
      v
Action
      |
      v
Result
```

Les Exception Filters ont leur propre rôle autour des exceptions.

L'ordre exact peut dépendre du type de Filter, de l'ordre configuré et de la structure du pipeline.

---

# 22. Filter vs Middleware

C'est une question d'entretien très fréquente.

### Middleware

Travaille au niveau du pipeline HTTP global.

```text
HTTP Request
    |
    v
Middleware
    |
    v
Middleware
    |
    v
Endpoint
```

Il peut agir sur :

```text
toute l'application
```

### Filter

Travaille dans le pipeline MVC/Controller.

```text
Controller
    |
    v
Filter
    |
    v
Action
```

Il connaît davantage le contexte MVC :

```text
Action
Controller
ActionArguments
ModelState
```

---

# 23. Comparaison Middleware / Filter

| Élément | Middleware | Filter |
|---|---|---|
| Niveau | HTTP | MVC/Controller |
| S'exécute avant le routing | Peut | Non, selon le type |
| Accès à l'action | Non directement | Oui |
| Accès aux arguments de l'action | Non | Oui |
| Global | Oui | Oui ou ciblé |
| Adapté à toute requête HTTP | Oui | Non |
| Spécifique aux Controllers | Non | Oui |
| Exception globale | Très adapté | Plus ciblé |

Règle mentale :

```text
Middleware = pipeline HTTP

Filter = pipeline MVC
```

---

# 24. Filter vs Middleware : exemple concret

Supposons que l'on veuille ajouter un Correlation ID.

C'est généralement un bon candidat pour un Middleware :

```text
Toutes les requêtes HTTP
        |
        v
Correlation ID
```

Supposons maintenant qu'on veuille inspecter les arguments d'une action :

```csharp
context.ActionArguments
```

Un Action Filter est beaucoup plus adapté.

---

# 25. Filter vs Controller

Le Controller contient normalement la logique d'orchestration de l'endpoint.

Exemple :

```csharp
[HttpGet]
public async Task<IActionResult> Get()
{
    var hotels = await _hotelService.GetAllAsync();

    return Ok(hotels);
}
```

Le Filter ne doit pas devenir un deuxième Controller.

Le Filter sert à des comportements transversaux.

---

# 26. Filter vs Service

Un service contient généralement une logique applicative.

Exemple :

```csharp
public async Task<Hotel> CreateAsync(
    CreateHotelDto dto)
{
    // règles métier/application
}
```

Un Filter ne devrait pas contenir toute cette logique.

Mentalement :

```text
Filter
 -> comportement transversal

Service
 -> logique applicative/métier
```

---

# 27. Endpoint Filters

Les **Endpoint Filters** sont particulièrement intéressants avec les Minimal APIs.

Exemple conceptuel :

```csharp
app.MapGet("/hotels/{id}", GetHotel)
   .AddEndpointFilter(async (context, next) =>
   {
       // Avant

       var result = await next(context);

       // Après

       return result;
   });
```

On retrouve le même principe :

```text
Avant
  |
  v
Endpoint
  |
  v
Après
```

Mais le contexte est celui des endpoints plutôt que celui d'un Controller MVC classique.

---

# 28. Quand utiliser un Filter ?

Utilise un Filter lorsqu'un comportement :

- concerne les Controllers ou endpoints MVC ;
- doit intervenir autour d'une action ;
- a besoin d'informations sur l'action ;
- doit éventuellement inspecter ses arguments ;
- doit s'appliquer à certaines actions ou certains contrôleurs.

Exemples :

```text
Logging spécifique aux actions
Validation spécifique
Transformation de résultats
Comportement transversal MVC
Autorisation
Gestion ciblée d'exceptions
```

---

# 29. Quand préférer un Middleware ?

Utilise plutôt un Middleware lorsqu'un comportement concerne le pipeline HTTP global.

Exemples :

```text
Gestion globale des exceptions
Correlation ID
Logging HTTP
Headers
CORS
Authentication
Compression
Rate limiting selon la configuration
```

Attention : certains mécanismes ASP.NET Core ont leurs propres abstractions et il faut respecter le pipeline prévu par le framework.

---

# 30. Exemple : Logging avec Middleware

Un Middleware peut mesurer toute la requête :

```csharp
public async Task InvokeAsync(HttpContext context)
{
    var stopwatch = Stopwatch.StartNew();

    await _next(context);

    stopwatch.Stop();

    _logger.LogInformation(
        "Request {Path} took {Elapsed} ms",
        context.Request.Path,
        stopwatch.ElapsedMilliseconds);
}
```

Il ne dépend pas d'un Controller particulier.

---

# 31. Exemple : Logging avec Action Filter

Un Filter peut mesurer une action :

```csharp
public async Task OnActionExecutionAsync(
    ActionExecutingContext context,
    ActionExecutionDelegate next)
{
    var stopwatch = Stopwatch.StartNew();

    var result = await next();

    stopwatch.Stop();

    Console.WriteLine(
        $"Action exécutée en {stopwatch.ElapsedMilliseconds} ms");
}
```

Ici, on est plus proche du monde MVC.

---

# 32. Ordre et `Order`

Certains Filters peuvent avoir un ordre d'exécution.

Exemple :

```csharp
public class MyFilter : IActionFilter, IOrderedFilter
{
    public int Order => 10;

    public void OnActionExecuting(
        ActionExecutingContext context)
    {
    }

    public void OnActionExecuted(
        ActionExecutedContext context)
    {
    }
}
```

L'ordre permet de contrôler la priorité relative entre Filters lorsque cela est nécessaire.

Il ne faut toutefois pas utiliser `Order` pour compenser une architecture confuse.

---

# 33. Async : attention à `next()`

Avec un Action Filter :

```csharp
var result = await next();
```

`next()` signifie :

> « Continue vers l'étape suivante, notamment l'action, puis reviens avec son résultat. »

Si tu oublies :

```csharp
await next();
```

le pipeline ne se comporte pas comme prévu.

Mentalement :

```text
Filter
 |
 | avant
 v
next()
 |
 v
Action
 |
 v
retour
 |
 v
Filter
 |
 | après
 v
Response
```

---

# 34. Les erreurs fréquentes

## Erreur 1 : utiliser un Filter pour tout

Un Filter n'est pas un remplacement universel du Middleware.

Si le comportement concerne toute requête HTTP, pense d'abord :

```text
Middleware
```

---

## Erreur 2 : mettre la logique métier dans un Filter

Éviter :

```text
Filter
 -> vérifie le stock
 -> modifie la commande
 -> calcule le prix
 -> sauvegarde
```

Cette logique appartient plutôt à l'application/domaine.

---

## Erreur 3 : confondre Authorization et Authentication

```text
Authentication
    -> Qui es-tu ?

Authorization
    -> As-tu le droit ?
```

Un Filter ou middleware ne change pas cette distinction conceptuelle.

---

## Erreur 4 : oublier la DI

Si ton Filter dépend de :

```csharp
ILogger<T>
IService
IRepository
```

il faut que son activation soit compatible avec le conteneur DI.

---

## Erreur 5 : créer trop de Filters

Un Filter doit résoudre un vrai problème transversal.

S'il contient une logique utilisée une seule fois, il est probablement inutile.

---

# 35. Exemple complet

Filter :

```csharp
public class ExecutionTimeFilter(
    ILogger<ExecutionTimeFilter> logger)
    : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(
        ActionExecutingContext context,
        ActionExecutionDelegate next)
    {
        var stopwatch = Stopwatch.StartNew();

        var result = await next();

        stopwatch.Stop();

        logger.LogInformation(
            "Action {Action} executed in {Elapsed} ms",
            context.ActionDescriptor.DisplayName,
            stopwatch.ElapsedMilliseconds);
    }
}
```

Enregistrement :

```csharp
builder.Services.AddScoped<ExecutionTimeFilter>();
```

Utilisation :

```csharp
[ServiceFilter(typeof(ExecutionTimeFilter))]
[HttpGet]
public async Task<IActionResult> GetHotels()
{
    var hotels = await _service.GetAllAsync();

    return Ok(hotels);
}
```

Flux :

```text
Request
   |
   v
ExecutionTimeFilter
   |
   | Start stopwatch
   v
GetHotels()
   |
   v
Service
   |
   v
Retour
   |
   v
ExecutionTimeFilter
   |
   | Stop stopwatch
   v
Response
```

---

# 36. Architecture mentale complète

Pour comprendre ASP.NET Core, imagine plusieurs couches :

```text
                    HTTP REQUEST
                         |
                         v
                  +--------------+
                  |  Middleware  |
                  +--------------+
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
              +----------------+
              | MVC / Endpoint |
              +----------------+
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

Cette représentation est beaucoup plus utile que de mémoriser des attributs isolés.

---

# 37. Règle mentale

Quand tu hésites entre Middleware et Filter, pose-toi une seule question :

> « Est-ce que j'ai besoin de connaître l'action MVC et ses arguments ? »

### Non

Pense :

```text
Middleware
```

### Oui

Pense :

```text
Filter
```

Exemple :

```text
Ajouter un header à toutes les réponses
    -> Middleware

Mesurer le temps d'une action précise
    -> Action Filter

Lire context.ActionArguments
    -> Action Filter

Gérer globalement les exceptions HTTP
    -> Middleware
```

---

# 38. À retenir

1. Un Filter intervient dans le pipeline MVC/Controller.
2. Un Middleware intervient dans le pipeline HTTP.
3. Les Action Filters peuvent exécuter du code avant et après une action.
4. `await next()` permet de continuer vers l'action puis de revenir au Filter.
5. Un Filter peut accéder à `ActionArguments`.
6. Un Filter peut être appliqué globalement, à un Controller ou à une action.
7. `ServiceFilter` et `TypeFilter` permettent notamment l'utilisation de DI avec des Filters.
8. `[Authorize]` participe au mécanisme d'autorisation.
9. Les Exception Filters existent mais le Middleware est souvent plus adapté pour une gestion globale des exceptions.
10. Les Result Filters travaillent autour de l'exécution du résultat.
11. Les Resource Filters interviennent plus tôt dans le pipeline MVC.
12. Les Endpoint Filters sont notamment utilisés avec les Minimal APIs.
13. Un Filter ne doit pas devenir un endroit où placer toute la logique métier.
14. Un Middleware n'a pas la même connaissance du contexte MVC qu'un Filter.
15. Le bon choix dépend du niveau auquel le comportement doit s'appliquer.

---

# Questions d'entretien

### 1. Qu'est-ce qu'un Filter ?

C'est un mécanisme ASP.NET Core permettant d'exécuter un comportement autour de certaines étapes du pipeline MVC, notamment autour des actions de Controller.

### 2. Quelle différence entre Middleware et Filter ?

Le Middleware travaille au niveau du pipeline HTTP global ; le Filter travaille dans le pipeline MVC et peut notamment accéder au contexte de l'action.

### 3. Que fait `await next()` dans un Action Filter ?

Il continue l'exécution vers l'étape suivante, notamment l'action, puis permet au Filter de reprendre après son exécution.

### 4. Comment accéder aux paramètres d'une action depuis un Filter ?

Avec :

```csharp
context.ActionArguments
```

### 5. Comment appliquer un Filter à une seule action ?

Par exemple avec :

```csharp
[ServiceFilter(typeof(MyFilter))]
```

ou une autre méthode adaptée comme `TypeFilter`.

### 6. Pourquoi utiliser un Filter plutôt que mettre le code dans le Controller ?

Pour centraliser un comportement transversal et éviter de dupliquer la même logique dans plusieurs actions.

### 7. Quand préférer un Middleware ?

Lorsqu'un comportement doit concerner le pipeline HTTP global plutôt qu'une action MVC spécifique.

### 8. Pourquoi ne pas mettre la logique métier dans un Filter ?

Parce que le Filter est destiné à des comportements transversaux liés au pipeline ; la logique métier doit rester dans les couches applicatives/domaines appropriées.

### 9. Quels sont les principaux types de Filters ?

Authorization, Resource, Action, Exception et Result Filters. Les Endpoint Filters constituent également une abstraction importante pour les endpoints, notamment les Minimal APIs.

### 10. Quelle différence entre Authentication et Authorization ?

Authentication :

```text
Qui es-tu ?
```

Authorization :

```text
As-tu le droit ?
```

---

# Phrase à retenir

> **Middleware = pipeline HTTP ; Filter = pipeline MVC ; Controller = orchestration de l'endpoint ; Service = logique applicative/métier.**
