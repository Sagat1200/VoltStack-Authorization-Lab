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

Al corte actual, `Quantum/Authorization` ya dispone de un core operativo consolidado, planner explainable, authority repositories `InMemory` + `Database`, memoization request-scoped, early-gate opt-in, ABAC runtime declarativo y una primera capa de resolucion automática tenant/scope (opt-in) integrada con metadata, routing, controllers y bridge opcional hacia `Controllers/Security`.

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
| Operativo        |       15 |
| Parcial          |       13 |
| Pendiente        |        4 |
| Total documentos |       32 |

## Matriz 01-32

| Doc | Area | Estado | Evidencia visible | Gap principal |
| --: | ---- | ------ | ----------------- | ------------- |
| 01 | Authorization Architecture | Operativo | la arquitectura objetivo ya tiene implementación real y extensible en `Quantum/Authorization`: core manager, planner por stages, metadata resolver, authority model, bridge Security y stores persistentes | falta seguir cerrando la alineacion de la arquitectura objetivo con ReBAC, delegation y coordinacion distribuida |
| 02 | Authorization Manager And Core Engine | Operativo | existen `AuthorizationManager`, `AuthorizationRequestFactory`, `AuthorizationRequest`, `DecisionResult`, `DecisionManager`, `AuthorizationDecisionPlan`, shortcuts `explain()/explainPlan()` y flujo `check/cannot/decide/authorize` con early-gate authority opt-in | faltan voters/strategies pluggables y una superficie operacional de auditoria mas profunda |
| 03 | Authorization Request Context And Subject Model | Operativo | existen `AuthorizationRequest`, `AuthorizationContext`, `SubjectDescriptor`, `SubjectResolver`, `Principal`, `AnonymousPrincipal` y `PrincipalResolver` | falta enriquecer el modelo para tenancy, relationships y contexto avanzado |
| 04 | Policy System And Policy Contracts | Operativo | existen `PolicyInterface`, `SupportsAuthorizationRequest`, atributos `#[PolicyFor]` y `#[HandlesAbility]`, mas policies classicas por metodo en `Quantum/Authorization/Policy/*` integradas a planner y manager | falta discovery/compilation mas profunda y composicion cross-surface mas rica |
| 05 | Policy Registry Discovery And Resolution System | Parcial | `PolicyRegistry` ya soporta multiples policies por subject, resolucion por jerarquia, carga desde config y discovery por `#[PolicyFor]` | falta discovery automatico global, compilacion y manifests |
| 06 | Policy Dispatcher Invocation And Result Normalization System | Operativo | `PolicyDispatcher` ya soporta `before()`, `PolicyInterface`, `SupportsAuthorizationRequest`, `#[HandlesAbility]` y normalizacion de `bool/null/DecisionResult` | falta pipeline mas profundo, hooks y telemetria dedicada |
| 07 | Gate System And Ability Registry | Operativo | existen `GateRegistry`, `Ability`, `AbilityRegistry`, `AbilityNormalizer` y pruebas sobre gates con principal bound y principal resuelto desde Auth | falta aliases, manifests y governance de abilities |
| 08 | Decision Manager Voters And Strategy System | Operativo | existe `DecisionManager` con estrategia base `default deny`; `AuthorizationManager` agrega resultados de gates/policies y opera en fail-closed | falta sistema de voters/strategies intercambiables y planner formal |
| 09 | Authorization Planner And Policy Pipeline System | Operativo | existen `AuthorizationPlannerInterface`, `AuthorizationPlanner`, enrichers previos a stages, stages explicitos para manifest/gates/policies, separacion formal entre request creation, enrichment, planning y finalizacion; `DecisionManager` aplica estrategia configurable; `ManifestRequirementsEnforcementStage` puede evaluar concretamente via authority y ABAC runtime; `AuthorizationDecisionPlan::explain()` entrega trazabilidad por stage; el manager soporta early-gate authority opt-in y `ControllerSecurityPlannerBridge` ofrece adaptador opcional hacia `Controllers/Security` | faltan telemetria operacional mas rica y evaluadores/voters externos enchufables |
| 10 | Authorization Attributes And Declarative Metadata System | Operativo | existen `#[Authorize]`, `#[PublicAccess]`, `#[AuthorizeWhen]`, DSL `Route::authorize()/publicAccess()/authorizeWhen()/authorizeWhenAll()`, proyeccion reusable hacia `authorization.public` y `authorization.requirements`, resolver propio del modulo, payload normalizado con fingerprint y `condition` serializable, `AuthorizationManifestStoreInterface` con stores InMemory y Filesystem, y `ManifestRequirementsEnforcementStage` que actua directamente sobre metadata del manifest | falta scope/tenant declarativo automatico y enforcement declarativo default-on por requirement |
| 11 | Controller Route And Action Authorization Integration System | Operativo | `ControllerEngine` ya consume `AuthorizationMetadataResolver`, proyecta metadata declarativa y de ruta hacia `AuthorizationManager`, resuelve subjects desde argumentos del controller, puede saltar enforcement con `publicAccess`, y el provider puede wiring opcionalmente `ControllerSecurityPlannerBridge`; `MetadataAuthorizationContextEnricher` expone fingerprint y requirements ya enriquecidos con `condition` | falta convivencia mas profunda con `Controllers/Security` y rollout uniforme a mas superficies no HTTP |
| 12 | Role Permission RBAC ABAC And ReBAC Integration System | Operativo | `Role`, `Permission`, `Scope`, `AttributeDefinition` como VOs en `Quantum/Authorization/Authority/*`; `AuthorityRepositoryInterface` con implementaciones `InMemoryAuthorityRepository`, `CachedAuthorityRepository` y `DatabaseAuthorityRepository`; `AuthorizationManager` soporta early-gate authority opt-in; `ManifestRequirementsEnforcementStage` y `AttributeConditionEvaluator` ejecutan RBAC/ABAC runtime sobre requirements declarativos; DSL `Condition` y `AuthorizeWhen` ya integran condiciones al pipeline | falta integracion ReBAC relacional y tenancy-aware relationships |
| 13 | Multi-Tenant Authorization And Data Isolation System | Parcial | `Scope` jerárquico `org:ws:proj` con `contains()`, `parent()`, wildcard `*`, `global`; repositorios authority (`InMemory` y `Database`) resuelven grants por `(principal_id, scope)`; `TenantScopeResolverInterface` + `TenantScopeResolver` (opt-in) normalizan `tenant.id`/`tenant_id` → `authorization.scope`, integrados con `AuthorizationContextFactory`, `AuthorizationManager` y `ManifestRequirementsEnforcementStage`; `ControllerEngine` ya proyecta `X-Tenant-Id`/tenant de runtime al contexto de Authorization; **nuevo:** el resolver ya entiende señales runtime adicionales (`Request`, `RouteMatch`, `controller.security.context`, route params `tenant|tenant_id|tenantId`) para reducir wiring manual | falta resolver tenant automático uniforme en mas superficies no HTTP, tenant enforcement transversal y isolation level granular por recurso |
| 14 | Authorization Cache Memoization And Decision Reuse System | Operativo | existen `AuthorityMemoizationCacheInterface`, `RequestScopedAuthorityMemoizationCache` y `CachedAuthorityRepository`; `authorization.authority.memoize=true` por defecto memoiza `effectivePermissionsForPrincipal(principalId, scope)` por request sin cambiar semantica; el hot path también puede cortar antes via early-gate authority opt-in | falta invalidacion/versionado distribuido de contexto entre workers y nodos |
| 17 | Authorization Testing Verification And Security Assurance System | Operativo | la suite cubre planner, manager, enricher/resolver de metadata, helpers, policies desde config, metadata declarativa en `ControllerEngine`, mapping HTTP, stores de manifest, authority models/repos, commands CLI, bridge Security, explainability (`AuthorizationDecisionPlanExplanationTest` 5), memoization (`AuthorityMemoizationAndCacheTest` 7), early-gate (`AuthorizationManagerAuthorityEarlyGateTest` 9), provider flags (`AuthorizationServiceProviderBridgeAndFlagsTest` 8), DB authority (`DatabaseAuthorityRepositoryTest` 2 + `AuthorizationServiceProviderDatabaseAuthorityTest` 3), ABAC runtime (`AttributeConditionEvaluatorTest` 5), metadata declarativa condicional (`MetadataEngineTest` 13), context enricher (`MetadataAuthorizationContextEnricherTest` 3), manifest stage (`ManifestRequirementsEnforcementStageTest` 16), runtime HTTP tenant propagation (`ControllerEngineTest` +1 escenario) y regresión focalizada `--filter=Authorization` **87 tests / 269 assertions exit 0** | faltan tests de ReBAC, tenancy automática cross-surface y proveedores externos no-DBAL |
| 18 | Authorization Compilation Optimization And Runtime Performance System | Parcial | `Quantum/Metadata`, bindings en `Application.php`, cache por request, payload con fingerprint estable, `AuthorizationManifestStore` persistente include-safe, `ManifestRequirementsEnforcementStage` como enforcement temprano, commands `authz:manifest:*`, memoization request-scoped de `effectivePermissions`, early-gate authority opt-in y `DatabaseAuthorityRepository` para evitar recomputo exclusivamente en memoria | faltan invalidacion selectiva por scope/contexto, budgets/gobernanza de recursos y estrategias lazy mas finas |
| 19 | Authorization Extensibility Plugin Provider And Custom Evaluator System | Parcial | block `authorization.authority.*` ya incluye flags `enabled`, `memoize`, `evaluate_requirements_concretely`, `evaluate_attribute_conditions`, `early_gate_enabled`, `driver`, `database.*`; `AuthorizationServiceProvider` resuelve providers `memory` y `database`; `AuthorityRepositoryInterface` y `AttributeConditionEvaluator` permiten extensiones; `ControllerSecurityPlannerBridge` sigue siendo adaptador opcional | falta plugin loader dinamico y providers externos adicionales (LDAP, OPA, Redis/remote) |
| 20 | Authorization Delegation Impersonation Capabilities And Service To Service System | Parcial | subject plano en `AuthorizationManager::check()` no requiere `RouteMatch`; multi-surface tests pasan con gate definitions sin HTTP context; `Principal` arbitrario puede construirse `new Principal('worker-1')` y usarse directamente en jobs/cli; bridge Security ↔ planner acepta Security principals arbitrarios y los normaliza | falta formalizar `impersonate()`, service-to-service Principal resolver y delegation grants scope-bound |
| 23 | Authorization Conditional Contextual And Risk-Based Access System | Parcial | `AttributeDefinition` typed (bool/string/int/float/array/enum/any) con constraints pattern/min-max/enum/required/default; `AttributeConditionEvaluator` ejecuta condiciones runtime sobre `AuthorizationContext` y `subject.*`; `ManifestRequirementsEnforcementStage` puede aplicar `condition` declarativa cuando `evaluate_attribute_conditions=true`; metadata declarativa soporta `#[AuthorizeWhen]`, `Route::authorizeWhen()/authorizeWhenAll()` y DSL `Condition::*` | falta risk scoring/adaptive access y politicas contextuales mas sofisticadas |
| 25 | Authorization Configuration Bootstrap And Service Container Integration System | Operativo | existen `AuthorizationServiceProvider`, defaults de config, `config/authorization.php`, facade y helpers; `Application.php` registra el provider base; el provider expone `commands()` para discovery automático de `authz:manifest:*`; block `authorization.authority.*` ya cubre `enabled`, `memoize`, `evaluate_requirements_concretely`, `evaluate_attribute_conditions`, `early_gate_enabled`, `driver`, `database.connection`, `database.tables.*` y **scope_resolution** (`enabled`, `scope_attribute_keys`, `tenant_attribute_keys`, `route_parameter_keys`, `request_header_keys`, `prefix`, `derive_from_tenant_id`), ademas del bridge opcional con `Controllers/Security`; inner authority repository vive en ciclo `scoped` para convivir bien con `DatabaseInterface` y caches request-scoped | falta discovery declarativo mas profundo de policies y bootstrap de providers externos |
| 27 | Authorization State Consistency Concurrency And Distributed Coordination System | Parcial | **nuevo:** InMemoryAuthorityRepository uses `$visited` array para break loops jerárquicos scope; scope() → parent() no produce infinite loops; todo stateless singleton o scoped-request | falta distributed coordination lock, cache invalidation multi-node y consistency audit |
| 29 | Authorization Administration Management And Operational Tooling System | Operativo | comandos CLI `authz:manifest:compile` y `authz:manifest:clear` con categorias `Authorization`, aliases y banderas `--verbose` / `--dry-run`, registrados automaticamente via `AuthorizationServiceProvider::commands()` y cubiertos por 12 tests unitarios; la explicacion del plan ya existe a nivel de runtime via `AuthorizationDecisionPlan::explain()` | faltan comandos de auditoria/revocacion y surfaces operativas que consuman explainability |
| 31 | Authorization Performance Compilation Optimization And Resource Governance System | Parcial | `evaluateRequirementsConcretely=true` y `authority.early_gate_enabled=true` pueden short-circuit gates/policies; seed de grants en PHP-array include-safe; `CachedAuthorityRepository` memoiza `effectivePermissions`; planner ya opera con fingerprint estable, manifest store persistente y hot path declarativo mas corto | faltan budgets/gobernanza de recursos, invalidacion cross-node y estrategias de cache multi-worker |

