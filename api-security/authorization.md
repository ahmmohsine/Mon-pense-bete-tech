# API Security — Authorization

## 1. Authorization : « As-tu le droit ? »

L'autorisation intervient après l'authentification.

L'authentification répond :

> **Qui es-tu ?**

L'autorisation répond :

> **Qu'as-tu le droit de faire ?**

Exemple :

```text
Utilisateur
    ↓
Authentication
    ↓
"Je sais qui tu es : Ahlame"
    ↓
Authorization
    ↓
"Es-tu autorisée à supprimer cet hôtel ?"
```

Mental model :

```text
Authentication = identité
Authorization  = permissions
```

---

# 2. Le fonctionnement global

Dans une API ASP.NET Core protégée :

```text
HTTP Request
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
     +---- Refusé ----> 401 / 403
     |
     v
Endpoint
```

L'autorisation ne cherche donc pas principalement à savoir qui est l'utilisateur.

Elle utilise l'identité déjà construite par l'authentification pour prendre une décision.

---

# 3. `[Authorize]`

L'attribut le plus connu est :

```csharp
[Authorize]
```

Exemple :

```csharp
[Authorize]
[HttpGet]
public IActionResult GetHotels()
{
    return Ok();
}
```

Cela signifie :

```text
L'utilisateur doit être authentifié
```

Sans authentification valide, l'accès est refusé.

On peut aussi utiliser :

```csharp
[AllowAnonymous]
```

pour autoriser explicitement un endpoint qui serait normalement protégé.

Exemple :

```csharp
[AllowAnonymous]
[HttpPost("login")]
public IActionResult Login()
{
    ...
}
```

---

# 4. Minimal APIs et RequireAuthorization

Avec les Minimal APIs, on peut utiliser :

```csharp
app.MapGet("/api/hotels", GetHotels)
   .RequireAuthorization();
```

Cela revient conceptuellement à dire :

```text
Cet endpoint nécessite une autorisation.
```

On peut également utiliser une policy :

```csharp
app.MapGet("/api/hotels", GetHotels)
   .RequireAuthorization("CanReadHotels");
```

Mentalement :

```text
[Authorize]
        ou
.RequireAuthorization()
        ↓
Endpoint protégé
```

---

# 5. Authentication vs Authorization

Il faut absolument savoir expliquer cette différence en entretien.

Supposons :

```text
Ahlame se connecte
```

Authentication :

```text
"Le mot de passe est correct.
Je sais que cette personne est Ahlame."
```

Authorization :

```text
"Ahlame est-elle autorisée à supprimer cet hôtel ?"
```

Donc :

```text
Authentication
    ↓
Identité

Authorization
    ↓
Permission
```

---

# 6. Roles

Une manière simple de gérer les permissions consiste à utiliser des rôles.

Exemple :

```text
Admin
Manager
User
```

Avec un contrôleur :

```csharp
[Authorize(Roles = "Admin")]
public IActionResult DeleteHotel(int id)
{
    ...
}
```

Seuls les utilisateurs possédant le rôle :

```text
Admin
```

pourront accéder à l'action.

Avec une Minimal API :

```csharp
app.MapDelete("/api/hotels/{id}", DeleteHotel)
   .RequireAuthorization(policy =>
       policy.RequireRole("Admin"));
```

---

# 7. Plusieurs rôles

On peut autoriser plusieurs rôles :

```csharp
[Authorize(Roles = "Admin,Manager")]
```

Cela signifie conceptuellement :

```text
Admin OU Manager
```

Attention à ne pas confondre avec :

```text
Admin ET Manager
```

La virgule dans `Roles` représente ici une alternative.

Pour des règles plus complexes, les policies sont généralement plus adaptées.

---

# 8. Claims

Une claim représente une information associée à l'identité.

Exemples :

```text
sub        = 123
email      = ahlame@example.com
role       = Admin
department = IT
```

Dans le code :

```csharp
var userId = User.FindFirst("sub")?.Value;
```

Ou :

```csharp
var department = User.FindFirst("department")?.Value;
```

Une claim n'est pas automatiquement une permission.

Elle devient utile lorsque les règles d'autorisation l'utilisent.

---

# 9. Policy-based Authorization

