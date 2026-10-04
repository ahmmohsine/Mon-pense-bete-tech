# ASP.NET Core

Cette section regroupe les mécanismes fondamentaux d'ASP.NET Core pour construire des applications Web et des API REST avec .NET.

L'objectif n'est pas seulement de mémoriser les attributs ou les méthodes, mais de comprendre ce qui se passe entre :

```text
HTTP Request
     |
     v
ASP.NET Core Pipeline
     |
     v
Routing
     |
     v
Model Binding / Validation
     |
     v
Controller / Endpoint
     |
     v
Service
     |
     v
Response
```

## Fiches

- Web API
- Routing
- Model Binding
- Validation
- Filters
- Middleware
- Authentication
- Authorization

## Ordre conseillé

Pour comprendre correctement le fonctionnement d'une API ASP.NET Core :

1. Web API
2. Routing
3. Model Binding
4. Validation
5. Middleware
6. Filters
7. Authentication
8. Authorization

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

Model Binding
    -> transformer les données HTTP en objets/paramètres C#

Validation
    -> vérifier que les données reçues respectent les règles

Controller / Endpoint
    -> exposer l'API HTTP

Authentication
    -> déterminer qui est l'utilisateur

Authorization
    -> déterminer ce que l'utilisateur a le droit de faire
```

## À retenir

ASP.NET Core fournit le pipeline et les mécanismes HTTP.

Les services applicatifs contiennent la logique métier.

Une bonne séparation permet de garder une API :

- compréhensible ;
- testable ;
- maintenable ;
- sécurisée ;
- facilement évolutive.
