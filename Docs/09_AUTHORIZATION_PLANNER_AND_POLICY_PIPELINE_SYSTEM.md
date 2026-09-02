# VoltStack Authorization System — Authorization Planner and Policy Pipeline System

## 1. Propósito

Este documento define el subsistema encargado de **construir, ordenar, optimizar y ejecutar el plan de autorización** dentro de VoltStack.

El `AuthorizationPlanner` será responsable de transformar una solicitud normalizada:

```text
AuthorizationRequest
```

en una representación ejecutable:

```text
AuthorizationPlan
```

que describa:

- qué evaluadores deben participar;
- en qué orden;
- en qué fase;
- con qué prioridad;
- bajo qué estrategia;
- cuáles son obligatorios;
- cuáles pueden producir short-circuit;
- cuáles requieren contexto adicional;
- cuáles pueden omitirse;
- cuáles pueden ejecutarse antes de resolver el Subject;
- cuáles requieren un Subject ya resuelto.

El Planner será, por tanto, el componente que conecta:

```text
AuthorizationRequest
        ↓
Policy/Gate Resolution
        ↓
Decision Strategy
        ↓
Evaluator Execution
```

sin ejecutar por sí mismo la lógica de las Policies.

---

# 2. Principio arquitectónico

VoltStack separará explícitamente:

```text
Resolution
Planning
Execution
Decision
```

La secuencia será:

```text
AuthorizationRequest
        ↓
Resolvers
        ↓
AuthorizationPlanner
        ↓
AuthorizationPlan
        ↓
AuthorizationExecutor
        ↓
DecisionManager
```

Esta separación permitirá:

- compilación;
- optimización;
- debugging;
- profiling;
- determinismo;
- pre-autorización;
- short-circuit;
- integración con Controllers;
- integración con Routing;
- ejecución eficiente bajo FrankenPHP.

---

# 3. Responsabilidad del AuthorizationPlanner

El Planner responderá:

```text
¿Qué evaluadores deben ejecutarse
para esta solicitud y en qué orden?
```

No responderá:

```text
¿Está autorizado?
```

Eso corresponde al `DecisionManager`.

---

# 4. Responsabilidades principales

El Planner deberá:

1. recibir `AuthorizationRequest`;
2. determinar la fase actual;
3. resolver evaluadores globales;
4. resolver Gates;
5. resolver Policies de Subject;
6. resolver evaluadores multi-tenant;
7. resolver evaluadores de seguridad;
8. resolver evaluadores contextuales;
9. filtrar evaluadores no aplicables;
10. resolver prioridades;
11. resolver obligatoriedad;
12. determinar estrategia;
13. determinar reglas de short-circuit;
14. producir `AuthorizationPlan`;
15. permitir optimización mediante metadata compilada.

---

# 5. Lo que no debe hacer

El Planner no deberá:

```text
invoke Policy methods
invoke Gate handlers
evaluate permissions
query domain state
decide GRANT or DENY
mutate Principal
mutate Subject
perform model binding
```

---

# 6. Contrato conceptual

```php
interface AuthorizationPlannerInterface
{
    public function plan(
        AuthorizationRequest $request,
    ): AuthorizationPlan;
}
```

En escenarios por fases podrá existir:

```php
public function planPhase(
    AuthorizationRequest $request,
    AuthorizationPhase $phase,
): AuthorizationPlan;
```

---

# 7. AuthorizationPlan

Representará el plan completo de evaluación.

Conceptualmente:

```php
final readonly class AuthorizationPlan
{
    public function __construct(
        public AuthorizationRequest $request,
        public AuthorizationPhase $phase,
        public array $evaluators,
        public DecisionStrategyDescriptor $strategy,
        public DefaultDecisionPolicy $defaultDecision,
        public AuthorizationPlanMetadata $metadata,
    ) {}
}
```

---

# 8. Plan como objeto inmutable

Una vez construido:

```text
AuthorizationPlan
```

deberá ser inmutable.

Esto garantiza:

- trazabilidad;
- ejecución determinista;
- seguridad;
- facilidad de testing;
- compatibilidad con concurrencia.

---

# 9. Plan ≠ ejecución

Debe distinguirse:

```text
AuthorizationPlan
```

de:

```text
AuthorizationExecution
```

El primero describe:

```text
qué debería ejecutarse
```

El segundo registra:

```text
qué se ejecutó y qué ocurrió
```

---

# 10. Pipeline conceptual

El pipeline general podrá ser:

```text
Global Policies
      ↓
Principal Security
      ↓
Tenant Policies
      ↓
Context Policies
      ↓
Gates
      ↓
Resource Policies
      ↓
Compliance / Domain Policies
      ↓
Decision Manager
```

Pero este orden deberá derivarse de prioridad y fase, no hardcodearse como una secuencia rígida.

---

# 11. AuthorizationPhase

VoltStack deberá modelar explícitamente las fases.

Propuesta inicial:

```php
enum AuthorizationPhase: string
{
    case PreResolution = 'pre_resolution';
    case Resource = 'resource';
    case PostResolution = 'post_resolution';
}
```

---

# 12. PreResolution

Se ejecuta antes de resolver recursos costosos.

Ejemplo:

```text
¿Puede este Principal acceder al módulo administrativo?
```

No requiere cargar:

```text
Invoice#928
```

---

# 13. Resource Phase

Se ejecuta cuando el Subject real ya está disponible.

Ejemplo:

```text
¿Puede User#42 actualizar Invoice#928?
```

---

# 14. PostResolution

Podrá utilizarse para evaluadores que dependan de información construida tras la resolución completa del contexto.

No deberá convertirse en una fase genérica para lógica tardía arbitraria.

---

# 15. Fases extensibles

En el futuro podrían añadirse:

```text
PreAuthentication
PostAuthentication
PostDecision
```

pero solo si aparecen necesidades reales.

V1 deberá mantenerse simple.

---

# 16. Planner por fase

Ejemplo Controller:

```text
Route Match
    ↓
Controller Resolution
    ↓
Plan PRE_RESOLUTION
    ↓
execute
    ↓
Argument Resolution
    ↓
Plan RESOURCE
    ↓
execute
    ↓
Controller Invocation
```

