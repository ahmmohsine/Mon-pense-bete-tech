# Generics en C#

Les **génériques** permettent d'écrire du code qui fonctionne avec différents types tout en conservant un typage fort.

On les reconnaît notamment avec la syntaxe :

```csharp
<T>
```

Exemples très fréquents dans .NET :

```csharp
List<int>
List<string>
Dictionary<int, User>
IEnumerable<User>
Task<User>
```

Dans :

```csharp
List<User>
```

`User` est le type utilisé par le générique.

Dans :

```csharp
Task<User>
```

`Task` représente l'opération et `User` représente le type du résultat.

---

# 1. Pourquoi les génériques existent-ils ?

Imaginons que l'on veuille créer une méthode permettant d'afficher une valeur.

Sans génériques, on pourrait écrire :

```csharp
void Print(string value)
{
    Console.WriteLine(value);
}
```

Mais cette méthode ne fonctionne qu'avec `string`.

On pourrait créer plusieurs méthodes :

```csharp
void Print(string value)
{
    Console.WriteLine(value);
}

void Print(int value)
{
    Console.WriteLine(value);
}

void Print(decimal value)
{
    Console.WriteLine(value);
}
```

Cela devient rapidement répétitif.

Les génériques permettent d'exprimer :

> "Cette méthode fonctionne avec un type quelconque, mais le compilateur connaîtra précisément ce type."

```csharp
void Print<T>(T value)
{
    Console.WriteLine(value);
}
```

Utilisation :

```csharp
Print("Alice");
Print(42);
Print(12.5m);
```

Le même code peut fonctionner avec plusieurs types.

---

# 2. Que signifie `T` ?

`T` est simplement un nom conventionnel.

Il signifie généralement **Type**.

```csharp
public class Box<T>
{
    public T Value { get; set; }
}
```

Ici :

```text
T = un type qui sera déterminé plus tard
```

On peut ensuite construire :

```csharp
var numberBox = new Box<int>();
var stringBox = new Box<string>();
```

On obtient conceptuellement :

```text
Box<int>
    |
    +-- Value : int
```

et :

```text
Box<string>
    |
    +-- Value : string
```

`T` n'est donc pas un type concret.

C'est un **paramètre de type**.

---

# 3. Paramètre de type vs argument de type

Cette distinction est importante.

Dans :

```csharp
public class Box<T>
```

`T` est le **paramètre de type**.

Dans :

```csharp
Box<int>
```

`int` est l'**argument de type**.

Mentalement :

```text
Déclaration :

Box<T>
    ^
    paramètre de type


Utilisation :

Box<int>
    ^
    argument de type
```

C'est comparable à une méthode :

```csharp
void Print(string value)
```

où `value` est un paramètre.

Avec les génériques, le paramètre concerne le **type**.

---

# 4. Exemple avec une classe générique

```csharp
public class Repository<T>
{
    private readonly List<T> _items = new();

    public void Add(T item)
    {
        _items.Add(item);
    }

    public T? GetFirst()
    {
        return _items.FirstOrDefault();
    }
}
```

On peut créer :

```csharp
var userRepository = new Repository<User>();
var productRepository = new Repository<Product>();
```

On obtient deux utilisations différentes du même modèle générique.

```text
Repository<User>
    |
    +-- List<User>


Repository<Product>
    |
    +-- List<Product>
```

Le code du repository reste générique.

---

# 5. Pourquoi c'est mieux que `object` ?

On pourrait essayer de faire :

```csharp
public class Box
{
    public object Value { get; set; }
}
```

Cela accepte presque n'importe quel objet :

```csharp
var box = new Box();

box.Value = 42;
box.Value = "Alice";
```

Mais on perd une partie importante du typage.

Pour récupérer un `int` :

```csharp
int number = (int)box.Value;
```

Il faut effectuer un cast.

Avec un générique :

```csharp
public class Box<T>
{
    public T Value { get; set; }
}
```

Puis :

```csharp
var box = new Box<int>();

box.Value = 42;

int number = box.Value;
```

Aucun cast n'est nécessaire.

---

# 6. L'avantage principal : le typage à la compilation

Avec :

```csharp
var users = new List<User>();
```

le compilateur sait que cette liste contient des `User`.

Donc :

```csharp
users.Add(new User());
```

est valide.

Mais :

```csharp
users.Add("Alice");
```

provoquera une erreur de compilation.

C'est exactement ce que l'on veut.

Le compilateur peut détecter une erreur avant l'exécution.

### Règle mentale

> Les génériques permettent de réutiliser du code sans abandonner la sécurité du typage.

---

# 7. Génériques et `List<T>`

Tu as déjà rencontré :

```csharp
List<T>
```

Dans :

```csharp
List<User>
```

