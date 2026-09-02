# VoltStack Authorization System

## Approval Workflow, Dual Control and Separation of Duties System

**Documento:** `24_AUTHORIZATION_APPROVAL_WORKFLOW_DUAL_CONTROL_AND_SEPARATION_OF_DUTIES_SYSTEM.md`
**Sistema:** Authorization
**Framework:** VoltStack
**Módulo sugerido:** `Quantum/Authorization`
**Estado:** Especificación arquitectónica
**Versión objetivo:** 1.x+

---

# 1. Propósito
Este documento define la arquitectura del sistema de Approval Workflow, Dual Control y Separation of Duties (SoD) del Authorization System de VoltStack.
Los documentos anteriores permiten determinar:
¿Quién es el Principal?
        ↓
¿Dentro de qué Tenant?
        ↓
¿Dentro de qué Scope?
        ↓
¿Qué Ability solicita?
        ↓
¿Sobre qué Resource?
        ↓
¿Qué Policies aplican?
        ↓
¿Qué Roles / Permissions posee?
        ↓
¿Qué Relationships existen?
        ↓
¿Qué Context existe?
        ↓
¿Qué Risk existe?
        ↓
¿Qué Assurance se requiere?
Sin embargo, existen operaciones donde:
La autorización de una sola identidad no debe ser suficiente para ejecutar una operación.

Ejemplo:
Employee#42
    ↓
payment.create
    ↓
Payment#9001
amount = $500,000
Aunque el usuario tenga:
payment.create
la organización puede exigir:
Creator
    ≠
Approver
y además:
required approvals = 2
La operación deberá quedar:
PENDING_APPROVAL
hasta satisfacer todas las condiciones.
# 2. Objetivo arquitectónico
VoltStack deberá soportar nativamente:
Approval Workflows

Maker-Checker

Four-Eyes Principle

Dual Control

N-of-M Approval

Sequential Approval

Parallel Approval

Hierarchical Approval

Conditional Approval

Risk-Based Approval

Separation of Duties

Static Separation of Duties

Dynamic Separation of Duties

Conflict of Interest

Self-Approval Prevention

Approval Delegation

Approval Expiration

Approval Revocation

Approval Evidence

Approval Proofs

Approval Audit Trail
sin introducir estas responsabilidades directamente en Controllers, Services o Models.
# 3. Principio fundamental
La arquitectura deberá distinguir:
AUTHORITY
de:
EXECUTION AUTHORITY
Un Principal puede poseer autoridad para solicitar una operación sin poseer autoridad suficiente para ejecutarla inmediatamente.
Formalmente:
CanRequest(Operation)
≠
CanExecute(Operation)
# 4. Ejemplo
Un usuario puede tener:
payment.request
pero no:
payment.execute
El flujo puede ser:
Requester
    ↓
Payment Request
    ↓
Approver A
    ↓
Approver B
    ↓
Execution Authorization
# 5. Approval no reemplaza Authorization
Approval deberá considerarse una capa adicional.
Authorization
    ↓
Approval Requirements
    ↓
Approval Workflow
    ↓
Final Authorization
No:
Approval
    ↓
bypass authorization
# 6. Regla crítica
Una aprobación nunca deberá crear autoridad inexistente.
Ejemplo:
User has no permission:
production.deploy
aunque exista:
CEO approval
el resultado seguirá siendo:
DENY
si la Policy exige que el ejecutor posea dicha Ability.
# 7. Modelo conceptual
La operación protegida será representada mediante:
Authorization Operation
        ↓
Approval Requirement
        ↓
Approval Workflow
        ↓
Approval Requests
        ↓
Approval Decisions
        ↓
Approval Evidence
        ↓
Approval Proof
        ↓
Authorization Re-evaluation
        ↓
Execution
# 8. ApprovalRequirement
Contrato conceptual:
final readonly class ApprovalRequirement
{
    public function __construct(
        public string $policy,
        public int $requiredApprovals,
        public ApprovalStrategy $strategy,
        public array $constraints = [],
    ) {}
}
# 9. ApprovalStrategy
enum ApprovalStrategy: string
{
    case Any = 'any';
    case All = 'all';
    case Sequential = 'sequential';
    case Parallel = 'parallel';
    case Threshold = 'threshold';
    case Hierarchical = 'hierarchical';
    case Custom = 'custom';
}
# 10. Any Approval
Ejemplo:
Approvers:
A
B
C

Required:
1
Cualquiera puede aprobar.
1-of-3
# 11. All Approval
Todos deben aprobar.
Approvers:
A
B
C

Required:
3
Resultado:
3-of-3
# 12. Threshold Approval
Ejemplo:
Approvers:
A
B
C
D
E

Required:
3
Modelo:
3-of-5
# 13. Sequential Approval
Los approvals deberán ejecutarse en orden.
Manager
    ↓
Finance
    ↓
Director
    ↓
Compliance
No podrá aprobar:
Director
antes de:
Finance
si el workflow exige secuencia estricta.
# 14. Parallel Approval
Ejemplo:
              ┌→ Finance ────────┐
Request ──────┼→ Compliance ─────┼→ Execute
              └→ Security ───────┘
Los approvals pueden ocurrir simultáneamente.
# 15. Hierarchical Approval
El nivel requerido puede depender de la operación.
Ejemplo:
amount <= 10,000
→ Supervisor

amount <= 100,000
→ Manager

amount <= 1,000,000
→ Director

amount > 1,000,000
→ Director + CFO
# 16. Conditional Approval
Los requirements pueden depender de:
resource attributes
operation attributes
tenant
scope
risk
principal
classification
amount
environment
# 17. Ejemplo
Payment.amount < 10,000
→ no approval

10,000 – 100,000
→ 1 approval

100,000 – 1,000,000
→ 2 approvals

> 1,000,000
→ 3 approvals + executive approval
 1. Risk-Based Approval
Integración con el documento 23:
Risk = Low
→ 1 approver

Risk = Medium
→ 2 approvers

Risk = High
→ senior approver + strong authentication

Risk = Critical
→ DENY
# 19. Maker-Checker
Uno de los patrones fundamentales será:
Maker
    ↓
creates operation
    ↓
Checker
    ↓
reviews operation
    ↓
approve / reject
# 20. Regla Maker-Checker
Por defecto:
Maker != Checker
# 21. Self-Approval Prevention
VoltStack deberá soportar explícitamente:
creator cannot approve
Contrato conceptual:
final readonly class SelfApprovalConstraint
{
    public function __construct(
        public bool $allowed = false,
    ) {}
}
# 22. Ejemplo
Payment created by User#42
User#42 posee:
payment.approve
pero la Policy establece:
self_approval = false
Resultado:
User#42
→ cannot approve Payment
# 23. Four-Eyes Principle
El principio de cuatro ojos exige al menos:
2 independent identities
para completar una operación.
# 24. Four-Eyes no significa necesariamente dos Roles
Puede existir:
User A
role=finance.manager

User B
role=finance.manager
y satisfacer:
two distinct principals
si la Policy lo permite.
# 25. Distinct Principal Constraint
final readonly class DistinctPrincipalConstraint
{
    public function __construct(
        public int $minimumDistinctPrincipals,
    ) {}
}
# 26. Dual Control
Dual Control será más estricto que simplemente solicitar dos approvals.
Puede requerir:
two independent authorized principals
para que una acción pueda ejecutarse.
Ejemplo:
cryptographic.key.activate
# 27. Dual Control Execution
Modelo:
Principal A
    ↓
authorization
    ↓
approval evidence A

Principal B
    ↓
authorization
    ↓
approval evidence B

A != B
    ↓
Dual Control satisfied
    ↓
Execute
# 28. No Shared Identity
Dos approvals realizados utilizando:
same Principal
no satisfacen Dual Control.
# 29. Actor vs Principal
El sistema deberá utilizar los conceptos definidos anteriormente:
Actor
Principal
para impedir bypass mediante impersonation.
# 30. Impersonation Rule
Ejemplo:
SupportAgent#10
impersonates
FinanceManager#20
El approval deberá conservar:
actor = SupportAgent#10
principal = FinanceManager#20
# 31. Impersonated Approval
Por defecto, operaciones críticas podrán establecer:
impersonated approvals forbidden
# 32. Razón
De otro modo:
Admin
    ↓
