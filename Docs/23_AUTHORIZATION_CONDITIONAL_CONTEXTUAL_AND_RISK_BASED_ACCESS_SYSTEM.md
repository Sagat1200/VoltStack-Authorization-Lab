# VoltStack Authorization System
## Conditional, Contextual and Risk-Based Access System

**Documento:** `23_AUTHORIZATION_CONDITIONAL_CONTEXTUAL_AND_RISK_BASED_ACCESS_SYSTEM.md`  
**Sistema:** Authorization  
**Framework:** VoltStack  
**Módulo sugerido:** `Quantum/Authorization`  
**Estado:** Especificación arquitectónica  
**Versión objetivo:** 1.x+

---

# 1. Propósito

Este documento define la arquitectura del sistema de **autorización condicional, contextual y basada en riesgo** de VoltStack.

Los documentos anteriores establecieron cómo determinar si un Principal posee autoridad mediante:

- Policies.
- Gates y Abilities.
- Voters y estrategias de decisión.
- RBAC.
- ABAC.
- ReBAC.
- Multi-Tenancy.
- Ownership.
- Sharing.
- Delegation.
- Impersonation.
- Capabilities.
- Service-to-Service Authorization.
- Organizations.
- Teams.
- Workspaces.
- Hierarchical Scopes.

Sin embargo, poseer autoridad sobre un recurso no significa necesariamente que una operación deba permitirse **en cualquier circunstancia**.

Ejemplo:

```text
User#42
    ↓
organization.admin
    ↓
Organization#14
    ↓
Ability:
billing.payment.approve
```

La autorización estructural podría producir:

```text
GRANT
```

pero la operación ocurre desde:

```text
new device
+
low authentication assurance
+
unusual session
+
high-value transaction
```

VoltStack podrá transformar el resultado en:

```text
CHALLENGE
```

requiriendo:

```text
step-up authentication
```

antes de permitir la operación.

El principio fundamental será:

> **Authorization answers whether a Principal has authority. Conditional authorization determines whether that authority may be exercised under the current circumstances.**

---

# 2. Objetivo arquitectónico

VoltStack deberá poder responder preguntas como:

```text
¿Tiene permiso?

¿Dentro de qué Tenant?

¿Dentro de qué Scope?

¿Sobre qué recurso?

¿En qué contexto?

¿Con qué nivel de autenticación?

¿Desde qué tipo de sesión?

¿La operación es sensible?

¿Existen señales de riesgo?

¿Se necesita una garantía adicional?

¿Debe permitirse, denegarse o requerirse una acción adicional?
```

Por tanto:

```text
Authorization
≠
simple boolean permission check
```

sino:

```text
Authorization =
    Structural Authority
    +
    Context
    +
    Conditions
    +
    Risk
    +
    Assurance
    +
    Security Requirements
```

---

# 3. Motivación

Los modelos clásicos suelen terminar en:

```php
if ($user->can('delete', $resource)) {
    // allow
}
```

Esto es insuficiente para operaciones empresariales sensibles.

Ejemplo:

```text
User has:
finance.payment.approve
```

pero:

```text
payment = $750,000
authentication = password only
device = unknown
session = recently created
```

La respuesta correcta puede no ser:

```text
DENY
```

ni:

```text
GRANT
```

sino:

```text
CHALLENGE:
Require stronger authentication.
```

---

# 4. Modelo conceptual

La decisión contextual será función de:

```text
Principal
+
Actor
+
Ability
+
Subject
+
Tenant
+
Scope
+
Authentication Context
+
Session Context
+
Request Context
+
Resource Context
+
Environmental Context
+
Operation Context
+
Risk Signals
+
Policy
```

---

# 5. Separación de responsabilidades

VoltStack deberá distinguir al menos cuatro conceptos:

```text
Authority
Context
Risk
Assurance
```

---

# 6. Authority

Responde:

```text
¿Tiene el Principal autoridad para realizar la acción?
```

Ejemplo:

```text
workspace.admin
→ workspace.members.manage
```

---

# 7. Context

Responde:

```text
¿En qué circunstancias se está intentando ejercer esa autoridad?
```

Ejemplo:

```text
interactive session
API request
service call
impersonated session
delegated authority
```

---

# 8. Risk

Responde:

```text
¿Existen señales que incrementen el riesgo de permitir la operación?
```

---

# 9. Assurance

Responde:

```text
¿Qué tan fuerte es la evidencia de identidad,
sesión y autenticación disponible?
```

---

# 10. Contextual Authorization

Definiremos:

```text
Contextual Authorization
```

como la evaluación de condiciones relacionadas con el contexto actual de ejecución.

---

# 11. Risk-Based Authorization

Definiremos:

```text
Risk-Based Authorization
```

como la modificación o restricción de una decisión utilizando señales y evaluaciones de riesgo.

---

# 12. Adaptive Authorization

La combinación podrá denominarse:

```text
Adaptive Authorization
```

porque la decisión puede adaptarse dinámicamente al contexto.

---

# 13. No reemplaza RBAC/ABAC/ReBAC

Este subsistema no reemplaza:

```text
RBAC
ABAC
ReBAC
Policies
```

Los complementa.

---

# 14. Ejemplo completo

```text
User#42
role=finance.manager
scope=Organization#14
```

Ability:

```text
payment.approve
```

RBAC:

```text
GRANT
```

ABAC:

```text
payment.amount <= approval_limit
→ GRANT
```

Risk:

```text
authentication assurance insufficient
→ CHALLENGE
```

Resultado:

```text
STEP-UP REQUIRED
```

---

# 15. Authorization Context

El núcleo deberá trabajar con un contexto explícito e inmutable.

Conceptualmente:

```php
final readonly class AuthorizationContext
{
    public function __construct(
        public AuthenticationContext $authentication,
        public SessionContext $session,
        public RequestContext $request,
        public EnvironmentContext $environment,
        public OperationContext $operation,
        public ?RiskContext $risk = null,
    ) {}
}
```

---

# 16. No Global Mutable Context

Prohibido depender de:

```php
static::$currentUser;
static::$currentTenant;
static::$currentWorkspace;
static::$riskScore;
```

---

# 17. Razón

VoltStack está diseñado para funcionar correctamente con:

```text
FrankenPHP
persistent workers
async execution
concurrency
long-running processes
```

El contexto deberá ser:

```text
request-scoped
execution-scoped
immutable
```

---

# 18. Authentication Context

Representará cómo se autenticó el Principal.

```php
final readonly class AuthenticationContext
{
    public function __construct(
        public AuthenticationAssuranceLevel $assurance,
        public array $methods,
        public ?DateTimeImmutable $authenticatedAt,
        public ?DateTimeImmutable $lastStepUpAt,
    ) {}
}
```

---

# 19. Authentication Methods

Ejemplos:

```text
password
passkey
TOTP
hardware key
certificate
federated identity
service credential
API token
```

---

# 20. No hardcodear métodos

Authorization no deberá asumir:

```text
MFA = TOTP
```

El sistema debe trabajar principalmente con:

```text
assurance requirements
```

y capacidades verificadas.

---

# 21. Authentication Assurance Level

Conceptualmente:

```php
enum AuthenticationAssuranceLevel: int
{
    case Anonymous = 0;
    case Low = 10;
    case Standard = 20;
    case Strong = 30;
    case VeryStrong = 40;
}
```

Los valores exactos deberán ser configurables o adaptables.

---

# 22. Semántica

Ejemplo conceptual:

```text
Low
→ weak/basic authentication

Standard
→ normal authenticated session

Strong
→ additional authentication evidence

VeryStrong
→ high-assurance authentication
```

---

# 23. No confundir método con assurance

Una Policy deberá preferir:

```text
assurance >= Strong
```

en lugar de:

```text
mustHaveTotp = true
```

cuando el objetivo sea fortaleza de autenticación.

---

# 24. Ventaja

Así nuevas tecnologías pueden satisfacer la misma Policy.

Por ejemplo:

```text
passkey
hardware key
certificate
```

sin modificar la autorización.

---

# 25. Authentication Age

Algunas operaciones requieren autenticación reciente.

Ejemplo:

```text
authenticated within last 30 minutes
```

---

# 26. Fresh Authentication

Deberá existir una abstracción como:

```text
AuthenticationFreshness
```

---

# 27. Example

```text
user authenticated 7 hours ago
```

Role:

```text
account.owner
```

Ability:

```text
account.security.disable_mfa
```

Policy:

```text
requires fresh strong authentication
```

Resultado:

```text
CHALLENGE
```

---

# 28. Step-Up Authentication

VoltStack deberá soportar explícitamente:

```text
step-up authentication
```

---

# 29. Definición

Step-up significa:

```text
la identidad ya está autenticada,
pero la operación requiere mayor assurance.
```

---

# 30. No debe producir simplemente DENY

Si la autoridad existe pero assurance es insuficiente:

```text
CHALLENGE
```

es semánticamente mejor que:

```text
DENY
```

---

# 31. Authorization Outcome

El sistema deberá extender el modelo clásico:

```text
ALLOW
DENY
```

hacia:

```text
ALLOW
DENY
ABSTAIN
CHALLENGE
FAILURE
```

---

# 32. Challenge

`CHALLENGE` significa:

> La autoridad potencialmente existe, pero falta satisfacer una condición recuperable.

---

# 33. Examples

```text
require stronger authentication
require recent authentication
require user confirmation
require approval
require additional verification
```

---

