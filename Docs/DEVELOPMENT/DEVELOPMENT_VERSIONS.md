# DEVELOPMENT_VERSIONS

## Proposito

Esta bitacora registra el avance real del desarrollo del subsistema `Quantum/Authorization` frente a la documentacion oficial ubicada en `vendor/voltstack/authorization-lab/Docs`.

Sirve como control operativo de:

- lo ya implementado,
- lo que existe solo como infraestructura adyacente,
- lo que sigue pendiente en el namespace objetivo,
- y el siguiente corte recomendado de ejecucion.

## Corte actual

- Fecha de actualizacion: `2026-10-10`
- Estado general: `Quantum/Authorization ya dispone de un core operativo consolidado, planner explainable, authority repositories InMemory y Database + RemoteCache, memoization request-scoped con versionado generacional distribuido, early-gate opt-in, ABAC runtime declarativo, tenant/scope automático opt-in ya conectado a metadata/routing/controllers/command/job runtime, una capa ReBAC opt-in integrada al pipeline declarativo mediante metadata relation con drivers memory, database y remote-cache, backend persistente compartido de consistencia para authority/relationships via filesystem y cache lock-safe con auditoria de bumps distribuidos, tooling CLI operativo para listar/otorgar/revocar grants de authority/relationships/delegation sobre drivers memory y database, una capa inicial de adaptive access, y una capa completa de Delegation/Impersonation + Service-to-Service Principals opt-in con evaluación semántica real en el manifest stage.`
- Clasificacion del corte: `V1+ multi-tenant relacional consistente operativo con adaptive access inicial + delegation impersonation y s2s principals`

## Resumen ejecutivo

Hoy Authorization se encuentra en esta situacion:

1. la arquitectura `00-32` ya esta escrita con mucho detalle,
2. `Quantum/Authorization` ya dispone de core, planner explainable, policies declarativas, metadata compilable, authority repositories, atributos/DSL de rutas y mapper de excepciones,
3. `Quantum/Controllers/Security` sigue demostrando muchas ideas reutilizables:
   - principal,
   - contexto,
   - decision engine,
   - atributos,
   - composicion de policies,
   - fail-closed,
   - worker safety,
4. `Quantum/Metadata` ya soporta la proyeccion declarativa de condiciones y fingerprints del modulo,
5. `Quantum/Auth` ya puede suministrar identidad y contexto de autenticacion,
6. el gap dominante ya no es abrir el modulo, sino llevar la invalidacion/versionado hacia un backend multi-worker real, providers externos y performance distribuida.

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

### DV-AUTHZ-007

**Tipo:** Consolidacion V1+
**Estado:** Cerrado (100%)
**Objetivo:** Aterrizar explainability del planner, memoization request-scoped e integración `AuthorityRepository` ↔ `AuthorizationManager` sin romper compatibilidad.

**Documentos impactados**

- `DEVELOPMENT_GUIDELINES.md`
- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- `EXECUTIVE_PLAN_IMPLEMENTATION.md`

**Alcance ejecutado en este corte**

1. introducir `AuthorizationDecisionPlan` como VO explainable con `explain(): array` y árbol por stages/resultados,
2. ampliar `AuthorizationPlanner` con `planAsDecisionPlan()` y timestamp/fingerprint reutilizable,
3. añadir shortcuts `AuthorizationManager::explain()` y `AuthorizationManager::explainPlan()`,
4. introducir `AuthorityMemoizationCacheInterface`, `RequestScopedAuthorityMemoizationCache` y `CachedAuthorityRepository`,
5. integrar early-gate opt-in en `AuthorizationManager::decide()` contra `AuthorityRepository` antes del planner completo,
6. cablear `AuthorizationServiceProvider` con defaults `memoize=true`, `early_gate_enabled=false` y binding opcional del `ControllerSecurityPlannerBridge`,
7. reforzar `DecisionResult::extractFingerprint()` para metadata explainable sin fingerprint explícito.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Decision/AuthorizationDecisionPlan.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorityMemoizationCacheInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/RequestScopedAuthorityMemoizationCache.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/CachedAuthorityRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/AuthorizationPlanner.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/AuthorizationManager.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Decision/DecisionResult.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationDecisionPlanExplanationTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorityMemoizationAndCacheTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManagerAuthorityEarlyGateTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationServiceProviderBridgeAndFlagsTest.php`

**Resultado operativo**

- el planner ya puede devolverse como plan explainable con estructura estable por stage/result,
- `AuthorizationManager` puede exponer explainability sin alterar las APIs públicas `check/cannot/decide/authorize`,
- `effectivePermissionsForPrincipal()` queda memoizado por request cuando `authorization.authority.memoize=true`,
- el hot path puede cortar por `AuthorityRepository` antes del planner completo cuando `authorization.authority.early_gate_enabled=true`,
- y el bridge `Controllers/Security` queda cableable vía provider bajo flag off-by-default.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\AuthorizationDecisionPlanExplanationTest.php` → **5 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorityMemoizationAndCacheTest.php` → **7 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorizationManagerAuthorityEarlyGateTest.php` → **7 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php` → **7 tests, exit 0**

### DV-AUTHZ-008

**Tipo:** Expansion declarativa y provider DBAL
**Estado:** Parcial avanzado
**Objetivo:** Abrir providers persistentes de authority y materializar ABAC runtime declarativo en metadata, rutas y controllers.

**Documentos impactados**

- `DEVELOPMENT_GUIDELINES.md`
- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- `EXECUTIVE_PLAN_IMPLEMENTATION.md`

**Alcance ejecutado en este corte**

1. introducir `DatabaseAuthorityRepository` con tablas `authorization_role_grants`, `authorization_permission_grants` y `authorization_role_permissions`,
2. extender `AuthorizationServiceProvider` con `authorization.authority.driver=memory|database|db|dbal`, `database.connection` y `database.tables.*`,
3. introducir `AttributeConditionEvaluator` para evaluar condiciones ABAC runtime sobre `AuthorizationContext` y `subject.*`,
4. ampliar `AttributeDefinition` para respetar `min` y `max` en `accepts()`,
5. habilitar enforcement ABAC runtime en `ManifestRequirementsEnforcementStage` bajo flag `authorization.authority.evaluate_attribute_conditions=false` por defecto,
6. propagar `condition` por `AuthorizationRequirement`, payload, resolver, enricher y manifest store,
7. introducir `#[AuthorizeWhen]`, `Route::authorizeWhen()`, `Route::authorizeWhenAll()` y DSL `Condition::*` para declarar condiciones reutilizables sin arrays crudos en runtime.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/DatabaseAuthorityRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/ABAC/AttributeConditionEvaluator.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/ABAC/Condition.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Attributes/AuthorizeWhen.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/AttributeDefinition.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/Stages/ManifestRequirementsEnforcementStage.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationRequirement.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationMetadataPayloadFactory.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationMetadataResolver.php`
- `vendor/voltstack/framework/src/Quantum/Metadata/Providers/AttributeMetadataProvider.php`
- `vendor/voltstack/framework/src/Quantum/Metadata/Providers/RouteMetadataProvider.php`
- `vendor/voltstack/framework/src/Quantum/Routing/Route.php`
- `vendor/voltstack/framework/tests/Unit/DatabaseAuthorityRepositoryTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationServiceProviderDatabaseAuthorityTest.php`
- `vendor/voltstack/framework/tests/Unit/AttributeConditionEvaluatorTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationMetadataResolverTest.php`
- `vendor/voltstack/framework/tests/Unit/MetadataAuthorizationContextEnricherTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManifestIntegrationTest.php`
- `vendor/voltstack/framework/tests/Unit/MetadataEngineTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManagerTest.php`

**Resultado operativo**

- Authorization ya puede resolver grants desde memoria o base de datos sin cambiar el contrato público,
- las condiciones ABAC declarativas viajan de atributo/ruta → metadata → manifest → stage de enforcement,
- `AuthorizeWhen` y `Condition::*` unifican la semántica declarativa entre controllers, rutas y adapters runtime,
- el manager ya puede tomar decisiones end-to-end usando requirements condicionados por contexto (`risk.score`, `department`, etc.),
- y todo sigue siendo opt-in por flags (`evaluate_attribute_conditions=false`, `early_gate_enabled=false`, bridge off).

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\DatabaseAuthorityRepositoryTest.php` → **2 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorizationServiceProviderDatabaseAuthorityTest.php` → **3 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\AttributeConditionEvaluatorTest.php` → **5 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorizationMetadataResolverTest.php` → **3 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\MetadataAuthorizationContextEnricherTest.php` → **2 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorizationManifestIntegrationTest.php` → **3 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\MetadataEngineTest.php` → **13 tests, exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorizationManagerTest.php` → **11 tests, exit 0**
- `vendor\bin\phpunit --filter=Authorization` → **82 tests / 256 assertions, exit 0**

**Gap natural siguiente**

