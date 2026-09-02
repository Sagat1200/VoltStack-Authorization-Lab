# VoltStack Authorization System — Role, Permission, RBAC, ABAC and ReBAC Integration System

## 1. Propósito

Este documento define la integración de **Roles, Permissions, RBAC, ABAC y ReBAC** dentro del Authorization System de VoltStack.

El objetivo es evitar que estos modelos se conviertan en motores independientes y fragmentados.

VoltStack deberá permitir combinar:

```text
RBAC
Role-Based Access Control

ABAC
Attribute-Based Access Control

ReBAC
Relationship-Based Access Control

Policies

Gates

Security Evaluators

Tenant Policies
```

dentro de una única solicitud:

```text
AuthorizationRequest
```

y una única infraestructura de decisión:

```text
AuthorizationPlanner
        ↓
Authorization Evaluators
        ↓
DecisionManager
```

Ejemplo:

```text
Principal:
User#42

Ability:
invoice.approve

Subject:
Invoice#928

Requirements:

Role:
finance-manager

Permission:
invoice.approve

ABAC:
invoice.amount <= user.approval_limit

ReBAC:
user belongs to organization owning invoice

Tenant:
same tenant

Resource Policy:
InvoicePolicy::approve
```

Todo deberá converger en:

```text
GRANT
DENY
ABSTAIN
```

---

# 2. Principio arquitectónico

VoltStack no deberá implementar:

```text
RoleAuthorizationEngine
PermissionAuthorizationEngine
AbacAuthorizationEngine
RebacAuthorizationEngine
PolicyAuthorizationEngine
```

como sistemas independientes.

La arquitectura será:

```text
RBAC Evaluator
ABAC Evaluator
ReBAC Evaluator
Policy Evaluator
Gate Evaluator
Security Evaluator
        │
        ↓
Unified Authorization Engine
```

Por tanto:

```text
Role ≠ Authorization Engine

Permission ≠ Authorization Engine

Relationship ≠ Authorization Engine

Attribute ≠ Authorization Engine
```

Son fuentes de información y reglas utilizadas por evaluadores.

---

# 3. Modelo general

```text
                    AuthorizationRequest
                            │
          ┌─────────────────┼─────────────────┐
          ↓                 ↓                 ↓
        RBAC               ABAC              ReBAC
          │                 │                 │
          ↓                 ↓                 ↓
     Role/Permission     Attribute       Relationship
       Evaluators        Evaluators       Evaluators
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ↓
                     DecisionResults
                            ↓
                     DecisionManager
                            ↓
                    Final Authorization
```

---

# 4. Objetivos

El subsistema deberá proporcionar:

- Roles;
- Permissions;
- asignación Role → Permission;
- asignación Principal → Role;
- permisos directos;
- scopes;
- tenant-aware roles;
- tenant-aware permissions;
- wildcards opcionales;
- inheritance opcional;
- cache seguro;
- request memoization;
- evaluadores RBAC;
- reglas ABAC;
- evaluadores de atributos;
- integración ReBAC;
- relaciones directas e indirectas;
- autorización híbrida;
- metadata compilada;
- integración con `#[RequiresRole]`;
- integración con `#[RequiresPermission]`;
- debugging;
- explainability;
- auditoría;
- extensibilidad.

---

# 5. RBAC

RBAC representa:

```text
Principal
    ↓ assigned
Role
    ↓ grants
Permission
```

Ejemplo:

```text
User#42
   ↓
finance-manager
   ↓
invoice.view
invoice.approve
invoice.export
```

---

# 6. Role

Un Role representa una agrupación lógica de capacidades.

Ejemplos:

```text
administrator
finance-manager
support-agent
auditor
developer
billing-operator
```

---

# 7. Role Value Object

Conceptualmente:

```php
final readonly class Role
{
    public function __construct(
        public string $name,
    ) {}
}
```

En aplicaciones con Roles persistidos podrá existir una entidad separada.

---

# 8. Role Name

El nombre deberá ser canónico.

Formato recomendado:

```text
administrator
finance.manager
support.agent
tenant.owner
```

---

# 9. Role Identity

En sistemas simples:

```text
role name
```

puede ser suficiente.

En sistemas empresariales:

```text
Role ID
+
canonical name
+
scope
```

podrá ser necesario.

---

# 10. Permission

Una Permission representa una capacidad concedida.

Ejemplos:

```text
invoice.view
invoice.update
invoice.approve
invoice.delete
customer.export
system.deploy
```

---

# 11. Permission Value Object

```php
final readonly class Permission
{
    public function __construct(
        public string $name,
    ) {}
}
```

---

# 12. Permission vs Ability

Esta distinción es crítica.

```text
Ability
=
la operación que se intenta ejecutar
```

```text
Permission
=
una capacidad asignada al Principal
```

Ejemplo:

```text
Ability:
approve

Subject:
Invoice#928

Permission:
invoice.approve
```

---

# 13. No equivalencia obligatoria

VoltStack podrá permitir:

```text
Ability name
=
Permission name
```

cuando resulte conveniente.

Pero no deberá asumirlo arquitectónicamente.

---

# 14. Ejemplo de mapping

```text
Ability:
approve

Subject:
Invoice

Mapped Permission:
invoice.approve
```

Este mapping puede definirse mediante:

```text
Policy
Ability metadata
PermissionRequirement
```

---

# 15. RoleRegistry

Roles definidos por código podrán registrarse mediante:

```text
RoleRegistry
```

Pero el sistema también deberá soportar Roles dinámicos almacenados en DB.

---

# 16. PermissionRegistry

Podrá existir un catálogo de Permissions conocidas.

Beneficios:

```text
validation
IDE support
documentation
linting
authorization coverage
```

---

# 17. Registry no contiene assignments

Debe distinguirse:

```text
PermissionRegistry
=
qué permissions existen
```

de:

```text
PermissionAssignmentRepository
=
qué Principal posee qué permission
```

---

# 18. Role Repository

Contrato conceptual:

```php
interface RoleRepositoryInterface
{
    public function rolesFor(
        PrincipalInterface $principal,
        AuthorizationContext $context,
    ): iterable;
}
```

---

# 19. Permission Repository

```php
interface PermissionRepositoryInterface
{
    public function permissionsFor(
        PrincipalInterface $principal,
        AuthorizationContext $context,
    ): iterable;
}
```

---

# 20. Direct Permission Lookup

Para evitar cargar todas las permissions:

```php
interface PermissionCheckerInterface
{
    public function has(
        PrincipalInterface $principal,
        Permission|string $permission,
        AuthorizationContext $context,
    ): bool;
}
```

---

# 21. Role Checker

```php
interface RoleCheckerInterface
{
    public function has(
        PrincipalInterface $principal,
        Role|string $role,
        AuthorizationContext $context,
    ): bool;
}
```

---

# 22. Principal Interface no debe acoplarse

No se recomienda obligar:

```php
PrincipalInterface
```

a implementar:

```php
roles()
permissions()
```

porque algunos Principals pueden obtener autorización desde:

```text
LDAP
SSO
external IAM
API token
service registry
database
```

---

# 23. Adapter architecture

Preferido:

```text
Principal
    ↓
RoleProvider
PermissionProvider
```

---

# 24. User convenience APIs

Una aplicación sí podrá ofrecer:

```php
$user->hasRole('admin');

$user->hasPermission('invoice.approve');
```

pero internamente deberían delegar al subsistema correspondiente.

---

# 25. No recursion accidental

Si:

```php
$user->can(...)
```

usa AuthorizationManager, y:

```php
$user->hasPermission(...)
```

también usa el mismo Manager de forma recursiva, deberá evitarse un ciclo.

Los checkers de RBAC deberán operar a un nivel inferior.

---

# 26. PrincipalRoleAssignment

Conceptualmente:

```text
Principal
+
Role
+
Scope
+
Validity
```

---

# 27. Scoped Roles

Un Role podrá existir únicamente dentro de un scope.

Ejemplo:

```text
User#42
Role:
manager

Scope:
Tenant#7
```

No necesariamente en:

```text
Tenant#9
```

---

# 28. Scope Model

Podrá representar:

```text
Global
Tenant
Organization
Project
Resource
Custom
```

---

# 29. AuthorizationScope

Conceptualmente:

```php
final readonly class AuthorizationScope
{
    public function __construct(
        public AuthorizationScopeType $type,
        public string|int|null $identifier,
    ) {}
}
```

---

# 30. Scope Type

```php
enum AuthorizationScopeType: string
{
    case Global = 'global';
    case Tenant = 'tenant';
    case Organization = 'organization';
    case Project = 'project';
    case Resource = 'resource';
    case Custom = 'custom';
}
```

