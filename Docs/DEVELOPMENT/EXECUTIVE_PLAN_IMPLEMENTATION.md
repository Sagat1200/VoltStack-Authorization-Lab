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
- `AuthorizationDecisionPlan` explainable por stages,
- `AuthorizationMetadataResolver` con projection a `Quantum/Metadata`,
- `AuthorizationMetadataPayload` normalizado con fingerprint estable,
- `AuthorizationManifestStoreInterface` con stores InMemory y Filesystem,
- configuracion `authorization.manifest.enabled` y `authorization.manifest.path`,
- `ManifestRequirementsEnforcementStage` como enforcement temprano,
- `AttributeConditionEvaluator` para ABAC runtime declarativo,
- `AuthorityMemoizationCacheInterface`, `RequestScopedAuthorityMemoizationCache` y `CachedAuthorityRepository`,
- `DatabaseAuthorityRepository`,
- fingerprint visible en `DecisionResult::metadataFingerprint()`,
- early-gate authority opt-in en `AuthorizationManager`,
- `TenantScopeResolverInterface` y `TenantScopeResolver` para proyeccion automática de scope (opt-in),
- `ControllerEngine` proyectando tenant HTTP real al contexto de Authorization antes del authorize(),
- `TenantScopeResolver` leyendo señales runtime (`Request`, `RouteMatch`, `controller.security.context`, route params`) para derivar tenant/scope en usos directos del planner/manager,
- `RelationshipRepositoryInterface`, `InMemoryRelationshipRepository`, `DatabaseRelationshipRepository` y `RelationshipEvaluator` para una primera capa ReBAC opt-in con backend persistente inicial,
- metadata declarativa `relation` en `Authorize`, `Route::authorize()` y `Route::authorizeRelated()`,
- atributos `#[AuthorizeWhen]` y DSL `Route::authorizeWhen()/authorizeWhenAll()`,
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
- la integracion declarativa ya es explainable y tiene ABAC runtime utilizable,
- y el siguiente movimiento debe profundizar invalidacion distribuida, providers externos y cache distribuida.

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
- Fase 6: completada en version V1+ consolidada
- Fase 7: completada (DV-AUTHZ-010A/B/C/D/E/F/G/H — drivers DB, tooling, tenancy, ReBAC DBAL, adaptive access base, drivers remote cache, consistency distribuido con envelope audit compartido)
- Fase 8: completada (DV-AUTHZ-010I — Delegation/Impersonation + S2S Principals)
- Fase 9: completada (DV-AUTHZ-010J — Adaptive Access tenant/canal/operación + Delegation TTL + Selective Flush)

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

## Estado actual - DV-AUTHZ-007 / DV-AUTHZ-009

**Material nuevo incorporado en runtime:**

1. `AuthorizationDecisionPlan` explainable por stages + `AuthorizationManager::explain()/explainPlan()`.
2. Memoization request-scoped de `effectivePermissionsForPrincipal()` vía `AuthorityMemoizationCacheInterface`, `RequestScopedAuthorityMemoizationCache` y `CachedAuthorityRepository`.
3. Early-gate authority opt-in en `AuthorizationManager` (`authorization.authority.early_gate_enabled=false` por defecto).
4. `DatabaseAuthorityRepository` y wiring por `authorization.authority.driver=memory|database|db|dbal`.
5. `AttributeConditionEvaluator` y `ManifestRequirementsEnforcementStage` con evaluación ABAC runtime bajo `evaluate_attribute_conditions=false` por defecto.
6. Metadata declarativa ampliada con `#[AuthorizeWhen]`, `Route::authorizeWhen()`, `Route::authorizeWhenAll()` y DSL runtime `Condition::*`.
7. Propagación estable de `condition` por metadata resolver, payload factory, enricher y manifest store.
8. Primera capa de tenant/scope resolver automático (opt-in) ya integrada con `AuthorizationContextFactory`, `AuthorizationManager`, `ManifestRequirementsEnforcementStage` y `ControllerEngine`.
9. El inner authority repository se ajustó a ciclo `scoped` para convivir correctamente con `DatabaseInterface` y memoization request-scoped.
10. Primera capa ReBAC opt-in ya integrada con `RelationshipRepositoryInterface`, `InMemoryRelationshipRepository`, `DatabaseRelationshipRepository`, `RelationshipEvaluator`, metadata `relation`, `Route::authorizeRelated()` y enforcement runtime en `ManifestRequirementsEnforcementStage`.
11. Regresión focalizada actual del corte de consistencia inicial: consistency+ReBAC+multi-surface **77 tests / 234 assertions exit 0** (2 deprecations no bloqueantes).

## Corte ejecutado

### DV-AUTHZ-010D

`Consistencia E Invalidacion Generacional Inicial`

Alcance sugerido:

- introducir `AuthorizationConsistencyInterface` como contrato de versionado del subsistema,
- versionar la memoization de authority sin romper el cache request-scoped existente,
- propagar invalidación desde revocaciones relacionales,
- abrir una surface operativa CLI para invalidar generaciones manualmente.

Estado del corte:
- planner, authority, explainability, memoization, ABAC declarativo, tenancy cross-surface inicial y ReBAC opt-in con driver persistente inicial y tooling operativo básico ya están operativos,
- se añadió `AuthorizationConsistencyInterface` con implementación `VersionedAuthorizationConsistency` sobre `VersionAuthorityInterface`,
- la memoization authority ya es sensible a generaciones (`global`, `principal`, `scope`, `principal_scope`),
- y el siguiente cuello de botella ya no está en el planner sino en conectar esa consistencia a un backend multi-worker real.

Entregables minimos:

1. contrato de consistencia para authority y relationships,
2. claves de memoization sensibles a generación,
3. invalidación desde revocaciones relacionales,
4. command CLI de invalidación,
5. documentación DEVELOPMENT sincronizada.

Resultado esperado:

- **Cierre DV-AUTHZ-010D** (consistencia e invalidación generacional inicial),
- suite del subsistema ampliada sobre la base actual sin romper el comportamiento opt-in.

## Fase 8 - Delegation + Service Principals (010I)

### Objetivo

Introducir una capa opt-in completa de Delegation / Impersonation + Service-to-Service Principals, alineada con `Doc 20 - Authorization Delegation Impersonation Capabilities And Service To Service System`.

### Entregables

1. **Contracts y VOs de Delegation:**
   - `DelegationAdministrationInterface` (`listDelegations`, `grantDelegation`, `revokeDelegation`)
   - `ServicePrincipalResolverInterface` (`resolve(?RuntimeContext, ?Request): ?PrincipalInterface`)
   - `DelegationGrant` VO readonly con shape `{trustee_id, grantor_id, scope, grant_type, grant_value, granted_at}`.
2. **Impersonation runtime + 3er param BoundAuthorization opcional:**
   - `AuthorizationManagerInterface::impersonate(caller, target, ?Scope)` como nuevo helper (sin romper API legacy),
   - `ImpersonationPrincipalBuilder` con duck-typing `getId()` / `id` property / `PrincipalInterface` / string/int,
   - `BoundAuthorization` tercer parámetro opcional `?AuthorizationContext $context = null` y helper `mergeContext(A?, A?)` aplicado en los 4 métodos `check/cannot/decide/authorize` (backward compat 100%).
3. **Delegation administration en repos InMemory y Database:**
   - `InMemoryAuthorityRepository` y `DatabaseAuthorityRepository` implementan `DelegationAdministrationInterface`,
   - key única 5-column `trustee#grantor#scope#type#value`,
   - `DatabaseAuthorityRepository` tabla configurable `authorization.tables.delegation_grants` default `authorization_delegation_grants`.
4. **Fallback delegation semántico en Manifest stage:**
   - Nuevos params `ManifestRequirementsEnforcementStage::__construct(?DelegationAdministrationInterface, bool evaluateDelegations=false)`,
   - Helper `impersonationContext()` detecta Principal tipo `ImpersonatedUser` y lee `authorization.impersonation.originator_id/target_id`,
   - **Orden fail-closed estricto:**
     1. hasPermission directo con principal target → si TRUE, ALLOW sin delegation.
     2. Solo si está en impersonation Y el target NO tenía permiso → check delegation:
        - listar grants trustee↔grantor en scope
        - match explícito por nombre de permission
        - **REGLA SEMÁNTICA PRINCIPAL:** authority lookup `hasPermission($grantorId, $permission, $scope)` si grantor lo tiene → ALLOW con metadata delegation
        - último fallback role expansion (solo si Role ctor trae permisos).
   - Metadata inyectada en ALLOW: `delegation_granted, delegation_trustee_id, delegation_grantor_id, originator_principal_id, target_principal_id, impersonation_scope`.
5. **Service Principal resolver + wiring fail-closed:**
   - `ConfigurableServicePrincipalResolver` orden resolución: runtime `as/as_service` flag → req attr `service_principal.as` → query param `as-service` → SERVER `VOLT_AS_SERVICE` → runtime `service_principal.id/type/claims` → config map `authorization.service_principals.map.<id>`. Final claims: `config ∪ runtime` (runtime wins).
   - `PrincipalResolver` nuevos params: `?ServicePrincipalResolverInterface`, `enabled=false` (default off = fail-closed). Antes del bloque Anonymous: si enabled + resolver not null → intenta resolve; catch todo → falla cerrada.
6. **DelegationContextEnricher:** proyecta `authorization.originator.* / target.* / impersonation.* / service.*` al pipeline de enrichers (solo si delegation o service_principal están habilitados).
7. **Tooling CLI:**
   - `authz:delegation:list|grant|revoke` (--trustee-id, --grantor-id, --scope, --role|--permission, --verbose, --dry-run, --require-published-config pattern standard)
   - `authz:authority:list --view=simple|delegations|all --grantor-id` (simple por defecto)
   - `authz:consistency:report` / `authz:consistency:doctor` → JSON y humano ahora incluyen `delegation_bumps` y `service_principal_bumps` (contando reasons `delegation.*` y `service.*`)
   - Mutaciones administrativas delegation: `invalidateAuthority(..., reason='delegation.grant' | 'delegation.revoke')` (consistency bump tracking).
8. **ServiceProvider wiring (todo opt-in default off):**
   - defaults fusiona: `delegation.enabled=false`, `service_principal_resolver.enabled=false`, `service_principals.map=[]`, `authority.evaluate_delegations=false`.
   - `registerDelegationAndServicePrincipalBindings()` condicional si `delegation.enabled || service_principal_resolver.enabled` (OR).
   - Binding `DelegationAdministrationInterface` al repositorio **inner authority** (INNER_AUTHORITY_REPOSITORY, no el wrapper memoized cached).
   - Binding `ServicePrincipalResolverInterface` singleton a `ConfigurableServicePrincipalResolver(map)`.
   - `Manifest stage`: `evaluateDelegations = explicit_flag || delegation.enabled` (inference rule: si delegation on, evaluate on).
   - `AuthorizationPlanner`: enrichers condicionalmente agrega `DelegationContextEnricher::class` al final (try/catch safe).
   - **Stage order preservado estrictamente:** `[AdaptiveAccessStage, ManifestRequirementsEnforcementStage, GateAuthorizationStage, PolicyAuthorizationStage]` (4 stages = backward compat 100%).
   - `commands()`: 13 comandos (10 anteriores + DelegationList/Grant/Revoke).

### Criterio de cierre

1. `--filter=Authorization` exit=0,
2. 8 archivos tests nuevos (Contracts, Impersonation, Runtime con fallback delegation semántico, Consistency bumps delegation/service, ServicePrincipal resolver en runtime y request, CLI commands 7 tests, Manager impersonate + S2S, Integration end-to-end) todos pasando,
3. contracts `DelegationAdministrationInterface` y `ServicePrincipalResolverInterface` resolveables desde el container cuando los flags están ON,
4. metadata de ALLOW bajo impersonation + delegation incluye `delegation_granted=true` y trazabilidad `trustee_id`/`grantor_id`,
5. Docs DEVELOPMENT sincronizados con el bloque entregado.

### Evidencia

- exit=0 de `phpunit --filter=Authorization` con **191 tests / 653 assertions en verde** (1 risky test pre-existente de ExceptionHandling sin relación al bloque).
- 8 archivos tests nuevos Task 9 del spec todos en verde.
- 4 archivos DEVELOPMENT sincronizados (Versions, Matrix, Guidelines, Executive Plan).

## Siguiente corte recomendado

### DV-AUTHZ-010J

`Políticas adaptativas tenant/canal/operación + AdaptiveAccessStage rico`

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

### DV-AUTHZ-010J

Alcance sugerido:

- ampliar `AdaptiveAccessStage` con políticas adaptativas parametrizadas por tenant/canal/operación: step-up auth explícito, deny por threshold de riesgo por principal/recurso/hora, rate-limit granular por principal y combinación tenant×action,
- uniformar tenancy automática + service-to-service en superficies CLI/Jobs/Workers sin HTTP RouteMatch,
- añadir shadow-admin delegation grants temporales con expiry/ttl y revoked-at programático,
- selective flush distribuido observable por principal/scope sobre el backend cache compartido real con métricas de hit/miss cross-worker,
- comandos doctor de adaptive tuning, métricas de evaluación por stage, overrides por tenant y tiempos de evaluación.

Estado del corte:

- `DV-AUTHZ-010I` (Delegation/Impersonation + S2S Principals) **CERRADO** → Doc 20 Operativo en DEVELOPMENT_MATRIX,
- planner, authority, explainability, memoization versionada distribuida, ReBAC DBAL y remote-cache, ABAC declarativo, tenancy cross-surface inicial, tooling CLI administrativo completo (manifest/authority/relationships/delegation/consistency report+doctor), consistency distribuido con envelope audit compartido file/cache, Delegation + impersonation + S2S Principals ya están operativos todo opt-in con defaults off y 191 tests 653 assertions exit 0 en el corte 010I,
- siguiente foco natural = ampliar AdaptiveAccessStage hacia políticas adaptativas ricas por tenant/canal/operación y endurecer tenancy en superficies sin HTTP.

Entregables minimos:

1. `AdaptiveAccessStage` con shape declarativo configurable: thresholds por tenant/canal/operación, stepUp/challenge/deny por combinado,
2. uniformar tenant resolver automático uniforme en CLI/Jobs sin HTTP,
3. delegation grants con expiry/ttl y workflow de revocación shadow-admin temporal,
4. selective flush distribuido + métricas cross-worker sobre store cache,
5. comandos adaptive-access:doctor y adaptive-access:tuning.

Resultado esperado:

- VoltStack Quantum/Authorization pasa de una base de motor + delegation + s2s a un motor adaptive-access completo con políticas adaptativas tenant-aware sin romper APIs públicas y manteniendo defaults off-by-default.
