# Options Pattern en .NET

## 1. Définition

Le **Options Pattern** permet de transformer une section de configuration en un objet C# fortement typé.

Au lieu de faire partout :

```csharp
var url = configuration["ApiSettings:BaseUrl"];
var timeout = configuration.GetValue<int>("ApiSettings:Timeout");
```

on peut créer une classe :

```csharp
public class ApiSettings
{
    public string BaseUrl { get; set; } = string.Empty;
    public int Timeout { get; set; }
}
```

Puis laisser .NET remplir cette classe depuis la configuration.

L'idée fondamentale est :

> Configuration externe -> objet C# fortement typé -> service.

---

# 2. Pourquoi utiliser le Options Pattern ?

Supposons cette configuration :

```json
{
  "ApiSettings": {
    "BaseUrl": "https://api.example.com",
    "Timeout": 30,
    "Enabled": true
  }
}
```

Sans Options Pattern :

```csharp
var baseUrl =
    configuration["ApiSettings:BaseUrl"];

var timeout =
    configuration.GetValue<int>(
        "ApiSettings:Timeout");

var enabled =
    configuration.GetValue<bool>(
        "ApiSettings:Enabled");
```

Le code dépend de plusieurs chaînes de configuration.

Avec le Options Pattern :

```csharp
public class ApiSettings
{
    public string BaseUrl { get; set; } = string.Empty;
    public int Timeout { get; set; }
    public bool Enabled { get; set; }
}
```

Puis :

```csharp
builder.Services.Configure<ApiSettings>(
    builder.Configuration.GetSection("ApiSettings"));
```

Le service reçoit directement :

```csharp
ApiSettings
```

---

# 3. Le fonctionnement global

Le processus est :

```text
appsettings.json
       |
       v
"ApiSettings"
       |
       v
Configuration section
       |
       v
Bind
       |
       v
ApiSettings
       |
       v
Service
```

Exemple :

```json
{
  "ApiSettings": {
    "BaseUrl": "https://api.example.com",
    "Timeout": 30
  }
}
```

devient conceptuellement :

```csharp
new ApiSettings
{
    BaseUrl = "https://api.example.com",
    Timeout = 30
};
```

C'est le **binding** de configuration.

---

# 4. Créer la classe d'options

Exemple :

```csharp
public class ApiSettings
{
    public string BaseUrl { get; set; } = string.Empty;

    public int Timeout { get; set; }

    public bool Enabled { get; set; }
}
```

La structure de la classe correspond à la structure de la configuration.

Configuration :

```json
{
  "ApiSettings": {
    "BaseUrl": "...",
    "Timeout": 30,
    "Enabled": true
  }
}
```

Classe :

```text
ApiSettings
    |
    +-- BaseUrl
    +-- Timeout
    +-- Enabled
```

---

# 5. Enregistrer les options

Dans `Program.cs` :

```csharp
builder.Services.Configure<ApiSettings>(
    builder.Configuration.GetSection("ApiSettings"));
```

On peut lire cela comme :

> « Lie la section `ApiSettings` à la classe `ApiSettings` et rends cette configuration disponible via le système DI. »

---

# 6. `IOptions<T>`

La forme classique consiste à injecter :

```csharp
IOptions<ApiSettings>
```

Exemple :

```csharp
public class ApiService
{
    private readonly ApiSettings _settings;

    public ApiService(IOptions<ApiSettings> options)
    {
        _settings = options.Value;
    }

    public void Execute()
    {
        var url = _settings.BaseUrl;
    }
}
```

`Value` contient l'objet configuré.

Mentalement :

```text
IOptions<ApiSettings>
          |
          v
       .Value
          |
          v
    ApiSettings
```

---

# 7. Pourquoi `IOptions<T>` ?

`IOptions<T>` fait le lien entre :

```text
Configuration
```

et :

```text
DI
```

Le service ne demande plus :

```csharp
IConfiguration
```

pour chercher lui-même des clés.

Il demande directement :

