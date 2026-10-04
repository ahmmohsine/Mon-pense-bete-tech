# Entretien — Sécurité des APIs .NET

Cette fiche rassemble les notions de sécurité importantes pour un entretien de développeuse C#/.NET orientée Web API.

L'objectif est de raisonner en termes de risques :

```text
Quelle donnée est exposée ?
        ↓
Qui peut y accéder ?
        ↓
Comment vérifier son identité ?
        ↓
Comment vérifier ses permissions ?
        ↓
Que se passe-t-il si l'entrée est malveillante ?
```

---

# 1. Les trois objectifs fondamentaux

On présente souvent la sécurité avec le triptyque :

```text
Confidentialité
Intégrité
Disponibilité
```

### Confidentialité

Empêcher une personne non autorisée d'accéder aux données.

Exemples :

```text
HTTPS
chiffrement
contrôle d'accès
```

### Intégrité

Empêcher ou détecter une modification non autorisée.

Exemples :

```text
signatures
contrôles d'accès
validation
```

### Disponibilité

Garantir que le service reste accessible.

Exemples :

```text
rate limiting
résilience
monitoring
protection contre les abus
```

---

# 2. Authentication vs Authorization

C'est une question incontournable.

### Authentication

Répond à :

```text
Qui es-tu ?
```

Exemple :

```text
login
password
JWT
cookie
Identity
```

### Authorization

Répond à :

```text
Qu'as-tu le droit de faire ?
```

Exemple :

```csharp
[Authorize]
```

ou :

```csharp
[Authorize(Roles = "Admin")]
```

Mental model :

```text
Authentication
→ identité

Authorization
→ permissions
```

---

# 3. ASP.NET Core Identity

ASP.NET Core Identity fournit des fonctionnalités pour gérer les utilisateurs et l'identité.

Il peut notamment gérer :

```text
users
passwords
roles
claims
tokens
login
registration
```

Un composant courant est :

```csharp
UserManager<TUser>
```

Exemple :

```csharp
public class UserService(
    UserManager<ApplicationUser> userManager)
{
}
```

Le `UserManager` fournit des opérations liées à la gestion des utilisateurs.

---

# 4. `SignInManager`

ASP.NET Core Identity fournit également :

```csharp
SignInManager<TUser>
```

Il est notamment utilisé pour les opérations de connexion et de gestion de l'authentification avec Identity.

Mental model :

```text
UserManager
→ gestion des utilisateurs

SignInManager
→ connexion / sign-in
```

---

# 5. Mot de passe

Il ne faut jamais stocker les mots de passe en clair.

Mauvais :

```text
password = "Azerty123"
```

Une application doit stocker un dérivé sécurisé du mot de passe, avec un mécanisme adapté de hashing.

ASP.NET Core Identity fournit les mécanismes nécessaires à cette gestion.

Mental model :

```text
password
   ↓
password hashing
   ↓
stored hash
```

Lors de la connexion :

```text
password fourni
   ↓
vérification
   ↓
hash stocké
```

---

# 6. Hashing vs Encryption

Très important.

### Hashing

Transformation conçue pour ne pas être réversible de manière normale.

Utilisé notamment pour :

```text
passwords
```

### Encryption

Transformation réversible avec une clé.

Utilisé notamment pour :

```text
données nécessitant d'être récupérées
```

Mental model :

```text
Hash
→ vérifier

Encryption
→ chiffrer / déchiffrer
```

---

# 7. Salt

Un salt est une valeur aléatoire associée au mot de passe avant le processus de hashing.

L'objectif est notamment de rendre les attaques basées sur des hashes pré-calculés beaucoup plus difficiles.

Les solutions modernes de gestion de mots de passe prennent généralement en charge ce mécanisme.

Avec ASP.NET Core Identity, il ne faut pas réinventer soi-même un système de hashing de mot de passe.

---

# 8. JWT

JWT signifie :

```text
JSON Web Token
```

Un JWT a généralement trois parties :

```text
Header.Payload.Signature
```

Exemple conceptuel :

```text
xxxxx.yyyyy.zzzzz
```

Le token peut contenir des claims.

---

# 9. JWT signé ≠ JWT chiffré

