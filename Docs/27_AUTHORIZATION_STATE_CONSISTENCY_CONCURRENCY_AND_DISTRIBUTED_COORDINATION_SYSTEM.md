# VoltStack Authorization System

## Authorization State Consistency, Concurrency and Distributed Coordination System

**Documento:** `27_AUTHORIZATION_STATE_CONSISTENCY_CONCURRENCY_AND_DISTRIBUTED_COORDINATION_SYSTEM.md`  
**Sistema:** Authorization  
**Framework:** VoltStack  
**Módulo sugerido:** `Quantum/Authorization`  
**Estado:** Especificación arquitectónica  
**Versión objetivo:** 1.x+

---

# 1. Propósito

Este documento define la arquitectura para garantizar la **consistencia del estado de autorización**, el manejo correcto de **concurrencia**, la prevención de **race conditions**, y la coordinación de decisiones en entornos:

```text
multi-process
multi-worker
multi-node
distributed
event-driven
queue-based
FrankenPHP persistent workers
```

El objetivo es evitar que una decisión válida en un instante sea utilizada incorrectamente después de cambios como:

```text
role revoked
permission removed
tenant membership revoked
delegation revoked
capability consumed
scope moved
resource changed
risk escalated
approval revoked
policy changed
```

La regla fundamental será:

> **Una autorización solo es válida respecto al estado, versiones, contexto y autoridad que existían bajo las condiciones para las cuales fue evaluada.**

---

# 2. Problema arquitectónico

Un Authorization Engine puede producir:

```text
ALLOW
```

en:

```text
T1
```

pero antes de que ocurra el efecto protegido, en:

```text
T2
```

puede cambiar:

```text
User role
Tenant membership
Resource state
Delegation
Capability
Approval
Risk
Policy
Scope hierarchy
```

y finalmente la operación ejecutarse en:

```text
T3
```

El problema es:

```text
T1 authorization
        ↓
state changes
        ↓
T3 execution
```

---

# 3. TOCTOU

Este problema corresponde al clásico:

```text
Time Of Check
    ↓
Time Of Use
```

o:

```text
TOCTOU
```

---

# 4. Ejemplo

```text
T1:
User has document.delete
→ ALLOW

T2:
Permission revoked

T3:
Document deleted using old decision
```

Sin mecanismos adicionales, el sistema ejecutó una operación utilizando autoridad obsoleta.

---

# 5. Objetivo

VoltStack deberá poder controlar:

```text
authorization state versioning
decision freshness
revocation visibility
concurrent mutation
single-use authority
atomic authorization operations
distributed invalidation
policy generation
resource versioning
tenant versioning
scope versioning
relationship versioning
```

---

# 6. Principio fundamental

No deberá existir un concepto implícito de:

```text
permission granted forever
```

Una decisión deberá ser interpretada como:

```text
Decision(
    principal,
    ability,
    resource,
    context,
    versions,
    time
)
```

---

# 7. Authorization State

Definiremos:

```text
Authorization State
```

como el conjunto de datos que puede alterar una decisión.

---

# 8. Ejemplos de state dimensions

```text
principal status
role assignments
permissions
tenant membership
scope membership
scope hierarchy
resource ownership
resource relationships
shares
delegation grants
capabilities
token scopes
impersonation state
risk
approval evidence
policy version
resource state
```

---

# 9. Authorization State Vector

VoltStack podrá representar estas dependencias mediante:

```php
final readonly class AuthorizationStateVector
{
    public function __construct(
        public int $principalVersion,
        public int $tenantVersion,
        public int $policyVersion,
        public ?int $scopeVersion = null,
        public ?int $resourceVersion = null,
        public ?int $relationshipVersion = null,
        public ?int $delegationVersion = null,
    ) {}
}
```

---

# 10. Objetivo del State Vector

Permitir detectar:

```text
decision produced under v10
current state v11
```

y evitar reutilización incorrecta.

---

# 11. No existe un único version counter universal obligatorio

El diseño deberá permitir:

```text
coarse-grained versioning
```

y:

```text
fine-grained versioning
```

según dominio.

---

# 12. Coarse-Grained Versioning

Ejemplo:

```text
tenant_authorization_version
```

Se incrementa ante cualquier cambio relevante dentro del Tenant.

Ventaja:

```text
simple invalidation
```

Desventaja:

```text
more cache invalidation
```

---

# 13. Fine-Grained Versioning

Ejemplo:

```text
principal_role_version
scope_membership_version
resource_acl_version
policy_version
```

Ventaja:

```text
precise invalidation
```

Desventaja:

```text
higher complexity
```

---

# 14. Recommendation

VoltStack deberá soportar ambos modelos.

Default empresarial razonable:

```text
Principal Authorization Version
Tenant Authorization Version
Policy Version
Resource Authorization Version
```

con extensiones más granulares.

---

# 15. Principal Authorization Version

Cada Principal podrá tener:

```text
authorization_version
```

---

# 16. Incrementar cuando cambia

```text
role assignment
permission assignment
account security status
membership affecting authority
delegation-sensitive identity state
```

---

# 17. Example

```text
User#42
authorization_version=38
```

Role revoked:

```text
authorization_version=39
```

Decisiones que dependían de:

```text
38
```

quedan stale.

---

# 18. Tenant Authorization Version

```text
Tenant#7
authorization_version=104
```

Puede cambiar cuando:

```text
tenant roles change
tenant policy changes
membership structure changes
tenant security configuration changes
```

---

# 19. Scope Version

Documento 22 ya introdujo:

```text
scope_hierarchy_version
scope_membership_version
```

Este documento formaliza su uso en consistencia.

---

# 20. Resource Authorization Version

Un recurso podrá exponer:

```text
authorization_version
```

si cambios en:

```text
owner
visibility
classification
share state
parent scope
resource state
```

alteran authorization.

---

# 21. Domain Version vs Authorization Version

No confundir:

```text
resource.version
```

con:

```text
resource.authorization_version
```

---

# 22. Ejemplo

Modificar:

```text
Document.title
```

puede cambiar:

```text
resource.version
```

pero no necesariamente:

```text
authorization_version
```

---

# 23. Authorization-Relevant Mutation

Toda mutación deberá declarar si afecta:

```text
authorization state
```

---

# 24. Contract

```php
interface AuthorizationVersionedResourceInterface
{
    public function authorizationVersion(): int|string;
}
```

---

# 25. Decision Snapshot

Una decisión podrá guardar:

```php
final readonly class AuthorizationDecisionSnapshot
{
    public function __construct(
        public AuthorizationDecision $decision,
        public AuthorizationStateVector $state,
        public DateTimeImmutable $evaluatedAt,
        public ?DateTimeImmutable $validUntil,
    ) {}
}
```

---

# 26. Freshness

Una decisión podrá estar:

```text
fresh
stale
expired
invalid
```

---

# 27. DecisionFreshness

```php
enum AuthorizationDecisionFreshness: string
{
    case Fresh = 'fresh';
    case Stale = 'stale';
    case Expired = 'expired';
    case Invalid = 'invalid';
}
```

---

# 28. Decision Freshness Evaluator

```php
interface AuthorizationDecisionFreshnessEvaluatorInterface
{
    public function evaluate(
        AuthorizationDecisionSnapshot $snapshot,
        AuthorizationStateVector $current
    ): AuthorizationDecisionFreshness;
}
```

---

# 29. Cache Reuse Rule

Solo reutilizar una decisión cuando:

```text
snapshot state
matches
current relevant state
```

o existe una estrategia explícitamente segura.

---

# 30. Never Assume Time Alone Is Enough

Una decisión de hace:

```text
1 second
```

puede ser inválida si:

```text
permission revoked 100ms ago
```

---

# 31. Version > TTL

Para security-sensitive decisions:

```text
version correctness
```

es más importante que:

```text
short TTL
```

---

# 32. TTL Still Useful

Sirve para:

```text
bounded reuse
volatile context
external providers
risk assessments
```

---

# 33. Revocation

VoltStack deberá tratar revocación como un concepto explícito.

---

# 34. Revocable authorities

```text
roles
permissions
delegations
capabilities
shares
memberships
approvals
sessions
service credentials
```

---

# 35. Revocation Requirement

Una autoridad revocada deberá dejar de producir ALLOW según la consistencia configurada.

---

# 36. Revocation Consistency Modes

```php
enum AuthorizationRevocationConsistency: string
{
    case Immediate = 'immediate';
    case Strong = 'strong';
    case Bounded = 'bounded';
    case Eventual = 'eventual';
}
```

---

# 37. Immediate

La siguiente operación deberá observar revocación.

Ideal para:

```text
critical authority
```

---

# 38. Strong

Puede requerir coordinación con canonical store.

---

# 39. Bounded

Revocación será visible dentro de una ventana máxima.

Ejemplo:

```text
<= 5 seconds
```

---

# 40. Eventual

