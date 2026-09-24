# DEVELOPMENT_MATRIX

## Proposito

Esta matriz controla el estado real del desarrollo del subsistema `Quantum/Authorization` frente a la documentacion arquitectonica ubicada en `vendor/voltstack/authorization-lab/Docs`.

El criterio del corte es conservador y se basa en evidencia visible en:

- `vendor/voltstack/framework/src/Quantum/Authorization`
- `vendor/voltstack/framework/src/Quantum/Controllers/Security`
- `vendor/voltstack/framework/src/Quantum/Metadata`
- `vendor/voltstack/framework/src/Quantum/Auth`
- `vendor/voltstack/framework/src/Platform/Application.php`
- `vendor/voltstack/framework/tests/Unit/ControllerSecurityModelTest.php`
- `vendor/voltstack/framework/tests/Unit/PolicyCompositionTest.php`
- `vendor/voltstack/framework/tests/Unit/PolicyWorkerSafetyTest.php`

## Nota de corte

Al corte actual, `Quantum/Authorization` ya dispone de un core minimo implementado, mas una primera capa declarativa integrada con controllers, routing y manejo de errores HTTP.

Por tanto:

- `Operativo` ya aplica a los bloques fundacionales efectivamente aterrizados en codigo,
- `Parcial` sigue significando infraestructura reusable o implementacion incompleta respecto del documento arquitectonico,
- `Pendiente` queda reservado a los bloques aun no iniciados de forma real.

## Leyenda

- `Operativo`: existe implementacion usable del bloque dentro de `Quantum/Authorization` o integracion formal del modulo Authorization en runtime.
- `Parcial`: existe infraestructura adyacente reutilizable, pero no el subsistema Authorization propiamente cerrado.
- `Pendiente`: no hay evidencia suficiente para considerar el bloque empezado de forma real.

## Resumen del corte

| Estado           | Cantidad |
| ---------------- | -------: |
| Operativo        |       10 |
| Parcial          |       15 |
| Pendiente        |        7 |
| Total documentos |       32 |

## Matriz 01-32

