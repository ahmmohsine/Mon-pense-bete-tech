# Validation en ASP.NET Core

## 1. Définition

La **validation** vérifie que les données reçues par une API respectent les règles attendues par l'application.

Exemple :

```json
{
  "name": "",
  "email": "abc"
}
```

Le Model Binding peut réussir à construire l'objet :

```csharp
CreateUserDto
```

mais la validation peut ensuite détecter :

```text
Name obligatoire
Email invalide
```

Mentalement :

```text
HTTP Request
      |
      v
Model Binding
      |
      v
Objet C#
      |
      v
Validation
      |
      v
Controller / Endpoint
```

---

# 2. Binding ≠ Validation

C'est l'une des distinctions les plus importantes à retenir.

### Model Binding

Répond à :

> « Comment transformer les données HTTP en paramètres C# ? »

### Validation

Répond à :

> « Les données obtenues sont-elles acceptables ? »

Exemple :

```http
GET /api/hotels/abc
```

avec :

```csharp
int id
```

Le problème est d'abord un problème de **binding** :

```text
"abc" -> int
```

Impossible.

Autre exemple :

```json
{
  "name": "",
  "price": -20
}
```

Le JSON peut parfaitement être transformé en objet C# :

```csharp
Name = ""
Price = -20
```

Mais les règles métier ou de validation peuvent refuser ces valeurs.

---

# 3. Data Annotations

ASP.NET Core permet d'utiliser des attributs de validation.

Exemple :

```csharp
using System.ComponentModel.DataAnnotations;

public class CreateHotelDto
{
    [Required]
    public string Name { get; set; } = string.Empty;

    [Range(1, 5)]
    public int Stars { get; set; }

    [StringLength(100)]
    public string? City { get; set; }
}
```

Les attributs indiquent les contraintes attendues.

Quelques attributs courants :

```text
[Required]
[StringLength]
[MinLength]
[MaxLength]
[Range]
[EmailAddress]
[Phone]
[Url]
[RegularExpression]
```

---

# 4. `[Required]`

`[Required]` indique qu'une valeur est obligatoire.

Exemple :

```csharp
public class CreateHotelDto
{
    [Required]
    public string Name { get; set; } = string.Empty;
}
```

Si le client envoie une valeur absente ou non acceptable selon les règles du validator, une erreur de validation peut être produite.

Attention :

```csharp
[Required]
public string Name { get; set; } = string.Empty;
```

L'initialisation à `string.Empty` ne signifie pas que la chaîne est valide.

La validation doit toujours être comprise indépendamment de la valeur par défaut utilisée par le code.

---

# 5. `[StringLength]`

Permet de limiter la longueur d'une chaîne.

```csharp
[StringLength(100)]
public string? Name { get; set; }
```

On peut également définir une longueur minimale :

```csharp
[StringLength(100, MinimumLength = 3)]
public string? Name { get; set; }
```

Cela signifie :

```text
Minimum = 3
Maximum = 100
```

---

# 6. `[MinLength]` et `[MaxLength]`

On peut les utiliser séparément :

```csharp
[MinLength(3)]
[MaxLength(100)]
public string? Name { get; set; }
```

Ils sont particulièrement utiles lorsque l'on veut exprimer clairement les deux limites.

---

# 7. `[Range]`

Pour limiter une valeur numérique :

```csharp
[Range(1, 5)]
public int Stars { get; set; }
```

Exemples :

```text
1 -> valide
3 -> valide
5 -> valide
0 -> invalide
6 -> invalide
```

---

# 8. `[EmailAddress]`

Pour vérifier le format général d'une adresse e-mail :

```csharp
[EmailAddress]
public string? Email { get; set; }
```

Attention :

> Une validation de format ne signifie pas que l'adresse existe réellement.

Elle vérifie principalement que la valeur respecte les règles de format attendues.

---

# 9. Validation et `[ApiController]`

Avec :

```csharp
[ApiController]
[Route("api/[controller]")]
public class HotelsController : ControllerBase
{
}
```

ASP.NET Core peut effectuer automatiquement certaines vérifications de validation avant d'exécuter l'action.

Exemple :

