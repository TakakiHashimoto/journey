
### Let's say I have RLS :
```
checks auth.uid()
=> user A has the same uid, user A may select "item_secret"
```

### Situation:
User A logs in
=> Browser has 
Browser 
  ├── Supabase URL ← public 
  ├── anon/publishable key ← public 
  └── A's session/JWT

User A can open dev-tools and send something like this
```
supabase .from("plaid_item_secrets") .select("access_token");
```

**My Current RLS only sees if the requsting user is the same user id**

## SOLUTION:
```
Alice's browser
       │
       │ "Sync my Plaid Item item-555"
       ▼
Next.js Route Handler
       │
       │ authenticate Alice
       ▼
Alice = abc123
       │
       │ verify item-555 belongs to abc123
       ▼
AUTHORIZED
       │
       │ now cross trusted boundary
       ▼
server-only privileged DB access
       │
       ▼
plaid_item_secrets
       │
       ▼
access_token
       │
       ▼
Plaid API
       │
       ▼
transactions
       │
       ▼
Next.js
       │
       ▼
Alice's browser

{
    added: 12,
    modified: 2
}
```

## createClient()

```javascript
createServerClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!, 
	{ cookies: { getAll() { return cookieStore.getAll(); }, ... }, }, );
```

This one uses "publishable key + request cookies"
=> not server-role client
cookies include logged-in supabase session

**You are supposed to use server-specfic client for secret items**


You don't need RLS that says if the same user, they can operate select, delete or whatever.
They CANNOT operate at all.
=> That's why we are droping RLS and revoking the priviledges.
```
REVOKE ALL PRIVILEGES
ON public.plaid_item_secrets
FROM authenticated;
```

### We are aiming:
```
                 plaid_item_secrets

anon              no GRANT
                  no policy
                       ❌

authenticated     no GRANT
                  no policy
                       ❌

service_role      privileged server authority
                       ✅

postgres          administrative authority
                       ✅
```


Supabase server-role access bypasses RLS. that's why you don't need authenticated-user RLS for server-role access.

