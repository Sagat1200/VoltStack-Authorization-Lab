# VoltStack Authorization System — Cache, Memoization and Decision Reuse System

## 1. Propósito

Este documento define la arquitectura de **cache, memoization y reutilización segura de decisiones** dentro del Authorization System de VoltStack.

El sistema deberá distinguir cuidadosamente entre:

```text
Metadata Cache
Registry Cache
Plan Cache
Request Memoization
Grant Cache
Decision Cache
Negative Cache
External Evaluator Cache
```

porque no todos estos mecanismos tienen las mismas garantías de seguridad.

Una regla fundamental será:

```text
Caching what should run
is usually safer than
caching what the answer was.
```

---

# 2. Problema que resuelve

Una autorización puede involucrar:

```text
Policy resolution
Gate resolution
Role lookup
Permission lookup
Tenant membership
Relationship traversal
ABAC attributes
External policy services
Decision aggregation
```

Repetir todo este trabajo constantemente puede generar:

```text
N+1 authorization queries
repeated graph traversal
repeated container resolution
unnecessary external calls
high latency
database load
```

Sin embargo, reutilizar decisiones incorrectamente puede causar:

```text
stale privileges
cross-tenant leaks
revocation delays
subject confusion
security-context bypass
```

El objetivo será optimizar sin degradar las garantías de autorización.

---

# 3. Principio arquitectónico

VoltStack separará cuatro niveles principales:

```text
LEVEL 1
Compiled Authorization Metadata

LEVEL 2
Structural Resolution / Plan Cache

LEVEL 3
Request-Scoped Memoization

LEVEL 4
Cross-Request Decision Cache
```

La confianza y complejidad disminuyen conforme nos acercamos al nivel 4.

---

# 4. Regla de seguridad

La política recomendada será:

```text
Cache structure aggressively.

Memoize repeated decisions per request.

Cache final authorization decisions
across requests only when explicitly safe.
```

---

# 5. Tipos de cache

El sistema deberá distinguir al menos:

```text
Policy Metadata Cache
Gate Metadata Cache
Ability Registry Cache
Authorization Metadata Cache
Authorization Plan Cache
Role Cache
Permission Cache
Relationship Cache
Attribute Cache
Request Decision Memoization
Cross-Request Decision Cache
```

---

# 6. Metadata Cache

Incluye:

```text
PolicyDescriptors
GateDescriptors
AbilityDescriptors
Compiled Controller Metadata
Compiled Route Metadata
Strategy Metadata
```

Este tipo de información es:

```text
structural
mostly immutable
principal-independent
subject-instance-independent
```

y por tanto es excelente candidata para cache.

---

# 7. Ejemplo de Metadata Cache

```text
Invoice
    ↓
InvoicePolicy

Ability:
update

Policy Method:
update()

Parameter Mapping:
Principal
Subject
Context
```

Puede compartirse entre muchos requests.

---

# 8. Metadata Cache Lifetime

En producción podrá vivir:

```text
process-wide
```

bajo FrankenPHP.

También podrá almacenarse como:

```text
compiled PHP cache
```

cargado mediante Opcache.

---

# 9. Registry Cache

Los registries inmutables definidos anteriormente:

```text
PolicyRegistry
GateRegistry
AbilityRegistry
DecisionStrategyRegistry
```

son esencialmente caches estructurales.

No deberán contener decisiones runtime.

---

# 10. Plan Cache

Puede cachearse:

```text
What evaluators apply?
In what order?
Under what strategy?
```

Ejemplo:

```text
Ability:
update

Subject Class:
Invoice

Channel:
web

Tenant Context:
present
```

→

```text
TenantIsolationPolicy
PermissionEvaluator
InvoicePolicy
CompliancePolicy
```

---

# 11. Plan Cache ≠ Decision Cache

```text
Plan Cache:
which evaluators should execute
```

```text
Decision Cache:
what final authorization result was returned
```

El primero es significativamente más seguro.

---

# 12. AuthorizationPlanTemplate

La unidad preferida de plan cache será:

```text
AuthorizationPlanTemplate
```

que contiene solo información estructural.

---

# 13. Runtime Binding

El template se enlaza con:

```text
Principal
Subject instance
Tenant
AuthorizationContext
```

en cada request.

---

# 14. Plan Cache Key

Una clave conceptual podrá incluir:

```text
Ability canonical name
Subject type/class
Authorization phase
Context shape
Registry version
Strategy ID
```

---

# 15. Ejemplo

```text
authz-plan:
v18:
resource:
Invoice:
update:
web:
tenant:
deny_overrides
```

---

# 16. Lo que no debe incluir normalmente

No:

```text
User#42
Invoice#928
```

si esos valores no cambian qué evaluadores estructurales aplican.

---

# 17. Context Shape

El plan puede depender de:

```text
hasTenant=true
channel=web
hasSecurityContext=true
principalType=UserPrincipal
```

No necesariamente de los valores concretos.

---

# 18. Plan Cache Invalidation

Debe invalidarse si cambia:

```text
Policy Registry
Gate Registry
Ability Registry
Authorization configuration
Strategy configuration
Metadata schema
Compiled Controller/Route metadata
```

---

# 19. Versioned Plan Cache

La técnica recomendada será usar:

```text
AuthorizationRegistryVersion
```

o fingerprint global.

Una nueva versión produce nuevas keys.

---

# 20. Request Memoization

La forma más segura de reutilizar resultados dinámicos será dentro de una misma unidad de trabajo.

Ejemplo:

```php
$user->can('update', $invoice);
$user->can('update', $invoice);
$user->can('update', $invoice);
```

debería poder evaluarse una sola vez.

---

# 21. Scope

Request memoization vivirá únicamente durante:

```text
HTTP request
CLI command execution
Queue job
Event handling unit
```

---

# 22. AuthorizationMemoizer

Contrato conceptual:

```php
interface AuthorizationMemoizerInterface
{
    public function get(
        AuthorizationMemoizationKey $key
    ): ?DecisionResult;

    public function put(
        AuthorizationMemoizationKey $key,
        DecisionResult $result
    ): void;
}
```

---

# 23. Memoization Key

Deberá representar todos los inputs relevantes de la decisión.

Conceptualmente:

```text
Principal Fingerprint
Ability
Subject Fingerprint
Tenant Fingerprint
Context Security Fingerprint
Authorization Version
```

---

# 24. Principal Fingerprint

No deberá ser solamente:

```text
user ID
```

si el Principal puede cambiar de:

```text
authentication strength
impersonation
delegated grant
security context
```

dentro de la misma ejecución.

---

# 25. PrincipalAuthorizationFingerprint

Podrá contener:

```text
principal type
principal ID
authorization version
actor/effective principal state
```

---

# 26. Authorization Version

El RBAC subsystem podrá mantener:

```text
authorization_version
```

por Principal.

Ejemplo:

```text
User#42
authorization_version=18
```

Cuando cambian Roles/Permissions:

```text
18 → 19
```

---

# 27. Benefit

Una key que incluye:

```text
principal 42 version 18
```

no reutilizará grants después de la revocación cuando la nueva ejecución use versión 19.

---

