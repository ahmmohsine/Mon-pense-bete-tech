# Azure — GitHub Actions

GitHub Actions permet d'automatiser le cycle **build → test → package → deploy** d'une application .NET.

Pour une API ASP.NET Core déployée sur Azure App Service, l'objectif classique est :

```text
Git push
   ↓
GitHub Actions
   ↓
Restore
   ↓
Build
   ↓
Tests
   ↓
Publish
   ↓
Déploiement
   ↓
Azure App Service
```

---

## 1. CI et CD : quelle différence ?

### CI — Continuous Integration

La CI vérifie automatiquement que le code peut être intégré.

Exemple :

```text
push sur main
    ↓
dotnet restore
    ↓
dotnet build
    ↓
dotnet test
```

Le but est de détecter rapidement :

- erreurs de compilation ;
- tests qui échouent ;
- dépendances incorrectes ;
- régressions.

### CD — Continuous Delivery / Deployment

Le CD va plus loin :

```text
Build + Tests
      ↓
Publication
      ↓
Déploiement Azure
```

Le code validé est envoyé automatiquement vers un environnement.

### À retenir

> CI = vérifier le code.
>
> CD = livrer ou déployer le code.

---

# 2. Qu'est-ce qu'un workflow GitHub Actions ?

Un workflow est un fichier YAML placé généralement dans :

```text
.github/workflows/
```

Par exemple :

```text
.github/
└── workflows/
    └── deploy.yml
```

GitHub détecte automatiquement les fichiers YAML présents dans ce dossier.

Un workflow décrit :

- quand il doit s'exécuter ;
- sur quel environnement ;
- quelles tâches effectuer ;
- dans quel ordre ;
- avec quelles permissions ;
- avec quels secrets.

---

# 3. Structure générale d'un workflow

Exemple minimal :

```yaml
name: Build and Test

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --no-restore

      - name: Test
        run: dotnet test --no-build
```

Voyons ce qui se passe réellement.

---

# 4. `name`

```yaml
name: Build and Test
```

C'est simplement le nom affiché dans GitHub Actions.

Il n'influence pas l'exécution.

---

# 5. `on`

```yaml
on:
  push:
    branches:
      - main
```

Cela signifie :

> Exécute le workflow lorsqu'un `push` est effectué sur `main`.

On peut aussi avoir :

```yaml
on:
  pull_request:
    branches:
      - main
```

Le workflow s'exécute alors lorsqu'une Pull Request cible `main`.

On peut également permettre un lancement manuel :

```yaml
on:
  workflow_dispatch:
```

Très utile pour relancer un déploiement manuellement.

---

# 6. `jobs`

```yaml
jobs:
  build:
```

Un workflow contient un ou plusieurs jobs.

Exemple :

```text
Workflow
│
├── build
│
├── test
│
└── deploy
```

Chaque job s'exécute dans un environnement appelé **runner**.

---

# 7. `runs-on`

```yaml
runs-on: ubuntu-latest
```

Cela indique le système utilisé pour exécuter le job.

Par exemple :

```yaml
ubuntu-latest
```

ou :

```yaml
windows-latest
```

Pour beaucoup de projets .NET, Ubuntu convient parfaitement.

---

# 8. Les `steps`

Un job contient plusieurs étapes :

```yaml
steps:
  - name: Checkout
    uses: actions/checkout@v4

  - name: Restore
    run: dotnet restore
```

Deux concepts importants existent ici.

## `uses`

```yaml
uses: actions/checkout@v4
```

On utilise une **Action GitHub existante**.

## `run`

```yaml
run: dotnet restore
```

On exécute directement une commande dans le runner.

### Mental model

```text
uses = utiliser une action existante

run = exécuter une commande
```

---

# 9. Checkout

```yaml
- name: Checkout
  uses: actions/checkout@v4
```

Le runner démarre avec un environnement propre.

Il faut donc récupérer le contenu du repository.

Conceptuellement :

```text
GitHub Repository
        ↓
actions/checkout
        ↓
Code disponible sur le runner
```

Sans cette étape, le runner ne possède pas ton projet.

---

# 10. Installer .NET

```yaml
- name: Setup .NET
  uses: actions/setup-dotnet@v4
  with:
    dotnet-version: '10.0.x'
```

Cette étape installe/configure le SDK .NET demandé.

Le `x` signifie que l'on accepte la dernière version disponible correspondant à cette ligne majeure/mineure.

Exemple :

```yaml
dotnet-version: '10.0.x'
```

---

# 11. Restore

```yaml
- name: Restore
  run: dotnet restore
```

`dotnet restore` récupère les dépendances NuGet nécessaires au projet.

Par exemple :

