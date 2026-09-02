# VoltStack Authorization System — Authorization Manager and Core Engine

## 1. Propósito

Este documento define el núcleo operativo del **Authorization System de VoltStack**.

Su objetivo es especificar formalmente los componentes responsables de recibir una solicitud de autorización, normalizar sus datos, construir el contexto de evaluación, coordinar la resolución de reglas, ejecutar el motor de decisión y devolver un resultado uniforme.

El núcleo estará centrado principalmente en:

```text
AuthorizationManager
AuthorizationRequest
AuthorizationContext
Ability
Principal
Subject
AuthorizationPlan
DecisionResult
```

El `AuthorizationManager` será el punto central por el que deberán pasar todas las decisiones del framework, independientemente de si fueron iniciadas desde:

```text
$user->can()
Authorization::check()
Authorization::authorize()
Gate::allows()
#[Authorize]
Routing
Controllers
Components
Commands
Jobs
SPA capability projection
```

Todos estos mecanismos deberán converger en un único:

```text
Authorization Core Engine
```

---

# 2. Responsabilidad del Core Engine

El Core Engine debe responder una pregunta fundamental:

```text
¿Puede este Principal ejecutar esta Ability
sobre este Subject dentro de este Context?
```

Representado formalmente como:

```text
Principal
    +
Ability
    +
Subject
    +
AuthorizationContext
        ↓
AuthorizationRequest
        ↓
Authorization Engine
        ↓
DecisionResult
```

El Core Engine no será responsable de:

- autenticar credenciales;
- resolver rutas;
- ejecutar controllers;
- cargar entidades desde la base de datos;
- validar payloads;
- ejecutar lógica de negocio;
- renderizar respuestas HTTP;
- administrar roles directamente;
- administrar permisos directamente.

Su única responsabilidad será:

```text
coordinar decisiones de autorización
```

---

# 3. AuthorizationManager

`AuthorizationManager` será el principal punto de entrada operacional.

Conceptualmente:

```php
final class AuthorizationManager
{
    public function check(...): bool;

    public function cannot(...): bool;

    public function decide(...): DecisionResult;

    public function authorize(...): DecisionResult;
}
```

Será responsable de coordinar:

```text
Input
 ↓
Normalization
 ↓
Principal Resolution
 ↓
Context Resolution
 ↓
Request Construction
 ↓
Authorization Planning
 ↓
Evaluator Execution
 ↓
Decision Aggregation
 ↓
Result Finalization
```

El `AuthorizationManager` no deberá contener directamente todas estas implementaciones.

Actuará como:

```text
orchestrator
```

sobre servicios especializados.

---

# 4. API pública principal

La API mínima propuesta será:

```php
interface AuthorizationManagerInterface
{
    public function check(
        string|Ability $ability,
        mixed $subject = null,
        ?AuthorizationContext $context = null,
        ?PrincipalInterface $principal = null,
    ): bool;

    public function cannot(
        string|Ability $ability,
        mixed $subject = null,
        ?AuthorizationContext $context = null,
        ?PrincipalInterface $principal = null,
    ): bool;

    public function decide(
        string|Ability $ability,
        mixed $subject = null,
        ?AuthorizationContext $context = null,
        ?PrincipalInterface $principal = null,
    ): DecisionResult;

    public function authorize(
        string|Ability $ability,
        mixed $subject = null,
        ?AuthorizationContext $context = null,
        ?PrincipalInterface $principal = null,
    ): DecisionResult;
}
```

Las implementaciones concretas podrán añadir métodos especializados sin romper este contrato base.

---

# 5. Diferencia entre `check()`, `cannot()`, `decide()` y `authorize()`

Estas operaciones compartirán el mismo motor, pero tendrán semánticas diferentes.

## 5.1 `check()`

Devuelve:

```php
bool
```

Ejemplo:

```php
if (Authorization::check('update', $post)) {
    // autorizado
}
```

Internamente:

```text
DecisionResult::GRANT → true

DecisionResult::DENY → false

DecisionResult::ABSTAIN
        ↓
Final policy
        ↓
false
```

En la frontera pública, un `ABSTAIN` final nunca deberá convertirse accidentalmente en autorización.

---

# 6. `cannot()`

Será la negación semántica de `check()`.

```php
if ($user->cannot('delete', $invoice)) {
    // ...
}
```

Equivalente conceptualmente a:

```php
!$user->can('delete', $invoice);
```

pero deberá delegar al mismo Core Engine y no implementar un segundo camino.

---

# 7. `decide()`

`decide()` devolverá información completa.

```php
$result = Authorization::decide(
    'approve',
    $invoice
);
```

Resultado conceptual:

```php
DecisionResult {
    decision: GRANT,
    reason: null,
    metadata: [...]
}
```

Será la API recomendada cuando el consumidor necesite:

- razones;
- metadata;
- trace identifiers;
- información de Policies;
- auditoría;
- decisiones explicables.

---

# 8. `authorize()`

`authorize()` tendrá comportamiento imperativo.

```php
Authorization::authorize(
    'update',
    $post
);
```

Si:

```text
GRANT
```

retorna normalmente.

Si:

```text
DENY
```

lanza:

```text
AuthorizationDeniedException
```

Conceptualmente:

```php
public function authorize(...): DecisionResult
{
    $result = $this->decide(...);

    if (!$result->granted()) {
        throw AuthorizationDeniedException::fromResult($result);
    }

    return $result;
}
```

Esta operación será especialmente apropiada para:

```text
Controllers
Commands
Application Services
```

---

# 9. Flujo único de decisión

Todos los métodos deberán converger internamente en:

```text
decide()
```

Por ejemplo:

```text
check()
   └── decide()

cannot()
   └── decide()

authorize()
   └── decide()
```

