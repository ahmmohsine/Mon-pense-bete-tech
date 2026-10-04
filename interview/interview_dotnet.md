# Entretien — .NET

Cette fiche regroupe les notions .NET importantes à maîtriser pour un entretien de développeuse C#/.NET.

L'objectif est de comprendre l'écosystème derrière le code :

```text
C#
 ↓
.NET
 ↓
Runtime
 ↓
Application
```

Il faut pouvoir expliquer non seulement **comment utiliser une fonctionnalité**, mais aussi **pourquoi elle existe et ce qu'elle fait derrière**.

---

# 1. Qu'est-ce que .NET ?

.NET est une plateforme de développement de Microsoft.

Elle fournit notamment :

```text
Runtime
Libraries
SDK
CLI
ASP.NET Core
Entity Framework Core
outils de développement
```

C# est un langage qui peut être utilisé avec .NET.

Mental model :

```text
C#
→ langage

.NET
→ plateforme/runtime + bibliothèques + outils

ASP.NET Core
→ framework web basé sur .NET
```

### Question d'entretien

**Quelle différence entre C# et .NET ?**

> C# est un langage de programmation. .NET est la plateforme qui fournit le runtime, les bibliothèques et les outils permettant notamment d'exécuter des applications C#.

---

# 2. .NET moderne vs .NET Framework

Il faut distinguer :

```text
.NET Framework
```

et :

```text
.NET moderne
```

Le .NET Framework historique est principalement associé à Windows.

Le .NET moderne est multiplateforme :

```text
Windows
Linux
macOS
```

Les versions modernes utilisent par exemple :

```text
.NET 6
.NET 7
.NET 8
.NET 9
.NET 10
```

Pour un nouveau projet, on travaille généralement avec le .NET moderne plutôt qu'avec l'ancien .NET Framework.

---

# 3. Le CLR

Le CLR signifie :

```text
Common Language Runtime
```

C'est le runtime qui exécute le code managé .NET.

Il s'occupe notamment de nombreux services comme :

```text
exécution du code
gestion mémoire
garbage collection
exceptions
type system
interopérabilité
```

Mental model :

```text
application
    ↓
CLR
    ↓
OS / machine
```

---

# 4. CIL / IL

Le code C# n'est pas directement transformé en instructions machine spécifiques au processeur lors de l'écriture du programme.

Le compilateur produit notamment de l'IL :

```text
C# source
   ↓
compilation
   ↓
IL / CIL
   ↓
runtime .NET
   ↓
code machine
```

IL signifie :

```text
Intermediate Language
```

Le runtime peut ensuite compiler le code pour la machine cible.

---

# 5. JIT

JIT signifie :

```text
Just-In-Time compilation
```

Le runtime compile du code IL vers du code machine utilisable par le processeur.

Mental model simplifié :

```text
C#
 ↓
IL
 ↓
JIT
 ↓
machine code
```

Le JIT intervient pendant l'exécution.

---

# 6. AOT

AOT signifie :

```text
Ahead-Of-Time
```

Une partie de la compilation vers du code natif est réalisée avant l'exécution.

Mental model :

```text
JIT
→ compilation pendant l'exécution

AOT
→ compilation anticipée
```

L'AOT peut notamment être intéressant dans certains scénarios où le temps de démarrage, la taille ou le comportement d'exécution sont importants.

---

# 7. Garbage Collector

Le Garbage Collector, ou GC, gère automatiquement une partie de la mémoire des objets managés.

Mental model :

```text
new object
   ↓
mémoire managée
   ↓
objet devient inaccessible
   ↓
GC peut récupérer la mémoire
```

Important :

Le GC ne signifie pas :

```text
"je peux ignorer toute gestion de ressources"
```

Certaines ressources ne sont pas simplement de la mémoire managée :

```text
fichiers
sockets
connexions
handles OS
```

Pour celles-ci, on utilise notamment `IDisposable`.

---

# 8. `IDisposable`

`IDisposable` indique qu'un objet possède une ressource qui doit être libérée explicitement.

