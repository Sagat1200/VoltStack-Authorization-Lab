# VoltStack Authorization System — Failure, Error, Denial and Exception Handling System

## 1. Propósito

Este documento define la arquitectura completa para el manejo de:

```text
DENIALS
FAILURES
ERRORS
EXCEPTIONS
CONFIGURATION FAULTS
SECURITY FAILURES
TRANSPORT MAPPING
RECOVERY
```

dentro del Authorization System de VoltStack.

El sistema deberá distinguir claramente entre:

```text
"El usuario no tiene permiso"
```

y:

```text
"No fue posible determinar de forma segura si tiene permiso"
```

porque ambos escenarios normalmente terminan bloqueando la operación, pero representan condiciones arquitectónicas completamente diferentes.

La regla fundamental será:

```text
DENY is an authorization decision.

FAILURE is an inability to safely obtain a decision.

Both may block execution.

They must never be treated as the same event.
```

---

# 2. Objetivos

El sistema deberá garantizar:

1. separación estricta entre decisiones y errores;
2. comportamiento fail-closed;
3. excepciones tipadas;
4. respuestas HTTP consistentes;
5. soporte para CLI, Queue y otros transports;
6. protección contra information leakage;
7. integración con tracing y audit;
8. manejo seguro de errores de Policies;
9. manejo de fallos de evaluadores externos;
10. detección de configuración inválida;
11. comportamiento determinista;
12. compatibilidad con FrankenPHP;
13. extensibilidad para adapters;
14. testing preciso de failure paths.

---

# 3. Modelo conceptual

VoltStack distinguirá cuatro resultados fundamentales:

```text
GRANT
DENY
ABSTAIN
FAILURE
```

Pero existe una diferencia importante.

Los primeros tres pertenecen al:

```text
Decision Domain
```

Mientras:

```text
FAILURE
```

pertenece al:

```text
Execution Domain
```

---

# 4. Decision Domain

Los evaluadores pueden producir:

```text
GRANT
DENY
ABSTAIN
```

Ejemplo:

```text
InvoicePolicy::update()

GRANT
```

o:

```text
DENY
reason=invoice.locked
```

---

# 5. Execution Failure

Ejemplo:

```text
InvoicePolicy::update()

throws DatabaseConnectionException
```

Esto no significa:

```text
DENY
```

semánticamente.

Significa:

```text
Policy execution failed.
```

---

# 6. Fail-Closed

Sin embargo, desde el punto de vista de enforcement:

```text
cannot safely prove GRANT
        ↓
operation must not execute
```

Por tanto:

```text
FAILURE
→ block operation
```

pero sin convertir silenciosamente el failure en un DENY normal.

---

# 7. Regla central

```text
GRANT
→ operation may continue

DENY
→ operation must stop

ABSTAIN
→ DecisionManager continues

FAILURE
→ FailurePolicy decides how execution terminates
```

---

# 8. FailurePolicy

Por defecto:

```text
FAILURE
→ fail closed
```

---

# 9. Authorization Outcome

Internamente podrá existir un concepto superior:

```php
enum AuthorizationOutcomeType: string
{
    case Granted = 'granted';
    case Denied = 'denied';
    case Failed = 'failed';
}
```

`ABSTAIN` no aparece porque es un estado intermedio de evaluación.

---

# 10. DecisionResult

Permanece dedicado a:

```text
GRANT
DENY
ABSTAIN
```

---

# 11. AuthorizationOutcome

Representará el resultado global de ejecución.

Conceptualmente:

```php
final readonly class AuthorizationOutcome
{
    public function __construct(
        public AuthorizationOutcomeType $type,
        public ?DecisionResult $decision = null,
        public ?AuthorizationFailure $failure = null,
    ) {}
}
```

---

# 12. Beneficio

Evita modelar incorrectamente:

```text
Database timeout
```

como:

```text
DENY database_timeout
```

---

# 13. AuthorizationFailure

Representará un fallo estructurado.

```php
final readonly class AuthorizationFailure
{
    public function __construct(
        public AuthorizationFailureCode $code,
        public AuthorizationFailureCategory $category,
        public ?Throwable $cause = null,
        public array $metadata = [],
    ) {}
}
```

---

# 14. Failure Categories

Propuesta:

```php
enum AuthorizationFailureCategory: string
{
    case Configuration = 'configuration';
    case Resolution = 'resolution';
    case Execution = 'execution';
    case Infrastructure = 'infrastructure';
    case External = 'external';
    case Security = 'security';
    case Context = 'context';
    case Cache = 'cache';
    case Audit = 'audit';
    case Internal = 'internal';
}
```

---

# 15. Failure Codes

Ejemplos:

```text
authorization.configuration.invalid

authorization.policy.not_found
authorization.policy.method_not_found

authorization.evaluator.failed
authorization.evaluator.timeout

authorization.context.missing

authorization.external.unavailable

authorization.cache.corrupt

authorization.audit.required_failed

authorization.internal.error
```

---

# 16. Failure Code vs Exception Class

El:

```text
Failure Code
```

es estable y machine-readable.

La:

```text
Exception Class
```

representa el mecanismo PHP mediante el cual puede propagarse el fallo.

---

# 17. Exception Hierarchy

VoltStack deberá proporcionar una jerarquía clara.

```text
Throwable
   ↓
AuthorizationException
```

---

# 18. Base Exception

```php
class AuthorizationException extends RuntimeException
{
}
```

---

# 19. Principales familias

```text
AuthorizationException

├── AuthorizationDeniedException
│
├── AuthorizationConfigurationException
│
├── AuthorizationResolutionException
│
├── AuthorizationExecutionException
│
├── AuthorizationContextException
│
├── AuthorizationInfrastructureException
│
├── AuthorizationExternalEvaluatorException
│
├── AuthorizationCacheException
│
├── AuthorizationAuditException
│
└── AuthorizationInternalException
```

---

# 20. AuthorizationDeniedException

Esta excepción es especial.

Representa:

```text
valid authorization decision
=
DENY
```

No representa un fallo del Authorization System.

---

# 21. Example

```php
Authorization::authorize(
    'update',
    $invoice
);
```

Si la decisión es DENY:

```php
throw AuthorizationDeniedException::fromDecision(
    $decision
);
```

---

# 22. `can()` vs `authorize()`

VoltStack deberá mantener una diferencia clara.

```php
Authorization::can(
    'update',
    $invoice
);
```

retorna:

```text
true / false
```

---

# 23. `authorize()`

```php
Authorization::authorize(
    'update',
    $invoice
);
```

retorna normalmente:

```text
void / successful result
```

y lanza excepción ante:

```text
DENY
FAILURE
```

aunque con clases diferentes.

---

# 24. `inspect()`

Podrá existir:

```php
Authorization::inspect(
    'update',
    $invoice
);
```

para obtener:

```text
AuthorizationOutcome
```

sin convertir inmediatamente el resultado en transport exception.

---

# 25. Recommended API

```text
can()
→ boolean convenience

inspect()
→ structured outcome

authorize()
→ enforcement API
```

---

# 26. Failure Semantics of `can()`

Existe un problema importante.

Si:

```text
Policy database lookup fails
```

¿`can()` debe devolver false?

---

# 27. Recommended Behavior

Desde seguridad:

```text
false
```

es el resultado seguro.

Pero el failure no debe desaparecer.

Por tanto:

```text
can()
→ false
```

y simultáneamente:

```text
failure reported to observability
```

según configuración.

---

# 28. Strict `can()`

Podrá existir configuración:

```text
authorization.can.throw_on_failure=true
```

para entornos donde un fallo técnico debe propagarse.

---

# 29. Default

Recomendación:

```text
can()
DENY → false
FAILURE → false + report failure
```

Mientras:

```text
authorize()
DENY → AuthorizationDeniedException
FAILURE → typed failure exception
```

---

# 30. Why

Esto mantiene ergonomía:

```php
if ($user->can('edit', $invoice)) {
}
```

sin permitir acceso durante fallos.

