# VoltStack Authorization System

## Configuration, Bootstrap and Service Container Integration System

**Documento:** `25_AUTHORIZATION_CONFIGURATION_BOOTSTRAP_AND_SERVICE_CONTAINER_INTEGRATION_SYSTEM.md`
**Sistema:** Authorization
**Framework:** VoltStack
**Módulo sugerido:** `Quantum/Authorization`
**Estado:** Especificación arquitectónica
**Versión objetivo:** 1.x+

---

# 1. Propósito
Este documento define cómo el sistema de autorización de VoltStack será:
configurado
registrado
compilado
inicializado
conectado al Service Container
preparado para runtime
aislado por request
cerrado correctamente
El objetivo es establecer la frontera entre:
Authorization Domain
y:
Framework Bootstrap Infrastructure
de manera que Policies, Gates, Voters, evaluadores RBAC/ABAC/ReBAC, scopes, tenant isolation, risk evaluators, approval workflows y demás componentes puedan utilizarse sin conocer cómo fueron construidos.
# 2. Problema arquitectónico
Hasta este punto hemos definido numerosos subsistemas:
AuthorizationManager
DecisionManager
PolicyRegistry
PolicyResolver
PolicyDispatcher
Gate
AbilityRegistry
Voters
AuthorizationPlanner
AuthorizationContext
RBAC
ABAC
ReBAC
Tenant Isolation
Decision Cache
Audit
Tracing
Failure Handling
Compilation
Plugins
Delegation
Ownership
Scopes
Risk
Approval
SoD
Pero todavía necesitamos responder:
¿Quién crea estos objetos?

¿Quién registra sus implementaciones?

¿Quién descubre Policies?

¿Quién compila metadata?

¿Quién construye los pipelines?

¿Quién selecciona adapters?

¿Cuándo ocurre todo esto?

¿Qué es singleton?

¿Qué es request-scoped?

¿Qué puede persistir en FrankenPHP?

¿Qué debe resetearse entre requests?

¿Cómo se sobreescriben componentes?

¿Cómo registra componentes un plugin?

¿Cómo se valida la configuración?

¿Cómo se genera el runtime optimizado?
Este documento responde esas preguntas.
# 3. Principio fundamental
VoltStack deberá separar estrictamente:
CONFIGURATION
      ↓
REGISTRATION
      ↓
DISCOVERY
      ↓
COMPILATION
      ↓
BOOTSTRAP
      ↓
RUNTIME
No deberán mezclarse estas fases.
# 4. Modelo general
Application Configuration
        │
        ↓
Authorization Configuration Loader
        │
        ↓
Configuration Validation
        │
        ↓
Service Registration
        │
        ↓
Policy / Metadata Discovery
        │
        ↓
Authorization Compilation
        │
        ↓
Compiled Authorization Manifest
        │
        ↓
Runtime Bootstrap
        │
        ↓
Authorization Runtime
# 5. Objetivos
El sistema deberá proporcionar:
Zero-config sensible defaults
Explicit configuration
Environment overrides
Tenant-aware configuration
Compiled production configuration
Container integration
Service replacement
Plugin registration
Lazy services
Request-scoped state
Persistent-worker safety
Runtime reset
Dependency validation
Configuration diagnostics
# 6. No mezclar AuthN y AuthZ
El bootstrap deberá mantener separados:
Authentication
y:
Authorization
Authentication determina:
Who are you?
Authorization determina:
What may you do?
El Authorization bootstrap podrá depender de contratos de identidad producidos por Authentication, pero no deberá inicializar el sistema de autenticación internamente.
# 7. Dependencias conceptuales
Authorization podrá consumir contratos de:
Container
Configuration
Cache
Events
Telemetry
Database
Authentication Identity
Tenant Context
Clock
Metadata
Compiler
pero deberá evitar dependencias circulares.
# 8. Dirección de dependencias
Modelo recomendado:
Framework Core
      ↑
Authorization Contracts
      ↑
Authorization Core
      ↑
Authorization Infrastructure
      ↑
Application Policies
# 9. Configuration Root
Configuración conceptual:
return [

    'enabled' => true,

    'default_strategy' => 'affirmative',

    'policies' => [
        'discovery' => true,
    ],

    'gates' => [
        'enabled' => true,
    ],

    'cache' => [
        'enabled' => true,
    ],

    'audit' => [
        'enabled' => true,
    ],

    'multi_tenancy' => [
        'enabled' => true,
    ],

];
# 10. Archivo sugerido
VoltStack podrá utilizar:
config/authorization.php
como configuración principal.
# 11. Configuración declarativa
La configuración deberá describir:
qué utilizar
no:
cómo construir manualmente todo el object graph
# 12. Evitar
'manager' => new AuthorizationManager(
    new DecisionManager(
        new PolicyResolver(...)
    )
)
# 13. Preferir
'decision_strategy' => 'affirmative',
y dejar que el Container construya las dependencias.
# 14. AuthorizationConfig
El runtime no deberá consumir directamente arrays arbitrarios.
Deberá existir un modelo tipado.
final readonly class AuthorizationConfig
{
    public function __construct(
        public bool $enabled,
        public string $defaultStrategy,
        public PolicyConfig $policies,
        public GateConfig $gates,
        public AuthorizationCacheConfig $cache,
        public AuditConfig $audit,
        public TenantAuthorizationConfig $multiTenancy,
    ) {}
}
# 15. Configuración por subsistema
La configuración deberá dividirse conceptualmente:
AuthorizationConfig
├── PolicyConfig
├── GateConfig
├── DecisionConfig
├── CacheConfig
├── AuditConfig
├── TelemetryConfig
├── TenantConfig
├── ScopeConfig
├── DelegationConfig
├── RiskConfig
├── ApprovalConfig
├── CompilationConfig
└── RuntimeConfig
# 16. Configuration Loader
Contrato:
interface AuthorizationConfigurationLoaderInterface
{
    public function load(): AuthorizationConfig;
}
# 17. Configuration Sources
Podrán existir:
framework defaults
application config
environment variables
package config
tenant policy config
runtime overrides
pero deberán poseer precedencia explícita.
# 18. Precedencia
Modelo recomendado:
Framework Defaults
        ↓
Package Configuration
        ↓
Application Configuration
        ↓
Environment Overrides
        ↓
Tenant Configuration
        ↓
Explicit Runtime Overrides
# 19. Security Rule
Una capa inferior no deberá poder deshabilitar silenciosamente una restricción marcada como:
non-overridable
# 20. Security Floor
VoltStack deberá introducir el concepto:
Authorization Security Floor
que representa requisitos mínimos que configuraciones posteriores no pueden debilitar.
# 21. Ejemplo
Platform:
cross_tenant_access = forbidden
Tenant intenta:
cross_tenant_access = allow
Si la regla es non-overridable:
configuration rejected
# 22. Configuration Validator
interface AuthorizationConfigurationValidatorInterface
{
    public function validate(
        AuthorizationConfig $config
    ): ConfigurationValidationResult;
}
# 23. Validaciones
Deberá detectar:
unknown strategies
unknown providers
duplicate policy mappings
duplicate abilities
invalid voter priorities
invalid cache configuration
invalid tenant modes
missing dependencies
invalid approval strategies
contradictory SoD rules
unsafe production options
# 24. Fail Fast
Errores estructurales deberán producir:
BOOT FAILURE
no esperar hasta la primera petición.
# 25. Ejemplo
Si se configura:
decision_strategy = "quantum_magic"
y no existe ese strategy:
AuthorizationConfigurationException
durante bootstrap.
# 26. Configuration Diagnostics
Los errores deberán incluir:
configuration path
invalid value
expected values
origin
suggested correction
# 27. Ejemplo
Authorization configuration error:

Path:
authorization.decision.strategy

Value:
"consensus_magic"

Expected:
affirmative | unanimous | consensus | priority

Source:
config/authorization.php
# 28. Service Provider
VoltStack podrá utilizar:
```php
final class AuthorizationServiceProvider
{
}
```
como punto principal de integración.
# 29. Responsabilidad del Provider
El Provider deberá:
register configuration
register contracts
register core services
register registries
register compilers
register runtime services
register resetters
register extension points
# 30. No debe
El Service Provider no deberá:
execute authorization decisions
load current user
resolve current tenant permanently
store request state
execute Policies
# 31. Fases del Provider
Modelo:
```text
register()
    ↓
boot()
```
pero internamente VoltStack podrá poseer fases más precisas.
# 32. Fases internas
CONFIGURE
REGISTER
DISCOVER
COMPILE
BOOT
READY
# 33. Bootstrap Phase Enum
```php
enum AuthorizationBootstrapPhase: string
{
    case Configure = 'configure';
    case Register = 'register';
    case Discover = 'discover';
    case Compile = 'compile';
    case Boot = 'boot';
    case Ready = 'ready';
}
```
# 34. Configure Phase
Responsabilidades:
load config
merge defaults
resolve environment
validate schema
construct AuthorizationConfig
# 35. Register Phase
Responsabilidades:
bind contracts
register factories
register registries
register compilers
register strategies
register providers
# 36. Discover Phase
Responsabilidades:
discover Policies
discover Voters
discover Attributes
discover Gate definitions
discover evaluators
discover plugins
# 37. Compile Phase
Responsabilidades:
compile metadata
compile policy maps
compile ability registry
compile voter plans
compile controller authorization metadata
compile approval policies
compile extension registry
# 38. Boot Phase
Responsabilidades:
load compiled manifests
freeze static registries
connect runtime services
register request lifecycle hooks
register resetters
# 39. Ready Phase
A partir de aquí:
AuthorizationManager
puede procesar decisiones.
# 40. Bootstrap Coordinator
interface AuthorizationBootstrapperInterface
{
    public function bootstrap(): void;
}
# 41. Implementation
final class AuthorizationBootstrapper
    implements AuthorizationBootstrapperInterface
{
    public function __construct(
        private AuthorizationConfigurationLoaderInterface $loader,
        private AuthorizationConfigurationValidatorInterface $validator,
        private AuthorizationRegistryCompilerInterface $compiler,
    ) {}

    public function bootstrap(): void
    {
        // orchestrate phases
    }
}
# 42. Service Container
Authorization deberá integrarse mediante contratos.
Ejemplo:
```php
$container->bind(
    AuthorizationManagerInterface::class,
    AuthorizationManager::class
);
```
# 43. Core Contracts
Como mínimo:
AuthorizationManagerInterface
DecisionManagerInterface
PolicyRegistryInterface
PolicyResolverInterface
PolicyDispatcherInterface
GateInterface
AbilityRegistryInterface
AuthorizationPlannerInterface
AuthorizationContextResolverInterface
# 44. Advanced Contracts
También:
AuthorizationCacheInterface
AuthorizationAuditInterface
AuthorizationTracerInterface
AuthorizationRiskEvaluatorInterface
AuthorizationScopeResolverInterface
DelegationResolverInterface
RelationshipResolverInterface
ApprovalManagerInterface
SeparationOfDutiesManagerInterface
# 45. Depend on Interfaces
Los consumidores deberán solicitar:
AuthorizationManagerInterface
no:
AuthorizationManager
cuando no necesiten detalles concretos.
# 46. Service Lifetimes
VoltStack deberá distinguir:
Singleton
Application Scoped
Request Scoped
Transient
Lazy
# 47. Singleton
Servicios inmutables o stateless podrán persistir durante toda la vida del worker.
Ejemplos:
CompiledPolicyRegistry
CompiledAbilityRegistry
DecisionStrategyRegistry
MetadataRegistry
AuthorizationConfig
# 48. Application Scoped
Componentes vinculados al runtime de la aplicación:
AuthorizationCompiler
AuthorizationManifestLoader
PolicyDiscovery
podrán vivir durante el lifecycle de aplicación.
# 49. Request Scoped
Información mutable asociada a una petición:
AuthorizationContext
CurrentPrincipal
CurrentActor
CurrentTenant
CurrentScope
RequestDecisionMemoization
RiskContext
DelegationContext
deberá ser request-scoped.
# 50. Transient
Evaluadores con estado temporal no compartible podrán construirse por uso.
# 51. Lazy Services
Subsistemas costosos deberán poder inicializarse bajo demanda.
Ejemplo:
Approval subsystem
Relationship graph provider
External PDP client
advanced audit exporter
# 52. FrankenPHP
VoltStack está diseñado para aprovechar runtimes persistentes como FrankenPHP.
Esto cambia radicalmente las reglas de lifecycle.
# 53. Modelo PHP clásico
Tradicionalmente:
Request
    ↓
Bootstrap
    ↓
Execute
    ↓
Process dies
Gran parte del estado desaparece automáticamente.
# 54. Worker persistente
Con FrankenPHP:
Worker Start
    ↓
Bootstrap Once
    ↓
Request A
    ↓
Reset
    ↓
Request B
    ↓
Reset
    ↓
Request C
# 55. Consecuencia
Authorization no podrá depender del final del proceso PHP para limpiar estado.
# 56. Golden Rule
Application state may persist.
Request authorization state must not.
# 57. Safe Persistent State
Puede persistir:
compiled policies
immutable configuration
static ability definitions
compiled metadata
strategy registries
immutable service graph
# 58. Unsafe Persistent State
No deberá persistir entre requests:
current principal
current actor
current tenant
current resource
current scope
decision memoization
current delegation
risk score
approval candidate
temporary authorization stack
# 59. Request Authorization Scope
VoltStack deberá crear:
AuthorizationRequestScope
por request.
# 60. Contract
interface AuthorizationRequestScopeInterface
{
    public function context(): AuthorizationContext;

    public function reset(): void;
}
# 61. Request Lifecycle
HTTP Request Begins
        ↓
Create Request Scope
        ↓
Resolve Principal
        ↓
Resolve Actor
        ↓
Resolve Tenant
        ↓
Initialize AuthorizationContext
        ↓
Process Application
        ↓
Authorization Decisions
        ↓
Request Terminates
        ↓
Reset Authorization Scope
# 62. Resettable Services
Contrato general:
```php
interface AuthorizationResettableInterface
{
    public function resetAuthorizationState(): void;
}
```
# 63. AuthorizationRuntimeResetter
```php
final class AuthorizationRuntimeResetter
{
    /** @var iterable<AuthorizationResettableInterface> */
    public function __construct(
        private iterable $services
    ) {}

    public function reset(): void
    {
        foreach ($this->services as $service) {
            $service->resetAuthorizationState();
        }
    }
}
```
# 64. Reset Order
El orden deberá ser determinista.
Ejemplo:
```text
Decision Memoization
        ↓
Risk Context
        ↓
Delegation Context
        ↓
Scope Context
        ↓
Tenant Context
        ↓
Principal / Actor
```
# 65. Reset Even on Exception
Debe ejecutarse:
finally
conceptualmente.
# 66. Ejemplo
```php
try {
    $kernel->handle($request);
} finally {
    $authorizationResetter->reset();
}
```
# 67. Critical Invariant
Una excepción en:
Controller
Policy
Voter
Database
Renderer
no puede impedir la limpieza del Authorization Context.
# 68. Request Scope ID
Para debugging podrá existir:
authorization_scope_id
único por request.
# 69. Leak Detection
En desarrollo, VoltStack podrá detectar:
scope from Request A
used during Request B
y lanzar:
AuthorizationContextLeakException
# 70. Scope Generation
Cada request podrá poseer:
generation
Ejemplo:
Worker generation: 519
Los objetos request-scoped pueden registrar esa generación.
# 71. Development Assertion
Si:
context.generation != current.generation
resultado:
STALE AUTHORIZATION CONTEXT
# 72. Context Resolver
interface AuthorizationContextResolverInterface
{
    public function resolve(): AuthorizationContext;
}
# 73. Resolver Composition
Podrá componerse:
PrincipalResolver
ActorResolver
TenantResolver
ScopeResolver
EnvironmentResolver
RequestMetadataResolver
RiskContextResolver
# 74. Context Construction
Identity Context
       +
