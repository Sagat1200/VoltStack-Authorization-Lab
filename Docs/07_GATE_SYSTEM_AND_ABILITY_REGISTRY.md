# VoltStack Authorization System — Gate System and Ability Registry

## 1. Propósito

Este documento define el subsistema de **Gates y registro central de Abilities** del Authorization System de VoltStack.

Las Policies resuelven principalmente autorización asociada a Subjects:

```text
User
  ↓
update
  ↓
Invoice
  ↓
InvoicePolicy
```

Los Gates cubrirán principalmente decisiones que:

- no pertenecen naturalmente a un modelo;
- representan capacidades globales;
- protegen características de aplicación;
- protegen operaciones administrativas;
- protegen infraestructura;
- representan capacidades transversales;
- requieren autorización sin Subject;
- sirven como punto de entrada simple hacia el Authorization Engine.

Ejemplo:

```php
Gate::allows('access-admin-panel');
```

o:

```php
Authorization::allows('access-admin-panel');
```

internamente:

```text
Ability
   ↓
AbilityRegistry
   ↓
GateResolver
   ↓
GateDescriptor
   ↓
GateDispatcher
   ↓
DecisionResult
   ↓
DecisionManager
```

El objetivo es conservar una experiencia similar a Laravel, pero integrada dentro de una arquitectura de autorización compilable, extensible y común a todo VoltStack.

---

# 2. Principio arquitectónico

VoltStack no implementará Gates como un segundo sistema de autorización independiente.

El principio será:

```text
Policies
Gates
Controller Authorization
Route Authorization
Command Authorization
Job Authorization
Security Policies
        │
        ↓
Unified Authorization Engine
```

Por tanto:

```text
Gate ≠ Authorization Engine
```

Un Gate será simplemente otro tipo de:

```text
Authorization Evaluator
```

---

# 3. Diferencia fundamental entre Gate y Policy

Una Policy normalmente responde:

```text
¿Puede este Principal realizar esta Ability
sobre este Subject?
```

Ejemplo:

```php
$user->can('update', $invoice);
```

Un Gate normalmente responde:

```text
¿Puede este Principal realizar esta Ability?
```

Ejemplo:

```php
$user->can('access-admin');
```

---

# 4. Modelo conceptual

```text
Policy

Principal
   +
Ability
   +
Subject
   ↓
Decision
```

```text
Gate

Principal
   +
Ability
   ↓
Decision
```

Sin embargo, VoltStack podrá permitir Gates con argumentos adicionales cuando sea útil.

---

# 5. Cuándo usar Policy

Utilizar Policy cuando la autorización pertenece claramente a:

```text
Invoice
Post
Project
User
Controller
Command
Job
Resource
Domain Entity
```

Ejemplo:

```php
$user->can('update', $invoice);
```

---

# 6. Cuándo usar Gate

Utilizar Gate para:

```text
access-admin
access-debug-tools
manage-platform
impersonate-users
view-system-health
execute-maintenance
use-beta-feature
access-developer-console
```

Ejemplo:

```php
Gate::allows('access-admin');
```

---

# 7. No duplicar Policies con Gates

No se recomienda:

```php
Gate::define('update-invoice', ...);
```

si ya existe:

```php
InvoicePolicy::update()
```

Preferido:

```php
$user->can('update', $invoice);
```

Esto mantiene la autorización alineada con el dominio.

---

# 8. Componentes principales

El Gate System estará compuesto conceptualmente por:

```text
Ability
AbilityName
AbilityRegistry
AbilityDescriptor
AbilityRegistryBuilder

Gate
GateRegistry
GateDescriptor
GateResolver
GateDispatcher

GateDefinition
GateHandler
GateInvocationDescriptor

GateReturnNormalizer
GateResponse

GateCompiler
GateCache
```

---

# 9. Ability como concepto central

Una Ability representa:

```text
una operación autorizable
```

Ejemplos:

```text
view
create
update
delete

invoice.approve
admin.access
platform.manage
user.impersonate
system.health.view
```

---

# 10. Ability Value Object

VoltStack podrá representar una Ability mediante:

```php
final readonly class Ability
{
    public function __construct(
        public string $name,
    ) {}
}
```

Esto evita pasar strings arbitrarios por todo el Core.

---

# 11. AbilityName

Podrá existir una clase especializada:

```php
final readonly class AbilityName
{
    public function __construct(
        public string $value,
    ) {}
}
```

con validación durante construcción.

---

# 12. Formato de nombres

Formato recomendado:

```text
namespace.action
```

Ejemplos:

```text
admin.access
invoice.approve
user.impersonate
system.health.view
deployment.execute
```

Para Resource Policies podrán mantenerse nombres simples:

```text
view
create
update
delete
```

---

# 13. Canonical Ability

Toda Ability deberá convertirse a una representación canónica.

Ejemplo:

```text
Access-Admin
```

no deberá coexistir ambiguamente con:

```text
access-admin
```

VoltStack deberá definir reglas estrictas de naming.

---

# 14. Recomendación de naming

Preferencia:

```text
lowercase
dot-separated namespaces
```

Ejemplo:

```text
admin.access
users.impersonate
billing.refund
```

---

# 15. Ability validation

Una Ability inválida:

```text
Admin Access!!!
```

podrá ser rechazada durante:

```text
registration
compilation
```

---

# 16. AbilityRegistry

`AbilityRegistry` será el catálogo runtime de abilities conocidas.

Contrato conceptual:

```php
interface AbilityRegistryInterface
{
    public function has(
        Ability|string $ability
    ): bool;

    public function get(
        Ability|string $ability
    ): ?AbilityDescriptor;

    public function resolveAlias(
        Ability|string $ability
    ): Ability;
}
```

---

# 17. Registry no ejecuta autorización

El `AbilityRegistry` responde:

```text
¿Qué sabemos sobre esta Ability?
```

No:

```text
¿Está autorizado este usuario?
```

---

# 18. AbilityDescriptor

Conceptualmente:

```php
final readonly class AbilityDescriptor
{
    public function __construct(
        public Ability $ability,
        public AbilityType $type,
        public array $aliases = [],
        public ?string $description = null,
        public array $tags = [],
    ) {}
}
```

---

# 19. AbilityType

Podrá existir:

```php
enum AbilityType: string
{
    case Resource = 'resource';
    case Global = 'global';
    case System = 'system';
    case Feature = 'feature';
    case Controller = 'controller';
    case Route = 'route';
    case Command = 'command';
    case Job = 'job';
    case Custom = 'custom';
}
```

---

# 20. Ability Registry Builder

Durante bootstrap:

```text
Application
Packages
Framework
Policies
Gates
Controller Metadata
Route Metadata
        ↓
AbilityRegistryBuilder
        ↓
Validation
        ↓
Compilation
        ↓
Immutable AbilityRegistry
```

---

# 21. Registro explícito de Ability

Ejemplo:

```php
Authorization::ability('admin.access');
```

o:

```php
$abilities->register(
    'admin.access'
);
```

---

# 22. Registro implícito

Al definir:

```php
Gate::define(
    'admin.access',
    AdminAccessGate::class
);
```

la Ability podrá registrarse automáticamente.

---

# 23. Policies y AbilityRegistry

Al compilar:

```php
InvoicePolicy::update()
```

podrá registrarse:

```text
update
```

dentro del contexto:

```text
Invoice
```

Sin embargo, esto no significa que `update` sea un Gate global.

---

# 24. Ability Scope

Debe distinguirse:

```text
Global Ability
```

de:

```text
Subject Ability
```

Ejemplo:

```text
admin.access
```

es global.

Mientras:

```text
Invoice + update
```

es subject-scoped.

---

# 25. Ability Identity

Conceptualmente:

```text
Global:
admin.access

Subject:
Invoice:update
```

Esto evita colisiones semánticas.

---

# 26. Ability aliases

VoltStack podrá soportar:

```text
admin
    ↓
admin.access
```

o:

```text
edit
    ↓
update
```

cuando se declare explícitamente.

---

# 27. Alias registration

Ejemplo:

```php
Authorization::alias(
    'admin',
    'admin.access'
);
```

---

# 28. Alias normalization

Antes de resolver evaluadores:

```text
Requested Ability
      ↓
AbilityRegistry
      ↓
Canonical Ability
```

El resto del sistema trabaja con la Ability canónica.

---

# 29. Alias chains

Podría existir:

```text
admin
 ↓
admin-panel
 ↓
admin.access
```

pero durante compilación deberá colapsarse a:

```text
admin
 ↓
admin.access
```

---

# 30. Circular aliases

Ejemplo inválido:

```text
a → b
b → c
c → a
```

deberá producir:

```text
CircularAbilityAliasException
```

durante compilación.

---

# 31. Alias depth

El runtime no deberá recorrer cadenas arbitrarias.

Los aliases deberán estar pre-resueltos.

---

# 32. Gate Definition

Un Gate asocia:

```text
Ability
    ↓
Gate Handler
```

Ejemplo:

```php
Gate::define(
    'admin.access',
    fn (User $user) => $user->isAdmin()
);
```

---

# 33. Gate API

VoltStack podrá ofrecer:

```php
Gate::define(...);

Gate::allows(...);

Gate::denies(...);

Gate::check(...);

Gate::authorize(...);

Gate::any(...);

Gate::none(...);
```

---

# 34. Authorization API común

También:

```php
Authorization::allows('admin.access');

Authorization::denies('admin.access');

Authorization::authorize('admin.access');
```

La Facade `Gate` será una API especializada.

---

# 35. Gate Definition mediante Closure

Ejemplo:

```php
Gate::define(
    'admin.access',
    function (User $user): bool {
        return $user->isAdmin();
    }
);
```

---

# 36. Closure Gates

Las closures serán cómodas para reglas pequeñas.

Pero presentan limitaciones para:

```text
compilation
serialization
cache
static analysis
dependency injection
profiling
```

---

# 37. Recomendación

Closures:

```text
development
small applications
simple abilities
```

Class-based Gates:

```text
production architecture
large applications
packages
enterprise systems
```

---

# 38. Class-based Gate

Ejemplo:

```php
final class AdminAccessGate
{
    public function __invoke(
        User $user
    ): bool {
        return $user->isAdmin();
    }
}
```

Registro:

```php
Gate::define(
    'admin.access',
    AdminAccessGate::class
);
```

---

# 39. Dependency Injection

Class-based Gates podrán usar constructor injection:

```php
final class PlatformManagementGate
{
    public function __construct(
        private readonly PlatformMembership $membership,
    ) {}

    public function __invoke(
        User $user
    ): bool {
        return $this->membership->canManage($user);
    }
}
```

---

# 40. Method-based Gate Handler

También podrá registrarse:

```php
Gate::define(
    'admin.access',
    [AdminGate::class, 'access']
);
```

---

# 41. Recomendación de handlers

Orden preferido:

```text
Invokable class
Method-based class
Closure
```

para sistemas grandes.

---

# 42. GateDescriptor

Cada Gate se representará mediante metadata.

```php
final readonly class GateDescriptor
{
    public function __construct(
        public string $gateId,
        public Ability $ability,
        public GateHandlerDescriptor $handler,
        public int $priority = 0,
        public bool $terminal = false,
    ) {}
}
```

---

# 43. GateHandlerDescriptor

Conceptualmente:

```php
final readonly class GateHandlerDescriptor
{
    public function __construct(
        public GateHandlerType $type,
        public mixed $target,
        public array $parameterMappings,
        public GateReturnType $returnType,
    ) {}
}
```

---

# 44. GateHandlerType

```php
enum GateHandlerType: string
{
    case Closure = 'closure';
    case InvokableClass = 'invokable_class';
    case ClassMethod = 'class_method';
}
```

---

# 45. GateRegistry

`GateRegistry` almacenará los handlers asociados a abilities.

Ejemplo:

```text
admin.access
    ↓
AdminAccessGate
```

---

# 46. Múltiples Gates por Ability

VoltStack podrá permitir:

```text
deployment.execute
       ↓
EnvironmentGate
SecurityGate
MaintenanceGate
```

Esto es importante para composición.

---

# 47. Gate no necesariamente uno-a-uno

No deberá imponerse:

```text
1 Ability = 1 Gate
```

La arquitectura será:

```text
1 Ability
   ↓
N evaluators
```

---

# 48. Ejemplo compuesto

Para:

```text
system.deploy
```

podrían participar:

```text
DeploymentPermissionGate
EnvironmentStateGate
SecurityClearanceGate
```

---

# 49. Gate priority

Cada Gate podrá declarar:

```text
priority
```

Ejemplo:

```text
SecurityClearanceGate     900
EnvironmentStateGate      600
DeploymentPermissionGate  500
```

---

# 50. Gate registration source

Cada descriptor podrá conocer:

```text
Application
Package
Framework
Compiled
```

para debugging y precedencia.

---

# 51. Gate Registration Precedence

Recomendación:

```text
Application Explicit
    ↓
Application Attributes
    ↓
Package Explicit
    ↓
Package Attributes
    ↓
Framework Defaults
```

---

# 52. Append vs Replace

Registrar otro Gate para la misma Ability deberá significar por defecto:

```text
append
```

no:

```text
replace
```

---

# 53. Replace explícito

Ejemplo:

```php
Gate::replace(
    'admin.access',
    CustomAdminGate::class
);
```

---

# 54. Remove

Durante bootstrap podrá existir:

```php
Gate::remove(
    'admin.access',
    PackageAdminGate::class
);
```

No durante una request normal.

---

# 55. Immutable Runtime Registry

Después del bootstrap:

```text
GateRegistry
    ↓
sealed
```

No deberán registrarse Gates durante requests productivas.

---

# 56. GateResolver

El `GateResolver` recibe una Ability canónica y devuelve Gates aplicables.

Contrato:

```php
interface GateResolverInterface
{
    public function resolve(
        AuthorizationRequest $request
    ): GateResolution;
}
```

---

# 57. GateResolution

Conceptualmente:

```php
final readonly class GateResolution
{
    public function __construct(
        public Ability $ability,
        public array $gates,
    ) {}
}
```

---

# 58. Gate Resolution Pipeline

```text
Requested Ability
      ↓
Alias Resolution
      ↓
Canonical Ability
      ↓
GateRegistry Lookup
      ↓
Applicability Filtering
      ↓
Priority Ordering
      ↓
GateResolution
```

---

# 59. Gate Applicability

Un Gate podrá limitarse por:

```text
Principal type
Channel
Tenant presence
Context type
Environment
Feature
```

