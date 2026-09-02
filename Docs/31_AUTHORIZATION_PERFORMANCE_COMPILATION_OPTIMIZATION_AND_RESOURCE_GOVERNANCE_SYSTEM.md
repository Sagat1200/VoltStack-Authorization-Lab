# VoltStack Authorization System

## Authorization Performance, Compilation, Optimization and Resource Governance System

**Documento:** `31_AUTHORIZATION_PERFORMANCE_COMPILATION_OPTIMIZATION_AND_RESOURCE_GOVERNANCE_SYSTEM.md`  
**Sistema:** Authorization  
**Framework:** VoltStack  
**Módulo:** `Quantum/Authorization`  
**Estado:** Especificación arquitectónica  
**Versión objetivo:** 1.x+

---

# 1. Propósito

Este documento define la arquitectura oficial de:

```text id="xm4g3v"
Performance
Compilation
Optimization
Caching
Memoization
Batch Authorization
Query Optimization
Resource Governance
Runtime Budgets
Algorithmic Complexity Protection
Persistent Worker Optimization
```

para el Authorization System de VoltStack.

La arquitectura de Authorization ya incluye:

```text id="lfsj8e"
Policies
Gates
RBAC
ABAC
ReBAC
Tenant Isolation
Scopes
Ownership
Sharing
Delegation
Capabilities
Impersonation
Risk
Approval
SoD
Distributed Consistency
Operational Tooling
Verification
```

Un sistema tan completo puede volverse costoso si cada:

```php id="mjvdk4"
$authorization->authorize(...)
```

ejecuta innecesariamente:

```text id="k73w71"
database queries
graph traversals
policy reflection
container lookups
metadata discovery
risk providers
distributed cache calls
relationship resolution
```

La meta es que la riqueza funcional no destruya el rendimiento.

La regla fundamental será:

> **VoltStack deberá mover todo trabajo estructural posible fuera del hot path y reservar el runtime únicamente para datos que realmente dependan del Principal, Resource, Context o estado dinámico.**

---

# 2. Objetivo

El Authorization System deberá perseguir simultáneamente:

```text id="v64s7b"
Correctness
Security
Low Latency
Low Allocation
Bounded Resource Usage
Predictable Complexity
Persistent Worker Safety
```

Nunca se optimizará sacrificando:

```text id="tq5ju3"
tenant isolation
revocation correctness
non-bypassable evaluators
approval requirements
risk requirements
```

---

# 3. Performance Philosophy

El modelo será:

```text id="e8truh"
DISCOVER ONCE
      ↓
VALIDATE ONCE
      ↓
COMPILE ONCE
      ↓
FREEZE
      ↓
EXECUTE MANY TIMES
```

Especialmente bajo:

```text id="90v090"
FrankenPHP
```

---

# 4. Static vs Dynamic Work

Authorization deberá clasificar todo trabajo como:

```php id="rb5ma9"
enum AuthorizationComputationKind: string
{
    case Static = 'static';
    case DeploymentStatic = 'deployment_static';
    case TenantStatic = 'tenant_static';
    case RequestStatic = 'request_static';
    case Dynamic = 'dynamic';
    case Volatile = 'volatile';
}
```

---

# 5. Static Work

Ejemplos:

```text id="4s0f07"
Ability definitions
Policy mappings
Evaluator dependency graph
Attribute metadata
Role definition metadata
Scope type definitions
Approval topology
Plugin registration
```

Debe compilarse.

---

# 6. Deployment-Static

Ejemplos:

```text id="962acs"
compiled policies
controller authorization metadata
route authorization metadata
authorization manifest
```

Cambian con deployment.

---

# 7. Tenant-Static

Ejemplos:

```text id="hb1ar9"
tenant policy profile
tenant scope rules
tenant-specific restrictions
```

Puede cachearse por versión.

---

# 8. Request-Static

Ejemplos:

```text id="iu73yw"
Principal
Actor
Tenant
Session
Authentication Assurance
```

Puede memoizarse dentro del request.

---

# 9. Dynamic

Ejemplos:

```text id="4e1lss"
resource owner
current membership
role assignment
relationship
```

Se resuelve según necesidad.

---

# 10. Volatile

Ejemplos:

```text id="77xspc"
risk
revocation status
single-use capability state
resource version
```

Debe tener freshness estricta.

---

# 11. Core Performance Equation

```text id="ev9wmb"
AUTHORIZATION COST
=
STATIC COST
+
DYNAMIC COST
+
EXTERNAL COST
```

El objetivo es llevar:

```text id="026y55"
STATIC COST
```

prácticamente a:

```text id="pxea4r"
zero per request
```

---

# 12. Authorization Compiler

VoltStack deberá tener un compilador de Authorization.

```php id="1ig8g4"
interface AuthorizationCompilerInterface
{
    public function compile(
        AuthorizationCompilationInput $input,
    ): CompiledAuthorizationManifest;
}
```

---

# 13. Compilation Inputs

```text id="2nzbqj"
Abilities
Policies
Gates
Voters
Evaluators
Attributes
Scopes
Roles metadata
Relationship plans
Approval definitions
Context requirements
Plugins
Extension hooks
```

---

# 14. Compilation Pipeline

```text id="m4y92v"
Discovery
   ↓
Normalization
   ↓
Validation
   ↓
Dependency Resolution
   ↓
Conflict Detection
   ↓
Static Analysis
   ↓
Plan Generation
   ↓
Optimization
   ↓
Manifest Generation
   ↓
Freeze
```

---

# 15. Compiled Manifest

```php id="qpp6ns"
final readonly class CompiledAuthorizationManifest
{
    public function __construct(
        public string $version,
        public string $fingerprint,
        public array $abilities,
        public array $policies,
        public array $plans,
        public array $evaluators,
        public array $metadata,
    ) {}
}
```

---

# 16. Manifest Goals

Debe permitir:

```text id="iy0ocn"
O(1) ability lookup
O(1) policy lookup
preordered evaluators
precomputed dependency sets
precomputed context requirements
precomputed cacheability
```

cuando sea posible.

---

# 17. No Runtime Reflection by Default

Producción no deberá depender de:

```text id="xavvks"
ReflectionClass
ReflectionMethod
Attribute scanning
filesystem scanning
```

en cada autorización.

---

# 18. Reflection at Compile Time

Permitido:

```text id="k2ax62"
deployment/bootstrap compilation
```

---

# 19. Controller Metadata Compilation

Attributes:

```php id="9jmn34"
#[Authorize('document.update')]
```

se convierten a metadata compilada.

---

# 20. Route Metadata Compilation

Rutas pueden tener:

```text id="3iicgp"
auth requirement
ability
subject resolver
scope requirement
```

ya preprocesados.

---

# 21. Policy Compilation

VoltStack podrá soportar:

```text id="qgtuy1"
interpreted policy
compiled policy
```

---

# 22. Compiled Policy

Un Policy declarativo puede transformarse a:

```text id="l9r1ke"
optimized evaluator plan
```

---

# 23. Example

Declarativo:

```text id="k6hcqg"
Role Editor
AND
Resource Workspace matches Current Workspace
AND
Resource not locked
```

Compilado:

```text id="abgkqy"
RBAC lookup
→ workspace equality
→ locked flag
```

---

# 24. Compile-Time Constant Folding

Ejemplo:

```text id="kd088w"
platform policy condition = always true
```

puede eliminarse.

---

# 25. Dead Rule Elimination

Reglas nunca alcanzables pueden detectarse.

---

# 26. Duplicate Rule Elimination

Reglas equivalentes pueden normalizarse.

---

# 27. Evaluator Dependency Graph

Antes de runtime:

```text id="h373fu"
TenantIsolation
   ↓
Scope
   ↓
RBAC
   ↓
Policy
   ↓
Risk
```

debe estar resuelto.

---

# 28. No Sorting on Hot Path

No hacer:

```php id="zk3hbc"
usort($evaluators, ...);
```

por autorización.

---

# 29. Compiled Evaluation Plan

```php id="x6nekm"
final readonly class CompiledAuthorizationPlan
{
    public function __construct(
        public string $ability,
        public array $steps,
        public AuthorizationPlanCapabilities $capabilities,
    ) {}
}
```

---

# 30. Plan Capabilities

```php id="kbq9yr"
final readonly class AuthorizationPlanCapabilities
{
    public function __construct(
        public bool $needsRbac,
        public bool $needsAbac,
        public bool $needsRebac,
        public bool $needsRisk,
        public bool $needsApproval,
        public bool $needsDelegation,
        public bool $needsCapability,
    ) {}
}
```

---

# 31. Lazy Subsystems

Si:

```text id="8cgl1i"
needsRisk = false
```

no resolver:

```text id="pewzq7"
RiskProvider
```

---

# 32. Example

Ability:

```text id="rzxgfr"
profile.avatar.view
```

puede requerir solo:

```text id="6oeuie"
tenant
public/resource visibility
```