```csharp
IOptions<ApiSettings>
```

Cela rend la dépendance plus explicite.

---

# 8. `IOptions<T>` vs `IConfiguration`

### `IConfiguration`

```csharp
IConfiguration configuration
```

permet d'accéder directement aux clés.

Exemple :

```csharp
configuration["ApiSettings:BaseUrl"];
```

### `IOptions<T>`

```csharp
IOptions<ApiSettings> options
```

donne un objet fortement typé.

Exemple :

```csharp
options.Value.BaseUrl
```

Mentalement :

```text
IConfiguration
    |
    +-- accès générique aux données

IOptions<T>
    |
    +-- configuration transformée en objet métier/technique
```

---

# 9. `IOptionsSnapshot<T>`

ASP.NET Core fournit également :

```csharp
IOptionsSnapshot<T>
```

Il est principalement utilisé dans les applications avec des scopes, notamment les applications web.

Exemple :

```csharp
public class ApiService
{
    private readonly ApiSettings _settings;

    public ApiService(
        IOptionsSnapshot<ApiSettings> options)
    {
        _settings = options.Value;
    }
}
```

Une différence importante est que `IOptionsSnapshot<T>` peut représenter une configuration recalculée pour chaque scope.

Dans une application ASP.NET Core :

```text
Request 1 -> snapshot de la configuration
Request 2 -> snapshot de la configuration
```

Il est donc utile lorsque la configuration peut être rechargée et qu'on veut prendre en compte les changements dans les scopes suivants.

---

# 10. `IOptionsMonitor<T>`

Pour observer une configuration qui peut changer, on peut utiliser :

```csharp
IOptionsMonitor<T>
```

Exemple :

```csharp
public class ApiService
{
    private readonly IOptionsMonitor<ApiSettings> _options;

    public ApiService(
        IOptionsMonitor<ApiSettings> options)
    {
        _options = options;
    }

    public void Execute()
    {
        var settings = _options.CurrentValue;

        // ...
    }
}
```

La propriété importante est :

```csharp
CurrentValue
```

Elle représente la valeur actuelle des options.

---

# 11. Comparaison des trois principales interfaces

| Interface | Idée principale |
|---|---|
| `IOptions<T>` | configuration classique |
| `IOptionsSnapshot<T>` | valeur liée au scope |
| `IOptionsMonitor<T>` | accès à la valeur actuelle et suivi des changements |

### Règle mentale

```text
IOptions
    -> simple

IOptionsSnapshot
    -> par scope

IOptionsMonitor
    -> surveiller les changements
```

---

# 12. Binding

Le **binding** consiste à faire correspondre les données de configuration aux propriétés d'un objet.

Exemple :

```json
{
  "Database": {
    "Server": "localhost",
    "Port": 1433
  }
}
```

Classe :

```csharp
public class DatabaseSettings
{
    public string Server { get; set; } = string.Empty;
    public int Port { get; set; }
}
```

Puis :

```csharp
builder.Services.Configure<DatabaseSettings>(
    builder.Configuration.GetSection("Database"));
```

Le binder fait conceptuellement :

```text
Database:Server -> DatabaseSettings.Server
Database:Port   -> DatabaseSettings.Port
```

---

# 13. Propriétés imbriquées

La configuration peut être plus complexe.

```json
{
  "Email": {
    "Host": "smtp.example.com",
    "Port": 587,
    "Credentials": {
      "Username": "user"
    }
  }
}
```

On peut représenter cela avec :

```csharp
public class EmailSettings
{
    public string Host { get; set; } = string.Empty;

    public int Port { get; set; }

    public EmailCredentials Credentials { get; set; } = new();
}

public class EmailCredentials
{
    public string Username { get; set; } = string.Empty;
}
```

Le binder peut remplir la structure correspondante.

---

# 14. Validation des options

Une configuration incorrecte peut provoquer des erreurs difficiles à diagnostiquer.

Exemple :

```csharp
public class ApiSettings
{
    public string BaseUrl { get; set; } = string.Empty;

    public int Timeout { get; set; }
}
```

