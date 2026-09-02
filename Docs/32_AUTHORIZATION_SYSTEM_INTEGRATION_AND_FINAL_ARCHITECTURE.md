# VoltStack Authorization System

## System Integration and Final Architecture

**Documento:** `32_AUTHORIZATION_SYSTEM_INTEGRATION_AND_FINAL_ARCHITECTURE.md`  
**Sistema:** Authorization  
**Framework:** VoltStack  
**Módulo:** `Quantum/Authorization`  
**Estado:** Arquitectura final consolidada  
**Versión objetivo:** 1.x+  
**Clasificación:** Documento maestro / Final Architecture Specification

---

# 1. Propósito

Este documento define la arquitectura final e integrada del **Authorization System de VoltStack**.

Su objetivo es consolidar en un único modelo todos los subsistemas definidos previamente:

```text
Authorization Core
Principal Model
Actor / Effective Principal
Request Context
Policies
Gates
Voters
Decision Strategies
RBAC
ABAC
ReBAC
Tenant Isolation
Hierarchical Scopes
Ownership
Sharing
Relationships
Delegation
Capabilities
Impersonation
Service-to-Service Authorization
Contextual Access
Risk-Based Access
Authentication Assurance
Challenges
Approval Workflows
Dual Control
Separation of Duties
Configuration
Bootstrap
Container Integration
Lifecycle
Events
Hooks
Plugins
State Consistency
Concurrency
Distributed Coordination
Persistence
Administration
Testing
Security Assurance
Compliance
Compilation
Caching
Performance
Resource Governance
```

Este documento establece la respuesta definitiva a la pregunta:

> **¿Cómo decide VoltStack si un Principal puede ejecutar una operación sobre un recurso determinado, bajo un contexto, tenant, scope y estado de seguridad concretos?**

---

# 2. Misión del Authorization System

Authorization tendrá una única responsabilidad fundamental:

```text
Determinar si una autoridad autenticada o explícitamente
representada puede realizar una operación determinada
sobre un objetivo concreto bajo las condiciones actuales.
```

Authorization no deberá confundirse con:

```text
Authentication
Identity Management
Session Management
Business Validation
Routing
HTTP
Database Security
Rate Limiting
Workflow Execution
```

Aunque se integrará profundamente con todos ellos.

---

# 3. Authentication vs Authorization

La separación será estricta.

```text
Authentication
    ↓
Who are you?

Authorization
    ↓
What may you do?
```

Authentication produce información como:

```text
Authenticated Identity
Authentication Method
Authentication Assurance
Session
Credential State
Actor
```

Authorization consume esa información.

---

# 4. Regla fundamental

```text
AUTHENTICATION SUCCESS
≠
AUTHORIZATION SUCCESS
```

Un usuario autenticado puede seguir obteniendo:

```text
DENY
CHALLENGE
FAILURE
```

---

# 5. Authorization Core Principle

La arquitectura completa se construye alrededor de la siguiente regla:

> **Toda autoridad debe tener un Principal, una Ability, un contexto, un alcance y una fuente explicable.**

Nunca deberá existir:

```text
"internal request = allowed"

"is admin = bypass everything"

"authenticated = allowed"

"system process = trusted"

"same tenant = allowed"
```

---

# 6. Los cinco ejes fundamentales

Toda decisión deberá responder:

```text
WHO?
WHAT?
WHICH RESOURCE?
WHERE?
WHY?
```

Representados como:

```text
WHO
→ Principal / Actor

WHAT
→ Ability

WHICH RESOURCE
→ Subject / Resource

WHERE
→ Tenant / Scope

WHY
→ Authority Sources + Policies + Conditions
```

---

# 7. Modelo conceptual final

```text
Principal
   │
   ├── has Role
   ├── has Permission
   ├── owns Resource
   ├── has Relationship
   ├── received Share
   ├── received Delegation
   ├── holds Capability
   └── belongs to Scope
             │
             ↓
          Ability
             │
             ↓
          Resource
             │
             ↓
           Policy
             │
             ↓
          Context
             │
             ↓
            Risk
             │
             ↓
        Approval / SoD
             │
             ↓
       Security Constraints
             │
             ↓
          Decision
```

---

# 8. Principal Model

Todo participante deberá representarse mediante:

```php
interface PrincipalInterface
{
    public function principalId(): PrincipalId;

    public function principalType(): PrincipalType;
}
```

---

# 9. Tipos oficiales de Principal

VoltStack deberá soportar al menos:

```php
enum PrincipalType: string
{
    case Human = 'human';
    case Service = 'service';
    case System = 'system';
    case Anonymous = 'anonymous';
}
```

---

# 10. Human Principal

Representa:

```text
User
Administrator
Operator
Customer
Employee
Guest User
```

---

# 11. Service Principal

Representa:

```text
Microservice
Worker Service
External Integration
Machine Identity
API Integration
```

No deberá considerarse automáticamente privilegiado.

---

# 12. System Principal

Representa procesos explícitamente internos del framework.

Ejemplos:

```text
scheduled maintenance
system migration
internal security process
```

Pero:

> **SystemPrincipal no significa omnipotencia.**

También necesita autoridad explícita.

---

# 13. Anonymous Principal

Las operaciones públicas deberán usar:

```text
AnonymousPrincipal
```

en lugar de:

```text
null
```

Esto evita ramas especiales dentro del motor.

---

# 14. Actor vs Effective Principal

La arquitectura deberá preservar siempre:

```text
Actor
Effective Principal
```

---

# 15. Actor

Es:

```text
quién realmente está ejecutando la operación
```

---

# 16. Effective Principal

Es:

```text
la identidad cuya autoridad está siendo evaluada
```

---

# 17. Operación normal

```text
Actor
=
Effective Principal
```

---

# 18. Impersonation

```text
Actor
=
Support Agent

Effective Principal
=
Customer
```

---

# 19. Contexto final

```php
final readonly class PrincipalContext
{
    public function __construct(
        public PrincipalInterface $actor,
        public PrincipalInterface $effectivePrincipal,
        public AuthorizationAuthorityContext $authority,
    ) {}
}
```

---

# 20. Nunca perder Actor

Regla de seguridad:

> **Toda impersonación, delegación o ejecución indirecta deberá preservar la identidad del Actor original.**

Esto afecta:

```text
Audit
SoD
Approval
Risk
Telemetry
Delegation
Incident Investigation
```

---

# 21. Ability

Toda operación autorizable deberá representarse mediante una Ability.

Ejemplos:

```text
document.view
document.update
document.delete

workspace.manage

user.impersonate

payment.approve
payment.execute

deployment.execute
```

---

# 22. Ability Contract

```php
final readonly class Ability
{
    public function __construct(
        public string $name,
        public AuthorizationSensitivity $sensitivity,
        public AbilityAuthorizationMetadata $metadata,
    ) {}
}
```

---

# 23. Ability Metadata

Puede declarar:

```text
subject type
tenant requirements
scope requirements
principal types
delegatable
redelegatable
capability issuable
impersonation allowed
service callable
approval requirements
consistency profile
context requirements
risk requirements
```

---

# 24. Ability Registry

```php
interface AbilityRegistryInterface
{
    public function get(string $ability): Ability;

    public function has(string $ability): bool;
}
```

Producción utilizará preferentemente un registry compilado.

---

# 25. Unknown Ability

Una Ability desconocida nunca deberá interpretarse como:

```text
ALLOW
```

---

# 26. AuthorizationRequest

Toda decisión deberá normalizarse a:

```php
final readonly class AuthorizationRequest
{
    public function __construct(
        public AbilityReference $ability,
        public PrincipalContext $principal,
        public ?AuthorizationSubjectReference $subject,
        public ?TenantReference $tenant,
        public ?AuthorizationScopeReference $scope,
        public AuthorizationOperationDescriptor $operation,
    ) {}
}
```

---

# 27. AuthorizationContext

El estado contextual estará separado del request estructural.

```php
final readonly class AuthorizationContext
{
    public function __construct(
        public AuthenticationContext $authentication,
        public SessionContext $session,
        public RequestContext $request,
        public EnvironmentContext $environment,
        public OperationContext $operation,
        public ?RiskContext $risk = null,
    ) {}
}
```

---

# 28. Contexto inmutable

AuthorizationContext deberá ser:

```text
immutable
execution scoped
explicitly replaceable
versioned
```

Nunca global mutable.

---

# 29. FrankenPHP Golden Rule

```text
Application state may persist.

Authorization request state must not.
```

---

# 30. Tenant Boundary

Tenant Isolation será una restricción estructural de primera clase.

No será una Policy opcional.

---

# 31. Tenant rule

Por defecto:

```text
Principal Tenant
=
Subject Tenant
=
Requested Tenant
```

cuando la Ability sea tenant-bound.

---

# 32. Cross-Tenant

Acceso cross-tenant deberá requerir autoridad explícita.

Nunca:

```text
platform admin
→ automatic access to every tenant resource
```

---

# 33. Tenant Evaluator

```php
interface TenantIsolationEvaluatorInterface
{
    public function evaluate(
        AuthorizationRequest $request,
        AuthorizationContext $context,
    ): AuthorizationCandidateDecision;
}
```

Será normalmente:

```text
non-bypassable
```

---

# 34. Hierarchical Scope Model

VoltStack soportará:

```text
Platform
   ↓
Tenant
   ↓
Organization
   ↓
Business Unit
   ↓
Team
   ↓
Workspace
   ↓
Project
   ↓
Resource
```

No todos los niveles son obligatorios.

