# VoltStack Authorization System — Delegation, Impersonation, Capabilities and Service-to-Service Authorization System

## 1. Propósito

Este documento define la arquitectura de **delegación de autorización, impersonación, capabilities temporales y autorización service-to-service** dentro del Authorization System de VoltStack.

El objetivo es permitir escenarios como:

```text
Usuario A delega una operación a Usuario B

Administrador de soporte actúa temporalmente como otro usuario

Un Job continúa una operación iniciada por un usuario

Un servicio backend llama a otro servicio

Una integración recibe autoridad limitada para realizar una acción

Un enlace firmado permite una operación concreta

Un proceso automático actúa con una identidad de máquina

Una autorización puede delegarse de forma limitada, temporal y auditable
```

sin introducir mecanismos como:

```text
isAdmin() => allow everything

trusted internal request => bypass authorization

X-User-ID header => impersonate

service token => unrestricted access

signed URL => permanent permission
```

El principio fundamental será:

```text
Delegation transfers limited authority.

Impersonation changes the effective identity.

Capabilities grant narrowly scoped authority.

Service identities represent machines.

None of these concepts means bypass authorization.
```

---

# 2. Objetivos

El sistema deberá proporcionar:

1. Actor y Effective Principal separados;
2. delegación explícita;
3. authority scope limitado;
4. delegation chains;
5. impersonation segura;
6. impersonation auditada;
7. soporte para service accounts;
8. machine identities;
9. service-to-service authorization;
10. capabilities;
11. bearer capabilities;
12. bound capabilities;
13. temporal grants;
14. single-use grants;
15. delegated jobs;
16. signed operation grants;
17. scope restrictions;
18. tenant restrictions;
19. subject restrictions;
20. ability restrictions;
21. expiration;
22. revocation;
23. purpose binding;
24. audience binding;
25. non-transferability;
26. delegation depth;
27. least privilege;
28. audit provenance;
29. fail-closed behavior;
30. replay protection.

---

# 3. Modelo general

VoltStack deberá distinguir al menos:

```text
Actor

Effective Principal

Authority Source

Authorization Scope

Target Subject

Delegation Chain
```

---

# 4. Actor

El `Actor` representa:

```text
who actually initiated or executed the operation
```

Ejemplo:

```text
SupportAgent#10
```

---

# 5. Effective Principal

Representa:

```text
whose authorization identity
is currently being evaluated
```

Ejemplo durante impersonation:

```text
Actor:
SupportAgent#10

Effective Principal:
User#42
```

---

# 6. Regla fundamental

Nunca perder:

```text
Actor
```

cuando cambia:

```text
Effective Principal
```

---

# 7. PrincipalContext

El Authorization System deberá poder representar ambos.

Conceptualmente:

```php
final readonly class PrincipalContext
{
    public function __construct(
        public PrincipalInterface $actor,
        public PrincipalInterface $effectivePrincipal,
        public ?DelegationContext $delegation = null,
    ) {}
}
```

---

# 8. Caso normal

Sin impersonation:

```text
Actor
=
Effective Principal
```

---

# 9. Caso delegado

Puede existir:

```text
Actor:
User#42

Effective Principal:
ServiceWorker#12

Authority Source:
DelegatedGrant#88
```

según el workflow.

---

# 10. Authority Source

VoltStack deberá saber de dónde proviene la autoridad.

Ejemplos:

```text
Direct Principal Grants
Role
Permission
Delegated Grant
Capability
Service Identity
Support Session
Machine Credential
```

---

# 11. AuthorizationAuthoritySource

Podrá modelarse mediante:

```php
enum AuthorizationAuthoritySource: string
{
    case Direct = 'direct';
    case Role = 'role';
    case Permission = 'permission';
    case Delegation = 'delegation';
    case Capability = 'capability';
    case Impersonation = 'impersonation';
    case ServiceIdentity = 'service_identity';
    case System = 'system';
}
```

---

# 12. Delegación

Delegar significa:

```text
Principal A grants Principal B
limited authority to perform
specific operations.
```

No significa:

```text
Principal B becomes Principal A.
```

---

# 13. Delegation vs Impersonation

Deben mantenerse separados.

```text
Delegation:
B acts as B
using authority delegated by A
```

```text
Impersonation:
B acts as A
while B remains recorded as Actor
```

---

# 14. Preferencia arquitectónica

Cuando sea suficiente:

```text
prefer delegation over impersonation
```

porque conserva mejor la identidad real del ejecutor.

---

# 15. Ejemplo

User#42 delega a Accountant#9:

```text
invoice.view
invoice.export
```

Accountant#9 sigue siendo:

```text
Accountant#9
```

No se convierte en User#42.

---

# 16. DelegatedAuthorizationGrant

Unidad central:

```php
final readonly class DelegatedAuthorizationGrant
{
    public function __construct(
        public string $id,
        public PrincipalReference $grantor,
        public PrincipalReference $grantee,
        public AuthorizationScopeSet $scope,
        public DateTimeImmutable $issuedAt,
        public ?DateTimeImmutable $expiresAt,
    ) {}
}
```

---

# 17. Scope

El grant deberá limitar qué puede hacerse.

---

# 18. AuthorizationScopeSet

Podrá incluir:

```text
Abilities
Tenants
Subjects
Subject Types
Operations
Channels
Purpose
Audience
Time
```

---

# 19. Ability Scope

Ejemplo:

```text
invoice.view
invoice.export
```

No:

```text
invoice.*
```

salvo que se declare explícitamente.

---

# 20. Tenant Scope

Delegación:

```text
Tenant#7 only
```

no deberá funcionar en:

```text
Tenant#9
```

---

# 21. Subject Scope

Puede limitarse a:

```text
Invoice#928
```

---

# 22. Subject-Type Scope

O:

```text
all Invoices in Tenant#7
```

si el grant lo permite.

---

# 23. Purpose

Podrá incluir:

```text
support_case
billing_reconciliation
document_signature
data_export
```

---

# 24. Purpose Binding

Una delegación otorgada para:

```text
billing_reconciliation
```

no debería reutilizarse para:

```text
customer_export
```

si el purpose forma parte del scope.

---

# 25. Audience

Podrá indicar:

```text
only service X
only API Y
only job type Z
```

---

# 26. Audience Binding

Especialmente útil para capabilities y service-to-service.

---

# 27. Expiration

Todo grant temporal deberá soportar:

```text
expires_at
```

---

# 28. Default

Delegaciones privilegiadas deberían:

```text
expire
```

por defecto.

---

# 29. Permanent Delegation

Solo cuando el dominio realmente lo necesite.

---

# 30. Revocation

Todo grant revocable deberá tener:

```text
revocation state
```

---

# 31. DelegationStatus

Ejemplo:

```php
enum DelegationStatus: string
{
    case Active = 'active';
    case Revoked = 'revoked';
    case Expired = 'expired';
    case Consumed = 'consumed';
}
```

---

# 32. Single-Use Delegation

Algunos grants podrán ser:

```text
single_use
```

---

# 33. Ejemplo

```text
Approve Invoice#928 once.
```

---

# 34. Consumption

La autorización y consumo deberán coordinarse cuidadosamente.

---

# 35. Atomicity

Para single-use capabilities:

```text
validate
+
consume
```

deberá realizarse atómicamente cuando sea necesario.

---

# 36. TOCTOU

Evitar:

```text
check capability
wait
use capability again
```

---

# 37. DelegationEvaluator

Podrá existir:

```text
DelegatedGrantEvaluator
```

---

# 38. Flujo

```text
Principal
    ↓
Direct authorization
    ↓
Delegated grants
    ↓
Scope match
    ↓
Grant validity
    ↓
Decision
```

---

# 39. Delegation no necesariamente precede Direct Grants

El Decision Plan podrá combinar:

```text
direct permissions
delegated authority
resource Policy
```

---

# 40. Regla importante

Un DelegatedGrant no deberá saltarse:

```text
TenantIsolation
Subject Policy
NonBypassable evaluators
Compliance
Security constraints
```

salvo que la semántica del sistema lo defina expresamente.

