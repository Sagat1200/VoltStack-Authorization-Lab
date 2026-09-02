# VoltStack Authorization System

## Authorization Data Model, Persistence and Storage Boundaries System

**Documento:** `28_AUTHORIZATION_DATA_MODEL_PERSISTENCE_AND_STORAGE_BOUNDARIES_SYSTEM.md`  
**Sistema:** Authorization  
**Framework:** VoltStack  
**Módulo sugerido:** `Quantum/Authorization`  
**Estado:** Especificación arquitectónica  
**Versión objetivo:** 1.x+

---

# 1. Propósito

Este documento define el **modelo de datos canónico del Authorization System de VoltStack**, sus contratos de persistencia, repositorios, proyecciones, índices, límites de almacenamiento y relaciones con otros subsistemas.

El objetivo no es obligar al framework a utilizar una estructura SQL concreta.

El objetivo es definir:

> **qué información pertenece a Authorization, quién es propietario de ella, cómo se representa conceptualmente, qué información puede derivarse, qué información puede cachearse y qué datos pertenecen a otros sistemas.**

El diseño deberá funcionar con:

```text
MySQL
PostgreSQL
MariaDB
SQLite

Redis
Distributed Cache

Graph Database

External IAM

LDAP / Active Directory

SCIM

Policy Services

Custom Authorization Providers
```

sin acoplar el Core a ninguno de ellos.

---

# 2. Principio fundamental

La persistencia de Authorization deberá seguir:

```text
Authorization Domain Model
        ↓
Repository Contracts
        ↓
Persistence Adapters
        ↓
Storage Technology
```

y nunca:

```text
Authorization Core
        ↓
Eloquent/Doctrine/SQL/Redis directly
```

Por tanto:

> **El modelo conceptual de autorización pertenece al Core; la representación física pertenece al adapter de persistencia.**

---

# 3. Qué pertenece al Authorization System

Authorization será propietario conceptual de información como:

```text
Abilities
Roles

Role → Ability assignments

Principal → Role assignments
Principal → Permission grants

Scoped role assignments
Scoped permission grants

Authorization memberships
cuando sean específicamente authorization memberships

Resource shares

Authorization relationships

Delegation grants

Capabilities
cuando sean authority objects

Impersonation authorization state

Approval authorization state

Authorization boundaries

Authorization versions

Revocation state

Authorization policy metadata

Authorization projections
```

---

# 4. Qué NO pertenece necesariamente a Authorization

No deberá duplicar automáticamente:

```text
users
passwords
credentials
MFA secrets
OAuth identities
sessions
business resources
organizations
teams
workspaces
documents
orders
payments
```

si esos objetos ya pertenecen a:

```text
Identity
Authentication
Application Domain
Tenant System
Database Domain
```

---

# 5. Authorization References External Objects

Authorization normalmente almacenará referencias:

```text
principal_type
principal_id

subject_type
subject_id

scope_type
scope_id
```

en lugar de apropiarse de las entidades.

---

# 6. Ejemplo

Authorization puede almacenar:

```text
principal:
    type = user
    id   = 42

role:
    workspace.editor

scope:
    type = workspace
    id   = 91
```

pero no necesita almacenar:

```text
users.name
users.email
workspaces.title
```

---

# 7. Aggregate Conceptual Model

El modelo general será:

```text
                    AUTHORIZATION
                         │
       ┌─────────────────┼─────────────────┐
       │                 │                 │
       ↓                 ↓                 ↓
    Authority         Context          Constraints
       │                                   │
 ┌─────┼──────┐                       ┌─────┼─────┐
 ↓     ↓      ↓                       ↓           ↓
Role Permission Relationship       Scope       Policy
       │      │
       │      ├── Ownership
       │      ├── Sharing
       │      └── ReBAC
       │
       ├── Delegation
       ├── Capability
       └── Approval
```

---

# 8. Canonical vs Derived State

Todo dato deberá clasificarse como:

```php
enum AuthorizationStorageSemantics: string
{
    case Canonical = 'canonical';
    case Derived = 'derived';
    case Projection = 'projection';
    case Cache = 'cache';
    case Ephemeral = 'ephemeral';
    case External = 'external';
}
```

---

# 9. Canonical State

Es la fuente de verdad.

Ejemplos:

```text
role assignment
permission grant
share
delegation
capability record
authorization relationship
```

si VoltStack administra esos objetos.

---

# 10. Derived State

Puede reconstruirse.

Ejemplo:

```text
effective permissions for Principal#42
```

derivadas de:

```text
roles
permissions
scope inheritance
relationships
```

---

# 11. Projection

Representación optimizada.

Ejemplo:

```text
principal_effective_permissions
```

---

# 12. Cache

Acelerador descartable.

Ejemplo:

```text
authorization decision cache
```

---

# 13. Ephemeral State

Existe solo durante:

```text
request
execution
transaction
step-up flow
```

Ejemplo:

```text
AuthorizationContext
DecisionSnapshot
request memoization
```

---

# 14. External State

Fuente gestionada fuera de Authorization.

Ejemplo:

```text
LDAP group membership
external IAM role
SCIM organization membership
device trust
risk score
```

---

# 15. Rule

VoltStack deberá poder responder para cada dato:

```text
Who owns it?
Where is canonical state?
Can it be reconstructed?
Can it be cached?
How is it invalidated?
What version identifies it?
```

---

# 16. Authorization Entity Identifiers

Los objetos de Authorization deberán utilizar identificadores opacos.

```php
final readonly class AuthorizationEntityId
{
    public function __construct(
        public string $value,
    ) {}
}
```

---

# 17. Identifier Strategy

VoltStack no deberá imponer:

```text
AUTO_INCREMENT
UUID
ULID
Snowflake
```

al Core.

Los adapters podrán decidir.

---

# 18. Recommended Public IDs

Para objetos que pueden cruzar servicios:

```text
ULID
UUIDv7
```

son opciones apropiadas.

No son requisito del Core.

---

# 19. PrincipalReference

```php
final readonly class PrincipalReference
{
    public function __construct(
        public string $type,
        public string|int $id,
    ) {}
}
```

---

# 20. SubjectReference

```php
final readonly class SubjectReference
{
    public function __construct(
        public string $type,
        public string|int $id,
    ) {}
}
```

---

# 21. ScopeReference

Ya definido conceptualmente:

```php
final readonly class AuthorizationScopeReference
{
    public function __construct(
        public string $type,
        public string|int $id,
        public string|int|null $tenantId = null,
    ) {}
}
```

---

# 22. Typed References

Nunca depender únicamente de:

```text
id = 42
```

porque:

```text
User#42
Team#42
Service#42
```

son entidades distintas.

---

# 23. Canonical Type Registry

Tipos deberán normalizarse:

```text
user
service
team
organization
workspace
document
```

no usar directamente:

```text
App\Models\User
Modules\Documents\Models\Document
```

como identificador persistente.

---

# 24. Why

Los nombres de clases pueden cambiar.

El identificador lógico deberá permanecer estable.

---

# 25. Authorization Type Registry

```php
interface AuthorizationTypeRegistryInterface
{
    public function resolveAlias(string $type): string;

    public function resolveClass(string $alias): ?string;
}
```

---

# 26. Stable Aliases

Ejemplo:

```php
AuthorizationTypes::register(
    alias: 'document',
    class: Document::class,
);
```

---

# 27. Ability Model

Ability representa:

```text
operation semantic identifier
```

Ejemplos:

```text
document.view
document.update
document.delete

workspace.member.invite

billing.invoice.refund

production.deploy
```

---

# 28. Ability Definition

```php
final readonly class AbilityDefinition
{
    public function __construct(
        public string $name,
        public ?string $subjectType,
        public array $metadata,
        public bool $enabled = true,
    ) {}
}
```

---

# 29. Ability Storage

Abilities pueden ser:

```text
code-defined
database-defined
hybrid
external
```

---

# 30. Code-Defined Abilities

Adecuado para:

```text
application capabilities
framework abilities
```

---

# 31. Database-Defined Abilities

Útil para:

```text
dynamic enterprise modules
administrative customization
```

---

# 32. Hybrid Recommendation

La existencia semántica de abilities críticas puede declararse en código.

Assignments viven dinámicamente en persistence.

---

# 33. Conceptual abilities table

```text
authorization_abilities

id
name
subject_type
metadata
enabled
version
created_at
updated_at
```

Esta tabla será opcional si abilities están completamente compiladas desde código.

---

# 34. Unique Constraint

```text
UNIQUE(name)
```

---

# 35. Role Model

```php
final readonly class AuthorizationRole
{
    public function __construct(
        public string $id,
        public string $name,
        public ?string $scopeType,
        public array $metadata,
        public int|string $version,
    ) {}
}
```

---

# 36. Role Semantics

Un Role será:

```text
named collection / source of authority
```

No deberá convertirse en:

```text
hardcoded application branch
```

como:

```php
if ($user->role === 'admin') {
    return true;
}
```

---

# 37. Conceptual roles table

```text
authorization_roles

id
tenant_id nullable
name
scope_type nullable
description nullable
metadata
version
created_at
updated_at
```

---

# 38. Role uniqueness

Puede ser:

```text
tenant_id + name + scope_type
```

dependiendo del modelo.

---

# 39. Platform Roles

```text
tenant_id = null
```

no significa automáticamente:

```text
global super-admin
```

La semántica depende del Role Definition.

