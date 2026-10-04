# Nullable Reference Types en C#

Les **Nullable Reference Types** (NRT) permettent au compilateur C# d'identifier plus tôt les situations où une référence pourrait être `null`.

Le but principal est de réduire les `NullReferenceException`.

L'idée essentielle est :

```text
string
→ je considère que la référence ne doit pas être null

string?
→ null est autorisé
```

Cette fonctionnalité repose principalement sur l'analyse du compilateur. Elle améliore la sécurité du code sans transformer `string` et `string?` en deux types CLR totalement différents.

---

# 1. Le problème de `null`

Prenons :

```csharp
string name = GetName();

Console.WriteLine(name.Length);
```

Si `GetName()` retourne `null`, on peut obtenir :

```text
NullReferenceException
```

Le problème est que l'erreur apparaît à l'exécution.

Les Nullable Reference Types cherchent à détecter ce genre de problème **à la compilation**.

---

# 2. `string` vs `string?`

Avec les Nullable Reference Types activés :

```csharp
string name;
```

signifie conceptuellement :

> `name` est censé toujours contenir une référence valide.

Alors que :

```csharp
string? name;
```

signifie :

> `name` peut contenir `null`.

Exemple :

```csharp
string name = "Alice";
```

valide.

```csharp
string? name = null;
```

valide.

Mais :

```csharp
string name = null;
```

produit généralement un avertissement du compilateur lorsque l'analyse nullable est activée.

---

# 3. Important : `string?` n'est pas un nouveau type CLR

Il faut éviter de penser :

```text
string
et
string?
```

comme deux classes totalement différentes au runtime.

Pour une référence, le `?` sert principalement au système d'analyse nullable du compilateur.

Conceptuellement :

```text
Compile-time
    ↓
analyse du risque de null
    ↓
warnings

Runtime
    ↓
référence CLR normale
```

C'est une distinction importante.

---

# 4. Pourquoi cette fonctionnalité existe ?

Avant les Nullable Reference Types, le compilateur pouvait difficilement savoir que :

```csharp
string name = null;
```

était probablement dangereux.

Le développeur devait constamment vérifier :

```csharp
if (name != null)
{
    Console.WriteLine(name.Length);
}
```

Les NRT permettent au compilateur de suivre plus précisément l'état nullable des références.

---

# 5. Activer les Nullable Reference Types

Dans un projet moderne .NET, on peut trouver :

```xml
<Nullable>enable</Nullable>
```

dans le `.csproj`.

Exemple :

```xml
<PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
</PropertyGroup>
```

Cela active l'analyse nullable pour le projet.

On peut également utiliser :

```csharp
#nullable enable
```

au niveau d'un fichier.

---

# 6. Warning vs exception

Avec :

```csharp
string name = null;
```

le compilateur peut signaler un warning.

Mais un warning n'empêche pas nécessairement la compilation.

Cela signifie :

```text
Le compilateur te prévient
        ↓
"Attention, cette référence pourrait être null."
```

Le développeur doit ensuite corriger ou justifier la situation.

---

# 7. Vérifier `null`

La vérification classique :

```csharp
if (name != null)
{
    Console.WriteLine(name.Length);
}
```

Après cette vérification, le compilateur comprend généralement que `name` n'est plus considéré comme nullable dans le bloc concerné.

C'est du **nullable flow analysis**.

---

# 8. Nullable flow analysis

Le compilateur suit l'état d'une variable.

Exemple :

```csharp
string? name = GetName();

if (name is not null)
{
    Console.WriteLine(name.Length);
}
```

Avant le `if` :

```text
name
↓
peut être null
```

Dans le bloc :

```text
name is not null
↓
le compilateur sait qu'il n'est pas null
```

Cette analyse permet de réduire les vérifications manuelles inutiles.

---

# 9. `is null` et `is not null`

On peut écrire :

```csharp
if (name is null)
{
    return;
}
```

Puis :

```csharp
Console.WriteLine(name.Length);
```

Le compilateur peut comprendre que `name` n'est plus nullable après le `return`.

On peut également écrire :

```csharp
if (name is not null)
{
    Console.WriteLine(name.Length);
}
```

---

# 10. Null-conditional operator `?.`

