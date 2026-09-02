# VoltStack Authorization System — Testing, Verification and Security Assurance System

## 1. Propósito

Este documento define la arquitectura completa para **testing, verificación y aseguramiento de seguridad** del Authorization System de VoltStack.

El objetivo no es únicamente comprobar que:

```text
GRANT funciona
DENY funciona
```

sino demostrar que el sistema mantiene correctamente propiedades más profundas:

```text
Policies reciben los argumentos correctos

Gates se resuelven correctamente

RBAC no cruza scopes

ABAC no usa atributos no confiables

ReBAC termina correctamente

Tenant A nunca recibe acceso a Tenant B

Failures nunca se convierten en GRANT

Caches no reutilizan decisiones inseguras

Metadata compilada equivale a metadata dinámica

Workers persistentes no filtran estado

Cambios de configuración no degradan silenciosamente seguridad
```

La filosofía será:

```text
Authorization correctness
must be verified as a security property,
not merely as application behavior.
```

---

# 2. Objetivos

El sistema de testing deberá cubrir:

1. Policies;
2. Gates;
3. Voters/Evaluators;
4. Decision Strategies;
5. AuthorizationManager;
6. Planner;
7. Pipeline;
8. Attributes y metadata;
9. Controller/Route integration;
10. RBAC;
11. ABAC;
12. ReBAC;
13. Multi-tenancy;
14. Cache y memoization;
15. Failures y exceptions;
16. Audit y observability;
17. concurrency;
18. persistent workers;
19. performance;
20. fuzzing;
21. property-based testing;
22. mutation testing;
23. static verification;
24. security regression suites.

---

# 3. Principio arquitectónico

VoltStack deberá separar al menos:

```text
Unit Tests
Integration Tests
Contract Tests
Security Property Tests
Runtime Isolation Tests
Performance Tests
Static Verification
```

Ningún tipo de prueba sustituye completamente a los demás.

---

# 4. Pirámide de pruebas

La estrategia recomendada:

```text
          Security / E2E
              ▲
         Integration
              ▲
        Contract Tests
              ▲
           Unit
```

con pruebas de propiedades y fuzzing atravesando varias capas.

---

# 5. Unit Testing de Policies

Una Policy deberá poder probarse sin:

```text
HTTP Kernel
Routing
Controller
Database real
```

cuando sus dependencias puedan sustituirse por doubles.

Ejemplo:

```php
$policy = new InvoicePolicy(
    permissions: $permissions,
    relationships: $relationships,
);

$result = $policy->update(
    $user,
    $invoice,
    $context,
);
```

---

# 6. Qué probar en una Policy

Debe verificarse:

```text
GRANT esperado

DENY esperado

reasonCode

ABSTAIN cuando corresponda

dependencias relevantes

comportamiento ante estados límite
```

---

# 7. Ejemplo

```php
expect(
    $policy->update($user, $invoice, $context)
)->toEqual(
    DecisionResult::deny('invoice.locked')
);
```

---

# 8. Evitar testing por implementación

No es ideal comprobar:

```text
method X called exactly twice
```

si la semántica no depende de ello.

Preferir probar:

```text
decision result
reason code
security property
```

---

# 9. Contract Test de Policies

Además de unit tests, VoltStack podrá proporcionar un contrato genérico que valide:

```text
return type supported

method signature válida

no unexpected nulls

no invalid scalar returns

supported abilities
```

---

# 10. Policy Contract Suite

Conceptualmente:

```php
PolicyContractTest::for(InvoicePolicy::class)
    ->supports('view')
    ->supports('update')
    ->hasValidSignatures();
```

---

# 11. Unit Testing de Gates

Un Gate deberá probarse como callable/handler independiente.

Ejemplo:

```php
$result = $gate(
    $principal,
    $context
);
```

---

# 12. Gate Contract Tests

Debe verificarse:

```text
registration
ability mapping
return normalization
priority
optional/required semantics
```

---

# 13. Evaluator Tests

Todo evaluator reusable deberá probar:

```text
input supported
DecisionResult correcto
failure normalization
cacheability metadata
reason codes
```

---

# 14. PermissionEvaluator Test

Ejemplo:

```text
Principal has permission
→ GRANT

Principal lacks permission
→ DENY

Permission provider fails
→ FAILURE
```

---

# 15. RoleEvaluator Test

Matriz:

```text
Any + one matching role
→ GRANT

Any + none
→ DENY

All + all present
→ GRANT

All + one missing
→ DENY
```

---

# 16. Decision Strategy Tests

Cada strategy deberá tener una suite exhaustiva.

Ejemplo `DenyOverrides`:

```text
GRANT
→ GRANT

DENY
→ DENY

GRANT + DENY
→ DENY

ABSTAIN + GRANT
→ GRANT

all ABSTAIN
→ default policy
```

---

# 17. Table-Driven Tests

Las strategies son excelentes candidatas para:

```php
yield [
    [GRANT, GRANT],
    [DENY, DENY],
    [[GRANT, DENY], DENY],
];
```

---

# 18. Strategy Invariants

También deberán verificarse propiedades:

```text
Adding a DENY to DenyOverrides
must never turn DENY into GRANT.
```

---

# 19. Planner Unit Tests

El Planner deberá probarse sin ejecutar evaluators.

Debe verificar:

```text
resolution
filtering
deduplication
phase selection
priority
stable ordering
strategy selection
required evaluators
```

---

# 20. Example Planner Test

```php
$plan = $planner->plan($request);

expect($plan->evaluatorIds())->toBe([
    'tenant_isolation',
    'permission',
    'invoice_policy',
]);
```

---

# 21. Stable Ordering Test

Con misma metadata:

```text
same input
→ same plan order
```

siempre.

---

# 22. Deduplication Test

Una Policy descubierta por dos fuentes:

```text
explicit registration
automatic resolution
```

no debe ejecutarse dos veces cuando representan la misma identidad.

---

# 23. Required Evaluator Test

Si falta:

```text
TenantIsolationPolicy
```

para un Subject tenant-owned:

```text
planning failure
```

cuando strict tenant mode lo exige.

---

# 24. AuthorizationManager Tests

Deberá probarse:

```text
GRANT
DENY
FAILURE
ABSTAIN finalization
memoization
cache
fresh mode
nested authorization
```

---

# 25. API Contract

Verificar:

```text
can()
inspect()
authorize()
```

---

# 26. `can()` Tests

```text
GRANT → true

DENY → false

FAILURE → false + report
```

---

# 27. `inspect()` Tests

Debe preservar:

```text
Granted
Denied
Failed
```

sin perder metadata.

---

# 28. `authorize()` Tests

```text
GRANT → returns normally

DENY → AuthorizationDeniedException

FAILURE → AuthorizationFailureException
```

---

# 29. Metadata Tests

Attributes deberán probar:

```text
extraction
normalization
inheritance
composition
conflicts
subject references
strategy
phase
```

---

# 30. Example Attribute Test

```php
#[Authorize('update', subject: 'invoice')]
public function update(Invoice $invoice)
{
}
```

deberá compilar a:

```text
Ability=update
SubjectReference=Argument(invoice)
Phase=Resource
```

---

# 31. Invalid Subject Test

