<!--
Artifact: Engineering Curriculum — track specification
Architecture version: v0.3.0
Curriculum content source: EIDOLON_ENGINEERING_CURRICULUM_v0.2.docx
-->

# **Computer Science Foundations**

**Track:** Computer Science Foundations  
**Competencias:** D2.1–D2.3, D3.1–D3.3  
**Fase:** P0  
**Nivel objetivo:** Aplicado  
**Prerequisites:** PF-M1–PF-M3; PF-M6 y PF-M8 para laboratorios coordinados de concurrencia  
**Build:** EIDOLON 0.0b  
**Curriculum source:** CS  
**Status:** active

> **Nota de migración:** esta metadata no reemplaza el contrato CS.1. El resto del contenido conserva la especificación v0.2.


*Estructuras, algoritmos y sistemas conectados a decisiones concretas de EIDOLON; rigor suficiente sin convertir P0 en una licenciatura completa.*

## CS.1 Contrato del dominio

**Objetivo.** Elegir y justificar estructuras de datos, estimar costo, modelar relaciones/estados y diagnosticar el comportamiento básico del sistema operativo, la memoria, la concurrencia y la red. Cada concepto se demuestra en un problema de EIDOLON, no en ejercicios aislados de pizarra.

**Prerrequisitos.** PF-M1–PF-M3 y capacidad de escribir funciones, colecciones y tests en Python. Para laboratorios de procesos/threads/async se requiere PF-M6 y PF-M8 o estudio paralelo coordinado.

**Nivel esperado.** Aplicado para D2.1–D2.3 y D3.1–D3.3: implementa y depura casos comunes, explica tradeoffs y reconoce límites. No se exige investigación en algoritmos, kernel development ni distributed systems.

**Esfuerzo.** Minimum path 20 h · Recommended path 35 h · Deep mastery path 55 h. Sumado a PF produce exactamente las 50/90/140 h del gate P0.

| **Principio de selección.** Se estudia una estructura porque una operación de EIDOLON la necesita: hash maps para identidad/cache, queues para consolidación, graphs para relaciones/evidencia y Big O para detectar cuándo retrieval o replay dejan de escalar. |
|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## CS.2 Prioridad de conceptos

| **Prioridad**  | **Conceptos**                                                                                                                                                                                                                                                                                                                               | **Decisión curricular**                      |
|----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|----------------------------------------------|
| **\[MUST\]**   | Arrays/dynamic arrays; hash maps/sets; stacks; queues/deques; trees y heaps a nivel de uso; graphs; recursion; sorting/searching; Big O; lógica/conjuntos; state machines; Unicode/serialización/tiempo; memoria; filesystem; procesos; threads; race conditions; locks; async model; DNS, TCP, TLS y HTTP; CPU/cache/I/O a nivel práctico. | Forma el gate antes de P1–P4.                |
| **\[SHOULD\]** | Implementar linked list una vez; binary search; BFS/DFS; top-k con heap; stable sorting; amortized analysis; virtual memory intuition; atomic file replacement; IPC básico; thread pools; bounded queues; sockets/packet tracing local.                                                                                                     | Profundiza decisiones frecuentes de EIDOLON. |
| **\[NICE\]**   | Balanced trees, tries, Bloom filters, union-find, shortest paths ponderados, memory mapping, page faults medidos, signals y scheduling más detallado.                                                                                                                                                                                       | Útil ante un problema medido; no bloquea P1. |
| **\[LATER\]**  | Dynamic programming extensivo, computational complexity formal, lock-free structures, SIMD/GPU, compiler construction, kernel internals, consensus y sistemas distribuidos.                                                                                                                                                                 | Fuera del problema actual o D17 por trigger. |

## CS.3 Mapa problema → estructura

