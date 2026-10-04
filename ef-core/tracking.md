# EF Core — Tracking

Le **tracking** est le mécanisme par lequel EF Core garde une trace des entités qu'un `DbContext` suit.

C'est grâce à ce mécanisme qu'EF Core peut notamment savoir :

```text
Quelle entité ?
Quel état ?
Quelles propriétés ont changé ?
Quelle opération SQL faut-il produire ?
```

Le concept central est le **Change Tracker**.

---

# 1. Le problème que le Tracking résout

Imagine :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == 1);

hotel.Name = "Hotel Central";

await context.SaveChangesAsync();
```

Tu n'as jamais écrit :

```sql
UPDATE Hotels
SET Name = 'Hotel Central'
WHERE Id = 1;
```

Alors comment EF Core sait-il qu'il doit faire un `UPDATE` ?

Grâce notamment au **Change Tracker**.

Mentalement :

```text
SELECT
  |
  v
Hotel récupéré
  |
  v
DbContext commence à le suivre
  |
  v
Modification en C#
  |
  v
EF Core détecte la modification
  |
  v
SaveChangesAsync()
  |
  v
UPDATE SQL
```

---

# 2. Qu'est-ce qu'une entité suivie ?

Une entité est **tracked** lorsqu'elle est enregistrée dans le Change Tracker du `DbContext`.

Exemple :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == 1);
```

Dans une requête de tracking classique :

```text
Database
   |
   v
Hotel
   |
   v
Change Tracker
   |
   v
DbContext
```

Le contexte garde alors des informations permettant de gérer l'entité.

---

# 3. Les états d'une entité

EF Core utilise plusieurs états principaux :

```text
Detached
Unchanged
Added
Modified
Deleted
```

Ils sont représentés par :

```csharp
EntityState.Detached
EntityState.Unchanged
EntityState.Added
EntityState.Modified
EntityState.Deleted
```

---

# 4. `Detached`

Une entité `Detached` n'est pas suivie par le `DbContext`.

Exemple :

```csharp
var hotel = new Hotel
{
    Id = 1,
    Name = "Hotel Central"
};
```

À ce moment-là, l'objet existe simplement en mémoire.

Il n'est pas automatiquement suivi par ton contexte.

Mentalement :

```text
Hotel
 |
 X
DbContext
```

---

# 5. `Unchanged`

Une entité peut être suivie sans avoir de modification détectée.

Exemple :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == 1);
```

Après son chargement :

```text
Hotel
  |
  v
Tracked
  |
  v
Unchanged
```

Cela signifie essentiellement :

> « Cette entité est suivie et EF Core ne voit actuellement aucune modification à persister. »

---

# 6. `Modified`

Supposons :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == 1);

hotel.Name = "New Name";
```

L'entité peut passer de :

```text
Unchanged
     |
     | modification
     v
Modified
```

Puis :

```csharp
await context.SaveChangesAsync();
```

permet à EF Core de persister le changement.

---

# 7. `Added`

Pour une nouvelle entité :

```csharp
var hotel = new Hotel
{
    Name = "New Hotel"
};

context.Hotels.Add(hotel);
```

L'état devient :

```text
Added
```

Puis :

```csharp
await context.SaveChangesAsync();
```

peut produire :

```sql
INSERT INTO Hotels ...
```

Mentalement :

```text
new Hotel
   |
   v
Add()
   |
   v
Added
   |
   v
SaveChangesAsync()
   |
   v
INSERT
```

---

# 8. `Deleted`

Avec :

```csharp
context.Hotels.Remove(hotel);
```

l'entité est marquée :

```text
Deleted
```

Puis :

```csharp
await context.SaveChangesAsync();
```

peut produire :

```sql
DELETE FROM Hotels
WHERE Id = ...;
```

---

# 9. Voir l'état d'une entité

Tu peux inspecter l'état :

```csharp
var entry = context.Entry(hotel);

Console.WriteLine(entry.State);
```

Par exemple :

```text
Modified
```

