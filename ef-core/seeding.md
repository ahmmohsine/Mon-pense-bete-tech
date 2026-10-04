# EF Core — Seeding

## 1. Qu'est-ce que le seeding ?

Le **seeding** consiste à initialiser une base de données avec des données dont l'application a besoin.

Exemples :

- catégories par défaut ;
- pays ;
- statuts ;
- types d'utilisateurs ;
- rôles ;
- données de référence nécessaires au fonctionnement de l'application.

Le seeding concerne donc principalement **les données**, alors que les migrations concernent principalement **l'évolution du schéma** de la base.

Mentalement :

```text
Migration
    → fait évoluer la structure de la base

Seeding
    → initialise certaines données
```

---

# 2. Seeding ≠ Migration

Les deux notions sont liées, mais elles ne font pas la même chose.

| Migration | Seeding |
|---|---|
| Évolue le schéma | Initialise des données |
| Crée une table | Peut insérer des lignes |
| Ajoute une colonne | Peut créer des données de référence |
| Modifie une contrainte | Peut mettre à jour des données prévues |
| Historique des migrations | Données initiales |

Exemple :

```text
Migration :
CREATE TABLE Categories (...)

Seeding :
INSERT INTO Categories (...)
VALUES (...)
```

Une migration répond à :

> « Comment faire passer ma base de l'ancien modèle au nouveau ? »

Le seeding répond à :

> « Quelles données doivent déjà être présentes pour que mon application fonctionne ? »

---

# 3. `HasData` : le model-managed data

EF Core permet de définir des données initiales avec `HasData()` dans `OnModelCreating`.

Exemple :

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Category>().HasData(
        new Category
        {
            Id = 1,
            Name = "Informatique"
        },
        new Category
        {
            Id = 2,
            Name = "Maison"
        }
    );
}
```

Puis :

```bash
dotnet ef migrations add SeedCategories
dotnet ef database update
```

EF Core peut intégrer ces données dans la migration.

Le principe devient :

```text
HasData()
   ↓
modèle EF Core
   ↓
migration
   ↓
INSERT / UPDATE / DELETE
   ↓
base de données
```

---

# 4. Pourquoi la clé primaire doit être explicite ?

Avec `HasData`, il faut généralement fournir explicitement la clé primaire.

Exemple :

```csharp
new Category
{
    Id = 1,
    Name = "Informatique"
}
```

Pourquoi ?

Parce qu'EF Core doit pouvoir identifier précisément la ligne seedée.

Il doit savoir :

```text
Id = 1
```

correspond à :

```text
Category #1
```

pour pouvoir déterminer ensuite si cette donnée doit être ajoutée, modifiée ou supprimée.

---

# 5. Attention aux valeurs dynamiques

Avec `HasData`, évite les valeurs qui changent à chaque exécution.

Mauvais exemple :

```csharp
new Product
{
    Id = 1,
    Name = "Produit",
    CreatedAt = DateTime.UtcNow
}
```

Pourquoi ?

Parce que :

```csharp
DateTime.UtcNow
```

produit une valeur différente dans le temps.

Même problème avec des valeurs générées aléatoirement :

```csharp
Guid.NewGuid()
```

Pour des données gérées par le modèle, les valeurs doivent être suffisamment **déterministes**.

Préférer par exemple :

```csharp
CreatedAt = new DateTime(2026, 1, 1)
```

ou ne pas seed cette propriété si elle doit être générée dynamiquement.

---

# 6. Quand utiliser `HasData` ?

`HasData` est particulièrement adapté aux données de référence.

Exemples :

```text
Country
Category
Status
Currency
ProductType
```

Exemple :

```csharp
modelBuilder.Entity<OrderStatus>().HasData(
    new OrderStatus { Id = 1, Name = "Pending" },
    new OrderStatus { Id = 2, Name = "Paid" },
    new OrderStatus { Id = 3, Name = "Cancelled" }
);
```

Ces données sont :

- connues à l'avance ;
- relativement stables ;
- nécessaires à l'application ;
- facilement identifiables par une clé.

---

# 7. Quand `HasData` n'est pas le bon outil ?

`HasData` devient moins adapté lorsque l'initialisation nécessite de la logique.

Par exemple :

```text
Créer un utilisateur Identity
Hasher un mot de passe
Vérifier si un utilisateur existe
Appeler un service
Lire une configuration
Créer des données conditionnellement
```

Dans ce genre de situation, une initialisation applicative est souvent plus appropriée.

---

# 8. Seeding applicatif

On peut initialiser les données depuis le code de l'application.

Le principe est :

```text
Application démarre
       ↓
