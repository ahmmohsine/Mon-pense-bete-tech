# Angular Authentication

L'authentification côté Angular consiste principalement à gérer l'état d'authentification de l'utilisateur et à communiquer correctement avec l'API backend.

## Architecture

Dans une application Full Stack .NET :

```text
Angular
    |
    | Login
    v
ASP.NET Core API
    |
    v
Authentication
    |
    v
JWT / Cookie / autre mécanisme
```

Angular ne doit pas être considéré comme l'autorité de sécurité principale.

La décision d'autorisation doit rester côté serveur.

## Login

Le flux peut être représenté ainsi :

```text
User
  ↓
Angular Login Form
  ↓
POST /api/auth/login
  ↓
ASP.NET Core
  ↓
Authentication
  ↓
Token / Session
  ↓
Angular
```

## JWT

Dans une architecture utilisant JWT, l'API peut retourner un token après authentification.

Angular peut ensuite utiliser ce token pour les requêtes protégées.

```text
Login
  ↓
JWT
  ↓
HTTP Request
  ↓
Authorization: Bearer <token>
  ↓
ASP.NET Core
```

## Interceptor

Un interceptor peut être utilisé pour ajouter automatiquement le token aux requêtes :

```text
Component
    ↓
HttpClient
    ↓
Interceptor
    ↓
Authorization Header
    ↓
API
```

Exemple conceptuel :

```typescript
Authorization: Bearer <token>
```

## Authentication vs Authorization

À retenir :

```text
Authentication
    → Qui es-tu ?

Authorization
    → Que peux-tu faire ?
```

Angular peut masquer certaines parties de l'interface selon l'état de l'utilisateur, mais cela ne remplace jamais la vérification côté API.

Exemple :

```text
Angular
    → masque le bouton Delete

API
    → vérifie réellement si l'utilisateur peut supprimer
```

## Route Guards

Un guard peut empêcher une navigation vers une page réservée aux utilisateurs authentifiés.

```text
Navigation
    ↓
Auth Guard
    ↓
Utilisateur authentifié ?
   /  Oui  Non
  |    |
  v    v
Page  Login
```

Le guard améliore l'expérience utilisateur, mais ne constitue pas une protection suffisante de l'API.

## À retenir

> Angular gère l'expérience d'authentification côté frontend, mais le backend reste l'autorité de sécurité.
