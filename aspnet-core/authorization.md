# Authorization en ASP.NET Core

## 1. Définition

L'**Authorization** détermine si un utilisateur authentifié a le droit d'effectuer une opération ou d'accéder à une ressource.

La question fondamentale est :

> **« As-tu le droit de faire cette action ? »**

À ne pas confondre avec l'Authentication :

```text
Authentication
    -> Qui es-tu ?

Authorization
    -> As-tu le droit ?
```

Mentalement :

```text
Request
   |
   v
Authentication
   |
   v
ClaimsPrincipal
   |
   v
Authorization
   |
   +---- refusé ----> 403
   |
   v
Endpoint
```

---

# 2. Exemple simple

Supposons :

```csharp
[Authorize]
public IActionResult GetPrivateData()
{
    return Ok();
}
```

L'utilisateur doit être authentifié.

Mais on peut être plus précis :

```csharp
[Authorize(Roles = "Admin")]
public IActionResult DeleteHotel(int id)
{
    return Ok();
}
```

Ici, être simplement authentifié ne suffit pas.

Il faut également :

```text
Role = Admin
```

---

# 3. Authentication vs Authorization

Exemple :

```text
Utilisateur
    |
    v
Login
    |
    v
Authentication
    |
    v
User = Ahlame
    |
    v
Authorization
    |
    +---- Admin ? ---- oui ---> accès
    |
    +---- Admin ? ---- non ---> 403
```

L'Authentication construit l'identité.

L'Authorization prend ensuite cette identité pour décider.

---

# 4. `ClaimsPrincipal`

L'autorisation travaille généralement avec :

```csharp
HttpContext.User
```

qui est un :

```csharp
ClaimsPrincipal
```

Il peut contenir des informations comme :

```text
UserId = 42
Name = Ahlame
Role = Admin
Permission = hotel.read
Permission = hotel.write
```

Ces informations sont appelées **claims**.

---

# 5. Role-based Authorization

La forme la plus simple est l'autorisation basée sur les rôles.

Exemple :

```csharp
[Authorize(Roles = "Admin")]
public IActionResult DeleteHotel(int id)
{
    return Ok();
}
```

On peut également accepter plusieurs rôles :

```csharp
[Authorize(Roles = "Admin,Manager")]
public IActionResult UpdateHotel(int id)
{
    return Ok();
}
```

Mentalement :

```text
Admin OU Manager
```

---

# 6. Pourquoi les rôles ne suffisent pas toujours ?

Imagine une application avec :

```text
Admin
Manager
User
```

Puis les permissions :

```text
hotel.read
hotel.create
hotel.update
hotel.delete
user.read
user.update
```

Un simple système de rôles peut devenir difficile à gérer.

Par exemple :

```text
Manager
    -> peut lire les hôtels
    -> peut modifier les hôtels
    -> ne peut pas supprimer les hôtels
```

Les **claims et policies** permettent d'exprimer ce type de règles plus précisément.

---

# 7. Claim-based Authorization

On peut baser l'autorisation sur un claim.

Exemple :

```text
permission = hotel.write
```

L'utilisateur peut avoir :

```text
UserId = 42
permission = hotel.read
permission = hotel.write
```

L'autorisation peut alors vérifier la présence du claim nécessaire.

Mentalement :

```text
User
 |
 +-- permission = hotel.read
 +-- permission = hotel.write
```

Puis :

```text
Endpoint
    |
    v
Requires hotel.write
    |
    v
Claim présent ?
```

---

# 8. Policy-based Authorization

Une **Policy** permet de définir une règle d'autorisation réutilisable.

Exemple :

```csharp
[Authorize(Policy = "CanManageHotels")]
public IActionResult UpdateHotel(int id)
{
    return Ok();
}
```

La policy :

```text
CanManageHotels
```

contient la règle permettant de décider si l'utilisateur est autorisé.

Mentalement :

```text
[Authorize]
      |
      v
Policy
      |
      v
Requirements
      |
      v
Handlers
      |
      v
Autorisé / Refusé
```