---

# 31. Global Role

```text
administrator
scope=global
```

---

# 32. Tenant Role

```text
tenant.admin
scope=Tenant#7
```

---

# 33. Project Role

```text
project.manager
scope=Project#81
```

---

# 34. Role Assignment

Conceptualmente:

```php
final readonly class RoleAssignment
{
    public function __construct(
        public string $role,
        public AuthorizationScope $scope,
    ) {}
}
```

---

# 35. Permission Assignment

Puede modelarse igual:

```text
Principal
+
Permission
+
Scope
```

---

# 36. Direct Permissions

VoltStack deberá soportar:

```text
Role-derived permissions
```

y:

```text
direct permissions
```

---

# 37. Example

```text
User#42

Roles:
finance.manager

Direct Permission:
invoice.override-limit
```

---

# 38. Permission Resolution

Conceptualmente:

```text
Direct Permissions
        +
Role Permissions
        +
External Permission Providers
        ↓
Effective Permissions
```

---

# 39. Effective Permission Set

No siempre será necesario materializar todo el set.

Podrá evaluarse mediante lookup.

---

# 40. Role → Permission Mapping

Ejemplo:

```text
finance.manager

grants:

invoice.view
invoice.create
invoice.update
invoice.approve
```

---

# 41. RolePermissionRepository

```php
interface RolePermissionRepositoryInterface
{
    public function roleHasPermission(
        Role|string $role,
        Permission|string $permission,
        AuthorizationScope $scope,
    ): bool;
}
```

---

# 42. Role inheritance

VoltStack podrá soportar opcionalmente:

```text
administrator
   ↓ inherits
manager
   ↓ inherits
employee
```

---

# 43. Recommendation

No activar inheritance implícita por defecto.

Las jerarquías de Roles pueden hacer difícil explicar permisos.

---

# 44. Explicit inheritance

Si se soporta:

```text
RoleGraph
```

deberá registrarse explícitamente.

---

# 45. Circular Roles

Ejemplo inválido:

```text
admin → manager
manager → supervisor
supervisor → admin
```

deberá detectarse durante configuración.

---

# 46. Role Graph Compilation

Las jerarquías estáticas podrán aplanarse.

Ejemplo:

```text
administrator
    ↓
[
  employee,
  manager,
  administrator
]
```

para lookup rápido.

---

# 47. Dynamic Roles

Si los Roles son DB-backed, el graph podrá cachearse con invalidación.

---

# 48. Permission inheritance

No se recomienda usar jerarquía implícita basada en dots.

Esto:

```text
invoice.*
```

no deberá concederse automáticamente porque existe:

```text
invoice.manage
```

salvo configuración explícita.

---

# 49. Wildcard Permissions

Podrán soportarse:

```text
invoice.*
```

para conceder:

```text
invoice.view
invoice.update
invoice.approve
```

---

# 50. Wildcard semantics

Deberán ser precisas.

```text
invoice.*
```

aplica a:

```text
invoice.view
invoice.update
```

No necesariamente:

```text
invoice.item.view
```

salvo que la implementación lo defina explícitamente.

---

# 51. Recommendation

Utilizar wildcard matching segment-aware.

---

# 52. Wildcards compile

Para Roles estáticos:

```text
invoice.*
```

podrá compilarse a estructuras optimizadas.

---

# 53. Wildcards dinámicos

Para DB permissions podrán usarse índices prefix-aware o patrones normalizados.

---

# 54. No arbitrary regex

No se recomienda permitir expresiones regex arbitrarias como permissions.

Dificultan:

```text
performance
analysis
security review
```

---

# 55. Permission Denials

Además de grants, sistemas avanzados pueden necesitar:

```text
explicit deny
```

Ejemplo:

```text
Role:
manager

grants:
invoice.*

Direct deny:
invoice.force_delete
```

---

# 56. Recommendation

V1 puede implementar solo grants.

`DENY` explícito podrá añadirse después si existe necesidad clara.

---

# 57. Why

Añadir deny assignments introduce:

```text
precedence
scope
inheritance
conflict
```

más complejos.

---

# 58. Permission Evaluator

La integración principal con Authorization Engine será mediante:

```text
PermissionEvaluator
```

---

# 59. PermissionEvaluator Contract

Conceptualmente:

```php
final class PermissionEvaluator
{
    public function evaluate(
        AuthorizationRequest $request,
        PermissionRequirement $requirement,
    ): DecisionResult {
        // ...
    }
}
```

---

# 60. Permission requirement result

Si posee permiso:

```text
GRANT
```

Si no posee permiso:

```text
DENY
```

cuando el requirement es obligatorio.

---

# 61. Permission voter vs no requirement

No deberá ejecutarse un PermissionEvaluator globalmente para toda request si no existe requirement o mapping relevante.

---

# 62. RoleEvaluator

Similar:

```text
Role requirement
      ↓
RoleEvaluator
```

---

# 63. Role Match Any

Ejemplo:

```text
admin
OR
finance-manager
```

si cualquiera existe:

```text
GRANT
```

---

# 64. Role Match All

```text
admin
AND
finance-manager
```

requiere ambos.

---

# 65. Permission Match Any/All

La misma semántica aplicará.

---

# 66. Integration with Attributes

Esto:

```php
#[RequiresRole('administrator')]
```

compila a:

```text
RoleRequirementDescriptor
```

y finalmente:

```text
RoleEvaluatorInvocation
```

---

# 67. RequiresPermission

Esto:

```php
#[RequiresPermission('invoice.approve')]
```

compila a:

```text
PermissionRequirementDescriptor
```

---

# 68. Planner Integration

```text
Compiled Metadata
      ↓
AuthorizationPlanner
      ↓
RoleEvaluator
PermissionEvaluator
```

---

# 69. Strategy

Por defecto estos evaluadores deberán ser:

```text
Required
```

en una declaración como:

```text
#[RequiresPermission]
```

---

# 70. Hybrid authorization

Ejemplo:

```php
#[RequiresRole('finance-manager')]
#[RequiresPermission('invoice.approve')]
#[Authorize('approve', subject: 'invoice')]
```

Plan:

```text
RoleEvaluator
PermissionEvaluator
TenantIsolationPolicy
InvoicePolicy
CompliancePolicy
```

---

# 71. Default semantics

Todos los requirements explícitos deberán pasar.

---

# 72. RBAC is coarse-grained

RBAC es especialmente adecuado para responder:

```text
¿qué categoría de operaciones puede ejecutar este Principal?
```

---

# 73. Policies are fine-grained

Policies pueden responder:

```text
¿puede ejecutar esta operación
sobre este recurso concreto?
```

---

# 74. Example

```text
Permission:
invoice.update
      ↓
GRANT

InvoicePolicy:
user belongs to invoice organization?
      ↓
DENY

Final:
DENY
```

---

# 75. ABAC

ABAC utiliza atributos de:

```text
Principal
Subject
Environment
Context
Action
```

para tomar decisiones.

---

# 76. ABAC general model

```text
Principal Attributes
+
Subject Attributes
+
Action
+
Context Attributes
        ↓
Rule
        ↓
Decision
```

---

# 77. Example ABAC

```text
Principal.department = finance

Subject.classification = confidential

Context.region = MX

Ability = view
```

Rule:

```text
finance users in MX
may view confidential financial reports
```

---

# 78. ABAC no debe requerir DSL inicialmente

V1 podrá implementar ABAC dentro de Policies normales.

Ejemplo:

```php
final class FinancialReportPolicy
{
    public function view(
        User $user,
        FinancialReport $report,
        AuthorizationContext $context,
    ): bool {
        return $user->department === 'finance'
            && $report->classification === 'confidential'
            && $context->attribute('region') === 'MX';
    }
}
```

---

# 79. Declarative ABAC

En versiones avanzadas podrá existir:

```text
AttributeRuleEvaluator
```

para reglas declarativas.

---

# 80. AttributeProvider

Para desacoplar ABAC de entidades concretas podrá existir:

```php
interface AuthorizationAttributeProviderInterface
{
    public function attributesFor(
        mixed $target,
        AuthorizationContext $context,
    ): AuthorizationAttributeBag;
}
```

---

# 81. Principal Attributes

Ejemplos:

```text
department
clearance_level
employment_type
country
risk_level
account_state
```

---

# 82. Subject Attributes

Ejemplos:

```text
classification
owner
department
region
status
sensitivity
amount
```

---

# 83. Context Attributes

Ejemplos:

```text
channel
time zone
security level
MFA
network zone
tenant
risk score
```

---

# 84. Ability attributes

La propia Ability podrá tener metadata:

```text
risk=critical
category=financial
requires_mfa=true
```

---