No:

```text id="knnsbb"
ReBAC
Risk
Approval
Delegation
```

---

# 33. Planner

El Planner es fundamental.

```php id="9s2w4w"
interface AuthorizationPlannerInterface
{
    public function plan(
        AuthorizationRequest $request,
    ): AuthorizationExecutionPlan;
}
```

---

# 34. Planner Goal

Determinar:

```text id="yurgf9"
minimum work required
```

---

# 35. Plan Phases

```text id="aqthhw"
Resolve Static Plan
        ↓
Apply Request Dimensions
        ↓
Determine Authority Sources
        ↓
Determine Context Requirements
        ↓
Determine Consistency Requirements
        ↓
Determine Cache Strategy
        ↓
Execute
```

---

# 36. Safe Short-Circuiting

Authorization puede detenerse temprano.

---

# 37. Example

```text id="5hvn7c"
Tenant mismatch
```

produce:

```text id="mqwc31"
DENY
```

No tiene sentido ejecutar:

```text id="vm2qke"
Risk Provider
Graph traversal
Approval lookup
```

---

# 38. Short-Circuit Rule

Solo puede omitirse trabajo que no pueda alterar de forma segura el resultado.

---

# 39. Candidate ALLOW

No puede cortar evaluadores:

```text id="az72bs"
mandatory
non-bypassable
restrictive
```

---

# 40. Candidate DENY

Puede cortar muchas fases cuando:

```text id="jypv76"
deny precedence
```

está garantizada.

---

# 41. Failure Short Circuit

Infrastructure failure crítica:

```text id="5lz18e"
FAILURE
```

puede terminar evaluación.

---

# 42. Evaluator Cost Model

Cada evaluator podrá declarar coste.

```php id="kvpo1c"
enum AuthorizationCostClass: string
{
    case Trivial = 'trivial';
    case Cheap = 'cheap';
    case Moderate = 'moderate';
    case Expensive = 'expensive';
    case Remote = 'remote';
}
```

---

# 43. Evaluator Descriptor

```php id="20wp8m"
final readonly class AuthorizationEvaluatorPerformanceDescriptor
{
    public function __construct(
        public AuthorizationCostClass $cost,
        public bool $sideEffectFree,
        public bool $reorderable,
        public bool $cacheable,
    ) {}
}
```

---

# 44. Cost-Aware Ordering

Cuando la semántica lo permite:

```text id="y3ojx8"
cheap restrictive checks
```

deben ejecutarse antes de:

```text id="xg0l1z"
expensive graph/network checks
```

---

# 45. Example

Preferir:

```text id="tzerk0"
Tenant Boundary
→ Scope
→ Principal State
→ RBAC
→ ReBAC
→ Risk Provider
```

sobre orden arbitrario.

---

# 46. Reordering Constraint

Solo evaluadores:

```text id="j9y2fg"
pure
independent
reorderable
```

pueden moverse.

---

# 47. No Semantic Optimization

Una optimización nunca deberá convertir:

```text id="c4adn0"
correct but slower
```

en:

```text id="vk597q"
fast but semantically different
```

---

# 48. Memoization Layers

VoltStack deberá distinguir:

```text id="xjcrpq"
Evaluation Memoization
Request Memoization
Worker Local Cache
Distributed Cache
```

---

# 49. Evaluation Memoization

Dentro de una decisión.

Ejemplo:

```text id="535tqb"
resolve owner(document)
```

solo una vez aunque varias Policies lo consulten.

---

# 50. Request Memoization

Dentro de un request.

Ejemplo:

```text id="5zc6bl"
roles for principal
tenant membership
scope ancestry
```

---

# 51. Worker Local Cache

Solo para:

```text id="l6mg1e"
immutable
versioned
bounded
```

data.

---

# 52. Distributed Cache

Para:

```text id="f0gj6t"
shared authority projections
compiled metadata
bounded decision results
```

---

# 53. No Global Mutable Decision State

Nunca:

```text id="ak67hg"
static $currentUserPermissions
```

---

# 54. Request Memoization Contract

```php id="eq6uet"
interface AuthorizationRequestMemoizerInterface
{
    public function remember(
        AuthorizationMemoizationKey $key,
        Closure $resolver,
    ): mixed;
}
```

---

# 55. Memoization Key

Debe considerar:

```text id="ip3527"
principal
tenant
scope
context version
authority mode
```

cuando corresponda.

---

# 56. Decision Memoization

Una misma request puede preguntar:

```text id="49wsxg"
document.view
```

varias veces.

---

# 57. Request Decision Cache

Puede retornar resultado anterior si:

```text id="3141ze"
same semantic inputs
```

---

# 58. Context Mutation

Después de:

```text id="lmzy4x"
step-up authentication
```

request decision cache anterior puede quedar inválida.

---

# 59. Context Versioning

Usar:

```text id="rbt1ru"
context_version
```

---

# 60. Authority Mode Version

Cambiar a:

```text id="1m0mk2"
delegated authority
```

debe generar nuevo cache namespace.

---

# 61. Distributed Decision Cache

Más delicado.

Debe considerar:

```text id="zvq6p1"
state vector
policy version
runtime generation
context fingerprint
validity
```

---

# 62. Cacheability Descriptor

```php id="9opz50"
final readonly class AuthorizationCacheability
{
    public function __construct(
        public bool $cacheable,
        public ?DateTimeImmutable $validUntil,
        public array $varyBy,
    ) {}
}
```

---

# 63. Volatile Decisions

No cachear ampliamente cuando dependen de:

```text id="h4kq5v"
single-use capability
approval proof
volatile risk
immediate revocation
```

---

# 64. Cache TTL Ceiling

```text id="ym9d61"
effective TTL
=
minimum of all validity windows
```

---

# 65. Example

```text id="i3kufg"
policy cache TTL: 60s
capability expires: 5s
risk valid: 10s
```

decision TTL:

```text id="rlkc2u"
5s
```

---

# 66. Negative Caching

DENY puede cachearse.

---

# 67. Caveat

Grant nuevo puede invalidarlo.

Por tanto:

```text id="v84lpi"
version-aware
```

---

# 68. Cache Stampede Protection

Cuando muchas requests fallan cache simultáneamente:

```text id="v1a4x1"
single-flight
```

puede ayudar.

---

# 69. Security Caveat

No servir stale ALLOW mientras se recalcula salvo modelo explícito.

---

# 70. Cache Eviction

Debe soportar:

```text id="awudcu"
TTL
LRU/LFU adapter-specific
version rotation
explicit invalidation
```

---

# 71. Memory-Bounded Cache

Todo worker-local cache deberá tener:

```text id="d67n1b"
maximum entries
maximum bytes where possible
TTL
```

---

# 72. Unbounded Cache Prohibited

Especialmente bajo FrankenPHP.

---

# 73. Preloading

VoltStack podrá cargar:

```text id="hf0mrt"
compiled authorization manifest
policy maps
ability maps
evaluator plans
```

al boot del worker.

---

# 74. FrankenPHP Advantage

```text id="k9n4ao"
Compile once
Load once
Reuse many requests
```

---

# 75. Persistent Worker Rule

Compartir:

```text id="xnj0sx"
immutable compiled structures
```

No compartir:

```text id="b9y08n"
current principal
tenant
request cache
risk
decision state
```

---

# 76. Copy-on-Write Friendly Structures

Compiled arrays/objects deberían ser inmutables para favorecer reutilización.

---

# 77. Avoid Container Churn

Hot path no debería resolver decenas de services repetidamente.

---

# 78. Prewired Plan

Compiled plan puede contener referencias/direct callables seguros según container architecture.

---

# 79. Service Locator Anti-Pattern

Evitar:

```php id="x1e9vy"
container()->get(...)
```

en cada evaluator sin necesidad.

---

# 80. Stateless Evaluator Reuse

Evaluadores stateless pueden ser singleton.

---

# 81. Stateful Evaluators

Deben ser:

```text id="oyiv7p"
request-local
execution-local
```

---

# 82. Allocations

Reducir creación excesiva de objetos temporales en hot path.

---

# 83. Immutable Value Objects

Sí, pero con criterio.

No convertir cada booleano en 20 objetos si no aporta semántica.

---

# 84. Result Reuse

Constants para:

```text id="kc8kj1"
ABSTAIN
PASS
```

pueden reutilizarse si son totalmente inmutables.

---

# 85. Batch Authorization

Una aplicación puede necesitar:

```text id="vxp3jq"
authorize 100 documents
```

---

# 86. Naive Approach

```text id="db6m4l"
100 resources
×
5 queries each
=
500 queries
```

inaceptable.

---

# 87. Batch API

```php id="57vnap"
interface BatchAuthorizationInterface
{
    public function decideMany(
        iterable $requests,
    ): iterable;
}
```

---

# 88. Batch Planning

Agrupar por:

```text id="czhsdm"
principal
ability
tenant
scope
subject type
policy
```

