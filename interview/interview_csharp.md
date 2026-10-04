# Entretien — C#

Cette fiche rassemble les notions C# les plus importantes à savoir expliquer en entretien.

L'objectif n'est pas de réciter une définition.

Pour chaque notion, il faut idéalement savoir répondre à :

```text
Qu'est-ce que c'est ?
        ↓
Comment ça fonctionne ?
        ↓
Pourquoi l'utiliser ?
        ↓
Quel exemple concret ?
        ↓
Quelles limites / pièges ?
```

---

# 1. Classe vs objet

Une classe est une définition.

```csharp
public class User
{
    public string Name { get; set; }
}
```

Un objet est une instance de cette classe :

```csharp
var user = new User
{
    Name = "Ahlame"
};
```

Mental model :

```text
class
 ↓
plan / modèle

object
 ↓
instance créée à partir du modèle
```

### Question d'entretien

**Quelle différence entre une classe et un objet ?**

> Une classe définit la structure et le comportement d'un type. Un objet est une instance concrète créée à partir de cette classe.

---

# 2. Value types vs reference types

C'est une notion importante.

Exemples de value types :

```csharp
int
double
bool
struct
enum
```

Exemples de reference types :

```csharp
class
string
array
delegate
```

Mental model simplifié :

```text
Value type
→ contient directement sa valeur

Reference type
→ variable contenant une référence vers un objet
```

Attention : cette simplification ne signifie pas que tout se résume simplement à "stack vs heap".

La localisation mémoire dépend du contexte et des optimisations du runtime.

---

# 3. Passage par valeur

Par défaut, les paramètres sont passés par valeur.

Exemple :

```csharp
void Change(int value)
{
    value = 10;
}

int number = 5;

Change(number);
```

Après l'appel :

```text
number = 5
```

Pourquoi ?

Parce que la méthode reçoit une copie de la valeur.

---

# 4. Reference type passé par valeur

C'est un piège classique.

```csharp
void Change(User user)
{
    user.Name = "Bob";
}
```

Puis :

```csharp
var user = new User
{
    Name = "Ahlame"
};

Change(user);
```

Après :

```text
user.Name = "Bob"
```

Pourquoi ?

Parce que la référence elle-même est passée par valeur.

Les deux variables référencent le même objet.

Mental model :

```text
user ────────┐
             ↓
          [User]
             ↑
        parameter
```

La référence est copiée, mais elle pointe vers le même objet.

---

# 5. `ref`

`ref` permet de passer une variable par référence.

```csharp
void Change(ref int value)
{
    value = 10;
}

int number = 5;

Change(ref number);
```

Résultat :

```text
number = 10
```

Le paramètre peut agir directement sur la variable appelante.

---

# 6. `out`

`out` permet à une méthode de produire une valeur via un paramètre.

Exemple :

```csharp
if (int.TryParse("42", out int number))
{
    Console.WriteLine(number);
}
```

Ici :

```text
TryParse
   ↓
retour bool
   +
valeur via out
```

C'est une ancienne forme courante de pattern de retour multiple.

---

# 7. `in`

`in` permet de passer un paramètre par référence en lecture seule.

Exemple :

```csharp
void Print(in LargeStruct value)
{
    ...
}
```

L'intention est :

```text
référence
+
pas de modification du paramètre
```

Cette notion est surtout pertinente pour certains value types volumineux.

---

# 8. `ref`, `out`, `in`

À mémoriser :

```text
ref
→ référence + lecture/écriture

out
→ référence + méthode doit produire une valeur

in
→ référence + lecture seule
```

---

# 9. Interface

Une interface définit un contrat.

```csharp
public interface IEmailService
{
    Task SendAsync(string email);
}
```

Une classe implémente ce contrat :

```csharp
public class EmailService : IEmailService
{
    public Task SendAsync(string email)
    {
        ...
    }
}
```

Mental model :

```text
Interface
    ↓
contrat

Class
    ↓
implémentation
```

---

# 10. Pourquoi utiliser une interface ?

Exemple :

```csharp
public class UserService
{
    private readonly IEmailService _emailService;

    public UserService(IEmailService emailService)
    {
        _emailService = emailService;
    }
}
```

Le `UserService` dépend de :

```text
IEmailService
```

et non de :

```text
EmailService
```

Cela réduit le couplage et facilite notamment les tests.

---

# 11. Polymorphisme

Le polymorphisme permet de manipuler différents objets via une abstraction commune.

Exemple :

```csharp
IAnimal animal = new Dog();

animal.MakeSound();
```

Une autre implémentation :

```csharp
IAnimal animal = new Cat();

animal.MakeSound();
```

