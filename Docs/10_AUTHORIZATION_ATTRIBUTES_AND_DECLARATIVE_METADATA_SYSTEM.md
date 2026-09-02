# VoltStack Authorization System — Attributes and Declarative Metadata System

## 1. Propósito

Este documento define el sistema declarativo de autorización de VoltStack basado en **PHP Attributes, metadata compilable y descriptores normalizados**.

El objetivo es permitir expresar reglas de autorización directamente sobre:

```text
Controllers
Controller Actions
Routes
Commands
Jobs
Components
Domain Actions
Services
Resources
```

sin acoplar estos elementos al motor interno de autorización.

Ejemplo:

```php
#[Authorize('admin.access')]
final class InvoiceController
{
    #[Authorize('update', subject: 'invoice')]
    public function update(Invoice $invoice): Response
    {
        // ...
    }
}
```

La declaración anterior deberá transformarse internamente en:

```text
PHP Attribute
      ↓
Metadata Extraction
      ↓
Metadata Normalization
      ↓
Validation
      ↓
Compiled Authorization Metadata
      ↓
AuthorizationPlanTemplate
      ↓
AuthorizationPlanner
```

---

# 2. Objetivos

El sistema deberá proporcionar:

1. autorización declarativa;
2. sintaxis simple para casos comunes;
3. soporte para múltiples requisitos;
4. soporte para Policies y Gates;
5. soporte para roles y permisos;
6. selección declarativa de estrategia;
7. selección de Subject;
8. selección de fase;
9. prioridades;
10. composición e herencia;
11. reglas no reemplazables;
12. compilación;
13. validación estática;
14. introspección;
15. integración con Routing y Controllers;
16. extensibilidad.

---

# 3. Principio arquitectónico

Los Attributes deberán ser:

```text
declarations
```

y no:

```text
authorization executors
```

Por tanto:

```text
Attribute
    ↓
Metadata
    ↓
Compiler
    ↓
Descriptor
```

Nunca:

```text
Attribute
    ↓
Policy execution
```

---

# 4. Regla fundamental

Un Attribute no deberá:

```text
consultar base de datos
resolver el usuario actual
resolver Tenant
ejecutar Policies
ejecutar Gates
acceder al Request global
producir respuestas HTTP
```

Su responsabilidad es únicamente describir intención.

---

# 5. Attributes iniciales

VoltStack deberá contemplar inicialmente:

```php
#[Authorize]
#[Policy]
#[RequiresRole]
#[RequiresPermission]
#[DecisionStrategy]
```

Podrán añadirse posteriormente:

```php
#[AuthorizationPriority]
#[AuthorizationPhase]
#[NonBypassable]
#[PublicAccess]
```

si proporcionan suficiente valor.

---

# 6. Evitar proliferación de Attributes

No toda opción deberá convertirse en un Attribute independiente.

Por ejemplo:

```php
#[Authorize(
    ability: 'update',
    subject: 'invoice',
    priority: 5000
)]
```

puede ser preferible a:

```php
#[Authorize('update')]
#[AuthorizationSubject('invoice')]
#[AuthorizationPriority(5000)]
```

VoltStack favorecerá APIs declarativas compactas.

---

# 7. `#[Authorize]`

Será el Attribute principal.

Ejemplo mínimo:

```php
#[Authorize('admin.access')]
```

---

# 8. Contrato conceptual

```php
#[Attribute(
    Attribute::TARGET_CLASS |
    Attribute::TARGET_METHOD |
    Attribute::IS_REPEATABLE
)]
final readonly class Authorize
{
    public function __construct(
        public string $ability,
        public string|array|null $subject = null,
        public ?AuthorizationPhase $phase = null,
        public ?string $strategy = null,
        public ?int $priority = null,
    ) {}
}
```

La implementación final podrá utilizar enums/value objects adicionales.

---

# 9. Ability

El argumento obligatorio será normalmente:

```text
ability
```

Ejemplo:

```php
#[Authorize('invoice.update')]
```

o:

```php
#[Authorize('update', subject: 'invoice')]
```

---

# 10. Ability canonicalization

Durante compilación:

```text
invoice.update
```

deberá convertirse a una representación canónica.

No realizar normalización repetitiva durante cada request.

---

# 11. Subject implícito

Algunas abilities no requieren un recurso concreto.

Ejemplo:

```php
#[Authorize('admin.access')]
```

Puede utilizar como Subject:

```text
Controller
Route
Named Resource
Virtual Subject
null
```

según el descriptor de la Ability.

---

# 12. Subject explícito

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

`invoice` representa una referencia declarativa al argumento.

---

# 13. SubjectReference

No deberá guardarse simplemente como un string ambiguo.

Durante compilación:

```text
"invoice"
```

se transforma en:

```text
ControllerArgumentSubjectReference
```

---

# 14. SubjectReferenceInterface

Conceptualmente:

```php
interface SubjectReferenceInterface
{
    public function type(): SubjectReferenceType;
}
```

---

# 15. Tipos de Subject Reference

VoltStack podrá soportar:

```text
Controller Argument
Route Parameter
Class
Named Subject
Request Context Value
Multiple Subjects
```

---

# 16. SubjectReferenceType

```php
enum SubjectReferenceType: string
{
    case Argument = 'argument';
    case RouteParameter = 'route_parameter';
    case ClassName = 'class';
    case Named = 'named';
    case Context = 'context';
    case Multiple = 'multiple';
}
```

---

# 17. Argument Subject

```php
#[Authorize('update', subject: 'invoice')]
public function update(Invoice $invoice)
{
}
```

Compila a:

```text
ArgumentSubjectReference
name=invoice
type=Invoice
```

---

# 18. Class Subject

Para acciones como:

```text
crear Invoice
```

todavía no existe instancia.

Ejemplo:

```php
#[Authorize(
    'create',
    subject: Invoice::class
)]
public function create()
{
}
```

---

# 19. Class-level Policy

Esto permite semántica similar a:

```php
$user->can('create', Invoice::class);
```

---

# 20. Named Subject

Ejemplo:

```php
#[Authorize(
    'access',
    subject: 'admin-dashboard'
)]
```

Si `admin-dashboard` está registrado como Named Subject.

---

# 21. Ambigüedad de strings

Un string como:

```text
invoice
```

podría significar:

```text
argument
named subject
route parameter
```

VoltStack deberá evitar depender de heurísticas ambiguas.

---

# 22. Subject helpers

La API podrá ofrecer value objects:

```php
Subject::argument('invoice')
Subject::route('invoice')
Subject::named('admin-dashboard')
Subject::class(Invoice::class)
```

Sin embargo, PHP Attributes requieren argumentos compatibles con expresiones constantes.

Por ello podrá utilizarse un enum/marcador o sintaxis explícita.

---

# 23. Sintaxis declarativa propuesta

Puede utilizarse:

```php
#[Authorize(
    'update',
    subject: 'argument:invoice'
)]
```

o permitir que en Controllers:

```text
plain string
```

signifique argumento por convención.

---

# 24. Recomendación

En Controller methods:

```php
subject: 'invoice'
```

deberá significar:

```text
Controller Argument
```

por defecto.

En otros targets, la interpretación vendrá determinada por el Metadata Adapter correspondiente.

Esto mantiene la API limpia sin introducir demasiada sintaxis.

---

# 25. Validación del Subject

El compilador deberá verificar:

```php
#[Authorize('update', subject: 'invoice')]
public function update(Order $order)
```

y producir error porque:

```text
invoice argument does not exist
```

---

# 26. Error temprano

Preferir:

```text
compile-time error
```

sobre:

```text
runtime authorization failure
```

si el problema puede detectarse estáticamente.

---

# 27. Multiple Subjects

Algunas operaciones dependen de varios recursos.

Ejemplo:

```php
#[Authorize(
    'move',
    subject: ['document', 'destination']
)]
public function move(
    Document $document,
    Folder $destination
) {}
```

---

# 28. MultiSubjectReference

Durante compilación:

```text
document
destination
```

se convierte en:

```text
MultiSubjectReference
├── Document argument
└── Folder argument
```

---

# 29. Subject ordering

El orden deberá mantenerse estable cuando forme parte del contrato de la Policy.

---

# 30. Phase

`#[Authorize]` podrá especificar:

```php
phase: AuthorizationPhase::PreResolution
```

---

# 31. Phase inference

Si no se especifica:

```text
Subject requires resolved argument
    ↓
Resource
```

Si no existe Subject runtime:

```text
PreResolution
```

podrá ser inferido cuando sea seguro.

---

# 32. Prefer explicit semantics in ambiguous cases

El compilador no deberá inferir una fase que pueda cambiar la seguridad.

En caso ambiguo:

```text
compile error
```

o exigir declaración explícita.

---

# 33. PreResolution example

```php
#[Authorize(
    'admin.access',
    phase: AuthorizationPhase::PreResolution
)]
final class AdminController
{
}
```

---

# 34. Resource example

```php
#[Authorize(
    'invoice.update',
    subject: 'invoice',
    phase: AuthorizationPhase::Resource
)]
```

---

# 35. Strategy

Puede declararse directamente:

```php
#[Authorize(
    'approve',
    subject: 'invoice',
    strategy: 'unanimous'
)]
```

---

# 36. `#[DecisionStrategy]`

También podrá declararse por separado:

```php
#[DecisionStrategy('deny_overrides')]
```

Útil cuando la misma estrategia afecta varias reglas.

---

# 37. DecisionStrategy Attribute

```php
#[Attribute(
    Attribute::TARGET_CLASS |
    Attribute::TARGET_METHOD
)]
final readonly class DecisionStrategy
{
    public function __construct(
        public string $strategy,
    ) {}
}
```

---

# 38. Strategy precedence

Recomendación:

```text
Authorize.strategy
        ↓
Method DecisionStrategy
        ↓
Class DecisionStrategy
        ↓
Route metadata
        ↓
Ability strategy
        ↓
Authorization default
```

La integración con Route podrá modificar la precedencia cuando la Route sea la fuente explícita de la regla.

---

# 39. Explicit local configuration wins

Una estrategia definida directamente sobre:

```php
#[Authorize(... strategy: 'unanimous')]
```

deberá tener precedencia sobre defaults heredados.

---

# 40. Strategy validation

Durante compilación:

```php
#[DecisionStrategy('something_unknown')]
```

deberá producir:

```text
UnknownDecisionStrategyException
```

---

# 41. Priority

Podrá definirse:

```php
#[Authorize(
    'approve',
    subject: 'invoice',
    priority: 5500
)]
```

---

# 42. Meaning of priority

Esta prioridad describe la regla/evaluator generado por esa declaración.

No necesariamente reemplaza la prioridad interna de Policies críticas globales.

---

# 43. Priority safety

Metadata de aplicación no deberá poder bajar arbitrariamente una Policy marcada:

```text
nonBypassable
critical
```

por debajo de una frontera insegura.

---

# 44. `#[Policy]`

Permitirá vincular explícitamente una Policy cuando la resolución automática no sea suficiente.

Ejemplo:

```php
#[Policy(
    InvoiceApprovalPolicy::class,
    ability: 'approve',
    subject: 'invoice'
)]
```

---

# 45. Policy Attribute contract

```php
#[Attribute(
    Attribute::TARGET_CLASS |
    Attribute::TARGET_METHOD |
    Attribute::IS_REPEATABLE
)]
final readonly class Policy
{
    public function __construct(
        public string $policy,
        public ?string $ability = null,
        public string|array|null $subject = null,
        public ?int $priority = null,
    ) {}
}
```

---

# 46. Policy vs Authorize

`#[Authorize]` expresa:

```text
authorization requirement
```

`#[Policy]` expresa:

```text
specific evaluator binding
```

No son equivalentes.

---

# 47. Prefer `#[Authorize]`

En código de aplicación deberá preferirse:

```php
#[Authorize('update', subject: 'invoice')]
```

porque desacopla Controller y Policy concreta.

---

# 48. `#[Policy]` use cases

Adecuado para:

```text
specialized policies
module-local evaluator
legacy integration
explicit evaluator composition
```

---

# 49. Policy class validation

La clase deberá:

```text
exist
be registered or valid
satisfy Policy contract
support target ability
```

cuando esto pueda comprobarse.

---

# 50. No arbitrary class instantiation

El runtime nunca deberá tomar un nombre de Policy proveniente de input HTTP.

