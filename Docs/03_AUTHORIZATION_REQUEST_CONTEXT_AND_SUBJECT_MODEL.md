# VoltStack Authorization System — Authorization Request, Context and Subject Model

## 1. Propósito

Este documento define el modelo de datos operacional utilizado por el **Authorization System de VoltStack** para representar una solicitud de autorización.

El objetivo es establecer una estructura clara, segura e inmutable para describir:

```text
Quién solicita la acción
Qué acción intenta ejecutar
Sobre qué recurso intenta ejecutarla
En qué contexto ocurre
```

Estos conceptos se representan mediante:

```text
Principal
Ability
Subject
AuthorizationContext
```

y convergen en:

```text
AuthorizationRequest
```

El diseño debe soportar tanto casos simples:

```php
$user->can('update', $post);
```

como escenarios complejos:

```text
Principal
    +
Tenant
    +
Action
    +
Resource
    +
Controller
    +
Route
    +
Security Context
    +
Runtime Context
```

sin acoplar el Core de autorización a:

- HTTP;
- Routing;
- Controllers;
- ORM;
- sesiones;
- un modelo concreto `User`;
- un sistema específico de tenants.

---

# 2. Modelo conceptual

Toda autorización en VoltStack deberá normalizarse a:

```text
AuthorizationRequest
│
├── Principal
├── Ability
├── Subject
└── AuthorizationContext
```

Formalmente:

```text
AuthorizationRequest =
    Principal
    + Ability
    + SubjectDescriptor
    + AuthorizationContext
```

Ejemplo:

```text
Principal:
User#42

Ability:
invoice.approve

Subject:
Invoice#928

Context:
Tenant#7
Route: invoices.approve
Channel: web
Authentication: session+mfa
```

---

# 3. AuthorizationRequest como unidad canónica

Una vez construida, toda la infraestructura interna deberá operar sobre:

```text
AuthorizationRequest
```

No deberán existir variantes incompatibles como:

```text
ControllerAuthorizationRequest
GateAuthorizationRequest
PolicyAuthorizationRequest
RouteAuthorizationRequest
```

En su lugar:

```text
Controller
Route
Gate
Policy
Command
Job
```

serán diferentes orígenes de la misma representación.

---

# 4. Contrato conceptual

```php
final readonly class AuthorizationRequest
{
    public function __construct(
        public PrincipalInterface $principal,
        public Ability $ability,
        public SubjectDescriptor $subject,
        public AuthorizationContext $context,
        public AuthorizationRequestId $id,
    ) {}
}
```

El identificador podrá ser opcional internamente durante V1, pero se recomienda mantenerlo como concepto arquitectónico.

---

# 5. Inmutabilidad

`AuthorizationRequest` deberá ser inmutable.

Después de creado no podrán modificarse:

```text
principal
ability
subject
context
request id
```

Cualquier enriquecimiento deberá ocurrir antes de finalizar la construcción del objeto.

La inmutabilidad aporta:

- reproducibilidad;
- seguridad;
- trazabilidad;
- mejor debugging;
- menor riesgo de side effects;
- compatibilidad con concurrencia;
- mayor seguridad bajo FrankenPHP.

---

# 6. AuthorizationRequestFactory

La creación deberá estar centralizada en:

```text
AuthorizationRequestFactory
```

Flujo:

```text
Raw Authorization Input
        ↓
Principal Resolution
        ↓
Ability Normalization
        ↓
Subject Resolution
        ↓
Context Construction
        ↓
Trusted Context Validation
        ↓
AuthorizationRequest
```

Contrato conceptual:

```php
interface AuthorizationRequestFactoryInterface
{
    public function create(
        string|Ability $ability,
        mixed $subject = null,
        ?PrincipalInterface $principal = null,
        ?AuthorizationContext $context = null,
    ): AuthorizationRequest;
}
```

---

# 7. Principal Model

El `Principal` representa la identidad que intenta ejecutar una operación.

No deberá estar acoplado a:

```php
App\Models\User
```

VoltStack deberá trabajar contra una abstracción.

```php
interface PrincipalInterface
{
    public function authorizationIdentifier(): string|int;
}
```

---

# 8. Tipos de Principal

El modelo deberá poder representar al menos:

```text
Authenticated User
Anonymous User
Service Account
API Client
System Process
Machine Identity
Background Worker Identity
Tenant Service Identity
```

Ejemplos de implementaciones:

```text
UserPrincipal
AnonymousPrincipal
ServicePrincipal
ApiClientPrincipal
SystemPrincipal
MachinePrincipal
```

---

# 9. User como Principal

Las aplicaciones normales podrán utilizar directamente su entidad `User` si implementa:

```php
PrincipalInterface
```

Ejemplo:

```php
final class User implements PrincipalInterface
{
    public function authorizationIdentifier(): int
    {
        return $this->id;
    }
}
```

Esto evita tener que envolver obligatoriamente cada usuario en otro objeto.

---

# 10. Principal Adapters

Cuando una aplicación no quiera modificar su clase `User`, podrá utilizar un adapter.

```php
final readonly class UserPrincipalAdapter
    implements PrincipalInterface
{
    public function __construct(
        private User $user,
    ) {}

    public function authorizationIdentifier(): int
    {
        return $this->user->id;
    }
}
```

El `PrincipalResolver` podrá realizar esta adaptación.

---

# 11. AnonymousPrincipal

No se recomienda representar siempre la ausencia de autenticación como `null`.

Podrá utilizarse:

```php
final readonly class AnonymousPrincipal
    implements PrincipalInterface
{
    public function authorizationIdentifier(): string
    {
        return 'anonymous';
    }
}
```

Esto permite:

```text
AnonymousPrincipal
        ↓
ArticlePolicy::view()
```

sin contratos llenos de tipos nullable.

---

# 12. Anonymous no significa autorizado

`AnonymousPrincipal` únicamente representa una identidad.

No significa:

```text
ALLOW
```

ni:

```text
DENY
```

La Policy sigue tomando la decisión.

Ejemplo:

```text
AnonymousPrincipal
+
view
+
PublicArticle
        ↓
GRANT
```

pero:

```text
AnonymousPrincipal
+
update
+
Article
        ↓
DENY
```

---

# 13. SystemPrincipal

Procesos internos podrán actuar mediante:

```text
SystemPrincipal
```

Ejemplo:

```text
NightlyBillingJob
      ↓
SystemPrincipal
      ↓
invoice.process
```

No deberá suponerse que un `SystemPrincipal` tiene autorización ilimitada.

También deberá estar sujeto a Policies.

---

# 14. ServicePrincipal

Servicios internos o externos podrán utilizar:

```text
ServicePrincipal
```

Ejemplo:

```text
Service:
report-generator

Ability:
customer.export
```

