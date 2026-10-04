# Logging en .NET

## 1. Définition

Le **logging** permet à une application d'enregistrer ce qui se passe pendant son exécution.

On peut enregistrer par exemple :

- le démarrage de l'application ;
- les requêtes importantes ;
- les erreurs ;
- les avertissements ;
- des informations de diagnostic ;
- des événements métier utiles au support.

En .NET, l'abstraction principale est :

```csharp
ILogger<T>
```

L'idée fondamentale est :

> Le logging permet d'observer le comportement d'une application sans modifier sa logique métier.

---

# 2. Pourquoi ne pas utiliser `Console.WriteLine` partout ?

On pourrait écrire :

```csharp
Console.WriteLine("User created");
```

Mais `Console.WriteLine` est très limité pour une application professionnelle.

Il ne fournit pas naturellement :

- de niveaux de log ;
- de catégories ;
- de filtrage structuré ;
- de providers configurables ;
- de contexte ;
- d'intégration avec des systèmes de collecte de logs.

Avec `ILogger` :

```csharp
_logger.LogInformation("User created");
```

on utilise le système de logging de .NET.

---

# 3. `ILogger<T>`

Dans une classe :

```csharp
public class UserService
{
    private readonly ILogger<UserService> _logger;

    public UserService(ILogger<UserService> logger)
    {
        _logger = logger;
    }
}
```

Le type :

```csharp
ILogger<UserService>
```

indique la catégorie associée au logger.

La catégorie permet notamment de savoir d'où vient le message.

Mentalement :

```text
UserService
     |
     v
ILogger<UserService>
     |
     v
Log
```

---

# 4. Les niveaux de log

.NET fournit plusieurs niveaux courants :

```text
Trace
Debug
Information
Warning
Error
Critical
```

Ils permettent d'indiquer la gravité ou l'importance du message.

---

# 5. Trace

```csharp
_logger.LogTrace("Entering method {Method}", nameof(CreateUser));
```

`Trace` est destiné aux informations très détaillées.

Il peut être utile pour diagnostiquer un problème très précis.

En production, ce niveau est généralement très filtré.

---

# 6. Debug

```csharp
_logger.LogDebug(
    "User {UserId} loaded from cache",
    userId);
```

`Debug` sert principalement aux informations utiles pendant le développement ou le diagnostic.

Il est généralement plus détaillé que `Information`.

---

# 7. Information

```csharp
_logger.LogInformation(
    "User {UserId} created",
    userId);
```

`Information` correspond aux événements normaux et importants du fonctionnement de l'application.

Exemples :

```text
Application started
Order created
Payment completed
Import finished
```

Il ne faut toutefois pas enregistrer absolument tout en Information.

---

# 8. Warning

```csharp
_logger.LogWarning(
    "Login attempt for unknown user {Email}",
    email);
```

`Warning` signale une situation anormale ou potentiellement problématique, sans forcément signifier que l'opération a échoué.

Exemples :

```text
Retrying an external API call
Configuration value is missing but a default exists
Unusual number of requests
```

---

# 9. Error

```csharp
_logger.LogError(
    exception,
    "Error while creating user {UserId}",
    userId);
```

`Error` indique généralement qu'une opération a échoué.

Lorsqu'une exception existe, il faut généralement la passer au logger :

```csharp
_logger.LogError(exception, "Operation failed");
```

et non simplement :

```csharp
_logger.LogError(exception.Message);
```

Pourquoi ?

Parce que le logger peut conserver les informations de l'exception, notamment :

- type ;
- message ;
- stack trace ;
- inner exception.

---

# 10. Critical

```csharp
_logger.LogCritical(
    exception,
    "Database service is unavailable");
```

`Critical` est réservé aux problèmes très graves pouvant compromettre le fonctionnement global de l'application.

Exemples possibles :

```text
Application cannot start
Critical infrastructure failure
Essential dependency unavailable
```

Il ne faut pas utiliser `Critical` pour toutes les exceptions.

---

# 11. Tableau des niveaux

| Niveau | Idée générale |
|---|---|
| Trace | détails extrêmement fins |
| Debug | diagnostic/développement |
| Information | fonctionnement normal important |
| Warning | situation anormale mais pas forcément bloquante |
| Error | opération échouée |
| Critical | problème très grave |

### Règle mentale

```text
Trace
  ↓
Debug
  ↓
Information
  ↓
Warning
  ↓
Error
  ↓
Critical
```

Plus on descend, plus la gravité est importante.

---

# 12. Logging structuré

Une notion très importante en .NET est le **structured logging**.

Éviter :

```csharp
_logger.LogInformation(
    $"User {userId} created");
```

Préférer :

```csharp
_logger.LogInformation(
    "User {UserId} created",
    userId);
```

Ici :

