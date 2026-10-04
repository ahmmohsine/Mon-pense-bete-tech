# Delegates et Events en C#

Les delegates et les events sont deux mécanismes fondamentaux de C# pour permettre à du code de transmettre ou d'appeler du comportement sans connaître directement l'implémentation de ce comportement.

Ils sont particulièrement importants pour comprendre :

- les lambdas ;
- LINQ ;
- les callbacks ;
- les événements ;
- `Action` et `Func` ;
- les événements .NET ;
- le fonctionnement de certains frameworks.

---

# 1. Le problème à comprendre

Imaginons une méthode :

```csharp
void Process()
{
    // traitement
}
```

Et imaginons que l'on souhaite dire :

> Quand `Process()` est terminé, exécute le comportement que l'appelant m'a fourni.

On pourrait avoir besoin d'un **callback**.

C'est précisément le genre de problème que les delegates permettent de résoudre.

---

# 2. Qu'est-ce qu'un delegate ?

Un delegate est un type qui représente une référence vers une méthode ayant une signature compatible.

Exemple :

```csharp
public delegate void MessageHandler(string message);
```

Cela définit un nouveau type de delegate.

Il peut représenter une méthode qui possède cette signature :

```text
string → void
```

Par exemple :

```csharp
void DisplayMessage(string message)
{
    Console.WriteLine(message);
}
```

On peut ensuite utiliser le delegate :

```csharp
MessageHandler handler = DisplayMessage;

handler("Bonjour");
```

---

# 3. Mental model

Ne pense pas :

```text
delegate = méthode
```

Pense plutôt :

```text
delegate = type qui peut référencer une méthode compatible
```

Par exemple :

```text
MessageHandler
       ↓
   peut référencer
       ↓
DisplayMessage
```

Le delegate impose la forme de la méthode.

---

# 4. Signature compatible

Avec :

```csharp
public delegate int Calculator(int a, int b);
```

la méthode doit être compatible :

```csharp
int Add(int a, int b)
{
    return a + b;
}
```

Mais ceci ne convient pas :

```csharp
string Add(int a, int b)
{
    return ...
}
```

car le type de retour ne correspond pas.

De même :

```csharp
int Add(int a)
```

ne possède pas la bonne signature.

### Règle

Le delegate définit le contrat du comportement qu'il peut référencer.

---

# 5. Appeler un delegate

```csharp
Calculator calculator = Add;

int result = calculator(10, 20);
```

On peut aussi écrire :

```csharp
int result = calculator.Invoke(10, 20);
```

Ces deux formes appellent le delegate.

```csharp
calculator(10, 20);
```

est simplement la syntaxe la plus naturelle.

---

# 6. Les delegates sont des objets

Un delegate est un type .NET.

Cela signifie qu'on peut le :

- stocker ;
- passer en paramètre ;
- retourner depuis une méthode ;
- combiner avec d'autres delegates ;
- invoquer.

Exemple :

```csharp
void Execute(Calculator calculator)
{
    int result = calculator(10, 20);
}
```

Puis :

```csharp
Execute(Add);
```

---

# 7. Pourquoi les delegates sont importants ?

Ils permettent de rendre un comportement configurable.

Exemple :

```csharp
void Process(int value, Func<int, int> transformation)
{
    int result = transformation(value);

    Console.WriteLine(result);
}
```

On peut appeler :

```csharp
Process(10, x => x * 2);
```

ou :

```csharp
Process(10, x => x + 5);
```

La méthode `Process` ne connaît pas à l'avance la transformation.

Elle reçoit le comportement.

### Mental model

```text
Donnée
 +
Comportement
 ↓
Traitement configurable
```

---

# 8. Delegate et lambda

Une lambda peut être utilisée lorsqu'un delegate est attendu.

Exemple :

```csharp
Func<int, int> square = x => x * x;
```

Ici :

```csharp
x => x * x
```

est une lambda.

Elle peut être convertie en :

```csharp
Func<int, int>
```

car `Func<int, int>` représente :

```text
int → int
```

---

# 9. `Action`

C# fournit des delegates génériques prêts à l'emploi.

`Action` représente une méthode qui retourne `void`.

Exemple :

```csharp
Action<string> print = message =>
{
    Console.WriteLine(message);
};
```

On peut ensuite :

```csharp
print("Hello");
```

Variantes :

