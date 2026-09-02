# VoltStack Authorization System
## Hierarchical Scopes, Organizations, Teams and Workspaces System

**Documento:** `22_AUTHORIZATION_HIERARCHICAL_SCOPES_ORGANIZATIONS_TEAMS_AND_WORKSPACES_SYSTEM.md`  
**Sistema:** Authorization  
**Framework:** VoltStack  
**Módulo sugerido:** `Quantum/Authorization`  
**Estado:** Especificación arquitectónica  
**Versión objetivo:** 1.x+

---

# 1. Propósito

Este documento define la arquitectura para autorización basada en **jerarquías de scope, organizaciones, equipos, workspaces, unidades operativas y contenedores lógicos** dentro de VoltStack.

El sistema deberá resolver correctamente escenarios como:

```text
Una organización contiene múltiples equipos.

Un equipo pertenece a una organización.

Un workspace pertenece a un equipo o a una organización.

Un proyecto vive dentro de un workspace.

Un usuario puede ser administrador de una organización,
editor de un workspace
y viewer de otro.

Un rol puede existir únicamente dentro de un scope.

Un permiso puede heredarse desde un scope superior.

Una jerarquía puede permitir o bloquear herencia.

Un administrador de Organization A
no debe obtener acceso a Organization B.

Un Team Manager puede administrar recursos del Team
sin convertirse en administrador del Tenant completo.

Un Workspace Admin puede compartir recursos
sin poder gestionar billing del Tenant.
```

La meta es construir un modelo de autoridad jerárquico capaz de representar estructuras empresariales complejas sin caer en un modelo simplista como:

```text
is_admin = true
```

o:

```text
role = admin
```

sin contexto de scope.

El principio fundamental será:

```text
Authority is always evaluated
inside an explicit scope.

A role without scope is incomplete
whenever the application is scoped.
```

---

# 2. Motivación

En aplicaciones empresariales reales no existe normalmente una única frontera de autorización.

Pueden coexistir:

```text
Platform
Tenant
Organization
Business Unit
Division
Department
Team
Workspace
Project
Resource
```

Ejemplo:

```text
Platform
└── Tenant ACME
    ├── Organization Mexico
    │   ├── Team Finance
    │   │   └── Workspace Payments
    │   └── Team Operations
    │
    └── Organization USA
        └── Team Finance
```

Un usuario podría ser:

```text
Tenant Member

Organization Mexico Admin

Team Finance Manager

Workspace Payments Editor
```

y no poseer autoridad equivalente en:

```text
Organization USA
```

---

# 3. Problema de los roles globales

Un modelo:

```text
User
    role=admin
```

es insuficiente cuando existen scopes múltiples.

La pregunta correcta no es:

```text
¿Es admin?
```

sino:

```text
¿Es admin de qué?
```

---

# 4. Scope

VoltStack definirá un `AuthorizationScope` como una frontera lógica dentro de la cual se interpreta una autoridad.

Conceptualmente:

```php
interface AuthorizationScopeInterface
{
    public function authorizationScopeReference(): AuthorizationScopeReference;
}
```

---

# 5. AuthorizationScopeReference

```php
final readonly class AuthorizationScopeReference
{
    public function __construct(
        public string $type,
        public string|int $id,
        public ?TenantReference $tenant = null,
    ) {}
}
```

Ejemplos:

```text
tenant:7
organization:14
team:22
workspace:91
project:400
```

---

# 6. Tipos de Scope

VoltStack no deberá hardcodear exclusivamente organizaciones o equipos.

El sistema deberá permitir scopes extensibles.

Tipos comunes:

```text
Platform
Tenant
Organization
BusinessUnit
Division
Department
Team
Workspace
Project
ResourceCollection
```

---

# 7. Scope Registry

Deberá existir:

```text
Scope Type
    ↓
Scope Descriptor
```

Ejemplo:

```php
interface AuthorizationScopeRegistryInterface
{
    public function register(
        AuthorizationScopeDefinition $definition
    ): void;

    public function resolve(
        string $scopeType
    ): AuthorizationScopeDefinition;
}
```

---

# 8. AuthorizationScopeDefinition

Podrá contener:

```php
final readonly class AuthorizationScopeDefinition
{
    public function __construct(
        public string $type,
        public ?string $parentType,
        public bool $supportsRoles,
        public bool $supportsPermissions,
        public bool $supportsInheritance,
    ) {}
}
```

---

# 9. Hierarquía de Scopes

Una jerarquía puede ser:

```text
Tenant
    ↓
Organization
    ↓
Team
    ↓
Workspace
    ↓
Project
```

Pero VoltStack no deberá asumir que toda aplicación usa esta estructura exacta.

---

# 10. ScopeHierarchyResolver

Contrato:

```php
interface ScopeHierarchyResolverInterface
{
    public function parentOf(
        AuthorizationScopeReference $scope
    ): ?AuthorizationScopeReference;
}
```

---

# 11. Ancestors

Podrá existir:

```php
public function ancestorsOf(
    AuthorizationScopeReference $scope
): iterable;
```

---

# 12. Descendants

Para operaciones administrativas:

```php
public function descendantsOf(
    AuthorizationScopeReference $scope
): iterable;
```

pero deberá usarse cuidadosamente por impacto de rendimiento.

---

# 13. Scope Path

Ejemplo:

```text
Tenant#7
→ Organization#14
→ Team#22
→ Workspace#91
```

Podrá representarse como:

```php
final readonly class AuthorizationScopePath
{
    /** @param AuthorizationScopeReference[] $scopes */
    public function __construct(
        public array $scopes
    ) {}
}
```

---

# 14. Scope Path Invariant

Cada scope consecutivo deberá validar:

```text
child.parent == previous scope
```

según el resolver.

---

# 15. Tenant como frontera principal

En aplicaciones multi-tenant:

```text
Tenant
```

deberá actuar normalmente como la frontera superior de scopes empresariales.

Ejemplo:

```text
Tenant#7
└── Organization#14
```

No deberá permitirse:

```text
Organization#14
parent = Tenant#9
```

si pertenece a Tenant#7.

---

# 16. Cross-Tenant Hierarchy

Por defecto:

```text
forbidden
```

---

# 17. Platform Scope

Puede existir un scope especial:

```text
platform
```

por encima de los Tenants.

Ejemplo:

```text
Platform
└── Tenant#7
```

Pero esto no significa que un Platform Role pueda actuar dentro de todos los Tenants automáticamente.

---

# 18. Global vs Scoped Authority

VoltStack deberá distinguir:

```text
Global authority
```

de:

```text
Scoped authority
```

---

# 19. Global Authority

Ejemplo:

```text
platform.tenants.list
```

---

# 20. Scoped Authority

Ejemplo:

```text
organization.members.manage
scope=Organization#14
```

---

# 21. Scope-Aware Role Assignment

Un Role Assignment deberá incluir scope.

Conceptualmente:

```php
final readonly class ScopedRoleAssignment
{
    public function __construct(
        public PrincipalReference $principal,
        public string $role,
        public AuthorizationScopeReference $scope,
    ) {}
}
```

---

# 22. Example

```text
User#42
role=organization-admin
scope=Organization#14
```

No:

```text
User#42
role=organization-admin
scope=Organization#15
```

salvo asignación independiente.

---

# 23. Same Role, Different Scopes

Es válido:

```text
User#42
organization-admin
Organization#14

User#42
viewer
Organization#15
```

---

# 24. Role Identity

Una Role puede tener nombre canonical:

```text
organization.admin
```

sin incluir el ID del scope.

El assignment determina dónde aplica.

---

# 25. Permission Assignment

Igualmente:

```text
document.export
scope=Workspace#91
```

---

# 26. ScopedPermissionGrant

```php
final readonly class ScopedPermissionGrant
{
    public function __construct(
        public PrincipalReference $principal,
        public string $permission,
        public AuthorizationScopeReference $scope,
    ) {}
}
```

---

# 27. Scope Resolution en AuthorizationRequest

Toda autorización scoped deberá poder determinar:

```text
requested scope
```

---

# 28. Requested Scope

Puede derivarse desde:

```text
TenantContext
Subject
Route
Controller argument
Explicit API
WorkspaceContext
OrganizationContext
```

---

# 29. ScopeResolver

Contrato:

```php
interface AuthorizationScopeResolverInterface
{
    public function resolve(
        AuthorizationRequest $request
    ): AuthorizationScopeResolution;
}
```

---

# 30. AuthorizationScopeResolution

Podrá contener:

```php
final readonly class AuthorizationScopeResolution
{
    public function __construct(
        public ?AuthorizationScopeReference $scope,
        public ?AuthorizationScopePath $path,
        public AuthorizationScopeSource $source,
    ) {}
}
```

---

# 31. Scope Source

