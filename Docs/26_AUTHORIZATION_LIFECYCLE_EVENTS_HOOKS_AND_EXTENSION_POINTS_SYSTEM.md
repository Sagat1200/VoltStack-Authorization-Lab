# VoltStack Authorization System

## Authorization Lifecycle, Events, Hooks and Extension Points System

**Documento:** `26_AUTHORIZATION_LIFECYCLE_EVENTS_HOOKS_AND_EXTENSION_POINTS_SYSTEM.md`  
**Sistema:** Authorization  
**Framework:** VoltStack  
**Módulo sugerido:** `Quantum/Authorization`  
**Estado:** Especificación arquitectónica  
**Versión objetivo:** 1.x+

---

# 1. Propósito

Este documento define el **lifecycle completo de una decisión de autorización** dentro de VoltStack y formaliza los mecanismos seguros de:

- eventos;
- hooks;
- interceptores;
- listeners;
- observers;
- extension points;
- pre-evaluators;
- post-evaluators;
- decorators;
- lifecycle middleware;
- instrumentation hooks;
- policy lifecycle hooks;
- decision transformation control;
- extensibilidad de proveedores.

El objetivo es permitir que otros subsistemas y paquetes extiendan Authorization sin convertir el pipeline en un conjunto impredecible de callbacks capaces de modificar decisiones arbitrariamente.

El principio fundamental será:

> **Una extensión puede observar, enriquecer o restringir una autorización cuando su contrato lo permite; nunca deberá poder ampliar autoridad fuera de los límites establecidos por el Authorization Core.**

---

# 2. Problema arquitectónico

Hasta ahora VoltStack dispone conceptualmente de:

```text
AuthorizationRequest
Policies
Gates
Abilities
Voters
DecisionManager
RBAC
ABAC
ReBAC
Tenant Isolation
Scopes
Ownership
Sharing
Delegation
Capabilities
Risk
Approvals
SoD
Caching
Audit
Telemetry
```

Sin un lifecycle formal surgirían preguntas ambiguas:

```text
¿Cuándo puede intervenir un plugin?

¿Antes o después de resolver el Principal?

¿Un listener puede modificar el Tenant?

¿Un hook puede transformar DENY en ALLOW?

¿Los eventos son observacionales o mutables?

¿Dónde se aplican los evaluadores non-bypassable?

¿En qué momento se escribe el audit?

¿Una excepción de un listener rompe la autorización?

¿Puede un plugin agregar un Challenge?

¿Cuándo se ejecuta el cache?

¿Cuándo se genera la explicación?

¿Qué eventos son síncronos?

¿Cuáles pueden enviarse async?

¿Qué ocurre en FrankenPHP al terminar la evaluación?
```

Este documento establece las reglas.

---

# 3. Principios del lifecycle

VoltStack deberá respetar:

```text
Determinism
Explicit ordering
Fail-closed security
Immutable request identity
Controlled mutation
Observable execution
Non-bypassable security phases
Idempotent hooks where required
Request isolation
Stable extension contracts
```

---

# 4. Lifecycle general

El flujo conceptual será:

```text
Authorization Invocation
        ↓
Request Construction
        ↓
Request Validation
        ↓
Execution Context Resolution
        ↓
Principal / Actor Resolution
        ↓
Tenant Resolution
        ↓
Scope Resolution
        ↓
Authority Context Resolution
        ↓
Plan Resolution
        ↓
Cache Lookup
        ↓
Preconditions
        ↓
Structural Evaluators
        ↓
Policy / Gate / Voter Evaluation
        ↓
Contextual / Risk Evaluation
        ↓
Approval / SoD Evaluation
        ↓
Non-Bypassable Security Evaluation
        ↓
Decision Aggregation
        ↓
Decision Normalization
        ↓
Challenge Normalization
        ↓
Explanation
        ↓
Audit / Telemetry
        ↓
Decision Cache
        ↓
Result Return
        ↓
Evaluation Cleanup
```

---

# 5. Lifecycle Phases

Se recomienda representar las fases con un enum explícito.

```php
enum AuthorizationLifecyclePhase: string
{
    case Invocation = 'invocation';
    case RequestConstruction = 'request_construction';
    case RequestValidation = 'request_validation';
    case ContextResolution = 'context_resolution';
    case PrincipalResolution = 'principal_resolution';
    case TenantResolution = 'tenant_resolution';
    case ScopeResolution = 'scope_resolution';
    case AuthorityResolution = 'authority_resolution';
    case Planning = 'planning';
    case CacheLookup = 'cache_lookup';
    case Preconditions = 'preconditions';
    case StructuralEvaluation = 'structural_evaluation';
    case PolicyEvaluation = 'policy_evaluation';
    case ContextualEvaluation = 'contextual_evaluation';
    case ApprovalEvaluation = 'approval_evaluation';
    case SecurityEvaluation = 'security_evaluation';
    case Aggregation = 'aggregation';
    case Normalization = 'normalization';
    case Explanation = 'explanation';
    case Audit = 'audit';
    case CacheStore = 'cache_store';
    case Completion = 'completion';
    case Cleanup = 'cleanup';
}
```

---

# 6. AuthorizationEvaluation

Cada autorización deberá poseer una identidad propia.

```php
final readonly class AuthorizationEvaluation
{
    public function __construct(
        public string $id,
        public AuthorizationRequest $request,
        public AuthorizationContext $context,
        public AuthorizationLifecyclePhase $phase,
    ) {}
}
```

Esto permite correlacionar:

```text
events
logs
traces
audit
errors
challenge
cache
```

---

# 7. Evaluation ID

El `evaluationId` deberá:

```text
ser único dentro del runtime
ser opaco
no contener información sensible
acompañar eventos y telemetry
```

---

# 8. Lifecycle Context

Durante una evaluación podrá existir un objeto interno:

```php
final class AuthorizationLifecycleContext
{
    public function __construct(
        public AuthorizationRequest $request,
        public AuthorizationContext $context,
        public AuthorizationEvaluationState $state,
    ) {}
}
```

---

# 9. Estado interno

`AuthorizationEvaluationState` podrá contener:

```text
current phase
resolved policy
resolved voters
candidate decisions
risk assessment
approval state
cache state
explanation fragments
timings
```

No deberá exponerse indiscriminadamente a plugins.

---

# 10. Immutable Core Request

Una vez completada `RequestValidation`, los atributos críticos:

```text
ability
principal
actor
subject
tenant
scope
authority mode
```

deberán considerarse inmutables salvo mecanismos explícitamente seguros.

---

# 11. Why

Un hook no deberá poder ejecutar:

```text
document.view
```

y silenciosamente convertirlo en:

```text
document.delete
```

---

# 12. Controlled Enrichment

Sí deberá permitirse enriquecer:

```text
request metadata
context attributes
telemetry metadata
explanation metadata
```

mediante contratos definidos.

---

# 13. Event vs Hook

VoltStack deberá distinguir estrictamente:

```text
Event
```

de:

```text
Hook
```

---

# 14. Event

Un evento describe:

> Algo que ha ocurrido.

Normalmente deberá ser:

```text
immutable
observational
non-authoritative
```

---

# 15. Hook

Un hook representa:

> Un punto explícito donde una extensión puede participar en el lifecycle bajo un contrato definido.

Puede producir:

```text
enrichment
restriction
additional evaluator
additional requirement
```

pero solo dentro de su capability.

---

# 16. Events must not mutate decisions

Regla general:

```text
Event Listener
    ↓
may observe
    ↓
must not alter AuthorizationDecision
```

---

# 17. Ejemplo incorrecto

```php
#[ListenTo(AuthorizationDenied::class)]
public function handle($event)
{
    $event->decision->allow();
}
```

Esto deberá ser imposible por diseño.

---

# 18. Event objects readonly

Preferir:

```php
final readonly class AuthorizationDenied
{
}
```

---

# 19. Hooks explícitos

Los hooks sí podrán actuar sobre estructuras específicas.

Ejemplo:

