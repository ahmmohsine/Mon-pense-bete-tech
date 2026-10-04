# Authentication en ASP.NET Core

## 1. Définition

L'**Authentication** consiste à déterminer l'identité d'un utilisateur ou d'un client.

La question fondamentale est :

> **« Qui es-tu ? »**

Exemples :

```text
Username / Password
JWT
Cookie
OpenID Connect
OAuth
API Key
Certificate
```

Une fois l'utilisateur authentifié, ASP.NET Core peut construire son identité et la placer dans :

```csharp
HttpContext.User
```

Mentalement :

```text
Credentials
    |
    v
Authentication
    |
    v
Identity / Claims
    |
    v
HttpContext.User
```

---

# 2. Authentication vs Authorization

Il faut absolument distinguer les deux.

### Authentication

```text
Qui es-tu ?
```

Exemple :

```text
Utilisateur = Ahlame
```

### Authorization

```text
As-tu le droit ?
```

Exemple :

```text
Ahlame possède le rôle Admin
    |
    v
Accès à /api/admin
```

Mentalement :

```text
Authentication
      |
      v
Qui es-tu ?
      |
      v
Authorization
      |
      v
Que peux-tu faire ?
```

---

# 3. Exemple concret

Un utilisateur envoie :

```http
Authorization: Bearer eyJ...
```

Le serveur :

```text
1. Récupère le token
2. Vérifie sa validité
3. Vérifie sa signature
4. Lit les claims
5. Construit l'identité
6. Place l'utilisateur dans HttpContext.User
```

Ensuite l'autorisation peut décider :

```text
Utilisateur authentifié ?
Utilisateur possède-t-il le rôle requis ?
```

---

# 4. `HttpContext.User`

Après authentification, l'utilisateur courant est généralement accessible avec :

```csharp
HttpContext.User
```

Dans un Controller :

```csharp
var user = User;
```

On peut consulter :

```csharp
User.Identity?.IsAuthenticated
```

et les claims :

```csharp
User.Claims
```

---

# 5. `ClaimsPrincipal`

`HttpContext.User` est généralement un :

```csharp
ClaimsPrincipal
```

Un `ClaimsPrincipal` représente le principal de sécurité courant.

Il peut contenir une ou plusieurs identités :

```text
ClaimsPrincipal
    |
    +-- ClaimsIdentity
           |
           +-- Claim
           +-- Claim
           +-- Claim
```

Exemple conceptuel :

```text
User
 |
 +-- Name = Ahlame
 +-- Email = ahlame@example.com
 +-- Role = Admin
 +-- UserId = 42
```

---

# 6. Qu'est-ce qu'un Claim ?

Un **Claim** représente une information concernant l'identité.

Exemple :

```csharp
new Claim(ClaimTypes.NameIdentifier, "42")
```

ou :

```csharp
new Claim(ClaimTypes.Name, "Ahlame")
```

ou :

```csharp
new Claim(ClaimTypes.Role, "Admin")
```

Mentalement :

```text
User
 |
 +-- UserId = 42
 +-- Name = Ahlame
 +-- Role = Admin
```

---

# 7. Pourquoi les Claims sont importants ?

L'autorisation peut ensuite utiliser ces informations.

Exemple :

```csharp
[Authorize(Roles = "Admin")]
public IActionResult GetAdminData()
{
    return Ok();
}
```

Le mécanisme d'autorisation cherche notamment les informations nécessaires dans l'identité de l'utilisateur.

---

# 8. Authentication Schemes

ASP.NET Core utilise des **Authentication Schemes**.

Un scheme indique essentiellement :

> « Quel mécanisme dois-je utiliser pour authentifier cette requête ? »

Exemples :

```text
Bearer
Cookies
OpenIdConnect
```

Avec JWT :

```csharp
.AddJwtBearer(...)
```

Le scheme correspond alors au mécanisme Bearer.

---

# 9. Authentication Handler

Un **Authentication Handler** contient la logique permettant de traiter un mécanisme d'authentification.

Pour JWT Bearer, ASP.NET Core utilise un handler adapté à ce mécanisme.

Mentalement :