On peut utiliser des attributs de validation :

```csharp
public class ApiSettings
{
    [Required]
    public string BaseUrl { get; set; } = string.Empty;

    [Range(1, 300)]
    public int Timeout { get; set; }
}
```

Puis configurer la validation.

```csharp
builder.Services
    .AddOptions<ApiSettings>()
    .Bind(builder.Configuration.GetSection("ApiSettings"))
    .ValidateDataAnnotations();
```

L'idée est :

```text
Configuration
      |
      v
Binding
      |
      v
Validation
      |
      v
Application
```

---

# 15. Validation avec une fonction

On peut également écrire une validation personnalisée :

```csharp
builder.Services
    .AddOptions<ApiSettings>()
    .Bind(builder.Configuration.GetSection("ApiSettings"))
    .Validate(settings =>
        settings.Timeout > 0,
        "Timeout must be greater than 0");
```

Cela permet d'exprimer une règle qui n'est pas simplement une annotation.

---

# 16. Validation au démarrage

Une configuration invalide peut être détectée au démarrage plutôt qu'au premier accès.

On peut demander une validation au démarrage :

```csharp
builder.Services
    .AddOptions<ApiSettings>()
    .Bind(builder.Configuration.GetSection("ApiSettings"))
    .ValidateDataAnnotations()
    .ValidateOnStart();
```

Mentalement :

```text
Sans ValidateOnStart
    |
    v
Erreur potentiellement découverte plus tard

Avec ValidateOnStart
    |
    v
Erreur détectée au démarrage
```

C'est particulièrement intéressant pour les paramètres obligatoires d'une application.

---

# 17. Options nommées

Il peut arriver qu'une application possède plusieurs configurations du même type.

Par exemple :

```text
Payment provider A
Payment provider B
```

On peut utiliser des **named options**.

Exemple :

```csharp
builder.Services.Configure<ApiSettings>(
    "Primary",
    builder.Configuration.GetSection("PrimaryApi"));

builder.Services.Configure<ApiSettings>(
    "Secondary",
    builder.Configuration.GetSection("SecondaryApi"));
```

Puis récupérer une configuration spécifique avec le nom correspondant.

L'idée est :

```text
ApiSettings
   |
   +-- Primary
   |
   +-- Secondary
```

Cela évite de créer une nouvelle classe uniquement parce qu'il existe plusieurs instances de la même configuration.

---

# 18. Options et Dependency Injection

Le Options Pattern s'intègre directement à la DI.

Exemple :

```csharp
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection("JwtSettings"));
```

Puis :

```csharp
public class TokenService
{
    private readonly JwtSettings _settings;

    public TokenService(IOptions<JwtSettings> options)
    {
        _settings = options.Value;
    }
}
```

Le graphe devient :

```text
TokenService
      |
      v
IOptions<JwtSettings>
      |
      v
JwtSettings
```

---

# 19. Exemple avec JWT

Configuration :

```json
{
  "JwtSettings": {
    "Issuer": "MyApi",
    "Audience": "MyClient",
    "ExpirationMinutes": 60
  }
}
```

Classe :

```csharp
public class JwtSettings
{
    public string Issuer { get; set; } = string.Empty;

    public string Audience { get; set; } = string.Empty;

    public int ExpirationMinutes { get; set; }
}
```

Enregistrement :

```csharp
builder.Services.Configure<JwtSettings>(
    builder.Configuration.GetSection("JwtSettings"));
```

Service :

```csharp
public class TokenService
{
    private readonly JwtSettings _settings;

    public TokenService(
        IOptions<JwtSettings> options)
    {
        _settings = options.Value;
    }
}
```

Le service ne connaît pas :

```text
appsettings.json
```

Il connaît simplement :

```text
JwtSettings
```

C'est un découplage important.

---

# 20. Pourquoi le strongly typed configuration est intéressant ?

Sans Options Pattern :

