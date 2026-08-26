# Programming Foundations — Revisión global acumulativa v1

**Track:** Programming Foundations  
**Alcance:** PF-M1–PF-M9, PF-L01–PF-L16, PF-MP1–PF-MP3 y EIDOLON 0.0a  
**Fuentes:** Master Prompt, Blueprint v0.2, Curriculum de Programming Foundations, estándar editorial v0.1 y módulos PF-M1–PF-M9  
**Fecha:** 2026-08-26  
**Resultado:** PASS

## Executive verdict

Programming Foundations forma una ruta acumulativa coherente desde el modelo de objetos de Python hasta entrega, diagnóstico y rollback de EIDOLON 0.0a. La secuencia enseña primero el problema, construye un modelo mental, muestra el mecanismo, reproduce fallos y pide evidencia. No es un catálogo de sintaxis ni depende de frameworks.

El gate encontró tres brechas acumulativas importantes: `SourceRecord` no aparecía en los modelos, el checksum requerido por el build se difería por completo y el CLI no enseñaba a despachar subcommands. Las tres quedaron corregidas localmente. También se corrigió una inconsistencia menor de estado en PF-M2. No quedan hallazgos CRITICAL ni IMPORTANT abiertos.

Fortalezas principales:

- source data, derived data y provenance conservan roles distintos;
- Unicode, tiempo aware, mutabilidad, errores y serialización se enseñan antes de integrarse;
- los índices in-memory se presentan como reconstruibles, nunca como source of truth persistente;
- Correction agrega evidencia y no reescribe Event;
- packaging, environment y dependencias se verifican desde una ubicación limpia;
- lifecycle síncrono y async incluyen ownership, cleanup y cancelación;
- PF-M9 consolida evidencia sin afirmar que tests o coverage prueban ausencia de bugs;
- el alcance excluye backend, database, AI y teoría formal de Computer Science.

Hallazgos del gate:

| Severidad | Encontrados | Pendientes |
|---|---:|---:|
| CRITICAL | 0 | 0 |
| IMPORTANT | 3 | 0 |
| MINOR | 1 | 0 |
| OPTIONAL | 1 | 1 |

El OPTIONAL pendiente es dividir el estudio de los módulos más densos en varias sesiones; no requiere cambiar IDs ni contenido y no bloquea el track.

## Curriculum coverage matrix

### Competencias

| Competencia | Enseñanza principal | Práctica | Lab / build | Evidencia observable |
|---|---|---|---|---|
| D1.1 — Python idiomático, tipos y errores | PF-M1–PF-M7 | predicciones, refactors, failure cases y challenges | PF-L02–PF-L12; EIDOLON 0.0a | modelos tipados, invariantes, errores específicos, round trips y review |
| D1.2 — debugging, logging, packaging y dependencies | PF-M4, PF-M6, PF-M9 | environments limpios, fallos de I/O/import, logs y diagnóstico | PF-L06, PF-L09, PF-L14–PF-L16 | instalación reproducible, failure record, logs redactados y rollback |
| D1.3 — Git, documentación, ADR y review | PF-M9, con package/README desde PF-M4 | commits, PR, ADR, bisect y revert | PF-L16; gate 0.0a | historia revisable, decisión registrada y reversión verificada |
| D2.3 — Unicode, serialización, tiempo y precisión | PF-M1, PF-M5, PF-M6, PF-M9 | Unicode hostil, aware time, Decimal y schema round trip | PF-L02, PF-L07, PF-L11, PF-L14 | tests de edge cases y source preservado |
| D3.1 — filesystem y environment | PF-M4, PF-M6, PF-M9 | venv, cwd, paths, handles, temporales y failure injection | PF-L06, PF-L09–PF-L11, PF-L15 | diagnóstico de interpreter/environment y lifecycle observable |
| D11.1 — testing unit/integration/property | PF-M9, preparado desde PF-M1–PF-M8 mediante `assert` y contratos | pytest, fixtures, parametrization, monkeypatch, Hypothesis | PF-L14–PF-L16 | suite por riesgo, regression tests y property introductoria |
| D12.3 — supply chain, soporte P0 | PF-M4 y consolidación PF-M9 | provenance, direct/transitive dependencies, typosquatting y secret hygiene | PF-L06; EIDOLON 0.0a | dependency review, environment limpio y scan acotado de archivos tracked |