# 34. Failure

`FAILURE` seguirá reservado para:

```text
authorization infrastructure unavailable
risk provider unavailable when mandatory
invalid context
provider failure
```

No para una denegación normal.

---

# 35. ChallengeRequirement

Conceptualmente:

```php
interface ChallengeRequirementInterface
{
    public function type(): string;
}
```

---

# 36. Authentication Challenge

```php
final readonly class AuthenticationChallenge
    implements ChallengeRequirementInterface
{
    public function __construct(
        public AuthenticationAssuranceLevel $minimumAssurance,
        public ?DateInterval $maximumAge = null,
    ) {}
}
```

---

# 37. Multiple Requirements

Una decisión podrá requerir:

```text
Strong Authentication
+
Fresh Authentication
+
Explicit Confirmation
```

---

# 38. AuthorizationDecision

Conceptualmente:

```php
final readonly class AuthorizationDecision
{
    public function __construct(
        public AuthorizationOutcome $outcome,
        public string $reason,
        public array $requirements = [],
        public array $metadata = [],
    ) {}
}
```

---

# 39. Challenge is not authentication implementation

Authorization solo determina:

```text
what assurance is required
```

El subsistema Authentication determina:

```text
how to satisfy it
```

---

# 40. Separation

```text
Authorization
    ↓
Requires Strong Assurance

Authentication
    ↓
chooses Passkey / MFA / other mechanism
```

---

# 41. Session Context

Representará propiedades de la sesión actual.

```php
final readonly class SessionContext
{
    public function __construct(
        public string $type,
        public ?string $id,
        public ?DateTimeImmutable $createdAt,
        public ?DateTimeImmutable $lastActivityAt,
        public bool $impersonated = false,
        public bool $delegated = false,
    ) {}
}
```

---

# 42. Session Type

Ejemplos:

```text
interactive
api
service
cli
background
impersonated
delegated
```

---

# 43. Session Age

Algunas operaciones pueden restringirse para sesiones demasiado antiguas.

---

# 44. New Session

Una sesión recién creada también puede ser relevante en determinados modelos de riesgo.

---

# 45. Session Binding

Authorization puede recibir señales verificadas como:

```text
session assurance
session type
credential class
```

pero no deberá implementar gestión de sesiones.

---

# 46. Impersonation Context

Documento 20 definió impersonation.

Este subsistema deberá reconocer:

```text
actor != principal
```

---

# 47. Sensitive Operation During Impersonation

Ejemplo:

```text
SupportAgent impersonates Customer
```

Puede:

```text
view support information
```

pero:

```text
change payout destination
```

puede ser:

```text
DENY
```

aunque el Customer pudiera hacerlo directamente.

---

# 48. Conditional Rule

```text
if context.impersonated
and ability is financial-sensitive
→ DENY
```

---

# 49. Delegated Session

También podrá restringirse:

```text
delegated authority
```

para determinadas operaciones.

---

# 50. Request Context

Representará información verificada de la solicitud.

Conceptualmente:

```php
final readonly class RequestContext
{
    public function __construct(
        public string $channel,
        public string $transport,
        public ?string $requestId = null,
        public array $attributes = [],
    ) {}
}
```

---

# 51. Channel

Ejemplos:

```text
web
api
cli
queue
scheduler
internal
service
```

---

# 52. Transport

Ejemplos:

```text
http
https
cli
message
rpc
```

---

# 53. Secure Transport Requirement

Una Ability sensible podrá exigir:

```text
secure transport
```

---

# 54. Importante

Authorization deberá consumir un atributo confiable como:

```text
transport_secure=true
```

No interpretar ciegamente headers controlados por cliente.

---

# 55. Trusted Context

Toda señal de seguridad deberá indicar su procedencia.

---

# 56. Context Provenance

Ejemplo:

```php
final readonly class ContextAttribute
{
    public function __construct(
        public string $name,
        public mixed $value,
        public ContextTrustLevel $trust,
        public string $source,
    ) {}
}
```

---

# 57. Context Trust Level

```php
enum ContextTrustLevel: string
{
    case Untrusted = 'untrusted';
    case Derived = 'derived';
    case Verified = 'verified';
    case Authoritative = 'authoritative';
}
```

---

# 58. Security Rule

Una condición crítica no deberá confiar en:

```text
untrusted context
```

---

# 59. Ejemplo incorrecto

Cliente envía:

```http
X-Trusted-Device: true
```

y Authorization lo acepta directamente.

Prohibido.

---

# 60. Ejemplo correcto

```text
DeviceTrustProvider
    ↓
verified device signal
    ↓
AuthorizationContext
```

---

# 61. Environment Context

Representará circunstancias externas relevantes.

```php
final readonly class EnvironmentContext
{
    public function __construct(
        public DateTimeImmutable $time,
        public array $attributes = [],
    ) {}
}
```

---

# 62. Time-Based Authorization

Ejemplo:

```text
maintenance.contractor
```

solo puede operar:

```text
08:00–18:00
```

---

# 63. Timezone

Las reglas deberán especificar claramente la timezone.

---

# 64. No usar timezone implícita

Una Policy deberá declarar:

```text
organization timezone
```

o:

```text
UTC
```

según dominio.

---

# 65. ClockInterface

Todos los checks temporales deberán depender de:

```php
interface ClockInterface
{
    public function now(): DateTimeImmutable;
}
```

---

# 66. Ventaja

Permite tests deterministas.

---

# 67. Operation Context

Representará propiedades de la operación solicitada.

```php
final readonly class OperationContext
{
    public function __construct(
        public string $ability,
        public AuthorizationSensitivity $sensitivity,
        public array $attributes = [],
    ) {}
}
```

---

# 68. Authorization Sensitivity

```php
enum AuthorizationSensitivity: string
{
    case Low = 'low';
    case Normal = 'normal';
    case Elevated = 'elevated';
    case High = 'high';
    case Critical = 'critical';
}
```

---

# 69. Ability Sensitivity Registry

Abilities podrán declarar sensibilidad.

Ejemplo:

```text
document.view
→ normal

workspace.member.invite
→ elevated

api_key.create
→ high

billing.payout.change
→ critical
```

---

# 70. Sensitivity no concede ni deniega

Solo influye en:

```text
required assurance
risk tolerance
logging
approval requirements
```

---

# 71. Authorization Condition

Una condición será una regla evaluable sobre el contexto.

```php
interface AuthorizationConditionInterface
{
    public function evaluate(
        AuthorizationRequest $request,
        AuthorizationContext $context
    ): ConditionResult;
}
```

---

# 72. ConditionResult

```php
final readonly class ConditionResult
{
    public function __construct(
        public ConditionOutcome $outcome,
        public string $reason,
        public array $requirements = [],
    ) {}
}
```

---

# 73. ConditionOutcome

```php
enum ConditionOutcome: string
{
    case Satisfied = 'satisfied';
    case Unsatisfied = 'unsatisfied';
    case Challenge = 'challenge';
    case Failure = 'failure';
}
```

---

# 74. Conditional Policies

Ejemplo:

```text
Ability:
billing.payout.change

Conditions:
    authentication >= Strong
    authentication age <= 15 minutes
    not impersonated
```

---

# 75. Declarative Condition

Podrá definirse mediante metadata.

Conceptualmente:

```php
#[Authorize('billing.payout.change')]
#[RequiresAssurance('strong')]
#[RequiresFreshAuthentication('PT15M')]
#[DisallowImpersonation]
public function updatePayout(...)
{
}
```

---

# 76. Compilation

Estas condiciones deberán compilarse junto con metadata de Authorization.

---

# 77. No Reflection Hot Path

En producción:

```text
Controller metadata
    ↓
compiled authorization plan
```

---

# 78. Conditional Rule Registry

Deberá existir:

```php
interface AuthorizationConditionRegistryInterface
{
    public function register(
        string $name,
        AuthorizationConditionInterface $condition
    ): void;
}
```

---

# 79. Built-In Conditions

VoltStack podrá incluir:

```text
RequiresAssurance
RequiresFreshAuthentication
RequiresSessionType
DisallowImpersonation
DisallowDelegation
RequiresSecureTransport
RequiresTrustedDevice
RequiresApproval
RequiresConfirmation
```

sin convertir todas en reglas obligatorias.

---

# 80. Extensible Conditions

Aplicaciones podrán registrar:

```text
RequiresCorporateNetwork
RequiresManagedEndpoint
RequiresTradingWindow
RequiresComplianceTraining
RequiresOnCallStatus
```

---

# 81. Conditional Access Policy

Una Policy contextual podrá representarse como:

```php
final readonly class ConditionalAccessPolicy
{
    public function __construct(
        public string $id,
        public array $abilities,
        public array $conditions,
        public int $priority = 0,
    ) {}
}
```

---

# 82. Policy Targeting

Puede aplicarse por:

```text
Ability
Role
Principal type
Tenant
Scope type
Resource type
Sensitivity
Channel
```

---

# 83. Tenant-Specific Conditional Policies

Tenant A puede exigir:

```text
Strong assurance
```

para exportaciones.

Tenant B puede exigirlo solo para:

```text
Critical operations
```

---

# 84. Scope-Specific Conditions

Workspace confidencial puede exigir:

```text
trusted device
```

aunque otros Workspaces no.

---

# 85. Resource-Specific Conditions

Documento marcado:

```text
classification=restricted
```

puede requerir:

```text
higher assurance
```

---

# 86. ABAC Integration

