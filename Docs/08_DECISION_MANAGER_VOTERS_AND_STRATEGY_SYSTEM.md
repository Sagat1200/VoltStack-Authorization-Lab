# VoltStack Authorization System — Decision Manager, Voters and Strategy System

## 1. Propósito

Este documento define el subsistema encargado de **agregar, interpretar y resolver múltiples decisiones de autorización** dentro de VoltStack.

El Authorization Engine podrá recibir resultados desde:

```text
Policies
Gates
Global Security Evaluators
Tenant Evaluators
Role Evaluators
Permission Evaluators
Context Evaluators
External Evaluators
```

Cada uno podrá producir:

```text
GRANT
DENY
ABSTAIN
```

El `DecisionManager` será responsable de convertir estos resultados parciales en una única decisión final.

Ejemplo:

```text
TenantIsolationPolicy  → GRANT
MfaSecurityGate        → GRANT
InvoicePolicy          → GRANT
CompliancePolicy       → DENY
```

El sistema deberá responder:

```text
¿cuál es el resultado final?
```

La respuesta dependerá de una:

```text
DecisionStrategy
```

---

# 2. Principio arquitectónico

La arquitectura deberá separar claramente:

```text
Evaluator
    ↓
produce individual decision

DecisionStrategy
    ↓
defines combination semantics

DecisionManager
    ↓
coordinates aggregation

ResultFinalizer
    ↓
applies final safety rules
```

Por tanto:

```text
Policy ≠ Decision Strategy
Gate ≠ Decision Strategy
DecisionManager ≠ Policy Resolver
```

---

# 3. Modelo general

```text
AuthorizationRequest
        ↓
AuthorizationPlan
        ↓
Evaluator 1 → DecisionResult
Evaluator 2 → DecisionResult
Evaluator 3 → DecisionResult
Evaluator N → DecisionResult
        ↓
DecisionManager
        ↓
DecisionStrategy
        ↓
Final DecisionResult
```

---

# 4. Influencia conceptual de Voters

VoltStack adoptará la idea de **Voters** como modelo conceptual para cualquier componente que pueda opinar sobre una solicitud de autorización.

Un Voter responde:

```text
GRANT
DENY
ABSTAIN
```

VoltStack no necesitará obligatoriamente que todas las implementaciones públicas se llamen `Voter`.

Podrán seguir existiendo:

```text
Policy
Gate
SecurityEvaluator
TenantEvaluator
PermissionEvaluator
```

Internamente todos podrán adaptarse a:

```text
AuthorizationVoter
```

---

# 5. AuthorizationVoter

Contrato conceptual:

```php
interface AuthorizationVoterInterface
{
    public function vote(
        AuthorizationRequest $request,
    ): DecisionResult;
}
```

Esto proporciona una abstracción común.

---

# 6. Voter como abstracción interna

La relación conceptual será:

```text
Policy
   ↓ adapter
AuthorizationVoter

Gate
   ↓ adapter
AuthorizationVoter

Global Evaluator
   ↓
AuthorizationVoter
```

Todos producen el mismo resultado normalizado.

---

# 7. Decision enum

El estado fundamental será:

```php
enum Decision: string
{
    case Grant = 'grant';
    case Deny = 'deny';
    case Abstain = 'abstain';
}
```

---

# 8. Semántica de GRANT

`GRANT` significa:

```text
Este evaluator considera que la operación
debe ser autorizada.
```

No significa necesariamente:

```text
Final Authorization = GRANT
```

si existen otros evaluadores.

---

# 9. Semántica de DENY

`DENY` significa:

```text
Este evaluator considera que la operación
debe ser rechazada.
```

Su impacto final dependerá de la estrategia.

---

# 10. Semántica de ABSTAIN

`ABSTAIN` significa:

```text
Este evaluator no toma posición
sobre esta solicitud.
```

No significa:

```text
GRANT
```

ni:

```text
DENY
```

---

# 11. Importancia de ABSTAIN

Sin `ABSTAIN`, cualquier evaluator que no aplicara tendría que devolver:

```text
false
```

lo que equivaldría a:

```text
DENY
```

y haría imposible una composición correcta.

---

# 12. Ejemplo de abstención

```text
MfaPolicy

Request:
Article view

MFA no es relevante
        ↓
ABSTAIN
```

No debe negar una operación que simplemente no le corresponde evaluar.

---

# 13. DecisionResult

Cada voto utilizará:

```php
final readonly class DecisionResult
{
    public function __construct(
        public Decision $decision,
        public ?string $reasonCode = null,
        public ?string $reason = null,
        public array $metadata = [],
    ) {}
}
```

---

# 14. DecisionManager

Contrato conceptual:

```php
interface DecisionManagerInterface
{
    public function decide(
        AuthorizationRequest $request,
        AuthorizationExecution $execution,
        DecisionStrategyInterface $strategy,
    ): DecisionResult;
}
```

---

# 15. Responsabilidades del DecisionManager

Deberá:

1. recibir resultados de evaluadores;
2. aplicar la estrategia;
3. respetar short-circuit;
4. diferenciar abstenciones;
5. identificar fallos de ejecución;
6. producir un resultado agregado;
7. conservar metadata relevante;
8. aplicar semántica determinista.

---

# 16. Lo que no debe hacer

No deberá:

```text
resolver Policies
resolver Gates
instanciar evaluadores
autenticar
resolver Subjects
hacer model binding
traducir respuestas HTTP
```

---

# 17. DecisionStrategyInterface

Contrato conceptual:

```php
interface DecisionStrategyInterface
{
    public function decide(
        iterable $votes,
        DecisionStrategyContext $context,
    ): DecisionResult;
}
```

---

# 18. DecisionStrategyContext

Podrá contener:

```text
AuthorizationRequest
strategy configuration
default decision
short-circuit rules
priority metadata
```

No deberá convertirse en un service locator.

---

# 19. Estrategias iniciales

VoltStack deberá soportar al menos:

```text
Unanimous
Affirmative
Consensus
DenyOverrides
AllowOverrides
FirstApplicable
Priority
```

---

# 20. Default Strategy

La estrategia global deberá ser configurable.

Para seguridad general se recomienda:

```text
DenyOverrides
```

o una variante estricta de:

```text
Unanimous
```

según el tipo de evaluadores involucrados.

---

# 21. Unanimous Strategy

Semántica:

```text
Ningún evaluator aplicable puede DENY.
```

Ejemplo:

```text
GRANT
GRANT
ABSTAIN
GRANT
   ↓
GRANT
```

Pero:

```text
GRANT
DENY
GRANT
   ↓
DENY
```

---

# 22. Unanimous y todos ABSTAIN

Si:

```text
ABSTAIN
ABSTAIN
ABSTAIN
```

resultado:

```text
Default Deny
```

No:

```text
GRANT
```

---

# 23. Unanimous formal

Conceptualmente:

```text
if any DENY:
    DENY

else if any GRANT:
    GRANT

else:
    ABSTAIN
```

Luego el finalizador convierte un `ABSTAIN` final según política global.

---

# 24. Affirmative Strategy

Semántica básica:

```text
Un GRANT puede ser suficiente.
```

Ejemplo:

```text
ABSTAIN
GRANT
ABSTAIN
   ↓
GRANT
```

---

# 25. Affirmative con DENY

Debe definirse explícitamente.

Dos variantes posibles:

```text
GrantWins
```

o:

```text
GrantUnlessCriticalDeny
```

VoltStack no deberá ocultar esta diferencia.

---

# 26. Recomendación

No utilizar `Affirmative` como estrategia global de seguridad sin configuración adicional.

Puede ser útil para:

```text
alternative authorization paths
feature access
multiple equivalent credentials
```

---

# 27. Consensus Strategy

Semántica:

```text
La mayoría de evaluadores decisivos determina el resultado.
```

Ejemplo:

```text
GRANT = 4
DENY  = 2
ABSTAIN = 3
```

Resultado:

```text
GRANT
```

---

# 28. ABSTAIN en Consensus

Las abstenciones no deberán contar como votos positivos ni negativos.

---

# 29. Empates

Ejemplo:

```text
GRANT = 2
DENY  = 2
```

La política de empate deberá ser configurable.

Recomendación:

```text
tie = DENY
```

---

# 30. Consensus Use Cases

Adecuado para escenarios como:

```text
multi-source risk assessment
multiple optional trust signals
federated policy evaluation
```

No necesariamente para aislamiento multi-tenant.

---

# 31. DenyOverrides Strategy

Semántica:

```text
Cualquier DENY aplicable prevalece.
```

Ejemplo:

```text
GRANT
GRANT
DENY
   ↓
DENY
```

---

# 32. DenyOverrides formal

```text
if any DENY:
    DENY

else if any GRANT:
    GRANT

else:
    ABSTAIN
```

Parece similar a `Unanimous`, pero podrá diferenciarse en cómo trata:

```text
mandatory voters
optional voters
priority
terminal results
```

---

# 33. Uso recomendado

Ideal para:

```text
TenantIsolation
SuspendedPrincipal
MFA Requirements
Compliance
Security Policies
```

---

# 34. AllowOverrides Strategy

Semántica:

```text
Cualquier GRANT aplicable prevalece.
```

Ejemplo:

```text
DENY
ABSTAIN
GRANT
   ↓
GRANT
```

---

# 35. Riesgo

`AllowOverrides` puede ser peligroso si un evaluator demasiado permisivo puede ignorar una restricción crítica.

Debe utilizarse explícitamente.

---

# 36. Use cases

Puede ser útil para:

```text
break-glass access
super-admin override
multiple alternate authorization paths
```

siempre con controles estrictos.

---

# 37. FirstApplicable Strategy

Semántica:

```text
Usar el primer resultado que no sea ABSTAIN.
```

Ejemplo:

```text
Policy A → ABSTAIN
Policy B → ABSTAIN
Policy C → GRANT
Policy D → DENY
```

Resultado:

```text
GRANT
```

Policy D no necesita ejecutarse.

---

# 38. Importancia del ordering

En `FirstApplicable`:

```text
priority
```

es parte esencial de la semántica.

---

# 39. Priority Strategy

Cada evaluator posee:

```text
priority
```

Se ejecutan de mayor a menor.

La primera decisión terminal relevante puede determinar el resultado.

---

# 40. Ejemplo Priority

```text
TenantIsolationPolicy  1000 → DENY
InvoicePolicy           500 → GRANT
```

Resultado:

```text
DENY
```

y el segundo puede no ejecutarse.

---

# 41. Priority no significa automáticamente FirstApplicable

Puede existir una estrategia que:

```text
ejecute todos en orden de prioridad
```

pero aún agregue todos los resultados.

Por tanto:

```text
priority
```

y:

```text
aggregation strategy
```

son conceptos relacionados pero distintos.

---

# 42. Strategy Composition

VoltStack podrá permitir estrategias compuestas.

Ejemplo:

```text
CriticalDenyOverrides
        +
Consensus for remaining votes
```

Esto será útil en sistemas complejos.

---

# 43. Mandatory Evaluators

Algunos evaluadores podrán declararse:

```text
mandatory
```

Ejemplo:

```text
TenantIsolationPolicy
MfaRequirementPolicy
```

Si un mandatory evaluator falla o no puede decidir, el sistema podrá denegar.

---

# 44. Optional Evaluators

Otros pueden ser:

```text
optional
```

Ejemplo:

```text
RecommendationRiskEvaluator
```

Una abstención es normal.

---

# 45. Mandatory ABSTAIN

Debe poder configurarse como:

```text
mandatory evaluator abstains
        ↓
DENY
```

o:

```text
configuration failure
```

según tipo de Policy.

---

# 46. EvaluatorRequirement

Conceptualmente:

```php
enum EvaluatorRequirement: string
{
    case Optional = 'optional';
    case Required = 'required';
    case Critical = 'critical';
}
```

---

# 47. Critical Evaluator

Un evaluator crítico podrá tener reglas como:

```text
failure → DENY
DENY → terminal
ABSTAIN → configuration error
```

Ejemplo:

```text
TenantIsolation
```

---

# 48. Strategy Descriptor

Podrá existir:

```php
final readonly class DecisionStrategyDescriptor
{
    public function __construct(
        public string $strategyClass,
        public array $options = [],
    ) {}
}
```

---

# 49. Strategy Registry

VoltStack podrá mantener:

```text
DecisionStrategyRegistry
```

Ejemplo:

```text
unanimous
affirmative
consensus
deny_overrides
allow_overrides
first_applicable
priority
```

---

# 50. Custom Strategies

Las aplicaciones podrán registrar:

```php
Authorization::strategy(
    'financial-approval',
    FinancialApprovalStrategy::class,
);
```

---

# 51. Strategy selection

La estrategia podrá definirse en:

```text
global authorization config
Ability metadata
Route metadata
Controller metadata
Policy plan metadata
```

---

# 52. Precedencia de estrategia

Recomendación:

```text
Explicit Authorization Request
        ↓
Controller/Route Metadata
        ↓
Ability Descriptor
        ↓
Subsystem Default
        ↓
Global Default
```

---

# 53. Strategy per Ability

Ejemplo:

```text
invoice.view
    → affirmative

invoice.approve
    → unanimous

system.deploy
    → deny_overrides
```

---

# 54. Strategy attribute

Podrá declararse:

```php
#[DecisionStrategy('unanimous')]
#[Authorize('approve', subject: 'invoice')]
public function approve(Invoice $invoice)
{
}
```

---

# 55. Compiled strategy metadata

La selección deberá compilarse para evitar resolver strings y clases repetidamente.

---

# 56. AuthorizationPlan

El plan deberá contener:

```text
evaluators
priority
requirement
strategy
short-circuit metadata
default decision
```

---

# 57. Example plan

```text
AuthorizationPlan

Strategy:
DenyOverrides

Evaluators:
1. SuspendedPrincipalPolicy
   priority=1000
   requirement=critical

2. TenantIsolationPolicy
   priority=900
   requirement=critical

3. InvoicePolicy
   priority=500
   requirement=required

4. CompliancePolicy
   priority=400
   requirement=required
```

---

# 58. EvaluatorResult

Cada ejecución debería distinguir entre:

```text
semantic decision
```

y:

```text
execution status
```

---

# 59. Execution Status

Conceptualmente:

```php
enum EvaluatorExecutionStatus
{
    case Completed;
    case Failed;
    case Skipped;
}
```

---

# 60. Completed + GRANT

```text
execution=COMPLETED
decision=GRANT
```

---

# 61. Completed + DENY

```text
execution=COMPLETED
decision=DENY
```

---

# 62. Completed + ABSTAIN

```text
execution=COMPLETED
decision=ABSTAIN
```

---

# 63. Failed

No existe un voto semántico válido.

```text
execution=FAILED
```

La estrategia/failure policy determinará cómo manejarlo.

---

# 64. Skipped

Puede ocurrir por:

```text
short-circuit
cancellation
plan optimization
```

No equivale a `ABSTAIN`.

---

# 65. DecisionVote

Podrá existir:

```php
final readonly class DecisionVote
{
    public function __construct(
        public string $evaluatorId,
        public EvaluatorExecutionStatus $status,
        public ?DecisionResult $result,
        public int $priority,
        public EvaluatorRequirement $requirement,
    ) {}
}
```

---

# 66. DecisionManager input

Así el manager puede conocer:

```text
who voted
what they decided
whether execution failed
whether voter was required
priority
```

---

# 67. Failure Policy

Debe distinguirse de Decision Strategy.

```text
Decision Strategy
=
cómo combinar votos válidos
```

```text
Failure Policy
=
qué hacer cuando un evaluator falla
```

---

# 68. Default Failure Policy

Recomendación:

```text
fail closed
```

Para evaluator requerido/crítico:

```text
failure
  ↓
DENY / authorization system failure
```

---

# 69. Optional evaluator failure

Podrá configurarse:

```text
log + ABSTAIN
```

solo si ese evaluator fue explícitamente marcado como no crítico.

---

# 70. Fail-closed invariant

Un fallo nunca podrá transformarse automáticamente en:

```text
GRANT
```

---

# 71. Failure Result

Internamente podrá generarse:

```text
authorization.evaluator_failure
```

como razón del `DENY` final.

---

# 72. No leakage

La razón externa no deberá exponer:

```text
database error
stack trace
service credentials
internal class names
```

---

# 73. Short-Circuit

El Decision Manager deberá colaborar con el Executor para detener evaluaciones cuando el resultado ya sea irreversible.

---

# 74. Short-Circuit Interface

Una estrategia podrá responder:

```php
public function shouldContinue(
    DecisionAccumulator $state,
    DecisionVote $latestVote,
): bool;
```

---

# 75. Unanimous short-circuit

Ante:

```text
DENY
```

puede detenerse inmediatamente.

---

# 76. DenyOverrides short-circuit

Igualmente:

```text
first DENY
  ↓
STOP
```

si no existe Allow override de prioridad superior pendiente.

---

# 77. Affirmative short-circuit

Podrá detenerse ante:

```text
GRANT
```

si la estrategia garantiza que futuros `DENY` no pueden cambiar el resultado.

---

# 78. Consensus short-circuit

Más complejo.

Solo podrá detenerse si:

```text
remaining evaluators cannot change majority
```

Ejemplo:

```text
GRANT=7
DENY=1
remaining=2
```

ya no puede perder GRANT.

---

# 79. FirstApplicable short-circuit

Siempre:

```text
first non-ABSTAIN
 ↓
STOP
```

---

# 80. Priority short-circuit

Dependerá de:

```text
terminal metadata
priority ranges
strategy configuration
```

---

# 81. Terminal Evaluator

Un evaluator puede declararse:

```text
terminal_on_deny
terminal_on_grant
```

pero la estrategia debe aprobar ese comportamiento.

---

# 82. No evaluator-owned global authority

Un evaluator no deberá poder decir por sí solo:

```text
stop entire authorization
```

fuera de metadata y estrategia.

Esto evita lógica oculta.

---

# 83. Critical Deny

Puede representarse mediante:

```text
requirement=critical
decision=DENY
```

La estrategia puede definir:

```text
critical deny always terminates
```

---

# 84. Critical Grant

Debe utilizarse con mucho cuidado.

Ejemplo:

```text
break-glass super-admin
```

podría producir:

```text
critical grant
```

pero deberá estar explícitamente configurado.

---

# 85. Super-admin strategy

Recomendación:

```text
SuperAdminOverrideEvaluator
```

con prioridad alta y una estrategia explícita de override.

No hardcodearlo en `DecisionManager`.

---

# 86. Default Deny

Después de la estrategia, puede quedar:

```text
ABSTAIN
```

Ejemplo:

```text
no voters
```

o:

```text
all abstain
```

La política global será:

```text
ABSTAIN final
     ↓
DENY
```

