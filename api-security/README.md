# API Security — Vue d'ensemble

## 1. Pourquoi sécuriser une API ?

Une API expose des fonctionnalités et des données à l'extérieur de l'application.

Exemple :

```text
Client
  ↓
HTTP Request
  ↓
ASP.NET Core API
  ↓
Application
  ↓
Database
```

Sans protections adaptées, un attaquant peut tenter de :

```text
Lire des données auxquelles il n'a pas accès
Modifier des données
Supprimer des données
Contourner l'autorisation
Surcharger l'API
Exploiter des entrées malveillantes
Voler des informations sensibles
```

La sécurité d'une API ne se résume donc pas à :

```text
JWT
```

Elle repose sur plusieurs couches.

---

# 2. Les grandes questions de sécurité

Pour chaque endpoint, il faut pouvoir répondre à plusieurs questions.

### Qui es-tu ?

```text
Authentication
```

### As-tu le droit de faire cela ?

```text
Authorization
```

### Quelles données peux-tu réellement voir ?

```text
Object-level authorization
```

### Les données reçues sont-elles valides ?

```text
Validation
```

### Combien de requêtes peux-tu envoyer ?

```text
Rate limiting
```

### Que se passe-t-il si quelque chose échoue ?

```text
Error handling
```

### Les données sont-elles protégées pendant le transport ?

```text
HTTPS / TLS
```

---

# 3. Authentication vs Authorization

C'est l'une des distinctions les plus importantes.

## Authentication

Elle répond :

> « Qui es-tu ? »

Exemples :

```text
Username + Password
JWT
Cookie
OpenID Connect
Identity
```

Après authentification, ASP.NET Core peut construire :

```text
HttpContext.User
```

---

## Authorization

Elle répond :

> « As-tu le droit de faire cette action ? »

Exemples :

```text
Role
Claim
Policy
Permission
Resource ownership
```

Donc :

```text
Authentication
    ↓
Qui es-tu ?

Authorization
    ↓
Que peux-tu faire ?
```

---

# 4. Exemple

Utilisateur :

```text
Alice
```

s'authentifie.

Authentication :

```text
Alice est bien authentifiée.
```

Mais Alice tente :

```http
DELETE /api/users/25
```

Authorization :

```text
Alice possède-t-elle la permission
de supprimer cet utilisateur ?
```

Elle peut être :

```text
Authentifiée = Oui
Autorisée = Non
```

---

# 5. 401 vs 403

Ces deux codes HTTP sont souvent confondus.

## 401 Unauthorized

Signifie généralement :

```text
Authentification absente ou invalide.
```

Exemple :

```http
GET /api/profile
Authorization: Bearer token-invalide
```

Réponse :

```http
401 Unauthorized
```

Mentalement :

> « Je ne peux pas établir correctement ton identité. »

---

## 403 Forbidden

Signifie généralement :

```text
Identité connue
+
permission insuffisante
```

Exemple :

```text
User
  ↓
endpoint Admin
```

L'utilisateur est connecté mais n'a pas le droit.

Réponse :

```http
403 Forbidden
```

Mentalement :

> « Je sais qui tu es, mais tu n'as pas le droit. »

---

# 6. JWT

Un JWT (**JSON Web Token**) est souvent utilisé pour transporter des informations signées concernant l'utilisateur.

Structure :

```text
Header.Payload.Signature
```

Exemple conceptuel :

```text
xxxxx.yyyyy.zzzzz
```

Un JWT contient notamment des claims.

Exemples :

```text
sub
name
role
email
permission
```

Attention :

> Un JWT signé n'est pas automatiquement chiffré.

Les données du payload peuvent généralement être décodées.

Il ne faut donc pas y placer de secrets.

---

# 7. Signature JWT

La signature permet notamment de vérifier que le token n'a pas été modifié et qu'il a été produit par une source qui possède la clé appropriée.

Conceptuellement :

```text
Header
+
Payload
+
Secret / Private Key
        ↓
Signature
```

Lors de la validation :