Estas condiciones pueden verse como una especialización avanzada de ABAC.

Sin embargo, mantener un subsistema explícito aporta:

```text
challenge semantics
risk evaluation
assurance levels
context provenance
step-up
adaptive policies
```

---

# 87. Risk Context

Representará la evaluación de riesgo disponible.

```php
final readonly class RiskContext
{
    public function __construct(
        public RiskLevel $level,
        public ?float $score,
        public array $signals,
        public string $source,
        public DateTimeImmutable $evaluatedAt,
    ) {}
}
```

---

# 88. Risk Level

```php
enum RiskLevel: string
{
    case Unknown = 'unknown';
    case Low = 'low';
    case Medium = 'medium';
    case High = 'high';
    case Critical = 'critical';
}
```

---

# 89. Score

Opcionalmente:

```text
0.0 – 1.0
```

o cualquier modelo normalizado por adapter.

Authorization Core deberá preferir:

```text
RiskLevel
```

como abstracción estable.

---

# 90. Risk Signal

```php
final readonly class RiskSignal
{
    public function __construct(
        public string $type,
        public RiskSignalSeverity $severity,
        public ContextTrustLevel $trust,
        public string $source,
        public array $metadata = [],
    ) {}
}
```

---

# 91. Ejemplos de señales

```text
authentication anomaly
untrusted session
credential anomaly
unusual access pattern
new device
high-value operation
rapid privilege change
suspicious delegation
unexpected service identity
```

---

# 92. Privacy

Risk signals deberán minimizar:

```text
personal data
raw telemetry
unnecessary identifiers
```

---

# 93. Authorization no es Fraud Engine

VoltStack Authorization no deberá intentar convertirse en:

```text
SIEM
fraud detection engine
UEBA
device fingerprinting platform
```

---

# 94. Risk Provider

Estas capacidades deberán integrarse mediante providers.

```php
interface AuthorizationRiskProviderInterface
{
    public function evaluate(
        AuthorizationRequest $request,
        AuthorizationContext $context
    ): RiskAssessment;
}
```

---

# 95. Risk Assessment

```php
final readonly class RiskAssessment
{
    public function __construct(
        public RiskLevel $level,
        public ?float $score,
        public array $signals,
        public RiskAssessmentConfidence $confidence,
    ) {}
}
```

---

# 96. Multiple Risk Providers

Podrán existir:

```text
SessionRiskProvider
IdentityRiskProvider
OperationRiskProvider
TenantRiskProvider
ExternalRiskProvider
```

---

# 97. Risk Aggregator

```php
interface RiskAggregatorInterface
{
    public function aggregate(
        iterable $assessments
    ): RiskAssessment;
}
```

---

# 98. Aggregation Strategy

Podrá utilizar:

```text
HighestRisk
Weighted
Priority
Custom
```

---

# 99. Security Default

No utilizar promedio ingenuo para señales críticas.

Ejemplo:

```text
LOW + LOW + CRITICAL
```

no debería producir simplemente:

```text
MEDIUM
```

si la señal Critical es autoritativa.

---

# 100. Risk Strategy

```php
enum RiskAggregationStrategy: string
{
    case Highest = 'highest';
    case Weighted = 'weighted';
    case Priority = 'priority';
    case Custom = 'custom';
}
```

---

# 101. Unknown Risk

`Unknown` deberá ser tratado explícitamente.

---

# 102. No asumir Unknown = Low

Eso podría crear fail-open.

---

# 103. Risk Requirement

Una Ability puede declarar:

```text
maximum acceptable risk
```

---

# 104. Example

```text
document.view
max risk = High

api_key.create
max risk = Medium

payout.change
max risk = Low
```

---

# 105. Risk Response

Cuando risk excede threshold:

```text
DENY
```

o:

```text
CHALLENGE
```

dependiendo de Policy.

---

# 106. Example

```text
Risk = Medium
Ability = payout.change
```

Policy:

```text
Medium → Strong fresh authentication
High → DENY
Critical → DENY + security event
```

---

# 107. Adaptive Assurance

El assurance requerido podrá aumentar según risk.

---

# 108. Example

Base:

```text
workspace.export
requires Standard
```

Risk Medium:

```text
requires Strong
```

Risk High:

```text
DENY
```

---

# 109. Risk Matrix

Conceptualmente:

| Operation | Low Risk | Medium Risk | High Risk | Critical Risk |
|---|---|---|---|---|
| Normal | Allow | Allow | Challenge | Deny |
| Elevated | Allow | Challenge | Deny | Deny |
| High | Challenge | Challenge | Deny | Deny |
| Critical | Challenge | Deny | Deny | Deny |

La tabla deberá ser configurable.

---

# 110. No hardcodear matriz universal

Cada aplicación/Tenant puede definir tolerancias diferentes.

---

# 111. RiskPolicy

Contrato:

```php
interface RiskPolicyInterface
{
    public function evaluate(
        AuthorizationRequest $request,
        AuthorizationContext $context,
        RiskAssessment $risk
    ): AuthorizationDecision;
}
```

---

# 112. Risk Policy Registry

Puede resolverse por:

```text
ability
operation sensitivity
tenant
scope
resource
```

---

# 113. Context Providers

No todo contexto estará disponible inicialmente.

Deberá existir:

```php
interface AuthorizationContextProviderInterface
{
    public function enrich(
        AuthorizationContext $context,
        AuthorizationRequest $request
    ): AuthorizationContext;
}
```

---

# 114. Provider Examples

```text
AuthenticationContextProvider
SessionContextProvider
DeviceContextProvider
NetworkContextProvider
RiskContextProvider
OperationContextProvider
```

---

# 115. Lazy Context Resolution

No cargar señales costosas si no son necesarias.

---

# 116. Example

Para:

```text
profile.avatar.view
```

no consultar un proveedor externo de risk.

---

# 117. Authorization Planner

Documento 09 deberá determinar:

```text
qué contexto necesita cada Ability.
```

---

# 118. Compiled Context Requirements

Ejemplo:

```text
billing.payout.change

requires:
    authentication
    authentication_freshness
    session
    impersonation
    risk
```

---

# 119. Minimal Context Principle

Recolectar únicamente:

```text
context required by policy
```

---

# 120. Beneficios

```text
performance
privacy
lower latency
lower external dependency
simpler observability
```

---

# 121. Context Dependency Graph

Ejemplo:

```text
RiskPolicy
    ↓
RiskContext
    ↓
SessionRiskProvider
    ↓
SessionContext
```

El Planner podrá ordenar providers.

---

# 122. Context Provider Failure

Debe distinguirse entre:

```text
context absent
```

y:

```text
provider failed
```

---

# 123. Example

Ability no requiere DeviceContext:

```text
missing device
→ irrelevant
```

Ability exige trusted device:

```text
device provider unavailable
→ FAILURE / DENY
```

según failure policy.

---

# 124. Fail-Closed

Para condiciones críticas:

```text
mandatory context unavailable
→ no ALLOW
```

---

# 125. Optional Context

Para optimizaciones o señales no críticas:

```text
provider failure
```

puede degradarse de forma controlada.

---

# 126. Risk Provider Failure Policy

```php
enum RiskProviderFailurePolicy: string
{
    case Deny = 'deny';
    case Challenge = 'challenge';
    case Failure = 'failure';
    case IgnoreOptional = 'ignore_optional';
}
```

---

# 127. Recommended

Operaciones críticas:

```text
FAIL CLOSED
```

---

# 128. Context Conditions vs Resource Policies

Orden conceptual:

```text
Structural Authority
    ↓
Resource Policy
    ↓
Context Conditions
    ↓
Risk
    ↓
Final Decision
```

Pero el Planner podrá reordenar checks seguros para eficiencia.

---

# 129. Important Invariant

Una optimización nunca deberá cambiar la semántica final.

---

# 130. Cheap Denials First

Ejemplo:

```text
Tenant mismatch
```

deberá denegar antes de consultar Risk Provider externo.

---

# 131. Suggested Pipeline

```text
Request Validation
        ↓
Principal Resolution
        ↓
Tenant Isolation
        ↓
Scope Resolution
        ↓
Structural Authority
        ↓
Resource Policy
        ↓
Context Requirement Resolution
        ↓
Context Enrichment
        ↓
Conditional Evaluation
        ↓
Risk Evaluation
        ↓
Assurance Evaluation
        ↓
Challenge Resolution
        ↓
Final Decision
```

---

# 132. Early Context Conditions

Algunas condiciones muy baratas pueden ejecutarse antes.

Ejemplo:

```text
impersonation forbidden
```

---

# 133. Planner Responsibility

El Planner deberá clasificar checks por:

```text
security precedence
cost
dependencies
short-circuitability
```

---

# 134. Authorization Plan

Ejemplo compilado:

```text
Ability: payout.change

1 TenantBoundary
2 ScopeResolution
3 RBAC
4 PayoutPolicy
5 DisallowImpersonation
6 ResolveAuthenticationContext
7 ResolveRiskContext
8 RequireRisk <= Medium
9 RequireAssurance >= Strong
10 RequireFreshAuthentication <= 15m
```

---

# 135. Challenge Accumulation

Si faltan varias condiciones recuperables:

```text
assurance too low
authentication too old
```

el sistema puede devolver ambas.

---

# 136. Example

```text
CHALLENGE

requirements:
- minimum_assurance: strong
- authentication_age: <= 15m
```

---

# 137. Challenge Deduplication