- materializar ReBAC relacional sujeto↔recurso,
- resolver tenant/scope automático desde request y superficies no HTTP,
- invalidar memoization/caches de authority en escenarios multi-worker/multi-node,
- y endurecer providers externos y tooling operativo de auditoría/revocación.

## Corte ejecutado

### DV-AUTHZ-009

**Tipo:** Expansion multi-tenant relacional inicial
**Estado:** Parcial avanzado
**Objetivo:** Introducir una primera capa ReBAC opt-in sobre el pipeline declarativo y cerrar la propagacion tenant/scope automática iniciada en el corte anterior.

**Documentos fuente**

- `12_ROLE_PERMISSION_RBAC_ABAC_AND_REBAC_INTEGRATION_SYSTEM.md`
- `13_MULTI_TENANT_AUTHORIZATION_AND_DATA_ISOLATION_SYSTEM.md`
- `19_AUTHORIZATION_EXTENSIBILITY_PLUGIN_PROVIDER_AND_CUSTOM_EVALUATOR_SYSTEM.md`
- `20_AUTHORIZATION_DELEGATION_IMPERSONATION_CAPABILITIES_AND_SERVICE_TO_SERVICE_SYSTEM.md`
- `23_AUTHORIZATION_CONDITIONAL_CONTEXTUAL_AND_RISK_BASED_ACCESS_SYSTEM.md`
- `27_AUTHORIZATION_STATE_CONSISTENCY_CONCURRENCY_AND_DISTRIBUTED_COORDINATION_SYSTEM.md`
- `29_AUTHORIZATION_ADMINISTRATION_MANAGEMENT_AND_OPERATIONAL_TOOLING_SYSTEM.md`
- `31_AUTHORIZATION_PERFORMANCE_COMPILATION_OPTIMIZATION_AND_RESOURCE_GOVERNANCE_SYSTEM.md`

**Alcance ejecutado en este corte**

1. `TenantScopeResolverInterface` + `TenantScopeResolver` opt-in ya existen y normalizan `tenant.id`/`tenant_id` hacia `authorization.scope`,
2. `AuthorizationContextFactory` ahora puede proyectar scope automático al contexto cuando el resolver está habilitado,
3. `AuthorizationManager` y `ManifestRequirementsEnforcementStage` ya consumen el resolver para early-gate y enforcement concreto,
4. `ControllerEngine` ya proyecta `X-Tenant-Id` y tenant de runtime al contexto que usa `AuthorizationManager` para requirements de ruta/controller,
5. `TenantScopeResolver` ahora también entiende `Request`, `RouteMatch`, `controller.security.context` y parámetros `tenant|tenant_id|tenantId`, reduciendo el wiring manual en usos directos del manager/planner,
6. `AuthorizationServiceProvider` ajustó el ciclo de vida del inner authority repository a `scoped` para convivir correctamente con `DatabaseInterface` y memoization request-scoped,
7. introducir `RelationshipRepositoryInterface`, `InMemoryRelationshipRepository` y `RelationshipEvaluator` como base ReBAC opt-in,
8. propagar `relation` por `AuthorizationRequirement`, payload factory, resolver, metadata providers y manifest stage,
9. extender la ergonomia declarativa con `Route::authorizeRelated()` y `#[Authorize(..., relation: ...)]` manteniendo backward compatibility por argumento opcional,
10. cablear `AuthorizationServiceProvider` con `authorization.relationships.evaluate=false` y `authorization.relationships.entries=[]`,
11. endurecer `InMemoryRelationshipRepository::candidateScopes()` para cortar correctamente al alcanzar `global` y evitar loops de parent scope,
12. añadir cobertura nueva para metadata relacional, repositorio/evaluador de relaciones, stage runtime, provider flags y manager end-to-end.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/RelationshipRepositoryInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Relationship/InMemoryRelationshipRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Relationship/RelationshipEvaluator.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Attributes/Authorize.php`
- `vendor/voltstack/framework/src/Quantum/Routing/Route.php`
- `vendor/voltstack/framework/src/Quantum/Metadata/Providers/AttributeMetadataProvider.php`
- `vendor/voltstack/framework/src/Quantum/Metadata/Providers/RouteMetadataProvider.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationRequirement.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationMetadataPayloadFactory.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Metadata/AuthorizationMetadataResolver.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/Stages/ManifestRequirementsEnforcementStage.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/tests/Unit/InMemoryRelationshipRepositoryTest.php`
- `vendor/voltstack/framework/tests/Unit/RelationshipEvaluatorTest.php`
- `vendor/voltstack/framework/tests/Unit/ManifestRequirementsEnforcementStageTest.php`
- `vendor/voltstack/framework/tests/Unit/MetadataEngineTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationMetadataResolverTest.php`
- `vendor/voltstack/framework/tests/Unit/MetadataAuthorizationContextEnricherTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManifestIntegrationTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationServiceProviderBridgeAndFlagsTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManagerTest.php`

**Resultado operativo**

- Authorization ya puede evaluar relaciones sujeto↔recurso de forma opt-in antes del check RBAC final dentro de `ManifestRequirementsEnforcementStage`,
- la metadata declarativa soporta `relation` end-to-end en atributos, rutas, payloads, resolver y manifest cache,
- `Route::authorizeRelated()` y el cuarto argumento opcional de `Authorize`/`Route::authorize()` permiten expresar ReBAC sin romper la API existente,
- el provider ya expone flags y seeds basicos para relaciones in-memory,
- y la primera capa ReBAC queda alineada con tenancy/scope automático ya existente, conservando defaults off-by-default.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\MetadataEngineTest.php tests\Unit\AuthorizationMetadataResolverTest.php tests\Unit\MetadataAuthorizationContextEnricherTest.php tests\Unit\AuthorizationManifestIntegrationTest.php` → **25 tests, 52 assertions, exit 0**
- `vendor\bin\phpunit tests\Unit\RelationshipEvaluatorTest.php tests\Unit\ManifestRequirementsEnforcementStageTest.php tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php tests\Unit\AuthorizationManagerTest.php tests\Unit\InMemoryRelationshipRepositoryTest.php` → **43 tests, 127 assertions, exit 0 con 2 deprecations no bloqueantes**
- regresión focalizada del corte ReBAC: **68 tests / 179 assertions, exit 0**

**Gap natural siguiente**

- provider persistente inicial para relaciones y tooling operativo sobre grants/relations,
- tenancy automática uniforme en mas superficies no HTTP,
- invalidacion distribuida/multi-worker para authority cache y relaciones,
- providers externos adicionales,
- y politicas contextuales/risk-based de mayor nivel.

## Corte ejecutado

### DV-AUTHZ-010A

**Tipo:** Persistencia relacional inicial
**Estado:** Cerrado
**Objetivo:** Añadir un backend `database` para `RelationshipRepositoryInterface` sin romper la semántica ReBAC existente.

**Alcance ejecutado**

1. se añadió `DatabaseRelationshipRepository` como implementación persistente read-only para relaciones,
2. la resolución usa `resource_key` derivado con la misma semántica observable que `InMemoryRelationshipRepository`,
3. `AuthorizationServiceProvider` ahora soporta `authorization.relationships.driver=memory|database|db|dbal`,
4. se añadieron defaults `authorization.relationships.database.connection` y `authorization.relationships.database.table`,
5. el manager y el stage ReBAC pueden resolver relaciones persistentes sin cambiar contratos públicos.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Relationship/DatabaseRelationshipRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/tests/Unit/DatabaseRelationshipRepositoryTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationServiceProviderDatabaseRelationshipTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManagerDatabaseRelationshipTest.php`

**Resultado operativo**

- ReBAC ya no depende solo de seeds en memoria,
- el provider puede resolver relationships desde SQLite/DBAL manteniendo el contrato `RelationshipRepositoryInterface`,
- `AuthorizationManager` ya valida relaciones persistentes end-to-end sobre rutas declarativas,
- y el modulo avanza hacia DV-AUTHZ-010 sin forzar aun tooling operativo ni invalidación distribuida.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\DatabaseRelationshipRepositoryTest.php tests\Unit\AuthorizationServiceProviderDatabaseRelationshipTest.php tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php tests\Unit\AuthorizationManagerDatabaseRelationshipTest.php` → **14 tests / 40 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests\Unit\InMemoryRelationshipRepositoryTest.php tests\Unit\RelationshipEvaluatorTest.php tests\Unit\ManifestRequirementsEnforcementStageTest.php tests\Unit\AuthorizationManagerTest.php tests\Unit\DatabaseRelationshipRepositoryTest.php tests\Unit\AuthorizationServiceProviderDatabaseRelationshipTest.php tests\Unit\AuthorizationManagerDatabaseRelationshipTest.php` → **39 tests / 121 assertions, exit 0 con 2 deprecations no bloqueantes**

**Gap natural siguiente**

- commands de auditoría/revocación sobre grants y relaciones,
- tenancy automática uniforme en mas superficies no HTTP,
- invalidacion distribuida/multi-worker para authority cache y relaciones,
- proveedores relacionales mas ricos y no-DBAL,
- y politicas contextuales/risk-based de mayor nivel.

## Corte ejecutado

### DV-AUTHZ-010B

**Tipo:** Tooling operativo inicial
**Estado:** Cerrado
**Objetivo:** Exponer operaciones básicas de auditoría/revocación para relaciones ReBAC sobre drivers `memory` y `database`.

**Alcance ejecutado**

1. se añadió `RelationshipAdministrationInterface` como contrato opt-in para operaciones administrativas,
2. `InMemoryRelationshipRepository` y `DatabaseRelationshipRepository` ahora soportan `listRelationships()` y `revokeRelationshipByKey()`,
3. `AuthorizationServiceProvider` expone el binding administrativo y registra `authz:relationships:list` + `authz:relationships:revoke`,
4. los comandos permiten filtrar/listar relaciones y revocar una relación exacta por `principal_id`, `relation`, `scope` y `resource_key`,
5. el tooling queda alineado con los drivers ya existentes sin romper el contrato runtime de evaluación.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/RelationshipAdministrationInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Relationship/InMemoryRelationshipRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Relationship/DatabaseRelationshipRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationRelationshipsListCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationRelationshipsRevokeCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationRelationshipCommandsTest.php`

