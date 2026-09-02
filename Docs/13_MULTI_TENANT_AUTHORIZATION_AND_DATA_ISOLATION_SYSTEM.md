# VoltStack Authorization System — Multi-Tenant Authorization and Data Isolation System

## 1. Propósito

Este documento define la arquitectura de **autorización multi-tenant y aislamiento de datos** del Authorization System de VoltStack.

El objetivo es garantizar que una aplicación pueda ejecutar múltiples tenants dentro de la misma infraestructura sin permitir que:

```text
Principal Tenant A
        ↓
acceda, modifique, descubra o infiera
        ↓
Resources Tenant B
```

salvo cuando exista una capacidad cross-tenant explícita, controlada y auditable.

El sistema deberá integrar:

```text
Tenant Context
Tenant Membership
Tenant-scoped Roles
Tenant-scoped Permissions
Tenant-aware Policies
Resource Ownership
Scoped Route Binding
Database Isolation
Cross-Tenant Administration
System Principals
Global Principals
Background Jobs
Queues
CLI
Cache
Events
Observability
Audit
FrankenPHP persistent workers
```

dentro del mismo Authorization Core definido en documentos anteriores.

La regla fundamental será:

```text
Multi-tenancy is not merely an authorization rule.

It is a security boundary.
```

---

# 2. Principio arquitectónico

VoltStack deberá aplicar aislamiento mediante múltiples capas:

```text
Tenant Resolution
      ↓
Tenant Context
      ↓
Data Access Scoping
      ↓
Resource Resolution
      ↓
Authorization
      ↓
Domain Operation
```

No deberá depender exclusivamente de:

```php
if ($user->tenant_id !== $resource->tenant_id) {
    return false;
}
```

dentro de Policies.

La Policy constituye una defensa importante, pero no debe ser la única barrera.

---

# 3. Defense in Depth

La arquitectura recomendada será:

```text
HTTP / Runtime
      ↓
Tenant Context
      ↓
Routing / Binding Isolation
      ↓
Database Query Isolation
      ↓
Authorization Tenant Boundary
      ↓
Resource Policy
      ↓
Persistence Isolation
```

Cada capa deberá reducir la posibilidad de cross-tenant access.

---

# 4. Objetivos

El sistema deberá proporcionar:

- representación canónica de Tenant;
- Tenant Context request-scoped;
- resolución segura del Tenant;
- Tenant Membership;
- tenant-scoped Roles;
- tenant-scoped Permissions;
- tenant-aware Policies;
- aislamiento de Subjects;
- aislamiento durante route binding;
- aislamiento de queries;
- integración con repositories;
- protección de operaciones de escritura;
- cross-tenant access explícito;
- global/system Principals;
- impersonation segura;
- jobs tenant-aware;
- queues tenant-aware;
- CLI tenant-aware;
- cache isolation;
- event isolation;
- authorization audit;
- protección bajo FrankenPHP;
- detección de context leakage;
- soporte para diferentes modelos físicos de tenancy.

---

# 5. Tenant

Un Tenant representa un dominio lógico de aislamiento.

Ejemplos:

```text
Company
Organization
Customer Account
Workspace
Store
Business Unit
SaaS Account
```

VoltStack no deberá asumir que Tenant significa necesariamente:

```text
Company
```

La aplicación define su semántica.

---

# 6. TenantIdentifier

Authorization no necesita necesariamente la entidad completa.

Podrá trabajar con:

```php
final readonly class TenantIdentifier
{
    public function __construct(
        public string|int $value,
    ) {}
}
```

---

# 7. TenantReference

Para contextos donde se requiera tipo:

```php
final readonly class TenantReference
{
    public function __construct(
        public string $type,
        public string|int $identifier,
    ) {}
}
```

Ejemplo:

```text
tenant:7
workspace:81
organization:15
```

---

# 8. TenantInterface

Podrá existir:

```php
interface TenantInterface
{
    public function tenantIdentifier(): TenantIdentifier;
}
```

pero el Authorization Core no deberá exigir que todos los modelos multi-tenant implementen una entidad concreta del framework.

---

# 9. Tenant Context

El runtime deberá representar el Tenant activo mediante:

```text
TenantContext
```

---

# 10. TenantContext

Conceptualmente:

```php
final readonly class TenantContext
{
    public function __construct(
        public ?TenantReference $tenant,
        public TenantContextMode $mode,
    ) {}
}
```

---

# 11. TenantContextMode

Podrá distinguir:

```php
enum TenantContextMode: string
{
    case Tenant = 'tenant';
    case Global = 'global';
    case System = 'system';
    case None = 'none';
}
```

---

# 12. Tenant Mode

```text
Tenant
```

significa:

```text
request/operation is constrained
to one explicit tenant
```

---

# 13. Global Mode

`Global` representa una operación autorizada para trabajar sobre más de un Tenant.

No significa:

```text
skip tenant security
```

---

# 14. System Mode

`System` estará reservado para operaciones internas del framework o plataforma.

Ejemplos:

```text
scheduled infrastructure operation
tenant provisioning
global maintenance
internal migration
```

---

# 15. None Mode

Representa:

```text
no tenant has been established
```

No deberá confundirse con:

```text
Global
```

Esta diferencia es crítica.

---

# 16. Regla fundamental

```text
Missing Tenant Context
≠
Global Tenant Access
```

---

# 17. TenantContextResolver

Contrato conceptual:

```php
interface TenantContextResolverInterface
{
    public function resolve(
        RuntimeContext $context,
    ): TenantContext;
}
```

---

# 18. Fuentes posibles

El Tenant podrá resolverse desde:

```text
validated hostname
subdomain
route parameter
authenticated membership
signed token
API credential
queue envelope
CLI argument
trusted infrastructure metadata
```

---

# 19. Fuentes no confiables

No deberá confiarse directamente en:

```text
X-Tenant-ID
query ?tenant=
POST tenant_id
cookie arbitraria
```

como autoridad suficiente.

---

# 20. Tenant Selection vs Tenant Authorization

Que el usuario solicite:

```text
tenant=7
```

solo significa:

```text
requested tenant
```

No:

```text
authorized tenant
```

---

# 21. Resolution Pipeline

```text
Tenant Candidate
      ↓
Candidate Validation
      ↓
Authentication Context
      ↓
Membership / Access Check
      ↓
TenantContext
```

---

# 22. Tenant Membership

Un Principal podrá pertenecer a cero, uno o múltiples tenants.

Modelo:

```text
Principal
    ↓ member
Tenant
```

---

# 23. TenantMembership

Conceptualmente:

```php
final readonly class TenantMembership
{
    public function __construct(
        public PrincipalReference $principal,
        public TenantReference $tenant,
        public TenantMembershipStatus $status,
    ) {}
}
```

---

# 24. Membership Status

Ejemplo:

```php
enum TenantMembershipStatus: string
{
    case Active = 'active';
    case Suspended = 'suspended';
    case Invited = 'invited';
    case Revoked = 'revoked';
}
```

---

# 25. Active Membership

Solo:

```text
Active
```

deberá considerarse acceso normal al Tenant.

---

# 26. Invited Membership

Una invitación no debe otorgar automáticamente:

```text
tenant access
```

antes de aceptación, salvo que el dominio lo defina expresamente.

---

# 27. Suspended Membership

Un usuario suspendido del Tenant deberá ser rechazado aunque:

```text
authentication = valid
```

---

# 28. Revoked Membership

Una membresía revocada deberá invalidar:

```text
tenant roles
tenant permissions
tenant access
```

tan pronto como lo permita la consistencia configurada.

---

# 29. TenantMembershipRepository

```php
interface TenantMembershipRepositoryInterface
{
    public function membership(
        PrincipalInterface $principal,
        TenantReference $tenant,
    ): ?TenantMembership;
}
```

---

# 30. TenantMembershipEvaluator

