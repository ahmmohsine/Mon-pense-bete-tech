# Routing en ASP.NET Core

## 1. Définition

Le **routing** est le mécanisme qui détermine quel endpoint doit traiter une requête HTTP.

Exemple :

```http
GET /api/hotels/5
```

ASP.NET Core doit déterminer :

```text
Quelle action doit être exécutée ?
```

Le routing fait donc le lien entre :

```text
URL + HTTP Method
        |
        v
Endpoint
```

Mentalement :

> Le routing répond à la question « Qui doit traiter cette requête ? ».

---

# 2. URL, route et endpoint

Il faut distinguer plusieurs notions.

### URL

Exemple :

```text
https://example.com/api/hotels/5
```

### Route

La route peut être :

```text
/api/hotels/{id}
```

### Endpoint

L'endpoint est la destination concrète sélectionnée par ASP.NET Core.

Par exemple :

```csharp
[HttpGet("{id}")]
public IActionResult GetHotel(int id)
{
    return Ok();
}
```

On peut représenter :

```text
GET /api/hotels/5
        |
        v
/api/hotels/{id}
        |
        v
GetHotel(int id)
```

---

# 3. Attribute Routing

Dans les Web APIs modernes, on utilise très souvent le **attribute routing**.

Exemple :

```csharp
[ApiController]
[Route("api/[controller]")]
public class HotelsController : ControllerBase
{
}
```

Puis :

```csharp
[HttpGet]
public IActionResult GetHotels()
{
    return Ok();
}
```

La route devient :

```http
GET /api/hotels
```

---

# 4. `[Route]`

L'attribut :

```csharp
[Route("api/[controller]")]
```

définit une route de base au niveau du contrôleur.

Pour :

```csharp
HotelsController
```

`[controller]` correspond généralement à :

```text
hotels
```

On obtient :

```text
/api/hotels
```

---

# 5. `[HttpGet]`

```csharp
[HttpGet]
public IActionResult GetHotels()
{
    return Ok();
}
```

Cette action répond à :

```http
GET /api/hotels
```

`HttpGet` indique donc la méthode HTTP acceptée.

---

# 6. Route avec paramètre

Exemple :

```csharp
[HttpGet("{id}")]
public IActionResult GetHotel(int id)
{
    return Ok();
}
```

La route est :

```text
/api/hotels/{id}
```

Une requête :

```http
GET /api/hotels/5
```

produit :

```csharp
id = 5
```

Mentalement :

```text
URL
/api/hotels/5

        |
        v

Template
/api/hotels/{id}

        |
        v

Parameter
id = 5
```

---

# 7. Contraintes de route

ASP.NET Core permet de contraindre les paramètres.

Exemple :

```csharp
[HttpGet("{id:int}")]
public IActionResult GetHotel(int id)
{
    return Ok();
}
```

La route accepte :

```text
/api/hotels/5
```

mais pas :

```text
/api/hotels/abc
```

car :

```text
id:int
```

indique que `id` doit être un entier.

---

# 8. Contraintes courantes

Quelques contraintes :

```text
:int
:long
:guid
:bool
:min(...)
:max(...)
:length(...)
:minlength(...)
:maxlength(...)
:alpha
```

Exemples :

```csharp
[HttpGet("{id:guid}")]
```

ou :

```csharp
[HttpGet("{id:int:min(1)}")]
```

La deuxième route signifie :

```text
id doit être un int
ET
id >= 1
```

---

# 9. Pourquoi utiliser les contraintes ?

Les contraintes permettent de rendre les routes plus précises.

Supposons :

```csharp
[HttpGet("{id}")]
public IActionResult Get(int id)
```

et :

```csharp
[HttpGet("search")]
public IActionResult Search()
```

Une requête :

```http
GET /api/hotels/search
```

pourrait être interprétée comme :

```text
id = "search"
```

selon les routes disponibles.

Avec :

```csharp
[HttpGet("{id:int}")]
```

`search` ne correspond pas au paramètre `id`.

