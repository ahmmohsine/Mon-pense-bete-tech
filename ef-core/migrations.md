# EF Core — Migrations

Les **migrations EF Core** permettent de faire évoluer le schéma d'une base de données lorsque le modèle de données de l'application change.

Le point essentiel à comprendre est :

```text
Modèle C#
    |
    v
Migration
    |
    v
Database Schema
```

Une migration est donc une **description d'un changement de schéma**, pas une copie de la base de données.

---

# 1. Pourquoi les migrations existent ?

Imagine que ton application commence avec :

```csharp
public class Hotel
{
    public int Id { get; set; }

    public string Name { get; set; } = string.Empty;
}
```

La base contient :

```text
Hotels
----------------
Id
Name
```

Puis tu ajoutes :

```csharp
public int Stars { get; set; }
```

Ton modèle C# devient :

```text
Hotels
----------------
Id
Name
Stars
```

Mais la base existante ne possède pas automatiquement cette colonne.

Il faut donc faire évoluer le schéma.

C'est précisément le rôle des migrations.

---

# 2. Le modèle mental

Pense à trois éléments :

```text
1. Model C#
       |
       v
2. Migration
       |
       v
3. Database
```

Le modèle représente :

> « Voilà à quoi mon application veut que les données ressemblent. »

La migration représente :

> « Voilà ce qui doit changer pour faire évoluer le schéma. »

La base représente :

> « Voilà l'état actuellement présent. »

---

# 3. Une migration n'est pas la base

Une migration n'est pas :

```text
Database backup
```

et ce n'est pas non plus :

```text
Copie complète du schéma
```

Elle décrit des opérations permettant de passer d'un état à un autre.

Exemple :

```text
Version 1
    |
    | Migration
    v
Version 2
```

---

# 4. Exemple simple

Modèle initial :

```csharp
public class Hotel
{
    public int Id { get; set; }

    public string Name { get; set; } = string.Empty;
}
```

Après modification :

```csharp
public class Hotel
{
    public int Id { get; set; }

    public string Name { get; set; } = string.Empty;

    public int Stars { get; set; }
}
```

On crée une migration.

Conceptuellement, elle dira :

```text
Ajouter la colonne Stars
```

Puis on applique cette migration à la base.

---

# 5. Commande `dotnet ef migrations add`

Avec la CLI :

```bash
dotnet ef migrations add AddHotelStars
```

Cette commande demande à EF Core de créer une nouvelle migration.

Le nom :

```text
AddHotelStars
```

est un nom choisi par le développeur.

Il doit être descriptif.

Exemples :

```text
InitialCreate
AddHotelStars
AddUserRoles
CreateBookingTable
AddHotelAddress
```

---

# 6. Que fait réellement `migrations add` ?

Une confusion fréquente :

```bash
dotnet ef migrations add AddHotelStars
```

ne signifie pas :

> « Mets à jour la base. »

Cela signifie plutôt :

> « Compare le modèle actuel avec l'état connu des migrations et génère une nouvelle migration représentant les changements. »

Mentalement :

```text
Model actuel
     |
     v
EF Core
     |
     v
Migration générée
```

La base n'est pas nécessairement modifiée par cette commande.

---

# 7. `dotnet ef database update`

Pour appliquer les migrations :

```bash
dotnet ef database update
```

Mentalement :

```text
Migrations
    |
    v
Database
```

Cette commande demande à EF Core d'appliquer les migrations nécessaires à la base ciblée.

---

# 8. Différence entre les deux commandes

Très important :

```bash
dotnet ef migrations add AddHotelStars
```

=

```text
Créer une migration
```

Alors que :

```bash
dotnet ef database update
```

=

```text
Appliquer les migrations à la base
```

À retenir :

```text
migrations add
    -> prépare le changement

database update
    -> applique le changement
```

---

# 9. Structure d'une migration

Une migration générée contient généralement deux méthodes importantes :

```csharp
protected override void Up(MigrationBuilder migrationBuilder)
{
    // appliquer le changement
}

protected override void Down(MigrationBuilder migrationBuilder)
{
    // annuler le changement
}
```