# 85. AuthorizationAttributeBag

Conceptualmente:

```php
final readonly class AuthorizationAttributeBag
{
    public function __construct(
        private array $attributes,
    ) {}
}
```

---

# 86. Typed Attributes

Para atributos sensibles/importantes se recomienda tipos dedicados.

No depender exclusivamente de arrays dinámicos.

---

# 87. Attribute Namespace

Ejemplos:

```text
principal.department
subject.classification
context.region
ability.risk
```

---

# 88. Trusted Attributes

No todos los atributos tienen igual nivel de confianza.

Ejemplo:

```text
context.mfa_verified
```

debe provenir de SecurityContext.

No de:

```text
HTTP query string
```

---

# 89. Trusted Attribute Providers

El sistema podrá distinguir providers registrados como confiables.

---

# 90. ABAC rule input safety

Una regla de autorización nunca deberá confiar directamente en:

```text
X-Role
X-Clearance
X-Tenant
```

sin validación previa.

---

# 91. Attribute Rule

Conceptualmente:

```php
interface AuthorizationRuleInterface
{
    public function evaluate(
        AuthorizationRuleContext $context,
    ): DecisionResult;
}
```

---

# 92. Rule Context

Podría contener:

```text
Principal attributes
Subject attributes
Ability metadata
AuthorizationContext
```

---

# 93. ABAC rule example

```text
Rule:
principal.clearance >= subject.classification_level
```

---

# 94. Declarative expression engine

VoltStack podría implementar en el futuro una DSL como:

```text
principal.clearance >= subject.classification
AND
context.region == subject.region
```

Pero no debe introducirse de forma apresurada.

---

# 95. Security risks of expression engines

Una DSL mal diseñada puede introducir:

```text
code execution
unsafe property access
unexpected coercion
performance issues
```

---

# 96. Recommendation

V1:

```text
ABAC via typed Policies / Rule Objects
```

Posteriormente:

```text
safe declarative expression language
```

si existe necesidad.

---

# 97. ABAC cacheability

Una regla ABAC depende de atributos concretos.

Una decisión no puede cachearse solo por:

```text
Principal ID
+
Ability
+
Subject ID
```

si atributos relevantes pueden cambiar.

---

# 98. Relevant Attribute Declaration

Futuro:

```text
Policy declares relevant attributes
```

podrá ayudar con memoization.

---

# 99. ReBAC

ReBAC autoriza según relaciones entre entidades.

Ejemplo:

```text
User
   ↓ member of
Organization
   ↓ owns
Project
```

Entonces:

```text
User may manage Project
```

---

# 100. Relationship Types

Ejemplos:

```text
owner
member
manager
editor
viewer
parent
child
belongs_to
assigned_to
delegated_to
```

---

# 101. Relationship Tuple

Modelo conceptual:

```text
Subject
Relation
Object
```

Ejemplo:

```text
user:42
member
organization:7
```

---

# 102. Tuple model

Podrá representarse:

```php
final readonly class RelationshipTuple
{
    public function __construct(
        public ResourceReference $subject,
        public string $relation,
        public ResourceReference $object,
    ) {}
}
```

---

# 103. ResourceReference

Representación serializable:

```text
type
identifier
```

Ejemplo:

```text
user:42
organization:7
project:81
```

---

# 104. Relationship Repository

```php
interface RelationshipRepositoryInterface
{
    public function has(
        ResourceReference $subject,
        string $relation,
        ResourceReference $object,
        AuthorizationContext $context,
    ): bool;
}
```

---

# 105. Direct relationship

```text
User#42
owner
Project#81
```

Lookup directo.

---

# 106. Indirect relationship

Ejemplo:

```text
User#42
member
Organization#7

Organization#7
owner
Project#81
```

Authorization:

```text
User#42 manages Project#81
```

puede requerir graph traversal.

---

# 107. Relationship Graph

Podrá existir:

```text
RelationshipGraph
```

para resolver caminos.

---

# 108. ReBAC Policy

Inicialmente ReBAC puede expresarse mediante Policy.

Ejemplo:

```php
final class ProjectPolicy
{
    public function manage(
        User $user,
        Project $project,
    ): bool {
        return $this->relationships->existsPath(
            subject: ResourceReference::user($user->id),
            relation: 'member_of_owner',
            object: ResourceReference::project($project->id),
        );
    }
}
```

---

# 109. RelationshipVoter

Para sistemas avanzados podrá existir un evaluator específico:

```text
RelationshipVoter
```

---

# 110. Relationship Requirement

Ejemplo conceptual:

```text
requires relationship:
principal member_of organization owning subject
```

---

# 111. Declarative ReBAC

Futuro:

```php
#[RequiresRelationship(
    relation: 'editor',
    subject: 'document'
)]
```

No es obligatorio para V1.

---

# 112. Relationship DSL

Podría soportar expresiones:

```text
document.owner
document.organization.member
project.parent.organization.admin
```

pero debe diseñarse con límites.

---

# 113. Graph depth

ReBAC deberá limitar traversal.

Ejemplo:

```text
max_relationship_depth
```

para evitar:

```text
cycles
unbounded queries
DoS
```

---

# 114. Cycle detection

Si el graph contiene:

```text
A → B → C → A
```

la resolución no deberá entrar en loop.

---

# 115. Relationship Indexes

Storage deberá estar indexado por:

```text
subject
relation
object
```

dependiendo de los patrones de consulta.

---

# 116. Database implementation

Una tabla conceptual:

```text
authorization_relationships

subject_type
subject_id
relation
object_type
object_id
tenant_id
```

podría utilizarse.

---

# 117. ReBAC and ORM

El Authorization Core no deberá depender de una implementación ORM concreta.

El Relationship Repository abstrae acceso.

---

# 118. ReBAC and domain relations

También podrá adaptarse a relaciones ya existentes.

Ejemplo:

```text
project_members
organization_members
document_owners
```

sin obligar a duplicarlas en una tabla genérica.

---

# 119. Relationship Adapters

Podrán existir:

```text
OrmRelationshipProvider
DomainRelationshipProvider
GenericTupleStore
RemoteRelationshipProvider
```

---

# 120. ReBAC performance

Graph traversal puede ser costoso.

Deberá soportar:

```text
memoization
batch lookup
precomputed relations
indexed storage
```

---

# 121. Precomputed Relations

Ejemplo:

```text
effective_project_admin
```

puede almacenarse para evitar traversal profundo.

---

# 122. Materialized authorization edges

Sistemas grandes podrán mantener:

```text
computed authorization relationships
```

actualizadas por eventos.

---

# 123. Eventual consistency

Si se utilizan relaciones materializadas, deberá definirse si autorización tolera eventual consistency.

Para operaciones críticas puede requerirse lectura fuerte.

---

# 124. Remote ReBAC

VoltStack podrá integrar en el futuro servicios tipo:

```text
relationship-based policy decision point
```

mediante un External Evaluator.

---

# 125. Timeout safety

Un servicio ReBAC remoto que falle:

```text
FAILED
```

no:

```text
GRANT
```

---

# 126. Hybrid RBAC + ABAC + ReBAC

El mayor valor del sistema será combinar modelos.

Ejemplo:

```text
Role:
finance-manager
        ↓
Permission:
invoice.approve
        ↓
ABAC:
amount <= approval limit
        ↓
ReBAC:
belongs to organization owning invoice
        ↓
Tenant:
same tenant
        ↓
InvoicePolicy:
invoice status allows approval
```

---

# 127. AuthorizationPlan example

```text
TenantIsolationPolicy
priority=9000

RoleEvaluator
priority=7000

PermissionEvaluator
priority=6500

RelationshipEvaluator
priority=6000

InvoicePolicy
priority=5000

FinancialAttributePolicy
priority=4500
```

---

# 128. Decision Strategy

Para este caso:

```text
DenyOverrides
```

puede ser apropiada.

---

# 129. Why hybrid

RBAC por sí solo no responde:

```text
¿este Invoice específico?
```

ABAC por sí solo puede no modelar jerarquía organizacional cómodamente.

ReBAC por sí solo no representa siempre permisos funcionales.

La combinación permite:

```text
coarse-grained
+
fine-grained
+
contextual
+
relational
```

authorization.

---

# 130. Policy as orchestrator

Una Policy podrá combinar todos estos servicios:

```php
final class InvoicePolicy
{
    public function __construct(
        private PermissionCheckerInterface $permissions,
        private RelationshipCheckerInterface $relationships,
    ) {}

    public function approve(
        User $user,
        Invoice $invoice,
        AuthorizationContext $context,
    ): DecisionResult {
        // ...
    }
}
```

---

# 131. Alternative evaluator composition