Podrá participar como evaluator temprano:

```text
TenantContext
      ↓
TenantMembershipEvaluator
      ↓
GRANT / DENY
```

---

# 31. Membership no siempre requerida

Un:

```text
SystemPrincipal
```

o ciertos:

```text
GlobalPrincipal
```

pueden no requerir membresía convencional.

Esto debe declararse mediante un modelo explícito, no bypass accidental.

---

# 32. TenantIsolationPolicy

VoltStack deberá proporcionar una política transversal:

```text
TenantIsolationPolicy
```

que participe con prioridad elevada.

---

# 33. Responsabilidad

Debe responder:

```text
¿el Subject pertenece al Tenant
que esta operación puede utilizar?
```

---

# 34. Ejemplo

Context:

```text
Tenant#7
```

Subject:

```text
Invoice#928
tenant=7
```

Resultado:

```text
GRANT / ABSTAIN
```

según estrategia.

---

# 35. Cross-Tenant Subject

Context:

```text
Tenant#7
```

Subject:

```text
Invoice#928
tenant=9
```

Resultado:

```text
DENY
```

---

# 36. Priority

`TenantIsolationPolicy` deberá ejecutarse antes de Policies normales cuando el Subject esté disponible.

Ejemplo:

```text
TenantIsolationPolicy
priority=9000

InvoicePolicy
priority=5000
```

---

# 37. NonBypassable

En configuración estricta podrá marcarse:

```text
NonBypassable
```

para impedir que:

```text
SuperAdmin
GrantOverride
AffirmativeStrategy
```

anule accidentalmente el aislamiento.

---

# 38. Critical Evaluator

La implementación recomendada será tratar Tenant Isolation como:

```text
Critical Security Evaluator
```

---

# 39. TenantAwareSubject

Los Subjects multi-tenant podrán exponer:

```php
interface TenantAwareSubjectInterface
{
    public function tenantReference(): TenantReference;
}
```

---

# 40. Adapter Alternative

No se debe obligar a entidades de dominio a implementar interfaces del Authorization package.

Podrá existir:

```text
SubjectTenantResolver
```

---

# 41. SubjectTenantResolver

```php
interface SubjectTenantResolverInterface
{
    public function supports(mixed $subject): bool;

    public function resolve(
        mixed $subject,
    ): ?TenantReference;
}
```

---

# 42. Resolver Registry

Podrán registrarse resolvers para:

```text
Invoice
Project
Document
Customer
Order
Custom package resources
```

---

# 43. Compiled Subject Tenant Metadata

Para modelos conocidos, metadata podrá indicar:

```text
tenant field
tenant relation
tenant resolver
```

sin discovery repetitivo.

---

# 44. Tenant-less Subjects

No todos los Subjects pertenecen a Tenant.

Ejemplos:

```text
Country
Currency
Framework configuration
Global catalog
```

`TenantIsolationPolicy` podrá:

```text
ABSTAIN
```

sobre ellos.

---

# 45. Global Subjects

Un Subject podrá declararse:

```text
GlobalResource
```

explícitamente.

No inferir globalidad porque:

```text
tenant_id = null
```

---

# 46. Regla importante

```text
tenant_id = null
```

puede significar:

```text
global
invalid data
legacy record
unassigned resource
```

La semántica debe ser explícita.

---

# 47. Tenant Ownership

Un resource puede pertenecer directamente al Tenant:

```text
Invoice
tenant_id=7
```

---

# 48. Indirect Ownership

También:

```text
Document
    ↓ belongs to
Project
    ↓ belongs to
Tenant
```

---

# 49. Ownership Resolver

Podrá existir:

```text
TenantOwnershipResolver
```

para resolver ownership indirecto.

---

# 50. Ownership vs Membership

Deben distinguirse:

```text
Tenant owns Resource
```

de:

```text
Principal belongs to Tenant
```

---

# 51. Authorization Chain

Una operación típica requiere:

```text
Principal member of Tenant
AND
Subject owned by Tenant
AND
Principal has required Permission
AND
Resource Policy grants Ability
```

---

# 52. Tenant-scoped RBAC

El sistema definido en el documento 12 deberá utilizar:

```text
AuthorizationScope::Tenant
```

---

# 53. Ejemplo

```text
User#42

Tenant#7:
  role = admin

Tenant#9:
  role = viewer
```

---

# 54. Effective Roles

Cuando:

```text
Current Tenant = 7
```

Roles efectivos:

```text
admin
```

No:

```text
viewer
```

---

# 55. Tenant-scoped Permissions

Ejemplo:

```text
User#42

invoice.approve
scope=Tenant#7
```

no concede:

```text
invoice.approve
scope=Tenant#9
```

---

# 56. Global Permissions

Algunas permissions podrán ser:

```text
scope=Global
```

Ejemplo:

```text
platform.tenants.view
```

---

# 57. Global Permission no implica Tenant Resource Access

Esto:

```text
platform.tenants.view
```

no debe convertir automáticamente todas las Policies tenant-aware en GRANT.

---

# 58. Explicit Cross-Tenant Permissions

Para operaciones legítimas:

```text
platform.tenant.inspect
platform.tenant.support
platform.tenant.manage
```

pueden existir permissions globales específicas.

---

# 59. Cross-Tenant Access

Debe modelarse como una operación explícita.

No:

```text
turn off tenant scope
```

---

# 60. CrossTenantAuthorizationRequirement

Conceptualmente:

```text
Actor
+
Target Tenant
+
Ability
+
Reason
```

---

# 61. Cross-Tenant Policy

Podrá existir:

```text
CrossTenantAccessPolicy
```

---

# 62. Ejemplo

Support engineer:

```text
Principal:
SupportAgent#10

Global Permission:
platform.tenant.support

Target:
Tenant#7
```

La Policy puede exigir:

```text
active support case
MFA
approved purpose
audit reason
time-limited access
```

---

# 63. Cross-Tenant Context

No se recomienda cambiar silenciosamente:

```text
TenantContext Tenant#1
```

a:

```text
Tenant#7
```

en mitad de una request.

---

# 64. Tenant Context Switching

Si debe soportarse:

```text
TenantContextSwitcher
```

deberá crear un scope controlado.

---

# 65. Scoped Tenant Execution

Ejemplo conceptual:

```php
$tenantRuntime->run(
    $tenant,
    function () {
        // tenant-scoped operation
    }
);
```

---

# 66. Context Restoration

Al salir:

```text
previous context
```

deberá restaurarse incluso si ocurre exception.

---

# 67. Nested Contexts

Deberán estar:

```text
stack-based
```

y protegidos contra corrupción.

---

# 68. Recommendation

En HTTP normal:

```text
one request
=
one tenant context
```

cuando sea posible.

---

# 69. Global Administration

Paneles SaaS pueden requerir:

```text
list tenants
inspect tenant
suspend tenant
support tenant
```

---

# 70. Platform Context

Estas operaciones deberán ejecutarse bajo:

```text
Global
```

o contexto administrativo explícito.

---

# 71. Global Admin ≠ Tenant Admin

Roles distintos:

```text
platform.admin
tenant.admin
```

---

# 72. Namespace Recommendation

Permissions:

```text
platform.*
tenant.*
```

deberán ser claramente diferenciables.

---

# 73. Platform Admin

No deberá obtener automáticamente:

```text
all tenant business permissions
```

---

# 74. Support Access

Un operador de soporte puede necesitar:

```text
read limited tenant information
```

sin:

```text
invoice.approve
customer.delete
```

---

# 75. Purpose-Limited Access

Cross-tenant access puede incluir:

```text
purpose
ticket
expiration
```

---

# 76. Support Session

Conceptualmente:

```text
SupportAccessSession

actor
target_tenant
permissions
reason
approved_by
expires_at
```

---

# 77. Support Session Security

Deberá ser:

```text
explicit
auditable
time-limited
revocable
```

---

