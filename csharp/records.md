# Records en C#

Les `record` sont des types C# conçus notamment pour représenter des données dont la valeur et l'égalité sont importantes.

Ils sont particulièrement utiles pour comprendre :

- l'égalité par valeur ;
- les types immuables ;
- `record class` ;
- `record struct` ;
- les propriétés `init` ;
- `with` ;
- la différence entre `record` et `class` ;
- la représentation de données dans les DTOs.

---

# 1. Pourquoi les records ?

Avec une classe classique :

```csharp
public class User
{
    public string Name { get; set; }
}
```

Deux objets contenant les mêmes données ne sont pas automatiquement considérés comme égaux :

```csharp
var user1 = new User { Name = "Alice" };
var user2 = new User { Name = "Alice" };

Console.WriteLine(user1 == user2);
```

Pour une classe classique, `==` compare normalement les références, sauf si l'égalité a été redéfinie.

Les records ont été conçus pour faciliter les types dont l'identité dépend principalement des données qu'ils contiennent.

---

# 2. Déclarer un record

Syntaxe simple :

```csharp
public record User(string Name, int Age);
```

On peut créer :

```csharp
var user = new User("Alice", 25);
```

Le record contient les données déclarées dans le constructeur primaire.

---

# 3. Égalité par valeur

Avec :

```csharp
public record User(string Name, int Age);
```

on peut écrire :

```csharp
var user1 = new User("Alice", 25);
var user2 = new User("Alice", 25);

Console.WriteLine(user1 == user2);
```

Le résultat est :

```text
True
```

Le record compare les valeurs qui participent à son égalité.

### Mental model

```text
class
→ identité de l'objet

record
→ valeur / données de l'objet
```

Ce modèle est simplifié mais très utile pour comprendre l'intention.

---

# 4. `record` est-il toujours immutable ?

Il faut être précis.

Un record est conçu pour faciliter les modèles de données immuables, mais le mot `record` ne signifie pas :

> absolument impossible à modifier.

Avec :

```csharp
public record User(string Name, int Age);
```

les propriétés générées sont généralement utilisables avec une initialisation et ne sont pas de simples propriétés `set` ordinaires.

Mais on peut écrire un record avec des propriétés mutables :

```csharp
public record User
{
    public string Name { get; set; } = "";
}
```

Donc :

```text
record
≠
immuabilité garantie dans toutes les situations
```

L'intention habituelle reste toutefois la représentation de données par valeur.

---

# 5. `init`

Les propriétés `init` peuvent être affectées pendant la création de l'objet, mais pas normalement modifiées après son initialisation.

Exemple :

```csharp
public class User
{
    public string Name { get; init; } = "";
}
```

On peut :

```csharp
var user = new User
{
    Name = "Alice"
};
```

Mais ensuite :

```csharp
user.Name = "Bob";
```

n'est pas autorisé.

Les records utilisent fréquemment ce style d'initialisation.

---

# 6. Pourquoi l'immuabilité est intéressante ?

Prenons :

```csharp
var user = new User("Alice", 25);
```

Si l'objet est traité comme une valeur immuable, on sait que son état ne change pas de manière inattendue.

Cela facilite notamment :

- le raisonnement sur le code ;
- les transformations ;
- le partage d'objets ;
- la programmation fonctionnelle ;
- certains traitements concurrents.

---

# 7. `with`

Une fonctionnalité très importante des records est :

```csharp
with
```

Exemple :

```csharp
var user1 = new User("Alice", 25);

var user2 = user1 with
{
    Age = 26
};
```

Conceptuellement :

```text
user1
Alice / 25
   ↓ with
user2
Alice / 26
```

Le premier objet n'est pas simplement modifié pour devenir le deuxième.

On crée une nouvelle valeur à partir de l'ancienne avec certaines propriétés différentes.

---

# 8. Mental model de `with`

Pense :

```text
"Copie-moi cet objet,
mais avec ces valeurs différentes."
```

Exemple :

```csharp
var updatedUser = user with
{
    Name = "Sarah"
};
```

C'est très pratique pour les modèles immuables.

---

# 9. `record class`

Un record peut être explicitement déclaré comme :

```csharp
public record class User(string Name, int Age);
```

C'est un record de type référence.

Quand on écrit simplement :

```csharp
public record User(string Name, int Age);
```

on obtient également un `record class`.

Donc :

```csharp
record User(...)
```

et :

```csharp
record class User(...)
```

ont le même modèle de type de référence.

---

# 10. `record struct`

C# permet aussi :

```csharp
public record struct Point(int X, int Y);
```

Ici, il s'agit d'un type valeur.

Conceptuellement :