**Resultado operativo**

- Authorization ya puede inspeccionar y revocar relaciones ReBAC sin tocar el código ni los seeds manualmente,
- el tooling funciona igual sobre repositorios `memory` y `database`,
- y el módulo gana una primera superficie operativa real antes de entrar en tenancy cross-surface e invalidación distribuida.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\AuthorizationRelationshipCommandsTest.php tests\Unit\InMemoryRelationshipRepositoryTest.php tests\Unit\DatabaseRelationshipRepositoryTest.php tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php tests\Unit\AuthorizationServiceProviderDatabaseRelationshipTest.php tests\Unit\AuthorizationManagerDatabaseRelationshipTest.php` → **26 tests / 82 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests\Unit\AuthorizationRelationshipCommandsTest.php tests\Unit\InMemoryRelationshipRepositoryTest.php tests\Unit\DatabaseRelationshipRepositoryTest.php tests\Unit\RelationshipEvaluatorTest.php tests\Unit\ManifestRequirementsEnforcementStageTest.php tests\Unit\AuthorizationManagerTest.php tests\Unit\AuthorizationManagerDatabaseRelationshipTest.php tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php tests\Unit\AuthorizationServiceProviderDatabaseRelationshipTest.php` → **58 tests / 181 assertions, exit 0 con 2 deprecations no bloqueantes**

**Gap natural siguiente**

- tenancy automática uniforme en mas superficies no HTTP,
- invalidacion distribuida/multi-worker para authority cache y relaciones,
- commands operativos de grant/auditoría más ricos,
- providers externos adicionales,
- y politicas contextuales/risk-based de mayor nivel.

## Corte ejecutado

### DV-AUTHZ-010C

**Tipo:** Tenancy cross-surface inicial
**Estado:** Cerrado
**Objetivo:** Extender la resolución automática de tenant/scope fuera de HTTP para surfaces `command`, `job` y runtime sintético.

**Alcance ejecutado**

1. `AuthorizationContextFactory` ahora puede construir contexto desde `RuntimeContext` activo cuando no existe auth context,
2. el factory proyecta `Request` y `RuntimeContext` hacia `AuthorizationContext` y conserva `runtime.channel`,
3. `TenantScopeResolver` ya resuelve tenant/scope desde `runtime_context`, metadata runtime y request sintético de jobs/commands,
4. gates y evaluaciones sin `RouteMatch` HTTP ya reciben `tenant.id` y `authorization.scope` normalizados cuando el runtime los expone,
5. el comportamiento sigue siendo opt-in via `authorization.authority.scope_resolution.enabled`.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Context/AuthorizationContextFactory.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Context/TenantScopeResolver.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationContextFactoryRuntimeContextTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationMultiSurfaceIntegrationTest.php`

**Resultado operativo**

- Authorization ya normaliza tenant/scope automáticamente en surfaces `cli` y `worker` sin exigir contexto manual,
- los gates pueden decidir con metadata multi-surface consistente usando el mismo pipeline que HTTP,
- y el subsistema queda mejor preparado para workers persistentes antes de introducir invalidación distribuida.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\AuthorizationContextFactoryRuntimeContextTest.php tests\Unit\AuthorizationMultiSurfaceIntegrationTest.php tests\Unit\AuthorizationManagerAuthorityEarlyGateTest.php tests\Unit\MetadataAuthorizationContextEnricherTest.php tests\Unit\ManifestRequirementsEnforcementStageTest.php tests\Unit\ControllerEngineTest.php tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php` → **71 tests / 249 assertions, exit 0 con 2 deprecations no bloqueantes**

**Gap natural siguiente**

- invalidacion distribuida/multi-worker para authority cache y relaciones,
- commands operativos de grant/auditoría más ricos,
- providers externos adicionales,
- y politicas contextuales/risk-based de mayor nivel.

## Corte ejecutado

### DV-AUTHZ-010D

**Tipo:** Consistencia e invalidación generacional inicial
**Estado:** Cerrado
**Objetivo:** Introducir una primera capa de versionado/invalidez para authority memoization y revocaciones relacionales, compatible con workers persistentes.

**Alcance ejecutado**

1. se añadió `AuthorizationConsistencyInterface` con implementación `VersionedAuthorizationConsistency` sobre `VersionAuthorityInterface`,
2. `RequestScopedAuthorityMemoizationCache` ya incorpora la versión compuesta del dominio authority en su clave de memoization,
3. `InMemoryRelationshipRepository` y `DatabaseRelationshipRepository` bump-ean generaciones de relaciones cuando revocan entradas,
4. `AuthorizationServiceProvider` registra el servicio de consistencia, defaults `authorization.consistency.*` y el comando `authz:consistency:invalidate`,
5. la invalidación puede dispararse por `principal`, `scope`, `principal_scope` o `global`, sin romper el comportamiento request-scoped existente.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorizationConsistencyInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Consistency/VersionedAuthorizationConsistency.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/RequestScopedAuthorityMemoizationCache.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Relationship/InMemoryRelationshipRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Relationship/DatabaseRelationshipRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationConsistencyInvalidateCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`

**Resultado operativo**

- Authorization ya puede invalidar memoization authority sin reiniciar el worker,
- las revocaciones ReBAC ya propagan un bump de consistencia reutilizable por futuros caches distribuidos,
- existe una superficie CLI explícita para invalidación operativa,
- y el módulo queda listo para conectar un backend externo real de versiones sin rediseñar el contrato.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\AuthorityMemoizationAndCacheTest.php tests\Unit\AuthorizationConsistencyVersioningTest.php tests\Unit\AuthorizationConsistencyInvalidateCommandTest.php tests\Unit\InMemoryRelationshipRepositoryTest.php tests\Unit\DatabaseRelationshipRepositoryTest.php tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php tests\Unit\AuthorizationRelationshipCommandsTest.php` → **37 tests / 113 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests\Unit\AuthorityMemoizationAndCacheTest.php tests\Unit\AuthorizationConsistencyVersioningTest.php tests\Unit\AuthorizationConsistencyInvalidateCommandTest.php tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php tests\Unit\AuthorizationRelationshipCommandsTest.php tests\Unit\InMemoryRelationshipRepositoryTest.php tests\Unit\DatabaseRelationshipRepositoryTest.php tests\Unit\AuthorizationContextFactoryRuntimeContextTest.php tests\Unit\AuthorizationMultiSurfaceIntegrationTest.php tests\Unit\AuthorizationManagerAuthorityEarlyGateTest.php tests\Unit\AuthorizationManagerDatabaseRelationshipTest.php tests\Unit\MetadataAuthorizationContextEnricherTest.php tests\Unit\ManifestRequirementsEnforcementStageTest.php` → **77 tests / 234 assertions, exit 0 con 2 deprecations no bloqueantes**

**Gap natural siguiente**

- backend remoto multi-node real para `AuthorizationConsistencyInterface` mas alla del backend compartido por filesystem,
- commands operativos de grant/auditoría más ricos,
- providers externos adicionales,
- y politicas contextuales/risk-based de mayor nivel.

## Corte ejecutado

### DV-AUTHZ-010E — CERRADO

`Backend Externo De Consistencia Y Auditoria Operativa Enriquecida`

**Objetivo:** Conectar `AuthorizationConsistencyInterface` a un backend persistente compartido real, manteniendo el modo opt-in y abriendo una superficie de inspeccion operativa de generaciones.

**Alcance ejecutado**

1. se añadió `Quantum\Cache\FileVersionAuthority` como backend lock-safe persistente para scopes/versiones,
2. `AuthorizationServiceProvider` ahora soporta `authorization.consistency.driver=local|file|filesystem|shared`,
3. `authorization.consistency.file.path` permite externalizar el storage de generaciones sin tocar el contrato publico del modulo,
4. `VersionedAuthorizationConsistency` expone `describeAuthority()` y `describeRelationships()` para auditoria operativa de segmentos `global|principal|scope|principal_scope`,
5. se añadió el comando `authz:consistency:report` y se enriquecio `authz:consistency:invalidate --verbose` con backend y namespace efectivos,
6. se validó el flujo cross-instance para demostrar que dos `Application` distintos pueden observar la misma invalidación cuando comparten el mismo storage path.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Cache/FileVersionAuthority.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Consistency/VersionedAuthorizationConsistency.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationConsistencyReportCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationConsistencyInvalidateCommand.php`
- `vendor/voltstack/framework/tests/Unit/FileVersionAuthorityTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationConsistencyReportCommandTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationServiceProviderBridgeAndFlagsTest.php`