| Doc | Area | Estado | Evidencia visible | Gap principal |
| --: | ---- | ------ | ----------------- | ------------- |
| 01 | Authorization Architecture | Parcial | arquitectura objetivo detallada en `Docs/01...`; infraestructura adyacente ya presente en `Controllers/Security`, `Metadata` y `Application.php` | falta implementacion propia de `Quantum/Authorization` |
| 02 | Authorization Manager And Core Engine | Operativo | existen `AuthorizationManager`, `AuthorizationRequestFactory`, `AuthorizationRequest`, `DecisionResult`, `DecisionManager` y flujo `check/cannot/decide/authorize` dentro de `Quantum/Authorization` | falta planner formal, extensibilidad avanzada y mayor cobertura de integracion |
| 03 | Authorization Request Context And Subject Model | Operativo | existen `AuthorizationRequest`, `AuthorizationContext`, `SubjectDescriptor`, `SubjectResolver`, `Principal`, `AnonymousPrincipal` y `PrincipalResolver` | falta enriquecer el modelo para tenancy, relationships y contexto avanzado |
| 04 | Policy System And Policy Contracts | Operativo | existen `PolicyInterface`, `SupportsAuthorizationRequest`, atributos `#[PolicyFor]` y `#[HandlesAbility]`, mas policies classicas por metodo en `Quantum/Authorization/Policy/*` | falta planner/pipeline formal para composicion avanzada |
| 05 | Policy Registry Discovery And Resolution System | Parcial | `PolicyRegistry` ya soporta multiples policies por subject, resolucion por jerarquia, carga desde config y discovery por `#[PolicyFor]` | falta discovery automatico global, compilacion y manifests |
| 06 | Policy Dispatcher Invocation And Result Normalization System | Operativo | `PolicyDispatcher` ya soporta `before()`, `PolicyInterface`, `SupportsAuthorizationRequest`, `#[HandlesAbility]` y normalizacion de `bool/null/DecisionResult` | falta pipeline mas profundo, hooks y telemetria dedicada |
| 07 | Gate System And Ability Registry | Operativo | existen `GateRegistry`, `Ability`, `AbilityRegistry`, `AbilityNormalizer` y pruebas sobre gates con principal bound y principal resuelto desde Auth | falta aliases, manifests y governance de abilities |
| 08 | Decision Manager Voters And Strategy System | Operativo | existe `DecisionManager` con estrategia base `default deny`; `AuthorizationManager` agrega resultados de gates/policies y opera en fail-closed | falta sistema de voters/strategies intercambiables y planner formal |
| 09 | Authorization Planner And Policy Pipeline System | Parcial | existen `AuthorizationPlannerInterface`, `AuthorizationPlanner`, enrichers previos a stages, stages explicitos para gates y policies, y separacion formal entre request creation, enrichment, planning y finalizacion de decisiones; `DecisionManager` ya aplica estrategia configurable | falta pipeline compilable, enrichment stages avanzados y manifests de metadata |
| 10 | Authorization Attributes And Declarative Metadata System | Operativo | existen `#[Authorize]`, `#[PublicAccess]`, DSL `Route::authorize()` y `Route::publicAccess()`, mas proyeccion reusable hacia `authorization.public` y `authorization.requirements` sobre `Quantum/Metadata`, un resolver propio del modulo y payload normalizado con fingerprint | falta compilacion unificada sobre manifests/artifacts del modulo |
| 11 | Controller Route And Action Authorization Integration System | Operativo | `ControllerEngine` ya consume `AuthorizationMetadataResolver`, proyecta metadata declarativa y de ruta hacia `AuthorizationManager`, resuelve subjects desde argumentos del controller y puede saltar enforcement con `publicAccess` | falta convivencia mas profunda con `Controllers/Security` y rollout a mas superficies |
| 12 | Role Permission RBAC ABAC And ReBAC Integration System | Parcial | policies compuestas y `SecurityAttributes` permiten roles, permissions y atributos simples; tests de expresiones en `PolicyCompositionTest.php` | falta modelo real de RBAC/ABAC/ReBAC, repositorios y evaluadores |
| 13 | Multi Tenant Authorization And Data Isolation System | Parcial | `TenantRequired`, `TenantIdentity`, validaciones de tenant en `ControllerSecurityDecisionEngine` | falta tenant isolation como subsistema general de Authorization |
| 14 | Authorization Cache Memoization And Decision Reuse System | Parcial | `SecurityDecisionCache`, request-scoped cache por clave y pruebas de worker safety | falta memoization y cache del modulo con fingerprints y versionado de contexto |
| 15 | Authorization Audit Observability Tracing And Explainability System | Pendiente | no existe trazabilidad ni auditoria propia de Authorization | falta trace, audit y explicabilidad del modulo |
| 16 | Authorization Failure Error Denial And Exception Handling System | Operativo | existen `AuthorizationDeniedException`, `AuthorizationChallengeException`, `AuthorizationEvaluationException`, `AuthorizationExceptionMapper` y registro en `ExceptionHandler` con respuestas `401/403/500` coherentes | falta enriquecer telemetria y contratos de error del planner futuro |
| 17 | Authorization Testing Verification And Security Assurance System | Parcial | la suite ya cubre planner, manager, enricher de metadata, resolver de metadata, helpers, policies desde config, metadata declarativa en `ControllerEngine` y mapping HTTP en `ExceptionHandlingTest.php` y `QuantumExceptionHandlerTest.php` | falta ampliar cobertura a metadata compilada y escenarios multi-surface |
| 18 | Authorization Compilation Optimization And Runtime Performance System | Parcial | `Quantum/Metadata`, bindings en `Application.php`, budget de evaluacion, cache por request, orientación a runtime persistente, proyeccion de metadata de Authorization compatible con modo compilado, enrichment contextual previo al pipeline y payload con fingerprint estable | falta compilador, manifest y hot path propio del modulo |
| 19 | Authorization Extensibility Plugin Provider And Custom Evaluator System | Parcial | registro de policies lazy, expresiones compuestas, infraestructura de metadata extensible | falta modelo formal de plugins, evaluators y extension points del modulo |
| 20 | Authorization Delegation Impersonation Capabilities And Service To Service System | Parcial | `PrincipalType` en controllers ya contempla `service`, `api_client`, `system`, `impersonated_user` | falta delegacion, capabilities, envelopes y evaluadores reales |
| 21 | Authorization Resource Ownership Sharing And Relationship Access System | Pendiente | solo hay hints ligeros como `owner:true` en expresiones de test | falta ownership, sharing y relationships como conceptos de primer nivel |
| 22 | Authorization Hierarchical Scopes Organizations Teams And Workspaces System | Pendiente | no existe modelo jerarquico de scopes visible | falta scope model y herencia |
| 23 | Authorization Conditional Contextual And Risk Based Access System | Parcial | `SecurityAttributes`, contexto de auth, tenant, scopes y atributos permiten condicionamiento basico | falta risk engine y contexto canonico del modulo |
| 24 | Authorization Approval Workflow Dual Control And Separation Of Duties System | Pendiente | sin evidencia suficiente | falta approval, SoD y challenge asociado |
| 25 | Authorization Configuration Bootstrap And Service Container Integration System | Operativo | existen `AuthorizationServiceProvider`, defaults de config, `config/authorization.php`, facade y helpers; `Application.php` ya registra el provider base | falta configuracion avanzada, discovery de policies y bootstrap declarativo |
| 26 | Authorization Lifecycle Events Hooks And Extension Points System | Pendiente | no hay lifecycle ni hooks del modulo | falta pipeline lifecycle formal y eventos |
| 27 | Authorization State Consistency Concurrency And Distributed Coordination System | Parcial | `PolicyEvaluationSandbox`, `HardenedControllerSecurityDecisionEngine`, recursion guard, circuit breaker, timeout, worker disposition | falta consistency model propio de Authorization y coordinacion distribuida |
| 28 | Authorization Data Model Persistence And Storage Boundaries System | Pendiente | no hay repositorios ni storage de Authorization | falta boundary de persistencia del subsistema |
| 29 | Authorization Administration Management And Operational Tooling System | Pendiente | no hay comandos ni tooling administrativo de Authorization | falta plano operativo del modulo |
| 30 | Authorization Testing Verification Security Assurance And Compliance System | Parcial | existen tests de seguridad, aislamiento y composición sobre infraestructura adyacente | falta harness, contract tests y compliance tests propios del modulo |
| 31 | Authorization Performance Compilation Optimization And Resource Governance System | Parcial | budget de evaluacion, timeout, circuit breaker, request cache y worker safety en controllers | falta resource governance y optimizacion formal del modulo |
| 32 | Authorization System Integration And Final Architecture | Parcial | documentación final completa y base técnica repartida entre `Controllers/Security`, `Metadata`, `Auth` y `Application.php` | falta aterrizar la arquitectura maestra en `Quantum/Authorization` |

