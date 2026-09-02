# VoltStack Authorization System — Policy Dispatcher, Invocation and Result Normalization System

## 1. Propósito

Este documento define el subsistema responsable de **instanciar, preparar, invocar y normalizar la ejecución de Policies** dentro del Authorization System de VoltStack.

Sus responsabilidades principales serán:

```text
Policy Instance Resolution
Policy Invocation Planning
Ability Method Resolution
Argument Mapping
before() Handling
Policy Method Invocation
Return Normalization
after() Handling
Exception Boundaries
Execution Metadata
Persistent Runtime Safety
```

El objetivo es convertir una Policy declarada de forma sencilla:

```php
final class InvoicePolicy
{
    public function update(
        User $user,
        Invoice $invoice,
    ): bool {
        return $invoice->user_id === $user->id;
    }
}
```

en una evaluación interna normalizada:

```text
AuthorizationRequest
        ↓
PolicyDescriptor
        ↓
PolicyDispatcher
        ↓
InvoicePolicy::update()
        ↓
bool
        ↓
PolicyReturnNormalizer
        ↓
DecisionResult
```

El `PolicyDispatcher` será responsable de ejecutar una Policy.

No será responsable de decidir qué Policies deben participar ni de determinar el resultado global.

---

# 2. Principio arquitectónico

La separación fundamental será:

```text
PolicyResolver
    ↓
determina qué Policy aplica

AuthorizationPlanner
    ↓
determina cuándo se ejecuta

PolicyDispatcher
    ↓
ejecuta una Policy

DecisionManager
    ↓
combina resultados
```

Por tanto:

```text
PolicyDispatcher ≠ PolicyResolver
PolicyDispatcher ≠ DecisionManager
```

El Dispatcher es exclusivamente la frontera de ejecución.

---

# 3. Posición dentro del pipeline

```text
AuthorizationRequest
        ↓
PolicyResolver
        ↓
PolicyDescriptor
        ↓
AuthorizationPlanner
        ↓
PolicyInvocationDescriptor
        ↓
PolicyDispatcher
        ↓
Policy Instance
        ↓
before()
        ↓
Ability Method
        ↓
Return Normalization
        ↓
after()
        ↓
DecisionResult
```

---

# 4. Responsabilidades del PolicyDispatcher

El Dispatcher deberá:

1. recibir `AuthorizationRequest`;
2. recibir metadata de Policy;
3. resolver la instancia de Policy;
4. resolver el tipo de invocación;
5. procesar `before()` si existe;
6. seleccionar el método de Ability;
7. construir los argumentos;
8. invocar el método;
9. capturar excepciones;
10. normalizar el resultado;
11. ejecutar hooks posteriores observacionales;
12. adjuntar metadata de ejecución;
13. devolver `DecisionResult`.

---

# 5. Responsabilidades que no pertenecen al Dispatcher

El Dispatcher no deberá:

```text
discover Policies
scan files
read attributes dynamically in production
decide which Policy wins
build the complete AuthorizationPlan
authenticate users
resolve routes
perform model binding
perform business actions
translate HTTP responses
persist audit logs directly
```

---

# 6. Contrato principal

Conceptualmente:

```php
interface PolicyDispatcherInterface
{
    public function dispatch(
        AuthorizationRequest $request,
        PolicyInvocationDescriptor $invocation,
    ): DecisionResult;
}
```

También podrá existir una API interna más rica:

```php
public function dispatchDetailed(
    AuthorizationRequest $request,
    PolicyInvocationDescriptor $invocation,
): PolicyExecutionResult;
```

---

# 7. PolicyInvocationDescriptor

La ejecución no deberá depender de reflexión repetitiva.

El Planner deberá entregar metadata suficientemente completa.

Ejemplo conceptual:

```php
final readonly class PolicyInvocationDescriptor
{
    public function __construct(
        public string $policyClass,
        public PolicyInvocationType $type,
        public ?string $method,
        public ?PolicyMethodDescriptor $methodMetadata,
        public bool $hasBefore,
        public bool $hasAfter,
        public int $priority,
        public string $policyId,
    ) {}
}
```

---

# 8. PolicyInvocationType

VoltStack deberá soportar varios estilos.

```php
enum PolicyInvocationType: string
{
    case ConventionalMethod = 'conventional_method';
    case EvaluateMethod = 'evaluate_method';
    case Invokable = 'invokable';
}
```

---

# 9. Conventional Policy

Ejemplo:

```php
final class PostPolicy
{
    public function update(
        User $user,
        Post $post,
    ): bool {
        // ...
    }
}
```

Metadata:

```text
InvocationType:
ConventionalMethod

Method:
update
```

---

# 10. Evaluator-style Policy

Ejemplo:

```php
final class TenantIsolationPolicy
{
    public function evaluate(
        AuthorizationRequest $request,
    ): DecisionResult {
        // ...
    }
}
```

Metadata:

```text
InvocationType:
EvaluateMethod
```

---

# 11. Invokable Policy

Ejemplo:

```php
final class MaintenancePolicy
{
    public function __invoke(
        AuthorizationRequest $request,
    ): DecisionResult {
        // ...
    }
}
```

Metadata:

```text
InvocationType:
Invokable
```

---

# 12. Recomendación de uso

Para aplicaciones:

```text
ConventionalMethod
```

deberá ser el estilo preferido.

Para infraestructura avanzada:

```text
EvaluateMethod
Invokable
```

podrán ser más adecuados.

---

# 13. PolicyInstanceResolver

El Dispatcher no deberá hacer:

```php
new InvoicePolicy();
```

directamente.

Utilizará:

```text
PolicyInstanceResolver
```

Contrato conceptual:

```php
interface PolicyInstanceResolverInterface
{
    public function resolve(
        PolicyInvocationDescriptor $descriptor
    ): object;
}
```

---

# 14. Resolución mediante Container

La implementación normal será:

```text
PolicyInvocationDescriptor
        ↓
PolicyInstanceResolver
        ↓
Container
        ↓
Policy Instance
```

Esto permite constructor injection.

---

# 15. Constructor injection

Ejemplo:

```php
final class InvoicePolicy
{
    public function __construct(
        private readonly MembershipService $memberships,
        private readonly ClockInterface $clock,
    ) {}
}
```

El Dispatcher no conoce estas dependencias.

El Container las resuelve.

---

# 16. Policy lifecycle

El Container podrá administrar Policies como:

```text
transient
request-scoped
shared
```

pero el Policy System deberá asumir:

```text
Policies are stateless by default.
```

---

# 17. Shared Policies

