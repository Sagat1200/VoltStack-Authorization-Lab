# VoltStack Authorization System — Audit, Observability, Tracing and Explainability System

## 1. Propósito

Este documento define el subsistema de **auditoría, observabilidad, tracing y explicabilidad** del Authorization System de VoltStack.

El objetivo es permitir responder, de forma segura y verificable:

```text
¿Quién intentó realizar una operación?

¿Qué Ability se evaluó?

¿Sobre qué Subject?

¿En qué Tenant?

¿Qué Policies, Gates, Voters o evaluadores participaron?

¿Qué decisiones produjo cada uno?

¿Qué estrategia se utilizó?

¿Qué componente fue decisivo?

¿Por qué se concedió o denegó acceso?

¿Cuánto costó la evaluación?

¿La decisión fue calculada o reutilizada desde cache?

¿Existieron fallos internos?

¿Debe persistirse esta decisión para auditoría?
```

El subsistema deberá proporcionar observabilidad profunda sin convertir:

```text
tracing
logging
profiling
audit
```

en parte de la semántica normal de autorización.

La regla fundamental será:

```text
Authorization decides.

Observability explains.

Audit records.

Neither may silently change the decision.
```

---

# 2. Separación de responsabilidades

VoltStack distinguirá explícitamente cuatro conceptos.

```text
Tracing
=
qué ocurrió durante una evaluación concreta
```

```text
Observability
=
cómo se comporta el Authorization System agregado
```

```text
Explainability
=
por qué se obtuvo una decisión
```

```text
Audit
=
qué hechos de seguridad deben conservarse
```

Estos conceptos podrán compartir infraestructura, pero no deberán confundirse.

---

# 3. Arquitectura general

```text
AuthorizationRequest
        ↓
AuthorizationPlanner
        ↓
AuthorizationExecution
        │
        ├── Trace Collector
        ├── Metrics
        ├── Profiler
        └── Audit Candidate
        ↓
DecisionManager
        ↓
Final DecisionResult
        │
        ├── Explain Model
        ├── Audit Record
        └── Observability Events
```

---

# 4. Principio de no interferencia

El sistema deberá garantizar:

```text
Tracing OFF
```

y:

```text
Tracing ON
```

producen la misma decisión.

Igualmente:

```text
Profiler OFF / ON
Audit OFF / ON
Metrics OFF / ON
```

no deberán cambiar semántica, salvo cuando una operación esté explícitamente configurada como:

```text
must_audit=true
```

y la imposibilidad de auditar sea en sí misma una condición de seguridad.

---

# 5. AuthorizationTrace

Cada ejecución podrá crear un:

```text
AuthorizationTrace
```

request-local.

Conceptualmente:

```php
final class AuthorizationTrace
{
    public function __construct(
        public readonly string $traceId,
        public readonly string $correlationId,
    ) {}
}
```

---

# 6. Trace ID

Cada autorización podrá poseer:

```text
trace_id
```

único.

Ejemplo:

```text
authz_01K2N3...
```

---

# 7. Correlation ID

Una request HTTP, Job o Command puede contener múltiples autorizaciones.

Todas podrán compartir:

```text
correlation_id
```

para reconstruir el flujo completo.

---

# 8. Trace vs Correlation

```text
Correlation ID
=
unidad externa de trabajo
```

```text
Authorization Trace ID
=
una decisión específica
```

---

# 9. Ejemplo

Request:

```text
correlation_id=req_abc
```

Puede contener:

```text
trace authz_1 → route access
trace authz_2 → invoice.view
trace authz_3 → invoice.export
```

---

# 10. AuthorizationTraceContext

Podrá contener:

```text
trace ID
correlation ID
parent trace ID
depth
channel
tenant
actor
effective principal
```

según configuración.

---

# 11. Nested Authorization

Cuando una Policy invoque una autorización secundaria:

```text
Outer Authorization
        ↓
Nested Authorization
```

el trace deberá conservar relación parent-child.

---

# 12. Example Nested Trace

```text
authz_100
InvoicePolicy::approve
    ↓
    authz_101
    CustomerPolicy::manage
```

---

# 13. AuthorizationExecutionTrace

Representará el flujo completo.

Podrá registrar:

```text
Request normalization
Plan resolution
Plan cache
Evaluator sequence
Decision aggregation
Finalization
Decision cache
Audit emission
```

---

# 14. Trace Stages

Propuesta:

```php
enum AuthorizationTraceStage: string
{
    case Request = 'request';
    case Planning = 'planning';
    case Resolution = 'resolution';
    case Evaluation = 'evaluation';
    case Decision = 'decision';
    case Finalization = 'finalization';
    case Cache = 'cache';
    case Audit = 'audit';
}
```

---

# 15. Trace Event

Conceptualmente:

```php
final readonly class AuthorizationTraceEvent
{
    public function __construct(
        public AuthorizationTraceStage $stage,
        public string $event,
        public int $timestampNs,
        public array $metadata = [],
    ) {}
}
```

---

# 16. Event Examples

```text
authorization.request.created
authorization.plan.cache_hit
authorization.plan.built
authorization.evaluator.started
authorization.evaluator.completed
authorization.evaluator.failed
authorization.strategy.short_circuit
authorization.decision.finalized
authorization.audit.persisted
```

---

# 17. Evaluator Trace

Por cada evaluator podrán registrarse:

```text
evaluator ID
type
priority
requirement
invocation target
execution status
decision
reason code
duration
cache source
```

---

# 18. Example Evaluator Trace

```text
Evaluator:
tenant_isolation

Type:
security

Priority:
9000

Status:
COMPLETED

Decision:
GRANT

Duration:
0.08 ms
```

---

# 19. Failed Evaluator Trace

```text
Evaluator:
external_risk

Status:
FAILED

Failure Class:
ExternalEvaluatorTimeout

Decision:
none

Failure Policy:
fail_closed
```

---

# 20. Error Details

En desarrollo pueden incluirse:

```text
exception class
safe message
stack reference
```

En producción:

```text
internal details redacted
```

---

# 21. Decision Trace

El DecisionManager deberá poder registrar:

```text
strategy
vote counts
short-circuit
decisive evaluator
primary reason
final decision
```

---

# 22. Example

```text
Strategy:
deny_overrides

Votes:
GRANT=3
DENY=1
ABSTAIN=2

Short Circuit:
yes

Decisive Evaluator:
compliance.invoice_approval

Final:
DENY
```

---

# 23. Explainability Model

No se recomienda construir explicaciones humanas directamente durante cada evaluator.

Primero deberá generarse una representación estructurada:

```text
AuthorizationExplanation
```

---

# 24. AuthorizationExplanation

Conceptualmente:

```php
final readonly class AuthorizationExplanation
{
    public function __construct(
        public Decision $decision,
        public string $reasonCode,
        public array $factors,
        public ?string $decisiveEvaluatorId,
        public string $strategy,
    ) {}
}
```

---

# 25. Structured First

Regla:

```text
structured explanation
        ↓
human formatter
```

No:

```text
arbitrary human strings
        ↓
attempt to infer structure
```

---

# 26. Explanation Factor

Cada factor puede representar:

```text
Role requirement
Permission requirement
Tenant rule
Relationship
Resource Policy
ABAC rule
Security requirement
```

---

# 27. Example

```text
Decision:
DENY

Reason:
invoice.approval_limit_exceeded

Factors:
✓ Tenant matched
✓ Permission invoice.approve
✓ Organization relationship
✗ Approval amount exceeds limit
```

---

# 28. Human Explanation

Una capa de presentación podrá convertirlo en:

```text
Access was denied because the approval amount
exceeds the Principal's configured limit.
```

---

# 29. Internal vs External Explanation