Si dos Policies requieren:

```text
Standard
Strong
```

se conserva:

```text
Strong
```

---

# 138. Requirement Normalizer

```php
interface ChallengeRequirementNormalizerInterface
{
    public function normalize(
        iterable $requirements
    ): array;
}
```

---

# 139. Assurance Merge

Usar:

```text
maximum required assurance
```

---

# 140. Freshness Merge

Usar:

```text
most restrictive maximum age
```

---

# 141. Deny Dominance

Si una Policy produce:

```text
DENY
```

y otra:

```text
CHALLENGE
```

la estrategia normal deberá producir:

```text
DENY
```

---

# 142. NonBypassable Deny

Security rules críticas seguirán dominando.

---

# 143. Risk Cannot Create Authority

Regla fundamental:

```text
LOW RISK
```

nunca puede convertir:

```text
NO AUTHORITY
```

en:

```text
ALLOW
```

---

# 144. Formally

```text
Risk may restrict authority.
Risk may require stronger assurance.
Risk must not invent authority.
```

---

# 145. Assurance Cannot Create Authority

Igualmente:

```text
VeryStrong Authentication
```

no concede:

```text
organization.admin
```

---

# 146. Trusted Device Cannot Create Authority

También.

---

# 147. Context is restrictive or qualifying

No una fuente primaria de permisos, salvo Policies ABAC explícitamente diseñadas para ello.

---

# 148. Risk Score Stability

Risk assessments deberán tener:

```text
evaluatedAt
```

---

# 149. Risk TTL

Una Policy podrá declarar:

```text
maximum risk assessment age
```

---

# 150. Example

```text
Critical operation
requires risk assessment <= 2 minutes old
```

---

# 151. Risk Reuse

Dentro de una misma request:

```text
memoization
```

será recomendable.

---

# 152. Cross-Request Cache

Solo si:

```text
provider permits it
TTL is explicit
cache key is safe
```

---

# 153. No Broad Risk Cache

Nunca:

```text
user risk = low forever
```

---

# 154. Risk Cache Key

Puede considerar:

```text
principal
session
operation class
tenant
provider version
```

según provider.

---

# 155. Risk Context and Tokens

Un token puede incluir claims verificadas como:

```text
authentication assurance
authentication time
credential class
```

---

# 156. Trust

Solo si token fue:

```text
cryptographically verified
```

y claims son emitidas por autoridad confiable.

---

# 157. Token Claim Adapter

Authentication subsystem deberá convertir claims en:

```text
verified AuthorizationContext
```

---

# 158. Service-to-Service

Servicios no siempre usan MFA.

Por ello assurance deberá ser genérico.

---

# 159. Service Assurance

Puede derivarse de:

```text
mTLS
workload identity
signed token
short-lived credentials
hardware-backed identity
```

---

# 160. Service Authentication Context

No debe falsificarse como:

```text
user MFA
```

---

# 161. Principal Class

Policies podrán distinguir:

```text
HumanPrincipal
ServicePrincipal
MachinePrincipal
```

---

# 162. Human-Only Operation

Ejemplo:

```text
legal.accept_terms
```

puede exigir:

```text
HumanPrincipal
```

---

# 163. Machine-Only Operation

Ejemplo:

```text
internal.replication.write
```

puede exigir:

```text
ServicePrincipal
```

---

# 164. Interactive Confirmation

Algunas operaciones podrán exigir:

```text
explicit user confirmation
```

---

# 165. Example

```text
delete organization
```

Puede requerir:

```text
Strong Authentication
+
Fresh Authentication
+
Explicit Confirmation
```

---

# 166. Confirmation Challenge

```php
final readonly class ConfirmationChallenge
    implements ChallengeRequirementInterface
{
    public function __construct(
        public string $operation,
        public string $noncePurpose,
    ) {}
}
```

---

# 167. Confirmation Security

No aceptar:

```text
confirmed=true
```

enviado arbitrariamente por cliente.

---

# 168. Confirmation Proof

Deberá ser emitido/verificado por subsistema confiable.

---

# 169. One-Time Proof

Para operaciones críticas podrá ser:

```text
short-lived
operation-bound
principal-bound
scope-bound
single-use
```

---

# 170. Challenge Token

VoltStack podrá definir una abstracción:

```text
AuthorizationChallengeToken
```

---

# 171. Purpose

Después de completar step-up:

```text
replay original authorization
```

con proof verificable.

---

# 172. Challenge Token Security

Debe estar ligado a:

```text
principal
actor
ability
subject
tenant
scope
requirements
expiration
nonce
```

---

# 173. No Generic MFA Bypass Token

Prohibido generar:

```text
mfa_passed=true
```

válido para cualquier operación indefinidamente.

---

# 174. Challenge Replay

Flow:

```text
Authorization
    ↓
CHALLENGE
    ↓
Authentication/Verification
    ↓
Proof
    ↓
Authorization replay
    ↓
ALLOW or DENY
```

---

# 175. Re-Evaluation Required

Completar challenge no deberá convertir automáticamente la decisión original en ALLOW.

Debe:

```text
re-evaluate authorization
```

---

# 176. Why

Durante el challenge pudo cambiar:

```text
role
resource
tenant membership
scope
risk
account status
```

---

# 177. Challenge Expiration

Todo proof deberá expirar.

---

# 178. Resource Binding

Challenge para:

```text
Document#100 delete
```

no debe servir para:

```text
Document#200 delete
```

salvo Policy explícita.

---

# 179. Ability Binding

Challenge para:

```text
api_key.create
```

no debe autorizar:

```text
organization.delete
```

---

# 180. Scope Binding

Challenge para:

```text
Workspace#91
```

no debe reutilizarse en:

```text
Workspace#92
```

por defecto.

---

# 181. Authentication Step-Up Window

Una vez completado step-up, una sesión podrá mantener:

```text
Strong assurance
```

por una ventana limitada.

---

# 182. Policy-Specific Freshness

Una operación puede requerir:

```text
step-up within 5 minutes
```

otra:

```text
within 30 minutes
```

---

# 183. No Universal Freshness Constant

Debe ser Policy-driven.

---

# 184. Risk Escalation During Session

Una sesión inicialmente Low Risk puede cambiar a:

```text
High Risk
```

---

# 185. Authorization Must Re-Evaluate

No asumir que una decisión de inicio de sesión es válida para toda la sesión.

---

# 186. Continuous Authorization

VoltStack podrá soportar conceptualmente:

```text
continuous authorization
```

para procesos largos.

---

# 187. Example

Long-running administrative operation:

```text
export 5M records
```

Puede requerir revalidación antes de etapas sensibles.

---

# 188. V1

No es necesario construir un engine continuo complejo.

Debe existir arquitectura que permita:

```text
authorization checkpoints
```

---

# 189. Authorization Checkpoint

```php
interface AuthorizationCheckpointInterface
{
    public function authorize(
        AuthorizationRequest $request
    ): AuthorizationDecision;
}
```

---

# 190. Queue Jobs

Contexto HTTP no debe serializarse completo hacia queue.

---

# 191. Security

Nunca serializar ciegamente:

```text
risk context
session object
request headers
authentication internals
```

---

# 192. Background Authorization Snapshot

Cuando sea necesario, crear:

```text
minimal verified authorization snapshot
```

---

# 193. Snapshot

Podrá contener:

```text
principal reference
actor reference
tenant
scope
delegation/capability
authorized operation reference
issued_at
expires_at
```

---

# 194. Revalidation Policy

Jobs sensibles deberán decidir entre:

```text
snapshot authority
```

y:

```text
fresh authorization
```

---

# 195. Recommended

Para efectos irreversibles o sensibles:

```text
fresh/revalidated authorization
```

cuando sea técnicamente posible.

---

# 196. CLI Context

CLI administrativa puede tener:

```text
channel=cli
```

---

# 197. CLI Is Not Automatically Trusted

Ejecutar desde consola no significa:

```text
bypass authorization
```

---

# 198. Explicit System Authority

Operaciones internas deberán utilizar:

```text
SystemPrincipal
```

o capability explícita.

---

# 199. Scheduler

Igualmente.

---

# 200. Conditional Service Policies

Ejemplo:

```text
backup.restore
```

puede exigir:

```text
ServicePrincipal
+
approved capability
+
maintenance window
```

---

# 201. Network Context

VoltStack podrá soportar:

```text
NetworkContext
```

mediante provider.

---

# 202. Examples

```text
trusted network
private network
corporate gateway
service mesh
```

---

# 203. Do Not Trust Raw IP Alone

Las reglas no deberán asumir automáticamente:

```text
IP = identity
```

---

# 204. Proxy Awareness

Network provider deberá respetar:

```text
trusted proxies
verified forwarding metadata
```

---

# 205. Corporate Network Condition

Ejemplo:

```text
production.deploy
requires trusted corporate network
```

---

# 206. Alternative

Una Policy puede permitir:

```text
corporate network
OR
VeryStrong assurance
```

---

# 207. Condition Composition

Deberá soportarse:

```text
ALL
ANY
NOT
THRESHOLD
```

---

# 208. Condition Group

```php
final readonly class ConditionGroup
{
    public function __construct(
        public ConditionOperator $operator,
        public array $conditions,
    ) {}
}
```

---

# 209. ConditionOperator

```php
enum ConditionOperator: string
{
    case All = 'all';
    case Any = 'any';
    case None = 'none';
    case Threshold = 'threshold';
}
```

