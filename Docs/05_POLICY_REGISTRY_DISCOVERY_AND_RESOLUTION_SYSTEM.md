# VoltStack Authorization System — Policy Registry, Discovery and Resolution System

## 1. Propósito

Este documento define el subsistema responsable de **registrar, descubrir, compilar, almacenar y resolver Policies** dentro del Authorization System de VoltStack.

Sus responsabilidades principales serán:

```text
Policy Registration
Policy Discovery
Policy Metadata Extraction
Policy Compilation
Policy Registry
Policy Resolution
Policy Precedence
Policy Conflict Detection
Policy Cache
```

El objetivo es garantizar que una solicitud como:

```php
$user->can('update', $invoice);
```

pueda transformarse eficientemente en:

```text
Invoice
   ↓
InvoicePolicy
   ↓
update()
```

sin realizar búsquedas costosas o ambiguas durante cada request.

El sistema deberá soportar:

- registro explícito;
- convención de nombres;
- atributos PHP;
- múltiples Policies por Subject;
- Policies sobre interfaces;
- herencia;
- Policies globales;
- Policies de Controller;
- Policies de Controller Action;
- Policies sobre rutas;
- Policies sobre Commands y Jobs;
- Policies sobre recursos virtuales;
- metadata compilada;
- resolución determinista;
- detección de conflictos.

---

# 2. Principio arquitectónico

La resolución de Policies seguirá:

```text
Discover once.
Compile once.
Resolve cheaply.
Execute predictably.
```

En desarrollo podrá existir descubrimiento dinámico.

En producción, el hot path deberá depender principalmente de:

```text
Compiled Policy Metadata
        +
PolicyRegistry
```

y no de:

```text
Filesystem scanning
Reflection scanning
Convention lookup
Attribute discovery
```

en cada request.

---

# 3. Componentes principales

El subsistema estará compuesto por:

```text
PolicyRegistry
PolicyRegistryBuilder
PolicyDiscovery
PolicyMetadataExtractor
PolicyMetadataCompiler
PolicyResolver
PolicyResolutionStrategy
PolicyDescriptor
PolicyMethodDescriptor
PolicyMapping
PolicySource
PolicyCache
PolicyConflictDetector
```

---

# 4. Flujo general

```text
Application Classes
        ↓
Policy Discovery
        ↓
Metadata Extraction
        ↓
Validation
        ↓
Conflict Detection
        ↓
Compilation
        ↓
PolicyRegistry
        ↓
Authorization Request
        ↓
PolicyResolver
        ↓
Applicable PolicyDescriptors
        ↓
Authorization Planner
```

---

# 5. PolicyRegistry

`PolicyRegistry` será la representación runtime de todas las Policies conocidas.

Conceptualmente:

```php
interface PolicyRegistryInterface
{
    public function policiesFor(
        SubjectDescriptor $subject
    ): array;

    public function globalPolicies(): array;

    public function hasPoliciesFor(
        SubjectDescriptor $subject
    ): bool;
}
```

El Registry no ejecutará Policies.

Su responsabilidad será responder:

```text
¿Qué Policies podrían aplicar a este Subject?
```

---

# 6. Registry inmutable

Después del bootstrap productivo, el Registry deberá ser preferentemente:

```text
immutable
```

Esto mejora:

- rendimiento;
- seguridad;
- predictibilidad;
- compatibilidad con FrankenPHP;
- concurrencia.

No deberán registrarse Policies arbitrariamente durante una request.

---

# 7. PolicyRegistryBuilder

Durante bootstrap o compilación podrá utilizarse:

```text
PolicyRegistryBuilder
```

para incorporar metadata desde distintas fuentes.

Ejemplo:

```text
Explicit registrations
        ↓
Attributes
        ↓
Convention discovery
        ↓
Package policies
        ↓
Framework policies
        ↓
Registry Builder
        ↓
Immutable PolicyRegistry
```

---

# 8. Policy Sources

Toda Policy deberá conocer conceptualmente su fuente de registro.

```php
enum PolicySource: string
{
    case Explicit = 'explicit';
    case Attribute = 'attribute';
    case Convention = 'convention';
    case Framework = 'framework';
    case Package = 'package';
    case Compiled = 'compiled';
}
```

Esto será útil para:

- precedencia;
- debugging;
- conflictos;
- tooling.

---

# 9. Registro explícito

La forma más fuerte de registro será explícita.

Ejemplo:

```php
Authorization::policy(
    Invoice::class,
    InvoicePolicy::class,
);
```

o:

```php
$registry->register(
    subject: Invoice::class,
    policy: InvoicePolicy::class,
);
```

---

# 10. Registro mediante configuración

También podrá declararse:

```php
return [
    'policies' => [
        Invoice::class => InvoicePolicy::class,
        Post::class => PostPolicy::class,
    ],
];
```

Esta configuración será procesada durante bootstrap o compilación.

---

# 11. Multiple Policies mediante configuración

El mapping podrá aceptar:

```php
return [
    'policies' => [
        Invoice::class => [
            InvoicePolicy::class,
            FinancialCompliancePolicy::class,
        ],
    ],
];
```

Esto refleja el modelo many-to-many del sistema.

---

# 12. PolicyFor Attribute

Una Policy podrá declarar directamente qué Subject protege.

```php
#[PolicyFor(Invoice::class)]
final class InvoicePolicy
{
}
```

Esto genera:

```text
Invoice
   ↓
InvoicePolicy
```

---

# 13. Múltiples Subjects

Una Policy transversal podrá declarar:

```php
#[PolicyFor([
    Invoice::class,
    CreditNote::class,
])]
final class FinancialDocumentPolicy
{
}
```

Esto deberá utilizarse con moderación.

---

# 14. Policy Attribute sobre Subject

Opcionalmente podrá existir:

```php
#[Policy(InvoicePolicy::class)]
final class Invoice
{
}
```

pero no será la opción recomendada por defecto.

La razón es mantener las entidades de dominio libres de metadata de infraestructura cuando sea posible.

---

# 15. Convención de nombres

VoltStack podrá descubrir automáticamente:

```text
App\Domain\Invoice
        ↓
App\Policies\InvoicePolicy
```

o:

```text
App\Models\Post
        ↓
App\Policies\PostPolicy
```

La convención deberá ser configurable.

---

# 16. ConventionResolver

Conceptualmente:

```php
interface PolicyConventionResolverInterface
{
    public function candidateFor(
        string $subjectClass
    ): ?string;
}
```

Ejemplo:

```text
App\Models\Post
     ↓
App\Policies\PostPolicy
```

---

# 17. Convención de namespaces

Configuración conceptual:

```php
return [
    'discovery' => [
        'subject_namespaces' => [
            'App\\Models\\',
            'App\\Domain\\',
        ],

        'policy_namespace' => 'App\\Policies\\',
    ],
];
```

---

# 18. Convención no exclusiva

Si existe:

```text
InvoicePolicy
```

por convención y además:

```text
FinancialCompliancePolicy
```

registrada explícitamente, ambas podrán coexistir.

El sistema no debe asumir mapping uno-a-uno.

---

# 19. Policy Discovery

`PolicyDiscovery` será responsable de encontrar candidatos durante:

```text
development
build
cache generation
```

No durante cada request productiva.

---

# 20. Discovery Sources

Podrá descubrir Policies desde:

```text
Application
Framework
Packages
Modules
Quantum modules
```

Cada fuente deberá estar claramente aislada.

---

# 21. Package Policies

Un paquete podrá registrar:

```text
Policies
Global Policies
Security Policies
Subject Policies
```

mediante un:

```text
AuthorizationServiceProvider
```

o manifest compilable.

---

# 22. Package isolation

Un paquete no deberá poder sobrescribir silenciosamente una Policy explícita de la aplicación.

La aplicación deberá tener precedencia sobre paquetes en configuraciones equivalentes.

---

# 23. Policy Precedence

Precedencia recomendada:

```text
Explicit Application Registration
        ↓
Application Attributes
        ↓
Application Convention
        ↓
Package Explicit Registration
        ↓
Package Attributes
        ↓
Framework Defaults
```

Esta precedencia afecta conflictos, no necesariamente orden de ejecución.

---

# 24. Precedencia ≠ prioridad

Debe distinguirse:

```text
registration precedence
```

de:

```text
execution priority
```

La precedencia resuelve:

```text
¿qué metadata es válida?
```

La prioridad resuelve:

```text
¿qué Policy se ejecuta primero?
```

---

# 25. PolicyDescriptor

Cada Policy se representará mediante metadata inmutable.

```php
final readonly class PolicyDescriptor
{
    public function __construct(
        public string $policyClass,
        public PolicyType $type,
        public PolicySource $source,
        public int $priority,
        public array $subjects,
        public array $abilities,
        public array $methods,
        public bool $global = false,
        public bool $terminal = false,
    ) {}
}
```

La estructura definitiva podrá enriquecerse.

---

# 26. Subject Match Metadata

Cada descriptor podrá declarar:

```text
Exact class
Parent class
Interface
Named subject
Virtual subject
Subject type
Any subject
```

Esto permitirá resolución flexible.

---

# 27. Exact Class Mapping

Ejemplo:

```text
Invoice
    ↓
InvoicePolicy
```

Será el match más específico para una instancia de `Invoice`.

---

# 28. Parent Class Mapping

Si:

```text
PremiumInvoice extends Invoice
```

y no existe una Policy específica:

```text
PremiumInvoice
      ↓
InvoicePolicy
```

podrá utilizarse como fallback.

---

# 29. Exact Policy precedence

Si existen:

```text
PremiumInvoicePolicy
InvoicePolicy
```

para `PremiumInvoice`, ambas podrían participar o la específica podría reemplazar parcialmente a la genérica según metadata.

Por defecto, VoltStack favorecerá composición.

---

# 30. Interface Policies

Ejemplo:

```php
#[PolicyFor(ExportableResource::class)]
final class ExportPolicy
{
}
```

Si:

```text
Invoice implements ExportableResource
```

podrá participar:

```text
ExportPolicy
```

---

# 31. Multiple interfaces

Un Subject:

```text
Invoice
```

podría implementar:

```text
TenantOwned
Exportable
SensitiveResource
```

y resolver:

```text
TenantPolicy
ExportPolicy
SensitiveResourcePolicy
InvoicePolicy
```

---

# 32. Interface ordering

Cuando varias interfaces tienen Policies, el orden deberá depender de:

```text
explicit priority
```

y no del orden devuelto por reflexión.

---

# 33. Named Subject Mapping

Ejemplo:

```text
admin-dashboard
       ↓
AdminDashboardPolicy
```

Mapping:

```php
$registry->registerNamed(
    'admin-dashboard',
    AdminDashboardPolicy::class,
);
```

---

# 34. Virtual Subject Mapping

Ejemplo:

```text
controller_action:
Admin\UserController::destroy
```

podrá mapear a:

```text
UserAdministrationPolicy
```

---

# 35. Controller Class Mapping

Un Controller es simplemente una clase Subject.

```text
AdminController
      ↓
AdminControllerPolicy
```

No requiere Registry separado.

---

# 36. Controller Action Mapping

Las acciones podrán registrarse mediante metadata específica.

Ejemplo:

```php
#[PolicyForControllerAction(
    controller: UserController::class,
    action: 'destroy'
)]
final class DeleteUserActionPolicy
{
}
```

Internamente se normalizará a un Virtual Subject.

---

# 37. Route Mapping

Ejemplo:

```text
route:admin.users.destroy
        ↓
AdminRoutePolicy
```

El Registry podrá manejar named/virtual subjects.

---

# 38. Command Mapping

```text
ImportCustomersCommand
         ↓
ImportCustomersPolicy
```

por exact class.

---

# 39. Job Mapping

```text
ExportDataJob
     ↓
ExportDataPolicy
```

también por class.

---

# 40. Policy Metadata Extraction

Un:

```text
PolicyMetadataExtractor
```

analizará clases durante discovery/compilation.

Podrá obtener:

```text
PolicyFor
HandlesAbility
Priority
Auditable
Memoizable
Context requirements
Subject filters
Anonymous support
Method signatures
```

---

# 41. Reflection boundary

Reflection podrá utilizarse durante:

```text
development
compilation
cache rebuild
```

pero deberá desaparecer del hot path en producción.

---

# 42. PolicyMethodDescriptor

Cada método relevante se compilará.

```php
final readonly class PolicyMethodDescriptor
{
    public function __construct(
        public string $method,
        public array $abilities,
        public bool $requiresSubject,
        public bool $supportsAnonymous,
        public array $parameterMappings,
        public array $contextRequirements,
    ) {}
}
```