Nunca:

```text
check engine
authorize engine
gate engine
controller engine
```

El principio será:

```text
One Authorization Engine
Multiple Public APIs
```

---

# 10. AuthorizationManager como orquestador

El manager deberá depender de componentes especializados.

Ejemplo conceptual:

```php
final class AuthorizationManager
{
    public function __construct(
        private PrincipalResolverInterface $principals,
        private AbilityNormalizerInterface $abilities,
        private SubjectResolverInterface $subjects,
        private AuthorizationContextFactoryInterface $contexts,
        private AuthorizationPlannerInterface $planner,
        private AuthorizationExecutorInterface $executor,
        private DecisionManagerInterface $decisions,
        private AuthorizationResultFinalizerInterface $finalizer,
    ) {}
}
```

Esto evita que el manager se convierta en:

```text
God Object
```

---

# 11. Lifecycle completo

Una llamada:

```php
Authorization::decide('update', $invoice);
```

deberá seguir conceptualmente:

```text
1. Receive Input
2. Normalize Ability
3. Resolve Principal
4. Normalize Subject
5. Build AuthorizationContext
6. Create AuthorizationRequest
7. Build AuthorizationPlan
8. Execute Evaluators
9. Collect DecisionResults
10. Aggregate Decision
11. Apply Default Decision
12. Finalize Result
13. Emit Trace/Audit Hooks
14. Return DecisionResult
```

---

# 12. AuthorizationRequest

Una vez normalizada la entrada, deberá construirse un objeto inmutable.

```php
final readonly class AuthorizationRequest
{
    public function __construct(
        public PrincipalInterface $principal,
        public Ability $ability,
        public SubjectDescriptor $subject,
        public AuthorizationContext $context,
    ) {}
}
```

El `AuthorizationRequest` será la representación canónica interna de una solicitud.

---

# 13. Inmutabilidad del AuthorizationRequest

El objeto deberá ser:

```text
immutable
```

Después de crearse no podrá cambiar:

```text
Principal
Ability
Subject
Context
```

Esto es importante para:

- reproducibilidad;
- tracing;
- seguridad;
- debugging;
- concurrencia;
- runtimes persistentes.

---

# 14. Request Identity

Opcionalmente podrá existir:

```text
AuthorizationRequestId
```

Ejemplo:

```text
authz_01J9M4F8N7Y...
```

Este identificador podrá utilizarse para correlacionar:

```text
Decision
Trace
Audit
Profiler
Security Logs
```

sin depender del identificador HTTP.

---

# 15. Ability

`Ability` representará una acción normalizada.

Ejemplo:

```php
final readonly class Ability
{
    public function __construct(
        public string $name,
    ) {}
}
```

Entrada pública:

```php
'update'
```

Internamente:

```php
new Ability('update')
```

---

# 16. Ability Namespaces

El sistema deberá permitir abilities simples:

```text
view
update
delete
```

y namespaced:

```text
post.update
invoice.approve
admin.users.delete
system.deploy
```

Esto permitirá utilizar el mismo motor tanto para Resource Policies como para Gates globales.

---

# 17. Normalización de Ability

Un componente:

```text
AbilityNormalizer
```

podrá convertir:

```text
UPDATE
posts.update
post:update
```

a una representación consistente según configuración.

Sin embargo, el framework deberá evitar demasiada normalización implícita.

La regla recomendada será:

```text
ability names are explicit and case-sensitive
```

salvo configuración en contrario.

---

# 18. Ability Aliases

Podrá permitirse:

```text
edit → update
remove → delete
```

pero los aliases deberán resolverse durante compilación o bootstrap.

No se recomienda resolver cadenas de aliases dinámicamente en cada request.

---

# 19. Ability Value Object

Una implementación más completa podría ofrecer:

```php
final readonly class Ability
{
    public function __construct(
        private string $name,
    ) {}

    public function name(): string
    {
        return $this->name;
    }

    public function equals(self|string $ability): bool
    {
        // ...
    }

    public function namespace(): ?string
    {
        // ...
    }
}
```

La API pública seguirá aceptando strings para no sacrificar ergonomía.

---

# 20. Principal

El manager deberá poder recibir un Principal explícito:

```php
Authorization::for($user)
    ->check('update', $post);
```

o resolver el actual automáticamente:

```php
Authorization::check('update', $post);
```

---

# 21. Resolución implícita del Principal

Cuando no se proporcione Principal:

```text
AuthorizationManager
        ↓
PrincipalResolver
        ↓
Current Authentication Context
```

El resolver podrá devolver:

```text
Authenticated Principal
```

o:

```text
AnonymousPrincipal
```

Nunca deberá asumir automáticamente una clase concreta `User`.

---

# 22. Principal explícito

El sistema deberá soportar evaluar autorización para otro Principal.

Por ejemplo:

```php
Authorization::for($user)
    ->decide('view', $report);
```

Esto es útil para:

- administración;
- previews;
- capability projection;
- testing;
- batch operations.

---

# 23. `Authorization::for()`

Podrá proporcionarse una API fluida:

```php
Authorization::for($user)
    ->check('update', $invoice);
```

Internamente podrá crear un:

```text
AuthorizationSession
```

o:

```text
BoundAuthorizationManager
```

que simplemente fija el Principal.

---

# 24. Bound Authorization

Ejemplo conceptual:

```php
final readonly class BoundAuthorization
{
    public function __construct(
        private AuthorizationManagerInterface $manager,
        private PrincipalInterface $principal,
    ) {}

    public function check(
        string|Ability $ability,
        mixed $subject = null,
    ): bool {
        return $this->manager->check(
            $ability,
            $subject,
            principal: $this->principal,
        );
    }
}
```

No deberá almacenar estado mutable.

---

