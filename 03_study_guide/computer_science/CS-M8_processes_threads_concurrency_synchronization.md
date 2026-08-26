# CS-M8 — Procesos, threads, concurrencia y sincronización

**Track:** Computer Science Foundations  
**Competencias:** D3.2; soporte D3.1  
**Fase:** P0  
**Nivel objetivo:** Aplicado-profesional  
**Prerequisites:** PF-M1–PF-M9, CS-M1–CS-M7  
**Build:** EIDOLON 0.0b  
**Curriculum source:** [CS-M8](../../02_curriculum/02_computer_science_foundations.md#cs-m8--procesos-threads-y-concurrencia)  
**Status:** review candidate

Tres trabajos de I/O pueden pasar gran parte del tiempo esperando. Tres cálculos CPU-bound pueden competir por CPU. Dos workers pueden leer el mismo contador y perder una actualización. Agregar concurrencia puede mejorar throughput o responsividad, pero también agrega overhead, sincronización, nondeterminism y failure modes.

La pregunta central no es “¿cómo hago todo concurrente?”, sino:

> ¿Qué estado existe, quién puede modificarlo, qué ordenamientos son posibles y qué garantías necesito?

```text
trabajo
↓
unidad de ejecución
↓
estado compartido o aislado
↓
interleavings posibles
↓
invariante
↓
sincronización
↓
failure mode
↓
medición
↓
elección de modelo
```

CS-M8 no introduce networking, bases de datos ni workers distribuidos. Trabaja con procesos, threads y datos sintéticos locales.

## Resultados de aprendizaje

Al terminar podrás:

- distinguir concurrency de parallelism y execution order de domain order;
- comparar process, thread y coroutine desde aislamiento, scheduling y ownership;
- enumerar interleavings y detectar race conditions sin depender de un test flaky;
- distinguir race condition de data race a nivel conceptual;
- declarar una invariante y delimitar su critical section;
- usar `Lock` como protocolo compartido y reducir su scope;
- reproducir circular wait sin colgar la ejecución y corregirlo con ordering;
- distinguir deadlock, starvation y livelock;
- elegir entre mutex, semaphore, event, condition y queue por el problema;
- implementar producer/consumer acotado con shutdown y accounting correctos;
- observar resultados y excepciones mediante `Future`;
- elegir `ThreadPoolExecutor` o `ProcessPoolExecutor` según workload y overhead;
- explicar el GIL de un build CPython sin convertirlo en mito de thread safety;
- demostrar process isolation e IPC explícito;
- diseñar cooperative cancellation y ownership mediante `join`;
- implementar un single-writer derived export con IDs y orden deterministas;
- medir baseline/threads/processes sin afirmar speedups universales;
- diagnosticar races/deadlocks con coordinación determinista, timeouts y evidencia.

## Cómo estudiar este módulo

1. Dibuja el estado y sus owners antes de crear workers.
2. Escribe la invariante antes del lock.
3. Enumera un interleaving roto antes de ejecutar.
4. Toda espera potencial tiene señal o timeout diagnóstico.
5. Toda unidad creada tiene owner, shutdown y `join`/`result`.
6. Compara siempre contra baseline secuencial.
7. Conserva source; los workers producen resultados derivados.

### Convenciones

- **Ejemplo ejecutable:** autónomo, acotado y con cleanup.
- **Script ejecutable:** debe guardarse como `.py`; process examples usan main guard.
- **Failure case controlado:** fuerza el orden con `Barrier`, `Event` o queue; no depende de azar.
- **Fragmento incorrecto:** ilustra un antipatrón y no se ejecuta como solución.
- **Benchmark educativo:** comprueba propiedades; sus tiempos no son universales.

Baseline: Python 3.14 y standard library. Los ejemplos de procesos eligen el start method `spawn` para revelar requisitos portables; deben ejecutarse desde archivos, no desde `python -c` ni un REPL interactivo.

---

## 1. Concurrency, parallelism y costo

**Concurrency** significa que varios trabajos progresan durante un periodo solapado. **Parallelism** significa ejecución simultánea real. No son sinónimos:

- un event loop puede intercalar coroutines en un solo thread;
- varios threads pueden progresar, aunque no ejecuten Python bytecode en paralelo en un build CPython con GIL habilitado;
- varios processes pueden ejecutar en cores distintos, sujeto al OS y hardware.

La concurrencia puede ocultar espera o mantener responsividad. También cuesta creación, scheduling, context switches, coordinación, memoria e IPC.

```text
throughput ≈ trabajos completados / tiempo transcurrido
speedup = tiempo_secuencial / tiempo_concurrente
```

Un speedup menor que 1 significa slowdown. La parte serial, la contention y el overhead limitan el máximo speedup; esta es la intuición útil de Amdahl, sin convertir CS-M8 en una derivación formal.

### Worker model

Clasifica: 100 lecturas blocking, una suma pequeña, 8 cálculos CPU intensivos y 1 000 operaciones con API async. ¿Cuál merece primero sync, threads, processes o asyncio? Declara qué medirías.

## 2. Process, thread y coroutine

Elige desde ownership y failure modes, no por una tabla memorizada:

| Unidad | Estado | Scheduling | Comunicación/costo |
|---|---|---|---|
| process | address space y recursos propios | OS, preemptive | IPC/serialización explícitos; mayor aislamiento |
| thread | comparte address space del process | OS, preemptive | shared objects directos; races posibles |
| coroutine | suele compartir process/thread | cooperativo en `await` | bajo costo; blocking code detiene su thread |

CS-M7 definió process y address space. PF-M8 definió coroutines, Tasks y cooperative cancellation. Aquí profundizamos threads/processes y solo usamos async como contraste.

Un thread puede ser preempted en puntos que el programador no escribió como `await`. Por eso dos líneas contiguas no constituyen una transaction. Un process aísla memoria, pero no evita races sobre files u otros recursos externos compartidos.

### Distingue

Dos processes escriben el mismo path y dos threads actualizan un dict. ¿Dónde existe shared state en cada caso?

## 3. Scheduling, interleaving y race condition

Un **interleaving** es un orden posible de pasos de varias unidades. Para `x = x + 1`, el modelo relevante puede expandirse:

```text
Thread A             Thread B
read x = 0
                     read x = 0
add 1
                     add 1
write x = 1
                     write x = 1
```

La invariante “dos increments aumentan en 2” se rompe: el resultado es 1.

Una **race condition** es una dependencia incorrecta del timing/interleaving. No exige dos writes simultáneos: check-then-act, read/use, file replacement y state transitions también pueden tener races. **Data race** es una categoría más estrecha de accesos conflictivos concurrentes a memoria bajo un memory model; aquí basta esta distinción conceptual.

**Failure case ejecutable y determinista:**

```python
from threading import Barrier, Thread

counter = {"value": 0}
both_read = Barrier(2)


def broken_increment() -> None:
    observed = counter["value"]
    both_read.wait(timeout=1)
    counter["value"] = observed + 1


threads = [Thread(target=broken_increment) for _ in range(2)]
for thread in threads:
    thread.start()
for thread in threads:
    thread.join(timeout=1)

assert all(not thread.is_alive() for thread in threads)
assert counter["value"] == 1
print("lost update reproduced: PASS")
```

`Barrier` fuerza que ambos lean antes de escribir; el test no espera que el scheduler “tenga suerte”.

### Interleaving

Traza dos órdenes que producen 2 y el orden forzado que produce 1. ¿Cuál es la mínima invariante rota?

## 4. Shared mutable state, ownership y message passing

El riesgo aumenta cuando el estado es:

```text
shared + mutable + accessed concurrently
```

Cuatro estrategias reducen la superficie:

1. un owner exclusivo modifica el estado;
2. datos inmutables cruzan fronteras;
3. workers envían mensajes/resultados;
4. una critical section sincroniza accesos inevitables.

Un lock no es la primera respuesta automática. Si cada worker calcula un resultado independiente y el owner los reúne, desaparece gran parte de la shared mutation.

### Ownership

Un índice `events_by_id` solo se actualiza en el coordinator; workers retornan pares `(event_id, event)`. ¿Qué race evitaste sin lock?

## 5. Check-then-act y atomicidad

Este código contiene una operación lógica multi-step:

```python
if key not in cache:
    cache[key] = compute()
```

Dos threads pueden pasar el check, calcular dos veces y producir effects duplicados. Ninguna línea Python debe tratarse como transaction por costumbre.

**Atomic** significa indivisible para observadores dentro de un modelo específico. La critical section debe abarcar la invariante, no una línea elegida al azar.

```text
invariante: un job_id tiene como máximo un resultado committed
critical section: check job_id + commit result
```

### Invariante

¿Basta bloquear solo `cache[key] = value`? Explica qué otro paso participa en “compute once”.

## 6. `Lock`: protocolo y critical section

`threading.Lock` implementa exclusión mutua dentro de un process. La forma segura de liberar ante exception es un context manager:

```python
from threading import Lock, Thread

counter = {"value": 0}
lock = Lock()


def increment_many() -> None:
    for _ in range(1_000):
        with lock:
            counter["value"] += 1


threads = [Thread(target=increment_many) for _ in range(2)]
for thread in threads:
    thread.start()
for thread in threads:
    thread.join(timeout=2)

assert all(not thread.is_alive() for thread in threads)
assert counter["value"] == 2_000
print("locked invariant: PASS")
```

El lock no “protege `counter`” mágicamente. La convención es que **toda** ruta que lea/modifique datos cuya consistencia depende de esta invariante usa el mismo lock. Un lock distinto por worker no crea exclusión mutua entre ellos; una lectura sin protocolo puede observar estado intermedio.

### Lock

Escribe la invariante, lista todas las rutas de acceso y señala cuál lock comparten. Si no puedes hacerlo, el protocolo no está completo.

## 7. Lock scope y contention

Un scope demasiado grande serializa trabajo que no necesita exclusión:

```python
# Código incorrecto: I/O lento dentro de la critical section.
with lock:
    result = slow_read(job)
    log_result(result)
    committed[job.id] = result
```

Refactor:

```python
# Fragmento: cálculo/I/O fuera; check+commit dentro.
result = slow_read(job)
with lock:
    if job.id in committed:
        raise ValueError("duplicate commit")
    committed[job.id] = result
```

Reducir scope baja contention, pero puede cambiar semántica: dos workers aún podrían calcular lo mismo. Si `slow_read` tiene effects, necesitas idempotency u ownership, no solo mover líneas.

### Modifica

Separa cálculo puro y commit en el ejemplo. ¿Qué duplicación toleras y cuál prohíbes?

## 8. Deadlock, circular wait y lock ordering

```text
Thread A holds L1 ──waits for──> L2
   ↑                              │
   └──────── Thread B holds L2 <──┘ waits for L1
```

El wait-for graph contiene un cycle. Las condiciones clásicas ayudan a razonar: mutual exclusion, hold-and-wait, no preemption y circular wait. Romper al menos una puede evitar este deadlock concreto.

**Failure case ejecutable y seguro:** ambos threads prueban el segundo lock sin bloquear y conservan el primero hasta que ambos intentaron.

```python
from queue import Queue
from threading import Barrier, Lock, Thread

left = Lock()
right = Lock()
ready = Barrier(2)
attempted = Barrier(2)
outcomes: Queue[bool] = Queue()


def attempt(first: Lock, second: Lock) -> None:
    with first:
        ready.wait(timeout=1)
        acquired = second.acquire(blocking=False)
        outcomes.put(acquired)
        attempted.wait(timeout=1)
        if acquired:
            second.release()


threads = [
    Thread(target=attempt, args=(left, right)),
    Thread(target=attempt, args=(right, left)),
]
for thread in threads:
    thread.start()
for thread in threads:
    thread.join(timeout=2)

assert all(not thread.is_alive() for thread in threads)
assert [outcomes.get(timeout=1) for _ in range(2)] == [False, False]
print("circular wait exposed without hang: PASS")
```

La corrección común es un orden global: siempre `left` antes de `right`. Funciona solo si todas las rutas lo respetan. Timeouts evitan espera infinita y aportan evidencia; no restauran por sí solos consistencia.

### Deadlock

Dibuja el cycle. ¿Qué condición rompe el orden global? ¿Qué evidencia registrarías si un acquire agota timeout?

## 9. Deadlock, starvation, livelock y reentrancy

- **deadlock:** un conjunto espera en cycle y ninguno progresa;
- **starvation:** una unidad espera indefinidamente mientras otras sí progresan;
- **livelock:** unidades siguen actuando/reintentando, pero no avanzan hacia el objetivo.

`RLock` permite que el mismo thread adquiera el mismo lock varias veces y exige releases correspondientes. Es útil en APIs reentrantes justificadas; no debe ocultar ownership confuso.

Fairness no está garantizada de la misma forma por toda primitive/plataforma. CS-M8 reconoce starvation, no diseña schedulers.

### Distingue

Dos workers se ceden prioridad sin terminar; un worker nunca obtiene permiso; dos locks forman cycle. Clasifica los tres.

## 10. Semaphore, Event y Condition

Un mutex admite aproximadamente un holder; un `Semaphore(N)` administra N permisos. Sirve para limitar acceso concurrente, no para proteger automáticamente una invariante multi-step.

```python
from threading import BoundedSemaphore, Lock, Thread

limit = BoundedSemaphore(2)
state_lock = Lock()
active = 0
peak = 0


def use_resource() -> None:
    global active, peak
    with limit:
        with state_lock:
            active += 1
            peak = max(peak, active)
        with state_lock:
            active -= 1


threads = [Thread(target=use_resource) for _ in range(5)]
for thread in threads:
    thread.start()
for thread in threads:
    thread.join(timeout=1)

assert all(not thread.is_alive() for thread in threads)
assert peak <= 2
print("bounded permits: PASS")
```

`Event` expresa una señal boolean-like: workers pueden esperar hasta `set()`. `Condition` coordina un predicate asociado a estado y un lock; `wait()` libera el lock mientras espera. Al despertar se reevalúa el predicate con `while`, porque el despertar puede ser espurio o el estado puede haber cambiado antes de recuperar el lock. Una notificación no garantiza por sí sola que el predicate sea verdadero.

```python
# Patrón de Condition; fragmento.
with condition:
    while not items:
        condition.wait()
    item = items.pop(0)
```

### Invariante

¿Qué limita el semaphore y qué protege `state_lock` en el ejemplo? No intercambies sus contratos.

## 11. Cooperative cancellation con `Event`

Python no ofrece una operación general segura para matar arbitrariamente un thread desde fuera. El worker debe alcanzar checkpoints y observar una señal.

```python
from threading import Event, Thread

stop = Event()
observed: list[str] = []


def worker() -> None:
    while not stop.wait(timeout=0.01):
        observed.append("tick")
    observed.append("stopped")


thread = Thread(target=worker, name="cooperative-worker")
thread.start()
stop.set()
thread.join(timeout=1)

assert not thread.is_alive()
assert observed[-1] == "stopped"
print("cooperative shutdown: PASS")
```

`sleep(0.1)` no es synchronization: no expresa qué evento esperas. `while not ready: pass` hace busy waiting. Usa `join`, `Event`, queue o `Condition`.

Terminar un process a la fuerza es posible mediante APIs específicas, pero puede saltarse `finally`, flush y protocolos de commit. “Pude terminarlo” no significa “el estado quedó consistente”.

### Cancellation

¿Quién hace `set`, quién observa, quién hace `join` y qué cleanup ejecuta el worker antes de salir?

## 12. Producer/consumer y `queue.Queue`

`queue.Queue` ofrece coordinación thread-safe de alto nivel. `maxsize` acota backlog; no uses una list + lock manual si necesitas `put/get`, espera, accounting y shutdown.

```text
producers
↓ Queue(maxsize=N) — backpressure al llenarse
consumers
↓ task_done por cada get
owner espera queue.join
```

**Ejemplo ejecutable:**

```python
from queue import Queue
from threading import Thread

jobs: Queue[int | None] = Queue(maxsize=2)
results: list[int] = []


def consumer() -> None:
    while True:
        item = jobs.get()
        try:
            if item is None:
                return
            results.append(item * 2)
        finally:
            jobs.task_done()


thread = Thread(target=consumer)
thread.start()
for value in (1, 2, 3):
    jobs.put(value)
jobs.put(None)
jobs.join()
thread.join(timeout=1)

assert not thread.is_alive()
assert results == [2, 4, 6]
print("producer/consumer: PASS")
```

`task_done()` corresponde exactamente a un `get()` completado, incluso para sentinel. `join()` de Queue espera accounting cero; `Thread.join()` espera lifecycle del thread. Son contratos distintos.

### Backpressure

¿Qué limita `maxsize=2`: workers activos, backlog pendiente o ambos? ¿Qué limita la cantidad de consumers?

## 13. Shutdown, daemon threads y ownership

Un shutdown ordenado:

1. deja de admitir trabajo;
2. solicita cancellation o añade un sentinel por consumer;
3. espera queue accounting;
4. hace `join` de workers;
5. observa/reporta failures;
6. confirma cleanup de resources.

Un daemon thread no mantiene vivo el process. El intérprete puede terminar con ese thread aún activo; no lo uses para writes de journal, flush o cleanup crítico. “Fire-and-forget” no define quién observa failure ni cuándo termina.

### Ownership

Un producer creó tres workers y envió un solo sentinel. ¿Qué workers pueden quedar vivos? Escribe la política correcta.

## 14. Exceptions en threads y `Future`

Una exception en un `Thread` manual no se propaga al creator como un return normal. Debes capturarla/reportarla o usar una abstracción que el owner observe.

Un `Future` representa trabajo pending/running/done y contiene result o exception. `result()` espera y vuelve a lanzar la exception del worker.

```python
from concurrent.futures import ThreadPoolExecutor


def parse_job(job_id: str) -> str:
    if job_id == "job-bad":
        raise ValueError("invalid synthetic job")
    return job_id.upper()


with ThreadPoolExecutor(max_workers=2) as executor:
    future = executor.submit(parse_job, "job-bad")
    try:
        future.result(timeout=1)
    except ValueError as error:
        assert str(error) == "invalid synthetic job"
    else:
        raise AssertionError("worker exception was hidden")

print("Future propagated exception: PASS")
```

Submit y olvidar el `Future` puede ocultar errores. El executor context manager hace shutdown y espera su trabajo según contrato, pero el owner aún debe consumir resultados/exceptions.

## 15. Thread pools e I/O-bound work

`ThreadPoolExecutor` es útil para funciones síncronas I/O-bound o APIs blocking cuando un pool controlado tiene sentido. `max_workers` limita threads activos del pool; **no garantiza por sí solo que enviar millones de futures mantenga un backlog acotado**. Usa batches, admission control o queue explícita.

**Benchmark educativo:**

```python
from concurrent.futures import ThreadPoolExecutor
from time import perf_counter, sleep


def simulated_io(value: int) -> int:
    sleep(0.02)
    return value * 2


inputs = list(range(4))
started = perf_counter()
sequential = [simulated_io(value) for value in inputs]
sequential_elapsed = perf_counter() - started

started = perf_counter()
with ThreadPoolExecutor(max_workers=2) as executor:
    threaded = list(executor.map(simulated_io, inputs))
threaded_elapsed = perf_counter() - started

assert threaded == sequential == [0, 2, 4, 6]
assert sequential_elapsed >= 0 and threaded_elapsed >= 0
print("I/O model measured")
```

`sleep` simula espera; no demuestra rendimiento de disco o red. `executor.map` devuelve resultados en orden de input, no completion order.

### Benchmark

Registra ambos tiempos, workers y tamaño. ¿Qué resultado invalidaría la elección de threads para este workload real?

## 16. Completion order y domain order

`executor.map(function, inputs)` itera resultados en el orden de inputs. `as_completed(futures)` entrega futures conforme terminan; ese orden puede variar.

```python
from concurrent.futures import ThreadPoolExecutor, as_completed
from time import sleep


def delayed(sequence: int) -> tuple[int, str]:
    sleep(0.005 * (2 - sequence))
    return sequence, f"result-{sequence}"


with ThreadPoolExecutor(max_workers=3) as executor:
    futures = [executor.submit(delayed, sequence) for sequence in range(3)]
    completion_order = [future.result() for future in as_completed(futures)]

domain_order = sorted(completion_order, key=lambda item: item[0])
assert {item[0] for item in completion_order} == {0, 1, 2}
assert domain_order == [(0, "result-0"), (1, "result-1"), (2, "result-2")]
print("domain order explicit: PASS")
```

No afirmamos un completion order exacto. EIDOLON conserva sequence/job ID y codifica domain order antes de escribir output determinista.

### Ordering

¿Por qué ordenar logs por la línea impresa no reconstruye causalidad perfecta? ¿Qué IDs/timestamps de dominio necesitas?

## 17. Process isolation e IPC

Cada process tiene su address space. Un child que modifica su copia de un dict no modifica automáticamente el objeto del parent. Para comunicar resultados usa return values, pipes o process-safe queues; shared memory reintroduce sincronización y queda como preview.

**Script ejecutable — `process_isolation.py`:**

```python
from multiprocessing import get_context


def child_update(original: dict[str, int], output) -> None:
    original["value"] = 99
    output.put(original["value"])


def main() -> None:
    context = get_context("spawn")
    output = context.Queue()
    parent_state = {"value": 1}
    process = context.Process(target=child_update, args=(parent_state, output))
    process.start()
    try:
        child_value = output.get(timeout=2)
    finally:
        process.join(timeout=2)
        if process.is_alive():  # cleanup diagnóstico; no garantiza consistencia de effects
            process.terminate()
            process.join(timeout=2)

    assert not process.is_alive()
    assert process.exitcode == 0
    assert child_value == 99
    assert parent_state == {"value": 1}
    output.close()
    output.join_thread()
    print("process isolation: PASS")


if __name__ == "__main__":
    main()
```

Con `spawn`, el child importa el módulo. El main guard evita crear processes recursivamente. Funciones top-level suelen ser más portables/picklable que closures o lambdas. Los start methods varían por plataforma; no dependas accidentalmente de `fork`. Copy-on-write es una optimización posible de algunos Unix-like start methods, no una garantía portable.

## 18. Process pools, CPU-bound work y serialización

`ProcessPoolExecutor` ofrece processes separados y puede habilitar parallelism CPU-bound. Paga startup, memoria, scheduling, IPC y serialization. Enviar objetos enormes o trabajos diminutos puede costar más que calcularlos.

**Script ejecutable — `process_pool_demo.py`:**

```python
from concurrent.futures import ProcessPoolExecutor
from multiprocessing import get_context


def count_modulo(limit: int) -> int:
    return sum(value % 7 for value in range(limit))


def main() -> None:
    inputs = [20_000, 25_000, 30_000]
    sequential = [count_modulo(value) for value in inputs]

    with ProcessPoolExecutor(
        max_workers=2,
        mp_context=get_context("spawn"),
    ) as executor:
        futures = [executor.submit(count_modulo, value) for value in inputs]
        parallel = [future.result(timeout=5) for future in futures]

    assert parallel == sequential
    print("process pool results: PASS")


if __name__ == "__main__":
    main()
```

El ejemplo verifica resultados y portabilidad del shape, no speedup: este input pequeño puede ser más lento en pool. Algunas APIs de multiprocessing serializan con pickle; nunca cargues pickle no confiable como dato seguro.

### Worker model

¿Qué tamaño mínimo vuelve útil el process pool en tu máquina? Responde con benchmark, no con `os.cpu_count()` como receta automática.

## 19. GIL: precisión para CPython 3.14

El GIL pertenece a builds/implementaciones, no al contrato eterno del lenguaje Python.

En un **build tradicional de CPython**, el Global Interpreter Lock limita la ejecución simultánea de Python bytecode por múltiples threads dentro de un process. Threads siguen siendo útiles durante blocking I/O; extensiones pueden liberar el GIL. El GIL:

- no elimina race conditions;
- no vuelve thread-safe una invariante multi-step;
- no sustituye locks;
- no significa que exista un solo thread;
- no garantiza que una secuencia de líneas sea atomic.

CPython 3.14 también admite builds free-threaded opcionales. Esos builds cambian assumptions de ejecución y compatibilidad; no convierten automáticamente una aplicación mal sincronizada en correcta. El baseline local verificado para este módulo fue CPython 3.14.7 con GIL habilitado, pero el estudiante debe inspeccionar su propio build.

**Diagnóstico ejecutable:**

```python
import sys
import sysconfig

assert sys.implementation.name == "cpython"
free_threaded_build = bool(sysconfig.get_config_var("Py_GIL_DISABLED"))
is_gil_enabled = getattr(sys, "_is_gil_enabled", None)

if is_gil_enabled is not None:
    assert isinstance(is_gil_enabled(), bool)
assert isinstance(free_threaded_build, bool)
print("CPython build inspected")
```

Output estable:

```text
CPython build inspected
```

En un build tradicional, threads pure-Python CPU-bound no suelen escalar sobre múltiples cores como un process pool. No extrapoles esto a toda extensión, implementación o build.

### Race

Refuta: “`dict` no necesita protocolo porque el GIL protege cada línea”. ¿Qué pasa con check + compute + commit?

## 20. Thread-safe no significa process-safe

`threading.Lock` coordina threads que comparten ese objeto dentro de un process. No sincroniza automáticamente otro process. Para processes usa IPC/primitives diseñadas para esa frontera o evita shared mutation.

Dos workers —threads o processes— que escriben el mismo file pueden interleave, perder updates o producir ordering nondeterminista. Append no convierte múltiples records en transaction. CS-M7 ya estableció atomicity/durability; CS-M8 añade ownership.

```text
many workers
↓ derived results queue
single writer
↓ one commit/order policy
derived output
```

Single writer reduce synchronization del filesystem, pero puede ser bottleneck y necesita bounded input, error propagation y shutdown. No es dogma.

## 21. Jobs, retries, idempotency y states

Un job local puede seguir:

```text
PENDING → RUNNING → SUCCEEDED
                  ↘ FAILED
                  ↘ CANCELLED
```

Si hay retry, define stable `job_id`. **At-most-once** puede perder trabajo; **at-least-once** puede duplicar effects. “Exactly once” no aparece por añadir un flag: depende de la frontera de commit y effects.

Invariante EIDOLON educativa:

> Cada `job_id` tiene como máximo un resultado derived committed.

El writer rechaza duplicate commits. Source permanece intacto; derived output puede reconstruirse. Un retry de cálculo puro es distinto de repetir un append externo.

### Invariante

¿En qué punto cambia `RUNNING → SUCCEEDED`: al terminar compute, al entrar a queue o después del commit del writer?

## 22. Importación EIDOLON con single writer

El siguiente ejemplo usa threads porque el trabajo simula I/O blocking. Workers producen resultados; un writer posee el derived index. Queues acotadas limitan backlog y sentinels cierran cada stage.

**Ejemplo ejecutable integrado:**

```python
from queue import Queue
from threading import Thread
from time import sleep

jobs: Queue[tuple[int, str] | None] = Queue(maxsize=2)
results: Queue[tuple[str, int, str, str] | None] = Queue(maxsize=2)
committed: dict[str, tuple[int, str]] = {}
errors: list[str] = []


def worker() -> None:
    while True:
        item = jobs.get()
        try:
            if item is None:
                return
            sequence, source_id = item
            sleep(0.002)
            results.put(("ok", sequence, source_id, source_id.upper()))
        except Exception as error:
            results.put(("error", sequence, source_id, repr(error)))
        finally:
            jobs.task_done()


def writer() -> None:
    while True:
        item = results.get()
        try:
            if item is None:
                return
            status, sequence, job_id, value = item
            if status == "error":
                errors.append(value)
                continue
            if job_id in committed:
                errors.append(f"duplicate commit: {job_id}")
                continue
            committed[job_id] = (sequence, value)
        finally:
            results.task_done()


source_jobs = ((2, "src-2"), (0, "src-0"), (1, "src-1"))
source_snapshot = tuple(source_jobs)
workers = [Thread(target=worker, name=f"worker-{index}") for index in range(2)]
writer_thread = Thread(target=writer, name="single-writer")

writer_thread.start()
for thread in workers:
    thread.start()
for item in source_jobs:
    jobs.put(item)
for _ in workers:
    jobs.put(None)

jobs.join()
for thread in workers:
    thread.join(timeout=1)
results.put(None)
results.join()
writer_thread.join(timeout=1)

ordered = [
    (job_id, value)
    for job_id, (_, value) in sorted(committed.items(), key=lambda item: item[1][0])
]

assert all(not thread.is_alive() for thread in workers)
assert not writer_thread.is_alive()
assert tuple(source_jobs) == source_snapshot
assert errors == []
assert ordered == [("src-0", "SRC-0"), ("src-1", "SRC-1"), ("src-2", "SRC-2")]
print("bounded single writer: PASS")
```

Execution order puede variar; sequence codifica domain order. Para un derived JSONL real, el writer aplicaría el protocolo temporal/replace de CS-M7 y nunca sobrescribiría source.

### Ordering

Inserta un duplicate `src-1`. ¿Qué debe rechazar el writer y qué datos conservar para diagnosticarlo?

## 23. Elección: sync, asyncio, threads o processes

Pregunta en este orden:

1. ¿Una solución sync cumple latency/throughput?
2. ¿El workload espera I/O o consume CPU?
3. ¿La library es async o blocking?
4. ¿Necesitas shared memory o prefieres aislamiento?
5. ¿Cuánto cuestan startup, serialization y coordination?
6. ¿Cómo se cancelan y poseen los workers?
7. ¿Qué evidencia del benchmark cambiaría la decisión?

- **sync:** menor complejidad para trabajo pequeño o secuencial;
- **asyncio:** muchas operaciones I/O con APIs async y scheduling cooperativo;
- **threads:** blocking I/O/libraries sync, con shared-state discipline;
- **processes:** CPU-bound parallelism/aislamiento, pagando IPC y startup.

Sistemas híbridos existen, pero cada modelo añadido multiplica failure modes. EIDOLON P0 prefiere el mínimo que satisfaga mediciones.

### Worker model

Una API async llama una library legacy blocking; un parser pure-Python consume CPU; un export pequeño tarda 3 ms. Elige y justifica sin usar moda.

## 24. Benchmarking y oversubscription

Un experimento serio registra:

- baseline secuencial;
- workload CPU-bound/I/O-bound y tamaño;
- workers y `os.cpu_count()` observado;
- startup/warmup y número de repeticiones;
- wall time, throughput y memoria relevante;
- serialization/copias;
- error/cancellation behavior;
- determinism del output.

`os.cpu_count()` es información, no número óptimo automático. Más workers pueden empeorar context switching, memory, contention y cache behavior. Para asyncio, compara solo si existe una implementación equivalente y no mezcles API async real con sleeps incompatibles.

### Benchmark

Un pool de 32 processes pierde contra sync para 32 trabajos de 1 ms. Enumera overheads y diseña tamaños crecientes para encontrar el crossover.

## 25. Testing y debugging sin sleeps frágiles

Usa `Event`, `Barrier`, queue y timeouts para forzar estados. Cada test potencialmente bloqueante debe terminar con timeout y afirmar que ningún worker quedó vivo.

Recoge:

- thread/process name y PID cuando aplique;
- `job_id`, `operation_id`, state y sequence;
- checkpoint/timestamp monotónico para intervalos;
- lock esperado/poseído conceptualmente;
- queue sizes/capacity y Future state;
- exception/exit code sin payload sensible.

Logs de workers pueden intercalarse y no constituyen causalidad perfecta. IDs y state transitions permiten reconstruir mejor el caso.

### Diagnóstico

Un test pasa con `sleep(1)` y falla con `sleep(0.1)`. Sustituye tiempo supuesto por una señal de estado observable.

## 26. Catálogo de failure cases

| Fallo | Por qué ocurre | Corrección |
|---|---|---|
| shared counter sin lock | read/add/write interleaved | ownership o lock de la invariante |
| lock distinto por worker | no hay mutex compartido | mismo lock/protocolo |
| write protegido, read libre | reader rompe consistencia | todas las rutas relevantes siguen protocolo |
| orden inverso de locks | circular wait | ordering/rediseño/timeout diagnóstico |
| `sleep` como coordinación | depende de timing | Event/Barrier/join/queue |
| busy waiting | quema CPU sin señal | primitive bloqueante |
| daemon escribe journal | process puede salir antes de cleanup | thread no daemon + shutdown/join |
| Future olvidado | exception no observada por owner | guardar y consumir result/exception |
| lambda/closure en process pool | serialización/import varía | worker top-level + main guard |
| multiprocessing sin guard | spawn reimporta y puede recursar | `if __name__ == "__main__"` |
| payload enorme al pool | IPC/serialization domina | medir, chunking o cambiar ownership |
| “GIL = thread-safe” | invariante multi-step sigue vulnerable | protocolo explícito |
| terminate = cleanup | salta finally/flush | cooperative shutdown o recovery |
| concurrent journal writers | interleaving/lost update/order | single writer o protocolo probado |
| map = completion order | `map` preserva input order | `as_completed` para completion; sequence para domain |
| demasiados workers | contention/memory/context switches | benchmark y límite |

## 27. Ejercicios guiados

### Guiado 1 — Crea y posee dos threads

```python
from queue import Queue
from threading import Thread

seen: Queue[str] = Queue()
threads = [Thread(target=lambda name=name: seen.put(name)) for name in ("A", "B")]
for thread in threads:
    thread.start()
for thread in threads:
    thread.join(timeout=1)
assert all(not thread.is_alive() for thread in threads)
assert {seen.get(timeout=1) for _ in range(2)} == {"A", "B"}
```

No fijes order; el owner conserva referencias y hace join.

### Guiado 2 — Observa interleaving

Usa dos `Event`: A registra `A1`, habilita B; B registra `B1`, habilita A; A registra `A2`. La solución aprueba con `A1,B1,A2` sin sleeps.

### Guiado 3 — Modela race explícito

Ejecuta el `Barrier` de la sección 3. Dibuja ambos reads antes de writes y afirma final 1.

### Guiado 4 — Protege critical section

Ejecuta el contador con `Lock` de la sección 6. El criterio es final 2 000 y threads terminados.

### Guiado 5 — Reduce lock scope

Mueve compute puro fuera; mantén check+commit dentro. Explica si duplicate compute es tolerable.

### Guiado 6 — Construye circular wait seguro

Ejecuta el non-blocking attempt de la sección 8. El criterio es cycle razonado y suite sin hang.

### Guiado 7 — Corrige con lock ordering

```python
from threading import Lock, Thread

first = Lock()
second = Lock()
completed: list[int] = []


def ordered(worker_id: int) -> None:
    with first:
        with second:
            completed.append(worker_id)


threads = [Thread(target=ordered, args=(worker_id,)) for worker_id in (1, 2)]
for thread in threads:
    thread.start()
for thread in threads:
    thread.join(timeout=1)
assert all(not thread.is_alive() for thread in threads)
assert set(completed) == {1, 2}
```

Todas las rutas adquieren el mismo orden.

### Guiado 8 — Event para shutdown

Ejecuta la sección 11. Añade cleanup en `finally` y comprueba que ocurrió antes de join.

### Guiado 9 — Producer/consumer

Ejecuta la sección 12. Explica un `task_done` por `get`, incluido sentinel.

### Guiado 10 — Demuestra bounded queue

```python
from queue import Full, Queue

queue: Queue[int] = Queue(maxsize=1)
queue.put(1)
try:
    queue.put(2, block=False)
except Full:
    pass
else:
    raise AssertionError("capacity was ignored")
assert queue.get() == 1
queue.task_done()
queue.join()
```

La evidencia es state/capacity, no duración.

### Guiado 11 — ThreadPoolExecutor

Mapea tres inputs con `max_workers=2`, observa todos los results y compara con baseline.

### Guiado 12 — Exception en Future

Ejecuta la sección 14. La solución debe observar el `ValueError`; no basta terminar el executor.

### Guiado 13 — ProcessPoolExecutor

Guarda y ejecuta `process_pool_demo.py`. Verifica results/exit limpio, sin exigir speedup.

### Guiado 14 — Main guard

Explica qué top-level code ejecuta un child con spawn. Mueve creación del pool dentro de `main()` y llama solo bajo guard.

### Guiado 15 — Compara CPU-bound

Mide `count_modulo` secuencial y process pool con tamaños crecientes. Registra crossover, startup y workers; no concluyas desde un tamaño.

### Guiado 16 — Compara I/O-bound

Ejecuta la sección 15 con 1, 2 y 4 workers. Compara outputs y timings; identifica que `sleep` es simulación.

### Guiado 17 — Single writer

Ejecuta la sección 22. El criterio es source intacto, queues cuadradas, workers cerrados y commits únicos.

### Guiado 18 — Orden determinista

Baraja delays, conserva sequence y ordena antes de serializar. El output debe ser igual en cinco ejecuciones.

### Guiado 19 — Failure injection

Haz que un job específico lance `ValueError`. El owner debe observarlo, dejar de admitir trabajo según policy, cerrar workers y no commit un success ficticio.

### Guiado 20 — Benchmark y decisión

Entrega tabla sync/threads/processes con workload, N, workers, mediana/rango, throughput, errors y memory note. Aprueba si la decisión cita evidencia y límites, aunque sync gane.

## 28. Ejercicios independientes

1. Compara process/thread/coroutine para un workload propio.
2. Enumera cuatro interleavings de dos operaciones read-modify-write.
3. Fuerza un lost update con Barrier.
4. Rediseña el mismo caso mediante single owner sin lock.
5. Identifica critical section de check+commit.
6. Demuestra que dos locks distintos no excluyen.
7. Añade una lectura sin lock y explica la ruptura.
8. Reduce un lock que contiene I/O lento.
9. Construye wait-for graph de tres locks.
10. Corrige el cycle con global ordering.
11. Distingue deadlock, starvation y livelock en tres trazas.
12. Limita dos recursos con semaphore y mide peak activo.
13. Implementa Condition con predicate recheck en `while`.
14. Implementa shutdown con Event y cleanup.
15. Implementa Queue bounded con dos consumers y dos sentinels.
16. Verifica accounting ante una exception de consumer.
17. Reemplaza sleep de coordinación por Event.
18. Demuestra el riesgo de un daemon sin escribir datos reales.
19. Observa y clasifica una exception de Future.
20. Compara `map` y `as_completed` sin fijar completion order.
21. Ejecuta process isolation con spawn.
22. Envía un resultado por multiprocessing Queue.
23. Explica el costo de serializar un payload creciente.
24. Sustituye closure de process worker por función top-level.
25. Inspecciona GIL/build sin inferir thread safety.
26. Compara CPU-bound en threads y processes.
27. Compara I/O-bound sync y threads.
28. Diseña cooperative cancellation para trabajo por chunks.
29. Explica qué podría perder termination forzada.
30. Implementa single writer con duplicate detection.
31. Codifica domain order independiente de completion.
32. Modela job states y transitions válidas.
33. Compara at-most-once y at-least-once para un effect local.
34. Encuentra oversubscription con tamaños crecientes.
35. Redacta un diagnóstico con IDs sin payload sensible.

## 29. Preguntas conceptuales

1. ¿Qué diferencia existe entre concurrency y parallelism?
2. ¿Qué comparte un thread y qué aísla un process?
3. ¿Cómo difiere preemption de cooperative scheduling?
4. ¿Por qué una race existe aunque cada línea parezca simple?
5. ¿Qué distingue race condition de data race aquí?
6. ¿Qué significa atomic dentro de un modelo?
7. ¿Qué protege realmente un lock?
8. ¿Qué diferencia hay entre atomic operation y critical section?
9. ¿Por qué el lock debe ser compartido?
10. ¿Qué costo agrega un lock scope grande?
11. ¿Cómo aparece un deadlock como cycle?
12. ¿Qué diferencia deadlock de starvation y livelock?
13. ¿Cuándo RLock aporta y cuándo oculta diseño?
14. ¿Por qué semaphore no es “otro lock”?
15. ¿Por qué Condition reevalúa con `while`?
16. ¿Por qué Queue es preferible a list compartida para producer/consumer?
17. ¿Qué diferencia `Queue.join` de `Thread.join`?
18. ¿Por qué daemon thread es peligroso para writes críticos?
19. ¿Por qué no puedes matar un thread arbitrariamente de forma segura?
20. ¿Cómo observa el owner una exception de worker?
21. ¿Qué estados representa un Future?
22. ¿Cuándo ThreadPoolExecutor tiene sentido?
23. ¿Cuándo ProcessPoolExecutor tiene sentido?
24. ¿Qué hace y qué no hace el GIL?
25. ¿Por qué el GIL no evita race conditions?
26. ¿Qué cambia en un build CPython free-threaded?
27. ¿Por qué multiprocessing añade IPC/serialization cost?
28. ¿Por qué necesitas main guard con spawn?
29. ¿Por qué thread-safe no implica process-safe?
30. ¿Por qué append no resuelve concurrent writers?
31. ¿Cómo simplifica single writer y qué bottleneck agrega?
32. ¿Qué diferencia completion order de domain order?
33. ¿Por qué exactly-once no aparece con un flag?
34. ¿Qué evidencia distingue CPU-bound de I/O-bound?
35. ¿Por qué más workers puede ser más lento?

## 30. Mini challenge — Coordinador local de jobs EIDOLON

### Objetivo y artefactos

Construye un coordinador reproducible con standard library:

```text
cs_m8_challenge/
├── coordinator.py
├── workers.py
├── synthetic_source.jsonl
├── derived_results.jsonl
└── BENCHMARK.md
```

Source es sintético e inmutable durante el experimento; output es derived.

### A. Baseline secuencial

1. Genera N jobs con stable `job_id` y sequence.
2. Procesa secuencialmente y mide wall time/throughput.
3. Conserva results esperados para comparar correctness.

### B. Thread pool I/O-bound

4. Usa función blocking sintética acotada.
5. Limita workers y backlog; no submit ilimitado.
6. Conserva cada Future y observa result/exception.
7. Inyecta un failure determinista por `job_id`.
8. Define cooperative shutdown y espera todos los owners.
9. Reconstruye output por sequence, no completion timing.

### C. Process pool CPU-bound

10. Worker es función top-level importable.
11. Pool se crea bajo main guard y usa estrategia portable documentada.
12. Compara baseline con 1/2/... workers acotados.
13. Mide startup/serialization con inputs pequeños y mayores.
14. No exige que process pool gane.

### D. Shared-state failure

15. Fuerza lost update con Barrier.
16. Corrige primero mediante un owner/message passing.
17. Implementa también versión con shared Lock y compara complejidad.

### E. Deadlock seguro

18. Construye circular wait con non-blocking acquire o timeouts/barriers.
19. Ningún test puede quedar colgado.
20. Corrige con global lock order o un único owner.

### F. Single writer

21. Workers nunca escriben source ni derived final.
22. Envían resultados a Queue bounded.
23. Un writer aplica check+commit por stable `job_id`.
24. Duplicate commit se rechaza y reporta.
25. Writer ordena por sequence y usa protocolo safe-derived de CS-M7.
26. Source permanece byte por byte intacto.

### G. Benchmark report

27. Compara sync, threads y processes por workload apropiado.
28. Registra N, workers, CPU count observado, wall time, throughput y memoria aproximada.
29. Discute overhead, GIL/build, IPC, determinism, failures y cleanup.
30. Explica qué dato cambiaría la elección.

### Comprobaciones contractuales

**Continuación — adapta nombres:**

```python
source_before = source_path.read_bytes()

sequential = run_sequential(jobs)
threaded = run_threaded(jobs, max_workers=2, max_pending=4)
processed = run_processes(jobs, max_workers=2)

assert normalize(threaded) == normalize(sequential)
assert normalize(processed) == normalize(sequential)
assert source_path.read_bytes() == source_before
assert all_workers_stopped()

commits = single_writer(threaded)
assert len(commits) == len({result.job_id for result in threaded})
assert commits == sorted(commits, key=lambda result: result.sequence)

try:
    single_writer([threaded[0], threaded[0]])
except ValueError:
    pass
else:
    raise AssertionError("duplicate commit accepted")

assert source_path.read_bytes() == source_before
```

### Failure cases obligatorios

- deterministic lost update y corrección;
- circular wait observado sin hang y corrección;
- Future failure propagado;
- cooperative shutdown confirmado;
- process worker no portable corregido a top-level/main guard;
- duplicate commit rechazado;
- source intacto tras failure;
- completion order distinto de domain order;
- un caso donde overhead vuelve peor al modelo concurrente.

### Criterio de aprobación

- todo worker tiene owner, límite, failure policy y shutdown;
- races/deadlocks se reproducen sin azar ni hangs;
- locks protegen invariantes con scope justificado;
- thread/process APIs se eligen por workload medido;
- main guard y workers top-level permiten spawn;
- Futures/exceptions nunca se abandonan;
- single writer conserva source y determina order;
- benchmark incluye baseline y no universaliza resultados;
- no aparecen networking, database, backend, brokers, Docker ni AI.

## 31. Resumen

- Concurrency es progreso solapado; parallelism es ejecución simultánea.
- Processes aíslan address spaces; threads comparten; coroutines ceden cooperativamente.
- Scheduling produce interleavings; una race depende incorrectamente de ellos.
- Shared mutable state se reduce con ownership, immutability o message passing.
- Atomicidad depende del modelo; una línea Python no es transaction.
- Un lock protege una invariante solo si todas las rutas comparten protocolo.
- Critical sections pequeñas reducen contention, pero no corrigen effects duplicados por sí solas.
- Deadlock aparece como circular wait; ordering rompe ese cycle bajo disciplina global.
- Starvation y livelock no son deadlock.
- Semaphore limita permisos; Event señala; Condition espera predicates; Queue coordina items.
- Shutdown requiere señal, accounting, join y failure observation.
- Daemon threads no son owners de writes críticos.
- Future transporta result o exception al owner.
- Thread pools sirven a blocking I/O; process pools pueden servir CPU-bound con overhead.
- El GIL es propiedad de ciertos builds CPython, no mutex de aplicación.
- Processes requieren IPC/serialization; main guard evita problemas con spawn.
- Thread-safe no implica process-safe.
- Single writer simplifica commits locales, pero requiere capacity/order/failure policy.
- Execution/completion order no define domain order.
- Benchmark secuencial y workload real deciden; más workers no implica más rapidez.

## 32. Checklist de dominio

- [ ] Puedo distinguir concurrency y parallelism.
- [ ] Puedo comparar process, thread y coroutine.
- [ ] Puedo explicar preemption y cooperative scheduling.
- [ ] Puedo enumerar interleavings posibles.
- [ ] Puedo reproducir una race determinista.
- [ ] Puedo distinguir race condition y data race conceptualmente.
- [ ] Puedo declarar una invariante y su critical section.
- [ ] Puedo usar el mismo Lock en todas las rutas relevantes.
- [ ] Puedo reducir lock scope y explicar tradeoffs.
- [ ] Puedo dibujar un wait-for cycle.
- [ ] Puedo evitar hang con timeouts/coordinación.
- [ ] Puedo aplicar global lock ordering.
- [ ] Puedo distinguir deadlock/starvation/livelock.
- [ ] Puedo elegir mutex/semaphore/Event/Condition.
- [ ] Puedo usar Queue bounded con accounting exacto.
- [ ] Puedo cerrar workers con sentinel/Event y join.
- [ ] Puedo rechazar daemon para writes críticos.
- [ ] Puedo observar exceptions mediante Future.
- [ ] Puedo elegir thread/process pool por workload.
- [ ] Puedo explicar GIL tradicional y build free-threaded.
- [ ] Puedo demostrar process isolation e IPC.
- [ ] Puedo usar worker top-level y main guard.
- [ ] Puedo explicar serialization/startup overhead.
- [ ] Puedo diseñar cooperative cancellation.
- [ ] Puedo explicar por qué terminate no asegura cleanup.
- [ ] Puedo separar thread-safe de process-safe.
- [ ] Puedo implementar single writer y duplicate detection.
- [ ] Puedo codificar domain order explícito.
- [ ] Puedo mantener source intacto y derived rebuildable.
- [ ] Puedo comparar baseline/threads/processes con límites.
- [ ] Puedo resolver el mini challenge con PF + CS-M1–CS-M8.

## 33. Preparación para labs y EIDOLON 0.0b

- **CS-L15 — Race condition:** reproduce un counter/índice compartido mediante interleaving controlado; corrige primero con ownership y luego compara con lock.
- **CS-L16 — Deadlock and cancellation:** observa dos locks/tasks bloqueados con timeout, documenta el wait-for cycle y corrige mediante orden global/cancellation cooperativa.

| Concepto | Secciones | Evidencia | Lab/build |
|---|---:|---|---|
| Interleaving/race | 3–5 | Guiados 2–4 | CS-L15 |
| Ownership/Lock/scope | 4, 6–7 | Guiados 4–5 | CS-L15 |
| Deadlock/ordering | 8–9 | Guiados 6–7 | CS-L16 |
| Cancellation/lifecycle | 11–14 | Guiados 8–12 | CS-L16 |
| Queue/backpressure | 12–13 | Guiados 9–10 | EIDOLON 0.0b |
| Futures/pools/GIL | 14–19 | Guiados 11–16 | EIDOLON 0.0b |
| Single writer/order | 20–22 | Guiados 17–19 | CS-L15, EIDOLON 0.0b |
| Benchmark/model choice | 23–25 | Guiado 20 | EIDOLON 0.0b |

Antes de avanzar entrega: interleaving trace, invariant/lock protocol, wait-for graph, bounded lifecycle, Future failure evidence, spawn-safe process script, single-writer output y benchmark contra baseline.

CS-M8 prepara CS-M9 al distinguir concurrencia local de comunicación por red: otro thread puede compartir memoria; otro process local requiere IPC; un peer remoto requerirá protocolos, timeouts y framing. Este módulo no abre sockets ni construye requests.

## 34. Recursos de ampliación

El módulo es autocontenido. Para ampliar usa [CS.11 Recursos recomendados](../../02_curriculum/02_computer_science_foundations.md#cs11-recursos-recomendados) y documentación oficial de Python 3.14 para `threading`, `queue`, `concurrent.futures`, `multiprocessing` y free-threaded CPython. Verifica siempre implementación, build, plataforma y start method.

## 35. Límite explícito del módulo

CS-M8 termina en process/thread/coroutine comparison, scheduling/interleavings, races, locks, deadlocks, semaphore/Event/Condition, producer/consumer, pools/Futures, GIL/build awareness, IPC/serialization, shutdown, single writer, deterministic ordering y benchmarking aplicado.

No desarrolla formal memory models, lock-free/CAS/futex, kernel scheduling, NUMA/shared-memory avanzada, networking, distributed systems, database isolation, message brokers, production job systems, Celery, Redis, Kafka, RabbitMQ, PostgreSQL, FastAPI, Docker ni AI.

El siguiente paso permitido es revisar CS-M8 como `review candidate`. **No se crea ni se desarrolla CS-M9 en esta entrega.**
