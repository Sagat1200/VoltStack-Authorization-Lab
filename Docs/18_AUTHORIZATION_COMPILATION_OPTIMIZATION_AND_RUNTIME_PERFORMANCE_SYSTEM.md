# VoltStack Authorization System — Compilation, Optimization and Runtime Performance System

## 1. Propósito

Este documento define la arquitectura de **compilación, optimización y rendimiento en runtime** del Authorization System de VoltStack.

El objetivo es que un sistema de autorización capaz de integrar:

```text
Policies
Gates
Voters
Decision Strategies
RBAC
ABAC
ReBAC
Multi-Tenancy
Controller Metadata
Route Metadata
Security Evaluators
Audit
Tracing
Cache
```

pueda ejecutarse en el hot path de aplicaciones de alto tráfico sin depender continuamente de:

```text
Reflection
filesystem scanning
attribute discovery
service discovery
dynamic method inspection
policy signature analysis
registry construction
repeated dependency resolution
```

La arquitectura deberá favorecer especialmente:

```text
PHP Opcache
FrankenPHP workers
persistent runtimes
precompiled route metadata
immutable registries
precomputed authorization plans
request-local memoization
batch authorization
```

El principio fundamental será:

```text
Resolve and validate authorization structure
before the request whenever possible.

At runtime, bind only the dynamic data
required to make the decision.
```

---

# 2. Objetivos

El sistema deberá proporcionar:

1. compilación de metadata;
2. compilación de Policies;
3. compilación de Gates;
4. compilación del Ability Registry;
5. compilación de Attributes;
6. compilación de Controller/Route authorization;
7. registries sellados;
8. AuthorizationPlanTemplates;
9. zero-reflection hot path;
10. Opcache-friendly artifacts;
11. FrankenPHP worker reuse;
12. lazy runtime binding;
13. optimización por fases;
14. short-circuiting;
15. deduplicación;
16. memoization;
17. batching;
18. reducción de allocations;
19. reducción de container lookups;
20. budgets de rendimiento;
21. profiling;
22. protección contra optimizaciones inseguras.

---

# 3. Principio arquitectónico

VoltStack dividirá Authorization en dos grandes mundos:

```text
COMPILE / BOOT TIME
        │
        ├── discovery
        ├── reflection
        ├── validation
        ├── normalization
        ├── dependency analysis
        ├── metadata merging
        └── plan template generation

RUNTIME
        │
        ├── route target lookup
        ├── Principal binding
        ├── Tenant binding
        ├── Subject binding
        ├── evaluator invocation
        └── decision aggregation
```

El objetivo será mover al primer grupo toda operación que no dependa del request actual.

---

# 4. Qué debe evitarse en runtime

Durante una autorización normal en producción no debería ser necesario:

```php
new ReflectionClass($policy);
```

ni:

```php
$reflection->getAttributes();
```

ni:

```php
method_exists($policy, $ability);
```

ni:

```text
scan authorization directories
```

ni:

```text
discover Policy from filesystem
```

por cada request.

---

# 5. Hot Path Ideal

Un check como:

```php
Authorization::authorize(
    'update',
    $invoice
);
```

debería conceptualmente convertirse en:

```text
Canonical Ability ID
        ↓
PlanTemplate Lookup
        ↓
Bind Principal
Bind Tenant
Bind Invoice
        ↓
Execute precomputed evaluators
        ↓
DecisionManager
```

---

# 6. Authorization Compilation Pipeline

```text
Source Code
    ↓
Policy Discovery
Gate Registration
Ability Discovery
Attribute Discovery
    ↓
Normalization
    ↓
Validation
    ↓
Dependency Analysis
    ↓
Metadata Merge
    ↓
Registry Construction
    ↓
PlanTemplate Compilation
    ↓
Artifact Generation
    ↓
Opcache / Worker Load
```

---

# 7. AuthorizationCompiler

Podrá existir:

```php
interface AuthorizationCompilerInterface
{
    public function compile(
        AuthorizationCompilationContext $context
    ): CompiledAuthorizationManifest;
}
```

---

# 8. Compilation Context

Podrá contener:

```text
Application configuration
Registered packages
Policy sources
Gate sources
Ability sources
Route metadata
Controller metadata
Extension adapters
Environment
```

---

# 9. Compiler Responsibilities

El compiler deberá:

```text
discover
normalize
validate
canonicalize
merge
sort
deduplicate
fingerprint
serialize
```

la información estructural.

---

# 10. No Runtime Principal

Nunca incluir durante compilación:

```text
Current User
Current Tenant
Current Request
Current Subject instance
Current MFA state
```

---

# 11. CompiledAuthorizationManifest

Será el descriptor raíz.

Conceptualmente:

```php
final readonly class CompiledAuthorizationManifest
{
    public function __construct(
        public string $schemaVersion,
        public string $registryVersion,
        public array $abilities,
        public array $policies,
        public array $gates,
        public array $planTemplates,
        public array $targets,
    ) {}
}
```

---

# 12. Manifest Inmutability

Después de cargarlo:

```text
CompiledAuthorizationManifest
```

deberá considerarse:

```text
immutable
```

durante la vida de esa versión del runtime.

---

# 13. Ability Compilation

Cada Ability deberá convertirse en un descriptor compacto.

Ejemplo fuente:

```text
invoice.update
```

podría compilar a:

```text
ID: 37
name: invoice.update
subject: Invoice
risk: normal
strategy: deny_overrides
phase: resource
```

---

# 14. Numeric Internal IDs

Opcionalmente VoltStack podrá asignar:

```text
integer IDs
```

a:

```text
Abilities
Policies
Evaluators
Strategies
```

para acelerar lookups internos.

---

# 15. Public API

La API seguirá utilizando:

```php
'invoice.update'
```

pero runtime podrá canonicalizar una vez:

```text
invoice.update → Ability ID 37
```

---

# 16. AbilityLookupTable

Podrá existir:

```text
name → descriptor ID
```

como array PHP optimizado.

---

# 17. Example

```php
return [
    'invoice.view' => 31,
    'invoice.update' => 37,
    'invoice.approve' => 41,
];
```

---

# 18. Policy Compilation

Policies deberán descubrirse y validarse durante compile.

---

# 19. Policy Descriptor

Ejemplo:

```text
Policy ID:
12

Class:
App\Authorization\InvoicePolicy

Subject:
App\Domain\Invoice

Methods:

view
→ method ID 1

update
→ method ID 2

approve
→ method ID 3
```

---

# 20. PolicyMethodDescriptor

Podrá contener:

```text
method name
ability ID
parameter binding map
return normalization mode
priority
phase
cacheability
failure mode
```

---

# 21. Parameter Binding Compilation

Fuente:

```php
public function update(
    User $user,
    Invoice $invoice,
    AuthorizationContext $context,
): DecisionResult
```

deberá compilar a algo similar:

```text
argument 0 → PRINCIPAL
argument 1 → SUBJECT
argument 2 → CONTEXT
```

---

# 22. Benefit

Runtime no necesita analizar tipos con Reflection.

Solo:

```text
invoke with precomputed argument map
```

---

# 23. Specialized Invoker

Podrá generarse un invoker optimizado.

Conceptualmente:

```php
$policy->update(
    $request->principal(),
    $request->subject(),
    $request->context(),
);
```

sin introspección.

---

# 24. Generated Closures

Podría explorarse:

```text
compiled invocation closure
```

pero deberá benchmarkearse frente a llamadas directas y arrays descriptivos.

---

# 25. Recommendation Inicial

Preferir:

```text
compact descriptors
+
simple dispatcher
```

antes que generar cantidades enormes de closures.

---

# 26. Gate Compilation

Los Gates registrados:

```php
Gate::define(
    'admin.access',
    AdminAccessGate::class
);
```

deberán compilarse.

---

# 27. Gate Descriptor

Podrá contener:

```text
Gate ID
Ability ID
Resolver ID
Callable type
Argument map
Priority
Strategy participation
```

---

# 28. Closure Gates

Si VoltStack permite closure Gates, su compatibilidad con cache/compilation dependerá del mecanismo general de route/config serialization.

---

# 29. Recommendation

Para aplicaciones totalmente compilables:

```text
invokable Gate classes
```

son preferibles a closures dinámicas.

---

# 30. Static Gate Classes

Ejemplo:

```php
final class AdminAccessGate
{
    public function __invoke(
        User $user
    ): bool {
        return $user->isAdmin();
    }
}
```

---

# 31. Decision Strategy Compilation

Los nombres:

```text
deny_overrides
unanimous
affirmative
consensus
```

deberán canonicalizarse durante compile.

---

# 32. Strategy ID

Runtime no debería buscar repetidamente por string si puede utilizar:

```text
Strategy ID
```

pre-resuelto.

---

# 33. Evaluator Compilation

Cada evaluator estructural deberá producir:

```text
EvaluatorDescriptor
```

---

# 34. EvaluatorDescriptor

Conceptualmente:

```php
final readonly class EvaluatorDescriptor
{
    public function __construct(
        public int $id,
        public string $serviceId,
        public int $priority,
        public AuthorizationPhase $phase,
        public bool $required,
        public bool $nonBypassable,
    ) {}
}
```

---

# 35. Service Resolution

Durante runtime deberá evitarse:

```text
container->get(serviceId)
```

en cada evaluator si la instancia puede resolverse una vez por worker.

---

# 36. Shared Stateless Evaluators