```php
interface AuthorizationContextEnricherInterface
{
    public function enrich(
        AuthorizationContext $context,
        AuthorizationRequest $request
    ): AuthorizationContext;
}
```

---

# 20. Hook capability model

Cada extensión deberá tener una capacidad limitada.

Tipos conceptuales:

```text
Observe
EnrichContext
AddRestriction
AddEvaluator
AddChallengeRequirement
DecorateService
AddExplanation
AddTelemetry
```

No deberá existir una capability genérica:

```text
ModifyAnything
```

---

# 21. AuthorizationExtensionCapability

```php
enum AuthorizationExtensionCapability: string
{
    case Observe = 'observe';
    case ContextEnrichment = 'context_enrichment';
    case Restriction = 'restriction';
    case Evaluator = 'evaluator';
    case Challenge = 'challenge';
    case Explanation = 'explanation';
    case Telemetry = 'telemetry';
}
```

---

# 22. Extension descriptor

```php
final readonly class AuthorizationExtensionDescriptor
{
    public function __construct(
        public string $id,
        public AuthorizationExtensionCapability $capability,
        public int $priority = 0,
        public bool $nonBypassable = false,
    ) {}
}
```

---

# 23. Non-Bypassable Extensions

Algunos evaluadores serán registrados como:

```text
nonBypassable=true
```

Ejemplos:

```text
TenantIsolationEvaluator
PrincipalStateEvaluator
ImpersonationRestrictionEvaluator
SecurityFloorEvaluator
```

---

# 24. Regla

Un extension point normal no podrá remover ni saltar un evaluator:

```text
nonBypassable
```

---

# 25. Event Taxonomy

Los eventos deberán dividirse en categorías.

```text
Lifecycle Events
Decision Events
Policy Events
Security Events
Challenge Events
Audit Events
Failure Events
```

---

# 26. Lifecycle Events

Ejemplos:

```text
AuthorizationInvoked
AuthorizationRequestConstructed
AuthorizationContextResolved
AuthorizationPlanResolved
AuthorizationEvaluationStarted
AuthorizationEvaluationCompleted
```

---

# 27. Decision Events

```text
AuthorizationGranted
AuthorizationDenied
AuthorizationAbstained
AuthorizationChallenged
AuthorizationFailed
```

---

# 28. Policy Events

```text
AuthorizationPolicyResolving
AuthorizationPolicyResolved
AuthorizationPolicyEvaluating
AuthorizationPolicyEvaluated
```

---

# 29. Security Events

```text
AuthorizationTenantViolationDetected
AuthorizationScopeViolationDetected
AuthorizationPrivilegeEscalationAttemptDetected
AuthorizationInvalidCapabilityDetected
AuthorizationSoDConflictDetected
```

---

# 30. Challenge Events

```text
AuthorizationChallengeIssued
AuthorizationChallengeSatisfied
AuthorizationChallengeExpired
AuthorizationChallengeRejected
```

---

# 31. Failure Events

```text
AuthorizationProviderFailed
AuthorizationPolicyFailed
AuthorizationExtensionFailed
AuthorizationInfrastructureFailed
```

---

# 32. Lifecycle event base

```php
interface AuthorizationLifecycleEventInterface
{
    public function evaluationId(): string;

    public function occurredAt(): DateTimeImmutable;
}
```

---

# 33. Event payload minimization

Los eventos deberán contener referencias, no grandes object graphs.

Preferir:

```text
PrincipalReference
SubjectReference
TenantReference
AbilityName
DecisionSummary
```

sobre:

```text
full ORM entity
full request
database connection
```

---

# 34. Why

Esto reduce:

```text
memory retention
serialization risks
async event coupling
privacy exposure
persistent-worker leaks
```

---

# 35. Sync vs Async Events

Cada evento deberá declarar si puede ser:

```text
synchronous
asynchronous
```

---

# 36. Security-critical events

Eventos necesarios para enforcement no deberán depender de ejecución async.

En realidad, enforcement deberá ocurrir antes y el evento solo describirá el resultado.

---

# 37. Audit events

Podrán enviarse async si existe una garantía apropiada de entrega.

---

# 38. Audit durability

Para operaciones críticas puede requerirse:

```text
synchronous local append
+
async export
```

---

# 39. Event dispatch failure

Un listener observacional no deberá alterar normalmente la decisión.

Ejemplo:

```text
Metrics listener failed
```

no debe convertir:

```text
DENY → ALLOW
```

ni:

```text
ALLOW → DENY
```

salvo que el evento sea parte de un requisito explícitamente crítico.

---

# 40. Critical observer

Si una organización exige audit obligatorio, no deberá modelarse como listener opcional.

Deberá existir un evaluator o sink:

```text
RequiredAuthorizationAuditSink
```

con failure policy propia.

---

# 41. Event priority

Listeners pueden tener prioridad.

```php
#[AuthorizationListener(priority: 100)]
```

---

# 42. Priority does not imply authority

Un listener con mayor prioridad no recibe más capacidad de modificación.

---

# 43. Hook ordering

Cada categoría deberá tener orden explícito.

Ejemplo:

```text
Platform Hooks
    ↓
Framework Hooks
    ↓
Package Hooks
    ↓
Application Hooks
    ↓
Tenant Hooks
```

---

# 44. Security floor

Las capas inferiores podrán:

```text
add restrictions
add challenge requirements
```

pero no remover:

```text
platform restrictions
```

---

# 45. Hook result

Contrato común:

```php
interface AuthorizationHookResultInterface
{
}
```

Especializaciones:

```text
ContextEnrichmentResult
RestrictionResult
ChallengeRequirementResult
EvaluatorRegistrationResult
```

---

# 46. Pre-Evaluation Hooks

Se ejecutan antes de los evaluadores principales.

Deben utilizarse para:

```text
context enrichment
additional preconditions
request-local diagnostics
```

---

# 47. No pre-allow

Un pre-hook no deberá retornar:

```text
ALLOW FINAL
```

saltándose Policies.

---

# 48. Pre-deny

Sí puede existir:

```text
hard security restriction
→ DENY
```

cuando el hook fue registrado con capability de `Restriction`.

---

# 49. Post-Evaluation Hooks

Se ejecutan después de obtener candidate decisions.

Podrán:

```text
add stricter constraints
enrich explanation
instrument
```

---

# 50. No post-allow escalation

Un Post Hook no deberá transformar:

```text
DENY
```

en:

```text
ALLOW
```

---

# 51. Restriction monotonicity

Regla:

```text
Extension authority is monotonic toward restriction.
```

Es decir:

```text
ALLOW → CHALLENGE
ALLOW → DENY
CHALLENGE → DENY
```

puede permitirse según hook.

Pero:

```text
DENY → ALLOW
CHALLENGE → ALLOW
```

no.

---

# 52. Monotonic Decision Ordering

Conceptualmente:

```text
ALLOW
    ↓
CHALLENGE
    ↓
DENY
```

con `FAILURE` fuera de esa escala.

---

# 53. RestrictionResult

```php
final readonly class AuthorizationRestrictionResult
{
    public function __construct(
        public AuthorizationRestrictionOutcome $outcome,
        public ?string $reason = null,
    ) {}
}
```

---

# 54. AuthorizationRestrictionOutcome

```php
enum AuthorizationRestrictionOutcome: string
{
    case NoChange = 'no_change';
    case Challenge = 'challenge';
    case Deny = 'deny';
    case Failure = 'failure';
}
```

---

# 55. Context Enrichment Hooks

Podrán añadir atributos como:

```text
device trust
network zone
business calendar
security classification
external identity attributes
```

---

# 56. Enrichment cannot overwrite authoritative context

Si el contexto ya contiene:

```text
tenant=Tenant#7
source=Authoritative
```

un hook no deberá reemplazarlo por:

```text
Tenant#9
```

---

# 57. Context merge policy

Debe considerar:

```text
attribute trust level
source priority
immutability
```

---

# 58. ContextConflictException

Cuando dos authoritative providers producen valores incompatibles:

```text
FAILURE
```

preferiblemente.

---

# 59. Hook isolation

Una extensión no deberá recibir referencias mutables a:

```text
PolicyRegistry
DecisionManager internals
Container internals
```

si no las necesita.

---

# 60. Extension Context

```php
final readonly class AuthorizationExtensionContext
{
    public function __construct(
        public string $evaluationId,
        public AuthorizationLifecyclePhase $phase,
        public AuthorizationRequestView $request,
        public AuthorizationContextView $context,
    ) {}
}
```

---

# 61. Read-only views

Plugins observacionales deberán recibir:

```text
AuthorizationRequestView
AuthorizationContextView
```

no los objetos internos mutables.

---

# 62. Policy Lifecycle

El lifecycle de una Policy será:

```text
Policy Resolution
      ↓
Policy Method Resolution
      ↓
Parameter Resolution
      ↓
Pre-Policy Hooks
      ↓
Policy Invocation
      ↓
Raw Result
      ↓
Result Normalization
      ↓
Post-Policy Hooks
```

---

# 63. PolicyResolving Event

Se emite cuando se inicia resolución.

---

# 64. PolicyResolved Event

Incluye:

```text
subject type
policy ID
resolution source
```

---

# 65. PolicyNotFound

Debe diferenciarse entre:

```text
expected policy missing
```

y:

```text
no policy required
```

---

# 66. Policy Resolver Extension

Plugins podrán registrar:

```php
interface AuthorizationPolicyProviderInterface
{
    public function resolve(
        AuthorizationPolicyQuery $query
    ): ?AuthorizationPolicyReference;
}
```

---

# 67. Provider order

Ejemplo:

```text
ExplicitMappingProvider
AttributePolicyProvider
ConventionPolicyProvider
PackagePolicyProvider
FallbackProvider
```

---

# 68. Resolution determinism

Si dos providers producen Policies incompatibles:

```text
ambiguous resolution
```

debe detectarse.

---

# 69. No first-random-wins

El orden accidental de descubrimiento no deberá definir la Policy.

---

# 70. Gate Lifecycle

```text
Ability Resolution
      ↓
Gate Definition Lookup
      ↓
Before Gate Restrictions
      ↓
Gate Evaluators
      ↓
Decision Strategy
      ↓
After Gate Observers
```

---

# 71. Gate before hook

VoltStack podrá soportar un equivalente conceptual de:

```text
before
```

pero no deberá convertirse en superuser callback arbitrario.

---

# 72. Safe Gate Before

Puede:

```text
DENY
CHALLENGE
ABSTAIN
```

---

# 73. Privileged Gate Grant

Si se requiere un grant temprano por arquitectura, deberá provenir de una fuente formal como:

```text
SystemPrincipal
Capability
PlatformAuthority
```

no de un callback:

```php
return true;
```

sin provenance.

---

# 74. Laravel-style before compatibility

Si VoltStack ofrece una API familiar tipo:

```php
Gate::before(...)
```

internamente deberá traducirse a:

```text
EarlyAuthorityEvaluator
```

con metadata y audit.

---

# 75. No invisible super-admin

Nunca:

```text
if user.isAdmin return true;
```

como mecanismo global oculto.

---

# 76. Voter Lifecycle

```text
Resolve Voter Plan
      ↓
Filter Applicable Voters
      ↓
Evaluate Voters
      ↓
Collect Candidate Decisions
      ↓
Decision Strategy
```

---

# 77. Voter applicability

Un Voter podrá devolver:

```text
ABSTAIN
```

cuando no aplica.

---

# 78. Voter exceptions

Una excepción deberá convertirse según Failure Policy en:

```text
FAILURE
```

no en ABSTAIN silencioso.

---

# 79. Voter events

```text
AuthorizationVoterEvaluating
AuthorizationVoterEvaluated
AuthorizationVoterFailed
```

---

# 80. Voter telemetry

Podrá registrar:

```text
voter ID
duration
outcome
```

sin datos sensibles.

---

# 81. Structural Evaluator Lifecycle

Los evaluadores estructurales incluyen:

```text
RBAC
ABAC
ReBAC
Ownership
Sharing
Delegation
Capability
Scope
Tenant
```

---

# 82. Evaluator Descriptor

```php
final readonly class AuthorizationEvaluatorDescriptor
{
    public function __construct(
        public string $id,
        public int $priority,
        public bool $nonBypassable,
        public bool $cacheable,
    ) {}
}
```

---

# 83. Evaluator Plan

El Planner deberá producir un orden explícito.

---

# 84. Example

```text
1 PrincipalStateEvaluator
2 TenantIsolationEvaluator
3 AuthorityModeEvaluator
4 ScopeEvaluator
5 RBACEvaluator
6 OwnershipEvaluator
7 RelationshipEvaluator
8 ResourcePolicyEvaluator
9 RiskEvaluator
10 ApprovalEvaluator
11 SecurityFloorEvaluator
```

---

# 85. Priority is not enough

Algunas dependencias deben modelarse como:

```text
before
after
requires
```

---

# 86. Evaluator dependency graph

```php
interface AuthorizationEvaluatorDependencyInterface
{
    public function before(): array;

    public function after(): array;
}
```

---

# 87. Cycle detection

Ejemplo inválido:

```text
A after B
B after A
```

debe fallar durante compilación.

---

# 88. Compiled evaluator order

El hot path no deberá ordenar evaluadores por request.

---

# 89. Lifecycle Middleware

VoltStack podrá definir un concepto:

```text
AuthorizationMiddleware
```

interno al Authorization Engine, diferente del HTTP Middleware.

---

# 90. Contract

```php
interface AuthorizationLifecycleMiddlewareInterface
{
    public function process(
        AuthorizationLifecycleContext $context,
        AuthorizationLifecycleNext $next
    ): AuthorizationDecision;
}
```

---

# 91. Use cases

Adecuado para:

```text
tracing
timing
audit wrappers
diagnostics
cache
```

---

# 92. Security warning

Lifecycle Middleware con capacidad de devolver cualquier decision es poderoso.

Por tanto deberá dividirse entre:

```text
TrustedCoreMiddleware
ObservationalMiddleware
RestrictiveMiddleware
```

---

# 93. TrustedCoreMiddleware

Solo Core/explicit trusted extensions pueden modificar flujo completo.

---

# 94. ObservationalMiddleware

Debe llamar a `$next` y no modificar resultado.

---

# 95. RestrictiveMiddleware

Puede:

```text
preserve
challenge
deny
failure
```

pero no ampliar.

---

# 96. Middleware classification

```php
enum AuthorizationMiddlewareCapability: string
{
    case Core = 'core';
    case Observe = 'observe';
    case Restrict = 'restrict';
}
```

---

# 97. Event Hooks vs Middleware

Usar Events para:

```text
observation
integration
notifications
telemetry
```

Usar Hooks/Middleware para:

```text
controlled lifecycle participation
```

---

# 98. Cache lifecycle

Cache lookup debe ocurrir únicamente después de haber identificado suficientes dimensiones de la decisión.

---

# 99. Cache lookup inputs

Como mínimo puede depender de:

```text
principal
actor when relevant
ability
subject
tenant
scope
authority mode
context fingerprint
policy version
```

---

# 100. CacheLookup Event

Podrá emitir:

```text
AuthorizationCacheHit
AuthorizationCacheMiss
AuthorizationCacheBypassed
```

---

# 101. Cache hit lifecycle

Incluso con cache hit, algunos evaluadores deberán ejecutarse.

Ejemplos:

```text
non-bypassable realtime constraints
critical revocation checks
```

---

# 102. Cached decision cannot bypass security

Pipeline:

```text
Cache Hit
    ↓
Cached Candidate Decision
    ↓
Mandatory Runtime Restrictions
    ↓
Final Decision
```

---

# 103. Cache write

Solo después de:

```text
normalization
cacheability assessment
```

---

# 104. Cache observers

Podrán observar hits/misses pero no alterar keys arbitrariamente.

---

# 105. Audit lifecycle

El Audit System deberá recibir un evento final normalizado.

---

# 106. Audit Record

Podrá contener:

```text
evaluation ID
principal
actor
tenant
scope
ability
subject
decision
reason codes
authority sources
policy version
```