```php
enum AuthorizationScopeSource: string
{
    case Explicit = 'explicit';
    case Subject = 'subject';
    case TenantContext = 'tenant_context';
    case Route = 'route';
    case Controller = 'controller';
    case Inherited = 'inherited';
}
```

---

# 32. Explicit Scope

Ejemplo:

```php
Authorization::scope($workspace)
    ->authorize('document.create');
```

---

# 33. Subject-Derived Scope

Ejemplo:

```text
Document#100
belongs to Workspace#91
```

Authorization puede derivar:

```text
Workspace#91
```

---

# 34. ScopeMismatch

Si API especifica:

```text
Workspace#91
```

pero Subject pertenece a:

```text
Workspace#92
```

esto deberá tratarse como:

```text
DENY
```

o configuration/context failure según cómo se haya construido la request.

---

# 35. Scope Consistency

El Authorization Planner deberá validar:

```text
Explicit Scope
Subject Scope
Tenant Scope
```

cuando coexistigan.

---

# 36. Hierarchical Permission Inheritance

VoltStack deberá permitir opcionalmente que una autoridad asignada en un scope padre sea válida en scopes hijos.

Ejemplo:

```text
Organization Admin
Organization#14
```

podría administrar:

```text
Teams
Workspaces
Projects
```

dentro de Organization#14.

---

# 37. Pero no siempre

La herencia deberá ser:

```text
explicit
```

---

# 38. Scoped Role Definition

Un Role podrá declarar:

```text
inheritance mode
```

---

# 39. RoleScopeInheritanceMode

```php
enum RoleScopeInheritanceMode: string
{
    case Exact = 'exact';
    case Descendants = 'descendants';
    case DirectChildren = 'direct_children';
    case Custom = 'custom';
}
```

---

# 40. Exact

```text
role applies only to assigned scope
```

---

# 41. Descendants

```text
role applies to assigned scope
and all allowed descendants
```

---

# 42. DirectChildren

Solo:

```text
scope
+
immediate children
```

---

# 43. Custom

Evaluador específico.

---

# 44. Permission Inheritance

Los permissions también podrán declarar estrategia de scope.

---

# 45. Example

```text
workspace.document.view
```

puede ser heredable hacia:

```text
Projects/Documents
```

mientras:

```text
workspace.billing.manage
```

no tiene sentido fuera de Workspace.

---

# 46. Ability Scope Policy

La herencia no deberá depender únicamente del Role.

También puede depender de la Ability.

---

# 47. Why

Un Organization Admin puede heredar:

```text
team.members.manage
```

hacia Teams.

Pero no necesariamente:

```text
workspace.secret.rotate
```

---

# 48. AbilityScopeDescriptor

Podrá declarar:

```php
final readonly class AbilityScopeDescriptor
{
    public function __construct(
        public string $ability,
        public array $validScopeTypes,
        public AuthorizationScopePropagation $propagation,
    ) {}
}
```

---

# 49. AuthorizationScopePropagation

```php
enum AuthorizationScopePropagation: string
{
    case None = 'none';
    case Exact = 'exact';
    case Descendants = 'descendants';
    case ExplicitPath = 'explicit_path';
}
```

---

# 50. Effective Scope Rule

La autoridad efectiva dependerá de:

```text
Role/Permission Assignment
AND
Ability propagation rule
AND
Scope hierarchy
AND
Boundary rules
```

---

# 51. Scope Boundary

Una jerarquía deberá poder bloquear herencia.

Ejemplo:

```text
Organization
    ↓
Team
    ↓
Restricted Workspace
```

---

# 52. AuthorizationScopeBoundary

Podrá representar:

```text
inheritance stops here
```

---

# 53. Example

Organization Admin puede administrar todos los Teams.

Pero un Workspace marcado:

```text
isolated=true
```

puede requerir un assignment explícito.

---

# 54. ScopeBoundaryPolicy

Contrato:

```php
interface ScopeBoundaryPolicyInterface
{
    public function allowsPropagationThrough(
        AuthorizationScopeReference $scope,
        AuthorizationContext $context
    ): bool;
}
```

---

# 55. Scope Types con aislamiento fuerte

Ejemplos:

```text
Legal Workspace
Security Workspace
Compliance Workspace
Confidential Project
```

---

# 56. No Implicit Super-Admin Propagation

Incluso un Role poderoso en parent scope deberá respetar:

```text
nonBypassable boundaries
```

---

# 57. Organization

Una Organization será un scope administrativo de alto nivel.

---

# 58. Organization Membership

Relación:

```text
User
member_of
Organization
```

---

# 59. OrganizationMember

Puede contener:

```text
membership status
role assignments
joined_at
expires_at
```

---

# 60. Membership no equivale a Role

```text
member_of Organization
```

solo significa pertenencia.

No:

```text
organization.admin
```

---

# 61. Organization Membership Status

```php
enum MembershipStatus: string
{
    case Active = 'active';
    case Invited = 'invited';
    case Suspended = 'suspended';
    case Revoked = 'revoked';
    case Expired = 'expired';
}
```

---

# 62. Active Membership Required

Roles dentro de Organization normalmente requerirán:

```text
membership=active
```

---

# 63. Stale Role Assignment

Si Membership se revoca:

```text
scoped Roles
```

no deberán seguir produciendo autoridad.

---

# 64. Organization Roles

Ejemplos:

```text
organization.owner
organization.admin
organization.member
organization.auditor
organization.billing
```

---

# 65. Organization Owner

No deberá confundirse con:

```text
Tenant Owner
```

---

# 66. Organization Owner Authority

Puede incluir:

```text
organization.settings.manage
organization.members.manage
organization.teams.manage
```

pero no:

```text
tenant.billing.manage
```

salvo reglas específicas.

---

# 67. Multiple Organizations per Tenant

Debe ser soportado de forma natural.

---

# 68. User in Multiple Organizations

También.

Ejemplo:

```text
User#42

Organization#14
admin

Organization#15
viewer
```

---

# 69. Organization Context

No deberá inferirse globalmente de:

```text
current user
```

porque un usuario puede pertenecer a varias.

---

# 70. Explicit Organization Scope

Debe derivarse de:

```text
resource
route
workspace
explicit context
```

---

# 71. Team

Un Team será un scope colaborativo más específico.

---

# 72. Team Membership

```text
User
member_of
Team
```

---

# 73. Team Parent

Normalmente:

```text
Team
→ Organization
```

pero podrá configurarse otro modelo.

---

# 74. Team Roles

Ejemplos:

```text
team.owner
team.manager
team.member
team.viewer
```

---

# 75. Team Manager

Puede poseer:

```text
team.members.manage
team.workspaces.create
team.resources.manage
```

---

# 76. Scope

Solo:

```text
Team#22
```

y sus descendants permitidos.

---

# 77. Team Membership Inheritance

Ser miembro de Organization no implica ser miembro de todos los Teams.

---

# 78. Team Access via Organization Role

Sin embargo, un Organization Admin puede tener autoridad administrativa sobre Teams mediante Role inheritance.

Eso no significa que aparezca como Team Member.

---

# 79. Important Difference

```text
Membership
```

y:

```text
Administrative Authority
```

son distintos.

---

# 80. Example

Organization Admin puede:

```text
rename Team
add members
```

sin ser:

```text
Team Member
```

para operaciones de negocio internas del Team.

---

# 81. Membership-Requiring Abilities

Algunas abilities podrán exigir membership explícita.

Ejemplo:

```text
team.chat.post
```

---

# 82. Admin Authority Not Enough

Organization Admin:

```text
team.manage
```

pero no necesariamente:

```text
team.chat.post
```

---

# 83. MembershipRequirement

Podrá existir:

```text
RequiresMembership(Team)
```

separado de Role checks.

---

# 84. Workspace

Un Workspace será un scope orientado a trabajo y recursos.

---

# 85. Workspace Parent

Puede pertenecer a:

```text
Organization
Team
Tenant
```

según aplicación.

---

# 86. Workspace Roles

Ejemplos:

```text
workspace.owner
workspace.admin
workspace.editor
workspace.viewer
workspace.guest
```

---

# 87. Workspace Resource Scope

Recursos como:

```text
Documents
Projects
Dashboards
Datasets
Models
Reports
```

pueden residir dentro de Workspace.

---

# 88. Workspace Role Inheritance

Ejemplo:

```text
workspace.editor
```

puede otorgar acceso base a Documents del Workspace.

---

# 89. Resource Policy Still Applies

Como en documento 21:

```text
Workspace Editor
```

no significa editar un Document:

```text
locked
legal_hold
confidential
```

si Policy lo deniega.

---

# 90. Workspace Guest

Puede tener authority muy limitada.

---

# 91. External Guests

Un guest puede pertenecer al Workspace sin ser miembro completo de Organization.

---

# 92. Membership Graph