`List<T>` est une classe générique.

On peut regarder son idée de manière simplifiée :

```csharp
public class List<T>
{
    // stockage interne

    public void Add(T item)
    {
        // ...
    }

    public T this[int index]
    {
        get
        {
            // ...
        }
    }
}
```

Quand tu écris :

```csharp
List<User>
```

les opérations de la collection travaillent avec `User`.

Quand tu écris :

```csharp
List<int>
```

elles travaillent avec `int`.

---

# 8. Plusieurs paramètres génériques

Un type générique peut avoir plusieurs paramètres.

Exemple :

```csharp
Dictionary<TKey, TValue>
```

Il possède deux paramètres de type :

```text
TKey
TValue
```

Utilisation :

```csharp
Dictionary<int, User>
```

signifie :

```text
TKey   = int
TValue = User
```

Donc :

```csharp
users[42]
```

utilise :

```text
int -> User
```

---

# 9. Méthodes génériques

Les génériques ne concernent pas seulement les classes.

Une méthode peut également être générique :

```csharp
public T GetValue<T>(T value)
{
    return value;
}
```

Utilisation :

```csharp
int number = GetValue(42);

string name = GetValue("Alice");
```

Le compilateur peut souvent déterminer automatiquement le type de `T`.

C'est ce qu'on appelle **l'inférence de type**.

---

# 10. Inférence de type

Avec :

```csharp
var result = GetValue(42);
```

le compilateur déduit :

```text
T = int
```

Avec :

```csharp
var result = GetValue("Alice");
```

il déduit :

```text
T = string
```

On peut également préciser explicitement :

```csharp
var result = GetValue<int>(42);
```

La plupart du temps, l'inférence permet d'éviter cette syntaxe.

---

# 11. Contraintes génériques

Par défaut, un type générique peut être très général.

Mais parfois, on veut imposer certaines règles.

Exemple :

```csharp
public class Repository<T>
    where T : class
{
}
```

Cela signifie :

> `T` doit être un type référence.

Donc :

```csharp
Repository<User>
```

peut être valide si `User` est une classe.

Mais :

```csharp
Repository<int>
```

ne respecte pas cette contrainte.

---

# 12. Contrainte `struct`

On peut imposer un type valeur :

```csharp
public class Container<T>
    where T : struct
{
}
```

Exemples possibles :

```csharp
Container<int>
Container<DateTime>
```

---

# 13. Contrainte `new()`

On peut demander que le type possède un constructeur public sans paramètre :

```csharp
public class Factory<T>
    where T : new()
{
    public T Create()
    {
        return new T();
    }
}
```

La contrainte :

```csharp
where T : new()
```

autorise donc :

```csharp
new T()
```

Le compilateur sait que cette opération est possible.

---

# 14. Contrainte sur une classe ou une interface

On peut également imposer une classe de base :

```csharp
public class Repository<T>
    where T : Entity
{
}
```

Ou une interface :

```csharp
public class Repository<T>
    where T : IEntity
{
}
```

Cela permet au code générique de travailler avec les membres garantis par la contrainte.

Exemple :

```csharp
public interface IEntity
{
    int Id { get; }
}

public class Repository<T>
    where T : IEntity
{
    public int GetId(T entity)
    {
        return entity.Id;
    }
}
```

Pourquoi le compilateur accepte-t-il :

```csharp
entity.Id
```

?

Parce que la contrainte garantit que `T` implémente `IEntity`.

---

# 15. Plusieurs contraintes

On peut combiner plusieurs contraintes :

```csharp
public class Repository<T>
    where T : Entity, IEntity, new()
{
}
```

Cela impose plusieurs conditions à `T`.

Il faut respecter les règles de syntaxe et d'ordre des contraintes du langage.

L'idée essentielle est :

> Les contraintes indiquent au compilateur ce qu'il peut garantir sur `T`.

---

# 16. Pourquoi les contraintes sont importantes

Sans contrainte :

```csharp
public void Save<T>(T entity)
{
    entity.Id;
}
```

Le compilateur ne sait pas que `T` possède une propriété `Id`.

Pour lui :

```text
T = n'importe quel type
```

Avec :

```csharp
public void Save<T>(T entity)
    where T : IEntity
{
    entity.Id;
}
```

le compilateur sait :

```text
T possède au minimum les membres de IEntity
```

On peut donc accéder à :

```csharp
entity.Id
```

---

# 17. Génériques et héritage

Les génériques ne fonctionnent pas exactement comme l'héritage classique.

Supposons :

```csharp
class Animal
{
}

class Dog : Animal
{
}
```

On pourrait penser :

```text
Dog est un Animal
```

donc :

```csharp
List<Dog>
```

devrait être automatiquement convertible en :