Acepta propagación asincrónica.

Adecuado solo para:

```text
low-risk non-critical decisions
```

---

# 41. Default Recommendation

Para:

```text
privilege revocation
tenant suspension
delegation revocation
capability consumption
critical approval revocation
```

usar:

```text
Strong / Immediate
```

cuando sea viable.

---

# 42. Distributed Nodes

Ejemplo:

```text
Node A
Node B
Node C
```

cada uno puede tener:

```text
local cache
persistent workers
compiled authorization state
```

---

# 43. Revocation Propagation

```text
Canonical State
      ↓
Version Change
      ↓
Invalidation Event
      ↓
Node A
Node B
Node C
```

---

# 44. Canonical Authority Store

Debe existir una fuente de verdad para datos dinámicos.

Ejemplos:

```text
SQL
distributed store
IAM
directory
policy service
```

---

# 45. Cache Is Not Authority Source

Un cache no debe convertirse accidentalmente en:

```text
canonical authorization store
```

salvo que esté diseñado expresamente para ello.

---

# 46. Authorization Invalidation Event

```php
final readonly class AuthorizationStateInvalidated
{
    public function __construct(
        public string $dimension,
        public string $reference,
        public int|string $newVersion,
        public DateTimeImmutable $occurredAt,
    ) {}
}
```

---

# 47. Examples

```text
principal:42 → version 39
tenant:7 → version 105
scope:91 → version 18
resource:500 → version 9
```

---

# 48. Invalidation Bus

Contrato:

```php
interface AuthorizationInvalidationBusInterface
{
    public function publish(
        AuthorizationStateInvalidated $event
    ): void;
}
```

---

# 49. Node Subscriber

```php
interface AuthorizationInvalidationSubscriberInterface
{
    public function handle(
        AuthorizationStateInvalidated $event
    ): void;
}
```

---

# 50. Invalidation Actions

Un node puede:

```text
evict cache
increment local generation
invalidate request-local projections
mark snapshot stale
```

---

# 51. Out-of-Order Events

En sistemas distribuidos pueden llegar:

```text
v12
v10
v11
```

---

# 52. Rule

Nunca retroceder version state.

```text
current=12
incoming=10
→ ignore stale event
```

---

# 53. Monotonic Version Store

```php
interface AuthorizationVersionStoreInterface
{
    public function current(
        AuthorizationVersionKey $key
    ): int|string;

    public function advance(
        AuthorizationVersionKey $key,
        int|string $version
    ): void;
}
```

---

# 54. Integer Versioning

Recomendable cuando la fuente puede mantener:

```text
monotonic counters
```

---

# 55. Alternative Version Tokens

También pueden utilizarse:

```text
UUID generation
ETag
revision hash
database row version
```

---

# 56. Comparable versions

Si no son ordenables, se deberá usar:

```text
equality semantics
```

no `>`.

---

# 57. Authorization Generation

Puede existir una generación global:

```text
authorization_generation
```

para invalidaciones gruesas.

---

# 58. Use Case

Deployment cambia cientos de Policies:

```text
generation 50
→ 51
```

Todo compiled decision cache de 50 queda inválido.

---

# 59. Policy Generation

Documento 25 definió runtime generations.

Aquí se formaliza:

```text
RuntimeGeneration
+
DynamicStateVersions
```

---

# 60. Decision Identity

Una decisión cacheada deberá incluir:

```text
runtime generation
policy version
state vector
context fingerprint
```

---

# 61. Race Condition: Role Revocation

Caso:

```text
Thread A:
authorize user
→ ALLOW

Thread B:
revoke role

Thread A:
execute
```

---

# 62. Solutions

Dependiendo de sensibilidad:

```text
revalidation
optimistic concurrency
transaction boundary
authority lease
atomic policy enforcement
```

---

# 63. Revalidation

Antes de ejecutar:

```text
current principal auth version
==
authorized principal auth version
```

---

# 64. If mismatch

```text
re-authorize
```

---

# 65. Optimistic Authorization Concurrency

Concepto:

```text
Authorize under version V
Execute only if version still V
```

---

# 66. Contract

```php
final readonly class AuthorizationExecutionPrecondition
{
    public function __construct(
        public AuthorizationStateVector $expectedState,
    ) {}
}
```

---

# 67. Atomic Check-and-Execute

Ideal cuando authority state y business state viven en el mismo datastore.

---

# 68. Example SQL Conceptual

```text
BEGIN

verify principal.authorization_version = 38
verify resource.authorization_version = 5

perform mutation

COMMIT
```

---

# 69. Caveat

No siempre principal authorization state comparte DB/transacción con resource.

---

# 70. Distributed TOCTOU

Entonces se requieren mecanismos como:

```text
revalidation
leases
short-lived proofs
versioned claims
idempotency
```

---

# 71. Authorization Lease

Para algunos sistemas se podrá emitir:

```text
AuthorizationLease
```

---

# 72. Definition

Una autoridad temporal y limitada:

```php
final readonly class AuthorizationLease
{
    public function __construct(
        public string $id,
        public AuthorizationOperationDescriptor $operation,
        public AuthorizationStateVector $state,
        public DateTimeImmutable $issuedAt,
        public DateTimeImmutable $expiresAt,
    ) {}
}
```

---

# 73. Lease Use

```text
authorize
    ↓
issue short lease
    ↓
execute remotely
```

---

# 74. Lease Must Be Narrow

Atado a:

```text
principal
ability
resource
tenant
scope
operation
state versions
expiry
audience
```

---

# 75. Lease Is Not General Token

No debe convertirse en:

```text
temporary superuser token
```

---

# 76. Revocation vs Lease

Una Policy debe definir si:

```text
revocation invalidates active lease immediately
```

o si:

```text
lease remains valid until expiry
```

---

# 77. Critical Authority

Preferir:

```text
revocation invalidates lease
```

---

# 78. Requires Central Revocation Check

Esto puede aumentar costo.

---

# 79. Bounded Lease

Alternativa:

```text
lease TTL <= acceptable revocation lag
```

---

# 80. Single-Use Authority

Capabilities y approval proofs pueden ser:

```text
single-use
```

---

# 81. Race

Dos workers intentan consumir el mismo capability.

---

# 82. Requirement

Debe existir:

```text
atomic consume
```

---

# 83. Compare-and-Swap

```text
status=ACTIVE
        ↓ CAS
status=CONSUMED
```

solo uno gana.

---

# 84. Contract

```php
interface AtomicAuthorizationGrantStoreInterface
{
    public function consume(
        AuthorizationGrantId $id,
        AuthorizationGrantVersion $expectedVersion
    ): AuthorizationGrantConsumptionResult;
}
```

---

# 85. Grant Consumption Result

```php
enum AuthorizationGrantConsumptionResult: string
{
    case Consumed = 'consumed';
    case AlreadyConsumed = 'already_consumed';
    case Revoked = 'revoked';
    case Expired = 'expired';
    case VersionConflict = 'version_conflict';
}
```

---

# 86. No Read-Then-Write

Incorrecto:

```text
read active
if active
    update consumed
```

sin atomicidad.

---

# 87. Use

```text
UPDATE ... WHERE status='active' AND version=7
```

conceptualmente.

---

# 88. Optimistic Locking

Resources de autorización deberán soportarlo cuando existe modificación concurrente.

---

# 89. Example

Dos admins editan Role.

Admin A:

```text
version=10
```

Admin B:

```text
version=10
```

A guarda:

```text
version=11
```

B intenta guardar version 10:

```text
conflict
```

---

# 90. Authorization Mutation Version

```php
interface AuthorizationMutableEntityInterface
{
    public function version(): int|string;
}
```

---

# 91. Authorization Mutation Command

```php
final readonly class AuthorizationMutationCommand
{
    public function __construct(
        public string $type,
        public string $target,
        public int|string $expectedVersion,
    ) {}
}
```

---

# 92. Why

Evita lost updates sobre:

```text
roles
policies
shares
delegations
memberships
```

---

# 93. Pessimistic Locking

Podrá utilizarse en operaciones muy críticas.

Ejemplo:

```text
last organization owner removal
```

---

# 94. Use Cases

```text
remove final admin
transfer ownership
consume single-use grant
approve final quorum
move authorization scope
```

---

# 95. Don't Lock Everything

Authorization hot path deberá permanecer mayormente lock-free.

---

# 96. Mutation vs Evaluation

Distinguir:

```text
Authorization Evaluation
```

de:

```text
Authorization State Mutation
```

---

# 97. Evaluation

Idealmente:

```text
read-only
```

---

# 98. Mutation

Puede necesitar:

```text
transactions
locks
CAS
version increments
invalidation
```

---

# 99. AuthorizationMutationManager

```php
interface AuthorizationMutationManagerInterface
{
    public function execute(
        AuthorizationMutation $mutation
    ): AuthorizationMutationResult;
}
```

---