Para reglas transversales, el Planner podrá crear evaluadores separados.

Esto mejora:

```text
explainability
reuse
security isolation
```

---

# 132. When to keep logic in Policy

Usar Policy cuando las condiciones:

```text
belong strongly to resource domain semantics
```

Ejemplo:

```text
invoice.status == pending
invoice.owner organization
approval limit
```

---

# 133. When to use separate Evaluator

Usar evaluator transversal para:

```text
MFA
tenant isolation
global permission
account state
compliance requirement
```

---

# 134. Avoid over-fragmentation

No convertir cada condición simple en un evaluator independiente.

Eso podría producir:

```text
large plans
higher overhead
harder reasoning
```

---

# 135. Permission-to-Ability Mapping

Podrá definirse en Ability metadata:

```text
Ability:
invoice.approve

Required Permission:
invoice.approve
```

---

# 136. Automatic Permission Evaluator

El Planner podrá añadirlo automáticamente cuando Ability metadata declare:

```text
requiresPermission
```

---

# 137. Explicit metadata takes precedence

`#[RequiresPermission]` deberá seguir siendo posible.

---

# 138. Avoid automatic hidden permission requirements

Si una Ability tiene una permission implícita, tooling debe mostrarlo claramente.

---

# 139. AbilityDescriptor extension

Podrá incluir:

```php
public ?string $requiredPermission;
```

o una lista estructurada.

---

# 140. Role requirements on Ability

Menos recomendable.

Las Permissions son generalmente una mejor abstracción entre Roles y Abilities.

---

# 141. Recommended model

```text
Role
    ↓
Permission
    ↓
Ability requirement
```

No:

```text
Role
    ↓
hardcoded Ability
```

---

# 142. Roles as application semantics

Aun así:

```php
#[RequiresRole('supervisor')]
```

debe seguir disponible cuando el rol en sí es una condición del negocio.

---

# 143. Super Admin

Un `super-admin` no deberá implementarse mágicamente como:

```text
if role=super-admin → bypass all
```

dentro del RoleChecker.

---

# 144. Correct Super Admin model

Implementar:

```text
SuperAdminOverrideEvaluator
```

con estrategia explícita.

---

# 145. Why

Así puede decidirse qué no puede saltarse:

```text
tenant isolation
system lockdown
legal restrictions
```

---

# 146. Role does not imply bypass

Regla crítica:

```text
role assignment
≠
authorization engine override
```

---

# 147. Tenant-aware RBAC

Roles y Permissions deberán soportar tenancy.

Ejemplo:

```text
User#42
Tenant#7
Role=admin
```

no debe implicar admin en Tenant#9.

---

# 148. Current Tenant

El RoleChecker recibirá:

```text
AuthorizationContext
```

para conocer scope actual.

---

# 149. No global tenant accessor in shared services

No guardar:

```text
$currentTenant
```

como estado mutable global.

---

# 150. Tenant Scope Resolution

Podrá existir:

```text
AuthorizationScopeResolver
```

---

# 151. Scope selection

Para un Subject:

```text
Invoice#928
tenant=7
```

el evaluator puede resolver:

```text
TenantScope(7)
```

---

# 152. Context vs Subject scope conflict

Si:

```text
Context Tenant = 7
Subject Tenant = 9
```

deberá existir un DENY previo de aislamiento.

El PermissionChecker no deberá resolverlo silenciosamente.

---

# 153. Organization-scoped Permissions

Ejemplo:

```text
invoice.approve
scope=Organization#10
```

---

# 154. Project-scoped permissions

Ejemplo:

```text
deployment.execute
scope=Project#81
```

---

# 155. Hierarchical Scopes

Futuro:

```text
Tenant
  ↓
Organization
  ↓
Project
```

podrá permitir herencia de permisos si se define explícitamente.

---

# 156. Scope inheritance

Ejemplo:

```text
organization.admin
```

puede implicar:

```text
project.manage
```

sobre proyectos hijos.

Pero esto se acerca a ReBAC.

---

# 157. Recommendation

Modelar relaciones de scope con ReBAC en lugar de demasiada magia dentro de RBAC.

---

# 158. Permission conditions

Una permission assignment podría tener condiciones:

```text
invoice.approve
only below $50,000
```

Esto sería RBAC + ABAC.

---

# 159. Recommendation

No introducir condiciones arbitrarias dentro de PermissionAssignment V1.

Mantener:

```text
Permission grant
+
Policy/ABAC condition
```

separados.

---

# 160. Time-bound Role

Podrá existir:

```text
valid_from
valid_until
```

en assignments.

---

# 161. Role assignment validity

El repository deberá filtrar Roles expirados.

---

# 162. Temporary elevated access

Ejemplo:

```text
temporary incident-admin role
expires in 30 minutes
```

---

# 163. Break-glass

Debe ser:

```text
audited
time-bound
explicit
```

---

# 164. Delegation

Puede modelarse mediante:

```text
delegated role
delegated permission
relationship
```

dependiendo del caso.

---

# 165. Delegation metadata

Deberá conservar:

```text
grantor
grantee
scope
validity
reason
```

para auditoría.

---

# 166. Impersonation

Cuando un admin impersona:

```text
Actor:
Admin#1

Effective Principal:
User#42
```

RBAC normalmente evalúa Roles/Permissions de:

```text
Effective Principal
```

---

# 167. Actor-based Restrictions

Un evaluator separado puede impedir ciertas acciones durante impersonation.

Ejemplo:

```text
Impersonated sessions cannot change passwords.
```

---

# 168. Service Principals

RBAC no deberá limitarse a Users.

Ejemplo:

```text
ServicePrincipal:
billing-worker

Role:
billing.service

Permission:
invoice.process
```

---

# 169. API Clients

Igualmente:

```text
ApiClientPrincipal
```

puede poseer:

```text
scopes
permissions
roles
```

según implementación.

---

# 170. OAuth scopes

Un OAuth scope puede adaptarse a Permission.

Ejemplo:

```text
scope:
invoice:write
```

→

```text
Permission:
invoice.update
```

o mantenerse como evaluator separado.

---

# 171. Recommendation

No mezclar semánticamente protocolos externos con Permissions internas sin adapter.

---

# 172. External Role Mapping

SSO puede entregar:

```text
groups
roles
claims
```

que se mapearán a Roles internas.

---

# 173. Claim trust

Solo claims validados por Authentication System podrán participar.

---

# 174. Claim-to-Role mapping

Ejemplo:

```text
IdP group:
Finance Managers

→

Role:
finance-manager
```

---

# 175. Mapping registry

Podrá existir:

```text
ExternalIdentityAuthorizationMapper
```

---

# 176. Runtime Mapping

Idealmente ocurre durante construcción del Security/Principal Context, no en cada Policy.

---

# 177. Cache architecture

RBAC puede necesitar cache.

Podrán cachearse:

```text
Role definitions
Role → Permission mappings
Principal Role assignments
Principal direct permissions
```

con distintos scopes.

---

# 178. Static mappings

Role definitions y Role → Permission mappings pueden ser:

```text
process-wide
```

si vienen de configuración inmutable.

---

# 179. Dynamic assignments

Principal assignments normalmente requerirán:

```text
request-scoped memoization
```

o cache externo con invalidación.

---

# 180. Request Memoization

Ejemplo:

```text
User#42 has role finance-manager?
```

consultado 20 veces.

Solo debería resolverlo una vez por request.

---

# 181. RoleResolutionCache

Request-scoped:

```text
Principal ID
+
Scope
    ↓
Roles
```

---

# 182. PermissionResolutionCache

Igualmente:

```text
Principal
+
Permission
+
Scope
```

---

# 183. Persistent worker safety

Request caches deberán limpiarse después de cada request.

---

# 184. No static arrays keyed by user

Nunca:

```php
private static array $permissionsByUser;
```

sin lifecycle explícito.

---

# 185. Cross-request cache

Si se utiliza Redis u otro backend:

```text
cache key
```

deberá incluir:

```text
Principal
Tenant/Scope
Authorization version
```

---

# 186. Cache invalidation

Cambios en:

```text
role assignment
permission assignment
role permission mapping
scope membership
```

deberán invalidar entradas afectadas.

---

# 187. Authorization Version

Una técnica útil:

```text
principal.authorization_version
```

incrementada al cambiar grants.

---

# 188. Cache Key Example

```text
authz:principal:42:v17:tenant:7
```

---

# 189. No long-lived stale privileges

Evitar caches prolongados sin versioning para permisos sensibles.

---

# 190. Event-driven invalidation

Eventos:

```text
RoleAssigned
RoleRevoked
PermissionGranted
PermissionRevoked
RolePermissionChanged
RelationshipChanged
```