```csharp
Action
Action<T>
Action<T1, T2>
...
```

Exemple :

```csharp
Action<int, int> addAndDisplay = (a, b) =>
{
    Console.WriteLine(a + b);
};
```

---

# 10. `Func`

`Func` représente une méthode qui retourne une valeur.

Exemple :

```csharp
Func<int, int> square = x => x * x;
```

Attention à la lecture :

```csharp
Func<int, int>
```

Le dernier type est le type de retour.

Donc :

```text
Func<int, int>
       │     │
       │     └── retour
       └──────── paramètre
```

Autre exemple :

```csharp
Func<int, int, int> add = (a, b) => a + b;
```

Signifie :

```text
int + int → int
```

---

# 11. `Predicate<T>`

C# possède également :

```csharp
Predicate<T>
```

Il représente une fonction qui reçoit un `T` et retourne un `bool`.

Exemple :

```csharp
Predicate<int> isPositive = x => x > 0;
```

C'est conceptuellement proche de :

```csharp
Func<int, bool>
```

---

# 12. Pourquoi LINQ utilise les delegates ?

Prenons :

```csharp
users.Where(u => u.Age >= 18);
```

La lambda :

```csharp
u => u.Age >= 18
```

représente le comportement :

```text
User → bool
```

Conceptuellement, `Where` reçoit donc quelque chose qui peut lui dire :

> Pour cet utilisateur, est-ce que je le conserve ?

C'est l'une des raisons pour lesquelles les delegates sont fondamentaux pour comprendre LINQ.

---

# 13. Delegate comme callback

Un callback est un comportement fourni à une méthode pour être appelé plus tard.

Exemple :

```csharp
void Process(Action onCompleted)
{
    Console.WriteLine("Traitement terminé.");

    onCompleted();
}
```

Appel :

```csharp
Process(() =>
{
    Console.WriteLine("Callback exécuté.");
});
```

Conceptuellement :

```text
Process
   ↓
travaille
   ↓
appelle le comportement fourni
   ↓
callback
```

---

# 14. Multicast delegates

Un delegate peut contenir plusieurs méthodes.

Exemple :

```csharp
Action action = Method1;
action += Method2;
action += Method3;
```

Lorsque :

```csharp
action();
```

est exécuté, les méthodes sont appelées selon l'ordre de la chaîne d'invocation.

Conceptuellement :

```text
action
  ↓
Method1
  ↓
Method2
  ↓
Method3
```

On parle de **multicast delegate**.

---

# 15. Retirer une méthode

On peut retirer une méthode :

```csharp
action -= Method2;
```

Il reste alors :

```text
Method1
Method3
```

---

# 16. Qu'est-ce qu'un event ?

Un `event` est construit autour du mécanisme des delegates.

Il permet à une classe de dire :

> Quelque chose vient de se produire ; les abonnés peuvent être prévenus.

Exemple :

```csharp
public event EventHandler? Completed;
```

Une autre partie du programme peut s'abonner :

```csharp
process.Completed += OnCompleted;
```

Et l'émetteur déclenche :

```csharp
Completed?.Invoke(this, EventArgs.Empty);
```

---

# 17. Delegate vs event

C'est une distinction très importante.

Un delegate permet notamment :

```csharp
handler();
```

Un event ajoute une restriction importante :

> Les abonnés extérieurs peuvent s'abonner ou se désabonner, mais l'événement est normalement déclenché par la classe qui le possède.

Exemple :

```csharp
public class Process
{
    public event EventHandler? Completed;

    public void Run()
    {
        // traitement

        Completed?.Invoke(this, EventArgs.Empty);
    }
}
```

L'extérieur fait :

```csharp
process.Completed += OnCompleted;
process.Completed -= OnCompleted;
```

mais il ne doit pas déclencher directement :

```csharp
process.Completed?.Invoke(...);
```

### Mental model

```text
delegate
→ comportement appelable

event
→ notification à laquelle on peut s'abonner
```

---

# 18. Pourquoi utiliser `event` ?

Supposons :

```csharp
public Action? Completed;
```

Une autre classe pourrait potentiellement remplacer le delegate :

```csharp
process.Completed = AnotherMethod;
```

Avec :

```csharp
public event EventHandler? Completed;
```

l'extérieur est limité au modèle d'abonnement :

```csharp
process.Completed += OnCompleted;
process.Completed -= OnCompleted;
```