---

# 9. Déclarer une Policy

Exemple :

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanManageHotels", policy =>
    {
        policy.RequireClaim(
            "permission",
            "hotel.write");
    });
});
```

Puis :

```csharp
[Authorize(Policy = "CanManageHotels")]
public IActionResult UpdateHotel(int id)
{
    return Ok();
}
```

Le flux devient :

```text
Request
   |
   v
Authentication
   |
   v
ClaimsPrincipal
   |
   v
Policy CanManageHotels
   |
   v
permission = hotel.write ?
   |
   +---- oui ---> Controller
   |
   +---- non ---> 403
```

---

# 10. `RequireClaim`

Exemple :

```csharp
options.AddPolicy("CanReadHotels", policy =>
{
    policy.RequireClaim(
        "permission",
        "hotel.read");
});
```

L'utilisateur doit posséder :

```text
permission = hotel.read
```

---

# 11. `RequireRole`

On peut également créer une policy basée sur un rôle :

```csharp
options.AddPolicy("HotelManager", policy =>
{
    policy.RequireRole("Admin", "Manager");
});
```

La policy signifie :

```text
Admin OU Manager
```

---

# 12. `RequireAuthenticatedUser`

Une policy peut simplement exiger une authentification :

```csharp
options.AddPolicy("Authenticated", policy =>
{
    policy.RequireAuthenticatedUser();
});
```

L'utilisateur doit avoir une identité authentifiée.

---

# 13. Plusieurs requirements

Une policy peut avoir plusieurs exigences.

Exemple :

```csharp
options.AddPolicy("SecureHotelManagement", policy =>
{
    policy.RequireAuthenticatedUser();
    policy.RequireRole("Admin");
    policy.RequireClaim("permission", "hotel.write");
});
```

Mentalement :

```text
Authentifié
    ET
Admin
    ET
permission = hotel.write
```

Si une condition obligatoire échoue, l'autorisation échoue.

---

# 14. Requirement

Un **Requirement** représente une condition d'autorisation.

Exemple :

```csharp
public class MinimumAgeRequirement(int age)
    : IAuthorizationRequirement
{
    public int Age { get; } = age;
}
```

Le requirement représente :

```text
Age >= X
```

Mais le requirement lui-même ne contient pas nécessairement toute la logique permettant de vérifier cette condition.

Cette logique appartient généralement au **Handler**.

---

# 15. Authorization Handler

Le Handler contient la logique permettant de déterminer si un requirement est satisfait.

Exemple :

```csharp
public class MinimumAgeHandler
    : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        var ageClaim =
            context.User.FindFirst("age");

        if (ageClaim != null &&
            int.TryParse(ageClaim.Value, out var age) &&
            age >= requirement.Age)
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

Mentalement :

```text
Policy
   |
   v
Requirement
   |
   v
Handler
   |
   v
context.User
   |
   v
Succeed / Fail
```

---

# 16. `context.Succeed`

Dans un handler :

```csharp
context.Succeed(requirement);
```

signifie que le requirement est satisfait.

Exemple :

```csharp
if (userIsAllowed)
{
    context.Succeed(requirement);
}
```

---

# 17. `context.Fail`

On peut également explicitement indiquer un échec :

```csharp
context.Fail();
```

Cela signifie que l'autorisation doit échouer.

Il faut comprendre qu'un handler n'est pas simplement une méthode qui retourne :

```csharp
true / false
```

Il travaille avec le contexte d'autorisation et les requirements.

---

# 18. Enregistrer un Handler

Avec DI :

```csharp
builder.Services.AddScoped<
    IAuthorizationHandler,
    MinimumAgeHandler>();
```

Le framework peut ensuite utiliser le Handler lorsqu'une policy correspondante est évaluée.

---

# 19. Policy + Requirement + Handler

C'est l'un des concepts les plus importants.

Exemple :

```text
Policy
"CanManageHotels"
        |
        v
Requirement
"Permission hotel.write"
        |
        v
Handler
"Vérifie le claim"
        |
        v
Authorization decision
```

