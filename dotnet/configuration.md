# Configuration en .NET

## 1. Définition

La **configuration** permet à une application .NET de récupérer des paramètres externes à son code.

Exemples :

```text
Connection strings
Clés API
URLs de services
Paramètres métier
Options de l'application
Environnement de déploiement
```

L'idée fondamentale est :

> Le code ne devrait généralement pas contenir en dur les paramètres qui peuvent changer selon l'environnement.

Par exemple, éviter :

```csharp
var connectionString =
    "Server=localhost;Database=HotelDb;...";
```

Préférer une configuration externe :

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "..."
  }
}
```

---

# 2. Pourquoi séparer la configuration du code ?

Une application peut fonctionner dans plusieurs environnements :

```text
Development
    |
    v
localhost / base de développement

Production
    |
    v
serveur / base de production
```

Le code métier reste le même.

Ce qui change :

```text
Configuration
```

Mentalement :

```text
CODE
  |
  +--> logique de l'application

CONFIGURATION
  |
  +--> valeurs qui changent selon l'environnement
```

Cela facilite :

- le déploiement ;
- les tests ;
- la maintenance ;
- la séparation Development / Production ;
- la gestion des secrets.

---

# 3. `appsettings.json`

ASP.NET Core utilise très souvent :

```text
appsettings.json
```

Exemple :

```json
{
  "Application": {
    "Name": "HotelListing",
    "Version": "1.0"
  }
}
```

On peut ensuite récupérer une valeur depuis la configuration.

```csharp
var name = builder.Configuration["Application:Name"];
```

Le `:` permet de naviguer dans une structure imbriquée.

```text
Application
    |
    +-- Name
    +-- Version
```

Donc :

```csharp
"Application:Name"
```

signifie :

```text
Application -> Name
```

---

# 4. `IConfiguration`

L'abstraction principale est :

```csharp
IConfiguration
```

Elle représente une source de configuration composée de clés et de valeurs.

Exemple :

```csharp
public class MyService
{
    private readonly IConfiguration _configuration;

    public MyService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void Execute()
    {
        var value = _configuration["Application:Name"];
    }
}
```

ASP.NET Core fournit automatiquement `IConfiguration` via la DI.

---

# 5. Plusieurs sources de configuration

Un point important est que .NET ne dépend pas uniquement de `appsettings.json`.

La configuration peut provenir de plusieurs sources :

```text
appsettings.json
       +
appsettings.{Environment}.json
       +
Environment Variables
       +
Command-line arguments
       +
User Secrets
       +
autres providers
```

Ces sources sont combinées pour construire une configuration finale.

---

# 6. `appsettings.Development.json`

On trouve souvent :

```text
appsettings.json
appsettings.Development.json
appsettings.Production.json
```

Exemple :

### `appsettings.json`

```json
{
  "Application": {
    "Name": "HotelListing"
  }
}
```

### `appsettings.Development.json`

```json
{
  "Application": {
    "Name": "HotelListing Development"
  }
}
```

Lorsque l'application tourne en Development, la configuration spécifique à Development peut remplacer les valeurs correspondantes.

Mentalement :

```text
appsettings.json
        +
Development overrides
        =
configuration finale
```

---

# 7. Environnement de l'application

ASP.NET Core utilise notamment :

```text
Development
Staging
Production
```

On peut récupérer l'environnement avec :

```csharp
builder.Environment.EnvironmentName
```

Ou utiliser :

```csharp
builder.Environment.IsDevelopment()
```

Exemple :

```csharp
if (builder.Environment.IsDevelopment())
{
    // comportement spécifique au développement
}
```

---

# 8. Variables d'environnement

Une application peut recevoir une configuration via les variables d'environnement.

Exemple conceptuel :

```text
ConnectionStrings__DefaultConnection=...
```

Le double underscore :

```text
__
```

permet de représenter une hiérarchie.

Ainsi :

```text
ConnectionStrings__DefaultConnection
```

correspond à :

```text
ConnectionStrings:DefaultConnection
```

C'est particulièrement utile dans les environnements de déploiement.

---

# 9. Pourquoi utiliser les variables d'environnement ?

Supposons :

```text
Production
```

On ne veut généralement pas mettre directement un mot de passe de base de données dans Git.

On peut plutôt configurer la valeur dans l'environnement de déploiement.

```text
Git
 |
 +-- code
 +-- configuration non sensible