Una Policy podrá ser compartida entre requests únicamente si no almacena estado mutable dependiente de:

```text
Principal
Tenant
Request
Subject
Decision
AuthorizationContext
```

---

# 18. Persistent Runtime Safety

Bajo FrankenPHP:

```text
Worker
  ↓
Shared Policy Instance
  ↓
Request A
  ↓
Request B
```

será seguro solo si la Policy es stateless.

Incorrecto:

```php
final class InvoicePolicy
{
    private ?User $currentUser = null;
}
```

Correcto:

```php
final readonly class InvoicePolicy
{
    public function __construct(
        private PermissionService $permissions,
    ) {}
}
```

si `PermissionService` también respeta su lifecycle.

---

# 19. Instance Resolution Failure

Si el Container no puede construir una Policy:

```text
PolicyInstanceResolutionException
```

El Dispatcher deberá tratarlo como:

```text
Authorization System Failure
```

y la frontera superior aplicará:

```text
fail closed
```

---

# 20. Instance type validation

El objeto retornado deberá coincidir con:

```text
descriptor.policyClass
```

Si no:

```text
InvalidPolicyInstanceException
```

Esto protege configuraciones o decorators defectuosos.

---

# 21. Policy decorators

El Container podrá devolver un decorator compatible.

Ejemplo:

```text
TracingInvoicePolicyDecorator
```

si continúa cumpliendo el contrato esperado.

El Dispatcher no deberá depender necesariamente de igualdad exacta de clase cuando exista un proxy/decorator válido.

---

# 22. Invocation Metadata

El Dispatcher deberá trabajar con metadata compilada.

Por ejemplo:

```text
method = update

parameters:
0 → principal
1 → subject
2 → context

return:
bool|DecisionResult
```

Esto evita reflexión runtime.

---

# 23. Policy Method Resolution

Para una Ability:

```text
update
```

el método puede resolverse como:

```text
InvoicePolicy::update()
```

si la metadata así lo establece.

---

# 24. No method-name guessing in hot path

En producción no se recomienda:

```php
method_exists($policy, $ability);
```

para cada autorización.

Preferido:

```text
PolicyMethodDescriptor
```

precompilado.

---

# 25. Missing Method Semantics

Deben distinguirse dos casos.

### Caso A — Policy no maneja la Ability

```text
Policy metadata:
abilities = [view, update]

Request:
delete
```

Resultado:

```text
ABSTAIN
```

o Policy no incluida en el plan.

### Caso B — Metadata dice que maneja `delete`, pero el método no existe

Resultado:

```text
PolicyMethodNotFoundException
```

Esto representa corrupción o error de configuración.

---

# 26. PolicyMethodDescriptor

Conceptualmente:

```php
final readonly class PolicyMethodDescriptor
{
    public function __construct(
        public string $method,
        public array $abilities,
        public array $parameters,
        public PolicyReturnType $returnType,
        public bool $requiresSubject,
        public bool $supportsAnonymous,
    ) {}
}
```

---

# 27. Parameter Mapping

Los parámetros no deberán resolverse usando heurísticas costosas durante cada request.

El compilador deberá producir:

```text
parameter 0 → PRINCIPAL
parameter 1 → SUBJECT
parameter 2 → AUTHORIZATION_CONTEXT
```

---

# 28. PolicyParameterSource

Conceptualmente:

```php
enum PolicyParameterSource: string
{
    case Principal = 'principal';
    case Subject = 'subject';
    case AuthorizationContext = 'authorization_context';
    case Ability = 'ability';
    case AuthorizationRequest = 'authorization_request';
    case ContextComponent = 'context_component';
}
```

---

# 29. Argument Builder

Podrá existir:

```text
PolicyArgumentResolver
```

responsable de construir:

```php
[
    $user,
    $invoice,
    $context,
]
```

a partir de:

```text
AuthorizationRequest
+
PolicyMethodDescriptor
```

---

# 30. PolicyArgumentResolver Contract

```php
interface PolicyArgumentResolverInterface
{
    public function resolve(
        AuthorizationRequest $request,
        PolicyMethodDescriptor $method,
    ): array;
}
```

---

# 31. Principal Argument

Para:

```php
public function update(
    User $user,
    Invoice $invoice
)
```

el mapping será:

```text
parameter 0
    ↓
request.principal
```

---

# 32. Principal type validation

Si la Policy espera:

```php
User $user
```

pero el Principal es:

```text
ServicePrincipal
```

el Dispatcher deberá aplicar una semántica definida.

No deberá dejar que PHP produzca un `TypeError` no controlado.

---

# 33. Principal Incompatibility

Recomendación:

Si la Policy no es compatible con ese tipo de Principal:

```text
ABSTAIN
```

cuando la metadata indique que la Policy simplemente no aplica.

Si la metadata declara compatibilidad pero el tipo es incorrecto:

```text
PolicyInvocationConfigurationException
```

---

# 34. AnonymousPrincipal

Cuando una Policy espera:

```php
?User $user
```

podrá existir un adapter:

```text
AnonymousPrincipal
        ↓
null
```

únicamente para Policies compatibles con este estilo.

---

# 35. Preferred anonymous signature

VoltStack favorecerá:

```php
public function view(
    PrincipalInterface $principal,
    Article $article,
): bool
```

cuando la Policy deba manejar múltiples clases de Principal.

---

# 36. Subject Argument

Para:

```php
public function update(
    User $user,
    Invoice $invoice
)
```

el segundo parámetro se resolverá desde:

```text
AuthorizationRequest.subject.value
```

---

# 37. Class-level Policy

Para:

```php
public function create(
    User $user
): bool
```

no se inyecta subject instance.

Sin embargo, internamente el request podrá contener:

```text
Subject:
Invoice::class
```

---

# 38. Explicit Class Subject Argument

Una Policy avanzada podrá solicitar:

```php
public function create(
    User $user,
    string $subjectClass,
): bool
```

si metadata explícita lo define.

No deberá inferirse automáticamente por tipos primitivos.

---

# 39. AuthorizationContext Argument

Ejemplo:

```php
public function approve(
    User $user,
    Invoice $invoice,
    AuthorizationContext $context,
): DecisionResult
```

mapping:

```text
parameter 2
    ↓
request.context
```

---

# 40. Ability Argument

Un hook o evaluator podrá solicitar:

```php
Ability $ability
```

que se resolverá desde:

```text
request.ability
```

---

# 41. AuthorizationRequest Argument

Policies avanzadas podrán recibir:

```php
AuthorizationRequest $request
```

pero esto deberá reservarse principalmente para:

```text
Global Policies
Security Policies
Infrastructure Policies
```