Tenant Context
       +
Scope Context
       +
Request Context
       +
Environment Context
       ↓
AuthorizationContext
# 75. Immutable Context
El resultado deberá preferentemente ser:
final readonly class AuthorizationContext
{
}
# 76. Context Evolution
Si durante el request ocurre:
step-up authentication
no mutar arbitrariamente el objeto anterior.
Crear:
AuthorizationContext v2
# 77. Context Version
Podrá incluir:
context_version
para invalidar memoización previa.
# 78. Request Memoization
El cache local de decisiones deberá estar asociado a:
RequestScope
+
ContextVersion
# 79. Step-Up
Si cambia:
Assurance Level
de:
AAL1
→ AAL2
decisiones memoizadas bajo AAL1 no deberán reutilizarse incorrectamente.
# 80. Tenant Switch
Igualmente:
Tenant A
→ Tenant B
deberá invalidar completamente:
authorization request memoization
# 81. Policy Registry
Durante desarrollo podrá existir:
MutablePolicyRegistry
pero producción deberá preferir:
CompiledPolicyRegistry
# 82. Registry Lifecycle
Registration
    ↓
Compilation
    ↓
Freeze
    ↓
Runtime Read Only
# 83. Frozen Registry
Una vez READY:
register new policy
deberá fallar salvo que runtime dinámico esté explícitamente habilitado.
# 84. Reason
Esto garantiza:
determinism
performance
cache safety
diagnostics
# 85. Ability Registry
Mismo principio:
MutableAbilityRegistry
        ↓
CompiledAbilityRegistry
        ↓
Frozen
# 86. Voter Registry
Voter Definitions
        ↓
Priority Resolution
        ↓
Compiled Voter Plan
# 87. Strategy Registry
Deberá permitir:
affirmative
unanimous
consensus
priority
custom
# 88. Registration API
```php
$strategies->register(
    'custom',
    CustomDecisionStrategy::class
);
```
# 89. Duplicate Strategy
Dos providers intentando registrar:
custom
deberán producir conflicto salvo override explícito.
# 90. Override Semantics
VoltStack deberá distinguir:
register
replace
decorate
# 91. Register
Falla si ya existe.
# 92. Replace
Sustituye explícitamente una implementación.
# 93. Decorate
Envuelve la implementación existente.
# 94. Example Decoration
```text
AuthorizationManager
        ↓
TracingAuthorizationManager
        ↓
AuditAuthorizationManager
```
aunque el orden real deberá ser definido por arquitectura.
# 95. Container Decoration
API conceptual:
```php
$container->decorate(
    AuthorizationManagerInterface::class,
    TracingAuthorizationManager::class
);
```
# 96. Avoid Hidden Replacement
Un paquete no deberá reemplazar silenciosamente:
AuthorizationManagerInterface
sin declarar intención.
# 97. Protected Services
VoltStack podrá marcar contratos críticos:
protected
# 98. Ejemplos
TenantBoundaryEvaluator
ApprovalProofVerifier
AuthorizationContextFactory
# 99. Replacement Policy
Para reemplazar un servicio protegido deberá requerirse:
explicit application authorization configuration
# 100. Service Tags
El Container deberá soportar agrupación lógica.
Ejemplos:
authorization.policy
authorization.voter
authorization.evaluator
authorization.context_resolver
authorization.resettable
authorization.compiler_pass
authorization.plugin
authorization.approval_strategy
# 101. Tagged Iterators
Ejemplo:
```php
public function __construct(
    #[Tagged('authorization.voter')]
    iterable $voters
) {}
```
# 102. Priorities
Los tagged services deberán poder declarar:
priority
# 103. Priority Determinism
Empates deberán resolverse determinísticamente.
Ejemplo:
priority
registration order
stable identifier
# 104. Prefer Explicit IDs
Producción deberá preferir IDs estables sobre orden accidental.
# 105. Policy Discovery
VoltStack deberá soportar:
explicit registration
attribute discovery
convention discovery
package registration
compiled manifest
# 106. Explicit Registration
```php
$policies->map(
    Invoice::class,
    InvoicePolicy::class
);
```
# 107. Convention Discovery
Ejemplo:
App\Models\Invoice
        ↓
App\Policies\InvoicePolicy
# 108. Convention is Convenience
No deberá ser el único mecanismo.
# 109. Attribute Discovery
Ejemplo conceptual:
```php
#[PolicyFor(Invoice::class)]
final class InvoicePolicy
{
}
```
# 110. Discovery Paths
Configuración:
```php
'discovery' => [
    'paths' => [
        app_path('Policies'),
    ],
],
```
# 111. Production Rule
No escanear filesystem completo en cada request.
# 112. Discovery Lifecycle
```text
Development
    ↓
Scan
    ↓
Discover
    ↓
Compile Manifest

Production
    ↓
Load Manifest
```
# 113. Policy Manifest
Ejemplo conceptual:
```php
return [
    Invoice::class => InvoicePolicy::class,
    Payment::class => PaymentPolicy::class,
];
```
# 114. Manifest Metadata
También podrá contener:
policy class
subject class
abilities
method map
attributes
priority
source file
fingerprint
# 115. Source Fingerprint
Permite detectar:
stale compiled authorization metadata
en desarrollo.
# 116. Authorization Compiler
El compiler deberá transformar configuración y metadata declarativa en estructuras optimizadas.
# 117. Compilation Pipeline
Configuration
      +
Policies
      +
Gates
      +
Voters
      +
Controller Attributes
      +
Scope Metadata
      +
Approval Metadata
      ↓
Normalization
      ↓
Validation
      ↓
Dependency Resolution
      ↓
Conflict Detection
      ↓
Optimization
      ↓
Compiled Authorization Manifest
# 118. Compilation Outputs
Podrán generarse:
policy-map.php
ability-map.php
voter-plan.php
controller-authorization.php
approval-plan.php
authorization-services.php
authorization-manifest.php
# 119. Cache Directory
Conceptualmente:
storage/framework/cache/authorization/
o equivalente definido por VoltStack.
# 120. No Sensitive Runtime Data
Nunca escribir ahí:
current users
current permissions
access tokens
approval evidence
tenant secrets
# 121. Compiled Manifest
Debe contener únicamente información estructural apropiada.
# 122. Manifest Version
format_version
framework_version
authorization_version
application_fingerprint
generated_at
# 123. Compatibility
VoltStack deberá rechazar manifests incompatibles.
# 124. Example
Manifest format: 3
Runtime supports: 4

→ recompilation required
# 125. Atomic Compilation
No escribir directamente sobre manifest activo.
Preferir:
compile temp
    ↓
validate
    ↓
atomic replace
# 126. Partial Compilation Failure
El runtime deberá conservar el último manifest válido cuando sea seguro.
# 127. Production Deployment
Modelo recomendado:
composer install
      ↓
VoltStack compile
      ↓
Authorization compile
      ↓
Validate
      ↓
Deploy
      ↓
