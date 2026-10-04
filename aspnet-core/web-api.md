# Web API avec ASP.NET Core

## 1. Définition

Une **Web API ASP.NET Core** est une application HTTP qui expose des fonctionnalités accessibles par des clients.

Le client peut être :

- une application Angular ;
- une application mobile ;
- une autre API ;
- un programme C# ;
- Postman ;
- un navigateur.

Le client communique avec l'API grâce au protocole HTTP.

Exemple :

```text
Angular
   |
   | GET /api/hotels
   v
ASP.NET Core API
   |
   v
Database
```

L'API reçoit une requête, exécute la logique nécessaire et renvoie une réponse HTTP.

---

# 2. Le modèle Request / Response

Une API REST fonctionne principalement autour de :

```text
Request
   |
   v
API
   |
   v
Response
```

Exemple :

```http
GET /api/hotels/5
```

L'API peut répondre :

```http
200 OK
Content-Type: application/json
```

avec :

```json
{
  "id": 5,
  "name": "Hotel Central"
}
```

Le principe fondamental :

> Une API transforme une requête HTTP en une réponse HTTP.

---

# 3. Les principales méthodes HTTP

Les méthodes les plus utilisées sont :

```text
GET
POST
PUT
PATCH
DELETE
```

Elles expriment généralement l'intention du client.

| Méthode | Utilisation courante |
|---|---|
| GET | récupérer une ressource |
| POST | créer une ressource |
| PUT | remplacer/modifier une ressource |
| PATCH | modifier partiellement une ressource |
| DELETE | supprimer une ressource |

Exemples :

```http
GET /api/hotels
```

Récupérer les hôtels.

```http
GET /api/hotels/5
```

Récupérer l'hôtel 5.

```http
POST /api/hotels
```

Créer un hôtel.

```http
PUT /api/hotels/5
```

Modifier l'hôtel 5.

```http
DELETE /api/hotels/5
```

Supprimer l'hôtel 5.

---

# 4. Créer un contrôleur

Avec ASP.NET Core :

```csharp
[ApiController]
[Route("api/[controller]")]
public class HotelsController : ControllerBase
{
}
```

Deux éléments sont particulièrement importants :

```csharp
[ApiController]
```

et :

```csharp
[Route("api/[controller]")]
```

---

# 5. `ControllerBase`

Pour une Web API, on utilise généralement :

```csharp
ControllerBase
```

Exemple :

```csharp
public class HotelsController : ControllerBase
{
}
```

`ControllerBase` fournit notamment des fonctionnalités utiles pour retourner des réponses HTTP.

Par exemple :

```csharp
Ok()
BadRequest()
NotFound()
CreatedAtAction()
NoContent()
Unauthorized()
Forbid()
```

Une API n'a généralement pas besoin de :

```csharp
Controller
```

qui contient en plus les fonctionnalités liées aux vues MVC.

### Mental model

```text
Controller
    -> MVC avec Views

ControllerBase
    -> API
```

---

# 6. `[ApiController]`

L'attribut :

```csharp
[ApiController]
```

active plusieurs comportements adaptés aux API.

Il améliore notamment :

- le model binding ;
- la validation automatique ;
- la gestion des erreurs de validation ;
- la détection de certains problèmes de paramètres.

Exemple :

```csharp
[ApiController]
[Route("api/[controller]")]
public class HotelsController : ControllerBase
{
}
```

### À retenir

> `[ApiController]` indique que le contrôleur est conçu pour une API HTTP et active des comportements spécifiques aux Web API.

---

# 7. Routing

Avec :

```csharp
[Route("api/[controller]")]
```

ASP.NET Core utilise le nom du contrôleur.

Pour :

```csharp
HotelsController
```

on obtient :

```text
/api/hotels
```

Puis :

```csharp
[HttpGet]
public IActionResult GetHotels()
{
    return Ok();
}
```

correspond à :

```http
GET /api/hotels
```

---

# 8. Route avec un identifiant

Exemple :

```csharp
[HttpGet("{id}")]
public IActionResult GetHotel(int id)
{
    return Ok();
}
```

La route devient :

```http
GET /api/hotels/5
```

La valeur :

```text
5
```

est liée au paramètre :

```csharp
int id
```

Mentalement :

```text
/api/hotels/{id}
          |
          v
       int id
```

---

# 9. Retourner `Ok`

Exemple :

```csharp
[HttpGet]
public IActionResult GetHotels()
{
    var hotels = _service.GetHotels();

    return Ok(hotels);
}
```

`Ok()` produit généralement :

```http
200 OK
```

avec les données dans le corps de la réponse.