podrán invalidar cache.

---

# 191. Database transaction considerations

La invalidación debe coordinarse con commit.

No invalidar/emitir cambios antes de que la transacción se confirme.

---

# 192. Permission Change Consistency

Para revocaciones críticas, puede requerirse:

```text
strong consistency
```

---

# 193. Request-in-progress revocation

Una decisión ya tomada no puede revertir una operación que ya está ejecutándose automáticamente.

Operaciones críticas pueden revalidar cerca del commit.

---

# 194. TOCTOU

Roles y Permissions también están sujetas a:

```text
check
vs
use
```

---

# 195. Revalidation

Para operaciones como:

```text
wire transfer
production deployment
```

puede requerirse reauthorization inmediatamente antes de ejecutar.

---

# 196. Role database schema

Una implementación base podría incluir:

```text
authorization_roles

id
name
scope_type
metadata
created_at
updated_at
```

---

# 197. Permissions table

```text
authorization_permissions

id
name
metadata
created_at
updated_at
```

---

# 198. Role Permission pivot

```text
authorization_role_permissions

role_id
permission_id
```

---

# 199. Principal Roles

La dificultad es soportar múltiples tipos de Principal.

Modelo conceptual:

```text
authorization_principal_roles

principal_type
principal_id
role_id
scope_type
scope_id
valid_from
valid_until
```

---

# 200. Principal Permissions

```text
authorization_principal_permissions

principal_type
principal_id
permission_id
scope_type
scope_id
valid_from
valid_until
```

---

# 201. Polymorphic security

`principal_type` no deberá aceptar clases PHP arbitrarias provenientes de input.

La aplicación mantendrá un mapping seguro.

---

# 202. Principal Type Registry

Ejemplo:

```text
user
service
api_client
```

→ adapters internos.

---

# 203. Tenant Column

En sistemas multi-tenant podrá utilizarse:

```text
tenant_id
```

como parte del scope.

---

# 204. Database-level isolation

Los queries de RBAC deberán respetar Tenant Context.

---

# 205. Unique Constraints

Ejemplo:

```text
principal_type
principal_id
role_id
scope_type
scope_id
```

debería ser único cuando aplique.

---

# 206. Soft Delete

No se recomienda soft delete para grants si complica la revocación.

Puede utilizarse historial separado.

---

# 207. Assignment Audit

Para sistemas empresariales:

```text
who granted
when
why
who revoked
```

deberá almacenarse.

---

# 208. Separation from runtime authorization audit

Hay dos auditorías distintas:

```text
authorization configuration audit
```

y:

```text
authorization decision audit
```

---

# 209. Authorization Configuration Audit

Ejemplo:

```text
Admin#1 granted finance-manager to User#42
```

---

# 210. Decision Audit

Ejemplo:

```text
User#42 approved Invoice#928
authorization GRANT
```

---

# 211. Permission normalization

Debe validar:

```text
empty
whitespace
invalid control characters
oversized identifier
```

---

# 212. Case sensitivity

Recomendación:

```text
case-sensitive canonical lowercase
```

---

# 213. Role normalization

Igual.

---

# 214. Permission aliases

Podrán existir para migraciones.

Ejemplo:

```text
invoice.edit
    ↓
invoice.update
```

---

# 215. Role aliases

Menos recomendable, pero posible.

---

# 216. Alias compilation

Resolver antes del hot path.

---

# 217. Deprecated Permissions

Permission metadata podrá incluir:

```text
deprecated
replacement
```

---

# 218. Permission Migration Tooling

Comando futuro:

```text
volt authorization:permission-usage invoice.edit
```

---

# 219. Unknown Permission

En static registry mode:

```text
UnknownPermissionException
```

durante compile.

---

# 220. Dynamic Permission Mode

Sistemas donde los clientes crean permissions pueden permitir IDs desconocidos por compile.

Deberá ser configuración explícita.

---

# 221. Role Registry Modes

```text
Static
Database
Hybrid
External
```

---

# 222. Permission Registry Modes

Igualmente.

---

# 223. Authorization Provider

Puede abstraerse:

```php
interface AuthorizationGrantProviderInterface
{
    public function hasRole(...): bool;

    public function hasPermission(...): bool;
}
```

---

# 224. Multiple Providers

Ejemplo:

```text
Database Provider
LDAP Provider
Token Scope Provider
Package Provider
```

---

# 225. Provider chain

Cada provider podría responder:

```text
GRANTED
NOT_GRANTED
ABSTAIN
```

pero no deberá confundirse con el DecisionManager de autorización global.

---

# 226. Internal grant resolution

Podría existir un modelo más simple:

```text
contains / does not contain
```

y la composición de providers se mantiene interna al RBAC subsystem.

---

# 227. Avoid nested decision strategy complexity

No es necesario convertir cada RoleProvider en Voter global.

Solo exponer un `RoleEvaluator` normalizado.

---

# 228. Provider priority

Puede utilizarse para:

```text
local override
external fallback
```

pero debe ser determinista.

---

# 229. ABAC Provider Cache

Atributos caros, como:

```text
risk score
GeoIP
device trust
```

podrán resolverse lazy.

---

# 230. Context attribute lifetime

Debe ser snapshot durante una decisión.

---

# 231. Subject Attribute Provider

No deberá copiar entidades completas a arrays.

Solo extraer atributos necesarios.

---

# 232. Sensitive Attribute Redaction

Tracing ABAC no deberá registrar automáticamente:

```text
salary
medical data
full customer record
secret classification values
```

---

# 233. Explainability

ABAC trace puede mostrar:

```text
Rule:
approval_limit

Result:
DENY

Reason:
amount exceeds principal approval threshold
```

sin revelar necesariamente valores completos.

---

# 234. ReBAC Explainability

Ejemplo:

```text
User#42
member of Organization#7

Organization#7
owns Project#81

Therefore:
relationship path matched
```

---

# 235. Path Redaction

En producción quizá solo:

```text
relationship_required_not_satisfied
```

---

# 236. Authorization Explain Model

El Authorization Core podrá almacenar internamente:

```text
Evaluator
Rule
ReasonCode
```

y subsistemas RBAC/ABAC/ReBAC aportan detalles seguros.

---

# 237. RBAC Reason Codes

Ejemplos:

```text
rbac.role_missing
rbac.permission_missing
rbac.role_expired
rbac.permission_expired
rbac.scope_mismatch
```

---

# 238. ABAC Reason Codes

```text
abac.rule_failed
abac.attribute_missing
abac.clearance_insufficient
abac.region_mismatch
```

---

# 239. ReBAC Reason Codes

```text
rebac.relationship_missing
rebac.path_not_found
rebac.scope_mismatch
rebac.graph_limit_exceeded
```

---

# 240. Missing Attribute

Debe distinguirse:

```text
attribute absent
```

de:

```text
attribute present but fails condition
```

---

# 241. Missing trusted attribute

Si una Policy requiere:

```text
security.mfa
```

y falta por fallo del Security subsystem:

```text
configuration/security failure
```

no simplemente ABSTAIN.

---

# 242. Optional ABAC attribute

Si una regla solo aplica cuando existe:

```text
risk.score
```

puede abstenerse según metadata.

---

# 243. Graph Limit Failure

Si ReBAC supera:

```text
max depth
```

no debe asumir relación inexistente silenciosamente si eso oculta un fallo técnico importante.

Podrá producir:

```text
FAILED
```

y fail closed.

---

# 244. Authorization Rule Registry

Reglas ABAC/ReBAC reutilizables podrán registrarse en:

```text
AuthorizationRuleRegistry
```

---

# 245. Rule ID

Ejemplos:

```text
invoice.approval_limit
resource.same_region
relationship.organization_member
```

---

# 246. Rule Object

```php
interface AuthorizationRuleInterface
{
    public function evaluate(
        AuthorizationRequest $request,
    ): DecisionResult;
}
```

---

# 247. Rule vs Voter

Una Rule puede ser utilizada dentro de Policy.

No necesariamente aparecer como evaluator en el plan.

---

# 248. Rule composition

Ejemplo:

```php
final class InvoicePolicy
{
    public function __construct(
        private SameTenantRule $sameTenant,
        private ApprovalLimitRule $approvalLimit,
    ) {}
}
```

---

# 249. Rule side effects

Igual que Policies:

```text
side-effect free
```

por diseño.

---

# 250. Rule cache

No debe cachear decisiones globales por sí sola.

---

# 251. Reusable conditions

Ejemplos:

```text
OwnsResourceRule
SameTenantRule
HasPermissionRule
RelationshipExistsRule
MeetsClearanceRule
```

---

# 252. Beware duplication

