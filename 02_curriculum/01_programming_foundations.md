<!--
Artifact: Engineering Curriculum — track specification
Architecture version: v0.3.0
Curriculum content source: EIDOLON_ENGINEERING_CURRICULUM_v0.2.docx
-->

# **Programming Foundations**

**Track:** Programming Foundations  
**Competencias:** D1.1–D1.3; soporte D2.3, D3.1, D11.1, D12.3  
**Fase:** P0  
**Nivel objetivo:** Profesional; Working Knowledge para async/await  
**Prerequisites:** ninguno  
**Build:** EIDOLON 0.0a  
**Curriculum source:** PF  
**Status:** active

> **Nota de migración:** esta metadata no reemplaza el contrato PF.1. El resto del contenido conserva la especificación v0.2.


*Python moderno e ingeniería de software suficiente para modelar, probar y mantener EIDOLON sin depender de frameworks.*

## PF.1 Contrato del dominio

**Objetivo.** Transformar una especificación acotada en software Python legible, tipado, probado, depurable y empaquetado; explicar las decisiones y conservar evidencia en Git. El resultado no es memorizar sintaxis, sino poder construir un núcleo determinista que sobreviva cambios, datos defectuosos y revisión de otra persona.

**Prerrequisitos.** Ninguno. El diagnóstico inicial puede reconocer experiencia previa en Python o Java, pero no exime de demostrar el gate con código nuevo y tests.

**Nivel esperado.** Profesional para D1.1–D1.3: diseña, implementa, prueba, revisa y entrega componentes Python pequeños y medianos. Working Knowledge para async/await: comprende event loop, cancelación y ownership de recursos antes de la fase web.

**Competencias canónicas.** D1.1 Python idiomático, tipos y manejo de errores; D1.2 debugging, logging, packaging y dependencias; D1.3 Git, documentación, ADR y revisión. Refuerza D2.3, D3.1, D11.1 y D12.3.

**Esfuerzo.** Minimum path 30 h · Recommended path 55 h · Deep mastery path 85 h. Estas cifras ocupan la porción de Programming Foundations dentro del presupuesto canónico P0 de 50/90/140 h.

| **Gate de salida.** Implementa desde una especificación, explica cada frontera, depura con evidencia y entrega un paquete reproducible con tests de Unicode, tiempo, errores y serialización. Copiar un tutorial o completar horas no satisface el gate. |
|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

## PF.2 Prioridad de conceptos

| **Prioridad**  | **Conceptos**                                                                                                                                                                                                                                                                                                 | **Decisión curricular**                                      |
|----------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|--------------------------------------------------------------|
| **\[MUST\]**   | Python 3.14 estable; valores, identidad y mutabilidad; funciones; LEGB/scopes; módulos y paquetes; dataclasses; type hints; excepciones; colecciones y comprehensions; iteradores; JSON/JSONL; archivos; Unicode; pathlib; venv; pyproject.toml; dependencias; pytest; debugging; logging; Git; README y ADR. | Bloquea P1. Debe demostrarse sin framework.                  |
| **\[SHOULD\]** | Generators; decorators; context managers; protocolos; composición; dependency groups; CLI con argparse; fixtures y parametrización; property-based testing introductorio; async/await, cancellation y timeout básicos.                                                                                        | Se usa en P1–P5 y reduce errores de lifecycle.               |
| **\[NICE\]**   | Structural pattern matching; Protocol y generics más avanzados; profiling con cProfile/timeit/tracemalloc; importlib.resources; builds y wheels locales.                                                                                                                                                      | Aporta claridad o diagnóstico, pero no bloquea P1.           |
| **\[LATER\]**  | Metaclasses, descriptors avanzados, C extensions, GIL internals profundos, optimización de bytecode, publicación pública en PyPI y frameworks de dependency injection.                                                                                                                                        | Solo por necesidad real; no mejora el núcleo P0 por sí solo. |