Estos son buenos candidatos a:

```text
shared services
```

si no retienen:

```text
Principal
Tenant
Subject
Request state
```

---

# 37. Request-Scoped Evaluators

Si alguno requiere state request-local, deberá resolverse desde el runtime scope adecuado.

---

# 38. Compiled Evaluator Table

El worker podrá mantener:

```text
Evaluator ID → evaluator instance
```

para evaluadores shared.

---

# 39. Zero Service-Locator Hot Path

Idealmente un PlanTemplate tendrá referencias compactas que el runtime resuelve mediante una tabla directa.

---

# 40. AuthorizationPlanTemplate

El Planner podrá precompilar estructuras.

Ejemplo:

```text
Ability:
invoice.update

Subject:
Invoice

Plan:

1 TenantIsolationPolicy
2 PermissionEvaluator
3 InvoicePolicy
4 CompliancePolicy

Strategy:
deny_overrides
```

---

# 41. Template vs Plan

```text
PlanTemplate
=
static execution structure
```

```text
AuthorizationPlan
=
template bound to runtime request
```

---

# 42. Runtime Plan Binding

```text
PlanTemplate
+
Principal
+
Tenant
+
Subject
+
SecurityContext
=
AuthorizationPlan
```

---

# 43. Avoid Runtime Plan Allocation

Incluso la creación de un nuevo `AuthorizationPlan` object por check puede no ser necesaria.

El runtime podrá ejecutar:

```text
PlanTemplate + AuthorizationExecutionContext
```

directamente.

---

# 44. AuthorizationExecutionContext

Request-local:

```php
final readonly class AuthorizationExecutionContext
{
    public function __construct(
        public PrincipalInterface $principal,
        public mixed $subject,
        public AuthorizationContext $context,
    ) {}
}
```

---

# 45. Compact Plan Representation

En production, un plan podría representarse internamente como:

```php
[
    4,  // evaluator ID
    12,
    21,
    31,
]
```

más metadata paralela.

---

# 46. Readability vs Performance

No debe sacrificarse toda mantenibilidad.

Los objetos descriptivos pueden mantenerse en development mientras production usa representación compilada.

---

# 47. Dual Representation

Podría existir:

```text
Debug Plan Representation
Compiled Runtime Plan Representation
```

producidas desde la misma fuente.

---

# 48. Semantic Equivalence

Testing deberá garantizar:

```text
debug plan
=
compiled plan
```

semánticamente.

---

# 49. Route Compilation Integration

El Route Compiler deberá enlazar una ruta con:

```text
authorization target ID
```

---

# 50. Example

Route:

```text
invoice.update
```

puede compilar:

```text
route ID 84
controller target ID 120
authorization template ID 33
```

---

# 51. Runtime Route Match

Después del match:

```text
RouteDescriptor
```

ya sabe qué autorización utilizar.

---

# 52. No Runtime Metadata Merge

No combinar repetidamente:

```text
Route metadata
Controller class metadata
Method metadata
```

por request.

---

# 53. Effective Metadata Compilation

El compiler deberá producir:

```text
EffectiveAuthorizationMetadata
```

por target cuando sea posible.

---

# 54. Route-Specific Targets

Dos rutas hacia el mismo Controller pueden tener metadata distinta.

Por tanto podrá existir:

```text
AuthorizationTarget
```

por Route + Controller invocation.

---

# 55. Target ID

Ejemplo:

```text
auth_target:84
```

---

# 56. Controller Compilation Integration

El Controller Compiler deberá proporcionar:

```text
ControllerMethodDescriptor
```

con:

```text
argument positions
subject references
authorization-required arguments
invocation-only arguments
```

---

# 57. Authorization-Aware Argument Resolution

El compiler puede generar:

```text
Authorization Argument Dependency Plan
```

---

# 58. Example

Controller:

```php
public function update(
    Invoice $invoice,
    UpdateInvoiceData $data,
    CurrencyService $currency,
) {}
```

Authorization necesita:

```text
invoice
```

El descriptor:

```text
auth arguments:
[0]

post-auth arguments:
[1,2]
```

---

# 59. Performance Benefit

Si authorization deniega:

```text
DTO validation
service resolution
```

pueden evitarse.

---

# 60. PreResolution Extraction

Los requirements independientes del Subject deberán ejecutarse antes de binding.

---

# 61. Example

```text
admin.access
permission invoice.update
```

pueden evaluarse antes de cargar Invoice si sus evaluators no necesitan Subject.

---

# 62. Dependency Analysis

Durante compilation:

```text
Requirement
    ↓
requires Principal?
requires Tenant?
requires Subject?
requires Request Context?
```

---

# 63. Dependency Bitmask

Podrá representarse mediante flags.

Ejemplo:

```text
PRINCIPAL = 1
TENANT = 2
SUBJECT = 4
SECURITY = 8
```

---

# 64. Example

PermissionEvaluator:

```text
PRINCIPAL | TENANT
```

InvoicePolicy:

```text
PRINCIPAL | SUBJECT | TENANT
```

---

# 65. Benefit

El runtime puede saber rápidamente qué fase puede ejecutar.

---

# 66. Authorization Phases Compilation

Plan:

```text
PRE_RESOLUTION
RESOURCE
POST_RESOLUTION
```

deberá segmentarse durante compile.

---

# 67. Compiled Phase Arrays

Ejemplo:

```php
[
    'pre' => [4, 8],
    'resource' => [12, 21, 31],
    'post' => [],
]
```

---

# 68. No Runtime Sorting

Evaluator priority debe resolverse durante compilation.

---

# 69. Stable Ordering

El compiler deberá aplicar:

```text
priority
criticality
stable tie-breaker
```

una sola vez.

---

# 70. Short-Circuit Metadata

El plan puede precomputar qué evaluadores pueden terminar la evaluación.

---

# 71. Strategy-Specific Execution

`DenyOverrides` puede beneficiarse de ejecutar primero evaluators baratos con alta probabilidad de DENY.

---

# 72. Pero

La prioridad semántica y security requirements siempre tienen precedencia sobre optimización probabilística.

---

# 73. Security Order

Nunca mover:

```text
TenantIsolationPolicy
```

después de un evaluator menos seguro solo porque sea más costoso/barato si la arquitectura exige que sea primero.

---

# 74. Cost Metadata

Opcionalmente un evaluator podrá declarar:

```text
cost=cheap
cost=medium
cost=expensive
```

---

# 75. Optimizer

Un futuro:

```text
AuthorizationPlanOptimizer
```

podrá reorganizar evaluators solo dentro de grupos semánticamente conmutativos.

---

# 76. Commutativity

Dos evaluators son reorder-safe solo si:

```text
same phase
same priority/security class
strategy semantics unchanged
no required execution ordering
```

---

# 77. Default

V1 deberá favorecer:

```text
deterministic priority order
```

sobre optimizer agresivo.

---

# 78. Optimization Rule

```text
Never optimize across a security boundary
whose semantics you cannot prove equivalent.
```

---

# 79. Evaluator Deduplication

Durante compile deberán eliminarse duplicates estructurales.

---

# 80. Example

La misma Permission requirement llega desde:

```text
Route
Controller Attribute
Ability metadata
```

Si son semánticamente idénticas:

```text
one evaluator invocation
```

puede ser suficiente.

---

# 81. Deduplication Identity

Debe incluir:

```text
evaluator type
requirement
scope
phase
strategy participation
```

---

# 82. Different Source Does Not Always Mean Different Check

Pero el trace podrá conservar múltiples metadata sources aunque runtime ejecute una sola comprobación.

---

# 83. Requirement Provenance

Compiled descriptor puede contener:

```text
sources=[route, controller, ability]
```

---

# 84. Performance Without Losing Explainability

Una única ejecución puede explicar que múltiples declaraciones convergieron en ella.

---

# 85. Compiled Metadata Cache Format

La representación recomendada será:

```text
PHP files returning arrays
```

o PHP classes generadas si benchmark demuestra beneficio.

---

# 86. Why PHP Arrays

Funcionan muy bien con:

```text
Opcache
preloading
immutable access
```

y evitan parsing de JSON/YAML.

---

# 87. Example

```php
<?php

return [
    'schema' => 1,
    'registry_version' => 'a7f3...',
    'abilities' => [
        // ...
    ],
];
```

---

# 88. No Serialization Objects by Default

Evitar depender de:

```text
serialize()
unserialize()
```

para metadata compilada.

---

# 89. Reasons

```text
security
backward compatibility
class changes
Opcache friendliness
```

---

# 90. Generated Artifact Directory

Podría utilizar:

```text
storage/framework/authorization/
```

o equivalente del framework.

---

# 91. Artifacts

Ejemplos:

```text
abilities.php
policies.php
gates.php
targets.php
plans.php
manifest.php
```

---

# 92. Single vs Multiple Files

Deberá benchmarkearse.

---

# 93. Small Apps

Un único:

```text
authorization.php
```

puede ser suficiente.

---

# 94. Large Apps

Podría dividirse por:

```text
module
target type
plan partition
```

para reducir memoria inicial.

---

# 95. Lazy Plan Loading

En aplicaciones enormes:

```text
load plan partition only when needed
```

podría ser útil.

---

# 96. But Persistent Workers

Una vez cargadas, partitions pueden mantenerse.

---

# 97. Opcache

Authorization compiled files deben ser:

```text
Opcache-friendly
```

