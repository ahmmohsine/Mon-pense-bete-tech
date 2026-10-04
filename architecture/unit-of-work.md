# Architecture — Unit of Work

## 1. Qu'est-ce que le Unit of Work ?

Le **Unit of Work** représente une unité de travail au cours de laquelle plusieurs changements sont préparés puis validés ensemble.

L'idée centrale :

```text
Plusieurs modifications
        ↓
Une même unité de travail
        ↓
SaveChanges
        ↓
Persistance
```

Avec EF Core, cette notion est déjà largement représentée par :

```csharp
DbContext
```

C'est pourquoi il faut comprendre le pattern avant d'ajouter une abstraction supplémentaire.

---

# 2. Le problème que le Unit of Work cherche à résoudre

Imagine une création de commande :

```text
Créer Order
Créer OrderLines
Mettre à jour Stock
```

On veut généralement éviter :

```text
Créer Order       → sauvegardé
Créer OrderLines  → sauvegardé
Stock             → erreur
```

On pourrait se retrouver avec une base partiellement modifiée.

On préfère :

```text
Préparer Order
Préparer OrderLines
Préparer Stock
        ↓
SaveChanges
        ↓
Tout persister
```

L'idée est donc :

> **Regrouper plusieurs changements dans une même unité de travail.**

---

# 3. Le rôle du DbContext

Avec EF Core :

```csharp
_context.Orders.Add(order);
_context.OrderLines.AddRange(lines);
_context.Products.Update(product);

await _context.SaveChangesAsync();
```

Le `DbContext` suit les changements grâce au **Change Tracker**.

Il sait notamment que certaines entités sont :

```text
Added
Modified
Deleted
Unchanged
```

Puis :

```csharp
SaveChangesAsync()
```

demande à EF Core de persister les changements détectés.

C'est pourquoi `DbContext` est souvent décrit comme jouant un rôle de **Unit of Work**.

---

# 4. Unit of Work et Repository

On peut représenter une architecture classique ainsi :

```text
Application
    ↓
Repositories
    ↓
Unit of Work
    ↓
DbContext
    ↓
Database
```

Exemple :

```csharp
await _orderRepository.AddAsync(order);
await _stockRepository.UpdateAsync(stock);

await _unitOfWork.SaveChangesAsync(
    cancellationToken);
```

L'idée est que les deux repositories participent à la même unité de travail.

---

# 5. Exemple d'interface

Une abstraction possible :

```csharp
public interface IUnitOfWork
{
    Task<int> SaveChangesAsync(
        CancellationToken cancellationToken);
}
```

Implémentation :

```csharp
public class UnitOfWork : IUnitOfWork
{
    private readonly AppDbContext _context;

    public UnitOfWork(AppDbContext context)
    {
        _context = context;
    }

    public Task<int> SaveChangesAsync(
        CancellationToken cancellationToken)
    {
        return _context.SaveChangesAsync(
            cancellationToken);
    }
}
```

Puis :

```csharp
builder.Services.AddScoped<
    IUnitOfWork,
    UnitOfWork>();
```

---

# 6. Mais faut-il vraiment créer cette abstraction ?

Avec EF Core, la réponse est souvent :

> **Pas forcément.**

Pourquoi ?

Parce que :

```csharp
DbContext
```

fournit déjà :

```text
Change Tracking
+
SaveChanges
+
coordination des modifications
```

Donc créer :

```text
IUnitOfWork
    ↓
UnitOfWork
    ↓
DbContext
```

peut parfois simplement ajouter une couche autour d'une abstraction qui existe déjà.

---

# 7. Exemple sans Unit of Work supplémentaire

On peut parfaitement avoir :

```csharp
public class CreateOrderService
{
    private readonly AppDbContext _context;

    public CreateOrderService(AppDbContext context)
    {
        _context = context;
    }

    public async Task ExecuteAsync(
        CancellationToken cancellationToken)
    {
        var order = new Order();

        _context.Orders.Add(order);

        // autres modifications

        await _context.SaveChangesAsync(
            cancellationToken);
    }
}
```