```text
Request
   |
   v
Authentication Scheme
   |
   v
Authentication Handler
   |
   v
Credentials
   |
   v
ClaimsPrincipal
```

---

# 10. JWT Bearer Authentication

Pour une API REST, le JWT Bearer est une approche très courante.

Le client envoie :

```http
Authorization: Bearer <token>
```

ASP.NET Core peut être configuré avec :

```csharp
builder.Services
    .AddAuthentication("Bearer")
    .AddJwtBearer(options =>
    {
        options.Authority = "...";
        options.Audience = "...";
    });
```

La configuration exacte dépend de ton fournisseur d'identité.

---

# 11. Que contient un JWT ?

Un JWT est généralement composé de trois parties :

```text
Header.Payload.Signature
```

Par exemple :

```text
xxxxx.yyyyy.zzzzz
```

### Header

Contient notamment des informations sur l'algorithme et le type de token.

### Payload

Contient des claims.

### Signature

Permet de vérifier que le token n'a pas été modifié et qu'il a été signé par une partie possédant la clé appropriée.

---

# 12. Attention : JWT signé ≠ données secrètes

Un JWT signé n'est généralement pas chiffré.

Le payload peut souvent être décodé.

Donc ne mets pas dans un JWT :

```text
mot de passe
secret
clé privée
données sensibles inutiles
```

La signature permet principalement de garantir l'intégrité et l'authenticité du token selon le mécanisme utilisé.

Mentalement :

```text
JWT signé
    |
    +--> intégrité / authenticité
    |
    X--> confidentialité automatique
```

---

# 13. Validation d'un JWT

Lorsqu'une API reçoit un JWT, elle doit notamment vérifier des éléments comme :

```text
Signature
Expiration
Issuer
Audience
Algorithme attendu
```

Selon la configuration, d'autres validations peuvent être nécessaires.

Mentalement :

```text
Token reçu
    |
    +-- Signature valide ?
    +-- Expiré ?
    +-- Bon issuer ?
    +-- Bonne audience ?
    |
    v
Authentication réussie
```

---

# 14. Expiration

Un JWT contient généralement une claim d'expiration :

```text
exp
```

Une fois expiré, le token ne doit plus être accepté comme valide.

Le client doit alors obtenir un nouveau token selon le mécanisme d'authentification utilisé.

---

# 15. Signature

La signature permet de vérifier que le token n'a pas été modifié.

Mentalement :

```text
Token original
     |
     v
Signature correcte
     |
     v
Token accepté

Token modifié
     |
     v
Signature incorrecte
     |
     v
Token rejeté
```

Le type de clé utilisé dépend de l'algorithme :

```text
Symmetric
    -> même secret pour signer/vérifier

Asymmetric
    -> clé privée pour signer
    -> clé publique pour vérifier
```

---

# 16. Symmetric vs Asymmetric

### Symétrique

Exemple :

```text
HMAC
```

Une même clé secrète est utilisée pour produire et vérifier la signature.

```text
Secret
  |
  +--> Sign
  |
  +--> Verify
```

### Asymétrique

Exemple :

```text
RSA
ECDSA
```

On utilise une paire :

```text
Private Key -> Sign
Public Key  -> Verify
```

Mentalement :

```text
Private = secret
Public  = partageable
```

---

# 17. `AddAuthentication`

Cette méthode configure le système d'authentification.

Exemple :

```csharp
builder.Services
    .AddAuthentication("Bearer")
    .AddJwtBearer(options =>
    {
        // configuration
    });
```

Il faut distinguer :

```text
AddAuthentication
```

de :

```text
UseAuthentication
```

---

# 18. `AddAuthentication` vs `UseAuthentication`

### `AddAuthentication`

Configure et enregistre les services d'authentification.

```csharp
builder.Services.AddAuthentication(...);
```

### `UseAuthentication`

Ajoute le middleware d'authentification dans le pipeline HTTP.

```csharp
app.UseAuthentication();
```

Mentalement :

```text
AddAuthentication
    -> configure le système

UseAuthentication
    -> exécute l'authentification dans le pipeline
```

---

# 19. `UseAuthentication`

Dans le pipeline :

```csharp
app.UseAuthentication();
```

permet au système d'authentification de tenter d'établir l'identité de l'utilisateur.

