# EXECUTIVE_PLAN_IMPLEMENTATION

## Proposito

Este documento traduce la arquitectura del sistema Authorization a un plan ejecutivo de implementacion para el paquete:

- `vendor/voltstack/framework/src/Quantum/Authorization`

Su objetivo es convertir la documentacion `00-32` en una secuencia de desarrollo incremental, realista y compatible con el estado actual del framework.

## Estado actual del plan

Hoy el plan ya no parte desde un namespace vacio.

### 1. El namespace objetivo ya cuenta con una base operativa

En `vendor/voltstack/framework/src/Quantum/Authorization` ya existen:

- core manager,
- request/context model,
- decision model,
- gates y abilities,
- policy registry/dispatcher declarativo inicial,
- atributos y contracts de policies,
- mapper de errores del modulo,
- provider y bootstrap base,
- planner formal con enrichers y stages,
- `AuthorizationMetadataResolver` con projection a `Quantum/Metadata`,
- `AuthorizationMetadataPayload` normalizado con fingerprint estable,
- `AuthorizationManifestStoreInterface` con stores InMemory y Filesystem,
- configuracion `authorization.manifest.enabled` y `authorization.manifest.path`,
- `ManifestRequirementsEnforcementStage` como enforcement temprano,
- fingerprint visible en `DecisionResult::metadataFingerprint()`,
- y commands CLI `authz:manifest:compile` + `authz:manifest:clear` registrados via `commands()` del provider.

### 2. Siguen existiendo piezas reutilizables de alto valor

El framework si dispone de infraestructura cercana que debe usarse como base y no como competencia:

- `Quantum/Controllers/Security`
- `Quantum/Metadata`
- `Quantum/Auth`
- bindings y wiring en `Platform/Application.php`

### 3. Ya existen pruebas que validan ideas utiles para Authorization

- composicion de policies,
- fail-closed,
- `default deny`,
- challenge,
- worker safety,
- decision cache,
- recursion guard,
- timeout y circuit breaker.

Conclusión:

- el trabajo fundacional ya quedo aterrizado,
- la integracion declarativa inicial ya esta abierta,
- y el siguiente movimiento debe consolidar planner, metadata compilable y cierre de V1 conectada.

## Objetivo del primer cierre real

El primer cierre util del subsistema `Quantum/Authorization` no debe intentar cubrir toda la plataforma definida en `Docs/00-32`.

El objetivo inmediato debe ser entregar un stack minimo, seguro y usable con estas capacidades:

1. principal explicito o resuelto desde contexto,
2. ability normalizada,
3. subject descriptor,
4. authorization request inmutable,
5. decision model tipado,
6. `AuthorizationManager` con `check()`, `decide()` y `authorize()`,
7. primer `PolicyRegistry` y `GateRegistry`,
8. bootstrap minimo del modulo,
9. comportamiento `default deny` y fail-closed,
10. aislamiento seguro para runtime persistente.

## Regla de alcance

No abrir en la primera fase:

- approval workflows,
- SoD,
- ReBAC completo,
- delegation completa,
- impersonation completa,
- capabilities,
- risk engine,
- tooling administrativo,
- coordinacion distribuida avanzada.

Esos bloques dependen de un core que todavia no existe.

## V1 operativa minima objetivo

La primera version operativa de `Quantum/Authorization` deberia quedar compuesta por estos bloques:

```text
Facade / Helper / Traits
        |
        v
AuthorizationManager
        |
        v
AuthorizationRequestFactory
        |
        +--> Principal Resolver
        +--> Ability Normalizer
        +--> Subject Resolver
        +--> Context Factory
        +--> Authorization Planner
        +--> Policy / Gate Executor
        +--> Decision Manager
        +--> Result Finalizer
```

## Estructura minima recomendada de namespaces

```text
Quantum/Authorization
    Contracts/
    Core/
    Ability/
    Principal/
    Subject/
    Context/
    Decision/
    Policy/
    Gate/
    Bootstrap/
    Exceptions/
    Testing/
```