```text
Token reçu
        ↓
Vérification signature
        ↓
Token accepté ou rejeté
```

---

# 8. JWT ne signifie pas automatiquement "sécurisé"

Un JWT mal configuré peut être problématique.

Il faut notamment réfléchir à :

```text
Signature
Issuer
Audience
Expiration
Signing key
HTTPS
Claims
Token lifetime
Refresh token strategy
```

Et surtout :

```text
Authorization
```

doit être correctement configurée.

Un token valide ne signifie pas :

> « Cet utilisateur peut tout faire. »

---

# 9. Expiration

Un token d'accès doit généralement avoir une durée de vie limitée.

Exemple conceptuel :

```text
Access token
    ↓
15 minutes
```

Si le token est volé, sa durée d'exploitation est ainsi limitée.

On peut ensuite utiliser un mécanisme de refresh token selon l'architecture.

---

# 10. HTTPS

Les credentials et tokens doivent être protégés pendant le transport.

Sans HTTPS :

```text
Client
  ↓
HTTP
  ↓
réseau
```

Des données sensibles peuvent être exposées.

Avec HTTPS :

```text
Client
  ↓
TLS
  ↓
API
```

Le transport est chiffré.

En production :

> Une API d'authentification ne doit pas être conçue autour d'un transport HTTP non sécurisé.

---

# 11. DTOs et sécurité

Les DTOs ne servent pas uniquement à organiser le code.

Ils peuvent également limiter les données exposées.

Mauvais exemple :

```csharp
public class User
{
    public int Id { get; set; }

    public string Email { get; set; }

    public string PasswordHash { get; set; }

    public bool IsAdmin { get; set; }
}
```

Retourner directement cette entité peut exposer des données qui ne devraient jamais apparaître dans la réponse.

Préférer :

```csharp
public record UserResponse(
    int Id,
    string Email);
```

Mentalement :

```text
Entity
   ↓
DTO
   ↓
HTTP
```

Le DTO contrôle le contrat externe.

---

# 12. Mass Assignment / Overposting

Un autre risque consiste à accepter directement un objet contenant des propriétés que le client ne devrait pas contrôler.

Exemple :

```csharp
public class UpdateUserRequest
{
    public string Name { get; set; }

    public bool IsAdmin { get; set; }
}
```

Si le client peut envoyer :

```json
{
  "name": "Alice",
  "isAdmin": true
}
```

il peut tenter de modifier une propriété sensible.

Une API doit donc contrôler explicitement les champs que le client est autorisé à modifier.

---

# 13. Validation des entrées

Toute donnée provenant du client doit être considérée comme non fiable.

Exemples :

```text
Query string
Route parameter
JSON body
Headers
Form data
Files
```

On valide :

```text
Format
Longueur
Valeurs autorisées
Contraintes métier
```

Exemple :

```csharp
public record CreateHotelRequest(
    [property: Required]
    string Name);
```

Mais attention :

> La validation de format n'est pas toujours suffisante pour une règle métier.

---

# 14. Validation vs règles métier

Exemple :

```text
Name obligatoire
```

peut être une validation.

Mais :

```text
Une réservation ne peut pas chevaucher
une réservation existante.
```

est une règle métier.

Donc :

```text
Validation
    ↓
Les données sont-elles acceptables ?

Domain
    ↓
L'opération respecte-t-elle les règles métier ?
```

Les deux sont nécessaires.

---

# 15. Injection SQL

Une API qui construit du SQL avec des chaînes provenant du client peut être vulnérable.

Mauvais principe :

```csharp
var sql =
    $"SELECT * FROM Users WHERE Name = '{name}'";
```

Le contenu de `name` devient une partie du SQL.

Avec EF Core, on utilise généralement LINQ :

```csharp
var user = await _context.Users
    .FirstOrDefaultAsync(
        u => u.Name == name,
        cancellationToken);
```

EF Core génère une requête paramétrée dans ce scénario.

---

# 16. SQL brut

Le SQL brut peut être nécessaire dans certains cas.