---

# 42. Context Component Injection

Podrá permitirse:

```php
TenantContext $tenant
SecurityContext $security
```

solo si estos tipos están registrados explícitamente como componentes del AuthorizationContext.

---

# 43. No arbitrary service injection

El method injection no deberá resolver:

```text
DatabaseManager
Mailer
HttpClient
EventDispatcher
```

desde el Container.

Estas dependencias pertenecen al constructor.

---

# 44. Razón

Si se permite DI arbitrario en métodos, el Dispatcher se convertiría en:

```text
secondary service container
```

y las firmas perderían claridad semántica.

---

# 45. Parameter Resolver Registry

Podrá existir internamente:

```text
PolicyParameterResolverRegistry
```

pero limitado a fuentes de autorización conocidas.

---

# 46. Compiled Parameter Mapping

Ejemplo:

```php
[
    ['source' => 'principal'],
    ['source' => 'subject'],
    ['source' => 'context'],
]
```

Esto puede ejecutarse sin reflexión.

---

# 47. `before()` Hook

Una Policy convencional podrá definir:

```php
public function before(
    User $user,
    string $ability
): bool|null {
}
```

El Dispatcher deberá ejecutarlo antes del método específico.

---

# 48. before flow

```text
Policy Instance
      ↓
has before?
      ↓ yes
invoke before()
      ↓
normalize result
      ↓
GRANT / DENY?
      ↓ yes
skip ability method
```

Si:

```text
ABSTAIN
```

se continúa.

---

# 49. before Return Types

Podrá soportar:

```text
bool
null
Decision
DecisionResult
PolicyResponse
```

igual que métodos de Ability.

---

# 50. before `true`

```php
return true;
```

se normaliza a:

```text
GRANT
```

---

# 51. before `false`

```php
return false;
```

se normaliza a:

```text
DENY
```

---

# 52. before `null`

```php
return null;
```

se normaliza a:

```text
ABSTAIN
```

y continúa con el método de Ability.

---

# 53. before DecisionResult

Ejemplo:

```php
return DecisionResult::deny(
    reasonCode: 'account.suspended'
);
```

debe conservar metadata y razón.

---

# 54. before metadata

El resultado deberá indicar internamente:

```text
source:
before_hook
```

para tracing.

---

# 55. before method descriptor

La metadata podrá contener:

```text
beforeMethod = before
beforeParameterMappings = [...]
```

sin usar reflexión runtime.

---

# 56. before no global

Un `before()` pertenece exclusivamente a esa Policy.

No deberá ejecutarse si la Policy no está incluida en el plan.

---

# 57. Global rules

Reglas verdaderamente globales deberán seguir siendo:

```text
Global Policies
```

no duplicarse mediante `before()` en muchas clases.

---

# 58. Ability Method Invocation

Si `before()` abstiene:

```text
PolicyDispatcher
    ↓
invoke configured ability method
```

---

# 59. Direct invocation

Conceptualmente:

```php
$result = $policy->{$method}(...$arguments);
```

Pero deberá estar rodeado por:

```text
PolicyExecutionBoundary
```

---

# 60. PolicyExecutionBoundary

Su responsabilidad será:

```text
catch unexpected throwable
capture execution metadata
classify failure
rethrow normalized framework exception
```

---

# 61. Exception categories

Debe distinguirse:

```text
Authorization Denial
Policy Configuration Failure
Policy Dependency Failure
Policy Execution Failure
```

---

# 62. Denial is not exception

Una Policy normal deberá expresar:

```text
DENY
```

mediante resultado.

No mediante:

```php
throw new AuthorizationDeniedException();
```

dentro de la Policy.

---

# 63. Unexpected Throwable

Ejemplo:

```php
$this->repository->something();
```

lanza:

```text
DatabaseException
```

El Dispatcher deberá envolverlo en:

```text
PolicyExecutionException
```

preservando la causa.

---

# 64. Fail-closed handling

La capa superior podrá transformar una falla en:

```text
DENY
```

para seguridad.

Pero internamente deberá mantenerse diferenciada de una denegación legítima.

---

# 65. Development behavior

En desarrollo podrá propagarse:

```text
PolicyExecutionException
```

con información detallada.

---

# 66. Production behavior

En producción:

```text
PolicyExecutionException
        ↓
Authorization failure handler
        ↓
safe DENY / framework exception boundary
```

sin exponer detalles sensibles al cliente.

---

# 67. PolicyReturnNormalizer

Toda salida deberá pasar por:

```text
PolicyReturnNormalizer
```

Contrato:

```php
interface PolicyReturnNormalizerInterface
{
    public function normalize(
        mixed $value,
        PolicyReturnContext $context,
    ): DecisionResult;
}
```

---

# 68. Valores soportados

V1 deberá soportar:

```text
true
false
null
Decision
DecisionResult
PolicyResponse
```

---

# 69. Boolean normalization

```text
true
 ↓
DecisionResult::grant()

false
 ↓
DecisionResult::deny()
```

---

# 70. Null normalization

Por defecto:

```text
null
 ↓
DecisionResult::abstain()
```

cuando la firma/hook permita abstención.

---

# 71. Strict returns

En `strict_policy_returns`:

```text
null
```

desde un método que debería tomar decisión podrá producir:

```text
InvalidPolicyResultException
```

---

# 72. Decision normalization

```php
Decision::Grant
```

→

```text
DecisionResult::grant()
```

Y equivalente para:

```text
Deny
Abstain
```

---

# 73. DecisionResult

Si ya se recibe:

```php
DecisionResult
```

se utilizará directamente, agregando solo metadata de ejecución que corresponda.

---

# 74. PolicyResponse

Para ergonomía:

```php
return PolicyResponse::deny(
    'Invoice cannot be updated.'
);
```

se convertirá a:

```text
DecisionResult
```

---

# 75. PolicyResponse como API externa

`PolicyResponse` podrá ofrecer métodos amigables:

```php
PolicyResponse::allow();

PolicyResponse::deny(
    message: '...',
    code: 'invoice.locked',
);

PolicyResponse::abstain();
```

Internamente no deberá sobrevivir más allá del normalizador.

---

# 76. Invalid return

Ejemplo:

```php
return 'allowed';
```

debe producir:

```text
InvalidPolicyResultException
```

No se utilizará truthiness de PHP.

---

# 77. Numeric return

Tampoco:

```php
return 1;
```

deberá interpretarse automáticamente como `GRANT`.

El retorno debe ser explícito.

---

# 78. Objects with `__toString()`

No deberán convertirse implícitamente.

