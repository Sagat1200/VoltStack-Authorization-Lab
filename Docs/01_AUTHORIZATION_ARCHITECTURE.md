# VoltStack Authorization System — Architecture

## 1. Propósito

Este documento define la arquitectura técnica del **Authorization System de VoltStack**, sus capas, componentes principales, contratos, responsabilidades, dependencias y flujo de ejecución.

El sistema debe proporcionar una API simple para el desarrollador:

```php
$user->can('update', $invoice);
```

```php
Authorization::check('update', $invoice);
```

```php
Authorization::authorize('update', $invoice);
```

mientras internamente utiliza una arquitectura desacoplada capaz de soportar:

- Policies.
- Gates.
- Controllers.
- Controller Actions.
- Routes.
- Models.
- Components.
- Commands.
- Jobs.
- Services.
- Multi-tenancy.
- RBAC.
- ABAC.
- PBAC.
- ReBAC.
- decisiones compuestas.
- autorización declarativa.
- compilación.
- caché.
- observabilidad.
- auditoría.
- runtimes persistentes.

La arquitectura se fundamenta en:

```text
Principal
    +
Ability
    +
Subject
    +
Context
    ↓
Authorization Engine
    ↓
DecisionResult
```

---

## 2. Objetivos arquitectónicos

El sistema deberá cumplir los siguientes objetivos.

## 2.1 API sencilla

La complejidad interna no deberá trasladarse al código de aplicación.

```php
if ($user->can('update', $post)) {
    // ...
}
```

deberá seguir siendo suficiente para la mayoría de aplicaciones.

---

## 2.2 Independencia del ORM

Authorization no deberá depender directamente del Database System ni del ORM.

Un `Subject` puede ser:

```text
Entity
DTO
Controller
Route
Command
Job
Component
Service
Resource
Class
Value Object
```

---

## 2.3 Composición

Múltiples evaluadores podrán participar en una misma decisión.

```text
TenantPolicy       → GRANT
OwnershipPolicy    → GRANT
RolePolicy         → ABSTAIN
CompliancePolicy   → DENY
```

Un `DecisionManager` determinará el resultado final.

---

## 2.4 Default Deny

Toda operación protegida que no pueda ser autorizada explícitamente deberá ser rechazada.

```text
No matching authorization rule
             ↓
            DENY
```

---

## 2.5 Fail Closed

Ante errores que impidan establecer de forma segura una autorización:

```text
Invalid Principal
Invalid Subject
Invalid Context
Policy Failure
Resolver Failure
Decision Failure
```

el sistema deberá favorecer la denegación.

---

## 2.6 Explicabilidad

Las decisiones deberán poder indicar:

```text
decision
reason
policy
ability
subject
metadata
trace
```

sin obligar a exponer esta información al cliente HTTP.

---

## 2.7 Alto rendimiento

El hot path deberá minimizar:

- reflexión;
- descubrimiento;
- construcción innecesaria de objetos;
- búsquedas repetitivas;
- resolución innecesaria del Container;
- consultas duplicadas;
- parsing de atributos;
- allocations evitables.

---

## 2.8 Seguridad en runtimes persistentes

La arquitectura deberá funcionar correctamente con:

- FrankenPHP;
- workers persistentes;
- queues;
- servidores long-running.

Nunca deberá existir fuga de:

```text
Principal
Tenant
Request
AuthorizationContext
Decision
```

entre ejecuciones.

---

## 3. Posición dentro de VoltStack

Authorization será un subsistema transversal del framework.

```text
                    VoltStack
                       │
        ┌──────────────┼──────────────┐
        │              │              │
   Authentication    Routing      Controllers
        │              │              │
        └──────────────┼──────────────┘
                       ↓
                 Authorization
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
    Components      Commands         Jobs
```

No pertenece exclusivamente a:

```text
HTTP
```

ni a:

```text
Database
```

Será utilizable independientemente del transporte.

---

## 4. Arquitectura de alto nivel

La arquitectura general será:

```text
┌─────────────────────────────────────────────┐
│              Application API                │
│                                             │
│ $user->can()                                │
│ Authorization::check()                      │
│ Authorization::authorize()                  │
│ Gate::allows()                              │
│ #[Authorize]                                │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│           AuthorizationManager              │
└──────────────────────┬──────────────────────┘
                       │
                       ▼
┌─────────────────────────────────────────────┐
│          AuthorizationRequest               │
│                                             │
│ Principal                                   │
│ Ability                                     │
│ Subject                                     │
│ Context                                     │
└──────────────────────┬──────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     GateResolver  PolicyResolver  Context
                                  Providers
          └────────────┼────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│             Authorization Plan              │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│               Policy Pipeline               │
│                                             │
│ Global → Tenant → Security → Resource       │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│              DecisionManager                │
│                                             │
│ Strategy                                    │
│ Aggregation                                 │
│ Final Decision                              │
└──────────────────────┬──────────────────────┘
                       ▼
┌─────────────────────────────────────────────┐
│               DecisionResult                │
│                                             │
│ GRANT / DENY / ABSTAIN                      │
│ Reason                                      │
│ Metadata                                    │
└──────────────────────┬──────────────────────┘
                       │
             ┌─────────┼─────────┐
             ▼         ▼         ▼
           Trace     Audit     Events
```