Esto permite aplicar autorización machine-to-machine usando el mismo motor.

---

# 15. Principal Identity

El identificador de autorización debe ser estable durante la decisión.

Ejemplos:

```text
User#42
service:billing
api-client:foo
system:scheduler
anonymous
```

No deberá depender de:

```text
spl_object_id()
```

como identidad semántica.

---

# 16. PrincipalDescriptor

Para tracing y auditoría podrá existir:

```php
final readonly class PrincipalDescriptor
{
    public function __construct(
        public PrincipalType $type,
        public string|int $identifier,
        public ?string $class = null,
        public array $metadata = [],
    ) {}
}
```

El descriptor no sustituye al Principal real.

Sirve para:

- observabilidad;
- profiling;
- logs;
- auditoría;
- fingerprints.

---

# 17. PrincipalType

Conceptualmente:

```php
enum PrincipalType: string
{
    case User = 'user';
    case Anonymous = 'anonymous';
    case Service = 'service';
    case ApiClient = 'api_client';
    case System = 'system';
    case Machine = 'machine';
    case Custom = 'custom';
}
```

El enum podrá permanecer interno.

---

# 18. PrincipalResolver

El `PrincipalResolver` se encargará de obtener el Principal cuando no haya sido proporcionado explícitamente.

```php
interface PrincipalResolverInterface
{
    public function resolve(
        ?PrincipalInterface $principal = null,
    ): PrincipalInterface;
}
```

---

# 19. Resolución explícita

Si se proporciona:

```php
Authorization::for($user)
    ->check('update', $post);
```

el resolver deberá preferir ese Principal.

```text
Explicit Principal
      ↓
Use directly
```

No deberá sustituirlo por el usuario autenticado actual.

---

# 20. Resolución implícita

Si no existe Principal explícito:

```text
Authorization
    ↓
PrincipalResolver
    ↓
Authentication Context
```

El resolver podrá consultar al Authentication System mediante un contrato.

---

# 21. Resolución anónima

Si Authentication no proporciona identidad:

```text
PrincipalResolver
      ↓
AnonymousPrincipal
```

cuando el modo anónimo esté habilitado.

Alternativamente, configuraciones estrictas podrán producir:

```text
PrincipalResolutionException
```

para determinadas APIs internas.

---

# 22. Principal Resolution Chain

Podrá existir una cadena:

```text
ExplicitPrincipalResolver
        ↓
AuthenticationPrincipalResolver
        ↓
SystemExecutionPrincipalResolver
        ↓
AnonymousPrincipalResolver
```

Cada resolver podrá:

```text
RESOLVE
ABSTAIN
```

pero la cadena deberá producir finalmente un Principal válido.

---

# 23. No autenticación dentro del resolver

`PrincipalResolver` no deberá:

```text
validate password
verify token cryptographically
create session
refresh token
authenticate credentials
```

Estas responsabilidades pertenecen a Authentication.

El resolver únicamente obtiene la identidad ya establecida.

---

# 24. Ability Model

La `Ability` representa la acción solicitada.

```php
final readonly class Ability
{
    public function __construct(
        public string $name,
    ) {}
}
```

Ejemplos:

```text
view
create
update
delete
approve
publish
archive
execute
access
```

---

# 25. Ability Naming

VoltStack deberá aceptar dos estilos principales.

Simple:

```text
update
delete
approve
```

Namespaced:

```text
invoice.approve
admin.access
customer.export
system.deploy
```

Las Policies orientadas a resources normalmente podrán usar abilities simples.

Los Gates y permisos globales podrán beneficiarse de namespaces.

---

# 26. Ability no es Permission

Debe distinguirse:

```text
Ability
```

de:

```text
Permission
```

Una Ability describe:

```text
la acción solicitada
```

Una Permission describe potencialmente:

```text
una capacidad concedida dentro de RBAC
```

Ejemplo:

```text
Ability:
approve

Permission:
finance.invoices.approve
```

Una Policy puede utilizar la Permission para decidir sobre la Ability.

---

# 27. AbilityNormalizer

La entrada pública deberá convertirse a un objeto `Ability`.

```php
interface AbilityNormalizerInterface
{
    public function normalize(
        string|Ability $ability
    ): Ability;
}
```

---

# 28. Validación de Ability

El normalizador deberá rechazar:

```text
empty names
invalid control characters
invalid malformed namespace
oversized ability names
```

según reglas definidas.

Ejemplo:

```php
Authorization::check('', $invoice);
```

deberá producir un error de configuración o entrada, no evaluarse silenciosamente.

---

# 29. Ability Aliases

Los aliases podrán resolverse:

```text
edit → update
remove → delete
```

pero deben estar registrados explícitamente.

No deberán existir equivalencias ambiguas automáticas.

---

# 30. Canonical Ability Name

Después de la normalización:

```text
Ability
    ↓
canonical name
```

Ejemplo:

```text
Alias: edit
Canonical: update
```

Todas las fases posteriores deberán utilizar el canonical name.

---

# 31. Subject Model

El `Subject` representa aquello sobre lo cual se intenta ejecutar la Ability.

Puede existir como:

```text
Object Subject
Class Subject
Named Subject
Virtual Subject
No Subject
```

---

# 32. Object Subject

Ejemplo:

```php
Authorization::check('update', $invoice);
```

Aquí:

```text
Subject:
Invoice#928
```

La Policy podrá evaluar propiedades específicas de esa instancia.

---

# 33. Class Subject

Ejemplo:

```php
Authorization::check(
    'create',
    Invoice::class
);
```

No existe todavía una instancia.

Esto permite Policies como:

```php
InvoicePolicy::create(User $user);
```

---

# 34. Named Subject

Algunas operaciones pueden apuntar a un recurso lógico.

Ejemplo:

```php
Authorization::check(
    'access',
    'admin-dashboard'
);
```

Esto podrá normalizarse como:

```text
NamedSubject("admin-dashboard")
```

No deberá confundirse automáticamente un string de clase con un named subject.

---

# 35. Virtual Subject

VoltStack podrá soportar subjects construidos expresamente.

Ejemplo:

```php
new VirtualSubject(
    type: 'report',
    identifier: 'monthly-financial-summary',
);
```

Esto será útil cuando no exista una clase real del dominio.

---

# 36. No Subject

Los Gates globales podrán ejecutarse sin recurso.

```php
Authorization::check('access-admin');
```

Internamente:

```text
SubjectType::None
```

en lugar de obligar a crear un recurso artificial.

---

# 37. SubjectDescriptor

El Core no deberá depender directamente de la estructura del subject.

Se normalizará mediante:

```php
final readonly class SubjectDescriptor
{
    public function __construct(
        public SubjectType $type,
        public mixed $value,
        public ?string $class = null,
        public string|int|null $identifier = null,
        public ?string $name = null,
        public array $metadata = [],
    ) {}
}
```

