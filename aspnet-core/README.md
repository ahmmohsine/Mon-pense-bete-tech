# ASP.NET Core

Cette section regroupe les mécanismes fondamentaux d'ASP.NET Core pour construire des applications Web et des API REST avec .NET.

L'objectif n'est pas seulement de mémoriser les attributs ou les méthodes, mais de comprendre ce qui se passe entre une requête HTTP et la réponse retournée par l'application.

## Fiches

* [Web API](web-api.md)
* [Routing](routing.md)
* [Model Binding](model-binding.md)
* [Validation](validation.md)
* [Middleware](middleware.md)
* [Filters](filters.md)
* [Authentication](authentication.md)
* [Authorization](authorization.md)

## Ordre conseillé

Pour comprendre correctement le fonctionnement d'une API ASP.NET Core :

1. [Web API](web-api.md)
2. [Routing](routing.md)
3. [Model Binding](model-binding.md)
4. [Validation](validation.md)
5. [Middleware](middleware.md)
6. [Filters](filters.md)
7. [Authentication](authentication.md)
8. [Authorization](authorization.md)

## Structure d'une fiche

Chaque fiche doit essayer de répondre à ces questions :

1. Qu'est-ce que c'est ?
2. Pourquoi ASP.NET Core utilise ce mécanisme ?
3. Quel problème cela résout ?
4. Que se passe-t-il derrière la syntaxe ?
5. Comment l'utiliser correctement ?
6. Quelles sont les erreurs fréquentes ?
7. Quelle différence avec les mécanismes proches ?
8. Quelle règle mentale permet de s'en souvenir ?
9. Que peut-on demander en entretien ?

## Mental model

Une requête HTTP ASP.NET Core peut être visualisée comme ceci :

```text
Client
  |
  | HTTP Request
  v
ASP.NET Core Pipeline
  |
  v
Middleware
  |
  v
Routing
  |
  v
Authentication
  |
  v
Authorization
  |
  v
Model Binding
  |
  v
Validation
  |
  v
Controller / Endpoint
  |
  v
Service
  |
  v
Repository / EF Core
  |
  v
HTTP Response
```

Les mécanismes ont des responsabilités différentes :

```text
Middleware
    -> pipeline HTTP global

Routing
    -> déterminer quel endpoint doit traiter la requête

Authentication
    -> déterminer qui est l'utilisateur

Authorization
    -> déterminer ce que l'utilisateur a le droit de faire

Model Binding
    -> transformer les données HTTP en objets ou paramètres C#

Validation
    -> vérifier que les données reçues respectent les règles

Controller / Endpoint
    -> exposer l'API HTTP

Service
    -> exécuter la logique applicative ou métier

Repository / EF Core
    -> accéder aux données lorsque cette couche est utilisée
```

## Pipeline et responsabilités

Il est important de distinguer les mécanismes qui appartiennent au **pipeline HTTP** des mécanismes qui interviennent lors de l'exécution d'un endpoint.

Une représentation simplifiée est :

```text
HTTP Request
     |
     v
Middleware Pipeline
     |
     v
Routing
     |
     +---- Authentication
     |
     +---- Authorization
     |
     v
Endpoint
     |
     +---- Model Binding
     |
     +---- Validation
     |
     v
Controller / Action
     |
     v
Service
     |
     v
Data Access / EF Core
     |
     v
HTTP Response
```

Cette représentation est volontairement simplifiée : l'ordre et le comportement exact dépendent de la configuration de l'application et des mécanismes utilisés.

## À retenir

ASP.NET Core fournit l'infrastructure HTTP et le pipeline permettant de recevoir une requête, de sélectionner un endpoint et de produire une réponse.

Les services applicatifs contiennent la logique applicative ou métier.

Une bonne séparation des responsabilités permet de garder une API :

* compréhensible ;
* testable ;
* maintenable ;
* sécurisée ;
* facilement évolutive.

La règle mentale principale :

> **ASP.NET Core gère le transport HTTP et le pipeline ; l'application gère la logique métier.**