pero estas condiciones deberán expresarse como metadata estructural cuando sea posible.

---

# 60. No business evaluation in resolver

El Resolver no deberá ejecutar:

```text
$user->isAdmin()
```

Eso pertenece al Gate handler.

---

# 61. GateDispatcher

Una vez resuelto un Gate:

```text
GateDescriptor
      ↓
GateDispatcher
      ↓
handler invocation
      ↓
DecisionResult
```

---

# 62. GateDispatcher Contract

```php
interface GateDispatcherInterface
{
    public function dispatch(
        AuthorizationRequest $request,
        GateDescriptor $gate,
    ): DecisionResult;
}
```

---

# 63. Reutilización del Policy Dispatcher Core

Policy y Gate dispatch comparten:

```text
instance resolution
argument mapping
return normalization
exception boundaries
```

Por ello VoltStack deberá extraer componentes comunes.

---

# 64. AuthorizationEvaluatorDispatcher

Arquitectura posible:

```text
AuthorizationEvaluatorDispatcher
        │
        ├── PolicyInvocationAdapter
        │
        └── GateInvocationAdapter
```

Esto evita duplicación.

---

# 65. Gate Arguments

El handler podrá recibir:

```text
Principal
Ability
AuthorizationContext
AuthorizationRequest
registered Context Components
explicit Gate arguments
```

---

# 66. Simple Gate

```php
Gate::define(
    'admin.access',
    fn (User $user): bool => $user->isAdmin()
);
```

---

# 67. Gate con argumento

Ejemplo:

```php
Gate::define(
    'billing.refund',
    function (
        User $user,
        Money $amount
    ): bool {
        return $user->refundLimit()->greaterThanOrEqual($amount);
    }
);
```

Invocación:

```php
Gate::allows(
    'billing.refund',
    $amount
);
```

---

# 68. Explicit Gate Arguments

Los argumentos adicionales deberán encapsularse en:

```text
AuthorizationArguments
```

o equivalente.

No mezclarse arbitrariamente con Context.

---

# 69. AuthorizationArguments

Conceptualmente:

```php
final readonly class AuthorizationArguments
{
    public function __construct(
        public array $values,
    ) {}
}
```

---

# 70. Argument Mapping

Metadata:

```text
parameter 0 → Principal
parameter 1 → GateArgument[0]
```

No deberá resolverse mediante guessing runtime.

---

# 71. Named Gate Arguments

Futuro:

```php
Gate::allows(
    'billing.refund',
    amount: $amount,
);
```

podrá mapearse mediante metadata compilada.

---

# 72. No Container injection en método

Igual que Policies:

```text
services
```

deberán inyectarse por constructor en class-based Gates.

---

# 73. Gate Return Types

Gates podrán retornar:

```text
bool
null
Decision
DecisionResult
GateResponse
```

---

# 74. GateReturnNormalizer

Toda salida deberá convertirse a:

```text
DecisionResult
```

---

# 75. Boolean normalization

```text
true
 ↓
GRANT

false
 ↓
DENY
```

---

# 76. Null

Cuando esté permitido:

```text
null
 ↓
ABSTAIN
```

---

# 77. GateResponse

API ergonómica:

```php
return GateResponse::deny(
    reasonCode: 'admin.disabled',
    reason: 'Administrative access is disabled.',
);
```

---

# 78. Invalid Gate Return

Ejemplo:

```php
return 'yes';
```

deberá producir:

```text
InvalidGateResultException
```

No truthiness.

---

# 79. Gate Exceptions

Una excepción inesperada:

```text
GateExecutionException
```

deberá mantener semántica distinta de:

```text
DENY
```

---

# 80. Fail Closed

Nunca:

```text
Gate crashes
    ↓
GRANT
```

El fallo deberá terminar en comportamiento seguro.

---

# 81. `Gate::allows()`

API:

```php
Gate::allows('admin.access');
```

retorna:

```text
bool
```

---

# 82. `Gate::denies()`

```php
Gate::denies('admin.access');
```

equivale conceptualmente a:

```php
! Gate::allows('admin.access');
```

pero deberá utilizar la decisión normalizada internamente.

---

# 83. `Gate::check()`

Podrá retornar:

```text
DecisionResult
```

en lugar de bool.

Ejemplo:

```php
$result = Gate::check('admin.access');
```

---

# 84. Naming de APIs

Una API consistente recomendada:

```text
allows()      → bool
denies()      → bool
check()       → DecisionResult
authorize()   → DecisionResult or throws
```

---

# 85. `Gate::authorize()`

Ejemplo:

```php
Gate::authorize('admin.access');
```

Si:

```text
GRANT
```

continúa.

Si:

```text
DENY
```

lanza:

```text
AuthorizationDeniedException
```

---

# 86. ABSTAIN en authorize

Si ningún evaluator decide:

```text
ABSTAIN
```

la política global:

```text
Default Deny
```

producirá denegación.

---

# 87. `Gate::any()`

Ejemplo:

```php
Gate::any([
    'admin.access',
    'support.access',
]);
```

Pregunta:

```text
¿Alguna Ability está autorizada?
```

---

# 88. `Gate::none()`

Ejemplo:

```php
Gate::none([
    'admin.access',
    'support.access',
]);
```

---

# 89. `Gate::all()`

VoltStack también podrá ofrecer:

```php
Gate::all([
    'reports.view',
    'reports.export',
]);
```

para requerir todas.

---

# 90. Multi-ability evaluation

Estas APIs deberán reutilizar:

```text
AuthorizationManager
```

y no implementar loops aislados con semánticas diferentes.

---

# 91. Short-circuit en `any()`

```text
Ability A → DENY
Ability B → GRANT
```

puede detenerse en:

```text
GRANT
```

---

# 92. Short-circuit en `all()`

```text
Ability A → GRANT
Ability B → DENY
```

puede detenerse en:

```text
DENY
```

---

# 93. Result API avanzada

Podría existir:

```php
Authorization::checkAny([...]);
```

que devuelva información sobre qué Ability concedió acceso.

---

# 94. Gate for Principal

API útil:

```php
Gate::forUser($user)
    ->allows('admin.access');
```

Pero VoltStack deberá generalizar:

```php
Gate::forPrincipal($principal)
```

---

# 95. `forUser()` convenience

Podrá mantenerse por ergonomía:

```php
Gate::forUser($user)
```

como alias cuando el Principal sea un User.

---

# 96. Scoped Gate Context

Ejemplo:

```php
Gate::forPrincipal($serviceAccount)
    ->withContext($context)
    ->allows('deployment.execute');
```

---

# 97. Immutable scoped facade

Preferido:

```text
GateContext
```

inmutable.

No mutar una Facade singleton con:

```text
currentPrincipal
```

---

# 98. FrankenPHP Safety

Incorrecto:

```php
Gate::setUser($user);
```

sobre estado global persistente.

Correcto:

```php
Gate::forPrincipal($user)
```

devuelve un evaluador scoped/inmutable.

---

# 99. GateEvaluationContext

Conceptualmente:

```php
final readonly class GateEvaluationContext
{
    public function __construct(
        public PrincipalInterface $principal,
        public AuthorizationContext $context,
    ) {}
}
```

---

# 100. Current Principal

La API simple:

```php
Gate::allows(...)
```

podrá obtener el Principal desde un:

```text
request-scoped PrincipalResolver
```

Nunca desde propiedad global mutable.

---

# 101. Guest Gates

Un Gate podrá soportar anonymous Principal.

Ejemplo:

```php
Gate::define(
    'catalog.view',
    function (?User $user): bool {
        return true;
    }
);
```

---

# 102. Preferred anonymous abstraction

VoltStack favorecerá:

```text
AnonymousPrincipal
```

donde resulte más consistente.

---

# 103. Gate `before()`

Laravel ofrece global before callbacks.

VoltStack deberá evitar convertir esto en lógica invisible sin estructura.

---

# 104. Global Gate Before

Podrá soportarse por compatibilidad:

```php
Gate::before(...);
```

pero internamente deberá convertirse a:

```text
High-Priority Global Authorization Evaluator
```

---

# 105. No special magic pipeline

Es decir:

```text
Gate::before()
```

no deberá crear un sistema paralelo.

Internamente:

```text
GlobalEvaluator(priority=very_high)
```

---

# 106. Gate `after()`

Igualmente:

```php
Gate::after(...);
```

podrá mapearse a:

```text
Authorization Observer
```

preferentemente.

---

# 107. after no decision mutation

Un callback `after` no deberá convertir una decisión.

Su propósito será:

```text
observability
audit
debugging
```

---

# 108. Super Admin Pattern

Ejemplo:

```php
Gate::before(
    fn (User $user) =>
        $user->isSuperAdmin()
            ? true
            : null
);
```

VoltStack podrá soportarlo.

Pero internamente debería ser:

```text
SuperAdminOverrideEvaluator
priority = 10000
```

---

# 109. Seguridad de overrides

Los overrides globales deberán ser explícitos y visibles en:

```text
authorization:resolve
```

para evitar privilegios ocultos.

---

# 110. Deny Overrides

VoltStack deberá soportar evaluadores globales de denegación.

Ejemplo:

```text
SuspendedPrincipalGate
```

puede negar antes de otros Gates.

---

# 111. Deny vs Allow precedence

No deberá codificarse directamente dentro del Gate Registry.

Esto pertenece a:

```text
Decision Strategy
```

Ejemplos:

```text
DenyOverrides
AllowOverrides
Unanimous
FirstApplicable
```

---

# 112. Gate Resolver vs Decision Strategy

GateResolver:

```text
¿Qué Gates aplican?
```

DecisionStrategy:

```text
¿Cómo combinamos sus resultados?
```

---

# 113. Multiple Gate Results

Ejemplo:

```text
SecurityGate      → GRANT
RoleGate          → GRANT
MaintenanceGate   → DENY
```

Con:

```text
DenyOverrides
```

resultado:

```text
DENY
```

---

# 114. Gate Groups

Podrá existir agrupación lógica:

```text
admin.*
billing.*
system.*
```

pero no deberá confundirse con wildcards de autorización.

---

# 115. Ability Namespace

`admin.*` podrá servir para:

```text
tooling
documentation
discovery
```

sin significar automáticamente:

```text
permission inheritance
```

---

# 116. Wildcard Gate

Podrá registrarse explícitamente:

```text
admin.*
```

para un evaluator global.

Pero deberá compilarse.

---

# 117. Wildcard semantics

Ejemplo:

```text
admin.*
```

podría aplicar a:

```text
admin.access
admin.users.manage
admin.settings.update
```

No a:

```text
administrator.access
```

---

# 118. Wildcard implementation

No ejecutar regex arbitraria para todos los Gates en cada request.

El compilador podrá construir:

```text
prefix index
```

---

# 119. Exact match priority

Cuando existen:

```text
admin.*
admin.users.manage
```

el exact match deberá poder tener mayor especificidad.

Pero ambos pueden participar si la estrategia lo permite.

---

# 120. Ability Hierarchy

VoltStack podría soportar en versiones futuras:

```text
admin.manage
    ├── admin.users.manage
    └── admin.settings.manage
```

pero no deberá asumirse jerarquía automáticamente por puntos.

---

# 121. Dot namespaces are not inheritance

Regla:

```text
admin.users.manage
```

no implica automáticamente:

```text
admin.manage
```

La jerarquía debe declararse explícitamente.

---

# 122. Ability Implications

Futuro:

```php
Authorization::implies(
    'admin.manage',
    'admin.users.manage',
);
```

Pero esta capacidad debe mantenerse separada del aliasing.

---

# 123. Alias vs implication

```text
Alias:
A = B
```

```text
Implication:
A grants B
```

Son conceptos diferentes.

---

# 124. Ability Graph

Si se implementan implications:

```text
AbilityGraph
```

deberá validar ciclos.

No es requisito de V1.

---

# 125. Ability Metadata

Podrá incluir:

```text
description
category
risk level
tags
deprecated
replacement
```

útil para tooling.

---

# 126. Risk Level

Ejemplo:

```text
users.view
risk = low

users.impersonate
risk = critical
```

Esto no modifica automáticamente la autorización.

---

# 127. Security tooling

El risk level podrá utilizarse para:

```text
audit
review
CI
documentation
MFA policies
```

---

# 128. Ability Deprecation

Una Ability podrá declararse:

```text
deprecated
```

con:

```text
replacement
```

Ejemplo:

```text
admin
    ↓ deprecated
admin.access
```

---

# 129. Runtime deprecation warnings

Solo en desarrollo.

Producción no deberá generar overhead excesivo.

---

# 130. Gate Attributes

Podrá existir:

```php
#[Gate(
    ability: 'admin.access'
)]
final class AdminAccessGate
{
}
```

---

# 131. Ability Attribute

También:

```php
#[HandlesAbility('admin.access')]
final class AdminAccessGate
{
}
```

Esto permite reutilizar metadata común con Policies.

---

# 132. Multiple abilities per Gate

Ejemplo:

```php
#[HandlesAbility([
    'reports.view',
    'reports.export',
])]
final class ReportingGate
{
}
```

Solo recomendable cuando la lógica es realmente compartida.

---

# 133. Ability-specific methods

Otra opción:

```php
final class ReportingGate
{
    #[HandlesAbility('reports.view')]
    public function view(...): bool {}

    #[HandlesAbility('reports.export')]
    public function export(...): bool {}
}
```

Esto se acerca a una Policy.

---

# 134. Gate vs Policy boundary

Si un Gate comienza a manejar:

```text
muchas abilities
muchos métodos
un Subject específico
```

probablemente debería convertirse en Policy.

---

# 135. Gate Discovery

El discovery podrá buscar Gates en:

```text
App\Authorization\Gates
App\Gates
Packages
Quantum Modules
```

durante compilación.

---

# 136. No runtime filesystem scanning

Igual que Policies:

```text
Discover once.
Compile once.
Resolve cheaply.
```

---

# 137. Gate Compiler

Responsabilidades:

```text
discover definitions
validate handlers
compile parameter mappings
compile return strategy
resolve aliases
detect conflicts
build indexes
```

---

# 138. Gate Conflict Detection

Conflicto no significa simplemente:

```text
multiple handlers
```

porque eso es válido.

Conflicto sería:

```text
multiple exclusive handlers
invalid replacement
duplicate canonical alias
incompatible metadata
```

---

# 139. Duplicate Closure Registration

Si la misma Ability se registra dos veces con closures diferentes:

```text
append
```

