# API Security — JWT

## 1. JWT : qu'est-ce que c'est ?

JWT signifie :

```text
JSON Web Token
```

Un JWT est un format de token permettant de transporter des informations sous forme de **claims**.

Dans une API, on rencontre très souvent :

```http
Authorization: Bearer <access-token>
```

Le client présente alors le token à l'API.

L'API ne demande pas simplement :

> « Est-ce que tu as un token ? »

Elle doit vérifier que le token est valide avant de construire l'identité de l'utilisateur.

ASP.NET Core fournit le handler `JwtBearerHandler` pour effectuer cette authentification. citeturn0search0turn0search9

---

# 2. JWT et Bearer : deux notions différentes

On voit souvent :

```text
JWT Bearer
```

mais il faut séparer les deux notions.

## JWT

JWT décrit principalement le **format du token**.

```text
HEADER.PAYLOAD.SIGNATURE
```

## Bearer

Bearer décrit la manière dont le token est présenté :

```http
Authorization: Bearer <token>
```

Le mot `Bearer` signifie essentiellement :

> Celui qui possède ce token peut le présenter.

Mental model :

```text
JWT
 ↓
Format du token

Bearer
 ↓
Mode de présentation du token
```

---

# 3. Exemple d'une requête

Le client appelle :

```http
GET /api/hotels
Authorization: Bearer eyJhbGciOi...
```

ASP.NET Core récupère le token et le handler JWT va effectuer les validations nécessaires.

Conceptuellement :

```text
HTTP Request
     |
     v
Authorization: Bearer <JWT>
     |
     v
JwtBearerHandler
     |
     v
Validation
     |
     v
ClaimsPrincipal
     |
     v
Authorization
     |
     v
Endpoint
```

---

# 4. Les trois parties d'un JWT

Un JWT classique contient trois parties séparées par un point :

```text
HEADER.PAYLOAD.SIGNATURE
```

Par exemple :

```text
eyJhbGciOiJIUzI1NiIs...
.
eyJzdWIiOiIxMjMiLCJyb2xlIjoiQWRtaW4ifQ
.
abc123...
```

Les trois parties sont :

```text
1. Header
2. Payload
3. Signature
```

---

# 5. Le Header

Le header contient des informations sur le token.

Exemple :

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

On peut notamment y trouver :

```text
alg
typ
```

`alg` indique l'algorithme utilisé pour la signature.

Exemples :

```text
HS256
RS256
ES256
```

Le choix de l'algorithme dépend de l'architecture et du système d'identité.

---

# 6. Le Payload

Le payload contient les claims.

Exemple :

```json
{
  "sub": "123",
  "name": "Ahlame",
  "role": "Admin",
  "iss": "https://identity.example.com",
  "aud": "hotel-api",
  "exp": 1790000000
}
```

On peut y trouver :

```text
sub
name
role
iss
aud
exp
```

Le payload transporte donc des informations qui pourront servir à construire l'identité et à prendre des décisions d'autorisation.

---

# 7. Attention : Base64Url n'est pas du chiffrement

C'est une erreur très fréquente.

Le payload d'un JWT peut être décodé.

Donc :

```text
JWT signé
≠
JWT chiffré
```

La signature protège principalement l'intégrité et permet de vérifier la confiance dans le token selon la configuration.

Il ne faut donc pas mettre de secret dans les claims simplement parce qu'elles sont dans un JWT.

Microsoft distingue d'ailleurs les access tokens qui peuvent varier de format et rappelle qu'un access token ne doit pas être interprété par l'interface qui le détient. citeturn0search0

---

# 8. La Signature

La troisième partie est la signature.

Conceptuellement :

```text
Header
   +
Payload
   +
Clé
   ↓
Signature
```

L'API peut utiliser la clé attendue pour vérifier que le token n'a pas été modifié et qu'il provient de la source de confiance attendue.

Supposons :

```json
{
    "role": "User"
}
```

Un attaquant modifie le payload :

```json
{
    "role": "Admin"
}
```

La signature ne correspondra plus au contenu modifié.

Le token doit être rejeté.

C'est pourquoi on ne doit jamais simplement décoder le JWT et faire confiance à son contenu.

---

# 9. Signature symétrique et asymétrique

Deux grandes familles sont importantes à comprendre.

## Symétrique

Une même clé sert à signer et à vérifier.

Exemple :

```text
HS256
```

Mental model :