# 100. Mutation Pipeline

```text
Authorize Mutation Ability
        ↓
Load Current Version
        ↓
Validate Invariants
        ↓
Apply Mutation
        ↓
Advance Version
        ↓
Persist
        ↓
Commit
        ↓
Publish Invalidation
```

---

# 101. Publish After Commit

Muy importante.

No publicar:

```text
role revoked
```

antes de saber que la transacción realmente commitió.

---

# 102. Transactional Outbox

Recomendable:

```text
DB transaction
├── authority mutation
└── outbox invalidation event

COMMIT
    ↓
Outbox dispatcher
```

---

# 103. Why

Evita:

```text
DB rollback
but invalidation event already published
```

---

# 104. Invalidation Event Idempotency

Consumers deben tolerar delivery:

```text
at least once
```

---

# 105. Duplicate invalidation

```text
v39
v39
```

debe ser harmless.

---

# 106. Cache Coherence

VoltStack deberá distinguir:

```text
request cache
local worker cache
shared distributed cache
canonical state
```

---

# 107. Layers

```text
L0 Evaluation Memoization
L1 Request Memoization
L2 Worker Local Cache
L3 Distributed Cache
L4 Canonical Store
```

---

# 108. Security Rule

Cada capa deberá tener:

```text
clear invalidation semantics
```

---

# 109. Request Memoization

Más fácil:

```text
destroy at request end
```

---

# 110. Worker Cache

Necesita:

```text
version checks
bounded TTL
invalidation subscriptions
```

---

# 111. Distributed Cache

Necesita:

```text
namespace
versions
atomic operations where needed
```

---

# 112. Canonical Store

Fuente de verdad.

---

# 113. Cache Stampede

Cambios de versión pueden provocar muchos misses.

---

# 114. Stampede Control

Puede utilizar:

```text
single-flight
jitter
soft TTL
background refresh
```

solo donde no comprometa freshness de seguridad.

---

# 115. Never Serve Stale Security Decision During Refresh

Patrón típico web:

```text
stale-while-revalidate
```

no debe aplicarse indiscriminadamente a:

```text
authorization ALLOW
```

---

# 116. Safe SWR

Podría utilizarse para:

```text
non-authoritative metadata
```

pero no para autoridad revocada.

---

# 117. Negative Decision Cache

`DENY` también puede cachearse.

---

# 118. Advantage

Evita repeated expensive checks.

---

# 119. Caution

Un nuevo grant puede volver stale el DENY.

---

# 120. Therefore

También necesita versions.

---

# 121. Monotonic Security Cache Principle

Cuando hay incertidumbre sobre freshness:

```text
do not reuse ALLOW
```

---

# 122. Fail-Safe Revalidation

Si version provider no está disponible:

```text
critical ALLOW cache
→ do not trust
```

---

# 123. Distributed Policy Updates

Supongamos:

```text
Node A deploys Policy v20
Node B still v19
```

---

# 124. Mixed Policy Generation

Puede producir decisiones inconsistentes.

---

# 125. Deployment Strategies

VoltStack deberá ser compatible con:

```text
rolling deployment
blue-green
canary
```

---

# 126. Runtime Generation

Cada node deberá anunciar:

```text
authorization_runtime_generation
```

---

# 127. Cross-Service Decisions

Si Service A envía una autorización derivada a B, deberá incluir:

```text
issuer generation
policy version
authority state version
```

cuando sea relevante.

---

# 128. Receiver Rule

Service B no deberá confiar en:

```text
upstream says allow
```

como documento 20.

---

# 129. Local Reauthorization

Receiver debe:

```text
validate delegated authority
then authorize locally
```

---

# 130. Version Mismatch

Si envelope contiene:

```text
authority_version=38
```

y B conoce:

```text
39
```

debe considerar grant stale.

---

# 131. Distributed Delegation Envelope

Puede incluir:

```text
grant_id
grant_version
principal_version
tenant_version
policy_generation
issued_at
expires_at
```

---

# 132. Message Queue Authorization

Job creado:

```text
T1
```

ejecutado:

```text
T2 hours later
```

---

# 133. Default Recommendation

No serializar una decisión:

```text
ALLOW
```

y confiar horas después.

---

# 134. Instead

Serializar:

```text
authorization intent
principal reference
tenant
scope
explicit delegated grant if required
```

---

# 135. Worker Reauthorization

En ejecución:

```text
load current state
validate grant
authorize
```

---

# 136. Snapshot Execution

Solo si domain requiere:

```text
authority snapshot at submission time
```

y está explícitamente modelado.

---

# 137. Snapshot Mode

Ejemplo:

```php
enum AsyncAuthorizationMode: string
{
    case ReauthorizeAtExecution = 'reauthorize_at_execution';
    case SnapshotUntilExpiry = 'snapshot_until_expiry';
    case DelegatedGrant = 'delegated_grant';
}
```

---

# 138. Default

```text
ReauthorizeAtExecution
```

para operaciones normales.

---

# 139. Snapshot Security

Si se usa Snapshot:

```text
short expiry
narrow operation
explicit revocation semantics
```

---

# 140. Eventual Consistency

Algunas relaciones pueden provenir de:

```text
LDAP
SCIM
external IAM
graph projection
```

y ser eventual.

---

# 141. Provider Consistency Metadata

```php
final readonly class AuthorizationProviderConsistency
{
    public function __construct(
        public AuthorizationConsistencyLevel $level,
        public ?DateInterval $maximumStaleness = null,
    ) {}
}
```

---

# 142. AuthorizationConsistencyLevel

```php
enum AuthorizationConsistencyLevel: string
{
    case Strong = 'strong';
    case ReadAfterWrite = 'read_after_write';
    case BoundedStaleness = 'bounded_staleness';
    case Eventual = 'eventual';
}
```

---

# 143. Policy Can Require Consistency

Ability crítica:

```text
production.root.rotate
```

puede exigir:

```text
Strong
```

---

# 144. Low-Risk Ability

```text
dashboard.view
```

podría aceptar:

```text
BoundedStaleness <= 30s
```

---

# 145. Consistency Requirement

```php
final readonly class AuthorizationConsistencyRequirement
{
    public function __construct(
        public AuthorizationConsistencyLevel $minimum,
        public ?DateInterval $maximumStaleness = null,
    ) {}
}
```

---

# 146. Provider Cannot Satisfy

Si provider solo ofrece:

```text
Eventual
```

pero Policy requiere:

```text
Strong
```

resultado:

```text
FAILURE / DENY
```

no ALLOW.

---

# 147. Consistency Is Context

La decisión deberá conocer:

```text
how fresh is authority data?
```

---

# 148. Read-Your-Writes

Muy importante para administración.

Admin:

```text
revokes user role
```

y acto seguido verifica acceso.

Debe poder observar su propia mutación.

---

# 149. Read-After-Write Barrier

```php
interface AuthorizationReadAfterWriteBarrierInterface
{
    public function waitUntilVisible(
        AuthorizationMutationCommit $commit
    ): void;
}
```

---

# 150. Distributed Systems

Esto puede implementarse mediante:

```text
version acknowledgements
leader reads
session consistency token
```

según backend.

---

# 151. Mutation Consistency Token

```php
final readonly class AuthorizationConsistencyToken
{
    public function __construct(
        public array $versions,
    ) {}
}
```

---

# 152. Use

Después de mutación:

```text
token version=39
```

siguientes reads exigen:

```text
>=39
```

---

# 153. No Global Locks Across Cluster by Default

Distributed lock debe reservarse para operaciones donde sea realmente necesario.

---

# 154. Why

Locks distribuidos introducen:

```text
latency
failure modes
deadlocks
lease expiry issues
```

---

# 155. Prefer

```text
versioning
idempotency
CAS
revalidation
```

antes que distributed locks.

---

# 156. Distributed Lock Use Cases

Puede justificarse para:

```text
single global owner transfer
unique break-glass activation
critical single-consumer authority
```

---

# 157. Lock Contract

```php
interface AuthorizationDistributedLockInterface
{
    public function acquire(
        AuthorizationLockKey $key,
        DateInterval $ttl
    ): AuthorizationLockLease;
}
```

---

# 158. Fencing Tokens

Si se usan locks distribuidos, preferir:

```text
fencing tokens
```

para evitar que un holder antiguo siga actuando tras perder lock.

---

# 159. Example

```text
Lock holder A fencing=50
Lock expires
B fencing=51
A wakes up
```

Resource debe rechazar:

```text
50
```

si ya observó:

```text
51
```

---

# 160. Critical for Distributed Safety

Un lock sin fencing puede dar falsa seguridad.

---

# 161. Approval Concurrency

Documento 24:

Dos approvers aprueban simultáneamente.

---

# 162. Quorum Race

```text
required=2
current=1

Approver A
Approver B
```

ambos pueden intentar completar workflow.

---

# 163. Requirement

Stage completion debe ser:

```text
idempotent
atomic
```

---

# 164. Workflow State CAS

```text
IN_PROGRESS version=6
→ APPROVED version=7
```

solo una transición lógica.

---

# 165. Duplicate Completion Event

Debe evitarse o ser idempotentemente procesable.

---

# 166. Approval Revocation Race

```text
executor validates approval
approver revokes
executor executes
```

---

# 167. Sensitive Execution

Debe utilizar:

```text
proof version
revocation version
atomic consume/revalidation
```

---

# 168. Resource Mutation Race

Authorization puede depender de:

```text
owner_id
classification
workspace_id
```

---

# 169. Example

Authorize:

```text
document belongs Workspace A
```

Concurrent move:

```text
Workspace B
```

Then update.

---

# 170. Resource Version Check

Execution deberá verificar:

```text
expected authorization version
```

---

# 171. Query-Level Authorization

Listados presentan otro problema.

---

# 172. Example

```text
SELECT authorized documents
```

Mientras query corre:

```text
membership revoked
```

---

# 173. Snapshot Semantics

DB puede ofrecer:

```text
transaction snapshot
```

pero autoridad externa puede cambiar fuera de esa transacción.

---

# 174. Recommendation

Para listado:

```text
bounded snapshot consistency
```

puede ser aceptable.

Para acción sobre un item:

```text
reauthorize item at mutation time
```

---

# 175. Never Treat Authorized Query Result as Execution Proof

Que un recurso aparezca en:

```text
authorized list
```

no significa que una posterior mutación esté autorizada.

---

# 176. Authorization Pagination

La página 1 y página 2 pueden observar versiones diferentes.

---

# 177. Optional Query Consistency Token

VoltStack podrá permitir:

```text
authorization query generation
```

para consultas que requieren snapshot coherente.

---

# 178. Example

```text
scope_membership_version=91
```

se fija para toda una exportación.

---

# 179. Security Trade-Off

Mantener snapshot demasiado tiempo puede permitir datos ya revocados.

Policy deberá definir máxima duración.

---

# 180. Transaction Authorization Context

Dentro de una transacción:

```php
final readonly class TransactionAuthorizationContext
{
    public function __construct(
        public AuthorizationStateVector $baseline,
        public string $transactionId,
    ) {}
}
```

---

# 181. Purpose

Correlacionar decisiones con:

```text
business transaction
```

sin convertir autorización en propietario de la transacción DB.

---

# 182. Database Boundary

Authorization no deberá:

```text
begin/commit arbitrary domain transactions
```

por sí solo.

---

# 183. Instead

Podrá producir:

```text
execution preconditions
```

que Domain/Database layer aplica.

---

# 184. Example

```text
Authorization:
expected resource auth_version = 12

Database:
UPDATE resources
SET ...
WHERE id=10
AND authorization_version=12
```

---

# 185. If 0 rows updated

```text
concurrency conflict
```

Re-load y re-authorize.

---

# 186. AuthorizationConcurrencyException

```text
not DENY
```

necesariamente.

Puede ser:

```text
CONFLICT / RETRY REQUIRED
```

---

# 187. Decision Outcome vs Execution Outcome

Authorization puede producir ALLOW pero ejecución puede fallar por concurrency.

---

# 188. API Mapping

Opcionalmente:

```text
409 Conflict
```

cuando sea un conflicto de resource version.

---

# 189. Do Not Return 403 for Every Race

Distinguir:

```text
authority denied
```

de:

```text
state changed, retry
```

---

# 190. AuthorizationExecutionValidation

```php
enum AuthorizationExecutionValidationOutcome: string
{
    case Valid = 'valid';
    case Stale = 'stale';
    case Conflict = 'conflict';
    case Revoked = 'revoked';
    case Expired = 'expired';
    case Failure = 'failure';
}
```

---

# 191. Retry Policy

Reauthorization automática puede permitirse cuando:

```text
operation is idempotent
no side effect occurred
retry count bounded
```

---

# 192. Never Infinite Retry

Concurrent hot resource podría causar loop.

---

# 193. Retry Budget

```php
final readonly class AuthorizationRetryPolicy
{
    public function __construct(
        public int $maxAttempts,
        public DateInterval $maxDuration,
    ) {}
}
```

---

# 194. High-Risk Operations

Preferir devolver conflicto y requerir nueva interacción.

---

# 195. Distributed Clock

Expiry de:

```text
capabilities
leases
approvals
delegations
```

depende de tiempo.

---

# 196. Clock Skew

Nodos pueden tener pequeñas diferencias.

---

# 197. Therefore

Policies temporales distribuidas deberán considerar:

```text
clock skew tolerance
```

---

# 198. Trusted Clock

VoltStack Core usará:

```text
ClockInterface
```

como documento 23.

---

# 199. Expiry Validation

Conceptualmente:

```text
now >= expiresAt
→ expired
```

con skew policy explícita cuando corresponda.

---

# 200. Never Extend Expiry Because of Cache

Cache TTL no debe prolongar una grant más allá de:

```text
expiresAt
```

---

# 201. Effective Cache TTL

```text
min(
    configured cache TTL,
    grant expiresAt - now,
    risk validity,
    approval validity
)
```

---

# 202. Versioned Context

Context también puede cambiar dentro del request.

Ejemplo:

```text
MFA step-up
```

---

# 203. Context Version

Documento 25:

```text
context_version
```

---

# 204. Decision Cache Key

Debe incluir:

```text
context_version
```

cuando los context changes afectan decision.

---

# 205. Tenant Switch

Debe generar:

```text
new context generation
```

---

# 206. Nested Authority Scope

`runWithDelegation()` deberá usar:

```text
scope stack
```

request-local.

---

# 207. Concurrent Fibers / Async PHP

Si VoltStack utiliza concurrencia cooperativa:

```text
Fiber A
Fiber B
```

no deberán compartir un mutable `CurrentAuthorizationContext`.

---

# 208. Important

Request-scoped no siempre significa:

```text
safe for concurrent tasks inside same request
```

---

# 209. Execution-Local Context

VoltStack deberá estar preparado para:

```text
execution-local / fiber-local context
```

cuando haya concurrencia.

---

# 210. AuthorizationExecutionScope

```php
interface AuthorizationExecutionScopeInterface
{
    public function context(): AuthorizationContext;

    public function fork(): AuthorizationExecutionScopeInterface;
}
```

---

# 211. Fork

Al crear concurrent task:

```text
immutable context snapshot
```

puede compartirse.

Mutable stacks deberán copiarse.

---

# 212. No Shared Mutable Stack

```text
delegation stack
scope stack
temporary restrictions
```

no pueden ser un array mutable compartido por Fibers.

---

# 213. Scoped Context Token

```php
final readonly class AuthorizationExecutionToken
{
    public function __construct(
        public string $executionId,
        public string $requestId,
        public int $generation,
    ) {}
}
```

---

# 214. Worker Safety

FrankenPHP:

```text
Request A
Request B
```

pueden usar el mismo proceso secuencialmente, y eventualmente runtimes pueden soportar más concurrencia.

La arquitectura deberá ser segura en ambos casos.

---

# 215. No Mutable Singleton

Nunca:

```php
final class AuthorizationContextHolder
{
    public static ?AuthorizationContext $current;
}
```

---

# 216. Distributed Cache Namespace

Formato conceptual:

```text
authz:
app:{appVersion}:
runtime:{generation}:
tenant:{tenantVersion}:
principal:{principalVersion}:
...
```

---

# 217. Avoid Giant Keys

La implementación deberá utilizar hashing/canonicalization.

---

# 218. Cache Fingerprint

```php
final readonly class AuthorizationCacheFingerprint
{
    public function __construct(
        public string $hash,
        public AuthorizationStateVector $versions,
    ) {}
}
```

---

# 219. Hash Is Not Version

Un hash de inputs ayuda a keying, pero no reemplaza mecanismos de revocación.

---

# 220. Cache Invalidation Strategies

Podrán combinarse:

```text
Versioned Keys
Explicit Eviction
Generation Rotation
Short TTL
```

---

# 221. Prefer Versioned Keys

En entornos distribuidos suelen evitar:

```text
delete every old key
```

---

# 222. Old Keys

Quedan inaccesibles y expiran posteriormente.

---

# 223. Memory Control

Debe existir TTL para limpiar generaciones antiguas.

---

# 224. Authorization Version Explosion

Fine-grained versions pueden generar demasiadas combinaciones.

---

# 225. Planner

Debe identificar solo:

```text
relevant version dimensions
```

para cada Ability.

---

# 226. Example

`profile.avatar.view` quizá necesita:

```text
principal status
resource visibility
```

pero no:

```text
approval_version
delegation_version
```

---

# 227. Compiled State Dependencies

```php
final readonly class AuthorizationStateDependencyPlan
{
    public function __construct(
        public array $dimensions,
    ) {}
}
```

