# VoltStack Authorization System

## 1. Propósito del documento

Este documento define el contexto general, alcance, principios arquitectónicos y objetivos del **Authorization System de VoltStack**.

El sistema de autorización será responsable de determinar si un **principal** puede realizar una determinada **acción** sobre un **subject**, teniendo en consideración el contexto de ejecución de la aplicación.

El sistema toma como referencia conceptos probados en frameworks como Laravel y Symfony, especialmente:

- Policies.
- Gates.
- Abilities.
- Voters.
- Decision Managers.
- Estrategias de decisión.
- Autorización declarativa.

VoltStack no pretende implementar una copia directa de ninguno de estos sistemas.

El objetivo es construir un **Authorization Engine propio**, integrado profundamente con la arquitectura del framework y preparado para aplicaciones tradicionales, SPA, sistemas empresariales, aplicaciones multi-tenant y runtimes persistentes como FrankenPHP.

---

## 2. Objetivo principal

El Authorization System debe proporcionar una API sencilla para el desarrollador:

```php
$user->can('update', $post);
```

```php
Authorization::check('update', $post);
```

```php
Authorization::authorize('update', $post);
```

pero internamente debe disponer de una arquitectura capaz de soportar:

```text
Principal
    +
Action
    +
Subject
    +
Context
    ↓
Authorization Engine
    ↓
GRANT / DENY / ABSTAIN
```

El sistema debe separar completamente:

```text
Authentication
```

de:

```text
Authorization
```

La autenticación responde:

```text
¿Quién es el usuario?
```

La autorización responde:

```text
¿Qué puede hacer?
```

---

## 3. Principios arquitectónicos

El sistema se diseñará alrededor de los siguientes principios.

## 3.1 Authorization as Infrastructure

La autorización será considerada infraestructura central del framework.

No será una característica exclusiva de:

```text
Models
```

ni de:

```text
Controllers
```

El mismo motor deberá poder utilizarse sobre cualquier tipo de recurso.

Por ejemplo:

```text
Model
Controller
Controller Action
Route
Component
Command
Job
Service
Resource
Tenant
Operation
```

---

## 4. Modelo general de autorización

Toda decisión de autorización estará conceptualmente representada por:

```text
Principal
Action
Subject
Context
```

Formalmente:

```text
AuthorizationRequest
{
    principal
    action
    subject
    context
}
```

Ejemplo:

```text
principal = User#42
action    = update
subject   = Invoice#783
tenant    = Tenant#7
channel   = web
```

La solicitud será procesada por:

```text
AuthorizationManager
        ↓
PolicyResolver
        ↓
PolicyPipeline
        ↓
DecisionManager
        ↓
DecisionResult
```

---

## 5. Principal

El `Principal` representa la entidad que intenta ejecutar una operación.

Normalmente será:

```text
User
```

pero el sistema no debe asumir que siempre existe un usuario tradicional.

Podrá representar:

```text
User
ServiceAccount
APIClient
MachineIdentity
SystemProcess
AnonymousPrincipal
TenantPrincipal
```

Esto permitirá utilizar el Authorization System tanto en aplicaciones web como en:

```text
CLI
Queues
Workers
APIs
Background Jobs
Distributed Systems
```

---

## 6. Action

La `Action` representa la operación que se intenta realizar.

Ejemplos:

```text
view
viewAny
create
update
delete
restore
forceDelete
approve
reject
publish
archive
download
execute
manage
```

VoltStack utilizará el término:

```text
Ability
```

como representación pública de estas acciones cuando resulte conveniente.

Ejemplo:

```php
$user->can('approve', $invoice);
```

---

## 7. Subject

El `Subject` representa aquello sobre lo que se intenta ejecutar una acción.

Puede ser una instancia:

```php
$post
```

o una clase:

```php
Post::class
```

Esto permitirá distinguir entre:

```text
create Post
```

y:

```text
update Post#42
```

Pero el sistema no estará limitado a modelos.