---

# 210. Example

```text
ALL:
    role authority
    not impersonated

ANY:
    trusted device
    very strong authentication
```

---

# 211. Condition Trees

Podrán compilarse como AST.

---

# 212. Conditional AST

Ejemplo:

```text
AND
├── RequiresStrongAuthentication
├── NOT Impersonated
└── OR
    ├── TrustedDevice
    └── CorporateNetwork
```

---

# 213. Compilation

El AST podrá convertirse a:

```text
optimized evaluation plan
```

---

# 214. Short-Circuit

`AND`:

```text
first hard DENY may stop
```

`OR`:

```text
first sufficient condition may stop
```

siempre respetando audit/mandatory evaluators.

---

# 215. Risk-Based Conditions

Ejemplo AST:

```text
IF Risk >= Medium
THEN RequireStrongAuthentication
```

---

# 216. Conditional Rule

Podrá representarse:

```php
final readonly class AdaptiveAuthorizationRule
{
    public function __construct(
        public AuthorizationPredicate $when,
        public AuthorizationRequirement $then,
    ) {}
}
```

---

# 217. Policy DSL

VoltStack podrá ofrecer posteriormente un DSL.

Ejemplo conceptual:

```php
Authorization::policy('payout.change')
    ->whenRiskAtLeast(RiskLevel::Medium)
    ->requireAssurance(AuthenticationAssuranceLevel::Strong)
    ->requireFreshAuthentication(minutes: 15)
    ->denyWhenImpersonated();
```

---

# 218. Declarative Config

También podría expresarse en configuración compilable.

```php
return [
    'payout.change' => [
        'risk' => [
            'medium' => 'step_up',
            'high' => 'deny',
        ],
    ],
];
```

---

# 219. Recommended

El Core deberá usar objetos tipados.

Config/DSL solo serán frontends que producen:

```text
Authorization Policy Model
```

---

# 220. No Stringly-Typed Runtime

Evitar runtime basado en:

```text
if ($rule['type'] === ...)
```

en hot paths.

---

# 221. Device Context

Opcionalmente:

```php
final readonly class DeviceContext
{
    public function __construct(
        public DeviceTrustLevel $trust,
        public bool $managed,
        public bool $verified,
        public array $attributes = [],
    ) {}
}
```

---

# 222. Device Trust

```php
enum DeviceTrustLevel: string
{
    case Unknown = 'unknown';
    case Untrusted = 'untrusted';
    case Trusted = 'trusted';
    case Managed = 'managed';
}
```

---

# 223. Device Identity Privacy

No almacenar innecesariamente:

```text
raw fingerprint
hardware identifiers
```

en Authorization logs.

---

# 224. Provider Boundary

El Device provider es responsable de producir:

```text
minimal trust assertion
```

---

# 225. Trusted Device Condition

Ejemplo:

```text
restricted.document.download
```

requiere:

```text
device.trust >= Trusted
```

---

# 226. Managed Device Condition

Operaciones corporativas pueden exigir:

```text
managed=true
```

---

# 227. Location Context

Puede existir como provider opcional.

---

# 228. Important

Authorization Core no deberá requerir geolocalización.

---

# 229. Logical Location

Es preferible modelar:

```text
trusted_region
tenant_allowed_region
network_zone
```

antes que coordenadas precisas cuando sea suficiente.

---

# 230. Geo-Based Conditions

Si una aplicación las necesita deberán tratarse como:

```text
external contextual attributes
```

con provenance.

---

# 231. Privacy by Design

Recolectar únicamente la granularidad necesaria.

---

# 232. Example

Policy necesita:

```text
allowed_country=true
```

No necesariamente:

```text
latitude
longitude
street address
```

---

# 233. Compliance Context

Puede incluir:

```text
data residency
classification
regulatory scope
retention status
legal hold
```

---

# 234. Example

```text
dataset.export
```

puede requerir:

```text
destination region allowed
```

---

# 235. Context from Subject

Algunos atributos pertenecen al recurso.

Ejemplo:

```text
Document.classification=restricted
```

Esto es:

```text
Resource Context
```

---

# 236. Resource Context

```php
final readonly class ResourceAuthorizationContext
{
    public function __construct(
        public array $attributes
    ) {}
}
```

---

# 237. Resource Context Provider

Deberá usar adapters para evitar acoplamiento al ORM.

---

# 238. Example

```text
Payment.amount
Document.classification
Server.environment
Dataset.sensitivity
```

---

# 239. Operation Risk

Risk puede depender del recurso.

Ejemplo:

```text
payment.amount = 50
```

vs:

```text
payment.amount = 5,000,000
```

---

# 240. Risk Is Contextual

No debe existir solo:

```text
User Risk Score
```

Puede ser:

```text
Risk(User, Session, Operation, Resource)
```

---

# 241. Example

Mismo usuario:

```text
profile.view
→ Low Risk

payout.change
→ Elevated Risk
```

---

# 242. Transaction Risk

Aplicaciones financieras podrán conectar:

```text
TransactionRiskProvider
```

---

# 243. Core Neutrality

VoltStack no implementará reglas financieras específicas.

---

# 244. Approval Requirement

Algunas operaciones requieren autorización de otra persona.

---

# 245. Four-Eyes Principle

Ejemplo:

```text
payment > threshold
```

requiere:

```text
maker != approver
```

---

# 246. Approval is not Step-Up

Debe modelarse como requisito diferente.

---

# 247. Approval Challenge

```php
final readonly class ApprovalRequirement
    implements ChallengeRequirementInterface
{
    public function __construct(
        public string $policy,
        public int $requiredApprovals,
    ) {}
}
```

---

# 248. Approval Workflow

Authorization determina:

```text
approval required
```

Workflow subsystem gestiona:

```text
request
approval
rejection
expiration
```

---

# 249. Approval Proof

Al reintentar Authorization:

```text
verified approval proof
```

entra al contexto.

---

# 250. Approval Binding

Debe vincularse a:

```text
operation
resource
scope
parameters/version
```

---

# 251. TOCTOU

Si el recurso cambia después de aprobación:

```text
approval may become invalid
```

---

# 252. Resource Version Binding

Para operaciones críticas:

```text
approval.resource_version
==
current.resource_version
```

---

# 253. Example

Pago aprobado por:

```text
$10,000
```

No debe reutilizarse si cambia a:

```text
$100,000
```

---

# 254. Operation Fingerprint

Podrá existir:

```text
AuthorizationOperationFingerprint
```

---

# 255. Purpose

Vincular:

```text
challenge
approval
confirmation
```

a la operación exacta.

---

# 256. Canonicalization

Debe evitar depender de serialización inestable.

---

# 257. Operation Descriptor

```php
final readonly class AuthorizationOperationDescriptor
{
    public function __construct(
        public string $ability,
        public PrincipalReference $principal,
        public ?SubjectReference $subject,
        public ?AuthorizationScopeReference $scope,
        public array $securityRelevantParameters,
    ) {}
}
```

---

# 258. Sensitive Parameters

Solo incluir parámetros relevantes para authorization.

---

# 259. No Secrets

No introducir:

```text
passwords
tokens
private keys
```

en operation fingerprint.

---

# 260. Risk Escalation from Privilege Changes

Una operación posterior a:

```text
recent role elevation
```

puede considerarse más sensible.

---

# 261. Authorization Event Integration

Documento 15 puede alimentar providers con eventos como:

```text
role assigned
impersonation started
delegation created
credential changed
```

---

# 262. Circular Dependency

Risk provider no deberá depender de ejecutar nuevamente la misma Authorization sin control.

---

# 263. Recursion Guard

AuthorizationContext deberá mantener:

```text
evaluation identity
```

para detectar recursion.

---

# 264. Risk Provider Authorization

Si provider necesita leer datos protegidos:

```text
internal capability
```

deberá usarse explícitamente.

---

# 265. No User Authority Reuse

Un Risk Provider no debe heredar accidentalmente autoridad del usuario para consultar infraestructura interna.

---

# 266. Explainability

Cada condición deberá producir:

```text
reason code
```

---

# 267. Example

```text
authentication.assurance_insufficient
authentication.too_old
session.impersonation_forbidden
device.trust_insufficient
risk.too_high
approval.required
confirmation.required
```

---

# 268. Public Reason

Puede ser:

```text
Additional verification is required.
```

---

# 269. Operator Reason

Puede ser:

```text
Ability payout.change requires Strong assurance.
Current assurance is Standard.
```

---

# 270. Security Reason

Detalles sensibles como:

```text
risk signal internals
fraud indicators
detection thresholds
```

no deberán exponerse al usuario.

---

# 271. Risk Explainability

Operator autorizado puede ver:

```text
RiskLevel=High
Provider=TransactionRiskProvider
```

sin necesariamente ver detalles internos.

---

# 272. Redaction

Audit/Explainability deberán soportar:

```text
redacted metadata
```

---

# 273. Reason Code Taxonomy

Propuesta:

```text
context.missing
context.untrusted
context.provider_failure

authentication.required
authentication.assurance_insufficient
authentication.too_old
authentication.step_up_required

session.invalid
session.expired
session.type_forbidden
session.impersonation_forbidden
session.delegation_forbidden

transport.insecure

device.untrusted
device.unmanaged
device.context_unavailable

network.untrusted
network.context_unavailable

risk.unknown
risk.medium
risk.high
risk.critical
risk.provider_failure
risk.assessment_stale

approval.required
approval.invalid
approval.expired

confirmation.required
confirmation.invalid
confirmation.expired

operation.sensitivity_requirement_failed
```