Serveur
 |
 +-- secrets
 +-- variables d'environnement
```

Le code reste identique.

---

# 10. Les secrets

Une erreur fréquente consiste à mettre ceci dans Git :

```json
{
  "ApiKey": "ma-cle-secrete"
}
```

ou :

```json
{
  "Password": "SuperSecret123"
}
```

Même dans un projet privé, il faut être prudent.

Pour le développement local, .NET propose notamment **User Secrets**.

---

# 11. User Secrets

User Secrets permet de stocker des secrets en dehors du fichier `appsettings.json` du projet.

Exemple :

```bash
dotnet user-secrets init
```

Puis :

```bash
dotnet user-secrets set "ApiKey" "secret-value"
```

Le code peut ensuite lire :

```csharp
var apiKey = builder.Configuration["ApiKey"];
```

L'objectif est principalement d'éviter de committer des secrets de développement dans le dépôt.

---

# 12. Configuration et secrets en production

En production, on peut utiliser des mécanismes adaptés à l'infrastructure :

```text
Environment Variables
Azure Key Vault
Managed Identity
Secret managers
```

Dans un environnement Azure, **Azure Key Vault** est particulièrement important pour la gestion des secrets.

La fiche configuration doit toutefois garder une distinction claire :

```text
Configuration
    |
    +-- paramètres normaux
    |
    +-- secrets
```

Les secrets nécessitent des mesures de protection supplémentaires.

---

# 13. Lire une valeur simple

Avec :

```json
{
  "ApiSettings": {
    "BaseUrl": "https://api.example.com"
  }
}
```

On peut faire :

```csharp
var url = builder.Configuration["ApiSettings:BaseUrl"];
```

Mais cette approche a une limite : tout devient une chaîne de caractères.

---

# 14. Le problème des chaînes magiques

Exemple :

```csharp
var url = configuration["ApiSettings:BaseUrl"];
var timeout = configuration["ApiSettings:Timeout"];
var enabled = configuration["ApiSettings:Enabled"];
```

On utilise des chaînes partout :

```text
"ApiSettings:BaseUrl"
"ApiSettings:Timeout"
"ApiSettings:Enabled"
```

Cela peut devenir difficile à maintenir.

C'est notamment pour cette raison que .NET propose le **Options Pattern**.

---

# 15. `GetValue<T>`

Pour convertir directement une valeur :

```csharp
var timeout =
    configuration.GetValue<int>("ApiSettings:Timeout");
```

Ou :

```csharp
var enabled =
    configuration.GetValue<bool>("ApiSettings:Enabled");
```

On évite ainsi de convertir manuellement :

```csharp
int.Parse(...)
bool.Parse(...)
```

---

# 16. `GetSection`

On peut récupérer une section :

```csharp
var section =
    configuration.GetSection("ApiSettings");
```

Puis :

```csharp
var url = section["BaseUrl"];
```

Cela peut être utile lorsque plusieurs valeurs appartiennent à la même section.

---

# 17. Options Pattern

Pour une configuration structurée, on préfère souvent créer une classe :

```csharp
public class ApiSettings
{
    public string BaseUrl { get; set; } = string.Empty;

    public int Timeout { get; set; }

    public bool Enabled { get; set; }
}
```

Configuration :

```json
{
  "ApiSettings": {
    "BaseUrl": "https://api.example.com",
    "Timeout": 30,
    "Enabled": true
  }
}
```

Puis on lie la section à la classe.

```csharp
builder.Services.Configure<ApiSettings>(
    builder.Configuration.GetSection("ApiSettings"));
```

Cela correspond au **Options Pattern**.

Une fiche séparée lui sera consacrée.

---

# 18. `IOptions<T>`

Une fois les options enregistrées :

```csharp
builder.Services.Configure<ApiSettings>(
    builder.Configuration.GetSection("ApiSettings"));
```

On peut injecter :

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
}
```

Le service travaille alors avec un objet fortement typé plutôt qu'avec des chaînes.

---

# 19. Pourquoi la configuration est-elle une abstraction ?

Le code ne devrait pas avoir besoin de savoir si la valeur vient de :

```text
JSON
Environment Variable
User Secrets
Azure Key Vault
Command line
```

