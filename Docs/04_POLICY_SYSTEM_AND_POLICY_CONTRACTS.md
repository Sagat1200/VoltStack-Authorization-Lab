# VoltStack Authorization System — Policy System and Policy Contracts

## 1. Propósito

Este documento define el **Policy System de VoltStack**, incluyendo los contratos, tipos de Policies, ciclo de evaluación, convenciones, resultados, composición y reglas arquitectónicas que deberán seguir las Policies del framework y de las aplicaciones.

Una Policy representa una regla de autorización capaz de responder:

```text
¿Puede este Principal ejecutar esta Ability
sobre este Subject dentro de este Context?
```

Las Policies constituirán una de las principales formas declarativas y programáticas de expresar autorización en VoltStack.

El sistema deberá combinar la ergonomía familiar de Laravel:

```php
$postPolicy->update($user, $post);
```

con una arquitectura más general capaz de proteger:

```text
Models
Entities
Controllers
Controller Actions
Routes
Commands
Jobs
Components
Services
Virtual Resources
Tenant Resources
System Operations
```

sin crear motores separados.

---

# 2. Principio fundamental

Todas las Policies deberán formar parte del mismo:

```text
Authorization Core Engine
```

Una Policy nunca constituirá un mecanismo de autorización independiente.

El flujo será:

```text
AuthorizationRequest
        ↓
AuthorizationPlanner
        ↓
PolicyResolver
        ↓
PolicyDescriptor
        ↓
PolicyDispatcher
        ↓
Policy Method
        ↓
DecisionResult
        ↓
DecisionManager
```

Por tanto:

```text
Policy ≠ Authorization Engine
```

Una Policy es:

```text
un evaluator dentro del Authorization Engine
```

---

# 3. Objetivos del Policy System

El sistema deberá proporcionar:

- Policies asociadas a recursos;
- Policies sobre clases;
- Policies sobre Controllers;
- Policies sobre Controller Actions;
- Policies globales;
- Policies de seguridad;
- Policies multi-tenant;
- Policies contextuales;
- Policies para recursos virtuales;
- Policies explícitamente registradas;
- Policies descubiertas por convención;
- Policies declaradas mediante atributos;
- múltiples Policies sobre un mismo recurso;
- prioridades;
- composición;
- `before()` compatible con ergonomía Laravel;
- resultados explicables;
- soporte para `GRANT`, `DENY` y `ABSTAIN`;
- compilación de metadata;
- resolución lazy;
- seguridad para runtimes persistentes.

---

# 4. Qué es una Policy

Conceptualmente:

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

Pero internamente VoltStack interpretará esto como:

```text
Policy:
InvoicePolicy

Ability:
update

Principal:
User#42

Subject:
Invoice#928

Result:
GRANT / DENY / ABSTAIN
```

---

# 5. Policy como componente de decisión

Una Policy deberá concentrarse en:

```text
authorization rules
```

No deberá asumir responsabilidades como:

```text
Authentication
Request Validation
Controller Dispatch
Business Command Execution
Response Generation
Rendering
Redirects
Database Transactions
```

---

# 6. Resource Policy

El tipo más común será:

```text
Resource Policy
```

Ejemplo:

```php
final class PostPolicy
{
    public function view(
        User $user,
        Post $post,
    ): bool {
        return true;
    }

    public function update(
        User $user,
        Post $post,
    ): bool {
        return $post->user_id === $user->id;
    }

    public function delete(
        User $user,
        Post $post,
    ): bool {
        return $post->user_id === $user->id;
    }
}
```

Mapping:

```text
Post
 ↓
PostPolicy
```

---

# 7. Class-level Policy operations

Algunas abilities no operan sobre una instancia.

Ejemplo:

```php
public function create(
    User $user
): bool {
    return $user->canCreatePosts();
}
```

Solicitud:

```php
$user->can('create', Post::class);
```

Aquí:

```text
SubjectType:
CLASS

Subject:
Post::class
```

---

# 8. Instance-level operations

Ejemplo:

```php
$user->can('update', $post);
```

Aquí:

```text
SubjectType:
OBJECT
```

y el dispatcher podrá invocar:

```php
PostPolicy::update(
    $user,
    $post,
);
```

---

# 9. Controller Policies

VoltStack deberá permitir Policies directamente sobre Controllers.

Ejemplo:

```php
final class AdminControllerPolicy
{
    public function access(
        User $user
    ): bool {
        return $user->isAdministrator();
    }
}
```

Mapping:

```text
AdminController
       ↓
AdminControllerPolicy
```

Uso:

```php
#[Authorize('access')]
final class AdminController
{
}
```

---

# 10. Controller Policy como Resource Policy

No será necesario crear un segundo sistema denominado:

```text
ControllerAuthorizationEngine
```

Un Controller simplemente podrá convertirse en:

```text
Subject
```

Por ejemplo:

```text
SubjectType:
CLASS

Subject:
AdminController::class
```

y el Policy System lo tratará igual que cualquier otro recurso.

---

# 11. Controller Action Policies

También deberá soportarse autorización específica de acciones.

Ejemplo:

```php
#[Authorize('delete', subject: 'user')]
public function destroy(User $user)
{
}
```

La Policy podría ser:

```php
UserPolicy::delete(
    $principal,
    $user
);
```

Pero también podrá existir una Policy orientada específicamente al Controller Action.

---

# 12. Controller Action como Virtual Subject

Ejemplo:

```text
SubjectType:
VIRTUAL

Type:
controller_action

Identifier:
Admin\UserController::destroy
```

Esto permitiría:

```text
UserControllerPolicy
        ↓
destroy()
```

o una Policy registrada explícitamente.

---

# 13. Cuándo usar Resource Policy vs Controller Policy

La regla recomendada será:

```text
Si la autorización pertenece al recurso o dominio:
    Resource Policy.

Si la autorización pertenece al acceso a una operación
de infraestructura/controller:
    Controller Policy.
```

Ejemplo correcto:

```text
¿Puede modificar esta factura?

InvoicePolicy
```

Ejemplo correcto:

```text
¿Puede acceder al panel administrativo?

AdminControllerPolicy
```

---

# 14. Evitar duplicación

No deberá escribirse simultáneamente:

```text
InvoiceControllerPolicy::update()
```

y:

```text
InvoicePolicy::update()
```

para representar exactamente la misma regla.

Preferido:

```text
Controller
   ↓
resolve Invoice
   ↓
InvoicePolicy
```

La Controller Policy deberá reservarse para reglas realmente asociadas al Controller.

---

# 15. Global Policies

VoltStack deberá permitir Policies aplicadas transversalmente.

Ejemplo:

```text
SuspendedPrincipalPolicy
```

que podría ejecutarse sobre cualquier autorización.

Conceptualmente:

```php
final class SuspendedPrincipalPolicy
{
    public function evaluate(
        AuthorizationRequest $request
    ): DecisionResult {
        if ($request->principal->isSuspended()) {
            return DecisionResult::deny(
                'principal.suspended'
            );
        }

        return DecisionResult::abstain();
    }
}
```

