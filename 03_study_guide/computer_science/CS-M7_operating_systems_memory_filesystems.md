# CS-M7 — Sistemas operativos, memoria y filesystems

**Track:** Computer Science Foundations  
**Competencias:** D3.1; soporte D2.1  
**Fase:** P0  
**Nivel objetivo:** Aplicado-profesional  
**Prerequisites:** PF-M1–PF-M9, CS-M1–CS-M6  
**Build:** EIDOLON 0.0b  
**Curriculum source:** [CS-M7](../../02_curriculum/02_computer_science_foundations.md#cs-m7--fundamentos-de-sistemas-operativos-memoria-y-filesystem)  
**Status:** review candidate

Una línea cotidiana parece sencilla:

```python
with open("journal.jsonl", "a", encoding="utf-8") as file:
    file.write(record)
```

Pero ¿quién crea el archivo?, ¿dónde están los bytes antes de llegar al almacenamiento?, ¿qué significa que el archivo quedó cerrado?, ¿qué sobrevive si muere el proceso?, ¿y si falla el sistema operativo o se corta la energía?

CS-M7 estudia esas fronteras con una pregunta central:

> ¿Qué garantías ofrece realmente el sistema y cuáles estamos imaginando?

```text
programa Python
↓
runtime
↓
operating system
↓
memoria / files / recursos
↓
failure model
↓
evidencia
↓
diseño seguro bajo un contrato explícito
```

No aprenderás a implementar un kernel. Aprenderás a no prometer más de lo que demuestran Python, el OS, el filesystem y el hardware bajo un failure model declarado.

## Resultados de aprendizaje

Al terminar podrás:

- distinguir programa, proceso, runtime, user space y kernel;
- explicar address space y virtual memory sin reducirlos a swap;
- usar stack y heap como modelos limitados para Python;
- predecir object lifetime a partir de referencias sin atribuir a `del` garantías inexistentes;
- distinguir garbage collection de lifecycle de recursos externos;
- provocar, medir y corregir retention lógica acotada;
- diferenciar tamaño superficial, allocations Python y RSS;
- distinguir path, directory entry, file object y file descriptor/handle;
- explicar las capas entre `write()` y almacenamiento estable;
- diferenciar `flush`, `fsync`, `close`, atomicity y durability;
- implementar un update con temporal, validación y replace dentro del mismo filesystem;
- declarar los límites portables de rename y directory `fsync`;
- diagnosticar truncamiento, descriptor leaks y fallos de permisos;
- preservar source data y reconstruir derived data corrupta;
- diseñar una inyección de fallo y recoger evidencia sin simular garantías físicas;
- explicar cómo page cache altera un benchmark de I/O;
- clasificar afirmaciones como Python, POSIX/Unix-like, Linux-specific o dependientes de plataforma.

## Cómo estudiar este módulo

1. Antes de ejecutar, escribe el failure model: process crash, OS crash o power loss.
2. Para cada operación, separa “retornó al programa” de “es durable”.
3. Dibuja ownership y lifecycle de cada recurso abierto.
4. Mide una métrica concreta; nunca llames “memoria” a números de capas distintas.
5. Conserva source data durante todo experimento destructivo.
6. Ejecuta los failure cases solo dentro de `TemporaryDirectory`.
7. Etiqueta cada observación dependiente del entorno.

### Convenciones

- **Ejemplo ejecutable:** autónomo, seguro y basado en standard library.
- **Failure case ejecutable:** reproduce un fallo acotado y declara qué evidencia conservar.
- **Fragmento:** requiere contexto omitido; no se ofrece como programa completo.
- **Comando Linux:** observa el entorno principal del track; no define semántica portable.
- **Modelo educativo:** reduce el sistema para razonar; no promete crash safety universal.

Baseline: Python 3.14 y standard library. Los timings, PID, paths, RSS y mensajes exactos dependen del entorno; los ejemplos comprueban propiedades estables.

---

## 1. Del programa a los recursos del sistema

Un **programa** es código y datos disponibles para ejecutarse. Un **proceso** es una instancia en ejecución, con estado y recursos asignados. Dos ejecuciones del mismo archivo son procesos distintos y normalmente tienen PID, address space y descriptores propios.

```text
process
├── código + Python runtime
├── address space virtual
├── execution state
├── environment + current working directory
└── recursos abiertos: files, pipes, ...
```

Un virtual environment selecciona intérprete y paquetes; **no** crea aislamiento de memoria, permisos o filesystem comparable al de un proceso o sandbox.

**Ejemplo ejecutable:**

```python
import os
import sys

pid = os.getpid()
print(pid > 0)
print(sys.executable != "")
print(os.getcwd() != "")
```

Output estable:

```text
True
True
True
```

El PID, ejecutable y working directory son evidencia de una ejecución concreta; no deben fijarse como output literal.

### Predice

Si abres dos terminales y ejecutas el mismo script al mismo tiempo, ¿comparten PID? ¿Comparten necesariamente current working directory?

### Comprueba en terminal

**Linux:**

```bash
python -c "import os, sys; print(os.getpid(), sys.executable, os.getcwd())"
ps -p <PID> -o pid,ppid,stat,rss,command
```

`ps` responde qué proceso existe y muestra una instantánea. Reemplaza `<PID>`; no copies los corchetes angulares.

## 2. Operating system, user space, kernel y system calls

El operating system administra procesos, memoria, filesystems, dispositivos, scheduling, permisos y aislamiento de recursos. El programa Python normalmente corre en **user space**: no manipula libremente el hardware. Pide operaciones al **kernel** a través de interfaces que terminan usando system calls.

```text
application / Python runtime       user space
               ↓
──────────── frontera del kernel ────────────
               ↓
processes / virtual memory / filesystem / devices
```

`open`, `read`, `write` y `close` son buenos nombres conceptuales para servicios del OS, pero una llamada Python no siempre corresponde uno-a-uno con una system call: el runtime puede validar, convertir, dividir, agrupar o satisfacer trabajo desde buffers.

Esto importa porque “mi función retornó” solo demuestra que terminó una capa. No demuestra automáticamente que un dispositivo hizo durable el dato.

### Distingue

Clasifica: interpretar bytecode, comprobar permisos de un path, mantener un buffer de texto de Python, mapear páginas virtuales. ¿Qué parte pertenece principalmente al runtime y cuál requiere servicios del OS?

## 3. Address space y virtual memory

Cada proceso observa un **address space**: un conjunto de direcciones virtuales que el OS y el hardware mapean a recursos subyacentes. La virtual memory aporta abstracción, aislamiento entre procesos y administración flexible de RAM.

```text
dirección observada por el proceso
↓
mapping administrado por OS + hardware
↓
RAM y, según política, otros backing mechanisms
```

Virtual memory **no significa swap**. Swap puede ser un mecanismo de respaldo, pero el modelo incluye mappings, permisos y aislamiento aunque no se use swap. Tampoco debe usarse este modelo para deducir qué significa `id()` en todas las implementaciones de Python.

Un acceso puede fallar o provocar que el OS cargue una página. CS-M10 profundizará CPU, caches y arquitectura; aquí solo necesitamos entender que dirección virtual, RAM física y storage no son sinónimos.

### Explica

¿Por qué dos procesos pueden observar el mismo valor numérico como dirección virtual sin que por ello compartan el mismo objeto?

## 4. Stack, heap, allocations y objetos Python

El **call stack** sirve como modelo de function calls, frames locales y flujo de retorno. El **heap** o memoria dinámica sirve como modelo de objetos cuya vida no coincide necesariamente con una sola llamada.

```text
call function → frame activo → return
create object → references lo mantienen alcanzable → lifetime termina más tarde
```

Python abstrae estos detalles. Decir “los objetos Python viven en el heap” es una simplificación útil, no un mapa completo de toda la memoria del runtime. El allocator de Python pide y reutiliza memoria, y el OS eventualmente respalda páginas; no hay una system call por cada objeto pequeño.

PF-M1 ya estableció el modelo esencial:

```text
name → reference → object
```

Crear una lista o `bytearray` implica allocations. Reasignar un nombre cambia una referencia; no necesariamente mueve ni destruye el objeto anterior.

**Ejemplo ejecutable:**

```python
events = [{"event_id": "evt-001"}]
alias = events
del events

print(alias[0]["event_id"])
```

Output:

```text
evt-001
```

`del events` elimina ese binding. El objeto sigue alcanzable mediante `alias`. Python no garantiza que `del` devuelva memoria al OS inmediatamente, incluso cuando desaparezca la última referencia.

### Predice

Añade `second_alias = alias` y luego `alias.clear()`. ¿Qué observa `second_alias` y por qué?

## 5. Object lifetime, garbage collection y resource lifecycle

Garbage collection recupera memoria de objetos que el runtime determina que ya no necesita. Eso no sustituye el cierre determinista de recursos externos.

```text
object memory lifecycle        external resource lifecycle
reachable / unreachable       open / usable / closed
GC policy                     ownership + close protocol
```

Un file object puede envolver un descriptor del OS. Esperar a que el objeto sea recolectado para cerrar el descriptor hace que el momento de cleanup dependa de implementación y circunstancias. La regla profesional ya presentada en PF-M6/PF-M7 sigue vigente: define ownership y usa `with` o `try/finally`.

**Failure case — código incorrecto:**

```python
file = open("journal.jsonl", encoding="utf-8")
first_line = file.readline()
del file  # no es un protocolo portable de cierre determinista
```

La corrección es:

```python
with open("journal.jsonl", encoding="utf-8") as file:
    first_line = file.readline()
```

Si mencionamos CPython: usa reference counting y un cyclic collector, pero esa es una estrategia de implementación, no una licencia para depender de destrucción inmediata en código portable.

### Distingue

¿Cuál requiere cleanup explícito: una lista temporal, un file object abierto, un lock futuro, una string? Distingue memoria del objeto y recurso externo.

## 6. Retention y memory leaks lógicos

Un lenguaje con GC puede sufrir crecimiento no deseado. Si una cache, collection global, closure o registry conserva referencias, los objetos siguen alcanzables; el collector no debe eliminarlos.

**Ejemplo ejecutable y acotado:**

```python
retained: list[bytearray] = []


def collect_batch(count: int) -> None:
    retained.extend(bytearray(1024) for _ in range(count))


collect_batch(64)
print(len(retained))
retained.clear()
print(len(retained))
```

Output:

```text
64
0
```

Esto demuestra retention por referencias y cleanup lógico; no demuestra cuánto ni cuándo baja RSS.

En EIDOLON, `events_by_id` puede crecer porque representa todos los eventos necesarios: crecimiento legítimo. Se vuelve un leak lógico si conserva entradas fuera del contrato, no admite rebuild/eviction cuando debería o mantiene objetos abandonados por accidente. La política de retención se decide antes de llamar “leak” a cualquier crecimiento.

### Diagnóstico

Un índice crece 1 MiB por cada 1 000 eventos y el dataset también crece. ¿Qué evidencia necesitas antes de afirmar que existe un leak?

## 7. Medir memoria: shallow size, tracemalloc y RSS

No existe una única cifra llamada “la memoria del objeto”:

| Medida | Qué aproxima | Qué no demuestra |
|---|---|---|
| `sys.getsizeof(x)` | tamaño superficial de `x` en esa implementación | contenido recursivo ni memoria total del proceso |
| `tracemalloc` | allocations Python rastreadas desde que inicia | toda RSS, mappings o memoria nativa no rastreada |
| RSS del OS | páginas residentes atribuidas al proceso, con matices de plataforma/compartición | ownership exacto por objeto Python |

Como modelo de Level 1:

```text
working set observado
≈ objetos retenidos + runtime + buffers nativos + páginas mapeadas residentes

bytes rastreados por item
≈ (traced_after - traced_before) / cantidad_de_items
```

Son estimaciones dependientes del experimento, no identidades universales. Un working set que excede la RAM disponible puede aumentar presión e I/O, pero el comportamiento depende del OS y workload.

**Ejemplo ejecutable:**

```python
import tracemalloc

tracemalloc.start()
before, _ = tracemalloc.get_traced_memory()
retained = [bytearray(1024) for _ in range(128)]
after, peak = tracemalloc.get_traced_memory()

assert after >= before
assert peak >= after
assert len(retained) == 128
tracemalloc.stop()
print("measurement properties: PASS")
```

Output:

```text
measurement properties: PASS
```

**Failure case:** sumar `getsizeof(list)` y llamarlo “RAM total” ignora objetos referenciados, allocator, runtime, shared pages y memoria nativa.

### Comprueba en terminal

**Linux:** `ps -p <PID> -o pid,rss,vsz,command` observa RSS/VSZ en una instantánea. `top -p <PID>` observa cambios. Las unidades y columnas dependen de la herramienta; documenta versión y comando.

## 8. Filesystem: path, directory entry, metadata y contenido

Un filesystem relaciona nombres con objetos y contenido persistente. Un modelo portable y deliberadamente abstracto es:

```text
path
↓ resolución de componentes
directory entries
↓
objeto del filesystem + metadata
↓
contenido
```

Un **path** es una ruta para localizar; no es el contenido ni un file object abierto. Renombrar una entry dentro del mismo filesystem normalmente cambia namespace/metadata, no copia todos los bytes, pero la garantía exacta depende del OS y filesystem.

En filesystems Unix-like, un **inode** modela metadata y referencias al contenido; el filename reside conceptualmente en una directory entry. Inode no es concepto universal ni debe proyectarse literalmente sobre Windows u otros filesystems.

**Ejemplo ejecutable:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory) / "event.txt"
    path.write_text("evt-001", encoding="utf-8")
    renamed = path.with_name("renamed.txt")
    path.replace(renamed)

    assert not path.exists()
    assert renamed.read_text(encoding="utf-8") == "evt-001"