---

# 31. No Silent Failure

Aunque `can()` devuelva false:

```text
FAILURE
```

debe permanecer visible para:

```text
logs
metrics
trace
error reporting
```

---

# 32. `inspect()` Difference

```php
$outcome = Authorization::inspect(
    'edit',
    $invoice
);
```

permite distinguir:

```php
$outcome->isGranted();
$outcome->isDenied();
$outcome->isFailed();
```

---

# 33. Denial Model

Un DENY debe contener información estructurada.

```php
final readonly class AuthorizationDenial
{
    public function __construct(
        public string $reasonCode,
        public ?string $decisiveEvaluator = null,
        public array $metadata = [],
    ) {}
}
```

---

# 34. Denial Is Expected

Un DENY puede ser una condición normal.

Ejemplo:

```text
user cannot edit invoice
```

No debe registrarse automáticamente como:

```text
ERROR
```

---

# 35. Log Level

Normal DENY:

```text
DEBUG / INFO
```

o ningún log persistente.

---

# 36. Security DENY

Ejemplo:

```text
cross-tenant access
```

puede generar:

```text
WARNING / SECURITY EVENT
```

---

# 37. Failure Is Unexpected

Ejemplo:

```text
Policy threw TypeError
```

debe ser:

```text
ERROR
```

---

# 38. Critical Failure

Ejemplo:

```text
Required audit storage unavailable
```

podría ser:

```text
CRITICAL
```

---

# 39. Denial Reasons

Ejemplos:

```text
authorization.denied
rbac.permission_missing
tenant.mismatch
security.mfa_required
resource.locked
relationship.missing
```

---

# 40. Failure Reasons

Deben usar namespace separado.

Ejemplo:

```text
authorization.failure.policy_execution
authorization.failure.external_timeout
authorization.failure.configuration
```

---

# 41. Never Mix

No:

```text
reason=database_timeout
decision=DENY
```

Preferir:

```text
outcome=FAILED
failure=authorization.failure.database
```

---

# 42. Configuration Failures

Representan errores que idealmente deberían detectarse durante:

```text
compile
boot
cache warmup
lint
```

---

# 43. Examples

```text
Unknown Ability
Policy class does not exist
Policy method missing
Invalid Policy signature
Unknown DecisionStrategy
Invalid metadata
Duplicate Gate
Conflicting Policy registration
Invalid evaluator priority
```

---

# 44. AuthorizationConfigurationException

Base:

```php
class AuthorizationConfigurationException
    extends AuthorizationException
{
}
```

---

# 45. Specialized Configuration Exceptions

```text
UnknownAbilityException
DuplicateAbilityException
PolicyRegistrationException
InvalidPolicySignatureException
UnknownDecisionStrategyException
InvalidAuthorizationMetadataException
```

---

# 46. Production Behavior

Muchos de estos errores deberían impedir:

```text
application boot
```

cuando pueden detectarse estáticamente.

---

# 47. Fail Early

Principio:

```text
configuration errors
should fail during compilation/boot,
not during customer requests.
```

---

# 48. Runtime Configuration Failure

Si aun así ocurre:

```text
FAIL CLOSED
```

y registrar error.

---

# 49. Unknown Ability

Debe distinguirse de:

```text
known ability but denied
```

---

# 50. Strict Mode

En:

```text
authorization.strict=true
```

una Ability desconocida:

```text
invoice.aprove
```

con typo deberá generar:

```text
UnknownAbilityException
```

---

# 51. Lenient Mode

Podría producir:

```text
DENY
reason=authorization.unknown_ability
```

pero no es la opción recomendada para desarrollo.

---

# 52. Recommended

```text
Development:
strict

Production compiled metadata:
strict
```

El sistema debería conocer sus Abilities.

---

# 53. Dynamic Abilities

Sistemas que permitan abilities dinámicas deberán registrar un resolver explícito.

No desactivar strict mode globalmente sin necesidad.

---

# 54. Policy Resolution Failure

Ejemplo:

```text
Invoice → InvoicePolicy
```

pero:

```text
InvoicePolicy not found
```

---

# 55. Is Missing Policy a DENY?

Depende de la arquitectura.

Si existe:

```text
explicit policy required
```

entonces:

```text
configuration failure
```

---

# 56. If Policy Is Optional

El resolver puede producir:

```text
no policy evaluator
```

y continuar con:

```text
Gates
RBAC
other evaluators
```

---

# 57. Required Evaluator

Cada plan entry podrá declarar:

```text
required=true
```

---

# 58. Missing Required Evaluator

Resultado:

```text
FAILURE
```

---

# 59. Missing Optional Evaluator

Resultado:

```text
skip
```

o:

```text
ABSTAIN
```

según semántica.

---

# 60. Policy Method Missing

Ejemplo:

```text
Policy:
InvoicePolicy

Ability:
archive

Expected:
archive()
```

pero no existe.

---

# 61. Explicit Mapping

Si Ability tenía mapping explícito:

```text
configuration failure
```

---

# 62. Convention Resolution

Si la Policy simplemente no implementa esa Ability:

```text
ABSTAIN
```

podría ser válido si el contrato así lo define.

---

# 63. Compile-Time Validation

Preferir detectar mappings inválidos antes de runtime.

---

# 64. Invalid Policy Signature

Ejemplo:

```php
public function update(
    string $foo,
    int $bar,
): bool
```

cuando metadata espera:

```text
Principal
Subject
AuthorizationContext
```

Debe generar:

```text
InvalidPolicySignatureException
```

---

# 65. Reflection Runtime

No intentar adivinar silenciosamente argumentos.

---

# 66. Dispatcher Failure

Si el dispatcher no puede construir parámetros:

```text
FAILURE
```

---

# 67. Policy Execution Exception

Ejemplo:

```php
public function approve(
    User $user,
    Invoice $invoice
): bool {
    throw new RuntimeException('...');
}
```

---

# 68. Wrapper Exception

El dispatcher deberá envolverlo:

```php
PolicyExecutionException
```

con:

```text
policy ID
method
ability
trace ID
previous exception
```

---

# 69. Do Not Expose Previous Message

El mensaje interno podría contener:

```text
SQL
credentials
customer data
```

---

# 70. Safe Public Message

```text
Authorization could not be completed.
```

---

# 71. Preserve Cause Internally

```php
throw new PolicyExecutionException(
    policy: InvoicePolicy::class,
    ability: 'approve',
    previous: $e,
);
```

---

# 72. Evaluator Failure

No solo Policies pueden fallar.

También:

```text
Gate evaluator
RBAC provider
ABAC attribute provider
ReBAC graph
Tenant evaluator
External PDP
```

---

# 73. Generic EvaluatorExecutionException

```php
class EvaluatorExecutionException
    extends AuthorizationExecutionException
{
}
```

---

# 74. Metadata

Puede contener:

```text
evaluator ID
evaluator type
phase
failure code
```

---

# 75. FailurePolicyResolver

No todos los evaluadores necesariamente tienen el mismo comportamiento ante fallo.

---

# 76. Default

```text
fail closed
```

---

# 77. Optional Evaluator Failure

Algunos evaluadores puramente:

```text
advisory
observational
non-security
```

podrían configurarse como:

```text
skip on failure
```

pero no deberían formar parte de la decisión de seguridad.

---

# 78. Security Evaluator

Todo evaluator capaz de producir un GRANT/DENY que afecta enforcement deberá ser:

```text
fail_closed
```

por defecto.

---

# 79. Failure Modes

Propuesta:

```php
enum AuthorizationFailureMode: string
{
    case FailClosed = 'fail_closed';
    case Propagate = 'propagate';
    case Abstain = 'abstain';
}
```

---

# 80. Restriction

```text
Abstain
```

solo debe permitirse para evaluadores explícitamente declarados:

```text
optional=true
```

---

# 81. Never Fail Open

No debe existir default:

```text
failure → GRANT
```

---

# 82. No `FailOpen`