# 25. Subject

El subject podrá recibirse como:

```php
$post
```

```php
Post::class
```

```php
AdminController::class
```

```php
null
```

En el caso de Gates globales:

```php
Authorization::check('access-admin');
```

el Subject puede no existir.

---

# 26. SubjectDescriptor

La normalización del Subject permitirá trabajar con una representación consistente.

```php
final readonly class SubjectDescriptor
{
    public function __construct(
        public SubjectType $type,
        public mixed $value,
        public ?string $class = null,
        public string|int|null $identifier = null,
    ) {}
}
```

Tipos posibles:

```text
NONE
OBJECT
CLASS
NAMED
VIRTUAL
```

---

# 27. No ORM lookup en el Core

Si se recibe:

```php
Authorization::check('update', 42);
```

Authorization no debe asumir:

```text
42 = Post ID
```

ni ejecutar:

```text
SELECT * FROM posts WHERE id = 42
```

La resolución del recurso debe haber ocurrido antes.

Por ejemplo:

```text
Route Binding
 ↓
Post#42
 ↓
Authorization
```

---

# 28. AuthorizationContext

El contexto será construido para cada decisión.

Podrá contener información como:

```text
channel
tenant
request
route
controller
runtime
authentication
attributes
security state
```

Ejemplo:

```php
$context = AuthorizationContext::make()
    ->withTenant($tenant)
    ->withAttribute('channel', 'api');
```

La implementación concreta podrá utilizar builders durante construcción y mantener el resultado final inmutable.

---

# 29. Context Factory

El manager no deberá construir manualmente todos los datos.

Utilizará:

```text
AuthorizationContextFactory
```

Entrada:

```text
Optional User Context
+
Environment Providers
```

Salida:

```text
AuthorizationContext
```

---

# 30. Context merge

Cuando el desarrollador proporcione:

```php
Authorization::check(
    'approve',
    $invoice,
    context: $customContext,
);
```

el sistema deberá definir reglas explícitas de merge.

Recomendación:

```text
Explicit Context
    >
Automatically Discovered Context
```

excepto campos protegidos que no deberán poder sobrescribirse arbitrariamente.

---

# 31. Campos protegidos del Context

Ciertos datos podrían considerarse:

```text
trusted context
```

Por ejemplo:

```text
authenticated principal
resolved tenant
runtime identity
security state
```

No deberán poder ser falsificados desde APIs de conveniencia sin pasar por mecanismos explícitos.

---

# 32. Authorization Context Attributes

Podrá existir un espacio para metadata adicional:

```php
$context->attribute('risk_score');
```

Pero deberá evitarse utilizarlo como:

```text
service locator
```

No se deben almacenar dentro:

```text
DatabaseConnection
Container
Mailer
ORM
HTTP Client
```

---

# 33. AuthorizationRequestFactory

La construcción final podrá delegarse a:

```text
AuthorizationRequestFactory
```

Flujo:

```text
Raw Input
   ↓
AbilityNormalizer
   ↓
PrincipalResolver
   ↓
SubjectResolver
   ↓
ContextFactory
   ↓
AuthorizationRequestFactory
   ↓
AuthorizationRequest
```

---

# 34. Authorization Planner

Después de crear la solicitud:

```text
AuthorizationManager
        ↓
AuthorizationPlanner
```

El Planner determinará:

```text
Which evaluators apply?
In what order?
Using which strategy?
In which phases?
```

---

# 35. Separación entre Planning y Execution

Esta separación es crítica.

Incorrecto:

```text
resolve policy
execute policy
resolve another policy
execute policy
...
```

Preferido:

```text
AuthorizationRequest
        ↓
Build Complete Plan
        ↓
Execute Plan
```

Ventajas:

- predictibilidad;
- compilación;
- tracing;
- profiling;
- optimización;
- testing;
- short-circuit controlado.

---

# 36. AuthorizationPlan

Ejemplo:

```php
final readonly class AuthorizationPlan
{
    public function __construct(
        public array $evaluators,
        public DecisionStrategyInterface $strategy,
        public AuthorizationPlanMetadata $metadata,
    ) {}
}
```

Los evaluators podrán ser:

```text
Gate
Global Policy
Tenant Policy
Security Policy
Resource Policy
Context Policy
```

---

# 37. AuthorizationEvaluator

Para unificar Gates y Policies internamente podrá existir:

```php
interface AuthorizationEvaluatorInterface
{
    public function evaluate(
        AuthorizationRequest $request
    ): DecisionResult;
}
```

Así:

```text
GateEvaluator
PolicyEvaluator
CustomEvaluator
```

podrán participar en el mismo plan.

---

# 38. EvaluatorDescriptor

Para evitar instanciar todos los evaluators durante planning podrá utilizarse metadata:

```php
final readonly class EvaluatorDescriptor
{
    public function __construct(
        public string $type,
        public string $target,
        public int $priority = 0,
        public bool $terminal = false,
    ) {}
}
```

El executor resolverá la implementación solo cuando sea necesaria.

---

# 39. AuthorizationExecutor

La ejecución del plan deberá estar separada del manager.

Contrato conceptual:

```php
interface AuthorizationExecutorInterface
{
    public function execute(
        AuthorizationRequest $request,
        AuthorizationPlan $plan,
    ): AuthorizationExecution;
}
```

---

# 40. AuthorizationExecution

En lugar de retornar inmediatamente una sola decisión, podrá producir:

```php
final readonly class AuthorizationExecution
{
    public function __construct(
        public iterable $results,
        public ExecutionMetadata $metadata,
    ) {}
}
```

Esto permitirá entregar todos los resultados al:

```text
DecisionManager
```

---

# 41. Ejecución secuencial

La primera implementación deberá favorecer evaluación secuencial determinista:

```text
Evaluator 1
   ↓
Evaluator 2
   ↓
Evaluator 3
```

Esto simplifica:

- prioridades;
- short-circuit;
- tracing;
- debugging.

No se recomienda paralelizar Policies por defecto.

---

# 42. Short-Circuit

El executor podrá detener el proceso cuando:

```text
Strategy permits terminal decision
```

o cuando un evaluator indique:

```text
Critical DENY
```

Ejemplo:

```text
TenantIsolationPolicy
        ↓
Critical DENY
        ↓
STOP
```

---

# 43. Terminal Decisions

`DecisionResult` podrá incorporar metadata como:

```text
terminal = true
```

Pero deberá estudiarse si esta propiedad pertenece realmente al resultado o al descriptor del evaluator.

La arquitectura recomendada es:

```text
Evaluator Metadata controls execution behavior.
DecisionResult describes decision.
```

Esto mantiene separadas responsabilidades.

---

# 44. DecisionManager

Después de ejecutar el plan:

```text
AuthorizationExecution
        ↓
DecisionManager
```

El manager agregará los resultados usando la estrategia especificada.

---

# 45. Default Decision

Si todos los evaluadores responden:

```text
ABSTAIN
```

o no existe evaluador aplicable:

```text
Final Decision = DENY
```

por defecto.

Este comportamiento podrá formalizarse mediante:

```text
DefaultDenyStrategy
```

o dentro del finalizador.

---

# 46. DecisionResult

El resultado final deberá incluir como mínimo:

```php
final readonly class DecisionResult
{
    public function __construct(
        public Decision $decision,
        public ?string $reason = null,
        public array $metadata = [],
    ) {}
}
```

Podrá evolucionar para incluir:

```text
decision code
reason code
policy
evaluator
trace id
security severity
```

---

# 47. Reason vs Reason Code

Se recomienda diferenciar:

```text
reasonCode
```

de:

```text
reasonMessage
```

Ejemplo:

```text
reasonCode:
tenant_mismatch

reasonMessage:
The resource belongs to another tenant.
```

El código resulta más útil para:

- testing;
- métricas;
- traducción;
- auditoría;
- máquinas.

---

# 48. Decision Codes

Podrán definirse códigos como:

```text
authorization.granted
authorization.denied
authorization.no_rule
authorization.tenant_mismatch
authorization.insufficient_role
authorization.missing_permission
authorization.policy_failure
```

Las Policies de aplicación podrán proporcionar sus propios códigos.

---

# 49. Información sensible

El `DecisionResult` interno podrá contener una razón detallada.

Ejemplo:

```text
User lacks permission finance.invoice.approve
```

pero la capa HTTP podrá convertirlo en:

```text
Forbidden
```

para evitar exposición de información de seguridad.

---

# 50. Result Finalizer

El resultado agregado pasará por:

```text
AuthorizationResultFinalizer
```

responsabilidades posibles:

- aplicar Default Deny;
- adjuntar trace id;
- normalizar reasons;
- eliminar estado interno;
- aplicar metadata final.

No deberá modificar un `DENY` a `GRANT`.

---

# 51. Authorization Outcome

Podrá diferenciarse:

```text
DecisionResult
```

de:

```text
AuthorizationOutcome
```

si posteriormente se necesita una representación externa.

Ejemplo:

```text
Internal DecisionResult
        ↓
HTTP AuthorizationOutcome
```

En V1 puede no ser necesario añadir esta abstracción.

---

# 52. AuthorizationDeniedException

`authorize()` lanzará una excepción específica.

```php
final class AuthorizationDeniedException
    extends AuthorizationException
{
    public function __construct(
        private readonly DecisionResult $result,
    ) {
        parent::__construct(
            $result->reason ?? 'Authorization denied.'
        );
    }
}
```

La excepción deberá conservar internamente el `DecisionResult`.

---

# 53. HTTP mapping

El Authorization Core no deberá conocer:

```text
403
```

El HTTP integration layer será responsable de convertir:

```text
AuthorizationDeniedException
```

en:

```text
HTTP 403 Forbidden
```

Esto mantiene el Core independiente de HTTP.

---

# 54. Anonymous Authorization

Una solicitud anónima podrá evaluarse normalmente:

```text
AnonymousPrincipal
      +
view
      +
PublicArticle
```

La Policy decide.

No toda ausencia de autenticación deberá resultar automáticamente en:

```text
DENY
```

ya que pueden existir recursos públicos.

---

# 55. Authentication-required Authorization

Cuando una acción exija usuario autenticado, podrá expresarse mediante:

```text
AuthenticatedPrincipalPolicy
```

o metadata equivalente.

Ejemplo:

```text
AnonymousPrincipal
      ↓
RequiresAuthenticatedPrincipal
      ↓
DENY
```

Esto evita mezclar autenticación con autorización.

---

# 56. Superuser / Before Hooks

VoltStack podrá permitir reglas globales equivalentes conceptualmente a:

```text
before()
```

pero deberán implementarse mediante el mismo pipeline.

Por ejemplo:

```text
SuperAdminPolicy
priority 10000
```

Resultado:

```text
GRANT terminal
```

si la configuración permite bypass.

---

# 57. Bypass de seguridad

Los mecanismos de:

```text
superuser
root
administrator bypass
```

deberán ser explícitos.

No deberán introducirse automáticamente como comportamiento mágico dentro del AuthorizationManager.

Preferido:

```text
Global Policy
```

que puede:

```text
GRANT
ABSTAIN
```

---

# 58. Deny Overrides

También podrá existir:

```text
SuspendedPrincipalPolicy
```

con prioridad superior.

Así:

```text
SuperAdminPolicy → GRANT
SuspendedPolicy  → DENY
```

