# Angular Dependency Injection

La Dependency Injection (DI) permet à Angular de créer et de fournir automatiquement les dépendances dont un component ou un service a besoin.

## Pourquoi utiliser la DI ?

Sans DI, un component devrait créer lui-même ses dépendances :

```typescript
const service = new HotelService();
```

Avec DI, le component déclare simplement ce dont il a besoin :

```typescript
constructor(private hotelService: HotelService) {}
```

Angular se charge de fournir l'instance.

## Mental model

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

Le component dépend d'une abstraction ou d'un service fourni par le système d'injection plutôt que de gérer lui-même sa création.

## Fournir un service

Une forme courante :

```typescript
@Injectable({
  providedIn: 'root'
})
export class HotelService {
}
```

`providedIn: 'root'` indique que le service est disponible au niveau racine de l'application.

## Injection avec inject()

Angular moderne permet également :

```typescript
private hotelService = inject(HotelService);
```

Mental model :

```text
Déclarer la dépendance
        ↓
Angular cherche un provider
        ↓
Angular crée ou récupère l'instance
        ↓
La dépendance est injectée
```

## DI et .NET

Le principe est similaire à celui d'ASP.NET Core.

En .NET :

```csharp
builder.Services.AddScoped<IHotelService, HotelService>();
```

Puis :

```csharp
public HotelController(IHotelService service)
{
    _service = service;
}
```

En Angular, le mécanisme et les scopes sont différents, mais l'idée générale reste :

```text
Je déclare ce dont j'ai besoin
        ↓
Le framework fournit la dépendance
```

## À retenir

> Dependency Injection = le component demande ses dépendances ; Angular se charge de les fournir.
