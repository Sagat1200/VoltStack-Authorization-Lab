# VoltStack Authorization System — Extensibility, Plugin, Provider and Custom Evaluator System

## 1. Propósito

Este documento define la arquitectura de **extensibilidad del Authorization System de VoltStack**.

El sistema de autorización no debe convertirse en un componente cerrado que únicamente conozca las implementaciones incluidas en el Core.

VoltStack deberá permitir que:

- aplicaciones;
- paquetes Composer;
- módulos Quantum;
- paquetes oficiales;
- paquetes de terceros;
- integraciones empresariales;
- motores externos de autorización;
- proveedores RBAC;
- proveedores ABAC;
- proveedores ReBAC;
- sistemas IAM;
- servicios de compliance;

puedan extender el sistema sin modificar directamente su núcleo.

La arquitectura deberá permitir incorporar:

```text
Policies
Gates
Abilities
Evaluators
Voters
Decision Strategies
Authorization Attributes
Metadata Resolvers
Policy Resolvers
Role Providers
Permission Providers
Relationship Providers
Attribute Providers
Tenant Authorization Providers
External Policy Engines
Authorization Compiler Passes
Audit Extensions
Observability Extensions
```

manteniendo las garantías establecidas por los documentos anteriores.

El principio fundamental será:

```text
Authorization Core
must define the security semantics.

Extensions may participate in those semantics,
but must not silently redefine or bypass them.
```

---

# 2. Objetivos

El sistema deberá proporcionar:

1. APIs públicas de extensión;
2. contratos estables;
3. Service Providers de autorización;
4. integración con módulos Quantum;
5. plugin discovery;
6. plugin registration;
7. custom Policies;
8. custom Gates;
9. custom Evaluators;
10. custom Voters;
11. custom Decision Strategies;
12. custom metadata;
13. custom Attributes;
14. custom Policy resolvers;
15. RBAC providers;
16. ABAC providers;
17. ReBAC providers;
18. external authorization adapters;
19. compiler extensions;
20. lifecycle control;
21. dependency management;
22. priority management;
23. capability declarations;
24. compatibility validation;
25. registry sealing;
26. namespace isolation;
27. deterministic registration;
28. conflict detection;
29. security validation;
30. observability;
31. testing contracts;
32. runtime safety;
33. FrankenPHP compatibility.

---

# 3. Principio arquitectónico

La extensibilidad deberá seguir:

```text
Extension
    ↓
Declaration
    ↓
Registration
    ↓
Normalization
    ↓
Validation
    ↓
Compilation
    ↓
Registry
    ↓
Security Validation
    ↓
Registry Seal
    ↓
Runtime
```

Nunca:

```text
Extension
    ↓
directly mutate runtime authorization
```

---

# 4. Core vs Extension

VoltStack deberá diferenciar claramente:

```text
AUTHORIZATION CORE
```

de:

```text
AUTHORIZATION EXTENSIONS
```

El Core define:

```text
Decision semantics
Authorization phases
Result model
Failure semantics
Security boundaries
Registry lifecycle
Compiler lifecycle
Execution lifecycle
```

Las extensiones proporcionan implementaciones dentro de esos contratos.

---

# 5. Regla principal

Una extensión puede decir:

```text
"I provide a new evaluator."
```

pero no:

```text
"From now on DENY means GRANT."
```

---

# 6. Extension Categories

VoltStack podrá reconocer categorías como:

```text
PolicyExtension
GateExtension
EvaluatorExtension
StrategyExtension
MetadataExtension
ProviderExtension
CompilerExtension
AuditExtension
ObservabilityExtension
ExternalEngineExtension
```

---

# 7. AuthorizationExtensionInterface

Contrato raíz opcional:

```php
interface AuthorizationExtensionInterface
{
    public function register(
        AuthorizationExtensionRegistry $registry
    ): void;
}
```

---

# 8. Bootstrap Separation

Conviene separar:

```text
register()
```

de:

```text
boot()
```

siguiendo una filosofía familiar para desarrolladores Laravel.

---

# 9. AuthorizationServiceProvider

Podrá existir:

```php
abstract class AuthorizationServiceProvider
{
    public function register(
        AuthorizationRegistrationContext $context
    ): void {
    }

    public function boot(
        AuthorizationBootContext $context
    ): void {
    }
}
```

---

# 10. Register Phase

`register()` será utilizada para declarar:

```text
Policies
Gates
Evaluators
Strategies
Providers
Compiler Passes
Metadata Resolvers
```

---

# 11. Boot Phase

`boot()` podrá utilizar información ya registrada para:

```text
cross-extension configuration
validation hooks
integration initialization
```

sin modificar estructuras que ya hayan sido selladas.

---

# 12. Lifecycle

```text
Application Bootstrap
        ↓
Discover Authorization Providers
        ↓
register()
        ↓
Build Registries
        ↓
boot()
        ↓
Normalize
        ↓
Compile
        ↓
Validate
        ↓
Seal
        ↓
Runtime
```

---

# 13. No Runtime Registration

Una vez ejecutado:

```text
AuthorizationRegistry::seal()
```

no podrá registrarse:

```text
new Policy
new Gate
new Evaluator
new Strategy
```

durante requests normales.

---

# 14. Exception

Intentarlo producirá:

```text
AuthorizationRegistrySealedException
```

---

# 15. Razón

Esto garantiza:

```text
determinism
cache correctness
compiled plan correctness
worker safety
FrankenPHP safety
```

---

# 16. Quantum Integration

Los módulos dentro de:

```text
src/Quantum/
```

podrán registrar extensiones de Authorization mediante providers.

Ejemplo:

```text
Quantum/
├── Authorization/
├── Tenant/
├── Database/
├── Routing/
└── Security/
```

---

# 17. Quantum Module

Un módulo podrá declarar:

```php
final class FinanceAuthorizationProvider
    extends AuthorizationServiceProvider
{
    public function register(
        AuthorizationRegistrationContext $context
    ): void {
        // ...
    }
}
```

---

# 18. Package Integration

Un paquete Composer podrá proporcionar:

```text
src/
    Authorization/
        FinanceAuthorizationProvider.php
```

y declararlo mediante el mecanismo general de package discovery de VoltStack.

---

# 19. No Authorization-Specific Package Scanner

Authorization deberá utilizar el sistema global de package/module discovery.

No deberá crear otro scanner Composer independiente.

---

# 20. Package Manifest

Un paquete podría declarar conceptualmente:

```text
voltstack.authorization.providers:
    - Vendor\Package\FinanceAuthorizationProvider
```

---

# 21. Compilation

El package discovery ocurre antes del:

```text
AuthorizationCompiler
```

para que sus componentes formen parte del manifest compilado.

---

# 22. Registration Context

Podrá proporcionar APIs como:

```php
interface AuthorizationRegistrationContext
{
    public function policies(): PolicyRegistrar;

    public function gates(): GateRegistrar;

    public function abilities(): AbilityRegistrar;

    public function evaluators(): EvaluatorRegistrar;

    public function strategies(): StrategyRegistrar;

    public function metadata(): MetadataRegistrar;

    public function providers(): AuthorizationProviderRegistrar;

    public function compiler(): AuthorizationCompilerRegistrar;
}
```

---

# 23. Ventaja

En lugar de exponer:

```text
raw mutable registries
```

las extensiones utilizan:

```text
controlled registrars
```

---

# 24. Policy Extension

Una extensión podrá registrar:

```php
$context->policies()->register(
    Invoice::class,
    InvoicePolicy::class
);
```

---

# 25. Multiple Policy Providers

VoltStack deberá decidir explícitamente qué ocurre si dos paquetes registran:

```text
Invoice
→ InvoicePolicyA

Invoice
→ InvoicePolicyB
```

---

# 26. Default

No utilizar:

```text
last registration wins
```

silenciosamente.

---

# 27. Conflict

Por defecto:

```text
PolicyRegistrationConflictException
```

---

# 28. Explicit Composition

Si se desean múltiples Policies:

```text
PolicyChain
CompositePolicy
multiple evaluators
```

deberán declararse explícitamente.

---

# 29. Policy Override

Podrá permitirse:

```php
$context->policies()->replace(
    Invoice::class,
    CustomInvoicePolicy::class
);
```

---

# 30. Explicit Replacement

`replace()` deberá identificar qué registration reemplaza.

---

# 31. Production Diagnostics

El compiler deberá poder mostrar:

```text
Invoice
→ App\Authorization\CustomInvoicePolicy

replaces:
Vendor\Finance\InvoicePolicy
```

---

# 32. Security-Critical Policy

Una Policy podrá declararse:

```text
nonReplaceable
```

si pertenece a un security boundary del framework o aplicación.

---

# 33. Example

```text
TenantIsolationPolicy
```

podría impedir replacement no autorizado.

---

# 34. Gate Extension

Registro:

```php
$context->gates()->define(
    'finance.export',
    FinanceExportGate::class
);
```

---

# 35. Duplicate Gate

Misma regla:

```text
duplicate canonical ability
→ compilation conflict
```

salvo composición explícita.

---

# 36. Gate Decorator

Podrá existir:

```php
$context->gates()->decorate(
    'finance.export',
    ComplianceGateDecorator::class
);
```

---

# 37. Decoration Order

Debe ser:

```text
explicit
deterministic
priority-aware
```

---

# 38. Ability Extension

Paquetes podrán registrar abilities.

Ejemplo:

```php
$context->abilities()->register(
    'finance.invoice.approve'
);
```

---

# 39. Ability Descriptor

También:

```php
$context->abilities()->register(
    AbilityDefinition::make('finance.invoice.approve')
        ->subject(Invoice::class)
        ->strategy('deny_overrides')
);
```

---

# 40. Namespace Recommendation

Paquetes deberán usar namespaces lógicos:

```text
package.resource.operation
```

Ejemplo:

```text
finance.invoice.approve
```

---

# 41. Core Abilities

VoltStack podrá reservar:

```text
volt.*
system.*
authorization.*
```

o namespaces equivalentes.

---

# 42. Reserved Namespace

Paquetes de terceros no podrán registrar abilities reservadas sin permiso explícito.

---

# 43. Custom Evaluators

Este será uno de los principales mecanismos de extensibilidad.

Contrato:

```php
interface AuthorizationEvaluatorInterface
{
    public function evaluate(
        AuthorizationEvaluationContext $context
    ): AuthorizationDecision;
}
```

---

# 44. Evaluation Context

Podrá proporcionar:

```text
Principal
Ability
Subject
Tenant
AuthorizationContext
Requirement
Execution metadata
```

sin exponer internals mutables del engine.

---

# 45. Custom Evaluator Example

```php
final class AccountStatusEvaluator
    implements AuthorizationEvaluatorInterface
{
    public function evaluate(
        AuthorizationEvaluationContext $context
    ): AuthorizationDecision {
        $principal = $context->principal();

        if (! $principal->isActive()) {
            return AuthorizationDecision::deny(
                'principal.inactive'
            );
        }

        return AuthorizationDecision::abstain();
    }
}
```

---

# 46. Registration

```php
$context->evaluators()->register(
    'account-status',
    AccountStatusEvaluator::class
);
```

---

# 47. Evaluator Definition

Para metadata avanzada:

```php
$context->evaluators()->register(
    EvaluatorDefinition::make(
        'account-status',
        AccountStatusEvaluator::class
    )
        ->phase(AuthorizationPhase::PreResolution)
        ->priority(100)
        ->dependencies(
            AuthorizationDependency::Principal
        )
);
```

---

# 48. Evaluator Metadata

Podrá declarar:

```text
ID
service
phase
priority
dependencies
required
nonBypassable
cacheability
batchability
external
cost
purity
```

---

# 49. Security Metadata

No toda metadata crítica deberá confiar ciegamente en lo declarado por un paquete.

El compiler podrá verificar lo verificable.

---

# 50. Evaluator Purity

El contrato normal deberá asumir:

```text
side-effect free
```

---

# 51. Side Effects Prohibited

No debería:

```text
send email
write database
modify Subject
change Principal
modify Tenant
```

---

# 52. Razón

Policies y evaluators deben producir:

```text
decisions
```

no ejecutar efectos de negocio.

---

# 53. Side-Effect Detection

No siempre puede comprobarse estáticamente.

Por ello será:

```text
contractual invariant
+
testing requirement
```

---

# 54. Decision Result

Un evaluator custom deberá devolver únicamente tipos normalizables.

---

# 55. Accepted Result

Idealmente:

```text
AuthorizationDecision
```

---

# 56. Convenience Results

Podrían permitirse:

```text
bool
null
AuthorizationDecision
```

si el normalizer global ya los soporta.

---

# 57. Recommendation for Extensions

Utilizar siempre:

```text
AuthorizationDecision
```

para preservar:

```text
reason
metadata
decision type
```

---

# 58. Custom Voters

VoltStack podrá proporcionar compatibilidad conceptual similar a Symfony.

Contrato:

```php
interface AuthorizationVoterInterface
{
    public function supports(
        Ability $ability,
        mixed $subject
    ): bool;

    public function vote(
        AuthorizationEvaluationContext $context
    ): AuthorizationDecision;
}
```

---

# 59. Difference

```text
Evaluator
```

forma parte de un plan explícito.

```text
Voter
```

puede determinar dinámicamente si soporta una combinación.

---

# 60. Performance

Los Voters dinámicos deberán compilarse cuando sea posible.

---

# 61. Voter Support Map

El compiler podrá transformar:

```text
supports()
```

estático en:

```text
Ability ID + Subject Type ID
→ Voter IDs
```

---

# 62. Dynamic Voter

Si `supports()` depende de runtime data:

```text
dynamic voter
```

deberá marcarse explícitamente.

---

# 63. Warning

Un número grande de dynamic voters puede degradar el hot path.

---

# 64. Custom Decision Strategies

Contrato:

```php
interface AuthorizationDecisionStrategyInterface
{
    public function decide(
        DecisionAccumulator $decisions,
        AuthorizationStrategyContext $context
    ): AuthorizationDecision;
}
```

---

# 65. Registration

```php
$context->strategies()->register(
    'risk_weighted',
    RiskWeightedStrategy::class
);
```

---

# 66. Strategy Semantics

Una Strategy deberá declarar capabilities.

---

# 67. StrategyCapabilities

Ejemplo:

```text
supportsGrantShortCircuit
supportsDenyShortCircuit
supportsAbstain
requiresAllVotes
orderSensitive
```

---

# 68. Why

El Planner necesita esta información para optimización segura.

---

# 69. Custom Strategy Validation

El compiler deberá verificar:

```text
strategy exists
compatible with evaluator plan
compatible with mandatory evaluators
```

---

# 70. NonBypassable Rules

Ninguna custom Strategy podrá saltarse:

```text
nonBypassable evaluators
```

---

# 71. Mandatory Pre-Phase

Antes de permitir un short-circuit de GRANT:

```text
all required mandatory security evaluators
```

deben haberse ejecutado.

---

# 72. Unsafe Strategy

Una Strategy que diga:

```text
first GRANT wins immediately
```

sin respetar mandatory evaluators será rechazada.

---

# 73. Strategy Security Validator

Podrá existir:

```php
interface AuthorizationStrategyValidatorInterface
{
    public function validate(
        StrategyDefinition $strategy,
        AuthorizationPlanTemplate $plan
    ): void;
}
```

---

# 74. Custom Authorization Attributes

Extensiones podrán crear metadata declarativa.

Ejemplo:

```php
#[RequiresSubscription('enterprise')]
final class ExportController
{
}
```

---

# 75. Attribute Contract

```php
interface AuthorizationAttributeInterface
{
}
```

---

# 76. Metadata Adapter

El Attribute por sí mismo no debería contener lógica de autorización.

---

# 77. Resolver

```php
interface AuthorizationAttributeResolverInterface
{
    public function supports(
        object $attribute
    ): bool;

    public function resolve(
        object $attribute,
        AuthorizationMetadataContext $context
    ): AuthorizationRequirement;
}
```

---

# 78. Example

```php
final class RequiresSubscriptionResolver
    implements AuthorizationAttributeResolverInterface
{
    public function resolve(
        object $attribute,
        AuthorizationMetadataContext $context
    ): AuthorizationRequirement {
        return new SubscriptionRequirement(
            $attribute->plan
        );
    }
}
```