Si `SameTenantRule` ya se ejecuta globalmente como Critical Tenant Policy, no repetirla en cada Policy sin razón.

---

# 253. Layering

Una arquitectura saludable:

```text
Critical cross-cutting evaluator
        +
RBAC requirements
        +
Resource Policy
        +
Domain-specific ABAC/ReBAC rules
```

---

# 254. RBAC inside Resource Policy

También es válido cuando el permiso depende fuertemente del dominio.

---

# 255. Example Policy

```php
final class InvoicePolicy
{
    public function __construct(
        private PermissionCheckerInterface $permissions,
        private RelationshipCheckerInterface $relationships,
    ) {}

    public function update(
        User $user,
        Invoice $invoice,
        AuthorizationContext $context,
    ): DecisionResult {
        if (!$this->permissions->has(
            $user,
            'invoice.update',
            $context
        )) {
            return DecisionResult::deny(
                'rbac.permission_missing'
            );
        }

        if (!$this->relationships->has(
            ResourceReference::user($user->id),
            'member',
            ResourceReference::organization(
                $invoice->organization_id
            ),
            $context
        )) {
            return DecisionResult::deny(
                'rebac.relationship_missing'
            );
        }

        return DecisionResult::grant();
    }
}
```

---

# 256. Alternative Plan

Lo anterior también puede separarse:

```text
PermissionEvaluator
      ↓
RelationshipEvaluator
      ↓
InvoicePolicy
```

---

# 257. Choosing composition style

Regla recomendada:

```text
Cross-cutting requirement
→ evaluator

Resource-specific requirement
→ Policy/Rule
```

---

# 258. Permission API

Podrá existir:

```php
Authorization::permission(
    'invoice.approve'
)->for($user)->check();
```

pero no es necesario si ya existe:

```php
$user->hasPermission(...)
```

---

# 259. Facade

Podría existir:

```php
Permissions::has(
    $user,
    'invoice.approve'
);
```

para acceso directo al RBAC subsystem.

---

# 260. Direct Permission check vs Authorization

Debe distinguirse:

```php
$user->hasPermission('invoice.update')
```

de:

```php
$user->can('update', $invoice)
```

El primero verifica grant RBAC.

El segundo ejecuta autorización completa.

---

# 261. Developer guidance

Nunca asumir:

```text
hasPermission == can
```

---

# 262. Role API

Ejemplos:

```php
$user->hasRole('admin');

$user->hasAnyRole([
    'admin',
    'manager',
]);

$user->hasAllRoles([
    'finance',
    'approver',
]);
```

---

# 263. Permission API

```php
$user->hasPermission('invoice.approve');

$user->hasAnyPermission([...]);

$user->hasAllPermissions([...]);
```

---

# 264. Scoped APIs

```php
$user->hasRole(
    'admin',
    scope: TenantScope::of($tenant)
);
```

---

# 265. Default Scope

Si no se especifica:

```text
AuthorizationContext scope
```

podrá utilizarse.

---

# 266. Explicit scope beats ambient scope

Cuando se proporciona scope explícito deberá validarse contra Context para evitar cross-tenant misuse.

---

# 267. No hidden scope escalation

Un caller no deberá pasar:

```text
Tenant#9
```

arbitrariamente si el Context actual es Tenant#7, salvo una API privilegiada.

---

# 268. Scope Trust

Scopes derivados de Security/Tenant Context serán trusted.

Scopes arbitrarios podrán requerir validación.

---

# 269. CLI

En CLI puede no existir Tenant Context.

Un check global deberá funcionar.

Un check tenant-scoped requerirá scope explícito.

---

# 270. Queue

Roles/Permissions no deberían serializarse como snapshot permanente dentro de un Job para autorización futura.

---

# 271. Re-resolve grants

Por defecto, al ejecutar Job:

```text
PrincipalReference
    ↓
current roles/permissions
```

deberán resolverse de nuevo.

---

# 272. Historical grant semantics

Algunos workflows pueden querer conservar autoridad del momento de dispatch.

Esto deberá ser un modo explícito:

```text
DelegatedAuthorizationGrant
```

---

# 273. Delegated Grant

Conceptualmente:

```text
signed capability
limited scope
specific ability
specific subject
expiration
```

Esto se acerca a capability-based authorization.

---

# 274. Capability Tokens

Podrán integrarse en el futuro.

No forman parte de RBAC V1.

---

# 275. Security invariant

Nunca confiar en una lista de permissions enviada desde frontend.

---

# 276. SPA permissions

El frontend podrá recibir:

```text
capability projection
```

pero solo como UX.

---

# 277. Backend reauthorization

Toda operación real se autoriza nuevamente.

---

# 278. Permission manifest

Para UI:

```json
{
    "invoice.view": true,
    "invoice.approve": false
}
```

deberá derivarse del backend.

---

# 279. Do not expose entire permission universe

Preferir permisos necesarios para la vista actual.

---

# 280. RBAC Administration

VoltStack podrá proporcionar APIs para:

```text
create role
assign role
revoke role
attach permission
detach permission
grant direct permission
revoke direct permission
```

---

# 281. Administration separate from decision engine

Estas son operaciones de configuración de autorización.

No pertenecen al `DecisionManager`.

---

# 282. RoleManager

Conceptualmente:

```php
interface RoleManagerInterface
{
    public function assign(...): void;

    public function revoke(...): void;
}
```

---

# 283. PermissionManager

```php
interface PermissionManagerInterface
{
    public function grant(...): void;

    public function revoke(...): void;
}
```

---

# 284. Administrative authorization

Para asignar Roles, también deberá autorizarse:

```text
role.assign
permission.grant
```

mediante el mismo Authorization Engine.

---

# 285. Meta-authorization

Ejemplo:

```text
Admin wants to grant:
system.admin

to:
User#42
```

La operación debe verificar:

```text
can actor delegate that permission?
```

---

# 286. Privilege escalation prevention

Un usuario no deberá poder otorgar una permission superior a su capacidad de delegación.

---

# 287. Delegation Policy

Podrá existir:

```text
PermissionDelegationPolicy
```

---

# 288. Role Assignment Policy

Igualmente:

```text
RoleAssignmentPolicy
```

---

# 289. Role management is a Subject

Un Role puede ser un Subject legítimo.

Ejemplo:

```text
Ability:
assign

Subject:
Role:administrator
```

---

# 290. Permission management Subject

Igualmente:

```text
Ability:
grant

Subject:
Permission:system.deploy
```

---

# 291. Authorization recursion

Administrar autorización utiliza el propio Authorization Engine, pero deberá evitar recursión accidental.

---

# 292. Bootstrap roles

El sistema deberá permitir crear roles iniciales durante installation/migrations sin depender de una request autorizada.

---

# 293. Trusted administrative context

Bootstrap, seeding y provisioning utilizarán APIs privilegiadas claramente separadas.

---

# 294. No silent bypass API

Una API tipo:

```php
assignRoleWithoutAuthorization()
```

no deberá ser pública de alto nivel sin naming/namespace explícito.

---

# 295. Internal Grant Writer

Podrá estar en:

```text
Authorization\Administration\Internal
```

y usarse por install/migrations.

---

# 296. Audit mandatory for grants

En sistemas empresariales, cambios de privilegios deberían ser auditados.

---

# 297. Testing RBAC

Debe poder probarse:

```text
role grant
permission grant
scope matching
role inheritance
permission inheritance
expiration
```

sin HTTP.

---

# 298. Testing Permission Evaluator

```php
$result = $permissionEvaluator->evaluate(
    $request,
    PermissionRequirement::all([
        'invoice.approve',
    ]),
);
```

---

# 299. Testing ABAC

Test matrices sobre:

```text
Principal attributes
Subject attributes
Context
```

---

# 300. Testing ReBAC

Deberá probar:

```text
direct paths
indirect paths
cycles
depth limits
missing relation
scope isolation
```

---

# 301. Property-based testing

Muy útil para Role Graph y Relationship Graph.

---

# 302. Example property

```text
Revoking a required permission
must never preserve GRANT
when no alternate grant exists.
```

---

# 303. Scope isolation property

```text
Role in Tenant A
must not grant same role in Tenant B.
```

---

# 304. Relationship cycle property

```text
Graph cycles must terminate.
```

---

# 305. Cache consistency testing

Después de:

```text
revoke permission
```

un nuevo request deberá observar revocación.

---

# 306. Concurrent assignment tests

Dos asignaciones concurrentes no deben crear duplicates.

---

# 307. Database unique constraints

Ayudan a garantizarlo.

---

# 308. FrankenPHP tests

Request A:

```text
User#42
permissions A
```

Request B:

```text
User#99
permissions B
```