Exemple :

```csharp
public class MyResource : IDisposable
{
    public void Dispose()
    {
        // libération de ressources
    }
}
```

Utilisation :

```csharp
using var resource = new MyResource();
```

À la sortie du scope, `Dispose()` est appelé.

Mental model :

```text
GC
→ mémoire managée

Dispose
→ libération déterministe d'une ressource
```

---

# 9. `using`

Exemple :

```csharp
using var connection = new SqlConnection(connectionString);
```

Le compilateur garantit que `Dispose()` sera appelé lorsque l'exécution quitte le scope approprié, y compris lorsqu'une exception survient.

On peut retenir :

```text
using
→ acquisition
→ utilisation
→ libération
```

---

# 10. Managed code vs unmanaged code

Le code managé est exécuté sous le contrôle du runtime .NET.

Le code unmanaged fonctionne en dehors du modèle de gestion mémoire du runtime.

Exemples de ressources unmanaged :

```text
handles système
API natives
certaines bibliothèques natives
```

C'est une raison importante pour laquelle `IDisposable` existe.

---

# 11. Assembly

Une assembly est une unité compilée de déploiement et de versionnement .NET.

Elle peut être :

```text
.dll
.exe
```

Elle contient notamment :

```text
IL
metadata
manifest
```

Mental model :

```text
projet
 ↓
build
 ↓
assembly
```

---

# 12. Namespace

Un namespace organise les types et évite notamment les collisions de noms.

Exemple :

```csharp
namespace MyApp.Services
{
    public class UserService
    {
    }
}
```

Puis :

```csharp
using MyApp.Services;
```

permet d'utiliser le type sans répéter son nom complet.

---

# 13. SDK vs Runtime

### Runtime

Permet principalement d'exécuter une application .NET.

### SDK

Contient les outils nécessaires pour développer et construire des applications.

Par exemple :

```bash
dotnet build
dotnet run
dotnet test
dotnet publish
```

nécessitent l'environnement de développement fourni par le SDK.

Mental model :

```text
Runtime
→ exécuter

SDK
→ développer / compiler / tester / publier
```

---

# 14. Le fichier `.csproj`

Le fichier projet décrit notamment :

```xml
<TargetFramework>net10.0</TargetFramework>
```

et les dépendances du projet.

Exemple :

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">

  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

</Project>
```

Il peut également contenir :

```text
PackageReference
ProjectReference
build settings
properties
```

---

# 15. NuGet

NuGet est le gestionnaire de packages de l'écosystème .NET.

Exemple :

```xml
<PackageReference Include="Microsoft.EntityFrameworkCore" Version="10.0.0" />
```

Le package est ensuite disponible dans le projet.

Commandes utiles :

```bash
dotnet add package NomDuPackage
dotnet remove package NomDuPackage
dotnet restore
dotnet list package
```

Mental model :

```text
NuGet
→ packages / dépendances .NET
```

---

# 16. `dotnet restore`

Cette commande restaure les dépendances nécessaires au projet.

```bash
dotnet restore
```

Elle s'appuie notamment sur :

```text
.csproj
NuGet sources
packages
```

---

# 17. `dotnet build`

Compile le projet.

```bash
dotnet build
```

Mental model :

```text
source code
   +
dependencies
   ↓
build
   ↓
binaries
```

Le build permet notamment de détecter les erreurs de compilation.

---

# 18. `dotnet run`

Compile et lance généralement l'application.

```bash
dotnet run
```

C'est particulièrement pratique pendant le développement.

---

# 19. `dotnet test`

Lance les tests du projet.

```bash
dotnet test
```

Dans un pipeline CI/CD, on peut avoir :

```text
restore
 ↓
build
 ↓
test
 ↓
publish
 ↓
deploy
```

---

# 20. `dotnet publish`

Prépare les fichiers nécessaires au déploiement.

```bash
dotnet publish
```

Le résultat est destiné à être déployé sur un environnement cible.

Mental model :

```text
build
→ compiler

