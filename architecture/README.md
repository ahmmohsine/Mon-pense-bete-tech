# Architecture — Vue d'ensemble

Cette section regroupe les principes fondamentaux d'architecture utilisés dans les applications .NET.

L'objectif n'est pas seulement de connaître des patterns ou de reproduire une structure de dossiers, mais de comprendre :

* comment organiser les responsabilités ;
* comment contrôler les dépendances ;
* comment protéger le métier des détails techniques ;
* comment rendre une application testable et maintenable ;
* quand une abstraction apporte réellement de la valeur.

---

## Fiches

* [Clean Architecture](clean-architecture.md)
* [CQRS](cqrs.md)
* [Dependency Inversion](dependency-inversion.md)
* [Repository Pattern](repository.md)
* [Unit of Work](unit-of-work.md)

---

## Ordre conseillé

Pour comprendre progressivement l'architecture d'une application .NET :

1. [Dependency Inversion](dependency-inversion.md)
2. [Clean Architecture](clean-architecture.md)
3. [Repository Pattern](repository.md)
4. [Unit of Work](unit-of-work.md)
5. [CQRS](cqrs.md)

L'ordre commence par le principe fondamental des dépendances avant d'aborder les patterns et les architectures qui s'appuient dessus.

---

## Structure d'une fiche

Chaque fiche doit essayer de répondre à ces questions :

1. Qu'est-ce que c'est ?
2. Quel problème cela résout ?
3. Pourquoi l'utiliser ?
4. Comment cela fonctionne derrière la syntaxe ?
5. Comment l'utiliser correctement ?
6. Quelles sont les erreurs fréquentes ?
7. Quelle différence avec les mécanismes proches ?
8. Quels sont les avantages et les inconvénients ?
9. Quand ne faut-il pas l'utiliser ?
10. Quelle règle mentale permet de s'en souvenir ?
11. Que peut-on demander en entretien ?

---

## Mental model

Une application .NET peut être organisée autour de plusieurs responsabilités :

```text
Application
    |
    +-- API / Presentation
    |    -> expose l'application au monde extérieur
    |
    +-- Application
    |    -> orchestre les cas d'utilisation
    |
    +-- Domain
    |    -> contient les règles métier
    |
    +-- Infrastructure
         -> contient les détails techniques
```

Une représentation simplifiée :

```text
Presentation
      |
      v
Application
      |
      v
Domain

Infrastructure
      |
      +---- implémente les abstractions nécessaires
```

L'idée essentielle est de contrôler la direction des dépendances.

---

## Responsabilités des couches

### Presentation / API

Comprend principalement le monde HTTP :

```text
Request
Response
Routing
Status codes
Model binding
Authentication
Authorization
```

Son rôle est notamment de recevoir une requête, appeler le cas d'utilisation approprié et transformer le résultat en réponse HTTP.

### Application

Orchestre les cas d'utilisation :

```text
CreateOrder
GetOrder
CancelOrder
UpdateCustomer
RegisterUser
```

Elle coordonne les différentes opérations nécessaires à l'exécution d'un cas d'utilisation.

### Domain

Contient les concepts et règles métier importantes :

```text
Entities
Value Objects
Domain Rules
Domain Events
Business invariants
```

Le Domain ne devrait pas dépendre inutilement d'ASP.NET Core, d'EF Core ou d'une technologie externe.

### Infrastructure

Contient les détails techniques :

```text
EF Core
SQL Server
Email
File system
Azure Blob Storage
Redis
External APIs
Message brokers
```

Infrastructure fournit les implémentations concrètes nécessaires à l'application.

---

## Dependency Inversion

Le principe fondamental est :

> Le cœur de l'application ne doit pas être fortement couplé aux détails techniques.

Sans abstraction :

```text
Application
     |
     v
EF Core
     |
     v
SQL Server
```

Avec une abstraction :

```text
Application
     |
     v
IOrderRepository
     ^
     |
OrderRepository
     |
     v
EF Core
```

L'application dépend de l'abstraction :

```csharp
IOrderRepository
```

et Infrastructure fournit l'implémentation :

```csharp
OrderRepository
```

L'injection de dépendances permet ensuite de connecter les deux :

```csharp
builder.Services.AddScoped<IOrderRepository, OrderRepository>();
```

---

## Architecture et responsabilités

Une bonne architecture cherche principalement à répondre à une question :

> Qui est responsable de quoi ?

Par exemple :

```text
API
    -> HTTP

Application
    -> cas d'utilisation

Domain
    -> règles métier

Infrastructure
    -> détails techniques
```

Le but n'est pas de créer le plus de projets ou de dossiers possible.

Le but est de rendre les responsabilités claires et les dépendances maîtrisées.

---

## Architecture et testabilité

Une séparation claire des responsabilités facilite les tests.

Par exemple :