---

# 17. Beneficio principal

Si la pre-autorización falla:

```text
DENY
```

VoltStack evita:

```text
database lookup
model hydration
extra middleware work
controller argument resolution
```

---

# 18. Controller-wide authorization

Ejemplo:

```php
#[Authorize(
    'admin.access',
    phase: AuthorizationPhase::PreResolution
)]
final class AdminInvoiceController
{
}
```

Esto puede ejecutarse antes de resolver argumentos.

---

# 19. Resource authorization

Ejemplo:

```php
#[Authorize(
    'update',
    subject: 'invoice',
    phase: AuthorizationPhase::Resource
)]
public function update(Invoice $invoice)
{
}
```

---

# 20. Multiple Authorization Requirements

Una acción podrá declarar múltiples requisitos:

```php
#[Authorize('admin.access', phase: AuthorizationPhase::PreResolution)]
#[Authorize('update', subject: 'invoice')]
public function update(Invoice $invoice)
{
}
```

Esto genera dos planes separados o un plan compuesto por fases.

---

# 21. Composite Authorization Plan

Podrá existir:

```text
ControllerAuthorizationPlan
```

conteniendo:

```text
PreResolution Plan
Resource Plan
```

pero internamente cada fase seguirá usando `AuthorizationPlan`.

---

# 22. Planner Inputs

Además de `AuthorizationRequest`, el Planner podrá utilizar:

```text
PolicyRegistry
GateRegistry
AbilityRegistry
EvaluatorRegistry
DecisionStrategyRegistry
Compiled Authorization Metadata
Authorization Configuration
```

---

# 23. Resolver orchestration

El Planner podrá invocar:

```text
GlobalEvaluatorResolver
GateResolver
PolicyResolver
TenantEvaluatorResolver
SecurityEvaluatorResolver
```

o un resolver unificado.

---

# 24. AuthorizationEvaluatorResolver

Arquitectura preferida:

```php
interface AuthorizationEvaluatorResolverInterface
{
    public function resolve(
        AuthorizationRequest $request,
        AuthorizationPhase $phase,
    ): iterable;
}
```

Múltiples resolvers podrán contribuir.

---

# 25. Resolver chain

Ejemplo:

```text
GlobalEvaluatorResolver
        ↓
SecurityEvaluatorResolver
        ↓
TenantEvaluatorResolver
        ↓
GateResolver
        ↓
PolicyResolver
        ↓
CustomEvaluatorResolver
```

Cada resolver devuelve descriptors.

---

# 26. EvaluatorDescriptor

Todos deberán normalizarse a una representación común.

```php
final readonly class EvaluatorDescriptor
{
    public function __construct(
        public string $id,
        public EvaluatorType $type,
        public int $priority,
        public AuthorizationPhase $phase,
        public EvaluatorRequirement $requirement,
        public mixed $invocation,
        public bool $terminalOnGrant = false,
        public bool $terminalOnDeny = false,
    ) {}
}
```

---

# 27. Invocation metadata

`invocation` podrá contener:

```text
PolicyInvocationDescriptor
GateInvocationDescriptor
PermissionInvocationDescriptor
ExternalEvaluatorDescriptor
```

El Executor utilizará el adapter adecuado.

---

# 28. EvaluatorType

Ejemplo:

```php
enum EvaluatorType: string
{
    case Policy = 'policy';
    case Gate = 'gate';
    case Security = 'security';
    case Tenant = 'tenant';
    case Permission = 'permission';
    case Role = 'role';
    case External = 'external';
    case Custom = 'custom';
}
```

---

# 29. Planner no debe depender demasiado del tipo

El tipo será útil para:

- tracing;
- grouping;
- tooling;
- defaults.

Pero el comportamiento principal deberá depender de:

```text
priority
requirement
phase
strategy
```

---

# 30. Global Evaluators

Podrán aplicar a cualquier solicitud.

Ejemplos:

```text
SuspendedPrincipalPolicy
PlatformLockdownPolicy
EmergencySecurityPolicy
```

---

# 31. Principal Security Evaluators

Ejemplos:

```text
MfaRequirementPolicy
TrustedDevicePolicy
SessionStrengthPolicy
CompromisedAccountPolicy
```

---

# 32. Tenant Evaluators

Ejemplos:

```text
TenantIsolationPolicy
TenantMembershipPolicy
TenantStatusPolicy
```

---

# 33. Resource Evaluators

Ejemplos:

```text
InvoicePolicy
PostPolicy
ProjectPolicy
```

---

# 34. Compliance Evaluators

Ejemplos:

```text
FinancialCompliancePolicy
ExportRestrictionPolicy
DataClassificationPolicy
```

---

# 35. Gate Evaluators

Ejemplo:

```text
admin.access
system.deploy
user.impersonate
```

---

# 36. Resolver result normalization

Cada resolver deberá devolver:

```text
EvaluatorDescriptor[]
```

No objetos ejecutables arbitrarios.

---

# 37. Deduplicación

El Planner deberá evitar que la misma Policy aparezca dos veces por distintos caminos.

Ejemplo:

```text
InvoicePolicy
resolved by exact class
+
resolved by explicit metadata
```

si representa el mismo descriptor.

---

# 38. Descriptor Identity

La deduplicación deberá utilizar una identidad canónica:

```text
evaluator id
+
invocation identity
```

no igualdad de objeto.

---

# 39. Duplicate but intentional evaluators

Dos descriptors de la misma clase podrán coexistir si representan configuraciones diferentes.

Ejemplo:

```text
CompliancePolicy
ability=approve
region=MX

CompliancePolicy
ability=approve
region=US
```

si la arquitectura lo requiere.

---

# 40. Applicability Filtering

Después de resolver candidatos:

```text
Candidate Evaluators
        ↓
Applicability Filter
```

Se analizará:

- Ability;
- Subject type;
- Subject class;
- phase;
- channel;
- context presence;
- tenant presence;
- Principal type;
- required feature.

---

# 41. No business logic en filtering

El Planner no debe preguntar:

```text
$user->isAdmin()
$invoice->amount > 100000
```

