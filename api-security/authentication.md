# API Security — Authentication

## 1. Authentication : « Qui es-tu ? »

L'authentification répond à une question simple :

> **Qui est en train de faire cette requête ?**

L'autorisation répond à une autre question :

> **Cette personne a-t-elle le droit de faire cette action ?**

Exemple :

```text
Authentication
    ↓
"Je suis Ahlame"
    ↓
ASP.NET Core vérifie mon identité
    ↓
ClaimsPrincipal créé
    ↓
Authorization
    ↓
"Est-ce qu'Ahlame peut supprimer cet hôtel ?"
```

Il est important de ne pas confondre les deux.

| Concept | Question |
|---|---|
| Authentication | Qui es-tu ? |
| Authorization | Qu'as-tu le droit de faire ? |

ASP.NET Core utilise un service d'authentification et des handlers associés à des **authentication schemes** pour construire l'identité de l'utilisateur à partir de la requête.

---

# 2. Le principe général

Pour une API protégée, on peut imaginer le pipeline suivant :

```text
Client
   |
   | HTTP Request
   | Authorization: Bearer <token>
   ↓
ASP.NET Core
   |
   ↓
Authentication
   |
   | Le token est-il valide ?
   | Qui est l'utilisateur ?
   ↓
ClaimsPrincipal
   |
   ↓
Authorization
   |
   | Cet utilisateur a-t-il le droit ?
   ↓
Endpoint
```

L'authentification ne donne donc pas automatiquement accès à toutes les ressources.

Elle permet d'abord à ASP.NET Core de construire l'identité de l'utilisateur.

---

# 3. HttpContext.User

Une fois l'utilisateur authentifié, ASP.NET Core place son identité dans :

```csharp
HttpContext.User
```

Le type est :

```csharp
ClaimsPrincipal
```

Exemple :

```csharp
app.MapGet("/profile", (HttpContext context) =>
{
    var user = context.User;

    return Results.Ok(new
    {
        IsAuthenticated = user.Identity?.IsAuthenticated,
        Name = user.Identity?.Name
    });
});
```

Dans un contrôleur :

```csharp
[Authorize]
[HttpGet("profile")]
public IActionResult GetProfile()
{
    var user = HttpContext.User;

    return Ok(user.Identity?.Name);
}
```

Mentalement :

```text
HttpContext
    └── User
         └── ClaimsPrincipal
              └── ClaimsIdentity
                   └── Claims
```

---

# 4. ClaimsPrincipal, ClaimsIdentity et Claims

Ces trois notions sont très importantes.

## ClaimsPrincipal

Le `ClaimsPrincipal` représente l'utilisateur authentifié.

```csharp
HttpContext.User
```

C'est l'objet que l'autorisation va utiliser pour prendre ses décisions.

## ClaimsIdentity

Une `ClaimsIdentity` représente une identité associée à cet utilisateur.

Un `ClaimsPrincipal` peut théoriquement contenir plusieurs identités.

## Claim

Une claim est une information concernant l'utilisateur.

Exemple :

```text
Name       = Ahlame
Email      = ahlame@example.com
Role       = Admin
UserId     = 123
Department = IT
```

En C# :

```csharp
var email = User.FindFirst("email")?.Value;
```

Ou :

```csharp
var userId = User.FindFirst("sub")?.Value;
```

Et pour un rôle :

```csharp
var isAdmin = User.IsInRole("Admin");
```

Mental model :

```text
ClaimsPrincipal
      |
      +-- Identity
      |
      +-- Claim : Name = Ahlame
      +-- Claim : Email = ...
      +-- Claim : Role = Admin
      +-- Claim : sub = 123
```

---

# 5. Authentication Scheme

Un **authentication scheme** indique à ASP.NET Core comment authentifier la requête.

Exemples :

```text
Cookies
JWT Bearer
Identity
External providers
```

On peut configurer un schéma par défaut :

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        // Configuration
    });
```

Ici :

```csharp
JwtBearerDefaults.AuthenticationScheme
```

indique que le mécanisme d'authentification par défaut est :

```text
Bearer
```

Un scheme possède notamment un handler chargé d'effectuer le travail d'authentification.

---

# 6. Cookie Authentication

Avec les cookies, le serveur authentifie l'utilisateur et le navigateur conserve un cookie.

Schéma :

```text
Login
  ↓
Serveur authentifie l'utilisateur
  ↓
Cookie créé
  ↓
Navigateur conserve le cookie
  ↓
Requête suivante
  ↓