---

# 41. Ejemplo

Delegated grant:

```text
invoice.approve
```

sobre Invoice#928.

Pero Invoice está:

```text
locked
```

Entonces:

```text
InvoicePolicy
→ DENY
```

aunque exista delegation.

---

# 42. Delegation puede otorgar Authority, no invalidar Domain Rules

---

# 43. Grantor Authority

Un Principal solo deberá poder delegar autoridad que posee o que está autorizado a delegar.

---

# 44. No Privilege Creation

Regla:

```text
A delegated grant
must not create more authority
than the grantor is allowed to delegate.
```

---

# 45. Ejemplo

User#42 posee:

```text
invoice.view
```

No podrá delegar:

```text
invoice.delete
```

---

# 46. Delegation Policy

Podrá existir:

```text
DelegationCreationPolicy
```

que determine:

```text
may grantor delegate this Ability?
```

---

# 47. Delegatable Metadata

Abilities podrán declarar:

```text
delegatable=true/false
```

---

# 48. Default

Para abilities críticas:

```text
delegatable=false
```

por defecto puede ser recomendable.

---

# 49. Ejemplos no delegables por default

```text
permission.grant
role.assign
system.deploy
tenant.delete
user.impersonate
```

---

# 50. Delegation Depth

Un grantee podrá o no volver a delegar.

---

# 51. Transitive Delegation

Ejemplo:

```text
A delegates to B
B delegates to C
```

debe ser explícitamente controlado.

---

# 52. Max Delegation Depth

Podrá existir:

```text
max_depth
```

---

# 53. Recommended Default

```text
transitive delegation disabled
```

salvo necesidad clara.

---

# 54. Re-delegation

Si se permite:

```text
child grant scope
⊆
parent grant scope
```

---

# 55. Scope Narrowing

Cada nueva delegación solo podrá:

```text
preserve or reduce authority
```

nunca ampliarla.

---

# 56. Delegation Chain

Podrá registrarse:

```text
A
→ B
→ C
```

---

# 57. AuthorizationDelegationChain

Conceptualmente:

```php
final readonly class AuthorizationDelegationChain
{
    public function __construct(
        public array $grants
    ) {}
}
```

---

# 58. Chain Validation

Debe verificar:

```text
every grant active
scope continuity
depth limit
tenant consistency
audience consistency
```

---

# 59. Chain Revocation

Si un grant intermedio se revoca:

```text
all descendant authority
must become invalid
```

salvo que los descendientes tengan otra fuente independiente.

---

# 60. Versioning

Puede utilizarse:

```text
delegation_version
```

o revision tokens para cache invalidation.

---

# 61. Impersonation

Impersonation deberá considerarse una operación privilegiada.

---

# 62. ImpersonationSession

Unidad central:

```php
final readonly class ImpersonationSession
{
    public function __construct(
        public string $id,
        public PrincipalReference $actor,
        public PrincipalReference $effectivePrincipal,
        public AuthorizationScopeSet $scope,
        public DateTimeImmutable $startedAt,
        public DateTimeImmutable $expiresAt,
    ) {}
}
```

---

# 63. Impersonation Authorization

Antes de iniciar:

```text
Authorization::authorize(
    'user.impersonate',
    targetUser
)
```

---

# 64. Target Restrictions

No todo usuario impersonable por todo administrador.

Podrán existir reglas como:

```text
same tenant
support ticket exists
target not platform admin
target not security operator
MFA required
```

---

# 65. Scope-Limited Impersonation

Impersonation no tiene por qué entregar todas las capacidades del target.

---

# 66. Ejemplo

Support agent impersona User#42 con:

```text
view
update profile
troubleshoot application
```

pero no:

```text
change password
create API keys
change billing
delete tenant
grant roles
```

---

# 67. Impersonation Restriction Evaluator

Podrá existir:

```text
ImpersonationRestrictionEvaluator
```

---

# 68. Mandatory

Cuando existe impersonation:

```text
always execute
```

antes de concluir GRANT.

---

# 69. NonBypassable

Recomendado:

```text
nonBypassable=true
```

---

# 70. Actor Permissions

La autorización durante impersonation podrá depender de:

```text
effective principal permissions
+
actor restrictions
```

---

# 71. Ejemplo

User#42 puede:

```text
invoice.delete
```

Pero SupportAgent#10 está impersonando.

La restriction Policy dice:

```text
destructive operations forbidden during support impersonation
```

Resultado:

```text
DENY
```

---

# 72. Actor-aware Policies

El AuthorizationContext deberá permitir consultar:

```text
actor
effective principal
```

de forma explícita.

---

# 73. No Hidden `currentUser()`

Policies no deberán perder quién actúa realmente.

---

# 74. Impersonation Nesting

Por defecto:

```text
impersonation inside impersonation
```

deberá estar prohibida.

---

# 75. Reason

Evita cadenas difíciles de auditar y analizar.

---

# 76. Optional Nested Impersonation

Si algún sistema lo requiere:

```text
max_depth=1/2
```

y chain audit completo.

---

# 77. Impersonation End

Debe existir una operación explícita:

```text
stop impersonation
```

---

# 78. Automatic Expiration

Además:

```text
expires_at
```

---

# 79. Session Revocation

Un security operator deberá poder invalidarla.

---

# 80. Browser Session

No guardar simplemente:

```text
impersonated_user_id
```

sin metadata segura.

---

# 81. Impersonation Token

Puede usarse un reference/token seguro asociado a:

```text
session ID
actor
target
scope
expiry
```

---

# 82. Token Rotation

Al iniciar o terminar impersonation podría ser necesario:

```text
session rotation
CSRF state rotation
```

según Authentication subsystem.

---

# 83. Impersonation UI

El frontend debería recibir:

```text
impersonation=true
actor display identity
effective user display identity
expires_at
```

para mostrar estado claro.

---

# 84. Security UX

Una interfaz impersonada debería ser visualmente distinguible.

---

# 85. Backend Remains Authority

La UI no impone restricciones de seguridad.

---

# 86. Audit

Toda operación durante impersonation deberá poder registrar:

```text
Actor
Effective Principal
Impersonation Session
Tenant
Ability
Subject
Decision
```

---

# 87. Example

```text
SupportAgent#10
performed invoice.view
as User#42
inside Tenant#7
session=imp_88
```

---

# 88. Capability-Based Authorization

Una capability representa:

```text
authority encoded or referenced
as a specific token/grant
```

---

# 89. Core Principle

Una capability debe ser:

```text
narrowly scoped
verifiable
limited
revocable when required
auditable
```

---

# 90. Capability Types

VoltStack podrá soportar:

```text
Reference Capability
Bearer Capability
Bound Capability
Single-Use Capability
Delegated Capability
Signed Capability
```

---

# 91. Reference Capability

Token contiene un ID opaco:

```text
cap_7x9...
```

y el estado real vive en servidor.

---

# 92. Advantages

```text
easy revocation
small token
server-side state
```

---

# 93. Disadvantages

```text
requires lookup
```

---

# 94. Signed Capability

El token puede contener claims firmados.

---

# 95. Claims

Ejemplo:

```text
issuer
subject/grantee
ability
tenant
resource
audience
issued_at
expires_at
nonce
```

---

# 96. Important

Firma valida integridad.

No necesariamente valida:

```text
revocation
current resource state
current tenant status
```

---

# 97. Signed Capability Evaluator

Deberá seguir aplicando:

```text
signature validation
expiry
audience
scope
replay protection
revocation if supported
resource policy
tenant isolation
```

---

# 98. Bearer Capability

Quien posee el token puede usarlo.

---

# 99. Risk

Si se filtra:

```text
authority transfers to attacker
```

---

# 100. Recommendation

Para capacidades sensibles, preferir:

```text
bound capability
```

---

# 101. Bound Capability

Puede estar ligada a:

```text
Principal
service identity
device
session
public key
tenant
```

---

# 102. Example

```text
Capability#88
bound to:
User#42
Tenant#7
```

---

# 103. CapabilityScope

Podrá definir:

```php
final readonly class CapabilityScope
{
    public function __construct(
        public array $abilities,
        public ?TenantReference $tenant = null,
        public ?SubjectReference $subject = null,
        public ?string $audience = null,
        public ?string $purpose = null,
    ) {}
}
```

---

# 104. Capability Grant

Una capability no debería ser:

```text
boolean magical allow
```

Sino una fuente de authority dentro del pipeline.

---

# 105. Pipeline

```text
Capability
    ↓
Validate
    ↓
Normalize Authority
    ↓
CapabilityEvaluator
    ↓
DecisionManager
```

---

# 106. Capability + Policy

Ejemplo:

Capability permite:

```text
document.download
Document#88
```

Pero Document fue archivado y ya no descargable.

Resource Policy puede:

```text
DENY
```

---

# 107. Single-Use Capability

Ejemplos:

```text
passwordless login link
approve operation once
accept invitation
download once
```

---

# 108. Replay Protection

Debe existir:

```text
nonce
consumption state
```

cuando corresponda.

---

# 109. Expiration

Single-use sin expiración puede ser peligroso.

---

# 110. Idempotency Consideration

Una capability single-use y una operación idempotente deberán coordinar:

```text
capability consumed
operation committed
```

---

# 111. Transactional Consumption

Idealmente:

```text
BEGIN
validate capability
consume capability
execute operation
COMMIT
```

cuando ambos vivan en la misma transacción lógica.

---

# 112. Distributed Case

Puede requerir:

```text
distributed lock
idempotency key
atomic capability store
```

---

# 113. Capability Revocation

Reference capabilities:

```text
easy
```

Signed stateless capabilities:

```text
requires revocation list/version
```

si revocación anticipada es necesaria.

---

# 114. Capability Version

Puede incluir:

```text
security_version
```

del issuer/grantee/tenant.

---

# 115. Audience

Capabilities service-to-service deberán incluir:

```text
aud
```

o equivalente conceptual.

---

# 116. Why

Un token emitido para:

```text
DocumentService
```

no debe poder usarse en:

```text
BillingService
```

---

# 117. Service Identities

VoltStack deberá modelar máquinas como Principals reales.

---

# 118. ServicePrincipal

Conceptualmente:

```php
final readonly class ServicePrincipal
    implements PrincipalInterface
{
    public function __construct(
        public string $serviceId,
    ) {}
}
```

---

# 119. Machine Principal

También podría existir:

```text
MachinePrincipal
WorkloadPrincipal
```

si el dominio requiere distinguirlos.

---

# 120. Service Account

Debe ser un Principal con:

```text
Roles
Permissions
Scopes
Tenant access
```

como cualquier otro.

---

# 121. No Internal Bypass

Nunca:

```php
if ($request->isInternal()) {
    return true;
}
```

---

# 122. Internal Network Is Not Authorization

Estar dentro de:

```text
private VPC
localhost
internal DNS
```

no otorga automáticamente autoridad.

---

# 123. Service Authentication

Primero debe autenticarse la identidad del servicio.

Ejemplos conceptuales:

```text
mTLS
signed workload identity
OAuth client credential
service token
platform-issued workload token
```

---

# 124. Separation

```text
Authentication
→ proves service identity
```

```text
Authorization
→ determines allowed operations
```

---

# 125. Service-to-Service Authorization Flow

```text
Service A
    ↓
Authenticate Service A
    ↓
ServicePrincipal(A)
    ↓
Resolve Tenant/Delegation Context
    ↓
Authorize Ability
    ↓
Service B operation
```

---

# 126. Example

```text
ReportService
calls
DocumentService
```

Ability:

```text
document.read_for_report
```

---

# 127. Better Than

```text
document.read.all
```

cuando solo necesita una función concreta.

---

# 128. Service Permissions

Deberán seguir least privilege.

---

# 129. Examples

```text
billing.invoice.read
billing.invoice.reconcile
email.message.send
storage.object.read
search.document.index
```

---

# 130. Service Roles

Podrán existir:

```text
report-generator
billing-reconciler
search-indexer
```

---

# 131. SystemPrincipal vs ServicePrincipal

Deben distinguirse.

---

# 132. ServicePrincipal

Representa:

```text
application/service identity
```

---

# 133. SystemPrincipal

Representa:

```text
trusted internal framework/platform operation
```

con capacidades muy restringidas.

---

# 134. No Omnipotent System Principal

Ambos deben poseer abilities explícitas.

---

# 135. Workload Instance

Una identidad puede representar:

```text
service logical identity
```

o:

```text
individual workload instance
```

---

# 136. Example

```text
service=report-service
instance=pod-18
```

---

# 137. Audit

Normalmente interesa al menos:

```text
service identity
```

y opcionalmente:

```text
workload instance
```

---

# 138. Service Tenant Scope

Un servicio puede estar limitado a:

```text
Tenant#7
```

---

# 139. Multi-Tenant Service

Puede necesitar múltiples tenants.

Debe ser explícito.

---

# 140. Example

Background billing service:

```text
Global service principal
```

pero ability:

```text
tenant.billing.reconcile
```

se ejecuta dentro de un TenantContext concreto.

---

# 141. Preferred Pattern

```text
Global machine authority
+
explicit target Tenant
+
Tenant-specific operation
```

---

# 142. Avoid

```text
global service token
+
unrestricted database queries
```

---

# 143. Service Delegation

Una request iniciada por usuario puede propagarse a otro servicio.

---

# 144. Important Design Choice

Service B puede autorizar:

```text
Service A as actor
```

o:

```text
Original User authority delegated through Service A
```

---

# 145. These Are Different

---

# 146. Service Authority

Ejemplo:

```text
SearchIndexer service
```

actúa por su propia identidad.

---

# 147. User Delegation

Ejemplo:

```text
DocumentExportService
```

realiza export solicitado por User#42 bajo autoridad delegada.

---

# 148. Delegated Service Context

Podrá contener:

```text
Original Actor
Service Actor
Effective Principal
Delegated Grant
Tenant
```

---

# 149. Example

```text
Original Actor:
User#42

Current Actor:
ExportService

Effective Authority:
DelegatedGrant#88

Tenant:
7
```

---

# 150. Authority Propagation

No propagar simplemente:

```text
X-User-ID: 42
X-Role: admin
```

---

# 151. Signed Delegation Envelope

Podrá existir:

```text
DelegatedAuthorizationEnvelope
```

---

# 152. Contents

Ejemplo:

```text
issuer service
original principal
delegated abilities
tenant
subject restrictions
audience
expires_at
nonce
correlation_id
```

---

# 153. Verification

Service receptor deberá validar:

```text
issuer trust
signature
audience
expiry
scope
revocation/version
```

---

# 154. Still Authorize Locally

Después:

```text
Delegated envelope
```

se convierte en authority source dentro del Authorization Core.

---

# 155. No Trusting Upstream Final GRANT Alone

Evitar:

```text
Service A says "authorized=true"
```

como única evidencia.

---

# 156. Why

Service B puede tener:

```text
different resource state
local compliance rules
different tenant boundary
```

---

# 157. Better

Propagar:

```text
who
what authority
for what
under what scope
```

---

# 158. Service Audience Binding

Envelope para:

```text
DocumentService
```

no válido para:

```text
BillingService
```

---

# 159. Hop Limiting

Puede declararse:

```text
max_hops
```

---

# 160. Delegation Chain Across Services

```text
User
→ API Gateway
→ Export Service
→ Storage Service
```

debe conservar provenance.

---

# 161. No Unlimited Forwarding

Cada hop deberá estar:

```text
allowed by grant
```

---

# 162. Delegation Hop

Podrá generar una nueva delegation limitada derivada de la anterior.

---

# 163. Scope Must Narrow

```text
Storage Service grant
⊆
Export Service grant
⊆
Original grant
```

---

# 164. Example

Original:

```text
report.export
Document#88
```

Derived for Storage:

```text
storage.object.read
Object#555
```

---

# 165. Capability Translation

Un servicio podrá traducir autoridad de dominio a una capability más estrecha.

---

# 166. Example

```text
report.export
```