print("path/content distinction: PASS")
```

El ejemplo demuestra el resultado visible en este entorno; no demuestra durability frente a power loss.

### Filesystem

Después de abrir un file y renombrar su path desde otro lugar, ¿el file object abierto se convierte en un string nuevo? Explica por qué path y recurso abierto son capas distintas.

## 9. File object, descriptor y handle

```text
Python file object
↓ runtime buffering/encoding
OS file descriptor (Unix/POSIX) o handle (Windows)
↓
open file resource
```

En Unix-like, un file descriptor es un entero pequeño perteneciente al proceso. Windows usa un modelo de handles distinto; Python ofrece una API de alto nivel común, pero no borra todas las diferencias.

**Ejemplo ejecutable:**

```python
from tempfile import TemporaryFile

with TemporaryFile(mode="w+b") as file:
    descriptor = file.fileno()
    assert isinstance(descriptor, int)
    assert not file.closed

assert file.closed
print("resource lifecycle: PASS")
```

Output:

```text
resource lifecycle: PASS
```

`open()` puede fallar por path inexistente, parent ausente, permisos, límite de recursos o problemas del filesystem. Conserva operation, path, mode, excepción y entorno relevante; no conviertas automáticamente cada fallo en “archivo corrupto”.

## 10. Abrir, leer, escribir y cerrar

Una lectura puede resolverse desde page cache sin acceso físico inmediato al dispositivo. Una escritura de alto nivel puede atravesar varias capas:

```text
str / bytes de la aplicación
↓ Python text encoding + user-space buffer
↓ write request al OS
↓ kernel page cache / filesystem
↓ device cache / storage, según sistema
```

`file.write(text)` informa progreso en la capa de Python. No significa por sí mismo “bytes físicamente durables”. `close()` libera el recurso de alto nivel y normalmente entrega sus buffers pendientes al OS, pero no equivale universalmente a `fsync()` ni a “durable forever”.

Los high-level buffered file objects manejan muchos detalles. En APIs low-level como `os.write`, una llamada puede escribir menos bytes de los solicitados; el caller debe atender el count y repetir. No extrapoles un ejemplo low-level a todas las escrituras de texto.

### Predice

Si `file.write()` retorna y luego el proceso termina normalmente, ¿qué capas sabemos que aceptaron el dato? ¿Qué failure model falta para afirmar durability?

## 11. Buffering, page cache y `flush()`

Hay, conceptualmente, al menos tres capas posibles:

1. buffering de Python/user space;
2. page cache y buffering del kernel;
3. caches del dispositivo o infraestructura de almacenamiento.

`file.flush()` entrega buffers de Python al OS según el contrato del file object. **No garantiza por sí solo almacenamiento durable.** Puede mejorar visibilidad para otros readers que consultan al OS, pero visibility y durability siguen siendo distintas.

**Ejemplo ejecutable:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory) / "buffered.txt"
    with path.open("w", encoding="utf-8") as file:
        file.write("record\n")
        file.flush()
        assert path.read_text(encoding="utf-8") == "record\n"

print("flush visibility: PASS")
```

