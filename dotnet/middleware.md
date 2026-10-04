# Middleware en ASP.NET Core

## 1. Définition

Un **middleware** est un composant qui participe au traitement d'une requête HTTP dans ASP.NET Core.

Il peut :

- examiner la requête ;
- modifier la requête ;
- exécuter du code avant le composant suivant ;
- appeler le composant suivant ;
- examiner ou modifier la réponse ;
- arrêter complètement le pipeline.

Le principe fondamental est :

```text
Request
   |
   v
Middleware 1
   |
   v
Middleware 2
   |
   v
Endpoint
   |
   v
Response
```

Un middleware est donc une étape du **pipeline HTTP**.

---

# 2. Le pipeline HTTP

Quand une requête arrive sur une API ASP.NET Core, elle traverse une chaîne de middlewares.

Exemple :

```text
Client
  |
  v
Exception Handling
  |
  v
HTTPS
  |
  v
Static Files
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
Controller / Endpoint
```

Puis la réponse repart dans l'autre sens.

C'est une notion essentielle à comprendre :

> Le pipeline est une chaîne de composants traversés par chaque requête.

---

# 3. Le principe `next`

Un middleware reçoit généralement une fonction représentant le composant suivant.

Conceptuellement :

```csharp
await next(context);
```

signifie :

> « J'ai terminé mon traitement avant le composant suivant ; continue le pipeline. »

Exemple :

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Avant");

    await next();

    Console.WriteLine("Après");
});
```

Si un endpoint répond ensuite :

```text
Avant
Endpoint
Après
```

Le middleware entoure donc l'exécution du reste du pipeline.

---

# 4. Comprendre le pipeline comme une pile

Imagine :

```text
Middleware A
    |
    +--> Middleware B
             |
             +--> Endpoint
             |
             +<-- Response
    |
    +<-- Response
```

Le code placé **avant** `await next()` s'exécute pendant la descente vers l'endpoint.

Le code placé **après** `await next()` s'exécute pendant la remontée de la réponse.

C'est une excellente règle mentale.

```csharp
app.Use(async (context, next) =>
{
    // Descente
    Console.WriteLine("Avant");

    await next();

    // Remontée
    Console.WriteLine("Après");
});
```

---

# 5. Créer un middleware simple

ASP.NET Core permet d'écrire un middleware directement avec `app.Use`.

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine($"Request : {context.Request.Path}");

    await next();

    Console.WriteLine($"Status : {context.Response.StatusCode}");
});
```

Ici :

```csharp
context.Request
```

représente la requête.

Et :

```csharp
context.Response
```

représente la réponse.

---

# 6. `HttpContext`

Le middleware travaille principalement avec :

```csharp
HttpContext
```

Il contient les informations relatives à la requête et à la réponse.

On y trouve notamment :

```csharp
context.Request
context.Response
context.User
context.RequestServices
context.Items
```

### `Request`

Contient notamment :

```text
Method
Path
QueryString
Headers
Cookies
Body
```

Exemple :

```csharp
var path = context.Request.Path;
var method = context.Request.Method;
```

### `Response`

Permet notamment de contrôler :

```text
StatusCode
Headers
Body
```

Exemple :

```csharp
context.Response.StatusCode = 401;
```

---

# 7. Arrêter le pipeline

Un middleware n'est pas obligé d'appeler `next`.

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

Si la clé est absente :

```text
Request
   |
   v
API Key Middleware
   |
   X
   |
   v
401 Unauthorized
```

L'endpoint ne sera jamais exécuté.

### À retenir

```csharp
await next();
```

= continuer.

Ne pas appeler `next()` = arrêter le pipeline.

---

# 8. `Use`, `Run` et `Map`

Ces méthodes sont importantes.

## `Use`

`Use` permet d'ajouter un middleware qui peut décider de continuer ou non.

```csharp
app.Use(async (context, next) =>
{
    await next();
});
```

Il possède généralement une logique avant et/ou après `next`.

---

## `Run`

`Run` ajoute un middleware terminal.

```csharp
app.Run(async context =>
{
    await context.Response.WriteAsync("Hello");
});
```

Il ne reçoit pas de `next`.

Il termine donc le pipeline.

Mentalement :

```text
Use -> peut continuer
Run -> termine
```

---

## `Map`

`Map` permet de créer une branche du pipeline selon un chemin.

