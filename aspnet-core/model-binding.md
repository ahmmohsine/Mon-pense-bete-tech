# Model Binding en ASP.NET Core

## 1. Définition

Le **Model Binding** est le mécanisme d'ASP.NET Core qui transforme les données provenant d'une requête HTTP en paramètres et objets C# utilisables par une action.

Exemple :

```http
GET /api/hotels/5
```

et :

```csharp
[HttpGet("{id}")]
public IActionResult GetHotel(int id)
{
    return Ok();
}
```

ASP.NET Core doit transformer :

```text
"5"
```

en :

```csharp
int id = 5;
```

C'est le rôle du Model Binding.

Mentalement :

```text
HTTP Request
     |
     v
Model Binding
     |
     v
Paramètres C#
```

---

# 2. Pourquoi le Model Binding existe ?

Sans Model Binding, le développeur devrait lire manuellement :

```text
Route
Query String
Headers
Body
Form
```

puis convertir les valeurs :

```csharp
int.Parse(...)
Guid.Parse(...)
DateTime.Parse(...)
```

ASP.NET Core automatise une grande partie de ce travail.

Au lieu de :

```csharp
var idText = Request.RouteValues["id"];
var id = int.Parse(idText!.ToString()!);
```

on écrit simplement :

```csharp
public IActionResult GetHotel(int id)
```

Le framework fait le travail de liaison.

---

# 3. Les principales sources de données

Le Model Binding peut récupérer des données depuis différentes parties de la requête :

```text
Route
Query String
Form
Headers
Body
```

Exemple :

```http
GET /api/hotels/5?city=Mons
```

On peut avoir :

```text
id   -> route
city -> query string
```

---

# 4. Paramètre provenant de la route

Exemple :

```csharp
[HttpGet("{id}")]
public IActionResult GetHotel(int id)
{
    return Ok(id);
}
```

Requête :

```http
GET /api/hotels/5
```

Le Model Binding construit :

```csharp
id = 5
```

On peut rendre la source explicite :

```csharp
public IActionResult GetHotel(
    [FromRoute] int id)
{
    return Ok(id);
}
```

---

# 5. Paramètre provenant de la Query String

Requête :

```http
GET /api/hotels?city=Mons
```

Code :

```csharp
[HttpGet]
public IActionResult Search(string city)
{
    return Ok(city);
}
```

ASP.NET Core lie :

```text
city=Mons
```

à :

```csharp
string city
```

On peut être explicite :

```csharp
public IActionResult Search(
    [FromQuery] string city)
{
    return Ok(city);
}
```

---

# 6. Plusieurs paramètres Query String

Requête :

```http
GET /api/hotels?city=Mons&page=2&pageSize=20
```

Code :

```csharp
public IActionResult Search(
    [FromQuery] string city,
    [FromQuery] int page,
    [FromQuery] int pageSize)
{
    return Ok();
}
```

Le binding donne conceptuellement :

```csharp
city = "Mons";
page = 2;
pageSize = 20;
```

---

# 7. Binding vers un objet

Le Model Binding peut également construire un objet.

Exemple :

```csharp
public class HotelFilter
{
    public string? City { get; set; }

    public int Page { get; set; }

    public int PageSize { get; set; }
}
```

Requête :

```http
GET /api/hotels?city=Mons&page=2&pageSize=20
```

Action :

```csharp
public IActionResult Search(
    [FromQuery] HotelFilter filter)
{
    return Ok(filter);
}
```

ASP.NET Core remplit :

```text
HotelFilter
    |
    +-- City = Mons
    +-- Page = 2
    +-- PageSize = 20
```

---

# 8. `[FromQuery]`

`[FromQuery]` indique explicitement que la valeur doit venir de la query string.

```csharp
public IActionResult Search(
    [FromQuery] HotelFilter filter)
{
    return Ok();
}
```

Exemple :

```http
GET /api/hotels?city=Mons
```