El assert observa que otro file object pudo leer el dato en este proceso/OS. No prueba supervivencia a power loss.

### Durability

Corrige esta frase: “Llamé `flush()`, por lo tanto el journal quedó durable”.

## 12. `fsync()`, `close()` y límites de la evidencia

Después de `flush()`, `os.fsync(file.fileno())` solicita al OS sincronizar datos relevantes del file con almacenamiento conforme a las garantías del sistema.

**Ejemplo ejecutable:**

```python
import os
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory) / "synced.txt"
    with path.open("w", encoding="utf-8") as file:
        file.write("evt-001\n")
        file.flush()
        os.fsync(file.fileno())

    assert path.read_text(encoding="utf-8") == "evt-001\n"

print("fsync request: PASS")
```

El ejemplo demuestra que la solicitud no falló y el contenido es visible. No simula pérdida de energía ni verifica controladores, caches con políticas incorrectas, filesystem remoto o fallo físico. La formulación rigurosa es: `fsync` fortalece la política de persistencia bajo garantías documentadas del OS/filesystem/storage; no ofrece inmunidad absoluta.

`close()` también puede fallar al finalizar una escritura. Un `with` asegura que se intente cerrar incluso ante excepción, pero no convierte ese cierre en una transacción durable.

### Explica

Ordena de menor a mayor intención de persistencia: `write`, `flush`, `fsync`. ¿Por qué ese orden no constituye una prueba de supervivencia ante cualquier fallo?

## 13. Atomicity, durability y crash consistency

**Atomicity** responde si una operación observable aparece completa o no aparece, bajo condiciones específicas. **Durability** responde si un resultado considerado committed sobrevive a fallos incluidos en el contrato. **Crash consistency** describe qué estados recuperables pueden quedar tras una interrupción inesperada.

| Pregunta | Concepto |
|---|---|
| ¿Un reader ve el archivo anterior o el nuevo, pero no una mezcla? | atomicity/visibility |
| ¿El archivo nuevo sobrevive al failure model declarado? | durability |
| ¿Qué combinaciones de datos y metadata pueden observarse tras el crash? | crash consistency |

Un rename atómico puede mejorar visibility y aun no ser durable si la actualización de la directory entry no quedó persistida. “Atómico” no significa “indestructible”.

### Failure model

- **Process crash:** el proceso deja de ejecutar; OS y otros procesos siguen vivos.
- **OS crash:** kernel/filesystem dejan de operar normalmente.
- **Power loss:** energía desaparece; buffers no durables pueden perderse.

Una excepción inyectada en Python modela parte de un process failure. No reproduce fielmente OS crash ni power loss.

## 14. Temporary file, validación y atomic replace

Para actualizar derived data sin exponer una escritura parcial:

```text
read immutable source
↓ compute derived output
write + flush + optional fsync(file)
↓ close and validate temporary
↓ replace target
optional directory sync where supported/required
```

Crear el temporal en el directorio del target aumenta la probabilidad de que ambos estén en el mismo filesystem. `os.replace`/`Path.replace` usa la operación del OS; las garantías exactas dependen de plataforma y filesystem.

**Ejemplo ejecutable:**

```python
import os
from pathlib import Path
from tempfile import mkstemp, TemporaryDirectory


def safe_replace_text(target: Path, text: str) -> None:
    descriptor, temporary_name = mkstemp(
        dir=target.parent,
        prefix=f".{target.name}.",
        suffix=".tmp",
    )
    temporary = Path(temporary_name)
    try:
        with os.fdopen(descriptor, "w", encoding="utf-8") as file:
            file.write(text)
            file.flush()
            os.fsync(file.fileno())

        if temporary.read_text(encoding="utf-8") != text:
            raise ValueError("temporary validation failed")
        os.replace(temporary, target)
    finally:
        temporary.unlink(missing_ok=True)


with TemporaryDirectory() as directory:
    target = Path(directory) / "timeline.txt"
    target.write_text("old\n", encoding="utf-8")
    safe_replace_text(target, "new\n")
    assert target.read_text(encoding="utf-8") == "new\n"

print("safe replace: PASS")
```

Output:

```text
safe replace: PASS
```

Este patrón:

- evita modificar el target hasta después de validar;
- cierra el temporal antes de replace, importante también para compatibilidad con Windows;
- solicita `fsync` del file;
- limpia un temporal sobrante ante fallo.

No garantiza:

- atomicidad cross-filesystem/cross-volume;
- persistencia de la directory entry después de power loss;
- comportamiento uniforme en filesystems remotos o no convencionales;
- recuperación ante todo fallo de hardware.

### Modifica

Agrega un parámetro `fail_before_replace`. Lanza `RuntimeError` después de validar y comprueba que el target anterior sigue intacto.

## 15. Directory `fsync` y portabilidad

En algunos Unix-like systems, una política más fuerte sincroniza el directorio después del rename para pedir persistencia de la directory entry. Es una operación POSIX/Unix-specific con variaciones de soporte; no es una receta cross-platform universal.

**Fragmento Unix-like, no portable:**

```python
directory_fd = os.open(target.parent, os.O_RDONLY)
try:
    os.fsync(directory_fd)
finally:
    os.close(directory_fd)
```

En Windows el modelo de handles, sharing y flush difiere. La API portable del módulo termina en “documenta y prueba la garantía requerida en plataformas soportadas”; no intenta fabricar durabilidad perfecta con un `if os.name` improvisado.

Mover entre filesystems o volumes puede fallar o convertirse, en herramientas de alto nivel, en copy + delete. Si necesitas atomic replace, verifica que temporal y target pertenezcan al mismo filesystem y usa una operación cuyo contrato lo documente.

### Portability

Clasifica el ejemplo anterior: ¿semántica Python, POSIX/Unix-like o Linux-specific? ¿Qué debes investigar antes de prometer la misma garantía en Windows?