Après cela, les mécanismes suivants peuvent utiliser :

```csharp
HttpContext.User
```

---

# 20. Ordre Authentication / Authorization

Configuration typique :

```csharp
app.UseAuthentication();

app.UseAuthorization();
```

Pourquoi cet ordre ?

Parce que l'autorisation doit généralement savoir :

```text
Qui est l'utilisateur ?
```

avant de décider :

```text
A-t-il le droit ?
```

Mentalement :

```text
Authentication
      |
      v
User / ClaimsPrincipal
      |
      v
Authorization
```

---

# 21. `[Authorize]`

On peut protéger une action :

```csharp
[Authorize]
[HttpGet]
public IActionResult GetPrivateData()
{
    return Ok();
}
```

L'utilisateur doit être authentifié selon la configuration de l'application.

---

# 22. `[AllowAnonymous]`

Pour autoriser explicitement l'accès à une action même lorsqu'une politique d'autorisation globale existe :

```csharp
[AllowAnonymous]
[HttpPost("login")]
public IActionResult Login(LoginDto dto)
{
    return Ok();
}
```

C'est notamment utile pour des endpoints comme :

```text
/login
/register
```

selon l'architecture.

---

# 23. Roles

On peut demander un rôle précis :

```csharp
[Authorize(Roles = "Admin")]
public IActionResult DeleteHotel(int id)
{
    return Ok();
}
```

Le principe :

```text
Authentication
    |
    v
Role = Admin
    |
    v
Authorization
    |
    v
Accès autorisé
```

Attention :

> Le rôle est une information utilisée par l'autorisation ; ce n'est pas l'authentification elle-même.

---

# 24. Policies

Les policies permettent une autorisation plus flexible que les simples rôles.

Exemple :

```csharp
[Authorize(Policy = "CanManageHotels")]
public IActionResult CreateHotel()
{
    return Ok();
}
```

La policy peut exprimer des règles basées sur :

```text
Claims
Roles
Requirements
Handlers
```

---

# 25. Claims-based authorization

Exemple :

```csharp
[Authorize(Policy = "CanManageHotels")]
```

La policy pourrait demander :

```text
Claim permission = hotel.write
```

Mentalement :

```text
User
 |
 +-- permission = hotel.read
 +-- permission = hotel.write
```

L'autorisation vérifie si le claim nécessaire existe.

---

# 26. Authorization Handler

Une policy complexe peut utiliser un handler.

Exemple conceptuel :

```csharp
public class CanManageHotelHandler
    : AuthorizationHandler<CanManageHotelRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        CanManageHotelRequirement requirement)
    {
        if (context.User.IsInRole("Admin"))
        {
            context.Succeed(requirement);
        }

        return Task.CompletedTask;
    }
}
```

Mentalement :

```text
Policy
   |
   v
Requirement
   |
   v
Handler
   |
   v
Autorisé / Refusé
```

---

# 27. Authentication avec ASP.NET Core Identity

ASP.NET Core Identity est une solution permettant de gérer notamment :

```text
Users
Passwords
Roles
Claims
Login
Registration
Tokens
```

Exemple conceptuel :

```csharp
builder.Services
    .AddIdentityCore<ApplicationUser>()
    .AddRoles<IdentityRole>()
    .AddEntityFrameworkStores<ApplicationDbContext>();
```

Identity gère les mécanismes liés aux utilisateurs, tandis que l'authentification d'une API peut utiliser différents mécanismes selon l'architecture.

---

# 28. `UserManager`

Avec ASP.NET Core Identity, `UserManager<TUser>` permet notamment de gérer les utilisateurs.

Exemple :

```csharp
var user = await userManager.FindByEmailAsync(email);
```

Il peut également être utilisé pour :

```text
Créer un utilisateur
Modifier un utilisateur
Vérifier un mot de passe
Ajouter des claims
Ajouter des rôles
```

---

# 29. `SignInManager`

`SignInManager<TUser>` fournit des mécanismes liés à la connexion des utilisateurs.

Il est particulièrement utilisé dans les scénarios basés sur Identity et cookies.

Mentalement :

```text
UserManager
    -> gestion du compte utilisateur

SignInManager
    -> gestion de la connexion / authentification
```

