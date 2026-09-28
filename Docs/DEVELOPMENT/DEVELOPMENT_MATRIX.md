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
| Operativo        |       13 |
| Parcial          |       15 |
| Pendiente        |        4 |
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
| 09 | Authorization Planner And Policy Pipeline System | Operativo | existen `AuthorizationPlannerInterface`, `AuthorizationPlanner`, enrichers previos a stages, stages explicitos para gates y policies, y separacion formal entre request creation, enrichment, planning y finalizacion de decisiones; `DecisionManager` ya aplica estrategia configurable; `ManifestRequirementsEnforcementStage` antepuesto a gates/policies que aplica enforcement directo (public→ALLOW, ability fuera de whitelist requirements→DENY/ABSTAIN); planner propaga fingerprint del contexto a TODOS los `DecisionResult` via `applyFingerprint()`; stage ampliado con evaluacion concreta `evaluateRequirementsConcretely=true` contra `AuthorityRepository` y `effect=deny` explicito gana siempre; **nuevo convergencia:** `ControllerSecurityPlannerBridge` ofrece adaptador opcional que invoca planner desde Controllers/Security sin romper Hardened engine | falta explainability del plan y hot path optimizado |
| 10 | Authorization Attributes And Declarative Metadata System | Operativo | existen `#[Authorize]`, `#[PublicAccess]`, DSL `Route::authorize()` y `Route::publicAccess()`, mas proyeccion reusable hacia `authorization.public` y `authorization.requirements` sobre `Quantum/Metadata`, un resolver propio del modulo, payload normalizado con fingerprint, `AuthorizationManifestStoreInterface` con stores InMemory y Filesystem, y `ManifestRequirementsEnforcementStage` que actua directamente sobre metadata del manifest; stage soporta efecto explícito `deny` y evaluacion concreta via authority cuando `evaluateRequirementsConcretely=true`; **nuevo:** 12 tests unitarios commands `authz:manifest:compile/clear` pasan 0 warnings | faltan ABAC runtime evaluator y enforcement default-on por requirement |
| 11 | Controller Route And Action Authorization Integration System | Operativo | `ControllerEngine` ya consume `AuthorizationMetadataResolver`, proyecta metadata declarativa y de ruta hacia `AuthorizationManager`, resuelve subjects desde argumentos del controller y puede saltar enforcement con `publicAccess`; `MetadataAuthorizationContextEnricher` ya expone el fingerprint del manifest en `authorization.metadata.fingerprint`; todos los DecisionResult del planner ya llevan fingerprint visible | falta convivencia mas profunda con `Controllers/Security` y rollout a mas superficies |
| 12 | Role Permission RBAC ABAC And ReBAC Integration System | Operativo | `Role`, `Permission`, `Scope`, `AttributeDefinition` como VOs en `Quantum/Authorization/Authority/*`; `AuthorityRepositoryInterface` + `InMemoryAuthorityRepository` con seed desde config `authorization.authority.grants` y herencia scope jerárquica upward; `ManifestRequirementsEnforcementStage` opt-in (`evaluateRequirementsConcretely=true`) evalúa cada requirement via authority; 9 tests authority + 12 tests stage + 5 tests multi-surface + 12 tests commands + 5 tests bridge pasando; SUITE ACUMULADA: 107 Unit + 74 Feature exit 0 | falta integracion ReBAC y attribute evaluation runtime ABAC directo en AuthorizationManager.check() |
| 13 | Multi-Tenant Authorization And Data Isolation System | Parcial | **nuevo:** `Scope` jerárquico `org:ws:proj` con `contains()`, `parent()`, wildcard `*`, `global`; `InMemoryAuthorityRepository` soporta grants con herencia ascendente por scope; seed config por (principal_id, scope) | falta tenant resolver automático del contexto de request, tenant enforcement en policy runtime, isolation level por scope granular |
| 14 | Authorization Cache Memoization And Decision Reuse System | Parcial | `SecurityDecisionCache`, request-scoped cache por clave, tests de worker safety y manifest store persistente con fingerprint estable que evita recalculos de metadata | falta memoization del propio planner effectivePermissions y versionado de contexto para invalidacion selectiva |
| 17 | Authorization Testing Verification And Security Assurance System | Operativo | la suite ya cubre planner, manager, enricher de metadata, resolver de metadata, helpers, policies desde config, metadata declarativa en `ControllerEngine`, mapping HTTP en `ExceptionHandlingTest.php` y `QuantumExceptionHandlerTest.php`, stores de manifest, integracion resolver+manifest+enricher, configuraciones del provider (`manifest.enabled=false`, `manifest.path` real) via `AuthorizationServiceProviderTest.php` (6 tests), enforcement stage via `ManifestRequirementsEnforcementStageTest.php` (12 tests: 8 legacy + 4 concrete authority); **nuevo coverage:** `AuthorityModelAndRepositoryTest` (9 tests), `AuthorizationMultiSurfaceIntegrationTest` (5 tests CLI/Jobs sin RouteMatch), `AuthorizationManifestCommandsTest` (12 tests commands CLI), `ControllerSecurityPlannerBridgeTest` (5 tests convergencia security+planner); **SUITE COMPLETA 107+74 tests / 393+901 assertions exit 0** | faltan tests de ReBAC y ABAC runtime |
| 18 | Authorization Compilation Optimization And Runtime Performance System | Parcial | `Quantum/Metadata`, bindings en `Application.php`, budget de evaluacion, cache por request, orientación a runtime persistente, proyeccion de metadata de Authorization compatible con modo compilado, enrichment contextual previo al pipeline, payload con fingerprint estable, `AuthorizationManifestStore` persistente en formato `include-safe PHP array export` (estilo `ArtifactStore`), `ManifestRequirementsEnforcementStage` como enforcement temprano y comandos `authz:manifest:compile` + `authz:manifest:clear` **con 12 tests unitarios de coverage completo**; stage con modo `evaluateRequirementsConcretely=true` (opt-in) aplica RBAC/ABAC sobre authority repository sin invocar gates/policies completos | falta cache policy de invalidacion por scope y evaluacion ABAC runtime lazy |
| 19 | Authorization Extensibility Plugin Provider And Custom Evaluator System | Parcial | block `authorization.authority.*` configuracion con 3 keys (`enabled`, `evaluate_requirements_concretely`, `grants`); `AuthorizationServiceProvider::registerAuthorityRepository()` singleton condicional; Manifest stage wiring acepta authorityRepository=null y evaluateRequirementsConcretely=false → backward compat; `AuthorityRepositoryInterface` extensible para custom providers; **nuevo:** `ControllerSecurityPlannerBridge` extensible para adaptadores planner ↔ Security | falta plugin loader dinamico y provider de authority externo (DB, LDAP, OPA) |
| 20 | Authorization Delegation Impersonation Capabilities And Service To Service System | Parcial | subject plano en `AuthorizationManager::check()` no requiere `RouteMatch`; multi-surface tests pasan con gate definitions sin HTTP context; `Principal` arbitrario puede construirse `new Principal('worker-1')` y usarse directamente en jobs/cli; bridge Security ↔ planner acepta Security principals arbitrarios y los normaliza | falta formalizar `impersonate()`, service-to-service Principal resolver y delegation grants scope-bound |
| 23 | Authorization Conditional Contextual And Risk-Based Access System | Parcial | **nuevo:** `AttributeDefinition` typed (bool/string/int/float/array/enum/any) con constraints pattern/min-max/enum/required/default; `AuthorizationContext` attributes ya se usa por Manifest stage cuando evaluateRequirementsConcretely=true | falta evaluador condicional runtime ABAC sobre attributes, risk scoring y adaptive access |
| 25 | Authorization Configuration Bootstrap And Service Container Integration System | Operativo | existen `AuthorizationServiceProvider`, defaults de config, `config/authorization.php`, facade y helpers; `Application.php` ya registra el provider base; provider ahora expone `commands()` para discovery automatico de los comandos de manifest; **nuevo:** defaults block `authorization.authority.*` con `enabled=true`, `evaluate_requirements_concretely=false` (opt-in), `grants=[]` | falta configuracion avanzada, discovery declarativo de policies y bootstrap por manifests |
| 27 | Authorization State Consistency Concurrency And Distributed Coordination System | Parcial | **nuevo:** InMemoryAuthorityRepository uses `$visited` array para break loops jerárquicos scope; scope() → parent() no produce infinite loops; todo stateless singleton o scoped-request | falta distributed coordination lock, cache invalidation multi-node y consistency audit |
| 29 | Authorization Administration Management And Operational Tooling System | Operativo | comandos CLI `authz:manifest:compile` y `authz:manifest:clear` con categorias `Authorization`, aliases alternativos y banderas `--verbose` / `--dry-run`, registrados automaticamente via `AuthorizationServiceProvider::commands()`; **nuevo:** 12 tests unitarios commands (metadata, empty routes, persist fingerprint, dry-run, verbose, skip throw/no-fp, clear entries, empty clear, clear verbose, clear exception → exit 1) | faltan comandos de auditoria y revocacion administrativas |
| 31 | Authorization Performance Compilation Optimization And Resource Governance System | Parcial | evaluateRequirementsConcretely=true short-circuits gates/policies cuando hay ALLOW o explicit DENY via authority; seed de grants en PHP-array include-safe (no JSON) para rendimiento; commands CLI 12 tests + coverage; planner ya opera sin overhead | falta cache memoization effectivePermissions y resource governance budget |

