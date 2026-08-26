# CS-M10 — Arquitectura de computadoras para programadores

**Track:** Computer Science Foundations  
**Competencias:** D3.1; soporte D2.1  
**Fase:** P0  
**Nivel objetivo:** Aplicado-profesional  
**Prerequisites:** PF-M1–PF-M9, CS-M1–CS-M9  
**Build:** EIDOLON 0.0b  
**Curriculum source:** [CS-M10](../../02_curriculum/02_computer_science_foundations.md#cs-m10--arquitectura-de-computadoras-para-programadores)  
**Status:** review candidate

CS-M1 explicó crecimiento; CS-M2–CS-M6, estructuras/algoritmos/estado; CS-M7, OS/memoria/files; CS-M8, concurrencia; CS-M9, red. Ahora conectamos esas abstracciones con la máquina física sin reemplazar el análisis algorítmico por folklore de hardware.

> ¿Qué recurso limita realmente este workload y qué evidencia tenemos?

```text
algoritmo
↓ representación de datos
↓ patrón de acceso
↓ CPU / memoria / I/O
↓ bottleneck
↓ measurement + profile
↓ hypothesis
↓ una optimización
↓ remeasure + correctness
↓ decisión / trigger
```

Dos recorridos O(n) pueden tener costos reales distintos. Big O explica crecimiento; hardware, runtime, layout y workload explican constantes y puntos de quiebre. Necesitamos ambos.

## Resultados de aprendizaje

Al terminar podrás:

- distinguir CPU package, core, process/thread e instruction flow conceptual;
- explicar source Python → runtime/bytecode → machine-level work sin igualar capas;
- construir una memory hierarchy de registers, caches, RAM, storage y remote boundary;
- distinguir cache line, hit/miss, spatial/temporal locality y working set;
- razonar con cautela sobre sequential access, scattered access y pointer chasing;
- comparar locality/costo de list, linked nodes, dict, tree, heap y graph representations;
- separar memory latency de bandwidth y CPU-bound/memory-bound/I/O-bound;
- distinguir CPU cache, OS page cache y application cache;
- explicar pipeline, branches, branch prediction, ILP y SIMD solo a nivel introductorio;
- reconocer interpreter/object/allocation/copy/serialization overhead sin inventar ciclos;
- diseñar benchmark reproducible y distinguirlo de profiling;
- usar `cProfile`, `pstats` y `tracemalloc` dentro de sus límites;
- calcular speedup aplicado con Amdahl y rechazar scaling lineal automático;
- medir throughput, latency, p50/p95 y effect size con método explícito;
- construir performance/memory budgets y optimization triggers;
- perfilar un replay EIDOLON sintético, cambiar una decisión y verificar equivalencia;
- preservar correctness, determinism, source authority y provenance al optimizar.

## Cómo estudiar este módulo

1. Define workload, N, metric y target environment.
2. Obtén baseline antes de formular una mejora.
3. Usa profiler para localizar, benchmark para comparar.
4. Cambia una sola decisión.
5. Ejecuta regression checks antes de celebrar timings.
6. Reporta median/tail y ruido, no solo la mejor corrida.
7. Separa inferencia de evidencia: timing no prueba un cache miss.

### Convenciones

- **Ejemplo ejecutable:** autónomo, seguro y con asserts estables.
- **Benchmark educativo:** timings ambientales; comprueba outputs, no un ganador universal.
- **Profile ejecutable:** demuestra uso de la herramienta, no causalidad de hardware.
- **Modelo conceptual:** describe capas; no pretende simular microarquitectura.
- **Comando Linux:** opcional/platform-specific; no es prerequisite.

Baseline: Python 3.14 y standard library. Timings, CPU/cache details y memory reports dependen de interpreter, build, OS y hardware.

---

## 1. Big O y máquina real son complementarios

Un flat traversal y una node chain pueden ser O(n). Aun así difieren en allocations, objects, indirection y locality. O(n) no dice “mismo tiempo”; dice cómo escala el trabajo dominante bajo un modelo.

```text
modelo algorítmico: operaciones conforme crece n
hardware/runtime: costo real de esas operaciones
```

No reemplaces O(n²) por explicaciones de cache: una mejora algorítmica suele dominar a escala. Cuando la complejidad ya es adecuada, representation/constants pueden decidir.

### Representation

Dos opciones son O(n), pero una crea un object por elemento y otra recorre referencias contiguas. ¿Qué hipótesis medirías además del tiempo?

## 2. CPU, instructions, core y clock

Modelo pedagógico de instruction cycle:

```text
fetch → decode → execute
```

CPUs modernas solapan/reordenan trabajo y usan pipelines; el diagrama sirve para orientación, no para predecir ciclos exactos.

Python source no son instrucciones CPU directas:

```text
Python source
↓ parser/compiler/runtime
Python bytecode en CPython
↓ interpreter + native runtime
machine-level instructions
↓ CPU
```

Python bytecode ≠ machine code.

**Ejemplo ejecutable:**

```python
import dis


def add_one(value: int) -> int:
    return value + 1


instructions = list(dis.get_instructions(add_one))
assert instructions
assert any(instruction.opname == "RETURN_VALUE" for instruction in instructions)
print("Python bytecode inspected")
```

Un CPU package puede contener múltiples cores; un core ejecuta instruction streams según hardware/OS. Un software thread no equivale permanentemente a un core. CS-M8 y el GIL/build de CPython siguen aplicando.

Clock frequency mide cycles/time, pero mayor GHz no garantiza programa más rápido: architecture, instructions per cycle, memory, cache, workload y parallelism importan.

### Distingue

Clasifica: CPU package, physical core, OS thread, Python bytecode y machine instruction.

## 3. Registers, pipeline, branches, ILP y SIMD

Registers son storage extremadamente pequeño y cercano a execution units. No controlamos register allocation desde Python ordinario; sirven para ubicar el extremo rápido/pequeño de la jerarquía.

Una pipeline solapa fases/instructions. Una branch cambia el posible control flow; branch prediction intenta mantener trabajo preparado. Una prediction incorrecta puede penalizar, pero CS-M10 no recomienda “branchless Python” sin profile.

Instruction-level parallelism (ILP) es solapamiento dentro del core; no es thread/process parallelism. SIMD permite que una instruction opere sobre múltiples datos; es preview para numerical computing, no requisito ni invitación a intrinsics.

### Bottleneck

Un loop tiene muchos `if`. ¿Qué evidencia necesitarías antes de culpar branch prediction y reescribirlo?

## 4. Memory hierarchy

```text
registers
↓ L1 cache
↓ L2 cache
↓ L3 / shared cache, según CPU
↓ RAM
↓ storage
↓ remote storage/service boundary
```

Más cerca suele significar menor latency y menor capacity; más lejos, mayor capacity y latency. “Suele” importa: arquitectura/workload cambian detalles. No fijamos cifras universales.

La jerarquía existe porque no obtenemos simultáneamente memoria tan grande como storage, tan rápida como registers y de igual costo.

Storage persiste pero es más lento; RAM es volátil; CPU caches son hardware-managed. CS-M7 ya separó virtual memory/page cache/durability.

### Jerarquía

Ubica: local variable conceptual, Python object en RAM, file data en page cache, CPU cache line y persisted JSONL. ¿Cuáles son copias/representaciones distintas?

## 5. CPU cache, cache line, hit y miss

CPU cache mantiene copies de instructions/data recientemente o probablemente usadas. Una **cache line** es una unidad fija de transferencia/almacenamiento dependiente de arquitectura; 64 bytes es común en algunos CPUs, no ley universal.

- **hit:** data satisfecha en un nivel cercano;
- **miss:** debe buscarse en nivel más lejano;
- **spatial locality:** usar pronto addresses cercanas;
- **temporal locality:** reutilizar pronto algo reciente.

Un miss puede costar mucho más que arithmetic simple, pero Python timings no identifican por sí solos misses.

### Locality

Clasifica: recorrer un bytes buffer una vez, incrementar el mismo counter y saltar entre nodes dispersos.

### False sharing: una advertencia introductoria

En código paralelo, dos cores pueden modificar datos lógicamente independientes que terminan compartiendo una misma cache line. El protocolo de coherencia debe transferir o invalidar esa línea completa, y el tráfico resultante puede limitar el escalamiento aunque los threads no compartan una variable de forma intencional. A este fenómeno se le llama **false sharing**.

Es una explicación posible, no una conclusión automática: depende del layout, el runtime y la arquitectura. En Python normalmente no controlas con precisión dónde queda cada objeto; por eso un slowdown no demuestra false sharing por sí solo. Los protocolos de coherencia y el control de layout a bajo nivel quedan fuera de CS-M10.

## 6. Working set y performance cliffs

Working set es el conjunto de data activamente usado durante una fase. Si cabe en niveles rápidos, el comportamiento puede ser mejor; cuando crece, cambia cache pressure, allocation, paging o I/O.

Un cambio brusco en una curva no prueba “se acabó L3”. También puede ser algorithmic threshold, allocator, GC, OS scheduling o page cache. Mide tamaños crecientes y formula varias hipótesis.

### Budget

¿Qué tamaños 10³/10⁴/10⁵ usarías para buscar un cliff sin agotar memoria? ¿Qué métricas acompañan time?

## 7. Sequential, scattered y pointer chasing

Sequential access suele favorecer spatial locality/prefetching. Scattered access salta entre posiciones. Pointer chasing añade dependency: necesitas una reference para encontrar la siguiente.

**Benchmark educativo reproducible:**

```python
from random import Random
from time import perf_counter_ns

values = list(range(20_000))
indices = list(range(len(values)))
Random(20260826).shuffle(indices)

started = perf_counter_ns()
sequential_sum = sum(values)
sequential_elapsed = perf_counter_ns() - started

started = perf_counter_ns()
scattered_sum = sum(values[index] for index in indices)
scattered_elapsed = perf_counter_ns() - started

assert sequential_sum == scattered_sum
assert sequential_elapsed >= 0 and scattered_elapsed >= 0
print("access patterns measured")
```

No afirmes que una diferencia prueba cache behavior: también cambian Python generator/indexing overhead. El experimento produce una hipótesis para herramientas/código más controlado.

### Locality

¿Qué variables además del orden cambiaron en este benchmark? Diseña una comparación más controlada.

## 8. Representaciones: arrays, linked nodes y copies

Python `list` almacena referencias en un array dinámico contiguous-like; los referents pueden vivir dispersos. Una linked structure suele crear node objects y pointer chasing. Por eso O(1) insertion teórica no garantiza mejor end-to-end traversal/memory.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass
from time import perf_counter_ns


@dataclass
class Node:
    value: int
    next: "Node | None" = None


def build_chain(size: int) -> Node | None:
    head = None
    for value in reversed(range(size)):
        head = Node(value, head)
    return head


def sum_chain(node: Node | None) -> int:
    total = 0
    while node is not None:
        total += node.value
        node = node.next
    return total


values = list(range(5_000))
chain = build_chain(len(values))

start = perf_counter_ns()
flat_result = sum(values)
flat_elapsed = perf_counter_ns() - start
start = perf_counter_ns()
chain_result = sum_chain(chain)
chain_elapsed = perf_counter_ns() - start

assert flat_result == chain_result
assert flat_elapsed >= 0 and chain_elapsed >= 0
print("flat/chain semantics match")
```

El código no declara winner universal; compara representation/overhead en target environment.

Copiar list/bytes/str/serialized payload consume time/memory. “Zero-copy” reduce copies en APIs especializadas futuras; no implementamos advanced buffers ni object pooling sin evidence.

## 9. Dict, tree, heap y graph locality

Expected O(1) de dict puede incluir hashing, probes, indirection y memory footprint. No invalida dict: complexity y locality son dimensiones distintas.

Node-based trees pueden dispersarse; un heap array-based suele ofrecer mejor traversal locality. Graph adjacency layout/object overhead puede dominar, pero representation debe elegirse por sparsity/queries/invariants antes de microoptimizar.

```text
adjacency list: O(V+E) space para sparse graph
adjacency matrix: O(V²), más regular pero enorme si sparse
```

### Representation

¿Cuándo aceptarías peor locality de dict? Conecta con repeated lookup vs linear scan y build/memory cost.

## 10. Byte order, alignment y binary representation

Machine-level data son bits/bytes con representation. Byte order define cómo se ordenan bytes multi-byte; CS-M9 usó `struct.pack("!I", n)` para network byte order. Alignment describe constraints/preferences de ubicación para ciertos datos, dependientes de architecture/runtime.

Python abstrae gran parte; el concepto importa al cruzar binary protocols/native buffers. No enseñamos ABI, padding exhaustivo ni assembly.

## 11. Memory latency y bandwidth

- memory latency: tiempo para satisfacer acceso bajo frontera declarada;
- memory bandwidth: bytes transferidos por tiempo.

Un random lookup latency-sensitive y un large sequential copy bandwidth-sensitive pueden exigir decisiones diferentes. Igual que en CS-M9, alta bandwidth no implica baja latency.

### Bottleneck

Un scan grande utiliza CPU pero throughput no crece con más cores. Propón memory bandwidth/coordination como hipótesis, no diagnóstico final.

## 12. CPU-bound, memory-bound e I/O-bound

- **CPU-bound:** computation domina;
- **memory-bound:** data movement/latency/bandwidth domina;
- **I/O-bound:** filesystem/network/external boundary domina.

El bottleneck depende de workload/size/environment y puede migrar. Optimizar arithmetic no acelera un read bloqueado; más workers pueden saturar memory/disk.

Pipeline EIDOLON:

```text
read JSONL → parse → normalize → build indexes/graph
→ replay states → query → serialize derived output
```

Mide cada stage antes de cambiar todo.

### Bottleneck

¿Qué observación distinguiría parse CPU de file I/O o repeated query scan?

## 13. Benchmark vs profiling

Benchmark responde “¿cuánto/cómo escala esta operación bajo setup X?”. Profiling responde “¿dónde se va time/calls dentro de esta ejecución?”.

Workflow:

```text
define workload/metric
→ baseline
→ profile
→ bottleneck hypothesis
→ one change
→ remeasure
→ correctness + decision
```

No uses profiler output de input diminuto como production truth.

## 14. `cProfile` y `pstats`

`cProfile` registra function calls/time. `tottime` atribuye time dentro de una function excluyendo callees; `cumtime` incluye callees. Call count ayuda a detectar repeated work.

**Ejemplo ejecutable:**

```python
import cProfile
from io import StringIO
import pstats


def build_index(records: list[dict[str, str]]) -> dict[str, dict[str, str]]:
    return {record["event_id"]: record for record in records}


records = [{"event_id": f"evt-{index:05d}"} for index in range(2_000)]
profiler = cProfile.Profile()
profiler.enable()
index = build_index(records)
profiler.disable()

stream = StringIO()
pstats.Stats(profiler, stream=stream).sort_stats("cumulative").print_stats(10)
report = stream.getvalue()

assert len(index) == len(records)
assert "build_index" in report
print("profile captured: PASS")
```

Line-level profiler es future external tooling; no añadimos dependency.

### Profile

¿Qué diferencia observas entre una function llamada una vez y helper llamado N veces? ¿Cuál cambio justificaría cada evidencia?

## 15. Memory profiling y `tracemalloc`

`tracemalloc` rastrea allocations Python desde que inicia. No mide CPU cache hits/misses ni toda RSS.

```python
import tracemalloc

tracemalloc.start()
before, _ = tracemalloc.get_traced_memory()
records = [{"event_id": f"evt-{index:05d}"} for index in range(1_000)]
after, peak = tracemalloc.get_traced_memory()

assert len(records) == 1_000
assert after >= before
assert peak >= after
tracemalloc.stop()
print("Python allocations measured")
```

`sys.getsizeof(x)` sigue siendo shallow/implementation-dependent, no full process memory.

### Profile

Si tracemalloc crece y RSS no baja tras cleanup, ¿qué puede y qué no puedes concluir?

## 16. Tres caches distintas

| Cache | Owner/capa | Riesgo de confusión |
|---|---|---|
| CPU cache | hardware/microarchitecture | no tiene app invalidation policy |
| OS page cache | kernel/filesystem data | repeated file read puede calentarse |
| application cache | código, por ejemplo dict | consume RAM y puede quedar stale |

Vaciar una application dict no “limpia L3”; repeated file reads no demuestran un algoritmo mejor. No intentes drop system caches desde el módulo: no es portable y puede afectar la máquina.

### Jerarquía

Clasifica: `events_by_id`, bytes de journal conservados por kernel y una cache line.

## 17. Python/runtime overhead y built-ins

Según implementación, una operación Python puede incluir interpreter dispatch, boxed objects, dynamic checks y reference management. No asignamos ciclos universales.

Built-ins/runtime loops pueden ejecutar más trabajo fuera del Python-level loop. A veces `sum(values)` vence a un loop manual, pero semántica/measurement deciden.

**Benchmark educativo:**

```python
from timeit import repeat

values = list(range(10_000))


def manual_sum(items: list[int]) -> int:
    total = 0
    for item in items:
        total += item
    return total


assert manual_sum(values) == sum(values)
builtin_times = repeat(lambda: sum(values), number=10, repeat=5)
manual_times = repeat(lambda: manual_sum(values), number=10, repeat=5)

assert len(builtin_times) == len(manual_times) == 5
assert all(value >= 0 for value in builtin_times + manual_times)
print("equivalent implementations measured")
```

No conviertas el resultado de un CPU/Python version en style rule universal.

## 18. Benchmark reproducible y noise

Documenta:

- Python/version/build, OS, hardware relevante;
- seed, N, input distribution;
- setup/warmup y cache state conocido;
- repetitions y statistic;
- background load/frequency/thermal caveats;
- failures/limitations.

OS scheduling, GC, cache warmth y frequency scaling agregan noise. Usa repeated trials/median, no solo minimum conveniente. “Cold-ish” debe describir método; Python no ofrece portable cache reset.

Performance portability: una optimización puede cambiar entre Python/CPU/OS.

### Correctness

¿Qué inputs/output checks debes ejecutar antes y después del benchmark?

## 19. Latency distribution, p50 y p95

Average puede ocultar tail latency. Declara el método de percentile; aquí usamos nearest-rank sobre muestra ordenada.

```python
from math import ceil
from statistics import median

samples = [12, 8, 15, 9, 30, 10, 11, 14, 13, 50]
ordered = sorted(samples)
p50 = median(ordered)
p95 = ordered[ceil(0.95 * len(ordered)) - 1]

assert p50 == 12.5
assert p95 == 50
assert p95 == max(samples)  # coincidencia de esta muestra pequeña, no definición
print("p50/p95 computed with declared method")
```

p95 significa aproximadamente 95% de observations en o debajo según sample/method; **no significa worst case**. Aquí coincide con max porque n=10/nearest-rank. p99 extiende la misma idea hacia una cola más extrema, pero requiere muestras suficientes y tampoco equivale al máximo.

### Tail

Construye una muestra donde p95 no sea max. ¿Qué cambia al aumentar n?

## 20. Scalability, Amdahl y cores

Performance en un tamaño no implica scalability. Si fracción `P` es paralelizable, fracción `1-P` serial y `N` workers ideales:

```text
S(N) = 1 / ((1 - P) + P/N)
```

**Ejemplo ejecutable:**

```python
def amdahl_speedup(parallel_fraction: float, workers: int) -> float:
    if not 0 <= parallel_fraction <= 1:
        raise ValueError("fraction outside [0, 1]")
    if workers < 1:
        raise ValueError("workers must be positive")
    return 1 / ((1 - parallel_fraction) + parallel_fraction / workers)


speedup_four = amdahl_speedup(0.8, 4)
serial_limit = 1 / (1 - 0.8)

assert abs(speedup_four - 2.5) < 1e-9
assert abs(serial_limit - 5.0) < 1e-9
print("Amdahl model: PASS")
```

Real pools agregan coordination, serialization, contention y bandwidth limits, por lo que Amdahl ideal puede ser optimista. Más cores/threads no produce linear speedup.

### Parallelism

Si profile muestra 40% serial I/O/merge, ¿qué upper bound ideal existe incluso con workers infinitos?

## 21. Throughput, latency, batching y queueing

Batching amortiza overhead por item y puede mejorar throughput; también agrega wait latency, memory y error-handling complexity. No es universal.

```text
arrival rate se acerca a service capacity
→ queueing/wait puede crecer
```

Es preview, no queueing theory formal. Una optimization puede mejorar aggregate throughput y empeorar latency individual/tail.

### Budget

¿Qué metric/limit usarías para evitar que un batch “rápido” espere demasiado antes de empezar?

## 22. Allocation, reuse, copying, serialization y compression

Muchos small objects cuestan allocation, memory overhead y runtime reference/GC work. No introduzcas object pools manuales sin evidence: agregan lifecycle bugs.

Copies de list/bytes/str/JSON cuestan time/memory. Serialization puede dominar CS-M8 IPC o CS-M9 request. Profile antes de reemplazar JSON.

Compression reduce bytes pero usa CPU y agrega latency/complexity. Zero-copy/SIMD/native formats quedan como previews, no requirements.

### Bottleneck

Un export tarda 70% serializando y 5% escribiendo. ¿Qué optimización no ayudaría y cuál hipótesis probarías?

## 23. Optimization workflow y effect size

1. Define workload/metric.
2. Establish correct baseline.
3. Profile bottleneck.
4. Escribe hypothesis.
5. Cambia una decisión.
6. Remeasure same workload.
7. Verifica correctness/invariants.
8. Decide según effect size/trigger.

Una mejora de microsegundos que no cambia UX/cost/SLO/trigger puede no justificar complexity. Performance regression tests deben usar environment comparable y tolerancias robustas; exact thresholds en hardware compartido son flaky.

Reporta también el tamaño del efecto. Si baseline fue 120 ms y candidate 90 ms, el ratio de tiempos es `120 / 90 ≈ 1.33×` y la reducción relativa es `(120 - 90) / 120 = 25%`. Declara qué denominador y qué statistic usaste; un porcentaje sin baseline ni dispersión puede engañar.

## 24. EIDOLON 0.0b: workload sintético

```text
source records
↓ parse/replay
derived events
↓ dict/set indexes
relationship adjacency
↓ state transitions
queries
↓ derived output
```

Source permanece autoritativo; indexes/order/graph current state son rebuildable. La optimization puede cambiar representation derived, no promotion/source/provenance.

Workload debe incluir N, distribution y query mix: startup/replay, lookup ID, tag membership, graph neighbors/path pequeño y export. Datos siempre sintéticos.

### Correctness

¿Qué snapshots/asserts garantizan que un índice “más rápido” no perdió duplicate/provenance o cambió domain order?

## 25. Scan vs derived index: cierre del arco

Build de dict cuesta O(n) time/space; lookup esperado O(1). Repeated scan cuesta O(n) por query. El crossover depende de query count, n, hashing, memory y cache behavior.

**Ejemplo ejecutable:**

```python
from time import perf_counter_ns

records = [{"event_id": f"evt-{index:05d}", "value": index} for index in range(10_000)]
queries = [f"evt-{index:05d}" for index in range(0, 10_000, 200)]


def scan_find(event_id: str) -> dict[str, object] | None:
    return next((record for record in records if record["event_id"] == event_id), None)


start = perf_counter_ns()
scan_results = [scan_find(event_id) for event_id in queries]
scan_elapsed = perf_counter_ns() - start

start = perf_counter_ns()
index = {record["event_id"]: record for record in records}
indexed_results = [index.get(event_id) for event_id in queries]
indexed_elapsed = perf_counter_ns() - start

assert indexed_results == scan_results
assert scan_elapsed >= 0 and indexed_elapsed >= 0
assert len(index) == len(records)
print("scan/index correctness preserved")
```

El indexed timing incluye build. Cambia query count/sizes antes de concluir. Index sigue derived, no source of truth.

## 26. Replay profile y una optimización

Un replay puede parsear JSON, copiar records, ordenar repetidamente, construir índices/adjacency y aplicar transitions. Profile stage boundaries.

**Ejemplo ejecutable:**

```python
import cProfile
import json
from io import StringIO
import pstats


def replay(lines: list[str]) -> tuple[list[dict[str, object]], dict[str, dict[str, object]]]:
    events = [json.loads(line) for line in lines]
    events.sort(key=lambda event: (event["sequence"], event["event_id"]))
    index = {event["event_id"]: event for event in events}
    return events, index


lines = [
    json.dumps({"event_id": f"evt-{index:05d}", "sequence": index})
    for index in reversed(range(2_000))
]
profiler = cProfile.Profile()
events, index = profiler.runcall(replay, lines)

stream = StringIO()
pstats.Stats(profiler, stream=stream).sort_stats("cumulative").print_stats(12)
assert len(events) == len(index) == 2_000
assert [event["sequence"] for event in events] == list(range(2_000))
assert "replay" in stream.getvalue()
print("synthetic replay profiled: PASS")
```

Una optimization válida podría evitar redundant sort o repeated scan, pero cambia solo después del profile y conserva output exacto.

## 27. Performance budget, memory budget y triggers

Performance budget:

| Operation | Workload | Metric | Baseline | Target | Hypothesis | Trigger |
|---|---|---|---|---|---|---|
| replay | N synthetic events | wall time/throughput | measured | local contract | stage profile | target exceeded repeatedly |
| lookup | Q IDs over N | p50/p95 | measured | UX contract | scan dominates | Q/N crossover |

Memory budget:

| Component | Estimated/measured | Growth variable | Bound | Trigger |
|---|---|---|---|---|
| source lines | bytes | N | workload-specific | RAM pressure |
| derived index | tracemalloc/RSS note | unique IDs | rebuildable bound | retention exceeded |

Targets no son universales. El trigger debe nombrar workload, metric y evidence. “Quizá escale algún día” no basta.

### Budget

Completa un budget para graph traversal con V/E/path queries y un trigger que no mencione tecnología por moda.

## 28. End-to-end thinking

```text
latency_total
≈ stage work + I/O + waiting/queueing + contention
```

Una mejora local de microsegundos puede ser irrelevante frente a remote latency o disk wait. Más workers pueden saturar CPU, memory bandwidth, disk o locks. Vertical/horizontal scaling existen como previews; distributed design queda después.

No cherry-pick best runs, ocultes failures, cambies input o mezcles warm/cold state sin reportarlo. Performance evidence también exige ética y reproducibilidad.

## 29. Hardware/environment observation

**Ejemplo ejecutable:**

```python
import os
import platform
import sys

environment = {
    "python": sys.version.split()[0],
    "implementation": platform.python_implementation(),
    "os": platform.system(),
    "machine": platform.machine(),
    "cpu_count_observed": os.cpu_count(),
}
assert environment["python"]
assert environment["os"]
assert environment["cpu_count_observed"] is None or environment["cpu_count_observed"] > 0
print("environment metadata captured")
```

**Linux-specific/optional:** `lscpu`, `/proc/cpuinfo`, `free -h`, `ps`. `perf stat` puede observar cycles/cache-misses/branches si hardware, kernel y permisos lo permiten; counters/noise varían. No es prerequisite ni prueba automática de causalidad.

## 30. Catálogo de failure cases

| Creencia/fallo | Impacto | Corrección |
|---|---|---|
| mismo Big O = mismo time | ignora constants/locality/runtime | benchmark + model |
| CPU cache = dict cache | mezcla owners/policies | separar hardware/app |
| page cache = CPU cache | interpreta repeated I/O mal | identificar kernel/hardware |
| GHz = program speed | ignora IPC/memory/workload | target benchmark |
| más cores = linear speedup | ignora serial fraction | Amdahl + measure |
| más threads = faster | contention/GIL/I/O saturation | baseline/workload |
| optimize sin profiler | mejora etapa irrelevante | profile first |
| benchmark un N | no muestra growth/cliff | sizes/distributions |
| microbenchmark = production | workload boundary falsa | end-to-end evidence |
| linked O(1) insertion = superior | traversal/memory/locality omitted | full operation mix |
| `getsizeof` = total memory | shallow number | tracemalloc/RSS/model |
| p95 = maximum | interpreta tail mal | method + max separately |
| optimization cambia output | correctness regression | tests/invariants first |
| optimized index = source | rompe authority/provenance | derived/rebuildable |
| application cache sin invalidation | stale state | policy/trigger or avoid |
| personal data en benchmark | privacy leak | synthetic dataset |

## 31. Ejercicios guiados

### Guiado 1 — Mapea hierarchy

Dibuja registers/caches/RAM/storage/remote y agrega capacity/latency cualitativas. No fija números universales.

### Guiado 2 — Distingue caches

Clasifica `events_by_id`, repeated file read y cache line. La solución identifica app/kernel/hardware.

### Guiado 3 — Spatial locality

Traza sequential list references y scattered indices. Explica qué otros Python overheads impiden atribución causal.

### Guiado 4 — Temporal locality

Compara hot counter/small table con una tabla mayor usada una vez. Formula hypothesis, no resultado garantizado.

### Guiado 5 — Sequential vs scattered

Ejecuta sección 7 con seed fija y 5 repetitions. Verifica checksum; reporta mediana/rango y límites.

### Guiado 6 — Linked locality

Ejecuta sección 8. Mide allocations/elapsed y compara semantics; no concluye desde un N.

### Guiado 7 — `cProfile`

Ejecuta sección 14. Localiza call count/tottime/cumtime y explica cada columna.

### Guiado 8 — Bottleneck

Perfila dos stages synthetic: repeated scan y one-time sort. Elige el dominante para ese workload.

### Guiado 9 — `tracemalloc`

Ejecuta sección 15 antes/después de cleanup. No exige que RSS o traced memory vuelvan inmediatamente al baseline.

### Guiado 10 — Bound classification

Diseña CPU arithmetic, large scan y temporary-file read. Mide, pero clasifica solo con multiple evidence.

### Guiado 11 — Amdahl

Calcula P=.8 con N=1,2,4,8 y limit. Explica por qué real pool puede quedar debajo.

### Guiado 12 — Sequential vs concurrent

Reutiliza CS-M8 con CPU/process y I/O/thread workloads. Incluye startup/serialization y correctness.

### Guiado 13 — Performance budget

Completa operation/workload/metric/baseline/target/hypothesis/trigger para replay.

### Guiado 14 — Memory budget

Registra source/index/adjacency, growth variable y bound. Separa tracemalloc de RSS.

### Guiado 15 — p50/p95

Ejecuta section 19 y crea 100 samples con outlier final. Reporta p50/p95/max por separado.

### Guiado 16 — Profile replay

Ejecuta sección 26 con N crecientes. Identifica stage dominante por cumulative time.

### Guiado 17 — Optimize one stage

Reemplaza repeated scan por dict o redundant sort por derived ordered view. Mantén source intacto.

### Guiado 18 — Regression correctness

Compara outputs antes/después: IDs, ordering, tags, adjacency y state. La mejora falla si cambia invariants.

## 32. Ejercicios independientes

1. Explica CPU/core/thread con un diagrama.
2. Usa `dis` y separa bytecode/machine code.
3. Ordena memory hierarchy cualitativamente.
4. Explica cache line sin fijar 64 bytes universal.
5. Clasifica spatial/temporal locality.
6. Diseña un working-set size sweep.
7. Propón causas alternativas para un performance cliff.
8. Compara sequential/scattered con seed/repetitions.
9. Implementa node chain y flat representation equivalentes.
10. Discute dict expected O(1) y locality.
11. Compara heap array y tree nodes conceptualmente.
12. Estima adjacency matrix/list space para sparse graph.
13. Distingue memory latency/bandwidth.
14. Clasifica CPU/memory/I/O bottleneck con evidence.
15. Profilea una function con `cProfile`.
16. Ordena stats por cumulative/tottime.
17. Mide allocations con tracemalloc.
18. Distingue tres caches en un benchmark.
19. Compara built-in/manual sin style dogma.
20. Documenta environment/noise/warmup.
21. Calcula p50/p95/max con método explícito.
22. Calcula Amdahl para tres P/N.
23. Diseña batching throughput/latency tradeoff.
24. Mide allocation/copying de synthetic payloads.
25. Profilea JSON serialization vs derived write.
26. Construye scaled replay workload.
27. Compara scan/index incluyendo build/memory.
28. Optimiza una sola stage y remeasure.
29. Crea regression checks deterministas.
30. Construye performance/memory budget.
31. Define migration trigger medible.
32. Explica false sharing sin afirmar que ocurre en tu test.
33. Documenta Linux `perf` limitations si está disponible.
34. Redacta un ADR sobre streaming/materialization.
35. Explica por qué source no se convierte en index.

## 33. Preguntas conceptuales

1. ¿Por qué dos algoritmos O(n) pueden rendir distinto?
2. ¿Por qué Big O y hardware model son complementarios?
3. ¿Qué diferencia CPU package, core y thread?
4. ¿Por qué Python bytecode no es machine code?
5. ¿Qué problema resuelve memory hierarchy?
6. ¿Qué son cache line, hit y miss?
7. ¿Qué diferencia spatial y temporal locality?
8. ¿Qué significa working set?
9. ¿Por qué un performance cliff requiere hipótesis alternativas?
10. ¿Por qué pointer chasing puede perjudicar locality?
11. ¿Qué locality tiene una Python list y sus referents?
12. ¿Cómo afectan dict/tree/graph representation?
13. ¿Qué diferencia memory latency y bandwidth?
14. ¿Cómo distingues CPU/memory/I/O-bound?
15. ¿Qué es bottleneck y por qué cambia?
16. ¿Qué diferencia benchmark y profiling?
17. ¿Qué significan tottime/cumtime/call count?
18. ¿Qué mide tracemalloc y qué no?
19. ¿Qué diferencia CPU/page/application cache?
20. ¿Por qué GHz no decide performance?
21. ¿Qué aportan branch prediction/pipeline a la intuición?
22. ¿Por qué built-ins pueden ser más rápidos?
23. ¿Qué hace reproducible un benchmark?
24. ¿Por qué warm/cold debe reportarse?
25. ¿Qué significa p95 y por qué no es max?
26. ¿Qué enseña Amdahl?
27. ¿Por qué más workers no escalan linealmente?
28. ¿Cómo batching cambia throughput/latency?
29. ¿Qué costos agregan allocations/copies/serialization?
30. ¿Qué es effect size relevante?
31. ¿Qué diferencia performance y scalability?
32. ¿Qué debe contener un performance budget?
33. ¿Qué hace accionable un optimization trigger?
34. ¿Por qué correctness sigue siendo gate?
35. ¿Por qué derived index nunca reemplaza source?

## 34. Mini challenge — Replay medido de EIDOLON 0.0b

### Objetivo y artefactos

```text
cs_m10_challenge/
├── workload.py
├── baseline.py
├── optimized.py
├── benchmark.py
├── synthetic_source.jsonl
├── PERFORMANCE_BUDGET.md
└── MEMORY_BUDGET.md
```

No es auditoría global; es un cierre práctico con synthetic data.

### A. Correct baseline

1. Genera N source records con seed/sequence/IDs estables.
2. Parse JSONL sin mutar source.
3. Construye derived events, dict/set indexes y sparse adjacency.
4. Replay synthetic state transitions.
5. Ejecuta query mix: ID, tag, neighbors y ordered timeline.
6. Guarda canonical observable snapshot.

### B. Measure

7. Registra wall time/throughput para N crecientes.
8. Toma repeated query latency samples.
9. Calcula p50/p95/max con método declarado.
10. Usa tracemalloc current/peak y nota que no es RSS/cache.
11. Captura Python/OS/CPU count/input/seed/repetitions.

### C. Profile y hypothesis

12. Usa cProfile/pstats por cumulative time.
13. Escribe `bottleneck → evidence → proposed change`.
14. No atribuye cache misses solo desde timings.

### D. Optimize one thing

15. Cambia exactamente una decisión: scan→dict, redundant sort→ordered view, unnecessary copy→reuse o flat representation.
16. No cambia source/provenance/domain order.
17. No añade tecnología/dependency.

### E. Remeasure/correctness

18. Ejecuta mismo workload/environment.
19. Compara latency/throughput/tracemalloc.
20. Compara canonical snapshot byte/structuralmente.
21. Reporta regressions/noise, incluso si la optimization no gana.

### F. Amdahl y scaling

22. Estima P desde stages medidos y calcula ideal S(N).
23. Explica serial replay/merge/I/O.
24. No paraleliza si profile no lo justifica.

### G. Budgets/triggers

25. Performance budget incluye operation/workload/metric/baseline/target/trigger.
26. Memory budget incluye component/size/growth/bound/trigger.
27. Define qué evidencia activaría migration/otra representation.

### H. Hardware explanation

28. Explica locality/working set/pointer chasing/copies/I/O cualitativamente.
29. Distingue CPU/page/application cache.
30. Declara límites: sin hardware counters no prueba cache misses.

### Comprobaciones contractuales

**Continuación — adapta nombres:**

```python
source_before = source_path.read_bytes()
baseline = run_pipeline(source_path, strategy="scan")
optimized = run_pipeline(source_path, strategy="index")

assert baseline.snapshot == optimized.snapshot
assert baseline.source_hash == optimized.source_hash
assert source_path.read_bytes() == source_before
assert baseline.ids == optimized.ids
assert baseline.timeline == optimized.timeline
assert baseline.adjacency == optimized.adjacency

for report in (baseline.metrics, optimized.metrics):
    assert report.wall_seconds >= 0
    assert report.throughput >= 0
    assert report.p50_seconds <= report.p95_seconds <= report.max_seconds
    assert report.traced_peak_bytes >= report.traced_current_bytes >= 0
```

### Criterio de aprobación

- baseline es correcto/reproducible y source intacto;
- workload/seed/N/repetitions/environment están documentados;
- profile localiza stage real;
- solo una decisión cambia;
- outputs/invariants coinciden;
- p95 no se llama worst case;
- memory/cache metrics no se confunden;
- Amdahl incluye serial fraction/overhead limits;
- budgets/triggers son medibles;
- no aparecen database, backend, Docker, external services ni AI.

## 35. Resumen

- Big O describe growth; runtime/hardware/layout explican constants/cliffs.
- CPU/core/thread/bytecode/instruction son capas distintas.
- Memory hierarchy intercambia latency/capacity; no usa cifras universales.
- Cache lines alimentan spatial/temporal locality; hit/miss son hardware events.
- Working set y access pattern pueden cambiar performance.
- Sequential access suele ayudar; pointer chasing/objects agregan indirection.
- Complexity/locality/memory footprint deben analizarse juntas.
- CPU, page y application caches tienen owners distintos.
- CPU/memory/I/O bottlenecks dependen del workload.
- Benchmark mide outcome; profiler localiza time/calls.
- tracemalloc observa Python allocations, no CPU cache/RSS total.
- Pipeline/branch/ILP/SIMD son intuiciones, no microoptimization mandate.
- Python/runtime/object/allocation/copy/serialization overhead requiere medición.
- Reproducibility incluye environment, N, seed, repetitions y noise.
- p50/p95/max describen distintas partes; p95 no es worst case.
- Amdahl limita ideal speedup por serial fraction.
- Throughput, latency y batching pueden tener tradeoffs.
- Baseline→profile→hypothesis→one change→remeasure es el workflow.
- Budgets/triggers evitan optimization por anticipación.
- EIDOLON optimiza derived representations sin sacrificar source/provenance/correctness.

## 36. Checklist de dominio

- [ ] Puedo complementar Big O con hardware/runtime evidence.
- [ ] Puedo distinguir CPU/core/thread/instruction layers.
- [ ] Puedo explicar Python bytecode sin llamarlo machine code.
- [ ] Puedo dibujar memory hierarchy.
- [ ] Puedo explicar cache line/hit/miss con límites.
- [ ] Puedo distinguir spatial/temporal locality.
- [ ] Puedo razonar sobre working set/cliffs.
- [ ] Puedo comparar sequential/scattered/pointer chasing.
- [ ] Puedo analizar locality de representations.
- [ ] Puedo distinguir memory latency/bandwidth.
- [ ] Puedo clasificar CPU/memory/I/O bottleneck con evidence.
- [ ] Puedo separar CPU/page/application caches.
- [ ] Puedo distinguir benchmark/profiling.
- [ ] Puedo usar cProfile/pstats.
- [ ] Puedo interpretar tracemalloc sin llamarlo RSS/cache profiler.
- [ ] Puedo explicar branch/pipeline/ILP/SIMD como preview.
- [ ] Puedo medir runtime/allocation/copy/serialization overhead.
- [ ] Puedo documentar reproducible benchmark.
- [ ] Puedo calcular p50/p95/max con método declarado.
- [ ] Puedo calcular/limitar speedup con Amdahl.
- [ ] Puedo explicar throughput/latency/batching tradeoff.
- [ ] Puedo aplicar baseline→profile→one change→remeasure.
- [ ] Puedo evaluar effect size/correctness.
- [ ] Puedo construir performance/memory budgets.
- [ ] Puedo definir optimization/migration trigger.
- [ ] Puedo perfilar replay EIDOLON sintético.
- [ ] Puedo conservar source/provenance/determinism.
- [ ] Puedo resolver mini challenge con PF + CS-M1–CS-M10.

## 37. Preparación para CS-L18 y EIDOLON 0.0b

- **CS-L18 — Locality and I/O:** compara acceso secuencial/aleatorio, batch sizes y métricas; entrega ADR sobre streaming.

| Concepto | Secciones | Evidencia | Lab/build |
|---|---:|---|---|
| Hierarchy/locality/working set | 4–8 | Guiados 1–6 | CS-L18 |
| Bound/bottleneck/cache layers | 11–16 | Guiados 7–10 | CS-L18 |
| Amdahl/latency/tail | 19–21 | Guiados 11–15 | EIDOLON 0.0b |
| Replay/profile/optimization | 24–26 | Guiados 16–18 | CS-L18, EIDOLON 0.0b |
| Budgets/triggers/environment | 27–29 | Mini challenge | EIDOLON 0.0b |

Antes de avanzar entrega: environment/workload record, sequential/scattered benchmark, profile, one-change experiment, regression snapshot, p50/p95/max y budgets/triggers.

EIDOLON 0.0b puede defender list/dict/set/deque, graph/state replay, file safety, concurrency/network experiments y cost triggers sin incorporar arquitectura de Backend/Database/AI.

## 38. Qué debes poder explicar después de CS-M1–CS-M10

- por qué una estructura encaja con operations/workload;
- cómo derivar complexity y comprobarla con measurement;
- cómo funcionan search/sort/traversal y sus invariants;
- cómo graph/state representations preservan semantics/provenance;
- dónde están OS, process, thread, filesystem y network boundaries;
- cómo separar bytes, framing, transport y HTTP semantics;
- cómo benchmark/profile localiza un bottleneck;
- cómo locality/working set/cache/I/O afectan constants;
- cómo Amdahl, tail latency y budgets limitan decisiones;
- cómo defender un tradeoff y señalar qué evidence lo cambiaría.

Estas capacidades cierran el material producido; **no constituyen una auditoría ni una aprobación global del track**.

## 39. Recursos de ampliación

El módulo es autocontenido. Para ampliar usa [CS.11 Recursos recomendados](../../02_curriculum/02_computer_science_foundations.md#cs11-recursos-recomendados) y documentación oficial de Python 3.14 para `dis`, `timeit`, `cProfile`, `pstats`, `tracemalloc`, `statistics`, `platform` y `os`.

## 40. Límite explícito del módulo

CS-M10 termina en CPU/core/instruction intuition, memory hierarchy/cache/locality/working set, bound classification, profiling/benchmarking, Amdahl/tail latency, budgets/triggers y measured EIDOLON replay optimization.

No desarrolla assembly/ISA/microcode, advanced pipeline/cache coherence/MESI/NUMA, SIMD intrinsics/GPU/CUDA, compiler/VM internals, formal queueing/roofline, distributed systems, databases, Backend Track, Docker ni AI.

El siguiente paso permitido es una auditoría global acumulativa separada de Computer Science Foundations. **No se declara aprobación global ni se genera Backend Track en esta entrega.**