```text
Issuer
   |
   | clé secrète
   v
Signature

API
   |
   | même clé secrète
   v
Validation
```

Le problème architectural est que toutes les parties qui doivent vérifier le token doivent connaître le secret.

---

## Asymétrique

Une paire de clés est utilisée :

```text
Private Key
Public Key
```

Le serveur d'identité signe avec :

```text
Private Key
```

L'API vérifie avec :

```text
Public Key
```

Mental model :

```text
Identity Provider
      |
      | Private Key
      v
    Signer
      |
      | JWT
      v
     API
      |
      | Public Key
      v
   Verify
```

Cette approche est très adaptée lorsqu'un système d'identité émet des tokens consommés par plusieurs APIs.

---

# 10. Les claims importantes

Certaines claims sont particulièrement importantes pour les access tokens.

## `iss` — Issuer

Indique l'émetteur du token.

```json
{
  "iss": "https://identity.example.com"
}
```

L'API doit vérifier que l'émetteur correspond à celui attendu.

---

## `aud` — Audience

Indique le destinataire prévu du token.

```json
{
  "aud": "hotel-api"
}
```

L'API doit vérifier que le token lui est destiné.

Un token valide destiné à une autre API ne doit pas être accepté simplement parce que sa signature est correcte.

---

## `exp` — Expiration

Indique la date d'expiration.

```json
{
  "exp": 1790000000
}
```

Après cette date, le token n'est plus valide.

---

## `sub` — Subject

Identifie généralement le sujet de l'identité.

Exemple :

```json
{
  "sub": "123"
}
```

Selon le système d'identité, cette valeur peut représenter l'identifiant de l'utilisateur.

---

## `iat` — Issued At

Indique quand le token a été émis.

```json
{
  "iat": 1789990000
}
```

---

## `jti` — JWT ID

Identifiant unique du token.

```json
{
  "jti": "abc-123"
}
```

Il peut être utile dans certains scénarios de suivi ou de révocation.

Microsoft indique notamment `iss`, `exp`, `aud`, `sub`, `client_id`, `iat` et `jti` parmi les claims attendues pour les access tokens OAuth 2.0. citeturn0search0

---

# 11. Les validations indispensables

Une API ne doit pas faire :

```text
Décoder JWT
    ↓
Lire role
    ↓
Faire confiance
```

Elle doit d'abord valider le token.

Les contrôles essentiels comprennent :

```text
Signature
Issuer
Audience
Expiration
```

Microsoft recommande de valider complètement les JWT bearer tokens côté API. citeturn0search0

Mental model :

```text
JWT
 ↓
Signature valide ?
 ↓
Issuer correct ?
 ↓
Audience correcte ?
 ↓
Pas expiré ?
 ↓
Oui
 ↓
Claims utilisables
```

---

# 12. `AddJwtBearer`

ASP.NET Core permet de configurer l'authentification JWT avec :

```csharp
builder.Services
    .AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = "https://identity.example.com";
        options.Audience = "hotel-api";
    });
```

Le package utilisé est :

```text
Microsoft.AspNetCore.Authentication.JwtBearer
```

Microsoft documente ce package comme le support ASP.NET Core permettant de valider les JWT bearer tokens. citeturn0search0turn0search6

---

# 13. Que signifie `Authority` ?

Dans une architecture utilisant un fournisseur d'identité :

```csharp
options.Authority = "https://identity.example.com";
```

l'`Authority` représente le serveur d'identité de confiance.

Conceptuellement :

```text
API
 |
 | "Je fais confiance à cet Identity Provider"
 v
Authority
```

L'API peut utiliser les métadonnées du fournisseur pour découvrir les informations nécessaires à la validation du token, selon la configuration.

---

# 14. Que signifie `Audience` ?

Exemple :

```csharp
options.Audience = "hotel-api";
```

Cela signifie que l'API attend des tokens destinés à :

```text
hotel-api
```

Mental model :

```text
Issuer
 ↓
"Qui a émis ce token ?"

Audience
 ↓
"À quelle API est-il destiné ?"
```

Ne pas confondre :

```text
Issuer ≠ Audience
```

---

# 15. `TokenValidationParameters`

On peut configurer explicitement les paramètres de validation.

Exemple :

```csharp
.AddJwtBearer(options =>
{
    options.TokenValidationParameters =
        new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateIssuerSigningKey = true,
            ValidateLifetime = true
        };
});
```