On peut résumer :

```text
Policy = règle nommée
Requirement = condition
Handler = logique qui vérifie la condition
```

---

# 20. Resource-based Authorization

Parfois, savoir si l'utilisateur peut faire quelque chose dépend de la ressource elle-même.

Exemple :

```text
Utilisateur A
    -> peut modifier ses propres hôtels

Utilisateur B
    -> ne peut pas modifier l'hôtel de A
```

Une simple policy globale peut ne pas suffire.

On doit alors prendre en compte :

```text
Utilisateur
+
Ressource
```

Mentalement :

```text
User
  +
Resource
  |
  v
Authorization Handler
  |
  v
Allowed / Denied
```

---

# 21. Exemple de resource-based authorization

Supposons :

```csharp
public class Hotel
{
    public int Id { get; set; }

    public int OwnerId { get; set; }
}
```

On veut permettre :

```text
Owner -> modification
Admin -> modification
```

Le Handler peut comparer :

```csharp
context.User
```

avec :

```csharp
hotel.OwnerId
```

Conceptuellement :

```csharp
if (isAdmin || isOwner)
{
    context.Succeed(requirement);
}
```

C'est un cas classique où la ressource doit être connue pour prendre la décision.

---

# 22. Pourquoi le Controller ne devrait pas tout gérer ?

On pourrait écrire :

```csharp
if (User.IsInRole("Admin") ||
    hotel.OwnerId == currentUserId)
{
    // autorisé
}
```

Mais si cette logique est répétée dans plusieurs endpoints :

```text
Get
Update
Delete
Publish
Archive
```

on obtient beaucoup de duplication.

Une policy ou un handler permet de centraliser cette logique.

---

# 23. `User.IsInRole`

Dans un Controller :

```csharp
if (User.IsInRole("Admin"))
{
    // ...
}
```

Cela peut être utile pour certaines décisions simples.

Mais si la règle devient complexe ou répétée, une policy est généralement plus adaptée.

---

# 24. `User.HasClaim`

On peut également vérifier un claim :

```csharp
if (User.HasClaim(
    "permission",
    "hotel.write"))
{
    // ...
}
```

Encore une fois :

> Une vérification ponctuelle peut être acceptable ; une règle réutilisable est souvent mieux exprimée avec une policy.

---

# 25. Authorization dans le pipeline

Dans une API Controller classique, on trouve typiquement :

```csharp
app.UseAuthentication();
app.UseAuthorization();

app.MapControllers();
```

Mentalement :

```text
HTTP
 |
 v
Authentication
 |
 v
ClaimsPrincipal
 |
 v
Authorization
 |
 v
Endpoint
```

---

# 26. Que se passe-t-il avec `[Authorize]` ?

Exemple :

```csharp
[Authorize]
[HttpGet]
public IActionResult GetPrivateData()
{
    return Ok();
}
```

Le pipeline vérifie si l'accès satisfait les exigences d'autorisation.

Si l'utilisateur n'est pas authentifié :

```text
401
```

selon le mécanisme d'authentification utilisé.

Si l'utilisateur est authentifié mais non autorisé :

```text
403
```

Mentalement :

```text
Pas d'identité valide
    -> 401

Identité valide mais droits insuffisants
    -> 403
```

---

# 27. `[Authorize(Roles = "...")]`

Exemple :

```csharp
[Authorize(Roles = "Admin")]
public IActionResult DeleteHotel(int id)
{
    return Ok();
}
```

Le système vérifie le rôle attendu.

Pour plusieurs rôles :

```csharp
[Authorize(Roles = "Admin,Manager")]
```

signifie généralement :

```text
Admin OU Manager
```

Attention à ne pas confondre avec plusieurs attributs.

---

# 28. Plusieurs `[Authorize]`

Exemple :

```csharp
[Authorize(Roles = "Admin")]
[Authorize(Policy = "CanManageHotels")]
public IActionResult UpdateHotel(int id)
{
    return Ok();
}
```