Cookie envoyé automatiquement
  ↓
ASP.NET Core authentifie l'utilisateur
```

Exemple conceptuel :

```http
Cookie: .AspNetCore.Cookies=...
```

Les cookies sont particulièrement adaptés aux applications web où le navigateur gère directement la session.

Pour les API consommées par différents clients, on rencontre très souvent le modèle Bearer Token.

---

# 7. JWT Bearer Authentication

Une API peut utiliser un access token envoyé dans :

```http
Authorization: Bearer <token>
```

Exemple :

```http
GET /api/hotels
Authorization: Bearer eyJhbGciOi...
```

Configuration classique :

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://identity-server.example.com";
        options.Audience = "hotel-api";
    });
```

Puis dans le pipeline :

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

Et sur un endpoint :

```csharp
app.MapGet("/api/hotels", () =>
{
    return Results.Ok();
})
.RequireAuthorization();
```

---

# 8. Attention : JWT n'est pas forcément chiffré

Un JWT classique possède trois parties :

```text
HEADER.PAYLOAD.SIGNATURE
```

Exemple conceptuel :

```text
xxxxx.yyyyy.zzzzz
```

Les parties sont encodées en Base64Url.

Le payload peut contenir :

```json
{
  "sub": "123",
  "email": "ahlame@example.com",
  "role": "Admin"
}
```

Un point fondamental :

> **Encoder n'est pas chiffrer.**

Le contenu d'un JWT signé peut généralement être lu par celui qui possède le token.

La signature sert principalement à vérifier :

```text
Qui a créé le token ?
Le token a-t-il été modifié ?
```

Il ne faut donc pas mettre de secret dans les claims d'un JWT simplement parce qu'elles sont dans le token.

---

# 9. Comment ASP.NET Core valide un JWT ?

Lorsqu'une requête arrive :

```http
Authorization: Bearer <token>
```

le handler JWT doit notamment vérifier que le token respecte les règles configurées.

Les contrôles importants comprennent :

```text
1. Signature
2. Issuer
3. Audience
4. Expiration
5. Validité temporelle
```

## Signature

Permet de vérifier l'intégrité et l'origine attendue du token.

Si quelqu'un modifie :

```json
"role": "User"
```

en :

```json
"role": "Admin"
```

la signature ne correspond plus.

Le token doit être rejeté.

## Issuer

L'`iss` indique qui a émis le token.

Exemple :

```text
https://identity.example.com
```

L'API doit vérifier que l'émetteur correspond à celui attendu.

## Audience

L'`aud` indique pour quelle ressource ou API le token est destiné.

Exemple :

```text
hotel-api
```

Un token destiné à une autre API ne doit pas être accepté simplement parce que sa signature est valide.

## Expiration

Le claim :

```text
exp
```

indique quand le token expire.

Un token expiré doit être refusé.

---

# 10. Access Token et Refresh Token

Un access token est généralement de courte durée.

Exemple conceptuel :

```text
Access Token
    durée courte
    ↓
API
```

Lorsque l'access token expire, l'application peut utiliser un refresh token pour obtenir un nouvel access token selon le mécanisme utilisé.

```text
Access Token
      ↓
   expire
      ↓
Refresh Token
      ↓
Nouveau Access Token
```

Pourquoi ne pas simplement faire durer l'access token très longtemps ?

Parce que si un access token est volé, sa durée de validité limite la fenêtre d'utilisation.

---

# 11. ASP.NET Core Identity

`ASP.NET Core Identity` fournit un système complet de gestion des utilisateurs.

Il peut gérer notamment :

```text
Users
Passwords
Roles
Claims
Email confirmation
Two-factor authentication
Password reset
Login
Logout
```

Exemple :

```csharp
public class ApplicationUser : IdentityUser
{
}
```

Le `UserManager<TUser>` permet de manipuler les utilisateurs.

Exemple :

```csharp
public class UserService(
    UserManager<ApplicationUser> userManager)
{
    public async Task<ApplicationUser?> FindByEmail(string email)
    {
        return await userManager.FindByEmailAsync(email);
    }
}
```

`UserManager` ne signifie pas :

> « Je suis le mécanisme d'authentification HTTP. »

Il sert principalement à gérer les utilisateurs Identity.

C'est une distinction importante.

---

# 12. Les mots de passe

Une application ne doit pas stocker les mots de passe en clair.

Mauvais :

```text
Email              Password
----------------------------
alice@example.com  Azerty123
```