Le but est de définir les règles selon lesquelles le token doit être considéré comme valide.

Dans une application réelle, les valeurs attendues doivent correspondre au fournisseur de tokens utilisé.

Microsoft recommande de privilégier les valeurs par défaut lorsqu'elles correspondent au token provider, plutôt que de remplacer inutilement les mécanismes standards. citeturn0search0

---

# 16. `UseAuthentication`

Après avoir enregistré les services :

```csharp
builder.Services
    .AddAuthentication(...)
    .AddJwtBearer(...);
```

le pipeline doit utiliser l'authentification :

```csharp
app.UseAuthentication();
```

Puis l'autorisation :

```csharp
app.UseAuthorization();
```

Mental model :

```text
AddJwtBearer()
    ↓
Configuration des services

UseAuthentication()
    ↓
Utilisation dans le pipeline
```

Ce sont deux étapes différentes.

---

# 17. Protéger un endpoint

Avec un contrôleur :

```csharp
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class HotelsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetHotels()
    {
        return Ok();
    }
}
```

Avec une Minimal API :

```csharp
app.MapGet("/api/hotels", () =>
{
    return Results.Ok();
})
.RequireAuthorization();
```

Le JWT est alors utilisé pour établir l'identité avant que l'autorisation décide si l'accès est permis.

---

# 18. Lire les claims

Une fois le JWT validé, les claims peuvent être accessibles via :

```csharp
HttpContext.User
```

Exemple :

```csharp
var userId =
    User.FindFirst("sub")?.Value;
```

Ou :

```csharp
var role =
    User.FindFirst("role")?.Value;
```

Ou :

```csharp
var email =
    User.FindFirst("email")?.Value;
```

Le point important :

```text
JWT validé
     ↓
Claims
     ↓
ClaimsPrincipal
     ↓
HttpContext.User
```

---

# 19. JWT et Authorization

Un JWT n'est pas l'autorisation elle-même.

Il fournit notamment des informations permettant à l'application de déterminer l'identité et les permissions selon sa configuration.

Exemple :

```text
JWT
 |
 +-- sub = 123
 |
 +-- role = Admin
 |
 +-- aud = hotel-api
 |
 v
ClaimsPrincipal
 |
 v
Authorization
 |
 v
Admin ?
 |
 +-- Oui → accès
 +-- Non → 403
```

L'API doit toujours appliquer ses propres règles d'autorisation.

---

# 20. 401 avec un JWT

Si le token n'est pas valide, l'API peut répondre :

```http
401 Unauthorized
```

Exemples :

```text
Token absent
Signature incorrecte
Token expiré
Issuer incorrect
Audience incorrecte
```

Microsoft indique notamment qu'une signature invalide, une expiration ou des claims critiques invalides comme `aud` ou `iss` peuvent conduire à une réponse 401. citeturn0search0

---

# 21. 403 avec un JWT

Le token peut être parfaitement valide :

```text
Signature OK
Issuer OK
Audience OK
Expiration OK
```

mais l'utilisateur peut ne pas avoir les permissions nécessaires.

Exemple :

```text
role = User
```

Endpoint :

```text
Admin uniquement
```

Résultat :

```http
403 Forbidden
```

Donc :

```text
401 = token / authentication
403 = authorization
```

---

# 22. Access Token vs ID Token

C'est une distinction très importante avec OAuth/OIDC.

## Access Token

Il sert à accéder à une API.

```text
Client
   |
   | Access Token
   v
API
```

## ID Token

Il sert à communiquer des informations sur l'authentification de l'utilisateur au client dans OpenID Connect.

```text
Identity Provider
        |
        | ID Token
        v
     Client
```

Un ID token ne doit pas être utilisé comme access token pour appeler une API.

Microsoft le précise explicitement : les ID tokens servent à confirmer l'authentification de l'utilisateur et ne doivent pas être utilisés pour accéder aux APIs. citeturn0search0

---

# 23. OAuth 2.0 et OpenID Connect

JWT n'est pas un protocole d'authentification complet.

Il faut distinguer :

```text
JWT
 ↓
Format de token

OAuth 2.0
 ↓
Framework d'autorisation

OpenID Connect
 ↓
Couche d'identité au-dessus d'OAuth 2.0
```

Une architecture moderne peut donc utiliser :

```text
OpenID Connect
       +
OAuth 2.0
       +
JWT access tokens
```

Le rôle de l'API est principalement de valider le access token qu'elle reçoit.