## Layout inicial sugerido

```text
Quantum/Authorization
    Contracts/
        AuthorizationManagerInterface.php
        PrincipalResolverInterface.php
        AbilityNormalizerInterface.php
        SubjectResolverInterface.php
        AuthorizationContextFactoryInterface.php
        AuthorizationPlannerInterface.php
    Core/
        AuthorizationManager.php
        AuthorizationRequest.php
        AuthorizationRequestFactory.php
        BoundAuthorization.php
    Ability/
        Ability.php
        AbilityRegistry.php
    Principal/
        PrincipalInterface.php
        AnonymousPrincipal.php
    Subject/
        SubjectDescriptor.php
        SubjectType.php
    Context/
        AuthorizationContext.php
    Decision/
        Decision.php
        DecisionResult.php
        DecisionManager.php
    Policy/
        PolicyRegistry.php
        PolicyDispatcher.php
    Gate/
        GateRegistry.php
    Bootstrap/
        AuthorizationServiceProvider.php
    Exceptions/
        AuthorizationException.php
        AuthorizationDeniedException.php
```

## Estado resumido de fases

- Fase 0: completada
- Fase 1: completada en version minima
- Fase 2: completada en version minima
- Fase 3: completada en version minima manual
- Fase 4: completada en version minima
- Fase 5: completada en version inicial conectada
- Fase 6: siguiente foco ejecutivo

## Fases ejecutivas

## Fase 0 - Fundacion del modulo

### Objetivo

Crear el namespace real del subsistema y fijar los contratos que impiden deriva arquitectonica temprana.

### Entregables

1. crear `Quantum/Authorization`,
2. introducir contratos base del manager y resolvers,
3. introducir layout minimo del modulo,
4. registrar `AuthorizationServiceProvider` vacio o muy fino si hace falta bootstrap temprano.

### Criterio de cierre

- el modulo existe,
- compila,
- y el framework puede resolver el provider.

## Fase 1 - Modelo canonico del request de autorizacion

### Objetivo

Construir el lenguaje minimo del sistema.

### Entregables

1. `Ability`
2. `PrincipalInterface`
3. `AnonymousPrincipal`
4. `SubjectDescriptor`
5. `AuthorizationContext`
6. `AuthorizationRequest`
7. `Decision`
8. `DecisionResult`

### Reglas

- `AuthorizationRequest` y `DecisionResult` deben ser inmutables,
- `AuthorizationContext` no debe ser service locator,
- el modelo no debe asumir `User` ni ORM.

### Tests minimos

1. value objects,
2. igualdad semantica basica,
3. inmutabilidad,
4. `ABSTAIN` nunca se interpreta como `ALLOW`.

## Fase 2 - Core engine minimo

### Objetivo

Crear el primer motor real del sistema.

### Entregables

1. `AuthorizationManagerInterface`
2. `AuthorizationManager`
3. `AuthorizationRequestFactory`
4. `BoundAuthorization`
5. primera estrategia `default deny`
6. excepciones minimas del modulo

### Operaciones minimas

- `check()`
- `cannot()`
- `decide()`
- `authorize()`

### Reglas

- todas las APIs convergen en `decide()`,
- fail-closed ante fallos internos,
- cero dependencia directa de HTTP.

### Tests minimos

1. `check()` y `cannot()` reflejan la misma decision,
2. `authorize()` lanza excepcion ante `DENY`,
3. ausencia de evaluadores aplicables produce denegacion,
4. error inesperado no produce `ALLOW`.

## Fase 3 - Policies, gates y dispatcher minimo

### Objetivo

Entregar el primer sistema reusable de evaluacion.

### Entregables

1. `PolicyRegistry`
2. `PolicyDispatcher`
3. `GateRegistry`
4. `DecisionManager`
5. primera semantica de `AbilityRegistry`

### Alcance minimo recomendado

- gates sin subject,
- policies sobre object/class/no-subject,
- normalizacion de bool a `DecisionResult`,
- integracion con `AuthorizationManager`.