---

# 43. Ability Method Mapping

Ejemplo:

```text
Ability:
approve

Method:
approve
```

o:

```text
Ability:
invoice.approve

Method:
approve
```

si se declaró metadata.

---

# 44. Metadata Validator

Antes de registrar una Policy:

```text
Extract Metadata
      ↓
PolicyMetadataValidator
```

deberá comprobar:

- clase existente;
- visibilidad correcta;
- métodos válidos;
- ability names válidos;
- subject mappings válidos;
- prioridades válidas;
- tipos de retorno soportados;
- firmas compatibles.

---

# 45. Invalid Policy class

Ejemplo:

```text
InvoicePolicy
registered
but class does not exist
```

deberá producir:

```text
PolicyClassNotFoundException
```

durante bootstrap/compilación.

---

# 46. Invalid subject mapping

Ejemplo:

```php
#[PolicyFor('NotAClass')]
```

cuando se esperaba clase.

Deberá detectarse anticipadamente.

---

# 47. Invalid method signature

Ejemplo:

```php
public function update(
    Post $post,
    Invoice $invoice
)
```

sin Principal compatible y con mapping inconsistente.

El compilador deberá marcarlo.

---

# 48. Policy Conflict Detector

Se encargará de detectar metadata incompatible.

```text
Policy metadata
       ↓
Conflict Detector
       ↓
Valid Registry
or
PolicyRegistrationConflictException
```

---

# 49. Conflicto real

Ejemplo:

```text
InvoicePolicy A
registered as exclusive exact Policy

InvoicePolicy B
registered as another exclusive exact Policy
```

Esto puede ser un conflicto.

---

# 50. Composición válida

En cambio:

```text
InvoicePolicy
FinancialCompliancePolicy
TenantIsolationPolicy
```

sobre Invoice es válido.

El sistema debe distinguir:

```text
multiple policies
```

de:

```text
ambiguous exclusive mapping
```

---

# 51. Exclusive Policies

Podrá existir metadata:

```text
exclusive = true
```

para Policies que pretendan ser la única Policy específica de determinado slot.

No deberá usarse por defecto.

---

# 52. Duplicate registration

Registrar exactamente la misma Policy dos veces desde la misma fuente deberá:

```text
deduplicate
```

o producir warning en desarrollo.

No deberá ejecutarse dos veces accidentalmente.

---

# 53. Descriptor Identity

Una identidad estable podría ser:

```text
policy class
+
subject mapping
+
ability filter
+
scope
```

para deduplicación.

---

# 54. PolicyRegistry indexes

El Registry no deberá buscar linealmente entre todas las Policies.

Mantendrá índices.

Conceptualmente:

```text
exactClassIndex
interfaceIndex
parentClassIndex
namedSubjectIndex
virtualSubjectIndex
globalPolicyIndex
```

---

# 55. Exact class index

Ejemplo:

```php
[
    Invoice::class => [
        InvoicePolicyDescriptor,
        CompliancePolicyDescriptor,
    ],
]
```

---

# 56. Interface index

Ejemplo:

```php
[
    ExportableResource::class => [
        ExportPolicyDescriptor,
    ],
]
```

---

# 57. Named subject index

```php
[
    'admin-dashboard' => [
        AdminDashboardPolicyDescriptor,
    ],
]
```

---

# 58. Global index

```php
[
    SuspendedPrincipalPolicyDescriptor,
    SecurityLockdownPolicyDescriptor,
]
```

---

# 59. PolicyResolver

`PolicyResolver` recibe:

```text
AuthorizationRequest
```

y devuelve:

```text
applicable PolicyDescriptors
```

Contrato conceptual:

```php
interface PolicyResolverInterface
{
    public function resolve(
        AuthorizationRequest $request
    ): PolicyResolution;
}
```

---

# 60. PolicyResolution

En lugar de un array bruto podrá utilizarse:

```php
final readonly class PolicyResolution
{
    public function __construct(
        public array $policies,
        public array $metadata = [],
    ) {}
}
```

Esto permitirá incluir:

```text
resolution path
matched indexes
fallbacks
conflicts avoided
```

para tracing.

---

# 61. Resolution Pipeline

Para object/class subjects:

```text
Subject
   ↓
Exact Class Policies
   ↓
Parent Class Policies
   ↓
Interface Policies
   ↓
Global Policies
   ↓
Ability Filtering
   ↓
Context Filtering
   ↓
Priority Ordering
```

El orden lógico de colección no implica necesariamente orden final de ejecución.

---

# 62. Exact Class Resolution

Para:

```text
PremiumInvoice instance
```

buscar:

```text
PremiumInvoice
```

primero.

---

# 63. Parent traversal

Después:

```text
PremiumInvoice
 ↓
Invoice
 ↓
FinancialDocument
```

podrán resolverse Policies heredables.

---

# 64. Inheritance metadata

Una Policy podrá declarar:

```text
inherit = true|false
```

Por defecto, las Resource Policies podrían ser heredables si la semántica es segura.

---

# 65. Non-inheritable Policy

Ejemplo:

```php
#[PolicyFor(
    PremiumInvoice::class,
    inherit: false
)]
```

solo aplicará a la clase exacta.

---

# 66. Interface resolution

Luego se resolverán interfaces implementadas.

Ejemplo:

```text
PremiumInvoice
implements:
TenantOwned
ExportableResource
```

---

# 67. Interface inheritance

Interfaces padre también podrán tener Policies.

El compilador podrá precalcular la jerarquía para evitar reflexión runtime.

---

# 68. Global Policies

Las Global Policies podrán añadirse independientemente del Subject.

Sin embargo, deberán filtrarse por:

```text
ability
channel
subject type
context requirements
```

para evitar ejecución innecesaria.

---

# 69. Ability Filtering

Ejemplo:

```text
DeleteProtectionPolicy
abilities:
delete
forceDelete
```

No deberá ejecutarse sobre:

```text
view
```

---

# 70. Ability match

Se podrá soportar:

```text
exact ability
```

y potencialmente:

```text
namespace pattern
```

Ejemplo:

```text
invoice.*
```

pero los wildcards deberán compilarse, no procesarse libremente en cada request.

---

# 71. Wildcard abilities

Ejemplo:

```php
#[HandlesAbility('invoice.*')]
```

puede ser útil para Global Policies.