impersonates User A
    ↓
approve

Admin
    ↓
impersonates User B
    ↓
approve
podría aparentar:
2 distinct approvers
cuando realmente existe:
1 actor
# 33. Distinct Actor Constraint
Debe existir:
final readonly class DistinctActorConstraint
{
    public function __construct(
        public int $minimumDistinctActors,
    ) {}
}
# 34. Dual Distinctness
Una Policy crítica podrá exigir:
distinct principals >= 2
AND
distinct actors >= 2
# 35. Separation of Duties
VoltStack deberá soportar:
Separation of Duties
para impedir concentraciones peligrosas de autoridad.
# 36. Static Separation of Duties
Static SoD restringe asignaciones de Roles/Permissions.
Ejemplo:
Role:
payment.creator
es incompatible con:
payment.auditor
# 37. Static Conflict
User
    ↓
Role A
al intentar asignar:
Role B
se detecta:
Role A conflicts Role B
Resultado:
assignment denied
# 38. StaticSoDRule
interface StaticSeparationOfDutiesRuleInterface
{
    public function evaluate(
        Principal $principal,
        AuthorityAssignment $assignment
    ): SoDDecision;
}
# 39. Role Conflict Registry
interface RoleConflictRegistryInterface
{
    public function conflicts(
        string $roleA,
        string $roleB
    ): bool;
}
# 40. Example
accounts_payable.creator
X
accounts_payable.auditor
# 41. Dynamic Separation of Duties
Dynamic SoD no necesariamente impide tener ambos Roles.
Restringe su uso dentro de una operación determinada.
# 42. Ejemplo
User posee:
invoice.create
invoice.approve
pero:
cannot approve invoice created by self
# 43. Ventaja
Esto permite organizaciones pequeñas donde una persona necesita múltiples Roles pero debe mantenerse separación por transacción.
# 44. DynamicSoDRule
interface DynamicSeparationOfDutiesRuleInterface
{
    public function evaluate(
        AuthorizationRequest $request,
        ApprovalContext $context
    ): SoDDecision;
}
# 45. SoDDecision
final readonly class SoDDecision
{
    public function __construct(
        public SoDOutcome $outcome,
        public string $reason,
        public array $conflicts = [],
    ) {}
}
# 46. SoDOutcome
enum SoDOutcome: string
{
    case Satisfied = 'satisfied';
    case Conflict = 'conflict';
    case Indeterminate = 'indeterminate';
    case Failure = 'failure';
}
# 47. Conflict of Interest
VoltStack deberá permitir reglas más generales:
ConflictOfInterestRule
# 48. Ejemplos
requester cannot approve

resource owner cannot audit

vendor manager cannot approve own vendor

team member cannot approve own expense

delegator cannot count as independent approver

impersonator cannot satisfy distinct approver rule
# 49. ConflictRule
interface ApprovalConflictRuleInterface
{
    public function evaluate(
        ApprovalCandidate $candidate,
        ApprovalRequest $request
    ): ConflictDecision;
}
# 50. Approval Workflow
Un workflow representará el proceso completo.
final readonly class ApprovalWorkflowDefinition
{
    public function __construct(
        public string $id,
        public string $version,
        public array $stages,
        public array $constraints = [],
    ) {}
}
# 51. Workflow Versioning
Todo workflow deberá poseer:
id
version
# 52. Razón
Una operación iniciada con:
Workflow v3
no deberá cambiar silenciosamente porque posteriormente se publique:
Workflow v4
# 53. Workflow Snapshot
Al iniciar un proceso se deberá conservar una representación verificable de:
workflow version
approval requirements
security constraints
operation fingerprint
# 54. Approval Stage
final readonly class ApprovalStage
{
    public function__construct(
        public string $id,
        public ApprovalStrategy $strategy,
        public int $requiredApprovals,
        public array $eligibleApprovers,
        public array $constraints = [],
    ) {}
}
# 55. Ejemplo
Stage 1
Finance Manager
1 approval

Stage 2
Compliance
1 approval

Stage 3
Executive
2 approvals
# 56. Stage State
enum ApprovalStageState: string
{
    case Pending = 'pending';
    case Active = 'active';
    case Approved = 'approved';
    case Rejected = 'rejected';
    case Expired = 'expired';
    case Cancelled = 'cancelled';
    case Failed = 'failed';
}
# 57. Workflow State
enum ApprovalWorkflowState: string
{
    case Pending = 'pending';
    case InProgress = 'in_progress';
    case Approved = 'approved';
    case Rejected = 'rejected';
    case Expired = 'expired';
    case Cancelled = 'cancelled';
    case Executed = 'executed';
    case Failed = 'failed';
}
# 58. State Machine
PENDING
   ↓
IN_PROGRESS
   ├────────→ REJECTED
   ├────────→ CANCELLED
   ├────────→ EXPIRED
   │
   ↓
APPROVED
   ↓
EXECUTED
# 59. Approved no significa Executed
Distinción crítica:
APPROVED
≠
EXECUTED
# 60. Razón
Después de aprobación todavía debe realizarse:
authorization revalidation
resource validation
operation validation
# 61. Approval Request
Representa una solicitud concreta.
final readonly class ApprovalRequest
{
    public function __construct(
        public ApprovalRequestId $id,
        public string $workflowId,
        public string $workflowVersion,
        public PrincipalReference $requester,
        public ActorReference $actor,
        public AuthorizationOperationDescriptor $operation,
        public DateTimeImmutable $createdAt,
        public ?DateTimeImmutable $expiresAt,
    ) {}
}
# 62. Immutable Security Data
Los campos críticos del Approval Request deberán ser inmutables.
# 63. Mutable Business Metadata
Información no crítica podrá almacenarse separadamente.
# 64. Approval Candidate
Antes de permitir aprobar:
candidate
    ↓
eligibility evaluation
# 65. Candidate Evaluation
Deberá comprobar:
identity
authority
tenant
scope
stage
SoD
conflicts
delegation
impersonation
assurance
risk
# 66. Approval Eligibility
interface ApprovalEligibilityEvaluatorInterface
{
    public function evaluate(
        Principal $principal,
        ApprovalRequest $request,
        ApprovalStage $stage
    ): ApprovalEligibilityDecision;
}
# 67. Eligibility no debe persistirse indefinidamente
Que un usuario fuese elegible ayer no significa que siga siéndolo hoy.
# 68. Re-Evaluate on Approval
Al aprobar:
re-evaluate eligibility
# 69. Approval Ability
El approval deberá ser una Ability real.
Ejemplo:
payment.approve
No simplemente:
is listed as approver
# 70. Eligible Approver Selector
Un stage podrá definir:
role
permission
policy
relationship
organization position
team membership
specific principals
custom selector
# 71. Selector Contract
interface ApprovalCandidateSelectorInterface
{
    public function candidates(
        ApprovalRequest $request,
        ApprovalStage $stage
    ): iterable;
}
# 72. Candidate Discovery vs Authorization
Importante:
candidate discovery
≠
candidate authorization
# 73. Ejemplo
Selector encuentra:
20 Finance Managers
pero cada uno todavía debe pasar:
Authorization
SoD
Scope
Context
Risk
# 74. Approval Decision
final readonly class ApprovalDecision
{
    public function __construct(
        public ApprovalDecisionId $id,
        public ApprovalRequestId $requestId,
        public string $stageId,
        public PrincipalReference $principal,
        public ActorReference $actor,
        public ApprovalDecisionType $decision,
        public DateTimeImmutable $decidedAt,
        public ?string $reason = null,
    ) {}
}
# 75. ApprovalDecisionType
enum ApprovalDecisionType: string
{
    case Approve = 'approve';
    case Reject = 'reject';
    case Abstain = 'abstain';
}
# 76. Approval Decision Immutable
Una decisión registrada no deberá editarse.
# 77. Correcciones
Si es necesario cambiarla:
revoke
supersede
new decision
pero no modificar historial.
# 78. Append-Only History
El historial de seguridad deberá ser conceptualmente:
append-only
# 79. Approval Evidence
Cada aprobación deberá producir evidencia.
final readonly class ApprovalEvidence
{
    public function __construct(
        public ApprovalDecisionId $decision,
        public PrincipalReference $principal,
        public ActorReference $actor,
        public AuthenticationAssuranceLevel $assurance,
        public DateTimeImmutable $approvedAt,
        public string $operationFingerprint,
        public array $securityMetadata = [],
    ) {}
}
# 80. Approval Proof
Cuando el workflow se satisface podrá producir:
ApprovalProof
# 81. ApprovalProof
final readonly class ApprovalProof
{
    public function__construct(
        public ApprovalRequestId $requestId,
        public string $workflowId,
        public string $workflowVersion,
        public string $operationFingerprint,
        public array $evidence,
        public DateTimeImmutable $issuedAt,
        public DateTimeImmutable $expiresAt,
    ) {}
}
# 82. Proof Purpose
Authorization podrá recibir:
verified ApprovalProof
y comprobar:
required approval condition satisfied
# 83. Proof no concede permisos
Regla:
ApprovalProof
+
No Authority
=