---

# 98. Opcache Preload

Opcionalmente:

```text
core authorization registry
```

podrá incluirse en preload.

---

# 99. Preload Candidates

```text
Enums
Value Objects
Core Evaluators
Compiled Manifest Loader
Strategy implementations
```

---

# 100. Do Not Preload Request State

Nunca:

```text
AuthorizationSession
TenantContext
PrincipalContext
```

con estado mutable.

---

# 101. FrankenPHP Worker Model

VoltStack deberá aprovechar:

```text
persistent PHP process
```

sin asumir modelo PHP-FPM clásico.

---

# 102. Shared Worker State

Seguro para compartir:

```text
CompiledAuthorizationManifest
PolicyRegistry
AbilityRegistry
PlanTemplates
Stateless evaluator instances
Strategy instances
```

---

# 103. Unsafe Shared State

Nunca compartir mutablemente:

```text
Current Principal
Current Tenant
Current Subject
Current Decision
Current AuthorizationSession
Request memo
Current trace
```

---

# 104. Worker Bootstrap

Flujo:

```text
Worker Start
    ↓
Load compiled authorization
    ↓
Validate manifest
    ↓
Build immutable runtime tables
    ↓
Seal registries
    ↓
Serve many requests
```

---

# 105. Request Start

```text
Create runtime scope
Create PrincipalContext
Create TenantContext
Create AuthorizationSession
```

---

# 106. Request End

```text
clear memoization
clear traces
clear context
clear request-local references
```

---

# 107. Worker Safety Requirement

Todo runtime state deberá estar:

```text
request-scoped
fiber-local
execution-context-local
```

según runtime.

---

# 108. Persistent Policy Instances

Policy classes podrán ser shared solo si son:

```text
stateless
```

y sus dependencies también son seguras para shared lifecycle.

---

# 109. Policy Scope Metadata

El container puede marcar:

```text
shared
request
transient
```

---

# 110. Recommendation

La mayoría de Policies deberían ser:

```text
shared stateless services
```

cuando solo reciben services shared/stateless.

---

# 111. Policy with Request Dependency

Ejemplo peligroso:

```php
final class InvoicePolicy
{
    public function __construct(
        private Request $request
    ) {}
}
```

---

# 112. Better

Pasar información relevante mediante:

```text
AuthorizationContext
```

---

# 113. Benefit

Mejora:

```text
testability
persistent runtime safety
compilation
portability
```

---

# 114. Container Lookup Optimization

El Policy Dispatcher podrá recibir:

```text
PolicyInstanceTable
```

con instancias shared ya resueltas.

---

# 115. Request-Scoped Policy

Si el descriptor indica:

```text
service_scope=request
```

resolver una vez por request y memoizar en runtime scope.

---

# 116. Transient Policies

Deberán ser raras.

Cada invocación implica object creation.

---

# 117. Compiler Warning

Podrá advertir:

```text
high-frequency Policy is transient
```

---

# 118. Object Allocation

Hot path deberá evitar crear muchos DTOs temporales.

---

# 119. Immutable Request Object

Un:

```text
AuthorizationRequest
```

puede seguir siendo útil, pero su diseño deberá ser compacto.

---

# 120. Alternative Internal Fast Path

La API pública puede crear `AuthorizationRequest`, mientras integration interna de Controller puede usar una estructura pre-normalizada.

---

# 121. Avoid Premature Micro-Optimization

No crear dos arquitecturas divergentes sin benchmarks.

---

# 122. Benchmark-Driven Changes

Toda optimización compleja deberá justificar:

```text
measurable latency
memory
allocation
query reduction
```

---

# 123. Policy Invocation Overhead

VoltStack deberá benchmarkear:

```text
method call
call_user_func
Reflection invoke
generated closure
direct dispatcher call
```

---

# 124. Likely Preference

Evitar:

```text
ReflectionMethod::invoke()
```

en hot path.

---

# 125. Direct Method Invocation Challenge

El método concreto cambia por Ability.

Se puede utilizar:

```php
$policy->{$descriptor->method}(...$args);
```

sin Reflection.

---

# 126. Dynamic Method Call

PHP dynamic invocation puede ser suficientemente rápida.

Benchmark decidirá si code generation aporta beneficio real.

---

# 127. Generated Policy Dispatcher

Para aplicaciones extremas, compiler podría generar:

```php
match ($policyMethodId) {
    1 => $invoicePolicy->view(...),
    2 => $invoicePolicy->update(...),
};
```

---

# 128. Not V1 Requirement

Esto aumenta:

```text
generated code
cache complexity
deployment complexity
```

y solo deberá implementarse si benchmarks lo justifican.

---

# 129. Ability Canonicalization

No normalizar strings complejos repetidamente.

---

# 130. API Call

```php
Authorization::can('invoice.update', $invoice);
```

deberá resolver rápidamente:

```text
Ability ID
```

---

# 131. Ability Intern Pool

Los IDs pueden actuar como canonical representation interna.

---

# 132. Route Integration Optimization

Route descriptor ya puede contener Ability IDs directamente.

---

# 133. Attribute Integration Optimization

Compiled Controller metadata también.

---

# 134. Programmatic Authorization

Solo llamadas dinámicas necesitan:

```text
string → Ability ID
```

lookup.

---

# 135. Permission IDs

RBAC puede usar el mismo patrón:

```text
permission string → permission ID
```

---

# 136. Role IDs

Igualmente para Roles estáticos.

---

# 137. Dynamic Permissions

Si la aplicación permite permissions creadas en DB:

```text
numeric runtime IDs
```

podrán venir del Grant Provider.

---

# 138. Plan Cache

Ya definido en documento 14.

La compilación deberá producir tantos templates como sea razonablemente posible.

---

# 139. Dynamic Plan Construction

Solo necesaria para:

```text
runtime-defined abilities
dynamic policy providers
external modules
custom runtime metadata
```

---

# 140. Dynamic Plan Cache

Estos plans podrán almacenarse en:

```text
worker-local PlanCache
```

con registry version.

---

# 141. Cold Start

FrankenPHP worker recién iniciado deberá minimizar latencia de primer request.

---

# 142. Warmup

Deployment podrá ejecutar:

```text
authorization warmup
```

---

# 143. AuthorizationWarmup

Podrá:

```text
load manifest
validate files
instantiate core evaluators
prime common plans
```

---

# 144. Do Not Execute User-Specific Checks During Warmup

Nunca cargar:

```text
user roles
tenant membership
```

en warmup global.

---

# 145. Pre-Warming Common Abilities

Solo estructuras:

```text
invoice.view plan
admin.access plan
```

---

# 146. Command

Futuro:

```text
volt authorization:compile
```

---

# 147. Compile Command

Deberá:

```text
discover
validate
compile
write artifacts
report diagnostics
```

---

# 148. Command Output

Ejemplo:

```text
Authorization compiled successfully.

Abilities:        184
Policies:          42
Gates:             18
Evaluators:        11
Targets:          327
Plan templates:   219
Warnings:           0
```

---

# 149. Warmup Command

```text
volt authorization:warm
```

podrá validar que artifacts puedan cargarse.

---

# 150. Optimize Integration

VoltStack podrá integrar Authorization en un comando general:

```text
volt optimize
```

---

# 151. Optimize Could Compile

```text
routes
controllers
container
authorization
config
```

de forma coordinada.

---

# 152. Coordinated Versions

Esto permitirá generar un:

```text
Application Compilation Fingerprint
```

---

# 153. Cross-System Manifest

Route cache y Authorization cache podrán compartir:

```text
build ID
```

para evitar incompatibilidades.

---

# 154. Atomic Compilation

Los nuevos artifacts no deberán hacerse visibles parcialmente.

---

# 155. Deployment Pattern

```text
compile to temp directory
        ↓
validate
        ↓
atomic switch
```

---

# 156. Why

Nunca permitir:

```text
new routes
+
old authorization
```

durante deployment.

---

# 157. Manifest Atomicity

La versión activa deberá referenciar un conjunto coherente de artifacts.

---

# 158. Build Directory

Ejemplo:

```text
authorization/builds/{build-id}/
```

---

# 159. Active Pointer

Podrá existir:

```text
authorization/current.php
```

o equivalente controlado.

---

# 160. Opcache Invalidation

Deployment deberá coordinarse con el mecanismo global del runtime.

---

# 161. Worker Reload

Si el manifest cambia:

```text
existing workers
```

no deberían mezclar metadata nueva con código viejo.

---

# 162. Recommendation

Usar:

```text
worker restart / graceful reload
```

para cambios estructurales de autorización.

---

# 163. Hot Reload Development

En development sí puede recompilarse dinámicamente.

---

# 164. Development Mode

Prioridad:

```text
developer feedback
```

más que micro-rendimiento.

---

# 165. Development Runtime

Puede usar Reflection y rebuild incremental.

---

# 166. Production Runtime

Debe usar:

```text
compiled mode
```

por defecto en deployments optimizados.

---

# 167. Compilation Modes

Propuesta:

```php
enum AuthorizationCompilationMode: string
{
    case Dynamic = 'dynamic';
    case Cached = 'cached';
    case Compiled = 'compiled';
}
```

---

# 168. Dynamic

Ideal para:

```text
development
rapid iteration
```

---

# 169. Cached

Metadata descubierta se cachea.

---

# 170. Compiled

Todo lo estructural posible se genera antes del runtime.

