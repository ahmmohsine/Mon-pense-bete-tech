# Entretien — ASP.NET Core

Cette fiche rassemble les notions ASP.NET Core les plus importantes pour un entretien de développeuse .NET orientée Web API.

L'objectif est de savoir expliquer le chemin complet d'une requête :

```text
Client
  ↓
HTTP Request
  ↓
ASP.NET Core
  ↓
Middleware
  ↓
Routing
  ↓
Authentication
  ↓
Authorization
  ↓
Endpoint / Controller
  ↓
Model Binding
  ↓
Validation
  ↓
Service
  ↓
Repository / EF Core
  ↓
HTTP Response
```

---

# 1. Qu'est-ce qu'ASP.NET Core ?

ASP.NET Core est le framework web de .NET.

Il permet de construire notamment :

```text
Web APIs
MVC applications
Minimal APIs
Razor Pages
SignalR applications
```

Il fournit notamment :

```text
HTTP pipeline
routing
middleware
dependency injection
configuration
logging
authentication / authorization
model binding
validation
```

Mental model :

```text
.NET
 ↓
ASP.NET Core
 ↓
application web
```

### Question d'entretien

**Quelle différence entre .NET et ASP.NET Core ?**

> .NET est la plateforme générale. ASP.NET Core est le framework web construit sur .NET et fournit les mécanismes nécessaires pour développer des applications HTTP.

---

# 2. Une API REST

Une API permet à des applications de communiquer.

Exemple :

```text
Frontend
   ↓ HTTP
ASP.NET Core API
   ↓
Database
```

Une API REST utilise généralement des ressources et des méthodes HTTP.

Exemple :

```http
GET /api/users
POST /api/users
GET /api/users/42
PUT /api/users/42
DELETE /api/users/42
```

---

# 3. Les méthodes HTTP

Les principales :

```text
GET
→ récupérer

POST
→ créer / déclencher une opération

PUT
→ remplacer une ressource

PATCH
→ modifier partiellement

DELETE
→ supprimer
```

Il faut connaître notamment la notion d'idempotence.

---

# 4. Idempotence

Une opération idempotente produit le même état final lorsqu'elle est répétée.

Exemple conceptuel :

```http
PUT /users/42
```

avec le même contenu plusieurs fois devrait laisser la ressource dans le même état final.

`GET`, `PUT` et `DELETE` sont généralement considérés comme idempotents selon la sémantique HTTP.

`POST` n'est généralement pas idempotent.

---

# 5. Status codes

Les catégories :

```text
1xx
→ information

2xx
→ succès

3xx
→ redirection

4xx
→ erreur côté client

5xx
→ erreur côté serveur
```

Les plus importants pour une API :

```text
200 OK
201 Created
204 No Content
400 Bad Request
401 Unauthorized
403 Forbidden
404 Not Found
409 Conflict
422 Unprocessable Content
500 Internal Server Error
```

---

# 6. `401` vs `403`

Question très fréquente.

### 401 Unauthorized

Le client n'est pas authentifié correctement.

Mental model :

```text
"Qui es-tu ?"
```

### 403 Forbidden

Le client est identifié mais n'a pas le droit d'effectuer l'action.

Mental model :

```text
"Je sais qui tu es,
mais tu n'as pas le droit."
```

---

# 7. Controller

Exemple :

```csharp
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    [HttpGet]
    public IActionResult GetUsers()
    {
        return Ok();
    }
}
```

Le controller représente généralement une couche d'entrée HTTP.

Il ne devrait pas contenir toute la logique métier.

Mental model :

```text
Controller
→ HTTP

Service
→ logique applicative

Repository / DbContext
→ accès aux données
```

---

# 8. `ControllerBase`

Pour une Web API, on utilise généralement :

```csharp
ControllerBase
```

plutôt que :

```csharp
Controller
```

`Controller` fournit notamment les fonctionnalités liées aux vues MVC.

`ControllerBase` fournit les fonctionnalités nécessaires aux APIs sans la couche Razor View.

---

# 9. `[ApiController]`

L'attribut :

```csharp
[ApiController]
```