---

# 9. `[FromRoute]`

`[FromRoute]` indique que la valeur vient de la route.

```csharp
[HttpGet("{id}")]
public IActionResult GetHotel(
    [FromRoute] int id)
{
    return Ok();
}
```

Requête :

```http
GET /api/hotels/5
```

Résultat :

```csharp
id = 5
```

---

# 10. `[FromHeader]`

On peut demander une valeur provenant des headers.

```csharp
public IActionResult Get(
    [FromHeader(Name = "X-Correlation-Id")]
    string correlationId)
{
    return Ok();
}
```

Avec :

```http
X-Correlation-Id: abc-123
```

le paramètre reçoit :

```text
abc-123
```

---

# 11. `[FromBody]`

Pour une API, les données JSON sont généralement envoyées dans le body.

Exemple :

```http
POST /api/hotels
Content-Type: application/json
```

Body :

```json
{
  "name": "Hotel Central",
  "city": "Mons"
}
```

DTO :

```csharp
public class CreateHotelDto
{
    public string Name { get; set; } = string.Empty;

    public string City { get; set; } = string.Empty;
}
```

Action :

```csharp
[HttpPost]
public IActionResult Create(
    [FromBody] CreateHotelDto dto)
{
    return Ok(dto);
}
```

ASP.NET Core utilise le body pour construire :

```csharp
CreateHotelDto
```

---

# 12. JSON et désérialisation

Quand le body contient :

```json
{
  "name": "Hotel Central",
  "city": "Mons"
}
```

ASP.NET Core doit transformer le JSON en objet C#.

Conceptuellement :

```text
JSON
 |
 v
JSON deserializer
 |
 v
CreateHotelDto
```

Le Model Binding et la désérialisation travaillent donc ensemble pour fournir les paramètres de l'action.

---

# 13. Model Binding vs JSON deserialization

Ces notions sont proches mais ne sont pas exactement identiques.

### Model Binding

Détermine comment les données HTTP alimentent les paramètres de l'action.

### JSON deserialization

Transforme le JSON du body en objet C#.

Mentalement :

```text
HTTP Request
     |
     v
Model Binding
     |
     +--> Route
     +--> Query
     +--> Header
     +--> Body
               |
               v
          JSON deserializer
               |
               v
           C# object
```

---

# 14. `[ApiController]` et Body

Avec :

```csharp
[ApiController]
```

ASP.NET Core peut déduire la source de certains paramètres complexes.

Exemple :

```csharp
[HttpPost]
public IActionResult Create(CreateHotelDto dto)
{
    return Ok(dto);
}
```

Le framework peut comprendre que :

```text
CreateHotelDto
    -> Body
```

Pour rendre l'intention explicite, on peut écrire :

```csharp
public IActionResult Create(
    [FromBody] CreateHotelDto dto)
{
    return Ok(dto);
}
```

---

# 15. Conversion des types

Supposons :

```http
GET /api/hotels/5
```

avec :

```csharp
int id
```

Le HTTP transporte essentiellement du texte.

ASP.NET Core doit donc effectuer une conversion conceptuelle :

```text
"5"
  |
  v
int.Parse / conversion adaptée
  |
  v
5
```

Pour un type invalide :

```http
GET /api/hotels/abc
```

avec :

```csharp
int id
```

le binding peut produire une erreur de binding.

Avec `[ApiController]`, cette situation peut conduire automatiquement à une réponse :

```http
400 Bad Request
```

---

# 16. `ModelState`

Les erreurs de binding et de validation peuvent être représentées dans :

```csharp
ModelState
```

Exemple :

```csharp
if (!ModelState.IsValid)
{
    return BadRequest(ModelState);
}
```

Cependant, avec :

```csharp
[ApiController]
```

ASP.NET Core gère automatiquement certaines erreurs de validation et de binding avant l'exécution de l'action.

---

# 17. Binding et validation : deux étapes différentes