---

## 5. Arquitectura por capas

El sistema se dividirá conceptualmente en nueve capas.

```text
1. Public API Layer
2. Request Model Layer
3. Resolution Layer
4. Planning Layer
5. Policy/Gate Layer
6. Decision Layer
7. Integration Layer
8. Compilation & Cache Layer
9. Observability & Audit Layer
```

Cada capa deberá tener responsabilidades claramente delimitadas.

---

## 6. Public API Layer

Esta capa proporciona las interfaces utilizadas por aplicaciones y otros subsistemas.

Ejemplos:

```php
$user->can('update', $invoice);
```

```php
$user->cannot('delete', $invoice);
```

```php
Authorization::check('approve', $loan);
```

```php
Authorization::authorize('publish', $post);
```

```php
Gate::allows('access-admin');
```

La API pública deberá delegar siempre en el mismo motor central.

No existirán motores separados para:

```text
Gate
Policy
Controller Authorization
Route Authorization
Component Authorization
```

Todos deberán converger en:

```text
AuthorizationManager
```

---

## 7. AuthorizationManager

`AuthorizationManager` será la fachada operacional central del sistema.

Responsabilidades:

- recibir solicitudes de autorización;
- normalizar abilities;
- resolver el Principal cuando sea necesario;
- construir `AuthorizationRequest`;
- obtener el contexto;
- solicitar un plan de autorización;
- ejecutar el pipeline;
- obtener el resultado;
- aplicar semántica `check()` o `authorize()`;
- coordinar tracing;
- emitir eventos cuando corresponda.

Interfaz conceptual:

```php
interface AuthorizationManagerInterface
{
    public function check(
        string|Ability $ability,
        mixed $subject = null,
        ?AuthorizationContext $context = null,
    ): bool;

    public function decide(
        string|Ability $ability,
        mixed $subject = null,
        ?AuthorizationContext $context = null,
    ): DecisionResult;

    public function authorize(
        string|Ability $ability,
        mixed $subject = null,
        ?AuthorizationContext $context = null,
    ): DecisionResult;
}
```

La diferencia fundamental será:

```text
check()
   ↓
bool

decide()
   ↓
DecisionResult

authorize()
   ↓
DecisionResult OR AuthorizationDeniedException
```

---

## 8. AuthorizationRequest

Cada evaluación deberá normalizarse internamente como un objeto inmutable.

```php
final readonly class AuthorizationRequest
{
    public function __construct(
        public PrincipalInterface $principal,
        public Ability $ability,
        public mixed $subject,
        public AuthorizationContext $context,
    ) {}
}
```

El objeto no deberá contener lógica de autorización.

Representa exclusivamente:

```text
¿Qué autorización se está solicitando?
```

---

## 9. Ability

Las acciones deberán normalizarse internamente.

En lugar de transportar continuamente strings arbitrarios:

```php
'update'
```

el sistema podrá utilizar:

```php
final readonly class Ability
{
    public function __construct(
        public string $name,
    ) {}
}
```

Esto permitirá posteriormente añadir:

- namespaces;
- validación;
- normalización;
- metadata;
- aliases;
- compilación.

Ejemplos:

```text
post.update
invoice.approve
admin.access
customer.export
```

Sin embargo, la API pública deberá continuar aceptando:

```php
$user->can('update', $post);
```

por ergonomía.

---

## 10. Principal Architecture

El motor no deberá depender directamente de una clase `User`.

Se definirá una abstracción:

```php
interface PrincipalInterface
{
    public function authorizationIdentifier(): string|int;
}
```

Implementaciones posibles:

```text
AuthenticatedUser
AnonymousPrincipal
ServicePrincipal
ApiClientPrincipal
SystemPrincipal
MachinePrincipal
```

Esto permitirá ejecutar Authorization fuera del ciclo HTTP.

---

## 11. PrincipalResolver

Cuando el Principal no se proporcione explícitamente:

```text
AuthorizationManager
        ↓
PrincipalResolver
        ↓
Authentication System
```

El resolver será responsable de traducir el contexto actual en un `PrincipalInterface`.

No será responsable de autenticar.

Authentication ya deberá haber realizado esa operación.

---

## 12. AnonymousPrincipal

La ausencia de usuario autenticado no deberá representarse necesariamente mediante:

```php
null
```

VoltStack podrá utilizar:

```php
AnonymousPrincipal
```

Esto simplifica contratos internos y permite Policies explícitas para usuarios anónimos.

Ejemplo:

```text
PrincipalInterface
       │
       ├── UserPrincipal
       └── AnonymousPrincipal
```

---

## 13. Subject Architecture

Authorization deberá aceptar subjects heterogéneos.

```text
Instance Subject
Class Subject
Named Subject
Virtual Subject
```

Ejemplos:

```php
Authorization::check('update', $post);
```

```php
Authorization::check('create', Post::class);
```

```php
Authorization::check('access', AdminController::class);
```

El sistema no deberá imponer que el subject implemente una interfaz.

---

## 14. SubjectDescriptor

Para evitar que los componentes internos necesiten conocer todas las posibles representaciones, podrá normalizarse el subject mediante:

```php
final readonly class SubjectDescriptor
{
    public function __construct(
        public string $type,
        public mixed $value,
        public ?string $class = null,
        public string|int|null $identifier = null,
    ) {}
}
```

Ejemplo:

```text
type       = object
class      = App\Models\Invoice
identifier = 928
value      = Invoice#928
```

El descriptor será especialmente útil para:

- tracing;
- logging;
- auditoría;
- resolución de Policies;
- caché de metadata.

---

## 15. SubjectResolver

El `SubjectResolver` deberá determinar la representación normalizada del subject.

No deberá realizar consultas de base de datos por defecto.

El binding de recursos pertenece al subsistema correspondiente.

Por ejemplo:

```text
Routing
   ↓
Model Binding
   ↓
Invoice instance
   ↓
Authorization SubjectResolver
```

Esto evita acoplar Authorization al ORM.

---

## 16. AuthorizationContext

`AuthorizationContext` transportará datos auxiliares relevantes para la decisión.

Conceptualmente:

```php
final readonly class AuthorizationContext
{
    public function __construct(
        public ?string $channel = null,
        public mixed $tenant = null,
        public mixed $request = null,
        public mixed $route = null,
        public mixed $controller = null,
        public array $attributes = [],
    ) {}
}
```

El diseño definitivo deberá evitar convertir este objeto en un contenedor arbitrario de servicios.

El contexto contiene:

```text
data
```

no:

```text
dependencies
```

---

## 17. Context Providers

El contexto podrá enriquecerse mediante providers.

Ejemplos:

```text
HttpContextProvider
TenantContextProvider
RouteContextProvider
AuthenticationContextProvider
RuntimeContextProvider
```

Pipeline:

```text
Base Context
    ↓
HTTP Provider
    ↓
Tenant Provider
    ↓
Route Provider
    ↓
Final AuthorizationContext
```

Los providers deberán poder activarse únicamente cuando el entorno correspondiente exista.

---

## 18. Authorization Planner

Antes de ejecutar Policies deberá determinarse:

```text
¿Qué reglas deben participar?
```

Esta responsabilidad corresponderá al:

```text
AuthorizationPlanner
```

Entrada:

```text
AuthorizationRequest
```

Salida:

```text
AuthorizationPlan
```

---

## 19. AuthorizationPlan

El plan será una representación predecible de los evaluadores que deberán ejecutarse.

Ejemplo:

```text
AuthorizationPlan

Pre:
  SuspendedPrincipalPolicy

Tenant:
  TenantIsolationPolicy

Security:
  MfaPolicy

Resource:
  InvoicePolicy

Post:
  ComplianceAuditPolicy

Strategy:
  Unanimous
```

El plan separa:

```text
resolution
```

de:

```text
execution
```

Esta separación será importante para compilación, debugging y optimización.

---

## 20. PolicyResolver

`PolicyResolver` determinará qué Policies corresponden al Subject.

Ejemplo:

```text
Invoice
   ↓
PolicyRegistry
   ↓
InvoicePolicy
```

La resolución podrá utilizar:

1. mapping explícito;
2. metadata compilada;
3. atributos;
4. convenciones;
5. resolvers personalizados.

Orden conceptual:

```text
Compiled Mapping
      ↓
Explicit Mapping
      ↓
Attribute Mapping
      ↓
Convention Mapping
      ↓
Custom Resolver
```

El orden definitivo será especificado posteriormente.

---

## 21. PolicyRegistry

El registro almacenará asociaciones conocidas.

Ejemplo:

```php
[
    Post::class => PostPolicy::class,
    Invoice::class => InvoicePolicy::class,
    Order::class => OrderPolicy::class,
]
```

También deberá soportar Policies no asociadas a modelos.

Ejemplo:

```text
AdminController
       ↓
AdminControllerPolicy
```

---

## 22. Policy Contracts

VoltStack no deberá obligar a todas las Policies a extender una clase base pesada.

Podrá proporcionar contratos opcionales:

```php
interface PolicyInterface
{
}
```

y contratos especializados:

```text
GlobalPolicyInterface
ResourcePolicyInterface
ContextPolicyInterface
TenantPolicyInterface
```

Sin embargo, deberá estudiarse cuidadosamente si realmente son necesarios.

Una Policy convencional deberá poder seguir siendo:

```php
final class InvoicePolicy
{
    public function update(
        User $user,
        Invoice $invoice
    ): bool {
        // ...
    }
}
```

VoltStack deberá favorecer:

```text
convention over mandatory inheritance
```

---

## 23. Policy Invocation

`PolicyDispatcher` será responsable de ejecutar métodos de Policy.

Ejemplo:

```text
ability = update
        ↓
InvoicePolicy
        ↓
InvoicePolicy::update()
```

El dispatcher podrá resolver argumentos como:

```text
Principal
Subject
AuthorizationContext
```

pero deberá evitar convertirse en un sistema de dependency injection arbitrario.

Las dependencias propias de una Policy deberán resolverse mediante constructor injection.

---

## 24. Result Normalization

Para conservar una API sencilla, las Policies podrán retornar:

```php
bool
```

o:

```php
Decision
```