```text
record class
→ référence

record struct
→ valeur
```

Cela permet d'utiliser les fonctionnalités d'égalité par valeur et de représentation des données des records avec un type `struct`.

---

# 11. `readonly record struct`

On peut également déclarer :

```csharp
public readonly record struct Point(int X, int Y);
```

Cela exprime davantage l'intention d'avoir une valeur immuable.

Exemple :

```csharp
var point = new Point(10, 20);
```

Les propriétés générées ne sont pas destinées à être modifiées comme des propriétés mutables classiques.

---

# 12. Record vs class

Une comparaison simplifiée :

| | `class` | `record class` |
|---|---|---|
| Type | référence | référence |
| Égalité par défaut | identité/référence | valeur |
| `with` | non | oui |
| Usage typique | objets avec identité/comportement | données / valeurs |
| Immuabilité | non | facilitée |

Il ne faut pas conclure :

> record = meilleur que class.

Ce sont deux outils pour des intentions différentes.

---

# 13. Exemple concret : entité vs DTO

Dans une application .NET :

```text
Entity
→ représente généralement une identité persistée

DTO
→ transporte des données
```

Un DTO peut être un bon candidat pour un record :

```csharp
public record UserDto(
    int Id,
    string Name,
    string Email);
```

Une entité EF Core, en revanche, est souvent représentée par une `class` classique :

```csharp
public class User
{
    public int Id { get; set; }
    public string Name { get; set; } = "";
}
```

Ce n'est pas une règle absolue, mais c'est une distinction très utile.

---

# 14. Pourquoi un DTO peut être un record ?

Un DTO sert principalement à transporter une valeur :

```text
Id
Name
Email
```

On se soucie souvent davantage de :

```text
"Quelles sont les données ?"
```

que de l'identité objet au sens de référence.

Un record exprime donc bien cette intention.

Exemple :

```csharp
public record CreateUserRequest(
    string Name,
    string Email);
```

---

# 15. Déconstruction

Les records permettent également la déconstruction.

Exemple :

```csharp
public record User(string Name, int Age);
```

Puis :

```csharp
var user = new User("Alice", 25);

var (name, age) = user;
```

On obtient :

```text
name = "Alice"
age  = 25
```

Cela peut être pratique dans certains traitements.

---

# 16. `ToString()`

Les records fournissent une représentation textuelle utile basée sur leurs données.

Exemple :

```csharp
var user = new User("Alice", 25);

Console.WriteLine(user);
```

La représentation contient généralement le nom du type et les propriétés.

C'est souvent plus utile immédiatement que le `ToString()` par défaut d'une classe qui n'a pas été redéfini.

---

# 17. Égalité des records

Les records génèrent une logique d'égalité basée sur les membres concernés.

Exemple :

```csharp
public record Product(
    int Id,
    string Name);
```

Puis :

```csharp
var p1 = new Product(1, "Laptop");
var p2 = new Product(1, "Laptop");

bool same = p1 == p2;
```

Le résultat est `true`.

Si une valeur change :

```csharp
var p3 = new Product(1, "Phone");
```

alors :

```csharp
p1 == p3
```

est `false`.

---

# 18. Attention aux propriétés complexes

Il faut comprendre que l'égalité par valeur n'implique pas automatiquement une comparaison profondément récursive de toutes les structures possibles.

Exemple :

```csharp
public record User(List<string> Roles);
```

La manière dont `Roles` est comparée dépend de la sémantique d'égalité du type utilisé.

Une `List<string>` utilise sa propre logique d'égalité de référence plutôt qu'une égalité élément par élément simplement parce qu'elle se trouve dans un record.

### À retenir

```text
record
→ égalité structurée des membres selon leurs propres règles d'égalité
```

Ne pas imaginer :

```text
record
→ deep comparison magique de tout le graphe d'objets
```

---

# 19. Héritage

Les records peuvent participer à une hiérarchie de types.

Exemple :

```csharp
public record Animal(string Name);

public record Dog(string Name, string Breed)
    : Animal(Name);
```

Le type dérivé reste un record.

Cela permet de représenter certaines hiérarchies de données avec une égalité adaptée aux records.

---

# 20. `record` et pattern matching

Les records sont souvent utilisés avec le pattern matching.

Exemple :

```csharp
var user = new User("Alice", 25);

if (user is User("Alice", var age))
{
    Console.WriteLine(age);
}
```

Le record fournit des données qui se prêtent bien à cette forme de décomposition.

---

# 21. Record et immutabilité : attention à la shallow copy

Avec :

```csharp
var user2 = user1 with
{
    Age = 26
};
```

les membres de référence internes ne sont pas automatiquement clonés profondément.

Exemple conceptuel :