Les policies permettent de créer des règles d'autorisation nommées.

Exemple :

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
    {
        policy.RequireRole("Admin");
    });
});
```

Puis :

```csharp
[Authorize(Policy = "AdminOnly")]
public IActionResult DeleteHotel(int id)
{
    ...
}
```

Avec Minimal API :

```csharp
app.MapDelete("/api/hotels/{id}", DeleteHotel)
   .RequireAuthorization("AdminOnly");
```

Mental model :

```text
Policy
   ↓
Règle d'autorisation
   ↓
Utilisateur respecte-t-il la règle ?
   ↓
Oui / Non
```

---

# 10. Pourquoi utiliser des policies ?

Imagine une application qui possède ces règles :

```text
Admin
Manager
Employee
PremiumCustomer
VerifiedUser
```

Si toute la logique est directement écrite dans les contrôleurs, on peut rapidement obtenir beaucoup de duplication.

Avec les policies :

```text
CanDeleteHotel
CanManageUsers
CanViewFinancialData
CanModifyBooking
```

Le nom de la policy exprime l'intention métier.

Exemple :

```csharp
options.AddPolicy("CanDeleteHotel", policy =>
{
    policy.RequireRole("Admin");
});
```

Puis :

```csharp
[Authorize(Policy = "CanDeleteHotel")]
```

Le contrôleur devient plus lisible.

---

# 11. RequireClaim

Une policy peut exiger une claim particulière.

Exemple :

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("ITOnly", policy =>
    {
        policy.RequireClaim("department", "IT");
    });
});
```

Puis :

```csharp
[Authorize(Policy = "ITOnly")]
public IActionResult GetInternalData()
{
    ...
}
```

L'utilisateur doit donc avoir :

```text
department = IT
```

---

# 12. Plusieurs exigences dans une policy

On peut construire une policy plus complexe.

Exemple :

```csharp
options.AddPolicy("AdminIT", policy =>
{
    policy.RequireAuthenticatedUser();
    policy.RequireRole("Admin");
    policy.RequireClaim("department", "IT");
});
```

Mentalement :

```text
Utilisateur authentifié
        ET
Role = Admin
        ET
Department = IT
        ↓
Autorisé
```

Cela illustre une grande différence entre :

```text
Role
```

et :

```text
Policy
```

Une policy permet de combiner plusieurs exigences.

---

# 13. RequireAuthenticatedUser

On peut demander explicitement qu'un utilisateur soit authentifié :

```csharp
policy.RequireAuthenticatedUser();
```

Exemple :

```csharp
options.AddPolicy("AuthenticatedOnly", policy =>
{
    policy.RequireAuthenticatedUser();
});
```

C'est utile lorsque la policy contient également d'autres règles.

---

# 14. Fallback Policy

ASP.NET Core permet également de définir une policy appliquée par défaut selon la configuration de l'application.

L'idée générale est :

```text
Par défaut :
    les endpoints nécessitent une authentification
```

Puis on ouvre explicitement certains endpoints avec :

```csharp
[AllowAnonymous]
```

Cette approche peut être intéressante pour une API où la majorité des endpoints doivent être protégés.

Mais elle doit être utilisée avec une compréhension claire des exceptions publiques.

---

# 15. 401 Unauthorized

Le code :

```http
401 Unauthorized
```

correspond généralement à un problème d'authentification.

Exemples :

```text
Token absent
Token invalide
Token expiré
Authentification non établie
```

Mental model :

> **Je ne sais pas qui tu es, ou je ne peux pas accepter ton identité.**

---

# 16. 403 Forbidden

Le code :

```http
403 Forbidden
```

signifie que l'utilisateur est authentifié mais que l'accès est refusé.

Exemple :

```text
Utilisateur :
    role = User

Endpoint :
    role requis = Admin
```

L'API sait qui est l'utilisateur.

Mais :

```text
User != Admin
```

Donc :

```http
403 Forbidden
```

À mémoriser :

```text
401 = problème d'identité
403 = problème de permission
```

---

# 17. Resource-based Authorization

Les rôles et policies sont très utiles, mais parfois la permission dépend de la ressource elle-même.

Exemple :

```text
GET /api/orders/123
```

L'utilisateur peut être autorisé à consulter :