---

# 107. Audit timing

Para decisión estándar:

```text
Decision Normalized
    ↓
Audit
    ↓
Return
```

---

# 108. Required audit

Si Policy exige audit durable antes de ejecutar:

```text
Audit Requirement
```

deberá modelarse como condición formal.

---

# 109. Telemetry lifecycle

Tracing deberá comenzar lo más cerca posible de:

```text
Authorization Invocation
```

y cerrar en:

```text
Completion / Failure
```

---

# 110. Root span

Conceptual:

```text
authorization.evaluate
```

---

# 111. Phase spans

```text
authorization.resolve_context
authorization.plan
authorization.evaluate_policy
authorization.evaluate_risk
authorization.evaluate_approval
authorization.normalize
```

---

# 112. Hook timing telemetry

Extensions podrán medirse para detectar:

```text
slow hooks
slow voters
slow external providers
```

---

# 113. Extension performance budgets

Cada extension point podrá declarar:

```text
timeout
cost class
sync/async
```

---

# 114. External hooks

Un hook síncrono que dependa de red externa deberá ser explícito.

---

# 115. External Hook Timeout

Debe existir timeout finito.

---

# 116. Failure policy

Ejemplo:

```php
enum AuthorizationExtensionFailurePolicy: string
{
    case Fail = 'fail';
    case Deny = 'deny';
    case IgnoreObservation = 'ignore_observation';
}
```

---

# 117. No Ignore Security Hook

Una extensión de tipo `Restriction` marcada mandatory no podrá usar:

```text
IgnoreObservation
```

---

# 118. Observation failure

Telemetry opcional sí puede ignorarse controladamente.

---

# 119. Hook idempotency

Algunos hooks deberán ser idempotentes.

Especialmente:

```text
context enrichers
audit projectors
retryable external observers
```

---

# 120. Evaluation retries

Si el engine reintenta una operación interna, no deberá duplicar side effects observacionales de forma incorrecta.

---

# 121. Lifecycle Hooks

Contratos sugeridos:

```text
BeforeAuthorizationEvaluationHook
AfterAuthorizationEvaluationHook
BeforePolicyEvaluationHook
AfterPolicyEvaluationHook
BeforeDecisionAggregationHook
AfterDecisionNormalizationHook
```

---

# 122. BeforeAuthorizationEvaluationHook

Permitido:

```text
context enrichment
restriction
diagnostics
```

---

# 123. AfterAuthorizationEvaluationHook

Permitido:

```text
observation
explanation enrichment
telemetry
```

No:

```text
decision escalation
```

---

# 124. BeforeDecisionAggregationHook

Puede añadir candidate restrictions si su capability lo permite.

---

# 125. AfterDecisionNormalizationHook

No podrá convertir una decisión normalizada en una más permisiva.

---

# 126. Explanation Hooks

Podrán agregar:

```text
reason fragments
relationship paths
authority provenance
debug hints
```

---

# 127. Explanation sanitization

Toda explicación deberá pasar por:

```text
AuthorizationExplanationSanitizer
```

antes de exponerse al cliente.

---

# 128. Explainability audiences

```text
Public
Developer
Operator
Security
Audit
```

---

# 129. Extension-provided explanation

Debe declarar audiencia máxima.

---

# 130. Example

Risk plugin puede aportar:

```text
risk_too_high
```

para público.

Y:

```text
provider=AcmeRisk, rule=R17
```

solo para Security.

---

# 131. Challenge lifecycle

```text
Candidate Challenge
      ↓
Requirement Collection
      ↓
Requirement Normalization
      ↓
Challenge Policy
      ↓
Challenge Issuance
```

---

# 132. Challenge Hooks

Plugins podrán añadir requirements.

Ejemplo:

```text
RequiresDeviceVerification
```

---

# 133. Challenge cannot remove stricter requirement

Si Core exige:

```text
VeryStrong
```

un plugin no puede cambiarlo a:

```text
Standard
```

---

# 134. Requirement merge monotonicity

Más de un requirement se combina preservando el más fuerte.

---

# 135. Approval lifecycle hook

Approval subsystem podrá engancharse en:

```text
Post Structural Authority
+
Post Contextual Evaluation
```

para producir:

```text
CHALLENGE approval.required
```

---

# 136. SoD lifecycle

Debe evaluarse:

```text
before approval evidence is accepted
```

y nuevamente cuando sea necesario:

```text
before execution
```

---

# 137. Delegation lifecycle

Delegation debe resolverse antes de evaluadores que dependan de autoridad efectiva.

---

# 138. Impersonation lifecycle

Debe establecerse:

```text
Actor
EffectivePrincipal
```

antes de Policies.

---

# 139. Capability lifecycle

Capability validation deberá ocurrir antes de utilizarla como authority source.

---

# 140. Capability event

```text
AuthorizationCapabilityValidated
AuthorizationCapabilityRejected
```

---

# 141. Security Event Dispatch

Eventos de seguridad deberán emitirse después de que la violación haya sido detectada, no antes.

---

# 142. Example

```text
Tenant mismatch
    ↓
DENY
    ↓
AuthorizationTenantViolationDetected
```

---

# 143. No security-event veto

Un listener no puede responder:

```text
ignore violation
```

---

# 144. Lifecycle cancellation

Algunas fases pueden finalizar evaluación tempranamente.

---

# 145. Terminal outcomes

```text
DENY
FAILURE
```

pueden ser terminales.

`CHALLENGE` puede ser terminal para la request actual.

---

# 146. ABSTAIN

No siempre terminal.

---

# 147. ALLOW candidate

Tampoco es terminal hasta ejecutar:

```text
remaining mandatory evaluators
```

---

# 148. Short Circuit Matrix

Ejemplo:

| Resultado | ¿Puede cortar? |
| --- | --- |
| NonBypassable DENY | Sí |
| Infrastructure FAILURE | Sí |
| Policy ABSTAIN | No |
| Candidate ALLOW | No necesariamente |
| Challenge | Depende de si quedan DENYs mandatory |
| Cache ALLOW | No |

---

# 149. Mandatory Tail

Debe existir una fase final:

```text
MandatorySecurityTail
```

---

# 150. Purpose

Garantizar que ningún shortcut omita:

```text
security floor
tenant constraints
revocation constraints
critical restrictions
```

---

# 151. Final decision point

Solo después de Mandatory Security Tail puede emitirse:

```text
AuthorizationGranted
```

---

# 152. Lifecycle state machine

```text
CREATED
   ↓
VALIDATED
   ↓
CONTEXT_RESOLVED
   ↓
PLANNED
   ↓
EVALUATING
   ↓
AGGREGATING
   ↓
NORMALIZED
   ↓
AUDITED
   ↓
COMPLETED
```

Alternativas:

```text
FAILED
CHALLENGED
DENIED
```

como outcomes, no necesariamente estados internos únicos.

---

# 153. AuthorizationEvaluationState

```php
enum AuthorizationEvaluationStatus: string
{
    case Created = 'created';
    case Running = 'running';
    case Completed = 'completed';
    case Failed = 'failed';
}
```

---

# 154. Decision separate from state

Una evaluación puede estar:

```text
Completed
```

con decisión:

```text
DENY
```

---

# 155. Reentrancy

Una Policy puede, en algunos casos, solicitar otra autorización.

Ejemplo:

```text
document.publish
    ↓
check document.update
```

---

# 156. Reentrant authorization

Debe soportarse controladamente.

---

# 157. Evaluation stack

Request local:

```text
Evaluation A
    ↓
Evaluation B
```

---

# 158. Recursive loop detection

Ejemplo inválido:

```text
A authorizes B
B authorizes A
```

sin base case.

---

# 159. Evaluation recursion guard

```php
interface AuthorizationRecursionGuardInterface
{
    public function enter(
        AuthorizationRequestFingerprint $request
    ): void;

    public function leave(): void;
}
```

---

# 160. Max evaluation depth

Configurable.

---

# 161. Recursive Authorization Failure

Debe producir:

```text
authorization.recursion_detected
```

---

# 162. Context inheritance in nested evaluations

Nested evaluation podrá heredar:

```text
principal
actor
tenant
session
```

pero subject/ability serán nuevos.

---

# 163. Scope changes

Solo mediante explicit nested scope resolution.

---

# 164. Delegation inheritance

Debe obedecer authority mode actual.

---

# 165. No ambient escalation

Nested authorization no deberá obtener authority nativa si la evaluación exterior está restringida a:

```text
Delegated Mode
```

salvo explicit policy.

---

# 166. Async listeners

Events async deberán usar payloads serializables.

---

# 167. Never serialize

```text
closures
container services
PDO objects
request objects
full policies
```

---

# 168. AuthorizationEventEnvelope

```php
final readonly class AuthorizationEventEnvelope
{
    public function __construct(
        public string $eventType,
        public string $evaluationId,
        public DateTimeImmutable $occurredAt,
        public array $references,
        public array $metadata,
    ) {}
}
```

---

# 169. Event schema version

Eventos externos deberán versionarse.

```text
authorization.event.version=1
```

---

# 170. Backward compatibility

Cambios breaking deberán crear nueva versión del schema.

---

# 171. Internal events

No todos los eventos internos necesitan estabilidad pública.

---

# 172. Public extension events

Deberán estar documentados y versionados.

---

# 173. Extension Point Stability

Clasificación:

```text
Internal
Experimental
Stable
Protected
```

---

# 174. Stable Extension Point

Promete compatibilidad semántica.

---

# 175. Protected Extension Point

Solo para Core o trusted modules.

---

# 176. Experimental

Puede cambiar entre versiones menores según política del framework.

---

# 177. Public plugin SDK

Authorization deberá exponer únicamente contracts necesarios.

---

# 178. No dependency on internals

Plugins no deberían importar clases bajo namespaces como:

```text
Internal\
Runtime\MutableState\
```

---

# 179. Extension Registry

```php
interface AuthorizationExtensionRegistryInterface
{
    public function register(
        AuthorizationExtensionDescriptor $descriptor,
        object $extension
    ): void;
}
```

---

# 180. Compile-time validation

El registry deberá validar:

```text
capability
phase compatibility
priority
dependencies
protected phases
```

---

# 181. Example invalid extension

Un plugin `Observe` intentando registrarse en:

```text
decision_mutation
```

debe fallar.

---

# 182. Extension dependencies

Ejemplo:

```text
RiskExplanationExtension
requires RiskEvaluator
```

---

# 183. Conditional extension activation

Puede depender de:

```text
feature enabled
tenant profile
ability
subject type
principal type
```

---

# 184. Avoid per-request full extension scan

El Planner deberá precompilar los extensions aplicables cuando sea posible.

---

# 185. Hook selectors

```php
final readonly class AuthorizationHookSelector
{
    public function __construct(
        public ?array $abilities = null,
        public ?array $subjectTypes = null,
        public ?array $principalTypes = null,
    ) {}
}
```

---

# 186. Compiled Hook Plan

```text
Ability
    ↓
Applicable Hooks
    ↓
Pre-sorted execution list
```

---

# 187. Dynamic tenant hooks

Podrán agregarse mediante tenant profile, pero no reescribir el core graph arbitrariamente.

---

# 188. Tenant-specific extension

Ejemplo:

```text
Tenant A
requires ComplianceRestrictionEvaluator
```

---

# 189. Tenant hook cache

Debe estar versionado por:

```text
tenant authorization version
```

---

# 190. Hot reload

En desarrollo podrán recompilarse extension plans.

---

# 191. Production

Preferir generation swap como documento 25.

---

# 192. Hook failure taxonomy

```text
HookUnavailable
HookTimeout
HookInvalidResult
HookException
HookSecurityViolation
```

---

# 193. Invalid hook result

Ejemplo:

Un `Restriction` hook intenta retornar:

```text
ALLOW_OVERRIDE
```

Debe producir:

```text
AuthorizationExtensionContractViolationException
```

---

# 194. Extension sandbox conceptual

Aunque PHP ejecute en el mismo proceso, el contrato debe limitar lo que puede hacer lógicamente.

---

# 195. Event listener recursion

Listener de:

```text
AuthorizationGranted
```

que solicita autorización puede crear ciclos.

---

# 196. Event dispatch recursion guard

Deberá existir para eventos sensibles.

---

# 197. Recommended

Listeners observacionales que requieren Authorization deberán hacerlo en un contexto separado o con guardas explícitas.

---

# 198. Audit listener exception

Audit crítico no deberá implementarse como simple listener opcional.

---

# 199. Transaction lifecycle

Algunas autorizaciones ocurren dentro de transacciones.

---

# 200. Event timing vs commit

Debe distinguirse:

```text
AuthorizationGranted
```

de:

```text
BusinessOperationCommitted
```

---

# 201. AuthorizationGranted does not mean business success

Importante.

---

# 202. Never infer execution from grant event

Un listener no deberá asumir:

```text
AuthorizationGranted
=
resource updated
```

---

# 203. Domain operation events

Eso pertenece al dominio.

---

# 204. Pre-execution authorization hook

Para operaciones críticas podría existir un checkpoint:

```text
AuthorizationPreExecutionCheckpoint
```

---

# 205. Use case

Revalidar:

```text
approval
risk
resource version
```

inmediatamente antes del efecto irreversible.

---

# 206. Checkpoint event

No reemplaza la autorización inicial.

---

# 207. Long-running operation

Puede haber:

```text
InitialAuthorization
CheckpointAuthorization
FinalCommit
```

---

# 208. Lifecycle snapshots

Para debugging, desarrollo podrá guardar:

```text
AuthorizationEvaluationSnapshot
```

---

# 209. Snapshot contents

```text
phase timings
candidate evaluators
decisions
reason codes
cache information
```

---

# 210. Security

Debe redactar:

```text
credentials
capability secrets
raw tokens
sensitive attributes
```

---

# 211. Debug hook

Podrá existir:

```text
AuthorizationDebugObserver
```

solo en development.

---

# 212. Production disable

No deberá incluir full snapshots por default.

---

# 213. Extension security policy

Una aplicación podrá definir:

```text
allowed plugin namespaces
trusted plugin IDs
allowed capabilities
```

---

# 214. Plugin trust levels

```php
enum AuthorizationPluginTrustLevel: string
{
    case Observational = 'observational';
    case Restricted = 'restricted';
    case Trusted = 'trusted';
    case Core = 'core';
}
```

---

# 215. Trust vs capability

Trust level limita qué capabilities puede registrar el plugin.

---

# 216. Example

Observational plugin:

```text
Telemetry extension
```

no puede registrar:

```text
nonBypassable evaluator
```

---

# 217. Trusted plugin

Puede registrar evaluadores adicionales, pero no eliminar core invariants.

---

# 218. Core

Solo Framework Core puede modificar protected lifecycle phases.

---

# 219. Extension provenance

Cada candidate decision producido por una extensión deberá incluir:

```text
extension ID
provider ID
source
```

---

# 220. Audit provenance

Permite explicar:

```text
DENY came from CompliancePlugin v2
```

---

# 221. Extension version

Deberá registrarse.

---

# 222. Manifest integration

Documento 25 deberá compilar:

```text
extensions
priorities
phases
dependencies
trust levels
```

en el Authorization Manifest.

---

# 223. Lifecycle Manifest

Ejemplo conceptual:

```php
return [
    'preconditions' => [
        TenantIsolationEvaluator::class,
        PrincipalStateEvaluator::class,
    ],

    'structural' => [
        RbacEvaluator::class,
        RelationshipEvaluator::class,
    ],

    'post' => [
        RiskEvaluator::class,
        ApprovalEvaluator::class,
    ],
];
```

---

# 224. Runtime immutability

La lista compilada deberá ser readonly.

---

# 225. Event listener registry

También puede compilarse.

---

# 226. Listener hot path

No descubrir attributes/listeners en cada request.

---

# 227. Event bus integration

Authorization usará el Event System general de VoltStack.

