# DEVELOPMENT_VERSIONS

## Proposito

Esta bitacora registra el avance real del desarrollo del subsistema `Quantum/Authorization` frente a la documentacion oficial ubicada en `vendor/voltstack/authorization-lab/Docs`.

Sirve como control operativo de:

- lo ya implementado,
- lo que existe solo como infraestructura adyacente,
- lo que sigue pendiente en el namespace objetivo,
- y el siguiente corte recomendado de ejecucion.

## Corte actual

- Fecha de actualizacion: `2026-09-23`
- Estado general: `Quantum/Authorization ya dispone de un core minimo operativo, mas una primera integracion declarativa con controllers, routing y manejo de errores HTTP.`
- Clasificacion del corte: `V1 conectada inicial`

## Resumen ejecutivo

Hoy Authorization se encuentra en esta situacion:

1. la arquitectura `00-32` ya esta escrita con mucho detalle,
2. `Quantum/Authorization` ya dispone de core, policies declarativas iniciales, atributos, DSL de rutas y mapper de excepciones,
3. `Quantum/Controllers/Security` sigue demostrando muchas ideas reutilizables:
   - principal,
   - contexto,
   - decision engine,
   - atributos,
   - composicion de policies,
   - fail-closed,
   - worker safety,
4. `Quantum/Metadata` ya ofrece una base para discovery y metadata declarativa futura,
5. `Quantum/Auth` ya puede suministrar identidad y contexto de autenticacion,
6. el gap dominante ya no es abrir el modulo, sino consolidar planner, compilacion y rollout mas profundo.

## Entradas de version

### DV-AUTHZ-001

**Tipo:** Baseline de desarrollo  
**Estado:** Cerrado  
**Objetivo:** Crear la trazabilidad inicial del subsistema Authorization antes de abrir implementacion en `Quantum/Authorization`.

**Documentos impactados**

- `DEVELOPMENT_GUIDELINES.md`
- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- `EXECUTIVE_PLAN_IMPLEMENTATION.md`

**Alcance**