DENY
# 84. Proof Verification
interface ApprovalProofVerifierInterface
{
    public function verify(
        ApprovalProof $proof,
        AuthorizationOperationDescriptor $operation
    ): ApprovalProofVerification;
}
# 85. Verification
Deberá validar:
signature/integrity
workflow
workflow version
operation
resource
tenant
scope
expiration
revocation
required evidence
distinctness constraints
# 86. Operation Binding
El ApprovalProof deberá estar vinculado a:
Ability
Principal/Requester
Resource
Tenant
Scope
Security-Relevant Parameters
Resource Version
según Policy.
# 87. Ejemplo
Approval:
payment.approve
Payment#900
amount=100,000
no sirve para:
Payment#901
# 88. Amount Binding
Tampoco para:
Payment#900
amount=900,000
si el amount forma parte del fingerprint.
# 89. Resource Version
Para recursos mutables:
resource_version
deberá formar parte del proof cuando sea relevante.
# 90. TOCTOU
Problema:
approve operation
        ↓
resource changes
        ↓
execute old approval
Debe prevenirse.
# 91. Revalidation
Antes de ejecución:
Current Operation Fingerprint
        ==
Approved Operation Fingerprint
# 92. Approval Expiration
Todo workflow sensible deberá poder definir:
approval TTL
# 93. Ejemplo
Payment approval valid for:
30 minutes
# 94. Stage Expiration
También:
Stage 1 expires in 24h
Stage 2 expires in 4h
Final approval expires in 30m
# 95. Expiration Policy
final readonly class ApprovalExpirationPolicy
{
    public function __construct(
        public ?DateInterval $workflowTtl,
        public ?DateInterval $proofTtl,
    ) {}
}
# 96. Expired Approval
Resultado:
approval.expired
No:
silently approve
# 97. Approval Revocation
Un approval podrá revocarse antes de ejecución.
# 98. Revocation
final readonly class ApprovalRevocation
{
    public function__construct(
        public ApprovalDecisionId $decision,
        public PrincipalReference $revokedBy,
        public DateTimeImmutable $revokedAt,
        public string $reason,
    ) {}
}
# 99. Revocation Authorization
Revocar también deberá ser una Ability.
Ejemplo:
payment.approval.revoke
# 100. Approved Workflow Revocation
Si una evidencia necesaria es revocada:
APPROVED
→ no longer executable
# 101. Execution Claim
Para evitar doble ejecución podrá existir:
ApprovalExecutionClaim
# 102. Purpose
Evitar:
same approval proof
→ execute twice
# 103. Single-Use Approval
Operaciones como:
payment.execute
organization.delete
secret.rotate
deberán soportar proofs:
single-use
# 104. Atomic Consumption
La transición:
APPROVED
→ EXECUTED
deberá realizarse de forma atómica.
# 105. Concurrency Problem
Dos workers:
Worker A → execute
Worker B → execute
usando el mismo proof.
Solo uno debe ganar.
# 106. ApprovalExecutionStore
interface ApprovalExecutionStoreInterface
{
    public function consume(
        ApprovalProof $proof
    ): ApprovalConsumptionResult;
}
# 107. Database Constraint
La implementación persistente deberá utilizar mecanismos como:
unique constraints
compare-and-swap
transactions
locking
según driver.
# 108. Idempotency
Operaciones aprobadas deberán integrarse con:
idempotency keys
cuando sea apropiado.
# 109. Approval Request Idempotency
Crear dos veces la misma solicitud accidentalmente deberá poder detectarse.
# 110. Operation Correlation
Podrá utilizarse:
operation_fingerprint
+
requester
+
tenant
según Policy.
# 111. Approval Delegation
Un approver podrá delegar su capacidad cuando la organización lo permita.
# 112. Approval Delegation no es Approval Transfer
Delegar autoridad significa:
A delegates approval capability to B
No:
A's approval becomes B's approval
# 113. Evidence
Si B aprueba mediante delegación:
principal = B
authority_source = delegation from A
# 114. Audit
Debe conservar:
delegator
delegate
delegation capability
delegation expiration
# 115. Delegation Constraints
Puede limitarse por:
tenant
scope
ability
resource type
amount
time
workflow
stage
# 116. No Delegation Policy
Stages críticos podrán declarar:
delegation_allowed=false
# 117. Delegation and Distinctness
Si:
A delegates to B
y:
A approves
B approves
la Policy debe decidir si eso constituye independencia suficiente.
# 118. Strict Dual Control
Para operaciones críticas puede establecerse:
delegation relationships cannot satisfy independence
# 119. Independence Model
Debe existir una abstracción:
interface ApprovalIndependenceEvaluatorInterface
{
    public function evaluate(
        iterable $evidence,
        ApprovalRequirement $requirement
    ): IndependenceDecision;
}
# 120. Independence Factors
Puede considerar:
distinct principals
distinct actors
delegation relationships
impersonation relationships
organizational hierarchy
same service identity
same credential
# 121. Credential Independence
Para sistemas altamente sensibles podría requerirse:
independent credentials
además de identidades distintas.
# 122. Ejemplo
Dos Service Principals controlados por:
same underlying credential
podrían no satisfacer Dual Control estricto.
# 123. Core vs Advanced
VoltStack Core deberá definir la abstracción.
Los modelos especializados podrán implementarse mediante plugins.
# 124. Approval Assurance
Aprobar puede requerir mayor assurance que visualizar la solicitud.
Ejemplo:
payment.view
→ Standard

payment.approve
→ Strong
# 125. Integration with Document 23
El Approval Engine deberá invocar Authorization normalmente.
Approver
    ↓
payment.approve
    ↓
Authorization
    ↓
Contextual Policy
    ↓
Strong Authentication required
# 126. Step-Up During Approval
Resultado:
CHALLENGE
hasta completar step-up.
# 127. Approval Risk
El riesgo puede evaluarse:
when request created
when approval occurs
when operation executes
# 128. Recommended
Operaciones críticas deberán revalidar:
execution-time risk
# 129. Approval Does Not Freeze Risk
Un approval emitido cuando:
Risk=Low
no implica que:
Risk=Low forever
# 130. Execution Risk
Si antes de ejecutar:
Risk=Critical
la operación podrá:
DENY
aunque tenga todos los approvals.
# 131. Requester Risk vs Approver Risk
Deben poder diferenciarse.
requester risk
approver risk
execution risk
# 132. Multi-Stage Assurance
Ejemplo:
Stage 1:
Standard

Stage 2:
Strong

Stage 3:
VeryStrong
# 133. Separation of Duties Scope
SoD puede aplicar a:
Tenant
Organization
Team
Workspace
Project
Resource
Transaction
Workflow
# 134. Tenant Isolation
Approval Requests siempre deberán pertenecer a un:
Tenant
cuando la aplicación sea multi-tenant.
# 135. Cross-Tenant Approval
Por defecto:
Approver Tenant A
no podrá aprobar:
Request Tenant B
# 136. Explicit Cross-Tenant Authority
Solo mediante:
platform authority
explicit capability
federated approval contract
# 137. Tenant Boundary First
Antes de evaluar eligibility:
TenantBoundary
deberá ejecutarse.
# 138. Hierarchical Scope
Ejemplo:
Organization
    ↓