---

# 79. Compile-Time Resolution

Los Attributes deberán convertirse a:

```text
AuthorizationRequirement
```

durante compilation.

---

# 80. No Attribute Reflection in Runtime

Compiled mode:

```text
Attribute
→ Requirement
→ Evaluator
```

se resuelve antes del request.

---

# 81. Custom Requirement Types

Paquetes podrán definir:

```php
interface AuthorizationRequirementInterface
{
}
```

---

# 82. Example

```php
final readonly class SubscriptionRequirement
    implements AuthorizationRequirementInterface
{
    public function __construct(
        public string $plan
    ) {}
}
```

---

# 83. Requirement Evaluator Mapping

```text
SubscriptionRequirement
        ↓
SubscriptionEvaluator
```

---

# 84. Registrar

```php
$context->evaluators()->forRequirement(
    SubscriptionRequirement::class,
    SubscriptionEvaluator::class
);
```

---

# 85. Requirement Immutability

Requirements deberán ser:

```text
immutable
serializable as compiled metadata
deterministic
```

---

# 86. Avoid Runtime Objects

No almacenar dentro de Requirement:

```text
Request
User
Tenant instance
Database connection
Closure
```

---

# 87. Metadata Providers

Extensiones podrán producir metadata desde fuentes distintas a PHP Attributes.

---

# 88. Sources

Ejemplos:

```text
PHP Attributes
configuration
module manifests
database-backed definitions
external policy definitions
generated metadata
```

---

# 89. Compile-Time Metadata Provider

Contrato:

```php
interface AuthorizationMetadataProviderInterface
{
    public function provide(
        AuthorizationMetadataCompilationContext $context
    ): iterable;
}
```

---

# 90. Dynamic Metadata Provider

Solo para casos genuinamente dinámicos.

---

# 91. Warning

Metadata estructural almacenada en DB y modificable durante runtime complica:

```text
compiled plans
cache invalidation
worker consistency
```

---

# 92. Recommendation

Separar:

```text
authorization structure
```

de:

```text
authorization data
```

---

# 93. Structure

Ejemplo:

```text
User must have SubscriptionRequirement.
```

---

# 94. Runtime Data

Ejemplo:

```text
Tenant#14 currently has enterprise subscription.
```

---

# 95. Provider Architecture

VoltStack deberá soportar providers especializados para datos de autorización.

---

# 96. Provider Categories

```text
PrincipalProvider
RoleProvider
PermissionProvider
AttributeProvider
RelationshipProvider
TenantAuthorizationProvider
RiskProvider
ExternalDecisionProvider
```

---

# 97. Role Provider

Contrato conceptual:

```php
interface RoleProviderInterface
{
    public function rolesFor(
        PrincipalInterface $principal,
        AuthorizationContext $context
    ): iterable;
}
```

---

# 98. Permission Provider

```php
interface PermissionProviderInterface
{
    public function permissionsFor(
        PrincipalInterface $principal,
        AuthorizationContext $context
    ): iterable;

    public function hasPermission(
        PrincipalInterface $principal,
        string $permission,
        AuthorizationContext $context
    ): bool;
}
```

---

# 99. Provider Optimization

Implementaciones no estarán obligadas a cargar todos los permisos si pueden responder directamente:

```text
hasPermission()
```

---

# 100. Batch Permission Provider

Opcional:

```php
interface BatchPermissionProviderInterface
{
    public function hasPermissions(
        PrincipalInterface $principal,
        iterable $permissions,
        AuthorizationContext $context
    ): iterable;
}
```

---

# 101. Provider Capabilities

Cada provider podrá declarar:

```text
batch support
cache support
strong consistency support
tenant-aware
external
```

---

# 102. RBAC Provider

Podrá combinar:

```text
RoleProvider
PermissionProvider
RoleHierarchyProvider
```

---

# 103. RoleHierarchyProvider

```php
interface RoleHierarchyProviderInterface
{
    public function inheritedRoles(
        string $role,
        AuthorizationContext $context
    ): iterable;
}
```

---

# 104. Multiple RBAC Sources

Una aplicación podría combinar:

```text
database roles
LDAP groups
IAM groups
application permissions
```

---

# 105. CompositePermissionProvider

Podrá existir:

```text
CompositePermissionProvider
```

---

# 106. Composition Strategy

Debe declararse:

```text
union
intersection
deny-overrides
source-priority
```

---

# 107. Default

No asumir que permisos de múltiples fuentes simplemente se suman.

---

# 108. Example

Una fuente corporativa podría contener:

```text
explicit revocations
```

que deben prevalecer.

---

# 109. ABAC Provider

Contrato:

```php
interface AuthorizationAttributeProviderInterface
{
    public function attributes(
        AuthorizationAttributeRequest $request
    ): AuthorizationAttributeSet;
}
```

---

# 110. Attribute Domains

Podrán existir:

```text
Principal attributes
Subject attributes
Environment attributes
Tenant attributes
Security attributes
```

---

# 111. Example

```text
Principal.department
Subject.classification
Environment.network_zone
Security.device_trust
```

---

# 112. Lazy ABAC Providers

No deberán resolver todos los atributos automáticamente.

---

# 113. Attribute Key API

Podrá solicitar:

```text
principal.department
```

solo cuando una regla lo requiera.

---

# 114. AttributeProviderInterface

Alternativa optimizada:

```php
interface AttributeProviderInterface
{
    public function get(
        AuthorizationAttributeKey $key,
        AuthorizationEvaluationContext $context
    ): mixed;
}
```

---

# 115. Batch Attributes

Podrá soportarse:

```text
getMany(keys)
```

---

# 116. Attribute Memoization

Valores estables dentro de la decisión/request deberán memoizarse.

---

# 117. ReBAC Provider

Contrato conceptual:

```php
interface RelationshipProviderInterface
{
    public function hasRelationship(
        RelationshipQuery $query
    ): bool;
}
```

---

# 118. Relationship Query

Podrá representar:

```text
Principal
relationship
Resource
Tenant
Context
```

---

# 119. Example

```text
User#42
is_member_of
Organization#8
```

---

# 120. Graph Provider

Implementaciones podrán usar:

```text
SQL
graph database
Redis
OpenFGA-like service
custom IAM
```

sin que Authorization Core conozca el almacenamiento.

---

# 121. Relationship Traversal

Un provider avanzado podrá implementar:

```php
interface RelationshipTraversalProviderInterface
{
    public function resolve(
        RelationshipTraversalQuery $query
    ): RelationshipResult;
}
```

---

# 122. Traversal Limits

Deberán respetar:

```text
depth
deadline
node budget
cycle detection
```

---

# 123. Batch ReBAC

Contrato:

```php
interface BatchRelationshipProviderInterface
{
    public function checkMany(
        iterable $queries
    ): iterable;
}
```

---

# 124. External Engines

VoltStack deberá poder integrarse con motores externos.

---

# 125. Examples Conceptuales

```text
OPA-style PDP
OpenFGA-style relationship engine
enterprise IAM
custom policy service
cloud authorization service
```

---

# 126. Core Independence

No deberá existir dependencia obligatoria hacia ninguno.

---

# 127. ExternalDecisionProvider

```php
interface ExternalDecisionProviderInterface
{
    public function decide(
        ExternalAuthorizationRequest $request
    ): ExternalAuthorizationResult;
}
```

---

# 128. External Adapter

Cada implementación transforma:

```text
VoltStack Authorization Request
```

al protocolo externo.

---

# 129. External Result Normalization

El adapter deberá convertir:

```text
allow
deny
indeterminate
timeout
error
```

a:

```text
GRANT
DENY
ABSTAIN
FAILURE
```

según contrato explícito.

---

# 130. Timeout

Nunca:

```text
timeout
→ GRANT
```

por defecto.

---

# 131. Default

Para evaluators required:

```text
timeout
→ FAILURE
→ fail closed
```

---

# 132. Optional External Evaluator

Podría:

```text
timeout
→ ABSTAIN
```

solo si la definición lo declara explícitamente y el security model lo permite.

---

# 133. External Provider Capabilities