---

# 30. `MapIdentityApi`

Avec les versions modernes d'ASP.NET Core Identity, on peut exposer des endpoints Identity via :

```csharp
app.MapIdentityApi<ApplicationUser>();
```

Cela permet notamment de fournir des endpoints pour des opérations comme :

```text
Register
Login
Refresh
```

selon la configuration et la version du framework.

Mentalement :

```text
ASP.NET Core Identity
        |
        v
MapIdentityApi
        |
        v
Endpoints HTTP Identity
```

Cela évite de devoir créer manuellement tous les endpoints standards d'authentification.

---

# 31. Authentication Cookie vs JWT

### Cookie

Typiquement utilisé pour :

```text
Applications web avec navigateur
```

Le navigateur conserve le cookie et l'envoie automatiquement selon les règles du cookie.

### JWT Bearer

Typiquement utilisé pour :

```text
APIs
SPA
Applications mobiles
Services
```

Le client envoie généralement :

```http
Authorization: Bearer <token>
```

Attention :

> JWT n'est pas automatiquement « meilleur » que Cookie. Le choix dépend de l'architecture et du contexte.

---

# 32. Authentication dans une architecture API

Une architecture classique :

```text
Client
   |
   | Authorization: Bearer ...
   v
ASP.NET Core
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
   v
Controller
   |
   v
Service
```

Le Controller ne devrait généralement pas implémenter lui-même toute la validation cryptographique du token.

Le framework possède déjà les abstractions adaptées.

---

# 33. Lire l'utilisateur courant

Dans un Controller :

```csharp
var userId = User.FindFirst(
    ClaimTypes.NameIdentifier)?.Value;
```

Ou :

```csharp
var email = User.FindFirst(
    ClaimTypes.Email)?.Value;
```

Il est également possible d'utiliser :

```csharp
User.Identity?.Name
```

selon la façon dont les claims ont été configurés.

---

# 34. Ne pas faire confiance aux données du client

Une API ne doit pas considérer comme fiable une donnée envoyée directement par le client.

Exemple dangereux :

```json
{
  "userId": 42
}
```

et utiliser aveuglément :

```csharp
dto.UserId
```

pour déterminer l'identité de l'utilisateur connecté.

Si l'identité est issue de l'authentification, elle devrait normalement être déterminée depuis :

```csharp
HttpContext.User
```

ou le mécanisme d'identité approprié.

---

# 35. Authentication et DTOs

Un DTO de login peut contenir :

```csharp
public class LoginDto
{
    public string Email { get; set; } = string.Empty;

    public string Password { get; set; } = string.Empty;
}
```

Le client envoie :

```json
{
  "email": "user@example.com",
  "password": "..."
}
```

Le serveur :

```text
1. Reçoit le DTO
2. Recherche l'utilisateur
3. Vérifie le mot de passe
4. Authentifie l'utilisateur
5. Produit le mécanisme d'authentification attendu
```

---

# 36. Ne jamais stocker les mots de passe en clair

Une base de données ne doit pas contenir :

```text
Password = "MonMotDePasse123"
```

Les mots de passe doivent être traités par un système de hashing adapté.

ASP.NET Core Identity fournit des mécanismes conçus pour cela.

Règle :

> Un mot de passe n'est pas une donnée que l'application doit pouvoir « déchiffrer ».

---

# 37. Access Token vs Refresh Token

Dans les architectures utilisant des tokens, on peut rencontrer :

```text
Access Token
Refresh Token
```

### Access Token

Utilisé pour accéder aux ressources protégées.

### Refresh Token

Permet selon le système d'obtenir un nouveau token d'accès sans demander à l'utilisateur de se reconnecter de la même manière.

Mentalement :

```text
Login
  |
  +--> Access Token
  |
  +--> Refresh Token
```

Les stratégies exactes dépendent de l'architecture d'identité utilisée.

---

# 38. Expiration et sécurité

Un access token devrait généralement avoir une durée de vie limitée.

Pourquoi ?

Si un token est compromis :

```text
Token volé
    |
    v
Durée limitée
    |
    v
Fenêtre d'exploitation réduite
```

La durée exacte dépend du contexte et de la stratégie de sécurité.