No deberá construir un event bus independiente salvo adaptador especializado.

---

# 228. Security Event Channel

Podrá utilizar un canal lógico:

```text
authorization.security
```

---

# 229. Observability channel

```text
authorization.telemetry
```

---

# 230. Domain separation

No mezclar:

```text
AuthorizationDenied
```

con eventos de:

```text
UserLoginFailed
```

que pertenecen a Authentication.

---

# 231. Cross-system correlation

Ambos pueden compartir:

```text
request ID
trace ID
principal reference
```

cuando proceda.

---

# 232. Runtime cleanup

Después de cada evaluación deberá limpiar:

```text
evaluation-local scratch state
nested evaluation stack
temporary extension state
```

---

# 233. Evaluation cleanup vs request cleanup

Distinguir:

```text
Evaluation cleanup
```

de:

```text
Request reset
```

---

# 234. Evaluation cleanup

Ocurre después de cada:

```text
Authorization::check()
```

---

# 235. Request reset

Ocurre al final del request completo.

---

# 236. Example

Un request puede ejecutar:

```text
20 authorization evaluations
```

Cada una debe liberar scratch state.

---

# 237. Hook local state

Si un hook necesita estado temporal deberá ser:

```text
evaluation-scoped
```

o:

```text
request-scoped
```

según contrato.

---

# 238. Never static mutable hook state

Especialmente bajo FrankenPHP.

---

# 239. Extension Reset Contract

```php
interface AuthorizationExtensionResettableInterface
{
    public function resetExtensionState(): void;
}
```

---

# 240. Lifecycle exceptions

Jerarquía propuesta:

```text
AuthorizationLifecycleException
├── AuthorizationHookException
├── AuthorizationEventDispatchException
├── AuthorizationExtensionException
├── AuthorizationExtensionTimeoutException
├── AuthorizationExtensionContractViolationException
├── AuthorizationLifecycleCycleException
└── AuthorizationEvaluationRecursionException
```

---

# 241. Failure normalization

Las excepciones deberán convertirse a:

```text
FAILURE
```

antes de salir del núcleo, salvo excepciones de programación en development.

---

# 242. Development strict mode

Puede permitir que ciertas violaciones contractuales hagan throw directamente para debugging.

---

# 243. Production mode

Debe producir:

```text
safe failure response
audit event
diagnostic correlation ID
```

---

# 244. Event delivery guarantees

Clasificación posible:

```text
BestEffort
AtLeastOnce
Transactional
Required
```

---

# 245. Authorization Event Delivery Policy

```php
enum AuthorizationEventDeliveryMode: string
{
    case BestEffort = 'best_effort';
    case AtLeastOnce = 'at_least_once';
    case Required = 'required';
}
```

---

# 246. Security events

Para algunos entornos:

```text
Required
```

puede ser necesario.

---

# 247. Event deduplication

Async consumers deberán utilizar:

```text
event ID
evaluation ID
event type
```

para deduplicación.

---

# 248. Hook side-effect rules

Por default, hooks de evaluation deberían ser:

```text
side-effect free
```

excepto categorías explícitas como:

```text
audit
notification
external reporting
```

---

# 249. Why

Re-evaluar autorización no debería:

```text
send duplicate emails
create duplicate DB records
trigger business workflows
```

---

# 250. Query vs Command distinction

Authorization checking es principalmente una:

```text
query
```

No un command.

---

# 251. Side-effecting extension points

Deben estar separados.

Ejemplo:

```text
AuthorizationDecisionObserver
```

puede emitir telemetry.

No debe ejecutar la operación empresarial.

---

# 252. Policy hook purity

`BeforePolicyEvaluationHook` no deberá modificar el resource.

---

# 253. Resource mutation

Dentro de autorización es un anti-pattern.

---

# 254. No approval creation from pure event

Como documento 24:

```text
check authorization
```

no debería crear automáticamente workflows salvo API side-effecting explícita.

---

# 255. Extension APIs

VoltStack podrá ofrecer:

```php
Authorization::extend()
    ->observe(...)
    ->restrict(...)
    ->enrichContext(...)
    ->addEvaluator(...);
```

conceptualmente.

---

# 256. But package registration preferable

La mayoría deberá registrarse durante bootstrap, no dinámicamente durante request.

---

# 257. Dynamic request hooks

Solo para casos controlados:

```text
temporary test overrides
scoped execution
```

---

# 258. Scoped Extension

Ejemplo:

```php
$authorization->runWithRestriction(
    $restriction,
    fn () => ...
);
```

---

# 259. Scoped extension cleanup

Siempre:

```text
push
try
finally
pop
```

---

# 260. No ambient hook leakage

Request A / nested operation no debe contaminar siguientes evaluaciones.

---

# 261. Scoped Restriction Use Case

Una operación de servicio podría ejecutar código bajo:

```text
read-only authority ceiling
```

---

# 262. Authority ceilings integration

Documento 20.

Lifecycle deberá aplicar ceilings antes de final ALLOW.

---

# 263. Hook conflicts

Dos hooks pueden producir:

```text
Challenge
Deny
```

Debe usar:

```text
DenyOverrides
```

para restricciones.

---

# 264. Challenge merge

Dos challenges compatibles se combinan.

---

# 265. Challenge conflict

Ejemplo:

```text
requires human principal
```

y:

```text
service principal only
```

podría ser insatisfacible.

Debe producir:

```text
DENY / invalid policy configuration
```

según origen.

---

# 266. Compile-time conflict detection

Siempre que pueda detectarse estáticamente, hacerlo durante compilation.

---

# 267. Runtime conflict

Si depende del contexto, normalizar de forma segura.

---

# 268. Lifecycle determinism property

Mismos:

```text
request
context
policy versions
provider outputs
```

deben producir:

```text
same final decision
```

---

# 269. Event listeners excluded from semantics

Listeners observacionales no deberán cambiar esa propiedad.

---

# 270. Hook semantics included

Restrictive hooks sí forman parte de inputs semánticos.

---

# 271. Extension versioning in decision fingerprint

Si una decisión cacheada depende de hook:

```text
extension version
```

debe formar parte de cache validity.

---

# 272. Extension policy version

Podrá existir:

```text
authorization_extension_version
```

---

# 273. Hot swap

Nueva extensión activa:

```text
generation++
```

invalida relevant compiled plans/caches.

---

# 274. Lifecycle compiler

Podrá existir:

```php
interface AuthorizationLifecycleCompilerInterface
{
    public function compile(
        AuthorizationLifecycleDefinition $definition
    ): CompiledAuthorizationLifecycle;
}
```

---

# 275. Lifecycle definition

Contiene:

```text
phases
evaluators
hooks
dependencies
non-bypassable flags
failure policies
```

---

# 276. Compiled lifecycle

Hot path:

```text
array of precomputed phase executors
```

---

# 277. No dynamic sorting

Producción no debería ordenar prioridades en cada decision.

---

# 278. Phase executor

```php
interface AuthorizationPhaseExecutorInterface
{
    public function execute(
        AuthorizationLifecycleContext $context
    ): AuthorizationPhaseResult;
}
```

---

# 279. PhaseResult

Podrá contener:

```text
continue
candidate decision
terminal decision
challenge requirements
failure
```

---

# 280. Authorization Pipeline Engine

Contrato central:

```php
interface AuthorizationPipelineInterface
{
    public function evaluate(
        AuthorizationRequest $request,
        AuthorizationContext $context
    ): AuthorizationDecision;
}
```

---

# 281. AuthorizationManager vs Pipeline

`AuthorizationManager` será API/orquestador.

`AuthorizationPipeline` ejecutará el lifecycle compilado.

---

# 282. Separation

```text
AuthorizationManager
    ↓
Request normalization
    ↓
AuthorizationPipeline
    ↓
Decision
```

---

# 283. Event integration architecture

```text
AuthorizationPipeline
      │
      ├── emits lifecycle events
      │
      ├── invokes controlled hooks
      │
      └── invokes evaluators
```

---

# 284. Full execution architecture

