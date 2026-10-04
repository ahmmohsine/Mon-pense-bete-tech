# Azure Managed Identity

## 1. Qu'est-ce qu'une Managed Identity ?

Une **Managed Identity** permet à une ressource Azure de disposer d'une identité gérée par Microsoft Entra ID.

L'idée fondamentale est :

> Une application Azure peut s'authentifier auprès d'autres services Azure sans devoir stocker elle-même un secret d'authentification.

Exemple :

```text
Azure App Service
       |
       | Managed Identity
       v
Microsoft Entra ID
       |
       v
Azure Key Vault
```

Sans Managed Identity, on pourrait être tenté de faire :

```text
App Service
    |
    | Client ID + Client Secret
    v
Azure Service
```

Le problème est alors évident :

```text
Où stocker le Client Secret ?
```

La Managed Identity permet de supprimer cette dépendance dans de nombreux scénarios Azure.

---

# 2. Le problème que Managed Identity résout

Imaginons :

```text
ASP.NET Core API
       |
       v
Azure Key Vault
```

L'API doit s'authentifier.

Une mauvaise solution serait :

```csharp
var clientSecret = "SuperSecret...";
```

Puis :

```text
API
 ↓
Client Secret
 ↓
Azure
```

Le secret doit alors être stocké quelque part.

Cela crée un nouveau problème de sécurité :

```text
Secret d'authentification
       ↓
Stockage
       ↓
Rotation
       ↓
Risque d'exposition
```

Avec Managed Identity :

```text
App Service
       |
       | identité gérée par Azure
       v
Microsoft Entra ID
```

L'identité n'a pas besoin d'être stockée comme un mot de passe dans le code.

---

# 3. Le modèle mental à retenir

Il faut séparer trois notions :

```text
Managed Identity
       ↓
QUI SUIS-JE ?

RBAC / Permissions
       ↓
AI-JE LE DROIT ?

Service cible
       ↓
À QUOI AI-JE ACCÈS ?
```

Exemple :

```text
App Service
    |
    | "Je suis l'identité X"
    v
Microsoft Entra ID
    |
    | "Cette identité a le rôle Y"
    v
Key Vault
    |
    v
Secret
```

Donc :

```text
Identity ≠ Permission
```

C'est probablement la notion la plus importante de ce fichier.

---

# 4. Microsoft Entra ID

Les Managed Identities s'appuient sur :

```text
Microsoft Entra ID
```

Microsoft Entra ID gère les identités utilisées dans l'écosystème Azure.

Mental model :

```text
Application
    |
    v
Managed Identity
    |
    v
Microsoft Entra ID
```

L'identité peut ensuite être utilisée pour accéder à des ressources Azure auxquelles elle a été autorisée.

---

# 5. Deux types de Managed Identity

Il existe principalement :

```text
System-assigned managed identity
User-assigned managed identity
```

Il faut bien comprendre leur différence.

---

# 6. System-assigned Managed Identity

Une identité **system-assigned** est liée directement à une ressource Azure.

Exemple :

```text
App Service
    |
    +-- System-assigned Identity
```

L'identité possède le même cycle de vie que la ressource.

Mental model :

```text
Créer App Service
        ↓
Créer son identité

Supprimer App Service
        ↓
Identité supprimée avec la ressource
```

C'est pratique lorsqu'une identité appartient exclusivement à une ressource.

---

# 7. User-assigned Managed Identity

Une identité **user-assigned** est une ressource Azure indépendante.

Exemple :

```text
User-assigned Identity
        |
        +------ App Service A
        |
        +------ App Service B
```

Plusieurs ressources peuvent utiliser la même identité selon les permissions et l'architecture.

Mental model :

```text
Identity
    |
    +---- Resource A
    |
    +---- Resource B
```

Elle possède donc un cycle de vie indépendant des ressources qui l'utilisent.

---

# 8. System-assigned vs User-assigned

| System-assigned | User-assigned |
|---|---|
| Liée à une ressource | Ressource indépendante |
| Cycle de vie lié | Cycle de vie indépendant |
| Simple pour un service unique | Pratique lorsqu'une identité doit être réutilisée |
| Supprimée avec la ressource | Peut continuer à exister |

Mental model :

```text
System-assigned
    ↓
"Cette identité appartient à cette ressource."

User-assigned
    ↓
"Cette identité existe indépendamment et peut être associée à plusieurs ressources."
```

