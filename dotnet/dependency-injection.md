# Dependency Injection en .NET

## 1. Définition

La **Dependency Injection (DI)** est un mécanisme qui permet à une classe de recevoir les objets dont elle dépend au lieu de les créer elle-même.

Exemple :

```csharp
public class UserService
{
    private readonly UserRepository _repository;

    public UserService(UserRepository repository)
    {
        _repository = repository;
    }
}
```

`UserService` dépend de `UserRepository`.

Au lieu de faire :

```csharp
var repository = new UserRepository();
```

dans `UserService`, la dépendance est fournie depuis l'extérieur.

C'est le principe fondamental :

> Une classe utilise ses dépendances ; elle ne devrait généralement pas être responsable de leur création.

---

## 2. Pourquoi utiliser la DI ?

Sans DI, une classe peut devenir fortement couplée à ses dépendances :

```csharp
public class UserService
{
    private readonly UserRepository _repository;

    public UserService()
    {
        _repository = new UserRepository();
    }
}
```

Le problème est que `UserService` connaît directement la manière de construire `UserRepository`.

Si demain on veut remplacer :

```text
UserRepository
```

par :

```text
SqlUserRepository
ApiUserRepository
FakeUserRepository
```

il faut modifier `UserService`.

Avec la DI :

```csharp
public class UserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }
}
```

`UserService` dépend de l'abstraction `IUserRepository`.

La création de l'objet est déplacée vers la configuration de l'application.

---

# 3. Dépendance : qu'est-ce que cela signifie ?

Une dépendance est simplement un objet dont une classe a besoin pour effectuer son travail.

Exemple :

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;
    private readonly ILogger<OrderService> _logger;

    public OrderService(
        IOrderRepository repository,
        ILogger<OrderService> logger)
    {
        _repository = repository;
        _logger = logger;
    }
}
```

`OrderService` dépend de :

- `IOrderRepository`
- `ILogger<OrderService>`

Ces objets sont donc ses dépendances.

---

# 4. Injection par constructeur

En .NET, l'injection par constructeur est généralement la forme à privilégier.

```csharp
public class UserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }
}
```

ASP.NET Core voit que `UserService` nécessite `IUserRepository`.

Le conteneur DI cherche alors une implémentation enregistrée pour cette interface.

Mentalement :

```text
UserService demande IUserRepository
             |
             v
       DI Container
             |
             v
       UserRepository
```

Le constructeur exprime donc clairement les besoins de la classe.

---

# 5. IoC et DI : quelle différence ?

Ces deux notions sont souvent confondues.

## IoC — Inversion of Control

L'**Inversion of Control** est le principe général.

Normalement, une classe contrôle elle-même la création de ses objets :

```csharp
var repository = new UserRepository();
```

Avec l'IoC, cette responsabilité est transférée à une autre partie du système.

## DI — Dependency Injection

La DI est une technique permettant d'appliquer cette inversion de contrôle.

```text
IoC
 |
 +-- DI
      |
      +-- Constructor Injection
      +-- Method Injection
      +-- Property Injection
```

Dans ASP.NET Core, la forme la plus courante est la constructor injection.

### À retenir

> IoC est le principe ; DI est une manière de l'implémenter.

---

# 6. Le conteneur DI d'ASP.NET Core

ASP.NET Core possède un conteneur DI intégré.

On enregistre les services dans `Program.cs`.

Exemple :

```csharp
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddScoped<UserService>();
```

Cela signifie :

```text
IUserRepository
       |
       v
UserRepository
```

Lorsque le framework doit créer `UserService`, il sait comment obtenir `IUserRepository`.

---

# 7. Les trois principales durées de vie

ASP.NET Core propose principalement :

```csharp
AddTransient
AddScoped
AddSingleton
```

La différence essentielle est la durée de vie de l'objet.

| Lifetime | Création | Utilisation typique |
|---|---|---|
| Transient | nouvelle instance à chaque résolution | services légers/stateless |
| Scoped | une instance par scope | services liés à une requête HTTP |
| Singleton | une seule instance pendant toute la durée de vie de l'application | services partagés et thread-safe |

---

# 8. AddTransient

```csharp
builder.Services.AddTransient<IEmailService, EmailService>();
```

Une nouvelle instance est créée chaque fois que le service est demandé.

Exemple conceptuel :

```text
Service A -> EmailService #1
Service B -> EmailService #2
```

Même type, deux instances différentes.

### Quand l'utiliser ?

Pour un service :

- léger ;
- sans état partagé ;
- dont on n'a pas besoin de conserver la même instance.

---

# 9. AddScoped

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

En ASP.NET Core, un scope correspond généralement à une requête HTTP.

Exemple :

```text
Request HTTP
     |
     +-- UserService #1
     +-- UserService #1
     +-- Repository #1