---

# 39. Authentication n'est pas la gestion des utilisateurs

Il faut distinguer :

```text
User Management
Authentication
Authorization
```

### User Management

```text
Créer un compte
Modifier l'utilisateur
Changer le mot de passe
```

### Authentication

```text
Établir l'identité
```

### Authorization

```text
Décider les permissions
```

Identity peut participer aux trois, mais les concepts restent différents.

---

# 40. Erreurs fréquentes

## Erreur 1 : confondre Authentication et Authorization

```text
Authentication = Qui es-tu ?
Authorization = As-tu le droit ?
```

---

## Erreur 2 : oublier `UseAuthentication`

Configurer :

```csharp
AddAuthentication(...)
```

ne signifie pas automatiquement que le middleware est exécuté dans le pipeline.

Il faut généralement :

```csharp
app.UseAuthentication();
```

---

## Erreur 3 : placer Authentication après Authorization

Configuration typique :

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

L'ordre compte.

---

## Erreur 4 : mettre le mot de passe dans un JWT

Un JWT n'est pas un coffre-fort.

Évite d'y mettre des données sensibles inutiles.

---

## Erreur 5 : faire confiance à `userId` envoyé dans le body

Exemple :

```json
{
  "userId": 10
}
```

Ne signifie pas :

```text
l'utilisateur connecté est l'utilisateur 10
```

L'identité authentifiée vient du système d'authentification.

---

## Erreur 6 : considérer JWT comme toujours supérieur aux cookies

Le bon mécanisme dépend du scénario.

```text
Browser application -> Cookie peut être très adapté
API -> Bearer token peut être adapté
```

---

## Erreur 7 : mettre toute la sécurité dans le Controller

Le Controller ne devrait pas réimplémenter :

```text
validation cryptographique
parsing JWT
gestion des clés
```

Le framework fournit des abstractions dédiées.

---

# 41. Flux complet Authentication

```text
                HTTP REQUEST
                     |
                     v
       Authorization: Bearer <JWT>
                     |
                     v
            Authentication
                     |
                     v
        Authentication Handler
                     |
                     v
             Token Validation
                     |
          +----------+----------+
          |                     |
       invalide               valide
          |                     |
          v                     v
       401                  Claims
                                |
                                v
                       ClaimsPrincipal
                                |
                                v
                     HttpContext.User
                                |
                                v
                         Authorization
                                |
                     +----------+----------+
                     |                     |
                  refusé                autorisé
                     |                     |
                     v                     v
                    403                Controller
```

---

# 42. 401 vs 403

Question très fréquente en entretien.

### 401 Unauthorized

Le client n'est pas correctement authentifié.

Exemples :

```text
Token absent
Token invalide
Token expiré
```

### 403 Forbidden

L'utilisateur est authentifié mais n'a pas le droit demandé.

Exemple :

```text
User = authentifié
Role = User
Endpoint = Admin
```

Résultat :

```http
403 Forbidden
```

Mentalement :

```text
401 -> « Je ne sais pas qui tu es. »
403 -> « Je sais qui tu es, mais tu n'as pas le droit. »
```

---

# 43. Authentication et Middleware

Le système d'authentification s'intègre dans le pipeline HTTP.

Typiquement :

```csharp
app.UseAuthentication();
app.UseAuthorization();
```

Cela s'insère entre les autres composants selon les besoins de l'application.

Mentalement :

```text
HTTP
 |
 v
Middleware
 |
 v
Authentication
 |
 v
Authorization
 |
 v
Endpoint
```

---

# 44. Règle mentale

Quand tu vois :

```csharp
HttpContext.User
```

pense :

> « Voici l'identité construite pour la requête courante. »

Quand tu vois :

```csharp
[Authorize]
```

pense :

> « Cette ressource nécessite une autorisation selon la configuration. »

Quand tu vois :

```csharp
AddAuthentication()
```

pense :

> « Je configure les mécanismes d'authentification. »

Quand tu vois :

```csharp
UseAuthentication()
```

pense :

> « J'active l'authentification dans le pipeline. »

Quand tu vois :

```csharp
AddJwtBearer()
```

pense :