Start workers
# 128. Worker Startup
El worker deberá cargar:
compiled authorization manifest
una vez.
# 129. Hot Reload
Desarrollo podrá soportar:
authorization metadata hot reload
# 130. Production
Hot reload deberá estar:
disabled by default
para preservar determinismo.
# 131. Reload Strategy
Si cambia Policy:
invalidate manifest
    ↓
recompile
    ↓
new application generation
# 132. Do Not Mutate Live Graph
Evitar:
replace half of registries
while requests are running
# 133. Generation Model
Puede existir:
AuthorizationRuntimeGeneration
# 134. Generation Swap
Generation 41
        ↓
Compile 42
        ↓
Validate 42
        ↓
Atomic activate 42
Requests existentes terminan con:
41
nuevos utilizan:
42
# 135. Container Compilation
El Service Container podrá precomputar:
constructor dependencies
tagged iterators
service decorators
aliases
lazy factories
# 136. Authorization Service Graph
Ejemplo:
AuthorizationManagerInterface
        ↓
AuthorizationManager
        ├── AuthorizationPlannerInterface
        ├── DecisionManagerInterface
        ├── AuthorizationContextResolverInterface
        ├── PolicyDispatcherInterface
        └── AuthorizationAuditInterface
# 137. Decision Manager Graph
DecisionManager
    ├── VoterRegistry
    ├── DecisionStrategy
    ├── DecisionNormalizer
    └── DecisionExplanationBuilder
# 138. Policy Dispatcher Graph
PolicyDispatcher
    ├── PolicyResolver
    ├── ParameterResolver
    ├── InvocationEngine
    └── ResultNormalizer
# 139. Circular Dependency Detection
El Container deberá detectar ciclos.
Ejemplo inválido:
AuthorizationManager
    ↓
AuditService
    ↓
AuthorizationManager
# 140. Solución
Usar:
events
lazy references
narrow contracts
pero no esconder ciclos arbitrariamente.
# 141. Boot-Time Dependency Graph Validation
Antes de READY:
validate authorization service graph
# 142. Missing Dependency
Ejemplo:
RiskBasedPolicy enabled
pero no existe:
RiskEvaluatorInterface
deberá producir error.
# 143. Optional Dependencies
Subsistemas deshabilitados no deberán exigir servicios innecesarios.
# 144. Example
Si:
approval.enabled = false
no exigir:
ApprovalRepository
# 145. Conditional Registration
Feature enabled?
    ↓
YES → register subsystem
NO  → omit subsystem
# 146. Performance
Esto reduce:
container size
startup work
memory
runtime branching
# 147. Null Implementations
Podrán utilizarse selectivamente:
NullAuthorizationTracer
pero no para ocultar errores críticos.
# 148. Good Null Service
NullAuthorizationMetrics
cuando telemetry esté deshabilitada.
# 149. Bad Null Service
NullTenantBoundary
→ always allow
Esto sería peligroso.
# 150. Secure Defaults
Cuando falta una dependencia de seguridad:
fail closed
o:
fail bootstrap
# 151. Configuration Profiles
VoltStack podrá proporcionar:
development
testing
production
# 152. Development Profile
Puede activar:
policy discovery
verbose explanations
context leak detection
manifest freshness validation
debug metadata
# 153. Testing Profile
Puede activar:
deterministic clock
fake audit
isolated registries
test policy providers
authorization assertions
# 154. Production Profile
Deberá preferir:
compiled metadata
frozen registries
minimal reflection
secure errors
optimized tracing
strict configuration validation
# 155. Environment Variables
Solo para configuración operacional apropiada.
Ejemplos:
AUTHORIZATION_CACHE=true
AUTHORIZATION_AUDIT=true
AUTHORIZATION_DEBUG=false
# 156. Avoid
No definir cientos de Policies mediante environment variables.
# 157. Tenant Configuration
Multi-tenancy puede requerir configuración específica.
# 158. Tenant Authorization Profile
final readonly class TenantAuthorizationProfile
{
    public function __construct(
        public TenantId $tenant,
        public array $policyOverrides,
        public array $riskRules,
        public array $approvalRules,
    ) {}
}
# 159. Tenant Config Is Runtime Data
A diferencia de configuración global compilada, algunos perfiles tenant pueden cambiar dinámicamente.
# 160. Therefore
Separar:
Static Authorization Configuration
de:
Dynamic Authorization Policy Data
# 161. Static Configuration
Ejemplos:
enabled evaluators
available strategies
provider implementations
compiler settings
# 162. Dynamic Policy Data
Ejemplos:
tenant approval threshold
tenant custom role mapping
tenant risk threshold
tenant resource restrictions
# 163. Dynamic Config Provider
interface TenantAuthorizationProfileProviderInterface
{
    public function forTenant(
        TenantId $tenant
    ): TenantAuthorizationProfile;
}
# 164. Cache
Podrá cachearse con:
tenant policy version
# 165. Versioned Tenant Policy
Tenant#50
Authorization Policy Version = 19
# 166. Cache Key
tenant:50:authorization:v19
# 167. Update
Cuando cambia configuración:
v19
→
v20
las entradas anteriores dejan de ser utilizadas.
# 168. Plugin Bootstrap
Plugins podrán participar en Authorization.
# 169. AuthorizationPluginInterface
interface AuthorizationPluginInterface
{
    public function register(
        AuthorizationPluginRegistry $registry
    ): void;
}
# 170. Plugin Capabilities
Podrá registrar:
voters
policy providers
evaluators
strategies
candidate selectors
context contributors
audit processors
compiler passes
# 171. Plugin Ordering
Debe declararse explícitamente cuando sea relevante.
# 172. Example
```php
final class GeoRiskAuthorizationPlugin
    implements AuthorizationPluginInterface
{
    public function register(
        AuthorizationPluginRegistry $registry
    ): void {
        $registry->evaluator(
            GeoRiskEvaluator::class
        );
    }
}
```
# 173. Plugin Isolation
Un plugin no deberá recibir acceso irrestricto a:
mutable Authorization internals
# 174. Capability-Based Registration
Preferir:
registry->addVoter(...)
sobre:
registry->getContainer()->replaceAnything(...)
# 175. Compiler Passes
Plugins avanzados podrán registrar:
AuthorizationCompilerPass
# 176. CompilerPass Contract
```php
interface AuthorizationCompilerPassInterface
{
    public function process(
        AuthorizationCompilationContext $context
    ): void;
}
```
# 177. Compiler Pass Phases
Podrán existir:
before_normalization
after_discovery
after_resolution
before_optimization
after_validation
# 178. Security
Compiler passes no deberán poder eliminar reglas:
non-bypassable
sin error.
# 179. Core Invariants
Después de todos los compiler passes deberá ejecutarse:
FinalAuthorizationInvariantValidator
# 180. Invariants
Deberá verificar:
tenant boundary exists when required
policy references valid
ability references valid
no unresolved strategies
no illegal protected-service replacement
no contradictory security floors
# 181. Facade
VoltStack podrá proporcionar:
```php
Authorization::check(...);
Authorization::authorize(...);
```
# 182. Facade Rule
La Facade deberá resolver:
AuthorizationManagerInterface
desde el contexto apropiado.
# 183. No Static Authorization State
Aunque exista Facade:
Authorization::...
no significa que el estado se almacene estáticamente.
# 184. Facade
Debe ser:
```text
static access syntax
        ↓
container-resolved request-scoped service
```
# 185. Helper
Opcionalmente:
```php
authorize('invoice.update', $invoice);
```
# 186. Helper Rule
Debe delegar al mismo Manager.
Nunca implementar lógica paralela.
# 187. One Authorization Engine
Todas las APIs:
Facade
Helper
Controller Attribute
Middleware
Gate
Domain Service
deben converger hacia:
Authorization Core
# 188. No Parallel Security Engines
Prohibido conceptualmente:
Controller Authorization Engine
Gate Authorization Engine
Policy Authorization Engine
independientes.
# 189. Correct Model
Controller
Middleware
Gate
Facade
Helper
Service
   │
   └──────────────┐
                  ↓
         AuthorizationManager
                  ↓
         AuthorizationPlanner
                  ↓
           Decision Engine
