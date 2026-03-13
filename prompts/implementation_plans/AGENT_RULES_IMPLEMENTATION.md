# Agent Rules — Implementation Plan

Exact procedures that every agent must follow, independent of role (backend, frontend, review). This plan operationalizes `AGENT_RULES.md`.

---

## PDCA Cycle — Step-by-Step

### CHECK (Before Any Work)

1. **Read the task** from `planificacion/BACKLOG.md`. Find the task ID (e.g., `P1-03`). Read:
   - Description
   - Acceptance criteria (all of them)
   - Dependencies (verify they are complete)
   - Referenced docs

2. **Read the feature spec** from `features/`. The task description tells you which one.

3. **Read the API contract** (`tecnico/API_CONTRACTS.md`) if the task touches any endpoint.

4. **Read existing code** in the affected files:
   ```bash
   # Backend
   @mantic "relevant query"
   # Or
   find api/ -name "*.py" | head -20
   
   # Frontend
   @mantic "relevant query"
   # Or
   find src/ -name "*.ts" -o -name "*.tsx" | head -20
   ```

5. **Run current state:**
   ```bash
   # Backend
   ruff check .
   pytest -q
   
   # Frontend
   npm run lint
   npm test
   ```

### ACT (Diagnose)

1. Identify what's missing vs. the acceptance criteria.
2. Identify what's broken (failing tests, lint errors).
3. List blocking issues (missing dependencies, unclear spec).
4. If blocked: document the blocker, stop. Do not guess.

### PLAN (Before Writing Code)

1. List the files you will create or modify. Max: the files in scope of the task.
2. For each file, list what changes you'll make.
3. Verify the plan doesn't touch files outside the task scope.
4. If the plan requires changes outside scope, document as a dependency.

### DO (Implement)

1. Implement exactly what was planned. No scope creep.
2. After implementation:
   ```bash
   # Backend
   ruff check .
   pytest --cov=api -q
   
   # Frontend
   npm run lint
   npm test
   ```
3. Fix any failures introduced by your changes.
4. Commit with conventional commit in Spanish:
   ```bash
   git add -A
   git commit -m "feat(auth): implementar login con JWT"
   ```

---

## Git Workflow

### Branch Creation

```bash
git checkout main
git pull
git checkout -b feature/<fase>-<nombre-corto>
# Examples:
# feature/f0-backend-skeleton
# feature/f1-login-auth
# feature/f1-grabado-crud
```

### Commit Convention

Format: `<type>(<scope>): <description in Spanish>`

| Type | Use |
|------|-----|
| `feat` | New functionality |
| `fix` | Bug fix |
| `test` | Adding/fixing tests |
| `refactor` | Code change without behavior change |
| `docs` | Documentation only |
| `chore` | Build, CI, config changes |

Examples:
```
feat(auth): implementar login con JWT
fix(sync): corregir conflicto de merge en cola offline
test(grabado): agregar tests para validación de patente
refactor(api): simplificar serializer de vidrios
```

### Pre-Push Checklist

```bash
# Backend
ruff check .                                    # 0 errors
pytest --cov=api --cov-fail-under=80 -q        # All pass, >80% coverage
python manage.py makemigrations --check --dry-run  # No unapplied migrations
grep -rn '"GrabaKar"' api/ --include="*.py" | grep -v fixture  # 0 matches

# Frontend
npm run lint                                    # 0 errors
npx vitest run --coverage                       # All pass
npx tsc --noEmit                                # No type errors
npm run build                                   # Builds successfully
grep -rn '"GrabaKar"' src/ --include="*.ts" --include="*.tsx" | grep -v DEFAULTS  # 0 matches
```

---

## Absolute Rules — Verification Commands

### 1. No Hardcoded Brand Names

```bash
# Run in repo root
grep -rn '"GrabaKar"\|"Grabakar"\|"grabakar"' . --include="*.py" --include="*.ts" --include="*.tsx" | grep -v node_modules | grep -v test | grep -v fixture | grep -v DEFAULTS | grep -v __pycache__
```

**Expected:** 0 matches. If any found, replace with `tenant.nombre` (backend) or `useAppConfig().appName` (frontend).

### 2. All User Text in Spanish

Every string displayed to the user must be in Spanish. Verify:

```bash
# Backend error messages
grep -rn "error\|Error\|message" api/ --include="*.py" | grep -v '#' | grep -v test
# Check each match is in Spanish
```