# 28. Subject Fingerprint

Para resource authorization podrá incluir:

```text
subject type
subject identifier
authorization-relevant version
```

---

# 29. Problema de solo usar ID

Si:

```text
Invoice#928
```

cambia:

```text
owner
tenant
status
classification
amount
```

una decisión previa puede dejar de ser válida.

---

# 30. SubjectAuthorizationVersion

Idealmente entidades sensibles podrán exponer:

```text
authorization_version
```

o un fingerprint de estado relevante.

---

# 31. Alternativa

Si no existe versionado confiable:

```text
cross-request Decision Cache
```

deberá evitarse para esa Policy.

Request memoization sigue siendo normalmente segura si el Subject no cambia durante la unidad de trabajo.

---

# 32. Mutable Subject inside request

Incluso request memoization puede ser incorrecta si:

```text
authorize
mutate authorization-relevant subject fields
authorize again
```

en la misma request.

---

# 33. Example

```text
Invoice status:
draft

can approve?
→ DENY

status changed:
pending

can approve?
```

No se debe reutilizar automáticamente el primer resultado.

---

# 34. Subject Mutation Awareness

Podrán existir varias estrategias:

```text
immutable domain objects
authorization version increment
memoization invalidation
explicit reauthorization
```

---

# 35. Recommended Rule

Request memoization será segura solo mientras:

```text
authorization-relevant inputs
remain stable
```

---

# 36. Decision Reuse Policy

Cada evaluator/Policy podrá declarar metadata sobre reutilización.

Ejemplo:

```php
enum DecisionReusePolicy: string
{
    case None = 'none';
    case Request = 'request';
    case Process = 'process';
    case Distributed = 'distributed';
}
```

---

# 37. Default

Recomendación:

```text
Request
```

para decisiones normales.

---

# 38. None

Utilizar para Policies dependientes de:

```text
time
nonce
single-use capability
rapidly changing external state
security counter
transaction state
```

---

# 39. Process

Solo para decisiones que permanecen válidas durante el proceso bajo versioning fuerte.

Debe ser raro.

---

# 40. Distributed

Para decisiones cross-request/cross-node almacenables en:

```text
Redis
distributed cache
```

requiere el nivel más alto de garantías.

---

# 41. Policy Cacheability Metadata

Un descriptor podrá incluir:

```text
memoizable
crossRequestCacheable
cacheTtl
relevantContextKeys
subjectVersionStrategy
principalVersionStrategy
```

---

# 42. Safe Default

Si no existe metadata explícita:

```text
crossRequestCacheable=false
```

---

# 43. Request Memoization by Default

Sí podrá habilitarse para la mayoría de decisiones si la unidad de trabajo mantiene estado estable.

---

# 44. Gate Memoization

Un Gate como:

```text
admin.access
```

puede memoizarse por request.

---

# 45. Resource Policy Memoization

```text
User#42
update
Invoice#928
```

también puede memoizarse por request.

---

# 46. Global Policy Memoization

Ejemplo:

```text
SuspendedPrincipalPolicy
```

podría evaluarse una vez por request y reutilizarse para múltiples abilities si su metadata declara independencia de Ability/Subject.

---

# 47. Evaluator Memoization vs Final Decision Memoization

También deberán distinguirse.

```text
Evaluator Memoization:
reuse result of TenantMembershipPolicy
```

```text
Decision Memoization:
reuse complete final answer
```

---

# 48. Evaluator-level Benefit

Si 100 recursos requieren:

```text
TenantMembershipEvaluator
```

este resultado puede evaluarse una vez.

---

# 49. Example

```text
User#42 member Tenant#7?
→ GRANT
```

La respuesta puede reutilizarse durante el request para 500 Invoice checks.

---

# 50. Permission Memoization

Igualmente:

```text
User#42 has invoice.update in Tenant#7?
```

se puede resolver una vez por request.

---

# 51. Relationship Memoization

ReBAC puede ser costoso.

Una consulta:

```text
User#42 member Organization#15?
```

puede memoizarse.

---

# 52. Attribute Memoization

ABAC attribute providers caros:

```text
device trust
risk score
GeoIP
```

también pueden memoizarse durante una decisión o request cuando sean estables.

---

# 53. Memoization Layers

Un request puede tener:

```text
Principal Grant Memo
Relationship Memo
Attribute Memo
Evaluator Memo
Final Decision Memo
```

---

# 54. Avoid Duplicated Caches

Estos layers deberán tener ownership claro para evitar:

```text
invalidation complexity
memory bloat
conflicting answers
```

---

# 55. AuthorizationSession

Una buena ubicación será:

```text
AuthorizationSession
```

request-scoped.

---

# 56. AuthorizationSession Contents

Conceptualmente:

```text
decision memo
evaluator memo
role memo
permission memo
relationship memo
context fingerprints
nested authorization stack
trace collector
```

---

# 57. Lifecycle

```text
Start unit of work
    ↓
Create AuthorizationSession
    ↓
Perform checks
    ↓
Reuse memoized values
    ↓
Destroy session
```

---

# 58. FrankenPHP Safety

`AuthorizationSession` nunca deberá ser:

```text
process-wide singleton state
```

---

# 59. Session Provider

Un shared service podrá resolver la sesión actual desde un:

```text
request-scoped runtime context
```

---

# 60. Cleanup

Después de cada:

```text
HTTP request
queue job
CLI operation
```

la sesión debe eliminarse.

---

# 61. Cross-Request Decision Cache

Este mecanismo será opcional y conservador.

---

# 62. Use Cases

Puede ser útil para:

```text
very expensive external authorization
stable role mappings
read-heavy high-volume APIs
large ReBAC graphs
```

---

# 63. Risk

Un cached:

```text
GRANT
```

es más peligroso que un cached:

```text
DENY
```

si los privilegios fueron revocados.

Pero un stale DENY también puede causar problemas funcionales.

---

# 64. Positive Cache

Almacena:

```text
GRANT
```

---

# 65. Negative Cache

Almacena:

```text
DENY
```

---

# 66. Different TTLs

VoltStack podrá permitir:

```text
grant TTL
deny TTL
```

diferentes.

---

# 67. Recommendation

Para autorización sensible:

```text
GRANT TTL <= DENY TTL
```

o no cachear GRANT cross-request.

---

# 68. Example

```text
GRANT TTL:
5 seconds

DENY TTL:
30 seconds
```

si el dominio tolera esa consistencia.

---

# 69. Revocation-sensitive Abilities

Abilities como:

```text
system.deploy
user.impersonate
permission.grant
wire.transfer
```

no deberían usar long-lived positive decision cache.

---

# 70. Ability Cache Policy

`AbilityDescriptor` podrá declarar:

```text
decisionCachePolicy
```

---

# 71. Risk-based Default

Ejemplo:

```text
read-only low-risk
→ short cross-request cache possible

administrative
→ request memo only

critical
→ no final decision cache
```

---

# 72. Security Critical Rule

No debe depender solo de categorías automáticas.

La aplicación/framework debe poder marcar explícitamente.

---

# 73. Cross-Request Cache Key

Una key robusta podría incluir:

```text
PrincipalAuthorizationFingerprint
Ability
SubjectAuthorizationFingerprint
TenantFingerprint
SecurityContextFingerprint
AuthorizationRegistryVersion
RelevantContextFingerprint
```

---

# 74. Example

```text
authz-decision:
registry-v18:
principal:user:42:auth-v19:
tenant:7:
ability:invoice.update:
subject:invoice:928:auth-v4:
security:mfa2:
ctx:8f4e...
```

---

# 75. Key Length

Internamente podrá hash-earse:

```text
SHA-256 / framework-safe hash
```

después de canonicalizar componentes.

---

# 76. Canonicalization

El mismo input semántico deberá generar la misma key.

---

# 77. Never Serialize Arbitrary Objects into Key

No utilizar directamente:

```php
serialize($request);
```

porque puede incluir:

```text
mutable state
sensitive state
unstable ordering
huge payloads
```

---

# 78. Cache Key Builder

Podrá existir:

```php
interface AuthorizationCacheKeyBuilderInterface
{
    public function decisionKey(
        AuthorizationRequest $request,
        DecisionCachePolicy $policy
    ): AuthorizationDecisionCacheKey;
}
```

---

# 79. Relevant Context

No toda la request debe formar parte de la key.

Solo valores relevantes declarados.

---

# 80. Example Relevant Context

Una Policy puede declarar:

```text
tenant
security.mfa_level
region
channel
```

---

# 81. Context Metadata

```php
#[AuthorizationDependsOnContext([
    'tenant',
    'security.mfa_level',
])]
```

podría existir en el futuro.

Más probablemente será metadata del descriptor compilado.

---

# 82. Unknown Context Dependency

Si el framework no puede demostrar qué Context afecta una Policy:

```text
crossRequestCacheable=false
```

---

# 83. Principle

```text
Unknown dependency
=
do not reuse across requests.
```

---

# 84. Time Dependency

Policies que usan:

```php
$this->clock->now()
```

son temporalmente variables.

---

# 85. Example

```text
users may edit invoice
only during business hours
```

---

# 86. Cache Policy

Estas decisiones necesitan:

```text
time bucket
```

en key o:

```text
no cross-request cache
```

---

# 87. Time Bucketing

Ejemplo:

```text
minute=2026-08-19T14:11
```

puede ser parte de la key si se desea.

Pero aumenta complejidad.

---

# 88. Recommendation

No cachear cross-request Policies time-sensitive salvo necesidad probada.

---

# 89. External State Dependency

Ejemplo:

```text
fraud score
external IAM
license status
```

requiere versioning/TTL apropiado.

---

# 90. External Evaluator Cache

Podrá existir cerca del adapter externo.

Ejemplo:

```text
OPA result cache
relationship service cache
identity provider claims cache
```

---

# 91. Ownership

La cache de un external evaluator deberá ser responsabilidad de su integration adapter cuando tenga semántica específica.

---

# 92. Core Awareness

El Authorization Core solo necesita conocer:

```text
result freshness
failure behavior
cacheability
```

---

# 93. External Cache Failure

Cache unavailable:

```text
evaluate source directly
```

si es posible.

Nunca:

```text
assume GRANT
```

---

# 94. Stale-While-Revalidate

No se recomienda como default para autorización.

Un stale GRANT es peligroso.

---

# 95. Stale-on-Error

Igualmente:

```text
external PDP unavailable
use stale allow
```

debe considerarse inseguro por defecto.

---

# 96. Default

```text
external evaluator failure
→ fail closed
```

---

# 97. Permission Cache

El RBAC system puede cachear:

```text
effective permissions
```

cross-request.

---

# 98. Permission Cache Key

Debe incluir:

```text
Principal
Scope/Tenant
Authorization Version
Provider Version
```

---

# 99. Example

```text
authz-grants:
user:42:
tenant:7:
version:19
```

---

# 100. Role Cache

Igual.

---

# 101. Grant Cache vs Decision Cache

Esto es muy importante:

```text
Grant Cache
=
what roles/permissions does Principal currently have?
```

```text
Decision Cache
=
may Principal perform ability on subject?
```

Grant Cache suele ser más reusable.

---

# 102. Why

Una permission:

```text
invoice.update
```

puede seguir siendo válida aunque el resultado sobre:

```text
Invoice#928
```

cambie.

---

# 103. Recommended Optimization Order

Antes de cachear decisiones completas:

```text
1. cache metadata
2. cache plans
3. memoize roles/permissions
4. memoize relationships
5. memoize evaluators
6. memoize final decisions per request
7. only then consider cross-request final decision cache
```

---

# 104. ReBAC Cache

Relaciones podrán cachearse por:

```text
subject relation object
```

---

# 105. Example

```text
user:42
member
organization:15
```

---

# 106. Relationship Version

Si el graph cambia, una cache debe invalidarse.

Podrá existir:

```text
relationship_version
```

por:

```text
principal
organization
tenant
graph partition
```

---

# 107. Graph Cache Versioning

Para graphs muy grandes puede usarse:

```text
tenant relationship version
```

aunque esto invalide más entradas.

---

# 108. Trade-off

```text
coarse version
=
easy invalidation
more cache misses
```

```text
fine version
=
better hit rate
complex invalidation
```

---

# 109. ABAC Attribute Cache

Attribute provider podrá cachear:

```text
principal.clearance
subject.classification
risk score
```

---

# 110. Attribute Freshness

Cada attribute puede tener TTL distinto.

---

# 111. Security Attributes

Ejemplo:

```text
MFA verified
```

debe obtenerse de:

```text
current SecurityContext
```

no de long-lived distributed cache.

---

# 112. Device Trust

Puede tener una expiración propia.

---

# 113. GeoIP

Puede cachearse por request/IP.

---

# 114. Subject State Cache

No deberá duplicar indiscriminadamente el ORM Identity Map.

Si el Subject ya está en memoria, usar esa instancia.

---

# 115. Policy Result Memoization

El resultado de:

```text
InvoicePolicy::update
```

puede memoizarse con su propio fingerprint.

---

# 116. Composite Final Decision Memo

Una final decision solo puede reutilizarse si todos los evaluadores que la componen siguen siendo válidos.

---

# 117. Weakest Dependency Rule

La cacheabilidad de una decisión agregada estará limitada por el evaluator menos cacheable.

Ejemplo:

```text
TenantPolicy → request cacheable
PermissionEvaluator → distributed cacheable
InvoicePolicy → request cacheable
RiskEvaluator → no cache
```

Final decision:

```text
no cross-request cache
```

---

# 118. DecisionCachePolicyAggregator

Podrá existir:

```text
Decision Cache Policy
=
intersection of evaluator cache capabilities
```

---

# 119. Example

```text
Evaluator A:
distributed 60s

Evaluator B:
distributed 10s

Evaluator C:
request-only
```

Final:

```text
request-only
```

---

# 120. Short-Circuit and Cacheability

Si una decisión terminó temprano, solo los evaluadores realmente decisivos participaron.

¿Puede cachearse basándose solo en ellos?

---

# 121. Conservative Rule

No asumir que evaluadores no ejecutados seguirían siendo irrelevantes si el plan/config cambia.