Identity utilise un mécanisme de hashage des mots de passe.

Mentalement :

```text
Password
   ↓
Password Hasher
   ↓
Hash stocké en base
```

Lors de la connexion :

```text
Password fourni
      ↓
Hashage / vérification
      ↓
Comparaison avec le hash enregistré
      ↓
Utilisateur authentifié
```

Le serveur n'a donc pas besoin de connaître le mot de passe original stocké en clair.

---

# 13. MapIdentityApi

Si tu utilises récemment :

```csharp
app.MapIdentityApi<ApplicationUser>();
```

il faut comprendre exactement ce que cette méthode fait.

Elle **mappe les endpoints HTTP fournis par ASP.NET Core Identity**.

Avec la configuration Identity API correspondante, on retrouve notamment des endpoints comme :

```text
POST /register
POST /login
POST /refresh
GET  /confirmEmail
POST /resendConfirmationEmail
POST /forgotPassword
POST /resetPassword
POST /manage/2fa
GET  /manage/info
POST /manage/info
```

Exemple de configuration :

```csharp
builder.Services
    .AddIdentityApiEndpoints<ApplicationUser>()
    .AddEntityFrameworkStores<ApplicationDbContext>();

builder.Services.AddAuthorization();

var app = builder.Build();

app.UseAuthentication();
app.UseAuthorization();

app.MapIdentityApi<ApplicationUser>();
```

Le point important :

```csharp
MapIdentityApi()
```

ne signifie pas :

> « J'ai magiquement sécurisé toute mon API. »

Cette méthode ajoute les endpoints Identity.

Pour protéger tes propres endpoints, tu dois utiliser l'autorisation :

```csharp
app.MapGet("/api/hotels", ...)
    .RequireAuthorization();
```

ou :

```csharp
[Authorize]
```

selon le style utilisé.

---

# 14. MapIdentityApi et JWT : attention

Un piège fréquent est de penser :

```text
MapIdentityApi = JWT standard
```

Ce n'est pas aussi simple.

`AddIdentityApiEndpoints<TUser>()` configure les services Identity API pour prendre en charge les cookies et les tokens Identity.

Les tokens utilisés par les Identity API intégrées ne sont pas simplement à considérer comme des JWT classiques que tu aurais générés toi-même avec `JwtSecurityTokenHandler`.

Donc :

```text
ASP.NET Core Identity API
        ≠
"Mon propre serveur JWT"
```

Si tu construis une architecture avec un véritable Identity Provider / Authorization Server, OAuth 2.0 et OpenID Connect peuvent être plus adaptés selon les besoins.

Pour une application simple, les Identity API intégrées peuvent cependant fournir beaucoup de fonctionnalités prêtes à l'emploi.

---

# 15. UseAuthentication()

Cette ligne :

```csharp
app.UseAuthentication();
```

ajoute le middleware d'authentification au pipeline HTTP.

Son rôle est notamment de permettre aux authentication handlers d'examiner la requête et de construire l'identité de l'utilisateur.

Conceptuellement :

```text
Request
  ↓
UseAuthentication()
  ↓
"Qui est cet utilisateur ?"
  ↓
HttpContext.User
  ↓
UseAuthorization()
```

Il faut placer l'authentification avant les composants qui ont besoin de savoir qui est l'utilisateur.

Configuration typique :

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

---

# 16. UseAuthorization()

L'autorisation intervient après l'authentification.

Elle répond à :

```text
"Maintenant que je sais qui tu es,
as-tu le droit d'accéder à cette ressource ?"
```

Exemple :

```csharp
app.MapDelete("/api/hotels/{id}", DeleteHotel)
    .RequireAuthorization();
```

Ou :

```csharp
[Authorize]
public IActionResult DeleteHotel(int id)
{
    ...
}
```

Le fonctionnement mental est donc :

```text
Authentication
    ↓
Qui es-tu ?
    ↓
ClaimsPrincipal
    ↓
Authorization
    ↓
Que peux-tu faire ?
```

---

# 17. 401 Unauthorized vs 403 Forbidden

C'est une question très fréquente en entretien.

## 401

Le serveur ne considère pas la requête comme authentifiée.

Exemples :

```text
Token absent
Token invalide
Token expiré
Signature incorrecte
Issuer incorrect
Audience incorrecte
```

Mentalement :

> **Je ne peux pas confirmer qui tu es.**

```http
401 Unauthorized
```

## 403

L'utilisateur est authentifié, mais il n'a pas le droit d'effectuer l'action.

