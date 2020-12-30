# typescript-advanced-types

## 1.Generic class for API requests

Let's assume that we have next allowed endpoints:

```typescript
const enum Endpoints {
    /**
     * Have only GET and POST
     */
    users = '/api/users',
    /**
     * Have only POST and DELETE
     */     
    notes = '/api/notes',  
    /**
     * Have only GET
     */
    entitlements = '/api/entitlements'
}
```
You might have noticed, that I used const enum instead of enum. This technique will reduce you code output. 
Please, keep in mind, it works bad with babel. 
Let's assume that backend developer allowed You to make:

 - GET | POST requests for users
 - POST | DELETE requests for notes
 - GET requests for entitlements
 
Let's define interfaces of our allowed fetch methods:
```typescript
interface HandleUsers {
    get<T>(url: Endpoints.users): Promise<T>;
    post(url: Endpoints.users): Promise<Response>;
}

interface HandleNotes {
    post(url: Endpoints.notes): Promise<Response>;
    delete(url: Endpoints.notes): Promise<Response>;
}

interface HandleEntitlements {
    get<T>(url: Endpoints.entitlements): Promise<T>;
}
```

Now, we can define our main API class

```typescript
class Api {
    get = <T = void>(url: Endpoints): Promise<T> => fetch(url).then(response => response.json())
    post = (url: Endpoints) => fetch(url, { method: 'POST' })
    delete = (url: Endpoints) => fetch(url, { method: 'DELETE' })
}
```
For now, class Api does not have any constraints.
So let's define them:
```typescript
// Just helper
type RequiredGeneric<T> = T extends void
    ? { __TYPE__ERROR__: 'Please provide generic parameter' }
    : T

interface HandleHttp {
    <T extends void>(): RequiredGeneric<T>
    <T extends Endpoints.users>(): HandleUsers;
    <T extends Endpoints.notes>(): HandleNotes;
    <T extends Endpoints.entitlements>(): HandleEntitlements;
}
```
As You see, HandleHttp is just overloading for function. Nothing special except the first line. I will come back to it later.
We have class Api and overloadings for function. How we can combine them? Very simple - we will just create a function which returns instance of Api class
```typescript
const handleHttp: HandleHttp = <_ extends Endpoints>() => new Api();
```