El resultado dependerá de la estrategia configurada.

Las prioridades deberán documentarse cuidadosamente.

---

# 59. `before()` de Policy

Si se desea compatibilidad ergonómica estilo Laravel:

```php
public function before(
    User $user,
    string $ability
): bool|null
{
}
```

el `PolicyDispatcher` podrá traducir:

```text
true  → GRANT
false → DENY
null  → ABSTAIN
```

pero esta será una conveniencia sobre el motor general.

---

# 60. `after()` hooks

Podrá existir un sistema posterior:

```text
Authorization After Hooks
```

para:

- observabilidad;
- auditoría;
- métricas.

No se recomienda que un hook posterior pueda convertir un `DENY` en `GRANT`, salvo que forme explícitamente parte del Decision Engine.

---

# 61. Events en el Core

Podrán emitirse eventos:

```text
AuthorizationRequested
AuthorizationPlanned
AuthorizationEvaluatorExecuted
AuthorizationGranted
AuthorizationDenied
AuthorizationFailed
```

Sin embargo:

```text
events must not be required
```

para ejecutar una autorización.

---

# 62. Event overhead

Cuando no existan listeners:

```text
event dispatch overhead ≈ minimal
```

El framework podrá compilar la ausencia de listeners o utilizar dispatchers optimizados.

---

# 63. Error Handling

Los errores se dividirán conceptualmente entre:

```text
Authorization Denial
```

y:

```text
Authorization System Failure
```

No son equivalentes.

---

# 64. Authorization Denial

Ejemplo:

```text
User does not own resource
```

Resultado:

```text
DENY
```

No debe tratarse como error técnico.

---

# 65. Authorization System Failure

Ejemplo:

```text
Policy throws unexpected exception
```

Esto representa:

```text
execution failure
```

y debe:

1. registrarse;
2. trazarse;
3. fallar de forma segura;
4. producir DENY o excepción de framework según contexto.

---

# 66. Fail-Closed Core

En producción, una falla inesperada nunca deberá causar:

```text
ALLOW
```

La regla fundamental:

```text
Authorization engine uncertainty
            ↓
           DENY
```

---

# 67. Development Mode

En desarrollo podrá lanzarse una excepción detallada:

```text
PolicyExecutionException
```

para facilitar debugging.

Pero el comportamiento seguro deberá mantenerse conceptualmente.

---

# 68. Policy Exception Boundary

El executor deberá envolver la invocación.

```text
Policy
  ↓
Throwable
  ↓
PolicyExecutionBoundary
  ↓
Trace failure
  ↓
Fail Closed
```

---

# 69. Cancellation

El Authorization Core podrá soportar cancelación si el runtime incorpora execution contexts cancelables.

Esto será especialmente útil para:

- requests abortados;
- operaciones distribuidas;
- evaluadores externos.

No es obligatorio para V1.

---

# 70. Async Policies

Las Policies locales deberán ser síncronas por defecto.

No se recomienda que una Policy tradicional ejecute:

```text
external HTTP authorization service
```

directamente sin abstracción.

Futuras integraciones remotas deberán utilizar un evaluator especializado.

---

# 71. Authorization Session

Dentro de una request podrá existir una sesión operacional:

```text
AuthorizationSession
```

que contenga únicamente estado request-scoped como:

```text
Principal
Context
Memoization
Trace Collector
```

Nunca deberá sobrevivir entre requests.

---

# 72. Manager shared vs request scoped

El `AuthorizationManager` podrá ser compartido si:

```text
it is stateless
```

y toda información variable se recibe o resuelve desde contexts aislados.

Alternativamente puede existir un manager request-scoped.

La decisión deberá optimizarse para FrankenPHP sin comprometer aislamiento.

---

# 73. Recomendación de lifecycle

Preferencia arquitectónica:

```text
AuthorizationManager
    = shared/stateless service

AuthorizationSession
    = request-scoped
```

Así se reutiliza infraestructura mientras se aísla estado dinámico.

---

# 74. Request-Scoped Memoization

Una sesión podrá almacenar:

```text
ability + principal + subject + context fingerprint
```

para evitar reevaluaciones idénticas.

Pero únicamente cuando la Policy sea:

```text
memoizable
```

---

# 75. Memoization Safety

Una Policy dependiente de:

```text
current time
database state
external service
random values
mutable context
```

no debe memoizarse automáticamente.

Por defecto:

```text
decision memoization = disabled
```

salvo seguridad conocida.

---

# 76. Decision Fingerprint

Podrá existir:

```text
AuthorizationFingerprint
```

compuesto por:

```text
Principal Identity
Ability
Subject Identity
Tenant
Relevant Context Version
```

No deberá basarse únicamente en serializar objetos PHP completos.

---

# 77. Metadata vs Runtime Data

El Core deberá distinguir:

```text
Authorization Metadata
```

de:

```text
Authorization Runtime Data
```

Metadata:

```text
Policy mapping
Policy method
Strategy
Priority
Attribute definitions
```

Runtime:

```text
Principal
Subject instance
Tenant
Request
Decision
```

---

# 78. Metadata reusable

La metadata puede mantenerse:

```text
process-wide
```

si es inmutable.

Esto es especialmente beneficioso para:

```text
FrankenPHP
```

---

# 79. Runtime data isolation

Nunca deberán almacenarse en singletons persistentes:

```text
current user
current tenant
current request
current resource
latest decision
```

---

# 80. FrankenPHP lifecycle

Ejemplo:

```text
Worker Boot
   ↓
Load Compiled Authorization Metadata
   ↓
Request A
   ├── AuthorizationSession A
   └── Destroy/reset
   ↓
Request B
   ├── AuthorizationSession B
   └── Destroy/reset
```

Metadata permanece.

Estado de request desaparece.