Il faut alors utiliser les mécanismes de paramétrage appropriés.

Le principe à retenir :

```text
Donnée utilisateur
        ↓
paramètre SQL
```

et non :

```text
Donnée utilisateur
        ↓
concaténation dans SQL
```

---

# 17. Authorization par ressource

Avoir :

```csharp
[Authorize]
```

ne signifie pas nécessairement que l'utilisateur a le droit d'accéder à **toutes** les ressources.

Exemple :

```http
GET /api/orders/123
```

L'utilisateur est authentifié.

Mais il faut encore vérifier :

```text
Order 123 appartient-il à cet utilisateur ?
```

C'est une autorisation au niveau de la ressource.

---

# 18. Exemple de problème IDOR

Supposons :

```http
GET /api/orders/123
```

puis :

```http
GET /api/orders/124
```

Si l'API retourne la commande 124 uniquement parce que l'utilisateur connaît son ID, elle peut exposer les données d'un autre utilisateur.

L'ID n'est pas une autorisation.

Il faut vérifier la relation :

```text
CurrentUser
     ↓
possède
     ↓
Order
```

---

# 19. Roles

ASP.NET Core permet notamment :

```csharp
[Authorize(Roles = "Admin")]
```

Cela signifie que l'utilisateur doit avoir le rôle approprié.

Exemple :

```text
Admin
Manager
User
```

Les rôles sont utiles pour des catégories globales d'accès.

Mais ils peuvent devenir insuffisants lorsque les permissions deviennent complexes.

---

# 20. Claims

Un claim représente une information concernant l'identité.

Exemples :

```text
sub = 123
email = alice@example.com
role = Admin
department = Sales
permission = hotel.read
```

ASP.NET Core peut utiliser ces claims dans l'autorisation.

---

# 21. Policies

Les policies permettent de centraliser des règles d'autorisation.

Exemple :

```csharp
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy(
        "CanManageHotels",
        policy =>
            policy.RequireClaim(
                "permission",
                "hotel.manage"));
});
```

Puis :

```csharp
[Authorize(Policy = "CanManageHotels")]
```

Cela permet d'exprimer :

```text
Endpoint
   ↓
Policy
   ↓
Permission
```

---

# 22. Pourquoi préférer parfois les policies aux rôles ?

Avec uniquement des rôles :

```text
Admin
Manager
User
```

les permissions peuvent devenir difficiles à gérer.

Avec des permissions :

```text
hotel.read
hotel.create
hotel.update
hotel.delete
```

on peut composer des règles plus fines.

Exemple :

```text
Manager
    → hotel.read
    → hotel.update

Admin
    → hotel.read
    → hotel.create
    → hotel.update
    → hotel.delete
```

Le choix dépend du domaine et de la complexité du système.

---

# 23. Rate Limiting

Une API peut être attaquée par un grand nombre de requêtes.

Exemple :

```text
Attaquant
   ↓
100 000 requests
   ↓
API
```

Cela peut provoquer :

```text
Surcharge
Brute force
Dégradation du service
Coût infrastructure
```

Le **Rate Limiting** limite le nombre de requêtes autorisées selon une règle.

Exemple conceptuel :

```text
100 requests / minute / client
```

---

# 24. Rate Limiting et authentification

Pour un endpoint de login, une limitation peut être particulièrement importante.

Exemple :

```text
POST /api/auth/login
```

Un attaquant peut essayer :

```text
password1
password2
password3
...
```

Le rate limiting peut réduire la vitesse de l'attaque.

Il ne remplace pas :

```text
MFA
Password policy
Account protection
Monitoring
```

mais ajoute une couche de défense.

---

# 25. Gestion des erreurs

Une API ne devrait pas retourner :

```text
stack trace complète
connection string
SQL interne
chemins du serveur
secrets
```

à un client externe.

Mauvais exemple :

```json
{
  "exception": "SqlException...",
  "stackTrace": "..."
}
```

En production, on préfère une réponse contrôlée.