---

# 35. Scope

Authority puede estar limitada a:

```text
Tenant
Organization
Team
Workspace
Project
```

---

# 36. Scoped Role

Ejemplo:

```text
Alice
→ Editor
→ Workspace A
```

no significa:

```text
Alice
→ Editor
→ Workspace B
```

---

# 37. Scope Inheritance

Deberá ser explícita.

Modos conceptuales:

```text
Exact
Direct Children
Descendants
Custom
```

---

# 38. Scope Boundary

Un scope puede detener herencia.

Ejemplo:

```text
Organization
   ↓
Workspace
   ↓
Confidential Workspace
       ↑
       inheritance boundary
```

---

# 39. Administrative vs Content Authority

Un administrador organizacional puede tener:

```text
workspace.create
workspace.delete
membership.manage
```

sin necesariamente:

```text
document.read
```

---

# 40. Authorization Scope Domains

```text
Administration
Content
Membership
Billing
Security
```

permiten separar ambas dimensiones.

---

# 41. RBAC

VoltStack integrará Role-Based Access Control.

Modelo:

```text
Principal
   ↓
Role
   ↓
Permission / Ability
```

---

# 42. RBAC no será el motor completo

RBAC es:

```text
one authority source
```

no:

```text
the entire Authorization System
```

---

# 43. Role Assignment

```php
final readonly class ScopedRoleAssignment
{
    public function __construct(
        public PrincipalReference $principal,
        public RoleReference $role,
        public AuthorizationScopeReference $scope,
    ) {}
}
```

---

# 44. Direct Permission Grants

Podrán existir cuando el modelo lo requiera.

Pero deberán ser explícitos y auditables.

---

# 45. ABAC

Attribute-Based Access Control permitirá condiciones basadas en:

```text
Principal Attributes
Resource Attributes
Environment
Operation
Authentication Assurance
Tenant Metadata
Scope Metadata
```

---

# 46. Example ABAC

```text
Principal.department
=
Document.department

AND

Document.classification <= Principal.clearance
```

---

# 47. Attribute Trust

Cada atributo sensible deberá poder indicar:

```text
source
trust level
provenance
```

---

# 48. Client Input

Un header proporcionado por el cliente no deberá convertirse automáticamente en atributo confiable.

---

# 49. ReBAC

Relationship-Based Access Control modelará relaciones como:

```text
owns
member_of
manager_of
viewer_of
editor_of
shared_with
parent_of
belongs_to
assigned_to
```

---

# 50. Relationship Graph

```text
User
  │
member_of
  ↓
Team
  │
editor_of
  ↓
Workspace
  │
contains
  ↓
Document
```

podrá generar autoridad si existe un path registrado.

---

# 51. No Arbitrary Graph Traversal

Solo relaciones y paths registrados podrán producir autoridad.

---

# 52. Bounded ReBAC

Toda evaluación tendrá:

```text
max depth
max nodes
max edges
max paths
cycle detection
```

---

# 53. Ownership

Ownership será explícito.

```text
User
→ owns
→ Document
```

---

# 54. Ownership no implica universal authority

El hecho de ser Owner puede alimentar una Policy.

No significa automáticamente:

```text
every possible operation allowed
```

---

# 55. Sharing

Un recurso podrá compartirse mediante:

```text
User
Team
Organization
Workspace Role
Public
```

---

# 56. Share Levels

Ejemplo:

```text
viewer
commenter
editor
manager
```

---

# 57. Share Authority

Una Share será:

```text
authority source
```

pero seguirá sujeta a:

```text
Tenant
Policy
Security
Risk
Resource State
```

---

# 58. ACL

VoltStack podrá representar sharing mediante ACLs.

Pero ACL será:

```text
storage representation
```

no otro motor paralelo.

---

# 59. Policy System

Policies contendrán reglas de dominio orientadas al Resource.

Ejemplo:

```php
final class DocumentPolicy
{
    public function update(
        PrincipalInterface $principal,
        DocumentAuthorizationView $document,
        AuthorizationContext $context,
    ): AuthorizationDecision
    {
        // domain authorization rule
    }
}
```

---

# 60. Policy Registry

```text
Subject Type
+
Ability
→
Policy Handler
```

preferentemente compilado.

---

# 61. Gate System

Gate ofrecerá una API ergonómica para abilities no necesariamente ligadas a un modelo.

Ejemplo:

```php
Gate::authorize('dashboard.access');
```

---

# 62. Gate y Policy

No serán motores independientes.

Ambos convergerán hacia:

```text
AuthorizationManager
```

---

# 63. Voters / Evaluators

Para decisiones complejas:

```text
RBAC evaluator
Policy evaluator
Tenant evaluator
Scope evaluator
Relationship evaluator
Risk evaluator
Approval evaluator
```

podrán producir Candidate Decisions.

---

# 64. Candidate Decision

```php
final readonly class AuthorizationCandidateDecision
{
    public function __construct(
        public AuthorizationDecisionOutcome $outcome,
        public string $source,
        public array $reasons = [],
    ) {}
}
```

---

# 65. Decision Outcomes

El resultado final soportará:

```php
enum AuthorizationDecisionOutcome: string
{
    case Allow = 'allow';
    case Deny = 'deny';
    case Abstain = 'abstain';
    case Challenge = 'challenge';
    case Failure = 'failure';
}
```

---

# 66. ALLOW

La operación puede continuar.

---

# 67. DENY

La autoridad no permite la operación.

---

# 68. ABSTAIN

Un evaluator no tiene opinión suficiente.

No significa:

```text
ALLOW
```

---

# 69. CHALLENGE

La autoridad potencial existe, pero falta una condición recuperable.

Ejemplos:

```text
MFA
fresh authentication
approval
confirmation
device verification
```

---

# 70. FAILURE

No fue posible producir una decisión confiable.

Ejemplos:

```text
mandatory provider unavailable
corrupt state
consistency failure
invalid configuration
```

---

# 71. Challenge Architecture

Ejemplo:

```text
payment.execute
      ↓
authority exists
      ↓
Strong Assurance required
      ↓
current assurance insufficient
      ↓
CHALLENGE
```

---

# 72. Step-Up Flow

```text
Authorization
    ↓
CHALLENGE
    ↓
Authentication Step-Up
    ↓
New Authentication Context
    ↓
RE-AUTHORIZE
```

---

# 73. Completing Challenge Is Not Grant

Nunca:

```text
MFA passed
=
automatic authorization
```

Se deberá reautorizar.

---

# 74. Risk-Based Authorization

Risk podrá:

```text
tighten requirements
require challenge
deny
```

pero nunca:

```text
create missing authority
```

---

# 75. Risk Principle

```text
Low Risk
≠
Permission
```

---

# 76. Risk Levels

```text
Unknown
Low
Medium
High
Critical
```

---

# 77. Unknown Risk

No deberá interpretarse automáticamente como Low.

---

# 78. Assurance

VoltStack preferirá requerimientos como:

```text
Strong Authentication Assurance
```

en lugar de acoplar Authorization a:

```text
TOTP
SMS
Passkey
```

Authentication decide cómo satisfacer el nivel.

---

# 79. Delegation

Delegation permite:

```text
Principal B
acts as B
using authority delegated by A
```

---

# 80. Delegation != Impersonation

Delegation conserva:

```text
Effective Principal = B
```

mientras la fuente de autoridad puede ser A.

---

# 81. Delegation Grant

Debe ser:

```text
scoped
expiring
revocable
auditable
optionally single-use
```

---

# 82. Delegation Ceiling

B nunca podrá obtener mediante delegation más autoridad que:

```text
Grantor authority
∩
Delegated scope
∩
Current security constraints
```

---

# 83. Transitive Delegation

Deshabilitada por defecto.

---

# 84. Impersonation

Impersonation permite:

```text
Actor B
acts as
Effective Principal A
```

---

# 85. Prefer Delegation

Cuando el caso de uso pueda resolverse con Delegation, será preferible a Impersonation.

---

# 86. Impersonation Restrictions

Puede prohibirse:

```text
password change
API key creation
role grants
billing changes
tenant deletion
security settings
```

---

# 87. Nested Impersonation

Prohibida por defecto.

---

# 88. Capability System

Una Capability representa autoridad limitada y portable.

Puede ser:

```text
Reference
Bearer
Bound
Signed
Single Use
Delegated
```

---

# 89. Signed URL

Será tratada como una Capability.

---

# 90. Capability Is Not Bypass

La Capability entra como:

```text
authority source
```

dentro del pipeline normal.

---

# 91. Capability Claims

Puede incluir:

```text
issuer
grantee
ability
tenant
resource
audience
issued_at
expires_at
nonce
```

---

# 92. Audience Binding

Será obligatorio cuando aplique.

---

# 93. Single-Use Capabilities

Deberán consumirse atómicamente.

---

# 94. Service-to-Service

Cada servicio tendrá:

```text
ServicePrincipal
```

---

# 95. Internal Network Rule

```text
Internal Network
≠
Authorization
```

---

# 96. Service Authentication

Authentication prueba:

```text
which service
```

Authorization decide:

```text
what service may do
```

---

# 97. Service Delegation

Una llamada entre servicios podrá transmitir:

```text
original principal
actor
delegated abilities
tenant
subject constraints
audience
expiry
nonce
correlation ID
```

---

# 98. Nunca transmitir

```text
authorized = true
```

como prueba suficiente.

---

# 99. Receiver Rule

El servicio receptor:

```text
verifies envelope
+
authorizes locally
```