---

# 81. Core Engine State Machine

Conceptualmente una autorización atraviesa:

```text
RECEIVED
   ↓
NORMALIZED
   ↓
PLANNED
   ↓
EXECUTING
   ↓
DECIDED
   ↓
FINALIZED
```

En caso de error:

```text
FAILED
```

Esto podrá utilizarse para tracing interno.

---

# 82. Authorization Lifecycle Enum

Opcionalmente:

```php
enum AuthorizationStage
{
    case Received;
    case Normalized;
    case Planned;
    case Executing;
    case Decided;
    case Finalized;
    case Failed;
}
```

No tiene por qué formar parte de la API pública.

---

# 83. Nested Authorization

Una Policy podría intentar ejecutar otra autorización:

```php
Authorization::check(...)
```

Esto debe manejarse cuidadosamente.

Podría producir:

```text
Policy A
 ↓
Authorization
 ↓
Policy A
 ↓
infinite recursion
```

---

# 84. Recursion Guard

El Core deberá disponer de protección frente a ciclos.

Ejemplo:

```text
AuthorizationExecutionStack
```

Si detecta:

```text
same Principal
same Ability
same Subject
same evaluator
```

en la misma cadena, puede lanzar:

```text
CircularAuthorizationException
```

---

# 85. Policies llamando Policies

No se recomienda:

```text
PostPolicy
  ↓
directly call InvoicePolicy
```

Preferido:

```text
Policy composition through Authorization Engine
```

o extracción de reglas compartidas a servicios de dominio/autorización.

---

# 86. Batch Authorization

El Core podrá evolucionar para soportar:

```php
Authorization::batch([
    ['view', $post1],
    ['view', $post2],
    ['update', $post3],
]);
```

Esto sería útil para:

- tablas;
- APIs;
- SPA payloads;
- colecciones.

---

# 87. N+1 Authorization Problem

En listas grandes:

```text
100 resources
×
3 abilities
=
300 evaluations
```

pueden aparecer N+1 queries dentro de Policies.

El Core deberá permitir futuras optimizaciones sin modificar la API individual.

---

# 88. Batch-Aware Policies

En versiones posteriores una Policy podría implementar:

```text
BatchAuthorizationPolicyInterface
```

para evaluar múltiples subjects eficientemente.

No debe ser requisito de V1.

---

# 89. Capability Matrix

Sobre batch authorization podrá construirse:

```text
Capability Matrix
```

Ejemplo:

```text
Invoice 101
  view    true
  update  true
  delete  false

Invoice 102
  view    true
  update  false
  delete  false
```

Esto será especialmente relevante para el frontend SPA.

---

# 90. `inspect()` alias

Podrá ofrecerse:

```php
Authorization::inspect(
    'update',
    $post
);
```

como alias de:

```php
decide()
```

si se desea una API familiar para desarrolladores Laravel.

Internamente no será un camino diferente.

---

# 91. `allows()` y `denies()`

La fachada podrá soportar:

```php
Gate::allows(...)
Gate::denies(...)
```

que mapearán a:

```text
AuthorizationManager::check()
```

y:

```text
AuthorizationManager::cannot()
```

respectivamente.

---

# 92. User-facing `can()`

La clase autenticable podrá utilizar un trait:

```php
trait Authorizable
{
    public function can(
        string|Ability $ability,
        mixed $subject = null
    ): bool {
        return Authorization::for($this)
            ->check($ability, $subject);
    }
}
```

Así la clase User no implementa el motor.

---

# 93. `canAny()`

Podrá existir:

```php
$user->canAny(
    ['update', 'delete'],
    $post
);
```

Semántica:

```text
at least one ability granted
```

Cada ability sigue siendo una decisión independiente.

---

# 94. `canAll()`

VoltStack podrá ofrecer:

```php
$user->canAll(
    ['view', 'update'],
    $post
);
```

Semántica:

```text
all abilities granted
```

No deberá confundirse con la estrategia `Unanimous`, que agrega evaluadores de una misma solicitud.

---

# 95. Distinción crítica

```text
canAll(['view', 'update'])
```

evalúa:

```text
two AuthorizationRequests
```

Mientras:

```text
UnanimousStrategy
```

evalúa múltiples Policies de:

```text
one AuthorizationRequest
```

---

# 96. Authorization Macros

No se recomienda permitir modificación arbitraria del Core mediante macros globales.

La extensibilidad debe realizarse mediante contratos:

```text
Resolvers
Evaluators
Strategies
Context Providers
Finalizers
```

Esto preserva previsibilidad.

---

# 97. Configuration

Configuración conceptual:

```php
return [

    'default_strategy' => 'unanimous',

    'default_decision' => 'deny',

    'anonymous_principal' => true,

    'memoization' => false,

    'fail_closed' => true,

];
```

La configuración definitiva se especificará posteriormente.

---

# 98. Bootstrap

Durante bootstrap:

```text
AuthorizationServiceProvider
        ↓
Load Config
        ↓
Load Compiled Metadata
        ↓
Build PolicyRegistry
        ↓
Build GateRegistry
        ↓
Register Strategies
        ↓
Create AuthorizationManager
```

No deberán escanearse todos los archivos del proyecto durante cada request.

---

# 99. Compiled Core Metadata

En producción podrán cargarse estructuras como:

```php
return [
    'policies' => [
        Invoice::class => InvoicePolicy::class,
    ],

    'abilities' => [
        // ...
    ],

    'controllers' => [
        // ...
    ],
];
```

El Core utilizará estas estructuras directamente.

---

# 100. Zero-reflection hot path

El objetivo de producción será que:

```php
$user->can('update', $invoice);
```

pueda resolverse normalmente sin:

```text
Filesystem Scan
Attribute Reflection
Class Discovery
Convention Search
```