VoltStack deberá distinguir:

```text
Internal Explanation
```

y:

```text
Public Explanation
```

---

# 30. Internal Explanation

Puede incluir:

```text
Policy IDs
specific reason codes
strategy
safe metadata
scope mismatch
graph path
```

---

# 31. Public Explanation

Debe ser mucho más limitada.

Ejemplo:

```text
You are not allowed to perform this action.
```

---

# 32. Reason Code

Los `reasonCode` serán el principal mecanismo de explicación estable.

Ejemplos:

```text
authorization.denied
authorization.no_decision

rbac.permission_missing
tenant.mismatch
tenant.membership_suspended
invoice.locked
invoice.approval_limit_exceeded
security.mfa_required
rebac.relationship_missing
```

---

# 33. Stable Reason Codes

Los códigos deberán ser:

```text
machine-readable
stable
non-sensitive
```

No usar mensajes humanos como IDs.

---

# 34. Reason Registry

Podrá existir:

```text
AuthorizationReasonRegistry
```

para:

```text
documentation
translation
public mapping
severity
audit category
```

---

# 35. Reason Descriptor

Conceptualmente:

```php
final readonly class AuthorizationReasonDescriptor
{
    public function __construct(
        public string $code,
        public AuthorizationReasonCategory $category,
        public AuthorizationReasonVisibility $visibility,
    ) {}
}
```

---

# 36. Reason Categories

Ejemplo:

```php
enum AuthorizationReasonCategory: string
{
    case Authentication = 'authentication';
    case Permission = 'permission';
    case Role = 'role';
    case Tenant = 'tenant';
    case Resource = 'resource';
    case Compliance = 'compliance';
    case Security = 'security';
    case Relationship = 'relationship';
    case System = 'system';
}
```

---

# 37. Reason Visibility

```php
enum AuthorizationReasonVisibility: string
{
    case Internal = 'internal';
    case Safe = 'safe';
    case Public = 'public';
}
```

---

# 38. Example

```text
tenant.mismatch
visibility=INTERNAL
```

Puede mapearse externamente a:

```text
resource_not_found
```

sin revelar que pertenece a otro tenant.

---

# 39. Reason Precedence

Si múltiples evaluadores deniegan:

```text
TenantIsolation → DENY
Compliance → DENY
```

el DecisionManager selecciona una razón primaria según reglas definidas.

---

# 40. Preserve Secondary Reasons

El trace puede conservar:

```text
all_decisive_reasons
```

aunque el resultado público use una sola.

---

# 41. Explanation Tree

Para evaluaciones complejas podrá generarse:

```text
AuthorizationExplanationTree
```

---

# 42. Example Tree

```text
DENY invoice.approve
│
├── GRANT tenant
│   ├── membership active
│   └── subject tenant matches
│
├── GRANT RBAC
│   ├── role finance-manager
│   └── permission invoice.approve
│
├── GRANT ReBAC
│   └── user member organization 15
│
└── DENY ABAC
    └── amount exceeds approval limit
```

---

# 43. Tree Depth

Debe limitarse en tracing para evitar explosión de memoria.

---

# 44. Max Trace Depth

Configuración:

```text
authorization.trace.max_depth
```

---

# 45. Max Trace Events

Igualmente:

```text
authorization.trace.max_events
```

---

# 46. Truncation

Si se supera:

```text
trace_truncated=true
```

deberá quedar indicado.

---

# 47. No Semantic Impact

Un trace truncado no cambia la decisión.

---

# 48. Audit

Audit es diferente a tracing.

No todas las decisiones deben persistirse.

---

# 49. AuthorizationAuditRecord

Conceptualmente:

```php
final readonly class AuthorizationAuditRecord
{
    public function __construct(
        public string $id,
        public string $traceId,
        public PrincipalReference $principal,
        public ?PrincipalReference $actor,
        public string $ability,
        public ?SubjectAuditReference $subject,
        public ?TenantReference $tenant,
        public Decision $decision,
        public ?string $reasonCode,
        public string $strategy,
        public DateTimeImmutable $occurredAt,
        public array $metadata = [],
    ) {}
}
```

---

# 50. Audit Record Minimalism

No deberá persistirse automáticamente:

```text
full request
full model
all headers
full Policy arguments
```

---

# 51. Audit Subject Reference

Preferir:

```text
type=invoice
id=928
```

No serializar la entidad completa.

---

# 52. Sensitive Identifiers

La aplicación podrá hash/redactar IDs cuando sea requerido.

---

# 53. Actor vs Effective Principal

En impersonation:

```text
Actor:
SupportAgent#10

Principal:
User#42
```

ambos deberán conservarse.

---

# 54. Audit Scope

Puede configurarse por:

```text
Ability
risk level
evaluator
decision
channel
tenant
resource type
```

---

# 55. Audit Modes

Propuesta:

```php
enum AuthorizationAuditMode: string
{
    case None = 'none';
    case Denials = 'denials';
    case Grants = 'grants';
    case All = 'all';
    case Critical = 'critical';
}
```

---

# 56. Recommended Default

Para la mayoría de aplicaciones:

```text
DENY critical/security events
+
selected GRANT operations
```

No necesariamente todos los `view` exitosos.

---

# 57. Why Not Audit Everything

En una aplicación grande:

```text
@can('view', $resource)
```

puede ejecutarse miles de veces.

Persistir cada check produciría:

```text
huge write volume
noise
storage cost
privacy risk
```

---

# 58. Critical Audit

Abilities como:

```text
permission.grant
role.assign
user.impersonate
system.deploy
invoice.approve
tenant.support
data.export
```

pueden requerir audit de GRANT y DENY.

---

# 59. Ability Metadata

`AbilityDescriptor` podrá incluir:

```text
auditMode
auditSeverity
```

---

# 60. Policy Audit Hint

Un evaluator podrá adjuntar metadata como:

```text
audit_category=financial_approval
```

pero no decidir por sí mismo el backend de audit.

---

# 61. AuditPolicyResolver

Podrá combinar:

```text
Ability metadata
Global config
Decision
Evaluator metadata
Request context
```

para decidir si persiste.

---

# 62. Audit Severity

Ejemplo:

```php
enum AuthorizationAuditSeverity: string
{
    case Info = 'info';
    case Notice = 'notice';
    case Warning = 'warning';
    case Critical = 'critical';
}
```

---

# 63. Example

Cross-tenant attempt:

```text
severity=CRITICAL
```

Missing optional permission:

```text
severity=INFO
```

según aplicación.

---

# 64. Audit Backend

Contrato:

```php
interface AuthorizationAuditSinkInterface
{
    public function write(
        AuthorizationAuditRecord $record
    ): void;
}
```

---

# 65. Possible Sinks

```text
Database
Structured Log
SIEM
Message Queue
File
External Compliance Service
Multiple sinks
```

---

# 66. Composite Sink

Podrá existir:

```text
CompositeAuthorizationAuditSink
```

---

# 67. Async Audit

Para reducir latencia, el audit podrá enviarse a queue.

---

# 68. But Critical Operations

Cuando:

```text
must_audit=true
```

una operación puede requerir confirmación de persistencia antes de continuar.

---

# 69. Audit Guarantees

Propuesta:

```php
enum AuditDeliveryGuarantee: string
{
    case BestEffort = 'best_effort';
    case AtLeastOnce = 'at_least_once';
    case Required = 'required';
}
```

---

# 70. Best Effort

Audit failure:

```text
log locally
continue decision
```

---

# 71. At Least Once

Utilizar:

```text
transactional outbox
durable queue
```

cuando se requiera.

---

# 72. Required

Si no puede garantizarse audit:

```text
operation must not proceed
```

---

# 73. Important Architecture