---

# 9. Exemple App Service + Key Vault

Architecture :

```text
                  Azure
                    |
          +---------+---------+
          |                   |
          v                   v
     App Service          Key Vault
          |
          |
    Managed Identity
          |
          v
    Microsoft Entra ID
          |
          v
      Permission
          |
          v
       Secret
```

Étapes conceptuelles :

```text
1. App Service possède une Managed Identity
2. Azure connaît cette identité
3. On attribue un rôle à cette identité
4. Le rôle donne les permissions nécessaires
5. L'application demande le secret
6. Key Vault vérifie l'identité et les permissions
7. Le secret est retourné si l'accès est autorisé
```

---

# 10. Managed Identity ne signifie pas « accès total »

C'est une erreur très importante.

Activer :

```text
Managed Identity
```

ne signifie pas :

```text
Accès à toutes les ressources Azure
```

Cela signifie principalement :

```text
Cette ressource possède une identité Azure
```

Il faut ensuite configurer :

```text
Permissions
```

Exemple :

```text
App Service
    ↓
Managed Identity
    ↓
Key Vault
    ↓
RBAC
    ↓
Secret access
```

---

# 11. RBAC

RBAC signifie :

```text
Role-Based Access Control
```

Le principe :

```text
Identity
    ↓
Role
    ↓
Resource
```

Exemple conceptuel :

```text
App Service Managed Identity
        |
        | Key Vault Secrets User
        v
Azure Key Vault
```

Le rôle détermine ce que l'identité peut faire.

---

# 12. Principe du moindre privilège

Il faut éviter :

```text
Managed Identity
      ↓
Permissions maximales
      ↓
Tous les services
```

Préférer :

```text
Managed Identity
      ↓
Permissions minimales
      ↓
Ressources nécessaires
```

Exemple :

```text
Application
    ↓
besoin uniquement de lire des secrets
    ↓
permission de lecture
```

Elle n'a pas nécessairement besoin de pouvoir :

```text
Supprimer les secrets
Créer des secrets
Administrer tout le Key Vault
```

---

# 13. Managed Identity et Key Vault

C'est l'un des scénarios les plus fréquents.

```text
ASP.NET Core
      |
      | DefaultAzureCredential
      v
Azure Identity
      |
      v
Managed Identity
      |
      v
Key Vault
```

Une application .NET peut utiliser le package :

```text
Azure.Identity
```

et notamment :

```csharp
DefaultAzureCredential
```

L'intérêt de `DefaultAzureCredential` est de permettre à l'application d'utiliser différentes méthodes d'authentification selon l'environnement.

Par exemple :

```text
Développement local
    ↓
Developer credential

Azure
    ↓
Managed Identity
```

Le même code peut donc être utilisé dans plusieurs environnements.

---

# 14. Exemple .NET

Un exemple conceptuel avec le SDK Azure :

```csharp
using Azure.Identity;
using Azure.Security.KeyVault.Secrets;

var credential = new DefaultAzureCredential();

var client = new SecretClient(
    new Uri("https://my-key-vault.vault.azure.net/"),
    credential);

KeyVaultSecret secret =
    await client.GetSecretAsync("MySecret");
```

Le point important est :

```csharp
new DefaultAzureCredential()
```

Le code ne contient pas directement :

```text
Client Secret
Password
```

Le mécanisme d'authentification dépend de l'environnement.

---

# 15. `DefaultAzureCredential`

`DefaultAzureCredential` est très pratique pour les applications .NET utilisant les services Azure.

Mental model :

```text
DefaultAzureCredential
          |
          +-- développement local
          |
          +-- environnement Azure
          |
          +-- Managed Identity
```

L'application utilise une chaîne de credentials adaptée à l'environnement.

Cela évite souvent d'écrire :

```csharp
if (environment == "Development")
{
    ...
}
else
{
    ...
}
```

uniquement pour changer le mécanisme d'authentification Azure.

---

# 16. Développement local vs Azure

Supposons :

```text
Développement
```

Tu exécutes l'application depuis Visual Studio ou ta machine.

Il n'y a pas nécessairement de Managed Identity locale.

Avec :

```csharp
DefaultAzureCredential
```

le SDK peut utiliser une identité de développeur disponible localement.

Puis en Azure :

```text
App Service
    ↓
Managed Identity
```