Ejemplo:

```text
User#90
guest_of Workspace#91
```

sin:

```text
member_of Organization#14
```

si el modelo lo permite.

---

# 93. Tenant Boundary Still Applies

Guest debe pertenecer a una identity/autorización válida dentro del Tenant boundary correspondiente.

---

# 94. Workspace Isolation

Un Workspace podrá tener:

```text
isolated_authorization=true
```

---

# 95. Meaning

Authority de Organization/Team no se propaga automáticamente.

---

# 96. Example

```text
Organization Admin
```

puede administrar metadata del Workspace:

```text
workspace.lifecycle.manage
```

pero no necesariamente leer su contenido.

---

# 97. Administrative vs Content Authority

VoltStack deberá poder distinguir:

```text
manage container
```

de:

```text
access container content
```

---

# 98. Example

Ability sets:

```text
workspace.settings.update
workspace.members.manage
```

vs:

```text
document.view
dataset.query
```

---

# 99. Scope Domain

Podrá declararse:

```text
administrative
content
billing
security
```

---

# 100. AuthorizationScopeDomain

```php
enum AuthorizationScopeDomain: string
{
    case Administration = 'administration';
    case Content = 'content';
    case Membership = 'membership';
    case Billing = 'billing';
    case Security = 'security';
}
```

---

# 101. Why

Una Role puede propagarse solo dentro de ciertos domains.

---

# 102. Example

Organization Admin:

```text
Administration → descendants
Membership → descendants
Content → no propagation
```

---

# 103. Powerful Separation

Esto evita que:

```text
organization.admin
```

se convierta accidentalmente en:

```text
read everything
```

---

# 104. Scope Domain Policy

RoleDefinition podrá declarar:

```text
propagation by domain
```

---

# 105. Example

```text
organization.admin

Administration:
descendants

Membership:
descendants

Content:
exact/no grant

Billing:
organization only
```

---

# 106. ScopedRoleDefinition

Conceptualmente:

```php
final readonly class ScopedRoleDefinition
{
    public function __construct(
        public string $name,
        public array $permissions,
        public array $propagationByDomain,
    ) {}
}
```

---

# 107. Organization Owner

Puede tener más dominios, pero aún bajo límites explícitos.

---

# 108. Team Manager

Ejemplo:

```text
Administration:
Team + Workspace descendants

Content:
maybe descendants

Billing:
none
```

---

# 109. Workspace Admin

```text
Administration:
Workspace

Content:
Workspace descendants
```

---

# 110. Scope Roles vs Resource Roles

Deben distinguirse.

---

# 111. Scope Role

Ejemplo:

```text
workspace.editor
```

aplica a un conjunto de recursos.

---

# 112. Resource Relationship

Ejemplo:

```text
editor_of Document#100
```

aplica a un recurso específico.

---

# 113. Priority

No deberá asumirse que uno reemplaza al otro.

---

# 114. Effective Authority

Puede surgir de:

```text
Scoped Role
OR
Resource Share
OR
Ownership
OR
Relationship
```

sujeto a constraints.

---

# 115. Hierarchical Role Resolution

El sistema deberá encontrar assignments relevantes para un requested scope.

---

# 116. Example

Requested:

```text
Workspace#91
```

Scope path:

```text
Tenant#7
→ Organization#14
→ Team#22
→ Workspace#91
```

---

# 117. Candidate Assignments

Resolver puede buscar:

```text
Workspace#91
Team#22
Organization#14
Tenant#7
Platform
```

según ability propagation.

---

# 118. But Not Always All

El `AbilityScopeDescriptor` permitirá reducir búsquedas.

---

# 119. Example

Ability:

```text
workspace.document.view
```

puede revisar:

```text
Workspace
Team
Organization
```

pero no:

```text
Platform
```

salvo explicit rule.

---

# 120. HierarchicalGrantResolver

Contrato:

```php
interface HierarchicalGrantResolverInterface
{
    public function resolve(
        PrincipalInterface $principal,
        string $ability,
        AuthorizationScopeReference $requestedScope,
        AuthorizationContext $context
    ): HierarchicalGrantResolution;
}
```

---

# 121. HierarchicalGrantResolution

Podrá contener:

```text
matching assignments
source scope
propagation rule
effective scope
grant provenance
```

---

# 122. Example

```text
GRANT document.view

source:
organization.admin

assigned at:
Organization#14

applies to:
Workspace#91

path:
Organization#14
→ Team#22
→ Workspace#91
```

---

# 123. Explainability

Esto deberá integrarse con documento 15.

---

# 124. Public Explanation

Puede decir:

```text
You have access through your workspace role.
```

---

# 125. Operator Explanation

Puede detallar:

```text
Role organization.admin at Organization#14
propagated through Team#22
to Workspace#91.
```

---

# 126. Role Shadowing

Podría existir un assignment más específico que restrinja authority del parent.

---

# 127. Cuidado

Esto introduce negative authorization y complejidad significativa.

---

# 128. V1 Recommendation

No utilizar:

```text
child role automatically revokes parent grants
```

como default.

---

# 129. Instead

Usar:

```text
scope boundary
explicit security DENY
resource Policy
```

para restricciones.

---

# 130. Example

Organization Admin hereda acceso administrativo.

Confidential Workspace:

```text
scope boundary
```

corta esa herencia.

---

# 131. Explicit Deny Assignments

Pueden añadirse en el futuro, pero deberán integrarse con `DenyOverrides`.

---

# 132. Organization Hierarchy

Algunas empresas necesitan:

```text
Parent Organization
    ↓
Subsidiary
    ↓
Division
```

---

# 133. Recursive Organization Hierarchy

Deberá ser bounded y validada.

---

# 134. Circular Organization

Prohibido:

```text
Org A parent Org B
Org B parent Org A
```

---

# 135. Scope Cycle Detection

El hierarchy resolver deberá detectar cycles.

---

# 136. Max Scope Depth

Podrá existir:

```text
authorization.scopes.max_depth
```

---

# 137. V1 Default

Un valor razonable como:

```text
16
```

podrá configurarse, pero el número exacto no debe formar parte rígida del contrato.

---

# 138. Organization Descendant Authority

Un Role en parent Organization puede o no propagarse a subsidiaries.

---

# 139. Explicit Propagation

Ejemplo:

```text
group.security_auditor
```

podría propagarse.

---

# 140. Local Role

```text
organization.billing_manager
```

puede quedarse exacto.

---

# 141. Teams anidados

Algunas aplicaciones pueden permitir:

```text
Team
└── Subteam
```

---

# 142. V1 Recommendation

Soportar jerarquía genérica de scope para que sea posible, pero no exigirla.

---

# 143. Membership Inheritance in Teams

Ser miembro de Team padre no implica automáticamente miembro de Subteam.

---

# 144. Separate Membership Semantics

La hierarchy de scopes no determina por sí sola membership.

---

# 145. Workspace Nesting

Puede existir:

```text
Workspace
└── Subworkspace
```

pero nuevamente la herencia debe ser configurable.

---

# 146. Scope Relationship vs Resource Hierarchy

Importante distinguir:

```text
Authorization Scope Hierarchy
```

de:

```text
Resource Containment Hierarchy
```

---

# 147. Example

```text
Workspace
```

es scope.

```text
Folder
```

puede ser resource container.

---

# 148. Some Objects Can Be Both

Un Project podría funcionar como:

```text
resource
+
authorization scope
```

---

# 149. ScopedSubject

Podrá declararse que un Subject crea un scope.

---

# 150. Example

```php
#[DefinesAuthorizationScope('project')]
final class Project
{
}
```

conceptualmente.

---

# 151. Child Resources

Documents bajo Project pueden heredar authority desde Project scope.

---

# 152. Scope Creation

Crear un nuevo Workspace/Project podrá requerir:

```text
scope.create
```

en el parent scope.

---

# 153. Example

```text
team.workspace.create
scope=Team#22
```

---

# 154. Scope Deletion

Ability separada:

```text
workspace.delete
```

---

# 155. Deleting Scope

Debe considerar:

```text
memberships
role assignments
child scopes
resources
shares
delegations
```

---

# 156. Lifecycle Policy

Authorization no ejecuta cleanup, pero deberá coordinarse con domain services.

---

# 157. Scope Move

Mover Workspace de un Team a otro cambia jerarquía de authority.

---

# 158. Critical Operation

Debe existir:

```text
workspace.move
```

---

# 159. Move Validation

Debe verificar:

```text
may manage source
may manage destination
tenant same
resulting hierarchy valid
no cycles
```

---

# 160. Authorization Effects

Después del move:

```text
inherited grants may change
```

---

# 161. Cache Invalidation

Debe incrementar version relevante.

---

# 162. ScopeVersion

Podrá existir:

```text
scope_hierarchy_version
```

---