```text
Ton projet
   ↓
PackageReference
   ↓
NuGet
   ↓
Packages téléchargés
```

C'est l'équivalent conceptuel de préparer les dépendances avant la compilation.

---

# 12. Build

```yaml
- name: Build
  run: dotnet build --no-restore
```

Le projet est compilé.

Pourquoi :

```text
--no-restore
```

?

Parce que le restore a déjà été effectué précédemment.

On évite donc de refaire inutilement cette opération.

---

# 13. Test

```yaml
- name: Test
  run: dotnet test --no-build
```

Les tests automatisés sont exécutés.

Exemple :

```text
Build
  ↓
Tests
  ↓
OK ?
 ├── Non → workflow échoue
 └── Oui → étape suivante
```

C'est essentiel avant un déploiement automatique.

---

# 14. Publish

Pour déployer une application ASP.NET Core, on utilise généralement :

```yaml
dotnet publish
```

Exemple :

```yaml
- name: Publish
  run: dotnet publish -c Release -o ./publish
```

Attention à la différence :

```text
build
```

compile le projet.

```text
publish
```

prépare les fichiers nécessaires à l'exécution et au déploiement.

Mentalement :

```text
build   = "Est-ce que mon code compile ?"

publish = "Prépare-moi l'application à être exécutée/déployée."
```

---

# 15. Exemple complet : ASP.NET Core → Azure App Service

Voici un exemple de workflow CI/CD.

```yaml
name: Build and Deploy ASP.NET Core

on:
  push:
    branches:
      - main

  workflow_dispatch:

permissions:
  contents: read
  id-token: write

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'

      - name: Restore
        run: dotnet restore

      - name: Build
        run: dotnet build --configuration Release --no-restore

      - name: Test
        run: dotnet test --configuration Release --no-build

      - name: Publish
        run: dotnet publish --configuration Release --output ./publish

      - name: Login to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Deploy to Azure App Service
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ secrets.AZURE_WEBAPP_NAME }}
          package: ./publish
```

Ce workflow réalise :

```text
1. Récupérer le code
        ↓
2. Installer .NET
        ↓
3. Restaurer NuGet
        ↓
4. Compiler
        ↓
5. Exécuter les tests
        ↓
6. Publier
        ↓
7. S'authentifier auprès d'Azure
        ↓
8. Déployer sur App Service
```

---

# 16. Pourquoi `id-token: write` ?

Cette partie est importante :

```yaml
permissions:
  contents: read
  id-token: write
```

Elle est liée à l'authentification OIDC.

GitHub Actions peut obtenir un jeton d'identité temporaire permettant de s'authentifier auprès d'Azure.

Cela évite d'avoir à stocker certaines informations d'authentification longues durées dans GitHub.

Mental model :

```text
GitHub Actions
      ↓
Identité GitHub
      ↓
OIDC
      ↓
Azure
```

C'est différent d'une Managed Identity utilisée par une application qui tourne déjà dans Azure.

---

# 17. OIDC et Managed Identity : ne pas les confondre

Les deux notions sont proches mais répondent à des problèmes différents.

## GitHub Actions + OIDC

Question :

> Comment GitHub Actions peut-il s'authentifier auprès d'Azure pour effectuer un déploiement ?

```text
GitHub Actions
      ↓
OIDC
      ↓
Azure
```

## Managed Identity

Question :

> Comment mon application hébergée sur Azure peut-elle accéder à une ressource Azure sans stocker de secret ?

```text
ASP.NET Core
      ↓
Managed Identity
      ↓
Key Vault / Azure SQL / Storage...
```

Donc :

```text
CI/CD → OIDC

Application runtime → Managed Identity
```

C'est une distinction importante en entretien.

---

# 18. Secrets GitHub

Les informations sensibles ne doivent pas être écrites directement dans le YAML.

Mauvais exemple :

```yaml
client-id: "123456789"
```

On préfère :

```yaml
client-id: ${{ secrets.AZURE_CLIENT_ID }}
```

Les secrets peuvent être configurés dans les paramètres du repository ou d'un environnement GitHub.

Exemples :

```text
AZURE_CLIENT_ID
AZURE_TENANT_ID
AZURE_SUBSCRIPTION_ID
AZURE_WEBAPP_NAME
```

Le workflow les récupère au moment de son exécution.

---

# 19. Secrets ≠ configuration applicative

Attention à cette distinction.

Les secrets GitHub servent principalement au workflow GitHub Actions.

Les paramètres de ton application ASP.NET Core sont plutôt configurés côté Azure.

Par exemple :

```text
GitHub
│
└── Secrets
    └── informations nécessaires au déploiement

Azure App Service
│
└── Configuration
    └── ConnectionStrings
    └── JWT settings
    └── API settings
```