Il faut absolument distinguer :

```text
Binding
```

et :

```text
Validation
```

### Binding

Répond à :

> « Est-ce que je peux construire le paramètre C# à partir des données HTTP ? »

### Validation

Répond à :

> « Les données obtenues respectent-elles les règles demandées ? »

Exemple :

```json
{
  "name": "",
  "age": -5
}
```

Le binding peut réussir :

```text
Name = ""
Age = -5
```

Mais la validation peut ensuite dire :

```text
Name obligatoire
Age doit être positif
```

Mentalement :

```text
HTTP
 |
 v
Binding
 |
 v
Objet C#
 |
 v
Validation
 |
 v
Action
```

---

# 18. Valeur absente vs valeur invalide

Ces situations sont différentes.

### Valeur absente

```http
GET /api/hotels
```

mais l'action demande :

```csharp
public IActionResult Get(int id)
```

Il manque la valeur.

### Valeur invalide

```http
GET /api/hotels/abc
```

alors que :

```csharp
int id
```

est attendu.

Ici la valeur existe, mais elle ne peut pas être convertie en `int`.

Cette distinction est importante pour comprendre les erreurs de binding.

---

# 19. Types nullable

Avec :

```csharp
public IActionResult Search(
    string? city)
```

`city` peut être absent.

Avec :

```csharp
public IActionResult Search(
    string city)
```

le code indique que `city` est censé ne pas être `null`.

Attention :

> Nullable Reference Types est principalement une fonctionnalité d'analyse du compilateur ; cela ne remplace pas la validation HTTP.

---

# 20. Paramètres complexes

Un DTO peut contenir plusieurs propriétés :

```csharp
public class SearchHotelDto
{
    public string? City { get; set; }

    public int Page { get; set; }

    public int PageSize { get; set; }
}
```

Pour une query string :

```http
GET /api/hotels?city=Mons&page=1&pageSize=20
```

ASP.NET Core peut construire :

```text
SearchHotelDto
    |
    +-- City = Mons
    +-- Page = 1
    +-- PageSize = 20
```

---

# 21. Collections

Le Model Binding peut également construire des collections.

Exemple :

```http
GET /api/hotels?ids=1&ids=5&ids=8
```

Action :

```csharp
public IActionResult Get(
    [FromQuery] int[] ids)
{
    return Ok(ids);
}
```

On obtient :

```text
ids = [1, 5, 8]
```

---

# 22. Boolean

Exemple :

```http
GET /api/hotels?active=true
```

Action :

```csharp
public IActionResult Get(
    [FromQuery] bool active)
{
    return Ok(active);
}
```

Le binding convertit la valeur vers :

```csharp
bool
```

---

# 23. DateTime

Exemple :

```http
GET /api/hotels?date=2026-10-04
```

Code :

```csharp
public IActionResult Get(
    [FromQuery] DateTime date)
{
    return Ok(date);
}
```

Le framework tente de convertir la valeur reçue vers le type demandé.

Pour les APIs, il est important de définir des formats cohérents et non ambigus lorsque les dates ont une importance fonctionnelle.

---

# 24. Guid

Exemple :

```csharp
[HttpGet("{id:guid}")]
public IActionResult Get(Guid id)
{
    return Ok(id);
}
```

Requête :

```http
GET /api/hotels/550e8400-e29b-41d4-a716-446655440000
```

Le binding convertit la représentation texte en :

```csharp
Guid
```

---

# 25. `FromForm`

Pour des données de formulaire :

```csharp
public IActionResult Upload(
    [FromForm] string description)
{
    return Ok();
}
```

Cela peut être utilisé notamment avec des formulaires multipart.

Pour les fichiers :

```csharp
public IActionResult Upload(
    IFormFile file)
{
    return Ok();
}
```

Le cas des fichiers nécessite une attention particulière au `Content-Type` et à la taille des uploads.

---