```text
ses propres commandes
```

mais pas :

```text
les commandes des autres utilisateurs
```

La question devient :

```text
"Cet utilisateur peut-il accéder à CETTE ressource ?"
```

Ce n'est plus simplement :

```text
Role = User
```

Il faut regarder la ressource.

Exemple conceptuel :

```csharp
if (order.UserId != currentUserId)
{
    return Forbid();
}
```

Pour des systèmes plus complexes, ASP.NET Core permet également de construire une autorisation basée sur des requirements et des handlers.

---

# 18. Authorization Requirements

Un requirement représente une condition d'autorisation.

Exemple conceptuel :

```csharp
public class MinimumAgeRequirement : IAuthorizationRequirement
{
    public int MinimumAge { get; }

    public MinimumAgeRequirement(int minimumAge)
    {
        MinimumAge = minimumAge;
    }
}
```

Un handler peut ensuite déterminer si cette condition est satisfaite.

Mental model :

```text
Policy
   ↓
Requirement
   ↓
AuthorizationHandler
   ↓
Requirement satisfait ?
   ↓
Authorized / Forbidden
```

Cette approche permet de déplacer la logique complexe hors des contrôleurs.

---

# 19. AuthorizationHandler

Un handler contient la logique permettant de déterminer si un requirement est satisfait.

Structure conceptuelle :

```csharp
public class MinimumAgeHandler
    : AuthorizationHandler<MinimumAgeRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        MinimumAgeRequirement requirement)
    {
        // Vérifier les claims de l'utilisateur

        context.Succeed(requirement);

        return Task.CompletedTask;
    }
}
```

Le principe est :

```text
Requirement
    ↓
Handler
    ↓
Vérification
    ↓
Succeed()
```

Le handler peut examiner :

```csharp
context.User
```

et, selon le type d'autorisation, la ressource concernée.

---

# 20. Roles vs Claims vs Policies vs Handlers

Voici le modèle à retenir :

| Mécanisme | Utilité |
|---|---|
| Claim | Information sur l'utilisateur |
| Role | Catégorie de permissions |
| Policy | Règle d'autorisation nommée |
| Requirement | Condition d'une policy |
| Handler | Logique qui vérifie une condition |

Exemple :

```text
User
 |
 +-- Claim: department = IT
 |
 +-- Role: Admin
 |
 ↓
Policy: CanDeleteHotel
 |
 +-- Requirement
       |
       +-- Handler
```

---

# 21. Authorization et HttpContext.User

L'autorisation utilise principalement :

```csharp
HttpContext.User
```

Exemple :

```csharp
[Authorize]
public IActionResult Get()
{
    var user = HttpContext.User;

    var userId = user.FindFirst("sub")?.Value;

    return Ok(userId);
}
```

Le pipeline peut donc être résumé :

```text
Authentication
      ↓
HttpContext.User
      ↓
Authorization
      ↓
Endpoint
```

---

# 22. Ne jamais faire confiance à une donnée envoyée par le client

Supposons une requête :

```json
{
    "userId": 123,
    "amount": 5000
}
```

Il ne faut pas considérer :

```text
userId = 123
```

comme une preuve que l'utilisateur connecté est l'utilisateur 123.

Le client contrôle les données qu'il envoie.

L'identité authentifiée doit venir du mécanisme d'authentification :

```csharp
User
```

et de ses claims validées.

Mental model :

```text
Client input
    ↓
Donnée non fiable

Authenticated identity
    ↓
Source de l'identité après validation
```

---

# 23. IDOR et contrôle d'accès

Un problème classique de sécurité API est l'IDOR :

```text
Insecure Direct Object Reference
```

Exemple :

```http
GET /api/orders/123
```

L'application vérifie seulement :

```text
Utilisateur authentifié ?
```

mais ne vérifie pas :

```text
Cette commande appartient-elle à cet utilisateur ?
```

Un attaquant peut alors essayer :

```http
GET /api/orders/124
GET /api/orders/125
GET /api/orders/126
```

L'authentification seule ne suffit donc pas.

Il faut également vérifier l'autorisation sur la ressource.

---

# 24. Exemple : propriétaire d'une ressource

Supposons :