### Reutilizacion recomendada

Tomar como referencias directas:

- `ControllerSecurityPolicy`
- `ControllerSecurityPolicyRegistry`
- `ControllerSecurityDecisionEngine`
- pruebas de `PolicyCompositionTest.php`

### Tests minimos

1. gate concede o deniega,
2. policy recibe principal y subject correcto,
3. `true/false` se normaliza correctamente,
4. multiples evaluadores respetan `default deny`.

## Fase 4 - Bootstrap y contexto de framework

### Objetivo

Conectar el modulo al runtime sin introducir fugas entre requests.

### Entregables

1. `AuthorizationServiceProvider`
2. configuracion `config/authorization.php`
3. principal resolver basado en `Quantum/Auth`
4. context factory minima
5. wiring en `Application.php`

### Reglas

- manager preferentemente stateless,
- contexto request-scoped,
- nada de principal/tenant/decision en singletons compartidos.

### Tests minimos

1. bindings del container,
2. principal anonimo cuando no hay auth,
3. principal autenticado cuando existe `AuthenticationContext`,
4. aislamiento entre requests consecutivos.

## Fase 5 - Integracion inicial con controllers y metadata

### Objetivo

Conectar el nuevo core al framework sin reescribir de golpe `Controllers/Security`.

### Entregables

1. adapter entre metadata declarativa y `AuthorizationRequest`,
2. primera compatibilidad con atributos/routing,
3. mapping inicial de errores del modulo,
4. estrategia de convivencia temporal con `Controllers/Security`.

### Regla critica

No reemplazar `Controllers/Security` con una migracion abrupta.

Primero:

- crear el core,
- luego adaptar,
- luego extraer o redirigir piezas.

### Tests minimos

1. metadata declarativa produce request correcto,
2. denegacion evita la ejecucion,
3. error del modulo se representa sin acoplar el core a HTTP,
4. no hay reevaluacion incoherente entre requests.

### Estado actual

- completada en una primera version usable:
  - existen `#[Authorize]`, `#[PublicAccess]`, `Route::authorize()` y `Route::publicAccess()`,
  - `ControllerEngine` ya proyecta metadata declarativa al `AuthorizationManager`,
  - y `AuthorizationExceptionMapper` ya representa denegaciones y challenges en HTTP.

Adicionalmente, el modulo ya abrio una base de planner formal:

- `AuthorizationManager` ya delega la evaluacion a `AuthorizationPlanner`,
- el planner ya compone stages explicitos para gates y policies,
- la metadata declarativa de Authorization ya se proyecta sobre `Quantum/Metadata`,
- el modulo ya dispone de un `AuthorizationMetadataResolver` reusable,
- el planner ya soporta enrichment contextual previo a sus stages,
- la metadata normalizada ya dispone de un payload con fingerprint estable,
- `DecisionManager` ya aplica `default_strategy`,
- y el motor ya diferencia `fail_closed` de `fail_open` ante fallos de evaluadores.

Adicionalmente, el corte DV-AUTHZ-005 ya materializo el siguiente nivel de madurez:

- `ManifestRequirementsEnforcementStage` ejecuta enforcement temprano directamente sobre metadata del manifest (public→ALLOW, ability no declarada→DENY/ABSTAIN),
- todos los `DecisionResult` del planner ya llevan `metadataFingerprint()` visible cuando el contexto trae fingerprint del manifest,
- commands `authz:manifest:compile` y `authz:manifest:clear` quedan descubiertos automaticamente via `AuthorizationServiceProvider::commands()`,
- y el pipeline de stages se ordena `[manifest_requirements, gates, policies]` para maximizar early returns.

### Estado actual - DV-AUTHZ-006 (CERRADO al 100%)

**Material nuevo incorporado en runtime:**