---

# 10. `NotFound`

Si une ressource n'existe pas :

```csharp
return NotFound();
```

répond généralement :

```http
404 Not Found
```

Exemple :

```csharp
[HttpGet("{id}")]
public IActionResult GetHotel(int id)
{
    var hotel = _service.GetHotel(id);

    if (hotel is null)
    {
        return NotFound();
    }

    return Ok(hotel);
}
```

---

# 11. `BadRequest`

Si la requête est invalide :

```csharp
return BadRequest();
```

répond :

```http
400 Bad Request
```

On peut également fournir des informations :

```csharp
return BadRequest("Invalid hotel data.");
```

Dans une API moderne, on peut aussi utiliser des mécanismes structurés comme `ProblemDetails`.

---

# 12. `CreatedAtAction`

Après la création d'une ressource, une API REST peut retourner :

```http
201 Created
```

Exemple :

```csharp
[HttpPost]
public IActionResult CreateHotel(CreateHotelDto dto)
{
    var hotel = _service.Create(dto);

    return CreatedAtAction(
        nameof(GetHotel),
        new { id = hotel.Id },
        hotel);
}
```

Cette réponse indique :

```text
La ressource a été créée.
```

et peut également fournir l'URL permettant de retrouver la ressource.

---

# 13. `NoContent`

Pour une opération réussie qui n'a pas besoin de renvoyer de contenu :

```csharp
return NoContent();
```

répond généralement :

```http
204 No Content
```

C'est courant après une mise à jour ou une suppression réussie.

Exemple :

```csharp
[HttpDelete("{id}")]
public IActionResult DeleteHotel(int id)
{
    _service.Delete(id);

    return NoContent();
}
```

---

# 14. POST et DTO

Éviter de recevoir directement une entité EF Core dans une API.

Mauvais exemple :

```csharp
[HttpPost]
public IActionResult Create(Hotel hotel)
{
}
```

Préférer un DTO :

```csharp
[HttpPost]
public IActionResult Create(CreateHotelDto dto)
{
}
```

Le DTO représente les données que l'API accepte.

Exemple :

```csharp
public class CreateHotelDto
{
    public string Name { get; set; } = string.Empty;

    public string Address { get; set; } = string.Empty;
}
```

Mentalement :

```text
HTTP Request
     |
     v
CreateHotelDto
     |
     v
Service
     |
     v
Hotel entity
```

---

# 15. Pourquoi utiliser des DTO ?

Les DTO permettent notamment de :

- contrôler les données exposées ;
- éviter le mass assignment ;
- séparer le contrat HTTP de l'entité de base de données ;
- faire évoluer l'API indépendamment de la structure interne ;
- éviter d'exposer des propriétés sensibles.

Exemple :

Entité :

```csharp
public class User
{
    public int Id { get; set; }

    public string Email { get; set; } = string.Empty;

    public string PasswordHash { get; set; } = string.Empty;

    public bool IsAdmin { get; set; }
}
```

DTO de réponse :

```csharp
public class UserDto
{
    public int Id { get; set; }

    public string Email { get; set; } = string.Empty;
}
```

On ne renvoie pas :

```text
PasswordHash
IsAdmin
```

si le client n'a pas besoin de ces informations.

---

# 16. Model Binding

ASP.NET Core doit transformer les données HTTP en paramètres C#.

C'est le rôle du **Model Binding**.

Exemple :

```http
GET /api/hotels/5
```

avec :

```csharp
public IActionResult GetHotel(int id)
```

ASP.NET Core transforme :

```text
"5"
```

en :

```csharp
int id = 5;
```

Le model binding peut également récupérer des données depuis :

- route ;
- query string ;
- body ;
- headers ;
- form data.

---

# 17. Query String

Exemple :

```http
GET /api/hotels?city=Mons
```

Code :

```csharp
[HttpGet]
public IActionResult GetHotels(string city)
{
    // city = "Mons"
    return Ok();
}
```

La valeur :

```text
Mons
```

vient de la query string.

---

# 18. Route Parameter

Exemple :

```http
GET /api/hotels/5
```

Code :

```csharp
[HttpGet("{id}")]
public IActionResult GetHotel(int id)
{
    // id = 5
    return Ok();
}
```

La valeur vient de la route.

---

# 19. Request Body

Pour un POST :

```http
POST /api/hotels
Content-Type: application/json
```

Body :

```json
{
  "name": "Hotel Central",
  "address": "Mons"
}
```

Code :

```csharp
[HttpPost]
public IActionResult Create(CreateHotelDto dto)
{
    return Ok(dto);
}
```