récupère un scope DI
       ↓
récupère les services nécessaires
       ↓
vérifie si les données existent
       ↓
crée ce qui manque
```

Exemple simplifié :

```csharp
using var scope = app.Services.CreateScope();

var context = scope.ServiceProvider
    .GetRequiredService<AppDbContext>();

if (!context.Categories.Any())
{
    context.Categories.AddRange(
        new Category { Name = "Informatique" },
        new Category { Name = "Maison" }
    );

    await context.SaveChangesAsync();
}
```

L'idée importante est que le seeding devient alors du **code d'application**, et non simplement une donnée déclarée dans le modèle EF Core.

---

# 9. Le seeding doit être idempotent

Une bonne initialisation doit pouvoir être exécutée plusieurs fois sans créer de doublons.

On appelle cela l'**idempotence**.

Mauvais principe :

```csharp
context.Categories.Add(
    new Category { Name = "Informatique" }
);

await context.SaveChangesAsync();
```

Si le code est exécuté à chaque démarrage, il peut créer plusieurs fois la même catégorie.

Meilleur principe :

```csharp
if (!context.Categories.Any(c => c.Name == "Informatique"))
{
    context.Categories.Add(
        new Category { Name = "Informatique" }
    );

    await context.SaveChangesAsync();
}
```

Mentalement :

```text
Seed
 ↓
Existe déjà ?
 ├── Oui → ne rien faire
 └── Non → créer
```

---

# 10. Seeding et ASP.NET Core Identity

Identity est un excellent exemple où `HasData` n'est généralement pas le meilleur choix pour créer des utilisateurs.

Un utilisateur Identity possède notamment :

```text
Username
Email
PasswordHash
Claims
Roles
SecurityStamp
...
```

Le mot de passe ne doit surtout pas être stocké en clair.

Au lieu de construire soi-même un hash de mot de passe, on utilise les services Identity.

Exemple :

```csharp
var userManager =
    scope.ServiceProvider.GetRequiredService<UserManager<ApplicationUser>>();

var user = await userManager.FindByEmailAsync("admin@example.com");

if (user is null)
{
    user = new ApplicationUser
    {
        UserName = "admin@example.com",
        Email = "admin@example.com"
    };

    await userManager.CreateAsync(user, "MotDePasseFort!");
}
```

`UserManager` se charge alors du mécanisme Identity approprié, notamment du hash du mot de passe.

Même principe pour les rôles :

```csharp
var roleManager =
    scope.ServiceProvider.GetRequiredService<RoleManager<IdentityRole>>();

if (!await roleManager.RoleExistsAsync("Admin"))
{
    await roleManager.CreateAsync(
        new IdentityRole("Admin")
    );
}
```

---

# 11. Pourquoi éviter un mot de passe hashé en dur ?

Un utilisateur Identity n'est pas simplement une ligne :

```text
Email + PasswordHash
```

La création d'un utilisateur passe par les mécanismes d'Identity.

Cela permet notamment de respecter la configuration de sécurité définie par Identity.

Donc :

```text
Application
   ↓
UserManager
   ↓
Identity
   ↓
hachage + validation + persistance
```

plutôt que :

```text
Application
   ↓