Solo metadata confiable compilada.

---

# 51. `#[RequiresRole]`

Permitirá expresar requisitos RBAC simples.

Ejemplo:

```php
#[RequiresRole('administrator')]
```

---

# 52. Multiple roles

```php
#[RequiresRole(
    ['administrator', 'finance-manager']
)]
```

requiere semántica explícita.

---

# 53. Role Match Mode

```php
enum RequirementMatch: string
{
    case Any = 'any';
    case All = 'all';
}
```

---

# 54. Role example

```php
#[RequiresRole(
    ['administrator', 'finance-manager'],
    match: RequirementMatch::Any
)]
```

---

# 55. RequiresRole contract

```php
#[Attribute(
    Attribute::TARGET_CLASS |
    Attribute::TARGET_METHOD |
    Attribute::IS_REPEATABLE
)]
final readonly class RequiresRole
{
    public function __construct(
        public string|array $roles,
        public RequirementMatch $match = RequirementMatch::All,
    ) {}
}
```

---

# 56. Default role semantics

Se recomienda:

```text
one role supplied → All/irrelevant
multiple roles → All
```

como default conservador.

Para `Any`, deberá declararse explícitamente.

---

# 57. Role Attribute normalization

No deberá ejecutarse directamente.

Se convierte en:

```text
RoleRequirementDescriptor
```

y posteriormente:

```text
RoleEvaluator
```

---

# 58. `#[RequiresPermission]`

Ejemplo:

```php
#[RequiresPermission('invoice.approve')]
```

---

# 59. Multiple permissions

```php
#[RequiresPermission(
    [
        'invoice.read',
        'invoice.approve'
    ],
    match: RequirementMatch::All
)]
```

---

# 60. Permission descriptor

Se convierte en:

```text
PermissionRequirementDescriptor
```

que el Planner transforma en un evaluator.

---

# 61. Roles y Permissions no sustituyen Policies

Ejemplo:

```php
#[RequiresPermission('invoice.update')]
#[Authorize('update', subject: 'invoice')]
```

significa:

```text
Permission
AND
Resource Policy
```

bajo la composición correspondiente.

---

# 62. Beneficio

Un usuario puede poseer:

```text
invoice.update
```

pero aun así no tener autorización sobre:

```text
Invoice belonging to another tenant
```

---

# 63. Permission Attribute no bypass

`#[RequiresPermission]` nunca deberá saltarse:

```text
TenantIsolation
Resource Policy
Critical Security
```

---

# 64. Role and Permission namespaces

Los identificadores deberán normalizarse:

```text
admin
finance.manager
invoice.approve
```

según convenciones del sistema RBAC.

---

# 65. Invalid identifiers

El compilador podrá detectar:

```text
empty permission
invalid characters
unknown permission in strict mode
```

---

# 66. Strict Registry Mode

En aplicaciones compiladas:

```text
unknown permission
```

podrá producir error.

En sistemas dinámicos podrá permitirse resolución runtime.

---

# 67. Repeatable Attributes

`#[Authorize]`, `#[Policy]`, `#[RequiresRole]` y `#[RequiresPermission]` podrán ser repeatable cuando tenga sentido.

Ejemplo:

```php
#[Authorize('admin.access')]
#[Authorize('update', subject: 'invoice')]
```

---

# 68. Default composition

Múltiples requirements independientes deberán componerse como:

```text
AND
```

por defecto.

---

# 69. Razón

La autorización declarativa debe ser conservadora.

Añadir una regla no debería accidentalmente ampliar permisos.

---

# 70. Requirement Group

Para casos OR complejos podrá existir en una versión posterior:

```php
#[AuthorizationGroup(
    mode: RequirementMatch::Any
)]
```

Pero no se recomienda introducirlo en V1 sin necesidad.

---

# 71. Alternativas OR

Preferir inicialmente:

```text
Gate
Policy
Custom Decision Strategy
```

para lógica compleja.

---

# 72. Class-level Attributes

Ejemplo:

```php
#[Authorize('admin.access')]
final class InvoiceAdminController
{
    public function index() {}

    public function show(Invoice $invoice) {}
}
```

La regla se hereda por todas las acciones.

---

# 73. Method-level Attributes

```php
#[Authorize('admin.access')]
final class InvoiceController
{
    #[Authorize('delete', subject: 'invoice')]
    public function destroy(Invoice $invoice) {}
}
```

Resultado:

```text
admin.access
AND
invoice.delete
```

---

# 74. Additive inheritance

Este será el comportamiento default.

---

# 75. Metadata Layers

Para Controllers:

```text
Framework Global
        ↓
Route
        ↓
Inherited Controller Class
        ↓
Controller Class
        ↓
Controller Method
```

Estas capas se normalizan antes de planning.

---

# 76. Metadata no siempre override

Una regla inferior normalmente se:

```text
append
```

no se reemplaza.

---

# 77. CompositionMode

Podrá existir:

```php
enum AuthorizationCompositionMode: string
{
    case Append = 'append';
    case Replace = 'replace';
    case Inherit = 'inherit';
    case Disable = 'disable';
}
```

---

# 78. Default

```text
Append
```

---

# 79. Replace

Ejemplo conceptual:

```php
#[AuthorizationMetadata(
    mode: AuthorizationCompositionMode::Replace
)]
```

Sin embargo, introducir un Attribute genérico adicional puede no ser necesario en V1.

---

# 80. Replace restrictions

`Replace` solo deberá afectar reglas reemplazables.

No podrá eliminar:

```text
nonBypassable
critical global evaluators
```

---

# 81. Disable

Una acción podría necesitar ser pública:

```text
login
health endpoint
public landing
```

---

# 82. `#[PublicAccess]`

Para estos casos puede ser preferible:

```php
#[PublicAccess]
public function login()
{
}
```

que:

```text
Disable all authorization
```

---

# 83. PublicAccess semantics

`PublicAccess` significa:

```text
no application-level authenticated authorization requirement
```

No significa:

```text
disable platform security
disable rate limiting
disable tenant boundary
disable CSRF
```

---

# 84. PublicAccess no es AllowAll

Es metadata de integración, no un voto:

```text
GRANT
```

---

# 85. Public route

Ejemplo:

```php
#[PublicAccess]
public function status(): Response
{
}
```

podrá evitar crear requisitos de usuario autenticado.

---

# 86. PublicAccess conflict

Esto deberá fallar:

```php
#[PublicAccess]
#[RequiresRole('admin')]
```

porque las declaraciones son contradictorias.

---

# 87. Explicit conflict detection

El compiler deberá detectar metadata incompatible.

---

# 88. Controller inheritance

Si:

```php
#[Authorize('admin.access')]
class BaseAdminController
{
}
```

y:

```php
final class UserController extends BaseAdminController
{
}
```

`admin.access` deberá heredarse si la metadata está marcada como heredable.

---

# 89. PHP Attribute inheritance

VoltStack no deberá depender exclusivamente del comportamiento nativo de Reflection para herencia.

El `AuthorizationMetadataCompiler` deberá definir sus propias reglas deterministas.

---

# 90. Interface metadata

Una interfaz podrá declarar autorización solo si VoltStack decide soportarlo explícitamente.

Ejemplo potencial:

```php
#[RequiresPermission('financial.operations')]
interface FinancialOperation
{
}
```

---

# 91. Recomendación V1

Soportar:

```text
Controller parent classes
Controller class
Controller method
Route
```

Primero.

Dejar interface/trait inheritance para una fase posterior si complica la resolución.

---

# 92. Trait metadata

Los Attributes sobre métodos de traits pueden resultar ambiguos tras composición.

Si se soportan:

```text
compiled effective method metadata
```

deberá ser la fuente de verdad.

---

# 93. Metadata Source

Cada descriptor deberá conservar opcionalmente su origen.

Ejemplo:

```text
ControllerClass
ControllerMethod
Route
InheritedClass
```

---

# 94. AuthorizationMetadataSource

```php
enum AuthorizationMetadataSource: string
{
    case Route = 'route';
    case ControllerClass = 'controller_class';
    case ControllerMethod = 'controller_method';
    case InheritedClass = 'inherited_class';
    case Command = 'command';
    case Job = 'job';
    case Component = 'component';
    case Programmatic = 'programmatic';
}
```

---

# 95. Source location

En desarrollo podrá conservar:

```text
file
line
class
method
```

para debugging.

---

# 96. Production optimization

En producción podrán omitirse:

```text
file paths
line numbers
```

si no son necesarios.

---

# 97. DeclarativeMetadataDescriptor

Toda metadata deberá normalizarse.

```php
final readonly class AuthorizationMetadataDescriptor
{
    public function __construct(
        public array $requirements,
        public ?string $strategy,
        public AuthorizationCompositionMode $composition,
        public AuthorizationMetadataSource $source,
    ) {}
}
```

---

# 98. AuthorizationRequirementDescriptor

Contrato base:

```php
interface AuthorizationRequirementDescriptorInterface
{
    public function type(): AuthorizationRequirementType;
}
```

---

# 99. Requirement types

```php
enum AuthorizationRequirementType: string
{
    case Ability = 'ability';
    case Policy = 'policy';
    case Role = 'role';
    case Permission = 'permission';
}
```

---

# 100. AbilityRequirementDescriptor

```php
final readonly class AbilityRequirementDescriptor
{
    public function __construct(
        public string $ability,
        public ?SubjectReferenceInterface $subject,
        public AuthorizationPhase $phase,
        public ?string $strategy,
        public ?int $priority,
    ) {}
}
```

---

# 101. PolicyRequirementDescriptor

Contendrá:

```text
Policy ID/Class
Ability
SubjectReference
Priority
Phase
```

---

# 102. RoleRequirementDescriptor

Contendrá:

```text
roles
match mode
phase
priority
```

---

# 103. PermissionRequirementDescriptor

Contendrá:

```text
permissions
match mode
phase
priority
```

---

# 104. Metadata extraction

Durante compilación:

```text
Reflection
    ↓
PHP Attributes
    ↓
Raw Metadata
```

---

# 105. Reflection boundary

Reflection deberá concentrarse en:

```text
development
boot
compile
cache generation
```

No en cada request.

---

# 106. MetadataExtractor

Contrato conceptual:

```php
interface AuthorizationMetadataExtractorInterface
{
    public function extract(
        ReflectionClass|ReflectionMethod $target
    ): RawAuthorizationMetadata;
}
```

---

# 107. Raw metadata

Es una representación temporal.

No deberá llegar al hot path.

---

# 108. MetadataNormalizer

```text
Raw Attributes
      ↓
AuthorizationMetadataNormalizer
      ↓
Descriptors
```

---

# 109. Normalization responsibilities

Deberá:

- canonicalizar Ability;
- normalizar SubjectReference;
- resolver phase;
- normalizar roles;
- normalizar permissions;
- resolver defaults;
- resolver strategy aliases;
- validar priority;
- registrar source.

---

# 110. MetadataValidator

Después:

```text
Descriptors
    ↓
MetadataValidator
```

---

# 111. Validation responsibilities

Validará:

```text
subject exists
strategy exists
Policy exists
Policy contract valid
phase compatible
priority valid
no contradictory attributes
no illegal bypass
```

---

# 112. MetadataMerger

Después:

```text
Parent Metadata
Route Metadata
Class Metadata
Method Metadata
        ↓
MetadataMerger
```

---

# 113. Merger responsibilities

Resolverá:

```text
Append
Replace
Inherit
Disable
```

y restricciones de seguridad.

---

# 114. MetadataCompiler

Finalmente:

```text
Effective Metadata
      ↓
AuthorizationMetadataCompiler
      ↓
CompiledAuthorizationMetadata
```

---

# 115. Compiled metadata

Debe estar preparada para:

```text
AuthorizationPlanner
```

sin Reflection.

---

# 116. Pipeline de compilación

```text
PHP Source
    ↓
Reflection / Attribute Extraction
    ↓
RawAuthorizationMetadata
    ↓
Normalization
    ↓
Validation
    ↓
Inheritance / Composition
    ↓
Security Validation
    ↓
Compilation
    ↓
CompiledAuthorizationMetadata
```

---

# 117. CompiledAuthorizationMetadata

Ejemplo:

```php
final readonly class CompiledAuthorizationMetadata
{
    public function __construct(
        public string $targetId,
        public array $requirements,
        public ?string $strategyId,
        public bool $publicAccess,
    ) {}
}
```