```text
AuthorizationManager
        ↓
AuthorizationRequest
        ↓
Lifecycle Pipeline
        │
        ├── Request Validation
        ├── Context Resolution
        ├── Principal / Actor
        ├── Tenant / Scope
        ├── Authority Resolution
        ├── Plan
        ├── Cache
        ├── Structural Evaluators
        ├── Policy / Gates / Voters
        ├── Context / Risk
        ├── Approval / SoD
        ├── Security Tail
        ├── Decision Aggregation
        ├── Challenge Normalization
        ├── Explanation
        ├── Audit
        └── Cache Store
        ↓
AuthorizationDecision
```

---

# 285. Hook architecture

```text
                     LIFECYCLE PHASE
                           │
           ┌───────────────┼───────────────┐
           ↓               ↓               ↓
      Observers        Enrichers       Restrictions
           │               │               │
           │          may enrich       may narrow
           │               │               │
           └───────────────┼───────────────┘
                           ↓
                     CORE EVALUATOR
                           │
                           ↓
                       DECISION
```

---

# 286. Events architecture

```text
CORE ACTION
    ↓
FACT OCCURS
    ↓
IMMUTABLE EVENT
    ↓
EVENT BUS
    ├── Telemetry
    ├── Audit
    ├── Diagnostics
    ├── Notification
    └── External Integration
```

---

# 287. Core invariant

Events observe facts.

Hooks participate under contracts.

Evaluators decide authority.

These roles must remain separate.

---

# 288. Proposed directory structure

```text
Quantum/
└── Authorization/
    └── Lifecycle/
        ├── Contracts/
        │   ├── AuthorizationPipelineInterface.php
        │   ├── AuthorizationPhaseExecutorInterface.php
        │   ├── AuthorizationLifecycleHookInterface.php
        │   ├── AuthorizationLifecycleEventInterface.php
        │   ├── AuthorizationExtensionRegistryInterface.php
        │   └── AuthorizationLifecycleCompilerInterface.php
        │
        ├── Model/
        │   ├── AuthorizationEvaluation.php
        │   ├── AuthorizationEvaluationState.php
        │   ├── AuthorizationLifecyclePhase.php
        │   ├── AuthorizationLifecycleContext.php
        │   └── AuthorizationPhaseResult.php
        │
        ├── Pipeline/
        │   ├── AuthorizationPipeline.php
        │   ├── CompiledAuthorizationPipeline.php
        │   ├── AuthorizationPhaseExecutor.php
        │   └── MandatorySecurityTail.php
        │
        ├── Hooks/
        │   ├── AuthorizationHookRegistry.php
        │   ├── AuthorizationHookSelector.php
        │   ├── AuthorizationHookPlan.php
        │   ├── AuthorizationRestrictionHookInterface.php
        │   ├── AuthorizationContextEnricherInterface.php
        │   └── AuthorizationExplanationHookInterface.php
        │
        ├── Extensions/
        │   ├── AuthorizationExtensionDescriptor.php
        │   ├── AuthorizationExtensionCapability.php
        │   ├── AuthorizationPluginTrustLevel.php
        │   ├── AuthorizationExtensionContext.php
        │   └── AuthorizationExtensionFailurePolicy.php
        │
        ├── Events/
        │   ├── AuthorizationInvoked.php
        │   ├── AuthorizationEvaluationStarted.php
        │   ├── AuthorizationPlanResolved.php
        │   ├── AuthorizationGranted.php
        │   ├── AuthorizationDenied.php
        │   ├── AuthorizationChallenged.php
        │   ├── AuthorizationFailed.php
        │   ├── AuthorizationPolicyResolved.php
        │   ├── AuthorizationVoterEvaluated.php
        │   └── AuthorizationEvaluationCompleted.php
        │
        ├── Middleware/
        │   ├── AuthorizationLifecycleMiddlewareInterface.php
        │   ├── AuthorizationMiddlewareCapability.php
        │   ├── AuthorizationTracingMiddleware.php
        │   └── AuthorizationAuditMiddleware.php
        │
        ├── Compilation/
        │   ├── AuthorizationLifecycleCompiler.php
        │   ├── AuthorizationLifecycleDefinition.php
        │   ├── AuthorizationLifecycleDependencyResolver.php
        │   └── AuthorizationLifecycleInvariantValidator.php
        │
        ├── Recursion/
        │   ├── AuthorizationRecursionGuardInterface.php
        │   └── AuthorizationEvaluationStack.php
        │
        ├── Reset/
        │   ├── AuthorizationEvaluationResetter.php
        │   └── AuthorizationExtensionResettableInterface.php
        │
        └── Exceptions/
            ├── AuthorizationLifecycleException.php
            ├── AuthorizationHookException.php
            ├── AuthorizationExtensionException.php
            ├── AuthorizationExtensionTimeoutException.php
            ├── AuthorizationExtensionContractViolationException.php
            ├── AuthorizationLifecycleCycleException.php
            └── AuthorizationEvaluationRecursionException.php
```

---

# 289. Lifecycle Invariants

### Invariante 1

Toda evaluación posee un `evaluationId`.

### Invariante 2

Las fases ejecutan en orden determinista.

### Invariante 3

Las dependencias de evaluadores se validan antes de runtime.

### Invariante 4

Ningún shortcut evita el Mandatory Security Tail.

### Invariante 5

Evaluation-local state se limpia después de cada decisión.

---

# 290. Event Invariants

### Invariante 1

Los eventos describen hechos.

### Invariante 2

Los eventos no amplían autoridad.

### Invariante 3

Los eventos públicos poseen payload mínimo y versionable.

### Invariante 4

Un fallo de telemetry opcional no altera la decisión.

### Invariante 5

Security enforcement nunca depende únicamente de listeners observacionales.

---

# 291. Hook Invariants

### Invariante 1

Todo hook posee capability explícita.

### Invariante 2

Un hook observacional no modifica decisiones.

### Invariante 3

Un restrictive hook solo puede mantener o restringir authority.

### Invariante 4

Un context enricher no reemplaza atributos authoritative sin autorización contractual.

### Invariante 5

Hooks non-bypassable no pueden eliminarse por plugins normales.

---

# 292. Extension Invariants

### Invariante 1

Todo extension point tiene nivel de estabilidad y trust.

### Invariante 2

Plugins no reciben acceso indiscriminado al Container.

### Invariante 3

Extension ordering es determinista.

### Invariante 4

Conflictos y ciclos se detectan durante compilación cuando sea posible.

### Invariante 5

Cambios de extensiones invalidan los planes/cache afectados.

---

# 293. Decision Invariants

### Invariante 1

`DENY` no puede convertirse en `ALLOW` mediante extensiones no-Core.

### Invariante 2

`CHALLENGE` no puede convertirse en `ALLOW` sin satisfacer requirements y reautorizar.

### Invariante 3

Candidate `ALLOW` no es final hasta completar evaluadores mandatory.

### Invariante 4

Cached `ALLOW` tampoco es final si existen runtime restrictions mandatory.

### Invariante 5

`FAILURE` nunca se interpreta como `ALLOW`.

---

# 294. Persistent Runtime Invariants

### Invariante 1

Evaluation stacks son request-scoped.

### Invariante 2

Hooks no mantienen estado mutable global.

### Invariante 3

Nested authorization restaura correctamente el contexto.

### Invariante 4

Request reset elimina extensión temporal.

### Invariante 5

FrankenPHP worker reuse no comparte lifecycle state entre requests.

---

# 295. Testing Strategy

El subsistema deberá probar:

```text
phase ordering
extension ordering
event immutability
hook capability enforcement
restriction monotonicity
non-bypassable evaluators
extension failure policies
recursion handling
nested authorization
cleanup
persistent workers
```

---

# 296. Event Immutability Test

Listener intenta modificar:

```text
AuthorizationDenied
```

Debe ser imposible por contrato.

---

# 297. Restriction Property

Para cualquier Restrictive Hook:

```text
input = DENY
```

salida nunca puede ser:

```text
ALLOW
```

---

# 298. Challenge Property

```text
input = CHALLENGE
```

