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

**Tipo:** Consolidacion de planner y estrategia
**Estado:** Abierto
**Objetivo:** Separar formalmente el planning de Authorization y empezar a hacer efectiva la estrategia configurable del motor.

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
5. ampliar la validacion unitaria y de regresion del subsistema.

**Evidencia**

- `vendor/voltstack/framework/src/Quantum/Authorization/Contracts/AuthorizationPlannerInterface.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/AuthorizationPlanner.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Core/AuthorizationManager.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/Decision/DecisionManager.php`
- `vendor/voltstack/framework/src/Quantum/Authorization/AuthorizationServiceProvider.php`
- `vendor/voltstack/framework/tests/Unit/AuthorizationManagerTest.php`

**Resultado operativo parcial**

- Authorization ya no agrega evaluadores inline dentro del manager,
- existe una primera separacion formal entre request normalization, planning y final decision,
- y la configuracion `authorization.default_strategy` / `authorization.fail_closed` ya altera el comportamiento real del engine.

**Validacion ejecutada**

- `vendor\bin\phpunit tests\Unit\AuthorizationManagerTest.php`
- `vendor\bin\phpunit tests\Unit\ControllerEngineTest.php tests\Unit\QuantumExceptionHandlerTest.php tests\Feature\AuthorizationIntegrationTest.php tests\Feature\ExceptionHandlingTest.php`

**Gap natural siguiente**

- enriquecer el planner con etapas/pipeline mas expresivo,
- acercar metadata declarativa a una capa compilable,
- y ampliar la observabilidad/trazabilidad de la evaluacion.

## Siguiente corte recomendado

### DV-AUTHZ-004-B

**Titulo sugerido:** `Planner, Metadata Compilable Y Cierre De V1 Conectada`

**Documentos fuente**

- `05_POLICY_REGISTRY_DISCOVERY_AND_RESOLUTION_SYSTEM.md`
- `09_AUTHORIZATION_PLANNER_AND_POLICY_PIPELINE_SYSTEM.md`
- `10_AUTHORIZATION_ATTRIBUTES_AND_DECLARATIVE_METADATA_SYSTEM.md`
- `17_AUTHORIZATION_TESTING_VERIFICATION_AND_SECURITY_ASSURANCE_SYSTEM.md`
- `18_AUTHORIZATION_COMPILATION_OPTIMIZATION_AND_RUNTIME_PERFORMANCE_SYSTEM.md`
- `32_AUTHORIZATION_SYSTEM_INTEGRATION_AND_FINAL_ARCHITECTURE.md`

**Alcance sugerido**

1. definir el primer planner/pipeline formal del modulo,
2. apoyar la metadata declarativa sobre una base compilable o manifestable,
3. ampliar la compatibilidad del engine con mas superficies del framework,
4. reforzar testing y trazabilidad de errores/decisiones,
5. acercar el cierre de una V1 conectada y explicable.

## Roadmap corto recomendado

### DV-AUTHZ-004

`Planner, Metadata Compilable Y Cierre De V1 Conectada`

Foco:

- `05`, `09`, `17`, `18`, `32`

### DV-AUTHZ-005

`Modelos Avanzados De Autoridad`

Foco:

- `12`, `13`, `14`, `19`, `20`, `23`, `27`, `31`

## Regla de mantenimiento

Cada vez que se cierre una nueva iteracion del subsistema Authorization, esta bitacora debe registrar:

1. identificador `DV-AUTHZ-00X`,
2. documentos impactados,
3. alcance real,
4. evidencia,
5. resultado operativo,
6. y siguiente gap natural.

No dejar esta actualizacion para una fase posterior.