durante la request.

---

# 101. Container Resolution

El executor podrá resolver Policies de forma lazy.

Ejemplo:

```text
AuthorizationPlan
     ↓
PolicyDescriptor
     ↓
Policy actually needed?
     ↓ yes
Container::get()
```

Esto evita construir Policies que nunca serán ejecutadas por short-circuit.

---

# 102. Stateless Policies

Las Policies deberán ser compartibles únicamente si son realmente stateless.

Por seguridad, el framework podrá tratarlas como servicios normales y dejar que el Container determine lifecycle.

La documentación deberá recomendar:

```text
No mutable request state inside Policy instances.
```

---

# 103. Dependency failures

Si una Policy no puede construirse:

```text
Policy Dependency Resolution Failure
```

debe convertirse en:

```text
Authorization Engine Failure
```

y aplicar fail-closed.

---

# 104. Observability Hooks

El Core ofrecerá puntos como:

```text
onRequestCreated
onPlanBuilt
onEvaluatorStarted
onEvaluatorCompleted
onDecisionFinalized
onFailure
```

No necesariamente como eventos públicos.

Pueden ser interfaces internas optimizadas.

---

# 105. Authorization Metrics

Podrán medirse:

```text
authorization.total
authorization.granted
authorization.denied
authorization.failed
authorization.duration
authorization.policy.duration
authorization.cache.hit
authorization.cache.miss
```

sin modificar la decisión.

---

# 106. Audit Triggering

El Core podrá adjuntar metadata:

```text
auditable = true
```

a ciertas abilities.

Ejemplo:

```text
user.delete
invoice.approve
data.export
```

El sistema de auditoría decidirá si persiste la operación.

---

# 107. Sensitive Trace Data

El Core deberá permitir marcar metadata como:

```text
sensitive
```

para evitar que:

- tokens;
- secrets;
- PII innecesaria;
- información interna de seguridad;

sean expuestos por profiler o logs.

---

# 108. Testing Architecture

El Core deberá ser fácilmente testeable sin HTTP.

Ejemplo:

```php
$result = $manager->decide(
    ability: 'update',
    subject: $invoice,
    principal: $user,
    context: $context,
);
```

El test podrá verificar:

```php
expect($result->granted())->toBeTrue();
```

---

# 109. Determinismo

Con:

```text
same Principal
same Ability
same Subject state
same Context
same Policies
same External State
```

el Core deberá producir la misma decisión.

La infraestructura no deberá introducir comportamiento aleatorio.

---

# 110. Fake Authorization Manager

El Testing package podrá proporcionar:

```php
Authorization::fake();
```

para pruebas de integración.

Ejemplo:

```php
Authorization::fake()
    ->grant('invoice.update');
```

Pero los tests de Policies deberán utilizar el motor real cuando se quiera verificar seguridad.

---

# 111. Spy

También podrá soportarse:

```php
Authorization::spy();
```

para verificar:

```text
ability checked
subject checked
decision requested
```

sin cambiar necesariamente el resultado.

---

# 112. Authorization Bypass en Tests

Podrá existir:

```php
Authorization::allowAllForTesting();
```

pero exclusivamente dentro del entorno de testing.

El framework deberá impedir activarlo accidentalmente en producción.

---

# 113. Invariantes del Core Engine

El Core deberá preservar las siguientes invariantes.

## Invariante 1

Toda autorización se representa internamente mediante un `AuthorizationRequest`.

## Invariante 2

Toda respuesta interna termina como `DecisionResult`.

## Invariante 3

`check()`, `cannot()` y `authorize()` utilizan `decide()`.

## Invariante 4

La ausencia de regla aplicable no concede acceso.

## Invariante 5

Los errores inesperados nunca producen autorización implícita.

## Invariante 6

El Authorization Core no depende del transporte HTTP.

## Invariante 7

El Authorization Core no depende obligatoriamente del ORM.

## Invariante 8

El Principal puede ser explícito o resolverse mediante Authentication.

## Invariante 9

El Subject puede representar cualquier tipo de recurso.

## Invariante 10

El contexto de request nunca se almacena globalmente.

## Invariante 11

El planning y la ejecución son fases separadas.

## Invariante 12

Gates y Policies terminan en el mismo Decision Engine.

---

# 114. Flujo resumido de `check()`

```text
$user->can('update', $invoice)
            ↓
BoundAuthorization
            ↓
AuthorizationManager::check()
            ↓
AuthorizationManager::decide()
            ↓
AbilityNormalizer
            ↓
SubjectResolver
            ↓
ContextFactory
            ↓
AuthorizationRequest
            ↓
AuthorizationPlanner
            ↓
AuthorizationExecutor
            ↓
DecisionManager
            ↓
ResultFinalizer
            ↓
DecisionResult::GRANT
            ↓
true
```

---

# 115. Flujo resumido de `authorize()`

```text
Authorization::authorize(
    'delete',
    $invoice
)
        ↓
decide()
        ↓
DecisionResult::DENY
        ↓
AuthorizationDeniedException
        ↓
Integration Layer
        ↓
HTTP 403 / CLI Failure / Job Failure
```

El Core no determina cómo se representa externamente la denegación.

---

# 116. Flujo con Gate

```text
Gate::allows('access-admin')
        ↓
AuthorizationManager
        ↓
AuthorizationRequest
        ↓
AuthorizationPlanner
        ↓
GateEvaluator
        ↓
DecisionResult
```

---

# 117. Flujo con Policy

```text
$user->can('update', $post)
        ↓
AuthorizationManager
        ↓
AuthorizationRequest
        ↓
PolicyResolver
        ↓
PostPolicy::update()
        ↓
DecisionResult
```

---

# 118. Flujo con Controller Attribute