# 26. Plusieurs sources dans une action

Une action peut recevoir des données de différentes sources.

Exemple :

```csharp
[HttpGet("{id}")]
public IActionResult GetHotel(
    [FromRoute] int id,
    [FromQuery] bool includeRooms)
{
    return Ok();
}
```

Requête :

```http
GET /api/hotels/5?includeRooms=true
```

On obtient :

```text
id            -> route
includeRooms  -> query
```

---

# 27. Ne pas mélanger inutilement les sources

Il faut que le contrat HTTP reste compréhensible.

Exemple :

```text
Route
    -> identité de la ressource

Query
    -> filtres/options

Body
    -> données complexes envoyées au serveur
```

Ce n'est pas une règle absolue, mais c'est une bonne convention pour concevoir des APIs lisibles.

---

# 28. Model Binding et DTOs

Un DTO est souvent le résultat du binding du body.

Exemple :

```http
POST /api/hotels
```

Body :

```json
{
  "name": "Hotel Central",
  "city": "Mons"
}
```

Puis :

```csharp
public IActionResult Create(
    CreateHotelDto dto)
```

Le flux est :

```text
JSON
 |
 v
Deserialization
 |
 v
CreateHotelDto
 |
 v
Validation
 |
 v
Controller
 |
 v
Service
```

---

# 29. Binding et architecture

Le Model Binding appartient principalement à la couche HTTP.

Une bonne séparation ressemble à :

```text
HTTP
 |
 +-- Model Binding
 +-- Validation
 |
 v
Controller
 |
 v
Application Service
 |
 v
Domain
```

Le service métier ne devrait généralement pas savoir si une valeur provenait :

```text
Route
Query String
JSON
Header
```

Il reçoit déjà des données sous une forme adaptée à son rôle.

---

# 30. Custom Model Binding

ASP.NET Core permet également de créer des mécanismes de binding personnalisés.

Cela peut être utile lorsqu'un type nécessite une logique particulière de conversion.

Exemple conceptuel :

```text
"hotel-5"
      |
      v
Custom Binder
      |
      v
HotelIdentifier
```

Ce mécanisme est puissant mais doit être utilisé seulement lorsque le binding standard ne suffit pas.

Pour la majorité des APIs, le binding standard + DTOs est préférable.

---

# 31. Erreurs fréquentes

## Erreur 1 : confondre Binding et Validation

```text
Binding -> construire l'objet
Validation -> vérifier les règles
```

Ce sont deux étapes différentes.

---

## Erreur 2 : lire manuellement `Request.Query`

Mauvais réflexe :

```csharp
var city = Request.Query["city"];
```

lorsqu'un simple paramètre suffit :

```csharp
public IActionResult Search(string city)
```

Le Model Binding existe justement pour éviter ce code répétitif.

---

## Erreur 3 : recevoir directement l'entité EF Core

Préférer :

```csharp
CreateHotelDto
```

plutôt que :

```csharp
Hotel
```

pour le contrat HTTP lorsque les deux modèles ont des responsabilités différentes.

---

## Erreur 4 : ne pas comprendre d'où vient une valeur

Lorsqu'un paramètre ne reçoit pas la valeur attendue, vérifier :

```text
Route ?
Query ?
Body ?
Header ?
Form ?
```

Puis utiliser explicitement :

```csharp
[FromRoute]
[FromQuery]
[FromBody]
[FromHeader]
[FromForm]
```

si nécessaire.

---

## Erreur 5 : oublier les erreurs de conversion

Une valeur comme :

```text
abc
```

ne peut pas être automatiquement convertie en :

```csharp
int
```

Le binding peut donc échouer avant l'exécution normale de l'action.

---

# 32. Schéma global

```text
                 HTTP REQUEST
                      |
        +-------------+-------------+
        |             |             |
        v             v             v
      Route         Query          Body
        |             |             |
        +-------------+-------------+
                      |
                      v
               Model Binding
                      |
                      v
                C# Parameters
                      |
                      v
                 Validation
                      |
                      v
                 Controller
                      |
                      v
                  Service
```