# 78. Impersonation

Cross-tenant support puede utilizar impersonation.

Pero debe distinguir:

```text
Actor
```

de:

```text
Effective Principal
```

---

# 79. Example

```text
Actor:
SupportAgent#10

Effective Principal:
TenantUser#42

Tenant:
7
```

---

# 80. Audit

Debe conservar:

```text
SupportAgent#10
performed X
as User#42
inside Tenant#7
```

---

# 81. No actor loss

Nunca reemplazar completamente la identidad original.

---

# 82. Impersonation Restrictions

Podrá prohibirse durante impersonation:

```text
password changes
API key creation
billing changes
role escalation
security configuration
```

---

# 83. Tenant Impersonation Policy

Podrá existir:

```text
TenantImpersonationPolicy
```

---

# 84. Resource Binding

Route Model Binding deberá integrarse con Tenant Context.

---

# 85. Unsafe Binding

Evitar:

```sql
SELECT * FROM invoices
WHERE id = :id
```

seguido únicamente por Policy.

---

# 86. Preferred Binding

```sql
SELECT * FROM invoices
WHERE id = :id
AND tenant_id = :tenant
```

---

# 87. Benefit

Un resource de otro Tenant aparece como:

```text
not found within tenant scope
```

---

# 88. Security Property

Esto reduce:

```text
resource enumeration
cross-tenant leakage
accidental model access
```

---

# 89. Scoped Route Binding

Conceptualmente:

```text
Route Parameter
      ↓
TenantScopedBindingResolver
      ↓
Tenant-aware Query
      ↓
Subject
```

---

# 90. Route Parameter

Ejemplo:

```text
/tenants/7/invoices/928
```

No debe asumirse que:

```text
tenant=7
```

y:

```text
invoice=928
```

están relacionados.

Debe verificarse mediante query/scoping.

---

# 91. Nested Binding

```text
Tenant#7
    ↓
Invoice#928
```

binding deberá garantizar ownership.

---

# 92. Parent-child Route Scoping

Ejemplo:

```text
/projects/{project}/documents/{document}
```

deberá validar:

```text
document belongs to project
```

además del Tenant cuando corresponda.

---

# 93. Authorization Still Required

Aunque scoped binding encuentre el resource:

```text
InvoicePolicy::view
```

deberá decidir si ese Principal concreto puede verlo.

---

# 94. 404 vs 403

Para cross-tenant resource lookup, la opción segura recomendada será normalmente:

```text
404
```

porque el resource no existe dentro del scope visible.

---

# 95. Same-Tenant Authorization Denial

Dentro del Tenant:

```text
resource exists
but Principal lacks permission
```

puede devolver:

```text
403
```

---

# 96. Concealment

Policies sensibles podrán seguir utilizar:

```text
404 concealment
```

incluso dentro del mismo Tenant.

---

# 97. Data Access Layer

El Database System deberá soportar tenant scoping.

---

# 98. Query Isolation

Dependiendo del modelo físico:

```text
WHERE tenant_id = ?
```

puede añadirse automáticamente.

---

# 99. Shared Database / Shared Schema

Modelo:

```text
Database
  ├── Tenant 1 rows
  ├── Tenant 2 rows
  └── Tenant N rows
```

requiere filtros fuertes por Tenant.

---

# 100. Shared Database / Separate Schema

Modelo:

```text
Database
  ├── tenant_1 schema
  ├── tenant_2 schema
  └── tenant_n schema
```

Tenant Context selecciona schema.

---

# 101. Database per Tenant

Modelo:

```text
Tenant 1 → DB1
Tenant 2 → DB2
Tenant 3 → DB3
```

Tenant Context participa en connection resolution.

---

# 102. Hybrid Tenancy

VoltStack deberá permitir:

```text
shared DB for small tenants
dedicated DB for enterprise tenants
```

sin cambiar la semántica de autorización.

---

# 103. Authorization Independence

Authorization deberá trabajar con:

```text
TenantReference
```

sin necesitar saber si los datos están en:

```text
same table
schema
database
cluster
```

---

# 104. Data Isolation Strategy

Esta responsabilidad pertenece principalmente al Database/Tenancy subsystem.

Authorization lo refuerza.

---

# 105. Tenant Query Scope

Podrá existir:

```text
TenantQueryScope
```

aplicado a entidades tenant-aware.

---

# 106. Scope Bypass

Una API:

```text
withoutTenantScope()
```

es peligrosa.

---

# 107. Recommendation

No proporcionar bypass genérico de alto nivel.

Utilizar APIs explícitas como:

```text
GlobalTenantQuery
SystemTenantQuery
```

con autorización/contexto adecuado.

---

# 108. Privileged Query Context

Una query cross-tenant deberá requerir:

```text
Global/System Context
```

o token interno específico.

---

# 109. Repository Isolation

Repositories tenant-aware deberán recibir o resolver Tenant Context de forma segura.

---

# 110. Explicit Repository Alternative

Para operaciones críticas:

```php
$repository->forTenant($tenant)->find($id);
```

puede ser preferible.

---

# 111. Ambient vs Explicit Context

VoltStack podrá soportar ambos estilos:

```text
ambient request-scoped TenantContext
```

y:

```text
explicit TenantReference
```

pero no deberá permitir contradicciones silenciosas.

---

# 112. Context Conflict

Si:

```text
Ambient Tenant = 7
Explicit Tenant = 9
```

una API normal deberá:

```text
throw / deny
```

---

# 113. Global API

Cross-tenant APIs deberán estar claramente diferenciadas.

---

# 114. Persistence Isolation

No basta con proteger reads.

Writes también deben garantizar Tenant ownership.

---

# 115. Insert

Al crear:

```text
Invoice
```

el `tenant_id` deberá derivarse de un contexto confiable.

---

# 116. Unsafe Insert

Evitar:

```php
$invoice->tenant_id = $request->tenant_id;
```

---

# 117. Preferred

```text
TenantContext
    ↓
Persistence Layer
    ↓
tenant_id
```

---

# 118. Mass Assignment

Campos como:

```text
tenant_id
organization_id
owner_tenant_id
```

deben considerarse security-sensitive.

---

# 119. Update

Una actualización normal no deberá poder mover un resource de:

```text
Tenant#7
```

a:

```text
Tenant#9
```

por mass assignment.

---

# 120. Tenant Transfer

Si el dominio permite mover recursos entre tenants:

```text
TenantTransferService
```

deberá manejarlo explícitamente.

---

# 121. Transfer Authorization

Podrá requerir:

```text
source tenant authorization
AND
target tenant authorization
AND
platform authorization
```

según dominio.

---

# 122. Transaction

Transferencias deberán ser transaccionales cuando sea posible.

---

# 123. Foreign Keys

El Database System debería utilizar constraints que ayuden a impedir relaciones cross-tenant inválidas cuando el esquema lo permita.

---

# 124. Composite Tenant Keys

Ejemplo:

```text
tenant_id
project_id
```

puede formar parte de constraints compuestos.

---

# 125. Database-Level Security

PostgreSQL RLS u otros mecanismos podrán utilizarse como capa adicional.

---

# 126. Row-Level Security

Arquitectura:

```text
Application Tenant Context
       ↓
DB session/context
       ↓
RLS policy
```

---

# 127. RLS no sustituye Authorization

RLS responde principalmente:

```text
which rows can be accessed
```

No necesariamente:

```text
may this user approve this invoice?
```

---

# 128. Defense Stack

```text
RLS
+
Tenant Query Scope
+
TenantIsolationPolicy
+
Resource Policy
```

puede proporcionar aislamiento robusto.

---

# 129. Database Connection Leakage

Con persistent connections debe garantizarse que:

```text
Tenant A session state
```

no sobreviva a:

```text
Tenant B request
```

---

# 130. RLS Session Variables

Si se utiliza:

```text
SET app.tenant_id = 7
```

