# VoltStack Authorization System
## Resource Ownership, Sharing and Relationship Access System

**Documento:** `21_AUTHORIZATION_RESOURCE_OWNERSHIP_SHARING_AND_RELATIONSHIP_ACCESS_SYSTEM.md`  
**Sistema:** Authorization  
**Framework:** VoltStack  
**Módulo sugerido:** `Quantum/Authorization`  
**Estado:** Especificación arquitectónica  
**Versión objetivo:** 1.x+

---

# 1. Propósito

Este documento define la arquitectura del sistema de **propiedad de recursos, compartición y autorización basada en relaciones** de VoltStack.

El subsistema deberá responder de forma consistente preguntas como:

```text
¿Este usuario es propietario del recurso?

¿El recurso pertenece a su organización?

¿Fue compartido directamente con él?

¿Fue compartido con uno de sus equipos?

¿Tiene acceso mediante una relación con el propietario?

¿Es administrador del workspace que contiene el recurso?

¿Puede visualizarlo pero no modificarlo?

¿Puede compartirlo con terceros?

¿Puede transferir su propiedad?

¿Puede acceder mediante una relación heredada?

¿Puede acceder porque pertenece a un proyecto padre?

¿Puede revocarse ese acceso inmediatamente?
```

El sistema complementará:

```text
RBAC
ABAC
ReBAC
Policies
Gates
Delegation
Capabilities
Multi-Tenancy
```

sin convertir ninguna de estas técnicas en un reemplazo de las demás.

---

# 2. Principio fundamental

VoltStack distinguirá claramente:

```text
Permission
≠
Ownership
≠
Relationship
≠
Sharing
≠
Delegation
```

Por ejemplo:

```text
User has permission:
document.update
```

no implica:

```text
User may update every Document.
```

La decisión real puede requerir:

```text
document.update
AND
(
    owns(document)
    OR
    document shared as editor
    OR
    manages(document.workspace)
)
AND
tenant isolation
AND
document policy
```

---

# 3. Objetivos

El sistema deberá proporcionar:

1. resource ownership;
2. direct ownership;
3. organizational ownership;
4. team ownership;
5. workspace ownership;
6. resource sharing;
7. direct user sharing;
8. group sharing;
9. team sharing;
10. role-based sharing;
11. relationship-based authorization;
12. relationship inheritance;
13. hierarchical resources;
14. access levels;
15. ACL integration;
16. ReBAC integration;
17. ownership transfer;
18. share revocation;
19. expiration;
20. tenant isolation;
21. resource visibility;
22. public sharing;
23. link sharing;
24. nested containers;
25. relationship graph traversal;
26. cycle protection;
27. decision explainability;
28. caching;
29. auditability;
30. scalable relationship resolution.

---

# 4. Ownership

`Ownership` representa una relación explícita entre:

```text
Principal
    ↓
owns
    ↓
Resource
```

Ejemplo:

```text
User#42
owns
Document#100
```

---

# 5. Ownership no es Permission

No deberá implementarse simplemente como:

```php
$user->hasPermission('document.owner');
```

La propiedad depende de un recurso concreto.

---

# 6. Resource Owner

Conceptualmente:

```php
interface OwnableResourceInterface
{
    public function authorizationOwner(): ?PrincipalReference;
}
```

Sin embargo, VoltStack no deberá obligar a todos los modelos a implementar esta interfaz.

También podrá resolver ownership mediante metadata.

---

# 7. OwnershipResolver

Contrato:

```php
interface OwnershipResolverInterface
{
    public function resolve(
        AuthorizationSubject $subject,
        AuthorizationContext $context
    ): OwnershipResolution;
}
```

---

# 8. OwnershipResolution

Podrá contener:

```php
final readonly class OwnershipResolution
{
    public function __construct(
        public bool $owned,
        public ?PrincipalReference $owner,
        public ?OwnershipType $type,
        public ?RelationshipPath $path = null,
    ) {}
}
```

---

# 9. Ownership Types

```php
enum OwnershipType: string
{
    case Direct = 'direct';
    case User = 'user';
    case Team = 'team';
    case Organization = 'organization';
    case Workspace = 'workspace';
    case Tenant = 'tenant';
    case Inherited = 'inherited';
}
```

---

# 10. Direct Ownership

Ejemplo:

```text
Document.owner_id = User#42
```

Entonces:

```text
User#42
→ OWNER
```

---

# 11. Organization Ownership

Un recurso puede pertenecer a:

```text
Organization#7
```

en vez de a una persona.

Ejemplo:

```text
Repository#80
owner = Organization#7
```

---

# 12. Team Ownership

También:

```text
Team#12
owns
Project#55
```

---

# 13. Workspace Ownership

Para aplicaciones colaborativas:

```text
Workspace#5
owns
Document#100
```

El acceso podrá derivarse de la relación del usuario con el Workspace.

---

# 14. Tenant Ownership

En determinados dominios:

```text
Tenant#7
owns
Resource
```

pero esto no significa que todos los usuarios del Tenant puedan utilizarlo.

---

# 15. Ownership vs Tenant Isolation

Son conceptos distintos.

```text
Tenant membership
```

solo establece:

```text
possible isolation boundary
```

No necesariamente:

```text
resource access
```

---

# 16. Ejemplo

```text
User#42
belongs Tenant#7

Document#100
belongs Tenant#7
```

Esto evita cross-tenant access.

Pero todavía puede requerirse:

```text
owns
shared
team membership
workspace membership
permission
```

---

# 17. Resource Ownership Metadata

VoltStack podrá declarar:

```php
#[AuthorizationOwner('user')]
private int $ownerId;
```

conceptualmente.

---

# 18. Declarative Ownership

También podría declararse:

```php
#[OwnedBy('owner')]
final class Document
{
}
```

---

# 19. External Ownership Resolver

Para dominios complejos:

```php
final class DocumentOwnershipResolver
    implements OwnershipResolverInterface
{
}
```

---

# 20. Ownership Registry

VoltStack deberá mantener:

```text
Subject Type
    ↓
Ownership Resolver
```

---

# 21. Example Registry

```text
Document
→ DocumentOwnershipResolver

Project
→ ProjectOwnershipResolver

Invoice
→ OrganizationOwnershipResolver
```

---

# 22. OwnershipEvaluator

Podrá existir:

```text
OwnershipEvaluator
```

dentro del Authorization Planner.

---

# 23. Example

Ability:

```text
document.update
```

Policy:

```text
permission
AND
owner
```

---

# 24. Ownership Requirement

Podrá expresarse declarativamente:

```php
#[RequiresOwnership]
public function update(Document $document)
{
}
```

---

# 25. Pero

`RequiresOwnership` no deberá significar necesariamente:

```text
owner_id === user_id
```

Debe delegar al Ownership System.

---

# 26. Multiple Owners

Algunos dominios podrán permitir:

```text
multiple owners
```

---

# 27. OwnershipSet

Entonces:

```php
final readonly class OwnershipSet
{
    /** @param PrincipalReference[] $owners */
    public function __construct(
        public array $owners
    ) {}
}
```

---

# 28. V1 Recommendation

VoltStack debería soportar:

```text
single owner
+
organizational owner
```

desde V1.

Multiple ownership puede existir mediante Relationship Graph.

---

# 29. Ownership Transfer

Operación explícita:

```text
resource.transfer_ownership
```

---

# 30. Transfer != Update

Tener:

```text
document.update
```

no significa automáticamente:

```text
document.transfer_ownership
```

---

# 31. Transfer Policy

Ejemplo:

```text
current owner
AND
target belongs same tenant
AND
resource transferable
```

---

# 32. OwnershipTransferManager

Conceptualmente:

```php
interface OwnershipTransferManagerInterface
{
    public function transfer(
        AuthorizationSubject $subject,
        PrincipalReference $newOwner,
        AuthorizationContext $context
    ): void;
}
```

---

# 33. Authorization First

El manager deberá autorizar:

```text
resource.transfer_ownership
```

antes de realizar la operación.

---

# 34. Atomic Transfer

Cuando sea posible:

```text
authorize
validate target
transfer
audit
```

deberá formar una unidad consistente.

---

# 35. Concurrent Transfer

Debe evitar:

```text
Owner A transfers to B

while

Owner A transfers to C
```

sin control.

---

# 36. Versioning