Podrá declarar:

```text
supportsBatch
supportsConsistencyToken
supportsExplain
supportsRevision
supportsDeadline
```

---

# 134. Revision Token

Útil para:

```text
cache invalidation
audit
decision consistency
```

---

# 135. Connection Reuse

Persistent workers podrán reutilizar clientes externos si son lifecycle-safe.

---

# 136. Provider Lifecycle

Los providers deberán declarar:

```text
shared
request
transient
```

---

# 137. Shared Provider

Debe ser:

```text
stateless
```

respecto al request.

---

# 138. Request Provider

Puede mantener:

```text
request-local memo
```

---

# 139. Transient Provider

Deberá evitarse en hot paths salvo necesidad.

---

# 140. Provider Registry

Podrá existir:

```php
interface AuthorizationProviderRegistryInterface
{
    public function register(
        AuthorizationProviderDefinition $provider
    ): void;
}
```

---

# 141. Provider Definition

Podrá contener:

```text
name
service
type
priority
scope
capabilities
consistency
```

---

# 142. Named Providers

Ejemplo:

```text
permissions.database
permissions.ldap
relationships.openfga
attributes.risk
```

---

# 143. Provider Selection

Requirements podrán especificar:

```text
provider=permissions.database
```

o una abstracción:

```text
provider=default_permission_provider
```

---

# 144. Aliases

La aplicación podrá configurar:

```text
default_permission_provider
→ permissions.database
```

---

# 145. Environment Configuration

Así production puede utilizar:

```text
enterprise IAM
```

mientras testing usa:

```text
in-memory provider
```

---

# 146. Provider Replacement

Deberá ser explícito y validado.

---

# 147. Decorators

Providers podrán decorarse.

Ejemplo:

```text
DatabasePermissionProvider
        ↓
MemoizedPermissionProvider
        ↓
ObservedPermissionProvider
```

---

# 148. Decoration

Debe integrarse con el Container general de VoltStack.

Authorization no deberá reinventar dependency injection.

---

# 149. Custom Policy Resolver

Extensiones podrán cambiar cómo se encuentra una Policy.

---

# 150. Contract

```php
interface PolicyResolverInterface
{
    public function resolve(
        PolicyResolutionRequest $request
    ): ?PolicyReference;
}
```

---

# 151. Resolver Chain

Podría incluir:

```text
ExplicitPolicyResolver
AttributePolicyResolver
ConventionPolicyResolver
PackagePolicyResolver
FallbackPolicyResolver
```

---

# 152. Priority

Orden determinista.

---

# 153. Explicit Wins

Recomendación:

```text
explicit mapping
```

debe ganar sobre:

```text
convention discovery
```

---

# 154. Multiple Matches

No elegir arbitrariamente.

---

# 155. Ambiguous Resolution

Resultado:

```text
AmbiguousPolicyResolutionException
```

durante compilation cuando sea posible.

---

# 156. Custom Subject Resolver

Una extensión podrá definir cómo extraer Subjects.

---

# 157. Example

```php
#[Authorize(
    'update',
    subject: 'invoice'
)]
```

normalmente usa Controller arguments.

---

# 158. Custom Resolver

Podría obtener Subject desde:

```text
route binding
command object
GraphQL resolver argument
RPC message
domain action
```

---

# 159. Contract

```php
interface AuthorizationSubjectResolverInterface
{
    public function supports(
        AuthorizationSubjectReference $reference
    ): bool;

    public function resolve(
        AuthorizationSubjectResolutionContext $context
    ): mixed;
}
```

---

# 160. Compile-Time Descriptor

El resolver deberá generar cuando sea posible un:

```text
SubjectBindingDescriptor
```

---

# 161. Runtime

Solo ejecuta el binding preseleccionado.

---

# 162. Custom Principal Resolver

Podrán existir aplicaciones con:

```text
HTTP user
API client
service account
machine identity
delegated identity
```

---

# 163. Contract

```php
interface PrincipalResolverInterface
{
    public function resolve(
        PrincipalResolutionContext $context
    ): PrincipalInterface;
}
```

---

# 164. Authentication Boundary

Authorization no deberá autenticar.

El resolver recibe identity ya establecida por Authentication/Security.

---

# 165. Anonymous Principal

Podrá existir:

```text
AnonymousPrincipal
```

en lugar de utilizar `null` indiscriminadamente.

---

# 166. Custom Tenant Resolver

Multi-tenant applications podrán integrar:

```php
interface AuthorizationTenantResolverInterface
{
    public function resolve(
        AuthorizationContext $context
    ): ?TenantContext;
}
```

---

# 167. Important

El Tenant resolver no debe confiar automáticamente en:

```text
request tenant ID
```

sin validación del sistema multi-tenant.

---

# 168. Security Boundary

Authorization consume:

```text
trusted TenantContext
```

del Tenant subsystem.

---

# 169. Compiler Extensions

Packages podrán extender compilation.

---

# 170. AuthorizationCompilerPassInterface

```php
interface AuthorizationCompilerPassInterface
{
    public function process(
        AuthorizationCompilationState $state
    ): void;
}
```

---

# 171. Pass Categories

Podrán clasificarse:

```text
Discovery
Normalization
Analysis
Optimization
Validation
Generation
```

---

# 172. Pass Registration

```php
$context->compiler()->addPass(
    FinancePolicyCompilerPass::class,
    CompilerPhase::Analysis,
    priority: 100
);
```

---

# 173. Compiler State

No deberá exponerse como array mutable sin restricciones.

---

# 174. Controlled Compilation API

El pass podrá utilizar operaciones definidas como:

```text
addRequirement
addEvaluator
annotatePlan
reportWarning
reportError
```

---

# 175. Critical Metadata

Un pass de terceros no podrá:

```text
removeNonBypassableEvaluator()
```

directamente.

---

# 176. Protected Graph

El compiler deberá distinguir:

```text
extension-owned nodes
core-protected nodes
application-protected nodes
```

---

# 177. Optimization Pass

Un paquete puede optimizar sus propios requirements.

---

# 178. Example

Convertir:

```text
RequiresPlan("enterprise")
+
RequiresFeature("bulk-export")
```

en un evaluator especializado.

---

# 179. Semantic Preservation

Deberá declarar que la transformación conserva semántica.

---

# 180. Final Security Validation

Independientemente de lo declarado:

```text
final validation
```

se ejecuta después de todos los passes.

---

# 181. Compiler Pass Failure

Una excepción deberá detener:

```text
authorization compilation
```

---

# 182. No Silent Ignore

No continuar con un manifest parcialmente modificado.

---

# 183. Extension Dependencies

Un plugin podrá declarar dependencias.

---

# 184. Example

```text
FinanceAuthorizationExtension
requires:
    RBAC >= 1
    TenantAuthorization >= 1
```

---

# 185. Extension Manifest

Conceptualmente:

```php
final readonly class AuthorizationExtensionManifest
{
    public function __construct(
        public string $name,
        public string $version,
        public array $requires,
        public array $provides,
    ) {}
}
```

---

# 186. Capabilities

`provides` podría incluir:

```text
evaluator:finance-compliance
attribute:requires-subscription
provider:finance-permissions
```

---

# 187. Dependency Resolution

Antes de compilation:

```text
resolve extension graph
```

---

# 188. Missing Dependency

```text
AuthorizationExtensionDependencyException
```

---

# 189. Circular Dependency

Debe detectarse.

---

# 190. Optional Dependency

Podrá declararse como:

```text
optional
```

---

# 191. Feature Detection

En lugar de:

```php
class_exists(...)
```

dentro del runtime, usar:

```text
extension capabilities
```

resueltas durante bootstrap.

---

# 192. Extension Priority

No debe utilizarse como solución universal para conflictos.

---

# 193. Priority Is For Ordering

Ejemplos:

```text
compiler passes
resolver chains
decorators
```

---

# 194. Priority Is Not For

```text
silently overriding duplicate Policy
```

---

# 195. Deterministic Tie Breaking

Si dos extensiones tienen misma prioridad:

```text
stable canonical ordering
```

deberá aplicarse.

---

# 196. Extension Namespace

Cada extension deberá poseer un ID canonical.

Ejemplo:

```text
vendor.finance.authorization
```

---

# 197. Component IDs

Sus componentes podrán usar:

```text
vendor.finance.authorization:compliance
```

internamente.

---

# 198. Avoid Collision

El compiler validará IDs globales.

---

# 199. Human Alias

Podrán existir aliases cortos si no generan conflicto.

---

# 200. Custom Reason Codes

Extensions podrán registrar reason codes.

---

# 201. Example

```text
finance.limit.exceeded
finance.account.suspended
subscription.plan.required
```

---

# 202. Namespace Reason Codes

Recomendado para evitar colisiones.

---

# 203. Reason Registry

Podrá existir:

```text
AuthorizationReasonRegistry
```

---

# 204. Reason Metadata

Podrá declarar:

```text
code
category
safeForClient
auditSeverity
```

---

# 205. Security

El mensaje interno no deberá convertirse automáticamente en mensaje HTTP.

---

# 206. Custom Denial Mappers

Extensiones podrán mapear reason codes a respuestas de aplicación.

---

# 207. Boundary

El Authorization Core produce:

```text
Decision
```

El HTTP layer produce:

```text
403
404
redirect
API error
```

---

# 208. Custom Failure Handlers

Podrán registrarse en integration layers.

No deben alterar una decisión DENY a GRANT.

---

# 209. Custom Audit Extensions

Una extensión podrá enriquecer:

```text
AuthorizationAuditRecord
```

---

# 210. Audit Enricher

```php
interface AuthorizationAuditEnricherInterface
{
    public function enrich(
        AuthorizationAuditRecordBuilder $record,
        AuthorizationAuditContext $context
    ): void;
}
```

---

# 211. Restrictions

No deberá modificar:

```text
decision
ability
principal identity
tenant identity
```

del evento original.

---

# 212. Audit Sink

Paquetes podrán proporcionar:

```text
database
file
SIEM
OpenTelemetry
message broker
```

sinks.

---

# 213. Sink Contract

```php
interface AuthorizationAuditSinkInterface
{
    public function write(
        AuthorizationAuditRecord $record
    ): void;
}
```

---

# 214. Required vs BestEffort

Cada sink deberá declarar su delivery semantics.

---

# 215. Observability Extensions

Podrán registrar:

```text
metrics exporters
trace enrichers
profilers
diagnostic collectors
```

---

# 216. Trace Enricher

No debe ejecutar lógica de negocio costosa por default.

---

# 217. Lazy Diagnostics

Los detalles costosos se obtendrán:

```text
on error
on sampled trace
on explicit explain
```

---

# 218. Custom Explain Providers

External engines podrán proporcionar explicación propia.

---

# 219. Sanitization

Toda explicación externa deberá pasar por:

```text
AuthorizationExplanationSanitizer
```

antes de exponerse.

---

# 220. Extension Security Levels

VoltStack podría clasificar extensiones:

```text
Application
Trusted
ThirdParty
Experimental
```

---

# 221. Important

Esto no sustituye el modelo de confianza real de PHP.

Código Composer instalado posee capacidad de ejecutar código de aplicación.

---

# 222. Purpose

La clasificación sirve para:

```text
diagnostics
policy
compiler restrictions
security review
```

no como sandbox real.

---

# 223. No Fake Sandbox

VoltStack no deberá afirmar que puede aislar completamente código PHP arbitrario cargado en el mismo proceso.

---

# 224. Trusted Extensions

Podrán acceder a APIs avanzadas si la aplicación lo permite.

---

# 225. Third-Party Extensions

Deberán utilizar APIs públicas y estables.

---

# 226. Internal APIs

Namespaces como:

```text
VoltStack\Quantum\Authorization\Internal
```

no serán contratos públicos.

---

# 227. Public SPI

VoltStack deberá definir una:

```text
Service Provider Interface
```

estable para extensiones.

---

# 228. SPI Components

Incluye:

```text
interfaces
registrars
definitions
contexts
result contracts
compiler extension APIs
```

---

# 229. Versioning

La SPI deberá versionarse cuidadosamente.

---

# 230. Breaking Changes

Cambios en:

```text
AuthorizationEvaluatorInterface
```

son mucho más sensibles que cambios internos.

---

# 231. Capability Versioning

Un extension manifest podrá declarar:

```text
authorization-spi: ^1.0
```

---

# 232. Compiler Compatibility

Un artifact compilado deberá guardar:

```text
SPI version
runtime format version
```

cuando sea necesario.

---

# 233. Unsupported Extension

Debe fallar durante:

```text
bootstrap
compile
```

no durante el primer request protegido.

---

# 234. Extension Validation

Cada extensión podrá tener un validator.

---

# 235. Global Validation

El AuthorizationCompiler también validará:

```text
duplicate IDs
unknown dependencies
unknown strategies
invalid evaluator scope
invalid phases
invalid requirements
```

---

# 236. Security Validation

Además:

```text
missing mandatory evaluator
attempted protected replacement
unsafe strategy
tenant boundary violation
```

---

# 237. Provider Validation

Ejemplo:

```text
Shared provider
depends on request-scoped service
```

deberá detectarse mediante integración con Container.

---

# 238. FrankenPHP Validation

Especialmente importante.

---

# 239. Extension Runtime State

Nunca almacenar en singleton:

```text
current user
current tenant
current subject
current authorization request
```

---

# 240. Request-Local State

Usar:

```text
AuthorizationSession
ExecutionContext
runtime context storage
```

---

# 241. Static Properties

Evitar:

```php
private static ?User $currentUser;
```

---

# 242. Worker Leakage

Esto podría mezclar usuarios entre requests persistentes.

---

# 243. Extension Certification Tests

VoltStack podrá proporcionar:

```text
AuthorizationExtensionTestCase
```

---

# 244. Example

```php
final class FinanceExtensionTest
    extends AuthorizationExtensionTestCase
{
}
```

---

# 245. Standard Contract Tests

Podrán verificar:

```text
registration
compilation
runtime equivalence
failure behavior
worker safety
serialization
determinism
```

---

# 246. Evaluator Contract Test

Debe verificar:

```text
GRANT
DENY
ABSTAIN
FAILURE
```

cuando aplique.

---

# 247. Purity Test

Podrá detectar algunos side effects conocidos, aunque no garantizar pureza absoluta.

---

# 248. Batch Equivalence

Si un provider implementa batch:

```text
batch result
=
individual result
```

deberá verificarse.

---

# 249. Compiled Equivalence

```text
dynamic extension execution
=
compiled extension execution
```

---

# 250. Worker Reuse Test

Ejecutar:

```text
Request A
Principal A

Request B
Principal B
```

en el mismo runtime y verificar ausencia de leakage.

---

# 251. Tenant Isolation Test

Especialmente para providers multi-tenant.

---

# 252. Extension Performance Test

Podrá establecer budgets.

---

# 253. Custom Evaluator Benchmark

Medir:

```text
cold
warm
memoized
batch
```

---

# 254. Extension Diagnostics

Comando propuesto:

```text
volt authorization:extensions
```

---

# 255. Output

Ejemplo:

```text
Authorization Extensions

VoltStack Core Authorization
version: 1.0

Finance Authorization
version: 2.1

Provides:
  evaluator: finance-compliance
  attribute: requires-subscription
  provider: finance-permissions

Status:
  compiled
  validated
```

---

# 256. Extension Inspect

```text
volt authorization:extension finance
```

---

# 257. Puede mostrar

```text
Provider
Dependencies
Abilities
Policies
Evaluators
Strategies
Compiler Passes
Decorators
Runtime Scopes
```

---

# 258. Security Diagnostics

También:

```text
Protected components replaced:
none

Dynamic evaluators:
1

Transient providers:
0

FrankenPHP unsafe services:
0
```

---

# 259. Registry Diagnostics

Comando:

```text
volt authorization:registry
```

podrá mostrar componentes por registry.

---

# 260. Resolver Diagnostics

```text
volt authorization:resolve-policy Invoice
```

podrá explicar:

```text
Explicit mapping:
Invoice → InvoicePolicy

Source:
FinanceAuthorizationProvider
```

---

# 261. Provider Diagnostics

```text
volt authorization:providers
```