---

# 89. Batch Example

```text id="18lymc"
100 documents
same principal
same ability
same workspace
```

resolver:

```text id="cmaey7"
Principal roles once
Workspace membership once
Policy metadata once
Resources in batch
```

---

# 90. Batch Data Loader Pattern

Providers deberán soportar:

```text id="5i6o55"
loadMany()
```

cuando aplique.

---

# 91. Role Provider Batch

```php id="yv461b"
interface BatchRoleProviderInterface
{
    public function resolveForPrincipals(
        iterable $principals,
        AuthorizationScopeReference $scope,
    ): iterable;
}
```

---

# 92. Ownership Batch Resolver

```php id="5v2s3r"
resolveMany(resources)
```

---

# 93. Share Batch Resolver

```text id="nqeqgi"
shares for N resources / target
```

---

# 94. ReBAC Batch

Graph backend puede resolver múltiples resources en una sola query.

---

# 95. N+1 Detection

Developer mode deberá detectar patrones como:

```text id="dfhlco"
Policy called 500 times
RelationshipStore called 500 times
```

---

# 96. Authorization N+1 Warning

```text id="oc3pnm"
AUTH-PERF-N1-001
```

---

# 97. Example

```text id="kvdpji"
Detected 250 repeated authorization lookups
for same principal and ability.

Consider:
authorizeMany()
```

---

# 98. Collection Authorization

Convenience API:

```php id="xefzrs"
$authorization->filterAuthorized(
    principal: $user,
    ability: 'document.view',
    resources: $documents,
);
```

---

# 99. Query-Level Authorization

Idealmente evitar cargar resources no autorizados.

---

# 100. Authorized Query Scope

Contrato:

```php id="gh54sl"
interface AuthorizedResourceScopeInterface
{
    public function apply(
        AuthorizationQueryBuilder $query,
        PrincipalReference $principal,
        string $ability,
    ): AuthorizationQueryBuilder;
}
```

---

# 101. Purpose

Convertir parte de Authorization en:

```text id="zfdpu6"
database predicate
```

---

# 102. Example

```text id="8vulzy"
WHERE owner_id = ?
OR EXISTS share(...)
OR EXISTS workspace_role(...)
```

---

# 103. Query Translation Limit

No todas las Policies son traducibles.

---

# 104. Query Translation Capability

```php id="rmefvg"
enum AuthorizationQueryTranslationCapability: string
{
    case Exact = 'exact';
    case Partial = 'partial';
    case Unsupported = 'unsupported';
}
```

---

# 105. Exact Translation

DB query produce exactamente conjunto autorizado.

---

# 106. Partial Translation

DB query produce candidate superset.

Luego:

```text id="t3gav6"
per-resource authorization
```

filtra.

---

# 107. Critical Rule

Partial translation debe producir:

```text id="o3qa24"
superset
```

nunca:

```text id="280nku"
unsafe broader final result
```

sin post-filter.

---

# 108. Unsupported

Fallback a:

```text id="94v22y"
batch authorization
```

---

# 109. Pagination Issue

Post-filter puede romper:

```text id="dpyx6n"
page size
```

---

# 110. Recommendation

Cuando query translation no es exacta:

```text id="qu9v5w"
candidate window + post-filter
```

debe manejar pagination cuidadosamente.

---

# 111. Streaming

Large authorization scans deberán soportar:

```text id="s1w7mm"
streaming
```

---

# 112. Avoid Full Materialization

No:

```text id="a6bkhh"
load 1 million relationships
```

en memoria.

---

# 113. Generator-Friendly APIs

```php id="ep1umr"
iterable
```

en contracts donde sea apropiado.

---

# 114. ReBAC Performance

ReBAC es uno de los componentes más costosos.

---

# 115. Bounded Traversal

Siempre definir:

```text id="sch067"
max depth
max nodes
max edges
max paths
```

---

# 116. Relationship Traversal Budget

```php id="e0p5aa"
final readonly class AuthorizationTraversalBudget
{
    public function __construct(
        public int $maxDepth,
        public int $maxNodes,
        public int $maxEdges,
        public int $maxPaths,
    ) {}
}
```

---

# 117. Budget Exhausted

Resultado no debe ser:

```text id="1ez5wm"
ALLOW because we stopped
```

---

# 118. Correct

```text id="cwkfop"
FAILURE
DENY
```

según policy.

---

# 119. Cycle Detection

Usar:

```text id="g5bl6k"
visited set
```

con límites.

---

# 120. Graph Explosion Protection

Ejemplo:

```text id="jr96mb"
1 node
→ 10
→ 100
→ 1000
→ ...
```

Debe detenerse por budget.

---

# 121. Precomputed Relationship Paths

Para relaciones frecuentes:

```text id="cz04bn"
workspace.editor → document
```

el compiler puede generar:

```text id="taassp"
relationship access plan
```

---

# 122. Relationship Plan

```php id="rhdrcz"
final readonly class CompiledRelationshipAccessPlan
{
    public function __construct(
        public array $paths,
        public AuthorizationTraversalBudget $budget,
    ) {}
}
```

---

# 123. No Arbitrary Runtime Graph Query Language

Evitar Policies que construyan traversal arbitrario por request.

---

# 124. Scope Hierarchy Performance

Usar estrategias adapter-specific:

```text id="x8s2rg"
closure table
materialized path
nested set
recursive CTE
projection
```

---

# 125. Core Neutrality

Authorization Core no impone una.

---

# 126. Scope Path Memoization

Dentro de request:

```text id="m7ffw7"
ancestors(workspace:91)
```

resolver una vez.

---

# 127. Role Resolution Performance

Evitar:

```text id="bt1uid"
load all roles
```

---

# 128. Effective Authority Query

Resolver solo:

```text id="gh4237"
roles relevant to current scope/ability
```

cuando storage lo soporte.

---

# 129. Ability-to-Role Reverse Index

Compiled metadata puede mantener:

```text id="4lvskq"
ability
→ candidate roles
```

---

# 130. Example

```text id="qy40fs"
document.update
→
workspace.editor
workspace.admin
document.manager
```

---

# 131. Optimization

Si principal no posee ninguno de esos roles:

```text id="le986y"
RBAC fast deny/abstain
```

sin analizar roles irrelevantes.

---

# 132. Permission Bitmap

Adapters muy especializados podrían usar:

```text id="9wqhme"
bitsets
```

para abilities compiladas.

---

# 133. Core Does Not Require Bitsets

Pero puede soportar una:

```text id="frv9f1"
CompiledPermissionIndex
```

---

# 134. Ability Numeric ID

Manifest puede asignar:

```text id="69li5b"
document.view → 41
```

internamente.

---

# 135. Stable External Name

Public API sigue usando:

```text id="bblhbo"
document.view
```

---

# 136. Internal Dense IDs

Pueden mejorar:

```text id="ngnse4"
memory
bitmap checks
array lookups
```

---

# 137. Compilation Fingerprint

Si cambia mapping:

```text id="q2xy0z"
manifest generation changes
```

---

# 138. Policy Dispatch Optimization

Map:

```text id="gx2kwj"
subject type + ability
→ policy handler
```

precompilado.

---

# 139. No Dynamic Method Name Search

Evitar por request buscar:

```text id="jkn0r9"
update()
delete()
manage()
```

por convención reflexiva.

---

# 140. Method Invoker Compilation

Resolver:

```text id="89y4x0"
callable
parameter mapping
context dependencies
```

antes del hot path.

---

# 141. Parameter Resolver Plan

Policy:

```php id="pxbxxr"
public function update(
    User $user,
    Document $document,
    AuthorizationContext $context
)
```

debe tener plan compilado.

---

# 142. No Container Autowiring Per Call

Parámetros conocidos deben resolverse directamente.

---

# 143. Context Provider Laziness

Documento 23.

No resolver:

```text id="x2neq3"
DeviceContext
NetworkContext
GeoContext
Risk
```

si Ability no lo necesita.

---

# 144. Context Requirement Set

```php id="dtip7x"
final readonly class AuthorizationContextRequirementSet
{
    public function __construct(
        public array $requiredProviders,
    ) {}
}
```

---

# 145. Compile Provider Dependencies

Ability:

```text id="4eq5tu"
payment.execute
```

puede necesitar:

```text id="7fxo92"
Risk
Assurance
Device
Approval
```

---

# 146. Lightweight Ability

```text id="9wb6gs"
profile.view
```

no.

---

# 147. External Provider Timeouts

Todos los providers remotos deberán tener:

```text id="zohv1i"
timeout
```

---

# 148. Timeout Budget

No cada provider con timeout de 5s dentro de request de 2s.

---

# 149. Deadline Propagation

Authorization deberá conocer:

```text id="4zz2ni"
remaining execution budget
```

---

# 150. Authorization Deadline