---

# 38. SubjectType

```php
enum SubjectType: string
{
    case None = 'none';
    case Object = 'object';
    case ClassName = 'class';
    case Named = 'named';
    case Virtual = 'virtual';
}
```

---

# 39. Object Descriptor

Ejemplo:

```text
type:
object

class:
App\Domain\Invoice

identifier:
928

value:
Invoice instance
```

El identifier podrá obtenerse mediante resolvers especializados.

---

# 40. Class Descriptor

Para:

```php
Invoice::class
```

se representará:

```text
type:
class

class:
App\Domain\Invoice

identifier:
null
```

---

# 41. Named Descriptor

Ejemplo:

```text
type:
named

name:
admin-dashboard
```

No habrá necesariamente:

```text
class
identifier
```

---

# 42. SubjectResolver

Contrato conceptual:

```php
interface SubjectResolverInterface
{
    public function resolve(
        mixed $subject
    ): SubjectDescriptor;
}
```

---

# 43. Subject Resolver Chain

Podrá existir:

```text
NullSubjectResolver
    ↓
ObjectSubjectResolver
    ↓
ClassSubjectResolver
    ↓
NamedSubjectResolver
    ↓
VirtualSubjectResolver
    ↓
CustomSubjectResolvers
```

El orden deberá evitar ambigüedad.

---

# 44. Class string detection

Si se recibe:

```php
Invoice::class
```

el resolver puede utilizar:

```php
class_exists($subject)
```

durante desarrollo.

En producción, metadata compilada podrá evitar verificaciones repetitivas.

---

# 45. No implicit database loading

Una regla central:

```text
Authorization does not perform model binding.
```

Si el developer proporciona:

```php
Authorization::check('update', 928);
```

VoltStack no deberá inferir:

```text
Invoice ID = 928
```

sin un descriptor explícito.

---

# 46. Resource binding pertenece a integración

Ejemplo HTTP:

```text
/invoices/{invoice}
        ↓
Routing
        ↓
Entity Binding
        ↓
Invoice#928
        ↓
Authorization
```

Esto conserva la independencia del ORM.

---

# 47. Subject Identity Resolver

Para auditoría y memoización podrá existir:

```php
interface SubjectIdentityResolverInterface
{
    public function identifier(
        object $subject
    ): string|int|null;
}
```

Implementaciones podrán conocer:

- entidades ORM;
- UUID objects;
- aggregate roots;
- custom domain resources.

---

# 48. Subject Identity no obligatoria

Una Policy puede autorizar un objeto sin identificador persistente.

Ejemplo:

```php
$draft = new ReportDraft();
```

Por ello:

```text
identifier = null
```

debe ser válido.

---

# 49. Subject Class Mapping

Policy resolution deberá usar principalmente:

```text
Subject class
```

para object/class subjects.

Ejemplo:

```text
Invoice
    ↓
InvoicePolicy
```

Para named/virtual subjects podrán utilizarse otros registries.

---

# 50. Inheritance

Si:

```text
PremiumInvoice extends Invoice
```

el PolicyResolver deberá definir si:

```text
PremiumInvoice
    ↓
PremiumInvoicePolicy
```

o fallback:

```text
InvoicePolicy
```

La resolución exacta se documentará dentro de Policy Resolution.

---

# 51. Interfaces como subjects

También podrán existir mappings contra interfaces.

Ejemplo:

```text
BillableResource
    ↓
BillingPolicy
```

Esto permite autorización transversal por capacidades del dominio.

Las reglas de precedencia deberán ser explícitas.

---

# 52. AuthorizationContext

El contexto representa información adicional relevante para la decisión.

No debe convertirse en un objeto gigante dependiente de toda la aplicación.

Conceptualmente:

```php
final readonly class AuthorizationContext
{
    public function __construct(
        public ?string $channel,
        public mixed $tenant,
        public mixed $request,
        public mixed $route,
        public mixed $controller,
        public SecurityContext $security,
        public RuntimeContext $runtime,
        public array $attributes = [],
    ) {}
}
```

La estructura final podrá ser más modular.

---

# 53. Context objetivo

El Context responde:

```text
¿En qué circunstancias ocurre esta autorización?
```

Ejemplos:

```text
tenant
channel
route
controller
authentication method
MFA state
origin
runtime
request metadata
security attributes
```

---

# 54. Context no debe duplicar Principal

No se recomienda almacenar:

```text
context.user
```

si ya existe:

```text
AuthorizationRequest.principal
```

El Context podrá contener información de autenticación, pero no deberá duplicar identidad innecesariamente.

---

# 55. Channel

Podrá identificar el origen lógico:

```text
web
api
cli
queue
scheduler
internal
websocket
spa
```

Ejemplo:

```php
$context->channel();
```

Una Policy podría decir:

```text
Ability export
allowed only from internal channel
```

---

# 56. Tenant Context

El contexto podrá contener:

```text
TenantContext
```

No simplemente:

```text
tenant_id
```

si la aplicación necesita metadata adicional.

Ejemplo:

```php
$context->tenant()?->identifier();
```

---

# 57. Request Context

Las integraciones HTTP podrán adjuntar una abstracción ligera del request.

No se recomienda que todas las Policies dependan directamente de:

```php
ServerRequestInterface
```

si solo necesitan:

```text
IP
method
origin
```

Podrá utilizarse un:

```text
AuthorizationHttpContext
```

reducido.

---

# 58. Route Context

Routing podrá adjuntar:

```text
route name
route parameters
route metadata
controller target
```

sin obligar al Core a conocer la implementación del router.

---

# 59. Controller Context

La integración con Controllers podrá incorporar:

```text
controller class
action method
controller metadata
authorization phase
```

Esto permite Policies aplicadas sobre controllers y actions.

---

# 60. SecurityContext

Información de seguridad adicional podrá agruparse en:

```php
final readonly class SecurityContext
{
    public function __construct(
        public bool $authenticated,
        public bool $mfaVerified,
        public ?string $authenticationMethod,
        public ?string $sessionStrength,
        public array $attributes = [],
    ) {}
}
```

Esto evita contaminar `AuthorizationContext` con demasiados campos.

---

# 61. RuntimeContext

Podrá representar:

```text
runtime type
worker id
execution type
environment
long-running flag
```

Ejemplo:

```text
runtime = frankenphp
execution = http
```

Normalmente las Policies no necesitarán esta información, pero ciertos controles de seguridad sí.

---

# 62. Context Attributes

Para extensión deberá existir:

```text
attributes
```

Ejemplo:

```php
$context->attribute('risk_score');
```

pero su uso deberá ser controlado.

---

# 63. Typed Context Extensions