```csharp
app.Map("/admin", adminApp =>
{
    adminApp.Run(async context =>
    {
        await context.Response.WriteAsync("Administration");
    });
});
```

Une requête vers :

```text
/admin
```

emprunte cette branche.

---

# 9. Ordre des middlewares

L'ordre est extrêmement important.

Exemple :

```csharp
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
```

L'authentification doit généralement être exécutée avant l'autorisation.

Pourquoi ?

Parce que l'autorisation doit savoir **qui est l'utilisateur**.

Mentalement :

```text
Authentication
      |
      v
Qui es-tu ?
      |
      v
Authorization
      |
      v
As-tu le droit ?
```

Changer l'ordre peut donc modifier complètement le comportement de l'application.

---

# 10. Exemple de pipeline

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddAuthentication();
builder.Services.AddAuthorization();
builder.Services.AddControllers();

var app = builder.Build();

app.UseExceptionHandler("/error");

app.UseHttpsRedirection();

app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

On peut lire cela comme :

```text
Erreur
  ↓
HTTPS
  ↓
Authentication
  ↓
Authorization
  ↓
Controllers
```

---

# 11. Middleware d'exception globale

Une utilisation très importante du middleware est la gestion centralisée des exceptions.

Sans middleware global, chaque contrôleur pourrait être tenté de faire :

```csharp
try
{
    // ...
}
catch (Exception ex)
{
    // ...
}
```

Cela duplique la logique.

On peut plutôt centraliser le traitement.

```csharp
app.UseExceptionHandler();
```

Ou créer un middleware personnalisé.

Conceptuellement :

```text
Request
   |
   v
Exception Middleware
   |
   v
Controller
   |
   X
Exception
   |
   v
Exception Middleware
   |
   v
HTTP 500 / ProblemDetails
```

Le middleware devient un point central pour transformer les exceptions en réponses HTTP cohérentes.

---

# 12. Middleware personnalisé avec une classe

Quand la logique devient importante, il est préférable de créer une classe.

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
            $"Request : {context.Request.Method} {context.Request.Path}");

        await _next(context);

        Console.WriteLine(
            $"Response : {context.Response.StatusCode}");
    }
}
```

Puis :

```csharp
app.UseMiddleware<RequestLoggingMiddleware>();
```

---

# 13. `RequestDelegate`

Dans un middleware :

```csharp
private readonly RequestDelegate _next;
```

`RequestDelegate` représente une fonction capable de traiter un `HttpContext`.

Conceptuellement :

```csharp
Task RequestDelegate(HttpContext context)
```

Donc :

```csharp
await _next(context);
```

signifie :

> Appelle le prochain composant du pipeline avec ce `HttpContext`.

---

# 14. Pourquoi `InvokeAsync` ?

Un middleware conventionnel possède généralement une méthode :

```csharp
public async Task InvokeAsync(HttpContext context)
```

ASP.NET Core l'utilise pour exécuter le middleware.

Le nom `Invoke` ou `InvokeAsync` est une convention reconnue par le framework.

---

# 15. Middleware et Dependency Injection

Un middleware peut recevoir des dépendances.

Pour les dépendances classiques :

```csharp
public class LoggingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<LoggingMiddleware> _logger;

    public LoggingMiddleware(
        RequestDelegate next,
        ILogger<LoggingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        _logger.LogInformation(
            "Request {Path}",
            context.Request.Path);

        await _next(context);
    }
}
```

Le middleware utilise donc lui aussi le conteneur DI.

---

# 16. Attention aux lifetimes dans les middlewares

Les middlewares conventionnels sont généralement créés une fois et réutilisés.

Il faut donc faire attention lorsqu'on leur injecte des services ayant une durée de vie scoped.

Plutôt que de capturer directement une dépendance scoped dans le constructeur, on peut l'injecter dans `InvokeAsync`.

Exemple :

```csharp
public async Task InvokeAsync(
    HttpContext context,
    IUserService userService)
{
    await _next(context);
}
```

Ici, `IUserService` peut être résolu dans le scope de la requête.

### Règle mentale

> Le constructeur du middleware est lié à la durée de vie du middleware ; les paramètres de `InvokeAsync` peuvent être résolus pour la requête.

---

# 17. Middleware vs Filter

Les deux sont souvent confondus.

## Middleware

Agit au niveau du pipeline HTTP global.

```text
Request
   |