active plusieurs comportements utiles aux Web APIs.

Il permet notamment une meilleure gestion du model binding et de la validation.

Un comportement important est la réponse automatique lorsqu'un modèle est invalide.

Mental model :

```text
Request
 ↓
binding
 ↓
validation
 ↓
modèle invalide
 ↓
réponse 400 automatique
```

---

# 10. Routing

Le routing détermine quel endpoint doit traiter une requête.

Exemple :

```csharp
[HttpGet("{id:int}")]
public IActionResult Get(int id)
{
    ...
}
```

Pour :

```http
GET /api/users/42
```

le framework peut sélectionner cette action.

Mental model :

```text
HTTP method
+
URL
+
constraints
      ↓
endpoint
```

---

# 11. Attribute routing

Exemple :

```csharp
[Route("api/users")]
public class UsersController : ControllerBase
{
    [HttpGet("{id}")]
    public IActionResult Get(int id)
    {
        ...
    }
}
```

Route :

```http
GET /api/users/42
```

Les attributs décrivent directement les routes.

---

# 12. Route constraints

Exemple :

```csharp
[HttpGet("{id:int}")]
```

Le `:int` indique que le segment doit correspondre à un entier.

Autres exemples :

```text
:int
:guid
:bool
:min(...)
:max(...)
:length(...)
```

Cela permet de rendre les routes plus précises.

---

# 13. Model Binding

Le model binding transforme les données HTTP en objets ou paramètres C#.

Exemple :

```http
GET /users/42
```

avec :

```csharp
public IActionResult Get(int id)
```

ASP.NET Core récupère :

```text
"42"
 ↓
int id
```

Le binding peut utiliser différentes sources :

```text
route
query string
headers
body
form
```

---

# 14. `[FromRoute]`

Permet d'indiquer explicitement que la valeur vient de la route.

```csharp
public IActionResult Get(
    [FromRoute] int id)
{
    ...
}
```

URL :

```http
GET /users/42
```

---

# 15. `[FromQuery]`

Exemple :

```http
GET /users?page=2&pageSize=20
```

Code :

```csharp
public IActionResult Get(
    [FromQuery] int page,
    [FromQuery] int pageSize)
{
    ...
}
```

Les paramètres viennent de la query string.

---

# 16. `[FromBody]`

Utilisé pour récupérer les données du body HTTP.

Exemple JSON :

```json
{
  "name": "Ahlame",
  "email": "test@example.com"
}
```

Code :

```csharp
public IActionResult Create(
    [FromBody] CreateUserDto dto)
{
    ...
}
```

ASP.NET Core utilise un input formatter, généralement JSON via `System.Text.Json`, pour désérialiser le contenu vers le DTO.

---

# 17. `[FromHeader]`

Permet de récupérer une valeur dans un header HTTP.

```csharp
public IActionResult Get(
    [FromHeader(Name = "X-Correlation-Id")]
    string correlationId)
{
    ...
}
```

---

# 18. DTO

DTO signifie :

```text
Data Transfer Object
```

Exemple :

```csharp
public class CreateUserDto
{
    public string Name { get; set; } = "";
    public string Email { get; set; } = "";
}
```

Le DTO définit les données exposées ou reçues par l'API.

---

# 19. Pourquoi utiliser des DTO ?

Éviter de recevoir directement une entité EF Core :

```csharp
public IActionResult Create(User user)
```

et préférer :

```csharp
public IActionResult Create(CreateUserDto dto)
```

Avantages :

```text
contrôle du contrat HTTP
séparation API / domaine
éviter d'exposer des propriétés internes
réduire les risques de mass assignment
validation adaptée au scénario
```

Mental model :

```text
HTTP
 ↓
DTO
 ↓
mapping
 ↓
Domain Entity
```

---

# 20. Validation

Exemple :

```csharp
public class CreateUserDto
{
    [Required]
    [EmailAddress]
    public string Email { get; set; } = "";
}
```

La validation permet de vérifier que les données respectent les règles attendues.

Avec `[ApiController]`, un modèle invalide peut entraîner automatiquement une réponse HTTP 400.