Tu peux également utiliser :

```csharp
var state = context.Entry(hotel).State;
```

---

# 10. Voir les entités suivies

Le contexte permet d'inspecter les entités suivies :

```csharp
var entries = context.ChangeTracker.Entries();
```

Tu peux par exemple faire :

```csharp
foreach (var entry in context.ChangeTracker.Entries())
{
    Console.WriteLine(
        $"{entry.Entity.GetType().Name} - {entry.State}");
}
```

Cela peut donner :

```text
Hotel - Unchanged
Room - Added
User - Modified
```

---

# 11. Comment EF Core détecte une modification ?

Conceptuellement, EF Core garde suffisamment d'informations sur l'état initial de l'entité pour pouvoir détecter des changements.

Exemple :

```text
Valeur initiale
Name = "Hotel A"

        |
        v

Code C#
hotel.Name = "Hotel B"

        |
        v

Valeur actuelle
Name = "Hotel B"
```

EF Core peut constater :

```text
Original: Hotel A
Current : Hotel B
```

Donc :

```text
Name modified
```

Il peut ensuite préparer l'opération de mise à jour.

---

# 12. `DetectChanges`

EF Core dispose d'un mécanisme de détection des changements appelé :

```text
DetectChanges
```

Il permet notamment de détecter les modifications apportées aux propriétés des entités suivies.

Il est utilisé automatiquement dans différents moments importants du fonctionnement d'EF Core.

Dans le code courant, tu n'as généralement pas besoin de l'appeler manuellement.

Exemple conceptuel :

```csharp
hotel.Name = "New Name";

await context.SaveChangesAsync();
```

Le développeur n'a normalement pas besoin de faire :

```csharp
context.ChangeTracker.DetectChanges();
```

lui-même.

---

# 13. `SaveChanges` et Change Tracker

`SaveChangesAsync()` est fortement lié au Change Tracker.

Mentalement :

```text
Change Tracker
      |
      +-- Added
      +-- Modified
      +-- Deleted
      |
      v
SaveChangesAsync()
      |
      v
SQL
```

EF Core utilise les informations de suivi pour déterminer les opérations à effectuer.

---

# 14. Pourquoi le Tracking est utile

Le tracking permet notamment ce style de code :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == id);

hotel.Name = "New Name";

await context.SaveChangesAsync();
```

Tu n'as pas besoin d'écrire explicitement :

```csharp
context.Entry(hotel).State =
    EntityState.Modified;
```

dans le cas normal où l'entité est déjà suivie.

C'est une des grandes commodités d'EF Core.

---

# 15. `AsNoTracking()`

Pour certaines lectures, le tracking est inutile.

Exemple :

```csharp
var hotels = await context.Hotels
    .AsNoTracking()
    .ToListAsync();
```

Ici, EF Core demande une lecture sans suivi des entités.

Mentalement :

```text
Database
   |
   v
Hotel
   |
   X
Change Tracker
```

Cela peut réduire le travail effectué par EF Core pour des lectures qui n'ont pas besoin d'être modifiées puis sauvegardées avec le même contexte.

---

# 16. Quand utiliser `AsNoTracking()` ?

Très souvent pour des scénarios comme :

```text
GET /api/hotels
GET /api/products
GET /api/categories
```

lorsque tu veux simplement lire et retourner les données.

Exemple :

```csharp
var hotels = await context.Hotels
    .AsNoTracking()
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .ToListAsync();
```

Ici :

```text
Lecture
+
Projection
+
Pas de modification de l'entité
```

Le tracking n'est généralement pas nécessaire.

---

# 17. Tracking pour une modification

Si tu veux charger une entité, la modifier et la sauvegarder :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == id);

hotel.Name = "New Name";

await context.SaveChangesAsync();
```

Le tracking est utile :

```text
SELECT
  |
  v
Tracked entity
  |
  v
Modification
  |
  v
SaveChanges
  |
  v
UPDATE
```

---

# 18. Attention : `AsNoTracking()` puis modification

