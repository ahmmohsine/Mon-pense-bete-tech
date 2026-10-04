# Angular

Cette section regroupe les mécanismes fondamentaux d'Angular utiles pour le développement frontend et Full Stack avec .NET.

L'objectif n'est pas seulement de mémoriser la syntaxe Angular, mais de comprendre comment les différents mécanismes travaillent ensemble :

```text
Application Angular

      |
      +-- Components
      |      -> interface et comportement UI
      |
      +-- Services
      |      -> logique réutilisable et communication
      |
      +-- Dependency Injection
      |      -> fournit les dépendances
      |
      +-- Signals
      |      -> état réactif
      |
      +-- Inputs / Outputs
      |      -> communication entre components
      |
      +-- Routing
      |      -> navigation entre les vues
      |
      +-- HttpClient
      |      -> communication avec l'API
      |
      +-- RxJS
             -> gestion des flux asynchrones
```

Dans un contexte Full Stack .NET :

```text
Angular
    |
    | HTTP / JSON
    v
ASP.NET Core Web API
    |
    v
Services
    |
    v
EF Core
    |
    v
Database
```

---

## Fiches

- [Components](components.md)
- [Inputs & Outputs](inputs-outputs.md)
- [Services](services.md)
- [Dependency Injection](dependency-injection.md)
- [Signals](signals.md)
- [Routing](routing.md)
- [HttpClient](httpclient.md)
- [RxJS](rxjs.md)
- [Forms](forms.md)
- [Authentication](authentication.md)

---

## Ordre conseillé

1. [Components](components.md)
2. [Inputs & Outputs](inputs-outputs.md)
3. [Services](services.md)
4. [Dependency Injection](dependency-injection.md)
5. [Signals](signals.md)
6. [Routing](routing.md)
7. [HttpClient](httpclient.md)
8. [RxJS](rxjs.md)
9. [Forms](forms.md)
10. [Authentication](authentication.md)

---

## Structure d'une fiche

Chaque fiche doit essayer de répondre à ces questions :

1. Qu'est-ce que c'est ?
2. Pourquoi Angular utilise ce mécanisme ?
3. Quel problème cela résout ?
4. Comment cela fonctionne derrière la syntaxe ?
5. Comment l'utiliser correctement ?
6. Quelles sont les erreurs fréquentes ?
7. Quelle différence avec les mécanismes proches ?
8. Quelle règle mentale permet de s'en souvenir ?
9. Que peut-on demander en entretien ?

---

## Mental model

```text
Component
    -> affiche et réagit à l'interface

Service
    -> centralise la logique réutilisable

Dependency Injection
    -> fournit les dépendances

Signal
    -> représente un état réactif

Input
    -> parent -> enfant

Output
    -> enfant -> parent

Router
    -> URL -> component

HttpClient
    -> Angular -> API

RxJS
    -> flux asynchrones
```

---

## Components

Le component constitue l'un des éléments centraux d'Angular.

Il regroupe généralement :

```text
Component
    |
    +-- TypeScript
    +-- Template HTML
    +-- Styles
```

Le component contient le comportement nécessaire à l'interface tandis que le template décrit ce qui doit être affiché.

Voir : [Components](components.md)

---

## Communication entre components

```text
Parent
   |
   | Input
   v
Child
   |
   | Output
   v
Parent
```

Règle mentale :

```text
Input  = parent -> enfant
Output = enfant -> parent
```

Voir : [Inputs & Outputs](inputs-outputs.md)

---

## Services

Les services permettent notamment de sortir certaines responsabilités des components.

```text
Component
    |
    v
Service
    |
    v
HTTP
    |
    v
API
```

Voir : [Services](services.md)

---

## Dependency Injection

Angular possède son propre système de Dependency Injection.

Le principe est similaire dans l'idée à celui utilisé dans ASP.NET Core :

```text
Component
    |
    | "J'ai besoin de HotelService"
    v
Angular DI
    |
    v
HotelService
```

En .NET :

```csharp
builder.Services.AddScoped<IHotelService, HotelService>();
```

En Angular, le mécanisme et la syntaxe sont différents, mais l'idée générale reste celle de l'injection de dépendances.

Voir : [Dependency Injection](dependency-injection.md)

---

## Signals

Les Signals permettent de représenter un état réactif dans Angular moderne.

```text
Signal
    |
    v
État réactif
    |
    v
Angular détecte les changements
    |
    v
Interface mise à jour
```

Distinction importante :

```text
Signal
    = donnée source

Computed
    = donnée dérivée
```

Voir : [Signals](signals.md)

---

## Routing

Le Router permet de naviguer entre différentes vues d'une application Angular sans recharger toute l'application.

```text
URL
 |
 v
Angular Router
 |
 v
Route
 |
 v
Component
```

Exemples :

```text
/hotels
/hotels/12
/login
/users
```