---

# 21. Validation métier vs validation de forme

Il faut distinguer :

### Validation de données

```text
Email obligatoire
Age numérique
Longueur maximale
```

### Validation métier

```text
un compte déjà existant ne peut pas être recréé
un utilisateur suspendu ne peut pas effectuer telle opération
```

La deuxième catégorie appartient généralement à la logique applicative/métier plutôt qu'à de simples attributs de validation.

---

# 22. ModelState

ASP.NET Core utilise notamment `ModelState` pour représenter l'état du binding et de la validation.

Exemple :

```csharp
if (!ModelState.IsValid)
{
    return BadRequest(ModelState);
}
```

Avec `[ApiController]`, cette vérification manuelle n'est généralement plus nécessaire pour la validation standard, car le framework peut produire automatiquement la réponse.

---

# 23. Dependency Injection dans ASP.NET Core

Exemple :

```csharp
builder.Services.AddScoped<IUserService, UserService>();
```

Puis :

```csharp
public UsersController(IUserService userService)
{
    _userService = userService;
}
```

Le controller ne fait pas :

```csharp
new UserService(...)
```

Le conteneur fournit la dépendance.

---

# 24. Middleware

Le middleware constitue le pipeline HTTP.

Exemple :

```csharp
app.UseHttpsRedirection();

app.UseAuthentication();

app.UseAuthorization();

app.MapControllers();
```

Mental model :

```text
Request
 ↓
HTTPS redirection
 ↓
Authentication
 ↓
Authorization
 ↓
Endpoint
```

---

# 25. Middleware vs Filter

Très bonne question d'entretien.

### Middleware

Agit dans le pipeline HTTP global.

Peut s'appliquer à de nombreuses requêtes.

### Filter

S'applique dans le pipeline MVC/controller à des niveaux plus ciblés.

Mental model :

```text
HTTP pipeline
   ↓
Middleware
   ↓
MVC / Endpoint processing
   ↓
Filters
   ↓
Action
```

Les deux sont utiles mais ne ciblent pas exactement le même niveau.

---

# 26. Authentication

L'authentification répond à :

```text
Qui est l'utilisateur ?
```

Exemple avec JWT :

```text
Authorization: Bearer <token>
```

Le système vérifie notamment le token et construit l'identité de l'utilisateur.

---

# 27. Authorization

L'autorisation répond à :

```text
Que peut faire cet utilisateur ?
```

Exemple :

```csharp
[Authorize]
public IActionResult GetProfile()
{
    ...
}
```

Ou avec un rôle :

```csharp
[Authorize(Roles = "Admin")]
```

Mental model :

```text
Authentication
→ identité

Authorization
→ permissions
```

---

# 28. Claims

Une claim représente une information sur l'utilisateur.

Exemple conceptuel :

```text
sub = 42
email = user@example.com
role = Admin
```

On peut récupérer des claims depuis :

```csharp
User.Claims
```

Les claims sont utilisées notamment par les mécanismes d'autorisation.

---

# 29. `[Authorize]`

Exemple :

```csharp
[Authorize]
[HttpGet]
public IActionResult GetPrivateData()
{
    ...
}
```

L'accès nécessite une identité authentifiée selon la configuration du système d'authentification.

Pour un rôle :

```csharp
[Authorize(Roles = "Admin")]
```

---

# 30. Policy-based authorization

Au lieu de multiplier les rôles, on peut définir des policies.

Exemple conceptuel :

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("CanManageUsers", policy =>
        policy.RequireClaim("permission", "users.manage"));
});
```

Puis :

```csharp
[Authorize(Policy = "CanManageUsers")]
```

Mental model :

```text
Role
→ catégorie d'utilisateur