Le `?.` permet d'accéder à un membre seulement si la référence n'est pas `null`.

Exemple :

```csharp
string? name = GetName();

int? length = name?.Length;
```

Si :

```text
name = "Alice"
```

alors :

```text
length = 5
```

Si :

```text
name = null
```

alors :

```text
length = null
```

---

# 11. `?.` avec des méthodes

Exemple :

```csharp
user?.SendEmail();
```

Si `user` est `null`, la méthode n'est pas appelée.

Cela évite une exception dans ce cas.

Mais attention :

```csharp
user?.SendEmail();
```

ne signifie pas :

> L'utilisateur existe forcément.

Cela signifie :

> Si l'utilisateur existe, appelle la méthode.

---

# 12. Null-coalescing `??`

Le `??` permet de fournir une valeur de remplacement si une expression vaut `null`.

Exemple :

```csharp
string displayName = name ?? "Unknown";
```

Lecture :

```text
name existe ?
    oui → name
    non → "Unknown"
```

---

# 13. Null-coalescing assignment `??=`

Exemple :

```csharp
name ??= "Unknown";
```

Cela signifie :

> Si `name` est `null`, affecte `"Unknown"`.

Équivalent conceptuel :

```csharp
if (name is null)
{
    name = "Unknown";
}
```

---

# 14. Null-forgiving operator `!`

Le `!` peut indiquer au compilateur :

> Je sais que cette référence n'est pas null ici.

Exemple :

```csharp
string? name = GetName();

Console.WriteLine(name!.Length);
```

Attention :

```text
! ne vérifie pas que name n'est pas null.
```

Il indique seulement au compilateur de ne plus signaler le risque à cet endroit.

Si `name` vaut réellement `null`, une exception peut toujours se produire.

### Règle mentale

```text
? → "null est possible"

! → "fais-moi confiance"
```

Le `!` ne rend pas l'objet non-null au runtime.

---

# 15. Pourquoi `!` peut être dangereux

Exemple :

```csharp
User? user = FindUser();

var name = user!.Name;
```

Si :

```text
user = null
```

le `!` ne protège pas le programme.

Il masque seulement le warning.

Il faut donc préférer :

```csharp
if (user is null)
{
    return;
}

var name = user.Name;
```

lorsque l'absence est réellement possible.

---

# 16. Nullable value types

Le `?` existe également pour les types valeur :

```csharp
int?
```

est une forme de :

```csharp
Nullable<int>
```

Exemple :

```csharp
int? age = null;
```

Ici, la situation est différente des Nullable Reference Types.

```text
string?
→ référence pouvant être null

int?
→ Nullable<int>, type valeur pouvant représenter l'absence de valeur
```

Cette distinction est importante.

---

# 17. `int?` et `string?` ne fonctionnent pas exactement de la même manière

Pour :

```csharp
int? age;
```

le compilateur représente un `Nullable<int>`.

Conceptuellement :

```text
HasValue
Value
```

Pour :

```csharp
string? name;
```

il s'agit toujours d'une référence `string`, avec une annotation nullable utilisée par l'analyse du compilateur.

---

# 18. Exemple avec `int?`

```csharp
int? age = GetAge();

if (age.HasValue)
{
    Console.WriteLine(age.Value);
}
```

Mais on peut souvent utiliser directement le pattern :

```csharp
if (age is int actualAge)
{
    Console.WriteLine(actualAge);
}
```

ou :

```csharp
int result = age ?? 0;
```

---

# 19. Nullable dans les paramètres

Supposons :

```csharp
void PrintName(string name)
{
    Console.WriteLine(name);
}
```

Le contrat est :

```text
name ne devrait pas être null
```

Alors :

```csharp
PrintName(null);
```

produira généralement un warning avec les NRT activés.

Si `null` est réellement accepté :

```csharp
void PrintName(string? name)
{
    Console.WriteLine(name ?? "Unknown");
}
```

Le type exprime maintenant clairement le contrat.

---

# 20. Nullable dans les valeurs de retour

Exemple :

```csharp
User? FindUser(int id)
{
    ...
}
```

Le `?` dit à l'appelant :

> Cette méthode peut ne pas trouver d'utilisateur.

L'appelant doit donc gérer le cas `null`.