Cela protège l'encapsulation.

---

# 19. Le pattern .NET standard

Le pattern traditionnel des événements .NET utilise :

```csharp
EventHandler
```

ou :

```csharp
EventHandler<TEventArgs>
```

Exemple :

```csharp
public event EventHandler? Completed;
```

Déclenchement :

```csharp
Completed?.Invoke(this, EventArgs.Empty);
```

---

# 20. Pourquoi `EventArgs` ?

Le premier argument permet généralement de connaître l'objet qui a déclenché l'événement :

```csharp
this
```

Le deuxième contient les données de l'événement.

Pour aucun détail particulier :

```csharp
EventArgs.Empty
```

Pour transmettre des données :

```csharp
public class OrderCreatedEventArgs : EventArgs
{
    public int OrderId { get; }

    public OrderCreatedEventArgs(int orderId)
    {
        OrderId = orderId;
    }
}
```

Puis :

```csharp
public event EventHandler<OrderCreatedEventArgs>? OrderCreated;
```

Déclenchement :

```csharp
OrderCreated?.Invoke(
    this,
    new OrderCreatedEventArgs(order.Id));
```

---

# 21. Abonnement à un event

Exemple :

```csharp
public class OrderService
{
    public event EventHandler? OrderCreated;

    public void CreateOrder()
    {
        // création

        OrderCreated?.Invoke(this, EventArgs.Empty);
    }
}
```

Abonnement :

```csharp
var service = new OrderService();

service.OrderCreated += OnOrderCreated;
```

Handler :

```csharp
void OnOrderCreated(object? sender, EventArgs e)
{
    Console.WriteLine("Commande créée.");
}
```

---

# 22. Désabonnement

On peut se désabonner :

```csharp
service.OrderCreated -= OnOrderCreated;
```

Le même delegate doit être utilisé pour retirer l'abonnement.

Cette notion est importante dans les applications qui ont des objets avec une durée de vie différente, car un abonnement peut conserver une référence vers le subscriber plus longtemps que prévu.

---

# 23. Pourquoi les events peuvent provoquer des problèmes de mémoire ?

Imaginons :

```text
Objet long-lived
      ↓ event
Objet court-lived
```

Si l'objet court-lived s'abonne et ne se désabonne jamais, l'objet long-lived peut conserver une référence vers lui via son événement.

Conceptuellement :

```text
Publisher
   ↓
event
   ↓
Subscriber
```

Tant que le publisher conserve l'abonnement, le subscriber peut rester référencé.

Il faut donc réfléchir à la durée de vie des objets et au désabonnement lorsque c'est nécessaire.

---

# 24. Events et encapsulation

Une bonne conception est :

```csharp
public event EventHandler? Completed;
```

Puis le déclenchement reste interne :

```csharp
public void Run()
{
    // ...

    Completed?.Invoke(this, EventArgs.Empty);
}
```

L'extérieur peut :

```csharp
Completed += ...
Completed -= ...
```

mais la classe reste responsable de décider **quand** l'événement se produit.

---

# 25. Delegate personnalisé ou `Action` / `Func` ?

On peut écrire :

```csharp
public delegate void MessageHandler(string message);
```

mais souvent :

```csharp
Action<string>
```

suffit.

### Utiliser `Action` / `Func`

Quand le comportement est générique :

```csharp
Func<User, bool>
```

### Utiliser un delegate nommé

Quand le nom apporte une vraie signification au domaine :

```csharp
public delegate bool UserAuthorizationHandler(User user);
```

Le nom peut rendre l'intention plus claire.

---

# 26. Delegate et type safety

Les delegates sont fortement typés.

Avec :

```csharp
Func<int, string> converter
```

le compilateur sait que :

```text
entrée  = int
sortie  = string
```

Ceci n'est donc pas valide :

```csharp
converter("hello");
```

Le compilateur détecte l'erreur.

---

# 27. Delegate et méthodes statiques

Un delegate peut référencer une méthode statique :

```csharp
static void Print(string message)
{
    Console.WriteLine(message);
}

Action<string> action = Print;
```

Il peut également référencer une méthode d'instance :

```csharp
class Printer
{
    public void Print(string message)
    {
        Console.WriteLine(message);
    }
}

var printer = new Printer();

Action<string> action = printer.Print;
```