**Resultado operativo**

- Authorization deja de depender solo de memoria local del proceso para versionar authority y relationships,
- la memoization request-scoped ya puede invalidarse con una fuente persistente compartida entre workers/aplicaciones que compartan storage,
- existe una superficie CLI read-only para inspeccionar generaciones activas antes de invalidarlas,
- y el corte mantiene compatibilidad hacia atras porque `authorization.consistency.driver` sigue arrancando en `local` por defecto.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\FileVersionAuthorityTest.php tests\Unit\AuthorizationConsistencyVersioningTest.php tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php tests\Unit\AuthorizationConsistencyInvalidateCommandTest.php tests\Unit\AuthorizationConsistencyReportCommandTest.php tests\Unit\AuthorityMemoizationAndCacheTest.php` → **30 tests / 93 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests\Unit\InMemoryRelationshipRepositoryTest.php tests\Unit\DatabaseRelationshipRepositoryTest.php tests\Unit\AuthorizationRelationshipCommandsTest.php tests\Unit\AuthorizationManagerDatabaseRelationshipTest.php tests\Unit\AuthorizationContextFactoryRuntimeContextTest.php tests\Unit\AuthorizationMultiSurfaceIntegrationTest.php tests\Unit\ManifestRequirementsEnforcementStageTest.php` → **42 tests / 131 assertions, exit 0 con 2 deprecations no bloqueantes**

**Gap natural siguiente**

- backend remoto multi-node real para `AuthorizationConsistencyInterface` mas alla del filesystem compartido,
- providers externos adicionales,
- y politicas contextuales/risk-based de mayor nivel.

## Corte ejecutado

### DV-AUTHZ-010F — CERRADO

`Providers Remotos, Grants Operativos Y Adaptive Access Inicial`

**Objetivo:** Abrir tooling operativo real para grants de authority y hacer que las mutaciones de RBAC/direct grants participen del mismo pipeline de consistencia que ya usan relaciones y memoization.

**Alcance ejecutado**

1. se añadió `AuthorityAdministrationInterface` como contrato administrativo explícito para listar, otorgar y revocar grants de authority,
2. `InMemoryAuthorityRepository` y `DatabaseAuthorityRepository` ya implementan administración operativa homogénea (`listGrants`, `grantRole`, `grantPermission`, `revokeRole`, `revokePermission`),
3. las mutaciones de grants ahora invalidan generaciones de `authority` mediante `AuthorizationConsistencyInterface`,
4. `AuthorizationServiceProvider` expone `AuthorityAdministrationInterface` desde el repositorio interno incluso cuando el repositorio de lectura está envuelto por `CachedAuthorityRepository`,
5. se añadieron los comandos `authz:authority:list`, `authz:authority:grant` y `authz:authority:revoke`,
6. el driver `database` ahora no solo resuelve grants, también soporta administración operativa exacta sobre tablas de grants y mantiene consistencia con memoization.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorityAdministrationInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/InMemoryAuthorityRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/DatabaseAuthorityRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationAuthorityListCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationAuthorityGrantCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationAuthorityRevokeCommand.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationAuthorityCommandsTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorityModelAndRepositoryTest.php`
- `vendor/voltstack/framework/tests/Unit/DatabaseAuthorityRepositoryTest.php`

**Resultado operativo**

- Authorization ya dispone de tooling CLI simétrico para authority y relationships,
- los grants RBAC/directos pueden auditarse y modificarse sin abrir APIs ad hoc fuera del módulo,
- las mutaciones de authority ya hacen bump de consistencia, reduciendo drift frente a caches request-scoped y workers persistentes,
- y el módulo queda mejor preparado para futuros providers remotos porque la administración ya no depende de helpers concretos del repo en memoria.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\AuthorityModelAndRepositoryTest.php tests\Unit\DatabaseAuthorityRepositoryTest.php tests\Unit\AuthorizationServiceProviderDatabaseAuthorityTest.php tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php tests\Unit\AuthorityMemoizationAndCacheTest.php tests\Unit\AuthorizationManagerAuthorityEarlyGateTest.php tests\Unit\RuntimeRequestIsolationTest.php` → **50 tests / 167 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests\Unit\AuthorizationAuthorityCommandsTest.php tests\Unit\AuthorizationRelationshipCommandsTest.php tests\Unit\AuthorizationConsistencyVersioningTest.php tests\Unit\AuthorizationConsistencyInvalidateCommandTest.php tests\Unit\AuthorizationConsistencyReportCommandTest.php tests\Unit\AuthorizationContextFactoryRuntimeContextTest.php tests\Unit\AuthorizationMultiSurfaceIntegrationTest.php tests\Unit\ManifestRequirementsEnforcementStageTest.php tests\Unit\AuthorizationManagerDatabaseRelationshipTest.php` → **54 tests / 171 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests\Unit\AuthorizationServiceProviderDatabaseRelationshipTest.php tests\Unit\AuthorizationServiceProviderDatabaseAuthorityTest.php tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php tests\Unit\DatabaseRelationshipRepositoryTest.php tests\Unit\DatabaseAuthorityRepositoryTest.php` → **22 tests / 70 assertions, exit 0 con 2 deprecations no bloqueantes**

**Gap natural siguiente**

- backend remoto multi-node real para `AuthorizationConsistencyInterface` mas alla del filesystem compartido,
- providers externos adicionales,
- y politicas contextuales/risk-based de mayor nivel.

## Corte ejecutado

### DV-AUTHZ-010G — CERRADO

`Providers Remotos Y Adaptive Access Inicial`

**Objetivo:** Activar una primera capa explícita de adaptive access en `Authorization`, aprovechando la señal de riesgo ya proyectada desde `Auth` y haciendo que el pipeline pueda devolver challenge/deny por thresholds antes de gate/policy.

**Alcance ejecutado**

1. se añadió `AdaptiveAccessStage` al planner de `Authorization` como etapa temprana del pipeline,
2. `authorization.adaptive_access.*` ahora permite habilitar thresholds de `step_up` y `deny`, definir claves de riesgo y parametrizar endpoints/metadatos de challenge,
3. `AuthorizationContextFactory` normaliza aliases de riesgo (`auth_risk_score`, `auth_risk_level`) y assurance desde el contexto de autenticación,
4. `AuthorizationExceptionMapper` ahora proyecta headers/extensiones de riesgo y step-up cuando la decisión viene de adaptive access,
5. el pipeline puede devolver `DecisionResult::challenge()` con `auth.step_up_required` o `DecisionResult::deny()` con `auth.risk_denied` sin romper el resto del módulo declarativo.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Core/Stages/AdaptiveAccessStage.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Context/AuthorizationContextFactory.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Exceptions/AuthorizationExceptionMapper.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManagerTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationServiceProviderTest.php`
- `vendor/voltstack/framework/tests/Unit/QuantumExceptionHandlerTest.php`

**Resultado operativo**

- Authorization ya no depende solo de ABAC declarativo manual para reaccionar al riesgo,
- un score alto puede forzar `step_up` o bloqueo temprano dentro del propio pipeline de autorización,
- los consumers HTTP reciben headers reutilizables (`X-Auth-Step-Up`, `X-Auth-Risk-*`) compatibles con la semántica ya usada por `Auth`,
- y el módulo gana una base inicial para politicas contextuales más ricas sin acoplar toda la lógica al subsistema de autenticación.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\AuthorizationManagerTest.php tests\Unit\AuthorizationServiceProviderTest.php tests\Unit\QuantumExceptionHandlerTest.php` → **38 tests / 148 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php tests\Unit\AuthorizationAuthorityCommandsTest.php tests\Unit\AuthorizationRelationshipCommandsTest.php tests\Unit\AuthorizationConsistencyReportCommandTest.php tests\Unit\AuthorizationConsistencyInvalidateCommandTest.php tests\Unit\AuthorizationMultiSurfaceIntegrationTest.php tests\Unit\ManifestRequirementsEnforcementStageTest.php` → **58 tests / 173 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests\Unit\AuthorizationContextFactoryRuntimeContextTest.php tests\Unit\ControllerSecurityContextFactoryTest.php` → **12 tests / 84 assertions, exit 0**
- `vendor\bin\phpunit tests\Unit\AuthorizationServiceProviderDatabaseAuthorityTest.php tests\Unit\DatabaseAuthorityRepositoryTest.php tests\Unit\AuthorityMemoizationAndCacheTest.php` → **15 tests / 40 assertions, exit 0 con 2 deprecations no bloqueantes**