Autorización requiere semántica estricta.

---

# 79. PolicyReturnContext

El normalizador podrá recibir información como:

```text
policy
method
hook type
ability
strict mode
```

para producir errores claros.

---

# 80. Normalized metadata

Todo `DecisionResult` emitido por PolicyDispatcher podrá incluir internamente:

```text
policyId
policyClass
method
ability
source
invocationType
```

si tracing está habilitado.

---

# 81. Metadata allocation optimization

Con tracing deshabilitado, no deberán crearse estructuras extensas innecesariamente.

---

# 82. Reason Code

Una Policy puede devolver:

```text
invoice.locked
```

El Dispatcher deberá conservarlo.

---

# 83. Human-readable reason

También podrá conservar:

```text
The invoice is locked.
```

pero no decidir si se expone al usuario.

---

# 84. Decision metadata ownership

El Dispatcher podrá agregar metadata técnica.

La Policy podrá agregar metadata semántica.

Ejemplo:

```text
Policy:
amount_limit = 100000

Dispatcher:
policy_id = invoice_policy
method = approve
```

---

# 85. Reserved Metadata Keys

VoltStack deberá reservar namespaces internos.

Ejemplo:

```text
voltstack.policy.*
voltstack.execution.*
```

Las aplicaciones deberán usar otro namespace.

---

# 86. Metadata collision

Una Policy no deberá poder sobrescribir:

```text
voltstack.policy.id
```

accidentalmente.

El normalizador deberá proteger estas claves.

---

# 87. `after()` Hook

Una Policy podrá declarar:

```php
public function after(
    User $user,
    string $ability,
    DecisionResult $result,
): void {
}
```

pero su función será observacional.

---

# 88. after execution

```text
Ability method
     ↓
Normalized DecisionResult
     ↓
after()
     ↓
same DecisionResult
```

---

# 89. after cannot alter decision

Por defecto:

```text
after()
```

no podrá convertir:

```text
DENY → GRANT
```

ni:

```text
GRANT → DENY
```

---

# 90. Por qué

Modificar decisiones en hooks posteriores crea:

```text
hidden authorization behavior
```

y dificulta debugging.

Las reglas que afectan la decisión deben ser evaluadores explícitos.

---

# 91. after return value

El retorno de `after()` deberá ignorarse o exigirse `void`.

Preferencia:

```php
public function after(...): void
```

---

# 92. after exceptions

Si `after()` es puramente diagnóstico y falla, deberá existir una política clara.

Recomendación:

```text
security-critical after hook
    → fail closed

observability-only hook
    → log failure without changing decision
```

Por ello podría ser mejor separar:

```text
Policy after()
```

de:

```text
Authorization Observers
```

---

# 93. Recomendación para V1

Mantener `after()` opcional y observacional.

Para auditoría/telemetry avanzada:

```text
Authorization Observability System
```

deberá ser el mecanismo preferido.

---

# 94. before + method + after lifecycle

```text
Resolve instance
      ↓
before()
      ↓
ABSTAIN?
 ┌────┴─────┐
 no         yes
 ↓           ↓
result    ability method
             ↓
          normalize
             ↓
             └─────┐
                   ↓
                 after()
                   ↓
              DecisionResult
```

---

# 95. before terminal result

Si `before()` retorna:

```text
GRANT
```

o:

```text
DENY
```

el Ability method no se ejecuta.

---

# 96. after on before result

Debe definirse si `after()` se ejecuta incluso cuando `before()` termina.

Recomendación:

```text
yes
```

porque observa la decisión final de esa Policy.

---

# 97. Hook source metadata

Entonces `after()` podrá saber:

```text
decisionSource:
before
or
ability_method
```

si lo requiere.

---

# 98. PolicyExecutionResult

Internamente podría utilizarse una representación más rica:

```php
final readonly class PolicyExecutionResult
{
    public function __construct(
        public DecisionResult $decision,
        public string $policyId,
        public ?string $method,
        public PolicyExecutionSource $source,
        public int $durationNs,
    ) {}
}
```

---

# 99. DecisionManager input

El DecisionManager podrá recibir:

```text
PolicyExecutionResult
```

o solo:

```text
DecisionResult
```

dependiendo del nivel de metadata necesario.

---

# 100. Recomendación

Mantener:

```text
DecisionResult
```

como representación semántica y:

```text
PolicyExecutionRecord
```

como metadata de ejecución separada.

---

# 101. PolicyExecutionRecord

Ejemplo:

```php
final readonly class PolicyExecutionRecord
{
    public function __construct(
        public string $policyId,
        public string $policyClass,
        public ?string $method,
        public Decision $decision,
        public int $durationNs,
    ) {}
}
```

Solo deberá generarse cuando tracing/profiling lo requiera.

---

# 102. Invocation timing

El Dispatcher podrá medir:

```text
instance resolution time
before time
method time
after time
total time
```

pero no necesariamente en producción normal.

---

# 103. Profiler mode

En modo profiler:

```text
InvoicePolicy::update
instance resolve   0.03 ms
before             0.01 ms
method             0.28 ms
after              0.00 ms
total              0.32 ms
```

---

# 104. Slow Policy Detection

Podrá emitirse diagnóstico si una Policy excede:

```text
configured threshold
```

Esto será responsabilidad de observabilidad, no del Dispatcher Core.

---

# 105. Invocation Caching

La metadata de invocación sí podrá cachearse.

Ejemplo:

```text
InvoicePolicy + update
        ↓
method descriptor
parameter mapping
return normalization
```

---

# 106. Instance caching

La instancia de Policy podrá reutilizarse únicamente según lifecycle del Container.

El Dispatcher no deberá mantener su propio cache paralelo de instancias.

---

# 107. Result caching

El Dispatcher no deberá cachear decisiones globalmente.

Eso pertenece al sistema de decision memoization/cache.

---

# 108. Request-scoped memoization

Si una decisión completa ya fue memoizada, idealmente el Dispatcher ni siquiera será llamado.

---

# 109. Ability aliases

Los aliases deberán estar resueltos antes de llegar al Dispatcher.

Ejemplo:

```text
edit
 ↓
update
```

El Dispatcher recibe:

```text
update
```

canónico.

---

# 110. Subject resolution

Igualmente el Dispatcher recibe:

```text
SubjectDescriptor
```

ya normalizado.

No realiza model binding.

---

# 111. Context validation

El Context ya deberá estar validado antes de Policy dispatch.

El Dispatcher solo resuelve argumentos declarados.

---

# 112. Missing Context Component

Si una Policy requiere:

```text
TenantContext
```

y no existe:

```text
MissingPolicyContextException
```

cuando el descriptor lo declara obligatorio.

---

# 113. Optional Context Component

Si la firma permite:

```php
?TenantContext $tenant
```

podrá inyectarse:

```text
null
```

si metadata lo permite.

---

# 114. Context mismatch

Si el descriptor indica:

```text
TenantContext available
```

pero el objeto tiene tipo incompatible:

```text
PolicyInvocationConfigurationException
```

---

# 115. Method visibility

Los métodos de Ability deberán ser:

```text
public
```

Los métodos privados/protected no podrán declararse como handlers.

---

# 116. Static methods

No se recomienda soportar Policies mediante métodos estáticos.

Razones:

```text
breaks DI
harder testing
less lifecycle control
less extensibility
```

Policies serán servicios de instancia.

---

# 117. Magic methods

No deberá dependerse de:

```php
__call()
```

para resolver abilities.

Esto impediría validación y compilación confiables.

---

# 118. Variadic arguments

No se recomienda soportar:

```php
public function update(...$args)
```

como Policy convencional.

Las firmas deben ser explícitas.

---

# 119. Union types

Podrán soportarse con cuidado:

```php
User|ServicePrincipal $principal
```

si el compilador puede validar el mapping.

No es necesario en V1.

---

# 120. Intersection types

Podrán soportarse cuando PHP y el compilador lo permitan, pero no deberán ser requisito inicial.

---

# 121. Scalar policy arguments

No deberán resolverse automáticamente desde Context por nombre.

Ejemplo:

```php
public function update(
    User $user,
    Invoice $invoice,
    string $channel
)
```

no deberá inferir mágicamente:

```text
channel = context.channel
```

Preferido:

```php
AuthorizationContext $context
```

o tipo especializado.

---

# 122. Explicit parameter attributes

Si en el futuro se desea:

```php
#[FromAuthorizationContext('channel')]
string $channel
```

podrá añadirse explícitamente.

No inferir por nombre.

---

# 123. Subject projections

Tampoco:

```php
int $invoiceId
```

deberá resolverse automáticamente desde:

```text
SubjectDescriptor.identifier
```

sin metadata explícita.

---

# 124. Framework parameter sources

V1 deberá limitarse a:

```text
Principal
Raw Subject
Ability
AuthorizationContext
AuthorizationRequest
registered Context Components
```

---

# 125. Nested Authorization

Una Policy podrá utilizar un `AuthorizationManagerInterface` por constructor.

Ejemplo:

```php
final class ProjectPolicy
{
    public function __construct(
        private AuthorizationManagerInterface $authorization,
    ) {}
}
```

Pero el Core deberá mantener:

```text
AuthorizationExecutionStack
```

---

# 126. Circular invocation

Si:

```text
Policy A
  ↓
authorize same request
  ↓
Policy A
```

se detectará:

```text
CircularAuthorizationException
```

---

# 127. Recursion fingerprint

Podrá considerar:

```text
Principal
Ability
Subject
Policy ID
```

para detectar ciclos.

---

# 128. Maximum nested depth

Además:

```text
authorization.max_nested_depth
```

podrá proteger contra grafos excesivos.

---

# 129. Nested trace

Tracing deberá poder mostrar:

```text
InvoicePolicy::approve
    ↓ nested
CustomerPolicy::manage
```

sin perder correlación.

---

# 130. Cancellation

Si VoltStack introduce cancellation tokens:

```text
PolicyDispatcher
```

podrá comprobarlos entre evaluaciones.

No es requisito de V1.

---

# 131. Remote Policy Calls

El Dispatcher convencional no deberá asumir soporte remoto.

Una Policy que llama a servicios externos lo hará mediante una dependencia explícita.

Pero para motores externos se recomienda:

```text
RemoteAuthorizationEvaluator
```

separado.

---

# 132. Timeouts externos

Cualquier dependencia remota relevante a autorización debe tener:

```text
strict timeout
```

y fallar de forma segura.

---

# 133. Side Effects

Una Policy deberá ser conceptualmente:

```text
read-only evaluation
```

El Dispatcher no puede garantizarlo completamente, pero la documentación y tooling deberán favorecerlo.

---

# 134. Mutation Detection

En tests avanzados podría existir tooling para detectar mutaciones inesperadas en Subjects.

No será responsabilidad del runtime V1.

---

# 135. Transaction context

Una Policy podrá ejecutarse dentro de una transacción existente.

El Dispatcher no deberá iniciar o cerrar transacciones automáticamente.

---

# 136. Database exceptions

Si la Policy consulta datos y falla DB:

```text
PolicyExecutionException
```

No:

```text
normal DENY
```

aunque la frontera de seguridad termine negando acceso.

---

# 137. Policy-specific exceptions

Podrá permitirse una excepción controlada como:

```text
PolicyAbstainException
```

pero no se recomienda.

Resultados explícitos son más claros y rápidos.

---

# 138. No exceptions for normal control flow

Regla:

```text
GRANT / DENY / ABSTAIN
=
return values

unexpected failure
=
exception
```

---

# 139. DecisionResult factories

La API podrá proporcionar:

```php
DecisionResult::grant();

DecisionResult::deny(
    reasonCode: 'invoice.locked',
);

DecisionResult::abstain();
```

---

# 140. Policy helpers

Podrá existir un trait/base helper:

```php
trait BuildsPolicyResponses
{
    protected function allow(...): DecisionResult;
    protected function deny(...): DecisionResult;
    protected function abstain(...): DecisionResult;
}
```

Pero no deberá ser obligatorio.

---

# 141. BasePolicy

VoltStack podrá ofrecer opcionalmente:

```php
abstract class BasePolicy
{
    protected function allow(...): DecisionResult {}

    protected function deny(...): DecisionResult {}
}
```

Sin embargo:

```text
inheritance must remain optional
```

---

# 142. Invocation decorators

El Dispatcher podrá rodearse de decorators:

```text
TracingPolicyDispatcher
ProfilingPolicyDispatcher
SafePolicyDispatcher
```

si esto no añade overhead excesivo.

---

# 143. Preferred architecture

Preferido:

```text
Core PolicyDispatcher
+
optional observer hooks
```

sobre una cadena excesiva de decorators en el hot path.

---

# 144. Policy Observer Hooks

Podrán existir:

```text
beforePolicyInvocation
afterPolicyInvocation
policyInvocationFailed
```

para observabilidad.

No deberán alterar la decisión.

---

# 145. Audit

El Dispatcher no persistirá audit records.

Solo podrá emitir metadata hacia:

```text
AuthorizationAuditor
```