Le code appelant travaille avec :

```text
IAnimal
```

mais le comportement concret dépend de l'objet réel.

Mental model :

```text
même contrat
   ↓
implémentations différentes
```

---

# 12. Héritage

Exemple :

```csharp
public class Animal
{
    public void Eat()
    {
    }
}

public class Dog : Animal
{
}
```

`Dog` hérite de `Animal`.

Il récupère les membres accessibles de la classe de base.

Mais l'héritage doit être utilisé lorsqu'il existe réellement une relation conceptuelle de type :

```text
Dog is an Animal
```

Il ne faut pas utiliser l'héritage uniquement pour réutiliser quelques lignes de code.

---

# 13. Composition vs héritage

La composition consiste à construire une classe à partir d'autres objets.

Exemple :

```csharp
public class OrderService
{
    private readonly IPaymentService _paymentService;

    public OrderService(IPaymentService paymentService)
    {
        _paymentService = paymentService;
    }
}
```

Ici :

```text
OrderService
    ↓
possède/utilise
PaymentService
```

En général, la composition permet souvent un couplage plus flexible que l'héritage.

Mental model :

```text
Inheritance
→ is-a

Composition
→ has-a / uses-a
```

---

# 14. Encapsulation

L'encapsulation consiste à contrôler l'accès à l'état interne d'un objet.

Mauvaise conception :

```csharp
public class BankAccount
{
    public decimal Balance;
}
```

N'importe quel code pourrait faire :

```csharp
account.Balance = -100000;
```

On préfère contrôler les modifications :

```csharp
public class BankAccount
{
    public decimal Balance { get; private set; }

    public void Deposit(decimal amount)
    {
        if (amount <= 0)
            throw new ArgumentOutOfRangeException(nameof(amount));

        Balance += amount;
    }
}
```

L'objet protège ses invariants.

---

# 15. `private`, `protected`, `public`, `internal`

Les principaux niveaux d'accès :

```text
public
→ accessible depuis l'extérieur

private
→ accessible uniquement dans le type

protected
→ accessible dans le type et ses classes dérivées

internal
→ accessible dans le même assembly
```

Question fréquente :

> Pourquoi mettre un setter en `private` ?

Exemple :

```csharp
public decimal Balance { get; private set; }
```

Cela permet de lire la valeur de l'extérieur mais d'empêcher une modification arbitraire.

---

# 16. `static`

Un membre `static` appartient au type plutôt qu'à une instance particulière.

Exemple :

```csharp
public static class MathHelper
{
    public static int Double(int value)
    {
        return value * 2;
    }
}
```

Utilisation :

```csharp
var result = MathHelper.Double(5);
```

On n'a pas besoin de :

```csharp
new MathHelper()
```

Mental model :

```text
instance member
→ appartient à un objet

static member
→ appartient au type
```

---

# 17. `const` vs `readonly`

### `const`

```csharp
public const double TaxRate = 0.21;
```

La valeur est une constante connue à la compilation.

### `readonly`

```csharp
public readonly string Name;
```

Elle peut être assignée lors de l'initialisation ou dans le constructeur selon le contexte.

Mental model :

```text
const
→ constante de compilation

readonly
→ assignable à l'initialisation / construction, puis non modifiable
```

---

# 18. `var`

`var` ne signifie pas :

```text
type dynamique
```

Le compilateur détermine le type à la compilation.

```csharp
var name = "Ahlame";
```

Le type est :

```csharp
string
```

Et :

```csharp
var number = 42;
```

est :

```csharp
int
```

Donc :

```text
var
→ typage statique avec inférence de type
```

---

# 19. `dynamic`

`dynamic` est différent.

```csharp
dynamic value = GetSomething();

value.DoSomething();
```

Une partie de la résolution est reportée à l'exécution.

Cela réduit certaines vérifications du compilateur.

Mental model :

```text
var
→ type déterminé à la compilation

dynamic
→ résolution dynamique à l'exécution
```

`dynamic` doit donc être utilisé avec discernement.

---

# 20. Nullable Reference Types

Avec les nullable reference types :

```csharp
string name;
```

signifie conceptuellement :

```text
name ne devrait pas être null
```

Alors que :

```csharp
string? name;
```

indique :

```text
name peut être null
```

Exemple :

```csharp
string? name = null;
```

Le compilateur peut alors signaler des utilisations potentiellement dangereuses.

---

# 21. Null-coalescing

```csharp
var displayName = name ?? "Unknown";
```

Cela signifie :

```text
si name != null
    → name

sinon
    → "Unknown"
```

Mental model :