D12.3 queda deliberadamente en soporte introductorio: SBOM, firmas, CI de supply chain y response a vulnerabilidades permanecen en D12. El track no presenta un secret scan limpio como garantía absoluta.

### Cobertura por módulo

| Módulo | Resultado curricular | Evidencia interna | Dictamen |
|---|---|---|---|
| PF-M1 | ejecución, objetos, mutabilidad, Unicode, precisión y tiempo | predicciones, failure cases, ingesta sintética y challenge | completa |
| PF-M2 | funciones, contratos, LEGB, pureza y effects | refactor funcional, defaults mutables y pipeline determinista | completa |
| PF-M3 | colecciones, iteración y materialización deliberada | índices derivados, iterator consumption y challenge de timeline | completa |
| PF-M4 | módulos, packages, imports, venv, pyproject y dependencias | package instalable, circular import, environment limpio y CLI mínima | completa tras corrección |
| PF-M5 | objetos pequeños, dataclasses, typing, Protocol y composition | SourceRecord/Event/Claim/Correction, checker y store in-memory | completa tras corrección |
| PF-M6 | exceptions, files, JSON/JSONL, schema y lifecycle | round trip, corruption, migration, replace seguro y checksum | completa tras corrección |
| PF-M7 | decorators/context managers como políticas | metadata preservada, exception propagation y atomic export | completa |
| PF-M8 | async/await, tasks, cancellation, timeout y backpressure | TaskGroup, Queue, bounded coordinator y cleanup | completa |
| PF-M9 | testing, debugging, logging, Git, ADR y review | suite, failure protocol, redaction, bisect, revert y challenge final | completa tras consolidación |

Las prioridades `[MUST]` están cubiertas. Generators, decorators, context managers, Protocol, property-based testing y async cubren el nivel `[SHOULD]` declarado. Los temas `[NICE]` no se convirtieron en requisitos universales.

## Dependency audit

### Grafo de prerequisites

```text
PF-M1
  ↓
PF-M2
  ↓
PF-M3
  ↓
PF-M4
  ↓
PF-M5
  ↓
PF-M6
  ↓
PF-M7
  ↓
PF-M8
  ↓
PF-M9
```

Cada módulo declara únicamente módulos anteriores. No se encontró uso-before-teaching importante después de las correcciones:

- PF-M1 introduce valores, objetos, Unicode, precisión y tiempo antes de que otros módulos los usen;
- PF-M2 introduce funciones, contracts, scopes y pureza antes de comprehensions, classes y adapters;
- PF-M3 introduce containers e iteración antes de stores, JSONL y async collections;
- PF-M4 introduce módulos, packages y environments antes de proyectos multiarchivo y dev tooling;
- PF-M5 introduce dataclasses, type hints, Enum y Protocol antes de serializarlos;
- PF-M6 introduce exceptions, files, JSON y resource lifecycle antes de custom context managers;
- PF-M7 introduce decorators y custom context managers antes de usarlos como policies integradas;
- PF-M8 introduce async lifecycle antes de sus tests en PF-M9;
- PF-M9 introduce pytest/Hypothesis/Git review como consolidación y no como prerequisite retroactivo.

`ValueError`, `with`, `assert`, main guards y APIs puntuales aparecen antes de su tratamiento profundo solo con explicación local y señalización, de acuerdo con el estándar editorial.

## Technical audit

### PF-M1

Se verificaron bindings, identity/equality, aliasing, shallow/deep copy, truthiness, Unicode/code points, UTF-8, Decimal, naive/aware datetime, `zoneinfo`, `fold`, `recorded_at` y `valid_time`. Las afirmaciones distinguen contrato de Python, dependencia de tzdata y decisiones EIDOLON.

### PF-M2

Se verificaron definición/llamada, `return`, argumentos, defaults mutables, LEGB, `global`, `nonlocal`, closures, pure functions y dependency boundaries. La corrección fue solo metadata: el archivo decía `review candidate` aunque los índices ya lo trataban como aprobado.

### PF-M3

Se verificaron list/tuple/dict/set/frozenset, indexing/slicing, aliasing anidado, hashability, iterable/iterator, `iter`, `next`, `StopIteration`, range, enumerate, zip/`strict=True`, comprehensions, generators y eager/lazy. Los costos permanecen intuitivos y remiten la formalización a CS-M1/CS-M2.