On peut donc avoir :

```text
/api/hotels/search
/api/hotels/5
```

sans ambiguïté.

---

# 10. Route statique vs route paramétrée

Route statique :

```csharp
[HttpGet("search")]
```

correspond à :

```text
/api/hotels/search
```

Route paramétrée :

```csharp
[HttpGet("{id}")]
```

correspond par exemple à :

```text
/api/hotels/5
/api/hotels/12
```

Mentalement :

```text
"search"
    -> texte fixe

"{id}"
    -> valeur variable
```

---

# 11. `[FromRoute]`

On peut indiquer explicitement qu'un paramètre vient de la route :

```csharp
[HttpGet("{id}")]
public IActionResult GetHotel(
    [FromRoute] int id)
{
    return Ok();
}
```

La source est :

```text
Route
```

Exemple :

```http
GET /api/hotels/5
```

donne :

```csharp
id = 5
```

---

# 12. Route et Query String

Il faut distinguer :

```text
/api/hotels/5
```

et :

```text
/api/hotels?id=5
```

Dans le premier cas :

```text
id = route parameter
```

Dans le second :

```text
id = query parameter
```

Exemple :

```csharp
[HttpGet]
public IActionResult GetHotel(
    [FromQuery] int id)
{
    return Ok();
}
```

Requête :

```http
GET /api/hotels?id=5
```

---

# 13. Route vs Query : règle pratique

Une bonne règle pour une API REST :

```text
Route
    -> identifier une ressource

Query
    -> filtrer, rechercher, trier, paginer
```

Exemple :

```http
GET /api/hotels/5
```

signifie :

```text
L'hôtel 5
```

Alors que :

```http
GET /api/hotels?city=Mons
```

signifie :

```text
Les hôtels filtrés par ville
```

---

# 14. Routes imbriquées

On peut représenter une relation entre ressources.

Exemple :

```http
GET /api/hotels/5/rooms
```

La route peut être :

```csharp
[HttpGet("{hotelId}/rooms")]
public IActionResult GetRooms(int hotelId)
{
    return Ok();
}
```

Cela signifie :

```text
Hotel 5
   |
   +-- Rooms
```

Cependant, il ne faut pas créer des URLs inutilement profondes.

---

# 15. Plusieurs paramètres de route

Exemple :

```csharp
[HttpGet("{hotelId}/rooms/{roomId}")]
public IActionResult GetRoom(
    int hotelId,
    int roomId)
{
    return Ok();
}
```

Requête :

```http
GET /api/hotels/5/rooms/12
```

ASP.NET Core associe :

```text
hotelId = 5
roomId  = 12
```

---

# 16. Routes nommées

On peut donner un nom à une route :

```csharp
[HttpGet("{id}", Name = "GetHotel")]
public IActionResult GetHotel(int id)
{
    return Ok();
}
```

Le nom peut être utilisé pour générer une URL ou référencer l'endpoint.

C'est notamment utile avec :

```csharp
CreatedAtRoute(...)
```

---

# 17. `CreatedAtAction` et routing

Exemple :

```csharp
[HttpPost]
public IActionResult Create(CreateHotelDto dto)
{
    var hotel = _service.Create(dto);

    return CreatedAtAction(
        nameof(GetHotel),
        new { id = hotel.Id },
        hotel);
}
```

ASP.NET Core peut utiliser l'action :

```csharp
GetHotel
```

pour construire l'URL de la ressource créée.

Le routing n'est donc pas uniquement utilisé pour recevoir des requêtes.

Il peut aussi servir à générer des URLs.

---

# 18. Convention Routing

ASP.NET Core supporte également le **conventional routing**, notamment dans les applications MVC.

Exemple conceptuel :

```csharp
app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");
```

Une URL :

```text
/Home/Index/5
```

peut alors correspondre à :

```text
HomeController
    |
    +-- Index(int? id)
```

Pour les Web APIs modernes, l'attribute routing est généralement très utilisé.