Un Subject podrá ser:

```text
Model
Controller
Controller Action
Route
Component
Command
Job
Service
Resource
Module
Tenant
```

Ejemplos:

```php
Authorization::check(
    'update',
    $invoice
);
```

```php
Authorization::check(
    'access',
    AdminController::class
);
```

```php
Authorization::check(
    'execute',
    ImportCustomersCommand::class
);
```

---

## 8. Authorization Context

Una decisión no siempre puede determinarse únicamente utilizando:

```text
User + Resource
```

Por ello VoltStack introducirá:

```text
AuthorizationContext
```

El contexto podrá contener información como:

```text
tenant
request
route
controller
channel
IP
authentication method
runtime
environment
metadata
attributes
```

Ejemplo conceptual:

```php
Authorization::check(
    ability: 'approve',
    subject: $loan,
    context: [
        'tenant' => $tenant,
        'channel' => 'web',
        'risk_score' => 0.25,
    ],
);
```

El contexto permitirá implementar modelos de autorización más avanzados.

---

## 9. Modelos de autorización

El Authorization System no estará acoplado exclusivamente a RBAC.

Debe poder soportar diferentes modelos.

## 9.1 RBAC

Role-Based Access Control.

```text
User
 ↓
Role
 ↓
Permission
```

Ejemplo:

```text
User
 └── Manager
      └── invoices.approve
```

---

## 9.2 ABAC

Attribute-Based Access Control.

Las decisiones pueden depender de atributos.

```text
User.department
Resource.department
Request.location
CurrentTime
Tenant
```

Ejemplo:

```text
user.department == invoice.department
```

---

## 9.3 PBAC

Policy-Based Access Control.

Las reglas se expresan mediante Policies.

```text
AuthorizationRequest
        ↓
Policies
        ↓
Decision
```

Este será uno de los modelos centrales de VoltStack.

---

## 9.4 ReBAC

Relationship-Based Access Control.

Las decisiones pueden depender de relaciones entre entidades.

Ejemplo:

```text
User
 ↓
belongs to
 ↓
Organization
 ↓
owns
 ↓
Project
```

El diseño del Authorization Engine no debe impedir futuras implementaciones de este modelo.

---

## 10. Policies

Las Policies serán la principal abstracción para organizar reglas de autorización relacionadas con recursos.

Ejemplo:

```php
final class PostPolicy
{
    public function update(
        User $user,
        Post $post
    ): DecisionResult {
        // ...
    }
}
```

Una Policy podrá implementar abilities como:

```text
view
create
update
delete
publish
archive
approve
```

---

## 11. Policies no limitadas a Models

VoltStack extenderá el concepto tradicional de Policy.

Una Policy podrá aplicarse sobre:

```text
Models
Controllers
Controller Actions
Routes
Commands
Jobs
Components
Services
Modules
Resources
```

Esto convierte las Policies en una infraestructura transversal.

---

## 12. Controller Policies

VoltStack soportará Policies aplicadas directamente sobre controladores.

Ejemplo:

```php
#[Policy(AdminControllerPolicy::class)]
final class AdminController
{
}
```

También podrá utilizarse autorización declarativa:

```php
#[Authorize('access-admin')]
final class AdminController
{
}
```

Esta autorización será evaluada antes de ejecutar cualquier acción del controlador.

---

## 13. Controller Action Policies

Las acciones individuales podrán declarar autorización.

Ejemplo:

```php
#[Authorize('update', subject: 'invoice')]
public function update(Invoice $invoice)
{
}
```

El framework deberá:

```text
Resolve Route
      ↓
Resolve Controller
      ↓
Resolve Arguments
      ↓
Resolve Subject
      ↓
Evaluate Authorization
      ↓
Execute Controller
```

si la autorización resulta satisfactoria.

---

## 14. Autorización por fases

No todas las reglas requieren que los argumentos del controlador hayan sido resueltos.

Por ello se contemplan diferentes fases.