genera temporalmente:

```text
storage.read object:555
```

---

# 167. Capability Minting

Debe ser una operación autorizada.

---

# 168. CapabilityIssuer

Contrato conceptual:

```php
interface CapabilityIssuerInterface
{
    public function issue(
        CapabilityIssuanceRequest $request
    ): Capability;
}
```

---

# 169. Issuance Policy

Antes:

```text
CapabilityIssuancePolicy
```

verifica que issuer puede emitir ese scope.

---

# 170. No Scope Escalation

Issued capability:

```text
scope
```

debe ser ≤ autoridad del issuer y propósito del workflow.

---

# 171. Capability Issuer Identity

El token deberá registrar:

```text
issuer
```

---

# 172. Service Credentials

Service credentials deberán poder rotarse sin alterar el Principal lógico.

---

# 173. Example

```text
ServicePrincipal:
report-service

Credential:
key-v18
```

---

# 174. Credential vs Principal

No deben ser lo mismo.

---

# 175. Credential Rotation

No debería requerir:

```text
reassign all service permissions
```

---

# 176. Credential Revocation

Invalidar credencial comprometida sin eliminar service identity.

---

# 177. Multiple Credentials

Puede existir temporalmente durante rotation.

---

# 178. Machine Authentication Metadata

SecurityContext puede registrar:

```text
credential type
credential ID/version
authentication strength
```

---

# 179. Authorization may depend on Authentication Strength

Ejemplo:

```text
sensitive service operation
requires mTLS workload identity
```

---

# 180. Token Scope

Un OAuth-style service token puede incluir scopes.

---

# 181. Important

Token scope debe ser:

```text
upper bound
```

sobre autoridad.

---

# 182. Token Scope ≠ Permission Grant Alone

Effective authorization:

```text
Token Scope
AND
Service Permission
AND
Tenant Scope
AND
Resource Policy
```

---

# 183. Example

Service has permission:

```text
invoice.read
```

Token only has:

```text
invoice.metadata.read
```

Entonces:

```text
effective authority limited by token
```

---

# 184. Scope Intersection

El sistema deberá combinar autoridad mediante:

```text
intersection
```

cuando un credential scope limita un Principal.

---

# 185. Authority Ceiling

Podrá modelarse:

```text
AuthorizationAuthorityCeiling
```

---

# 186. Sources of Ceiling

```text
token scopes
delegated grants
capability scope
impersonation restrictions
tenant context
```

---

# 187. Direct Permissions

No pueden superar esos ceilings.

---

# 188. Example

User has:

```text
invoice.*
```

Delegated grant only:

```text
invoice.view
```

Effective delegated authority:

```text
invoice.view
```

---

# 189. Delegation Cannot Inherit Unrelated Direct Grants

Si una service operation debe actuar solo bajo delegated authority:

```text
ambient service permissions
```

no deberían ampliar la operación accidentalmente.

---

# 190. Authority Mode

Podrá existir:

```php
enum AuthorizationAuthorityMode: string
{
    case Native = 'native';
    case Delegated = 'delegated';
    case Capability = 'capability';
    case Impersonated = 'impersonated';
}
```

---

# 191. Delegated Mode

Puede indicar:

```text
evaluate within delegated ceiling
```

---

# 192. No Ambient Privilege Escalation

Importantísimo para service-to-service.

---

# 193. Example

ExportService posee globalmente:

```text
storage.read.all
```

por otra función interna.

Pero un export de usuario delegado debe limitarse a:

```text
Document#88
```

No aprovechar accidentalmente `storage.read.all`.

---

# 194. Execution Authority Context

Podrá contener:

```php
final readonly class AuthorizationAuthorityContext
{
    public function __construct(
        public AuthorizationAuthorityMode $mode,
        public AuthorizationAuthoritySet $authority,
        public ?AuthorizationAuthorityCeiling $ceiling,
    ) {}
}
```

---

# 195. Authority Combination

La lógica deberá ser explícita.

---

# 196. Native Mode

```text
Principal effective grants
```

---

# 197. Delegated Mode

```text
Principal/runtime grants
INTERSECT
delegated scope
```

o:

```text
delegated scope only
```

según el modelo definido.

---

# 198. Recommendation

Para delegation real:

```text
delegated authority should be explicit,
not mixed accidentally with ambient authority.
```

---

# 199. Service Jobs

Jobs asíncronos son un caso clave.

---

# 200. Problem

User solicita:

```text
export report
```

Job se ejecuta 10 minutos después.

¿Qué authority utiliza?

---

# 201. Options

```text
Reauthorize current user at execution time

Use delegated grant

Use service authority

Use capability
```

---

# 202. Default Recommendation

Cuando el Job representa una acción del usuario:

```text
reauthorize at execution
```

o usar:

```text
explicit limited delegated grant
```

---

# 203. Do Not Serialize Entire User Permission Snapshot

Evitar:

```text
roles at dispatch time
permissions at dispatch time
```

como autoridad permanente.

---

# 204. Why

Podrían ser revocados antes de ejecución.

---

# 205. Delegated Job Grant

Puede incluir:

```text
ability
subject
tenant
expires_at
issuer
```

---

# 206. Execution

```text
Job starts
    ↓
load grant
    ↓
verify active
    ↓
restore TenantContext
    ↓
authorize under delegated authority
    ↓
execute
```

---

# 207. User Revocation Semantics

La aplicación deberá definir si revocar al usuario invalida delegation existente.

---

# 208. Recommended Sensitive Default

Sí:

```text
grant linked to grantor authorization version
```

---

# 209. Grantor Version

Si User#42 pierde permiso:

```text
authorization_version changes
```

Delegated grant puede invalidarse.

---

# 210. Independent Delegation

Algunos grants legales/contractuales pueden sobrevivir cambios del grantor.

Debe declararse explícitamente.

---

# 211. Delegation Consistency Policy

Podrá existir:

```php
enum DelegationConsistencyMode: string
{
    case LiveGrantorAuthority = 'live_grantor_authority';
    case SnapshotUntilExpiry = 'snapshot_until_expiry';
    case ExplicitRevocation = 'explicit_revocation';
}
```

---

# 212. Default

Para operaciones normales:

```text
LiveGrantorAuthority
```

es más seguro.

---

# 213. Snapshot Authority

Debe usarse con cautela.

---

# 214. Example Legitimate Snapshot

Aprobación formal delegada por un workflow que debe seguir vigente incluso si el usuario cambia de departamento, según reglas del dominio.

---

# 215. Security Requires Explicit Semantics

Nunca inferir.

---

# 216. Human-to-Service Delegation

Ejemplo:

```text
User#42
→ ExportService
```

---

# 217. Service-to-Service Delegation

```text
ExportService
→ StorageService
```

---

# 218. Service-to-Human Delegation

Raro, pero posible en workflows de tareas asignadas.

---

# 219. Principal Types

Authorization Core deberá poder manejar:

```text
HumanPrincipal
ServicePrincipal
SystemPrincipal
AnonymousPrincipal
```

sin asumir que todo Principal es `User`.

---

# 220. Principal Type Policies

Abilities podrán restringir:

```text
allowed principal types
```

---

# 221. Example

```text
user.impersonate
```

solo:

```text
HumanPrincipal
```

---

# 222. Example

```text
system.cache.rebuild
```

solo:

```text
ServicePrincipal/SystemPrincipal
```

---

# 223. PrincipalTypeEvaluator

Puede ejecutarse temprano.

---

# 224. Machine-only Operations

Ayuda a evitar que cuentas humanas invoquen APIs internas por accidente.

---

# 225. Human-only Operations

Evita que un service account ejecute ciertas acciones sensibles.

---

# 226. Session-Based Capability

Una capability podrá estar ligada a:

```text
current authenticated session
```

---

# 227. Example

MFA elevation:

```text
temporary sensitive-operation capability
```

---

# 228. SecurityContext

Podrá llevar:

```text
elevated until 15:30
```

---

# 229. Step-Up Authorization

Cuando una ability requiere más assurance:

```text
DENY / challenge
security.mfa_required
```