Ces exigences se combinent.

Mentalement :

```text
Admin
   ET
CanManageHotels
```

Cela peut devenir puissant, mais aussi difficile à lire si on empile trop de règles.

---

# 29. Default Policy

On peut configurer une policy par défaut :

```csharp
builder.Services.AddAuthorization(options =>
{
    options.DefaultPolicy =
        new AuthorizationPolicyBuilder()
            .RequireAuthenticatedUser()
            .Build();
});
```

L'idée est de définir ce qui doit être exigé lorsqu'un endpoint utilise l'autorisation standard.

---

# 30. Fallback Policy

Une fallback policy peut permettre d'exiger une autorisation par défaut pour les endpoints qui n'ont pas explicitement défini de policy d'autorisation.

Conceptuellement :

```text
Endpoint sans règle explicite
        |
        v
Fallback Policy
```

Cela peut être utile pour adopter une stratégie :

> « Tout est protégé par défaut, sauf ce qui est explicitement public. »

Dans ce type d'architecture, on utilise notamment :

```csharp
[AllowAnonymous]
```

pour les endpoints qui doivent rester publics.

---

# 31. Policy naming

Les noms des policies doivent être compréhensibles.

Préférer :

```text
CanManageHotels
CanDeleteHotels
CanReadUsers
```

plutôt que :

```text
Policy1
Policy2
Policy3
```

Le nom représente l'intention de sécurité.

---

# 32. Roles vs Claims vs Policies

### Roles

Simple :

```text
Admin
Manager
User
```

### Claims

Plus précis :

```text
permission = hotel.write
department = sales
country = BE
```

### Policies

Expriment une règle :

```text
CanManageHotels
```

qui peut combiner :

```text
roles
claims
requirements
handlers
```

Mentalement :

```text
Role / Claim
      |
      v
Policy
      |
      v
Authorization decision
```

---

# 33. Exemple d'architecture par permissions

Imaginons :

```text
User
 |
 +-- hotel.read
 +-- hotel.create
 +-- hotel.update
```

Policies :

```text
CanReadHotels
CanCreateHotels
CanUpdateHotels
```

Configuration :

```csharp
options.AddPolicy(
    "CanUpdateHotels",
    policy =>
    {
        policy.RequireClaim(
            "permission",
            "hotel.update");
    });
```

Controller :

```csharp
[Authorize(Policy = "CanUpdateHotels")]
public IActionResult UpdateHotel(int id)
{
    return Ok();
}
```

Avantage :

```text
Le Controller exprime l'intention.
La Policy contient la règle.
```

---

# 34. Authorization Handler avec accès à une base

Un Handler peut avoir des dépendances.

Exemple conceptuel :

```csharp
public class HotelAuthorizationHandler(
    IHotelRepository repository)
    : AuthorizationHandler<
        ManageHotelRequirement,
        Hotel>
{
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        ManageHotelRequirement requirement,
        Hotel hotel)
    {
        var userId =
            context.User.FindFirst(
                ClaimTypes.NameIdentifier)?.Value;

        if (/* utilisateur propriétaire */)
        {
            context.Succeed(requirement);
        }

        await Task.CompletedTask;
    }
}
```

Cela permet d'implémenter des règles dépendant des données.

Attention :

> Les handlers doivent rester raisonnablement simples et ne doivent pas devenir une seconde couche métier géante.

---

# 35. Authorization et logique métier

Il faut distinguer :

```text
Authorization
```

et :

```text
Business Rule
```

Exemple :

```text
L'utilisateur peut modifier cet hôtel
```

peut être une règle d'autorisation.

Mais :

```text
Un hôtel ne peut pas être publié sans au moins une chambre
```

est une règle métier.

Ce sont deux concepts différents.

---

# 36. Authorization et validation

Encore une autre distinction :

### Validation

```text
Les données sont-elles correctes ?
```

### Authorization

```text
L'utilisateur a-t-il le droit ?
```

Exemple :