Supposons :

```csharp
var hotel = await context.Hotels
    .AsNoTracking()
    .FirstAsync(h => h.Id == id);

hotel.Name = "New Name";

await context.SaveChangesAsync();
```

Le problème est que :

```csharp
hotel
```

n'est pas suivi par le contexte.

Donc modifier l'objet en mémoire ne suffit pas à demander automatiquement un `UPDATE`.

Il faut alors explicitement rattacher ou marquer l'entité selon la stratégie utilisée.

Exemple :

```csharp
context.Hotels.Update(hotel);

await context.SaveChangesAsync();
```

Mais attention : `Update()` peut marquer une grande partie de l'entité comme modifiée.

Il ne faut donc pas utiliser `Update()` aveuglément.

---

# 19. `Update()` n'est pas la même chose que la détection normale

Avec une entité suivie :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == id);

hotel.Name = "New Name";

await context.SaveChangesAsync();
```

EF Core peut détecter les propriétés modifiées.

Avec :

```csharp
context.Hotels.Update(hotel);
```

tu indiques explicitement à EF Core que l'entité doit être considérée comme modifiée.

Cela peut avoir des conséquences sur les colonnes envoyées dans l'`UPDATE`.

Mentalement :

```text
Tracking normal
    =
EF Core observe les changements

Update()
    =
Tu donnes explicitement un état de modification
```

---

# 20. Pourquoi `Update()` peut être dangereux avec des DTO

Imaginons un DTO :

```csharp
public class UpdateHotelDto
{
    public string Name { get; set; } = string.Empty;
}
```

Et :

```csharp
var hotel = new Hotel
{
    Id = id,
    Name = dto.Name
};

context.Hotels.Update(hotel);

await context.SaveChangesAsync();
```

Tu risques de traiter l'entité comme entièrement modifiée alors que le client n'a peut-être fourni qu'une partie des informations.

Une autre stratégie consiste à charger l'entité suivie :

```csharp
var hotel = await context.Hotels
    .FirstOrDefaultAsync(h => h.Id == id);

if (hotel is null)
{
    return;
}

hotel.Name = dto.Name;

await context.SaveChangesAsync();
```

Mentalement :

```text
DTO
 |
 v
Entity tracked
 |
 v
Modification ciblée
 |
 v
SaveChanges
```

---

# 21. Tracking et DTO

Le tracking concerne principalement les **entités EF Core**, pas les DTO utilisés pour transporter les données.

Exemple :

```csharp
var result = await context.Hotels
    .Select(h => new HotelDto
    {
        Id = h.Id,
        Name = h.Name
    })
    .ToListAsync();
```

Le résultat est un DTO.

Le but est généralement :

```text
Database
   |
   v
EF Core
   |
   v
DTO
   |
   v
API Response
```

Le DTO n'a pas besoin d'être traité comme une entité persistée.

---

# 22. Tracking et projection

Une projection :

```csharp
.Select(h => new HotelDto
{
    Id = h.Id,
    Name = h.Name
})
```

est souvent un excellent choix pour les lectures API.

Elle permet notamment de limiter les données récupérées.

Le principe mental :

```text
Entity complète
       |
       | Select
       v
Données nécessaires
       |
       v
DTO
```

---

# 23. Tracking et navigation properties

Les relations peuvent également être suivies.

Exemple :

```csharp
var hotel = await context.Hotels
    .Include(h => h.Rooms)
    .FirstAsync(h => h.Id == id);
```

EF Core peut suivre :

```text
Hotel
 |
 +-- Room
 +-- Room
 +-- Room
```

Les changements apportés aux entités suivies peuvent ensuite être pris en compte par le Change Tracker.

---

# 24. Plusieurs entités avec la même clé

Le `DbContext` cherche à maintenir une identité cohérente pour les entités qu'il suit.

Un contexte ne doit normalement pas suivre simultanément deux instances différentes représentant la même ligne avec la même clé.

Exemple conceptuel problématique :

```text
Hotel Id = 1
     |
     +-- instance A