o:

```php
DecisionResult
```

El sistema normalizará:

```text
true
 ↓
GRANT

false
 ↓
DENY

Decision::Abstain
 ↓
ABSTAIN
```

Internamente todo terminará siendo:

```text
DecisionResult
```

---

## 25. Decision

El estado interno fundamental será:

```php
enum Decision
{
    case Grant;
    case Deny;
    case Abstain;
}
```

Significado:

### GRANT

La Policy autoriza.

### DENY

La Policy rechaza.

### ABSTAIN

La Policy no toma una decisión.

---

## 26. DecisionResult

El resultado deberá ser inmutable.

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

Podrán proporcionarse factories:

```php
DecisionResult::grant();
DecisionResult::deny();
DecisionResult::abstain();
```

y:

```php
DecisionResult::deny(
    'Invoice belongs to another tenant.'
);
```

---

## 27. DecisionManager

Cuando exista más de un resultado:

```text
Policy A → GRANT
Policy B → ABSTAIN
Policy C → DENY
```

el `DecisionManager` será responsable de agregarlos.

Interfaz conceptual:

```php
interface DecisionManagerInterface
{
    public function decide(
        AuthorizationRequest $request,
        iterable $results,
        DecisionStrategyInterface $strategy,
    ): DecisionResult;
}
```

---

## 28. Decision Strategies

Las estrategias serán intercambiables.

Contrato:

```php
interface DecisionStrategyInterface
{
    public function decide(
        iterable $results
    ): DecisionResult;
}
```

Implementaciones iniciales:

```text
AffirmativeStrategy
UnanimousStrategy
ConsensusStrategy
PriorityStrategy
FirstApplicableStrategy
```

---

## 29. Unanimous Strategy

Requerirá que ningún evaluador aplicable deniegue.

```text
GRANT
GRANT
ABSTAIN
GRANT
   ↓
GRANT
```

pero:

```text
GRANT
DENY
GRANT
   ↓
DENY
```

Será apropiada para operaciones de alta seguridad.

---

## 30. Affirmative Strategy

Permitirá conceder cuando exista una autorización positiva según las reglas configuradas.

Su comportamiento exacto frente a un `DENY` deberá definirse explícitamente para evitar ambigüedades.

No deberá asumirse automáticamente que:

```text
GRANT > DENY
```

en todos los contextos.

---

## 31. Consensus Strategy

Podrá decidir mediante balance entre resultados.

Ejemplo:

```text
GRANT = 4
DENY  = 2
       ↓
GRANT
```

Las reglas de empate deberán ser configurables y seguras por defecto.

---

## 32. Priority Strategy

Permitirá que determinadas Policies tengan precedencia.

Ejemplo:

```text
TenantIsolationPolicy  priority 1000
SecurityPolicy         priority 900
ResourcePolicy         priority 100
```

Una denegación crítica podrá detener el procesamiento.

---

## 33. First Applicable Strategy

Utilizará la primera Policy que no responda `ABSTAIN`.

```text
Policy A → ABSTAIN
Policy B → ABSTAIN
Policy C → GRANT
                  ↓
                GRANT
```

Puede resultar útil en resoluciones jerárquicas.

---

## 34. Policy Pipeline

El pipeline organiza la ejecución.

```text
AuthorizationRequest
        ↓
Global Policies
        ↓
Tenant Policies
        ↓
Security Policies
        ↓
Resource Policies
        ↓
Context Policies
        ↓
DecisionManager
```

No todas las aplicaciones deberán utilizar todas las etapas.

---

## 35. Short-Circuiting

El sistema deberá soportar finalización anticipada.

Ejemplo:

```text
TenantIsolationPolicy
        ↓
       DENY
        ↓
Critical Denial
        ↓
STOP
```

Esto mejora:

```text
security
performance
predictability
```

No todos los `DENY` deberán necesariamente provocar short-circuit; dependerá de la estrategia y metadata.

---

## 36. Gates Architecture

Los Gates serán una API simplificada sobre el mismo motor.

Ejemplo:

```php
Gate::define('access-admin', function ($user) {
    return $user->isAdmin();
});
```

Internamente:

```text
Gate API
   ↓
GateRegistry
   ↓
GateResolver
   ↓
GateEvaluator
   ↓
DecisionResult
   ↓
DecisionManager
```

No deberán existir semánticas incompatibles entre Gates y Policies.

---

# 37. GateRegistry

Almacenará abilities globales.

Ejemplo:

```text
access-admin
view-monitoring
manage-system
deploy-production
```

Los Gates podrán implementarse mediante:

```text
Closure
Invokable Class
Callable
Service Reference
```

La representación compilable deberá favorecer clases sobre closures en producción cuando resulte necesario.

---

# 38. Autorización declarativa

La arquitectura deberá soportar metadata declarativa.

Ejemplo:

```php
#[Authorize('update', subject: 'invoice')]
public function update(Invoice $invoice)
{
}
```

El atributo no ejecutará directamente la Policy.

Será convertido a:

```text
AuthorizationMetadata
        ↓
AuthorizationRequest
        ↓
AuthorizationManager
```

Esto evita crear un segundo camino de autorización.

---