Dans ce deuxième cas, le delegate représente à la fois le comportement et la cible de l'appel.

---

# 28. Delegate et méthodes d'instance : notion importante

Avec :

```csharp
Action action = printer.Print;
```

le delegate sait quelle méthode appeler sur quelle instance.

Conceptuellement :

```text
delegate
   ↓
instance = printer
method   = Print
```

C'est une façon utile de comprendre pourquoi un delegate d'instance peut conserver une référence vers l'objet concerné.

---

# 29. Delegates et covariance / contravariance

Les delegates génériques peuvent utiliser les concepts de variance.

Exemple :

```csharp
Action<Animal> handleAnimal;
Action<Dog> handleDog;
```

La contravariance permet dans certains cas d'utiliser un handler capable de traiter un type plus général là où un type plus spécifique est attendu.

Ce sujet devient particulièrement important lorsqu'on travaille avec :

- interfaces génériques ;
- delegates ;
- événements ;
- collections ;
- API génériques.

Pour l'instant, retiens surtout :

```text
out → covariance
in  → contravariance
```

et que les règles dépendent de la position du type générique.

---

# 30. Events et delegates ne sont pas la même chose

Il faut éviter de dire :

> Un event est simplement un delegate.

Plus précisément :

```text
Delegate
→ type représentant un ou plusieurs appels de méthodes

Event
→ mécanisme basé sur un delegate qui contrôle comment les autres
  parties peuvent s'abonner et se désabonner
```

Un `event` apporte donc une forme d'encapsulation autour du delegate.

---

# 31. Exemple complet

```csharp
public class DownloadService
{
    public event EventHandler? DownloadCompleted;

    public void Download()
    {
        Console.WriteLine("Téléchargement...");

        // Simulation du téléchargement

        DownloadCompleted?.Invoke(
            this,
            EventArgs.Empty);
    }
}
```

Utilisation :

```csharp
var service = new DownloadService();

service.DownloadCompleted += OnDownloadCompleted;

service.Download();
```

Handler :

```csharp
void OnDownloadCompleted(
    object? sender,
    EventArgs e)
{
    Console.WriteLine("Téléchargement terminé.");
}
```

Flux :

```text
Download()
   ↓
traitement
   ↓
DownloadCompleted.Invoke(...)
   ↓
OnDownloadCompleted()
```

---

# 32. Delegate + stratégie

Les delegates peuvent être utilisés pour injecter un comportement.

Exemple :

```csharp
decimal CalculatePrice(
    decimal price,
    Func<decimal, decimal> strategy)
{
    return strategy(price);
}
```

Utilisation :

```csharp
var discountedPrice =
    CalculatePrice(
        100,
        price => price * 0.8m);
```

La méthode ne connaît pas la stratégie.

Elle reçoit la stratégie.

Cela rejoint l'idée du **Strategy Pattern**, même si un delegate peut parfois suffire sans créer toute une hiérarchie de classes.

---

# 33. Delegate et Dependency Injection

Il ne faut pas confondre delegates et Dependency Injection.

Un delegate peut injecter :

```text
un comportement
```

La DI injecte généralement :

```text
une dépendance / abstraction
```

Exemple :

```csharp
void Process(Func<User, bool> filter)
```

injecte une fonction.

Alors que :

```csharp
public UserService(IUserRepository repository)
```

injecte une dépendance.

Les deux permettent de découpler du code, mais à des niveaux différents.

---

# 34. Delegates dans le code moderne .NET

Tu rencontreras souvent directement :

```csharp
Func<T, TResult>
Action<T>
Predicate<T>
```

et indirectement les delegates à travers :

```csharp
LINQ
events
callbacks
ASP.NET Core
middleware
```

Par exemple, dans ASP.NET Core :

```csharp
app.Use(async (context, next) =>
{
    await next();
});
```

Les lambdas utilisées par certaines APIs représentent des comportements transmis à des méthodes.

---

# 35. Erreurs fréquentes

## Erreur 1 : confondre delegate et méthode

Un delegate n'est pas une méthode.

```text
Méthode
→ implémentation

Delegate
→ type/référence permettant de représenter un comportement compatible
```

---

## Erreur 2 : confondre `event` et delegate public

```csharp
public Action? SomethingHappened;
```

n'a pas les mêmes garanties d'encapsulation que :

```csharp
public event EventHandler? SomethingHappened;
```

