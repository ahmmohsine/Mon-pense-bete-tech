# API Security

Cette section regroupe les mécanismes et principes fondamentaux nécessaires pour sécuriser une API ASP.NET Core.

L'objectif n'est pas seulement de mémoriser JWT ou `[Authorize]`, mais de comprendre les différentes couches de sécurité et les responsabilités de chacune.

Une API sécurisée doit notamment répondre à plusieurs questions :

```text
Qui es-tu ?
    ↓
Authentication

As-tu le droit ?
    ↓
Authorization

Quelle ressource peux-tu utiliser ?
    ↓
Object-level authorization

Les données reçues sont-elles valides ?
    ↓
Validation

La requête est-elle raisonnable ?
    ↓
Rate limiting

Les données sont-elles protégées ?
    ↓
HTTPS / TLS

Que se passe-t-il en cas d'erreur ?
    ↓
Error handling

Peut-on détecter une attaque ?
    ↓
Logging / Monitoring
```

---

## Fiches

* [Authentication](authentication.md)
* [Authorization](authorization.md)
* [JWT](jwt.md)
* [DTOs et Rate Limiting](dto-et-rate-limiting.md)
* [Risques OWASP API Security](risques-owasp.md)

---

## Ordre conseillé

Pour comprendre progressivement la sécurité d'une API ASP.NET Core :

1. [Authentication](authentication.md)
2. [Authorization](authorization.md)
3. [JWT](jwt.md)
4. [DTOs et Rate Limiting](dto-et-rate-limiting.md)
5. [Risques OWASP API Security](risques-owasp.md)

L'idée est de commencer par comprendre l'identité, puis les permissions, les tokens et enfin les protections complémentaires et les principaux risques des APIs.

---

## Structure d'une fiche

Chaque fiche doit essayer de répondre à ces questions :

1. Qu'est-ce que c'est ?
2. Pourquoi ASP.NET Core utilise ce mécanisme ?
3. Quel problème cela résout ?
4. Comment cela fonctionne derrière la syntaxe ?
5. Comment l'utiliser correctement ?
6. Quelles sont les erreurs fréquentes ?
7. Quelle différence avec les mécanismes proches ?
8. Quels sont les risques de mauvaise configuration ?
9. Quelle règle mentale permet de s'en souvenir ?
10. Que peut-on demander en entretien ?

---

## Mental model

La sécurité d'une API peut être vue comme plusieurs couches :

```text
HTTPS / TLS
    ↓
Authentication
    ↓
Authorization
    ↓
Validation
    ↓
DTOs
    ↓
Rate Limiting
    ↓
Database Security
    ↓
Error Handling
    ↓
Logging / Monitoring
```

Chaque couche répond à une question différente.

### Authentication

```text
Qui es-tu ?
```

Elle permet d'établir l'identité de l'utilisateur ou du client.

### Authorization

```text
As-tu le droit ?
```

Elle détermine ce que l'utilisateur authentifié peut faire.

### Validation

```text
Les données reçues sont-elles valides ?
```

Elle vérifie que les données respectent les contraintes attendues.

### DTOs

```text
Quelles données peuvent entrer ou sortir de l'API ?
```

Ils permettent notamment de contrôler le contrat HTTP et d'éviter d'exposer directement les entités.

### Rate Limiting

```text
Combien de requêtes peuvent être envoyées ?
```

Il permet de limiter certains abus et la consommation excessive de ressources.

### OWASP

```text
Quels sont les principaux risques auxquels une API peut être exposée ?
```

OWASP fournit notamment une classification des risques courants liés à la sécurité des APIs.

---

## Authentication vs Authorization

C'est l'une des distinctions les plus importantes à retenir :

```text
Authentication
    ↓
Qui es-tu ?

Authorization
    ↓
Que peux-tu faire ?
```

Exemple :

```text
Alice se connecte
    ↓
Authentication
    ↓
Alice est authentifiée
    ↓
Elle tente DELETE /api/users/25
    ↓
Authorization
    ↓
Alice a-t-elle le droit de supprimer cet utilisateur ?
```

Une personne peut donc être :

```text
Authentifiée = Oui
Autorisé     = Non
```

---

## 401 vs 403

Mental model :

```text
401
↓
Je ne peux pas établir correctement ton identité.

403
↓
Je sais qui tu es,
mais tu n'as pas le droit.
```

---

