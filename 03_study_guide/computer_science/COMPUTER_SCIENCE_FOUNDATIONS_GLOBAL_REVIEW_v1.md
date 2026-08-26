# Computer Science Foundations — Global Review v1

**Track:** Computer Science Foundations  
**Scope:** CS-M1–CS-M10, CS-L01–CS-L18, CS-MP1–CS-MP3 y EIDOLON 0.0b  
**Canonical curriculum:** [`02_computer_science_foundations.md`](../../02_curriculum/02_computer_science_foundations.md)  
**Editorial standard:** [`STUDY_GUIDE_EDITORIAL_STANDARD_v0.1.md`](../STUDY_GUIDE_EDITORIAL_STANDARD_v0.1.md)  
**Review date:** 2026-08-26  
**Gate status:** PASS

## 1. Executive verdict

El track es técnicamente correcto, curricularmente completo, pedagógicamente progresivo y ejecutable con el baseline Python 3.14. Los diez módulos enseñan primero el problema y el modelo de costo, luego la representación o mecanismo, sus invariantes, failure cases, medición y aplicación acotada a EIDOLON.

El gate cierra con:

- **CRITICAL:** 0;
- **IMPORTANT:** 1, corregido;
- **MINOR:** 2, uno corregido y uno documentado como asunto upstream no bloqueante;
- **OPTIONAL:** 0.

No quedan defectos que impidan resolver los labs, mini projects o el build EIDOLON 0.0b. CS-M1–CS-M10 pueden aprobarse. No se requiere una nueva versión del estándar editorial.

## 2. Curriculum coverage matrix

| Competencia | Enseñanza principal | Práctica y evidencia | Estado |
|---|---|---|---|
| D2.1 | CS-M1–CS-M5 y CS-M10: modelos de costo, estructuras, búsqueda, sorting, trees, heaps y medición | CS-L01–CS-L09, CS-L18; CS-MP1; benchmarks e invariantes ejecutables | completa |
| D2.2 | CS-M5–CS-M6: jerarquías, graphs tipados, BFS/DFS, paths, cycles y state machines | CS-L09–CS-L11; CS-MP3; provenance y transiciones exhaustivas | completa |
| D2.3 | CS-M1–CS-M6: edge cases, determinismo, estabilidad, Unicode/tiempo ya adquiridos y replay | CS-L01–CS-L13; CS-MP1 y CS-MP3; suites de propiedades | completa |
| D3.1 | CS-M7 y CS-M10: process, virtual memory, resources, filesystem, cache hierarchy y profiling | CS-L12–CS-L14, CS-L18; diagnóstico y failure injection | completa |
| D3.2 | CS-M3 y CS-M8: queues, ownership, threads, processes, synchronization, cancellation y backpressure | CS-L05, CS-L15–CS-L16; CS-MP2; races deterministas | completa |
| D3.3 | CS-M9: DNS, TCP, UDP, TLS/HTTP, framing, timeouts e idempotency | CS-L17; siete escenarios de loopback y clasificación de fallos | completa |

### Cobertura por módulo

| Módulo | Núcleo curricular verificado | Profundidad | Resultado |
|---|---|---|---|
| CS-M1 | O, Ω, Θ; costo temporal/espacial; amortized vs average; medición | alta | completo |
| CS-M2 | dynamic arrays, hash maps, sets, collisions, hashing y rebuild | alta | completo |
| CS-M3 | ADTs, stack, queue, deque, linked structures e invariantes | alta | completo |
| CS-M4 | recursion, linear/binary search, sorting, estabilidad y bisect | muy alta | completo |
| CS-M5 | trees, BST, heaps, priority queues y top-k | muy alta | completo tras corrección |
| CS-M6 | graphs, BFS/DFS, paths/cycles, logic y state machines | alta | completo |
| CS-M7 | proceso, memoria, resources, filesystem y crash model | muy alta | completo |
| CS-M8 | processes, threads, races, locks, pools, cancellation y backpressure | muy alta | completo |
| CS-M9 | DNS, TCP/UDP, TLS/HTTP, timeouts, retry e idempotency | muy alta | completo |
| CS-M10 | CPU, memoria/cache, locality, profiling, latency y budgets | alta | completo |

## 3. Dependency audit

La progresión efectiva es acumulativa y no contiene cycles:

```text
PF-M1–PF-M9 aprobados
↓
CS-M1 → CS-M2 → CS-M3 → CS-M4 → CS-M5 → CS-M6
↓
CS-M7 → CS-M8 → CS-M9 → CS-M10
```

