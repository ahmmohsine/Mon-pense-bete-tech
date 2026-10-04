# Architecture — Dependency Inversion Principle (DIP)

## 1. Qu'est-ce que le Dependency Inversion Principle ?

Le **Dependency Inversion Principle**, ou **DIP**, est le dernier principe de **SOLID**.

Il peut être résumé ainsi :

> **Les modules de haut niveau ne doivent pas dépendre directement des modules de bas niveau. Les deux doivent dépendre d'abstractions.**

Et :

> **Les abstractions ne doivent pas dépendre des détails. Les détails doivent dépendre des abstractions.**

En .NET, on rencontre ce principe très souvent avec :

```text
Interfaces
+
Dependency Injection
```

---

# 2. Pourquoi parle-t-on de "Dependency Inversion" ?

Imaginons :

```text
OrderService
     ↓
SqlOrderRepository
     ↓
SQL Server
```

`OrderService` dépend directement de :

```csharp
SqlOrderRepository
```

Si on veut changer la manière de stocker les commandes, le service est fortement couplé à cette implémentation.

Avec DIP :

```text
OrderService
     ↓
IOrderRepository
     ↑
SqlOrderRepository
```

Le service dépend d'une abstraction.

L'implémentation dépend elle aussi de cette abstraction.

---

# 3. Dépendance classique

Sans inversion :

```csharp
public class OrderService
{
    private readonly SqlOrderRepository _repository;

    public OrderService()
    {
        _repository = new SqlOrderRepository();
    }
}
```

Le service :

1. connaît l'implémentation ;
2. crée lui-même la dépendance ;
3. est couplé à SQL ;
4. est plus difficile à tester.

Le graphe est :

```text
OrderService
     ↓
SqlOrderRepository
```

---

# 4. Dépendance inversée

Avec une abstraction :

```csharp
public interface IOrderRepository
{
    Task<Order?> GetByIdAsync(
        int id,
        CancellationToken cancellationToken);
}
```

Puis :

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;

    public OrderService(
        IOrderRepository repository)
    {
        _repository = repository;
    }
}
```

Et l'implémentation :

```csharp
public class SqlOrderRepository
    : IOrderRepository
{
    ...
}
```

On obtient :

```text
OrderService
      ↓
IOrderRepository
      ↑
SqlOrderRepository
```

Le service ne connaît plus directement le détail.

---

# 5. Le point important : l'interface n'est pas le but

Il ne faut pas retenir :

> « DIP = mettre des interfaces partout. »

Ce serait faux.

Le vrai principe est :

> **Réduire le couplage entre le code de haut niveau et les détails d'implémentation.**

Une interface est seulement un outil permettant parfois d'obtenir cette inversion.

---

# 6. Module de haut niveau vs module de bas niveau

### Module de haut niveau

Il contient la logique importante de l'application.

Exemple :

```text
CreateOrderService
ProcessPayment
RegisterCustomer
CancelReservation
```

Il exprime :

> « Que doit faire l'application ? »

### Module de bas niveau

Il contient les détails techniques.

Exemple :

```text
SQL repository
SMTP client
Azure Blob client
HTTP client
Redis client
```

Il exprime :

> « Comment le faire techniquement ? »

---

# 7. Exemple avec un email

Sans DIP :

```csharp
public class RegistrationService
{
    private readonly SmtpEmailSender _emailSender;

    public RegistrationService()
    {
        _emailSender = new SmtpEmailSender();
    }

    public async Task RegisterAsync(...)
    {
        // création utilisateur

        await _emailSender.SendAsync(...);
    }
}
```

Le service métier connaît :

```text
SMTP
```

Ce n'est pas idéal.

---

# 8. Avec DIP

On définit :

```csharp
public interface IEmailSender
{
    Task SendAsync(
        string recipient,
        string subject,
        string body,
        CancellationToken cancellationToken);
}
```

Puis :

```csharp
public class RegistrationService
{
    private readonly IEmailSender _emailSender;

    public RegistrationService(
        IEmailSender emailSender)
    {
        _emailSender = emailSender;
    }

    public async Task RegisterAsync(...)
    {
        // logique de registration

        await _emailSender.SendAsync(
            recipient,
            subject,
            body,
            cancellationToken);
    }
}
```

Infrastructure :

```csharp
public class SmtpEmailSender : IEmailSender
{
    public async Task SendAsync(...)
    {
        // SMTP
    }
}
```

Le graphe :

```text
RegistrationService
        ↓
    IEmailSender
        ↑
 SmtpEmailSender