---

# 100. Authority Narrows Across Hops

```text
A
↓
Service 1
↓
Service 2
↓
Service 3
```

nunca deberá ampliar autoridad silenciosamente.

---

# 101. API Keys and PATs

Los scopes de tokens serán:

```text
authority ceilings
```

no permisos por sí mismos.

---

# 102. Effective Token Authority

```text
Principal Authority
∩
Token Scope
```

---

# 103. Approval Workflows

Operaciones sensibles podrán requerir aprobación independiente.

---

# 104. Fundamental Approval Rule

```text
Can Request
≠
Can Approve
≠
Can Execute
```

---

# 105. Approval Strategies

```text
Any
All
Sequential
Parallel
Threshold
Hierarchical
Custom
```

---

# 106. Maker-Checker

```text
Maker
≠
Checker
```

cuando la Policy lo requiera.

---

# 107. Dual Control

Puede exigir:

```text
2 distinct Principals
```

y para operaciones más estrictas:

```text
2 distinct Actors
```

---

# 108. Impersonation Protection

Un Actor no deberá poder:

```text
impersonate User A
approve

impersonate User B
approve
```

y satisfacer falsamente Four-Eyes.

---

# 109. Separation of Duties

VoltStack soportará:

```text
Static SoD
Dynamic SoD
```

---

# 110. Static SoD

Evita asignaciones incompatibles.

Ejemplo:

```text
PaymentCreator
+
PaymentAuditor
```

---

# 111. Dynamic SoD

Permite roles pero restringe su combinación sobre una operación.

Ejemplo:

```text
cannot approve own payment
```

---

# 112. Approved != Executed

Después de Approval:

```text
RE-AUTHORIZE
```

antes del efecto crítico.

---

# 113. Approval Proof

Deberá estar ligada a:

```text
operation
resource
resource version
tenant
scope
workflow version
```

---

# 114. Operation Fingerprint

Operaciones sensibles tendrán un fingerprint canónico.

Ejemplo:

```text
payment ID
amount
currency
beneficiary
resource version
```

---

# 115. Modification Invalidates Approval

Si cambia:

```text
amount
beneficiary
resource version
```

el Approval Proof puede quedar inválido.

---

# 116. Authorization State Consistency

La corrección no termina al producir:

```text
ALLOW
```

El estado puede cambiar antes del efecto.

---

# 117. TOCTOU

Ejemplo:

```text
10:00:00 authorize delete → ALLOW
10:00:01 role revoked
10:00:02 delete executes
```

Debe existir estrategia para operaciones críticas.

---

# 118. Authorization State Versions

VoltStack podrá versionar:

```text
Principal Authorization State
Tenant Authorization State
Scope State
Resource Authorization State
Relationship State
Delegation State
Capability Revocation State
Policy Generation
Approval State
```

---

# 119. Version Vector

Conceptualmente:

```php
final readonly class AuthorizationVersionVector
{
    public function __construct(
        public array $versions,
    ) {}
}
```

---

# 120. Consistency Profiles

```text
Eventual
Bounded Staleness
Fresh
Strong At Execution
Strict
```

---

# 121. Ability Determines Consistency

Ejemplo:

```text
article.view
→ Bounded Staleness

payment.execute
→ Strong At Execution
```

---

# 122. Pre-Execution Checkpoint

Operaciones críticas deberán poder hacer:

```text
authorize
↓
prepare operation
↓
pre-execution checkpoint
↓
effect
```

---

# 123. Execution Checkpoint

Revalida al menos los elementos críticos:

```text
principal state
tenant state
authority version
resource version
revocation
approval proof
risk when required
```

---

# 124. Atomic Claims

Para:

```text
single-use capability
approval proof
nonce
execution claim
delegated single-use authority
```

exactamente un worker deberá poder consumirlos.

---

# 125. Optimistic Concurrency

Preferencia general:

```text
expected version
+
compare-and-swap
```

---

# 126. Pessimistic Locks

Solo cuando sea necesario.

Nunca mantener locks durante:

```text
user interaction
remote network request
long approval workflow
```

---

# 127. Distributed Coordination

Authorization deberá funcionar bajo:

```text
multiple workers
multiple application nodes
queues
microservices
multiple regions
```

---

# 128. Cache Invalidation

Cambios de seguridad producirán:

```text
version bump
and/or
invalidation event
```

---

# 129. TTL Alone Is Insufficient

Para revocaciones críticas:

```text
wait until cache expires
```

no será suficiente.

---

# 130. Transactional Outbox

Mutaciones críticas podrán usar:

```text
DB Transaction
├── authority mutation
└── outbox event
```

para evitar dual-write gaps.

---

# 131. Event Delivery

Consumers deberán soportar:

```text
duplicate events
out-of-order events
```

mediante versiones.

---

# 132. Policy Generation

Cada manifest compilado tendrá:

```text
generation
fingerprint
schema version
```

---

# 133. Request Generation Pinning

Una evaluación deberá ejecutarse contra una generación coherente.

---

# 134. Persistence Architecture

Authorization Core no dependerá de:

```text
Eloquent
Doctrine
specific SQL database
graph database
Redis
```

---

# 135. Persistence Contracts

Se usarán interfaces como:

```text
RoleAssignmentRepository
PermissionGrantRepository
RelationshipStore
ShareRepository
DelegationRepository
CapabilityStore
ApprovalRepository
AuthorizationVersionStore
```

---

# 136. Canonical vs Projection

Debe distinguirse:

```text
Canonical Security State
```

de:

```text
Performance Projection
```

---

# 137. Cache Is Not Canonical Authority

Regla:

> **La pérdida de un cache no debe destruir el modelo de seguridad.**

---

# 138. Append-Only Security History

Datos como:

```text
approval decisions
security audit records
delegation lifecycle
capability consumption
```

deberán favorecer modelos append-only cuando sea apropiado.

---

# 139. Secrets

Bearer capability secrets y equivalentes:

```text
must not be stored in plaintext
```

cuando sea posible verificar mediante hash/reference.

---

# 140. Tenant Storage Boundary

Toda persistencia multi-tenant deberá preservar:

```text
tenant_id
```

o aislamiento equivalente.

---

# 141. Administration

El sistema tendrá un Control Plane separado del Data Plane.

---

# 142. Data Plane

```text
Can Principal perform Ability?
```

Debe ser:

```text
small
predictable
fast
```

---

# 143. Control Plane

Incluye:

```text
role management
permission management
policy inspection
relationship administration
delegation administration
capability inventory
approval management
diagnostics
audit exploration
```

---

# 144. Administrative Operations Are Authorized

Nunca:

```text
/admin
=
trusted
```

---

# 145. Meta-Authorization

Modificar Authorization requiere Authorization.

Ejemplo:

```text
role.assign
role.revoke
permission.grant
policy.manage
delegation.issue
capability.issue
authorization.inspect
```

---

# 146. Privilege Escalation Prevention

Para asignar autoridad deberá validarse:

```text
Actor authority
Target authority
Scope
Delegatability
Role constraints
Security floor
```

---

# 147. Effective Authority Inspection

Tooling podrá responder:

```text
Why can Alice update Document 123?
```

---

# 148. Explainability

Resultado:

```text
ALLOW

Sources:
- Workspace Editor role
- Active workspace membership
- Document belongs to workspace

Constraints:
- Same tenant
- Resource active

Policy:
- DocumentPolicy::update
```

---

# 149. Explainability Security

No revelar a usuarios finales:

```text
hidden relationships
internal role names
security rules
risk telemetry
secret tenant metadata
```

---

# 150. Explanation Audiences

```text
Public
Developer
Operator
Security
Audit
```

---

# 151. Lifecycle Architecture

Toda evaluación seguirá un lifecycle formal.

---

# 152. Final Lifecycle

```text
Invocation
   ↓
Request Construction
   ↓
Request Validation
   ↓
Principal / Actor Resolution
   ↓
Tenant Resolution
   ↓
Scope Resolution
   ↓
Authority Mode Resolution
   ↓
Compiled Plan Resolution
   ↓
Cache Candidate
   ↓
Preconditions
   ↓
Structural Authority
   ↓
Policy Evaluation
   ↓
Contextual Evaluation
   ↓
Risk
   ↓
Approval / SoD
   ↓
Mandatory Security Tail
   ↓
Aggregation
   ↓
Normalization
   ↓
Explanation
   ↓
Audit / Telemetry
   ↓
Cache Store
   ↓
Completion
   ↓
Cleanup
```

---

# 153. Mandatory Security Tail

Será una de las garantías centrales.

Incluso cuando exista:

```text
cached ALLOW
role ALLOW
policy ALLOW
capability ALLOW
```

las restricciones críticas podrán ejecutarse al final.

---

# 154. Security Tail Examples

```text
Tenant suspended
Principal suspended
Capability revoked
Impersonation forbidden
Critical security epoch
Mandatory SoD restriction
```

---

# 155. Events

Events serán:

```text
immutable facts
```

---

# 156. Event Example

```text
AuthorizationGranted
AuthorizationDenied
AuthorizationChallenged
AuthorizationFailed
```

---

# 157. Events Do Not Grant Authority

Un listener no podrá convertir:

```text
DENY
→
ALLOW
```

---

# 158. Hooks

Hooks participarán mediante contratos restringidos.

---

# 159. Hook Monotonicity

Extensiones normales podrán:

```text
ALLOW → CHALLENGE
ALLOW → DENY
CHALLENGE → DENY
```

pero no:

```text
DENY → ALLOW
```

---

# 160. Extension Capabilities

```text
Observe
Context Enrichment
Restriction
Evaluator
Challenge
Explanation
Telemetry
```

---

# 161. Plugin Trust Levels

```text
Observational
Restricted
Trusted
Core
```

---

# 162. Protected Invariants

Ni plugins Trusted podrán eliminar arbitrariamente:

```text
tenant isolation
principal state checks
mandatory security floor
protected revocation checks
```

---

# 163. Configuration

Authorization deberá utilizar configuración tipada e inmutable.

---

# 164. Configuration Pipeline

```text
Defaults
   ↓
Package Configuration
   ↓
Application Configuration
   ↓
Environment
   ↓
Tenant Profile
   ↓
Security Floor
   ↓
Typed AuthorizationConfig
```

---

# 165. Lower Layers Cannot Weaken Security Floor

Un tenant no podrá configurar:

```text
critical_mfa = false
```

si Platform exige:

```text
critical_mfa = true
```

---

# 166. Bootstrap

```text
Configure
↓
Register
↓
Discover
↓
Compile
↓
Boot
↓
Ready
```

---

# 167. No Authorization Before READY

Intentarlo será error de lifecycle.

---

# 168. Registries

Seguirán:

```text
Register
↓
Compile
↓
Freeze
↓
Read Only Runtime
```

---

# 169. Container Lifetimes

## Application Singleton

Adecuado para:

```text
immutable config
compiled manifest
ability registry
policy registry
stateless evaluators
```

## Request Scoped

Adecuado para:

```text
Principal Context
Tenant Context
Scope Context
Memoization
Evaluation Stack
Risk Context
```

---

# 170. Preferred Manager Architecture

```text
Stateless AuthorizationManager
+
Request-Scoped AuthorizationContext
```

---

# 171. Public Authorization Manager

```php
interface AuthorizationManagerInterface
{
    public function decide(
        AuthorizationRequest $request,
    ): AuthorizationDecision;

    public function authorize(
        AuthorizationRequest $request,
    ): void;
}
```

---

# 172. Convenience API

```php
$authorization->authorize(
    ability: 'document.update',
    subject: $document,
);
```

---

# 173. Boolean API

```php
if ($authorization->allows('document.update', $document)) {
    // ...
}
```

---

# 174. Detailed Decision API

```php
$decision = $authorization->decide(
    ability: 'document.update',
    subject: $document,
);
```

---

# 175. Decision Object

```php
final readonly class AuthorizationDecision
{
    public function __construct(
        public AuthorizationDecisionOutcome $outcome,
        public array $reasons,
        public array $authoritySources,
        public array $challenges,
        public AuthorizationDecisionMetadata $metadata,
    ) {}
}
```

---

# 176. Decision Metadata

Puede incluir:

```text
evaluation ID
policy generation
state version vector
evaluated at
valid until
consistency profile
cacheability
```

---

# 177. `authorize()` Semantics

```text
ALLOW
→ return

DENY
→ throw AuthorizationDeniedException

CHALLENGE
→ throw AuthorizationChallengeException

FAILURE
→ throw AuthorizationEvaluationException
```

---

# 178. Facade

VoltStack podrá ofrecer:

```php
Authorization::authorize(
    'document.update',
    $document
);
```

---

# 179. Facade Rule

Facade no contiene estado.

Solo delega al mismo:

```text
AuthorizationManager
```

---

# 180. Helpers

Podrán existir:

```php
authorize('document.update', $document);

can('document.update', $document);

cannot('document.update', $document);
```

---

# 181. One Engine Rule

Todos:

```text
Facade
Helper
Controller
Middleware
Gate
Policy
Route
CLI
Queue
```

deberán converger hacia el mismo motor.

---

# 182. No Parallel Authorization Engines

Nunca:

```text
Controller authorization
≠
Gate authorization
≠
Policy authorization
```

---

# 183. Routing Integration

Routing podrá declarar:

```text
ability
subject binding
tenant
scope
```

---

# 184. Route Example

Conceptualmente:

```php
Route::put('/documents/{document}', ...)
    ->authorize('document.update', 'document');
```

---

# 185. Route Binding Security

Debe validar:

```text
resource belongs to tenant/scope
```

antes o durante Authorization.

---

# 186. Controller Integration

```php
#[Authorize(
    ability: 'document.update',
    subject: 'document'
)]
public function update(Document $document)
{
}
```

---

# 187. Metadata Compilation

Controller attributes deberán compilarse.

No reflection por request en producción.

---

# 188. Middleware Integration

Middleware podrá exigir:

```text
authentication
tenant context
authorization
```

pero no implementará Policies.

---

# 189. HTTP Integration

Authorization producirá semántica.

HTTP layer traduce.

Ejemplo:

```text
DENY
→ 403

Unauthenticated
→ Authentication subsystem / 401

CHALLENGE
→ framework-specific challenge response

FAILURE
→ safe 5xx/security response
```

---

# 190. No HTTP Coupling in Core

Authorization Core no devolverá:

```text
JsonResponse
RedirectResponse
```

---

# 191. Database Integration

Authorization usará:

```text
Repository Contracts
Query Scope Contracts
Transaction Contracts
```

---

# 192. Query-Level Authorization

Para colecciones:

```text
authorized resource query
```

es preferible a:

```text
load everything
→ authorize each
```

cuando sea traducible.

---

# 193. Exact Query Translation

Puede ejecutarse directamente.

---

# 194. Partial Translation

Debe:

```text
fetch candidate superset
→ post-authorize
```

---

# 195. Never Trust Partial Translation Alone

---

# 196. Cache Integration

Cache podrá almacenar:

```text
compiled metadata
role projections
scope projections
decision candidates
```

con:

```text
versioning
TTL
generation
invalidation
```

---

# 197. Cache Rule

```text
cache hit
≠
security bypass
```

---

# 198. Event System Integration

Authorization usará el Event System general de VoltStack.

No creará un Event Bus paralelo innecesario.

---

# 199. Telemetry Integration

Deberá emitir:

```text
structured logs
metrics
traces
audit signals
```

---

# 200. Logging vs Audit

No son equivalentes.

```text
Application Log
≠
Security Audit Trail
```

---

# 201. Audit

Audit deberá preservar:

```text
Principal
Actor
Ability
Tenant
Scope
Subject
Outcome
Authority Source
Policy Version
Timestamp
Evaluation ID
```

según sensibilidad.

---

# 202. Sensitive Data Minimization

No registrar:

```text
passwords
tokens
capability secrets
raw MFA proof
full confidential resource payload
```

---

# 203. Queue Integration

Un Job no deberá serializar:

```text
AuthorizationDecision(ALLOW)
```

y confiar en él horas después.

---

# 204. Queue Rule

Job deberá transportar:

```text
Principal Reference
Tenant
Authority Reference
Delegated Grant if needed
Operation Descriptor
Relevant Versions
```

---

# 205. Queue Execution

```text
Job received
↓
reconstruct minimal context
↓
validate authority
↓
authorize again
↓
execute
```

---

# 206. Scheduler

Scheduler no será superuser implícito.

Usará:

```text
SystemPrincipal
or
ServicePrincipal
```

con authority explícita.

---

# 207. CLI

CLI tampoco implica:

```text
ALLOW EVERYTHING
```

---

# 208. CLI Administrative Commands

Podrán requerir:

```text
operator identity
explicit capability
system authority
break-glass workflow
```

según deployment.

---

# 209. WebSocket / Persistent Connection

No asumir que autorización inicial dura toda la conexión.

Operaciones sensibles deberán reautorizarse.

---

# 210. FrankenPHP Runtime

Arquitectura:

```text
Worker Boot
    ↓
Load Compiled Authorization Manifest
    ↓
Freeze Registries
    ↓
Request A
    ↓
Create Authorization Request Scope
    ↓
Evaluate
    ↓
Cleanup / Reset
    ↓
Request B
    ↓
Fresh Authorization Request Scope
```

---

# 211. Request Cleanup

Siempre:

```php
try {
    // request
} finally {
    $authorizationRuntime->reset();
}
```

conceptualmente.

---

# 212. Persistent State Allowed

```text
Compiled manifest
Immutable registry
Static policy metadata
Bounded version-aware worker cache
```

---

# 213. Persistent State Forbidden

```text
Current Principal
Current Actor
Current Tenant
Current Scope
Current Risk
Current Delegation
Current Impersonation
Request Decision Cache
Evaluation Stack
```

---

# 214. Compilation Architecture

VoltStack deberá mover al build/bootstrap:

```text
Policy discovery
Ability discovery
Attribute discovery
Evaluator sorting
Dependency graph
Context requirements
Approval plans
Relationship plans
Controller metadata
Route metadata
Cacheability metadata
```

---

# 215. Compiled Manifest

Será el principal artefacto runtime.

---

# 216. Compilation Principle

```text
DISCOVER ONCE
VALIDATE ONCE
COMPILE ONCE
EXECUTE MANY TIMES
```

---

# 217. Planner

AuthorizationPlanner seleccionará el mínimo trabajo requerido.

---

# 218. Example

```text
profile.view
```

no deberá ejecutar automáticamente:

```text
ReBAC
Risk
Approval
Delegation
```

---

# 219. Complex Ability

```text
payment.execute
```

sí puede requerir:

```text
RBAC
Policy
Assurance
Risk
Approval
SoD
Consistency Checkpoint
```

---

# 220. Performance Principle

> **No todas las Abilities deben pagar el coste de todas las capacidades del sistema.**