La key incluye:

```text
Registry/Plan version
```

lo cual ayuda.

---

# 122. Under Same Plan Version

Si la estrategia garantiza que:

```text
first critical DENY
```

es final, puede reutilizarse según cacheability de ese camino.

---

# 123. Decision Provenance

El cached result deberá conocer:

```text
plan version
strategy
decisive evaluators
cache policy
expiresAt
```

internamente.

---

# 124. AuthorizationCachedDecision

Conceptualmente:

```php
final readonly class AuthorizationCachedDecision
{
    public function __construct(
        public DecisionResult $result,
        public string $planFingerprint,
        public string $registryVersion,
        public int $createdAt,
        public ?int $expiresAt,
    ) {}
}
```

---

# 125. Do Not Cache Full Trace by Default

El cache no deberá almacenar:

```text
stack traces
sensitive evaluator metadata
full subject snapshots
```

---

# 126. Cached Reason

Puede conservar:

```text
reasonCode
```

si es necesario.

No necesariamente el mensaje humano completo.

---

# 127. Security Context Fingerprint

Una decisión puede depender de:

```text
authenticated
MFA level
device trust
impersonation
token scope
session age
```

---

# 128. Example

Un GRANT con:

```text
MFA level 2
```

no debe reutilizarse después de que:

```text
MFA assurance expires
```

---

# 129. Context Version

`SecurityContext` podrá exponer:

```text
authorizationFingerprint()
```

---

# 130. Tenant Fingerprint

Debe incluir:

```text
Tenant ID
Tenant authorization/status version
```

si:

```text
tenant suspended
tenant changed to read-only
```

puede afectar decisiones.

---

# 131. TenantOperationalVersion

Ejemplo:

```text
Tenant#7
security_version=8
```

se incrementa al cambiar:

```text
status
security configuration
authorization settings
```

---

# 132. Revocation

El gran problema del cache de autorización es la revocación.

---

# 133. Revocation Sources

Ejemplos:

```text
role revoked
permission revoked
membership revoked
subject ownership changed
tenant suspended
MFA expired
support session revoked
relationship removed
```

---

# 134. Invalidation Strategy

VoltStack deberá soportar:

```text
version-based invalidation
event-based invalidation
TTL expiration
request lifecycle expiration
```

---

# 135. Version-Based Invalidation

Es el mecanismo preferido cuando es viable.

---

# 136. Benefits

No necesita encontrar y borrar todas las keys antiguas.

Solo:

```text
new version
→ new key namespace
```

---

# 137. Old Entries

Podrán expirar naturalmente.

---

# 138. Principal Version

Incrementar cuando cambian:

```text
Roles
Permissions
Memberships
Delegated grants
```

---

# 139. Subject Version

Incrementar cuando cambian campos relevantes a autorización.

---

# 140. Tenant Version

Incrementar cuando cambian:

```text
status
security policy
membership model
```

---

# 141. Relationship Version

Incrementar cuando graph relevante cambia.

---

# 142. Policy Registry Version

Incrementar/recompilar al cambiar código/config.

---

# 143. Composite Fingerprint

Una final key combina esas versiones.

---

# 144. Event-Based Invalidation

Eventos:

```text
RoleRevoked
PermissionRevoked
TenantMembershipRevoked
TenantSuspended
ResourceOwnershipChanged
RelationshipRemoved
```

pueden eliminar entradas.

---

# 145. Best Use

Event invalidation es útil para:

```text
distributed caches
large TTLs
```

---

# 146. Race Consideration

Un evento puede llegar después de que una request haya leído un cached GRANT.

---

# 147. Strong Revocation

Para operaciones críticas:

```text
cache bypass
```

o version lookup fuerte puede ser necesario.

---

# 148. Revocation Consistency Modes

Podrá existir:

```php
enum AuthorizationConsistencyMode: string
{
    case Strong = 'strong';
    case Bounded = 'bounded';
    case Request = 'request';
}
```

---

# 149. Strong

No utilizar decision cache cross-request o validar versiones contra fuente autoritativa.

---

# 150. Bounded

Permite una ventana máxima de stale data.

Ejemplo:

```text
5 seconds
```

---

# 151. Request

Solo memoization durante la unidad de trabajo.

---

# 152. Default

Para decisiones finales:

```text
Request
```

será el default recomendado.

---

# 153. Ability Consistency Requirements

Una Ability podrá declarar:

```text
consistency=strong
```

para:

```text
system.deploy
payment.release
user.impersonate
```

---

# 154. Plan enforcement

El `AuthorizationPlanner` o `DecisionCachePolicyResolver` deberá impedir cache incompatible con ese requisito.

---

# 155. Cache Bypass

Una llamada podrá solicitar:

```text
fresh authorization
```

---

# 156. API conceptual

```php
Authorization::fresh()
    ->authorize('approve', $invoice);
```

o internamente:

```text
DecisionReuseMode::Fresh
```

---

# 157. Use Cases

```text
before financial commit
before destructive action
after privilege change
security-sensitive revalidation
```

---

# 158. `fresh()` must not bypass structural caches

Fresh significa:

```text
do not reuse dynamic authorization result
```

No necesita ignorar:

```text
compiled metadata
plan cache
```

---

# 159. Important Distinction

```text
Fresh Decision
```

puede seguir usando:

```text
PolicyRegistry
PlanTemplate
compiled descriptors
```

---

# 160. Memoization Opt-Out

Una Policy/evaluator puede declarar:

```text
memoizable=false
```

---

# 161. Example

```text
SingleUseTokenPolicy
```

deberá evaluar siempre.

---

# 162. Single-Use Capability

Si el check consume una capability, idealmente authorization + consume debe ser atómico.

---

# 163. Side Effects Warning

Policies no deberían consumir tokens directamente.

Mejor:

```text
validate capability
then transactional consume in domain operation
```

con revalidation cuando corresponda.

---

# 164. Nested Authorization

Si una Policy realiza otro authorization check:

```text
outer authorization
    ↓
inner authorization
```

memoization puede evitar loops/repetición.

---

# 165. Circular Detection Still Required

Cache hit no sustituye:

```text
AuthorizationExecutionStack
```

para detectar recursion antes de crear una entrada válida.

---

# 166. In-Progress Entry

No deberá cachearse temporalmente como GRANT/DENY antes de finalizar la decisión.

---

# 167. Promise/Future Deduplication

En runtime concurrente dentro de la misma request podría existir:

```text
single-flight
```

para evitar dos evaluaciones idénticas simultáneas.

No es requisito V1.

---

# 168. Parallel Authorization

Si futuras features ejecutan checks concurrentes, el memoizer deberá ser concurrency-safe dentro de la execution scope.

---

# 169. Cache Backend Interface

Cross-request:

```php
interface AuthorizationDecisionCacheInterface
{
    public function get(
        AuthorizationDecisionCacheKey $key
    ): ?AuthorizationCachedDecision;

    public function put(
        AuthorizationDecisionCacheKey $key,
        AuthorizationCachedDecision $decision
    ): void;

    public function forget(
        AuthorizationDecisionCacheKey $key
    ): void;
}
```