# 163. MembershipVersion

Separado:

```text
scope_membership_version
```

---

# 164. RoleAssignmentVersion

También:

```text
role_assignment_version
```

cuando granularidad lo justifique.

---

# 165. Coarse Version Strategy

V1 puede utilizar:

```text
scope_authorization_version
```

como versión combinada.

---

# 166. Cache Key

Debe considerar:

```text
principal version
tenant version
scope authorization version
ability
requested scope
```

según consistency policy.

---

# 167. Hierarchical Cache

La resolución:

```text
User#42
ability=document.view
scope=Workspace#91
```

puede memoizar:

```text
source Organization#14
role organization.admin
```

durante request.

---

# 168. Membership Caching

Memberships son buenos candidatos a memoización.

---

# 169. Batch Resolution

Listados de múltiples Workspaces deberán evitar:

```text
one hierarchy lookup per workspace
```

---

# 170. Example

Resolver authority para:

```text
Workspace IDs [1..100]
```

podrá usar:

```text
preloaded scope paths
role assignment map
```

---

# 171. ScopePathCache

Podrá mantener:

```text
scope ID
→ parent chain
```

con versioning.

---

# 172. Persistent Worker

Los paths estructurales relativamente estables podrán cachearse en worker si se invalidan correctamente.

---

# 173. But Dynamic State

Membership y role assignments no deberán quedar stale indefinidamente.

---

# 174. Hierarchical Permission Resolution Algorithm

Conceptualmente:

```text
1 resolve requested scope
2 resolve scope path
3 validate tenant boundary
4 load principal assignments
5 filter by ability
6 filter by scope propagation
7 apply scope boundaries
8 evaluate membership requirements
9 normalize grants
10 continue authorization pipeline
```

---

# 175. Candidate Grant Resolution

Ejemplo:

```text
Requested scope:
Workspace#91
```

Assignments:

```text
organization.admin @ Organization#14
team.viewer @ Team#22
workspace.editor @ Workspace#91
```

---

# 176. Ability

```text
document.update
```

---

# 177. Evaluation

Puede determinar:

```text
organization.admin
does not propagate Content domain

team.viewer
does not include update

workspace.editor
includes document.update
```

Resultado:

```text
GRANT
```

---

# 178. Different Ability

```text
team.members.manage
```

Requested:

```text
Team#22
```

Organization Admin sí puede heredarse para Membership domain.

---

# 179. Domain-Aware Propagation

Esto evita roles excesivamente poderosos.

---

# 180. Scope Constraints

Una Ability podrá declarar:

```text
scopeType=workspace
```

---

# 181. Wrong Scope

Si se intenta:

```text
document.update
scope=Organization#14
```

y ability requiere Workspace/Project/Resource scope, deberá resolverse al child apropiado o marcarse inválida.

---

# 182. AbilityScopeValidation

El Planner deberá validar scope types.

---

# 183. Scope Coercion

No deberá existir coerción arbitraria.

---

# 184. Parent Derivation

Solo cuando metadata declare:

```text
resolve resource's workspace scope
```

---

# 185. Scope Binding Metadata

Ejemplo:

```php
#[Authorize(
    'document.update',
    subject: 'document',
    scope: 'document.workspace'
)]
```

conceptualmente.

---

# 186. Compiled Binding

En compiled mode:

```text
argument 0 → Subject
subject.workspace_id → Scope
```

deberá estar precomputado mediante adapter seguro.

---

# 187. ORM Independence

Authorization Core no deberá depender de acceder a propiedades Eloquent/Doctrine.

---

# 188. Scope Adapter

Un adapter de dominio/ORM podrá resolver:

```text
Document
→ Workspace reference
```

---

# 189. ScopeProvider

Contrato:

```php
interface SubjectScopeProviderInterface
{
    public function scopeFor(
        mixed $subject,
        AuthorizationContext $context
    ): ?AuthorizationScopeReference;
}
```

---

# 190. Scope Provider Registry

```text
Document
→ DocumentWorkspaceScopeProvider

Project
→ ProjectScopeProvider
```

---

# 191. Organization Context

Podrá existir convenience context:

```text
OrganizationContext
```

pero Authorization deberá seguir usando el modelo genérico de Scope.

---

# 192. TeamContext

Igualmente.

---

# 193. WorkspaceContext

Igualmente.

---

# 194. Why Generic Scope

Evita construir tres engines:

```text
OrganizationAuthorization
TeamAuthorization
WorkspaceAuthorization
```

---

# 195. Unified Scope Engine

Todo converge en:

```text
AuthorizationScopeReference
```

---

# 196. Scoped Gates

Gates podrán declararse para scope.

Ejemplo:

```php
Gate::define(
    'workspace.manage',
    WorkspaceManagementGate::class
);
```

con ScopeContext.

---

# 197. Scoped Policy

Policy puede recibir:

```text
requested scope
scope path
```

mediante AuthorizationContext.

---

# 198. Direct Role Check

Convenience API:

```php
$user->hasRole(
    'workspace.editor',
    scope: $workspace
);
```

---

# 199. Important

Debe usar Authorization/RBAC provider, no un helper desconectado.

---

# 200. Permission Check

```php
$user->hasPermission(
    'document.update',
    scope: $workspace
);
```

---

# 201. Inherited Check

Opcionalmente:

```php
$user->hasEffectivePermission(
    'document.update',
    scope: $workspace
);
```

---

# 202. Distinction

`hasPermission()` puede significar exact assignment.

`hasEffectivePermission()` incluye inherited authority.

La API deberá documentarlo claramente.

---

# 203. Recommended Authorization API

Para decisiones de negocio usar:

```php
Authorization::can(...)
```

No construir lógica compleja mezclando manualmente estos helpers.

---

# 204. Membership API

```php
$memberships->isMemberOf(
    $user,
    $organization
);
```

es data lookup.

No decisión global.

---

# 205. Scoped Roles as ReBAC

Conceptualmente:

```text
User
has_role
Role
within
Scope
```

puede verse como relación.

---

# 206. But

Mantener un modelo RBAC explícito facilita:

```text
administration
queries
UI
role catalogs
permission expansion
```

---

# 207. RBAC + ReBAC Integration

VoltStack deberá permitir ambos:

```text
Scoped Role
+
Relationship Graph
```

---

# 208. Example

```text
User#42
role=team.manager
scope=Team#22
```

y:

```text
Team#22
owns
Workspace#91
```

pueden producir authority sobre Workspace.

---

# 209. Avoid Double Modeling

No guardar necesariamente lo mismo simultáneamente como:

```text
role assignment
+
duplicate graph edge
```

si un adapter puede proyectarlo.

---

# 210. Projection

RBAC assignments podrán exponerse al ReBAC engine como virtual relationships.

---

# 211. Example Virtual Edge

```text
User#42
manager_of
Team#22
```

derivado de:

```text
role=team.manager
```

---

# 212. Source of Truth

Debe quedar claro si el Role Assignment o Graph es canonical.

---

# 213. Recommended

Para roles:

```text
RBAC store = source of truth
```

Graph puede usar projection.

---

# 214. Organization Membership Source

Puede venir de:

```text
database
directory
SCIM
LDAP
external IAM
```

---

# 215. MembershipProvider

Contrato:

```php
interface ScopeMembershipProviderInterface
{
    public function membership(
        PrincipalInterface $principal,
        AuthorizationScopeReference $scope,
        AuthorizationContext $context
    ): ScopeMembershipResult;
}
```

---

# 216. ScopeMembershipResult

Podrá contener:

```text
active
role hints
membership attributes
expiration
source
```

---

# 217. Multiple Membership Providers

Una Organization puede sincronizarse desde directory.

Workspace guests pueden almacenarse localmente.

---

# 218. Composite Membership Resolver

Deberá normalizar ambos.

---

# 219. Conflicts

Si provider A dice:

```text
active
```

y provider B:

```text
revoked
```

la estrategia deberá estar explícita.

---

# 220. Security Default

Una revocation autoritativa deberá prevalecer.

---

# 221. Membership Attributes

Ejemplos:

```text
department
employment_type
location
manager
external_guest
```

pueden alimentar ABAC.

---

# 222. Example

Workspace allows:

```text
workspace.editor
```

solo si:

```text
membership.external_guest=false
```

---

# 223. Domain Rule

Eso puede expresarse vía:

```text
RBAC + ABAC
```

---

# 224. Organization Owner Bootstrap

Crear Organization requiere asignar primer Owner de forma segura.

---

# 225. Bootstrap Transaction

Ideal:

```text
create organization
create owner membership
assign organization.owner
commit
```

---

# 226. No Ownerless Organization

Si el dominio lo exige, deberá ser invariante.

---

# 227. Last Owner Protection

Eliminar al último Organization Owner puede ser:

```text
DENY
```

---