Exemple conceptuel :

```csharp
protected override void Up(
    MigrationBuilder migrationBuilder)
{
    migrationBuilder.AddColumn<int>(
        name: "Stars",
        table: "Hotels",
        nullable: false,
        defaultValue: 0);
}
```

Et :

```csharp
protected override void Down(
    MigrationBuilder migrationBuilder)
{
    migrationBuilder.DropColumn(
        name: "Stars",
        table: "Hotels");
}
```

---

# 10. `Up()` et `Down()`

Il faut retenir :

```text
Up()
   =
avance vers la nouvelle version

Down()
   =
retour vers la version précédente
```

Mentalement :

```text
V1
 |
 | Up()
 v
V2
 |
 | Down()
 v
V1
```

---

# 11. Exemple avec une table

Tu ajoutes :

```csharp
public class Room
{
    public int Id { get; set; }

    public string Number { get; set; } = string.Empty;
}
```

Une migration peut conceptuellement générer :

```text
CREATE TABLE Rooms
```

Puis :

```bash
dotnet ef database update
```

applique le changement.

---

# 12. Ajouter une relation

Supposons :

```text
Hotel 1 ---- N Room
```

Tu ajoutes :

```csharp
public int HotelId { get; set; }

public Hotel Hotel { get; set; } = null!;
```

La migration peut devoir créer :

```text
HotelId
Foreign Key
Index
```

selon le modèle et le provider.

Mentalement :

```text
C# relation
    |
    v
EF Core model
    |
    v
Migration
    |
    v
Foreign Key SQL
```

---

# 13. Historique des migrations

EF Core garde un historique des migrations appliquées dans la base.

Une table système dédiée permet notamment de savoir quelles migrations ont déjà été appliquées.

Conceptuellement :

```text
Database
 |
 +-- Hotels
 +-- Rooms
 +-- Users
 |
 +-- __EFMigrationsHistory
```

Cette table permet à EF Core de déterminer quelles migrations restent à appliquer.

---

# 14. Exemple d'historique

Supposons que le projet possède :

```text
202610010001_InitialCreate
202610020002_AddRooms
202610030003_AddHotelStars
```

Et que la base ait seulement :

```text
InitialCreate
AddRooms
```

EF Core peut déterminer :

```text
AddHotelStars
```

comme migration restante.

---

# 15. `dotnet ef migrations list`

Pour afficher les migrations :

```bash
dotnet ef migrations list
```

Cela permet de voir l'historique des migrations présentes dans le projet.

Selon les options et la configuration utilisée, tu peux également distinguer les migrations appliquées de celles qui ne le sont pas.

---

# 16. Supprimer la dernière migration

Si tu viens de créer une migration mais que tu ne veux finalement pas la conserver :

```bash
dotnet ef migrations remove
```

Cette commande sert notamment à retirer la dernière migration du projet lorsque les conditions permettent sa suppression.

Attention :

> Supprimer une migration du projet n'est pas la même chose que revenir en arrière sur une migration déjà appliquée en production.

---

# 17. Revenir à une migration précédente

Pour revenir à une migration donnée :

```bash
dotnet ef database update PreviousMigration
```

Conceptuellement :

```text
V3
 |
 | database update V2
 v
V2
```

EF Core utilise les opérations `Down()` nécessaires pour revenir à l'état demandé.

---

# 18. Revenir à zéro

Dans certains scénarios de développement, on peut utiliser :

```bash
dotnet ef database update 0
```

Cela demande à EF Core de revenir avant la première migration.

Attention :

```text
Migration rollback
    !=
simple suppression de fichiers
```

Le schéma de la base est réellement modifié.

---

# 19. Supprimer toutes les migrations

Il ne faut pas supprimer les fichiers de migrations au hasard.

Les migrations font partie de l'historique du schéma.

Si tu veux repartir de zéro dans un environnement de développement, une stratégie peut consister à :

```text
1. supprimer/recréer la base
2. supprimer les migrations
3. créer une nouvelle migration initiale
```