```text
Route Match
     ↓
Pre-Resolution Authorization
     ↓
Argument Resolution
     ↓
Resource Authorization
     ↓
Controller Execution
```

Esto permitirá rechazar solicitudes tempranamente.

Ejemplo:

```text
/admin/invoices/928
```

Primero:

```text
¿Puede el usuario acceder al módulo administrativo?
```

Si la respuesta es:

```text
DENY
```

no será necesario cargar:

```text
Invoice#928
```

desde la base de datos.

---

## 15. Gates

VoltStack también proporcionará Gates para reglas que no necesiten estar organizadas alrededor de una Policy.

Ejemplo:

```php
Gate::define(
    'access-admin',
    fn (User $user) => $user->isAdmin()
);
```

Conceptualmente:

```text
Authorization System
│
├── Gates
│
└── Policies
```

Los Gates y las Policies utilizarán internamente el mismo Decision Engine.

---

## 16. Decision Model

Las decisiones internas no estarán limitadas a:

```text
true
false
```

VoltStack utilizará tres estados fundamentales:

```text
GRANT
DENY
ABSTAIN
```

Conceptualmente:

```php
enum Decision
{
    case Grant;
    case Deny;
    case Abstain;
}
```

`ABSTAIN` permitirá que una Policy indique:

```text
Esta Policy no tiene autoridad suficiente
para decidir esta solicitud.
```

Esto será fundamental para composición de Policies.

---

## 17. DecisionResult

Las decisiones podrán transportar información adicional.

Conceptualmente:

```php
final readonly class DecisionResult
{
    public function __construct(
        public Decision $decision,
        public ?string $reason = null,
        public array $metadata = [],
    ) {}
}
```

Ejemplo:

```text
DENY

reason:
"Invoice belongs to another tenant."
```

Esto permitirá construir:

```text
debugging
observability
audit trails
profiling
security analysis
```

---

## 18. Decision Manager

Cuando múltiples Policies participen en una decisión, el:

```text
DecisionManager
```

será responsable de calcular el resultado final.

Ejemplo:

```text
TenantPolicy      → GRANT
OwnershipPolicy   → GRANT
RolePolicy        → ABSTAIN
CompliancePolicy  → DENY
```

El resultado dependerá de la estrategia configurada.

---

## 19. Decision Strategies

El sistema deberá permitir estrategias como:

```text
Affirmative
Unanimous
Consensus
Priority
FirstApplicable
```

Las aplicaciones podrán seleccionar diferentes estrategias dependiendo del contexto.

Para operaciones críticas podría utilizarse:

```text
Unanimous
```

mientras otras operaciones podrían utilizar:

```text
Affirmative
```

---

## 20. Policy Pipeline

Las Policies podrán organizarse mediante un pipeline.

Ejemplo:

```text
AuthorizationRequest
        ↓
SuspendedUserPolicy
        ↓
TenantIsolationPolicy
        ↓
SecurityPolicy
        ↓
ResourcePolicy
        ↓
CompliancePolicy
        ↓
DecisionManager
```

Esto permitirá implementar reglas transversales sin duplicarlas en cada Policy.

---

## 21. Global Policies

VoltStack permitirá registrar Policies globales.

Ejemplos:

```text
SuspendedUserPolicy
MaintenancePolicy
TenantIsolationPolicy
SecurityContextPolicy
```

Estas Policies podrán ejecutarse antes de las Policies específicas del recurso.

---

## 22. Resource Policies

Las Policies tradicionales seguirán siendo soportadas.

Ejemplo:

```text
Post
    ↓
PostPolicy

Invoice
    ↓
InvoicePolicy

Order
    ↓
OrderPolicy
```

Esto conserva una experiencia familiar para desarrolladores provenientes de Laravel.

---

## 23. Policy Discovery

VoltStack deberá descubrir Policies automáticamente mediante convenciones.

Ejemplo:

```text
App\Models\Post
        ↓
App\Policies\PostPolicy
```