## Lectura ejecutiva del corte

### Lo que si existe hoy

1. un core real y reusable en `Quantum/Authorization`,
2. planner formal con stages, `AuthorizationDecisionPlan::explain()` y `explain()/explainPlan()` en manager,
3. bootstrap, config, facade, helpers y mapper de errores del modulo,
4. integración con Authentication para resolver principal/contexto,
5. metadata declarativa ampliada con `#[Authorize]`, `#[AuthorizeWhen]`, `Route::authorize()`, `authorizeWhen()` y `authorizeWhenAll()`,
6. proyeccion reusable de Authorization sobre `Quantum/Metadata` con payload/fingerprint/condition serializable,
7. integración con `ControllerEngine` y bridge opcional hacia `Controllers/Security`,
8. modelos RBAC/ABAC (`Role`, `Permission`, `Scope`, `AttributeDefinition`) + repositorios authority `InMemory`, `Cached` y `Database`,
9. memoization request-scoped de `effectivePermissions` y early-gate authority opt-in en `AuthorizationManager`,
10. `ManifestRequirementsEnforcementStage` ampliado para evaluar requirements concretos y condiciones ABAC runtime,
11. DSL `Condition::*` para declarar condiciones reutilizables fuera de atributos,
12. comandos CLI `authz:manifest:*` operativos y pruebas focalizadas del subsistema (`--filter=Authorization` 82 tests exit 0),
13. cobertura adicional de explainability, memoization, early-gate, DBAL authority, ABAC runtime y metadata declarativa condicional.