---

# 40. Role Ability Assignment

Conceptualmente:

```text
Role
   ↓
Ability
```

---

# 41. Table

```text
authorization_role_abilities

role_id
ability
effect
conditions nullable
created_at
```

---

# 42. Primary / Unique Key

```text
UNIQUE(role_id, ability)
```

---

# 43. Effect

V1 puede soportar:

```text
ALLOW
```

principalmente.

Si se implementa explicit DENY:

```text
ALLOW
DENY
```

deberá tener semántica formal.

---

# 44. Avoid Nullable Semantics

No usar combinaciones ambiguas como:

```text
allowed = null
```

---

# 45. Principal Role Assignment

```php
final readonly class PrincipalRoleAssignment
{
    public function __construct(
        public string $id,
        public PrincipalReference $principal,
        public string $roleId,
        public ?AuthorizationScopeReference $scope,
        public ?DateTimeImmutable $activeFrom,
        public ?DateTimeImmutable $expiresAt,
        public int|string $version,
    ) {}
}
```

---

# 46. Conceptual table

```text
authorization_principal_roles

id

tenant_id

principal_type
principal_id

role_id

scope_type nullable
scope_id nullable

active_from nullable
expires_at nullable

status

version

assigned_by_type nullable
assigned_by_id nullable

created_at
updated_at
```

---

# 47. Indexes

```text
(principal_type, principal_id)

(role_id)

(tenant_id)

(scope_type, scope_id)

(principal_type, principal_id, scope_type, scope_id)

(expires_at)
```

---

# 48. Assignment Status

```php
enum AuthorizationAssignmentStatus: string
{
    case Active = 'active';
    case Suspended = 'suspended';
    case Revoked = 'revoked';
    case Expired = 'expired';
}
```

---

# 49. Do Not Physically Delete Critical Revocations Immediately

Puede ser útil mantener:

```text
revocation history
version
audit reference
```

según política de retención.

---

# 50. Direct Permission Grant

Permite:

```text
Principal → Ability
```

sin Role.

---

# 51. Conceptual table

```text
authorization_principal_permissions

id
tenant_id

principal_type
principal_id

ability

scope_type nullable
scope_id nullable

effect

conditions nullable

active_from nullable
expires_at nullable

status
version

granted_by_type nullable
granted_by_id nullable

created_at
updated_at
```

---

# 52. Use Direct Permissions Sparingly

Roles suelen ser preferibles para administración.

Direct grants son útiles para:

```text
exceptions
temporary grants
service identities
special authority
```

---

# 53. Role vs Permission Storage

No deberán mezclarse en una sola tabla genérica si eso destruye:

```text
semantic clarity
foreign-key integrity
query planning
audit provenance
```

Un adapter puede optimizar físicamente, pero el Domain Model debe distinguirlos.

---

# 54. Tenant Dimension

Toda entidad tenant-bound deberá incluir:

```text
tenant_id
```

o una referencia equivalente.

---

# 55. Tenant Is Not Just Query Filter

Es una:

```text
security partition
```

---

# 56. Tenant Storage Invariant

Un record tenant-bound nunca deberá relacionarse accidentalmente con entidades de otro Tenant.

---

# 57. Database Constraint

Cuando sea viable:

```text
composite keys
foreign keys
tenant-aware unique constraints
```

pueden reforzar aislamiento.

---

# 58. Example

No basta:

```text
share.resource_id = 42
```

si `42` podría existir por tenant.

Debe conocerse:

```text
tenant_id
resource_type
resource_id
```

---

# 59. Scope Storage

Documento 22 propuso:

```text
authorization_scopes
authorization_scope_memberships
authorization_scope_roles
authorization_scope_permissions
```

Este documento formaliza sus boundaries.

---

# 60. authorization_scopes

Conceptualmente:

```text
id
tenant_id

scope_type
external_scope_id

parent_scope_id nullable

status

hierarchy_version
authorization_version

metadata

created_at
updated_at
```

---

# 61. Important

Puede no ser necesario duplicar todos los Workspaces/Organizations en esta tabla.

---

# 62. Scope Storage Modes

```php
enum AuthorizationScopeStorageMode: string
{
    case Native = 'native';
    case Reference = 'reference';
    case Projection = 'projection';
    case External = 'external';
}
```

---

# 63. Native

Authorization posee el scope.

---

# 64. Reference

Scope pertenece al dominio.

Authorization solo mantiene:

```text
type + id
```

---

# 65. Projection

Mantiene representación optimizada de hierarchy.

---

# 66. External

Hierarchy viene de:

```text
directory
IAM
organization service
```

---

# 67. Default Recommendation

Para VoltStack:

```text
business Organization/Workspace
→ Domain owned

Authorization
→ references + optional hierarchy projection
```

---

# 68. Scope Membership

No toda membership debe duplicarse.

---

# 69. Example

Si `workspace_members` ya es canonical domain table:

Authorization puede consumirla mediante:

```php
WorkspaceMembershipProvider
```

en lugar de crear:

```text
authorization_scope_memberships
```

duplicada.

---

# 70. Rule

> **No duplicar canonical membership únicamente porque Authorization necesita consultarla.**

---

# 71. Authorization-Owned Membership

Crear tabla propia cuando membership existe exclusivamente para authorization.

---

# 72. Membership Provider

```php
interface AuthorizationMembershipProviderInterface
{
    public function membershipsFor(
        PrincipalReference $principal,
        AuthorizationScopeReference $scope,
    ): iterable;
}
```

---

# 73. Resource Ownership

Documento 21:

```text
created_by ≠ owner
```

---

# 74. Ownership Storage

Puede vivir:

```text
inside resource domain table
```

por ejemplo:

```text
documents.owner_id
```

---

# 75. Authorization Must Not Duplicate It Automatically

Utilizar:

```php
OwnershipResolverInterface
```

---

# 76. Authorization-Owned Ownership

Solo si ownership es un concepto independiente administrado por Authorization.

---

# 77. Ownership Relation Table

Opcional:

```text
authorization_relationships
```

con:

```text
relation = owns
```

---

# 78. Resource Sharing Model

Sharing sí puede pertenecer naturalmente a Authorization.

---

# 79. ResourceShare

```php
final readonly class ResourceShare
{
    public function __construct(
        public string $id,
        public SubjectReference $resource,
        public ShareTarget $target,
        public string $accessLevel,
        public ShareInheritanceMode $inheritance,
        public DateTimeImmutable $createdAt,
        public ?DateTimeImmutable $expiresAt,
        public AuthorizationAssignmentStatus $status,
        public int|string $version,
    ) {}
}
```

---

# 80. Conceptual table

```text
authorization_resource_shares

id
tenant_id

resource_type
resource_id

target_type
target_id

access_level
inheritance_mode

conditions nullable

active_from nullable
expires_at nullable

status
version

granted_by_type
granted_by_id

created_at
updated_at
```

---

# 81. Share Indexes

```text
(resource_type, resource_id)

(target_type, target_id)

(tenant_id, target_type, target_id)

(resource_type, resource_id, status)

(expires_at)
```

---

# 82. Public Shares

`target_type` podría ser:

```text
public
```

sin `target_id`.

Pero deberá estar explícitamente modelado.

---

# 83. Link Sharing

No guardar el secret directamente.

---

# 84. Instead

```text
capability identifier
secret hash
```

---

# 85. Relationship Storage

Documento 21 definió:

```text
authorization_relationships
```

---

# 86. Conceptual schema

```text
id

tenant_id

subject_type
subject_id

relation

object_type
object_id

attributes nullable

status

active_from nullable
expires_at nullable

source

version

created_at
updated_at
```

---

# 87. Semantics

Ejemplo:

```text
subject:
    user:42

relation:
    member_of

object:
    team:9
```

---

# 88. Another

```text
team:9
editor_of
document:500
```

---

# 89. Relationship Direction

Debe ser canónica.

No almacenar arbitrariamente:

```text
team contains user
```

y a veces:

```text
user member_of team
```

como equivalentes sin definición.

---

# 90. Relation Definition Registry

```php
final readonly class AuthorizationRelationshipDefinition
{
    public function __construct(
        public string $name,
        public string $subjectType,
        public string $objectType,
        public bool $transitive,
    ) {}
}
```

---

# 91. Relationship Unique Constraint

Dependerá de relation cardinality.

Ejemplo:

```text
UNIQUE(
    tenant_id,
    subject_type,
    subject_id,
    relation,
    object_type,
    object_id
)
```

para relaciones no duplicables.

---

# 92. Graph Backend

La misma abstracción puede almacenarse en:

```text
Neo4j
JanusGraph
SQL
external Zanzibar-like service
```

sin cambiar Policies.

---

# 93. SQL Is First-Class

VoltStack no deberá requerir Graph DB para ReBAC.

---

# 94. Closure / Projection Tables

Para traversal frecuente podrán existir:

```text
authorization_relationship_paths
```

o materialized projections.

---

# 95. But

Esos paths son:

```text
derived / projection
```

no necesariamente canonical.

---

# 96. Delegation Persistence

Documento 20.

---

# 97. Conceptual delegation table

```text
authorization_delegations

id
tenant_id

grantor_type
grantor_id

grantee_type
grantee_id

status

active_from
expires_at

redelegatable
max_depth

authority_mode

version
authorization_version

created_at
updated_at
revoked_at nullable
```

---

# 98. Delegation Scope