Puede usarse:

```text
ownership_version
```

para optimistic concurrency y cache invalidation.

---

# 37. Sharing

Sharing representa otorgar acceso específico a un recurso.

---

# 38. Share

Modelo conceptual:

```text
Resource
   ↓
shared with
   ↓
Principal
   ↓
Access Level
```

---

# 39. Example

```text
Document#100
shared with
User#57
as
Editor
```

---

# 40. ResourceShare

```php
final readonly class ResourceShare
{
    public function __construct(
        public string $id,
        public SubjectReference $resource,
        public ShareTarget $target,
        public AccessLevel $access,
        public PrincipalReference $grantedBy,
        public DateTimeImmutable $createdAt,
        public ?DateTimeImmutable $expiresAt = null,
    ) {}
}
```

---

# 41. Share Target

Puede ser:

```text
User
Team
Group
Organization
Workspace Role
Public
Capability
```

---

# 42. ShareTargetType

```php
enum ShareTargetType: string
{
    case Principal = 'principal';
    case User = 'user';
    case Team = 'team';
    case Group = 'group';
    case Organization = 'organization';
    case WorkspaceRole = 'workspace_role';
    case Public = 'public';
}
```

---

# 43. Direct User Sharing

```text
Document#100
→ User#57
→ editor
```

---

# 44. Team Sharing

```text
Document#100
→ Team#8
→ viewer
```

Todos los miembros válidos del Team podrán obtener acceso.

---

# 45. Group Sharing

Útil para:

```text
departments
security groups
distribution groups
custom application groups
```

---

# 46. Organization Sharing

Ejemplo:

```text
Document#100
shared with Organization#7
as viewer
```

---

# 47. Workspace Role Sharing

Ejemplo:

```text
All Workspace#5 editors
may edit Document#100
```

---

# 48. Access Levels

No deberían ser hardcoded globalmente.

VoltStack podrá ofrecer una abstracción:

```php
interface ResourceAccessLevelInterface
{
    public function abilities(): array;
}
```

---

# 49. Common Access Levels

Podrán existir:

```text
viewer
commenter
editor
manager
owner
```

---

# 50. Pero

`owner` debería tratarse cuidadosamente.

Un share:

```text
access=owner
```

no debería transferir ownership automáticamente.

---

# 51. Recommendation

Mantener:

```text
Ownership
```

separado de:

```text
ShareAccessLevel
```

---

# 52. Example Access Mapping

```text
viewer:
    document.view

commenter:
    document.view
    document.comment

editor:
    document.view
    document.comment
    document.update

manager:
    document.view
    document.comment
    document.update
    document.share
```

---

# 53. Access Level Registry

```text
Resource Type
+
Access Level
    ↓
Abilities
```

---

# 54. Why Resource-Specific

`editor` en Document puede significar algo diferente a `editor` en Dashboard.

---

# 55. ResourceAccessRegistry

Conceptualmente:

```php
interface ResourceAccessRegistryInterface
{
    public function abilitiesFor(
        string $subjectType,
        string $accessLevel
    ): array;
}
```

---

# 56. Share Does Not Bypass Policy

Un share `editor` puede otorgar:

```text
document.update
```

pero la Policy puede denegar si:

```text
document.locked
document.archived
legal_hold
tenant_suspended
```

---

# 57. Share as Authority Source

Sharing deberá convertirse en:

```text
Authorization Authority Source
```

dentro del pipeline.

---

# 58. Authority Source

Podrá añadirse:

```php
case ResourceShare = 'resource_share';
```

al modelo de Authority Source.

---

# 59. ShareEvaluator

```text
ResourceShareEvaluator
```

resolverá si existe un share válido para:

```text
Principal
Ability
Subject
Context
```

---

# 60. Share Validity

Deberá comprobar:

```text
active
not expired
target matches
resource matches
ability included
tenant matches
conditions match
```

---

# 61. Share Expiration

Ejemplo:

```text
External consultant
can edit Document#100
until Friday.
```

---

# 62. Expired Share

```text
DENY
```

como fuente de authority.

---

# 63. Share Revocation

Debe poder revocarse explícitamente.

---

# 64. ResourceShareStatus

```php
enum ResourceShareStatus: string
{
    case Active = 'active';
    case Revoked = 'revoked';
    case Expired = 'expired';
}
```

---

# 65. Share Revocation Consistency

Para recursos sensibles:

```text
revocation
→ effective immediately
```

debe ser objetivo arquitectónico.

---

# 66. Share Repository

```php
interface ResourceShareRepositoryInterface
{
    public function findApplicableShares(
        PrincipalReference $principal,
        SubjectReference $subject,
        AuthorizationContext $context
    ): iterable;
}
```

---

# 67. Avoid N+1

No consultar un share por cada Policy individual.

---

# 68. Batch Resolution

Para colecciones:

```text
resolve shares for:
User#42
+
Document IDs [1...100]
```

---

# 69. Relationship-Based Access

ReBAC determina autorización mediante relaciones entre entidades.

---

# 70. Relationship Model

Ejemplos:

```text
User#42
member_of
Team#8
```

```text
Team#8
member_of
Workspace#5
```

```text
Workspace#5
owns
Document#100
```

Entonces:

```text
User#42
may have relationship path
to Document#100
```

---

# 71. Authorization Graph

Conceptualmente:

```text
Principal
   │
   ├── member_of → Team
   │                  │
   │                  └── member_of → Workspace
   │                                      │
   │                                      └── owns → Document
   │
   └── shared_with → Document
```

---

# 72. Relationship Edge

```php
final readonly class AuthorizationRelationship
{
    public function __construct(
        public RelationshipNode $source,
        public string $relation,
        public RelationshipNode $target,
    ) {}
}
```

---

# 73. Nodes

Podrán representar:

```text
User
Team
Group
Organization
Workspace
Project
Folder
Document
Tenant
Service
```

---

# 74. Relationship Types

Ejemplos:

```text
owns
member_of
manager_of
editor_of
viewer_of
parent_of
contains
shared_with
assigned_to
belongs_to
created_by
```

---

# 75. Relationship Registry

No cualquier string arbitrario deberá aceptarse.

---

# 76. RelationshipDefinition

```php
final readonly class RelationshipDefinition
{
    public function __construct(
        public string $name,
        public string $sourceType,
        public string $targetType,
        public bool $transitive = false,
    ) {}
}
```

---

# 77. Typed Relationships

Ejemplo:

```text
User
member_of
Team
```

válido.

Pero:

```text
Document
member_of
User
```

podría ser inválido.

---

# 78. Compiler Validation

Relationship metadata podrá validarse durante compilation.

---

# 79. Direct Relationship

```text
User#42
editor_of
Document#100
```

---

# 80. Indirect Relationship

```text
User#42
member_of
Team#8

Team#8
editor_of
Document#100
```

---

# 81. Relationship Path

```text
User#42
→ member_of Team#8
→ editor_of Document#100
```

---

# 82. RelationshipPath

```php
final readonly class RelationshipPath
{
    /** @param AuthorizationRelationship[] $edges */
    public function __construct(
        public array $edges
    ) {}
}
```

---

# 83. Relationship Access Rule

Una ability podrá declarar:

```text
document.update
allowed when:
owner
OR
editor
OR
workspace_manager
```

---

# 84. ReBAC Rule

Conceptualmente:

```text
document.update =
owner
OR
editor
OR
manager_of(parent_workspace)
```

---

# 85. RelationshipAccessPolicy

Contrato:

```php
interface RelationshipAccessPolicyInterface
{
    public function relationshipsFor(
        Ability $ability,
        AuthorizationSubject $subject
    ): RelationshipRequirement;
}
```

---

# 86. Declarative Relationships

Podrá expresarse mediante metadata:

```php
#[AuthorizeRelationship(
    ability: 'document.update',
    anyOf: ['owner', 'editor', 'workspace_manager']
)]
```

---

# 87. Policies Remain Preferred for Domain Logic

Metadata es adecuada para relaciones estructurales.

Policies siguen siendo mejores para:

```text
complex domain conditions
state-dependent rules
business invariants
```

---

# 88. ReBAC Evaluator

```text
RelationshipAccessEvaluator
```

deberá:

```text
resolve requirements
query graph
validate path
normalize result
```

---

# 89. Graph Traversal