Middleware
   |
Routing
   |
Controller
```

Il peut donc concerner pratiquement toute l'application.

## Filter

Agit dans le pipeline MVC/API autour des actions ou contrôleurs.

```text
Middleware
   |
Controller pipeline
   |
Filter
   |
Action
```

### Mental model

> Middleware = niveau HTTP/application.

> Filter = niveau MVC/action.

---

# 18. Middleware vs Controller

Un contrôleur contient généralement la logique liée à une fonctionnalité métier ou à une ressource HTTP.

Un middleware contient plutôt une préoccupation transversale.

Exemples de middleware :

- gestion globale des exceptions ;
- logging ;
- authentification ;
- sécurité ;
- headers ;
- métriques ;
- corrélation des requêtes ;
- traitement global de certaines requêtes.

Il ne faut donc pas transformer un middleware en énorme classe contenant toute la logique métier.

---

# 19. Middleware et authentification

Un pipeline peut contenir :

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

Authentication répond à :

> Qui est l'utilisateur ?

Authorization répond à :

> Cet utilisateur a-t-il le droit d'effectuer cette opération ?

Exemple :

```text
Request
   |
Authentication
   |
   +--> construit HttpContext.User
   |
Authorization
   |
   +--> vérifie les policies/roles
   |
Controller
```

---

# 20. Middleware et `HttpContext.User`

Après l'authentification, l'identité de l'utilisateur est généralement accessible via :

```csharp
context.User
```

Dans un contrôleur :

```csharp
User
```

correspond au même concept.

On peut donc récupérer des informations comme les claims.

Exemple :

```csharp
var userId = context.User.FindFirst("sub")?.Value;
```

---

# 21. Middleware et réponse HTTP

Un middleware peut modifier la réponse après l'appel au composant suivant.

```csharp
app.Use(async (context, next) =>
{
    await next();

    context.Response.Headers["X-App-Version"] = "1";
});
```

Le middleware laisse d'abord l'endpoint produire la réponse, puis ajoute un header.

Attention : certains éléments de la réponse peuvent déjà être envoyés au client. Il n'est donc pas toujours possible de modifier la réponse tardivement.

---

# 22. Middleware avec traitement avant/après

C'est un modèle très fréquent :

```csharp
public async Task InvokeAsync(HttpContext context)
{
    // Avant
    var start = DateTime.UtcNow;

    await _next(context);

    // Après
    var duration = DateTime.UtcNow - start;
}
```

Cela permet par exemple de mesurer la durée d'une requête.

```text
Start
  |
  v
Pipeline
  |
  v
Endpoint
  |
  v
End
```

On calcule alors :

```text
duration = End - Start
```

---

# 23. Middleware et cancellation

Le token de la requête est accessible via :

```csharp
context.RequestAborted
```

Il représente l'annulation liée à la requête.

Exemple :

```csharp
await service.ExecuteAsync(
    context.RequestAborted);
```

Si le client abandonne la requête, les opérations qui respectent ce token peuvent être annulées.

Cela rejoint la notion de `CancellationToken` étudiée dans une autre fiche.

---

# 24. Erreurs fréquentes

## Erreur 1 : mauvais ordre

```csharp
app.UseAuthorization();
app.UseAuthentication();
```

Ce n'est généralement pas l'ordre attendu.

Préférer :

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

---

## Erreur 2 : oublier `next`

```csharp
app.Use(async (context, next) =>
{
    Console.WriteLine("Hello");
});
```

Le pipeline est arrêté ici.

Si ce n'est pas volontaire :

```csharp
await next();
```

est nécessaire.

---

## Erreur 3 : mettre de la logique métier partout

Un middleware n'est pas un service métier.

Mieux :

```text
Middleware
    |
    v
Service métier
    |
    v
Repository
```

---

## Erreur 4 : modifier la réponse trop tard

Une réponse peut devenir impossible à modifier après le début de son envoi.

Il faut donc comprendre à quel moment le middleware agit.

---

## Erreur 5 : capturer incorrectement des dépendances scoped

Attention particulièrement aux services scoped utilisés depuis un middleware réutilisé pendant toute la vie de l'application.

---

# 25. Middleware classique vs Endpoint

Il faut distinguer :

```text
Middleware
```

et :

```text
Endpoint
```

Le middleware est une étape du pipeline.

L'endpoint est généralement la destination finale correspondant à une route.

Exemple :

```text
GET /api/users
       |
       v