una extensión no-Core no puede devolver directamente:

```text
ALLOW
```

---

# 299. Mandatory Evaluator Test

Registrar:

```text
EarlyAllowCandidate
```

Tenant mismatch todavía debe producir:

```text
DENY
```

---

# 300. Event Failure Test

Telemetry listener lanza exception.

Si es optional:

```text
Authorization result unchanged
```

---

# 301. Mandatory Extension Failure Test

Security restriction provider unavailable.

Resultado:

```text
FAILURE / DENY
```

según policy, nunca ALLOW.

---

# 302. Ordering Test

Dadas prioridades y dependencies, el compiled lifecycle debe ser estable.

---

# 303. Cycle Test

```text
Hook A after B
Hook B after A
```

Compilation:

```text
FAIL
```

---

# 304. Nested Evaluation Test

Evaluation A inicia B.

B termina.

A recupera exactamente su contexto anterior.

---

# 305. Recursion Test

A → B → A sin terminación.

Resultado:

```text
authorization.recursion_detected
```

---

# 306. Cleanup Test

Hook lanza exception.

`finally` debe limpiar:

```text
evaluation stack
temporary hook state
scratch state
```

---

# 307. Worker Leakage Test

Request A activa temporary restriction.

Request B en mismo worker no la observa.

---

# 308. Cache Hook Version Test

Cambio en una restricción que afecta decisiones deberá invalidar el cache semánticamente relacionado.

---

# 309. Property-Based Test

Agregar un restrictive hook jamás debe incrementar el conjunto de operaciones autorizadas.

---

# 310. Property-Based Test

Remover un observational hook jamás debe cambiar la semántica de autorización.

---

# 311. Property-Based Test

Reordenar listeners observacionales no debe modificar la decisión final.

---

# 312. Property-Based Test

Un event handler no debe poder alterar:

```text
Principal
Actor
Tenant
Ability
Subject
```

de una evaluación ya validada.

---

# 313. Example — Simple Policy Evaluation

```text
Authorization::check(document.update)
        ↓
AuthorizationInvoked
        ↓
Request Validated
        ↓
Context Resolved
        ↓
Policy Resolved
        ↓
DocumentPolicy::update
        ↓
GRANT candidate
        ↓
Mandatory Security Tail
        ↓
ALLOW
        ↓
AuthorizationGranted
        ↓
Audit / Telemetry
```

---

# 314. Example — Restrictive Hook

Policy:

```text
ALLOW
```

Plugin:

```text
ComplianceRestriction
```

detecta:

```text
resource under legal hold
```

y produce:

```text
DENY
```

Final:

```text
DENY
```

válido porque restringe.

---

# 315. Example — Illegal Escalation

Policy:

```text
DENY
```

Plugin intenta:

```text
ALLOW
```

Lifecycle Contract Validator:

```text
Extension Contract Violation
```

Final:

```text
FAILURE / DENY
```

según modo.

---

# 316. Example — Challenge Extension

Policy:

```text
ALLOW candidate
```

Risk extension:

```text
Requires Strong Authentication
```

Resultado:

```text
CHALLENGE
```

---

# 317. Example — Challenge Reauthorization

Usuario completa step-up.

Nueva evaluación:

```text
Policy → ALLOW
Risk Requirement → satisfied
Mandatory Tail → satisfied
```

Final:

```text
ALLOW
```

---

# 318. Example — Event Observer

`AuthorizationGranted` listener:

```text
MetricsCollector
```

incrementa:

```text
authorization.granted.total
```

No forma parte de la semántica.

---

# 319. Example — Required Audit

Ability:

```text
production.deploy
```

exige durable audit.

El audit no se implementa como simple observador opcional.

Pipeline:

```text
Authorization
    ↓
RequiredAuditPrecondition
    ↓
Audit sink available?
    ↓
YES → continue
NO → FAILURE
```

---

# 320. Example — Policy Provider Extension

Package registra:

```text
ExternalPolicyProvider
```

para recursos concretos.

El compiler resuelve:

```text
subject type
→ provider
```

antes del runtime.

---

# 321. Example — Tenant Restriction Extension

Tenant A configura:

```text
No exports outside business hours
```

La extensión:

```text
BusinessHoursRestriction
```

solo se agrega al compiled/runtime plan de ese Tenant profile.

---

# 322. Example — Nested Authorization

`document.publish` Policy solicita:

```text
document.update
```

Stack:

```text
Evaluation A: publish
    ↓
Evaluation B: update
    ↓
ALLOW
    ↑
Evaluation A continues
```

Contextos siguen aislados.

---

# 323. Example — Cache Hit

Cache:

```text
ALLOW document.view
```

Pero current security tail detecta:

```text
Tenant suspended
```

Final:

```text
DENY
```

El cache nunca omite mandatory runtime constraints.

---

# 324. Example — Plugin Timeout

External compliance restriction:

```text
timeout
```

Policy:

```text
mandatory restriction provider
```

Resultado:

```text
FAILURE
```

No:

```text
ALLOW
```

---

# 325. Arquitectura final del lifecycle

```text
                   PUBLIC AUTHORIZATION API
                            │
                            ↓
                 AuthorizationManager
                            │
                            ↓
                  AuthorizationRequest
                            │
                            ↓
                  LIFECYCLE PIPELINE
                            │
       ┌────────────────────┼────────────────────┐
       ↓                    ↓                    ↓
     Hooks               Evaluators           Events
       │                    │                    │
 controlled            authority /         observation
 participation         restrictions        integration
       │                    │                    │
       └────────────────────┼────────────────────┘
                            ↓
                    Candidate Decisions
                            │
                            ↓
                    Decision Strategy
                            │
                            ↓
                 Mandatory Security Tail
                            │
                            ↓
                  Normalized Decision
                            │
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
           Audit        Explanation    Telemetry
              │             │             │
              └─────────────┼─────────────┘
                            ↓
                         RESULT
```

---

# 326. Filosofía arquitectónica

VoltStack deberá mantener estas reglas:

```text
Lifecycle is explicit.

Events observe facts.

Hooks participate through constrained contracts.

Evaluators decide authority.

Plugins do not receive implicit superpowers.

Extensions may restrict authority,
but may not silently broaden it.

Security-critical evaluators are non-bypassable.

Candidate ALLOW is never automatically final.

Cached decisions still respect mandatory runtime security.

Event listeners are not security enforcement.

Side effects do not belong inside ordinary authorization checks.

Every nested evaluation is isolated.

Every evaluation is cleaned up.

Every request is isolated.

Persistent workers retain structure,
not mutable authorization state.
```

---

# 327. Resultado esperado

Con `26_AUTHORIZATION_LIFECYCLE_EVENTS_HOOKS_AND_EXTENSION_POINTS_SYSTEM.md`, VoltStack podrá ofrecer un Authorization System donde paquetes y aplicaciones puedan agregar:

```text
custom evaluators
context providers
risk rules
audit integrations
telemetry
policy providers
tenant restrictions
compliance rules
challenge requirements
diagnostics
```

sin convertir el motor de autorización en un sistema de callbacks donde cualquier extensión pueda hacer:

```text
return true;
```

y saltarse la seguridad.

La regla definitiva será:

> **VoltStack deberá ser extensible en cada fase importante de Authorization, pero esa extensibilidad estará gobernada por contratos explícitos, capabilities limitadas, orden determinista y reglas monotónicas de seguridad.**

De esta manera:

```text
EXTENSIBILITY
+
DETERMINISM
+
NON-BYPASSABLE SECURITY
+
OBSERVABILITY
+
LIFECYCLE ISOLATION
=
SAFE AUTHORIZATION EXTENSION MODEL
```

El siguiente documento de la secuencia es **`27_AUTHORIZATION_STATE_CONSISTENCY_CONCURRENCY_AND_DISTRIBUTED_COORDINATION_SYSTEM.md`**, donde corresponde definir race conditions, versiones de estado, revocación concurrente, TOCTOU, consistencia entre nodos, locks, compare-and-swap, workers persistentes, decisiones distribuidas e invalidación coordinada.