Le `DbContext` joue déjà son rôle d'unité de travail.

---

# 8. Pourquoi un seul DbContext est important ?

Si plusieurs repositories utilisent le même `DbContext` :

```text
OrderRepository
       \
        \
         → même DbContext
        /
       /
StockRepository
```

les modifications peuvent être suivies dans la même unité de travail.

Si chaque repository crée son propre contexte :

```text
OrderRepository
      ↓
 DbContext A

StockRepository
      ↓
 DbContext B
```

on n'a plus le même Change Tracker.

Cela peut compliquer :

- la transaction ;
- le suivi des entités ;
- la cohérence des modifications ;
- la coordination de `SaveChanges`.

---

# 9. Lifetime du DbContext

Dans ASP.NET Core, `DbContext` est généralement enregistré comme :

```csharp
builder.Services.AddDbContext<AppDbContext>(
    options => ...);
```

Par défaut, le lifetime est **Scoped**.

Cela correspond bien au principe :

```text
HTTP Request
      ↓
Scope DI
      ↓
DbContext
      ↓
plusieurs opérations
      ↓
SaveChanges
```

Le même contexte peut donc être partagé entre plusieurs services scoped pendant une même requête.

---

# 10. Une transaction n'est pas exactement la même chose

Il faut distinguer :

```text
Unit of Work
```

et :

```text
Transaction
```

### Unit of Work

Représente un regroupement logique de modifications.

### Transaction

Garantit les propriétés d'une opération transactionnelle, notamment l'atomicité :

```text
Tout réussit
OU
Tout est annulé
```

On peut donc avoir :

```text
Unit of Work
    ↓
SaveChanges
```

sans avoir besoin de gérer manuellement une transaction dans chaque scénario.

EF Core utilise des transactions autour de `SaveChanges` dans les situations appropriées.

---

# 11. Transaction explicite

Lorsqu'un scénario nécessite plusieurs étapes contrôlées explicitement, on peut utiliser une transaction.

Exemple :

```csharp
await using var transaction =
    await _context.Database
        .BeginTransactionAsync(cancellationToken);

try
{
    _context.Orders.Add(order);

    _context.Products.Update(product);

    await _context.SaveChangesAsync(
        cancellationToken);

    await transaction.CommitAsync(
        cancellationToken);
}
catch
{
    await transaction.RollbackAsync(
        cancellationToken);

    throw;
}
```

Le principe devient :

```text
BEGIN
  ↓
modifications
  ↓
SaveChanges
  ↓
COMMIT

ou

ROLLBACK
```

---

# 12. Pourquoi ne pas créer une transaction partout ?

Une transaction explicite ajoute de la complexité.

Il faut donc se demander :

> « Est-ce que `SaveChanges` suffit pour mon scénario ? »

Si toutes les modifications sont persistées ensemble par un seul `SaveChanges`, EF Core gère déjà la transaction nécessaire à ce niveau dans les cas standards.

Une transaction explicite devient surtout intéressante lorsque plusieurs opérations doivent être coordonnées au-delà d'un simple `SaveChanges`.

---

# 13. Exemple : commande e-commerce

Supposons :

```text
Order
OrderLine
Stock
Payment
```

Une opération peut être :

```text
1. Créer Order
2. Créer OrderLine
3. Diminuer Stock
4. Enregistrer Payment
```

Selon l'architecture et les systèmes concernés, certaines étapes peuvent être dans une même transaction et d'autres non.

Si tout appartient à la même base :

```text
BEGIN
    Create Order
    Create OrderLines
    Update Stock
    Create Payment
COMMIT
```

Si le paiement passe par un système externe :

```text
Database
    +
External Payment API
```

la situation devient beaucoup plus complexe.