```php
#[Authorize('update', subject: 'invoice')]
public function update(Order $order)
{
}
```

deberá fallar durante compilación.

---

# 32. PublicAccess Conflict Test

```php
#[PublicAccess]
#[RequiresPermission('admin')]
```

deberá producir metadata conflict.

---

# 33. Inheritance Tests

Base Controller:

```text
admin.access
```

Child method:

```text
invoice.update
```

Effective metadata:

```text
admin.access
AND
invoice.update
```

---

# 34. Replace/Disable Tests

Deberán probar que:

```text
nonBypassable
```

no puede eliminarse.

---

# 35. Controller Integration Tests

La integración HTTP deberá comprobar lifecycle.

---

# 36. PreResolution DENY

Debe garantizar:

```text
Controller not invoked

Model binding not executed
```

cuando no es requerido.

---

# 37. Resource DENY

Debe garantizar:

```text
Resource bound

Policy executed

Controller not invoked
```

---

# 38. Successful Authorization

```text
PreResolution GRANT
Resource GRANT
→ Controller invoked once
```

---

# 39. Argument Resolution Optimization Test

Si solo:

```text
invoice
```

es requerido para autorización, otros argumentos costosos no deben resolverse antes de un DENY si la integración soporta lazy resolution.

---

# 40. Route DSL Equivalence

Esto:

```php
->authorize('update', 'invoice')
```

y:

```php
#[Authorize('update', subject: 'invoice')]
```

deben producir descriptors equivalentes.

---

# 41. Route Group Tests

Nested groups deberán componer requisitos de forma determinista.

---

# 42. Resource Convention Tests

`authorizeResource()` deberá mapear correctamente:

```text
index → viewAny
show → view
store → create
update → update
destroy → delete
```

cuando se use la convención.

---

# 43. HTTP Mapping Tests

Debe verificarse separación entre:

```text
401
403
404
500
503
```

---

# 44. 401 Test

```text
AnonymousPrincipal
+
authentication required
→ 401
```

---

# 45. 403 Test

```text
Authenticated Principal
+
valid DENY
→ 403
```

---

# 46. 404 Concealment Test

```text
tenant mismatch
→ 404
```

sin exponer ownership.

---

# 47. 500 Test

Policy bug:

```text
TypeError
→ failure
→ 500
```

---

# 48. 503 Test

External authorization provider unavailable:

```text
→ 503
```

si esa es la mapping policy.

---

# 49. RBAC Testing

Debe cubrir:

```text
direct Roles
direct Permissions
Role → Permission
scopes
inheritance
expiration
revocation
wildcards
provider composition
```

---

# 50. Scope Matrix

Ejemplo obligatorio:

| Principal | Tenant | Role | Expected |
|---|---|---|---|
| User#42 | 7 | admin@7 | GRANT |
| User#42 | 9 | admin@7 | DENY |
| User#42 | 9 | viewer@9 | según ability |

---

# 51. Direct Permission Test

Un direct grant deberá funcionar aunque no exista Role.

---

# 52. Revocation Test

Después de revocar:

```text
new authorization execution
→ must not see old grant
```

---

# 53. Expiration Test

Una Role assignment vencida:

```text
→ must not grant
```

---

# 54. Wildcard Tests

Debe verificarse exactamente qué cubre:

```text
invoice.*
```

y qué no cubre.

---

# 55. Circular Role Graph

Debe fallar la configuración.

---

# 56. Permission Provenance Test

Explainability puede comprobar:

```text
permission obtained via finance-manager
```

---

# 57. ABAC Testing

Debe utilizar matrices sobre:

```text
Principal attributes
Subject attributes
Context
Ability metadata
```

---

# 58. Example

Rule:

```text
principal.clearance >= subject.classification
```

Matriz:

| Clearance | Classification | Result |
|---:|---:|---|
| 3 | 2 | GRANT |
| 2 | 2 | GRANT |
| 1 | 2 | DENY |

---

# 59. Missing Attribute Test

Debe distinguir:

```text
attribute missing
```

de:

```text
attribute present but insufficient
```

---

# 60. Trusted Attribute Test

Un header client-controlled:

```text
X-MFA: true
```

nunca deberá satisfacer:

```text
security.mfa_verified
```

si no fue producido por el trusted SecurityContext.

---

# 61. Attribute Provider Failure

Debe producir:

```text
FAILURE
```

cuando el atributo es obligatorio y el provider falla.

---

# 62. ReBAC Testing

Debe probar:

```text
direct edge
indirect path
missing path
cycles
max depth
scope boundaries
batch resolution
```

---

# 63. Direct Relationship

```text
User#42 owner Project#81
→ match
```

---

# 64. Indirect Relationship

```text
User#42 member Organization#7
Organization#7 owns Project#81
→ path matched
```

---

# 65. Cycle

```text
A → B → C → A
```

debe terminar sin loop.

---

# 66. Graph Depth

Si path válido excede límite configurado:

```text
FAILED / graph limit
```

según contrato.

---

# 67. Relationship Isolation

Relación de Tenant#7 no debe resolver un objeto Tenant#9 salvo regla cross-tenant explícita.

---

# 68. Multi-Tenant Testing

Esta será una de las suites de seguridad más importantes.

---

# 69. Regla de matriz cruzada

Para todo recurso tenant-owned:

```text
Principal A
Tenant A
Resource A
→ normal authorization

Principal A
Tenant A
Resource B
→ never GRANT
```

---

# 70. Same-ID Isolation Test

Muy importante:

```text
Tenant#7 Invoice#100
Tenant#9 Invoice#100
```

El contexto Tenant#7 nunca debe resolver el recurso de Tenant#9.

---

# 71. Scoped Binding Test

La query/binding deberá incorporar Tenant scope.

---

# 72. Tenant Context Missing

Una operation tenant-required sin TenantContext:

```text
FAILURE
```

no acceso global.

---

# 73. Tenant Mismatch

Con TenantContext válido pero Subject de otro Tenant:

```text
DENY
```

---

# 74. Membership Revocation Test

Revocar membresía y ejecutar una nueva request:

```text
DENY
```

aunque session siga viva.

---

# 75. Tenant Suspension Test

Tenant suspendido:

```text
mutations denied
```

según TenantOperationalPolicy.

---

# 76. Read-Only Tenant Test

Debe distinguir:

```text
read
write
```

abilities.

---

# 77. Cross-Tenant Support Test

Debe exigir:

```text
global permission
support session
MFA
target tenant
```

según policy.

---

# 78. Actor Preservation

Durante impersonation deberá verificarse que:

```text
actor != effective principal
```

se conserva en audit y AuthorizationRequest.

---

# 79. Platform Admin Test

`platform.admin` no deberá otorgar automáticamente:

```text
tenant.invoice.approve
```

sin regla explícita.

---

# 80. Tenant Hierarchy Test

Parent Tenant no recibe acceso Child automáticamente.

---

# 81. Shared Resource Test

Debe distinguir:

```text
owner
shared viewer
shared editor
```

según ReBAC/Policy.

---

# 82. Query Isolation Tests

No limitarse a `can()`.

También verificar:

```text
list
count
search
aggregation
export
bulk operations
```

---

# 83. List Test