---

# 221. Request Memoization

Dentro de un request podrán reutilizarse:

```text
roles
memberships
scope ancestry
ownership
policy resolution
risk assessment when valid
```

---

# 222. Batch Authorization

API conceptual:

```php
$decisions = $authorization->decideMany(
    $requests
);
```

---

# 223. Batch Invariant

Debe producir resultados semánticamente equivalentes a decisiones individuales.

---

# 224. Resource Governance

Toda operación potencialmente costosa deberá ser bounded.

---

# 225. Bounded Resources

```text
Graph Depth
Graph Nodes
Nested Authorizations
Provider Calls
Database Queries
Batch Size
Explanation Size
Runtime Deadline
Worker Cache Size
```

---

# 226. Budget Exhaustion

Nunca:

```text
budget exceeded
→ ALLOW
```

---

# 227. Security over Availability

En incertidumbre crítica:

```text
FAILURE
or
DENY
```

según contrato.

---

# 228. Testing Architecture

Authorization requerirá múltiples niveles.

---

# 229. Unit Tests

Para:

```text
Policies
Evaluators
Resolvers
Strategies
Condition Nodes
```

---

# 230. Integration Tests

Para:

```text
RBAC + Policy
Tenant + Scope
ReBAC
Delegation
Capabilities
Approval
Risk
Persistence
Cache
```

---

# 231. Contract Tests

Todo adapter deberá pasar suites comunes.

Ejemplo:

```text
RelationshipStore contract
RoleRepository contract
CapabilityStore contract
VersionStore contract
```

---

# 232. Security Tests

Casos obligatorios:

```text
cross-tenant access
impersonation abuse
delegation escalation
capability replay
stale cache
approval replay
SoD bypass
actor laundering
scope confusion
```

---

# 233. Concurrency Tests

Ejemplo:

```text
two workers consume same proof
```

Resultado obligatorio:

```text
exactly one success
```

---

# 234. Distributed Tests

```text
Node A revokes role
Node B has stale cache
```

Critical operation deberá detectar la revocación según su consistency contract.

---

# 235. Persistent Worker Tests

```text
Request A
Principal Alice
Tenant A

Request B
Principal Bob
Tenant B
```

B nunca deberá observar estado de A.

---

# 236. Property-Based Security Tests

Propiedades importantes:

```text
Adding restriction cannot increase authority.

Increasing risk cannot increase authority.

Expired capability cannot restore authority.

Revocation cannot increase authority.

Tenant mismatch cannot become ALLOW through Policy.

Observational hook cannot change decision.

Batch optimization cannot change semantics.
```

---

# 237. Mutation Testing

Muy recomendable para Policies y evaluadores críticos.

Debe detectar cambios como:

```text
&& → ||
=== → !==
DENY → ALLOW
```

---

# 238. Differential Testing

Comparar:

```text
Reference Authorization Engine
vs
Optimized Compiled Engine
```

---

# 239. Required Property

```text
Reference Decision
=
Optimized Decision
```

---

# 240. Security Assurance

Authorization deberá mantener un catálogo de invariantes verificables.

---

# 241. Core Security Axiom 1

```text
No authority without identifiable source.
```

---

# 242. Core Security Axiom 2

```text
Authentication does not imply authorization.
```

---

# 243. Core Security Axiom 3

```text
Tenant isolation cannot be bypassed by ordinary application policy.
```

---

# 244. Core Security Axiom 4

```text
Delegated authority cannot exceed its source.
```

---

# 245. Core Security Axiom 5

```text
Capability authority cannot exceed capability scope.
```

---

# 246. Core Security Axiom 6

```text
Impersonation never erases Actor identity.
```

---

# 247. Core Security Axiom 7

```text
Approval never creates missing authority.
```

---

# 248. Core Security Axiom 8

```text
Risk cannot create authority.
```

---

# 249. Core Security Axiom 9

```text
Cache cannot create authority.
```

---

# 250. Core Security Axiom 10

```text
Plugin cannot silently remove protected security constraints.
```

---

# 251. Core Security Axiom 11

```text
Internal execution does not imply privilege.
```

---

# 252. Core Security Axiom 12

```text
Authorization state never leaks between requests.
```

---

# 253. Core Security Axiom 13

```text
A stale positive decision cannot override a required fresh revocation check.
```

---

# 254. Core Security Axiom 14

```text
Single-use authority succeeds at most once.
```

---

# 255. Core Security Axiom 15

```text
Every final ALLOW has explainable provenance.
```

---

# 256. Compliance Architecture

Authorization deberá poder soportar controles asociados con:

```text
least privilege
separation of duties
dual control
access review
privileged access monitoring
auditability
revocation
traceability
data minimization
```

sin acoplar el Core a una regulación específica.

---

# 257. Compliance Profiles

Plugins o application layers podrán implementar:

```text
SOX-oriented controls
PCI-oriented controls
HIPAA-oriented controls
ISO-oriented controls
internal corporate controls
```

---

# 258. Compliance Does Not Bypass Core

Un Compliance plugin solo podrá:

```text
restrict
challenge
observe
audit
```

según capability.

---

# 259. Access Reviews

Control Plane podrá generar:

```text
Who has access?
Why?
Through which source?
Since when?
Until when?
```

---

# 260. Certification

Puede existir workflow:

```text
Manager
reviews
effective authority
```

pero pertenece al Control Plane.

---

# 261. Break-Glass

Emergency access será explícito.

Nunca:

```text
hidden superadmin
```

---

# 262. Break-Glass Requirements

Puede exigir:

```text
special ability
strong authentication
reason
time limit
restricted scope
mandatory audit
notification
post-event review
```

---

# 263. Break-Glass Is Authority Mode

Debe aparecer en:

```text
Decision provenance
Audit
Telemetry
```

---

# 264. No Omnipotent Root Flag

Evitar:

```php
if ($user->isRoot()) {
    return true;
}
```

como arquitectura general.

---

# 265. Authority Sources Finales

VoltStack reconocerá fuentes como:

```text
Direct Permission
Scoped Role
Ownership
Share
Relationship
Delegation
Capability
Service Identity
System Authority
Break-Glass
```

---

# 266. Authority Source Registry

```php
enum AuthorizationAuthoritySource: string
{
    case Direct = 'direct';
    case Role = 'role';
    case Permission = 'permission';
    case Ownership = 'ownership';
    case Share = 'share';
    case Relationship = 'relationship';
    case Delegation = 'delegation';
    case Capability = 'capability';
    case ServiceIdentity = 'service_identity';
    case System = 'system';
    case BreakGlass = 'break_glass';
}
```

---

# 267. Authority Source Is Provenance

No significa que todas las fuentes tengan igual prioridad.

Decision Strategy determina combinación.

---

# 268. Authority Ceiling

Toda evaluación puede estar limitada por:

```text
Token Scope
Delegation Scope
Capability Scope
Tenant
Authorization Scope
Impersonation Restriction
Service Contract
```

---

# 269. Effective Authority Formula

Conceptualmente:

```text
EFFECTIVE AUTHORITY
=
AVAILABLE AUTHORITY
∩
AUTHORITY CEILINGS
∩
TENANT BOUNDARY
∩
SCOPE BOUNDARY
∩
RESOURCE POLICY
∩
SECURITY CONSTRAINTS
```

---

# 270. Contextual Formula

Después:

```text
EXERCISABLE AUTHORITY
=
EFFECTIVE AUTHORITY
∩
CURRENT CONTEXT
∩
ASSURANCE REQUIREMENTS
∩
RISK REQUIREMENTS
∩
APPROVAL / SoD
```

---

# 271. Executable Authority Formula

Finalmente:

```text
EXECUTABLE AUTHORITY
=
EXERCISABLE AUTHORITY
∩
CURRENT STATE VERSION
∩
REVOCATION FRESHNESS
∩
EXECUTION CHECKPOINT
```

---

# 272. Final Mathematical Model

```text
Principal
×
Ability
×
Subject
×
Tenant
×
Scope
×
Authority Sources
×
Context
×
Risk
×
Approval
×
State Version
×
Security Constraints
──────────────────────────
Authorization Decision
```

---

# 273. Decision Provenance

Toda ALLOW deberá poder responder:

```text
who?
as whom?
using whose authority?
through which source?
inside which tenant?
inside which scope?
against which resource?
under which policy generation?
under which state version?
under which conditions?
```

---

# 274. Final Pipeline

```text
┌──────────────────────────────────────────────────────────────┐
│                    AUTHORIZATION REQUEST                     │
└──────────────────────────────┬───────────────────────────────┘
                               ↓
                     Request Validation
                               ↓
                  Principal / Actor Context
                               ↓
                       Tenant Boundary
                               ↓
                        Scope Boundary
                               ↓
                    Authority Mode / Ceiling
                               ↓
                    Compiled Plan Resolution
                               ↓
                   Principal State Validation
                               ↓
                ┌────────────────────────────┐
                │ Structural Authority       │
                │                            │
                │ RBAC                       │
                │ Ownership                  │
                │ Sharing                    │
                │ ReBAC                      │
                │ Delegation                 │
                │ Capability                 │
                │ Service Authority          │
                └──────────────┬─────────────┘
                               ↓
                       Resource Policy
                               ↓
                         ABAC Rules
                               ↓
                    Context Requirements
                               ↓
                   Authentication Assurance
                               ↓
                         Risk Engine
                               ↓
                     Approval / Dual Control
                               ↓
                              SoD
                               ↓
                Mandatory Security Restrictions
                               ↓
                      Decision Aggregation
                               ↓
                      Decision Normalization
                               ↓
                ┌──────────────┼──────────────┐
                ↓              ↓              ↓
              ALLOW          CHALLENGE       DENY
                │                             
                └──────────────┬──────────────┘
                               ↓
                    Consistency Metadata
                               ↓
                    Audit / Explanation
                               ↓
                        Telemetry
                               ↓
                     Execution Boundary
                               ↓
                Critical Pre-Execution Check
                               ↓
                          SIDE EFFECT
```