---

# 170. Backends

Podrán existir:

```text
Null
Array
APCu
Redis
VoltStack Cache abstraction
```

---

# 171. Recommendation

Authorization no deberá crear su propia infraestructura de cache distribuida.

Debe utilizar:

```text
Quantum Cache
```

o el cache central de VoltStack.

---

# 172. Namespace

Todas las keys deberán usar un namespace reservado:

```text
voltstack:authz:
```

---

# 173. Environment Isolation

Debe incluir:

```text
application/environment namespace
```

para evitar mezclar:

```text
staging
production
tests
```

---

# 174. Multi-App Worker Isolation

Si un proceso sirve múltiples aplicaciones, cada una tendrá namespace separado.

---

# 175. Multi-Tenant Isolation

Tenant debe ser parte de la key cuando sea relevante.

---

# 176. Cache Poisoning

Input del usuario no deberá convertirse directamente en key sin canonicalization.

---

# 177. Key Injection

Evitar construir keys ambiguas mediante concatenación sin escaping.

---

# 178. Structured Key

Preferido:

```text
canonical parts
    ↓
stable encoding
    ↓
hash
```

---

# 179. Cache Value Validation

Al leer una entrada debe validarse:

```text
schema version
registry version
expiration
decision type
```

---

# 180. Corrupt Cache

Una entrada corrupta debe tratarse como:

```text
cache miss
```

y registrarse.

Nunca producir GRANT.

---

# 181. Deserialize Safety

No utilizar:

```php
unserialize()
```

sobre datos no confiables.

Preferir estructuras seguras.

---

# 182. Distributed Cache Trust

Incluso cache interno debe considerarse infraestructura.

Manipulación de cache no debe permitir ejecución de clases arbitrarias.

---

# 183. Cache Stampede

Autorizaciones caras pueden generar stampede.

---

# 184. Mitigation

Podrán usarse:

```text
small locking
single-flight
jittered TTL
```

cuando no afecten latencia/safety.

---

# 185. Do Not Serve Stale Grant During Stampede

Por defecto no usar stale GRANT como fallback.

---

# 186. Jitter

TTL puede variar ligeramente para evitar expiración simultánea.

---

# 187. Negative Cache

Puede ser útil para:

```text
unknown permission
missing membership
missing relationship
```

---

# 188. Risk

Un DENY cacheado demasiado tiempo puede persistir después de que se otorgó acceso.

---

# 189. Negative TTL

Debe ser pequeño si grants cambian frecuentemente.

---

# 190. Permission Negative Cache

Con versioning de Principal puede ser mucho más seguro.

---

# 191. Example

```text
User#42 auth version 18
permission missing
```

Si se concede permission:

```text
version → 19
```

la negative cache de v18 ya no aplica.

---

# 192. All-Abstain Cache

Una final:

```text
authorization.all_abstained
```

puede cachearse solo según metadata estructural/dinámica apropiada.

---

# 193. No-Evaluator Result

Si no hay evaluator por configuración:

```text
no evaluator
```

es esencialmente estructural.

Podría detectarse en Plan Cache.

---

# 194. Strict Mode

Preferible detectar:

```text
unknown ability
no handler
```

en compile/lint en lugar de cachear runtime errors.

---

# 195. Exception Caching

No deberán cachearse:

```text
PolicyExecutionException
DB exception
timeout
configuration failure
```

como DENY ordinario salvo política muy explícita.

---

# 196. Why

Un fallo temporal no es una decisión semántica.

---

# 197. External Failure Cache

Circuit breakers pueden cachear el estado del servicio, pero esto pertenece al resilience layer.

Authorization sigue:

```text
failure → fail closed
```

---

# 198. Result Provenance

Un cache hit deberá poder marcarse internamente:

```text
decisionSource=cache
```

---

# 199. Observability

Tracing podrá mostrar:

```text
Authorization:
cache hit

Level:
request memoization

Decision:
GRANT
```

---

# 200. Do Not Change Semantics

Un cache hit debe comportarse igual que evaluación fresh.

---

# 201. Cached Reason Metadata

Tracing puede indicar:

```text
original_decisive_evaluator
```

si se almacenó de manera segura.

---

# 202. Request Memo Trace

Ejemplo:

```text
InvoicePolicy::view
first check:
MISS → 0.4 ms

next 19 checks:
HIT
```

---

# 203. Metrics

Posibles:

```text
authorization.cache.plan.hit
authorization.cache.plan.miss

authorization.memo.decision.hit
authorization.memo.evaluator.hit

authorization.cache.decision.hit
authorization.cache.decision.miss
authorization.cache.decision.stale
authorization.cache.invalidation
```

---

# 204. Cache Hit Ratio

Debe observarse por capa.

Una única métrica global puede ser engañosa.

---

# 205. Security Metrics

```text
authorization.cache.version_mismatch
authorization.cache.corrupt_entry
authorization.cache.revocation_bypass_prevented
```

---

# 206. Debugging

Tooling podrá explicar:

```text
why was this decision reusable?
```

---

# 207. Cache Explain

Ejemplo:

```text
Decision Cache Policy:
REQUEST_ONLY

Reasons:
- InvoicePolicy depends on mutable subject state
- CompliancePolicy not cross-request cacheable
```

---

# 208. Tool Command

Futuro:

```text
volt authorization:cache-explain \
App\Domain\Invoice update
```

---

# 209. Output

```text
Plan Cache:
YES

Evaluator Memoization:
YES

Final Request Memoization:
YES

Cross-Request Decision Cache:
NO

Blocking evaluator:
FinancialCompliancePolicy
```

---

# 210. Cache Inspection

```text
volt authorization:cache-status
```

podrá mostrar:

```text
Registry version
Plan cache entries
Decision cache backend
Hit ratio
Configured TTLs
```

sin revelar datos sensibles.

---

# 211. Cache Clear

```text
volt authorization:cache-clear
```

podrá limpiar metadata/decision caches según flags.

---

# 212. Production Deployment

Después de actualizar Policies:

```text
new Registry Version
    ↓
old decision keys obsolete
```

---

# 213. Zero-Downtime Deployment

Dos versiones de la aplicación pueden coexistir.

Por tanto cache key deberá incluir:

```text
authorization schema/registry version
```

---

# 214. Benefit

App version A no reutiliza decision cache de App version B si su Policy behavior difiere.

---

# 215. Rolling Deployments

Las entradas antiguas pueden expirar naturalmente.

---

# 216. Shared Redis

Version namespaces son esenciales.

---

# 217. Request Memo Size

Una página con miles de resources podría crear gran número de entries.

---

# 218. Memory Bound

El memoizer podrá tener:

```text
max_entries
```

por request.

---

# 219. Eviction

Podrá usar una estrategia simple:

```text
bounded map
LRU
```

si es necesario.

---

# 220. Default

Para la mayoría de requests el número será pequeño.

No introducir LRU complejo sin benchmarks.

---

# 221. Cache Large Subjects

Nunca almacenar el Subject completo en la memo key/value.

Solo fingerprints.

---

# 222. DecisionResult Size

Mantenerlo compacto.

---

# 223. Trace Separation