---

# 118. Target ID

Ejemplos:

```text
controller:App\Controller\InvoiceController
controller_method:App\Controller\InvoiceController::update
route:invoice.update
command:invoice:approve
job:GenerateInvoiceJob
```

---

# 119. Compiled metadata registry

Podrá existir:

```text
CompiledAuthorizationMetadataRegistry
```

indexado por Target ID.

---

# 120. Runtime lookup

En Controller dispatch:

```text
ControllerMethod ID
      ↓
Compiled Metadata Registry
      ↓
Authorization metadata
```

O(1) idealmente.

---

# 121. Metadata cache

Podrá serializarse como PHP arrays optimizados.

Ejemplo conceptual:

```php
return [
    'controller_method:App\Controller\InvoiceController::update' => [
        // compiled descriptor data
    ],
];
```

---

# 122. Opcache friendliness

Los caches generados deberán favorecer:

```text
PHP arrays
immutable value objects
preloaded registries
```

y evitar serialización compleja innecesaria.

---

# 123. FrankenPHP

En worker mode:

```text
Compiled Metadata Registry
```

podrá vivir durante toda la vida del worker.

---

# 124. Restricción crítica

Nunca almacenar en metadata compartida:

```text
current User
current Tenant
current Request
current Subject instance
```

---

# 125. Runtime Binding

Los SubjectReferences compilados se enlazan posteriormente:

```text
Compiled Metadata
      ↓
Runtime Subject Resolver
      ↓
AuthorizationRequest
```

---

# 126. Controller integration

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

Compilación:

```text
Ability:
update

SubjectReference:
ControllerArgument(invoice)

Expected Type:
Invoice

Phase:
Resource
```

Runtime:

```text
invoice argument
    ↓
Invoice#928
```

---

# 127. Route integration

Ejemplo conceptual:

```php
Route::put('/invoices/{invoice}', ...)
    ->authorize('update', 'invoice');
```

La API fluida de Routing deberá producir el mismo descriptor que:

```php
#[Authorize('update', subject: 'invoice')]
```

---

# 128. Unificación

Este principio es esencial:

```text
PHP Attribute
Route DSL
Controller Configuration
Programmatic Metadata
```

deben terminar en:

```text
AuthorizationRequirementDescriptor
```

---

# 129. No motores paralelos

No deberá existir:

```text
AttributeAuthorizationEngine
RouteAuthorizationEngine
ControllerAuthorizationEngine
```

Solo:

```text
Authorization Core
```

---

# 130. Route Attribute

Si VoltStack soporta Controllers attribute-based:

```php
#[Route('/invoices/{invoice}', methods: ['PUT'])]
#[Authorize('update', subject: 'invoice')]
```

ambos sistemas deberán compilar metadata de forma coordinada.

---

# 131. Command integration

```php
#[RequiresPermission('system.cache.clear')]
final class ClearCacheCommand
{
}
```

---

# 132. Command Subject

Podrá ser:

```text
Command class
```

como Virtual Subject.

---

# 133. Job integration

```php
#[Authorize(
    'process',
    subject: 'invoice'
)]
final class ProcessInvoiceJob
{
}
```

La integración deberá definir cómo resolver `invoice` desde propiedades/serialized references del Job.

---

# 134. Job security caveat

Un Job no deberá confiar simplemente en autorización histórica del momento en que fue encolado si la operación requiere autorización actual.

Esto deberá definirse en el lifecycle de autorización.

---

# 135. Component integration

Ejemplo:

```php
#[RequiresPermission('invoice.read')]
final class InvoiceTable extends Component
{
}
```

---

# 136. Component action

```php
#[Authorize('delete', subject: 'invoice')]
public function delete(Invoice $invoice)
{
}
```

Podrá reutilizar exactamente el mismo sistema.

---

# 137. Domain Action

```php
#[Authorize(
    'approve',
    subject: 'invoice'
)]
final class ApproveInvoice
{
}
```

La invocación a través del Action Dispatcher podrá activar autorización.

---

# 138. Direct method call caveat

PHP no intercepta automáticamente:

```php
$action->execute();
```

solo porque la clase tenga un Attribute.

La autorización ocurre cuando el objeto se invoca mediante una integración consciente del framework.

---

# 139. Regla importante

Los Attributes son metadata.

No son AOP mágico.

---

# 140. Framework invocation boundaries

VoltStack deberá documentar qué dispatchers respetan authorization metadata:

```text
HTTP Controller Dispatcher
Route Dispatcher
Action Dispatcher
Command Dispatcher
Job Dispatcher
Component Action Dispatcher
```

---

# 141. Services

No se recomienda aplicar Attributes arbitrariamente a cualquier método de cualquier Service esperando interceptación automática.

---

# 142. Service authorization

Preferir:

```text
explicit AuthorizationManager call
```

o invocación mediante un Dispatcher registrado.

---

# 143. `#[Policy]` on Controller

Sí será posible.

Ejemplo:

```php
#[Policy(AdminControllerPolicy::class)]
final class AdminController
{
}
```

La Policy puede evaluar:

```text
controller-level access
```

---

# 144. Controller Policy Subject

El Subject podrá ser:

```text
Controller class
Controller action descriptor
```

sin necesidad de una entidad ORM.

---

# 145. Controller action policy

Ejemplo:

```php
#[Authorize(
    'execute',
    subject: InvoiceController::class
)]
```

Esto confirma que VoltStack Policies no estarán limitadas a Models.

---

# 146. Route Policies

También podrán aplicarse sobre:

```text
RouteDescriptor
```

si se necesita.

---

# 147. Named Operation Subjects

VoltStack podrá registrar:

```text
system.settings
admin.dashboard
billing.console
```

como Subjects virtuales.

---

# 148. Declarative security on non-model resources

Esto permitirá proteger:

```text
Controllers
Pages
Dashboards
Exports
Commands
System Operations
Feature Areas
```

con la misma infraestructura.

---

# 149. Attribute constructors

Deberán ser simples y libres de efectos secundarios.

Correcto:

```php
public function __construct(
    public string $ability
) {}
```

Incorrecto:

```php
public function __construct(
    AuthorizationManager $manager
) {}
```