Cuando una feature sea importante, será preferible un objeto tipado:

```text
RiskContext
ComplianceContext
TenantContext
HttpContext
```

sobre:

```text
$context->attributes['foo']
```

Esto mejora:

- análisis estático;
- documentación;
- contratos;
- seguridad.

---

# 64. AuthorizationContextBuilder

La construcción podrá utilizar un builder mutable de corta vida:

```php
$builder
    ->channel('web')
    ->tenant($tenant)
    ->security($securityContext)
    ->attribute('risk_score', 0.28);
```

Finalmente:

```php
$context = $builder->build();
```

El resultado será inmutable.

---

# 65. ContextFactory

La factory coordinará providers.

```php
interface AuthorizationContextFactoryInterface
{
    public function create(
        ?AuthorizationContext $explicit = null,
    ): AuthorizationContext;
}
```

---

# 66. Context Provider Model

Contrato conceptual:

```php
interface AuthorizationContextProviderInterface
{
    public function contribute(
        AuthorizationContextBuilder $builder,
        AuthorizationContextEnvironment $environment,
    ): void;
}
```

Providers iniciales:

```text
AuthenticationContextProvider
TenantContextProvider
HttpContextProvider
RouteContextProvider
ControllerContextProvider
RuntimeContextProvider
```

---

# 67. Context Provider Order

Podrá utilizarse prioridad:

```text
Runtime           100
Authentication    200
Tenant            300
HTTP              400
Routing           500
Controller        600
Explicit Context  1000
```

Sin embargo, campos trusted podrán tener reglas diferentes.

---

# 68. Explicit Context

El developer podrá proporcionar contexto:

```php
Authorization::check(
    'approve',
    $invoice,
    context: $context,
);
```

Esto será especialmente útil fuera de HTTP.

---

# 69. Context Merge Policy

El merge deberá seguir reglas deterministas.

Ejemplo conceptual:

```text
Base Context
    ↓
Automatic Providers
    ↓
Explicit Extensions
    ↓
Trusted Field Enforcement
    ↓
Final Context
```

---

# 70. Trusted Context

Ciertos valores deben considerarse de confianza únicamente si provienen de fuentes autorizadas.

Ejemplos:

```text
current tenant
authenticated principal state
MFA state
security level
verified client identity
```

Un developer no debería poder falsificar accidentalmente:

```php
['mfa_verified' => true]
```

mediante attributes genéricos.

---

# 71. Trusted Context Source

Podrá existir metadata interna:

```text
ContextValue
├── value
├── source
└── trustLevel
```

Esto podría ser excesivo para V1, pero debe conservarse como posibilidad arquitectónica.

---

# 72. Context Spoofing Protection

Las APIs públicas deberán diferenciar:

```text
application context
```

de:

```text
security context
```

Ejemplo seguro:

```php
$context->withAttribute('document_version', 3);
```

pero no:

```php
$context->withSecurityAttribute('mfa_verified', true);
```

sin una API privilegiada.

---

# 73. Security Context Providers

Solo providers registrados como trusted deberán poder aportar:

```text
authentication state
tenant identity
MFA
token assurance
```

---

# 74. Context precedence

Si existe conflicto:

```text
Explicit Tenant
vs
Resolved Tenant
```

el framework no deberá resolverlo silenciosamente.

Dependiendo de API:

```text
trusted override
or
ContextConflictException
```

Será preferible detectar inconsistencias.

---

# 75. ContextConflictException

Ejemplo:

```text
HTTP tenant = 12
Explicit tenant = 18
```

Resultado:

```text
AuthorizationContextConflictException
```

en lugar de elegir arbitrariamente.

Esto es particularmente importante para multi-tenancy.

---

# 76. Context completeness

No todas las decisiones requieren todos los datos.

El Context podrá tener:

```text
tenant = null
route = null
request = null
controller = null
```

cuando se ejecute desde CLI o tests.

Policies deberán declarar o manejar correctamente los datos requeridos.

---

# 77. Context Requirements

Una Policy avanzada podrá declarar:

```text
requires tenant
requires authenticated security context
requires HTTP context
```

Ejemplo conceptual:

```php
#[RequiresAuthorizationContext('tenant')]
final class TenantResourcePolicy
{
}
```

Si el contexto requerido no existe:

```text
ABSTAIN
DENY
or configuration failure
```

según tipo de requisito.

---

# 78. Context Missing vs Authorization Denied

Deben distinguirse:

```text
Required context missing
```

de:

```text
Context exists but authorization fails
```

El primero puede indicar un fallo de integración.

El segundo es una decisión legítima.

---

# 79. Authorization Environment

Para construir contextos podrá existir:

```text
AuthorizationEnvironment
```

que represente dependencias del entorno de ejecución.

No deberá exponerse a Policies.

Ejemplo:

```text
Current request accessor
Current route accessor
Current tenant accessor
Authentication accessor
Runtime accessor
```

---

# 80. Request metadata

`AuthorizationRequest` podrá contener metadata independiente del Context:

```text
request id
created at monotonic timestamp
origin
```

Pero deberá evitarse duplicar datos.

---

# 81. AuthorizationRequestId

El identificador podrá generarse usando:

- UUID;
- ULID;
- runtime-optimized monotonic IDs.

No deberá contener datos sensibles.

Ejemplo:

```text
authz_01K3...
```

---

# 82. Correlation

Una decisión podrá correlacionarse:

```text
HTTP Request ID
        ↓
AuthorizationRequest ID
        ↓
Trace ID
        ↓
Audit Event ID
```

Esto facilitará troubleshooting.

---

# 83. Parent Authorization Request

Para nested authorization podrá existir:

```text
parentRequestId
```

Ejemplo:

```text
Authorization A
    ↓
Policy invokes Authorization B
```

Esto ayuda a detectar ciclos y construir traces jerárquicos.

---

# 84. Authorization Depth

El execution context podrá rastrear:

```text
depth
```

Ejemplo:

```text
0 → initial authorization
1 → nested authorization
2 → nested authorization
```

Se podrá imponer un máximo configurable para prevenir abuso.

---

# 85. Subject Resolution in Controllers

Para:

```php
#[Authorize('update', subject: 'invoice')]
public function update(Invoice $invoice)
{
}
```

la integración deberá traducir:

```text
subject = "invoice"
```

a:

```text
Controller Argument
        ↓
Invoice instance
        ↓
SubjectDescriptor
```

---

# 86. Subject Reference

Los atributos declarativos podrán utilizar:

```text
SubjectReference
```

en lugar de strings arbitrarios.

Conceptualmente:

```php
new SubjectReference(
    source: ControllerArgument,
    name: 'invoice',
);
```

El compilador podrá generar esta metadata.

---

# 87. Controller as Subject

