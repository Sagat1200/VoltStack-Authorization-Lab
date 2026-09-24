# DEVELOPMENT_GUIDELINES

## Proposito

Este documento define la guia operativa para iniciar el desarrollo del subsistema `Quantum/Authorization` de VoltStack usando como fuente principal la documentacion ubicada en `vendor/voltstack/authorization-lab/Docs`.

Su objetivo es mantener alineados:

- la arquitectura objetivo,
- el estado real del framework,
- la implementacion futura en `vendor/voltstack/framework/src/Quantum/Authorization`,
- y la trazabilidad de avance en `Docs/DEVELOPMENT`.

## Estado real del modulo

Al corte actual, `Quantum/Authorization` ya cuenta con un primer bloque funcional minimo que incluye:

- `Ability`, `AbilityRegistry` y `AbilityNormalizer`,
- `Principal`, `AnonymousPrincipal` y `PrincipalResolver`,
- `SubjectDescriptor` y `SubjectResolver`,
- `AuthorizationContext` y `AuthorizationContextFactory`,
- `AuthorizationRequest` y `AuthorizationRequestFactory`,
- `Decision`, `DecisionResult` y `DecisionManager`,
- `AuthorizationManager`,
- `GateRegistry`,
- `PolicyRegistry` y `PolicyDispatcher`,
- `AuthorizationServiceProvider`,
- facade `Quantum\Facades\Authorization`,
- helpers globales `authorization()`, `can()`, `cannot()` y `authorize()`.

Ademas, el framework ya dispone de infraestructura adyacente reutilizable que sigue tratandose como base tecnica de Authorization:

- `vendor/voltstack/framework/src/Quantum/Controllers/Security`
- `vendor/voltstack/framework/src/Quantum/Metadata`
- `vendor/voltstack/framework/src/Quantum/Auth`
- `vendor/voltstack/framework/src/Platform/Application.php`
- `vendor/voltstack/framework/tests/Unit/ControllerSecurityModelTest.php`
- `vendor/voltstack/framework/tests/Unit/PolicyCompositionTest.php`
- `vendor/voltstack/framework/tests/Unit/PolicyWorkerSafetyTest.php`

Conclusión operativa:

- Authorization ya no parte desde un namespace vacio,
- pero todavia sigue en una V1 minima,
- y la siguiente prioridad ya no es fundacional, sino de consolidacion de policies, metadata e integracion con controllers.

## Fuentes de verdad

El orden de autoridad para decidir que construir y como validarlo es:

1. `vendor/voltstack/authorization-lab/Docs/00-32`
2. `Docs/DEVELOPMENT/DEVELOPMENT_MATRIX.md`
3. `Docs/DEVELOPMENT/DEVELOPMENT_VERSIONS.md`
4. evidencia real en el framework:
   - `src/Quantum/Controllers/Security`
   - `src/Quantum/Metadata`
   - `src/Quantum/Auth`
   - `src/Platform/Application.php`
   - tests unitarios relacionados

## Principios de desarrollo

### 1. Authorization no es Authentication

`Quantum/Authorization` debe decidir:

- quien puede hacer algo,
- sobre que recurso,
- bajo que contexto,
- y con que restricciones.

No debe autenticar credenciales, crear sesiones ni asumir que un principal autenticado ya esta autorizado.

### 2. No reconstruir Controllers Security dentro de Authorization

`Quantum/Controllers/Security` ya contiene:

- decision engine,
- atributos declarativos,
- contexto de seguridad,
- composicion de policies,
- y protecciones para worker persistente.

El objetivo no es duplicarlo, sino:

- extraer abstracciones reutilizables,
- crear un core general en `Quantum/Authorization`,
- y despues integrar o adaptar Controllers Security sobre ese core.

### 3. El core primero; las integraciones despues

No abrir primero:

- aprobaciones,
- delegacion,
- impersonation,
- ReBAC completo,
- risk engine,
- tooling administrativo,
- ni coordinacion distribuida avanzada,

si antes no existen:

- `AuthorizationRequest`,
- `AuthorizationContext`,
- `Ability`,
- `Principal`,
- `DecisionResult`,
- `AuthorizationManager`,
- `PolicyRegistry`,
- `PolicyDispatcher`,
- `GateRegistry`,
- y bootstrap del modulo.

### 4. El core no depende de HTTP ni de Controllers

El nucleo de `Quantum/Authorization` no debe depender directamente de:

- `Request`,
- `Route`,
- `ControllerDispatcher`,
- ni clases concretas del subsistema de controllers.

HTTP, routing y controllers seran capas de integracion, no el dominio central.

### 5. Request-scoped siempre

Todo estado mutable de autorizacion debe vivir dentro del scope activo del request o de la operacion.

