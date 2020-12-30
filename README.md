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