---

# 87. DefaultDecisionPolicy

Podrá existir:

```php
enum DefaultDecisionPolicy
{
    case Deny;
    case Grant;
}
```

Pero:

```text
Grant
```

deberá desaconsejarse fuertemente.

---

# 88. Default recomendado

```text
DefaultDecisionPolicy::Deny
```

---

# 89. Strict No-Voter Mode

Podrá existir:

```text
no evaluator applies
      ↓
AuthorizationConfigurationException
```

en desarrollo.

En producción:

```text
DENY
```

---

# 90. No-voter diagnostics

Ejemplo:

```text
Ability:
invoice.archive

Subject:
Invoice

No evaluator handled this request.
```

---

# 91. All Abstain diagnostics

Distinto:

```text
Evaluators resolved:
3

All returned ABSTAIN.
```

Esto puede indicar un error de Policy o un caso legítimo.

---

# 92. DecisionAccumulator

Para estrategias incrementales podrá existir:

```php
final class DecisionAccumulator
{
    public int $grants = 0;
    public int $denies = 0;
    public int $abstains = 0;
}
```

Más metadata si se necesita.

---

# 93. Inmutabilidad vs performance

El acumulador puede ser mutable internamente porque vive únicamente dentro de una decisión.

No debe escapar del DecisionManager.

---

# 94. DecisionSummary

El resultado final podrá incluir un resumen:

```text
grantCount
denyCount
abstainCount
failedCount
skippedCount
```

solo cuando observabilidad esté habilitada.

---

# 95. DecisionReason selection

Si múltiples evaluadores DENY, ¿qué razón se devuelve?

VoltStack deberá definir una política.

---

# 96. Primary Denial Reason

Recomendación:

```text
highest-priority decisive DENY
```

como razón primaria.

---

# 97. Additional reasons

Tracing podrá conservar:

```text
all denial reasons
```

sin exponerlas al usuario.

---

# 98. Example

```text
TenantIsolationPolicy
DENY
reason=tenant.mismatch

CompliancePolicy
DENY
reason=compliance.approval_required
```

Razón primaria:

```text
tenant.mismatch
```

si tiene mayor prioridad.

---

# 99. Grant reason

Normalmente no será necesario mostrar una razón de GRANT.

Pero puede existir para auditoría.

---

# 100. ReasonStrategy

Podrá existir internamente:

```text
PrimaryReasonSelector
```

separado de la estrategia de decisión.

---

# 101. Why separation matters

Una estrategia determina:

```text
GRANT vs DENY
```

El selector de razones determina:

```text
qué explicación usar
```

---

# 102. DecisionMetadata

El resultado agregado podrá contener:

```text
strategy
decisiveEvaluator
counts
traceId
```

sin incorporar todo el execution graph.

---

# 103. Final DecisionResult

Ejemplo:

```php
DecisionResult::deny(
    reasonCode: 'tenant.mismatch',
    metadata: [
        'strategy' => 'deny_overrides',
        'decisive_evaluator' => 'tenant_isolation',
    ],
);
```

---

# 104. Internal vs Public Metadata

Metadata técnica deberá filtrarse antes de exponer respuestas.

---

# 105. Decision Strategy Registry

Conceptualmente:

```php
interface DecisionStrategyRegistryInterface
{
    public function get(
        string $name
    ): DecisionStrategyInterface;
}
```

---

# 106. Immutable Strategy Registry

Podrá compartirse entre requests.

Las estrategias deberán ser:

```text
stateless
```

---

# 107. Stateful strategies

Si una estrategia necesita estado, deberá mantenerlo en un objeto de decisión local, no en la instancia compartida.

---

# 108. FrankenPHP Safety

Nunca guardar:

```text
latest votes
current user
current request
current decision
```

en una estrategia singleton.

---

# 109. Strategy Context

Toda información dinámica deberá recibirse mediante argumentos.

---

# 110. Strategy Configuration

Configuración conceptual:

```php
return [
    'decision' => [
        'default_strategy' => 'deny_overrides',
        'default_decision' => 'deny',
        'consensus_tie' => 'deny',
    ],
];
```

---

# 111. Ability-specific config

```php
'strategies' => [
    'system.deploy' => 'unanimous',
    'admin.access' => 'deny_overrides',
]
```

---

# 112. Policy metadata override

Una operación crítica podrá declararse:

```php
#[DecisionStrategy('unanimous')]
```

---

# 113. Controller strategy

Ejemplo:

```php
#[Authorize(
    ability: 'approve',
    subject: 'invoice',
    strategy: 'unanimous',
)]
public function approve(Invoice $invoice)
{
}
```

---

# 114. Strategy Validation

Toda estrategia referenciada deberá existir durante compilación.

Si no:

```text
UnknownDecisionStrategyException
```

---

# 115. Strategy aliases

Podrán existir aliases:

```text
strict
  ↓
deny_overrides
```

pero deben compilarse igual que Ability aliases.

---

# 116. Built-in Strategy IDs

Recomendación:

```text
unanimous
affirmative
consensus
deny_overrides
allow_overrides
first_applicable
priority
```

---

# 117. Custom strategy contract

Ejemplo:

```php
final class FinancialApprovalStrategy
    implements DecisionStrategyInterface
{
    public function decide(
        iterable $votes,
        DecisionStrategyContext $context,
    ): DecisionResult {
        // ...
    }
}
```

---

# 118. Custom strategy security

Las custom strategies poseen autoridad crítica.

Deberán provenir exclusivamente de código confiable.

---

# 119. No database class names

No almacenar:

```text
strategy_class
```

arbitraria desde input o DB no confiable.

---

# 120. Strategy Decorators

Podrán existir:

```text
TracingDecisionStrategy
```

pero no se recomienda una pila pesada en el hot path.

---

# 121. DecisionManager Observability

Tracing podrá registrar:

```text
strategy
vote sequence
short-circuit point
decisive evaluator
final decision
duration
```

---

# 122. Example trace

```text
Authorization Decision

Strategy:
DenyOverrides

Votes:

SuspendedPrincipalPolicy
ABSTAIN

TenantIsolationPolicy
GRANT

InvoicePolicy
GRANT

CompliancePolicy
DENY

Short Circuit:
yes

Decisive Evaluator:
CompliancePolicy

Final:
DENY
```

---

# 123. Voter explanation

Cada evaluator podrá proporcionar:

```text
reasonCode
reason
metadata
```

El DecisionManager no deberá reescribir semánticamente estas razones salvo para seleccionar la principal.

---

# 124. Explainability Tree

Para nested authorization:

```text
InvoicePolicy
  ↓ nested check
CustomerPolicy
```

el trace podrá formar un árbol.

Esto pertenece a observabilidad, no a la estrategia.

---

# 125. Decision audit

Para operaciones críticas podrá persistirse:

```text
principal
ability
subject
strategy
final decision
decisive evaluator
reason code
timestamp
```

---

# 126. Audit of all votes

No deberá persistirse siempre.

Podrá habilitarse para:

```text
critical operations
compliance
security investigations
```

---

# 127. Decision Metrics

Ejemplos:

```text
authorization.decisions.total
authorization.decisions.granted
authorization.decisions.denied
authorization.decisions.abstained
authorization.voters.failed
authorization.short_circuit.count
```

---

# 128. Strategy Metrics

```text
authorization.strategy.unanimous.total
authorization.strategy.deny_overrides.total
```

---

# 129. Performance Considerations

El DecisionManager está en el hot path.

Debe evitar:

```text
unnecessary allocations
sorting if plan already sorted
building full traces when disabled
copying large metadata arrays
```

---

# 130. Streaming Votes

Idealmente el manager podrá consumir votos secuencialmente:

```text
vote
 ↓
update accumulator
 ↓
should continue?
```

sin esperar a construir un array completo.

---

# 131. Advantage

Esto permite short-circuit real.

No:

```text
execute all voters
then decide
```

cuando no es necesario.

---

# 132. AuthorizationExecutor integration

Arquitectura recomendada:

```text
DecisionManager
    ↓
requests next evaluator result
    ↓
Executor dispatches evaluator
    ↓
DecisionManager updates state
```

o un coordinador común.

---

# 133. Alternative architecture

El Executor puede ejecutar y entregar votos progresivamente mediante:

```text
Iterator<DecisionVote>
```

El DecisionManager consume el iterator.

---

# 134. Recommended V1

```text
Evaluator iterator
        ↓
DecisionManager
        ↓
strategy accumulator
```

Esto mantiene buena separación.

---

# 135. Pre-sorted Evaluators

El AuthorizationPlan deberá llegar ya ordenado.

DecisionManager no debe ordenar Policies por sí mismo.

---

# 136. Strategy-specific ordering

Si una estrategia necesita orden particular, el Planner deberá conocerlo.

---

# 137. Critical voters first

Para rendimiento:

```text
TenantIsolation
SuspendedAccount
```

pueden ejecutarse antes que Policies costosas.

---

# 138. Security and performance alignment

En muchos casos:

```text
high-priority security deny
```

también permite:

```text
early exit
```

---

# 139. Consensus ordering

Aunque Consensus necesita potencialmente más votos, evaluadores baratos pueden ejecutarse primero.

---

# 140. Cost Metadata

En versiones futuras un evaluator podrá declarar:

```text
estimated_cost
```

para optimización.

No deberá alterar la semántica.

---

# 141. No semantic reordering

El Planner nunca deberá reordenar evaluadores de forma que cambie el significado de estrategias sensibles al orden.

---

# 142. Determinism

Con:

```text
same AuthorizationPlan
same evaluator outputs
same strategy
```

el resultado deberá ser idéntico.

---

# 143. No random tie-breaking

Empates nunca deberán resolverse aleatoriamente.

---

# 144. Consensus tie default

```text
DENY
```

por seguridad.

---

# 145. Priority tie

Si dos evaluadores tienen igual prioridad, utilizar orden estable compilado.

---

# 146. FirstApplicable tie

Igualmente depende del orden estable.

---

# 147. Decision Stability

La metadata observacional no deberá modificar el resultado.

---

# 148. Runtime errors

Si tracing falla:

```text
authorization decision
```

no deberá cambiar salvo que tracing sea una obligación de compliance explícita.

---

# 149. Audit-critical operations

En una operación configurada como:

```text
must_audit=true
```

si auditoría obligatoria falla, la política puede ser:

```text
DENY
```

pero esto debe modelarse como Security/Compliance Evaluator, no como efecto lateral oculto del DecisionManager.

---

# 150. External Voters

VoltStack podrá soportar en el futuro:

```text
OPA
remote PDP
policy service
```

mediante:

```text
ExternalAuthorizationVoter
```

---

# 151. Remote Voter Semantics

Debe producir:

```text
GRANT
DENY
ABSTAIN
FAILED
```

distinguiendo fallo de servicio de abstención.

---

# 152. Timeout

Un timeout remoto:

```text
FAILED
```

no:

```text
ABSTAIN
```

salvo configuración explícita y segura.

---

# 153. Circuit Breaker

Evaluadores remotos podrán usar resiliencia del framework.

No será responsabilidad directa del DecisionManager.

---

# 154. Distributed Decision Versioning

Futuro:

```text
policy version
decision source
remote evaluator version
```

podrán adjuntarse a auditoría.

---

# 155. RBAC Voter

El sistema de roles/permisos podrá integrarse como:

```text
PermissionVoter
```

Ejemplo:

```text
Ability:
invoice.approve

PermissionVoter
    ↓
GRANT
```

---

# 156. Policy + Permission voter

Plan:

```text
PermissionVoter      → GRANT
TenantPolicy         → GRANT
InvoicePolicy        → DENY
```

Con `DenyOverrides`:

```text
DENY
```

Así un permiso no elimina restricciones de recurso.

---

# 157. ABAC Voter

Podrá existir:

```text
AttributeVoter
```

que evalúe reglas declarativas.

---

# 158. ReBAC Voter

Podrá existir:

```text
RelationshipVoter
```

sobre relaciones de dominio.

---

# 159. Voter Registry

No necesariamente será necesario un Registry separado si todo se normaliza durante planning.

Podrá existir un:

```text
AuthorizationEvaluatorRegistry
```

común.

---

# 160. EvaluatorDescriptor

Conceptualmente:

```php
final readonly class EvaluatorDescriptor
{
    public function __construct(
        public string $id,
        public EvaluatorType $type,
        public int $priority,
        public EvaluatorRequirement $requirement,
        public bool $terminalOnGrant = false,
        public bool $terminalOnDeny = false,
    ) {}
}
```

---

# 161. EvaluatorType