publish
→ préparer l'application pour son déploiement
```

---

# 21. Configuration .NET

ASP.NET Core et .NET utilisent un système de configuration flexible.

Sources fréquentes :

```text
appsettings.json
appsettings.{Environment}.json
variables d'environnement
ligne de commande
secrets
providers personnalisés
```

Exemple :

```json
{
  "ConnectionStrings": {
    "Default": "..."
  }
}
```

---

# 22. Environnements

ASP.NET Core utilise notamment :

```text
Development
Staging
Production
```

L'environnement permet d'adapter la configuration et certains comportements.

Par exemple :

```text
Development
→ diagnostics détaillés

Production
→ configuration sécurisée et comportement adapté au déploiement
```

Il ne faut pas exposer des informations sensibles en production.

---

# 23. Dependency Injection

La Dependency Injection est un mécanisme central de .NET.

Au lieu de créer directement une dépendance :

```csharp
var service = new EmailService();
```

une classe reçoit sa dépendance :

```csharp
public class UserService
{
    private readonly IEmailService _emailService;

    public UserService(IEmailService emailService)
    {
        _emailService = emailService;
    }
}
```

Puis on configure l'implémentation :

```csharp
builder.Services.AddScoped<
    IEmailService,
    EmailService>();
```

Mental model :

```text
classe
 ↓
demande une abstraction
 ↓
DI container
 ↓
fournit l'implémentation
```

---

# 24. Les trois lifetimes

.NET fournit principalement :

```csharp
AddTransient
AddScoped
AddSingleton
```

## Transient

```csharp
services.AddTransient<IService, Service>();
```

Une nouvelle instance est créée chaque fois que le service est demandé.

```text
demande
 ↓
nouvelle instance
```

## Scoped

```csharp
services.AddScoped<IService, Service>();
```

Une instance est généralement créée par scope.

Dans une application ASP.NET Core web, le scope correspond typiquement à une requête HTTP.

```text
Request
   ↓
Scope
   ↓
même instance du service dans ce scope
```

## Singleton

```csharp
services.AddSingleton<IService, Service>();
```

Une instance est partagée pendant la durée de vie du conteneur d'application.

---

# 25. Piège des lifetimes

Un singleton ne doit pas capturer aveuglément une dépendance scoped.

Exemple conceptuellement problématique :

```text
Singleton
   ↓
Scoped
```

Pourquoi ?

Parce que le singleton vit plus longtemps que le scope.

Cela peut provoquer des problèmes de durée de vie et de données partagées.

À l'inverse, la conception des dépendances doit respecter les lifetimes.

---

# 26. Middleware

Dans ASP.NET Core, le middleware forme un pipeline.

Mental model :

```text
Request
   ↓
Middleware A
   ↓
Middleware B
   ↓
Endpoint
   ↓
Response
```

Un middleware peut :

```text
inspecter la requête
modifier la requête
appeler le middleware suivant
modifier la réponse
court-circuiter le pipeline
```

Exemple :

```csharp
app.Use(async (context, next) =>
{
    // avant
    await next();
    // après
});
```

---

# 27. Pourquoi l'ordre des middleware est important ?

Exemple :

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

L'ordre a une importance.

La pipeline est exécutée dans l'ordre de déclaration.

Une mauvaise organisation peut donc provoquer des comportements inattendus.

Mental model :

```text
ordre du code
    ↓
ordre du pipeline
    ↓
ordre des effets
```

---

# 28. Logging

.NET fournit `ILogger<T>`.

Exemple :

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

Puis :

```csharp
_logger.LogInformation(
    "User {UserId} created",
    userId);
```

Le logging permet notamment :

```text
diagnostic
monitoring
audit technique
analyse des erreurs
```

---

# 29. Pourquoi utiliser des logs structurés ?

Préférer :

```csharp
_logger.LogInformation(
    "User {UserId} created",
    userId);
```

à :

```csharp
_logger.LogInformation(
    $"User {userId} created");