El sistema deberá soportar traversal limitado.

---

# 90. Never Unlimited Recursive Traversal

Debe existir:

```text
max_depth
```

---

# 91. Why

Evita:

```text
cycles
runaway queries
DoS
unexpected inheritance
```

---

# 92. Cycle Detection

Ejemplo accidental:

```text
Team A
member_of
Team B

Team B
member_of
Team A
```

El resolver no deberá entrar en loop.

---

# 93. Visited Nodes

Traversal deberá mantener:

```text
visited set
```

---

# 94. Depth Limit

Configuración conceptual:

```text
authorization.relationships.max_depth = 8
```

---

# 95. Different Rules, Different Depth

Una relación podrá declarar:

```text
max_depth=1
```

mientras otra:

```text
max_depth=5
```

---

# 96. Transitive Relationships

Ejemplo:

```text
member_of
```

podría ser transitiva.

---

# 97. Non-Transitive Relationship

```text
created_by
```

normalmente no lo sería.

---

# 98. Inheritance

Recursos jerárquicos necesitan reglas específicas.

---

# 99. Example

```text
Workspace
   ↓
Project
   ↓
Folder
   ↓
Document
```

---

# 100. Parent Access Inheritance

Si User es:

```text
Workspace Editor
```

podría heredar:

```text
edit Project
edit Folder
edit Document
```

---

# 101. But Not Always

Algunos recursos podrán declarar:

```text
inherit_access=false
```

---

# 102. Access Boundary

Ejemplo:

```text
Confidential Folder
```

puede cortar herencia.

---

# 103. Relationship Boundary

Podrá existir:

```text
AuthorizationInheritanceBoundary
```

---

# 104. Example

```text
Workspace Editor
        ↓
Project
        ↓
Confidential Folder
        X
Document
```

---

# 105. InheritancePolicy

```php
interface AuthorizationInheritancePolicyInterface
{
    public function mayInheritThrough(
        AuthorizationSubject $subject,
        AuthorizationContext $context
    ): bool;
}
```

---

# 106. Resource Hierarchy Resolver

```php
interface ResourceHierarchyResolverInterface
{
    public function parentOf(
        AuthorizationSubject $subject
    ): ?AuthorizationSubject;
}
```

---

# 107. Hierarchy Metadata

Podrá declararse:

```php
#[AuthorizationParent('folder')]
private Folder $folder;
```

conceptualmente.

---

# 108. Nested Containers

Ejemplo:

```text
Folder A
  Folder B
    Folder C
      Document
```

---

# 109. Inherited Sharing

Si Folder A se comparte con Team#8:

```text
viewer
```

¿Document hereda?

Depende de:

```text
share inheritance policy
```

---

# 110. ShareInheritanceMode

```php
enum ShareInheritanceMode: string
{
    case None = 'none';
    case Children = 'children';
    case Descendants = 'descendants';
}
```

---

# 111. Default Recommendation

Para V1:

```text
explicit inheritance metadata
```

No asumir inheritance global.

---

# 112. Direct Share vs Inherited Share

Deben distinguirse.

---

# 113. Example

```text
Document#100

direct:
User#42 = viewer

inherited:
Team#8 = editor
```

---

# 114. Conflict

¿Qué acceso gana?

---

# 115. Access Combination Strategy

VoltStack deberá definir estrategia explícita.

---

# 116. Possible Strategies

```text
MostPermissive
MostRestrictive
ExplicitDenyWins
NearestRelationship
Custom
```

---

# 117. Recommended Default

Para authorization general:

```text
explicit DENY
>
GRANT
```

cuando existan denials formales.

---

# 118. Pero Shares

Un share normalmente representa:

```text
positive grant
```

No necesariamente un deny.

---

# 119. Share Removal

Para quitar acceso:

```text
revoke share
```

en lugar de crear:

```text
negative share
```

como default.

---

# 120. Negative ACLs

Podrán soportarse en sistemas avanzados, pero complican inheritance.

---

# 121. V1 Recommendation

```text
positive shares
+
Policy DENY
+
security DENY
```

sin negative ACL arbitrario.

---

# 122. ACL

Access Control Lists pueden implementarse como representación de Resource Shares.

---

# 123. Example ACL

```text
Document#100 ACL

User#42      viewer
User#57      editor
Team#8       commenter
Organization#7 viewer
```

---

# 124. ACL Is Storage Model

No debería convertirse en un Authorization Engine separado.

---

# 125. Unified Model

```text
ACL
    ↓
Resource Shares
    ↓
Relationship/Share Evaluator
    ↓
Authorization Decision
```

---

# 126. Why

Evita mantener:

```text
Policies engine
Gate engine
ACL engine
Sharing engine
```

con decisiones incompatibles.

---

# 127. Resource Membership

Algunos recursos representan containers.

Ejemplo:

```text
Project
```

puede tener:

```text
ProjectMember
```

---

# 128. Project Membership

Puede mapearse a relaciones:

```text
User
viewer_of
Project
```

o:

```text
User
member_of
Project
with role=editor
```

---

# 129. Relationship Attributes

Las relaciones podrán tener atributos.

---

# 130. Example

```text
User#42
member_of
Project#10

attributes:
role=editor
joined_at=...
expires_at=...
```

---

# 131. RelationshipEdgeAttributes

Conceptualmente:

```php
final readonly class RelationshipEdgeAttributes
{
    public function __construct(
        public array $attributes
    ) {}
}
```

---

# 132. ReBAC + ABAC

Esto permite:

```text
relationship:
member_of Project

AND

relationship.role == editor
```

---

# 133. Example

```text
User#42
member_of Project#10
role=reviewer
```

Puede:

```text
document.review
```

pero no:

```text
document.publish
```

---

# 134. Temporal Relationships

Una relación puede expirar.

---

# 135. Example

```text
Consultant#7
member_of
Project#10

until:
2026-09-30
```

---

# 136. Expired Relationship

No deberá participar en access paths.

---

# 137. Relationship Status

```php
enum RelationshipStatus: string
{
    case Active = 'active';
    case Revoked = 'revoked';
    case Expired = 'expired';
}
```

---

# 138. Relationship Provenance

Una relación podrá provenir de:

```text
manual assignment
directory sync
team membership
resource share
organization membership
automation
```

---

# 139. Provenance Matters

Especialmente para:

```text
audit
revocation
debugging
```

---

# 140. Relationship Source

```php
enum RelationshipSource: string
{
    case Direct = 'direct';
    case Share = 'share';
    case Membership = 'membership';
    case Inherited = 'inherited';
    case Directory = 'directory';
    case External = 'external';
}
```

---

# 141. Relationship Store

VoltStack deberá abstraer almacenamiento.

---

# 142. Contract

```php
interface RelationshipStoreInterface
{
    public function findRelationships(
        RelationshipQuery $query
    ): RelationshipResultSet;
}
```

---

# 143. Storage Backend

Podrá implementarse sobre:

```text
SQL
graph database
Redis projections
external authorization graph
custom provider
```

---

# 144. Core Must Not Depend on Graph Database

ReBAC no obliga a usar Neo4j u otra base orientada a grafos.

---

# 145. SQL Backend

Para muchos sistemas será suficiente.

Ejemplo tabla:

```text
authorization_relationships
```

---

# 146. Conceptual Schema

```text
id
tenant_id
source_type
source_id
relation
target_type
target_id
attributes
status
created_at
expires_at
version
```

---

# 147. Indexing

Índices importantes:

```text
(source_type, source_id, relation)

(target_type, target_id, relation)

(tenant_id, source_type, source_id)

(tenant_id, target_type, target_id)
```

---

# 148. Polymorphic IDs

VoltStack deberá evitar ambigüedad entre:

```text
User#10
Team#10
Document#10
```

---

# 149. Node Identity

Usar:

```text
type + id
```

como identidad lógica.

---

# 150. Tenant Qualification

En multi-tenancy:

```text
tenant + type + id
```

puede formar el namespace efectivo.

---

# 151. Cross-Tenant Relationships

Por default:

```text
forbidden
```

---

# 152. Exceptions

Solo relaciones explícitamente:

```text
cross_tenant=true
```

podrán cruzar boundaries.

---

# 153. Example

Platform support relationship podría ser global.

---

# 154. But

Debe pasar por:

```text
MultiTenantAuthorizationEvaluator
```

y las reglas de impersonation/delegation correspondientes.

---

# 155. Relationship Query

```php
final readonly class RelationshipQuery
{
    public function __construct(
        public RelationshipNode $source,
        public ?string $relation = null,
        public ?RelationshipNode $target = null,
        public int $maxDepth = 1,
    ) {}
}
```

---

# 156. Specialized Queries

Conviene evitar un graph query language arbitrario dentro del Core V1.

---

# 157. Why

Un lenguaje demasiado genérico introduce:

```text
complexity
query injection risk
unbounded traversal
poor predictability
```

---

# 158. V1 Relationship Requirements

Usar estructuras compilables.

Ejemplo:

```text
ANY OF:
    owns(subject)
    editor_of(subject)
    manager_of(parent(subject))
```

---

# 159. RelationshipRequirement

```php
interface RelationshipRequirementInterface
{
}
```

---

# 160. Composite Requirements

```text
AnyOf
AllOf
Direct
ViaParent
ViaMembership
```

---

# 161. Example

```php
AnyOfRelationshipRequirement(
    new DirectRelationship('owner'),
    new DirectRelationship('editor'),
    new ViaParentRelationship('manager')
);
```

---

# 162. Compilation

Metadata declarativa podrá convertirse en:

```text
RelationshipAccessPlan
```

---

# 163. RelationshipAccessPlan

Contendrá:

```text
starting node
target subject
allowed paths
depth
inheritance
conditions
```

---

# 164. Authorization Planner

Integración:

```text
AuthorizationRequest
        ↓
AuthorizationPlanner
        ↓
RelationshipAccessPlan
        ↓
RelationshipResolver
        ↓
RelationshipDecision
```

---

# 165. Planner Optimization

Si una Policy no requiere relaciones:

```text
no relationship query
```

---

# 166. Short Circuit

Si TenantIsolation ya produce:

```text
DENY
```

no consultar graph backend innecesariamente.

---

# 167. Ownership Fast Path

Ownership directo podrá evaluarse antes que traversal complejo.

---

# 168. Example

```text
subject.owner_id === principal.id
```

puede resolver:

```text
owner
```

sin graph query.

---

# 169. Share Fast Path

Shares directos también pueden consultarse mediante índice específico.

---

# 170. Graph Fallback

Solo después:

```text
indirect relationship traversal
```

---

# 171. Proposed Resolution Order

```text
Direct Ownership
      ↓
Direct Share
      ↓
Direct Relationship
      ↓
Inherited Share
      ↓
Relationship Graph
```

Esto es una optimización, no necesariamente precedence semántica.

---

# 172. Decision Normalization

Todos deberán producir:

```text
GRANT
DENY
ABSTAIN
FAILURE
```

según el modelo general del Authorization System.

---

# 173. Ownership Result

Si no es owner:

```text
ABSTAIN
```

normalmente.

No:

```text
DENY
```

si otras relaciones podrían conceder acceso.

---

# 174. Share Result

No share:

```text
ABSTAIN
```

---

# 175. Relationship Result

No matching relationship:

```text
ABSTAIN
```

salvo que la regla específica requiera obligatoriamente dicha relación.

---

# 176. Mandatory Relationship

Ejemplo:

```text
invoice.approve
requires manager_of(invoice.department)
```

Si no existe:

```text
DENY
```

---

# 177. Requirement Semantics

Por eso distinguir:

```text
authority provider
```

de:

```text
mandatory constraint
```

---

# 178. Ownership as Authority

Ejemplo:

```text
owner may update
```

Ownership produce authority.

---

# 179. Ownership as Constraint

Ejemplo:

```text
only owner may transfer ownership
```

Entonces ownership es mandatory requirement.

---

# 180. Sharing as Authority

Normalmente:

```text
share
→ positive authority
```

---

# 181. Relationship as Authority

Ejemplo:

```text
workspace manager
→ document.manage
```

---

# 182. Relationship as Constraint

Ejemplo:

```text
invoice approval requires manager_of department
```

aunque usuario tenga permission global.

---

# 183. Policy DSL

En futuro podría expresarse:

```text
ALLOW IF
    permission("document.update")
AND
    ANY(
        owns(subject),
        shared_as("editor", subject),
        relation("manager_of", parent(subject))
    )
```

---

# 184. V1 Recommendation

Mantener DSL interna y compilada.

No necesariamente exponer un lenguaje textual público.

---

# 185. Policy Integration

Ejemplo:

```php
final class DocumentPolicy
{
    public function update(
        AuthorizationContext $context,
        Document $document
    ): AuthorizationDecision {
        // domain-specific restrictions
    }
}
```

La Policy puede consultar:

```text
context.relationships()
context.ownership()
context.shares()
```

---

# 186. Prefer Precomputed Context

Evitar que cada Policy dispare consultas independientes.

---

# 187. AuthorizationRelationshipContext

Podrá contener:

```php
final readonly class AuthorizationRelationshipContext
{
    public function __construct(
        public OwnershipResolution $ownership,
        public ShareResolutionSet $shares,
        public RelationshipResolutionSet $relationships,
    ) {}
}
```

---

# 188. Lazy Resolution

No todo debe calcularse siempre.

Podrá resolverse bajo demanda con request memoization.

---

# 189. Memoization Key

Debe considerar:

```text
principal
tenant
subject
relationship requirement
authorization revision
```

---

# 190. Cache

Relationship traversal puede ser costoso.

---

# 191. Cache Layers

Podrán existir:

```text
request memoization
short-lived relationship cache
compiled relationship-plan cache
membership projection cache
```

---

# 192. Cache Key

Ejemplo conceptual:

```text
tenant:7
principal:user:42
subject:document:100
relationship-plan:v5
relationship-version:981
```

---

# 193. Revocation Problem

Si User sale de Team#8:

```text
cached relationship access
```

no deberá sobrevivir más de lo permitido.

---

# 194. Relationship Versioning

Podrá mantenerse:

```text
principal_relationship_version
team_membership_version
resource_share_version
```

---

# 195. Coarse Version

Inicialmente puede utilizarse:

```text
tenant_authorization_version
```

si simplicidad es prioritaria.

---

# 196. Tradeoff

Coarse invalidation:

```text
simpler
more cache misses
```

Fine-grained invalidation:

```text
better performance
more complexity
```

---

# 197. V1 Recommendation

Usar:

```text
principal version
resource/share version
tenant version
```

cuando sea práctico.

---

# 198. Bulk Authorization

Crítico para:

```text
document lists
search results
dashboards
```

---

# 199. Problem

```php
foreach ($documents as $document) {
    Authorization::allows('document.view', $document);
}
```

puede generar cientos de relationship queries.

---

# 200. BulkAuthorization

El sistema deberá poder ejecutar:

```text
Principal
+
Ability
+
Subject Collection
```

---

# 201. Bulk Relationship Resolution

```text
find all Documents among [1..100]
accessible by User#42
```

---

# 202. Query-Level Authorization

Para algunos casos:

```text
authorized scope
```

puede traducirse a filtros SQL.

---

# 203. Example

```text
documents
WHERE
owner_id = user
OR
shared_with_user
OR
shared_with_team
```

---

# 204. Important

Query filtering y per-resource authorization deben compartir semántica.

---

# 205. No Security Drift

No debe ocurrir:

```text
list query says accessible
but Policy says forbidden
```

sin una razón explícita.

---

# 206. AuthorizedResourceScope

Contrato conceptual:

```php
interface AuthorizedResourceScopeInterface
{
    public function apply(
        mixed $query,
        PrincipalInterface $principal,
        Ability $ability,
        AuthorizationContext $context
    ): mixed;
}
```

---

# 207. Scope Optimization

Puede filtrar candidatos.

---

# 208. Final Authorization

Para operaciones sensibles todavía puede ejecutarse Policy individual.

---

# 209. Search Integration

Search results deberán respetar:

```text
authorized resource scope
```

antes de mostrarse.

---

# 210. Search Index Security

No confiar solo en:

```text
frontend filtering
```

---

# 211. Search Documents

Podría indexarse metadata como:

```text
tenant
owner
workspace
visibility
```

para prefiltrar.

---

# 212. Dynamic Relationships

Para relaciones muy dinámicas:

```text
final authorization
```