```csharp
[HttpPost]
public IActionResult Create(CreateHotelDto dto)
{
    return Ok(dto);
}
```

Si le DTO est invalide, l'action peut ne pas être exécutée et ASP.NET Core retourner automatiquement une réponse :

```http
400 Bad Request
```

---

# 10. Pourquoi `[ApiController]` est important ?

Sans automatisation, on pourrait écrire :

```csharp
if (!ModelState.IsValid)
{
    return BadRequest(ModelState);
}
```

Avec `[ApiController]`, ASP.NET Core peut prendre en charge cette logique automatiquement pour les contrôleurs API.

Cela permet d'avoir des contrôleurs plus propres :

```csharp
[HttpPost]
public IActionResult Create(CreateHotelDto dto)
{
    // Si l'action est exécutée,
    // les validations automatiques ont déjà été prises en compte.

    return Ok();
}
```

---

# 11. `ModelState`

`ModelState` contient notamment les informations relatives au binding et à la validation des données reçues.

On peut vérifier :

```csharp
if (!ModelState.IsValid)
{
    return BadRequest(ModelState);
}
```

Pour diagnostiquer une erreur :

```csharp
foreach (var error in ModelState)
{
    Console.WriteLine(error.Key);

    foreach (var message in error.Value!.Errors)
    {
        Console.WriteLine(message.ErrorMessage);
    }
}
```

Dans une API moderne utilisant `[ApiController]`, cette vérification manuelle n'est généralement pas nécessaire pour les cas standards.

---

# 12. Exemple complet avec DTO

```csharp
public class CreateHotelDto
{
    [Required]
    [StringLength(100, MinimumLength = 2)]
    public string Name { get; set; } = string.Empty;

    [Required]
    [StringLength(50)]
    public string City { get; set; } = string.Empty;

    [Range(1, 5)]
    public int Stars { get; set; }
}
```

Controller :

```csharp
[ApiController]
[Route("api/[controller]")]
public class HotelsController : ControllerBase
{
    [HttpPost]
    public IActionResult Create(CreateHotelDto dto)
    {
        return Ok(dto);
    }
}
```

Requête valide :

```json
{
  "name": "Hotel Central",
  "city": "Mons",
  "stars": 4
}
```

Requête potentiellement invalide :

```json
{
  "name": "",
  "city": "",
  "stars": 8
}
```

Les règles déclarées dans le DTO permettent de détecter ces problèmes.

---

# 13. Validation automatique et réponse 400

Avec `[ApiController]`, une requête invalide peut produire automatiquement une réponse `400 Bad Request`.

Le client peut recevoir une réponse structurée indiquant les erreurs.

Conceptuellement :

```json
{
  "errors": {
    "Name": [
      "The Name field is required."
    ],
    "Stars": [
      "The field Stars must be between 1 and 5."
    ]
  }
}
```

Le format exact dépend notamment de la configuration de l'API et de la version/framework utilisés.

---

# 14. ProblemDetails

ASP.NET Core peut utiliser le format **ProblemDetails** pour représenter les erreurs HTTP de manière standardisée.

L'idée est d'avoir une réponse structurée contenant par exemple :

```text
status
title
detail
instance
errors
```

Exemple conceptuel :

```json
{
  "type": "...",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "Name": [
      "The Name field is required."
    ]
  }
}
```

L'intérêt est de fournir au client une structure prévisible.

---

# 15. Validation côté client ≠ validation côté serveur

Une interface Angular peut vérifier :

```text
Email obligatoire
Mot de passe trop court
```

Mais cela ne suffit jamais.

Le serveur doit également valider.

Pourquoi ?

Parce qu'un client HTTP peut être :

```text
Angular
Postman
curl
une application mobile
un script
un attaquant
```

Le serveur ne doit jamais faire confiance aux validations exécutées uniquement dans le navigateur.

Règle :

> La validation côté client améliore l'expérience utilisateur ; la validation côté serveur protège l'application.

---

# 16. Validation syntaxique vs validation métier

Il faut également distinguer plusieurs niveaux.

### Validation de format

Exemple :

```text
Email doit avoir un format valide
```

### Validation de contraintes