---

# 275. Failure Path

En cualquier fase:

```text
mandatory infrastructure unavailable
invalid security state
corrupt manifest
consistency uncertainty
provider contract violation
```

puede producir:

```text
FAILURE
```

---

# 276. Fail Closed

Para requisitos críticos:

```text
uncertain
≠
ALLOW
```

---

# 277. Final Core Components

La arquitectura tendrá como piezas centrales:

```text
AuthorizationManager
AuthorizationRequestFactory
AuthorizationContextResolver
AbilityRegistry
AuthorizationPlanner
AuthorizationPipeline
DecisionManager
DecisionNormalizer
PolicyRegistry
PolicyResolver
PolicyDispatcher
Gate
EvaluatorRegistry
TenantIsolationEvaluator
ScopeResolver
RoleResolver
RelationshipResolver
OwnershipResolver
ShareResolver
DelegationManager
CapabilityVerifier
ImpersonationManager
RiskManager
ApprovalManager
SoDEvaluator
ConsistencyCoordinator
ExecutionCheckpoint
AuthorizationAuditManager
AuthorizationExplanationManager
AuthorizationCompiler
RuntimeResetter
```

---

# 278. High-Level Dependency Direction

```text
Application
    ↓
Authorization Public API
    ↓
Authorization Manager
    ↓
Planner / Pipeline
    ↓
Domain Contracts
    ↓
Infrastructure Adapters
```

---

# 279. Dependency Rule

Core deberá depender de:

```text
interfaces
value objects
domain abstractions
```

no de infraestructura concreta.

---

# 280. Final Namespace

```text
VoltStack\Quantum\Authorization
```

---

# 281. Final Package Structure

```text
src/
└── Quantum/
    └── Authorization/
        │
        ├── Contracts/
        │   ├── AuthorizationManagerInterface.php
        │   ├── AuthorizationPipelineInterface.php
        │   ├── AuthorizationPlannerInterface.php
        │   ├── AuthorizationEvaluatorInterface.php
        │   └── AuthorizationContextResolverInterface.php
        │
        ├── Core/
        │   ├── AuthorizationManager.php
        │   ├── AuthorizationPipeline.php
        │   ├── AuthorizationPlanner.php
        │   ├── AuthorizationRequestFactory.php
        │   └── AuthorizationDecisionNormalizer.php
        │
        ├── Principal/
        │   ├── Contracts/
        │   ├── Model/
        │   ├── Human/
        │   ├── Service/
        │   ├── System/
        │   └── Anonymous/
        │
        ├── Ability/
        │   ├── Contracts/
        │   ├── Registry/
        │   ├── Metadata/
        │   └── Compilation/
        │
        ├── Decision/
        │   ├── Contracts/
        │   ├── Model/
        │   ├── Strategy/
        │   ├── Normalization/
        │   └── Reason/
        │
        ├── Policy/
        │   ├── Contracts/
        │   ├── Registry/
        │   ├── Resolver/
        │   ├── Dispatcher/
        │   ├── Metadata/
        │   └── Compilation/
        │
        ├── Gate/
        │   ├── Contracts/
        │   ├── Registry/
        │   └── Gate.php
        │
        ├── Evaluator/
        │   ├── Contracts/
        │   ├── Registry/
        │   ├── Model/
        │   └── Compilation/
        │
        ├── RBAC/
        │   ├── Role/
        │   ├── Permission/
        │   ├── Assignment/
        │   ├── Resolver/
        │   └── Persistence/
        │
        ├── ABAC/
        │   ├── Attribute/
        │   ├── Condition/
        │   ├── Expression/
        │   └── Evaluation/
        │
        ├── Relationships/
        │   ├── Ownership/
        │   ├── Sharing/
        │   ├── ReBAC/
        │   ├── Graph/
        │   ├── Hierarchy/
        │   └── Persistence/
        │
        ├── Tenant/
        │   ├── Context/
        │   ├── Isolation/
        │   ├── Profile/
        │   └── Evaluator/
        │
        ├── Scope/
        │   ├── Contracts/
        │   ├── Model/
        │   ├── Hierarchy/
        │   ├── Membership/
        │   ├── Resolver/
        │   └── Persistence/
        │
        ├── Authority/
        │   ├── Model/
        │   ├── Source/
        │   ├── Ceiling/
        │   └── Mode/
        │
        ├── Delegation/
        │   ├── Contracts/
        │   ├── Model/
        │   ├── Grant/
        │   ├── Chain/
        │   ├── Revocation/
        │   └── Persistence/
        │
        ├── Capability/
        │   ├── Contracts/
        │   ├── Model/
        │   ├── Issuance/
        │   ├── Verification/
        │   ├── Revocation/
        │   └── Persistence/
        │
        ├── Impersonation/
        │   ├── Model/
        │   ├── Session/
        │   ├── Restriction/
        │   └── Evaluation/
        │
        ├── ServiceToService/
        │   ├── Principal/
        │   ├── Envelope/
        │   ├── Verification/
        │   └── Delegation/
        │
        ├── Contextual/
        │   ├── Context/
        │   ├── Condition/
        │   ├── Assurance/
        │   ├── Challenge/
        │   ├── Device/
        │   ├── Network/
        │   └── Provider/
        │
        ├── Risk/
        │   ├── Contracts/
        │   ├── Model/
        │   ├── Provider/
        │   ├── Aggregation/
        │   └── Evaluation/
        │
        ├── Approval/
        │   ├── Contracts/
        │   ├── Model/
        │   ├── Workflow/
        │   ├── Strategy/
        │   ├── Requirements/
        │   ├── Eligibility/
        │   ├── SoD/
        │   ├── Conflict/
        │   ├── Proof/
        │   └── Persistence/
        │
        ├── Consistency/
        │   ├── Contracts/
        │   ├── Versioning/
        │   ├── Coordination/
        │   ├── Invalidation/
        │   ├── Concurrency/
        │   ├── Checkpoint/
        │   ├── Distributed/
        │   └── Events/
        │
        ├── Persistence/
        │   ├── Contracts/
        │   ├── Repository/
        │   ├── Projection/
        │   ├── Transaction/
        │   └── Storage/
        │
        ├── Lifecycle/
        │   ├── Contracts/
        │   ├── Pipeline/
        │   ├── Hooks/
        │   ├── Extensions/
        │   ├── Middleware/
        │   ├── Recursion/
        │   └── Reset/
        │
        ├── Events/
        │   ├── Lifecycle/
        │   ├── Decision/
        │   ├── Security/
        │   ├── Challenge/
        │   └── Failure/
        │
        ├── Audit/
        │   ├── Contracts/
        │   ├── Model/
        │   ├── Sink/
        │   └── Redaction/
        │
        ├── Explanation/
        │   ├── Model/
        │   ├── Builder/
        │   ├── Sanitizer/
        │   └── Audience/
        │
        ├── Telemetry/
        │   ├── Metrics/
        │   ├── Tracing/
        │   ├── Logging/
        │   └── Performance/
        │
        ├── Cache/
        │   ├── Decision/
        │   ├── Metadata/
        │   ├── Versioning/
        │   └── Invalidation/
        │
        ├── Compilation/
        │   ├── Contracts/
        │   ├── Compiler/
        │   ├── Pass/
        │   ├── Manifest/
        │   └── Validation/
        │
        ├── Performance/
        │   ├── Planning/
        │   ├── Memoization/
        │   ├── Batch/
        │   ├── Query/
        │   ├── Governance/
        │   └── Benchmark/
        │
        ├── Administration/
        │   ├── Commands/
        │   ├── Inspection/
        │   ├── Simulation/
        │   ├── Diagnostics/
        │   └── Health/
        │
        ├── Bootstrap/
        │   ├── AuthorizationServiceProvider.php
        │   ├── AuthorizationBootstrapper.php
        │   └── AuthorizationBootstrapState.php
        │
        ├── Config/
        │   ├── AuthorizationConfig.php
        │   ├── Loader/
        │   ├── Validation/
        │   └── SecurityFloor/
        │
        ├── Integration/
        │   ├── Authentication/
        │   ├── Routing/
        │   ├── Controller/
        │   ├── Http/
        │   ├── Database/
        │   ├── Queue/
        │   ├── Scheduler/
        │   ├── Cli/
        │   └── FrankenPHP/
        │
        ├── Testing/
        │   ├── Fakes/
        │   ├── Assertions/
        │   ├── Contracts/
        │   ├── Security/
        │   ├── Concurrency/
        │   └── Benchmark/
        │
        └── Exceptions/
```

---

# 282. Directory Philosophy

No todos los directorios deberán implementarse desde V1.

La estructura representa:

```text
architectural boundaries
```

no obligación de crear cientos de clases vacías.

---

# 283. V1 Minimal Core

La primera implementación funcional deberá priorizar:

```text
AuthorizationManager
AuthorizationRequest
AuthorizationDecision
AbilityRegistry
Policy System
Gate
RBAC
Tenant Isolation
Scope
Context
Basic Audit
Compilation
FrankenPHP-safe runtime
Testing
```

