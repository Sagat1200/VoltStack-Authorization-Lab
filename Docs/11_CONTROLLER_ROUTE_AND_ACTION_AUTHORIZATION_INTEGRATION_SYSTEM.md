# VoltStack Authorization System — Controller, Route and Action Authorization Integration System

## 1. Propósito

Este documento define la integración del Authorization System de VoltStack con:

```text
Routing
Controllers
Controller Actions
Action Classes
Argument Resolution
Route Binding
HTTP Kernel
Middleware Pipeline
Response Handling
```

El objetivo es garantizar que toda autorización declarada o programática sobre una ruta, un Controller o una acción se ejecute:

```text
en el momento correcto
con el Subject correcto
con el Principal correcto
sin duplicar motores
sin resolver recursos innecesariamente
sin exponer información sensible
```

La integración deberá soportar:

```php
#[Authorize('admin.access')]
final class AdminController
{
    #[Authorize('update', subject: 'invoice')]
    public function update(Invoice $invoice): Response
    {
    }
}
```

así como:

```php
Route::put('/invoices/{invoice}', [InvoiceController::class, 'update'])
    ->authorize('update', 'invoice');
```

Ambas formas deberán converger en:

```text
Compiled Authorization Metadata
        ↓
Authorization Integration Layer
        ↓
AuthorizationManager
        ↓
AuthorizationPlanner
        ↓
AuthorizationPlan
        ↓
DecisionManager
```

---

# 2. Principio arquitectónico

Routing y Controllers no deberán implementar autorización por sí mismos.

Su responsabilidad será:

```text
exponer metadata
resolver lifecycle
proporcionar Subjects
invocar AuthorizationManager
```

La autoridad final será siempre:

```text
Authorization Core Engine
```

Por tanto:

```text
Route authorization
Controller authorization
Action authorization
```

son integraciones del mismo motor.

---

# 3. Objetivos principales

La integración deberá proporcionar:

- autorización a nivel de ruta;
- autorización a nivel de Controller;
- autorización a nivel de método;
- autorización de Action Classes;
- pre-resolution authorization;
- resource authorization;
- integración con route binding;
- integración con Controller Argument Resolver;
- metadata compilada;
- herencia de requisitos;
- composición Route + Controller + Method;
- soporte para Policies sobre Controllers;
- soporte para Policies sobre recursos;
- soporte para Gates;
- integración con HTTP exceptions;
- soporte para `403` y estrategias de ocultamiento mediante `404`;
- ejecución eficiente bajo FrankenPHP;
- tracing completo del lifecycle.

---

# 4. Pipeline HTTP general

El pipeline conceptual será:

```text
HTTP Request
    ↓
Kernel
    ↓
Global Middleware
    ↓
Route Matching
    ↓
Route Metadata Resolution
    ↓
Controller Target Resolution
    ↓
Authorization Metadata Resolution
    ↓
PRE-RESOLUTION AUTHORIZATION
    ↓
Route / Argument Binding
    ↓
Controller Argument Resolution
    ↓
RESOURCE AUTHORIZATION
    ↓
Controller / Action Invocation
    ↓
POST-RESOLUTION AUTHORIZATION (if configured)
    ↓
Response Transformation
```

---

# 5. Ubicación de Authorization en el lifecycle

Authorization deberá ejecutarse tan pronto como exista suficiente información para tomar la decisión.

Regla:

```text
Do not resolve more than authorization needs.
```

Por ello existirán varias fases.

---

# 6. Pre-Resolution Authorization

Se ejecuta antes de resolver Subjects costosos.

Ejemplo:

```php
#[Authorize(
    'admin.access',
    phase: AuthorizationPhase::PreResolution
)]
final class AdminController
{
}
```

Pipeline:

```text
Route matched
    ↓
Controller known
    ↓
admin.access
    ↓
Authorization
    ↓
DENY
```

En ese caso no se ejecuta:

```text
Model Binding
Controller Argument Resolution
Database Lookup
Controller Construction (when avoidable)
```

---

# 7. Resource Authorization

Se ejecuta una vez que el Subject real está disponible.

Ejemplo:

```php
#[Authorize(
    'update',
    subject: 'invoice'
)]
public function update(Invoice $invoice)
{
}
```

Pipeline:

```text
Route
 ↓
{invoice}
 ↓
Binding
 ↓
Invoice#928
 ↓
AuthorizationManager
 ↓
InvoicePolicy::update()
```

---

# 8. Post-Resolution Authorization

Podrá existir para escenarios en que una decisión dependa de información creada después del binding o enriquecimiento contextual.

Ejemplo:

```text
Resolved resource
+
resolved workflow context
+
security context
```

No deberá utilizarse como sustituto arbitrario de Resource Authorization.

---

# 9. Fases y orden

Orden recomendado:

```text
PRE_RESOLUTION
        ↓
RESOURCE
        ↓
POST_RESOLUTION
```

Una fase denegada termina el lifecycle de autorización de esa operación.

---

# 10. Controller Authorization Metadata

La integración deberá consultar metadata compilada asociada a:

```text
Controller class
Controller method
Route
Inherited Controller class
Action class
```

No realizar Reflection repetitiva durante requests productivas.

---

# 11. ControllerMetadata ID

Ejemplo:

```text
controller:App\Controller\InvoiceController
```

---

# 12. Controller Method Metadata ID

Ejemplo:

```text
controller_method:
App\Controller\InvoiceController::update
```

---

# 13. Route Metadata ID

Ejemplo:

```text
route:invoice.update
```

---

# 14. ControllerAuthorizationMetadataResolver

Contrato conceptual:

```php
interface ControllerAuthorizationMetadataResolverInterface
{
    public function resolve(
        ControllerDescriptor $controller,
        RouteDescriptor $route,
    ): EffectiveAuthorizationMetadata;
}
```

---

# 15. EffectiveAuthorizationMetadata

Representará la composición final de:

```text
route metadata
+
inherited controller metadata
+
controller class metadata
+
controller method metadata
```

---

# 16. Composición por defecto

La composición será:

```text
additive
```

Ejemplo:

```php
#[Authorize('admin.access')]
final class InvoiceController
{
    #[Authorize('update', subject: 'invoice')]
    public function update(Invoice $invoice)
    {
    }
}
```

Resultado:

```text
admin.access
AND
invoice.update
```

---

# 17. Route + Controller composition

Ejemplo:

```php
Route::put('/invoices/{invoice}', ...)
    ->authorize('api.internal');
```

Controller:

```php
#[Authorize('update', subject: 'invoice')]
```

Resultado:

```text
api.internal
AND
invoice.update
```

---

# 18. Route authorization DSL

VoltStack Routing podrá proporcionar:

```php
Route::get('/admin', AdminController::class)
    ->authorize('admin.access');
```

---

# 19. Resource route authorization

```php
Route::put('/invoices/{invoice}', ...)
    ->authorize(
        'update',
        subject: 'invoice'
    );
```

---

# 20. Multiple route requirements

```php
Route::delete('/users/{user}', ...)
    ->authorize('admin.access')
    ->authorize('delete', 'user');
```

---

# 21. Route DSL normalization

La Route DSL deberá producir exactamente el mismo:

```text
AuthorizationRequirementDescriptor
```

que los PHP Attributes.

---

# 22. No separate route semantics

No deberá existir:

```text
RouteAuthorizationResult
```

con reglas distintas.

Todo debe pasar por:

```text
AuthorizationManager
```

---

# 23. Route-level PreResolution

Un requirement sin resource podrá ejecutarse inmediatamente después del match.

Ejemplo:

```text
admin.access
```

---

# 24. Route-level Resource Authorization

Un requirement con:

```text
subject = route parameter
```

se ejecutará después de binding.

---

# 25. Route Parameter Subject

Ejemplo:

```php
Route::get('/projects/{project}', ...)
    ->authorize('view', 'project');
```

Compilación:

```text
SubjectReference:
RouteParameter(project)
```

---

# 26. Route Binding

Routing sigue siendo responsable de convertir:

```text
project = "81"
```

en:

```text
Project#81
```

Authorization no hará esa conversión por sí mismo.

---

# 27. Binding failure

Si el recurso no existe:

```text
Route Binding
    ↓
Not Found
```

Authorization del resource no se ejecuta porque no existe Subject válido.

---

# 28. 404 vs 403 order

Esto introduce una consideración importante:

```text
¿debe saberse primero si existe el resource
o si el usuario puede acceder a él?
```

VoltStack deberá permitir políticas seguras según tipo de aplicación.

---

# 29. Default resource lifecycle

Recomendación inicial:

```text
PreAuthorization
    ↓
Resource Binding
    ↓
Resource Authorization
```

Si el resource no existe:

```text
404
```

Si existe pero el Principal no puede acceder:

```text
403
```

por defecto.

---

# 30. Resource existence concealment

Algunas aplicaciones necesitan ocultar la existencia del recurso.

Ejemplo:

```text
GET /private-documents/928
```

No se desea revelar si `928` existe.

VoltStack deberá soportar:

```text
Authorization Denial
        ↓
404 Not Found
```

cuando la metadata lo indique.

---

# 31. Concealment Policy

Podrá existir:

```text
AuthorizationDenialVisibility
```

con opciones:

```text
Forbidden
NotFound
Custom
```

---

# 32. DenialVisibility enum

Conceptualmente:

```php
enum AuthorizationDenialVisibility: string
{
    case Forbidden = 'forbidden';
    case NotFound = 'not_found';
}
```

---

# 33. Declarative usage

Futuro:

```php
#[Authorize(
    'view',
    subject: 'document',
    denial: AuthorizationDenialVisibility::NotFound
)]
```

---

# 34. Importante

La selección entre `403` y `404` pertenece a:

```text
HTTP Integration
```

no al `DecisionManager`.

El Core simplemente produce:

```text
DENY
```

---

# 35. AuthorizationDeniedException

El Core podrá lanzar:

```text
AuthorizationDeniedException
```

con:

```text
DecisionResult
```

La capa HTTP decidirá la representación.

---

# 36. HTTPAuthorizationExceptionMapper

Contrato conceptual:

```php
interface HttpAuthorizationExceptionMapperInterface
{
    public function map(
        AuthorizationDeniedException $exception,
        AuthorizationHttpContext $context,
    ): Response;
}
```

---

# 37. Default mapping

```text
DENY
 ↓
403 Forbidden
```

---

# 38. Concealed mapping

```text
DENY
+
NotFound visibility
 ↓
404 Not Found
```

---

# 39. No reason leakage

El HTTP response no deberá exponer automáticamente:

```text
tenant.mismatch
missing role
missing internal permission
policy class
```

---

# 40. Error representation

API response conceptual:

```json
{
    "error": {
        "code": "forbidden",
        "message": "You are not allowed to perform this action."
    }
}
```

---

# 41. Development mode

Podrá existir información adicional en profiler/debug tools, no necesariamente en la respuesta.

---

# 42. Controller Policy

Un Controller puede ser el Subject.

Ejemplo:

```php
#[Authorize(
    'access',
    subject: AdminController::class
)]
final class AdminController
{
}
```

---

# 43. Simplified Controller syntax

Si la metadata está aplicada sobre la clase:

```php
#[Authorize('access')]
final class AdminController
{
}
```

VoltStack podrá inferir:

```text
Subject:
AdminController::class
```

cuando `access` sea una Controller-level Ability.

---

# 44. Controller-specific Policy

Ejemplo:

```php
#[PolicyFor(AdminController::class)]
final class AdminControllerPolicy
{
    public function access(
        User $user
    ): bool {
        return $user->isAdmin();
    }
}
```

---

# 45. Controller Policy Resolution

Pipeline:

```text
AdminController class
    ↓
SubjectDescriptor(Class)
    ↓
PolicyResolver
    ↓
AdminControllerPolicy
```

---

# 46. Controller Action as Subject

También podrá modelarse:

```text
controller_action:
InvoiceController::approve
```

---

# 47. Action-specific Policy

Ejemplo:

```php
#[PolicyForControllerAction(
    InvoiceController::class,
    'approve'
)]
final class ApproveInvoiceActionPolicy
{
}
```

---

# 48. Recomendación

Utilizar Controller Action Policies solo cuando la regla pertenece realmente a la operación de Controller.

Si la regla pertenece al Invoice:

```text
InvoicePolicy
```

es preferible.

---

# 49. Controller Instantiation

VoltStack deberá decidir cuándo construir el Controller.

Idealmente:

```text
PreResolution Authorization
```

podrá ejecutarse antes de instanciar el Controller si toda metadata está compilada.

---

# 50. Beneficio

Esto evita constructor DI costoso para requests que serán rechazadas.