Eso pertenece a evaluadores.

---

# 42. Structural filtering only

Correcto:

```text
Policy handles ability approve
Policy requires TenantContext
Policy applies to Invoice
```

---

# 43. Required Context

Si un evaluator requiere:

```text
SecurityContext
```

y no existe, puede ocurrir:

```text
configuration failure
```

si es obligatorio.

---

# 44. Optional Applicability

Si un evaluator solo aplica en:

```text
channel=web
```

y la request es CLI:

```text
skip
```

sin error.

---

# 45. ContextRequirementType

Podrá modelarse:

```php
enum ContextRequirementType
{
    case Required;
    case OptionalApplicability;
}
```

---

# 46. Principal-type applicability

Ejemplo:

```text
HumanMfaPolicy
```

puede aplicar solo a:

```text
UserPrincipal
```

y no a:

```text
SystemPrincipal
```

---

# 47. Anonymous applicability

Una Policy puede declarar:

```text
supportsAnonymous=false
```

Entonces:

```text
AnonymousPrincipal
```

puede causar exclusión o denegación según requisito.

---

# 48. Policy vs planner denial

El Planner deberá evitar generar decisiones semánticas.

No debe decir:

```text
anonymous unsupported
→ DENY
```

por sí mismo salvo que se trate de una regla estructural de seguridad explícita.

Preferido:

```text
AuthenticatedPrincipalEvaluator
```

para ese tipo de decisión.

---

# 49. Planner validation failures

El Planner sí puede fallar ante:

```text
missing mandatory context
invalid strategy
corrupt descriptor
unresolvable subject reference
```

porque son fallos de integración/configuración.

---

# 50. Priority Resolution

Una vez filtrados, los evaluadores se ordenarán.

Convención:

```text
higher number
=
earlier execution
```

---

# 51. Priority sources

La prioridad podrá venir de:

```text
descriptor default
attribute
registration metadata
framework default
```

---

# 52. Priority precedence

Recomendación:

```text
explicit registration priority
        ↓
attribute priority
        ↓
policy/gate default
        ↓
framework category default
```

---

# 53. Category defaults

Ejemplo conceptual:

```text
Critical Security     10000
Global Security        9000
Tenant Isolation       8000
Authentication State   7000
Permissions            6000
Resource Policy        5000
Compliance             4000
Contextual Optional    3000
```

Estos valores deberán ser configurables y documentados.

---

# 54. No semantic dependency on magic numbers

Los números son implementación.

La arquitectura debe permitir constantes:

```php
AuthorizationPriority::CriticalSecurity
AuthorizationPriority::Tenant
AuthorizationPriority::Resource
```

---

# 55. AuthorizationPriority

Podrá existir:

```php
final class AuthorizationPriority
{
    public const CRITICAL_SECURITY = 10000;
    public const SECURITY = 9000;
    public const TENANT = 8000;
    public const AUTHENTICATION = 7000;
    public const PERMISSION = 6000;
    public const RESOURCE = 5000;
    public const COMPLIANCE = 4000;
    public const CONTEXT = 3000;
}
```

---

# 56. Stable Ordering

Para prioridades iguales:

```text
source precedence
        ↓
compiled registration order
        ↓
stable descriptor id
```

---

# 57. Deterministic Plan

Misma metadata + misma request estructural:

```text
same AuthorizationPlan
```

---

# 58. Requirement Resolution

Cada evaluator será:

```text
Optional
Required
Critical
```

---

# 59. Optional

Puede abstenerse o fallar bajo una política configurada sin necesariamente invalidar todo el plan.

---

# 60. Required

Se espera que participe correctamente.

Un fallo deberá provocar fail-closed.

---

# 61. Critical

Representa una frontera de seguridad.

Ejemplo:

```text
TenantIsolationPolicy
```

Semántica típica:

```text
failure → DENY
DENY → short-circuit
```

---

# 62. Mandatory policy presence

El Planner podrá validar:

```text
Critical evaluator configured
but missing from resolution
```

como error.

---

# 63. Strategy Resolution

Después de resolver evaluadores:

```text
AuthorizationPlanner
    ↓
DecisionStrategyResolver
```

---

# 64. Strategy precedence

Como definido previamente:

```text
Request/metadata explicit
        ↓
Route/Controller metadata
        ↓
Ability descriptor
        ↓
Subsystem default
        ↓
Global default
```

---

# 65. Strategy Compatibility

El Planner deberá validar si la estrategia es compatible con los evaluadores.

Ejemplo:

```text
FirstApplicable
+
unordered parallel evaluators
```

sería inválido.

---

# 66. StrategyCapabilities

Se podrán consultar capacidades compiladas:

```text
orderSensitive
supportsShortCircuit
parallelSafe
```

---

# 67. Short-Circuit Planning

El Planner podrá marcar qué evaluadores pueden permitir early termination.

Ejemplo:

```text
TenantIsolationPolicy
terminalOnDeny=true
```

---

# 68. Strategy authority

Sin embargo, el Planner no garantiza que termine.

La estrategia decide si el short-circuit es semánticamente válido.

---

# 69. Critical Deny

Ejemplo:

```text
TenantIsolationPolicy
priority=8000
critical
terminalOnDeny
```

con `DenyOverrides`.

Si DENY:

```text
STOP
```

---

# 70. Super-admin override

Ejemplo:

```text
SuperAdminEvaluator
priority=10000
terminalOnGrant
```

solo deberá ser efectivo si la estrategia permite allow override.

---

# 71. Cost-aware Planning

En versiones avanzadas podrá existir:

```text
estimatedCost
```

por evaluator.

Ejemplo:

```text
local boolean check = low
DB relation lookup = medium
remote PDP = high
```

---

# 72. Reordering limits

Nunca reordenar por costo si cambia semántica.

La prioridad y estrategia tienen precedencia.

---

# 73. Cheap critical checks first

Cuando semánticamente equivalente:

```text
SuspendedPrincipalPolicy
```

debe ejecutarse antes de una Policy costosa.

---

# 74. Precompiled Plan Templates

Para Controllers y Routes estáticos podrá compilarse un:

```text
AuthorizationPlanTemplate
```