`Required` no debería implementarse como un side effect oculto del logger.

Debe formar parte explícita de la política operacional de la Ability.

---

# 74. Transactional Audit

Para operaciones de dominio críticas:

```text
domain mutation
+
audit outbox record
```

podrán escribirse en la misma transacción.

---

# 75. Decision Audit vs Business Audit

Debe distinguirse:

```text
Authorization Audit:
User was allowed to approve Invoice#928
```

de:

```text
Business Audit:
Invoice#928 was approved
```

Son eventos diferentes.

---

# 76. Authorization GRANT Does Not Mean Action Happened

Después del GRANT:

```text
Controller may fail
transaction may rollback
user may cancel
```

Por tanto no registrar:

```text
invoice approved
```

desde Authorization.

---

# 77. Correct

Authorization audit:

```text
approval authorization granted
```

Business subsystem:

```text
invoice approval committed
```

---

# 78. Decision Attempt Audit

Incluso si la operación no ocurre, puede ser relevante auditar:

```text
attempted authorization
```

---

# 79. Security Event

Un DENY puede generar un:

```text
AuthorizationSecurityEvent
```

separado del audit general.

---

# 80. Security Event Examples

```text
cross-tenant access attempt
repeated permission escalation
disabled account access
impersonation violation
invalid support session
critical evaluator failure
```

---

# 81. Security Event Sink

Podrá integrarse con:

```text
Security Event System
SIEM
alerts
incident management
```

---

# 82. Alerting

Authorization Core no deberá enviar correos/SMS directamente.

Emitirá eventos estructurados.

---

# 83. Observability Metrics

El sistema deberá soportar métricas agregadas.

---

# 84. Core Metrics

Ejemplos:

```text
authorization.requests.total
authorization.decisions.grant
authorization.decisions.deny
authorization.decisions.default_deny
authorization.decisions.all_abstain
```

---

# 85. Evaluator Metrics

```text
authorization.evaluator.executions
authorization.evaluator.failures
authorization.evaluator.abstains
authorization.evaluator.duration
```

---

# 86. Strategy Metrics

```text
authorization.strategy.deny_overrides
authorization.strategy.unanimous
authorization.strategy.consensus
authorization.short_circuit.total
```

---

# 87. Cache Metrics

```text
authorization.cache.plan.hit
authorization.cache.plan.miss
authorization.memo.hit
authorization.decision_cache.hit
authorization.decision_cache.miss
```

---

# 88. Tenant Metrics

```text
authorization.tenant.denies
authorization.tenant.membership_denies
authorization.cross_tenant.attempts
```

---

# 89. RBAC Metrics

```text
authorization.rbac.permission_checks
authorization.rbac.role_checks
authorization.rbac.grant_cache_hits
```

---

# 90. ReBAC Metrics

```text
authorization.rebac.lookups
authorization.rebac.path_depth
authorization.rebac.failures
```

---

# 91. ABAC Metrics

```text
authorization.abac.rules
authorization.abac.denies
authorization.abac.attribute_provider_latency
```

---

# 92. Cardinality Control

No usar labels de alta cardinalidad como:

```text
user_id
invoice_id
full URL
```

en métricas.

---

# 93. Safe Metric Labels

Preferir:

```text
ability
evaluator_type
decision
strategy
channel
```

y aun `ability` puede requerir límites si es dinámica.

---

# 94. Tenant ID in Metrics

Normalmente no debe usarse como label global por cardinalidad.

Puede registrarse en logs/traces cuando sea necesario.

---

# 95. Metrics Backend

Authorization utilizará la infraestructura general de observabilidad de VoltStack.

No implementará un sistema propio de Prometheus/OpenTelemetry.

---

# 96. OpenTelemetry Integration

Podrá crear spans como:

```text
authorization.check
authorization.evaluator
authorization.plan
```

---

# 97. Root Span

Ejemplo:

```text
authorization.check
```

attributes:

```text
authz.ability
authz.decision
authz.strategy
authz.evaluator_count
authz.cache
```

---

# 98. Evaluator Spans

Podrán crearse solo en:

```text
debug/profiling mode
```

para evitar overhead.

---

# 99. Sensitive OTel Attributes

No incluir:

```text
full Subject payload
reason containing PII
authorization token
raw session data
```

---

# 100. AuthorizationProfiler

VoltStack podrá proporcionar un profiler específico.

---

# 101. Profile Data

Podrá mostrar:

```text
planning time
policy resolution
gate resolution
permission lookup
relationship lookup
evaluator durations
decision aggregation
cache hits
```

---

# 102. Example Profiler Output

```text
Authorization: invoice.approve

Total                2.31 ms

Plan cache           HIT
Tenant isolation     0.05 ms
Permission check     0.11 ms
Relationship check   0.80 ms
InvoicePolicy        0.31 ms
CompliancePolicy     0.92 ms
Decision manager     0.04 ms
```

---

# 103. Slow Evaluator Detection

Configuración:

```text
authorization.profiler.slow_evaluator_ms
```

---

# 104. Warning

```text
FinancialCompliancePolicy
took 42.8 ms
```

---

# 105. N+1 Detection

Profiler podrá detectar:

```text
same permission evaluated 200 times
without memoization
```

---

# 106. Example Warning

```text
Potential authorization N+1:

permission invoice.view
requested 500 times
source queries 500
```

---

# 107. Relationship N+1

Igualmente:

```text
organization membership lookup
```

---

# 108. Plan Rebuild Detection

Si un plan debería estar cacheado pero se reconstruye:

```text
plan cache miss rate high
```

podrá advertirse.

---

# 109. Reflection Detection

En profiling development:

```text
runtime reflection used during authorization
```

puede marcarse.

---

# 110. Filesystem Scan Detection

No debería ocurrir en producción.

---

# 111. AuthorizationExplainService

Contrato conceptual:

```php
interface AuthorizationExplainServiceInterface
{
    public function explain(
        AuthorizationExecution $execution,
    ): AuthorizationExplanation;
}
```

---

# 112. Explain Existing Decision

Preferido:

```text
execution
        ↓
explanation
```

sobre volver a ejecutar Policies para explicar.

---

# 113. Why

Re-ejecutar puede:

```text
change state
call external systems again
produce different answer
increase cost
```

---

# 114. Explain Fresh Simulation

En tooling development puede existir una ejecución simulada, pero deberá estar claramente diferenciada.

---

# 115. Explain Modes

```php
enum AuthorizationExplainMode: string
{
    case Recorded = 'recorded';
    case Live = 'live';
    case Structural = 'structural';
}
```

---

# 116. Recorded

Explica una ejecución existente.

---

# 117. Live

Ejecuta una evaluación actual para debugging.

---

# 118. Structural

Explica:

```text
qué evaluadores participarían
```

sin ejecutarlos.

---

# 119. Structural Example

```text
Ability:
invoice.approve

Would evaluate:
1 TenantIsolationPolicy
2 PermissionEvaluator
3 RelationshipEvaluator
4 InvoicePolicy
5 CompliancePolicy
```

---

# 120. Explain Permissions

El RBAC subsystem podrá aportar:

```text
permission granted directly
```

o:

```text
permission inherited via role finance-manager
```

---

# 121. Role Explanation

Ejemplo:

```text
Role finance-manager
assigned in Tenant#7
active until 2026-12-31
```

si esa metadata es safe para el operador.

---

# 122. ReBAC Explanation

Puede aportar:

```text
relationship path matched
```

---

# 123. Path Example

```text
User#42
member_of
Organization#15

Organization#15
owns
Invoice#928
```

---

# 124. Path Redaction

Para output público:

```text
required relationship was not satisfied
```

---

# 125. ABAC Explanation

Puede mostrar:

```text
approval limit check failed
```