On ne peut pas simplement supposer qu'une transaction SQL classique englobe aussi l'API externe.

---

# 14. Unit of Work et services externes

Exemple :

```text
Database
    ↓
SaveChanges

External API
    ↓
Payment
```

Si :

```text
Database → COMMIT
```

puis :

```text
Payment API → ERROR
```

on peut avoir un état incohérent.

Dans ces architectures, on peut avoir besoin d'autres stratégies :

```text
Outbox Pattern
Retry
Compensation
Saga
Idempotency
```

Le Unit of Work ne résout pas automatiquement les transactions distribuées.

---

# 15. Unit of Work et Repository

Un scénario classique :

```csharp
await _orderRepository.AddAsync(order);

await _stockRepository.UpdateAsync(stock);

await _unitOfWork.SaveChangesAsync(
    cancellationToken);
```

Les repositories préparent les changements.

Le Unit of Work coordonne la persistance.

Mentalement :

```text
Repository
    = prépare / récupère

Unit of Work
    = valide l'ensemble
```

Avec EF Core :

```text
DbSet
    = accès aux données

DbContext
    = suivi + unité de travail + persistance
```

---

# 16. Attention aux repositories qui appellent tous SaveChanges

Imagine :

```csharp
await _orderRepository.CreateAsync(order);
```

qui fait :

```csharp
_context.Orders.Add(order);
await _context.SaveChangesAsync();
```

Puis :

```csharp
await _stockRepository.UpdateAsync(stock);
```

qui fait également :

```csharp
_context.Products.Update(product);
await _context.SaveChangesAsync();
```

On obtient :

```text
SaveChanges
    ↓
SaveChanges
```

Les opérations ne sont plus regroupées naturellement.

Si elles doivent être atomiques, cette organisation devient problématique.

---

# 17. Une approche plus cohérente

Les repositories peuvent simplement modifier le contexte :

```csharp
public async Task AddAsync(
    Order order,
    CancellationToken cancellationToken)
{
    await _context.Orders.AddAsync(
        order,
        cancellationToken);
}
```

Puis la couche qui orchestre le cas d'utilisation déclenche :

```csharp
await _context.SaveChangesAsync(
    cancellationToken);
```

ou :

```csharp
await _unitOfWork.SaveChangesAsync(
    cancellationToken);
```

Cela permet de contrôler le moment où l'ensemble est persisté.

---

# 18. Unit of Work et Change Tracker

Le Change Tracker est central dans le fonctionnement d'EF Core.

Exemple :

```csharp
var order = await _context.Orders
    .FirstAsync(o => o.Id == id);

order.Status = OrderStatus.Paid;

await _context.SaveChangesAsync();
```

EF Core détecte :

```text
Order
    Unchanged
       ↓
Status modifié
       ↓
Modified
       ↓
SaveChanges
       ↓
UPDATE
```

Le même `DbContext` conserve donc la connaissance des modifications de l'unité de travail.

---

# 19. Unit of Work et plusieurs entités

Exemple :

```csharp
var order = new Order();

var line = new OrderLine();

_context.Orders.Add(order);
_context.OrderLines.Add(line);
```

Avant :

```csharp
SaveChanges
```

les modifications sont seulement préparées dans le contexte.

Après :

```csharp
await _context.SaveChangesAsync();
```

EF Core persiste l'ensemble.

Mentalement :

```text
Add
Add
Update
Delete
     ↓
Change Tracker
     ↓
SaveChanges
     ↓
Database
```

---

# 20. Unit of Work et CancellationToken

Comme pour les opérations EF Core modernes, il est préférable de propager le `CancellationToken`.

Exemple :

```csharp
await _unitOfWork.SaveChangesAsync(
    cancellationToken);
```

Cela permet d'annuler l'opération lorsque cela est nécessaire.

Flux :