# 190. Middleware Integration
Middleware deberá resolver servicios del request scope.
# 191. Example
AuthorizationMiddleware
        ↓
AuthorizationManagerInterface
        ↓
Current AuthorizationContext
# 192. Route Compilation
Metadata:
```php
#[Authorize('invoice.update')]
```
deberá integrarse durante:
Route / Controller compilation
# 193. Avoid Runtime Reflection
Producción:
Route
    ↓
Compiled Authorization Metadata
no:
Route
    ↓
ReflectionClass
    ↓
ReflectionMethod
    ↓
scan attributes every request
# 194. Controller Resolver
El Controller subsystem deberá recibir:
CompiledAuthorizationMetadata
desde el manifest.
# 195. Integration Boundary
Authorization no debe construir Controllers.
Controller System no debe implementar Authorization.
# 196. Contract
Ambos se conectan mediante:
Authorization Metadata
+
AuthorizationManagerInterface
# 197. Route Authorization
Igualmente Routing no decide Policies.
Routing identifica:
authorization requirements
Authorization decide.
# 198. Authentication Integration
Authentication deberá publicar al request scope:
AuthenticatedIdentity
AuthenticationAssurance
AuthenticationMethod
Actor
# 199. Authorization Adapter
Authorization podrá convertir:
AuthenticatedIdentity
en:
AuthorizationSubject
# 200. Anonymous Principal
Si no existe autenticación:
AnonymousPrincipal
podrá utilizarse explícitamente.
# 201. Avoid Null Identity
Preferir:
AnonymousPrincipal
sobre:
principal = null
en el núcleo.
# 202. System Principal
Para tareas internas:
SystemPrincipal
# 203. Service Principal
Para servicios:
ServicePrincipal
# 204. Identity Factory
interface AuthorizationPrincipalFactoryInterface
{
    public function fromIdentity(
        IdentityInterface $identity
    ): Principal;
}
# 205. Database Integration
Authorization Core no deberá depender directamente de un ORM específico.
# 206. Repository Contracts
Ejemplos:
RoleRepositoryInterface
PermissionRepositoryInterface
RelationshipProviderInterface
DelegationRepositoryInterface
ApprovalRepositoryInterface
# 207. Infrastructure Bindings
Database module podrá proporcionar:
SqlRoleRepository
SqlPermissionRepository
SqlRelationshipProvider
# 208. Decoupling
Esto permite:
SQL
Redis
Remote PDP
LDAP-derived roles
External IAM
custom storage
# 209. Cache Integration
Authorization deberá consumir:
AuthorizationCacheInterface
no un driver concreto.
# 210. Cache Provider
Container selecciona:
Array
Redis
Memory
Distributed
Null
según configuración.
# 211. Cache Namespacing
Debe incluir:
application
authorization version
tenant
policy version
cuando corresponda.
# 212. Event Integration
Authorization deberá registrar sus eventos en el Event System durante bootstrap.
# 213. Events
Ejemplos:
AuthorizationEvaluating
AuthorizationGranted
AuthorizationDenied
AuthorizationChallenged
AuthorizationFailed
# 214. Event Listener Registration
Debe ocurrir en bootstrap.
No en cada decisión.
# 215. Telemetry Integration
Telemetry decorators/exporters también se construyen durante bootstrap.
# 216. No Telemetry Logic in Policies
Policy:
public function update(...)
no debería llamar manualmente:
metrics
tracer
logger
para observabilidad estándar.
# 217. Instrumentation Layer
Debe envolver el pipeline automáticamente.
# 218. Testing Integration
El Container deberá permitir reemplazos controlados.
# 219. Example
```php
$this->swap(
    AuthorizationRiskEvaluatorInterface::class,
    FakeRiskEvaluator::class
);
```
# 220. Test Isolation
Cada test deberá recibir:
fresh request authorization scope
# 221. Test Registry
En tests podrá existir:
MutableTestPolicyRegistry
sin afectar producción.
# 222. Fake Authorization
VoltStack podrá proporcionar:
FakeAuthorizationManager
pero su uso deberá ser explícito.
# 223. Danger
Un Fake global tipo:
allow everything
puede ocultar fallos de seguridad.
# 224. Better Testing API
Preferir:
```php
Authorization::fake()
    ->allow('invoice.view')
    ->deny('invoice.delete');
```
con assertions.
# 225. Assertions
Ejemplos:
```text
assertAuthorizationChecked
assertAuthorizationGranted
assertAuthorizationDenied
assertPolicyInvoked
assertVoterInvoked
```
# 226. Bootstrap Tests
También deberán existir pruebas para:
container bindings
compiler manifests
service lifetimes
registry freezing
reset behavior
plugin conflicts
# 227. Persistent Worker Test
Crítico:
Request A
Principal A
Tenant A
Decision ALLOW
        ↓
RESET
        ↓
Request B
Principal B
Tenant B
Request B jamás deberá observar información de A.
# 228. Container Scope Test
Debe comprobar:
singleton registry
same instance across requests
pero:
request context
different instance per request
# 229. Startup Performance
Authorization bootstrap deberá optimizar:
filesystem scanning
reflection
container resolution
manifest parsing
policy discovery
# 230. Development vs Production
Development:
flexibility > startup performance
Production:
compiled deterministic startup
# 231. Runtime Performance
Una vez READY, una autorización sencilla idealmente recorrerá:
compiled metadata
        ↓
request context
        ↓
planner
        ↓
relevant evaluators only
        ↓
decision
# 232. No Bootstrap Work in Hot Path
No realizar:
scan policy directory
parse config files
discover plugins
compile attributes
durante cada autorización.
# 233. Lazy Advanced Subsystems
Si Ability no necesita:
ReBAC
Risk
Approval
Delegation
no resolver esos servicios.
# 234. Planner-Driven Dependency Resolution
El plan compilado puede indicar:
needs_rbac = true
needs_abac = false
needs_rebac = false
needs_risk = false
needs_approval = false
# 235. Optimized Runtime Graph
Entonces:
Request
  ↓
RBAC
  ↓
Policy
  ↓
Decision
sin cargar los demás subsistemas.
# 236. Diagnostics Command
VoltStack deberá proporcionar tooling equivalente conceptualmente a:
authorization:debug
# 237. Debug Output
Podrá mostrar:
Authorization enabled: yes
Compiled manifest: yes
Policies: 83
Abilities: 241
Voters: 12
Decision strategy: affirmative
Tenant isolation: enabled
Risk engine: enabled
Approval engine: enabled
Runtime generation: 52
# 238. Container Debug
Comando conceptual:
authorization:container
# 239. Output
AuthorizationManagerInterface
→ AuthorizationManager
lifetime=request-aware

PolicyRegistryInterface
→ CompiledPolicyRegistry
lifetime=singleton

AuthorizationContextInterface
→ RequestAuthorizationContext
lifetime=request
# 240. Policy Debug
authorization:policy Invoice
podrá mostrar:
Policy:
App\Policies\InvoicePolicy

Source:
explicit mapping

Abilities:
view
create
update
delete