Elle n'a pas besoin de connaître toute la procédure qui a permis au client d'obtenir le token.

Microsoft recommande de s'appuyer sur OAuth 2.0 / OpenID Connect pour l'acquisition sécurisée des tokens plutôt que d'inventer son propre mécanisme. citeturn0search0

---

# 24. Ne pas fabriquer son propre système JWT

Une erreur fréquente est de penser :

```csharp
"Je vais simplement créer un JWT moi-même
avec un login et un mot de passe."
```

Créer techniquement un JWT n'est pas difficile.

Construire correctement un système d'identité sécurisé est beaucoup plus complexe.

Il faut gérer :

```text
Login
Password
Token issuance
Signing keys
Key rotation
Expiration
Refresh
Revocation
Scopes
Claims
Consent
MFA
Account recovery
Security events
```

Microsoft recommande d'utiliser des standards établis comme OAuth 2.0 et OpenID Connect pour la création et l'acquisition des tokens, plutôt que de créer son propre protocole. citeturn0search0

---

# 25. Access Token et Refresh Token

Un access token est généralement de durée relativement courte.

```text
Login
  ↓
Access Token
  ↓
API
  ↓
Expiration
```

Un refresh token peut permettre d'obtenir un nouvel access token sans demander à nouveau les identifiants de l'utilisateur, selon le flux utilisé.

Mental model :

```text
Access Token
    ↓
courte durée
    ↓
API access

Refresh Token
    ↓
obtenir un nouveau access token
```

Les deux ne doivent pas être traités comme s'ils avaient exactement le même rôle.

---

# 26. Pourquoi un JWT volé est dangereux ?

Un bearer token fonctionne selon l'idée :

> Celui qui possède le token peut le présenter.

Si un attaquant obtient un access token valide :

```text
Attaquant
   |
   | Bearer Token volé
   v
API
```

l'API peut considérer la requête comme authentifiée tant que le token reste valide et satisfait les contrôles.

C'est pourquoi il faut notamment :

```text
HTTPS
Durée de vie raisonnable
Stockage sécurisé
Permissions minimales
Validation stricte
Protection du refresh token
```

Un JWT n'est donc pas sécurisé uniquement parce qu'il porte le nom JWT.

---

# 27. HTTPS est indispensable

Le token est une donnée sensible.

Une requête :

```http
Authorization: Bearer eyJ...
```

doit être protégée par HTTPS en production.

Mental model :

```text
HTTPS
 ↓
TLS
 ↓
Protection du transport
 ↓
Token moins exposé aux interceptions réseau
```

---

# 28. Plusieurs Identity Providers

Une API peut parfois devoir accepter des tokens provenant de plusieurs émetteurs.

ASP.NET Core permet de configurer plusieurs authentication schemes.

Exemple conceptuel :

```csharp
builder.Services
    .AddAuthentication()
    .AddJwtBearer("ProviderA", options =>
    {
        ...
    })
    .AddJwtBearer("ProviderB", options =>
    {
        ...
    });
```

Une policy scheme peut également être utilisée pour sélectionner le scheme approprié selon la requête ou certaines propriétés du token.

Microsoft documente cette approche pour les APIs qui doivent gérer plusieurs issuers. citeturn0search0turn0search11

---

# 29. `RequireHttpsMetadata`

Les options JWT comportent notamment :

```csharp
options.RequireHttpsMetadata
```

Cette option concerne l'utilisation de HTTPS pour les métadonnées de l'authority.

Sa valeur par défaut est :

```text
true
```

Elle ne doit normalement être désactivée que dans des scénarios de développement contrôlés. citeturn0search6

---

# 30. JWT et sécurité : le vrai modèle

Il faut éviter de penser :

```text
JWT = sécurité
```

Le vrai modèle est :

```text
Identity Provider
       |
       v
Token issuance
       |
       v
Access Token
       |
       v
HTTPS
       |
       v
API
       |
       v
Token validation
       |
       +-- Signature
       +-- Issuer
       +-- Audience
       +-- Expiration
       |
       v
ClaimsPrincipal
       |
       v
Authorization
       |
       v
Resource
```

La sécurité vient de l'ensemble.

---

# 31. Erreurs fréquentes

## Erreur 1 : décoder le JWT et lui faire confiance

Décoder :

```text
Base64Url.Decode(payload)
```

ne valide rien.

Le contenu doit être utilisé seulement après validation du token.