---

# 19. Attribute Routing vs Conventional Routing

| | Attribute Routing | Conventional Routing |
|---|---|---|
| Définition | attributs sur contrôleurs/actions | modèle global |
| Contrôle | précis | basé sur convention |
| Web API | très courant | moins fréquent |
| MVC avec Views | possible | très courant |

### Mental model

```text
Attribute Routing
    -> la route est proche de l'action

Conventional Routing
    -> une convention globale construit les routes
```

---

# 20. Routing dans le pipeline

Le routing fait partie du pipeline ASP.NET Core.

Conceptuellement :

```text
HTTP Request
      |
      v
Middleware
      |
      v
Routing
      |
      v
Endpoint sélectionné
      |
      v
Endpoint execution
```

Le routing doit déterminer l'endpoint avant son exécution.

---

# 21. `MapControllers`

Dans une Web API :

```csharp
builder.Services.AddControllers();

var app = builder.Build();

app.MapControllers();
```

`MapControllers()` ajoute les endpoints correspondant aux contrôleurs utilisant notamment l'attribute routing.

Exemple :

```csharp
[ApiController]
[Route("api/[controller]")]
public class HotelsController : ControllerBase
{
    [HttpGet]
    public IActionResult Get()
    {
        return Ok();
    }
}
```

Avec :

```csharp
app.MapControllers();
```

l'endpoint peut être exposé.

---

# 22. Route tokens

ASP.NET Core fournit des tokens comme :

```text
[controller]
[action]
[area]
```

Exemple :

```csharp
[Route("api/[controller]/[action]")]
```

Pour :

```csharp
HotelsController
```

et :

```csharp
GetAll()
```

on peut obtenir :

```text
/api/hotels/getall
```

Il faut toutefois éviter de rendre les routes inutilement dépendantes des noms de méthodes lorsque l'objectif est une API REST claire.

---

# 23. HTTP Method Constraint

Deux actions peuvent avoir le même chemin mais des méthodes HTTP différentes.

Exemple :

```csharp
[HttpGet]
public IActionResult Get()
{
    return Ok();
}
```

et :

```csharp
[HttpPost]
public IActionResult Create()
{
    return Ok();
}
```

Toutes deux peuvent utiliser :

```text
/api/hotels
```

mais :

```text
GET  /api/hotels -> Get()
POST /api/hotels -> Create()
```

Le routing utilise donc notamment la méthode HTTP pour sélectionner l'endpoint.

---

# 24. Ambiguïté de routes

Il faut éviter deux endpoints qui correspondent au même type de requête.

Exemple problématique :

```csharp
[HttpGet("{value}")]
public IActionResult GetByValue(string value)
{
    return Ok();
}

[HttpGet("{name}")]
public IActionResult GetByName(string name)
{
    return Ok();
}
```

Les deux templates sont identiques du point de vue du routing :

```text
/api/hotels/{value}
/api/hotels/{name}
```

Le nom du paramètre ne suffit pas à différencier les routes.

Il faut utiliser des chemins ou contraintes différents.

---

# 25. Routing et Model Binding

Ces deux concepts sont liés mais différents.

### Routing

Détermine :

```text
Quel endpoint ?
```

### Model Binding

Détermine :

```text
Comment remplir les paramètres de cet endpoint ?
```

Exemple :

```http
GET /api/hotels/5
```

Routing :

```text
GetHotel()
```

puis Model Binding :

```text
id = 5
```

Mentalement :

```text
Routing
   |
   v
Qui ?
   |
   v
Model Binding
   |
   v
Avec quelles données ?
```

---

# 26. Routing et validation

Même chose :

```text
Routing
    -> sélectionne l'endpoint

Binding
    -> construit les paramètres

Validation
    -> vérifie les données

Action
    -> exécute la logique
```

On peut visualiser :

```text
Request
   |
   v
Routing
   |
   v
Binding
   |
   v
Validation
   |
   v
Action
```

---

# 27. Contraintes de route vs validation