Incluso si una aplicación quisiera comportamiento especial, no debería exponerse como configuración global simple.

---

# 83. Why

Una línea:

```text
authorization.fail_open=true
```

sería demasiado peligrosa.

---

# 84. External Evaluators

Ejemplos:

```text
OPA
remote IAM
relationship service
fraud service
license service
```

---

# 85. External Failure Types

```text
timeout
connection refused
5xx
invalid response
authentication failure
rate limit
schema mismatch
```

---

# 86. ExternalEvaluatorException

Base:

```php
class AuthorizationExternalEvaluatorException
    extends AuthorizationException
{
}
```

---

# 87. Specialized

```text
ExternalEvaluatorTimeoutException
ExternalEvaluatorUnavailableException
ExternalEvaluatorProtocolException
ExternalEvaluatorAuthenticationException
```

---

# 88. Timeout

Debe existir timeout explícito.

Nunca permitir:

```text
authorization request hangs indefinitely
```

---

# 89. Timeout Budget

El Authorization Planner podrá derivar:

```text
remaining execution budget
```

para evaluadores externos.

---

# 90. Example

Authorization budget:

```text
100 ms
```

Evaluators:

```text
Tenant        1 ms
Permission    3 ms
Remote PDP   max 50 ms
```

---

# 91. Timeout Result

```text
FAILED
authorization.failure.external_timeout
```

---

# 92. Retry

No reintentar indiscriminadamente una autorización dentro de request.

---

# 93. Why

Retries pueden:

```text
increase latency
amplify outage
duplicate external load
```

---

# 94. Resilience Layer

Retries, circuit breakers y backoff pertenecen al:

```text
resilience/infrastructure layer
```

---

# 95. Authorization Behavior

Después de que ese layer reporta failure:

```text
fail closed
```

---

# 96. Circuit Breaker Open

Resultado:

```text
ExternalEvaluatorUnavailableException
```

no:

```text
DENY permission_missing
```

---

# 97. Cache Fallback

Como se definió anteriormente:

```text
stale GRANT
```

no debe usarse automáticamente ante external failure.

---

# 98. Fresh Cache Hit

Si existe una decisión válida dentro de su policy de consistencia:

```text
cache hit
```

puede utilizarse normalmente.

Eso no es un fallback stale.

---

# 99. Stale Decision

Si expiró:

```text
do not resurrect automatically
```

---

# 100. Context Failures

Ejemplos:

```text
Missing Principal
Missing TenantContext
Missing SecurityContext
Invalid Subject
Invalid Scope
```

---

# 101. Missing Principal

Debe distinguirse:

```text
anonymous principal intentionally allowed
```

de:

```text
authentication context unexpectedly missing
```

---

# 102. AnonymousPrincipal

Preferir modelar explícitamente:

```text
AnonymousPrincipal
```

---

# 103. Benefit

Así:

```text
no principal
```

es un error de contexto.

Mientras:

```text
AnonymousPrincipal
```

es un estado válido.

---

# 104. Missing TenantContext

Para una Ability tenant-required:

```text
FAILURE
```

o denial de security context según contrato.

---

# 105. Recommended

Si la aplicación debería haber establecido TenantContext y no lo hizo:

```text
AuthorizationContextException
```

---

# 106. Tenant Mismatch

En cambio:

```text
context tenant=7
subject tenant=9
```

es una decisión:

```text
DENY
```

---

# 107. Important Difference

```text
Missing required TenantContext
→ system/context failure
```

```text
TenantContext present but incompatible
→ authorization denial
```

---

# 108. Invalid Subject

Ejemplo:

```text
invoice.update
```

recibe:

```text
Customer
```

---

# 109. Subject Type Validation

Si Ability define:

```text
subjectType=Invoice
```

debe producir:

```text
InvalidAuthorizationSubjectException
```

---

# 110. Do Not DENY Silently

Un tipo incorrecto normalmente indica bug de programación.

---

# 111. Null Subject

Para class-level ability:

```text
invoice.create
```

puede ser válido.

---

# 112. Resource Ability

Para:

```text
invoice.update
```

null puede ser inválido.

---

# 113. Ability Descriptor

Deberá declarar:

```text
subjectRequirement
```

---

# 114. Subject Requirement

```php
enum SubjectRequirement: string
{
    case None = 'none';
    case Optional = 'optional';
    case Required = 'required';
}
```

---

# 115. Invalid Principal Type

Igualmente:

```text
InvalidPrincipalException
```

si un evaluator exige tipo incompatible.

---

# 116. Security Context Failure

Ejemplo:

```text
MFA state provider failed
```

No equivale a:

```text
MFA not completed
```

---

# 117. Difference

```text
MFA level = 1
required = 2
→ DENY
```

```text
MFA state unavailable
→ FAILURE
```

---

# 118. Fail Closed

Ambos bloquean.

Pero audit y response handling pueden diferenciarlos.

---

# 119. Cache Failures

Authorization usa distintos caches.

---

# 120. Cache Miss

No es error.

```text
MISS
→ evaluate normally
```

---

# 121. Cache Backend Unavailable

Normalmente:

```text
bypass cache
→ evaluate source
```

---

# 122. Cache Is Optimization

Si:

```text
Plan Cache
Decision Cache
Permission Cache
```

falla, la autorización debería poder continuar si la fuente autoritativa está disponible.

---

# 123. Exception

Si una arquitectura configuró un cache como:

```text
authoritative grant store
```

entonces ya no es realmente un cache y su failure es infraestructura crítica.

---

# 124. Corrupt Cache Entry

```text
discard
report
evaluate fresh
```

---

# 125. Never

```text
corrupt cached GRANT
→ trust it
```

---

# 126. Cache Deserialization Failure

Debe tratarse como:

```text
cache miss + observability warning
```

cuando sea seguro reconstruir.

---

# 127. Cache Key Failure

Si no puede generarse una key segura:

```text
skip cross-request cache
```

y evaluar fresh.

---

# 128. Plan Cache Corruption

Si existe metadata compilada válida:

```text
rebuild plan
```

---

# 129. Compiled Metadata Corruption

Si no puede reconstruirse de manera segura:

```text
FAILURE
```

---

# 130. Audit Failures

Definido previamente:

```text
BestEffort
AtLeastOnce
Required
```

---

# 131. BestEffort Audit Failure

```text
report failure
authorization decision unchanged
```

---

# 132. AtLeastOnce

Si puede persistirse en durable outbox:

```text
continue
```

---

# 133. Required

Si no puede garantizarse:

```text
FAILURE
```

aunque la Policy haya producido GRANT.

---

# 134. Example

```text
Policy:
GRANT

Required Audit:
FAILED
```

Final enforcement outcome:

```text
FAILED
operation blocked
```

---

# 135. Do Not Rewrite Policy Decision

La Policy sigue habiendo producido:

```text
GRANT
```

pero el operational authorization outcome es:

```text
FAILED
```

---

# 136. Trace

Debe mostrar ambas cosas.

```text
policy_decision=GRANT
audit_delivery=FAILED
final_outcome=FAILED
```

---

# 137. Authorization Pipeline Failure Phases

Failures podrán ocurrir en:

```text
REQUEST
NORMALIZATION
CONTEXT
PLANNING
RESOLUTION
INVOCATION
EVALUATION
DECISION
FINALIZATION
CACHE
AUDIT
TRANSPORT
```

---

# 138. Failure Phase

Propuesta:

```php
enum AuthorizationFailurePhase: string
{
    case Request = 'request';
    case Context = 'context';
    case Planning = 'planning';
    case Resolution = 'resolution';
    case Invocation = 'invocation';
    case Evaluation = 'evaluation';
    case Decision = 'decision';
    case Finalization = 'finalization';
    case Cache = 'cache';
    case Audit = 'audit';
    case Transport = 'transport';
}
```

---

# 139. Failure Metadata

Cada failure podrá incluir:

```text
phase
trace ID
ability
evaluator ID
safe reason
```

---

# 140. No Full Request