No intentar meter toda la semántica en:

```text
scope TEXT
```

sin estructura.

---

# 99. Separate Delegation Restrictions

Puede existir:

```text
authorization_delegation_abilities
authorization_delegation_scopes
authorization_delegation_subjects
```

---

# 100. Example

```text
authorization_delegation_abilities

delegation_id
ability
```

---

# 101. Scope

```text
authorization_delegation_scopes

delegation_id
scope_type
scope_id
```

---

# 102. Resource Restrictions

```text
authorization_delegation_subjects

delegation_id
subject_type
subject_id
```

---

# 103. Flexible Conditions

ABAC restrictions adicionales pueden almacenarse como:

```text
structured JSON
```

si son validadas por schema/compiler.

---

# 104. Never Store Arbitrary Executable PHP

No guardar:

```php
'condition' => 'return eval(...)'
```

---

# 105. Capability Persistence

Capabilities pueden ser:

```text
reference
self-contained
hybrid
```

---

# 106. Reference Capability

Necesita canonical record.

---

# 107. Conceptual table

```text
authorization_capabilities

id
tenant_id

issuer_type
issuer_id

holder_type nullable
holder_id nullable

capability_type

audience

status

active_from
expires_at

single_use

secret_hash nullable

version

created_at
updated_at
consumed_at nullable
revoked_at nullable
```

---

# 108. Capability Abilities

```text
authorization_capability_abilities

capability_id
ability
```

---

# 109. Capability Restrictions

Pueden incluir:

```text
scope
resource
tenant
purpose
channel
IP/network condition
authentication assurance
```

---

# 110. Secret Storage

Bearer capability secret:

```text
NEVER plaintext
```

cuando persistence solo necesita verificarlo.

---

# 111. Hashing

Similar conceptualmente a:

```text
API token hashing
```

---

# 112. Signed Self-Contained Capability

Puede no requerir full record.

Pero para revocation puede mantenerse:

```text
capability id
revocation status
```

---

# 113. Capability Revocation Table

Opcional:

```text
authorization_capability_revocations

capability_id
revoked_at
reason
version
```

---

# 114. Single-Use Storage

Debe soportar atomic transition:

```text
ACTIVE
→
CONSUMED
```

como documento 27.

---

# 115. Impersonation Persistence

Una impersonation activa podrá representarse como:

```text
authorization_impersonation_sessions
```

si requiere persistencia distribuida.

---

# 116. Schema

```text
id
tenant_id

actor_type
actor_id

effective_principal_type
effective_principal_id

scope_definition

reason
ticket_reference nullable

started_at
expires_at
ended_at nullable

status

version
```

---

# 117. Impersonation Is Not Authentication Session

No mezclar con:

```text
auth_sessions
```

---

# 118. Relationship

Authentication Session:

```text
proves Actor identity
```

Authorization Impersonation Session:

```text
changes Effective Principal semantics
```

---

# 119. Approval Workflow Persistence

Documento 24.

---

# 120. Conceptual workflow table

```text
authorization_approval_workflows

id
tenant_id

workflow_type

requested_by_type
requested_by_id

subject_type
subject_id

ability

status

current_stage

version

created_at
expires_at
completed_at nullable
```

---

# 121. Approval Stage

```text
authorization_approval_stages

id
workflow_id

stage_order
required_approvals

status
version
```

---

# 122. Approval Decisions

```text
authorization_approval_decisions

id
workflow_id
stage_id

approver_type
approver_id

decision

reason nullable

created_at
version
```

---

# 123. Unique Approval Constraint

Para impedir doble voto:

```text
UNIQUE(
    workflow_id,
    stage_id,
    approver_type,
    approver_id
)
```

cuando el workflow no permita múltiples decisiones del mismo approver.

---

# 124. Approval Evidence

Final approval proof podrá tener:

```text
authorization_approval_proofs
```

---

# 125. Proof Fields

```text
id
workflow_id

ability
subject_type
subject_id

status

issued_at
expires_at
consumed_at nullable

version
```

---

# 126. Dual Control Constraint

Database puede ayudar, pero SoD sigue siendo regla del Domain.

---

# 127. Example

No basta con foreign keys para impedir:

```text
requester == approver
```

en todos los workflows.

Debe validarlo Approval Engine.

---

# 128. Contextual/Risk Data Persistence

Documento 23.

La mayoría del:

```text
AuthorizationContext
```

no debe persistirse como authorization canonical state.

---

# 129. Examples

```text
current IP
request timestamp
device posture
current MFA assurance
risk score
network zone
```

son generalmente runtime context.

---

# 130. Historical Risk

Si Audit necesita conservarlo:

```text
audit subsystem
```

debe recibir una representación segura.

---

# 131. Do Not Create Giant Context Table

Evitar:

```text
authorization_contexts
```

que almacene indiscriminadamente todos los requests.

---

# 132. Policy Storage

Policies pueden ser:

```text
code
compiled metadata
database configuration
external provider
```

---

# 133. Code Policies

No requieren persistencia dinámica.

---

# 134. Policy Metadata Registry

Puede existir una proyección:

```text
authorization_policy_registry
```

para:

```text
diagnostics
administration
versioning
```

---

# 135. Conceptual fields

```text
policy_name
subject_type
ability_pattern
provider
version
enabled
metadata
```

---

# 136. Policy Source

No guardar código PHP arbitrario en DB.

---

# 137. Dynamic Rule DSL

Si VoltStack soporta reglas configurables:

```text
DSL / AST
```

deberá ser:

```text
validated
typed
versioned
compiled
sandboxed semantically
```

---

# 138. Example

```json
{
  "all": [
    {"attribute": "resource.classification", "operator": "!=", "value": "restricted"},
    {"attribute": "context.risk", "operator": "<=", "value": "medium"}
  ]
}
```

---

# 139. Not

```text
eval("$user->...")
```

---

# 140. Policy Rule Storage

Opcional:

```text
authorization_policy_rules

id
tenant_id nullable
name
dsl_version
rule_document
version
enabled
```

---

# 141. Compile Result

No confundir canonical DSL con:

```text
compiled policy cache
```

---

# 142. Compiled Policy Cache

Es:

```text
derived
```

y puede borrarse.

---

# 143. Authorization Version Storage

Documento 27.

---

# 144. Version Table

Una implementación genérica podría usar:

```text
authorization_versions

dimension
reference_type
reference_id
version
updated_at
```

---

# 145. Example

```text
principal | user | 42 | 39
tenant    | tenant | 7 | 105
scope     | workspace | 91 | 18
```

---

# 146. Alternative

Versions pueden vivir directamente en:

```text
users.authorization_version
tenants.authorization_version
resources.authorization_version
```

---

# 147. Core Must Support Both

Mediante:

```php
AuthorizationVersionProviderInterface
```

---

# 148. Security Epoch Storage

Puede ser:

```text
authorization_system_state
```

---

# 149. Example

```text
key                  value
security_epoch       19
policy_generation    51
```

---

# 150. System State Must Be Tiny

No convertirlo en:

```text
generic configuration table
```

---

# 151. Revocation Storage

Puede existir un índice central.

---

# 152. Conceptual table

```text
authorization_revocations

id

authority_type
authority_id

tenant_id nullable

revoked_at
expires_at nullable

reason_code

version
```

---

# 153. Revocation Tombstones

Documento 27 recomienda conservar tombstones cuando eventos antiguos podrían reaparecer.

---

# 154. Revocation Retention

Debe considerar:

```text
max token lifetime
max event delay
audit requirements
```

---

# 155. Cache Tables

No deberán confundirse con canonical tables.

Naming recomendado:

```text
authorization_cache_*
authorization_projection_*
```

si están en SQL.

---

# 156. Effective Permission Projection

Ejemplo opcional:

```text
authorization_projection_principal_permissions

tenant_id
principal_type
principal_id

ability

scope_type
scope_id

source_fingerprint
projection_version
```

---

# 157. This Is Derived

Debe poder reconstruirse desde canonical authority.

---

# 158. Projection Lag

Guardar:

```text
source_version
projection_version
generated_at
```

---

# 159. Projection Validity

Solo usar si satisface consistency requirement.

---

# 160. Hierarchy Projection

Ejemplo:

```text
authorization_projection_scope_closure

ancestor_scope_id
descendant_scope_id
depth
hierarchy_version
```

---

# 161. Relationship Projection

```text
authorization_projection_relationship_paths
```

puede acelerar ReBAC.

---

# 162. Projection Rebuild

Debe poder ejecutarse:

```text
incrementally
full rebuild
```

---

# 163. Canonical Data Must Survive Projection Deletion

Test fundamental:

```text
DROP authorization_projection_*
```

no debe destruir autoridad canónica.

---

# 164. Repository Architecture

Core deberá hablar con:

```text
Repository Interfaces
```

---

# 165. Role Repository

```php
interface AuthorizationRoleRepositoryInterface
{
    public function find(string $id): ?AuthorizationRole;

    public function findByName(
        string $name,
        ?AuthorizationScopeReference $scope = null
    ): ?AuthorizationRole;
}
```

---

# 166. Assignment Repository

```php
interface PrincipalRoleAssignmentRepositoryInterface
{
    public function forPrincipal(
        PrincipalReference $principal,
        AuthorizationScopeReference $scope
    ): iterable;
}
```

---

# 167. Permission Repository