```csharp
List<Animal>
```

Mais ce n'est pas le cas.

Ceci n'est pas autorisé :

```csharp
List<Dog> dogs = new();

List<Animal> animals = dogs;
```

Pourquoi ?

Parce que si cela était autorisé :

```csharp
animals.Add(new Cat());
```

on pourrait alors mettre un `Cat` dans une collection qui contient réellement des `Dog`.

Cela casserait la sécurité du typage.

---

# 18. Covariance

C'est là qu'intervient la **variance**.

Certaines interfaces génériques supportent la covariance.

Exemple :

```csharp
IEnumerable<Dog> dogs = ...;

IEnumerable<Animal> animals = dogs;
```

Cela fonctionne parce que `IEnumerable<out T>` est covariant.

Le mot clé :

```csharp
out
```

indique une relation de covariance dans la déclaration générique.

Mentalement :

```text
IEnumerable<Dog>
       |
       v
IEnumerable<Animal>
```

parce que l'interface ne permet pas d'ajouter arbitrairement un `Animal` dans la séquence.

Elle est principalement utilisée pour produire/lire des valeurs de type `T`.

---

# 19. Contravariance

La contravariance utilise :

```csharp
in
```

Elle concerne principalement les types génériques utilisés en entrée.

Exemple conceptuel :

```csharp
Action<Animal>
```

peut être utilisé là où :

```csharp
Action<Dog>
```

est attendu.

Pourquoi ?

Parce qu'une méthode capable de traiter n'importe quel `Animal` est également capable de traiter un `Dog`.

Mentalement :

```text
Animal
  |
  v
Dog

Une fonction qui accepte Animal
peut accepter Dog.
```

La covariance et la contravariance sont des notions plus avancées.

L'important est d'abord de retenir :

```text
out T = covariance
in T  = contravariance
```

---

# 20. Génériques et types valeur

Les génériques sont particulièrement intéressants avec les types valeur.

Exemple :

```csharp
List<int>
```

Un `int` est un type valeur.

Grâce aux génériques, on peut stocker des `int` dans une collection générique sans devoir les convertir explicitement en `object`.

Cela permet notamment d'éviter certains coûts liés au **boxing**.

---

# 21. Boxing et génériques

Prenons :

```csharp
int number = 42;

object value = number;
```

Ici, le `int` doit être représenté comme un objet.

C'est le **boxing**.

Conceptuellement :

```text
int
 |
 | boxing
 v
object
```

Puis :

```csharp
int number = (int)value;
```

nécessite un **unboxing**.

Les collections génériques comme :

```csharp
List<int>
```

permettent de conserver le type `int` sans utiliser `object` comme type de stockage général.

C'est une des raisons importantes pour lesquelles les génériques sont préférables à certaines anciennes API basées sur `object`.

---

# 22. Génériques et réutilisabilité

Sans générique :

```csharp
class UserRepository
{
    ...
}

class ProductRepository
{
    ...
}

class OrderRepository
{
    ...
}
```

Une partie du code peut être identique.

Avec un type générique :

```csharp
class Repository<T>
{
    ...
}
```

on peut réutiliser la structure.

```text
Repository<User>
Repository<Product>
Repository<Order>
```

Attention cependant :

> Générique ne signifie pas automatiquement "meilleure architecture".

Il faut éviter de créer des abstractions génériques uniquement pour éviter quelques lignes de code.

---

# 23. Erreur fréquente : utiliser `object` à la place des génériques

Mauvais réflexe :

```csharp
void Process(object value)
{
    ...
}
```

puis multiplier les casts :

```csharp
var user = (User)value;
```

Cela déplace une partie de la vérification vers l'exécution.

Avec un générique :

```csharp
void Process<T>(T value)
{
    ...
}
```

le compilateur conserve davantage d'informations sur le type.

### Règle mentale

> `object` dit : "je ne veux pas connaître précisément le type."
>
> `T` dit : "je ne connais pas encore le type, mais je veux que le compilateur le connaisse et le conserve."

---

# 24. Génériques et LINQ

LINQ utilise énormément les génériques.

Exemple :

```csharp
IEnumerable<User>
```

Puis :

```csharp
users.Where(u => u.IsActive);
```

`Where` est lui-même une méthode générique.

Conceptuellement :

```csharp
IEnumerable<T> Where<T>(
    IEnumerable<T> source,
    Func<T, bool> predicate)
```

La signature réelle est plus complexe, mais cette forme simplifiée permet de comprendre l'idée :

```text
T = User
```

Donc :

```text
IEnumerable<User>
```

reste :

```text
IEnumerable<User>
```

après le `Where`.

---

# 25. Génériques et `Task<T>`

Tu as également rencontré :

```csharp
Task<User>
```