---

# 274. Audit

Registrar decisiones sensibles:

```text
ability
principal
actor
tenant
scope
subject
outcome
risk level
assurance level
challenge requirements
reason code
```

---

# 275. Do Not Log

Por defecto evitar:

```text
raw credentials
raw tokens
full device fingerprints
sensitive risk telemetry
unnecessary personal information
```

---

# 276. Challenge Audit

Registrar:

```text
challenge issued
challenge satisfied
challenge expired
challenge failed
```

---

# 277. Correlation

Usar:

```text
authorization evaluation ID
challenge ID
request ID
```

---

# 278. Metrics

Ejemplos:

```text
authorization.context.challenge.total
authorization.context.deny.total
authorization.risk.high.total
authorization.step_up.required.total
authorization.risk.provider.failure.total
```

---

# 279. Labels

Mantener baja cardinalidad.

Bueno:

```text
ability_group
outcome
risk_level
```

Malo:

```text
user_id
resource_id
session_id
```

como labels métricos.

---

# 280. Tracing

Span:

```text
authorization.contextual.evaluate
```

Child spans:

```text
context.resolve
risk.evaluate
conditions.evaluate
assurance.evaluate
challenge.normalize
```

---

# 281. Performance Budget

Contextual authorization deberá tener budgets.

Ejemplo:

```text
Structural Authorization:
local

Risk Provider:
possibly external

Device Provider:
possibly external
```

---

# 282. Timeout

Providers externos deberán tener:

```text
explicit timeout
```

---

# 283. No Unlimited Provider Calls

Authorization no deberá bloquear indefinidamente esperando risk evaluation.

---

# 284. Provider Circuit Breaker

Puede integrarse con resilience subsystem.

---

# 285. Security

Circuit breaker abierto no significa:

```text
ALLOW
```

---

# 286. Failure Strategy

La Policy decide:

```text
deny
challenge
failure
```

---

# 287. Bulk Authorization

No realizar risk provider call por recurso cuando el risk es:

```text
session-level
```

---

# 288. Example

Listado de 100 Documents.

Risk puede resolverse:

```text
once per session/request
```

mientras Resource Policy se evalúa individualmente.

---

# 289. Risk Scope

RiskAssessment deberá declarar:

```text
scope
```

conceptualmente:

```text
principal
session
operation
resource
```

---

# 290. Risk Reuse Rules

Solo reutilizar cuando el assessment cubra la operación.

---

# 291. Session Risk

Puede reutilizarse para varias abilities dentro de TTL.

---

# 292. Transaction Risk

No.

---

# 293. Request Memoization

Context providers deberán poder memoizar por:

```text
provider + relevant inputs
```

---

# 294. Persistent Worker Safety

Memoization cross-request:

```text
must be bounded and keyed safely
```

---

# 295. No Static Risk

Prohibido:

```php
static $currentRisk;
```

---

# 296. Policy Cache

Policies compiladas:

```text
safe for worker reuse
```

porque son metadata inmutable.

---

# 297. Context Cache

Context:

```text
not globally reusable by default
```

---

# 298. Testing Strategy

Este subsistema requerirá pruebas específicas.

---

# 299. Authentication Assurance Tests

Cubrir:

```text
sufficient assurance
insufficient assurance
unknown assurance
fresh authentication
stale authentication
```

---

# 300. Challenge Tests

```text
challenge generated
requirements normalized
challenge expires
challenge bound to ability
challenge bound to subject
challenge replay re-authorizes
```

---

# 301. Risk Tests

```text
low
medium
high
critical
unknown
provider unavailable
stale assessment
```

---

# 302. Risk Aggregation Tests

```text
multiple low
low + high
critical authoritative signal
provider conflicts
```

---

# 303. Impersonation Tests

Sensitive operation:

```text
direct session → maybe ALLOW
impersonated session → DENY
```

---

# 304. Delegation Tests

Delegated authority puede tener condiciones más estrictas.

---

# 305. Device Tests

```text
trusted
untrusted
unknown
provider failure
```

---

# 306. Context Provenance Tests

Untrusted client attribute no satisface:

```text
RequiresTrustedDevice
```

---

# 307. Time Tests

Clock controlado:

```text
inside allowed window
outside allowed window
boundary instant
timezone conversion
DST transition
```

---

# 308. Approval Tests

```text
missing approval
valid approval
expired approval
wrong resource
wrong amount/version
same actor when separation required
```

---

# 309. Risk Cannot Grant Property

Property:

```text
If structural authority is DENY,
lowering risk must never produce ALLOW.
```

---

# 310. Assurance Cannot Grant Property

```text
Increasing authentication assurance
must never create an Ability
that the Principal does not possess.
```

---

# 311. Challenge Monotonicity

Si Policy requiere:

```text
Strong
```

un contexto:

```text
VeryStrong
```

debe satisfacerla.

---

# 312. Freshness Monotonicity

Una autenticación más reciente no debe fallar una condición que una más antigua satisface, manteniendo demás inputs iguales.

---

# 313. Risk Restriction Property

Incrementar Risk no debe aumentar authority.

---

# 314. Boundary Property

Agregar una condición de seguridad no debe convertir DENY en ALLOW.

---

# 315. Challenge Replay Property

Un proof para:

```text
Ability A
```

no satisface:

```text
Ability B
```

salvo explicit policy.

---

# 316. Cross-Tenant Challenge Test

Proof de:

```text
Tenant#7
```

no puede utilizarse en:

```text
Tenant#9
```

---

# 317. Cross-Scope Challenge Test

Igualmente:

```text
Workspace#91
≠
Workspace#92
```

---

# 318. Persistent Worker Tests

Request A:

```text
Strong assurance
Low Risk
```

Request B en mismo worker:

```text
Standard assurance
High Risk
```

B jamás debe heredar contexto de A.

---

# 319. Concurrency Tests

Dos evaluaciones concurrentes:

```text
Principal A
Principal B
```

no comparten:

```text
risk
session
challenge
assurance
```

---

# 320. Mutation Tests

Importantes para:

```text
comparison operators
risk thresholds
assurance levels
time boundaries
```

---

# 321. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Contextual/
        ├── Contracts/
        │   ├── AuthorizationConditionInterface.php
        │   ├── AuthorizationContextProviderInterface.php
        │   ├── AuthorizationRiskProviderInterface.php
        │   ├── RiskAggregatorInterface.php
        │   ├── RiskPolicyInterface.php
        │   └── ChallengeRequirementInterface.php
        │
        ├── Context/
        │   ├── AuthenticationContext.php
        │   ├── SessionContext.php
        │   ├── RequestContext.php
        │   ├── EnvironmentContext.php
        │   ├── OperationContext.php
        │   ├── DeviceContext.php
        │   ├── RiskContext.php
        │   ├── ResourceAuthorizationContext.php
        │   └── ContextAttribute.php
        │
        ├── Authentication/
        │   ├── AuthenticationAssuranceLevel.php
        │   ├── AuthenticationFreshness.php
        │   ├── AssuranceEvaluator.php
        │   └── FreshAuthenticationEvaluator.php
        │
        ├── Conditions/
        │   ├── RequiresAssurance.php
        │   ├── RequiresFreshAuthentication.php
        │   ├── RequiresSessionType.php
        │   ├── RequiresSecureTransport.php
        │   ├── RequiresTrustedDevice.php
        │   ├── DisallowImpersonation.php
        │   ├── DisallowDelegation.php
        │   └── ConditionGroup.php
        │
        ├── Risk/
        │   ├── RiskLevel.php
        │   ├── RiskSignal.php
        │   ├── RiskAssessment.php
        │   ├── RiskAssessmentConfidence.php
        │   ├── RiskAggregationStrategy.php
        │   ├── RiskAggregator.php
        │   ├── RiskPolicyRegistry.php
        │   └── RiskProviderFailurePolicy.php
        │
        ├── Challenge/
        │   ├── AuthenticationChallenge.php
        │   ├── ConfirmationChallenge.php
        │   ├── ApprovalRequirement.php
        │   ├── ChallengeRequirementNormalizer.php
        │   ├── AuthorizationChallenge.php
        │   ├── AuthorizationChallengeToken.php
        │   └── ChallengeVerifier.php
        │
        ├── Policy/
        │   ├── ConditionalAccessPolicy.php
        │   ├── ConditionalAccessPolicyRegistry.php
        │   ├── AdaptiveAuthorizationRule.php
        │   ├── AuthorizationSensitivity.php
        │   └── AbilitySensitivityRegistry.php
        │
        ├── Evaluation/
        │   ├── ContextualAuthorizationEvaluator.php
        │   ├── ConditionalPolicyEvaluator.php
        │   ├── AssuranceRequirementEvaluator.php
        │   ├── RiskEvaluator.php
        │   └── ChallengeEvaluator.php
        │
        ├── Metadata/
        │   ├── RequiresAssurance.php
        │   ├── RequiresFreshAuthentication.php
        │   ├── DisallowImpersonation.php
        │   ├── RequiresTrustedDevice.php
        │   └── RequiresApproval.php
        │
        ├── Compilation/
        │   ├── ConditionalPolicyCompiler.php
        │   ├── ConditionTreeCompiler.php
        │   └── ContextRequirementCompiler.php
        │
        └── Exceptions/
            ├── ContextResolutionException.php
            ├── RiskProviderException.php
            ├── InvalidChallengeException.php
            └── ConditionalAuthorizationException.php