---

# 51. Controller instance Policy

Si una Policy realmente requiere la instancia del Controller:

```text
Controller must be instantiated first.
```

Esta capacidad deberá ser explícita y desaconsejada para reglas comunes.

---

# 52. Preferred Controller Subject

Preferido:

```text
Controller class
```

sobre:

```text
Controller instance
```

para pre-autorización.

---

# 53. Argument Resolution

Después de aprobar PreResolution:

```text
ControllerArgumentResolver
```

construye parámetros.

Ejemplo:

```php
public function update(
    Invoice $invoice,
    UpdateInvoiceRequest $request
)
```

---

# 54. Subject dependency

Solo algunos argumentos serán Subjects de autorización.

Aquí:

```text
invoice
```

---

# 55. Non-subject arguments

`UpdateInvoiceRequest` podrá resolverse antes o después de Resource Authorization dependiendo de costo y seguridad.

---

# 56. Argument Resolution Optimization

La integración podrá conocer:

```text
which arguments are required for authorization
```

y resolverlos primero.

---

# 57. Authorization-aware Argument Resolution

Ejemplo:

```text
invoice          needed for authorization
validated DTO    not yet needed
service          only for controller invocation
```

Pipeline optimizado:

```text
Resolve invoice
    ↓
Authorize
    ↓
Resolve remaining arguments
```

---

# 58. Benefit

Evita trabajo innecesario cuando la Policy deniega.

---

# 59. ControllerArgumentDependencyPlan

El Controller compiler podrá producir:

```text
Authorization-required arguments
Invocation-only arguments
```

---

# 60. Example

Método:

```php
public function update(
    Invoice $invoice,
    UpdateInvoiceData $data,
    CurrencyService $currency,
): Response
```

Authorization:

```text
subject=invoice
```

VoltStack podría:

```text
1. resolve Invoice
2. authorize
3. resolve UpdateInvoiceData
4. resolve CurrencyService
5. invoke
```

---

# 61. Validation ordering

La relación entre Validation y Authorization debe ser configurable/semántica.

No siempre se desea validar un payload completo antes de saber si el usuario puede modificar el resource.

---

# 62. Recommended order

Para resource mutation:

```text
bind resource
    ↓
authorize
    ↓
validate mutation input
```

cuando sea seguro.

---

# 63. Why

Evita revelar validaciones o gastar recursos a usuarios no autorizados.

---

# 64. Exceptions

Hay casos donde parte de la validación es necesaria para identificar correctamente el Subject.

La integración deberá permitir dependencies explícitas.

---

# 65. Authorization Dependencies

Un requirement podrá declarar qué argumentos/contexto necesita.

El compiler podrá construir un dependency graph.

---

# 66. No circular argument dependencies

Ejemplo inválido:

```text
Argument A resolution
needs authorization

Authorization
needs Argument A
```

sin fase intermedia resoluble.

Debe detectarse.

---

# 67. Model Binding Security

Binding de un resource multi-tenant deberá idealmente respetar scope tenant desde la capa DB/Routing.

Authorization sigue siendo defensa adicional.

---

# 68. Defense in Depth

Ejemplo:

```text
Route Model Binding
   ↓ tenant scoped query

Authorization
   ↓ TenantIsolationPolicy
```

Ambas capas son útiles.

---

# 69. Authorization no sustituye scoped binding

Si el binding trae datos de otro tenant y luego Policy los rechaza, el sistema sigue teniendo mayor superficie de riesgo.

---

# 70. Scoped Binding Recommendation

Para multi-tenancy:

```text
Tenant Context
    ↓
Scoped Resource Resolution
    ↓
Authorization
```

---

# 71. Controller-level requirements

Ejemplo:

```php
#[RequiresRole('administrator')]
final class AdminController
{
}
```

se aplica a todas las acciones.

---

# 72. Method additions

```php
#[RequiresPermission('user.delete')]
public function destroy(User $user)
{
}
```

Resultado:

```text
Role administrator
AND
Permission user.delete
```

---

# 73. Inherited Controllers

Base:

```php
#[Authorize('admin.access')]
abstract class BaseAdminController
{
}
```

Child:

```php
final class UserAdminController
    extends BaseAdminController
{
}
```

La metadata heredada deberá compilarse.

---

# 74. Method inheritance

Si un método sobrescribe otro:

```text
parent method metadata
```

deberá combinarse según reglas del Metadata System.

---

# 75. Recommended semantics

Por defecto:

```text
parent class requirements remain
```

pero method-level requirements se calculan sobre el método efectivo.

---

# 76. Explicit override

Eliminar una autorización heredada deberá requerir metadata explícita.

---

# 77. NonBypassable

Requirements críticos nunca podrán eliminarse mediante un Controller hijo.

---

# 78. Route aliases

Si dos rutas apuntan a la misma acción:

```text
same Controller method
```

cada Route puede añadir requisitos propios.

---

# 79. Effective Metadata per Route Invocation

Por tanto el cache no puede asumir siempre:

```text
Controller method = identical requirements
```

si Route añade metadata.

---

# 80. Effective Target Key

Podrá ser:

```text
route ID
+
controller method metadata ID
```

---

# 81. Route-specific PlanTemplate

Podrá compilarse:

```text
route:invoice.update
```

como plan completo.

---

# 82. Route Cache Integration

Cuando Routing se compile:

```text
Route Metadata
+
Authorization Metadata
```

deberán compilarse juntos cuando sea posible.

---

# 83. Benefit

El router puede entregar directamente:

```text
CompiledAuthorizationTargetReference
```

después de match.

---

# 84. Zero Reflection Controller Integration

En producción:

```text
Route Match
 ↓
Compiled Route Descriptor
 ↓
Compiled Controller Descriptor
 ↓
Compiled Authorization Metadata
```

Sin:

```text
ReflectionClass
ReflectionMethod
```

---

# 85. Action Classes

VoltStack podrá soportar:

```php
final class ApproveInvoice
{
    public function __invoke(
        Invoice $invoice
    ): void {
    }
}
```

---

# 86. Action authorization

Ejemplo:

```php
#[Authorize(
    'approve',
    subject: 'invoice'
)]
final class ApproveInvoice
{
}
```

---

# 87. Action Dispatcher

La autorización solo se ejecutará automáticamente cuando la Action sea invocada mediante:

```text
ActionDispatcher
```

---

# 88. Direct invocation

Esto:

```php
$action($invoice);
```

no debe suponerse interceptado mágicamente salvo que `$action` sea proxy administrado por framework.

---

# 89. Avoid hidden AOP

VoltStack deberá evitar depender de interceptación transparente difícil de depurar.

Preferido:

```text
explicit framework dispatch boundaries
```

---

# 90. Controller-to-Action flow

Ejemplo:

```text
Controller Authorization
        ↓
Action Authorization
        ↓
Action Execution
```

si ambos poseen metadata.

---

# 91. Duplicate Authorization

Debe evitarse evaluar dos veces exactamente la misma decisión si:

```text
Controller
```

y:

```text
Action
```

declaran el mismo requirement.

---

# 92. Deduplication scope

La deduplicación deberá considerar:

```text
Principal
Ability
Subject
Context-relevant fingerprint
```

y reglas de memoization.

---

# 93. Do not deduplicate semantically different requirements

Aunque ambos digan:

```text
approve
```

pueden usar:

```text
different Subject
different Strategy
different Context
```

y deben ejecutarse por separado.

---

# 94. Controller Authorization Coordinator

Podrá existir:

```php
interface ControllerAuthorizationCoordinatorInterface
{
    public function authorize(
        ControllerInvocationContext $context
    ): void;
}
```

---

# 95. Responsibilities

El Coordinator:

- obtiene metadata efectiva;
- ejecuta PreResolution;
- solicita binding requerido;
- ejecuta Resource phase;
- coordina post-resolution;
- traduce fallos hacia el HTTP integration layer.

---

# 96. No Decision Logic

El Coordinator no decide:

```text
GRANT/DENY aggregation
```

Eso sigue perteneciendo al Core.

---

# 97. RouteAuthorizationCoordinator

Routing podrá tener un adapter equivalente.

Pero ambos deberían compartir infraestructura.

---

# 98. Preferred Integration abstraction

Podría existir:

```text
HttpAuthorizationCoordinator
```

que coordine Route + Controller metadata en un único lifecycle.

---

# 99. HttpAuthorizationContext

Contendrá:

```text
RouteDescriptor
ControllerDescriptor
ControllerMethodDescriptor
HTTP request abstraction
resolved arguments
```

sin convertirse en Core dependency.

---

# 100. HTTP-only types stay outside Core

El Authorization Core no deberá importar:

```text
Route
HttpRequest
ControllerDispatcher
```

---

# 101. Dependency direction

Correcto:

```text
Routing
Controllers
HTTP
   ↓
Authorization Contracts
```

No:

```text
Authorization Core
   ↓
Routing
```

---

# 102. Middleware Authorization

VoltStack podrá seguir permitiendo middleware como:

```text
can:update,invoice
```

por compatibilidad/ergonomía.

---

# 103. Middleware implementation

El middleware deberá ser un adapter sobre:

```text
AuthorizationManager
```

No contener un motor propio.

---

# 104. Middleware vs metadata

Para casos estáticos se recomienda:

```text
compiled authorization metadata
```

sobre middleware string-based.

---

# 105. Middleware use cases

Útil para:

```text
route groups
legacy integrations
runtime-defined routes
package compatibility
```

---

# 106. Route group authorization

Ejemplo:

```php
Route::group(
    authorization: 'admin.access',
    routes: function () {
        // ...
    }
);
```

---

# 107. Group metadata

El Router compiler deberá propagar esa metadata a las rutas hijas.

---

# 108. Nested groups

La composición deberá ser aditiva y determinista.

---

# 109. Group override

Eliminar reglas de un parent group deberá requerir una configuración explícita.

---

# 110. Middleware ordering

Authorization middleware no deberá ejecutarse antes de que exista la información que necesita.

Ejemplo:

```text
Authentication
    ↓
Tenant Resolution
    ↓
Authorization
```

cuando requiere Principal/Tenant.

---

# 111. Global HTTP middleware ordering

Orden conceptual:

```text
Request normalization
Authentication context
Tenant context
Routing
Authorization
Controller execution
```

---

# 112. Authentication before Authorization

Authorization necesita un Principal.

Si no hay autenticación:

```text
AnonymousPrincipal
```

podrá utilizarse.

---

# 113. Tenant before tenant-aware authorization

Si la ruta es tenant-aware:

```text
TenantContext
```

deberá resolverse antes de Policies de tenant.

---

# 114. Route Match before route authorization

Para saber metadata de ruta:

```text
route matching
```

debe haber ocurrido.

---

# 115. Authorization before Controller Invocation

La acción protegida nunca deberá ejecutarse antes de completar sus fases requeridas.

---

# 116. Authorization failure short-circuit

Cuando se produce DENY:

```text
Controller invocation = skipped
```

---

# 117. Response Transformations

No deberán ejecutarse transformaciones dependientes del Controller result porque no existe resultado.

La exception pipeline construirá la respuesta.

---

# 118. Observability

El trace HTTP deberá mostrar:

```text
Route matched
Controller selected
PreAuthorization
Binding
Resource Authorization
Controller skipped/executed
```

---

# 119. Example trace

```text
Route:
invoice.update

Controller:
InvoiceController::update

Authorization:

PRE_RESOLUTION
  admin.access
  → GRANT

RESOURCE BINDING
  invoice → Invoice#928

RESOURCE AUTHORIZATION
  tenant isolation → GRANT
  invoice.update → DENY

Controller Invocation:
SKIPPED
```

---

# 120. Authorization Timing

Podrán registrarse:

```text
pre_authorization_duration
binding_duration
resource_authorization_duration
```

para profiling.

---

# 121. Detect expensive pre-binding

Tooling podrá identificar:

```text
resource resolved before unrelated pre-authorization
```

como oportunidad de optimización.

---

# 122. Authorization plan profiling

Ejemplo:

```text
admin.access          0.10 ms
invoice binding       2.50 ms
InvoicePolicy         0.35 ms
CompliancePolicy      1.20 ms
```

---

# 123. Route Enumeration Security

Con concealment mode, el sistema deberá evitar diferencias excesivas de respuesta que permitan inferir recursos privados cuando sea posible.

---

# 124. Timing side channels

VoltStack no podrá eliminar todos los timing side channels automáticamente, pero deberá evitar APIs que revelen deliberadamente:

```text
resource exists but forbidden
```

cuando concealment está configurado.

---

# 125. Binding strategy for concealed resources

Para ciertos sistemas, podrá ser preferible:

```text
tenant/user-scoped binding
```

de modo que un recurso no autorizado simplemente no sea encontrado.

---

# 126. Authorization remains necessary

Scoped binding no sustituye la Policy.

---

# 127. Soft-deleted resources

Route Binding deberá decidir si son resolubles.

Authorization podrá aplicar abilities como:

```text
restore
forceDelete
```

solo cuando el Subject sea resuelto apropiadamente.

---

# 128. Polymorphic resources

El binding podrá producir subclases.

PolicyResolver trabajará sobre el tipo runtime real.

---

# 129. Parent Policy fallback

La resolución seguirá las reglas del Policy Registry.

---

# 130. Controller Argument Types

La metadata compilada podrá validar:

```text
subject parameter type
```

contra Policy subject types.

---

# 131. Compile-time mismatch example

```php
#[Authorize('update', subject: 'invoice')]
public function update(Order $invoice)
{
}
```

deberá producir diagnóstico.

---

# 132. Nullable Subjects

Si un argumento subject puede ser `null`:

```php
?Invoice $invoice
```

el compiler deberá exigir semántica explícita.

---

# 133. Recommended rule

Un resource authorization Subject requerido no deberá ser nullable por defecto.

---

# 134. Optional Subject Authorization

Podrá existir metadata específica para casos legítimos.

No asumir automáticamente `null = no authorization required`.

---

# 135. Collection Subjects

Ejemplo:

```php
public function export(Collection $invoices)
{
}
```

Si se autoriza la colección, deberá utilizarse una Policy o Batch Authorization apropiada.

---

# 136. Do not authorize only first item

Nunca inferir:

```text
collection authorization
=
authorize first resource
```

---

# 137. Batch authorization

La integración podrá detectar:

```text
BatchAuthorizationRequirement
```

en versiones posteriores.

---

# 138. Controller Constructor Authorization

No se recomienda colocar autorización dentro del constructor.

Incorrecto:

```php
public function __construct()
{
    Authorization::authorize(...);
}
```

---

# 139. Why

El constructor puede ejecutarse:

```text
antes del momento óptimo
en tests
during container resolution
```

y dificulta compilación/lifecycle.

---

# 140. Preferred

Usar:

```php
#[Authorize(...)]
```

en clase/método.

---

# 141. Programmatic Authorization in Controller

Seguirá siendo válido:

```php
public function update(Invoice $invoice)
{
    Authorization::authorize(
        'update',
        $invoice
    );
}
```

---

# 142. Programmatic vs declarative

Ambas usan el mismo manager.

Declarative:

```text
better for static security contracts
```

Programmatic:

```text
better for dynamic branching
```

---

# 143. Example dynamic branching

```php
if ($request->boolean('publish')) {
    Authorization::authorize(
        'publish',
        $article
    );
}
```

Esto no se modela necesariamente bien con metadata estática.

---

# 144. Avoid duplicate checks

Si declarative metadata ya garantiza:

```text
update
```

no repetir la misma llamada manual dentro del método sin razón.

---

# 145. Testing Controller Authorization

Tests deberán poder verificar metadata sin enviar HTTP.

Ejemplo:

```php
$metadata = $authorizationMetadata
    ->forControllerMethod(
        InvoiceController::class,
        'update'
    );
```

---

# 146. HTTP integration test

También:

```text
una request no autorizada
```

debe verificar:

```text
403
controller not invoked
```

---

# 147. Controller invocation spy

Testing package podrá verificar:

```text
controller action was not called
```

tras DENY.

---

# 148. Model binding test

En PreResolution DENY:

```text
model binding should not execute
```

cuando no sea necesario.

---

# 149. Resource authorization test

En PreResolution GRANT + Resource DENY:

```text
resource binding executes
controller does not
```

---

# 150. Concealment test

Cuando `NotFound` esté configurado:

```text
DENY
```

deberá mapearse a:

```text
404
```

---

# 151. Inheritance test

Controller child debe conservar requirements heredados.

---

# 152. Route composition test

Route requirement + Controller requirement deben ambos aparecer en Effective Metadata.

---

# 153. Route cache equivalence

Dynamic route metadata y compiled route cache deberán producir idéntico AuthorizationPlan.

---

# 154. FrankenPHP test

Dos requests consecutivas:

```text
Request A → User A → Invoice A
Request B → User B → Invoice B
```

no deberán compartir:

```text
SubjectReference binding
Decision
Principal
Route authorization state
```

---

# 155. Shared HTTP metadata

Sí podrán compartir:

```text
Compiled Route Descriptor
Compiled Controller Descriptor
Compiled Authorization Metadata
PlanTemplate
```

---

# 156. Request-scoped integration state

Debe ser request-scoped:

```text
resolved route values
resolved Controller arguments
Principal
Tenant
AuthorizationExecution
```

---

# 157. HttpAuthorizationExecutionContext

Podrá existir:

```php
final class HttpAuthorizationExecutionContext
{
    // request-local state
}
```

---

# 158. No singleton current subject

Nunca:

```php
$authorization->currentSubject = $invoice;
```

sobre un servicio shared.

---

# 159. Exceptions

Jerarquía conceptual de integración:

```text
AuthorizationIntegrationException
├── ControllerAuthorizationException
├── RouteAuthorizationException
├── ActionAuthorizationException
├── AuthorizationSubjectBindingException
├── InvalidControllerAuthorizationMetadataException
└── AuthorizationLifecycleException
```

---

# 160. SubjectBindingException

Ocurre cuando metadata requiere:

```text
invoice
```

pero el binding no puede proporcionarlo por una causa distinta de un 404 normal.

---

# 161. Missing route parameter

Ejemplo:

```text
metadata references project
route has no {project}
```

deberá detectarse en compilación.

---

# 162. Stale route metadata

Si cache y Controller signature divergen:

```text
fail closed
```

y registrar cache inconsistency.

---

# 163. Development fallback

En desarrollo podrá sugerirse reconstruir:

```text
route cache
authorization metadata cache
```

---

# 164. HTTP Response responsibility

Authorization Core no conoce:

```text
403
404
JSON
HTML
Redirect
```

---

# 165. Integration responsibility

La capa HTTP transforma:

```text
AuthorizationDeniedException
```

en representación apropiada.

---

# 166. API behavior

Para API:

```text
403 JSON
```

---

# 167. Browser behavior

Para web:

```text
403 page
```

o custom exception renderer.