Tenant A listing no contiene rows Tenant B.

---

# 84. Count Test

`COUNT(*)` no debe incluir otro Tenant.

---

# 85. Search Test

Full-text search no devuelve hits cross-tenant.

---

# 86. Export Test

CSV/JSON no contiene datos externos.

---

# 87. Bulk Mutation Test

Update/delete de Tenant A no afecta Tenant B.

---

# 88. Cache Testing

Debe probar múltiples capas separadamente.

---

# 89. Plan Cache Test

Misma estructura:

```text
→ same plan
```

sin almacenar Principal/Subject runtime.

---

# 90. Request Memo Test

Misma decisión:

```text
same key
→ evaluator executes once
```

---

# 91. Different Tenant Memo Test

Mismo Principal + Ability + Subject ID pero Tenant distinto:

```text
must not hit same memo entry
```

---

# 92. Principal Version Test

Después de incrementarse:

```text
auth_version
```

un cache entry antiguo no debe reutilizarse.

---

# 93. Subject Version Test

Cambio de estado relevante:

```text
new subject fingerprint
→ fresh decision
```

---

# 94. Cache Corruption Test

Entry corrupta:

```text
discard
report
fresh evaluation
```

Nunca GRANT por corrupción.

---

# 95. Cache Backend Failure

Si cache es opcional:

```text
fresh evaluation
```

debe continuar.

---

# 96. Stale Grant Test

Un expired GRANT:

```text
must not be resurrected
```

por fallo externo.

---

# 97. Fresh Mode Test

```php
Authorization::fresh()
```

debe ignorar dynamic decision reuse pero seguir aprovechando metadata/plan cache.

---

# 98. Mutable Subject Memo Test

```text
authorize
mutate auth-relevant field
authorize again
```

debe exigir invalidación/freshness según contrato.

---

# 99. Failure Testing

Debe existir una suite explícita de fail-closed.

---

# 100. Failure Property

```text
Any unexpected authorization failure
must never produce GRANT.
```

---

# 101. Policy Exception Test

Policy lanza Throwable:

```text
→ PolicyExecutionException
→ FAILED
```

---

# 102. External Timeout Test

```text
→ FAILED
→ operation blocked
```

---

# 103. Missing Required Context Test

```text
→ FAILED
```

---

# 104. Invalid Return Type Test

Policy retorna:

```php
"false"
```

Debe producir:

```text
InvalidEvaluatorResultException
```

No GRANT por truthiness.

---

# 105. Required Audit Failure Test

Policy GRANT + required audit sink fail:

```text
final outcome FAILED
```

---

# 106. Optional Observability Failure Test

Metrics collector falla:

```text
decision unchanged
```

---

# 107. Denial vs Failure Test

Esta diferencia debe comprobarse constantemente.

Ejemplo:

```text
missing permission
→ DENIED

permission repository unavailable
→ FAILED
```

---

# 108. Audit Testing

Debe probar:

```text
GRANT audit policy
DENY audit policy
critical audit
redaction
delivery guarantees
```

---

# 109. Audit Minimalism Test

Un AuditRecord no debe contener:

```text
raw Request
token
password
full Subject payload
```

---

# 110. Redaction Tests

Puede utilizarse una lista de secretos conocidos y comprobar que no aparecen en:

```text
trace
audit
logs
explain output
```

---

# 111. Explainability Tests

Debe verificarse:

```text
reason code
decisive evaluator
safe formatting
audience redaction
```

---

# 112. Public Explanation Test

Internal:

```text
tenant.mismatch
```

EndUser:

```text
resource_not_found
```

---

# 113. Developer Explanation Test

Puede mostrar más detalle sin revelar secretos.

---

# 114. Trace Determinism

No depender de timestamps para estructura.

Debe poder afirmarse:

```text
evaluator order
decision path
short circuit
```

---

# 115. Sampling Test

Mandatory audit nunca debe perderse por trace sampling.

---

# 116. Observability Non-Interference Test

Misma AuthorizationRequest con:

```text
trace=off
```

y:

```text
trace=full
```

debe producir misma decisión.

---

# 117. Property-Based Testing

VoltStack deberá considerar property-based testing como herramienta de primera clase para autorización.

---

# 118. Por qué

Las combinaciones de:

```text
Principal
Tenant
Role
Permission
Subject
Relationship
Context
```

crecen exponencialmente.

Ejemplos manuales no cubren suficientemente el espacio.

---

# 119. Propiedad fundamental

```text
If no evaluator produces GRANT
under a strategy requiring positive authorization,
final result must not be GRANT.
```

---

# 120. Tenant Isolation Property

Para:

```text
Tenant A != Tenant B
```

y un Subject exclusivamente de B:

```text
normal Tenant A authorization
must never GRANT
```

---

# 121. Permission Revocation Property

Si un Grant es requisito y se revoca:

```text
GRANT must not remain
```

salvo una ruta alternativa explícita de autorización.

---

# 122. Monotonic Denial Property

Para `DenyOverrides`:

```text
Adding another DENY
must not convert result to GRANT.
```

---

# 123. NonBypassable Property

Agregar un `SuperAdminOverrideEvaluator` no deberá saltarse:

```text
nonBypassable DENY
```

---

# 124. ABAC Clearance Property

Para una regla monotónica:

```text
increasing required clearance
cannot increase access
```

---

# 125. Graph Depth Property

Todo graph finito debe:

```text
terminate
```

dentro del max depth/cycle detection policy.

---

# 126. Cache Equivalence Property

Para una entrada de cache válida:

```text
cached decision
=
fresh decision
```

bajo el mismo fingerprint/version.

---

# 127. Metadata Compilation Equivalence

```text
dynamic metadata pipeline
=
compiled metadata pipeline
```

para el mismo código/configuración.

---

# 128. Plan Cache Equivalence

```text
cold plan
=
cached plan
```

estructuralmente.

---

# 129. Serialization Roundtrip Property

Compiled metadata serializada y cargada:

```text
same semantics
```

---

# 130. Fuzz Testing

El sistema deberá fuzzear inputs estructurales relevantes.

---

# 131. Fuzz Ability Names

Ejemplos:

```text
empty
very long
unicode
control characters
unexpected separators
```

deben producir comportamiento seguro.

---

# 132. Fuzz Role/Permission Identifiers

Nunca provocar:

```text
parser confusion
wildcard escalation
cache key collision
```

---

# 133. Fuzz Subject References

Metadata inválida:

```text
unknown argument
malformed named subject
invalid route reference
```

debe fallar temprano.

---

# 134. Fuzz Cache Keys

Debe garantizar:

```text
different canonical inputs
do not accidentally collide
```

en la capa de composición previa al hash.

---

# 135. Fuzz ReBAC Graph

Generar:

```text
cycles
deep paths
self loops
duplicate edges
```

y verificar terminación.

---

# 136. Fuzz Tenant IDs

No deben escapar scopes ni producir queries ambiguas.

---

# 137. Fuzz Transport Inputs

Headers/query params que parecen representar Tenant, Roles o Permissions nunca deben convertirse en autoridad por sí solos.

---

# 138. Mutation Testing

Especialmente valioso para security code.

---

# 139. Ejemplos de mutaciones