Debe evitarse sobreuso porque reduce precisión.

---

# 72. Context Filtering

Una Policy podrá requerir:

```text
tenant
http
security
channel=web
```

Si el requisito no está presente, el Planner podrá excluirla o señalar un error según la metadata.

---

# 73. Hard vs optional requirements

Debe distinguirse:

```text
required context
```

de:

```text
applicable context
```

Ejemplo:

```text
SecurityPolicy requires SecurityContext
```

Si no existe:

```text
integration failure
```

Pero:

```text
WebOnlyPolicy applies only when channel=web
```

en CLI simplemente no aplica.

---

# 74. Policy Applicability Metadata

Conceptualmente:

```php
final readonly class PolicyApplicability
{
    public function __construct(
        public array $abilities = [],
        public array $subjectTypes = [],
        public array $channels = [],
        public array $requiredContexts = [],
    ) {}
}
```

---

# 75. Resolution determinista

Dado el mismo:

```text
Registry
AuthorizationRequest
```

la resolución deberá producir exactamente el mismo conjunto y orden lógico.

---

# 76. Stable ordering

Después de resolver:

```text
priority DESC
source precedence
stable descriptor id
```

podrá utilizarse para garantizar orden determinista.

---

# 77. Priority first

Ejemplo:

```text
SuspendedPolicy       1000
TenantPolicy           900
InvoicePolicy          500
ExportPolicy           400
```

Orden:

```text
Suspended
Tenant
Invoice
Export
```

---

# 78. Source precedence as tie-breaker

Si tienen misma prioridad:

```text
Application Explicit
```

podrá preceder a:

```text
Package
```

si esto resulta semánticamente apropiado.

---

# 79. Stable descriptor ID

Como último criterio:

```text
compiled registration index
```

garantizará reproducibilidad.

---

# 80. Policy Resolution Strategy

Podrá existir un contrato:

```php
interface PolicyResolutionStrategyInterface
{
    public function resolve(
        AuthorizationRequest $request,
        PolicyRegistryInterface $registry,
    ): PolicyResolution;
}
```

Esto permite estrategias personalizadas sin sustituir todo el Registry.

---

# 81. Default Resolution Strategy

La implementación base seguirá:

```text
Global
Exact Subject
Parent Subjects
Interfaces
Named/Virtual Matches
Ability Filters
Context Filters
Priority Sort
Deduplicate
```

El orden final exacto podrá optimizarse durante compilación.

---

# 82. Resolución para No Subject

Para:

```php
Authorization::check('access-admin');
```

Subject:

```text
NONE
```

Resolver:

```text
Global Policies
Ability-bound Policies
Subject-less Policies
```

No ejecutará Resource Policies.

---

# 83. Subject-less Policy

Ejemplo:

```php
#[PolicyForNoSubject]
#[HandlesAbility('access-admin')]
final class AdminAccessPolicy
{
}
```

Aunque para este caso un Gate podría ser más ergonómico.

---

# 84. Resolución para Named Subject

Para:

```text
admin-dashboard
```

buscar:

```text
namedSubjectIndex['admin-dashboard']
```

más Global Policies aplicables.

---

# 85. Resolución para Virtual Subject

Ejemplo:

```text
controller_action:
UserController::destroy
```

buscar:

```text
virtualType=controller_action
identifier=...
```

y Policies genéricas del tipo.

---

# 86. Virtual type Policies

Podrá existir:

```text
all controller actions
        ↓
ControllerSecurityPolicy
```

mediante:

```text
virtual subject type
=
controller_action
```

---

# 87. Controller Policy Resolution

Para un Controller class:

```text
AdminController
```

el resolver utiliza exact class + parent + interfaces.

---

# 88. Controller inheritance

Si:

```text
UserAdminController extends AdminController
```

una Policy asociada a `AdminController` podrá heredarse si está marcada como tal.

---

# 89. Controller action + resource authorization

Una acción podrá producir más de una autorización separada:

```text
Controller class authorization
+
Controller action authorization
+
Resource authorization
```

Cada una tendrá su propia resolución de Policies.

---

# 90. No merging accidental de Subjects

No deberá resolverse:

```text
AdminControllerPolicy
```

automáticamente durante:

```text
Invoice resource authorization
```

solo porque la solicitud provino de ese Controller.

El Controller está en Context, no es el Subject de esa decisión.

---

# 91. Contextual Global Policies

Si una Policy debe ejecutarse porque el Context contiene un Controller, deberá registrarse como Global/Context Policy.

No inferirse por accidente.

---

# 92. Policy Resolution Cache

La resolución estructural podrá cachearse.

Ejemplo:

```text
Subject class:
Invoice

Ability:
update

Context shape:
tenant+web

        ↓
Resolved descriptors
```

Pero deberá distinguirse de cachear decisiones.

---

# 93. Metadata resolution cache

Seguro para reutilizar:

```text
Invoice
+
update
        ↓
Policy descriptors
```

si depende únicamente de metadata inmutable.

---

# 94. Context-sensitive resolution

Si filtering depende de:

```text
channel
tenant presence
security context presence
```

la clave deberá considerar estos aspectos.

No el contenido completo del Context.

---

# 95. Resolution Shape

Podrá existir:

```text
AuthorizationResolutionShape
```

Ejemplo:

```text
subjectClass=Invoice
ability=update
channel=web
hasTenant=true
hasSecurity=true
```

Esto puede servir para cache.

---

# 96. No Principal en metadata cache

Normalmente el Principal no debe formar parte del cache de resolución porque el Registry no depende de quién es el usuario.

La decisión sí dependerá de él.

---

# 97. No Subject instance identity en resolution cache

Para resolver qué Policy aplica a:

```text
Invoice#1
Invoice#2
```

normalmente basta:

```text
Invoice::class
```

No debe crear una entrada por entidad.

---

# 98. Resolution Cache vs Decision Cache

```text
Resolution Cache
=
which policies apply
```

```text
Decision Cache
=
whether access is granted
```

Son problemas completamente distintos.

---

# 99. Compiled Registry

Durante compilación se podrá generar:

```php
return [
    'exact' => [
        Invoice::class => [
            // descriptors
        ],
    ],

    'interfaces' => [
        ExportableResource::class => [
            // descriptors
        ],
    ],

    'global' => [
        // descriptors
    ],
];
```

---

# 100. Compiled descriptors