```text
#[Authorize('update', subject: 'invoice')]
        ↓
Controller Integration
        ↓
AuthorizationMetadata
        ↓
AuthorizationManager
        ↓
AuthorizationRequest
        ↓
Core Engine
```

De esta forma el atributo nunca crea una vía paralela.

---

# 119. Arquitectura final del Core

```text
┌──────────────────────────────────────────────┐
│               PUBLIC APIs                    │
│                                              │
│ can()                                        │
│ cannot()                                     │
│ check()                                      │
│ decide()                                     │
│ authorize()                                  │
│ allows()                                     │
│ #[Authorize]                                 │
└──────────────────────┬───────────────────────┘
                       ↓
┌──────────────────────────────────────────────┐
│          AuthorizationManager                │
│               Orchestrator                   │
└──────────────────────┬───────────────────────┘
                       ↓
          ┌────────────┼────────────┐
          ↓            ↓            ↓
      Principal      Ability      Subject
      Resolver      Normalizer    Resolver
          └────────────┼────────────┘
                       ↓
             AuthorizationContext
                       ↓
             AuthorizationRequest
                       ↓
┌──────────────────────────────────────────────┐
│          AuthorizationPlanner                │
└──────────────────────┬───────────────────────┘
                       ↓
              AuthorizationPlan
                       ↓
┌──────────────────────────────────────────────┐
│          AuthorizationExecutor               │
│                                              │
│ Gate Evaluators                              │
│ Global Policies                              │
│ Tenant Policies                              │
│ Security Policies                            │
│ Resource Policies                            │
└──────────────────────┬───────────────────────┘
                       ↓
             Individual Results
                       ↓
┌──────────────────────────────────────────────┐
│             DecisionManager                  │
└──────────────────────┬───────────────────────┘
                       ↓
                DecisionResult
                       ↓
┌──────────────────────────────────────────────┐
│          AuthorizationFinalizer              │
│                                              │
│ Default Deny                                 │
│ Trace                                        │
│ Audit metadata                               │
└──────────────────────┬───────────────────────┘
                       ↓
            GRANT / DENY / ABSTAIN
```

---

# 120. Estructura de clases propuesta

```text
Quantum/
└── Authorization/
    ├── Contracts/
    │   ├── AuthorizationManagerInterface.php
    │   ├── AuthorizationExecutorInterface.php
    │   ├── AuthorizationPlannerInterface.php
    │   ├── PrincipalResolverInterface.php
    │   ├── SubjectResolverInterface.php
    │   ├── AbilityNormalizerInterface.php
    │   └── AuthorizationContextFactoryInterface.php
    │
    ├── Core/
    │   ├── AuthorizationManager.php
    │   ├── AuthorizationRequest.php
    │   ├── AuthorizationRequestFactory.php
    │   ├── AuthorizationSession.php
    │   ├── BoundAuthorization.php
    │   └── AuthorizationStage.php
    │
    ├── Ability/
    │   ├── Ability.php
    │   ├── AbilityNormalizer.php
    │   └── AbilityRegistry.php
    │
    ├── Principal/
    │   ├── PrincipalResolver.php
    │   └── AnonymousPrincipal.php
    │
    ├── Subject/
    │   ├── SubjectResolver.php
    │   ├── SubjectDescriptor.php
    │   └── SubjectType.php
    │
    ├── Context/
    │   ├── AuthorizationContext.php
    │   ├── AuthorizationContextFactory.php
    │   └── Providers/
    │
    ├── Planning/
    │   ├── AuthorizationPlanner.php
    │   ├── AuthorizationPlan.php
    │   └── EvaluatorDescriptor.php
    │
    ├── Execution/
    │   ├── AuthorizationExecutor.php
    │   ├── AuthorizationExecution.php
    │   └── AuthorizationExecutionStack.php
    │
    ├── Decisions/
    │   ├── Decision.php
    │   ├── DecisionResult.php
    │   ├── DecisionManager.php
    │   └── AuthorizationResultFinalizer.php
    │
    └── Exceptions/
        ├── AuthorizationException.php
        ├── AuthorizationDeniedException.php
        ├── AuthorizationExecutionException.php
        └── CircularAuthorizationException.php
```

La ubicación final podrá refinarse conforme avancen las siguientes especificaciones.

---

# 121. Filosofía del Core Engine

La filosofía central será:

```text
Normalize once.
Plan once.
Execute predictably.
Decide explicitly.
Fail closed.
Expose simply.
```

Desde el punto de vista del desarrollador:

```php
$user->can('update', $invoice);
```

Desde el punto de vista del framework:

```text
Principal
    +
Ability
    +
Subject
    +
Context
        ↓
AuthorizationRequest
        ↓
AuthorizationPlan
        ↓
AuthorizationExecution
        ↓
DecisionManager
        ↓
DecisionResult
```

---

# 122. Resultado esperado

El `AuthorizationManager` y el Core Engine deberán constituir la infraestructura estable sobre la que se construyan todos los demás componentes del Authorization System.

El núcleo permitirá que VoltStack mantenga una experiencia sencilla:

```php
$user->can('update', $post);
```

sin sacrificar capacidades avanzadas como:

```text
Multiple Policies
Decision Strategies
Tenant Isolation
Context-Aware Authorization
Explainable Decisions
Controller Policies
Route Authorization
Batch Evaluation
Persistent Runtime Safety
Observability
Audit
```

El principio arquitectónico fundamental queda definido como:

```text
One authorization request.
One authorization engine.
One normalized decision model.
Multiple ways to consume it.
```

Esto permitirá que Policies, Gates, Controllers, Routes, Components, Commands, Jobs y futuros subsistemas de VoltStack compartan una infraestructura de autorización única, consistente y extensible.