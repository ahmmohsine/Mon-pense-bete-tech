# Azure SQL Database

## 1. Qu'est-ce qu'Azure SQL Database ?

Azure SQL Database est un service de base de données relationnelle managé basé sur le moteur SQL Server.

Mental model :

```text
SQL Server
   ↓
Base de données relationnelle

Azure SQL Database
   ↓
SQL Server dans un service Cloud managé par Azure
```

L'objectif est de pouvoir utiliser SQL sans avoir à administrer soi-même une machine virtuelle contenant SQL Server.

Une architecture .NET classique peut être :

```text
Client
   |
   v
ASP.NET Core API
   |
   v
Entity Framework Core
   |
   v
Azure SQL Database
```

---

# 2. Pourquoi utiliser Azure SQL ?

Avec une base SQL classique, on doit notamment penser à :

```text
Serveur
OS
Installation SQL Server
Patches
Backups
Disponibilité
Sécurité
Monitoring
```

Avec Azure SQL Database, Azure prend en charge une grande partie de l'infrastructure.

Le développeur peut principalement se concentrer sur :

```text
Tables
Relations
Requêtes
Transactions
Indexes
Application
```

Mental model :

```text
Application
    ↓
EF Core
    ↓
Azure SQL
    ↓
Azure gère une grande partie de l'infrastructure
```

---

# 3. Azure SQL Database vs SQL Server

Il ne faut pas penser :

```text
Azure SQL = autre langage SQL
```

Azure SQL Database repose sur SQL Server et utilise T-SQL.

On retrouve donc des concepts familiers :

```text
Tables
Primary Keys
Foreign Keys
Indexes
Views
Stored Procedures
Transactions
Constraints
```

Mais le modèle de service et l'administration diffèrent.

---

# 4. Architecture avec ASP.NET Core

Une API ASP.NET Core peut utiliser EF Core :

```csharp
builder.Services.AddDbContext<ApplicationDbContext>(
    options =>
        options.UseSqlServer(
            builder.Configuration.GetConnectionString("Default")));
```

La chaîne de connexion n'est pas nécessairement écrite directement dans le code.

On peut avoir :

```text
ASP.NET Core
      |
      v
IConfiguration
      |
      v
Connection String
      |
      v
EF Core
      |
      v
Azure SQL
```

---

# 5. Connection String

Une connection string contient les informations nécessaires pour se connecter à la base.

Exemple conceptuel :

```text
Server=tcp:myserver.database.windows.net,1433;
Initial Catalog=HotelDb;
User ID=...;
Password=...;
Encrypt=True;
TrustServerCertificate=False;
```

Il ne faut pas copier aveuglément une connection string réelle dans un dépôt Git public.

Une connection string peut contenir des informations sensibles.

Mauvais :

```json
{
  "ConnectionStrings": {
    "Default": "Server=...;Password=secret;"
  }
}
```

dans un dépôt public.

Préférer une configuration sécurisée :

```text
Environment Variables
Key Vault
Managed Identity
```

selon le mécanisme d'authentification choisi.

---

# 6. Configuration ASP.NET Core

Dans le code :

```csharp
var connectionString =
    builder.Configuration
        .GetConnectionString("Default");
```

Puis :

```csharp
builder.Services.AddDbContext<ApplicationDbContext>(
    options =>
        options.UseSqlServer(connectionString));
```

Le code ne connaît donc pas nécessairement la valeur réelle.

Mental model :

```text
Code
 ↓
"Default"

Configuration
 ↓
valeur réelle

EF Core
 ↓
Azure SQL
```

---

# 7. Local vs Production

En développement :

```text
Application locale
        |
        v
Local SQL Server / LocalDB / Docker
```

En production :

```text
Application Azure
        |
        v
Azure SQL
```

Le code peut rester identique.

Ce qui change principalement :

```text
Configuration
```

Exemple :

```text
Development
ConnectionStrings:Default
        ↓
localhost

Production
ConnectionStrings:Default
        ↓
myserver.database.windows.net
```

C'est un principe fondamental des applications Cloud :

> Le code ne doit pas être rempli de valeurs spécifiques à un environnement.

---

# 8. Firewall Azure SQL

Azure SQL doit contrôler qui peut se connecter.

Le serveur SQL possède des règles réseau permettant de limiter les connexions.

Mental model :

```text
Internet
   |
   | Connexion
   v
Azure SQL
   |
   | Firewall
   |
   +---- Autorisé
   |
   +---- Refusé
```

Une erreur de firewall peut produire une situation où :

```text
API fonctionne
```

mais :

```text
API → Azure SQL = impossible
```