Workspace
    ↓
Project
Un Organization Admin no necesariamente puede aprobar operaciones de todos los Projects.
# 139. Scope Resolution
Approval deberá reutilizar el sistema definido en el documento 22.
# 140. Approval Scope
final readonly class ApprovalScope
{
    public function __construct(
        public AuthorizationScopeReference $scope,
    ) {}
}
# 141. Scope-Bound Evidence
Un approval para:
Workspace#10
no podrá utilizarse para:
Workspace#11
# 142. Role-Based Approvers
Stage:
required_role=finance.manager
# 143. Permission-Based Approvers
Preferiblemente:
required_ability=payment.approve
para mayor flexibilidad.
# 144. Relationship-Based Approvers
Ejemplo:
manager_of(requester)
# 145. ReBAC Integration
Podrá utilizarse:
Principal
    manager_of
Requester
o:
Principal
    security_owner_of
Resource
# 146. Organizational Approval
Ejemplo:
requester.department.manager
# 147. Dynamic Candidate Resolution
El approver no necesita estar definido al crear el workflow.
Puede resolverse cuando el Stage se activa.
# 148. Snapshot vs Dynamic Candidates
La Policy deberá poder elegir:
snapshot candidates
o:
resolve dynamically
# 149. Snapshot Candidates
Ventaja:
workflow stability
# 150. Dynamic Candidates
Ventaja:
organizational changes reflected
# 151. Security Trade-Off
Ejemplo:
Manager cambia mientras request está pendiente.
¿Debe aprobar:
old manager
o:
current manager
Debe ser Policy-driven.
# 152. Approval Groups
Podrá existir:
ApprovalGroup
# 153. Ejemplo
FinanceApprovalGroup
SecurityApprovalGroup
ComplianceApprovalGroup
ExecutiveApprovalGroup
# 154. Group no concede authority automáticamente
Ser miembro de:
FinanceApprovalGroup
solo identifica candidatos.
La autorización real deberá verificarse.
# 155. Quorum
Approval Group podrá definir:
quorum
# 156. Ejemplo
Security Committee
members=7
quorum=4
# 157. Quorum vs Threshold
Threshold:
3 approvals required
Quorum puede incorporar:
minimum participation
antes de considerar válida una decisión colectiva.
# 158. Weighted Approval
Arquitectura extensible podrá soportar:
weighted approvals
# 159. Ejemplo
Manager = 1 vote
Director = 2 votes
CFO = 3 votes

required weight = 4
# 160. Core Recommendation
No usar weighted approval como default.
Mantenerlo como estrategia extensible.
# 161. Veto
Un workflow puede permitir:
veto authority
# 162. Ejemplo
Security approval = normal vote
Compliance rejection = veto
# 163. VetoRule
interface ApprovalVetoRuleInterface
{
    public function isVeto(
        ApprovalDecision $decision,
        ApprovalStage $stage
    ): bool;
}
# 164. Rejection Strategy
Deberá configurarse.
Ejemplos:
AnyRejectFails

ThresholdReject

StageSpecific

VetoOnly

Custom
# 165. Default Security Strategy
Para workflows críticos:
explicit rejection
→ reject workflow
salvo Policy diferente.
# 166. Abstention
ABSTAIN no deberá contarse como:
APPROVE
ni necesariamente como:
REJECT
# 167. Stage Completion
interface ApprovalStageCompletionStrategyInterface
{
    public function evaluate(
        ApprovalStage $stage,
        iterable $decisions
    ): ApprovalStageCompletion;
}
# 168. Approval Policy Resolver
interface ApprovalPolicyResolverInterface
{
    public function resolve(
        AuthorizationRequest $request
    ): ?ApprovalRequirement;
}
# 169. Multiple Approval Policies
Una operación puede activar varias.
Ejemplo:
Payment amount policy
+
Risk policy
+
Tenant policy
+
Compliance policy
# 170. Requirement Merge
El sistema deberá combinarlas.
# 171. Ejemplo
Policy A:
2 approvers
Policy B:
1 compliance approver
Resultado no debe ser simplemente:
max(2,1)=2
Puede requerir:
2 general approvers
+
1 compliance approver
# 172. Requirement Graph
Approval Requirements deberán poder representarse como:
AND
├── 2 Finance Approvals
└── 1 Compliance Approval
# 173. Alternative Requirements
También:
OR
├── CFO Approval
└──
    AND
    ├── 2 Directors
    └── 1 Finance Manager
# 174. Approval Requirement AST
interface ApprovalRequirementNodeInterface
{
}
Implementaciones:
ApprovalRequirementLeaf
ApprovalAndNode
ApprovalOrNode
ApprovalThresholdNode
# 175. Compilation
El AST podrá compilarse.
Declarative Policy
        ↓
Approval Requirement AST
        ↓
Compiled Approval Plan
# 176. No Runtime Reflection
Metadata de Controllers/Actions deberá compilarse.
# 177. Declarative Attributes
Ejemplo conceptual:
```php
#[Authorize('payment.execute')]
#[RequiresApproval(
    ability: 'payment.approve',
    approvals: 2
)]
#[DisallowSelfApproval]
public function execute(Payment $payment)
{
}
```
# 178. Four-Eyes Attribute
```php
#[RequiresDualControl]
```
# 179. SoD Attribute
```php
#[SeparationOfDuties(
    conflictsWith: ['payment.request']
)]
```
# 180. Prefer Domain Policies
Attributes deberán utilizarse para reglas simples.
Reglas empresariales complejas deberán vivir en:
Approval Policies
# 181. Ejemplo Policy
```php
final class PaymentApprovalPolicy
{
    public function requirements(
        Payment $payment,
        AuthorizationContext $context
    ): ApprovalRequirement
    {
        // domain-specific requirements
    }
}
```
# 182. Controller Integration
Controller:
```php
#[Authorize('payment.execute', subject: 'payment')]
public function execute(Payment $payment)
{
    // authorization already satisfied
}
```
No debería contener:
```php
if ($payment->approvals()->count() < 2) {
    ...
}
```
# 183. Service Integration
Domain Service también podrá solicitar:
```php
$authorization->authorize(
    ability: 'payment.execute',
    subject: $payment,
);
```
# 184. Approval Challenge
Si falta aprobación:
AuthorizationDecision:
CHALLENGE
Requirement:
ApprovalRequirement
# 185. Example Response
```json
{
    "error": "authorization_challenge",
    "requirements": [
        {
            "type": "approval",
            "workflow": "payment-high-value",
            "required": 2
        }
    ]
}
```
# 186. API Security
No necesariamente revelar:
exact internal approval topology
approver identities
security thresholds
a clientes no autorizados.
# 187. Public Challenge Representation
Puede limitarse a:
approval_required
# 188. Internal Representation
Puede contener:
workflow
stages
requirements
reason codes
# 189. SPA Integration
VoltStack Frontend Runtime podrá interpretar:
approval_required
# 190. UX
Puede mostrar:
Operation pending approval
sin ejecutar la mutación.
# 191. Asynchronous Approval
Los approvals normalmente serán:
asynchronous
# 192. Original Request
No debe mantenerse abierta una request HTTP esperando horas.
# 193. Flow
```text
HTTP Request
    ↓
Authorization
    ↓
Approval Required
    ↓
Create Approval Request
    ↓
202 / Domain Response
    ↓
Approvals asynchronously
    ↓
Operation becomes executable
```
# 194. Important Separation
Authorization Engine determina:
approval required
Approval Workflow Service administra:
lifecycle
# 195. No Automatic Workflow Creation Everywhere
authorize() no necesariamente deberá crear automáticamente workflows.
# 196. Razón
Authorization checks deben poder ser:
side-effect free
# 197. Recommended API
Separar:
```php
$decision = $authorization->check(...);
$approval = $approvalManager->request(...);
```
# 198. Authorization Check Purity
Idealmente:
check
→ no persistent mutation
# 199. Explicit Approval Request
Ejemplo:
```php
$decision = $authorization->check(
    'payment.execute',
    $payment
);

if ($decision->requiresApproval()) {
    $approvalManager->request(
        $decision->approvalRequirement()
    );
}
```
# 200. Framework Convenience
VoltStack podrá proporcionar una abstracción superior:
$authorization->authorizeOrRequestApproval(...);
pero deberá ser explícitamente side-effecting.
# 201. Command Pattern
Para operaciones empresariales complejas podrá utilizarse:
AuthorizedCommand
# 202. Example
ExecutePaymentCommand
se registra como operación pendiente.
# 203. Command Snapshot
Puede contener:
command type
security-relevant parameters
operation fingerprint
resource version
# 204. No Arbitrary Object Serialization
No serializar directamente cualquier objeto PHP como command persistente.
# 205. Command Reconstruction
Usar:
typed command payload
versioned schema
# 206. Workflow Events
El sistema deberá emitir eventos.
ApprovalRequested
ApprovalStageActivated
ApprovalSubmitted
ApprovalGranted
ApprovalRejected
ApprovalRevoked
ApprovalStageCompleted
ApprovalWorkflowApproved
ApprovalWorkflowRejected
ApprovalWorkflowExpired
ApprovalWorkflowCancelled
ApprovalProofIssued
ApprovalProofConsumed
# 207. Events Are Facts
Los eventos deberán representar hechos ya ocurridos.
# 208. No Authorization Through Event Listener
Un listener no deberá poder convertir:
DENY
en:
ALLOW
accidentalmente.
# 209. Domain Events vs Security Events
Separar:
ApprovalWorkflowApproved
de:
AuthorizationApprovalSatisfied
cuando sea necesario.
# 210. Audit Trail
Cada workflow deberá conservar:
who requested
who acted
as whom
what operation
what resource
what tenant
what scope
which policy
which workflow version
which decisions
which assurance
timestamps
final outcome
# 211. Audit Integrity
Para entornos regulados podrá soportarse:
tamper-evident audit
mediante plugin.
# 212. Audit Event
final readonly class ApprovalAuditEvent
{
    public function __construct(
        public string $type,
        public DateTimeImmutable $occurredAt,
        public array $references,
        public array $metadata,
    ) {}
}
# 213. Audit Privacy
No registrar innecesariamente:
passwords
tokens
private keys
raw MFA data
sensitive business payloads
# 214. Reason Codes
Ejemplos:
approval.required
approval.pending
approval.satisfied
approval.rejected
approval.expired
approval.revoked