debe resetearse rigurosamente.

---

# 131. Connection Pool Safety

Al devolver una conexión al pool:

```text
tenant-specific DB state
```

debe limpiarse.

---

# 132. FrankenPHP

Esto es especialmente importante bajo workers persistentes.

---

# 133. Persistent Worker Lifecycle

```text
Request A
Tenant#7
      ↓
cleanup
      ↓
Request B
Tenant#9
```

---

# 134. Mandatory Cleanup

Al terminar cada request:

```text
TenantContext
PrincipalContext
AuthorizationExecution
Tenant-scoped caches
resolved subjects
DB tenant state
```

deberán resetearse.

---

# 135. Worker Invariant

```text
No tenant-specific mutable state
may remain attached to process-wide services.
```

---

# 136. Singleton Safety

Correcto:

```text
TenantResolver
TenantIsolationPolicy
PermissionChecker
```

pueden ser shared si son stateless.

Incorrecto:

```php
$this->currentTenant = $tenant;
```

dentro de ellos.

---

# 137. Request-scoped Context Holder

Si se necesita acceso ambiental:

```text
TenantContextHolder
```

deberá estar ligado al lifecycle del request.

---

# 138. Context Propagation

El contexto deberá propagarse explícitamente hacia:

```text
Authorization
Database
Cache
Events
Filesystem
Queues
Logging
```

cuando esos sistemas sean tenant-aware.

---

# 139. Cache Isolation

Cache keys tenant-aware deberán incluir Tenant.

---

# 140. Unsafe Cache Key

```text
invoice:928
```

---

# 141. Safe Cache Key

```text
tenant:7:invoice:928
```

---

# 142. Cache Namespace

Podrá existir:

```text
TenantCacheNamespace
```

---

# 143. Authorization Cache

RBAC cache:

```text
permissions:user:42
```

es insuficiente si los grants son tenant-scoped.

---

# 144. Correct RBAC Cache

```text
authz:tenant:7:principal:42:v18
```

---

# 145. Decision Memoization

Debe incluir:

```text
Tenant Context fingerprint
```

cuando el Tenant afecte la decisión.

---

# 146. Cross-Tenant Cache Poisoning

Nunca reutilizar una decisión tomada en:

```text
Tenant#7
```

para:

```text
Tenant#9
```

---

# 147. Filesystem Isolation

Aunque este documento se centra en Authorization, un recurso de filesystem tenant-aware deberá usar:

```text
tenant-specific path/prefix/storage
```

además de autorización.

---

# 148. Example

```text
tenants/7/documents/...
```

---

# 149. Object Storage

Keys/buckets deberán respetar estrategia de tenancy.

Authorization deberá ocurrir antes de generar acceso.

---

# 150. Signed URLs

Una signed URL a storage no deberá otorgar un scope mayor al recurso autorizado.

---

# 151. Signed URL Lifetime

Debe ser:

```text
limited
purpose-specific
```

cuando el recurso sea sensible.

---

# 152. Event Isolation

Eventos tenant-aware deberán transportar:

```text
TenantReference
```

---

# 153. Event Envelope

Conceptualmente:

```text
Event
Actor
Tenant
Correlation ID
```

---

# 154. Async Consumers

Al consumir:

```text
restore TenantContext
    ↓
process event
    ↓
clear TenantContext
```

---

# 155. Event Trust

El Tenant incluido en un evento interno debe provenir de envelope firmado/confiable o infraestructura interna.

---

# 156. Queue Jobs

Jobs tenant-aware deberán serializar:

```text
TenantReference
```

no un objeto mutable completo.

---

# 157. Queue Envelope

```text
Job ID
TenantReference
PrincipalReference / Delegated Grant
Correlation ID
Payload
```

---

# 158. Job Execution

```text
Deserialize
    ↓
Validate Tenant
    ↓
Establish TenantContext
    ↓
Resolve Principal
    ↓
Reauthorize when required
    ↓
Execute
    ↓
Cleanup
```

---

# 159. Tenant Deletion

Si el Tenant fue eliminado entre dispatch y ejecución:

```text
job must not run as normal
```

---

# 160. Membership Revocation

Si el Principal perdió acceso antes de ejecutar el Job:

```text
reauthorization
```

deberá observarlo por defecto.

---

# 161. Delegated Jobs

Cuando el Job representa una autorización delegada específica, podrá utilizar:

```text
DelegatedAuthorizationGrant
```

con:

```text
tenant
ability
subject
expiration
```

---

# 162. Queue Leakage

Un worker que procesa múltiples tenants debe limpiar contexto después de cada Job, igual que HTTP.

---

# 163. Scheduler

Scheduled tasks podrán ser:

```text
global
per-tenant
system
```

---

# 164. Per-Tenant Scheduler

Conceptualmente:

```text
for each Tenant:
    establish context
    execute task
    clear context
```

---

# 165. Error Isolation

Un error en Tenant#7 no deberá dejar el contexto activo para Tenant#9.

Utilizar:

```text
try/finally
```

semánticamente.

---

# 166. CLI

CLI deberá distinguir:

```text
tenant command
global command
system command
```

---

# 167. Tenant CLI

Ejemplo:

```text
volt invoices:rebuild --tenant=7
```

---

# 168. CLI Tenant Validation

`--tenant=7` no deberá convertirse directamente en un Context válido sin resolver/verificar Tenant.

---

# 169. Interactive Admin

Si un comando es ejecutado por un operador autenticado, pueden aplicarse Policies administrativas.

---

# 170. Automated CLI

Migrations/provisioning podrán utilizar System Context.

---

# 171. System Context Guard

No debe estar disponible mediante input HTTP normal.

---

# 172. System Principal

Representa procesos internos.

Ejemplo:

```text
SystemPrincipal:
tenant-provisioner
```

---

# 173. System Principal no es omnipotente

Cada System Principal deberá tener capacidades específicas.

---

# 174. Bad Model

```text
if principal instanceof SystemPrincipal:
    GRANT
```

---

# 175. Correct Model

```text
SystemPrincipal:
billing-reconciler

Capabilities:
billing.reconcile
```

---

# 176. Least Privilege

Aplica también a procesos internos.

---

# 177. Tenant Provisioning

Crear un Tenant puede requerir:

```text
Global/System Context
```

---

# 178. Provisioning Pipeline

```text
Create Tenant
      ↓
Create isolation resources
      ↓
Create initial membership
      ↓
Assign tenant owner role
      ↓
Commit
      ↓
Emit TenantProvisioned
```

---

# 179. Initial Owner

La asignación inicial deberá ocurrir mediante un provisioning workflow confiable.

---

# 180. Tenant Deprovisioning

Eliminar/suspender un Tenant deberá afectar:

```text
authentication access
authorization grants
jobs
storage
cache
database access
API credentials
```

según lifecycle.

---

# 181. Tenant Suspension

Una Policy global:

```text
TenantStatusPolicy
```

puede rechazar operaciones si:

```text
Tenant.status = suspended
```

---

# 182. Priority

Deberá ejecutarse temprano.

---

# 183. Read-Only Tenant

Un Tenant puede entrar en:

```text
read_only
```

por billing, migration o incident response.

---

# 184. TenantOperationalPolicy

Puede decidir:

```text
read abilities → allowed
mutation abilities → denied
```

---

# 185. Ability Classification

Ability metadata puede indicar:

```text
read
write
administrative
destructive
```

para facilitar esta Policy.

---

# 186. Tenant State

Ejemplos:

```text
active
suspended
read_only
provisioning
deleting
archived
```

---

# 187. State-specific Authorization

El Planner puede incluir:

```text
TenantStatusEvaluator
```

cuando existe Tenant Context.

---

# 188. Tenant Hierarchies

Algunas aplicaciones poseen:

```text
Parent Tenant
    ↓
Child Tenant
```

---

# 189. No implicit hierarchy