debe ejecutarse contra fuente suficientemente fresca.

---

# 213. Public Resources

Un recurso puede declarar:

```text
visibility=public
```

---

# 214. Public Is a Relationship/Visibility State

No debe equivaler automáticamente a:

```text
all abilities
```

---

# 215. Public Ability Mapping

Ejemplo:

```text
public
→ document.view
```

pero no:

```text
document.update
document.delete
document.share
```

---

# 216. Visibility

Podrá modelarse:

```php
enum ResourceVisibility: string
{
    case Private = 'private';
    case Restricted = 'restricted';
    case Tenant = 'tenant';
    case Organization = 'organization';
    case Public = 'public';
}
```

---

# 217. Private

Solo:

```text
owner
explicit shares
authorized relationships
```

---

# 218. Tenant Visibility

Puede permitir:

```text
all authorized tenant members
```

pero no necesariamente todos los Principals técnicos.

---

# 219. Organization Visibility

Depende de:

```text
organization membership
```

---

# 220. Public

Puede permitir:

```text
AnonymousPrincipal
```

para abilities concretas.

---

# 221. VisibilityEvaluator

Podrá funcionar como authority provider.

---

# 222. Visibility Does Not Override Deny

Un recurso público bajo:

```text
legal hold
disabled
removed
```

puede seguir siendo inaccesible.

---

# 223. Link Sharing

Debe integrarse con el Capability System del documento 20.

---

# 224. Example

```text
Document#100
share mode:
link
```

deberá emitir:

```text
Capability
```

---

# 225. No Duplicate Security Model

No crear un sistema paralelo:

```text
ShareLinkAuth
```

---

# 226. Unified Flow

```text
Share Link
    ↓
Capability
    ↓
Authorization Authority
    ↓
Document Policy
```

---

# 227. Link Access Level

Capability podrá mapear:

```text
viewer
commenter
```

a abilities concretas.

---

# 228. Password-Protected Links

Password verification pertenece principalmente al security/authentication mechanism de la share capability.

Authorization recibe:

```text
validated capability
```

---

# 229. Link Expiration

Debe ser capability expiry.

---

# 230. Link Revocation

Capability revocation.

---

# 231. Link Usage Limits

Capability consumption/use-count system.

---

# 232. Share Management

Abilities separadas:

```text
resource.share
resource.unshare
resource.view_shares
resource.manage_shares
```

---

# 233. Sharing Does Not Follow Update

Un editor no necesariamente puede compartir.

---

# 234. Example

```text
editor:
    view
    comment
    update

manager:
    view
    comment
    update
    share
    manage_shares
```

---

# 235. Resharing

Debe definirse explícitamente.

---

# 236. Share Delegation

Un usuario con acceso compartido no deberá poder compartirlo nuevamente por default.

---

# 237. Share Metadata

Puede incluir:

```text
reshare_allowed=false
```

---

# 238. Authority Narrowing

Si reshare está permitido:

```text
child access level
<=
parent access level
```

---

# 239. Example

User tiene:

```text
editor
```

Puede compartir:

```text
viewer
```

si se permite.

No:

```text
manager
```

---

# 240. Share Provenance Chain

```text
Owner
→ User A editor
→ User B viewer
```

si resharing existe.

---

# 241. Recommendation

V1:

```text
only resource owner/manager can create shares
```

Simplifica considerablemente seguridad.

---

# 242. Later

Delegated resharing puede implementarse mediante el Delegation System.

---

# 243. Ownership vs Creator

No asumir:

```text
created_by == owner
```

---

# 244. Example

Employee crea Document dentro de Company Workspace.

```text
created_by:
User#42

owner:
Workspace#5
```

---

# 245. CreatedBy Relationship

Puede seguir siendo útil para:

```text
audit
business rules
```

pero no necesariamente autoridad.

---

# 246. Assignment

`assigned_to` tampoco implica ownership.

---

# 247. Example

Ticket:

```text
owner:
Organization

assigned_to:
Agent#10
```

Agent puede:

```text
ticket.handle
```

por relationship, no ownership.

---

# 248. Custodianship

Algunos dominios pueden requerir:

```text
custodian_of
```

diferente de owner.

---

# 249. Extensible Relationships

VoltStack no deberá hardcodear únicamente:

```text
owner
editor
viewer
```

---

# 250. Domain Relations

Podrán existir:

```text
approver_of
reviewer_of
guardian_of
supervisor_of
accountant_for
maintainer_of
subscriber_of
```

---

# 251. ReBAC Enables Domain Language

Esto permite Policies más expresivas.

---

# 252. Example

```text
User
accountant_for
Company
```

Company:

```text
owns
Invoice
```

Ability:

```text
invoice.view
```

puede derivarse mediante path permitido.

---

# 253. Path Definition

```text
accountant_for
→ owns
```

---

# 254. Arbitrary Path Traversal Forbidden

Solo paths registrados.

---

# 255. Relationship Path Registry

```text
invoice.view:
    accountant_for → owns

document.update:
    member_of → owns

ticket.manage:
    supervisor_of → assigned_to
```

---

# 256. Compiled Paths

Deben compilarse para performance.

---

# 257. Reverse Relationships

El sistema podrá inferir o registrar:

```text
owns
↔
owned_by
```

---

# 258. Recommendation

No inferir inversas automáticamente salvo definición explícita.

---

# 259. Why

No todas las relaciones tienen semántica inversa útil.

---

# 260. Cardinality

Relationship definitions podrán declarar:

```text
one-to-one
one-to-many
many-to-many
```

para validación y optimización.

---

# 261. Ownership Cardinality

Ejemplo:

```text
Document
has one owner
```

---

# 262. Membership

```text
User
member_of
many Teams
```

---

# 263. Relationship Mutation

Crear o eliminar relaciones sensibles debe autorizarse.

---

# 264. Example

```text
team.member.add
team.member.remove
project.editor.assign
```

---

# 265. No Direct Repository Writes

Código de aplicación no debería modificar relaciones sensibles saltándose:

```text
RelationshipManager
```

---

# 266. RelationshipManager

```php
interface RelationshipManagerInterface
{
    public function attach(
        AuthorizationRelationship $relationship,
        AuthorizationContext $context
    ): void;

    public function detach(
        AuthorizationRelationship $relationship,
        AuthorizationContext $context
    ): void;
}
```

---

# 267. Mutation Policy

Antes de attach:

```text
authorize relationship.create
```

o ability específica.

---

# 268. Example

Manager añade User#42 a Team#8.

```text
team.member.add
```

---

# 269. Relationship Change Audit

Debe registrar:

```text
actor
relationship
source
target
previous state
new state
```

---

# 270. Share Audit

Registrar:

```text
resource
target
access level
granted by
expiry
```

---

# 271. Ownership Transfer Audit

Especialmente importante:

```text
old owner
new owner
actor
reason
```

---

# 272. Relationship Decision Audit

No toda lectura necesita registrar graph path completo.

---

# 273. Risk-Based Audit

Para operaciones sensibles sí podrá registrar:

```text
authorization path
```

---

# 274. Example

```text
GRANT invoice.approve

because:

User#42
manager_of Department#5
Department#5
owns Invoice#99
```

---

# 275. Explainability

AuthorizationExplanation podrá incluir:

```text
authority_source=relationship
relationship_path=...
```

---

# 276. End User Explanation

Podrá mostrarse:

```text
You have access because this document
was shared with your team.
```

---

# 277. Operator Explanation

Más detallada:

```text
User#42
member_of Team#8
Team#8
editor_of Document#100
```

---

# 278. Security

No revelar relaciones sensibles a usuarios no autorizados.

---

# 279. Example

No decir:

```text
Document belongs to secret Team#88
```

a un atacante.

---

# 280. Concealment

Public denial puede seguir siendo:

```text
resource_not_found
```

---

# 281. Error Codes

Internamente:

```text
ownership.required
ownership.mismatch
ownership.transfer_forbidden

share.missing
share.expired
share.revoked
share.access_insufficient
share.reshare_forbidden

relationship.missing
relationship.expired
relationship.path_invalid
relationship.depth_exceeded
relationship.cycle_detected

visibility.restricted
inheritance.blocked
```

---

# 282. Graph Backend Failure

Si una relación es necesaria para decidir y el backend falla:

```text
FAILURE
```

---

