# Azure App Service

## 1. Qu'est-ce qu'Azure App Service ?

Azure App Service est un service **PaaS** permettant notamment d'héberger des applications web et des APIs.

Pour une application ASP.NET Core :

```text
Application .NET
      |
      v
Azure App Service
      |
      v
Internet
```

L'intérêt principal est de ne pas devoir gérer directement une machine virtuelle et son système d'exploitation.

Avec une VM, on doit davantage gérer :

```text
OS
Patches
Runtime
Serveur web
Réseau
Application
```

Avec App Service, Azure prend en charge une grande partie de l'infrastructure.

Mental model :

```text
VM
↓
Je gère davantage l'infrastructure

App Service
↓
Azure gère davantage la plateforme
```

---

# 2. Pourquoi utiliser App Service pour une API .NET ?

Pour une API ASP.NET Core classique, App Service permet notamment de :

```text
Héberger l'API
Exposer des endpoints HTTP/HTTPS
Configurer des variables d'environnement
Activer HTTPS
Déployer l'application
Consulter les logs
Configurer la mise à l'échelle
Connecter des services Azure
```

Une architecture simple :

```text
Client
  |
  | HTTPS
  v
Azure App Service
  |
  v
ASP.NET Core API
  |
  +---- Azure SQL
  |
  +---- Key Vault
```

---

# 3. PaaS

App Service est un exemple de :

```text
Platform as a Service
```

Le principe est :

```text
Azure
 |
 +-- Infrastructure
 +-- OS / plateforme
 +-- gestion du service
 |
 v
Ton application
```

Le développeur se concentre principalement sur :

```text
Code
Configuration
Données
Déploiement
Application
```

Cela permet d'aller plus vite qu'une approche basée sur une VM lorsque l'application correspond bien au modèle App Service.

---

# 4. App Service Plan

Une notion importante :

```text
App Service
```

et :

```text
App Service Plan
```

ne sont pas exactement la même chose.

## App Service

C'est l'application web ou l'API hébergée.

Exemple :

```text
hotel-api
```

## App Service Plan

Il définit notamment les ressources de calcul utilisées par les applications :

```text
CPU
Mémoire
Instances
Pricing tier
Scaling
```

Mental model :

```text
App Service Plan
       |
       +---- App Service A
       |
       +---- App Service B
```

Plusieurs applications peuvent partager un même App Service Plan.

---

# 5. Scale Up

Le **Scale Up** consiste à utiliser un niveau de ressources plus puissant.

Conceptuellement :

```text
Plan
 ↓
2 CPU / 4 GB
```

devient :

```text
Plan
 ↓
4 CPU / 8 GB
```

On augmente donc les ressources disponibles pour les instances.

Mental model :

```text
Scale Up
=
plus de puissance par instance
```

---

# 6. Scale Out

Le **Scale Out** consiste à augmenter le nombre d'instances.

Exemple :

```text
Instance 1
```

devient :

```text
Instance 1
Instance 2
Instance 3
```

Les requêtes peuvent alors être distribuées entre plusieurs instances selon l'infrastructure Azure.

Mental model :

```text
Scale Up
→ machine plus puissante

Scale Out
→ plus de machines / instances
```

---

# 7. Stateless : une notion importante

Le Scale Out est beaucoup plus simple lorsque l'application est **stateless**.

Stateless signifie que l'application ne dépend pas de l'état stocké uniquement dans une instance particulière.

Mauvais modèle :

```text
Client
  |
  v
Instance 1
  |
  +-- Session uniquement en mémoire
```

Puis :

```text
Client
  |
  v
Instance 2
```

L'instance 2 ne possède pas forcément les données de session de l'instance 1.

Pour une application distribuée, il est préférable de placer l'état partagé dans une ressource adaptée :

```text
Database
Distributed Cache
External Storage
```

Mental model :

```text
Application
   ↓
peu ou pas d'état local
   ↓
Scale Out plus simple
```

---

# 8. Configuration

Une application ASP.NET Core utilise :

```csharp
IConfiguration
```

pour accéder à sa configuration.

Exemple :

```csharp
var connectionString =
    builder.Configuration.GetConnectionString("Default");
```

En local, on peut avoir :

```text
appsettings.json
appsettings.Development.json
```

Dans Azure App Service, on peut configurer les paramètres de l'application dans Azure.

Le principe :

```text
Azure Configuration
       |
       v
Environment Variables
       |
       v
ASP.NET Core Configuration
       |
       v
IConfiguration
```