por defecto podría ser peligroso.

Para closures se recomienda exigir intención explícita cuando ya existe handler.

---

# 140. Strict Gate Registration

Configuración:

```php
'strict_gate_registration' => true,
```

podrá convertir registros ambiguos en errores.

---

# 141. Gate Registry indexes

Estructura conceptual:

```text
exactAbilityIndex
wildcardPrefixIndex
globalGateIndex
aliasIndex
```

---

# 142. Exact index

```php
[
    'admin.access' => [
        AdminAccessGateDescriptor,
    ],
]
```

---

# 143. Wildcard index

```php
[
    'admin.' => [
        AdminSecurityGateDescriptor,
    ],
]
```

---

# 144. Alias index

```php
[
    'admin' => 'admin.access',
]
```

---

# 145. Pre-sorted Gate buckets

Los handlers deberán almacenarse ordenados por:

```text
priority
source precedence
stable descriptor id
```

para evitar sorting por request.

---

# 146. Ability-specific Strategy

Una Ability podrá opcionalmente declarar:

```text
decision strategy
```

Ejemplo:

```text
system.deploy
strategy = unanimous
```

---

# 147. Default Strategy

Si no se declara:

```text
Authorization System default strategy
```

---

# 148. Strategy metadata ownership

La estrategia deberá pertenecer preferentemente al:

```text
AuthorizationPlan / AbilityDescriptor
```

no a cada Gate individual.

---

# 149. Gate Cache

Podrá cachearse:

```text
Ability
    ↓
Resolved GateDescriptors
```

si depende únicamente de metadata estructural.

---

# 150. No decision cache in GateRegistry

Nunca:

```text
admin.access + user42 = true
```

dentro del Registry.

Eso pertenece a memoization/decision cache.

---

# 151. Closure Cache Problem

Closures no son adecuadas para cache de metadata portable.

Por ello producción podrá:

```text
retain closure registration during bootstrap
```

pero no serializarla directamente.

---

# 152. Class-based Gate Compilation

Una clase:

```php
AdminAccessGate::class
```

sí puede representarse fácilmente como:

```text
class name
invocation type
parameter mappings
return type
```

---

# 153. Production Recommendation

Para aplicaciones con:

```text
authorization cache
FrankenPHP
persistent workers
large deployments
```

preferir class-based Gates.

---

# 154. Package Gates

Un package podrá registrar:

```php
return [
    'gates' => [
        'package.feature.use' => FeatureGate::class,
    ],
];
```

---

# 155. Quantum Module Gates

Ejemplo:

```text
Quantum/Admin
    ↓
Authorization Manifest
    ↓
admin.access
admin.users.manage
```

---

# 156. Feature-dependent Gates

Un Gate podrá registrarse solo cuando un módulo esté habilitado.

Ejemplo:

```text
Debug Module disabled
    ↓
debug.tools.access not registered
```

---

# 157. Unknown Ability

¿Qué sucede con:

```php
Gate::allows('does.not.exist');
```

?

Por defecto:

```text
No evaluator
    ↓
ABSTAIN
    ↓
Default Deny
```

---

# 158. Strict Unknown Ability Mode

Podrá configurarse:

```php
'strict_unknown_abilities' => true,
```

para lanzar:

```text
UnknownAbilityException
```

---

# 159. Development recommendation

En desarrollo:

```text
warning/error
```

es útil.

En producción:

```text
fail closed
```

sin exponer detalles.

---

# 160. Ability Coverage Analysis

VoltStack podrá analizar:

```text
Routes
Controllers
Commands
Jobs
Templates
Components
```

que referencian abilities.

Luego comparar con:

```text
AbilityRegistry
```

---

# 161. Ejemplo de lint

```text
Route:
admin.users

Requires:
admin.users.view

Status:
NO REGISTERED EVALUATOR
```

---

# 162. CI integration

Comando futuro:

```text
volt authorization:lint
```

podrá fallar cuando existan:

```text
unknown abilities
circular aliases
invalid handlers
invalid return types
unresolved references
```

---

# 163. Ability Introspection

Comando:

```text
volt authorization:abilities
```

podrá mostrar:

```text
Ability                Type       Evaluators
------------------------------------------------
admin.access            global     2
invoice.approve         resource   InvoicePolicy
system.health.view      system     1
```

---

# 164. Gate Introspection

```text
volt authorization:gates
```

podrá mostrar:

```text
admin.access

Handlers:
1. SuspendedPrincipalGate
2. AdminAccessGate

Strategy:
DenyOverrides
```

---

# 165. Resolve command

```text
volt authorization:resolve admin.access
```

podrá mostrar:

```text
Canonical ability:
admin.access

Resolved evaluators:
1. SuspendedPrincipalPolicy
2. AdminAccessGate
3. MfaSecurityGate
```

---

# 166. Unified Introspection

La herramienta deberá mostrar tanto:

```text
Policies
Gates
Global Evaluators
```

en el mismo plan cuando corresponda.

---

# 167. Template Integration

Las vistas podrán utilizar:

```php
@can('admin.access')
    ...
@endcan
```

o:

```php
@if (Gate::allows('admin.access'))
```

---

# 168. Template compiler

Si la Ability es literal:

```php
@can('admin.access')
```

el compilador podrá validar que exista.

---

# 169. Unknown Template Ability

En strict development mode:

```text
Unknown ability:
admin.acess

Did you mean:
admin.access?
```

---

# 170. SPA Integration

El frontend nunca deberá ser autoridad final.

VoltStack podrá exponer:

```text
allowed abilities
```

como metadata de UI.

Pero:

```text
Frontend Authorization
        ↓
UX optimization only
```

Backend:

```text
source of truth
```

---

# 171. Ability Manifest

Para SPA podrá generarse:

```json
{
    "admin.access": true,
    "reports.view": true,
    "reports.export": false
}
```

para el Principal actual.

---

# 172. Manifest security

Solo deberán incluirse abilities necesarias para la UI actual cuando sea posible.

No exponer metadata interna innecesaria.

---

# 173. Manifest does not replace checks

Aunque frontend tenga:

```text
reports.export = true
```

la request backend deberá autorizar nuevamente.

---

# 174. Component Integration

Un componente podrá declarar:

```php
#[RequiresAbility('admin.access')]
final class AdminDashboard
{
}
```

y utilizar el mismo AbilityRegistry.

---

# 175. Controller Integration

```php
#[Authorize('admin.access')]
final class AdminController
{
}
```

podrá resolver hacia Gates.

---

# 176. Controller + Policy

Una acción:

```php
#[Authorize('admin.access')]
#[Authorize('update', subject: 'invoice')]
```

podrá requerir:

```text
Gate:
admin.access

Policy:
InvoicePolicy::update
```

---

# 177. Route Integration

Ejemplo:

```php
Route::get('/admin', ...)
    ->can('admin.access');
```

El Route Authorization System utilizará el mismo manager.

---

# 178. Command Integration

```php
#[Authorize('system.maintenance.execute')]
final class MaintenanceCommand
{
}
```

podrá utilizar Gate.

---

# 179. Job Integration

Jobs podrán declarar:

```text
Ability
```

cuando exista un Principal propagado de forma segura.

No asumir automáticamente que un Job tiene usuario.

---

# 180. Gate and Roles

Un Gate podrá preguntar:

```php
return $user->hasRole('admin');
```