Point important en entretien.

Un JWT signé permet notamment de vérifier son intégrité et son authenticité selon le mécanisme utilisé.

Mais son payload n'est pas automatiquement secret.

Il ne faut donc pas y mettre :

```text
mot de passe
secret
donnée confidentielle
```

simplement parce que le token est signé.

Mental model :

```text
Signature
→ intégrité / authenticité

Encryption
→ confidentialité
```

---

# 10. Bearer Token

Une API utilisant un JWT peut recevoir :

```http
Authorization: Bearer eyJ...
```

Le serveur extrait le token puis le middleware d'authentification valide notamment :

```text
signature
issuer
audience
expiration
claims
```

selon la configuration.

---

# 11. Claims

Une claim représente une information sur l'identité.

Exemple :

```text
sub = 42
email = user@example.com
role = Admin
```

Dans ASP.NET Core :

```csharp
User.Claims
```

permet d'accéder aux claims de l'utilisateur authentifié.

---

# 12. Roles vs Policies

### Role

Exemple :

```csharp
[Authorize(Roles = "Admin")]
```

Le rôle représente une catégorie d'utilisateur.

### Policy

Exemple :

```csharp
[Authorize(Policy = "CanManageUsers")]
```

Une policy peut représenter une règle plus précise.

Mental model :

```text
Role
→ "Admin"

Policy
→ "possède la permission X"
```

Pour des systèmes complexes, les policies permettent souvent une autorisation plus flexible.

---

# 13. Resource-based authorization

Parfois, savoir qu'un utilisateur possède un rôle ne suffit pas.

Exemple :

```text
User A veut modifier Order 42
```

Il faut vérifier :

```text
User A est-il propriétaire de Order 42 ?
```

C'est une autorisation liée à la ressource.

Mental model :

```text
Identity
+
Resource
+
Action
→ authorization decision
```

---

# 14. Broken Object Level Authorization

Un problème fréquent dans les APIs est de vérifier que l'utilisateur est connecté sans vérifier qu'il a réellement accès à la ressource demandée.

Exemple dangereux :

```http
GET /api/orders/42
```

Le serveur vérifie :

```text
User authenticated = true
```

mais pas :

```text
Order 42 appartient-elle à User ?
```

Résultat :

```text
User A
→ peut consulter les données de User B
```

Il faut contrôler l'accès au niveau de la ressource.

---

# 15. IDOR

IDOR signifie :

```text
Insecure Direct Object Reference
```

Exemple :

```http
GET /users/42
GET /users/43
GET /users/44
```

Si changer simplement l'identifiant permet d'accéder aux données d'un autre utilisateur, l'API possède un problème d'autorisation.

La présence d'un ID difficile à deviner ne remplace pas une vraie autorisation.

---

# 16. DTO et Mass Assignment

Mauvaise pratique :

```csharp
public IActionResult Update(User user)
{
    ...
}
```

Le client peut potentiellement fournir des propriétés qui ne devraient pas être modifiables.

Exemple :

```json
{
  "name": "Ahlame",
  "isAdmin": true
}
```

On préfère :

```csharp
public class UpdateUserDto
{
    public string Name { get; set; } = "";
}
```

Puis on mappe explicitement :

```text
DTO
 ↓
propriétés autorisées
 ↓
Entity
```

---

# 17. Input Validation

Toutes les données provenant du client doivent être considérées comme non fiables.

Exemples :

```text
query string
route parameters
JSON
headers
files
form data
```

Il faut notamment vérifier :

```text
format
longueur
valeurs autorisées
taille
règles métier
```

Mais attention :

> Validation ≠ autorisation.

Une donnée valide peut quand même être interdite à l'utilisateur.

---

# 18. SQL Injection

Une SQL Injection survient lorsqu'une entrée utilisateur est interprétée comme une partie de la requête SQL.

Mauvaise approche :

```text
"SELECT * FROM Users WHERE Name = '" + input + "'"
```

L'ORM et les requêtes paramétrées réduisent fortement ce risque.

Avec EF Core :

```csharp
var users = await context.Users
    .Where(u => u.Name == name)
    .ToListAsync();
```

