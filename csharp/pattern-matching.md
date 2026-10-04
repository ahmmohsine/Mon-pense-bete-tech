# Pattern Matching en C#

Le **pattern matching** permet à C# de tester la forme, le type ou la valeur d'une donnée tout en pouvant extraire les informations utiles.

Il est particulièrement important avec :

- `is`
- `switch`
- les propriétés ;
- les types ;
- les valeurs ;
- les relations (`>`, `<`, `>=`, `<=`) ;
- les listes ;
- les records ;
- les types nullable.

L'objectif est de comprendre le pattern matching comme un outil permettant d'écrire des conditions **expressives et sûres**.

---

# 1. Le problème

Sans pattern matching, on peut écrire :

```csharp
if (user != null)
{
    Console.WriteLine(user.Name);
}
```

Ou :

```csharp
if (value is string)
{
    var text = (string)value;
}
```

Le pattern matching permet de combiner le test et l'extraction :

```csharp
if (value is string text)
{
    Console.WriteLine(text);
}
```

On peut lire :

> Si `value` est une `string`, récupère-la dans `text`.

---

# 2. Le mot-clé `is`

Le pattern matching commence souvent avec :

```csharp
is
```

Exemple :

```csharp
if (value is string)
{
    Console.WriteLine("C'est une string.");
}
```

On vérifie ici le type réel de la valeur.

---

# 3. Type pattern

On peut tester un type :

```csharp
if (value is string)
{
}
```

Mais on peut également récupérer directement la valeur :

```csharp
if (value is string text)
{
    Console.WriteLine(text.Length);
}
```

La variable `text` est disponible dans le bloc parce que le test a confirmé le type.

### Mental model

```text
value
 ↓
est-ce une string ?
 ↓ oui
text = value convertie/sûre comme string
```

---

# 4. Pourquoi c'est mieux qu'un cast manuel ?

Ancienne approche :

```csharp
if (value is string)
{
    var text = (string)value;
}
```

Pattern matching :

```csharp
if (value is string text)
{
}
```

Le deuxième code combine :

```text
test du type
+
extraction
```

Cela réduit le code répétitif et améliore la lisibilité.

---

# 5. Pattern `null`

On peut tester :

```csharp
if (value is null)
{
    return;
}
```

Ou :

```csharp
if (value is not null)
{
    // value est non-null ici
}
```

Cette forme est particulièrement utile avec les Nullable Reference Types.

---

# 6. `is not`

Exemple :

```csharp
if (user is not null)
{
    Console.WriteLine(user.Name);
}
```

Lecture :

> Si `user` n'est pas null.

On peut aussi utiliser :

```csharp
if (value is not string)
{
    return;
}
```

---

# 7. `switch` avec pattern matching

Le pattern matching devient particulièrement puissant avec `switch`.

Exemple :

```csharp
string GetDescription(object value)
{
    return value switch
    {
        int => "Integer",
        string => "String",
        bool => "Boolean",
        _ => "Unknown"
    };
}
```

Chaque branche correspond à un pattern.

---

# 8. Le discard pattern `_`

Dans :

```csharp
_ => "Unknown"
```

`_` signifie :

> Tout ce qui n'a pas été capturé par les patterns précédents.

C'est souvent utilisé comme cas par défaut.

### Mental model

```text
cas 1
cas 2
cas 3
sinon _
```

---

# 9. Expression `switch` vs statement `switch`

Ancienne forme :

```csharp
switch (value)
{
    case int:
        ...
        break;

    case string:
        ...
        break;
}
```

Expression :

```csharp
var result = value switch
{
    int => "Integer",
    string => "String",
    _ => "Other"
};
```

La `switch expression` produit directement une valeur.

### Mental model

```text
switch statement
→ exécute des instructions

switch expression
→ produit une valeur
```

---

# 10. Constant pattern

On peut tester directement une valeur :

```csharp
if (status is "Active")
{
}
```

Ou :

```csharp
var message = status switch
{
    "Active" => "Utilisateur actif",
    "Blocked" => "Utilisateur bloqué",
    _ => "Statut inconnu"
};
```

Le pattern compare la valeur avec la constante.

---

# 11. Numeric constant patterns

Exemple :

```csharp
var result = age switch
{
    0 => "Nouveau-né",
    18 => "Majeur",
    _ => "Autre âge"
};
```

On peut donc comparer directement des valeurs.

---

# 12. Relational patterns

C# permet des patterns relationnels :

```csharp
var category = age switch
{
    < 18 => "Mineur",
    >= 18 => "Majeur"
};
```

Autres opérateurs :

```text
>
<
>=
<=
```

Exemple :

```csharp
var category = price switch
{
    <= 10 => "Cheap",
    <= 100 => "Normal",
    _ => "Expensive"
};
```