---

# 171. Production Recommendation

```text
Compiled
```

---

# 172. Strict Compiled Mode

Podrá prohibir fallback a Reflection.

---

# 173. Why

Si artifact falta, mejor detectar deployment incorrecto que degradar silenciosamente rendimiento/semántica.

---

# 174. Compilation Failure

Debe detener:

```text
build/deployment
```

cuando el error afecta seguridad.

---

# 175. Warnings vs Errors

Warning:

```text
transient Policy on hot path
```

Error:

```text
unknown nonBypassable evaluator
invalid Policy signature
```

---

# 176. Incremental Compilation

Para development, el compiler podrá recompilar solo:

```text
modified Controller
modified Policy
affected Plans
```

---

# 177. Dependency Graph

Necesario para saber qué artifacts invalidar.

---

# 178. AuthorizationCompilationGraph

Podrá modelar:

```text
Policy → Abilities
Ability → Plans
Controller → Targets
Route → Targets
Evaluator → Plans
```

---

# 179. Example

Cambio en:

```text
InvoicePolicy
```

invalida:

```text
Invoice abilities
Invoice-related plans
affected controller targets
```

---

# 180. Global Evaluator Change

Puede invalidar:

```text
all plans
```

---

# 181. Strategy Change

Invalida plans que usan esa strategy.

---

# 182. Attribute Change

Solo targets afectados.

---

# 183. Dependency Fingerprints

Podrán utilizar hashes por component.

---

# 184. Full Build

En CI/production, un rebuild completo puede ser más sencillo y seguro.

---

# 185. Runtime Lookup Complexity

Objetivo típico:

```text
Ability lookup:
O(1)

Policy lookup:
O(1)

Plan lookup:
O(1)

Evaluator instance lookup:
O(1)
```

---

# 186. ReBAC Exception

Graph traversal depende del tamaño del graph y no puede reducirse siempre a O(1).

---

# 187. Permission Lookup

Request memoization debería reducir checks repetidos a aproximadamente:

```text
O(1)
```

después del primer lookup.

---

# 188. Plan Execution Complexity

Idealmente proporcional a:

```text
number of evaluators actually executed
```

---

# 189. Short-Circuit

Puede reducir esa cantidad.

---

# 190. Evaluator Cost Budget

Cada autorización podrá tener un:

```text
latency budget
```

opcional.

---

# 191. AuthorizationPerformanceBudget

Conceptualmente:

```php
final readonly class AuthorizationPerformanceBudget
{
    public function __construct(
        public ?int $maxDurationMicroseconds,
        public ?int $maxExternalCalls,
        public ?int $maxQueries,
    ) {}
}
```

---

# 192. Not Security Decision

Exceder budget no debería producir GRANT.

---

# 193. Timeout

Si una autorización crítica excede su deadline:

```text
FAILURE
```

y fail closed.

---

# 194. Development Budget

Puede generar warnings.

---

# 195. Production Deadline

External evaluators deberán respetar deadlines para evitar requests colgadas.

---

# 196. Central Deadline

AuthorizationExecutionContext podrá contener:

```text
deadline
```

---

# 197. Remaining Time

Cada external adapter obtiene:

```text
remaining budget
```

---

# 198. Query Budget

Profiler/testing puede advertir:

```text
permission check generated 15 queries
```

---

# 199. Runtime Enforcement

No necesariamente bloquear por número de queries en V1.

---

# 200. Batch Authorization

Un sistema empresarial necesita autorizar colecciones eficientemente.

---

# 201. Naive Pattern

```php
foreach ($invoices as $invoice) {
    $authorization->can(
        'view',
        $invoice
    );
}
```

puede causar:

```text
N Policy evaluations
N relationship lookups
N database queries
```

---

# 202. Batch API

Podrá existir:

```php
Authorization::batch()
    ->ability('view')
    ->subjects($invoices)
    ->inspect();
```

---

# 203. BatchAuthorizationRequest

Conceptualmente:

```text
Principal
Ability
Subjects[]
Context
```

---

# 204. Batch Result

```text
Subject ID → Decision
```

---

# 205. Shared Evaluators

El batch engine puede ejecutar una sola vez:

```text
Principal state
Tenant membership
Global permission
MFA
```

---

# 206. Subject Evaluators

Luego:

```text
Tenant isolation
Resource Policy
ABAC resource conditions
```

por Subject.

---

# 207. ReBAC Batch

Un provider podrá resolver:

```text
relationships for N subjects
```

en una query.

---

# 208. Policy Batch Support

Policies podrán implementar opcionalmente:

```php
interface BatchAuthorizationPolicyInterface
```

---

# 209. Example

```php
public function viewMany(
    User $user,
    iterable $invoices,
    AuthorizationContext $context,
): iterable;
```

---

# 210. Not Required

Policies normales seguirán funcionando.

---

# 211. Batch Adapter

Si no existe batch implementation:

```text
fallback to per-subject evaluation
```

usando memoization.

---

# 212. Query-Driven Authorization

Para listados, a menudo es mejor:

```text
authorization-aware query scope
```

que evaluar 10,000 rows después de cargarlas.

---

# 213. Example

```text
visibleInvoicesFor(User#42)
```

debería producir query filtrada cuando la regla sea traducible.

---

# 214. AuthorizationQueryConstraint

Un subsystem futuro puede generar constraints.

---

# 215. Important

No toda Policy es traducible a SQL.

---

# 216. Compiler Could Mark

```text
query_translatable=true/false
```

para reglas soportadas.

---

# 217. ViewAny Optimization

Antes de query:

```text
viewAny
```

puede evaluarse una vez.

---

# 218. Permission-First Listing

Si Principal carece de:

```text
invoice.view
```

el sistema puede evitar completamente la query.

---

# 219. PreAuthorization Benefits

Este es un caso clave de optimization by phase.

---

# 220. N+1 Detection

Profiler debe detectar:

```text
same evaluator
same Principal
same scope
repeated source lookup
```

---

# 221. Recommended Automatic Memoization

RBAC checks:

```text
Role
Permission
Membership
```

son excelentes candidatos request-local.

---

# 222. ReBAC Memoization

Relaciones repetidas también.

---

# 223. ABAC

Atributos globales del Principal pueden memoizarse.

---

# 224. Resource Attributes

Dependen de cada Subject.

---

# 225. Lazy Attribute Resolution

No obtener:

```text
risk score
geo
device trust
```

si un evaluator anterior ya denegó.

---

# 226. Lazy Context Providers

AuthorizationContext podrá ofrecer valores lazily.

---

# 227. But Snapshot Semantics

Una vez materializado un atributo security-relevant, deberá mantenerse estable durante esa decisión.

---

# 228. Cost-Aware Evaluation

Un futuro optimizer podrá ejecutar primero condiciones:

```text
cheap
high rejection probability
```

si son conmutativas.

---

# 229. Example

```text
permission check 0.05 ms
remote compliance check 20 ms
```

Si ambas son required y conmutativas:

```text
permission first
```

es razonable.

---

# 230. Security Guard

El optimizer solo puede hacerlo con metadata explícita.

---

# 231. Evaluator Purity

Optimización/reordering depende de que evaluators sean:

```text
side-effect free
```

---

# 232. Side Effects

Un evaluator con side effects no debería formar parte del modelo estándar.

---

# 233. Why

Short-circuit/reordering harían comportamiento impredecible.

---

# 234. Compiler Warning

Un evaluator marcado:

```text
side_effecting
```

debería rechazarse para ejecución normal.

---

# 235. Audit Is Separate

Audit side effects ocurren en fase dedicada, no dentro de evaluator.

---

# 236. JIT-Like Runtime Optimization

No es necesario implementar un JIT propio.

PHP Opcache ya proporciona optimización significativa.

---

# 237. Focus Areas

Priorizar:

```text
fewer lookups
fewer DB queries
less Reflection
less object construction
less repeated work
```

sobre micro-optimizar operadores PHP.

---

# 238. Memory Layout

Workers persistentes pueden mantener miles de descriptors.

---

# 239. Compact Descriptors

Evitar guardar en production:

```text
full source file
line
reflection objects
verbose debug metadata
```

si no se necesita.

---

# 240. Debug Metadata Partition

Puede almacenarse separadamente y cargarse solo en development.

---

# 241. Production Descriptor

```text
IDs
flags
method names
priorities
dependency masks
```

---

# 242. Development Descriptor

Añade:

```text
source file
line
metadata provenance
human names
```

---

# 243. String Duplication

Ability/Policy names repetidos muchas veces pueden aumentar memoria.

---

# 244. ID Tables

IDs internos reducen duplicación.

---

# 245. Interned Strings

PHP/Opcache ya puede ayudar parcialmente, pero IDs siguen siendo útiles.

---

# 246. PlanTemplate Sharing

Múltiples targets con mismo plan estructural podrían referenciar el mismo template.

---

# 247. Example

100 CRUD routes que requieren:

```text
authenticated
tenant membership
permission
resource policy
```

pueden compartir partes estructurales.

---

# 248. Plan Interning

El compiler podrá fingerprint plans y deduplicar templates idénticos.

---

# 249. Plan Fingerprint

Debe incluir toda semántica relevante.

---

# 250. Do Not Deduplicate Incorrectly

Dos plans con mismo evaluator list pero distinto:

```text
strategy
phase
requirement
subject binding
```

no son iguales.

---

# 251. Compile-Time Constant Folding