```php id="t4p1lu"
final readonly class AuthorizationDeadline
{
    public function __construct(
        public DateTimeImmutable $deadline,
    ) {}
}
```

---

# 151. Provider Receives Deadline

```text id="ic4mzi"
RiskProvider timeout
≤
remaining authorization deadline
```

---

# 152. Overall Budget

```php id="kyzggy"
final readonly class AuthorizationExecutionBudget
{
    public function __construct(
        public ?int $maxWallTimeMs,
        public ?int $maxProviderCalls,
        public ?int $maxDatabaseQueries,
        public ?int $maxRelationshipNodes,
        public ?int $maxNestedAuthorizations,
    ) {}
}
```

---

# 153. Budget Purpose

Proteger:

```text id="ci2mbv"
latency
CPU
memory
database
external services
```

---

# 154. Budget Exceeded

Debe producir:

```text id="urqktv"
AuthorizationBudgetExceededException
```

internamente.

---

# 155. Normalization

Resultado:

```text id="66rafd"
FAILURE
```

o DENY según policy.

---

# 156. Never Allow on Budget Exhaustion

Regla absoluta.

---

# 157. Provider Call Budget

Evita Policy accidental:

```text id="9k6c69"
loop over 1000 resources
→ call IAM 1000 times
```

---

# 158. Database Query Budget

Developer tooling podrá contar:

```text id="q2so53"
authorization queries
```

---

# 159. Query Budget Exceeded

En development:

```text id="gdhq90"
warning / exception
```

según strict mode.

---

# 160. CPU Budget

PHP no siempre puede medir CPU estrictamente por operación.

Pero sí:

```text id="x5meib"
wall time
operation counts
graph nodes
policy steps
```

como proxies.

---

# 161. Policy Instruction Budget

Declarative DSL puede contar:

```text id="1q4ju2"
condition evaluations
```

---

# 162. Expression Complexity

Compiler deberá limitar:

```text id="k0zcfb"
depth
node count
branch count
```

---

# 163. Policy Complexity Score

```php id="1sxr9r"
final readonly class AuthorizationPolicyComplexity
{
    public function __construct(
        public int $nodes,
        public int $depth,
        public int $externalDependencies,
        public int $relationshipTraversals,
    ) {}
}
```

---

# 164. Compile-Time Rejection

Policy extremadamente compleja puede:

```text id="a0i4zs"
fail compilation
```

---

# 165. Complexity Warning

Moderately complex:

```text id="pfuz3j"
warning
```

---

# 166. Complexity Is Security

Previene:

```text id="n2q286"
policy-based DoS
```

---

# 167. Nested Authorization Budget

Documento 26 soporta nested checks.

Debe existir:

```text id="ckpzfc"
max_nested_authorizations
```

---

# 168. Recursion Guard

Además del depth.

---

# 169. Nested Evaluation Reuse

Si Policy A pregunta exactamente la misma decisión ya en stack:

```text id="iocast"
detect recursion
```

no recalcular indefinidamente.

---

# 170. Batch vs Nested

Preferir:

```text id="ddhpzn"
batch evaluation
```

sobre loops de nested authorize.

---

# 171. Risk Provider Optimization

Risk puede ser caro.

---

# 172. Request Risk Memoization

Mismo:

```text id="u8vza8"
Principal
Session
Operation Class
```

puede reutilizar RiskAssessment si validity lo permite.

---

# 173. Operation-Specific Risk

No reutilizar assessment general si:

```text id="38o4cj"
payment.execute
```

requiere risk específico.

---

# 174. Risk Validity

Assessment debe indicar:

```text id="hjwvd6"
validUntil
scope
```

---

# 175. Approval Performance

No consultar Approval subsystem si Ability no requiere approval.

---

# 176. Approval Plan

Compiler determina:

```text id="a4ifjo"
approval possible/required
```

---

# 177. Approval Pending Query

Optimizar índices por:

```text id="jm8794"
operation fingerprint
resource
workflow status
```

---

# 178. Delegation Performance

Si request está en:

```text id="mvdm3z"
Native Authority Mode
```

no resolver delegations automáticamente salvo policy que lo requiera.

---

# 179. Delegated Mode

Resolver solo cadena activa relevante.

---

# 180. Capability Performance

Self-contained signed capability puede evitar DB lookup cuando:

```text id="r5d2i2"
revocation semantics
```

lo permitan.

---

# 181. Trade-Off

```text id="x5yl3n"
fewer datastore calls
↔
harder immediate revocation
```

---

# 182. Capability Planner

Debe decidir:

```text id="xa1xzu"
local verification only
local + revocation check
reference lookup
```

---

# 183. Cryptographic Cost

Verificación de firmas también consume CPU.

---

# 184. Do Not Verify Same Capability Repeatedly

Memoizar resultado dentro de execution/request cuando safe.

---

# 185. Secret Hash Verification

High-cost hashes para credentials pertenecen Authentication.

Authorization capability tokens pueden usar primitives adecuadas al threat model.

---

# 186. Avoid Password Hash Cost for Every Authorization Check

---

# 187. Admin Control Plane Performance

Documento 29 permite operaciones pesadas.

Estas no deben contaminar hot path.

---

# 188. Separate Resource Budgets

Control Plane puede tener budgets mayores.

```php id="mqzqft"
AuthorizationResourceBudget::forControlPlane()
```

vs:

```php id="0suxnl"
AuthorizationResourceBudget::forDataPlane()
```

---

# 189. Heavy Reverse Queries

```text id="e6z1v4"
Who can access this resource?
```

pueden ir a:

```text id="5ui5hh"
async job
```

---

# 190. Impact Analysis

También puede ser:

```text id="hklh92"
offline/async
```

---

# 191. Data Plane Must Stay Small

Nunca cargar:

```text id="anbz8l"
AuthorizationImpactAnalyzer
SecurityScanner
ReverseQueryEngine
```

en request normal.

---

# 192. Service Provider Split

```text id="aw2e9f"
AuthorizationCoreServiceProvider
AuthorizationRuntimeServiceProvider
AuthorizationAdministrationServiceProvider
AuthorizationDeveloperToolsServiceProvider
```

---

# 193. Lazy Container Bindings

Heavy control-plane services:

```text id="xfsxaq"
lazy
```

---

# 194. Opcode / Preload Integration

PHP environments pueden aprovechar:

```text id="rdz96j"
OPcache
preload
```

para compiled classes.

---

# 195. Core Does Not Depend on Preload

Pero deberá ser compatible.

---

# 196. Generated PHP Manifest

Usar:

```php id="6n9icw"
return [
    // normalized compiled data
];
```

puede ser más rápido que parsear YAML/JSON en runtime.

---

# 197. Safe Generated PHP

Debe ser generado por framework.

Nunca compilar input no confiable como código ejecutable sin validación.

---

# 198. Serialization Format

Alternativas:

```text id="yqaj3a"
PHP array
binary cache
JSON
```

---

# 199. Default

PHP array compilado es razonable para framework PHP.

---

# 200. Manifest Integrity

Debe tener:

```text id="8efolm"
fingerprint
schema version
framework version
```

---

# 201. Manifest Validation at Boot

No en cada authorize.

---

# 202. Atomic Manifest Swap

Deployment:

```text id="sn3x6l"
compile temp
validate
rename/swap
```

---

# 203. No Partial Manifest

Worker nunca debe leer archivo a medio escribir.

---

# 204. Generation Pinning

Cada evaluation usa una generación consistente.

---

# 205. Worker Upgrade

Nuevo worker:

```text id="izlf0p"
generation 51
```

viejo:

```text id="ac8wkm"
50
```

durante rolling deployment.

---

# 206. Cross-Generation Cache

No compartir decisiones si semantics/fingerprint incompatibles.

---

# 207. Cache Namespace by Generation

```text id="35jmz0"
authz:g51:...
```

---

# 208. Runtime Generation Check

Puede ser cheap integer/token comparison.

---

# 209. Security Epoch Fast Path

Documento 27.

Un:

```text id="k48ecj"
security epoch
```

debe poder verificarse eficientemente.

---

# 210. Fast Revocation Checks

Critical abilities pueden consultar:

```text id="d0j2cy"
small revocation/version index
```

en lugar de cargar toda authority.

---

# 211. Hot Revocation Index

Diseñado para:

```text id="j87bot"
Principal suspended?
Tenant suspended?
Grant revoked?
```

---

# 212. Local Snapshot + Version

Puede acelerar si:

```text id="kjfqwa"
version unchanged
```

---

# 213. Bounded Version Cache

Version lookup itself puede cachearse por muy corto tiempo según consistency.

---

# 214. Critical Immediate Revocation

Puede exigir canonical/shared strong lookup.

---

# 215. Consistency Profile Drives Performance

```text id="vd3u9v"
Relaxed
Standard
Strong
Critical
```

tienen distintos costes.

---

# 216. Planner Selects Consistency Path

No todas las Abilities pagan `Critical`.

---

