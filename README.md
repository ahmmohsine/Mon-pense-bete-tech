# Base de connaissances & mémos techniques

Dépôt personnel de rappels, explications et fiches de révision autour de **C#, .NET, ASP.NET Core, EF Core, architecture, sécurité, Azure et Angular**.

L'objectif est de disposer d'un pense-bête pratique pour :

- réviser les concepts techniques ;
- retrouver rapidement une notion ;
- préparer des entretiens .NET ;
- comprendre les mécanismes derrière le code ;
- conserver des exemples et bonnes pratiques.

---

## C#

Notions fondamentales et avancées du langage.

- [C# — Index](csharp/README.md)
- [Collections](csharp/collections.md)
- [Generics](csharp/generics.md)
- [LINQ](csharp/linq.md)
- [Exceptions](csharp/exceptions.md)
- [Delegates & Events](csharp/delegates-events.md)
- [Records](csharp/records.md)
- [Nullable Reference Types](csharp/nullable-reference-types.md)
- [Pattern Matching](csharp/pattern-matching.md)
- [Async / Await](csharp/async-await.md)

---

## .NET

Concepts fondamentaux de la plateforme .NET.

- [.NET — Index](dotnet/README.md)
- [Dependency Injection](dotnet/dependency-injection.md)
- [Middleware](dotnet/middleware.md)
- [Configuration](dotnet/configuration.md)
- [Logging](dotnet/logging.md)
- [Options Pattern](dotnet/options-pattern.md)
- [CancellationToken](dotnet/cancellation-token.md)

---

## ASP.NET Core

Développement d'APIs et d'applications Web.

- [ASP.NET Core — Index](aspnet-core/README.md)
- [Web API](aspnet-core/web-api.md)
- [Routing](aspnet-core/routing.md)
- [Model Binding](aspnet-core/model-binding.md)
- [Validation](aspnet-core/validation.md)
- [Filters](aspnet-core/filters.md)
- [Middleware](aspnet-core/middleware.md)
- [Authentication](aspnet-core/authentication.md)
- [Authorization](aspnet-core/authorization.md)

---

## Entity Framework Core

Accès aux données et persistance.

- [EF Core — Index](ef-core/README.md)
- [DbContext](ef-core/dbcontext.md)
- [Tracking](ef-core/tracking.md)
- [Relationships](ef-core/relationships.md)
- [Migrations](ef-core/migrations.md)
- [Querying](ef-core/querying.md)
- [Performance](ef-core/performance.md)
- [Seeding](ef-core/seeding.md)

---

## Architecture

Conception et organisation des applications .NET.

- [Architecture — Index](architecture/README.md)
- [Clean Architecture](architecture/clean-architecture.md)
- [CQRS](architecture/cqrs.md)
- [Repository Pattern](architecture/repository.md)
- [Unit of Work](architecture/unit-of-work.md)
- [Dependency Inversion](architecture/dependency-inversion.md)

---

## Sécurité des APIs

Sécurisation des applications et APIs.

- [Sécurité des APIs — Index](api-security/README.md)
- [Risques OWASP API Top 10](api-security/01-risques-owasp.md)
- [DTOs & Rate Limiting](api-security/02-dto-et-rate-limiting.md)
- [Authentication](api-security/authentication.md)
- [Authorization](api-security/authorization.md)
- [JWT](api-security/jwt.md)

---

## Azure

Déploiement et services cloud Microsoft Azure.

- [Azure — Index](azure/README.md)
- [Azure App Service](azure/app-service.md)
- [Azure SQL](azure/azure-sql.md)
- [Azure Key Vault](azure/key-vault.md)
- [Managed Identity](azure/managed-identity.md)
- [GitHub Actions](azure/github-actions.md)

---

## Angular

Notions Angular utiles pour le développement Full Stack.

- [Angular — Index](angular/README.md)
- [Components](angular/components.md)
- [Signals](angular/signals.md)
- [Inputs & Outputs](angular/inputs-outputs.md)
- [Services](angular/services.md)
- [RxJS](angular/rxjs.md)

---

## Préparation aux entretiens

Fiches spécialement orientées entretien technique.

### C#

- [Entretien C#](interview/csharp.md)

### .NET

- [Entretien .NET](interview/dotnet.md)

### ASP.NET Core

- [Entretien ASP.NET Core](interview/aspnet-core.md)

### EF Core

- [Entretien EF Core](interview/ef-core.md)

### Architecture

- [Entretien Architecture](interview/architecture.md)

### Sécurité

- [Entretien Sécurité](interview/security.md)

---

## Parcours de révision conseillé

Pour préparer un entretien de développeuse .NET, un ordre efficace est :

```text
C#
 ↓
.NET
 ↓
ASP.NET Core
 ↓
EF Core
 ↓
Architecture
 ↓
Sécurité
 ↓
Azure
 ↓
Angular
```

Puis terminer par :

```text
Interview
```

pour vérifier que les notions peuvent être expliquées oralement.

---

## Principe de ce pense-bête

Une notion ne devrait pas seulement être mémorisée comme une définition.

Pour chaque concept, essayer de répondre à :

```text
Qu'est-ce que c'est ?
        ↓
Comment ça fonctionne ?
        ↓
Pourquoi l'utiliser ?
        ↓
Quel problème résout-il ?
        ↓
Quels sont ses pièges ?
        ↓
Quand ne faut-il pas l'utiliser ?
```

## Objectif

> Comprendre suffisamment les concepts pour pouvoir expliquer clairement ses choix techniques et raisonner sur le code, plutôt que simplement mémoriser de la syntaxe.