sin mostrar montos sensibles.

---

# 126. Security Evaluation Explanation

Ejemplo:

```text
MFA level insufficient
```

Puede ser safe o internal según política.

---

# 127. Public Error Mapping

El HTTP layer podrá mapear:

```text
security.mfa_required
```

a una respuesta específica si se desea.

---

# 128. No Automatic Leakage

No todo reason code debe aparecer al cliente.

---

# 129. Explanation Audience

Podrá distinguirse:

```php
enum AuthorizationExplanationAudience: string
{
    case Internal = 'internal';
    case Developer = 'developer';
    case Operator = 'operator';
    case EndUser = 'end_user';
}
```

---

# 130. Audience-specific Formatting

Mismo trace:

```text
DENY tenant.mismatch
```

Developer:

```text
Subject Tenant#9 does not match TenantContext#7.
```

End user:

```text
Resource not found.
```

---

# 131. AuthorizationExplanationFormatter

Contrato:

```php
interface AuthorizationExplanationFormatterInterface
{
    public function format(
        AuthorizationExplanation $explanation,
        AuthorizationExplanationAudience $audience,
    ): mixed;
}
```

---

# 132. Localization

Mensajes humanos podrán pasar por:

```text
Translator
```

usando:

```text
reasonCode
```

como clave.

---

# 133. Core Language Independence

El Core no almacenará mensajes traducidos.

---

# 134. Audit Localization

Audit deberá almacenar:

```text
reasonCode
```

no una traducción dependiente de idioma.

---

# 135. Authorization Trace Storage

Los traces completos no necesariamente deben persistirse.

---

# 136. Trace Modes

```php
enum AuthorizationTraceMode: string
{
    case Off = 'off';
    case Errors = 'errors';
    case Sampled = 'sampled';
    case Full = 'full';
}
```

---

# 137. Off

Hot path mínimo.

---

# 138. Errors

Solo fallos y decisiones relevantes.

---

# 139. Sampled

Ejemplo:

```text
1% of normal GRANT
100% of critical DENY
```

---

# 140. Full

Desarrollo/debug.

---

# 141. Sampling

No deberá muestrear aleatoriamente eventos de auditoría obligatoria.

---

# 142. Trace Sampling ≠ Audit Sampling

Audit mandatory siempre se conserva.

Tracing diagnóstico sí puede samplearse.

---

# 143. Sampling Strategy

Podrá considerar:

```text
decision
ability risk
evaluator failure
duration
tenant mismatch
```

---

# 144. Tail Sampling

Un trace inicialmente ligero puede promoverse a completo si:

```text
DENY
slow authorization
failure
```

pero esto requiere buffering.

No es requisito de V1.

---

# 145. Memory Overhead

Tracing completo deberá usar estructuras compactas.

---

# 146. Lazy Metadata

Información costosa solo se construirá cuando tracing esté activo.

---

# 147. No `debug_backtrace()` on Hot Path

Salvo debugging explícito.

---

# 148. Stopwatch

Medición de duración podrá utilizar:

```text
hrtime(true)
```

o abstracción equivalente.

---

# 149. Nano vs Milliseconds

Internamente:

```text
nanoseconds
```

externamente:

```text
milliseconds
```

---

# 150. AuthorizationObserver

Podrá existir un mecanismo de eventos observacionales.

```php
interface AuthorizationObserverInterface
{
    public function authorizationStarted(...): void;

    public function evaluatorCompleted(...): void;

    public function authorizationCompleted(...): void;
}
```

---

# 151. Observer Restrictions

Observers:

```text
cannot change DecisionResult
```

---

# 152. Observer Failure

Por defecto:

```text
observability observer failure
        ↓
log
        ↓
continue
```

---

# 153. Critical Audit Observer

No debe modelarse como observer normal si su fallo debe bloquear la operación.

Usar `AuditDeliveryGuarantee::Required`.

---

# 154. Event Dispatcher

Authorization podrá emitir eventos hacia el Event System.

---

# 155. Example Events

```text
AuthorizationGranted
AuthorizationDenied
AuthorizationEvaluatorFailed
AuthorizationCriticalDenial
AuthorizationAuditRequired
```

---

# 156. Avoid Event Storms

No emitir eventos de dominio pesados por cada `@can()`.

---

# 157. Event Policy

Configuración por:

```text
ability
decision
risk
```

---

# 158. Synchronous vs Async

Eventos observacionales normalmente podrán ser async.

---

# 159. Security Event

Ciertas denegaciones críticas podrán emitirse sincrónicamente al security pipeline, pero sin modificar la decisión salvo regla explícita.

---

# 160. Log Structure

Logs deberán ser estructurados.

Ejemplo:

```json
{
    "event": "authorization.denied",
    "trace_id": "authz_123",
    "ability": "invoice.approve",
    "decision": "deny",
    "reason_code": "invoice.approval_limit_exceeded",
    "strategy": "deny_overrides"
}
```

---

# 161. Optional Context

Podrá incluir:

```text
principal type
subject type
tenant reference
channel
```

según política de privacidad.

---

# 162. No Sensitive Object Dumping

Nunca:

```php
logger()->debug($authorizationRequest);
```

si eso serializa tokens, payloads o entidades completas.

---

# 163. Redaction System

VoltStack deberá proporcionar:

```text
AuthorizationRedactor
```

---

# 164. Redaction Responsibilities

Eliminar o transformar:

```text
access tokens
session IDs
passwords
secrets
PII
sensitive subject attributes
internal exception messages
```

---

# 165. Redaction Policies

Podrán clasificarse campos:

```text
public
internal
sensitive
secret
```

---

# 166. AuthorizationMetadataRedactor

Podrá procesar metadata de traces y audit.

---

# 167. Safe-by-Default

Metadata no reconocida no debería publicarse automáticamente.

---

# 168. Allowlist Strategy

Para output externo:

```text
allowlist
```

es preferible a:

```text
blocklist
```

---

# 169. Trace Redaction

Internamente puede almacenarse más información, pero aun así deberá evitar secretos.

---

# 170. Audit Privacy

Audit tiene requisitos de retención y privacidad.

---

# 171. Data Minimization

Persistir solo lo necesario para:

```text
security
compliance
forensics
accountability
```

---

# 172. Retention

Podrá definirse:

```text
critical audit: 7 years
normal denial: 90 days
debug trace: 7 days
```

según aplicación.

---

# 173. Retention Is Not Core Hardcoded

El Authorization System expondrá metadata/categorías; el Audit/Storage subsystem gestiona lifecycle.

---

# 174. Legal Hold

Sistemas empresariales podrán impedir eliminación de ciertos records.

---

# 175. Audit Integrity

Para auditorías críticas podrá requerirse:

```text
append-only storage
hash chaining
WORM storage
signed records
```

---

# 176. Tamper Evidence

Futuro:

```text
audit record hash
previous hash
```

puede crear una cadena verificable.

---

# 177. Audit ID

Cada record deberá tener ID estable.

---

# 178. Clock

Utilizar:

```text
ClockInterface
```

para timestamps testables.

---

# 179. UTC

Audit timestamps deberán guardarse preferentemente en UTC.

---

# 180. Sequence Ordering

En distributed systems, timestamp por sí solo no garantiza orden absoluto.

Puede conservarse:

```text
correlation ID
sequence number
```

por ejecución.

---

# 181. AuthorizationTrace Sequence

Cada event puede tener:

```text
sequence=1,2,3...
```

---

# 182. Profiler UI

VoltStack Dev Tools podrá mostrar una sección:

```text
Authorization
```

por request.

---

# 183. UI Example

```text
Authorization checks: 7
Granted: 5
Denied: 2
Plan cache hits: 6
Decision memo hits: 18
Slow evaluators: 1
```