Exemple conceptuel :

```json
{
  "title": "An unexpected error occurred.",
  "status": 500
}
```

ASP.NET Core peut utiliser :

```text
ProblemDetails
```

pour standardiser les erreurs HTTP.

---

# 26. Logging et sécurité

Les logs sont importants pour diagnostiquer les attaques.

Mais il ne faut pas logger aveuglément :

```text
Password
Access Token
Refresh Token
Secret
Credit card data
Sensitive personal data
```

Exemple dangereux :

```csharp
_logger.LogInformation(
    "Login with password {Password}",
    password);
```

Les logs peuvent être accessibles à davantage de personnes et de systèmes que prévu.

---

# 27. Secrets

Ne pas mettre les secrets directement dans :

```text
Git
appsettings.json
source code
```

Exemples :

```text
JWT signing key
Database password
API key
SMTP password
Azure credentials
```

Selon l'environnement, utiliser des mécanismes adaptés :

```text
User Secrets
Environment variables
Azure Key Vault
Secret manager
```

---

# 28. CORS

CORS (**Cross-Origin Resource Sharing**) contrôle quels origins peuvent effectuer certaines requêtes cross-origin vers l'API depuis un navigateur.

Exemple :

```text
Frontend
https://myapp.com

API
https://api.myapp.com
```

On peut configurer les origins autorisés.

Attention :

> CORS n'est pas un mécanisme d'authentification.

Il contrôle principalement les restrictions imposées par les navigateurs pour les requêtes cross-origin.

---

# 29. CORS mal configuré

Une configuration trop permissive peut être problématique.

Exemple à éviter sans justification :

```csharp
.AllowAnyOrigin()
.AllowAnyMethod()
.AllowAnyHeader()
```

Il faut définir une politique correspondant réellement au besoin.

CORS doit être considéré comme une politique navigateur, pas comme une protection générale de l'API contre tous les clients.

---

# 30. HTTPS et cookies

Lorsque l'authentification utilise des cookies, des protections supplémentaires peuvent être pertinentes :

```text
Secure
HttpOnly
SameSite
```

### Secure

Le cookie est transmis via HTTPS.

### HttpOnly

Le JavaScript côté navigateur ne peut normalement pas lire le cookie.

### SameSite

Contrôle notamment certains comportements cross-site du cookie.

Le choix exact dépend de l'architecture d'authentification.

---

# 31. Passwords

Les mots de passe doivent être **hachés**, pas chiffrés comme une donnée qu'on pourrait simplement déchiffrer.

On ne doit pas stocker :

```text
password = "Azerty123!"
```

Une bibliothèque d'authentification appropriée doit gérer :

```text
Hash
Salt
Verification
Password policy
```

Avec ASP.NET Core Identity, on utilise les mécanismes fournis par Identity.

---

# 32. Principle of Least Privilege

Le principe du moindre privilège signifie :

> Donner uniquement les permissions nécessaires.

Exemple :

Un compte utilisé par une API qui doit seulement lire une table ne devrait pas automatiquement avoir :

```text
DROP DATABASE
DELETE EVERYTHING
CREATE LOGIN
```

Ce principe s'applique à :

```text
Users
Roles
Database accounts
Azure identities
API keys
Services
```

---

# 33. Defense in Depth

Une bonne sécurité repose sur plusieurs couches.

Exemple :

```text
HTTPS
  ↓
Authentication
  ↓
Authorization
  ↓
Validation
  ↓
Rate limiting
  ↓
Database security
  ↓
Logging / Monitoring
```

Si une couche échoue, une autre peut limiter l'impact.

Mentalement :

> Ne jamais dépendre d'une seule protection.

---

# 34. OWASP API Security

OWASP publie des ressources consacrées aux risques de sécurité des APIs.

Parmi les catégories importantes :

```text
Broken Object Level Authorization
Broken Authentication
Broken Object Property Level Authorization
Unrestricted Resource Consumption
Broken Function Level Authorization
Unrestricted Access to Sensitive Business Flows
Server Side Request Forgery
Security Misconfiguration
Improper Inventory Management
Unsafe Consumption of APIs
```