---

# 16. Global Policy vs Gate

Debe distinguirse:

```text
Gate
```

de:

```text
Global Policy
```

Un Gate normalmente representa una Ability específica:

```text
access-admin
```

Una Global Policy puede participar en muchas abilities:

```text
SuspendedPrincipalPolicy
TenantIsolationPolicy
MaintenanceRestrictionPolicy
```

---

# 17. Security Policies

Podrán existir Policies especializadas para restricciones de seguridad.

Ejemplos:

```text
MfaRequiredPolicy
TrustedDevicePolicy
NetworkRestrictionPolicy
SuspendedAccountPolicy
CompromisedCredentialPolicy
```

Estas Policies podrán participar antes de Resource Policies.

---

# 18. Tenant Policies

Multi-tenancy podrá implementar evaluadores como:

```text
TenantMembershipPolicy
TenantIsolationPolicy
TenantStatusPolicy
TenantResourceOwnershipPolicy
```

Ejemplo:

```text
Principal
Tenant#7
Invoice Tenant#9
      ↓
TenantIsolationPolicy
      ↓
DENY
```

---

# 19. Context Policies

Algunas Policies estarán orientadas al contexto.

Ejemplo:

```php
final class ProductionDeploymentPolicy
{
    public function deploy(
        PrincipalInterface $principal,
        DeploymentTarget $target,
        AuthorizationContext $context,
    ): DecisionResult {
        if (
            $target->isProduction()
            && !$context->security()->mfaVerified
        ) {
            return DecisionResult::deny(
                'security.mfa_required'
            );
        }

        return DecisionResult::abstain();
    }
}
```

---

# 20. Policy Contracts

VoltStack deberá permitir diferentes niveles de contrato.

No todas las Policies deberán implementar obligatoriamente una interfaz si se utilizan métodos convencionales.

Sin embargo, internamente deberán normalizarse a:

```text
AuthorizationEvaluatorInterface
```

---

# 21. AuthorizationEvaluatorInterface

Contrato interno:

```php
interface AuthorizationEvaluatorInterface
{
    public function evaluate(
        AuthorizationRequest $request
    ): DecisionResult;
}
```

Las Policies convencionales podrán adaptarse mediante:

```text
PolicyEvaluator
```

---

# 22. PolicyEvaluator

Flujo:

```text
AuthorizationRequest
       ↓
PolicyEvaluator
       ↓
PolicyResolver
       ↓
PolicyDispatcher
       ↓
Policy Method
       ↓
Return Normalizer
       ↓
DecisionResult
```

Esto permite que una Policy sencilla no necesite conocer:

```text
AuthorizationRequest
DecisionManager
AuthorizationPlan
```

---

# 23. PolicyInterface

Para Policies que quieran trabajar directamente con el modelo completo podrá existir:

```php
interface PolicyInterface
{
    public function evaluate(
        AuthorizationRequest $request
    ): DecisionResult;
}
```

Esto será útil principalmente para:

```text
Global Policies
Security Policies
Infrastructure Policies
Advanced Context Policies
```

---

# 24. Conventional Policy

Una Policy de aplicación típica podrá seguir siendo:

```php
final class InvoicePolicy
{
    public function view(
        User $user,
        Invoice $invoice,
    ): bool {
        // ...
    }

    public function update(
        User $user,
        Invoice $invoice,
    ): bool {
        // ...
    }
}
```

No necesitará:

```php
implements PolicyInterface
```

---

# 25. Ventaja del modelo dual

VoltStack podrá proporcionar:

```text
Simple Policies
        +
Advanced Policy Contracts
```

Así:

```text
Application Developer
    ↓
simple methods
```

mientras:

```text
Framework / Enterprise
    ↓
AuthorizationRequest evaluators
```

---

# 26. Policy Method Convention

Por defecto:

```text
Ability name
      ↓
Policy method name
```

Ejemplo:

```text
view     → view()
create   → create()
update   → update()
delete   → delete()
approve  → approve()
publish  → publish()
```

---

# 27. Namespaced Ability mapping

Una Ability:

```text
invoice.approve
```

no deberá convertirse automáticamente en un método PHP inválido.

El Policy metadata system podrá definir:

```text
invoice.approve
      ↓
approve()
```

cuando la Policy ya esté asociada a `Invoice`.

---

# 28. Explicit Ability Mapping

Podrá declararse:

```php
#[HandlesAbility('invoice.approve')]
public function approve(
    User $user,
    Invoice $invoice
): bool {
}
```

Esto evita depender exclusivamente del nombre del método.

---

# 29. Multiple Abilities per method

Podrá permitirse:

```php
#[HandlesAbility([
    'view',
    'download',
])]
public function access(
    User $user,
    Document $document,
): bool {
}
```

pero deberá utilizarse con moderación.

Abilities diferentes deberían conservar semántica independiente cuando las reglas puedan divergir.

---

# 30. Policy Method Signature

La forma básica será:

```php
public function update(
    User $user,
    Post $post,
): bool
```

Pero podrán aceptarse:

```php
public function update(
    User $user,
    Post $post,
    AuthorizationContext $context,
): DecisionResult
```

---

# 31. Principal parameter

El primer parámetro convencional será el Principal.

Ejemplo:

```php
public function update(
    User $user,
    Post $post
)
```

o:

```php
public function execute(
    PrincipalInterface $principal,
    CommandDefinition $command
)
```

---

# 32. Anonymous Principal signatures

Una Policy que soporte anonymous deberá utilizar un tipo compatible.

Preferido:

```php
public function view(
    PrincipalInterface $principal,
    Article $article
): bool
```

o un contrato específico compatible.

---

# 33. Nullable User compatibility

Para ergonomía compatible con Laravel podrá permitirse:

```php
public function view(
    ?User $user,
    Article $article
): bool
```

El dispatcher podrá mapear:

```text
AnonymousPrincipal
      ↓
null
```

solo dentro del adapter de compatibilidad.

Internamente VoltStack seguirá utilizando:

```text
AnonymousPrincipal
```

---

# 34. Subject parameter

Para instance-level Policies:

```php
public function update(
    User $user,
    Invoice $invoice
)
```

Para class-level operations:

```php
public function create(
    User $user
)
```

No será necesario pasar:

```php
Invoice::class
```

al método `create()` salvo que la Policy lo solicite explícitamente.

---

# 35. AuthorizationContext injection

Podrá añadirse:

```php
public function update(
    User $user,
    Invoice $invoice,
    AuthorizationContext $context,
): bool
```

El dispatcher lo resolverá desde:

```text
AuthorizationRequest.context
```

---

# 36. Context subtypes

Podrá estudiarse soporte para:

```php
TenantContext $tenant
SecurityContext $security
```

pero únicamente para tipos registrados como:

```text
Authorization Context Components
```

No deberá convertirse la firma de Policy en un acceso irrestricto al Container.

---

# 37. No arbitrary dependency injection in methods

No se recomienda:

```php
public function update(
    User $user,
    Invoice $invoice,
    DatabaseManager $db,
    Mailer $mailer,
    HttpClient $http,
)
```

Las dependencias de la Policy deberán inyectarse normalmente por constructor.

---

# 38. Constructor Dependency Injection

Ejemplo:

```php
final class InvoicePolicy
{
    public function __construct(
        private readonly MembershipService $memberships,
    ) {}

    public function approve(
        User $user,
        Invoice $invoice,
    ): bool {
        return $this->memberships
            ->canApprove($user, $invoice);
    }
}
```

---

# 39. Policy statelessness

Las Policies deberán diseñarse preferentemente como:

```text
stateless services
```

Correcto:

```php
final readonly class InvoicePolicy
{
    public function __construct(
        private MembershipService $memberships,
    ) {}
}
```

Incorrecto:

```php
final class InvoicePolicy
{
    private ?User $currentUser = null;
}
```

---

# 40. Persistent runtime safety

Especialmente bajo FrankenPHP:

```text
Policy instance
```

no deberá almacenar:

```text
current user
current tenant
current request
last subject
last decision
```

entre evaluaciones.

---

# 41. Policy lifetime

El Container podrá resolver Policies como:

```text
transient
scoped
shared
```

según configuración.

Pero solo las Policies realmente stateless deberán ser seguras como servicios compartidos.

---

# 42. Return Types

Una Policy podrá devolver inicialmente:

```text
bool
Decision
DecisionResult
null
```

El sistema normalizará todos estos tipos.

---

# 43. Boolean return

```php
return true;
```

se convierte en:

```text
GRANT
```

```php
return false;
```

se convierte en:

```text
DENY
```

---

# 44. Null return

```php
return null;
```

se convertirá en:

```text
ABSTAIN
```

cuando el método tenga permitido abstenerse.

Esto resulta especialmente útil para:

```text
before()
Global Policies
Composable Policies
```

---

# 45. Decision enum

Podrá existir:

```php
enum Decision: string
{
    case Grant = 'grant';
    case Deny = 'deny';
    case Abstain = 'abstain';
}
```

Una Policy podrá retornar:

```php
return Decision::Grant;
```

---

# 46. DecisionResult

Para decisiones explicables:

```php
return DecisionResult::deny(
    reasonCode: 'invoice.not_owner',
    reason: 'The principal does not own this invoice.',
);
```

---

# 47. Static factories

La API recomendada:

```php
DecisionResult::grant();

DecisionResult::deny();

DecisionResult::abstain();
```

y:

```php
DecisionResult::grant(
    reasonCode: 'invoice.owner'
);
```

---

# 48. Policy Response Alias

Para familiaridad con Laravel podrá existir:

```php
use VoltStack\Authorization\PolicyResponse;
```

Ejemplo:

```php
return PolicyResponse::allow();
```

o:

```php
return PolicyResponse::deny(
    'You cannot update this invoice.'
);
```

Internamente:

```text
PolicyResponse
      ↓
DecisionResult
```

---

# 49. Una representación interna

Aunque existan APIs:

```text
bool
Decision
PolicyResponse
DecisionResult
null
```

todas deberán convertirse inmediatamente en:

```text
DecisionResult
```

No deberán propagarse diferentes tipos al Decision Engine.

---

# 50. PolicyReturnNormalizer

Contrato conceptual:

```php
interface PolicyReturnNormalizerInterface
{
    public function normalize(
        mixed $result
    ): DecisionResult;
}
```

---

# 51. Invalid return type

Si una Policy retorna:

```php
return 'yes';
```

deberá producir:

```text
InvalidPolicyResultException
```

No deberá interpretarse mediante truthiness de PHP.

---

# 52. Fail closed

Una excepción durante normalización nunca podrá producir:

```text
GRANT
```

El boundary de ejecución aplicará:

```text
failure
 ↓
fail closed
```

---

# 53. `before()`

Para ergonomía Laravel, una Policy convencional podrá definir:

```php
public function before(
    User $user,
    string $ability,
): bool|null {
    // ...
}
```

---

# 54. Semántica de `before()`

Mapeo:

```text
true
    ↓
GRANT

false
    ↓
DENY

null
    ↓
continue
```

Ejemplo:

```php
public function before(
    User $user,
    string $ability
): ?bool {
    if ($user->isSuperAdmin()) {
        return true;
    }

    return null;
}
```

---

# 55. `before()` no debe ser mágico

Internamente:

```text
PolicyDescriptor
    ↓
Before Method
    ↓
PolicyDispatcher
```

deberá formar parte explícita del plan de Policy.

No deberá implementarse como una excepción escondida al Decision Engine.

---

# 56. `before()` con DecisionResult

VoltStack podrá extenderlo:

```php
public function before(
    User $user,
    Ability $ability,
): DecisionResult|null
```

Ejemplo:

```php
return DecisionResult::deny(
    'account.suspended'
);
```

---

# 57. `before()` y ABSTAIN

Preferido:

```php
return DecisionResult::abstain();
```

para APIs avanzadas.

Para compatibilidad:

```php
return null;
```

seguirá siendo equivalente.

---

# 58. Policy-level before vs Global Policy

Debe distinguirse:

```text
InvoicePolicy::before()
```

de:

```text
Global SuspendedPrincipalPolicy
```

El primero solo aplica a:

```text
InvoicePolicy
```

El segundo puede aplicar al Authorization Plan completo.

---

# 59. Recomendación

Reglas realmente globales como:

```text
suspended user
tenant isolation
system lockdown
```

deberán implementarse preferentemente como:

```text
Global/Security Policies
```

y no duplicarse en cada `before()`.

---

# 60. `after()`

Podrá soportarse:

```php
public function after(
    User $user,
    string $ability,
    DecisionResult $result,
): void {
}
```

pero su uso deberá ser limitado.

---

# 61. After hooks no modificadores

Por defecto:

```text
after()
```

será observacional.

Podrá utilizarse para:

```text
Audit
Metrics
Diagnostics
```

No deberá modificar la decisión.

---

# 62. Modificación post-decision

Si una regla necesita modificar:

```text
GRANT / DENY
```

debe ser:

```text
Policy
Evaluator
Decision Strategy
```

no un hook oculto.

---

# 63. Policy Registration

Las Policies podrán registrarse explícitamente.

Ejemplo:

```php
Authorization::policy(
    Invoice::class,
    InvoicePolicy::class,
);
```

---

# 64. Config registration

También:

```php
return [
    'policies' => [
        Invoice::class => InvoicePolicy::class,
        Post::class => PostPolicy::class,
    ],
];
```

---

# 65. Attribute registration

Podrá permitirse:

```php
#[Policy(InvoicePolicy::class)]
final class Invoice
{
}
```

o alternativamente sobre la Policy:

```php
#[PolicyFor(Invoice::class)]
final class InvoicePolicy
{
}
```

---

# 66. Recomendación sobre atributos

Se recomienda favorecer:

```php
#[PolicyFor(Invoice::class)]
final class InvoicePolicy
{
}
```