```json
{
  "name": "Hotel Central"
}
```

Validation :

```text
Name obligatoire ?
Longueur correcte ?
```

Authorization :

```text
Cet utilisateur peut-il modifier cet hôtel ?
```

---

# 37. Authorization et Authentication

Troisième distinction fondamentale :

```text
Authentication
    -> Qui es-tu ?

Authorization
    -> As-tu le droit ?

Validation
    -> Les données sont-elles correctes ?
```

Ces trois questions doivent rester séparées.

---

# 38. 401 vs 403

### 401 Unauthorized

Utilisé lorsqu'une authentification valide est nécessaire mais absente ou incorrecte.

Exemples :

```text
Token absent
Token invalide
Token expiré
```

Mentalement :

> « Je ne peux pas établir correctement ton identité. »

### 403 Forbidden

L'utilisateur est authentifié mais n'a pas l'autorisation nécessaire.

Exemple :

```text
Role = User
Endpoint = Admin
```

Mentalement :

> « Je sais qui tu es, mais tu n'as pas le droit. »

---

# 39. Erreurs fréquentes

## Erreur 1 : mettre toutes les permissions dans les rôles

Exemple :

```text
Admin
Manager
ManagerCanReadHotels
ManagerCanWriteHotels
ManagerCanDeleteHotels
```

Cela peut rapidement produire une explosion de rôles.

Les permissions/claims/policies peuvent être plus adaptées.

---

## Erreur 2 : mettre l'autorisation dans le frontend uniquement

Angular peut cacher :

```text
Bouton Delete
```

mais cela ne sécurise pas l'API.

Un attaquant peut appeler directement :

```http
DELETE /api/hotels/5
```

Le serveur doit vérifier l'autorisation.

---

## Erreur 3 : faire confiance à un `userId` du body

Exemple :

```json
{
  "userId": 42
}
```

Le client peut modifier :

```json
{
  "userId": 43
}
```

L'identité authentifiée doit venir du principal de sécurité, pas d'une donnée contrôlée par le client.

---

## Erreur 4 : mélanger validation et autorisation

```text
Name vide
```

n'est pas une erreur d'autorisation.

```text
User non propriétaire
```

n'est pas une erreur de validation de DTO.

---

## Erreur 5 : mettre toute l'autorisation dans les Controllers

Si plusieurs endpoints utilisent la même règle :

```text
Get
Update
Delete
```

une policy/handler peut éviter la duplication.

---

## Erreur 6 : créer des policies impossibles à comprendre

Éviter :

```text
Policy1
Policy2
Policy3
```

Préférer des noms exprimant l'intention :

```text
CanManageHotels
CanDeleteHotels
CanReadUsers
```

---

# 40. Flux complet

```text
                    HTTP REQUEST
                         |
                         v
                  Authentication
                         |
                         v
                  ClaimsPrincipal
                         |
                         v
                   Authorization
                         |
             +-----------+-----------+
             |                       |
          refusé                  autorisé
             |                       |
             v                       v
           401/403               Controller
                                     |
                                     v
                                   Service
                                     |
                                     v
                                  Database
```

Pour une policy :

```text
[Authorize(Policy = "CanManageHotels")]
                    |
                    v
                 Policy
                    |
                    v
              Requirements
                    |
                    v
                 Handlers
                    |
                    v
             Authorization Result
```

---

# 41. Règle mentale

Quand tu vois :

```csharp
[Authorize]
```

pense :

> « Cet endpoint n'est pas public. Une règle d'autorisation doit être satisfaite. »

Quand tu vois :

```csharp
[Authorize(Roles = "Admin")]
```

pense :

> « L'utilisateur doit avoir le rôle Admin. »

Quand tu vois :

```csharp
[Authorize(Policy = "CanManageHotels")]
```

pense :

> « Je délègue la décision à une policy nommée. »

Quand tu vois :

```csharp
IAuthorizationRequirement
```

pense :

> « Voici une condition que l'autorisation doit satisfaire. »

Quand tu vois :