Algunas condiciones estáticas pueden resolverse durante compile.

---

# 252. Example

Si feature/config:

```text
Authorization evaluator disabled permanently
```

puede excluirse del plan.

---

# 253. But Environment-Specific Build

El artifact queda ligado al environment/config fingerprint.

---

# 254. Do Not Constant-Fold Dynamic Security State

Nunca predecidir:

```text
Tenant active
User admin
MFA complete
```

durante compile.

---

# 255. Static PublicAccess

Una ruta `#[PublicAccess]` sí puede omitir requirements application-level innecesarios durante compile, respetando global security boundaries.

---

# 256. Static Conflict Detection

Compiler debe eliminar runtime branches detectando:

```text
PublicAccess + RequiresRole
```

como error.

---

# 257. Feature-Specific Plans

Si feature flags cambian dinámicamente, no constant-fold salvo que sean build-time flags.

---

# 258. Configuration Immutability

Compiled mode asume ciertas configs inmutables durante worker lifetime.

---

# 259. Dynamic Configuration

Si cambia:

```text
authorization strategy
global evaluator set
```

deberá generar nueva registry version/reload.

---

# 260. No Mutable Registry

No añadir evaluators a registry shared durante request.

---

# 261. Registry Sealing

Después de bootstrap:

```text
PolicyRegistry
AbilityRegistry
GateRegistry
StrategyRegistry
EvaluatorRegistry
```

deberán sellarse.

---

# 262. SealedRegistry

Intentar:

```text
register new Policy
```

después:

```text
RegistrySealedException
```

---

# 263. Benefits

```text
determinism
thread/fiber safety
cache correctness
plan stability
```

---

# 264. Plugin Registration

Plugins deben registrar durante bootstrap/compile.

---

# 265. Runtime Plugin Loading

No debe modificar silenciosamente autorización en worker activo.

---

# 266. Dynamic Tenant Policies

Si ciertos tenants poseen reglas configurables, la estructura del evaluator puede seguir compilada mientras los datos de policy son runtime.

---

# 267. Example

Compiled evaluator:

```text
TenantRuleEvaluator
```

Runtime data:

```text
Tenant#7 approval limit
```

---

# 268. Avoid Per-Tenant Plan Explosion

No compilar un PlanTemplate distinto por cada Tenant si solo cambian valores de configuración.

---

# 269. Structural vs Data Variation

```text
structure
→ compiled

tenant-specific values
→ runtime
```

---

# 270. Tenant-Specific Policy Modules

Si tenants pueden habilitar módulos que cambian estructura, usar:

```text
tenant policy profile ID
```

como parte de plan cache key.

---

# 271. Profile Count

Debe evitarse explosión combinatoria de templates.

---

# 272. Policy Profile

Ejemplo:

```text
standard
regulated_finance
healthcare
enterprise
```

---

# 273. Profile Plan Cache

Puede compartir templates entre tenants del mismo profile.

---

# 274. External Policy Engines

Remote PDP calls dominarán latencia mucho más que PHP dispatch.

---

# 275. Optimization Focus

Para external evaluators:

```text
batching
connection reuse
timeouts
request memoization
versioned cache
```

son más importantes.

---

# 276. HTTP Client Reuse

En persistent worker puede reutilizar conexiones si el client es seguro.

---

# 277. External Request Payload

Enviar solo información necesaria.

No serializar Subject completo.

---

# 278. Precomputed External Payload Mapping

Puede compilarse qué fields requiere el evaluator.

---

# 279. Sensitive Data

Reducir payload también mejora seguridad.

---

# 280. External Batching

Si se evalúan múltiples resources, enviar batch cuando el provider lo soporte.

---

# 281. Distributed Authorization

En microservices, Authorization puede ejecutarse repetidamente.

---

# 282. Context Propagation

No propagar decisiones GRANT como autoridad genérica salvo capability/delegation explícita.

---

# 283. Performance Temptation

Evitar:

```text
Service A already authorized,
so Service B trusts arbitrary header.
```

---

# 284. Correct

Usar:

```text
signed delegated authorization context
```

si se diseña formalmente.

---

# 285. Compilation Does Not Change Trust Model

Optimización nunca debe convertirse en bypass.

---

# 286. Startup Validation

Al cargar compiled artifacts:

```text
schema compatible?
build ID compatible?
registry fingerprint valid?
```

---

# 287. CompiledAuthorizationLoader

Contrato:

```php
interface CompiledAuthorizationLoaderInterface
{
    public function load(
        string $path
    ): CompiledAuthorizationManifest;
}
```

---

# 288. Loader Validation

Debe verificar:

```text
schema
application ID
environment
build version
required sections
```

---

# 289. Corrupt Artifact

Resultado:

```text
boot failure
```

en strict compiled mode.

---

# 290. Never

```text
corrupt authorization manifest
→ run application without authorization
```

---

# 291. Optional Development Fallback

En development:

```text
compiled load fails
→ rebuild dynamically
```

puede permitirse.

---

# 292. Production

Preferir:

```text
fail deployment / boot
```

---

# 293. Compilation Fingerprint

Puede incluir hashes de:

```text
authorization config
policies
abilities
gates
metadata adapters
strategies
routes
controllers
```

---

# 294. Avoid Hashing Full Files Per Request

Fingerprint se genera en build/compile.

---

# 295. Runtime Version

Solo cargar el valor ya calculado.

---

# 296. Authorization Runtime

Podrá existir:

```text
CompiledAuthorizationRuntime
```

que centralice las tablas.

---

# 297. Conceptual API

```php
final class CompiledAuthorizationRuntime
{
    public function ability(
        string $name
    ): AbilityDescriptor;

    public function plan(
        int $abilityId,
        int $subjectTypeId,
        int $targetId = 0,
    ): AuthorizationPlanTemplate;
}
```

---

# 298. Runtime Should Be Stateless

El objeto puede ser shared porque solo contiene metadata inmutable.

---

# 299. Execution Engine

Separado:

```text
AuthorizationExecutionEngine
```

trabaja con request-local context.

---

# 300. Separation

```text
CompiledAuthorizationRuntime
→ shared immutable structure

AuthorizationExecution
→ request-local state
```

---

# 301. Performance Metrics

El sistema deberá medir:

```text
authorization.total_duration
authorization.plan_lookup_duration
authorization.plan_build_duration
authorization.evaluator_duration
authorization.container_resolution_duration
authorization.external_duration
```

---

# 302. Allocation Metrics

Si tooling lo permite:

```text
objects created
memory allocated
```

podrán medirse en benchmarks.

---

# 303. Query Metrics

Especialmente:

```text
RBAC queries
ReBAC queries
Tenant queries
```

---

# 304. Cache Metrics

Ya definidas, pero rendimiento debe correlacionarlas con latency.

---

# 305. Cold vs Warm

Benchmarks deben separar:

```text
cold worker
warm worker
```

---

# 306. FrankenPHP Warm Worker

Este será uno de los escenarios principales de optimización.

---

# 307. Baseline Scenarios

Medir:

```text
simple Gate
single Policy
RBAC + Policy
multi-tenant Policy
RBAC + ReBAC + ABAC
cached permission
batch authorization
```

---

# 308. Benchmark Should Include DENY

Short-circuit behavior puede hacer DENY más rápido que GRANT.

---

# 309. Critical Worst Case

También medir:

```text
all evaluators execute
```

---

# 310. Performance Regression Budget

Podrá existir:

```text
max allowed regression %
```

en CI.

---

# 311. Avoid Brittle Microbenchmarks

Rendimiento varía por:

```text
PHP version
CPU
Opcache
OS
```

Comparar principalmente contra baseline del mismo entorno.

---

# 312. Compilation Time

También debe medirse.

---

# 313. Large Application Test

Ejemplo synthetic:

```text
5,000 abilities
1,000 Policies
20,000 targets
```

para evaluar:

```text
compile time
artifact size
worker memory
lookup latency
```

---

# 314. Memory Budget

Un authorization manifest gigantesco no debe consumir memoria desproporcionada.

---

# 315. Lazy Partitions

Podrán introducirse si large-app benchmarks lo requieren.

---

# 316. Partition Key

Puede ser:

```text
module
package
route group
subject namespace
```

---

# 317. Module Compilation

VoltStack Quantum modules podrían producir:

```text
authorization fragments
```

---

# 318. Fragment Merge

Durante application compile:

```text
module manifests
    ↓
merge/validate
    ↓
application manifest
```

---

# 319. Package Isolation

IDs internos deberán reasignarse al compilar para evitar colisiones.

---

# 320. Package Metadata

Cada package puede declarar:

```text
abilities
policies
evaluators
reason codes
```

---

# 321. No Package Runtime Scan

Todo debe integrarse en manifest durante bootstrap/compile.

---

# 322. Compiler Extensibility

Packages podrán registrar:

```text
AuthorizationCompilerPass
```

---

# 323. Compiler Pass

Conceptualmente:

```php
interface AuthorizationCompilerPassInterface
{
    public function process(
        AuthorizationCompilationState $state
    ): void;
}
```

---

# 324. Examples

```text
PolicyDiscoveryPass
GateCompilationPass
AttributeMetadataPass
PlanOptimizationPass
ValidationPass
```

---

# 325. Pass Ordering

Debe ser determinista.

---

# 326. Compiler Pass Security

Custom packages no deberán modificar final compiled authorization después de validation seal sin pasar nueva validación.