```php
enum EvaluatorType: string
{
    case Policy = 'policy';
    case Gate = 'gate';
    case Permission = 'permission';
    case Role = 'role';
    case Security = 'security';
    case Tenant = 'tenant';
    case External = 'external';
    case Custom = 'custom';
}
```

---

# 162. DecisionPlan normalization

El Planner podrá convertir:

```text
PolicyDescriptor
GateDescriptor
```

en:

```text
EvaluatorDescriptor
```

más invocation metadata específica.

---

# 163. Strategy sees normalized voters

La estrategia no debería saber:

```text
si un voto vino de Gate o Policy
```

salvo que metadata explícita lo requiera.

Esto mejora desacoplamiento.

---

# 164. Type-aware strategy

Si un sistema necesita:

```text
Security DENY always overrides
```

podrá usar metadata:

```text
requirement=critical
```

mejor que inspeccionar el tipo de clase.

---

# 165. Decision Weight

Consensus avanzado podría soportar:

```text
weights
```

Ejemplo:

```text
RiskVoter weight=2
OwnershipVoter weight=1
```

No es necesario para V1.

---

# 166. Weighted Consensus

Futuro:

```text
GRANT weight 5
DENY weight 3
```

resultado:

```text
GRANT
```

con tie safe-deny.

---

# 167. Warning

Los pesos aumentan complejidad y dificultan explicabilidad.

No deben ser parte del default.

---

# 168. Strategy inheritance

No se recomienda crear jerarquías de herencia complejas.

Preferir composición con:

```text
accumulator
failure policy
reason selector
```

---

# 169. Decision Pipeline

Podrá conceptualizarse:

```text
Votes
  ↓
Execution Failure Handling
  ↓
Strategy Aggregation
  ↓
Default Decision
  ↓
Reason Selection
  ↓
Result Finalization
```

---

# 170. Failure before Strategy

Un `FAILED` requerido puede convertirse inmediatamente en una decisión fail-closed.

---

# 171. Optional failure

Puede excluirse del conjunto de votos efectivos si la configuración lo permite.

---

# 172. Finalizer

Después del DecisionManager:

```text
AuthorizationResultFinalizer
```

podrá garantizar:

```text
no final ABSTAIN exposed as allow
reason normalization
trace id attachment
```

---

# 173. Public bool mapping

```text
GRANT → true
DENY  → false
ABSTAIN final → false
```

---

# 174. `authorize()` mapping

```text
GRANT
 ↓
continue

DENY
 ↓
AuthorizationDeniedException
```

---

# 175. Final ABSTAIN

Idealmente el finalizador lo convertirá a:

```text
DENY
reasonCode=authorization.no_decision
```

---

# 176. Authorization Denied Reason Codes

Posibles códigos agregados:

```text
authorization.no_evaluator
authorization.all_abstained
authorization.denied
authorization.evaluator_failure
authorization.strategy_tie
```

---

# 177. Preserve source reason

Si existe un DENY claro:

```text
tenant.mismatch
```

deberá preferirse sobre:

```text
authorization.denied
```

genérico.

---

# 178. Strategy-specific reason

Consensus tie:

```text
authorization.consensus_tie
```

podrá ser útil internamente.

---

# 179. Decision History

No deberá almacenarse dentro del DecisionManager singleton.

Si se necesita, se envía a:

```text
AuthorizationTrace
```

---

# 180. Decision Manager statelessness

`DecisionManager` deberá ser:

```text
stateless
```

y seguro para compartir en FrankenPHP.

---

# 181. Decision Strategy statelessness

También.

Toda acumulación vive por evaluación.

---

# 182. Parallel Evaluation

V1 deberá favorecer ejecución secuencial.

Razones:

```text
determinism
short-circuit
simple priority
easy tracing
```

---

# 183. Future Parallelism

Solo evaluadores marcados como:

```text
independent
parallel_safe
```

podrían ejecutarse en paralelo.

---

# 184. Parallel strategy risks

Puede complicar:

```text
short-circuit
ordering
cancellation
external calls
trace ordering
```

Por ello no deberá ser default.

---

# 185. Batch Decisions

El DecisionManager también deberá poder agregarse por item dentro de batch authorization.

Cada Subject tendrá su propia decisión final.

---

# 186. Batch strategy

La misma estrategia podrá reutilizarse por Subject.

No mezclar votos de Subjects diferentes.

---

# 187. Multi-subject operations

Si una operación requiere:

```text
Document
SourceFolder
TargetFolder
```

deberá existir un único AuthorizationRequest con contexto/subject model apropiado, no combinar decisions arbitrariamente fuera del plan.

---

# 188. Testing Strategies

Cada estrategia deberá tener una matriz exhaustiva.

Ejemplo Unanimous:

```text
[]                → ABSTAIN
[A]               → según A
[G,G]             → GRANT
[G,A]             → GRANT
[A,A]             → ABSTAIN
[G,D]             → DENY
[D,A]             → DENY
```

---

# 189. Legend

```text
G = GRANT
D = DENY
A = ABSTAIN
```

---

# 190. Affirmative tests

Debe cubrir:

```text
G + D
```

según configuración exacta.

No dejar semántica ambigua.

---

# 191. Consensus tests

Cubrir:

```text
majority grant
majority deny
tie
all abstain
mixed failures
```

---

# 192. Short-circuit tests

Verificar que evaluadores posteriores no se ejecutan cuando no deben.

---

# 193. Failure tests

Ejemplo:

```text
Critical evaluator → FAILED
```

resultado nunca debe ser `GRANT`.

---

# 194. Ordering tests

Verificar prioridad estable.

---

# 195. Persistent runtime tests

Ejecutar múltiples decisiones seguidas con diferentes Principals y comprobar que no existe fuga de estado.

---

# 196. Property-based Testing

Las estrategias se benefician de tests basados en propiedades.

Ejemplo:

```text
DenyOverrides:
adding another DENY can never change DENY to GRANT
```

---

# 197. Unanimous property

Agregar un `ABSTAIN` no debe alterar una decisión ya otorgada.

---

# 198. AllowOverrides property

Agregar un `GRANT` puede convertir DENY a GRANT según semántica declarada.

---

# 199. Consensus property

Permutaciones del mismo conjunto de votos deben producir el mismo resultado si la estrategia no depende del orden.

---

# 200. FirstApplicable property

Sí depende del orden.