Parent no deberá acceder automáticamente a Child.

---

# 190. Hierarchical Access

Deberá modelarse mediante:

```text
ReBAC
```

o una Policy explícita.

---

# 191. Example

```text
HoldingCompany
parent_of
Subsidiary
```

no implica:

```text
all permissions
```

---

# 192. Delegated Parent Access

Puede existir:

```text
parent.billing.view
```

sin:

```text
parent.customer.view
```

---

# 193. Tenant Groups

Múltiples tenants podrán agruparse.

Esto también se modela mejor mediante ReBAC.

---

# 194. Tenant-to-Tenant Relationships

Ejemplos:

```text
parent_of
managed_by
partner_of
reseller_of
```

---

# 195. ReBAC Integration

Cross-tenant access puede requerir:

```text
Principal
member_of
Tenant A

Tenant A
manages
Tenant B
```

más Permission adecuada.

---

# 196. Hybrid Decision

```text
Global/Tenant Permission
+
Tenant Relationship
+
Resource Policy
```

---

# 197. Public Resources

Un Tenant puede publicar recursos.

Ejemplo:

```text
public catalog
public article
public profile
```

---

# 198. Public does not remove ownership

El resource sigue perteneciendo al Tenant.

Solo una Ability específica puede ser pública.

---

# 199. Public Policy

```text
viewPublished
```

podrá permitir AnonymousPrincipal.

---

# 200. Mutation Remains Tenant-Protected

Public visibility no implica:

```text
update
delete
manage
```

---

# 201. Shared Resources

Un resource puede ser compartido entre tenants.

Esto rompe el modelo simple:

```text
resource.tenant_id
```

---

# 202. Resource Access Relation

Podrá modelarse:

```text
Resource
owned_by Tenant#7

shared_with Tenant#9
shared_with Tenant#10
```

---

# 203. ReBAC for Shared Resources

Recomendado:

```text
Tenant#9
viewer_of
Document#100
```

---

# 204. Ownership Still Unique

Si el dominio lo requiere:

```text
owner tenant
```

puede seguir siendo único aunque haya múltiples access relationships.

---

# 205. Shared Resource Policy

Deberá distinguir:

```text
owner permissions
shared permissions
```

---

# 206. Data Querying

Una query de recursos visibles puede requerir:

```text
owned resources
UNION
shared resources
```

---

# 207. Authorization vs Query Visibility

La lista debe filtrar correctamente recursos visibles.

No cargar todos y llamar `can()` fila por fila si puede evitarse.

---

# 208. Authorization-aware Query Constraints

Futuro:

```text
AuthorizationQueryScope
```

podrá traducir ciertas reglas de visibilidad a queries.

---

# 209. Important Boundary

No todas las Policies pueden traducirse a SQL.

El sistema no debe prometer equivalencia automática.

---

# 210. Collection Authorization

Para endpoints:

```text
GET /invoices
```

deben distinguirse:

```text
viewAny
```

y:

```text
which invoices are visible
```

---

# 211. `viewAny`

Responde:

```text
may Principal access invoice listing?
```

---

# 212. Query Scope

Responde:

```text
which invoice rows may Principal see?
```

---

# 213. Tenant Query Scope

Primero limita:

```text
Tenant#7
```

y luego otras reglas de visibilidad.

---

# 214. Avoid Cross-Tenant Pagination Leakage

La paginación debe ejecutarse después de aplicar Tenant scope.

---

# 215. Counts

Queries como:

```text
COUNT(*)
```

también deben respetar Tenant.

---

# 216. Aggregations

Igualmente:

```text
SUM
AVG
GROUP BY
```

---

# 217. Search

Full-text search deberá incluir Tenant scope antes de devolver resultados.

---

# 218. Search Index Isolation

Si se usa Elasticsearch/OpenSearch/etc.:

```text
tenant filter
```

deberá ser obligatorio o utilizar índices separados.

---

# 219. Search Result Authorization

Resultados sensibles pueden requerir Policy adicional.

---

# 220. Export

Exportaciones son un vector importante de fuga cross-tenant.

---

# 221. Export Policy

Deberá verificar:

```text
tenant
permission
query scope
requested columns
resource sensitivity
```

---

# 222. Bulk Operations

Bulk update/delete deberá aplicar Tenant scope a toda la operación.

---

# 223. Unsafe Bulk Delete

```sql
DELETE FROM invoices
WHERE status = 'expired'
```

---

# 224. Tenant-safe Bulk Delete

```sql
DELETE FROM invoices
WHERE tenant_id = :tenant
AND status = 'expired'
```

---

# 225. Global Bulk Operation

Solo APIs privilegiadas podrán operar sobre todos los tenants.

---

# 226. Database Import

Import deberá asociar datos al Tenant Context confiable.

---

# 227. User-provided Tenant IDs

No confiar en tenant IDs dentro del archivo importado salvo que el workflow global lo permita.

---

# 228. Data Export

El Tenant deberá formar parte del scope de exportación.

---

# 229. Backup

Backups pueden ser:

```text
global
tenant-specific
```

---

# 230. Restore

Un restore tenant-specific no deberá escribir sobre otro Tenant.

---

# 231. Operational Tooling

Administradores de infraestructura podrán necesitar acceso global, pero ese acceso debe estar separado del modelo normal de aplicación.

---

# 232. Logs

Logs deberán incluir:

```text
tenant identifier
```

cuando exista, para trazabilidad.

---

# 233. Sensitive Logging

No incluir información de otros tenants en mensajes de error visibles al usuario.

---

# 234. Correlation

Una request podrá registrar:

```text
correlation_id
tenant_id
principal_id
```

según política de privacidad.

---

# 235. Audit Record

Una decisión tenant-aware puede incluir:

```text
Actor
Effective Principal
Tenant
Ability
Subject Reference
Decision
Reason Code
Timestamp
Correlation ID
```

---

# 236. Cross-Tenant Audit

Para acceso global:

```text
source context
target tenant
reason
support session
```

deberán registrarse.

---

# 237. Failed Cross-Tenant Attempts

Intentos rechazados pueden ser security events.

---

# 238. Enumeration Protection

No registrar al cliente detalles como:

```text
resource belongs to tenant 9
```

---

# 239. Observability

Internamente sí puede registrarse de forma segura:

```text
tenant_mismatch
```

para diagnóstico.

---

# 240. Metrics

Ejemplos:

```text
authorization.tenant.denies
authorization.tenant.context_missing
authorization.tenant.scope_mismatch
authorization.cross_tenant.attempts
authorization.cross_tenant.grants
authorization.tenant.context_leaks_detected
```

---

# 241. Context Leak Detector

En development/testing podrá existir:

```text
TenantContextLeakDetector
```

---

# 242. Leak Detection

Después del request:

```text
assert TenantContext cleared
assert tenant cache cleared
assert DB tenant state reset
```

---

# 243. FrankenPHP Diagnostics

Especialmente útil para worker mode.

---

# 244. Testing Strategy

El sistema multi-tenant deberá tener tests específicos de aislamiento.

---

# 245. Fundamental Isolation Matrix

Para cada operación:

```text
User A / Tenant A / Resource A
→ expected result

User A / Tenant A / Resource B
→ DENY / NOT FOUND

User B / Tenant B / Resource A
→ DENY / NOT FOUND
```

---

# 246. Cross-Tenant Test

Nunca limitar tests a:

```text
authorized user
unauthorized user
```

Debe probarse:

```text
authorized in wrong tenant
```

---

# 247. Critical Test

Un usuario con:

```text
invoice.update
```

en Tenant#7 intentando Invoice Tenant#9 debe fallar.

---

# 248. Same ID Test

Muy importante:

```text
Tenant#7 Invoice#100
Tenant#9 Invoice#100
```

cuando arquitectura permite IDs repetidos.

Resolver Tenant#7 nunca debe obtener el Invoice de Tenant#9.