Il faut donc distinguer :

```text
Problème applicatif
```

et :

```text
Problème réseau / firewall
```

---

# 9. Authentification SQL

Il existe plusieurs façons d'authentifier une application auprès d'Azure SQL.

Un modèle traditionnel utilise :

```text
SQL Username
+
Password
```

Mais dans Azure, l'authentification Microsoft Entra peut également être utilisée.

L'intérêt est de réduire la dépendance aux mots de passe SQL et de mieux intégrer l'identité Azure.

Mental model :

```text
Application
    |
    | Identity
    v
Microsoft Entra ID
    |
    v
Azure SQL
```

---

# 10. Managed Identity + Azure SQL

Une architecture Azure moderne peut utiliser une Managed Identity pour permettre à l'application de s'authentifier auprès d'Azure SQL.

Conceptuellement :

```text
Azure App Service
       |
       | Managed Identity
       v
Microsoft Entra ID
       |
       v
Azure SQL
```

L'application n'a alors pas nécessairement besoin de stocker un mot de passe SQL dans sa configuration.

Il faut cependant toujours configurer correctement les permissions côté Azure SQL.

Important :

```text
Managed Identity activée
        ≠
accès automatique à Azure SQL
```

L'identité doit avoir les droits nécessaires.

---

# 11. EF Core et Azure SQL

Pour EF Core, Azure SQL est généralement utilisé avec le provider SQL Server :

```csharp
Microsoft.EntityFrameworkCore.SqlServer
```

Configuration :

```csharp
builder.Services.AddDbContext<ApplicationDbContext>(
    options =>
        options.UseSqlServer(connectionString));
```

EF Core génère ensuite les requêtes SQL nécessaires à partir des opérations LINQ.

Mental model :

```text
C#
 ↓
LINQ
 ↓
EF Core
 ↓
SQL
 ↓
Azure SQL
```

---

# 12. Exemple de DbContext

```csharp
public class ApplicationDbContext
    : DbContext
{
    public ApplicationDbContext(
        DbContextOptions<ApplicationDbContext> options)
        : base(options)
    {
    }

    public DbSet<Hotel> Hotels => Set<Hotel>();
}
```

Configuration :

```csharp
builder.Services.AddDbContext<ApplicationDbContext>(
    options =>
        options.UseSqlServer(
            builder.Configuration
                .GetConnectionString("Default")));
```

Puis :

```csharp
public class HotelService(
    ApplicationDbContext context)
{
    public async Task<List<Hotel>> GetHotelsAsync()
    {
        return await context.Hotels
            .AsNoTracking()
            .ToListAsync();
    }
}
```

Le code métier n'a pas besoin de connaître les détails internes du serveur Azure SQL.

---

# 13. Migrations EF Core

Les migrations permettent de faire évoluer le schéma de la base.

Exemple :

```text
Model C#
   ↓
Migration
   ↓
SQL
   ↓
Database
```

Commande typique :

```bash
dotnet ef migrations add InitialCreate
```

Puis :

```bash
dotnet ef database update
```

Attention :

```text
database update
```

sur une base de production doit être utilisé avec une stratégie de déploiement réfléchie.

Une migration de production est une opération potentiellement critique.

---

# 14. Migration en production

Une application peut évoluer :

```text
Version 1
    ↓
Version 2
```

Le schéma SQL doit également évoluer :

```text
Database v1
    ↓
Migration
    ↓
Database v2
```

Une stratégie de déploiement sérieuse doit prendre en compte :

```text
Compatibilité
Rollback
Durée de migration
Verrous
Volume de données
Disponibilité
```

Il ne faut pas considérer une migration de production comme une simple commande sans risque.

---

# 15. Index

Un index permet d'améliorer certaines recherches.

Exemple :

```sql
CREATE INDEX IX_Hotels_Name
ON Hotels(Name);
```

Sans index adapté, une base peut devoir examiner beaucoup de lignes.

Avec un index :

```text
Query
  ↓
Index
  ↓
Rows pertinentes
```

Mais un index n'est pas gratuit.

Il consomme :

```text
Storage
CPU
I/O
```

et peut ralentir certaines opérations d'écriture.

Mental model :

> Un index accélère certaines lectures au prix d'un coût supplémentaire sur le stockage et les écritures.

---

# 16. Performance SQL

Avec EF Core :

```csharp
var hotels = await context.Hotels
    .Where(h => h.Country == "Belgium")
    .ToListAsync();
```

EF Core traduit la requête LINQ en SQL.

Le développeur doit donc comprendre que :

```text
LINQ
 ↓
SQL généré
 ↓
Database execution plan
```