---

## Erreur 3 : oublier de se désabonner

Selon la durée de vie des objets :

```csharp
publisher.Event += Handler;
```

peut nécessiter :

```csharp
publisher.Event -= Handler;
```

---

## Erreur 4 : utiliser un event alors qu'on veut simplement retourner une valeur

Un event est adapté à une notification :

```text
"Quelque chose vient de se produire."
```

Il n'est pas nécessaire pour tous les échanges entre méthodes.

---

## Erreur 5 : utiliser des delegates partout

Un delegate est utile quand on veut rendre un comportement configurable.

Mais parfois une interface ou une classe dédiée est beaucoup plus lisible.

---

# 36. Comparaison rapide

| Concept | Rôle |
|---|---|
| Méthode | Implémente un comportement |
| Delegate | Représente un comportement compatible |
| `Action` | Delegate qui retourne `void` |
| `Func` | Delegate qui retourne une valeur |
| `Predicate<T>` | Delegate `T → bool` |
| Event | Notification avec abonnement |
| Lambda | Syntaxe concise pour créer un comportement |

---

# 37. Mental model à retenir

Quand tu vois :

```csharp
Func<User, bool>
```

pense :

```text
User
 ↓
bool
```

Quand tu vois :

```csharp
Action<User>
```

pense :

```text
User
 ↓
aucune valeur de retour
```

Quand tu vois :

```csharp
event EventHandler
```

pense :

```text
"Je peux m'abonner pour être prévenu."
```

Quand tu vois :

```csharp
+=
```

sur un event :

```text
abonnement
```

Quand tu vois :

```csharp
-=
```

sur un event :

```text
désabonnement
```

---

# 38. À retenir

1. Un delegate représente un comportement compatible avec une signature.
2. Les delegates sont fortement typés.
3. Une lambda peut être convertie en delegate compatible.
4. `Action` représente un comportement qui retourne `void`.
5. `Func` représente un comportement qui retourne une valeur.
6. `Predicate<T>` représente `T → bool`.
7. LINQ utilise énormément les delegates.
8. Un callback est souvent implémenté avec un delegate.
9. Un delegate peut référencer une méthode statique ou d'instance.
10. Un delegate peut contenir plusieurs méthodes.
11. Un `event` fournit un mécanisme d'abonnement à une notification.
12. L'extérieur peut normalement s'abonner ou se désabonner d'un event, mais ne le déclenche pas.
13. `EventHandler` et `EventHandler<TEventArgs>` sont les patterns classiques des événements .NET.
14. Les events renforcent l'encapsulation par rapport à un delegate public.
15. Les abonnements aux events doivent être pensés en fonction de la durée de vie des objets.

---

# 39. Questions d'entretien

### Qu'est-ce qu'un delegate ?

Un type fortement typé représentant une référence vers une ou plusieurs méthodes compatibles avec une signature donnée.

---

### Quelle est la différence entre `Action` et `Func` ?

`Action` retourne `void`.

`Func` retourne une valeur ; son dernier paramètre générique représente le type de retour.

---

### Pourquoi LINQ utilise-t-il des delegates ?

Pour recevoir des comportements comme les prédicats et les projections.

Par exemple :

```csharp
Where(u => u.Age >= 18)
```

fournit le comportement permettant de déterminer quels éléments conserver.

---

### Quelle est la différence entre un delegate et un event ?

Un delegate représente un comportement appelable.

Un event est un mécanisme de notification basé sur un delegate, avec des règles d'encapsulation permettant principalement aux consommateurs de s'abonner ou de se désabonner.

---

### Pourquoi utiliser un event plutôt qu'un delegate public ?

Parce qu'un event empêche normalement les consommateurs externes de remplacer ou déclencher directement la notification. La classe propriétaire conserve le contrôle du déclenchement.

---

### Qu'est-ce qu'un callback ?

C'est un comportement fourni à une méthode afin qu'elle puisse l'appeler à un moment donné.

---

### Pourquoi une lambda peut-elle être passée à `Where` ?

Parce que la lambda peut être convertie en un type de delegate compatible avec le paramètre attendu par `Where`.

---

# 40. La phrase à mémoriser

> **Un delegate représente un comportement ; une lambda permet de l'écrire rapidement ; un event permet de notifier des abonnés tout en gardant le contrôle du déclenchement.**