Full tracing debe estar separado de memoized decision data.

---

# 224. Collection Authorization

Un listado puede evaluar:

```text
view
```

sobre cientos de resources.

---

# 225. Request Memoization Limitation

Si cada Subject ID es distinto, final decision memo no ayuda demasiado.

---

# 226. Better Optimizations

Usar:

```text
memoized global evaluators
permission preloading
batch relationship checks
query scopes
batch policies
```

---

# 227. Important Principle

No usar cache para ocultar un algoritmo N+1 mal diseñado.

---

# 228. Batch Authorization

Para:

```text
500 invoices
```

preferir:

```text
batch authorization
```

cuando sea posible.

---

# 229. Cache + Batch

Ambos mecanismos pueden coexistir.

---

# 230. `viewAny` Cache

Una decisión como:

```text
may list invoices?
```

puede memoizarse una vez.

Pero no reemplaza:

```text
which invoices are visible?
```

---

# 231. Query Scope Cache

Un Authorization Query Scope estructural puede cachearse como plan, no necesariamente como resultados de datos.

---

# 232. Transaction Boundaries

Dentro de una transacción, autorización puede depender de datos modificados aún no committed.

---

# 233. Distributed Decision Cache Warning

No debe almacenar una decisión basada en:

```text
uncommitted state
```

---

# 234. Request Memoization

Incluso request memo debe invalidarse si la transacción cambia inputs relevantes.

---

# 235. Post-Commit

Eventos de invalidación deben emitirse:

```text
after commit
```

---

# 236. Rollback

No incrementar versions persistentemente antes de commit si la transacción puede rollback.

---

# 237. Cache Invalidation Transactionality

Authorization cache version changes deben ser coherentes con datos de grants.

---

# 238. Example Role Grant

```text
BEGIN

assign role
increment auth version

COMMIT

publish invalidation event
```

---

# 239. Read Replica Lag

Si authorization grants se leen desde replica con lag, una revocación puede tardar en observarse.

---

# 240. Security Recommendation

Para revocation-sensitive checks:

```text
read authoritative primary
```

o usar un grant store con consistencia adecuada.

---

# 241. Cache Does Not Fix Replica Lag

Debe considerarse separadamente.

---

# 242. Multi-Tenant Versioning

Principal authorization version puede ser:

```text
global
```

o:

```text
per tenant
```

---

# 243. Per-Tenant Version

Ejemplo:

```text
User#42
Tenant#7 auth-v12
Tenant#9 auth-v4
```

---

# 244. Benefit

Cambiar permiso en Tenant#7 no invalida caches del Tenant#9.

---

# 245. Complexity

Requiere almacenamiento más detallado.

---

# 246. Recommendation

Para sistemas multi-tenant grandes:

```text
per-scope authorization version
```

es atractivo.

---

# 247. Tenant Version + Principal Version

Key:

```text
principal scope version
+
tenant security version
```

---

# 248. Subject Version

Igualmente podrá ser per-resource.

---

# 249. Version Overflow

Usar enteros suficientemente grandes o UUID/version tokens.

---

# 250. Hash Version

También puede usarse:

```text
opaque authorization version token
```

---

# 251. Cache Policy Object

Conceptualmente:

```php
final readonly class AuthorizationDecisionCachePolicy
{
    public function __construct(
        public DecisionReusePolicy $reuse,
        public ?int $grantTtlSeconds = null,
        public ?int $denyTtlSeconds = null,
        public AuthorizationConsistencyMode $consistency =
            AuthorizationConsistencyMode::Request,
        public array $contextDependencies = [],
    ) {}
}
```

---

# 252. Policy Resolution

El cache policy final se derivará de:

```text
Ability metadata
Evaluator metadata
Strategy
Security classification
Request override
```

---

# 253. Precedence

Recomendación:

```text
Explicit Fresh Requirement
    ↓
Critical Ability Restrictions
    ↓
Evaluator Cache Constraints
    ↓
Ability Cache Policy
    ↓
Global Defaults
```

---

# 254. No Caller Downgrade

Un caller no deberá poder pedir:

```text
cache longer
```

que lo permitido por la Ability/evaluators.

---

# 255. Caller Can Be Stricter

Sí puede solicitar:

```text
fresh
```

o menor reutilización.

---

# 256. Cache Policy Intersection

Ejemplo:

```text
Global default:
distributed 60s

Ability:
request only

Caller:
fresh
```

Resultado:

```text
fresh
```

---

# 257. AuthorizationManager Flow

Conceptualmente:

```text
AuthorizationRequest
      ↓
Resolve cache policy
      ↓
Check request memo
      ↓ hit
return

      ↓ miss
Check cross-request cache if allowed
      ↓ hit
store request memo
return

      ↓ miss
Resolve/bind plan
      ↓
Execute
      ↓
Decision
      ↓
Store request memo
      ↓
Store cross-request cache if allowed
      ↓
Return
```

---

# 258. Structural Cache Order

Antes de ejecutar:

```text
Plan Cache
```

se consulta independientemente del decision cache.

---

# 259. Full Flow

```text
Request
 │
 ↓
Canonicalize
 │
 ↓
Decision Reuse Policy
 │
 ├── Request Memo?
 │        ↓
 │       HIT ───────────────→ Decision
 │
 ├── Distributed Decision Cache?
 │        ↓
 │       HIT ───────────────→ Decision
 │
 ↓
Plan Template Cache
 │
 ↓
Authorization Execution
 │
 ↓
Decision Manager
 │
 ↓
Final Decision
 │
 ├── Store Request Memo
 │
 └── Store Cross-Request if safe
```

---

# 260. Finalizer Before Cache

Solo deberá cachearse:

```text
finalized DecisionResult
```

no un estado intermedio `ABSTAIN` si el public result sería DENY.

---

# 261. Internal Evaluator Cache

Evaluator memoization sí puede guardar:

```text
ABSTAIN
```

si semánticamente válido.

---

# 262. Default Deny

Final result:

```text
ABSTAIN
→ DENY
```

y ese DENY puede memoizarse según policy.

---

# 263. Decision Strategy Change

Registry/plan version en key impide reutilizar resultados generados bajo:

```text
Unanimous
```

después de cambiar a:

```text
DenyOverrides
```

---

# 264. Observability Difference

Un cache hit no ejecuta evaluadores.

Por ello:

```text
audit requirements
```

pueden limitar cache.

---

# 265. Mandatory Per-Decision Audit

Si una Ability requiere registrar cada intento individual:

```text
decision cache
```

aún podría usarse si el audit record se genera en cada request con:

```text
source=cached
```

---

# 266. Mandatory Evaluator Execution

Pero si compliance exige que un evaluator externo sea llamado en cada intento:

```text
cache prohibited
```

---

# 267. Evaluator Metadata

Podrá declarar:

```text
mustExecuteEveryTime=true
```

---

# 268. Example

```text
FraudRealtimeEvaluator
```

---

# 269. Cache Policy Result

Un solo evaluator `mustExecuteEveryTime` hace final decision:

```text
non-reusable
```

fuera del mismo execution point.

---

# 270. Rate Limits