- CS-M1 establece el lenguaje de costo y medición antes de comparar estructuras.
- CS-M2–CS-M3 separan representación, ADT e invariantes antes de algoritmos recursivos.
- CS-M4 prepara traversal y orden para trees/heaps de CS-M5.
- CS-M5 prepara la frontera tree/graph; CS-M6 generaliza relaciones y lifecycle.
- CS-M7 fija recursos y failure model antes de concurrencia; CS-M8 fija ownership antes de networking.
- CS-M10 integra costo algorítmico, runtime, OS, concurrencia y hardware sin redefinirlos.

Los metadatos de cada módulo declaran los prerequisites realmente usados. El índice obliga a completar Programming Foundations antes del track, por lo que no existe prerequisite oculto en la ruta publicada.

## 4. Technical audit

### CS-M1–CS-M4

- Big O no se reduce incorrectamente a worst case; O, Ω y Θ se distinguen.
- El análisis separa dimensión de entrada, build, query, tiempo y espacio.
- `list` se modela como dynamic array de referencias; append amortizado no se presenta como garantía worst-case por llamada.
- `dict` y `set` usan expected O(1) con supuestos explícitos; collisions, equality, hashability, load y resizing aparecen sin prometer internals universales.
- Stack/queue/deque y linked structures mantienen contratos e invariantes claros, incluidos empty, one-element, stale tail, cycles y lost chains.
- Binary search declara precondition y convención de intervalos; los sort educativos son correctos; estabilidad, tie-breakers y no mutación del source están cubiertos.

### CS-M5–CS-M6

- Height y shape explican O(h) sin prometer balance del BST.
- La representación array de heap, parent/child indices, sift, `heapify`, tie-breaking y bounded top-k son correctos.
- Adjacency list/matrix, density, directedness, typed edges y provenance se distinguen.
- BFS/DFS, parent maps, components y cycle detection conservan sus invariantes y costos O(V + E) para adjacency lists.
- State machines separan transition table, estado alcanzable, historial append-only y replay determinista.

### CS-M7–CS-M10

- Object lifetime no se confunde con resource lifecycle ni RSS con `tracemalloc`.
- `write`, `flush`, `fsync`, `close`, atomic replace, durability y crash consistency se presentan como garantías distintas y dependientes de plataforma/filesystem.
- Threads, processes, shared state, GIL tradicional y builds free-threaded opcionales se describen sin recetas universales.
- Locks protegen protocolos, no objetos mágicamente; deadlock, starvation, cancellation, futures, queue accounting y backpressure están diferenciados.
- TCP se enseña como byte stream; framing y partial reads son responsabilidad de aplicación. DNS, UDP, TLS, HTTP, timeout, retry e idempotency no se mezclan por capa.
- CPU/bytecode, cache hierarchy, locality, working set, latency distribution y Amdahl se usan como modelos medibles, no como folklore de optimización.

## 5. Mathematical audit

Las derivaciones y límites centrales son correctos:

- sumas secuenciales O(f(n) + g(n)); loops nested multiplican dimensiones cuando corresponde;
- linear search O(n), binary search O(log n) con input ordenado;
- selection/insertion sort O(n²), merge sort O(n log n) tiempo y O(n) espacio auxiliar para la implementación presentada;
- traversal de tree O(n), operaciones BST O(h), heap push/pop O(log n), `heapify` O(n);
- BFS/DFS O(V + E) sobre adjacency list y O(V²) sobre matrix al inspeccionar filas completas;
- Amdahl usa `S(N) = 1 / ((1 - P) + P/N)` y no promete speedup real sin medir overhead.

### Hallazgo IMPORTANT corregido

**Ubicación:** CS-M1, anticipación de top-k; CS-M5, sección 20.  
**Problema:** `O(n log k)` se aplicaba sin separar el caso literal `k = 1`, donde `log 1 = 0`, ni el contrato `k <= 0`.  
**Impacto:** la fórmula podía producir una interpretación matemática imposible aunque el algoritmo fuera correcto.  
**Corrección:** se definió `k_eff = min(k, n)`; `k <= 0` retorna vacío, `k = 1` realiza un scan O(n), y `2 <= k <= n` usa O(n log k) más ordenamiento final O(k log k).