Cambiar:

```php
=== 
```

por:

```php
!==
```

en tenant checks debería hacer fallar tests.

---

# 140. Eliminar Permission Check

Mutation suite deberá detectar pérdida de protección.

---

# 141. Invertir Decision

```text
DENY → GRANT
```

debe romper rápidamente la suite.

---

# 142. Remove NonBypassable

Debe detectarse mediante security regression tests.

---

# 143. Remove Cache Version

Una mutación que elimine Tenant/auth version de cache key debe ser detectada.

---

# 144. Mutation Score

Los módulos críticos deberían mantener umbrales altos.

No perseguir 100% ciegamente, pero sí cubrir paths de seguridad relevantes.

---

# 145. Static Verification

Parte de los errores deben detectarse sin ejecutar requests.

---

# 146. Compiler Checks

```text
unknown abilities
invalid Policy signatures
missing Subject references
phase mismatches
strategy conflicts
nonBypassable override
```

---

# 147. PHPStan/Psalm

VoltStack podrá proporcionar extensiones para detectar:

```text
invalid #[Authorize]
unknown Permission constants
incorrect Policy method types
```

---

# 148. IDE/LSP

Podrá advertir:

```text
route references subject parameter that does not exist
```

---

# 149. Policy Static Rules

Warnings para:

```php
catch (Throwable) {
    return false;
}
```

dentro de Policies.

---

# 150. Why

Ese pattern puede ocultar:

```text
FAILURE
```

como:

```text
DENY
```

---

# 151. Static Rule: Direct HTTP Response

Policy que retorna:

```text
Response
```

debe marcarse inválida.

---

# 152. Static Rule: Service Locator

Puede advertirse el uso de estado global como:

```text
currentUser()
currentTenant()
```

dentro de shared authorization services si rompe runtime isolation.

---

# 153. Architecture Tests

VoltStack deberá poder verificar dependencias.

Ejemplo:

```text
Authorization Core
must not depend on HTTP
```

---

# 154. Dependency Rule

```text
Quantum\Authorization\Core
```

no puede importar:

```text
Quantum\Http
Quantum\Routing
```

---

# 155. Transport Layer

Sí puede depender de Contracts del Core.

---

# 156. Policy Purity Architecture Check

Policies no deberían depender directamente de:

```text
Controller
Response
Request globals
```

salvo abstractions autorizadas.

---

# 157. Security Regression Tests

Cada vulnerabilidad corregida deberá generar un test permanente.

---

# 158. Ejemplo

Si se descubre:

```text
Tenant scope omitted in route binding
```

crear test que reproduzca exactamente el cross-tenant access.

---

# 159. CVE-like Internal Tracking

Podrá etiquetarse:

```text
AUTHZ-SEC-001
```

para rastrear regressions.

---

# 160. Golden Security Matrix

Proyectos grandes pueden mantener matrices declarativas.

Ejemplo:

```text
Role        Ability              Resource       Expected
admin       invoice.view         own tenant     GRANT
admin       invoice.view         other tenant   DENY
viewer      invoice.update       own tenant     DENY
```

---

# 161. Matrix Runner

VoltStack Testing podría ofrecer:

```php
AuthorizationMatrix::run([...]);
```

---

# 162. Benefit

Facilita revisión por:

```text
security
product
compliance
QA
```

---

# 163. Contract Testing de Providers

Todos los providers externos deberán pasar una suite común.

---

# 164. PermissionProvider Contract

Debe verificar:

```text
scope handling
revocation
failure propagation
determinism
```

---

# 165. RelationshipProvider Contract

```text
direct match
missing match
cycle safety
failure normalization
```

---

# 166. AttributeProvider Contract

```text
trusted source
missing attribute
failure
redaction metadata
```

---

# 167. External Evaluator Contract

```text
timeout
invalid payload
5xx
schema mismatch
authentication error
```

deben normalizarse correctamente.

---

# 168. Fake Providers

Testing package deberá incluir:

```text
FakePermissionProvider
FakeRoleProvider
FakeRelationshipProvider
FakeAttributeProvider
FakePolicyDispatcher
```

---

# 169. AuthorizationFake

Por ergonomía podrá existir:

```php
Authorization::fake();
```

---

# 170. Pero con cuidado

Un Fake no debe hacer que tests importantes pasen sin probar seguridad real.

---

# 171. Recommended Modes

```text
Fake All
Fake Specific Ability
Spy Only
Real Core + Fake Dependencies
```

---

# 172. Example

```php
Authorization::fake()
    ->grant('invoice.view')
    ->deny('invoice.delete');
```

útil para tests de UI/Controller no centrados en autorización.

---

# 173. Authorization Spy

Podrá verificar:

```text
ability checked
subject checked
number of checks
fresh mode used
```

---

# 174. Real Authorization Tests

Security suite deberá evitar fakes en paths críticos.

---

# 175. `actingAs`

Testing helper:

```php
actingAs($user)
```

debe construir correctamente PrincipalContext.

---

# 176. Tenant Helpers

```php
actingAsTenant($tenant);

actingAsTenantUser(
    $user,
    $tenant
);
```

---

# 177. Scoped Grant Helpers

```php
grantRole(
    $user,
    'admin',
    scope: $tenant
);
```

---

# 178. Permission Helpers

```php
grantPermission(
    $user,
    'invoice.update',
    scope: $tenant
);
```

---

# 179. Relationship Helpers

```php
relate(
    user($user),
    'member',
    organization($organization)
);
```

---

# 180. Assertions

Podrán existir:

```php
assertAuthorized(...);
assertDenied(...);
assertAuthorizationFailed(...);
assertAuthorizationReason(...);
```

---

# 181. Tenant Assertions

```php
assertTenantAuthorized(...);
assertTenantDenied(...);
assertTenantNotFound(...);
```

---

# 182. Route Assertions

```php
assertRouteRequiresAbility(
    'invoice.update',
    'update'
);
```

---

# 183. Controller Assertions

```php
assertControllerRequires(
    InvoiceController::class,
    'update',
    'invoice.update'
);
```

---

# 184. Policy Assertions

```php
assertPolicyMapped(
    Invoice::class,
    InvoicePolicy::class
);
```

---

# 185. Plan Assertions

```php
assertAuthorizationPlanContains(
    'tenant_isolation'
);
```

---

# 186. Explain Assertions

```php
assertAuthorizationDeniedBecause(
    'invoice.locked'
);
```

---

# 187. Audit Assertions

```php
assertAuthorizationAudited(
    ability: 'invoice.approve',
    decision: 'grant',
);
```

---

# 188. No-Audit Assertions

```php
assertAuthorizationNotAudited(...);
```

para low-risk checks.

---

# 189. Cache Assertions

```php
assertAuthorizationMemoHit();
assertAuthorizationFreshEvaluation();
```

---

# 190. Isolation Assertions

```php
assertNoTenantContextLeak();
```

---

# 191. Persistent Runtime Verification

Crítico bajo FrankenPHP.

---

# 192. Worker Reuse Test

Simular:

```text
Request A:
User A / Tenant A

Request B:
User B / Tenant B
```

sobre el mismo container/worker.

---

# 193. Must Verify

No leakage de:

```text
Principal
Tenant
Subject
AuthorizationSession
memo cache
trace
audit buffer
failure context
```

---

# 194. Exception Reuse Test

Request A termina con exception.

Request B debe seguir limpio.

---

# 195. 1000-Iteration Test

Alternar múltiples Principals/Tenants en el mismo worker.

---

# 196. Memory Growth Test

Authorization request-scoped caches no deben crecer indefinidamente entre requests.

---

# 197. Static Service State Test

Podrá inspeccionarse que servicios shared no retienen:

```text
currentPrincipal
currentTenant
lastDecision
```

---

# 198. Queue Worker Isolation

Misma idea para Jobs.

---

# 199. Job A Failure

Después de exception, Job B debe iniciar con Context limpio.

---

# 200. CLI Long-Running Process

Commands daemonizados deberán limpiar context entre unidades de trabajo.

---

# 201. Concurrency Testing

VoltStack deberá verificar thread/fiber/concurrent task safety cuando el runtime lo permita.

---

# 202. Concurrent Requests

Dos requests simultáneas:

```text
Tenant A
Tenant B
```

no deben compartir request-scoped state.

---

# 203. Fiber-Local / Context-Local State

Si runtime utiliza fibers:

```text
TenantContext
AuthorizationSession
```

deberán resolverse correctamente por execution context.

---

# 204. Race in Grant Changes

Test:

```text
authorization check
concurrent revoke
new authorization
```

debe respetar consistency mode definido.

---

# 205. Cache Stampede Test

Muchas requests concurrentes sobre misma expensive decision no deben producir unsafe stale behavior.

---

# 206. Race in Relationship Update

Remove relation y ejecutar nuevas requests.

El resultado deberá ajustarse al consistency contract.

---

# 207. Performance Testing

Authorization es security-sensitive, pero también hot path.

---

# 208. Benchmarks

Deberán medir por separado:

```text
metadata lookup
plan cache hit
cold plan build
simple Gate
Policy invocation
permission memo hit
permission repository lookup
ReBAC direct
ReBAC path
final decision memo hit
```

---

# 209. Benchmark Targets

Los objetivos exactos se definirán tras profiling.

No hardcodear números prematuramente.

---

# 210. Performance Regression

CI podrá comparar:

```text
baseline
current branch
```

para detectar degradaciones importantes.

---

# 211. Zero Reflection Verification

Production benchmark/test deberá comprobar que hot path no invoca:

```text
ReflectionClass
ReflectionMethod
Attribute discovery
```

cuando metadata compilada está activa.

---

# 212. Zero Filesystem Verification

Igualmente no debe hacer discovery durante authorization runtime.

---

# 213. Query Count Assertions

Ejemplo:

```text
500 invoice checks
```

no deben generar:

```text
500 permission queries
```

si memoization funciona.

---

# 214. ReBAC Query Count

Batch/listados deberán evitar graph N+1.

---

# 215. Memory Budget

Profiler test puede comprobar que:

```text
AuthorizationSession
```

se destruye al final.

---

# 216. Benchmark Security Equivalence

Las rutas optimizadas deben producir exactamente la misma decisión que rutas no optimizadas.

---

# 217. Cold vs Warm Equivalence

```text
cold plan
warm plan
→ same decision
```

---

# 218. Cache On vs Off Equivalence

Bajo mismas versiones:

```text
cache enabled
cache disabled
→ same result
```

---

# 219. Trace On vs Off Equivalence

Ya definido:

```text
same result
```

---

# 220. Compiled vs Dynamic Equivalence

Crítico para producción.

---

# 221. Differential Testing

Ejecutar misma batería contra:

```text
dynamic resolver implementation
compiled resolver implementation
```

y comparar.

---

# 222. Reference Implementation

Puede mantenerse una implementación simple no optimizada para testing differential.

---

# 223. Benefit

Permite optimizar el hot path sin perder semántica.

---

# 224. Formal-ish Security Properties

Aunque VoltStack no use formal verification completa, deberá documentar propiedades explícitas.

---

# 225. Property: Default Deny

Si no existe decisión positiva suficiente:

```text
must not GRANT
```

---

# 226. Property: Fail Closed

Todo failure relevante:

```text
must not GRANT
```

---

# 227. Property: Tenant Isolation

Tenant mismatch:

```text
must not GRANT
```

sin excepción por Roles comunes.

---

# 228. Property: NonBypassable

Un evaluator marcado `NonBypassable`:

```text
cannot be overridden by ordinary GRANT
```

---

# 229. Property: Request Isolation

State de Request A:

```text
must not influence Request B
```

---

# 230. Property: Cache Freshness

Un cached result:

```text
may be reused only while fingerprints remain valid
```

---

# 231. Property: Subject Type Safety

Una Ability con Subject type definido:

```text
cannot silently authorize another type
```

---

# 232. Property: Trusted Context

Client-controlled values:

```text
cannot become security attributes without trusted validation
```

---

# 233. Property: Explainability Non-Authority

Reason codes/explanations:

```text
cannot influence DecisionManager
```

---

# 234. Property: Audit Non-Authority

Excepto `Required audit` operational policy:

```text
audit observers cannot alter decision semantics
```

---

# 235. Security Assurance Levels

VoltStack podrá clasificar módulos/apps.

Ejemplo:

```php
enum AuthorizationAssuranceLevel: string
{
    case Standard = 'standard';
    case Elevated = 'elevated';
    case Critical = 'critical';
}
```

---

# 236. Standard

Requiere:

```text
unit
integration
basic tenant tests
failure tests
```

---

# 237. Elevated

Añade:

```text
property tests
mutation testing
cache validation
runtime isolation
```

---

# 238. Critical

Añade:

```text
full tenant matrices
fuzzing
strong consistency tests
audit guarantees
performance + security differential tests
manual security review
```

---

# 239. Ability Risk Mapping

Critical abilities pueden exigir un assurance level superior.

---

# 240. Example

```text
system.deploy
permission.grant
tenant.support
financial.transfer
```

---

# 241. Security Review Checklist

Antes de release de un nuevo authorization evaluator:

```text
What can it GRANT?

What can it DENY?

What happens if it fails?

Is it tenant-aware?

Is it cacheable?

Can its input be client-controlled?

Does it reveal sensitive reasons?

Can it be bypassed?

Is it safe under persistent workers?
```

---

# 242. Policy Review Checklist

```text
Does Policy return DENY for expected domain conditions?

Does it throw on technical failure?

Does it avoid catching Throwable into false?

Does it avoid HTTP dependencies?

Does it avoid global mutable context?

Does it have tests for negative cases?
```

---

# 243. Negative Tests Required

Una Policy que solo tiene GRANT tests está incompleta.

---

# 244. Minimum Matrix per Ability

Debe incluir:

```text
authorized
unauthorized
missing dependency/failure
wrong tenant if tenant-aware
```

---

# 245. Destructive Ability

Añadir:

```text
revoked permission
locked resource
critical context failure
```

---

# 246. CI Pipeline

Un pipeline ideal:

```text
Lint
    ↓
Static Analysis
    ↓
Unit Tests
    ↓
Contract Tests
    ↓
Integration Tests
    ↓
Security Property Tests
    ↓
Mutation Tests
    ↓
Performance Regression
```