---

# 75. PlanTemplate

Podrá contener:

```text
known Ability
known Controller
known SubjectReference
known Global Evaluators
known Resource Policy Class
known Strategy
known Phase
```

---

# 76. Runtime binding

El template no contiene:

```text
current Principal
current Subject instance
current Tenant
```

Estos se enlazan en runtime.

---

# 77. Controller Plan Template

Ejemplo:

```text
InvoiceController::update

Pre:
admin.access
security.mfa

Resource:
update
subject argument invoice
strategy deny_overrides
```

---

# 78. Benefit

En producción:

```text
Controller metadata lookup
        ↓
compiled PlanTemplate
        ↓
bind runtime values
```

sin re-resolver toda la estructura.

---

# 79. Dynamic Authorization Requests

Llamadas como:

```php
Authorization::check(
    $ability,
    $subject
);
```

seguirán usando planning dinámico.

---

# 80. Dynamic no significa reflection-heavy

El Planner seguirá utilizando:

```text
compiled registries
indexed resolution
```

---

# 81. Pipeline stages

Podrá existir una representación más explícita:

```text
AuthorizationPipeline
├── PreSecurity
├── Tenant
├── Permission
├── Resource
└── Compliance
```

Pero estas stages deberán verse como grupos lógicos, no motores independientes.

---

# 82. PolicyPipeline

El término `PolicyPipeline` representará la secuencia ejecutable de evaluadores.

No solo Policies en sentido estricto.

---

# 83. Naming

Podría preferirse:

```text
AuthorizationPipeline
```

como nombre interno más exacto.

`PolicyPipeline` podrá mantenerse en documentación por familiaridad.

---

# 84. AuthorizationPipeline

Conceptualmente:

```php
final readonly class AuthorizationPipeline
{
    public function __construct(
        public array $evaluators,
    ) {}
}
```

---

# 85. Pipeline item

Cada elemento será:

```text
EvaluatorInvocation
```

no un objeto Policy directamente.

---

# 86. EvaluatorInvocation

```php
final readonly class EvaluatorInvocation
{
    public function __construct(
        public EvaluatorDescriptor $descriptor,
        public mixed $invocationMetadata,
    ) {}
}
```

---

# 87. Pipeline construction

```text
Resolved Evaluators
        ↓
Normalize descriptors
        ↓
Deduplicate
        ↓
Apply phase filter
        ↓
Apply applicability filter
        ↓
Resolve requirements
        ↓
Resolve priorities
        ↓
Stable sort
        ↓
AuthorizationPipeline
```

---

# 88. Plan Metadata

El plan podrá incluir:

```text
source
compiled template id
resolution trace id
strategy source
phase
build duration
```

solo cuando observabilidad esté habilitada.

---

# 89. AuthorizationPlanMetadata

Conceptualmente:

```php
final readonly class AuthorizationPlanMetadata
{
    public function __construct(
        public ?string $templateId = null,
        public ?string $strategySource = null,
        public array $tags = [],
    ) {}
}
```

---

# 90. Plan Hash

Podrá existir:

```text
AuthorizationPlanFingerprint
```

para memoizar templates o tracing.

No incluir datos sensibles.

---

# 91. Plan Fingerprint inputs

Ejemplo:

```text
Ability
Subject class/type
Phase
Context shape
Registry version
Strategy ID
```

---

# 92. No Principal identity

Un plan estructural normalmente no necesita:

```text
User#42
```

porque los evaluadores aplicables son los mismos para otros usuarios del mismo contexto estructural.

---

# 93. No Subject instance ID

Igualmente:

```text
Invoice#928
```

no suele alterar qué evaluadores existen.

---

# 94. Context Shape

El Planner puede necesitar:

```text
channel=web
hasTenant=true
hasSecurity=true
```

pero no necesariamente todos los valores.

---

# 95. AuthorizationContextShape

Podrá modelarse:

```php
final readonly class AuthorizationContextShape
{
    public function __construct(
        public string $channel,
        public bool $hasTenant,
        public bool $hasSecurity,
        public bool $hasHttp,
    ) {}
}
```

---

# 96. Plan Cache

Puede cachearse:

```text
structural AuthorizationPlan
```

si no contiene runtime state.

---

# 97. Plan Cache vs Decision Cache

```text
Plan Cache
=
what should run
```

```text
Decision Cache
=
what result was produced
```

Son conceptos distintos.

---

# 98. Plan Cache safety

Es mucho más seguro compartir planes estructurales que decisiones.

---

# 99. Plan Cache invalidation

Debe invalidarse al cambiar:

```text
Policy Registry
Gate Registry
Ability metadata
strategy config
authorization metadata schema
```

---

# 100. Plan Cache scope

Podrá ser:

```text
process-wide
```

bajo FrankenPHP si es inmutable.

---

# 101. Request Runtime Binding

El Executor recibirá:

```text
AuthorizationPlan
+
AuthorizationRequest
```

y resolverá instancias.

---

# 102. AuthorizationExecutor

Contrato conceptual:

```php
interface AuthorizationExecutorInterface
{
    public function execute(
        AuthorizationPlan $plan,
    ): AuthorizationExecution;
}
```

o streaming:

```php
public function votes(
    AuthorizationPlan $plan,
): iterable;
```

---

# 103. Streaming Recommendation

Para short-circuit:

```text
AuthorizationExecutor
    ↓
yield DecisionVote
    ↓
DecisionManager
```

es una arquitectura apropiada.

---

# 104. Execution order

El Executor deberá respetar exactamente:

```text
plan.evaluators
```

---

# 105. Executor no reorders

Nunca:

```text
sort again
```

de manera independiente.

---

# 106. Dispatcher Selection

Por cada evaluator:

```text
EvaluatorType
    ↓
DispatcherRegistry
```

Ejemplo:

```text
Policy → PolicyDispatcher
Gate → GateDispatcher
External → ExternalEvaluatorDispatcher
```

---

# 107. AuthorizationEvaluatorDispatcherRegistry

Conceptualmente:

```php
interface EvaluatorDispatcherRegistryInterface
{
    public function dispatcherFor(
        EvaluatorType $type
    ): AuthorizationEvaluatorDispatcherInterface;
}
```