# 228. Policy

```text
organization.owner.remove
```

deberá comprobar que quede otro owner.

---

# 229. Team Last Manager

Similar si el dominio requiere mínimo un manager.

---

# 230. Workspace Last Admin

Igualmente configurable.

---

# 231. Membership Mutation

Abilities:

```text
organization.member.invite
organization.member.remove
organization.member.suspend

team.member.add
team.member.remove

workspace.member.add
workspace.member.remove
```

---

# 232. Role Assignment Mutation

Separadas:

```text
organization.role.assign
team.role.assign
workspace.role.assign
```

---

# 233. Why Separate

Agregar miembro no equivale a:

```text
make admin
```

---

# 234. Role Grant Ceiling

Quien asigna un Role no debería poder otorgar authority superior a la que está autorizado a administrar.

---

# 235. No Role Escalation

Team Manager no debe poder asignar:

```text
organization.owner
```

---

# 236. Role Assignment Policy

Validará:

```text
assigner authority
target role
target scope
scope relationship
role delegatability
```

---

# 237. Role Delegatability

RoleDefinition podrá declarar:

```text
assignableBy
```

o un policy ID.

---

# 238. Example

```text
workspace.viewer
assignable by workspace.admin

workspace.admin
assignable only by organization.admin
```

---

# 239. Scoped Role Escalation

También evitar:

```text
assign same Role at broader scope
```

---

# 240. Example

User es:

```text
workspace.admin @ Workspace#91
```

No puede asignarse:

```text
organization.admin @ Organization#14
```

---

# 241. Scope Containment

Role assignment debe validar que target scope esté dentro de authority permitida del assigner.

---

# 242. Membership Invite Scope

Invitar a Organization no debe ser posible para alguien que solo administra Team.

---

# 243. Team Invite

Sí puede ser posible si Team permite direct membership externa.

---

# 244. Parent Membership Requirement

Algunas aplicaciones requerirán:

```text
Team member
must also be Organization member
```

---

# 245. Other Applications

Pueden permitir:

```text
external team guest
```

---

# 246. MembershipPolicy

Debe ser configurable por scope type.

---

# 247. Workspace Guest Model

Ejemplo típico:

```text
external user
→ guest of Workspace
```

sin Organization membership completa.

---

# 248. Guest Restrictions

Puede tener:

```text
no resharing
no export
read-only
expiration
```

---

# 249. Guest Policy

Se puede implementar con:

```text
membership attribute
+
ABAC
```

---

# 250. Scope Expiration

Un membership o Role assignment puede expirar.

---

# 251. Example

Consultant:

```text
workspace.editor
until 2026-09-30
```

---

# 252. Expired Assignment

No concede authority.

---

# 253. Temporal Scope Authority

ClockInterface deberá usarse para tests.

---

# 254. Scheduled Activation

También podrá existir:

```text
active_from
```

---

# 255. Suspended Scope

Organization/Workspace puede estar:

```text
suspended
```

---

# 256. Operational Status

Scope status puede ser:

```php
enum AuthorizationScopeStatus: string
{
    case Active = 'active';
    case ReadOnly = 'read_only';
    case Suspended = 'suspended';
    case Archived = 'archived';
    case Deleted = 'deleted';
}
```

---

# 257. Scope Operational Evaluator

Deberá ejecutarse temprano.

---

# 258. ReadOnly Scope

Puede permitir:

```text
view
```

pero denegar:

```text
update
delete
invite
```

---

# 259. Suspended Scope

Normalmente:

```text
DENY most operations
```

---

# 260. Archived Workspace

Puede permitir lectura a ciertos roles.

---

# 261. ScopeStatusPolicy

La mapping Ability → allowed status debe ser configurable.

---

# 262. Platform Operations

Algunas operaciones pueden ejecutar mantenimiento sobre scope suspendido.

---

# 263. Dedicated Ability

Ejemplo:

```text
platform.workspace.recover
```

No usar bypass general.

---

# 264. Nested Tenant Context

No deberá existir:

```text
Tenant#7
→ Organization belonging Tenant#9
```

en ScopePath válido.

---

# 265. Hierarchy Validator

Deberá verificar esto en:

```text
scope creation
scope move
compile/static metadata
runtime resolution
```

según aplicabilidad.

---

# 266. Scope Graph

Aunque conceptualmente jerárquico, debería ser:

```text
DAG/tree
```

según configuración.

---

# 267. Single Parent

V1 Recommendation:

```text
each authorization scope has at most one structural parent
```

---

# 268. Why

Simplifica:

```text
inheritance
cache invalidation
explainability
cycle detection
```

---

# 269. Multi-Parent Scope

Puede soportarse después mediante ReBAC si se requiere.

---

# 270. Workspace Shared Between Teams

En vez de hacer multi-parent structural hierarchy, puede modelarse:

```text
one owner Team
+
relationship/shared access from another Team
```

---

# 271. Recommended

Mantener hierarchy estructural simple y usar ReBAC para relaciones adicionales.

---

# 272. Hierarchical Authorization vs ReBAC

Hierarchy responde:

```text
where does this scope live?
```

ReBAC responde:

```text
how is this Principal related to this scope/resource?
```

---

# 273. Role Scope vs Ownership

Organization puede:

```text
own Workspace
```

pero Role resolution sigue siendo independiente.

---

# 274. Workspace Ownership

Ownership puede ser:

```text
Organization#14
```

mientras hierarchy:

```text
Team#22 → Workspace#91
```

si el dominio lo permite.

---

# 275. Recommendation

Mantener hierarchy y ownership explícitos, aunque normalmente coincidan.

---

# 276. Scope Alias

UI puede mostrar:

```text
Finance Team
```

pero cache/security usa ID canonical.

---

# 277. Rename Scope

No cambia authority IDs.

---

# 278. Slugs

No deberán ser única identity de autorización si pueden cambiar.

---

# 279. Scope Canonical ID

Debe ser estable.

---

# 280. Scope Lookup

Route:

```text
/org/acme/workspaces/finance
```

debe resolver a IDs canonical antes de authorization.

---

# 281. Concealment

Un usuario sin access puede recibir:

```text
404
```

aunque Organization exista.

---

# 282. Scope Discovery

Listing Organizations/Teams/Workspaces también es authorization.

---

# 283. Ability

Ejemplo:

```text
workspace.list
```

---

# 284. Query-Level Scope Filtering

El sistema deberá poder producir:

```text
visible scopes for Principal
```

sin cargar todos y hacer N checks.

---

# 285. AuthorizedScopeQuery

Contrato conceptual:

```php
interface AuthorizedScopeQueryInterface
{
    public function apply(
        mixed $query,
        PrincipalInterface $principal,
        string $ability,
        AuthorizationContext $context
    ): mixed;
}
```

---

# 286. Example

List Workspaces visible por:

```text
direct membership
team membership
organization role
explicit share
```

---

# 287. Final Resource Checks

Una vez seleccionado Workspace, operaciones sensibles siguen usando Authorization Engine normal.

---

# 288. Organization Listing

Tenant member no necesariamente ve todas Organizations del Tenant.

---

# 289. Team Listing

Organization viewer puede o no ver todos Teams.

---

# 290. Scope Visibility

Puede modelarse separadamente de operational authority.

---

# 291. ScopeVisibility

```php
enum ScopeVisibility: string
{
    case Private = 'private';
    case Members = 'members';
    case Tenant = 'tenant';
    case Public = 'public';
}
```

---

# 292. Visibility Not Authority

Ver que existe un Team no implica:

```text
team.manage
```

---

# 293. Scope Sharing

Un Workspace puede compartirse con:

```text
another Team
external group
guest user
```

mediante Relationships/Sharing del documento 21.

---

# 294. No Hierarchy Mutation Needed

No convertir a shared Team en structural parent.

---

# 295. Delegation in Scoped Context

Documento 20 aplica.

Delegated grant deberá incluir:

```text
scope
```

---

# 296. Example

```text
workspace.document.export
scope=Workspace#91
```

---

# 297. Scope Narrowing

Derived delegation no puede ampliar:

```text
Workspace#91
```

a toda Organization.

---

# 298. Impersonation

Impersonated principal no debe cambiar Scope hierarchy.

---

# 299. Actor Restrictions

SupportAgent puede impersonar dentro de:

```text
Organization#14
```

pero no salir de ese support scope.

---

# 300. Service Accounts

ServicePrincipal puede tener:

```text
Role
scope=Tenant
```

o scopes inferiores.

---

# 301. Example

```text
SearchIndexer
workspace.indexer
Workspace#91
```

---

# 302. Better Than Global Service Admin

Reduce blast radius.

---

# 303. Scope-Aware API Keys

Token puede limitarse a:

```text
Organization#14
```

---

# 304. Effective Authority