---

# 228. Cache Key Optimization

Solo incluir dependencias usadas.

---

# 229. Dynamic Dependency

Si Policy accede a una fuente dinámica:

```text
register dependency
```

durante planificación/compilation cuando sea posible.

---

# 230. Hidden Dependency Is Dangerous

Policy que consulta directamente una tabla de autorización fuera de engine puede romper invalidation.

---

# 231. Rule

Authorization-relevant state deberá pasar por:

```text
declared providers
```

o declarar dependency.

---

# 232. Dependency Tracking

Conceptualmente:

```php
interface AuthorizationStateDependencyCollectorInterface
{
    public function dependOn(
        AuthorizationStateDependency $dependency
    ): void;
}
```

---

# 233. Development Mode

VoltStack puede detectar algunas undeclared state dependencies mediante diagnostics.

---

# 234. Distributed Decision Service

VoltStack podría soportar un PDP remoto.

---

# 235. PDP Consistency

Request debería incluir:

```text
principal version
tenant version
policy generation
resource version
```

cuando estén disponibles.

---

# 236. PDP Response

Debe incluir:

```text
decision
evaluated versions
valid_until
cacheability
```

---

# 237. Example

```text
ALLOW
principal_version=38
tenant_version=104
policy_generation=51
valid_until=...
```

---

# 238. PEP Validation

Policy Enforcement Point podrá comparar versiones antes de usar la respuesta.

---

# 239. PDP Response Signature

En arquitecturas distribuidas podría firmarse.

---

# 240. Signature ≠ Freshness

Una respuesta firmada puede seguir estando stale.

---

# 241. Distributed Enforcement

Cada servicio deberá mantener:

```text
local enforcement responsibility
```

---

# 242. Never Trust Upstream Execution Claim

Ejemplo:

```text
X-Authorized: true
```

no.

---

# 243. Security Headers

Si existe metadata upstream, deberá estar:

```text
authenticated
integrity-protected
audience-bound
versioned
```

y aun así su semántica deberá ser explícita.

---

# 244. Policy Change Race

Authorization comienza bajo:

```text
Policy v5
```

mientras deployment activa:

```text
v6
```

---

# 245. Evaluation Generation Pinning

Cada evaluación deberá usar:

```text
one consistent runtime generation
```

de principio a fin.

---

# 246. No Half-v5 Half-v6

Generation swap debe ocurrir entre evaluaciones.

---

# 247. Request-Level Generation

Opcionalmente todo request puede fijar:

```text
runtime generation
```

para coherencia interna.

---

# 248. Trade-Off

Una request muy larga puede seguir usando una generación antigua.

---

# 249. Critical Checkpoints

Operaciones críticas pueden exigir:

```text
current generation
```

antes de ejecución.

---

# 250. Policy Emergency Revocation

Si se descubre una vulnerabilidad, puede requerirse:

```text
global security generation
```

que invalide inmediatamente generaciones anteriores.

---

# 251. Security Epoch

Concepto:

```text
AuthorizationSecurityEpoch
```

---

# 252. Purpose

Un counter global de emergencia.

```text
epoch 10 → 11
```

Toda decisión de epoch 10 es inválida.

---

# 253. Use Sparingly

Es una invalidación gruesa y costosa.

---

# 254. Great for

```text
critical global policy incident
credential compromise
authorization vulnerability
```

---

# 255. Decision Snapshot Security Epoch

Deberá guardar:

```text
security_epoch
```

si el feature está habilitado.

---

# 256. Principal Suspension

Debe tener propagación fuerte.

---

# 257. Suggested

Separar:

```text
principal_status_version
```

de roles si se necesita immediate suspension.

---

# 258. Why

Suspender una cuenta puede requerir invalidar:

```text
all sessions
all cached allows
delegated actions
```

rápidamente.

---

# 259. Emergency Status Store

Puede utilizar un provider de alta consistencia.

---

# 260. Tenant Suspension

Igualmente puede tener:

```text
tenant_security_version
```

independiente.

---

# 261. Mandatory Runtime Checks

Algunos estados críticos deberán comprobarse incluso con cached ALLOW.

Ejemplos:

```text
principal suspended
tenant suspended
capability consumed
grant revoked
```

---

# 262. Fast Revocation Index

Puede existir:

```text
revocation index
```

optimizado para checks baratos.

---

# 263. Revocation Index

No necesariamente contiene toda authority.

Solo:

```text
critical invalidations
```

---

# 264. Bloom Filter Caution

Structures probabilísticas podrían ayudar a detectar posibles revocations, pero:

```text
false positive → extra verification
```

aceptable.

```text
false negative → unauthorized access
```

no.

---

# 265. Therefore

Si se usa probabilistic structure, debe diseñarse sin false negatives para revocation safety.

---

# 266. Event Ordering

Mutaciones relacionadas deben conservar orden lógico.

Ejemplo:

```text
Grant created v1
Grant revoked v2
```

Node recibe:

```text
v2
then v1
```

v1 no debe reactivar grant.

---

# 267. State Machine Version

Cada entidad autorizativa debe impedir backward transition por stale event.

---

# 268. Delegation State

```text
ACTIVE v3
REVOKED v4
```

evento v3 posterior no puede volver a ACTIVE.

---

# 269. Tombstones

En sistemas eventuales puede ser necesario conservar:

```text
revocation tombstone
```

durante un período.

---

# 270. Why

Si se borra completamente una grant revocada y llega posteriormente un evento viejo de creación, podría reaparecer.

---

# 271. Tombstone TTL

Debe exceder:

```text
maximum message/event delay
```

según arquitectura.

---

# 272. Authorization Event Sourcing

VoltStack no deberá exigir Event Sourcing.

---

# 273. But Compatible

Los contratos deberán permitir que un backend use:

```text
event log
projections
```

como fuente.

---

# 274. Projection Consistency

Una proyección atrasada deberá exponer:

```text
projection version / lag
```

---

# 275. Policy

Puede rechazar una proyección demasiado stale.

---

# 276. ProjectionMetadata

```php
final readonly class AuthorizationProjectionMetadata
{
    public function __construct(
        public int|string $version,
        public DateTimeImmutable $updatedAt,
        public ?DateInterval $lag,
    ) {}
}
```

---

# 277. Replicated Databases

Read replicas pueden tener lag.

---

# 278. Security Warning

Consultar revocation-sensitive authority desde replica atrasada puede permitir acceso revocado.

---

# 279. Read Source Policy

Ability crítica puede requerir:

```text
primary / strongly consistent source
```

---

# 280. Repository Contract Extension

```php
interface ConsistencyAwareAuthorizationRepositoryInterface
{
    public function supports(
        AuthorizationConsistencyRequirement $requirement
    ): bool;
}
```

---

# 281. Read/Write Splitting Integration

Database subsystem deberá poder forzar:

```text
primary read
```

para checks de alta consistencia.

---

# 282. Same-Request Mutation

User actualiza:

```text
Role
```

y después hace authorization en mismo request.

---

# 283. Request Memoization Invalidation

Mutation Manager debe invalidar:

```text
request-local cached decisions
```

inmediatamente.

---

# 284. Mutation Event Internal

Antes incluso de distributed event:

```text
local request invalidation
```

debe ocurrir tras commit.

---

# 285. Version-aware memoization

La próxima consulta verá:

```text
new version
```

---

# 286. Transaction Rollback

No avanzar versión visible localmente antes de commit irreversible.

---

# 287. Post-Commit Hook

Authorization Mutation Manager necesita:

```text
after commit
```

integration.

---

# 288. Outbox recommended

Cuando el Database system lo soporte.

---

# 289. State Transition Invariants

Cada mutación deberá validar estado anterior.

Ejemplo:

```text
REVOKED → ACTIVE
```

puede ser ilegal.

---

# 290. Reissue Instead

Crear:

```text
new delegation grant
```

en lugar de revivir grant revocada, según dominio.

---

# 291. Identity Map Interaction

Si ORM mantiene entity cached:

```text
User role collection
```

puede estar stale aunque DB ya cambió.

---

# 292. Authorization Core Must Avoid Blind ORM State

Para security-critical authority:

```text
repository / version-aware provider
```

deberá controlar freshness.

---

# 293. Unit of Work

Mutaciones pendientes no commitidas presentan otra semántica.

---

# 294. Example

Transaction actual añade permission pero aún no commit.

¿Debe Authorization verla?

---

# 295. Policy

Debe ser explícito.

Opciones:

```text
committed state only
transaction-local state
```

---

# 296. Default Recommendation

Authorization general:

```text
committed state
```

---

# 297. Domain Transaction Authorization

Puede usar:

```text
transaction-local projection
```

si está cuidadosamente diseñado.

---

# 298. Never Leak Uncommitted Authority Across Requests

Crítico.

---

# 299. Distributed Transaction

VoltStack no deberá exigir 2PC.

---

# 300. Instead

