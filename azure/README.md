# Azure — README

Cette section regroupe les notions Azure utiles pour un développeur .NET, notamment pour déployer, sécuriser et automatiser une API ASP.NET Core.

L'objectif n'est pas de mémoriser une liste de services Azure, mais de comprendre **quel problème chaque service résout et comment les services s'intègrent ensemble**.

---

## 1. La carte mentale Azure

Pour une application .NET classique :

```text
                    Azure
                      |
        +-------------+-------------+
        |             |             |
      Compute       Data         Security
        |             |             |
   App Service    Azure SQL     Key Vault
        |
        +------------------+
        |
    Deployment
        |
   GitHub Actions
```

On peut donc raisonner par problème :

```text
"Je dois héberger mon API"
        ↓
Azure App Service

"Je dois stocker mes données SQL"
        ↓
Azure SQL Database

"Je dois protéger mes secrets"
        ↓
Azure Key Vault

"Je ne veux pas stocker de mot de passe Azure dans mon code"
        ↓
Managed Identity

"Je veux déployer automatiquement depuis GitHub"
        ↓
GitHub Actions
```

---

# 2. Les fichiers de cette section

## App Service

[Azure App Service](app-service.md)

À étudier pour comprendre :

```text
Web App
Hosting
Configuration
Environment Variables
Scaling
Deployment
Logs
```

C'est l'un des services les plus importants à connaître pour déployer une application ASP.NET Core.

---

## Azure SQL

[Azure SQL Database](azure-sql.md)

À étudier pour comprendre :

```text
Database as a Service
SQL Server dans Azure
Connection Strings
Firewall
Authentication
Backups
Scaling
```

---

## Key Vault

[Azure Key Vault](key-vault.md)

À étudier pour comprendre :

```text
Secrets
Keys
Certificates
Secret management
Application configuration
```

L'objectif est notamment d'éviter de mettre des secrets sensibles directement dans :

```text
appsettings.json
Code source
Git
```

---

## Managed Identity

[Azure Managed Identity](managed-identity.md)

À étudier pour comprendre comment une ressource Azure peut s'authentifier auprès d'autres services Azure sans stocker explicitement un secret dans l'application.

Mental model :

```text
App Service
    |
    | Managed Identity
    v
Key Vault
```

Au lieu de :

```text
App Service
    |
    | Client Secret dans la configuration
    v
Key Vault
```

---

## GitHub Actions

[GitHub Actions + Azure](github-actions.md)

À étudier pour comprendre :

```text
Git push
   ↓
GitHub Actions
   ↓
Build
   ↓
Tests
   ↓
Publish
   ↓
Azure
   ↓
Application déployée
```

Cela permet de comprendre les bases d'un pipeline CI/CD pour une application .NET.

---

# 3. Azure et .NET : architecture type

Une architecture simple peut ressembler à :

```text
                         Internet
                            |
                            v
                    Azure App Service
                            |
                            | Managed Identity
                            |
              +-------------+-------------+
              |                           |
              v                           v
         Azure Key Vault              Azure SQL
              |
              |
       Secrets / Certificates
```

Le déploiement peut être automatisé :

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
    +-- dotnet restore
    +-- dotnet build
    +-- dotnet test
    +-- dotnet publish
    |
    v
Azure App Service
```

---

# 4. IaaS, PaaS et SaaS

Une notion importante dans le Cloud est le niveau de responsabilité.

## IaaS

```text
Infrastructure as a Service
```

Azure fournit principalement l'infrastructure.

Exemple conceptuel :

```text
Virtual Machine
```

Tu dois gérer davantage de choses :

```text
OS
Patches
Runtime
Application
Configuration
```

---

## PaaS

```text
Platform as a Service
```

Azure gère une grande partie de l'infrastructure.

Exemple :

```text
Azure App Service
Azure SQL Database
```

Le développeur peut davantage se concentrer sur :

```text
Application
Code
Configuration
Données
```

---

## SaaS

```text
Software as a Service
```

Tu consommes directement une application ou un service logiciel.

Mental model :

```text
IaaS
 ↓
Je gère beaucoup d'infrastructure

PaaS
 ↓
Azure gère davantage de plateforme

SaaS
 ↓