---

# 13. Logical patterns

On peut combiner des patterns.

Les principaux opérateurs sont :

```text
and
or
not
```

Exemple :

```csharp
if (age is >= 18 and < 65)
{
    Console.WriteLine("Adulte actif");
}
```

Lecture :

```text
age >= 18
ET
age < 65
```

---

# 14. `or`

Exemple :

```csharp
if (status is "Active" or "Pending")
{
}
```

Lecture :

```text
status == "Active"
OU
status == "Pending"
```

---

# 15. `not`

Exemple :

```csharp
if (value is not null)
{
}
```

On peut également combiner :

```csharp
if (value is not string)
{
}
```

Ou :

```csharp
if (age is not < 18)
{
}
```

---

# 16. Parenthèses

Lorsque les conditions deviennent complexes, les parenthèses rendent l'intention plus claire.

Exemple :

```csharp
if (age is >= 18 and (<= 30 or >= 60))
{
}
```

Il est préférable de privilégier la lisibilité plutôt que d'écrire des patterns trop compliqués sur une seule ligne.

---

# 17. Property patterns

On peut tester les propriétés d'un objet.

Exemple :

```csharp
if (user is
{
    IsActive: true
})
{
    Console.WriteLine("Utilisateur actif");
}
```

Le pattern vérifie :

```text
user
 ↓
IsActive == true ?
```

---

# 18. Plusieurs propriétés

Exemple :

```csharp
if (user is
{
    IsActive: true,
    Age: >= 18
})
{
    Console.WriteLine("Utilisateur adulte et actif.");
}
```

Le pattern vérifie plusieurs propriétés.

Conceptuellement :

```text
IsActive == true
ET
Age >= 18
```

---

# 19. Property pattern avec extraction

On peut également extraire une propriété :

```csharp
if (user is
{
    Name: string name
})
{
    Console.WriteLine(name);
}
```

Le pattern vérifie que `Name` est une `string` non-null et l'extrait dans `name`.

---

# 20. Nested property patterns

On peut descendre dans les objets imbriqués.

Exemple :

```csharp
if (order is
{
    Customer.Address.Country: "Belgium"
})
{
    Console.WriteLine("Client belge.");
}
```

Cela permet d'éviter certaines chaînes de conditions :

```csharp
if (order.Customer != null &&
    order.Customer.Address != null &&
    order.Customer.Address.Country == "Belgium")
{
}
```

Le pattern peut rendre l'intention plus compacte.

---

# 21. Positional patterns

Les positional patterns permettent de décomposer certaines valeurs.

Avec un record :

```csharp
public record Point(int X, int Y);
```

On peut écrire :

```csharp
if (point is (0, 0))
{
    Console.WriteLine("Origine");
}
```

Ou :

```csharp
var result = point switch
{
    (0, 0) => "Origine",
    (0, _) => "Axe X",
    (_, 0) => "Axe Y",
    _ => "Autre"
};
```

---

# 22. Pourquoi les records fonctionnent bien avec les patterns ?

Un record peut fournir les éléments nécessaires à la décomposition.

Exemple :

```csharp
public record User(string Name, int Age);
```

On peut écrire :

```csharp
if (user is User("Alice", var age))
{
    Console.WriteLine(age);
}
```

Le pattern permet de tester et d'extraire les données.

---

# 23. List patterns

C# permet également de faire du pattern matching sur des collections.

Exemple :

```csharp
int[] numbers = [1, 2, 3];

if (numbers is [1, 2, 3])
{
    Console.WriteLine("Séquence exacte");
}
```

On peut utiliser :

```text
[ ... ]
```

pour décrire la forme d'une séquence.

---

# 24. List pattern avec éléments

Exemple :

```csharp
if (numbers is [1, 2, 3])
{
}
```

Cela signifie que la séquence correspond à cette structure.

---

# 25. Discard dans un list pattern

On peut ignorer certaines valeurs :

```csharp
if (numbers is [1, _, 3])
{
}
```

Cela signifie :

```text
premier élément = 1
deuxième = n'importe quelle valeur
troisième = 3
```

---

# 26. Slice pattern `..`

Le pattern `..` représente une partie variable de la séquence.

Exemple :

```csharp
if (numbers is [1, .., 5])
{
}
```

Cela signifie conceptuellement :

```text
premier élément = 1
dernier élément = 5
éléments intermédiaires = n'importe quelle séquence
```

---

# 27. Capturer une slice

On peut également capturer la partie intermédiaire :

```csharp
if (numbers is [1, .. var middle, 5])
{
    Console.WriteLine(middle.Length);
}
```

Le nom exact de la variable dépend de la forme du pattern utilisée, mais l'idée est :