---

# 327. Final Validation Pass

Antes de escribir artifacts:

```text
validate final graph
```

---

# 328. Seal Compilation State

Después:

```text
immutable manifest
```

---

# 329. Compilation Diagnostics

Deberán producir:

```text
errors
warnings
optimization hints
```

---

# 330. Example Hint

```text
InvoicePolicy is resolved dynamically
but can be compiled.
```

---

# 331. Example Warning

```text
FinanceCompliancePolicy depends on Request service
and cannot be shared across FrankenPHP requests.
```

---

# 332. Example Error

```text
TenantIsolationPolicy missing from tenant-owned Invoice plan.
```

---

# 333. Optimization Report

Comando:

```text
volt authorization:optimize-report
```

podrá mostrar:

```text
Compiled plans: 94%
Dynamic plans: 6%
Zero-reflection targets: 100%
Shared evaluators: 24
Request evaluators: 3
Transient evaluators: 1
```

---

# 334. Performance Explain

Para una Ability:

```text
volt authorization:performance invoice.update
```

podrá mostrar:

```text
Plan:
compiled

Policy:
shared

Reflection:
none

Permission:
request memoized

Potential external calls:
0

Estimated evaluator count:
4
```

---

# 335. Dynamic Hotspot Detection

Profiler podrá detectar targets que todavía requieren:

```text
Reflection
dynamic discovery
repeated service resolution
```

---

# 336. Development Recommendation

Tooling puede sugerir:

```text
run volt optimize
```

para simular production path.

---

# 337. Testing Compiled Mode

La security suite deberá ejecutarse también en:

```text
compiled mode
```

no solo dynamic.

---

# 338. Differential Verification

```text
Dynamic Authorization Runtime
        ↓
Decision A

Compiled Authorization Runtime
        ↓
Decision B

A must equal B
```

---

# 339. All Critical Abilities

Este differential testing debe ser obligatorio.

---

# 340. Cache Independence

Compiled runtime no debe depender de distributed decision cache para ser rápido.

---

# 341. Core Fast Path

Incluso con:

```text
Redis unavailable
```

la estructura Authorization seguirá optimizada.

---

# 342. Security Over Performance

Si existe conflicto entre:

```text
performance optimization
```

y:

```text
correct authorization
```

gana siempre:

```text
correctness/security
```

---

# 343. No Unsafe Fast Path

No crear:

```php
if ($user->isAdmin()) {
    return true;
}
```

como fast path global fuera del Planner.

---

# 344. Why

Podría saltarse:

```text
TenantIsolation
Suspension
Compliance
NonBypassable rules
```

---

# 345. Safe Fast Paths

Solo pueden existir cuando el compiled plan demuestra que el resultado es semánticamente equivalente.

---

# 346. Example Safe Short Circuit

`DenyOverrides` + NonBypassable DENY:

```text
DENY
→ stop
```

---

# 347. Unsafe Short Circuit

Ordinary GRANT antes de evaluators críticos:

```text
not allowed
```

---

# 348. GRANT Short-Circuit

Solo estrategias que lo permitan y después de respetar evaluators non-bypassable/mandatory.

---

# 349. Critical Evaluator Partition

El plan podrá separar:

```text
mandatory
ordinary
```

---

# 350. Example

```text
Mandatory:
PrincipalState
TenantIsolation

Ordinary:
Permission
InvoicePolicy
```

Un `Affirmative` GRANT dentro de ordinary no puede omitir mandatory.

---

# 351. Precompiled Critical Mask

PlanTemplate puede tener:

```text
mandatory evaluator indexes
```

---

# 352. Branch Reduction

Esto permite un execution loop simple y seguro.

---

# 353. Runtime Loop Conceptual

```php
foreach ($plan->evaluators as $descriptor) {
    $result = $engine->evaluate(
        $descriptor,
        $execution
    );

    if ($strategy->canShortCircuit(...)) {
        break;
    }
}
```

---

# 354. Specialized Strategy Loop

Se podrá optimizar en el futuro, pero mantener primero claridad.

---

# 355. DecisionManager Overhead

Debe ser pequeño comparado con evaluator work.

---

# 356. Preallocate Vote Storage

Para planes pequeños puede evitarse crear colecciones pesadas.

---

# 357. Streaming Decisions

DecisionManager puede procesar votos incrementalmente.

---

# 358. Benefit

No necesita guardar todos los votos si:

```text
tracing disabled
```

y strategy permite streaming.

---

# 359. Trace Mode

Con full tracing, deberá conservar detalles adicionales.

---

# 360. Dual Execution Mode

```text
Minimal runtime mode
Detailed trace mode
```

con misma semántica.

---

# 361. DecisionAccumulator

Podrá mantener:

```text
grant count
deny count
abstain count
decisive evaluator
```

sin guardar objetos completos.

---

# 362. Full Vote Objects

Solo si:

```text
trace/audit/explain
```

lo requiere.

---

# 363. Lazy Explanation

Ya definido, pero clave para rendimiento.

---

# 364. Audit Construction

Construir AuditRecord solo si:

```text
AuditPolicyResolver
```

indica que será necesario.

---

# 365. Metrics

Counters simples podrán mantenerse con bajo overhead.

---

# 366. Sampling

Evaluator spans detallados solo cuando sampleado.

---

# 367. Error Path

Optimizar happy path no debe degradar diagnóstico de failures.

---

# 368. Source Metadata Lookup

Debug source details pueden cargarse solo cuando existe error/trace.

---

# 369. Source Map

Compiled artifacts podrán tener un:

```text
authorization source map
```

separado.

---

# 370. Production Error Explain

Al ocurrir failure:

```text
descriptor ID
    ↓
source map lookup
```

si debug tooling está disponible.

---

# 371. Not Hot Path

Así se conserva diagnóstico sin pagar costo normal.

---

# 372. Authorization Source Map

Podrá incluir:

```text
Policy ID → file/line
Target ID → controller/method
Attribute descriptor → source
```

---

# 373. Source Map Optional

Puede omitirse en hardened production deployments.

---

# 374. Runtime Statistics

Workers podrán acumular métricas agregadas, pero no estado por Principal.

---

# 375. Safe Shared Metrics

Counters son posibles si la metrics infrastructure los soporta correctamente.

---

# 376. No Decision State

Nunca:

```text
lastAllowedUser
```

en shared runtime.

---

# 377. Memory Leak Detection

Testing debe inspeccionar:

```text
AuthorizationSession references
Subject references
Principal references
```

después del request.

---

# 378. Weak References

No deberían ser necesarias si lifecycle está bien diseñado.

---

# 379. Long-Lived Closures

Evitar closures shared que accidentalmente capturen:

```text
Principal
Tenant
Request
```

---

# 380. Compiler Warning

Si una compiled closure captura objetos request-scoped:

```text
reject
```

cuando pueda detectarse.

---

# 381. Fiber/Concurrency Safety

En runtimes concurrentes:

```text
request-local authorization state
```

debe mantenerse por execution context.

---

# 382. Avoid Global Static Context

No:

```php
AuthorizationContext::$current
```

---

# 383. Context Local Abstraction

Usar el runtime context system de VoltStack.

---

# 384. Concurrent Batch Authorization

Podría ejecutar evaluators externos en paralelo en el futuro.

---

# 385. V1 Recommendation

No introducir paralelismo interno hasta tener:

```text
correct context propagation
cancellation
deadline handling
deterministic tracing
```

---

# 386. Potential Benefit

ABAC/ReBAC remotos independientes podrían reducir latencia total.

---

# 387. Potential Risk

```text
more external load
complex failure aggregation
lost short-circuit savings
```

---

# 388. Default

Sequential, optimized by phase and short-circuit.

---

# 389. Deadline-Aware Parallelism

Podrá estudiarse en versiones futuras.

---

# 390. Authorization Compiler Passes Propuestos

Orden conceptual:

```text
1 Source Discovery
2 Attribute Extraction
3 Ability Normalization
4 Policy Normalization
5 Gate Normalization
6 Extension Metadata
7 Metadata Merge
8 Contract Validation
9 Subject Type Analysis
10 Dependency Analysis
11 Phase Assignment
12 Priority Ordering
13 Deduplication
14 Plan Construction
15 Plan Optimization
16 Security Validation
17 Fingerprint Generation
18 Artifact Generation
19 Final Validation
```

---

# 391. Security Validation After Optimization

Crítico.

Nunca asumir que si el plan era seguro antes de optimizer seguirá siéndolo.

---

# 392. Post-Optimization Checks

Verificar:

```text
NonBypassable present
required evaluators present
phase constraints valid
strategy compatible
TenantIsolation preserved
```

---

# 393. Optimization Proof Metadata

Un optimizer podrá registrar:

```text
what transformation was applied
```

para tooling.

---

# 394. Example

```text
PermissionEvaluator moved before RelationshipEvaluator

Reason:
same phase
equal security priority
both side-effect free
DenyOverrides commutative in this group
```

---

# 395. V1

Puede omitir optimizer dinámico complejo y mantener orden fijo.

---

# 396. Compiler Determinism

Mismo código + misma config debe producir:

```text
same manifest semantically
```

---

# 397. Stable Output

Idealmente también mismo hash, ignorando timestamps/build IDs cuando se excluyan del fingerprint.

---

# 398. Deterministic Build Benefits

```text
reproducibility
cache validation
CI diffs
security reviews
```