---

# 184. Check Detail

Al abrir:

```text
invoice.approve
```

mostrar:

```text
Principal
Tenant
Subject
Plan
Evaluators
Votes
Strategy
Decision
Timing
Cache
Reason
```

con redacción.

---

# 185. Route Debug Toolbar

Podrá indicar:

```text
Route requires:
admin.access
invoice.update
```

---

# 186. Controller Debug

Mostrar metadata heredada y source.

---

# 187. Authorization Explain CLI

Comando futuro:

```text
volt authorization:explain
```

---

# 188. Structural Invocation

Ejemplo:

```text
volt authorization:explain \
--ability=invoice.approve \
--subject-type=Invoice
```

sin Principal concreto:

```text
show structural plan
```

---

# 189. Live Invocation

Con fixtures o development adapters:

```text
--principal=user:42
--subject=invoice:928
```

podrá realizar evaluación real.

---

# 190. Production Safety

No permitir arbitrariamente en producción:

```text
inspect any user's authorization
```

sin controles administrativos.

---

# 191. Explain Authorization

El propio comando/panel debe estar protegido por:

```text
authorization.debug.inspect
```

o equivalente.

---

# 192. No Security Backdoor

Debug tooling no debe crear bypass.

---

# 193. `authorization:plan`

Ya definido previamente.

---

# 194. `authorization:trace`

Podrá consultar un trace persistido:

```text
volt authorization:trace authz_123
```

---

# 195. `authorization:audit`

Podrá buscar audit records.

---

# 196. Filter Examples

```text
by ability
by decision
by reason code
by actor
by tenant
by date
```

con controles de acceso.

---

# 197. Audit Query Security

El acceso a auditoría puede ser altamente sensible.

---

# 198. AuditPolicy

Podrá existir:

```text
AuthorizationAuditRecordPolicy
```

para proteger quién puede ver audit records.

---

# 199. Recursive Concern

Consultar audit no necesita autorizarse mediante el mismo record.

Se protege como cualquier resource normal.

---

# 200. Decision Replay

Futuro:

```text
authorization:replay
```

podría reconstruir una evaluación usando snapshots.

---

# 201. Limitation

Sin snapshots de:

```text
Principal grants
Subject state
Tenant state
External evaluator state
```

no puede garantizarse replay exacto.

---

# 202. Explain Historical Decision

Audit podrá explicar:

```text
what was recorded
```

no necesariamente recalcular fielmente.

---

# 203. Policy Version

Audit crítico podrá registrar:

```text
authorization_registry_version
policy_version/fingerprint
```

---

# 204. Strategy Version

También:

```text
strategy_id
```

y version si las custom strategies evolucionan.

---

# 205. Configuration Fingerprint

Puede almacenarse:

```text
authorization_config_fingerprint
```

para forensics.

---

# 206. Deployment ID

Opcional:

```text
application_deployment_id
```

útil en rolling deployments.

---

# 207. Cached Decision Audit

Si la decisión vino de cache:

```text
decision_source=cross_request_cache
```

deberá poder auditarse.

---

# 208. Original Decision Timestamp

Opcionalmente:

```text
cached_decision_created_at
```

para saber freshness.

---

# 209. Request Memoization

Puede registrarse:

```text
source=request_memo
```

solo en profiler/debug, normalmente no en audit persistido.

---

# 210. Audit of Cache

Critical operations pueden querer saber si el check fue:

```text
fresh
```

o:

```text
cached
```

---

# 211. Freshness Metadata

Ejemplo:

```text
authorization_freshness=strong
```

---

# 212. Critical Ability

Para:

```text
system.deploy
```

audit puede exigir:

```text
decision_source=fresh
```

---

# 213. Explain Cached Decision

Debe poder mostrar:

```text
Decision reused from cache.

Created:
...

Valid under:
principal auth version 19
tenant security version 8
subject auth version 4
```

sin exponer detalles públicos.

---

# 214. Sampling and Cache

Cache hits pueden reducir los evaluator traces disponibles.

Por tanto:

```text
explain cached decision
```

depende de provenance almacenada.

---

# 215. Minimal Provenance

Cross-request cached decision debería conservar:

```text
decisive evaluator IDs
reason code
strategy
plan fingerprint
```

si explainability lo requiere.

---

# 216. Full Provenance

No necesario por defecto.

---

# 217. Observability Performance Budget

El subsistema deberá tener niveles configurables.

---

# 218. Production Minimal

```text
metrics
critical audit
errors
selected structured logs
```

---

# 219. Production Detailed

```text
sampled traces
evaluator timings
decision provenance
```

---

# 220. Development

```text
full traces
plan explanations
metadata source locations
full safe profiling
```

---

# 221. Zero/Low Allocation Path

Cuando:

```text
trace=off
audit=none
metrics=minimal
```

el Authorization Core deberá evitar construir:

```text
trace event arrays
full explanation trees
string messages
source location objects
```

---

# 222. Lazy Explanation

Solo construir explicación cuando:

```text
requested
audit requires it
debugging active
```

---

# 223. Reason Codes Are Cheap

Aunque tracing esté desactivado, `reasonCode` puede mantenerse porque es útil para final decision.

---

# 224. Human Messages Are Expensive

No traducir mensajes en hot path si no se necesitan.

---

# 225. String Interpolation

Evitar generar:

```text
User 42 cannot approve Invoice 928 because...
```

durante cada check.

---

# 226. Observability Configuration

Ejemplo conceptual:

```php
return [
    'authorization' => [
        'trace' => [
            'mode' => 'errors',
            'sample_rate' => 0.01,
        ],

        'audit' => [
            'default' => 'denials',
        ],

        'profiler' => [
            'enabled' => false,
        ],
    ],
];
```

---

# 227. Per-Ability Override

```text
system.deploy
audit=all
trace=full-on-deny
freshness=strict
```

---

# 228. Per-Evaluator Override

Un external evaluator podrá solicitar timing siempre que metrics estén activadas.

---

# 229. Authorization Observability Policy

Podrá existir un resolver que combine:

```text
global config
ability risk
channel
environment
decision
```

---

# 230. Environment Constraints

En producción:

```text
full request dumps
```

deberán estar deshabilitados aunque debug esté activado accidentalmente.

---

# 231. Safe Production Guard

Authorization debug output público deberá requerir configuración explícita muy restrictiva.

---

# 232. Exception Tracing

Cuando `PolicyExecutionException` ocurre:

```text
trace
```

deberá preservar:

```text
evaluator ID
phase
safe failure category
```

---

# 233. Root Cause

El stack trace real queda en error logging interno.

---

# 234. User Response

Solo:

```text
Forbidden
```

o mapping adecuado.

---

# 235. Failure Correlation

El error log deberá incluir:

```text
authorization_trace_id
```

para cruzarlo con trace/audit.

---

# 236. Audit Failure Correlation

Si audit sink falla:

```text
audit_error_id
```

puede vincularse.

---

# 237. Security Incident Correlation

Cross-tenant attempts repetidos pueden correlacionarse por:

```text
actor
IP
session
tenant targets
```

en Security subsystem.

Authorization no necesita hacer detección de amenazas avanzada por sí solo.

---

# 238. Abuse Detection Integration

Emitirá señales estructuradas.

---

# 239. Example Security Signal

```text
event:
authorization.cross_tenant_denied

actor:
user:42

target_tenant:
9

ability:
invoice.view
```

con campos sensibles filtrados.

---

# 240. No User-Controlled Log Injection

Todos los valores deben estructurarse/escaparse.

---

# 241. Reason Messages from Policies

Si una Policy permite:

```php
DecisionResult::deny(
    reason: $dynamicValue
)
```