```

Pendant cette requête, plusieurs composants peuvent recevoir la même instance scoped.

À la requête suivante :

```text
Request HTTP
     |
     +-- UserService #2
```

Une nouvelle instance est créée.

### Mental model

```text
1 requête HTTP = 1 scope
```

C'est pourquoi `Scoped` est très courant avec :

- `DbContext`
- services métier
- repositories

---

# 10. AddSingleton

```csharp
builder.Services.AddSingleton<ICacheService, CacheService>();
```

Une seule instance est créée et réutilisée pendant la durée de vie de l'application.

```text
Request 1 ----\
Request 2 -----+----> CacheService #1
Request 3 ----/
```

Attention : un singleton peut être utilisé simultanément par plusieurs requêtes.

Il doit donc être conçu pour être **thread-safe** si son état est mutable.

---

# 11. Comparaison simple des lifetimes

Imagine une application qui reçoit trois requêtes :

```text
Request 1
Request 2
Request 3
```

### Transient

```text
R1 -> Service #1
R1 -> Service #2

R2 -> Service #3

R3 -> Service #4
```

### Scoped

```text
R1 -> Service #1
     -> Service #1

R2 -> Service #2

R3 -> Service #3
```

### Singleton

```text
R1 -> Service #1
R2 -> Service #1
R3 -> Service #1
```

---

# 12. Le problème du Captive Dependency

Une erreur importante consiste à injecter une dépendance avec une durée de vie plus courte dans une dépendance avec une durée de vie plus longue.

Exemple :

```csharp
builder.Services.AddScoped<IUserRepository, UserRepository>();
builder.Services.AddSingleton<UserService>();
```

Et :

```csharp
public class UserService
{
    public UserService(IUserRepository repository)
    {
    }
}
```

On a :

```text
Singleton
   |
   v
Scoped
```

Le singleton pourrait conserver une référence vers une dépendance scoped au-delà du scope pour lequel celle-ci était prévue.

C'est le problème appelé **captive dependency**.

### Règle mentale

> Une dépendance ne doit pas avoir une durée de vie plus courte que le service qui la capture.

En particulier :

```text
Singleton -> Scoped
```

est une combinaison dangereuse.

---

# 13. Enregistrement avec une factory

On peut demander au conteneur de créer un service avec une fonction :

```csharp
builder.Services.AddSingleton<IMyService>(sp =>
{
    var configuration = sp.GetRequiredService<IConfiguration>();

    return new MyService(configuration);
});
```

`sp` représente le `IServiceProvider`.

Le conteneur peut donc lui-même résoudre d'autres dépendances.

---

# 14. IServiceProvider

`IServiceProvider` permet de demander manuellement un service au conteneur :

```csharp
var service = serviceProvider.GetRequiredService<IUserService>();
```

Cela fonctionne, mais il ne faut pas en faire la manière normale d'utiliser la DI.

Pourquoi ?

Parce que cela cache les dépendances.

Mauvais exemple :

```csharp
public class UserService
{
    private readonly IServiceProvider _provider;

    public UserService(IServiceProvider provider)
    {
        _provider = provider;
    }

    public void Execute()
    {
        var repository =
            _provider.GetRequiredService<IUserRepository>();
    }
}
```

On ne voit plus directement dans le constructeur que `UserService` dépend de `IUserRepository`.

Préférer :

```csharp
public UserService(IUserRepository repository)
{
    _repository = repository;
}
```

### À retenir

> `IServiceProvider` est un outil du conteneur ; ce n'est généralement pas une dépendance métier à injecter partout.

---

# 15. Injection de plusieurs implémentations

On peut enregistrer plusieurs implémentations d'une même interface :

```csharp
builder.Services.AddTransient<INotificationService, EmailNotificationService>();
builder.Services.AddTransient<INotificationService, SmsNotificationService>();
```

On peut ensuite demander :

```csharp
public class NotificationManager
{
    private readonly IEnumerable<INotificationService> _services;

    public NotificationManager(
        IEnumerable<INotificationService> services)
    {
        _services = services;
    }
}
```

Le conteneur fournit les implémentations enregistrées.

Mentalement :

```text
IEnumerable<INotificationService>
          |
          +-- EmailNotificationService
          +-- SmsNotificationService