Idealmente la estructura será optimizada para lectura:

```text
arrays
scalar metadata
class names
method names
integer priorities
bit flags
```

en lugar de construir grafos complejos durante bootstrap.

---

# 101. Runtime hydration

El Registry podrá hidratar objetos descriptor inmutables o trabajar con arrays compilados internos.

La decisión dependerá de benchmarks.

---

# 102. Zero filesystem lookup

En producción:

```php
$user->can('update', $invoice);
```

no deberá provocar:

```text
glob()
FilesystemIterator
Composer classmap scan
directory recursion
```

---

# 103. Zero discovery reflection

Idealmente tampoco:

```text
ReflectionClass(InvoicePolicy::class)
```

para determinar methods/attributes en cada request.

---

# 104. Cache invalidation

La metadata deberá invalidarse cuando cambie:

```text
Policy class
Policy attributes
Subject mapping
Authorization config
Package registration
Framework version affecting schema
```

---

# 105. Cache fingerprint

Podrá generarse utilizando:

```text
framework version
authorization metadata version
configuration fingerprint
class/source fingerprints
package manifests
```

---

# 106. Development mode

En desarrollo podrá permitirse:

```text
lazy metadata rebuild
```

o descubrimiento automático.

El developer experience debe ser:

```text
create Policy
refresh
works
```

sin ejecutar manualmente cache rebuild continuamente.

---

# 107. Production mode

En producción:

```text
build/deploy
   ↓
compile authorization metadata
   ↓
immutable runtime registry
```

---

# 108. Hot reload

En servidores de desarrollo persistentes podrá existir invalidación del Registry cuando cambien archivos.

No deberá confundirse con comportamiento productivo.

---

# 109. Policy Discovery Cache

Puede existir una cache intermedia para:

```text
which classes are Policies
```

separada del Registry compilado.

---

# 110. Composer integration

VoltStack podrá aprovechar:

```text
Composer classmap
PSR-4 metadata
package manifests
```

para discovery durante compilación.

No deberá recorrer todo `vendor/` indiscriminadamente.

---

# 111. Quantum Module Registration

Cada módulo Quantum podrá exponer un manifest:

```php
return [
    'authorization' => [
        'policies' => [...],
        'global_policies' => [...],
    ],
];
```

Esto facilita modularidad.

---

# 112. Framework Policies

VoltStack podrá registrar internamente:

```text
TenantIsolationPolicy
SuspendedPrincipalPolicy
SecurityPolicy
```

si los módulos correspondientes están habilitados.

---

# 113. Optional framework Policies

No deberán activarse Policies globales pesadas cuando el módulo no exista.

Ejemplo:

```text
MultiTenancy disabled
        ↓
no TenantIsolationPolicy
```

---

# 114. Feature-dependent Policies

El RegistryBuilder deberá conocer features habilitadas durante bootstrap.

No evaluar configuración de feature en cada decisión.

---

# 115. Policy Resolver no instancia Policies

El Resolver trabaja sobre:

```text
PolicyDescriptor
```

No deberá hacer:

```php
new InvoicePolicy();
```

Eso pertenece al `PolicyDispatcher` o `PolicyInstanceResolver`.

---

# 116. Lazy instance resolution

Flujo:

```text
PolicyResolver
      ↓
Descriptor
      ↓
Planner
      ↓
Executor
      ↓
Policy actually needed
      ↓
Container resolve
```

Así el short-circuit puede evitar instanciaciones.

---

# 117. PolicyInstanceResolver

Contrato conceptual:

```php
interface PolicyInstanceResolverInterface
{
    public function resolve(
        PolicyDescriptor $descriptor
    ): object;
}
```

Normalmente delegará al Container.

---

# 118. Class validation

El descriptor ya deberá haber sido validado durante bootstrap.

Por ello el InstanceResolver no debe realizar validaciones costosas repetitivas.

---

# 119. Policy lifecycle metadata

El descriptor podrá incluir:

```text
shared
scoped
transient
```

si el Container necesita hints.

Pero el Authorization System no deberá reemplazar al Container como gestor de lifecycle.

---

# 120. Policy aliases

Podría permitirse registrar:

```text
billing.invoice
        ↓
InvoicePolicy
```

como alias interno.

Sin embargo, el Registry deberá trabajar siempre con una identidad canónica.

---

# 121. Policy IDs

Para tracing podrá asignarse:

```text
policy:invoice
policy:tenant_isolation
```

o IDs compilados.

Esto evita exponer nombres completos de clases en telemetry externa.

---

# 122. Descriptor ID

Conceptualmente:

```text
authz.policy.invoice
```

será estable entre despliegues si la Policy no cambia semánticamente.

No es requisito estricto para V1.

---

# 123. Registration Tags

Policies podrán tener tags:

```text
tenant
security
compliance
domain
admin
```

útiles para:

- tooling;
- profiler;
- auditoría;
- debugging.

No deberán modificar semántica por sí mismos.

---

# 124. Policy Groups

Podrán existir grupos configurables:

```text
security_core
financial_compliance
tenant_protection
```

El Planner podrá activarlos según contexto.

Esta capacidad debe evitarse en V1 salvo necesidad real.

---

# 125. Policy Modules

Una alternativa más estructurada será registrar Policies desde módulos:

```text
AuthorizationModule
```

en lugar de grupos dinámicos.

---

# 126. Dynamic policy data

El Registry debe contener:

```text
Policy definitions
```

no:

```text
dynamic user permissions
```

Los permisos, roles y reglas dinámicas se consultan durante evaluación.

---

# 127. Database-backed Policy Registry

No se recomienda almacenar directamente clases de Policy arbitrarias en base de datos.

Ejemplo peligroso:

```text
policy_class = user-controlled string
```

El Registry debe construirse a partir de código/configuración confiable.

---

# 128. Database-backed authorization rules

Sí podrá existir:

```text
InvoicePolicy
      ↓
AuthorizationRuleRepository
      ↓
database rules
```

sin cambiar el Registry.

---

# 129. Security of discovery

Nunca deberá cargarse una clase no confiable simplemente porque su nombre coincide con:

```text
*Policy
```

El discovery debe limitarse a namespaces configurados y código instalado.

---

# 130. Package trust

Las Policies de un package tienen los mismos privilegios de código que el package.

Por tanto la seguridad del package manager queda fuera del Authorization Core.

---

# 131. Policy Resolver tracing