```text
{UserId}
```

est une propriété structurée du log.

Le système de logging peut donc conserver :

```text
Message : User {UserId} created
UserId  : 42
```

au lieu de recevoir uniquement une chaîne déjà construite.

---

# 13. Pourquoi le logging structuré est important ?

Supposons qu'on ait des milliers de logs.

Avec :

```text
User 42 created
User 43 created
User 44 created
```

un système d'observabilité peut avoir plus de difficulté à exploiter les informations si tout est traité comme une simple chaîne.

Avec :

```csharp
_logger.LogInformation(
    "User {UserId} created",
    userId);
```

on conserve la donnée :

```text
UserId = 42
```

Cela facilite :

- les recherches ;
- les filtres ;
- les dashboards ;
- les corrélations ;
- l'analyse des incidents.

---

# 14. Ne pas concaténer inutilement les messages

Éviter :

```csharp
_logger.LogInformation(
    "User " + userId + " created");
```

Préférer :

```csharp
_logger.LogInformation(
    "User {UserId} created",
    userId);
```

La deuxième forme correspond au logging structuré.

---

# 15. Logger une exception

Bon :

```csharp
try
{
    await service.CreateAsync();
}
catch (Exception ex)
{
    _logger.LogError(
        ex,
        "Unable to create user");

    throw;
}
```

L'exception est fournie séparément :

```csharp
_logger.LogError(ex, "...");
```

Cela permet au système de conserver les informations détaillées de l'exception.

---

# 16. Ne pas avaler les exceptions

Mauvais :

```csharp
try
{
    await service.ExecuteAsync();
}
catch (Exception ex)
{
    _logger.LogError(ex, "Error");
}
```

Si aucune autre action n'est effectuée, l'exception est avalée.

Le reste de l'application peut alors croire que l'opération s'est bien terminée.

Selon l'architecture, il peut être préférable de :

```csharp
throw;
```

après le logging, ou laisser remonter l'exception vers un middleware global.

---

# 17. Logging et middleware global

Dans une API ASP.NET Core, une architecture fréquente est :

```text
Controller
    |
    v
Service
    |
    X
Exception
    |
    v
Exception Middleware
    |
    v
Logger
    |
    v
HTTP response
```

On peut ainsi centraliser la gestion des erreurs.

Par exemple :

```csharp
try
{
    await _next(context);
}
catch (Exception ex)
{
    _logger.LogError(
        ex,
        "Unhandled exception");

    throw;
}
```

Le middleware peut ensuite transformer l'exception en réponse HTTP appropriée.

---

# 18. Logging et DI

`ILogger<T>` est fourni par le système DI.

On n'a généralement pas besoin de faire :

```csharp
new Logger(...)
```

dans son service.

On demande simplement :

```csharp
public UserService(
    ILogger<UserService> logger)
{
    _logger = logger;
}
```

Le framework fournit l'implémentation appropriée.

Cela correspond au principe de Dependency Injection vu précédemment.

---

# 19. Les providers

`ILogger` est une abstraction.

Le système de logging peut envoyer les logs vers différents **providers**.

Conceptuellement :

```text
ILogger
   |
   +---- Console
   |
   +---- Debug
   |
   +---- Event Log
   |
   +---- système externe
```

Le code de l'application utilise :

```csharp
ILogger
```

sans devoir connaître directement le système de destination.

---

# 20. Configuration du logging

Le logging peut être configuré dans :

```text
appsettings.json
```

Exemple :

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  }
}
```

Cela signifie conceptuellement :

```text
Application
    -> Information minimum

Microsoft.AspNetCore
    -> Warning minimum
```

On peut donc réduire le bruit produit par certains composants du framework.

---

# 21. Pourquoi filtrer les logs ?

Une application peut produire énormément de logs.

Si tout est enregistré :

```text
Trace
Debug
Information
Warning
Error
```

les logs peuvent devenir difficiles à exploiter et coûteux à stocker.

Le filtrage permet de conserver uniquement les niveaux nécessaires.

Exemple :

```text
Development
    Debug

Production
    Information / Warning / Error
```

La configuration exacte dépend des besoins de l'application.

---

# 22. Logger les données utiles

Un bon log doit permettre de comprendre ce qui s'est passé.

Mauvais :

```csharp
_logger.LogError("Error");
```

Meilleur :

```csharp
_logger.LogError(
    ex,
    "Unable to create order {OrderId} for user {UserId}",
    orderId,
    userId);
```

Le deuxième message donne davantage de contexte.

---

# 23. Attention aux données sensibles

Il ne faut pas logger aveuglément toutes les informations.

Éviter de mettre dans les logs :

```text
Passwords
Tokens
API keys
Secrets
Données personnelles sensibles
Informations bancaires
```

Exemple particulièrement mauvais :

```csharp
_logger.LogInformation(
    "Login with password {Password}",
    password);