```

---

# 9. Pourquoi cette inversion est utile ?

Imaginons qu'on abandonne SMTP.

On veut utiliser :

```text
Azure Communication Services
```

On peut créer :

```csharp
public class AzureEmailSender : IEmailSender
{
    ...
}
```

Puis modifier la configuration DI :

```csharp
builder.Services.AddScoped<
    IEmailSender,
    AzureEmailSender>();
```

Le `RegistrationService` n'a pas besoin de connaître le changement.

---

# 10. Le rôle de Dependency Injection

DIP et Dependency Injection sont liés, mais ce sont deux concepts différents.

### DIP

C'est un **principe de conception**.

Il dit notamment :

```text
Le haut niveau ne doit pas dépendre
directement des détails.
```

### Dependency Injection

C'est une **technique** permettant de fournir les dépendances à un objet.

Exemple :

```csharp
builder.Services.AddScoped<
    IEmailSender,
    SmtpEmailSender>();
```

Donc :

```text
DIP
    = principe

DI
    = mécanisme
```

---

# 11. DI sans DIP

On peut techniquement faire de la Dependency Injection tout en ayant une mauvaise abstraction.

Exemple :

```csharp
public class OrderService
{
    public OrderService(
        SqlOrderRepository repository)
    {
        ...
    }
}
```

La dépendance est injectée.

Mais le service dépend toujours directement de :

```text
SqlOrderRepository
```

Donc :

```text
DI utilisée
≠
DIP automatiquement respecté
```

---

# 12. DIP avec une interface

On préfère :

```csharp
public OrderService(
    IOrderRepository repository)
{
    ...
}
```

Puis :

```csharp
builder.Services.AddScoped<
    IOrderRepository,
    SqlOrderRepository>();
```

Le service dépend de l'abstraction.

Le conteneur DI choisit l'implémentation.

---

# 13. DIP et Clean Architecture

Le DIP est particulièrement important dans Clean Architecture.

On veut :

```text
Domain
   ↑
Application
   ↑
Infrastructure
```

L'idée est que les détails techniques puissent dépendre des abstractions définies par le cœur.

Exemple :

```text
Application
    ↓
IUserRepository
    ↑
Infrastructure
    ↓
EF Core
```

Le cœur définit ce dont il a besoin.

Infrastructure fournit le détail technique.

---

# 14. "Abstractions ne doivent pas dépendre des détails"

Prenons :

```csharp
public interface IEmailSender
{
    Task SendAsync(...);
}
```

Cette interface ne dit rien sur :

```text
SMTP
Azure
SendGrid
MailKit
```

Elle exprime seulement le besoin :

> « Envoyer un email. »

À l'inverse :

```csharp
public class SmtpEmailSender : IEmailSender
```

connaît le détail SMTP.

Donc :

```text
IEmailSender
      ↑
SmtpEmailSender
```

Le détail dépend de l'abstraction.

---

# 15. Attention aux abstractions mal conçues

Une mauvaise abstraction peut être :

```csharp
public interface IEmailSender
{
    Task SendUsingSmtpAsync(
        string smtpHost,
        int smtpPort,
        ...);
}
```

Pourquoi ?

Parce que l'interface elle-même connaît :

```text
SMTP
```

Elle n'est donc plus vraiment indépendante du détail.

Une meilleure abstraction :

```csharp
public interface IEmailSender
{
    Task SendAsync(
        string recipient,
        string subject,
        string body,
        CancellationToken cancellationToken);
}
```

Le contrat exprime le besoin métier/technique général.

---

# 16. DIP et testabilité

Supposons :

```csharp
public class PaymentService
{
    private readonly IPaymentGateway _gateway;

    public PaymentService(
        IPaymentGateway gateway)
    {
        _gateway = gateway;
    }
}
```

Pour un test, on peut fournir une fausse implémentation :

```csharp
public class FakePaymentGateway
    : IPaymentGateway
{
    ...
}
```

Le test devient :

```text
PaymentService
      ↓
FakePaymentGateway
```

au lieu de :

```text
PaymentService
      ↓
Real Payment API
```

Cela permet d'isoler le comportement testé.

---

# 17. DIP et changement de technologie

Supposons :

```text
IFileStorage
```

Implémentation actuelle :

```text
LocalFileStorage
```

Puis demain :

```text
AzureBlobStorage
```

Le code applicatif peut continuer à utiliser :

```csharp
IFileStorage
```

Seule l'implémentation change.

Mentalement :

```text
Application
      ↓
Contrat
      ↑