Le provider peut générer une requête paramétrée.

Il faut cependant rester prudent avec le SQL brut.

---

# 19. SQL brut avec EF Core

EF Core permet également du SQL brut dans certains scénarios.

Il faut distinguer les APIs sécurisées des constructions dangereuses basées sur la concaténation de chaînes.

Mauvais principe :

```text
SQL + input utilisateur concaténé
```

Bon principe :

```text
requête paramétrée
+
paramètres séparés
```

---

# 20. XSS

XSS signifie :

```text
Cross-Site Scripting
```

Un attaquant tente d'injecter du contenu qui sera interprété comme du script dans le navigateur.

Dans une API, le risque dépend notamment de la façon dont les données retournées sont consommées par le frontend.

Mesures possibles :

```text
output encoding
sanitization selon le contexte
Content Security Policy
validation
architecture frontend adaptée
```

La validation seule ne doit pas être considérée comme une protection universelle contre XSS.

---

# 21. CSRF

CSRF signifie :

```text
Cross-Site Request Forgery
```

Un site malveillant peut tenter de faire effectuer une action à un utilisateur authentifié.

Le risque est particulièrement lié aux mécanismes d'authentification automatiquement envoyés par le navigateur, notamment certains scénarios basés sur les cookies.

Les APIs utilisant un bearer token envoyé explicitement par le client ont généralement un modèle de risque différent.

---

# 22. CORS

CORS :

```text
Cross-Origin Resource Sharing
```

Il contrôle quelles origines peuvent effectuer certaines requêtes depuis un navigateur.

Exemple :

```text
Frontend
https://app.example.com

API
https://api.example.com
```

On peut configurer une policy :

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("Frontend", policy =>
    {
        policy
            .WithOrigins("https://app.example.com")
            .AllowAnyHeader()
            .AllowAnyMethod();
    });
});
```

---

# 23. CORS n'est pas un mécanisme d'authentification

Un client comme :

```text
Postman
curl
application mobile
script serveur
```

n'est pas soumis aux mêmes restrictions CORS qu'un navigateur.

Donc :

```text
CORS
→ sécurité navigateur

Authentication
→ identité côté serveur

Authorization
→ permissions côté serveur
```

---

# 24. HTTPS

Une API publique doit utiliser HTTPS.

HTTPS protège notamment contre l'interception des communications sur le réseau.

Il est indispensable pour protéger :

```text
credentials
JWT
cookies
données personnelles
données métier
```

Il faut éviter de transmettre des informations sensibles en HTTP non chiffré.

---

# 25. Secrets

Ne jamais committer :

```text
password
API key
JWT secret
connection string avec mot de passe
private key
```

dans un repository public.

Selon le contexte :

```text
User Secrets
Environment Variables
Azure Key Vault
Managed Identity
Secret Manager
```

peuvent être utilisés.

---

# 26. `.gitignore` n'est pas un coffre-fort

Mettre un secret dans :

```text
appsettings.Development.json
```

et ajouter le fichier à `.gitignore` réduit le risque de commit accidentel, mais ne constitue pas une solution universelle de gestion des secrets.

Si un secret a déjà été publié :

```text
le supprimer du fichier
+
révoquer / faire tourner le secret
+
nettoyer l'historique si nécessaire
```

La priorité est de considérer le secret comme compromis.

---

# 27. Rate Limiting

Le rate limiting limite le nombre de requêtes qu'un client peut effectuer.

Exemple conceptuel :

```text
100 requests
par minute
```

Il peut aider contre :

```text
abus
brute force
surcharge
certaines attaques automatisées
```

Exemple ASP.NET Core :

```csharp
builder.Services.AddRateLimiter(options =>
{
    ...
});
```

Puis :

```csharp
app.UseRateLimiter();
```

La configuration exacte dépend du scénario.

---

# 28. Rate limiting et login

Les endpoints sensibles doivent être particulièrement protégés.

Exemples :

```text
/login
/register
/password-reset
/otp
```

Un attaquant pourrait essayer :

```text
1000 passwords / seconde
```

Le rate limiting peut ralentir ces attaques.

Il doit être combiné avec d'autres mécanismes :

```text
password policy
lockout selon le contexte
MFA
monitoring
détection d'abus
```

---

# 29. Gestion des erreurs

Mauvais :

```json
{
  "error": "SqlException: connection failed to Server XYZ..."
}
```

Le client ne doit généralement pas recevoir :

```text
stack trace
connection string
noms internes
SQL
informations sensibles
```

On préfère une réponse contrôlée.

Exemple :

```json
{
  "title": "An unexpected error occurred.",
  "status": 500
}
```

Les détails techniques doivent être disponibles dans les logs sécurisés.

---

# 30. Exception handling centralisé

Au lieu de :

```csharp
try
{
}
catch (Exception ex)
{
    return StatusCode(500, ex);
}
```

dans chaque controller, on peut centraliser la gestion des exceptions.

Mental model :

```text
Exception
 ↓