> « Je configure un mécanisme Bearer basé sur des tokens JWT. »

---

# 45. À retenir

1. Authentication répond à « Qui es-tu ? ».
2. Authorization répond à « As-tu le droit ? ».
3. L'utilisateur authentifié est généralement représenté par `HttpContext.User`.
4. `HttpContext.User` est un `ClaimsPrincipal`.
5. Les Claims représentent des informations sur l'identité.
6. Un Authentication Scheme indique quel mécanisme utiliser.
7. Un Authentication Handler implémente la logique du mécanisme.
8. JWT Bearer est un mécanisme courant pour les APIs.
9. Un JWT signé n'est pas automatiquement chiffré.
10. La signature protège notamment l'intégrité et l'authenticité du token.
11. Un JWT doit être validé : signature, expiration, issuer, audience, etc. selon la configuration.
12. `AddAuthentication()` configure les services.
13. `UseAuthentication()` exécute l'authentification dans le pipeline.
14. `UseAuthentication()` doit généralement être placé avant `UseAuthorization()`.
15. `[Authorize]` protège une ressource selon les règles d'autorisation.
16. Les roles et claims peuvent participer à l'autorisation.
17. Les policies permettent d'exprimer des règles d'autorisation plus riches.
18. Identity gère notamment les utilisateurs, mots de passe, rôles et claims.
19. `UserManager` sert principalement à gérer les utilisateurs.
20. `SignInManager` est lié aux mécanismes de connexion avec Identity.
21. `MapIdentityApi` permet d'exposer des endpoints Identity dans les versions modernes d'ASP.NET Core.
22. 401 signifie généralement problème d'authentification.
23. 403 signifie généralement utilisateur authentifié mais non autorisé.
24. L'identité ne doit pas être déterminée aveuglément à partir d'un `userId` envoyé par le client.
25. L'authentification, la gestion des utilisateurs et l'autorisation sont trois concepts différents.

---

# Questions d'entretien

### 1. Quelle différence entre Authentication et Authorization ?

Authentication :

```text
Qui es-tu ?
```

Authorization :

```text
As-tu le droit ?
```

### 2. Qu'est-ce qu'un Claim ?

Une information associée à l'identité d'un utilisateur, par exemple un identifiant, un nom, un rôle ou une permission.

### 3. Qu'est-ce que `HttpContext.User` ?

Le principal de sécurité associé à la requête courante, généralement un `ClaimsPrincipal`.

### 4. Quelle différence entre `AddAuthentication` et `UseAuthentication` ?

`AddAuthentication` configure/enregistre les services d'authentification ; `UseAuthentication` ajoute le middleware d'authentification au pipeline HTTP.

### 5. Pourquoi `UseAuthentication` doit-il généralement être avant `UseAuthorization` ?

Parce que l'autorisation doit normalement pouvoir utiliser l'identité établie par l'authentification.

### 6. Qu'est-ce qu'un Authentication Scheme ?

Un nom/configuration qui identifie le mécanisme d'authentification à utiliser, par exemple un scheme Bearer ou Cookies.

### 7. Qu'est-ce qu'un Authentication Handler ?

Le composant chargé de mettre en œuvre le traitement d'un mécanisme d'authentification donné.

### 8. Quelle différence entre 401 et 403 ?

```text
401 -> authentification absente ou invalide
403 -> authentifié mais accès refusé
```

### 9. Pourquoi un JWT signé n'est-il pas un endroit pour stocker des secrets ?

Parce qu'un JWT signé n'est généralement pas chiffré : son contenu peut être lu par quelqu'un qui possède le token.

### 10. Pourquoi ne faut-il pas faire confiance à `userId` envoyé dans un DTO ?

Parce que le client contrôle cette valeur. L'identité authentifiée doit provenir du mécanisme d'authentification, par exemple des claims du `ClaimsPrincipal`.

### 11. Cookie ou JWT ?

Cela dépend du scénario. Les cookies sont très adaptés à certaines applications web ; les Bearer tokens sont courants pour les APIs et autres clients.

---

# Phrase à retenir

> **Authentication établit l'identité et remplit `HttpContext.User`; Authorization utilise ensuite cette identité pour décider ce que l'utilisateur a le droit de faire.**