Ces risques montrent pourquoi :

```text
Authentication
```

seule ne suffit pas.

---

# 35. Broken Object Level Authorization

C'est un problème particulièrement important pour les APIs.

Exemple :

```http
GET /api/users/42
```

Le serveur doit vérifier :

```text
L'utilisateur connecté
        ↓
a-t-il le droit
        ↓
d'accéder à User 42 ?
```

Ne jamais considérer :

```text
ID = permission
```

---

# 36. Broken Function Level Authorization

Exemple :

```http
DELETE /api/users/42
```

Le serveur doit vérifier non seulement :

```text
Utilisateur authentifié ?
```

mais aussi :

```text
Utilisateur autorisé à supprimer ?
```

Une route cachée dans l'interface graphique n'est pas une protection.

La sécurité doit être appliquée côté serveur.

---

# 37. Inventory Management

Une API peut avoir plusieurs versions :

```text
/api/v1
/api/v2
/api/internal
/api/test
```

Une ancienne API oubliée peut rester vulnérable.

Il faut donc savoir :

```text
Quels endpoints existent ?
Quelles versions sont actives ?
Qui peut les appeler ?
Quels endpoints sont obsolètes ?
```

---

# 38. SSRF

**Server-Side Request Forgery** concerne notamment les situations où une API récupère une URL fournie par un utilisateur puis demande elle-même cette ressource.

Exemple conceptuel :

```text
Client
  ↓
API : "Télécharge cette URL"
  ↓
URL contrôlée par attaquant
  ↓
Ressource interne
```

L'API peut alors être utilisée comme intermédiaire pour atteindre des ressources qui ne devraient pas être accessibles.

Les URLs externes doivent donc être traitées avec prudence.

---

# 39. Sécurité des fichiers

Si l'API accepte des uploads :

```text
POST /api/files
```

il faut réfléchir à :

```text
Taille maximale
Extension
Content-Type
Contenu réel
Nom du fichier
Stockage
Exécution éventuelle
Path traversal
Malware
```

Ne jamais considérer :

```text
file.jpg
```

comme une preuve suffisante que le contenu est réellement une image sûre.

---

# 40. Pagination et consommation de ressources

Un endpoint comme :

```http
GET /api/orders
```

ne devrait pas nécessairement retourner des millions de lignes.

Préférer :

```http
GET /api/orders?page=1&pageSize=50
```

avec une limite maximale :

```text
pageSize <= 100
```

Cela protège notamment contre la consommation excessive de ressources.

---

# 41. Protection contre les gros payloads

Une API doit également contrôler :

```text
Taille du body
Nombre d'éléments
Taille des fichiers
Complexité des requêtes
Temps d'exécution
```

Exemple :

```json
{
  "items": [
    "... des millions d'éléments ..."
  ]
}
```

Même si le JSON est valide, il peut représenter une attaque de consommation de ressources.

---

# 42. Security Headers

Selon l'application, des headers de sécurité peuvent contribuer à la protection du navigateur et du transport.

Exemples :

```text
Content-Security-Policy
Strict-Transport-Security
X-Content-Type-Options
Referrer-Policy
```

La pertinence de chaque header dépend du type d'application et de son exposition.

---

# 43. Sécurité et observabilité

Une API sécurisée doit également être observable.

Il faut pouvoir détecter :

```text
Tentatives de login répétées
403 anormalement nombreux
404 sur de nombreux IDs
Rate limit déclenché
Erreurs inhabituelles
Accès à des endpoints sensibles
```

La sécurité ne consiste pas uniquement à bloquer.

Elle consiste aussi à :

```text
Détecter
+
Analyser
+
Réagir
```

---

# 44. Checklist de sécurité API

Pour un endpoint :

### Transport

- [ ] HTTPS
- [ ] TLS correctement configuré

### Authentication