Le même code peut alors fonctionner avec l'identité Azure.

Mental model :

```text
LOCAL
Developer Credential
      ↓
Azure

PRODUCTION
Managed Identity
      ↓
Azure
```

---

# 17. Pourquoi c'est pratique ?

Sans cette approche :

```text
if Development
    utiliser credential A

if Production
    utiliser credential B
```

Avec une abstraction comme :

```csharp
DefaultAzureCredential
```

on peut davantage séparer :

```text
Application
```

de :

```text
Mécanisme concret d'authentification
```

Le code reste donc plus portable entre les environnements.

---

# 18. Managed Identity avec Azure SQL

Managed Identity ne sert pas uniquement à Key Vault.

Elle peut également être utilisée dans des scénarios avec Azure SQL et Microsoft Entra authentication.

Architecture :

```text
App Service
      |
      | Managed Identity
      v
Microsoft Entra ID
      |
      v
Azure SQL
```

L'idée est similaire :

```text
Application
    ↓
Identity
    ↓
Permission
    ↓
Database
```

Cela permet de réduire l'utilisation de mots de passe SQL dans certaines architectures.

---

# 19. Managed Identity avec Storage

Autre scénario :

```text
App Service
      |
      | Managed Identity
      v
Azure Storage
```

Par exemple, une application peut avoir besoin de lire ou d'écrire des blobs.

On attribue alors à l'identité le rôle nécessaire.

Mental model :

```text
Managed Identity
       |
       v
Role
       |
       v
Storage Account
       |
       v
Blob
```

---

# 20. Managed Identity avec d'autres services

Le principe peut être appliqué à différents services Azure compatibles avec Microsoft Entra ID.

Exemple conceptuel :

```text
             Managed Identity
                    |
       +------------+------------+
       |            |            |
       v            v            v
  Key Vault      Azure SQL     Storage
```

La question à se poser est toujours :

```text
1. Quelle identité ?
2. Quelle ressource ?
3. Quel rôle ?
4. Quelle permission ?
```

---

# 21. Managed Identity et secret

Le but n'est pas :

```text
Managed Identity = un nouveau secret
```

Au contraire.

L'idée est :

```text
Pas de secret applicatif
        ↓
Identité gérée par Azure
        ↓
Authentification auprès du service
```

C'est particulièrement utile pour les communications :

```text
Application → Azure Service
```

---

# 22. Différence avec un Service Principal

Un **Service Principal** représente une identité d'application dans Microsoft Entra ID.

Il peut être utilisé pour des automatisations et applications.

Mais historiquement, une application peut être configurée avec des credentials comme :

```text
Client ID
Client Secret
```

Une Managed Identity est une forme d'identité gérée par Azure pour les ressources Azure compatibles.

Mental model simplifié :

```text
Service Principal
    ↓
identité d'application pouvant utiliser des credentials

Managed Identity
    ↓
identité d'application gérée par Azure
```

La Managed Identity est particulièrement intéressante lorsqu'une ressource Azure doit appeler un autre service Azure.

---

# 23. Managed Identity et GitHub Actions

Il faut distinguer :

```text
Application runtime
```

et :

```text
CI/CD pipeline
```

Une application dans App Service peut utiliser :

```text
Managed Identity
```

pour accéder à Azure.

Un workflow GitHub Actions n'est pas simplement « l'App Service ».

Pour GitHub Actions, on peut notamment utiliser une fédération d'identité basée sur OIDC avec Azure afin d'éviter les secrets persistants.

Mental model :

```text
Application
    ↓
Managed Identity

GitHub Actions
    ↓
OIDC / federated identity
```

Les deux répondent à des problèmes proches mais dans des contextes différents.

---

# 24. Sécurité en profondeur

Une Managed Identity ne remplace pas toutes les autres protections.

Il faut toujours penser :

```text
Identity
+
RBAC
+
Network
+
Encryption
+
Monitoring
+
Least Privilege
```

Exemple :

```text
App Service
   |
   | Managed Identity
   v
Key Vault
   |
   | RBAC
   v
Secret
```

Mais il faut également protéger :

```text
App Service
Network
Application
Dependencies
Logs
```

---

# 25. Erreurs fréquentes

## Erreur 1 : penser que Managed Identity = accès automatique

Faux.

```text
Identity
+
Permission
```

sont nécessaires.

---

## Erreur 2 : donner trop de droits

Éviter :