---

# 230. After MFA

Authentication subsystem puede emitir:

```text
elevated capability
```

o actualizar SecurityContext.

---

# 231. Capability Scope

Ejemplo:

```text
wire.transfer
Account#88
valid 5 minutes
```

---

# 232. Better Than Global MFA Flag

Permite step-up específico.

---

# 233. Challenge Is Not GRANT

Authorization puede indicar:

```text
additional_authentication_required
```

pero operación sigue bloqueada hasta completar challenge.

---

# 234. API Keys

API keys deberán representar credenciales de un Principal.

---

# 235. API Key Principal

Puede pertenecer a:

```text
User
Service Account
Integration
Tenant
```

---

# 236. API Key Scopes

Actúan como authority ceiling.

---

# 237. API Key Rotation

Credential lifecycle separado de authorization grants.

---

# 238. API Key Revocation

Debe invalidar authentication inmediatamente según consistency contract.

---

# 239. Personal Access Tokens

Mismo principio.

---

# 240. Token Scope Evaluation

```text
Principal Permission
AND
Token Scope
```

---

# 241. Never

```text
Token says admin
→ grant
```

sin verificar issuer y current Principal authorization model.

---

# 242. Delegated API Token

Un token puede representar:

```text
grantee + delegated authority
```

---

# 243. Token Claims

No deberían duplicar arbitrariamente toda Policy lógica.

---

# 244. Prefer References

Para authority compleja:

```text
delegation_id
capability_id
```

puede ser mejor que codificar cientos de rules dentro del token.

---

# 245. Revocation vs Statelessness Tradeoff

Reference token:

```text
better revocation
```

Signed self-contained token:

```text
lower lookup cost
```

---

# 246. Authorization Consistency Wins

Critical capabilities deberán favorecer revocation correcta.

---

# 247. Signed URLs

Signed URLs deberán tratarse como capabilities.

---

# 248. Example

```text
/download/928?signature=...
```

---

# 249. Must Bind

Firma deberá cubrir al menos:

```text
resource
operation
expiry
```

y cuando aplique:

```text
tenant
principal
audience
```

---

# 250. Avoid Generic Signed URL

No firmar solo:

```text
path
```

si parámetros relevantes pueden cambiarse.

---

# 251. Resource State

Una URL firmada válida no obliga a servir un recurso que ahora está prohibido.

---

# 252. Resource Policy Still Applies

Según capability semantics.

---

# 253. Public Share Links

Podrán estar diseñados para AnonymousPrincipal.

---

# 254. Example

```text
Document share capability
```

puede permitir:

```text
document.view_shared
```

sin login.

---

# 255. Capability becomes Principal Authority Source

No una bypass route.

---

# 256. Tenant Isolation

Share link deberá estar ligado al Tenant del resource internamente.

---

# 257. Enumeration

No exponer raw sequential IDs si el dominio requiere concealment.

---

# 258. Capability Identifier

Debe ser de entropía adecuada si bearer.

---

# 259. Secret Storage

Reference bearer capabilities deberán almacenarse de forma segura.

---

# 260. Hash Bearer Tokens

Como API keys, puede almacenarse:

```text
hash
```

en vez del token plano.

---

# 261. Capability Presentation

El raw token no debe aparecer en:

```text
logs
audit
traces
exceptions
```

---

# 262. Audit Reference

Guardar:

```text
capability ID
```

no token secreto.

---

# 263. Capability Leakage

Debe considerarse security incident si otorga autoridad significativa.

---

# 264. Revocation API

Deberá estar autorizada:

```text
capability.revoke
```

---

# 265. Delegation Creation API

También:

```text
authorization.delegate
```

o ability específica.

---

# 266. Granular Delegation Abilities

Ejemplo:

```text
invoice.delegate_view
invoice.delegate_approval
```

podría ser más seguro que un genérico.

---

# 267. Policy Metadata

Ability descriptor podrá declarar:

```text
delegatable
impersonationAllowed
capabilityIssuable
serviceCallable
```

---

# 268. Example

```text
invoice.view
delegatable=true
capabilityIssuable=true
serviceCallable=true
```

---

# 269. Critical Ability

```text
permission.grant
delegatable=false
capabilityIssuable=false
impersonationAllowed=false
```

---

# 270. Compiler Validation

Podrá detectar incompatibilidades.

---

# 271. Example

Route intenta emitir capability para:

```text
system.deploy
```

pero Ability descriptor:

```text
capabilityIssuable=false
```

Compilation/config validation falla.

---

# 272. Authorization Planner Integration

Cuando hay delegation/capability:

```text
AuthorityContextEvaluator
```

deberá ejecutarse temprano.

---

# 273. Proposed Pipeline

```text
PrincipalState
    ↓
Actor/EffectivePrincipal Validation
    ↓
AuthorityMode Validation
    ↓
Delegation/Capability Validation
    ↓
Tenant Context
    ↓
Permission/RBAC
    ↓
Subject Isolation
    ↓
Resource Policy
    ↓
Impersonation Restrictions
    ↓
Decision
```

---

# 274. Authority Context Evaluator

Debe verificar:

```text
scope
expiry
revocation
audience
purpose
delegation depth
```

---

# 275. NonBypassable

Para delegated/capability mode:

```text
recommended
```

---

# 276. Impersonation Restriction

También.

---

# 277. Scope Intersection

Effective requested operation deberá satisfacer:

```text
requested ability ∈ authority scope
```

---

# 278. Subject Matching

Si scope está ligado a:

```text
Invoice#928
```

entonces Invoice#929:

```text
DENY
```

---

# 279. Subject Type Matching

Si:

```text
Invoice
```

solo resources de ese tipo.

---

# 280. Tenant Matching

Delegation Tenant#7 sobre Tenant#9:

```text
DENY
```

---

# 281. Purpose Matching

Mismatch:

```text
DENY
```

o failure si context requerido falta por bug.

---

# 282. Expired Grant

Esto normalmente es:

```text
DENY
```

porque grant ya no otorga autoridad.

---

# 283. Invalid Signature

En cambio:

```text
FAILURE / invalid credential
```

según dónde ocurra y cómo se modele Authentication.

---

# 284. Unknown Capability

Puede mapearse a:

```text
DENY / not found
```

para concealment.

---

# 285. Revoked Capability

```text
DENY
```

---

# 286. Replay Detected

Puede producir:

```text
DENY
+
security event
```

---

# 287. Delegation Provider Failure

Si el authoritative grant store está caído:

```text
FAILURE
→ fail closed
```

---

# 288. Cache

Delegation validation puede memoizarse dentro del request.

---

# 289. Cross-Request Cache

Solo con:

```text
revision/version
expiry
revocation semantics
```

adecuados.

---

# 290. Impersonation Cache

Request-local normalmente.

---

# 291. Service Permission Cache

Puede utilizar:

```text
service authorization version
```

como Principals humanos.

---

# 292. Capability Cache

Reference capability lookup puede cachearse brevemente si revocation consistency lo permite.

---

# 293. Single-Use Capability

No debe tener un cache que permita múltiples consumos.

---

# 294. Audit Requirements

Delegation creation:

```text
audit
```

---

# 295. Delegation Use

Abilities sensibles:

```text
audit actor + grantor + grantee + delegation ID
```

---

# 296. Impersonation Start/Stop

Siempre debería auditarse.

---

# 297. Capability Issuance

Sensitive capability minting:

```text
audit
```

---

# 298. Capability Use

Dependerá de risk level.

---

# 299. Service-to-Service Calls

Critical calls podrán registrar:

```text
calling service
effective user if delegated
target service
ability
tenant
```

---

# 300. Trace Context

Correlation IDs deberán propagarse.

---

# 301. Provenance

AuthorizationExplanation podrá indicar:

```text
authority granted via DelegatedGrant#88
```

para operadores autorizados.

---

# 302. End User Explanation

No necesariamente revelar internal delegation IDs.

---

# 303. Reason Codes

Propuestos:

```text
delegation.missing
delegation.expired
delegation.revoked
delegation.scope_mismatch
delegation.audience_mismatch
delegation.depth_exceeded
delegation.redelegation_forbidden

impersonation.not_allowed
impersonation.expired
impersonation.operation_restricted
impersonation.nesting_forbidden

capability.invalid
capability.expired
capability.revoked
capability.consumed
capability.scope_mismatch
capability.audience_mismatch
capability.replay_detected

service.permission_missing
service.audience_mismatch
service.delegation_required
```

---

# 304. Public Mapping

Muchos podrán mapear a:

```text
forbidden
invalid_link
link_expired
authorization_unavailable
```

sin revelar detalles internos.

---

# 305. Delegation Repository

Contrato:

```php
interface DelegatedAuthorizationGrantRepositoryInterface
{
    public function find(
        string $grantId
    ): ?DelegatedAuthorizationGrant;
}
```

---

# 306. Revocation

```php
public function revoke(
    string $grantId,
    RevocationContext $context
): void;
```

podría pertenecer a un manager separado.

---

# 307. DelegationManager

```php
interface DelegationManagerInterface
{
    public function create(
        DelegationRequest $request
    ): DelegatedAuthorizationGrant;

    public function revoke(
        DelegatedAuthorizationGrant $grant
    ): void;
}
```

---

# 308. Capability Repository

Para reference capabilities:

```php
interface CapabilityRepositoryInterface
{
    public function findByPresentedToken(
        string $token
    ): ?Capability;
}
```

---

# 309. Better Security API

Preferir que repository reciba:

```text
hashed token
```

internamente.

---

# 310. CapabilityManager

Responsabilidades:

```text
issue
validate
revoke
consume
```

---

# 311. Do Not Mix Authentication Secrets into Authorization Objects

El raw secret deberá manejarse en un layer seguro.

---

# 312. ServiceIdentityProvider

Puede resolver:

```text
credential identity
→ ServicePrincipal
```

pero eso pertenece principalmente a Authentication.

---

# 313. Authorization Receives Principal

No debería verificar certificados directamente salvo adapter específico.

---

# 314. Integration Boundary

```text
Security/Authentication
    ↓
ServicePrincipal
    ↓
Authorization
```

---

# 315. Multi-Tenant Service-to-Service

Request interna deberá llevar TenantContext confiable.

---

# 316. Never Trust

```text
X-Tenant-ID
```

solo porque viene de otro servicio.

---

# 317. Trusted Propagation

Tenant context deberá estar:

```text
authenticated/signed/bound
```

por la infraestructura adecuada.

---

# 318. Revalidate Target Tenant

El servicio receptor debe confirmar que la authority tiene scope para ese Tenant.

---

# 319. Service Token with Tenant List

Puede contener:

```text
allowed tenant scope
```

pero para grandes sistemas quizá sea mejor referencia a policy data.

---

# 320. Cross-Tenant Service

Deberá usar global service permissions explícitas.

---

# 321. Example

```text
platform.tenant.backup
```

---

# 322. Backup Worker

ServicePrincipal:

```text
backup-service
```

con:

```text
platform.tenant.backup
```

pero cada operación aún establece:

```text
Target Tenant
```

---

# 323. No Human Impersonation Needed

Machine workflows deberían usar service identity, no fingir ser un usuario humano salvo que el workflow realmente represente autoridad humana delegada.

---

# 324. Principle

```text
Use the identity that actually acts.
```

---

# 325. Delegated Human Workflow

Cuando sí representa al usuario:

```text
record both service actor
and original human authority.
```

---

# 326. Event Propagation

Eventos async podrán transportar:

```text
ActorReference
EffectivePrincipalReference
AuthorityContextReference
TenantReference
```

según necesidad.

---

# 327. Avoid Full Grant Serialization

Preferir:

```text
delegation_id
```

o signed envelope limitado.

---

# 328. Event Consumer

Debe:

```text
revalidate authority
```

si el consistency model lo requiere.

---

# 329. Durable Long-Running Workflows

Una saga que dura días no debería depender de un token de 5 minutos.

---

# 330. Workflow Authority

Puede usar:

```text
workflow-specific delegation grant
```

con scope y lifecycle propios.

---

# 331. Workflow Principal

Incluso podría existir:

```text
WorkflowPrincipal
```

si el dominio lo justifica.

---

# 332. But Avoid Principal Explosion

Preferir ServicePrincipal + DelegationContext cuando sea suficiente.

---

# 333. Scheduled Work

Scheduled automation puede tener:

```text
ServicePrincipal
```

y no requerir usuario.

---

# 334. User-Created Automation

Si una automatización representa una autorización del usuario:

```text
delegated grant
```

puede ser más correcto.

---

# 335. Example

User crea regla:

```text
send monthly report
```

Scheduler actúa como ServicePrincipal usando:

```text
DelegatedGrant from User
```

---

# 336. Revocation

Si usuario revoca automatización:

```text
delegation revoked
```

---

# 337. Ownership Change

Si resource cambia de Tenant/owner:

```text
resource Policy
```

deberá impedir uso stale del grant.

---

# 338. Delegation Scope by Query

No se recomienda delegar:

```text
all resources matching arbitrary user query
```

sin canonicalizar scope.

---

# 339. Better

Scope estructurado:

```text
subject_type=Invoice
tenant=7
ability=view
```

---

# 340. Dynamic Predicate Capabilities

Podrían existir en sistemas avanzados, pero complican:

```text
serialization
validation
audit
replay
```

---

# 341. V1 Recommendation

Scope basado en:

```text
Abilities
Tenant
Subject IDs
Subject Types
Audience
Purpose
Time
```

---

# 342. Capability Conditions

Más adelante podrían añadirse:

```text
IP binding
device binding
max uses
amount ceiling
```

---

# 343. Financial Capability

Ejemplo:

```text
invoice.approve
Tenant#7
max_amount=10,000
expires=10min
```

---

# 344. ABAC Integration

Capability conditions pueden convertirse en:

```text
ABAC attributes/requirements
```

---

# 345. Example

```text
amount <= capability.max_amount
```

---

# 346. Resource Policy Still Executes

---

# 347. Non-Transferable Delegation

Grant puede declarar:

```text
redelegatable=false
```

---

# 348. Bound Grantee

No puede ser presentado por otro Principal.

---

# 349. Bearer Delegation

Debe evitarse salvo que se modele deliberadamente como Capability.

---

# 350. Delegation ≠ Bearer Token

Delegation normalmente identifica:

```text
grantee
```

---

# 351. Security Distinction

```text
Delegation:
authority belongs to named grantee
```

```text
Bearer capability:
authority belongs to presenter
```

---

# 352. Compiler Metadata

AbilityDescriptor podrá incluir:

```text
delegationPolicy
impersonationPolicy
capabilityPolicy
servicePolicy
```

---

# 353. Example

```text
invoice.view:
    delegatable=true
    redelegatable=false
    impersonation=true
    service=true
    capability=true
```

---

# 354. Example Critical

```text
tenant.delete:
    delegatable=false
    impersonation=false
    capability=false
    service=false
```

salvo dedicated system path.

---

# 355. Runtime Authority Selection

AuthorizationRequest podrá incluir:

```text
AuthorityContext
```

---

# 356. Default

Si no se proporciona:

```text
Native
```

---

# 357. Explicit Delegated Request

```php
Authorization::underDelegation($grant)
    ->authorize('invoice.view', $invoice);
```

conceptualmente.

---

# 358. Impersonation

```php
Authorization::asImpersonated($session)
    ->authorize(...);
```

---

# 359. Capability

```php
Authorization::withCapability($capability)
    ->authorize(...);
```

---

# 360. Avoid Global Mutable Modes

No:

```php
Authorization::setCurrentDelegation($grant);
```

en shared static state.

---

# 361. Scoped Execution

Preferir:

```php
$runtime->runWithAuthority(
    $authorityContext,
    fn () => ...
);
```

con restoration segura.

---

# 362. Persistent Worker Safety

Critical under FrankenPHP.

---

# 363. Cleanup

Después de cada execution:

```text
Actor context
Impersonation context
Delegation context
Capability context
```