# 39. AuthorizationMetadata

La metadata declarativa podrá representar:

```text
ability
subject
phase
strategy
priority
context requirements
```

Ejemplo conceptual:

```php
final readonly class AuthorizationMetadata
{
    public function __construct(
        public string $ability,
        public ?string $subject = null,
        public AuthorizationPhase $phase = AuthorizationPhase::Resource,
    ) {}
}
```

---

# 40. Authorization Phases

La integración con HTTP y Controllers utilizará fases explícitas.

Inicialmente:

```text
PRE_RESOLUTION
RESOURCE
POST_RESOLUTION
```

Podrán evolucionar si aparecen necesidades reales.

El objetivo no será crear fases arbitrarias, sino resolver correctamente dependencias.

---

# 41. Pre-Resolution Authorization

Se ejecuta antes de resolver subjects costosos.

Ejemplo:

```php
#[Authorize('access-admin', phase: 'pre')]
final class AdminInvoiceController
{
}
```

Pipeline:

```text
Route
 ↓
Controller Metadata
 ↓
access-admin
 ↓
DENY
```

No se ejecutará:

```text
Model Binding
```

si no es necesario.

---

# 42. Resource Authorization

Se ejecutará cuando el Subject ya esté disponible.

Ejemplo:

```php
#[Authorize('update', subject: 'invoice')]
public function update(Invoice $invoice)
{
}
```

Pipeline:

```text
Route Binding
     ↓
Invoice#928
     ↓
InvoicePolicy::update()
```

---

# 43. Controller-Level Authorization

Una clase podrá declarar requisitos globales.

```php
#[Authorize('access-admin')]
final class AdminController
{
}
```

La metadata será heredada por las acciones según las reglas que se definan.

---

# 44. Action-Level Authorization

Una acción podrá añadir reglas.

```php
#[Authorize('delete', subject: 'user')]
public function destroy(User $user)
{
}
```

Resultado:

```text
Controller Authorization
           +
Action Authorization
           ↓
Authorization Plan
```

---

# 45. Composición Controller + Action

Ejemplo:

```php
#[Authorize('access-admin')]
final class UserController
{
    #[Authorize('delete', subject: 'user')]
    public function destroy(User $user)
    {
    }
}
```

Plan:

```text
1. access-admin
2. resolve user
3. delete User#42
4. execute controller
```

Esto permitirá aplicar seguridad general y específica sin duplicación.

---

# 46. Integración con Routing

Routing podrá proporcionar metadata:

```php
Route::delete('/users/{user}', ...)
    ->can('delete', 'user');
```

Routing no decidirá la autorización.

Solo declarará:

```text
AuthorizationMetadata
```

que será consumida por Authorization.

---

# 47. Integración con Controller System

El Controller System deberá exponer puntos de integración claramente definidos.

```text
Controller Resolution
       ↓
Controller Metadata
       ↓
Pre-Authorization
       ↓
Argument Resolution
       ↓
Resource Authorization
       ↓
Controller Invocation
```

Authorization no deberá controlar directamente el lifecycle completo del Controller.

Cada subsistema conservará su responsabilidad.

---

# 48. Integración con Authentication

Dependencia:

```text
Authorization
      ↓
PrincipalResolver
      ↓
AuthenticationContext
```

Authorization podrá preguntar:

```text
Who is the current Principal?
```

pero no deberá:

```text
authenticate credentials
validate passwords
create sessions
refresh tokens
```

---

# 49. Integración Multi-Tenant

El `TenantContextProvider` podrá incorporar el tenant activo.

```text
AuthorizationContext
      │
      └── TenantContext
```

Las reglas de aislamiento podrán ejecutarse como Policies globales.

```text
Request
   ↓
TenantIsolationPolicy
   ↓
ResourcePolicy
```

No obstante, Authorization no deberá convertirse en el único mecanismo de aislamiento de datos.

La protección multi-tenant deberá existir también en las capas apropiadas del Database System.

---

# 50. Integración con Components

Los Components deberán delegar al AuthorizationManager.

```php
$this->can('update', $post);
```

Internamente:

```text
Component
    ↓
AuthorizationManager
```

No:

```text
Component
    ↓
Custom Component Authorization Engine
```

---

# 51. Integración con Directives

El futuro sistema de directivas podrá implementar:

```php
@can('update', $post)
    ...
@endcan
```

como una consulta al Authorization System.

El compilador podrá optimizar la invocación, pero no alterar la semántica de seguridad.

---

# 52. Integración con SPA Runtime

El frontend podrá recibir capacidades calculadas.

```text
Authorization System
        ↓
Capability Projection
        ↓
SPA Payload
```

Ejemplo:

```json
{
    "abilities": {
        "invoice.update": true,
        "invoice.delete": false
    }
}
```

Estas capacidades son:

```text
UI hints
```

y nunca:

```text
security authority
```

---

# 53. Commands

Commands podrán utilizar el mismo motor.

```php
Authorization::authorize(
    'execute',
    ImportCustomersCommand::class
);
```

También podrán existir atributos:

```php
#[Authorize('customers.import')]
final class ImportCustomersCommand
{
}
```

---

# 54. Jobs

Jobs que ejecuten operaciones privilegiadas podrán utilizar:

```text
SystemPrincipal
ServicePrincipal
UserPrincipal
```

según el contexto de origen.

Deberá evitarse transportar objetos completos de autenticación innecesariamente dentro de queues.

La estrategia exacta de propagación de identidad será definida en una especificación posterior.

---

# 55. Events

El Authorization System podrá emitir eventos como:

```text
AuthorizationRequested
AuthorizationGranted
AuthorizationDenied
PolicyEvaluated
AuthorizationFailed
```

Los eventos no deberán formar parte obligatoria del hot path si no existen listeners.

---

# 56. Observability Architecture

La observabilidad será opcional y de bajo overhead.

```text
AuthorizationManager
       ↓
TraceCollector
       ↓
AuthorizationTrace
```

Podrá registrar:

```text
request
principal descriptor
ability
subject descriptor
resolved policies
individual decisions
strategy
final decision
duration
```

---

# 57. AuthorizationTrace

Ejemplo conceptual:

```text
AuthorizationTrace #A91

ability:
invoice.approve

principal:
User#42

subject:
Invoice#928

evaluations:

TenantIsolationPolicy
  GRANT
  0.08 ms

InvoicePolicy
  GRANT
  0.14 ms

CompliancePolicy
  DENY
  0.22 ms

strategy:
Unanimous

final:
DENY
```

---

# 58. Auditoría

Tracing y auditing serán conceptos diferentes.

```text
Trace
 ↓
Developer / Diagnostics

Audit
 ↓
Security / Compliance / Historical Record
```

No todas las decisiones deberán persistirse como auditoría.

El `AuthorizationAuditor` podrá aplicar reglas específicas.

---

# 59. Compilation Architecture

En producción:

```text
Source Code
    ↓
Policy Discovery
    ↓
Attribute Discovery
    ↓
Metadata Normalization
    ↓
Authorization Compiler
    ↓
Compiled Authorization Metadata
```

El runtime deberá consumir preferentemente esta representación compilada.

---

# 60. Metadata compilable

Podrán compilarse:

```text
Policy mappings
Controller attributes
Action attributes
Route authorization metadata
Policy methods
Ability mappings
Decision strategies
Priorities
```

No deberán compilarse decisiones dependientes del usuario.

---

# 61. Cache Architecture

Debe distinguirse entre:

```text
Metadata Cache
```

y:

```text
Decision Cache
```

La metadata es generalmente segura para reutilizar.

Las decisiones requieren muchísimo más cuidado.

---

# 62. Metadata Cache

Ejemplo:

```text
Invoice
  → InvoicePolicy

InvoicePolicy
  update → method metadata
  delete → method metadata
```

Esta información puede mantenerse incluso entre requests si es inmutable.

---

# 63. Decision Cache

No se habilitará indiscriminadamente.

Una decisión puede depender de:

```text
Principal
Tenant
Subject
Database State
Time
Request
IP
Authentication State
External Policy
```

Por ello:

```text
Decision Cache != ordinary cache
```

Cualquier mecanismo de cache de decisiones deberá tener reglas estrictas.

---

# 64. Request-Scoped Memoization

Sí será posible optimizar evaluaciones repetidas dentro de una misma ejecución.

Ejemplo:

```php
$user->can('update', $invoice);
$user->can('update', $invoice);
$user->can('update', $invoice);
```

podrá utilizar una memoización request-scoped cuando la Policy esté marcada como segura para ello.

---

# 65. Persistent Runtime Architecture

Los componentes se clasificarán conceptualmente como:

```text
Runtime-Safe Shared State
Request-Scoped State
```

## Shared

```text
Compiled Metadata
PolicyRegistry
Immutable Configuration
Compiled Plans
```

## Request-Scoped

```text
Principal
AuthorizationContext
Tenant
Trace
Decision Memoization
```

Esta separación será obligatoria para FrankenPHP.

---

# 66. Dependency Injection

Policies podrán utilizar constructor injection:

```php
final class InvoicePolicy
{
    public function __construct(
        private FraudService $fraud,
    ) {}
}
```

El Container resolverá la Policy.

Sin embargo, el sistema deberá conocer el lifecycle apropiado para evitar state leakage.

Las Policies deberán ser stateless por defecto.

---

# 67. Policy State

Se recomienda:

```text
Policy = Stateless
```

Una Policy no deberá almacenar:

```text
Current User
Current Request
Current Tenant
Current Subject
```

como estado mutable interno.

Estos valores deberán recibirse mediante:

```text
AuthorizationRequest
AuthorizationContext
method parameters
```

---

# 68. Exceptions

Se establecerá una jerarquía propia.

Ejemplo conceptual:

```text
AuthorizationException
│
├── AuthorizationDeniedException
├── PolicyNotFoundException
├── InvalidPolicyException
├── InvalidDecisionException
├── PrincipalResolutionException
├── SubjectResolutionException
└── AuthorizationConfigurationException
```

No todas las excepciones deberán exponerse al cliente.

---

# 69. Error Boundary

El AuthorizationManager actuará como frontera para errores del motor.

```text
Policy
  ↓
Exception
  ↓
AuthorizationManager
  ↓
Fail-Closed Handling
  ↓
DENY / Framework Exception
```