Hotel Id = 1
     |
     +-- instance B
```

Cela peut provoquer une erreur du type :

```text
The instance of entity type 'Hotel'
cannot be tracked because another instance
with the same key value is already being tracked.
```

---

# 25. Pourquoi cette erreur arrive ?

Exemple :

```csharp
var hotel1 = await context.Hotels
    .FindAsync(1);

var hotel2 = new Hotel
{
    Id = 1,
    Name = "Another object"
};

context.Hotels.Update(hotel2);
```

Le contexte suit déjà :

```text
Hotel Id = 1
```

et on lui donne une autre instance :

```text
Hotel Id = 1
```

Il ne sait pas simplement remplacer l'une par l'autre.

Il signale donc le conflit.

---

# 26. Solution générale

Une approche courante consiste à travailler avec l'entité déjà suivie :

```csharp
var hotel = await context.Hotels
    .FindAsync(id);

if (hotel is null)
{
    return;
}

hotel.Name = dto.Name;

await context.SaveChangesAsync();
```

Mentalement :

```text
1 ligne DB
      |
      v
1 instance suivie
      |
      v
modifications
      |
      v
SaveChanges
```

---

# 27. Tracking et durée de vie du DbContext

Le tracking explique aussi pourquoi un `DbContext` ne doit pas vivre trop longtemps.

Plus le contexte reste vivant et plus il suit potentiellement d'entités :

```text
DbContext
 |
 +-- Hotel
 +-- Room
 +-- User
 +-- Booking
 +-- ...
```

Cela peut augmenter la mémoire utilisée et le travail du Change Tracker.

Dans une API ASP.NET Core, le modèle Scoped est donc particulièrement adapté à l'unité de travail d'une requête.

---

# 28. `ChangeTracker.Clear()`

Dans certains scénarios spécifiques, on peut demander au contexte de ne plus suivre les entités actuellement suivies :

```csharp
context.ChangeTracker.Clear();
```

Cela détache les entités suivies par ce contexte.

Mais ce n'est généralement pas une solution à utiliser pour masquer un mauvais design de durée de vie du `DbContext`.

La meilleure solution est souvent de corriger la portée du contexte.

---

# 29. `AsNoTrackingWithIdentityResolution()`

EF Core propose également :

```csharp
.AsNoTrackingWithIdentityResolution()
```

C'est un comportement intermédiaire utile dans certains scénarios de lecture avec des relations.

L'idée est :

```text
Pas de tracking normal
        +
Réutilisation cohérente des mêmes instances
pour certaines entités répétées dans le résultat
```

C'est plus spécialisé que `AsNoTracking()`.

À retenir surtout :

```text
AsNoTracking()
    -> lecture sans tracking

AsNoTrackingWithIdentityResolution()
    -> lecture sans tracking persistant,
       mais avec résolution d'identité pendant la matérialisation
```

---

# 30. Tracking et performance

Le tracking apporte une fonctionnalité utile, mais il a un coût.

EF Core doit notamment gérer :

```text
Entités suivies
États
Valeurs originales
Détection des changements
Relations
```

Pour une grosse lecture pure, cela peut représenter du travail inutile.

Exemple :

```csharp
var hotels = await context.Hotels
    .AsNoTracking()
    .ToListAsync();
```

peut être pertinent si les entités ne seront pas modifiées dans le même contexte.

Mais il ne faut pas appliquer `AsNoTracking()` automatiquement à toutes les requêtes sans comprendre le besoin.

---

# 31. Tracking et lecture seule

Une bonne règle pratique :

```text
Je lis uniquement
        |
        v
AsNoTracking() peut être pertinent

Je lis puis je modifie
        |
        v
Tracking généralement utile
```

Ce n'est pas une loi absolue.

Il faut surtout comprendre **pourquoi** le tracking est nécessaire ou non.

---

# 32. Tracking et concurrence

Il faut distinguer deux notions :

```text
Tracking EF Core
        ≠