Exemple :

```csharp
User? user = FindUser(id);

if (user is null)
{
    return;
}

Console.WriteLine(user.Name);
```

---

# 21. NRT comme contrat

Les annotations nullable sont particulièrement utiles pour exprimer les contrats entre méthodes.

Exemple :

```csharp
public User? GetUser(int id)
```

signifie :

```text
Résultat :
    User
    OU
    null
```

Alors que :

```csharp
public User GetUser(int id)
```

exprime :

```text
Résultat :
    User non-null
```

Cela aide le développeur à comprendre immédiatement le contrat de la méthode.

---

# 22. Nullable et propriétés

Exemple :

```csharp
public class User
{
    public string Name { get; set; }
}
```

Le compilateur peut signaler que `Name` doit être initialisé.

Une solution :

```csharp
public class User
{
    public string Name { get; set; } = "";
}
```

ou :

```csharp
public string Name { get; init; } = "";
```

ou, si la propriété peut réellement être absente :

```csharp
public string? Name { get; set; }
```

Le choix doit correspondre au modèle métier.

---

# 23. Le problème de `required`

Dans les versions modernes de C#, on peut utiliser :

```csharp
public class User
{
    public required string Name { get; set; }
}
```

Cela signifie que l'appelant doit initialiser la propriété.

Exemple :

```csharp
var user = new User
{
    Name = "Alice"
};
```

Sans `Name`, le compilateur peut signaler une erreur.

### Mental model

```text
required
→ "tu dois fournir cette valeur lors de l'initialisation"

string?
→ "null est autorisé"

string
→ "null n'est pas prévu"
```

Ces concepts répondent à des problèmes différents.

---

# 24. Nullable et ASP.NET Core

Les NRT sont très utiles dans les Web APIs.

Exemple :

```csharp
public record UserDto(
    int Id,
    string Name,
    string? Phone);
```

On exprime :

```text
Id
→ obligatoire / non-null

Name
→ obligatoire / non-null

Phone
→ peut être null
```

Cela rend le contrat du modèle beaucoup plus clair.

---

# 25. Attention : nullable n'est pas automatiquement validation HTTP

Une annotation :

```csharp
string Name
```

indique au compilateur que la référence n'est pas censée être `null`.

Cela ne signifie pas automatiquement :

```text
HTTP 400
```

ou :

```text
validation métier complète
```

Il faut distinguer :

```text
Nullable Reference Types
→ analyse statique du code

Validation ASP.NET
→ validation des données entrantes

Base de données
→ contraintes de persistance
```

---

# 26. Nullable et Entity Framework Core

EF Core peut travailler avec les NRT.

Exemple :

```csharp
public class User
{
    public int Id { get; set; }

    public string Name { get; set; } = "";

    public string? Phone { get; set; }
}
```

L'intention est claire :

```text
Name
→ non-null

Phone
→ nullable
```

Avec EF Core, cette distinction peut également influencer la compréhension du modèle de données et la configuration des colonnes selon la configuration utilisée.

---

# 27. Nullable et relations EF Core

Exemple :

```csharp
public int DepartmentId { get; set; }

public Department Department { get; set; } = null!;
```

Le `null!` peut être utilisé lorsqu'une propriété sera initialisée par un mécanisme externe, par exemple EF Core, mais que le compilateur ne peut pas le déduire.

Cela mérite de la prudence.

```csharp
public Department Department { get; set; } = null!;
```

ne signifie pas :

> Department ne pourra jamais être null au runtime.

Cela signifie plutôt :

> Je demande au compilateur de me faire confiance concernant l'initialisation.

---

# 28. `null!` dans EF Core : mental model

Quand tu vois :

```csharp
public Department Department { get; set; } = null!;
```

pense :

```text
Le compilateur voit :
"Cette propriété pourrait être non initialisée."

Le développeur répond :
"EF Core l'initialisera ; ne me signale pas ce warning."
```

Mais si le programme utilise réellement la propriété alors qu'elle est null, le `!` ne protège pas contre une exception.

---

# 29. Pattern matching et nullable

On peut utiliser :

```csharp
if (user is User actualUser)
{
    Console.WriteLine(actualUser.Name);
}
```