```

Même avec une bonne infrastructure de logs, une donnée secrète peut finir dans :

- fichiers ;
- systèmes centralisés ;
- dashboards ;
- sauvegardes ;
- outils d'observabilité.

### Règle mentale

> Un log est une donnée persistante potentielle. Traite-le comme une information qui peut sortir de ton application.

---

# 24. Logging des requêtes HTTP

ASP.NET Core peut produire des informations sur les requêtes HTTP.

Conceptuellement :

```text
GET /api/users
200
35 ms
```

Cela peut être utile pour diagnostiquer :

- endpoints lents ;
- erreurs ;
- volumes de trafic ;
- comportements inattendus.

Mais attention au niveau de détail et au volume.

---

# 25. Correlation ID

Dans une architecture distribuée :

```text
Client
   |
   v
API
   |
   v
Service A
   |
   v
Service B
```

Une seule opération peut générer plusieurs logs.

On veut pouvoir relier ces événements.

On peut utiliser un identifiant de corrélation :

```text
CorrelationId = abc-123
```

Puis :

```text
API       -> abc-123
Service A -> abc-123
Service B -> abc-123
```

Cela facilite énormément le diagnostic d'une requête qui traverse plusieurs services.

---

# 26. `ILogger<T>` et catégorie

Avec :

```csharp
ILogger<UserService>
```

la catégorie est généralement liée au type :

```text
UserService
```

Cela permet notamment de filtrer les logs par catégorie.

Par exemple :

```text
MyApp.Services.UserService
MyApp.Services.OrderService
MyApp.Controllers.UsersController
```

On peut donc appliquer des règles différentes selon les parties de l'application.

---

# 27. Logging dans un contrôleur

Exemple :

```csharp
public class UsersController : ControllerBase
{
    private readonly ILogger<UsersController> _logger;

    public UsersController(
        ILogger<UsersController> logger)
    {
        _logger = logger;
    }

    [HttpGet("{id}")]
    public IActionResult Get(int id)
    {
        _logger.LogInformation(
            "Retrieving user {UserId}",
            id);

        return Ok();
    }
}
```

Le contrôleur n'a pas besoin de connaître le provider de logging.

---

# 28. Logging dans un service

Le même principe :

```csharp
public class UserService
{
    private readonly ILogger<UserService> _logger;

    public UserService(
        ILogger<UserService> logger)
    {
        _logger = logger;
    }

    public async Task CreateAsync(int userId)
    {
        _logger.LogInformation(
            "Creating user {UserId}",
            userId);

        // ...
    }
}
```

---

# 29. Logging et performance

Le logging a un coût.

Un log peut nécessiter :

```text
formatage
allocation
écriture
transport
stockage
```

Il faut donc éviter de générer inutilement énormément de logs.

Le logging structuré permet notamment de mieux intégrer le logging dans des systèmes d'observabilité.

Pour les chemins extrêmement chauds, .NET propose aussi des techniques plus avancées comme `LoggerMessage` et les source generators.

Exemple conceptuel :

```csharp
[LoggerMessage(
    EventId = 1001,
    Level = LogLevel.Information,
    Message = "User {UserId} created")]
static partial void UserCreated(
    ILogger logger,
    int userId);
```

Ce mécanisme est surtout intéressant lorsque les performances du logging sont importantes.

---

# 30. EventId

On peut associer un identifiant à un événement de log.

Conceptuellement :

```text
EventId = 1001
Message = User created
```

Cela peut faciliter :

- le filtrage ;
- la recherche ;
- l'analyse ;
- la corrélation de certains événements.

Dans une petite application, ce n'est pas toujours indispensable.

Dans une application importante, cela peut devenir très utile.

---

# 31. Logging vs Monitoring

Ces notions sont liées mais différentes.

## Logging

Répond principalement à :

> Qu'est-ce qui s'est passé ?

Exemple :

```text
Order 123 failed.
```

## Metrics

Répondent plutôt à :

> Combien ? À quelle fréquence ? Quelle valeur ?

Exemple :

```text
95th percentile latency = 250 ms
```

## Tracing

Répond principalement à :

> Quel chemin cette opération a-t-elle parcouru ?

Exemple :

```text
API
  -> Service A
      -> Service B
          -> Database
```

On peut retenir :

```text
Logs    -> événements
Metrics -> mesures
Traces  -> parcours
```

---

# 32. Une bonne stratégie de logging

Pour une API .NET :

```text
Information
    |
    +-- événements importants

Warning
    |
    +-- situations anormales

Error
    |
    +-- opérations échouées

Critical
    |
    +-- problèmes graves

Debug / Trace
    |
    +-- diagnostic détaillé