Exemple :

```text
Utilisateur = User
Endpoint = réservé aux Admin
```

Le serveur sait qui est l'utilisateur.

Mais :

```text
User ≠ Admin
```

Donc :

```http
403 Forbidden
```

À retenir :

```text
401 = pas authentifié
403 = authentifié mais interdit
```

---

# 18. Roles

Une claim peut représenter un rôle :

```text
role = Admin
```

En ASP.NET Core :

```csharp
[Authorize(Roles = "Admin")]
public IActionResult DeleteHotel(int id)
{
    ...
}
```

Ou avec Minimal APIs :

```csharp
app.MapDelete("/api/hotels/{id}", DeleteHotel)
    .RequireAuthorization(policy =>
        policy.RequireRole("Admin"));
```

Mentalement :

```text
Authentication
      ↓
Claims
      ↓
Role = Admin
      ↓
Authorization
      ↓
Autorisé ?
```

Mais les rôles ne sont qu'une manière de faire de l'autorisation.

Pour des règles plus complexes, les policies sont généralement plus adaptées.

---

# 19. Claims vs Roles vs Policies

## Claims

Décrivent l'utilisateur :

```text
department = IT
country = BE
subscription = Premium
```

## Roles

Représentent souvent une catégorie de permissions :

```text
Admin
Manager
User
```

## Policies

Décrivent une règle d'autorisation :

```text
Utilisateur authentifié
ET
Role = Manager
ET
Department = IT
```

Mental model :

```text
Claims
   ↓
Informations

Roles
   ↓
Catégories de permissions

Policies
   ↓
Règles de décision
```

---

# 20. Stockage des tokens côté navigateur

Il n'existe pas une règle universelle disant :

> « Toujours mettre le token dans X. »

Le choix dépend du type d'application et du modèle d'authentification.

Pour les cookies, les attributs importants comprennent notamment :

```text
HttpOnly
Secure
SameSite
```

`HttpOnly` empêche le JavaScript côté navigateur de lire directement le cookie.

`Secure` indique que le cookie doit être envoyé via HTTPS.

`SameSite` contrôle notamment les conditions dans lesquelles le navigateur envoie le cookie dans des contextes cross-site.

Le stockage d'un token dans un mécanisme accessible au JavaScript doit être étudié avec attention, notamment face au risque XSS.

La bonne question n'est donc pas :

> « Où mettre mon token parce que tout le monde fait pareil ? »

Mais :

> « Quel modèle d'authentification correspond à mon architecture et quelles menaces dois-je maîtriser ? »

---

# 21. HTTPS

L'authentification ne doit pas être pensée indépendamment du transport.

Envoyer :

```http
Authorization: Bearer <token>
```

sur une connexion non sécurisée peut exposer le token.

En production :

```text
HTTPS
  ↓
TLS
  ↓
Confidentialité + intégrité du transport
```

Il faut donc protéger les communications entre :

```text
Frontend
   ↕
API
   ↕
Identity Provider
```

---

# 22. Pourquoi un JWT volé est dangereux

Imagine :

```text
Access Token
   ↓
volé
```

L'attaquant peut potentiellement l'utiliser comme le client légitime jusqu'à son expiration ou jusqu'à ce qu'il soit invalidé par le mécanisme concerné.

C'est pourquoi on cherche notamment à :

```text
Limiter la durée de vie
Protéger le stockage
Utiliser HTTPS
Limiter les permissions
Valider correctement le token
Protéger les refresh tokens
```

Un JWT n'est pas « sécurisé parce qu'il est JWT ».

La sécurité vient de tout le système autour :

```text
Issuance
+
Signature
+
Validation
+
Transport
+
Storage
+
Expiration
+
Authorization
```

---

# 23. Authentication complète : exemple mental

Supposons :

```text
POST /login
```

avec :

```json
{
    "email": "ahlame@example.com",
    "password": "********"
}
```

Le serveur :

```text
1. Recherche l'utilisateur
2. Vérifie le mot de passe
3. Authentifie l'utilisateur
4. Produit un mécanisme de session/token
5. Le client le conserve selon le mécanisme choisi
```

Puis :

```text
GET /api/hotels
Authorization: Bearer <access-token>
```

L'API :

```text
1. Récupère le token
2. Vérifie sa validité
3. Construit ClaimsPrincipal
4. Place l'identité dans HttpContext.User
5. Authorization vérifie les règles
6. Endpoint exécuté
```

---

# 24. Schéma global à mémoriser