Technologie
```

---

# 18. Le mauvais exemple classique

```csharp
public class OrderService
{
    public void CreateOrder()
    {
        var context = new AppDbContext();

        // EF Core
        // SQL
        // logique métier
    }
}
```

Ici, tout est mélangé :

```text
Métier
+
Persistance
+
Création de dépendance
```

Cela augmente fortement le couplage.

---

# 19. Meilleure approche

```csharp
public class OrderService
{
    private readonly IOrderRepository _repository;

    public OrderService(
        IOrderRepository repository)
    {
        _repository = repository;
    }

    public async Task CreateAsync(...)
    {
        // logique métier

        await _repository.AddAsync(...);
    }
}
```

Puis :

```csharp
builder.Services.AddScoped<
    IOrderRepository,
    OrderRepository>();
```

Le service ne crée plus son infrastructure.

---

# 20. DIP et composition root

Dans une application .NET, le point où l'on connecte :

```text
interfaces
+
implémentations
```

est souvent le **Composition Root**.

Dans ASP.NET Core, cela se trouve généralement autour de :

```csharp
Program.cs
```

Exemple :

```csharp
builder.Services.AddScoped<
    IOrderRepository,
    OrderRepository>();

builder.Services.AddScoped<
    IEmailSender,
    SmtpEmailSender>();
```

L'application construit ainsi le graphe de dépendances.

---

# 21. Pourquoi le Composition Root est important ?

Le code métier ne devrait pas faire :

```csharp
new SmtpEmailSender()
```

ou :

```csharp
new SqlOrderRepository()
```

Le point de composition connaît les implémentations concrètes.

Donc :

```text
Application
    ↓
Interfaces

Program.cs
    ↓
choisit les implémentations

Infrastructure
    ↓
implémente les interfaces
```

---

# 22. DIP et Factory

Il existe des situations où une Factory est utile.

Exemple :

```text
IPaymentGateway
       ↑
+------+------+
|             |
Stripe       PayPal
```

Le choix de l'implémentation peut être dynamique.

Une Factory peut alors retourner :

```csharp
IPaymentGateway
```

sans exposer les détails à l'appelant.

Mais là encore :

> Utiliser une Factory seulement lorsque le choix dynamique le justifie.

---

# 23. DIP et polymorphisme

DIP fonctionne naturellement avec le polymorphisme.

Exemple :

```csharp
public interface INotificationSender
{
    Task SendAsync(...);
}
```

Implémentations :

```text
EmailNotificationSender
SmsNotificationSender
PushNotificationSender
```

Le code de haut niveau peut travailler avec :

```csharp
INotificationSender
```

sans connaître la classe concrète.

---

# 24. DIP ne signifie pas "tout abstraire"

C'est un point très important.

Il n'est pas nécessaire de créer :

```text
IStringHelper
IListHelper
IDateTimeHelper
IProductMapper
IUserManager
```

pour chaque classe.

On introduit une abstraction lorsqu'elle protège une frontière ou apporte une vraie flexibilité.

Exemple pertinent :

```text
IPaymentGateway
```

car le système externe peut changer.

Exemple souvent inutile :

```text
IStringFormatter
```

pour une simple méthode triviale qui n'a aucune raison d'être remplacée.

---

# 25. Le principe de stabilité

Les parties importantes et stables du système doivent être protégées contre les détails qui changent fréquemment.

Exemple :

```text
Métier
   ↓
stable

SMTP
EF Core
Azure SDK
   ↓
détails susceptibles de changer
```

DIP aide à placer une frontière entre les deux.

---

# 26. DIP et EF Core

Exemple :

```text
Application
      ↓
IProductRepository
      ↑
ProductRepository
      ↓
EF Core
```

Le code applicatif n'a pas besoin de connaître :

```csharp
DbContext
DbSet<Product>
FirstOrDefaultAsync
```

si l'abstraction choisie lui suffit.

Cependant, avec EF Core, il faut éviter d'appliquer mécaniquement cette règle partout.

Une abstraction est utile lorsqu'elle correspond à une vraie frontière.

---

# 27. DIP et ASP.NET Core

Même ASP.NET Core utilise énormément ce principe.

Exemples :

```text
ILogger<T>
IConfiguration
IHttpClientFactory
IHostEnvironment
```

Le code utilise des abstractions ou contrats plutôt que de construire lui-même tous les détails.

Le conteneur DI permet ensuite de fournir les implémentations.

---

# 28. DIP et `new`

Voir :

```csharp
new SmtpEmailSender()
```

dans une classe métier est souvent un signal d'alerte.

Cela signifie :

```text
La classe choisit elle-même son détail technique.
```

À l'inverse :

```csharp
public OrderService(
    IEmailSender emailSender)