Policy
→ règle d'autorisation
```

Les policies permettent des règles plus flexibles.

---

# 31. JWT

JWT signifie :

```text
JSON Web Token
```

Un JWT contient notamment :

```text
Header
Payload
Signature
```

Format :

```text
xxxxx.yyyyy.zzzzz
```

Le token est signé afin que le serveur puisse vérifier son intégrité selon le mécanisme configuré.

Attention :

> Un JWT signé n'est pas automatiquement un contenu chiffré.

Il ne faut donc pas mettre des informations secrètes dans le payload simplement parce qu'il est signé.

---

# 32. CORS

CORS signifie :

```text
Cross-Origin Resource Sharing
```

Il concerne les requêtes effectuées depuis une origine vers une autre origine.

Exemple :

```text
Angular
http://localhost:4200

API
https://localhost:7000
```

Le navigateur applique des restrictions de sécurité liées aux origines.

ASP.NET Core permet de configurer les politiques CORS.

---

# 33. CORS n'est pas une authentification

CORS répond principalement à :

```text
"Cette origine est-elle autorisée par la politique du serveur à effectuer ce type de requête depuis un navigateur ?"
```

Il ne remplace pas :

```text
JWT
cookies
authentication
authorization
```

---

# 34. Minimal APIs vs Controllers

ASP.NET Core permet deux styles.

### Controllers

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
}
```

### Minimal APIs

```csharp
app.MapGet("/users", () =>
{
    ...
});
```

Minimal APIs peuvent être pratiques pour des APIs simples.

Les controllers peuvent être intéressants pour des applications ayant :

```text
beaucoup d'endpoints
filtres
conventions MVC
organisation par controllers
besoins plus structurés
```

Il n'y a pas une réponse universelle : le choix dépend du contexte.

---

# 35. `IActionResult` vs `ActionResult<T>`

### `IActionResult`

```csharp
public IActionResult Get()
{
    return Ok(user);
}
```

Permet différents types de résultats HTTP.

### `ActionResult<T>`

```csharp
public ActionResult<UserDto> Get()
{
    return Ok(user);
}
```

Indique également le type attendu du résultat métier.

Exemple :

```csharp
public ActionResult<UserDto> Get(int id)
{
    var user = ...

    if (user == null)
        return NotFound();

    return user;
}
```

---

# 36. `Ok()`, `CreatedAtAction()`, `NoContent()`

Pour une API REST, les résultats doivent refléter correctement l'opération.

### GET réussi

```csharp
return Ok(user);
```

→ `200 OK`

### Création

```csharp
return CreatedAtAction(
    nameof(Get),
    new { id = user.Id },
    user);
```

→ `201 Created`

### Suppression / mise à jour sans contenu

```csharp
return NoContent();
```

→ `204 No Content`

---

# 37. Global exception handling

Il est préférable d'éviter de mettre :

```csharp
try/catch
```

dans chaque controller pour gérer les erreurs inattendues.

On peut centraliser la gestion des exceptions avec un middleware ou le mécanisme de gestion des exceptions d'ASP.NET Core.

Mental model :

```text
exception
 ↓
central exception handling
 ↓
HTTP error response
```

Cela permet d'avoir une réponse cohérente.

---

# 38. Problem Details

ASP.NET Core peut utiliser le format :

```text
Problem Details
```

pour représenter les erreurs HTTP de manière standardisée.

Exemple conceptuel :

```json
{
  "type": "...",
  "title": "Validation failed",
  "status": 400,
  "detail": "...",
  "instance": "..."
}
```

L'objectif est de fournir un format d'erreur exploitable par les clients.

---

# 39. HTTPS

Une API publique doit généralement être servie via HTTPS.

HTTPS protège notamment :

```text
confidentialité
intégrité
authenticité du serveur
```

C'est particulièrement important pour :

```text
credentials
JWT
cookies
données personnelles
```

---

# 40. Swagger / OpenAPI

OpenAPI décrit le contrat d'une API.

Swagger UI peut fournir une interface permettant notamment de visualiser et tester les endpoints.

Exemple :

```text
GET /api/users
POST /api/users
```

avec :

```text
schemas
parameters
responses
authentication
```

---

# 41. Rate limiting

Le rate limiting limite le nombre de requêtes autorisées sur une période donnée.

Exemple conceptuel :

```text
100 requests / minute
```

Cela peut aider contre :

```text
abus
surcharge
certaines attaques automatisées
```