pero:

```text
Gate
```

y:

```text
Role
```

no son equivalentes.

---

# 181. Gate and Permissions

Un Gate podrá delegar:

```php
return $permissions->has(
    $user,
    'admin.access'
);
```

pero el Permission System será un subsistema separado.

---

# 182. Ability as Permission Name

VoltStack podrá permitir que abilities y permisos compartan nombres.

Ejemplo:

```text
invoice.approve
```

pero no deberán ser conceptualmente idénticos.

---

# 183. Ability vs Permission

```text
Ability
=
operation being authorized
```

```text
Permission
=
authorization data granted to a Principal
```

---

# 184. Gate vs Permission

```text
Gate
=
evaluator
```

```text
Permission
=
input to evaluators
```

---

# 185. Gate vs Feature Flag

Tampoco son equivalentes.

```text
Feature Flag
=
is feature enabled?
```

```text
Gate
=
may this Principal use it?
```

---

# 186. Combined Example

```php
final class BetaAnalyticsGate
{
    public function __construct(
        private FeatureManager $features,
    ) {}

    public function __invoke(
        User $user
    ): bool {
        return $this->features->enabled('analytics-v2')
            && $user->hasPermission('analytics.use');
    }
}
```

---

# 187. Gate purity

Idealmente Gates deberán:

```text
read state
evaluate
return decision
```

No:

```text
modify state
send email
write audit records
dispatch business commands
```

---

# 188. Gate side effects

El framework no podrá impedir todos los side effects, pero tooling/documentación deberán desaconsejarlos.

---

# 189. Database access

Un Gate puede consultar DB mediante dependencias.

Sin embargo, Gates globales muy frecuentes deberán optimizarse cuidadosamente.

---

# 190. N+1 Authorization

Ejemplo peligroso:

```text
render 500 rows
    ↓
Gate::allows() x 500
    ↓
DB query x 500
```

El sistema deberá soportar memoization/batch authorization posteriormente.

---

# 191. Gate memoization

No pertenece al Gate Registry.

Se implementará en:

```text
Authorization Decision Cache / Request Memoization
```

---

# 192. Gate performance target

Para un Gate simple ya compilado:

```text
Ability normalization
    ↓
exact registry lookup
    ↓
handler instance resolution
    ↓
argument mapping
    ↓
method invocation
    ↓
return normalization
```

---

# 193. Zero Reflection Hot Path

No:

```text
ReflectionFunction
ReflectionMethod
Attribute scanning
```

durante requests productivas normales.

---

# 194. Zero Filesystem Hot Path

No:

```text
filesystem scanning
Composer discovery
directory traversal
```

---

# 195. Ability Registry Memory

El Registry podrá permanecer:

```text
process-wide immutable
```

bajo FrankenPHP.

---

# 196. Gate Registry Memory

También:

```text
process-wide immutable
```

si contiene exclusivamente metadata.

---

# 197. Gate instances

Su lifecycle será administrado por Container.

No por GateRegistry.

---

# 198. Request State

Nunca guardar en Registry:

```text
current principal
current tenant
current request
last decision
```

---

# 199. Scoped Evaluation

Toda información dinámica deberá viajar mediante:

```text
AuthorizationRequest
AuthorizationContext
AuthorizationSession
```

---

# 200. Concurrency Safety

Los registries inmutables permiten:

```text
Request A ─┐
           ├─ shared AbilityRegistry
Request B ─┤
           ├─ shared GateRegistry
Request C ─┘
```

sin locks de escritura.

---

# 201. Gate Registration Security

No deberá permitirse que datos del usuario definan:

```text
PHP class names
callables
closures
handler methods
```

dentro del Gate Registry.

---

# 202. Database-defined abilities

Sí podrá almacenarse:

```text
permission/ability identifiers
```

en DB.

Pero no handlers PHP arbitrarios.

---

# 203. Dynamic Rules

Reglas dinámicas deberán ser evaluadas por un Gate confiable.

Ejemplo:

```text
DynamicPermissionGate
    ↓
PermissionRepository
    ↓
DB
```

---

# 204. Gate IDs

Cada Gate deberá poseer identidad estable.

Ejemplo:

```text
authz.gate.admin_access
authz.gate.mfa_security
```

útil para:

```text
tracing
audit
profiling
debugging
```

---

# 205. Stable Descriptor IDs

Durante compilación se podrá generar:

```text
GateDescriptor ID
```

determinista.

---

# 206. Gate Tags

Ejemplo:

```text
security
admin
billing
infrastructure
```

útiles para tooling.

No cambian semántica automáticamente.

---

# 207. Security Critical Gate

Metadata opcional:

```text
critical=true
```

podrá afectar:

```text
observability
audit requirements
error severity
```

pero no convertir automáticamente la estrategia.

---

# 208. Gate Execution Tracing

Ejemplo:

```text
Ability:
system.deploy

Gate:
SecurityClearanceGate

Result:
DENY

Reason:
security.clearance_required

Duration:
0.24ms
```

---

# 209. Resolution Tracing

También:

```text
Requested:
admin

Alias:
admin → admin.access

Resolved Gates:
1. SuspendedPrincipalGate
2. AdminAccessGate
```

---

# 210. Production tracing

Debe poder desactivarse completamente o reducirse a metadata mínima.

---

# 211. Exceptions

Jerarquía conceptual:

```text
GateException
├── GateRegistrationException
├── GateRegistrySealedException
├── GateHandlerNotFoundException
├── InvalidGateHandlerException
├── InvalidGateResultException
├── GateExecutionException
├── UnknownAbilityException
├── InvalidAbilityException
├── CircularAbilityAliasException
└── GateCompilationException
```

---

# 212. GateRegistrationException

Para configuraciones inválidas durante bootstrap.

---

# 213. GateRegistrySealedException

Para intentos de mutación después de:

```text
seal()
```

---

# 214. InvalidGateHandlerException

Ejemplo:

```php
Gate::define(
    'admin.access',
    NonInvokableClass::class
);
```

---

# 215. UnknownAbilityException

Solo cuando strict mode así lo requiera.

---

# 216. GateExecutionException

Envuelve errores inesperados del handler.

---

# 217. Gate Compilation Cache

Representación conceptual:

```php
return [
    'abilities' => [
        'admin.access' => [
            'type' => 'global',
            'handlers' => [
                // descriptors
            ],
        ],
    ],

    'aliases' => [
        'admin' => 'admin.access',
    ],
];
```

---

# 218. Closure Definitions in Cache

Las closures no deberán serializarse ingenuamente.

Opciones:

```text
re-register during bootstrap
```

o:

```text
disable full Gate cache
```

para esa definición.

---

# 219. Cacheability Metadata

Cada Gate podrá marcarse:

```text
metadataCacheable=true|false
```

---

# 220. Cache Warning

En producción:

```text
Closure Gate prevents full authorization compilation.
```

podrá aparecer como warning de optimización.

---

# 221. Application Bootstrap Example

```php
final class AuthorizationServiceProvider
{
    public function boot(): void
    {
        Gate::define(
            'admin.access',
            AdminAccessGate::class
        );

        Gate::define(
            'users.impersonate',
            ImpersonateUserGate::class
        );
    }
}
```

---

# 222. Package Bootstrap Example