a través de una capa superior.

---

# 146. Security Logging

Errores como:

```text
invalid policy return
policy method mismatch
unexpected policy exception
```

deberán poder registrarse como eventos de seguridad/operación.

---

# 147. Sensitive Errors

Los mensajes originales de excepciones no deberán enviarse directamente a clientes.

---

# 148. Exception sanitization

Ejemplo interno:

```text
SQLSTATE[...]
```

puede quedar en logs seguros.

Respuesta externa:

```text
Forbidden
```

o error genérico apropiado.

---

# 149. PolicyExecutionException

Conceptualmente:

```php
final class PolicyExecutionException
    extends AuthorizationException
{
    public function __construct(
        public readonly string $policyClass,
        public readonly ?string $method,
        Throwable $previous,
    ) {
        parent::__construct(
            'Policy execution failed.',
            previous: $previous,
        );
    }
}
```

---

# 150. InvalidPolicyResultException

Debe incluir:

```text
policy
method
actual return type
expected types
```

en desarrollo.

---

# 151. PolicyMethodNotFoundException

Debe representar inconsistencias entre:

```text
compiled metadata
```

y:

```text
runtime code
```

Puede indicar cache obsoleto.

---

# 152. Stale Metadata Detection

Si el método compilado ya no existe:

```text
compiled cache stale
```

VoltStack podrá sugerir:

```text
rebuild authorization cache
```

en desarrollo.

---

# 153. Production stale cache

En producción, si existe inconsistencia:

```text
fail closed
```

y registrar incidente.

---

# 154. Dev fallback reflection

Podría permitirse:

```text
development only
```

recalcular metadata mediante reflection si el cache está obsoleto.

No deberá ocurrir silenciosamente en producción.

---

# 155. Return Metadata Enrichment

Después de normalización:

```text
DecisionResult
```

podrá enriquecerse con:

```text
policy id
method
source
```

sin mutarlo, generando un nuevo objeto si es inmutable.

---

# 156. DecisionResult immutability

Ejemplo:

```php
$result = $result->withMetadata(
    'voltstack.policy.id',
    $descriptor->policyId,
);
```

debe devolver un nuevo resultado.

---

# 157. Metadata copy cost

Este enriquecimiento deberá optimizarse para no clonar arrays grandes continuamente.

Puede utilizarse metadata estructurada o lazy tracing.

---

# 158. Preferred V1

Mantener `DecisionResult` pequeño:

```text
decision
reasonCode
reason
metadata
```

y guardar detalles técnicos en:

```text
PolicyExecutionRecord
```

separado.

---

# 159. Result Ownership

La Policy decide:

```text
semantic outcome
```

El Dispatcher aporta:

```text
execution outcome
```

El DecisionManager decide:

```text
aggregate outcome
```

---

# 160. Distinción

Ejemplo:

```text
Policy semantic result:
DENY
reason=invoice.locked

Execution result:
success
duration=0.3ms

Aggregate result:
DENY
```

Una Policy puede ejecutar correctamente y denegar.

---

# 161. Failed Execution

Otro caso:

```text
Policy semantic result:
none

Execution result:
failed

Aggregate:
fail closed
```

Esto no debe confundirse con una denegación normal.

---

# 162. PolicyExecutionStatus

Conceptualmente:

```php
enum PolicyExecutionStatus: string
{
    case Completed = 'completed';
    case Failed = 'failed';
    case Skipped = 'skipped';
}
```

Puede ser útil para tracing.

---

# 163. Skipped Policy

Una Policy puede estar en un plan pero ser omitida por:

```text
short-circuit
cancellation
dependency precondition
```

Esto no equivale a `ABSTAIN`.

---

# 164. ABSTAIN vs SKIPPED

```text
ABSTAIN
=
Policy executed and chose not to decide.
```

```text
SKIPPED
=
Policy did not execute.
```

Esta distinción es importante para observabilidad.

---

# 165. Planner filtering

Idealmente Policies no aplicables se excluyen antes.

Así `SKIPPED` quedará principalmente para short-circuit.

---

# 166. Short-Circuit Interaction

El Dispatcher no decide globalmente si debe continuar con la siguiente Policy.

Retorna:

```text
DecisionResult
```

y el Executor/Strategy decide.

---

# 167. Internal Policy Terminal Metadata

El descriptor puede indicar:

```text
terminalOnGrant
terminalOnDeny
```

pero la estrategia final conserva autoridad.

---

# 168. Policy `before()` and global short-circuit

Si `before()` devuelve `GRANT`, esto termina esa Policy.

No necesariamente termina todo el AuthorizationPlan.

---

# 169. Critical distinction

```text
Policy-local short-circuit
```

≠

```text
Authorization-plan short-circuit
```

---

# 170. Local short-circuit

Dentro de:

```text
InvoicePolicy
```

`before()` puede impedir ejecutar:

```text
InvoicePolicy::update()
```

---

# 171. Plan short-circuit

Después de recibir el resultado, el Executor puede decidir:

```text
stop evaluating additional Policies
```

según estrategia.

---

# 172. Batch Invocation

Si una Policy implementa:

```text
BatchPolicyInterface
```

podrá existir:

```text
BatchPolicyDispatcher
```

o un método especializado.

---

# 173. Batch Dispatcher Contract

Conceptualmente:

```php
public function dispatchBatch(
    PrincipalInterface $principal,
    Ability $ability,
    iterable $subjects,
    AuthorizationContext $context,
    PolicyInvocationDescriptor $invocation,
): iterable;
```

No es requisito de V1 base.

---

# 174. Batch normalization

Cada item deberá producir:

```text
Subject → DecisionResult
```

y mantener las mismas reglas de normalización.

---

# 175. Partial batch failure

Debe definirse cuidadosamente.

Recomendación inicial:

```text
unexpected evaluator failure
        ↓
affected batch evaluation fails closed
```

sin asumir `GRANT` para elementos no evaluados.

---

# 176. Policy Dispatcher performance targets

Hot path común:

```text
descriptor already resolved
policy instance resolvable
arguments precompiled
method known
```

deberá aproximarse a:

```text
container lookup
argument array construction
method call
return normalization
```

---

# 177. Avoid runtime reflection

No:

```php
(new ReflectionMethod(...))->invoke(...)
```

en producción normal.

Se utilizará llamada directa dinámica:

```php
$policy->{$method}(...$arguments);
```

con metadata ya validada.

---

# 178. Avoid named argument overhead internally

Los argumentos internos podrán construirse posicionalmente según metadata compilada.

---

# 179. Fast-path boolean