El desarrollador no deberá registrar manualmente cada Policy cuando se respeten las convenciones establecidas.

---

## 24. Policy Registry

Las asociaciones descubiertas serán almacenadas en:

```text
PolicyRegistry
```

Conceptualmente:

```text
Post
    → PostPolicy

Invoice
    → InvoicePolicy

Order
    → OrderPolicy
```

El registro también deberá permitir asociaciones explícitas.

---

## 25. Policy Compilation

En producción, VoltStack podrá compilar las asociaciones y metadata de autorización.

Por ejemplo:

```text
Policy Discovery
      ↓
Attribute Scanning
      ↓
Policy Metadata
      ↓
Policy Compiler
      ↓
Compiled Authorization Map
```

Esto evitará reflexión y descubrimiento repetitivo durante cada request.

---

## 26. Policy Cache

El sistema podrá almacenar:

```text
Policy mappings
Attribute metadata
Controller authorization metadata
Route authorization metadata
Decision configuration
```

en una caché compilada.

Esto será especialmente importante para:

```text
FrankenPHP
long-running workers
high-throughput APIs
```

---

## 27. Autorización declarativa

VoltStack soportará atributos PHP.

Ejemplo:

```php
#[Authorize('update', subject: 'post')]
public function update(Post $post)
{
}
```

También podrán existir atributos especializados:

```php
#[RequiresRole('admin')]
```

```php
#[RequiresPermission('posts.update')]
```

```php
#[Policy(PostPolicy::class)]
```

Estos atributos serán transformados internamente en solicitudes normales al Authorization Engine.

---

## 28. Integración con Routing

Las rutas podrán declarar requisitos de autorización.

Ejemplo conceptual:

```php
Route::put('/posts/{post}', [PostController::class, 'update'])
    ->can('update', 'post');
```

El pipeline será:

```text
Route Match
     ↓
Route Binding
     ↓
Authorization
     ↓
Controller
```

---

## 29. Integración con Controllers

El Authorization System deberá integrarse directamente con el Controller System de VoltStack.

Pipeline conceptual:

```text
HTTP Request
     ↓
Routing
     ↓
Controller Resolution
     ↓
Authentication
     ↓
Pre-Authorization
     ↓
Argument Resolution
     ↓
Resource Authorization
     ↓
Controller Execution
     ↓
Result Transformation
     ↓
Response Transport
```

La autorización será una fase formal del lifecycle del controlador.

---

## 30. Integración con Components

Los componentes podrán consultar capacidades:

```php
$this->can('update', $post);
```

También podrán utilizar directivas:

```php
@can('update', $post)

@endcan
```

El sistema de componentes no implementará su propio motor de autorización.

Utilizará siempre:

```text
Authorization System
```

---

## 31. Integración SPA

VoltStack podrá proporcionar al frontend información de capacidades.

Ejemplo:

```json
{
    "invoice": {
        "view": true,
        "update": true,
        "delete": false
    }
}
```

Esto podrá utilizarse para mejorar la experiencia del usuario.

Por ejemplo:

```text
hide button
disable action
prevent navigation
```

Sin embargo:

> La autorización frontend nunca será considerada una frontera de seguridad.

Toda operación deberá volver a ser autorizada por el servidor.

---

## 32. Multi-Tenancy

El Authorization System deberá integrarse nativamente con el contexto multi-tenant.

Una Policy podrá acceder a:

```text
TenantContext
```

y validar:

```text
Principal Tenant
        =
Resource Tenant
```

El sistema podrá incorporar una:

```text
TenantIsolationPolicy
```

global.

Esto permitirá prevenir accesos accidentales entre tenants.

---

## 33. Seguridad por defecto

El Authorization System seguirá el principio:

```text
Default Deny
```

Si ninguna regla autoriza explícitamente una operación protegida:

```text
DENY
```

El framework deberá evitar configuraciones ambiguas que accidentalmente concedan acceso.

---

## 34. Fail Closed

Ante errores internos de autorización:

```text
Policy exception
Invalid context
Unknown subject
Missing principal
Invalid metadata
```

el sistema deberá favorecer:

```text
DENY
```

sobre:

```text
ALLOW
```

para operaciones protegidas.

Las excepciones y estrategias específicas se definirán posteriormente.

---

## 35. Observabilidad

Cada decisión podrá producir información de diagnóstico.

Ejemplo:

```text
Authorization Trace

Principal:
User#42

Ability:
invoice.approve

Subject:
Invoice#928

Policies:

TenantIsolationPolicy → GRANT
RolePolicy            → GRANT
InvoicePolicy         → GRANT
CompliancePolicy      → DENY

Final:
DENY

Reason:
Compliance approval required.
```

Esta información deberá estar disponible para herramientas de desarrollo y profiling sin exponer información sensible al usuario final.

---

## 36. Auditoría

El sistema deberá permitir registrar decisiones relevantes.

Especialmente:

```text
administrative actions
financial operations
security changes
tenant administration
privileged operations
data exports
destructive actions
```

El sistema de auditoría deberá poder conocer:

```text
principal
action
subject
decision
reason
timestamp
tenant
context
```

---

## 37. Performance

La autorización puede ejecutarse muchas veces durante una sola request.

Por ello deberán evitarse:

```text
reflection repetitiva
policy discovery repetitivo
container resolution innecesario
database queries duplicadas
metadata parsing repetitivo
```

El sistema utilizará:

```text
compiled metadata
registries
request-scoped caches
runtime caches
```

cuando resulte seguro hacerlo.

---

## 38. Runtimes persistentes

VoltStack está diseñado para aprovechar runtimes persistentes como FrankenPHP.

El Authorization System deberá evitar:

```text
state leakage
principal leakage
tenant leakage
request context leakage
decision cache leakage
```

entre requests.

La metadata inmutable podrá mantenerse en memoria.

Los datos relacionados con:

```text
User
Tenant
Request
AuthorizationContext
```

deberán ser correctamente aislados por request.

---

## 39. Extensibilidad

El sistema deberá permitir añadir:

```text
Custom Policies
Custom Gates
Custom Decision Strategies
Custom Principal Resolvers
Custom Subject Resolvers
Custom Context Providers
Custom Policy Resolvers
Custom Authorization Attributes
Custom Auditors
Custom Observability Providers
```

sin modificar el núcleo.

---

## 40. Arquitectura conceptual

```text
                    Application
                         │
             ┌───────────┴───────────┐
             │                       │
          $user->can()         #[Authorize]
             │                       │
             └───────────┬───────────┘
                         ↓
              AuthorizationManager
                         │
                         ↓
              AuthorizationRequest
                         │
             ┌───────────┼───────────┐
             ↓           ↓           ↓
         Principal     Subject     Context
         Resolver      Resolver     Builder
             └───────────┼───────────┘
                         ↓
                   Gate / Policy
                     Resolver
                         │
                         ↓
                  Policy Pipeline
                         │
          ┌──────────────┼──────────────┐
          ↓              ↓              ↓
       Global         Resource        Context
       Policies       Policies        Policies
          └──────────────┼──────────────┘
                         ↓
                  DecisionManager
                         │
                         ↓
                  DecisionResult
                         │
                ┌────────┼────────┐
                ↓        ↓        ↓
              GRANT     DENY    ABSTAIN
                         │
                         ↓
                Trace / Audit /
                Observability
```

---

## 41. Componentes principales previstos

El sistema estará compuesto inicialmente por:

```text
AuthorizationManager
AuthorizationRequest
AuthorizationContext

PrincipalResolver
SubjectResolver

GateManager
GateRegistry

PolicyManager
PolicyRegistry
PolicyResolver
PolicyDiscovery
PolicyDispatcher

PolicyPipeline

Decision
DecisionResult
DecisionManager
DecisionStrategy

AuthorizationMetadata
AuthorizationAttributeResolver

PolicyCompiler
PolicyCache

AuthorizationTrace
AuthorizationProfiler
AuthorizationAuditor
```