Mais cette approche n'est généralement pas acceptable pour une base de production contenant des données importantes.

---

# 20. Migration et production

En production, il faut être beaucoup plus prudent.

Une migration peut modifier :

```text
Tables
Colonnes
Indexes
Foreign Keys
Contraintes
Types
```

et potentiellement avoir un impact important sur les données ou les performances.

Il faut donc considérer les migrations comme des changements de schéma réels.

---

# 21. Pourquoi les migrations sont versionnées dans Git ?

Les fichiers de migration font partie de l'historique technique du projet.

Exemple :

```text
Git
 |
 +-- Migration 1
 +-- Migration 2
 +-- Migration 3
```

Un autre développeur récupère le projet et peut comprendre comment le schéma a évolué.

Mentalement :

```text
Code source
    +
Migrations
    =
Historique reproductible du schéma
```

---

# 22. Migration et collaboration

Supposons :

```text
Développeur A
    |
    +-- AddHotelStars

Développeur B
    |
    +-- AddHotelAddress
```

Les deux migrations doivent être gérées correctement lorsqu'elles sont intégrées.

Il faut éviter de manipuler l'historique des migrations comme de simples fichiers indépendants sans comprendre leur ordre et leurs dépendances.

---

# 23. Migration et branches Git

Une migration est liée à l'état du modèle au moment où elle est créée.

Si plusieurs branches modifient simultanément le modèle :

```text
main
 |
 +-- branch A -> migration A
 |
 +-- branch B -> migration B
```

un merge peut nécessiter une attention particulière.

EF Core possède des mécanismes pour gérer certaines situations de migrations multiples, mais il faut surtout retenir :

> Les migrations sont une histoire ordonnée de l'évolution du schéma.

---

# 24. Migration et `ModelSnapshot`

EF Core utilise également un fichier de snapshot du modèle dans les projets utilisant les migrations.

Conceptuellement :

```text
Current Model
      |
      v
Model Snapshot
      |
      v
Migration generation
```

Le snapshot aide EF Core à déterminer les différences entre le modèle actuel et l'état représenté par les migrations.

---

# 25. Attention au snapshot

Le fichier snapshot est généré par EF Core.

Il ne faut généralement pas le modifier manuellement sans raison très précise.

Il fait partie des fichiers techniques nécessaires à la gestion des migrations.

---

# 26. Migration et données existantes

Ajouter une colonne peut sembler simple :

```csharp
public int Stars { get; set; }
```

Mais si la table contient déjà des lignes, il faut réfléchir à :

```text
Quelle valeur pour les anciennes lignes ?
```

Par exemple, une colonne obligatoire :

```csharp
public int Stars { get; set; }
```

peut nécessiter une valeur par défaut ou une stratégie de migration adaptée.

C'est particulièrement important lorsque la base contient déjà des données.

---

# 27. Migration destructive

Certaines modifications peuvent être destructrices.

Exemple :

```text
DropColumn
```

Si tu supprimes :

```text
Email
```

les données existantes peuvent être perdues.

Donc :

```text
Migration simple
    ≠
Migration sans risque
```

Avant une migration destructive, il faut réfléchir aux données existantes et à la stratégie de déploiement.

---

# 28. Renommer une colonne

Attention à la différence entre :

```text
Rename
```

et :

```text
Drop + Add
```

Supposons :

```text
Name
```

devient :

```text
DisplayName
```

Une migration qui comprend réellement le changement comme un renommage peut préserver les données :

```text
Name
  |
  v
DisplayName
```

Alors qu'un :

```text
Drop Name
Add DisplayName
```

peut supprimer l'ancien contenu.

Il faut donc vérifier la migration générée.

---

# 29. Toujours lire une migration

Après :

```bash
dotnet ef migrations add AddHotelStars
```

ne considère pas la migration comme une boîte noire.

Ouvre-la.

Regarde :

```csharp
Up()
Down()
```

Demande-toi :

```text
Qu'est-ce qui va réellement changer dans la base ?
```

C'est une excellente habitude professionnelle.