```text
user1
  ↓
Roles ──────┐
            │
user2       │
  ↓         │
Roles ──────┘
```

Le mécanisme `with` ne signifie donc pas :

> clone récursivement tout le graphe d'objets.

Il crée une nouvelle instance du record avec les valeurs de membres correspondantes.

---

# 22. Records et EF Core

Il faut éviter de conclure :

> Tous mes modèles EF Core doivent être des records.

Les entités de persistance ont généralement :

- une identité ;
- un cycle de vie ;
- des relations ;
- parfois un état mutable ;
- un suivi par EF Core.

Les DTOs et objets de transport sont souvent des candidats plus naturels pour les records.

### Mental model

```text
Entity
→ identité + cycle de vie

DTO / Value-like data
→ données à transporter
```

---

# 23. Record comme Value Object

Les records sont particulièrement intéressants pour représenter des valeurs.

Exemple :

```csharp
public record Money(decimal Amount, string Currency);
```

On peut avoir :

```csharp
var price1 = new Money(100, "EUR");
var price2 = new Money(100, "EUR");
```

Ces deux valeurs sont considérées comme égales selon la sémantique du record.

Cela correspond bien à l'idée de **Value Object** en Domain-Driven Design.

---

# 24. Attention : record n'est pas synonyme de Value Object

Un record peut faciliter la création d'un type orienté valeur.

Mais un vrai Value Object métier peut avoir :

- des invariants ;
- des règles métier ;
- une validation ;
- des opérations spécifiques.

Exemple :

```csharp
public record EmailAddress
{
    public string Value { get; }

    public EmailAddress(string value)
    {
        // validation
        Value = value;
    }
}
```

Le record aide pour l'égalité et la représentation, mais le domaine reste responsable des règles métier.

---

# 25. Record avec validation

Exemple :

```csharp
public record UserName
{
    public string Value { get; }

    public UserName(string value)
    {
        if (string.IsNullOrWhiteSpace(value))
        {
            throw new ArgumentException(
                "Name is required.",
                nameof(value));
        }

        Value = value;
    }
}
```

Le record peut donc contenir du comportement.

Il ne faut pas penser :

```text
record = simple sac de données obligatoire
```

Il peut également avoir des méthodes, propriétés calculées et validations.

---

# 26. Primary constructor

La syntaxe :

```csharp
public record User(string Name, int Age);
```

est une forme concise de déclaration.

Elle permet d'exprimer rapidement :

```text
type
+
données principales
```

On peut aussi écrire une version plus développée :

```csharp
public record User
{
    public string Name { get; init; }
    public int Age { get; init; }

    public User(string name, int age)
    {
        Name = name;
        Age = age;
    }
}
```

La syntaxe courte est particulièrement pratique pour les DTOs et modèles simples.

---

# 27. Record avec propriétés supplémentaires

Un record peut avoir d'autres propriétés :

```csharp
public record User(string Name, int Age)
{
    public bool IsAdult => Age >= 18;
}
```

On peut donc avoir :

```csharp
var user = new User("Alice", 25);

Console.WriteLine(user.IsAdult);
```

Le record n'est pas limité aux seules valeurs du constructeur primaire.

---

# 28. `with` et propriétés supplémentaires

Avec :

```csharp
public record User(string Name, int Age)
{
    public string Role { get; init; } = "User";
}
```

on peut :

```csharp
var admin = user with
{
    Role = "Admin"
};
```

Le `with` permet donc de produire une nouvelle instance avec des valeurs modifiées.

---

# 29. Record et référence

Même si un `record class` utilise l'égalité par valeur, il reste un type référence.

Donc :

```csharp
User user1 = ...;
User user2 = user1;
```

les deux variables peuvent référencer la même instance.

Le record ne change pas le fonctionnement fondamental d'une référence.

Il change notamment la manière dont l'égalité est définie.

### Important

```text
record class
→ type référence
+
égalité par valeur
```

---

# 30. Record struct et copie

Un `record struct` est un type valeur.

Par exemple :

```csharp
public record struct Point(int X, int Y);
```

Puis :

```csharp
var p1 = new Point(10, 20);
var p2 = p1;
```

`p2` reçoit une copie de la valeur.

Cela correspond aux règles générales des `struct`.

---

# 31. Quand utiliser un record ?

Un record est particulièrement intéressant pour :

- DTOs ;
- commandes ;
- requêtes ;
- réponses ;
- données immuables ;
- Value Objects ;
- objets dont l'égalité dépend des données.

Exemple :

```csharp
public record CreateOrderRequest(
    int ProductId,
    int Quantity);
```

---

# 32. Quand préférer une class ?

Une `class` est souvent plus naturelle pour :