No adjuntar automáticamente:

```text
headers
body
cookies
tokens
```

---

# 141. Exception Context

Las excepciones podrán exponer:

```php
$exception->traceId();
$exception->ability();
$exception->failureCode();
```

sin exponer datos sensibles.

---

# 142. Error Handler Integration

El Authorization System se integrará con el Error Handling global de VoltStack.

---

# 143. Core Should Not Render HTTP

`Quantum\Authorization` no debe retornar:

```text
Response
JsonResponse
RedirectResponse
```

---

# 144. Instead

Core produce:

```text
Decision
Outcome
Exception
```

El transport layer realiza mapping.

---

# 145. HTTP Mapping

El HTTP adapter podrá mapear:

```text
AuthorizationDeniedException
→ 403
```

---

# 146. Authentication vs Authorization

Un Principal no autenticado puede producir:

```text
401
```

pero esto debe manejarse cuidadosamente.

---

# 147. 401 Meaning

```text
authentication required / invalid
```

---

# 148. 403 Meaning

```text
identity understood
but operation forbidden
```

---

# 149. Anonymous Authorization

Si una Policy evalúa `AnonymousPrincipal` y devuelve DENY:

el HTTP adapter podrá decidir:

```text
401
```

si la Ability requiere autenticación.

---

# 150. Ability Metadata

Podrá declarar:

```text
authenticationRequired=true
```

---

# 151. Mapping

```text
AnonymousPrincipal
+
authenticationRequired
+
DENY
→ 401
```

---

# 152. Authenticated Principal

```text
DENY
→ 403
```

normalmente.

---

# 153. 404 Concealment

Algunos DENY deben mapearse a:

```text
404
```

para evitar revelar existencia.

---

# 154. Example

Cross-tenant resource:

```text
tenant.mismatch
→ 404
```

---

# 155. Resource Concealment

Ability metadata o Reason mapping podrá declarar:

```text
conceal=true
```

---

# 156. Do Not Let Policy Return HTTP Status

No:

```php
return Decision::deny(status: 404);
```

Preferir:

```text
reason=tenant.mismatch
```

y transport policy decide.

---

# 157. Why

La misma autorización puede ejecutarse en:

```text
HTTP
CLI
Queue
WebSocket
RPC
```

---

# 158. AuthorizationHttpExceptionMapper

Contrato conceptual:

```php
interface AuthorizationHttpExceptionMapperInterface
{
    public function map(
        AuthorizationException $exception,
        HttpContext $context,
    ): HttpAuthorizationResponse;
}
```

---

# 159. HTTP Failure Mapping

Un fallo interno de autorización normalmente no debería responder:

```text
403
```

porque eso implica un DENY semántico.

---

# 160. Recommended

Dependiendo del failure:

```text
500
503
```

mientras la operación sigue bloqueada.

---

# 161. Example

External authorization provider unavailable:

```text
503 Service Unavailable
```

si es apropiado.

---

# 162. Policy Bug

```text
500 Internal Server Error
```

---

# 163. Missing Context Due to App Bug

```text
500
```

---

# 164. Required Audit Backend Unavailable

Puede ser:

```text
503
```

---

# 165. Production Information Leakage

Response:

```json
{
    "error": "authorization_unavailable"
}
```

No:

```json
{
    "error": "Redis authz cluster 10.0.0.18 unavailable"
}
```

---

# 166. API Error Codes

VoltStack podrá mapear a códigos públicos:

```text
authentication_required
forbidden
resource_not_found
authorization_unavailable
additional_authentication_required
```

---

# 167. Internal Codes

Permanecen:

```text
tenant.mismatch
authorization.failure.external_timeout
```

---

# 168. Public Error Mapper

Debe existir una capa:

```text
internal authorization reason
        ↓
safe public error
```

---

# 169. Browser Response

Puede ser:

```text
403 page
redirect to login
redirect to MFA
404 page
```

según contexto.

---

# 170. Redirect Decisions

El Authorization Core no decide redirects.

---

# 171. SPA Response

El SPA transport podrá retornar:

```json
{
    "type": "authorization_error",
    "code": "forbidden"
}
```

o protocolo VoltStack equivalente.

---

# 172. SPA MFA

Puede devolver una instrucción segura:

```text
additional_authentication_required
```

---

# 173. Frontend Must Not Decide Security

Aunque el frontend reciba reason:

```text
backend remains authoritative
```

---

# 174. CLI Mapping

En CLI:

```text
DENY
→ non-zero exit code
```

---

# 175. Example

```text
Permission denied: system.deploy
```

si el operador puede ver esa información.

---

# 176. CLI Failure

```text
Authorization service unavailable.
```

con exit code diferente.

---

# 177. Exit Codes

Podrá existir:

```text
10 authorization denied
11 authentication required
12 authorization unavailable
13 authorization configuration error
```

sin depender de esos números exactos en el Core.

---

# 178. Queue Mapping

Un Queue Job autorizado puede fallar por:

```text
DENY
```

o:

```text
FAILURE
```

---

# 179. Queue DENY

Normalmente:

```text
do not retry
```

porque repetir probablemente no cambiará la decisión inmediatamente.

---

# 180. Queue Infrastructure Failure

Puede ser:

```text
retryable
```

según tipo.

---

# 181. Queue Failure Classification

Podrá existir:

```php
enum AuthorizationFailureRetryability: string
{
    case Never = 'never';
    case Retryable = 'retryable';
    case Unknown = 'unknown';
}
```

---

# 182. Example

```text
permission_missing
→ Never
```

```text
external PDP timeout
→ Retryable
```

---

# 183. But Reauthorization on Retry

Cada retry deberá:

```text
authorize again
```

No reutilizar un resultado previo del Job automáticamente.

---

# 184. Why

Entre retries pueden cambiar:

```text
Roles
Permissions
Tenant
Resource
```

---

# 185. Scheduled Jobs

Igualmente deben usar Principal/Context actual válido.

---

# 186. WebSocket Mapping

Una operación prohibida podrá retornar:

```text
protocol authorization error
```

sin cerrar necesariamente toda la conexión.

---

# 187. Connection-Level Failure

Si la conexión pierde autenticación:

```text
close/re-authenticate
```

según transport.

---

# 188. RPC Mapping

Podrá mapear a:

```text
PERMISSION_DENIED
UNAUTHENTICATED
UNAVAILABLE
INTERNAL
```

según protocolo.

---

# 189. Transport Neutrality

Core no debe conocer esos códigos.

---

# 190. Denial Exception Contents

`AuthorizationDeniedException` podrá contener:

```text
Ability
Decision
ReasonCode
Trace ID
Subject reference
```

con getters seguros.

---

# 191. Exception Message

No debe ser fuente de lógica.

---

# 192. Example

```php
catch (AuthorizationDeniedException $e) {
    $e->reasonCode();
}
```

No:

```php
if ($e->getMessage() === 'Permission denied') {
}
```

---

# 193. Exception Serialization

No serializar excepciones completas en Queue payloads.

---

# 194. Why

Pueden contener:

```text
stack traces
object references
sensitive data
```

---

# 195. Failure DTO

Si un failure debe cruzar procesos:

```text
AuthorizationFailureEnvelope
```

con campos seguros.

---

# 196. Error Boundary

Cada `AuthorizationManager` invocation deberá actuar como boundary.

---

# 197. Conceptual Flow

```text
try
    normalize
    plan
    execute
    decide
    finalize
catch known authorization failure
    normalize to AuthorizationFailure
catch unexpected Throwable
    wrap as AuthorizationInternalException
```

---

# 198. Fatal Errors

No todos los errores PHP pueden recuperarse.

Pero los `Throwable` capturables deberán envolverse cuando sea apropiado.

---

# 199. Do Not Catch Too Broad Too Early

Cada layer debe capturar solo cuando puede añadir contexto o normalizar.

---

# 200. Example

PolicyDispatcher:

```text
catch Throwable
→ PolicyExecutionException
```

AuthorizationManager:

```text
catch AuthorizationException
→ preserve
```

---

# 201. Previous Exception

Siempre conservar:

```text
previous
```

internamente para debugging.

---

# 202. No Double Wrapping

Si ya es:

```text
AuthorizationException
```

no envolver innecesariamente.

---

# 203. Exception Taxonomy

La jerarquía deberá permitir:

```php
catch (AuthorizationDeniedException $e)
```

y separadamente:

```php
catch (AuthorizationExecutionException $e)
```

---

# 204. Recommended Hierarchy

```text
AuthorizationException
│
├── AuthorizationDeniedException
│
├── AuthorizationFailureException
│   │
│   ├── AuthorizationConfigurationException
│   ├── AuthorizationContextException
│   ├── AuthorizationResolutionException
│   ├── AuthorizationExecutionException
│   │   ├── PolicyExecutionException
│   │   ├── GateExecutionException
│   │   └── EvaluatorExecutionException
│   │
│   ├── AuthorizationInfrastructureException
│   ├── AuthorizationExternalEvaluatorException
│   ├── AuthorizationCacheException
│   ├── AuthorizationAuditException
│   └── AuthorizationInternalException
```

---

# 205. Why Failure Base

Permite:

```php
catch (AuthorizationFailureException $e)
```

sin capturar un DENY esperado.

---

# 206. Denial Should Not Be Error Reporter Exception

El global error reporter podrá ignorar:

```text
AuthorizationDeniedException
```

por defecto.

---

# 207. Failure Should Be Reported

```text
AuthorizationFailureException
```

sí deberá reportarse normalmente.

---

# 208. Reporting Severity

El exception podrá exponer:

```text
reportSeverity
```

o resolverlo externamente por failure code.

---

# 209. Exception Handler Registration

VoltStack podrá registrar handlers para:

```text
render
report
```

separadamente.

---

# 210. Report

```text
AuthorizationDeniedException
→ usually do not report
```

---

# 211. Render

```text
AuthorizationDeniedException
→ 403/404/401 mapping
```

---

# 212. Report Failure

```text
AuthorizationFailureException
→ error reporting
```

---

# 213. Render Failure

```text
→ safe 500/503 response
```

---

# 214. Trace Integration

Toda exception deberá incluir o poder correlacionarse con:

```text
trace_id
```

---

# 215. Audit Integration

Un failure crítico puede producir audit:

```text
authorization.execution_failed
```

---

# 216. Audit Decision vs Failure

Audit record podrá distinguir:

```text
decision=DENY
```

de:

```text
outcome=FAILED
```

---

# 217. Suggested Audit Schema Extension

Agregar:

```text
outcome_type
failure_code
failure_phase
```

opcionales.

---

# 218. Example

```text
outcome_type=failed
failure_code=authorization.failure.external_timeout
failure_phase=evaluation
```

---

# 219. Security Event Integration

Algunos failures pueden indicar ataque.

Ejemplo:

```text
invalid signed authorization metadata
```

---

# 220. But Not Every Failure Is Attack

DB timeout no debe marcarse automáticamente como incidente.

---

# 221. Failure Classification

Podrá incluir:

```text
securityRelevant=true/false
```

---

# 222. Explainability

Para DENY:

```text
why denied?
```

puede explicarse.

Para FAILURE:

```text
why could authorization not be completed?
```

debe ser una explicación diferente.

---

# 223. Example

Developer:

```text
Authorization failed because RelationshipEvaluator timed out.
```

End user:

```text
This operation cannot be completed right now.
```

---

# 224. No Fake Denial Explanation

No decir:

```text
You do not have permission.
```

si realmente el authorization provider estaba caído.

---

# 225. Why

Eso dificulta:

```text
support
operations
incident response
```

y oculta fallos del sistema.

---

# 226. Public Concealment

Aun así, una aplicación de alta seguridad puede deliberadamente retornar:

```text
403
```

para ciertos failures externos.

Pero internamente deben permanecer clasificados como FAILURE.

---

# 227. Transport Policy

La capa HTTP puede decidir ocultar diferencias externas.

---

# 228. Internal Semantics Remain Correct

Nunca sacrificar el modelo interno por conveniencia del transport.

---

# 229. Authorization Failure Context

Objeto conceptual:

```php
final readonly class AuthorizationFailureContext
{
    public function __construct(
        public string $traceId,
        public AuthorizationFailurePhase $phase,
        public string $ability,
        public ?string $evaluatorId = null,
    ) {}
}
```

---

# 230. Failure Normalizer

```php
interface AuthorizationFailureNormalizerInterface
{
    public function normalize(
        Throwable $throwable,
        AuthorizationFailureContext $context,
    ): AuthorizationFailure;
}
```

---

# 231. Why Normalizer

Adapters externos pueden lanzar excepciones propias.

Ejemplo:

```text
RedisException
PDOException
HttpTimeoutException
```

Authorization puede normalizarlas a categorías estables.

---

# 232. Preserve Original

El Throwable original permanece para logging.

---

# 233. Failure Mapper Registry

Podrá mapear:

```text
exception class
→ failure code/category
```

---

# 234. Example

```text
ExternalPdpTimeoutException
→ external_timeout
```

---

# 235. Unknown Throwable

```text
authorization.internal.error
```

---

# 236. Never Include Raw Throwable Message in Public Failure

---

# 237. Error IDs

Cada failure reportado puede generar:

```text
error_id
```

---

# 238. Public Response

```text
Authorization unavailable.
Reference: err_ABC123
```

si la aplicación quiere soporte correlacionable.

---

# 239. Error ID != Trace ID

Podrán relacionarse internamente, pero no siempre se desea exponer trace IDs.

---

# 240. Error Recovery

Authorization deberá distinguir entre:

```text
recoverable infrastructure optimization failure
```

y:

```text
security decision source failure
```

---

# 241. Recoverable

Ejemplo:

```text
decision cache unavailable
```

→ evaluate fresh.

---

# 242. Non-Recoverable

Ejemplo:

```text
authoritative permission database unavailable
```

→ FAILURE.

---

# 243. RecoveryPolicy

Conceptualmente:

```php
enum AuthorizationRecoveryAction: string
{
    case ContinueFresh = 'continue_fresh';
    case SkipOptional = 'skip_optional';
    case Abort = 'abort';
}
```

---

# 244. No Grant Recovery

No incluir:

```text
AssumeGrant
```

---

# 245. Recovery Resolver

Debe basarse en:

```text
failure source
evaluator criticality
cache semantics
ability consistency
```

---

# 246. Example

```text
Plan cache unavailable
→ ContinueFresh
```

---

# 247. Example

```text
Optional metrics collector unavailable
→ SkipOptional
```

---

# 248. Example

```text
Permission provider unavailable
→ Abort
```

---

# 249. Retryability

Separado de recovery inmediato.

```text
Abort
```

puede ser:

```text
retryable later
```

---

# 250. Failure Descriptor

Podrá contener:

```text
code
category
phase
retryability
security relevance
default transport hint
```

---

# 251. Transport Hint

Solo hint.

No status HTTP hardcoded en Core.

---

# 252. Example

```text
external_timeout
hint=temporarily_unavailable
```

---

# 253. Error Handling Pipeline

```text
Throwable
    ↓
Failure Normalizer
    ↓
AuthorizationFailure
    ↓
Recovery Policy
    │
    ├── Continue Fresh
    ├── Skip Optional
    └── Abort
             ↓
    AuthorizationFailureException
             ↓
        Transport Mapper
```

---

# 254. Denial Pipeline

```text
Evaluator Decisions
        ↓
DecisionManager
        ↓
DENY
        ↓
AuthorizationDeniedException
        ↓
Transport Mapper
```

---

# 255. Important Separation

```text
DENIAL PIPELINE
```

y:

```text
FAILURE PIPELINE
```

solo convergen en:

```text
operation blocked
```

---

# 256. Controller Integration

En un Controller:

```php
public function update(Invoice $invoice)
{
    Authorization::authorize('update', $invoice);

    // domain operation
}
```