**Gap natural siguiente**

- backend remoto multi-node real para `AuthorizationConsistencyInterface`,
- providers externos adicionales para authority/relationships,
- y politicas adaptativas mas finas por tenant/canal/operacion.

## Siguiente corte recomendado

### DV-AUTHZ-010H — SIGUIENTE

`Providers Remotos Y Consistencia Distribuida Real`

### Avance parcial actual sobre DV-AUTHZ-010H

`Extensibilidad formal de drivers/providers`

**Objetivo parcial ejecutado:** Desacoplar `AuthorizationServiceProvider` de la resolución fija `memory|database|file` y abrir un punto estable para que paquetes externos registren drivers propios de `authority`, `relationships` y `consistency`.

**Alcance ejecutado**

1. se añadió `AuthorizationDriverRegistry` como registry singleton para registrar factories de drivers personalizados,
2. `AuthorizationServiceProvider` ahora consulta esa registry antes de caer en los drivers built-in de authority, relationships y consistency,
3. un provider externo ya puede registrar adapters remotos sin parchear el core del módulo,
4. se cubrió la resolución real desde container para drivers personalizados en las tres superficies.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationDriverRegistry.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationServiceProviderTest.php`

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\AuthorizationServiceProviderTest.php tests\Unit\AuthorizationServiceProviderBridgeAndFlagsTest.php tests\Unit\AuthorizationServiceProviderDatabaseAuthorityTest.php tests\Unit\AuthorizationServiceProviderDatabaseRelationshipTest.php` → **26 tests / 62 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests\Unit\AuthorizationAuthorityCommandsTest.php tests\Unit\AuthorizationRelationshipCommandsTest.php tests\Unit\AuthorizationConsistencyReportCommandTest.php tests\Unit\AuthorizationConsistencyInvalidateCommandTest.php tests\Unit\AuthorizationManagerTest.php tests\Unit\AuthorizationMultiSurfaceIntegrationTest.php` → **44 tests / 131 assertions, exit 0 con 2 deprecations no bloqueantes**

**Gap restante para cerrar DV-AUTHZ-010H**

- backend remoto multi-node real para `AuthorizationConsistencyInterface`,
- selective flush remoto por `principal` y/o `scope`,
- adapters remotos concretos para `authority`/`relationships`,
- y observabilidad operativa de invalidación distribuida.

### Avance parcial adicional sobre DV-AUTHZ-010H

`Backend compartido de consistency sobre CacheModule (driver cache configurable)`

**Objetivo parcial ejecutado:** Añadir un adapter concreto de `VersionAuthorityInterface` sobre `Quantum/Cache` y habilitar `authorization.consistency.driver=cache` con store/prefix/ttl configurables, obteniendo un backend multi-instancia compartido sin inventar un storage nuevo.

**Alcance ejecutado**

1. se añadió `CacheVersionAuthority` implementando `VersionAuthorityInterface` sobre cualquier `StoreInterface` del CacheModule (FileStore, Redis, APCu, DB, etc.),
2. `VersionedAuthorizationConsistency` ya puede operar con ese adapter sin cambios de contrato,
3. `AuthorizationServiceProvider` ahora reconoce `consistency.driver=cache` y crea el adapter via `CacheManager::store()`,
4. el block de config `authorization.consistency.cache.*` expone `store`, `prefix` y `ttl_seconds`,
5. se valida que la invalidación de un proceso se observa desde otro proceso cuando ambos comparten el mismo store físico (FileStore compartido).

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Cache/CacheVersionAuthority.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/tests/Unit/CacheVersionAuthorityTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationServiceProviderBridgeAndFlagsTest.php`

**Validacion ejecutada**

- `vendor\bin\phpunit tests/Unit/CacheVersionAuthorityTest.php tests/Unit/FileVersionAuthorityTest.php tests/Unit/AuthorizationServiceProviderTest.php tests/Unit/AuthorizationServiceProviderBridgeAndFlagsTest.php tests/Unit/AuthorizationServiceProviderDatabaseAuthorityTest.php tests/Unit/AuthorizationServiceProviderDatabaseRelationshipTest.php` → **34 tests / 84 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests/Unit/AuthorizationConsistencyReportCommandTest.php tests/Unit/AuthorizationConsistencyInvalidateCommandTest.php tests/Unit/AuthorizationConsistencyVersioningTest.php tests/Unit/AuthorityMemoizationAndCacheTest.php tests/Unit/DatabaseRelationshipRepositoryTest.php tests/Unit/AuthorizationManagerAuthorityEarlyGateTest.php tests/Unit/AuthorizationMultiSurfaceIntegrationTest.php tests/Unit/AuthorizationAuthorityCommandsTest.php tests/Unit/AuthorizationRelationshipCommandsTest.php` → **53 tests / 150 assertions, exit 0 con 2 deprecations no bloqueantes**

**Gap restante para cerrar DV-AUTHZ-010H**

- selective flush remoto observado y auditado,
- adapters remotos concretos para `authority` y `relationships` sobre la registry abierta,
- y observabilidad operativa + comandos de doctor sobre invalidación distribuida.

### Tercer avance parcial sobre DV-AUTHZ-010H

`Observabilidad de selective flush + command doctor de consistencia`

**Objetivo parcial ejecutado:** Cerrar la brecha “observabilidad + doctor” del plan: registrar `reason`/`last_bump_at`/contadores por segmento en el flujo de invalidación, exponerlos programáticamente y superponer un comando `authz:consistency:doctor` que diagnostique driver, backend (file/cache/custom), config declarada y auditoria de bumps del proceso actual.

**Alcance ejecutado**

1. `AuthorizationConsistencyInterface` amplía las firmas `invalidateAuthority(..., ?string $reason = null)` e `invalidateRelationships(..., ?string $reason = null)` y añade `inspect(): array` genérico para backends custom,
2. `VersionedAuthorizationConsistency` ahora mantiene `lastBumpAt`, `bumpCounters()`, `bumpReasons()`, `lastBumpTimestamps()` por segmento (`authority.global`, `authority.principal`, `relationships.scope`, etc.),
3. `describeVersionAuthority()` reporta `kind=file` (storage_path) y `kind=cache` (store class + prefix), y esa proyección vuelca a `inspect()` + report/doctor commands,
4. `authz:consistency:report` JSON incluye `inspect` y `last_bump_at`, mientras que la salida humana `--verbose` muestra `Backend details` y `Bump counters`,
5. `authz:consistency:invalidate` acepta `--reason` y expone `reason` + `inspect()` en JSON,
6. nuevo `authz:consistency:doctor` emite `ok`, `is_versioned`, la config activa (`consistency_driver_config`) y `inspect`; en `--verbose` detalla bump counters con sus razones; soporta tanto `VersionedAuthorizationConsistency` como backends custom vía la proyección genérica de `inspect()`,
7. `AuthorizationServiceProvider::commands()` registra el doctor sin configuración extra.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorizationConsistencyInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Consistency/VersionedAuthorizationConsistency.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationConsistencyReportCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationConsistencyInvalidateCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationConsistencyDoctorCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationConsistencyVersioningTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationConsistencyReportCommandTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationConsistencyInvalidateCommandTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationConsistencyDoctorCommandTest.php`

**Validacion ejecutada**