ese reason no debe enviarse sin sanitización.

---

# 242. Preferred API

Usar:

```text
reasonCode
+
safe metadata
```

---

# 243. Dynamic Reason Metadata

Ejemplo:

```text
required_approval_level=2
```

puede ser internal-only.

---

# 244. PII Classification

Metadata podrá marcarse:

```text
sensitive=true
```

si se utiliza un objeto estructurado.

---

# 245. AuthorizationMetadataValue

Futuro:

```php
final readonly class AuthorizationMetadataValue
{
    public function __construct(
        public mixed $value,
        public DataClassification $classification,
    ) {}
}
```

---

# 246. V1 Recommendation

Mantener metadata pequeña y documentar que valores sensibles no deben agregarse a `DecisionResult`.

---

# 247. Audit Encryption

Si audit contiene datos sensibles:

```text
encryption at rest
```

pertenece al storage subsystem.

---

# 248. Audit Access Logging

Consultar registros de auditoría también puede auditarse.

---

# 249. Compliance Export

Audit export deberá tener su propia Policy.

---

# 250. Multi-Tenant Audit Isolation

Tenant administrators normalmente solo podrán ver audit de su Tenant.

---

# 251. Platform Audit

Global administrators podrán tener acceso cross-tenant limitado.

---

# 252. Tenant ID in Audit

Un record tenant-scoped deberá incluir:

```text
TenantReference
```

---

# 253. Global Operation

Puede tener:

```text
tenant=null
context_mode=global
```

---

# 254. System Operation

Debe distinguir:

```text
context_mode=system
```

---

# 255. Tenant Audit Leak

Queries de audit deben aplicar Tenant scope igual que otros datos.

---

# 256. Audit Storage Multi-Tenancy

Puede ser:

```text
shared with tenant_id
separate partition
separate store
```

según arquitectura.

---

# 257. Audit Immutable Models

Los records no deberían editarse como datos ordinarios.

---

# 258. Correction

Si un record necesita corrección, preferir:

```text
append correction record
```

sobre mutar el original.

---

# 259. Authorization Dashboard

Futuro panel podrá mostrar:

```text
Top denied abilities
Slowest Policies
Critical evaluator failures
Cross-tenant attempts
Permission miss rate
Decision cache hit rate
```

---

# 260. No Business Analytics Coupling

Este dashboard es operacional/security, no reemplaza analytics de producto.

---

# 261. Alerts

Ejemplos:

```text
Cross-tenant denies > threshold
Policy failures > threshold
Permission evaluator latency spike
Audit sink unavailable
```

---

# 262. Alert Evaluation

Debe ocurrir en observability/security infrastructure, no dentro de request authorization logic.

---

# 263. Testing Tracing

Tests deberán verificar:

```text
trace contains expected evaluator order
```

sin depender de timestamps exactos.

---

# 264. Testing Redaction

Debe probarse que:

```text
tokens
passwords
sensitive attributes
```

no aparecen.

---

# 265. Testing Audit

Verificar:

```text
critical GRANT produces audit
normal low-risk GRANT may not
DENY produces required audit
```

según policy.

---

# 266. Testing Required Audit Failure

Si:

```text
AuditDeliveryGuarantee::Required
```

y sink falla:

```text
operation does not proceed
```

---

# 267. Testing BestEffort Failure

Sink falla:

```text
authorization result remains unchanged
```

---

# 268. Testing Explainability

Misma ejecución deberá producir:

```text
same final Decision
```

con explainability on/off.

---

# 269. Testing Sampling

Mandatory audit nunca debe desaparecer por sampling.

---

# 270. Testing Nested Traces

Parent-child IDs deben conservarse.

---

# 271. Testing Cache Provenance

Cache hit deberá indicar:

```text
source=cache
```

en trace.

---

# 272. Testing FrankenPHP

Trace collectors y audit buffers no deberán cruzar requests.

---

# 273. Request A

```text
trace A
tenant 7
```

---

# 274. Request B

Nunca deberá contener:

```text
events from trace A
```

---

# 275. AuthorizationObserver Concurrency

Observers shared deben ser stateless o concurrency-safe.

---

# 276. Audit Buffer

Si existe buffer request-scoped, debe limpiarse en finally.

---

# 277. Queue Observability

Jobs deberán tener:

```text
job correlation ID
tenant
principal reference
authorization traces
```

cuando aplique.

---

# 278. CLI Observability

Commands podrán escribir explanations directamente a consola en modo verbose.

---

# 279. Example CLI Debug

```text
$ volt authorization:explain ...

Decision: DENY
Reason: rbac.permission_missing

Pipeline:
✓ tenant
✗ invoice.approve permission
- InvoicePolicy skipped
```

---

# 280. Production CLI

No exponer secretos en output de shell.

---

# 281. API Explain Endpoint

No se recomienda habilitar un endpoint genérico:

```text
/authz/explain
```

en producción.

---

# 282. Why

Podría convertirse en:

```text
permission enumeration oracle
resource existence oracle
security rule disclosure
```

---

# 283. Controlled Explain API

Si una aplicación lo necesita:

```text
strongly authenticated
privileged
rate-limited
audited
```

---

# 284. End-User Explanations

Algunas aplicaciones pueden querer razones seguras.

Ejemplo:

```text
You need MFA before exporting this report.
```

---

# 285. Safe Actionable Denials

Esto puede mejorar UX para:

```text
MFA
subscription upgrade
workflow prerequisite
```

---

# 286. Unsafe Denials

No revelar:

```text
resource belongs to Tenant B
user lacks hidden investigation role
account is flagged by fraud engine
```

---

# 287. Public Reason Mapping

Podrá existir:

```text
AuthorizationPublicReasonMapper
```

---

# 288. Example

```text
security.mfa_required
→ mfa_required
```

```text
tenant.mismatch
→ resource_not_found
```

---

# 289. Policy-Controlled Public Messages

No se recomienda que cada Policy escriba directamente mensajes públicos.

Central mapping es más consistente.

---

# 290. Reason Localization

```text
mfa_required
```

puede traducirse en frontend/backend.

---

# 291. Explainability Completeness

No toda Policy podrá explicar cada detalle.

El sistema deberá permitir:

```text
reason=authorization.denied
```

genérico.

---

# 292. Do Not Fabricate Explanation

Si no existe información suficiente:

```text
Decision denied by policy.
```

es mejor que inventar una causa.

---

# 293. Authorization Audit Schema

Ejemplo conceptual:

```text
authorization_audit

id
trace_id
correlation_id

actor_type
actor_id

principal_type
principal_id

tenant_type
tenant_id

ability

subject_type
subject_id

decision
reason_code
strategy

risk_level
decision_source

occurred_at
metadata
```

---

# 294. Indexes

Posibles índices:

```text
occurred_at
principal
actor
tenant
ability
decision
reason_code
```

según volumen.

---

# 295. Partitioning

Para alto volumen:

```text
time partitioning
tenant partitioning
```

puede utilizarse.

---

# 296. Audit Repository

Contrato:

```php
interface AuthorizationAuditRepositoryInterface
{
    public function append(
        AuthorizationAuditRecord $record
    ): void;
}
```

---

# 297. Append Semantics

No:

```text
save/update
```

sino:

```text
append
```

para enfatizar inmutabilidad.

---

# 298. Audit Outbox

Podrá integrarse con el Event/Database System.

---

# 299. Audit Serialization

Debe ser versionada.

---

# 300. Audit Schema Version

Cada record podrá incluir:

```text
schema_version
```

---

# 301. Backward Compatibility

Los readers deberán manejar versiones históricas según soporte.

---

# 302. Trace Serialization

Los traces completos podrán utilizar un formato separado.

No mezclar schema de audit estable con debug traces volátiles.