---

# 168. Authentication distinction

No confundir:

```text
unauthenticated
```

con:

```text
unauthorized
```

---

# 169. Anonymous authorization denial

Si la operación requiere autenticación y Principal es anonymous:

la integración podrá mapear a:

```text
401 Unauthorized
```

cuando el DecisionResult indique explícitamente:

```text
authentication_required
```

---

# 170. Important HTTP terminology

HTTP:

```text
401
=
authentication required/invalid
```

```text
403
=
authenticated or known Principal is forbidden
```

---

# 171. Core still transport-independent

Esta diferenciación se realiza en HTTP mapping.

---

# 172. Decision reason mapping

Ejemplo:

```text
authorization.authentication_required
    ↓
401

authorization.denied
    ↓
403

authorization.concealed
    ↓
404
```

---

# 173. Do not infer 401 from Anonymous alone

Una Policy podría permitir anonymous access.

Solo una decisión que requiera autenticación debe producir 401.

---

# 174. Redirect to login

En aplicaciones browser:

```text
authentication_required
```

podrá convertirse en:

```text
redirect login
```

según configuración.

No es Core behavior.

---

# 175. API no redirect

En API:

```text
401 JSON
```

es preferible.

---

# 176. Content negotiation

HTTP Exception Mapper podrá considerar:

```text
route type
request expected format
API context
```

---

# 177. Controller response transport security

Una vez autorizado, el Controller puede ejecutar normalmente.

Authorization no debe modificar el Response salvo mediante exception handling.

---

# 178. Action-level composition example

```php
#[Authorize('admin.access')]
final class UserController
{
    #[RequiresPermission('users.delete')]
    #[Authorize('delete', subject: 'user')]
    public function destroy(User $user): Response
    {
    }
}
```

---

# 179. Effective PreResolution

```text
admin.access
```

---

# 180. Effective Resource

```text
permission users.delete
AND
UserPolicy::delete(User#...)
```

más evaluadores globales.

---

# 181. Final outcome

Solo si:

```text
PreResolution GRANT
AND
Resource GRANT
```

se invoca:

```text
destroy()
```

---

# 182. Route + Controller + Action example

Route:

```php
Route::delete('/admin/users/{user}', ...)
    ->authorize('internal.api');
```

Controller:

```php
#[Authorize('admin.access')]
```

Method:

```php
#[RequiresPermission('users.delete')]
#[Authorize('delete', subject: 'user')]
```

Resultado:

```text
internal.api
AND
admin.access
AND
users.delete permission
AND
UserPolicy::delete
```

---

# 183. Planner grouping

Estos requirements se podrán distribuir entre:

```text
PreResolution
Resource
```

de forma optimizada.

---

# 184. Example optimized phases

```text
PRE_RESOLUTION
  internal.api
  admin.access
  users.delete permission

RESOURCE
  TenantIsolation
  UserPolicy::delete
```

si el permiso no requiere User resource.

---

# 185. Important optimization

No es necesario esperar al binding para evaluar requisitos que no dependen del Subject.

---

# 186. Requirement Dependency Analysis

El compiler/planner podrá determinar:

```text
requiresSubject = yes/no
```

---

# 187. Phase promotion

Una regla declarada sin fase podría moverse a:

```text
PreResolution
```

si no depende de Subject y es seguro.

---

# 188. Conservative default

Si no puede probarse:

```text
keep declared/default phase
```

No mover de forma que cambie semántica.

---

# 189. Authorization Before Validation

Para requests de modificación, se recomienda evitar entregar errores de validación detallados a usuarios no autorizados.

---

# 190. Validation dependency

Si validation es necesaria para determinar resource identity, podrá ejecutarse la parte mínima necesaria.

---

# 191. Partial Validation

Futuro:

```text
Binding Validation
Authorization
Full Domain Validation
```

---

# 192. Middleware Pipeline integration

El routing middleware pipeline podrá dividirse:

```text
PreRouting
PostRouting
PreBinding
PostBinding
PreController
PostController
```

Authorization puede conectarse a puntos específicos.

---

# 193. Preferred hooks

```text
PostRouteMatch / PreBinding
    ↓
PreResolution Authorization

PostBinding / PreController
    ↓
Resource Authorization
```

---

# 194. Middleware interoperability

Un middleware externo podrá modificar Context antes de autorización si se encuentra en una fase autorizada.

Ejemplo:

```text
TenantResolverMiddleware
```

---

# 195. Security context immutability

Una vez creado el `AuthorizationRequest`, el contexto observado por esa decisión no deberá cambiar.

---

# 196. Middleware after plan

Un middleware posterior no debe poder modificar retroactivamente una decisión ya tomada.

---

# 197. Reauthorization

Si cambia un SecurityContext relevante:

```text
MFA completed
```

deberá generarse una nueva AuthorizationRequest.

---

# 198. Request mutation

No reutilizar una decisión anterior si cambió:

```text
Principal
Tenant
Subject
Security state
```

---

# 199. Controller Actions through invokable classes

Ejemplo:

```php
final class UpdateInvoiceController
{
    public function __invoke(Invoice $invoice)
    {
    }
}
```

Metadata class + `__invoke` deberán componerse normalmente.

---

# 200. Invokable Controller target ID

```text
controller_method:
UpdateInvoiceController::__invoke
```

---

# 201. Closures as route handlers

Una route closure también podrá tener metadata mediante Route DSL.

Ejemplo:

```php
Route::get('/admin/status', function () {
    // ...
})->authorize('admin.status.view');
```

---

# 202. Closure limitations

No puede utilizar Controller Attributes.

La Route metadata será la fuente de autorización.

---

# 203. Closure serialization

Route cache deberá manejar closure routes según las reglas generales del Routing System.

Authorization metadata seguirá siendo serializable.

---

# 204. Named Controllers

Si VoltStack soporta string targets:

```text
InvoiceController@update
```

deberán canonicalizarse a ControllerDescriptor durante compile.

---

# 205. No dynamic arbitrary Controller strings

Input externo no deberá controlar directamente Controller class/action resolution.

---

# 206. Resource Controllers

Con APIs como:

```text
Route::resource()
```

VoltStack podrá generar mappings por convención:

```text
index     → viewAny
show      → view
store     → create
update    → update
destroy   → delete
```

---

# 207. Optional Resource Authorization Convention

Esto puede reducir boilerplate.

---

# 208. Example

