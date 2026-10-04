# Mesures de Base : DTOs Stricts & Rate Limiting Centralisé

Ce document détaille l'approche Security by Design ainsi que deux mesures de sécurité fondamentales pour neutraliser les vulnérabilités majeures au niveau de la couche API : la séparation stricte des contrats DTO et la limitation de débit (Rate Limiting).

---

## 1. Pourquoi la Sécurité Doit Être Fondatrice (Security by Design)

Traiter la sécurité en fin de cycle de développement est une anti-pattern coûteuse et inefficace. Les vulnérabilités d'autorisation et de conception de données ne sont pas de simples bugs de code, mais des défauts structurels.

### Le Principe du Shift-Left et l'Impact Économique

Le coût de correction d'une faille augmente de manière exponentielle au fil du cycle de vie du logiciel :

* **Phase de Conception :** Modification d'un schéma, d'un DTO ou d'une règle de gestion (Coût faible).
* **Phase de Développement :** Refactorisation locale des contrôleurs et des services (Coût modéré).
* **Phase de Test / QA :** Reprise des tests d'intégration et modification des contrats d'interface (Coût élevé).
* **Phase de Production :** Correctif d'urgence, interruption de service, migration de données complexes, violation de conformité (GDPR) et atteinte à la réputation (Coût critique).

### Spécificités de la Surface d'Attaque des APIs REST

1. **Absence d'Interface Utilisateur (UI) comme Filtre :** Les attaquants ne passent pas par les formulaires de l'application web ou mobile. Ils interrogent directement les endpoints HTTP à l'aide d'outils comme Curl, Postman ou des scripts automatisés. Aucune validation côté client ne constitue une protection.
2. **Architecture Zero Trust :** Dans un environnement de microservices, le réseau interne ne doit jamais être considéré comme sûr. Chaque endpoint d'API doit authentifier, autoriser et valider de manière autonome chaque requête entrante.

---

## 2. Mesure 1 : Contrats DTO Stricts et Validation aux Frontières

### Problématique

Consommer ou retourner directement des entités de base de données (ORM / Entity Framework) au niveau des contrôleurs API expose directement l'application à deux risques majeurs :

* **Mass Assignment (BOPLA) :** L'injecteur modifie des propriétés non prévues (ex: `IsAdmin`, `TenantId`) car le framework lie automatiquement le JSON entrant aux propriétés de l'entité.
* **Excessive Data Exposure :** L'API retourne des objets complets contenant des champs internes sensibles (ex: `PasswordHash`, `SecurityStamp`, métadonnées internes).

### Solution Technique

1. **Isolation de la Couche de Domaine :** Les entités de base de données restent strictement confinées à la couche de persistance.
2. **DTOs Dédiés (Request / Response) :** Chaque opération possède son propre modèle d'entrée et son propre modèle de sortie.
3. **Validation Déclarative :** Validation des données aux frontières du système avant toute exécution de la logique métier.

### Exemple d'Implémentation C# / .NET Core

```csharp
using System.ComponentModel.DataAnnotations;
using Microsoft.AspNetCore.Authorization;
using Microsoft.AspNetCore.Mvc;

namespace Application.Controllers;

// 1. Contrat d'entrée strict (Request DTO)
public record UpdateUserProfileRequest(
    [Required, EmailAddress, MaxLength(256)] string Email,
    [Required, MaxLength(100)] string DisplayName
);

// 2. Contrat de sortie strict (Response DTO)
public record UserProfileResponse(
    Guid Id,
    string Email,
    string DisplayName
);

// 3. Contrôleur sécurisé
[ApiController]
[Route("api/users")]
public class UserController : ControllerBase
{
    private readonly IUserService _userService;

    public UserController(IUserService userService)
    {
        _userService = userService;
    }

    [HttpPut("profile")]
    [Authorize]
    public async Task<ActionResult<UserProfileResponse>> UpdateProfile(
        [FromBody] UpdateUserProfileRequest request,
        CancellationToken cancellationToken)
    {
        // La validation DataAnnotations est automatiquement vérifiée par le framework.
        // Extraction sécurisée de l'identité à partir du jeton vérifié.
        var userId = Guid.Parse(User.FindFirst("sub")?.Value 
            ?? throw new UnauthorizedAccessException());

        var updatedUser = await _userService.UpdateProfileAsync(userId, request, cancellationToken);

        // Mappage explicite vers le DTO de réponse pour éviter la fuite de données internes.
        return Ok(new UserProfileResponse(
            updatedUser.Id, 
            updatedUser.Email, 
            updatedUser.DisplayName
        ));
    }
}