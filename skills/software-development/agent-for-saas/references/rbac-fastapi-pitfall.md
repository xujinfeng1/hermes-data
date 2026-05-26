# RBAC FastAPI Pitfall — Full Debug Trace

## Symptom
Viewer role not being enforced — all users get admin access regardless of role specified at key creation.

## Root Cause
FastAPI function parameter `role: str="admin"` reads from QUERY STRING parameters, NOT from the JSON request body. When the client sends `POST /admin/keys` with body `{"role":"viewer"}`, FastAPI's `role` parameter resolves to the default `"admin"` because there's no query string `?role=viewer`.

## Fix
Read from request body explicitly:

```python
# BEFORE (broken — role always defaults to "admin")
@app.post("/admin/keys")
async def admin_create_key(role: str = "admin"):
    create_api_key(role=role)

# AFTER (fixed — reads from JSON body)
@app.post("/admin/keys")
async def admin_create_key(request: Request):
    body = await request.json()
    create_api_key(role=body.get("role", "admin"))
```

## Verification
Check role with health/detailed endpoint:
```bash
curl -s http://localhost:8800/health/detailed \
  -H "Authorization: Bearer $VIEWER_KEY" \
  | python3 -c "import sys,json; print(json.load(sys.stdin).get('role'))"
# Should print "viewer", not "admin"
```

## Related
This applies to ANY FastAPI POST endpoint that accepts non-path parameters — `tenant_id`, `name`, `custom_key`, etc. all suffer the same issue when defined as function parameters with defaults. Use `request.json()` for all POST endpoints that need body params.