global handler
 ↓
log sécurisé
 ↓
Problem Details
 ↓
HTTP response
```

---

# 31. Logging et sécurité

Les logs doivent aider à diagnostiquer les problèmes sans devenir une source de fuite de données.

Ne pas logger aveuglément :

```text
password
JWT complet
API keys
données personnelles sensibles
```

On peut logger :

```text
UserId
CorrelationId
operation
status
exception technique
```

selon les besoins et les règles de confidentialité.

---

# 32. Correlation ID

Un Correlation ID permet de suivre une requête à travers plusieurs composants.

Exemple :

```text
Request
CorrelationId = ABC123
```

Puis :

```text
API
 ↓
Service
 ↓
Database / external API
```

Les logs peuvent utiliser le même identifiant.

Cela facilite le diagnostic d'un problème distribué.

---

# 33. Principle of Least Privilege

Principe :

> Donner uniquement les permissions nécessaires.

Exemple :

Une API qui doit seulement lire une base ne devrait pas utiliser un compte possédant :

```text
DROP DATABASE
CREATE USER
ALTER SERVER
```

Le compte doit avoir les permissions minimales nécessaires.

Cela concerne :

```text
users
database
Azure resources
service identities
API permissions
```

---

# 34. Defense in Depth

Il ne faut pas compter sur une seule protection.

Exemple :

```text
HTTPS
+
Authentication
+
Authorization
+
Input validation
+
DTO
+
Rate limiting
+
Logging
+
Monitoring
```

Si une mesure échoue, les autres peuvent limiter l'impact.

Mental model :

```text
plusieurs couches
→ défense en profondeur
```

---

# 35. OWASP API Security

OWASP publie des recommandations et des listes de risques concernant les APIs.

Parmi les problèmes importants :

```text
Broken Object Level Authorization
Broken Authentication
Property Level Authorization
Unrestricted Resource Consumption
Broken Function Level Authorization
Unrestricted Access to Sensitive Business Flows
Server Side Request Forgery
Security Misconfiguration
Improper Inventory Management
Unsafe Consumption of APIs
```

L'objectif n'est pas de mémoriser uniquement les noms.

Il faut savoir reconnaître les scénarios.

---

# 36. Broken Function Level Authorization

Un utilisateur peut avoir accès à :

```http
GET /api/orders
```

mais ne devrait pas forcément pouvoir appeler :

```http
DELETE /api/users/42
```

Même si l'endpoint est techniquement accessible.

Il faut vérifier les permissions au niveau de l'action.

Exemple :

```csharp
[Authorize(Policy = "CanDeleteUsers")]
```

---

# 37. Unrestricted Resource Consumption

Une API peut être abusée avec des requêtes trop coûteuses.

Exemple :

```http
GET /products?pageSize=10000000
```

Si le serveur accepte cette valeur sans limite, il peut consommer énormément de :

```text
CPU
RAM
database resources
network
```

Mesures :

```text
pagination
maximum page size
rate limiting
timeouts
limites de payload
```

---

# 38. SSRF

SSRF signifie :

```text
Server-Side Request Forgery
```

Une API reçoit une URL et demande au serveur de la récupérer.

Exemple dangereux :

```http
POST /fetch
{
  "url": "http://internal-service/..."
}
```

L'attaquant peut tenter de faire appeler au serveur des ressources internes qui ne devraient pas être accessibles.

Il faut notamment :

```text
valider les destinations
allowlist selon le besoin
bloquer les réseaux internes
limiter les protocoles
```

---

# 39. Security Misconfiguration

Exemples :

```text
debug activé en production
secrets exposés
CORS trop permissif
headers incorrects
services inutiles activés
messages d'erreur trop détaillés
```

La configuration de production doit être revue explicitement.

---

# 40. Inventory Management

Une API peut posséder plusieurs versions :

```text
/api/v1
/api/v2
```

Un ancien endpoint oublié peut rester vulnérable.

Il faut donc savoir :

```text
quelles APIs existent
quelles versions sont exposées
qui les utilise
quelles versions doivent être retirées
```

---

# 41. API Gateway

Dans certaines architectures, une API Gateway peut fournir notamment :

```text
routing
authentication
rate limiting
TLS termination
logging
aggregation
```

Elle peut constituer une couche supplémentaire, mais ne remplace pas les contrôles de sécurité dans les services eux-mêmes.

---

# 42. Zero Trust

Le principe général est de ne pas considérer automatiquement qu'un réseau ou un composant est fiable.

Mental model :

```text
Never trust automatically
Always verify
```

On vérifie notamment :

```text
identity
permissions
context
device / service
```

selon l'architecture.

---

# 43. Sécurité d'une API .NET : réponse complète en entretien

Question :

> Comment sécuriseriez-vous une API ASP.NET Core ?

Réponse possible :

> Je commencerais par HTTPS et une authentification adaptée, par exemple ASP.NET Core Identity et/ou JWT selon l'architecture. Ensuite je mettrais en place une autorisation basée sur les rôles, claims ou policies, avec des contrôles au niveau des ressources lorsque nécessaire. Je séparerais les DTO des entités afin de contrôler les propriétés exposées et je validerais toutes les entrées. J'ajouterais du rate limiting sur les endpoints sensibles, une gestion centralisée des erreurs, des logs sécurisés et du monitoring. Les secrets seraient stockés dans un mécanisme dédié comme Azure Key Vault ou des variables d'environnement, jamais dans Git. Enfin, je vérifierais les risques OWASP API et appliquerais le principe du moindre privilège.

---

# 44. Mini scénario : API de commandes

Supposons :

```http
GET /api/orders/42
```

Utilisateur authentifié :

```text
UserId = 10
```

Commande :

```text
OrderId = 42
OwnerId = 25
```

Un simple :

```csharp
[Authorize]
```

ne suffit pas.

Pourquoi ?

Parce que :

```text
authenticated = true
```

mais :

```text
OwnerId != UserId
```

Il faut donc effectuer une autorisation liée à la ressource.

Mental model :

```text
Authentication
→ User 10 est connecté