deberán limpiarse.

---

# 364. No Authority Leak

Request B nunca debe heredar authority de Request A.

---

# 365. Nested Authority Context

Si se permite:

```text
stack-based
```

y se restaura en `finally`.

---

# 366. Example

Service handling user delegation genera una narrower capability para StorageService.

---

# 367. Parent Authority

Debe permanecer disponible para audit, no como ambient privilege.

---

# 368. Testing

Este subsistema requerirá suites específicas.

---

# 369. Delegation Tests

Debe cubrir:

```text
valid grant
expired grant
revoked grant
scope mismatch
tenant mismatch
subject mismatch
audience mismatch
unauthorized delegation creation
```

---

# 370. Privilege Escalation Test

Grantor con:

```text
invoice.view
```

intenta delegar:

```text
invoice.delete
```

Resultado:

```text
DENY
```

---

# 371. Re-delegation Test

```text
A → B
```

no redelegatable.

B intenta:

```text
B → C
```

Resultado:

```text
DENY
```

---

# 372. Scope Narrowing Test

Parent:

```text
invoice.view + invoice.export
```

Child:

```text
invoice.view
```

válido.

---

# 373. Scope Expansion Test

Child solicita:

```text
invoice.delete
```

inválido.

---

# 374. Impersonation Tests

Cubrir:

```text
authorized start
unauthorized start
expired session
actor preserved
target preserved
restricted ability
nested impersonation
stop/revocation
```

---

# 375. Impersonation Restriction Test

Target tiene:

```text
role.assign
```

pero session support mode lo prohíbe.

Resultado:

```text
DENY
```

---

# 376. Capability Tests

Cubrir:

```text
valid signature
invalid signature
expired
revoked
wrong audience
wrong subject
wrong tenant
consumed
replay
```

---

# 377. Single-Use Concurrency Test

Dos requests concurrentes presentan misma capability.

Resultado esperado:

```text
exactly one succeeds
```

cuando esa sea la semántica.

---

# 378. Service-to-Service Tests

Cubrir:

```text
valid service principal
missing permission
wrong audience
wrong tenant
delegated user context
expired delegation
```

---

# 379. Internal Header Attack Test

Enviar:

```text
X-Internal: true
X-User-ID: admin
```

no deberá otorgar autoridad.

---

# 380. Service Token Scope Test

Service has broad permission, token scope narrow.

Resultado efectivo:

```text
narrow scope
```

---

# 381. Worker Reuse Test

Request A:

```text
ImpersonationSession
```

Request B:

```text
normal user
```

Request B no debe conservar impersonation.

---

# 382. Queue Worker Test

Job A delegated.

Job B native service authority.

No leakage.

---

# 383. Cache Test

Revocar grant y cambiar version/revision.

Nueva ejecución no usa cached GRANT.

---

# 384. Audit Test

Debe registrar:

```text
actor
effective principal
authority source
```

correctamente.

---

# 385. Explain Test

Debe indicar:

```text
GRANT via delegated authority
```

en operator/developer mode.

---

# 386. Failure Test

Delegation store unavailable:

```text
FAILURE
```

no:

```text
GRANT
```

---

# 387. Property-Based Test

Propiedad:

```text
A derived delegation
must never authorize an ability
outside every ancestor's scope.
```

---

# 388. Capability Property

```text
Changing tenant, audience, ability or subject
outside capability scope
must never increase access.
```

---

# 389. Impersonation Property

```text
Impersonation restrictions
must never grant more authority
than ordinary target authority.
```

---

# 390. Service Property

```text
An unauthenticated/unrecognized service
must never gain internal privileges
because of network location alone.
```

---

# 391. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Delegation/
        ├── Context/
        │   ├── AuthorizationAuthorityContext.php
        │   ├── AuthorizationAuthorityMode.php
        │   ├── AuthorizationAuthoritySource.php
        │   ├── AuthorizationAuthoritySet.php
        │   └── AuthorizationAuthorityCeiling.php
        │
        ├── Grants/
        │   ├── DelegatedAuthorizationGrant.php
        │   ├── DelegationStatus.php
        │   ├── DelegationConsistencyMode.php
        │   ├── AuthorizationDelegationChain.php
        │   ├── DelegationRequest.php
        │   ├── DelegationManager.php
        │   └── DelegatedAuthorizationGrantRepositoryInterface.php
        │
        ├── Impersonation/
        │   ├── ImpersonationSession.php
        │   ├── ImpersonationManager.php
        │   ├── ImpersonationRestrictionEvaluator.php
        │   └── ImpersonationSessionRepositoryInterface.php
        │
        ├── Capability/
        │   ├── Capability.php
        │   ├── CapabilityScope.php
        │   ├── CapabilityType.php
        │   ├── CapabilityManager.php
        │   ├── CapabilityIssuerInterface.php
        │   ├── CapabilityRepositoryInterface.php
        │   ├── CapabilityEvaluator.php
        │   └── CapabilityReplayGuard.php
        │
        ├── Service/
        │   ├── ServicePrincipal.php
        │   ├── ServiceAuthorizationEvaluator.php
        │   ├── DelegatedAuthorizationEnvelope.php
        │   ├── DelegatedEnvelopeVerifier.php
        │   └── ServiceAuthorityPolicy.php
        │
        ├── Evaluators/
        │   ├── AuthorityContextEvaluator.php
        │   ├── DelegatedGrantEvaluator.php
        │   ├── DelegationChainEvaluator.php
        │   └── CapabilityScopeEvaluator.php
        │
        ├── Scope/
        │   ├── AuthorizationScopeSet.php
        │   ├── AbilityScope.php
        │   ├── TenantScope.php
        │   ├── SubjectScope.php
        │   ├── AudienceScope.php
        │   └── PurposeScope.php
        │
        └── Exceptions/
            ├── DelegationException.php
            ├── InvalidDelegationException.php
            ├── DelegationScopeException.php
            ├── ImpersonationException.php
            ├── CapabilityException.php
            ├── CapabilityReplayException.php
            └── ServiceAuthorizationException.php
```

---

# 392. Delegation Invariants

### Invariante 1

Delegation no cambia la identidad del grantee.

### Invariante 2

Un grant no puede ampliar autoridad del grantor.

### Invariante 3

Derived grants no pueden ampliar el scope del parent.

### Invariante 4

Expired/revoked grants no conceden autoridad.

### Invariante 5

Re-delegation requiere autorización explícita.

---

# 393. Impersonation Invariants

### Invariante 1

Actor y Effective Principal nunca se confunden.

### Invariante 2

Toda impersonation es explícita.

### Invariante 3

Impersonation puede restringir autoridad del target.

### Invariante 4

Nested impersonation está prohibida por default.

### Invariante 5

Impersonation debe ser auditable y revocable.

---

# 394. Capability Invariants

### Invariante 1

Capability scope es un límite superior de autoridad.

### Invariante 2

Wrong audience nunca concede acceso.

### Invariante 3

Expired/revoked/consumed capability no concede acceso.

### Invariante 4

Bearer secrets nunca se registran en logs.

### Invariante 5

Single-use capabilities implementan protección contra replay.

---

# 395. Service Identity Invariants

### Invariante 1

Servicios son Principals reales.

### Invariante 2

Internal network location no otorga autoridad.

### Invariante 3

Service permissions aplican least privilege.

### Invariante 4

Credential y Principal son conceptos separados.

### Invariante 5

Token scopes limitan, no amplían, autoridad.

---

# 396. Service-to-Service Invariants

### Invariante 1

Cada servicio receptor sigue ejecutando autorización.

### Invariante 2

Un upstream GRANT no es una autoridad genérica.

### Invariante 3

Delegated envelopes tienen audience y expiry.

### Invariante 4

Authority debe estrecharse a través de delegation chains.

### Invariante 5

Tenant context propagado debe estar autenticado y validado.

---

# 397. Runtime Invariants

### Invariante 1

AuthorityContext es execution-scoped.

### Invariante 2

No existe delegation/impersonation state en static globals.

### Invariante 3

FrankenPHP workers limpian authority context después de cada request.

### Invariante 4

Nested scopes se restauran correctamente incluso con exceptions.

### Invariante 5

Ambient service privilege no amplía un delegated operation accidentalmente.

---

# 398. Security Invariants

### Invariante 1

Delegation, impersonation y capabilities nunca son bypasses de TenantIsolation.

### Invariante 2

NonBypassable evaluators siguen ejecutándose.

### Invariante 3

Capability issuance requiere autorización.

### Invariante 4

Delegation creation requiere autorización.

### Invariante 5

Service credentials comprometidas pueden revocarse sin alterar el modelo de permisos completo.

---

# 399. Arquitectura de delegación

```text
Grantor Principal
       │
       ↓