```php
interface PrincipalPermissionRepositoryInterface
{
    public function forPrincipal(
        PrincipalReference $principal,
        AuthorizationScopeReference $scope
    ): iterable;
}
```

---

# 168. Share Repository

```php
interface ResourceShareRepositoryInterface
{
    public function forResource(
        SubjectReference $resource
    ): iterable;

    public function forTarget(
        ShareTarget $target
    ): iterable;
}
```

---

# 169. Relationship Store

Documento 21:

```php
interface RelationshipStoreInterface
{
    public function findRelationships(
        RelationshipQuery $query
    ): RelationshipResult;
}
```

---

# 170. Delegation Repository

```php
interface DelegationGrantRepositoryInterface
{
    public function find(string $id): ?DelegatedAuthorizationGrant;

    public function activeFor(
        PrincipalReference $principal
    ): iterable;
}
```

---

# 171. Capability Repository

```php
interface AuthorizationCapabilityRepositoryInterface
{
    public function find(string $id): ?AuthorizationCapability;

    public function consume(
        string $id,
        int|string $expectedVersion
    ): CapabilityConsumptionResult;
}
```

---

# 172. Approval Repository

```php
interface AuthorizationApprovalRepositoryInterface
{
    public function findWorkflow(
        string $id
    ): ?AuthorizationApprovalWorkflow;
}
```

---

# 173. Mutation Repositories

Read contracts y write contracts pueden separarse.

---

# 174. CQRS-Friendly Design

```text
AuthorizationRoleReader
AuthorizationRoleWriter
```

pueden ser implementaciones distintas.

---

# 175. Core Does Not Require CQRS

Pero deberá ser compatible.

---

# 176. Query Interfaces

Hot-path authorization deberá preferir queries orientadas al problema.

No:

```text
load all roles
load all permissions
load all relationships
then filter in PHP
```

---

# 177. Example

```php
interface EffectiveAuthorityProviderInterface
{
    public function resolve(
        AuthorizationAuthorityQuery $query
    ): EffectiveAuthorityResult;
}
```

---

# 178. Provider Optimization

Adapter puede resolver:

```text
SQL joins
CTEs
Redis projections
Graph traversal
external PDP
```

---

# 179. Domain Model Remains Stable

Policy no necesita saber cómo.

---

# 180. Repository Does Not Mean ORM Repository

Puede ser:

```text
SQL
API
Redis
Graph
LDAP
```

---

# 181. Persistence Boundary

```text
Authorization Core
       │
       ↓
Repository Contract
       │
       ↓
Infrastructure Adapter
       │
       ├── Database
       ├── Redis
       ├── Graph
       └── External IAM
```

---

# 182. Database Integration Boundary

Database System provee:

```text
connections
transactions
query execution
locking
schema
migrations
```

Authorization define:

```text
what authorization data means
```

---

# 183. Authorization Must Not Implement Database Engine

No duplicar:

```text
transaction manager
query builder
connection pool
migration engine
```

---

# 184. Transaction Integration

Authorization Mutation Manager podrá solicitar:

```php
TransactionManagerInterface
```

desde Database subsystem.

---

# 185. Authorization Owns Transaction Semantics

Ejemplo:

```text
revoke role
advance auth version
write outbox
```

deben ocurrir juntos.

Database ejecuta la transacción.

Authorization define qué debe ser atómico.

---

# 186. Authentication Boundary

Authentication posee:

```text
password hashes
login sessions
MFA secrets
passkeys
remember-me tokens
OAuth authentication state
authentication credentials
```

---

# 187. Authorization Does Not Own Them

Authorization consume:

```text
authenticated Principal
AuthenticationAssuranceContext
session security metadata
```

---

# 188. Authentication Version

Si Auth expone:

```text
session_security_version
```

Authorization puede consumirlo.

No debe copiar toda la sesión.

---

# 189. Identity Boundary

Identity posee:

```text
User
Service identity
identity lifecycle
identity profile
identity linking
```

Authorization mantiene:

```text
PrincipalReference
```

---

# 190. Principal Deletion

Identity deberá producir:

```text
PrincipalDeleted
```

---

# 191. Authorization Reaction

Dependiendo de policy:

```text
revoke assignments
tombstone references
anonymize audit actor
cascade authorization-owned ephemeral grants
```

---

# 192. Do Not Blindly Cascade Delete Audit Evidence

Audit/Compliance retention puede exigir conservar referencias pseudonimizadas.

---

# 193. Tenant Boundary

Tenant System posee:

```text
tenant lifecycle
tenant metadata
tenant configuration
```

Authorization posee:

```text
tenant-scoped authority
```

---

# 194. Tenant Deletion

Debe coordinar:

```text
authorization assignments
shares
delegations
capabilities
approvals
projections
cache
```

---

# 195. Tenant Suspension

No necesariamente elimina nada.

Debe bloquear autoridad según Policy y mantener state para reactivación.

---

# 196. Organization Boundary

Si Organization es Domain entity:

```text
Organization System owns organization
Authorization references it
```

---

# 197. Team Boundary

Igual.

---

# 198. Workspace Boundary

Igual.

---

# 199. Resource Domain Boundary

Document System posee:

```text
Document
```

Authorization no debe duplicar:

```text
document title
content
domain state
```

---

# 200. Authorization-Relevant Resource Attributes

Policies pueden consumir:

```text
owner
classification
workspace
status
```

mediante:

```text
SubjectAttributeProvider
OwnershipResolver
ScopeResolver
```

---

# 201. No Authorization Shadow Database

Evitar copiar todos los atributos de negocio dentro de Authorization solo para evaluar Policies.

---

# 202. When Projection Is Justified

Puede proyectarse si:

```text
authorization query performance requires it
distributed PDP requires local copy
source is expensive
```

---

# 203. Projection Must Declare Source

```text
source system
source version
updated_at
consistency
```

---

# 204. Audit Boundary

Authorization genera:

```text
events
decision metadata
mutation metadata
```

Audit System decide:

```text
storage
retention
immutability
compliance export
```

---

# 205. Authorization Should Not Become Audit Database

No guardar todos los authorization attempts en:

```text
authorization_decisions
```

por default.

---

# 206. Decision Logging

Enviar:

```text
AuthorizationDecisionMade
```

a Telemetry/Audit.

---

# 207. Exception

Un sistema regulado puede instalar adapter que persista decisiones.

Eso sigue siendo:

```text
Audit concern
```

no canonical authority.

---

# 208. Telemetry Boundary

Telemetry almacena:

```text
metrics
traces
logs
```

Authorization no debe usar Telemetry como fuente de autoridad.

---

# 209. Risk Boundary

Risk Engine puede mantener:

```text
risk history
device reputation
behavior model
```

Authorization consume:

```text
RiskAssessment
```

---

# 210. Approval Boundary

Approval específicamente creado para autorizar una operación pertenece al Authorization subsystem.

Workflow empresarial general puede pertenecer a:

```text
Workflow System
```

---

# 211. Distinction

```text
Purchase order business approval
```

puede ser dominio.

```text
Security dual-control approval required to execute ability
```

pertenece naturalmente a Authorization.

---

# 212. Capability vs Token Boundary

Authentication token:

```text
proves identity
```

Authorization capability:

```text
carries authority
```

---

# 213. API Token

Puede cumplir ambos papeles parcialmente.

Debe separar conceptualmente:

```text
credential validity
```

de:

```text
authorization scope
```

---

# 214. Encryption at Rest

Sensitive authorization data puede requerir:

```text
encryption
```

---

# 215. Candidates

```text
capability metadata
delegation reason
approval justification
impersonation support ticket
policy sensitive conditions
```

---

# 216. Secrets

Nunca almacenar plaintext:

```text
bearer capability secrets
authorization recovery codes
signed-link secrets
```

cuando hashing sea suficiente.

---

# 217. Hash vs Encryption

Si solo se verifica igualdad:

```text
hash
```

Si se necesita recuperar:

```text
encryption
```

---

# 218. Data Minimization

Authorization deberá almacenar únicamente datos necesarios.

---

# 219. Example

Para ShareTarget User#42:

No copiar:

```text
name
email
phone
address
```

---

# 220. Store

```text
principal_type
principal_id
```

---

# 221. Metadata JSON

Útil, pero peligrosa si se abusa.

---

# 222. Use Metadata For

```text
extensible non-query-critical attributes
provider-specific metadata
optional descriptive properties
```

---

# 223. Do Not Put Core Query Fields in JSON

No esconder:

```text
tenant_id
principal_id
ability
status
expires_at
```

dentro de JSON.

---

# 224. Why

Necesitamos:

```text
indexes
constraints
query planner
foreign keys where possible
```

---

# 225. Schema Evolution

Cada persisted authorization object deberá ser versionable.

---

# 226. Storage Schema Version

Distinto de:

```text
authorization state version
```

---

# 227. Example

```text
schema_version = migration structure

authorization_version = security state generation
```

---

# 228. Migration Strategy

Authorization package deberá proporcionar migrations/adapters para su default SQL persistence.

---

# 229. Migration Ownership

Cada Quantum module deberá registrar sus migrations.

---

# 230. Migration Namespaces

Ejemplo:

```text
authorization/
    roles
    permissions
    relationships
    shares
    delegation
    capabilities
    approvals
    consistency
```

---

# 231. Optional Modules

No instalar 25 tablas si aplicación solo usa:

```text
Policies + simple RBAC
```

---

# 232. Modular Persistence