También deberá poder autorizarse el propio Controller:

```php
#[Authorize('access', subject: self::class)]
final class AdminController
{
}
```

Internamente:

```text
SubjectType:
class

Subject:
AdminController
```

---

# 88. Controller Action as Subject

Opcionalmente podrá modelarse una acción como subject virtual:

```text
App\Controller\UserController::destroy
```

Esto será útil para Policies orientadas a operaciones.

Ejemplo:

```text
SubjectType:
virtual

type:
controller_action

identifier:
UserController::destroy
```

---

# 89. Route as Subject

Una ruta también podrá convertirse en Subject.

Ejemplo:

```text
route:admin.users.destroy
```

Esto permite Policies sobre infraestructura de routing sin mezclarla con Controllers.

---

# 90. Command as Subject

Ejemplo:

```php
Authorization::check(
    'execute',
    ImportCustomersCommand::class
);
```

Se resolverá como:

```text
SubjectType::ClassName
```

---

# 91. Job as Subject

Un Job podrá autorizarse de igual manera:

```text
Ability:
dispatch

Subject:
ExportCustomerDataJob
```

o sobre la instancia si contiene parámetros relevantes.

---

# 92. Component as Subject

Los componentes UI podrán utilizarse como Subject:

```text
Ability:
render

Subject:
AdminDashboardComponent
```

aunque normalmente las Policies deberán proteger la operación de backend, no simplemente la visibilidad del componente.

---

# 93. Multiple Subjects

Una operación puede involucrar más de un recurso.

Ejemplo:

```text
move document
from Folder A
to Folder B
```

V1 puede utilizar:

```text
Primary Subject:
Document

Context:
source folder
target folder
```

Pero podría evolucionarse hacia:

```text
SubjectSet
```

---

# 94. SubjectSet

Futura abstracción:

```php
final readonly class SubjectSet
{
    public function __construct(
        public SubjectDescriptor $primary,
        public array $related,
    ) {}
}
```

No deberá introducirse hasta existir una necesidad clara.

---

# 95. Related Resources

Mientras tanto, los recursos relacionados podrán incorporarse mediante Context tipado.

Ejemplo:

```text
DocumentMoveContext
├── sourceFolder
└── targetFolder
```

---

# 96. Context Serialization

`AuthorizationContext` no deberá asumirse serializable.

Puede contener referencias runtime.

Para tracing se generará:

```text
AuthorizationContextDescriptor
```

con datos seguros y normalizados.

---

# 97. ContextDescriptor

Ejemplo:

```php
final readonly class AuthorizationContextDescriptor
{
    public function __construct(
        public ?string $channel,
        public string|int|null $tenantId,
        public ?string $route,
        public ?string $controller,
        public array $safeAttributes,
    ) {}
}
```

---

# 98. Sensitive Data Filtering

No deberán entrar automáticamente en traces:

```text
Authorization headers
cookies
session IDs
tokens
passwords
full request body
PII unrelated to authorization
```

---

# 99. Authorization Input Sanitization

Aunque las Policies reciben objetos internos, abilities y named subjects pueden provenir indirectamente de configuración.

El Core deberá validar:

```text
ability names
subject names
context keys
metadata size
```

para evitar abuso o contaminación de logs.

---

# 100. Context attribute naming

Se recomienda namespacing:

```text
risk.score
compliance.region
http.ip
application.workflow
```

para extensiones genéricas.

Campos estructurales importantes deberán seguir siendo propiedades tipadas.

---

# 101. Context Copying

Al añadir atributos a un contexto inmutable:

```php
$new = $context->withAttribute(
    'risk.score',
    0.25
);
```

deberá producirse un nuevo objeto.

No modificar:

```php
$context
```

existente.

---

# 102. Context performance

Dado que autorización puede ejecutarse muchas veces por request, no se recomienda reconstruir completamente el Context en cada llamada.

Podrá existir:

```text
Base Request AuthorizationContext
```

request-scoped.

Cada decisión podrá derivar de él de forma barata.

---

# 103. Base Authorization Context

Durante el inicio de la request:

```text
Authentication
Tenant
HTTP
Routing
        ↓
BaseAuthorizationContext
```

Luego:

```text
Authorization Decision
    ↓
derive context
```

solo cuando exista información específica.

---

# 104. Context Snapshot

La Policy deberá observar una snapshot consistente.

Si el tenant o authentication context cambian durante una misma decisión, no deberán alterar el request ya creado.

Esto refuerza la inmutabilidad de `AuthorizationRequest`.

---

# 105. Persistent Runtime Safety

Nunca deberán persistir accidentalmente:

```text
Principal
Tenant
Request
Route
Controller instance
SecurityContext
```

entre requests.

El runtime podrá mantener:

```text
Context Provider definitions
Metadata
Resolvers
```

pero no:

```text
Context values
```

---

# 106. Safe Shared State

Puede ser process-wide:

```text
Ability registry
Subject resolver registry
Context provider definitions
Compiled metadata
```

Debe ser request-scoped:

```text
current principal
current tenant
base authorization context
request id
nested authorization stack
```

---

# 107. Context Reset

En FrankenPHP:

```text
Request start
    ↓
Create AuthorizationExecutionContext
    ↓
Process decisions
    ↓
Request terminate
    ↓
Clear execution state
```

Este cleanup deberá ser parte del runtime lifecycle del framework.

---

# 108. CLI Context

En CLI:

```text
channel:
cli

request:
null

route:
null

controller:
null
```

El Principal podrá ser:

```text
SystemPrincipal
```

o uno explícito.

---

# 109. Queue Context

En una Queue:

```text
channel:
queue
```

Podrán existir:

```text
originating principal descriptor
tenant
job metadata
```

pero la propagación de identidad deberá hacerse explícitamente.

---

# 110. Identity propagation

No se deberá serializar automáticamente un objeto completo `User` dentro de contextos de queue únicamente para autorización.

Preferido:

```text
PrincipalReference
```

con:

```text
type
identifier
tenant scope
authorization-relevant version
```

El Job resolverá la identidad según reglas seguras.

---

# 111. PrincipalReference

Conceptualmente:

```php
final readonly class PrincipalReference
{
    public function __construct(
        public PrincipalType $type,
        public string|int $identifier,
    ) {}
}
```

Su resolución pertenecerá a una integración especializada.

---

# 112. Impersonation

El sistema deberá contemplar impersonación administrativa.

Debe distinguirse:

```text
Effective Principal
```

de:

```text
Actor Principal
```

Ejemplo:

```text
Admin#1 impersonates User#42
```

Authorization normalmente evalúa:

```text
effective = User#42
```

mientras Audit registra:

```text
actor = Admin#1
effective = User#42
```

---

# 113. Actor Context

Podrá existir:

```text
ActorPrincipal
```

