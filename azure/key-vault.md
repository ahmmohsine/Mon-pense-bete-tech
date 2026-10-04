# Azure Key Vault

## 1. Qu'est-ce qu'Azure Key Vault ?

Azure Key Vault est un service Azure destiné à stocker et gérer des informations sensibles.

On peut notamment y gérer :

```text
Secrets
Keys
Certificates
```

Pour une application .NET, un cas très courant est le stockage de secrets :

```text
Database credentials
API keys
Connection information
Certificates
Tokens ou informations sensibles selon l'architecture
```

Mental model :

```text
Application
    |
    | "J'ai besoin d'un secret"
    v
Azure Key Vault
    |
    v
Secret
```

L'objectif principal est d'éviter de mettre les secrets directement dans :

```text
Code source
appsettings.json versionné
GitHub repository
```

---

# 2. Pourquoi ne pas mettre les secrets dans le code ?

Mauvais exemple :

```csharp
var apiKey = "123456-secret";
```

Ou :

```json
{
  "ConnectionStrings": {
    "Default": "Server=...;Password=MySecret123"
  }
}
```

Si le fichier est commité :

```text
git add
    ↓
git commit
    ↓
git push
    ↓
secret exposé
```

Même supprimer le secret du dernier commit ne garantit pas qu'il n'existe plus dans l'historique.

Mental model :

```text
Code
    ↓
public / partagé / versionné

Secret
    ↓
doit être géré séparément
```

---

# 3. Les trois grandes catégories

Azure Key Vault peut gérer notamment :

```text
Secrets
Keys
Certificates
```

Il faut comprendre leur différence.

## Secret

Une valeur sensible.

Exemple :

```text
DatabasePassword
ApiKey
ClientSecret
```

## Key

Une clé cryptographique utilisée pour certaines opérations cryptographiques.

## Certificate

Un certificat numérique associé à une clé et à des informations de certificat.

Pour un développeur .NET débutant avec Azure, le concept le plus important à retenir au départ est :

```text
Key Vault
    ↓
Secrets
```

---

# 4. Secret vs configuration

Il faut distinguer :

```text
Configuration
```

et :

```text
Secret
```

Exemple :

```json
{
  "ApplicationName": "HotelListing",
  "LogLevel": "Information"
}
```

Ce ne sont pas nécessairement des secrets.

En revanche :

```text
DatabasePassword
ApiKey
ClientSecret
```

sont sensibles.

Mental model :

```text
Configuration
   ↓
informations nécessaires à l'application

Secret
   ↓
configuration sensible
```

---

# 5. Architecture classique avec App Service

Une architecture simple :

```text
                    Azure
                     |
          +----------+----------+
          |                     |
          v                     v
    App Service             Key Vault
          |                     |
          |                     |
          +---------------------+
```

Mais il faut répondre à une question essentielle :

> Comment App Service prouve-t-il qu'il a le droit de lire Key Vault ?

C'est là que **Managed Identity** intervient.

---

# 6. Managed Identity + Key Vault

Architecture :

```text
Azure App Service
       |
       | Managed Identity
       v
Microsoft Entra ID
       |
       | identité
       v
Azure Key Vault
       |
       v
Secret
```

L'application ne stocke donc pas nécessairement un mot de passe pour accéder à Key Vault.

On délègue l'identité à Azure.

Mental model :

```text
Qui es-tu ?
    ↓
Managed Identity

As-tu le droit ?
    ↓
RBAC / permissions

Quel secret ?
    ↓
Key Vault
```

Cette distinction est très importante :

```text
Identity
≠
Permission
```

---

# 7. Managed Identity ne donne pas automatiquement accès

Une erreur fréquente :

```text
J'active Managed Identity
        ↓
Key Vault fonctionne automatiquement
```

Non.

L'identité existe, mais il faut également lui donner les permissions nécessaires.

Conceptuellement :