Si `user` est nullable :

```csharp
User? user;
```

le pattern garantit dans le bloc que :

```text
actualUser
→ User non-null
```

C'est une manière élégante de combiner test et extraction.

---

# 30. Nullable et `switch`

Exemple :

```csharp
return name switch
{
    null => "Unknown",
    _ => name.ToUpper()
};
```

Le pattern matching permet de traiter explicitement le cas `null`.

---

# 31. Collections nullable

Attention à la différence entre :

```csharp
List<User> users
```

et :

```csharp
List<User>? users
```

Le premier signifie :

```text
la liste ne doit pas être null
```

mais ses éléments peuvent encore être contrôlés séparément.

Le deuxième signifie :

```text
la liste elle-même peut être null
```

On peut également avoir :

```csharp
List<User?> users
```

ce qui signifie :

```text
la liste existe
mais elle peut contenir des éléments null
```

Et :

```csharp
List<User?>? users
```

signifie :

```text
la liste peut être null
ET
ses éléments peuvent être null
```

---

# 32. C'est très important

Compare :

```csharp
List<User>?
```

et :

```csharp
List<User?> 
```

Ils ne veulent pas dire la même chose.

```text
List<User>?
       ↑
       liste nullable

List<User?>
           ↑
           éléments nullable
```

---

# 33. Nullable dans les propriétés imbriquées

Exemple :

```csharp
User? user;
```

On ne peut pas faire simplement :

```csharp
user.Address.City
```

si `user` ou `Address` peut être null.

On peut utiliser :

```csharp
user?.Address?.City
```

ou effectuer des vérifications explicites selon le comportement attendu.

---

# 34. Attention au `?.` en cascade

Ce code :

```csharp
var city = user?.Address?.City;
```

peut être pratique.

Mais si `City` est requis dans le contexte métier, transformer silencieusement l'absence en `null` peut cacher un problème.

Parfois il vaut mieux :

```csharp
if (user is null)
{
    throw new UserNotFoundException(...);
}
```

Le bon choix dépend du contrat métier.

### Règle

Ne pas utiliser `?.` uniquement pour faire disparaître les warnings.

Utilise-le lorsque `null` est réellement un résultat acceptable.

---

# 35. NRT et conception d'API

Les annotations nullable permettent de documenter implicitement les contrats.

Exemple :

```csharp
public User? FindUser(int id)
```

dit immédiatement :

```text
"Il est possible que l'utilisateur n'existe pas."
```

Alors que :

```csharp
public User GetUser(int id)
```

dit :

```text
"Cette méthode doit retourner un User."
```

Si aucun utilisateur n'existe, la méthode peut alors choisir un autre comportement :

```text
exception
Result<T>
```

selon l'architecture.

---

# 36. NRT ne supprime pas `NullReferenceException`

C'est très important.

Même avec :

```xml
<Nullable>enable</Nullable>
```

une `NullReferenceException` reste possible.

Exemple :

```csharp
User user = GetUserFromExternalLibrary();
```

Si une API externe retourne réellement `null` malgré le contrat :

```csharp
user.Name
```

peut toujours échouer.

Les NRT sont un système d'analyse statique, pas une protection runtime absolue.

---

# 37. Sources externes

Le compilateur ne peut pas toujours connaître parfaitement le comportement réel d'un système externe.

Exemple :

```csharp
var user = SomeExternalApi.GetUser();
```

Si les annotations de la bibliothèque sont incorrectes ou absentes, le compilateur peut avoir une vision imparfaite du risque.

Il faut donc conserver de bonnes pratiques runtime.

---

# 38. Erreurs fréquentes

## Erreur 1 : mettre `!` partout

```csharp
user!.Name
```

Le warning disparaît, mais le problème peut rester.

---

## Erreur 2 : utiliser `?` partout

```csharp
string? name
```

alors que le nom est obligatoire.

Cela affaiblit le contrat du code.

---

## Erreur 3 : confondre `string?` et `int?`

```text
string?
→ annotation nullable d'une référence

int?
→ Nullable<int>
```

---

## Erreur 4 : croire que nullable = validation HTTP

NRT et validation des données sont deux mécanismes différents.

---

## Erreur 5 : confondre liste nullable et éléments nullable