Il demande simplement :

```csharp
IConfiguration
```

ou une option fortement typée.

Mentalement :

```text
Source 1 ----\
Source 2 -----+
Source 3 -----+----> Configuration ----> Application
Source 4 -----+
```

Le provider est caché derrière l'abstraction.

---

# 20. Ordre et surcharge des providers

Lorsque plusieurs sources fournissent la même clé, une valeur provenant d'une source avec une priorité supérieure peut remplacer celle d'une source précédente.

Conceptuellement :

```text
appsettings.json
       |
       v
appsettings.Development.json
       |
       v
Environment Variables
       |
       v
Command Line
```

La valeur finale dépend donc de l'ordre des providers et de l'environnement.

### Mental model

> Plusieurs sources alimentent une seule configuration finale.

---

# 21. Configuration dans `Program.cs`

Avec le modèle moderne de .NET :

```csharp
var builder = WebApplication.CreateBuilder(args);
```

`builder` contient notamment :

```csharp
builder.Configuration
builder.Environment
builder.Services
```

La configuration est donc disponible très tôt dans le démarrage de l'application.

Exemple :

```csharp
var connectionString =
    builder.Configuration.GetConnectionString("DefaultConnection");
```

---

# 22. Connection Strings

Pour les chaînes de connexion, on utilise souvent :

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Server=...;Database=..."
  }
}
```

Puis :

```csharp
var connectionString =
    builder.Configuration.GetConnectionString(
        "DefaultConnection");
```

Cette convention est particulièrement utilisée avec Entity Framework Core.

Exemple :

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlServer(
        builder.Configuration.GetConnectionString(
            "DefaultConnection"));
});
```

---

# 23. Ne pas confondre configuration et logique métier

La configuration contient des paramètres.

Elle ne doit pas devenir un endroit où l'on met la logique métier.

Mauvais concept :

```text
appsettings.json
    |
    +-- calcul du prix
    +-- règles métier
    +-- logique de validation
```

Bon concept :

```text
appsettings.json
    |
    +-- paramètres

Service métier
    |
    +-- règles métier
```

---

# 24. Configuration fortement typée

Pour quelques valeurs simples :

```csharp
configuration["App:Name"]
```

peut être suffisant.

Pour une configuration plus importante :

```text
ApiSettings
EmailSettings
JwtSettings
StorageSettings
```

il est généralement préférable d'utiliser des classes fortement typées avec le Options Pattern.

Cela donne :

```text
JSON
  |
  v
ApiSettings
  |
  v
Service
```

au lieu de :

```text
JSON
  |
  v
"ApiSettings:BaseUrl"
"ApiSettings:Timeout"
"ApiSettings:Enabled"
```

---

# 25. Validation de configuration

Une configuration invalide peut provoquer des erreurs difficiles à diagnostiquer.

Exemple :

```text
BaseUrl manquante
Timeout négatif
Clé API vide
```

Avec les options fortement typées, on peut ajouter de la validation.

Conceptuellement :

```csharp
builder.Services
    .AddOptions<ApiSettings>()
    .Bind(builder.Configuration.GetSection("ApiSettings"))
    .Validate(settings =>
        settings.Timeout > 0,
        "Timeout must be greater than 0");
```

Le principe est important :

> Valider la configuration le plus tôt possible permet de détecter les erreurs avant qu'elles deviennent des erreurs métier.

La validation détaillée appartient toutefois au sujet Options Pattern.

---

# 26. Configuration et Azure

Dans Azure, l'application peut recevoir sa configuration depuis plusieurs mécanismes.

Exemple :

```text
Azure App Service
       |
       +-- Application Settings
       |
       +-- Environment Variables
       |
       +-- Key Vault
```

L'application .NET peut continuer à utiliser :

```csharp
IConfiguration
```

sans que le code métier ait besoin de connaître la source exacte.

C'est un avantage majeur de l'abstraction de configuration.

---

# 27. Erreurs fréquentes

## Erreur 1 : mettre les secrets dans Git

Éviter :

```json
{
  "Password": "mon-vrai-mot-de-passe"
}
```

Utiliser plutôt :

```text
User Secrets
Environment Variables
Key Vault
```

selon l'environnement.

---

## Erreur 2 : multiplier les chaînes magiques