### PF-M4

Se verificaron import execution/cache, `__name__`, main guard, `sys.path`, shadowing, packages, absolute/relative imports, ciclos, `src` layout, venv, pip, pyproject, entry points y dependencies. Se añadió `argparse` mínimo para conectar el launcher con subcommands sin enseñar una arquitectura CLI ni mover domain behavior al borde.

### PF-M5

Se verificaron class/instance, identity, invariants, class vs instance attributes, methods, property, inheritance/composition, dataclasses, frozen shallow state, typing, `Any`, Enum y Protocol. Se añadió `SourceRecord` como source evidence distinta de `SourceRef` y de Event; dataclasses y type hints continúan sin presentarse como runtime validation automática.

### PF-M6

Se verificaron propagation, chaining, `try/except/else/finally`, pathlib, UTF-8, resource lifetime, JSON types, schema versions, JSONL, corruption, quarantine, migration y temporal + replace. Se añadió checksum SHA-256 sobre JSON canónico: detecta mismatch, no autentica origen ni promete resistencia ante un actor que recalcula el valor.

### PF-M7

Se verificaron higher-order functions, wrappers, decorator syntax, `functools.wraps`, parameters, order, exception propagation, sync/async contract boundaries, context manager protocol, `contextlib.contextmanager`, cleanup y nested managers. Los decorators no ocultan domain behavior ni payloads.

### PF-M8

Se verificaron coroutine objects, event loop, `await`, tasks, TaskGroup, gather, cancellation, `asyncio.timeout`, Semaphore, Queue accounting, shutdown, idempotency, async iterables y determinismo. Los ejemplos usan I/O simulado; no introducen red, threads profundos ni paralelismo CPU automático.

### PF-M9

Se verificaron pytest, AAA, positive/negative/boundary/invariant tests, `pytest.raises`, parametrization, fixtures, `tmp_path`, monkeypatch, doubles, test pyramid, Hypothesis, coverage limits, regression, traceback/pdb, logging/redaction, async tests, Git/PR/ADR, bisect y revert. El challenge final ahora prueba el contrato acumulado de EIDOLON 0.0a y no solo utilities aisladas.

## Pedagogical audit

La progresión dominante es:

```text
problema → modelo mental → mecanismo → ejemplo → fallo → corrección
→ tradeoff → EIDOLON → práctica → evidencia
```

La práctica está distribuida. PF-M3–PF-M8 usan checkpoints locales frecuentes; PF-M1/PF-M2 enlazan práctica inmediata con ejercicios guiados; PF-M9 alterna diseño de tests, diagnóstico, review y terminal antes del challenge. Los ejercicios independientes no revelan solución inmediata y los guiados explican decisiones.

Comprehensions, OOP, decorators y async se presentan como herramientas bajo un problema concreto, no como objetivos estilísticos. Los módulos declaran cuándo una función, loop explícito, código síncrono o standard library es mejor.

Carga conceptual cualitativa:

| Módulo | Carga | Nota de estudio |
|---|---|---|
| PF-M1 | heavy | separar objetos/Unicode de precisión/tiempo en sesiones |
| PF-M2 | very heavy | practicar contracts/scopes antes de effects/closures |
| PF-M3 | very heavy | dividir collections, iteration y lazy processing |
| PF-M4 | very heavy | separar imports/layout de environments/dependencies |
| PF-M5 | very heavy | separar object model, dataclasses y static typing |
| PF-M6 | very heavy | separar exceptions/resources de serialization/persistence |
| PF-M7 | heavy | estudiar decorators y managers como dos políticas relacionadas |
| PF-M8 | heavy | avanzar desde scheduling hacia ownership/backpressure |
| PF-M9 | very heavy | ejecutar por bloques: tests, debugging/logging y Git/review |

La densidad es consistente con el objetivo profesional y con las rutas de esfuerzo. Se recomienda dividir sesiones, no fragmentar IDs ni eliminar failure cases útiles.

## Cross-module consistency

Terminología consistente:

- source data / source of truth designan evidencia bajo contrato, no una proyección conveniente;
- derived data incluye normalizaciones, índices, migraciones y resúmenes reconstruibles;
- provenance se conserva y no se inventa durante transformaciones;
- Event, Claim, Correction, SourceRef y SourceRecord mantienen responsabilidades distintas;
- invariant, contract, scope, side effect, schema, lifecycle, ownership, cancellation, replay, deterministic, regression y evidence conservan una acepción estable;
- task se usa para `asyncio.Task` cuando corresponde y no como sinónimo ambiguo de process/thread.

La repetición transversal es reinforcement, no redundancia: mutabilidad reaparece en collections/dataclasses; determinismo en funciones/replay/async/tests; source preservation en modelado/JSONL/export/review. Cada reaparición agrega un failure mode o una decisión nueva.

Se validaron jerarquía principal, fences, tablas, archivos relativos y anchors locales. No quedan links locales rotos ni referencias a módulos inexistentes. El estándar editorial v0.1 sigue siendo suficiente; no se justifica crear v0.2.

## EIDOLON integration audit

### Labs PF-L01–PF-L16

| Lab | Módulos que preparan | Conocimiento y práctica previa | Prerequisite oculto |
|---|---|---|---|
| PF-L01 | PF-M1, PF-M9 | ejecución reproducible, asserts/tests y registro de brechas | ninguno |
| PF-L02 | PF-M1, PF-M9 | Unicode, normalización no destructiva, aware time y edge tests | tzdata ambiental declarada |
| PF-L03 | PF-M2, PF-M4, PF-M9 | pure functions, CLI boundary, package y tests | ninguno |
| PF-L04 | PF-M3, PF-M9 | dict/set derivados, duplicates y costo intuitivo | ninguno |
| PF-L05 | PF-M3, PF-M6, PF-M9 | generator single-pass, JSONL line iteration y error tardío | ninguno |
| PF-L06 | PF-M4, PF-M9 | src layout, pyproject, venv, entry point y clean install | build requirements disponibles según environment |
| PF-L07 | PF-M5, PF-M9 | dataclasses, IDs, Enum, invariants y impossible states | ninguno |
| PF-L08 | PF-M5, PF-M9 | checker vs runtime/domain validity | checker declarado como dev dependency |
| PF-L09 | PF-M6, PF-M9 | taxonomía, chaining, CLI translation y traceback | ninguno |
| PF-L10 | PF-M6, PF-M7, PF-M9 | temporal/replace, custom manager y failure injection | garantías de OS declaradas |
| PF-L11 | PF-M5, PF-M6, PF-M9 | SourceRecord/Event/Claim/Correction, JSONL, schema, checksum y quarantine | ninguno |
| PF-L12 | PF-M7, PF-M9 | wraps, duration/result/ID, exception transparency y redaction | ninguno |
| PF-L13 | PF-M8, PF-M9 | TaskGroup, semaphore, timeout, cancellation, retry e idempotency | ninguno |
| PF-L14 | PF-M1, PF-M5, PF-M6, PF-M8, PF-M9 | boundaries Unicode/time/files/async, fixtures y property | pytest/Hypothesis dev dependencies |
| PF-L15 | PF-M6, PF-M9 | corruption, minimal reproduction, pdb/logs y bisect | historia Git preparada por el lab |
| PF-L16 | PF-M4, PF-M9 | branch, PR, ADR, review, revert y clean install | Git local; remoto no obligatorio |

Las precondiciones ambientales están declaradas y no son conceptos ocultos. Ningún lab exige PF-M5+ antes de su enseñanza ni tecnologías fuera del track.

### Mini projects

| Mini project | Preparación suficiente | Evidencia de resolubilidad |
|---|---|---|
| PF-MP1 — Journal CLI | PF-M1–PF-M6 + PF-M9 | package/entry point, subcommands, dataclasses, JSONL, UTF-8 y tests |
| PF-MP2 — Migrador de schema JSONL | PF-M5–PF-M7 + PF-M9 | pure migration, source preservation, atomic export, failure injection y ADR |
| PF-MP3 — Import Coordinator | PF-M6–PF-M9 | synthetic I/O, bounded concurrency, timeout, cancellation, redacted logs y sync comparison |

### EIDOLON 0.0a

El conocimiento acumulado permite construir:

| Contrato del build | Preparación |
|---|---|
| SourceRecord/Event/Claim/Correction separados | PF-M5 |
| SourceRef, IDs e invariantes | PF-M5 |
| CLI instalable y ocho subcommands | PF-M4 + challenge PF-M9 |
| JSONL append-only y schema version | PF-M6 |
| recorded_at/valid_time aware | PF-M1, PF-M5, PF-M6 |
| checksum inconsistente y línea truncada | PF-M6 |
| Correction sin reescritura | PF-M1, PF-M5, PF-M6 |
| índices por ID/persona/tiempo/tags reconstruidos por replay | PF-M3, PF-M6, PF-M9 |
| temporal + validation + replace | PF-M6, PF-M7 |
| quarantine/error reporting | PF-M6, PF-M9 |
| tests, logging redactado y failure evidence | PF-M9 |
| package, README, ADR, changelog e historia Git | PF-M4, PF-M9 |
| threat notes y dependency/secret hygiene P0 | PF-M4, PF-M9 |

Una implementación temporal mínima de compatibilidad ejercitó los cuatro modelos, UTF-8, aware time, checksum, append/load, índices derivados, atomic export, los ocho nombres de subcommand, coordinación async acotada y logging sin payload. Terminó con `PROGRAMMING FOUNDATIONS integrated reference: PASS`. No se incorporó esa aplicación temporal como contenido canónico.

## Code validation

Environment de validación:

- Python 3.14.7;
- pytest 9.1.1;
- Hypothesis 6.165.10;
- standard library para la implementación integrada.

Resultados:

| Comprobación | Resultado |
|---|---|
| Bloques Python PF-M1–PF-M9 | 699/699 compilan sintácticamente |
| Bloques TOML | 8/8 parsean |
| Candidatos autónomos PF-M1–PF-M4 | 188 clasificados; 176 ejecutan aislados |
| Bloques PF-M4 dependientes de árbol/package | 12 clasificados como contextuales, no como fallos autónomos |
| Ejemplos seleccionados PF-M5 | 45 ejecutados; 0 fallos |
| Failure cases seleccionados PF-M5 | 4/4 producen la categoría esperada |
| Integración y challenge PF-M6 | PASS |
| Integración y challenge PF-M7 | PASS |
| Ejemplos seleccionados PF-M8 | 33 ejecutados; 0 fallos |
| Challenge PF-M8 | PASS |
| Suite de referencia PF-M9 | 15 tests PASS |
| Bisect/revert sintético PF-M9 | first bad commit localizado; suite restaurada |
| Ejemplos nuevos `argparse`/SourceRecord/checksum | outputs exactos verificados |
| Referencia integrada acumulativa | PASS |
| Fences, Markdown y links locales | sin defectos bloqueantes |

Los bloques contextuales de PF-M4 representan archivos de un árbol o una secuencia declarada; ejecutarlos como `python -c` sin crear ese árbol produce el `ModuleNotFoundError` que corresponde a la ausencia deliberada de contexto. No se contabilizaron como ejemplos autónomos fallidos.

Los outputs dependientes de paths, clocks, scheduler, mensajes de excepción o platform se expresan mediante propiedades estables o asserts, no mediante texto literal frágil.

## Changes applied

| Archivo | Cambio | Razón |
|---|---|---|
| PF-M2 | estado `approved` | corregir inconsistencia con índice y aprobación previa |
| PF-M4 | `argparse` mínimo, práctica y checklist | cubrir dispatcher de CLI requerido para EIDOLON 0.0a |
| PF-M5 | SourceRecord, diagrama, challenge y checklist | separar source evidence de SourceRef/Event |
| PF-M6 | checksum canónico, failure case y challenge ampliado | detectar checksum inconsistente sin invadir security profunda |
| PF-M7 | estado `approved` | resultado del gate acumulativo |
| PF-M8 | estado `approved` | resultado del gate acumulativo |
| PF-M9 | contrato 0.0a, tests, docs y secret hygiene P0 | demostrar integración y soporte D12.3 |
| README del Study Guide | PF-M1–PF-M9 `approved` | navegación y estado acumulativo |
| README del track | PF-M1–PF-M9 `approved` + enlace a este reporte | índice canónico mínimo |
| CHANGELOG | entrada v0.3.4 | registrar gate, correcciones y autorización del siguiente track |