---

# 399. No Random Internal IDs

IDs pueden asignarse por orden canonical estable.

---

# 400. Example

Ordenar Abilities alfabéticamente antes de asignar IDs.

---

# 401. Incremental Build Caveat

Debe preservar stable IDs o regenerar todo manifest coherentemente.

---

# 402. Internal IDs Not Persistent API

Nunca almacenar Ability internal ID permanentemente en DB esperando estabilidad entre releases.

---

# 403. Persistent Data Uses Canonical Names

Audit, domain data y external protocols deberán usar:

```text
canonical ability string
```

o versioned public identifiers.

---

# 404. Internal Runtime IDs

Solo optimización efímera por build.

---

# 405. Security Artifact Integrity

Compiled authorization files deberán tratarse como:

```text
trusted application code artifacts
```

---

# 406. File Permissions

Deployment debe impedir modificación por usuarios no autorizados.

---

# 407. Manifest Signature/Hash

Podrá verificarse integridad mediante build manifest general.

---

# 408. Not User-Controlled

Nunca cargar authorization manifest indicado por request input.

---

# 409. Container Optimization

Authorization services pueden compilarse en el Container.

---

# 410. Constructor Dependency Resolution

Policies shared pueden instanciarse en worker bootstrap o lazy-first-use.

---

# 411. Eager vs Lazy

```text
eager
=
higher startup
lower first check latency

lazy
=
faster worker boot
small first-use cost
```

---

# 412. Recommended

Core evaluators eager.

Application Policies lazy shared, salvo profiling que indique otra cosa.

---

# 413. Policy Instance Cache

Worker-local:

```text
Policy ID → shared Policy instance
```

---

# 414. Request Policy Cache

Para request-scoped Policies:

```text
AuthorizationSession
```

puede almacenar instancia.

---

# 415. Service Lifetime Validation

Compiler/Container integration deberá comprobar que una shared Policy no dependa de un request-scoped service de forma incompatible.

---

# 416. Lifetime Mismatch

Ejemplo:

```text
Shared InvoicePolicy
    ↓ depends on
RequestScoped CurrentRequest
```

debe:

```text
error/warning
```

según container semantics.

---

# 417. Preferred Dependency

```text
Clock
PermissionChecker
RelationshipChecker
Domain Repository
```

siempre que sean lifecycle-safe.

---

# 418. Repository Cost

Policy que hace queries por sí misma puede convertirse en hotspot.

---

# 419. Profiling Recommendation

Detectar:

```text
Policy database queries
```

y optimizar mediante:

```text
preloaded Subject
batch resolver
memoization
```

---

# 420. Avoid Hidden Queries

Property access que lazy-load relations dentro de Policy puede provocar N+1.

---

# 421. Compiler Cannot Always Detect

Profiler/testing sí puede.

---

# 422. Policy Design Recommendation

Policies hot-path deben preferir datos ya disponibles o explícitamente precargados.

---

# 423. Authorization Data Loader

Futuro:

```text
AuthorizationDataLoader
```

podrá precargar relaciones requeridas por un plan.

---

# 424. Dependency Declaration

Policy descriptor puede declarar:

```text
requires relationship invoice.organization
```

si el framework ofrece metadata suficiente.

---

# 425. Batch Preloading

Listados podrían cargar:

```text
organizations
owners
```

en conjunto.

---

# 426. Avoid ORM Coupling

Authorization Core no debe depender de ORM.

Un adapter del Data/ORM system puede aprovechar metadata.

---

# 427. Database Read/Write Split

Authorization reads sensibles a revocación pueden requerir primary.

---

# 428. Performance Tradeoff

No forzar replica lenta/obsoleta solo por velocidad.

---

# 429. Consistency Policy

Documento 14 sigue gobernando esto.

---

# 430. Performance Cannot Override Consistency

Ability strong-consistency:

```text
no stale replica/cached decision
```

aunque sea más costoso.

---

# 431. Synchronous Audit

Required audit puede aumentar latencia.

---

# 432. Optimization

Usar:

```text
transactional outbox
```

en lugar de remote SIEM sync cuando el guarantee lo permita.

---

# 433. BestEffort Audit

Puede emitirse async.

---

# 434. Trace Sampling

Reduce overhead.

---

# 435. Explain on Demand

No construir explanation tree por default.

---

# 436. Production Modes

Propuesta:

```text
authorization.runtime.mode = compiled

authorization.trace.mode = errors

authorization.audit.default = denials

authorization.profiler = false
```

según aplicación.

---

# 437. Development Modes

```text
runtime.mode = dynamic/cached

trace.mode = full

profiler = true
```

---

# 438. Benchmark Mode

Podrá desactivar:

```text
audit
trace
metrics
```

para medir core puro y después medir producción real por separado.

---

# 439. Realistic Benchmark

También debe medirse con observability normal activa.

---

# 440. Performance Target Philosophy

No fijar todavía:

```text
Authorization must be < 0.1 ms
```

sin medir escenarios reales.

---

# 441. Better

Definir budgets por clase:

```text
local authorization
external authorization
batch authorization
```

---

# 442. Local Fast Path

Debería ser extremadamente ligero después de warmup.

---

# 443. External Path

Dominado por network latency.

---

# 444. Batch Path

Medir por:

```text
subjects/second
queries
external calls
```

---

# 445. Runtime Profiling Hooks

Cada stage podrá marcar timing solo cuando metrics/profiler lo requiera.

---

# 446. Null Instrumentation

Cuando apagado:

```text
NullProfiler
```

evita branches complejos.

---

# 447. Compiler Could Specialize

En build de producción minimal, container puede enlazar directamente Null implementations.

---

# 448. Code Path Simplicity

No introducir decenas de:

```php
if ($debug) { ... }
```

dentro del loop principal si se puede abstraer.

---

# 449. But Virtual Call Overhead

También debe benchmarkearse.

---

# 450. Balanced Design

Priorizar legibilidad hasta que profiling demuestre hotspot real.

---

# 451. Security Performance Regression

Un optimization change deberá ejecutar:

```text
security differential suite
```

antes de aceptarse.

---

# 452. Example

Cambiar:

```text
Policy dispatcher
```

por generated dispatch table debe verificar:

```text
same GRANT/DENY/FAILURE
same ordering
same reason codes
same mandatory evaluator behavior
```

---

# 453. Optimization Feature Flags

Nuevos fast paths podrán habilitarse experimentalmente.

---

# 454. Safe Fallback

Si fast path no aplica:

```text
normal compiled execution
```

no bypass.

---

# 455. Versioned Optimization

Artifact debe indicar:

```text
runtime_format_version
```

---

# 456. Old Worker

No debe cargar un format que no entiende.

---

# 457. Compatibility Failure

```text
boot failure
```

---

# 458. Runtime Manifest Validation Cost

Debe realizarse una vez por worker/startup, no por authorization check.

---

# 459. Hash Validation

Igualmente.

---

# 460. Hot Request Path

Ideal:

```text
Route/Ability ID already known
        ↓
Array lookup
        ↓
Bind runtime values
        ↓
Few evaluator calls
        ↓
Streaming DecisionManager
```

---

# 461. Programmatic Dynamic Path

```text
String canonical lookup
        ↓
Plan lookup
        ↓
same execution engine
```

---

# 462. No Duplicate Runtime Engines

Compiled y dynamic modes deberán converger en el mismo:

```text
AuthorizationExecutionEngine
```

después de resolución estructural.

---

# 463. Why

Evita semantic drift.

---

# 464. Dynamic Resolver

Produce:

```text
PlanTemplate
```

---

# 465. Compiled Resolver

Carga:

```text
PlanTemplate
```

---

# 466. Same Executor

Ambos pasan a:

```text
AuthorizationExecutionEngine
```

---

# 467. Testing Advantage

Hace differential testing más sencillo.

---

# 468. Compilation Invariants

### Invariante 1

Todo dato estructural resoluble antes del request deberá compilarse cuando compiled mode esté activo.

### Invariante 2

Compiled metadata no contiene Principal, Tenant ni Subject runtime.

### Invariante 3

Mismo input produce manifest determinista.

### Invariante 4

Errors de seguridad bloquean compilation.

### Invariante 5

Optimization passes se validan nuevamente antes de escribir artifacts.

---

# 469. Registry Invariants

### Invariante 1

Registries quedan sellados después de bootstrap.

### Invariante 2

No se modifican durante requests.

### Invariante 3

Registry version forma parte de cache/fingerprint relevante.

### Invariante 4

Plugins registran authorization metadata antes del seal.

---

# 470. Plan Invariants

### Invariante 1

PlanTemplates son inmutables.

### Invariante 2

Evaluator ordering se calcula antes del hot path.

### Invariante 3

Required y NonBypassable evaluators no se eliminan por optimizer.

### Invariante 4

Plan deduplication preserva requirements y provenance.

### Invariante 5

Dynamic y compiled plans son semánticamente equivalentes.

---

# 471. Runtime Invariants

### Invariante 1

No Reflection en compiled hot path.

### Invariante 2

No filesystem discovery durante authorization normal.

### Invariante 3

Shared runtime metadata es inmutable.

### Invariante 4

Dynamic state vive en execution/request scope.

### Invariante 5

Worker reuse no mezcla authorization state entre requests.

---

# 472. Performance Invariants

### Invariante 1

Cache no oculta N+1 estructural.

