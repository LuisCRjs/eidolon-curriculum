# PF-M8 — Async/await, tasks, cancellation y backpressure

**Track:** Programming Foundations  
**Competencias:** D1.1; refuerza D3.1, D3.2 y D11.1  
**Fase:** P0  
**Nivel objetivo:** Working Knowledge para async/await; profesional para el alcance de ownership y errores  
**Prerequisites:** PF-M1, PF-M2, PF-M3, PF-M4, PF-M5, PF-M6, PF-M7  
**Build:** EIDOLON 0.0a  
**Curriculum source:** [PF-M8](../../02_curriculum/01_programming_foundations.md#pf-m8--asyncawait-antes-del-web-framework)  
**Status:** review candidate

Una operación secuencial que espera I/O puede dejar al programa ocioso aunque exista otro trabajo listo. Crear miles de tasks tampoco resuelve el problema: puede convertir espera en saturación, trabajo huérfano y shutdown impredecible.

PF-M8 enseña concurrencia cooperativa como un contrato de lifecycle:

```text
qué problema existe
      ↓
por qué la espera desperdicia tiempo
      ↓
qué significa ceder control
      ↓
cómo coordinar trabajo concurrente
      ↓
cómo limitar, cancelar y limpiar
```

PF-M1–PF-M7 ya aportan objetos, funciones, iteración, exceptions, `try/finally`, context managers y resource ownership. Aquí se reutilizan. No se enseñan networking, backend, databases, internals de threads/processes ni event loops custom.

## Resultados de aprendizaje

Al terminar deberías poder:

- distinguir concurrency y parallelism sin atribuir paralelismo CPU a `asyncio`;
- clasificar trabajo I/O-bound y CPU-bound;
- trazar qué task ejecuta y dónde cede control;
- explicar qué produce llamar una `async def`;
- distinguir coroutine object, awaitable y Task al nivel práctico;
- usar `asyncio.run` como frontera sync → async;
- explicar por qué `await` no crea automáticamente otra Task;
- comparar `asyncio.sleep` con `time.sleep` dentro del event loop;
- distinguir async secuencial de concurrente;
- crear Tasks con owner y esperar sus resultados;
- usar `TaskGroup` para structured concurrency;
- observar resultados y exceptions sin tasks olvidadas;
- cancelar una Task y preservar `CancelledError`;
- ejecutar cleanup ante cancellation;
- aplicar `asyncio.timeout` sin confundir timeout, retry y medición;
- detectar blocking I/O/CPU dentro del event loop;
- usar `asyncio.to_thread` solo como adapter acotado de blocking I/O;
- reproducir un race lógico alrededor de `await`;
- limitar concurrencia con `Semaphore`;
- usar `async with` al nivel requerido por APIs async;
- construir producer/consumer con `Queue(maxsize=...)`;
- explicar backpressure y shutdown explícito;
- usar `task_done`/`join` con correspondencia correcta;
- distinguir retry temporal, invalid input, bug y cancellation;
- introducir una idempotency key estable sin diseñar infraestructura distribuida;
- separar completion order de output order;
- decidir una política para partial results/effects;
- construir un Import Coordinator sintético con resumen determinista.

## Cómo estudiar este módulo

Para cada ejemplo:

1. enumera coroutines creadas y Tasks que las poseen;
2. marca cada `await`;
3. predice qué trabajo puede progresar allí;
4. identifica quién espera/cancela cada Task;
5. provoca failure, timeout y cancellation;
6. comprueba cleanup;
7. separa completion order de contrato final;
8. pregunta qué limita memoria y trabajo en vuelo.

### Convenciones

- **Ejemplo ejecutable:** bloque autónomo.
- **Continuación:** depende solo del bloque anterior.
- **Código incorrecto:** antipatrón deliberado.
- **Failure case:** demuestra fallo o contrato peligroso.
- **Fragmento:** no es programa completo.

Los ejemplos usan Python 3.14 y delays sintéticos cortos. No se fijan duraciones exactas ni orden que el contrato no garantice. `assert` verifica propiedades estables; PF-M9 ampliará tests/debugging/logging.

---

## 1. Por qué existe async

### 1.1 Esperas secuenciales

```text
leer A: trabajar ─ esperar ─ terminar
leer B:                         trabajar ─ esperar ─ terminar
```

Durante espera de I/O, el CPU puede tener otro trabajo listo. Async permite que una coroutine ceda control cooperativamente.

### 1.2 No acelera todo

```text
task A ejecuta
    ↓ await de I/O simulado
cede control
    ↓
task B ejecuta
    ↓ await
event loop retoma A cuando puede continuar
```

No existe interrupción arbitraria en cualquier línea de Python ordinario. La coroutine progresa hasta ceder, terminar o fallar.

### Predice

Si ninguna operación cede control, ¿qué oportunidad tiene otra Task de progresar en ese thread?

### Explica

¿Qué tiempo ocioso intenta aprovechar concurrencia async?

---

## 2. Concurrency, parallelism, I/O-bound y CPU-bound

### 2.1 Distinción prudente

- **Concurrency:** varios trabajos progresan durante el mismo periodo.
- **Parallelism:** trabajo ejecuta realmente al mismo tiempo sobre recursos de cómputo distintos.

`asyncio` ofrece principalmente concurrencia cooperativa en un event loop. No crea paralelismo CPU automático.

### 2.2 Clasificar workload

I/O-bound: esperar file, timer, subprocess I/O o servicios futuros.  
CPU-bound: hashing masivo, compresión pesada, cálculo numérico.

**Código que no gana nada:**

```python
async def add(a, b):
    return a + b
```

La suma rápida no contiene espera. `async def` agrega un coroutine contract sin beneficio.

### Clasifica

Clasifica: leer 10 files; comprimir 10 GB; esperar timer; ordenar millones de records.

### Explica

¿Por qué convertir cálculo CPU a `async def` no lo hace paralelo?

---

## 3. Event loop y cooperative scheduling

### 3.1 Modelo mental

```text
event loop
├── ejecuta A hasta await
├── ejecuta B hasta await
├── observa qué awaitables están listos
└── reanuda una Task lista
```

El scheduler puede elegir entre Tasks listas. No conviertas su elección incidental en contrato.

### 3.2 Punto sin `await`

Entre dos puntos donde se cede control, otra Task del mismo loop no ejecuta Python. Esto simplifica algunos cambios locales, pero una section larga/blocking detiene a todas.

### Predice

En `read → compute sin await → await sleep`, ¿dónde puede progresar otra Task?

### Determinismo

¿Qué parte del orden pertenece al scheduler y cuál debes reconstruir?

---

## 4. `async def`, coroutine objects y `asyncio.run`

### 4.1 Llamar no ejecuta el body completo

**Ejemplo ejecutable:**

```python
async def operation():
    print("body")
    return 7


coroutine = operation()
print(type(coroutine).__name__)
coroutine.close()
```

Output:

```text
coroutine
```

No aparece `body`. La llamada crea un coroutine object que representa ejecución pendiente. `close` evita dejarlo sin resolver en este experimento; normalmente se espera.

Crear una coroutine y abandonarla puede producir `RuntimeWarning: coroutine was never awaited` cuando Python detecta su destrucción. El momento/texto no es contrato para tests.

### 4.2 Frontera sync → async

**Ejemplo ejecutable:**

```python
import asyncio


async def operation():
    return 7


async def main():
    result = await operation()
    print(result)


asyncio.run(main())
```

Output:

```text
7
```

```text
programa sync
  ↓ asyncio.run
event loop
  ↓ ejecuta
main coroutine
```

`asyncio.run` crea/administra loop y cierra la frontera. No lo llames normalmente desde otro event loop activo.

### Predice

¿Qué imprime llamar `operation()` sin await?

### Detecta el bug

¿Por qué `result = operation()` no contiene 7?

---

## 5. `await` no significa “crear Task”

### 5.1 Esperar un awaitable

Dentro de async context:

```python
result = await operation()
```

`await` ejecuta/espera el awaitable y puede ceder mientras está pendiente. La coroutine actual no avanza más allá hasta obtener resultado o exception.

### 5.2 Una llamada sigue dentro de la misma Task

**Ejemplo ejecutable:**

```python
import asyncio


async def child():
    await asyncio.sleep(0)
    return "child"


async def main():
    result = await child()
    print(result)


asyncio.run(main())
```

Output:

```text
child
```

No se creó explícitamente una Task hija. `main` espera child dentro de su flujo. Para progreso concurrente se necesitan Tasks separadas u otra API que las cree.

### Explica

¿Dónde puede ceder child y qué espera main?

### Predice

¿`await child()` permite que main ejecute su siguiente línea antes del resultado?

---

## 6. `asyncio.sleep` y blocking sleep

### 6.1 Simular I/O cooperativo

**Ejemplo ejecutable:**

```python
import asyncio


async def operation(name):
    print(f"{name}:start")
    await asyncio.sleep(0)
    print(f"{name}:end")


asyncio.run(operation("A"))
```

Output:

```text
A:start
A:end
```

`asyncio.sleep` suspende la coroutine y permite al loop ejecutar otra Task.

### 6.2 Failure case: `time.sleep`

**Código incorrecto:**

```python
import time


async def operation():
    time.sleep(5)
```

`time.sleep` bloquea el thread del event loop; ninguna Task del mismo loop progresa durante esa espera. No se arregla escribiendo `async def`.

### Detecta el bug

¿Dónde cede control la coroutine incorrecta?

### Refactoriza

Si el delay solo simula I/O, usa `await asyncio.sleep`.

---

## 7. Async secuencial y concurrente

### 7.1 Dos awaits siguen en secuencia

**Ejemplo ejecutable:**

```python
import asyncio


async def work(name, events):
    events.append(f"{name}:start")
    await asyncio.sleep(0)
    events.append(f"{name}:end")
    return name


async def main():
    events = []
    first = await work("A", events)
    second = await work("B", events)
    print(events)
    print(first, second)


asyncio.run(main())
```

Output:

```text
['A:start', 'A:end', 'B:start', 'B:end']
A B
```

### 7.2 Tasks separadas

**Ejemplo ejecutable:**

```python
import asyncio


async def work(name, events):
    events.append(f"{name}:start")
    await asyncio.sleep(0)
    events.append(f"{name}:end")
    return name


async def main():
    events = []
    first = asyncio.create_task(work("A", events))
    second = asyncio.create_task(work("B", events))
    results = await asyncio.gather(first, second)
    print(results)
    assert set(events) == {"A:start", "A:end", "B:start", "B:end"}


asyncio.run(main())
```

Output:

```text
['A', 'B']
```

`gather` retorna resultados en orden de awaitables recibidos, no por completion order. No fijamos el orden de `events`.

### Clasifica

Clasifica ambos ejemplos y explica por qué ninguno demuestra parallel CPU.

### Ownership

¿Quién conserva referencias y espera ambas Tasks?

---

## 8. `create_task` y ownership

`asyncio.create_task(coro)` envuelve/schedulea una coroutine en el loop actual. La Task puede progresar cuando el loop recibe control.

Pregunta central:

> ¿Quién creó esta Task y quién esperará, cancelará o inspeccionará su resultado?

### 8.1 Anti-pattern: forgotten Task

```python
async def main():
    asyncio.create_task(operation())
    return "done"
```

La Task no tiene owner visible. El loop puede cerrar antes de que termine; su exception puede quedar sin observación y sus resources en estado parcial.

Fire-and-forget no es default seguro. Si un lifecycle realmente supera el scope, necesita un owner explícito de aplicación, shutdown y error policy, fuera de este módulo.

### Detecta el bug

¿Quién observa la exception de `operation`?

### Ownership

Rediseña para guardar Task y await dentro del mismo scope.

---

## 9. `gather` y `TaskGroup`

### 9.1 Gather para resultados ordenados

`await asyncio.gather(a(), b())` ejecuta awaitables concurrentemente y devuelve resultados en el orden de inputs. Su error/cancellation contract requiere leer documentación antes de usar variantes; PF-M8 no cataloga todas.

### 9.2 Structured concurrency

**Ejemplo ejecutable:**

```python
import asyncio


async def double(value):
    await asyncio.sleep(0)
    return value * 2


async def main():
    async with asyncio.TaskGroup() as group:
        first = group.create_task(double(2))
        second = group.create_task(double(3))

    print(first.result(), second.result())


asyncio.run(main())
```

Output:

```text
4 6
```

```text
child Tasks pertenecen al TaskGroup scope
↓
scope espera que terminen
↓
después pueden inspeccionarse results
```

Si una child falla con exception distinta de cancellation, TaskGroup cancela siblings restantes y al salir agrupa failures en `ExceptionGroup`. PF-M8 necesita reconocer ese contrato, no manipular grupos complejos.

### Predice

¿Puede `main` salir del `async with` mientras una child sigue activa?

### Explica

¿Qué ownership mejora frente a una list global de Tasks?

---

## 10. Resultados y exceptions de Tasks

Una Task termina en uno de tres estados prácticos:

- produjo un resultado;
- terminó con una exception;
- fue cancelada.

`await task` observa ese desenlace. Ignorarlo no elimina el error; solo pierde un lugar claro para manejarlo.

**Ejemplo ejecutable:**

```python
import asyncio


async def parse_source(source_id):
    await asyncio.sleep(0)
    if source_id == "bad":
        raise ValueError("synthetic invalid source")
    return source_id.upper()


async def main():
    task = asyncio.create_task(parse_source("bad"))
    try:
        await task
    except ValueError as error:
        print(type(error).__name__)


asyncio.run(main())
```

Output:

```text
ValueError
```

La taxonomía completa de errors pertenece a PF-M6. Aquí importa el lifecycle: el owner observa el failure y decide qué significa para el scope.

### Detecta el bug

Una función crea diez Tasks, retorna inmediatamente y nadie conserva referencias. Enumera dos riesgos.

### Modifica

Haz que el owner espere la Task y convierta solo `ValueError` en un resultado explícito.

---

## 11. Cancellation es un contrato

`task.cancel()` solicita cancellation. No mata de forma instantánea una función en una línea arbitraria. En el próximo punto apropiado de suspensión, normalmente se inyecta `asyncio.CancelledError`.

**Ejemplo ejecutable:**

```python
import asyncio


async def worker(events):
    try:
        events.append("started")
        await asyncio.sleep(10)
        events.append("completed")
    finally:
        events.append("cleanup")


async def main():
    events = []
    task = asyncio.create_task(worker(events))
    await asyncio.sleep(0)
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        events.append("cancelled observed")

    print(events)


asyncio.run(main())
```

Output:

```text
['started', 'cleanup', 'cancelled observed']
```

El owner cancela **y después espera**. Así observa que la Task terminó y que ejecutó cleanup.

### 11.1 No tragar cancellation

**Código incorrecto:**

```python
async def import_one():
    try:
        await long_operation()
    except asyncio.CancelledError:
        return "ok"
```

La Task finge éxito y evita que cancellation se propague al owner. Si necesitas cleanup o telemetría local:

```python
async def import_one():
    try:
        await long_operation()
    except asyncio.CancelledError:
        record_local_cleanup()
        raise
```

En Python 3.14, `asyncio.CancelledError` deriva directamente de `BaseException`, no de `Exception`. Aun así, no dependas de un `except Exception` como estrategia de cancellation: captura explícitamente solo cuando debas limpiar y vuelve a lanzar.

### Predice

En el primer ejemplo, ¿aparece `completed`? ¿Por qué sí aparece `cleanup`?

### Explica

¿Por qué `cancel()` seguido de abandono no constituye shutdown completo?

---

## 12. Cleanup y estado parcial

Cancellation puede llegar después de adquirir un recurso o registrar estado temporal. Todo recurso adquirido necesita ownership y liberación simétricos.

**Ejemplo ejecutable:**

```python
import asyncio


async def stage_import(source_id, staged):
    staged.add(source_id)
    try:
        await asyncio.sleep(10)
        return source_id
    finally:
        staged.discard(source_id)


async def main():
    staged = set()
    task = asyncio.create_task(stage_import("src-1", staged))
    await asyncio.sleep(0)
    print(staged)

    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        pass

    print(staged)


asyncio.run(main())
```

Output:

```text
{'src-1'}
set()
```

`discard` hace cleanup idempotente para este set local: repetirlo no falla. Esto no vuelve idempotente cualquier efecto externo futuro.

### Detecta el bug

Un import agrega `source_id` a `staged`, hace await y lo elimina solo en la ruta de éxito. ¿Qué estado deja una cancellation?

### Invariante

Escribe la condición que `staged` debe cumplir después de success, failure o cancellation.

---

## 13. Timeouts y duración

Un timeout limita cuánto acepta esperar un scope. No mide con precisión contractual cuánto tardó, no reintenta y no garantiza que una operación externa desconocida sea reversible.

**Ejemplo ejecutable:**

```python
import asyncio


async def synthetic_read():
    await asyncio.sleep(0.05)
    return "event"


async def main():
    try:
        async with asyncio.timeout(0.01):
            await synthetic_read()
    except TimeoutError:
        print("timed out")


asyncio.run(main())
```

Output:

```text
timed out
```

`asyncio.timeout` cancela el trabajo dentro del context y transforma esa cancellation en `TimeoutError` al salir. Captura `TimeoutError` fuera del `async with` cuando quieras decidir la política del caller.

### 13.1 Contratos distintos

| Concepto | Pregunta |
|---|---|
| timeout | ¿Cuánto acepto esperar? |
| retry | ¿Intento otra vez un failure temporal? |
| duration | ¿Cuánto tiempo observé que tomó? |
| cancellation | ¿Debe detenerse este trabajo? |

No uses un timeout de milisegundos exactos como test de performance. Scheduling y carga de la máquina varían.

Medir duration es otra operación. PF-M7 ya introdujo el performance clock:

```python
started = time.perf_counter()
result = await operation()
elapsed = time.perf_counter() - started
```

Este fragmento observa elapsed; no detiene `operation`.

### Predice

Si `synthetic_read` tarda menos que el límite, ¿qué valor retorna el context?

### Explica

¿Por qué timeout no implica retry?

---

## 14. Blocking work dentro del event loop

El event loop solo puede coordinar mientras el código devuelve control. Una llamada blocking retiene el thread.

**Código incorrecto:**

```python
import time


async def import_one():
    time.sleep(1)
    return "done"
```

Mientras `time.sleep` bloquea, otras Tasks del mismo loop no progresan.

### 14.1 Adapter acotado con `to_thread`

Si una library existente expone I/O blocking y todavía no tiene API async:

```python
import asyncio
import time


def legacy_blocking_read(source_id):
    time.sleep(0.01)
    return source_id.upper()


async def main():
    result = await asyncio.to_thread(legacy_blocking_read, "src-1")
    print(result)


asyncio.run(main())
```

Output:

```text
SRC-1
```

`to_thread` es un bridge para una función blocking, especialmente I/O. No la vuelve cancelable internamente: cancelar el await no detiene por fuerza la función que ya ejecuta en su thread. Tampoco es la solución general para CPU-bound work ni un permiso para ocultar blocking code.

### Detecta el bug

Una coroutine llama `time.sleep` dentro de un loop sobre 100 sources. ¿Qué ocurre con el resto de Tasks?

### Decide

Para cada caso, elige: función sync normal, API async, `to_thread` acotado o tema posterior de parallelism: suma de dos ints; library legacy que espera disco; HTTP futuro con cliente async; cálculo CPU intensivo.

---

## 15. Races lógicos y estado mutable compartido

Async cooperativo no elimina races. Un `await` puede separar lectura y escritura de una invariante.

**Ejemplo ejecutable:**

```python
import asyncio


counter = 0


async def increment():
    global counter
    current = counter
    await asyncio.sleep(0)
    counter = current + 1


async def main():
    await asyncio.gather(increment(), increment())
    print(counter)


asyncio.run(main())
```

Output:

```text
1
```

Ambas coroutines leen `0` antes de escribir. Una actualización se pierde.

### 15.1 Primera defensa: ownership y mensajes

Antes de añadir locks, pregunta:

- ¿puede una sola Task poseer y actualizar el estado?;
- ¿pueden workers retornar resultados en lugar de mutar un dict global?;
- ¿puede una Queue transportar trabajo hacia el owner?;

`asyncio.Lock` existe, pero no corrige un modelo sin ownership. Su uso profundo queda para cuando un invariante realmente necesite exclusión mutua.

### Predice

¿Qué cambiaría si no hubiera `await` entre read y write?

### Refactoriza

Haz que cada coroutine retorne `1` y que el owner sume los resultados.

---

## 16. Bounded concurrency con `Semaphore`

Crear una Task por cada uno de 100 000 items puede agotar memoria, file descriptors o cuotas futuras. El límite debe representar capacidad real.

**Ejemplo ejecutable:**

```python
import asyncio


async def main():
    limit = asyncio.Semaphore(2)
    active = 0
    maximum = 0

    async def import_one(source_id):
        nonlocal active, maximum
        async with limit:
            active += 1
            maximum = max(maximum, active)
            try:
                await asyncio.sleep(0.01)
                return source_id
            finally:
                active -= 1

    results = await asyncio.gather(
        *(import_one(source_id) for source_id in ["a", "b", "c", "d"])
    )
    print(results)
    print(maximum <= 2)


asyncio.run(main())
```

Output:

```text
['a', 'b', 'c', 'd']
True
```

`async with limit` adquiere y libera el permiso incluso ante exception o cancellation. El semaphore limita la sección protegida, pero este patrón todavía crea todas las coroutine objects de entrada. Para streams grandes, Queue acota también el trabajo pendiente.

### Predice

¿El orden de completion debe coincidir con `a, b, c, d`? ¿Por qué el output de `gather` sí mantiene ese orden?

### Modifica

Cambia el límite a uno y describe el comportamiento, sin afirmar una duración exacta.

---

## 17. `async with` en APIs async

PF-M7 enseñó que un context manager liga adquisición y liberación a un scope. Un async context manager permite que ambos extremos necesiten `await`.

```python
async with limiter:
    await operation()
```

El estudiante de PF-M8 necesita **usar** contratos como `Semaphore` y `asyncio.timeout`. La implementación con `__aenter__`/`__aexit__` y `asynccontextmanager` se reserva para profundización posterior.

### Explica

¿Qué lifecycle hace visible `async with limiter`?

---

## 18. `Queue` y backpressure

Una Queue desacopla productores y consumidores. `maxsize` define cuántos items pendientes puede aceptar antes de hacer esperar al producer.

```text
producer rápido
     ↓ put
Queue(maxsize=N) llena
     ↓ producer espera
consumers avanzan
```

Esa espera es **backpressure**: la capacidad downstream limita la producción upstream.

**Ejemplo ejecutable:**

```python
import asyncio


async def main():
    queue = asyncio.Queue(maxsize=1)
    await queue.put("first")

    second_put = asyncio.create_task(queue.put("second"))
    await asyncio.sleep(0)
    print(second_put.done())

    item = await queue.get()
    queue.task_done()
    await second_put
    print(item, await queue.get())
    queue.task_done()


asyncio.run(main())
```

Output:

```text
False
first second
```

La segunda inserción espera capacidad. Una Queue sin límite práctico puede mover el bottleneck hacia memoria.

### 18.1 Failure case: Queue sin límite

```python
queue = asyncio.Queue()

for item in very_large_input:
    await queue.put(item)
```

`put` no espera por capacidad porque `maxsize=0` significa sin límite. Si producer supera consumers durante suficiente tiempo, backlog, memoria y latencia pueden crecer. No toda Queue necesita límite: un input pequeño y ya acotado puede ser seguro. La decisión requiere conocer la capacidad máxima.

Este mismo razonamiento preparará, en tracks posteriores, ingestion, consolidation, embeddings y model calls. Aquí no se implementa ninguno.

### Predice

¿Qué ocurriría con `second_put.done()` si un consumer hubiera retirado `first` antes de comprobarlo?

### Explica

¿Qué diferencia hay entre limitar workers con Semaphore y limitar backlog con Queue?

---

## 19. Producer/consumer, `task_done` y `join`

Un producer crea trabajo; uno o más consumers lo procesan. El owner coordina el lifecycle completo.

**Ejemplo ejecutable:**

```python
import asyncio


async def producer(queue, source_ids, worker_count):
    for source_id in source_ids:
        await queue.put(source_id)
    for _ in range(worker_count):
        await queue.put(None)


async def consumer(queue, results):
    while True:
        source_id = await queue.get()
        try:
            if source_id is None:
                return
            await asyncio.sleep(0)
            results.append(source_id.upper())
        finally:
            queue.task_done()


async def main():
    worker_count = 2
    queue = asyncio.Queue(maxsize=2)
    results = []

    async with asyncio.TaskGroup() as group:
        group.create_task(producer(queue, ["a", "b", "c"], worker_count))
        for _ in range(worker_count):
            group.create_task(consumer(queue, results))
        await queue.join()

    print(sorted(results))


asyncio.run(main())
```

Output:

```text
['A', 'B', 'C']
```

El sentinel `None` es un mensaje de shutdown por worker. `task_done()` afirma que un item retirado terminó de procesarse; `join()` espera que cada `put` tenga su `task_done` correspondiente.

### 19.1 Failure cases

- Omitir `task_done`: `join` puede esperar indefinidamente.
- Llamarlo dos veces: `ValueError: task_done() called too many times`.
- Enviar un solo sentinel a varios workers: algunos quedan esperando.
- No incluir el sentinel en la contabilidad: rompe la correspondencia entre `put` y `task_done`.

`task_done` no significa “la Task de asyncio terminó”. Pertenece a la contabilidad de items de Queue.

### Detecta el bug

Hay tres consumers y un solo `None`. ¿Qué Tasks no pueden terminar?

### Modifica

Agrega un campo sintético de delay por source y comprueba el resumen con `sorted`, no el orden incidental de append.

---

## 20. Shutdown ordenado

Shutdown no es cerrar el event loop y esperar suerte. Necesita una secuencia explícita:

```text
dejar de admitir trabajo
↓
señalar consumers o cancelar el scope
↓
esperar Tasks
↓
ejecutar cleanup
↓
verificar invariantes finales
```

Hay dos políticas válidas según el contrato:

- **drain:** terminar el trabajo ya aceptado antes de cerrar;
- **cancel:** detener pronto y limpiar estado parcial.

No son intercambiables. Un command interactivo quizá prefiera cancel; un batch pequeño quizá prefiera drain. La decisión pertenece al owner.

### Failure case: orphan work

```python
async def main():
    for source_id in source_ids:
        asyncio.create_task(import_one(source_id))
```

No hay scope que espere ni política que decida drain/cancel.

### Explica

¿Qué información necesita el owner para elegir drain o cancel?

### Diseña

Escribe cinco pasos de shutdown para un coordinator con Queue y dos consumers.

---

## 21. Retry mínimo e idempotency

Retry sirve para un failure **temporal** y acotado. No arregla invalid input, bugs deterministas ni cancellation.

| Situación | Retry automático |
|---|---|
| espera sintética falla una vez y luego puede funcionar | posiblemente, con límite |
| source_id vacío | no |
| `NameError` por bug | no |
| owner cancela | no |

**Ejemplo ejecutable:**

```python
import asyncio


async def read_with_retry(source_id, attempt_read, max_attempts=2):
    for attempt in range(1, max_attempts + 1):
        try:
            return await attempt_read(source_id)
        except ConnectionError:
            if attempt == max_attempts:
                raise
            await asyncio.sleep(0)


async def main():
    attempts = 0

    async def flaky_read(source_id):
        nonlocal attempts
        attempts += 1
        await asyncio.sleep(0)
        if attempts == 1:
            raise ConnectionError("synthetic transient failure")
        return source_id

    print(await read_with_retry("src-1", flaky_read))
    print(attempts)


asyncio.run(main())
```

Output:

```text
src-1
2
```

El ejemplo usa `ConnectionError` solo como failure sintético reconocible. No hay networking.

### 21.1 Idempotency key

Repetir un intento puede duplicar efectos. Una key estable permite reconocer la misma operación lógica:

```python
job = {
    "source_id": "src-1",
    "idempotency_key": "import:src-1",
}
```

La key no vuelve idempotente el sistema por sí sola; el componente que aplica el efecto debe usarla correctamente. PF-M8 introduce la pregunta, no diseña transacciones distribuidas.

### Detecta el bug

Un retry genera una key aleatoria nueva por intento. ¿Por qué ya no identifica la misma operación lógica?

### Clasifica

Decide cuáles failures reintentarías: timeout temporal conocido, source inválido, cancellation, bug de código.

---

## 22. Async iterables y async generators: introducción

Un iterable sync entrega items sin esperar mediante `for`. Un async iterable puede necesitar espera entre items y se consume con `async for`.

**Ejemplo ejecutable:**

```python
import asyncio


async def synthetic_sources():
    for source_id in ["src-1", "src-2"]:
        await asyncio.sleep(0)
        yield source_id


async def main():
    collected = []
    async for source_id in synthetic_sources():
        collected.append(source_id)
    print(collected)


asyncio.run(main())
```

Output:

```text
['src-1', 'src-2']
```

La llamada crea un async generator object. Su consumo es lazy: produce conforme `async for` pide items. No significa que procese múltiples items concurrentemente ni sustituye backpressure explícito entre producer y workers.

La implementación profunda del protocolo (`__aiter__`, `__anext__`, `StopAsyncIteration`) es LATER.

### Predice

¿Se producen ambos IDs antes de iniciar `async for`?

### Explica

¿Por qué lazy no equivale a concurrente?

---

## 23. Determinismo: completion order no es output order

Dos imports pueden terminar en cualquier orden permitido por delays y scheduling. Si el producto promete un resumen estable, debe reconstruirlo.

**Ejemplo ejecutable:**

```python
import asyncio


async def import_one(source_id, delay):
    await asyncio.sleep(delay)
    return {"source_id": source_id, "event_count": 1}


async def main():
    results = await asyncio.gather(
        import_one("src-b", 0.01),
        import_one("src-a", 0),
    )
    summary = sorted(results, key=lambda item: item["source_id"])
    print(summary)


asyncio.run(main())
```

Output:

```text
[{'source_id': 'src-a', 'event_count': 1}, {'source_id': 'src-b', 'event_count': 1}]
```

`gather` ya retorna según input order; `sorted` expresa aquí otro contrato: output por `source_id`, independiente del orden de entrada y completion.

### Contrato explícito

Define por separado:

- qué orden acepta el input;
- qué orden usa el procesamiento;
- qué orden promete el output.

### Predice

¿Qué source probablemente completa primero? ¿Está permitido depender de ello?

### Modifica

Invierte los inputs y confirma que el summary continúa ordenado por ID.

---

## 24. Partial results y effects

Cuando tres jobs corren concurrentemente y uno falla, necesitas una política:

- **fail-fast:** cancelar siblings y fallar el scope;
- **collect:** capturar failures esperados por job y conservar successes;
- **all-or-nothing:** requiere garantías de efectos que exceden este módulo.

TaskGroup ofrece una buena frontera fail-fast para exceptions no manejadas. Para collect, cada worker puede convertir **failures esperados y específicos** en un result explícito.

```python
async def import_safely(job):
    try:
        value = await import_one(job)
        return {"source_id": job["source_id"], "status": "ok", "value": value}
    except ValueError as error:
        return {"source_id": job["source_id"], "status": "invalid", "error": str(error)}
```

No captures `BaseException` ni conviertas cancellation en dato ordinario. Un result debe distinguir éxito, invalid input y temporal failure sin fingir que todos son equivalentes.

### 24.1 Effects y retry

Si un job modifica estado y luego falla, retry puede repetir parte del efecto. Antes de reintentar pregunta:

1. ¿qué se alcanzó a aplicar?;
2. ¿el cleanup lo revirtió?;
3. ¿la operación es idempotente?;
4. ¿la key permanece estable?;

Cancellation no equivale a rollback. `finally` puede liberar lo que la Task posee, pero no deshace automáticamente un effect ya confirmado fuera de ella.

### Decide

¿Usarías fail-fast o collect para previsualizar imports sintéticos independientes? Justifica.

### Explica

¿Por qué “devolver lo que haya” no es una política suficiente?

---

## 25. Aplicación EIDOLON: Import Coordinator sintético

Ahora reunimos contracts sin networking ni persistencia. El coordinator:

- recibe jobs sintéticos;
- limita backlog y workers;
- aplica timeout por job;
- limpia IDs staged;
- conserva partial results esperados;
- produce un resumen determinista.

### 25.1 Modelo pequeño

**Ejemplo ejecutable:**

```python
import asyncio
from dataclasses import dataclass


@dataclass(frozen=True)
class ImportJob:
    source_id: str
    delay: float
    outcome: str = "ok"


@dataclass(frozen=True)
class ImportResult:
    source_id: str
    status: str
    event_ids: tuple[str, ...] = ()


async def fake_read(job):
    await asyncio.sleep(job.delay)
    if job.outcome == "invalid":
        raise ValueError("invalid synthetic source")
    return (f"evt-{job.source_id}",)


async def process_job(job, staged, timeout_seconds):
    staged.add(job.source_id)
    try:
        async with asyncio.timeout(timeout_seconds):
            event_ids = await fake_read(job)
            return ImportResult(job.source_id, "ok", event_ids)
    except TimeoutError:
        return ImportResult(job.source_id, "timeout")
    except ValueError:
        return ImportResult(job.source_id, "invalid")
    finally:
        staged.discard(job.source_id)


async def produce(queue, jobs, worker_count):
    for job in jobs:
        await queue.put(job)
    for _ in range(worker_count):
        await queue.put(None)


async def consume(queue, results, staged, timeout_seconds):
    while True:
        job = await queue.get()
        try:
            if job is None:
                return
            result = await process_job(job, staged, timeout_seconds)
            results.append(result)
        finally:
            queue.task_done()


async def coordinate(jobs, worker_count=2, queue_size=2, timeout_seconds=0.02):
    queue = asyncio.Queue(maxsize=queue_size)
    results = []
    staged = set()

    async with asyncio.TaskGroup() as group:
        group.create_task(produce(queue, jobs, worker_count))
        for _ in range(worker_count):
            group.create_task(
                consume(queue, results, staged, timeout_seconds)
            )
        await queue.join()

    assert staged == set()
    return sorted(results, key=lambda result: result.source_id)


async def main():
    jobs = [
        ImportJob("src-b", 0),
        ImportJob("src-a", 0.05),
        ImportJob("src-c", 0, "invalid"),
    ]
    results = await coordinate(jobs)
    for result in results:
        print(result.source_id, result.status, result.event_ids)


asyncio.run(main())
```

Output:

```text
src-a timeout ()
src-b ok ('evt-src-b',)
src-c invalid ()
```

### 25.2 Leer el diseño

| Decisión | Contrato |
|---|---|
| Queue acotada | backlog máximo visible |
| número fijo de consumers | concurrencia máxima de jobs |
| TaskGroup | ownership de producer y consumers |
| timeout por job | un job lento no bloquea indefinidamente su worker |
| `finally` | `staged` queda limpio |
| resultados por status | partial results explícitos |
| `sorted` final | output determinista |

No existe source of truth persistente: jobs, results y staged viven en memoria. La aplicación real de effects, archivos o databases pertenece a otros tracks.

### 25.3 Cancellation del coordinator

Si el owner cancela `coordinate`, TaskGroup propaga cancellation a children y espera su salida. Los `finally` de `process_job` y consumers conservan cleanup y contabilidad. El caller todavía debe esperar la Task cancelada.

### Explica

¿Por qué `results.append` no define el orden final del producto?

### Modifica

Agrega `src-d` exitoso y comprueba que el resumen continúe ordenado por `source_id`.

### Detecta el bug

Mueve `queue.task_done()` dentro de la rama success. ¿Qué sucede ante timeout?

---

## 26. Catálogo de anti-patterns

### 26.1 Async de apariencia

```python
async def normalize_tag(tag):
    return tag.strip().lower()
```

No espera. Una función sync expresa mejor el contrato.

### 26.2 Blocking disfrazado

```python
async def wait_for_source():
    time.sleep(2)
```

Bloquea el event loop; cambiar `def` por `async def` no cambia la llamada blocking.

### 26.3 Task sin owner

```python
asyncio.create_task(import_one(job))
```

Sin referencia, await ni scope estructurado, success/failure/cancellation carecen de owner.

### 26.4 Unbounded fan-out

```python
await asyncio.gather(*(import_one(job) for job in huge_input))
```

Crea trabajo para todo el input. Usa fixed workers, Queue acotada o batches según el caso.

### 26.5 Swallow cancellation

```python
async def import_one():
    try:
        await operation()
    except asyncio.CancelledError:
        return None
```

Rompe el shutdown del owner salvo que exista un contrato extraordinario y documentado.

### 26.6 Polling con sleep

```python
while not results:
    await asyncio.sleep(0.1)
```

Una Queue, Task completion u otra primitiva expresa el evento sin polling arbitrario.

### 26.7 Output dependiente de completion

```python
results.append(await import_one(job))
```

Append puede ser correcto como almacenamiento, pero no como promesa de orden si múltiples workers lo ejecutan. Normaliza al final o conserva un índice estable.

### 26.8 Retry indiscriminado

```python
async def import_with_retry(job):
    try:
        return await import_one(job)
    except Exception:
        await asyncio.sleep(0)
        return await import_with_retry(job)
```

Reintenta bugs e invalid input, no tiene límite y altera lifecycle.

### 26.9 Queue sin shutdown

Consumers con `while True` necesitan sentinel, cancellation estructurada u otra señal explícita.

### 26.10 Lock como parche universal

Proteger cada acceso a estado global puede serializar el sistema sin aclarar ownership. Prefiere retornar values y concentrar mutation.

### Revisión rápida

Para cada anti-pattern identifica: contrato oculto, failure observable y refactor mínimo.

---

## 27. Ejercicios guiados

### 27.1 De coroutine a resultado

**Objetivo:** distinguir crear una coroutine de observar su resultado.

**Código inicial:**

```python
async def load_id():
    await asyncio.sleep(0)
    return "src-1"


async def main():
    value = load_id()
    print(value)
```

**Predice antes de ejecutar:** ¿`value` contiene `"src-1"` o un coroutine object?

**Guía:** llama la coroutine dentro de `asyncio.run` y observa el resultado con `await`.

**Solución razonada:**

```python
import asyncio


async def load_id():
    await asyncio.sleep(0)
    return "src-1"


async def main():
    value = await load_id()
    print(value)


asyncio.run(main())
```

**Razonamiento:** `load_id()` crea un coroutine object; `await` ejecuta/espera su contrato dentro del event loop.

**Criterio:** imprime exactamente `src-1` y no produce warning de coroutine abandonada.

**Variación:** retorna dos IDs como tuple sin crear Tasks; explica por qué sigue siendo un solo flujo.

### 27.2 De secuencial a concurrente

**Objetivo:** crear concurrencia con ownership estructurado.

**Código inicial:** dos llamadas `await read("a")` y `await read("b")` consecutivas.

**Predice antes de ejecutar:** ¿cuántas Tasks ejecutan los reads en esa versión?

Después crea Tasks dentro de TaskGroup y recupera results al salir. No uses performance exacta como evidencia.

**Solución posible:**

```python
import asyncio


async def read(source_id):
    await asyncio.sleep(0)
    return source_id


async def main():
    async with asyncio.TaskGroup() as group:
        first = group.create_task(read("a"))
        second = group.create_task(read("b"))
    print(first.result(), second.result())


asyncio.run(main())
```

**Razonamiento:** el scope crea dos child Tasks y conserva owner. El print lee sus resultados después de que el grupo cerró; no depende de completion order.

**Criterio:** ambos resultados se observan y ninguna child sobrevive al scope.

**Variación:** haz que un read lance `ValueError` y explica el efecto fail-fast del grupo.

### 27.3 Cancellation con cleanup

**Objetivo:** demostrar cleanup y propagation bajo cancellation.

**Código inicial:** una coroutine agrega `"src-1"` a `active` y espera diez segundos sin `finally`.

**Predice antes de ejecutar:** ¿qué contiene `active` si el owner cancela durante la espera?

**Solución ejecutable:**

```python
import asyncio


async def operation(active):
    active.add("src-1")
    try:
        await asyncio.sleep(10)
    finally:
        active.discard("src-1")


async def main():
    active = set()
    task = asyncio.create_task(operation(active))
    await asyncio.sleep(0)
    task.cancel()
    try:
        await task
    except asyncio.CancelledError:
        pass
    print(active)


asyncio.run(main())
```

**Razonamiento:** el caller crea, cancela y espera; `finally` retira el ID antes de que observe `CancelledError`.

**Criterio:** imprime `set()` y cancellation no se convierte en success.

**Variación:** agrega dos operaciones en TaskGroup y cancela la Task que posee el grupo.

### 27.4 Timeout por job

**Objetivo:** limitar un job sin capturar cancellation del owner.

**Código inicial:** `fake_read` espera `0.05` y no tiene límite.

**Predice antes de ejecutar:** con límite `0.01`, ¿se imprime `ok` o `timeout`?

**Solución ejecutable:**

```python
import asyncio


async def fake_read(job):
    await asyncio.sleep(0.05)
    return job


async def process(job):
    try:
        async with asyncio.timeout(0.01):
            return {"status": "ok", "value": await fake_read(job)}
    except TimeoutError:
        return {"status": "timeout", "value": None}


async def main():
    print((await process("src-1"))["status"])


asyncio.run(main())
```

**Razonamiento:** el context transforma su timeout interno en `TimeoutError`; `process` convierte solo ese outcome esperado.

**Criterio:** imprime `timeout`; no existe `except BaseException` ni captura de `CancelledError`.

**Variación:** cambia delay a cero y comprueba `ok`.

### 27.5 Race sin lock

**Objetivo:** eliminar shared mutation en lugar de protegerla por reflejo.

**Código inicial:** usa el counter con lost update de la sección 15.

**Predice antes de ejecutar:** ¿qué componente debería poseer el counter final?

**Solución ejecutable:**

```python
import asyncio


async def increment():
    await asyncio.sleep(0)
    return 1


async def main():
    increments = await asyncio.gather(increment(), increment())
    counter = sum(increments)
    print(counter)


asyncio.run(main())
```

**Razonamiento:** cada worker produce un value; el owner único materializa el estado final.

**Criterio:** imprime `2` sin global, `nonlocal` ni Lock.

**Variación:** retorna cantidades distintas y confirma que el patrón sigue funcionando.

### 27.6 Backpressure observable

**Objetivo:** observar backpressure por estado, no por cronómetro.

**Código inicial:** Queue sin `maxsize` y dos `put` consecutivos.

**Predice antes de ejecutar:** ¿puede completar el segundo `put` con una Queue acotada ya llena?

**Solución ejecutable:**

```python
import asyncio


async def main():
    queue = asyncio.Queue(maxsize=1)
    await queue.put("first")
    blocked_put = asyncio.create_task(queue.put("second"))
    await asyncio.sleep(0)
    assert not blocked_put.done()

    first = await queue.get()
    queue.task_done()
    await blocked_put
    second = await queue.get()
    queue.task_done()
    print(first, second)


asyncio.run(main())
```

**Razonamiento:** el primer item ocupa la capacidad; retirar uno habilita el put pendiente.

**Criterio:** la assertion prueba espera por capacidad y el programa imprime `first second`.

**Variación:** usa `maxsize=2` y predice qué assertion cambia.

### 27.7 Producer/consumer limpio

**Objetivo:** cerrar workers y cuadrar Queue accounting.

**Input:**

```text
2 workers
3 jobs
2 sentinels
5 puts
5 task_done
join termina
```

**Predice antes de ejecutar:** ¿cuántos sentinels y cuántos `task_done` hacen falta?

**Solución ejecutable:**

```python
import asyncio


async def worker(queue, output):
    while True:
        item = await queue.get()
        try:
            if item is None:
                return
            output.append(item)
        finally:
            queue.task_done()


async def main():
    queue = asyncio.Queue(maxsize=2)
    output = []
    async with asyncio.TaskGroup() as group:
        for _ in range(2):
            group.create_task(worker(queue, output))
        for item in [1, 2, 3, None, None]:
            await queue.put(item)
        await queue.join()
    print(sorted(output))


asyncio.run(main())
```

**Razonamiento:** cada `get` entra en `try/finally` y llama exactamente una vez `task_done`, incluido sentinel. Cada worker recibe una señal.

**Criterio:** imprime `[1, 2, 3]`, `join` retorna y TaskGroup cierra.

**Variación:** agrega un tercer worker y ajusta solo lo necesario.

### 27.8 Resumen determinista

**Objetivo:** separar collection order de output contract.

**Input:** results en orden `src-c, src-a, src-b`.

**Predice antes de ejecutar:** ¿qué cambia si inviertes el input?

**Solución ejecutable:**

```python
results = [
    {"source_id": "src-c"},
    {"source_id": "src-a"},
    {"source_id": "src-b"},
]
summary = sorted(results, key=lambda result: result["source_id"])
print([result["source_id"] for result in summary])
```

**Razonamiento:** ordenar al boundary expresa output contract; no intenta controlar el scheduler.

**Criterio:** imprime `['src-a', 'src-b', 'src-c']` para cualquier permutación del mismo input.

**Variación:** conserva también un índice original y ordena por `(source_id, index)` cuando haya IDs repetidos.

### 27.9 Observar la exception de una Task

**Objetivo:** impedir que un child failure desaparezca del control flow.

**Código inicial:** `asyncio.create_task(fail())` sin conservar ni esperar el resultado.

**Predice antes de ejecutar:** ¿en qué Task aparece `ValueError` y quién debe observarlo?

**Solución ejecutable:**

```python
import asyncio


async def fail():
    await asyncio.sleep(0)
    raise ValueError("synthetic failure")


async def main():
    task = asyncio.create_task(fail())
    try:
        await task
    except ValueError as error:
        print(str(error))


asyncio.run(main())
```

**Razonamiento:** `await task` entrega el failure al owner, que captura solo el error esperado.

**Criterio:** imprime `synthetic failure`; no aparece “Task exception was never retrieved”.

**Variación:** deja propagar `ValueError` y explica por qué también es observación correcta, aunque cambie el caller contract.

### 27.10 Blocking sleep y cooperative sleep

**Objetivo:** localizar dónde se impide o permite progreso.

**Código inicial:** dos coroutines; una usa `time.sleep` y otra solo registra un marker.

**Predice antes de ejecutar:** escribe los dos órdenes de markers.

**Solución ejecutable:**

```python
import asyncio
import time


async def blocking(markers):
    markers.append("A:start")
    time.sleep(0.01)
    markers.append("A:end")


async def cooperative(markers):
    markers.append("A:start")
    await asyncio.sleep(0)
    markers.append("A:end")


async def ticker(markers):
    markers.append("B")


async def main():
    blocked = []
    await asyncio.gather(blocking(blocked), ticker(blocked))
    cooperative_markers = []
    await asyncio.gather(cooperative(cooperative_markers), ticker(cooperative_markers))
    print(blocked)
    print(cooperative_markers)


asyncio.run(main())
```

**Razonamiento:** `time.sleep` retiene el thread; `await asyncio.sleep(0)` cede antes de `A:end`.

**Criterio:** el primer list ubica `B` al final y el segundo entre start/end; la comparación no depende de duration exacta.

**Variación:** agrega cálculo sync corto antes del await y marca dónde puede ejecutar `B`.

### 27.11 Límite con Semaphore

**Objetivo:** comprobar bounded concurrency como propiedad.

**Código inicial:** cinco operations creadas concurrentemente sin limiter.

**Predice antes de ejecutar:** con Semaphore de dos, ¿qué valor máximo puede alcanzar `active`?

**Solución ejecutable:**

```python
import asyncio


async def main():
    limiter = asyncio.Semaphore(2)
    active = 0
    maximum = 0

    async def operation(value):
        nonlocal active, maximum
        async with limiter:
            active += 1
            maximum = max(maximum, active)
            try:
                await asyncio.sleep(0)
                return value
            finally:
                active -= 1

    values = await asyncio.gather(*(operation(value) for value in range(5)))
    print(values)
    print(maximum)


asyncio.run(main())
```

**Razonamiento:** el permiso abarca exactamente la operation limitada y se libera con `async with`.

**Criterio:** imprime `[0, 1, 2, 3, 4]` y un maximum no mayor que dos.

**Variación:** cambia el límite a tres y verifica `maximum <= 3`.

### 27.12 Diagnosticar `task_done` faltante

**Objetivo:** relacionar `join` con unfinished item accounting.

**Código inicial:** un `get` sin `task_done`.

**Predice antes de ejecutar:** ¿puede `join` saber que el item terminó?

**Solución ejecutable de diagnóstico:**

```python
import asyncio


async def main():
    queue = asyncio.Queue()
    await queue.put("job")
    await queue.get()

    try:
        async with asyncio.timeout(0.01):
            await queue.join()
    except TimeoutError:
        print("missing task_done")

    queue.task_done()
    await queue.join()
    print("accounting restored")


asyncio.run(main())
```

**Razonamiento:** el timeout hace reproducible el diagnóstico sin dejar el ejercicio colgado; no es la solución. La solución es un `task_done` por item retirado.

**Criterio:** imprime ambos mensajes y el segundo `join` termina.

**Variación:** llama `task_done` una vez adicional y explica el `ValueError`.

### 27.13 Shutdown mediante drain

**Objetivo:** dejar de producir, procesar lo aceptado y cerrar cada worker.

**Código inicial:** workers infinitos sin sentinel.

**Predice antes de ejecutar:** con dos workers, ¿cuántas señales de shutdown hacen falta?

**Solución ejecutable:**

```python
import asyncio


async def worker(queue, completed):
    while True:
        job = await queue.get()
        try:
            if job is None:
                return
            await asyncio.sleep(0)
            completed.append(job)
        finally:
            queue.task_done()


async def main():
    queue = asyncio.Queue(maxsize=2)
    completed = []
    async with asyncio.TaskGroup() as group:
        for _ in range(2):
            group.create_task(worker(queue, completed))
        for job in ["a", "b", "c"]:
            await queue.put(job)
        for _ in range(2):
            await queue.put(None)
        await queue.join()
    print(sorted(completed))


asyncio.run(main())
```

**Razonamiento:** el owner deja de admitir jobs, añade una señal por worker, espera Queue accounting y sale del TaskGroup.

**Criterio:** imprime los tres jobs y no queda consumer esperando.

**Variación:** describe cómo cambiaría una política cancel, sin implementarla con sleeps artificiales.

### 27.14 Comparación sync/async sin benchmark frágil

**Objetivo:** demostrar equivalencia de resultado y diferencia de scheduling contract.

**Input:** dos IDs y una espera sintética corta.

**Predice antes de ejecutar:** ¿qué versión permite que ambos reads estén pendientes durante el mismo periodo?

**Solución ejecutable:**

```python
import asyncio
import time


def read_sync(source_id):
    time.sleep(0.01)
    return source_id.upper()


async def read_async(source_id):
    await asyncio.sleep(0.01)
    return source_id.upper()


async def main():
    sync_results = [read_sync(source_id) for source_id in ["a", "b"]]
    async_results = await asyncio.gather(
        *(read_async(source_id) for source_id in ["a", "b"])
    )
    print(sync_results == async_results)


asyncio.run(main())
```

**Razonamiento:** ambas versiones prometen el mismo output. La sync termina un read antes de llamar el siguiente; la async crea dos awaitables para `gather` y ambos pueden progresar durante sus esperas.

**Criterio:** imprime `True`; la explicación usa el trace, no una cifra de speedup.

**Variación:** sustituye la espera por una suma rápida y explica por qué async deja de aportar.

---

## 28. Ejercicios independientes

### Modelo mental

1. Dibuja tres Tasks y marca cada punto donde ceden control.
2. Explica por qué `await child()` puede seguir siendo secuencial.
3. Distingue coroutine function, coroutine object y Task con un ejemplo.
4. Decide si una normalización de string debe ser async.
5. Clasifica cuatro workloads propios como I/O-bound o CPU-bound.
6. Explica por qué scheduling order no es output contract.

### Tasks y ownership

7. Refactoriza tres forgotten Tasks a TaskGroup.
8. Haz que el owner conserve dos resultados por nombre.
9. Provoca un child failure y observa el efecto sobre siblings.
10. Documenta quién crea, espera y cancela cada Task.
11. Compara `gather` y TaskGroup para un caso de resultados ordenados.
12. Encuentra un scope que pueda terminar con work activo y corrígelo.

### Cancellation y timeout

13. Cancela una Task antes de su resultado y verifica cleanup.
14. Reproduce el anti-pattern que traga `CancelledError`; luego corrígelo.
15. Agrega timeout por job, no al batch completo.
16. Agrega timeout al batch y describe el contrato distinto.
17. Distingue timeout, retry y duration en tres funciones.
18. Verifica que `staged` quede vacío ante success, failure y cancellation.

### Blocking y estado

19. Localiza una llamada `time.sleep` en una coroutine y sustitúyela en un ejemplo sintético.
20. Adapta una función blocking de I/O con `to_thread`.
21. Explica por qué cancelar el await no detiene necesariamente el thread.
22. Reproduce una lost update alrededor de `await`.
23. Elimina estado mutable compartido mediante resultados.
24. Justifica si un Lock sería necesario después del refactor.

### Capacidad y backpressure

25. Mide con un counter que Semaphore nunca supera tres operaciones activas.
26. Compara límites 1, 2 y 4 sin afirmar speedup universal.
27. Demuestra que `Queue(maxsize=1)` bloquea al producer.
28. Omite `task_done` deliberadamente y explica por qué `join` no termina; no dejes el ejercicio colgado, usa un timeout de diagnóstico.
29. Llama `task_done` dos veces y observa el `ValueError`.
30. Usa tres workers y el número correcto de sentinels.
31. Cambia la política de shutdown de drain a cancel.
32. Separa límite de backlog y límite de operaciones activas.

### Retry y resultados

33. Implementa máximo de tres intentos para un failure sintético específico.
34. Demuestra que invalid input no se reintenta.
35. Conserva la misma idempotency key en cada intento.
36. Modela status `ok`, `invalid` y `timeout` sin capturar cancellation.
37. Ordena partial results por source ID.
38. Cambia de collect a fail-fast y explica el tradeoff.

### Async iteration y EIDOLON

39. Produce tres IDs con async generator y consúmelos secuencialmente.
40. Envía esos IDs hacia una Queue acotada.
41. Agrega dos workers sin convertir el generator en source of truth.
42. Amplía Import Coordinator con event count determinista.
43. Cancela el coordinator mientras hay staged IDs y verifica cleanup.
44. Demuestra que no quedan Tasks propias pendientes al salir del scope.

---

## 29. Preguntas de comprensión

1. ¿Qué problema resuelve async y cuál no?
2. ¿Qué significa concurrencia cooperativa?
3. ¿Dónde puede otra Task progresar?
4. ¿Qué devuelve una llamada a `async def`?
5. ¿Qué diferencia existe entre coroutine object y Task?
6. ¿Por qué `await` no implica concurrencia entre siblings?
7. ¿Qué hace `asyncio.run` y dónde debe ubicarse normalmente?
8. ¿Por qué `time.sleep` daña al event loop?
9. ¿Qué garantiza `gather` sobre sus resultados y qué no garantiza sobre completion?
10. ¿Qué ownership aporta TaskGroup?
11. ¿Qué debe hacer el owner después de `cancel`?
12. ¿Por qué `CancelledError` no debe convertirse en éxito?
13. ¿Qué función cumple `finally` bajo cancellation?
14. ¿Qué diferencia hay entre timeout y cancellation del owner?
15. ¿Qué no detiene `to_thread` al cancelar su await?
16. ¿Cómo aparece un race lógico en un solo event-loop thread?
17. ¿Cuándo retornar values supera mutar estado compartido?
18. ¿Qué capacidad limita Semaphore?
19. ¿Qué capacidad limita `Queue(maxsize)`?
20. ¿Qué es backpressure?
21. ¿Qué correspondencia existe entre `put`, `task_done` y `join`?
22. ¿Por qué se necesita una señal de shutdown por worker en el patrón mostrado?
23. ¿Qué diferencia hay entre drain y cancel?
24. ¿Qué failures son candidatos razonables para retry?
25. ¿Qué aporta una idempotency key y qué no garantiza?
26. ¿Por qué lazy no significa concurrente?
27. ¿Por qué completion order no debe filtrar al resumen final?
28. ¿Qué políticas existen ante partial failures?
29. ¿Qué invariantes debe cumplir `staged` al terminar?
30. ¿Por qué un Import Coordinator in-memory no es source of truth persistente?

---

## 30. Mini challenge — Bounded Import Coordinator

Construye una aplicación local y sintética resoluble con PF-M1–PF-M8.

### 30.1 Estructura mínima

```text
pf_m8_challenge/
├── pyproject.toml
├── src/
│   └── eidolon_async/
│       ├── __init__.py
│       ├── model.py
│       ├── fake_io.py
│       ├── baseline.py
│       ├── coordinator.py
│       └── cli.py
└── checks/
    ├── success.py
    ├── timeout.py
    ├── cancellation.py
    ├── retry.py
    └── backpressure.py
```

Usa el packaging aprendido en PF-M4. No agregues dependencias externas.

### 30.2 Modelos

Define dataclasses pequeñas:

- `ImportJob(source_id, delay, outcome, idempotency_key)`;
- `ImportResult(source_id, status, event_ids, attempts)`.

Valida invariantes simples ya conocidas de PF-M5: IDs/key no vacíos, delay no negativo y outcomes permitidos. `fake_io` no abre files ni usa red.

### 30.3 Fake I/O

Implementa una coroutine determinista que:

- espera con `asyncio.sleep`;
- retorna IDs sintéticos para `ok`;
- falla una vez y luego funciona para `transient_once`;
- produce invalid input específico para `invalid`;
- permite provocar timeout mediante delay.

El contador de intentos debe pertenecer a un objeto/factory de fake I/O usado por el challenge, no a global oculto.

### 30.4 Coordinator

Debe:

1. recibir jobs, worker count, queue size, timeout y max attempts;
2. usar `Queue(maxsize=...)`;
3. usar TaskGroup para poseer producer y workers;
4. no crear una Task por todo el dataset;
5. recibir un operation limit y aplicarlo con Semaphore alrededor de fake I/O;
6. aplicar timeout por job;
7. reintentar solo `transient_once` hasta el límite;
8. conservar idempotency key entre intentos;
9. no reintentar invalid input, bugs ni cancellation;
10. limpiar `staged_source_ids` con `finally`;
11. llamar `task_done` exactamente una vez por item retirado;
12. apagar todos los workers;
13. devolver summary ordenado por `source_id`;
14. no dejar Tasks huérfanas;
15. incluir una baseline sync con los mismos success inputs para comparar contracts, no un benchmark obligatorio.

### 30.5 Success check

Incluye jobs que completen en orden distinto del input. Comprueba:

- todos los status esperados;
- concurrencia activa de fake I/O nunca mayor al operation limit;
- summary estable por source ID;
- staged vacío;
- idempotency keys sin cambio;
- baseline sync y coordinator async producen el mismo summary para jobs sin failure/timeout.

Output final:

```text
PF-M8 bounded coordinator: PASS
```

### 30.6 Timeout check

Un job excede el límite y se reporta `timeout`. Los demás siguen según la política collect. No uses medición exacta como assertion.

### 30.7 Cancellation check

Inicia coordinator como Task, cede control, cancélalo y espéralo. Comprueba:

- caller observa `CancelledError`;
- staged queda vacío;
- workers no quedan activos;
- cancellation no se convierte en result `ok`.

### 30.8 Retry check

Comprueba que:

- `transient_once` necesita dos intentos;
- key es idéntica en ambos;
- `invalid` usa un intento;
- ninguna ruta tiene retry infinito.

### 30.9 Backpressure check

En un check aislado, llena una `Queue(maxsize=1)`, inicia otro `put` como Task y demuestra que permanece pendiente hasta retirar el primer item. No infieras backpressure a partir de duración.

### 30.10 Explicación obligatoria

Entrega un `README.md` corto que responda:

1. ¿quién posee cada Task?;
2. ¿qué limita Queue, worker count y Semaphore?;
3. ¿dónde ocurre cleanup?;
4. ¿qué policy maneja partial results?;
5. ¿cómo se obtiene determinismo?;
6. ¿qué operation es idempotente y por qué?;
7. ¿qué cambiaría con real I/O sin implementarlo?
8. ¿qué demuestra la comparación sync/async y qué no demuestra sin un benchmark controlado?

### 30.11 Comprobaciones

```bash
python -m compileall -q src checks
python checks/success.py
python checks/timeout.py
python checks/cancellation.py
python checks/retry.py
python checks/backpressure.py
```

### 30.12 Criterio de aprobación

- no hay `time.sleep` dentro de coroutines;
- no hay fire-and-forget;
- structured concurrency posee todo child work;
- Queue, workers y Semaphore tienen límites positivos;
- success, timeout, retry y cancellation se distinguen;
- cancellation se propaga;
- cleanup preserva invariantes;
- retry es acotado, específico e idempotente;
- Queue accounting es exacto;
- summary no depende del scheduler;
- baseline sync y versión async coinciden en output para el caso comparable;
- el proyecto funciona en environment limpio;
- solo usa Python standard library y PF-M1–PF-M8.

### 30.13 Límites

No uses networking, database, ORM, backend, FastAPI, Docker, threads manuales, processes, custom event loop, async generators avanzados, distributed queues, circuit breakers ni pytest avanzado.

---

## 31. Resumen

- Async aprovecha periodos de espera; no acelera automáticamente CPU-bound work.
- Concurrency es progreso solapado; parallelism es ejecución simultánea real.
- El event loop ejecuta una Task hasta que cede, termina o falla.
- Llamar `async def` crea un coroutine object; no ejecuta el body inmediatamente.
- `asyncio.run` establece una frontera sync → async y administra el loop principal.
- `await` espera un awaitable; no crea automáticamente otra Task.
- Async secuencial sigue siendo secuencial aunque use `await`.
- `time.sleep` bloquea el event loop; `asyncio.sleep` cede control.
- Una Task permite progreso concurrente, no paralelismo CPU garantizado.
- Cada Task necesita owner que observe resultado, failure o cancellation.
- TaskGroup liga child Tasks a un scope y aporta structured concurrency.
- `gather` devuelve results en input order, no promete completion order.
- `cancel` solicita cancellation; el owner debe esperar la Task.
- `CancelledError` debe propagarse después de cleanup local.
- `finally` protege invariantes ante success, failure y cancellation.
- Timeout limita espera; no equivale a retry ni a duration measurement.
- `to_thread` adapta blocking I/O acotado; no detiene la función al cancelar el await.
- Un `await` puede separar read/write y producir un race lógico.
- Retornar values hacia un owner reduce shared mutable state.
- Semaphore limita operations simultáneas en una section.
- Queue acotada limita backlog y transmite backpressure.
- Cada Queue `get` procesado requiere exactamente un `task_done`.
- `join` espera contabilidad de items, no Task completion general.
- Shutdown decide drain o cancel y espera el cierre.
- Retry debe ser específico, limitado y compatible con idempotency.
- Async generator produce lazily, no concurrentemente por sí mismo.
- Partial results necesitan una policy explícita.
- Output determinista se construye; no se obtiene controlando scheduler.
- El coordinator EIDOLON de PF-M8 es in-memory y sintético, no persistencia.

---

## 32. Checklist de dominio

- [ ] Puedo distinguir concurrency y parallelism.
- [ ] Puedo clasificar I/O-bound y CPU-bound.
- [ ] Puedo trazar dónde una coroutine cede control.
- [ ] Puedo explicar qué devuelve llamar una `async def`.
- [ ] Puedo distinguir coroutine object y Task.
- [ ] Puedo usar `asyncio.run` como frontera principal.
- [ ] Puedo explicar por qué `await` no crea otra Task.
- [ ] Puedo detectar async secuencial accidental.
- [ ] Puedo evitar `time.sleep` dentro del event loop.
- [ ] Puedo asignar owner a cada Task.
- [ ] Puedo usar TaskGroup para structured concurrency.
- [ ] Puedo observar result y exception de una Task.
- [ ] Puedo cancelar y después esperar una Task.
- [ ] Puedo preservar `CancelledError`.
- [ ] Puedo ejecutar cleanup con `finally`.
- [ ] Puedo aplicar timeout a un scope apropiado.
- [ ] Puedo distinguir timeout, retry y duration.
- [ ] Puedo reconocer límites de `to_thread`.
- [ ] Puedo reproducir un race lógico alrededor de `await`.
- [ ] Puedo reducir shared mutable state mediante return values.
- [ ] Puedo limitar concurrencia con Semaphore.
- [ ] Puedo explicar `async with` al usar primitives async.
- [ ] Puedo crear una Queue acotada.
- [ ] Puedo explicar backpressure.
- [ ] Puedo implementar producer/consumer.
- [ ] Puedo emparejar `put`/`get`/`task_done`/`join`.
- [ ] Puedo cerrar todos los workers.
- [ ] Puedo elegir drain o cancel.
- [ ] Puedo limitar retry a failures temporales específicos.
- [ ] Puedo conservar una idempotency key estable.
- [ ] Puedo consumir un async generator sencillo.
- [ ] Puedo distinguir lazy de concurrente.
- [ ] Puedo separar completion order y output order.
- [ ] Puedo definir policy de partial results.
- [ ] Puedo mantener staged state limpio.
- [ ] Puedo construir un coordinator bounded y determinista.
- [ ] Puedo completar el mini challenge solo con PF-M1–PF-M8.

---

## 33. Preparación para labs y EIDOLON 0.0a

### PF-L13 — Concurrencia acotada y cancelación

PF-M8 prepara evidencia directa para:

- Tasks con owner visible;
- Semaphore o fixed worker count;
- timeout sintético;
- cancellation observada;
- cleanup de staged state;
- Queue acotada y backpressure;
- retry idempotente y limitado;
- summary determinista.

El lab debe trabajar con fake I/O. Agregar network haría más difícil separar el contrato async de failures externos.

### Evidencia antes de avanzar

1. traza coroutine → Task → await;
2. reproduce async secuencial y concurrente;
3. demuestra que blocking call detiene progreso;
4. cancela y verifica cleanup;
5. provoca timeout sin assertion de duración exacta;
6. reproduce/refactoriza un race lógico;
7. prueba un límite de concurrencia;
8. demuestra backpressure por estado de Queue;
9. cierra producer/consumers sin orphans;
10. distingue transient retry de invalid input;
11. ordena partial results explícitamente;
12. ejecuta mini challenge en environment limpio.

PF-M8 deja EIDOLON 0.0a preparado para coordinar imports sintéticos sin convertir async en arquitectura distribuida.

---

## 34. Recursos de ampliación

La explicación fundamental está contenida aquí. Consulta [PF.11 Recursos recomendados](../../02_curriculum/01_programming_foundations.md#pf11-recursos-recomendados) y la documentación oficial de Python 3.14 para `asyncio`, especialmente coroutines/Tasks, TaskGroup, timeout, synchronization primitives y Queue.

Lee primero el contrato de lifecycle; después consulta detalles de parámetros y versiones. Evita copiar recipes que no declaren ownership, cancellation y capacity.

---

## 35. Límite del módulo

PF-M8 termina en async/await, Tasks, structured concurrency, cancellation, timeout, cleanup, blocking adapters acotados, races lógicos, bounded concurrency, Queue/backpressure, producer/consumer, shutdown, retry/idempotency mínimos, async iteration introductoria, determinismo y partial results.

PF-M9 enseñará pytest, debugging, logging y review avanzados. Tracks posteriores profundizarán networking, backend, databases, distributed systems, performance y security.

No se introducen custom event loops, implementación profunda de awaitables, threads/process pools avanzados, locks complejos, async decorators/context managers propios, networking, APIs, FastAPI, databases, distributed queues, brokers, circuit breakers, transactions distribuidas, Docker, LLMs ni observability avanzada.