Preferir:

```text
sagas
outbox
idempotency
revalidation
```

---

# 301. Cross-System Revocation

Servicio de Identity revoca user.

Authorization service debe recibirlo.

---

# 302. Contract

```text
IdentityStateChanged
        ↓
Authorization invalidation adapter
        ↓
principal security version++
```

---

# 303. Bounded Propagation SLO

Puede definirse:

```text
critical principal suspension visible cluster-wide <= 2s
```

---

# 304. Authorization Consistency SLO

VoltStack podrá exponer métricas:

```text
authorization.invalidation.propagation_latency
authorization.cache.stale_detected.total
authorization.version_conflict.total
authorization.reauthorization.total
```

---

# 305. Metrics

También:

```text
authorization.revocation.check.total
authorization.revocation.hit.total
authorization.execution.stale.total
authorization.distributed.version_lag
```

---

# 306. Tracing

Spans:

```text
authorization.version.resolve
authorization.revocation.check
authorization.state.revalidate
authorization.invalidation.publish
authorization.invalidation.consume
authorization.atomic.consume
```

---

# 307. Audit

Registrar security-relevant conflicts:

```text
stale decision rejected
single-use grant replay
approval already consumed
version conflict
revoked authority attempted
```

---

# 308. Replay Detection

Si capability single-use ya consumida vuelve a usarse:

```text
DENY
+
security audit
```

---

# 309. Replay Is Not Normal Cache Miss

Debe tener reason code específico.

---

# 310. Reason Codes

Propuesta:

```text
authorization.state.stale
authorization.state.version_mismatch
authorization.state.generation_mismatch

authorization.revoked
authorization.revocation.provider_failure

authorization.concurrency.conflict
authorization.concurrency.retry_required

authorization.grant.already_consumed
authorization.grant.version_conflict

authorization.policy.generation_changed

authorization.resource.authorization_version_changed

authorization.consistency.insufficient
authorization.consistency.provider_lag

authorization.invalidation.out_of_order
authorization.invalidation.failed
```

---

# 311. Failure Semantics

Distinguir:

```text
DENY
```

de:

```text
STALE
```

y:

```text
FAILURE
```

---

# 312. DENY

Authority actual no permite operación.

---

# 313. STALE / CONFLICT

La decisión anterior ya no puede confiarse.

---

# 314. FAILURE

No podemos determinar correctamente current authority.

---

# 315. Authorization Decision Extension

Puede existir metadata:

```php
final readonly class AuthorizationConsistencyMetadata
{
    public function __construct(
        public AuthorizationStateVector $state,
        public AuthorizationConsistencyLevel $consistency,
        public DateTimeImmutable $evaluatedAt,
    ) {}
}
```

---

# 316. Execution Validator

```php
interface AuthorizationExecutionValidatorInterface
{
    public function validate(
        AuthorizationDecisionSnapshot $decision,
        AuthorizationExecutionContext $execution
    ): AuthorizationExecutionValidationOutcome;
}
```

---

# 317. Critical Operation Flow

```text
Authorization Request
        ↓
ALLOW
        ↓
Decision Snapshot
        ↓
Business Preparation
        ↓
Execution Validator
        ↓
Versions Still Valid?
   ┌────┴────┐
   │         │
 YES        NO
   │         │
Execute   Reauthorize
```

---

# 318. Irreversible Operations

Ejemplos:

```text
money transfer
production deployment
credential rotation
organization deletion
```

deberán usar stronger pre-execution validation.

---

# 319. Read Operations

Pueden aceptar más relaxed consistency.

---

# 320. Ability Consistency Metadata

Abilities podrán declarar:

```php
final readonly class AbilityConsistencyDescriptor
{
    public function __construct(
        public AuthorizationConsistencyLevel $requiredConsistency,
        public bool $revalidateBeforeExecution,
        public bool $allowCachedGrant,
    ) {}
}
```

---

# 321. Example

```text
document.view:
BoundedStaleness
cached grant allowed

payment.execute:
Strong
revalidate before execution
cached grant false
```

---

# 322. Configuration

Conceptual:

```php
'consistency' => [
    'default' => 'read_after_write',

    'critical' => [
        'revalidate_before_execution' => true,
    ],

    'invalidation' => [
        'enabled' => true,
    ],
],
```

---

# 323. Defaults

VoltStack deberá proporcionar defaults seguros sin exigir configuración exhaustiva.

---

# 324. Distributed Coordinator

Puede existir:

```php
interface AuthorizationDistributedCoordinatorInterface
{
    public function publishInvalidation(
        AuthorizationStateInvalidated $event
    ): void;

    public function currentVersion(
        AuthorizationVersionKey $key
    ): int|string;
}
```

---

# 325. Core Neutrality

No depender directamente de:

```text
Redis
Kafka
NATS
RabbitMQ
database notifications
```

---

# 326. Adapters

Podrán implementarse en infraestructura.

---

# 327. Redis

Puede proporcionar:

```text
distributed version store
pub/sub invalidation
atomic consume
locks
```

---

# 328. Database

Puede proporcionar:

```text
canonical versions
transactions
CAS
outbox
```

---

# 329. Event Bus

Puede distribuir:

```text
invalidation events
```

---

# 330. Hybrid Model

Recomendado para sistemas grandes:

```text
SQL canonical authority
+
Redis hot version/index
+
Event Bus invalidation
```

sin que Core dependa de ninguno.

---

# 331. Distributed Coordinator Failure

Si coordinator está caído:

```text
what happens?
```

---

# 332. Policy by Criticality

Low-risk:

```text
fallback to canonical store
```

Critical:

```text
FAIL / DENY
```

si freshness no puede garantizarse.

---

# 333. Never Fallback to Stale ALLOW Silently

Regla crítica.

---

# 334. Circuit Breaker

Puede evitar bombardear servicio caído.

Pero circuito abierto:

```text
does not equal allow
```

---

# 335. State Synchronizer

```php
interface AuthorizationStateSynchronizerInterface
{
    public function synchronize(
        AuthorizationVersionKey $key
    ): AuthorizationSynchronizationResult;
}
```

---

# 336. Recovery

Node que estuvo offline deberá:

```text
resync versions
```

antes de confiar en cache local.

---

# 337. Startup Rule

Worker nuevo no deberá asumir que:

```text
local cache empty
```

equivale a state current.

---

# 338. It should

```text
load current generation/version roots
```

cuando sea necesario.

---

# 339. Node Epoch

Puede existir:

```text
node observed security epoch
```

---

# 340. If behind

```text
refresh mandatory security state
```

---

# 341. Split Brain

Distributed stores pueden sufrir partición.

---

# 342. Security Preference

Para critical authority:

```text
availability may yield to safety
```

---

# 343. CAP Trade-Off

Authorization deberá permitir que Policy exprese:

```text
fail closed under partition
```

---

# 344. Do Not Hide Distributed Trade-Offs

VoltStack deberá documentar que no existe:

```text
perfect availability + perfect immediate consistency
```

en todas las topologías.

---

# 345. Consistency Profiles

Podrán existir:

```text
Relaxed
Standard
Strong
Critical
```

---

# 346. Example

```php
enum AuthorizationConsistencyProfile: string
{
    case Relaxed = 'relaxed';
    case Standard = 'standard';
    case Strong = 'strong';
    case Critical = 'critical';
}
```

---

# 347. Mapping

```text
Relaxed
→ bounded staleness

Standard
→ version-aware read-after-write

Strong
→ current canonical/version check

Critical
→ strong + pre-execution revalidation + mandatory revocation
```

---

# 348. Ability Declaration

```text
payment.execute
consistency=critical
```

---

# 349. Planner Integration

Authorization Planner deberá insertar:

```text
version resolution
revocation checks
execution precondition
```

según profile.

---

# 350. Avoid Cost for All Abilities

No ejecutar heavy distributed consistency protocol para:

```text
public profile view
```

---

# 351. Risk-Based Consistency

Documento 23 permite adaptar requirements.

Ejemplo:

```text
Risk Low
→ Standard consistency

Risk High
→ Strong consistency
```

---

# 352. Contextual Stronger Consistency

También:

```text
impersonation
delegation
critical resource
```

pueden elevar consistency.

---

# 353. Approval-Based Consistency

Workflow aprobado no reduce requirements.

Puede elevarlos.

---

# 354. Example

```text
final financial execution
→ critical consistency
```

---

# 355. Delegation Revocation Version

Cada grant puede tener:

```text
version
status_version
```

---

# 356. Parent Delegation Revocation

Si grant hijo depende del padre:

```text
parent_generation
```

debe invalidarlo cuando padre se revoca.

---

# 357. Delegation Chain Version

Puede calcularse usando:

```text
ancestor grant versions
```

o:

```text
delegation tree generation
```

---

# 358. Capability Revocation

Reference capability:

```text
strong revocation possible
```

Bearer self-contained capability:

```text
needs revocation index or short expiry
```

---

# 359. Architectural Trade-Off

Self-contained authority reduce datastore calls pero dificulta:

```text
immediate revocation
```

---

# 360. VoltStack Rule

El tipo de capability deberá declarar:

```text
revocation semantics
```

---

# 361. Token Scope Version

API tokens podrán incluir:

```text
credential_version
```

---

# 362. Credential Revocation

Incrementar:

```text
credential_version
```

o registrar JTI revocation.

---

# 363. Identity Linking Changes

Documento Auth 34 puede alterar Principal identity linkage.

Authorization deberá invalidar:

```text
identity authorization projection
```

cuando corresponda.

---

# 364. Scope Move

Documento 22:

```text
Workspace moved from Org A to Org B
```

---

# 365. This changes

```text
scope hierarchy
inherited roles
resource authority
```

---

# 366. Required

Atomic hierarchy mutation + version increment + invalidation.

---

# 367. Move Race

Authorization durante move deberá observar:

```text
old complete hierarchy
```

o:

```text
new complete hierarchy
```

no un híbrido incoherente.

---

# 368. Transactional Hierarchy Update

Cuando storage lo permita.

---

# 369. Projection Rebuild

Si closure table/materialized path se reconstruye async:

```text
hierarchy projection version
```

debe marcarse stale hasta consistente.

---

# 370. Relationship Graph Consistency

ReBAC puede usar graph projection.

Debe exponer:

```text
relationship_version
```

---

# 371. Direct Fast Paths

Ownership/share directos pueden tener versiones separadas de graph projection.

---

# 372. Policy Resolver Consistency

Dynamic Policy Provider también puede cambiar.

---

# 373. Policy Provider Version

```text
policy_provider_version
```

cuando aplica.

---

# 374. Extension Version

Documento 26:

```text
authorization_extension_version
```

también forma parte de semántica si extension afecta decision.

---

# 375. Full Decision Dependency Vector

Conceptualmente:

```text
RuntimeGeneration
SecurityEpoch
PrincipalVersion
TenantVersion
ScopeVersion
ResourceVersion
RelationshipVersion
DelegationVersion
PolicyVersion
ExtensionVersion
ContextVersion
```

---

# 376. Important

No todas las decisiones necesitan todas las dimensiones.

---

# 377. Dependency-Minimized Vector

Planner selecciona:

```text
minimal relevant subset
```

---

# 378. Performance Goal

Seguridad correcta sin convertir cada:

```text
can('view')
```

en 10 consultas distribuidas.

---

# 379. Version Projection

Puede existir una fuente compacta:

```text
AuthorizationVersionProjection
```

que agregue múltiples versions.

---

# 380. Example

```text
principal_effective_auth_version
```

cambia si:

```text
roles
permissions
membership
critical status
```

cambian.

---

# 381. Trade-Off

Reduce key complexity, aumenta invalidations.

---

# 382. Testing Strategy

Este subsystem requiere pruebas agresivas de concurrencia.

---

# 383. Role Revocation Race Test

```text
authorize ALLOW
revoke role
attempt execution
```

debe resultar:

```text
revalidation failure
```

para critical ability.

---

# 384. Permission Grant Race

Cached DENY bajo version 10.

Grant permission → version 11.

La próxima decisión no debe reutilizar DENY v10.

---

# 385. Single-Use Capability Test

100 concurrent consumers.

Resultado:

```text
exactly 1 success
99 already_consumed
```

---

# 386. Approval Proof Test

Dos executors concurrentes.

Solo uno consume proof single-use.

---

# 387. Out-of-Order Invalidation Test

Recibir:

```text
v12
v9
v11
```

final local version:

```text
12
```

---

# 388. Duplicate Event Test

Recibir v12 dos veces.

No error ni rollback.

---

# 389. Missed Event Test

Node pierde v11 pero luego recibe v12.

Debe quedar current sin requerir recibir todos intermedios si version model lo permite.

---

# 390. Worker Local Cache Test

Node cachea ALLOW v5.

Invalidation v6.

Next request no reutiliza v5.

---

# 391. FrankenPHP Leakage Test

State vector de Request A no aparece en Request B.

---

# 392. Fiber Concurrency Test

Dos Fibers con distintas scoped delegations.

No comparten authority stack.

---

# 393. Policy Generation Swap Test

Evaluation comienza en generation 50.

Generation 51 se activa.

Evaluation termina consistentemente bajo 50.

Nueva evaluation usa 51.

---

# 394. Security Epoch Test

Decision snapshot epoch 8.

Global epoch cambia a 9.

Execution validator rechaza snapshot 8.

---

# 395. Resource Move Race Test

Authorize under Workspace A.

Resource moves B.

Execution must detect resource auth version mismatch.

---

# 396. Read Replica Lag Test

Critical policy requiere Strong.

Replica only offers bounded stale.

Engine must not ALLOW from replica.

---

# 397. Read-After-Write Test

Revoke permission then authorize same Principal.

Must observe revocation according to configured consistency.

---

# 398. Transaction Rollback Test

Mutation increments temporary state but transaction rolls back.

No invalidation/version advance must become globally visible.

---

# 399. Outbox Test

Mutation + outbox commit atomically.

Dispatcher retry produces idempotent invalidation.

---

# 400. Retry Test

Optimistic conflict:

```text
attempt 1 conflict
attempt 2 reload + reauthorize
```

works when Policy permits.

---

# 401. Property-Based Invariant

If current relevant version differs from snapshot, cached ALLOW cannot be considered fresh.

---

# 402. Property-Based Invariant

A revocation event with older version can never restore authority.

---

# 403. Property-Based Invariant

Consuming a single-use authority more than once never yields more than one successful result.

---

# 404. Property-Based Invariant

Increasing security epoch never increases validity of old decisions.

---

# 405. Property-Based Invariant

Stronger consistency requirement cannot accept a provider that failed the weaker requirement.

---

# 406. Property-Based Invariant

Adding a revocation cannot increase authority.

---

# 407. Chaos Tests

Para distributed deployment:

```text
delayed invalidation
duplicate messages
node restart
cache loss
redis outage
database replica lag
message bus partition
```

---

# 408. Critical Expected Behavior

Ante incertidumbre:

```text
Critical ability
→ fail closed
```

---

# 409. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Consistency/
        ├── Contracts/
        │   ├── AuthorizationVersionStoreInterface.php
        │   ├── AuthorizationInvalidationBusInterface.php
        │   ├── AuthorizationInvalidationSubscriberInterface.php
        │   ├── AuthorizationDecisionFreshnessEvaluatorInterface.php
        │   ├── AuthorizationExecutionValidatorInterface.php
        │   ├── AuthorizationDistributedCoordinatorInterface.php
        │   ├── AuthorizationReadAfterWriteBarrierInterface.php
        │   └── AuthorizationStateSynchronizerInterface.php
        │
        ├── Model/
        │   ├── AuthorizationStateVector.php
        │   ├── AuthorizationDecisionSnapshot.php
        │   ├── AuthorizationConsistencyMetadata.php
        │   ├── AuthorizationConsistencyRequirement.php
        │   ├── AuthorizationConsistencyLevel.php
        │   ├── AuthorizationConsistencyProfile.php
        │   ├── AuthorizationSecurityEpoch.php
        │   └── AuthorizationConsistencyToken.php
        │
        ├── Versioning/
        │   ├── AuthorizationVersionKey.php
        │   ├── PrincipalAuthorizationVersion.php
        │   ├── TenantAuthorizationVersion.php
        │   ├── ResourceAuthorizationVersion.php
        │   ├── ScopeAuthorizationVersion.php
        │   └── AuthorizationVersionResolver.php
        │
        ├── Freshness/
        │   ├── AuthorizationDecisionFreshness.php
        │   ├── AuthorizationDecisionFreshnessEvaluator.php
        │   └── AuthorizationStateDependencyPlan.php
        │
        ├── Invalidation/
        │   ├── AuthorizationStateInvalidated.php
        │   ├── AuthorizationInvalidationManager.php
        │   ├── AuthorizationInvalidationConsumer.php
        │   └── AuthorizationRevocationIndex.php
        │
        ├── Concurrency/
        │   ├── AuthorizationExecutionPrecondition.php
        │   ├── AuthorizationRetryPolicy.php
        │   ├── AuthorizationExecutionValidationOutcome.php
        │   ├── AuthorizationConcurrencyManager.php
        │   └── AtomicAuthorizationGrantStoreInterface.php
        │
        ├── Lease/
        │   ├── AuthorizationLease.php
        │   ├── AuthorizationLeaseIssuer.php
        │   └── AuthorizationLeaseVerifier.php
        │
        ├── Distributed/
        │   ├── AuthorizationDistributedCoordinator.php
        │   ├── AuthorizationStateSynchronizer.php
        │   ├── AuthorizationDistributedLockInterface.php
        │   └── AuthorizationLockLease.php
        │
        ├── Mutation/
        │   ├── AuthorizationMutationManager.php
        │   ├── AuthorizationMutation.php
        │   ├── AuthorizationMutationCommand.php
        │   ├── AuthorizationMutationResult.php
        │   └── AuthorizationMutationCommit.php
        │
        ├── Runtime/
        │   ├── AuthorizationExecutionScope.php
        │   ├── AuthorizationExecutionToken.php
        │   └── AuthorizationStateDependencyCollector.php
        │
        └── Exceptions/
            ├── AuthorizationConsistencyException.php
            ├── AuthorizationConcurrencyException.php
            ├── AuthorizationStaleDecisionException.php
            ├── AuthorizationVersionConflictException.php
            ├── AuthorizationRevocationException.php
            └── AuthorizationDistributedCoordinationException.php