Routing
       |
       v
Endpoint correspondant
       |
       v
UsersController.GetUsers()
```

Le middleware peut entourer l'exécution de cet endpoint.

---

# 26. Visualiser toute la requête

Une bonne représentation mentale est :

```text
                REQUÊTE
                   |
                   v
        +----------------------+
        | Exception Middleware |
        +----------------------+
                   |
                   v
        +----------------------+
        |      HTTPS           |
        +----------------------+
                   |
                   v
        +----------------------+
        |      Routing         |
        +----------------------+
                   |
                   v
        +----------------------+
        |  Authentication      |
        +----------------------+
                   |
                   v
        +----------------------+
        |  Authorization       |
        +----------------------+
                   |
                   v
        +----------------------+
        |      Endpoint        |
        +----------------------+
                   |
                   v
                RÉPONSE
```

Chaque middleware peut décider :

```text
continuer
   ou
arrêter
```

et peut effectuer un traitement :

```text
avant
   |
next()
   |
après
```

---

# 27. Règle mentale

Quand tu vois :

```csharp
app.Use(...)
```

pense :

> « J'ajoute une étape dans le pipeline HTTP. »

Quand tu vois :

```csharp
await next();
```

pense :

> « Je passe la requête au prochain composant. »

Quand tu vois du code après :

```csharp
await next();
```

pense :

> « Je suis maintenant dans la phase de remontée de la réponse. »

Quand tu vois :

```csharp
app.Run(...)
```

pense :

> « Ici, le pipeline se termine. »

---

# 28. À retenir

1. Un middleware est une étape du pipeline HTTP.
2. Chaque requête traverse le pipeline dans un ordre déterminé.
3. `await next()` permet de continuer vers le middleware suivant.
4. Le code avant `next()` s'exécute à la descente.
5. Le code après `next()` s'exécute à la remontée.
6. Ne pas appeler `next()` peut arrêter le pipeline.
7. `Use` permet généralement de continuer.
8. `Run` est terminal.
9. `Map` permet de créer une branche du pipeline.
10. L'ordre des middlewares est essentiel.
11. Middleware et Filter ne travaillent pas exactement au même niveau.
12. Les préoccupations transversales sont de bons candidats pour les middlewares.
13. `HttpContext` donne accès à la requête, à la réponse et au contexte utilisateur.
14. `RequestAborted` permet de respecter l'annulation d'une requête.
15. Un middleware ne doit pas devenir un endroit où l'on met toute la logique métier.

---

# Questions d'entretien

### 1. Qu'est-ce qu'un middleware ?

Un composant du pipeline HTTP ASP.NET Core qui peut traiter une requête, appeler le composant suivant et éventuellement traiter la réponse.

### 2. À quoi sert `next` ?

Il représente le prochain composant du pipeline.

### 3. Que se passe-t-il si on n'appelle pas `next()` ?

Le pipeline s'arrête à cet endroit, sauf si le middleware produit lui-même une réponse ou si cet arrêt est volontaire.

### 4. Pourquoi l'ordre des middlewares est-il important ?

Parce que chaque middleware s'exécute dans l'ordre où il a été ajouté et peut dépendre du travail effectué par les précédents.

### 5. Différence entre Authentication et Authorization ?

Authentication détermine l'identité de l'utilisateur. Authorization détermine s'il possède les permissions nécessaires.

### 6. Différence entre Middleware et Filter ?

Le middleware intervient au niveau du pipeline HTTP global ; le filter intervient dans le pipeline MVC/API autour des contrôleurs ou actions.

### 7. Pourquoi utiliser un middleware d'exception globale ?

Pour centraliser la gestion des exceptions et produire des réponses HTTP cohérentes sans répéter des `try/catch` dans chaque contrôleur.

### 8. Pourquoi le code après `await next()` est-il important ?

Parce qu'il permet d'effectuer un traitement après l'exécution du reste du pipeline, par exemple mesurer la durée ou modifier certains éléments de la réponse.

---

# Phrase à retenir

> **Un middleware est une étape du pipeline HTTP : il peut agir avant `next()`, laisser la requête continuer, puis agir à nouveau après `next()`.**