| **Concepto**             | **Problema EIDOLON**                                             | **Señal de uso incorrecto**                                              |
|--------------------------|------------------------------------------------------------------|--------------------------------------------------------------------------|
| **Array/list**           | Secuencia ordenada de mensajes o eventos pequeños.               | Inserciones arbitrarias dominantes o escaneos repetidos por ID.          |
| **Hash map/set**         | Acceso por entity_id, deduplicación y cache derivado.            | Usarlo como source of truth persistente o depender del orden incidental. |
| **Stack**                | Undo local, recorrido DFS y parsing anidado.                     | Necesidad de procesar por antigüedad o prioridad.                        |
| **Queue/deque**          | Jobs de consolidación, backpressure y BFS.                       | Queue ilimitada o trabajo sin idempotency/cancellation.                  |
| **Linked list**          | Aprender enlaces y costo de inserción; rara vez ideal en Python. | Elegirla por O(1) teórico ignorando cache locality y traversal.          |
| **Tree/heap**            | Jerarquías y top-k de evidencia/candidatos.                      | Construir árbol complejo cuando sort/bisect o heapq basta.               |
| **Graph**                | Personas, eventos, claims, hipótesis y evidence edges.           | Confundir relación con hecho o perder provenance/dirección.              |
| **Big O + benchmark**    | Predecir el punto donde replay/retrieval deja de escalar.        | Optimizar por notación sin medir constantes, I/O o distribución real.    |
| **Process/thread/async** | Separar CPU, blocking I/O y concurrencia cooperativa.            | Compartir estado sin ownership o usar async como paralelismo CPU.        |

## CS.4 Teoría y habilidades por módulo

### CS-M1 — Complejidad, medición y modelos de costo

**Problema que resuelve.** Predecir cuándo una implementación simple deja de ser viable y evitar optimización prematura.

**Teoría.** Big O describe crecimiento asintótico, no tiempo real. Deben distinguirse mejor/medio/peor caso, costo amortizado y complejidad espacial. O(1) puede ocultar hashing, colisiones o I/O; O(n log n) puede vencer una solución sofisticada con n pequeño. El análisis produce una hipótesis y el benchmark representativo decide.

**Aplicación en EIDOLON.** Comparar scan de eventos O(n), índice hash esperado O(1), sorting O(n log n), top-k O(n log k) y recorridos de graph O(V+E). Definir tamaños a los que replay/retrieval exigirán otra estrategia.

**Cuándo no usarlo o sobredimensionarlo.** No rechazar una solución legible por constantes teóricas ni extrapolar microbenchmarks sintéticos a producción.

**Profundidad matemática.** Level 2 — mathematical understanding: funciones de crecimiento, logaritmos, sumatorias simples y costo amortizado intuitivo.

**Habilidades prácticas**

- Derivar el costo de bucles, nested loops y operaciones de colecciones.

- Diseñar benchmarks con warmup, tamaños crecientes y métricas de tiempo/memoria.

- Explicar el punto de cruce y qué dato invalidaría la decisión.

### CS-M2 — Arrays, hash maps y sets

**Problema que resuelve.** Representar secuencias e índices de identidad con operaciones previsibles.

**Teoría.** Un array dinámico almacena referencias contiguas y crece por capacidad; acceso por índice es O(1), insertar en medio O(n) y append amortizado O(1). Un hash map calcula un bucket desde una clave hashable; búsqueda promedio O(1) depende de distribución, resize e igualdad correctos. Un set representa pertenencia/unicidad. Cache locality y overhead importan además de Big O.

**Aplicación en EIDOLON.** Lista para orden cronológico acotado; dict para entity_id → entity; set para deduplicar SourceRef. Los índices son proyecciones reconstruibles, no autoridad canónica.

**Cuándo no usarlo o sobredimensionarlo.** No usar índice hash para consultas por rango ni conservar dos fuentes de verdad que puedan divergir.

**Profundidad matemática.** Level 1 — practical formulas: load factor, amortización y memoria por elemento.

**Habilidades prácticas**