Exemple :

```text
Age entre 18 et 120
```

### Validation métier

Exemple :

```text
Un hôtel ne peut pas être réservé si toutes ses chambres sont déjà réservées.
```

Cette dernière règle ne devrait pas forcément être placée dans un simple attribut :

```csharp
[Range(...)]
```

Elle appartient souvent à la couche métier/application.

---

# 17. Validation métier dans le service

Exemple :

```csharp
public async Task<Result> CreateBookingAsync(CreateBookingDto dto)
{
    var roomAvailable = await _repository.IsRoomAvailableAsync(dto.RoomId);

    if (!roomAvailable)
    {
        return Result.Failure("Room is not available.");
    }

    // Création...
}
```

Le DTO peut vérifier :

```text
Date valide
RoomId fourni
```

Mais le service vérifie :

```text
La chambre est-elle réellement disponible ?
```

La différence est importante.

---

# 18. Pourquoi ne pas tout mettre dans le DTO ?

Un DTO doit principalement représenter le contrat de données.

Exemple :

```csharp
public class CreateBookingDto
{
    [Required]
    public int RoomId { get; set; }

    [Required]
    public DateTime StartDate { get; set; }

    [Required]
    public DateTime EndDate { get; set; }
}
```

Il peut exprimer des contraintes simples.

Mais une règle comme :

```text
La chambre ne peut pas être réservée si elle est déjà occupée.
```

nécessite généralement l'accès aux données et au contexte métier.

Elle appartient donc plutôt à la logique applicative/métier.

---

# 19. Validation personnalisée

Lorsque les attributs standards ne suffisent pas, on peut créer une validation personnalisée.

Exemple :

```csharp
public class DateRangeAttribute : ValidationAttribute
{
    protected override ValidationResult? IsValid(
        object? value,
        ValidationContext validationContext)
    {
        // Logique de validation

        return ValidationResult.Success;
    }
}
```

Puis :

```csharp
[DateRange]
public DateTime StartDate { get; set; }
```

Cependant, il ne faut pas créer des attributs personnalisés pour toutes les règles métier.

---

# 20. Validation de plusieurs propriétés

Certaines règles dépendent de plusieurs propriétés.

Exemple :

```text
StartDate < EndDate
```

Ce n'est pas simplement une propriété isolée.

Il faut comparer :

```csharp
StartDate
EndDate
```

Ce type de règle peut être géré avec une validation au niveau de l'objet ou avec une solution de validation dédiée.

Mentalement :

```text
Validation simple
    -> une propriété

Validation complexe
    -> plusieurs propriétés

Validation métier
    -> contexte de l'application
```

---

# 21. FluentValidation

Une autre approche courante dans l'écosystème .NET est **FluentValidation**.

Au lieu d'écrire :

```csharp
[Required]
[StringLength(100)]
public string Name { get; set; }
```

on peut définir les règles dans un validator :

```csharp
public class CreateHotelValidator
    : AbstractValidator<CreateHotelDto>
{
    public CreateHotelValidator()
    {
        RuleFor(x => x.Name)
            .NotEmpty()
            .MaximumLength(100);

        RuleFor(x => x.Stars)
            .InclusiveBetween(1, 5);
    }
}
```

L'avantage est de centraliser des règles de validation complexes dans une classe dédiée.

---

# 22. Data Annotations vs FluentValidation

### Data Annotations

```csharp
[Required]
[StringLength(100)]
public string Name { get; set; }
```

Avantages :

- simple ;
- intégré à .NET ;
- très lisible pour les règles simples ;
- peu de configuration.

Inconvénients :

- moins flexible pour les règles complexes ;
- les règles sont directement placées sur le modèle.

### FluentValidation

```csharp
RuleFor(x => x.Name)
    .NotEmpty()
    .MaximumLength(100);
```

Avantages :

- très expressif ;
- règles complexes plus faciles à organiser ;
- validation conditionnelle plus naturelle ;
- logique de validation séparée du DTO.

Inconvénients :

- dépendance supplémentaire ;
- nécessite une configuration/intégration adaptée au projet.

---

# 23. Validation conditionnelle