VoltStack deberá permitir habilitar:

```text
RBAC storage
Relationship storage
Sharing storage
Delegation storage
Capability storage
Approval storage
```

por separado.

---

# 233. Core Tables Minimal Profile

Una aplicación simple podría necesitar:

```text
authorization_roles
authorization_role_abilities
authorization_principal_roles
authorization_principal_permissions
```

---

# 234. Enterprise Profile

Puede añadir:

```text
authorization_relationships
authorization_resource_shares
authorization_delegations
authorization_capabilities
authorization_approval_*
authorization_versions
authorization_revocations
authorization_projection_*
```

---

# 235. Externalized Profile

Podría usar:

```text
External IAM
External ReBAC/PDP
```

y casi ninguna tabla local.

---

# 236. Persistence Profiles

```php
enum AuthorizationPersistenceProfile: string
{
    case Minimal = 'minimal';
    case Standard = 'standard';
    case Enterprise = 'enterprise';
    case Externalized = 'externalized';
    case Custom = 'custom';
}
```

---

# 237. No Profile Changes Semantics

Solo cambia:

```text
where/how state is stored
```

---

# 238. Foreign Keys

Usarlas donde sea seguro.

---

# 239. Polymorphic External References

`principal_type + principal_id` dificulta FK directa.

---

# 240. Do Not Fake Referential Integrity

Authorization deberá compensar mediante:

```text
reference validators
domain events
cleanup jobs
consistency checks
```

cuando DB no pueda imponer FK.

---

# 241. Internal Authorization Foreign Keys

Sí deberán usarse.

Ejemplo:

```text
role_abilities.role_id
→ authorization_roles.id
```

---

# 242. Delete Policies

Distinguir:

```text
RESTRICT
CASCADE
SOFT REVOKE
ARCHIVE
TOMBSTONE
```

---

# 243. Roles

Eliminar Role con assignments activos:

Preferible:

```text
RESTRICT
```

o workflow explícito.

---

# 244. Why

Un `CASCADE DELETE` silencioso puede revocar autoridad masivamente sin audit adecuado.

---

# 245. Revocation Instead of Deletion

Para objetos security-relevant:

```text
revoked
disabled
archived
```

suele ser mejor que physical delete inmediato.

---

# 246. Cleanup

Physical deletion puede ocurrir después según:

```text
retention policy
```

---

# 247. Soft Deletes

No usar `deleted_at` como semántica universal.

---

# 248. Why

```text
revoked
expired
consumed
archived
deleted
```

significan cosas diferentes.

---

# 249. Explicit Status

Preferible.

---

# 250. Timestamps

Authorization deberá usar:

```text
created_at
updated_at
active_from
expires_at
revoked_at
consumed_at
```

según semántica.

---

# 251. Timezone

Persistir timestamps en forma inequívoca.

Preferiblemente:

```text
UTC
```

---

# 252. Clock

Domain logic usa:

```text
ClockInterface
```

no `now()` disperso.

---

# 253. Version Columns

Security entities mutables deberían tener:

```text
version
```

para optimistic concurrency.

---

# 254. Version Increment

Debe ocurrir atómicamente con la mutación.

---

# 255. Example

```sql
UPDATE authorization_delegations
SET
    status = 'revoked',
    version = version + 1
WHERE id = ?
  AND version = ?
```

---

# 256. Zero Rows

Significa:

```text
not found
or
version conflict
```

adapter deberá distinguirlo cuando sea necesario.

---

# 257. Unique Constraints as Security Controls

No son solo performance.

Ejemplo:

```text
UNIQUE(role_id, ability)
```

evita estados duplicados ambiguos.

---

# 258. Another

```text
UNIQUE(workflow_id, stage_id, approver_type, approver_id)
```

puede reforzar dual control.

---

# 259. Another

Una sola owner assignment cuando el dominio exige exactamente uno.

---

# 260. Constraints Complement Domain Rules

No las reemplazan.

---

# 261. Index Design

Authorization suele consultar por:

```text
Principal
Ability
Tenant
Scope
Resource
Status
Expiry
```

---

# 262. General Index Rule

Indexar según:

```text
authorization access paths
```

no solo según tablas.

---

# 263. Hot Path Queries

Ejemplos:

```text
roles for principal in scope
permissions for principal
shares targeting principal
relationships from principal
active delegation by ID
capability by secure identifier
```

---

# 264. Expiration Indexes

Necesarios para cleanup:

```text
expires_at
```

---

# 265. Partial Indexes

PostgreSQL adapters podrían aprovechar:

```text
WHERE status = 'active'
```

---

# 266. Portability

Core no dependerá de partial indexes.

---

# 267. Partitioning

Grandes installations pueden particionar por:

```text
tenant
time
entity type
```

---

# 268. Core Neutrality

No imponer partitioning.

---

# 269. Multi-Tenant Physical Isolation

Authorization deberá funcionar con:

```text
shared database/shared schema

shared database/separate schema

database per tenant

external tenant authorization store
```

---

# 270. Repository Context

Cada repository operation deberá conocer:

```text
TenantContext
```

cuando corresponda.

---

# 271. Avoid Optional Tenant Accident

No diseñar:

```php
findAssignments($principal, $tenant = null)
```

si `null` puede accidentalmente significar:

```text
all tenants
```

---

# 272. Explicit Platform Context

Usar:

```text
PlatformScope
```

o API separada.

---

# 273. Cross-Tenant Administration

Debe ser explícita.

Nunca consecuencia de:

```text
tenant_id omitted
```

---

# 274. Sharding

Database subsystem puede shardear Authorization por:

```text
tenant_id
principal_id
```

---

# 275. Cross-Shard Queries

Evitar en hot path.

---

# 276. Data Locality

Idealmente authority de un Tenant se resuelve desde:

```text
same shard / region
```

---

# 277. Distributed Authorization Store

Repository contracts deberán soportar:

```text
network failures
consistency metadata
timeouts
```

---

# 278. Repository Result Metadata

Puede incluir:

```php
final readonly class AuthorizationRepositoryMetadata
{
    public function __construct(
        public AuthorizationConsistencyLevel $consistency,
        public ?int $latencyMs,
        public int|string|null $version,
    ) {}
}
```

---

# 279. Not Found vs Failure

Crítico.

---

# 280. Incorrect

```text
repository timeout
→ []
→ no roles
→ DENY
```

Aunque fail closed, perdimos diferencia entre:

```text
definitive no authority
```

y:

```text
unable to determine authority
```

---

# 281. Correct

```text
FOUND
NOT_FOUND
FAILURE
STALE
```

---

# 282. Repository Result

```php
enum AuthorizationRepositoryOutcome: string
{
    case Success = 'success';
    case NotFound = 'not_found';
    case Stale = 'stale';
    case Failure = 'failure';
}
```

---

# 283. Why

Observability y Policy pueden tratar failures correctamente.

---

# 284. Data Corruption

Authorization debe detectar estados imposibles.

Ejemplo:

```text
ACTIVE delegation
expires_at in past
status version inconsistent
```

---

# 285. Normalization

Runtime puede interpretar:

```text
expired by time
```

aunque status físico aún diga ACTIVE.

---

# 286. Cleanup Is Not Correctness

No depender de cron para que:

```text
expires_at
```

funcione.

---

# 287. Expiry Query

La evaluación debe incluir:

```text
expires_at > now
```

o validación equivalente.

---

# 288. Cleanup Jobs

Solo eliminan/archivan datos viejos.

---

# 289. Scheduled Cleanup

Puede procesar:

```text
expired assignments
expired delegations
consumed capabilities
old tombstones
old projections
```

---

# 290. Cleanup Authorization

El cleanup interno deberá usar:

```text
service identity
```

o privileged system operation bien definida.

No omnipotent bypass.

---

# 291. Backup and Restore

Authorization state es security-critical.

---

# 292. Restore Risk

Restaurar backup antiguo puede:

```text
reactivate revoked role
restore consumed capability
restore old delegation
```

---

# 293. Critical Principle

> **Restaurar datos de autorización no puede tratarse como restaurar datos ordinarios sin reconciliar el estado de seguridad posterior al backup.**

---

# 294. Example

```text
Monday:
capability ACTIVE

Tuesday:
capability REVOKED

Wednesday:
restore Monday backup
```

No debe quedar válidamente ACTIVE sin reconciliación.

---

# 295. Security Epoch after Restore

Una estrategia posible:

```text
increment global security epoch
```

y:

```text
invalidate sessions/tokens/grants as required
```

---

# 296. Restore Reconciliation

Debe considerar:

```text
revocation log
external identity state
security epoch
credential versions
audit history
```

---

# 297. Import/Export

Authorization configuration export puede incluir:

```text
roles
abilities
policy definitions
```

---

# 298. Avoid Exporting

```text
capability secrets
sensitive impersonation data
temporary security proofs
```

sin protección especial.

---

# 299. Role Templates vs Assignments

Exportar:

```text
role definitions
```

no implica exportar:

```text
actual user assignments
```

---

# 300. Data Portability

Logical IDs y aliases ayudan a mover configuration entre environments.

---

# 301. Environment Boundaries

No copiar automáticamente:

```text
production principal assignments
```

a:

```text
development
```

---

# 302. Seeders

Authorization podrá proporcionar:

```text
role/ability seed definitions
```

---

# 303. Idempotent Seeding

Ejemplo:

```text
ensure role exists
ensure ability assignment exists
```

---

# 304. Seeders Must Not Blindly Reset Production Authority

No hacer:

```text
DELETE all roles
INSERT defaults
```

---

# 305. Reconciliation Seeder

Preferir:

```text
desired-state reconciliation
```

con preview/diff.

---

# 306. Configuration as Code

Roles/policies pueden declararse:

```text
code/config
```

y sincronizarse con DB.

---

# 307. Source Ownership

Cada entity deberá saber si es:

```text
code-managed
admin-managed
external-managed
```

---

# 308. Managed Source

```php
enum AuthorizationManagedSource: string
{
    case Code = 'code';
    case Database = 'database';
    case External = 'external';
    case System = 'system';
}
```

---

# 309. Code-Managed Role

Admin UI no deberá modificarlo si policy lo prohíbe.

---

# 310. External-Managed Membership

Local admin no deberá sobrescribir authoritative external revocation.

---

# 311. Conflict Resolution

Debe ser explícito.

---

# 312. No Last-Writer-Wins by Accident

Especialmente entre:

```text
SCIM
admin UI
code sync
```

---

# 313. Storage Provider Registry

```php
interface AuthorizationStorageProviderRegistryInterface
{
    public function providerFor(
        AuthorizationStorageCapability $capability
    ): AuthorizationStorageProviderInterface;
}
```

---

# 314. Capabilities

Ejemplos:

```text
roles
relationships
delegation
capabilities
approvals
versions
```

---

# 315. Different Stores

VoltStack puede permitir:

```text
RBAC → SQL
ReBAC → Graph
Versions → Redis
Policies → Code
Memberships → LDAP
```

---

# 316. Unified Engine

Aunque persistence sea heterogénea:

```text
one AuthorizationDecision pipeline
```

---

# 317. Provider Precedence

No deberá resolverse accidentalmente por:

```text
registration order
```

si múltiples providers son authoritative.

---

# 318. Provider Strategy

Declarar:

```text
authoritative
supplemental
projection
fallback
```

---

# 319. Provider Authority Mode

```php
enum AuthorizationProviderAuthorityMode: string
{
    case Authoritative = 'authoritative';
    case Supplemental = 'supplemental';
    case Projection = 'projection';
    case Fallback = 'fallback';
}
```

---

# 320. Authoritative Revocation

Debe ganar frente a stale supplemental grant cuando así lo declare el modelo.

---

# 321. Composite Repository

Puede combinar providers.

---

# 322. Example

```text
Local RBAC
+
LDAP Groups
+
Temporary Delegation
```

---

# 323. Effective Authority

Se calcula en Engine.

No materializar necesariamente una giant table con todo.

---

# 324. Persistence Event Model

Mutaciones deberán emitir Domain Events.

---

# 325. Examples

```text
RoleCreated
RoleUpdated
RoleDeleted

RoleAssigned
RoleRevoked

PermissionGranted
PermissionRevoked

ShareCreated
ShareRevoked

RelationshipCreated
RelationshipRevoked

DelegationIssued
DelegationRevoked

CapabilityIssued
CapabilityConsumed
CapabilityRevoked

ApprovalCreated
ApprovalCompleted

AuthorizationVersionAdvanced
```

---

# 326. Events Are Not Canonical State by Default

A menos que backend sea event-sourced.

---

# 327. Transactional Event Publishing

Documento 27:

```text
mutation
+
version update
+
outbox
```

deben coordinarse.

---

# 328. Outbox Table

Si Database adapter la implementa:

```text
authorization_outbox
```

---

# 329. Fields

```text
id
event_type
aggregate_type
aggregate_id
payload
occurred_at
published_at nullable
attempts
```

---

# 330. Sensitive Payload

No colocar:

```text
secrets
full capability token
MFA material
```

en outbox.

---

# 331. Inbox

Consumers que requieran deduplication pueden utilizar:

```text
authorization_inbox
```

o infraestructura general del Event System.

---

# 332. Prefer Shared Event Infrastructure

Authorization no deberá duplicar Event Bus si VoltStack ya posee uno.

---

# 333. Persistence Exceptions

Propuesta:

```text
AuthorizationStorageException
AuthorizationRepositoryException
AuthorizationEntityNotFoundException
AuthorizationVersionConflictException
AuthorizationConstraintViolationException
AuthorizationReferenceIntegrityException
AuthorizationProjectionStaleException
AuthorizationProviderUnavailableException
```

---

# 334. Do Not Leak Database Exceptions

No exponer directamente:

```text
PDOException
SQLSTATE
RedisException
```

al Domain.

---

# 335. Preserve Cause Internally

Para diagnostics:

```text
previous exception
provider metadata
correlation id
```

---

# 336. Error Mapping

```text
duplicate assignment
→ AuthorizationConstraintViolation

optimistic lock mismatch
→ AuthorizationVersionConflict

provider timeout
→ AuthorizationProviderUnavailable
```

---

# 337. Storage Security

Database credentials pertenecen:

```text
Config / Secrets / Database
```

no Authorization.

---

# 338. Least Privilege Database User

Deployment avanzado puede usar DB principal con permisos limitados.

---

# 339. Example

Authorization read service:

```text
SELECT only
```

Mutation service:

```text
SELECT/INSERT/UPDATE
```

---

# 340. Direct Table Access

Aplicaciones no deberían modificar:

```text
authorization_*
```

arbitrariamente.

---

# 341. Correct

```text
AuthorizationMutationManager
```

---

# 342. Why

Direct SQL puede omitir:

```text
invariants
version increment
audit event
cache invalidation
outbox
```

---

# 343. Administrative Bulk Changes

También deben pasar por:

```text
bulk mutation service
```

que preserve invariants.

---

# 344. Bulk Role Assignment

No hacer N transacciones necesariamente.

Puede existir:

```php
AuthorizationBulkMutationManager
```

---

# 345. Bulk Atomicity

Policy deberá definir:

```text
all-or-nothing
best-effort
chunked
```

---

# 346. Versioning Bulk Mutation

Puede incrementar:

```text
tenant authorization version
```

una sola vez al final del batch.

---

# 347. Cache Invalidation

También puede emitir:

```text
coarse tenant invalidation
```

para evitar miles de eventos.

---

# 348. Data Retention

Distinguir:

```text
active authority data
revocation tombstones
operational history
audit history
```

---

# 349. Authorization Does Not Own Compliance Retention Automatically

Debe integrarse con:

```text
Data Lifecycle
Audit
Privacy
```

---

# 350. Personal Data Deletion

Si Principal se elimina:

Authorization puede:

```text
delete active assignments
revoke grants
pseudonymize retained references
```

según regulación.

---

# 351. Audit Identity

Puede conservar:

```text
opaque historical principal reference
```

sin perfil personal.

---

# 352. Region/Data Residency

Authorization stores pueden necesitar permanecer en:

```text
tenant region
```

---

# 353. Repository Routing

Puede utilizar:

```text
TenantContext
→ StorageRouter
→ regional store
```

---

# 354. Cross-Region Authorization

Debe considerar:

```text
latency
consistency
revocation propagation
```

---

# 355. Global Cache

No almacenar indiscriminadamente tenant authority fuera de residency constraints.

---

# 356. Encryption Key Boundary

Encryption System/Secrets System administra keys.

Authorization declara:

```text
which fields require protection
```

---

# 357. Serialization

Persisted structured data deberá tener:

```text
schema/version discriminator
```

---

# 358. Example

```json
{
  "schema": "authorization.condition",
  "version": 2,
  "data": {}
}
```

---

# 359. Never PHP serialize Domain Objects as Canonical Storage

Evitar:

```php
serialize($policyObject)
```

---

# 360. Why

```text
class changes
security risks
portability
language coupling
```

---

# 361. Canonical Serialization

Preferir:

```text
JSON
structured columns
normalized relational data
protobuf/message schema
```

según adapter.

---

# 362. Unknown Fields

Forward-compatible serializers pueden preservar o ignorar campos según schema.

---

# 363. Strict Security Fields

Campos security-critical desconocidos:

```text
fail validation
```

en lugar de ignorarlos silenciosamente.

---

# 364. Data Validation on Load

No confiar en DB únicamente porque:

```text
"it came from our database"
```

---

# 365. Validate

```text
status enum
ability format
scope type
expiry
version
condition schema
```

---

# 366. Corrupted Policy Data

Debe producir:

```text
FAILURE
```

no ALLOW.

---

# 367. Ability Naming Validation

Recomendación:

```text
[a-z][a-z0-9_.:-]*
```

o grammar formal equivalente.

---

# 368. Reserved Namespace

VoltStack puede reservar:

```text
voltstack.*
system.*
authorization.*
```

---

# 369. Application Namespace

Ejemplo:

```text
document.*
billing.*
workspace.*
```

---

# 370. Storage Boundaries Diagram