---

# 284. V2

Agregar:

```text
Ownership
Sharing
ReBAC
Advanced ABAC
Decision Cache
Batch Authorization
Query-Level Authorization
```

---

# 285. V3

Agregar:

```text
Delegation
Capabilities
Impersonation
Service-to-Service
```

---

# 286. V4

Agregar:

```text
Risk
Adaptive Assurance
Challenges
Device Trust
```

---

# 287. V5

Agregar:

```text
Approval
Dual Control
SoD
```

---

# 288. V6 Enterprise

Agregar:

```text
Distributed consistency
Multi-region coordination
Advanced compliance
Access reviews
External IAM
Remote PDP
Advanced administrative tooling
```

---

# 289. Implementation Priority

Dentro de cada fase:

```text
Correctness
↓
Security
↓
Testing
↓
Observability
↓
Performance
↓
Convenience API
```

---

# 290. Security Cannot Be Added Later

Especialmente:

```text
Tenant isolation
Actor preservation
Request isolation
Fail-closed behavior
Authority provenance
```

deben existir desde el diseño inicial.

---

# 291. Developer Experience Goal

El usuario común deberá poder escribir:

```php
final class DocumentPolicy
{
    public function update(
        User $user,
        Document $document,
    ): bool {
        return $document->owner_id === $user->id;
    }
}
```

sin comprender inmediatamente toda la arquitectura interna.

---

# 292. Progressive Complexity

Casos simples:

```text
simple Policy
```

Casos medianos:

```text
RBAC + Scope + Policy
```

Casos complejos:

```text
ReBAC + Delegation + Risk + Approval + SoD
```

El Core deberá soportarlos sin obligar a usar todos.

---

# 293. Laravel Ergonomics

VoltStack buscará APIs como:

```php
$user->can('document.update', $document);

Authorization::authorize(
    'document.update',
    $document
);
```

---

# 294. Symfony Rigor

Internamente mantendrá:

```text
explicit contracts
typed decisions
voter/evaluator model
dependency injection
compiled metadata
strict lifecycle
```

---

# 295. VoltStack Differentiation

VoltStack deberá ir más allá integrando nativamente:

```text
RBAC
ABAC
ReBAC
Tenant Scopes
Delegation
Capabilities
Impersonation
Risk
Approval
SoD
Distributed Consistency
Compilation
Persistent Worker Safety
```

bajo un solo sistema.

---

# 296. Framework Integration Map

```text
                    ┌─────────────────┐
                    │ Authentication  │
                    └────────┬────────┘
                             ↓
┌──────────┐          ┌──────────────┐          ┌────────────┐
│ Routing  │─────────→│Authorization │←─────────│ Controller │
└──────────┘          └──────┬───────┘          └────────────┘
                             │
          ┌──────────────────┼───────────────────┐
          ↓                  ↓                   ↓
      Database             Cache              Events
          │                  │                   │
          └──────────────────┼───────────────────┘
                             ↓
                        Telemetry
                             │
             ┌───────────────┼──────────────┐
             ↓               ↓              ↓
            Queue         Scheduler         CLI
                             │
                             ↓
                         FrankenPHP
```

---

# 297. Authentication Integration Contract

Authentication entrega:

```text
Identity
Actor
Authentication State
Assurance
Session
Credential Metadata
```

Authorization no vuelve a autenticar.

---

# 298. Database Integration Contract

Database ofrece:

```text
Persistence
Transactions
Locking
Query Builder
Read/Write Splitting
Consistency primitives
```

Authorization define qué necesita.

---

# 299. Cache Integration Contract

Cache ofrece almacenamiento temporal.

Authorization controla:

```text
cache key semantics
versions
TTL
invalidation
security rules
```

---

# 300. Telemetry Integration Contract

Telemetry recibe:

```text
logs
metrics
traces
```

sin decidir autoridad.

---

# 301. Event Integration Contract

Event System transporta hechos.

No reemplaza:

```text
Authorization Pipeline
```

---

# 302. Queue Integration Contract

Queue transporta trabajo.

No transporta autoridad infinita.

---

# 303. Configuration Integration Contract

Config define comportamiento estructural.

No se convierte en request state.

---

# 304. Container Integration Contract

Container administra dependencies/lifetimes.

No debe transformarse en almacén global de:

```text
current principal
```

---

# 305. Final Runtime Architecture

```text
APPLICATION REQUEST
        │
        ↓
AUTHENTICATION
        │
        ├── Identity
        ├── Actor
        └── Assurance
        │
        ↓
REQUEST CONTEXT
        │
        ├── Tenant
        ├── Scope
        ├── Session
        └── Operation
        │
        ↓
AUTHORIZATION REQUEST FACTORY
        │
        ↓
AUTHORIZATION MANAGER
        │
        ↓
COMPILED PLANNER
        │
        ↓
AUTHORIZATION PIPELINE
        │
        ├── Structural Authority
        ├── Policies
        ├── Context
        ├── Risk
        ├── Approval
        └── Security Tail
        │
        ↓
DECISION MANAGER
        │
        ↓
ALLOW / DENY / CHALLENGE / FAILURE
        │
        ├── Audit
        ├── Explanation
        ├── Telemetry
        └── Consistency Metadata
        │
        ↓
APPLICATION
```

---

# 306. Final Critical Operation Architecture

```text
AUTHENTICATE
     ↓
AUTHORIZE
     ↓
CHALLENGE IF REQUIRED
     ↓
RE-AUTHORIZE
     ↓
APPROVAL IF REQUIRED
     ↓
RE-AUTHORIZE
     ↓
PREPARE OPERATION
     ↓
PRE-EXECUTION CONSISTENCY CHECK
     ↓
ATOMIC CLAIM / VERSION CHECK
     ↓
EXECUTE
     ↓
AUDIT RESULT
```

---

# 307. Final Distributed Architecture

```text
Client
  ↓
Service A
  │
  ├── Authenticate
  ├── Authorize
  └── Create Delegated Authority Envelope
             ↓
          Service B
             │
             ├── Verify Envelope
             ├── Check Revocation
             ├── Resolve Current Tenant State
             ├── Authorize Locally
             └── Execute
```

---

# 308. Final Decision Principle

No decisión deberá depender únicamente de:

```text
Role
Policy
Token
Capability
Share
Relationship
Approval
Risk
```

aisladamente cuando el plan requiera otras restricciones.

---

# 309. Composition over Bypass

Cada subsistema aporta:

```text
authority
constraint
context
evidence
```

al mismo motor.

---

# 310. No Magic Admin

El framework no tendrá como arquitectura central:

```text
if admin => true
```

---

# 311. No Magic Service

Tampoco:

```text
if internal => true
```

---

# 312. No Magic Tenant Owner

Tampoco:

```text
if tenant owner => everything
```

Roles privilegiados seguirán siendo:

```text
explicit authority definitions
```

---

# 313. Default Security Posture

VoltStack Authorization deberá ser:

```text
deny by default
explicit authority
least privilege
tenant aware
scope aware
actor aware
auditable
fail closed
```

---

# 314. Secure Defaults

Por defecto:

```text
transitive delegation = disabled
nested impersonation = disabled
cross-tenant relationships = disabled
self approval = disabled where independent approval required
stale critical allow = disabled
unknown ability = deny/error
unknown risk != low
runtime policy discovery in production = disabled
unbounded graph traversal = disabled
```

---

# 315. Performance Defaults

Por defecto:

```text
compiled metadata
frozen registries
request memoization
bounded worker caches
lazy context providers
batch-capable repositories
version-aware decision caching
```

---

# 316. Operational Defaults

Producción deberá favorecer:

```text
strict configuration validation
manifest verification
safe diagnostics
redacted explanations
audit
security telemetry
bounded resources
```

---

# 317. Testing Defaults

Framework packages deberán incluir:

```text
contract tests
security invariants
persistent-worker tests
concurrency tests
optimization equivalence tests
```

---

# 318. Extension Defaults

Plugins:

```text
cannot broaden authority
unless operating through an explicit Core-level authority contract
```

---

# 319. Final Failure Philosophy

Authorization deberá distinguir:

```text
Not Authorized
```

de:

```text
Could Not Safely Determine Authorization
```

---

# 320. DENY vs FAILURE

```text
DENY
=
decision reached

FAILURE
=
decision could not be safely reached
```

---

# 321. Public Error Normalization

Ambos pueden exponerse cuidadosamente para evitar information disclosure.

Internamente deberán permanecer distintos.

---

# 322. Final Audit Philosophy

Toda operación privilegiada deberá permitir reconstruir:

```text
who acted
as whom
on what
inside which tenant
inside which scope
using which authority
under which policy
with which approvals
under which context
with which result
```

---

# 323. Final Performance Philosophy

```text
Compile structural work.
Resolve only dynamic work.
Load only required data.
Cache only with correct validity.
Batch whenever semantics permit.
Bound every potentially explosive operation.
```

---

# 324. Final Distributed Philosophy

```text
Never trust an old ALLOW merely because another node produced it.

Transmit authority and provenance.

Re-authorize where the effect occurs.
```

---

# 325. Final Persistent Worker Philosophy

```text
Immutable framework state may survive.

Authorization identity state may not.
```

---

# 326. Final Extensibility Philosophy

```text
Events observe.

Hooks participate through constrained contracts.

Evaluators contribute decisions.

Policies express domain rules.

Plugins extend capabilities.

Core protects invariants.
```

