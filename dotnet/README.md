# .NET

Cette section regroupe les mécanismes fondamentaux de la plateforme .NET utilisés quotidiennement dans les applications C# et ASP.NET Core.

L'objectif n'est pas seulement de mémoriser les méthodes, mais de comprendre ce que fait réellement .NET derrière le code.

## Fiches

* [Dependency Injection](dependency-injection.md)
* [Middleware](middleware.md)
* [Configuration](configuration.md)
* [Logging](logging.md)
* [Options Pattern](options-pattern.md)
* [CancellationToken](cancellation-token.md)
* [Async / Await](async-await.md)

## Ordre conseillé

Pour comprendre correctement le fonctionnement d'une application .NET moderne :

1. [Dependency Injection](dependency-injection.md)
2. [Middleware](middleware.md)
3. [Configuration](configuration.md)
4. [Logging](logging.md)
5. [Options Pattern](options-pattern.md)
6. [Async / Await](async-await.md)
7. [CancellationToken](cancellation-token.md)

## Structure d'une fiche

Chaque fiche doit essayer de répondre à ces questions :

1. Qu'est-ce que c'est ?
2. Pourquoi .NET utilise ce mécanisme ?
3. Quel problème cela résout ?
4. Comment cela fonctionne derrière la syntaxe ?
5. Comment l'utiliser correctement ?
6. Quelles sont les erreurs fréquentes ?
7. Quelle différence avec les mécanismes proches ?
8. Quelle règle mentale permet de s'en souvenir ?
9. Que peut-on demander en entretien ?

## Mental model

Une application .NET moderne peut être vue comme plusieurs mécanismes qui travaillent ensemble :

```text
Application
    |
    +-- DI
    |    -> crée et fournit les dépendances
    |
    +-- Middleware
    |    -> traite les requêtes HTTP
    |
    +-- Configuration
    |    -> fournit les paramètres de l'application
    |
    +-- Logging
    |    -> observe ce qui se passe
    |
    +-- Options
    |    -> transforme la configuration en objets typés
    |
    +-- Async / Await
    |    -> permet d'effectuer des opérations asynchrones
    |
    +-- CancellationToken
         -> permet d'arrêter proprement un travail
```

L'idée importante est que ces mécanismes ne sont pas isolés : ils constituent ensemble l'infrastructure d'une application .NET.