Ici encore, `Task` est générique.

```text
Task<T>
```

signifie :

> Une opération asynchrone qui produira un résultat de type `T`.

Donc :

```csharp
Task<User>
```

signifie :

```text
opération asynchrone
        +
résultat de type User
```

Et :

```csharp
await task
```

permet d'obtenir le :

```text
User
```

---

# 26. Ce qui se passe conceptuellement derrière

Lorsqu'on écrit :

```csharp
List<User>
```

le compilateur et le runtime connaissent l'instanciation générique utilisée.

Le runtime .NET utilise un mécanisme appelé **reification des génériques** : les informations génériques font partie du type à l'exécution, contrairement à certaines implémentations de génériques d'autres langages qui effacent davantage ces informations.

Pour un développeur applicatif, l'idée importante est :

```text
List<User>
```

et :

```text
List<Product>
```

sont deux utilisations différentes du même type générique.

Le runtime peut également optimiser certaines instanciations, notamment pour les types valeur.

Il n'est donc pas correct de penser simplement :

> "Le compilateur remplace T par du texte."

Le fonctionnement réel est plus riche.

---

# 27. Quand utiliser les génériques ?

Les génériques sont particulièrement utiles lorsque :

### 1. Le même algorithme fonctionne avec plusieurs types

```csharp
T Find<T>(...)
```

### 2. Une structure stocke un type déterminé

```csharp
List<User>
Dictionary<int, User>
```

### 3. On veut conserver le typage

Plutôt que :

```csharp
object
```

on utilise :

```csharp
T
```

### 4. On veut imposer des capacités à `T`

Avec :

```csharp
where T : IEntity
```

---

# 28. Quand ne pas utiliser les génériques ?

Évite une abstraction générique lorsqu'elle rend le code plus complexe sans apporter de véritable réutilisation.

Exemple :

```csharp
GenericManager<T, TId, TResult, TOptions>
```

avec une logique très spécifique peut devenir difficile à comprendre.

Le but des génériques est de fournir :

- réutilisabilité ;
- sécurité de typage ;
- abstraction utile.

Pas de rendre les signatures compliquées.

---

# 29. Règle mentale

Quand tu vois :

```csharp
<T>
```

pense :

> **"Le code ne connaît pas encore le type exact, mais il veut le conserver de manière fortement typée."**

Exemples :

```text
List<T>
    → collection d'un type T

Task<T>
    → opération qui produit un T

IEnumerable<T>
    → séquence de T

Dictionary<TKey,TValue>
    → association entre deux types

Repository<T>
    → repository travaillant avec un type T
```

---

# 30. Résumé

- Un générique permet de travailler avec un type déterminé plus tard.
- `T` signifie généralement "Type".
- `T` est un paramètre de type.
- `int` dans `List<int>` est un argument de type.
- Les génériques conservent le typage fort.
- Ils évitent souvent les casts nécessaires avec `object`.
- Ils favorisent la réutilisation du code.
- Les contraintes `where` permettent de limiter les types acceptés.
- `class`, `struct`, `new()` et les interfaces/classes peuvent servir de contraintes.
- `List<T>`, `Dictionary<TKey,TValue>`, `IEnumerable<T>` et `Task<T>` utilisent les génériques.
- `List<Dog>` n'est pas automatiquement convertible en `List<Animal>`.
- `IEnumerable<Dog>` peut être utilisé comme `IEnumerable<Animal>` grâce à la covariance.
- `out` indique la covariance.
- `in` indique la contravariance.
- Les génériques permettent également d'éviter certains coûts de boxing avec les types valeur.
- Une abstraction générique doit rester utile et lisible.

---

# Questions d'entretien

1. Pourquoi les génériques existent-ils en C# ?
2. Quelle est la différence entre un paramètre de type et un argument de type ?
3. Que signifie `T` dans `List<T>` ?
4. Pourquoi `List<int>` est-elle préférable à une collection `object` dans de nombreux cas ?
5. Qu'est-ce qu'une contrainte générique ?
6. Que signifie `where T : class` ?
7. Que signifie `where T : new()` ?
8. Pourquoi `List<Dog>` ne peut-elle pas être assignée à `List<Animal>` ?
9. Qu'est-ce que la covariance ?
10. Que signifient `in` et `out` dans les interfaces génériques ?
11. Quel est le rôle des génériques dans `Task<T>` ?
12. Pourquoi `Dictionary<TKey,TValue>` possède-t-il deux paramètres génériques ?
13. Quelle différence entre utiliser `object` et utiliser `T` ?
14. Quel rapport existe-t-il entre les génériques et le boxing ?
15. Pourquoi ne faut-il pas créer des abstractions génériques uniquement pour éviter quelques lignes de code ?