Il ne remplace pas une stratégie complète de sécurité.

---

# 42. Health checks

Les health checks permettent de vérifier l'état de l'application ou de certaines dépendances.

Exemple conceptuel :

```http
GET /health
```

On peut vérifier :

```text
API
Database
services externes
```

C'est particulièrement utile dans les environnements cloud et orchestrés.

---

# 43. Configuration d'une API moderne

Une API ASP.NET Core peut suivre ce chemin :

```text
Program.cs
    ↓
Configuration
    ↓
DI
    ↓
Middleware
    ↓
Authentication
    ↓
Authorization
    ↓
Routing
    ↓
Controller / Endpoint
    ↓
Application Service
    ↓
EF Core
```

Comprendre cette chaîne est beaucoup plus important que mémoriser chaque méthode individuellement.

---

# 44. Question d'entretien : que se passe-t-il avec un POST JSON ?

Requête :

```http
POST /api/users
Content-Type: application/json
```

Body :

```json
{
  "name": "Ahlame",
  "email": "test@example.com"
}
```

Endpoint :

```csharp
[HttpPost]
public ActionResult<UserDto> Create(CreateUserDto dto)
{
    ...
}
```

Chemin conceptuel :

```text
HTTP request
 ↓
routing
 ↓
endpoint sélectionné
 ↓
body JSON
 ↓
deserialization
 ↓
CreateUserDto
 ↓
validation
 ↓
controller
 ↓
service
 ↓
database
 ↓
HTTP response
```

---

# 45. Question d'entretien : pourquoi ne pas mettre la logique métier dans le controller ?

Un controller doit principalement gérer le transport HTTP.

Si on met toute la logique dedans :

```text
Controller
 ├── validation
 ├── business rules
 ├── database
 ├── mapping
 ├── email
 └── logging
```

il devient difficile à :

```text
tester
maintenir
réutiliser
faire évoluer
```

On préfère séparer les responsabilités.

```text
Controller
    ↓
Application Service
    ↓
Infrastructure
```

---

# 46. Question d'entretien : pourquoi les DTO sont importants ?

Réponse :

> Les DTO permettent de contrôler le contrat de l'API et de séparer les données exposées par l'API des entités internes. Ils permettent également d'avoir des règles de validation adaptées au cas d'utilisation et de limiter l'exposition de propriétés sensibles.

---

# 47. Question d'entretien : Authentication vs Authorization

Réponse courte :

> L'authentification détermine l'identité du client. L'autorisation détermine si cette identité possède les droits nécessaires pour accéder à une ressource ou effectuer une action.

À mémoriser :

```text
Authentication
→ Who are you?

Authorization
→ What are you allowed to do?
```

---

# 48. Question d'entretien : Middleware vs Controller

Réponse :

> Le middleware intervient dans le pipeline HTTP et peut agir sur de nombreuses requêtes avant et après l'appel au composant suivant. Le controller intervient lorsqu'un endpoint MVC correspondant à la requête a été sélectionné.

---

# 49. Question d'entretien : 401 vs 403

Réponse :

> 401 signifie généralement que l'authentification est absente ou invalide. 403 signifie que l'utilisateur est authentifié mais que l'accès lui est refusé.

---

# 50. Question d'entretien : pourquoi utiliser `ActionResult<T>` ?

Réponse :

> Il permet d'indiquer le type de données attendu tout en autorisant différents résultats HTTP comme `NotFound()`, `BadRequest()` ou `Ok()`.

---

# 51. Pièges classiques

## Mettre les entités EF directement dans l'API

Ce n'est pas forcément toujours incorrect, mais cela augmente le couplage entre le contrat HTTP et le modèle de persistance.

Préférer généralement :

```text
Request DTO
 ↓
mapping
 ↓
Entity
```

et :

```text
Entity
 ↓
mapping
 ↓
Response DTO
```

---

## Confondre authentication et authorization

```text
Authentication
→ identité

Authorization
→ permission
```

---

## Mettre un secret JWT dans `appsettings.json` versionné

Mauvaise pratique.

Utiliser un mécanisme adapté :