## 16. Partial writes, truncamiento y append

Una escritura directa sobre el target puede dejar un archivo truncado o parcialmente actualizado si ocurre un fallo. En low-level I/O, una write puede completar menos bytes y devolver el count. En high-level buffered I/O, Python normalmente gestiona repeticiones, pero siguen existiendo excepciones, fallos durante close y crashes entre records.

Append es útil para un journal porque evita reescribir todo el archivo, pero no equivale a:

- transaction;
- multi-process coordination;
- record durability;
- ausencia universal de interleaving;
- validación de que cada record está completo.

CS-M8 tratará coordinación y concurrencia. Aquí usamos un solo writer y failure injection controlada.

**Failure case ejecutable — truncamiento simulado:**

```python
import json

complete = json.dumps({"event_id": "evt-001"}) + "\n"
truncated = '{"event_id":"evt-002"'
payload = complete + truncated

lines = payload.splitlines()
assert json.loads(lines[0])["event_id"] == "evt-001"

try:
    json.loads(lines[1])
except json.JSONDecodeError:
    print("trailing record invalid")
```

Output:

```text
trailing record invalid
```

El recovery educativo conserva los bytes source, acepta solo líneas completas según contrato y reporta/quarantines el suffix inválido con número de línea. Nunca inventa el contenido faltante. Una línea inválida intermedia puede indicar un fallo distinto de un trailing partial record y requiere política explícita.

### Detecta el bug

Un loader ignora silenciosamente toda `JSONDecodeError` y continúa. ¿Qué evidencia pierde y cómo podría promover corrupción a un estado aparentemente válido?

## 17. Source, derived data y protocolo seguro EIDOLON

Para EIDOLON:

- raw source no se sobrescribe durante normalización;
- un índice o timeline exportado es derived y rebuildable;
- una migration produce otra versión antes de cualquier promotion explícita;
- provenance usa IDs y metadata de dominio, no solo atributos del filesystem.

**Ejemplo ejecutable:**

```python
import json
from pathlib import Path
from tempfile import TemporaryDirectory


def rebuild_index(source: Path) -> dict[str, int]:
    index: dict[str, int] = {}
    with source.open(encoding="utf-8") as file:
        for line_number, line in enumerate(file, start=1):
            record = json.loads(line)
            index[record["event_id"]] = line_number
    return index


with TemporaryDirectory() as directory:
    root = Path(directory)
    source = root / "journal.jsonl"
    derived = root / "events_by_id.json"
    source.write_text(
        '{"event_id":"evt-001"}\n{"event_id":"evt-002"}\n',
        encoding="utf-8",
    )
    source_snapshot = source.read_bytes()
    derived.write_text("corrupt", encoding="utf-8")

    rebuilt = rebuild_index(source)
    derived.write_text(json.dumps(rebuilt, sort_keys=True), encoding="utf-8")

    assert source.read_bytes() == source_snapshot
    assert json.loads(derived.read_text(encoding="utf-8")) == {
        "evt-001": 1,
        "evt-002": 2,
    }

print("derived rebuild: PASS")
```

En producción, la escritura derived usaría el protocolo temporal + validación + replace de la sección 14. El ejemplo aísla authority y rebuild.

### Source discipline

Si `events_by_id.json` y `journal.jsonl` difieren, ¿cuál puede descartarse y reconstruirse bajo este contrato? ¿Qué evidencia justifica la respuesta?

## 18. Crash receipt como evidencia, no como transacción

Un receipt sintético puede registrar:

```text
operation_id
last_confirmed_record
source_path
target_path
temporary_path
status
```

Sirve para diagnosticar dónde se interrumpió una operación. No garantiza que cada dato indicado sea durable, no reemplaza el source y no constituye un transaction log completo. El receipt también necesita su propio lifecycle y failure model.

Una buena inyección de fallo registra el checkpoint intentado, conserva paths temporales cuando la política lo exige y verifica estados permitidos: target anterior intacto, target nuevo completo o temporal identificable. “No hubo excepción” no basta como evidencia de crash consistency.

**Failure case ejecutable — excepción antes de replace:**

```python
import os
from pathlib import Path
from tempfile import mkstemp, TemporaryDirectory


def fail_before_replace(target: Path, text: str) -> None:
    descriptor, name = mkstemp(dir=target.parent, suffix=".tmp")
    temporary = Path(name)
    try:
        with os.fdopen(descriptor, "w", encoding="utf-8") as file:
            file.write(text)
            file.flush()
            os.fsync(file.fileno())
        assert temporary.read_text(encoding="utf-8") == text
        raise RuntimeError("injected before replace")
    finally:
        temporary.unlink(missing_ok=True)


with TemporaryDirectory() as directory:
    target = Path(directory) / "derived.txt"
    target.write_text("previous\n", encoding="utf-8")
    try:
        fail_before_replace(target, "candidate\n")
    except RuntimeError as error:
        assert str(error) == "injected before replace"
    else:
        raise AssertionError("failure was not injected")

    assert target.read_text(encoding="utf-8") == "previous\n"

print("pre-replace failure preserved target: PASS")
```

El experimento demuestra una excepción de proceso en un checkpoint controlado. No simula un kill entre instrucciones, un OS crash ni power loss.

### Diagnóstico

¿Qué campos mínimos conservarías para distinguir “falló antes de replace” de “replace retornó pero no se verificó el resultado”?

## 19. Metadata y tiempo de dominio

Metadata común incluye size, timestamps, permissions y ownership. Su significado depende del OS/filesystem. En particular:

```text
filesystem mtime ≠ event valid_time ≠ recorded_at
```

`mtime` indica una modificación del objeto según semántica del filesystem. Copias, restores, editores o sincronización pueden cambiarlo. No demuestra cuándo ocurrió el evento ni quién lo originó.

**Ejemplo ejecutable:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory) / "event.txt"
    path.write_text("happened_at=2020-01-01T00:00:00+00:00", encoding="utf-8")
    metadata = path.stat()

    assert metadata.st_size > 0
    assert metadata.st_mtime > 0

print("metadata observed, domain time remains explicit")
```

Output:

```text
metadata observed, domain time remains explicit
```

La provenance de dominio debe usar `source_id`, operation IDs y timestamps explícitos con su contrato; `mtime` puede ser evidencia operacional complementaria.

## 20. Permissions y least privilege

En Unix-like, permisos clásicos distinguen user, group y others con capacidades read/write/execute. Windows usa un modelo diferente; no fuerces una equivalencia exacta.

`PermissionError` significa que la operación fue rechazada bajo permisos/estado actuales. Antes de cambiar permisos, recoge:

- operación y path resuelto;
- identidad efectiva del proceso;
- permisos/ownership observados;
- parent directory y mount/contexto relevante;
- si el path esperado es realmente el usado.

No uses `chmod 777` como solución general: amplía acceso, puede ocultar ownership/configuration incorrectos y viola least privilege. El proceso debe recibir solo permisos necesarios para su contrato.

### Diagnóstico

El journal no abre para append. Propón una secuencia de evidencia antes de modificar permisos. Distingue bug de path, parent ausente y permiso insuficiente.

## 21. Open-file limits y descriptor leaks

Un proceso y el OS mantienen límites de files abiertos. Este antipatrón conserva recursos:

```python
files = [open(path, encoding="utf-8") for path in paths]
```

Un test responsable no intenta agotar el límite. Reproduce un leak pequeño y lo corrige:

**Ejemplo ejecutable y controlado:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory) / "small.txt"
    path.write_text("data", encoding="utf-8")

    leaked = [path.open(encoding="utf-8") for _ in range(8)]
    assert all(not file.closed for file in leaked)

    for file in leaked:
        file.close()
    assert all(file.closed for file in leaked)

print("controlled descriptors closed: PASS")
```