---

# 108. Unified dispatcher contract

```php
interface AuthorizationEvaluatorDispatcherInterface
{
    public function dispatch(
        AuthorizationRequest $request,
        EvaluatorInvocation $invocation,
    ): DecisionResult;
}
```

---

# 109. Execution boundary

Cada invocation deberá ejecutarse dentro de:

```text
EvaluatorExecutionBoundary
```

para normalizar fallos.

---

# 110. Execution Record

El Executor podrá generar:

```text
DecisionVote
```

con:

```text
status
result
evaluator id
priority
requirement
```

---

# 111. Failed evaluator

Si dispatcher lanza una excepción:

```text
DecisionVote
status=FAILED
```

El DecisionManager aplicará failure policy.

---

# 112. Skipped evaluators

Si hay short-circuit:

```text
remaining evaluators
status=SKIPPED
```

solo en tracing detallado.

No es necesario materializarlos en hot path.

---

# 113. Plan Short-Circuit Metadata

El Plan podrá indicar:

```text
shortCircuitEnabled=true
```

según estrategia.

---

# 114. Executor + DecisionManager coordination

Arquitectura posible:

```text
for evaluator in plan:
    vote = execute evaluator
    decisionManager.consume(vote)

    if decisionManager.isFinal():
        break
```

---

# 115. DecisionSession

Podrá existir:

```text
DecisionSession
```

local a la autorización.

Contiene acumulador y estrategia.

---

# 116. DecisionSession no shared

Nunca debe sobrevivir entre requests.

---

# 117. Multiple phases and final decision

Debe distinguirse:

```text
phase authorization result
```

de:

```text
whole operation authorization result
```

---

# 118. Controller example

Pre phase:

```text
admin.access → GRANT
```

Resource phase:

```text
invoice.update → DENY
```

Operación final:

```text
DENY
```

---

# 119. Phase combination

Por defecto, todas las fases requeridas deberán conceder acceso.

Conceptualmente:

```text
PreResolution GRANT
AND
Resource GRANT
AND
PostResolution GRANT
```

---

# 120. Phase result strategy

La combinación de fases no deberá reutilizar arbitrariamente `Consensus`.

Será normalmente:

```text
all required phases must grant
```

---

# 121. AuthorizationLifecyclePlan

Futuro:

```php
final readonly class AuthorizationLifecyclePlan
{
    public function __construct(
        public array $phasePlans,
    ) {}
}
```

útil para Controller integration.

---

# 122. Route Authorization

Una Route podrá producir:

```text
PreResolution Gate
Resource Policy
```

igual que Controller metadata.

---

# 123. Route-bound Subject

Ejemplo:

```php
Route::delete('/users/{user}', ...)
    ->can('delete', 'user');
```

Plan:

```text
Route matched
  ↓
possible pre checks
  ↓
bind user
  ↓
resource plan
```

---

# 124. Preventing unnecessary binding

Si la ruta además exige:

```text
admin.access
```

pre-resolution puede denegar antes de consultar DB.

---

# 125. Controller inheritance metadata

El Planner deberá combinar:

```text
parent Controller authorization
class authorization
method authorization
route authorization
```

según reglas definidas.

---

# 126. Metadata precedence

Recomendación:

```text
Route Explicit
Controller Method
Controller Class
Inherited Controller
Framework Defaults
```

Pero combinación no siempre significa override.

---

# 127. Additive authorization

Por defecto múltiples requisitos deberán ser:

```text
additive
```

Ejemplo:

```text
Controller requires admin.access
+
method requires invoice.update
```

ambos deben pasar.

---

# 128. Replacement metadata

Podrá existir:

```text
replaceParentAuthorization=true
```

para casos excepcionales.

No debe ser default.

---

# 129. Authorization composition modes

Podrán definirse:

```text
Append
Replace
Inherit
Disable
```

para metadata de Controller/Route.

---

# 130. Disable authorization

Deshabilitar una regla heredada deberá requerir intención explícita y tooling visible.

---

# 131. Security warning

Una acción que haga:

```text
Disable inherited security requirement
```

deberá generar warning en linting si la regla era crítica.

---

# 132. Global Policies always-on

Algunas evaluaciones críticas no deberán poder ser deshabilitadas por metadata de aplicación.

Ejemplo:

```text
TenantIsolationPolicy
```

si está configurada como framework critical.

---

# 133. Non-bypassable Evaluators

Podrá existir:

```text
nonBypassable=true
```

para fronteras de seguridad.

---

# 134. Use with caution

Esto debe reservarse para:

```text
tenant isolation
system lockdown
mandatory compliance
```

no para reglas ordinarias.

---

# 135. Break-glass interaction

Incluso super-admin podría no saltarse:

```text
nonBypassable critical policy
```

según estrategia.

---

# 136. Planner validation

Si una estrategia `AllowOverrides` intenta combinarse con un evaluator `nonBypassable` de manera insegura, el Planner/Compiler deberá rechazar configuración.

---

# 137. Policy Pipeline Composition

Ejemplo completo:

```text
Critical Global Security
        ↓
Principal State
        ↓
Tenant Isolation
        ↓
Authentication Assurance
        ↓
Permissions
        ↓
Resource Policy
        ↓
Compliance Policy
        ↓
Contextual Rules
```

---

# 138. No fixed category ordering

Los niveles anteriores son defaults lógicos.

La prioridad explícita será la fuente de verdad.

---

# 139. Pipeline visualization

Tooling podrá mostrar:

```text
Priority  Type       Evaluator
------------------------------------------
10000     Security   PlatformLockdown
9000      Security   SuspendedPrincipal
8000      Tenant     TenantIsolation
7000      Security   MfaRequirement
6000      Permission InvoicePermission
5000      Policy     InvoicePolicy
4000      Compliance FinancialCompliance
```

---

# 140. Pipeline Debugging

En desarrollo:

```text
Why did this evaluator execute?
```

Debe poder responderse:

```text
exact subject match
global policy
ability match
tenant context present
priority 8000
```

---

# 141. Filtered Evaluators Trace