```

C'est utile pour les systèmes de stratégies, handlers ou notifications multiples.

---

# 16. DI et tests unitaires

Un des grands avantages de la DI est la testabilité.

Supposons :

```csharp
public class UserService
{
    private readonly IUserRepository _repository;

    public UserService(IUserRepository repository)
    {
        _repository = repository;
    }
}
```

En production :

```text
IUserRepository
      |
      v
SqlUserRepository
```

Dans un test :

```text
IUserRepository
      |
      v
FakeUserRepository
```

On peut donc tester `UserService` sans avoir besoin de la vraie base de données.

Exemple conceptuel :

```csharp
var fakeRepository = new FakeUserRepository();

var service = new UserService(fakeRepository);
```

La classe n'a pas besoin de savoir d'où vient la dépendance.

---

# 17. DI ne signifie pas forcément interface

On voit souvent :

```csharp
public class UserService
{
}
```

enregistré directement :

```csharp
builder.Services.AddScoped<UserService>();
```

Il n'est pas obligatoire d'avoir :

```csharp
IUserService
UserService
```

La DI fonctionne aussi avec des classes concrètes.

Une interface est utile lorsqu'elle apporte une vraie abstraction :

```text
plusieurs implémentations
testabilité
découplage
contrat clair
```

Créer systématiquement une interface pour chaque classe peut au contraire ajouter du bruit inutile.

---

# 18. Trop de dépendances dans un constructeur

Exemple :

```csharp
public UserService(
    IRepository repository,
    ILogger<UserService> logger,
    IEmailService email,
    IPaymentService payment,
    IFileService file,
    ICacheService cache,
    IAuditService audit,
    IMapper mapper)
{
}
```

Cela peut être un signal que la classe possède trop de responsabilités.

La DI rend ce problème visible parce que le constructeur expose les dépendances.

### Règle pratique

> Un constructeur avec énormément de dépendances peut être un symptôme d'une classe qui fait trop de choses.

Ce n'est pas une règle mathématique, mais c'est un bon signal architectural.

---

# 19. DI et ASP.NET Core

Dans une API ASP.NET Core :

```csharp
builder.Services.AddScoped<IUserService, UserService>();
builder.Services.AddScoped<IUserRepository, UserRepository>();
```

Puis :

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;

    public UsersController(IUserService userService)
    {
        _userService = userService;
    }

    [HttpGet]
    public IActionResult GetUsers()
    {
        return Ok();
    }
}
```

Le contrôleur ne fait pas :

```csharp
new UserService(...)
```

ASP.NET Core demande au conteneur :

```text
Je dois créer UsersController.

UsersController demande IUserService.

IUserService correspond à UserService.

UserService demande IUserRepository.

IUserRepository correspond à UserRepository.

Créer les objets nécessaires.
```

C'est le conteneur DI qui construit l'ensemble du graphe de dépendances.

---

# 20. Le Dependency Graph

C'est une notion importante pour comprendre la DI.

Supposons :

```csharp
public class UserService
{
    public UserService(IUserRepository repository)
    {
    }
}
```

et :

```csharp
public class UserRepository
{
    public UserRepository(AppDbContext context)
    {
    }
}
```

Le graphe devient :

```text
UsersController
       |
       v
 IUserService
       |
       v
 UserService
       |
       v
IUserRepository
       |
       v
UserRepository
       |
       v
 AppDbContext
```

Le conteneur DI doit résoudre tout ce graphe.

Si une dépendance n'est pas enregistrée ou impossible à construire, l'application peut échouer lors de la résolution.

### Mental model

> Le conteneur DI construit un arbre/graphe d'objets à partir des dépendances déclarées dans les constructeurs.

---

# 21. Ce qu'il ne faut généralement pas faire

## Créer les dépendances manuellement

```csharp
public UserService()
{
    _repository = new UserRepository();
}
```

Cela augmente le couplage.

## Utiliser IServiceProvider partout

```csharp
_provider.GetRequiredService<IRepository>();
```

Cela masque les dépendances.

## Construire soi-même un ServiceProvider

Éviter notamment :

```csharp
var provider = services.BuildServiceProvider();
```

dans la configuration normale de l'application.

Cela peut créer plusieurs conteneurs et entraîner des problèmes de lifetime et de gestion des ressources.

## Injecter un Scoped dans un Singleton

```text
Singleton
   |
   +--> Scoped
```

Risque de captive dependency.

---

# 22. AddTransient vs AddScoped vs AddSingleton