Ne mets donc pas automatiquement toute la configuration de ton application dans GitHub Actions.

---

# 20. Environments GitHub

GitHub permet également de définir des environnements :

```text
Development
Staging
Production
```

On peut associer des secrets à ces environnements.

Exemple :

```text
Environment: production

AZURE_CLIENT_ID
AZURE_TENANT_ID
AZURE_SUBSCRIPTION_ID
AZURE_WEBAPP_NAME
```

Un environnement peut également nécessiter une approbation avant le déploiement.

Exemple :

```text
main
 ↓
Build
 ↓
Test
 ↓
Production
 ↓
Approval
 ↓
Deploy
```

C'est particulièrement intéressant pour protéger une production.

---

# 21. Séparer Build et Deploy

Pour un projet plus sérieux, on peut séparer les jobs.

```yaml
jobs:

  build:
    ...

  deploy:
    needs: build
    ...
```

`needs` signifie :

> Ce job dépend de l'exécution réussie d'un autre job.

Exemple :

```yaml
deploy:
  needs: build
```

Donc :

```text
build
  │
  ├── échec → deploy ne démarre pas
  │
  └── succès
       ↓
     deploy
```

---

# 22. Pourquoi utiliser plusieurs jobs ?

Cela permet de construire un pipeline plus clair :

```text
BUILD
 ↓
TEST
 ↓
PACKAGE
 ↓
DEPLOY
```

On peut également avoir :

```text
BUILD
  ↓
TEST
  ↓
┌───────────────┐
│               │
DEV          STAGING
                ↓
            PRODUCTION
```

Cela devient particulièrement utile avec plusieurs environnements.

---

# 23. Déployer un artefact

Dans une CI/CD plus avancée, le résultat du build peut être traité comme un **artifact**.

Mental model :

```text
Source code
    ↓
Build
    ↓
Artifact
    ↓
Deploy
```

L'intérêt est de séparer :

```text
"Construire l'application"
```

de :

```text
"Déployer l'application"
```

On peut alors construire une seule fois et déployer le même résultat dans différents environnements.

---

# 24. Pourquoi ne pas compiler directement sur Azure ?

On pourrait imaginer :

```text
GitHub
  ↓
Azure
  ↓
Azure compile
```

Mais une approche CI/CD classique est plutôt :

```text
GitHub Actions
      ↓
Build
      ↓
Test
      ↓
Publish
      ↓
Artifact
      ↓
Azure
```

Cela permet de contrôler précisément ce qui a été testé avant le déploiement.

---

# 25. Attention aux migrations EF Core

Un point très important avec une application ASP.NET Core + EF Core.

Le déploiement de l'application ne signifie pas automatiquement :

```text
Database schema updated
```

Par exemple :

```text
GitHub Actions
     ↓
Deploy API
     ↓
Azure App Service
```

La base de données peut toujours avoir l'ancien schéma.

Il faut donc réfléchir séparément à la stratégie de migration :

```text
Application deployment
```

et

```text
Database migration
```

Dans un vrai projet, il faut décider où et quand exécuter les migrations.

---

# 26. Variables d'environnement

ASP.NET Core récupère une grande partie de sa configuration via la configuration .NET.

Par exemple :

```text
Azure App Service
    ↓
Environment variables
    ↓
IConfiguration
    ↓
Application
```

Donc ton application peut avoir :

```csharp
var connectionString =
    configuration.GetConnectionString("DefaultConnection");
```

sans que la chaîne de connexion soit écrite dans le repository.

---

# 27. Erreur classique : "Ça marche en local"

Situation :

```text
Local
  ↓
OK

Azure
  ↓
Erreur
```

Les causes fréquentes :

- variable d'environnement absente ;
- connection string absente ;
- mauvais runtime .NET ;
- mauvais nom d'App Service ;
- mauvaise configuration Azure ;
- secret GitHub incorrect ;
- permissions Azure insuffisantes ;
- migration EF Core non appliquée ;
- configuration différente entre Development et Production.

Il faut donc diagnostiquer séparément :

```text
Code
Configuration
Infrastructure
Database
Identity / Permissions
```

---

# 28. Erreur classique : le workflow échoue avant Azure

Si ceci échoue :

```yaml
dotnet build
```

le problème est probablement dans :

```text
Code
Dependencies
SDK
Build configuration
```

Si ceci échoue :

```yaml
azure/login@v2
```

le problème est plutôt :

```text
Authentication
Permissions
OIDC
Azure configuration
```

Si ceci échoue :

```yaml
azure/webapps-deploy@v3
```

il faut vérifier notamment :

```text
App Service
App name
Azure permissions
Published files
Configuration
```