INSERT manuel d'un PasswordHash
```

---

# 12. `UseSeeding` et `UseAsyncSeeding`

Dans les versions modernes d'EF Core, EF Core propose également des mécanismes de seeding configurés avec :

```csharp
UseSeeding(...)
```

et :

```csharp
UseAsyncSeeding(...)
```

Ils permettent de définir une logique de seeding plus flexible que `HasData`.

Exemple conceptuel :

```csharp
optionsBuilder.UseSeeding((context, _) =>
{
    if (!context.Set<Category>().Any())
    {
        context.Set<Category>().Add(
            new Category
            {
                Name = "Informatique"
            });

        context.SaveChanges();
    }
});
```

Version asynchrone :

```csharp
optionsBuilder.UseAsyncSeeding(async (context, _, cancellationToken) =>
{
    if (!await context.Set<Category>().AnyAsync(cancellationToken))
    {
        context.Set<Category>().Add(
            new Category
            {
                Name = "Informatique"
            });

        await context.SaveChangesAsync(cancellationToken);
    }
});
```

L'intérêt principal est de pouvoir effectuer une logique d'initialisation plus riche tout en restant intégrée au mécanisme d'initialisation/migration d'EF Core.

Pour les projets qui utilisent cette API, il faut toujours vérifier la documentation de la version exacte d'EF Core utilisée.

---

# 13. `HasData` vs seeding applicatif

| Besoin | Solution adaptée |
|---|---|
| Quelques données statiques | `HasData` |
| Données de référence | `HasData` |
| Données nécessitant de la logique | Seeding applicatif / mécanisme de seeding |
| Création d'un utilisateur Identity | `UserManager` |
| Création de rôles Identity | `RoleManager` |
| Données de test | Stratégie de test dédiée |
| Données métier créées par les utilisateurs | Pas du seeding classique |

Le point important :

> Choisis ton mécanisme de seeding selon la nature des données.

---

# 14. Données de référence vs données métier

Une bonne question à se poser est :

> « Est-ce que cette donnée fait partie de la configuration initiale de l'application ou de son activité ? »

### Données de référence

Exemples :

```text
Belgique
France
Allemagne

Pending
Paid
Cancelled

Admin
Manager
User
```

Elles peuvent être de bonnes candidates pour le seeding.

### Données métier

Exemples :

```text
Client
Commande
Facture
Paiement
Réservation
```

Ces données sont normalement créées pendant la vie de l'application.

Il ne faut donc pas remplir la base de production avec de fausses commandes simplement parce qu'on sait faire du seeding.

---

# 15. Seeding et environnements

Attention au seeding différent selon :

```text
Development
Test
Production
```

En développement, on peut vouloir :

```text
Admin
Users de test
Produits de démonstration
Données de test
```

En production, on peut seulement vouloir :

```text
Roles
Countries
Statuses
Configuration de référence
```

Il faut éviter de mettre automatiquement des données de test en production.

Exemple :

```csharp
if (app.Environment.IsDevelopment())
{
    // Données de développement uniquement
}
```

---

# 16. Attention au démarrage de l'application

Un seeding exécuté au démarrage peut avoir un impact sur :

```text
temps de démarrage
déploiement
scaling
concurrence
permissions
```

Imagine plusieurs instances de ton API :

```text
Instance A ──┐
             ├──> Database
Instance B ──┘
```

Les deux instances peuvent démarrer presque simultanément.

Ton seeding doit donc être conçu avec suffisamment de précautions pour éviter :

- doublons ;
- courses concurrentes ;
- erreurs au démarrage ;
- dépendance à des permissions excessives.

---

# 17. Ne pas confondre `EnsureCreated()` et migrations

On rencontre souvent :

```csharp
context.Database.EnsureCreated();
```

Cela peut créer une base selon le modèle actuel, mais ce n'est pas une stratégie de migration classique.

Pour une application utilisant EF Core Migrations, on travaille généralement avec :

```bash
dotnet ef migrations add InitialCreate
```

puis :

```bash
dotnet ef database update
```

Mentalement :

```text
EnsureCreated
    → création rapide d'une base basée sur le modèle