```

Et surtout :

```text
Contexte
+
Données utiles
-
Secrets
-
Bruit inutile
```

---

# 33. Erreurs fréquentes

## Erreur 1 : utiliser `Console.WriteLine` partout

Préférer :

```csharp
ILogger<T>
```

pour le logging applicatif.

---

## Erreur 2 : logger seulement `"Error"`

Mieux :

```csharp
_logger.LogError(
    ex,
    "Unable to process order {OrderId}",
    orderId);
```

---

## Erreur 3 : logger des secrets

Ne jamais mettre des mots de passe, tokens ou clés secrètes dans les logs.

---

## Erreur 4 : utiliser `Error` pour tout

Une situation anormale n'est pas nécessairement une erreur.

Choisir le niveau adapté.

---

## Erreur 5 : avaler une exception après l'avoir loggée

Le logging ne remplace pas la gestion correcte de l'exception.

---

## Erreur 6 : trop logger

Plus de logs ne signifie pas automatiquement plus de visibilité.

Un système rempli de messages inutiles devient difficile à analyser.

---

# 34. Schéma global

```text
Application
     |
     v
  ILogger<T>
     |
     v
Logging abstraction
     |
     +------------------+
     |                  |
     v                  v
  Filtering          Providers
     |                  |
     |          +-------+-------+
     |          |       |       |
     |          v       v       v
     |       Console  File   External
     |
     v
Logs exploitables
```

L'application reste découplée de la destination finale.

---

# 35. Règle mentale

Quand tu vois :

```csharp
ILogger<UserService>
```

pense :

> « Je demande au système DI un logger associé à UserService. »

Quand tu vois :

```csharp
LogInformation(...)
```

pense :

> « Événement normal important. »

Quand tu vois :

```csharp
LogWarning(...)
```

pense :

> « Situation anormale, mais pas forcément une panne. »

Quand tu vois :

```csharp
LogError(ex, ...)
```

pense :

> « Une opération a échoué et je conserve l'exception. »

Quand tu vois :

```csharp
{UserId}
```

pense :

> « Je crée une propriété structurée exploitable par le système de logs. »

---

# 36. À retenir

1. `ILogger<T>` est l'abstraction principale du logging .NET.
2. `ILogger<T>` est généralement injecté via DI.
3. Les niveaux principaux vont de `Trace` à `Critical`.
4. Le niveau doit refléter l'importance de l'événement.
5. Le logging structuré utilise des propriétés comme `{UserId}`.
6. Pour une exception, passer l'exception au logger.
7. Ne pas avaler une exception simplement parce qu'elle a été loggée.
8. Ne jamais logger des secrets ou des données sensibles inutilement.
9. Les providers déterminent où les logs sont envoyés.
10. La configuration permet de filtrer les niveaux et catégories.
11. Les logs doivent contenir suffisamment de contexte pour diagnostiquer un problème.
12. Un système distribué bénéficie fortement des identifiants de corrélation.
13. Logging, metrics et tracing sont complémentaires.
14. Trop de logs peuvent être aussi problématiques que trop peu.
15. `Console.WriteLine` n'est pas un remplacement du système de logging applicatif.

---

# Questions d'entretien

### 1. Pourquoi utiliser `ILogger<T>` plutôt que `Console.WriteLine` ?

Parce que `ILogger` fournit une abstraction structurée avec niveaux, catégories, filtrage et providers configurables.

### 2. Quelle différence entre Information, Warning et Error ?

`Information` correspond au fonctionnement normal important, `Warning` à une situation anormale et `Error` à une opération qui a échoué.

### 3. Pourquoi utiliser le logging structuré ?

Pour conserver les données du contexte comme des propriétés exploitables par les systèmes d'observabilité.

### 4. Comment logger correctement une exception ?

Par exemple :

```csharp
_logger.LogError(
    ex,
    "Unable to process order {OrderId}",
    orderId);
```

### 5. Pourquoi ne faut-il pas logger un mot de passe ?

Parce que les logs peuvent être persistés, centralisés, sauvegardés et accessibles à plusieurs systèmes ou personnes.

### 6. Qu'est-ce qu'un provider de logging ?

C'est le composant qui détermine où et comment les logs sont écrits ou transmis.

### 7. Pourquoi utiliser des niveaux de logs ?

Pour pouvoir filtrer la quantité de détails selon l'environnement et les besoins de diagnostic.

### 8. Quelle différence entre logs, metrics et traces ?

- Logs : événements.
- Metrics : mesures quantitatives.
- Traces : parcours d'une opération à travers les composants.

---

# Phrase à retenir

> **`ILogger` permet à l'application de décrire ce qui se passe, avec un niveau et un contexte structurés, sans dépendre directement de l'endroit où les logs seront stockés.**