Le premier réflexe est donc de regarder **quelle étape précise a échoué**.

---

# 29. Branche `main` et déploiement automatique

Un scénario simple :

```text
feature/login
      ↓
Pull Request
      ↓
main
      ↓
GitHub Actions
      ↓
Production
```

Mais attention :

> Déployer automatiquement chaque push sur `main` vers la production n'est pas toujours une bonne stratégie.

Une organisation peut préférer :

```text
feature/*
   ↓
develop
   ↓
staging
   ↓
main
   ↓
production
```

ou utiliser une validation manuelle avant production.

---

# 30. Sécurité : bonnes pratiques

Évite :

```text
Secrets dans le code
Secrets dans le YAML
Secrets dans Git
Long-lived credentials
Permissions excessives
```

Privilégie :

```text
OIDC
Secrets GitHub lorsque nécessaire
Managed Identity pour les applications Azure
Permissions minimales
Environments protégés
Approvals pour Production
```

Principe important :

> Une identité ne doit avoir que les permissions dont elle a réellement besoin.

C'est le principe du **Least Privilege**.

---

# 31. Workflow mental complet

Quand tu vois :

```yaml
name: Deploy

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
```

Lis-le comme une phrase :

> "Quand un push arrive sur main, GitHub démarre une machine Ubuntu et exécute les étapes suivantes."

Puis :

```yaml
uses:
```

signifie :

> "Utilise cette Action."

Et :

```yaml
run:
```

signifie :

> "Exécute cette commande."

---

# 32. CI/CD dans un projet .NET réel

Un pipeline professionnel peut ressembler à :

```text
Developer
   │
   │ git push
   ↓
GitHub
   │
   ↓
GitHub Actions
   │
   ├── Checkout
   │
   ├── Setup .NET
   │
   ├── Restore
   │
   ├── Build
   │
   ├── Unit Tests
   │
   ├── Integration Tests
   │
   ├── Security checks
   │
   ├── Publish
   │
   └── Artifact
          │
          ↓
       Staging
          │
          ↓
      Approval
          │
          ↓
      Production
          │
          ↓
    Azure App Service
```

C'est beaucoup plus proche d'un environnement professionnel qu'un simple :

```text
git push → deploy
```

---

# 33. Exemple d'entretien

### Question

**Quelle est la différence entre CI et CD ?**

Réponse :

> La CI automatise l'intégration et la validation du code, notamment le restore, le build et les tests. Le CD automatise ensuite la livraison ou le déploiement de l'application vers un environnement.

---

### Question

**Pourquoi utiliser `dotnet publish` ?**

Réponse :

> `dotnet build` compile le projet, tandis que `dotnet publish` prépare les fichiers nécessaires à l'exécution et au déploiement de l'application.

---

### Question

**Comment sécuriser l'authentification GitHub Actions vers Azure ?**

Réponse :

> Je privilégie OIDC avec une identité Azure configurée pour GitHub Actions afin d'éviter autant que possible les credentials persistants. Les permissions doivent également respecter le principe du moindre privilège.

---

### Question

**Quelle différence entre OIDC et Managed Identity ?**

Réponse :

> OIDC permet notamment à GitHub Actions de s'authentifier auprès d'Azure pendant le pipeline CI/CD. Une Managed Identity permet à une ressource Azure, comme une App Service, de s'authentifier auprès d'autres services Azure sans stocker de secret dans l'application.

---

# 34. Checklist GitHub Actions + Azure

Avant de considérer le pipeline terminé :

```text
[ ] Workflow dans .github/workflows/
[ ] Trigger correct
[ ] Version .NET correcte
[ ] Restore
[ ] Build
[ ] Tests
[ ] Publish
[ ] Authentification Azure
[ ] Permissions minimales
[ ] Secrets correctement configurés
[ ] Nom de l'App Service correct
[ ] Configuration Azure correcte
[ ] Connection strings configurées
[ ] Variables d'environnement configurées
[ ] Stratégie EF Core migrations définie
[ ] Production protégée si nécessaire
```

---

# À retenir

```text
GitHub Actions = automatisation du pipeline

CI:
Restore → Build → Test

CD:
Publish → Deploy

uses:
Utiliser une Action

run:
Exécuter une commande

OIDC:
Authentifier GitHub Actions auprès d'Azure

Managed Identity:
Authentifier une application Azure auprès d'autres ressources Azure

Secrets:
Ne jamais mettre les credentials directement dans le code

Artifact:
Résultat produit par le build et réutilisable pour le déploiement
```

## Phrase à mémoriser

> **GitHub Actions automatise mon pipeline : je récupère le code, je restaure, je compile, je teste, je publie puis je déploie vers Azure, avec une authentification sécurisée et des permissions minimales.**