```text
Service/User grants
INTERSECT
token scope
INTERSECT
requested scope
```

---

# 305. Scope Ceiling

AuthorizationAuthorityCeiling del documento 20 deberá soportar:

```text
scope ceilings
```

---

# 306. Example

Principal tiene Organization Admin.

API token:

```text
scope=Workspace#91
```

Entonces token no podrá administrar otro Workspace.

---

# 307. Compilation

Scope metadata deberá compilarse.

---

# 308. Compile-Time Elements

```text
Scope types
Hierarchy type rules
Ability scope types
Role propagation
Domain propagation
Boundary metadata
Subject scope mappings
```

---

# 309. Runtime Elements

```text
Actual scope IDs
Memberships
Role assignments
Scope status
Scope hierarchy instances
```

---

# 310. No Reflection Hot Path

Subject-to-scope mapping deberá estar precompilado cuando sea posible.

---

# 311. Scope Descriptor IDs

En compiled runtime podrán utilizarse IDs internos.

---

# 312. Internal IDs

No deben almacenarse como public persistent identifiers.

---

# 313. Planner Integration

Propuesta:

```text
AuthorizationRequest
        ↓
ScopeResolution
        ↓
TenantValidation
        ↓
ScopeOperationalStatus
        ↓
MembershipRequirements
        ↓
HierarchicalGrantResolution
        ↓
Resource Relationship Resolution
        ↓
Policy
        ↓
Decision
```

---

# 314. Phase Placement

Scope resolution puede ocurrir:

```text
pre-resolution
```

si route conoce scope.

---

# 315. Subject Scope

Puede requerir:

```text
resource phase
```

---

# 316. PreResolution Example

Route:

```text
/workspaces/{workspace}/documents
```

Workspace se resuelve primero.

Antes de consultar Document:

```text
workspace access
```

puede verificarse.

---

# 317. Resource Example

Document route:

```text
/workspaces/{workspace}/documents/{document}
```

deberá validar:

```text
document.workspace_id == workspace.id
```

antes de Policy.

---

# 318. Scope Binding Security

Esto ayuda contra:

```text
resource ID from another workspace
```

---

# 319. Nested Resource Binding

Deberá estar integrado con data isolation.

---

# 320. Scope Path Verification

```text
Tenant#7
→ Org#14
→ Team#22
→ Workspace#91
```

debe coincidir con route hierarchy si la route expresa todos esos niveles.

---

# 321. Route Scope Metadata

Route compiler podrá generar:

```text
required scope chain
```

---

# 322. Example

```text
tenant > organization > workspace
```

---

# 323. Runtime

Solo compara IDs resueltos.

---

# 324. Avoid Multiple DB Lookups

Scope hierarchy puede precargarse durante route binding.

---

# 325. AuthorizationScopeContext

Request-local:

```php
final readonly class AuthorizationScopeContext
{
    public function __construct(
        public AuthorizationScopeReference $current,
        public AuthorizationScopePath $path,
    ) {}
}
```

---

# 326. Context Immutability

No modificar:

```text
current workspace
```

globalmente durante una autorización anidada.

---

# 327. Scoped Execution

Podrá existir:

```php
Authorization::withinScope($workspace)
    ->authorize(...);
```

conceptualmente.

---

# 328. No Static Current Workspace

Critical for FrankenPHP.

---

# 329. Scope Stack

Nested authorization podrá mantener stack request-local.

---

# 330. Example

Workspace operation llama Project authorization.

Child scope deberá ser descendant válido del parent si el contexto lo requiere.

---

# 331. Scoped Authorization Nesting

Parent:

```text
Workspace#91
```

Child:

```text
Project#401
```

válido si:

```text
Project parent == Workspace#91
```

---

# 332. Wrong Nested Scope

Debe producir:

```text
DENY / context failure
```

según origen.

---

# 333. Performance

El sistema deberá evitar recalcular ScopePath múltiples veces.

---

# 334. Request Scope Memoization

```text
Workspace#91
→ ScopePath
```

una vez.

---

# 335. Hierarchical Grant Memoization

```text
Principal
Ability
Requested Scope
```

una vez por request cuando inputs estables.

---

# 336. Role Expansion

Role → Permissions deberá estar precompilado/cacheado.

---

# 337. Scope Filtering

Assignments irrelevantes no deberán cargarse.

---

# 338. Database Schema Conceptual

Scopes:

```text
authorization_scopes

id
tenant_id
type
parent_type
parent_id
status
authorization_version
```

---

# 339. Memberships

```text
authorization_scope_memberships

id
tenant_id
scope_type
scope_id
principal_type
principal_id
status
attributes
active_from
expires_at
version
```

---

# 340. Scoped Role Assignments

```text
authorization_scope_roles

id
tenant_id
scope_type
scope_id
principal_type
principal_id
role
active_from
expires_at
version
```

---

# 341. Scoped Permissions

Opcional:

```text
authorization_scope_permissions
```

para direct grants.

---

# 342. Indexes

Importantes:

```text
tenant_id + scope_type + scope_id

principal_type + principal_id

scope parent

principal + scope + role
```

---

# 343. Polymorphic Safety

Siempre:

```text
type + id
```

---

# 344. Tenant Qualification

Siempre donde corresponda.

---

# 345. Scope Path Storage

No es obligatorio persistir path materializado.

---

# 346. Options

```text
adjacency list
materialized path
closure table
nested set
external hierarchy provider
```

---

# 347. Authorization Core Neutrality

Core no deberá exigir un único storage strategy.

---

# 348. ScopeHierarchyStoreInterface

```php
interface ScopeHierarchyStoreInterface
{
    public function parentOf(
        AuthorizationScopeReference $scope
    ): ?AuthorizationScopeReference;
}
```

---

# 349. Batch API

```php
public function parentsOf(
    iterable $scopes
): iterable;
```

---

# 350. Closure Table

Puede ser eficiente para:

```text
ancestor checks
```

---

# 351. Materialized Path

También.

---

# 352. Recursive CTE

También.

---

# 353. Choose by Database Adapter

El Database subsystem podrá proveer implementaciones.

---

# 354. External Directory Hierarchy

Organization structure puede venir de:

```text
LDAP
SCIM
HR system
```

---

# 355. Provider Projection

Puede sincronizarse a local store.

---

# 356. Staleness

Debe declarar SLA como en documento 21.

---

# 357. Critical Operations

Podrán exigir validation más fresca.

---

# 358. Audit

Cambios de scope son importantes.

Registrar:

```text
scope created
scope moved
scope deleted
membership added
membership removed
role assigned
role revoked
boundary changed
```

---

# 359. Role Assignment Audit

Debe contener:

```text
actor
target principal
role
scope
source scope
timestamp
```

---

# 360. Inherited Authorization Audit

Para decisiones críticas puede registrar:

```text
source assignment
propagation path
requested scope
```

---

# 361. Example

```text
GRANT team.member.remove

via:
organization.admin

assigned:
Organization#14

target:
Team#22
```

---

# 362. Explainability

Developer/operator output:

```text
User#42 has organization.admin at Organization#14.

The role propagates Membership-domain authority
to descendant Team#22.

Therefore team.member.remove is granted.
```

---

# 363. Public Explanation

Más simple:

```text
Your organization role allows this action.
```

---

# 364. Denial Explanation

```text
Role exists, but does not propagate into this workspace.
```

solo para audience interna autorizada.

---

# 365. Reason Codes

Propuestos:

```text
scope.missing
scope.invalid
scope.type_mismatch
scope.tenant_mismatch
scope.not_descendant
scope.inheritance_blocked
scope.status_suspended
scope.status_read_only

membership.missing
membership.inactive
membership.expired
membership.suspended

role.scope_mismatch
role.propagation_forbidden
role.assignment_expired
role.assignment_forbidden

organization.access_denied
team.access_denied
workspace.access_denied
```

---

# 366. Scope Backend Failure

Si hierarchy es necesaria y no puede resolverse:

```text
FAILURE
```

---

# 367. Membership Provider Failure

Si membership es mandatory:

```text
FAILURE
```

---

# 368. No Ambiguous False

No confundir:

```text
not member
```

con:

```text
membership provider unavailable
```

---

# 369. Testing

El subsistema requerirá pruebas específicas.

---

# 370. Scope Hierarchy Tests

Cubrir:

```text
valid parent
invalid parent type
cross-tenant parent
cycles
max depth
scope move
```

---

# 371. Role Inheritance Tests

```text
exact scope
direct child
descendant
blocked boundary
wrong domain
```

---

# 372. Organization Tests

```text
multiple organizations
same user different roles
organization admin propagation
organization owner protections
```

---

# 373. Team Tests

```text
team membership
team manager
organization admin over team
non-member admin distinction
```

---

# 374. Workspace Tests

```text
workspace admin
workspace guest
isolated workspace
content vs admin authority
```