También:

```text
ExportPolicy
skipped:
ability update not supported
```

---

# 142. Plan compilation diagnostics

Tooling podrá mostrar:

```text
PlanTemplate:
InvoiceController::update

Phase:
resource

Subject:
argument invoice

Strategy:
deny_overrides
```

---

# 143. Planner Errors

Jerarquía conceptual:

```text
AuthorizationPlanningException
├── InvalidAuthorizationPlanException
├── MissingRequiredEvaluatorException
├── InvalidEvaluatorDescriptorException
├── InvalidAuthorizationPhaseException
├── AuthorizationStrategyConflictException
├── MissingRequiredContextForPlanException
└── AuthorizationPlanCompilationException
```

---

# 144. Missing Required Evaluator

Ejemplo:

```text
Ability marked as requiring TenantIsolation
but module did not register evaluator
```

Debe detectarse antes de producción cuando sea posible.

---

# 145. Strategy Conflict

Ejemplo:

```text
Critical non-bypassable evaluator
+
AllowOverrides strategy
```

puede considerarse inválido.

---

# 146. Phase mismatch

Ejemplo:

```text
Policy requires resolved Subject
configured for PreResolution
```

deberá fallar en compilación.

---

# 147. Subject Reference Validation

Para:

```php
#[Authorize('update', subject: 'invoice')]
```

el compilador deberá verificar que el Controller method tenga:

```text
invoice
```

resolvable.

---

# 148. Unknown Subject Reference

Debe producir:

```text
InvalidAuthorizationSubjectReferenceException
```

durante compilación cuando sea posible.

---

# 149. PreResolution subject limitations

Una regla pre-resolution no podrá depender de:

```text
entity instance not loaded yet
```

salvo que use datos ya disponibles en Route metadata.

---

# 150. Route Parameter Values

Puede usarse un ID crudo en pre-resolution solo si una Policy está diseñada explícitamente para eso.

No convertirlo automáticamente en entidad.

---

# 151. Resource Resolution Contract

La integración de Controllers/Routes deberá indicar cuándo un SubjectReference está disponible.

---

# 152. Planner + ControllerArgumentResolver

La relación será:

```text
Authorization Metadata
        ↓
needs subject invoice
        ↓
ControllerArgumentResolver
        ↓
Invoice instance
        ↓
Resource Authorization Plan
```

---

# 153. Circular Dependency Avoidance

AuthorizationPlanner no deberá depender directamente del ControllerArgumentResolver Core.

La integración layer coordinará ambos.

---

# 154. Correct dependency direction

```text
Controller Integration
    ↓
Authorization Contracts
```

No:

```text
Authorization Core
    ↓
Controller System
```

---

# 155. Command Pipeline

Para Commands:

```text
Command resolved
   ↓
Pre authorization
   ↓
argument/input resolution
   ↓
resource authorization if needed
   ↓
execute command
```

---

# 156. Job Pipeline

Para Jobs:

```text
Job instance
   ↓
Principal Reference Resolution
   ↓
Authorization Plan
   ↓
Job execution
```

si el Job requiere autorización.

---

# 157. Component Pipeline

Para Components:

```text
Component request
   ↓
authorization metadata
   ↓
resource/context plan
   ↓
render/action
```

---

# 158. SPA Projection

Cuando se calculen capabilities:

```text
CapabilityProjectionPlanner
```

podrá reutilizar el AuthorizationPlanner para múltiples abilities.

---

# 159. Batch planning

Para 50 recursos del mismo tipo:

```text
Invoice
```

no debería construir 50 planes estructurales idénticos.

---

# 160. Plan reuse

Puede reutilizarse:

```text
Invoice + update + web + tenant
```

como plan estructural.

---

# 161. Batch resource evaluation

Luego se enlaza cada:

```text
Invoice#1
Invoice#2
...
```

al mismo template.

---

# 162. N+1 prevention

El Planner podrá detectar evaluadores batch-aware.

Ejemplo:

```text
InvoicePolicy supports batch
```

y producir un plan batch especial.

No es necesario en V1.

---

# 163. Planner Performance Goals

El hot path común debería ser:

```text
Request already normalized
    ↓
derive structural key
    ↓
plan template/cache lookup
    ↓
bind runtime request
    ↓
execute
```

---

# 164. Cold Path

Solo cuando no existe template:

```text
resolve registries
filter
sort
select strategy
build plan
cache
```

---

# 165. Zero Reflection Hot Path

No:

```text
ReflectionClass
ReflectionMethod
Attribute scanning
```

en producción normal.

---

# 166. Zero Filesystem

No:

```text
filesystem discovery
```

durante planificación runtime.

---

# 167. Memory Efficiency

El Plan deberá evitar duplicar descriptors completos si puede referenciar metadata inmutable compartida.

---

# 168. Descriptor references

Ejemplo:

```text
plan evaluator
    ↓
descriptor id
```

podrá apuntar a Registry metadata.

---

# 169. Immutable Shared Metadata

Ideal para FrankenPHP.

---

# 170. Request-local data

El Plan no deberá almacenar accidentalmente:

```text
mutable request object
mutable tenant manager
global principal
```

fuera del `AuthorizationRequest` asociado.

---

# 171. PlanTemplate vs Plan

La distinción puede ser:

```text
AuthorizationPlanTemplate
=
shared structural metadata
```

```text
AuthorizationPlan
=
template + runtime request binding
```

---

# 172. Recommended Runtime Model

Bajo FrankenPHP:

```text
Worker Boot
    ↓
Load PlanTemplates
    ↓
Request A
    ↓
bind Request A
    ↓
execute
    ↓
discard binding
    ↓
Request B
```

---

# 173. No state leakage

PlanTemplate nunca contiene:

```text
Principal A
Tenant A
Invoice A
```

---

# 174. Planner statelessness

`AuthorizationPlanner` deberá ser:

```text
stateless
```

y seguro como shared service.

---

# 175. Resolver statelessness

Igualmente, resolvers deberán usar registries y contexto recibido.

---

# 176. Plan Cache Service

Puede ser shared porque cachea estructura.

---

# 177. Cache Key Security