```text
[ début, .. milieu, fin ]
```

Le `..` capture une portion variable.

---

# 28. Type pattern + property pattern

Les patterns peuvent être combinés.

Exemple :

```csharp
if (value is User
{
    IsActive: true,
    Age: >= 18
})
{
    Console.WriteLine("Utilisateur valide.");
}
```

On vérifie simultanément :

```text
type = User
+
IsActive = true
+
Age >= 18
```

---

# 29. Pattern matching et polymorphisme

Supposons :

```csharp
abstract class Payment
{
}

class CreditCardPayment : Payment
{
}

class PaypalPayment : Payment
{
}
```

On peut faire :

```csharp
var message = payment switch
{
    CreditCardPayment => "Carte",
    PaypalPayment => "PayPal",
    _ => "Autre"
};
```

Cela permet de traiter différents types dérivés.

Cependant, il faut réfléchir à la conception : si chaque ajout de type oblige à modifier de nombreux `switch`, le polymorphisme ou une autre abstraction peut être plus approprié.

---

# 30. Pattern matching vs polymorphisme

Il ne faut pas considérer le pattern matching comme un remplacement systématique du polymorphisme.

### Pattern matching

Utile quand :

```text
la décision dépend de la forme/type/valeur
```

### Polymorphisme

Utile quand :

```text
chaque type possède son propre comportement
```

Exemple :

```text
Payment
  ↓
CreditCardPayment
  ↓
CalculateFee()
```

peut être plus naturel que :

```csharp
payment switch
{
    CreditCardPayment => ...,
    PaypalPayment => ...
}
```

si le comportement appartient réellement aux types.

---

# 31. Exhaustivité d'une switch expression

Une `switch expression` doit généralement couvrir tous les cas possibles.

Exemple :

```csharp
var result = value switch
{
    int => "Integer",
    string => "String",
    _ => "Other"
};
```

Le `_` couvre les autres cas.

Sans cas approprié, le compilateur peut signaler que l'expression n'est pas exhaustive.

Une expression `switch` non exhaustive peut également produire une `SwitchExpressionException` au runtime lorsqu'aucun pattern ne correspond.

---

# 32. Pattern matching et nullable

Exemple :

```csharp
User? user = GetUser();

var name = user switch
{
    null => "Unknown",
    { Name: var userName } => userName
};
```

Le `switch` traite explicitement le cas `null`.

Cela peut être plus lisible que plusieurs `if` lorsque plusieurs formes doivent être distinguées.

---

# 33. Pattern matching et `var`

On peut utiliser :

```csharp
if (value is var x)
{
}
```

Mais ce pattern correspond pratiquement à n'importe quelle valeur, y compris `null`.

Il faut donc éviter de l'utiliser sans raison.

Les patterns doivent exprimer une information utile.

---

# 34. `var` pattern et extraction

Dans certains patterns :

```csharp
if (value is User user)
{
}
```

le pattern fait à la fois :

```text
test
+
extraction
```

C'est l'un des usages les plus importants à retenir.

---

# 35. Pattern matching et casts

Ancienne approche :

```csharp
var text = value as string;

if (text != null)
{
    Console.WriteLine(text.Length);
}
```

Pattern matching :

```csharp
if (value is string text)
{
    Console.WriteLine(text.Length);
}
```

Le second exprime directement :

```text
"Si c'est une string, donne-moi la string."
```

---

# 36. `as` vs `is`

`as` :

```csharp
var user = value as User;
```

renvoie :

```text
User
ou
null
```

si la conversion échoue.

`is` avec pattern :

```csharp
if (value is User user)
{
}
```

permet :

```text
test
+
extraction
```

Pour les conditions, le pattern matching est souvent plus expressif.

---

# 37. Pattern matching et conditions complexes

Au lieu de :

```csharp
if (user != null &&
    user.IsActive &&
    user.Age >= 18)
{
}
```

on peut écrire :

```csharp
if (user is
{
    IsActive: true,
    Age: >= 18
})
{
}
```

Le second code décrit davantage la **forme attendue**.

---

# 38. Quand éviter un pattern trop complexe ?

Ce code :

```csharp
if (value is
{
    Customer:
    {
        Address:
        {
            Country: "Belgium",
            City: "Mons"
        }
    },
    Status: "Active"
})
{
}
```

peut être valide mais difficile à lire.

Il faut parfois extraire une méthode :

```csharp
bool IsEligible(Order order)
{
    ...
}
```

ou simplifier la logique.

### Règle

Le pattern matching doit améliorer la lisibilité, pas devenir un puzzle.

---

# 39. Erreurs fréquentes

## Erreur 1 : utiliser `is` uniquement comme test alors qu'on peut extraire