Delegation Creation Authorization
       │
       ↓
DelegatedAuthorizationGrant
       │
       ↓
Grantee Principal
       │
       ↓
Authorization Request
       │
       ↓
AuthorityContextEvaluator
       │
       ↓
DelegatedGrantEvaluator
       │
       ↓
Scope Validation
       │
       ↓
RBAC / ABAC / ReBAC
       │
       ↓
Resource Policy
       │
       ↓
Decision
```

---

# 400. Arquitectura de impersonation

```text
Support Agent
      │
      ↓
user.impersonate
      │
      ↓
ImpersonationSession
      │
      ├── Actor = Support Agent
      └── Effective Principal = User
                    │
                    ↓
             Authorization
                    │
        ┌───────────┼────────────┐
        ↓           ↓            ↓
   User Grants   Tenant Rules   Actor Restrictions
        │           │            │
        └───────────┼────────────┘
                    ↓
                 Decision
```

---

# 401. Arquitectura de Capability

```text
Capability Presented
        │
        ↓
Credential/Signature Validation
        │
        ↓
Capability Resolution
        │
        ↓
Expiry / Revocation / Replay
        │
        ↓
Scope / Audience / Tenant
        │
        ↓
Capability Authority
        │
        ↓
Authorization Pipeline
        │
        ↓
Resource / Security Policies
        │
        ↓
Decision
```

---

# 402. Arquitectura Service-to-Service

```text
SERVICE A
   │
   ↓
Authenticate Workload
   │
   ↓
ServicePrincipal(A)
   │
   ├─────────────── optional user delegation ───────────┐
   │                                                    │
   ↓                                                    ↓
Service Permission                              Delegated Grant
   │                                                    │
   └──────────────────────┬─────────────────────────────┘
                          ↓
                 Authority Context
                          ↓
                 Signed/Bound Envelope
                          ↓
                      SERVICE B
                          ↓
               Verify Service Identity
                          ↓
               Verify Delegated Authority
                          ↓
                Local Authorization Plan
                          ↓
               Tenant + Resource Policies
                          ↓
                       Decision
```

---

# 403. Ejemplo — Delegación humana

User#42:

```text
Role:
finance-manager

Permission:
invoice.approve
```

Desea delegar a User#57:

```text
invoice.approve
Invoice#928
valid 2 hours
```

---

# 404. Creation

VoltStack verifica:

```text
User#42 can approve Invoice#928
User#42 may delegate invoice.approve
Requested scope does not exceed own authority
```

---

# 405. Grant

```text
DelegatedGrant#88

grantor:
User#42

grantee:
User#57

ability:
invoice.approve

subject:
Invoice#928

tenant:
7

expires:
17:00
```

---

# 406. Execution

User#57 intenta aprobar Invoice#928.

Pipeline:

```text
PrincipalState
→ GRANT

TenantIsolation
→ GRANT

DelegatedGrantEvaluator
→ GRANT

InvoicePolicy
→ GRANT

Final
→ GRANT
```

---

# 407. Wrong Invoice

User#57 intenta Invoice#929.

```text
DelegatedGrantEvaluator
→ DENY
delegation.scope_mismatch
```

---

# 408. Example — Support Impersonation

Actor:

```text
SupportAgent#10
```

Target:

```text
User#42
```

Tenant:

```text
7
```

---

# 409. Session

```text
ImpersonationSession#19

scope:
read
profile troubleshooting

expires:
30 min
```

---

# 410. User#42 puede eliminar Invoice.

SupportAgent intenta hacerlo durante impersonation.

Pipeline:

```text
Effective Principal permission
→ GRANT

InvoicePolicy
→ GRANT

ImpersonationRestrictionEvaluator
→ DENY

reason:
impersonation.operation_restricted
```

Final:

```text
DENY
```

---

# 411. Example — Signed Download Capability

Document#100 requiere compartir archivo externamente.

Se emite:

```text
Capability#88

ability:
document.download

subject:
Document#100

tenant:
7

expires:
15 min
```

---

# 412. Anonymous Request

Presenta capability válida.

Pipeline:

```text
Capability signature
→ valid

Capability expiry
→ valid

Subject scope
→ match

Tenant binding
→ match

DocumentPolicy::downloadShared
→ GRANT
```

Final:

```text
GRANT
```

---

# 413. Después de Expiry

```text
CapabilityEvaluator
→ DENY
capability.expired
```

---

# 414. Example — Service-to-Service

User#42 solicita:

```text
export Report#81
```

API autoriza:

```text
report.export
```

---

# 415. Delegated Grant

API emite para ExportService:

```text
grant:
report.export
Report#81
Tenant#7
expires 10min
audience=export-service
```

---

# 416. ExportService

Necesita Document#100.

Genera narrower capability:

```text
storage.object.read
Object#555
Tenant#7
audience=storage-service
expires 2min
```

---

# 417. StorageService

No confía simplemente en:

```text
"ExportService said yes"
```

Verifica:

```text
ServicePrincipal(export-service)
Capability audience
Capability scope
Tenant
Object
Local storage Policy
```

---

# 418. Resultado

Autoridad se estrecha:

```text
User report.export
        ↓
ExportService Report#81
        ↓
StorageService Object#555 read only
```

No existe un token:

```text
"act as User#42 everywhere"
```

---

# 419. Filosofía del sistema

La filosofía definitiva será:

```text
Know who is actually acting.

Know whose authority is being used.

Know where that authority came from.

Limit delegated authority by ability, tenant,
subject, audience, purpose and time.

Prefer delegation to impersonation when possible.

Treat service accounts as real Principals.

Treat capabilities as authority, not bypasses.

Reauthorize locally across service boundaries.

Narrow authority as it moves through a distributed system.

Preserve the complete provenance of privileged actions.

And always fail closed when delegated authority
cannot be validated safely.
```

---

# 420. Resultado esperado

El `Delegation, Impersonation, Capabilities and Service-to-Service Authorization System` permitirá que VoltStack cubra escenarios avanzados como:

```text
Customer support impersonation
Delegated approvals
Background jobs
Long-running workflows
API keys
Personal access tokens
Signed URLs
Temporary download links
Service accounts
Machine identities
Internal microservices
Delegated distributed requests
Single-use operation grants
MFA step-up capabilities
```

sin reducirlos a simples:

```text
tokens
headers
admin flags
```

El modelo final será:

```text
IDENTITY
   ↓
ACTOR
   ↓
EFFECTIVE PRINCIPAL
   ↓
AUTHORITY SOURCE
   ↓
AUTHORITY CEILING
   ↓
TENANT / SUBJECT / PURPOSE / AUDIENCE
   ↓
NORMAL AUTHORIZATION PIPELINE
   ↓
DECISION
   ↓
AUDITABLE PROVENANCE
```

El principio definitivo será:

```text
Authority may be delegated.

Identity may be impersonated.

Capabilities may be issued.

Services may act autonomously.

But every operation must still answer:

Who is acting?

Whose authority is being used?

What exactly was delegated?

For which tenant and resource?

For how long?

For what audience and purpose?

And which non-bypassable rules still apply?
```

Con este subsistema, VoltStack podrá utilizar un único modelo coherente para autorización humana, administrativa, asíncrona y distribuida, conservando **least privilege, multi-tenant isolation, trazabilidad y fail-closed behavior** incluso cuando la autoridad viaje entre usuarios, workers y servicios.