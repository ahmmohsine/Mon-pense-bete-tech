**# Préparation aux entretiens — .NET / Angular / Azure**

Cette section regroupe les questions et notions importantes à réviser pour un entretien de développeuse **\*\*C# / .NET / ASP.NET Core\*\***, avec les sujets complémentaires **\*\*EF Core, architecture, sécurité, Angular et Azure\*\***.

L'objectif n'est pas seulement de mémoriser des réponses.

Il faut être capable de :

\`\`\`text

comprendre

   ↓

expliquer

   ↓

justifier

   ↓

comparer

   ↓

appliquer

\`\`\`

**---**

**## Fiches d'entretien

Les fiches détaillées disponibles dans ce dossier :

- [Questions C#](interview_csharp.md)
- [Questions .NET](interview_dotnet.md)
- [Questions ASP.NET Core](interview_aspnet-core.md)
- [Questions EF Core](interview_ef-core.md)
- [Questions Architecture](interview_architecture.md)
- [Questions Security](interview_security.md)

---

# 1. Comment utiliser cette section ?**

Pour chaque question, essaie de répondre seule avant de regarder la réponse.

Méthode :

\`\`\`text

Question

   ↓

Réponse de mémoire

   ↓

Comparer avec la fiche

   ↓

Identifier les lacunes

   ↓

Revoir le concept

   ↓

Répondre à nouveau

\`\`\`

Une bonne réponse d'entretien doit généralement contenir :

\`\`\`text

1\. Définition

2\. Fonctionnement

3\. Exemple

4\. Pourquoi l'utiliser

5\. Limites / pièges

\`\`\`

**---**

**# 2. C# / .NET**

**## Fiche principale**

[Questions C#]\(csharp.md)

Thèmes à connaître :

\`\`\`text

Types valeur / référence

Classes

Interfaces

Héritage

Polymorphisme

Generics

LINQ

Delegates

Events

Records

Nullable Reference Types

Pattern Matching

Exceptions

Collections

async / await

\`\`\`

Questions typiques :

\- Quelle différence entre une classe et une interface ?

\- Quelle différence entre \`IEnumerable\` et \`IQueryable\` ?

\- Comment fonctionne \`async/await\` ?

\- Qu'est-ce qu'un delegate ?

\- Quelle différence entre \`Task\` et \`Thread\` ?

\- Pourquoi utiliser les generics ?

\- Quelle différence entre \`record\` et \`class\` ?

\- Qu'est-ce qu'une exception ?

\- Quelle différence entre \`List\`, \`Dictionary\`, \`HashSet\` ?

\- Que signifie \`string?\` ?

**---**

**# 3. ASP.NET Core**

**## Fiche principale**

[Questions ASP.NET Core]\(aspnet-core.md)

Thèmes :

\`\`\`text

Web API

Routing

Model Binding

Validation

Middleware

Filters

Authentication

Authorization

Dependency Injection

Configuration

HTTP

REST

DTOs

\`\`\`

Questions typiques :

\- Qu'est-ce qu'un middleware ?

\- Comment fonctionne le pipeline ASP.NET Core ?

\- Quelle différence entre Authentication et Authorization ?

\- Comment fonctionne le Model Binding ?

\- Pourquoi utiliser des DTOs ?

\- Quelle différence entre \`IActionResult\` et \`ActionResult\<T>\` ?

\- Comment fonctionne le routing ?

\- Comment enregistrer un service avec DI ?

\- Où configurer les paramètres d'une application ?

\- Comment gérer les erreurs globalement ?

**---**

**# 4. Entity Framework Core**

**## Fiche principale**

[Questions EF Core]\(ef-core.md)

Thèmes :

\`\`\`text

DbContext

DbSet

Change Tracking

Relationships

Migrations

LINQ

Querying

Performance

Seeding

\`\`\`

Questions typiques :

\- À quoi sert \`DbContext\` ?

\- Quelle différence entre \`Add\`, \`Attach\` et \`Update\` ?

\- Qu'est-ce que le Change Tracking ?

\- Quelle différence entre \`AsNoTracking()\` et une requête normale ?

\- Comment fonctionnent les migrations ?

\- Quelle différence entre \`Include\` et une projection avec \`Select\` ?

\- Qu'est-ce que le problème N+1 ?

\- Quelle différence entre \`IEnumerable\` et \`IQueryable\` ?

\- Quand utiliser \`AsNoTracking()\` ?

\- Comment configurer une relation entre deux entités ?

**---**

**# 5. Architecture**

**## Fiche principale**

[Questions Architecture]\(architecture.md)

Thèmes :

\`\`\`text

Clean Architecture

CQRS

Repository

Unit of Work

Dependency Inversion

SOLID

Separation of Concerns

\`\`\`

Questions typiques :

\- Pourquoi utiliser Clean Architecture ?

\- Qu'est-ce que le principe Dependency Inversion ?

\- Quelle différence entre Repository et Service ?

\- Qu'est-ce que CQRS ?

\- Pourquoi séparer Domain et Infrastructure ?

\- Que signifie SOLID ?

\- Pourquoi éviter le couplage fort ?

\- Quand un Repository est-il réellement utile ?

\- Quels sont les avantages et inconvénients d'une architecture en couches ?

**---**

**# 6. Sécurité**

**## Fiche principale**

[Questions Security]\(security.md)

Thèmes :

\`\`\`text

Authentication

Authorization

JWT

OWASP

DTOs

Rate Limiting

Secrets

HTTPS

CORS

CSRF

SQL Injection

XSS

\`\`\`

Questions typiques :

\- Quelle différence entre Authentication et Authorization ?

\- Comment fonctionne un JWT ?

\- Où faut-il vérifier les permissions ?

\- Pourquoi le frontend ne suffit-il pas pour sécuriser une API ?

\- Qu'est-ce qu'une SQL Injection ?

\- Qu'est-ce qu'une XSS ?

\- Pourquoi utiliser des DTOs ?

\- Pourquoi limiter le nombre de requêtes ?

\- Comment stocker les secrets ?

\- Pourquoi HTTPS est-il indispensable ?

**---**

**# 7. Angular**

Thèmes :

\`\`\`text

Components

Services

Dependency Injection

Inputs / Outputs

Signals

RxJS

Routing

HttpClient

Forms

Authentication

\`\`\`

Questions typiques :

\- Qu'est-ce qu'un Component Angular ?

\- Quelle différence entre Input et Output ?

\- Pourquoi utiliser un Service ?

\- Comment fonctionne la Dependency Injection Angular ?

\- Qu'est-ce qu'un Signal ?

\- Quelle différence entre Signal et Observable ?

\- Quelle différence entre \`computed()\` et \`effect()\` ?

\- Qu'est-ce que \`switchMap()\` ?

\- Pourquoi utiliser \`async\` pipe ?

\- Comment Angular communique-t-il avec une API ASP.NET Core ?

**---**

**# 8. Azure**

Thèmes :

\`\`\`text

App Service

Azure SQL

Key Vault

Managed Identity

GitHub Actions

CI/CD

OIDC

Configuration

Monitoring

\`\`\`

Questions typiques :

\- Qu'est-ce qu'Azure App Service ?

\- Comment déployer une API .NET sur Azure ?

\- Pourquoi utiliser Key Vault ?

\- Qu'est-ce qu'une Managed Identity ?

\- Quelle différence entre Managed Identity et OIDC ?

\- Comment fonctionne un pipeline GitHub Actions ?

\- Quelle différence entre CI et CD ?

\- Où stocker les secrets ?

\- Comment configurer une application ASP.NET Core dans Azure ?

**---**

**# 9. Questions transversales**

Les entretiens ne portent pas uniquement sur la syntaxe.

Il faut aussi savoir expliquer des choix techniques.

Exemples :

**### Pourquoi utiliser Dependency Injection ?**

Réponse attendue :

\`\`\`text

réduire le couplage

\+

faciliter les tests

\+

contrôler la création des dépendances

\+

favoriser la séparation des responsabilités

\`\`\`

**### Pourquoi utiliser des DTOs ?**

\`\`\`text

contrôler les données exposées

\+

éviter d'exposer directement les entités

\+

adapter le contrat API

\+

limiter certains risques de sécurité

\`\`\`

**### Pourquoi utiliser async/await ?**

\`\`\`text

ne pas bloquer inutilement le thread

\+

mieux gérer les opérations I/O

\+

améliorer la scalabilité des applications serveur

\`\`\`

**### Pourquoi utiliser un service ?**

\`\`\`text

séparer les responsabilités

\+

réutiliser la logique

\+

faciliter les tests

\`\`\`

**---**

**# 10. Comparaisons à connaître**

Les questions de comparaison sont très fréquentes.

**## \`IEnumerable\` vs \`IQueryable\`**

\`\`\`text

IEnumerable

→ traitement côté mémoire

IQueryable

→ peut construire une requête destinée à une source de données

\`\`\`

**---**

**## \`Task\` vs \`Thread\`**

\`\`\`text

Task

→ abstraction d'une opération asynchrone

Thread

→ unité d'exécution du système

\`\`\`

**---**

**## Authentication vs Authorization**

\`\`\`text

Authentication

→ Qui es-tu ?

Authorization

→ As-tu le droit ?

\`\`\`

**---**

**## \`class\` vs \`record\`**

\`\`\`text

class

→ identité / comportement mutable possible

record

→ adapté aux données et à la value equality

\`\`\`

**---**

**## Signal vs Observable**

\`\`\`text

Signal

→ état réactif actuel

Observable

→ flux de valeurs dans le temps

\`\`\`

**---**

**## \`computed\` vs \`effect\`**

\`\`\`text

computed

→ calculer une valeur dérivée

effect

→ effectuer une action secondaire

\`\`\`

**---**

**## \`switchMap\` vs \`mergeMap\`**

\`\`\`text

switchMap

→ nouveau flux remplace l'ancien

mergeMap

→ plusieurs flux peuvent être actifs

\`\`\`

**---**

**# 11. Questions de conception**

Un entretien peut également donner un problème concret.

Exemple :

\> Tu dois développer une API de gestion d'hôtels. Comment l'architectures-tu ?

Une réponse possible :

\`\`\`text

ASP.NET Core API

       ↓

Controllers

       ↓

Application Services

       ↓

Domain

       ↓

Infrastructure

       ↓

EF Core

       ↓

SQL Server

\`\`\`

Et côté frontend :

\`\`\`text

Angular

       ↓

Components

       ↓

Services

       ↓

HttpClient

       ↓

ASP.NET Core API

\`\`\`

Il faut ensuite expliquer les choix.

**---**

**# 12. Questions comportementales techniques**

Il peut également être demandé :

**### "Que fais-tu lorsqu'un bug est difficile à comprendre ?"**

Bonne approche :

\`\`\`text

1\. Reproduire

2\. Isoler

3\. Lire les logs

4\. Vérifier les hypothèses

5\. Utiliser le debugger

6\. Identifier la cause

7\. Corriger

8\. Ajouter un test si pertinent

9\. Vérifier qu'il n'y a pas de régression

\`\`\`

**---**

**### "Que fais-tu si tu ne connais pas une technologie ?"**

Une bonne réponse n'est pas :

\> Je ne connais pas.

Mais plutôt :

\> Je ne l'ai pas encore utilisée en production, mais je connais les concepts proches et je commencerais par comprendre son rôle, sa documentation officielle et un petit exemple avant de l'intégrer au projet.

**---**

**# 13. Comment répondre à une question technique**

Utilise cette structure :

\`\`\`text

Définition

   ↓

Fonctionnement

   ↓

Exemple

   ↓

Pourquoi

   ↓

Limites

\`\`\`

Exemple avec Dependency Injection :

\`\`\`text

Définition

→ technique permettant de fournir les dépendances à une classe.

Fonctionnement

→ un container crée/résout les objets.

Exemple

→ Controller reçoit IHotelService.

Pourquoi

→ faible couplage et testabilité.

Limites

→ une mauvaise configuration peut rendre les dépendances difficiles à comprendre.

\`\`\`

Cette structure donne une réponse beaucoup plus professionnelle qu'une simple définition.

**---**

**# 14. Niveau junior : ce qu'on attend**

Pour un profil junior / junior reconversion, on ne s'attend pas nécessairement à connaître chaque détail du framework.

On attend surtout :

\`\`\`text

bases solides

\+

raisonnement

\+

compréhension des concepts

\+

capacité à apprendre

\+

capacité à expliquer ses choix

\`\`\`

Il vaut mieux savoir expliquer parfaitement :

\`\`\`text

DI

async/await

API REST

EF Core

DTO

Authentication / Authorization

\`\`\`

que réciter cinquante méthodes sans comprendre leur rôle.

**---**

**# 15. Niveau intermédiaire : ce qu'il faut ajouter**

Pour aller vers un profil plus autonome :

\`\`\`text

Architecture

Performance

Testing

Security

CI/CD

Cloud

Observability

Design patterns

\`\`\`

Il faut être capable de répondre à :

\`\`\`text

Pourquoi ?

\`\`\`

et pas seulement :

\`\`\`text

Comment ?

\`\`\`

**---**

**# 16. Méthode de révision**

Pour chaque concept :

**### Étape 1**

Explique-le sans regarder la fiche.

**### Étape 2**

Donne un exemple de code.

**### Étape 3**

Explique ce qui se passe derrière.

**### Étape 4**

Compare-le avec une notion proche.

**### Étape 5**

Donne un cas où tu ne l'utiliserais pas.

**### Étape 6**

Réponds à une question d'entretien.

Exemple :

\`\`\`text

DI

 ↓

définition

 ↓

code

 ↓

fonctionnement interne

 ↓

Scoped / Transient / Singleton

 ↓

avantages

 ↓

pièges

 ↓

question d'entretien

\`\`\`

**---**

**# 17. Checklist avant un entretien .NET**

\`\`\`text

C#

[ ] OOP

[ ] Interfaces

[ ] Generics

[ ] LINQ

[ ] async/await

[ ] Exceptions

[ ] Collections

[ ] Delegates / Events

ASP.NET Core

[ ] Web API

[ ] Routing

[ ] Model Binding

[ ] Validation

[ ] Middleware

[ ] DI

[ ] Authentication

[ ] Authorization

[ ] DTOs

EF Core

[ ] DbContext

[ ] Tracking

[ ] Relationships

[ ] Migrations

[ ] LINQ

[ ] Performance

[ ] Seeding

Architecture

[ ] SOLID

[ ] Clean Architecture

[ ] CQRS

[ ] Repository

[ ] Dependency Inversion

Security

[ ] JWT

[ ] OWASP

[ ] Authorization

[ ] Secrets

[ ] Rate Limiting

Angular

[ ] Components

[ ] Services

[ ] Inputs / Outputs

[ ] Signals

[ ] RxJS

[ ] Routing

[ ] HttpClient

Azure

[ ] App Service

[ ] Azure SQL

[ ] Key Vault

[ ] Managed Identity

[ ] GitHub Actions

[ ] CI/CD

\`\`\`

**---**

**# 18. La compétence la plus importante**

Une bonne développeuse ne mémorise pas uniquement :

\`\`\`text

"quelle commande ?"

\`\`\`

Elle comprend :

\`\`\`text

Pourquoi cette commande existe ?

        ↓

Quel problème résout-elle ?

        ↓

Que se passe-t-il derrière ?

        ↓

Quelles sont les alternatives ?

        ↓

Quels sont les compromis ?

\`\`\`

C'est cette compréhension qui permet de transférer ses connaissances d'un projet à un autre.

**---**

**# À retenir**

\`\`\`text

Un entretien technique ne teste pas uniquement

la mémoire.

Il teste :

Compréhension

\+

Raisonnement

\+

Expérience

\+

Communication

\+

Capacité à faire des choix techniques

\`\`\`

**## Phrase à mémoriser**

\> **\*\*Pour réussir un entretien .NET, je ne dois pas seulement savoir utiliser une technologie : je dois pouvoir expliquer ce qu'elle fait, pourquoi je l'utilise, ce qui se passe derrière et quelles alternatives ou limites existent.\*\***