sobre modificar entidades de dominio con metadata de infraestructura.

Sin embargo, ambas estrategias podrán soportarse.

---

# 67. Convention-based discovery

VoltStack podrá detectar:

```text
App\Models\Post
        ↓
App\Policies\PostPolicy
```

o:

```text
App\Domain\Invoice
        ↓
App\Policies\InvoicePolicy
```

durante:

```text
development discovery
```

---

# 68. No runtime scanning

En producción:

```text
Policy convention discovery
```

deberá compilarse.

No deberá ejecutarse filesystem scanning en cada request.

---

# 69. PolicyRegistry

Todas las asociaciones terminarán en:

```text
PolicyRegistry
```

Ejemplo conceptual:

```php
$registry->policiesFor(
    Invoice::class
);
```

---

# 70. Multiple Policies per Subject

A diferencia de un modelo estrictamente:

```text
Invoice → InvoicePolicy
```

VoltStack deberá permitir:

```text
Invoice
 ├── InvoicePolicy
 ├── TenantIsolationPolicy
 ├── FinancialCompliancePolicy
 └── SensitiveDataPolicy
```

---

# 71. Por qué múltiples Policies

Esto permite separar:

```text
Domain Authorization
Tenant Security
Compliance
Risk
Infrastructure Security
```

sin construir una Policy monolítica.

---

# 72. PolicyDescriptor

Cada Policy registrada deberá producir metadata.

```php
final readonly class PolicyDescriptor
{
    public function __construct(
        public string $policyClass,
        public PolicyType $type,
        public int $priority,
        public array $abilities,
        public bool $terminal,
        public bool $shared,
    ) {}
}
```

La estructura concreta evolucionará.

---

# 73. PolicyType

Conceptualmente:

```php
enum PolicyType: string
{
    case Resource = 'resource';
    case Global = 'global';
    case Security = 'security';
    case Tenant = 'tenant';
    case Context = 'context';
    case Controller = 'controller';
    case Custom = 'custom';
}
```

---

# 74. Tipos como metadata

`PolicyType` no deberá crear motores diferentes.

Todos terminan en:

```text
AuthorizationEvaluatorInterface
```

El tipo ayuda al:

```text
Planner
Ordering
Tracing
Configuration
```

---

# 75. Policy Priority

Policies podrán tener prioridad.

Ejemplo:

```text
SuspendedPrincipalPolicy   1000
TenantIsolationPolicy       900
SecurityPolicy              800
ResourcePolicy              500
ContextPolicy               300
```

Los valores definitivos deberán configurarse mediante rangos estables.

---

# 76. Priority direction

VoltStack deberá elegir una única convención.

Recomendación:

```text
higher number
=
earlier execution
```

y mantenerla en todo Authorization System.

---

# 77. Equal priority

Si dos Policies tienen la misma prioridad, el orden deberá ser determinista.

Puede utilizarse:

```text
registration order
```

o:

```text
compiled stable order
```

Nunca depender del orden accidental del filesystem.

---

# 78. Policy Ordering

El `AuthorizationPlanner` construirá:

```text
PolicyExecutionPlan
```

Ejemplo:

```text
1 SuspendedPrincipalPolicy
2 TenantIsolationPolicy
3 MfaRequiredPolicy
4 InvoicePolicy
5 FinancialCompliancePolicy
```

---

# 79. Terminal Policy

Una Policy podrá estar marcada como:

```text
terminal
```

pero la terminalidad deberá depender también de:

```text
Decision Strategy
```

No debe asumirse que cualquier `GRANT` termina automáticamente la evaluación.

---

# 80. Critical Deny

Ciertas Policies podrán producir un:

```text
critical deny
```

conceptualmente.

Ejemplo:

```text
TenantIsolationPolicy
```

La implementación deberá modelarse mediante metadata de ejecución y estrategia, no añadiendo comportamiento mágico a `DecisionResult`.

---

# 81. Decision Strategies

Con múltiples Policies, el resultado dependerá de una estrategia.

Ejemplos:

```text
Affirmative
Unanimous
Consensus
DenyOverrides
GrantOverrides
Priority
FirstApplicable
```

Las estrategias se especificarán en un documento posterior.

---

# 82. Recomendación default

Para seguridad general se recomienda que la configuración base favorezca:

```text
DENY overrides GRANT
```

cuando múltiples Policies obligatorias participan.

Pero el comportamiento exacto deberá ser explícito por plan.

---

# 83. ABSTAIN

`ABSTAIN` será fundamental para composición.

Significa:

```text
Esta Policy no toma una decisión
para esta solicitud.
```

No significa:

```text
DENY
```

ni:

```text
GRANT
```

---

# 84. Ejemplo ABSTAIN

```php
final class ProductionMfaPolicy
{
    public function deploy(
        PrincipalInterface $principal,
        DeploymentTarget $target,
    ): DecisionResult {
        if (!$target->isProduction()) {
            return DecisionResult::abstain();
        }

        // evaluar regla
    }
}
```

---

# 85. All abstain

Si todas las Policies responden:

```text
ABSTAIN
```

el resultado final deberá ser:

```text
DENY
```

por Default Deny.

---

# 86. Policy Applicability

Idealmente el Planner deberá evitar ejecutar Policies que claramente no aplican.

Ejemplo:

```text
InvoicePolicy
```

no debería recibir:

```text
Post
```

para luego devolver `ABSTAIN`.

La applicability deberá resolverse mediante metadata cuando sea posible.

---

# 87. Supports Contract

Policies avanzadas podrán implementar:

```php
interface SupportsAuthorizationRequest
{
    public function supports(
        AuthorizationRequest $request
    ): bool;
}
```

Pero esto puede añadir costo runtime.

Preferido:

```text
compiled applicability metadata
```

cuando sea posible.

---

# 88. Ability-specific Policies

Podrá registrarse una Policy solo para:

```text
delete
```

Ejemplo:

```php
#[PolicyFor(
    subject: Invoice::class,
    abilities: ['delete'],
)]
final class InvoiceDeletionPolicy
{
}
```

---

# 89. Global ability filter

Una Global Policy también podrá declarar:

```text
abilities:
[
    'delete',
    'export',
    'deploy'
]
```

para evitar ejecutarse en decisiones irrelevantes.

---

# 90. Subject filters

Podrá aplicarse sobre:

```text
subject classes
interfaces
subject types
named subjects
virtual subject types
```

---

# 91. Context filters

Policies avanzadas podrán declarar:

```text
channels:
api
web

tenantRequired:
true

securityContextRequired:
true
```

El Planner utilizará esta metadata.

---

# 92. Policy Composition

Una Policy podrá reutilizar reglas mediante servicios.

Ejemplo:

```php
final class InvoicePolicy
{
    public function __construct(
        private readonly InvoiceAuthorizationRules $rules,
    ) {}

    public function update(
        User $user,
        Invoice $invoice
    ): bool {
        return $this->rules
            ->canModify($user, $invoice);
    }
}
```