No utilizar objetos completos serializados como keys.

Preferir:

```text
ability
subject class
phase
context shape
registry version
```

---

# 178. Cache poisoning prevention

Las claves deben derivarse de valores normalizados y confiables.

---

# 179. Dynamic named subjects

Para Named Subjects:

```text
admin-dashboard
```

el nombre canónico puede formar parte del plan key.

---

# 180. Virtual subject key

Ejemplo:

```text
controller_action:UserController::destroy
```

---

# 181. Plan Cache Eviction

Al actualizar registries:

```text
registry version changes
```

las entradas antiguas dejan de usarse.

No necesita invalidación granular inicialmente.

---

# 182. Versioned Plan Cache

Clave:

```text
authorization_registry_v17
```

puede ser suficiente.

---

# 183. Plan Compilation Command

Futuro:

```text
volt authorization:compile
```

podrá generar:

```text
Policy Registry
Gate Registry
Ability Registry
Controller Authorization Metadata
Authorization Plan Templates
```

---

# 184. Build-time validation

Durante compile deberán detectarse:

```text
missing strategy
invalid priority
invalid subject reference
incompatible phase
missing mandatory context
unknown evaluator
```

---

# 185. Production boot

Debe ser:

```text
load compiled metadata
seal registries
create planner
serve requests
```

---

# 186. Development Mode

Podrá reconstruir plans cuando cambie código.

---

# 187. Dev Plan Trace

Ejemplo:

```text
AuthorizationPlan built in 0.21 ms

Resolved candidates: 8
Filtered: 3
Final evaluators: 5
Strategy: deny_overrides
Cache: miss
```

---

# 188. Production Plan Trace

Con observabilidad mínima:

```text
plan_id
strategy
evaluator_count
```

---

# 189. Tooling

Comandos futuros:

```text
volt authorization:plan
volt authorization:pipeline
volt authorization:explain-plan
volt authorization:compile
```

---

# 190. `authorization:plan`

Ejemplo:

```text
volt authorization:plan \
    App\Domain\Invoice update
```

Salida:

```text
Phase:
resource

Strategy:
deny_overrides

Pipeline:
1. SuspendedPrincipalPolicy
2. TenantIsolationPolicy
3. PermissionVoter
4. InvoicePolicy
5. CompliancePolicy
```

---

# 191. Controller plan introspection

```text
volt authorization:plan \
    App\Controller\InvoiceController::update
```

Podrá mostrar:

```text
PreResolution:
admin.access

Resource:
invoice.update

Subject:
invoice argument
```

---

# 192. Linting

`authorization:lint` podrá advertir:

```text
Expensive resource binding occurs before
a pre-resolution authorization that could reject earlier.
```

---

# 193. Planner optimization hints

También:

```text
CompliancePolicy priority 9000
executes before TenantIsolationPolicy 8000.
Consider reviewing ordering.
```

si metadata sugiere una anomalía.

---

# 194. Security lint

Ejemplo:

```text
Non-bypassable TenantIsolationPolicy
is configured after a terminal allow override.
```

deberá fallar.

---

# 195. Testing Planner

Tests deberán poder inspeccionar planes sin ejecutar Policies.

Ejemplo:

```php
$plan = $planner->plan($request);

expect($plan->evaluators)
    ->toHaveCount(4);
```

---

# 196. Ordering test

```php
expect($plan->evaluators[0]->id)
    ->toBe('tenant_isolation');
```

---

# 197. Strategy test

```php
expect($plan->strategy->id())
    ->toBe('deny_overrides');
```

---

# 198. Phase test

Verificar que una Resource Policy no aparezca en PreResolution.

---

# 199. Deduplication test

Una misma Policy resuelta por dos fuentes debe ejecutarse una vez cuando corresponde.

---

# 200. Required evaluator test

Si falta evaluator crítico:

```text
planning failure
```

---

# 201. Compiled equivalence test

Dynamic plan:

```text
=
```

Compiled PlanTemplate bound at runtime.

---

# 202. Persistent runtime test

Plan cache no deberá transportar Subject de Request A a Request B.

---

# 203. Concurrency test

Dos requests pueden usar el mismo PlanTemplate simultáneamente.

---

# 204. Short-circuit integration test

Debe verificarse:

```text
critical DENY
```

evita ejecutar evaluadores posteriores cuando la estrategia lo permite.

---

# 205. Planning vs Execution failure tests

Distinguir:

```text
Planner couldn't build valid plan
```

de:

```text
Evaluator failed while executing valid plan
```

---

# 206. Planning Failure

Ejemplo:

```text
unknown strategy
```

es:

```text
configuration/runtime planning failure
```

---

# 207. Execution Failure

Ejemplo:

```text
InvoicePolicy throws DB exception
```

es:

```text
evaluator execution failure
```

---

# 208. Security outcome

Ambos deben producir comportamiento fail-closed en la frontera de autorización.

---

# 209. Pipeline Invariants

### Invariante 1

Todo evaluator del Pipeline debe estar representado por metadata validada.

### Invariante 2

El Pipeline tiene orden determinista.

### Invariante 3

El Pipeline no contiene estado mutable global.

### Invariante 4

El Executor no reordena el Pipeline.

### Invariante 5

Short-circuit solo ocurre cuando la estrategia lo permite.

---

# 210. Planner Invariants

### Invariante 1

Planner no ejecuta Policies.

### Invariante 2

Planner no toma decisiones semánticas de negocio.

### Invariante 3

Planner separa resolución estructural de ejecución.

### Invariante 4

Planner selecciona estrategia explícitamente.

### Invariante 5

Planner valida incompatibilidades antes de ejecutar cuando sea posible.

---

# 211. Phase Invariants

### Invariante 1

PreResolution no depende de Subjects aún no resueltos.

### Invariante 2

Resource phase puede utilizar el Subject real.

### Invariante 3

Las fases obligatorias son acumulativas salvo metadata explícita.

### Invariante 4

Una fase denegada impide ejecutar fases posteriores de la operación.

---

# 212. Security Invariants

### Invariante 1

Evaluadores críticos no pueden desaparecer silenciosamente.

### Invariante 2