---

# 249. Cache Isolation Test

Mismo key lógico:

```text
settings
```

en dos tenants debe producir valores separados.

---

# 250. RBAC Cache Test

User con diferentes roles por Tenant debe recibir grants correctos tras cambiar de contexto controladamente.

---

# 251. Worker Reuse Test

Ejecutar:

```text
1000 requests
alternating tenants
```

y comprobar ausencia de leakage.

---

# 252. Parallel Test

Requests concurrentes de diferentes tenants no deben compartir contexto.

---

# 253. Queue Worker Test

Procesar:

```text
Job Tenant A
Job Tenant B
Job Tenant A
```

en el mismo worker y verificar limpieza.

---

# 254. Exception Cleanup Test

Provocar exception durante Tenant A y ejecutar Tenant B después.

Tenant A no debe persistir.

---

# 255. DB Connection Test

Si hay session-level Tenant state:

```text
set Tenant A
release connection
reuse connection for Tenant B
```

deberá resetearse correctamente.

---

# 256. Route Binding Test

ID de recurso perteneciente a otro Tenant debe producir resultado seguro.

---

# 257. Bulk Operation Test

Verificar que update/delete no afecte filas de otro Tenant.

---

# 258. Search Isolation Test

Queries de búsqueda no deben devolver hits cross-tenant.

---

# 259. Export Isolation Test

CSV/JSON exports no deben contener registros externos al Tenant.

---

# 260. Property-Based Testing

Propiedad:

```text
For every Tenant A != Tenant B:

A normal Tenant-scoped operation in A
must not mutate B.
```

---

# 261. Mutation Property

Después de cualquier operación tenant-scoped:

```text
state of unrelated tenants remains unchanged
```

---

# 262. Fuzz Testing

Podrá variar:

```text
tenant IDs
resource IDs
route parameters
cache keys
job envelopes
```

buscando inconsistencias.

---

# 263. Authorization Test Helpers

Podrán existir:

```php
actingAsTenant($tenant);

actingAsTenantUser($user, $tenant);

assertTenantDenied(...);

assertTenantNotFound(...);
```

---

# 264. Testing Context

Helpers deberán limpiar automáticamente contexto después de cada test.

---

# 265. Static Analysis

Tooling podrá detectar queries tenant-aware ejecutadas sin Tenant Context.

---

# 266. Tenant Model Metadata

Ejemplo:

```php
#[TenantOwned(
    field: 'tenant_id'
)]
final class Invoice
{
}
```

---

# 267. Repository Metadata

También:

```text
TenantScopedRepository
```

---

# 268. Compiler Validation

Puede verificar:

```text
TenantOwned entity
+
route binding
+
repository
+
policy
```

para detectar configuraciones incompletas.

---

# 269. Authorization Coverage

Tooling podrá reportar:

```text
Tenant-aware Subjects: 42

Protected by TenantIsolationPolicy: 42

Tenant-scoped bindings: 39

Potential unsafe bindings: 3
```

---

# 270. Unsafe Query Detection

En development podrá advertir:

```text
Tenant-owned model queried
without Tenant Context
```

---

# 271. Explicit Global Query

Para evitar falsos positivos:

```text
GlobalTenantQuery
```

marca intención explícita.

---

# 272. Tooling Commands

Posibles comandos:

```text
volt tenant:authorization-check
volt tenant:isolation-audit
volt tenant:bindings
volt tenant:context
volt authorization:tenant-explain
```

---

# 273. Isolation Audit

Podría revisar:

```text
models
bindings
repositories
permissions
cache namespaces
jobs
```

---

# 274. Explain

Ejemplo:

```text
volt authorization:tenant-explain \
--principal=user:42 \
--tenant=7 \
--ability=update \
--subject=invoice:928
```

---

# 275. Output

```text
Tenant Context:
Tenant#7

Membership:
ACTIVE → GRANT

Subject Tenant:
Tenant#7 → MATCH

Role:
finance-manager → GRANT

Permission:
invoice.update → GRANT

InvoicePolicy:
GRANT

Final:
GRANT
```

---

# 276. Cross-Tenant Explain

```text
Tenant Context:
Tenant#7

Subject Tenant:
Tenant#9

TenantIsolationPolicy:
DENY

Remaining resource policies:
SKIPPED

Final:
DENY
```

---

# 277. Fail Closed

Si un tenant-aware Subject debería proporcionar Tenant pero el resolver falla:

```text
FAILED / DENY
```

No:

```text
ABSTAIN → accidental grant
```

---

# 278. Unknown Tenant

Si el Tenant solicitado no existe:

```text
404 / domain-specific not found
```

según integración.

---

# 279. Unauthorized Tenant

Si existe pero el Principal no debe saberlo:

```text
404 concealment
```

puede utilizarse.

---

# 280. Tenant Status Failure

Un Tenant suspendido podrá producir:

```text
403
423
custom domain response
```

en HTTP integration.

El Core sigue devolviendo una decisión independiente de transporte.

---

# 281. Tenant Deletion Race

Si Tenant desaparece durante operación:

```text
transaction/domain layer
```

debe manejar consistencia.

---

# 282. Subject Transfer Race

Si Resource cambia de Tenant entre check y update:

```text
TOCTOU
```

debe prevenirse con transacciones/locking/constraints cuando sea crítico.

---

# 283. Authorization Near Mutation

Operaciones críticas podrán revalidar:

```text
Tenant ownership
```

dentro de la transacción.

---

# 284. Query-and-Update

Preferible:

```sql
UPDATE invoices
SET ...
WHERE id = :id
AND tenant_id = :tenant
```

para operaciones sensibles.

---

# 285. Authorization Does Not Replace Constraints

La seguridad completa requiere:

```text
Authorization
+
correct query
+
correct transaction
+
database constraints
```

---

# 286. Multi-Tenant Policy Example

```php
final class InvoicePolicy
{
    public function update(
        User $user,
        Invoice $invoice,
        AuthorizationContext $context,
    ): DecisionResult {
        if (!$this->permissions->has(
            $user,
            'invoice.update',
            $context
        )) {
            return DecisionResult::deny(
                'rbac.permission_missing'
            );
        }

        if ($invoice->status->isLocked()) {
            return DecisionResult::deny(
                'invoice.locked'
            );
        }

        return DecisionResult::grant();
    }
}
```

Nótese que no necesita repetir necesariamente:

```php
$user->tenant_id === $invoice->tenant_id
```

si `TenantIsolationPolicy` ya garantiza esa frontera.

---

# 287. Defense Duplication

Para recursos extremadamente críticos puede repetirse una comprobación tenant-aware dentro del dominio.

Pero no debe convertirse en lógica inconsistente duplicada.

---

# 288. Policy Assumptions

Una Policy podrá declarar:

```text
RequiresTenantIsolation
```

para indicar que depende de que el evaluator crítico ya se haya ejecutado.

---

# 289. Planner Validation

El Planner puede garantizar:

```text
TenantIsolationPolicy
runs before
TenantDependentPolicy
```

---

# 290. Missing Tenant Isolation Evaluator

Para Subject marcado `TenantOwned`:

```text
fail plan compilation
```

si el sistema exige aislamiento estricto.

---

# 291. Tenant Authorization Plan

Ejemplo:

```text
PrincipalStatePolicy
priority=10000

TenantContextPolicy
priority=9500

TenantMembershipPolicy
priority=9200

TenantStatusPolicy
priority=9100

TenantIsolationPolicy
priority=9000

RoleEvaluator
priority=7000

PermissionEvaluator
priority=6500

RelationshipEvaluator
priority=6000

ResourcePolicy
priority=5000
```

---

# 292. Early vs Resource Phase

Antes de binding:

```text
Tenant Context
Tenant Membership
Tenant Status
global permission requirements
```

pueden evaluarse.

Después de binding:

```text
Subject Tenant Isolation
Resource Relationships
Resource Policy
```