```csharp
var issuer =
    configuration["JwtSettings:Issuer"];

var audience =
    configuration["JwtSettings:Audience"];

var expiration =
    configuration.GetValue<int>(
        "JwtSettings:ExpirationMinutes");
```

Avec Options Pattern :

```csharp
_settings.Issuer
_settings.Audience
_settings.ExpirationMinutes
```

Avantages :

- typage ;
- IntelliSense ;
- meilleure lisibilité ;
- moins de chaînes magiques ;
- dépendance plus explicite ;
- validation centralisée.

---

# 21. Options et tests

Le Options Pattern facilite également les tests.

Au lieu de construire toute la configuration :

```csharp
var configuration = ...
```

on peut fournir directement les options.

Conceptuellement :

```csharp
var settings = new ApiSettings
{
    BaseUrl = "https://test.example.com",
    Timeout = 10
};
```

Puis fournir un objet d'options adapté au test.

Le service reste indépendant du fichier `appsettings.json`.

---

# 22. Ne pas mettre toute la configuration dans une seule classe

Mauvais :

```csharp
public class AppSettings
{
    public string Database { get; set; }
    public string JwtSecret { get; set; }
    public string EmailHost { get; set; }
    public int CacheDuration { get; set; }
    public string StorageUrl { get; set; }
    // 50 autres propriétés...
}
```

Préférer des groupes cohérents :

```text
DatabaseSettings
JwtSettings
EmailSettings
StorageSettings
CacheSettings
```

Cela respecte mieux la séparation des responsabilités.

---

# 23. Options vs variables de configuration simples

Pour une seule valeur :

```json
{
  "ApplicationName": "HotelListing"
}
```

on peut simplement utiliser :

```csharp
var name = configuration["ApplicationName"];
```

Créer une classe uniquement pour une valeur très simple peut être inutile.

Le Options Pattern devient particulièrement intéressant lorsque plusieurs valeurs forment une configuration cohérente.

---

# 24. Options vs `IConfiguration`

La question à se poser :

### Besoin ponctuel d'une valeur ?

```csharp
IConfiguration
```

peut être suffisant.

### Groupe de paramètres utilisé par un service ?

```csharp
IOptions<T>
```

est généralement plus propre.

Exemple :

```text
JwtSettings
EmailSettings
StorageSettings
PaymentSettings
```

sont de bons candidats pour le Options Pattern.

---

# 25. Options et secrets

Une classe d'options peut contenir une valeur sensible :

```csharp
public class DatabaseSettings
{
    public string ConnectionString { get; set; } = string.Empty;
}
```

Mais le fait d'utiliser Options Pattern ne rend pas automatiquement le secret sécurisé.

Le secret doit toujours provenir d'une source adaptée :

```text
User Secrets
Environment Variables
Azure Key Vault
Secret Manager
```

Mentalement :

> Options Pattern organise la configuration ; il ne sécurise pas les secrets à lui seul.

---

# 26. Erreurs fréquentes

## Erreur 1 : utiliser `IConfiguration` partout

```csharp
public UserService(IConfiguration configuration)
```

puis :

```csharp
configuration["Jwt:Issuer"]
configuration["Jwt:Audience"]
configuration["Jwt:Expiration"]
```

Pour une configuration structurée, préférer une classe d'options.

---

## Erreur 2 : oublier le binding

Créer :

```csharp
public class ApiSettings
{
}
```

ne suffit pas.

Il faut enregistrer la section :

```csharp
builder.Services.Configure<ApiSettings>(
    builder.Configuration.GetSection("ApiSettings"));
```

---

## Erreur 3 : croire que `IOptions<T>` valide automatiquement

Le binding et la validation sont deux concepts différents.

```text
Binding
    -> remplit l'objet

Validation
    -> vérifie que l'objet est correct
```

---

## Erreur 4 : mettre les secrets en clair dans Git

Le Options Pattern ne change rien à cette règle.

---

## Erreur 5 : créer une classe géante

Une classe d'options doit représenter une configuration cohérente.

---