---

## Erreur 2 : oublier l'audience

Un token peut être correctement signé mais destiné à une autre API.

Toujours comprendre :

```text
iss = qui a émis ?
aud = pour qui ?
```

---

## Erreur 3 : confondre ID Token et Access Token

```text
ID Token
    → identité du client

Access Token
    → accès à une API
```

---

## Erreur 4 : mettre des secrets dans le payload

Le payload n'est pas un coffre-fort.

---

## Erreur 5 : créer son propre protocole d'authentification

Utiliser les standards :

```text
OAuth 2.0
OpenID Connect
```

lorsqu'ils correspondent au besoin.

---

## Erreur 6 : croire que JWT signifie toujours chiffrement

Un JWT signé peut être lisible.

```text
Signature ≠ Encryption
```

---

## Erreur 7 : confondre JWT et OAuth

```text
JWT = format
OAuth = framework d'autorisation
OIDC = identité
```

---

# 32. Checklist JWT

```text
[ ] JWT utilisé comme access token lorsque c'est approprié
[ ] HTTPS utilisé
[ ] Signature validée
[ ] Issuer validé
[ ] Audience validée
[ ] Expiration validée
[ ] Authentication scheme correctement configuré
[ ] UseAuthentication() présent
[ ] UseAuthorization() présent
[ ] Endpoints protégés
[ ] ID Token non utilisé comme access token
[ ] Pas de secrets dans le payload
[ ] Access tokens de durée raisonnable
[ ] Refresh tokens correctement protégés
[ ] Standards OAuth/OIDC utilisés lorsque nécessaire
[ ] Pas de protocole d'authentification maison
```

---

# 33. Questions d'entretien

### Que signifie JWT ?

JSON Web Token.

C'est un format de token contenant notamment des claims.

### Quelles sont les trois parties d'un JWT ?

```text
Header
Payload
Signature
```

### Le payload d'un JWT est-il chiffré ?

Pas nécessairement.

Un JWT signé peut être décodé.

### À quoi sert la signature ?

À permettre de vérifier l'intégrité du token et sa confiance selon les clés et règles configurées.

### Que vérifie une API JWT ?

Notamment :

```text
Signature
Issuer
Audience
Expiration
```

### Différence entre `iss` et `aud` ?

```text
iss = émetteur
aud = destinataire
```

### Différence entre Access Token et ID Token ?

```text
Access Token → accès à une API
ID Token     → informations d'identité pour le client
```

### Que fait `AddJwtBearer()` ?

Il configure le support de l'authentification JWT Bearer dans ASP.NET Core.

### À quoi sert `Authority` ?

À identifier le fournisseur d'identité de confiance utilisé pour obtenir les informations nécessaires à la validation des tokens.

### Pourquoi `Audience` est-elle importante ?

Pour vérifier que le token est destiné à l'API qui le reçoit.

### JWT est-il un protocole d'authentification ?

Non.

C'est principalement un format de token.

### Pourquoi ne faut-il pas inventer son propre système de tokens ?

Parce qu'un système d'identité sécurisé nécessite beaucoup plus que la simple création d'un JWT. Les standards OAuth 2.0 et OpenID Connect existent précisément pour fournir des flux et mécanismes standardisés.

---

# 34. Schéma mental final

```text
                 CLIENT
                    |
                    | Bearer JWT
                    v
                  API
                    |
                    v
            JwtBearerHandler
                    |
       +------------+------------+
       |            |            |
   Signature      Issuer      Audience
       |            |            |
       +------------+------------+
                    |
               Expiration
                    |
                    v
            ClaimsPrincipal
                    |
                    v
              Authorization
                    |
              +-----+-----+
              |           |
            403          OK
              |           |
              v           v
           REFUS        ENDPOINT
```

---

# À retenir

Un JWT n'est pas simplement une chaîne de caractères contenant un utilisateur.

C'est un token dont l'API doit vérifier la validité avant de faire confiance aux informations qu'il transporte.

Le modèle essentiel est :

```text
JWT
 ↓
Validation
 ↓
Identity
 ↓
Authorization
 ↓
Resource
```

Et les quatre validations à retenir en priorité :

```text
Signature
Issuer
Audience
Expiration
```

## Phrase à mémoriser

> **Un JWT n'est pas fiable parce que je peux le décoder ; il devient exploitable parce que l'API a validé sa signature, son émetteur, son audience et sa durée de validité.**