```php
Route::resource(
    'invoices',
    InvoiceController::class
)->authorizeResource(
    Invoice::class,
    'invoice'
);
```

---

# 209. Generated metadata

```text
index:
  viewAny Invoice::class

show:
  view invoice

store:
  create Invoice::class

update:
  update invoice

destroy:
  delete invoice
```

---

# 210. Laravel familiarity

Esto ofrece una experiencia similar a:

```text
authorizeResource()
```

pero compilada sobre el motor unificado.

---

# 211. Convention must remain optional

No todas las APIs REST siguen esos mappings.

La aplicación podrá sobrescribirlos explícitamente.

---

# 212. Resource mapping validation

El compiler verificará que:

```text
route parameter exists
controller method exists
subject type matches
ability is valid
```

---

# 213. Controller helper methods

Podrá existir:

```php
$this->authorize(
    'update',
    $invoice
);
```

por ergonomía.

---

# 214. Implementation

El helper delega a:

```text
AuthorizationManager
```

---

# 215. No hidden Controller state

El helper podrá obtener Principal mediante request-scoped resolver.

No guardar `$this->currentUser` automáticamente en Controller base.

---

# 216. AuthorizesRequests trait

Podrá existir:

```php
trait AuthorizesRequests
{
    protected function authorize(...): DecisionResult;
}
```

similar en ergonomía a Laravel.

---

# 217. Optional inheritance

No deberá exigir que Controllers extiendan una clase base específica.

Trait, helper o Facade serán opcionales.

---

# 218. Controller Contracts

Authorization integration deberá funcionar también con:

```text
plain callable Controller
invokable object
Action class
```

---

# 219. Route Cache + Authorization Cache

Ambos caches deberán coordinar versiones.

---

# 220. Cache fingerprint

Puede incluir:

```text
route metadata version
controller metadata version
authorization schema version
policy registry version
```

---

# 221. Stale cache protection

Si alguno cambia:

```text
rebuild
```

---

# 222. Production deployment

Flujo:

```text
discover routes
compile routes
compile Controller metadata
compile authorization metadata
validate bindings
build PlanTemplates
deploy
```

---

# 223. Development hot reload

Cambios en Attributes o Route authorization deberán invalidar metadata afectada.

---

# 224. Tooling — Route authorization

Comando:

```text
volt authorization:route invoice.update
```

Salida:

```text
Route:
PUT /invoices/{invoice}

Controller:
InvoiceController::update

PreResolution:
- authenticated
- admin.access

Resource:
- update invoice

Subject:
invoice → App\Domain\Invoice

Strategy:
deny_overrides
```

---

# 225. Tooling — Controller authorization

```text
volt authorization:controller \
App\Controller\InvoiceController::update
```

---

# 226. Output

```text
Inherited:
admin.access

Method:
permission invoice.update
ability update subject invoice

Routes:
invoice.update
internal.invoice.update
```

---

# 227. Explain lifecycle

```text
volt authorization:explain-route invoice.update
```

podrá mostrar qué se ejecuta antes y después del binding.

---

# 228. Linting

El sistema podrá detectar:

```text
resource binding unnecessarily before pre-auth
missing subject
unknown ability
Controller method without expected authorization
unsafe PublicAccess override
```

---

# 229. Security coverage

Tooling podrá exigir que determinados patrones de rutas estén protegidos.

Ejemplo:

```text
/admin/*
must require admin.access
```

---

# 230. CI policy

Esto podrá configurarse como reglas de arquitectura.

---

# 231. Route Authorization Coverage

Reporte:

```text
Protected Routes: 182
Public Routes: 24
Unclassified Routes: 3
```

---

# 232. Unclassified route

Puede producir warning/error.

---

# 233. Public routes explicit

Para sistemas estrictos, toda ruta deberá declarar:

```text
authorization requirement
```

o:

```text
PublicAccess
```

---

# 234. Secure-by-default routing mode

Configuración:

```text
routes default protected
```

podrá habilitarse.

---

# 235. Authentication default

Una app podría establecer:

```text
all routes require authenticated Principal
unless PublicAccess
```

---

# 236. Authorization default

Sin evaluator aplicable:

```text
Default Deny
```

sigue siendo válido.

---

# 237. Controller integration invariants

### Invariante 1

El Controller nunca ejecuta antes de completar autorización requerida.

### Invariante 2

Controller metadata se compila fuera del hot path.

### Invariante 3

Controller-level y method-level requirements son aditivos por defecto.

### Invariante 4

Controller Policies utilizan el mismo Policy System.

### Invariante 5

Programmatic y declarative authorization comparten AuthorizationManager.

---

# 238. Routing integration invariants

### Invariante 1

Route DSL produce metadata normalizada.

### Invariante 2

Route authorization no tiene motor propio.

### Invariante 3

Route Binding sigue siendo responsabilidad de Routing.

### Invariante 4

Un missing resource produce binding outcome, no Policy evaluation artificial.

### Invariante 5

Route groups heredan requirements de forma determinista.

---

# 239. Subject invariants

### Invariante 1

Resource Policies reciben Subjects ya resueltos.

### Invariante 2

Authorization no interpreta IDs crudos como entidades.

### Invariante 3

Subject references deben validarse durante compile cuando sea posible.

### Invariante 4

El runtime binding es request-scoped.

---

# 240. Phase invariants

### Invariante 1

PreResolution debe ejecutarse antes de trabajo innecesario cuando sea posible.

### Invariante 2

Resource Authorization solo ocurre cuando el Subject está disponible.

### Invariante 3

Una fase denegada detiene fases posteriores.

### Invariante 4

Todas las fases requeridas deben conceder acceso.

---

# 241. HTTP security invariants

### Invariante 1

DENY no expone razones internas por defecto.

### Invariante 2

HTTP mapping está fuera del Authorization Core.

### Invariante 3

403/404 concealment debe ser explícito.

### Invariante 4

Anonymous no implica automáticamente 401.

### Invariante 5

`authentication_required` puede mapearse a 401/redirect según canal.

---

# 242. Runtime invariants

### Invariante 1

Compiled metadata puede compartirse.

### Invariante 2

Resolved route arguments no pueden compartirse entre requests.

### Invariante 3

No se almacena Current Controller/Subject en singletons shared.

### Invariante 4

FrankenPHP workers limpian execution context después de cada request.

---

# 243. Performance invariants

### Invariante 1

No Reflection en producción normal.

### Invariante 2

