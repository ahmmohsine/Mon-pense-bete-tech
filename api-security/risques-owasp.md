# Risques Majeurs OWASP API Top 10 : Authentification & Autorisation

Ce document présente une analyse détaillée de trois des vulnérabilités les plus critiques affectant les APIs REST lorsque les mécanismes d'authentification et d'autorisation sont mal implémentés ou absents.

---

## 1. Fondamentaux : Authentification vs Autorisation

Une architecture API sécurisée repose sur la distinction stricte entre deux concepts fondamentaux :

* **Authentification (AuthN) :** Processus qui consiste à vérifier l'identité d'un utilisateur ou d'un système. *Question : "Qui êtes-vous ?"* (Exemple : Vérification d'un jeton JWT signé).
* **Autorisation (AuthZ) :** Processus qui consiste à vérifier si l'entité authentifiée dispose des permissions requises pour effectuer une action ou accéder à une ressource donnée. *Question : "Avez-vous le droit de faire cela ici ?"* (Exemple : Vérification des rôles ou de la propriété d'un enregistrement).

L'échec des contrôles d'autorisation constitue la cause principale des fuites de données massives dans les APIs modernes.

---

## 2. Broken Object Level Authorization (BOLA / IDOR)

### Description
BOLA (anciennement connu sous le nom d'IDOR - Insecure Direct Object References) survient lorsqu'un endpoint d'API expose un identifiant d'objet dans l'URL ou le corps de la requête sans vérifier si l'utilisateur authentifié possède les droits d'accès à cette ressource spécifique.

### Mécanisme d'Exploitation
1. Un utilisateur légitime (User A, ID = 101) s'authentifie et obtient un jeton JWT valide.
2. Il émet une requête légitime : `GET /api/orders/1001`.
3. L'utilisateur modifie l'identifiant dans la requête : `GET /api/orders/1002`.
4. Le serveur valide la signature du jeton JWT (Authentification OK), mais ne vérifie pas si l'ordre `1002` appartient au User A (Autorisation KO).
5. Le serveur retourne les données de la commande du User B.

### Impact
Fuite de données massive et exfiltration automatisée. Les attaquants utilisent des scripts simples pour incrémenter les identifiants séquentiels ou récolter des UUIDs afin de télécharger des bases de données entières.

### Règle de Remédiation
Le contrôle d'accès doit être exécuté au niveau de la couche d'accès aux données. L'identifiant de l'utilisateur extrait du jeton validé doit être systématiquement inclus dans la clause de recherche en base de données.

```csharp
// Exemple de remédiation en C# / Entity Framework Core
public async Task<OrderDto?> GetOrderByIdAsync(int orderId, int currentUserId)
{
    return await _dbContext.Orders
        .Where(o => o.Id == orderId && o.UserId == currentUserId) // Validation d'appartenance stricte
        .Select(o => new OrderDto 
        {
            Id = o.Id,
            TotalAmount = o.TotalAmount,
            CreatedAt = o.CreatedAt
        })
        .FirstOrDefaultAsync();
}