```text
A ?? B
→ A si A n'est pas null
→ sinon B
```

---

# 22. Null-conditional

```csharp
user?.Address?.City
```

Cela signifie :

```text
si user existe
    ↓
si Address existe
    ↓
City
```

Sinon le résultat devient `null`.

Cela permet d'éviter certains `NullReferenceException`.

---

# 23. Generics

Les generics permettent d'écrire du code réutilisable tout en conservant le typage.

Exemple :

```csharp
public class Repository<T>
{
    public void Add(T entity)
    {
        ...
    }
}
```

On peut avoir :

```csharp
Repository<User>
Repository<Product>
Repository<Order>
```

Mental model :

```text
T
→ type paramétrique
```

---

# 24. Pourquoi les generics ?

Sans generics, on pourrait être tenté de travailler avec :

```csharp
object
```

Mais cela réduit le typage et peut nécessiter des conversions.

Les generics permettent :

```text
réutilisation
+
type safety
+
moins de casts
```

---

# 25. Contraintes génériques

On peut limiter `T`.

Exemple :

```csharp
public class Repository<T>
    where T : class
{
}
```

Cela signifie :

```text
T doit être un type référence
```

Autres contraintes possibles :

```csharp
where T : struct
where T : new()
where T : BaseEntity
where T : IMyInterface
```

---

# 26. LINQ

LINQ permet de manipuler des collections et des sources de données avec une syntaxe déclarative.

Exemple :

```csharp
var activeUsers = users
    .Where(u => u.Active)
    .OrderBy(u => u.Name)
    .Select(u => u.Name);
```

Mental model :

```text
source
 ↓
Where
 ↓
OrderBy
 ↓
Select
 ↓
résultat
```

---

# 27. `IEnumerable` vs `IQueryable`

Question très fréquente.

### `IEnumerable`

Travaille généralement sur des données déjà en mémoire.

```text
Database
 ↓
données récupérées
 ↓
IEnumerable
 ↓
LINQ en mémoire
```

### `IQueryable`

Permet à un provider de traduire l'expression vers une source de données.

Avec EF Core :

```text
IQueryable
 ↓
LINQ expression
 ↓
SQL
 ↓
Database
```

Mental model :

```text
IEnumerable
→ LINQ to Objects

IQueryable
→ requête composable pouvant être traduite par un provider
```

---

# 28. Deferred execution

Certaines requêtes LINQ sont exécutées seulement lorsqu'elles sont consommées.

Exemple :

```csharp
var query = users
    .Where(u => u.Active);
```

À ce stade, selon le type de source, le traitement peut ne pas avoir encore été exécuté.

Puis :

```csharp
var result = query.ToList();
```

déclenche l'exécution.

Mental model :

```text
construction de la requête
        ↓
...
consommation
        ↓
exécution
```

---

# 29. `ToList()` et `ToArray()`

```csharp
var users = query.ToList();
```

matérialise les résultats dans une liste.

```csharp
var users = query.ToArray();
```

matérialise dans un tableau.

Avec EF Core, cela signifie généralement que la requête SQL est exécutée à ce moment-là.

---

# 30. Delegates

Un delegate représente une référence typée vers une méthode.

Exemple :

```csharp
public delegate int Operation(int a, int b);
```

Puis :

```csharp
int Add(int a, int b)
{
    return a + b;
}
```

On peut avoir :

```csharp
Operation operation = Add;
```

Puis :

```csharp
var result = operation(2, 3);
```

Mental model :

```text
delegate
→ type représentant une méthode compatible
```

---

# 31. `Action`, `Func`, `Predicate`

C# fournit des delegates génériques courants.

### `Action`

Retourne `void`.

```csharp
Action<string> print = Console.WriteLine;
```

### `Func`

Retourne une valeur.

```csharp
Func<int, int, int> add =
    (a, b) => a + b;
```

### `Predicate`

Retourne un booléen.

```csharp
Predicate<int> isPositive =
    value => value > 0;
```

Mental model :

```text
Action
→ void

Func
→ retourne une valeur

Predicate
→ bool
```

---

# 32. Events

Un event permet à un objet de notifier des abonnés qu'un événement s'est produit.

Exemple :

```csharp
public event EventHandler? Saved;
```

Puis :

```csharp
Saved?.Invoke(this, EventArgs.Empty);
```

Un autre objet peut s'abonner :

```csharp
service.Saved += OnSaved;
```

Mental model :

```text
Publisher
    ↓
event
   ↙ ↘
Subscriber A  Subscriber B
```

---

# 33. `record` vs `class`

Un `record` est particulièrement adapté à la représentation de données lorsque la **value equality** est souhaitée.