Compiled:
yes
# 241. Ability Debug
authorization:ability invoice.update
podrá mostrar el pipeline estructural sin ejecutar una autorización real.
# 242. Production Security
Los comandos de debug no deberán exponer secretos.
# 243. Runtime Debug Endpoint
No habilitar automáticamente endpoints HTTP como:
/__authorization/debug
en producción.
# 244. Health Check
Podrá existir:
AuthorizationHealthCheck
# 245. Health Check Validation
Podrá verificar:
manifest readable
required providers available
policy registry valid
tenant provider reachable
cache optional
# 246. Health Check Must Not Authorize
No deberá ejecutar acciones reales.
# 247. Startup Failure Classes
Jerarquía propuesta:
AuthorizationBootstrapException
├── AuthorizationConfigurationException
├── AuthorizationContainerException
├── AuthorizationCompilationException
├── AuthorizationManifestException
├── AuthorizationPluginException
└── AuthorizationInvariantException
# 248. Runtime vs Bootstrap Exceptions
Separar:
bootstrap failure
de:
authorization denial
# 249. Important
Una Policy mal registrada:
500 / bootstrap error
no:
403
# 250. Denial Is Not Misconfiguration
DENY
es una decisión válida.
# 251. Missing Policy
Dependiendo de configuración:
fail closed
pero además generar diagnóstico de misconfiguración cuando se esperaba una Policy.
# 252. Strict Mode
Podrá existir:
'strict' => true,
# 253. Strict Mode
Detectará:
unknown ability
missing policy
unregistered evaluator
ambiguous policy
invalid metadata
# 254. Production Recommendation
VoltStack deberá favorecer:
strict = true
para aplicaciones compiladas.
# 255. Dynamic Ability Mode
Aplicaciones extremadamente dinámicas podrán permitir:
strict abilities = false
pero explícitamente.
# 256. Package Integration
Un Quantum package podrá publicar:
Policies
Abilities
Voters
Approval strategies
Authorization metadata
# 257. Package Manifest
Ejemplo:
```php
final class BillingAuthorizationProvider
{
    public function register(
        AuthorizationRegistry $authorization
    ): void {
        $authorization->policy(
            Invoice::class,
            InvoicePolicy::class
        );

        $authorization->ability(
            'billing.invoice.refund'
        );
    }
}
```
# 258. Package Namespace
Abilities deberán evitar colisiones.
Preferir:
billing.invoice.refund
sobre:
refund
# 259. Package Conflict Detection
Dos packages registrando la misma Ability con definiciones incompatibles deberán fallar en compilación.
# 260. Application Overrides
La aplicación podrá reemplazar Policies de package explícitamente.
# 261. Example
Package:
Invoice → DefaultInvoicePolicy

Application:
Invoice → EnterpriseInvoicePolicy
# 262. Override Must Be Visible
El compiled manifest deberá registrar:
original provider
replacement provider
reason/source
# 263. Configuration Immutability
Después de READY:
AuthorizationConfig
deberá considerarse inmutable.
# 264. Runtime Changes
Cambios dinámicos deberán realizarse mediante:
versioned policy stores
tenant profiles
runtime providers
no mutando config global.
# 265. Bootstrap Idempotency
Ejecutar accidentalmente:
```text
bootstrap()
bootstrap()
```
no deberá duplicar:
voters
listeners
policies
plugins
# 266. Bootstrap State
```php
enum AuthorizationBootstrapState
{
    case NotStarted;
    case Bootstrapping;
    case Ready;
    case Failed;
}
```
# 267. Recursive Bootstrap
Si durante bootstrap un servicio intenta usar Authorization:
AuthorizationManager->authorize(...)
deberá detectarse.
# 268. Error
AuthorizationBootstrapReentrancyException
# 269. Reason
Evitar ciclos como:
bootstrap
    ↓
register audit
    ↓
audit asks authorization
    ↓
bootstrap authorization
# 270. Boot Events
Podrán emitirse:
AuthorizationBootstrapping
AuthorizationConfigured
AuthorizationRegistered
AuthorizationCompiled
AuthorizationBooted
AuthorizationReady
AuthorizationBootstrapFailed
# 271. Event Restrictions
Listeners de boot no deberán ejecutar decisiones antes de READY salvo APIs específicamente diseñadas.
# 272. Readiness Barrier
Authorization Manager deberá comprobar:
bootstrapState === READY
antes de procesar decisiones.
# 273. CLI
CLI también puede utilizar Authorization.
# 274. CLI Context
No asumir:
HTTP request always exists
# 275. Context Sources
Authorization Context deberá poder originarse desde:
HTTP
CLI
Queue
Scheduler
WebSocket
RPC
Worker
# 276. Context Adapter
interface AuthorizationExecutionContextAdapterInterface
{
    public function supports(
        ExecutionContext $context
    ): bool;

    public function createAuthorizationContext(
        ExecutionContext $context
    ): AuthorizationContext;
}
# 277. HTTP Adapter
HttpAuthorizationContextAdapter
# 278. CLI Adapter
CliAuthorizationContextAdapter
# 279. Queue Adapter
QueueAuthorizationContextAdapter
# 280. Scheduler Adapter
SchedulerAuthorizationContextAdapter
# 281. Queue Security
Nunca asumir que porque un Job llegó a la Queue:
it is authorized
# 282. Job Authorization Context
Debe reconstruirse de forma segura desde referencias verificables.
# 283. No Serialize Entire Context
Evitar serializar:
AuthorizationContext object
completo dentro del Job.
# 284. Prefer
principal reference
tenant reference
delegation reference
authorization intent
security version
y reconstruir el contexto.
# 285. Long-Lived Jobs
Si permisos cambian entre:
job dispatched
y:
job executed
la autorización deberá poder revalidarse.
# 286. WebSocket
Conexiones persistentes requieren lifecycle especial.
# 287. Connection Authentication != Eternal Authorization
Aunque el WebSocket se autentique al conectar:
authorization
puede cambiar después.
# 288. Per-Message Authorization
Operaciones sensibles deberán reautorizarse por mensaje/acción.
# 289. WebSocket Scope
No reutilizar indefinidamente:
decision memoization
de la conexión.
# 290. Scheduler
Scheduled tasks deberán usar:
SystemPrincipal
o:
ServicePrincipal
explícito.
# 291. No Implicit Superuser
El Scheduler no debe convertirse automáticamente en:
allow all
# 292. Console Commands
Igualmente:
CLI execution
no implica bypass.
# 293. Trusted Internal Calls
Si VoltStack soporta trusted internal capabilities deberán modelarse mediante:
Capabilities
del documento 20.
# 294. Never
if (PHP_SAPI === 'cli') {
    return true;
}
# 295. Directory Structure
Propuesta:
Quantum/
└── Authorization/
    ├── Bootstrap/
    │   ├── AuthorizationBootstrapper.php
    │   ├── AuthorizationBootstrapState.php
    │   ├── AuthorizationBootstrapPhase.php
    │   ├── AuthorizationRuntimeResetter.php
    │   └── AuthorizationReadinessBarrier.php
    │
    ├── Config/
    │   ├── AuthorizationConfig.php
    │   ├── PolicyConfig.php
    │   ├── GateConfig.php
    │   ├── DecisionConfig.php
    │   ├── AuthorizationCacheConfig.php
    │   ├── AuthorizationRuntimeConfig.php
    │   ├── AuthorizationConfigurationLoader.php
    │   └── AuthorizationConfigurationValidator.php
    │
    ├── Container/
    │   ├── AuthorizationServiceProvider.php
    │   ├── AuthorizationServiceRegistrar.php
    │   ├── AuthorizationServiceGraphValidator.php
    │   └── AuthorizationServiceTags.php
    │
    ├── Compilation/
    │   ├── AuthorizationCompiler.php
    │   ├── AuthorizationCompilationContext.php
    │   ├── AuthorizationCompilerPassInterface.php
    │   ├── AuthorizationManifest.php
    │   ├── AuthorizationManifestLoader.php
    │   ├── AuthorizationManifestWriter.php
    │   └── AuthorizationInvariantValidator.php
    │
    ├── Discovery/
    │   ├── PolicyDiscovery.php
    │   ├── VoterDiscovery.php
    │   ├── AbilityDiscovery.php
    │   ├── AuthorizationMetadataDiscovery.php
    │   └── AuthorizationPluginDiscovery.php
    │
    ├── Runtime/
    │   ├── AuthorizationRuntime.php
    │   ├── AuthorizationRuntimeGeneration.php
    │   ├── AuthorizationRequestScope.php
    │   ├── AuthorizationContextFactory.php
    │   └── AuthorizationResettableInterface.php
    │
    ├── Integration/
    │   ├── HttpAuthorizationContextAdapter.php
    │   ├── CliAuthorizationContextAdapter.php
    │   ├── QueueAuthorizationContextAdapter.php
    │   ├── SchedulerAuthorizationContextAdapter.php
    │   └── WebSocketAuthorizationContextAdapter.php
    │
    └── Exceptions/
        ├── AuthorizationBootstrapException.php
        ├── AuthorizationConfigurationException.php
        ├── AuthorizationCompilationException.php
        ├── AuthorizationContainerException.php
        ├── AuthorizationManifestException.php
        ├── AuthorizationInvariantException.php
        └── AuthorizationContextLeakException.php