---

# 257. DENY

```text
AuthorizationDeniedException
        ↓
HTTP exception mapper
        ↓
403/404/etc.
```

---

# 258. FAILURE

```text
AuthorizationFailureException
        ↓
global error handler
        ↓
500/503 safe response
```

---

# 259. Attribute Integration

```php
#[Authorize('invoice.update')]
public function update(Invoice $invoice)
{
}
```

debe producir exactamente las mismas semantics.

---

# 260. Middleware Integration

Igualmente.

No debe existir una semántica diferente entre:

```text
middleware
attribute
controller helper
Gate facade
Policy call
```

---

# 261. Route Authorization Failure

Si route metadata está corrupta:

```text
configuration failure
```

no:

```text
403
```

---

# 262. Route Denial

Si metadata válida evalúa DENY:

```text
403/404
```

según mapping.

---

# 263. Policy Direct Invocation

No se recomienda:

```php
(new InvoicePolicy())->update(...)
```

porque bypassa:

```text
normalization
failure handling
audit
tracing
strategy
```

---

# 264. All Calls Through Authorization Core

Principio:

```text
Policy methods are evaluators,
not public enforcement APIs.
```

---

# 265. Domain Exceptions

Una Policy puede necesitar consultar dominio.

Si el dominio lanza:

```text
InvoiceNotFoundException
```

¿qué ocurre?

---

# 266. Depends on Meaning

Si significa:

```text
subject no longer exists
```

podría normalizarse a:

```text
resource unavailable
```

pero no asumir automáticamente DENY.

---

# 267. Prefer Stable Inputs

Idealmente el Subject ya fue resuelto antes de Policy.

---

# 268. ORM Not Found

Normalmente:

```text
404
```

debería ocurrir antes de autorización cuando route binding seguro lo permita.

---

# 269. Multi-Tenant Caveat

Pero tenant-safe resolution puede integrar authorization/data isolation.

Esto deberá coordinarse con:

```text
13_MULTI_TENANT_AUTHORIZATION_AND_DATA_ISOLATION_SYSTEM
```

---

# 270. Data Isolation Failure

Una query que intente acceder fuera de tenant debería:

```text
return no resource
```

o generar security exception según subsystem.

---

# 271. Authorization Mapping

No convertir indiscriminadamente:

```text
all NotFound
```

en DENY.

---

# 272. Database Exceptions

Nunca deben exponerse como permission errors.

---

# 273. Transaction Failure After Authorization

Si:

```text
Authorization → GRANT
Domain transaction → FAIL
```

eso no es Authorization failure.

---

# 274. Trace Correlation

Puede correlacionarse, pero cada subsystem conserva ownership.

---

# 275. Authorization Does Not Guarantee Business Success

---

# 276. TOCTOU

Existe un riesgo:

```text
authorize
        ↓
state changes
        ↓
execute
```

---

# 277. Example

```text
Invoice unlocked
→ GRANT

Another transaction locks invoice

Current request updates anyway
```

---

# 278. Authorization Error Handling Does Not Solve TOCTOU Alone

Para invariantes críticas se requiere:

```text
transactional domain validation
locking
version checks
```

---

# 279. Reauthorization

En algunos workflows:

```text
fresh authorization immediately before commit
```

puede ayudar.

---

# 280. But Domain Invariants Remain Domain Responsibility

---

# 281. Failure During Nested Authorization

Outer Policy:

```text
GRANT condition A
```

llama inner authorization.

Inner:

```text
FAILURE
```

---

# 282. Default

Propagar failure al outer execution.

No convertirlo en:

```text
inner DENY
```

---

# 283. Why

Outer Policy no puede distinguir correctamente una denegación real de una infraestructura caída.

---

# 284. Explicit Handling

Si un Policy author captura un failure, deberá hacerlo deliberadamente.

---

# 285. Strong Recommendation

No capturar:

```php
catch (Throwable) {
    return false;
}
```

dentro de Policies.

---

# 286. Why

Oculta errores como DENY.

---

# 287. Static Analysis

VoltStack tooling podrá advertir patterns peligrosos en Policies.

---

# 288. Policy Contract

Documentar:

```text
Return decision for authorization semantics.

Throw for unexpected execution failure.
```

---

# 289. Expected Domain Conditions

Ejemplo:

```text
Invoice is locked
```

no lanzar excepción.

Retornar:

```text
DENY invoice.locked
```

---

# 290. Exceptional Conditions

Ejemplo:

```text
database unavailable
```

lanzar/propagar failure.

---

# 291. Never Use Exceptions for Normal Denial Inside Policy

Evitar:

```php
throw new PermissionDeniedException();
```

desde cada Policy.

Preferir:

```php
return Decision::deny('...');
```

---

# 292. Enforcement Boundary Throws

El:

```text
AuthorizationManager
```

convierte final DENY a:

```text
AuthorizationDeniedException
```

cuando se usa `authorize()`.

---

# 293. Benefit

Mantiene evaluadores:

```text
pure-ish
composable
strategy-friendly
testable
```

---

# 294. DecisionManager Failure

Una custom strategy puede fallar.

---

# 295. Example

```text
invalid voter result
division by zero in custom consensus
```

---

# 296. DecisionStrategyExecutionException

Debe producir:

```text
FAILURE
```

---

# 297. Built-In Strategies

Deberán validar inputs y evitar estados imposibles.

---

# 298. Invalid Decision Value

Si un custom evaluator devuelve algo fuera del contrato:

```text
InvalidEvaluatorResultException
```

---

# 299. Result Normalization Failure

Si devuelve:

```text
object inesperado
```

no intentar convertirlo a truthy/falsy arbitrariamente.

---

# 300. Strict Normalization

Solo formatos explícitamente soportados:

```text
bool
Decision
DecisionResult
```

si así lo define el contrato.

---

# 301. `null`

Podría significar:

```text
ABSTAIN
```

solo si está documentado.

---

# 302. Unknown Return Type

```text
FAILURE
```

---

# 303. Security Reason

PHP truthiness no debe decidir autorización accidentalmente.

---

# 304. Example Dangerous

```php
return "false";
```

PHP truthy.

VoltStack no debe interpretarlo como GRANT.

---

# 305. Result Normalizer

Debe usar:

```text
explicit type matching
```

---

# 306. Boot-Time Validation

Cuando sea posible, validar return types declarados.

---

# 307. Error Boundaries by Layer

Propuesta:

```text
AuthorizationManager
    ↓
Planner Boundary
    ↓
Dispatcher Boundary
    ↓
Evaluator Boundary
    ↓
Decision Boundary
    ↓
Audit Boundary
```

---

# 308. Each Boundary Adds Context

Pero conserva:

```text
original cause
trace ID
```

---

# 309. No Excessive Exception Wrapping

Un mismo failure no debería terminar como:

```text
AuthorizationException
  → ManagerException
    → ExecutionException
      → DispatcherException
        → PolicyException
```

sin beneficio.

---

# 310. Rule

Wrap only when:

```text
exception crosses abstraction boundary
and new semantic context is added
```

---

# 311. Exception Message Strategy

Mensajes internos:

```text
Policy [InvoicePolicy::approve] failed while evaluating [invoice.approve].
```

---

# 312. Public Message

Separado:

```text
Authorization could not be completed.
```

---

# 313. Localization

Public messages pueden traducirse.

Exception messages internas no necesitan localization.

---

# 314. Exception Codes

No usar numeric PHP exception code como semántica principal.

Usar:

```text
AuthorizationFailureCode
```

---

# 315. Failure Registry

Podrá existir:

```text
AuthorizationFailureRegistry
```

para documentar:

```text
code
category
severity
retryability
public mapping
```

---

# 316. Custom Failure Codes

Paquetes podrán registrar:

```text
billing.authorization_provider_unavailable
```

bajo reglas de namespace.

---

# 317. Reserved Namespace

```text
authorization.*
```

reservado para Core.

---