```text
App Service
    |
    | Managed Identity
    v
"Voici mon identité"
    |
    v
Key Vault
    |
    | Autorisation ?
    +---- Non → Access denied
    |
    +---- Oui → Secret
```

---

# 8. RBAC

Azure permet notamment de gérer les accès avec :

```text
Role-Based Access Control
```

Exemple conceptuel :

```text
Managed Identity
       |
       | Role
       v
Key Vault Secrets User
       |
       v
Key Vault
```

Le principe est :

```text
Identité
    ↓
Rôle
    ↓
Ressource
    ↓
Permission
```

Il faut choisir le rôle correspondant réellement au besoin.

---

# 9. Principe du moindre privilège

Supposons qu'une application ait uniquement besoin de lire des secrets.

Il n'est pas nécessaire de lui donner des permissions permettant de tout administrer.

Mauvais principe :

```text
Application
    ↓
Full Access
    ↓
Key Vault
```

Meilleur principe :

```text
Application
    ↓
Permission minimale
    ↓
Secrets nécessaires
```

Mental model :

> Une application doit avoir uniquement les permissions dont elle a réellement besoin.

---

# 10. Exemple de secret

Imaginons un secret :

```text
HotelDbConnectionString
```

avec comme valeur une information sensible.

Dans Key Vault :

```text
Secret name:
HotelDbConnectionString

Secret value:
<valeur sensible>
```

Le code ne doit pas nécessairement contenir :

```text
Server=...
Password=...
```

L'application peut récupérer cette configuration depuis Key Vault selon le mécanisme choisi.

---

# 11. Intégration avec ASP.NET Core

ASP.NET Core possède un système de configuration extensible.

Mental model :

```text
IConfiguration
      |
      +-- appsettings.json
      +-- Environment Variables
      +-- User Secrets
      +-- Azure Key Vault
      +-- autres providers
```

L'intérêt est que le code peut continuer à utiliser :

```csharp
builder.Configuration["MySetting"];
```

ou :

```csharp
builder.Configuration
    .GetConnectionString("Default");
```

sans nécessairement connaître la source exacte de la valeur.

C'est le principe des **configuration providers**.

---

# 12. Configuration Provider

Un provider est une source de configuration.

Exemples :

```text
JSON
Environment Variables
Azure Key Vault
User Secrets
Command Line
```

Mental model :

```text
                 IConfiguration
                       |
        +--------------+--------------+
        |              |              |
       JSON       Environment      Key Vault
                    Variables
```

L'application demande :

```csharp
configuration["MySetting"]
```

et le système de configuration cherche la valeur selon les providers configurés et leur ordre.

---

# 13. Key Vault comme source de configuration

Une application ASP.NET Core peut être configurée pour utiliser Key Vault comme source de configuration.

Conceptuellement :

```text
Key Vault
    |
    | Configuration Provider
    v
IConfiguration
    |
    v
Application
```

Cela permet de séparer :

```text
Code
```

de :

```text
Secrets
```

---

# 14. Secret names et configuration

Supposons une configuration :

```json
{
  "ConnectionStrings": {
    "Default": "..."
  }
}
```

Selon le provider utilisé, les noms de secrets peuvent être adaptés pour représenter une hiérarchie de configuration.

Le principe général est :

```text
Configuration hiérarchique
        ↓
provider
        ↓
clé de configuration
```

Il faut toutefois respecter les conventions du provider utilisé plutôt que supposer qu'un nom de secret sera automatiquement interprété de la même manière partout.

---

# 15. Azure Key Vault et App Service

Dans une architecture App Service :

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

Une configuration courante consiste à laisser Azure/App Service intégrer les secrets de Key Vault à la configuration de l'application.

L'avantage :

```text
Application
    ↓
ne contient pas le secret en dur
```

---

# 16. Secret rotation

Un secret peut devoir être changé.

Exemple :

```text
API Key A
    ↓
compromise / expiration
    ↓
API Key B
```