---

# 303. Observability Events API

Podrá haber DTOs específicos:

```text
AuthorizationStartedEvent
EvaluatorCompletedEvent
AuthorizationCompletedEvent
```

---

# 304. Avoid Generic Arrays Everywhere

Tipos estructurados mejoran:

```text
static analysis
redaction
versioning
testing
```

---

# 305. Hot Path Optimization

Internamente puede utilizarse un collector especializado para reducir allocations.

---

# 306. Null Collector

Cuando tracing está deshabilitado:

```text
NullAuthorizationTraceCollector
```

---

# 307. Compiler Optimization

El Container podrá enlazar directamente el NullCollector en producción mínima.

---

# 308. Metrics Collector

Igualmente:

```text
NullAuthorizationMetricsCollector
```

si no está habilitado.

---

# 309. Audit Decision at End

El sistema no debe crear records completos antes de saber:

```text
final decision
```

salvo eventos parciales críticos.

---

# 310. Failed Evaluation Audit

Aunque no exista Decision semántica inicial, final fail-closed puede auditarse como:

```text
DENY
reason=authorization.evaluator_failure
```

con metadata técnica internal-only.

---

# 311. Planning Failure Audit

Igualmente:

```text
authorization.plan_failure
```

en operaciones críticas.

---

# 312. Unknown Ability

En strict mode:

```text
configuration failure
```

puede generar observability event.

---

# 313. Suspicious Unknown Ability

Si proviene de input controlado por cliente, puede ser un security signal.

---

# 314. Policy Registry Drift

Cache stale o metadata inconsistency deberá ser visible.

---

# 315. Metrics

```text
authorization.metadata.stale
authorization.registry.mismatch
```

---

# 316. Audit Is Not Debug Log

Los records de audit deben ser:

```text
stable
minimal
intentional
```

---

# 317. Debug Logs Are Not Audit

Un logger rotado semanalmente no sustituye audit regulatorio.

---

# 318. SIEM Is Not Necessarily Audit Source of Truth

Puede recibir copia, pero la aplicación debe definir garantías.

---

# 319. Explainability Is Not Authorization

Una explicación nunca deberá ser usada como fuente de decisión.

---

# 320. Never Parse Human Reason

No hacer:

```php
if (str_contains($reason, 'permission')) {
    ...
}
```

---

# 321. Use Reason Codes

Siempre.

---

# 322. Authorization Evaluation Snapshot

Para debug puede capturarse un snapshot seguro:

```text
ability
subject reference
principal fingerprint
tenant fingerprint
security context fingerprint
```

---

# 323. No Full Principal Snapshot

Evitar roles/claims completos salvo que sean necesarios y redacted.

---

# 324. Explain RBAC Provenance

Se puede almacenar:

```text
permission source:
role finance-manager
```

---

# 325. Permission Source Types

```text
direct
role
external provider
delegated
token scope
```

---

# 326. Explain ReBAC Provenance

```text
direct relation
derived path
materialized edge
```

---

# 327. Explain Cache Provenance

```text
request memo
distributed grant cache
final decision cache
```

---

# 328. Explain Plan Provenance

```text
compiled route metadata
controller attribute
ability metadata
global evaluator
```

---

# 329. Full Explain Example

```text
Decision:
DENY

Ability:
invoice.approve

Principal:
user:42

Tenant:
tenant:7

Subject:
invoice:928

Strategy:
deny_overrides

Requirements:

1. Tenant membership
   Source:
   global security policy
   Result:
   GRANT

2. Permission invoice.approve
   Source:
   #[RequiresPermission]
   Grant Source:
   role finance-manager
   Result:
   GRANT

3. Organization relation
   Source:
   Invoice authorization plan
   Result:
   GRANT

4. InvoicePolicy::approve
   Result:
   GRANT

5. ApprovalLimitRule
   Result:
   DENY

Final Reason:
invoice.approval_limit_exceeded
```

---

# 330. Explain Performance Warning

```text
RelationshipEvaluator was the slowest evaluator: 8.4 ms
```

solo en developer/operator modes.

---

# 331. Tooling Permissions

Cada herramienta deberá tener ability propia:

```text
authorization.debug.view
authorization.audit.view
authorization.audit.export
authorization.trace.view
```

---

# 332. Separation of Duties

Quien puede:

```text
manage Roles
```

no necesariamente puede:

```text
view security audit
```

---

# 333. Audit Export

Debe ser altamente controlado.

---

# 334. Audit Redaction Per Viewer

Un Tenant admin y Platform security operator pueden recibir niveles distintos de metadata.

---

# 335. Explanation Determinism

Dado el mismo `AuthorizationExecution`, la explicación estructural debe ser determinista.

---

# 336. Formatter Differences

Mensajes humanos pueden variar por locale/audience.

---

# 337. Stable Audit IDs

No depender de mensajes traducidos.

---

# 338. Audit Failure Handling

Por defecto:

```text
BestEffort audit failure
```

debe emitir:

```text
authorization.audit.failed
```

a logging seguro.

---

# 339. Avoid Recursion

Ese fallo no debe volver a intentar auditarse infinitamente.

---

# 340. Circuit Breaker

Un sink externo de audit puede usar resilience infrastructure.

---

# 341. Local Fallback

Para `AtLeastOnce` podría usar:

```text
local durable outbox
```

si SIEM está caído.

---

# 342. Required Mode

Debe garantizar que no exista un "success" si la política exige audit duradero y no se pudo registrar.

---

# 343. Audit Record Timing

Para autorización pre-operación:

```text
record may indicate authorization granted
```

pero no business success.

---

# 344. Business Correlation

El business event puede guardar:

```text
authorization_trace_id
```

para enlazar ambos.

---

# 345. Example

```text
AuthorizationGranted trace=authz_55
```

Luego:

```text
InvoiceApproved authorization_trace_id=authz_55
```

---

# 346. Powerful Forensics

Esto permite saber:

```text
qué decisión autorizó qué operación
```

sin acoplar sistemas.

---

# 347. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Observability/
        ├── Trace/
        │   ├── AuthorizationTrace.php
        │   ├── AuthorizationTraceContext.php
        │   ├── AuthorizationTraceEvent.php
        │   ├── AuthorizationTraceStage.php
        │   ├── AuthorizationTraceMode.php
        │   ├── AuthorizationTraceCollectorInterface.php
        │   ├── AuthorizationTraceCollector.php
        │   └── NullAuthorizationTraceCollector.php
        │
        ├── Explain/
        │   ├── AuthorizationExplanation.php
        │   ├── AuthorizationExplanationTree.php
        │   ├── AuthorizationExplanationFactor.php
        │   ├── AuthorizationExplainService.php
        │   ├── AuthorizationExplainMode.php
        │   ├── AuthorizationExplanationAudience.php
        │   ├── AuthorizationExplanationFormatterInterface.php
        │   └── AuthorizationPublicReasonMapper.php
        │
        ├── Reasons/
        │   ├── AuthorizationReasonDescriptor.php
        │   ├── AuthorizationReasonRegistry.php
        │   ├── AuthorizationReasonCategory.php
        │   └── AuthorizationReasonVisibility.php
        │
        ├── Audit/
        │   ├── AuthorizationAuditRecord.php
        │   ├── AuthorizationAuditMode.php
        │   ├── AuthorizationAuditSeverity.php
        │   ├── AuditDeliveryGuarantee.php
        │   ├── AuthorizationAuditPolicyResolver.php
        │   ├── AuthorizationAuditSinkInterface.php
        │   ├── AuthorizationAuditRepositoryInterface.php
        │   └── CompositeAuthorizationAuditSink.php
        │
        ├── Metrics/
        │   ├── AuthorizationMetricsCollectorInterface.php
        │   ├── AuthorizationMetricsCollector.php
        │   └── NullAuthorizationMetricsCollector.php
        │
        ├── Profiling/
        │   ├── AuthorizationProfiler.php
        │   ├── AuthorizationProfile.php
        │   ├── EvaluatorProfile.php
        │   └── AuthorizationNPlusOneDetector.php
        │
        ├── Redaction/
        │   ├── AuthorizationRedactor.php
        │   ├── AuthorizationMetadataRedactor.php
        │   └── AuthorizationDataClassification.php
        │
        ├── Events/
        │   ├── AuthorizationStarted.php
        │   ├── AuthorizationGranted.php
        │   ├── AuthorizationDenied.php
        │   ├── AuthorizationEvaluatorFailed.php
        │   └── AuthorizationSecurityEvent.php
        │
        └── Exceptions/
            ├── AuthorizationObservabilityException.php
            ├── AuthorizationAuditException.php
            ├── AuthorizationAuditDeliveryException.php
            └── AuthorizationTraceException.php