1. `Value Objects` RBAC/ABAC en `Quantum/Authorization/Authority/*`:
   - `Permission` single-string name + wildcard matching (`admin:*` matches `admin:read`),
   - `Role` named bag con deduplicacion de permisos por nombre,
   - `Scope` jerárquico `org:ws:proj` con `contains()`, `parent()`, wildcard `*`, global scope,
   - `AttributeDefinition` typed ABAC schema (bool/string/int/float/array/enum/any) con constraints pattern/min-max/enum/required/default.
2. `AuthorityRepositoryInterface` + `InMemoryAuthorityRepository`:
   - seed desde config `authorization.authority.grants` shape `{principal_id, scope, roles, permissions}`,
   - herencia upward por scope (while loop parent()) para colectar grants ascendentes,
   - métodos `hasPermission`, `hasRole`, `effectivePermissionsForPrincipal`, `effectiveRolesForPrincipal`, `attributesForPrincipal`.
3. `ManifestRequirementsEnforcementStage` ampliado:
   - constructor backward-compatible: nuevos params `authorityRepository=null`, `evaluateRequirementsConcretely=false` POR DEFECTO,
   - `evaluateRequirementsConcretely=false` → whitelist filter legacy (downstream gates/policies) sin regression V1,
   - `evaluateRequirementsConcretely=true` (opt-in) → `normalizedMatchedRequirements` + `evaluateConcretelyEachMatchedRequirement()` con `effect=deny` gana siempre sobre grants,
   - reason codes nuevos: `manifest_requirement_granted_by_authority`, `manifest_requirement_not_granted_by_authority`, `manifest_requirement_explicit_deny`.
4. Wiring provider:
   - defaults `authorization.authority.enabled=true`, `evaluate_requirements_concretely=false` (opt-in), `grants=[]`,
   - `registerAuthorityRepository()` singleton condicional sobre enabled,
   - Manifest stage wiring pasa 3 params con fallback correcto.
5. RouteDefinition + CompileCommand fixes:
   - CompileCommand usa getter `$route->definition()->action()` NO propiedad privada (fix de acceso prohibido en tests),
   - `formatActionForOutput(?ControllerDefinition): string` nuevo helper normaliza action callable `[Class,method]` a `Class::method` para sprintf sin warnings.
6. Commands CLI `authz:manifest:*` tests completados:
   - `AuthorizationManifestCommandsTest` 12 tests (metadata command name/category/aliases, empty routes 0, persist metadata con fingerprint, `--dry-run` no llama store.put, `--verbose` imprime fp+requirements, skip routes con throw/no-fp, clear entries count=7, clear empty=0, clear dry-run no-op, clear verbose reporta store, clear exception exit 1),
   - Spies Store/Resolver anonymous classes implementando `AuthorizationManifestStoreInterface` / `AuthorizationMetadataResolverInterface`,
   - Workaround `final Command`: bootstrap temporal `basePath/bootstrap/app.php` return `$GLOBALS['__volt_authz_test_app']`,
   - ReflectionProperty leer buffers privados `Output::stdoutBuffer/stderrBuffer`.
7. Bridge mínimo Security ↔ Planner: `Quantum/Authorization/Bridges/ControllerSecurityPlannerBridge`:
   - `tryEvaluate(SecurityEvaluationRequest): ?SecurityDecision` retorna `null` si no hay `authorization_requirements` ni `permissions` en metadata → HardenedEngine continua intacto,
   - Mapeo metadata `[authorization_requirements.{ability,effect=allow|deny}]` o fallback `permissions[]`,
   - Mapea `SecurityPrincipal` (Controllers/Security) → `Quantum/Authorization/Principal` con `mapSecurityPrincipalTypeToAuthorizationType()` enum-compatible,
   - Invoca `AuthorizationManager::decide()` y normaliza `DecisionResult` → `SecurityDecision` (Allow/Deny/Abstain/Challenge) con obligations `{requirements, fingerprint}` bajo obligaciones.