approval.self_approval_forbidden
approval.actor_not_independent
approval.principal_not_independent
approval.insufficient_quorum
approval.stage_not_active
approval.not_eligible

sod.static_conflict
sod.dynamic_conflict
sod.conflict_of_interest

approval.proof_invalid
approval.proof_expired
approval.proof_consumed
approval.operation_changed
approval.resource_version_changed
# 215. Observability
Métricas:
authorization.approval.requested.total
authorization.approval.approved.total
authorization.approval.rejected.total
authorization.approval.expired.total
authorization.approval.revoked.total

authorization.sod.conflict.total

authorization.approval.duration
authorization.approval.stage.duration
# 216. Metric Labels
Baja cardinalidad:
workflow_type
stage_type
outcome
tenant_tier
ability_group
# 217. Avoid
No utilizar como labels:
approval_request_id
user_id
payment_id
# 218. Tracing
Span:
authorization.approval.evaluate
Child spans:
approval.policy.resolve
approval.candidate.authorize
approval.sod.evaluate
approval.stage.evaluate
approval.proof.verify
approval.execution.consume
# 219. Notifications
El sistema podrá integrarse con:
Notification subsystem
Mail
WebSocket
Push
Queue
# 220. Core Independence
Approval Core no deberá depender directamente de:
email provider
Slack
Teams
SMS
# 221. Notification Event
Emitirá:
ApprovalRequested
y adapters externos decidirán cómo notificar.
# 222. Approval Inbox
Aplicaciones podrán construir:
My Pending Approvals
sobre queries del Approval Repository.
# 223. Repository
interface ApprovalRequestRepositoryInterface
{
    public function find(
        ApprovalRequestId $id
    ): ?ApprovalRequest;
}
# 224. Query Service
Separar lectura:
interface ApprovalQueryServiceInterface
{
    public function pendingFor(
        PrincipalReference $principal,
        ApprovalQuery $query
    ): iterable;
}
# 225. Query Authorization
Ver Approval Requests también deberá estar autorizado.
# 226. Information Leakage
Un usuario no debe poder descubrir:
sensitive pending operations
solo porque existe un ID predecible.
# 227. IDs
Utilizar identificadores:
opaque
non-sequential when exposed externally
cuando sea apropiado.
# 228. Persistence Model
Conceptualmente:
authorization_approval_requests
authorization_approval_stages
authorization_approval_decisions
authorization_approval_revocations
authorization_approval_proofs
authorization_approval_executions
# 229. Approval Requests Table
Ejemplo conceptual:
id
tenant_id
workflow_id
workflow_version
requester_type
requester_id
actor_type
actor_id
ability
subject_type
subject_id
scope_type
scope_id
operation_fingerprint
state
created_at
expires_at
# 230. Decisions Table
id
approval_request_id
stage_id
principal_type
principal_id
actor_type
actor_id
decision
assurance_level
operation_fingerprint
decided_at
# 231. Immutability
Security-relevant decision rows deberán ser:
insert-only
cuando sea posible.
# 232. Revocations Table
decision_id
revoked_by
reason
revoked_at
# 233. Execution Table
approval_request_id
proof_id
operation_fingerprint
consumed_at
execution_reference
# 234. Database Isolation
Todas las queries multi-tenant deberán incluir:
tenant boundary
# 235. Indexes
Posibles índices:
tenant_id + state
tenant_id + requester
tenant_id + expires_at
approval_request_id + stage_id
principal + state
operation_fingerprint
# 236. Unique Constraints
Ejemplo:
unique(
    approval_request_id,
    stage_id,
    principal_type,
    principal_id
)
si la Policy permite solo una decisión por Principal.
# 237. Actor Constraint
En entornos estrictos también puede requerirse unicidad por:
actor
# 238. Decision Supersession
Si se permite cambiar de:
ABSTAIN
→ APPROVE
deberá registrarse como nueva decisión con:
supersedes
# 239. Rejection Reversal
Por default:
REJECT
deberá ser terminal para el Stage si la estrategia así lo define.
# 240. Administrative Override
Puede existir:
approval.override
pero deberá ser excepcional.
# 241. Override no debe ser bypass oculto
Debe producir:
explicit audit event
reason
actor
authority source
# 242. Non-Overrideable Workflows
Operaciones críticas podrán declarar:
override_allowed=false
# 243. Break-Glass
Si se integra con acceso de emergencia:
break-glass
deberá ser un mecanismo independiente y altamente auditable.
# 244. Break-Glass no es Approval
No modelar:
emergency bypass
como:
fake approval
# 245. Approval Cancellation
Requester podrá cancelar mientras:
not executed
si la Policy lo permite.
# 246. Cancellation Ability
approval.cancel
# 247. Cancelled Workflow
No puede volver a ejecutarse.
# 248. Restart
Debe generarse:
new ApprovalRequest
# 249. Workflow Migration
Requests existentes no deberán migrarse silenciosamente a nuevas versiones.
# 250. Explicit Migration
Si se requiere:
ApprovalWorkflowMigration
con auditoría.
# 251. Policy Change During Workflow
Caso:
Workflow started under Policy v5
Policy becomes v6
# 252. Two Models
VoltStack deberá soportar conceptualmente:
Snapshot Policy
y:
Revalidate Current Policy
# 253. Security Recommendation
Antes de ejecución deberá verificarse al menos que:
current platform security requirements
no sean menos permisivas que las utilizadas originalmente.
# 254. Tightened Policy
Si Policy nueva exige:
3 approvals
y antigua:
2 approvals
operaciones críticas podrán exigir el nuevo requisito.
# 255. Relaxed Policy
Si Policy nueva baja de:
3
→ 1
no necesariamente deberá invalidar approvals existentes.
# 256. Policy Version Metadata
Proof deberá conservar:
policy_version
workflow_version
# 257. Separation of Duties Graph
Para sistemas avanzados, SoD podrá modelarse como grafo:
Role A ──conflicts── Role B