Le code peut ainsi rester identique entre les environnements.

---

# 9. Environnements

Une application possède généralement plusieurs environnements :

```text
Development
Test
Staging
Production
```

On veut éviter de mettre :

```text
Database Production
```

dans :

```text
appsettings.Development.json
```

Chaque environnement possède sa propre configuration.

Mental model :

```text
Même code
   |
   +---- Development
   |
   +---- Test
   |
   +---- Production
```

Ce qui change principalement :

```text
Configuration
Services
Secrets
Endpoints
```

---

# 10. Variables d'environnement

ASP.NET Core peut récupérer des valeurs depuis les variables d'environnement.

Exemple conceptuel :

```text
ConnectionStrings__Default
```

peut correspondre à une configuration imbriquée :

```json
{
  "ConnectionStrings": {
    "Default": "..."
  }
}
```

Le double underscore :

```text
__
```

est utilisé pour représenter une hiérarchie de configuration.

Dans le code :

```csharp
builder.Configuration
    .GetConnectionString("Default");
```

reste inchangé.

C'est très pratique pour les déploiements Cloud.

---

# 11. Secrets

Il ne faut pas placer les secrets directement dans le code source.

Mauvais :

```csharp
var password = "SuperSecret123";
```

Mauvais également :

```json
{
  "Password": "SuperSecret123"
}
```

dans un fichier versionné publiquement.

Pour une architecture Azure, on peut notamment utiliser :

```text
Key Vault
Managed Identity
Environment configuration
```

Le principe idéal :

```text
Code
  ↓
ne contient pas le secret

Application
  ↓
récupère le secret de manière sécurisée
```

---

# 12. Managed Identity avec App Service

Une application hébergée dans App Service peut utiliser une **Managed Identity**.

Mental model :

```text
App Service
     |
     | identité Azure
     v
Azure Key Vault
```

L'application n'a pas besoin de stocker un client secret simplement pour s'authentifier auprès d'un autre service Azure.

Exemple conceptuel :

```text
App Service
    |
    | Managed Identity
    v
Key Vault
    |
    v
Secret
```

La partie importante est l'autorisation :

```text
Managed Identity
       ↓
RBAC / permissions
       ↓
Key Vault
```

---

# 13. HTTPS

Une API exposée sur Internet doit utiliser HTTPS.

Mental model :

```text
Client
  |
  | HTTPS / TLS
  v
App Service
```

HTTPS protège notamment la communication entre le client et l'application contre plusieurs formes d'interception du trafic.

Pour une API qui reçoit :

```text
Passwords
Tokens
Personal data
Business data
```

le transport sécurisé est essentiel.

---

# 14. Custom Domain

App Service peut être associé à un nom de domaine personnalisé.

Au lieu de :

```text
mon-api.azurewebsites.net
```

on peut utiliser un domaine appartenant à l'application :

```text
api.example.com
```

Le principe :

```text
DNS
 ↓
api.example.com
 ↓
Azure App Service
```

---

# 15. Deployment

Une application peut être déployée vers App Service de différentes manières.

Exemples :

```text
Visual Studio
GitHub Actions
Azure CLI
Zip deployment
Azure DevOps
```

Pour un développeur, GitHub Actions est particulièrement intéressant lorsqu'on veut automatiser le déploiement.

Mental model :

```text
git push
   ↓
GitHub
   ↓
GitHub Actions
   ↓
Build
   ↓
Test
   ↓
Publish
   ↓
Azure App Service
```

---

# 16. CI/CD avec GitHub Actions

Un pipeline .NET classique peut ressembler à :

```yaml
- dotnet restore
- dotnet build
- dotnet test
- dotnet publish
- deploy to Azure
```

Le principe est :

```text
Developer
    |
    | git push
    v
GitHub
    |
    v
Workflow
    |
    +-- Restore
    +-- Build
    +-- Test
    +-- Publish
    |
    v
Azure App Service
```

L'objectif est d'éviter les déploiements manuels répétitifs.

---

# 17. Slot de déploiement

Les App Service Plans qui prennent en charge les deployment slots permettent de disposer de plusieurs environnements de déploiement pour une même application.

Conceptuellement :

```text
Production
     |
     +---- Staging
```

On peut déployer une nouvelle version dans :

```text
Staging
```

puis effectuer les vérifications nécessaires avant de la promouvoir vers :

```text
Production
```

Mental model :