8. Tests nuevos totales (ciclo actual + parcial anterior 006):
   - `AuthorityModelAndRepositoryTest` 9 tests (VO + InMemory + herencia scope),
   - `ManifestRequirementsEnforcementStageTest` 4 tests concretos + 8 legacy = 12,
   - `AuthorizationMultiSurfaceIntegrationTest` 5 tests (CLI surface gate, job exception, authority seed directo, helper, authority disabled),
   - `AuthorizationManifestCommandsTest` 12 tests (commands CLI),
   - `ControllerSecurityPlannerBridgeTest` 5 tests (convergencia Security ↔ Planner),
   - **SUITE COMPLETA ACUMULADA:** 107 Unit tests / 393 assertions → exit 0 + 74 Feature Authorization/Security tests / 901 assertions → exit 0 salvo 1 error pre-existente `AuthManager::password_expired` no relacionado.

## Siguiente corte recomendado

### DV-AUTHZ-007

`Explainability, Memoization De Permisos E Integración Authority ↔ Manager`

Alcance sugerido:

- método `AuthorizationDecisionPlan::explain(): array` con árbol por stage (name, decision, reasonCode, metadata, requirements matched),
- cache memoization scoped-request `effectivePermissionsForPrincipal` por clave `(principalId, scope, tenantId)` con invalidación por put seed nuevo o invalidate(),
- integración `AuthorizationManager::check()` + `authorize()` con opt-in early-gate `authorization.authority.enabled` que evalúa grants/denials explícitos antes del planner completo,
- integración wiring `ControllerSecurityPlannerBridge` en `AuthorizationServiceProvider` bajo flag opcional `authorization.security_bridge.enabled=false` (default off por compatibilidad),
- ABAC runtime evaluador condicional sobre `AttributeDefinition` constraints pattern/min-max/enum/required,
- tests unitarios de explain + memoization + early-gate authority + wiring bridge + ABAC runtime (15-20 tests).

Estado del corte:

- V1 conectada (005) + Authority RBAC/Scope (006 cerrado 100%) + CommandsOperative + Convergencia SecurityBridge ya dan un subsistema usable,
- faltan explainability/trazabilidad + memoization performance + early-gate en AuthorizationManager para cerrar V1+,
- por lo que el siguiente trabajo debe abrir esos gaps y consolidar la integración de Authority con AuthorizationManager.

Entregables minimos:

1. método `explain()` en el plan final (o `AuthorizationPlanner`) con árbol stages + decision parciales,
2. memoization `effectivePermissionsForPrincipal()` en InMemoryAuthorityRepository con clave tupla + TTL scoped-request + invalidate/clear API mínima,
3. integración `AuthorizationManager::check/authorize` early-gate contra `AuthorityRepository` cuando `authorization.authority.enabled=true` y `authorization.authority.early_gate_enabled=true`,
4. wiring provider del `ControllerSecurityPlannerBridge` con binding singleton + flag config off-by-default,
5. tests 15+ (explain tree, memoization cache hit/miss, early-gate allow/deny, bridge wiring desactivado/activado, ABAC constraints eval).

Resultado esperado:

- **Cierre parcial DV-AUTHZ-007** (explain + memo + early-gate + wiring bridge + ABAC runtime),
- Suite total: 140+ tests / 550+ assertions exit 0.

### Criterio de cierre de V1+ (post-DV-AUTHZ-007)

Se puede considerar cerrada la versión consolidada cuando exista:

1. ability + principal + subject + context,
2. decision model tipado,
3. `check()` y `authorize()` reales con early-gate authority opt-in,
4. policy/gate system mínimo,
5. bootstrap en el framework,
6. request isolation compatible con FrankenPHP,
7. pruebas unitarias e integracion completas (140+ tests),
8. DecisionPlan::explain() trazabilidad por stage,
9. memoization effectivePermissions con TTL scoped-request,
10. SecurityBridge planner wiring off-by-default.

## Mapa de clases prioritarias

## Prioridad P0