## Lectura ejecutiva del corte

### Lo que si existe hoy

1. un core minimo real en `Quantum/Authorization`,
2. gates, abilities, request model y decision model funcionales,
3. bootstrap, config, facade, helpers, planner y mapper de errores del modulo,
4. integración con Authentication para resolver principal/contexto,
5. metadata declarativa inicial con atributos y DSL de rutas,
6. proyeccion reusable de Authorization sobre `Quantum/Metadata`,
7. integración inicial con `ControllerEngine`,
8. Modelos RBAC/ABAC (Role/Permission/Scope/AttributeDefinition) + AuthorityRepository con herencia scope jerárquica,
9. ManifestRequirementsEnforcementStage ampliado (opt-in) para evaluar requirements concretos vía authority con `effect=deny` gana siempre,
10. 5 tests multi-surface CLI/Jobs/Workers (sin HTTP RouteMatch) + 9 tests authority models + 4 tests stage concrete,
11. 12 tests commands CLI `authz:manifest:*` (metadata, persist, dry-run, verbose, skip, clear, exceptions),
12. Bridge mínimo `ControllerSecurityPlannerBridge` convergencia Controllers/Security ↔ Planner,
13. pruebas unitarias y feature ampliadas del subsistema (107 Unit + 74 Feature exit 0).