No todos deberán ejecutarse en cada commit si son costosos.

---

# 247. Fast CI

Cada PR:

```text
lint
static analysis
unit
integration
core security properties
```

---

# 248. Full Security CI

Nightly/pre-release:

```text
mutation
fuzz
large property runs
persistent worker stress
performance
```

---

# 249. Security Baseline

Resultados deberán conservarse para detectar regresiones.

---

# 250. Flaky Tests

Security tests no deben aceptarse como flaky de forma permanente.

---

# 251. Deterministic Clock

Policies time-based deberán usar:

```text
ClockInterface
```

para tests reproducibles.

---

# 252. Deterministic IDs

Tests no deberían depender de UUIDs aleatorios cuando dificulten reproducibilidad.

---

# 253. Randomized Property Seeds

Cuando falle un property/fuzz test:

```text
seed
```

debe registrarse.

---

# 254. Reproducibility

El comando debe poder reejecutar:

```text
same seed
```

---

# 255. Test Fixtures

Authorization fixtures deberán ser mínimas y explícitas.

---

# 256. Avoid Hidden Global Fixtures

No crear:

```text
admin has every permission globally
```

en fixtures base si puede ocultar errores.

---

# 257. Principle of Least Privilege in Tests

Crear Principals con únicamente los grants requeridos para cada escenario.

---

# 258. Why

Tests con super-admin para todo no validan authorization real.

---

# 259. Factory Support

Podrán existir:

```php
UserFactory::new()
    ->withRole('finance-manager', $tenant)
    ->create();
```

---

# 260. Authorization Scenario Builder

Podrá existir:

```php
AuthorizationScenario::new()
    ->principal($user)
    ->tenant($tenant)
    ->ability('invoice.approve')
    ->subject($invoice)
    ->expectDenied('invoice.locked');
```

---

# 261. Useful for Large Matrices

Sí, especialmente en tests declarativos.

---

# 262. Snapshot Testing

Puede usarse para:

```text
compiled plans
effective metadata
explanation trees
```

---

# 263. Snapshot Warning

No debe sustituir assertions semánticas.

Un snapshot puede cambiar y ser aceptado sin revisar un security regression.

---

# 264. Recommended

Snapshot + explicit security assertions.

---

# 265. Plan Snapshot

Puede mostrar:

```text
TenantIsolation
Permission
InvoicePolicy
```

y test explícito:

```text
TenantIsolation must exist
```

---

# 266. Audit Snapshot

Evitar snapshot de timestamps/IDs inestables.

---

# 267. Test Isolation

Cada test deberá limpiar:

```text
registries
fake providers
AuthorizationSession
TenantContext
cache
```

---

# 268. Container Reset

En tests de persistent runtime, no resetear todo el container entre requests para poder detectar leaks reales.

---

# 269. But Standard Unit Tests

Sí pueden usar container limpio por test.

---

# 270. Registry Sealing Tests

Después de bootstrap:

```text
register new Policy
```

deberá fallar si registry está sealed.

---

# 271. Hot Reload Development Tests

En development mode, rebuild controlado debe funcionar sin dejar metadata parcial.

---

# 272. Configuration Reload Atomicity

No debe existir ventana donde:

```text
half old
half new
```

authorization metadata se mezcle.

---

# 273. Rolling Deployment Tests

Dos versions de registry compartiendo Redis:

```text
must not reuse each other's decision cache
```

---

# 274. Schema Version Test

Cached entries con schema viejo:

```text
cache miss
```

---

# 275. Backward Compatibility Tests

Si el package promete compatibilidad, verificar deserialización de metadata soportada.

---

# 276. Plugin/Extension Testing

Custom evaluator packages deberán usar contract test kit.

---

# 277. AuthorizationExtensionTestKit

Podrá proporcionar:

```text
EvaluatorContract
MetadataAdapterContract
ProviderContract
CacheabilityContract
FailureContract
```

---

# 278. Extension Must Declare

```text
type
priority
failure mode
cache policy
context requirements
```

---

# 279. Missing Declaration

Strict extension mode deberá fallar.

---

# 280. Security of Custom Strategies

Una Strategy custom deberá pasar properties básicas.

---

# 281. Example Strategy Contract

```text
must produce valid final Decision
must not treat invalid vote as GRANT
must handle all-abstain
must respect nonBypassable restrictions
```

---

# 282. Decision Strategy Fuzzing

Enviar combinaciones aleatorias de:

```text
GRANT
DENY
ABSTAIN
```

y validar invariants.

---

# 283. Failure Injection

VoltStack Testing deberá facilitar inyectar fallos.

---

# 284. Examples

```text
next permission lookup throws

relationship provider timeout

cache corrupt

audit sink unavailable

Policy throws
```

---

# 285. Chaos-Style Authorization Testing

En sistemas críticos puede probarse:

```text
random dependency failures
```

y comprobar:

```text
no operation gets accidental GRANT
```

---

# 286. Fail-Closed Chaos Property

Aunque múltiples dependencies fallen:

```text
final enforcement never GRANT
```

salvo que exista una ruta autorizada que no dependa de ellas y el plan lo permita explícitamente.

---

# 287. External Service Degradation

Debe probar:

```text
timeout
rate limit
malformed response
network unavailable
```

---

# 288. Circuit Breaker Open

Authorization deberá seguir clasificando el outcome como failure.

---

# 289. Security Headers / Transport

Authorization tests no necesitan cubrir todo HTTP security, pero sí que reason mapping no filtre datos.

---

# 290. Information Leakage Tests

Ejemplo:

```text
Tenant mismatch response
```

no debe incluir:

```text
actual tenant ID
policy class
database ID
```

---

# 291. Timing Tests

No pueden probar absence absoluta de timing side channels, pero pueden detectar diferencias grotescas.

---

# 292. Concealment Performance

`404` para cross-tenant no debería devolver payload claramente distinto al `404` normal.

---

# 293. Security Assurance Report

El tooling podrá generar un reporte.

---

# 294. Example

```text
Authorization Security Assurance

Policies tested: 42/42
Abilities with negative tests: 98%
Tenant-aware subjects covered: 100%
Cross-tenant matrices: passed
Fail-closed suite: passed
Cache isolation: passed
Persistent worker isolation: passed
Mutation score: 91%
```

---

# 295. Command

Futuro:

```text
volt authorization:verify
```

---

# 296. Modes

```text
--fast
--full
--security
--tenant
--cache
--runtime
```

---

# 297. `authorization:verify --security`

Podrá ejecutar:

```text
security properties
nonBypassable checks
fail-closed checks
metadata conflicts
tenant isolation assertions
```

---

# 298. `authorization:verify --runtime`

Ejecuta:

```text
FrankenPHP worker reuse
queue worker reuse
context cleanup
```

---

# 299. `authorization:verify --cache`

Ejecuta:

```text
fingerprints
version invalidation
cross-tenant cache isolation
corruption recovery
```

---

# 300. `authorization:coverage`

Podrá mostrar qué abilities poseen tests registrados.

---

# 301. Security Coverage != Code Coverage

Un 100% de líneas no significa 100% de security assurance.

---

# 302. Ability Coverage