```text
HTTP Request
    ↓
CancellationToken
    ↓
Application
    ↓
Unit of Work
    ↓
EF Core
```

---

# 21. Unit of Work et architecture propre

Dans une Clean Architecture, on peut avoir :

```text
Application
    ↓
IUnitOfWork
    ↑
Infrastructure
    ↓
DbContext
```

L'Application connaît uniquement :

```csharp
IUnitOfWork
```

Infrastructure fournit l'implémentation.

Mais cette abstraction n'est pertinente que si elle apporte une vraie valeur.

Avec EF Core, on peut également décider que :

```text
Application
    ↓
DbContext
```

est suffisamment simple et acceptable selon les contraintes du projet.

Il n'existe pas de règle universelle imposant `IUnitOfWork`.

---

# 22. Unit of Work et CQRS

Dans CQRS, une Command peut représenter une unité de travail.

Exemple :

```text
CreateOrderCommand
        ↓
CreateOrderHandler
        ↓
plusieurs changements
        ↓
SaveChanges
```

La Command peut donc orchestrer :

```text
Order
OrderLines
Stock
```

puis déclencher la persistance.

Une Query, elle, ne nécessite généralement pas de Unit of Work au même sens, puisqu'elle est orientée lecture.

---

# 23. Les erreurs classiques

### Erreur 1 — penser que EF Core n'a pas de Unit of Work

`DbContext` joue déjà largement ce rôle.

---

### Erreur 2 — créer un Unit of Work uniquement pour suivre un pattern

Si :

```text
IUnitOfWork
    ↓
UnitOfWork
    ↓
DbContext
```

ne fait que déléguer :

```csharp
SaveChangesAsync();
```

demande-toi si cette abstraction apporte réellement quelque chose.

---

### Erreur 3 — chaque Repository appelle `SaveChanges`

Cela empêche souvent de regrouper facilement plusieurs modifications.

---

### Erreur 4 — confondre Unit of Work et transaction

```text
Unit of Work
≠
Transaction
```

Ils sont liés mais représentent des concepts différents.

---

### Erreur 5 — croire qu'une transaction SQL englobe automatiquement une API externe

Une transaction de base de données ne rend pas automatiquement atomique :

```text
SQL Server
+
Stripe
+
Email
+
API externe
```

---

### Erreur 6 — créer plusieurs DbContext sans raison

Plusieurs contextes signifient plusieurs Change Trackers et potentiellement plusieurs unités de travail.

---

# 24. Quand utiliser un Unit of Work explicite ?

Il peut être intéressant lorsque :

- l'architecture veut explicitement abstraire la persistance ;
- plusieurs repositories doivent partager une même unité de travail ;
- l'abstraction apporte une valeur métier ou architecturale ;
- le projet doit isoler davantage l'infrastructure ;
- plusieurs opérations doivent être coordonnées avant la persistance.

---

# 25. Quand ne pas en ajouter ?

Il peut être inutile lorsque :

- EF Core est déjà directement utilisé comme unité de travail ;
- `DbContext` est correctement injecté ;
- l'application est simple ;
- l'interface ne fait que recopier `SaveChangesAsync` ;
- cela ajoute une couche sans bénéfice concret.

---

# 26. Comparaison

| Concept | Rôle |
|---|---|
| `DbSet<T>` | Accès aux entités |
| `DbContext` | Session EF Core + Change Tracker + persistance |
| Repository | Abstraction éventuelle de l'accès aux données |
| Unit of Work | Regroupement logique des changements |
| Transaction | Garantit l'atomicité d'un ensemble d'opérations |

Mentalement :

```text
DbSet
  ↓
accéder aux données

DbContext
  ↓
suivre les changements

SaveChanges
  ↓
persister

Transaction
  ↓
tout ou rien
```

---

# 27. Exemple complet

Supposons :

```text
Order
Product
Stock
```

Service :