Common required translations:
- "Invalid" → "Inválido/a"
- "Required" → "Obligatorio"
- "Not found" → "No encontrado"
- "Unauthorized" → "No autorizado"
- "Forbidden" → "No tienes permiso"

### 3. Tests for Business Logic

For every function in:
- Backend: `api/services/`, `api/serializers/` (custom validators)
- Frontend: `src/services/`, `src/utils/`

There must be a corresponding test. Verify:

```bash
# Backend
for f in $(find api/services -name "*.py" -not -name "__init__*"); do
  base=$(basename "$f" .py)
  test_file="api/tests/test_${base}.py"
  [ -f "$test_file" ] || echo "MISSING TEST: $test_file for $f"
done

# Frontend
for f in $(find src/services -name "*.ts" -not -name "*.test.*" -not -name "*.d.ts"); do
  dir=$(dirname "$f")
  base=$(basename "$f" .ts)
  test_file="${dir}/__tests__/${base}.test.ts"
  [ -f "$test_file" ] || echo "MISSING TEST: $test_file for $f"
done
```

### 4. Files in Scope

Before committing, verify:
```bash
git diff --name-only main...HEAD
```
Every file must relate to the current task. If you modified a file outside scope, either:
- Revert it: `git checkout main -- <file>`
- Document it as a dependency in the task

### 5. Multi-Tenant (Backend)

Every DB query must filter by tenant. Verify all ViewSets:
```bash
grep -rn 'class.*ViewSet\|class.*View(' api/views/ --include="*.py" | while read line; do
  file=$(echo "$line" | cut -d: -f1)
  class=$(echo "$line" | grep -oP 'class \K\w+')
  grep -q 'TenantQuerySetMixin\|tenant.*request.user.tenant' "$file" || echo "NO TENANT FILTER: $class in $file"
done
```

---

## Documentation Rules

1. **Never create documentation** unless explicitly requested.
2. Documentation lives in `grabakar-docs/`, not in code repos.
3. Code comments only for:
   - Non-obvious intent
   - Trade-offs and constraints
   - "why", not "what"
4. If a spec needs updating (you find a discrepancy), document it as a follow-up task. Do not modify the spec mid-implementation.

---

## Tool Usage

| Need | Tool | Fallback |
|------|------|----------|
| Find code | `@mantic "query"` | `grep -rn "pattern" .` |
| External docs | `@exa "query"` | Web search |
| File navigation | Read tool | `cat` / `less` |
| Text replacement | StrReplace tool | `sed` |

---

## Parallel Work Protocol

When multiple agents work simultaneously:

1. **Never modify files outside your task scope.** If two agents touch the same file, conflicts arise.
2. **Use feature branches.** Each agent works on `feature/<fase>-<task>`.
3. **API contracts are the interface.** Frontend mocks the API; backend implements it. Both reference `API_CONTRACTS.md`.
4. **Merge order matters.** Base infrastructure (P0-*) merges first. Backend models (P1-01) before auth (P1-02). Frontend IndexedDB (P1-06) before auth service (P1-07).
5. **If your task depends on another task that isn't merged yet,** mock the dependency and note it. Don't block.

### Parallelization Map

```
ROUND 1 (0 deps, 3 agents): P0-01 + P0-02 + P0-03
ROUND 2 (after P0):          P0-04 + P0-05 + P1-01 + P1-06 + P1-13
ROUND 3 (after models):      P1-02 + P1-07
ROUND 4 (after auth):        P1-03 + P1-04 + P1-05 + P1-08 + P1-09
ROUND 5 (after pages):       P1-10 + P1-11
ROUND 6 (after form):        P1-12
```

---

## Error Handling Standards

### Backend

All errors return this format:
```json
{
  "error": "Mensaje legible en español",
  "code": "ERROR_CODE",
  "detalles": {}
}
```

Standard codes: `VALIDATION_ERROR`, `INVALID_CREDENTIALS`, `TOKEN_EXPIRED`, `INVALID_REFRESH_TOKEN`, `PERMISSION_DENIED`, `NOT_FOUND`, `DUPLICATE_UUID`, `RATE_LIMITED`, `SERVER_ERROR`.

### Frontend

- Network errors: show offline indicator + queue for sync.
- Validation errors: show inline per field on `onBlur`.
- Auth errors: trigger token refresh or redirect to login.
- Storage errors (`QuotaExceededError`): show "Almacenamiento lleno" message.