---

# 93. No direct Policy-to-Policy coupling

No se recomienda:

```php
final class InvoicePolicy
{
    public function __construct(
        private CustomerPolicy $customers,
    ) {}
}
```

Esto crea:

```text
hidden authorization graph
```

---

# 94. Nested authorization

Cuando realmente se necesite otra autorización:

```php
$this->authorization->check(
    'manage',
    $invoice->customer
);
```

deberá pasar nuevamente por el Core Engine.

Así se conservan:

```text
Tracing
Decision Strategies
Security Policies
Recursion Detection
Audit
```

---

# 95. Circular Policy detection

Ejemplo peligroso:

```text
InvoicePolicy
   ↓
authorize Customer
   ↓
CustomerPolicy
   ↓
authorize Invoice
   ↓
InvoicePolicy
```

El `AuthorizationExecutionStack` deberá detectar ciclos.

---

# 96. Shared Authorization Rules

Cuando dos Policies comparten una condición:

```text
same organization membership
```

preferido:

```text
OrganizationMembershipRule
```

o:

```text
MembershipService
```

No invocar una Policy desde otra únicamente para reutilizar código.

---

# 97. Rule Objects

VoltStack podrá incorporar posteriormente:

```text
AuthorizationRuleInterface
```

para reglas pequeñas reutilizables.

Ejemplo:

```text
OwnsResource
BelongsToTenant
HasPermission
RequiresMfa
```

Estas Rules no necesariamente serán evaluadores completos.

---

# 98. Policy vs Rule

```text
Rule
=
reusable condition
```

```text
Policy
=
authorization decision boundary
```

Una Policy puede combinar múltiples Rules.

---

# 99. Roles dentro de Policies

Una Policy podrá consultar:

```php
$user->hasRole('manager');
```

pero se recomienda abstraer el sistema de roles cuando exista un componente dedicado.

Ejemplo:

```php
$this->roles->has(
    $user,
    'manager'
);
```

---

# 100. Permissions dentro de Policies

Ejemplo:

```php
if (!$this->permissions->allows(
    $user,
    'invoice.update'
)) {
    return DecisionResult::deny(
        'permission.missing'
    );
}
```

Esto permite combinar:

```text
RBAC
+
Resource Policy
```

---

# 101. RBAC no sustituye Policies

Un permiso:

```text
invoice.update
```

puede permitir acceder a la operación.

La Policy aún puede verificar:

```text
tenant
ownership
status
amount
workflow
```

Por tanto:

```text
RBAC
+
Policy
```

son complementarios.

---

# 102. ABAC dentro de Policies

Las Policies son un lugar natural para reglas ABAC.

Ejemplo:

```text
Principal.department
Subject.classification
Context.region
Context.security
```

---

# 103. ReBAC dentro de Policies

También pueden consultar relaciones:

```text
User
  ↓ belongs to
Organization
  ↓ owns
Project
```

El Policy System no deberá estar limitado a RBAC.

---

# 104. Policy Reason Codes

Toda denegación importante debería poder proporcionar:

```text
reasonCode
```

Ejemplos:

```text
invoice.not_owner
invoice.locked
tenant.mismatch
security.mfa_required
account.suspended
permission.missing
```

---

# 105. Human-readable reason

Opcionalmente:

```text
The invoice is locked.
```

pero deberá tratarse como información interna hasta que una capa externa determine si es seguro exponerla.

---

# 106. Localization

No se recomienda almacenar mensajes traducidos dentro del Core.

Preferido:

```text
reasonCode
      ↓
HTTP/UI layer
      ↓
translation
```

---

# 107. Policy metadata

Una Policy podrá declarar metadata como:

```text
auditable
sensitive
priority
abilities
subjects
channels
required contexts
```

---

# 108. Attributes

Ejemplo conceptual:

```php
#[PolicyFor(Invoice::class)]
#[Priority(500)]
#[Auditable]
final class InvoicePolicy
{
}
```

---

# 109. Method metadata

También:

```php
#[HandlesAbility('approve')]
#[RequiresMfa]
#[Auditable]
public function approve(...)
{
}
```

---

# 110. Compilation

Durante desarrollo:

```text
Reflection
      ↓
Policy Metadata
```

Durante build/cache:

```text
Policy Metadata
      ↓
Compiled Authorization Registry
```

Durante producción:

```text
Compiled Registry
      ↓
Direct Lookup
```

---

# 111. Reflection-free hot path

Objetivo:

```php
$user->can('update', $invoice);
```

no deberá requerir normalmente:

```text
ReflectionClass
ReflectionMethod
Filesystem Scan
Attribute Discovery
```

en producción.

---

# 112. PolicyMethodDescriptor

Metadata compilada:

```php
final readonly class PolicyMethodDescriptor
{
    public function __construct(
        public string $method,
        public array $abilities,
        public bool $supportsAnonymous,
        public bool $requiresSubject,
        public array $contextRequirements,
    ) {}
}
```

---

# 113. Policy Invocation Descriptor

El Planner podrá producir:

```text
PolicyInvocationDescriptor
```

conteniendo:

```text
policy class
method
priority
subject mapping
principal mapping
context arguments
return normalization
```

---

# 114. PolicyDispatcher

Responsabilidades:

```text
resolve Policy instance
resolve method
map arguments
invoke method
capture exceptions
normalize result
return DecisionResult
```

---

# 115. Dispatcher no decide estrategia

El `PolicyDispatcher` no deberá decidir:

```text
qué Policy gana
```

Eso corresponde al:

```text
DecisionManager
```

El dispatcher únicamente ejecuta una Policy.

---

# 116. Policy Method Not Found

Si una Policy está registrada para una Ability pero el método esperado no existe:

```text
PolicyMethodNotFoundException
```

en desarrollo.

En producción:

```text
fail closed
+
diagnostic trace
```

---

# 117. Missing ability method

Si la Policy está asociada al recurso pero no declara una Ability:

```text
ABSTAIN
```

puede ser correcto.

Ejemplo:

```text
InvoicePolicy
```

solo implementa:

```text
view
update
```

y se pregunta:

```text
archive
```

No necesariamente es un error.

---

# 118. Distinción crítica

```text
Policy does not handle ability
```

puede producir:

```text
ABSTAIN
```

Mientras:

```text
Metadata says Policy handles ability
but method does not exist
```

es:

```text
configuration error
```

---

# 119. Policy Exceptions

Una excepción inesperada:

```php
throw new RuntimeException(...);
```

no deberá interpretarse como:

```text
DENY normal
```

Internamente deberá registrarse como:

```text
Policy Execution Failure
```

aunque la frontera de seguridad aplique fail-closed.

---

# 120. Exception Boundary

```text
PolicyDispatcher
      ↓
try invoke
      ↓
Throwable
      ↓
PolicyExecutionException
      ↓
Trace
      ↓
Fail Closed
```

---

# 121. Domain exceptions

Una Policy puede encontrarse con una excepción de dominio.

Se recomienda evitar utilizar excepciones para decisiones normales.