En código normal, abre solo lo necesario y usa `with`; el loop explícito aquí existe para observar ownership conjunto. Síntomas de un leak real incluyen “too many open files”, fallos al abrir otros recursos y crecimiento del conjunto de descriptors.

### Detecta el bug

Una función retorna un iterator que lee de un file abierto dentro de `with`, pero el caller consume después. ¿Qué lifecycle se rompió? Relaciónalo con PF-M7.

## 22. Temporary files y ownership

`tempfile` crea nombres/files/directories temporales con mecanismos portables de la standard library. “Temporal” describe lifecycle esperado, no storage durable ni cleanup infalible ante toda terminación.

```text
temporary name ≠ open temporary file ≠ durable permanent record
```

`TemporaryDirectory` limpia al salir normalmente de su context manager. Un crash abrupto puede dejar residuos. El owner debe decidir si residuos se borran, se inspeccionan o se conservan como evidencia. Para atomic replace, ubica el temporal junto al target; el temp directory global podría estar en otro filesystem.

### Modifica

Haz que el ejemplo de safe replace use un prefix identificable por operation ID. Explica cómo distinguir un temporal huérfano legítimo de un archivo arbitrario antes de borrarlo.

## 23. Page cache y patrones de I/O

El OS puede conservar file data en RAM mediante page cache. Por eso un segundo read puede ser más rápido sin que el algoritmo haya cambiado. La page cache del OS no es un `dict` cache o índice de la aplicación: tienen owners, políticas y observabilidad distintas.

Un sequential scan recorre posiciones cercanas; random access salta entre offsets. El workload, dataset, cache state, buffering, storage y logging cambian el costo. Una operación lógica también puede causar más trabajo físico —read/write amplification—, pero CS-M10 profundizará hardware y locality.

**Benchmark educativo ejecutable:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory
from time import perf_counter_ns

with TemporaryDirectory() as directory:
    path = Path(directory) / "sample.bin"
    path.write_bytes(b"E" * 262_144)

    timings = []
    for _ in range(3):
        started = perf_counter_ns()
        payload = path.read_bytes()
        timings.append(perf_counter_ns() - started)
        assert len(payload) == 262_144

assert len(timings) == 3
assert all(elapsed >= 0 for elapsed in timings)
print("repeated reads measured")
```

Output estable:

```text
repeated reads measured
```

No exige que el segundo tiempo sea menor: scheduler, filesystem, hardware y ruido pueden invertir el orden. “Cold-ish” solo significa que intentaste reducir efectos previos y documentaste el método; limpiar page cache requiere privilegios y afecta el sistema, por lo que no forma parte de este laboratorio. No hagas `print` o logging por record dentro del hot path.

### Benchmark

La segunda ejecución es 4× más rápida. Enumera al menos tres hipótesis y una medición adicional antes de atribuir la mejora al algoritmo.

## 24. Observabilidad práctica y límites de plataforma

**Comandos Linux; disponibilidad y output varían:**

| Comando | Pregunta que ayuda a responder |
|---|---|
| `ps -p <PID> -o pid,ppid,stat,rss,vsz,command` | ¿Qué proceso/estado/memoria reporta una instantánea? |
| `top -p <PID>` | ¿Cómo cambian CPU y memoria observadas? |
| `free -h` | ¿Cómo reporta Linux memoria del sistema y cache? |
| `df -h <PATH>` | ¿Qué filesystem/mount contiene un path y cuánto espacio reporta? |
| `du -sh <PATH>` | ¿Cuánto espacio suman entries accesibles bajo un path? |
| `lsof -p <PID>` | ¿Qué recursos/files abiertos reporta la herramienta? |

`lsof` puede no estar instalado. `df` y `du` responden preguntas distintas; no deben coincidir. `/proc/<PID>` es una interfaz Linux-specific con datos de proceso, por ejemplo `status`, `fd/` y `cwd`; su estructura no es API portable de Python.

Antes de interpretar un resultado registra OS, filesystem cuando sea relevante, Python executable, cwd, PID, comando y timestamp de observación.

Un exit code también pertenece a la evidencia del proceso. La forma de consultarlo depende del shell:

```bash
# POSIX shell
python -c "raise SystemExit(3)"
echo $?
```

```powershell
# Windows PowerShell
python -c "raise SystemExit(3)"
$LASTEXITCODE
```

Ambos deben observar `3` si el launcher `python` ejecutó ese intérprete. En `cmd.exe`, la variable correspondiente es `%ERRORLEVEL%`. No confundas el exit code del proceso con una excepción que otro proceso Python pueda capturar internamente.

```text
Python semantics        open(), with, exceptions de la API
POSIX/Unix-like         descriptors, inode model, directory fsync pattern
Linux-specific          /proc, columnas/herramientas del entorno Linux
platform-dependent      rename, permissions, cache y durability exactas
```

### Portability

Clasifica `Path`, inode, `/proc/<PID>/fd`, `os.replace` y ACLs de Windows. Una API disponible en Python no vuelve idéntica la garantía subyacente.

## 25. Locking, `mmap` y child processes: previews

Abrir un file no bloquea automáticamente a otros writers. Coordinar procesos puede requerir locks, ownership o un solo writer; CS-M8 desarrollará processes, threads y sincronización.

`mmap` relaciona contenido de file con virtual memory mapping. Es una muestra de que memory y filesystem interactúan; no es “leer gratis”, no elimina page faults ni reemplaza una política de durability. Su implementación queda fuera.

Un proceso puede crear child processes que reciben un environment y ciertos recursos según plataforma/API. `fork`, spawn, multiprocessing, zombie/orphan lifecycle y coordinación quedan para CS-M8. Aquí solo necesitamos reconocer que recursos pertenecen a una ejecución concreta.

## 26. Catálogo de failure cases y correcciones

| Creencia o fallo | Síntoma/riesgo | Corrección y evidencia |
|---|---|---|
| `flush()` = durable | dato visible pero vulnerable al failure model | separa buffer Python de sync al OS; documenta modelo |
| `close()` = `fsync()` | cierre correcto interpretado como commit durable | solicita política explícita; prueba plataforma/filesystem |
| rename = durable | target visible pero directory metadata no garantizada tras power loss | distingue atomic visibility y durability |
| temp en otro filesystem | replace falla o tooling copia/borra | crea temporal junto al target y verifica contrato |
| overwrite source in-place | pérdida de evidencia ante fallo | conserva source; escribe derived/version nueva |
| `del file` cierra confiablemente | cleanup dependiente de runtime | `with`/`try-finally` y ownership |
| files abiertos sin límite | descriptor exhaustion | apertura acotada y cierre determinista |
| page cache = app cache | atribución/invalidación incorrecta | identifica owner y capa de cada cache |
| repeated read = mejor algoritmo | benchmark engañoso | controla cache state, workload e instrumentación |
| `mtime` = event time | semántica histórica falsa | timestamps de dominio explícitos |
| `chmod 777` arregla permisos | acceso excesivo y causa oculta | diagnostica path/identity/ownership; least privilege |
| Unix = Windows | fallos de rename/handles/permissions | etiqueta plataforma y prueba supported targets |
| GC impide leaks | collections alcanzables crecen | inspecciona referencias y retention policy |
| `getsizeof` = process RAM | número incompleto | elige shallow size, tracemalloc o RSS por pregunta |

## 27. Ejercicios guiados

### Guiado 1 — Inspecciona proceso y entorno

**Objetivo:** distinguir programa y proceso. Predice qué campos cambian entre dos ejecuciones.

**Solución ejecutable:**

```python
import os
import sys