# 27. Schéma global

```text
             appsettings.json
                    |
                    v
             IConfiguration
                    |
                    v
          GetSection("ApiSettings")
                    |
                    v
              Binding
                    |
                    v
              ApiSettings
                    |
             +------+------+
             |             |
             v             v
       Validation       DI
             |             |
             +------+------+
                    |
                    v
                 Service
```

---

# 28. Règle mentale

Quand tu vois :

```csharp
builder.Services.Configure<ApiSettings>(
    configuration.GetSection("ApiSettings"));
```

pense :

> « Je prends une section de configuration et je la transforme en objet `ApiSettings` géré par le système d'options. »

Quand tu vois :

```csharp
IOptions<ApiSettings>
```

pense :

> « Donne-moi la configuration `ApiSettings`. »

Quand tu vois :

```csharp
options.Value
```

pense :

> « Voici l'objet `ApiSettings` réellement configuré. »

Quand tu vois :

```csharp
IOptionsSnapshot<T>
```

pense :

> « Options liées au scope. »

Quand tu vois :

```csharp
IOptionsMonitor<T>
```

pense :

> « Je veux accéder à la valeur actuelle et pouvoir suivre les changements. »

---

# 29. À retenir

1. Le Options Pattern transforme une configuration en objet C# fortement typé.
2. `Configure<T>()` réalise le lien entre une section de configuration et une classe.
3. `IOptions<T>` donne accès à la configuration classique via `.Value`.
4. `IOptionsSnapshot<T>` est lié au scope.
5. `IOptionsMonitor<T>` permet d'accéder à la valeur actuelle et de réagir aux changements.
6. Le binding remplit l'objet.
7. La validation vérifie que l'objet est correct.
8. `ValidateOnStart()` permet de détecter certaines erreurs de configuration au démarrage.
9. Les options nommées permettent plusieurs configurations du même type.
10. Le Options Pattern améliore le typage, la lisibilité et la testabilité.
11. Il réduit les chaînes de configuration dispersées dans le code.
12. Options Pattern n'est pas un système de sécurité des secrets.
13. Une classe d'options doit représenter une configuration cohérente.
14. `IConfiguration` reste utile pour les accès simples ou génériques.

---

# Questions d'entretien

### 1. Qu'est-ce que le Options Pattern ?

C'est un mécanisme permettant de lier une section de configuration à une classe C# fortement typée et de rendre cette configuration disponible via DI.

### 2. Pourquoi utiliser `IOptions<T>` ?

Pour accéder à une configuration structurée sous forme d'un objet fortement typé plutôt que de manipuler des chaînes de configuration.

### 3. Quelle différence entre `IOptions<T>` et `IOptionsSnapshot<T>` ?

`IOptions<T>` représente une configuration classique, tandis que `IOptionsSnapshot<T>` est conçue pour fournir une valeur liée au scope, notamment dans les applications web.

### 4. Quelle différence entre `IOptionsSnapshot<T>` et `IOptionsMonitor<T>` ?

`IOptionsSnapshot<T>` fonctionne par scope. `IOptionsMonitor<T>` permet d'accéder à la valeur actuelle et de surveiller les changements de configuration.

### 5. Qu'est-ce que le binding ?

C'est le mécanisme qui fait correspondre les valeurs de configuration aux propriétés d'un objet C#.

### 6. Le Options Pattern protège-t-il les secrets ?

Non. Il organise et typifie la configuration, mais les secrets doivent provenir d'une source sécurisée.

### 7. Pourquoi utiliser `ValidateOnStart()` ?

Pour détecter les erreurs de configuration dès le démarrage plutôt que lors de la première utilisation.

### 8. Pourquoi ne pas mettre toutes les configurations dans `AppSettings` ?

Parce qu'une classe d'options doit représenter une configuration cohérente et respecter la séparation des responsabilités.

---

# Phrase à retenir

> **Le Options Pattern transforme une section de configuration externe en objet C# fortement typé, injectable et éventuellement validé.**