```text
CreateOrderHandler
        |
        v
IOrderRepository
```

Pendant un test, on peut fournir une implémentation simulée :

```text
Application
     |
     v
Mock IOrderRepository
```

plutôt que de dépendre systématiquement d'une base SQL Server réelle.

---

## Architecture et changement de technologie

Une bonne abstraction peut réduire le couplage.

Exemple :

```text
Application
      |
      v
IEmailSender
      ^
      |
SmtpEmailSender
```

Demain, une autre implémentation peut être utilisée :

```text
Application
      |
      v
IEmailSender
      ^
      |
AzureEmailSender
```

L'abstraction ne rend pas automatiquement le changement gratuit.

Elle permet surtout de limiter le couplage lorsqu'elle correspond à une vraie frontière de responsabilité.

---

## Repository Pattern

EF Core fournit déjà beaucoup de fonctionnalités proches du :

```text
Repository
Unit of Work
```

Par exemple :

```csharp
_context.Products
```

fournit déjà un mécanisme d'accès aux données.

Et :

```csharp
_context.SaveChangesAsync();
```

regroupe les modifications à persister.

Il ne faut donc pas créer automatiquement :

```text
IProductRepository
ProductRepository
IProductUnitOfWork
ProductUnitOfWork
```

pour chaque entité sans besoin réel.

Le Repository Pattern peut être utile lorsqu'il apporte une véritable abstraction métier ou technique, mais ce n'est pas une obligation.

---

## Architecture ne signifie pas beaucoup de projets

Une architecture avec :

```text
20 projets
```

n'est pas nécessairement meilleure qu'une architecture avec :

```text
3 projets
```

La vraie question est :

> Les responsabilités et les dépendances sont-elles cohérentes ?

Le nombre de projets n'est pas une mesure de la qualité architecturale.

---

## Attention à l'overengineering

Une architecture peut devenir inutilement complexe.

Pour un petit CRUD, on pourrait théoriquement avoir :

```text
Controller
Application Service
Command
CommandHandler
Repository
UnitOfWork
Factory
Domain Service
Mapper
Mediator
...
```

Mais cette complexité peut parfois apporter moins de valeur qu'elle n'en coûte.

La bonne architecture est celle qui :

* protège les règles importantes ;
* limite le couplage ;
* facilite les tests ;
* facilite les changements ;
* reste compréhensible ;
* n'est pas plus complexe que nécessaire.

---

## Checklist architecture

Avant de considérer une architecture comme saine :

* [ ] Les responsabilités sont clairement séparées.
* [ ] Le Domain ne dépend pas du HTTP.
* [ ] Le Domain ne dépend pas inutilement d'EF Core.
* [ ] Les contrôleurs restent relativement minces.
* [ ] Les DTOs séparent les contrats HTTP des entités.
* [ ] Les détails techniques restent dans Infrastructure lorsque cela est pertinent.
* [ ] Les abstractions sont utilisées lorsqu'elles apportent une vraie valeur.
* [ ] Les dépendances vont dans la bonne direction.
* [ ] L'application est testable.
* [ ] Les règles métier importantes sont protégées par le Domain.
* [ ] L'architecture n'est pas plus complexe que nécessaire.

---

## Questions d'entretien

### Qu'est-ce que la Clean Architecture ?

C'est une approche architecturale qui cherche notamment à isoler le cœur métier des détails techniques et à contrôler la direction des dépendances.

### Pourquoi utiliser Dependency Inversion ?

Pour éviter que le cœur de l'application dépende directement des implémentations techniques et pour permettre de remplacer plus facilement certaines implémentations.

### Pourquoi un controller doit-il rester mince ?

Parce qu'il doit principalement adapter HTTP vers les cas d'utilisation. La logique métier importante doit être placée ailleurs.

### Quelle est la différence entre Domain et Infrastructure ?

Le Domain contient les règles et concepts métier.

Infrastructure contient les détails techniques nécessaires pour communiquer avec l'extérieur.

### Faut-il toujours utiliser Repository Pattern avec EF Core ?

Non.

EF Core fournit déjà des mécanismes proches du Repository et du Unit of Work. Il faut ajouter une abstraction uniquement lorsqu'elle apporte une vraie valeur.

### Une architecture avec beaucoup de projets est-elle forcément meilleure ?

Non.

La qualité dépend de la séparation des responsabilités et de la direction des dépendances, pas du nombre de projets.

---

## À retenir

```text
Domain
    = règles métier

Application
    = cas d'utilisation

Infrastructure
    = détails techniques

API / Presentation
    = exposition HTTP
```

Et surtout :

```text
Le cœur de l'application
ne doit pas être prisonnier
des détails techniques.
```

## Phrase à mémoriser

> Une bonne architecture protège le métier, contrôle les dépendances et évite de mélanger les responsabilités.