- [ ] Identité correctement vérifiée
- [ ] Tokens correctement validés
- [ ] Expiration appropriée

### Authorization

- [ ] `[Authorize]` si nécessaire
- [ ] Roles / Claims / Policies
- [ ] Vérification des ressources
- [ ] Vérification des permissions côté serveur

### Input

- [ ] Validation
- [ ] DTOs
- [ ] Protection contre overposting
- [ ] Limitation des payloads

### Data

- [ ] Pas de secrets dans les réponses
- [ ] Pas de PasswordHash exposé
- [ ] Paramétrage SQL
- [ ] Principe du moindre privilège

### Availability

- [ ] Rate limiting
- [ ] Pagination
- [ ] Limites de taille
- [ ] Timeouts

### Errors

- [ ] Pas de stack trace publique
- [ ] ProblemDetails / erreurs contrôlées
- [ ] Logs utiles mais sans secrets

### Monitoring

- [ ] Logs
- [ ] Alertes
- [ ] Audit si nécessaire

---

# 45. Mental model

Pour chaque endpoint, pense :

```text
1. Qui es-tu ?
       ↓
2. As-tu le droit ?
       ↓
3. Quelle ressource peux-tu utiliser ?
       ↓
4. Les données reçues sont-elles valides ?
       ↓
5. La requête est-elle raisonnable ?
       ↓
6. Que se passe-t-il en cas d'erreur ?
       ↓
7. Peut-on détecter une attaque ?
```

Cette grille permet de ne pas réduire la sécurité à JWT.

---

# 46. Exemple complet

Endpoint :

```http
DELETE /api/hotels/15
```

Vérifications :

```text
HTTPS
  ↓
Authentication
  ↓
JWT valide ?
  ↓
User authentifié ?
  ↓
Permission hotel.delete ?
  ↓
Hotel 15 existe ?
  ↓
Utilisateur autorisé sur Hotel 15 ?
  ↓
Suppression
  ↓
Logging
```

Chaque étape répond à une question différente.

---

# 47. Questions d'entretien

### 1. Quelle différence entre 401 et 403 ?

`401` indique généralement un problème d'authentification. `403` indique généralement que l'identité est connue mais que l'accès est interdit.

### 2. JWT est-il chiffré ?

Pas par défaut. Un JWT signé permet notamment de vérifier son intégrité, mais son payload peut généralement être décodé.

### 3. Pourquoi utiliser des DTOs ?

Pour contrôler les données entrantes et sortantes et éviter d'exposer directement les entités et leurs propriétés sensibles.

### 4. `[Authorize]` suffit-il pour sécuriser une ressource ?

Non. Il peut être nécessaire de vérifier que l'utilisateur a réellement le droit d'accéder à la ressource demandée.

### 5. Qu'est-ce que l'IDOR/BOLA ?

C'est notamment une situation où un utilisateur peut accéder à une ressource en manipulant son identifiant sans que le serveur vérifie correctement l'autorisation.

### 6. Pourquoi utiliser Rate Limiting ?

Pour limiter la consommation excessive de ressources et réduire notamment certains abus comme le brute force.

### 7. Pourquoi ne faut-il pas logger les tokens ?

Parce qu'un log compromis pourrait permettre à quelqu'un de récupérer des informations d'authentification sensibles.

### 8. CORS sécurise-t-il l'API contre tous les clients ?

Non. CORS concerne principalement les restrictions des navigateurs pour les requêtes cross-origin. Un client non navigateur peut appeler directement l'API.

---

# À retenir

Une API sécurisée ne repose pas sur une seule technologie.

```text
HTTPS
   +
Authentication
   +
Authorization
   +
Validation
   +
DTOs
   +
Rate Limiting
   +
Database Security
   +
Error Handling
   +
Logging / Monitoring
```

Et la règle la plus importante :

> **Le serveur ne doit jamais faire confiance au client.**

# Phrase à mémoriser

> **Sécuriser une API, c'est vérifier l'identité, les permissions, les données, les ressources consommées et les erreurs à chaque frontière importante.**