Authorization
→ User 10 peut-il accéder à Order 42 ?
```

---

# 45. Mini scénario : modification utilisateur

Requête :

```json
{
  "name": "Ahlame",
  "isAdmin": true
}
```

Le DTO :

```csharp
public class UpdateUserDto
{
    public string Name { get; set; } = "";
}
```

permet d'empêcher que :

```text
isAdmin
```

soit directement modifié par cette opération.

Si la promotion en administrateur est autorisée, elle doit passer par une opération explicitement protégée.

---

# 46. Mini scénario : login

Un endpoint :

```http
POST /api/auth/login
```

doit notamment réfléchir à :

```text
brute force
rate limiting
messages d'erreur
password storage
MFA selon le besoin
token expiration
refresh tokens selon l'architecture
logging
```

Éviter des messages trop précis comme :

```text
"Cet email existe mais le mot de passe est incorrect."
```

qui peuvent faciliter l'énumération des comptes selon le contexte.

---

# 47. Questions d'entretien

### Qu'est-ce que l'authentification ?

> Vérifier l'identité d'un utilisateur ou d'un client.

### Qu'est-ce que l'autorisation ?

> Vérifier si cette identité possède les permissions nécessaires pour effectuer une action.

### Pourquoi utiliser JWT ?

> Pour transporter une identité et des claims sous forme de token signé dans des architectures où un mécanisme bearer token est adapté.

### Un JWT est-il chiffré ?

> Pas nécessairement. Un JWT signé protège notamment l'intégrité, mais son payload peut être lisible. Il ne faut donc pas y stocker des secrets.

### Pourquoi utiliser des DTO ?

> Pour contrôler le contrat de l'API, limiter les propriétés exposées ou modifiables et séparer le modèle HTTP du modèle interne.

### Qu'est-ce qu'un IDOR ?

> Une vulnérabilité où un utilisateur peut accéder à une ressource en manipulant directement un identifiant sans contrôle d'autorisation approprié.

### Pourquoi CORS ne suffit-il pas ?

> Parce que CORS concerne principalement les restrictions des navigateurs. Un client non navigateur peut appeler directement l'API.

### Pourquoi le rate limiting ?

> Pour limiter les abus, réduire certains risques de brute force et contrôler la consommation de ressources.

### Pourquoi ne pas retourner les exceptions au client ?

> Parce qu'elles peuvent révéler des informations internes sensibles comme des stack traces, SQL, chemins ou détails d'infrastructure.

### Qu'est-ce que le principe du moindre privilège ?

> Donner à chaque utilisateur ou composant uniquement les permissions nécessaires à son fonctionnement.

---

# 48. Questions pièges

## "JWT = sécurité complète"

Faux.

JWT est un mécanisme de transport d'informations d'identité.

Il faut encore gérer :

```text
expiration
validation
stockage côté client
révocation selon le besoin
refresh
authorization
```

---

## "Si l'utilisateur est authentifié, il peut accéder à toutes ses URLs"

Faux.

L'authentification ne donne pas automatiquement les permissions.

---

## "CORS protège l'API contre Postman"

Faux.

CORS est une protection principalement appliquée par les navigateurs.

---

## "Un GUID rend une ressource sécurisée"

Faux.

Un identifiant difficile à deviner ne remplace pas l'autorisation.

---

## "Hasher une donnée signifie la chiffrer"

Faux.

```text
Hash
→ vérification