### Lo que no existe todavia

1. ReBAC relacional y evaluacion de relaciones sujeto-recurso,
2. tenant resolver automático desde request/surface y scope derivado cross-surface sin wiring manual,
3. invalidacion distribuida/versionado multi-worker para memoization y caches authority,
4. providers externos adicionales y lifecycle plugin mas rico,
5. risk scoring/adaptive access y tooling operacional de auditoria/revocacion.

## Prioridades reales segun la matriz

### Prioridad alta

1. `13`, `19`, `23`, `31`
2. `20`, `27`

Motivo:

- el subsistema ya se encuentra operativo en core, planner, metadata, authority, explainability y ABAC runtime declarativo,
- el siguiente retorno de valor esta en tenancy automatica, providers externos, performance distribuida y modelos relacionales,
- y la integracion profunda restante ya no es fundacional sino de endurecimiento y rollout operacional.

### Prioridad media

1. `05`, `18`, `29`
2. `01`

### Prioridad posterior

1. `15`, `21`, `22`, `24`, `26`, `28`, `30`, `32`

## Regla de mantenimiento

Cada vez que se cierre un bloque relevante del subsistema Authorization, esta matriz debe actualizar:

1. el estado del documento impactado,
2. la evidencia visible en codigo y tests,
3. el gap principal restante,
4. y la prioridad posterior.