Para Policies simples que retornan bool:

```text
true → static/common grant representation
false → static/common deny representation
```

podría optimizarse evitando allocations excesivas.

---

# 180. Shared DecisionResult instances

Podrían existir resultados inmutables compartidos:

```text
GRANT without metadata
DENY without metadata
ABSTAIN without metadata
```

si benchmarks muestran beneficio.

---

# 181. Metadata disables sharing

Si hay:

```text
reason
reasonCode
metadata
```

deberá crearse una instancia específica.

---

# 182. PHP enums

`Decision` deberá ser enum nativo cuando la versión mínima de PHP lo permita.

---

# 183. Return type compile optimization

Si metadata sabe:

```text
returnType = bool
```

el normalizador puede usar un camino rápido.

---

# 184. Mixed return metadata

Si la Policy declara:

```php
bool|DecisionResult
```

se utilizará normalización general.

---

# 185. Untyped Policy methods

VoltStack podrá soportarlos por compatibilidad, pero deberá emitir warning de tooling.

Recomendación:

```text
explicit return types
```

---

# 186. Strict Policy Signature Mode

Configuración:

```php
'strict_signatures' => true,
```

podrá exigir:

```text
typed principal
typed subject
supported return type
```

durante compilación.

---

# 187. Development warnings

Ejemplo:

```text
InvoicePolicy::update has no return type.
```

VoltStack podrá recomendar:

```php
: bool
```

o:

```php
: DecisionResult
```

---

# 188. Security mode

En aplicaciones empresariales se podrá habilitar:

```text
strict_policy_signatures
strict_policy_returns
```

---

# 189. Invocation Plan Compilation

Durante compilación:

```text
Policy class
    ↓
Reflection
    ↓
Method signatures
    ↓
Parameter mappings
    ↓
Return strategy
    ↓
PolicyInvocationDescriptor
```

---

# 190. Compiled Example

Conceptualmente:

```php
[
    'policy' => InvoicePolicy::class,
    'methods' => [
        'update' => [
            'method' => 'update',
            'parameters' => [
                'principal',
                'subject',
                'context',
            ],
            'return' => 'decision_result',
        ],
    ],
]
```

---

# 191. Runtime Example

Solicitud:

```php
$user->can('update', $invoice);
```

Dispatcher:

```text
descriptor lookup:
InvoicePolicy::update

resolve policy:
Container → InvoicePolicy

arguments:
$user
$invoice
$context

invoke

result:
true

normalize:
GRANT
```

---

# 192. Controller Policy Example

Policy:

```php
final class AdminControllerPolicy
{
    public function access(
        User $user
    ): bool {
        return $user->isAdmin();
    }
}
```

Request Subject:

```text
AdminController::class
```

Invocation:

```text
AdminControllerPolicy::access(
    User#42
)
```

No existe Subject instance argument.

---

# 193. Global Policy Example

```php
final class SuspendedPrincipalPolicy
{
    public function evaluate(
        AuthorizationRequest $request
    ): DecisionResult {
        // ...
    }
}
```

Dispatcher:

```text
InvocationType = EvaluateMethod
        ↓
evaluate($request)
```

---

# 194. before Example

```php
final class InvoicePolicy
{
    public function before(
        User $user,
        string $ability
    ): ?bool {
        if ($user->isSuperAdmin()) {
            return true;
        }

        return null;
    }

    public function update(
        User $user,
        Invoice $invoice
    ): bool {
        return $invoice->owner_id === $user->id;
    }
}
```

Flow:

```text
before()
  ↓
SuperAdmin?
  ↓ yes
GRANT
  ↓
skip update()
```

---

# 195. before ABSTAIN Example

```text
before()
  ↓
null / ABSTAIN
  ↓
update()
  ↓
DENY
```

Final Policy result:

```text
DENY
```

---

# 196. DecisionResult Example

```php
public function update(
    User $user,
    Invoice $invoice
): DecisionResult {
    if ($invoice->locked) {
        return DecisionResult::deny(
            reasonCode: 'invoice.locked',
            reason: 'The invoice is locked.',
        );
    }

    return DecisionResult::grant();
}
```

---

# 197. after Example

```php
public function after(
    User $user,
    string $ability,
    DecisionResult $result,
): void {
    // observación local opcional
}
```

No podrá reemplazar `$result`.

---

# 198. Error Example

Policy:

```php
public function update(...): bool
{
    throw new RuntimeException('Database unavailable');
}
```

Flow:

```text
Policy method
    ↓
Throwable
    ↓
PolicyExecutionBoundary
    ↓
PolicyExecutionException
    ↓
Trace / logging
    ↓
Authorization failure handling
    ↓
Fail Closed
```

---

# 199. Invalid Result Example

```php
public function update(...)
{
    return 'yes';
}
```

Flow:

```text
'yes'
 ↓
PolicyReturnNormalizer
 ↓
InvalidPolicyResultException
 ↓
Fail Closed
```

---

# 200. Persistent Worker Example

```text
FrankenPHP Worker Boot
      ↓
Compiled Invocation Metadata
      ↓
Shared Authorization Infrastructure
      ↓
Request A
   ├─ Principal A
   ├─ Context A
   ├─ dispatch
   └─ cleanup
      ↓
Request B
   ├─ Principal B
   ├─ Context B
   ├─ dispatch
   └─ cleanup
```

No debe persistirse:

```text
last principal
last subject
last context
last result
```

en PolicyDispatcher ni Policy.

---

# 201. Dispatcher state

`PolicyDispatcher` deberá ser:

```text
stateless
```

o depender únicamente de servicios stateless/shared-safe.

---

# 202. Request-specific execution state

Si se necesita:

```text
execution stack
trace collector
nested depth
```

deberá obtenerse desde:

```text
request-scoped AuthorizationSession
```

no propiedades mutables globales.

---

# 203. Concurrency

Dos requests podrán invocar simultáneamente la misma Policy compartida siempre que:

```text
Policy is stateless
```

El Dispatcher no deberá introducir locks innecesarios.

---

# 204. Thread-local assumptions

VoltStack no deberá depender de supuestos implícitos de thread-local state.

Todo estado dinámico deberá estar asociado explícitamente al contexto de ejecución.

---

# 205. Invocation Security Invariants

### Invariante 1

El Dispatcher solo ejecuta Policies previamente validadas.

### Invariante 2

Los métodos se seleccionan mediante metadata confiable.

### Invariante 3

No se utiliza truthiness para normalizar resultados.

### Invariante 4

Una excepción nunca produce `GRANT`.

### Invariante 5

Los argumentos provienen exclusivamente de fuentes autorizadas.