ASP.NET Core désérialise le JSON vers :

```csharp
CreateHotelDto
```

---

# 20. `[FromRoute]`

On peut explicitement indiquer la source :

```csharp
public IActionResult GetHotel(
    [FromRoute] int id)
{
}
```

Cela signifie :

```text
id <- route
```

---

# 21. `[FromQuery]`

```csharp
public IActionResult Search(
    [FromQuery] string city)
{
}
```

Cela signifie :

```text
city <- query string
```

Exemple :

```http
GET /api/hotels?city=Mons
```

---

# 22. `[FromBody]`

```csharp
public IActionResult Create(
    [FromBody] CreateHotelDto dto)
{
}
```

Cela signifie :

```text
dto <- HTTP body
```

Pour un objet complexe dans une API, ASP.NET Core peut généralement déduire le body avec `[ApiController]`, mais l'attribut peut rendre l'intention explicite.

---

# 23. Validation

On peut utiliser des annotations :

```csharp
public class CreateHotelDto
{
    [Required]
    public string Name { get; set; } = string.Empty;

    [StringLength(100)]
    public string Address { get; set; } = string.Empty;
}
```

Avec :

```csharp
[ApiController]
```

les erreurs de validation peuvent provoquer automatiquement une réponse :

```http
400 Bad Request
```

Le contrôleur n'a donc pas besoin de faire manuellement tous les tests de validation de base.

---

# 24. Injection de services dans un contrôleur

Un contrôleur peut recevoir un service via DI :

```csharp
public class HotelsController : ControllerBase
{
    private readonly IHotelService _service;

    public HotelsController(IHotelService service)
    {
        _service = service;
    }
}
```

Le contrôleur ne crée pas :

```csharp
new HotelService()
```

Il reçoit la dépendance.

Cela permet :

```text
Controller
    |
    v
IHotelService
    |
    v
HotelService
```

---

# 25. Le contrôleur ne devrait pas contenir toute la logique

Mauvaise architecture :

```text
Controller
 |
 +-- validation complexe
 +-- règles métier
 +-- accès DB
 +-- mapping
 +-- emails
 +-- calculs
```

Préférer :

```text
Controller
    |
    v
Application/Service
    |
    +-- logique métier
    |
    v
Repository / EF Core
```

Le contrôleur doit principalement faire le lien entre HTTP et l'application.

---

# 26. `ActionResult<T>`

Au lieu de :

```csharp
public IActionResult GetHotel(int id)
```

on peut utiliser :

```csharp
public ActionResult<HotelDto> GetHotel(int id)
```

Cela exprime davantage le type de données attendu.

Exemple :

```csharp
[HttpGet("{id}")]
public ActionResult<HotelDto> GetHotel(int id)
{
    var hotel = _service.GetHotel(id);

    if (hotel is null)
    {
        return NotFound();
    }

    return Ok(hotel);
}
```

Le résultat peut être :

```text
HotelDto
```

ou une réponse HTTP comme :

```text
404 Not Found
```

---

# 27. API asynchrone

Dans une API moderne, les accès I/O sont généralement asynchrones.

Exemple :

```csharp
[HttpGet]
public async Task<ActionResult<List<HotelDto>>> GetHotels(
    CancellationToken cancellationToken)
{
    var hotels =
        await _service.GetHotelsAsync(
            cancellationToken);

    return Ok(hotels);
}
```

On retrouve ici plusieurs concepts :

```text
async/await
DI
CancellationToken
DTO
HTTP response
```

---

# 28. `CancellationToken` dans une API

ASP.NET Core peut fournir un `CancellationToken` lié à la requête.

Exemple :

```csharp
[HttpGet]
public async Task<IActionResult> Get(
    CancellationToken cancellationToken)
{
    var result =
        await _service.GetAsync(
            cancellationToken);

    return Ok(result);
}
```

Le token peut ensuite être propagé :

```text
Controller
    |
    v
Service
    |
    v
Repository
    |
    v
EF Core
```

Cela permet d'annuler les opérations lorsque la requête n'est plus nécessaire.

---

# 29. Codes HTTP importants

Quelques codes à connaître :

| Code | Signification |
|---|---|
| 200 | OK |
| 201 | Created |
| 204 | No Content |
| 400 | Bad Request |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not Found |
| 409 | Conflict |
| 422 | Unprocessable Content |
| 500 | Internal Server Error |

Il faut distinguer :

```text
401
```

de :

```text
403
```

### 401 Unauthorized

Le client n'est pas correctement authentifié.

### 403 Forbidden