# 296. Bootstrap Architecture
                    VOLTSTACK APPLICATION
                            │
                            ↓
                   CONFIGURATION SYSTEM
                            │
                            ↓
                   AuthorizationConfig
                            │
                            ↓
                 Configuration Validator
                            │
                            ↓
                  SERVICE REGISTRATION
                            │
           ┌────────────────┼────────────────┐
           ↓                ↓                ↓
       Core Services     Registries      Extensions
           │                │                │
           └────────────────┼────────────────┘
                            ↓
                       DISCOVERY
                            │
              ┌─────────────┼─────────────┐
              ↓             ↓             ↓
           Policies       Voters       Metadata
              │             │             │
              └─────────────┼─────────────┘
                            ↓
                       COMPILATION
                            │
                            ↓
               Authorization Manifest
                            │
                            ↓
                     INVARIANT CHECK
                            │
                            ↓
                         FREEZE
                            │
                            ↓
                      RUNTIME READY
# 297. Runtime Architecture
                   PERSISTENT WORKER
                          │
                          ↓
                Compiled Authorization
                          │
                 ┌────────┴────────┐
                 │                 │
           Static Services     Static Registries
                 │                 │
                 └────────┬────────┘
                          │
             ───────── REQUEST A ─────────
                          │
                          ↓
                 Request Scope A
                          │
                 Principal / Actor
                          │
                     Tenant A
                          │
                     Context A
                          │
                     Decisions
                          │
                          ↓
                        RESET
                          │
             ───────── REQUEST B ─────────
                          │
                          ↓
                 Request Scope B
                          │
                 Principal / Actor
                          │
                     Tenant B
                          │
                     Context B
                          │
                     Decisions
                          │
                          ↓
                        RESET
# 298. Container Lifetime Matrix
Componente Lifetime recomendado
AuthorizationConfig Singleton
CompiledPolicyRegistry Singleton
CompiledAbilityRegistry Singleton
DecisionStrategyRegistry Singleton
AuthorizationManifest Singleton / Generation
AuthorizationManager Stateless singleton o request-aware proxy
AuthorizationContext Request
CurrentPrincipal Request
CurrentActor Request
CurrentTenant Request
CurrentScope Request
DecisionMemoization Request
RiskContext Request
DelegationContext Request
ApprovalContext Operation / Request
Compiler Application
Discovery services Application / Build
Repository adapters Según driver
Telemetry exporters Application
Audit context Request

  1. Important Manager Design
Idealmente:
AuthorizationManager
deberá ser:
stateless
y recibir el contexto mediante:
AuthorizationContextResolver
o explícitamente.
  2. Alternative
Si el Manager almacena estado request-scoped, entonces necesariamente deberá:
lifetime=request
  3. Recommendation
Preferir:
Stateless AuthorizationManager

+

Request-scoped AuthorizationContext
para reducir riesgos en FrankenPHP.
# 302. Explicit Context API
También:
```php
$authorization->check(
    request: new AuthorizationRequest(...),
    context: $context,
);
```
facilita:
testing
async execution
simulation
explainability
# 303. Implicit Context API
Convenience:
```php
$authorization->check(
    'invoice.update',
    $invoice
);
```
resuelve el contexto actual.
# 304. Both APIs
Deben converger al mismo método interno.
# 305. Security Invariants
El bootstrap deberá garantizar:
Invariant 1
No Authorization runtime before READY.
Invariant 2
Request state never becomes singleton state.
Invariant 3
Tenant state never survives request reset.
Invariant 4
Compiled registries are immutable at runtime.
Invariant 5
Missing security dependencies fail closed.
Invariant 6
Plugins cannot silently weaken protected invariants.
Invariant 7
Production does not require runtime filesystem discovery.
Invariant 8
Authorization configuration is validated before use.
Invariant 9
All public Authorization APIs converge on one core engine.
Invariant 10
CLI, Queue and Scheduler do not imply superuser authority.
# 306. Testing Invariants
Property:
Request A Context
∩
Request B Context
=

∅
except immutable application-level objects.
# 307. Container Property
```text
resolve(CompiledPolicyRegistry)
multiple times:
same immutable registry
```
# 308. Request Property
```text
resolve(AuthorizationContext)
across requests:
different context
```
# 309. Reset Property
Después de:
```text
reset()
```
debe cumplirse:
```text
currentPrincipal = none
currentActor = none
currentTenant = none
currentScope = none
memoization = empty
riskContext = empty
```
delegationContext = empty
# 310. Exception Property
Aunque:
Policy throws exception
el reset deberá ejecutarse.
# 311. Compilation Property
Misma configuración + mismas fuentes deberán producir semánticamente:
same authorization manifest
# 312. Determinism
El manifest no deberá depender accidentalmente del orden del filesystem.
# 313. Plugin Property
Plugins con igual priority deberán resolverse de forma determinista.
# 314. Security Floor Property
Ninguna configuración tenant podrá reducir una regla:
non-overridable
# 315. Manifest Integrity
Podrá calcularse:
manifest hash
para detectar corrupción accidental.
# 316. Optional Signature
Entornos avanzados podrán firmar manifests.
# 317. Deployment Integrity
Worker podrá verificar:
expected manifest hash
==

loaded manifest hash
# 318. No Secret in Manifest
Una firma no convierte el manifest en lugar seguro para secretos.
# 319. Configuration Secrets
Authorization normalmente no debería necesitar muchos secretos.
Cuando los requiera un provider externo:
secret reference
deberá resolverse mediante Secret Management.
# 320. Example
external_pdp.credentials
no deberá compilarse como texto plano dentro del manifest.
# 321. External PDP
Si Authorization utiliza un Policy Decision Point remoto:
OPA
custom IAM
external policy service
el Container deberá registrarlo como provider.
# 322. Provider Failure Policy
Configurable:
fail_closed
local_fallback
cached_safe_decision
según subsistema.
# 323. Default
Para autoridad crítica:
fail_closed
# 324. Startup vs Runtime Dependency
No exigir conexión al PDP durante compilación si no es necesario.
Pero sí validar:
provider configuration
# 325. Warmup
VoltStack podrá ejecutar:
AuthorizationWarmup
al iniciar worker.
# 326. Warmup
Puede cargar:
compiled manifest
strategy registries
policy class metadata
common immutable lookup tables
# 327. Warmup Must Not
No deberá cargar:
all users
all roles
all permissions
all tenant relationships
# 328. Memory Governance
Persistent workers requieren controlar:
registry growth
tenant cache growth
dynamic policy cache growth
# 329. Bounded Runtime Caches
Todo cache in-memory dinámico deberá poseer:
size limit
TTL
eviction
versioning
# 330. Never
static array $tenantPolicies = [];
sin límites en un worker de larga duración.
# 331. Worker Recycling
VoltStack podrá coordinar con Runtime:
max requests
memory threshold
generation update
para reciclar workers.
# 332. Authorization Shutdown
Al detener worker:
flush telemetry
flush audit buffers
release external clients
cuando corresponda.
# 333. Shutdown Is Not Request Reset
Separar:
Request Reset
de:
Worker Shutdown
# 334. Runtime Lifecycle
WORKER START
      ↓
