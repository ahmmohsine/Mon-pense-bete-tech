# Angular HttpClient

`HttpClient` permet à Angular de communiquer avec des APIs HTTP, notamment une API ASP.NET Core.

## Mental model

```text
Angular
    |
    | HTTP Request
    v
ASP.NET Core API
    |
    v
Service
    |
    v
EF Core
    |
    v
Database
```

Puis la réponse revient :

```text
Database
    ↓
EF Core
    ↓
ASP.NET Core
    ↓ JSON
Angular
    ↓
Component
    ↓
Template
```

## Effectuer une requête GET

Exemple :

```typescript
this.http.get<Hotel[]>('/api/hotels');
```

La réponse est généralement un `Observable<Hotel[]>`.

## POST

```typescript
this.http.post<Hotel>(
  '/api/hotels',
  hotel
);
```

## PUT

```typescript
this.http.put<Hotel>(
  `/api/hotels/${hotel.id}`,
  hotel
);
```

## DELETE

```typescript
this.http.delete(
  `/api/hotels/${hotel.id}`
);
```

## Service HTTP

Il est généralement préférable de centraliser les appels API dans un service :

```typescript
@Injectable({
  providedIn: 'root'
})
export class HotelService {

  private http = inject(HttpClient);

  getHotels() {
    return this.http.get<Hotel[]>('/api/hotels');
  }
}
```

Le component peut ensuite utiliser le service :

```text
Component
    ↓
HotelService
    ↓
HttpClient
    ↓
ASP.NET Core API
```

## Interceptors

Les interceptors permettent notamment d'intervenir sur les requêtes ou réponses HTTP.

Ils peuvent être utilisés pour des besoins comme :

```text
HTTP Request
    ↓
Interceptor
    ↓
API
```

Par exemple, ajouter un token d'authentification à une requête.

## À retenir

> HttpClient = le mécanisme Angular utilisé pour communiquer avec une API HTTP.