```text
┌─────────────────────────────────────────────────────────┐
│                  APPLICATION DOMAIN                     │
│ Documents / Orders / Organizations / Workspaces         │
└──────────────────────────┬──────────────────────────────┘
                           │ references / attributes
                           ↓
┌─────────────────────────────────────────────────────────┐
│                 AUTHORIZATION DOMAIN                    │
│                                                         │
│ Roles                                                   │
│ Permissions                                             │
│ Scoped Assignments                                      │
│ Shares                                                  │
│ Relationships                                           │
│ Delegations                                             │
│ Capabilities                                            │
│ Approval Authority                                      │
│ Versions / Revocations                                  │
│                                                         │
└──────────┬───────────────────────┬──────────────────────┘
           │                       │
           ↓                       ↓
┌─────────────────────┐   ┌──────────────────────────────┐
│ Persistence Adapters│   │ External Providers           │
│                     │   │                              │
│ SQL                 │   │ LDAP                         │
│ Redis               │   │ IAM                          │
│ Graph               │   │ SCIM                         │
└──────────┬──────────┘   └──────────────────────────────┘
           │
           ↓
┌─────────────────────────────────────────────────────────┐
│                    DATABASE SYSTEM                      │
│ Connections / Transactions / Queries / Locks / Schema   │
└─────────────────────────────────────────────────────────┘
```

---

# 371. Ownership Matrix

| Data | Canonical Owner |
| --- | --- |
| Password | Authentication |
| MFA Secret | Authentication |
| User identity | Identity |
| Tenant | Tenant Domain |
| Workspace | Application Domain |
| Document | Application Domain |
| Role | Authorization |
| Role Assignment | Authorization |
| Direct Permission | Authorization |
| Resource Share | Authorization |
| Authorization Relationship | Authorization or authoritative provider |
| Delegation Grant | Authorization |
| Capability Authority | Authorization |
| Impersonation Authority | Authorization |
| Security Approval Proof | Authorization |
| Risk Score | Risk Provider |
| Decision Logs | Audit/Telemetry |
| Decision Cache | Authorization Runtime |
| Authorization Projection | Authorization Infrastructure |

---

# 372. Canonical vs Projection Matrix

| Object | Canonical | Derived/Projection |
| --- | ---: | ---: |
| Role | Yes | No |
| Role Ability Assignment | Yes | No |
| Principal Role Assignment | Yes | No |
| Effective Permission Set | No | Yes |
| Resource Share | Yes | No |
| Relationship | Usually | Sometimes external |
| Relationship Path | No | Yes |
| Delegation | Yes | No |
| Capability Record | Depends on capability type | Possible |
| Approval Decision | Yes | No |
| Effective Authorization Decision | No | Runtime/cache |
| Scope Closure | Usually no | Yes |
| Security Epoch | Yes | No |

---

# 373. Suggested Package Structure

```text
Quantum/
└── Authorization/
    ├── Domain/
    │   ├── Ability/
    │   ├── Role/
    │   ├── Permission/
    │   ├── Assignment/
    │   ├── Share/
    │   ├── Relationship/
    │   ├── Delegation/
    │   ├── Capability/
    │   ├── Approval/
    │   └── Versioning/
    │
    ├── Persistence/
    │   ├── Contracts/
    │   │   ├── AuthorizationRoleRepositoryInterface.php
    │   │   ├── PrincipalRoleAssignmentRepositoryInterface.php
    │   │   ├── PrincipalPermissionRepositoryInterface.php
    │   │   ├── ResourceShareRepositoryInterface.php
    │   │   ├── RelationshipStoreInterface.php
    │   │   ├── DelegationGrantRepositoryInterface.php
    │   │   ├── AuthorizationCapabilityRepositoryInterface.php
    │   │   ├── AuthorizationApprovalRepositoryInterface.php
    │   │   └── AuthorizationVersionProviderInterface.php
    │   │
    │   ├── Model/
    │   │   ├── AuthorizationStorageSemantics.php
    │   │   ├── AuthorizationPersistenceProfile.php
    │   │   ├── AuthorizationManagedSource.php
    │   │   ├── AuthorizationProviderAuthorityMode.php
    │   │   └── AuthorizationRepositoryMetadata.php
    │   │
    │   ├── Registry/
    │   │   ├── AuthorizationTypeRegistry.php
    │   │   ├── AuthorizationStorageProviderRegistry.php
    │   │   └── AuthorizationRepositoryRegistry.php
    │   │
    │   ├── Projection/
    │   │   ├── EffectivePermissionProjection.php
    │   │   ├── ScopeHierarchyProjection.php
    │   │   ├── RelationshipProjection.php
    │   │   └── ProjectionRebuilder.php
    │   │
    │   ├── SQL/
    │   │   ├── Repository/
    │   │   ├── Mapper/
    │   │   ├── Query/
    │   │   ├── Schema/
    │   │   └── Migration/
    │   │
    │   ├── Cache/
    │   ├── Graph/
    │   └── External/
    │
    └── Exceptions/
        ├── AuthorizationStorageException.php
        ├── AuthorizationRepositoryException.php
        ├── AuthorizationEntityNotFoundException.php
        ├── AuthorizationConstraintViolationException.php
        ├── AuthorizationReferenceIntegrityException.php
        ├── AuthorizationProjectionStaleException.php
        └── AuthorizationProviderUnavailableException.php
```

---

# 374. SQL Adapter Suggested Structure

```text
Persistence/
└── SQL/
    ├── Role/
    │   ├── SqlRoleRepository.php
    │   └── SqlRoleMapper.php
    │
    ├── Assignment/
    │   ├── SqlRoleAssignmentRepository.php
    │   └── SqlPermissionRepository.php
    │
    ├── Relationship/
    │   └── SqlRelationshipStore.php
    │
    ├── Sharing/
    │   └── SqlResourceShareRepository.php
    │
    ├── Delegation/
    │   └── SqlDelegationRepository.php
    │
    ├── Capability/
    │   └── SqlCapabilityRepository.php
    │
    ├── Approval/
    │   └── SqlApprovalRepository.php
    │
    ├── Consistency/
    │   └── SqlAuthorizationVersionStore.php
    │
    └── Schema/
        └── AuthorizationSchemaManager.php
```

---

# 375. Suggested Default Table Family

```text
authorization_roles
authorization_role_abilities

authorization_principal_roles
authorization_principal_permissions

authorization_resource_shares
authorization_relationships

authorization_delegations
authorization_delegation_abilities
authorization_delegation_scopes
authorization_delegation_subjects

authorization_capabilities
authorization_capability_abilities
authorization_capability_revocations

authorization_impersonation_sessions

authorization_approval_workflows
authorization_approval_stages
authorization_approval_decisions
authorization_approval_proofs

authorization_versions
authorization_revocations

authorization_system_state

authorization_outbox
```

Opcionales:

```text
authorization_scopes
authorization_scope_memberships

authorization_policy_rules

authorization_projection_principal_permissions
authorization_projection_scope_closure
authorization_projection_relationship_paths
```

---

# 376. Important Naming Rule

No utilizar tablas ambiguas como:

```text
permissions
roles
relationships
```

en el paquete base si pueden colisionar con aplicación/otros paquetes.

Preferir:

```text
authorization_*
```

---

# 377. Configurable Prefix

Adapter puede permitir:

```php
'table_prefix' => 'authorization_',
```

pero mappings internos deberán centralizarse.

---

# 378. No Hardcoded Table Names in Domain

Nunca:

```php
DB::table('authorization_roles')
```

desde Policy/Manager.

---

# 379. Testing Persistence

Cada repository contract deberá tener:

```text
contract test suite
```

---

# 380. Repository Contract Tests

La misma suite deberá poder ejecutarse contra:

```text
SQLite
MySQL
PostgreSQL
MariaDB
```

cuando adapter lo soporte.

---

# 381. Role Persistence Test

```text
create
find
update
version conflict
disable
```

---

# 382. Assignment Uniqueness Test

No crear assignment duplicado incompatible.

---

# 383. Expiration Test

Expired assignment nunca aparece como active authority aunque cleanup no haya corrido.

---

# 384. Tenant Isolation Test

Queries de Tenant A nunca retornan records B.

---

# 385. Cross-Tenant Reference Test

Share A → resource B debe rechazarse salvo feature explícito.

---

# 386. Capability Secret Test

Database nunca contiene secret plaintext.

---

# 387. Single-Use Capability Test

Concurrent consumption produce exactamente un success.

---

# 388. Projection Deletion Test

Eliminar projection no elimina canonical authority.

---

# 389. Projection Rebuild Test

Rebuild produce semántica equivalente al canonical source.

---

# 390. Provider Failure Test

Timeout no se interpreta como empty result.

---

# 391. Schema Serialization Test

Unknown critical field falla correctamente.

---

# 392. Migration Upgrade Test

Data existente conserva semántica de autoridad.

---

# 393. Migration Rollback Security Test

Rollback no debe reactivar autoridad previamente revocada inadvertidamente.

---

# 394. Restore Test

Restore antiguo requiere reconciliation/security generation strategy.

---

# 395. Bulk Mutation Test

Batch mantiene:

```text
versioning
invalidation
audit events
```

---

# 396. Persistent Worker Test

Repository objects compartibles no retienen:

```text
Principal
Tenant
Decision
Scope
```

entre requests.

---

# 397. Property-Based Invariant

Eliminar cualquier cache/projection nunca crea autoridad nueva.

---

# 398. Property-Based Invariant

Una projection nunca puede ser más nueva que su source version de manera ficticia.

---

# 399. Property-Based Invariant

Un record tenant-bound nunca cambia de Tenant mediante update ordinario si la semántica exige recreación/move controlado.

---

# 400. Property-Based Invariant

Un expired grant nunca produce ALLOW independientemente del status materializado.

---

# 401. Property-Based Invariant

Un revoked capability no vuelve a ACTIVE mediante evento/version anterior.

---