```csharp
public class Order
{
    public int Id { get; set; }
    public string UserId { get; set; } = null!;
}
```

Une règle peut être :

```text
Un utilisateur peut consulter uniquement ses propres commandes.
```

Conceptuellement :

```csharp
var currentUserId =
    User.FindFirst("sub")?.Value;

if (order.UserId != currentUserId)
{
    return Forbid();
}
```

Le point essentiel :

```text
Authenticated ≠ Authorized
```

---

# 25. `[Authorize]` ne suffit pas toujours

Cette règle :

```csharp
[Authorize]
```

dit essentiellement :

```text
L'utilisateur doit être authentifié.
```

Elle ne signifie pas :

```text
L'utilisateur peut accéder à toutes les ressources.
```

Exemple :

```text
[Authorize]
GET /orders/123
```

peut encore nécessiter :

```text
order.UserId == authenticatedUserId
```

Donc une API correctement sécurisée doit contrôler :

```text
Identité
+
Permissions
+
Accès à la ressource
```

---

# 26. Principe du moindre privilège

Une bonne architecture applique le principe :

> **Donner uniquement les permissions nécessaires.**

Exemple :

```text
User
    → lire ses données

Manager
    → lire + modifier certaines données

Admin
    → gérer les utilisateurs
```

Il est préférable d'éviter :

```text
Tout le monde = Admin
```

ou :

```text
Toutes les API = accessibles à tous les utilisateurs authentifiés
```

Mental model :

```text
Minimum permissions
        ↓
Minimum impact
        ↓
Minimum damage if compromised
```

---

# 27. Authorization dans une API REST

Imaginons :

```text
GET    /api/hotels
POST   /api/hotels
PUT    /api/hotels/10
DELETE /api/hotels/10
```

On peut définir :

```text
GET
    User authentifié

POST
    Manager ou Admin

PUT
    Manager ou Admin

DELETE
    Admin
```

Exemple :

```csharp
[Authorize]
[HttpGet]
public IActionResult GetHotels()
{
    ...
}
```

Puis :

```csharp
[Authorize(Roles = "Admin,Manager")]
[HttpPost]
public IActionResult CreateHotel()
{
    ...
}
```

Et :

```csharp
[Authorize(Roles = "Admin")]
[HttpDelete("{id}")]
public IActionResult DeleteHotel(int id)
{
    ...
}
```

---

# 28. Authorization avec Minimal APIs

Exemple :

```csharp
app.MapGet("/api/hotels", GetHotels)
    .RequireAuthorization();

app.MapPost("/api/hotels", CreateHotel)
    .RequireAuthorization("CanManageHotels");

app.MapDelete("/api/hotels/{id}", DeleteHotel)
    .RequireAuthorization("AdminOnly");
```

Cela permet de voir directement la règle de sécurité au niveau de la route.

---

# 29. Une architecture propre

Une API peut organiser ses règles ainsi :

```text
Controller / Endpoint
        |
        ↓
Authorization
        |
        ↓
Policy
        |
        ↓
Requirement
        |
        ↓
Handler
        |
        ↓
Domain / Resource
```

L'objectif est d'éviter de transformer les contrôleurs en gros blocs de logique de sécurité.

Mauvais :

```csharp
if (user != null)
{
    if (user.Role == "Admin")
    {
        if (...)
        {
            if (...)
            {
                ...
            }
        }
    }
}
```

Plus propre :

```csharp
[Authorize(Policy = "CanDeleteHotel")]
```

et la logique complexe dans les mécanismes d'autorisation appropriés.

---

# 30. Common mistakes

## Erreur 1 : penser que `[Authorize]` signifie Admin

```csharp
[Authorize]
```

signifie principalement :

```text
Utilisateur authentifié
```

Pas :

```text
Utilisateur administrateur
```

Pour un rôle :

```csharp
[Authorize(Roles = "Admin")]
```

---

## Erreur 2 : vérifier seulement le rôle

Même un utilisateur authentifié avec un rôle correct peut ne pas avoir le droit d'accéder à une ressource particulière.

Il faut parfois vérifier :

```text
Ownership
Tenant
Resource
Business rule
```

---

## Erreur 3 : utiliser un UserId fourni par le client comme identité

Ne jamais faire :