---

# 33. Règle mentale

Quand tu vois :

```csharp
int id
```

dans une action, pense :

> « ASP.NET Core doit trouver une valeur HTTP et la convertir en `int`. »

Quand tu vois :

```csharp
[FromRoute]
```

pense :

> « La valeur vient du chemin de l'URL. »

Quand tu vois :

```csharp
[FromQuery]
```

pense :

> « La valeur vient après `?` dans l'URL. »

Quand tu vois :

```csharp
[FromBody]
```

pense :

> « La valeur vient du contenu HTTP, généralement du JSON pour une API. »

Quand tu vois :

```csharp
[ApiController]
```

pense :

> « ASP.NET Core peut automatiser une partie du binding et de la gestion des erreurs de validation. »

---

# 34. À retenir

1. Le Model Binding transforme les données HTTP en paramètres C#.
2. Les principales sources sont Route, Query, Body, Header et Form.
3. `[FromRoute]` indique une valeur provenant de la route.
4. `[FromQuery]` indique une valeur provenant de la query string.
5. `[FromBody]` indique une valeur provenant du body.
6. `[FromHeader]` indique une valeur provenant d'un header.
7. `[FromForm]` indique une valeur provenant de données de formulaire.
8. Le body JSON est désérialisé vers un objet C#.
9. Binding et validation sont deux mécanismes différents.
10. `[ApiController]` automatise une partie de la gestion des erreurs de binding/validation.
11. Le Model Binding réalise également les conversions de types.
12. Les DTOs sont particulièrement adaptés aux contrats HTTP.
13. Il faut savoir distinguer valeur absente et valeur invalide.
14. Le Model Binding appartient principalement à la couche HTTP.
15. Le code métier devrait recevoir des données déjà adaptées à son rôle.

---

# Questions d'entretien

### 1. Qu'est-ce que le Model Binding ?

C'est le mécanisme ASP.NET Core qui récupère les données d'une requête HTTP et les transforme en paramètres ou objets C#.

### 2. Quelles sont les principales sources du Model Binding ?

Route, Query String, Body, Headers et Form.

### 3. Quelle différence entre `[FromRoute]` et `[FromQuery]` ?

`[FromRoute]` récupère une valeur depuis le chemin de l'URL ; `[FromQuery]` la récupère depuis la query string.

### 4. Que se passe-t-il avec `GET /api/hotels/abc` si l'action demande `int id` ?

Le binding ne peut pas convertir `abc` en `int`, ce qui produit une erreur de binding. Avec `[ApiController]`, cela peut conduire automatiquement à une réponse 400.

### 5. Quelle différence entre Model Binding et désérialisation JSON ?

Le Model Binding est le mécanisme global qui fournit les paramètres depuis les différentes sources HTTP. La désérialisation JSON transforme spécifiquement le contenu JSON du body en objet C#.

### 6. Quelle différence entre Binding et Validation ?

Le binding construit les paramètres ; la validation vérifie ensuite si les valeurs obtenues respectent les règles attendues.

### 7. Pourquoi utiliser des DTOs ?

Pour définir un contrat HTTP indépendant des entités internes et contrôler précisément les données reçues ou renvoyées.

### 8. Pourquoi utiliser `[FromBody]` ?

Pour indiquer explicitement que le paramètre doit être construit à partir du contenu du body HTTP.

### 9. Pourquoi éviter de lire systématiquement `Request.Query` ?

Parce que le Model Binding fournit déjà un mécanisme typé et déclaratif pour récupérer ces valeurs.

---

# Phrase à retenir

> **Le Model Binding transforme les données de la requête HTTP en paramètres C# : le routing choisit l'endpoint, le binding construit ses paramètres, puis la validation vérifie que les données sont acceptables.**