---

# 327. Final Authority Philosophy

```text
Permission
does not imply
resource access.

Ownership
does not imply
unlimited permission.

Relationship
does not imply
unrestricted authority.

Approval
does not imply
permission.

Low risk
does not imply
permission.

Authentication
does not imply
authorization.
```

---

# 328. Final Security Formula

```text
IDENTITY
    +
EXPLICIT AUTHORITY
    +
TENANT ISOLATION
    +
SCOPE
    +
RESOURCE RELATIONSHIP
    +
POLICY
    +
CONTEXT
    +
ASSURANCE
    +
RISK
    +
APPROVAL / SoD
    +
SECURITY FLOOR
    +
STATE FRESHNESS
    =
AUTHORIZATION DECISION
```

---

# 329. Final Executable Authority Formula

```text
AUTHORIZATION DECISION
        +
CURRENT STATE VERSION
        +
REVOCATION FRESHNESS
        +
OPERATION FINGERPRINT
        +
PRE-EXECUTION REVALIDATION
        +
ATOMIC CLAIM
        =
CONSISTENT EXECUTABLE AUTHORITY
```

---

# 330. Architectural Definition

El Authorization System de VoltStack queda definido como:

> **Un motor unificado, compilable, extensible, multi-tenant, scope-aware, relationship-aware, context-aware, risk-aware, approval-aware y distributed-consistency-aware para determinar, explicar, verificar y hacer cumplir la autoridad efectiva de Principals humanos, servicios, sistemas y actores anónimos sobre operaciones y recursos dentro de VoltStack.**

---

# 331. Architectural Identity

VoltStack Authorization combina conceptualmente:

```text
Laravel-style developer ergonomics

+

Symfony-style explicit security architecture

+

RBAC / ABAC / ReBAC

+

Enterprise IAM concepts

+

Zero-trust service authorization

+

Approval / SoD workflows

+

Distributed authorization consistency

+

FrankenPHP persistent-worker optimization
```

sin depender arquitectónicamente de ninguno de ellos.

---

# 332. Lo que Authorization no deberá convertirse

No deberá convertirse en:

```text
Authentication System
General Workflow Engine
General Rules Engine
Database ORM
Identity Provider
SIEM
Fraud Detection Platform
HTTP Middleware Collection
Graph Database
```

---

# 333. Boundary Discipline

Puede integrarse con esos sistemas mediante:

```text
contracts
providers
adapters
events
```

manteniendo responsabilidades claras.

---

# 334. Final Architectural Invariants

La implementación de VoltStack Authorization se considerará conforme únicamente si mantiene estas propiedades:

1. Toda decisión identifica Principal y Ability.

2. Actor nunca se pierde durante impersonation/delegation.

3. Tenant Isolation se evalúa explícitamente.

4. Scope forma parte de la autoridad cuando aplica.

5. Ninguna fuente de autoridad evade automáticamente Policies.

6. Delegation nunca amplía autoridad.

7. Capability nunca amplía su propio scope.

8. Impersonation está restringida y auditada.

9. ServicePrincipal no es superuser.

10. Risk nunca crea permisos.

11. Approval nunca crea permisos.

12. SoD puede bloquear autoridad existente.

13. CHALLENGE nunca equivale a ALLOW.

14. FAILURE nunca equivale a ALLOW.

15. Cache nunca es fuente canónica de autoridad.

16. Revocaciones críticas pueden invalidar ALLOW cacheado.

17. Single-use authority es consumida atómicamente.

18. Authorization state de un request no sobrevive al siguiente.

19. Plugins ordinarios no pueden convertir DENY en ALLOW.

20. Observational events no cambian semántica.

21. Runtime usa metadata compilada cuando sea posible.

22. Operaciones recursivas tienen límites.

23. Optimización no cambia decisiones.

24. Batch authorization conserva semántica individual.

25. Toda ALLOW final posee provenance explicable.

---

# 335. Definition of Done

El Authorization System podrá considerarse arquitectónicamente implementado cuando existan:

```text
Core contracts
Principal model
Ability model
Request/context model
Decision model
AuthorizationManager
Planner
Pipeline
Policies
Gate
RBAC
Tenant Isolation
Scopes
Lifecycle
Compilation
Audit
Testing
FrankenPHP request isolation
```

y sus invariantes fundamentales estén verificadas.

---

# 336. Enterprise Definition of Done

La implementación Enterprise completa añadirá:

```text
ABAC
ReBAC
Ownership
Sharing
Delegation
Capabilities
Impersonation
Service-to-Service
Risk
Adaptive Assurance
Approval
SoD
Distributed Consistency
Advanced Administration
Compliance Tooling
Multi-region Coordination
```

---

# 337. Documentation Set Status

Con este documento queda cerrada la serie arquitectónica principal:

```text
01 → 32
```

del Authorization System.

El documento:

```text
32_AUTHORIZATION_SYSTEM_INTEGRATION_AND_FINAL_ARCHITECTURE.md
```

se convierte en:

```text
MASTER ARCHITECTURE DOCUMENT
```

del paquete.

Los documentos anteriores pasan a ser especificaciones especializadas de cada subsistema.

---

# 338. Regla para futuras extensiones

Cualquier nueva capacidad de Authorization deberá responder antes de incorporarse:

```text
Is it an Authority Source?

Is it a Constraint?

Is it Context?

Is it Evidence?

Is it a Challenge?

Is it an Evaluator?

Is it an Administrative Capability?

Is it actually Authorization?
```

Si no pertenece claramente a alguna categoría, deberá reconsiderarse si corresponde a este módulo.

---

# 339. Future Evolution

Posibles extensiones futuras:

```text
Policy-as-Code DSL
External PDP Integration
OPA-compatible adapters
Cedar-like policy adapters
SCIM integration
LDAP/Directory federation
Cloud IAM bridges
Fine-grained graph authorization services
Formal policy verification
Policy model checking
Security decision replay
Access certification
Entitlement governance
Just-in-Time privileged access
Continuous authorization
Workload identity federation
```

Estas extensiones deberán respetar los invariantes definidos aquí.

---

# 340. Regla final

> **Authorization no es simplemente preguntar si un usuario tiene un permiso. Es demostrar que un Actor concreto posee autoridad válida, limitada, vigente y explicable para realizar una operación concreta sobre un objetivo concreto, dentro de un tenant y scope concretos, bajo el contexto y estado de seguridad actuales.**

---

# 341. Filosofía final de VoltStack Authorization

```text
Explicit over implicit.

Scoped over global.

Least privilege over convenience bypasses.

Provenance over magic.

Composition over parallel security engines.

Revalidation over stale trust.

Immutable compiled state over runtime discovery.

Request isolation over ambient global state.

Bounded computation over uncontrolled flexibility.

Explainable authority over opaque decisions.

Security correctness over unsafe optimization.
```

---

# 342. Estado final

```text
┌─────────────────────────────────────────────────────────┐
│                                                         │
│            VOLTSTACK AUTHORIZATION SYSTEM               │
│                                                         │
│              ARCHITECTURE: COMPLETE                     │
│                                                         │
│  Authentication Integration              ✓              │
│  Principal / Actor Model                 ✓              │
│  Policies / Gates / Voters               ✓              │
│  RBAC / ABAC / ReBAC                     ✓              │
│  Tenant / Scope Isolation                ✓              │
│  Ownership / Sharing                     ✓              │
│  Delegation / Capabilities               ✓              │
│  Impersonation / Service-to-Service      ✓              │
│  Context / Risk / Assurance              ✓              │
│  Approval / Dual Control / SoD           ✓              │
│  Lifecycle / Hooks / Extensions          ✓              │
│  Persistence / Storage                   ✓              │
│  Concurrency / Distributed State         ✓              │
│  Administration / Operations             ✓              │
│  Testing / Security Assurance            ✓              │
│  Compilation / Performance               ✓              │
│  FrankenPHP Runtime Safety               ✓              │
│                                                         │
└─────────────────────────────────────────────────────────┘
```

---

# 343. Conclusión

Con `32_AUTHORIZATION_SYSTEM_INTEGRATION_AND_FINAL_ARCHITECTURE.md`, VoltStack dispone de una especificación completa para construir un sistema de autorización que pueda comenzar con la ergonomía de:

```php
$user->can('document.update', $document);
```

pero escalar arquitectónicamente hasta escenarios como:

```text
multi-tenant SaaS
enterprise RBAC
fine-grained ReBAC
service-to-service authorization
delegated authority
temporary capabilities
privileged impersonation
risk-adaptive authorization
dual-control operations
separation of duties
distributed revocation
multi-node deployments
persistent FrankenPHP workers
```

sin crear motores de seguridad separados para cada caso.

La decisión arquitectónica definitiva es mantener un solo núcleo:

```text
                    AUTHORIZATION CORE
                           │
          ┌────────────────┼────────────────┐
          │                │                │
      Authority        Constraints       Context
          │                │                │
          └────────────────┼────────────────┘
                           ↓
                        Policy
                           ↓
                    Security State
                           ↓
                       Decision
```

El principio rector del sistema queda resumido en:

> **Toda autoridad debe ser explícita, limitada, contextual, verificable, revocable, auditable y explicable.**

Y la regla operacional definitiva será:

```text
DO NOT TRUST AN ALLOW
BEYOND THE CONTEXT,
SCOPE,
AUTHORITY,
STATE,
AND TIME
FOR WHICH IT WAS PROVEN.
```

**Fin de la arquitectura principal del VoltStack Authorization System.**