```csharp
var userId = request.UserId;
```

et considérer automatiquement que c'est l'utilisateur connecté.

Pour l'identité authentifiée :

```csharp
var userId = User.FindFirst("sub")?.Value;
```

selon le claim utilisé par ton système.

---

## Erreur 4 : mettre toute la logique d'autorisation dans les contrôleurs

Si les règles deviennent complexes :

```text
Policy
Requirement
Handler
```

peuvent rendre l'architecture plus claire et réutilisable.

---

## Erreur 5 : confondre 401 et 403

```text
401 → authentication problem

403 → authorization problem
```

---

# 31. Checklist Authorization

```text
[ ] Les endpoints sensibles sont protégés
[ ] Authentication et Authorization sont séparées
[ ] Les rôles sont utilisés uniquement lorsque cela correspond au besoin
[ ] Les policies sont utilisées pour les règles métier réutilisables
[ ] Les claims sont validées par le mécanisme d'authentification
[ ] Les ressources sont contrôlées lorsque nécessaire
[ ] L'identité vient de HttpContext.User
[ ] Les UserId envoyés par le client ne sont pas considérés comme une preuve d'identité
[ ] Le principe du moindre privilège est appliqué
[ ] 401 et 403 sont correctement gérés
[ ] Les règles complexes ne sont pas dispersées dans les contrôleurs
```

---

# 32. Questions d'entretien

### Quelle différence entre Authentication et Authorization ?

Authentication vérifie l'identité.

Authorization vérifie les permissions.

### Que fait `[Authorize]` ?

Il indique qu'un endpoint nécessite une autorisation. Dans le cas courant, cela signifie notamment que l'utilisateur doit être authentifié.

### Comment limiter un endpoint à un rôle ?

```csharp
[Authorize(Roles = "Admin")]
```

### Qu'est-ce qu'une policy ?

Une règle d'autorisation nommée qui peut contenir une ou plusieurs exigences.

### Pourquoi utiliser une policy plutôt qu'un simple rôle ?

Parce qu'une policy peut exprimer une règle plus riche et centraliser une logique réutilisable.

### Qu'est-ce qu'un requirement ?

Une condition qu'une policy doit satisfaire.

### Qu'est-ce qu'un AuthorizationHandler ?

Un composant qui contient la logique permettant de déterminer si un requirement est satisfait.

### Pourquoi `[Authorize]` ne suffit-il pas toujours ?

Parce qu'un utilisateur authentifié peut ne pas avoir accès à une ressource particulière.

Exemple :

```text
Utilisateur authentifié
        ↓
GET /orders/123
        ↓
La commande appartient-elle à cet utilisateur ?
```

### Quelle différence entre 401 et 403 ?

```text
401 = non authentifié / authentification invalide
403 = authentifié mais non autorisé
```

### Qu'est-ce qu'un IDOR ?

Une vulnérabilité où un utilisateur peut accéder à une ressource en manipulant un identifiant sans que l'API vérifie correctement qu'il a le droit d'y accéder.

---

# 33. Schéma mental final

```text
                 REQUEST
                    |
                    v
             AUTHENTICATION
                    |
                    v
            "Qui es-tu ?"
                    |
                    v
             HttpContext.User
                    |
                    v
             AUTHORIZATION
                    |
          +---------+---------+
          |                   |
       Refusé              Autorisé
          |                   |
       401 / 403              v
                         ENDPOINT
                              |
                              v
                        RESOURCE CHECK
                              |
                              v
                            DATA
```

Il faut retenir trois niveaux :

```text
1. Identity
   Qui es-tu ?

2. Permission
   Qu'as-tu le droit de faire ?

3. Resource
   As-tu le droit d'accéder à CETTE donnée ?
```

---

# À retenir

L'autorisation n'est pas simplement :

```text
"Est-ce que l'utilisateur est connecté ?"
```

Elle peut aller beaucoup plus loin :

```text
Utilisateur authentifié
        ↓
Role / Claims
        ↓
Policy
        ↓
Resource
        ↓
Business rule
        ↓
Autorisé ou refusé
```

## Phrase à mémoriser

> **Authentication établit l'identité ; Authorization décide si cette identité peut effectuer l'action demandée sur la ressource demandée.**
