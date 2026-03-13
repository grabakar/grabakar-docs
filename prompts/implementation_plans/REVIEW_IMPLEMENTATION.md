# Review Agent — Implementation Plan

Exact checklist and procedures for code review. No judgment calls needed — follow each step mechanically.

---

## Pre-Review Setup

Before reviewing any PR or changeset:

1. Read the task reference: find the backlog ID (e.g., `P1-03`) in the branch name or PR title.
2. Open `grabakar-docs/planificacion/BACKLOG.md` and read the task's acceptance criteria.
3. Open `grabakar-docs/tecnico/API_CONTRACTS.md` if the change touches API endpoints.
4. Note the repository: `grabakar-backend` or `grabakar-frontend`.

---

## Checklist Execution

Run each check in order. Record result as OK / FALLA / N/A.

### 1. Acceptance Criteria

**Procedure:**
1. Read all acceptance criteria from the backlog task.
2. For each criterion, find the code that implements it.
3. If a criterion is not implemented, mark FALLA and list it.

**Commands:**
```bash
# Get list of changed files
git diff --name-only main...HEAD
```

### 2. Hardcoded Brand Names

**Procedure — Backend:**
```bash
cd grabakar-backend
grep -rn '"GrabaKar"' api/ --include="*.py"
grep -rn "'GrabaKar'" api/ --include="*.py"
grep -rni 'grabakar' api/ --include="*.py" | grep -v 'grabakar_test' | grep -v 'import'
```

**Procedure — Frontend:**
```bash
cd grabakar-frontend
grep -rn '"GrabaKar"' src/ --include="*.tsx" --include="*.ts" | grep -v 'ThemeContext' | grep -v 'DEFAULTS'
```

**Expected result:** Zero matches (exceptions: fixture data, ThemeContext defaults, test factories).

### 3. Language Check

**Procedure:**
```bash
# Find all user-visible strings
grep -rn "error\|message\|label\|placeholder\|tooltip" src/ --include="*.tsx" | head -30
# Or for backend:
grep -rn "error\|mensaje\|Error" api/ --include="*.py" | head -30
```

**Rule:** Every user-visible string must be in Spanish (Chile). Variable names and code stay in English.

**Common violations:**
- `"Invalid credentials"` → debe ser `"Credenciales inválidas"`
- `"Field required"` → debe ser `"Este campo es obligatorio"`
- `"Plate"` → debe ser `"Patente"`

### 4. Tests

**Procedure — Backend:**
```bash
pytest --cov=api --cov-report=term-missing -q
# Check coverage % on changed files
```

**Procedure — Frontend:**
```bash
npx vitest run --coverage
```

**Rule:** Every NEW function with business logic must have a test. Check:
- Services: `api/services/` has test in `api/tests/`
- Serializer validations: custom `validate_*` methods have tests
- Frontend services: `src/services/` has test in `__tests__/`
- Utils: `src/utils/` has test in `__tests__/`

**If new business logic exists without tests → FALLA.**

### 5. Scope of Changes

**Procedure:**
```bash
git diff --name-only main...HEAD
```

**Rule:** Every changed file must relate to the task. Common violations:
- Reformatting unrelated files
- Adding imports to modules not in scope
- Modifying test infrastructure (conftest) beyond what's needed

**Exception:** Minimal changes to shared imports, types, or base classes are acceptable.

### 6. Security (Backend Only)

**SQL injection:**
```bash
grep -rn '\.raw(' api/ --include="*.py"
grep -rn '\.extra(' api/ --include="*.py"
grep -rn 'cursor\.execute' api/ --include="*.py"
```
All matches → verify parameterized queries. Non-parameterized → FALLA.

**Secrets:**
```bash
grep -rn 'password\|secret\|api_key\|token' api/ --include="*.py" | grep -v 'os.getenv\|settings\.\|test\|factory\|mock'
```
Hardcoded secrets → FALLA.