```

---

# 410. Complete Consistency Pipeline

```text
Authorization Request
        ↓
Resolve Relevant State Dependencies
        ↓
Resolve Current Versions
        ↓
Resolve Required Consistency Profile
        ↓
Load Authority State
        ↓
Evaluate Authorization
        ↓
Capture State Vector
        ↓
ALLOW / DENY / CHALLENGE
        ↓
[if operation is sensitive]
        ↓
Execution Precondition
        ↓
Business Preparation
        ↓
Revalidate Current Versions
        ↓
Revocation Check
        ↓
Resource Version Check
        ↓
Policy Generation Check
        ↓
Still Valid?
   ┌────┴────┐
   │         │
 YES        NO
   │         │
Execute   Reauthorize / Conflict
```

---

# 411. Distributed Invalidation Architecture

```text
                CANONICAL AUTHORIZATION STATE
                           │
                           ↓
                    MUTATION COMMIT
                           │
                           ↓
                    VERSION ADVANCE
                           │
                           ↓
                 TRANSACTIONAL OUTBOX
                           │
                           ↓
                   INVALIDATION BUS
             ┌─────────────┼─────────────┐
             ↓             ↓             ↓
          Node A         Node B         Node C
             │             │             │
         Local Version  Local Version  Local Version
             │             │             │
         Cache Evict    Cache Evict    Cache Evict
```

---

# 412. Single-Use Authority Architecture

```text
Capability / Approval Proof
            │
            ↓
        ACTIVE v5
            │
      concurrent use
       ┌────┴────┐
       ↓         ↓
    Worker A   Worker B
       │         │
       └────┬────┘
            ↓
       ATOMIC CAS
            │
      ┌─────┴─────┐
      ↓           ↓
 CONSUMED     CONFLICT
 Worker A      Worker B
```

---

# 413. Persistent Runtime Architecture

```text
                 FRANKENPHP WORKER
                        │
         ┌──────────────┴──────────────┐
         │                             │
    Immutable Runtime             Shared Clients
    Generation                    / Version Adapters
         │                             │
         └──────────────┬──────────────┘
                        │
                 REQUEST / EXECUTION
                        │
              Current State Vector
                        │
                Decision Memoization
                        │
                 Evaluation Stack
                        │
                        ↓
                      RESET
```

No deben persistir:

```text
current principal state
current versions snapshot
current delegation stack
request memoization
```

entre requests.

---

# 414. Core Invariants

### Invariante 1

Una decisión está ligada a las versiones relevantes bajo las que fue evaluada.

### Invariante 2

Una versión stale nunca debe interpretarse como fresh.

### Invariante 3

Revocaciones nunca incrementan autoridad.

### Invariante 4

Eventos de invalidación antiguos no pueden revertir estados nuevos.

### Invariante 5

Single-use authority se consume atómicamente.

---

# 415. Concurrency Invariants

### Invariante 1

Los cambios concurrentes no deben producir lost updates sobre estado crítico.

### Invariante 2

Operaciones sensibles pueden exigir revalidación antes de ejecución.

### Invariante 3

Un conflict de versión no debe interpretarse como ALLOW.

### Invariante 4

Retries son bounded e idempotentes.

### Invariante 5

El authorization hot path no requiere locks globales por default.

---

# 416. Distributed Invariants

### Invariante 1

Cada node conoce la generación/version correspondiente a decisiones cacheadas.

### Invariante 2

La pérdida de un evento no puede volver permanente un estado stale si existe resincronización/versioning.

### Invariante 3

Cache distribuido no es fuente de autoridad por accidente.

### Invariante 4

Una partición no produce ALLOW cuando la Policy requiere strong consistency.

### Invariante 5

Upstream authorization nunca elimina la necesidad de enforcement local cuando corresponda.

---

# 417. Versioning Invariants

### Invariante 1

Versions solo avanzan semánticamente.

### Invariante 2

Rollback no publica versiones no commitidas.

### Invariante 3

Runtime generation es consistente dentro de una evaluación.

### Invariante 4

Security epoch invalida decisiones anteriores cuando se incrementa.

### Invariante 5

Solo las dependencias relevantes deben formar parte del vector de una decisión.

---

# 418. Cache Invariants

### Invariante 1

TTL nunca extiende una autoridad más allá de su expiración real.

### Invariante 2

Cached ALLOW no omite revocation checks mandatory.

### Invariante 3

Cached DENY también se invalida cuando aparecen nuevos grants.

### Invariante 4

Context/version changes invalidan memoization relevante.

### Invariante 5

Stale-while-revalidate no se aplica a ALLOW crítico sin una semántica explícitamente segura.

---

# 419. Async Invariants

### Invariante 1

Jobs no deben depender de una decisión histórica sin contrato explícito.

### Invariante 2

Reauthorization-at-execution es el default.

### Invariante 3

Snapshots o grants async son estrechos, expiran y poseen revocation semantics.

### Invariante 4

Authority envelopes son audience-bound.

### Invariante 5

El receptor revalida authority localmente cuando el modelo lo exige.

---

# 420. Filosofía arquitectónica

VoltStack deberá adoptar las siguientes reglas:

```text
Authorization decisions are state-dependent.

Cached authority is never timeless authority.

Revocation is a first-class consistency problem.

Versions make stale decisions detectable.

TTL is not a substitute for revocation.

Critical operations revalidate before irreversible effects.

Single-use authority is consumed atomically.

Authorization evaluation is read-oriented;
authorization mutation is transaction-oriented.

Versioning, CAS and idempotency are preferred
over broad distributed locks.

Distributed caches accelerate authority resolution;
they do not become authority accidentally.

Every distributed authority claim carries provenance,
scope, version and expiry when required.

A network partition must not silently turn uncertainty
into permission.

Persistent workers retain immutable structure,
not mutable decision state.
```

---

# 421. Resultado esperado

Con `27_AUTHORIZATION_STATE_CONSISTENCY_CONCURRENCY_AND_DISTRIBUTED_COORDINATION_SYSTEM.md`, VoltStack podrá mantener una semántica de autorización correcta incluso cuando:

```text
roles change while requests are executing

permissions are revoked across several servers

FrankenPHP workers reuse memory

resources move between scopes

approval proofs are consumed concurrently

delegations are revoked

capabilities are replayed

queues execute hours after dispatch

database replicas are lagging

authorization caches exist on multiple nodes

policies change during rolling deployment
```

El Authorization System pasa así de preguntar únicamente:

```text
"¿está autorizado?"
```

a poder responder también:

```text
"¿bajo qué versión del estado fue autorizado?"

"¿sigue siendo válida esa autoridad?"

"¿puedo reutilizar esta decisión?"

"¿fue revocada?"

"¿debo reautorizar antes de ejecutar?"

"¿qué consistencia exige esta Ability?"
```

La regla arquitectónica definitiva será:

> **En VoltStack, una decisión de autorización no es una verdad permanente: es una conclusión producida sobre una versión concreta del estado de seguridad. Si ese estado cambia, la validez de la decisión debe poder detectarse, invalidarse o revaluarse de manera explícita.**

Por tanto:

```text
AUTHORITY
+
STATE VERSION
+
CONSISTENCY
+
REVOCATION
+
CONCURRENCY CONTROL
+
EXECUTION REVALIDATION
=
SAFE AUTHORIZATION IN DISTRIBUTED RUNTIMES
```

Con este documento quedan formalizadas las garantías necesarias para que Authorization pueda operar correctamente desde una aplicación monolítica hasta arquitecturas distribuidas con múltiples workers y servicios.

El siguiente documento de la secuencia es **`28_AUTHORIZATION_DATA_MODEL_PERSISTENCE_AND_STORAGE_BOUNDARIES_SYSTEM.md`**, donde corresponde definir con precisión qué datos pertenecen al Authorization System, qué tablas/proyecciones/repositorios necesita, cuáles son datos canónicos frente a caches o derivados, y dónde terminan las responsabilidades de Authorization frente a Authentication, Identity, Database, Audit y los dominios de la aplicación.