Preferido:

```php
return DecisionResult::deny(
    'invoice.locked'
);
```

No:

```php
throw new InvoiceLockedException();
```

para expresar autorización.

---

# 122. Pure authorization decision

Idealmente una Policy debe ser:

```text
side-effect free
```

No debería:

```text
send email
modify database
dispatch job
write audit record directly
change subject
change principal
```

---

# 123. Database reads

Sí podrá realizar consultas necesarias para decidir.

Ejemplo:

```text
organization membership
permission lookup
relationship lookup
```

pero deberán optimizarse para evitar N+1.

---

# 124. Database writes

No se recomienda realizar:

```text
INSERT
UPDATE
DELETE
```

como efecto secundario de una Policy.

La auditoría deberá realizarse fuera mediante hooks del Authorization Engine.

---

# 125. Idempotencia

Evaluar una Policy dos veces sobre el mismo estado debería producir la misma decisión y no generar efectos secundarios adicionales.

---

# 126. Determinismo

Con:

```text
same Principal
same Subject
same Ability
same Context
same relevant external state
```

la Policy deberá producir el mismo resultado.

---

# 127. Time-dependent Policies

Cuando dependan del tiempo deberán utilizar una abstracción:

```php
ClockInterface
```

Ejemplo:

```php
if ($this->clock->now() > $invoice->approvalDeadline()) {
    return DecisionResult::deny(
        'invoice.approval_expired'
    );
}
```

Esto mejora testabilidad.

---

# 128. External authorization services

Policies que dependan de un servicio remoto deberán manejarse cuidadosamente.

No se recomienda introducir llamadas HTTP ocultas en cada Policy.

Podrá utilizarse:

```text
RemoteAuthorizationEvaluator
```

para escenarios especializados.

---

# 129. Timeout

Cualquier evaluator remoto deberá tener:

```text
strict timeout
```

y aplicar:

```text
fail closed
```

si la decisión no puede obtenerse.

---

# 130. Policy Memoization

Una Policy podrá declararse:

```text
memoizable
```

solo si su resultado es seguro de reutilizar dentro del scope definido.

---

# 131. Default memoization

Por defecto:

```text
Policy decision memoization
=
disabled
```

hasta que exista metadata suficiente.

---

# 132. Memoizable Policy

Futuro atributo:

```php
#[Memoizable(
    scope: 'request'
)]
final class StaticRolePolicy
{
}
```

---

# 133. No cross-request decision cache by default

Especialmente importante:

```text
User#42 can update Invoice#928
```

no deberá almacenarse globalmente entre requests sin un sistema explícito de invalidación.

---

# 134. Batch Policies

Para listas grandes podrá existir:

```php
interface BatchPolicyInterface
{
    public function evaluateBatch(
        PrincipalInterface $principal,
        string|Ability $ability,
        iterable $subjects,
        AuthorizationContext $context,
    ): iterable;
}
```

Esto permitirá evitar N+1.

---

# 135. Batch result

Ejemplo:

```text
Invoice#1 → GRANT
Invoice#2 → DENY
Invoice#3 → GRANT
```

Cada resultado seguirá siendo conceptualmente una decisión independiente.

---

# 136. Batch compatibility

Una Policy normal no deberá estar obligada a implementar batch.

El Core podrá hacer fallback a evaluación individual.

---

# 137. Controller class Policy example

```php
#[PolicyFor(AdminController::class)]
final class AdminControllerPolicy
{
    public function access(
        User $user,
    ): DecisionResult {
        if (!$user->isAdmin()) {
            return DecisionResult::deny(
                'admin.access_denied'
            );
        }

        return DecisionResult::grant();
    }
}
```

---

# 138. Controller method example

```php
#[Authorize(
    ability: 'update',
    subject: 'invoice',
)]
public function update(
    Invoice $invoice
): Response {
    // autorización ya ejecutada
}
```

Resolución:

```text
Invoice
   ↓
InvoicePolicy
   ↓
update()
```

---

# 139. Controller-wide + Resource Policy

Podrán coexistir:

```text
AdminControllerPolicy::access()
```

y:

```text
InvoicePolicy::update()
```

para una misma request.

Ejemplo:

```text
AdminController access
        ↓
GRANT

Invoice update
        ↓
GRANT

Final
        ↓
GRANT
```

---

# 140. Controller-wide deny

Si:

```text
AdminControllerPolicy
        ↓
DENY
```

el planner podrá evitar ejecutar:

```text
InvoicePolicy
```

cuando la estrategia establezca:

```text
deny overrides
+
short circuit
```

---

# 141. Route Policy

Una ruta podrá ser subject de una Policy.

Ejemplo:

```text
route:admin.dashboard
       ↓
AdminRoutePolicy
```

No será un motor diferente.

---

# 142. Command Policy

```php
#[PolicyFor(RebuildSearchIndexCommand::class)]
final class RebuildSearchIndexPolicy
{
    public function execute(
        PrincipalInterface $principal,
    ): bool {
        // ...
    }
}
```

---

# 143. Job Policy

Podrá controlarse:

```text
dispatch
execute
retry
cancel
```

sobre Jobs.

Esto puede ser útil en paneles administrativos y sistemas empresariales.

---

# 144. Component Policy

Un componente podrá utilizar una Policy:

```text
AdminDashboardComponent
       ↓
DashboardPolicy
```

pero deberá recordarse:

```text
UI authorization is not backend security.
```

Ocultar un botón no sustituye proteger la operación real.

---

# 145. Policy inheritance

VoltStack deberá evitar depender demasiado de herencia de clases.

Podrá soportarse:

```php
abstract class BasePolicy
{
}
```

pero se favorecerá:

```text
composition
```

---

# 146. Subject inheritance resolution

Si:

```text
PremiumInvoice extends Invoice
```

podrá existir:

```text
PremiumInvoicePolicy
```

Si no:

```text
InvoicePolicy
```

podrá utilizarse como fallback según configuración.

---

# 147. Interface Policy

Ejemplo:

```php
#[PolicyFor(ExportableResource::class)]
final class ExportPolicy
{
}
```

Aplicable a:

```text
Invoice
Report
CustomerList
```

si implementan:

```text
ExportableResource
```

---

# 148. Policy precedence for inheritance

Recomendación:

```text
Exact Class Policy
        ↓
Parent Class Policy
        ↓
Interface Policies
        ↓
Global Policies
```

La prioridad explícita podrá modificar el orden de ejecución.

---

# 149. Multiple interface Policies

Si un Subject implementa:

```text
TenantOwned
SensitiveResource
Exportable
```

podrían participar:

```text
TenantIsolationPolicy
SensitiveResourcePolicy
ExportPolicy
```

Esta es una ventaja importante frente a mappings estrictamente uno-a-uno.

---

# 150. Policy discovery conflicts

Si existen simultáneamente:

```text
explicit registration
attribute registration
convention registration
```

deberán existir reglas claras.

Recomendación:

```text
Explicit
   >
Attribute
   >
Convention
```