evidence = {
    "pid": os.getpid(),
    "cwd": os.getcwd(),
    "python": sys.executable,
    "has_path_variable": "PATH" in os.environ,
}
assert evidence["pid"] > 0
assert all(evidence[key] for key in ("cwd", "python"))
assert isinstance(evidence["has_path_variable"], bool)
print("process evidence: PASS")
```

El contrato comprueba presencia, no valores ambientales. Ejecuta dos veces y compara PID; documenta si el OS reutiliza un PID en otro momento.

### Guiado 2 — Observa crecimiento rastreado

**Objetivo:** medir allocations Python sin llamarlas RSS.

```python
import tracemalloc

tracemalloc.start()
before, _ = tracemalloc.get_traced_memory()
batch = [bytearray(2048) for _ in range(64)]
current, peak = tracemalloc.get_traced_memory()
assert current >= before and peak >= current and len(batch) == 64
tracemalloc.stop()
print("traced growth: PASS")
```

El resultado prueba crecimiento rastreado durante el experimento, no working set total.

### Guiado 3 — Provoca retention lógica

**Objetivo:** mostrar que alcanzable no es collectible.

```python
registry: dict[int, bytearray] = {}
for item_id in range(32):
    registry[item_id] = bytearray(512)

assert len(registry) == 32
registry.clear()
assert registry == {}
print("logical cleanup: PASS")
```

La corrección es eliminar referencias según policy. No afirmes que RSS debe caer inmediatamente.

### Guiado 4 — Corrige el lifecycle

**Objetivo:** separar cleanup de memoria y resource cleanup.

```python
from tempfile import TemporaryFile

with TemporaryFile(mode="w+b") as file:
    file.write(b"event")
    assert not file.closed

assert file.closed
print("deterministic close: PASS")
```

`with` delimita ownership; GC deja de ser parte del contrato de cierre.

### Guiado 5 — Abre y cierra correctamente

**Objetivo:** comprobar que el file no se usa fuera de lifecycle.

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory) / "one.txt"
    with path.open("w", encoding="utf-8") as file:
        file.write("one")
    assert file.closed
    assert path.read_text(encoding="utf-8") == "one"

print("open/close: PASS")
```

La lectura posterior abre otro recurso; el anterior está cerrado.

### Guiado 6 — Descriptor leak pequeño

**Objetivo:** observar ownership múltiple sin agotar límites.

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory) / "data.txt"
    path.write_text("x", encoding="utf-8")
    resources = [path.open() for _ in range(4)]
    assert sum(not item.closed for item in resources) == 4
    for item in resources:
        item.close()
    assert sum(not item.closed for item in resources) == 0

print("bounded leak corrected: PASS")
```

El test prueba estado de los file objects, no el límite del OS.

### Guiado 7 — Compara write, flush y close

**Objetivo:** escribir qué demuestra cada checkpoint.

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory) / "steps.txt"
    file = path.open("w", encoding="utf-8")
    written = file.write("step\n")
    assert written == 5
    file.flush()
    assert path.read_text(encoding="utf-8") == "step\n"
    file.close()
    assert file.closed

print("layer checkpoints: PASS")
```

Demuestra count, visibility observada y cierre. No demuestra power-loss durability.

### Guiado 8 — Solicita `fsync`

**Objetivo:** ubicar `flush` antes de `fsync`.

```python
import os
from tempfile import TemporaryFile

with TemporaryFile(mode="w+b") as file:
    file.write(b"record\n")
    file.flush()
    os.fsync(file.fileno())
    file.seek(0)
    assert file.read() == b"record\n"

print("fsync sequence: PASS")
```

El assert valida contenido, no hardware. Registra OS/filesystem si la garantía importa.

### Guiado 9 — Temporal + replace

**Objetivo:** conservar target anterior hasta promotion.

Usa `safe_replace_text` de la sección 14 con target inicial `old`. Predice el estado antes/después de `os.replace`. La solución correcta crea el temporal en `target.parent`, valida cerrado, solicita sync según policy y limpia residuos. El criterio no es el nombre exacto del temporal.

### Guiado 10 — Simula partial write

**Objetivo:** detectar longitud incompleta sin depender de que el OS produzca una short write.

```python
expected = b"abcdefgh"
simulated_count = 3
partial = expected[:simulated_count]

assert len(partial) != len(expected)
assert partial == b"abc"
print("partial result detected")
```

La simulación enseña la invariante `bytes completed == bytes requested`; no afirma que `Path.write_bytes` se comporte así.

### Guiado 11 — Detecta JSONL final truncado

**Objetivo:** preservar records completos y reportar suffix inválido.

```python
import json

payload = '{"id":"evt-1"}\n{"id":"evt-2"'
valid = []
invalid_lines = []

for line_number, line in enumerate(payload.splitlines(), start=1):
    try:
        valid.append(json.loads(line))
    except json.JSONDecodeError:
        invalid_lines.append(line_number)

assert valid == [{"id": "evt-1"}]
assert invalid_lines == [2]
print("trailing failure reported: PASS")
```

El contenido inválido se reporta; una política real también conserva bytes/path y no asume que todo error es trailing.

### Guiado 12 — Rebuild derived sin tocar source

**Objetivo:** demostrar authority.

Usa `rebuild_index` de la sección 17, captura `source.read_bytes()` y corrompe solo el derived. La solución aprueba si el nuevo índice coincide con replay y source permanece byte por byte idéntico.

### Guiado 13 — Observa metadata

**Objetivo:** inspeccionar size/mode/time sin atribuir semántica de dominio.

```python
from pathlib import Path
from tempfile import TemporaryDirectory

with TemporaryDirectory() as directory:
    path = Path(directory) / "metadata.txt"
    path.write_text("abc", encoding="utf-8")
    stat = path.stat()
    assert stat.st_size == 3
    assert stat.st_mtime_ns > 0

print("metadata properties: PASS")
```

`st_mode` y ownership son platform-dependent; no fijes un valor universal.

### Guiado 14 — Separa mtime y domain time

**Objetivo:** impedir una inferencia falsa.

```python
record = {
    "event_id": "evt-001",
    "valid_time": "2020-01-01T00:00:00+00:00",
}
filesystem_mtime_is_domain_time = False

assert record["valid_time"].startswith("2020-")
assert not filesystem_mtime_is_domain_time
print("time semantics separated: PASS")
```

El filesystem puede aportar evidence operacional, no reemplazar el campo de dominio.

### Guiado 15 — Repeated read benchmark

**Objetivo:** medir sin concluir causalidad automática.

Ejecuta el benchmark de la sección 23 con 5 repeticiones. Reporta median/range, tamaño, Python/OS y si había logging. La solución correcta menciona page cache como hipótesis, no exige monotonía ni limpia caches del sistema.

### Guiado 16 — Documenta assumptions de portabilidad

**Objetivo:** clasificar garantías.

Completa:

| Operación/observación | Capa | Assumption que debe verificarse |
|---|---|---|
| `with open(...)` | Python | encoding/mode/path |
| directory `fsync` | Unix-like | soporte del OS/filesystem |
| `/proc/<PID>/fd` | Linux | procfs disponible/permisos |
| atomic replace | plataforma/filesystem | mismo filesystem y contrato |

Una solución correcta no llama portable a `/proc` ni promete semántica Windows idéntica.