Une contrainte de route ne remplace pas toute la validation métier.

Exemple :

```csharp
[HttpGet("{id:int}")]
```

vérifie que :

```text
id est un entier
```

Mais cela ne signifie pas :

```text
l'hôtel existe
```

ou :

```text
l'utilisateur a le droit de consulter cet hôtel
```

Donc :

```text
Route constraint
    -> forme de la valeur

Validation métier
    -> règles de l'application
```

---

# 28. Slugs

Une API peut utiliser un slug :

```http
GET /api/hotels/hotel-central-mons
```

au lieu de :

```http
GET /api/hotels/5
```

Exemple :

```csharp
[HttpGet("{slug}")]
public IActionResult GetBySlug(string slug)
{
    return Ok();
}
```

Le routing transmet :

```text
hotel-central-mons
```

au paramètre :

```csharp
slug
```

---

# 29. Routing et versioning

Une API peut avoir plusieurs versions.

Exemple :

```text
/api/v1/hotels
/api/v2/hotels
```

Le routing peut participer à cette séparation.

Par exemple :

```csharp
[Route("api/v1/[controller]")]
```

et :

```csharp
[Route("api/v2/[controller]")]
```

La stratégie exacte de versioning dépend de l'architecture et des outils utilisés.

L'idée fondamentale est :

```text
Version 1 -> contrat V1
Version 2 -> contrat V2
```

---

# 30. Routes et REST

Une API REST claire utilise généralement des noms de ressources.

Préférer :

```text
GET    /api/hotels
GET    /api/hotels/5
POST   /api/hotels
PUT    /api/hotels/5
DELETE /api/hotels/5
```

plutôt que :

```text
GET /api/getHotels
POST /api/createHotel
POST /api/deleteHotel
```

Mentalement :

```text
URL = ressource
HTTP method = intention
```

---

# 31. Exemple complet

```csharp
[ApiController]
[Route("api/[controller]")]
public class HotelsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll()
    {
        return Ok();
    }

    [HttpGet("{id:int}")]
    public IActionResult GetById(int id)
    {
        return Ok();
    }

    [HttpGet("search")]
    public IActionResult Search(
        [FromQuery] string city)
    {
        return Ok();
    }

    [HttpPost]
    public IActionResult Create(
        CreateHotelDto dto)
    {
        return Created();
    }

    [HttpDelete("{id:int}")]
    public IActionResult Delete(int id)
    {
        return NoContent();
    }
}
```

Routes :

```text
GET    /api/hotels
GET    /api/hotels/5
GET    /api/hotels/search?city=Mons
POST   /api/hotels
DELETE /api/hotels/5
```

---

# 32. Erreurs fréquentes

## Erreur 1 : confondre route et query string

```text
/api/hotels/5
```

n'est pas la même chose que :

```text
/api/hotels?id=5
```

---

## Erreur 2 : oublier la contrainte

```csharp
[HttpGet("{id}")]
```

peut être trop permissif.

Lorsque le type fait partie du contrat de la route :

```csharp
[HttpGet("{id:int}")]
```

peut être plus précis.

---

## Erreur 3 : créer des routes ambiguës

```text
/{id}
/{name}
```

sont identiques pour le routing.

---

## Erreur 4 : utiliser des actions dans toutes les URLs

```text
/api/getHotels
/api/createHotel
```

n'est généralement pas nécessaire pour une API REST.

---

## Erreur 5 : utiliser des URLs trop profondes

```text
/api/countries/1/cities/2/hotels/3/rooms/4
```

peut devenir difficile à maintenir.

Une URL doit représenter une relation utile, pas reproduire toute la structure de la base de données.

---

## Erreur 6 : confondre routing et validation

```csharp
{id:int}
```

ne vérifie pas que l'entité existe.

---

# 33. Schéma global