# 217. Example

```text id="g1r0z8"
dashboard.view
→ standard

production.delete
→ critical
```

---

# 218. Adaptive Performance

Context puede elevar requirement.

Ejemplo:

```text id="pyjta3"
normal session
→ Standard

impersonation
→ Strong
```

---

# 219. Query Coalescing

Múltiples evaluators necesitan mismo dato:

```text id="6b3uzh"
Workspace membership
```

debe resolverse una vez.

---

# 220. Shared Dependency Resolver

```php id="2ehjc6"
interface AuthorizationDependencyResolverInterface
{
    public function resolve(
        AuthorizationDependency $dependency,
        AuthorizationExecutionContext $context,
    ): mixed;
}
```

---

# 221. Data Loader

Puede agrupar dependencies antes de ejecutar.

---

# 222. Plan-Ahead Batch Loading

Compiled plan conoce:

```text id="h4iag9"
roles
ownership
scope ancestry
```

necesarios.

Podrá prefetch en lote.

---

# 223. Over-Prefetch Warning

No cargar todo anticipadamente.

---

# 224. Lazy Prefetch Strategy

```text id="esadjj"
cheap likely-needed data
```

prefetch.

```text id="bgr7tv"
expensive conditional data
```

lazy.

---

# 225. Selectivity

Evaluadores restrictivos muy selectivos deben ir temprano si semántica lo permite.

---

# 226. Example

```text id="6vbmh7"
Principal suspended
```

rechaza inmediatamente.

---

# 227. Statistics

Runtime puede recopilar:

```text id="4atp6e"
evaluator hit rates
deny rates
latency
```

---

# 228. Adaptive Planner

Future enterprise optimization podría ajustar ordering basado en estadísticas.

---

# 229. Safety Rule

Adaptive ordering solo entre evaluators declarados:

```text id="5h5g6x"
commutative/reorderable
```

---

# 230. Default Planner

Deterministic compiled ordering.

---

# 231. Profile-Guided Optimization

Opcional futuro.

No requirement v1.

---

# 232. Resource Governance

Authorization deberá proteger recursos del sistema.

---

# 233. Resource Dimensions

```text id="wdfywp"
CPU
Wall Time
Memory
Database Queries
Provider Calls
Graph Traversal
Nested Evaluations
Cache Entries
Output Size
```

---

# 234. Governance Policy

```php id="lwe3rl"
final readonly class AuthorizationResourceGovernancePolicy
{
    public function __construct(
        public AuthorizationExecutionBudget $dataPlane,
        public AuthorizationExecutionBudget $controlPlane,
    ) {}
}
```

---

# 235. Memory Budget

Difícil medir perfectamente por evaluación.

Pero pueden limitarse estructuras:

```text id="kh3hby"
visited nodes
explanation entries
memoization entries
batch size
```

---

# 236. Explanation Budget

Una explicación no debe crecer infinitamente.

---

# 237. AuthorizationExplanationBudget

```php id="6bc597"
final readonly class AuthorizationExplanationBudget
{
    public function __construct(
        public int $maxNodes,
        public int $maxEdges,
        public int $maxReasons,
        public int $maxMetadataBytes,
    ) {}
}
```

---

# 238. Truncated Explanation

Debe declararse:

```text id="yq58bi"
truncated=true
```

---

# 239. Never Truncate Semantics

Solo representación.

No decision logic.

---

# 240. Batch Size Limits

```text id="wm49v2"
authorizeMany()
```

debe tener max batch.

---

# 241. Huge Batch

Puede dividirse en chunks.

---

# 242. Streaming Batch Results

```php id="fmiey4"
iterable
```

para grandes conjuntos.

---

# 243. Rate Limiting

Authorization Data Plane normalmente está protegido por request rate limits generales.

Pero operaciones caras específicas pueden tener:

```text id="tpobwc"
authorization cost rate limiting
```

---

# 244. Example

Reverse graph queries administrativas.

---

# 245. Cost Tokens

Opcional:

```text id="y4nnst"
cheap query = 1
graph reverse query = 100
```

---

# 246. Core Need Not Implement Billing-Like Cost Model

Pero Resource Governance puede exponer hooks.

---

# 247. Circuit Breakers

Providers remotos:

```text id="0llgck"
Risk
IAM
Graph service
PDP
```

pueden usar circuit breaker.

---

# 248. Circuit Breaker Purpose

Evitar:

```text id="k8r72m"
cascading failure
```

---

# 249. Open Circuit

Nunca implica ALLOW.

---

# 250. Failure Semantics

Según provider:

```text id="pvp15d"
optional
→ skip/abstain

mandatory
→ FAILURE/DENY
```

---

# 251. Bulkhead Isolation

External provider calls pueden separarse por pools/limits.

---

# 252. Example

Risk provider saturado no debería bloquear todo worker pool indefinidamente.

---

# 253. Timeout + Circuit Breaker + Bulkhead

Recomendado para remote dependencies.

---

# 254. Retry

No reintentar indiscriminadamente dentro del hot path.

---

# 255. Retry Budget

Máximo pequeño y solo para errores transient si idempotent.

---

# 256. Exponential Backoff

Más adecuado fuera del request.

---

# 257. Queue Fallback

No para Authorization síncrona que necesita respuesta inmediata.

---

# 258. Async Authorization

Solo use cases explícitos como:

```text id="e0d4mh"
administrative scans
impact analysis
reports
```

---

# 259. Performance Telemetry

Authorization deberá emitir métricas.

---

# 260. Core Metrics

```text id="34ko4a"
authorization_decision_duration
authorization_decisions_total
authorization_cache_hits_total
authorization_cache_misses_total
authorization_provider_calls_total
authorization_provider_duration
authorization_graph_nodes_visited
authorization_batch_size
authorization_budget_exceeded_total
```

---

# 261. Decision Outcome Metrics

Labels low-cardinality:

```text id="lyvzx6"
ALLOW
DENY
CHALLENGE
FAILURE
```

---

# 262. Avoid

```text id="u58anb"
principal_id
resource_id
tenant_id
```

como labels generales.

---

# 263. Ability Label

Puede ser alta cardinalidad en apps grandes.

Debe ser configurable.

---

# 264. Ability Group

Preferir:

```text id="ul0e6g"
ability_class
sensitivity
subsystem
```

para métricas agregadas.

---

# 265. Tracing

Root span:

```text id="uq05u4"
authorization.evaluate
```

---

# 266. Child Spans

Solo para operaciones relevantes/caras.

```text id="qothd6"
authorization.policy
authorization.relationship
authorization.risk
authorization.cache
```

---

# 267. Trace Sampling

No trazar detalle completo en 100% de tráfico de alto volumen.

---

# 268. Slow Authorization Trace

Si duración > threshold:

```text id="jeqouj"
force detailed diagnostic trace
```

puede ser útil.

---

# 269. Slow Decision Log

Developer/operations:

```text id="qj9ycm"
AUTH-PERF-SLOW
```

---

# 270. Example

```text id="4b4sdl"
Authorization decision exceeded 50ms.

Ability:
    document.view

Time:
    83ms

Breakdown:
    RBAC            1ms
    ReBAC          61ms
    Policy          2ms
    Risk           18ms
```

---

# 271. Performance Profiler

Documento 29 puede mostrar:

```text id="zyadxr"
evaluator timing
query count
cache hit
provider count
```

---

# 272. Performance Debug Mode

No obligatorio en producción.

---

# 273. SLA / SLO

VoltStack no debe imponer una cifra universal.

Pero permitirá configurar targets.

---

# 274. Example

```text id="6tlzt0"
simple authorization:
p95 < 1ms

relationship authorization:
p95 < 10ms

remote risk authorization:
p95 < 50ms
```

depende de deployment.

---

# 275. Latency Classes

```php id="h299ju"
enum AuthorizationLatencyClass: string
{
    case InMemory = 'in_memory';
    case LocalStore = 'local_store';
    case DistributedStore = 'distributed_store';
    case RemoteProvider = 'remote_provider';
}
```

---

# 276. Plan Expected Latency

Compiler puede estimar:

```text id="4xjdkg"
expected cost class
```

---

# 277. Developer Warning

Ability crítica en endpoint muy caliente que requiere:

```text id="u404mo"
3 remote providers
```

puede generar warning.

---

# 278. Performance Linter

```text id="nbnq54"
volt authorization:performance:lint
```

---

# 279. Findings

```text id="ljo1pf"
Policy triggers 2 graph traversals
Risk provider required for low-sensitivity ability
Unbounded relationship path
No batch resolver available
Controller loop performs authorize() repeatedly
```

---

# 280. Compilation CLI

```text id="0u39j5"
volt authorization:compile
```

---

# 281. Compile Output

```text id="0rbw2y"
Abilities compiled:        184
Policies compiled:          61
Evaluator plans:           184
Route requirements:        329
Controller metadata:       118
Approval plans:             14
Relationship plans:         29

Fingerprint:
    sha256:...

Status:
    READY
```