Migrations
    → historique et évolution contrôlée du schéma
```

Pour une application qui doit évoluer dans le temps, les migrations sont généralement le mécanisme approprié.

---

# 18. Seeding et transactions

Lorsque plusieurs opérations doivent être réalisées ensemble, il faut réfléchir à leur atomicité.

Exemple :

```text
Créer le rôle Admin
        +
Créer l'utilisateur Admin
        +
Attribuer le rôle Admin
```

Si une opération échoue au milieu, il faut éviter de laisser la base dans un état incohérent.

Selon le scénario, une transaction peut être pertinente :

```csharp
await using var transaction =
    await context.Database.BeginTransactionAsync();

try
{
    // opérations de seeding

    await context.SaveChangesAsync();

    await transaction.CommitAsync();
}
catch
{
    await transaction.RollbackAsync();
    throw;
}
```

Attention toutefois : Identity utilise ses propres services et mécanismes de persistance. La stratégie transactionnelle doit donc être pensée avec l'architecture réelle de l'application.

---

# 19. Seeding de test ≠ Seeding de production

Pour les tests automatisés, on peut avoir besoin de beaucoup de données.

Exemple :

```text
100 users
500 products
2000 orders
```

Ce n'est pas forcément du seeding de production.

On peut utiliser :

- fixtures ;
- builders ;
- factories ;
- données générées ;
- bases de test dédiées.

Mentalement :

```text
Production
    → données nécessaires au fonctionnement

Tests
    → données nécessaires à la vérification du code
```

---

# 20. Les erreurs classiques

### Erreur 1 — utiliser `HasData` pour tout

```text
Tout mettre dans HasData
```

Ce n'est pas idéal si les données nécessitent une logique complexe.

---

### Erreur 2 — utiliser des valeurs dynamiques

```csharp
DateTime.UtcNow
Guid.NewGuid()
Random.Shared.Next()
```

dans des données gérées par le modèle peut provoquer des changements inattendus dans les migrations.

---

### Erreur 3 — créer plusieurs fois les mêmes données

```csharp
context.Add(new Category { Name = "Admin" });
```

à chaque démarrage sans vérifier l'existence.

---

### Erreur 4 — mettre des mots de passe en clair

Ne jamais faire :

```csharp
Password = "123456"
```

et considérer cela comme une solution de sécurité.

Avec Identity, utiliser :

```csharp
UserManager.CreateAsync(user, password)
```

---

### Erreur 5 — mettre des données de test en production

Exemple :

```text
admin@test.com
john@test.com
Fake Product
Test Order
```

dans la base de production.

---

### Erreur 6 — confondre seeding et migration

```text
Migration = évolution du schéma
Seeding = initialisation des données
```

Les deux peuvent fonctionner ensemble, mais ce sont deux concepts différents.

---

# 21. Exemple complet : catégories de référence

Entité :

```csharp
public class Category
{
    public int Id { get; set; }

    public string Name { get; set; } = null!;
}
```

Configuration :

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Category>().HasData(
        new Category
        {
            Id = 1,
            Name = "Informatique"
        },
        new Category
        {
            Id = 2,
            Name = "Maison"
        },
        new Category
        {
            Id = 3,
            Name = "Jardin"
        });
}
```

Migration :

```bash
dotnet ef migrations add SeedCategories
```

Application :

```bash
dotnet ef database update
```

Résultat conceptuel :

```text
Code
 ↓
HasData
 ↓
Migration
 ↓
Database
 ↓
Categories
 ├── 1 Informatique
 ├── 2 Maison
 └── 3 Jardin
```

---