- entités ayant une identité propre ;
- objets avec un cycle de vie ;
- objets fortement mutables ;
- objets dont l'égalité doit représenter l'identité ;
- modèles complexes dont la sémantique n'est pas naturellement celle d'une valeur.

Exemple :

```csharp
public class Order
{
    public int Id { get; set; }

    public decimal Total { get; private set; }

    public void AddItem(decimal price)
    {
        Total += price;
    }
}
```

Ici, l'objet possède un état et un comportement qui évoluent dans le temps.

---

# 33. Erreurs fréquentes

## Erreur 1 : penser que record = toujours immutable

Ce n'est pas garanti dans toutes les formes de déclaration.

---

## Erreur 2 : penser que record = struct

Un simple :

```csharp
record User(...)
```

est un `record class`.

Pour un type valeur :

```csharp
record struct Point(...)
```

---

## Erreur 3 : utiliser un record pour toutes les entités

Le record ne rend pas automatiquement un modèle meilleur pour EF Core.

---

## Erreur 4 : croire que `with` réalise un deep copy

`with` ne clone pas récursivement tous les objets référencés.

---

## Erreur 5 : confondre égalité par valeur et égalité profonde

Les membres utilisent leurs propres règles d'égalité.

---

## Erreur 6 : croire que record signifie "sans comportement"

Un record peut avoir :

- méthodes ;
- propriétés calculées ;
- validations ;
- logique métier.

---

# 34. Comparaison rapide

| Concept | `class` | `record class` | `record struct` |
|---|---|---|---|
| Type référence | Oui | Oui | Non |
| Type valeur | Non | Non | Oui |
| Égalité par valeur | Non par défaut | Oui | Oui |
| `with` | Non | Oui | Oui |
| Usage typique | Entité / objet métier | Données / DTO / valeur | Petite valeur structurée |
| Immuabilité facilitée | Non | Oui | Oui |

---

# 35. Mental model

Quand tu vois :

```csharp
public record User(string Name, int Age);
```

pense :

```text
Je définis une valeur composée de données
        ↓
égalité basée sur ces données
        ↓
création facile de nouvelles variantes avec `with`
```

Quand tu vois :

```csharp
user with { Age = 30 }
```

pense :

```text
"Nouvelle instance basée sur user,
mais avec Age = 30."
```

Quand tu vois :

```csharp
record struct
```

pense :

```text
record + sémantique de type valeur
```

---

# 36. À retenir

1. Un `record` est particulièrement adapté aux données dont la valeur est importante.
2. `record` crée par défaut un `record class`.
3. `record class` est un type référence.
4. `record struct` est un type valeur.
5. Les records fournissent une égalité orientée valeur.
6. `with` permet de créer une nouvelle instance à partir d'une autre avec des modifications.
7. `with` ne réalise pas un deep copy automatique.
8. Les records sont souvent utilisés pour les DTOs.
9. Les records sont également utiles pour les Value Objects.
10. Un record peut contenir du comportement.
11. `record` ne signifie pas automatiquement "absolument immutable".
12. Une entité EF Core n'a pas besoin d'être un record.
13. Il faut distinguer identité d'objet et valeur d'objet.
14. `record class` et `record struct` ont des sémantiques différentes.

---

# 37. Questions d'entretien

### Quelle est la différence principale entre une class et un record ?

Une classe utilise normalement une égalité basée sur l'identité/référence, tandis qu'un record est conçu pour une égalité basée sur les valeurs de ses membres.

---

### Un record est-il une struct ?

Non.

```csharp
record User(...)
```

déclare un `record class`.

Pour un type valeur :

```csharp
record struct Point(...)
```

---

### À quoi sert `with` ?

À créer une nouvelle instance à partir d'un record existant avec certaines valeurs modifiées.

---

### Pourquoi utiliser un record pour un DTO ?

Parce qu'un DTO représente souvent des données transportées, pour lesquelles l'égalité par valeur et une sémantique plus immuable sont utiles.

---

### Un record est-il toujours immutable ?

Non. Le mot `record` ne garantit pas à lui seul que tous les membres seront impossibles à modifier.

---

### Peut-on utiliser un record avec EF Core ?

Oui, mais il faut choisir le type de modèle en fonction de sa sémantique. Les entités persistées sont souvent des classes, tandis que les DTOs et objets orientés valeur sont des candidats naturels aux records.

---

### Quelle est la différence entre `record class` et `record struct` ?

`record class` est un type référence.

`record struct` est un type valeur.

---

# 38. La phrase à mémoriser

> **Une class représente souvent une identité et un cycle de vie ; un record représente souvent une valeur dont les données définissent l'égalité.**