Je consomme principalement le logiciel
```

---

# 5. Région Azure

Azure possède plusieurs régions géographiques.

Une région correspond à une zone géographique contenant des infrastructures Azure.

Lorsqu'on déploie une ressource, on choisit notamment une région.

Exemple :

```text
West Europe
North Europe
France Central
```

Le choix de région peut influencer :

```text
Latence
Disponibilité des services
Résidence des données
Coût
Exigences réglementaires
```

Mental model :

```text
Utilisateur
    |
    | réseau
    v
Région Azure
    |
    v
Ressources
```

Pour une application destinée principalement à la Belgique, une région européenne proche peut être pertinente, mais le choix réel dépend des contraintes du projet.

---

# 6. Resource Group

Un **Resource Group** permet de regrouper des ressources Azure qui appartiennent à une même solution ou à un même cycle de vie.

Exemple :

```text
rg-hotel-api
    |
    +-- App Service
    +-- Azure SQL
    +-- Key Vault
    +-- Application Insights
```

Mental model :

```text
Resource Group
       ↓
Conteneur logique de ressources
```

Ce n'est pas simplement un dossier de fichiers.

Il joue notamment un rôle important pour :

```text
Gestion
Permissions
Déploiement
Organisation
Cycle de vie
```

---

# 7. Subscription

Une Azure Subscription représente un niveau important d'organisation et de facturation.

On peut avoir :

```text
Tenant
  |
  +-- Subscription A
  |      |
  |      +-- Resource Groups
  |
  +-- Subscription B
         |
         +-- Resource Groups
```

Une subscription est notamment associée :

```text
Billing
Quotas
Accès
Ressources
```

---

# 8. Tenant / Microsoft Entra ID

Dans les environnements Azure modernes, l'identité est gérée avec :

```text
Microsoft Entra ID
```

anciennement :

```text
Azure Active Directory
```

Il faut distinguer :

```text
Microsoft Entra ID
        ↓
Identités / utilisateurs / applications / permissions

Azure Resource Manager
        ↓
Gestion des ressources Azure
```

Cette distinction devient importante dès qu'on travaille avec :

```text
RBAC
Managed Identity
Service Principals
Applications
Azure CLI
GitHub Actions
```

---

# 9. RBAC

RBAC signifie :

```text
Role-Based Access Control
```

Le principe :

```text
Qui ?
  ↓
Quel rôle ?
  ↓
Sur quelle ressource ?
  ↓
Quel accès ?
```

Exemple :

```text
Developer
    ↓
Contributor
    ↓
Resource Group
```

Ou :

```text
Application
    ↓
Key Vault Secrets User
    ↓
Key Vault
```

L'objectif est d'appliquer le principe du moindre privilège.

---

# 10. Configuration d'une application .NET sur Azure

Une application ASP.NET Core utilise souvent :

```text
appsettings.json
appsettings.Development.json
Environment Variables
Options Pattern
Secret stores
```

Dans Azure App Service, les paramètres d'application peuvent être utilisés comme variables d'environnement.

Le principe est :

```text
Application
     |
     v
IConfiguration
     |
     +-- appsettings.json
     +-- Environment Variables
     +-- Key Vault
     +-- autres providers
```

Le code .NET peut donc rester relativement indépendant de l'environnement.

---

# 11. Secrets : ne pas les mettre dans Git

Mauvais :

```json
{
  "ConnectionStrings": {
    "Default": "Server=...;Password=SuperSecret123"
  }
}
```

et pousser ce fichier dans un dépôt public.

Même si le secret est supprimé ensuite, il peut avoir été récupéré dans l'historique Git.

Préférer selon le contexte :

```text
Environment Variables
Key Vault
Managed Identity
GitHub Secrets
```

Mental model :

```text
Code
  ↓
ne contient pas le secret

Configuration
  ↓
récupère le secret au runtime
```

---

# 12. CI/CD

CI/CD signifie généralement :

```text
Continuous Integration
Continuous Delivery / Deployment
```

Un pipeline typique .NET :

```text
git push
   ↓
Build
   ↓
Tests
   ↓
Publish
   ↓