La documentación oficial de Python confirma que `heapify` es linear, que las APIs max-heap se incorporaron en 3.14 y que para `n == 1` conviene `max()`/`min()`: [Python 3.14 `heapq`](https://docs.python.org/3.14/library/heapq.html).

## 6. Pedagogical audit

Todos los módulos conservan la secuencia editorial relevante:

```text
problema → modelo mental → mecanismo → invariante
→ ejemplo → failure case → medición/tradeoff
→ aplicación EIDOLON → práctica → challenge → labs
```

La práctica está distribuida mediante Predice, Explica, Detecta, Modifica, Mide, Profile y preguntas de decisión. Los ejercicios guiados incluyen razonamiento y los independientes no dependen de conceptos posteriores. Las estructuras manuales se presentan como modelos educativos; la biblioteca estándar se recomienda cuando corresponde. Performance se formula como hipótesis verificable y no como carrera por micro-optimizaciones.

La carga crece deliberadamente: CS-M1–CS-M3 son densos, CS-M4–CS-M6 integran algoritmos e invariantes, y CS-M7–CS-M10 alcanzan profundidad aplicado-profesional. No hay un salto conceptual sin puente, aunque CS-M4, CS-M5 y CS-M7–CS-M9 requieren más tiempo de práctica que una lectura lineal.

## 7. Cross-module consistency

- `source of truth`, derived index y replay conservan el mismo significado en CS-M1–CS-M10.
- Orden de llegada, timeline derivada, stability y tie-breakers no se contradicen.
- Identity de dominio no se reemplaza por `hash()`, dirección de memoria, inode, PID o metadata del filesystem.
- Atomicity de estructura, operación lógica, file replace y network effect se mantienen como contratos diferentes.
- Backpressure aparece primero como capacidad de queue y luego como control de admisión concurrente, sin redefinición.
- Los previews señalan el módulo posterior y no desarrollan anticipadamente databases, distributed systems, Security, backend o AI.

### Hallazgo MINOR corregido

**Ubicación:** títulos H1 de CS-M3–CS-M9 e índice.  
**Problema:** varios títulos usaban variantes editoriales válidas pero no la cadena canónica exacta del Curriculum.  
**Impacto:** búsqueda y trazabilidad curricular menos precisas.  
**Corrección:** se alinearon los títulos visibles y las notas del índice; se preservaron filenames y links existentes para no romper referencias.

## 8. Systems and portability audit

- Los ejemplos destructivos operan en temporary directories o sobre datos sintéticos.
- Los ejemplos de red enlazan `127.0.0.1` y ports efímeros; no dependen de Internet.
- Los scripts con start method `spawn` usan funciones top-level y main guard.
- Las garantías Unix-only —directory `fsync`, `/proc`, permisos— se etiquetan; Windows se trata como modelo distinto de handles y semantics.
- `os.replace` se limita a misma filesystem boundary y a las garantías del OS/filesystem; no se presenta como transacción universal. Véase [`os.replace`](https://docs.python.org/3.14/library/os.html#os.replace).
- El GIL se atribuye a builds tradicionales de CPython y los builds free-threaded se describen como opcionales, coherente con [`threading`](https://docs.python.org/3.14/library/threading.html).
- Retry HTTP se condiciona a semántica idempotente, conocimiento de aplicación e idempotency keys; no se deriva solo del método. La semántica base coincide con [RFC 9110](https://www.rfc-editor.org/rfc/rfc9110.html).

## 9. EIDOLON integration audit

El hilo EIDOLON se mantiene pequeño y educativo:

- journal/source append-only como autoridad;
- índices in-memory por ID, persona, tag y tiempo, siempre descartables y reconstruibles;
- timeline estable y range query mediante sort/bisect;
- undo/correction sin borrar source;
- top-k, queues y graphs como derived views con políticas explícitas;
- edges tipadas con provenance;
- lifecycle reproducible mediante transition tables;
- un writer dueño del commit y workers que producen resultados;
- export derivado mediante temporal, validación y replace;
- perfil y budgets antes de optimizar.

No aparecen databases, graph databases, frameworks web, Docker, embeddings, LLMs ni arquitectura de backend como requisito. Las interpretaciones derivadas no se promueven a hechos; se preservan fuente, provenance y contradicción.

## 10. Labs and mini-project resolvability

| Evidencia | Preparación suficiente | Resultado |
|---|---|---|
| CS-L01–CS-L03 | CS-M1–CS-M2: curvas, lookup, memoria, chaining y worst case | resoluble |
| CS-L04–CS-L06 | CS-M3: undo, bounded deque, retries, nodes y reality check | resoluble |
| CS-L07 | CS-M4: stable sort, tie-breakers, timestamps aware y bisect | resoluble |
| CS-L08–CS-L09 | CS-M5: heap bounded, tree traversals, depth y cycle rejection | resoluble |
| CS-L10–CS-L11 | CS-M6: graph tipado, provenance, paths y transition properties | resoluble |
| CS-L12–CS-L14 | CS-M7: streaming/materialización, memory evidence, crash-safe file y process inspection | resoluble |
| CS-L15–CS-L16 | CS-M8: race, ownership, locks, deadlock, timeout y cancellation | resoluble |
| CS-L17 | CS-M9: DNS→TCP→HTTP/TLS conceptual y failure classification | resoluble |
| CS-L18 | CS-M10 con CS-M7: locality, batch, I/O metrics y ADR | resoluble |
| CS-MP1 | CS-M1–CS-M4: índice Event in-memory, rebuild, queries, benchmark y ADR | resoluble |
| CS-MP2 | CS-M3, CS-M6–CS-M8: scheduler, state, idempotency, backpressure y cancellation | resoluble |
| CS-MP3 | CS-M5–CS-M6: relationship/evidence graph, traversal y provenance | resoluble |

El prototipo temporal de validación de EIDOLON 0.0b reconstruyó índices por ID/persona/tag, una timeline determinista, range queries, top-k, graph traversal y lifecycle; rechazó duplicate IDs, una transición imposible y JSON truncado; produjo un export derivado sin mutar el source. Tres replays produjeron el mismo snapshot.

## 11. Code validation

Baseline observado: CPython 3.14.7.

| Comprobación | Resultado |
|---|---|
| Bloques Python extraídos | 237/237 compilan con `ast.parse` |
| Bloques autónomos/benchmarks/modelos seleccionados | 147/147 ejecutan sin fallo |
| Ejemplos CS-M9 de socket/HTTP en loopback | 7/7 pasan fuera del sandbox restrictivo |
| Failure cases autónomos seguros | 19/19 producen el failure contract esperado |
| Casos aleatorios adicionales | 1,200/1,200 para sorting, binary search y top-k |
| Traversal adicional | BFS determinista, disconnected vertex y reachability pasan |
| Integración EIDOLON 0.0b | 3/3 replays deterministas; source sin cambios |
| Markdown y links locales | 10 módulos, fences balanceados y targets/anchors válidos |

Los fragmentos deliberadamente incompletos y continuaciones se validaron sintácticamente y por inspección contextual; no se ejecutaron aislados cuando hacerlo cambiaría su contrato pedagógico. Los benchmarks solo fijan correctness y forma de medición, no tiempos universales.

## 12. Changes applied

1. Se corrigió el tratamiento matemático de `top-k` en CS-M1 y CS-M5.
2. Se alinearon con el Curriculum los títulos H1 de CS-M3–CS-M9.
3. Los estados de CS-M1–CS-M10 cambiaron de `review candidate` a `approved`.
4. El índice del Study Guide y el CHANGELOG registran el gate acumulativo.
5. Se añadió este reporte; no se generó ningún módulo de un track posterior.

## 13. Remaining risks

- Los resultados temporales y de memoria dependen de Python, OS, filesystem, hardware y carga; los módulos ya exigen registrar ese entorno.
- Atomic replace y `fsync` no eliminan todos los failure modes de hardware/filesystem; CS-M7 conserva las garantías condicionadas.
- La concurrencia y la red tienen interleavings y fallos que una suite local no agota; los labs prueban invariantes y escenarios controlados, no distributed systems.
- Los mini projects son evidencia educativa, no una autorización para convertir estructuras in-memory en source of truth persistente.

Ningún riesgo residual bloquea el alcance de Computer Science Foundations.

## 14. Upstream issues

### MINOR — Prerequisites resumidos del Curriculum

**Ubicación:** introducción/prerequisites generales de `02_computer_science_foundations.md`.  
**Problema:** el resumen menciona un mínimo PF-M1–PF-M3 y PF-M6/PF-M8 para labs coordinados, mientras algunos módulos reales declaran PF-M5, PF-M9 o PF-M1–PF-M9.  
**Impacto:** una lectura aislada del resumen podría subestimar la ruta recomendada.  
**Estado:** no bloqueante: los metadatos de cada módulo y el README publicado son explícitos y el track comienza después del gate completo de Programming Foundations. Se recomienda alinear el resumen upstream en una futura revisión curricular; no se modificó la fuente canónica durante este gate.

## 15. Final gate

Se cumplen las condiciones de aprobación: cero hallazgos críticos, cobertura completa, dependencies coherentes, algoritmos y matemáticas correctos, sistemas y portabilidad tratados con límites, código validado, labs/mini projects/build resolubles, continuidad EIDOLON y cumplimiento editorial.

COMPUTER SCIENCE FOUNDATIONS GLOBAL GATE: PASS