```csharp
AuthorizationHandler<TRequirement>
```

pense :

> « Voici la logique qui sait comment vérifier cette condition. »

---

# 42. À retenir

1. Authorization répond à « As-tu le droit ? ».
2. Authentication répond à « Qui es-tu ? ».
3. Validation répond à « Les données sont-elles correctes ? ».
4. `HttpContext.User` contient généralement le principal authentifié.
5. `ClaimsPrincipal` contient une ou plusieurs identités et leurs claims.
6. Les rôles permettent une autorisation simple basée sur des catégories d'utilisateurs.
7. Les claims permettent de transporter des informations utiles à l'autorisation.
8. Les policies permettent de centraliser des règles d'autorisation réutilisables.
9. Un requirement représente une condition d'autorisation.
10. Un handler contient la logique permettant de vérifier un requirement.
11. `context.Succeed(requirement)` indique qu'un requirement est satisfait.
12. Resource-based authorization permet de prendre la ressource elle-même en compte.
13. Les policies peuvent combiner plusieurs exigences.
14. `[Authorize]` protège un endpoint.
15. `[AllowAnonymous]` permet explicitement un accès public lorsqu'une règle globale existe.
16. 401 correspond généralement à un problème d'authentification.
17. 403 correspond généralement à un problème d'autorisation.
18. Cacher un bouton dans Angular ne sécurise pas une API.
19. L'autorisation doit toujours être vérifiée côté serveur.
20. Les règles métier ne doivent pas être confondues avec les règles d'autorisation.
21. Les policies doivent avoir des noms exprimant leur intention.
22. Les handlers peuvent recevoir des dépendances via DI.
23. Une autorisation basée sur une ressource peut comparer l'utilisateur courant avec la ressource.
24. Une bonne architecture sépare Authentication, Authorization, Validation et Business Logic.

---

# Questions d'entretien

### 1. Quelle différence entre Authentication et Authorization ?

```text
Authentication -> Qui es-tu ?
Authorization   -> As-tu le droit ?
```

### 2. Quelle différence entre rôle, claim et policy ?

Un rôle représente une catégorie comme `Admin`. Un claim représente une information comme `permission = hotel.write`. Une policy représente une règle d'autorisation pouvant utiliser des rôles, claims, requirements et handlers.

### 3. Qu'est-ce qu'un `IAuthorizationRequirement` ?

C'est une condition que le système d'autorisation doit satisfaire.

### 4. Quel est le rôle d'un `AuthorizationHandler` ?

Il contient la logique permettant de déterminer si un ou plusieurs requirements sont satisfaits.

### 5. Pourquoi utiliser une policy plutôt que `User.IsInRole()` partout ?

Une policy permet de centraliser une règle réutilisable et de la faire évoluer sans dupliquer la logique dans les Controllers.

### 6. Qu'est-ce que Resource-based Authorization ?

C'est une autorisation qui prend en compte la ressource concernée en plus de l'utilisateur.

Exemple :

```text
User A peut modifier son hôtel
User B ne peut pas modifier cet hôtel
```

### 7. Quelle différence entre 401 et 403 ?

```text
401 -> problème d'authentification
403 -> utilisateur authentifié mais non autorisé
```

### 8. Pourquoi le frontend ne suffit-il pas pour l'autorisation ?

Parce qu'un client peut appeler directement l'API sans passer par l'interface graphique.

### 9. Où doit-on récupérer l'identité de l'utilisateur connecté ?

Généralement depuis :

```csharp
HttpContext.User
```

et ses claims, plutôt que depuis une valeur contrôlée par le client.

### 10. Pourquoi séparer Authorization et Business Logic ?

L'autorisation répond à la question « cet utilisateur peut-il effectuer cette opération ? », tandis que la logique métier décrit les règles de fonctionnement du domaine.

---

# Phrase à retenir

> **Authentication identifie l'utilisateur, Authorization décide s'il a le droit, et une Policy + Requirements + Handlers permettent de centraliser les règles d'autorisation complexes.**