Une bonne architecture doit pouvoir changer les secrets sans modifier le code source.

Mental model :

```text
Code
   ↓
référence logique

Secret
   ↓
valeur changeable
```

Cela facilite notamment :

```text
Rotation
Expiration
Révocation
Gestion d'incidents
```

---

# 17. Pourquoi la rotation est importante ?

Imaginons :

```text
DatabasePassword = ABC123
```

et que cette valeur soit compromise.

Si elle est directement dans le code :

```text
Code
 ↓
secret
 ↓
build
 ↓
déploiement
```

le changement peut devenir pénible.

Avec une gestion externe :

```text
Key Vault
 ↓
nouveau secret
 ↓
application récupère la nouvelle valeur
```

La séparation permet une gestion plus propre.

---

# 18. Key Vault n'est pas un fichier de configuration

Ne pas penser :

```text
Key Vault = appsettings.json dans Azure
```

Key Vault est un service spécialisé dans la gestion de secrets, clés et certificats.

Le fichier :

```text
appsettings.json
```

est une source de configuration.

Key Vault :

```text
Security / Secret Management
```

Mental model :

```text
appsettings
    ↓
configuration

Key Vault
    ↓
secrets sensibles
```

---

# 19. Key Vault et environnement local

En développement local, on peut utiliser :

```text
User Secrets
```

ou d'autres mécanismes adaptés.

Exemple conceptuel :

```text
Development
    ↓
User Secrets

Production
    ↓
Key Vault
```

L'idée est de ne pas obliger le développeur à commiter des secrets simplement parce que l'application en a besoin.

---

# 20. Key Vault et GitHub Actions

Key Vault peut également entrer dans une architecture CI/CD.

Exemple conceptuel :

```text
GitHub
   |
   v
GitHub Actions
   |
   v
Azure
   |
   v
Key Vault
```

Selon l'architecture, GitHub Actions peut utiliser une identité fédérée / OIDC pour s'authentifier auprès d'Azure sans stocker un long-lived client secret.

Le principe général reste :

```text
Pipeline
   ↓
Identity
   ↓
Permissions
   ↓
Azure resource
```

---

# 21. Key Vault et certificats

Key Vault peut également gérer des certificats.

Exemple :

```text
TLS Certificate
      |
      v
Key Vault
      |
      v
Azure service
```

Cela peut simplifier la gestion du cycle de vie des certificats.

Le certificat n'est donc pas nécessairement stocké directement dans le repository de l'application.

---

# 22. Key Vault et cryptographie

Les **keys** de Key Vault peuvent être utilisées pour certaines opérations cryptographiques.

Mental model :

```text
Application
    |
    | demande une opération
    v
Key Vault
    |
    | utilise la clé
    v
Résultat
```

Cela peut permettre de garder certaines clés cryptographiques sous contrôle du service plutôt que de les distribuer directement dans l'application.

---

# 23. Key Vault et sécurité en profondeur

Key Vault n'est qu'une partie de la sécurité.

Une architecture complète peut être :

```text
HTTPS
  +
Identity
  +
RBAC
  +
Key Vault
  +
Managed Identity
  +
Network controls
  +
Monitoring
```

Il faut donc éviter :

```text
"J'utilise Key Vault donc mon application est sécurisée."
```

Key Vault protège une catégorie importante de risques, mais pas tous les risques de l'application.

---

# 24. Erreurs fréquentes

## Erreur 1 : mettre le secret dans Git avant de le déplacer vers Key Vault

Si un secret a déjà été exposé :

```text
déplacer le secret
```

ne suffit pas forcément.

Il faut envisager :

```text
Révoquer
Rotationner
Remplacer
```

selon le type de secret.

---

## Erreur 2 : donner trop de permissions

```text
Managed Identity
    ↓
Full Key Vault access
```

alors que l'application a seulement besoin de lire quelques secrets.

Préférer le moindre privilège.

---

