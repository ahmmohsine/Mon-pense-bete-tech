# C#

Cette section regroupe les concepts fondamentaux du langage C# nécessaires pour développer des applications .NET modernes.

L'objectif n'est pas seulement de mémoriser la syntaxe, mais de comprendre comment les mécanismes du langage fonctionnent et dans quelles situations les utiliser.

---

## Concepts

### Asynchronisme

- [Async / Await](async-await.md)

Comprendre `Task`, `Task<T>`, `async`, `await`, le ThreadPool, les opérations I/O et les différences entre concurrence et parallélisme.

### LINQ

- [LINQ](linq.md)

Comprendre les méthodes LINQ, les expressions lambda, l'exécution différée, `IEnumerable<T>`, `IQueryable<T>` et les pièges fréquents.

### Delegates et Events

- [Delegates & Events](delegates-events.md)

Comprendre les delegates, les méthodes anonymes, les expressions lambda, les événements et leur utilisation dans .NET.

### Generics

- [Generics](generics.md)

Comprendre les types génériques, pourquoi ils existent, les contraintes génériques et leurs avantages en termes de réutilisabilité et de sécurité de typage.

### Records

- [Records](records.md)

Comprendre les `record`, `record class`, `record struct`, l'égalité par valeur et les différences avec les classes traditionnelles.

### Nullable Reference Types

- [Nullable Reference Types](nullable-reference-types.md)

Comprendre `string` vs `string?`, le rôle du compilateur, le `null` et les mécanismes permettant de réduire les `NullReferenceException`.

### Pattern Matching

- [Pattern Matching](pattern-matching.md)

Comprendre `is`, `switch`, les property patterns, relational patterns et les possibilités modernes du pattern matching en C#.

### Exceptions

- [Exceptions](exceptions.md)

Comprendre le fonctionnement des exceptions, `try/catch/finally`, la propagation des exceptions, les bonnes pratiques et les erreurs fréquentes.

### Collections

- [Collections](collections.md)

Comprendre les principales collections .NET : `List<T>`, `Dictionary<TKey,TValue>`, `HashSet<T>`, `Queue<T>`, `Stack<T>` et leurs compromis.

---

## Méthode de lecture des fiches

Chaque fiche importante suit autant que possible cette structure :

1. **Définition**  
   Qu'est-ce que le concept ?

2. **Pourquoi ?**  
   Quel problème cherche-t-il à résoudre ?

3. **Fonctionnement**  
   Que se passe-t-il derrière la syntaxe lorsque c'est pertinent ?

4. **Exemple C#**  
   Un exemple concret et représentatif.

5. **Erreurs fréquentes**  
   Les erreurs à reconnaître et comprendre.

6. **Comparaisons**  
   Les différences importantes entre plusieurs solutions.

7. **Règle mentale**  
   Une manière simple de mémoriser le concept.

8. **À retenir**  
   Les points essentiels à connaître.

9. **Questions d'entretien**  
   Lorsque le concept s'y prête, quelques questions permettant de vérifier sa compréhension.

---

## Ordre conseillé

Pour construire progressivement les bases C# :

1. Collections
2. Generics
3. LINQ
4. Exceptions
5. Delegates & Events
6. Records
7. Nullable Reference Types
8. Pattern Matching
9. Async / Await

Cet ordre n'est pas obligatoire. Certaines notions se recoupent et peuvent être étudiées en parallèle.

---

## Objectif

À terme, cette section doit permettre de retrouver rapidement une notion C# et surtout de comprendre :

- ce que fait le langage ;
- pourquoi une fonctionnalité existe ;
- comment elle fonctionne ;
- quand l'utiliser ;
- quand l'éviter ;
- quelles erreurs elle peut provoquer ;
- comment expliquer le concept à quelqu'un d'autre.