Certaines règles dépendent d'une autre propriété.

Exemple conceptuel :

```text
IsCompany = true
    -> CompanyName obligatoire
```

Avec une solution de validation adaptée, on peut exprimer :

```text
Si IsCompany == true
alors CompanyName doit être renseigné
```

Ce type de règle est difficile à exprimer proprement avec de simples `[Required]`.

---

# 24. Validation et sécurité

La validation participe également à la sécurité.

Elle peut empêcher certaines données manifestement incorrectes ou inattendues.

Mais attention :

> La validation n'est pas une solution de sécurité complète.

Elle ne remplace pas :

```text
Authentication
Authorization
DTOs
Rate Limiting
Input sanitization selon le contexte
Contrôles métier
Protection contre les attaques
```

---

# 25. Validation et Entity Framework Core

Il ne faut pas supposer que les validations du DTO remplacent les contraintes de la base de données.

Exemple :

```text
DTO validation
    |
    v
Application
    |
    v
EF Core
    |
    v
Database constraints
```

La base peut encore refuser une opération à cause de :

```text
UNIQUE
FOREIGN KEY
NOT NULL
CHECK
```

La validation applicative et les contraintes de base ont des rôles différents.

---

# 26. Validation et concurrence

Certaines règles peuvent être vraies au moment de la validation mais devenir fausses juste après.

Exemple :

```text
1. Vérifier que le produit est disponible
2. Un autre utilisateur achète le produit
3. Notre application tente l'achat
```

La validation initiale ne garantit donc pas à elle seule l'intégrité de l'opération.

Il faut parfois utiliser :

```text
transactions
contraintes DB
concurrency control
```

selon le problème.

---

# 27. Où placer les validations ?

Une architecture typique peut ressembler à :

```text
HTTP
 |
 v
DTO
 |
 v
Validation technique
 |
 v
Application Service
 |
 v
Validation métier
 |
 v
Domain
 |
 v
Database
```

Exemple :

```text
"Name obligatoire"
        -> DTO / validation

"Stars entre 1 et 5"
        -> DTO / validation

"Impossible de réserver une chambre occupée"
        -> logique métier

"Deux réservations ne peuvent pas avoir le même identifiant"
        -> base de données / contrainte
```

---

# 28. Validation et Controller

Un contrôleur doit rester relativement mince.

Éviter :

```csharp
[HttpPost]
public IActionResult Create(CreateBookingDto dto)
{
    // 100 lignes de règles métier
}
```

Préférer :

```csharp
[HttpPost]
public async Task<IActionResult> Create(CreateBookingDto dto)
{
    var result = await _bookingService.CreateAsync(dto);

    return Ok(result);
}
```

Le contrôleur orchestre la requête HTTP.

Le service porte la logique applicative.

---

# 29. Erreurs fréquentes

## Erreur 1 : croire que la validation client suffit

```text
Angular valide
    -> donc c'est sécurisé
```

Faux.

Le serveur doit toujours valider.

---

## Erreur 2 : confondre validation et autorisation

Validation :

```text
La donnée est-elle correcte ?
```

Authorization :

```text
Cet utilisateur a-t-il le droit de faire cette action ?
```

Ce sont deux problèmes différents.

---

## Erreur 3 : mettre toute la logique métier dans les Data Annotations

Les attributs sont adaptés aux règles simples.

Ils ne doivent pas devenir un conteneur de toute la logique métier.

---

## Erreur 4 : valider uniquement dans le Controller

Si une règle métier est importante, elle ne devrait pas dépendre uniquement d'un seul endpoint.

Elle doit être protégée au niveau approprié de l'application.

---

## Erreur 5 : retourner des informations sensibles dans les erreurs

Éviter de retourner :

```text
stack trace
SQL interne
nom de table
connection string
informations d'infrastructure
```

au client en production.

---

# 30. Validation et HTTP status codes

Pour une requête dont les données ne respectent pas le contrat attendu :

```http
400 Bad Request
```

est fréquemment utilisé.

Mais il faut distinguer :

```text
400 -> requête invalide
401 -> authentification nécessaire/invalide
403 -> accès refusé
404 -> ressource introuvable
409 -> conflit
```