## PF.3 Secuencia de módulos

| **Módulo** | **Resultado observable**                                                       | **Ruta**       |
|------------|--------------------------------------------------------------------------------|----------------|
| **PF-M1**  | Lee y razona sobre ejecución, objetos, mutabilidad, Unicode y precisión.       | Minimum        |
| **PF-M2**  | Descompone comportamiento en funciones con contratos y scopes explícitos.      | Minimum        |
| **PF-M3**  | Elige colecciones, comprehensions e iteración sin materializar de más.         | Minimum        |
| **PF-M4**  | Organiza módulos/paquetes y reproduce el environment con pyproject.toml.       | Minimum        |
| **PF-M5**  | Modela Event/Claim con dataclasses, composición y type hints.                  | Minimum        |
| **PF-M6**  | Diseña errores, lifecycle de recursos y persistencia JSONL segura.             | Minimum        |
| **PF-M7**  | Usa decorators y context managers donde expresan una política real.            | Recommended    |
| **PF-M8**  | Comprende async/await, tasks, cancellation y backpressure en simulación local. | Recommended    |
| **PF-M9**  | Prueba, depura, registra y reproduce fallas; revisa código con Git/ADR.        | Minimum → Deep |

## PF.4 Teoría y habilidades por módulo

### PF-M1 — Modelo de ejecución y datos de Python

**Problema que resuelve.** Evitar código que parece funcionar hasta que aliasing, mutabilidad, Unicode o precisión numérica cambian el resultado.