---

# 282. Compile Verify

```text id="csiawv"
volt authorization:compile --verify
```

---

# 283. Warmup

Optional:

```text id="8to6vd"
authorization:warmup
```

para:

```text id="k7v90s"
load compiled manifest
prime static caches
validate providers
```

---

# 284. No User-Specific Warmup by Default

No precargar:

```text id="ip41rh"
all users permissions
```

---

# 285. Tenant Warmup

Large SaaS puede precargar:

```text id="wpr1vo"
tenant policy metadata
```

para tenants calientes.

---

# 286. Warmup Budgets

Debe ser bounded.

---

# 287. Cold Start

Authorization debe funcionar correctamente incluso sin warmup.

Warmup solo mejora rendimiento.

---

# 288. Cache Precomputation

Puede precomputar:

```text id="o94q9f"
role ability maps
scope type maps
```

---

# 289. Avoid Precompute Explosion

No generar:

```text id="ue0knx"
every principal × every resource × every ability
```

---

# 290. Materialized Effective Permissions

Puede ser útil para ciertos RBAC cases.

---

# 291. Trade-Off

```text id="68b77g"
faster reads
↔
more invalidation complexity
```

---

# 292. Planner Chooses Projection

Si projection:

```text id="tc9h93"
fresh enough
```

usar.

Si stale:

```text id="v3y4y0"
fallback canonical / failure
```

según consistency.

---

# 293. Projection Freshness Check

Debe ser barato.

---

# 294. Projection Version

Comparar:

```text id="55ah4u"
projection version
source version
```

---

# 295. Adaptive Fallback

```text id="yb479s"
Projection stale
→ canonical query
```

para standard mode.

---

# 296. Critical Mode

Puede exigir canonical directamente.

---

# 297. Database Query Optimization

Repositories deberán usar índices adecuados.

---

# 298. Hot Index Dimensions

```text id="v7lq8s"
tenant
principal
ability
scope
resource
status
expiration
```

---

# 299. Composite Indexes

Adapter-specific.

---

# 300. Explain Query Plans

Operational tooling puede ejecutar:

```text id="9d9kw7"
authorization:db:explain
```

conceptualmente.

---

# 301. Core Boundary

Database optimizer pertenece Database subsystem.

Authorization define query patterns.

---

# 302. Prepared Statements

Database layer maneja.

---

# 303. Connection Pooling

FrankenPHP/Database layer maneja.

---

# 304. Authorization Should Reuse Connections

No abrir conexión por decision.

---

# 305. Data Locality

Tenant-sharded deployments:

```text id="kxjffn"
authority lookup
```

debe ir al shard correcto temprano.

---

# 306. Tenant Resolution Before DB Fanout

Evita consultar varios shards.

---

# 307. Cross-Tenant Query Prohibition

Además de seguridad, mejora performance.

---

# 308. Distributed ReBAC

Remote graph service debe soportar:

```text id="pgqunk"
batch
bounded traversal
deadline
```

---

# 309. PDP Remote Mode

Authorization podría externalizar decisiones.

---

# 310. Network Cost

Debe ser explícito.

---

# 311. Local Fast Path

Para abilities simples:

```text id="32t2bn"
do not call remote PDP
```

si deployment híbrido lo permite.

---

# 312. Remote PDP Cache

Version-aware.

---

# 313. Remote PDP Deadline

Propagar.

---

# 314. PDP Batch API

Muy importante para listas.

---

# 315. Serialization Cost

Remote authorization envelopes deben ser compactos.

---

# 316. Avoid Full Resource Serialization

Enviar:

```text id="l3tbq8"
resource ID
required attributes
version
```

no objeto completo.

---

# 317. Attribute Projection

Compiler conoce atributos que Policy necesita.

---

# 318. Example

Policy solo requiere:

```text id="nl9ux7"
owner_id
workspace_id
status
```

No enviar:

```text id="dt5yuo"
document content
attachments
```

---

# 319. Attribute Dependency Compilation

```php id="a7xxth"
final readonly class AuthorizationAttributeDependencySet
{
    public function __construct(
        public array $principalAttributes,
        public array $subjectAttributes,
        public array $contextAttributes,
    ) {}
}
```

---

# 320. Minimal Data Principle

Performance + privacy.

---

# 321. Subject Hydration Avoidance

No cargar full ORM model si solo necesitamos:

```text id="fgi60g"
id
tenant_id
owner_id
```

---

# 322. Authorization Subject View

```php id="vljdg1"
interface AuthorizationSubjectViewInterface
{
    public function authorizationAttributes(): array;
}
```

---

# 323. Lazy Attribute Provider

Puede cargar solo atributo solicitado.

---

# 324. N+1 Risk

Lazy loading dentro de loops puede ser malo.

Batch planner deberá detectar.

---

# 325. Eager Attribute Projection

Para batch:

```text id="3207ff"
SELECT id, tenant_id, owner_id, workspace_id
```

---

# 326. ORM Neutral

Core no exige Eloquent/Doctrine.

---

# 327. Serialization of Compiled Plans

Plans deben poder:

```text id="y2989m"
serialize/load quickly
```

sin closures no serializables si se almacenan en filesystem cache.

---

# 328. Callable IDs

Usar:

```text id="qpyx1k"
service ID + method
```

o compiled descriptors.

---

# 329. At Runtime

Container puede resolver handler predeclared.

---

# 330. Fast Dispatch Map

```text id="v3na45"
abilityId
→ planId
```

---

# 331. Arrays vs Maps

Implementation deberá benchmarkear.

No dogmatismo.

---

# 332. Benchmark Suite

Documento 30 testing se extiende aquí.

---

# 333. Authorization Benchmarks

Escenarios oficiales:

```text id="c5iie1"
B01 simple RBAC
B02 RBAC + Policy
B03 scoped RBAC
B04 ownership
B05 ReBAC direct
B06 ReBAC depth 3
B07 delegation
B08 capability
B09 risk provider mocked
B10 approval check
B11 batch 100 resources
B12 query-level authorization
B13 FrankenPHP repeated requests
```

---

# 334. Benchmark Output

```text id="5v0mm2"
ops/sec
p50
p95
p99
memory/op
queries/op
provider calls/op
```

---

# 335. Regression Threshold

CI puede detectar:

```text id="sfffms"
>15% slowdown
```

según scenario.

---

# 336. Performance Baseline

Versioned por release.

---

# 337. No Single Benchmark

Debe medir distintos authority models.

---

# 338. Microbenchmarks

Útiles para:

```text id="ylirxz"
ability lookup
policy dispatch
decision normalization
```

---

# 339. Macrobenchmarks

Más importantes para:

```text id="3cldr7"
full request
database
ReBAC
distributed providers
```

---

# 340. Persistent Worker Benchmarks

Comparar:

```text id="m881nc"
cold request
warm worker
```

---

# 341. Memory Growth Test

Ejecutar:

```text id="j8a64z"
100k authorization checks
```

y observar si memoria crece sin límite.

---

# 342. Leak Detection

Especialmente:

```text id="g2dy95"
memoization
explanation traces
plugin state
tenant cache
```

---

# 343. Worker Recycling

FrankenPHP puede reciclar workers.

Authorization no deberá depender de memoria eterna.

---

# 344. Bounded Growth

Todo cache persistente necesita límites.

---

# 345. GC Considerations

No mantener references innecesarias a:

```text id="tr6v5u"
request objects
ORM graphs
response objects
```

---

# 346. Weak References

Podrían usarse en casos específicos, pero no requisito.

---

# 347. Shutdown Cleanup

Worker shutdown puede liberar resources.

Separado de request reset.

---

# 348. Request Reset Cost

Debe ser:

```text id="7ov19x"
small
deterministic
```

---

# 349. Reset Should Not Clear Immutable Manifest

---

# 350. Reset Clears

```text id="9m219d"
request memoization
execution stacks
temporary restrictions
context
```

---

# 351. Performance Safety Under Error

Excepción no debe dejar:

```text id="a6x3th"
cache lock held
context retained
provider handle leaked
```

---

# 352. `finally`

Critical.

---

# 353. Async/Fiber Performance

Si VoltStack soporta concurrency:

```text id="xulvn2"
independent provider calls
```

podrían ejecutarse en paralelo cuando semántica lo permita.

---

# 354. Example

Risk provider y external relationship provider independientes.

---

# 355. Parallel Evaluation

Solo si:

```text id="n8t2vh"
both may be needed
no earlier cheap deny likely
budget allows
```

---

# 356. Trade-Off

Paralelizar puede reducir latency pero aumentar total resource use.

---

# 357. Default V1

Preferir secuencial cost-aware.

Parallelism opcional.

---

# 358. Cancellation

Si una rama produce terminal DENY:

```text id="a394u9"
cancel other remote calls
```

cuando runtime lo soporte.

---