```

signifie :

```text
Le code reçoit ce dont il a besoin.
```

Attention : le mot-clé `new` n'est pas interdit.

Créer un objet métier simple peut être parfaitement correct :

```csharp
var order = new Order(...);
```

Le problème est surtout de créer directement une dépendance technique depuis un composant de haut niveau.

---

# 29. DIP et SRP

DIP et SRP sont liés mais différents.

### SRP

> Une classe doit avoir une responsabilité cohérente.

### DIP

> Les dépendances doivent être dirigées vers des abstractions appropriées plutôt que vers des détails.

Exemple :

```text
OrderService
    ↓
IEmailSender
```

peut respecter DIP.

Mais si `OrderService` fait :

```text
Order
+
Payment
+
Email
+
Logging
+
PDF
+
Reporting
```

il peut toujours violer SRP.

Les principes SOLID se complètent.

---

# 30. DIP et OCP

Le DIP facilite également l'**Open/Closed Principle**.

Si le code dépend d'une abstraction :

```text
IPaymentGateway
```

on peut ajouter :

```text
StripePaymentGateway
PayPalPaymentGateway
AdyenPaymentGateway
```

sans modifier nécessairement le code qui utilise le contrat.

Cela peut faciliter l'extension du système.

---

# 31. Architecture mentale

Imagine une prise électrique.

```text
Application
    ↓
Prise standard
    ↑
Appareil
```

L'application ne veut pas savoir comment l'électricité est produite.

Elle veut un contrat :

```text
« Fournis-moi l'électricité dont j'ai besoin. »
```

La centrale électrique est le détail.

Dans une application :

```text
IEmailSender
    ↑
SmtpEmailSender
```

L'interface est la prise.

L'implémentation est l'appareil/détail concret.

---

# 32. Checklist DIP

Avant de considérer une dépendance comme bien conçue :

- [ ] Le code métier dépend-il directement d'un détail technique ?
- [ ] La dépendance est-elle créée avec `new` au mauvais endroit ?
- [ ] Une abstraction est-elle réellement nécessaire ?
- [ ] L'interface exprime-t-elle un besoin plutôt qu'une technologie ?
- [ ] L'implémentation dépend-elle de l'abstraction ?
- [ ] Le Composition Root choisit-il l'implémentation ?
- [ ] La DI est-elle utilisée pour fournir la dépendance ?
- [ ] L'abstraction améliore-t-elle réellement le test ou le découplage ?
- [ ] Ai-je évité de créer des interfaces inutiles ?

---

# 33. Questions d'entretien

### 1. Qu'est-ce que le Dependency Inversion Principle ?

C'est un principe SOLID selon lequel les modules de haut niveau ne doivent pas dépendre directement des modules de bas niveau ; les deux doivent dépendre d'abstractions.

### 2. DIP signifie-t-il utiliser des interfaces partout ?

Non. Les interfaces sont un moyen possible d'obtenir l'inversion de dépendance, mais elles ne sont pas une fin en soi.

### 3. Quelle différence entre DIP et DI ?

DIP est un principe de conception. DI est une technique permettant de fournir les dépendances à un objet.

### 4. Peut-on utiliser DI sans respecter DIP ?

Oui. On peut injecter directement une classe concrète :

```csharp
OrderService(SqlOrderRepository repository)
```

La dépendance reste alors liée au détail concret.

### 5. Où enregistrer les implémentations ?

Généralement dans le Composition Root, par exemple dans `Program.cs` pour une application ASP.NET Core.

### 6. Pourquoi DIP est-il important en Clean Architecture ?

Parce qu'il permet aux couches centrales de dépendre d'abstractions et aux détails techniques de dépendre de ces abstractions.

### 7. Quel est l'intérêt pour les tests ?

On peut remplacer une implémentation réelle par une fausse implémentation ou un mock :

```text
RealPaymentGateway
        ↓
FakePaymentGateway
```

sans modifier le code testé.

---

# À retenir

Le mauvais sens :

```text
OrderService
      ↓
SqlOrderRepository
```

Le sens recherché :

```text
OrderService
      ↓
IOrderRepository
      ↑
SqlOrderRepository
```

Le cœur dépend du contrat.

Le détail dépend du contrat.

# Phrase à mémoriser

> **DIP : le métier ne dépend pas du détail ; le détail dépend de l'abstraction.**