Voir : [Routing](routing.md)

---

## HttpClient

Angular communique généralement avec une API grâce à `HttpClient`.

```text
Angular
    |
    | HTTP GET / POST / PUT / DELETE
    v
ASP.NET Core API
    |
    v
Service
    |
    v
EF Core
    |
    v
Database
```

Voir : [HttpClient](httpclient.md)

---

## RxJS

Angular utilise largement RxJS pour gérer les flux asynchrones.

```text
Observable
    |
    v
Flux de valeurs
    |
    v
Operators
    |
    v
Résultat
```

Exemples :

```text
map
filter
catchError
```

Il faut notamment savoir distinguer Promise et Observable.

Voir : [RxJS](rxjs.md)

---

## Forms

Angular propose plusieurs approches pour gérer les formulaires.

Les deux principales sont les Template-driven Forms et les Reactive Forms.

Voir : [Forms](forms.md)

---

## Authentication

Dans une application Full Stack Angular + ASP.NET Core, l'authentification permet notamment d'identifier l'utilisateur.

Il faut distinguer :

```text
Authentication
    = Qui es-tu ?

Authorization
    = As-tu le droit ?
```

Voir : [Authentication](authentication.md)

---

## Angular et ASP.NET Core

```text
+---------------------------+
|          Angular          |
| Components                |
| Services                  |
| Signals                   |
| Router                    |
| HttpClient                |
+-------------+-------------+
              |
              | HTTP / JSON
              v
+---------------------------+
|     ASP.NET Core API      |
| Controllers               |
| Services                  |
| DTOs                      |
| Authentication            |
| Authorization             |
+-------------+-------------+
              |
              v
+---------------------------+
|          EF Core          |
| DbContext                 |
| Entities                  |
| LINQ                      |
+-------------+-------------+
              |
              v
+---------------------------+
|          Database         |
+---------------------------+
```

La séparation principale est :

```text
Angular
    = frontend

ASP.NET Core
    = backend / API

EF Core
    = accès aux données

Database
    = persistance
```

---

## Où placer la logique ?

Une erreur fréquente consiste à mettre trop de responsabilités dans les components.

À éviter :

```text
Component
    |
    +-- affichage
    +-- appels HTTP
    +-- logique complexe
    +-- transformation des données
    +-- gestion importante de l'état
```

On préfère généralement :

```text
Component
    |
    v
Service
    |
    v
API
```

Voir également :

- [Components](components.md)
- [Services](services.md)
- [HttpClient](httpclient.md)
- [Signals](signals.md)

---

## Correspondances Angular / .NET

| Angular | .NET / ASP.NET Core |
|---|---|
| Component | Classe orientée UI |
| Service | Service applicatif |
| Dependency Injection | Dependency Injection |
| HttpClient | HttpClient |
| Observable | Abstraction de flux asynchrone |
| Router | Routing |
| Guard | Contrôle d'accès / navigation |
| Interceptor | Conceptuellement proche d'un pipeline |
| Signal | État réactif |
| Template | Interface / vue |
| TypeScript | Langage frontend |
| ASP.NET Core API | Backend |
| EF Core | Accès aux données |

Attention : ces correspondances servent à construire une intuition. Les mécanismes internes ne sont pas identiques.

---

## Questions d'entretien

### Quelle est la différence entre un component et un service ?

Un component gère principalement une partie de l'interface utilisateur et son comportement.

Un service permet de centraliser une logique réutilisable, par exemple des appels HTTP.

### À quoi sert un Input ?

À transmettre une donnée d'un component parent vers un component enfant.

### À quoi sert un Output ?

À permettre à un component enfant d'émettre un événement que le parent peut écouter.

### À quoi sert un Signal ?

À représenter un état réactif qu'Angular peut suivre afin de réagir aux changements de valeur.

### À quoi sert `computed()` ?

À représenter une valeur dérivée calculée à partir d'autres Signals.

### Quel est le rôle de HttpClient ?

À effectuer des communications HTTP avec des services externes, notamment une API ASP.NET Core.

### Pourquoi utiliser des services ?

Pour séparer la logique de l'interface et éviter de concentrer toute la responsabilité dans les components.

---

## À retenir

```text
Component
    -> interface + comportement UI

Service
    -> logique réutilisable / communication

DI
    -> fournit les dépendances

Input
    -> parent -> enfant

Output
    -> enfant -> parent

Signal
    -> état réactif

Computed
    -> état dérivé

Router
    -> URL -> component

HttpClient
    -> Angular -> API

RxJS
    -> gestion des flux asynchrones
```

## Phrase à mémoriser

> Angular construit l'interface avec des components, centralise les services avec la Dependency Injection, gère l'état avec des mécanismes réactifs comme les Signals, et communique avec mon API ASP.NET Core via HTTP.