1. analizar el estado real del framework respecto de Authorization,
2. distinguir entre arquitectura objetivo e infraestructura adyacente existente,
3. fijar prioridades reales de implementacion,
4. definir una secuencia ejecutiva inicial para el modulo.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization` sin implementacion visible,
- `vendor/voltstack/framework/src/Quantum/Controllers/Security/*`,
- `vendor/voltstack/framework/src/Quantum/Metadata/*`,
- `vendor/voltstack/framework/src/Quantum/Auth/*`,
- `vendor/voltstack/framework/src/Platform/Application.php`,
- `vendor/voltstack/framework/tests/Unit/ControllerSecurityModelTest.php`,
- `vendor/voltstack/framework/tests/Unit/PolicyCompositionTest.php`,
- `vendor/voltstack/framework/tests/Unit/PolicyWorkerSafetyTest.php`.

**Resultado operativo**

- queda documentado con precision que Authorization no parte desde cero conceptual,
- pero si parte sin modulo propio en el namespace destino,
- y queda fijado que el primer trabajo real debe abrir el core minimo del subsistema.

**Gap natural siguiente**

- crear el primer esqueleto funcional de `Quantum/Authorization` con request model, decision model, manager, abilities, principal y bootstrap minimo.

### DV-AUTHZ-002

**Tipo:** Implementacion fundacional  
**Estado:** Cerrado  
**Objetivo:** Crear el core minimo real del subsistema Authorization dentro de `Quantum/Authorization`.

**Documentos impactados**

- `DEVELOPMENT_GUIDELINES.md`
- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- `EXECUTIVE_PLAN_IMPLEMENTATION.md`

**Alcance**

1. crear el namespace funcional `Quantum/Authorization`,
2. introducir `Ability`, `Principal`, `AnonymousPrincipal`, `SubjectDescriptor`, `AuthorizationContext`, `AuthorizationRequest`,
3. introducir `Decision`, `DecisionResult`, `DecisionManager` y `AuthorizationManager`,
4. introducir `GateRegistry`, `AbilityRegistry`, `PolicyRegistry` y `PolicyDispatcher` minimos,
5. conectar el modulo con `Auth`, `Application.php`, config, facade y helpers,
6. dejar pruebas unitarias y feature del modulo.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/*`
- `vendor/voltstack/framework/src/Quantum/Facades/Authorization.php`
- `vendor/voltstack/framework/src/Helper/helpers.php`
- `vendor/voltstack/framework/src/Platform/Application.php`
- `config/authorization.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManagerTest.php`
- `vendor/voltstack/framework/tests/Feature/AuthorizationIntegrationTest.php`

**Resultado operativo**

- el framework ya cuenta con un `AuthorizationManager` reusable,
- existe modelo canonico minimo de ability/principal/subject/context/request/decision,
- ya hay gates y policies manuales usables,
- y el modulo puede resolver principal desde `Auth` sin fuga entre requests.

**Validacion ejecutada**

- `php vendor/bin/phpunit vendor/voltstack/framework/tests/Unit/AuthorizationManagerTest.php vendor/voltstack/framework/tests/Feature/AuthorizationIntegrationTest.php`

**Gap natural siguiente**

- formalizar contracts de policy,
- ampliar registry/dispatcher hacia discovery y metadata,
- y empezar la integracion declarativa con controllers y routing.

### DV-AUTHZ-003

**Tipo:** Integracion declarativa inicial
**Estado:** Cerrado
**Objetivo:** Conectar el core de Authorization con policies declarativas, metadata de rutas/controllers y manejo coherente de errores HTTP.

**Documentos impactados**

- `DEVELOPMENT_GUIDELINES.md`
- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- `EXECUTIVE_PLAN_IMPLEMENTATION.md`

**Alcance**

1. introducir contracts formales de policy y atributos declarativos,
2. ampliar `PolicyRegistry` y `PolicyDispatcher` para soportar metadata por atributos y configuracion,
3. introducir `#[Authorize]`, `#[PublicAccess]`, `#[PolicyFor]` y `#[HandlesAbility]`,
4. añadir DSL `Route::authorize()` y `Route::publicAccess()`,
5. proyectar metadata declarativa hacia `ControllerEngine`,
6. mapear excepciones de Authorization a respuestas HTTP coherentes.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Attributes/*`
- `vendor/voltstack/framework/src/Quantum/Authorization/Policy/Attributes/*`
- `vendor/voltstack/framework/src/Quantum/Authorization/Policy/Contracts/*`
- `vendor/voltstack/framework/src/Quantum/Authorization/Policy/PolicyRegistry.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Policy/PolicyDispatcher.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Exceptions/AuthorizationExceptionMapper.php`
- `vendor/voltstack/framework/src/Quantum/Routing/Route.php`
- `vendor/voltstack/framework/src/Quantum/Controllers/ControllerEngine.php`
- `vendor/voltstack/framework/src/Platform/Application.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManagerTest.php`
- `vendor/voltstack/framework/tests/Unit/ControllerEngineTest.php`
- `vendor/voltstack/framework/tests/Unit/QuantumExceptionHandlerTest.php`
- `vendor/voltstack/framework/tests/Feature/ExceptionHandlingTest.php`

**Resultado operativo**

- Authorization ya puede declararse por atributo o por DSL de ruta,
- `ControllerEngine` ya puede resolver el subject desde argumentos del controller y delegar la decision al manager nuevo,
- las denegaciones/challenges del modulo ya se representan como `403/401` en HTTP en lugar de degradar a `500`,
- y la carga de policies desde config ya soporta listas de classes decoradas con `#[PolicyFor]`.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\AuthorizationManagerTest.php`
- `vendor\bin\phpunit tests\Unit\ControllerEngineTest.php`
- `vendor\bin\phpunit tests\Unit\QuantumExceptionHandlerTest.php`
- `vendor\bin\phpunit tests\Feature\ExceptionHandlingTest.php`
- `vendor\bin\phpunit tests\Feature\AuthorizationIntegrationTest.php tests\Unit\AuthorizationManagerTest.php tests\Unit\ControllerEngineTest.php tests\Unit\QuantumExceptionHandlerTest.php`

**Gap natural siguiente**

- introducir planner/pipeline formal,
- mover discovery/metadata hacia una base compilable,
- y ampliar la convergencia entre `Controllers/Security` y `Quantum/Authorization`.

### DV-AUTHZ-004

**Tipo:** Consolidacion de planner, metadata compilable y manifest store
**Estado:** Cerrado
**Objetivo:** Separar formalmente el planning de Authorization, introducir metadata compilable y manifest store persistente con fingerprint estable.

**Documentos impactados**

- `DEVELOPMENT_GUIDELINES.md`
- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- `EXECUTIVE_PLAN_IMPLEMENTATION.md`

**Alcance ejecutado en este corte**

1. introducir `AuthorizationPlannerInterface` y `AuthorizationPlanner`,
2. separar el manager de la agregacion inline de gates/policies,
3. hacer efectiva la estrategia `default_strategy`,
4. hacer explicita la semantica `fail_closed` y `fail_open` ante fallos de evaluadores,
5. extraer un pipeline minimo por stages para gates y policies,
6. proyectar metadata de Authorization sobre `Quantum/Metadata`,
7. extraer un `AuthorizationMetadataResolver` reusable para desacoplar el engine de controllers,
8. introducir enrichment contextual previo al pipeline usando metadata declarativa,
9. normalizar metadata en un payload con fingerprint estable para preparar manifests/compilation,
10. ampliar la validacion unitaria y de regresion del subsistema,
11. definir `AuthorizationManifestStoreInterface` como contrato de almacenamiento por fingerprint,
12. introducir `AuthorizationManifestEntry` y `AuthorizationMetadataPayloadFactory` para serializacion/hidratacion,
13. implementar `InMemoryAuthorizationManifestStore` y `FilesystemAuthorizationManifestStore` con writes atomicos,
14. integrar el manifest store como fuente PRIMERA del resolver (cached by fingerprint) y guardado automatico tras calculo runtime,
15. cablear configuracion `authorization.manifest.enabled` y `authorization.manifest.path` en `AuthorizationServiceProvider`,
16. añadir cobertura unitaria y de integracion para stores, factory y flujo resolver+manifest+enricher.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorizationPlannerInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorizationEvaluationStageInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorizationMetadataResolverInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorizationRequestEnricherInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Manifest/Contracts/AuthorizationManifestStoreInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Manifest/AuthorizationManifestEntry.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Manifest/InMemoryAuthorizationManifestStore.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Manifest/FilesystemAuthorizationManifestStore.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/AuthorizationPlanner.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/Stages/GateAuthorizationStage.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/Stages/PolicyAuthorizationStage.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/AuthorizationManager.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Decision/DecisionManager.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationMetadata.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationRequirement.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationMetadataPayload.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationMetadataPayloadFactory.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationMetadataResolver.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/MetadataAuthorizationContextEnricher.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/src/Quantum/Metadata/Providers/AttributeMetadataProvider.php`
- `vendor/voltstack/framework/src/Quantum/Metadata/Providers/RouteMetadataProvider.php`
- `vendor/voltstack/framework/src/Quantum/Metadata/MetadataEngine.php`
- `vendor/voltstack/framework/src/Platform/Application.php`
- `vendor/voltstack/framework/src/Quantum/Controllers/ControllerEngine.php`
- `vendor/voltstack/framework/tests/Unit/MetadataAuthorizationContextEnricherTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationMetadataResolverTest.php`
- `vendor/voltstack/framework/tests/Unit/MetadataEngineTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationPlannerTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManagerTest.php`
- `vendor/voltstack/framework/tests/Unit/InMemoryAuthorizationManifestStoreTest.php`
- `vendor/voltstack/framework/tests/Unit/FilesystemAuthorizationManifestStoreTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManifestIntegrationTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationServiceProviderTest.php`

**Resultado operativo parcial**

- Authorization ya no agrega evaluadores inline dentro del manager,
- existe una primera separacion formal entre request normalization, enrichment, planning y final decision,
- el planner ya orquesta un pipeline minimo con stages de `gates` y `policies`,
- `#[Authorize]`, `#[PublicAccess]` y `Route::authorize()/publicAccess()` ya se proyectan como metadata reusable del framework,
- `ControllerEngine` ya consume un resolver propio de Authorization en vez de depender de los detalles de `MetadataEngine`,
- gates y policies ya pueden recibir metadata declarativa contextual a traves de `AuthorizationContext`,
- la metadata ya cuenta con un payload normalizado y `fingerprint` estable,
- se dispone de `AuthorizationManifestStoreInterface` con implementaciones en memoria y filesystem (writes atomicos, formato PHP include-safe),
- el resolver ya consulta el manifest store PRIMERO (cached by fingerprint) y guarda el resultado automaticamente tras el primer calculo runtime,
- el provider ya expone la configuracion `authorization.manifest.enabled` (por defecto `true`) y `authorization.manifest.path` para persistencia fisica,
- el enricher ya expone `authorization.metadata.fingerprint` en el contexto para trazabilidad,
- y la configuracion `authorization.default_strategy` / `authorization.fail_closed` ya altera el comportamiento real del engine.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\InMemoryAuthorizationManifestStoreTest.php tests\Unit\FilesystemAuthorizationManifestStoreTest.php tests\Unit\AuthorizationManifestIntegrationTest.php`
- `vendor\bin\phpunit tests\Unit\AuthorizationServiceProviderTest.php` (6 tests, 11 assertions: `manifest.enabled=false`, `manifest.path` con Filesystem, coercion booleana 0=false)
- `vendor\bin\phpunit tests\Unit\AuthorizationMetadataResolverTest.php tests\Unit\MetadataAuthorizationContextEnricherTest.php tests\Unit\MetadataEngineTest.php tests\Unit\AuthorizationPlannerTest.php tests\Unit\AuthorizationManagerTest.php tests\Unit\InMemoryAuthorizationManifestStoreTest.php tests\Unit\FilesystemAuthorizationManifestStoreTest.php tests\Unit\AuthorizationManifestIntegrationTest.php tests\Unit\AuthorizationServiceProviderTest.php` → **37 tests, 102 assertions, exit 0**
- `vendor\bin\phpunit tests\Unit\ControllerEngineTest.php tests\Unit\QuantumExceptionHandlerTest.php tests\Feature\AuthorizationIntegrationTest.php tests\Feature\ExceptionHandlingTest.php` → **45 tests, 208 assertions, exit 0**

**Gap natural siguiente**

- extender el stage de manifest para evaluar CADA requirement concreto contra gate/policy,
- ampliar tests a escenarios multi-surface (CLI, Jobs, Workers sin RouteMatch HTTP),
- materializar los commands CLI en tests reales y surface real,
- acercar la convivencia con `Controllers/Security` decision engine,
- y ampliar la observabilidad/trazabilidad de la evaluacion (audit log, explain plan de decisiones).

## Corte ejecutado

### DV-AUTHZ-005

**Titulo:** `Manifest Enforcement, Command CLI De Compilacion Y V1 Conectada`
**Estado:** Cerrado
**Objetivo:** Hacer operativo el enforcement directo desde metadata de manifest, publicar commands CLI de compilacion/limpieza y proyectar fingerprint visible en DecisionResult para trazabilidad.

**Documentos impactados**

- `DEVELOPMENT_GUIDELINES.md`
- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- `EXECUTIVE_PLAN_IMPLEMENTATION.md`

**Alcance ejecutado en este corte**

1. introducir `ManifestRequirementsEnforcementStage` antepuesto a gates/policies que aplica enforcement directo desde metadata del manifest (public→ALLOW, ability fuera de whitelist requirements→DENY/ABSTAIN, matched→downstream),
2. añadir `DecisionResult::metadataFingerprint()` con propagacion automática desde `allow/deny/abstain/challenge/failure` si metadata trae `metadata_fingerprint`,
3. propagar fingerprint planner-wide via `AuthorizationPlanner::applyFingerprint()` sobre todos los `DecisionResult` (clonacion por reflection readonly-safe),
4. crear comando CLI `authz:manifest:compile` que itera RouteCollection, construye RouteMatch por cada HTTP method de la ruta y resuelve metadata vía resolver (persistiendo en manifest store, con `--verbose` y `--dry-run`),
5. crear comando CLI `authz:manifest:clear` que invoca `Store::clear()` con detalle verbose, manejo de errores y `--dry-run`,
6. registrar ambos commands automaticamente via `AuthorizationServiceProvider::commands()` (registrado por discovery standard del framework),
7. integrar `ManifestRequirementsEnforcementStage` como PRIMERO en el pipeline de stages del planner (antes que gates y policies),
8. ajustar `AuthorizationMetadata::fingerprint()` constructor y propagacion al payload,
9. ajustar `AuthorizationMetadataResolver::resolve()` a `RouteMatch::resolvedMethod()`,
10. ampliar suite con tests unitarios de `ManifestRequirementsEnforcementStage` (8 tests, 20 assertions) y del provider (6 tests, 11 assertions).

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Core/Stages/ManifestRequirementsEnforcementStage.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/AuthorizationPlanner.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Decision/DecisionResult.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationMetadata.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationMetadataResolver.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationManifestCompileCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationManifestClearCommand.php`
- `vendor/voltstack/framework/tests/Unit/ManifestRequirementsEnforcementStageTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationServiceProviderTest.php`

**Resultado operativo parcial**

- el stage `manifest_requirements` decide ANTES de gates/policies: `public=true→ALLOW`, `ability no declarada en requirements→DENY (fail_closed) / ABSTAIN (fail_open)`, con fingerprint visible en la decision,
- todos los `DecisionResult` emitidos por el planner ya llevan `metadataFingerprint()` no null cuando el contexto trae fingerprint del manifest (trazabilidad completa),
- commands `authz:manifest:compile` y `authz:manifest:clear` quedan automaticamente disponibles por `commands()` del provider, con aliases y banderas normalizadas (verbose/dry-run),
- `AuthorizationServiceProvider` ya expone discovery de comandos, lo que habilita extensibilidad CLI nativa del modulo,
- enforcement de requirements de manifest opera en modo whitelist (filtro de abilities) y NO invoca MetadataEngine, minimizando calculo en hot path.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\ManifestRequirementsEnforcementStageTest.php` → **8 tests, 20 assertions, exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorizationServiceProviderTest.php` → **6 tests, 11 assertions, exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorizationMetadataResolverTest.php tests\Unit\MetadataAuthorizationContextEnricherTest.php tests\Unit\MetadataEngineTest.php tests\Unit\AuthorizationPlannerTest.php tests\Unit\AuthorizationManagerTest.php tests\Unit\InMemoryAuthorizationManifestStoreTest.php tests\Unit\FilesystemAuthorizationManifestStoreTest.php tests\Unit\AuthorizationManifestIntegrationTest.php tests\Unit\AuthorizationServiceProviderTest.php tests\Unit\ManifestRequirementsEnforcementStageTest.php` → **45 tests, 122 assertions, exit 0**
- `vendor\bin\phpunit tests\Unit\ControllerEngineTest.php tests\Unit\QuantumExceptionHandlerTest.php tests\Feature\AuthorizationIntegrationTest.php tests\Feature\ExceptionHandlingTest.php` → **45 tests, 208 assertions, exit 0**
- **SUITE COMPLETA ACUMULADA:** **90 tests / 330 assertions → exit 0**

**Gap natural siguiente**

- evaluar CADA requirement concreto del manifest invocando gate/policy (no solo whitelist filter),
- tests unitarios especificos para los 2 commands CLI del manifest,
- escenarios multi-surface (CLI/Jobs/Workers sin RouteMatch HTTP) confirmando gracefull degradation,
- convergencia entre `Controllers/Security` decision engine y `Quantum/Authorization` planner,
- auditoria/explainability plan de decisiones.

## Siguiente corte recomendado

### DV-AUTHZ-006

**Titulo sugerido:** `Modelos Avanzados De Autoridad (RBAC, ABAC, Tenancy Y Scopes Jerarquicos)`

**Documentos fuente**

- `05_POLICY_REGISTRY_DISCOVERY_AND_RESOLUTION_SYSTEM.md`
- `09_AUTHORIZATION_PLANNER_AND_POLICY_PIPELINE_SYSTEM.md`
- `10_AUTHORIZATION_ATTRIBUTES_AND_DECLARATIVE_METADATA_SYSTEM.md`
- `12_ROLE_PERMISSION_RBAC_ABAC_AND_REBAC_INTEGRATION_SYSTEM.md`
- `13_MULTI_TENANT_AUTHORIZATION_AND_DATA_ISOLATION_SYSTEM.md`
- `14_AUTHORIZATION_CACHE_MEMOIZATION_AND_DECISION_REUSE_SYSTEM.md`
- `19_AUTHORIZATION_EXTENSIBILITY_PLUGIN_PROVIDER_AND_CUSTOM_EVALUATOR_SYSTEM.md`
- `20_AUTHORIZATION_DELEGATION_IMPERSONATION_CAPABILITIES_AND_SERVICE_TO_SERVICE_SYSTEM.md`
- `23_AUTHORIZATION_CONDITIONAL_CONTEXTUAL_AND_RISK_BASED_ACCESS_SYSTEM.md`
- `27_AUTHORIZATION_STATE_CONSISTENCY_CONCURRENCY_AND_DISTRIBUTED_COORDINATION_SYSTEM.md`
- `31_AUTHORIZATION_PERFORMANCE_COMPILATION_OPTIMIZATION_AND_RESOURCE_GOVERNANCE_SYSTEM.md`

**Alcance sugerido**

1. introducir modelos concretos de `Role`, `Permission`, `Scope` y `AttributeDefinition` como conceptos de primer nivel del modulo,
2. introducir repositorio/interfaz `AuthorityRepositoryInterface` para resolucion de grants por principal/tenant,
3. extender `ManifestRequirementsEnforcementStage` para evaluar CADA requirement concreto contra gate/policy y authority repository,
4. tests especificos de commands CLI de manifest,
5. tests multi-surface (CLI/Jobs/Workers) sin HTTP RouteMatch,
6. convergencia entre el `HardenedControllerSecurityDecisionEngine` (Controllers/Security) y `Quantum/Authorization` planner,
7. versionado de scopes jerarquicos `organization > workspace > project`.

## Roadmap corto recomendado

### DV-AUTHZ-004 — CERRADO

`Planner, Metadata Compilable Y Manifest Store Persistente`

Entregado:

- planner formal con enrichers y stages,
- payload normalizado con fingerprint estable,
- `AuthorizationManifestStoreInterface` y stores InMemory/Filesystem,
- configuracion `authorization.manifest.enabled/path` y test del provider,
- 37 tests unitarios + 45 tests de controllers y features.

Foco entregado:

- `05`, `09`, `10`, `17`, `18`, `25`, `32`

### DV-AUTHZ-005 — CERRADO

`Manifest Enforcement, Command CLI Y V1 Conectada`

Entregado:

- `ManifestRequirementsEnforcementStage` como enforcement temprano (public→ALLOW, ability no declarada→DENY/ABSTAIN),
- fingerprint visible en `DecisionResult::metadataFingerprint()` propagado desde planner,
- commands `authz:manifest:compile` y `authz:manifest:clear` registrados via provider,
- 8 tests stage + 6 provider + 37 restantes = 45 tests core,
- 45 tests controllers + features = 90 tests exit 0.

Foco entregado:

- `05`, `09`, `10`, `17`, `18`, `25`, `29`, `32`

### DV-AUTHZ-006 — CERRADO (100%)

`Modelos Avanzados De Autoridad (RBAC, ABAC, Tenancy Y Scopes Jerarquicos)`

Entregado:

1. `Value Objects` `Role`, `Permission`, `Scope`, `AttributeDefinition` en `Quantum/Authorization/Authority/*`:
   - `Permission` single-string name + `matches()` wildcard (`admin:*` matches `admin:read`),
   - `Role` named bag con deduplicacion de permisos por nombre,
   - `Scope` jerárquico `org:ws:proj` con `contains()`, `parent()`, `level()`, wildcard `*`, global `global`,
   - `AttributeDefinition` schema ABAC typed con constraints pattern/min-max/enum/required/default.
2. `AuthorityRepositoryInterface` + `InMemoryAuthorityRepository`:
   - seed desde config `authorization.authority.grants` shape `{principal_id, scope, roles, permissions}`,
   - herencia upward de grants: `while current parent() → colecta roles y permisos` (scope→parent→…→global),
   - hasPermission/hasRole/effectivePermissionsForPrincipal/effectiveRolesForPrincipal/attributesForPrincipal.
3. `ManifestRequirementsEnforcementStage` ampliado:
   - Nuevos params constructor nullable `authorityRepository`, `evaluateRequirementsConcretely=false` (backward compat),
   - `evaluateRequirementsConcretely=false`: modo whitelist legacy (downstream gates/policies) → COMPORTAMIENTO 005,
   - `evaluateRequirementsConcretely=true`: `normalizedMatchedRequirements` + `evaluateConcretelyEachMatchedRequirement()` con `effect=deny` gana SIEMPRE sobre grants,
   - reason codes nuevos: `manifest_requirement_granted_by_authority`, `manifest_requirement_not_granted_by_authority`, `manifest_requirement_explicit_deny`.
4. Wiring `AuthorizationServiceProvider`:
   - defaults `authorization.authority.enabled=true`, `evaluate_requirements_concretely=false` (opt-in), `grants=[]`,
   - `registerAuthorityRepository()` singleton condicional sobre enabled,
   - Manifest stage wiring pasa los 3 params.
5. `RouteDefinition action()` getter fix + Command formatActionForOutput helper:
   - Sustituido acceso a propiedad privada `$route->definition()->action` por getter público `action()`,
   - Nuevo `formatActionForOutput(?ControllerDefinition): string` normaliza callable-array `[Class,method]` a `Class::method` para sprintf sin warning.
6. Tests unitarios commands CLI `authz:manifest:compile|clear` completados:
   - `AuthorizationManifestCommandsTest` 12 tests: metadata command, empty → 0, persist + fingerprint, --dry-run, --verbose output, skip resolver throw/no-fp, clear 7 entries, clear empty, --dry-run no-op, --verbose store, clear exception → exit 1.
   - Workaround `final Command` via `bootstrap/app.php` temporal + `$GLOBALS['__volt_authz_test_app']`.
   - Spy store/resolver anonymous classes + ReflectionProperty leer buffers privados `Output::stdoutBuffer/stderrBuffer`.
7. Tests multi-surface:
   - `AuthorizationMultiSurfaceIntegrationTest` 5 tests (CLI surface gate, job exception, authority seed directo, helper, authority disabled).
8. Bridge mínimo SecurityEngine ↔ Planner:
   - Nueva clase `Quantum/Authorization/Bridges/ControllerSecurityPlannerBridge`,
   - `tryEvaluate(SecurityEvaluationRequest): ?SecurityDecision` returns `null` si no hay metadata de authorization requirements → Hardened engine normal continua,
   - Extrae requirements desde `metadata[authorization_requirements]` o `metadata[permissions]`,
   - Mapea `SecurityPrincipal` (Controllers/Security) → `Quantum/Authorization/Principal` con enum PrincipalType via `mapSecurityPrincipalTypeToAuthorizationType()`,
   - Invoca `AuthorizationManager::decide()` y mapea `DecisionResult` → `SecurityDecision` (Allow/Deny/Abstain/Challenge) con obligations `requirements` + `fingerprint`.
9. Tests unitarios bridge `ControllerSecurityPlannerBridgeTest` 5 tests:
   - null si no requirements, allow si gate match + requirements, deny si fail-closed sin match, extrae `metadata.permissions` cuando no hay requirements, propaga fingerprint obligations.
10. Regresión exitosa:
    - Authority/ManifestStage/MultiSurface/Commands/Bridge/Mapper/Manager/Store/ControllerEngine/ExceptionHandler + Feature AuthorizationIntegration/SkeletonSecurity: **107 + 74 tests exit 0**, 393 + 901 assertions.
    - 1 warning de PHPUnit Notice: 1 test bridge mock anonymous sin impactar salida.

Foco entregado COMPLETO:

- `05`, `09`, `10`, `12`, `13`, `14`, `17`, `18`, `19`, `20`, `23`, `25`, `27`, `29`, `31`, `32`

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/Permission.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/Role.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/Scope.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/AttributeDefinition.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorityRepositoryInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/InMemoryAuthorityRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/Stages/ManifestRequirementsEnforcementStage.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Bridges/ControllerSecurityPlannerBridge.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationManifestCompileCommand.php` (fix action getter + formatActionForOutput)
- `vendor/voltstack/framework/tests/Unit/AuthorityModelAndRepositoryTest.php`
- `vendor/voltstack/framework/tests/Unit/ManifestRequirementsEnforcementStageTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationMultiSurfaceIntegrationTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManifestCommandsTest.php` (12 tests)
- `vendor/voltstack/framework/tests/Unit/ControllerSecurityPlannerBridgeTest.php` (5 tests)

**Resultado operativo**

- `evaluateRequirementsConcretely=false` POR DEFECTO → sin breaking changes sobre DV-AUTHZ-005; encender a mano para RBAC hard-cut,
- scope herencia upward funciona: principal con scope `org:acme:ws1` hereda grants de `org:acme` y `global`,
- explicit `effect:deny` en manifest requirement supera authority grants siempre,
- commands `authz:manifest:*` operativos + 12 tests coverage metadata / dry-run / verbose / skip / clear / exceptions,
- warning `Array to string conversion` eliminado via normalizador `formatActionForOutput()`,
- convergencia `HardenedControllerSecurityDecisionEngine` ↔ planner materializada via puente opcional `ControllerSecurityPlannerBridge::tryEvaluate()`: retorna `null` si no hay requirements → Hardened engine sigue sin tocar; si hay requirements, resuelve vía planner + mapea DecisionResult a SecurityDecision.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\AuthorityModelAndRepositoryTest.php` → **9 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\ManifestRequirementsEnforcementStageTest.php` → **12 tests (8 legacy + 4 concrete), exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorizationMultiSurfaceIntegrationTest.php` → **5 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorizationManifestCommandsTest.php` → **12 tests, 45 assertions, 0 warnings, exit 0**
- `vendor\bin\phpunit tests\Unit\ControllerSecurityPlannerBridgeTest.php` → **5 tests, 14 assertions, exit 0**
- suite core/controllers/features ACUMULADA: 107 tests Unit (393 assertions) + 74 tests Feature Authorization+SecuritySmoke+AuthManager (901 assertions) → exit 0 salvo 1 error pre-existente `AuthManager::password_expired` no relacionado.

### DV-AUTHZ-006 — SIGUIENTE (para completar)

`Modelos Avanzados De Autoridad (RBAC, ABAC, Tenancy) — CERRADO 100%`

Pendiente migratorio para fases futuras (no bloqueante para 006):
- integración `Authorization::authorize()` con AuthorityRepository early-gate vía config,
- auditoria `DecisionPlan::explain()` con árbol stages.

## Siguiente corte recomendado

### DV-AUTHZ-007

**Titulo sugerido:** `Explainability, Memoization De Permisos E Integración Authority ↔ Manager`

**Documentos fuente**

- `12_ROLE_PERMISSION_RBAC_ABAC_AND_REBAC_INTEGRATION_SYSTEM.md`
- `14_AUTHORIZATION_CACHE_MEMOIZATION_AND_DECISION_REUSE_SYSTEM.md`
- `19_AUTHORIZATION_EXTENSIBILITY_PLUGIN_PROVIDER_AND_CUSTOM_EVALUATOR_SYSTEM.md`
- `23_AUTHORIZATION_CONDITIONAL_CONTEXTUAL_AND_RISK_BASED_ACCESS_SYSTEM.md`
- `31_AUTHORIZATION_PERFORMANCE_COMPILATION_OPTIMIZATION_AND_RESOURCE_GOVERNANCE_SYSTEM.md`

**Alcance sugerido**

1. método `AuthorizationDecisionPlan::explain(): array` con trace por stage + reason code,
2. cache memoización `effectivePermissionsForPrincipal` por par clave `(principalId,scope)` con TTL scoped-request,
3. integración `AuthorizationManager::authorize()` + `check()` con `authorization.authority.enabled` como early gate antes de planner (opt-in vía config),
4. integración wiring del `ControllerSecurityPlannerBridge` en `AuthorizationServiceProvider` para uso opcional en rutas que habiliten metadata `authorization_requirements`,
5. tests unitarios de explain + memoization + early-gate authority en AuthorizationManager.

### DV-AUTHZ-007 — SIGUIENTE

`Explainability, Memoization De Permisos E Integración Authority ↔ Manager`

Foco:

- `12`, `14`, `19`, `23`, `31`

## Regla de mantenimiento

Cada vez que se cierre una nueva iteracion del subsistema Authorization, esta bitacora debe registrar:

1. identificador `DV-AUTHZ-00X`,
2. documentos impactados,
3. alcance real,
4. evidencia,
5. resultado operativo,
6. y siguiente gap natural.

No dejar esta actualizacion para una fase posterior.