---

# 293. Optimized Lifecycle

```text
Request
   ↓
Authenticate
   ↓
Resolve Tenant Candidate
   ↓
Validate Membership
   ↓
Tenant Status
   ↓
Route Match
   ↓
Pre-Authorization
   ↓
Tenant-Scoped Binding
   ↓
Tenant Isolation Verification
   ↓
Resource Authorization
   ↓
Controller
```

---

# 294. Benefit

Un usuario sin acceso al Tenant puede rechazarse antes de:

```text
resource lookup
controller argument resolution
expensive services
```

---

# 295. Tenant Context Source Metadata

El Context podrá registrar internamente:

```text
source=subdomain
source=route
source=token
source=queue
source=cli
```

para observability.

---

# 296. Do Not Trust Source Equally

Un Tenant de:

```text
signed service credential
```

tiene diferente trust que:

```text
route parameter
```

pero ambos deben pasar las reglas correspondientes.

---

# 297. Tenant Selection Policy

Podrá validar que el Principal puede seleccionar ese Tenant.

---

# 298. Multi-Membership UX

Un usuario miembro de varios tenants podrá cambiar de Tenant mediante un endpoint.

---

# 299. Tenant Switch Endpoint

Ejemplo:

```text
POST /tenant/switch
```

deberá:

```text
validate requested Tenant
verify active membership
establish future session context
rotate/update security state if necessary
audit switch
```

---

# 300. Session

La sesión podrá almacenar:

```text
selected_tenant_id
```

pero este valor debe revalidarse.

---

# 301. Stale Session Tenant

Si membership fue revocada:

```text
selected tenant
```

no deberá seguir otorgando acceso.

---

# 302. API Tokens

Tokens pueden ser:

```text
tenant-bound
global
multi-tenant
```

---

# 303. Tenant-Bound Token

Preferido para la mayoría de APIs:

```text
Token
tenant=7
```

---

# 304. Token Tenant Conflict

Si token:

```text
tenant=7
```

y request intenta:

```text
tenant=9
```

debe fallar.

---

# 305. Multi-Tenant Token

Debe ser explícito y restringido.

---

# 306. Service Credentials

Integraciones B2B pueden poseer acceso a múltiples tenants.

Utilizar scopes/relationships explícitos.

---

# 307. WebSocket / Realtime

Conexiones persistentes también necesitan Tenant Context.

---

# 308. Channel Authorization

Suscripción:

```text
tenant.7.invoices
```

deberá verificar acceso a Tenant#7.

---

# 309. Channel Leakage

No permitir que cambiar el nombre del canal otorgue acceso a otro Tenant.

---

# 310. Long-Lived Connections

Si membership cambia, deberá definirse:

```text
reauthorize
disconnect
token expiration
```

---

# 311. SPA

El frontend podrá conocer:

```text
current tenant
available tenant switch options
projected abilities
```

para UX.

---

# 312. SPA Is Not Security Boundary

Toda navegación/API mutation deberá autorizarse en backend.

---

# 313. Tenant Frontend State

Al cambiar Tenant:

```text
clear tenant-specific frontend cache/state
```

para evitar mostrar datos anteriores.

---

# 314. Hydration

Payloads de hidratación deberán estar ligados al Tenant correcto.

---

# 315. Stale Component

Un componente creado en Tenant#7 no deberá reutilizarse después de cambiar a Tenant#9 sin validación.

---

# 316. Tenant Fingerprint

Hydration state podrá incluir:

```text
tenant fingerprint
```

firmado.

---

# 317. Frontend Route Manifest

Rutas tenant-aware podrán marcarse:

```text
requiresTenant=true
```

solo como metadata UX/runtime.

---

# 318. Authorization Remains Server-Side

Siempre.

---

# 319. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── MultiTenancy/
        ├── Context/
        │   ├── TenantContext.php
        │   ├── TenantContextMode.php
        │   ├── TenantContextResolverInterface.php
        │   ├── TenantContextHolder.php
        │   └── TenantContextSwitcher.php
        │
        ├── Tenant/
        │   ├── TenantIdentifier.php
        │   ├── TenantReference.php
        │   └── TenantInterface.php
        │
        ├── Membership/
        │   ├── TenantMembership.php
        │   ├── TenantMembershipStatus.php
        │   ├── TenantMembershipRepositoryInterface.php
        │   └── TenantMembershipEvaluator.php
        │
        ├── Isolation/
        │   ├── TenantIsolationPolicy.php
        │   ├── TenantStatusPolicy.php
        │   ├── TenantOperationalPolicy.php
        │   ├── SubjectTenantResolverInterface.php
        │   ├── SubjectTenantResolverRegistry.php
        │   └── TenantOwnershipResolver.php
        │
        ├── CrossTenant/
        │   ├── CrossTenantAccessPolicy.php
        │   ├── CrossTenantAuthorizationRequirement.php
        │   ├── SupportAccessSession.php
        │   └── TenantImpersonationPolicy.php
        │
        ├── Runtime/
        │   ├── TenantRuntime.php
        │   ├── TenantExecutionScope.php
        │   ├── TenantContextLeakDetector.php
        │   └── TenantRuntimeCleaner.php
        │
        ├── Binding/
        │   ├── TenantScopedBindingResolver.php
        │   └── TenantBindingException.php
        │
        ├── Cache/
        │   ├── TenantCacheNamespace.php
        │   └── TenantAuthorizationCacheKey.php
        │
        ├── Queue/
        │   ├── TenantJobContext.php
        │   └── TenantJobContextMiddleware.php
        │
        └── Exceptions/
            ├── TenantAuthorizationException.php
            ├── TenantContextMissingException.php
            ├── TenantContextConflictException.php
            ├── TenantMembershipDeniedException.php
            ├── TenantIsolationViolationException.php
            └── CrossTenantAccessDeniedException.php