# 283. Fail Closed

Nunca:

```text
graph unavailable
→ assume access
```

---

# 284. Optional Authority Provider Failure

Si un provider opcional falla, la estrategia debe estar definida.

Para seguridad:

```text
uncertain authority
→ no GRANT
```

---

# 285. Relationship Consistency

El sistema deberá definir consistency requirements.

---

# 286. Sensitive Operations

Para:

```text
delete
approve
transfer ownership
grant access
```

usar relaciones suficientemente frescas.

---

# 287. Low-Risk Reads

Podrán tolerar cache corto si configuración lo permite.

---

# 288. External Relationship Providers

Puede integrarse:

```text
LDAP
Active Directory
enterprise directory
external graph
IAM provider
```

---

# 289. Provider Interface

```php
interface RelationshipProviderInterface
{
    public function resolve(
        RelationshipQuery $query,
        AuthorizationContext $context
    ): RelationshipResultSet;
}
```

---

# 290. Multiple Providers

Ejemplo:

```text
LocalDatabaseProvider
DirectoryProvider
OrganizationProvider
```

---

# 291. Provider Priority

Deberá ser determinista.

---

# 292. Provider Aggregation

Puede combinar resultados.

---

# 293. Provider Deny

Recomendación:

Providers de relaciones normalmente producen:

```text
relationship exists / does not exist
```

No decisiones globales.

---

# 294. Decision Remains Centralized

```text
Relationship Data
    ↓
Authorization Evaluator
    ↓
Decision Manager
```

---

# 295. External Provider Failure

No confundir:

```text
no relationship
```

con:

```text
provider unavailable
```

---

# 296. Failure Must Remain Failure

Para evitar false denials difíciles de diagnosticar y, sobre todo, unsafe fallbacks.

---

# 297. Synchronization

Relaciones externas podrán sincronizarse a projections locales.

---

# 298. Example

Directory groups:

```text
LDAP
→ synchronized group membership
→ local Relationship Store
```

---

# 299. Benefits

```text
performance
availability
bulk queries
cacheability
```

---

# 300. Tradeoff

```text
eventual consistency
```

---

# 301. Revocation SLA

Cada provider deberá poder declarar:

```text
maximum authorization staleness
```

---

# 302. Example

```text
directory membership:
max stale 5 min
```

---

# 303. Critical Membership

Puede requerir live validation.

---

# 304. Data Isolation

Relationship queries siempre deberán considerar TenantContext.

---

# 305. Query Requirement

Nunca:

```sql
SELECT *
FROM authorization_relationships
WHERE source_id = ?
```

sin type/tenant constraints cuando correspondan.

---

# 306. Correct Concept

```text
tenant
source type
source id
relation
target
```

---

# 307. Cross-Tenant Attack

User#42 Tenant#7 no debe aprovechar:

```text
User#42
```

de Tenant#9 si IDs locales se repiten.

---

# 308. Globally Unique Principal IDs

Pueden simplificar, pero Tenant validation sigue siendo necesaria.

---

# 309. Resource Deletion

Al eliminar resource:

```text
shares
relationships
cache entries
```

deben invalidarse.

---

# 310. Soft Delete

Soft-deleted resources normalmente:

```text
DENY normal access
```

aunque relationship siga existiendo.

---

# 311. Restore

Al restaurar, la aplicación debe decidir si:

```text
old shares restore
```

o:

```text
shares remain revoked
```

---

# 312. Recommended Sensitive Default

No reactivar automáticamente grants revocados explícitamente.

---

# 313. Resource Move

Mover:

```text
Document
from Folder A
to Folder B
```

puede cambiar inherited access.

---

# 314. Authorization Impact

Move operation deberá evaluar:

```text
source hierarchy
destination hierarchy
ownership
share inheritance
```

---

# 315. Resource Move Ability

```text
resource.move
```

debe estar separada.

---

# 316. Security Check

Mover un recurso no debe convertirse en forma de:

```text
privilege escalation
```

---

# 317. Example Attack

User puede mover documento privado a Workspace donde tiene manager rights.

Después obtiene acceso ampliado.

---

# 318. Move Policy

Debe validar:

```text
may move resource
AND
may place resource in destination
AND
resulting ownership/access is valid
```

---

# 319. Copy Operation

Copy crea un nuevo resource.

---

# 320. Shares on Copy

No deberían copiarse automáticamente por default.

---

# 321. Ownership on Copy

Debe definirse por dominio.

---

# 322. Example

Copy Document:

```text
creator may become owner
```

o:

```text
destination workspace becomes owner
```

---

# 323. Forking

Fork puede conservar:

```text
source attribution
```

sin conservar authorization relationships.

---

# 324. Security Principle

```text
data lineage
≠
authorization lineage
```

---

# 325. Collections

Folders/projects pueden contener miles de recursos.

---

# 326. Avoid Materializing Every Inherited Share

No crear necesariamente:

```text
1 million child ACL rows
```

cuando share de Folder puede resolverse por hierarchy.

---

# 327. But

Para alta escala puede ser útil crear:

```text
materialized authorization projections
```

---

# 328. Projection System

Podrá mantener:

```text
effective access edges
```

precalculados.

---

# 329. Source of Truth

Debe distinguirse:

```text
canonical relationships
```

de:

```text
derived projections
```

---

# 330. Projection Rebuild

Debe ser posible reconstruir projections desde source of truth.

---

# 331. Event-Driven Projection

Ejemplo:

```text
TeamMembershipChanged
    ↓
RelationshipProjectionUpdater
```

---

# 332. Cache/Projection Failure

Nunca perder canonical authorization data.

---

# 333. Eventual Projection

Para critical authorization puede requerirse fallback al source of truth.

---

# 334. Performance Model

Resolver acceso idealmente mediante:

```text
O(1)
```

para direct ownership/share.

---

# 335. Graph Traversal

Debe permanecer:

```text
bounded
```

---

# 336. Avoid General Graph Search

Authorization paths deberían estar predefinidos.

---

# 337. Database Query Compilation

RelationshipAccessPlan podrá compilarse a:

```text
SQL joins
EXISTS queries
recursive CTE
provider-specific query
```

---

# 338. Recursive CTE

Puede utilizarse para hierarchy limitada cuando DB lo soporte.

---

# 339. Dialect Integration

Database subsystem deberá manejar diferencias entre:

```text
PostgreSQL
MySQL
MariaDB
SQLite
```

cuando sea necesario.

---

# 340. Authorization Core Independence

Core no deberá generar SQL directamente.

---

# 341. Relationship Query Compiler

Backend SQL específico:

```text
SqlRelationshipStore
```

podrá hacerlo.

---

# 342. Observability

Registrar métricas como:

```text
relationship_resolution_duration
relationship_depth
relationship_cache_hit
share_resolution_count
ownership_resolution_duration
```

---

# 343. Security Metrics

También:

```text
cross_tenant_relationship_denials
relationship_cycle_detected
expired_share_attempts
unauthorized_share_attempts
ownership_transfer_denials
```

---

# 344. Tracing

Span conceptual:

```text
authorization.relationship.resolve
```

---

# 345. Attributes

```text
subject_type
ability
provider
depth
cache_hit
result
```

sin exponer secretos.

---

# 346. Cardinality Protection

No poner raw:

```text
user IDs
document IDs
```

como metric labels de alta cardinalidad.

---

# 347. Logging

Debug autorizado podrá registrar IDs.

Metrics no.

---

# 348. Testing Strategy

El subsistema requerirá pruebas unitarias, integración, seguridad y performance.

---

# 349. Ownership Tests

Cubrir:

```text
direct owner
wrong owner
organization owner
workspace owner
inherited owner
ownership transfer
concurrent transfer
```

---

# 350. Sharing Tests

```text
direct user share
team share
group share
expired share
revoked share
insufficient access level
```

---

# 351. Relationship Tests

```text
direct relationship
indirect relationship
transitive relationship
non-transitive relationship
invalid path
cycle
depth limit
```

---

# 352. Hierarchy Tests

```text
parent access
descendant inheritance
inheritance disabled
confidential boundary
resource moved
```

---

# 353. Multi-Tenant Tests

```text
same ID different tenants
cross-tenant share
cross-tenant relationship
tenant-qualified ownership
```

---

# 354. Public Visibility Tests