```text
Code v2
  ↓
Staging
  ↓
Tests
  ↓
Swap / promotion
  ↓
Production
```

L'intérêt est de réduire les risques liés à un déploiement direct en production.

---

# 18. Logs

Lorsqu'une API fonctionne en local mais pas dans Azure, les logs deviennent essentiels.

Une application ASP.NET Core peut utiliser :

```csharp
ILogger<T>
```

Exemple :

```csharp
public class HotelService(
    ILogger<HotelService> logger)
{
    public void Execute()
    {
        logger.LogInformation(
            "Hotel service started");
    }
}
```

En production :

```text
Application
   ↓
Logs
   ↓
Azure monitoring / diagnostics
   ↓
Developer
```

Les logs permettent notamment de comprendre :

```text
Exception
Configuration incorrecte
Erreur réseau
Erreur SQL
Erreur d'authentification
```

---

# 19. Monitoring

Les logs ne sont pas les seuls éléments importants.

On peut également surveiller :

```text
CPU
Mémoire
Requêtes
Latence
Exceptions
Disponibilité
Dépendances
```

L'écosystème Azure fournit notamment :

```text
Azure Monitor
Application Insights
Log Analytics
```

Mental model :

```text
Application
   |
   +-- Logs
   +-- Metrics
   +-- Traces
   |
   v
Monitoring
```

---

# 20. Health Checks

Une API peut exposer un endpoint de santé.

Exemple :

```text
GET /health
```

ASP.NET Core permet de configurer les health checks.

Exemple :

```csharp
builder.Services.AddHealthChecks();

var app = builder.Build();

app.MapHealthChecks("/health");
```

Le endpoint peut ensuite être utilisé pour vérifier que l'application répond correctement.

Attention :

```text
Application répond
```

ne signifie pas nécessairement :

```text
Toutes les dépendances fonctionnent
```

On peut donc ajouter des vérifications pour certaines dépendances selon les besoins.

---

# 21. App Service et base de données

Une architecture courante :

```text
             App Service
                  |
                  |
                  v
             Azure SQL
```

L'API utilise une connection string ou un mécanisme d'authentification adapté.

Le code peut rester :

```csharp
builder.Configuration
    .GetConnectionString("Default");
```

La valeur réelle est fournie par la configuration de l'environnement.

On sépare ainsi :

```text
Code
```

et :

```text
Configuration de production
```

---

# 22. App Service et Key Vault

Architecture plus sécurisée :

```text
                App Service
                    |
                    |
             Managed Identity
                    |
                    v
                Key Vault
                    |
                    v
                  Secret
```

Exemples de secrets :

```text
Database credentials
API keys
Certificates
Connection information
```

L'idée est d'éviter de mettre directement ces informations dans Git.

---

# 23. Architecture complète

Une API ASP.NET Core peut être organisée ainsi :

```text
                         Internet
                            |
                            v
                     HTTPS / TLS
                            |
                            v
                    Azure App Service
                            |
              +-------------+-------------+
              |                           |
              v                           v
        Application API             Application Insights
              |
              | Managed Identity
              |
       +------+------+
       |             |
       v             v
   Key Vault      Azure SQL
```

Déploiement :

```text
Developer
    |
    | git push
    v
GitHub
    |
    v
GitHub Actions
    |
    +-- Restore
    +-- Build
    +-- Test
    +-- Publish
    |
    v
Azure App Service
```

---

# 24. App Service vs Virtual Machine

| App Service | Virtual Machine |
|---|---|
| PaaS | IaaS |
| Moins d'infrastructure à gérer | Plus d'administration |
| Déploiement simplifié | Plus de contrôle |
| Scaling intégré selon le plan | Scaling à gérer davantage |
| Idéal pour de nombreuses apps web/API | Utile lorsqu'un contrôle système important est nécessaire |

Le choix dépend des besoins.

App Service est très pratique lorsque :

```text
Application web/API
+
Besoin de simplicité
+
Peu de contrôle OS nécessaire
```

---

# 25. App Service vs Container

Une application ASP.NET Core peut également être déployée sous forme de conteneur.

Mental model :

```text
App Service
   |
   +-- Code déployé directement
   |
   ou
   |
   +-- Container
```

Le conteneur permet notamment de standardiser l'environnement d'exécution.

Mais cela ajoute également des notions :

```text
Docker
Image
Container Registry
Container configuration
```

Il faut donc choisir l'approche en fonction du projet.