```php
public function authorization(
    AbilityRegistryBuilder $abilities,
    GateRegistryBuilder $gates,
): void {
    $gates->add(
        'package.feature.use',
        PackageFeatureGate::class,
    );
}
```

---

# 223. Attribute Example

```php
#[HandlesAbility('admin.access')]
final class AdminAccessGate
{
    public function __invoke(
        User $user
    ): bool {
        return $user->isAdmin();
    }
}
```

---

# 224. Compiled Example

```text
Ability:
admin.access

Canonical ID:
admin.access

Handlers:
1 AdminSecurityGate
2 AdminAccessGate

Aliases:
admin

Strategy:
DenyOverrides
```

---

# 225. Runtime Example

```php
Gate::allows('admin');
```

internamente:

```text
admin
 ↓
AbilityRegistry
 ↓
admin.access
 ↓
GateResolver
 ↓
AdminSecurityGate
AdminAccessGate
 ↓
AuthorizationPlan
 ↓
execute
 ↓
DecisionManager
 ↓
bool
```

---

# 226. Gate with Explicit Principal

```php
Gate::forPrincipal($user)
    ->allows('admin.access');
```

Pipeline:

```text
Principal
+
Ability
+
Context
 ↓
AuthorizationRequest
 ↓
AuthorizationManager
```

---

# 227. Gate with Arguments

```php
Gate::forPrincipal($user)
    ->allows(
        'billing.refund',
        $amount
    );
```

Request:

```text
Principal:
$user

Ability:
billing.refund

Arguments:
[$amount]

Subject:
NONE
```

---

# 228. Policy-like Gate misuse

Si aparece:

```php
Gate::allows(
    'invoice.update',
    $invoice
);
```

repetidamente, probablemente debe modelarse:

```php
$user->can('update', $invoice);
```

con:

```text
InvoicePolicy
```

---

# 229. Migration from Laravel

Laravel:

```php
Gate::define(
    'update-post',
    fn (User $user, Post $post) =>
        $user->id === $post->user_id
);
```

VoltStack podrá soportarlo.

Pero recomendará:

```text
PostPolicy::update
```

cuando exista un Subject claro.

---

# 230. Laravel Compatibility Philosophy

VoltStack conservará conceptos familiares:

```text
Gate::define()
Gate::allows()
Gate::denies()
Gate::authorize()
Gate::forUser()
Gate::before()
```

pero los implementará sobre:

```text
Unified Authorization Engine
```

---

# 231. Symfony Influence

Del enfoque Symfony se adopta la idea de:

```text
multiple evaluators
abstain
decision strategies
```

Por tanto Gates no serán simplemente:

```text
ability → closure → bool
```

sino evaluadores composables.

---

# 232. VoltStack Extension

VoltStack añade:

```text
Ability Registry
Compiled Gate Metadata
Aliases
Multiple Gate Evaluators
Priority
Unified DecisionResult
Controller integration
Route integration
Command integration
FrankenPHP-safe runtime
```

---

# 233. Testing Gates

API conceptual:

```php
$tester
    ->gate('admin.access')
    ->for($user)
    ->assertGranted();
```

---

# 234. Unit Testing Handler

Class-based Gates podrán probarse directamente:

```php
$gate = new AdminAccessGate(...);

$result = $gate($user);
```

---

# 235. Integration Testing

También:

```php
expect(
    Gate::forPrincipal($user)
        ->allows('admin.access')
)->toBeTrue();
```

---

# 236. Resolution Testing

```php
$resolution = $gateResolver->resolve($request);

expect($resolution->gates)
    ->toContain(AdminAccessGate::class);
```

---

# 237. Alias Testing

```text
admin
```

deberá resolver exactamente a:

```text
admin.access
```

---

# 238. Compilation Equivalence

Debe probarse que:

```text
dynamic registry
```

y:

```text
compiled registry
```

resuelven exactamente los mismos Gates.

---

# 239. Concurrency Testing

Bajo runtime persistente:

```text
Principal A
Principal B
Principal C
```

no deberán compartir estado de autorización mutable.

---

# 240. Performance Testing

Benchmarks:

```text
exact ability lookup
alias lookup
wildcard lookup
multi-gate resolution
class handler dispatch
closure dispatch
DecisionResult normalization
```

---

# 241. Recommended V1 Feature Set

V1 debería incluir:

```text
Ability value object
AbilityRegistry
GateRegistry
Gate::define()
Closure Gates
Class-based Gates
Gate::allows()
Gate::denies()
Gate::check()
Gate::authorize()
Gate::any()
Gate::all()
Gate::none()
Gate::forPrincipal()
Ability aliases
Multiple Gates per Ability
GRANT / DENY / ABSTAIN
Compiled class-based Gate metadata
Strict return normalization
```

---

# 242. Features para versiones posteriores

Podrán añadirse:

```text
Ability implication graph
Ability hierarchy
Batch Gate evaluation
Distributed authorization
Remote evaluators
Advanced wildcard indexes
Static source-code ability extraction
Frontend ability manifests
Risk-aware authorization tooling
```

---

# 243. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    ├── Abilities/
    │   ├── Ability.php
    │   ├── AbilityName.php
    │   ├── AbilityType.php
    │   ├── AbilityDescriptor.php
    │   ├── AbilityRegistry.php
    │   ├── AbilityRegistryInterface.php
    │   ├── AbilityRegistryBuilder.php
    │   ├── AbilityAliasResolver.php
    │   └── AbilityValidator.php
    │
    ├── Gates/
    │   ├── Registry/
    │   │   ├── GateRegistry.php
    │   │   ├── GateRegistryInterface.php
    │   │   └── GateRegistryBuilder.php
    │   │
    │   ├── Resolution/
    │   │   ├── GateResolver.php
    │   │   ├── GateResolverInterface.php
    │   │   └── GateResolution.php
    │   │
    │   ├── Dispatch/
    │   │   ├── GateDispatcher.php
    │   │   ├── GateDispatcherInterface.php
    │   │   └── GateInvocationDescriptor.php
    │   │
    │   ├── Metadata/
    │   │   ├── GateDescriptor.php
    │   │   ├── GateHandlerDescriptor.php
    │   │   ├── GateHandlerType.php
    │   │   └── GateReturnType.php
    │   │
    │   ├── Results/
    │   │   ├── GateReturnNormalizer.php
    │   │   └── GateResponse.php
    │   │
    │   ├── Compilation/
    │   │   ├── GateCompiler.php
    │   │   └── CompiledGateRegistryLoader.php
    │   │
    │   └── Exceptions/
    │       ├── GateException.php
    │       ├── GateRegistrationException.php
    │       ├── GateRegistrySealedException.php
    │       ├── InvalidGateHandlerException.php
    │       ├── InvalidGateResultException.php
    │       ├── GateExecutionException.php
    │       └── GateCompilationException.php
    │
    └── Exceptions/
        ├── UnknownAbilityException.php
        ├── InvalidAbilityException.php
        └── CircularAbilityAliasException.php