## 28. Ejercicios independientes

1. Dibuja program → process → runtime → kernel para un script que abre un journal.
2. Explica address space y virtual memory sin usar “swap” como definición.
3. Traza call stack conceptual y referencias de una función que retorna una lista.
4. Diseña un experimento pequeño que distingue temporary allocations de retained allocations.
5. Compara `getsizeof`, `tracemalloc` y RSS para tres preguntas concretas.
6. Encuentra una closure o registry que retenga objetos y añade cleanup verificable.
7. Refactoriza cuatro aperturas manuales para que ownership/cierre sean explícitos.
8. Reproduce un descriptor leak de máximo 10 files y corrígelo sin depender de GC.
9. Dibuja path → directory entry → metadata/content; etiqueta inode como Unix-like.
10. Explica qué puede fallar durante `open()` y qué evidencia conservarías.
11. Escribe un cuadro de garantías para write/flush/fsync/close.
12. Implementa temporal + validate + replace en el mismo directorio.
13. Inyecta una excepción antes de replace y comprueba que el target anterior queda intacto.
14. Explica por qué directory sync puede importar y por qué no es receta portable.
15. Simula short write con una función que entrega chunks y exige consumir todos.
16. Diseña recovery para una última línea JSONL inválida sin modificar source.
17. Contrasta process crash, OS crash y power loss para el mismo update.
18. Corrompe un índice derived y reconstruye desde journal source.
19. Registra size/mtime/permisos y separa cada dato de event time/provenance.
20. Diagnostica `PermissionError` sin proponer permisos universales.
21. Compara sequential y random reads bajo un dataset reproducible pequeño.
22. Ejecuta repeated reads, reporta distribución y discute page cache.
23. Repite el benchmark con y sin print dentro del loop y explica la contaminación.
24. Usa `ps` y, si está disponible, `lsof`; documenta qué es Linux/tool-specific.
25. Describe qué cambia conceptualmente al aparecer un segundo writer, pero difiere la solución a CS-M8.

## 29. Preguntas conceptuales

1. ¿Qué diferencia existe entre programa y proceso?
2. ¿Qué recursos forman parte del proceso en este modelo?
3. ¿Por qué un venv no es process isolation?
4. ¿Qué abstrae virtual memory y por qué no equivale a swap?
5. ¿Qué límites tiene el modelo stack/heap en Python?
6. ¿Por qué `del name` no garantiza devolución inmediata de memoria al OS?
7. ¿Cómo puede existir un memory leak lógico con garbage collection?
8. ¿Por qué GC no sustituye a `with`?
9. ¿Qué diferencia existe entre shallow size, traced allocations y RSS?
10. ¿Qué diferencia hay entre path, file object y contenido?
11. ¿Qué representa un file descriptor y qué matiz agrega Windows?
12. ¿Por qué un read no implica acceso físico al dispositivo?
13. ¿Por qué `file.write()` no implica durability?
14. ¿Qué diferencia hay entre `flush()` y `fsync()`?
15. ¿Qué demuestra `close()` y qué no demuestra?
16. ¿Qué diferencia existe entre atomicity y durability?
17. ¿Qué garantiza un atomic replace solo bajo assumptions declaradas?
18. ¿Por qué directory `fsync` puede ser relevante después de rename?
19. ¿Por qué cross-filesystem rename puede tener otra semántica?
20. ¿Qué es crash consistency?
21. ¿Qué cambia entre process crash, OS crash y power loss?
22. ¿Por qué append no es una transaction?
23. ¿Cómo recuperas un trailing JSONL inválido sin inventar datos?
24. ¿Por qué derived corruption se corrige mediante rebuild desde source?
25. ¿Por qué filesystem `mtime` no es event time ni provenance suficiente?
26. ¿Qué evidencia distingue PermissionError de un path incorrecto?
27. ¿Por qué `chmod 777` no es un diagnóstico?
28. ¿Cómo page cache puede alterar un benchmark?
29. ¿Por qué la segunda lectura más rápida no prueba una mejora algorítmica?
30. ¿Qué diferencias debes etiquetar entre Python, POSIX y Linux?

## 30. Mini challenge — Journal y export recuperable

### Objetivo y artefactos

Construye un experimento local, sintético y reproducible con:

```text
cs_m7_challenge/
├── journal.jsonl          # source preservado
├── derived_timeline.jsonl # rebuildable
├── experiment.py
└── NOTES.md               # guarantees + portability + measurements
```

Todo el test destructivo debe trabajar sobre una copia dentro de `TemporaryDirectory`; el artefacto source original del repositorio no se modifica.

### A. Process y memoria

1. Registra PID, Python executable y cwd.
2. Inicia `tracemalloc`, crea un batch acotado de `bytearray` y consérvalo en un registry.
3. Demuestra retention por cantidad de referencias y traced growth.
4. Limpia el registry; afirma cleanup lógico, no descenso inmediato de RSS.
5. Si observas RSS con `ps`, etiqueta la medición Linux y conserva comando/unidades.

### B. Resource lifecycle

6. Abre cuatro copies del journal en un leak controlado.
7. Comprueba que están abiertos, ciérralos en `finally` y comprueba `closed`.
8. Implementa el camino normal con `with` y explica por qué no depende de GC.

### C. Safe derived write

9. Lee records JSONL completos desde source sin mutarlo.
10. Construye una timeline determinista derived.
11. Crea un temporal en `derived_target.parent`.
12. Escribe, cierra y valida cada línea.
13. Usa `flush()` seguido de `os.fsync()` en el file según la política educativa.
14. Ejecuta `os.replace()` solo después de validar.
15. Documenta que atomic visibility depende de mismo filesystem/OS y no implica por sí sola durability de directory/hardware.

### D. Failure injection

16. Parámetro `failure_point` admite `before_replace` y `truncate_temporary`.
17. `before_replace` lanza una excepción controlada después de validar y antes de replace.
18. `truncate_temporary` elimina bytes del último record; validation debe rechazarlo.
19. Corrompe solo el derived final y demuestra que source permanece idéntico.
20. Conserva un crash receipt sintético con operation ID, checkpoint y paths; no lo llames transaction log.

### E. Recovery

21. Calcula hash o snapshot byte por byte del source antes/después solo para detectar cambios accidentales en el test.
22. Reporta trailing invalid JSONL con line number y raw fragment controlado.
23. No inventa el record incompleto.
24. Rebuild derived desde source y valida output determinista.
25. El target anterior permanece intacto si el fallo fue antes de replace.

### F. Measurement

26. Lee el mismo derived file al menos cinco veces con `perf_counter_ns`.
27. Reporta todos los tiempos, tamaño y entorno; no exige que disminuyan.
28. Discute page cache, scheduler y ruido; no imprime por record dentro del hot path.

### G. Portability note

29. `NOTES.md` clasifica cada mechanism como Python, POSIX/Unix-like, Linux-specific o platform/filesystem-dependent.
30. Explica qué validaría adicionalmente en Windows.

### Comprobaciones contractuales

**Continuación — adapta nombres a tu implementación:**

```python
source_before = source_path.read_bytes()

records = load_complete_records(source_path)
write_derived_safely(derived_path, records)
first_output = derived_path.read_bytes()

assert source_path.read_bytes() == source_before
assert validate_jsonl(derived_path)

try:
    write_derived_safely(
        derived_path,
        records,
        failure_point="before_replace",
    )
except RuntimeError:
    pass
else:
    raise AssertionError("failure was not injected")

assert derived_path.read_bytes() == first_output
assert source_path.read_bytes() == source_before

derived_path.write_text("corrupt", encoding="utf-8")
write_derived_safely(derived_path, load_complete_records(source_path))
assert derived_path.read_bytes() == first_output
assert source_path.read_bytes() == source_before
```