PreResolution debe permitir evitar bindings innecesarios.

### Invariante 3

Solo deben resolverse inicialmente los argumentos necesarios para autorización cuando sea rentable.

### Invariante 4

PlanTemplates pueden precompilarse por Route/Controller action.

---

# 244. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Integration/
        ├── Http/
        │   ├── HttpAuthorizationCoordinator.php
        │   ├── HttpAuthorizationContext.php
        │   ├── HttpAuthorizationExceptionMapper.php
        │   ├── AuthorizationDenialVisibility.php
        │   └── AuthorizationHttpResultMapper.php
        │
        ├── Routing/
        │   ├── RouteAuthorizationMetadataAdapter.php
        │   ├── RouteAuthorizationCoordinator.php
        │   ├── RouteSubjectReferenceResolver.php
        │   ├── ResourceAuthorizationConvention.php
        │   └── RouteAuthorizationCompiler.php
        │
        ├── Controllers/
        │   ├── ControllerAuthorizationCoordinator.php
        │   ├── ControllerAuthorizationMetadataResolver.php
        │   ├── ControllerSubjectReferenceResolver.php
        │   ├── ControllerAuthorizationCompiler.php
        │   ├── ControllerArgumentDependencyPlan.php
        │   └── AuthorizesRequests.php
        │
        ├── Actions/
        │   ├── ActionAuthorizationCoordinator.php
        │   ├── ActionAuthorizationMetadataResolver.php
        │   └── ActionAuthorizationCompiler.php
        │
        └── Exceptions/
            ├── AuthorizationIntegrationException.php
            ├── ControllerAuthorizationException.php
            ├── RouteAuthorizationException.php
            ├── ActionAuthorizationException.php
            ├── AuthorizationSubjectBindingException.php
            └── AuthorizationLifecycleException.php
```

---

# 245. Arquitectura de integración final

```text
HTTP REQUEST
     │
     ↓
ROUTE MATCHER
     │
     ↓
Compiled Route Descriptor
     │
     ↓
Controller Target Descriptor
     │
     ↓
Effective Authorization Metadata
     │
     ├────────────────────────────┐
     │                            │
     ↓                            │
PRE-RESOLUTION                    │
AuthorizationPlan                │
     │                            │
     ↓                            │
DecisionManager                  │
     │                            │
 ┌───┴────┐                       │
 │        │                       │
DENY    GRANT                     │
 │        │                       │
STOP      ↓                       │
     Authorization-required       │
     Argument Resolution          │
             │                    │
             ↓                    │
        Subject Binding           │
             │                    │
             ↓                    │
      RESOURCE PHASE ◄────────────┘
             │
             ↓
      AuthorizationPlan
             │
             ↓
      DecisionManager
             │
       ┌─────┴─────┐
       │           │
     DENY        GRANT
       │           │
      STOP         ↓
            Resolve Remaining
            Controller Arguments
                   │
                   ↓
            Controller Invocation
                   │
                   ↓
                Response
```

---

# 246. Ejemplo completo

Route:

```php
Route::put(
    '/admin/invoices/{invoice}',
    [InvoiceController::class, 'update']
)
    ->authorize('internal.api');
```

Controller:

```php
#[Authorize(
    'admin.access',
    phase: AuthorizationPhase::PreResolution
)]
final class InvoiceController
{
    #[RequiresPermission('invoice.update')]
    #[Authorize(
        'update',
        subject: 'invoice'
    )]
    public function update(
        Invoice $invoice,
        UpdateInvoiceData $data
    ): Response {
        // ...
    }
}
```

---

# 247. Metadata efectiva

```text
Route:
internal.api

Controller:
admin.access

Method:
invoice.update permission
update Invoice
```

---

# 248. Fase PreResolution

Puede contener:

```text
PlatformLockdownPolicy
SuspendedPrincipalPolicy
internal.api Gate
admin.access Gate
invoice.update PermissionVoter
```

si ninguna requiere Invoice.

---

# 249. Resultado PreResolution

Si todos conceden:

```text
GRANT
```

continúa.

---

# 250. Binding

Solo ahora:

```text
invoice
    ↓
Invoice#928
```

---

# 251. Resource Plan

```text
TenantIsolationPolicy
InvoicePolicy::update
CompliancePolicy
```

---

# 252. Resource result

Si:

```text
DENY
```

entonces:

```text
UpdateInvoiceData
```

puede incluso no terminar de resolverse si no era requerido antes.

---

# 253. HTTP response

El mapper devuelve:

```text
403
```

o:

```text
404
```

si concealment está configurado.

---

# 254. Controller invocation

Solo con:

```text
PreResolution = GRANT
Resource = GRANT
```

se ejecuta:

```php
InvoiceController::update()
```

---

# 255. Filosofía del sistema

La integración deberá seguir:

```text
Authorize as early as possible.

Resolve only what authorization needs.

Do not mix Routing, Controller and Authorization responsibilities.

Treat Controllers and actions as legitimate Subjects when appropriate.

Use resource Policies when the rule belongs to the resource.

Compile static metadata.

Keep runtime bindings request-scoped.

Map authorization failures to transport-specific responses outside the Core.
```

---

# 256. Resultado esperado

El `Controller, Route and Action Authorization Integration System` permitirá que VoltStack proteja una operación mediante:

```php
#[Authorize('admin.access')]
```

```php
#[RequiresPermission('invoice.update')]
```

```php
#[Authorize('update', subject: 'invoice')]
```

o:

```php
Route::put(...)
    ->authorize('update', 'invoice');
```

sin crear mecanismos independientes.

Toda integración convergerá en:

```text
Route / Controller / Action Metadata
        ↓
EffectiveAuthorizationMetadata
        ↓
Authorization Phase
        ↓
AuthorizationManager
        ↓
AuthorizationPlanner
        ↓
AuthorizationPlan
        ↓
DecisionManager
        ↓
GRANT / DENY
```

El principio definitivo será:

```text
Routing determines where the request goes.

Binding determines what the resource is.

Controllers determine what code will execute.

Authorization determines whether that execution is allowed.

Each subsystem owns its responsibility,
and all authorization paths converge on one Core Engine.
```

Con esta integración, VoltStack podrá utilizar Policies no solo sobre modelos, sino también sobre Controllers, acciones, rutas y operaciones completas, manteniendo al mismo tiempo un lifecycle HTTP predecible, seguro y altamente optimizable.