---

# 30. Générer le script SQL

EF Core peut également générer un script SQL à partir des migrations.

Conceptuellement :

```bash
dotnet ef migrations script
```

Cela permet d'obtenir les instructions SQL correspondantes.

C'est particulièrement utile lorsque l'équipe ou le processus de déploiement souhaite contrôler explicitement les changements de base.

---

# 31. Migration script idempotent

EF Core peut également générer un script idempotent.

L'idée est :

> « Applique les migrations manquantes sans réappliquer celles qui existent déjà. »

Conceptuellement :

```bash
dotnet ef migrations script --idempotent
```

Cela peut être utile dans certains scénarios de déploiement.

---

# 32. Migrations et CI/CD

Dans une architecture avec CI/CD :

```text
Developer
    |
    v
Git
    |
    v
Build
    |
    v
Tests
    |
    v
Deployment
    |
    v
Database migration
```

Il faut décider clairement :

```text
Qui applique les migrations ?
Quand ?
Sur quelle base ?
Avec quelles permissions ?
```

Il ne faut pas considérer :

```csharp
Database.Migrate();
```

comme une réponse universelle pour tous les environnements.

---

# 33. `Database.Migrate()`

EF Core permet de déclencher l'application des migrations depuis le code :

```csharp
await context.Database.MigrateAsync();
```

C'est pratique dans certains scénarios, notamment certains environnements de développement ou de déploiement contrôlé.

Mais en production, il faut réfléchir aux droits de la connexion et au contrôle des changements.

---

# 34. Migration vs Seeding

Ces deux concepts sont souvent confondus.

### Migration

Modifie principalement le **schéma** :

```text
Table
Column
Foreign Key
Index
Constraint
```

### Seeding

Initialise ou ajoute certaines **données** :

```text
Admin
Roles
Categories
Default data
```

Mentalement :

```text
Migration
    =
structure

Seeding
    =
données
```

---

# 35. Migration vs `EnsureCreated()`

Il existe également :

```csharp
context.Database.EnsureCreated();
```

Il ne faut pas le confondre avec les migrations.

`EnsureCreated()` sert notamment à créer une base si elle n'existe pas, sans utiliser le système de migrations pour gérer l'évolution progressive du schéma.

Pour une application dont le schéma doit évoluer avec le temps, les migrations sont généralement le mécanisme approprié.

Mentalement :

```text
EnsureCreated
    =
Créer directement le schéma

Migrations
    =
Faire évoluer le schéma version par version
```

---

# 36. Erreurs fréquentes

## Erreur 1 : penser que `migrations add` modifie la base

```bash
dotnet ef migrations add AddRoom
```

crée une migration.

Il faut ensuite appliquer la migration.

```bash
dotnet ef database update
```

---

## Erreur 2 : ne pas regarder la migration

Toujours vérifier :

```text
Up()
Down()
```

avant de considérer le changement comme correct.

---

## Erreur 3 : supprimer des migrations déjà utilisées en production

Les migrations appliquées font partie de l'historique du schéma.

Il ne faut pas simplement réécrire l'histoire parce qu'une migration n'est plus jolie.

---

## Erreur 4 : oublier les données existantes

Une nouvelle colonne obligatoire peut poser problème pour les lignes déjà présentes.

---

## Erreur 5 : utiliser `EnsureCreated()` avec une stratégie de migrations

Les deux mécanismes répondent à des besoins différents.

---

## Erreur 6 : ignorer les migrations destructives

```text
DropColumn
DropTable
```

peuvent entraîner une perte de données.

---

# 37. Workflow classique

En développement :

```text
1. Modifier le modèle C#
        |
        v
2. Créer une migration
        |
        v
3. Lire la migration
        |
        v
4. Appliquer la migration
        |
        v
5. Tester
```

Exemple :

```bash
dotnet ef migrations add AddHotelStars
```

puis :

```bash
dotnet ef database update
```

---

# 38. Workflow mental complet

Imagine :