```

Le logging structuré permet au système de logging de conserver des propriétés exploitables.

Exemple :

```text
Message
UserId = 42
```

Cela facilite notamment la recherche et l'analyse dans les systèmes de monitoring.

---

# 30. Options Pattern

Pour mapper une configuration vers une classe :

```csharp
public class JwtOptions
{
    public string Issuer { get; set; } = "";
    public string Audience { get; set; } = "";
}
```

On peut utiliser :

```csharp
builder.Services.Configure<JwtOptions>(
    builder.Configuration.GetSection("Jwt"));
```

Puis injecter :

```csharp
IOptions<JwtOptions>
```

Mental model :

```text
appsettings
    ↓
section
    ↓
classe Options
    ↓
injection
```

---

# 31. `IOptions`, `IOptionsSnapshot`, `IOptionsMonitor`

### `IOptions<T>`

Utilisé pour accéder à une configuration typée.

### `IOptionsSnapshot<T>`

Permet notamment de prendre en compte les changements de configuration selon le scope.

### `IOptionsMonitor<T>`

Permet notamment de surveiller les changements et d'obtenir la valeur courante.

Mental model simplifié :

```text
IOptions
→ valeur configurée

IOptionsSnapshot
→ valeur liée au scope

IOptionsMonitor
→ surveillance / valeur courante
```

---

# 32. CancellationToken

`CancellationToken` permet de propager une demande d'annulation.

Exemple :

```csharp
public async Task ProcessAsync(
    CancellationToken cancellationToken)
{
    await repository.GetDataAsync(
        cancellationToken);
}
```

On peut ensuite vérifier :

```csharp
cancellationToken.ThrowIfCancellationRequested();
```

Mental model :

```text
request cancelled
      ↓
token
      ↓
service
      ↓
repository
      ↓
operation
```

L'annulation est coopérative.

Le token ne tue pas brutalement une opération.

---

# 33. Pourquoi transmettre le CancellationToken ?

Imagine une requête HTTP qui est annulée par le client.

Si le serveur continue inutilement :

```text
HTTP request annulée
       ↓
application continue
       ↓
database continue
       ↓
travail inutile
```

En transmettant le token :

```text
HTTP cancellation
       ↓
CancellationToken
       ↓
service
       ↓
database call
```

l'opération peut s'arrêter proprement lorsqu'elle le permet.

---

# 34. Dependency Injection et testabilité

Une classe qui crée directement ses dépendances est difficile à tester.

Exemple :

```csharp
public class UserService
{
    private readonly EmailService _email;

    public UserService()
    {
        _email = new EmailService();
    }
}
```

Il est difficile de remplacer `EmailService`.

Avec une abstraction :

```csharp
public UserService(IEmailService email)
{
    _email = email;
}
```

on peut injecter un mock/fake dans un test.

Mental model :

```text
DI
→ faible couplage
→ remplacement des dépendances
→ meilleure testabilité
```

---

# 35. `IHost` / Generic Host

Le Host fournit une infrastructure commune pour les applications .NET.

Il peut gérer notamment :

```text
Dependency Injection
Configuration
Logging
Lifetime de l'application
Hosted services
```

ASP.NET Core construit son application autour de ce modèle d'hébergement.

---

# 36. Hosted Services

Un service en arrière-plan peut implémenter notamment :

```csharp
BackgroundService
```

Exemple :

```csharp
public class Worker : BackgroundService
{
    protected override async Task ExecuteAsync(
        CancellationToken stoppingToken)
    {
        while (!stoppingToken.IsCancellationRequested)
        {
            await DoWorkAsync(stoppingToken);
        }
    }
}
```

Cas d'utilisation :

```text
traitement périodique
queue
maintenance
synchronisation
tâche longue
```

---

# 37. Configuration vs Secrets

Une mauvaise pratique serait de mettre des secrets directement dans Git :

```json
{
  "ConnectionString":
    "Server=...;Password=secret"
}
```

Les secrets doivent être gérés par un mécanisme adapté.

Selon l'environnement :

```text
User Secrets
Environment Variables
Azure Key Vault
Managed Identity
secret stores
```

Mental model :

```text
configuration
≠
secret management
```

---

# 38. Architecture typique d'une application .NET

Une API .NET peut être organisée par exemple ainsi :

```text
API
 ↓