Ability X ──conflicts── Ability Y

Operation Maker ──cannot── Operation Checker
# 258. Conflict Graph
interface AuthorityConflictGraphInterface
{
    public function conflicts(
        AuthorityReference $left,
        AuthorityReference $right
    ): bool;
}
# 259. Transitive Conflicts
No deberán asumirse automáticamente.
Si:
A conflicts B
B conflicts C
no necesariamente:
A conflicts C
# 260. Explicit Semantics
Cada relación deberá declarar:
symmetric
directional
transitive
scope
# 261. Static SoD Assignment Integration
RBAC Assignment:
assign Role
    ↓
SoD Validator
    ↓
GRANT / DENY
# 262. Permission Assignment
También:
assign Permission
    ↓
Conflict Evaluation
# 263. Dynamic SoD Runtime
Authorization Request
    ↓
Operation History
    ↓
SoD Rules
    ↓
Decision
# 264. Operation History Provider
interface AuthorizationOperationHistoryProviderInterface
{
    public function history(
        PrincipalReference $principal,
        AuthorizationOperationDescriptor $operation
    ): iterable;
}
# 265. Example
Regla:
creator cannot approve
requiere conocer:
who created resource
# 266. Prefer Resource Metadata
Cuando sea posible:
resource.created_by
es más eficiente que consultar historial completo.
# 267. Approval Planner
Deberá existir:
ApprovalPlanner
# 268. Responsibility
Transformar:
Authorization Request
+
Approval Policies
+
SoD Rules
+
Context
en:
Compiled Approval Plan
# 269. Approval Plan
Ejemplo:
Ability:
payment.execute

Requirements:

Stage 1
    ability: payment.approve
    group: finance
    required: 2
    distinct_principals: 2
    self_approval: forbidden

Stage 2
    ability: payment.compliance.approve
    required: 1

Execution:
    strong authentication
    risk <= medium
    proof single-use
# 270. Planner Optimization
Puede precompilar:
static policy structure
candidate selector definitions
SoD rule references
stage topology
# 271. Dynamic Data
No precompilar:
current approvers
current risk
current roles
resource amount
cuando sean variables runtime.
# 272. Approval Compiler
interface ApprovalPolicyCompilerInterface
{
    public function compile(
        ApprovalPolicyDefinition $definition
    ): CompiledApprovalPolicy;
}
# 273. Cache
Compiled Approval Policies podrán almacenarse en:
authorization cache
# 274. Cache Key
Debe incluir:
application version
authorization policy version
tenant policy version
workflow version
según corresponda.
# 275. Runtime State no va al Policy Cache
Nunca cachear globalmente:
pending approvals
current approvers
current decisions
como metadata compilada.
# 276. Approval Manager
Contrato central:
interface ApprovalManagerInterface
{
    public function request(
        ApprovalOperation $operation
    ): ApprovalRequest;

    public function approve(
        ApprovalRequestId $request,
        Principal $principal
    ): ApprovalDecision;

    public function reject(
        ApprovalRequestId $request,
        Principal $principal,
        ?string $reason = null
    ): ApprovalDecision;

    public function cancel(
        ApprovalRequestId $request,
        Principal $principal
    ): void;
}
# 277. Internal Pipeline
ApprovalManager
    ↓
Load Request
    ↓
Tenant Boundary
    ↓
Resolve Active Stage
    ↓
Candidate Eligibility
    ↓
Authorization
    ↓
Contextual Authorization
    ↓
SoD Evaluation
    ↓
Conflict Evaluation
    ↓
Record Decision
    ↓
Evaluate Stage
    ↓
Evaluate Workflow
    ↓
Emit Events
# 278. Approval Request Pipeline
Operation
    ↓
Authorization Check
    ↓
Approval Requirement
    ↓
Approval Planner
    ↓
Operation Fingerprint
    ↓
Persist Request
    ↓
Activate Stage
    ↓
Emit ApprovalRequested
# 279. Execution Pipeline
Execute Request
    ↓
Load Approval Proof
    ↓
Verify Integrity
    ↓
Verify Expiration
    ↓
Verify Tenant
    ↓
Verify Scope
    ↓
Verify Operation Fingerprint
    ↓
Verify Resource Version
    ↓
Verify SoD
    ↓
Re-Authorize Executor
    ↓
Re-Evaluate Risk
    ↓
Atomic Proof Consumption
    ↓
Execute Operation
# 280. Critical Ordering
Nunca:
execute
    ↓
consume approval
Debe coordinarse de forma atómica o idempotente.
# 281. Transaction Boundaries
Approval persistence y domain mutation pueden estar en bases diferentes.
Por ello no asumir:
single DB transaction
universal.
# 282. Distributed Execution
Para sistemas distribuidos podrá utilizarse:
Outbox
Inbox
Idempotency
Saga
Transactional Messaging
# 283. Core Scope
Authorization definirá contratos.
Infrastructure modules implementarán mecanismos distribuidos.
# 284. Service-to-Service Approval
Un servicio puede solicitar una operación que requiere aprobación humana.
AutomationService
    ↓
production.change.request
    ↓
Human Approval
    ↓
DeploymentService
    ↓
Execute
# 285. Requester != Executor
El modelo deberá soportarlo explícitamente.
# 286. Requester
Quien solicita.
# 287. Approver
Quien aprueba.
# 288. Executor
Quien ejecuta.
# 289. Actor
Quien materialmente realiza cada acción.
# 290. Principal
Identidad bajo cuya autoridad se realiza.
# 291. Complete Identity Model
RequesterPrincipal
RequesterActor

ApproverPrincipal
ApproverActor