```text
                   HTTP REQUEST
                        |
                        v
              +-------------------+
              |      Routing      |
              +-------------------+
                        |
                        v
                Endpoint trouvé
                        |
                        v
              +-------------------+
              |  Model Binding    |
              +-------------------+
                        |
                        v
              +-------------------+
              |    Validation     |
              +-------------------+
                        |
                        v
                  Controller
                        |
                        v
                    Service
                        |
                        v
                   Response
```

---

# 34. Règle mentale

Quand tu vois :

```csharp
[Route("api/[controller]")]
```

pense :

> « Je définis la base de l'URL du contrôleur. »

Quand tu vois :

```csharp
[HttpGet]
```

pense :

> « Cette action correspond à GET sur la route courante. »

Quand tu vois :

```csharp
[HttpGet("{id:int}")]
```

pense :

> « GET + paramètre de route qui doit être un entier. »

Quand tu vois :

```csharp
[FromRoute]
```

pense :

> « Cette valeur vient de l'URL. »

Quand tu vois :

```csharp
[FromQuery]
```

pense :

> « Cette valeur vient de la query string. »

Quand tu vois :

```csharp
app.MapControllers();
```

pense :

> « J'ajoute les endpoints des contrôleurs au pipeline de l'application. »

---

# 35. À retenir

1. Le routing détermine quel endpoint doit traiter une requête.
2. Une route est un template d'URL.
3. Un endpoint est la destination concrète sélectionnée.
4. `[Route]` définit une route.
5. `[HttpGet]`, `[HttpPost]`, `[HttpPut]`, `[HttpDelete]` associent une route à une méthode HTTP.
6. `{id}` représente un paramètre de route.
7. `{id:int}` ajoute une contrainte de type.
8. `[FromRoute]` indique une valeur provenant de la route.
9. `[FromQuery]` indique une valeur provenant de la query string.
10. Routing et Model Binding sont deux mécanismes différents.
11. Les contraintes de route ne remplacent pas la validation métier.
12. `MapControllers()` rend les endpoints des contrôleurs disponibles.
13. Les routes ambiguës doivent être évitées.
14. Pour une API REST, l'URL représente généralement une ressource et la méthode HTTP représente l'opération.
15. Le routing peut également servir à générer des URLs.

---

# Questions d'entretien

### 1. Qu'est-ce que le routing ?

C'est le mécanisme qui associe une requête HTTP à l'endpoint qui doit la traiter.

### 2. Quelle différence entre route et endpoint ?

La route est un template décrivant une URL ; l'endpoint est la destination concrète sélectionnée pour traiter la requête.

### 3. Quelle différence entre `/api/hotels/5` et `/api/hotels?id=5` ?

Dans le premier cas, `5` est un paramètre de route. Dans le second, `5` est une valeur de query string.

### 4. Pourquoi utiliser `{id:int}` ?

Pour ajouter une contrainte indiquant que le segment `id` doit être un entier.

### 5. Quelle différence entre routing et model binding ?

Le routing sélectionne l'endpoint ; le model binding construit les paramètres de cet endpoint à partir des données HTTP.

### 6. Que fait `MapControllers()` ?

Il ajoute les endpoints définis par les contrôleurs au système de routing de l'application.

### 7. Pourquoi deux routes `/{id}` et `/{name}` sont-elles ambiguës ?

Parce que le nom du paramètre n'est pas pris en compte pour distinguer la structure des routes.

### 8. Une contrainte `{id:int}` vérifie-t-elle que l'entité existe ?

Non. Elle vérifie seulement que la valeur correspond à la contrainte de route. L'existence de l'entité relève de la logique applicative.

### 9. Pourquoi utiliser la méthode HTTP pour exprimer l'opération ?

Parce qu'une API REST représente généralement la ressource dans l'URL et utilise la méthode HTTP pour exprimer l'intention.

---

# Phrase à retenir

> **Le routing répond à « quel endpoint doit traiter cette requête ? » : il compare l'URL et la méthode HTTP aux templates de routes, sélectionne l'endpoint, puis le Model Binding fournit les paramètres nécessaires à son exécution.**