### Lo que no existe todavia

1. explainability `DecisionPlan::explain()` arbol por stage,
2. memoization cache effectivePermissions por (principalId,scope),
3. convergencia ABAC runtime evaluator y ReBAC relationships,
4. integración AuthorizationManager.check() early-gate via AuthorityRepository,
5. integración declarativa con controllers/routing para scope automático (tenancy resolver).

## Prioridades reales segun la matriz

### Prioridad alta

1. `02`, `03`, `04`, `06`, `07`, `08`, `10`, `11`, `17`, `25`, `29`
2. `05`, `09`, `12`, `16`, `18`, `32`

Motivo:

- subsistema ya se encuentra operativo en core/planner/metadata/tests,
- queda avanzar auditoria explainability y memoization para performance,
- y converger aún mas profundo con Controllers/Security para eliminar duplicados.

### Prioridad media

1. `13`, `14`, `19`, `23`, `27`, `31`
2. `20`

### Prioridad posterior

1. `15`, `21`, `22`, `24`, `26`, `28`, `30`

## Regla de mantenimiento

Cada vez que se cierre un bloque relevante del subsistema Authorization, esta matriz debe actualizar:

1. el estado del documento impactado,
2. la evidencia visible en codigo y tests,
3. el gap principal restante,
4. y la prioridad posterior.