# 22. Comment choisir la bonne approche ?

Pose-toi ces questions :

```text
1. Les données sont-elles connues à l'avance ?
        ↓
       Oui
        ↓
2. Sont-elles simples et déterministes ?
        ↓
       Oui
        ↓
     HasData
```

Sinon :

```text
Données nécessitant de la logique
        ↓
Seeding applicatif / UseSeeding
```

Et pour Identity :

```text
Utilisateur
    ↓
UserManager

Rôle
    ↓
RoleManager
```

---

# 23. Architecture mentale

Dans une application .NET + EF Core :

```text
                 Application
                      |
                      v
             Initialisation
                      |
          +-----------+-----------+
          |                       |
          v                       v
      Migration                Seeding
          |                       |
          v                       v
   Schéma database         Données initiales
                                  |
                    +-------------+-------------+
                    |             |             |
                 Categories     Roles         Users
                                  |
                              Identity
                                  |
                         UserManager / RoleManager
```

Cette séparation permet de comprendre rapidement où placer chaque responsabilité.

---

# 24. Checklist

Avant de mettre un seeding en place :

- [ ] Les données sont-elles réellement nécessaires au démarrage ?
- [ ] S'agit-il de données de référence ou de données métier ?
- [ ] Les valeurs sont-elles déterministes ?
- [ ] Les clés sont-elles correctement définies ?
- [ ] Le seeding est-il idempotent ?
- [ ] Les doublons sont-ils impossibles ?
- [ ] Les données de test sont-elles séparées de la production ?
- [ ] Les utilisateurs Identity passent-ils par `UserManager` ?
- [ ] Les rôles Identity passent-ils par `RoleManager` ?
- [ ] Les migrations sont-elles utilisées pour l'évolution du schéma ?
- [ ] Le seeding fonctionne-t-il correctement avec plusieurs instances ?
- [ ] Les permissions nécessaires sont-elles acceptables en production ?

---

# 25. À retenir

Le seeding sert à **initialiser des données**, pas à faire évoluer le schéma de la base.

Les trois idées principales :

```text
HasData
    → données statiques et déterministes

Seeding applicatif / UseSeeding
    → logique d'initialisation plus flexible

Identity
    → UserManager / RoleManager
```

Et surtout :

```text
Migration = structure
Seeding   = données
```

---

# Questions d'entretien

### 1. Quelle est la différence entre migration et seeding ?

**Réponse :**

Une migration fait évoluer le schéma de la base de données. Le seeding initialise ou maintient certaines données nécessaires à l'application.

---

### 2. Quand utiliser `HasData` ?

Pour des données statiques, connues à l'avance et déterministes, comme des catégories, statuts ou pays.

---

### 3. Pourquoi éviter `DateTime.UtcNow` dans `HasData` ?

Parce que la valeur change dans le temps. Cela peut entraîner des différences dans le modèle et des migrations inattendues.

---

### 4. Comment créer un utilisateur Identity lors du seeding ?

Avec `UserManager`, notamment :

```csharp
await userManager.CreateAsync(user, password);
```

plutôt que de manipuler directement le hash du mot de passe.

---

### 5. Pourquoi le seeding doit-il être idempotent ?

Parce qu'il peut être exécuté plusieurs fois. Il doit donc éviter de créer plusieurs fois les mêmes données.

---

### 6. Quelle est la différence entre données de référence et données métier ?

Les données de référence sont généralement stables et nécessaires au fonctionnement de l'application. Les données métier sont produites pendant l'utilisation de l'application.

---

### 7. Pourquoi `EnsureCreated()` n'est-il pas équivalent aux migrations ?

`EnsureCreated()` sert principalement à créer une base correspondant au modèle sans gérer l'historique d'évolution du schéma comme les migrations.

---

# Phrase à mémoriser

> **Migration = faire évoluer la structure ; Seeding = mettre les bonnes données au démarrage.**