```text
User Secrets
Environment Variables
Key Vault
Secret Manager
```

selon l'environnement.

---

## Croire que CORS protège l'API contre tous les appels externes

Faux.

CORS est principalement une politique appliquée par les navigateurs.

Un client non navigateur peut toujours appeler directement l'API.

Il faut donc une vraie sécurité côté serveur :

```text
Authentication
Authorization
Validation
Rate limiting
etc.
```

---

## Utiliser `try/catch` partout

Mieux vaut centraliser la gestion des exceptions inattendues.

---

# 52. Mini simulation d'entretien

### Recruteur

**Décris-moi le pipeline d'une requête ASP.NET Core.**

### Réponse

> La requête arrive dans le pipeline HTTP et traverse les middleware dans l'ordre de configuration. Le routing permet ensuite de déterminer l'endpoint. L'authentification établit l'identité et l'autorisation vérifie les permissions. Pour un controller, le model binding transforme les données HTTP en paramètres ou DTO et la validation vérifie le modèle. Le controller appelle ensuite la couche applicative, qui peut utiliser EF Core ou d'autres services. Le résultat est transformé en réponse HTTP et remonte à travers le pipeline.

---

### Recruteur

**Pourquoi utiliser des DTO plutôt que les entités directement ?**

### Réponse

> Pour découpler le contrat de l'API du modèle interne ou de persistance, contrôler les données exposées et adapter la validation à chaque cas d'utilisation.

---

### Recruteur

**Quelle différence entre middleware et filter ?**

### Réponse

> Le middleware travaille au niveau du pipeline HTTP global, tandis que les filters sont intégrés au pipeline MVC et permettent d'intervenir plus près de l'exécution des controllers et actions.

---

### Recruteur

**Comment sécuriserais-tu une API ?**

### Réponse

> Je commencerais par HTTPS, une authentification robuste, une autorisation basée sur les rôles ou policies, la validation stricte des entrées, des DTO, une gestion sécurisée des secrets, du rate limiting, une gestion centralisée des erreurs et une journalisation adaptée. J'ajouterais ensuite les mesures nécessaires selon le contexte et les risques.

---

# 53. Checklist ASP.NET Core

```text
[ ] ASP.NET Core vs .NET
[ ] Web API
[ ] REST
[ ] HTTP methods
[ ] Idempotence
[ ] Status codes
[ ] 401 vs 403
[ ] ControllerBase
[ ] ApiController
[ ] Routing
[ ] Route constraints
[ ] Model Binding
[ ] FromRoute
[ ] FromQuery
[ ] FromBody
[ ] FromHeader
[ ] DTO
[ ] Validation
[ ] ModelState
[ ] Dependency Injection
[ ] Middleware
[ ] Filters
[ ] Authentication
[ ] Authorization
[ ] Claims
[ ] Roles
[ ] Policies
[ ] JWT
[ ] CORS
[ ] Minimal APIs
[ ] IActionResult
[ ] ActionResult<T>
[ ] Problem Details
[ ] Exception handling
[ ] HTTPS
[ ] OpenAPI / Swagger
[ ] Rate limiting
[ ] Health checks
```

# À retenir

```text
ASP.NET Core
→ framework web .NET

Controller
→ entrée HTTP

Middleware
→ pipeline HTTP

Routing
→ sélection de l'endpoint

Model Binding
→ HTTP → objets/paramètres C#

DTO
→ contrat de données

Validation
→ vérifier les données

Authentication
→ identité

Authorization
→ permissions

Claims
→ informations sur l'identité

Policy
→ règle d'autorisation

JWT
→ token signé transportant des claims

CORS
→ politique d'origine côté navigateur

Problem Details
→ format standardisé d'erreur HTTP

OpenAPI
→ description du contrat de l'API
```

## Phrase à mémoriser

> **Dans ASP.NET Core, une requête traverse un pipeline de middleware, est associée à un endpoint par le routing, puis ses données sont bindées et validées avant que la couche applicative traite la demande ; l'authentification établit l'identité et l'autorisation décide si cette identité peut effectuer l'action.**