---

# 151. Conflict detection

Si dos fuentes de igual precedencia intentan registrar una Policy incompatible:

```text
PolicyRegistrationConflictException
```

en lugar de seleccionar una silenciosamente.

---

# 152. Multiple registration no siempre es conflicto

Registrar:

```text
InvoicePolicy
FinancialCompliancePolicy
```

para `Invoice` es válido si el sistema soporta múltiples Policies.

El conflicto ocurre cuando metadata mutuamente exclusiva no puede reconciliarse.

---

# 153. Policy Registry API

Conceptualmente:

```php
$registry->register(
    subject: Invoice::class,
    policy: InvoicePolicy::class,
);

$registry->registerGlobal(
    SuspendedPrincipalPolicy::class,
);
```

---

# 154. Immutable compiled registry

Después del bootstrap productivo:

```text
PolicyRegistry
```

deberá ser preferentemente:

```text
immutable
```

Las aplicaciones no deberían modificar Policies dinámicamente durante una request.

---

# 155. Dynamic Policies

Si un caso empresarial requiere reglas dinámicas, deberán representarse como:

```text
data consumed by Policy
```

no registrando/desregistrando clases continuamente.

Ejemplo:

```text
Policy
 ↓
Database-backed Authorization Rules
```

---

# 156. Policy Versioning

Sistemas avanzados podrán asociar:

```text
policy version
```

a metadata o auditoría.

Ejemplo:

```text
InvoiceApprovalPolicy:v3
```

Esto puede ser útil para compliance.

No es requisito de V1.

---

# 157. Explainability

Una Policy avanzada deberá poder explicar:

```text
which rule denied
why
which policy
which ability
```

sin exponer información sensible al cliente.

---

# 158. Decision source

El Core podrá adjuntar internamente:

```text
policy:
InvoicePolicy

method:
approve

reasonCode:
invoice.amount_limit
```

al `DecisionResult`.

---

# 159. Policy trace

En desarrollo:

```text
Authorization
 ├── SuspendedPolicy → ABSTAIN
 ├── TenantPolicy → GRANT
 ├── InvoicePolicy::approve → DENY
 └── Final → DENY
```

Esto será muy útil para debugging.

---

# 160. Production tracing

En producción deberá poder reducirse a:

```text
request id
decision
policy ids
duration
reason codes
```

sin guardar información innecesaria.

---

# 161. Policy profiling

Podrá medirse:

```text
InvoicePolicy::update
2.3 ms
```

y detectar Policies costosas.

---

# 162. N+1 detection

El profiler podrá eventualmente advertir:

```text
InvoicePolicy::view
executed 250 times
database queries: 250
```

y recomendar batch authorization.

---

# 163. Policy Testing

Cada Policy deberá poder probarse sin HTTP.

Ejemplo:

```php
$result = $authorization->decide(
    ability: 'update',
    subject: $invoice,
    principal: $user,
    context: $context,
);
```

---

# 164. Unit testing simple Policy

También podrá invocarse directamente cuando se pruebe solo lógica pura:

```php
$policy = new InvoicePolicy();

expect(
    $policy->update($user, $invoice)
)->toBeTrue();
```

---

# 165. Prefer Core tests for integration

Cuando se quieran verificar:

```text
before()
multiple Policies
priorities
context
decision strategies
```

deberá probarse mediante:

```text
AuthorizationManager
```

---

# 166. Policy Test Helpers

Podrán existir:

```php
assertPolicyAllows(...);

assertPolicyDenies(...);

assertPolicyAbstains(...);
```

dentro de:

```text
Quantum\Authorization\Testing
```

---

# 167. Policy coverage

Herramientas futuras podrán detectar:

```text
Controller action requires ability update

but no Policy/Gate handles it
```

antes de producción.

---

# 168. Static Analysis

Podrá verificarse:

```text
Policy method signatures
Subject types
Principal types
Return types
Ability names
Missing methods
Invalid attributes
```

durante compilación.

---

# 169. Compile-time validation

Ejemplo:

```text
InvoicePolicy registered for Invoice
        ↓
method update(
    User $user,
    Post $post
)
```

deberá detectarse como inconsistencia.

---

# 170. Authorization linting

VoltStack podrá incorporar:

```text
volt authorization:lint
```

o herramienta equivalente para detectar:

```text
unhandled abilities
ambiguous mappings
unsafe Policies
invalid return signatures
Policy conflicts
```

---

# 171. Policy generation

CLI futuro:

```text
volt make:policy InvoicePolicy --subject=Invoice
```

podrá generar:

```php
final class InvoicePolicy
{
    public function view(...): bool {}

    public function create(...): bool {}

    public function update(...): bool {}

    public function delete(...): bool {}
}
```

---

# 172. Controller Policy generation

```text
volt make:policy AdminControllerPolicy
    --subject=AdminController
```

deberá ser igualmente válido.

Esto refleja que Controllers son subjects legítimos.

---

# 173. Policy security invariant

Una Policy nunca deberá poder autorizar accidentalmente por:

```text
missing return
```

Ejemplo:

```php
public function delete(...)
{
    if ($condition) {
        return true;
    }
}
```

El retorno implícito:

```text
null
```

podría significar `ABSTAIN`, no `GRANT`.

Default Deny protegerá el resultado final.

---

# 174. Strict Policy Mode

Podrá existir:

```text
strict_policy_returns = true
```

En este modo, métodos Resource Policy declarados como decisivos deberán retornar explícitamente:

```text
GRANT
or
DENY
```

y un `null` inesperado será error.

---

# 175. Declarative abstention

Para Policies composables será preferible declarar explícitamente:

```php
DecisionResult::abstain();
```

en lugar de depender de `null`.

---

# 176. Policy mutation protection

El framework no puede impedir toda mutación PHP, pero la documentación deberá establecer:

```text
Policies must not mutate Principal or Subject.
```

Una Policy observa estado y produce una decisión.

---

# 177. Transaction boundary

Authorization normalmente ocurre antes de ejecutar la operación.

Ejemplo:

```text
Authorize
   ↓
Begin/execute business transaction
```

Sin embargo, ciertas operaciones podrán requerir autorización dentro de una transacción para evitar race conditions.

---

# 178. TOCTOU

Debe considerarse:

```text
Time Of Check
vs
Time Of Use
```

Ejemplo:

```text
check invoice status
      ↓
status changes
      ↓
update executes
```

Las reglas críticas deberán revalidarse dentro del dominio/transacción cuando sea necesario.

---

# 179. Policy no reemplaza invariantes de dominio

Una Policy puede decidir:

```text
User may attempt to approve invoice
```

pero la entidad/servicio de dominio todavía debe proteger invariantes como:

```text
invoice cannot be approved twice
```

Authorization y domain invariants son capas diferentes.

---

# 180. Controller Authorization Example

```php
#[Authorize('access')]
final class BillingAdminController
{
    #[Authorize(
        ability: 'approve',
        subject: 'invoice',
    )]
    public function approve(
        Invoice $invoice
    ): Response {
        // ...
    }
}
```