Le client est identifié mais n'a pas l'autorisation nécessaire.

Mentalement :

```text
401 -> Qui es-tu ?
403 -> Tu es identifié, mais tu n'as pas le droit.
```

---

# 30. `ProblemDetails`

Pour les erreurs HTTP, ASP.NET Core peut utiliser le format :

```text
ProblemDetails
```

Il permet de représenter une erreur de manière standardisée.

Exemple conceptuel :

```json
{
  "status": 404,
  "title": "Not Found",
  "detail": "Hotel not found."
}
```

L'intérêt est d'avoir des réponses d'erreur cohérentes.

---

# 31. Swagger / OpenAPI

Une API ASP.NET Core peut être documentée avec **OpenAPI**.

La documentation permet de décrire :

```text
Endpoints
Methods
Parameters
Request bodies
Responses
Schemas
Authentication
```

Un outil comme Swagger UI peut ensuite permettre de tester les endpoints.

Mentalement :

```text
ASP.NET Core API
      |
      v
OpenAPI description
      |
      v
Swagger UI / autres outils
```

---

# 32. Séparation HTTP / métier

Une bonne architecture évite de faire dépendre le domaine métier directement de HTTP.

Exemple :

```text
Controller
    |
    | HTTP
    v
Service
    |
    | métier
    v
Domain
```

Le service ne devrait généralement pas avoir besoin de connaître :

```csharp
HttpContext
```

pour chaque opération métier.

Cela facilite notamment les tests et la réutilisation.

---

# 33. API REST et ressources

Dans une API REST, les URLs représentent généralement des ressources.

Préférer :

```http
GET /api/hotels
GET /api/hotels/5
POST /api/hotels
DELETE /api/hotels/5
```

plutôt que :

```http
GET /api/getHotels
POST /api/createHotel
GET /api/deleteHotel
```

L'action est principalement exprimée par la méthode HTTP.

Mentalement :

```text
URL -> ressource
HTTP method -> opération
```

---

# 34. Idempotence

Certaines méthodes HTTP sont conçues pour être idempotentes.

Une opération idempotente peut être répétée sans changer le résultat final au-delà de la première application.

Exemple conceptuel :

```http
PUT /api/hotels/5
```

avec exactement les mêmes données peut être envoyé plusieurs fois.

L'état final reste le même.

`POST` n'est généralement pas considéré comme idempotent par défaut.

Cette notion devient importante lorsqu'on travaille avec :

- retries ;
- réseaux instables ;
- systèmes distribués ;
- API externes.

---

# 35. Pagination

Une API qui retourne des milliers d'éléments ne devrait généralement pas tout renvoyer en une seule réponse.

Exemple :

```http
GET /api/hotels?page=1&pageSize=20
```

Le service peut alors récupérer uniquement une partie des données.

Mentalement :

```text
10000 hôtels
      |
      v
Pagination
      |
      v
20 hôtels
```

La pagination protège notamment :

- mémoire ;
- temps de réponse ;
- taille des réponses ;
- base de données ;
- client.

---

# 36. Architecture typique

Une API ASP.NET Core peut être organisée ainsi :

```text
HTTP
 |
 v
Controller
 |
 v
Application Service
 |
 v
Domain
 |
 v
Infrastructure
 |
 v
Database / External API
```

Exemple :

```text
HotelsController
      |
      v
HotelService
      |
      v
HotelRepository
      |
      v
AppDbContext
      |
      v
SQL Server
```

Chaque couche possède une responsabilité différente.

---

# 37. Erreurs fréquentes

## Erreur 1 : mettre `new` partout

```csharp
var service = new HotelService();
```

Préférer la DI.

---

## Erreur 2 : retourner directement les entités EF Core

Préférer des DTO lorsque l'entité ne correspond pas exactement au contrat HTTP.

---

## Erreur 3 : mettre la logique métier dans les contrôleurs

Le contrôleur doit rester relativement mince.

---

## Erreur 4 : renvoyer toujours `200 OK`

Une API doit utiliser des codes HTTP cohérents :

```text
201
204
400
401
403
404
409
500
```

selon la situation.

---

## Erreur 5 : exposer des informations sensibles

Ne jamais renvoyer directement :

```text
PasswordHash
secrets
tokens internes
informations privées inutiles
```

---

## Erreur 6 : ignorer l'annulation

Pour les opérations asynchrones longues, propager le `CancellationToken`.

---

## Erreur 7 : retourner des milliers d'éléments sans pagination

Les grandes collections doivent généralement être paginées.

---

# 38. Schéma global d'une Web API