Más útil:

```text
Ability
Grant path tested?
Deny path tested?
Failure path tested?
Tenant mismatch tested?
```

---

# 303. Example

| Ability | Grant | Deny | Failure | Tenant |
|---|---:|---:|---:|---:|
| invoice.view | ✓ | ✓ | ✓ | ✓ |
| invoice.approve | ✓ | ✓ | ✓ | ✓ |
| system.deploy | ✓ | ✓ | ✓ | N/A |

---

# 304. Policy Coverage

También:

```text
each evaluator path
```

---

# 305. Reason Code Coverage

Puede detectar reason codes nunca probados.

---

# 306. Critical Reason Codes

Especialmente:

```text
tenant.mismatch
security.mfa_required
rbac.permission_missing
authorization.failure.*
```

---

# 307. Test Tagging

Podrán usarse grupos:

```text
authz
authz-security
authz-tenant
authz-cache
authz-runtime
authz-slow
```

---

# 308. Parallel Test Safety

Los tests deberán aislar registries/config para ejecutarse en paralelo.

---

# 309. Shared Redis in Tests

Usar namespaces únicos por test worker.

---

# 310. Database Isolation in Tests

Tenant fixtures no deben colisionar entre workers.

---

# 311. Time-Based Test Stability

Freeze clock.

---

# 312. Randomness

Inject `RandomInterface` si una authorization rule realmente depende de randomness, aunque esto debería ser raro.

---

# 313. No Network by Default

Unit/integration tests normales deberían fakear external PDP.

---

# 314. Contract Environment

Sí deberá existir una suite opcional contra servicios reales.

---

# 315. External Integration CI

Puede ejecutarse en:

```text
pre-release
scheduled
```

no necesariamente cada PR.

---

# 316. Security Test Failure Policy

Un fallo en:

```text
tenant isolation
fail closed
nonBypassable
```

debe bloquear release.

---

# 317. Performance Test Failure

Puede usar thresholds/budgets diferenciados.

---

# 318. Manual Review

Algunas modificaciones deben exigir revisión adicional.

Ejemplos:

```text
DecisionStrategy
TenantIsolationPolicy
AuthorizationManager
cache key builder
failure normalization
```

---

# 319. CODEOWNERS

El repositorio podrá requerir reviewers de seguridad/core para estas áreas.

---

# 320. Change Risk Classification

Cambios authorization pueden etiquetarse:

```text
low
medium
high
critical
```

---

# 321. Critical Changes

Ejemplos:

```text
default strategy
fail closed policy
tenant isolation
cache reuse rules
public reason mapping
```

---

# 322. Release Verification

Antes de release:

```text
compile authorization metadata
run lint
run security verify
run persistent worker isolation
run critical ability matrices
```

---

# 323. Upgrade Tests

Cambiar versión del framework no deberá:

```text
silently broaden access
```

---

# 324. Behavioral Compatibility

Si cambia semántica deliberadamente, debe documentarse como breaking security behavior.

---

# 325. Golden Plan Comparison

Puede compararse el conjunto de authorization plans entre releases.

---

# 326. Unexpected Plan Diff

Ejemplo:

```text
TenantIsolation removed
```

debe marcarse crítico.

---

# 327. Authorization Plan Diff Tool

Futuro:

```text
volt authorization:diff
```

---

# 328. Example

```text
InvoiceController::update

- TenantIsolationPolicy
  REMOVED

+ NewCompliancePolicy
  ADDED
```

---

# 329. Security Review Gate

Eliminar un evaluator crítico deberá requerir aprobación explícita.

---

# 330. Production Verification

Opcionalmente, health checks pueden comprobar:

```text
authorization metadata loaded
registries sealed
cache schema current
```

---

# 331. No Live Dangerous Self-Test

No ejecutar acciones destructivas reales para verificar authorization.

---

# 332. Structural Health

Sí puede comprobar:

```text
required Policies registered
required strategies present
```

---

# 333. Runtime Health

Podrá probar una synthetic non-destructive ability.

---

# 334. Security Monitoring Feedback

Incidentes reales deberán alimentar regression tests.

---

# 335. Closed Loop

```text
Incident
   ↓
Root Cause
   ↓
Security Test
   ↓
Permanent Regression Guard
```

---

# 336. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Testing/
        ├── Assertions/
        │   ├── AuthorizationAssertions.php
        │   ├── TenantAuthorizationAssertions.php
        │   ├── PolicyAssertions.php
        │   ├── PlanAssertions.php
        │   └── AuditAssertions.php
        │
        ├── Fakes/
        │   ├── AuthorizationFake.php
        │   ├── FakePermissionProvider.php
        │   ├── FakeRoleProvider.php
        │   ├── FakeRelationshipProvider.php
        │   ├── FakeAttributeProvider.php
        │   └── FakeAuditSink.php
        │
        ├── Spies/
        │   ├── AuthorizationSpy.php
        │   ├── PolicyInvocationSpy.php
        │   └── EvaluatorSpy.php
        │
        ├── Scenarios/
        │   ├── AuthorizationScenario.php
        │   ├── AuthorizationMatrix.php
        │   └── AuthorizationFixtureBuilder.php
        │
        ├── Contracts/
        │   ├── PolicyContractTest.php
        │   ├── EvaluatorContractTest.php
        │   ├── StrategyContractTest.php
        │   ├── PermissionProviderContractTest.php
        │   ├── RelationshipProviderContractTest.php
        │   ├── AttributeProviderContractTest.php
        │   └── AuthorizationExtensionTestKit.php
        │
        ├── Properties/
        │   ├── DefaultDenyProperty.php
        │   ├── FailClosedProperty.php
        │   ├── TenantIsolationProperty.php
        │   ├── NonBypassableProperty.php
        │   ├── CacheEquivalenceProperty.php
        │   └── RuntimeIsolationProperty.php
        │
        ├── Fuzz/
        │   ├── AuthorizationIdentifierFuzzer.php
        │   ├── SubjectReferenceFuzzer.php
        │   ├── RelationshipGraphFuzzer.php
        │   └── CacheKeyFuzzer.php
        │
        ├── Runtime/
        │   ├── PersistentWorkerAuthorizationTest.php
        │   ├── QueueWorkerAuthorizationTest.php
        │   ├── ConcurrentAuthorizationTest.php
        │   └── AuthorizationContextLeakTest.php
        │
        ├── Security/
        │   ├── AuthorizationSecurityMatrix.php
        │   ├── TenantIsolationSecuritySuite.php
        │   ├── FailClosedSecuritySuite.php
        │   ├── CacheSecuritySuite.php
        │   └── InformationLeakageSecuritySuite.php
        │
        ├── Performance/
        │   ├── AuthorizationBenchmark.php
        │   ├── AuthorizationQueryCountAssertion.php
        │   └── AuthorizationPerformanceBaseline.php
        │
        └── Verification/
            ├── AuthorizationVerifier.php
            ├── AuthorizationVerificationReport.php
            ├── AuthorizationCoverageReport.php
            └── AuthorizationPlanDiff.php