no deben compartir request cache.

---

# 309. Static Metadata

Sí puede compartirse:

```text
Role definitions
Permission definitions
compiled mappings
```

---

# 310. Dynamic Authorization State

Debe aislarse:

```text
effective roles
effective permissions
relationship paths
attribute snapshots
```

---

# 311. Performance Goals

Un permission check frecuente deberá evitar:

```text
N DB queries
```

por request.

---

# 312. Request preloading

Al autenticar al Principal podrá cargarse opcionalmente:

```text
effective grants
```

si benchmarks lo justifican.

---

# 313. Lazy loading

Alternativamente:

```text
resolve only requested permissions
```

---

# 314. Trade-off

```text
eager loading
=
more startup cost

lazy lookup
=
potential repeated queries
```

Request memoization permite combinar ambos.

---

# 315. Permission Bitsets

Para sistemas con permissions estáticas y gran volumen, podría explorarse:

```text
bitset-based permissions
```

---

# 316. Not default

Dificulta dynamic permissions y debugging.

No es necesario V1.

---

# 317. Integer Permission IDs

Podrán usarse internamente para índices optimizados.

API pública seguirá usando nombres.

---

# 318. Compiled Permission Map

Ejemplo:

```text
invoice.view     → 1
invoice.update   → 2
invoice.approve  → 3
```

---

# 319. Persistent worker optimization

Mapas inmutables de nombre → ID son excelentes candidatos para memoria process-wide.

---

# 320. RBAC query batching

Para listados de recursos, el mismo Permission global no debe consultarse por fila.

El Planner/memoization deberá resolverlo una vez.

---

# 321. ABAC batch

Atributos de muchos recursos podrán precargarse.

---

# 322. ReBAC batch

Graph query puede consultar múltiples Subjects en una sola operación.

---

# 323. BatchRelationshipChecker

Futuro contrato:

```php
interface BatchRelationshipCheckerInterface
{
    public function checkMany(...): iterable;
}
```

---

# 324. Observability

Profiler podrá mostrar:

```text
Permission checks: 12
DB queries: 1
cache hits: 11
```

---

# 325. RBAC trace

```text
Permission:
invoice.approve

Resolved via:
Role finance-manager

Scope:
Tenant#7

Result:
GRANT
```

---

# 326. ABAC trace

```text
Rule:
invoice.approval_limit

Principal limit:
REDACTED

Subject amount:
REDACTED

Result:
DENY
```

---

# 327. ReBAC trace

```text
Relationship:
principal → organization → invoice

Path:
matched

Result:
GRANT
```

---

# 328. Production Redaction

Valores sensibles pueden omitirse.

---

# 329. Metrics

Posibles métricas:

```text
authorization.rbac.role_checks
authorization.rbac.permission_checks
authorization.rbac.cache_hits

authorization.abac.rules
authorization.abac.denies

authorization.rebac.lookups
authorization.rebac.paths
authorization.rebac.depth
```

---

# 330. Slow Relationship Query

Profiler podrá advertir graph traversals costosos.

---

# 331. Tooling

Comandos futuros:

```text
volt authorization:roles
volt authorization:permissions
volt authorization:grants
volt authorization:relationships
volt authorization:rbac
```

---

# 332. `authorization:roles`

Ejemplo:

```text
Role              Permissions
--------------------------------------
finance-manager   12
administrator     48
auditor            6
```

---

# 333. Role details

```text
volt authorization:role finance-manager
```

podrá mostrar:

```text
Permissions
Scopes
Inheritance
Assignments
```

según privilegios/herramienta.

---

# 334. `authorization:permissions`

Mostrará catálogo y usage.

---

# 335. `authorization:grants`

Podrá inspeccionar:

```text
Principal User#42
Tenant#7

Roles:
finance-manager

Direct Permissions:
invoice.override-limit
```

---

# 336. Security of tooling

Comandos que expongan grants deberán respetar entorno y controles operativos.

---

# 337. Relationship tooling

```text
volt authorization:relationship-check \
user:42 member organization:7
```

útil en desarrollo.

---

# 338. Explain

```text
volt authorization:explain \
--principal=user:42 \
--ability=invoice.approve \
--subject=invoice:928
```

podría mostrar:

```text
Tenant → GRANT
Role → GRANT
Permission → GRANT
Relationship → GRANT
ABAC approval limit → DENY
```

---

# 339. CI lint

Puede detectar:

```text
unknown Roles
unknown Permissions
circular Role inheritance
invalid wildcard
unused Permissions
orphaned Roles
```

---

# 340. Unused permission

No necesariamente error.

Puede ser:

```text
future
package extension
dynamic use
```

pero tooling puede advertir.

---

# 341. Orphan Role

Role sin permissions ni special semantics puede ser warning.

---

# 342. Permission coverage

Tooling podrá mapear:

```text
Permission
→ Attributes
→ Gates
→ Policies
→ Routes
```

---

# 343. Database migrations

El RBAC persistence system deberá usar el Database subsystem de VoltStack.

No introducir un segundo DB layer.

---

# 344. Transaction management

Role grants/revocations deberán integrarse con:

```text
Database Transactions
```

---

# 345. Events

Cambios podrán emitir:

```text
RoleCreated
RoleAssigned
RoleRevoked
PermissionCreated
PermissionGranted
PermissionRevoked
RelationshipCreated
RelationshipRemoved
```

---

# 346. Event timing

Preferir:

```text
after commit
```

para invalidación/cache y side effects.

---

# 347. Domain Events vs Authorization Events

No deberán mezclarse indiscriminadamente.

---

# 348. GrantEvent metadata

Podrá incluir:

```text
actor
target principal
role/permission
scope
```

---

# 349. Security

Los eventos no deben contener secretos innecesarios.

---

# 350. Extension Providers

Packages podrán proporcionar:

```text
RoleProvider
PermissionProvider
AttributeProvider
RelationshipProvider
```

---

# 351. Provider Registration

A través de registry durante bootstrap.

---

# 352. Registry sealing

Después de bootstrap:

```text
provider registry
```

deberá ser inmutable.

---

# 353. Provider failure

Si un provider requerido falla:

```text
FAILED
```

y fail closed.

---

# 354. Optional provider failure

Solo podrá degradarse a abstención cuando el provider sea explícitamente opcional.

---

# 355. External IAM Integration

Ejemplo:

```text
AWS IAM
LDAP
Active Directory
custom enterprise IAM
```

podrán adaptarse mediante providers.

---

# 356. Local fallback

Si una organización utiliza:

```text
LDAP roles
+
local application permissions
```

ambos pueden combinarse.

---

# 357. Avoid provider ambiguity

Debe definirse si providers:

```text
append
override
fallback
```

grants.

---

# 358. Recommended default

```text
append grants
```

sin implicit deny.

---

# 359. Provider-specific deny

Si en el futuro se soporta explicit deny, la precedencia deberá documentarse separadamente.

---

# 360. RBAC Administration Namespace

Propuesta:

```text
Quantum\Authorization\RBAC
```

---

# 361. ABAC Namespace

```text
Quantum\Authorization\ABAC
```

---

# 362. ReBAC Namespace

```text
Quantum\Authorization\ReBAC
```

---

# 363. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    ├── RBAC/
    │   ├── Roles/
    │   │   ├── Role.php
    │   │   ├── RoleRegistry.php
    │   │   ├── RoleChecker.php
    │   │   ├── RoleRepositoryInterface.php
    │   │   ├── RoleAssignment.php
    │   │   └── RoleGraph.php
    │   │
    │   ├── Permissions/
    │   │   ├── Permission.php
    │   │   ├── PermissionRegistry.php
    │   │   ├── PermissionChecker.php
    │   │   ├── PermissionRepositoryInterface.php
    │   │   └── PermissionAssignment.php
    │   │
    │   ├── Scope/
    │   │   ├── AuthorizationScope.php
    │   │   ├── AuthorizationScopeType.php
    │   │   └── AuthorizationScopeResolver.php
    │   │
    │   ├── Evaluation/
    │   │   ├── RoleEvaluator.php
    │   │   ├── PermissionEvaluator.php
    │   │   ├── RoleRequirement.php
    │   │   └── PermissionRequirement.php
    │   │
    │   ├── Administration/
    │   │   ├── RoleManager.php
    │   │   ├── PermissionManager.php
    │   │   ├── RoleAssignmentPolicy.php
    │   │   └── PermissionDelegationPolicy.php
    │   │
    │   └── Cache/
    │       ├── RoleResolutionCache.php
    │       └── PermissionResolutionCache.php
    │
    ├── ABAC/
    │   ├── AuthorizationAttributeBag.php
    │   ├── AuthorizationAttributeProviderInterface.php
    │   ├── AuthorizationRuleInterface.php
    │   ├── AuthorizationRuleRegistry.php
    │   ├── AuthorizationRuleContext.php
    │   └── Evaluation/
    │       └── AttributeRuleEvaluator.php
    │
    ├── ReBAC/
    │   ├── ResourceReference.php
    │   ├── RelationshipTuple.php
    │   ├── RelationshipRepositoryInterface.php
    │   ├── RelationshipChecker.php
    │   ├── RelationshipGraph.php
    │   ├── RelationshipPath.php
    │   ├── RelationshipProviderInterface.php
    │   └── Evaluation/
    │       └── RelationshipEvaluator.php
    │
    └── Exceptions/
        ├── AuthorizationGrantException.php
        ├── UnknownRoleException.php
        ├── UnknownPermissionException.php
        ├── CircularRoleInheritanceException.php
        ├── InvalidAuthorizationScopeException.php
        ├── AuthorizationAttributeException.php
        ├── RelationshipResolutionException.php
        └── RelationshipGraphLimitException.php