No deben implementarse mediante cached authorization decisions.

Rate limiting tiene state mutation y semantics distintas.

---

# 271. Quotas

Igual.

```text
can upload?
```

puede depender de una cuota cambiante.

No cachear ampliamente salvo que el quota subsystem lo permita.

---

# 272. License Entitlements

Pueden cachearse con TTL/versioning, pero deben tratarse como external grant data, no hardcoded Policy results.

---

# 273. Feature Flags

Cache según feature subsystem.

La Policy puede usar el snapshot actual.

---

# 274. Identity Claims

Authentication claims podrán permanecer durante el lifetime del token/session según security model.

---

# 275. Authorization Freshness and Token Claims

Un JWT con roles embebidos puede estar stale hasta expirar.

Esto es un problema del Identity/Authorization source, no solo del Decision Cache.

---

# 276. Token Revocation

Para permisos críticos puede requerirse introspection/version check.

---

# 277. Cache Layering Warning

No aumentar involuntariamente stale window:

```text
JWT stale 15 min
+
Role cache 5 min
+
Decision cache 5 min
```

La consistencia efectiva estará limitada por la fuente más stale.

---

# 278. Freshness Budget

En sistemas avanzados podrá definirse:

```text
authorization freshness budget
```

por Ability.

---

# 279. Example

```text
invoice.view
max stale=30 sec

system.deploy
max stale=0 sec
```

---

# 280. Cache Policy Compiler

Podrá validar que TTLs configurados no excedan el budget.

---

# 281. Development Warnings

Ejemplo:

```text
Ability system.deploy requires strong consistency
but distributed decision cache TTL is 60 seconds.
```

---

# 282. Security Lint

Debe fallar si configuración contradice una restricción crítica.

---

# 283. Request-local Negative Memoization

Muy segura para:

```text
Permission missing
Role missing
Relationship missing
```

mientras no cambie el estado en la misma request.

---

# 284. Grant Changes Mid-Request

Si una request administra Roles y después ejecuta una operación como ese Principal:

```text
invalidate local memo
```

o crear una nueva AuthorizationSession.

---

# 285. Recommended Administration Flow

Después de mutar grants:

```text
AuthorizationSession::invalidatePrincipal($principal)
```

---

# 286. Automatic Invalidation

RoleManager/PermissionManager podrán notificar al request-scoped memoizer.

---

# 287. Subject Mutation Invalidation

Domain services no deberían depender de que Authorization conozca cada mutation.

Para operaciones que cambian autorización-relevant state:

```text
reauthorize fresh
```

es más fiable.

---

# 288. Mutable Workflow

Ejemplo:

```text
authorize draft edit
change status to locked
attempt another edit
```

debe realizar fresh check.

---

# 289. Transaction-local Version

Futuro:

```text
authorization state token
```

podría cambiar en memoria con mutations.

No es necesario V1.

---

# 290. Cache Hierarchy

Arquitectura recomendada:

```text
Compiled Metadata
      ↓
Process-wide

Plan Templates
      ↓
Process-wide / shared

Grant Definitions
      ↓
Process-wide or versioned distributed

Principal Grants
      ↓
Request memo + versioned cache

Relationships
      ↓
Request memo + optional versioned cache

ABAC Attributes
      ↓
Provider-specific freshness

Evaluator Results
      ↓
Request memo by default

Final Decisions
      ↓
Request memo by default
      ↓
Cross-request only if explicitly safe
```

---

# 291. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Cache/
        ├── Contracts/
        │   ├── AuthorizationMemoizerInterface.php
        │   ├── AuthorizationDecisionCacheInterface.php
        │   └── AuthorizationCacheKeyBuilderInterface.php
        │
        ├── Metadata/
        │   ├── AuthorizationRegistryVersion.php
        │   ├── AuthorizationPlanFingerprint.php
        │   └── AuthorizationCacheSchemaVersion.php
        │
        ├── Memoization/
        │   ├── AuthorizationMemoizer.php
        │   ├── AuthorizationMemoizationKey.php
        │   ├── EvaluatorMemoizer.php
        │   └── AuthorizationSessionCache.php
        │
        ├── Decisions/
        │   ├── AuthorizationDecisionCache.php
        │   ├── AuthorizationDecisionCacheKey.php
        │   ├── AuthorizationCachedDecision.php
        │   ├── AuthorizationDecisionCachePolicy.php
        │   ├── DecisionReusePolicy.php
        │   └── AuthorizationConsistencyMode.php
        │
        ├── Plans/
        │   ├── AuthorizationPlanCache.php
        │   ├── AuthorizationPlanTemplateCache.php
        │   └── AuthorizationPlanCacheKey.php
        │
        ├── Fingerprints/
        │   ├── PrincipalAuthorizationFingerprint.php
        │   ├── SubjectAuthorizationFingerprint.php
        │   ├── TenantAuthorizationFingerprint.php
        │   ├── SecurityContextFingerprint.php
        │   └── RelevantContextFingerprint.php
        │
        ├── Invalidation/
        │   ├── AuthorizationCacheInvalidator.php
        │   ├── AuthorizationVersionManager.php
        │   └── AuthorizationInvalidationSubscriber.php
        │
        └── Exceptions/
            ├── AuthorizationCacheException.php
            ├── InvalidAuthorizationCacheKeyException.php
            ├── AuthorizationCacheVersionException.php
            └── AuthorizationCacheCorruptionException.php