El comportamiento podrá variar entre:

```text
development
production
```

sin modificar el resultado seguro.

---

# 70. Security Boundary

Authorization deberá considerarse una frontera de seguridad backend.

Por tanto:

```text
Frontend can()
```

no reemplaza:

```text
Backend authorize()
```

y:

```text
Hidden button
```

no equivale a:

```text
Protected operation
```

---

# 71. No Business Logic Duplication

Las Policies deberán decidir:

```text
whether an action is allowed
```

No deberán convertirse en servicios de dominio.

Ejemplo incorrecto:

```text
InvoicePolicy
  ↓
calculate invoice
charge payment
send email
update inventory
```

Ejemplo correcto:

```text
InvoicePolicy
  ↓
Can this Principal approve this Invoice?
```

---

# 72. Policy vs Validation

Authorization tampoco reemplaza validación.

```text
Authorization:
Can the user change the price?

Validation:
Is the supplied price valid?
```

Ambas operaciones pueden participar en el mismo request, pero son responsabilidades distintas.

---

# 73. Policy vs Domain Invariants

Una regla como:

```text
Only managers can approve invoices
```

es candidata a autorización.

Una regla como:

```text
A cancelled invoice cannot be paid
```

puede ser una invariancia del dominio.

VoltStack deberá evitar trasladar automáticamente todas las reglas empresariales a Policies.

---

# 74. Extensibility Points

Los principales puntos de extensión serán:

```text
PrincipalResolver
SubjectResolver
ContextProvider
GateResolver
PolicyResolver
PolicyDispatcher
AuthorizationPlanner
DecisionStrategy
TraceCollector
AuthorizationAuditor
MetadataLoader
AuthorizationCompiler
```

Las extensiones deberán utilizar contratos públicos estables.

---

# 75. Arquitectura de contratos

Estructura conceptual:

```text
Contracts/
├── AuthorizationManagerInterface.php
├── PrincipalInterface.php
├── PrincipalResolverInterface.php
├── SubjectResolverInterface.php
├── ContextProviderInterface.php
├── PolicyResolverInterface.php
├── PolicyDispatcherInterface.php
├── AuthorizationPlannerInterface.php
├── DecisionManagerInterface.php
├── DecisionStrategyInterface.php
├── TraceCollectorInterface.php
└── AuthorizationAuditorInterface.php
```

No todos deberán necesariamente exponerse públicamente en V1.

---

# 76. Estructura inicial del paquete

```text
src/
└── Quantum/
    └── Authorization/
        ├── Contracts/
        ├── Core/
        │   ├── AuthorizationManager.php
        │   ├── AuthorizationRequest.php
        │   └── Ability.php
        │
        ├── Principal/
        │   ├── PrincipalResolver.php
        │   └── AnonymousPrincipal.php
        │
        ├── Subject/
        │   ├── SubjectResolver.php
        │   └── SubjectDescriptor.php
        │
        ├── Context/
        │   ├── AuthorizationContext.php
        │   └── Providers/
        │
        ├── Gates/
        │   ├── GateManager.php
        │   ├── GateRegistry.php
        │   └── GateResolver.php
        │
        ├── Policies/
        │   ├── PolicyManager.php
        │   ├── PolicyRegistry.php
        │   ├── PolicyResolver.php
        │   ├── PolicyDispatcher.php
        │   └── Discovery/
        │
        ├── Planning/
        │   ├── AuthorizationPlanner.php
        │   └── AuthorizationPlan.php
        │
        ├── Pipeline/
        │   └── PolicyPipeline.php
        │
        ├── Decisions/
        │   ├── Decision.php
        │   ├── DecisionResult.php
        │   ├── DecisionManager.php
        │   └── Strategies/
        │
        ├── Attributes/
        │   ├── Authorize.php
        │   ├── Policy.php
        │   ├── RequiresRole.php
        │   └── RequiresPermission.php
        │
        ├── Metadata/
        │   ├── AuthorizationMetadata.php
        │   └── MetadataRegistry.php
        │
        ├── Compilation/
        │   └── AuthorizationCompiler.php
        │
        ├── Cache/
        │   └── AuthorizationMetadataCache.php
        │
        ├── Integration/
        │   ├── Http/
        │   ├── Routing/
        │   ├── Controllers/
        │   ├── Components/
        │   ├── Commands/
        │   └── Jobs/
        │
        ├── Observability/
        │   ├── AuthorizationTrace.php
        │   ├── TraceCollector.php
        │   └── AuthorizationProfiler.php
        │
        ├── Audit/
        │   └── AuthorizationAuditor.php
        │
        ├── Events/
        ├── Exceptions/
        └── Support/
```

Esta estructura es conceptual y podrá refinarse conforme se documenten los subsistemas.

---

# 77. Flujo completo de autorización programática

Para:

```php
$user->can('update', $invoice);
```

el flujo conceptual será:

```text
User::can()
    ↓
AuthorizationManager::check()
    ↓
Normalize Ability
    ↓
Resolve Principal
    ↓
Describe Subject
    ↓
Build AuthorizationContext
    ↓
Create AuthorizationRequest
    ↓
AuthorizationPlanner
    ↓
Resolve Global Policies
    ↓
Resolve Subject Policy
    ↓
Build AuthorizationPlan
    ↓
Execute PolicyPipeline
    ↓
Normalize individual results
    ↓
DecisionManager
    ↓
DecisionResult
    ↓
bool
```

---

# 78. Flujo completo mediante Controller

Para:

```php
#[Authorize('update', subject: 'invoice')]
public function update(Invoice $invoice)
{
}
```

el flujo será:

```text
HTTP Request
     ↓
Route Matcher
     ↓
Controller Resolver
     ↓
Authorization Metadata Lookup
     ↓
Pre-Resolution Authorization
     ↓
Argument Resolver
     ↓
Route / Entity Binding
     ↓
Invoice#928
     ↓
Resource Authorization
     ↓
AuthorizationManager
     ↓
InvoicePolicy::update()
     ↓
DecisionManager
     ↓
GRANT
     ↓
Controller Invocation
```

Ante:

```text
DENY
```

el flujo termina antes de ejecutar el controlador.

---

# 79. Flujo con múltiples Policies

```text
AuthorizationRequest
        ↓
AuthorizationPlanner
        ↓
┌────────────────────────────┐
│ SuspendedPrincipalPolicy   │
│ TenantIsolationPolicy      │
│ MfaPolicy                  │
│ InvoicePolicy              │
│ CompliancePolicy           │
└────────────────────────────┘
        ↓
PolicyPipeline
        ↓
┌────────────────────────────┐
│ GRANT                      │
│ GRANT                      │
│ ABSTAIN                    │
│ GRANT                      │
│ DENY                       │
└────────────────────────────┘
        ↓
DecisionManager
        ↓
UnanimousStrategy
        ↓
DENY
```

---

# 80. Dependencias permitidas

Authorization podrá consumir contratos de:

```text
Container
Config
Authentication
Tenant Context
Events
Cache
Observability
```

Las integraciones HTTP podrán consumir además:

```text
HTTP
Routing
Controllers
```

El Core de Authorization no deberá depender directamente de estos últimos.

---

# 81. Dependencias prohibidas del Core

El núcleo no deberá depender obligatoriamente de:

```text
ORM
Database Driver
HTTP Request Implementation
Controller Implementation
Frontend Runtime
SPA Runtime
Specific Cache Driver
Specific Authentication Provider
```

Esto mantiene el sistema reutilizable.

---

# 82. Dependency Direction

La regla general será:

```text
Framework Integrations
        ↓
Authorization Contracts
        ↓
Authorization Core
```

No:

```text
Authorization Core
        ↓
Controllers
        ↓
Routing
        ↓
HTTP
```

Las integraciones deben apuntar hacia el Core, no al contrario.

---

# 83. Invariantes arquitectónicas

El sistema deberá preservar permanentemente las siguientes invariantes.

### Invariante 1

Toda decisión pasa por el Authorization Engine.

### Invariante 2

Una Policy no autentica usuarios.

### Invariante 3

Authorization no depende obligatoriamente del ORM.

### Invariante 4

Frontend authorization nunca constituye una frontera de seguridad.

### Invariante 5

Metadata compartida debe ser inmutable.

### Invariante 6

Estado relacionado con Principal/Tenant/Request debe ser request-scoped.

### Invariante 7

Una ausencia de autorización explícita en operaciones protegidas produce `DENY`.

### Invariante 8

Policies deben ser stateless por defecto.

### Invariante 9

La API declarativa y la programática utilizan el mismo motor.

### Invariante 10

La observabilidad no debe modificar la decisión.

---

# 84. Filosofía arquitectónica

VoltStack deberá ofrecer dos niveles claramente separados.

Para el desarrollador:

```text
$user->can('update', $post);
```

Para el framework:

```text
AuthorizationRequest
        ↓
Principal Resolution
        ↓
Subject Resolution
        ↓
Context Construction
        ↓
Authorization Planning
        ↓
Policy Resolution
        ↓
Policy Pipeline
        ↓
Decision Aggregation
        ↓
DecisionResult
        ↓
Trace / Audit
```

Esta separación permitirá mantener simultáneamente:

```text
Simple Developer Experience
+
Advanced Authorization Architecture
```

---

# 85. Resultado arquitectónico esperado

El Authorization System deberá comportarse como un **motor transversal de decisiones de acceso**, no simplemente como una colección de helpers `can()`.

La arquitectura resultante permitirá evolucionar desde:

```php
$user->can('edit', $post);
```

hasta escenarios como:

```text
User
  +
Tenant
  +
Role
  +
Permission
  +
Resource Ownership
  +
Request Context
  +
Security State
  +
Compliance Rules
        ↓
Authorization Engine
        ↓
Explainable Decision
```

sin sustituir el motor ni romper la API pública.

La arquitectura queda definida alrededor de cinco conceptos fundamentales:

```text
AuthorizationRequest
        ↓
AuthorizationPlan
        ↓
PolicyPipeline
        ↓
DecisionManager
        ↓
DecisionResult
```

Estos componentes constituirán el núcleo sobre el cual se desarrollarán las siguientes especificaciones del Authorization System de VoltStack.