Concurrency control
```

Le Change Tracker sait qu'une entité a été modifiée.

Mais cela ne signifie pas automatiquement que deux utilisateurs ne peuvent pas modifier la même ligne simultanément.

Pour gérer certaines situations de concurrence optimiste, EF Core peut utiliser notamment des propriétés de concurrence et des mécanismes comme les tokens de concurrence.

Mentalement :

```text
Tracking
    =
"Qu'est-ce qui a changé ?"

Concurrency
    =
"Est-ce que quelqu'un d'autre a changé
la donnée entre-temps ?"
```

Cette distinction est importante.

---

# 33. Tracking vs Concurrency : exemple

Utilisateur A :

```text
Hotel Name = "Hotel A"
```

Utilisateur B lit également :

```text
Hotel Name = "Hotel A"
```

A modifie :

```text
Hotel A -> Hotel B
```

B modifie ensuite :

```text
Hotel A -> Hotel C
```

Le tracking seul ne répond pas entièrement à la question :

> « Est-ce que B doit écraser la modification de A ? »

C'est un problème de concurrence.

---

# 34. Erreurs fréquentes

## Erreur 1 : croire que toutes les requêtes sont trackées

Certaines requêtes peuvent explicitement utiliser :

```csharp
AsNoTracking()
```

et les projections ne se comportent pas exactement comme une requête d'entités suivies.

Il faut regarder le type de résultat et la manière dont la requête est construite.

---

## Erreur 2 : utiliser `Update()` partout

```csharp
context.Update(entity);
```

n'est pas nécessaire simplement parce qu'une entité a été modifiée.

Si elle est déjà suivie :

```csharp
entity.Name = "New Name";

await context.SaveChangesAsync();
```

suffit généralement.

---

## Erreur 3 : utiliser `AsNoTracking()` puis attendre un UPDATE automatique

Une entité non suivie ne sera pas automatiquement détectée comme modifiée par ce contexte.

---

## Erreur 4 : garder un DbContext pendant des heures

Cela augmente la quantité d'état suivie et peut provoquer des problèmes de performance et de conception.

---

## Erreur 5 : mélanger deux instances avec la même clé

Éviter :

```text
Instance A -> Id 1 -> tracked
Instance B -> Id 1 -> attach/update
```

sans comprendre le comportement du contexte.

---

# 35. Tableau des états

| État | Signification | Exemple |
|---|---|---|
| `Detached` | Non suivie | objet créé hors contexte |
| `Unchanged` | Suivie sans modification | entité juste chargée |
| `Added` | À insérer | `Add()` |
| `Modified` | À modifier | modification détectée / `Update()` |
| `Deleted` | À supprimer | `Remove()` |

---

# 36. Tableau des méthodes importantes

| Méthode | Rôle |
|---|---|
| `Add()` | Marque une entité comme `Added` |
| `AddAsync()` | Ajout avec API async, surtout utile dans certains cas spécifiques |
| `Update()` | Marque une entité comme modifiée |
| `Remove()` | Marque une entité comme `Deleted` |
| `Attach()` | Attache une entité sans la marquer normalement comme modifiée |
| `Entry()` | Accède aux informations de tracking d'une entité |
| `SaveChangesAsync()` | Persiste les changements |
| `AsNoTracking()` | Désactive le tracking pour une requête |
| `ChangeTracker.Entries()` | Inspecte les entités suivies |
| `ChangeTracker.Clear()` | Détache les entités actuellement suivies |

---

# 37. Mental model complet

Quand tu écris :

```csharp
var hotel = await context.Hotels
    .FirstAsync(h => h.Id == id);
```

pense :

```text
Database
   |
   v
EF Core
   |
   v
Hotel
   |
   v
Change Tracker
   |
   v
Unchanged
```

Puis :

```csharp
hotel.Name = "New Name";
```

pense :

```text
Unchanged
    |
    v
Modification
    |
    v