```

---

# 292. Metadata Cache Invariants

### Invariante 1

Metadata cache no contiene Principal runtime.

### Invariante 2

Metadata cache no contiene Subject instances.

### Invariante 3

Metadata cache puede compartirse process-wide.

### Invariante 4

Los cambios estructurales invalidan por versión.

---

# 293. Plan Cache Invariants

### Invariante 1

Plan Cache describe qué ejecutar, no qué resultado devolver.

### Invariante 2

Las keys son estructurales.

### Invariante 3

PlanTemplates son inmutables.

### Invariante 4

Runtime values se enlazan por request.

---

# 294. Memoization Invariants

### Invariante 1

Request memo nunca cruza unidades de trabajo.

### Invariante 2

La key incluye todos los inputs dinámicos relevantes.

### Invariante 3

Mutaciones relevantes requieren invalidación o fresh authorization.

### Invariante 4

Un in-progress check no se considera resultado cacheable.

---

# 295. Decision Cache Invariants

### Invariante 1

Cross-request decision caching está deshabilitado por defecto.

### Invariante 2

Una decisión solo se cachea si todos sus evaluadores lo permiten.

### Invariante 3

La key incluye versiones/fingerprints relevantes.

### Invariante 4

Un cache corrupto nunca concede acceso.

### Invariante 5

Failures técnicos no se cachean como decisiones normales.

---

# 296. Revocation Invariants

### Invariante 1

Role/Permission revocation debe cambiar el fingerprint correspondiente.

### Invariante 2

Tenant revocation/status debe invalidar decisiones relevantes.

### Invariante 3

Relationship changes deben afectar graph cache.

### Invariante 4

Critical abilities pueden exigir consistencia fuerte.

---

# 297. Multi-Tenant Invariants

### Invariante 1

Tenant forma parte de toda key tenant-sensitive.

### Invariante 2

Nunca reutilizar caches entre tenants por error.

### Invariante 3

Per-tenant authorization versions pueden utilizarse para invalidación eficiente.

### Invariante 4

Global/System contexts tienen fingerprints distintos a Tenant context.

---

# 298. FrankenPHP Invariants

### Invariante 1

Compiled metadata puede permanecer en worker memory.

### Invariante 2

Request memoization debe limpiarse entre requests.

### Invariante 3

No existen static decision arrays por Principal.

### Invariante 4

AuthorizationSession es request/job scoped.

---

# 299. Security Invariants

### Invariante 1

Un stale GRANT nunca debe utilizarse como fallback por defecto.

### Invariante 2

Unknown dependency deshabilita cross-request reuse.

### Invariante 3

El caller puede exigir mayor frescura, nunca menor.

### Invariante 4

Critical evaluator `mustExecuteEveryTime` bloquea decision cache.

### Invariante 5

Cache backend failure no produce GRANT.

---

# 300. Arquitectura final

```text
                      AuthorizationRequest
                              │
                              ↓
                 Decision Cache Policy Resolver
                              │
                  ┌───────────┴───────────┐
                  ↓                       ↓
            Fresh Required?        Reuse Allowed?
                  │                       │
                 yes                     yes
                  │                       ↓
                  │              Request Memoization
                  │                  ┌────┴────┐
                  │                 HIT      MISS
                  │                  │         │
                  │                  ↓         ↓
                  │              Decision   Cross-Request
                  │                          Decision Cache
                  │                          ┌────┴────┐
                  │                         HIT      MISS
                  │                          │         │
                  │                          ↓         ↓
                  │                      Decision   Plan Cache
                  │                                    │
                  └────────────────────────────────────┤
                                                       ↓
                                              AuthorizationPlan
                                                       │
                                                       ↓
                                                  Execution
                                                       │
                                                       ↓
                                               DecisionManager
                                                       │
                                                       ↓
                                               Final Decision
                                                       │
                         ┌─────────────────────────────┴─────────────┐
                         ↓                                           ↓
                 Request Memo Store                      Cross-Request Store
                                                           if safe
```

---

# 301. Ejemplo — request memoization

Template:

```php
@foreach ($invoices as $invoice)
    @can('update', $invoice)
        ...
    @endcan
@endforeach
```

Supongamos que internamente una misma decisión es solicitada dos veces por componente.

Primera:

```text
User#42
update
Invoice#928
Tenant#7

Memo:
MISS
```

Se ejecuta:

```text
TenantIsolation
Permission
InvoicePolicy
```

Resultado:

```text
GRANT
```

Se almacena request memo.

Segunda:

```text
same Principal fingerprint
same Ability
same Subject fingerprint
same Tenant
same Context
```

Resultado:

```text
MEMO HIT
→ GRANT
```

No se ejecutan nuevamente Policies.

---

# 302. Ejemplo — permiso global reutilizado

Lista de 500 invoices.

Cada autorización necesita:

```text
invoice.view
```

PermissionEvaluator:

```text
User#42 has invoice.view in Tenant#7?
```

Primera vez:

```text
DB/cache lookup
→ GRANT
```

Siguientes 499:

```text
request permission memo
→ GRANT
```

La Resource Policy todavía puede ejecutarse por cada Invoice.

---

# 303. Ejemplo — cross-request cache rechazado

Policies:

```text
TenantIsolationPolicy
request cacheable

PermissionEvaluator
distributed cacheable

InvoicePolicy
depends on mutable invoice.status
request only
```

Final cacheability:

```text
REQUEST ONLY
```

Aunque dos de tres evaluadores sean más cacheables.

---

# 304. Ejemplo — safe cross-request read

Ability:

```text
documentation.public.view
```

Subject:

```text
GlobalDocumentationSection v4
```

Evaluators:

```text
SubscriptionGate
versioned by Principal entitlement version

DocumentationPolicy
subject version=v4
```

Si ambos permiten:

```text
distributed cache 15 sec
```

la final decision podrá cachearse por 15 segundos.

---

# 305. Ejemplo — critical operation

Ability:

```text
system.deploy
```

Metadata:

```text
consistency=strong
decisionReuse=request
```

Evaluators:

```text
MFA
DeploymentPermission
EnvironmentPolicy
RealtimeSecurityEvaluator
```

Resultado:

```text
no cross-request decision cache
```

y una llamada:

```php
Authorization::fresh()
    ->authorize('system.deploy');
```

puede forzar nueva evaluación incluso dentro del workflow.

---

# 306. Ejemplo — revocación

Antes:

```text
User#42
auth_version=18

Permission:
invoice.approve
```

Cache key:

```text
user:42:v18:tenant:7:invoice.approve
```

Se revoca permission:

```text
authorization_version
18 → 19
```

La nueva request construye:

```text
user:42:v19:tenant:7:invoice.approve
```

La entrada de v18 ya no puede producir GRANT.

---

# 307. Ejemplo — cambio de Tenant

Request A:

```text
User#42
Tenant#7
invoice.update
→ GRANT
```

Request B:

```text
User#42
Tenant#9
invoice.update
```

Aunque el mismo usuario y Ability participen:

```text
Tenant fingerprint differs
```

por tanto la decisión de Tenant#7 nunca se reutiliza.

---

# 308. Ejemplo — mutation inside request

```text
Invoice#928
status=pending

authorize approve
→ GRANT

invoice.status=approved

authorize approve again
```

Si `status` afecta autorización, el segundo check deberá ser:

```text
fresh
```

o usar un Subject fingerprint/version actualizado.

No reutilizar el GRANT anterior.

---

# 309. Filosofía del sistema

La filosofía definitiva será:

```text
Compile static authorization once.

Reuse structural resolution aggressively.

Memoize stable dynamic facts per request.

Version grants and relationships.

Treat final cross-request decisions as privileged cache entries.

Assume unknown dependencies are unsafe to reuse.

Invalidate through versions rather than fragile key deletion when possible.

Never trade revocation correctness for a small latency gain on critical operations.

Keep all request memoization isolated under persistent workers.
```

---

# 310. Resultado esperado

El `Authorization Cache, Memoization and Decision Reuse System` deberá permitir que VoltStack ejecute autorización intensiva con alto rendimiento sin convertir el cache en una nueva superficie de escalamiento de privilegios.

El modelo final será:

```text
Compiled Metadata
        ↓
Plan Cache
        ↓
Request Memoization
        ↓
Versioned Grant / Relationship Caches
        ↓
Optional Safe Decision Cache
        ↓
Fresh Authorization for Critical Operations
```

El principio central será:

```text
Cache what is structurally stable.

Memoize what is stable for the current execution.

Reuse final decisions only when every relevant
authorization dependency can prove its freshness.

When freshness cannot be proven,
VoltStack evaluates again.
```

Con esta arquitectura, VoltStack podrá aprovechar de forma segura los beneficios de FrankenPHP, Opcache, Redis y caches request-scoped sin permitir que una decisión obsoleta de autorización se convierta en una vulnerabilidad persistente.