```

---

# 348. Trace Invariants

### Invariante 1

Tracing no modifica decisiones.

### Invariante 2

Trace state es request/execution scoped.

### Invariante 3

Nested authorization conserva parent-child relation.

### Invariante 4

Trace truncation no modifica semántica.

### Invariante 5

Tracing deshabilitado no construye estructuras costosas innecesariamente.

---

# 349. Explainability Invariants

### Invariante 1

La explicación deriva de resultados estructurados.

### Invariante 2

No se inventan razones ausentes.

### Invariante 3

Reason codes son estables y machine-readable.

### Invariante 4

La audiencia determina cuánto puede mostrarse.

### Invariante 5

Internal explanation y public explanation están separadas.

---

# 350. Audit Invariants

### Invariante 1

Audit registra hechos de autorización, no éxito de negocio.

### Invariante 2

Los records son mínimos y estructurados.

### Invariante 3

Actor y Effective Principal se conservan cuando difieren.

### Invariante 4

Audit obligatorio no puede desaparecer por sampling.

### Invariante 5

Records críticos deben ser append-only conceptualmente.

---

# 351. Privacy Invariants

### Invariante 1

No se serializan entidades completas.

### Invariante 2

Tokens y secretos jamás se registran.

### Invariante 3

Metadata pública usa allowlist.

### Invariante 4

Los valores sensibles se redactan antes de salir del Core.

### Invariante 5

Audit y trace respetan Tenant isolation.

---

# 352. Observability Invariants

### Invariante 1

Metrics evitan cardinalidad no controlada.

### Invariante 2

Observers normales no cambian decisiones.

### Invariante 3

Profiler no ejecuta Policies nuevamente.

### Invariante 4

Cache provenance puede observarse sin afectar cache semantics.

---

# 353. Runtime Invariants

### Invariante 1

Trace collectors no sobreviven entre requests.

### Invariante 2

Audit buffers request-scoped se limpian siempre.

### Invariante 3

Shared observers son stateless o concurrency-safe.

### Invariante 4

FrankenPHP worker reuse no mezcla trazas de distintos tenants/principals.

---

# 354. Security Invariants

### Invariante 1

Explain tooling está autorizado.

### Invariante 2

Debug output no crea permission oracle público.

### Invariante 3

Cross-tenant denials no revelan ownership.

### Invariante 4

Failure details permanecen internos.

### Invariante 5

Audit failure nunca produce un GRANT adicional.

---

# 355. Arquitectura final

```text
                     AuthorizationRequest
                             │
                             ↓
                      Trace Context
                             │
                             ↓
                  AuthorizationPlanner
                             │
                    ┌────────┴────────┐
                    ↓                 ↓
               Plan Trace         Plan Metrics
                    │                 │
                    └────────┬────────┘
                             ↓
                    Evaluator Execution
                             │
                 ┌───────────┼───────────┐
                 ↓           ↓           ↓
              Trace       Profiler     Metrics
                 │           │           │
                 └───────────┼───────────┘
                             ↓
                      DecisionManager
                             │
                             ↓
                    Final DecisionResult
                             │
          ┌──────────────────┼──────────────────┐
          ↓                  ↓                  ↓
     Explanation          Audit Policy       Metrics
          │                  │                  │
          ↓                  ↓                  ↓
     Formatter           Audit Sink         Collector
```

---

# 356. Flujo completo — DENY crítico

Solicitud:

```text
Ability:
system.deploy

Principal:
user:42

Context:
MFA level 1
```

Plan:

```text
MfaRequirementEvaluator
DeploymentPermissionEvaluator
EnvironmentPolicy
```

---

# 357. Execution

```text
MfaRequirementEvaluator
→ DENY
reason=security.mfa_required
```

Con `DenyOverrides`:

```text
short-circuit
```

---

# 358. Trace

```text
trace_id:
authz_500

decision:
DENY

decisive evaluator:
mfa_requirement

reason:
security.mfa_required

duration:
0.17 ms

short_circuit:
true
```

---

# 359. Public Explain

```text
Additional authentication is required.
```

---

# 360. Audit

Como `system.deploy` es crítica:

```text
Actor
Principal
Ability
Decision
ReasonCode
Timestamp
Security Context fingerprint
```

se registra.

---

# 361. Security Event

Puede emitirse:

```text
authorization.critical_denial
```

sin enviar secretos.

---

# 362. Flujo completo — Cross-Tenant DENY

Context:

```text
Tenant#7
```

Subject:

```text
Invoice#928
Tenant#9
```

---

# 363. Internal Trace

```text
TenantIsolationPolicy
→ DENY

reason:
tenant.mismatch
```

---

# 364. End-User Output

```text
404 Not Found
```

si concealment está configurado.

---

# 365. Operator Explain

```text
Subject tenant did not match active TenantContext.
```

---

# 366. Audit

```text
decision=DENY
reason=tenant.mismatch
severity=critical
```

según política.

---

# 367. Flujo completo — GRANT con cache

Solicitud:

```text
invoice.view
```

Request memo:

```text
HIT
```

---

# 368. Trace

```text
Decision Source:
request_memo

Evaluator Execution:
skipped due to memoization

Decision:
GRANT
```

---

# 369. Audit

Si `invoice.view` no requiere audit:

```text
no persistent record
```

---

# 370. Metrics

Sí puede incrementarse:

```text
authorization.memo.hit
authorization.decisions.grant
```

---

# 371. Resultado esperado

El `Authorization Audit, Observability, Tracing and Explainability System` deberá permitir que VoltStack sea capaz de explicar y auditar desde una autorización simple:

```php
$user->can('update', $invoice);
```

hasta una decisión compleja:

```text
Tenant Membership
        +
Tenant Isolation
        +
Role
        +
Permission
        +
ReBAC Relationship
        +
ABAC Rule
        +
Resource Policy
        +
Compliance Policy
        ↓
Decision Strategy
        ↓
DENY
```

sin comprometer rendimiento ni privacidad.

El modelo final será:

```text
Authorization Execution
        ↓
Structured Trace
        ↓
Decision Provenance
        ↓
Reason Codes
        ↓
Explainability
        ↓
Audit / Metrics / Profiling
```

El principio definitivo será:

```text
Authorization must be enforceable.

Security decisions must be explainable.

Critical decisions must be auditable.

Operational behavior must be observable.

Sensitive information must remain protected.

And none of those capabilities may become
a hidden second authorization engine.
```

Con esta arquitectura, VoltStack podrá ofrecer autorización empresarial con trazabilidad completa, auditoría selectiva, diagnósticos profundos y soporte para herramientas de desarrollo sin convertir el sistema de observabilidad en una fuente de fugas de información o degradación de seguridad.