Une requête LINQ élégante n'est pas automatiquement une requête SQL optimale.

Il faut surveiller :

```text
Indexes
Joins
Filters
Pagination
N+1 queries
Projection
Tracking
```

---

# 17. Projection

Si on a :

```csharp
var hotels = await context.Hotels
    .ToListAsync();
```

on peut récupérer davantage de données que nécessaire.

Une projection permet de sélectionner uniquement les colonnes utiles :

```csharp
var hotels = await context.Hotels
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .ToListAsync();
```

Mental model :

```text
Mauvais réflexe
Database
   ↓
toutes les colonnes
   ↓
application
   ↓
on jette la moitié

Meilleur réflexe
Database
   ↓
colonnes nécessaires
   ↓
application
```

Cela peut réduire la quantité de données transférées et le travail effectué.

---

# 18. Pagination

Une API ne devrait généralement pas renvoyer des millions de lignes en une seule requête.

Mauvais :

```csharp
await context.Hotels.ToListAsync();
```

si la table contient énormément de données.

On peut utiliser une pagination :

```text
page = 1
pageSize = 20
```

Mental model :

```text
100000 records
     ↓
API
     ↓
20 records
```

La pagination doit être pensée côté SQL pour éviter de charger inutilement toutes les lignes.

---

# 19. AsNoTracking

Pour une requête en lecture seule :

```csharp
var hotels = await context.Hotels
    .AsNoTracking()
    .ToListAsync();
```

`AsNoTracking()` indique à EF Core de ne pas suivre les entités récupérées pour une modification ultérieure dans le contexte.

Cela peut réduire le coût du tracking lorsque les données sont simplement lues.

Mental model :

```text
Lecture seule
   ↓
AsNoTracking()
   ↓
moins de travail de tracking
```

---

# 20. Transactions

Une transaction permet de regrouper plusieurs opérations dans une unité logique.

Mental model :

```text
Transaction
   |
   +-- Operation A
   |
   +-- Operation B
   |
   +-- Operation C
```

Si tout réussit :

```text
COMMIT
```

Si une opération critique échoue :

```text
ROLLBACK
```

Le but est notamment de préserver la cohérence des données.

---

# 21. Backups

Une base de production doit avoir une stratégie de sauvegarde.

Azure SQL fournit des mécanismes managés de sauvegarde et de restauration selon le service et sa configuration.

Il faut toutefois comprendre :

```text
Backup disponible
        ≠
Plan de récupération entièrement testé
```

Une vraie stratégie doit considérer :

```text
RPO
RTO
Retention
Restore
Disaster Recovery
```

---

# 22. RPO et RTO

Deux notions importantes en Cloud.

## RPO

```text
Recovery Point Objective
```

Question :

> Quelle quantité maximale de données puis-je accepter de perdre ?

Exemple :

```text
RPO = 15 minutes
```

signifie conceptuellement que l'organisation accepte au maximum une perte de données correspondant à cette fenêtre.

## RTO

```text
Recovery Time Objective
```

Question :

> Combien de temps puis-je accepter que le service soit indisponible ?

Exemple :

```text
RTO = 1 heure
```

Mental model :

```text
RPO = combien de données puis-je perdre ?

RTO = combien de temps puis-je être indisponible ?
```

---

# 23. Sécurité des données

Pour une base Azure SQL, penser à plusieurs niveaux :

```text
Network
Authentication
Authorization
Encryption
Secrets
Monitoring
Backups
```

Une base ne doit pas être considérée comme sécurisée uniquement parce qu'elle se trouve dans Azure.

Il faut configurer correctement :

```text
Firewall
Identity
Permissions
Connection
Secrets
Application
```

---

# 24. Least Privilege

L'application ne devrait pas utiliser inutilement un compte possédant tous les droits.

Mauvais principe :

```text
API
 ↓
db_owner
```

si l'application n'en a pas besoin.

Meilleur principe :

```text
API
 ↓
permissions minimales nécessaires
```

Cela limite l'impact en cas de compromission.

---

# 25. Azure SQL et Key Vault

Une architecture peut être :

```text
App Service
    |
    | Managed Identity
    v
Key Vault
    |
    v
Database configuration
    |
    v
Azure SQL
```

Mais dans une architecture utilisant Microsoft Entra authentication, on peut aller plus loin et éviter de stocker certains mots de passe SQL.

Il faut donc distinguer :

```text
Secret-based authentication
```

et :

```text
Identity-based authentication
```

---

# 26. Erreurs fréquentes

## Erreur 1 : mettre la connection string dans Git

```text
Password=...
```

dans un dépôt public est une mauvaise pratique.