| | Transient | Scoped | Singleton |
|---|---|---|---|
| Instances | nombreuses | une par scope | une pour l'application |
| Requête HTTP | plusieurs possibles | généralement une | même instance |
| État partagé | non recommandé | limité au scope | partagé |
| Thread-safe nécessaire | selon l'état | selon l'état | particulièrement important |
| Exemple courant | petit service stateless | service métier / DbContext | cache/configuration |

---

# 23. DI et `DbContext`

Avec Entity Framework Core, on rencontre souvent :

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(connectionString);
});
```

`AddDbContext` enregistre normalement le `DbContext` avec une durée de vie scoped.

Pourquoi ?

Parce qu'un contexte EF Core est généralement associé à une unité de travail correspondant à une requête HTTP.

On obtient donc souvent :

```text
HTTP Request
      |
      v
 AppDbContext
      |
      +-- Query
      +-- Add
      +-- Update
      +-- SaveChangesAsync
```

Le `DbContext` n'est pas conçu pour être utilisé simultanément par plusieurs threads.

---

# 24. Une façon simple de choisir le lifetime

Commencer par poser trois questions :

### Question 1

Le service doit-il être différent à chaque résolution ?

```text
Oui -> Transient
```

### Question 2

Doit-il être partagé pendant une requête/scope ?

```text
Oui -> Scoped
```

### Question 3

Doit-il être partagé pendant toute la durée de vie de l'application ?

```text
Oui -> Singleton
```

Mais pour un singleton, vérifier toujours :

```text
État partagé ?
Thread-safe ?
Dépendances compatibles ?
```

---

# 25. Règle mentale globale

Quand tu vois :

```csharp
public UserService(IUserRepository repository)
```

lis-le comme :

> « UserService ne sait pas comment créer son repository. Il demande simplement le contrat dont il a besoin. »

Puis :

```csharp
builder.Services.AddScoped<IUserRepository, UserRepository>();
```

signifie :

> « Si quelqu'un demande IUserRepository, donne-lui une instance de UserRepository avec cette durée de vie. »

Et :

```csharp
builder.Services.AddScoped<UserService>();
```

signifie :

> « Le conteneur sait maintenant comment construire UserService. »

---

# 26. À retenir

Les points essentiels :

1. **DI = fournir les dépendances depuis l'extérieur.**
2. **Le constructeur est généralement le meilleur endroit pour les injecter.**
3. **ASP.NET Core possède un conteneur DI intégré.**
4. `AddTransient` = nouvelle instance à chaque résolution.
5. `AddScoped` = une instance par scope, généralement une requête HTTP.
6. `AddSingleton` = une instance pendant toute la vie de l'application.
7. Éviter `Singleton -> Scoped`.
8. `IServiceProvider` ne doit pas devenir un Service Locator.
9. La DI améliore le découplage et la testabilité.
10. Le conteneur construit un **graphe de dépendances**.
11. Une classe avec énormément de dépendances peut avoir trop de responsabilités.
12. Une interface n'est pas obligatoire pour utiliser la DI.

---

# Questions d'entretien

### 1. Qu'est-ce que la Dependency Injection ?

La DI est un mécanisme permettant de fournir à une classe ses dépendances depuis l'extérieur plutôt que de les créer elle-même.

### 2. Quelle différence entre IoC et DI ?

IoC est le principe d'inversion du contrôle. DI est une technique permettant notamment de réaliser cette inversion.

### 3. Quelle différence entre Transient, Scoped et Singleton ?

- Transient : nouvelle instance à chaque résolution.
- Scoped : une instance par scope.
- Singleton : une instance pendant toute la durée de vie de l'application.

### 4. Pourquoi utiliser Constructor Injection ?

Parce que les dépendances sont explicites et que l'objet ne peut pas être créé sans recevoir les éléments nécessaires à son fonctionnement.

### 5. Pourquoi Singleton -> Scoped est problématique ?

Parce qu'un singleton peut capturer une dépendance scoped et la conserver au-delà du scope auquel elle appartient.

### 6. Pourquoi la DI facilite-t-elle les tests ?

Parce qu'on peut remplacer une dépendance réelle par une implémentation de test ou un mock.

### 7. Est-ce qu'une interface est obligatoire avec la DI ?

Non. On peut enregistrer et injecter directement une classe concrète.

### 8. Qu'est-ce qu'un Dependency Graph ?

C'est l'ensemble des relations entre les objets nécessaires pour construire un service.

---

# Phrase à retenir

> **Avec la DI, une classe demande ce dont elle a besoin ; elle ne décide pas comment le créer.**