```

---

# 337. Unit Testing Invariants

### Invariante 1

Policies pueden probarse sin HTTP.

### Invariante 2

Expected domain denial se prueba como DENY.

### Invariante 3

Technical failure se prueba como FAILURE.

### Invariante 4

Reason codes forman parte de assertions relevantes.

---

# 338. Planner Invariants

### Invariante 1

Plan order es determinista.

### Invariante 2

Required evaluators no desaparecen.

### Invariante 3

Dynamic y compiled planning son equivalentes.

### Invariante 4

Plan cache no modifica semántica.

---

# 339. Tenant Security Invariants

### Invariante 1

Cross-tenant normal access nunca produce GRANT.

### Invariante 2

Tenant Roles no cruzan scopes.

### Invariante 3

Same-ID resources permanecen aislados.

### Invariante 4

Queries, counts, exports y bulk operations respetan Tenant scope.

### Invariante 5

Worker reuse no filtra TenantContext.

---

# 340. Cache Invariants

### Invariante 1

Cache ON y OFF producen la misma decisión bajo inputs equivalentes.

### Invariante 2

Version change invalida reuse.

### Invariante 3

Cache corrupto no concede acceso.

### Invariante 4

Tenant forma parte de fingerprints necesarios.

### Invariante 5

Fresh mode obtiene decisión dinámica nueva.

---

# 341. Failure Invariants

### Invariante 1

Failures nunca producen GRANT.

### Invariante 2

DENY y FAILURE permanecen distinguibles.

### Invariante 3

External failure bloquea según fail-closed.

### Invariante 4

Observability opcional no altera outcome.

### Invariante 5

Required audit failure puede bloquear operación.

---

# 342. Runtime Invariants

### Invariante 1

AuthorizationSession no cruza unidades de trabajo.

### Invariante 2

Exceptions no impiden cleanup.

### Invariante 3

Shared services no retienen Principal/Tenant mutable.

### Invariante 4

Concurrent executions conservan context independiente.

---

# 343. Security Assurance Invariants

### Invariante 1

Toda ability crítica posee path GRANT y DENY probado.

### Invariante 2

Abilities tenant-aware incluyen cross-tenant tests.

### Invariante 3

Correcciones de seguridad generan regression tests.

### Invariante 4

Cambios críticos requieren verificación reforzada.

### Invariante 5

Code coverage no se interpreta como security coverage.

---

# 344. Arquitectura final de verificación

```text
                    Authorization Source
                           │
                           ↓
                    Static Verification
                           │
                           ↓
                      Compilation
                           │
                           ↓
                      Unit Tests
                           │
                           ↓
                    Contract Tests
                           │
                           ↓
                  Integration Testing
                           │
                           ↓
              Security Property Verification
                           │
          ┌────────────────┼────────────────┐
          ↓                ↓                ↓
      RBAC/ABAC         Multi-Tenant     Cache/Failure
         ReBAC            Isolation        Assurance
          │                │                │
          └────────────────┼────────────────┘
                           ↓
                  Persistent Runtime Tests
                           │
                           ↓
                Mutation / Fuzz / Chaos Tests
                           │
                           ↓
                  Performance Verification
                           │
                           ↓
                  Security Assurance Report
```

---

# 345. Ejemplo — Invoice Approval Security Matrix

Supongamos:

```text
Ability:
invoice.approve
```

Requisitos:

```text
Tenant match
finance-manager Role
invoice.approve Permission
Organization membership
InvoicePolicy
Approval limit
```

La suite deberá incluir al menos:

| Scenario | Expected |
|---|---|
| Todo válido | GRANT |
| Permission ausente | DENY |
| Role ausente | DENY |
| Tenant incorrecto | DENY / concealed |
| Membership organizacional ausente | DENY |
| Invoice bloqueado | DENY |
| Approval limit excedido | DENY |
| Permission provider caído | FAILURE |
| Relationship provider timeout | FAILURE |
| Audit requerido caído | FAILURE |
| Cache corrupto | fresh evaluation |
| Reused worker con otro Tenant | no leakage |

---

# 346. Ejemplo — Fail-Closed Property

Pseudo test:

```php
foreach ($failureInjectors as $failure) {
    $failure->enable();

    $outcome = $authorization->inspect(
        'invoice.approve',
        $invoice
    );

    expect($outcome->isGranted())
        ->toBeFalse();
}
```

---

# 347. Ejemplo — Tenant Property Test

Generar múltiples tenants:

```text
Tenant 1..N
```

con resources propios.

Propiedad:

```php
for every principalTenant != resourceTenant:
    assert authorization != GRANT
```

---

# 348. Ejemplo — Persistent Worker

```text
Request 1
User#1
Tenant#1
invoice.view
→ GRANT

Request 2
User#2
Tenant#2
same worker
same service instances
```

La suite deberá demostrar que Request 2 no hereda:

```text
User#1
Tenant#1
memoized permission
trace
decision
```

---

# 349. Ejemplo — Decision Strategy Property

Para `DenyOverrides`:

```php
$votes = randomVotes();

$result = $strategy->decide($votes);

if (containsDeny($votes)) {
    expect($result)->toBeDenied();
}
```

salvo condiciones explícitas de evaluadores no participantes.

---

# 350. Ejemplo — Compiled Equivalence

```text
Source Attributes
        ↓
Dynamic Metadata Resolver
        ↓
Plan A

Compiled Metadata Cache
        ↓
Compiled Resolver
        ↓
Plan B
```

Test:

```text
Plan A == Plan B
```

y después:

```text
Decision A == Decision B
```

---

# 351. Filosofía del sistema

La filosofía definitiva será:

```text
Test individual rules.

Verify subsystem contracts.

Prove cross-cutting security properties.

Attack tenant boundaries deliberately.

Inject infrastructure failures.

Compare optimized and reference implementations.

Reuse workers to expose state leaks.

Mutate security checks to prove tests can detect their removal.

Fuzz structural inputs.

Measure performance without sacrificing semantics.

Turn every discovered security defect into a permanent regression test.
```

---

# 352. Resultado esperado

El `Authorization Testing, Verification and Security Assurance System` permitirá que VoltStack no dependa únicamente de tests funcionales como:

```text
"admin can open page"
```

sino que pueda demostrar propiedades mucho más fuertes:

```text
cross-tenant access cannot silently succeed

revoked grants cannot survive valid version changes

technical failures cannot become grants

non-bypassable evaluators remain enforced

compiled optimization preserves semantics

persistent workers do not leak authorization state

public explanations do not reveal internal security details
```

El modelo final será:

```text
Authorization Implementation
        ↓
Unit Verification
        ↓
Contract Verification
        ↓
Integration Verification
        ↓
Property-Based Security Verification
        ↓
Tenant / Cache / Failure Assurance
        ↓
Persistent Runtime Verification
        ↓
Mutation + Fuzz Testing
        ↓
Security Assurance Report
```

El principio definitivo será:

```text
Authorization is not secure because the happy path works.

It is secure when invalid states, wrong tenants,
revoked privileges, dependency failures, stale caches,
malformed metadata and persistent-runtime edge cases
all fail in predictable and verifiably safe ways.
```

Con este subsistema, VoltStack podrá tratar la autorización como una **propiedad verificable de seguridad del framework**, no simplemente como un conjunto de condicionales cubiertos por tests unitarios.