dentro de `AuthorizationContext` o de un contexto especializado.

No deberá sustituir al Principal efectivo.

---

# 114. Delegation

Escenarios de delegación pueden necesitar:

```text
Actor
Principal
Delegation Grant
```

El modelo deberá permitir agregar esta información posteriormente sin modificar `AuthorizationRequest`.

---

# 115. Multi-Tenant Principal

No se recomienda que:

```text
User#42@Tenant#7
```

se represente obligatoriamente como una identidad totalmente distinta.

Puede modelarse:

```text
Principal:
User#42

Context:
Tenant#7
```

Esto simplifica identidad.

---

# 116. Tenant-bound Principal

Sin embargo, ciertos sistemas podrán implementar:

```text
TenantPrincipal
```

cuando la identidad sea realmente específica del tenant.

El Core no deberá impedirlo.

---

# 117. Tenant mismatch

Ejemplo:

```text
Principal User#42
Context Tenant#7
Subject Invoice#928 Tenant#9
```

Una Policy transversal puede devolver:

```text
DENY
reasonCode:
tenant_mismatch
```

---

# 118. Subject tenant metadata

Authorization no deberá asumir que todo Subject tiene:

```php
$subject->tenant_id
```

La integración multi-tenant podrá utilizar:

```text
TenantSubjectResolver
```

para extraer scope de manera desacoplada.

---

# 119. Subject Metadata Providers

Similar al Context, podrán existir providers de metadata del Subject.

Ejemplos:

```text
TenantSubjectMetadataProvider
DomainSubjectMetadataProvider
ORMSubjectIdentityProvider
```

Esto permitirá enriquecer descriptors sin modificar entidades.

---

# 120. Metadata vs Policy data

Los descriptors deben contener información estructural.

No deben reemplazar acceso real al objeto.

Por ejemplo:

```text
SubjectDescriptor.identifier
```

es correcto.

Copiar todos los campos del Invoice a metadata:

```text
amount
customer
currency
status
...
```

no es recomendable.

La Policy puede recibir la instancia real.

---

# 121. Policy Invocation Model

Una Policy podrá recibir:

```php
public function update(
    User $user,
    Invoice $invoice,
    AuthorizationContext $context,
): DecisionResult
```

El dispatcher resolverá:

```text
Principal
Raw Subject Value
Context
```

desde `AuthorizationRequest`.

---

# 122. Principal type compatibility

Si una Policy espera:

```php
User $user
```

pero el request contiene:

```text
AnonymousPrincipal
```

el dispatcher no deberá provocar un `TypeError` sin control.

Podrá determinar que la Policy:

```text
does not support this Principal
```

y responder:

```text
ABSTAIN
```

o `DENY` según metadata.

La semántica exacta se documentará en Policy Dispatch.

---

# 123. Context typed injection

Si una Policy espera:

```php
AuthorizationContext $context
```

el dispatcher podrá inyectarlo.

También podrían existir tipos específicos:

```php
TenantContext $tenant
SecurityContext $security
```

pero deberá evitarse que la firma de Policy se convierta en DI arbitrario.

---

# 124. Subject-specific context

La construcción del Context podrá realizarse antes o después de subject resolution según fase.

Para Pre-Resolution Authorization:

```text
Context
without resource instance
```

Para Resource Authorization:

```text
Context
+
resolved subject
```

No deberán mezclarse fases de manera implícita.

---

# 125. Authorization Phase

El Context podrá incluir:

```text
AuthorizationPhase
```

Ejemplo:

```php
enum AuthorizationPhase
{
    case PreResolution;
    case Resource;
    case PostResolution;
}
```

Esto permite a evaluadores transversales conocer el momento de ejecución.

---

# 126. Phase no es Ability

Debe distinguirse:

```text
Ability:
update
```

de:

```text
Phase:
resource
```

La primera expresa qué se intenta hacer.

La segunda dónde está el framework dentro del lifecycle.

---

# 127. Request Origin

El Context podrá contener:

```text
AuthorizationOrigin
```

Ejemplos:

```text
Programmatic
Gate
Route
Controller
Component
Command
Job
Directive
SPAProjection
```

Esto es útil para tracing.

No debería cambiar normalmente el resultado de seguridad.

---

# 128. Origin-independent policies

Las Policies de dominio deberían producir la misma decisión independientemente de si fueron invocadas desde:

```text
Controller
Command
API
```

salvo que el canal sea realmente una condición del negocio o seguridad.

---

# 129. Capability Projection Context

Cuando el frontend pregunte:

```text
Can user update this Invoice?
```

el contexto podrá tener:

```text
origin:
spa_projection
```

La Policy seguirá siendo la misma.

Esto evita divergencia entre UI y backend.

---

# 130. Context versioning

Podrá existir:

```text
ContextFingerprint
```

o versión request-scoped para memoización.

No se recomienda serializar el Context entero para crear una clave.

---

# 131. Relevant Context Keys

Policies memoizables podrán declarar qué contexto afecta su decisión.

Ejemplo:

```text
tenant
mfa
channel
```

Esto permitiría construir fingerprints seguros en el futuro.

---

# 132. Request Equality

Dos `AuthorizationRequest` no serán necesariamente equivalentes por tener:

```text
same principal
same ability
same subject
```

si cambia:

```text
context
```

Por tanto la igualdad debe considerar contexto relevante.

---

# 133. Decision Reuse Safety

Ejemplo:

```text
User#42
approve
Invoice#928

Context A:
MFA verified

Context B:
MFA not verified
```

No debe reutilizarse la misma decisión.

---

# 134. Time Context

No se recomienda incorporar automáticamente:

```text
current timestamp
```

al Context de todas las decisiones.

Eso destruiría posibilidades de memoización.

Policies dependientes del tiempo deberán solicitar explícitamente:

```text
ClockInterface
```

como dependencia o un `TimeAuthorizationContext`.

---

# 135. External Risk Context

Para sistemas avanzados:

```text
RiskContext
├── score
├── level
└── source
```

podrá incorporarse mediante un provider.

Ejemplo:

```text
risk.score = 0.92
```

Una Policy puede requerir MFA adicional.

---

# 136. Compliance Context

Podrá existir:

```text
ComplianceContext
├── region
├── classification
├── regulatoryFlags
```

Esto facilita ABAC/PBAC empresarial.

---

# 137. Device Context

Opcionalmente:

```text
DeviceContext
├── trusted
├── type
├── posture
```

Puede participar en decisiones zero-trust.

---

# 138. Network Context

Podrá contener:

```text
source IP
network zone
proxy verification status
```

pero únicamente si los datos fueron normalizados y validados por infraestructura confiable.

---

# 139. Untrusted HTTP headers

Policies no deberán confiar directamente en:

```text
X-Forwarded-For
X-User-Role
X-Tenant
```

sin que el Http/Security subsystem los haya validado.

---

# 140. Request Context boundary

Esta separación es crítica:

```text
Raw HTTP Request
        ↓
Trusted HTTP Context Builder
        ↓
AuthorizationHttpContext
        ↓
Policy
```

No:

```text
Raw client-controlled headers
        ↓
Policy
```

---

# 141. AuthorizationContext API

API conceptual:

```php
$context->channel();

$context->tenant();

$context->security();

$context->runtime();

$context->route();

$context->controller();

$context->attribute('risk.score');
```

No deberán exponerse demasiados setters porque el objeto final será inmutable.

---

# 142. Context Presence API

Podrá ofrecer:

```php
$context->hasTenant();

$context->hasHttp();

$context->hasRoute();
```

para simplificar Policies multi-environment.

---

# 143. Required context API

Opcional:

```php
$tenant = $context->requireTenant();
```

que lance:

```text
MissingAuthorizationContextException
```

si la integración está incompleta.

Esto debe utilizarse solo cuando ausencia de contexto sea un error técnico.

---

# 144. Request descriptor

Para observabilidad podrá generarse:

```text
AuthorizationRequestDescriptor
```

Ejemplo:

```text
id: authz_123
principal: user:42
ability: invoice.approve
subject: invoice:928
tenant: 7
origin: controller
```

No deberá almacenar objetos runtime completos.

---

# 145. Descriptor factory

```text
AuthorizationRequest
        ↓
DescriptorFactory
        ↓
Safe Serializable Descriptor
```

Esto será utilizado por:

```text
Trace
Profiler
Audit
Metrics
```

---

# 146. Redaction

El descriptor deberá aplicar reglas:

```text
safe
sensitive
secret
```

para atributos de Context.

Ejemplo:

```text
risk.score → safe
session.token → forbidden
customer.email → redact unless required
```

---

# 147. Audit Context

Auditoría podrá requerir más datos que observabilidad, pero deberá seguir principios de minimización.

No se deberá persistir todo el AuthorizationContext por defecto.

---

# 148. Context minimization

Principio:

```text
Only authorization-relevant data
should enter AuthorizationContext.
```

Esto mejora:

- seguridad;
- privacidad;
- rendimiento;
- claridad arquitectónica.

---

# 149. Request construction performance

La construcción debe evitar:

```text
deep cloning
serialization
reflection
filesystem access
database access
```

en el hot path.

Resolvers y providers deberán utilizar metadata ya disponible.

---

# 150. Lazy Context Values

Algunos valores costosos podrán resolverse lazy.

Ejemplo:

```text
GeoIP
Risk Score
Device Reputation
```

pero no deberán evaluarse si ninguna Policy los necesita.

---

# 151. LazyAuthorizationContextValue

Futura abstracción:

```text
Lazy Context Value
        ↓
Resolve only when requested
```

Debe implementarse con cuidado para preservar inmutabilidad semántica.

---

# 152. Context dependency planning

El `AuthorizationPlanner` podría conocer:

```text
Policy A requires tenant
Policy B requires risk
Policy C requires security
```

y solicitar únicamente esos providers.

Esto sería una optimización avanzada.

---

# 153. V1 recommendation

Para V1:

```text
Build lightweight standard context eagerly.
Resolve expensive extensions lazily.
```

Esto ofrece un balance razonable.

---

# 154. Context provider failure

Si un provider opcional falla:

```text
ABSTAIN contribution
```

podrá ser válido.

Si falla un provider trusted requerido:

```text
Authorization Context Failure
        ↓
Fail Closed
```

---

# 155. Tenant provider failure

Ejemplo:

```text
Route expects tenant
TenantContextProvider cannot resolve tenant
```

No deberá continuar como:

```text
tenant = null
```

silenciosamente si la ruta exige tenant.

Debe tratarse como error de integración/seguridad.

---

# 156. Principal resolver failure

Si no puede determinarse identidad y anonymous está permitido:

```text
AnonymousPrincipal
```

Si existe una autenticación supuestamente establecida pero está inconsistente:

```text
PrincipalResolutionException
```

No debe degradarse silenciosamente a anonymous.

---

# 157. Subject resolver failure

Si el subject no puede interpretarse:

```text
SubjectResolutionException
```

Ejemplo:

```php
Authorization::check(
    'update',
    fopen(...)
);
```

si el tipo no está soportado.

---

# 158. Ability normalization failure

Debe producir:

```text
InvalidAbilityException
```

No:

```text
DENY
```

silencioso en desarrollo, porque puede ocultar errores de programación.

En producción, la frontera de autorización aplicará fail-closed.

---

# 159. Context validation

Antes de crear `AuthorizationRequest` podrá ejecutarse:

```text
AuthorizationContextValidator
```

para comprobar:

```text
tenant conflicts
invalid trusted fields
phase consistency
origin consistency
security state
```

---

# 160. Request Factory final flow

```text
                    Raw Input
                       │
         ┌─────────────┼─────────────┐
         ↓             ↓             ↓
    Principal       Ability       Subject
    Resolver       Normalizer     Resolver
         └─────────────┼─────────────┘
                       ↓
              Context Factory
                       ↓
              Context Providers
                       ↓
              Context Validator
                       ↓
          AuthorizationRequestFactory
                       ↓
             AuthorizationRequest
```

---

# 161. Controller example

Código:

```php
#[Authorize(
    'update',
    subject: 'invoice'
)]
public function update(
    Invoice $invoice
): Response {
}
```

Transformación:

```text
Principal:
Authenticated User#42

Ability:
update

Subject:
Invoice#928

Context:
channel = web
tenant = Tenant#7
route = invoices.update
controller = InvoiceController::update
phase = resource
origin = controller
```

Resultado:

```text
AuthorizationRequest
```

---

# 162. Gate example

Código:

```php
Gate::allows('access-admin');
```

Transformación:

```text
Principal:
User#42

Ability:
access-admin

Subject:
NONE

Context:
Current request context
```

---

# 163. CLI example

Código:

```php
Authorization::for($system)
    ->authorize(
        'execute',
        GenerateMonthlyReport::class
    );
```

Request:

```text
Principal:
SystemPrincipal:scheduler

Ability:
execute

Subject:
GenerateMonthlyReport

Context:
channel = cli
runtime = console
```

---

# 164. Queue example

```text
Principal:
ServicePrincipal:billing-worker

Ability:
charge

Subject:
Invoice#928

Context:
channel = queue
tenant = 7
job = ProcessInvoicePayment
```

---

# 165. Multi-tenant example

```text
Principal:
User#42

Ability:
update

Subject:
Customer#800

Context:
Tenant#7
```

Subject metadata:

```text
resourceTenant:
Tenant#9
```

Policy:

```text
TenantIsolationPolicy
        ↓
DENY
```

---

# 166. ABAC example

```text
Principal:
Employee#18

Ability:
view

Subject:
FinancialReport#90

Context:
region = MX
security.mfa = true
```

Policy puede considerar:

```text
principal.department
subject.classification
context.region
context.security.mfa
```

---

# 167. ReBAC example

```text
Principal:
User#42

Ability:
manage

Subject:
Project#81
```

La Policy puede consultar relaciones:

```text
User
  ↓ member of
Organization
  ↓ owns
Project
```

El modelo Request no necesita cambiar.

---

# 168. Subject-less ABAC

También puede existir:

```text
Principal:
User#42

Ability:
system.deploy

Subject:
NONE

Context:
environment = production
security.mfa = true
```

Esto demuestra que `Subject` no siempre es obligatorio.

---

# 169. Request invariants

Todo `AuthorizationRequest` deberá cumplir:

### Invariante 1

Siempre existe un Principal válido.

### Invariante 2

Siempre existe una Ability válida.

### Invariante 3

Siempre existe un `SubjectDescriptor`, incluso si representa `NONE`.

### Invariante 4

Siempre existe un AuthorizationContext.

### Invariante 5

Todos los componentes son semánticamente inmutables durante la evaluación.

---

# 170. Principal invariants

### Invariante 1

La identidad debe ser estable durante la decisión.

### Invariante 2

Anonymous es un Principal real, no un bypass.

### Invariante 3

SystemPrincipal no implica superusuario.

### Invariante 4

Impersonación conserva actor y principal efectivo.

---

# 171. Subject invariants

### Invariante 1

Authorization no carga recursos implícitamente desde database.

### Invariante 2

Un Subject no tiene que ser un modelo.

### Invariante 3

Class subjects y instance subjects son conceptos diferentes.

### Invariante 4

Subject identity puede ser null.

---

# 172. Context invariants

### Invariante 1

El Context contiene datos, no servicios.

### Invariante 2

Trusted security fields solo provienen de fuentes autorizadas.

### Invariante 3

Context final es inmutable.

### Invariante 4

Estado request-scoped nunca puede sobrevivir entre requests.

### Invariante 5

Context ausente e información falsa no son equivalentes.

---

# 173. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    ├── Core/
    │   ├── AuthorizationRequest.php
    │   ├── AuthorizationRequestFactory.php
    │   └── AuthorizationRequestId.php
    │
    ├── Ability/
    │   ├── Ability.php
    │   ├── AbilityNormalizer.php
    │   ├── AbilityAliasRegistry.php
    │   └── Exceptions/
    │
    ├── Principal/
    │   ├── PrincipalInterface.php
    │   ├── PrincipalResolver.php
    │   ├── PrincipalDescriptor.php
    │   ├── PrincipalReference.php
    │   ├── PrincipalType.php
    │   └── Principals/
    │       ├── AnonymousPrincipal.php
    │       ├── ServicePrincipal.php
    │       ├── SystemPrincipal.php
    │       └── ApiClientPrincipal.php
    │
    ├── Subject/
    │   ├── SubjectDescriptor.php
    │   ├── SubjectResolver.php
    │   ├── SubjectType.php
    │   ├── SubjectIdentityResolver.php
    │   ├── NamedSubject.php
    │   ├── VirtualSubject.php
    │   └── Resolvers/
    │
    ├── Context/
    │   ├── AuthorizationContext.php
    │   ├── AuthorizationContextBuilder.php
    │   ├── AuthorizationContextFactory.php
    │   ├── AuthorizationContextValidator.php
    │   ├── AuthorizationContextDescriptor.php
    │   ├── SecurityContext.php
    │   ├── RuntimeContext.php
    │   └── Providers/
    │       ├── AuthenticationContextProvider.php
    │       ├── TenantContextProvider.php
    │       ├── HttpContextProvider.php
    │       ├── RouteContextProvider.php
    │       ├── ControllerContextProvider.php
    │       └── RuntimeContextProvider.php
    │
    └── Exceptions/
        ├── PrincipalResolutionException.php
        ├── SubjectResolutionException.php
        ├── InvalidAbilityException.php
        ├── MissingAuthorizationContextException.php
        └── AuthorizationContextConflictException.php
```

---

# 174. Arquitectura final del modelo

```text
┌─────────────────────────────────────────┐
│         Authorization Input             │
│                                         │
│ Ability                                 │
│ Subject                                 │
│ Optional Principal                      │
│ Optional Explicit Context               │
└────────────────────┬────────────────────┘
                     ↓
          ┌─────────────────────┐
          │ Principal Resolver  │
          └──────────┬──────────┘
                     │
          ┌──────────▼──────────┐
          │ Ability Normalizer  │
          └──────────┬──────────┘
                     │
          ┌──────────▼──────────┐
          │  Subject Resolver   │
          └──────────┬──────────┘
                     │
          ┌──────────▼──────────┐
          │   Context Factory   │
          └──────────┬──────────┘
                     │
         ┌───────────▼───────────┐
         │ Context Provider Chain │
         └───────────┬───────────┘
                     │
          ┌──────────▼──────────┐
          │ Context Validation │
          └──────────┬──────────┘
                     ↓
┌─────────────────────────────────────────┐
│         AuthorizationRequest            │
│                                         │
│ Principal                               │
│ Ability                                 │
│ SubjectDescriptor                       │
│ AuthorizationContext                    │
│ Request ID                              │
└────────────────────┬────────────────────┘
                     ↓
          Authorization Planner
```

---

# 175. Filosofía del modelo

La filosofía será:

```text
Explicit identity.
Explicit action.
Generic subject.
Structured context.
Trusted security data.
Immutable request.
```

Desde el punto de vista del desarrollador:

```php
$user->can('update', $invoice);
```

Desde el punto de vista del framework:

```text
Principal:
user:42

Ability:
update

Subject:
invoice:928

Context:
tenant:7
channel:web
route:invoices.update
security:mfa
```

Todo esto se encapsula en una única representación:

```text
AuthorizationRequest
```

---

# 176. Resultado esperado

El modelo `AuthorizationRequest + Principal + Ability + Subject + AuthorizationContext` deberá constituir el lenguaje común utilizado por todo el Authorization System.

Gracias a esta abstracción, VoltStack podrá aplicar el mismo motor a:

```text
Models
Controllers
Controller Actions
Routes
Components
Commands
Jobs
Services
Tenants
Virtual Resources
System Operations
```

sin modificar la arquitectura central.

El principio definitivo será:

```text
Authorization should not care
where the request came from.

It should only need to know:

WHO
wants to do
WHAT
to WHICH SUBJECT
under WHICH CONTEXT.
```