```text
anonymous view
anonymous update denied
public resource disabled
public resource tenant leakage
```

---

# 355. Capability Sharing Tests

```text
valid share link
expired
revoked
wrong resource
wrong tenant
wrong audience
```

---

# 356. Bulk Authorization Tests

Verificar:

```text
same results
```

entre:

```text
single-resource authorization
```

y:

```text
bulk authorization
```

---

# 357. Query Scope Tests

`AuthorizedResourceScope` no deberá retornar recursos que el sistema individual denegaría bajo la semántica cubierta por ese scope.

---

# 358. Revocation Tests

Eliminar User#42 de Team#8.

El acceso derivado deberá desaparecer según consistency SLA.

---

# 359. Cache Tests

Cambiar:

```text
share version
membership version
ownership version
```

debe invalidar decisions relevantes.

---

# 360. Graph Failure Test

Backend no disponible:

```text
FAILURE
```

no GRANT.

---

# 361. Cycle Security Test

Crear:

```text
A → B
B → C
C → A
```

Resolver debe terminar de forma segura.

---

# 362. Depth Attack Test

Crear cadena extremadamente profunda.

Resolver:

```text
max_depth exceeded
```

sin consumo ilimitado.

---

# 363. Relationship Injection Test

Un usuario no podrá crear relaciones arbitrarias como:

```text
User#42
owner_of
Tenant#1
```

sin pasar por mutation authorization y schema validation.

---

# 364. Reshare Escalation Test

User recibe:

```text
viewer
```

intenta compartir:

```text
manager
```

Resultado:

```text
DENY
```

---

# 365. Move Escalation Test

Mover resource a container con permisos más amplios deberá seguir las reglas explícitas del dominio.

---

# 366. Property-Based Invariant

```text
Removing an authorization relationship
must never increase access.
```

---

# 367. Property

```text
Increasing graph traversal depth
must not discover paths
that are not explicitly permitted
by the relationship plan.
```

---

# 368. Property

```text
A share cannot grant abilities
outside its access-level definition.
```

---

# 369. Property

```text
Cross-tenant relationship paths
must never be traversed unless explicitly allowed.
```

---

# 370. Property

```text
An inherited relationship cannot cross
an inheritance boundary.
```

---

# 371. Directory Structure propuesta

```text
Quantum/
└── Authorization/
    └── Relationships/
        ├── Ownership/
        │   ├── OwnershipResolverInterface.php
        │   ├── OwnershipResolution.php
        │   ├── OwnershipType.php
        │   ├── OwnershipRegistry.php
        │   ├── OwnershipEvaluator.php
        │   ├── OwnershipTransferManager.php
        │   └── OwnershipTransferPolicy.php
        │
        ├── Sharing/
        │   ├── ResourceShare.php
        │   ├── ResourceShareStatus.php
        │   ├── ShareTarget.php
        │   ├── ShareTargetType.php
        │   ├── ShareInheritanceMode.php
        │   ├── ResourceShareRepositoryInterface.php
        │   ├── ResourceShareManager.php
        │   ├── ResourceShareEvaluator.php
        │   └── ResourceAccessRegistry.php
        │
        ├── Graph/
        │   ├── AuthorizationRelationship.php
        │   ├── RelationshipNode.php
        │   ├── RelationshipDefinition.php
        │   ├── RelationshipPath.php
        │   ├── RelationshipStatus.php
        │   ├── RelationshipSource.php
        │   ├── RelationshipRegistry.php
        │   ├── RelationshipQuery.php
        │   ├── RelationshipResultSet.php
        │   ├── RelationshipStoreInterface.php
        │   ├── RelationshipProviderInterface.php
        │   └── RelationshipManager.php
        │
        ├── Requirements/
        │   ├── RelationshipRequirementInterface.php
        │   ├── DirectRelationshipRequirement.php
        │   ├── AnyOfRelationshipRequirement.php
        │   ├── AllOfRelationshipRequirement.php
        │   ├── ViaParentRelationshipRequirement.php
        │   └── ViaMembershipRelationshipRequirement.php
        │
        ├── Hierarchy/
        │   ├── ResourceHierarchyResolverInterface.php
        │   ├── AuthorizationInheritancePolicyInterface.php
        │   ├── AuthorizationInheritanceBoundary.php
        │   └── HierarchicalRelationshipResolver.php
        │
        ├── Planning/
        │   ├── RelationshipAccessPlan.php
        │   ├── RelationshipAccessPlanner.php
        │   └── RelationshipPlanCompiler.php
        │
        ├── Evaluation/
        │   ├── RelationshipAccessEvaluator.php
        │   ├── VisibilityEvaluator.php
        │   └── RelationshipDecisionNormalizer.php
        │
        ├── Context/
        │   └── AuthorizationRelationshipContext.php
        │
        ├── Visibility/
        │   ├── ResourceVisibility.php
        │   └── ResourceVisibilityResolver.php
        │
        ├── Bulk/
        │   ├── BulkRelationshipResolver.php
        │   └── AuthorizedResourceScopeInterface.php
        │
        ├── Cache/
        │   ├── RelationshipCache.php
        │   ├── RelationshipVersionResolver.php
        │   └── RelationshipMemoizer.php
        │
        └── Exceptions/
            ├── OwnershipException.php
            ├── ShareException.php
            ├── RelationshipException.php
            ├── RelationshipCycleException.php
            ├── RelationshipDepthException.php
            └── RelationshipProviderException.php
```

---

# 372. Ownership Invariants

### Invariante 1

Ownership siempre pertenece a un recurso concreto.

### Invariante 2

Ownership no equivale a una permission global.

### Invariante 3

Creator y Owner son conceptos independientes.

### Invariante 4

Transferir ownership requiere autorización explícita.

### Invariante 5

Tenant ownership no implica acceso universal dentro del Tenant.

---

# 373. Sharing Invariants

### Invariante 1

Un Share es una fuente limitada de autoridad.

### Invariante 2

Un Share no evita Resource Policies.

### Invariante 3

Un Share expirado o revocado no concede acceso.

### Invariante 4

Resharing nunca puede ampliar authority.

### Invariante 5

Ownership y shared manager access permanecen separados.

---

# 374. Relationship Invariants

### Invariante 1

Solo relaciones registradas pueden participar en authorization.

### Invariante 2

Solo paths registrados pueden recorrerse.

### Invariante 3

Traversal siempre es bounded.

### Invariante 4

Cycles nunca producen loops infinitos.

### Invariante 5

Cross-tenant traversal está prohibido por default.

---

# 375. Hierarchy Invariants

### Invariante 1

Inheritance debe ser explícita.

### Invariante 2

Inheritance boundaries detienen propagación.

### Invariante 3

Mover recursos puede cambiar authorization y debe evaluarse.

### Invariante 4

Copy no copia automáticamente shares.

### Invariante 5

Data lineage no implica authorization lineage.

---

# 376. Performance Invariants

### Invariante 1

Direct ownership debe disponer de fast path.

### Invariante 2

Direct shares deben resolverse sin graph traversal innecesario.

### Invariante 3

Bulk authorization evita N+1.

### Invariante 4

Relationship plans son compilables/cacheables.

### Invariante 5

Caches respetan authorization revisions.

---

# 377. Security Invariants

### Invariante 1

Un relationship provider unavailable nunca produce GRANT por fallback.

### Invariante 2

Un usuario no puede crear relationships sensibles directamente.

### Invariante 3

Resource sharing requiere autorización.

### Invariante 4

Tenant isolation se ejecuta independientemente de ownership/sharing.

### Invariante 5

Resource Policies pueden negar incluso cuando existe una relación válida.

---

# 378. Arquitectura general

```text
                    AUTHORIZATION REQUEST
                            │
                            ↓
                    Tenant Isolation
                            │
                            ↓
                   Permission / Ability
                            │
                            ↓
               Resource Relationship Layer
                            │
          ┌─────────────────┼─────────────────┐
          ↓                 ↓                 ↓
      Ownership           Shares        Relationships
          │                 │                 │
          ↓                 ↓                 ↓
   Direct Owner      User / Team /      ReBAC Graph
   Org / Team        Group / Org         Resolution
          │                 │                 │
          └─────────────────┼─────────────────┘
                            ↓
                  Effective Authority
                            │
                            ↓
                    Resource Policy
                            │
                            ↓
              Non-Bypassable Evaluators
                            │
                            ↓
                         DECISION
```