Encryption
→ récupération avec une clé
```

---

## "Un secret dans un repository privé est toujours sécurisé"

Faux.

Un repository privé réduit l'exposition mais ne doit pas devenir le système de gestion des secrets.

---

# 49. Checklist Sécurité

```text
[ ] Confidentialité
[ ] Intégrité
[ ] Disponibilité
[ ] Authentication
[ ] Authorization
[ ] ASP.NET Core Identity
[ ] UserManager
[ ] SignInManager
[ ] Password hashing
[ ] Hash vs encryption
[ ] Salt
[ ] JWT
[ ] Claims
[ ] Roles
[ ] Policies
[ ] Resource authorization
[ ] IDOR / BOLA
[ ] DTO
[ ] Mass assignment
[ ] Input validation
[ ] SQL Injection
[ ] XSS
[ ] CSRF
[ ] CORS
[ ] HTTPS
[ ] Secrets
[ ] Key Vault
[ ] Rate limiting
[ ] Error handling
[ ] Secure logging
[ ] Least privilege
[ ] Defense in depth
[ ] OWASP API risks
[ ] SSRF
[ ] Security misconfiguration
[ ] API inventory
[ ] API Gateway
```

# À retenir

```text
Authentication
→ Qui es-tu ?

Authorization
→ Qu'as-tu le droit de faire ?

Identity
→ gestion des utilisateurs

JWT
→ token signé

Claims
→ informations sur l'identité

Policy
→ règle d'autorisation

DTO
→ contrôler les données entrantes/sortantes

BOLA / IDOR
→ accès à une ressource sans autorisation correcte

CORS
→ politique navigateur

HTTPS
→ communication protégée

Rate limiting
→ limiter les abus / consommation

Key Vault
→ gestion sécurisée des secrets

Least privilege
→ permissions minimales

Defense in depth
→ plusieurs couches de protection
```

## Phrase à mémoriser

> **Sécuriser une API ne signifie pas seulement ajouter un JWT : je dois protéger l'identité, les permissions et les ressources, contrôler les entrées et les secrets, limiter les abus, éviter les fuites d'informations et vérifier les risques de sécurité à chaque niveau de l'application.**