---

## Erreur 2 : croire que Managed Identity donne automatiquement accès à SQL

Faux.

Il faut configurer les permissions nécessaires.

---

## Erreur 3 : croire qu'Azure SQL résout automatiquement les problèmes de performance

Azure gère l'infrastructure du service, mais une mauvaise requête reste une mauvaise requête.

Exemples :

```text
N+1
Pas d'index
SELECT inutilement énorme
Pas de pagination
Joins coûteux
```

---

## Erreur 4 : charger toutes les données

```csharp
ToListAsync()
```

sur une énorme table peut être problématique.

---

## Erreur 5 : créer trop d'indexes

Les indexes ont un coût.

```text
Plus d'indexes
    ↓
lectures potentiellement plus rapides
    +
écritures potentiellement plus coûteuses
    +
plus de stockage
```

---

## Erreur 6 : modifier directement la base de production

Les changements de schéma doivent être contrôlés et idéalement automatisés selon une stratégie de déploiement adaptée.

---

# 27. Checklist Azure SQL

```text
[ ] Connection string hors du code source
[ ] HTTPS / TLS utilisé
[ ] Firewall correctement configuré
[ ] Authentication adaptée
[ ] Managed Identity envisagée
[ ] Permissions minimales
[ ] EF Core configuré correctement
[ ] Migrations maîtrisées
[ ] Indexes adaptés
[ ] Pagination pour les gros volumes
[ ] Projections utilisées lorsque pertinentes
[ ] AsNoTracking pour certaines lectures
[ ] Transactions utilisées lorsque nécessaire
[ ] Backups compris
[ ] RPO défini
[ ] RTO défini
[ ] Monitoring configuré
[ ] Logs disponibles
```

---

# 28. Questions d'entretien

### Qu'est-ce qu'Azure SQL Database ?

Un service PaaS de base de données relationnelle managé basé sur le moteur SQL Server.

### Quelle différence entre Azure SQL et SQL Server installé sur une VM ?

Avec Azure SQL Database, Azure prend en charge une grande partie de l'infrastructure et de l'administration du service. Avec une VM SQL Server, l'équipe doit gérer beaucoup plus d'éléments.

### Comment connecter EF Core à Azure SQL ?

Avec le provider SQL Server et une configuration telle que :

```csharp
options.UseSqlServer(connectionString);
```

### Où stocker la connection string ?

Selon l'architecture :

```text
Environment Variables
Key Vault
Identity-based authentication
```

plutôt que directement dans le code ou un dépôt public.

### À quoi sert un index ?

À accélérer certaines opérations de recherche, au prix d'un coût en stockage et potentiellement en écriture.

### Pourquoi utiliser AsNoTracking ?

Pour les lectures où les entités n'ont pas besoin d'être suivies par EF Core pour une modification ultérieure.

### Pourquoi la pagination est-elle importante ?

Pour éviter de charger et transférer inutilement un très grand nombre de lignes.

### Qu'est-ce que RPO ?

La quantité maximale de données que l'organisation accepte de perdre lors d'une récupération.

### Qu'est-ce que RTO ?

Le délai maximal acceptable pour restaurer le service après une interruption.

### Managed Identity suffit-elle pour accéder à Azure SQL ?

Non. Elle fournit l'identité ; les permissions nécessaires doivent également être configurées côté Azure SQL.

---

# 29. Schéma mental final

```text
                    ASP.NET CORE
                         |
                         v
                    EF CORE
                         |
                  Connection / Identity
                         |
                         v
                    AZURE SQL
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
     Tables           Indexes         Transactions
        |
        v
      Data
```

Sécurité :

```text
App Service
     |
     | Managed Identity
     v
Microsoft Entra ID
     |
     v
Azure SQL
     |
     v
Permissions
```

Performance :

```text
LINQ
  ↓
SQL
  ↓
Indexes
  ↓
Execution
  ↓
Rows nécessaires
```

---

# À retenir

Azure SQL Database est une base relationnelle managée adaptée aux applications .NET.

Les notions essentielles sont :

```text
Connection String
        ↓
Configuration

EF Core
        ↓
Accès aux données

Firewall
        ↓
Contrôle réseau

Authentication / Managed Identity
        ↓
Identité

RBAC / SQL permissions
        ↓
Autorisation

Indexes + Queries + Pagination
        ↓
Performance

Backup + RPO + RTO
        ↓
Résilience
```

## Phrase à mémoriser

> **Azure SQL me fournit une base SQL managée, mais c'est toujours à mon application de gérer correctement ses requêtes, ses permissions, sa configuration et son modèle de données.**