---

# 150. Dependency Injection

No se inyectarán servicios en Attributes.

Los servicios se utilizan durante:

```text
compilation
planning
execution
```

---

# 151. Attribute immutability

Todos los Attributes deberán ser:

```php
final readonly
```

cuando sea práctico.

---

# 152. Attribute namespace

Propuesta:

```text
VoltStack\Quantum\Authorization\Attributes
```

---

# 153. Directory Structure

```text
Quantum/
└── Authorization/
    ├── Attributes/
    │   ├── Authorize.php
    │   ├── Policy.php
    │   ├── RequiresRole.php
    │   ├── RequiresPermission.php
    │   ├── DecisionStrategy.php
    │   └── PublicAccess.php
    │
    ├── Metadata/
    │   ├── Contracts/
    │   │   ├── AuthorizationMetadataExtractorInterface.php
    │   │   └── AuthorizationRequirementDescriptorInterface.php
    │   │
    │   ├── RawAuthorizationMetadata.php
    │   ├── AuthorizationMetadataDescriptor.php
    │   ├── AuthorizationMetadataSource.php
    │   ├── AuthorizationCompositionMode.php
    │   ├── AuthorizationRequirementType.php
    │   ├── AbilityRequirementDescriptor.php
    │   ├── PolicyRequirementDescriptor.php
    │   ├── RoleRequirementDescriptor.php
    │   ├── PermissionRequirementDescriptor.php
    │   ├── CompiledAuthorizationMetadata.php
    │   └── CompiledAuthorizationMetadataRegistry.php
    │
    ├── Subjects/
    │   └── References/
    │       ├── SubjectReferenceInterface.php
    │       ├── SubjectReferenceType.php
    │       ├── ArgumentSubjectReference.php
    │       ├── RouteParameterSubjectReference.php
    │       ├── ClassSubjectReference.php
    │       ├── NamedSubjectReference.php
    │       ├── ContextSubjectReference.php
    │       └── MultiSubjectReference.php
    │
    ├── Compilation/
    │   ├── AuthorizationMetadataExtractor.php
    │   ├── AuthorizationMetadataNormalizer.php
    │   ├── AuthorizationMetadataValidator.php
    │   ├── AuthorizationMetadataMerger.php
    │   └── AuthorizationMetadataCompiler.php
    │
    └── Exceptions/
        ├── AuthorizationMetadataException.php
        ├── InvalidAuthorizationAttributeException.php
        ├── InvalidAuthorizationSubjectReferenceException.php
        ├── AuthorizationMetadataConflictException.php
        └── AuthorizationMetadataCompilationException.php
```

---

# 154. Namespace separation

Attributes no deberán importar:

```text
PolicyDispatcher
DecisionManager
HTTP Response
Database
```

Esto mantiene la capa declarativa limpia.

---

# 155. Metadata extensibility

Paquetes externos podrán añadir metadata declarativa.

Ejemplo:

```php
#[RequiresSubscription('enterprise')]
```

---

# 156. Custom Attribute Adapter

En vez de hacer que el Core conozca cada Attribute:

```text
Custom Attribute
      ↓
AuthorizationMetadataAdapter
      ↓
RequirementDescriptor
```

---

# 157. Adapter contract

```php
interface AuthorizationAttributeAdapterInterface
{
    public function supports(object $attribute): bool;

    public function normalize(
        object $attribute,
        AuthorizationMetadataContext $context
    ): iterable;
}
```

---

# 158. Built-in adapters

Ejemplo:

```text
AuthorizeAttributeAdapter
PolicyAttributeAdapter
RoleAttributeAdapter
PermissionAttributeAdapter
```

---

# 159. Extensibility without planner modification

Un paquete podrá registrar:

```text
SubscriptionAttributeAdapter
```

sin modificar:

```text
AuthorizationPlanner
DecisionManager
```

---

# 160. Metadata adapter security

Los adapters serán código confiable registrado durante bootstrap.

---

# 161. Unknown Authorization Attribute

Un Attribute no registrado simplemente no pertenece al sistema.

Pero un Attribute declarado explícitamente como extensión de autorización y sin adapter deberá producir error de compilación.

---

# 162. Declarative metadata version

El cache compilado podrá incluir:

```text
metadata_schema_version
```

---

# 163. Cache invalidation

Cambios en:

```text
Attributes
Metadata Adapters
Authorization configuration
Policies
Abilities
Strategies
```

deberán invalidar metadata compilada cuando afecten su estructura.

---

# 164. Attribute aliases

No se recomienda crear múltiples Attributes equivalentes:

```text
#[Can]
#[Allowed]
#[Permit]
```

`#[Authorize]` deberá ser la API canónica.

---

# 165. Laravel familiarity

La experiencia buscada será tan sencilla como:

```php
$this->authorize('update', $invoice);
```

pero declarativamente:

```php
#[Authorize('update', subject: 'invoice')]
```

---

# 166. Symfony familiarity

A la vez podrá sentirse natural para desarrolladores acostumbrados a:

```text
IsGranted
Security Attributes
Voters
```

sin copiar literalmente su arquitectura.

---

# 167. VoltStack differentiation

VoltStack integrará esa metadata con:

```text
AuthorizationPlanner
Policy Pipeline
Decision Strategies
Controller Compiler
Route Compiler
FrankenPHP Runtime
```

---

# 168. Attribute compilation example

Fuente:

```php
#[Authorize('admin.access')]
#[DecisionStrategy('deny_overrides')]
final class InvoiceController
{
    #[RequiresPermission('invoice.update')]
    #[Authorize('update', subject: 'invoice')]
    public function update(Invoice $invoice): Response
    {
    }
}
```

---

# 169. Extracted metadata

```text
Class:
  Authorize(admin.access)
  DecisionStrategy(deny_overrides)

Method:
  RequiresPermission(invoice.update)
  Authorize(update, invoice)
```

---

# 170. Normalized metadata

```text
Requirement #1
type=ABILITY
ability=admin.access
phase=PRE_RESOLUTION

Requirement #2
type=PERMISSION
permission=invoice.update

Requirement #3
type=ABILITY
ability=update
subject=Argument(invoice)
subjectType=Invoice
phase=RESOURCE

Strategy:
deny_overrides
```

