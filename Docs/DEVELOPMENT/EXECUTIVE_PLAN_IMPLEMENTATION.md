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
- provider y bootstrap base.

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
- `DecisionManager` ya aplica `default_strategy`,
- y el motor ya diferencia `fail_closed` de `fail_open` ante fallos de evaluadores.

## Fase 6 - Cierre de V1 minima

### Objetivo

Declarar una primera version util y segura del Authorization Engine.

### Entregables

1. `AuthorizationManager` usable,
2. modelado canonico del request,
3. policies y gates minimos,
4. bootstrap y config propios,
5. integracion inicial con framework,
6. pruebas base del subsistema,
7. actualizacion de matriz y bitacora.

### Criterio de cierre de V1

Se puede considerar cerrada la primera version cuando exista:

1. ability + principal + subject + context,
2. decision model tipado,
3. `check()` y `authorize()` reales,
4. policy/gate system minimo,
5. bootstrap en el framework,
6. request isolation compatible con FrankenPHP,
7. pruebas unitarias e integracion basicas.

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

### DV-AUTHZ-004

Alcance sugerido:

- definir el primer planner/pipeline formal,
- mover discovery/metadata hacia una capa compilable o manifestable,
- ampliar la convergencia entre `Controllers/Security` y `Quantum/Authorization`.

Estado del corte:

- el primer planner formal ya existe en version minima,
- ya existe un pipeline minimo por stages,
- por lo que el siguiente trabajo debe enriquecer pipeline, metadata compilable y trazabilidad.

Entregables minimos:

1. planner/pipeline de evaluacion,
2. discovery/configuracion compilable,
3. metadata declarativa soportada por infraestructura reusable,
4. mayor integracion transversal del framework,
5. pruebas ampliadas de integracion y errores,
6. actualizacion de matriz y bitacora.

Resultado esperado:

- VoltStack pasa de una V1 conectada inicial a una V1 mas estable, compilable y lista para rollout incremental.