Esto podría generar dos solicitudes:

```text
Request A
Ability:
access

Subject:
BillingAdminController

Request B
Ability:
approve

Subject:
Invoice#928
```

---

# 181. Policy Plan para el ejemplo

```text
Authorization Plan A
 ├── SuspendedPrincipalPolicy
 ├── AdminControllerPolicy
 └── SecurityPolicy

Authorization Plan B
 ├── SuspendedPrincipalPolicy
 ├── TenantIsolationPolicy
 ├── InvoicePolicy
 └── FinancialCompliancePolicy
```

Ambas utilizan el mismo Core Engine.

---

# 182. Policy composition diagram

```text
                     AuthorizationRequest
                              │
                              ▼
                     AuthorizationPlanner
                              │
          ┌───────────────────┼───────────────────┐
          ▼                   ▼                   ▼
    Global Policies     Subject Policies    Context Policies
          │                   │                   │
          ▼                   ▼                   ▼
       Evaluator           Evaluator           Evaluator
          │                   │                   │
          └───────────────────┼───────────────────┘
                              ▼
                       DecisionResults
                              │
                              ▼
                       DecisionManager
                              │
                              ▼
                         Final Result
```

---

# 183. Policy execution lifecycle

```text
1. Resolve applicable PolicyDescriptor
2. Resolve Policy instance
3. Check Policy-level before()
4. Resolve Ability method
5. Map Principal
6. Map Subject
7. Map Context arguments
8. Invoke Policy method
9. Normalize return
10. Attach decision metadata
11. Run observational after hook
12. Return DecisionResult
```

---

# 184. `before()` short-circuit

Dentro de una Policy:

```text
before()
  ↓
GRANT / DENY
  ↓
skip ability method
```

Si:

```text
ABSTAIN
```

continúa:

```text
ability method
```

---

# 185. Policy result lifecycle

```text
bool|null|Decision|PolicyResponse|DecisionResult
                     │
                     ▼
             PolicyReturnNormalizer
                     │
                     ▼
               DecisionResult
                     │
                     ▼
             Evaluator Metadata
                     │
                     ▼
              DecisionManager
```

---

# 186. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    ├── Policies/
    │   ├── Contracts/
    │   │   ├── PolicyInterface.php
    │   │   ├── BatchPolicyInterface.php
    │   │   └── SupportsAuthorizationRequest.php
    │   │
    │   ├── PolicyRegistry.php
    │   ├── PolicyResolver.php
    │   ├── PolicyEvaluator.php
    │   ├── PolicyDispatcher.php
    │   ├── PolicyReturnNormalizer.php
    │   ├── PolicyDescriptor.php
    │   ├── PolicyMethodDescriptor.php
    │   ├── PolicyInvocationDescriptor.php
    │   ├── PolicyType.php
    │   │
    │   ├── Attributes/
    │   │   ├── PolicyFor.php
    │   │   ├── HandlesAbility.php
    │   │   ├── Priority.php
    │   │   ├── Auditable.php
    │   │   └── Memoizable.php
    │   │
    │   ├── Global/
    │   ├── Security/
    │   ├── Tenant/
    │   └── Context/
    │
    ├── Decisions/
    │   ├── Decision.php
    │   ├── DecisionResult.php
    │   └── PolicyResponse.php
    │
    └── Exceptions/
        ├── PolicyException.php
        ├── PolicyResolutionException.php
        ├── PolicyRegistrationConflictException.php
        ├── PolicyMethodNotFoundException.php
        ├── PolicyExecutionException.php
        └── InvalidPolicyResultException.php
```

---

# 187. Invariantes del Policy System

### Invariante 1

Toda Policy participa dentro del Authorization Core Engine.

### Invariante 2

Toda salida de Policy se normaliza a `DecisionResult`.

### Invariante 3

Una Policy no autentica al Principal.

### Invariante 4

Una Policy no ejecuta la operación que está autorizando.

### Invariante 5

Una Policy debe evitar efectos secundarios.

### Invariante 6

Una excepción nunca implica autorización.

### Invariante 7

La ausencia de Policy aplicable nunca implica autorización.

### Invariante 8

Controllers pueden ser Subjects legítimos.

### Invariante 9

Controller Policies no requieren un Authorization Engine separado.

### Invariante 10

Múltiples Policies pueden participar en una misma decisión.

### Invariante 11

`ABSTAIN` no equivale a `GRANT`.

### Invariante 12

Las Policies no deberán almacenar estado mutable de request.

### Invariante 13

Las Policies de UI no sustituyen la autorización backend.

### Invariante 14

Las Policies no sustituyen las invariantes del dominio.

### Invariante 15

El orden de Policies deberá ser determinista.

---

# 188. Filosofía del Policy System

La filosofía deberá ser:

```text
Simple to write.
Explicit to register.
Predictable to execute.
Composable by design.
Explainable when needed.
Safe by default.
```

Para el desarrollador:

```php
final class InvoicePolicy
{
    public function update(
        User $user,
        Invoice $invoice
    ): bool {
        return $invoice->user_id === $user->id;
    }
}
```

Para VoltStack:

```text
PolicyDescriptor
       ↓
PolicyEvaluator
       ↓
PolicyDispatcher
       ↓
PolicyReturnNormalizer
       ↓
DecisionResult
       ↓
DecisionManager
```

---

# 189. Diferenciador arquitectónico

VoltStack no limitará el concepto de Policy a:

```text
User + ORM Model
```

El modelo general será:

```text
Principal
   +
Ability
   +
Any Subject
   +
AuthorizationContext
        ↓
Policies
```

Por ello será posible aplicar Policies a:

```text
Invoice
Post
Controller
Controller Action
Route
Command
Job
Component
Tenant
Service
Virtual Resource
System Operation
```

sin cambiar el Core.

---

# 190. Resultado esperado

El Policy System deberá ofrecer una experiencia familiar para desarrolladores provenientes de Laravel:

```php
$user->can('update', $post);
```

```php
final class PostPolicy
{
    public function update(
        User $user,
        Post $post
    ): bool {
        return $user->id === $post->user_id;
    }
}
```

mientras proporciona capacidades arquitectónicas más amplias:

```text
Multiple Policies
Global Policies
Controller Policies
Security Policies
Tenant Policies
Context-Aware Policies
GRANT / DENY / ABSTAIN
Decision Strategies
Explainable Decisions
Compiled Metadata
Persistent Runtime Safety
Batch Authorization
ABAC
ReBAC
RBAC Integration
```

El principio definitivo será:

```text
A Policy expresses an authorization rule.

It does not own the authorization engine.

Every Policy, regardless of what it protects,
participates in the same normalized
AuthorizationRequest → DecisionResult pipeline.
```

De esta manera, VoltStack podrá ofrecer Policies sencillas para el desarrollo cotidiano y, al mismo tiempo, una infraestructura capaz de soportar sistemas de autorización empresariales complejos sin fragmentar el framework en mecanismos incompatibles.