### Failure cases obligatorios

- retention registry no limpiado;
- descriptor leak acotado;
- `flush()` descrito falsamente como durable;
- excepción antes de replace;
- temporal truncado rechazado;
- derived corrupto reconstruido desde source;
- trailing JSONL inválido reportado;
- repeated-read speedup atribuido sin evidencia al algoritmo;
- assumption Unix presentada y luego clasificada correctamente.

### Criterio de aprobación

- el experimento corre con standard library y datos sintéticos;
- nunca modifica source durante derived export/recovery;
- mide tracemalloc y, opcionalmente, RSS sin confundirlos;
- cierra todos los recursos incluso tras failure injection;
- temporal y target están en el mismo directorio;
- validación precede a replace;
- atomicity/durability/failure model se documentan por separado;
- recovery conserva evidencia y no inventa records;
- timings no se presentan como universales;
- notas distinguen Python/POSIX/Linux/Windows;
- no aparecen threads, multiprocessing, sockets, database, backend ni AI.

## 31. Resumen

- Programa en disco y proceso en ejecución no son lo mismo.
- El runtime opera en user space y pide servicios al kernel; una API Python no mapea necesariamente uno-a-uno a una system call.
- Virtual memory abstrae y aísla address spaces; no equivale a swap.
- Stack/heap son modelos útiles pero incompletos para Python.
- Referencias determinan reachability; `del` no promete devolver memoria inmediatamente al OS.
- GC administra object memory; ownership/context managers administran recursos externos.
- Un leak lógico puede consistir en objetos correctamente alcanzables que ya no deberían retenerse.
- `getsizeof`, `tracemalloc` y RSS miden capas distintas.
- Path, directory entry, file object y descriptor/handle no son equivalentes.
- `write`, `flush`, `fsync` y `close` expresan checkpoints distintos.
- Atomicity, durability y crash consistency responden preguntas diferentes.
- Temporal + validate + sync policy + same-filesystem replace reduce estados parciales; no garantiza todo failure model.
- Directory sync es un detalle Unix-like relevante pero no una receta universal.
- Append no es transaction ni coordinación multi-process.
- JSONL puede dejar un último record incompleto; recovery conserva source y reporta.
- Derived data se reconstruye desde source; filesystem metadata no reemplaza provenance ni domain time.
- Page cache y logging pueden dominar un benchmark de I/O.
- Las observaciones Linux deben etiquetarse y no convertirse en semántica portable.

## 32. Checklist de dominio

- [ ] Puedo distinguir programa, proceso, runtime y OS.
- [ ] Puedo explicar user space/kernel boundary sin asumir una system call por API.
- [ ] Puedo explicar virtual memory sin reducirla a swap.
- [ ] Puedo usar stack/heap como modelos limitados para Python.
- [ ] Puedo predecir lifetime desde referencias.
- [ ] Puedo explicar por qué `del` no garantiza liberación inmediata al OS.
- [ ] Puedo distinguir GC de resource lifecycle.
- [ ] Puedo provocar y limpiar retention lógica acotada.
- [ ] Puedo elegir entre `getsizeof`, tracemalloc y RSS por pregunta.
- [ ] Puedo distinguir path, directory entry, file object y descriptor/handle.
- [ ] Puedo explicar read/write sin asumir acceso físico inmediato.
- [ ] Puedo diferenciar buffering de Python, page cache y storage caches.
- [ ] Puedo declarar qué demuestran write, flush, fsync y close.
- [ ] Puedo distinguir atomicity, durability y crash consistency.
- [ ] Puedo implementar temp + validate + replace dentro del mismo filesystem.
- [ ] Puedo explicar límites de rename y directory fsync.
- [ ] Puedo detectar partial/truncated JSONL sin inventar records.
- [ ] Puedo distinguir process crash, OS crash y power loss.
- [ ] Puedo preservar source y reconstruir derived data.
- [ ] Puedo separar filesystem metadata de domain time/provenance.
- [ ] Puedo diagnosticar permisos sin ampliar acceso indiscriminadamente.
- [ ] Puedo detectar y corregir descriptor leaks.
- [ ] Puedo explicar page cache como factor de benchmark.
- [ ] Puedo usar herramientas Linux como evidencia etiquetada.
- [ ] Puedo documentar assumptions Python/POSIX/Linux/Windows.
- [ ] Puedo resolver el mini challenge con PF + CS-M1–CS-M7.

## 33. Preparación para labs y EIDOLON 0.0b

- **CS-L12 — Memory pressure:** procesa JSONL streaming frente a materialización, usa `timeit`/`tracemalloc` y explica working set sin confundirlo con RSS total.
- **CS-L13 — Crash-safe file:** inyecta fallo entre temporal, write y rename; verifica que los estados resultantes sean recuperables bajo el contrato.
- **CS-L14 — Process inspection:** documenta PID, environment, cwd, open files y exit codes con herramientas del entorno.

| Concepto | Secciones | Evidencia | Lab/build |
|---|---:|---|---|
| Process/environment | 1–3, 24 | Guiado 1 | CS-L14 |
| Allocation/retention/measurement | 4–7 | Guiados 2–3 | CS-L12 |
| Resource lifecycle/descriptors | 5, 9, 21–22 | Guiados 4–6 | CS-L14 |
| Buffering/flush/fsync | 10–12 | Guiados 7–8 | CS-L13 |
| Atomicity/durability/crash model | 13–15 | Guiado 9 | CS-L13 |
| Partial JSONL/recovery | 16–18 | Guiados 10–12 | CS-L13, EIDOLON 0.0b |
| Metadata/permissions | 19–20 | Guiados 13–14 | CS-L14 |
| Page cache/benchmark | 23 | Guiado 15 | CS-L12, EIDOLON 0.0b |
| Portability | 15, 24–25 | Guiado 16 | CS-L13, CS-L14 |

Antes de avanzar entrega: código del mini challenge, outputs/asserts, tabla de failure models, evidencia de source intacto, resultados de failure injection, mediciones con contexto y nota de portabilidad.

CS-M7 aporta a EIDOLON 0.0b límites de working set, lifecycle de temporales/logs, export derived recuperable y trazabilidad desde CLI/process hasta filesystem. CS-M8 podrá partir de un modelo explícito del proceso y sus recursos, sin que este módulo implemente concurrencia.

## 34. Recursos de ampliación

El módulo es autocontenido. Para profundizar usa los recursos canónicos de [CS.11 Recursos recomendados](../../02_curriculum/02_computer_science_foundations.md#cs11-recursos-recomendados) y documentación oficial de Python para `os`, `pathlib`, `tempfile` y `tracemalloc`. Consulta documentación del OS/filesystem desplegado antes de convertir una observación en garantía.

## 35. Límite explícito del módulo

CS-M7 termina en process/address-space fundamentals, object/resource lifecycle, medición de memoria, filesystem model, descriptors/handles, buffering/page cache, flush/fsync, atomic replace, durability/crash consistency, metadata/permissions, I/O measurement y safe derived recovery.

No desarrolla page tables/TLB, scheduling algorithms, kernel/ABI/assembly, filesystem journaling internals, SSD internals, locking, threads, multiprocessing, sockets, networking, database transactions, distributed storage, backups de producción, Docker, backend ni AI.

El siguiente paso permitido es revisar CS-M7 como `review candidate`. **No se crea ni se desarrolla CS-M8 en esta entrega.**