Durante desarrollo se podrá mostrar:

```text
Subject:
PremiumInvoice

Resolved:
1. SuspendedPrincipalPolicy
   source=framework
2. TenantIsolationPolicy
   source=framework
3. PremiumInvoicePolicy
   source=explicit
4. InvoicePolicy
   source=parent
5. ExportPolicy
   source=interface
```

---

# 132. Why resolved

Cada descriptor podrá indicar:

```text
matchReason
```

Ejemplos:

```text
global
exact_class
parent_class
interface
named_subject
virtual_type
```

muy útil para debugging.

---

# 133. PolicyResolutionTrace

Conceptualmente:

```php
final readonly class PolicyResolutionTrace
{
    public function __construct(
        public string $subject,
        public array $matches,
        public array $filtered,
    ) {}
}
```

Solo se construirá cuando tracing esté habilitado.

---

# 134. Filter diagnostics

Podría indicar:

```text
MfaPolicy skipped:
channel cli not supported

ExportPolicy skipped:
ability update not supported
```

---

# 135. Production optimization

Con tracing deshabilitado no deberán construirse estos strings/objetos.

---

# 136. Policy Resolution Errors

Jerarquía conceptual:

```text
PolicyResolutionException
├── PolicyClassNotFoundException
├── PolicyRegistrationConflictException
├── InvalidPolicyMetadataException
├── InvalidPolicySubjectException
├── PolicyDiscoveryException
└── PolicyCacheException
```

---

# 137. Resolution failure

Si el Registry está corrupto o inconsistente durante producción:

```text
Authorization System Failure
        ↓
Fail Closed
```

Nunca:

```text
No policy found
        ↓
Allow
```

---

# 138. No Policy Found

En cambio, un caso legítimo:

```text
No applicable Policy
```

no necesariamente es una excepción.

El resultado normal será:

```text
no Policy evaluators
        ↓
DecisionManager
        ↓
Default Deny
```

---

# 139. Unhandled ability

Ejemplo:

```text
Invoice has InvoicePolicy
but no policy handles archive
```

Resultado:

```text
ABSTAIN / no evaluator
        ↓
Default Deny
```

No tiene por qué ser excepción runtime.

---

# 140. Strict registration mode

Podrá existir tooling que convierta ese caso en warning/error durante compilación cuando una ability declarada por Controller no tenga handler.

Ejemplo:

```text
#[Authorize('archive', subject: 'invoice')]
```

pero ninguna Policy/Gate cubre `archive`.

---

# 141. Authorization coverage analysis

El compilador podrá construir un mapa:

```text
Controller Actions
Routes
Commands
    ↓
Declared abilities
    ↓
Policy/Gate coverage
```

para detectar huecos.

---

# 142. Static resolution

Cuando Controller metadata conozca:

```text
Subject class
Ability
```

podrá incluso precompilarse parte del plan.

Ejemplo:

```text
InvoiceController::update
      ↓
ability update
subject Invoice
      ↓
known PolicyDescriptors
```

---

# 143. Dynamic resolution

Si el Subject se determina en runtime:

```text
Authorization::check(
    $ability,
    $subject
)
```

se utilizará el Registry runtime.

---

# 144. Precompiled Authorization Plan

Futuro:

```text
Controller Action Metadata
        ↓
Precompiled Policy Resolution
        ↓
AuthorizationPlan template
```

Esto puede reducir aún más overhead.

---

# 145. Plan template

El template no contiene:

```text
Principal
Subject instance
Context values
```

solo:

```text
Policy descriptors
Ability
Strategy
Subject reference
```

---

# 146. Registry memory model

Bajo FrankenPHP, el Registry puede permanecer:

```text
process-wide
```

porque contiene metadata inmutable.

---

# 147. Request isolation

El Registry jamás deberá almacenar:

```text
current user
current tenant
current resource instance
last policy result
```

---

# 148. Multi-app runtimes

Si un proceso sirve múltiples aplicaciones aisladas, cada application container deberá tener su propio Registry o namespace de metadata.

Nunca compartir mapeos accidentalmente entre aplicaciones.

---

# 149. Multi-tenant runtime

Los tenants normalmente compartirán Registry.

Las diferencias por tenant deberán expresarse como:

```text
runtime Policy data
```

no registries diferentes, salvo escenarios avanzados.

---

# 150. Tenant-specific policy configuration

Si un tenant habilita reglas distintas:

```text
Policy
      ↓
Tenant configuration
      ↓
Decision
```

preferido sobre:

```text
rebuild registry per tenant
```

---

# 151. Plugin Policies

Plugins podrán registrar Policies mediante manifests.

La aplicación deberá poder:

```text
enable
disable
override
```

según reglas explícitas.

---

# 152. Plugin disable

Al deshabilitar un plugin:

```text
its Policy descriptors
```

deberán desaparecer en siguiente rebuild.

---

# 153. Policy Override

VoltStack podrá permitir:

```php
Authorization::replacePolicy(
    PackageInvoicePolicy::class,
    AppInvoicePolicy::class,
);
```

pero esta operación deberá ocurrir en bootstrap.

---

# 154. Replace vs append

Debe distinguirse:

```text
append Policy
```

de:

```text
replace Policy
```

porque en un sistema composable son operaciones distintas.

---

# 155. Registry API explícita

Conceptualmente:

```php
$builder->add(...);

$builder->replace(...);

$builder->remove(...);

$builder->addGlobal(...);
```

Estas operaciones solo estarán disponibles durante construcción.

---

# 156. Registry sealing

Después:

```php
$registry = $builder->seal();
```

El Registry queda inmutable.

---

# 157. Late registration

Si alguien intenta:

```text
register policy after registry sealed
```

deberá producir:

```text
PolicyRegistrySealedException
```

---

# 158. Development escape hatch

El entorno de desarrollo podrá reconstruir un nuevo Registry.

No mutar el existente concurrentemente.

---

# 159. Thread/concurrency safety

Un Registry inmutable es naturalmente más seguro para:

```text
concurrent requests
persistent workers
```

---

# 160. Cache serialization

La representación cacheada no deberá serializar:

```text
Policy instances
Closures
Container objects
Runtime objects
```

Solo metadata.

---

# 161. Closures as Policy mappings

No se recomienda utilizar closures dentro del Policy Registry.

Las closures pertenecen mejor a:

```text
Gates
```