**Teoría.** Python enlaza nombres con objetos; asignar no copia. Identidad (\`is\`) y equivalencia (\`==\`) responden preguntas distintas. Los tipos mutables pueden compartir estado; los inmutables favorecen invariantes. \`str\` representa Unicode y \`bytes\` representa octetos: encoding y decoding son fronteras explícitas. \`float\` es aproximado y no debe usarse como si fuera decimal exacto.

**Aplicación en EIDOLON.** Un Event sintético debe conservar texto español, emojis, timestamps timezone-aware y valores numéricos sin corrupción. La comparación de IDs, fechas y payloads no puede depender de coincidencias accidentales.

**Cuándo no usarlo o sobredimensionarlo.** No copiar estructuras defensivamente en todas partes ni convertir todo en objetos inmutables sin medir; se protege únicamente la frontera que posee un invariante.

**Profundidad matemática.** Level 1 — practical formulas: representación binaria intuitiva, precisión finita y costo de operaciones básicas.

**Habilidades prácticas**

- Distinguir \`None\`, truthiness, igualdad e identidad en tests positivos y negativos.

- Elegir \`str\`, \`bytes\`, \`int\`, \`float\`, \`Decimal\`, \`date\` y \`datetime\` según el contrato.

- Normalizar texto solo cuando el caso de uso lo exige y conservar el original para provenance.

- Reproducir un bug de aliasing y demostrar la corrección con un test.

### PF-M2 — Funciones, contratos y scopes

**Problema que resuelve.** Traducir reglas del dominio a unidades pequeñas sin depender de variables globales ni efectos invisibles.

**Teoría.** Una función define una frontera: entradas, salida, errores y efectos. Python resuelve nombres con LEGB (local, enclosing, global, builtins). Closures conservan bindings; los argumentos por defecto se evalúan una vez, por lo que un mutable puede filtrar estado entre llamadas. Funciones puras simplifican replay y testing; efectos como I/O deben quedar en adaptadores explícitos.

**Aplicación en EIDOLON.** Reglas como \`is_valid_transition\`, \`normalize_tags\` y \`filter_events\` deben ser deterministas y reutilizables por CLI, tests y fases posteriores.

**Cuándo no usarlo o sobredimensionarlo.** No fragmentar cada línea en una función ni usar closures para ocultar estado que debería ser un objeto explícito.

**Profundidad matemática.** Level 0 — intuition: composición, predicados y pre/postcondiciones.

**Habilidades prácticas**

- Escribir firmas claras con parámetros posicionales, keyword-only y valores por default seguros.

- Separar cálculo puro de filesystem, clock, environment y console I/O.

- Usar closures solo cuando el estado capturado es pequeño, estable y verificable.

- Diseñar docstrings y tests que expresen contrato, edge cases y errores.

### PF-M3 — Colecciones, comprehensions e iteración

**Problema que resuelve.** Procesar eventos y entidades sin búsquedas lineales innecesarias, duplicados silenciosos ni consumo de memoria accidental.

**Teoría.** \`list\`, \`tuple\`, \`dict\` y \`set\` expresan orden, identidad, asociación y unicidad. Una comprehension es apropiada cuando transforma o filtra de forma legible; una cadena de condiciones compleja merece funciones nombradas. El protocolo iterable separa producir elementos de almacenarlos. Iterator y generator permiten streaming y una sola pasada, pero implican consumo, errores tardíos y ownership de recursos.

**Aplicación en EIDOLON.** Los diccionarios indexan entidades por ID; los sets detectan tags duplicados; los generators recorren un journal grande sin cargarlo entero.

**Cuándo no usarlo o sobredimensionarlo.** No reemplazar consultas persistentes futuras con diccionarios gigantes; no usar comprehensions anidadas que oculten complejidad o side effects.

**Profundidad matemática.** Level 1 — practical formulas: costo amortizado y relación tiempo/memoria.

**Habilidades prácticas**

- Elegir colección según operaciones dominantes y documentar el tradeoff.

- Implementar iterables perezosos y detectar cuándo ya fueron consumidos.

- Construir índices derivados sin convertirlos en source of truth.

- Comparar una búsqueda lineal con un índice dict usando datos sintéticos.

### PF-M4 — Módulos, paquetes y dependency management

**Problema que resuelve.** Evitar scripts monolíticos, imports circulares y environments imposibles de reproducir.

**Teoría.** Un módulo es una unidad de namespace; un paquete organiza módulos alrededor de responsabilidades. La dirección de dependencias debe apuntar hacia contratos estables. \`venv\` aísla instalaciones; \`pyproject.toml\` concentra metadata y configuración; un lock o mecanismo reproducible fija resoluciones cuando el proyecto lo requiere. Dependencia directa y transitiva no son equivalentes, y todo paquete amplía la supply chain.

**Aplicación en EIDOLON.** La estructura mínima separa \`domain\`, \`application\`, \`adapters\`, \`cli\` y \`tests\` sin crear microservicios ni capas vacías.

**Cuándo no usarlo o sobredimensionarlo.** No publicar en PyPI ni introducir monorepo tooling en P0; tampoco envolver la standard library con abstractions sin valor.

**Profundidad matemática.** Level 0 — intuition: grafos de dependencias y ciclos.

**Habilidades prácticas**

- Crear y recrear virtual environments desde instrucciones limpias.

- Definir \`pyproject.toml\`, dependencias runtime/dev y entry point de CLI.

- Resolver un import circular cambiando responsabilidades, no con imports tardíos arbitrarios.

- Auditar por qué existe cada dependencia y eliminar la que no justifica su costo.

### PF-M5 — POO, dataclasses y type hints

**Problema que resuelve.** Representar estados e invariantes sin convertir el dominio en diccionarios anónimos o jerarquías rígidas.

**Teoría.** La programación orientada a objetos (OOP) agrupa estado y comportamiento cuando comparten invariantes. Encapsulation protege reglas, no oculta todo. Composition suele evolucionar mejor que inheritance. \`dataclass\` reduce boilerplate para value objects, pero no valida datos externos por sí sola. Los type hints documentan contratos para herramientas estáticas; no sustituyen validación runtime ni tests. Protocol favorece interfaces estructurales y dependencias pequeñas.

**Aplicación en EIDOLON.** Event, Claim, Correction y SourceRef pueden ser dataclasses con IDs, timestamps e invariantes; repositories se expresan como protocolos sin implementar aún base de datos.

**Cuándo no usarlo o sobredimensionarlo.** No crear una clase por cada función, herencia profunda, getters/setters triviales ni un \`BaseModel\` universal.

**Profundidad matemática.** Level 0 — intuition: conjuntos de estados válidos e invariantes.

**Habilidades prácticas**

- Diseñar value objects y entities con igualdad y mutabilidad deliberadas.

- Aplicar composición y Protocol en lugar de una jerarquía anticipatoria.

- Ejecutar un type checker y distinguir error estático de error de datos.

- Rechazar estados imposibles en constructores o funciones de creación.

### PF-M6 — Excepciones, archivos, JSON y lifecycle de recursos

**Problema que resuelve.** Conservar datos y recursos sin ocultar corrupción, dejar archivos abiertos o producir estados parciales.

**Teoría.** Una excepción representa una ruta de fallo que el caller puede manejar o propagar. La taxonomía debe distinguir entrada inválida, conflicto de dominio, I/O y bug. \`try/except/else/finally\` expresa recuperación y cleanup; capturar \`Exception\` sin acción destruye evidencia. Un context manager delimita adquisición y liberación. JSON es interoperable pero no conserva automáticamente tipos como datetime; JSONL favorece append y replay. La escritura temporal + rename reduce archivos parciales.

**Aplicación en EIDOLON.** El journal debe escribir una entrada por línea, validar schema/version, reportar la línea dañada y conservar el archivo original para diagnóstico.

**Cuándo no usarlo o sobredimensionarlo.** No usar pickle para datos no confiables ni inventar una base de datos sobre JSON; esta persistencia es deliberadamente transitoria para P0–P1.

**Profundidad matemática.** Level 0 — intuition: atomicidad y estados de fallo.

**Habilidades prácticas**

- Diseñar excepciones del dominio con mensajes accionables y chaining.

- Leer/escribir UTF-8 explícitamente mediante \`pathlib\` y context managers.

- Serializar dataclasses con versionado y convertir datetime de forma reversible.

- Implementar escritura atómica y recovery ante una línea JSONL corrupta.

### PF-M7 — Decorators y context managers como políticas

**Problema que resuelve.** Aplicar una preocupación transversal sin duplicar código ni esconder control flow crítico.

**Teoría.** Un decorator recibe y devuelve callables; debe preservar metadata y contrato. Es adecuado para observación, autorización declarativa futura o registro consistente, pero puede ocultar orden y errores. Un context manager modela una región con setup/teardown; \`contextlib\` permite expresarlo sin clases cuando el lifecycle es simple.

**Aplicación en EIDOLON.** Un decorator de métricas puede registrar duración y resultado sin payload sensible; un context manager puede crear un archivo temporal y promoverlo solo al completar una exportación.

**Cuándo no usarlo o sobredimensionarlo.** No envolver lógica de dominio, mutar argumentos o atrapar errores silenciosamente dentro de decorators mágicos.

**Profundidad matemática.** Level 0 — intuition: funciones de orden superior.

**Habilidades prácticas**

- Crear decorators con \`functools.wraps\` y comportamiento probado.

- Diseñar un context manager que limpie correctamente ante excepción.

- Explicar cuándo una función explícita es más legible que un decorator.

### PF-M8 — Async/await antes del web framework

**Problema que resuelve.** Comprender concurrencia cooperativa antes de usar endpoints async o streaming.

**Teoría.** Una coroutine progresa cuando el event loop la ejecuta y cede control en un \`await\`. Una task permite concurrencia, no paralelismo CPU automático. Cancellation forma parte del contrato; timeouts, bounded queues y structured concurrency limitan trabajo abandonado. El código bloqueante dentro del loop detiene a todas las tasks.

**Aplicación en EIDOLON.** Se simulan importaciones de archivos con latencia, límites de concurrencia, cancelación y reintentos idempotentes; no existe todavía red ni API.

**Cuándo no usarlo o sobredimensionarlo.** No convertir funciones puramente CPU-bound en async ni crear tasks huérfanas. Si la secuencia es simple, el código síncrono es preferible.

**Profundidad matemática.** Level 1 — practical formulas: throughput, latencia y límites de concurrencia.

**Habilidades prácticas**

- Trazar el estado de una coroutine y explicar dónde cede control.

- Aplicar timeout, cancellation cleanup y semaphore/queue acotada.

- Distinguir concurrencia, paralelismo, thread, process y async.

- Probar una tarea cancelada sin dejar archivos temporales ni estado parcial.

### PF-M9 — Testing, debugging, logging y revisión

**Problema que resuelve.** Sustituir intuición y prints dispersos por evidencia reproducible.

**Teoría.** Un test protege comportamiento o invariante, no implementación accidental. Unit, integration y property tests responden preguntas distintas. Arrange–Act–Assert mejora lectura; fixtures administran setup/teardown; parametrization cubre una clase de casos. Debugging parte de una reproducción mínima, hipótesis y observación. Logging debe ser estructurado y redactado. Git conserva decisiones mediante commits pequeños, branches, review, revert y ADR.

**Aplicación en EIDOLON.** La suite prueba replay, correction, Unicode, timezones, schema versions y fallos parciales. Los logs usan IDs sintéticos y nunca payloads privados.

**Cuándo no usarlo o sobredimensionarlo.** No perseguir coverage como sustituto de calidad, mockear todo ni registrar secretos para facilitar debugging.

**Profundidad matemática.** Level 1 — practical formulas: particiones de entrada, tasa de fallos y costo de regresión.

**Habilidades prácticas**

- Escribir tests positivos, negativos, de frontera y de invariantes.

- Usar debugger, traceback, logging y \`git bisect\` para localizar una regresión.

- Crear commits atómicos, una pull request autocontenida y un ADR corto.

- Revisar código buscando contrato, seguridad, tests, nombres, acoplamiento y rollback.

## PF.5 Laboratorios

Cada laboratorio se entrega como commit independiente con README breve, tests y una nota de reflexión: qué falló, cómo se observó y qué decisión cambiaría en producción.

| **ID**     | **Laboratorio**              | **Evidencia y conexión EIDOLON**                                                                            |
|------------|------------------------------|-------------------------------------------------------------------------------------------------------------|
| **PF-L01** | Diagnóstico reproducible     | Resolver una especificación pequeña, ejecutar tests iniciales y registrar brechas; baseline personal de P0. |
| **PF-L02** | Unicode y tiempo hostiles    | Normalizar solo para búsqueda, conservar original y probar emojis, acentos, DST y timestamps aware.         |
| **PF-L03** | Funciones sin estado oculto  | Refactorizar un script global a funciones puras + adaptador CLI; tests de defaults mutables y scopes.       |
| **PF-L04** | Índice de entidades          | Construir dict/set derivados, detectar duplicados y explicar costo frente a búsqueda lineal.                |
| **PF-L05** | Stream de eventos            | Generator que filtra JSONL grande sin cargarlo entero; prueba de consumo único y error tardío.              |
| **PF-L06** | Paquete instalable local     | Layout \`src/\`, entry point, pyproject.toml, venv limpio y dependencia dev separada.                       |
| **PF-L07** | Modelo Event/Claim           | Dataclasses + enums/IDs + invariantes; composición y tests de estados imposibles.                           |
| **PF-L08** | Type-check failure lab       | Introducir cinco fallos que el checker detecta y dos que solo runtime/tests pueden detectar.                |
| **PF-L09** | Taxonomía de errores         | Errores de parseo, schema, conflicto e I/O con chaining; CLI traduce sin perder traceback técnico.          |
| **PF-L10** | Exportación atómica          | Context manager con archivo temporal, rename y cleanup ante fallo inyectado.                                |
| **PF-L11** | Repositorio JSONL versionado | Append, load, replay, schema_version y quarantine de línea inválida; nunca pickle.                          |
| **PF-L12** | Decorator de observabilidad  | Duración/resultado/ID de operación con \`wraps\`; prueba que no registra payload ni altera excepciones.     |
| **PF-L13** | Coordinador async simulado   | Tasks con semaphore, timeout, cancellation y retry idempotente sobre I/O falso.                             |
| **PF-L14** | Pytest de fronteras          | Fixtures, parametrization, monkeypatch de clock/environment y property test introductorio.                  |
| **PF-L15** | Debugging con evidencia      | Reproducir una corrupción, reducir caso, usar pdb/logs y localizar el commit con bisect.                    |
| **PF-L16** | Review y rollback            | Branch, PR local, checklist, ADR y revert limpio sin reescribir historial compartido.                       |

## PF.6 Mini proyectos

### PF-MP1 — Journal CLI

**Alcance.** CLI que crea, lista y filtra eventos sintéticos. Debe separar dominio e I/O, conservar UTF-8, usar dataclasses, type hints y tests. No admite base de datos, Pydantic ni frameworks de CLI.

**Evidencia.** Comandos reproducibles, paquete instalable, cobertura de casos críticos y README con modelo de datos.

### PF-MP2 — Migrador de schema JSONL

**Alcance.** Lee v1, valida, migra a v2 y escribe atómicamente sin destruir la fuente. Una línea corrupta va a quarantine con receipt no sensible.

**Evidencia.** Golden files, failure injection, rollback y ADR sobre compatibilidad hacia atrás.

### PF-MP3 — Import coordinator

**Alcance.** Procesa varios archivos sintéticos con async/await, límite de concurrencia, timeout y cancelación. Cada trabajo posee un idempotency key.

**Evidencia.** Prueba de cancelación, ausencia de temporales huérfanos, logs redactados y comparación con versión síncrona.

## PF.7 Proyecto de integración — EIDOLON 0.0a

| **Build.** Núcleo determinista de journal y provenance operado por CLI, únicamente con Python estándar más pytest y herramientas de desarrollo. Usa datos sintéticos; no contiene chat, backend, base de datos ni AI. |
|-----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

- Modelar \`SourceRecord\`, \`Event\`, \`Claim\` y \`Correction\` como dataclasses separadas; una corrección no reescribe el evento original.

- Persistir un JSONL append-only con \`schema_version\`, IDs estables, \`recorded_at\` y \`valid_time\` timezone-aware.

- Implementar comandos \`init\`, \`append\`, \`list\`, \`show\`, \`correct\`, \`export\`, \`verify\` y \`replay\`.

- Mantener índices derivados en memoria para ID, persona, rango temporal y tags; reconstruirlos por replay.

- Escribir de forma atómica; detectar línea truncada o checksum inconsistente; generar receipt de quarantine.

- Probar Unicode, timestamps, duplicate IDs, replay, correction, schema migration, cancelación y fallos de filesystem.

- Entregar \`pyproject.toml\`, instrucciones desde un venv vacío, README, threat notes P0, ADR-001 y changelog.

- Excluir expresamente Pydantic, FastAPI, SQL/PostgreSQL, Docker, Ollama, embeddings y datos personales reales.

**Criterio de aceptación.** Un revisor clona en un environment limpio, instala, ejecuta tests, crea un journal, induce una falla, recupera por replay y explica por qué cada frontera existe. El autor puede cambiar una regla sin romper contratos no relacionados.

## PF.8 Errores comunes

- Confundir asignación con copia y usar \`is\` para comparar valores.

- Usar argumentos default mutables o variables globales para compartir estado.

- Atrapar \`Exception\`, imprimir un mensaje y continuar con estado corrupto.

- Mezclar parsing, reglas de dominio, filesystem y UI en una sola función.

- Crear herencia profunda o clases sin invariantes solo para 'usar POO'.

- Creer que type hints validan JSON externo en runtime.

- Serializar datetime de forma ambigua o mezclar naive/aware timestamps.

- Abrir archivos sin encoding/context manager o sobrescribir la fuente durante una migración.

- Usar comprehensions con side effects o generators que dependen de recursos ya cerrados.

- Añadir async a código CPU-bound o iniciar tasks que nadie cancela/espera.

- Mockear tanto que los tests ya no ejecutan el comportamiento real.

- Registrar payloads, secrets o datos privados para facilitar debugging.

- Agregar dependencias para operaciones ya cubiertas por la standard library.

- Hacer commits grandes sin intención, ADR o camino de rollback.

## PF.9 Preguntas de evaluación

1.  ¿Qué diferencia existe entre identidad, igualdad y hashability, y cómo afecta a un índice de entidades? **\[Conceptual\]**

2.  ¿Por qué un type hint no valida un payload JSON y qué capa debe hacerlo después? **\[Conceptual\]**

3.  ¿Cuándo una dataclass debe ser frozen y qué tradeoff introduce? **\[Conceptual\]**

4.  Explica LEGB y el bug de un default mutable con un ejemplo reproducible. **\[Conceptual\]**

5.  Implementa una función pura que aplique una Correction sin mutar el Event original. **\[Código\]**

6.  Diseña un iterator que lea JSONL y reporte número de línea al fallar sin cargar todo el archivo. **\[Código\]**

7.  Escribe un context manager de exportación atómica y prueba su cleanup ante excepción. **\[Código\]**

8.  Modela un repository como Protocol y demuestra dos implementaciones in-memory sin framework. **\[Código\]**

9.  Un replay produce distinto orden en dos ejecuciones: ¿qué observaciones reunirías antes de cambiar código? **\[Debugging\]**

10. Una task cancelada deja un archivo temporal: localiza el error de ownership y propón el test de regresión. **\[Debugging\]**

11. El programa funciona en tu laptop pero falla en un venv limpio: ¿cómo separas dependency, packaging e import-path failures? **\[Debugging\]**

12. ¿Qué debe pertenecer a \`domain\`, \`application\`, \`adapters\` y \`cli\` en EIDOLON 0.0a? **\[Diseño\]**

13. ¿Cuándo un decorator de logging vuelve menos seguro o menos comprensible el sistema? **\[Diseño\]**

14. Defiende JSONL para P0 y explica con precisión cuándo deja de ser apropiado. **\[Diseño\]**

15. Revisa una función de 80 líneas que parsea, valida, guarda y registra: plantea una refactorización incremental y reversible. **\[Review\]**

16. Explica async/await, thread y process a un estudiante usando un ejemplo de importación, sin analogías engañosas. **\[Enseñanza\]**

## PF.10 Criterio de dominio y checkpoints

| **Competencia** | **Evidencia obligatoria**                                           | **Umbral**                                                                 |
|-----------------|---------------------------------------------------------------------|----------------------------------------------------------------------------|
| **D1.1**        | EIDOLON 0.0a + code review de Python idiomático, tipos y errores.   | Implementa y defiende sin tutorial; ningún estado imposible silencioso.    |
| **D1.2**        | Failure lab, paquete reproducible, logs redactados y release local. | Reproduce, diagnostica y corrige tres fallas; instalación limpia aprobada. |
| **D1.3**        | Historia Git, PR, README, ADR y defensa oral.                       | Commits atómicos, rollback seguro y decisión explicable a otro ingeniero.  |
| **Gate P0**     | Suite Unicode/tiempo/serialización + replay + diagnóstico.          | 100% de tests críticos; no se compensa con coverage o promedio.            |

### CHECKPOINT PF-C1 — Código determinista

- ¿Puedes explicar mutabilidad, scopes, funciones y colecciones sin consultar documentación?

- ¿Puedes implementar el mismo filtro primero de forma clara y luego medir una alternativa?

- ¿Puedes demostrar con tests que Unicode y timezones no se corrompen?

### CHECKPOINT PF-C2 — Diseño y lifecycle

- ¿Puedes modelar Event/Claim/Correction con composición y estados válidos?

- ¿Puedes empaquetar el proyecto e instalarlo desde cero en otra ruta?

- ¿Puedes detectar cuándo NO usar inheritance, decorator, generator o async?

### CHECKPOINT PF-C3 — Defensa profesional

- ¿Puedes depurar una falla desconocida siguiendo reproducción → hipótesis → observación → fix → regression test?

- ¿Puedes revisar código ajeno, proponer un cambio incremental y revertirlo?

- ¿Puedes enseñar el diseño del journal y reconocer sus límites antes de PostgreSQL?

## PF.11 Recursos recomendados

*Ruta de lectura: documentación oficial primero; después capítulos seleccionados. No se requiere leer libros completos antes de construir.*

**Documentación oficial.** [*Python 3.14 Documentation*](https://docs.python.org/3/contents.html) — Tutorial; Data Model; exceptions; modules; dataclasses; typing; contextlib; pathlib; json; asyncio; pdb y profiling.

**Guía oficial.** [*Python Packaging User Guide — pyproject.toml*](https://packaging.python.org/en/latest/guides/writing-pyproject-toml/) — Estructura mínima, build-system, project metadata y configuración de herramientas.

**Guía oficial.** [*Install packages with pip and venv*](https://packaging.python.org/guides/installing-using-pip-and-virtual-environments/) — Crear environments aislados y reproducir instalaciones.

**Documentación.** [*pytest — Fixtures*](https://docs.pytest.org/en/stable/how-to/fixtures.html) — Setup/teardown seguro; después parametrización y monkeypatch.

**Libro.** [*Effective Python, 3rd Edition*](https://effectivepython.com/) — Seleccionar items sobre functions, comprehensions/generators, classes, concurrency, robustness y testing; omitir extensiones C en P0.

**Libro.** [*Fluent Python, 2nd Edition*](https://www.oreilly.com/library/view/fluent-python-2nd/9781492056348/) — Data model, sequences, mappings, functions, decorators, type hints, protocols y concurrency como profundización.

**Libro abierto.** [*Pro Git*](https://git-scm.com/book/en/v2) — Capítulos 1–3, 7.10 Debugging y workflows; internals solo en Deep mastery.

## PF.12 Rutas de esfuerzo

| **Ruta**              | **Horas** | **Contenido y evidencia**                                                                                                                                                                 |
|-----------------------|-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Minimum path**      | ~30 h     | PF-M1–M6 y PF-M9 esencial; L01–L11, L14–L16; MP1; EIDOLON 0.0a con gate completo. No puede omitir Unicode, tiempo, errores, tests, Git ni dependency hygiene.                             |
| **Recommended path**  | ~55 h     | Todo Minimum + PF-M7–M8; L12–L13; MP2 y MP3; type checker, property tests introductorios, cancellation y review más profundo.                                                             |
| **Deep mastery path** | ~85 h     | Todo Recommended + profiling, Protocol/generics, packaging/build local, escenarios de failure injection ampliados, lectura selectiva de Effective/Fluent Python y defensa a otro revisor. |

| **Salida del track PF.** Programming Foundations termina cuando EIDOLON 0.0a es reproducible y defendible. El siguiente track profundiza las estructuras, sistemas y redes que explican por qué esas decisiones escalan o fallan; todavía no se introduce backend, SQL ni AI. |
|-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|

**TRACK CS · D2–D3**