**Multi-tenant:**
For every ViewSet and view in the changed files, verify it either:
- Inherits `TenantQuerySetMixin`, OR
- Explicitly filters by `request.user.tenant` in `get_queryset()`

```bash
# List all ViewSets without TenantQuerySetMixin
grep -rn 'class.*ViewSet' api/views/ --include="*.py" | while read line; do
  file=$(echo "$line" | cut -d: -f1)
  grep -q 'TenantQuerySetMixin' "$file" || echo "MISSING TENANT FILTER: $line"
done
```

**Permissions:**
Every endpoint must have `permission_classes`. Check:
```bash
grep -rn 'class.*View\|def.*view' api/views/ --include="*.py" -A5 | grep -v 'permission_classes' | head -20
```

### 7. API Contract Compliance

**Procedure (Backend):**
1. Open `tecnico/API_CONTRACTS.md`.
2. For each endpoint touched in the PR, compare response shape with contract.
3. Verify: field names, types, status codes, error format.

**Procedure (Frontend):**
1. Check that mock data in tests matches the API contract exactly.
2. Verify response parsing handles all documented fields.

### 8. Offline (Frontend Only)

For changes touching data flows:
1. Verify data saves to IndexedDB before any API call.
2. Verify `SyncManager` handles the new data type.
3. Verify UI shows feedback when offline (`ConnectivityBadge`, error messages).
4. Verify retry logic covers the new flow.

### 9. Multi-tenant (Backend Only)

For each queryset in changed code:
```python
# MUST have one of these patterns:
.filter(tenant=self.request.user.tenant)
# OR inherit TenantQuerySetMixin
```

Check in:
- List/detail querysets
- Create/update: verify no FK pointing to another tenant's data
- Celery tasks: if processing data, verify tenant filtering
- Report endpoints: verify CSV only contains tenant data

---

## Response Format

For each checklist item:

```
### 1. Acceptance Criteria — OK
All 5 criteria verified in code.

### 2. Brand Names — OK
grep found 0 matches.

### 3. Language — FALLA
- api/views/auth.py:45 — "Invalid credentials" should be "Credenciales inválidas"

...

## Verdict: RECHAZAR
Items to fix:
1. Language: api/views/auth.py:45
2. Tests: missing test for SyncService._crear_grabado
```

---

## Automated Pre-Check Script

Create and run this before starting manual review:

```bash
#!/bin/bash
# review_precheck.sh
echo "=== Brand Name Check ==="
grep -rn '"GrabaKar"' api/ src/ --include="*.py" --include="*.ts" --include="*.tsx" 2>/dev/null | grep -v test | grep -v fixture | grep -v DEFAULTS

echo "=== Raw SQL Check ==="
grep -rn '\.raw(' api/ --include="*.py" 2>/dev/null
grep -rn '\.extra(' api/ --include="*.py" 2>/dev/null

echo "=== Secrets Check ==="
grep -rn 'password.*=' api/ --include="*.py" 2>/dev/null | grep -v 'os.getenv\|settings\.\|test\|factory\|mock\|fixture'

echo "=== Tenant Mixin Check ==="
grep -rn 'class.*ViewSet' api/views/ --include="*.py" 2>/dev/null | while read line; do
  file=$(echo "$line" | cut -d: -f1)
  grep -q 'TenantQuerySetMixin' "$file" || echo "MISSING: $line"
done

echo "=== Pre-check done ==="
```

---

## Edge Cases to Watch

| Pattern | Why it's a problem |
|---------|--------------------|
| `Grabado.objects.all()` without tenant filter | Data leak between tenants |
| `serializer.save()` without `tenant=request.user.tenant` | Record created without tenant |
| `LeyCaso.objects.filter(activa=True)` without tenant check | May show laws from other tenants (if LeyCaso becomes tenant-specific) |
| `f"..."` with user input in queryset | SQL injection risk |
| Error messages in English | Violates Spanish UI requirement |
| Direct `localStorage.setItem` instead of Dexie | Bypasses encryption and structured storage |