```

---

# 244. Invariantes del Ability Registry

### Invariante 1

Toda Ability posee una identidad canónica.

### Invariante 2

Los aliases se resuelven antes de la evaluación.

### Invariante 3

Los aliases no forman ciclos.

### Invariante 4

Los namespaces con puntos no implican herencia.

### Invariante 5

El Registry contiene metadata, no decisiones.

### Invariante 6

El Registry productivo es inmutable.

---

# 245. Invariantes del Gate System

### Invariante 1

Un Gate es un evaluator, no un Authorization Engine independiente.

### Invariante 2

Una Ability puede tener múltiples Gates.

### Invariante 3

Múltiples Gates no constituyen conflicto automáticamente.

### Invariante 4

El GateResolver no ejecuta lógica de negocio.

### Invariante 5

El GateDispatcher no decide la estrategia global.

### Invariante 6

Los resultados se normalizan a `DecisionResult`.

### Invariante 7

Una excepción nunca significa `GRANT`.

### Invariante 8

Los Gates no almacenan estado de request.

---

# 246. Invariantes de Seguridad

### Invariante 1

Una Ability desconocida nunca concede acceso.

### Invariante 2

Un Gate sin evaluadores termina bajo Default Deny.

### Invariante 3

Los handlers PHP no pueden provenir de input no confiable.

### Invariante 4

Los errores de ejecución fallan de forma segura.

### Invariante 5

El frontend nunca sustituye la autorización backend.

### Invariante 6

Los overrides globales deben ser visibles mediante tooling.

---

# 247. Invariantes de Performance

### Invariante 1

No filesystem scanning en el hot path.

### Invariante 2

No attribute scanning en el hot path.

### Invariante 3

No Reflection para handlers compilados.

### Invariante 4

Los buckets pueden almacenarse preordenados.

### Invariante 5

La metadata inmutable puede compartirse bajo FrankenPHP.

---

# 248. Arquitectura unificada

```text
                       AUTHORIZATION REQUEST
                               │
                               ↓
                        Ability Registry
                               │
                     Canonical Ability
                               │
              ┌────────────────┴────────────────┐
              │                                 │
              ↓                                 ↓
       Subject present?                  No Subject
              │                                 │
              ↓                                 ↓
       Policy Resolver                    Gate Resolver
              │                                 │
              ↓                                 ↓
      Policy Descriptors                 Gate Descriptors
              │                                 │
              └────────────────┬────────────────┘
                               ↓
                    Authorization Planner
                               ↓
                     Evaluator Execution
                               │
                  ┌────────────┴────────────┐
                  ↓                         ↓
          Policy Dispatcher          Gate Dispatcher
                  │                         │
                  └────────────┬────────────┘
                               ↓
                        DecisionResults
                               ↓
                        DecisionManager
                               ↓
                    GRANT / DENY / ABSTAIN
```

---

# 249. Importante: Policy y Gate pueden coexistir

Aunque normalmente:

```text
Subject
 ↓
Policies
```

y:

```text
No Subject
 ↓
Gates
```

VoltStack no deberá codificar esta división de manera rígida.

Una autorización sobre un Subject puede necesitar además:

```text
Global Security Gate
```

Por ejemplo:

```php
$user->can('delete', $invoice);
```

podría producir:

```text
SuspendedPrincipalEvaluator
TenantSecurityEvaluator
InvoicePolicy::delete
```

---

# 250. Unified Evaluator Model

La arquitectura definitiva deberá pensar en:

```text
Authorization Evaluator
```

como abstracción común.

Implementaciones:

```text
Policy Evaluator
Gate Evaluator
Global Security Evaluator
Permission Evaluator
Role Evaluator
External Evaluator
```

Todos producen:

```text
DecisionResult
```

---

# 251. AuthorizationEvaluatorDescriptor

Futuro contrato común:

```php
interface AuthorizationEvaluatorDescriptor
{
    public function id(): string;

    public function priority(): int;

    public function type(): EvaluatorType;
}
```

---

# 252. EvaluatorType

```php
enum EvaluatorType: string
{
    case Policy = 'policy';
    case Gate = 'gate';
    case Global = 'global';
    case Permission = 'permission';
    case External = 'external';
}
```

---

# 253. Ventaja arquitectónica

Esto permitirá que el AuthorizationPlanner reciba:

```text
PolicyDescriptor
GateDescriptor
```

y los transforme en:

```text
EvaluatorInvocationDescriptor
```

antes de ejecución.

---

# 254. Ejemplo completo

Solicitud:

```php
Gate::forPrincipal($user)
    ->authorize('system.deploy');
```

Resolución:

```text
system.deploy
      ↓
AbilityRegistry
      ↓
canonical ability
      ↓
GateResolver
      ↓
SecurityClearanceGate
EnvironmentGate
DeploymentPermissionGate
      ↓
AuthorizationPlanner
      ↓
DenyOverrides strategy
      ↓
GateDispatcher
```

Resultados:

```text
SecurityClearanceGate
    → GRANT

EnvironmentGate
    → GRANT

DeploymentPermissionGate
    → DENY
      reason:
      deployment.permission_missing
```

DecisionManager:

```text
DenyOverrides
      ↓
DENY
```

`authorize()`:

```text
DENY
 ↓
AuthorizationDeniedException
```

---

# 255. Ejemplo con alias

Solicitud:

```php
Gate::allows('admin');
```

Registry:

```text
admin
 ↓ alias
admin.access
```

Gates:

```text
SuspendedPrincipalGate
AdminAccessGate
```

Resultados:

```text
SuspendedPrincipalGate
    → ABSTAIN

AdminAccessGate
    → GRANT
```

Resultado:

```text
GRANT
```

API:

```php
true
```

---

# 256. Ejemplo Policy + Global Gate

Solicitud:

```php
$user->can(
    'update',
    $invoice
);
```

Plan:

```text
Ability:
update

Subject:
Invoice

Evaluators:

1. SuspendedPrincipalEvaluator
2. TenantIsolationEvaluator
3. InvoicePolicy::update
```

Resultados:

```text
SuspendedPrincipal
    → ABSTAIN

TenantIsolation
    → GRANT

InvoicePolicy
    → GRANT
```

Resultado:

```text
GRANT
```

---

# 257. Filosofía del sistema

La API para el desarrollador debe seguir siendo:

```php
Gate::define(...);

Gate::allows(...);

$user->can(...);
```

Mientras internamente VoltStack mantiene:

```text
Canonical Abilities
        ↓
Immutable Registries
        ↓
Compiled Metadata
        ↓
Multiple Evaluators
        ↓
Explicit Invocation
        ↓
Normalized Decisions
        ↓
Decision Strategies
```

---

# 258. Resultado esperado

El `Gate System and Ability Registry` permitirá combinar la simplicidad familiar de Laravel:

```php
Gate::define(
    'admin.access',
    fn (User $user) => $user->isAdmin()
);

Gate::allows('admin.access');
```

con capacidades propias de un Authorization Engine más avanzado:

```text
GRANT / DENY / ABSTAIN
Multiple evaluators
Decision strategies
Ability aliases
Canonical ability names
Compiled metadata
Class-based Gates
Global evaluators
Controller authorization
Route authorization
Command authorization
SPA manifests
Static analysis
FrankenPHP-safe runtime
```

El principio definitivo será:

```text
An Ability names the operation.

The AbilityRegistry defines its identity.

A Gate evaluates whether it may happen.

A Policy evaluates whether it may happen
to a particular Subject.

Both feed the same Authorization Engine.

Neither owns the final authorization strategy.
```

Con esta arquitectura, VoltStack evita mantener dos sistemas separados —uno de Gates y otro de Policies— y establece una base común para todo el modelo de autorización del framework.