## Lectura ejecutiva del corte

### Lo que si existe hoy

1. un core minimo real en `Quantum/Authorization`,
2. gates, abilities, request model y decision model funcionales,
3. bootstrap, config, facade, helpers, planner y mapper de errores del modulo,
4. integración con Authentication para resolver principal/contexto,
5. metadata declarativa inicial con atributos y DSL de rutas,
6. proyeccion reusable de Authorization sobre `Quantum/Metadata`,
7. integración inicial con `ControllerEngine`,
8. pruebas unitarias y feature ampliadas del subsistema,
9. un subsistema fuerte de `Controllers/Security` y metadata reutilizable alrededor.

### Lo que no existe todavia

1. contracts completos de policies,
2. discovery/compilacion formal de policies y metadata,
3. planner/pipeline del modulo,
4. integración declarativa con controllers/routing,
5. modelos avanzados de tenancy, relationships, risk, approvals y tooling.

## Prioridades reales segun la matriz

### Prioridad alta

1. `02`, `03`, `04`, `06`, `07`, `08`, `10`, `11`, `16`, `25`
2. `05`, `09`, `17`, `18`, `32`

Motivo:

- el modulo no existe aun,
- pero ya hay bastante infraestructura adyacente para arrancarlo con criterio.

### Prioridad media

1. `12`, `13`, `14`, `19`, `23`, `27`, `31`
2. `20`

### Prioridad posterior

1. `15`, `21`, `22`, `24`, `26`, `28`, `29`, `30`

## Regla de mantenimiento

Cada vez que se cierre un bloque relevante del subsistema Authorization, esta matriz debe actualizar:

1. el estado del documento impactado,
2. la evidencia visible en codigo y tests,
3. el gap principal restante,
4. y la prioridad posterior.