Non-bypassable evaluators no pueden eliminarse mediante metadata ordinaria.

### Invariante 3

Config incompatibles deben rechazarse.

### Invariante 4

Planning failure nunca concede acceso.

### Invariante 5

Default Deny sigue vigente si el plan queda sin evaluadores decisivos.

---

# 213. Runtime Invariants

### Invariante 1

AuthorizationPlanner es stateless.

### Invariante 2

PlanTemplates son inmutables.

### Invariante 3

Runtime bindings son request-scoped.

### Invariante 4

No existe fuga de Principal/Tenant/Subject entre requests.

### Invariante 5

Plan cache estructural puede compartirse bajo FrankenPHP.

---

# 214. Performance Invariants

### Invariante 1

No Reflection en hot path productivo.

### Invariante 2

No filesystem discovery en planning runtime.

### Invariante 3

Los registries deberán estar indexados.

### Invariante 4

Los evaluadores deberán llegar preordenados cuando sea posible.

### Invariante 5

Los PlanTemplates podrán reutilizarse entre requests.

---

# 215. Arquitectura final

```text
                    AuthorizationRequest
                            │
                            ↓
                  AuthorizationPlanner
                            │
        ┌───────────────────┼───────────────────┐
        ↓                   ↓                   ↓
 Global Resolver      Gate Resolver       Policy Resolver
        │                   │                   │
        └───────────────────┼───────────────────┘
                            ↓
                  Candidate Evaluators
                            ↓
                  Applicability Filter
                            ↓
                     Deduplication
                            ↓
                   Requirement Resolve
                            ↓
                     Priority Resolve
                            ↓
                    Strategy Resolve
                            ↓
                  Stable Pipeline Sort
                            ↓
                    AuthorizationPlan
                            │
                            ↓
                  AuthorizationExecutor
                            │
                            ↓
                     DecisionVote
                            │
                            ↓
                    DecisionManager
                            │
                            ↓
                  Final DecisionResult
```

---

# 216. Arquitectura por fases

```text
HTTP Request
    ↓
Route Match
    ↓
Controller Resolution
    ↓
┌─────────────────────────────┐
│ PRE_RESOLUTION PLAN         │
│                             │
│ admin.access                │
│ account.suspended           │
│ security.mfa                │
└──────────────┬──────────────┘
               ↓
             GRANT
               ↓
       Argument Resolution
               ↓
        Resource Binding
               ↓
┌─────────────────────────────┐
│ RESOURCE PLAN               │
│                             │
│ tenant isolation            │
│ permission                  │
│ InvoicePolicy::update       │
│ compliance                  │
└──────────────┬──────────────┘
               ↓
             GRANT
               ↓
       Controller Execution
```

---

# 217. Ejemplo completo

Controller:

```php
#[Authorize(
    'admin.access',
    phase: AuthorizationPhase::PreResolution
)]
final class InvoiceAdminController
{
    #[Authorize(
        'approve',
        subject: 'invoice'
    )]
    public function approve(
        Invoice $invoice
    ): Response {
        // ...
    }
}
```

---

# 218. Plan PreResolution

```text
Ability:
admin.access

Subject:
InvoiceAdminController::class

Evaluators:

PlatformLockdownPolicy
priority=10000
critical

SuspendedPrincipalPolicy
priority=9000
critical

AdminAccessGate
priority=6000
required

Strategy:
DenyOverrides
```

---

# 219. Resultado PreResolution

```text
PlatformLockdown → ABSTAIN
SuspendedPrincipal → ABSTAIN
AdminAccessGate → GRANT
```

Final:

```text
GRANT
```

---

# 220. Resource Binding

Ahora se resuelve:

```text
Invoice#928
```

---

# 221. Resource Plan

```text
Ability:
approve

Subject:
Invoice#928

Evaluators:

TenantIsolationPolicy
priority=8000
critical

MfaRequirementPolicy
priority=7000
required

InvoicePermissionVoter
priority=6000
required

InvoicePolicy::approve
priority=5000
required

FinancialCompliancePolicy
priority=4000
required

Strategy:
DenyOverrides
```

---

# 222. Resultados

```text
TenantIsolation → GRANT
MfaRequirement → GRANT
Permission → GRANT
InvoicePolicy → GRANT
Compliance → DENY
```

---

# 223. Short-circuit

Con `DenyOverrides`:

```text
Compliance DENY
      ↓
final DENY
```

No se ejecutan evaluadores posteriores si existieran y ya no pueden cambiar la decisión.

---

# 224. Controller outcome

```text
PreResolution = GRANT
Resource = DENY
```

Por tanto:

```text
Controller action is not executed.
```

---

# 225. Diferenciador arquitectónico

El Planner permitirá que VoltStack haga algo más sofisticado que:

```text
$user->can(...)
```

sin sacrificar esa simplicidad.

Internamente podrá construir:

```text
Critical Security
        +
Tenant Isolation
        +
Role / Permission
        +
Resource Policy
        +
Compliance
        +
Context Rules
```

en un orden explícito y verificable.

---

# 226. Filosofía del sistema

La filosofía será:

```text
Resolve structurally.
Filter predictably.
Order deterministically.
Execute lazily.
Short-circuit safely.
Compile what can be compiled.
Keep runtime state isolated.
```

---

# 227. Resultado esperado

El `Authorization Planner and Policy Pipeline System` deberá proporcionar el puente entre:

```text
Authorization metadata
```

y:

```text
actual authorization execution
```

permitiendo que el Core mantenga una separación clara:

```text
AuthorizationRequest
        ↓
AuthorizationPlanner
        ↓
AuthorizationPlan
        ↓
AuthorizationPipeline
        ↓
AuthorizationExecutor
        ↓
DecisionManager
```

El principio definitivo será:

```text
Resolvers determine what may apply.

The Planner determines what will run.

The Pipeline defines the execution order.

The Executor performs the evaluations.

The DecisionManager determines the result.
```

Con este subsistema, VoltStack podrá aplicar autorización de forma temprana, eficiente y coherente sobre Routes, Controllers, Models, Commands, Jobs y cualquier otro Subject sin duplicar motores ni introducir lógica implícita.