---

# 375. Tenant Isolation Tests

Same IDs:

```text
Workspace#91 Tenant#7
Workspace#91 Tenant#9
```

must remain isolated.

---

# 376. Scope Move Test

Mover Workspace de Team A a Team B.

Old inherited authority debe desaparecer según consistency model.

---

# 377. Boundary Test

Organization Admin intenta acceder a isolated Workspace.

Resultado según policy:

```text
DENY / ABSTAIN
```

pero nunca accidental GRANT.

---

# 378. Membership Revocation Test

Revocar Organization Membership debe invalidar Roles dependientes.

---

# 379. Role Expiration Test

Expired role no concede authority.

---

# 380. API Token Scope Test

Broad Principal + narrow token scope.

Effective authority must remain narrow.

---

# 381. Delegation Scope Test

Delegation en Workspace A no aplica Workspace B.

---

# 382. Nested Authorization Test

Workspace context + child Project valid.

Wrong Project produces DENY/failure.

---

# 383. Bulk Query Test

Listado de 100 Workspaces debe producir mismos resultados que checks individuales.

---

# 384. Cache Invalidation Test

Cambiar parent o boundary invalida relevant hierarchical decisions.

---

# 385. Persistent Worker Test

Request A:

```text
Workspace#91
```

Request B:

```text
Workspace#92
```

mismo FrankenPHP worker.

No scope leakage.

---

# 386. Property-Based Tests

Propiedad:

```text
A role assigned to Scope A
must never authorize Scope B
unless B is inside an explicitly permitted
propagation path from A.
```

---

# 387. Property

```text
Adding an inheritance boundary
must never increase inherited authority.
```

---

# 388. Property

```text
Moving a scope outside an ancestor
must remove grants that depended exclusively
on that ancestor path.
```

---

# 389. Property

```text
A scoped token or delegation
must never authorize outside its scope ceiling.
```

---

# 390. Property

```text
A role in Tenant A
must never propagate into Tenant B.
```

---

# 391. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Scope/
        ├── Contracts/
        │   ├── AuthorizationScopeInterface.php
        │   ├── AuthorizationScopeResolverInterface.php
        │   ├── ScopeHierarchyResolverInterface.php
        │   ├── ScopeHierarchyStoreInterface.php
        │   ├── ScopeBoundaryPolicyInterface.php
        │   ├── ScopeMembershipProviderInterface.php
        │   └── SubjectScopeProviderInterface.php
        │
        ├── Model/
        │   ├── AuthorizationScopeReference.php
        │   ├── AuthorizationScopeDefinition.php
        │   ├── AuthorizationScopePath.php
        │   ├── AuthorizationScopeContext.php
        │   ├── AuthorizationScopeStatus.php
        │   ├── AuthorizationScopeDomain.php
        │   ├── AuthorizationScopePropagation.php
        │   └── ScopeVisibility.php
        │
        ├── Registry/
        │   ├── AuthorizationScopeRegistry.php
        │   ├── SubjectScopeProviderRegistry.php
        │   └── ScopeBoundaryRegistry.php
        │
        ├── Hierarchy/
        │   ├── ScopeHierarchyManager.php
        │   ├── ScopeHierarchyValidator.php
        │   ├── ScopeCycleDetector.php
        │   ├── ScopePathResolver.php
        │   └── ScopeVersionResolver.php
        │
        ├── Resolution/
        │   ├── AuthorizationScopeResolver.php
        │   ├── AuthorizationScopeResolution.php
        │   ├── AuthorizationScopeSource.php
        │   └── HierarchicalGrantResolver.php
        │
        ├── RBAC/
        │   ├── ScopedRoleAssignment.php
        │   ├── ScopedPermissionGrant.php
        │   ├── ScopedRoleDefinition.php
        │   ├── RoleScopeInheritanceMode.php
        │   ├── AbilityScopeDescriptor.php
        │   └── HierarchicalGrantResolution.php
        │
        ├── Membership/
        │   ├── ScopeMembership.php
        │   ├── ScopeMembershipResult.php
        │   ├── MembershipStatus.php
        │   ├── ScopeMembershipResolver.php
        │   └── CompositeScopeMembershipProvider.php
        │
        ├── Organization/
        │   ├── OrganizationScopeAdapter.php
        │   └── OrganizationAuthorizationPolicy.php
        │
        ├── Team/
        │   ├── TeamScopeAdapter.php
        │   └── TeamAuthorizationPolicy.php
        │
        ├── Workspace/
        │   ├── WorkspaceScopeAdapter.php
        │   └── WorkspaceAuthorizationPolicy.php
        │
        ├── Evaluation/
        │   ├── ScopeOperationalEvaluator.php
        │   ├── ScopeMembershipEvaluator.php
        │   ├── HierarchicalRoleEvaluator.php
        │   └── ScopeBoundaryEvaluator.php
        │
        ├── Query/
        │   ├── AuthorizedScopeQueryInterface.php
        │   └── AuthorizedScopeQueryResolver.php
        │
        ├── Cache/
        │   ├── ScopePathCache.php
        │   ├── ScopeGrantMemoizer.php
        │   └── ScopeAuthorizationVersion.php
        │
        └── Exceptions/
            ├── AuthorizationScopeException.php
            ├── InvalidScopeHierarchyException.php
            ├── ScopeCycleException.php
            ├── ScopeMembershipException.php
            └── ScopedRoleAssignmentException.php
```

---

# 392. Scope Invariants

### Invariante 1

Toda authority scoped tiene un scope canonical.

### Invariante 2

Scopes pertenecientes a distintos Tenants no forman una hierarchy válida.

### Invariante 3

Scope hierarchy no contiene cycles.

### Invariante 4

Scope IDs y display names son conceptos separados.

### Invariante 5

La hierarchy estructural no se modifica por simples shares o relationships.

---

# 393. Role Invariants

### Invariante 1

Un Role scoped no se interpreta globalmente.

### Invariante 2

Role inheritance debe ser explícita.

### Invariante 3

Ability propagation puede restringir propagation del Role.

### Invariante 4

Scope boundaries detienen authority heredada.

### Invariante 5

Expired Role assignments no conceden authority.

---

# 394. Membership Invariants

### Invariante 1

Membership no equivale a admin authority.

### Invariante 2

Admin authority no implica business membership.

### Invariante 3

Inactive membership no sostiene Roles dependientes.

### Invariante 4

External guests pueden modelarse sin falsificar Organization membership.

### Invariante 5

Membership provider failure no se interpreta como valid membership.

---

# 395. Organization Invariants

### Invariante 1

Un usuario puede tener diferentes Roles en diferentes Organizations.

### Invariante 2

Organization authority no cruza Organizations hermanas.

### Invariante 3

Organization Admin no implica Tenant Admin.

### Invariante 4

Organization membership no concede acceso automático a todos sus recursos.

### Invariante 5

Organization Roles respetan propagation domains.

---

# 396. Team Invariants

### Invariante 1

Organization membership no implica Team membership.

### Invariante 2

Organization Admin puede administrar Teams sin convertirse necesariamente en Team Member.

### Invariante 3

Team Roles aplican únicamente al scope permitido.

### Invariante 4

Nested Team authority requiere propagation explícita.

### Invariante 5

Team authority no cruza el Tenant boundary.

---

# 397. Workspace Invariants

### Invariante 1

Workspace administrative authority y content authority pueden diferir.

### Invariante 2

Workspace isolation puede bloquear inherited authority.

### Invariante 3

Workspace guests poseen authority explícitamente limitada.

### Invariante 4

Workspace Roles no sustituyen Resource Policies.

### Invariante 5

Mover Workspace debe recalcular su autoridad heredada.

---

# 398. Multi-Tenant Invariants

### Invariante 1

Tenant scope es una frontera independiente.

### Invariante 2

Same-ID scopes de distintos Tenants permanecen separados.

### Invariante 3

Roles, Memberships y Permissions están tenant-qualified cuando corresponde.

### Invariante 4

Platform authority no entra automáticamente a Tenant scopes.

### Invariante 5

Cross-tenant hierarchy está prohibida por default.

---

# 399. Runtime Invariants

### Invariante 1

AuthorizationScopeContext es request/execution scoped.

### Invariante 2

No existe `static currentWorkspace`.

### Invariante 3

Nested scope contexts se restauran en `finally`.

### Invariante 4

FrankenPHP worker reuse no mezcla scopes.

### Invariante 5

Cached ScopePaths nunca contienen Principal mutable state.

---

# 400. Security Invariants

### Invariante 1

Scope inheritance nunca produce authority fuera del path permitido.

### Invariante 2

Un boundary nunca puede aumentar authority.

### Invariante 3

Mover un scope no puede convertirse en privilege escalation inadvertida.

### Invariante 4

Role assignment requiere authorization y scope validation.

### Invariante 5

Delegation, token scope e impersonation respetan scope ceilings.

---

# 401. Arquitectura general

```text
                         PRINCIPAL
                            │
                            ↓
                    Tenant Boundary
                            │
                            ↓
                   Requested Scope
                            │
                            ↓
                    Scope Path Resolver
                            │
          ┌─────────────────┼─────────────────┐
          ↓                 ↓                 ↓
      Memberships       Scoped Roles      Direct Grants
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ↓
                  Propagation Evaluation
                            │
                    ┌───────┴────────┐
                    ↓                ↓
                 Allowed          Boundary
                    │                │
                    ↓                ↓
             Effective Grant       STOP
                    │
                    ↓
          Resource / Relationship Layer
                    │
                    ↓
                 Policies
                    │
                    ↓
                  Decision