ExecutorPrincipal
ExecutorActor
# 292. Reason
Esto permite auditoría correcta en:
delegation
impersonation
service-to-service
automation
# 293. Human Approval
Una Policy podrá exigir:
HumanPrincipal
# 294. Machine Approval
Otras podrán permitir:
ServicePrincipal
# 295. Mixed Approval
Ejemplo:
1 automated compliance approval
+
1 human finance approval
# 296. Approval Source
Evidence deberá indicar:
human
service
policy engine
external authority
# 297. Automated Approval
Nunca deberá confundirse con:
human approval
si la Policy exige humano.
# 298. External Approval Systems
VoltStack podrá integrarse con sistemas externos.
Contrato:
interface ExternalApprovalProviderInterface
{
    public function verify(
        ExternalApprovalReference $reference
    ): ExternalApprovalEvidence;
}
# 299. External Trust
Cada provider deberá declarar:
trust level
issuer
verification method
# 300. External Evidence
Debe normalizarse al modelo interno.
# 301. Plugin System
Integración con documento 19.
Podrán registrarse:
Approval Strategies
Candidate Selectors
SoD Evaluators
Conflict Evaluators
External Approval Providers
Proof Stores
Workflow Stores
Notification Adapters
# 302. No Plugin Override of NonBypassable Rules
Un plugin no deberá poder ignorar:
tenant isolation
platform SoD
proof integrity
# 303. Directory Structure propuesta
Quantum/
└── Authorization/
    └── Approval/
        ├── Contracts/
        │   ├── ApprovalManagerInterface.php
        │   ├── ApprovalPolicyResolverInterface.php
        │   ├── ApprovalEligibilityEvaluatorInterface.php
        │   ├── ApprovalCandidateSelectorInterface.php
        │   ├── ApprovalProofVerifierInterface.php
        │   ├── ApprovalIndependenceEvaluatorInterface.php
        │   ├── ApprovalExecutionStoreInterface.php
        │   ├── ApprovalStageCompletionStrategyInterface.php
        │   └── ApprovalPolicyCompilerInterface.php
        │
        ├── Model/
        │   ├── ApprovalRequest.php
        │   ├── ApprovalStage.php
        │   ├── ApprovalDecision.php
        │   ├── ApprovalEvidence.php
        │   ├── ApprovalProof.php
        │   ├── ApprovalRevocation.php
        │   ├── ApprovalOperation.php
        │   └── ApprovalScope.php
        │
        ├── Workflow/
        │   ├── ApprovalWorkflowDefinition.php
        │   ├── ApprovalWorkflowState.php
        │   ├── ApprovalStageState.php
        │   ├── ApprovalWorkflowEngine.php
        │   ├── ApprovalPlanner.php
        │   └── CompiledApprovalPlan.php
        │
        ├── Strategy/
        │   ├── ApprovalStrategy.php
        │   ├── AnyApprovalStrategy.php
        │   ├── AllApprovalStrategy.php
        │   ├── ThresholdApprovalStrategy.php
        │   ├── SequentialApprovalStrategy.php
        │   ├── ParallelApprovalStrategy.php
        │   └── HierarchicalApprovalStrategy.php
        │
        ├── Requirements/
        │   ├── ApprovalRequirement.php
        │   ├── ApprovalRequirementNodeInterface.php
        │   ├── ApprovalRequirementLeaf.php
        │   ├── ApprovalAndNode.php
        │   ├── ApprovalOrNode.php
        │   ├── ApprovalThresholdNode.php
        │   ├── DistinctPrincipalConstraint.php
        │   ├── DistinctActorConstraint.php
        │   └── SelfApprovalConstraint.php
        │
        ├── SoD/
        │   ├── StaticSeparationOfDutiesRuleInterface.php
        │   ├── DynamicSeparationOfDutiesRuleInterface.php
        │   ├── SeparationOfDutiesManager.php
        │   ├── RoleConflictRegistry.php
        │   ├── AuthorityConflictGraph.php
        │   ├── SoDDecision.php
        │   └── SoDOutcome.php
        │
        ├── Conflict/
        │   ├── ApprovalConflictRuleInterface.php
        │   ├── SelfApprovalConflictRule.php
        │   ├── ImpersonationConflictRule.php
        │   ├── DelegationConflictRule.php
        │   └── ConflictDecision.php
        │
        ├── Eligibility/
        │   ├── ApprovalEligibilityEvaluator.php
        │   ├── AbilityCandidateSelector.php
        │   ├── RoleCandidateSelector.php
        │   ├── RelationshipCandidateSelector.php
        │   └── ExplicitCandidateSelector.php
        │
        ├── Proof/
        │   ├── ApprovalProofIssuer.php
        │   ├── ApprovalProofVerifier.php
        │   ├── ApprovalProofSerializer.php
        │   ├── ApprovalProofConsumption.php
        │   └── OperationFingerprint.php
        │
        ├── Persistence/
        │   ├── ApprovalRequestRepositoryInterface.php
        │   ├── ApprovalDecisionRepositoryInterface.php
        │   ├── ApprovalProofRepositoryInterface.php
        │   └── ApprovalQueryServiceInterface.php
        │
        ├── Compilation/
        │   ├── ApprovalPolicyCompiler.php
        │   ├── ApprovalWorkflowCompiler.php
        │   ├── ApprovalRequirementCompiler.php
        │   └── ApprovalMetadataCompiler.php
        │
        ├── Metadata/
        │   ├── RequiresApproval.php
        │   ├── RequiresDualControl.php
        │   ├── DisallowSelfApproval.php
        │   └── SeparationOfDuties.php
        │
        ├── Events/
        │   ├── ApprovalRequested.php
        │   ├── ApprovalSubmitted.php
        │   ├── ApprovalGranted.php
        │   ├── ApprovalRejected.php
        │   ├── ApprovalRevoked.php
        │   ├── ApprovalWorkflowApproved.php
        │   ├── ApprovalWorkflowExpired.php
        │   └── ApprovalProofConsumed.php
        │
        └── Exceptions/
            ├── ApprovalException.php
            ├── ApprovalConflictException.php
            ├── ApprovalExpiredException.php
            ├── ApprovalProofException.php
            ├── SeparationOfDutiesException.php
            └── ApprovalExecutionException.php
# 304. Testing Strategy
El sistema deberá tener pruebas específicas para:
workflow lifecycle
approval strategies
SoD
dual control
self approval
proof integrity
expiration
revocation
concurrency
multi-tenancy
delegation
impersonation
risk
assurance
# 305. Self-Approval Property
Para Policy:
self_approval=false
debe cumplirse:
RequesterPrincipal
==

ApproverPrincipal
→ DENY
# 306. Distinct Principal Property
Para:
required_distinct_principals=2
dos approvals del mismo Principal jamás satisfacen el requirement.
# 307. Distinct Actor Property
Dos Principals diferentes controlados mediante el mismo Actor no satisfacen:
required_distinct_actors=2
# 308. Approval Cannot Grant Authority Property
StructuralAuthorization = DENY
implica:
FinalAuthorization != ALLOW
independientemente de approvals.
# 309. Expiration Property
now > proof.expiresAt
implica:
proof invalid
# 310. Operation Binding Property
approvedFingerprint != currentFingerprint
implica:
proof invalid
# 311. Tenant Binding Property
proof.tenant != currentTenant
implica:
proof invalid
# 312. Scope Binding Property
Igualmente para Scope.
# 313. Single Consumption Property
Para proof single-use:
```text
consume(proof)
→ success

consume(proof)
→ failure
```
# 314. Concurrency Property
Dos ejecuciones simultáneas:
consume same proof
deben producir:
exactly one successful consumption
# 315. Workflow Version Property
Un proof de:
Workflow v2
no deberá validarse como:
Workflow v3
sin migración explícita.
# 316. Revocation Property
Evidence revocada no deberá contar hacia quorum.
# 317. Rejection Property
Una decisión terminal de rechazo no deberá desaparecer del historial.
# 318. Append-Only Property
Las decisiones registradas no deberán modificarse silenciosamente.
# 319. Impersonation Property
Cuando:
impersonated_approval=false
cualquier approval bajo impersonation deberá fallar.
# 320. Delegation Property
Una delegación expirada no concede eligibility.
# 321. Static SoD Property
Dos Roles incompatibles no deberán coexistir cuando Static SoD lo prohíba.
# 322. Dynamic SoD Property
Poseer dos Roles compatibles estáticamente no permite violar una regla transaccional dinámica.
# 323. Persistent Worker Test
Request A:
Tenant A
Approval#1
Request B en mismo FrankenPHP worker:
Tenant B
Approval#2
No deberán compartir:
workflow
evidence
candidate state
tenant
approval context
# 324. Approval Context
Deberá ser:
immutable
execution scoped
tenant scoped
# 325. No Static State
Prohibido:
static $currentApprovalRequest;
static $currentApprover;
static $currentWorkflow;
# 326. Performance
El Approval System no estará normalmente en el mismo hot path que:
simple authorization checks
# 327. Important Optimization
Abilities sin approval requirements deberán evitar cargar:
Approval subsystem
Approval repositories
Workflow state
SoD history
# 328. Planner
El Authorization Planner deberá conocer:
approval_required=false
y eliminar completamente ese pipeline.
# 329. Batch Candidate Checks
Para listar posibles approvers se deberá evitar:
N+1 authorization queries
# 330. Candidate Query Optimization
Podrá utilizar:
RBAC indexes
scope indexes
relationship indexes
compiled eligibility predicates
# 331. Security Over Performance
Nunca cachear eligibility de forma que permita:
revoked role
expired delegation
removed tenant membership
seguir aprobando.
# 332. Final Architecture
                    AUTHORIZATION REQUEST
                            │
                            ↓
                   STRUCTURAL AUTHORITY
                            │
                            ↓
                      DOMAIN POLICY
                            │
                            ↓
                  CONTEXT / RISK / ASSURANCE
                            │
                            ↓
                  APPROVAL REQUIREMENTS
                            │
              ┌─────────────┴─────────────┐
              │                           │
        NO APPROVAL                  APPROVAL REQUIRED
              │                           │
              ↓                           ↓
         FINAL CHECK                CREATE WORKFLOW
                                          │
                                          ↓
                                    STAGE RESOLUTION
                                          │
                                          ↓
                                  APPROVER CANDIDATES
                                          │
                                          ↓
                                      AUTHORIZATION
                                          │
                                          ↓
                                   SoD / CONFLICTS
                                          │
                                          ↓
                                  APPROVAL EVIDENCE
                                          │
                                          ↓
                                  REQUIREMENT GRAPH
                                          │
                                          ↓
                                    APPROVAL PROOF
                                          │
                                          ↓
                                 RE-AUTHORIZATION
                                          │
                                          ↓
                                OPERATION REVALIDATION
                                          │
                                          ↓
                               ATOMIC PROOF CONSUMPTION
                                          │
                                          ↓
                                       EXECUTE