- `vendor\bin\phpunit tests/Unit/CacheVersionAuthorityTest.php tests/Unit/FileVersionAuthorityTest.php tests/Unit/AuthorizationServiceProviderTest.php tests/Unit/AuthorizationServiceProviderBridgeAndFlagsTest.php tests/Unit/AuthorizationPublishedConfigGateTest.php tests/Unit/AuthorizationConsistencyVersioningTest.php tests/Unit/AuthorizationConsistencyReportCommandTest.php tests/Unit/AuthorizationConsistencyInvalidateCommandTest.php tests/Unit/AuthorizationConsistencyDoctorCommandTest.php` → **50 tests / 184 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests/Unit/AuthorizationConsistencyReportCommandTest.php tests/Unit/AuthorizationConsistencyInvalidateCommandTest.php tests/Unit/AuthorizationConsistencyVersioningTest.php tests/Unit/AuthorizationConsistencyDoctorCommandTest.php tests/Unit/AuthorityMemoizationAndCacheTest.php tests/Unit/AuthorizationManagerAuthorityEarlyGateTest.php tests/Unit/AuthorizationMultiSurfaceIntegrationTest.php tests/Unit/AuthorizationAuthorityCommandsTest.php tests/Unit/AuthorizationRelationshipCommandsTest.php tests/Unit/AuthorizationManagerTest.php tests/Unit/AuthorizationManagerDatabaseRelationshipTest.php tests/Unit/AuthorizationContextFactoryRuntimeContextTest.php tests/Unit/AuthorityModelAndRepositoryTest.php` → **84 tests / 305 assertions, exit 0 con 2 deprecations no bloqueantes**
- `vendor\bin\phpunit tests/Unit/DatabaseRelationshipRepositoryTest.php tests/Unit/DatabaseAuthorityRepositoryTest.php tests/Unit/AuthorizationServiceProviderDatabaseAuthorityTest.php tests/Unit/AuthorizationServiceProviderDatabaseRelationshipTest.php` → **12 tests / 38 assertions, exit 0 con 2 deprecations no bloqueantes** (suite DB pesada, ejecutada en aislamiento para evitar “premature end of process” por mezcla con suites muy grandes del mismo proceso PHPUnit, sin relación con este corte)

**Gap restante para cerrar DV-AUTHZ-010H**

- backend remoto nativo multi-node real (Redis/equivalente distribuido fuerte) como driver built-in,
- adapters remotos concretos para `authority` y `relationships` sobre `AuthorizationDriverRegistry`,
- y propagación distribuida real de razones/contadores (hoy el proceso acumula contadores en memoria; un backend compartido debe persistirlos cross-node).

Foco:

- `12`, `13`, `19`, `20`, `23`, `27`, `29`, `31`

### Cuarto avance parcial sobre DV-AUTHZ-010H

`Providers Remotos Y Consistencia Distribuida Real`

**Objetivo parcial ejecutado:** Cerrar los tres gaps pendientes del tercer corte: (1) backend distribuido nativo multi-nodo equivalente a Redis via `CacheVersionAuthority` sobre stores que soportan incremento atómico (CAS-like), (2) adapters remotos concretos para authority y relationships sobre `AuthorizationDriverRegistry` con provider público de bootstrap, y (3) persistencia distribuida de metadatos de auditoría (`bump_counter`, `last_reason`, `last_bump_at`) en los envelopes compartidos file/cache (dejaron de ser solo in-process).

**Alcance ejecutado**

1. Nueva interface `Quantum\Cache\Contracts\AtomicIncrementableStoreInterface extends StoreInterface` con `incrementInt(key, step, initial, ttl)` — semántica CAS-like sobre backends compartidos,
2. `MemoryStore` implementa `AtomicIncrementableStoreInterface` (backing in-process); `FileStore` también la implementa con `LOCK_EX` de sistema operativo sobre archivo `.lock` adyacente (equivalente local de CAS distribuido para deploy single-host, utilizable en testing y staging),
3. `CacheVersionAuthority.ENVELOPE_VERSION = 2`, nuevo `?string $reason = null` en `bump()`, y ruta `bumpAtomically()` sobre `.ctr` counter cuando el store es `AtomicIncrementableStoreInterface`; migra on-the-fly desde schema `ENVELOPE_VERSION=1` preservando la versión y derivando `bump_counter = version - 1`,
4. `FileVersionAuthority.ENVELOPE_VERSION = 2`, mismo shape de envelope distribuido (`envelope`, `scope`, `version`, `updated_at`, `bump_counter`, `last_reason`, `last_bump_at`) con upgrade on-the-fly desde schema legacy,
5. `CacheVersionAuthority::readEnvelope(scope)` y `FileVersionAuthority::readEnvelope(scope)` como surface pública estable para commands doctor/report,
6. `VersionedAuthorizationConsistency::bumpSegment()` discrimina authority File/Cache para pasar `$reason` al `bump()` subyacente, y añade helper privado `readSharedEnvelope()` que proyecta `bump_counter / last_reason / last_bump_at` desde el envelope compartido cross-node sobre los mapas in-process,
7. Repositorios administrativos (InMemory y Database, authority y relationships) enhebran `reason` automáticos (`authority.grant_role`, `authority.grant_permission`, `authority.revoke_*`, `relationships.revoke`, etc.) en cada invalidación,
8. `RemoteCacheAuthorityRepository` implementa `AuthorityRepositoryInterface` + `AuthorityAdministrationInterface` sobre store compartido: guarda envelopes por (principal, scope), mantiene índice por principal + registry global, y produce `effectivePermissionsForPrincipal / hasRole / hasPermission / listGrants / scopesForPrincipal`,
9. `RemoteCacheRelationshipRepository` implementa `RelationshipRepositoryInterface` + `RelationshipAdministrationInterface` sobre store compartido: `assignRelationship`, `relationshipsOf`, `hasRelationship`, `revokeRelationships`, `storeRelationship` (para seedings/scripts),
10. `AuthorizationRemoteCacheDriverProvider::register($app, $options)` registra en `AuthorizationDriverRegistry` el driver `remote-cache` para consistency, authority y relationships con claves de configuración separadas (`prefix`, `consistency_prefix`, `store` opcional).

**Evidencia — archivos creados**

- `vendor/voltstack/framework/src/Quantum/Cache/Contracts/AtomicIncrementableStoreInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/RemoteCacheAuthorityRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Relationship/RemoteCacheRelationshipRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationRemoteCacheDriverProvider.php`
- `vendor/voltstack/framework/tests/Unit/SharedVersionEnvelopeAndAtomicIncrementTest.php`
- `vendor/voltstack/framework/tests/Unit/RemoteCacheAuthorityRepositoryTest.php`
- `vendor/voltstack/framework/tests/Unit/RemoteCacheRelationshipRepositoryTest.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationRemoteCacheDriverProviderTest.php`

**Evidencia — archivos modificados**

- `vendor/voltstack/framework/src/Quantum/Cache/MemoryStore.php`
- `vendor/voltstack/framework/src/Quantum/Cache/FileStore.php`
- `vendor/voltstack/framework/src/Quantum/Cache/CacheVersionAuthority.php`
- `vendor/voltstack/framework/src/Quantum/Cache/FileVersionAuthority.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Consistency/VersionedAuthorizationConsistency.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/InMemoryAuthorityRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/DatabaseAuthorityRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Relationship/InMemoryRelationshipRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Relationship/DatabaseRelationshipRepository.php`

**Validacion ejecutada**

- Suite tests nuevos del cuarto corte (11 tests): `SharedVersionEnvelopeAndAtomicIncrementTest`, `RemoteCacheAuthorityRepositoryTest`, `RemoteCacheRelationshipRepositoryTest`, `AuthorizationRemoteCacheDriverProviderTest` → **11 tests / 86 assertions, exit 0 con 1 PHPUnit notice no bloqueante**
- Bloque 1 de regresión (50 tests originales + 11 nuevos = 61 tests total): `CacheVersionAuthorityTest`, `FileVersionAuthorityTest`, `AuthorizationServiceProviderTest`, `AuthorizationServiceProviderBridgeAndFlagsTest`, `AuthorizationPublishedConfigGateTest`, `AuthorizationConsistencyVersioningTest`, `AuthorizationConsistencyReportCommandTest`, `AuthorizationConsistencyInvalidateCommandTest`, `AuthorizationConsistencyDoctorCommandTest` + 4 suites nuevas arriba → **61 tests / 270 assertions, exit 0 con 2 deprecations / 1 notice no bloqueantes**
- Bloque 2 de regresión (surfaces administrativas + memoization + manager): `AuthorizationConsistencyReportCommandTest`, `AuthorizationConsistencyInvalidateCommandTest`, `AuthorizationConsistencyVersioningTest`, `AuthorizationConsistencyDoctorCommandTest`, `AuthorityMemoizationAndCacheTest`, `AuthorizationManagerAuthorityEarlyGateTest`, `AuthorizationMultiSurfaceIntegrationTest`, `AuthorizationAuthorityCommandsTest`, `AuthorizationRelationshipCommandsTest`, `AuthorizationManagerTest`, `AuthorizationManagerDatabaseRelationshipTest`, `AuthorizationContextFactoryRuntimeContextTest`, `AuthorityModelAndRepositoryTest` → **84 tests / 305 assertions, exit 0 con 2 deprecations no bloqueantes**
- Bloque 3 DB (aislado por teardown pesado): `DatabaseRelationshipRepositoryTest`, `DatabaseAuthorityRepositoryTest`, `AuthorizationServiceProviderDatabaseAuthorityTest`, `AuthorizationServiceProviderDatabaseRelationshipTest` → **12 tests / 38 assertions, exit 0 con 2 deprecations no bloqueantes**

**Gap restante (prioridad MEDIUM, después de cerrado 010H)**

- Extender `AtomicIncrementableStoreInterface` a adaptadores externos no built-in (Memcached, Redis real beyond Predis/phpredis custom),
- Thread explícito `--reason` desde los comandos CLI `authz:authority:*` y `authz:relationships:*` hacia las interfaces administrativas (hoy los reasons son strings fijos dentro de cada repo),
- Documentar el orden óptimo de suites PHPUnit (DB pesadas aisladas del resto).

**Siguiente fase natural: `DV-AUTHZ-010I = Delegation / Impersonation + Service-to-Service Principals`**

Foco:

- `14`, `15`, `16`, `21`, `22`, `24`, `25`, `26`

## Corte ejecutado

### DV-AUTHZ-010I

**Tipo:** Delegation / Impersonation + Service-to-Service Principals  
**Estado:** Cerrado  
**Fecha:** `2026-10-10`  
**Objetivo:** Introducir una capa opt-in de delegación de permisos entre principals (trustee ↔ grantor), soporte de impersonation (originator → target) con contexto explícito, resolución de principals Service-to-Service configurables sin fuga de identidad de usuario humano, y proyección de contexto delegation/impersonation al pipeline de decisión de authorization.

**Documentos impactados**

- `DEVELOPMENT_GUIDELINES.md`
- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- `EXECUTIVE_PLAN_IMPLEMENTATION.md`

**Alcance ejecutado en este corte**

1. **Contracts y Value Objects de Delegation:**
   - `DelegationAdministrationInterface` con `listDelegations/grantDelegation/revokeDelegation` (filters por trustee_id, grantor_id, scope, type, value).
   - `ServicePrincipalResolverInterface` como punto de extensión para resolver principals de tipo Service/ApiClient sin depender de capa Auth HTTP.
   - `DelegationGrant` VO readonly con `trusteeId/grantorId/scope/grantType/grantValue/grantedAt`, `toArray()` y `JsonSerializable`.
2. **Impersonation runtime:**
   - `AuthorizationManagerInterface::impersonate(caller,target,?Scope)` añadido como 3er binding helper al final de la interfaz (compat 100% legacy).
   - `ImpersonationPrincipalBuilder` build de `ImpersonatedUser` soporta duck-typing: id() getter, propiedad pública `id`, `PrincipalInterface`, string/int.
   - `BoundAuthorization` añade 3er parámetro opcional `?AuthorizationContext $context = null` + helper `mergeContext(?A,?A)` con merge de atributos `context.bind ∪ context.explicit`. Los 4 métodos públicos `check/cannot/decide/authorize` aplican el merge transparente, sin impacto para usuarios de la API legacy 2-args.
3. **Delegation repositories + manifest fallback semántico:**
   - `InMemoryAuthorityRepository` y `DatabaseAuthorityRepository` implementan `DelegationAdministrationInterface` (key única 5-column: `trustee#grantor#scope#type#value`).
   - `DatabaseAuthorityRepository` tabla configurable: `authorization.tables.delegation_grants` (default `authorization_delegation_grants`) con `trustee_id, grantor_id, scope, grant_type, grant_value, granted_at, UNIQ(5cols)`.
   - `ManifestRequirementsEnforcementStage` nuevo: params `?DelegationAdministrationInterface`, `evaluateDelegations=false`. Detecta impersonation (principal tipo `ImpersonatedUser` + `authorization.impersonation.originator_id/target_id`). **Orden estricto fail-closed:**
     1. `hasPermission(targetId, perm, scope)` directo → si TRUE, ALLOW sin delegation.
     2. SOLO si está en impersonation Y falló el permiso propio del target → check delegation grants:
        - `listDelegations(trustee, grantor, scope)` → match explícito por permission name.
        - **Regla semántica principal:** si existe vínculo trustee-grantor, consultar `authorityRepository->hasPermission($grantorId, $perm, $scope)` → grantor lo tiene → ALLOW con metadata delegation.
        - Fallback último: role expansion si Role ctor trae permisos.
   - Metadata en ALLOW por delegation: `delegation_granted, delegation_trustee_id, delegation_grantor_id, originator_principal_id, target_principal_id, impersonation_scope`.