---

# 262. Example

```text
permissions.default
→ DatabasePermissionProvider
scope: shared
batch: yes
tenant-aware: yes
```

---

# 263. Explain Extension Contribution

El sistema de explain podrá indicar:

```text
Decision DENY

Evaluator:
FinanceComplianceEvaluator

Provided by:
vendor.finance.authorization

Reason:
finance.limit.exceeded
```

---

# 264. Provenance

Cada compiled component deberá conservar:

```text
extension/provider provenance
```

cuando sea útil.

---

# 265. Production Compactness

Puede conservarse como:

```text
extension ID
```

en vez de metadata verbose.

---

# 266. Extension Uninstallation

Eliminar un package deberá requerir:

```text
recompile authorization
```

---

# 267. Existing Workers

Deberán reiniciarse/gracefully reload.

---

# 268. No Runtime Removal

No:

```text
unregister evaluator
```

en worker activo.

---

# 269. Deployment

```text
install/update packages
        ↓
build container
        ↓
compile authorization
        ↓
validate
        ↓
atomic deploy
        ↓
reload workers
```

---

# 270. Extension Update

Un cambio de versión puede modificar:

```text
Policy
Evaluator
Strategy
Provider
```

y por ello invalidar:

```text
plans
registry version
decision caches
```

según el componente.

---

# 271. Registry Version

Debe cambiar cuando cambia semántica estructural.

---

# 272. Provider Data Version

Un provider podrá suministrar además:

```text
revision
```

para cambios de datos.

---

# 273. Structural vs Data Version

```text
registry version
→ authorization structure

provider revision
→ authorization data
```

---

# 274. Cache Keys

Podrán incorporar ambas según consistency requirements.

---

# 275. Dynamic Provider Data

No obliga a recompilar mientras estructura sea igual.

---

# 276. Example

Agregar un permiso a User:

```text
provider data revision changes
```

pero no:

```text
authorization manifest
```

---

# 277. Adding New Ability

Sí requiere:

```text
structural recompilation
```

si está definida en código/config compilada.

---

# 278. Custom Evaluator Example Completo

Supongamos un paquete empresarial que requiere verificar el nivel de suscripción.

Attribute:

```php
#[Attribute(Attribute::TARGET_METHOD)]
final readonly class RequiresSubscription
    implements AuthorizationAttributeInterface
{
    public function __construct(
        public string $plan
    ) {}
}
```

---

# 279. Requirement

```php
final readonly class SubscriptionRequirement
    implements AuthorizationRequirementInterface
{
    public function __construct(
        public string $plan
    ) {}
}
```

---

# 280. Resolver

```php
final class SubscriptionAttributeResolver
    implements AuthorizationAttributeResolverInterface
{
    public function resolve(
        object $attribute,
        AuthorizationMetadataContext $context
    ): AuthorizationRequirementInterface {
        return new SubscriptionRequirement(
            $attribute->plan
        );
    }
}
```

---

# 281. Provider

```php
interface SubscriptionProviderInterface
{
    public function hasPlan(
        TenantContext $tenant,
        string $plan
    ): bool;
}
```

---

# 282. Evaluator

```php
final class SubscriptionEvaluator
    implements AuthorizationEvaluatorInterface
{
    public function __construct(
        private SubscriptionProviderInterface $subscriptions
    ) {}

    public function evaluate(
        AuthorizationEvaluationContext $context
    ): AuthorizationDecision {
        $requirement = $context->requirement();

        if (! $requirement instanceof SubscriptionRequirement) {
            return AuthorizationDecision::abstain();
        }

        $tenant = $context->tenant();

        if ($tenant === null) {
            return AuthorizationDecision::deny(
                'subscription.tenant_required'
            );
        }

        if (! $this->subscriptions->hasPlan(
            $tenant,
            $requirement->plan
        )) {
            return AuthorizationDecision::deny(
                'subscription.plan_required'
            );
        }

        return AuthorizationDecision::grant();
    }
}
```

---

# 283. Provider Registration

```php
final class SubscriptionAuthorizationProvider
    extends AuthorizationServiceProvider
{
    public function register(
        AuthorizationRegistrationContext $context
    ): void {
        $context->metadata()->attribute(
            RequiresSubscription::class,
            SubscriptionAttributeResolver::class
        );

        $context->evaluators()->forRequirement(
            SubscriptionRequirement::class,
            SubscriptionEvaluator::class
        );

        $context->providers()->bind(
            SubscriptionProviderInterface::class,
            DatabaseSubscriptionProvider::class
        );
    }
}
```

---

# 284. Controller

```php
#[RequiresSubscription('enterprise')]
#[Authorize(
    'export',
    subject: 'report'
)]
public function export(
    Report $report
): Response {
    // ...
}
```

---

# 285. Compilation

VoltStack transforma:

```text
RequiresSubscription("enterprise")
```

en:

```text
SubscriptionRequirement("enterprise")
```

---

# 286. Plan

```text
TenantIsolationEvaluator
SubscriptionEvaluator
ReportPolicy
```

---

# 287. Runtime

No existe:

```text
Attribute Reflection
resolver discovery
provider discovery
```

---

# 288. Runtime Path

```text
PlanTemplate
    ↓
SubscriptionEvaluator
    ↓
SubscriptionProvider
    ↓
Decision
```

---

# 289. Symfony-Style Voter Example

Una extensión podría declarar:

```php
final class DocumentVoter
    implements AuthorizationVoterInterface
{
    public function supports(
        Ability $ability,
        mixed $subject
    ): bool {
        return $subject instanceof Document
            && in_array(
                $ability->name(),
                ['document.view', 'document.edit'],
                true
            );
    }

    public function vote(
        AuthorizationEvaluationContext $context
    ): AuthorizationDecision {
        // ...
    }
}
```

---

# 290. Compilation Optimization

El compiler podrá inferir:

```text
Document + document.view
→ DocumentVoter

Document + document.edit
→ DocumentVoter
```

evitando recorrer todos los voters.

---

# 291. Laravel-Style Policy Example

```php
final class DocumentPolicy
{
    public function update(
        User $user,
        Document $document
    ): bool {
        return $document->ownerId === $user->id;
    }
}
```

---

# 292. Registration

```php
$context->policies()->register(
    Document::class,
    DocumentPolicy::class
);
```

---

# 293. Unified Runtime

Tanto:

```text
Laravel-style Policy
```

como:

```text
Symfony-style Voter
```

terminan normalizados a:

```text
Authorization Evaluators
        ↓
Decision Manager
```

---

# 294. Valor arquitectónico

VoltStack no necesita elegir exclusivamente entre:

```text
Laravel Policies
```

y:

```text
Symfony Voters
```

Puede soportar ambos modelos sobre un Core común.

---

# 295. Extension Composition

Ejemplo empresarial:

```text
Invoice Authorization
        │
        ├── Core Tenant Isolation
        ├── RBAC Permission Evaluator
        ├── Application InvoicePolicy
        ├── Finance Compliance Plugin
        ├── ReBAC Organization Provider
        └── Risk ABAC Provider
```

---

# 296. Decision Pipeline

Todos producen:

```text
AuthorizationDecision
```

y participan según:

```text
Plan
Phase
Priority
Strategy
Mandatory rules
```

---

# 297. No Special Plugin Path

Plugins no deben ejecutarse por una ruta paralela.

---

# 298. Same Engine

```text
Core Evaluator
Application Evaluator
Package Evaluator
External Evaluator
```

todos pasan por:

```text
AuthorizationExecutionEngine
```

---

# 299. Razón

Esto conserva:

```text
observability
failure handling
audit
short-circuit rules
security invariants
```

---

# 300. Extension Boundaries

Una extensión no deberá acceder directamente a:

```text
private AuthorizationManager state
mutable DecisionManager internals
worker-global Principal state
raw registry mutation
```

---

# 301. Public Context Objects

Se expondrán:

```text
RegistrationContext
CompilationContext
EvaluationContext
AuditContext
ExplanationContext
```

con APIs controladas.

---

# 302. Context Immutability

Los contextos de ejecución deberán ser:

```text
readonly / immutable
```

cuando sea práctico.

---

# 303. Extension Errors

Se distinguirán:

```text
Registration Error
Compilation Error
Provider Failure
Evaluator Failure
External Failure
Configuration Error
```

---

# 304. Runtime Evaluator Exception

No deberá propagarse arbitrariamente.

---

# 305. Normalization

```text
Evaluator exception
        ↓
Authorization FAILURE
        ↓
Failure Policy
        ↓
fail closed
```

según documento 16.

---

# 306. Extension Exception Metadata

Podrá incluir:

```text
extension ID
component ID
provider ID
```

para diagnóstico interno.

---

# 307. Client Safety

No exponer automáticamente:

```text
package class
filesystem path
stack trace
provider endpoint
```

al cliente.

---

# 308. Circuit Breakers

External providers podrán integrarse con mecanismos de resiliencia.

---

# 309. Important

Circuit breaker abierto no significa:

```text
GRANT
```

---

# 310. Required Provider

```text
circuit open
→ FAILURE
```

---

# 311. Optional Provider

Podrá:

```text
ABSTAIN
```

si fue diseñado explícitamente así.

---

# 312. Provider Retry

Debe respetar:

```text
authorization deadline
```

---

# 313. Avoid Retry Storm

Especialmente en workers concurrentes.

---

# 314. Bulkheads

External authorization providers podrán usar límites de concurrencia.

---

# 315. Authorization Core

No necesita implementar todos los mecanismos de resiliencia.

Puede reutilizar:

```text
VoltStack Resilience System
```

---

# 316. Custom Cache Providers

Extensions podrán aportar almacenamiento de cache, pero deberán utilizar los contratos del Authorization Cache System.

---

# 317. No Custom Cache Semantics

Un plugin no deberá decidir arbitrariamente:

```text
GRANT forever
```

---

# 318. Cacheability

La Ability/Decision/Provider metadata sigue gobernando.

---

# 319. Extension Configuration

Cada package podrá publicar configuración.

Ejemplo conceptual:

```php
return [
    'provider' => 'database',

    'cache' => true,

    'fail_mode' => 'closed',
];
```

---

# 320. Security Configuration

Valores peligrosos deberán validarse.

---

# 321. Example

```text
required_external_policy.fail_mode = allow
```

podría ser rechazado.

---

# 322. Config Compilation

Configuración estructural deberá formar parte del:

```text
authorization fingerprint
```

---

# 323. Runtime Configuration

Datos dinámicos podrán venir de providers.

---

# 324. Extension Discovery Modes

Podrán existir:

```text
explicit
package auto-discovery
module discovery
```

---

# 325. Production Recommendation

Todo discovery debe completarse antes de generar compiled manifest.

---

# 326. Disable Discovery

Aplicaciones hardened podrán deshabilitar:

```text
package auto-discovery
```

y registrar extensions explícitamente.

---

# 327. Allowlist

Podrá existir:

```text
authorization.extensions.allowed
```

---

# 328. Denylist

También:

```text
authorization.extensions.disabled
```

---

# 329. Security Review

Enterprise deployments podrán requerir:

```text
explicit extension allowlist
```

---

# 330. Extension Manifest Hash

Podrá incluirse en build fingerprint.

---

# 331. Supply Chain

Authorization extensions deberán tratarse como código privilegiado.

---

# 332. Package Installation

Composer package desconocido con acceso a Authorization debe someterse a las mismas prácticas de seguridad que cualquier dependency PHP.

---

# 333. No Security Theater

La Extension API mejora control arquitectónico, no convierte paquetes maliciosos en seguros.

---

# 334. Testing Fake Providers

VoltStack deberá ofrecer:

```text
InMemoryRoleProvider
InMemoryPermissionProvider
InMemoryRelationshipProvider
InMemoryAttributeProvider
```

para testing.

---

# 335. Fake External Engine

Podrá configurarse:

```php
FakeExternalDecisionProvider::deny(
    'finance.export'
);
```

---

# 336. Provider Swap in Tests

La aplicación podrá reemplazar:

```text
Production provider
```

por:

```text
Fake provider
```

antes de registry seal.

---

# 337. No Test Mutation After Boot

Preferir rebuild del test application context a mutaciones inseguras.

---

# 338. Extension Test Harness

Podrá compilar una aplicación mínima:

```text
Core
+
Extension Under Test
```

---

# 339. Validation

Así puede probar:

```text
manifest generation
component registration
plan generation
runtime execution
```

---

# 340. Developer Experience

La API deberá ser potente pero sencilla para casos comunes.

---

# 341. Simple Custom Evaluator

Idealmente:

```php
final class SubscriptionEvaluator
    implements AuthorizationEvaluatorInterface
{
    public function evaluate(
        AuthorizationEvaluationContext $context
    ): AuthorizationDecision {
        // ...
    }
}
```

más:

```php
$authorization
    ->evaluators()
    ->register(
        'subscription',
        SubscriptionEvaluator::class
    );
```

---

# 342. Advanced API

Solo casos complejos necesitarán:

```text
compiler passes
capabilities
dependency masks
custom strategies
provider revisions
```

---

# 343. Progressive Complexity

Filosofía:

```text
simple things should be simple

advanced things should remain possible
```

---

# 344. Laravel Familiarity

Developers Laravel reconocerán:

```text
Policies
Gates
Service Providers
explicit registrations
```

---

# 345. Symfony Familiarity

Developers Symfony reconocerán:

```text
Voters
strategies
service-based evaluators
attributes
compiler passes
```

---

# 346. VoltStack Extension

Sobre ambos modelos VoltStack añadirá:

```text
compiled plans
provider abstraction
RBAC/ABAC/ReBAC unification
multi-tenancy
persistent runtime safety
extension manifests
security validation
```

---

# 347. Proposed Namespace

Conceptualmente:

```text
VoltStack\Quantum\Authorization\Contracts
VoltStack\Quantum\Authorization\Extension
VoltStack\Quantum\Authorization\Provider
VoltStack\Quantum\Authorization\Evaluator
VoltStack\Quantum\Authorization\Compiler
VoltStack\Quantum\Authorization\Metadata
```

---

# 348. Possible Structure

```text
Authorization/
├── Contracts/
│   ├── AuthorizationExtensionInterface.php
│   ├── AuthorizationEvaluatorInterface.php
│   ├── AuthorizationVoterInterface.php
│   ├── AuthorizationDecisionStrategyInterface.php
│   ├── PermissionProviderInterface.php
│   ├── RoleProviderInterface.php
│   ├── AttributeProviderInterface.php
│   └── RelationshipProviderInterface.php
│
├── Extension/
│   ├── AuthorizationServiceProvider.php
│   ├── AuthorizationExtensionManifest.php
│   ├── AuthorizationRegistrationContext.php
│   └── AuthorizationBootContext.php
│
├── Registry/
│   ├── EvaluatorRegistry.php
│   ├── ProviderRegistry.php
│   ├── StrategyRegistry.php
│   └── ReasonRegistry.php
│
├── Provider/
│   ├── CompositePermissionProvider.php
│   ├── CompositeRoleProvider.php
│   └── ProviderDefinition.php
│
├── Evaluator/
│   ├── EvaluatorDefinition.php
│   └── EvaluatorCapabilities.php
│
├── Metadata/
│   ├── AuthorizationAttributeResolverInterface.php
│   └── AuthorizationRequirementInterface.php
│
├── Compiler/
│   ├── AuthorizationCompilerPassInterface.php
│   ├── CompilerPhase.php
│   └── ExtensionValidationPass.php
│
└── Testing/
    ├── AuthorizationExtensionTestCase.php
    ├── InMemoryPermissionProvider.php
    └── InMemoryRelationshipProvider.php
```

---

# 349. Core Interfaces vs Implementations

Los contratos públicos deberán vivir separados de implementaciones internas.

---

# 350. Dependency Direction

```text
Application Extension
        ↓
Authorization Public Contracts
        ↓
Authorization Core
```

Nunca:

```text
Authorization Core
        ↓
Application Extension
```

---

# 351. Package Dependency

Packages dependen del SPI.

Core no conoce packages concretos.

---

# 352. Extension Registry Invariants

### Invariante 1

Toda extensión se registra antes del registry seal.

### Invariante 2