Authorization Bootstrap
      ↓
Warmup
      ↓
READY
      ↓
Request
      ↓
Reset
      ↓
Request
      ↓
Reset
      ↓
...
      ↓
Shutdown
# 335. Developer Experience
El objetivo final deberá permitir que una aplicación básica solo necesite:
```php
final class InvoicePolicy
{
    public function update(
        User $user,
        Invoice $invoice
    ): bool {
        return $invoice->ownerId === $user->id;
    }
}
```
mientras VoltStack gestiona:
discovery
registration
compilation
container
context
lifecycle
reset
telemetry
cache
# 336. Progressive Complexity
Una aplicación simple podrá usar:
Policies + Gates
sin configurar:
ReBAC
Risk
Approval
SoD
External PDP
# 337. Enterprise Application
Una aplicación avanzada podrá activar:
RBAC
ABAC
ReBAC
Multi-Tenant
Delegation
Risk
Approval
SoD
Distributed Cache
Audit
External PDP
sin cambiar el Authorization Core.
# 338. Architectural Principle
Complexity should be opt-in.
Security should not be opt-out.
# 339. Complete Bootstrap Flow
VoltStack Runtime
       │
       ↓
Load Framework Configuration
       │
       ↓
Authorization Configuration Loader
       │
       ↓
Merge Defaults
       │
       ↓
Apply Security Floor
       │
       ↓
Validate Configuration
       │
       ↓
Authorization Service Provider
       │
       ↓
Register Core Contracts
       │
       ↓
Register Optional Subsystems
       │
       ↓
Register Plugins
       │
       ↓
Discover Policies / Gates / Voters
       │
       ↓
Discover Controller Metadata
       │
       ↓
Compile Registries
       │
       ↓
Compile Authorization Plans
       │
       ↓
Validate Security Invariants
       │
       ↓
Generate Manifest
       │
       ↓
Freeze Static Registries
       │
       ↓
Create Runtime Generation
       │
       ↓
Register Request Lifecycle
       │
       ↓
Warmup
       │
       ↓
AUTHORIZATION READY
# 340. Complete Request Flow
REQUEST START
      │
      ↓
Create Request Scope
      │
      ↓
Authentication Identity
      │
      ↓
Principal Factory
      │
      ↓
Actor Resolution
      │
      ↓
Tenant Resolution
      │
      ↓
Scope Resolution
      │
      ↓
AuthorizationContext
      │
      ↓
Application Execution
      │
      ├────→ Controller Authorization
      │
      ├────→ Gate
      │
      ├────→ Policy
      │
      ├────→ Domain Authorization
      │
      └────→ Approval Checks
      │
      ↓
Response
      │
      ↓
finally
      │
      ↓
Authorization Runtime Reset
      │
      ↓
REQUEST STATE DESTROYED
# 341. Relación con FrankenPHP
Este documento es especialmente importante para VoltStack porque FrankenPHP permite mantener el framework cargado entre requests.
El diseño deberá aprovechar:
compiled immutable authorization structures
sin permitir que persistan:
identity-dependent authorization state
La optimización buscada es:
Keep the expensive structure.
Destroy the request-specific security state.
# 342. Relación con Laravel
VoltStack conserva conceptos útiles del ecosistema Laravel:
Service Providers
Container bindings
Policies
Gates
Facades
configuration conventions
pero deberá reforzar explícitamente:
compiled authorization metadata
service lifetimes
persistent-worker safety
request scopes
registry freezing
policy generation
security invariant validation
# 343. Relación con Symfony
Del enfoque de Symfony resulta especialmente útil:
container compilation
service contracts
tagged services
compiler passes
voter architecture
service decoration
explicit dependency graph
VoltStack deberá combinar estas ideas con una API más directa para el desarrollador.
# 344. Filosofía VoltStack
El resultado buscado no será copiar:
Laravel Authorization
ni:
Symfony Security Authorization
sino construir una arquitectura donde:
Laravel-like Developer Experience
            +
Symfony-like Container Architecture
            +
Compiled Metadata
            +
Persistent Runtime Awareness
            +
Enterprise Authorization
            =
VoltStack Authorization Runtime
# 345. Resultado final
Después del bootstrap:
AuthorizationManagerInterface
deberá estar completamente preparado para ejecutar:
Policy authorization
Gate authorization
Voter authorization
RBAC
ABAC
ReBAC
Tenant isolation
Scope authorization
Ownership
Delegation
Impersonation
Capabilities
Contextual authorization
Risk-based authorization
Approval workflows
Dual control
Separation of duties
sin que el consumidor conozca:
cómo fueron descubiertos
cómo fueron registrados
cómo fueron compilados
cómo fueron construidos
cómo fueron cacheados
cómo serán reseteados
# 346. Conclusión
El 25_AUTHORIZATION_CONFIGURATION_BOOTSTRAP_AND_SERVICE_CONTAINER_INTEGRATION_SYSTEM.md establece la infraestructura que convierte todas las piezas definidas en los documentos 01–24 en un subsistema real del framework.
La separación fundamental queda:
BUILD / BOOT TIME
────────────────────────────────
Configuration
Discovery
Registration
Compilation
Validation
Container Graph
Registry Freeze
Manifest Generation

             ↓

RUNTIME
────────────────────────────────
Principal
Actor
Tenant
Scope
Context
Authorization Request
Decision
Audit

             ↓

REQUEST END
────────────────────────────────
Memoization Reset
Context Reset
Tenant Reset
Identity Reset
Security State Reset
Y para FrankenPHP:
┌──────────────────────────────────────────┐
│          PERSISTENT WORKER               │
│                                          │
│  AuthorizationConfig                    │
│  CompiledPolicyRegistry                 │
│  CompiledAbilityRegistry                │
│  CompiledMetadata                       │
│  StrategyRegistry                       │
│                                          │
│          SAFE TO PERSIST                 │
│                                          │
├──────────────────────────────────────────┤
│             REQUEST N                    │
│                                          │
│  Principal                              │
│  Actor                                  │
│  Tenant                                 │
│  Scope                                  │
│  Risk                                   │
│  Delegation                             │
│  Decision Memoization                   │
│                                          │
│        MUST BE DESTROYED                 │
│                                          │
└──────────────────────────────────────────┘
La regla arquitectónica definitiva de este documento es:
VoltStack deberá compilar y conservar todo aquello que sea estructural e inmutable, mientras que todo estado de autorización dependiente de una identidad, tenant, contexto u operación deberá estar estrictamente limitado al lifecycle correspondiente.

Con esto queda definido no solamente qué hace el Authorization System, sino cómo entra realmente a formar parte del runtime de VoltStack.
El siguiente documento de la secuencia es:
26_AUTHORIZATION_LIFECYCLE_EVENTS_HOOKS_AND_EXTENSION_POINTS_SYSTEM.md, donde debemos formalizar el lifecycle completo de una decisión —desde que nace el AuthorizationRequest hasta ALLOW, DENY, CHALLENGE o FAILURE— y definir exactamente qué eventos y hooks pueden observar, modificar o extender cada fase sin permitir bypasses de seguridad.