Exemple :

```csharp
public record UserDto(
    int Id,
    string Name);
```

Deux records ayant les mêmes valeurs peuvent être considérés égaux selon les règles du record.

Pour une entité avec une identité et un cycle de vie, une `class` est souvent plus naturelle.

Mental model :

```text
record
→ valeur / données

class
→ identité / comportement
```

Ce n'est pas une règle absolue, mais une bonne intuition.

---

# 34. Exceptions

Une exception représente une situation exceptionnelle ou une erreur qui interrompt le flux normal.

Exemple :

```csharp
throw new InvalidOperationException(
    "The operation is invalid.");
```

On peut gérer :

```csharp
try
{
    ...
}
catch (InvalidOperationException ex)
{
    ...
}
finally
{
    ...
}
```

Mental model :

```text
try
→ code surveillé

catch
→ traitement de l'erreur

finally
→ nettoyage
```

---

# 35. Ne pas utiliser les exceptions pour le contrôle normal

Mauvaise idée :

```csharp
try
{
    var user = users.Single(u => u.Id == id);
}
catch
{
    ...
}
```

si l'absence de l'utilisateur est une situation normale.

Il existe souvent des méthodes mieux adaptées :

```csharp
SingleOrDefault()
FirstOrDefault()
Any()
```

Les exceptions doivent généralement représenter des situations réellement exceptionnelles.

---

# 36. `async` / `await`

Exemple :

```csharp
public async Task<User> GetUserAsync()
{
    return await repository.GetUserAsync();
}
```

`async` indique que la méthode utilise le modèle asynchrone.

`await` permet d'attendre une opération asynchrone sans bloquer inutilement le thread pendant une opération I/O.

Mental model :

```text
appel I/O
   ↓
await
   ↓
thread peut être libéré
   ↓
opération terminée
   ↓
suite de la méthode
```

---

# 37. `Task` vs `Thread`

Très fréquente en entretien.

```text
Thread
→ unité d'exécution

Task
→ abstraction représentant une opération asynchrone
```

Une `Task` n'est pas simplement synonyme de "nouveau thread".

Pour une opération I/O :

```text
HTTP
Database
File
```

l'asynchronisme permet notamment d'éviter de bloquer inutilement un thread pendant l'attente.

---

# 38. `Task` vs `Task<T>`

```csharp
Task
```

représente une opération asynchrone sans résultat.

```csharp
Task<User>
```

représente une opération asynchrone qui produira un `User`.

Exemple :

```csharp
public async Task SaveAsync()
{
}
```

et :

```csharp
public async Task<User> GetUserAsync()
{
    ...
}
```

---

# 39. Pourquoi éviter `async void` ?

Sauf cas particuliers comme les handlers d'événements, on préfère :

```csharp
Task
```

ou :

```csharp
Task<T>
```

Exemple :

```csharp
public async Task SaveAsync()
```

plutôt que :

```csharp
public async void SaveAsync()
```

`Task` permet notamment à l'appelant de :

```text
await
gérer les exceptions
composer les opérations
```

---

# 40. Collections importantes

### `List<T>`

Collection indexée dynamique.

```csharp
var users = new List<User>();
```

### `Dictionary<TKey,TValue>`

Association clé → valeur.

```csharp
var users = new Dictionary<int, User>();
```

### `HashSet<T>`

Collection de valeurs uniques.

```csharp
var ids = new HashSet<int>();
```

Mental model :

```text
List
→ ordre / index

Dictionary
→ clé → valeur

HashSet
→ unicité / recherche par appartenance
```

---

# 41. `StringBuilder`

Pour construire beaucoup de texte progressivement, `StringBuilder` peut être préférable à de nombreuses concaténations répétées.

```csharp
var builder = new StringBuilder();

builder.Append("Hello ");
builder.Append("World");

var result = builder.ToString();
```

Pourquoi ?

`string` est immutable.

Les opérations répétées de concaténation peuvent donc créer de nouvelles chaînes.

---

# 42. `string` est immutable

Exemple :

```csharp
string name = "Ahlame";

name += " Mohsine";
```

La chaîne originale n'est pas modifiée.

Une nouvelle valeur de chaîne est créée.

Mental model :

```text
string
→ immutable
```

Cela explique notamment pourquoi `StringBuilder` peut être utile pour certaines constructions répétitives.

---

# 43. `==` vs `Equals`

Pour les types et objets, le comportement de `==` dépend du type et de ses opérateurs définis.

`Equals()` est une méthode utilisée pour comparer l'égalité selon les règles du type.

Il faut donc éviter de retenir :

```text
== = référence
Equals = valeur
```