## Erreur 3 : oublier Managed Identity

Créer un Key Vault ne permet pas automatiquement à l'application d'y accéder.

Il faut définir :

```text
Identity
+
Permission
```

---

## Erreur 4 : confondre identité et autorisation

```text
Managed Identity
    ↓
Qui es-tu ?

RBAC
    ↓
As-tu le droit ?
```

---

## Erreur 5 : mettre des secrets dans les logs

Mauvais :

```csharp
logger.LogInformation(
    "Connection string = {ConnectionString}",
    connectionString);
```

Les logs doivent eux aussi être considérés comme des données potentiellement sensibles.

---

# 25. Checklist Key Vault

```text
[ ] Secrets séparés du code source
[ ] Key Vault utilisé pour les informations sensibles
[ ] Managed Identity activée lorsque pertinente
[ ] Permissions RBAC correctement configurées
[ ] Principe du moindre privilège appliqué
[ ] Secrets non présents dans Git
[ ] Secrets non présents dans les logs
[ ] Rotation des secrets prévue
[ ] Environnements séparés
[ ] HTTPS utilisé
[ ] Monitoring et audit pris en compte
```

---

# 26. Questions d'entretien

### À quoi sert Azure Key Vault ?

À stocker et gérer de manière sécurisée des secrets, clés et certificats.

### Pourquoi ne pas mettre un mot de passe dans `appsettings.json` ?

Parce que le fichier peut être versionné, partagé ou exposé. Les secrets doivent être gérés séparément du code source.

### Quelle différence entre Key Vault et Managed Identity ?

```text
Key Vault
→ stocke / gère le secret

Managed Identity
→ fournit une identité permettant à l'application de s'authentifier auprès d'Azure
```

### Managed Identity donne-t-elle automatiquement accès à Key Vault ?

Non. L'identité doit recevoir les permissions nécessaires.

### Qu'est-ce que RBAC ?

Un mécanisme permettant d'attribuer des permissions via des rôles à des identités sur des ressources.

### Pourquoi utiliser le principe du moindre privilège ?

Pour limiter les actions possibles et réduire l'impact d'une compromission.

### Pourquoi la rotation des secrets est-elle importante ?

Pour pouvoir remplacer régulièrement ou rapidement une valeur sensible compromise ou arrivée à expiration sans modifier le code source.

### Key Vault remplace-t-il toute la configuration de l'application ?

Non. Il est principalement destiné à la gestion de secrets, clés et certificats. ASP.NET Core peut combiner plusieurs sources de configuration.

---

# 27. Schéma mental final

```text
                    APPLICATION
                         |
                         | "J'ai besoin d'un secret"
                         v
                  Managed Identity
                         |
                         | "Qui suis-je ?"
                         v
                  Microsoft Entra ID
                         |
                         | identité validée
                         v
                     Key Vault
                         |
                         | RBAC
                         |
                         v
                  "Ai-je le droit ?"
                         |
                    +----+----+
                    |         |
                   NON       OUI
                    |         |
                  Denied       v
                            Secret
                              |
                              v
                         Application
```

Le point essentiel :

```text
Managed Identity
       ↓
IDENTITÉ

RBAC
       ↓
AUTORISATION

Key Vault
       ↓
SECRET
```

---

# À retenir

Azure Key Vault sert à sortir les informations sensibles du code et de la configuration versionnée.

L'architecture la plus importante à mémoriser est :

```text
App Service
    ↓
Managed Identity
    ↓
Azure Key Vault
    ↓
Secret
```

Et surtout :

```text
Identity ≠ Permission

Managed Identity
    → qui suis-je ?

RBAC
    → ai-je le droit ?

Key Vault
    → quelle donnée sensible dois-je récupérer ?
```

## Phrase à mémoriser

> **Key Vault protège mes secrets ; Managed Identity permet à mon application de s'identifier auprès d'Azure ; RBAC décide si cette identité a le droit d'y accéder.**
