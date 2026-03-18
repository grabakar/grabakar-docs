# QA Checklist — Pre-Release

Execute this checklist before every release. All P0 items must PASS. P1 items: ≥ 95% pass rate.

---

## Environment

| Field | Value |
|-------|-------|
| **Build** | _version / commit_ |
| **Date** | _YYYY-MM-DD_ |
| **Tester** | _name or "QA Agent"_ |
| **Backend** | _local / staging / prod_ |
| **Emulator/Device** | _AVD name or device model_ |

---

## 1. Login & Session (P0)

- [ ] **TC-LOGIN-001**: Login exitoso con credenciales válidas → Home
- [ ] **TC-LOGIN-013**: Credenciales inválidas → error genérico
- [ ] **TC-LOGIN-020**: Login sin conexión → mensaje claro
- [ ] **TC-LOGIN-021**: Auto-login con offline token → Home sin login
- [ ] **TC-LOGIN-022**: Offline token expirado → login screen con mensaje
- [ ] **TC-LOGIN-030**: Access token auto-refresh silencioso
- [ ] **TC-LOGIN-040**: Logout elimina tokens, preserva registros pendientes
- [ ] **TC-LOGIN-076**: Botón login deshabilitado sin datos

---

## 2. Creación de Grabado (P0)

- [ ] **TC-GRAB-001**: Grabado completo → guardado en IndexedDB
- [ ] **TC-GRAB-004**: Grabado offline → sin errores
- [ ] **TC-GRAB-010**: Patente normalizada a mayúsculas
- [ ] **TC-GRAB-013**: Patente < 6 chars → error
- [ ] **TC-GRAB-016**: Confirmación no coincide → error
- [ ] **TC-GRAB-030**: Duplicado detectado → alerta con fecha
- [ ] **TC-GRAB-040**: Post-guardado: patente read-only
- [ ] **TC-GRAB-041**: Botón cambia a "Continuar a Grabado de Vidrios"

---

## 3. Flujo Multi-Vidrio (P0)

- [ ] **TC-MV-001**: Auto: 6 vidrios, progreso, imprimir, finalizar
- [ ] **TC-MV-002**: Moto: 1 vidrio, botón "Finalizar" directo
- [ ] **TC-MV-010**: Imprimir incrementa contador
- [ ] **TC-MV-020**: Omitir con confirmación
- [ ] **TC-MV-041**: "Finalizar" solo en último vidrio
- [ ] **TC-MV-050**: Patente grande (≥ 48px, monospace, bold)
- [ ] **TC-MV-051**: Patente correcta en todos los vidrios

---

## 4. Sincronización (P0)

- [ ] **TC-SYNC-001**: Sync automática al recuperar conexión
- [ ] **TC-SYNC-002**: Sync post-guardado online
- [ ] **TC-SYNC-010**: Indicador online/offline visible
- [ ] **TC-SYNC-020**: Backoff exponencial funciona
- [ ] **TC-SYNC-022**: Errores parciales no detienen batch completo
- [ ] **TC-SYNC-040**: Conflicto: dispositivo gana si más reciente
- [ ] **TC-SYNC-051**: Registros no sincronizados NUNCA se purgan

---

## 5. Seguridad (P0)

- [ ] **TC-SEC-001**: Operador: solo acciones de operador
- [ ] **TC-SEC-002**: Supervisor: acceso ampliado correcto
- [ ] **TC-SEC-010**: Tenant aislado (no ve datos de otro tenant)
- [ ] **TC-SEC-020**: Sin auth → 401
- [ ] **TC-SEC-030**: Patente no editable post-guardado
- [ ] **TC-SEC-050**: SQL injection no funciona
- [ ] **TC-SEC-061**: Logout limpia todos los tokens

---

## 6. Reportes (P0)

- [ ] **TC-REP-001**: Reporte diario contiene datos correctos
- [ ] **TC-REP-010**: Operador NO puede acceder reportes (403)
- [ ] **TC-REP-011**: Supervisor PUEDE descargar reportes
- [ ] **TC-REP-013**: Reportes aislados por tenant

---

## 7. White-Label (P0)

- [ ] **TC-WL-001**: Tenant branding aplicado tras login
- [ ] **TC-WL-002**: Branding default sin config
- [ ] **TC-WL-003**: Branding persiste offline
- [ ] **TC-WL-010**: Zero strings "GrabaKar" hardcodeadas en componentes

---

## 8. Performance (P1)

- [ ] **TC-PERF-001**: Arranque en frío < 4 segundos
- [ ] **TC-PERF-020**: Sync 100 registros < 30 segundos
- [ ] **TC-PERF-022**: Sync no bloquea UI

---

## 9. Usability (P1)

- [ ] **TC-UX-011**: Errores en español y accionables
- [ ] **TC-UX-020**: Indicador conectividad siempre visible
- [ ] **TC-UX-030**: Doble digitación de patente funciona
- [ ] **TC-UX-031**: Patente grande en multi-vidrio
- [ ] **TC-UX-050**: Confirmación visual al guardar

---

## 10. Anti-Fraud (P0)

- [ ] Patente no editable post-guardado
- [ ] Cada impresión incrementa contador auditable
- [ ] No se puede cambiar patente y reimprimir en otro vidrio

---

## Sign-off

| Role | Name | Date | Status |
|------|------|------|--------|
| QA Lead | | | ☐ Approved / ☐ Rejected |
| Dev Lead | | | ☐ Approved / ☐ Rejected |
| Product Owner | | | ☐ Approved / ☐ Rejected |

### Release Decision

- [ ] **All P0 tests PASS** (mandatory)
- [ ] **≥ 95% of P1 tests PASS** (mandatory)
- [ ] **No open P0/P1 GitHub issues** (mandatory)
- [ ] **All S1 issues resolved** (mandatory)
- [ ] **Smoke suite passes on target build** (mandatory)

**Decision**: ☐ GO / ☐ NO-GO

**Notes**: _any release notes or known issues accepted for this release_