comme une règle universelle.

Pour les `record`, par exemple, l'égalité par valeur est conçue comme une caractéristique importante.

---

# 44. SOLID

Les cinq principes :

```text
S
Single Responsibility

O
Open/Closed

L
Liskov Substitution

I
Interface Segregation

D
Dependency Inversion
```

Ils cherchent notamment à favoriser :

```text
cohésion
faible couplage
extensibilité
testabilité
maintenabilité
```

Il faut surtout savoir expliquer les principes avec des exemples plutôt que réciter les cinq lettres.

---

# 45. Dependency Inversion

Principe :

> Les modules de haut niveau ne doivent pas dépendre directement des détails de bas niveau ; les deux doivent dépendre d'abstractions.

Exemple :

Mauvais :

```csharp
public class OrderService
{
    private readonly SqlPaymentService _payment;
}
```

Plus flexible :

```csharp
public class OrderService
{
    private readonly IPaymentService _payment;

    public OrderService(IPaymentService payment)
    {
        _payment = payment;
    }
}
```

Puis :

```csharp
builder.Services.AddScoped<
    IPaymentService,
    SqlPaymentService>();
```

---

# 46. Questions d'entretien rapides

### Pourquoi utiliser une interface ?

> Pour définir un contrat et permettre notamment de réduire le couplage entre les classes.

### Qu'est-ce que le polymorphisme ?

> La possibilité de manipuler différentes implémentations à travers une abstraction commune, tout en obtenant le comportement correspondant à l'objet réel.

### `var` est-il du typage dynamique ?

> Non. `var` utilise l'inférence de type à la compilation. Le type reste statique.

### `dynamic` est-il identique à `var` ?

> Non. `dynamic` reporte certaines résolutions à l'exécution.

### Pourquoi utiliser les generics ?

> Pour écrire du code réutilisable tout en conservant le typage et en réduisant les conversions.

### Quelle différence entre `IEnumerable` et `IQueryable` ?

> `IEnumerable` est principalement utilisé pour parcourir des données déjà disponibles côté mémoire, tandis que `IQueryable` permet à un provider de construire et traduire une requête vers une source de données.

### Pourquoi utiliser `async/await` ?

> Pour gérer efficacement les opérations asynchrones, notamment les opérations I/O, sans bloquer inutilement un thread pendant l'attente.

### Pourquoi `Task` plutôt que `Thread` ?

> `Task` représente une opération asynchrone et fournit une abstraction plus haut niveau que la gestion directe d'un thread.

### Pourquoi préférer la composition à l'héritage dans de nombreux cas ?

> Parce qu'elle permet généralement de réduire le couplage et de remplacer plus facilement les composants utilisés.

---

# 47. Pièges classiques en entretien

## "Une Task crée forcément un thread"

Faux.

```text
Task
≠
Thread
```

---

## "`var` signifie dynamic"

Faux.

```text
var
→ type connu à la compilation

dynamic
→ résolution dynamique
```

---

## "Un reference type est toujours sur le heap"

Trop simpliste.

La réalité dépend notamment du runtime, du contexte et des optimisations.

---

## "IQueryable est toujours meilleur qu'IEnumerable"

Faux.

`IQueryable` est utile lorsqu'une source fournit un provider capable de traduire l'expression.

Il faut ensuite considérer :

```text
performance
complexité
exécution
matérialisation
```

---

## "async rend automatiquement le code plus rapide"

Faux.

L'asynchronisme n'accélère pas magiquement une opération.

Il permet notamment de mieux utiliser les ressources pendant l'attente d'opérations I/O.

---

# À retenir

```text
class
→ définition d'un type

object
→ instance

interface
→ contrat

polymorphism
→ plusieurs implémentations derrière une abstraction

encapsulation
→ protéger l'état interne

composition
→ utiliser d'autres objets

var
→ inférence statique

dynamic
→ résolution dynamique

generic
→ code réutilisable et typé

LINQ
→ requêtes / transformations

IEnumerable
→ données côté mémoire

IQueryable
→ requête composable avec un provider

delegate
→ référence typée vers une méthode

event
→ notification aux abonnés

record
→ données + value equality

Task
→ opération asynchrone

Thread
→ unité d'exécution

async/await
→ modèle d'asynchronisme

SOLID
→ principes de conception
```

## Phrase à mémoriser

> **En entretien C#, il faut montrer qu'on comprends les abstractions derrière le langage : les types et leur comportement, le polymorphisme et la composition, le typage générique, LINQ, les delegates/events et surtout le modèle asynchrone avec `Task` et `async/await`, plutôt que de simplement réciter la syntaxe.**