- Implementar un hash map didáctico con colisiones y luego usar \`dict\` profesionalmente.

- Medir list scan vs dict lookup bajo distintos n y distribuciones.

- Diseñar claves estables y evitar objetos mutables como identidad.

### CS-M3 — Stacks, queues, deques y linked lists

**Problema que resuelve.** Elegir orden de procesamiento y ownership de trabajo.

**Teoría.** Stack es LIFO; queue es FIFO; deque permite extremos O(1). Una linked list conecta nodos y favorece inserciones conocidas, pero su traversal es O(n) y su locality suele ser pobre. En Python, \`collections.deque\` es la opción normal para queues; implementar nodos sirve para entender referencias, no para reemplazar la librería.

**Aplicación en EIDOLON.** Stack para undo/DFS; queue acotada para consolidación; deque para BFS y buffers. Cada job necesita ID, estado, retry policy y límite; una estructura no resuelve idempotencia.

**Cuándo no usarlo o sobredimensionarlo.** No usar \`list.pop(0)\` como queue grande ni linked list por dogma académico.

**Profundidad matemática.** Level 1 — practical formulas: costo por operación y Little's Law solo como intuición en Deep mastery.

**Habilidades prácticas**

- Implementar y probar stack/queue, overflow lógico y empty-state.

- Modelar backpressure con queue bounded.

- Comparar linked list y deque considerando memoria y locality.

### CS-M4 — Recursion, searching y sorting

**Problema que resuelve.** Recorrer estructuras y ordenar/ubicar datos sin perder estabilidad ni desbordar la pila.

**Teoría.** Recursion requiere caso base, progreso y límite de profundidad; una versión iterativa suele ser más segura en Python. Linear search no exige orden; binary search exige colección ordenada y reduce el espacio a la mitad. Sorting estable preserva orden relativo y permite criterios múltiples. Python usa Timsort y \`key=\`; reimplementar algoritmos sirve para entender, no para producción.

**Aplicación en EIDOLON.** Binary search ubica un rango temporal en eventos ordenados; stable sort prioriza fecha, source y tie-breaker determinista; recursion/iteration recorre claims anidados o graph trees acotados.

**Cuándo no usarlo o sobredimensionarlo.** No mantener una lista ordenada con inserciones costosas si el patrón es batch; no usar recursion sobre graphs sin visited set.

**Profundidad matemática.** Level 2 — mathematical understanding: log2 n, recurrencias simples y estabilidad.

**Habilidades prácticas**

- Implementar linear/binary search y demostrar precondiciones.

- Diseñar claves de orden total determinista para replay.

- Convertir una recursion peligrosa a loop con stack explícito.

### CS-M5 — Trees, heaps y top-k

**Problema que resuelve.** Representar jerarquías y mantener los mejores candidatos sin ordenar todo.

**Teoría.** Un tree impone parent/child y caminos únicos; traversal puede ser DFS/BFS. Un binary search tree ordena claves, pero puede degradarse sin balance. Un heap mantiene el mínimo o máximo prioritario en O(log n) por inserción/extracción; no ofrece búsqueda arbitraria rápida. Para top-k, un heap de tamaño k evita ordenar n elementos completos.

**Aplicación en EIDOLON.** Tree para taxonomía explícita; heap para seleccionar k evidencias/candidatos por score. El score no sustituye provenance ni reglas de empate.

**Cuándo no usarlo o sobredimensionarlo.** No construir un BST propio en producción cuando \`heapq\`, \`bisect\`, sort o la base de datos futura resuelven el patrón.

**Profundidad matemática.** Level 2 — mathematical understanding: altura logarítmica y O(n log k).

**Habilidades prácticas**

- Implementar traversal y detectar cycles cuando la entrada viola la premisa de tree.

- Usar heapq con tie-breaker estable y explicar la invariante del heap.

- Comparar sort completo vs top-k con benchmark.

### CS-M6 — Graphs, lógica, conjuntos y máquinas de estado

**Problema que resuelve.** Modelar relaciones, evidencia y transiciones sin convertir inferencias en hechos.

**Teoría.** Un graph contiene vertices y edges dirigidos/no dirigidos, con metadata y posiblemente peso. BFS explora por distancia en aristas; DFS profundiza y detecta estructura. Sets y lógica permiten expresar predicados e invariantes. Una finite-state machine enumera estados y transiciones válidas; eventos causan transiciones, no asignaciones arbitrarias.

**Aplicación en EIDOLON.** Vertices persona/event/claim/hypothesis y edges supports/refutes/mentions/derived_from. MemoryCandidate transita proposed → approved/rejected → superseded/purged; una edge siempre conserva tipo y provenance.

**Cuándo no usarlo o sobredimensionarlo.** No adoptar graph database ni probabilidades en P0; primero modelar adjacency lists y queries reales.

**Profundidad matemática.** Level 2 — mathematical understanding: conjuntos, relaciones, lógica proposicional básica y O(V+E).

**Habilidades prácticas**

- Construir adjacency list dirigida con edge metadata.

- Implementar BFS/DFS con visited set y path reconstruction.

- Definir una state machine y property tests de transiciones inválidas.

### CS-M7 — Fundamentos de sistemas operativos: memoria y filesystem

**Problema que resuelve.** Entender por qué el programa puede agotar RAM, perder datos o ver estado distinto al esperado.

**Teoría.** El proceso observa un address space virtual; stack y heap son modelos útiles, aunque Python administra objetos y referencias. Garbage collection no equivale a liberar recursos externos. El OS cachea páginas y archivos; buffer, flush y durable write no son sinónimos. Rename atómico, journaling y fsync tienen garantías dependientes del sistema. Paths, permisos y environment forman parte del contrato.

**Aplicación en EIDOLON.** Replay streaming limita RAM; export atómico evita archivos parciales; logs y temporales tienen lifecycle; el runtime debe conocer qué datos persisten tras crash.

**Cuándo no usarlo o sobredimensionarlo.** No prometer durabilidad total con \`close()\` ni optimizar allocations sin medición.

**Profundidad matemática.** Level 1 — practical formulas: bytes por objeto, working set y relación RAM/I/O.

**Habilidades prácticas**

- Medir memoria con tracemalloc y explicar crecimiento retenido vs temporal.

- Inspeccionar file descriptors, permisos, paths y environment.

- Diseñar un crash test para una escritura atómica.

### CS-M8 — Procesos, threads y concurrencia

**Problema que resuelve.** Ejecutar trabajo concurrente sin races, deadlocks ni recursos huérfanos.

**Teoría.** Un process posee address space y recursos del OS; threads comparten memoria dentro del proceso. Scheduling interleaves ejecución y vuelve no determinista el orden. Race condition aparece cuando el resultado depende del interleaving. Locks protegen invariantes, no líneas aisladas; deadlock surge por ciclos de espera. Async coordina I/O cooperativo; process pools ayudan a CPU-bound con costos de serialización.

**Aplicación en EIDOLON.** Jobs de importación/consolidación deben ser idempotentes, acotados y cancelables. Un solo writer puede simplificar el journal P0; compartir una dict mutable entre threads requiere ownership explícito.

**Cuándo no usarlo o sobredimensionarlo.** No agregar threads/processes si una secuencia síncrona cumple; no mantener un lock durante I/O lento.

**Profundidad matemática.** Level 1 — practical formulas: throughput, latency, contention y speedup intuitivo.

**Habilidades prácticas**

- Reproducir un race y corregir la invariante con ownership o lock.

- Detectar un deadlock mediante orden de locks y timeout diagnóstico.

- Elegir sync, async, thread o process según CPU/I/O/aislamiento.

### CS-M9 — Networking básico: DNS, TCP, TLS y HTTP

**Problema que resuelve.** Comprender el request path antes de construir APIs o asumir que 'localhost' elimina toda frontera de seguridad.

**Teoría.** DNS resuelve nombres; IP enruta; TCP ofrece un byte stream confiable y ordenado, no mensajes; TLS autentica/cifra el canal según configuración; HTTP define semántica de requests/responses sobre transporte. Connection, timeout, retry e idempotency son decisiones separadas. Loopback reduce exposición de red, pero browser, procesos locales y egress siguen siendo threat boundaries.

**Aplicación en EIDOLON.** Antes de FastAPI, se traza un request local: resolución, connect, bytes, headers, response y cierre. Se identifica qué cambiará si EIDOLON 1.1 se expone remotamente.

**Cuándo no usarlo o sobredimensionarlo.** No implementar protocolos propios ni usar WebSockets antes de una necesidad bidireccional; P0 solo observa y simula.

**Profundidad matemática.** Level 1 — practical formulas: latencia acumulada, throughput y timeout budgets.

**Habilidades prácticas**

- Explicar framing de mensajes sobre un byte stream.

- Usar herramientas del sistema para observar DNS, sockets y conexiones locales.

- Distinguir error de resolución, connect, TLS, HTTP y aplicación.

### CS-M10 — Arquitectura de computadoras para programadores

**Problema que resuelve.** Relacionar decisiones de código con CPU, memoria, caches, almacenamiento e I/O sin estudiar hardware por separado del software.

**Teoría.** La CPU ejecuta instrucciones sobre registros y memoria; caches explotan localidad temporal/espacial; RAM es rápida pero volátil; storage persiste con mayor latencia; syscalls cruzan la frontera al kernel. Representación binaria, byte order, alignment y cache misses explican parte del rendimiento. En Python importan el overhead de objetos y la locality aunque el intérprete oculte instrucciones.

**Aplicación en EIDOLON.** Elegir streaming vs materialización, batch size, estructura compacta e índice tiene impacto en working set y I/O. El objetivo es diagnosticar, no microoptimizar el intérprete.

**Cuándo no usarlo o sobredimensionarlo.** No escribir assembly, SIMD o C extensions en P0; únicamente activar esa rama con profiling y bottleneck real.

**Profundidad matemática.** Level 1 — practical formulas: unidades de bytes, órdenes de magnitud y jerarquía de memoria.

**Habilidades prácticas**

- Trazar desde código Python hasta runtime, syscall, kernel y dispositivo.

- Comparar acceso secuencial/aleatorio y explicar locality.

- Leer métricas básicas de CPU, RAM, I/O y file descriptors durante un laboratorio.

## CS.5 Laboratorios

| **ID**     | **Laboratorio**            | **Evidencia y conexión EIDOLON**                                                             |
|------------|----------------------------|----------------------------------------------------------------------------------------------|
| **CS-L01** | Curvas de crecimiento      | Medir O(n), O(n log n) y lookup hash para 10²–10⁶ elementos; explicar punto de cruce.        |
| **CS-L02** | List vs dict entity lookup | Índice derivado por entity_id, memoria medida y test de rebuild por replay.                  |
| **CS-L03** | Hash collisions            | Hash map didáctico con chaining; claves inválidas y análisis de peor caso.                   |
| **CS-L04** | Undo stack                 | Aplicar/deshacer corrections sin borrar eventos; límite y semántica de empty stack.          |
| **CS-L05** | Consolidation queue        | Deque bounded con job IDs, retry count, idempotency y backpressure simulada.                 |
| **CS-L06** | Linked list reality check  | Implementar nodos, comparar traversal/memoria con list/deque y justificar por qué no usarla. |
| **CS-L07** | Timeline search            | Stable sort + bisect para rango temporal; tie-breakers deterministas y timestamps aware.     |
| **CS-L08** | Top-k evidence             | Heap de k candidatos con score/tie-breaker; comparar con sort completo.                      |
| **CS-L09** | Tree traversal             | Taxonomía sintética; DFS/BFS, depth limit y entrada cíclica rechazada.                       |
| **CS-L10** | Evidence graph             | Adjacency list typed, supports/refutes edges, BFS path y provenance obligatorio.             |
| **CS-L11** | Memory state machine       | Transiciones proposed/approved/rejected/superseded/purged + property tests.                  |
| **CS-L12** | Memory pressure            | Procesar JSONL streaming vs materializado; timeit/tracemalloc y explicación del working set. |
| **CS-L13** | Crash-safe file            | Inyectar crash entre temp/write/rename y verificar estados recuperables.                     |
| **CS-L14** | Process inspection         | PID, environment, cwd, open files y exit codes; diagnóstico documentado.                     |
| **CS-L15** | Race condition             | Counter/índice compartido con interleaving; fix por ownership y luego lock.                  |
| **CS-L16** | Deadlock and cancellation  | Dos locks o tasks bloqueadas; timeout, dump y orden global de adquisición.                   |
| **CS-L17** | Local request trace        | DNS/hosts, TCP connect y request HTTP local observado; clasificar cada fallo.                |
| **CS-L18** | Locality and I/O           | Acceso secuencial vs aleatorio, batch sizes y métricas; ADR sobre streaming.                 |

## CS.6 Mini proyectos

### CS-MP1 — In-memory Event Index

**Alcance.** Carga eventos sintéticos y mantiene índices por ID, persona, tag y rango temporal. Incluye stable ordering, rebuild, métricas de memoria y comparación scan/index.

**Evidencia.** D2.1 y D2.3; benchmark y ADR con umbral de cambio.

### CS-MP2 — Consolidation Scheduler

**Alcance.** Queue bounded con prioridades top-k, estados de job, retries idempotentes, cancelación y simulación de fallos. Versión síncrona primero; async después.

**Evidencia.** D3.2; pruebas de race, backpressure, cancelación y replay.

### CS-MP3 — Relationship & Evidence Graph

**Alcance.** Graph dirigido de personas, events, claims e hypotheses; edges typed supports/refutes/mentions/derived_from; BFS/DFS y state machine de hypotheses.

**Evidencia.** D2.2; ninguna edge sin provenance y ninguna inferencia promovida a hecho.

## CS.7 Proyecto de integración — EIDOLON 0.0b

| **Build.** Motor de estructuras y estados sobre el journal 0.0a. Sigue siendo local, determinista, in-memory + JSONL y con datos sintéticos; su objetivo es revelar costos y fallas antes de persistencia transaccional o APIs. |
|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

- Reconstruir índices de entities/tags/timestamps desde el journal; verificar que nunca son source of truth.

- Exponer consultas de lookup, range, top-k y graph path con contrato y complejidad documentados.

- Modelar MemoryCandidate e Hypothesis mediante state machines con transiciones explícitas y auditables.

- Procesar jobs de consolidación sintéticos con queue bounded, idempotency key, retry budget y cancelación.

- Incluir benchmark escalonado para 10³, 10⁴, 10⁵ y, si el equipo lo permite, 10⁶ eventos; medir tiempo y memoria.

- Inyectar corrupción de archivo, race, deadlock/timeout y trabajo duplicado; documentar diagnóstico y recuperación.

- Trazar una operación desde CLI → process → filesystem y un request HTTP local de laboratorio; no construir servidor.

- Entregar Complexity Budget, System Model, ADR de estructuras y tabla de triggers para futuras migraciones.

**Criterio de aceptación.** El estudiante explica por qué cada estructura fue elegida, predice su costo, confirma o corrige la predicción con mediciones y demuestra comportamiento seguro bajo fallas. Un revisor puede sustituir una estrategia sin cambiar los invariantes del dominio.

## CS.8 Errores comunes

- Recitar Big O sin definir n, operación dominante, distribución o memoria.

- Elegir una estructura por su mejor caso y omitir peor caso o costo de mantener índices.

- Usar list como queue con \`pop(0)\` o linked list por prestigio académico.

- Aplicar binary search a datos no ordenados o con orden inconsistente.

- Perder estabilidad/tie-breaker y producir replay no determinista.

- Recorrer graph sin visited set, direction o edge type.

- Promover un path o score del graph a hecho sin evidencia/provenance.

- Confundir process, thread, coroutine, concurrencia y paralelismo.

- Proteger una línea con lock en lugar de la invariante completa.

- Usar queues ilimitadas y llamar 'escalable' al crecimiento de memoria.

- Suponer que close/flush equivale a durabilidad ante crash.

- Tratar TCP como mensajes o retry como operación segura sin idempotencia.

- Creer que loopback elimina riesgos del browser, procesos locales o egress.

- Optimizar CPU ignorando I/O, serialization, allocation y working set.

- Adoptar graph DB, Redis, workers o microservices antes del trigger medido.

## CS.9 Preguntas de evaluación

17. Define n y deriva el costo de reconstruir tres índices desde un journal de n eventos. **\[Complejidad\]**

18. ¿Cuándo O(n) puede ser preferible a O(1) esperado en EIDOLON? **\[Complejidad\]**

19. Diseña las estructuras para lookup por ID, rango temporal, top-k y path de evidencia; justifica cada una. **\[Estructuras\]**

20. ¿Por qué una linked list suele ser mala elección en Python aunque inserte en O(1)? **\[Estructuras\]**

21. Explica la precondición de binary search y cómo un timestamp ambiguo puede invalidarla. **\[Algoritmos\]**

22. Convierte un DFS recursivo en iterativo y preserva path/provenance. **\[Algoritmos\]**

23. Modela supports/refutes sin asumir que una relación vuelve verdadero un Claim. **\[Graphs\]**

24. ¿Qué transiciones deben prohibirse después de \`purged\` y cómo se prueban? **\[Estados\]**

25. Distingue stack/heap del proceso, heap data structure y storage persistente. **\[Sistemas\]**

26. ¿Qué puede quedar en RAM, page cache o storage después de escribir y cerrar un archivo? **\[Sistemas\]**

27. Reproduce un race en un índice compartido y corrígelo primero con ownership, luego compara con lock. **\[Concurrencia\]**

28. Diseña cancellation y retry para que un job duplicado no duplique efectos. **\[Concurrencia\]**

29. Clasifica una falla como DNS, TCP, TLS, HTTP o aplicación y especifica la evidencia necesaria. **\[Networking\]**

30. ¿Por qué TCP no preserva mensajes y qué implicación tiene para un protocolo? **\[Networking\]**

31. Explica cómo cache locality y object overhead pueden contradecir una lectura superficial de Big O. **\[Arquitectura\]**

32. Define un trigger cuantitativo para abandonar scan exacto, JSONL o ejecución síncrona, sin elegir aún la tecnología sustituta. **\[Diseño\]**

## CS.10 Criterio de dominio y checkpoints

| **Competencia** | **Evidencia obligatoria**                                 | **Umbral**                                                                  |
|-----------------|-----------------------------------------------------------|-----------------------------------------------------------------------------|
| **D2.1**        | Event Index + problemas + benchmark explicado.            | Elige estructura y deriva costo; medición confirma o corrige la hipótesis.  |
| **D2.2**        | Evidence Graph + state machines + invariantes.            | BFS/DFS y transiciones correctas; provenance y contradicción preservadas.   |
| **D2.3**        | Edge-case suite Unicode/serialización/tiempo/precisión.   | Ninguna corrupción silenciosa; orden determinista y round-trip probado.     |
| **D3.1**        | Diagnóstico de process, memory, filesystem y environment. | Explica recursos, lifecycle y crash behavior con evidencia del sistema.     |
| **D3.2**        | Race/failure labs + scheduler idempotente.                | Cancelación, backpressure y duplicate work probados; sin tasks huérfanas.   |
| **D3.3**        | Local request trace y clasificación de fallas.            | Traza DNS→TCP→TLS/HTTP conceptualmente y no confunde transporte/aplicación. |

### CHECKPOINT CS-C1 — Estructuras y costo

- ¿Puedes implementar list/dict/stack/queue y explicar operaciones dominantes y memoria?

- ¿Puedes derivar y luego medir el crecimiento con tamaños distintos?

- ¿Puedes reconocer cuándo la solución más simple es suficiente?

### CHECKPOINT CS-C2 — Relaciones y estados

- ¿Puedes implementar search/sort/tree/heap/graph sin perder determinismo?

- ¿Puedes separar edge, evidencia e inferencia y probar transiciones inválidas?

- ¿Puedes enseñar BFS/DFS, top-k y state machines con ejemplos de EIDOLON?

### CHECKPOINT CS-C3 — Sistemas y fallas

- ¿Puedes distinguir memory, filesystem, process, thread y coroutine con un trazado real?

- ¿Puedes provocar y diagnosticar race, deadlock/timeout y crash de archivo?

- ¿Puedes seguir un request local y explicar qué cambia antes de async web/backend?

## CS.11 Recursos recomendados

**Curso universitario.** [*MIT OCW 6.006 — Introduction to Algorithms*](https://ocw.mit.edu/courses/6-006-introduction-to-algorithms-spring-2020/) — Priorizar lectures 1–10: modelos, estructuras, sorting, hashing, trees, heaps, BFS y DFS. Weighted paths/dynamic programming son NICE/LATER.

**Libro de referencia.** [*Introduction to Algorithms, 4th Edition*](https://mitpress.mit.edu/9780262046305/introduction-to-algorithms/) — Usar capítulos selectivos sobre growth, hashing, sorting, trees, heaps y elementary graph algorithms; no lectura lineal completa.

**Libro aplicado.** [*The Algorithm Design Manual, 3rd Edition*](https://link.springer.com/book/10.1007/978-3-030-54256-6) — War stories, modelado del problema y selección de estructuras; útil para pasar de especificación a algoritmo.

**Libro abierto.** [*Operating Systems: Three Easy Pieces*](https://pages.cs.wisc.edu/~remzi/OSTEP/) — Virtualization, threads/locks, concurrency y persistence; leer capítulos de procesos, address spaces, threads, locks, files y crash consistency.

**Libro/sitio académico.** [*Computer Systems: A Programmer's Perspective*](https://csapp.cs.cmu.edu/) — Machine-level representation, memory hierarchy, linking, exceptional control flow, virtual memory, system I/O y network programming desde la perspectiva del programador.

**Curso práctico.** [*Nand2Tetris*](https://www.nand2tetris.org/) — Deep mastery opcional: proyectos 1–5 para lógica, memoria, machine language y computer architecture; no es gate P0 completo.

**Guía.** [*Beej's Guide to Network Concepts*](https://beej.us/guide/bgnet0/html/) — DNS/IP/TCP/sockets y trazado conceptual antes de HTTP/backend.

**Estándares.** [*RFC 9110 — HTTP Semantics*](https://www.rfc-editor.org/rfc/rfc9110.html) — Lectura selectiva en Deep mastery; semántica, no implementación de servidor en este track.

## CS.12 Rutas de esfuerzo

| **Ruta**              | **Horas** | **Contenido y evidencia**                                                                                                                                                  |
|-----------------------|-----------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Minimum path**      | ~20 h     | CS-M1–M4, M6–M9 esenciales; L01–L05, L07, L10–L17; MP1; EIDOLON 0.0b con graph/state/queue simplificados y gate completo.                                                  |
| **Recommended path**  | ~35 h     | Todo Minimum + CS-M5 y M10; L06, L08–L09, L18; MP2 y MP3; benchmarks de tiempo/memoria y failure labs completos.                                                           |
| **Deep mastery path** | ~55 h     | Todo Recommended + árboles balanceados/tries/Bloom como lectura, OSTEP/CS:APP selectivos, Nand2Tetris 1–5, análisis amortizado más formal y defensa de migration triggers. |

## CS.13 Gate conjunto de Foundation Tracks

- EIDOLON 0.0a y 0.0b se instalan y ejecutan en un environment limpio con datos sintéticos.

- Las 51 competencias de la Parte I permanecen sin renumerar; esta entrega aporta evidencia concreta para D1.1–D3.3.

- El estudiante implementa, prueba, integra, evalúa, critica y enseña; no solo completa labs.

- Unicode, tiempo, serialización, replay, provenance, corrections y state transitions tienen tests de edge cases.

- La selección de estructuras incluye complejidad, benchmark, memoria y trigger de migración.

- Race, cancellation, timeout, crash parcial y duplicate work tienen failure labs reproducibles.

- No existen dependencias o capas prematuras: sin FastAPI, Pydantic, SQL/PostgreSQL, React, Docker, Ollama o LLMs.

- Se aprobaron code review, README, ADR, threat notes P0 y defensa oral; una falla crítica no se compensa con promedio.

| **Próximo gate permitido.** Tras aprobar PF-C3 y CS-C3 se puede iniciar P1: modelado epistemológico sin frameworks. Backend y Database Engineering continúan bloqueados hasta completar el orden canónico P1 → P2 → P3. |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