```text
Contributor
Owner
Full access
```

si l'application n'en a pas besoin.

---

## Erreur 3 : utiliser une Managed Identity partout sans réfléchir

Le mécanisme est puissant, mais il faut choisir l'architecture adaptée.

Pour une application locale :

```text
Managed Identity Azure
```

n'est pas nécessairement disponible.

`DefaultAzureCredential` permet justement de gérer les différences entre environnements.

---

## Erreur 4 : mettre quand même un client secret dans le code

Exemple à éviter :

```csharp
var credential =
    new ClientSecretCredential(
        tenantId,
        clientId,
        "SUPER_SECRET");
```

Le secret ne doit pas être hardcodé.

---

## Erreur 5 : oublier les permissions SQL / Key Vault / Storage

Avoir :

```text
Managed Identity = activée
```

ne signifie pas :

```text
Authorization = configurée
```

---

# 26. Checklist Managed Identity

```text
[ ] Managed Identity choisie selon le besoin
[ ] System-assigned ou User-assigned compris
[ ] Microsoft Entra ID compris
[ ] RBAC configuré
[ ] Principe du moindre privilège appliqué
[ ] Aucun secret hardcodé
[ ] Key Vault utilisé lorsque nécessaire
[ ] DefaultAzureCredential compris
[ ] Différence local / Azure comprise
[ ] Permissions de la ressource cible vérifiées
[ ] Logs ne contenant pas de secrets
[ ] CI/CD distingué du runtime de l'application
```

---

# 27. Questions d'entretien

### Qu'est-ce qu'une Managed Identity ?

Une identité gérée par Azure permettant à une ressource Azure de s'authentifier auprès de services compatibles sans devoir stocker elle-même un secret d'authentification.

### Quels sont les deux types principaux ?

```text
System-assigned
User-assigned
```

### Quelle différence ?

```text
System-assigned
→ cycle de vie lié à la ressource

User-assigned
→ identité indépendante pouvant être associée à plusieurs ressources
```

### Managed Identity donne-t-elle automatiquement des permissions ?

Non.

L'identité doit recevoir les permissions nécessaires sur la ressource cible.

### Quel est le rôle de RBAC ?

Déterminer les permissions d'une identité sur une ressource via des rôles.

### À quoi sert `DefaultAzureCredential` ?

À fournir une approche unifiée pour obtenir des credentials Azure selon l'environnement d'exécution, notamment pour faciliter le développement local et l'utilisation de Managed Identity dans Azure.

### Pourquoi utiliser Managed Identity avec Key Vault ?

Pour permettre à l'application d'accéder aux secrets sans stocker elle-même un secret d'authentification destiné à Key Vault.

### Peut-on utiliser Managed Identity avec Azure SQL ?

Oui, dans les scénarios où Azure SQL et l'application utilisent l'authentification Microsoft Entra appropriée.

### Managed Identity et GitHub Actions sont-ils la même chose ?

Non.

```text
Managed Identity
→ identité du runtime Azure

GitHub Actions + OIDC
→ identité utilisée par le pipeline CI/CD
```

---

# 28. Schéma mental final

```text
                         APPLICATION
                              |
                              v
                      Managed Identity
                              |
                              v
                       Microsoft Entra ID
                              |
                              v
                         Authorization
                              |
                       +------+------+
                       |             |
                     Refusé        Autorisé
                       |             |
                       v             v
                    Access        Service
                    denied        Azure
                                     |
                    +----------------+----------------+
                    |                |                |
                    v                v                v
                Key Vault        Azure SQL         Storage
```

La logique fondamentale est :

```text
WHO ?
  ↓
Managed Identity

CAN DO WHAT ?
  ↓
RBAC / Permissions

ACCESS WHAT ?
  ↓
Azure Resource
```

---

# À retenir

Managed Identity permet de résoudre un problème très concret :

```text
"Comment mon application Azure peut-elle
s'authentifier auprès d'un autre service Azure
sans stocker un secret d'authentification ?"
```

La réponse est souvent :

```text
Managed Identity
       ↓
Microsoft Entra ID
       ↓
RBAC
       ↓
Azure Resource
```

## Phrase à mémoriser

> **Managed Identity fournit l'identité de mon application ; elle ne lui donne pas automatiquement les permissions : c'est l'autorisation sur la ressource qui détermine ce qu'elle peut réellement faire.**