Application
 ↓
Domain
 ↓
Infrastructure
```

Avec une séparation des responsabilités.

Exemple :

```text
API
→ HTTP / controllers / endpoints

Application
→ cas d'utilisation

Domain
→ règles métier

Infrastructure
→ EF Core / database / services externes
```

L'objectif est notamment d'éviter que le métier dépende directement de détails techniques.

---

# 39. Build configuration

Un projet peut avoir plusieurs configurations :

```text
Debug
Release
```

On peut également avoir des environnements de déploiement distincts :

```text
Development
Staging
Production
```

Il faut distinguer :

```text
Build configuration
```

et :

```text
Application environment
```

Ce ne sont pas exactement la même chose.

---

# 40. CI/CD

CI signifie :

```text
Continuous Integration
```

CD signifie selon le contexte :

```text
Continuous Delivery
```

ou :

```text
Continuous Deployment
```

Pipeline typique :

```text
push
 ↓
restore
 ↓
build
 ↓
test
 ↓
security checks
 ↓
publish
 ↓
deploy
```

Avec GitHub Actions, on peut automatiser ce processus.

---

# 41. Exemple de pipeline .NET

```yaml
- name: Restore
  run: dotnet restore

- name: Build
  run: dotnet build --no-restore

- name: Test
  run: dotnet test --no-build

- name: Publish
  run: dotnet publish -c Release