No se cambió Master Prompt, Blueprint ni Curriculum. No se creó `STUDY_GUIDE_EDITORIAL_STANDARD_v0.2.md`.

## Hallazgos

### IMPORTANT I-01 — SourceRecord ausente

**Ubicación:** PF-M5, PF-M6 y challenge PF-M9.  
**Problema:** el build canónico exigía SourceRecord, pero el track solo modelaba SourceRef/Event/Claim/Correction.  
**Impacto:** el estudiante podía colapsar evidencia recibida con una interpretación derivada.  
**Corrección:** SourceRecord se añadió como dataclass separada, se incorporó al round trip/challenges y se reforzó la distinción source/derived.

### IMPORTANT I-02 — Checksum exigido pero no enseñado

**Ubicación:** PF-M6 §17, caso integrado y challenge.  
**Problema:** PF-M6 declaraba explícitamente que no calculaba checksums, mientras EIDOLON 0.0a exige detectar inconsistencia.  
**Impacto:** el build no podía satisfacer un criterio canónico solo con PF-M1–PF-M9.  
**Corrección:** checksum SHA-256 sobre JSON canónico sin el field recursivo; failure case, line evidence y límites de autenticidad explícitos.

### IMPORTANT I-03 — Integración CLI/build insuficientemente demostrable

**Ubicación:** PF-M4 §13 y PF-M9 challenge.  
**Problema:** el console entry point no enseñaba argumentos/subcommands y el challenge final podía aprobar sin demostrar el contrato completo 0.0a ni hygiene P0.  
**Impacto:** PF-MP1 y el build podían depender de conocimiento no enseñado o entregar solo utilities.  
**Corrección:** dispatcher `argparse` mínimo; contrato acumulado con ocho subcommands, modelos, replay, checksum, documentación y evidencia acotada de dependency/secret hygiene.

### MINOR M-01 — Estado incoherente de PF-M2

**Ubicación:** metadata de PF-M2.  
**Problema:** el archivo decía `review candidate` mientras README y continuidad lo trataban como aprobado.  
**Impacto:** navegación y prerequisite status ambiguos.  
**Corrección:** estado alineado a `approved`.

### OPTIONAL O-01 — Densidad de estudio

**Ubicación:** especialmente PF-M2–PF-M6 y PF-M9.  
**Problema:** varios módulos son conceptualmente muy densos.  
**Impacto:** estudiar el archivo completo en una sesión puede aumentar carga cognitiva.  
**Recomendación:** usar sus secciones principales como sesiones; no dividir IDs ni retirar práctica útil.

## Remaining risks

Quedan conscientemente para tracks posteriores:

- complejidad formal, estructuras, crash consistency, OS internals y networking;
- locks y writers concurrentes sobre persistencia;
- SQL/PostgreSQL, transactions y durable source of truth;
- HTTP, FastAPI, Pydantic y frontend;
- autenticación criptográfica, key management, SBOM/firmas y assurance de supply chain;
- privacy operations profundas, datos reales y threat modeling completo;
- AI, LLMs, embeddings, RAG y agents.

JSONL continúa siendo una persistencia educativa P0, no una database definitiva. El checksum detecta mismatch bajo el formato enseñado, no demuestra provenance criptográfica. Async continúa en Working Knowledge y con I/O sintético.

## Checkpoints finales

### PF-C1 — Código determinista

Soportado por PF-M1–PF-M3 y reforzado en PF-M9: el estudiante predice mutabilidad/scopes/colecciones, conserva Unicode/time y produce filtros/resúmenes deterministas.

### PF-C2 — Diseño y lifecycle

Soportado por PF-M4–PF-M8: modela SourceRecord/Event/Claim/Correction, instala el package, maneja errores/resources, aplica decorators/context managers con criterio y explica ownership/cancellation async.

### PF-C3 — Defensa profesional

Soportado por PF-M9: reproduce, formula hipótesis, observa con debugger/logs, agrega regression test, revisa diff, documenta PR/ADR, localiza con bisect y revierte sin reescribir historia compartida.

No quedan CRITICAL ni IMPORTANT pendientes. PF-L01–PF-L16 están preparados, PF-MP1–PF-MP3 son resolubles y EIDOLON 0.0a es construible con el conocimiento enseñado.

PROGRAMMING FOUNDATIONS GLOBAL GATE: PASS