4. **Service Principal resolver + wire:**
   - `ConfigurableServicePrincipalResolver` orden resolución: runtime flag `as/as_service` → request attribute `service_principal.as` → query param `as-service` → SERVER `VOLT_AS_SERVICE` → runtime metadata `service_principal.id/type/claims` → config map `authorization.service_principals.map.<id>`. Claims finales: `config ∪ runtime.metadata` (runtime gana).
   - `PrincipalResolver` nuevos params: `?ServicePrincipalResolverInterface`, `enabled=false` (default off = fail-closed). Bloque previo a Anonymous: si flag enabled + resolver not null → intenta resolve; catch todo falla cerrada (no romper).
5. **DelegationContextEnricher:** proyecta atributos `authorization.originator.id/type`, `authorization.target.id/type`, `authorization.impersonation.active/originator_id/target_id/scope`, `authorization.service.id/type/claims/resolved_via` cuando corresponde, al final del pipeline de enrichers (solo si delegation o service_principal están habilitados).
6. **CLI delegation commands + authority --view=delegations + consistency bumps report/doctor:**
   - Nuevos: `authz:delegation:list` (filtros trustee/grantor/scope/type, JSON/simple, paged).
   - Nuevos: `authz:delegation:grant` + `authz:delegation:revoke` (trustee/grantor/scope + role|permission, verbose, dry-run, `--require-published-config` pattern estándar).
   - `authz:authority:list` nuevo `--view=simple|delegations|all` (simple default), `--grantor-id` filter.
   - `authz:consistency:report` y `authz:consistency:doctor` JSON y humano incluyen `delegation_bumps` y `service_principal_bumps` (contando reasons con prefijo `delegation.` y `service.`).
   - Mutaciones administrativas delegation emiten consistency reason `delegation.grant` / `delegation.revoke`.
7. **ServiceProvider wiring (incremental, todo off):**
   - defaults fusiona: `delegation.enabled=false`, `service_principal_resolver.enabled=false`, `service_principals.map=[]`, `authority.evaluate_delegations=false`.
   - `registerDelegationAndServicePrincipalBindings()` condicional si `delegationOrServicePrincipalEnabled($app)` (OR). Binda `DelegationAdministrationInterface` a la misma instancia del repositorio authority (no wrapper memoized, mismo inner que AuthorityAdmin). Binda `ServicePrincipalResolverInterface` singleton → `ConfigurableServicePrincipalResolver(config map)`.
   - `ManifestRequirementsEnforcementStage` recibe `evaluateDelegations = explicit_flag || (delegation.enabled=true)` (inference rule).
   - `AuthorizationPlanner` enrichers array condicionalmente agrega `DelegationContextEnricher::class` al final (try/catch, solo si flags ON).
   - Stage order preservation estricto: `[AdaptiveAccessStage, ManifestRequirements, Gate, Policy]` (4 stages = backward compat 100%).
   - `commands()` → 13 comandos (10 anteriores + Delegation[List|Grant|Revoke]).

**Evidencia — archivos NUEVOS**

- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/DelegationAdministrationInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/ServicePrincipalResolverInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/DelegationGrant.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/ImpersonationPrincipalBuilder.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/ServicePrincipal/ConfigurableServicePrincipalResolver.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Enrichers/DelegationContextEnricher.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationDelegationListCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationDelegationGrantCommand.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationDelegationRevokeCommand.php`
- `tests/Unit/AuthorizationDelegationContractsTest.php`
- `tests/Unit/AuthorizationImpersonationTest.php`
- `tests/Unit/AuthorizationDelegationRuntimeTest.php`
- `tests/Unit/AuthorizationDelegationConsistencyTest.php`
- `tests/Unit/AuthorizationServicePrincipalTest.php`
- `tests/Unit/AuthorizationDelegationCommandsTest.php`
- `tests/Unit/AuthorizationManagerImpersonationAndServiceTest.php`
- `tests/Feature/AuthorizationDelegationIntegrationTest.php`

**Evidencia — archivos MODIFICADOS (subset relevante)**

- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorizationManagerInterface.php` (+ `impersonate` al final)
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/AuthorizationManager.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/BoundAuthorization.php` (3er param opcional + mergeContext)
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/InMemoryAuthorityRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/DatabaseAuthorityRepository.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/Stages/ManifestRequirementsEnforcementStage.php` (delegation fallback semantic con authority lookup)
- `vendor/voltstack/framework/src/Quantum/Authorization/Principal/PrincipalResolver.php` (service resolver integration fail-closed)
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationAuthorityListCommand.php` (--view=, --grantor-id)
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationConsistencyReportCommand.php` (delegation/service bumps)
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationConsistencyDoctorCommand.php` (delegation/service bumps)
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php` (wiring delegation, s2s, inference, planner enricher)

**Resultado operativo**

1. Impersonation funciona con APIs públicas nuevas y semántica legacy transparente: `$manager->impersonate($orig, $target, $scope)->check('perm')`.
2. Delegation grants se otorgan/revocan/listan desde CLI (drivers memory/database/remote-cache implementando la interfaz).
3. Fallback delegation semánticamente correcto: "el trustee NO tenía permiso propio PERO el vínculo trustee↔grantor existe Y el grantor SÍ tiene el permiso" → ALLOW con metadata trazable. Fail-closed SIEMPRE: sin impersonation, sin vinculo, o sin permiso del grantor → NO pasa.
4. Service Principals se resuelven sin fuga de identidad de usuario: resolución en runtime request attribute / query / server / metadata / config. Defaults off por surface.
5. Consistency tracing ahora distingue authority.grant/revoke de delegation.grant/revoke y service.* bumps; report y doctor lo muestran en JSON y humano.
6. Cero breaking changes: ninguna API pública preexistente se rompió (Bound 3rd param opcional, ManagerInterface `impersonate()` añadido al final, stage order intacto, bindings opcionales).