Estos componentes podrán dividirse posteriormente en subsistemas especializados.

---

## 42. Ubicación dentro de VoltStack

El sistema podrá implementarse como un módulo Quantum:

```text
src/
└── Quantum/
    └── Authorization/
```

Estructura conceptual inicial:

```text
Authorization/
├── Contracts/
├── Core/
├── Context/
├── Decisions/
├── Gates/
├── Policies/
├── Pipeline/
├── Resolvers/
├── Attributes/
├── Metadata/
├── Compilation/
├── Cache/
├── Integration/
├── Observability/
├── Audit/
├── Exceptions/
└── Support/
```

La estructura definitiva será definida en el documento correspondiente de arquitectura.

---

## 43. Dependencias

El Authorization System podrá depender de abstracciones proporcionadas por:

```text
Container
Config
Authentication
Routing
Http
HttpKernel
Controllers
Metadata
Cache
Events
Observability
```

pero deberá evitar dependencias innecesarias con implementaciones concretas.

En particular:

```text
Authorization
```

no deberá depender obligatoriamente del ORM.

Esto permitirá utilizar Policies sobre objetos que no sean entidades persistentes.

---

## 44. Relación con Authentication

Authentication y Authorization serán subsistemas diferentes.

```text
Authentication
     ↓
Principal
     ↓
Authorization
     ↓
Decision
```

El Authorization System consumirá un Principal proporcionado por Authentication, pero no será responsable de autenticarlo.

---

## 45. Relación con Security

El Authorization System formará parte del modelo general de seguridad de VoltStack, pero no reemplazará otros mecanismos.

```text
Security
│
├── Authentication
├── Authorization
├── CSRF
├── Session Security
├── Input Validation
├── Rate Limiting
├── Tenant Isolation
├── Transport Security
└── Security Observability
```

Authorization será responsable específicamente de:

```text
access decisions
```

---

## 46. Objetivos de diseño

El sistema deberá conseguir simultáneamente:

### Developer Experience

API sencilla y familiar.

### Security

Default deny y fail closed.

### Performance

Metadata compilable y mínima resolución dinámica.

### Extensibility

Policies y estrategias personalizables.

### Observability

Decisiones explicables.

### Multi-Tenancy

Aislamiento integrado.

### Runtime Safety

Compatible con FrankenPHP y workers persistentes.

### Framework Integration

Integración directa con Routing, Controllers, Components, Commands y Jobs.

---

## 47. Filosofía del sistema

La filosofía fundamental será:

```text
Simple API.
Explicit decisions.
Composable policies.
Centralized authorization.
Secure defaults.
Observable execution.
```

El desarrollador deberá poder escribir simplemente:

```php
$user->can('update', $invoice);
```

mientras VoltStack internamente puede ejecutar:

```text
Principal Resolution
        ↓
Context Construction
        ↓
Policy Resolution
        ↓
Global Policies
        ↓
Tenant Policies
        ↓
Resource Policies
        ↓
Decision Strategy
        ↓
Decision Result
        ↓
Audit / Trace
```

La complejidad arquitectónica debe permanecer dentro del framework y no trasladarse innecesariamente al desarrollador.

---

## 48. Resultado esperado

Al finalizar la implementación, VoltStack deberá disponer de un Authorization System capaz de proporcionar una experiencia sencilla para aplicaciones pequeñas y, al mismo tiempo, soportar requisitos de autorización complejos en aplicaciones empresariales.

El sistema combinará:

```text
Laravel-style Developer Experience
            +
Symfony-style Decision Architecture
            +
VoltStack Compilation
            +
VoltStack Multi-Tenancy
            +
VoltStack Observability
            +
VoltStack Persistent Runtime Safety
```

El resultado será un sistema de autorización transversal, extensible y desacoplado del ORM que pueda convertirse en una de las infraestructuras fundamentales de VoltStack