## JWT

Un JWT est généralement utilisé pour transporter des informations signées concernant l'identité d'un utilisateur ou d'un client.

Structure :

```text
Header.Payload.Signature
```

À retenir :

```text
JWT signé ≠ JWT chiffré
```

Le payload peut généralement être décodé.

Il ne faut donc pas y placer de secrets.

---

## Authorization au niveau de la ressource

Une authentification réussie ne signifie pas que l'utilisateur peut accéder à toutes les ressources.

Exemple :

```http
GET /api/orders/123
```

L'API doit également vérifier :

```text
L'utilisateur est-il propriétaire
ou autorisé à accéder à Order 123 ?
```

L'identifiant d'une ressource n'est jamais une autorisation en lui-même.

---

## OWASP API Security

Les risques de sécurité des APIs comprennent notamment :

```text
Broken Object Level Authorization
Broken Authentication
Broken Object Property Level Authorization
Unrestricted Resource Consumption
Broken Function Level Authorization
Security Misconfiguration
Improper Inventory Management
Server Side Request Forgery
Unsafe Consumption of APIs
```

L'objectif est de comprendre ces risques et surtout de savoir comment les reconnaître dans une API réelle.

---

## Defense in Depth

Une API ne doit pas dépendre d'une seule protection.

Mental model :

```text
HTTPS
  +
Authentication
  +
Authorization
  +
Validation
  +
Rate Limiting
  +
Database Security
  +
Logging / Monitoring
```

Si une protection échoue, une autre couche peut limiter l'impact.

> Ne jamais dépendre d'une seule protection.

---

## Checklist de sécurité API

Pour chaque endpoint, vérifier :

### Transport

* [ ] HTTPS
* [ ] TLS correctement configuré

### Authentication

* [ ] Identité correctement vérifiée
* [ ] Tokens correctement validés
* [ ] Expiration appropriée

### Authorization

* [ ] `[Authorize]` lorsque nécessaire
* [ ] Roles / Claims / Policies
* [ ] Vérification des ressources
* [ ] Permissions vérifiées côté serveur

### Input

* [ ] Validation
* [ ] DTOs
* [ ] Protection contre l'overposting
* [ ] Limitation des payloads

### Data

* [ ] Pas de secrets dans les réponses
* [ ] Pas de `PasswordHash` exposé
* [ ] Requêtes SQL paramétrées
* [ ] Principe du moindre privilège

### Availability

* [ ] Rate limiting
* [ ] Pagination
* [ ] Limites de taille
* [ ] Timeouts

### Errors

* [ ] Pas de stack trace publique
* [ ] Erreurs contrôlées
* [ ] `ProblemDetails` lorsque pertinent
* [ ] Logs sans secrets

### Monitoring

* [ ] Logs
* [ ] Alertes
* [ ] Audit lorsque nécessaire

---

## Questions d'entretien

### Quelle différence entre 401 et 403 ?

`401` indique généralement un problème d'authentification.

`403` indique généralement que l'identité est connue mais que l'accès est interdit.

### JWT est-il chiffré ?

Pas par défaut.

Un JWT signé permet notamment de vérifier son intégrité, mais son payload peut généralement être décodé.

### `[Authorize]` suffit-il pour sécuriser une ressource ?

Non.

Il peut être nécessaire de vérifier que l'utilisateur a réellement le droit d'accéder à la ressource demandée.

### Pourquoi utiliser des DTOs ?

Pour contrôler les données entrantes et sortantes et éviter notamment d'exposer directement les entités et leurs propriétés sensibles.

### Qu'est-ce que BOLA / IDOR ?

C'est notamment une situation où un utilisateur peut accéder à une ressource en manipulant son identifiant sans que le serveur vérifie correctement l'autorisation.

### Pourquoi utiliser Rate Limiting ?

Pour limiter la consommation excessive de ressources et réduire certains abus, notamment le brute force.

### CORS sécurise-t-il l'API ?

Non.

CORS concerne principalement les restrictions imposées par les navigateurs pour les requêtes cross-origin. Un client non navigateur peut appeler directement l'API.

---

## À retenir

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

La règle la plus importante :

> **Le serveur ne doit jamais faire confiance au client.**

## Phrase à mémoriser

> **Sécuriser une API, c'est vérifier l'identité, les permissions, les données, les ressources consommées et les erreurs à chaque frontière importante.**