```

---

# 364. RBAC Invariants

### Invariante 1

Roles agrupan capabilities; no conceden autorización final por sí solos.

### Invariante 2

Permissions son grants, no decisiones completas.

### Invariante 3

Un Role en un scope no se aplica automáticamente a otro scope.

### Invariante 4

Un Permission grant no bypassa Policies.

### Invariante 5

Super-admin no se implementa como bypass oculto dentro de RoleChecker.

### Invariante 6

Roles/Permissions dinámicos no se almacenan en registries estáticos.

---

# 365. ABAC Invariants

### Invariante 1

ABAC solo utiliza atributos de fuentes autorizadas.

### Invariante 2

Atributos de seguridad client-controlled no son trusted.

### Invariante 3

Missing attribute y failed condition son estados diferentes.

### Invariante 4

ABAC Rules no deben producir side effects.

### Invariante 5

Una DSL futura deberá ser sandboxed y estrictamente limitada.

---

# 366. ReBAC Invariants

### Invariante 1

Toda relación tiene Subject, Relation y Object.

### Invariante 2

Traversal tiene profundidad limitada.

### Invariante 3

Los ciclos se detectan.

### Invariante 4

Fallos de graph resolution no producen GRANT.

### Invariante 5

ReBAC no obliga a utilizar una tabla genérica; puede adaptarse al dominio existente.

---

# 367. Scope Invariants

### Invariante 1

Los scopes son explícitos.

### Invariante 2

El Tenant Context no puede sobrescribirse arbitrariamente.

### Invariante 3

Cross-scope checks requieren validación explícita.

### Invariante 4

Las scopes dinámicas son request-scoped.

---

# 368. Cache Invariants

### Invariante 1

Request memoization nunca sobrevive accidentalmente entre requests.

### Invariante 2

Caches cross-request deberán disponer de invalidación/versionado.

### Invariante 3

Revocaciones críticas no deben permanecer ocultas por caches prolongados.

### Invariante 4

El cache de grants no es Decision Cache.

---

# 369. Security Invariants

### Invariante 1

Frontend-provided Roles/Permissions nunca son autoridad.

### Invariante 2

External identity claims deben estar validados por Authentication.

### Invariante 3

Un provider fallido nunca implica grant.

### Invariante 4

Privilege delegation debe autorizarse.

### Invariante 5

Role/Permission changes deben ser auditables cuando la aplicación lo requiera.

---

# 370. Runtime Invariants

### Invariante 1

RoleChecker y PermissionChecker shared deben ser stateless.

### Invariante 2

Estado efectivo del Principal se resuelve por request o cache seguro.

### Invariante 3

No existe `$currentUser` o `$currentTenant` persistente en servicios compartidos.

### Invariante 4

Metadata estática sí puede permanecer en FrankenPHP workers.

---

# 371. Arquitectura final

```text
                          AuthorizationRequest
                                  │
                                  ↓
                         AuthorizationPlanner
                                  │
           ┌──────────────────────┼──────────────────────┐
           ↓                      ↓                      ↓
      RBAC Requirements      ABAC Requirements       ReBAC Rules
           │                      │                      │
      ┌────┴────┐                 ↓                      ↓
      ↓         ↓          Attribute Evaluator    Relationship Evaluator
 RoleEvaluator  PermissionEvaluator   │                      │
      │         │                     │                      │
      └────┬────┘                     │                      │
           └──────────────────────────┼──────────────────────┘
                                      ↓
                             DecisionResults
                                      ↓
                              DecisionManager
                                      ↓
                               Final Decision
```

---

# 372. Arquitectura híbrida completa

```text
Principal
   │
   ├── Roles
   ├── Permissions
   ├── Attributes
   └── Relationships
          │
          ↓
AuthorizationRequest
          │
          ↓
AuthorizationPlanner
          │
          ├── Global Security
          ├── Tenant Isolation
          ├── RoleEvaluator
          ├── PermissionEvaluator
          ├── RelationshipEvaluator
          ├── Resource Policy
          ├── ABAC Rules
          └── Compliance Policy
                  │
                  ↓
             DecisionManager
                  │
                  ↓
             GRANT / DENY
```

---

# 373. Ejemplo completo

Código:

```php
#[RequiresRole('finance-manager')]
#[RequiresPermission('invoice.approve')]
#[Authorize(
    'approve',
    subject: 'invoice'
)]
public function approve(
    Invoice $invoice
): Response {
    // ...
}
```

---

# 374. Principal

```text
User#42
```

Roles:

```text
finance-manager
```

Permissions:

```text
invoice.approve
```

Tenant:

```text
Tenant#7
```

---

# 375. Subject

```text
Invoice#928

tenant:
7

organization:
15

amount:
$85,000

status:
pending
```

---

# 376. Authorization Plan

```text
1. SuspendedPrincipalPolicy
   priority=10000

2. TenantIsolationPolicy
   priority=9000

3. RoleEvaluator
   requires finance-manager
   priority=7000

4. PermissionEvaluator
   requires invoice.approve
   priority=6500

5. OrganizationRelationshipEvaluator
   requires membership in Organization#15
   priority=6000

6. InvoicePolicy::approve
   priority=5000

7. ApprovalLimitRule
   priority=4500

Strategy:
DenyOverrides
```

---

# 377. Votes

```text
SuspendedPrincipalPolicy
→ ABSTAIN

TenantIsolationPolicy
→ GRANT

RoleEvaluator
→ GRANT

PermissionEvaluator
→ GRANT

RelationshipEvaluator
→ GRANT

InvoicePolicy
→ GRANT

ApprovalLimitRule
→ DENY
reason:
invoice.approval_limit_exceeded
```

---

# 378. Result

```text
DENY
```

Aunque el usuario tenga:

```text
Role
Permission
Relationship
```

ABAC puede rechazar la operación.

---

# 379. Segundo ejemplo

Si:

```text
Invoice amount:
$25,000
```

y todos los evaluadores conceden:

```text
GRANT
```

el DecisionManager devuelve:

```text
GRANT
```

---

# 380. Ejemplo cross-tenant

User:

```text
Tenant#7 admin
```

Invoice:

```text
Tenant#9
```

Aunque:

```text
role admin
permission invoice.update
```

TenantIsolationPolicy:

```text
DENY
```

Resultado:

```text
DENY
```

---

# 381. Filosofía del sistema

La filosofía será:

```text
Roles describe organizational position.

Permissions describe granted capabilities.

ABAC describes contextual conditions.

ReBAC describes relationships.

Policies describe resource-specific authorization.

The Authorization Engine combines all of them
into one deterministic decision.
```

---

# 382. Resultado esperado

El `Role, Permission, RBAC, ABAC and ReBAC Integration System` permitirá que VoltStack soporte desde aplicaciones sencillas:

```php
#[RequiresRole('admin')]
```

hasta sistemas empresariales complejos:

```text
Principal Role
        +
Permission
        +
Tenant Scope
        +
Resource Relationship
        +
Security Attributes
        +
Resource Policy
        +
Compliance Rule
        ↓
Decision Strategy
        ↓
GRANT / DENY
```

sin fragmentar el framework en varios sistemas de autorización incompatibles.

El principio definitivo será:

```text
RBAC answers what capabilities a Principal has.

ABAC answers under which conditions they apply.

ReBAC answers through which relationships access exists.

Policies apply those concepts to real resources.

The DecisionManager decides how all those answers
combine into the final authorization result.
```

Con esta arquitectura, VoltStack podrá escalar desde un sistema familiar de Roles y Permissions hasta modelos empresariales de seguridad contextual y relacional manteniendo una única infraestructura de autorización coherente.