Modified
```

Puis :

```csharp
await context.SaveChangesAsync();
```

pense :

```text
Modified
    |
    v
SQL UPDATE
    |
    v
Database
```

---

# 38. À retenir

1. Le tracking est le mécanisme de suivi des entités par le `DbContext`.
2. Le Change Tracker conserve l'état des entités suivies.
3. Les principaux états sont `Detached`, `Unchanged`, `Added`, `Modified` et `Deleted`.
4. Une entité chargée normalement peut être suivie par le contexte.
5. Une modification d'une entité suivie peut être détectée automatiquement.
6. `SaveChangesAsync()` utilise ces informations pour persister les changements.
7. `AsNoTracking()` désactive le tracking pour une requête.
8. `AsNoTracking()` est particulièrement intéressant pour certaines lectures pures.
9. Une entité non suivie n'est pas automatiquement mise à jour simplement parce qu'elle a été modifiée en mémoire.
10. `Update()` force une stratégie de modification et ne doit pas être utilisé aveuglément.
11. `Attach()` permet notamment d'attacher une entité au contexte sans la traiter comme nouvellement ajoutée.
12. Deux instances représentant la même clé ne doivent normalement pas être suivies simultanément par le même contexte.
13. Un `DbContext` trop long peut accumuler beaucoup d'entités suivies.
14. `ChangeTracker.Clear()` peut détacher les entités, mais ne remplace pas une bonne gestion de la durée de vie.
15. Tracking et concurrence sont deux problèmes différents.
16. Tracking signifie essentiellement : « quelles entités et quels changements ce contexte doit-il gérer ? »

---

# Questions d'entretien

### 1. Qu'est-ce que le Change Tracker ?

C'est le mécanisme d'EF Core qui suit les entités et leur état afin de pouvoir détecter et persister les changements.

### 2. Quels sont les principaux états d'une entité ?

```text
Detached
Unchanged
Added
Modified
Deleted
```

### 3. Pourquoi `SaveChangesAsync()` sait-il qu'il doit faire un UPDATE ?

Parce que le `DbContext` suit l'entité et peut détecter que son état ou certaines de ses valeurs ont changé.

### 4. Quelle différence entre Tracking et `AsNoTracking()` ?

Le tracking conserve l'entité dans le Change Tracker ; `AsNoTracking()` demande une lecture sans suivi normal par le contexte.

### 5. Pourquoi utiliser `AsNoTracking()` ?

Pour les lectures qui ne nécessitent pas de modifier et sauvegarder les entités avec le même contexte, afin d'éviter le coût du tracking.

### 6. Pourquoi `Update()` peut-il être problématique ?

Parce qu'il peut considérer l'entité comme modifiée sans avoir la connaissance fine des propriétés réellement changées, ce qui est particulièrement important avec des objets partiellement remplis ou des DTO.

### 7. Pourquoi l'erreur « another instance with the same key value is already being tracked » apparaît-elle ?

Parce que le même `DbContext` suit déjà une instance pour une clé donnée et qu'on tente de lui faire suivre une autre instance représentant la même clé.

### 8. Quelle différence entre Tracking et Concurrency ?

Le tracking répond principalement à :

> « Qu'est-ce qui a changé dans les entités que je suis ? »

La concurrence répond à :

> « Est-ce qu'une autre opération a modifié les mêmes données entre-temps ? »

### 9. Pourquoi le DbContext ne doit-il pas être Singleton ?

Parce qu'il conserve un état de tracking et n'est pas conçu pour être partagé globalement ou utilisé simultanément par plusieurs opérations concurrentes.

### 10. Que fait `Attach()` ?

Il attache une entité au contexte, généralement sans la considérer comme nouvellement ajoutée. L'état exact dépend de l'entité et des opérations effectuées ensuite.

---

# Phrase à retenir

> **Le Tracking permet au DbContext de savoir quelles entités il connaît, dans quel état elles sont et quels changements doivent être persistés ; `AsNoTracking()` signifie que je lis sans demander à ce contexte de suivre ces entités.**