# 359. Cooperative Cancellation

Provider contracts pueden recibir:

```text id="a29fbd"
CancellationToken
```

---

# 360. Resource Governance Still Applies

Parallelism no evade budgets.

---

# 361. Thundering Herd

Global security epoch increment puede invalidar grandes caches.

---

# 362. Mitigation

```text id="obx6xz"
jitter
single-flight
staged warming
```

sin servir stale authority insegura.

---

# 363. Emergency Security Beats Performance

Durante security epoch event:

```text id="xrse4u"
temporary cache miss spike
```

es aceptable.

---

# 364. Policy Rollout

Blue/green compiled manifests permiten evitar recompilation on request.

---

# 365. Canary Generation

Puede existir:

```text id="57lkjx"
generation 52-canary
```

para subset controlado.

---

# 366. Cache Isolation

Canary y stable no comparten decision cache si semantics difieren.

---

# 367. Multi-Tenant Performance

No todos tenants tienen misma complejidad.

---

# 368. Tenant Resource Profile

```php id="b89s9g"
final readonly class TenantAuthorizationResourceProfile
{
    public function __construct(
        public AuthorizationExecutionBudget $budget,
        public int $maxRelationships,
        public int $maxRoles,
        public int $maxScopes,
    ) {}
}
```

---

# 369. Security Floor

Tenant puede tener budgets más estrictos.

No puede elevarlos por encima de platform safe maximum sin config autorizada.

---

# 370. Tenant Abuse Prevention

Un tenant no debe poder crear:

```text id="7xsy5a"
millions of cyclic relationships
```

y afectar todo cluster.

---

# 371. Quotas

Opcional:

```text id="plx2di"
max authorization relationships
max roles
max dynamic policies
max active capabilities
max delegations
```

---

# 372. Quotas Are Resource Governance

No authority semantics.

---

# 373. Quota Exceeded

Bloquea mutation administrativa.

No cambia decision ya válida por sí solo.

---

# 374. Role Explosion

Miles de roles por Principal pueden degradar performance.

---

# 375. Effective Role Limit

Puede haber:

```text id="jrcm3p"
warning threshold
hard platform maximum
```

---

# 376. Relationship Explosion

Igual.

---

# 377. Capability Explosion

Inventory y revocation index deben mantenerse bounded.

---

# 378. Abuse Prevention

Administrative APIs tendrán rate/cost limits.

---

# 379. Compilation Resource Limits

Un package malicioso podría registrar:

```text id="wzz0dz"
millions of abilities
```

---

# 380. Compiler Budget

```php id="k4dn8d"
final readonly class AuthorizationCompilationBudget
{
    public function __construct(
        public int $maxAbilities,
        public int $maxPolicies,
        public int $maxEvaluators,
        public int $maxPlanNodes,
    ) {}
}
```

---

# 381. Compile Failure

Exceso:

```text id="ncm0f2"
AuthorizationCompilationBudgetExceeded
```

---

# 382. Compile Time Is Deployment Cost

Puede ser mayor que runtime, pero también bounded.

---

# 383. Compile Cache

Si inputs fingerprint no cambia:

```text id="18dm6n"
reuse manifest
```

---

# 384. Incremental Compilation

Futuro:

```text id="2no5qq"
only recompile affected abilities
```

---

# 385. V1 Simplicity

Full compile probablemente suficiente.

---

# 386. Dependency Fingerprints

Cada source:

```text id="j8fnb5"
policy
plugin
ability registry
```

puede contribuir al manifest fingerprint.

---

# 387. Reproducible Builds

Mismos inputs:

```text id="ocqjjx"
same manifest semantic content
```

---

# 388. Deterministic Ordering

No depender de filesystem enumeration order.

---

# 389. Manifest Diff

Tooling:

```text id="fftcx9"
authorization:compile:diff
```

---

# 390. Show

```text id="vb8gko"
abilities changed
policies changed
plans changed
cache generation impact
```

---

# 391. Optimization Verification

Documento 30 debe verificar:

```text id="7zbpi1"
optimized decision
=
reference decision
```

---

# 392. Every Optimization Needs Semantic Test

---

# 393. Performance Anti-Patterns

Evitar:

```text id="s662pf"
reflection per authorization
```

Evitar:

```text id="9r4g8u"
container autowiring per policy call
```

Evitar:

```text id="o3d8rm"
load all permissions per request
```

Evitar:

```text id="qv1yjl"
unbounded graph traversal
```

Evitar:

```text id="o5pwbk"
remote provider for every trivial ability
```

Evitar:

```text id="es2559"
unbounded static cache in FrankenPHP worker
```

Evitar:

```text id="ft4mlw"
authorization inside large loops without batching
```

Evitar:

```text id="8clfw6"
cache without version semantics
```

Evitar:

```text id="kwg5eu"
stale-while-revalidate ALLOW for critical authority
```

Evitar:

```text id="sn0e3z"
full ORM hydration for simple authorization attributes
```

Evitar:

```text id="ynq1m0"
global locks on authorization reads
```

Evitar:

```text id="3eshzs"
all evaluations pay critical consistency cost
```

---

# 394. Performance Invariants

## Invariante 1

Static authorization metadata is not rediscovered per decision.

## Invariante 2

Compiled evaluator ordering is deterministic.

## Invariante 3

Unused subsystems are not invoked.

## Invariante 4

Caches never weaken security semantics.

## Invariante 5

All persistent caches are bounded or externally governed.

## Invariante 6

Graph traversal is bounded.

## Invariante 7

Budget exhaustion never yields ALLOW.

## Invariante 8

Batch authorization preserves per-item semantics.

## Invariante 9

Query-level authorization never returns unauthorized resources.

## Invariante 10

Persistent workers retain only safe reusable state.

---

# 395. Compilation Invariants

## Invariante 1

Manifest is immutable after boot.

## Invariante 2

Manifest contains a version/fingerprint.

## Invariante 3

Invalid manifests are rejected before runtime.

## Invariante 4

Compilation detects dependency cycles.

## Invariante 5

Compilation output is deterministic for equivalent inputs.

## Invariante 6

Policy/evaluator conflicts are detected before runtime when possible.

## Invariante 7

Generated plans preserve reference semantics.

---

# 396. Cache Invariants

## Invariante 1

Cache is never canonical authority.

## Invariante 2

Cache keys include all relevant semantic dimensions.

## Invariante 3

TTL never outlives authority validity.

## Invariante 4

Version mismatch invalidates cached result.

## Invariante 5

CHALLENGE is never reused as ALLOW.

## Invariante 6

Critical revocation checks can bypass ordinary cached ALLOW.

---

# 397. Batch Invariants

## Invariante 1

`decideMany()` equals individual decisions semantically.

## Invariante 2

Batch loading cannot merge tenants.

## Invariante 3

A failure for one item follows declared batch semantics.

## Invariante 4

Batch optimization does not collapse distinct scopes/resources incorrectly.

---

# 398. Resource Governance Invariants

## Invariante 1

Every recursive/traversal operation has a bound.

## Invariante 2

Every remote provider has a timeout/deadline.

## Invariante 3

Nested authorization depth is bounded.

## Invariante 4

Control Plane and Data Plane may have different budgets.

## Invariante 5

Untrusted configuration cannot create unbounded computation.

---

# 399. FrankenPHP Invariants

## Invariante 1

Compiled immutable manifests may persist across requests.

## Invariante 2

Request memoization never persists.

## Invariante 3

Current Principal/Tenant/Scope never persists.

## Invariante 4

Worker-local caches are bounded and version-aware.

## Invariante 5

Request reset executes even on exceptions.

---

# 400. Suggested Directory Structure