Deploy
```

Exemple :

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

# 13. Monitoring

Une application en production doit être observable.

On veut savoir :

```text
L'application fonctionne-t-elle ?
Combien de requêtes ?
Combien d'erreurs ?
Quelle latence ?
Quelle dépendance est lente ?
```

Dans l'écosystème Azure, on rencontre notamment :

```text
Azure Monitor
Application Insights
Log Analytics
```

Le principe général :

```text
Application
    |
    +-- Logs
    +-- Metrics
    +-- Traces
    |
    v
Monitoring
    |
    v
Diagnostic
```

---

# 14. Scalabilité

Il faut distinguer :

## Scale up

Augmenter la puissance d'une instance.

```text
2 CPU
  ↓
4 CPU
```

## Scale out

Ajouter plusieurs instances.

```text
Instance 1
Instance 2
Instance 3
```

Mental model :

```text
Scale Up
= plus puissant

Scale Out
= plus d'instances
```

Azure App Service permet différents mécanismes de mise à l'échelle selon le plan utilisé.

---

# 15. Haute disponibilité

Une application critique ne doit pas forcément dépendre d'une seule instance.

Conceptuellement :

```text
             Load Balancing
                  |
          +-------+-------+
          |               |
          v               v
      Instance 1      Instance 2
```

Si une instance rencontre un problème, une autre peut continuer à traiter les requêtes selon l'architecture et la configuration.

La haute disponibilité dépend cependant de l'ensemble du système :

```text
Compute
Database
Network
Dependencies
Deployment
Monitoring
```

---

# 16. Architecture .NET Azure à retenir

Pour un projet ASP.NET Core moderne :

```text
                    GitHub
                      |
                      | CI/CD
                      v
                GitHub Actions
                      |
                      v
                Azure App Service
                      |
              +-------+-------+
              |               |
              v               v
         Azure SQL        Key Vault
              ^               ^
              |               |
              +--- Managed ---+
                   Identity
```

C'est une architecture de base très utile à comprendre pour un développeur .NET qui travaille avec Azure.

---

# 17. Parcours recommandé

Pour apprendre Azure côté développeur .NET, l'ordre conseillé est :

```text
1. App Service
       ↓
2. Azure SQL
       ↓
3. Configuration
       ↓
4. Key Vault
       ↓
5. Managed Identity
       ↓
6. GitHub Actions
       ↓
7. Monitoring
       ↓
8. Scaling
       ↓
9. Architecture Cloud
```

Dans ce mémo, les cinq premiers fichiers se concentrent sur :

```text
App Service
Azure SQL
Key Vault
Managed Identity
GitHub Actions
```

---

# Questions d'entretien

### Qu'est-ce qu'Azure App Service ?

Un service PaaS permettant notamment d'héberger des applications web et APIs sans gérer directement l'infrastructure sous-jacente comme avec une machine virtuelle.

### Quelle différence entre IaaS et PaaS ?

```text
IaaS = davantage d'infrastructure à gérer
PaaS = Azure gère davantage de plateforme
```

### À quoi sert un Resource Group ?

À regrouper logiquement des ressources Azure qui appartiennent à une solution ou à un même cycle de vie.

### Qu'est-ce que RBAC ?

Role-Based Access Control : un mécanisme permettant d'attribuer des permissions à des identités via des rôles sur des ressources.

### Pourquoi utiliser Key Vault ?

Pour centraliser et protéger des secrets, clés et certificats au lieu de les placer directement dans le code ou le dépôt Git.

### Pourquoi Managed Identity ?

Pour permettre à une ressource Azure de s'authentifier auprès d'autres services Azure sans devoir stocker explicitement un secret d'authentification dans l'application.

### Quelle différence entre Scale Up et Scale Out ?

```text
Scale Up   = augmenter la puissance
Scale Out  = augmenter le nombre d'instances
```

### Qu'est-ce que CI/CD ?

Un ensemble de pratiques permettant notamment d'automatiser l'intégration, les tests, la construction et le déploiement d'une application.

---

# À retenir

Pour raisonner sur Azure, ne mémorise pas seulement les noms des services.

Pars du problème :

```text
Héberger
   → App Service

Données SQL
   → Azure SQL

Secrets
   → Key Vault

Authentification entre services Azure
   → Managed Identity

Déploiement automatique
   → GitHub Actions

Monitoring
   → Azure Monitor / Application Insights
```

## Phrase à mémoriser

> **Azure n'est pas un service unique : c'est un ensemble de services Cloud que l'on assemble selon les besoins de l'application.**