```csharp
public class CreateOrderService
{
    private readonly IOrderRepository _orders;
    private readonly IProductRepository _products;
    private readonly IUnitOfWork _unitOfWork;

    public CreateOrderService(
        IOrderRepository orders,
        IProductRepository products,
        IUnitOfWork unitOfWork)
    {
        _orders = orders;
        _products = products;
        _unitOfWork = unitOfWork;
    }

    public async Task ExecuteAsync(
        CancellationToken cancellationToken)
    {
        var order = new Order();

        var product = await _products.GetByIdAsync(
            1,
            cancellationToken);

        // règles métier

        await _orders.AddAsync(
            order,
            cancellationToken);

        // modification du stock

        await _unitOfWork.SaveChangesAsync(
            cancellationToken);
    }
}
```

Flux :

```text
CreateOrderService
       |
       +----> OrderRepository
       |
       +----> ProductRepository
       |
       +----> UnitOfWork
                    |
                    v
                DbContext
                    |
                    v
                 Database
```

---

# 28. Mental model

Imagine un panier d'achat.

Tu ajoutes :

```text
Produit A
Produit B
Produit C
```

Mais tu ne vas pas forcément à la caisse après chaque produit.

Tu prépares ton panier :

```text
Add A
Add B
Add C
```

Puis :

```text
Checkout
```

Le `Checkout` représente mentalement le moment où l'ensemble est validé.

Avec EF Core :

```text
Add
Add
Update
Delete
      ↓
SaveChanges
```

Le `DbContext` joue le rôle du panier qui connaît l'ensemble des changements.

---

# 29. Checklist

Avant d'ajouter `IUnitOfWork` :

- [ ] EF Core `DbContext` ne suffit-il vraiment pas ?
- [ ] Plusieurs repositories doivent-ils partager la même unité de travail ?
- [ ] Qui appelle `SaveChangesAsync()` ?
- [ ] Les repositories évitent-ils de sauvegarder individuellement ?
- [ ] Les transactions sont-elles réellement nécessaires ?
- [ ] Les opérations externes sont-elles prises en compte ?
- [ ] Le même `DbContext` est-il partagé lorsque cela est nécessaire ?
- [ ] Le `CancellationToken` est-il propagé ?
- [ ] L'abstraction apporte-t-elle une vraie valeur ?

---

# 30. Questions d'entretien

### 1. Qu'est-ce que le Unit of Work ?

C'est un pattern qui regroupe plusieurs changements afin de les traiter comme une même unité de travail avant leur persistance.

### 2. Quel objet EF Core joue déjà ce rôle ?

`DbContext` joue largement le rôle de Unit of Work grâce notamment au Change Tracker et à `SaveChanges`.

### 3. Unit of Work et transaction, est-ce la même chose ?

Non. Le Unit of Work représente le regroupement des modifications ; une transaction garantit notamment leur atomicité.

### 4. Pourquoi éviter `SaveChanges()` dans chaque Repository ?

Parce que cela empêche ou complique le regroupement de plusieurs modifications qui devraient être persistées ensemble.

### 5. Pourquoi le même `DbContext` peut-il être important ?

Parce qu'il possède le même Change Tracker et peut coordonner plusieurs modifications dans une même unité de travail.

### 6. Faut-il toujours créer `IUnitOfWork` avec EF Core ?

Non. EF Core fournit déjà `DbContext`, qui peut être utilisé directement lorsque cela correspond aux besoins du projet.

---

# À retenir

```text
DbContext
    ↓
Change Tracker
    ↓
plusieurs changements
    ↓
SaveChanges
    ↓
Database
```

Et :

```text
Unit of Work
≠ Transaction
```

Le Unit of Work organise le travail.

La transaction garantit notamment le tout-ou-rien.

# Phrase à mémoriser

> **Avec EF Core, le DbContext joue déjà largement le rôle de Unit of Work : il suit les changements puis les persiste avec SaveChanges.**