```csharp
List<User>?
```

n'est pas :

```csharp
List<User?>
```

---

## Erreur 6 : utiliser `?.` pour cacher une erreur métier

Si une valeur doit absolument exister, il vaut mieux traiter explicitement le problème.

---

# 39. Comparaison des opérateurs

| Syntaxe | Signification |
|---|---|
| `string` | référence censée être non-null |
| `string?` | référence pouvant être null |
| `?.` | accéder seulement si non-null |
| `??` | utiliser une valeur de remplacement si null |
| `??=` | affecter seulement si null |
| `!` | supprimer le warning nullable à cet endroit |
| `is null` | tester explicitement null |
| `is not null` | tester explicitement non-null |

---

# 40. Mental model complet

Quand tu vois :

```csharp
User?
```

pense :

```text
"Il peut ne pas y avoir de User."
```

Quand tu vois :

```csharp
user?.Name
```

pense :

```text
"Si user existe, donne-moi Name ; sinon null."
```

Quand tu vois :

```csharp
name ?? "Unknown"
```

pense :

```text
"Si name est null, utilise Unknown."
```

Quand tu vois :

```csharp
user!
```

pense :

```text
"Le compilateur, fais-moi confiance.
Mais le runtime ne me protège pas."
```

---

# 41. À retenir

1. Les Nullable Reference Types réduisent les risques de `NullReferenceException`.
2. `string` signifie que la référence est censée être non-null.
3. `string?` signifie que `null` est autorisé.
4. Le `?` des références est principalement une information utilisée par l'analyse du compilateur.
5. `int?` est un `Nullable<int>` et fonctionne différemment.
6. `?.` permet un accès conditionnel.
7. `??` fournit une valeur de remplacement.
8. `??=` affecte une valeur seulement si la variable est null.
9. `!` supprime un warning mais ne protège pas au runtime.
10. Le compilateur utilise le nullable flow analysis pour suivre les vérifications.
11. `List<User>?` signifie que la liste peut être null.
12. `List<User?>` signifie que les éléments peuvent être null.
13. NRT n'est pas la même chose que la validation ASP.NET.
14. NRT ne supprime pas toutes les `NullReferenceException`.
15. Il faut utiliser `?` pour exprimer un vrai contrat, pas pour faire disparaître des warnings.
16. Il faut éviter d'utiliser `!` comme solution systématique.

---

# 42. Questions d'entretien

### Quelle est la différence entre `string` et `string?` ?

Avec les NRT activés, `string` exprime une référence censée être non-null, tandis que `string?` exprime qu'une référence null est autorisée.

---

### Est-ce que `string?` est un nouveau type CLR ?

Non. Pour les références, le `?` sert principalement à l'analyse nullable du compilateur.

---

### Quelle est la différence entre `string?` et `int?` ?

`string?` est une annotation nullable pour une référence.

`int?` représente `Nullable<int>`, un type valeur pouvant représenter l'absence de valeur.

---

### Que fait l'opérateur `!` ?

Il indique au compilateur de ne plus considérer l'expression comme potentiellement null à cet endroit. Il ne réalise aucune vérification runtime.

---

### Pourquoi `!` peut-il être dangereux ?

Parce qu'il peut masquer un vrai risque de `null` sans modifier le comportement runtime.

---

### Quelle est la différence entre `List<User>?` et `List<User?>` ?

`List<User>?` signifie que la liste elle-même peut être null.

`List<User?>` signifie que la liste existe mais qu'elle peut contenir des références null.

---

### Les Nullable Reference Types empêchent-ils toutes les `NullReferenceException` ?

Non. Ils fournissent une analyse statique qui permet de détecter de nombreux risques, mais une erreur peut toujours se produire au runtime.

---

### Nullable Reference Types et validation ASP.NET sont-ils la même chose ?

Non.

Les NRT décrivent les contrats de nullabilité du code et permettent une analyse du compilateur.

La validation ASP.NET valide les données reçues par l'application.

---

# 43. La phrase à mémoriser

> **`?` décrit le contrat de nullabilité, `?.` gère un accès potentiellement null, `??` fournit une valeur de remplacement, et `!` dit au compilateur de me faire confiance sans me protéger au runtime.**