```text
V1
 |
 | ajout de Stars
 v
Modèle C# V2
 |
 | migrations add
 v
Migration V2
 |
 | database update
 v
Database V2
```

Puis :

```text
V2
 |
 | ajout de Rooms
 v
Modèle C# V3
 |
 | migrations add
 v
Migration V3
 |
 | database update
 v
Database V3
```

Les migrations sont donc une sorte de :

```text
historique versionné du schéma
```

---

# 39. Règle mentale

Quand tu modifies une entité C# :

```text
Je modifie le modèle.
```

Quand tu fais :

```bash
dotnet ef migrations add ...
```

pense :

> « Je décris le changement de schéma. »

Quand tu fais :

```bash
dotnet ef database update
```

pense :

> « J'applique le changement à la base. »

Quand tu regardes :

```csharp
Up()
```

pense :

> « Comment le schéma avance ? »

Quand tu regardes :

```csharp
Down()
```

pense :

> « Comment revenir en arrière ? »

Quand tu vois :

```text
__EFMigrationsHistory
```

pense :

> « Quelles migrations cette base connaît-elle déjà ? »

---

# 40. À retenir

1. Une migration représente une évolution du schéma.
2. `migrations add` crée une migration.
3. `database update` applique les migrations.
4. `Up()` applique l'évolution.
5. `Down()` représente le retour en arrière.
6. Les migrations doivent généralement être versionnées avec le code.
7. EF Core conserve un historique des migrations appliquées.
8. Le `ModelSnapshot` aide EF Core à comparer le modèle.
9. Il faut toujours lire les migrations générées.
10. Les changements destructifs doivent être traités avec prudence.
11. Les données existantes doivent être prises en compte lors des changements de schéma.
12. Migration et seeding sont deux concepts différents.
13. `EnsureCreated()` n'est pas un remplacement général des migrations.
14. Les migrations peuvent être intégrées à une stratégie CI/CD.
15. Les migrations de production doivent être déployées avec une stratégie contrôlée.
16. Une migration est une évolution versionnée, pas une sauvegarde de base.
17. Il faut toujours comprendre ce que la migration fera réellement à la base.

---

# Questions d'entretien

### 1. Qu'est-ce qu'une migration EF Core ?

Une migration est une représentation versionnée d'un changement du schéma de base de données correspondant à l'évolution du modèle EF Core.

### 2. Quelle différence entre `migrations add` et `database update` ?

```text
migrations add
    -> crée la migration

database update
    -> applique les migrations à la base
```

### 3. À quoi servent `Up()` et `Down()` ?

`Up()` applique la migration et `Down()` représente le retour à l'état précédent.

### 4. Pourquoi versionner les migrations avec Git ?

Parce qu'elles font partie de l'historique reproductible de l'évolution du schéma du projet.

### 5. Pourquoi lire une migration avant de l'appliquer ?

Pour vérifier les opérations réelles : création, suppression, modification de colonnes, clés étrangères, indexes et éventuelles opérations destructives.

### 6. Qu'est-ce que `__EFMigrationsHistory` ?

C'est la table utilisée par EF Core pour conserver l'information sur les migrations appliquées à la base.

### 7. Quelle différence entre migration et seeding ?

Une migration fait principalement évoluer le schéma ; le seeding initialise ou ajoute des données.

### 8. Pourquoi `EnsureCreated()` est différent des migrations ?

`EnsureCreated()` crée directement le schéma sans gérer son évolution versionnée comme le système de migrations.

### 9. Que peut-il se passer si on ajoute une colonne obligatoire sur une table déjà remplie ?

Il faut prévoir comment les anciennes lignes vont recevoir une valeur valide, par exemple via une valeur par défaut ou une stratégie de migration appropriée.

### 10. Pourquoi une migration peut-elle être dangereuse ?

Parce qu'elle peut modifier ou supprimer des structures et potentiellement des données, notamment avec des opérations destructives.

---

# Phrase à retenir

> **Une migration est une version de l'évolution du schéma : `migrations add` prépare le changement, `database update` l'applique, et `Up()`/`Down()` décrivent respectivement l'aller et le retour.**