# 318. Package Namespace

Ejemplo:

```text
payments.authorization.*
```

---

# 319. Failure Metadata Safety

Metadata deberá pasar por redaction antes de:

```text
logs
audit
traces
```

---

# 320. Error Reporting

VoltStack global error reporter podrá recibir:

```text
AuthorizationFailureException
```

con context seguro.

---

# 321. Example Context

```text
trace_id
ability
failure_code
failure_phase
evaluator_id
```

---

# 322. Do Not Include

```text
raw JWT
session cookie
database credentials
full subject
```

---

# 323. Metrics

Necesarias:

```text
authorization.failures.total
authorization.failures.configuration
authorization.failures.execution
authorization.failures.infrastructure
authorization.failures.external
authorization.failures.context
```

---

# 324. Denial Metrics

Separadas:

```text
authorization.denials.total
authorization.denials.permission
authorization.denials.tenant
authorization.denials.security
```

---

# 325. Why Separate

Un aumento de:

```text
DENY
```

puede ser comportamiento de usuarios.

Un aumento de:

```text
FAILURE
```

puede indicar outage.

---

# 326. SLO

Authorization infrastructure podrá medir:

```text
successful evaluation rate
```

sin contar DENY como error técnico.

---

# 327. Example

```text
1000 checks

600 GRANT
390 DENY
10 FAILURE
```

Technical success rate:

```text
99%
```

No:

```text
60%
```

---

# 328. Latency Metrics

Failures deben registrarse también para detectar timeouts.

---

# 329. Alert

Ejemplo:

```text
external evaluator failure rate > 1%
```

---

# 330. Error Budget

Puede definirse para Authorization infrastructure.

---

# 331. Circuit Breaker Metrics

Pertenecen a resilience layer pero pueden correlacionarse.

---

# 332. Testing Denial

Test:

```text
permission absent
```

debe verificar:

```text
DENY
not FAILURE
```

---

# 333. Testing Policy Bug

Policy lanza:

```text
TypeError
```

debe verificar:

```text
FAILURE
not DENY
```

---

# 334. Testing External Timeout

Debe bloquear operación y clasificar:

```text
external_timeout
```

---

# 335. Testing Cache Failure

Cache backend cae:

```text
fresh evaluation succeeds
```

si cache era opcional.

---

# 336. Testing Required Audit Failure

Debe producir:

```text
FAILED
```

aunque evaluator haya dado GRANT.

---

# 337. Testing 401

Anonymous + authentication-required:

```text
401
```

---

# 338. Testing 403

Authenticated + ordinary DENY:

```text
403
```

---

# 339. Testing 404 Concealment

Cross-tenant DENY:

```text
404
```

sin revelar:

```text
Tenant#9
```

---

# 340. Testing 500

Policy programming error:

```text
500
```

en HTTP integration.

---

# 341. Testing 503

External authorization provider unavailable:

```text
503
```

cuando transport policy lo configure.

---

# 342. Testing `can()`

Failure:

```text
false
```

y:

```text
failure metric/report emitted
```

---

# 343. Testing `inspect()`

Debe distinguir:

```text
Denied
Failed
```

---

# 344. Testing `authorize()`

DENY:

```text
AuthorizationDeniedException
```

FAILURE:

```text
AuthorizationFailureException
```

---

# 345. Testing Redaction

Exception rendering nunca debe incluir secretos.

---

# 346. Testing Nested Failure

Inner authorization failure debe propagarse correctamente.

---

# 347. Testing FrankenPHP

Una exception en Request A no debe contaminar:

```text
AuthorizationSession
trace
failure context
```

de Request B.

---

# 348. Cleanup in `finally`

Todo request-scoped state debe limpiarse incluso cuando:

```text
exception thrown
```

---

# 349. Persistent Worker Safety

Especial atención a:

```text
last failure
last denial
last principal
last trace
```

Nunca mantenerlos en static mutable state.

---

# 350. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Failure/
        ├── Contracts/
        │   ├── AuthorizationFailureNormalizerInterface.php
        │   ├── AuthorizationFailurePolicyResolverInterface.php
        │   └── AuthorizationRecoveryPolicyInterface.php
        │
        ├── Model/
        │   ├── AuthorizationOutcome.php
        │   ├── AuthorizationOutcomeType.php
        │   ├── AuthorizationFailure.php
        │   ├── AuthorizationFailureCode.php
        │   ├── AuthorizationFailureCategory.php
        │   ├── AuthorizationFailurePhase.php
        │   ├── AuthorizationFailureMode.php
        │   ├── AuthorizationFailureRetryability.php
        │   └── AuthorizationFailureContext.php
        │
        ├── Normalization/
        │   ├── AuthorizationFailureNormalizer.php
        │   ├── AuthorizationFailureMapperRegistry.php
        │   └── ThrowableFailureMapper.php
        │
        ├── Recovery/
        │   ├── AuthorizationRecoveryAction.php
        │   ├── AuthorizationRecoveryPolicy.php
        │   └── AuthorizationFailurePolicyResolver.php
        │
        ├── Exceptions/
        │   ├── AuthorizationException.php
        │   ├── AuthorizationDeniedException.php
        │   ├── AuthorizationFailureException.php
        │   ├── AuthorizationConfigurationException.php
        │   ├── AuthorizationContextException.php
        │   ├── AuthorizationResolutionException.php
        │   ├── AuthorizationExecutionException.php
        │   ├── PolicyExecutionException.php
        │   ├── GateExecutionException.php
        │   ├── EvaluatorExecutionException.php
        │   ├── DecisionStrategyExecutionException.php
        │   ├── InvalidEvaluatorResultException.php
        │   ├── AuthorizationInfrastructureException.php
        │   ├── AuthorizationExternalEvaluatorException.php
        │   ├── AuthorizationInternalException.php
        │   ├── UnknownAbilityException.php
        │   ├── InvalidAuthorizationSubjectException.php
        │   └── InvalidPrincipalException.php
        │
        └── Transport/
            ├── AuthorizationTransportMapperInterface.php
            ├── Http/
            │   └── AuthorizationHttpExceptionMapper.php
            ├── Cli/
            │   └── AuthorizationCliExceptionMapper.php
            ├── Queue/
            │   └── AuthorizationQueueFailureMapper.php
            └── Rpc/
                └── AuthorizationRpcFailureMapper.php
```

---

# 351. Integration with Existing Authorization Architecture

El subsistema interactúa con:

```text
AuthorizationManager
AuthorizationPlanner
PolicyDispatcher
GateSystem
DecisionManager
AuthorizationSession
AuthorizationCache
AuthorizationObservability
AuthorizationAudit
```

pero no reemplaza ninguno.

---

# 352. Manager Responsibilities

`AuthorizationManager`:

```text
coordinates execution
defines enforcement boundary
normalizes final outcome
```

---

# 353. Dispatcher Responsibilities

`PolicyDispatcher`:

```text
invoke Policy safely
normalize invocation failures
```

---

# 354. DecisionManager Responsibilities

```text
aggregate valid decisions
```

No debería recibir failures como votos ordinarios.

---

# 355. Important

No:

```text
GRANT
DENY
FAILURE
```

como tres votos iguales.

---

# 356. Correct

```text
GRANT/DENY/ABSTAIN
→ DecisionManager

FAILURE
→ Failure pipeline
```

---

# 357. Cache Responsibilities

```text
cache miss
→ normal

cache optimization failure
→ recover fresh

authoritative source failure
→ failure
```

---

# 358. Observability Responsibilities

```text
record failure
```

sin cambiar outcome salvo audit requerido.

---

# 359. Transport Responsibilities

```text
convert outcome/exception
into protocol-specific behavior
```

---

# 360. Flujo completo — DENY

```text
HTTP Request
    ↓
Controller
    ↓
Authorization::authorize()
    ↓
AuthorizationManager
    ↓
Policy
    ↓
DENY
reason=rbac.permission_missing
    ↓
DecisionManager
    ↓
Final DENY
    ↓