```text
                 CLIENT
                   |
                   | HTTP
                   v
          +------------------+
          | ASP.NET Pipeline |
          +------------------+
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
             Controller
                   |
                   v
               DTO
                   |
                   v
              Service
                   |
                   v
          Repository / EF Core
                   |
                   v
              Database
                   |
                   v
               Response
                   |
                   v
                 CLIENT
```

---

# 39. Règle mentale

Quand tu vois :

```csharp
[ApiController]
```

pense :

> « Ce contrôleur est traité comme une Web API et bénéficie des comportements spécifiques aux API. »

Quand tu vois :

```csharp
[HttpGet]
```

pense :

> « Cette action répond à une requête HTTP GET. »

Quand tu vois :

```csharp
[HttpGet("{id}")]
```

pense :

> « Une valeur de la route doit être associée au paramètre correspondant. »

Quand tu vois :

```csharp
return Ok(data);
```

pense :

> « Je renvoie une réponse HTTP 200 avec des données. »

Quand tu vois :

```csharp
return NotFound();
```

pense :

> « La ressource demandée n'existe pas. »

Quand tu vois :

```csharp
CreatedAtAction(...)
```

pense :

> « Une nouvelle ressource vient d'être créée et je peux indiquer comment la retrouver. »

---

# 40. À retenir

1. Une Web API ASP.NET Core transforme des requêtes HTTP en réponses HTTP.
2. `ControllerBase` est généralement utilisé pour les APIs.
3. `[ApiController]` active des comportements spécifiques aux Web APIs.
4. Le routing détermine quel endpoint doit traiter une requête.
5. Le Model Binding transforme les données HTTP en paramètres C#.
6. La validation vérifie les données reçues.
7. Les DTO définissent le contrat entre le client et l'API.
8. La logique métier doit généralement être placée dans les services, pas dans les contrôleurs.
9. La DI fournit les services aux contrôleurs.
10. Les méthodes HTTP expriment l'intention sur une ressource.
11. `200`, `201`, `204`, `400`, `401`, `403`, `404`, `409` et `500` sont des codes importants à connaître.
12. `401` signifie principalement problème d'authentification ; `403` signifie problème d'autorisation.
13. `ProblemDetails` permet de standardiser les erreurs HTTP.
14. Les opérations I/O doivent généralement être asynchrones.
15. Le `CancellationToken` doit être propagé lorsque l'opération peut être annulée.
16. Les grandes collections doivent généralement être paginées.
17. Une API REST représente généralement les ressources dans les URLs et utilise les méthodes HTTP pour exprimer les opérations.

---

# Questions d'entretien

### 1. Quelle différence entre `Controller` et `ControllerBase` ?

`Controller` fournit les fonctionnalités MVC avec notamment les vues. `ControllerBase` est destiné aux contrôleurs d'API.

### 2. À quoi sert `[ApiController]` ?

Il indique que le contrôleur est une API et active plusieurs comportements utiles comme la gestion automatique de certaines erreurs de validation et des conventions de model binding.

### 3. Quelle différence entre `IActionResult` et `ActionResult<T>` ?

`IActionResult` représente une réponse HTTP de manière générale. `ActionResult<T>` permet en plus d'exprimer le type de données attendu.

### 4. Pourquoi utiliser des DTO ?

Pour contrôler le contrat HTTP, éviter d'exposer directement les entités internes et limiter notamment les risques de mass assignment.

### 5. Quelle différence entre 401 et 403 ?

401 concerne l'authentification ; 403 signifie que l'utilisateur est identifié mais n'a pas les permissions nécessaires.

### 6. Pourquoi utiliser `CreatedAtAction` après un POST ?

Pour indiquer qu'une ressource a été créée avec le statut 201 et fournir les informations permettant de retrouver cette ressource.

### 7. Pourquoi ne pas mettre toute la logique dans le contrôleur ?

Pour séparer les responsabilités et garder la logique métier testable et réutilisable.

### 8. Pourquoi utiliser `CancellationToken` dans une API ?

Pour permettre aux opérations longues de s'arrêter lorsque la requête HTTP n'est plus nécessaire.

### 9. Pourquoi paginer une API ?

Pour éviter de charger et transférer inutilement de grandes quantités de données.

### 10. Quel est le rôle du Model Binding ?

Transformer les données provenant de la requête HTTP en paramètres et objets C# utilisables par l'action.

---

# Phrase à retenir

> **Une Web API ASP.NET Core fait le lien entre HTTP et l'application : le routing choisit l'endpoint, le binding transforme la requête en objets C#, le contrôleur délègue au métier et l'API renvoie une réponse HTTP adaptée.**