Los tests deben reflejarlo.

---

# 201. Strategy Metadata

Cada estrategia podrá declarar:

```text
orderSensitive
supportsShortCircuit
supportsParallelism
defaultTieBehavior
```

---

# 202. DecisionStrategyCapabilities

Conceptualmente:

```php
final readonly class DecisionStrategyCapabilities
{
    public function __construct(
        public bool $orderSensitive,
        public bool $supportsShortCircuit,
        public bool $parallelSafe,
    ) {}
}
```

---

# 203. Planner optimization

El Planner podrá usar estas capacidades para optimizar ejecución.

---

# 204. No strategy introspection runtime

Estas capacidades deberán estar registradas/compiladas.

---

# 205. Strategy IDs in traces

Preferir:

```text
deny_overrides
```

sobre nombres completos de clase en telemetry externa.

---

# 206. Security configuration

Operaciones sensibles podrían forzar:

```text
DenyOverrides
```

independientemente de configuración menos estricta del módulo.

Pero debe ser una regla explícita.

---

# 207. Minimum Security Strategy

Futuro:

```text
minimum_strategy_strength
```

podría impedir usar `AllowOverrides` en ciertas abilities críticas.

No es requisito de V1.

---

# 208. Strategy Strength

Concepto potencial:

```text
permissive
balanced
strict
critical
```

Solo para tooling; no sustituir semántica explícita.

---

# 209. Tooling

Comando:

```text
volt authorization:strategy invoice.approve
```

podrá mostrar:

```text
Strategy:
Unanimous

Default:
Deny

Short Circuit:
On Deny

Evaluators:
4
```

---

# 210. Resolve diagnostics

```text
volt authorization:resolve invoice.approve
```

deberá mostrar también la estrategia.

---

# 211. Explain command

Futuro:

```text
volt authorization:explain
```

en entorno de desarrollo podrá mostrar cómo se obtuvo una decisión con un fixture.

---

# 212. Strategy lint

`authorization:lint` podrá detectar:

```text
critical ability uses allow_overrides
consensus has undefined tie behavior
unknown strategy
mandatory voter may abstain without rule
```

---

# 213. CI Security Rule

Una organización podrá establecer:

```text
system.*
must use deny_overrides or unanimous
```

mediante linting.

---

# 214. Decision Strategy Cache

Las instancias stateless podrán compartirse globalmente.

No necesitan cache de decisiones.

---

# 215. Strategy resolution cache

`ability → strategy descriptor` sí puede compilarse.

---

# 216. No dynamic strategy from user input

Nunca permitir:

```text
?strategy=allow_overrides
```

desde request cliente.

La estrategia forma parte de configuración confiable.

---

# 217. External policy configuration

Si una organización administra estrategia desde DB, solo deberá elegir entre IDs permitidos, con validación y controles administrativos.

---

# 218. Security auditability

Cambios de estrategia en runtime/config deberían ser auditables en sistemas empresariales.

---

# 219. DecisionVersion

Futuro:

```text
AuthorizationDecisionVersion
```

podrá identificar:

```text
strategy version
policy registry version
configuration fingerprint
```

para auditorías reproducibles.

---

# 220. Reproducibility

Dado un snapshot de:

```text
Principal
Subject
Context
Evaluator versions
Strategy
```

debería poder explicarse por qué se obtuvo una decisión.

---

# 221. No guarantee of historical replay

Si Policies consultan DB externa mutable, reproducir exactamente puede requerir snapshots de dominio.

El DecisionManager no resolverá este problema por sí solo.

---

# 222. Explainable Decision

El resultado podrá indicar:

```text
Final:
DENY

Strategy:
DenyOverrides

Decisive voter:
TenantIsolationPolicy

Reason:
tenant.mismatch
```

---

# 223. Multiple decisive voters

En `Consensus` pueden existir múltiples voters relevantes.

El trace deberá mostrar la distribución completa.

---

# 224. Strategy-independent observability

La infraestructura de tracing deberá funcionar igual para custom strategies.

---

# 225. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    ├── Decisions/
    │   ├── Decision.php
    │   ├── DecisionResult.php
    │   ├── DecisionVote.php
    │   ├── DecisionManager.php
    │   ├── DecisionManagerInterface.php
    │   ├── DecisionAccumulator.php
    │   ├── DecisionStrategyContext.php
    │   ├── DecisionSummary.php
    │   ├── DefaultDecisionPolicy.php
    │   └── EvaluatorExecutionStatus.php
    │
    ├── Strategies/
    │   ├── Contracts/
    │   │   └── DecisionStrategyInterface.php
    │   │
    │   ├── UnanimousStrategy.php
    │   ├── AffirmativeStrategy.php
    │   ├── ConsensusStrategy.php
    │   ├── DenyOverridesStrategy.php
    │   ├── AllowOverridesStrategy.php
    │   ├── FirstApplicableStrategy.php
    │   ├── PriorityStrategy.php
    │   ├── DecisionStrategyRegistry.php
    │   ├── DecisionStrategyDescriptor.php
    │   └── DecisionStrategyCapabilities.php
    │
    ├── Evaluators/
    │   ├── AuthorizationVoterInterface.php
    │   ├── EvaluatorDescriptor.php
    │   ├── EvaluatorType.php
    │   └── EvaluatorRequirement.php
    │
    ├── Finalization/
    │   ├── AuthorizationResultFinalizer.php
    │   └── PrimaryReasonSelector.php
    │
    └── Exceptions/
        ├── DecisionException.php
        ├── UnknownDecisionStrategyException.php
        ├── InvalidDecisionStrategyException.php
        ├── DecisionStrategyConfigurationException.php
        └── AuthorizationVotingException.php