**Validacion ejecutada (Task 9)**

- `.\vendor\bin\phpunit.bat --filter=Authorization` → **191 tests, 653 assertions, exit 0**.
- Tests nuevos Task 9 (8 archivos): **passing** (incluyen impersonation, delegation runtime direct + manifest stage fallback semántico, consistency bumps delegation/service, CLI commands 7 tests, contracts grant/revoke/idempotency, service principal resolver con Request y RuntimeContext, provider wiring resolución DelegationAdmin + S2S Resolver, integration end-to-end impersonation → delegation allow).
- Riesgo 1 risky test preexistente (`ExceptionHandlingTest`) no relacionado al corte.

**Siguiente corte recomendado: DV-AUTHZ-010K = Shadow delegation grants + revocation policy engine + selective flush cross-worker**

## Corte ejecutado

### DV-AUTHZ-010J

**Tipo:** Adaptive Access tenant/canal/operación + Delegation TTL + Selective Flush  
**Estado:** Cerrado  
**Fecha:** `2026-10-10`  
**Objetivo:** Enriquecer `AdaptiveAccessStage` con políticas adaptativas por dimensión (tenant/canal/operación) y umbral `allow_threshold` de corte temprano, añadir TTL/expiry a grants de delegación con filtrado automático de grants expirados en runtime, exponer `revokeAllDelegations` para revocación masiva con consistency bump, y añadir `flushMetrics` observabilidad al backend de consistencia + comando `authz:consistency:invalidate --dry-run`.

**Documentos impactados**

- `DEVELOPMENT_GUIDELINES.md`
- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- `EXECUTIVE_PLAN_IMPLEMENTATION.md`

**Alcance ejecutado en este corte**

1. **Adaptive Access por dimensiones + allow_threshold:**
   - `AdaptiveAccessStage` ahora soporta `policies` indexadas por dimensión (`tenant:<id>`, `canal:<canal>`, `operation:<ability>`) con overrides de `deny_threshold`, `step_up_threshold`, `allow_threshold`.
   - Resolución de política: operation > canal > tenant > global fallback. Si no hay policy para la dimensión, usa los thresholds globales del config.
   - Nuevo `allow_threshold`: si `risk_score <= allow_threshold` → ALLOW directo (corte temprano), sin pasar por los demás stages.
   - Nuevo `AdaptiveAccessRuntimeConfig` (scoped) permite overrides runtime de thresholds (para CLI `authz:adaptive:tune`).
2. **Delegation TTL / Expiry:**
   - `DelegationGrant` VO ahora incluye `expiresAt` (?string) y `toArray()` expone `expires_at`.
   - `DelegationAdministrationInterface::grantDelegation` gana parámetro opcional `?string $expiresAt = null` (backward compat).
   - Repositorios (`InMemoryAuthorityRepository`, `DatabaseAuthorityRepository`) normalizan `expiresAt` (ISO 8601 o strtotime relative) y lo persisten.
   - `listDelegations` filtra grants expirados por defecto; flag `include_expired=true` los devuelve.
   - `ManifestRequirementsEnforcementStage::delegationGrantsPermission()` tiene defensa en profundidad: salta grants con `expires_at <= now` incluso si el repo no los filtró.
3. **revokeAllDelegations:**
   - `DelegationAdministrationInterface::revokeAllDelegations(?trusteeId, ?grantorId, ?scope): int` revoca todos los grants que coincidan con los filtros. Al menos uno de trusteeId/grantorId obligatorio; ambos null = no-op (retorna 0).
   - Emite consistency bump único con reason `delegation.revoke_all` (no múltiple por grant).
4. **flushMetrics + CLI --dry-run:**
   - `AuthorizationConsistencyInterface::flushMetrics(): array` devuelve métricas por segmento (`authority.global`, `authority.principal`, `authority.scope`, `authority.principal_scope`, `relationships.*`, `delegation.*`) con `version`, `bump_counter`, `last_bump_at`, `last_reason`, más `total_bumps`.
   - `authz:consistency:invalidate` ahora soporta `--dry-run`: muestra qué segmentos se invalidarían sin mutar estado.
5. **CLI `authz:adaptive:tune`:**
   - Comando nuevo: modo default read-only emite JSON con `enabled`, thresholds, `policies_count`, `policies` (keys), `runtime_overrides`, `tune_enabled`, `applied`.
   - Modo `--apply` requiere `adaptive.tune.enabled=true` (exit 1 si off); aplica overrides a `AdaptiveAccessRuntimeConfig` scoped.
   - Sigue el patrón `--require-published-config` estándar.
6. **ServiceProvider wiring (incremental, todo opt-in):**
   - defaults fusiona: `adaptive_access.policies=[]`, `adaptive_access.allow_threshold=null`, `adaptive.tune.enabled=false`, `delegation.default_ttl_seconds=null`.
   - Binding `AdaptiveAccessRuntimeConfig::class` scoped.
   - `AdaptiveAccessStage` inyecta `policies`, `allowThreshold`, `runtimeConfig`.
   - `commands()` ahora tiene 14 comandos (+`AuthorizationAdaptiveTuneCommand`).

**Evidencia — archivos NUEVOS**

- `vendor/voltstack/framework/src/Quantum/Authorization/Core/Stages/AdaptiveAccessRuntimeConfig.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationAdaptiveTuneCommand.php`
- `tests/Unit/AuthorizationAdaptiveDimensionsTest.php`
- `tests/Unit/AuthorizationDelegationTtlTest.php`
- `tests/Unit/AuthorizationDelegationTtlRuntimeTest.php`
- `tests/Unit/AuthorizationDelegationRevokeAllTest.php`
- `tests/Unit/AuthorizationConsistencyFlushMetricsTest.php`
- `tests/Unit/AuthorizationAdaptiveTuneCommandTest.php`
- `tests/Feature/AuthorizationAdaptiveAndTtlIntegrationTest.php`

**Evidencia — archivos MODIFICADOS (subset relevante)**

- `vendor/voltstack/framework/src/Quantum/Authorization/Core/Stages/AdaptiveAccessStage.php` (dimensiones + allow_threshold + runtime config)
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/DelegationGrant.php` (expiresAt)
- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/DelegationAdministrationInterface.php` (grantDelegation $expiresAt, revokeAllDelegations)
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/InMemoryAuthorityRepository.php` (expiry + revokeAll)
- `vendor/voltstack/framework/src/Quantum/Authorization/Authority/DatabaseAuthorityRepository.php` (expiry + revokeAll)
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/Stages/ManifestRequirementsEnforcementStage.php` (skip expired grants defense)
- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorizationConsistencyInterface.php` (flushMetrics)
- `vendor/voltstack/framework/src/Quantum/Authorization/Console/Commands/AuthorizationConsistencyInvalidateCommand.php` (--dry-run)
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php` (wiring adaptive tune + delegation ttl defaults)

**Resultado operativo**

1. Adaptive Access ahora puede tunearse por tenant/canal/operación sin tocar código: un `operation:sales.invoices.issue` con `deny_threshold=30` deniega aunque el global sea 80.
2. `allow_threshold` permite ALLOW rápido para señales de bajo riesgo sin evaluar authority/delegation.
3. Grants de delegación con `expiresAt` expiran automáticamente: `listDelegations` los omite y el stage los salta por defensa en profundidad.
4. `revokeAllDelegations` limpia grants masivamente con un solo consistency bump (`delegation.revoke_all`).
5. `flushMetrics` da observabilidad del estado de consistencia sin mutar nada; `--dry-run` en invalidate permite auditar antes de ejecutar.
6. CLI `authz:adaptive:tune` inspecciona y ajusta thresholds en runtime (modo `--apply` requiere flag `adaptive.tune.enabled`).
7. Todo opt-in: `adaptive_access.enabled=false`, `adaptive.tune.enabled=false`, `delegation.default_ttl_seconds=null`. Cero breaking changes.

**Validacion ejecutada**

- `.\vendor\bin\phpunit.bat --filter=Authorization` → **218 tests, 733 assertions, exit 0** (1 risky test preexistente `ExceptionHandlingTest` no relacionado).

**Siguiente corte recomendado: DV-AUTHZ-010K = Shadow delegation grants + revocation policy engine + selective flush cross-worker**

## Regla de mantenimiento

Cada vez que se cierre una nueva iteracion del subsistema Authorization, esta bitacora debe registrar:

1. identificador `DV-AUTHZ-00X`,
2. documentos impactados,
3. alcance real,
4. evidencia,
5. resultado operativo,
6. y siguiente gap natural.

No dejar esta actualizacion para una fase posterior.