---

# 379. Relationship Resolution Architecture

```text
Principal
   │
   ↓
RelationshipAccessPlan
   │
   ├── Direct Ownership Fast Path
   │
   ├── Direct Share Fast Path
   │
   ├── Membership Resolution
   │
   ├── Parent Resource Resolution
   │
   └── Bounded Graph Traversal
              │
              ↓
       Relationship Path
              │
              ↓
       Requirement Evaluation
              │
              ↓
        GRANT / ABSTAIN /
        DENY / FAILURE
```

---

# 380. Example — Document Ownership

```text
User#42
owns
Document#100
```

Request:

```text
document.update
```

Pipeline:

```text
TenantIsolation
→ GRANT

Permission
→ GRANT

OwnershipEvaluator
→ GRANT

DocumentPolicy
→ GRANT

Final
→ GRANT
```

---

# 381. Example — Direct Share

```text
Document#100
shared with User#57
as editor
```

User#57 solicita:

```text
document.update
```

Resolution:

```text
Ownership
→ ABSTAIN

ResourceShare
→ editor
→ document.update
→ GRANT

DocumentPolicy
→ GRANT
```

Final:

```text
GRANT
```

---

# 382. Example — Insufficient Share

User#57 tiene:

```text
viewer
```

Solicita:

```text
document.update
```

ShareEvaluator:

```text
viewer
does not contain
document.update
```

Resultado:

```text
ABSTAIN
```

Si no existe otra authority:

```text
DENY
```

---

# 383. Example — Team Relationship

```text
User#42
member_of
Team#8

Team#8
editor_of
Document#100
```

Allowed path:

```text
member_of
→ editor_of
```

Resultado:

```text
GRANT document.update
```

---

# 384. Example — Workspace Inheritance

```text
User#42
manager_of
Workspace#5

Workspace#5
contains
Project#10

Project#10
contains
Document#100
```

Rule:

```text
workspace manager
may manage descendants
max_depth=3
```

Entonces:

```text
document.manage
→ GRANT
```

---

# 385. Example — Inheritance Boundary

```text
Workspace#5
    ↓
Project#10
    ↓
ConfidentialFolder#7
    ↓
Document#100
```

Aunque User sea:

```text
Workspace Manager
```

si:

```text
ConfidentialFolder
inherit_access=false
```

entonces:

```text
relationship path blocked
```

---

# 386. Example — Resource Policy Override

User#42 es owner.

Tiene:

```text
document.update
```

Pero:

```text
Document#100
status=legal_hold
```

Ownership:

```text
GRANT
```

Policy:

```text
DENY
document.legal_hold
```

Final:

```text
DENY
```

---

# 387. Example — Tenant Boundary

```text
User#42
Tenant#7

Document#100
Tenant#9
```

Aunque exista accidentalmente:

```text
User#42
editor_of
Document#100
```

TenantIsolation:

```text
DENY
```

Graph traversal no deberá convertirlo en GRANT.

---

# 388. Example — Sharing Link

Owner genera:

```text
viewer link
```

VoltStack crea:

```text
Capability#900

ability:
document.view

subject:
Document#100

expires:
24h
```

AnonymousPrincipal presenta capability.

Pipeline:

```text
Capability
→ valid

Tenant Binding
→ valid

Document Policy
→ GRANT

Final
→ GRANT
```

---

# 389. Example — Folder Share

```text
Folder#10
shared with Team#8
as viewer

inheritance:
descendants
```

Document#100 pertenece a Folder#10.

User#42:

```text
member_of Team#8
```

Resolution:

```text
User#42
→ member_of Team#8
→ shared viewer Folder#10
→ inherited to Document#100
```

Resultado:

```text
document.view
→ GRANT
```

---

# 390. Example — Revocation

User#42 sale de Team#8.

Anteriormente:

```text
User#42
→ Team#8
→ editor Document#100
```

Después:

```text
membership removed
relationship version incremented
cache invalidated
```

Nueva autorización:

```text
ABSTAIN
→ DENY
```

---

# 391. Example — Ownership Transfer

User#42 owns Document#100.

Solicita:

```text
transfer ownership
to User#57
```

VoltStack evalúa:

```text
resource.transfer_ownership
→ GRANT

same tenant
→ GRANT

target eligible
→ GRANT
```

Transaction:

```text
Owner:
42 → 57

ownership_version:
8 → 9

authorization cache:
invalidate

audit:
record transfer
```

---

# 392. Example — Preventing Privilege Escalation

User#42 es editor de Document#100.

Intenta:

```text
share Document#100
with User#57
as manager
```

Si editor no posee:

```text
document.share
```

Resultado:

```text
DENY
```

Aunque pueda editar contenido.

---

# 393. Relationship Authorization Model

VoltStack podrá expresar decisiones de este tipo:

```text
ALLOW document.update

IF

principal has base ability document.update

AND

(
    principal owns document

    OR

    document is shared with principal as editor

    OR

    document is shared with one of principal's teams as editor

    OR

    principal manages document's workspace
)

AND

document is inside principal's authorized tenant

AND

DocumentPolicy does not deny

AND

security constraints do not deny
```

---

# 394. Modelo unificado

El sistema final deberá converger en:

```text
WHO
│
├── Principal
├── Actor
└── Effective Principal

WHAT
│
├── Ability
└── Resource

WHERE
│
└── Tenant

WHY
│
├── Permission
├── Ownership
├── Share
├── Relationship
├── Delegation
└── Capability

UNDER WHICH CONDITIONS
│
├── ABAC
├── Resource State
├── Security Context
├── Inheritance
├── Expiration
└── Policy

            ↓

       AUTHORIZATION
          DECISION
```

---

# 395. Filosofía arquitectónica

El sistema seguirá estos principios:

```text
Ownership is a relationship, not a global role.

Sharing is scoped authority, not ownership.

Membership creates relationships, not automatic universal access.

ReBAC describes how principals relate to resources.

RBAC describes what categories of actions principals may perform.

ABAC constrains those actions using contextual attributes.

Policies express domain-specific authorization rules.

Tenant isolation remains an independent security boundary.

Capabilities represent explicit portable authority.

Delegation represents explicit transferred authority.

No relationship bypasses non-bypassable security rules.
```

---

# 396. Resultado esperado

`21_AUTHORIZATION_RESOURCE_OWNERSHIP_SHARING_AND_RELATIONSHIP_ACCESS_SYSTEM.md` permitirá que VoltStack soporte desde aplicaciones tradicionales hasta plataformas colaborativas complejas:

```text
SaaS multi-tenant
CRMs
ERPs
Document management
Cloud storage
Project management
Collaboration suites
Developer platforms
Financial systems
Enterprise applications
AI platforms
Content management
Team workspaces
```

con modelos como:

```text
User owns Resource

Organization owns Resource

Workspace contains Resource

Team manages Resource

User belongs Team

Resource shared with Team

Manager supervises Department

Department owns Invoice

Agent assigned Ticket

Reviewer reviews Document

Anonymous user accesses shared capability
```

sin convertir el Authorization System en una colección de verificaciones particulares.

La arquitectura definitiva será:

```text
PRINCIPAL
    │
    ↓
BASE AUTHORITY
    │
    ↓
TENANT BOUNDARY
    │
    ↓
RESOURCE
    │
    ├── OWNERSHIP
    │
    ├── SHARES
    │
    ├── MEMBERSHIPS
    │
    ├── HIERARCHY
    │
    └── RELATIONSHIP GRAPH
    │
    ↓
EFFECTIVE RESOURCE AUTHORITY
    │
    ↓
RESOURCE POLICY
    │
    ↓
SECURITY CONSTRAINTS
    │
    ↓
DECISION
```

El principio definitivo será:

> **VoltStack no deberá preguntar únicamente qué permisos tiene un usuario, sino qué relación existe entre el Principal, el recurso, su propietario, su organización, su Tenant y el contexto en el que intenta ejecutar la operación.**

De esta forma, **RBAC + ABAC + ReBAC + Policies + Ownership + Sharing + Delegation + Capabilities** podrán funcionar como partes de un mismo Authorization Engine, manteniendo una semántica común de decisiones, trazabilidad, aislamiento multi-tenant y extensibilidad.