```

---

# 322. Integración con Authorization Manager

Flujo:

```text
AuthorizationManager
        ↓
AuthorizationPlanner
        ↓
Structural Authorization
        ↓
ContextualAuthorizationEvaluator
        ↓
RiskEvaluator
        ↓
AssuranceRequirementEvaluator
        ↓
DecisionManager
```

---

# 323. Integración con Policies

Policy tradicional:

```php
public function update(User $user, Document $document): bool
{
    // resource authorization
}
```

VoltStack podrá permitir una Policy avanzada:

```php
public function update(
    User $user,
    Document $document,
    AuthorizationContext $context
): AuthorizationDecision {
    // contextual authorization
}
```

---

# 324. Recomendación

Policies de dominio deberían concentrarse principalmente en:

```text
domain authorization
resource state
ownership
relationships
```

y condiciones reutilizables como:

```text
assurance
risk
device
session
```

deberían vivir en evaluadores dedicados.

---

# 325. Why

Evita repetir:

```php
if (!$session->hasMfa()) {
    ...
}
```

en cientos de Policies.

---

# 326. Integración con Controllers

Ejemplo conceptual:

```php
#[Authorize('organization.delete', subject: 'organization')]
#[RequiresAssurance(AuthenticationAssuranceLevel::VeryStrong)]
#[RequiresFreshAuthentication('PT10M')]
#[DisallowImpersonation]
public function destroy(Organization $organization)
{
}
```

---

# 327. Compiled Plan

```text
organization.delete
    ↓
OrganizationPolicy
    ↓
VeryStrong assurance
    ↓
authentication <= 10 minutes
    ↓
not impersonated
```

---

# 328. Integración con Routes

También:

```php
Route::delete('/organizations/{organization}')
    ->authorize('organization.delete')
    ->requiresAssurance('very_strong');
```

si Routing bridge lo soporta.

---

# 329. Integración con Gates

```php
Gate::define('production.deploy', ...)
    ->requiresAssurance('strong')
    ->maximumRisk('low');
```

conceptualmente.

---

# 330. Integración con Voters

Un Voter puede utilizar:

```text
AuthorizationContext
```

pero no debería consultar providers externos directamente.

---

# 331. Provider Resolution

Debe pasar por:

```text
Contextual Authorization Engine
```

para:

```text
memoization
timeouts
tracing
failure policy
```

---

# 332. Integración con RBAC

Role:

```text
production.operator
```

concede:

```text
production.deploy
```

pero Conditional Policy exige:

```text
Strong assurance
Risk <= Low
Not impersonated
```

---

# 333. Integración con ReBAC

Relationship:

```text
User
owner_of
Repository
```

concede:

```text
repository.secret.rotate
```

pero operación sigue pudiendo requerir:

```text
step-up
```

---

# 334. Integración con Multi-Tenancy

Cada evaluación contextual deberá conservar:

```text
TenantContext
```

---

# 335. Tenant Policy

Tenant podrá configurar:

```text
minimum assurance
risk tolerances
conditional rules
```

sin alterar otros Tenants.

---

# 336. Tenant Configuration Security

Un Tenant Admin no deberá poder desactivar:

```text
platform non-bypassable security rules
```

---

# 337. Policy Layers

Orden conceptual:

```text
Platform Security Policy
        ↓
Tenant Conditional Policy
        ↓
Scope Conditional Policy
        ↓
Ability Policy
        ↓
Resource Policy
```

---

# 338. Restrictive Composition

Por default:

```text
child policy may tighten
```

pero no relajar reglas non-bypassable superiores.

---

# 339. Example

Platform:

```text
Critical operations require Strong+
```

Tenant:

```text
Critical operations require VeryStrong
```

Resultado:

```text
VeryStrong
```

---

# 340. Invalid Relaxation

Tenant intenta:

```text
Critical → Standard
```

Platform minimum:

```text
Strong
```

Resultado efectivo:

```text
Strong
```

---

# 341. Policy Requirement Merge

El engine deberá combinar:

```text
minimum assurance → maximum requirement
maximum risk → minimum tolerance
authentication age → shortest window
```

---

# 342. Deny Rules

Se combinan mediante:

```text
DenyOverrides
```

cuando sean security rules.

---

# 343. Platform Emergency Policy

Puede existir:

```text
EmergencyAuthorizationRestriction
```

para bloquear temporalmente abilities.

---

# 344. Example

```text
Disable:
production.deploy
```

durante incidente.

---

# 345. No Global Mutable Flag

Debe integrarse mediante provider/versioned policy.

---

# 346. Cache Invalidation

Cambiar Conditional Policy deberá incrementar:

```text
authorization_policy_version
```

---

# 347. Tenant Conditional Policy Version

Puede existir:

```text
tenant_contextual_authorization_version
```

---

# 348. Decision Cache

Las decisiones contextuales requieren más cuidado.

---

# 349. Never Cache Blindly

Una decisión:

```text
ALLOW
```

basada en:

```text
Risk Low
Strong Authentication
```

no debe reutilizarse cuando esas señales cambian.

---

# 350. Cacheability Descriptor

Cada decisión podrá declarar:

```php
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

# 351. Vary By

Ejemplos:

```text
session
authentication assurance
authentication time
risk assessment
device trust
policy version
resource version
```

---

# 352. Request Memoization

Normalmente segura si los inputs son inmutables durante la evaluación.

---

# 353. Cross-Request Cache

Solo para Policies que explícitamente lo permitan.

---

# 354. Challenge Cache

No cachear como:

```text
user always needs MFA
```

sin considerar que step-up pudo completarse.

---

# 355. Authorization Decision Reuse

Documento 14 deberá extenderse para considerar:

```text
context fingerprint
```

---

# 356. Context Fingerprint

Deberá contener únicamente:

```text
security-relevant normalized attributes
```

---

# 357. No Raw Sensitive Data

Fingerprint deberá ser:

```text
canonical
minimal
non-reversible when possible
```

---

# 358. Failure Handling

Conditional authorization deberá distinguir:

```text
DENY
CHALLENGE
FAILURE
```

---

# 359. HTTP Mapping

Ejemplo conceptual:

```text
DENY
→ 403 / concealed 404

CHALLENGE
→ authentication/verification response

FAILURE
→ controlled 5xx / security-safe failure
```

---

# 360. API Challenge Response

Podrá devolver metadata estructurada:

```json
{
    "error": "authorization_challenge",
    "requirements": [
        {
            "type": "authentication_assurance",
            "level": "strong"
        }
    ]
}
```

---

# 361. No Internal Risk Details

No devolver:

```json
{
    "risk_score": 0.932817,
    "fraud_rule": "..."
}
```

al cliente por defecto.

---

# 362. SPA Integration

Frontend Runtime podrá recibir:

```text
authorization challenge
```

y abrir:

```text
step-up flow
```

sin perder navegación.

---

# 363. SPA Replay

Después de completar challenge:

```text
original action
```

podrá reintentarse de forma segura.

---

# 364. Mutation Safety

No ejecutar la acción antes de completar Authorization.

---

# 365. No Partial Side Effects

Authorization challenge deberá producirse:

```text
before irreversible side effects
```

---

# 366. Controller Integration

Authorization middleware debe ejecutarse antes del Controller cuando sea posible.

---

# 367. Domain-Level Authorization

Si la decisión depende de datos cargados dentro del dominio:

```text
authorize before mutation
```

---

# 368. TOCTOU

Operaciones críticas podrán requerir:

```text
authorization immediately before commit
```

---

# 369. Transaction Integration

Ejemplo:

```text
begin transaction
load current resource
authorize
mutate
commit
```

---

# 370. Risk Evaluation Inside Transaction

Evitar mantener transacciones DB abiertas durante llamadas externas lentas cuando sea posible.

---

# 371. Two-Phase Strategy

Puede usarse:

```text
pre-authorize contextual requirements
    ↓
begin transaction
    ↓
revalidate critical resource state
    ↓
commit
```

---

# 372. Contextual Authorization Invariants

### Invariante 1

Context nunca crea autoridad inexistente.

### Invariante 2

Risk nunca crea autoridad.

### Invariante 3

Assurance nunca crea autoridad.

### Invariante 4

Las condiciones pueden restringir o cualificar authority existente.

### Invariante 5

Context security-critical debe tener provenance confiable.

---

# 373. Challenge Invariants

### Invariante 1

`CHALLENGE` no equivale a `ALLOW`.

### Invariante 2

Completar un challenge requiere re-autorización.

### Invariante 3

Challenge proofs tienen scope y expiration.

### Invariante 4

Challenge proof no amplía authority.

### Invariante 5

Challenge tokens no son credenciales universales.

---

# 374. Risk Invariants

### Invariante 1

`Unknown` no equivale automáticamente a `Low`.

### Invariante 2

Risk creciente nunca incrementa authority.

### Invariante 3

Risk provider failure crítico no produce ALLOW.

### Invariante 4

Risk assessment posee timestamp.

### Invariante 5

Risk reuse respeta su scope y TTL.

---

# 375. Authentication Invariants

### Invariante 1

Método y assurance son conceptos distintos.

### Invariante 2

Freshness es Policy-specific.

### Invariante 3

Strong authentication no reemplaza RBAC/Policy.

### Invariante 4

Authentication context debe ser verificable.