### Invariante 2

Global requirements pueden memoizarse dentro del request.

### Invariante 3

Batch authorization reutiliza trabajo común.

### Invariante 4

Short-circuit no salta evaluators mandatory.

### Invariante 5

External timeouts respetan el authorization deadline.

---

# 473. Security Invariants

### Invariante 1

Ninguna optimización produce fail-open.

### Invariante 2

Super-admin fast paths no bypassan seguridad crítica.

### Invariante 3

Corrupt compiled artifacts nunca deshabilitan autorización silenciosamente.

### Invariante 4

Compiled mode inválido falla durante boot/deployment.

### Invariante 5

Performance nunca tiene precedencia sobre authorization correctness.

---

# 474. FrankenPHP Invariants

### Invariante 1

Manifest, Plans y stateless evaluators pueden compartirse entre requests.

### Invariante 2

AuthorizationSession no se comparte.

### Invariante 3

Policy instances shared no contienen request state.

### Invariante 4

Toda request limpia referencias dinámicas.

### Invariante 5

Graceful worker reload acompaña cambios estructurales de autorización.

---

# 475. Arquitectura final de compilación

```text
                     APPLICATION SOURCE
                            │
                            ↓
                Authorization Discovery
                            │
          ┌─────────────────┼──────────────────┐
          ↓                 ↓                  ↓
      Policies            Gates            Attributes
          │                 │                  │
          └─────────────────┼──────────────────┘
                            ↓
                     Normalization
                            │
                            ↓
                       Validation
                            │
                            ↓
                   Dependency Analysis
                            │
                            ↓
                    Metadata Merging
                            │
                            ↓
                     Plan Creation
                            │
                            ↓
                     Optimization
                            │
                            ↓
                  Security Revalidation
                            │
                            ↓
                    Registry Sealing
                            │
                            ↓
              CompiledAuthorizationManifest
                            │
                            ↓
                  PHP/Opcache Artifacts
```

---

# 476. Arquitectura final de runtime

```text
                       REQUEST
                          │
                          ↓
               Route / Ability Resolution
                          │
                          ↓
                Compiled Target Lookup
                          │
                          ↓
               AuthorizationPlanTemplate
                          │
            ┌─────────────┼──────────────┐
            ↓             ↓              ↓
       Principal       Tenant         Subject
            │             │              │
            └─────────────┼──────────────┘
                          ↓
              AuthorizationExecution
                          │
                          ↓
                PRE-RESOLUTION PHASE
                          │
                          ↓
                    Short Circuit?
                    ┌─────┴─────┐
                    ↓           ↓
                   YES          NO
                    │           │
                    │           ↓
                    │     Subject Resolution
                    │           │
                    │           ↓
                    │     RESOURCE PHASE
                    │           │
                    └───────────┤
                                ↓
                         DecisionManager
                                │
                                ↓
                       GRANT / DENY / FAILURE
```

---

# 477. Arquitectura FrankenPHP

```text
FRANKENPHP WORKER
│
├── CompiledAuthorizationManifest      shared
├── AbilityRegistry                    shared
├── PolicyRegistry                     shared
├── GateRegistry                       shared
├── StrategyRegistry                   shared
├── AuthorizationPlanTemplates         shared
├── Stateless Policies/Evaluators      shared
│
└── REQUEST N
    │
    ├── PrincipalContext               local
    ├── TenantContext                  local
    ├── AuthorizationSession           local
    ├── Memoization                    local
    ├── Subject bindings               local
    ├── Trace                          local
    └── Failure context                local
```

Al terminar:

```text
REQUEST N state
→ destroyed/reset
```

mientras la metadata compilada permanece disponible para `REQUEST N+1`.

---

# 478. Ejemplo completo

Código:

```php
#[RequiresPermission('invoice.update')]
#[Authorize(
    'update',
    subject: 'invoice'
)]
public function update(
    Invoice $invoice,
    UpdateInvoiceData $data,
): Response {
    // ...
}
```

---

# 479. Compile Time

El compiler descubre:

```text
Ability:
update

Permission:
invoice.update

Subject:
Controller argument #0

Subject Type:
Invoice

Requirements:
PermissionEvaluator
TenantIsolationPolicy
InvoicePolicy

Strategy:
deny_overrides
```

---

# 480. Dependency Analysis

```text
PermissionEvaluator
requires:
Principal + Tenant

TenantIsolationPolicy
requires:
Tenant + Subject

InvoicePolicy
requires:
Principal + Subject + Context
```

---

# 481. Compiled Phases

```text
PRE_RESOLUTION

PermissionEvaluator
```

```text
RESOURCE

TenantIsolationPolicy
InvoicePolicy
```

---

# 482. Controller Dependency Plan

```text
Authorization Required Arguments:
#0 Invoice

Post-Authorization Arguments:
#1 UpdateInvoiceData
```

---

# 483. Runtime

Request llega.

Route compiler ya conoce:

```text
AuthorizationTarget ID=84
```

---

# 484. Lookup

```text
Target 84
→ PlanTemplate 31
```

O(1).

---

# 485. PreResolution

```text
PermissionEvaluator
```

consulta:

```text
request permission memo
```

---

# 486. Missing Permission

Resultado:

```text
DENY
```

El sistema no resuelve:

```text
Invoice
UpdateInvoiceData
Controller invocation
```

si ni siquiera se requiere binding para esa fase.

---

# 487. Permission GRANT

Entonces:

```text
resolve Invoice only
```

---

# 488. Resource Phase

```text
TenantIsolationPolicy
→ GRANT

InvoicePolicy
→ GRANT
```

---

# 489. Controller Argument Completion

Solo entonces:

```text
resolve UpdateInvoiceData
```

---

# 490. Final Result

```text
GRANT
→ Controller invoked
```

---

# 491. Performance Benefits

La misma arquitectura evitó:

```text
runtime Reflection
metadata merging
Policy discovery
plan sorting
unnecessary DTO construction
unnecessary resource work after early DENY
```

---

# 492. Batch Example

500 invoices:

```text
invoice.view
```

El engine puede resolver una sola vez:

```text
PrincipalState
TenantMembership
Permission invoice.view
```

y después ejecutar resource-specific checks.

---

# 493. Result

En lugar de:

```text
500 permission queries
```

podría producir:

```text
1 permission resolution
+
batch relationship lookup
+
resource-specific Policy evaluation
```

---

# 494. Compilation Tooling

Comandos propuestos:

```text
volt authorization:compile
volt authorization:warm
volt authorization:optimize-report
volt authorization:performance
volt authorization:manifest
```

---

# 495. `authorization:manifest`

En development podrá mostrar:

```text
Schema Version
Registry Version
Build ID
Abilities
Policies
Targets
Plan Templates
Compiled Mode
```

---

# 496. `authorization:optimize-report`

Ejemplo:

```text
Authorization Optimization Report

Runtime mode:
compiled

Abilities:
184 / 184 compiled

Policy invocation maps:
42 / 42 compiled

Controller targets:
327 / 327 compiled

Plan templates:
219

Reflection hot-path:
0

Transient policy services:
1

Potential N+1 evaluators:
2
```

---

# 497. `authorization:performance`

Para una Ability:

```text
Ability:
invoice.update

Plan:
compiled

PreResolution evaluators:
2

Resource evaluators:
3

External calls:
0

Request memoizable:
yes

Cross-request decision cache:
no

Reflection:
none
```

---

# 498. Filosofía del sistema

La filosofía definitiva será:

```text
Discover once.

Validate early.

Compile structure.

Seal registries.

Precompute plans.

Bind only runtime state.

Resolve only arguments that authorization actually needs.

Short-circuit when semantics permit.

Memoize repeated facts within the execution.

Batch expensive repeated checks.

Share immutable metadata across FrankenPHP requests.

Never share mutable authorization state.

Use Opcache instead of rebuilding structure.

And never trade security semantics for a faster branch.
```

---

# 499. Resultado esperado

El `Authorization Compilation, Optimization and Runtime Performance System` permitirá que VoltStack mantenga una arquitectura de autorización empresarial muy completa sin convertir cada:

```php
$user->can(...)
```

o:

```php
#[Authorize(...)]
```

en una cadena costosa de Reflection, discovery y resolución dinámica.

El modelo final será:

```text
SOURCE
   ↓
DISCOVERY
   ↓
NORMALIZATION
   ↓
VALIDATION
   ↓
COMPILATION
   ↓
SEALED REGISTRIES
   ↓
PLAN TEMPLATES
   ↓
OPCACHE / FRANKENPHP WORKER
   ↓
RUNTIME BINDING
   ↓
MINIMAL EVALUATION
   ↓
DECISION
```

La separación fundamental será:

```text
Static Authorization Structure
        ↓
compiled once and shared

Dynamic Authorization State
        ↓
bound per execution and never shared
```

El principio definitivo será:

```text
VoltStack should spend CPU discovering authorization
when the application is built,
not every time a protected method is called.

Runtime authorization should focus on the only questions
that truly remain dynamic:

Who is acting?

In which security and tenant context?

On which Subject?

And do the already-compiled rules grant the operation?
```

Con esta arquitectura, VoltStack podrá aprovechar de manera nativa **Opcache, caches compilados y FrankenPHP workers persistentes**, manteniendo al mismo tiempo fail-closed behavior, aislamiento multi-tenant, consistencia de Policies y equivalencia exacta entre los modos dinámico y compilado.