Le choix du code dépend du contexte exact de l'erreur.

---

# 31. Flux complet d'une requête

Voici le modèle mental à retenir :

```text
Client
  |
  | HTTP Request
  v
Middleware
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
Model Binding
  |
  v
Validation
  |
  +---- invalide ----> 400
  |
  v
Controller / Endpoint
  |
  v
Application Service
  |
  v
Domain
  |
  v
EF Core / Database
  |
  v
Response
```

Ce schéma permet de situer chaque mécanisme ASP.NET Core.

---

# 32. Règle mentale

Quand tu vois :

```csharp
[Required]
```

pense :

> « Cette donnée doit respecter une contrainte simple. »

Quand tu vois :

```csharp
[ApiController]
```

pense :

> « ASP.NET Core peut déclencher automatiquement la réponse d'erreur lorsque le modèle est invalide. »

Quand tu vois une règle comme :

```text
StartDate < EndDate
```

pense :

> « Cette règle concerne plusieurs valeurs. »

Quand tu vois :

```text
La chambre doit être disponible
```

pense :

> « C'est une règle métier, pas simplement une validation de format. »

---

# 33. À retenir

1. La validation vérifie que les données reçues respectent les contraintes attendues.
2. Le Model Binding et la validation sont deux étapes différentes.
3. Les Data Annotations sont adaptées aux validations simples.
4. `[Required]` indique qu'une valeur est obligatoire.
5. `[StringLength]` contrôle la longueur d'une chaîne.
6. `[Range]` contrôle une plage de valeurs.
7. `[EmailAddress]` vérifie principalement le format d'un e-mail.
8. `[ApiController]` automatise une partie de la gestion des erreurs de validation.
9. `ModelState` contient l'état du binding et de la validation.
10. ProblemDetails permet de représenter les erreurs HTTP de manière structurée.
11. La validation côté serveur est obligatoire même si le frontend valide déjà les données.
12. Les règles métier complexes ne doivent pas toutes être placées dans les DTOs.
13. FluentValidation peut être utilisé pour organiser des validations plus complexes.
14. Les contraintes de la base de données restent importantes.
15. Validation, authentification et autorisation sont trois concepts différents.

---

# Questions d'entretien

### 1. Quelle différence entre Model Binding et Validation ?

Le Model Binding transforme les données HTTP en objets ou paramètres C#. La validation vérifie ensuite que ces données respectent les contraintes attendues.

### 2. Que fait `[ApiController]` concernant la validation ?

Il permet notamment à ASP.NET Core de gérer automatiquement les erreurs de validation et de retourner une réponse `400 Bad Request` avant l'exécution normale de l'action dans les cas concernés.

### 3. Quelle différence entre validation côté client et côté serveur ?

La validation client améliore l'expérience utilisateur, mais elle n'est pas fiable pour la sécurité. Le serveur doit toujours refaire les contrôles nécessaires.

### 4. Quand utiliser Data Annotations ?

Pour des règles simples et directement liées au contrat de données.

### 5. Pourquoi utiliser FluentValidation ?

Pour disposer d'une approche plus expressive et flexible, notamment lorsque les règles deviennent nombreuses ou complexes.

### 6. Où placer une règle métier ?

Dans la couche applicative/métier appropriée, et non uniquement dans le DTO ou le Controller.

### 7. Pourquoi une validation ne garantit-elle pas toujours l'intégrité des données ?

Parce que l'état du système peut changer entre la validation et l'opération finale. Les transactions, contraintes de base et mécanismes de concurrence peuvent donc rester nécessaires.

### 8. Quelle différence entre validation et authorization ?

La validation vérifie si les données sont acceptables. L'autorisation vérifie si l'utilisateur a le droit d'effectuer l'opération.

### 9. Quel status HTTP est généralement associé à une erreur de validation ?

`400 Bad Request`, lorsque la requête ne respecte pas le contrat attendu.

---

# Phrase à retenir

> **Le Model Binding construit les données C# à partir de la requête ; la validation vérifie qu'elles sont acceptables ; la logique métier vérifie ensuite qu'elles sont autorisées par les règles de l'application.**