```

---

# 226. Invariantes del Decision Model

### Invariante 1

Toda decisión individual es `GRANT`, `DENY` o `ABSTAIN`.

### Invariante 2

`ABSTAIN` nunca significa autorización.

### Invariante 3

Una ejecución fallida no es lo mismo que `ABSTAIN`.

### Invariante 4

La ausencia de evaluadores no produce `GRANT`.

### Invariante 5

Un resultado final indeterminado termina en Default Deny.

---

# 227. Invariantes del DecisionManager

### Invariante 1

No resuelve Policies o Gates.

### Invariante 2

No ejecuta lógica de negocio.

### Invariante 3

No almacena estado entre decisiones.

### Invariante 4

No modifica arbitrariamente los votos de evaluadores.

### Invariante 5

Aplica una estrategia explícita y determinista.

---

# 228. Invariantes de Strategy

### Invariante 1

Toda estrategia debe definir comportamiento para todos los conjuntos posibles de votos.

### Invariante 2

Los empates deben tener semántica explícita.

### Invariante 3

Los fallos nunca conceden acceso implícitamente.

### Invariante 4

Las estrategias sensibles al orden requieren orden estable.

### Invariante 5

Short-circuit no puede alterar la semántica final.

---

# 229. Invariantes de Seguridad

### Invariante 1

Default Deny será la configuración recomendada.

### Invariante 2

Un Critical DENY no deberá ignorarse salvo una estrategia de override explícita.

### Invariante 3

Custom strategies solo provienen de código/configuración confiable.

### Invariante 4

Los errores internos no se exponen directamente al cliente.

### Invariante 5

La observabilidad no modifica decisiones.

---

# 230. Invariantes de Runtime

### Invariante 1

DecisionManager es stateless.

### Invariante 2

Strategies compartidas son stateless.

### Invariante 3

Accumulators son locales a una decisión.

### Invariante 4

No existe fuga de votos entre requests.

### Invariante 5

Metadata compilada puede compartirse bajo FrankenPHP.

---

# 231. Arquitectura final

```text
                  AuthorizationRequest
                          │
                          ↓
                  AuthorizationPlan
                          │
              ┌───────────┴───────────┐
              ↓                       ↓
       EvaluatorDescriptor      StrategyDescriptor
              │                       │
              ↓                       ↓
       Evaluator Execution      Strategy Registry
              │                       │
              ↓                       │
          DecisionVote                │
              │                       │
              └───────────┬───────────┘
                          ↓
                   DecisionManager
                          │
                          ↓
                Decision Accumulator
                          │
                          ↓
                  Strategy Semantics
                          │
                ┌─────────┼─────────┐
                ↓         ↓         ↓
              GRANT      DENY    ABSTAIN
                          │
                          ↓
               Default Decision Policy
                          │
                          ↓
                 Reason Selection
                          │
                          ↓
              Authorization Finalizer
                          │
                          ↓
                Final DecisionResult
```

---

# 232. Ejemplo completo — Invoice approval

Solicitud:

```php
$user->can(
    'approve',
    $invoice
);
```

Plan:

```text
Strategy:
DenyOverrides

Evaluators:

SuspendedPrincipalPolicy
priority=1000
requirement=critical

TenantIsolationPolicy
priority=900
requirement=critical

PermissionVoter
priority=700
requirement=required

InvoicePolicy
priority=500
requirement=required

CompliancePolicy
priority=400
requirement=required
```

Resultados:

```text
SuspendedPrincipalPolicy
    → ABSTAIN

TenantIsolationPolicy
    → GRANT

PermissionVoter
    → GRANT

InvoicePolicy
    → GRANT

CompliancePolicy
    → DENY
      compliance.second_approval_required
```

Resultado:

```text
DenyOverrides
      ↓
DENY
```

Razón primaria:

```text
compliance.second_approval_required
```

---

# 233. Ejemplo — Super Admin Override

Plan:

```text
Strategy:
AllowOverrides

SuperAdminVoter
priority=10000

NormalPermissionVoter
priority=500

ResourcePolicy
priority=400
```

Resultado:

```text
SuperAdminVoter → GRANT
```

La estrategia puede terminar inmediatamente.

Este comportamiento deberá ser explícito y auditable.

---

# 234. Ejemplo — Tenant Security

Resultados:

```text
TenantIsolationPolicy → DENY
ResourcePolicy        → GRANT
```

Con:

```text
DenyOverrides
```

resultado:

```text
DENY
```

Esto impide que una Policy de aplicación pueda accidentalmente ignorar aislamiento multi-tenant.

---

# 235. Ejemplo — Consensus

Votos:

```text
RiskEvaluator A → GRANT
RiskEvaluator B → GRANT
RiskEvaluator C → DENY
RiskEvaluator D → ABSTAIN
```

Conteo:

```text
GRANT = 2
DENY = 1
```

Resultado:

```text
GRANT
```

si la estrategia está configurada para mayoría simple.

---

# 236. Ejemplo — All Abstain

```text
Policy A → ABSTAIN
Policy B → ABSTAIN
Gate C   → ABSTAIN
```

Strategy:

```text
ABSTAIN
```

Finalizer:

```text
Default Deny
```

Resultado:

```text
DENY
reasonCode:
authorization.all_abstained
```

---

# 237. Ejemplo — Evaluator Failure

```text
TenantPolicy → GRANT

PermissionVoter → FAILED

InvoicePolicy → GRANT
```

Si `PermissionVoter` es:

```text
required
```

resultado:

```text
DENY
reasonCode:
authorization.evaluator_failure
```

---

# 238. Ejemplo — FirstApplicable

Orden:

```text
EmergencyAccessVoter
DepartmentPolicy
ResourcePolicy
```

Votos:

```text
EmergencyAccess → ABSTAIN
DepartmentPolicy → GRANT
```

Resultado:

```text
GRANT
```

ResourcePolicy no se ejecuta.

---

# 239. Filosofía del sistema

La filosofía deberá ser:

```text
Evaluators express opinions.

Strategies define how opinions interact.

The DecisionManager coordinates them.

The Finalizer guarantees a safe outcome.
```

---

# 240. Resultado esperado

El `Decision Manager, Voters and Strategy System` permitirá que VoltStack evolucione desde una autorización simple:

```php
$user->can('update', $post);
```

hasta decisiones compuestas por múltiples fuentes:

```text
Authentication State
        +
Tenant Isolation
        +
Role
        +
Permission
        +
Ownership
        +
Security Context
        +
Compliance
        +
Risk
        ↓
GRANT / DENY / ABSTAIN votes
        ↓
Decision Strategy
        ↓
Explainable Final Decision
```

sin cambiar la API pública.

El principio definitivo será:

```text
Policies and Gates answer locally.

Voters normalize those answers.

Strategies define how answers interact.

The DecisionManager produces one deterministic result.

When nobody can safely authorize,
VoltStack denies by default.
```

Con este subsistema, VoltStack dispondrá de un motor de autorización composable, explicable y preparado para reglas empresariales complejas sin sacrificar seguridad ni rendimiento.