1. `Ability`
2. `PrincipalInterface`
3. `AnonymousPrincipal`
4. `SubjectDescriptor`
5. `AuthorizationContext`
6. `AuthorizationRequest`
7. `Decision`
8. `DecisionResult`
9. `AuthorizationManagerInterface`
10. `AuthorizationManager`

## Prioridad P1

1. `AuthorizationRequestFactory`
2. `PolicyRegistry`
3. `PolicyDispatcher`
4. `GateRegistry`
5. `DecisionManager`
6. `AuthorizationServiceProvider`
7. `ManifestRequirementsEnforcementStage`
8. `AuthorizationManifestStoreInterface` y stores
9. `AuthorizationManifestCompileCommand` y `AuthorizationManifestClearCommand`

## Prioridad P2

1. auth principal resolver,
2. metadata adapter,
3. exception mapper,
4. request-scope/reset tests,
5. traits o facade ergonomicos.

## Integraciones del framework que probablemente deben tocarse

1. `vendor/voltstack/framework/src/Platform/Application.php`
2. `vendor/voltstack/framework/src/Helper/helpers.php`
3. `vendor/voltstack/framework/src/Quantum/Metadata/*`
4. `vendor/voltstack/framework/src/Quantum/Controllers/Security/*`
5. `config/authorization.php`
6. `vendor/voltstack/framework/tests/Unit/*Authorization*`

## Riesgos a vigilar

### 1. Copiar Controllers Security en vez de extraer un core general

Ese camino aceleraria el arranque, pero dejaria dos motores semanticos distintos dentro del framework.

### 2. Crear una arquitectura enorme sin bloque operable

Si se crean demasiadas clases vacias antes de tener request, decision y manager reales, el modulo se volvera ceremonial y no ejecutable.

### 3. Acoplar demasiado pronto a controllers

El primer cierre debe seguir siendo un engine reusable por:

- controllers,
- jobs,
- CLI,
- workers,
- y servicios internos.

### 4. Olvidar runtime persistente

Authorization debe nacer ya con aislamiento de request, porque luego corregir fugas de estado es mucho mas costoso.

## Definition of Done por fase

Una fase se considera realmente cerrada solo si:

1. existe codigo fuente identificable en `Quantum/Authorization`,
2. existe al menos un test representativo,
3. existe integracion real o punto de uso verificable,
4. no introduce estado global mutable,
5. se actualizan los artefactos en `Docs/DEVELOPMENT`.

## Siguiente corte recomendado

### DV-AUTHZ-006

Alcance sugerido:

- introducir modelos concretos de `Role`, `Permission`, `Scope` y repositorio `AuthorityRepositoryInterface`,
- extender el manifest stage para evaluar CADA requirement concreto contra gate/policy y authority repository,
- tests especificos de commands CLI del manifest,
- escenarios multi-surface (CLI/Jobs/Workers) sin HTTP RouteMatch,
- convergencia entre `HardenedControllerSecurityDecisionEngine` de Controllers/Security y `Quantum/Authorization` planner,
- versionado de scopes jerarquicos `organization > workspace > project`.

Estado del corte:

- el planner formal ya se compone de `manifest_requirements → gates → policies`,
- ya existe fingerprint estable, manifest store persistente y commands de compilation/clearing,
- por lo que el siguiente trabajo debe aterrizar modelos avanzados de autoridad y enforcement semantico real de requirements.

Entregables minimos:

1. `Role`, `Permission`, `Scope` como conceptos de primer nivel del modulo,
2. `AuthorityRepositoryInterface` para grants por principal/tenant,
3. `ManifestRequirementsEnforcementStage` evaluando requirements concretos (no solo whitelist filter),
4. tests especificos de commands CLI del manifest,
5. tests multi-surface sin HTTP RouteMatch,
6. convergencia con `HardenedControllerSecurityDecisionEngine`,
7. actualizacion de matriz y bitacora.

Resultado esperado:

- VoltStack pasa de una V1 conectada inicial a una V1 RBAC+ABAC real, con enforcement evaluable, trazable y usable en todas las superficies del framework.