---

# 171. Compiled lifecycle metadata

```text
PRE_RESOLUTION

admin.access
    ↓
AuthorizationPlanner
```

Después:

```text
RESOURCE

invoice.update permission
+
update Invoice#928
    ↓
AuthorizationPlanner
```

---

# 172. Execution

El Attribute ya no participa.

El runtime trabaja únicamente con:

```text
CompiledAuthorizationMetadata
```

---

# 173. Attribute object lifetime

Idealmente los objetos Attribute solo existen durante compilación.

---

# 174. No Reflection request cost

Esto es especialmente importante bajo:

```text
FrankenPHP
high concurrency
long-running workers
```

---

# 175. Controller security example

```php
#[Authorize('admin.access')]
#[RequiresRole('administrator')]
final class UserAdminController
{
    #[Authorize('delete', subject: 'user')]
    public function destroy(User $user): Response
    {
    }
}
```

Semántica:

```text
admin.access
AND
role administrator
AND
user.delete
```

---

# 176. Policy example

```php
#[Policy(
    SensitiveUserDeletionPolicy::class,
    ability: 'delete',
    subject: 'user'
)]
```

añade un evaluator específico.

---

# 177. Resulting pipeline

Podría resultar:

```text
Platform Security
Tenant Isolation
Role Requirement
Permission Requirement
UserPolicy
SensitiveUserDeletionPolicy
```

dependiendo del Planner.

---

# 178. Attribute order

El orden textual de Attributes no deberá determinar por sí mismo el orden de ejecución.

---

# 179. Priority is explicit

Esto:

```php
#[RequiresPermission('invoice.update')]
#[Authorize('update', subject: 'invoice')]
```

no significa necesariamente que Permission se ejecute primero porque aparezca primero.

El Planner usa prioridades.

---

# 180. Determinism

Cambiar:

```php
#[A]
#[B]
```

a:

```php
#[B]
#[A]
```

no deberá alterar semántica salvo que ambos tengan explícitamente la misma prioridad y el sistema documente el orden de declaración como tie-breaker.

---

# 181. Recomendación

Usar un tie-breaker compilado estable independiente del orden textual cuando sea posible.

---

# 182. Security attribute conflicts

Ejemplos a detectar:

```text
PublicAccess + RequiresRole
PublicAccess + RequiresPermission
PublicAccess + mandatory Authorize
Resource phase + unresolved Subject
Unknown Strategy
NonBypassable + Disable
```

---

# 183. Duplicate requirements

```php
#[RequiresPermission('invoice.update')]
#[RequiresPermission('invoice.update')]
```

deberá deduplicarse durante compilación cuando sean semánticamente idénticos.

---

# 184. Duplicate with different semantics

```php
#[RequiresRole(
    ['admin', 'manager'],
    match: RequirementMatch::All
)]

#[RequiresRole(
    ['admin', 'manager'],
    match: RequirementMatch::Any
)]
```

no deberá deduplicarse.

---

# 185. Metadata identity

Cada requirement tendrá una identidad canónica.

Ejemplo:

```text
permission:invoice.update:all
```

---

# 186. Error messages

Deben ser accionables.

Incorrecto:

```text
Invalid authorization metadata.
```

Correcto:

```text
Authorization subject "invoice" declared on
InvoiceController::update() cannot be resolved.

Available controller arguments:
- order: App\Domain\Order
```

---

# 187. Source-aware errors

El compiler deberá conocer:

```text
class
method
attribute
source
```

para producir diagnósticos claros.

---

# 188. Development tooling

Podrá existir:

```text
volt authorization:metadata
```

---

# 189. Example output

```text
Target:
InvoiceController::update

Inherited Requirements:
✓ admin.access

Method Requirements:
✓ permission invoice.update
✓ ability update
  subject: invoice
  type: Invoice

Strategy:
deny_overrides

Phases:
PRE_RESOLUTION
RESOURCE
```

---

# 190. Explain inheritance

Comando:

```text
volt authorization:metadata --explain
```

podrá mostrar:

```text
admin.access
source:
InvoiceController class

invoice.update
source:
update() method

strategy deny_overrides
source:
InvoiceController class
```

---

# 191. Linting

`volt authorization:lint` deberá revisar metadata declarativa.

---

# 192. Lint checks

Entre otros:

```text
unknown ability
unknown strategy
unknown permission
missing subject
invalid subject
phase mismatch
unsafe replacement
public/private conflict
duplicate redundant metadata
```

---

# 193. Static analysis integration

En el futuro podrán ofrecerse plugins para:

```text
PHPStan
Psalm
IDE/LSP
```

---

# 194. PHPStan example

Podría detectar:

```php
#[Authorize('update', subject: 'invoice')]
public function update(Order $order)
```

sin ejecutar la aplicación.

---

# 195. IDE support

El tooling podrá ofrecer autocompletado de:

```text
Abilities
Permissions
Strategies
Named Subjects
```

---

# 196. Attribute constants

Para evitar strings:

```php
#[Authorize(InvoiceAbilities::UPDATE, subject: 'invoice')]
```

podrá utilizarse.

---

# 197. Enum abilities

Si PHP y la arquitectura lo permiten cómodamente:

```php
#[Authorize(InvoiceAbility::Update, subject: 'invoice')]
```

podrá normalizarse a string canónico.

---

# 198. No requirement to use enums

Strings seguirán siendo una API de primera clase.

---

# 199. Testing metadata

Los tests podrán inspeccionar metadata compilada.

```php
$metadata = $registry->forMethod(
    InvoiceController::class,
    'update'
);
```

---

# 200. Testing without HTTP

Esto permitirá probar:

```text
inheritance
subject resolution
strategy selection
requirement composition
```

sin levantar Kernel HTTP.

---

# 201. Metadata snapshot tests

Para módulos grandes podrán utilizarse snapshots estructurales.

---

# 202. Security regression tests

Ejemplo:

```text
InvoiceController::destroy
must always include:
TenantIsolation
invoice.delete
```

---

# 203. Mutation tests

Eliminar accidentalmente:

```php
#[Authorize('delete', subject: 'invoice')]
```

debería hacer fallar tests de seguridad del módulo.

---

# 204. Compile tests

Todos los Controllers registrados deberán compilar authorization metadata sin errores.