y aun así su compilación tendrá limitaciones.

---

# 162. Policy class requirement

Una Policy registrada deberá ser una clase resoluble.

Esto facilita:

- DI;
- compilación;
- cache;
- profiling;
- static analysis.

---

# 163. Abstract Policies

Una clase abstracta no deberá registrarse directamente como ejecutable.

Podrá actuar como base de herencia.

---

# 164. Trait-based Policy discovery

Traits no deberán registrarse como Policies.

Pueden aportar comportamiento reutilizable.

---

# 165. Invokable Policy

Policies avanzadas podrán ser invocables:

```php
final class MaintenancePolicy
{
    public function __invoke(
        AuthorizationRequest $request
    ): DecisionResult {
    }
}
```

si están registradas como evaluator-style Policies.

---

# 166. Conventional vs evaluator Policy metadata

El descriptor deberá saber:

```text
InvocationType:
method
invoke
evaluate
```

para evitar detectar esto dinámicamente.

---

# 167. PolicyInvocationType

Conceptualmente:

```php
enum PolicyInvocationType: string
{
    case ConventionalMethod = 'method';
    case EvaluateMethod = 'evaluate';
    case Invokable = 'invokable';
}
```

---

# 168. Policy discovery of conventional methods

No deberán considerarse abilities todos los métodos públicos.

Ejemplo:

```php
public function helper()
```

no necesariamente es una ability.

En V1 podrán utilizarse:

```text
known ability conventions
explicit HandlesAbility
```

---

# 169. Known resource abilities

Convenciones iniciales:

```text
viewAny
view
create
update
delete
restore
forceDelete
```

más métodos públicos explícitamente declarados como ability.

---

# 170. Custom ability convention

Métodos como:

```text
approve
publish
archive
```

podrán inferirse si forman parte de un mapping usado por Authorization.

Sin embargo, el compilador deberá evitar tratar helpers públicos como abilities accidentalmente.

---

# 171. Recomendación

Policies deberían mantener como públicos únicamente:

```text
ability methods
before/after hooks
required framework contract methods
```

y utilizar métodos privados para helpers.

---

# 172. Before metadata

El descriptor podrá guardar:

```text
hasBefore=true
beforeMethod='before'
```

para evitar `method_exists()` runtime.

---

# 173. After metadata

Igualmente:

```text
hasAfter=true
```

---

# 174. Anonymous compatibility metadata

El compilador analizará firmas y guardará:

```text
supportsAnonymous
```

evitando reflexión runtime.

---

# 175. Subject requirement metadata

Para `create()`:

```text
requiresSubjectInstance=false
```

Para `update()`:

```text
requiresSubjectInstance=true
```

---

# 176. Argument mapping metadata

Ejemplo compilado:

```text
parameter 0 → Principal
parameter 1 → Raw Subject
parameter 2 → AuthorizationContext
```

El dispatcher podrá invocar directamente.

---

# 177. Context requirements metadata

Ejemplo:

```text
requires:
TenantContext
SecurityContext
```

El Planner podrá comprobarlo antes de instanciar la Policy.

---

# 178. Metadata schema version

El cache deberá tener:

```text
authorization_policy_schema_version
```

para invalidarse cuando cambie la estructura interna entre versiones de VoltStack.

---

# 179. Backward compatibility

La API pública de Policies deberá evolucionar con más estabilidad que el formato interno de metadata.

Los caches compilados podrán regenerarse tras actualizar VoltStack.

---

# 180. Developer tooling

Comandos futuros:

```text
volt authorization:policies
volt authorization:policy Invoice
volt authorization:resolve Invoice update
volt authorization:cache
volt authorization:clear
volt authorization:lint
```

---

# 181. `authorization:policies`

Podrá mostrar:

```text
Subject                     Policies
------------------------------------------------
Invoice                     InvoicePolicy
Invoice                     CompliancePolicy
Post                        PostPolicy
ExportableResource          ExportPolicy
GLOBAL                      SuspendedPolicy
```

---

# 182. `authorization:resolve`

Ejemplo:

```text
volt authorization:resolve App\Domain\Invoice update
```

Salida:

```text
Resolved Policies:

1. SuspendedPrincipalPolicy
2. TenantIsolationPolicy
3. InvoicePolicy
4. CompliancePolicy

Strategy:
Unanimous
```

---

# 183. Source diagnostics

También:

```text
InvoicePolicy
source: explicit
priority: 500

CompliancePolicy
source: package
priority: 300
```

---

# 184. Policy cache command

```text
volt authorization:cache
```

podrá:

```text
discover
validate
compile
write immutable metadata
```

---

# 185. CI validation

`authorization:lint` deberá poder ejecutarse en CI.

Fallará ante:

```text
invalid signatures
missing classes
conflicting mappings
invalid attributes
unresolved required authorization metadata
```

---

# 186. Testing Registry

El Testing package podrá crear un Registry aislado:

```php
$registry = PolicyRegistry::testing()
    ->add(Invoice::class, InvoicePolicy::class)
    ->seal();
```

---

# 187. Policy resolution unit testing

Ejemplo:

```php
$resolution = $resolver->resolve($request);

expect($resolution->policies)
    ->toContain(InvoicePolicy::class);
```

---

# 188. Conflict testing

Deberá ser fácil verificar:

```text
duplicate exclusive mapping
        ↓
exception
```

---

# 189. Cache equivalence testing

El sistema deberá probar que:

```text
discovered registry
```

y:

```text
compiled registry
```

producen la misma resolución.

---

# 190. Performance testing

Benchmarks importantes:

```text
exact class lookup
parent class resolution
interface resolution
global policy filtering
compiled metadata loading
policy resolution cache hit
```

---

# 191. Target hot path

Para un caso común:

```php
$user->can('update', $invoice);
```

el Policy Resolution hot path debería aproximarse a:

```text
subject class already known
        ↓
exact index lookup
        ↓
global index lookup
        ↓
ability filter
        ↓
priority ordering/precompiled ordering
        ↓
descriptors
```

sin filesystem ni reflection.

---

# 192. Pre-sorted registry buckets

Para reducir sorting runtime, cada Registry bucket podrá almacenarse ya ordenado por prioridad.

Ejemplo:

```text
Invoice bucket:
1 TenantPolicy 900
2 InvoicePolicy 500
3 CompliancePolicy 300
```

---

# 193. Ability-specific indexes