Nunca persistir en singletons:

- current principal,
- current tenant,
- current subject,
- current decision,
- caches de decisiones sin version/contexto,
- ni metadata mutable de una evaluacion.

### 6. Default deny y fail closed desde el dia uno

Una ausencia de regla aplicable no debe autorizar.

Una falla inesperada del motor no debe producir `ALLOW`.

El baseline de V1 debe nacer con:

- `default deny`,
- `abstain != allow`,
- `unknown ability != allow`,
- `policy failure != allow`.

### 7. Ergonomia simple, semantica explicita

La API publica puede ser sencilla:

- `Authorization::check()`
- `Authorization::authorize()`
- `$user->can()`

pero internamente todo debe converger en una sola semantica:

- request normalizado,
- plan de evaluacion,
- decision final tipada,
- y trazabilidad.

### 8. Reutilizar Metadata antes de inventar un parser paralelo

VoltStack ya tiene un subsistema `Quantum/Metadata`.

La autorizacion declarativa futura debe apoyarse preferentemente en esa infraestructura en vez de crear otro sistema separado de discovery y atributos sin necesidad.

### 9. No introducir clases vacias por cumplir una estructura

La arquitectura objetivo es amplia, pero la implementacion debe crecer por bloques operables.

No crear decenas de carpetas y clases sin comportamiento solo para reflejar la arquitectura final.

### 10. Definition of Done conservadora

Un bloque de Authorization solo puede considerarse cerrado si cumple:

1. implementacion visible en `src/Quantum/Authorization`,
2. integracion real o testable desde framework,
3. pruebas unitarias o de integracion relevantes,
4. compatibilidad con runtime persistente,
5. actualizacion de `DEVELOPMENT_MATRIX.md`,
6. nueva entrada en `DEVELOPMENT_VERSIONS.md`.

## Regla de priorizacion

## Prioridad 0: crear el nucleo minimo operable

Abrir primero estos documentos:

1. `02_AUTHORIZATION_MANAGER_AND_CORE_ENGINE.md`
2. `03_AUTHORIZATION_REQUEST_CONTEXT_AND_SUBJECT_MODEL.md`
3. `07_GATE_SYSTEM_AND_ABILITY_REGISTRY.md`
4. `08_DECISION_MANAGER_VOTERS_AND_STRATEGY_SYSTEM.md`
5. `25_AUTHORIZATION_CONFIGURATION_BOOTSTRAP_AND_SERVICE_CONTAINER_INTEGRATION_SYSTEM.md`

Motivo:

- el namespace objetivo esta vacio,
- falta un lenguaje canonico del subsistema,
- y sin eso no hay base segura para policies, gates ni integraciones.

## Prioridad 1: cerrar el sistema minimo de policies

1. `04_POLICY_SYSTEM_AND_POLICY_CONTRACTS.md`
2. `05_POLICY_REGISTRY_DISCOVERY_AND_RESOLUTION_SYSTEM.md`
3. `06_POLICY_DISPATCHER_INVOCATION_AND_RESULT_NORMALIZATION_SYSTEM.md`
4. `09_AUTHORIZATION_PLANNER_AND_POLICY_PIPELINE_SYSTEM.md`
5. `10_AUTHORIZATION_ATTRIBUTES_AND_DECLARATIVE_METADATA_SYSTEM.md`

Motivo:

- Controllers Security ya demuestra ideas de policy y metadata,
- pero falta el motor general y reusable para todo el framework.

## Prioridad 2: cerrar la integracion usable del framework

1. `11_CONTROLLER_ROUTE_AND_ACTION_AUTHORIZATION_INTEGRATION_SYSTEM.md`
2. `16_AUTHORIZATION_FAILURE_ERROR_DENIAL_AND_EXCEPTION_HANDLING_SYSTEM.md`
3. `17_AUTHORIZATION_TESTING_VERIFICATION_AND_SECURITY_ASSURANCE_SYSTEM.md`
4. `18_AUTHORIZATION_COMPILATION_OPTIMIZATION_AND_RUNTIME_PERFORMANCE_SYSTEM.md`
5. `32_AUTHORIZATION_SYSTEM_INTEGRATION_AND_FINAL_ARCHITECTURE.md`

Motivo:

- esta fase convierte el core en una capacidad real del framework,
- y define la ruta de migracion desde `Controllers/Security`.

## Prioridad 3: modelos de autoridad y restricciones avanzadas

1. `12`, `13`, `14`
2. `19`, `20`, `21`, `22`, `23`
3. `24`, `26`, `27`, `28`, `29`, `30`, `31`

Motivo:

- estos bloques dependen de un core, bootstrap e integracion ya estables.