---

# 205. Attribute invariants

### Invariante 1

Attributes solo declaran metadata.

### Invariante 2

Attributes no ejecutan autorización.

### Invariante 3

Attributes no contienen estado runtime.

### Invariante 4

Attributes no reciben servicios mediante DI.

### Invariante 5

Toda metadata debe normalizarse antes de llegar al Planner.

---

# 206. Subject invariants

### Invariante 1

Todo Subject declarativo debe resolverse inequívocamente.

### Invariante 2

Un Subject requerido debe existir antes de su fase de autorización.

### Invariante 3

Las referencias compiladas no contienen instancias runtime.

### Invariante 4

Multi-subject mantiene orden determinista.

---

# 207. Composition invariants

### Invariante 1

Múltiples requisitos son aditivos por defecto.

### Invariante 2

Agregar un requisito no amplía acceso implícitamente.

### Invariante 3

Replace y Disable requieren intención explícita.

### Invariante 4

Non-bypassable requirements no pueden eliminarse mediante metadata ordinaria.

---

# 208. Compilation invariants

### Invariante 1

Reflection queda fuera del hot path.

### Invariante 2

Metadata inválida falla lo antes posible.

### Invariante 3

El resultado compilado es determinista.

### Invariante 4

El cache compilado no contiene Principal, Tenant ni Subject runtime.

### Invariante 5

El Planner consume metadata normalizada, no Attributes.

---

# 209. Runtime invariants

### Invariante 1

Metadata compilada puede compartirse entre requests.

### Invariante 2

Runtime Subject binding es request-scoped.

### Invariante 3

No existe fuga de estado bajo FrankenPHP.

### Invariante 4

El mismo sistema funciona en HTTP, CLI, Jobs y Components mediante adapters de integración.

---

# 210. Security invariants

### Invariante 1

Metadata contradictoria debe rechazarse.

### Invariante 2

Unknown strategy nunca degrada a una estrategia permisiva.

### Invariante 3

Missing Subject nunca concede acceso.

### Invariante 4

Compilation failure nunca produce bypass.

### Invariante 5

`PublicAccess` no desactiva fronteras globales de seguridad.

---

# 211. Arquitectura final

```text
                 PHP Source
                     │
                     ↓
              PHP Attributes
                     │
                     ↓
          Metadata Attribute Adapters
                     │
                     ↓
            Raw Authorization Metadata
                     │
                     ↓
                Normalizer
                     │
                     ↓
                 Validator
                     │
                     ↓
          Inheritance / Composition
                     │
                     ↓
             Metadata Compiler
                     │
                     ↓
       CompiledAuthorizationMetadata
                     │
          ┌──────────┴──────────┐
          ↓                     ↓
 Controller Integration    Route Integration
          │                     │
          └──────────┬──────────┘
                     ↓
             Runtime Binding
                     │
                     ↓
           AuthorizationRequest
                     │
                     ↓
          AuthorizationPlanner
                     │
                     ↓
           AuthorizationPlan
                     │
                     ↓
                Execution
```

---

# 212. Flujo Controller completo

Código:

```php
#[Authorize(
    'admin.access',
    phase: AuthorizationPhase::PreResolution
)]
#[DecisionStrategy('deny_overrides')]
final class InvoiceController
{
    #[RequiresPermission('invoice.approve')]
    #[Authorize('approve', subject: 'invoice')]
    public function approve(
        Invoice $invoice
    ): Response {
        // ...
    }
}
```

Compilación:

```text
InvoiceController
│
├── Class Requirement
│   └── admin.access
│       phase=PRE_RESOLUTION
│
├── Strategy
│   └── deny_overrides
│
└── approve()
    │
    ├── Permission
    │   └── invoice.approve
    │
    └── Ability
        └── approve
            subject=Argument(invoice)
            type=Invoice
            phase=RESOURCE
```

---

# 213. Runtime PreResolution

```text
HTTP Request
    ↓
Route Match
    ↓
Controller Metadata Lookup
    ↓
admin.access
    ↓
AuthorizationPlanner
    ↓
Security Evaluators
Gate/Policy
    ↓
DecisionManager
```

Si:

```text
DENY
```

termina antes del model binding.

---

# 214. Runtime Resource Phase

Si PreResolution concede:

```text
Resolve Invoice#928
       ↓
Bind SubjectReference(invoice)
       ↓
Permission:
invoice.approve

Ability:
approve Invoice#928
       ↓
AuthorizationPlanner
       ↓
Policy Pipeline
       ↓
DecisionManager
```

---

# 215. Resultado

Solo si ambas fases producen:

```text
GRANT
```

se ejecuta:

```php
InvoiceController::approve()
```

---

# 216. Filosofía del sistema

La filosofía del sistema declarativo será:

```text
Declare intent close to the protected operation.

Compile declarations into normalized metadata.

Keep Attributes free of runtime behavior.

Resolve Subjects explicitly.

Compose requirements conservatively.

Validate security configuration early.

Execute everything through one Authorization Core.
```

---

# 217. Resultado esperado

El `Authorization Attributes and Declarative Metadata System` permitirá escribir autorización sencilla:

```php
#[Authorize('update', subject: 'invoice')]
```

o combinar reglas empresariales:

```php
#[RequiresRole('finance-manager')]
#[RequiresPermission('invoice.approve')]
#[Authorize('approve', subject: 'invoice')]
#[DecisionStrategy('deny_overrides')]
```

sin crear motores separados.

Toda declaración terminará convergiendo en:

```text
Declarative Metadata
        ↓
CompiledAuthorizationMetadata
        ↓
AuthorizationPlanner
        ↓
AuthorizationPlan
        ↓
Policy Pipeline
        ↓
DecisionManager
```

El principio definitivo será:

```text
Attributes describe authorization.

Metadata gives those declarations structure.

Compilation validates and optimizes them.

The Planner turns them into executable plans.

The Authorization Core remains the only authority
that decides whether an operation is allowed.
```

Con esta arquitectura, las Policies de VoltStack podrán aplicarse de manera uniforme no solo sobre Models, sino también sobre **Controllers, acciones, rutas, comandos, jobs, componentes y operaciones de dominio**, manteniendo una API declarativa sencilla y un motor interno mucho más potente.