### Invariante 5

Step-up proof debe estar limitado.

---

# 376. Context Invariants

### Invariante 1

AuthorizationContext es immutable.

### Invariante 2

Context no se comparte accidentalmente entre requests.

### Invariante 3

Client-supplied attributes son untrusted por default.

### Invariante 4

Providers producen atributos con provenance.

### Invariante 5

Solo se resuelve contexto requerido.

---

# 377. Multi-Tenant Invariants

### Invariante 1

Conditional Policies son tenant-isolated.

### Invariante 2

Challenge proof de Tenant A no sirve en Tenant B.

### Invariante 3

Tenant Policy no debilita Platform non-bypassable rules.

### Invariante 4

Risk context no cruza Tenants sin contrato explícito.

### Invariante 5

Context cache keys son tenant-qualified cuando corresponde.

---

# 378. Persistent Runtime Invariants

### Invariante 1

No existen static mutable contexts.

### Invariante 2

Risk state es request/execution scoped.

### Invariante 3

Session context no permanece en worker global.

### Invariante 4

Nested evaluations restauran contexto.

### Invariante 5

FrankenPHP worker reuse no produce context leakage.

---

# 379. Arquitectura general

```text
                       AUTHORIZATION REQUEST
                               │
                               ↓
                         PRINCIPAL / ACTOR
                               │
                               ↓
                         TENANT / SCOPE
                               │
                               ↓
                     STRUCTURAL AUTHORITY
                               │
                ┌──────────────┼──────────────┐
                ↓              ↓              ↓
              RBAC           ReBAC          Policy
                │              │              │
                └──────────────┼──────────────┘
                               ↓
                      RESOURCE AUTHORITY
                               │
                               ↓
                   CONTEXT REQUIREMENT PLAN
                               │
               ┌───────────────┼────────────────┐
               ↓               ↓                ↓
        Authentication       Session        Environment
               │               │                │
               └───────────────┼────────────────┘
                               ↓
                         RISK PROVIDERS
                               │
                               ↓
                        RISK ASSESSMENT
                               │
                               ↓
                   CONDITIONAL POLICY ENGINE
                               │
              ┌────────────────┼────────────────┐
              ↓                ↓                ↓
            ALLOW            DENY           CHALLENGE
                                                │
                                                ↓
                                    Authentication /
                                    Confirmation /
                                    Approval
                                                │
                                                ↓
                                         RE-AUTHORIZE
```

---

# 380. Ejemplo — operación normal

User:

```text
workspace.editor
```

Ability:

```text
document.update
```

Context:

```text
Standard authentication
Low Risk
Normal operation
```

Structural:

```text
GRANT
```

Conditional:

```text
SATISFIED
```

Resultado:

```text
ALLOW
```

---

# 381. Ejemplo — step-up

User:

```text
organization.admin
```

Ability:

```text
api_key.create
```

Structural:

```text
GRANT
```

Context:

```text
Authentication=Standard
```

Policy:

```text
Requires Strong
```

Resultado:

```text
CHALLENGE
```

Requirement:

```text
minimum_assurance=Strong
```

---

# 382. Ejemplo — step-up completado

Authentication subsystem completa verificación.

Nuevo contexto:

```text
Authentication=Strong
last_step_up_at=now
```

Authorization se ejecuta nuevamente.

Structural:

```text
GRANT
```

Conditional:

```text
SATISFIED
```

Resultado:

```text
ALLOW
```

---

# 383. Ejemplo — High Risk

User posee authority:

```text
billing.payout.change
```

Risk:

```text
High
```

Policy:

```text
High Risk
→ DENY
```

Resultado:

```text
DENY
```

Incluso con:

```text
VeryStrong authentication
```

si la Policy así lo define.

---

# 384. Ejemplo — Medium Risk Adaptive Step-Up

Structural:

```text
GRANT
```

Risk:

```text
Medium
```

Base assurance:

```text
Standard
```

Adaptive Policy:

```text
Medium Risk
→ require Strong
```

Resultado:

```text
CHALLENGE
```

---

# 385. Ejemplo — Impersonation

Actor:

```text
SupportAgent#5
```

Principal:

```text
Customer#90
```

Ability:

```text
billing.payout.change
```

Customer podría realizarla normalmente.

Conditional Policy:

```text
DisallowImpersonation
```

Resultado:

```text
DENY
```

---

# 386. Ejemplo — Confidential Workspace

User:

```text
workspace.editor
Workspace#91
```

Workspace:

```text
classification=restricted
```

Conditional Policy:

```text
Restricted workspace
→ Trusted Device
→ Strong Authentication
```

Context:

```text
Device=Unknown
Authentication=Strong
```

Resultado:

```text
CHALLENGE / DENY
```

según si device trust puede satisfacerse dinámicamente.

---

# 387. Ejemplo — Service Principal

```text
DeploymentService
```

Ability:

```text
production.deploy
```

Authority:

```text
GRANT
```

Conditional Policy:

```text
ServicePrincipal
Workload assurance >= Strong
Risk <= Low
```

No se exige:

```text
human MFA
```

---

# 388. Ejemplo — Approval

Manager posee:

```text
payment.approve
```

Payment:

```text
amount=500,000
```

Policy:

```text
amount > 100,000
→ requires second approver
```

Resultado:

```text
CHALLENGE
```

Requirement:

```text
approval.required
```

---

# 389. Ejemplo — Resource Changed

Approval fue emitido para:

```text
Payment#100
amount=500,000
version=4
```

Antes de ejecución:

```text
amount=700,000
version=5
```

Approval proof:

```text
resource_version=4
```

Resultado:

```text
approval.invalid
→ CHALLENGE/DENY
```

---

# 390. Ejemplo — Provider Failure

Ability:

```text
production.secret.rotate
```

Policy:

```text
risk assessment mandatory
```

Risk Provider:

```text
unavailable
```

Nunca:

```text
ALLOW
```

Resultado según policy:

```text
FAILURE
```

o:

```text
DENY
```

---

# 391. Ejemplo — Low-Risk operation avoids provider

Ability:

```text
profile.avatar.view
```

Compiled Plan:

```text
risk context not required
```

Entonces:

```text
Risk Provider
→ not invoked
```

Resultado más rápido y con menor recolección de contexto.

---

# 392. Modelo final

VoltStack deberá evolucionar desde:

```text
User
    ↓
Role
    ↓
Permission
    ↓
true / false
```

hacia:

```text
Principal
    ↓
Tenant
    ↓
Scope
    ↓
Ability
    ↓
Resource
    ↓
RBAC / ABAC / ReBAC / Policy
    ↓
Structural Authority
    ↓
Authentication Context
    ↓
Session Context
    ↓
Environment Context
    ↓
Operation Sensitivity
    ↓
Risk Assessment
    ↓
Conditional Policies
    ↓
Assurance Requirements
    ↓
ALLOW / DENY / CHALLENGE / FAILURE
```

---

# 393. Filosofía arquitectónica

VoltStack deberá adoptar los siguientes principios:

```text
Authority is necessary but may not be sufficient.

Context qualifies how authority may be exercised.

Risk may restrict authority but never create it.

Authentication assurance is independent from permissions.

Sensitive operations may require stronger assurance.

Step-up authentication is an authorization outcome,
not an authorization bypass.

Security context must have provenance.

Unknown security state must not silently become trusted state.

Context should be resolved lazily.

Risk providers are pluggable.

Challenges must be scoped and short-lived.

Completing a challenge requires re-authorization.

Platform security requirements cannot be weakened
by lower policy layers.

Persistent workers must never retain mutable authorization context.
```

---

# 394. Resultado esperado

Con `23_AUTHORIZATION_CONDITIONAL_CONTEXTUAL_AND_RISK_BASED_ACCESS_SYSTEM.md`, VoltStack podrá soportar de forma nativa escenarios como:

```text
Step-Up Authentication

Adaptive MFA

Authentication Assurance Levels

Fresh Authentication

Risk-Based Authorization

Context-Aware Policies

Sensitive Operations

Trusted Device Requirements

Session Restrictions

Impersonation Restrictions

Delegation Restrictions

Time-Based Access

Network-Based Conditions

Conditional Access

Approval Requirements

Four-Eyes Principle

Explicit Confirmation

Service Identity Assurance

Tenant-Specific Security Policies

Scope-Specific Security Policies

Resource Classification Policies

Dynamic Authorization Challenges
```

sin mezclar estas responsabilidades dentro de Controllers, Middlewares o Policies monolíticas.

La arquitectura definitiva será:

```text
STRUCTURAL AUTHORITY
        ↓
CONTEXT
        ↓
RISK
        ↓
ASSURANCE
        ↓
CONDITIONS
        ↓
SECURITY REQUIREMENTS
        ↓
FINAL AUTHORIZATION OUTCOME
```

El principio definitivo será:

> **En VoltStack, tener permiso para realizar una operación no significa necesariamente que pueda ejecutarse bajo cualquier circunstancia. La autoridad establece qué puede hacer un Principal; el contexto, el riesgo y el nivel de garantía determinan cuándo y bajo qué condiciones puede ejercer esa autoridad.**

De esta manera, el Authorization System podrá evolucionar desde un sistema tradicional de permisos hacia un **motor de autorización adaptativo**, manteniendo al mismo tiempo separación de responsabilidades, extensibilidad, explicabilidad, seguridad multi-tenant, compatibilidad con FrankenPHP y un runtime optimizable.