Au lieu de :

```csharp
if (value is string)
{
    var text = (string)value;
}
```

préférer :

```csharp
if (value is string text)
{
}
```

---

## Erreur 2 : oublier `_` dans une `switch expression`

Si tous les cas ne sont pas couverts, la switch peut être non exhaustive.

---

## Erreur 3 : utiliser `!` alors qu'un pattern peut vérifier la valeur

Au lieu de :

```csharp
user!.Name
```

si `user` est réellement nullable, envisager :

```csharp
if (user is User actualUser)
{
    Console.WriteLine(actualUser.Name);
}
```

---

## Erreur 4 : écrire des patterns illisibles

Un pattern très complexe n'est pas forcément meilleur qu'un `if` bien structuré.

---

## Erreur 5 : remplacer systématiquement le polymorphisme par `switch`

Le pattern matching est un outil, pas une architecture complète.

---

# 40. Comparaison rapide

| Syntaxe | Utilité |
|---|---|
| `is string` | tester un type |
| `is string text` | tester + extraire |
| `is null` | tester null |
| `is not null` | tester non-null |
| `x switch` | choisir selon des patterns |
| `int =>` | type pattern |
| `"Active" =>` | constant pattern |
| `>= 18` | relational pattern |
| `and` | combiner avec ET |
| `or` | combiner avec OU |
| `not` | inverser |
| `{ Age: >= 18 }` | property pattern |
| `(0, 0)` | positional pattern |
| `[1, 2, 3]` | list pattern |
| `..` | slice pattern |
| `_` | ignorer / cas général |

---

# 41. Mental model

Quand tu vois :

```csharp
value is Type variable
```

pense :

```text
TEST
+
EXTRACTION
```

Quand tu vois :

```csharp
value switch
{
    ...
}
```

pense :

```text
"Quelle forme possède cette valeur ?"
        ↓
"Quel résultat correspond à cette forme ?"
```

Quand tu vois :

```csharp
{ Age: >= 18 }
```

pense :

```text
"Un objet dont Age est au moins 18."
```

Quand tu vois :

```csharp
[1, .., 5]
```

pense :

```text
"Une séquence qui commence par 1,
se termine par 5,
avec n'importe quoi au milieu."
```

---

# 42. À retenir

1. Le pattern matching permet de tester et extraire des données.
2. `is Type variable` combine test de type et extraction.
3. `switch expression` permet de produire directement une valeur.
4. `_` représente le cas général.
5. Les constant patterns testent des valeurs.
6. Les relational patterns utilisent `>`, `<`, `>=`, `<=`.
7. `and`, `or` et `not` permettent de combiner les patterns.
8. Les property patterns testent les propriétés d'un objet.
9. Les positional patterns permettent la décomposition.
10. Les list patterns permettent de tester la forme d'une collection.
11. `..` permet de représenter une portion variable d'une séquence.
12. Le pattern matching fonctionne particulièrement bien avec les records.
13. Il peut simplifier la gestion des valeurs nullable.
14. Il ne remplace pas automatiquement le polymorphisme.
15. Un pattern doit améliorer la lisibilité, pas la compliquer.

---

# 43. Questions d'entretien

### Qu'est-ce que le pattern matching ?

C'est un mécanisme permettant de tester la forme, le type ou la valeur d'une donnée et éventuellement d'en extraire les informations utiles.

---

### Quelle est la différence entre `is string` et `is string text` ?

`is string` vérifie seulement le type.

`is string text` vérifie le type et extrait la valeur dans `text`.

---

### Quelle est la différence entre une `switch statement` et une `switch expression` ?

Une `switch statement` exécute des instructions.

Une `switch expression` produit une valeur.

---

### À quoi sert `_` ?

À représenter un cas général ou une valeur que l'on souhaite ignorer.

---

### Que sont les property patterns ?

Ils permettent de tester les propriétés d'un objet directement dans un pattern.

Exemple :

```csharp
user is { Age: >= 18 }
```

---

### Que signifie `and` dans un pattern ?

Les deux conditions doivent être satisfaites.

---

### Quand utiliser le pattern matching plutôt que le polymorphisme ?

Lorsque la décision dépend naturellement de la forme, du type ou de la valeur d'une donnée. Si chaque type possède son propre comportement, le polymorphisme peut être plus approprié.

---

### Pourquoi utiliser `is Type variable` plutôt qu'un cast après un test ?

Parce que le pattern réalise le test et l'extraction en une seule expression, avec une meilleure lisibilité et une analyse de type par le compilateur.

---

# 44. La phrase à mémoriser

> **Le pattern matching permet de demander : "À quoi ressemble cette valeur ?" puis de traiter directement le cas correspondant sans multiplier les casts et les conditions.**