### Invariante 6

La inyección arbitraria de servicios en métodos de Policy no está permitida.

### Invariante 7

Un `before()` solo hace short-circuit dentro de su propia Policy.

### Invariante 8

`after()` no modifica decisiones.

### Invariante 9

El Dispatcher no decide la estrategia global.

### Invariante 10

El Dispatcher no cachea decisiones.

---

# 206. Result Normalization Invariants

### Invariante 1

Toda salida válida termina como `DecisionResult`.

### Invariante 2

`true` significa `GRANT`.

### Invariante 3

`false` significa `DENY`.

### Invariante 4

`null` solo significa `ABSTAIN` cuando está permitido.

### Invariante 5

Tipos no soportados producen error.

### Invariante 6

La metadata interna reservada no puede ser sobrescrita por la aplicación.

---

# 207. Runtime Invariants

### Invariante 1

PolicyDispatcher no almacena Principal actual.

### Invariante 2

PolicyDispatcher no almacena Subject actual.

### Invariante 3

PolicyDispatcher no almacena Context actual.

### Invariante 4

Policy instances compartidas deberán ser stateless.

### Invariante 5

La metadata compilada sí puede compartirse entre requests.

---

# 208. Observability Invariants

### Invariante 1

Tracing no modifica decisiones.

### Invariante 2

Profiling no modifica decisiones.

### Invariante 3

Los datos sensibles deben ser filtrados.

### Invariante 4

Con observabilidad deshabilitada no deberán construirse registros costosos innecesariamente.

---

# 209. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Policies/
        ├── Dispatch/
        │   ├── PolicyDispatcher.php
        │   ├── PolicyDispatcherInterface.php
        │   ├── PolicyInstanceResolver.php
        │   ├── PolicyArgumentResolver.php
        │   ├── PolicyExecutionBoundary.php
        │   └── PolicyInvocationDescriptor.php
        │
        ├── Invocation/
        │   ├── PolicyInvocationType.php
        │   ├── PolicyMethodDescriptor.php
        │   ├── PolicyParameterDescriptor.php
        │   ├── PolicyParameterSource.php
        │   ├── PolicyReturnType.php
        │   └── PolicyExecutionSource.php
        │
        ├── Results/
        │   ├── PolicyReturnNormalizer.php
        │   ├── PolicyResponse.php
        │   ├── PolicyExecutionRecord.php
        │   └── PolicyReturnContext.php
        │
        ├── Hooks/
        │   ├── PolicyBeforeHookInvoker.php
        │   └── PolicyAfterHookInvoker.php
        │
        └── Exceptions/
            ├── PolicyDispatchException.php
            ├── PolicyInstanceResolutionException.php
            ├── InvalidPolicyInstanceException.php
            ├── PolicyInvocationConfigurationException.php
            ├── PolicyMethodNotFoundException.php
            ├── MissingPolicyContextException.php
            ├── PolicyExecutionException.php
            └── InvalidPolicyResultException.php
```

---

# 210. Flujo arquitectónico final

```text
┌────────────────────────────────────┐
│        AuthorizationRequest        │
└─────────────────┬──────────────────┘
                  ↓
┌────────────────────────────────────┐
│     PolicyInvocationDescriptor     │
│                                    │
│ policy                             │
│ method                             │
│ parameter mappings                 │
│ hooks                              │
│ invocation type                    │
└─────────────────┬──────────────────┘
                  ↓
┌────────────────────────────────────┐
│       PolicyInstanceResolver       │
└─────────────────┬──────────────────┘
                  ↓
             Policy Instance
                  ↓
┌────────────────────────────────────┐
│            before()                │
└─────────────────┬──────────────────┘
                  ↓
           ABSTAIN / decision
                  │
            ┌─────┴─────┐
            │           │
         ABSTAIN      GRANT/DENY
            │           │
            ↓           │
┌────────────────────────────────────┐
│       Ability Method Invocation    │
└─────────────────┬──────────────────┘
                  ↓
           Raw Return Value
                  ↓
┌────────────────────────────────────┐
│      PolicyReturnNormalizer        │
└─────────────────┬──────────────────┘
                  ↓
            DecisionResult
                  ↓
┌────────────────────────────────────┐
│             after()                │
│         observational only         │
└─────────────────┬──────────────────┘
                  ↓
            DecisionResult
                  ↓
           DecisionManager
```

---

# 211. Flujo de errores

```text
Policy Resolution
      ↓
Instance Resolution
      ↓
Argument Resolution
      ↓
Method Invocation
      ↓
Return Normalization
```

Cualquier fallo técnico:

```text
Throwable
   ↓
PolicyExecutionBoundary
   ↓
Normalized Authorization Exception
   ↓
Tracing / Logging
   ↓
Fail-Closed Handler
```

---

# 212. Flujo optimizado de producción

El objetivo para una Policy convencional será:

```text
PolicyInvocationDescriptor already compiled
        ↓
Container resolve
        ↓
Build positional arguments
        ↓
Call known method
        ↓
Fast normalize return
        ↓
DecisionResult
```

Sin:

```text
filesystem scanning
attribute scanning
method discovery
reflection-based argument resolution
dynamic policy lookup
```

---

# 213. Filosofía del Dispatcher

La filosofía deberá ser:

```text
Resolve metadata before runtime.
Instantiate lazily.
Map arguments explicitly.
Invoke predictably.
Normalize strictly.
Treat failures separately from denials.
Never leak request state.
```

---

# 214. Resultado esperado

El `Policy Dispatcher, Invocation and Result Normalization System` deberá permitir que una Policy sencilla:

```php
final class PostPolicy
{
    public function update(
        User $user,
        Post $post
    ): bool {
        return $post->user_id === $user->id;
    }
}
```

sea tan fácil de escribir como en Laravel, mientras VoltStack internamente dispone de:

```text
Compiled invocation metadata
Explicit parameter mapping
Multiple Principal types
Context-aware invocation
before() / after()
GRANT / DENY / ABSTAIN
Strict return normalization
Exception boundaries
Tracing
Profiling
FrankenPHP-safe lifecycle
```

El principio definitivo será:

```text
The PolicyResolver decides what applies.

The AuthorizationPlanner decides when it runs.

The PolicyDispatcher decides how it is invoked.

The PolicyReturnNormalizer decides how its value
becomes a DecisionResult.

The DecisionManager decides what the combined
authorization outcome means.
```

Con esta separación, VoltStack podrá conservar una API de Policies extremadamente sencilla sin introducir ambigüedad ni lógica oculta en el núcleo del Authorization System.