IDs son únicos.

### Invariante 3

Conflictos no se resuelven silenciosamente.

### Invariante 4

Overrides son explícitos.

### Invariante 5

Protected components no pueden reemplazarse sin autorización.

---

# 353. Evaluator Invariants

### Invariante 1

Evaluators son side-effect free.

### Invariante 2

Devuelven decisiones normalizables.

### Invariante 3

No mutan Principal, Subject o Tenant.

### Invariante 4

No pueden bypassar mandatory evaluators.

### Invariante 5

Sus dependencies y phase deben declararse correctamente.

---

# 354. Provider Invariants

### Invariante 1

Providers respetan el lifecycle declarado.

### Invariante 2

Shared providers no contienen request state.

### Invariante 3

Provider failures nunca se convierten implícitamente en GRANT.

### Invariante 4

Batch y single evaluation deben ser equivalentes.

### Invariante 5

Provider revision debe reflejar cambios relevantes cuando participe en caching.

---

# 355. Compiler Extension Invariants

### Invariante 1

Compiler passes tienen orden determinista.

### Invariante 2

No pueden eliminar security boundaries protegidos.

### Invariante 3

Toda optimización se somete a validación final.

### Invariante 4

Compiler errors detienen el build.

### Invariante 5

Mismo input produce estructura semánticamente equivalente.

---

# 356. Plugin Invariants

### Invariante 1

Plugins no tienen un runtime paralelo.

### Invariante 2

Utilizan el mismo Decision Manager.

### Invariante 3

Utilizan el mismo Failure System.

### Invariante 4

Utilizan el mismo Audit/Tracing System.

### Invariante 5

Utilizan las mismas reglas de cache y consistency.

---

# 357. Multi-Tenant Invariants

### Invariante 1

Un provider tenant-aware recibe TenantContext validado.

### Invariante 2

Nunca deriva confianza únicamente de input HTTP.

### Invariante 3

Caches incluyen Tenant cuando corresponde.

### Invariante 4

Shared provider state nunca mezcla tenants.

### Invariante 5

Custom extensions no pueden eliminar TenantIsolation mandatory rules.

---

# 358. FrankenPHP Invariants

### Invariante 1

Extension metadata puede compartirse si es immutable.

### Invariante 2

Shared evaluators son stateless.

### Invariante 3

Shared providers son stateless respecto al request.

### Invariante 4

No se usan static properties para current authorization state.

### Invariante 5

Cada request libera sus referencias dinámicas.

---

# 359. External Provider Invariants

### Invariante 1

Timeout nunca significa GRANT por defecto.

### Invariante 2

Required provider failure produce FAILURE.

### Invariante 3

Deadlines se propagan.

### Invariante 4

Payloads contienen únicamente datos necesarios.

### Invariante 5

External decisions se normalizan antes de entrar al Decision Manager.

---

# 360. Arquitectura general

```text
                  APPLICATION / PACKAGES
                           │
          ┌────────────────┼────────────────┐
          ↓                ↓                ↓
     Policies           Voters        Custom Attributes
          │                │                │
          ├────────────────┼────────────────┤
          ↓                ↓                ↓
      Evaluators       Providers       Requirements
          │                │                │
          └────────────────┼────────────────┘
                           ↓
              Authorization Extension SPI
                           │
                           ↓
                 Registration Context
                           │
                           ↓
                    Registries
                           │
                           ↓
               Authorization Compiler
                           │
                           ↓
                  Security Validation
                           │
                           ↓
                    Registry Seal
                           │
                           ↓
               Compiled Authorization
                           │
                           ↓
              Authorization Runtime Engine
```

---

# 361. Provider Architecture

```text
                 Authorization Evaluator
                         │
            ┌────────────┼─────────────┐
            ↓            ↓             ↓
          RBAC          ABAC           ReBAC
            │            │             │
            ↓            ↓             ↓
      Permission      Attribute    Relationship
       Provider        Provider       Provider
            │            │             │
            └────────────┼─────────────┘
                         ↓
                Application / DB /
                IAM / Graph / PDP
```

---

# 362. Extension Compilation Architecture

```text
Package Discovery
        ↓
Extension Discovery
        ↓
Dependency Resolution
        ↓
Provider Registration
        ↓
Policy/Gate Registration
        ↓
Evaluator Registration
        ↓
Metadata Registration
        ↓
Compiler Pass Registration
        ↓
Normalization
        ↓
Conflict Detection
        ↓
Compilation
        ↓
Extension Passes
        ↓
Optimization
        ↓
FINAL SECURITY VALIDATION
        ↓
Registry Seal
        ↓
Manifest
```

---

# 363. Runtime Architecture

```text
AuthorizationRequest
        ↓
Compiled PlanTemplate
        ↓
┌──────────────────────────────────┐
│ Core Evaluators                  │
│ Application Policies             │
│ Package Evaluators               │
│ Voters                           │
│ RBAC Providers                   │
│ ABAC Providers                   │
│ ReBAC Providers                  │
│ External Policy Engines          │
└──────────────────────────────────┘
        ↓
Normalized Decisions
        ↓
DecisionManager
        ↓
GRANT / DENY / FAILURE
        ↓
Audit / Trace / Explain
```

---

# 364. Filosofía de extensibilidad

VoltStack deberá evitar dos extremos.

No deberá ser:

```text
closed authorization framework
```

donde cada integración requiera modificar Core.

Pero tampoco:

```text
unrestricted mutable authorization container
```

donde cualquier package pueda alterar reglas críticas en mitad de un request.

La filosofía será:

```text
Extensible before runtime.

Controlled during compilation.

Immutable during execution.
```

---

# 365. Modelo Laravel + Symfony + VoltStack

El sistema podrá combinar:

```text
Laravel
├── Policies
├── Gates
└── Service Providers

Symfony
├── Voters
├── Attributes
├── Decision Strategies
└── Compiler Passes

VoltStack
├── Unified Authorization Planner
├── Evaluator Pipeline
├── RBAC Providers
├── ABAC Providers
├── ReBAC Providers
├── Multi-Tenant Security
├── External PDP Integration
├── Compiled Authorization Plans
├── Extension Manifests
├── Registry Sealing
└── FrankenPHP-safe Runtime
```

---

# 366. Principio definitivo

```text
Authorization extensibility must happen
through contracts, not shortcuts.

Packages may add authorization knowledge.

Providers may supply authorization data.

Evaluators may participate in decisions.

Strategies may aggregate decisions.

Compiler passes may transform authorization structure.

But every extension must ultimately pass through
the same validation, planning, execution,
failure, audit and security boundaries
as the VoltStack Core itself.
```

---

# 367. Resultado esperado

El `Authorization Extensibility, Plugin, Provider and Custom Evaluator System` permitirá construir sobre VoltStack integraciones como:

```text
Corporate LDAP Authorization
Active Directory Role Provider
Database RBAC
OpenFGA-style ReBAC
OPA-style Policy Engine
Subscription Authorization
Financial Compliance Policies
Geo Restrictions
Device Trust
Risk-Based Authorization
API Client Authorization
Organization Membership
Feature Entitlements
Custom Tenant Policies
```

sin acoplar el Authorization Core a ninguna implementación concreta.

La arquitectura final será:

```text
                AUTHORIZATION CORE
                       │
                       ↓
                PUBLIC EXTENSION SPI
                       │
      ┌────────────────┼─────────────────┐
      ↓                ↓                 ↓
 APPLICATION        QUANTUM           PACKAGES
      │                │                 │
      └────────────────┼─────────────────┘
                       ↓
                 REGISTRATION
                       ↓
                  COMPILATION
                       ↓
                  VALIDATION
                       ↓
                 SEALED GRAPH
                       ↓
                RUNTIME ENGINE
```

De esta manera, VoltStack podrá ofrecer un sistema de autorización extensible comparable conceptualmente con la familiaridad de **Laravel Policies/Gates** y la flexibilidad de **Symfony Voters/Decision Strategies**, pero integrado dentro de una arquitectura más amplia de compilación, multi-tenancy, RBAC/ABAC/ReBAC, observabilidad y workers persistentes.

La regla final será:

```text
Extensions can extend authorization.

They cannot escape authorization.
```