---

# 26. Configuration à ne pas oublier

Lorsqu'une API fonctionne localement mais pas dans App Service, vérifier :

```text
[ ] Connection string
[ ] Environment variables
[ ] ASPNETCORE_ENVIRONMENT
[ ] Database access
[ ] HTTPS
[ ] Secrets
[ ] Managed Identity
[ ] Key Vault permissions
[ ] CORS
[ ] Logs
[ ] Health checks
[ ] Deployment
```

Très souvent, le problème n'est pas le code métier.

Il vient de la différence entre :

```text
Local
```

et :

```text
Production
```

---

# 27. Erreurs fréquentes

## Erreur 1 : mettre les secrets dans Git

```text
appsettings.json
    ↓
password production
    ↓
git push
```

À éviter.

---

## Erreur 2 : utiliser une configuration locale en production

Exemple :

```text
localhost
```

dans une connection string de production.

L'API Azure ne doit pas essayer de contacter le SQL Server local de ton PC.

---

## Erreur 3 : oublier les variables d'environnement

Une configuration présente sur le PC :

```text
User Secrets
```

n'est pas automatiquement disponible dans App Service.

---

## Erreur 4 : oublier les permissions Key Vault

Même avec une Managed Identity activée :

```text
Identity existe
```

ne signifie pas :

```text
Identity peut lire le secret
```

Il faut également lui attribuer les permissions nécessaires.

---

## Erreur 5 : confondre Scale Up et Scale Out

```text
Scale Up
→ plus de ressources

Scale Out
→ plus d'instances
```

---

# 28. Questions d'entretien

### Qu'est-ce qu'Azure App Service ?

Un service PaaS permettant notamment d'héberger des applications web et APIs.

### Qu'est-ce qu'un App Service Plan ?

Il fournit les ressources de calcul utilisées par les App Services et définit notamment le niveau de service et les possibilités de scaling.

### Quelle différence entre Scale Up et Scale Out ?

```text
Scale Up = augmenter les ressources d'une instance

Scale Out = augmenter le nombre d'instances
```

### Pourquoi une application stateless facilite-t-elle le Scale Out ?

Parce que plusieurs instances peuvent traiter les requêtes sans dépendre d'un état stocké uniquement dans une instance particulière.

### Comment stocker les secrets d'une API Azure ?

Selon le contexte :

```text
Key Vault
Managed Identity
Environment configuration
```

plutôt que de les placer dans le code source.

### Pourquoi utiliser Managed Identity ?

Pour permettre à une ressource Azure de s'authentifier auprès d'autres services Azure sans stocker explicitement un secret d'authentification.

### Comment automatiser un déploiement ?

Par exemple :

```text
GitHub Actions
    ↓
Build
    ↓
Test
    ↓
Publish
    ↓
Azure App Service
```

### Pourquoi utiliser un deployment slot ?

Pour déployer et tester une version séparément de la production avant sa promotion.

### Pourquoi les logs sont-ils importants ?

Parce qu'une application Cloud peut fonctionner différemment de l'environnement local. Les logs permettent de diagnostiquer les erreurs de configuration, de réseau, de base de données ou d'exécution.

---

# 29. Schéma mental final

```text
                    AZURE APP SERVICE
                           |
              +------------+------------+
              |                         |
          Application               Configuration
              |                         |
              |                    Environment
              |                    Variables
              |                         |
              v                         v
         ASP.NET Core              IConfiguration
              |
       +------+------+
       |             |
       v             v
   Azure SQL      Key Vault
                      ^
                      |
               Managed Identity
```

Déploiement :

```text
GitHub
   |
   v
GitHub Actions
   |
   +-- Build
   +-- Test
   +-- Publish
   |
   v
App Service
```

---

# À retenir

Azure App Service est une solution PaaS particulièrement adaptée à de nombreuses applications web et APIs ASP.NET Core.

Les notions fondamentales sont :

```text
App Service
    ↓
héberger l'application

App Service Plan
    ↓
ressources + scaling

Configuration
    ↓
adapter l'application à l'environnement

Key Vault
    ↓
protéger les secrets

Managed Identity
    ↓
authentification Azure sans secret applicatif

GitHub Actions
    ↓
automatiser le déploiement
```

## Phrase à mémoriser

> **App Service me permet de me concentrer sur mon application .NET pendant qu'Azure prend en charge une grande partie de l'infrastructure nécessaire pour l'héberger et la faire fonctionner.**