```text id="3nbfzb"
Quantum/
└── Authorization/
    ├── Compilation/
    │   ├── Contracts/
    │   │   ├── AuthorizationCompilerInterface.php
    │   │   ├── AuthorizationCompilationPassInterface.php
    │   │   └── AuthorizationManifestLoaderInterface.php
    │   │
    │   ├── Model/
    │   │   ├── AuthorizationCompilationInput.php
    │   │   ├── CompiledAuthorizationManifest.php
    │   │   ├── CompiledAuthorizationPlan.php
    │   │   └── AuthorizationCompilationBudget.php
    │   │
    │   ├── Pass/
    │   │   ├── AbilityCompilationPass.php
    │   │   ├── PolicyCompilationPass.php
    │   │   ├── EvaluatorOrderingPass.php
    │   │   ├── ContextDependencyPass.php
    │   │   ├── RelationshipPlanPass.php
    │   │   ├── ApprovalPlanPass.php
    │   │   ├── CacheabilityPass.php
    │   │   └── OptimizationPass.php
    │   │
    │   └── Manifest/
    │       ├── AuthorizationManifestWriter.php
    │       ├── AuthorizationManifestLoader.php
    │       └── AuthorizationManifestValidator.php
    │
    ├── Planning/
    │   ├── AuthorizationPlanner.php
    │   ├── AuthorizationExecutionPlan.php
    │   ├── AuthorizationPlanCapabilities.php
    │   └── AuthorizationAttributeDependencySet.php
    │
    ├── Performance/
    │   ├── Cost/
    │   │   ├── AuthorizationCostClass.php
    │   │   └── AuthorizationEvaluatorPerformanceDescriptor.php
    │   │
    │   ├── Memoization/
    │   │   ├── AuthorizationRequestMemoizer.php
    │   │   └── AuthorizationMemoizationKey.php
    │   │
    │   ├── Batch/
    │   │   ├── BatchAuthorization.php
    │   │   ├── AuthorizationBatchPlanner.php
    │   │   └── AuthorizationBatchLoader.php
    │   │
    │   ├── Query/
    │   │   ├── AuthorizedResourceScope.php
    │   │   └── AuthorizationQueryTranslationCapability.php
    │   │
    │   ├── Governance/
    │   │   ├── AuthorizationExecutionBudget.php
    │   │   ├── AuthorizationTraversalBudget.php
    │   │   ├── AuthorizationResourceGovernancePolicy.php
    │   │   └── AuthorizationDeadline.php
    │   │
    │   ├── Telemetry/
    │   │   ├── AuthorizationPerformanceCollector.php
    │   │   └── AuthorizationSlowDecisionDetector.php
    │   │
    │   └── Benchmark/
    │       ├── AuthorizationBenchmarkSuite.php
    │       └── Scenarios/
    │
    └── Exceptions/
        ├── AuthorizationCompilationException.php
        ├── AuthorizationCompilationBudgetExceededException.php
        ├── AuthorizationBudgetExceededException.php
        ├── AuthorizationTraversalBudgetExceededException.php
        └── AuthorizationManifestException.php
```

---

# 401. Runtime Fast Path Architecture

```text id="eu27ve"
                 COMPILED MANIFEST
                         │
                         ↓
                Ability Plan Lookup
                         │
                         ↓
                  Request Context
                         │
                         ↓
                Cheap Preconditions
                         │
                         ↓
              Required Authority Sources
                         │
                         ↓
                Request Memoization
                         │
              ┌──────────┼──────────┐
              ↓          ↓          ↓
            RBAC       Policy      ReBAC
              │          │          │
              └──────────┼──────────┘
                         ↓
             Only Required Context
                         ↓
                 Risk / Approval
                  if required
                         ↓
               Mandatory Security Tail
                         ↓
                    Decision
```

---

# 402. Compilation Architecture

```text id="p31ehf"
                    APPLICATION CODE
                         +
                    PACKAGE METADATA
                         +
                     CONFIGURATION
                          │
                          ↓
                      DISCOVERY
                          ↓
                     NORMALIZE
                          ↓
                      VALIDATE
                          ↓
                 DEPENDENCY GRAPH
                          ↓
                     OPTIMIZE
                          ↓
                    COMPILE PLAN
                          ↓
                 GENERATE MANIFEST
                          ↓
                       FREEZE
                          ↓
                  FRANKENPHP WORKER
                          ↓
                     HOT PATH
```

---

# 403. Batch Architecture

```text id="5m8o3z"
100 Authorization Requests
          │
          ↓
      Batch Planner
          │
  ┌───────┼─────────┐
  ↓       ↓         ↓
Group    Group      Group
Tenant   Ability    Scope
  │       │         │
  └───────┼─────────┘
          ↓
   Shared Dependency Load
          ↓
   Per-Resource Evaluation
          ↓
        Results
```

---

# 404. Resource Governance Architecture

```text id="2251ob"
Authorization Request
        │
        ↓
Execution Budget
        │
 ┌──────┼────────┬─────────┐
 ↓      ↓        ↓         ↓
Time   Queries  Providers  Graph
 │      │        │         │
 └──────┼────────┴─────────┘
        ↓
   Budget Monitor
     /       \
 within     exceeded
 budget       │
   │           ↓
   ↓        FAILURE
continue
```

---

# 405. Optimization Priority Order

VoltStack deberá priorizar optimizaciones en este orden:

```text id="8jsgbn"
1. Remove unnecessary work

2. Compile static work

3. Avoid duplicate work

4. Batch data access

5. Use correct indexes/projections

6. Cache version-safe data

7. Reduce allocations

8. Parallelize only when justified
```

---

# 406. Important Principle

La mejor query es:

```text id="8i8fa3"
the query you did not need to execute
```

Por eso Planner y short-circuiting tienen prioridad sobre caching agresivo.

---

# 407. Fast Path Example

Ability:

```text id="e2u1gy"
document.view
```

Request:

```text id="w47uqj"
same tenant
resource public
```

Compiled plan puede decidir:

```text id="h1wc9r"
Tenant check
→ visibility check
→ Policy
→ ALLOW
```

sin:

```text id="edsc7f"
roles
ReBAC
risk
approval
delegation
```

---

# 408. Complex Path Example

Ability:

```text id="5rps1t"
production.deploy
```

Puede requerir:

```text id="64jdff"
Principal State
Tenant
Scope
RBAC
Policy
Strong Assurance
Risk
Approval
SoD
Security Epoch
Pre-execution Revalidation
```

Debe ser más costoso porque la semántica lo exige.

---

# 409. Performance Is Ability-Specific

No existe:

```text id="x2ql0m"
one fixed authorization cost
```

---

# 410. Security-Preserving Optimization Formula

```text id="v87hb3"
OPTIMIZATION
=
LESS WORK
+
SAME SEMANTICS
+
BOUNDED RESOURCES
+
VERSION-SAFE CACHE
+
DETERMINISTIC COMPILATION
```

---

# 411. Final Philosophy

VoltStack no deberá perseguir rendimiento mediante:

```text id="tlt2xf"
bypassing Policy
trusting stale grants
disabling revocation checks
making every internal request trusted
using global mutable permission caches
```

En cambio deberá obtener rendimiento mediante:

```text id="nwzdte"
compilation
planning
memoization
batching
query translation
projection
bounded caching
lazy providers
short-circuiting
persistent immutable state
```

---

# 412. Core Rule

> **Una optimización válida reduce el coste de obtener la misma decisión; nunca redefine qué significa estar autorizado.**

---

# 413. Authorization Performance Formula

```text id="i5tfap"
COMPILED STRUCTURE
       +
MINIMAL EXECUTION PLAN
       +
LAZY DYNAMIC DATA
       +
REQUEST MEMOIZATION
       +
BATCH RESOLUTION
       +
SAFE CACHING
       +
BOUNDED COMPLEXITY
       +
PERSISTENT WORKER REUSE
       =
HIGH-PERFORMANCE AUTHORIZATION
```

---

# 414. Estado del Sistema

Con este documento quedan formalizados:

```text id="5jhjzi"
Authorization Compiler
Static metadata compilation
Policy compilation
Evaluator planning
Safe short-circuiting
Lazy subsystem resolution
Request memoization
Version-aware caching
Batch authorization
Authorized query scopes
N+1 prevention
ReBAC traversal budgets
Scope optimization
Role indexes
Context dependency planning
Remote provider deadlines
Execution budgets
Circuit breakers
Resource governance
Performance telemetry
Benchmarks
FrankenPHP optimization
Memory governance
```

---

# 415. Siguiente y último documento

Solo queda:

## `32_AUTHORIZATION_SYSTEM_INTEGRATION_AND_FINAL_ARCHITECTURE.md`

Este deberá ser el **documento maestro y cierre definitivo del Authorization System**.

Deberá consolidar todo el diseño anterior en una sola arquitectura final:

```text id="lr8jfq"
Principal / Actor / Effective Principal

AuthorizationRequest
AuthorizationContext
AuthorizationManager
Planner
Pipeline
DecisionManager

Policies
Gates
Voters

RBAC
ABAC
ReBAC

Tenant Isolation
Scopes

Ownership
Sharing
Relationships

Delegation
Capabilities
Impersonation
Service-to-Service

Contextual Access
Risk
Assurance
Challenges

Approval
Dual Control
SoD

Configuration
Container
Bootstrap

Lifecycle
Events
Hooks
Plugins

State Versions
Concurrency
Distributed Consistency

Persistence
Repositories
Storage Boundaries

Administration
Operational Tooling

Testing
Security Assurance
Compliance

Compilation
Caching
Performance
Resource Governance
```

También deberá definir la integración final con:

```text id="m802p0"
Authentication
Identity
Container
Config
Routing
Controllers
Middleware
HTTP
Database
Cache
Events
Telemetry
Queue
Scheduler
CLI
FrankenPHP
```

y establecer:

```text id="l41b1y"
final package structure
final decision flow
public APIs
security invariants
implementation phases
extension boundaries
runtime lifecycle
```

Al finalizar el documento **32**, la arquitectura documental de `Quantum/Authorization` podrá considerarse formalmente completa.