```

L'idée importante en entretien est de comprendre le rôle de chaque étape.

---

# 42. Questions d'entretien

### Qu'est-ce que .NET ?

> Une plateforme de développement qui fournit notamment un runtime, des bibliothèques et des outils pour construire et exécuter des applications.

### Quelle différence entre C# et .NET ?

> C# est le langage. .NET est la plateforme d'exécution et l'écosystème qui l'accompagne.

### Qu'est-ce que le CLR ?

> Le Common Language Runtime est le runtime qui exécute le code managé .NET et fournit notamment la gestion mémoire, le GC et d'autres services d'exécution.

### Qu'est-ce que le JIT ?

> Le Just-In-Time compiler transforme du code intermédiaire en code machine pendant l'exécution.

### Différence entre SDK et Runtime ?

> Le Runtime sert principalement à exécuter une application, tandis que le SDK fournit les outils nécessaires au développement, au build, aux tests et à la publication.

### Qu'est-ce que NuGet ?

> Le gestionnaire de packages de l'écosystème .NET.

### Différence entre `build` et `publish` ?

> `build` compile le projet. `publish` prépare les artefacts nécessaires à son déploiement.

### Quels sont les lifetimes DI ?

> Transient, Scoped et Singleton.

### Différence entre Scoped et Singleton ?

> Scoped crée généralement une instance par scope, alors que Singleton partage une instance pendant toute la durée de vie du conteneur.

### Pourquoi l'ordre des middleware est important ?

> Parce que les middleware forment un pipeline exécuté dans l'ordre de déclaration.

### Pourquoi utiliser `CancellationToken` ?

> Pour permettre une annulation coopérative d'une opération asynchrone et propager cette demande dans les différentes couches.

### Pourquoi utiliser `IOptions<T>` ?

> Pour accéder à une configuration fortement typée plutôt que de manipuler directement des chaînes de configuration.

### Pourquoi utiliser `ILogger<T>` ?

> Pour produire des logs structurés et exploitables afin de diagnostiquer et monitorer l'application.

---

# 43. Questions pièges

## "Le GC libère toutes les ressources"

Faux.

Le GC gère la mémoire managée.

Les ressources externes doivent généralement être libérées explicitement via `Dispose()` / `IAsyncDisposable`.

---

## "Scoped signifie une instance par utilisateur"

Pas exactement.

Dans ASP.NET Core, le scoped correspond généralement à un **scope**, typiquement une requête HTTP.

Ce n'est pas automatiquement une instance par utilisateur.

---

## "Singleton est toujours plus performant"

Pas nécessairement.

Un singleton implique un état partagé et une durée de vie longue.

Il faut tenir compte :

```text
thread safety
état mutable
mémoire
concurrence
dependencies
```

---

## "async crée un thread"

Faux.

`async/await` est un modèle d'asynchronisme.

Pour les opérations I/O, il permet notamment de ne pas bloquer inutilement un thread pendant l'attente.

---

## "Middleware et service sont la même chose"

Non.

Un middleware participe au pipeline HTTP.

Un service est généralement une abstraction applicative injectée via la DI.

Un middleware peut utiliser des services.

---

# 44. Mini simulation d'entretien

### Recruteur

**Explique-moi ce qui se passe lorsqu'une requête arrive dans une API ASP.NET Core.**

### Réponse attendue

> La requête entre dans le pipeline HTTP ASP.NET Core. Elle traverse les middleware dans l'ordre de configuration. Selon la configuration, l'authentification et l'autorisation sont exécutées, puis le routage permet de déterminer l'endpoint correspondant. Le framework effectue ensuite notamment le model binding et la validation, puis le code applicatif est exécuté. La réponse remonte ensuite à travers le pipeline avant d'être renvoyée au client.

---

### Recruteur

**Pourquoi utiliser la Dependency Injection ?**

### Réponse attendue

> Pour découpler les classes de leurs implémentations concrètes. Une classe dépend d'une abstraction et le conteneur DI fournit l'implémentation. Cela facilite notamment la maintenance, le remplacement des composants et les tests.

---

### Recruteur

**Un Scoped peut-il être injecté dans un Singleton ?**

### Réponse attendue

> Il faut éviter de capturer directement une dépendance scoped dans un singleton, car les durées de vie sont incompatibles. Le singleton peut vivre pendant toute l'application alors que le scoped est lié à un scope, typiquement une requête HTTP.

---

### Recruteur

**Pourquoi utiliser `CancellationToken` dans une API ?**

### Réponse attendue

> Pour propager une demande d'annulation depuis la requête jusqu'aux opérations asynchrones sous-jacentes. Cela permet d'éviter de continuer inutilement un travail lorsque l'opération n'est plus nécessaire.

---

# 45. Checklist avant entretien .NET

Je dois être capable d'expliquer sans chercher :

```text
[ ] C# vs .NET
[ ] CLR
[ ] IL / CIL
[ ] JIT
[ ] AOT
[ ] Garbage Collector
[ ] IDisposable
[ ] using
[ ] Assembly
[ ] SDK vs Runtime
[ ] csproj
[ ] NuGet
[ ] restore / build / test / publish
[ ] configuration
[ ] environnements
[ ] Dependency Injection
[ ] Transient / Scoped / Singleton
[ ] Middleware
[ ] Logging
[ ] Options Pattern
[ ] CancellationToken
[ ] Hosted Services
[ ] Generic Host
[ ] CI/CD
```

# À retenir

```text
.NET
→ plateforme

CLR
→ runtime

C#
→ langage

IL
→ code intermédiaire

JIT
→ compilation pendant l'exécution

GC
→ gestion de la mémoire managée

IDisposable
→ libération déterministe de ressources

SDK
→ développer / compiler / tester / publier

Runtime
→ exécuter

NuGet
→ packages

DI
→ injection des dépendances

Transient
→ nouvelle instance par résolution

Scoped
→ instance par scope

Singleton
→ instance partagée

Middleware
→ pipeline HTTP

ILogger
→ logs

IOptions
→ configuration typée

CancellationToken
→ annulation coopérative

HostedService
→ travail en arrière-plan
```

## Phrase à mémoriser

> **.NET fournit le runtime et l'écosystème autour de C# ; dans une application moderne, je dois surtout comprendre comment le runtime, la DI, la configuration, le middleware, le logging, les lifetimes et l'asynchronisme travaillent ensemble pour faire fonctionner l'application.**