# 402. Property-Based Invariant

Failure de repository no equivale a autoridad positiva.

---

# 403. Property-Based Invariant

Duplicar un canonical row no debe duplicar semánticamente authority; constraints deberán evitar estados ambiguos.

---

# 404. Property-Based Invariant

Una mutación security-relevant incrementa la versión correspondiente o invalida mediante mecanismo equivalente.

---

# 405. Security Invariants

### Invariante 1

Secrets de bearer capabilities no se almacenan plaintext cuando no es necesario recuperarlos.

### Invariante 2

Tenant isolation se aplica en persistence y no únicamente después de cargar resultados.

### Invariante 3

Direct DB mutation no es API soportada para cambiar authority.

### Invariante 4

External references utilizan aliases estables, no nombres arbitrarios de clases persistidos.

### Invariante 5

Corrupted persisted authorization state falla de forma segura.

---

# 406. Data Ownership Invariants

### Invariante 1

Authorization no es propietario de Password.

### Invariante 2

Authorization no es propietario de User identity.

### Invariante 3

Authorization no duplica business resources innecesariamente.

### Invariante 4

Authorization es propietario de la semántica de Roles, Permissions, Shares, Delegations y Capabilities que administra.

### Invariante 5

Audit records pertenecen a Audit aunque sean originados por Authorization.

---

# 407. Persistence Invariants

### Invariante 1

Canonical state y cache son distinguibles.

### Invariante 2

Projection puede reconstruirse.

### Invariante 3

Cache puede eliminarse sin pérdida de authority configuration.

### Invariante 4

Version updates ocurren atómicamente con mutaciones relevantes.

### Invariante 5

Events externos se publican después de commit mediante mecanismo seguro.

---

# 408. Repository Invariants

### Invariante 1

Repositories no filtran Tenant de forma opcional o accidental.

### Invariante 2

NotFound y Failure son estados diferentes.

### Invariante 3

Consistency metadata puede propagarse cuando el provider es distribuido.

### Invariante 4

Repositories no exponen detalles tecnológicos al Domain.

### Invariante 5

Providers heterogéneos mantienen una semántica de Authorization uniforme.

---

# 409. Complete Persistence Flow

```text
Authorization Request
        ↓
Authorization Planner
        ↓
Required Authority Sources
        ↓
Repository / Provider Registry
        ↓
 ┌──────┼────────┬──────────┐
 ↓      ↓        ↓          ↓
RBAC   ReBAC  Delegation   External
 │      │        │          IAM
 ↓      ↓        ↓           │
SQL    Graph     SQL          API
 │      │        │           │
 └──────┴────┬───┴───────────┘
             ↓
     Normalized Authority
             ↓
       Policy Evaluation
             ↓
           Decision
```

---

# 410. Complete Mutation Flow

```text
Administrative Request
        ↓
Authorize Mutation Ability
        ↓
AuthorizationMutationManager
        ↓
Load Current Canonical State
        ↓
Validate Domain Invariants
        ↓
Begin Transaction
        ↓
Apply Mutation
        ↓
Advance Authorization Version
        ↓
Create Outbox Event
        ↓
Commit
        ↓
Invalidate Local State
        ↓
Publish Distributed Invalidation
        ↓
Audit / Telemetry
```

---

# 411. Canonical vs Runtime Architecture

```text
               CANONICAL AUTHORITY
                       │
        ┌──────────────┼──────────────┐
        ↓              ↓              ↓
      Roles       Relationships    Delegations
        │              │              │
        └──────────────┼──────────────┘
                       ↓
                 PROJECTIONS
                       │
                       ↓
                    CACHE
                       │
                       ↓
              REQUEST MEMOIZATION
                       │
                       ↓
                  DECISION
```

Regla:

```text
Canonical
   ↓
Projection
   ↓
Cache
   ↓
Decision
```

Nunca:

```text
Decision Cache
   ↓
becomes Canonical Authority
```

---

# 412. Minimal VoltStack Deployment

```text
Authorization Core
       ↓
SQL RBAC Repositories
       ↓
VoltStack Database
```

Suficiente para:

```text
roles
permissions
policies
scoped assignments
```

---

# 413. Standard Deployment

```text
Authorization Core
       │
       ├── SQL canonical authority
       │
       ├── request memoization
       │
       └── shared cache
```

---

# 414. Enterprise Deployment

```text
                    Authorization Core
                           │
        ┌──────────────────┼──────────────────┐
        ↓                  ↓                  ↓
      SQL RBAC          ReBAC Store       External IAM
        │                  │                  │
        └──────────────┬───┴──────────────────┘
                       ↓
               Version Projection
                       ↓
                 Shared Cache
                       ↓
              Distributed Events
                       ↓
              Multiple VoltStack Nodes
```

---

# 415. Architectural Decision

VoltStack no deberá construir Authorization alrededor de una única tabla:

```text
user_permissions
```

porque el sistema completo ya necesita representar:

```text
RBAC
ABAC
ReBAC
Scopes
Tenant Isolation
Ownership
Sharing
Delegation
Capabilities
Impersonation
Service Authority
Risk Conditions
Approval Workflows
Revocation
Distributed Consistency
```

La persistencia debe reflejar esos conceptos sin mezclarlos innecesariamente.

---

# 416. Anti-Patterns

Evitar:

```text
one giant authorization table
```

Evitar:

```text
all metadata in JSON
```

Evitar:

```text
serialize PHP objects
```

Evitar:

```text
class names as persistent public types
```

Evitar:

```text
cache as source of truth
```

Evitar:

```text
duplicate every domain entity
```

Evitar:

```text
repository timeout = empty permissions
```

Evitar:

```text
tenant_id nullable means global
```

Evitar:

```text
physical delete = revocation
```

Evitar:

```text
direct DB updates bypassing AuthorizationMutationManager
```

---

# 417. Filosofía arquitectónica

El modelo de persistencia de VoltStack Authorization deberá seguir estas reglas:

```text
Store authority, not application duplication.

Reference identities and resources;
do not own what belongs to another domain.

Canonical state is explicit.

Derived state is rebuildable.

Caches are disposable.

Projections advertise freshness.

Repositories expose authorization semantics,
not database technology.

Tenant boundaries exist at storage time,
not only after retrieval.

Revocation is data, not merely deletion.

Expiration is evaluated at runtime,
not delegated to cleanup jobs.

Security mutations are versioned.

External providers declare authority
and consistency semantics.

Authorization tables are not modified
outside controlled mutation APIs.

Persistent workers may retain repositories,
but never request authority state.
```

---

# 418. Resultado esperado

Con `28_AUTHORIZATION_DATA_MODEL_PERSISTENCE_AND_STORAGE_BOUNDARIES_SYSTEM.md`, VoltStack obtiene un modelo claro para responder:

```text
¿Qué datos pertenecen realmente a Authorization?

¿Qué datos solo referencia?

¿Qué información es canonical?

¿Qué información es una projection?

¿Qué puede almacenarse en cache?

¿Qué puede reconstruirse?

¿Qué debe versionarse?

¿Qué necesita revocación explícita?

¿Qué pertenece a Authentication?

¿Qué pertenece a Identity?

¿Qué pertenece al Tenant/Domain?

¿Cómo soportamos SQL, Redis, Graph e IAM externo
sin contaminar el Core?
```

El resultado será un Authorization System capaz de utilizar desde:

```text
SQLite
```

en una aplicación pequeña hasta:

```text
SQL
+
Redis
+
Graph Store
+
External IAM
+
Distributed Event Bus
```

en una arquitectura empresarial sin cambiar la semántica fundamental del framework.

La regla final será:

> **VoltStack Authorization posee la semántica de la autoridad, no todos los datos que participan en una decisión.**

Por tanto:

```text
DOMAIN DATA
     +
AUTHORIZATION CANONICAL STATE
     +
EXTERNAL AUTHORITY
     +
DERIVED PROJECTIONS
     +
DISPOSABLE CACHE
     ↓
NORMALIZED AUTHORIZATION MODEL
     ↓
AUTHORIZATION DECISION
```

Esta separación permitirá que el sistema permanezca modular, portable, testeable y seguro incluso cuando la infraestructura de almacenamiento cambie completamente.

---

# 419. Siguiente documento recomendado

Después de haber definido configuración/bootstrap, lifecycle/extensibilidad, consistencia distribuida y ahora persistencia, todavía falta una pieza importante antes de cerrar definitivamente Authorization:

**`29_AUTHORIZATION_ADMINISTRATION_MANAGEMENT_AND_OPERATIONAL_TOOLING_SYSTEM.md`**

Este documento debería formalizar la capa administrativa y operacional del sistema:

```text
Role management
Permission management
Assignments
Policy inspection

Authorization CLI

Role/permission synchronization

Authorization diagnostics

Principal authority inspection

"Why can this user do this?"

"Who can access this resource?"

Permission simulation

Policy dry-run

Impact analysis before role changes

Bulk authorization administration

Delegation/capability administration

Approval administration

Revocation operations

Cache/projection management

Consistency diagnostics

Authorization health checks

Administrative security boundaries

Break-glass operational tooling

Framework developer tooling
```

Después de ese documento, la secuencia ya estaría muy cerca del cierre y tendría sentido pasar a **testing/security verification**, **performance/compilation** y finalmente al documento de **integración y arquitectura final del Authorization System**, evitando seguir agregando documentos indefinidamente.
