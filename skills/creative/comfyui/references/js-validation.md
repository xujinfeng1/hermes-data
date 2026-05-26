# JavaScript Inline Validation

## Problem

Repeated patching of inline JS in HTML files leads to corrupted syntax:
- Orphaned `async` keywords from deleted functions
- Over-escaped quotes in template literals (`\\\\\\\\'` instead of `'`)
- Duplicate function declarations that silently override each other
- Missing function calls that leave UI buttons disabled

## Validation Pattern

After every patch to inline JS, validate syntax:

```python
import subprocess

with open('index.html') as f:
    html = f.read()
start = html.index('<script>')
end = html.index('</script>')
js = html[start+8:end]

r = subprocess.run(
    ['node', '-e', f'new Function({repr(js)})'],
    capture_output=True, text=True
)
if r.stderr:
    print('SYNTAX ERROR:', r.stderr[:200])
else:
    print('JS: OK')
```

## Common Corruptions

### Orphaned async keywords
After removing `async function optVideo(e){...}`, the `async` keyword may remain on its own line:
```
async              ← orphaned, causes SyntaxError
function toggleTheme(){...}
```
Fix: check for standalone `async` lines after function removal.

### Over-escaped template literals
When template literals contain HTML with `onclick` attributes, escaping spirals out of control:
```javascript
// ❌ Wrong — 4 backslashes before quote
onclick=\"...replace(/'/g,\\\"\\\\\\\\'\\\")...\"
// ✅ Correct — just escape for JS string
onclick=\"...replace(/'/g,'')...\"
```

### Duplicate functions
Two `function switchTab(...)` blocks — second silently overrides first. JS doesn't warn.
```bash
grep -c "function switchTab" index.html  # should be 1
```

### Missing init() calls
If `init()` doesn't enable buttons (no `onKeyChange()` or equivalent), all UI is frozen.
```javascript
async function init(){
  // ... setup ...
  document.getElementById('genBtn')?.removeAttribute('disabled');  // MUST have this
}
```