Si benchmarks lo justifican:

```text
Invoice
  update → [...]
  delete → [...]
```

podrá compilarse para evitar filtering runtime.

---

# 194. Trade-off de memoria

Ability-specific indexes consumen más memoria.

Por ello el compiler deberá buscar equilibrio entre:

```text
memory
vs
lookup speed
```

especialmente en runtimes persistentes.

---

# 195. Hybrid index

Recomendación inicial:

```text
subject bucket
+
compact ability filters
```

y optimizar después con benchmarks reales.

---

# 196. Registry architecture

```text
                       Policy Sources
                            │
        ┌───────────────────┼────────────────────┐
        ↓                   ↓                    ↓
     Explicit            Attributes          Convention
        │                   │                    │
        └───────────────────┼────────────────────┘
                            ↓
                    Policy Discovery
                            ↓
                  Metadata Extraction
                            ↓
                     Validation
                            ↓
                 Conflict Detection
                            ↓
                  Registry Builder
                            ↓
                     Compilation
                            ↓
                Immutable PolicyRegistry
                            │
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
         Exact Class    Interfaces      Globals
              │             │             │
              └─────────────┼─────────────┘
                            ↓
                      PolicyResolver
                            ↓
                    PolicyResolution
                            ↓
                 AuthorizationPlanner
```

---

# 197. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Policies/
        ├── Registry/
        │   ├── PolicyRegistry.php
        │   ├── PolicyRegistryBuilder.php
        │   ├── PolicyRegistryInterface.php
        │   └── PolicyRegistrySealedException.php
        │
        ├── Discovery/
        │   ├── PolicyDiscovery.php
        │   ├── PolicyConventionResolver.php
        │   ├── PolicyMetadataExtractor.php
        │   ├── PolicyMetadataValidator.php
        │   └── Sources/
        │
        ├── Resolution/
        │   ├── PolicyResolver.php
        │   ├── PolicyResolution.php
        │   ├── PolicyResolutionStrategy.php
        │   ├── DefaultPolicyResolutionStrategy.php
        │   └── PolicyResolutionTrace.php
        │
        ├── Metadata/
        │   ├── PolicyDescriptor.php
        │   ├── PolicyMethodDescriptor.php
        │   ├── PolicyApplicability.php
        │   ├── PolicyInvocationType.php
        │   ├── PolicySource.php
        │   └── PolicyType.php
        │
        ├── Compilation/
        │   ├── PolicyMetadataCompiler.php
        │   ├── CompiledPolicyRegistryLoader.php
        │   └── PolicyMetadataSchema.php
        │
        ├── Cache/
        │   ├── PolicyCache.php
        │   ├── PolicyCacheFingerprint.php
        │   └── PolicyResolutionCache.php
        │
        └── Exceptions/
            ├── PolicyResolutionException.php
            ├── PolicyClassNotFoundException.php
            ├── PolicyRegistrationConflictException.php
            ├── InvalidPolicyMetadataException.php
            ├── InvalidPolicySubjectException.php
            ├── PolicyDiscoveryException.php
            └── PolicyCacheException.php
```

---

# 198. Invariantes del Registry

### Invariante 1

El Registry contiene metadata, no Policy instances.

### Invariante 2

El Registry productivo es inmutable.

### Invariante 3

El Registry no contiene estado del Principal, Tenant o Request.

### Invariante 4

Una Policy puede mapearse a múltiples Subjects.

### Invariante 5

Un Subject puede tener múltiples Policies.

### Invariante 6

La precedencia de registro y la prioridad de ejecución son conceptos diferentes.

### Invariante 7

El orden de resolución debe ser determinista.

### Invariante 8

La ausencia de Policy no implica autorización.

### Invariante 9

Controllers utilizan el mismo Registry que cualquier otro Subject.

### Invariante 10

La resolución no instancia Policies.

---

# 199. Invariantes de Discovery

### Invariante 1

Discovery dinámico no pertenece al hot path productivo.

### Invariante 2

Solo se escanean namespaces/fuentes confiables.

### Invariante 3

Toda metadata descubierta debe validarse.

### Invariante 4

Los conflictos se detectan antes de ejecutar requests cuando sea posible.

### Invariante 5

La aplicación tiene precedencia sobre defaults de packages/framework donde corresponda.

---

# 200. Invariantes de Resolution

### Invariante 1

Exact class, inheritance e interfaces deben seguir reglas explícitas.

### Invariante 2

La resolución depende de metadata estructural, no de lógica de negocio.

### Invariante 3

Las decisiones del usuario no se cachean dentro del Registry.

### Invariante 4

Context filtering debe basarse solo en propiedades estructurales declaradas.

### Invariante 5

La resolución debe ser reproducible.

---

# 201. Filosofía del sistema

El desarrollador deberá poder escribir simplemente:

```php
final class InvoicePolicy
{
    public function update(
        User $user,
        Invoice $invoice
    ): bool {
        return $user->id === $invoice->user_id;
    }
}
```

y VoltStack podrá descubrir:

```text
Invoice
   ↓
InvoicePolicy
```

Pero internamente el framework mantendrá:

```text
Validated Metadata
        ↓
Compiled Registry
        ↓
Indexed Resolution
        ↓
Deterministic Policy Set
```

---

# 202. Resultado esperado

El `Policy Registry, Discovery and Resolution System` deberá permitir que VoltStack combine:

```text
Laravel-like convention
        +
explicit mappings
        +
attribute-based metadata
        +
multiple Policies
        +
Symfony-like composability
        +
compiled runtime metadata
```

sin sacrificar rendimiento ni previsibilidad.

Una solicitud:

```php
$user->can('update', $invoice);
```

deberá resolverse conceptualmente como:

```text
Invoice instance
      ↓
SubjectDescriptor
      ↓
PolicyRegistry exact lookup
      ↓
Parent/interface/global lookup
      ↓
Ability filtering
      ↓
Context applicability
      ↓
Priority ordering
      ↓
PolicyResolution
      ↓
AuthorizationPlanner
```

El principio definitivo será:

```text
Registration defines what exists.

Discovery finds it.

Compilation validates and optimizes it.

The Registry remembers it.

Resolution selects what applies.

Execution happens elsewhere.
```

Esta separación permitirá mantener el Authorization System de VoltStack rápido, extensible, modular y seguro incluso en aplicaciones empresariales grandes y runtimes persistentes.