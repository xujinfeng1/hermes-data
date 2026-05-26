# FastAPI RBAC Patterns

## The POST Body vs Query String Trap

When a FastAPI endpoint function has default parameters like `role: str = "admin"`,
FastAPI reads them from **query strings**, NOT from JSON POST bodies.

### Wrong (role always defaults to "admin")
```python
@app.post("/admin/keys")
async def create_key(role: str = "admin"):
    # role is always "admin" — POST body {"role": "viewer"} is ignored
    create_api_key(role=role)
```

### Right (read from JSON body)
```python
@app.post("/admin/keys")
async def create_key(request: Request):
    body = await request.json()
    role = body.get("role", "admin")
    create_api_key(role=role)
```

This affected RBAC in the CDP agent — all viewer keys were silently created
as admin because the role parameter never came from the JSON body.

## Dependency-based Permission Check

```python
def require_role(action: str):
    async def checker(auth=Depends(get_auth)):
        if not check_permission(auth["role"], action):
            raise HTTPException(403, f"角色 '{auth['role']}' 无权执行 '{action}'")
        return auth
    return checker

@app.post("/chat")
async def chat(req, auth=Depends(require_role("chat"))):
    ...
```

## Bootstrap Problem

When `/admin/keys` endpoint itself requires auth (role="manage"), there's no
way to create the first admin key. Solution: keep `/admin/keys` auth-free for
bootstrapping, lock other admin endpoints.

## validate_api_key Return Type

Return a dict `{"tenant": Tenant, "role": str}` rather than just a Tenant
object, so all downstream code can access both tenant info and role without
a second lookup.