```

---

# 320. Tenant Context Invariants

### Invariante 1

`None` nunca equivale a `Global`.

### Invariante 2

El Tenant activo debe provenir de una fuente validada.

### Invariante 3

Tenant Context es runtime/request scoped.

### Invariante 4

Un Context no puede cambiarse silenciosamente.

### Invariante 5

Toda elevación a Global/System debe ser explícita.

---

# 321. Membership Invariants

### Invariante 1

Una membership pertenece a Principal + Tenant.

### Invariante 2

Roles de un Tenant no se transfieren a otro.

### Invariante 3

Suspended/Revoked membership no concede acceso.

### Invariante 4

La selección de Tenant no prueba membership.

---

# 322. Resource Isolation Invariants

### Invariante 1

Un Tenant-owned Subject tiene ownership resoluble.

### Invariante 2

Tenant mismatch produce DENY/fail closed.

### Invariante 3

`tenant_id=null` no significa global automáticamente.

### Invariante 4

TenantIsolationPolicy se ejecuta antes de resource Policies dependientes.

### Invariante 5

Cross-tenant resource access debe ser explícito.

---

# 323. Data Isolation Invariants

### Invariante 1

Queries normales están tenant-scoped.

### Invariante 2

Writes no aceptan Tenant ownership arbitrario desde input.

### Invariante 3

Bulk operations respetan Tenant scope.

### Invariante 4

Cross-tenant queries requieren API/contexto privilegiado.

### Invariante 5

Authorization no sustituye constraints de persistencia.

---

# 324. RBAC Invariants

### Invariante 1

Tenant Role solo aplica a su scope.

### Invariante 2

Tenant Permission solo aplica a su scope.

### Invariante 3

Global Role no implica acceso ilimitado a resources tenant-owned.

### Invariante 4

Super-admin no bypassa aislamiento automáticamente.

---

# 325. Cross-Tenant Invariants

### Invariante 1

Global access es una capacidad explícita.

### Invariante 2

Support access debe ser limitado y auditable.

### Invariante 3

Actor y Effective Principal permanecen diferenciados.

### Invariante 4

Platform Admin y Tenant Admin son conceptos distintos.

### Invariante 5

Parent Tenant no obtiene acceso a Child automáticamente.

---

# 326. Runtime Invariants

### Invariante 1

No existe estado Tenant mutable en singletons compartidos.

### Invariante 2

Workers limpian Tenant Context después de cada unidad de trabajo.

### Invariante 3

DB session state tenant-specific se limpia.

### Invariante 4

Caches request-scoped se limpian.

### Invariante 5

Exception paths también ejecutan cleanup.

---

# 327. Cache Invariants

### Invariante 1

Cache tenant-aware incluye Tenant namespace.

### Invariante 2

RBAC cache incluye scope.

### Invariante 3

Decision memoization incluye Tenant fingerprint.

### Invariante 4

Un valor de Tenant A nunca puede satisfacer lookup de Tenant B.

---

# 328. Async Invariants

### Invariante 1

Jobs transportan TenantReference explícita.

### Invariante 2

Workers restablecen y limpian contexto.

### Invariante 3

Principal authorization se revalida por defecto.

### Invariante 4

Tenant inexistente/suspendido no ejecuta normalmente.

### Invariante 5

Delegated grants son explícitos y limitados.

---

# 329. Security Invariants

### Invariante 1

Tenant ID proveniente del cliente no es autoridad.

### Invariante 2

Tenant Isolation es una frontera de seguridad.

### Invariante 3

Cross-tenant failures no revelan ownership innecesariamente.

### Invariante 4

Revocaciones críticas deben propagarse adecuadamente.

### Invariante 5

Toda operación privilegiada debe seguir least privilege.

---

# 330. Arquitectura final

```text
                         REQUEST
                            │
                            ↓
                     Authentication
                            │
                            ↓
                   Principal Context
                            │
                            ↓
                  Tenant Candidate
                            │
                            ↓
                Tenant Context Resolver
                            │
                            ↓
                   Tenant Context
                            │
             ┌──────────────┴──────────────┐
             ↓                             ↓
     Membership Validation           Tenant Status
             │                             │
             └──────────────┬──────────────┘
                            ↓
                   PRE-AUTHORIZATION
                            │
                            ↓
                      Route Match
                            │
                            ↓
                Tenant-Scoped Binding
                            │
                            ↓
                         Subject
                            │
                            ↓
                TenantIsolationPolicy
                            │
                     ┌──────┴──────┐
                     ↓             ↓
                   DENY          GRANT
                     │             │
                    STOP           ↓
                             RBAC / ABAC
                                  ReBAC
                                    │
                                    ↓
                             Resource Policy
                                    │
                                    ↓
                             DecisionManager
                                    │
                             ┌──────┴──────┐
                             ↓             ↓
                           DENY          GRANT
                             │             │
                            STOP           ↓
                                  Domain Operation
                                         │
                                         ↓
                                  Tenant-Safe
                                   Persistence
```

---

# 331. Arquitectura de aislamiento completa

```text
                    TENANT SECURITY BOUNDARY

Request
   │
   ├── Authentication
   │
   ├── Tenant Resolution
   │
   ├── Membership
   │
   ├── Tenant Status
   │
   ├── Tenant-scoped Routing
   │
   ├── Tenant-scoped Binding
   │
   ├── Tenant-scoped Database Queries
   │
   ├── TenantIsolationPolicy
   │
   ├── Tenant-scoped RBAC
   │
   ├── Resource Policy
   │
   ├── Tenant-safe Mutation
   │
   ├── Tenant Cache Namespace
   │
   ├── Tenant Event Context
   │
   └── Runtime Cleanup
```

No existe una única barrera cuya falla exponga automáticamente todos los datos.

---

# 332. Ejemplo completo — acceso permitido

Request:

```text
PUT /invoices/928
```

Principal:

```text
User#42
```

Context:

```text
Tenant#7
```

Membership:

```text
User#42
member of Tenant#7
ACTIVE
```

Role:

```text
finance-manager
scope=Tenant#7
```

Permission:

```text
invoice.update
scope=Tenant#7
```

Subject:

```text
Invoice#928
tenant=7
```

---

# 333. Pipeline

```text
Authentication
→ GRANT

Tenant Membership
→ GRANT

Tenant Status
→ GRANT

Tenant-scoped Invoice Binding
→ Invoice#928

TenantIsolationPolicy
→ GRANT

RoleEvaluator
→ GRANT

PermissionEvaluator
→ GRANT

InvoicePolicy::update
→ GRANT
```

Resultado:

```text
GRANT
```

---

# 334. Ejemplo completo — cross-tenant denial

Principal:

```text
User#42
```

Context:

```text
Tenant#7
```

Permission:

```text
invoice.update
scope=Tenant#7
```

Requested:

```text
Invoice#500
tenant=9
```

Con tenant-scoped binding:

```text
SELECT invoice
WHERE id=500
AND tenant_id=7
```

Resultado:

```text
NOT FOUND
```

El resource de Tenant#9 nunca llega a `InvoicePolicy`.

---

# 335. Segunda barrera

Si por una ruta interna el Subject ya estuviera cargado:

```text
Invoice#500
tenant=9
```

entonces:

```text
TenantIsolationPolicy
→ DENY
```

---

# 336. Defense in Depth Result

Dos mecanismos independientes protegen la frontera:

```text
Data Scoping
+
Authorization Isolation
```

---

# 337. Ejemplo — Platform Support

Actor:

```text
SupportAgent#10
```

Global Permission:

```text
platform.tenant.support
```

Target:

```text
Tenant#7
```

Support Session:

```text
ticket=SUP-8821
expires=30 minutes
MFA=true
```

---

# 338. Plan

```text
PrincipalStatePolicy
→ GRANT

MFAEvaluator
→ GRANT

CrossTenantAccessPolicy
→ GRANT

SupportSessionPolicy
→ GRANT

Target Tenant Status
→ GRANT

Resource Policy
→ evaluated under restricted support capabilities
```

No existe:

```text
skipTenantAuthorization()
```

---

# 339. Filosofía del sistema

La filosofía definitiva será:

```text
Resolve the tenant explicitly.

Validate that the Principal may operate there.

Scope data before it becomes a Subject.

Verify ownership again at authorization boundaries.

Scope Roles and Permissions to the tenant.

Treat cross-tenant access as privileged behavior.

Never interpret missing tenant context as global authority.

Propagate tenant identity through every runtime boundary.

Clean tenant state after every request, job and command.

Use multiple independent layers to protect tenant data.
```

---

# 340. Resultado esperado

El `Multi-Tenant Authorization and Data Isolation System` permitirá que VoltStack soporte desde aplicaciones SaaS simples con:

```text
shared database
+
tenant_id
```

hasta plataformas empresariales con:

```text
multiple databases
dedicated tenant infrastructure
global administrators
support impersonation
tenant-scoped RBAC
ReBAC sharing
background workers
distributed services
persistent FrankenPHP workers
```

manteniendo una frontera coherente de seguridad.

La arquitectura final será:

```text
Principal
   ↓
Tenant Context
   ↓
Membership
   ↓
Tenant Scope
   ↓
Data Isolation
   ↓
Subject Ownership
   ↓
TenantIsolationPolicy
   ↓
RBAC / ABAC / ReBAC
   ↓
Resource Policy
   ↓
DecisionManager
   ↓
Tenant-Safe Operation
```

El principio central de VoltStack será:

```text
Authorization determines whether an operation is allowed.

Tenant isolation determines the security boundary
inside which that operation may exist.

Neither one replaces the other.

VoltStack enforces both.
```

Con esta arquitectura, el sistema de autorización no tratará multi-tenancy como un simple `tenant_id` añadido a una Policy, sino como una propiedad transversal del runtime, Routing, Database, Cache, Queue, Events y Authorization Engine.