```csharp
configuration["ApiSettings:Timeout"]
```

répété dans de nombreuses classes peut devenir difficile à maintenir.

Pour une configuration structurée, utiliser le Options Pattern.

---

## Erreur 3 : confondre configuration et logique métier

Les fichiers de configuration doivent contenir des paramètres, pas les règles principales de l'application.

---

## Erreur 4 : supposer que `appsettings.json` est la seule source

.NET peut agréger plusieurs providers.

---

## Erreur 5 : ne pas différencier les environnements

Une configuration adaptée au développement n'est pas forcément adaptée à la production.

---

# 28. Schéma global

Une bonne représentation mentale :

```text
                    SOURCES
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
 appsettings.json   Environment   User Secrets
        |           Variables         |
        +--------------+--------------+
                       |
                       v
                IConfiguration
                       |
             +---------+---------+
             |                   |
             v                   v
      Lecture directe       Options Pattern
             |                   |
             v                   v
        Service            Objet fortement typé
```

---

# 29. Règle mentale

Quand tu vois :

```csharp
IConfiguration
```

pense :

> « Je demande accès à la configuration agrégée de l'application. »

Quand tu vois :

```csharp
"Section:Key"
```

pense :

> « Je navigue dans une configuration hiérarchique. »

Quand tu vois :

```csharp
GetValue<int>()
```

pense :

> « Je récupère une valeur et je la convertis vers le type demandé. »

Quand tu vois :

```csharp
Configure<MySettings>()
```

pense :

> « Je transforme une section de configuration en objet fortement typé. »

---

# 30. À retenir

1. La configuration sépare les paramètres du code.
2. `IConfiguration` représente la configuration agrégée.
3. `appsettings.json` est une source de configuration, pas la seule.
4. `appsettings.Development.json` permet des valeurs spécifiques à Development.
5. Les variables d'environnement sont particulièrement utiles en déploiement.
6. Les secrets ne doivent pas être commités dans Git.
7. User Secrets est utile pour le développement local.
8. Azure Key Vault est adapté à la gestion des secrets dans Azure.
9. `:` représente une hiérarchie de configuration.
10. `__` permet de représenter cette hiérarchie dans les variables d'environnement.
11. `GetValue<T>` permet de récupérer une valeur typée.
12. `GetConnectionString()` est pratique pour les connection strings.
13. Pour une configuration structurée, le Options Pattern est généralement préférable.
14. Plusieurs providers peuvent être combinés pour construire la configuration finale.
15. Le code métier ne devrait pas dépendre directement de la source de configuration.

---

# Questions d'entretien

### 1. Qu'est-ce que `IConfiguration` ?

C'est l'abstraction qui permet à l'application .NET d'accéder à la configuration provenant de différentes sources.

### 2. `appsettings.json` est-il la seule source de configuration ?

Non. .NET peut agréger plusieurs providers, notamment les fichiers JSON, variables d'environnement, User Secrets et arguments de ligne de commande.

### 3. Pourquoi ne faut-il pas mettre les secrets dans `appsettings.json` commité ?

Parce que le dépôt peut exposer ces secrets. Il faut utiliser un mécanisme de gestion des secrets adapté à l'environnement.

### 4. À quoi sert `appsettings.Development.json` ?

À fournir des valeurs spécifiques à l'environnement Development.

### 5. Pourquoi utiliser le Options Pattern ?

Pour transformer une configuration structurée en objets fortement typés, ce qui améliore la lisibilité et la maintenabilité.

### 6. Quelle différence entre `IConfiguration` et `IOptions<T>` ?

`IConfiguration` permet d'accéder directement aux clés de configuration. `IOptions<T>` représente une configuration liée à une classe fortement typée.

### 7. Pourquoi utiliser des variables d'environnement en production ?

Elles permettent de fournir des paramètres au processus sans devoir les stocker dans le code source ou dans les fichiers versionnés.

### 8. Comment fonctionne une clé comme `Database:ConnectionString` ?

`Database` représente la section et `ConnectionString` représente la clé à l'intérieur de cette section.

---

# Phrase à retenir

> **La configuration permet de changer les paramètres de l'application sans changer son code ; .NET rassemble plusieurs sources derrière `IConfiguration`, puis le Options Pattern permet de transformer cette configuration en objets fortement typés.**