# 333. Arquitectura SoD
                   PRINCIPAL
                       │
             ┌─────────┴─────────┐
             ↓                   ↓
         ROLE / ABILITY      OPERATION
             │                   │
             ↓                   ↓
        STATIC SoD           DYNAMIC SoD
             │                   │
             ↓                   ↓
     Assignment Conflict    Runtime Conflict
             │                   │
             └─────────┬─────────┘
                       ↓
                CONFLICT ENGINE
                       │
              ┌────────┴────────┐
              ↓                 ↓
          SATISFIED          CONFLICT
              │                 │
              ↓                 ↓
          CONTINUE            DENY
# 334. Arquitectura Dual Control
                  CRITICAL OPERATION
                          │
                          ↓
                  Approval Requirement
                          │
             ┌────────────┴────────────┐
             ↓                         ↓
        Principal A               Principal B
             │                         │
        Actor A                    Actor B
             │                         │
      Authorization              Authorization
             │                         │
       Strong Assurance          Strong Assurance
             │                         │
       Approval Evidence         Approval Evidence
             │                         │
             └────────────┬────────────┘
                          ↓
                 Independence Check
                          │
                 Principal A != B
                 Actor A != B
                          │
                          ↓
                   DUAL CONTROL
                       SATISFIED
                          │
                          ↓
                   Approval Proof
                          │
                          ↓
                    Re-Authorize
                          │
                          ↓
                       Execute
# 335. Invariantes fundamentales
VoltStack deberá garantizar:
Authority
Approval cannot create authority.
Execution
Approved does not mean executed.
Revalidation
Every sensitive execution may re-authorize.
Identity
Principal and Actor remain distinguishable.
Self Approval
Self-approval restrictions cannot be bypassed
through roles, delegation or impersonation.
Dual Control
Two approvals do not necessarily mean
two independent authorities.
Proof
Approval proofs are scoped, expiring,
verifiable and optionally single-use.
Resource Integrity
An approval is valid only for the operation
that was actually approved.
Tenant Isolation
Approval evidence never crosses Tenant
boundaries accidentally.
SoD
Conflicting authorities and operations
must be explicitly detected.
Runtime
Approval context never leaks between
persistent worker requests.
# 336. Filosofía arquitectónica
VoltStack deberá adoptar los siguientes principios:
Authorization determines whether authority exists.

Approval determines whether additional independent
authorization is required before execution.

Approval is not permission.

Approval is not authentication.

Approval is not a bypass.

Approval is evidence of an independently authorized decision.

Approvers must themselves be authorized.

Approval eligibility must be re-evaluated.

Maker and checker are distinct concepts.

Principal independence and Actor independence
are different security properties.

Dual Control requires independent authority,
not merely multiple database rows.

Separation of Duties applies both during
authority assignment and during runtime.

Approval workflows are versioned.

Approval decisions are immutable security facts.

Approval proofs are operation-bound.

Resource changes may invalidate approval.

Risk may invalidate previously satisfied approvals.

Approved operations must be revalidated before execution.

Single-use approvals must be consumed atomically.

Tenant boundaries apply to the complete approval lifecycle.

Security-critical approval state must never live
in global mutable worker state.
# 337. Resultado esperado
Con este sistema, VoltStack podrá modelar desde operaciones relativamente simples:
Expense
    ↓
Manager Approval
hasta procesos empresariales complejos:
High-Value Payment
        ↓
Requester Authorization
        ↓
Maker-Checker
        ↓
2 Finance Approvers
        ↓
Compliance Approval
        ↓
Distinct Principal Validation
        ↓
Distinct Actor Validation
        ↓
Separation of Duties
        ↓
Strong Authentication
        ↓
Risk Re-Evaluation
        ↓
Approval Proof
        ↓
Resource Version Validation
        ↓
Final Authorization
        ↓
Atomic Proof Consumption
        ↓
Payment Execution
El Authorization System deja así de limitarse al modelo:
User → Permission → Allow/Deny
y puede manejar:
Principal
    ↓
Authority
    ↓
Context
    ↓
Risk
    ↓
Approval Requirement
    ↓
Independent Authorities
    ↓
Separation of Duties
    ↓
Approval Evidence
    ↓
Execution Authority
Esto permitirá utilizar VoltStack en aplicaciones con requisitos de autorización considerablemente más estrictos, incluyendo sistemas:
financieros
ERP
administrativos
SaaS empresariales
infraestructura
DevOps
gestión documental
multi-tenant
compliance
operaciones críticas
sin acoplar la lógica de aprobación al dominio HTTP ni convertir Controllers y Services en motores de seguridad.
# 338. Relación con documentos anteriores
Este documento extiende directamente:
08_DECISION_MANAGER_VOTERS_AND_STRATEGY_SYSTEM.md

09_AUTHORIZATION_PLANNER_AND_POLICY_PIPELINE_SYSTEM.md

10_AUTHORIZATION_ATTRIBUTES_AND_DECLARATIVE_METADATA_SYSTEM.md

11_CONTROLLER_ROUTE_AND_ACTION_AUTHORIZATION_INTEGRATION_SYSTEM.md

12_ROLE_PERMISSION_RBAC_ABAC_AND_REBAC_INTEGRATION_SYSTEM.md

13_MULTI_TENANT_AUTHORIZATION_AND_DATA_ISOLATION_SYSTEM.md

14_AUTHORIZATION_CACHE_MEMOIZATION_AND_DECISION_REUSE_SYSTEM.md

15_AUTHORIZATION_AUDIT_OBSERVABILITY_TRACING_AND_EXPLAINABILITY_SYSTEM.md

16_AUTHORIZATION_FAILURE_ERROR_DENIAL_AND_EXCEPTION_HANDLING_SYSTEM.md

18_AUTHORIZATION_COMPILATION_OPTIMIZATION_AND_RUNTIME_PERFORMANCE_SYSTEM.md

19_AUTHORIZATION_EXTENSIBILITY_PLUGIN_PROVIDER_AND_CUSTOM_EVALUATOR_SYSTEM.md

20_AUTHORIZATION_DELEGATION_IMPERSONATION_CAPABILITIES_AND_SERVICE_TO_SERVICE_SYSTEM.md

21_AUTHORIZATION_RESOURCE_OWNERSHIP_SHARING_AND_RELATIONSHIP_ACCESS_SYSTEM.md

22_AUTHORIZATION_HIERARCHICAL_SCOPES_ORGANIZATIONS_TEAMS_AND_WORKSPACES_SYSTEM.md

23_AUTHORIZATION_CONDITIONAL_CONTEXTUAL_AND_RISK_BASED_ACCESS_SYSTEM.md
La integración conceptual queda:
RBAC / ABAC / ReBAC
        ↓
Policies / Gates / Voters
        ↓
Tenant + Scope + Relationships
        ↓
Context + Risk + Assurance
        ↓
Approval Requirements
        ↓
Dual Control
        ↓
Separation of Duties
        ↓
Approval Evidence
        ↓
Final Re-Authorization
        ↓
Execution
# 339. Conclusión
El Approval Workflow, Dual Control and Separation of Duties System deberá convertir las aprobaciones en una capacidad de primera clase del Authorization System de VoltStack, sin confundir aprobación con permisos ni introducir bypasses administrativos implícitos.
El principio definitivo será:
En VoltStack, una operación puede requerir no solo que quien la solicita tenga autoridad, sino que otras identidades independientes confirmen esa operación bajo sus propias autoridades, contextos y restricciones. Una aprobación demuestra consentimiento autorizado; nunca sustituye la autorización.

La regla arquitectónica final será:
AUTHORITY
    +
CONTEXT
    +
RISK
    +
INDEPENDENT APPROVAL
    +
SEPARATION OF DUTIES
    +
FINAL REVALIDATION
    =
EXECUTABLE AUTHORITY
Con esto, VoltStack queda preparado para implementar autorización empresarial donde solicitar, aprobar y ejecutar son responsabilidades independientes, verificables y auditables.