AuthorizationDeniedException
    ↓
HTTP Mapper
    ↓
403 Forbidden
```

---

# 361. Flujo completo — Concealed DENY

```text
TenantContext#7
    ↓
Invoice#928 / Tenant#9
    ↓
TenantIsolationPolicy
    ↓
DENY
tenant.mismatch
    ↓
AuthorizationDeniedException
    ↓
HTTP Mapper
    ↓
404 Not Found
```

Internamente:

```text
DENY
```

Externamente:

```text
404
```

---

# 362. Flujo completo — Policy Failure

```text
Authorization::authorize()
    ↓
InvoicePolicy::approve()
    ↓
PDOException
    ↓
PolicyExecutionException
    ↓
AuthorizationFailure
category=execution
    ↓
FAIL CLOSED
    ↓
AuthorizationFailureException
    ↓
Error Reporter
    ↓
HTTP Mapper
    ↓
500
```

---

# 363. Flujo completo — External Failure

```text
Authorization
    ↓
External Relationship Evaluator
    ↓
timeout
    ↓
ExternalEvaluatorTimeoutException
    ↓
failure policy
    ↓
Abort
    ↓
FAIL CLOSED
    ↓
503
```

---

# 364. Flujo completo — Cache Failure

```text
Decision Cache
    ↓
Redis unavailable
    ↓
Recovery Policy
    ↓
Continue Fresh
    ↓
Policies evaluated
    ↓
GRANT
```

El cache failure se reporta, pero:

```text
authorization succeeds
```

porque el cache no era autoritativo.

---

# 365. Flujo completo — Required Audit Failure

```text
Policies
    ↓
GRANT
    ↓
DecisionManager
    ↓
GRANT
    ↓
Required Audit
    ↓
Storage unavailable
    ↓
AuthorizationAuditException
    ↓
Final Outcome
FAILED
    ↓
Operation blocked
```

---

# 366. Flujo completo — `can()`

```php
if (Authorization::can('update', $invoice)) {
    // execute
}
```

### GRANT

```text
true
```

### DENY

```text
false
```

### FAILURE

```text
false
+
failure reported
```

---

# 367. Flujo completo — `inspect()`

```php
$outcome = Authorization::inspect(
    'update',
    $invoice
);
```

Puede producir:

```text
GRANTED
DENIED
FAILED
```

sin perder información.

---

# 368. Flujo completo — `authorize()`

```php
Authorization::authorize(
    'update',
    $invoice
);
```

Puede:

```text
return successfully

throw AuthorizationDeniedException

throw AuthorizationFailureException
```

---

# 369. Failure Invariants

### Invariante 1

Un failure nunca produce GRANT.

### Invariante 2

Un failure técnico no se convierte silenciosamente en DENY semántico.

### Invariante 3

La operación se bloquea cuando no puede demostrarse autorización de forma segura.

### Invariante 4

Failures contienen códigos estructurados.

### Invariante 5

El Throwable original permanece disponible internamente.

---

# 370. Denial Invariants

### Invariante 1

DENY es una decisión válida.

### Invariante 2

DENY no se reporta como error técnico por defecto.

### Invariante 3

Reason codes no dependen del transport.

### Invariante 4

Policies retornan DENY para condiciones esperadas.

### Invariante 5

El enforcement boundary puede convertir DENY en excepción.

---

# 371. Exception Invariants

### Invariante 1

`AuthorizationDeniedException` está separada de `AuthorizationFailureException`.

### Invariante 2

Excepciones internas no exponen secretos.

### Invariante 3

No se usa el mensaje de excepción como API semántica.

### Invariante 4

No se envuelven excepciones innecesariamente.

### Invariante 5

Toda failure exception puede correlacionarse con trace.

---

# 372. Transport Invariants

### Invariante 1

Authorization Core no conoce HTTP.

### Invariante 2

401, 403, 404, 500 y 503 son decisiones del transport mapper.

### Invariante 3

Una Policy nunca retorna status HTTP.

### Invariante 4

CLI/Queue/RPC pueden mapear la misma semántica de forma diferente.

### Invariante 5

Public responses nunca exponen detalles internos.

---

# 373. Cache Failure Invariants

### Invariante 1

Cache miss no es failure.

### Invariante 2

Cache opcional caído no bloquea si puede evaluarse fresh.

### Invariante 3

Cache corrupto nunca concede acceso.

### Invariante 4

Stale GRANT no se revive automáticamente.

---

# 374. Multi-Tenant Failure Invariants

### Invariante 1

Tenant mismatch es DENY.

### Invariante 2

Missing required TenantContext es failure de contexto.

### Invariante 3

Cross-tenant DENY puede ocultarse como 404.

### Invariante 4

El error público no revela Tenant destino.

---

# 375. Persistent Runtime Invariants

### Invariante 1

Failures no permanecen en static state.

### Invariante 2

AuthorizationSession se limpia después de exception.

### Invariante 3

Trace/failure context no cruza requests.

### Invariante 4

Un worker FrankenPHP sigue siendo seguro después de una excepción.

---

# 376. Arquitectura final

```text
                    Authorization Request
                             │
                             ↓
                     Request Validation
                             │
                 ┌───────────┴───────────┐
                 │                       │
               VALID                   ERROR
                 │                       │
                 ↓                       ↓
          Authorization Plan       Context Failure
                 │                       │
                 ↓                       │
          Evaluator Execution            │
                 │                       │
       ┌─────────┼─────────┐             │
       ↓         ↓         ↓             │
    GRANT      DENY     ABSTAIN       FAILURE
       │         │         │             │
       └─────────┼─────────┘             │
                 ↓                       │
           DecisionManager               │
                 │                       │
          ┌──────┴──────┐                │
          ↓             ↓                │
       GRANT           DENY              │
          │             │                │
          ↓             ↓                │
       Finalize    Denial Exception      │
          │             │                │
          ↓             │                │
       Audit            │                │
          │             │                │
       ┌──┴──┐          │                │
       │     │          │                │
      OK   FAIL         │                │
       │     │          │                │
       ↓     └──────────┼────────────────┘
   SUCCESS              ↓
                  Failure Pipeline
                        │
                        ↓
                   FAIL CLOSED
                        │
                        ↓
                Failure Exception
                        │
                        ↓
                 Transport Mapper
```

---

# 377. Filosofía del sistema

VoltStack deberá seguir estas reglas:

```text
A denial is not a crash.

A crash is not a denial.

An abstention is not a grant.

A cache miss is not an error.

An unavailable authorization dependency is not permission denial.

An unknown ability is not silently false.

A missing TenantContext is not the same as tenant mismatch.

A Policy bug must not become a permission decision.

A failed audit requirement can invalidate operational authorization.

HTTP status codes belong to transport adapters.

And when authorization cannot be determined safely,
execution stops.
```

---

# 378. Resultado esperado

El `Authorization Failure, Error, Denial and Exception Handling System` permitirá que VoltStack mantenga una separación rigurosa entre:

```text
Decision Semantics
        │
        ├── GRANT
        ├── DENY
        └── ABSTAIN

Execution Semantics
        │
        └── FAILURE
```

y después transformar estos resultados mediante:

```text
AuthorizationOutcome
        ↓
Enforcement Boundary
        ↓
Typed Exceptions
        ↓
Transport Mapper
```

De esta forma, una autorización podrá fallar de manera segura sin esconder errores técnicos detrás de falsos `403`, sin conceder acceso ante fallos de infraestructura y sin acoplar Policies a HTTP, CLI, Queue, SPA o RPC.

El principio definitivo será:

```text
If VoltStack knows access must be denied,
return DENY.

If VoltStack knows access may be granted,
return GRANT.

If an evaluator has no opinion,
return ABSTAIN.

If VoltStack cannot determine authorization safely,
return FAILURE and fail closed.

Never confuse those states.
```

Esto proporciona una base sólida para construir autorización predecible, observable, segura y preparada para entornos empresariales y runtimes persistentes como FrankenPHP.