## Flujo obligatorio por cada nueva fase

Cada iteracion de Authorization debe seguir este flujo:

1. identificar el documento fuente principal,
2. detectar que ya existe en `Controllers/Security`, `Metadata` o `Auth`,
3. delimitar que debe vivir en `Quantum/Authorization`,
4. diseñar el bloque minimo operable,
5. implementar,
6. validar con tests,
7. actualizar `DEVELOPMENT_MATRIX.md`,
8. registrar la iteracion en `DEVELOPMENT_VERSIONS.md`.

## Reglas por tipo de bloque

### Core y modelo de decision

Cualquier trabajo sobre `02`, `03`, `07`, `08` debe respetar:

- value objects tipados,
- inmutabilidad de request/context/result,
- separacion entre planning y execution,
- una sola semantica de decision final.

### Policies y metadata

Cualquier trabajo sobre `04`, `05`, `06`, `09`, `10` debe respetar:

- policies stateless por defecto,
- discovery compilable,
- adapters sobre `Quantum/Metadata`,
- y cero reflection en hot path productivo cuando exista metadata compilada.

### Integracion framework

Cualquier trabajo sobre `11`, `16`, `17`, `18`, `25`, `32` debe respetar:

- integración con `Application`,
- excepción y mapping sin acoplar el core a HTTP,
- request isolation para FrankenPHP,
- y migracion incremental desde `Controllers/Security`.

### Modelos avanzados

Cualquier trabajo sobre `12-31` debe respetar:

- que Authorization siga siendo un motor unificado,
- sin crear mini-motores paralelos para RBAC, risk, approvals o delegation.

## Validacion minima obligatoria

Antes de cerrar cualquier bloque de Authorization, validar como minimo:

- tests unitarios del core tocado,
- tests de integracion del flujo afectado,
- ausencia de fuga de principal/tenant/decision entre requests,
- comportamiento `default deny`,
- comportamiento fail-closed ante fallas del motor,
- y evidencia visible de bootstrap o uso real.

## Regla de evidencia para la matriz

No subir un bloque a `Operativo` si la evidencia solo existe en:

- documentación,
- ideas adyacentes en `Controllers/Security`,
- o tests de conceptos sin implementación propia en `Quantum/Authorization`.

Para `Operativo`, debe existir implementacion real en el namespace objetivo o una integracion formal del modulo Authorization ya conectada al framework.

## Regla permanente de mantenimiento

Siempre que se realice desarrollo sobre `Quantum/Authorization`, en el mismo ciclo deben quedar actualizados:

- `DEVELOPMENT_GUIDELINES.md`
- `DEVELOPMENT_MATRIX.md`
- `DEVELOPMENT_VERSIONS.md`
- `EXECUTIVE_PLAN_IMPLEMENTATION.md` cuando cambie la secuencia ejecutiva

## Anti-patrones a evitar

No continuar el desarrollo con estos patrones:

1. copiar `Controllers/Security` dentro de `Quantum/Authorization` con rename cosmetico,
2. mezclar autenticacion y autorizacion en el manager,
3. guardar principal o tenant actual en singletons shared,
4. usar strings y arrays sin value objects para todo el core,
5. acoplar el core a HTTP o al ORM,
6. tratar `ABSTAIN` como `ALLOW`,
7. abrir approval, risk o delegation antes de tener core y policies,
8. considerar la arquitectura "implementada" solo porque existe un árbol de carpetas.

## Siguiente ejecucion recomendada

### Fase sugerida inmediata

`DV-AUTHZ-003: Policies, Registry, Metadata E Integracion Inicial`

Documentos objetivo:

- `04_POLICY_SYSTEM_AND_POLICY_CONTRACTS.md`
- `05_POLICY_REGISTRY_DISCOVERY_AND_RESOLUTION_SYSTEM.md`
- `06_POLICY_DISPATCHER_INVOCATION_AND_RESULT_NORMALIZATION_SYSTEM.md`
- `09_AUTHORIZATION_PLANNER_AND_POLICY_PIPELINE_SYSTEM.md`
- `10_AUTHORIZATION_ATTRIBUTES_AND_DECLARATIVE_METADATA_SYSTEM.md`
- `11_CONTROLLER_ROUTE_AND_ACTION_AUTHORIZATION_INTEGRATION_SYSTEM.md`

### Entregables minimos sugeridos

1. introducir contratos explicitos de policy,
2. pasar de registry manual a discovery/configuracion mas formal,
3. definir metadata declarativa propia de Authorization apoyada en `Quantum/Metadata`,
4. cerrar una primera integracion entre controllers/routing y `AuthorizationManager`,
5. ampliar pruebas hacia flows de integracion y denegacion observable.