```

---

# 402. Organization-Team-Workspace Model

```text
Tenant#7
│
├── Organization#14
│   │
│   ├── Team#22
│   │   │
│   │   ├── Workspace#91
│   │   │   ├── Project#400
│   │   │   └── Project#401
│   │   │
│   │   └── Workspace#92
│   │
│   └── Team#23
│
└── Organization#15
```

User#42:

```text
organization.admin @ Organization#14

workspace.editor @ Workspace#92

viewer @ Organization#15
```

La autoridad deberá resolverse en función del scope solicitado, nunca mediante un único atributo global del usuario.

---

# 403. Ejemplo — Organization Admin

User#42:

```text
organization.admin
scope=Organization#14
```

Solicita:

```text
team.members.manage
Team#22
```

Scope path:

```text
Organization#14
→ Team#22
```

Role propagation:

```text
Membership domain
→ descendants
```

Resultado:

```text
GRANT
```

---

# 404. Ejemplo — Organization Admin no puede leer contenido aislado

Mismo User.

Solicita:

```text
document.view
Document#100
Workspace#91
```

Workspace#91:

```text
isolated_content=true
```

Role propagation:

```text
organization.admin
Content domain
→ none
```

Resultado:

```text
ABSTAIN / DENY
```

salvo otra authority.

---

# 405. Ejemplo — Workspace Editor

User#42:

```text
workspace.editor
scope=Workspace#91
```

Solicita:

```text
document.update
Document#100
```

Document pertenece a:

```text
Workspace#91
```

Pipeline:

```text
Tenant Isolation
→ GRANT

Scope Resolution
→ Workspace#91

Hierarchical Role
→ workspace.editor
→ GRANT

Document Policy
→ GRANT

Final
→ GRANT
```

---

# 406. Ejemplo — Wrong Workspace

User mantiene:

```text
workspace.editor @ Workspace#91
```

Solicita:

```text
document.update
Document#200
Workspace#92
```

Resultado:

```text
Role does not apply
→ ABSTAIN

No other authority
→ DENY
```

---

# 407. Ejemplo — Team Manager

```text
User#50
team.manager
Team#22
```

Solicita:

```text
workspace.settings.update
Workspace#91
```

Role:

```text
Administration domain
→ descendants
```

Team#22:

```text
parent of Workspace#91
```

Resultado:

```text
GRANT
```

---

# 408. Ejemplo — Team Manager no es Workspace Member

Mismo User solicita:

```text
workspace.chat.post
```

Ability exige:

```text
explicit workspace membership
```

Team admin authority:

```text
GRANT administrative authority
```

Membership requirement:

```text
DENY
```

Final:

```text
DENY
```

---

# 409. Ejemplo — Isolated Workspace

Organization#14:

```text
User#42
organization.admin
```

Workspace#91:

```text
authorization_boundary=true
```

Solicita:

```text
workspace.members.manage
```

Propagation:

```text
Organization#14
→ Team#22
→ Workspace#91
```

Boundary:

```text
STOP
```

Resultado:

```text
DENY
```

salvo direct Role en Workspace.

---

# 410. Ejemplo — Direct Role supera falta de inherited grant

User tiene además:

```text
workspace.admin
Workspace#91
```

Boundary corta parent inheritance, pero direct Role es válido.

Resultado:

```text
GRANT
```

---

# 411. Ejemplo — Token Scope

User#42:

```text
organization.admin @ Organization#14
```

API token:

```text
scope ceiling:
Workspace#91
```

Solicita:

```text
workspace.settings.update
Workspace#92
```

Role podría aplicar vía Organization.

Pero authority ceiling:

```text
Workspace#91 only
```

Resultado:

```text
DENY
```

---

# 412. Ejemplo — Delegation

Organization Admin delega:

```text
workspace.members.manage
Workspace#91
```

a User#80.

Delegated grant:

```text
scope=Workspace#91
```

User#80 intenta administrar:

```text
Workspace#92
```

Resultado:

```text
delegation.scope_mismatch
→ DENY
```

---

# 413. Ejemplo — Moving Workspace

Workspace#91:

```text
parent=Team#22
```

User#42 posee inherited authority desde Team#22.

Workspace se mueve a:

```text
Team#23
```

Después:

```text
scope_hierarchy_version++
authorization caches invalidated
```

Nueva autorización de User#42:

```text
old inherited path no longer valid
```

---

# 414. Ejemplo — Membership Revocation

User#42:

```text
organization.member
Organization#14

organization.admin Role Assignment
Organization#14
```

Membership pasa a:

```text
revoked
```

Role assignment sigue físicamente almacenado por error o historial.

MembershipEvaluator:

```text
DENY/invalid assignment context
```

Authority no deberá mantenerse.

---

# 415. Ejemplo — Guest

External User#99:

```text
guest_of Workspace#91

workspace.viewer
scope=Workspace#91

expires=2026-09-01
```

Puede:

```text
document.view
```

pero no:

```text
document.export
workspace.share
workspace.members.manage
```

según Role/ABAC.

---

# 416. Modelo de resolución completo

```text
AUTHORIZATION REQUEST
        │
        ↓
Principal / Actor
        │
        ↓
TenantContext
        │
        ↓
Requested Scope
        │
        ↓
Scope Path
        │
        ↓
Scope Status
        │
        ↓
Membership
        │
        ↓
Scoped Role Assignments
        │
        ↓
Permission Expansion
        │
        ↓
Scope Propagation Rules
        │
        ↓
Scope Boundaries
        │
        ↓
Authority Ceiling
        │
        ↓
Resource Ownership / Shares / ReBAC
        │
        ↓
Resource Policy / ABAC
        │
        ↓
NonBypassable Security Rules
        │
        ↓
FINAL DECISION
```

---

# 417. Filosofía arquitectónica

VoltStack deberá adoptar estas reglas:

```text
A role has meaning only inside its scope.

Membership and authority are not the same thing.

Parent authority propagates only when explicitly allowed.

Scope boundaries are security boundaries.

Organizations, Teams and Workspaces are specializations
of one generic scoped authorization model.

Administrative authority does not automatically imply
content authority.

Structural hierarchy should remain simple.

Additional relationships belong to ReBAC.

Tenant isolation remains independent and non-bypassable.

Delegations, capabilities and API tokens can only narrow
scoped authority, never broaden it accidentally.
```

---

# 418. Resultado esperado

`22_AUTHORIZATION_HIERARCHICAL_SCOPES_ORGANIZATIONS_TEAMS_AND_WORKSPACES_SYSTEM.md` permitirá que VoltStack represente estructuras como:

```text
Global Platform

Multi-Tenant SaaS

Enterprise Organizations

Subsidiaries

Business Units

Departments

Teams

Workspaces

Projects

Resource Collections

External Guests

Service Accounts

Scoped API Tokens
```

sin introducir un `admin` global que destruya el modelo de seguridad.

La arquitectura definitiva será:

```text
PRINCIPAL
    ↓
TENANT
    ↓
AUTHORIZATION SCOPE
    ↓
SCOPE HIERARCHY
    ↓
MEMBERSHIP
    ↓
SCOPED ROLES / PERMISSIONS
    ↓
PROPAGATION
    ↓
BOUNDARIES
    ↓
RESOURCE RELATIONSHIPS
    ↓
POLICIES
    ↓
DECISION
```

El principio definitivo será:

> **En VoltStack, la autoridad no vive únicamente en el usuario ni únicamente en el rol: vive en la relación entre Principal, Ability y Scope.**

Así, un usuario podrá ser simultáneamente:

```text
Admin aquí,
Editor allá,
Viewer en otro lugar,
y completamente no autorizado fuera de esos scopes.
```

sin contradicción y sin recurrir a hacks de aplicación.

Con este subsistema queda formalizada la capa jerárquica necesaria para que **RBAC, ReBAC, Multi-Tenancy, Organizations, Teams, Workspaces, Delegation y Resource Authorization** operen bajo un único modelo de scopes explícitos.