```text
                  LOGIN
                    |
                    v
             Identity / IdP
                    |
                    v
          Cookie ou Access Token
                    |
                    v
               CLIENT
                    |
                    | HTTP Request
                    v
          ASP.NET Core API
                    |
                    v
            Authentication
                    |
                    v
           ClaimsPrincipal
                    |
                    v
            Authorization
                    |
             +------+------+
             |             |
          Refusé         Autorisé
             |             |
          401/403          v
                       Endpoint
```

---

# 25. Erreurs fréquentes

## Erreur 1 : confondre Authentication et Authorization

```text
Authentication = identité
Authorization = permissions
```

## Erreur 2 : penser que JWT = chiffrement

Faux.

Un JWT signé n'est pas automatiquement chiffré.

## Erreur 3 : ne pas valider l'audience

Un token valide cryptographiquement peut être destiné à une autre API.

## Erreur 4 : accepter un token expiré

Toujours vérifier sa validité temporelle.

## Erreur 5 : croire que MapIdentityApi protège toutes les routes

```csharp
app.MapIdentityApi<ApplicationUser>();
```

ajoute les endpoints Identity.

Tes propres endpoints doivent encore être protégés par l'autorisation.

## Erreur 6 : mettre des secrets dans les claims

Les claims ne sont pas un coffre-fort.

## Erreur 7 : utiliser un access token avec une durée excessive

Plus il reste valide longtemps après un vol, plus la fenêtre d'exploitation est grande.

## Erreur 8 : oublier UseAuthentication()

Configurer les services :

```csharp
AddAuthentication(...)
```

et utiliser le middleware :

```csharp
UseAuthentication()
```

sont deux choses différentes.

---

# 26. Checklist Authentication API

Avant de considérer l'authentification correctement configurée :

```text
[ ] HTTPS activé
[ ] Authentication scheme clairement défini
[ ] Token/cookie correctement validé
[ ] Signature validée
[ ] Issuer validé
[ ] Audience validée
[ ] Expiration validée
[ ] UseAuthentication() présent lorsque nécessaire
[ ] UseAuthorization() configuré
[ ] Endpoints protégés explicitement
[ ] 401 et 403 correctement compris
[ ] Passwords jamais stockés en clair
[ ] Secrets hors du code source
[ ] Tokens protégés côté client
[ ] Access tokens de durée raisonnable
[ ] Refresh tokens protégés
[ ] Claims limitées aux informations nécessaires
```

---

# 27. Questions d'entretien

### Quelle différence entre Authentication et Authorization ?

Authentication vérifie l'identité.

Authorization vérifie les permissions.

### À quoi sert HttpContext.User ?

Il représente le principal associé à la requête, notamment l'identité et les claims de l'utilisateur authentifié.

### Qu'est-ce qu'une claim ?

Une information associée à l'identité de l'utilisateur, par exemple son identifiant, son email ou son rôle.

### Qu'est-ce qu'un authentication scheme ?

C'est le mécanisme configuré par ASP.NET Core pour effectuer l'authentification, par exemple Cookies ou Bearer.

### Pourquoi UseAuthentication avant UseAuthorization ?

Parce que l'autorisation doit pouvoir s'appuyer sur l'identité construite par l'authentification.

### Différence entre 401 et 403 ?

```text
401 = authentification absente ou invalide
403 = authentification réussie mais accès refusé
```

### Un JWT est-il chiffré ?

Pas nécessairement. Un JWT signé peut être décodé. La signature protège surtout l'intégrité et permet de vérifier l'émetteur selon la configuration.

### À quoi sert MapIdentityApi ?

À mapper les endpoints HTTP fournis par ASP.NET Core Identity API, comme register, login et refresh.

### MapIdentityApi protège-t-il automatiquement mes endpoints ?

Non. Les endpoints de ton application doivent être protégés avec les mécanismes d'autorisation appropriés.

---

# 28. À retenir

Le fonctionnement essentiel est :

```text
LOGIN
  ↓
Authentication
  ↓
Identity
  ↓
ClaimsPrincipal
  ↓
Authorization
  ↓
Permission
  ↓
Endpoint
```

Et pour une API Bearer :

```text
Authorization: Bearer <token>
             ↓
       Validate token
             ↓
       Build identity
             ↓
      HttpContext.User
             ↓
        [Authorize]
             ↓
          Access
```

## Phrase à mémoriser

> **Authentication détermine qui je suis ; Authorization détermine ce que j'ai le droit de faire.**
