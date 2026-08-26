# PF-M6 — Excepciones, archivos, JSON y lifecycle de recursos

**Track:** Programming Foundations  
**Competencias:** D1.1; refuerza D2.3, D3.1 y D11.1  
**Fase:** P0  
**Nivel objetivo:** Profesional para el alcance del módulo  
**Prerequisites:** PF-M1, PF-M2, PF-M3, PF-M4, PF-M5  
**Build:** EIDOLON 0.0a  
**Curriculum source:** [PF-M6](../../02_curriculum/01_programming_foundations.md#pf-m6--excepciones-archivos-json-y-lifecycle-de-recursos)  
**Status:** approved

Guardar un Event no consiste solo en llamar `write`. Entre el objeto válido y los bytes persistidos pueden aparecer datos no serializables, paths equivocados, permisos insuficientes, texto con otro encoding, escrituras parciales, versiones desconocidas o una línea corrupta. Ocultar cualquiera de esos fallos puede convertir un problema detectable en pérdida silenciosa de evidencia.

PF-M6 enseña a diseñar esa frontera con excepciones específicas, resources con lifecycle delimitado, JSON como contrato explícito y JSONL como journal educativo append-only. El hilo conductor es:

```text
operación normal
      ↓
puede fallar
      ↓
clasificar el fallo
      ↓
propagar o recuperar
      ↓
proteger recursos
      ↓
preservar datos originales
      ↓
hacer el comportamiento reproducible
```

PF-M1–PF-M5 ya aportan Unicode, `datetime`, funciones puras, iteración, packages, dataclasses, type hints e invariantes. Aquí se reutilizan sin reabrir su teoría. Se usan context managers existentes mediante `with`, pero PF-M7 enseñará a implementar los propios. PF-M8 cubrirá async y PF-M9 ampliará testing, debugging y logging.

## Resultados de aprendizaje

Al terminar deberías poder:

- distinguir resultado válido, ausencia válida, fallo esperado y bug;
- explicar propagación, traceback, tipo, mensaje y call stack al nivel práctico;
- rechazar una entrada que viola un contrato mediante `raise`;
- delimitar un `try` y capturar excepciones específicas sin destruir evidencia;
- usar `else` y `finally` cuando expresan una frontera real;
- decidir qué capa puede recuperar un fallo y cuándo debe propagarlo;
- conservar una causa original mediante exception chaining;
- diseñar una taxonomía pequeña para input, dominio, serialization, I/O y bugs;
- comparar LBYL y EAFP sin tratar ninguno como dogma;
- construir paths portables con `pathlib.Path` y diagnosticar el working directory;
- abrir texto y bytes con modos y encoding deliberados;
- administrar el lifecycle de un file handle mediante `with`;
- comparar lectura completa y procesamiento línea por línea;
- transformar Python values a JSON text y reconstruirlos;
- explicar qué tipos Python no conserva JSON automáticamente;
- separar un domain object de su representación persistida;
- serializar y reconstruir `datetime` timezone-aware, `Decimal`, Enum y dataclasses mediante una política explícita;
- validar `schema_version` y rechazar versiones desconocidas;
- escribir y leer JSONL en UTF-8 conservando número de línea;
- elegir una política fail fast, quarantine o reporte sin alterar source data;
- explicar por qué append no equivale a transacción;
- reducir el riesgo de un destino parcial mediante archivo temporal y replace;
- distinguir atomic replace práctico de durabilidad física;
- migrar records mediante una función pura y promover un resultado validado sin sobrescribir la única fuente;
- demostrar un round trip de Event, Claim y Correction sin mutar el objeto original.

## Cómo estudiar este módulo

Para cada operación:

1. escribe su resultado normal y sus fallos previsibles;
2. identifica qué capa posee suficiente información para actuar;
3. predice qué recurso existe antes, durante y después del fallo;
4. conserva tipo, ubicación, causa y evidencia no sensible;
5. separa transformación pura de efectos de filesystem;
6. prueba el round trip con Unicode y timestamps aware;
7. induce un fallo antes de afirmar que la recuperación funciona.

Los ejemplos usan datos sintéticos. No copies payloads reales en tracebacks, receipts o archivos de práctica.

### Convenciones de código

- **Ejemplo ejecutable:** bloque autónomo o archivo completo.
- **Continuación:** reutiliza solo el bloque o árbol inmediatamente anterior.
- **Código incorrecto:** antipatrón deliberado que puede aparentar funcionar.
- **Failure case:** provoca el error o daño indicado y declara la evidencia estable.
- **Fragmento:** omite contexto de forma explícita; no se ofrece como programa completo.
- **Solución parcial:** resuelve una frontera local, no el journal integrado.
- **Dependencia ambiental:** el tipo exacto de fallo depende del OS, permisos o filesystem y se indica.

Los ejemplos se validan con Python 3.14. Los mensajes completos de `OSError`, paths temporales y tracebacks pueden variar; se comprueba el tipo, el chaining o la propiedad estable. `assert` sirve para comprobaciones locales, no sustituye la estrategia de tests de PF-M9.

### Sintaxis de apoyo

- `ValueError` se reutiliza para invariantes locales; aquí se aprende a clasificarlo y propagarlo;
- `with` usa context managers de la standard library; `__enter__`, `__exit__` y `contextlib.contextmanager` quedan para PF-M7;
- `tempfile` crea temporales de forma portable;
- `os.fsync` aparece solo para distinguir buffers de durabilidad; CS Foundations profundizará crash consistency;
- el journal educativo no admite acceso concurrente ni se presenta como database.

---

## 1. El problema: una operación puede terminar de varias maneras

### 1.1 Un único sentinel borra información

**Código incorrecto:**

```python
def load_event(path):
    try:
        return path.read_text(encoding="utf-8")
    except Exception:
        return None
```

El caller que recibe `None` no sabe si el Event no existía, el path era incorrecto, faltaban permisos, el disco reportó otro fallo, el contenido no pudo decodificarse o existe un bug dentro del bloque demasiado amplio. `False` y `-1` tienen el mismo problema si representan indiscriminadamente cualquier fallo.

### 1.2 Cuatro resultados conceptuales

```text
resultado válido  → la función completó su contrato
ausencia válida   → el contrato admite que no exista un valor
error esperado    → una condición prevista impide completar la operación
bug               → el programa viola una suposición interna
```

Una función como `find_event(...) -> Event | None` puede usar `None` para “no encontrado” si esa ausencia forma parte del contrato. `load_journal(path)` no debería convertir un `PermissionError` en “journal vacío”: eso fabrica un estado válido falso.

Una exception es una ruta de control para informar que la operación no puede completar su contrato normalmente. El caller puede manejarla si sabe recuperar, traducirla en una frontera exterior o dejarla propagarse.

### 1.3 Tres fronteras distintas

```text
load_event(path)  → filesystem puede fallar
parse_event(text) → syntax/schema puede fallar
save_event(event) → serialization y filesystem pueden fallar
```

Separarlas evita que “no se pudo cargar” oculte dónde ocurrió el fallo.

### Predice

Si `load_journal` devuelve una lista vacía tanto para archivo vacío como para `PermissionError`, ¿qué estado falso observará el caller?

### Clasifica

Clasifica: Event no encontrado en una búsqueda opcional; JSON malformado; división entre cero por un cálculo interno; duplicate ID.

### Explica

¿Por qué `None` puede ser correcto en `find_event` y peligroso en `load_journal`?

### Refactoriza

Divide una función `load_and_parse` en una operación de I/O y una de parsing. Escribe el contrato de error de cada una antes del código.

---

## 2. Anatomía y propagación de una exception

### 2.1 `raise` interrumpe la ruta normal

**Ejemplo ejecutable:**

```python
def require_event_id(event_id: str) -> str:
    if not event_id:
        raise ValueError("event_id must not be empty")
    return event_id


print(require_event_id("evt-001"))
```

Output:

```text
evt-001
```

Si el ID está vacío, `return` no se alcanza.

**Failure case:**

```python
def require_event_id(event_id: str) -> str:
    if not event_id:
        raise ValueError("event_id must not be empty")
    return event_id


require_event_id("")
```

La propiedad estable es `ValueError` en la llamada a `require_event_id("")`. El traceback completo incluye rutas y líneas que dependen del archivo.

### 2.2 Propagación por el call stack

**Failure case ejecutable:**

```python
def parse_event_id(raw: str) -> str:
    if not raw:
        raise ValueError("event_id must not be empty")
    return raw


def build_event(raw_id: str) -> dict[str, str]:
    return {"event_id": parse_event_id(raw_id)}


def import_event(raw_id: str) -> dict[str, str]:
    return build_event(raw_id)


import_event("")
```

Python abandona `parse_event_id`, después `build_event` y después `import_event` porque ninguna capa captura la exception. El traceback permite reconstruir esa ruta:

```text
import_event
└── build_event
    └── parse_event_id
        └── raise ValueError
```

El traceback aporta tipo, mensaje, punto donde se ejecutó `raise` y callers activos que condujeron a ese punto.

### 2.3 Rechazar no es “arreglar” silenciosamente

**Código incorrecto:**

```python
def normalize_schema_version(version: int) -> int:
    if version not in {1, 2}:
        return 2
    return version
```

Convertir una versión desconocida a 2 inventa compatibilidad. Una frontera más honesta la rechaza:

```python
def require_supported_version(version: int) -> int:
    if version not in {1, 2}:
        raise ValueError(f"unsupported schema version: {version}")
    return version
```

El mismo criterio aplica a timestamp naive, ID vacío o estado imposible: una reparación solo es correcta si el contrato define la transformación y conserva la evidencia original.

### Predice

En la cadena anterior, ¿qué functions dejan de ejecutarse después del `raise`?

### Explica

¿Qué diferencia hay entre el punto donde se detecta el fallo y la capa que decide cómo presentarlo?

### Modifica

Agrega a `require_supported_version` una versión soportada sin cambiar qué ocurre con versiones desconocidas.

### Detecta el bug

¿Qué evidencia destruye `normalize_schema_version(99)` y por qué el valor devuelto parece válido?

---

## 3. `try` y `except`: capturar únicamente lo que puedes manejar

### 3.1 Captura específica

**Ejemplo ejecutable:**

```python
import json
from json import JSONDecodeError


def parse_json_object(text: str) -> dict[str, object]:
    try:
        value = json.loads(text)
    except JSONDecodeError as exc:
        print(f"invalid JSON near character {exc.pos}")
        raise

    if not isinstance(value, dict):
        raise ValueError("expected a JSON object")
    return value


record = parse_json_object('{"event_id": "evt-001"}')
print(record["event_id"])
```

Output:

```text
evt-001
```

`as exc` enlaza la instancia de la exception. El `raise` sin argumento relanza la misma exception con su traceback.

### 3.2 Múltiples ramas expresan decisiones distintas

**Fragmento — frontera de lectura:**

```python
from pathlib import Path


def read_journal_text(path: Path) -> str:
    try:
        return path.read_text(encoding="utf-8")
    except FileNotFoundError:
        raise
    except PermissionError:
        raise
    except OSError:
        raise
```

Tal como está, las ramas no agregan valor y deberían omitirse: propagar es el default. Se justifican solo si cada rama recupera, traduce o añade contexto sin perder la causa. `FileNotFoundError` y `PermissionError` son subclasses de `OSError`, por eso las ramas específicas deben aparecer antes de `except OSError`.

### 3.3 Mantén pequeño el `try`

**Código incorrecto:**

```python
try:
    text = path.read_text(encoding="utf-8")
    record = json.loads(text)
    event = event_from_record(record)
    index[event.event_id] = event
except OSError:
    print("filesystem error")
```

Aunque `OSError` no captura los demás tipos, el bloque amplio dificulta identificar qué operación se pretendía manejar y favorece agregar después un `except Exception` indiscriminado. Aísla la operación recuperable.

### 3.4 El antipatrón que destruye evidencia

**Código incorrecto:**

```python
try:
    append_record(path, record)
except Exception:
    pass
```

Qué ocurrió: cualquier exception quedó silenciada.  
Por qué: `Exception` cubre fallos previstos y numerosos bugs.  
Evidencia perdida: tipo, traceback, causa, path y estado de la operación.  
Capa adecuada: solo una frontera que pueda recuperar debe capturar; de otro modo propaga.

Imprimir y continuar tampoco repara:

```python
try:
    append_record(path, record)
except Exception:
    print("algo salió mal")

publish_success()
```

Ahora el programa anuncia éxito después de un estado desconocido.

### Predice

¿`except OSError` captura `JSONDecodeError`? ¿Por qué importa la taxonomía?

### Detecta el bug

En el segundo antipatrón, ¿qué afirmación falsa puede producir `publish_success()`?

### Repara

Retira la captura o limita el `try` a una exception que la capa pueda resolver. Escribe qué debe ocurrir después del fallo.

### Clasifica

¿Qué rama usarías para `FileNotFoundError`, `PermissionError`, malformed JSON y duplicate ID? No escribas una rama si la función no sabe recuperarse.

---

## 4. `else` y `finally`

### 4.1 `else` separa el camino exitoso

**Ejemplo ejecutable:**

```python
import json
from json import JSONDecodeError


text = '{"schema_version": 1}'

try:
    record = json.loads(text)
except JSONDecodeError:
    print("invalid JSON")
else:
    print(f"schema={record['schema_version']}")
```

Output:

```text
schema=1
```

Solo `json.loads` necesita el `try`. El acceso posterior queda en `else`; si contiene un bug como una key incorrecta, no se confunde con el fallo de parsing.

### 4.2 `finally` ejecuta cleanup

**Ejemplo ejecutable:**

```python
events: list[str] = []

try:
    events.append("operation-started")
    raise ValueError("synthetic failure")
except ValueError:
    events.append("failure-observed")
finally:
    events.append("cleanup")

print(events)
```

Output:

```text
['operation-started', 'failure-observed', 'cleanup']
```

`finally` se ejecuta tanto en éxito como durante propagación. No significa “ignorar el error”: después de cleanup, una exception no capturada continúa.

### 4.3 No son ceremonia obligatoria

- usa `else` si hace visible qué operación estaba protegida;
- usa `finally` si adquiriste algo que siempre requiere cleanup;
- para file handles, `with` suele expresar el lifecycle con menos riesgo;
- no agregues `finally: pass` ni ramas vacías.

### Predice

Si eliminas el `except ValueError`, ¿se agrega `"cleanup"` antes de que la exception continúe?

### Explica

¿Por qué mover código exitoso a `else` puede impedir capturas accidentales?

### Modifica

Haz que el ejemplo no falle. Comprueba que `cleanup` sigue apareciendo.

---

## 5. Capturar, propagar y traducir entre capas

### 5.1 La pregunta de diseño

> ¿Esta capa realmente sabe qué hacer con este error?

```text
domain function
      ↓
serialization
      ↓
filesystem adapter
      ↓
CLI
```

- dominio rechaza una invariante, pero no decide mensajes de terminal;
- serialization conoce records y schema, no permisos del usuario;
- filesystem conoce paths y I/O, no si un Claim es verdadero;
- CLI puede traducir un fallo conocido a un mensaje breve y un exit status, pero no debe fingir éxito.

Si una capa no puede recuperar ni añadir contexto útil, normalmente deja propagar.

### 5.2 Exception chaining conserva dos niveles

**Ejemplo ejecutable:**

```python
import json
from json import JSONDecodeError


class JournalCorruptionError(Exception):
    pass


def parse_journal_line(raw_line: str, line_number: int) -> dict[str, object]:
    try:
        value = json.loads(raw_line)
    except JSONDecodeError as exc:
        raise JournalCorruptionError(
            f"invalid JSON at line {line_number}"
        ) from exc

    if not isinstance(value, dict):
        raise JournalCorruptionError(
            f"expected object at line {line_number}"
        )
    return value


try:
    parse_journal_line("{bad json}", 27)
except JournalCorruptionError as exc:
    print(type(exc).__name__)
    print(str(exc))
    print(type(exc.__cause__).__name__)
```

Output:

```text
JournalCorruptionError
invalid JSON at line 27
JSONDecodeError
```

El error exterior expresa el contrato del journal; `__cause__` conserva el fallo técnico original.

### 5.3 Cómo se pierde o suprime la causa

**Código incorrecto:**

```python
try:
    path.write_text(text, encoding="utf-8")
except OSError:
    raise JournalWriteError("could not write journal") from None
```

`from None` suprime intencionalmente el contexto al mostrar el traceback. Puede existir un caso de UX para ocultar detalle al usuario final, pero no debe ser la única evidencia técnica. En código interno, `raise JournalWriteError(...) from exc` conserva una causa explícita. Capturar y devolver `False` destruye todavía más información.

### 5.4 Taxonomía pequeña

```python
class JournalError(Exception):
    pass


class UnsupportedSchemaVersionError(JournalError):
    pass


class JournalCorruptionError(JournalError):
    pass


class JournalWriteError(JournalError):
    pass
```

No hace falta una class por cada mensaje. Una excepción propia aporta valor cuando callers diferentes necesitan reconocer una categoría estable o cuando traduce una frontera sin perder `__cause__`.

### Predice

En el ejemplo de chaining, ¿qué exception captura el caller exterior y cuál queda como causa?

### Explica

¿Por qué la CLI puede traducir `JournalCorruptionError`, pero no debería capturar cualquier bug como “archivo inválido”?

### Detecta el bug

¿Qué efecto produce `from None` en la evidencia visible?

### Diseña el contrato

Decide si schema desconocido y JSON malformado deben compartir tipo. Justifica qué acción distinta permitiría separarlos.

---

## 6. LBYL y EAFP sin dogma

### 6.1 Dos estilos

**LBYL — Look Before You Leap:**

```python
if "event_id" in record:
    event_id = record["event_id"]
else:
    event_id = None
```

**EAFP — Easier to Ask Forgiveness than Permission:**

```python
try:
    event_id = record["event_id"]
except KeyError:
    event_id = None
```

Ambos pueden expresar un contrato válido. Si ausencia es normal, `dict.get` puede ser más claro. Si la key es obligatoria, convertirla a `None` quizá oculte un record inválido.

### 6.2 `exists()` no garantiza que `open` funcionará

**Código incorrecto:**

```python
if path.exists():
    text = path.read_text(encoding="utf-8")
```

Entre `exists` y `read_text` otro proceso puede mover el archivo, cambiar permisos o reemplazarlo. También puede existir un directory con ese nombre. La comprobación es una observación pasada, no una reserva del recurso.

**EAFP con la operación real:**

```python
try:
    text = path.read_text(encoding="utf-8")
except FileNotFoundError:
    # Esta capa decide si ausencia permite crear un journal nuevo.
    raise
```

`exists()` sigue siendo útil para UI o decisiones aproximadas. No elimina la necesidad de manejar el fallo de la operación real.

### 6.3 Cuándo mirar primero

LBYL sirve cuando la comprobación expresa una regla ya poseída en memoria. EAFP sirve cuando la operación es la fuente de verdad y una exception esperada puede capturarse específicamente. Ninguno elimina races externas por sí solo.

### Predice

¿Qué puede cambiar entre `path.exists()` y `path.open()`?

### Decide

Para una key opcional, compara `record.get("tags", [])` con capturar `KeyError`. ¿Cuál comunica mejor ausencia normal?

### Detecta el bug

¿Por qué `exists()` seguido de escritura no concede permisos ni espacio disponible?

---

## 7. `pathlib` y el significado de un path

### 7.1 Construir intención, no concatenar separadores

**Ejemplo ejecutable:**

```python
from pathlib import Path


project = Path("eidolon")
journal = project / "data" / "journal.jsonl"

print(journal.name)
print(journal.suffix)
print(journal.parent)
```

Output:

```text
journal.jsonl
.jsonl
eidolon/data
```

El operador `/` compone componentes y `Path` usa las reglas de la plataforma.

### 7.2 Operaciones prácticas

**Ejemplo ejecutable con directory temporal:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as directory:
    root = Path(directory)
    data_dir = root / "data"
    data_dir.mkdir(parents=True)

    path = data_dir / "journal.txt"
    path.write_text("Llegué 👋", encoding="utf-8")

    assert path.exists()
    assert path.is_file()
    assert path.parent == data_dir
    assert path.name == "journal.txt"
    assert path.suffix == ".txt"
    assert path.read_text(encoding="utf-8") == "Llegué 👋"

print("path checks: PASS")
```

Output:

```text
path checks: PASS
```

`mkdir(parents=True)` crea parents faltantes; `exist_ok=True` acepta que el directory ya exista. Decide si un path preexistente es estado válido.

### 7.3 Relative, absolute y working directory

`Path("data/journal.jsonl")` es relativo. Se interpreta desde `Path.cwd()`, no desde la ubicación del archivo `.py`.

**Ejemplo ejecutable:**

```python
from pathlib import Path


relative = Path("data/journal.jsonl")
absolute = relative.resolve()

print(relative.is_absolute())
print(absolute.is_absolute())
assert absolute == Path.cwd() / relative
```

Output:

```text
False
True
```

El texto del path absoluto depende de la máquina.

### 7.4 Ubicación de código no es ubicación de datos

**Fragmento — archivo Python real:**

```python
from pathlib import Path


MODULE_DIR = Path(__file__).resolve().parent
```

`__file__` ubica el módulo; `Path.cwd()` ubica el proceso. Un proyecto instalado puede tener código en `site-packages` y datos del usuario en otro lugar. PF-M6 recibe el path del journal como argumento; no escribe junto al package.

### Predice

Si ejecutas el mismo script desde dos directories, ¿a qué archivos puede referirse `Path("journal.jsonl")`?

### Modifica

Haz que el ejemplo temporal cree `data/archive` con `parents=True` y comprueba sus parents.

### Explica

¿Por qué escribir junto a `__file__` puede fallar en un package instalado?

### Comprueba

Imprime `Path.cwd()` desde la raíz del proyecto y desde otro directory. No fijes el output como contrato.

---

## 8. Abrir archivos, modos y encoding

### 8.1 File handle y modos

`open(...)` o `Path.open(...)` devuelve un file object que representa acceso a un recurso del OS.

| Modo | Contrato principal |
|---|---|
| `"r"` | leer; falla si el archivo no existe |
| `"w"` | escribir; crea o trunca el destino |
| `"a"` | escribir al final; crea si falta |
| `"b"` | modo binario, combinado como `"rb"` o `"wb"` |

`"w"` no significa “actualizar con cuidado”: trunca al abrir. `"a"` no vuelve transaccional ni seguro un journal concurrente.

### 8.2 Texto UTF-8 explícito

**Ejemplo ejecutable:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as directory:
    path = Path(directory) / "event.txt"

    with path.open("w", encoding="utf-8") as file:
        file.write("Llegué a casa 🏠")

    with path.open("r", encoding="utf-8") as file:
        text = file.read()

    assert text == "Llegué a casa 🏠"

print("UTF-8 round trip: PASS")
```

Output:

```text
UTF-8 round trip: PASS
```

El encoding default depende del entorno. Para datos textuales persistidos por EIDOLON, `encoding="utf-8"` forma parte del contrato.

### 8.3 Failure case: encoding implícito

**Código incorrecto:**

```python
with path.open("w") as file:
    file.write("Canción 🎵")
```

Qué ocurrió: el archivo depende del encoding default.  
Por qué importa: otro environment puede usar un encoding diferente o no representar el emoji.  
Evidencia: encoding elegido, bytes originales y exception de codec si aparece.  
Capa adecuada: la frontera de I/O declara UTF-8.

No afirmes que siempre fallará: en muchos systems modernos el default ya es UTF-8. El defecto es que el contrato quedó ambiental.

### 8.4 Texto y bytes

```text
str ──encode("utf-8")──▶ bytes
bytes ──decode("utf-8")──▶ str
```

**Ejemplo ejecutable:**

```python
text = "México 🌎"
payload = text.encode("utf-8")
restored = payload.decode("utf-8")

assert isinstance(payload, bytes)
assert restored == text
print(len(text), len(payload))
```

Output:

```text
8 12
```

La cantidad de Unicode code points y bytes no tiene por qué coincidir.

### Predice

¿Abrir `"w"` preserva el contenido anterior? ¿Qué hace `"a"`?

### Round trip

Cambia el texto por acentos, ñ y dos emojis. Comprueba igualdad, no longitud fija.

### Detecta el bug

¿Por qué un archivo que funciona en tu laptop no demuestra que omitir encoding sea portable?

---

## 9. Lifecycle del recurso y `with`

### 9.1 Tres cosas relacionadas, no idénticas

```text
objeto Python file
      ↕
file descriptor / handle del OS
      ↕
contenido administrado por filesystem/storage
```

Perder la última referencia Python no es un contrato de cierre oportuno y garbage collection no sustituye cleanup explícito.

### 9.2 Abrir → usar → cerrar

**Código frágil:**

```python
file = path.open("r", encoding="utf-8")
text = file.read()
file.close()
```

Si `read` falla, `close` no se alcanza.

**Frontera preferida:**

```python
with path.open("r", encoding="utf-8") as file:
    text = file.read()

# El file handle ya se cerró; text es un str independiente.
```

`with` delimita dónde el handle está disponible. No enseña todavía cómo implementar el context manager.

### 9.3 Failure case: usar el recurso cerrado

**Failure case ejecutable:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as directory:
    path = Path(directory) / "event.txt"
    path.write_text("evento", encoding="utf-8")

    with path.open("r", encoding="utf-8") as file:
        first = file.read()

    assert first == "evento"
    file.read()
```

La última línea termina con `ValueError` porque la operación usa un file cerrado. El string `first` sí puede utilizarse: ya no depende del recurso abierto.

### 9.4 Buffer, flush y close

- `file.write` puede escribir primero en un buffer del proceso;
- `file.flush()` solicita pasar ese buffer a capas inferiores;
- `close()` hace flush del buffer Python y libera el handle;
- nada de eso promete por sí solo persistencia física ante cualquier crash;
- `os.fsync(file.fileno())` solicita sincronización al OS, pero las garantías finales dependen de filesystem, hardware y metadata.

CS Foundations profundizará atomicidad y crash consistency.

### Predice

¿El string leído dentro de `with` desaparece al cerrar el file? ¿Y el handle puede seguir leyendo?

### Explica

¿Por qué garbage collection no es una política suficiente para un recurso externo?

### Detecta el bug

En el código frágil, ¿qué ruta evita `close`?

### Modifica

Escribe dos líneas dentro de `with` y comprueba el contenido solo después del cierre.

---

## 10. Lectura completa y procesamiento línea por línea

### 10.1 Materializar todo

```python
with path.open("r", encoding="utf-8") as file:
    text = file.read()
```

Es apropiado para un archivo pequeño cuando la operación necesita el texto completo. Memoria aproximada crece con el contenido materializado.

### 10.2 Iterar mientras el resource está abierto

**Ejemplo ejecutable:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as directory:
    path = Path(directory) / "journal.jsonl"
    path.write_text("uno\ndos\ntres\n", encoding="utf-8")

    seen: list[tuple[int, str]] = []
    with path.open("r", encoding="utf-8") as file:
        for line_number, raw_line in enumerate(file, start=1):
            seen.append((line_number, raw_line.rstrip("\n")))

    assert seen == [(1, "uno"), (2, "dos"), (3, "tres")]

print("streamed lines: PASS")
```

Output:

```text
streamed lines: PASS
```

El file es iterable y produce líneas de forma perezosa. El `with` sigue abierto durante el loop. No conserves el iterator para consumirlo fuera de la frontera.

### 10.3 Tradeoff

Línea por línea reduce memoria, localiza fallos y permite replay incremental, pero los errores aparecen al alcanzar la línea y el owner debe mantener el resource abierto. Leer todo simplifica operaciones globales y cierra pronto, pero materializa el contenido completo.

### Predice

¿Cuándo aparece un error ubicado en la línea 10 000 si consumes línea por línea?

### Explica

¿Qué objeto posee el file handle durante el loop?

### Decide

Para un journal sintético de diez líneas y para uno de diez millones, justifica estrategias diferentes.

---

## 11. JSON: un formato interoperable, no una database

### 11.1 Qué problema resuelve

Un dict Python existe dentro de un proceso. JSON define texto con una sintaxis compartida por distintos lenguajes y herramientas.

| Python | JSON |
|---|---|
| `dict` con keys string | object |
| `list` y tuple serializable | array |
| `str` | string |
| `int` / `float` finitos | number |
| `True` / `False` | `true` / `false` |
| `None` | `null` |

La tabla es una correspondencia práctica, no identidad de tipos. JSON no conserva tuple frente a list, Enum, dataclass ni `datetime`.

### 11.2 `dumps` y `loads` transforman values y text

**Ejemplo ejecutable:**

```python
import json


record = {
    "schema_version": 1,
    "event_id": "evt-001",
    "text": "Llegué a casa 🏠",
    "active": True,
    "tags": ["home", "arrival"],
    "note": None,
}

text = json.dumps(record, ensure_ascii=False, sort_keys=True)
restored = json.loads(text)

assert isinstance(text, str)
assert restored == record
print(text)
```

Output:

```text
{"active": true, "event_id": "evt-001", "note": null, "schema_version": 1, "tags": ["home", "arrival"], "text": "Llegué a casa 🏠"}
```

`ensure_ascii=False` conserva caracteres Unicode legibles en el texto; el archivo todavía debe escribirse como UTF-8. `sort_keys=True` se usa aquí para un output determinista, no como requisito universal del journal.

### 11.3 Indentación tiene un propósito

`indent=2` mejora una exportación destinada a lectura humana, pero agrega bytes y múltiples líneas. En JSONL cada record debe ocupar una sola línea, por lo que no se usa indent.

### 11.4 Malformed JSON

**Failure case ejecutable:**

```python
import json


json.loads('{"event_id": "evt-001",}')
```

Termina con `json.JSONDecodeError`. La posición, línea y columna están en la exception; el texto exacto del mensaje puede variar.

Qué ocurrió: existe una coma inválida antes de `}`.  
Evidencia: input original, posición y tipo de parser error.  
Capa adecuada: parsing reporta; la política exterior decide fail fast o quarantine.

### Predice

¿Qué tipo Python reconstruye un JSON array? ¿Se preserva tuple?

### Round trip

Agrega `"city": "México"` y comprueba igualdad después de `dumps`/`loads`.

### Detecta el bug

¿Por qué usar `indent=2` al escribir JSONL rompería “una línea = un record”?

---

## 12. `dump`, `load` y la frontera filesystem

### 12.1 Separar transformación de persistencia

```text
json.dumps / json.loads → values ↔ text
json.dump / json.load   → values ↔ text file object
```

**Ejemplo ejecutable:**

```python
import json
from pathlib import Path
from tempfile import TemporaryDirectory


record = {"schema_version": 1, "text": "Canción 🎵"}

with TemporaryDirectory() as directory:
    path = Path(directory) / "record.json"

    with path.open("w", encoding="utf-8") as file:
        json.dump(record, file, ensure_ascii=False)

    with path.open("r", encoding="utf-8") as file:
        restored = json.load(file)

    assert restored == record

print("JSON file round trip: PASS")
```

Output:

```text
JSON file round trip: PASS
```

`json.dump` no abre ni cierra el archivo: el caller posee ese lifecycle. `json.load` parsea desde un text file object.

### 12.2 Serialización no es persistencia

```text
Event
  ↓ event_to_record
record JSON-safe
  ↓ json.dumps
JSON text
  ↓ file write
persistencia en filesystem
```

- serialización define una representación;
- filesystem I/O coloca bytes bajo un path;
- persistencia incluye lifecycle, fallos, reemplazo, compatibilidad y recuperación.

### Predice

¿Quién cierra el file object del ejemplo: `json.dump` o el bloque `with`?

### Explica

¿Qué prueba `json.dumps(record)` que todavía no prueba `path.write_text(...)`?

### Modifica

Usa `json.dumps` y `Path.write_text` para obtener el mismo round trip. Declara UTF-8 en ambas fronteras.

---

## 13. JSON no conserva todos los tipos Python

### 13.1 Failure cases reales

**Failure case ejecutable — `datetime`:**

```python
import json
from datetime import UTC, datetime


json.dumps({"valid_time": datetime(2026, 8, 26, 18, 30, tzinfo=UTC)})
```

Termina con `TypeError`: el encoder default no conoce `datetime`.

**Failure case ejecutable — `Decimal`:**

```python
import json
from decimal import Decimal


json.dumps({"amount": Decimal("0.10")})
```

Termina con `TypeError`.

Enum, dataclass y set también necesitan política explícita. Convertir “todo lo desconocido” mediante `default=str` puede producir texto, pero oculta el contrato, pierde estructura y acepta accidentalmente tipos no previstos.

### 13.2 Dataclass y `asdict()`

**Failure case ejecutable:**

```python
import json
from dataclasses import asdict, dataclass
from datetime import UTC, datetime
from enum import Enum


class EventStatus(Enum):
    ACTIVE = "active"


@dataclass(frozen=True)
class Event:
    event_id: str
    valid_time: datetime
    status: EventStatus


event = Event(
    "evt-001",
    datetime(2026, 8, 26, 18, 30, tzinfo=UTC),
    EventStatus.ACTIVE,
)

json.dumps(asdict(event))
```

`asdict` convierte la dataclass y nested dataclasses en containers, pero deja `datetime` y Enum como objetos de esos tipos. El encoder termina con `TypeError`.

Incluso cuando `asdict` produce algo serializable, no decide:

- nombres persistentes de fields;
- `schema_version`;
- política temporal;
- compatibilidad futura;
- qué fields internos no pertenecen al contrato;
- cómo reconstruir el domain object.

### 13.3 Set, tuple y semántica

JSON representa arrays, no set ni tuple. Convertir set a list exige decidir orden si el output debe ser determinista. Al reconstruir, el decoder no sabe si el array original era `list`, `tuple` o `set`.

### Predice

¿Qué objeto del failure case impide primero la serialización? ¿Por qué resolver solo ese objeto no define todavía un schema?

### Explica

¿Qué decisiones faltan después de `asdict(event)`?

### Repara

Escribe un dict explícito que convierta status a `.value` y `valid_time` a texto ISO. No uses `default=str`.

---

## 14. Dominio y representación persistida son contratos distintos

### 14.1 La frontera explícita

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass
from datetime import UTC, datetime


@dataclass(frozen=True)
class SourceRef:
    source_id: str


@dataclass(frozen=True)
class Event:
    event_id: str
    text: str
    valid_time: datetime
    source: SourceRef


def event_to_record(event: Event) -> dict[str, object]:
    return {
        "schema_version": 1,
        "record_type": "event",
        "event_id": event.event_id,
        "text": event.text,
        "valid_time": event.valid_time.isoformat(),
        "source_id": event.source.source_id,
    }


event = Event(
    event_id="evt-001",
    text="Llegué a casa",
    valid_time=datetime(2026, 8, 26, 18, 30, tzinfo=UTC),
    source=SourceRef("src-001"),
)

record = event_to_record(event)
print(record["valid_time"])
```

Output:

```text
2026-08-26T18:30:00+00:00
```

El domain object compone `SourceRef`; el record persiste `source_id` plano por decisión de schema. Cambiar la class no obliga automáticamente a cambiar el archivo, y viceversa.

### 14.2 Reconstrucción explícita

**Continuación:**

```python
def require_string(record: dict[str, object], key: str) -> str:
    value = record.get(key)
    if not isinstance(value, str) or not value:
        raise ValueError(f"{key} must be a non-empty string")
    return value


def event_from_record(record: dict[str, object]) -> Event:
    if record.get("schema_version") != 1:
        raise ValueError("unsupported schema version")
    if record.get("record_type") != "event":
        raise ValueError("expected event record")

    valid_time_text = require_string(record, "valid_time")
    valid_time = datetime.fromisoformat(valid_time_text)
    if valid_time.tzinfo is None or valid_time.utcoffset() is None:
        raise ValueError("valid_time must be timezone-aware")

    return Event(
        event_id=require_string(record, "event_id"),
        text=require_string(record, "text"),
        valid_time=valid_time,
        source=SourceRef(require_string(record, "source_id")),
    )


restored = event_from_record(event_to_record(event))
assert restored == event
print("Event round trip: PASS")
```

Output:

```text
Event round trip: PASS
```

`datetime.fromisoformat` puede producir naive si el texto no tiene offset; por eso la validación sigue siendo necesaria. La dataclass tampoco valida automáticamente un record externo.

### 14.3 Round trip semántico

El objetivo no siempre es reproducir los mismos objetos de runtime, sino conservar el significado definido:

```text
domain object → record → JSON → record → domain object
                      semántica conservada
```

Comprueba fields, tipos reconstruidos, timezone awareness e invariantes. No compares solo que “se imprimen parecido”.

### Predice

¿`restored is event` debería ser verdadero? ¿Y `restored == event`?

### Diseña el contrato

¿Debe `source` persistirse anidado o como `source_id`? Defiende una forma; no afirmes que la dataclass decide.

### Round trip

Cambia el texto a Unicode y el offset temporal a `-06:00`. Comprueba que el instante y el offset se reconstruyen según el contrato.

---

## 15. Tiempo y números exactos en JSON

### 15.1 `datetime` aware e ISO 8601 práctico

**Ejemplo ejecutable:**

```python
from datetime import UTC, datetime


original = datetime(2026, 8, 26, 18, 30, 45, 123456, tzinfo=UTC)
encoded = original.isoformat()
restored = datetime.fromisoformat(encoded)

assert restored == original
assert restored.tzinfo is not None
assert restored.utcoffset() is not None
print(encoded)
```

Output:

```text
2026-08-26T18:30:45.123456+00:00
```

Un contrato puede normalizar a UTC antes de serializar o conservar el offset original. Debe decidirlo explícitamente. No agregues `Z` con string replacement sin comprobar qué parser y contrato lo consumirán; `+00:00` es inequívoco y round-trips con `fromisoformat`.

`valid_time` y `recorded_at` responden preguntas distintas. PF-M6 conserva ambos si el modelo los incluye; no los colapsa en “timestamp”.

### 15.2 `Decimal` como texto deliberado

**Ejemplo ejecutable:**

```python
import json
from decimal import Decimal


amount = Decimal("0.10")
record = {"amount_decimal": str(amount)}
restored = Decimal(json.loads(json.dumps(record))["amount_decimal"])

assert restored == amount
assert str(restored) == "0.10"
print(restored)
```

Output:

```text
0.10
```

Persistir como string comunica que el consumer debe reconstruir Decimal y conserva escala textual. Otra API podría usar JSON number y `json.loads(..., parse_float=Decimal)`, pero esa política debe abarcar productores y consumers.

### 15.3 Failure case: convertir a float

**Código incorrecto:**

```python
from decimal import Decimal


amount = Decimal("0.10")
persisted = float(amount)
restored = Decimal(persisted)
print(restored)
```

El resultado contiene la aproximación binaria exacta del float, no necesariamente `Decimal("0.10")`.  
Evidencia: valor Decimal original, representación persistida y conversión aplicada.  
Capa adecuada: serialization decide una representación que conserve la semántica.

### Predice

¿Un type hint `valid_time: datetime` garantiza timezone awareness al deserializar?

### Explica

¿Por qué string puede ser una mejor representación contractual de Decimal que float?

### Modifica

Agrega `recorded_at` al record de Event y comprueba round trip independiente de `valid_time`.

---

## 16. Schema y `schema_version`

### 16.1 La forma persistida es una API

```json
{
  "schema_version": 1,
  "record_type": "event",
  "event_id": "evt-001",
  "text": "Llegué a casa"
}
```

Keys, tipos, valores permitidos y semántica forman un contrato. JSON válido sintácticamente todavía puede violarlo.

### 16.2 Versión de schema no es versión de aplicación

- application version identifica una release del programa;
- schema version identifica la forma y semántica del record;
- una release puede leer varias schema versions;
- dos releases podrían producir el mismo schema;
- cambiar comportamiento interno no exige necesariamente migrar datos.

### 16.3 Excepción específica para versión desconocida

**Ejemplo ejecutable:**

```python
class UnsupportedSchemaVersionError(Exception):
    pass


def require_schema_version(record: dict[str, object]) -> int:
    version = record.get("schema_version")
    if type(version) is not int:
        raise ValueError("schema_version must be an int")
    if version != 1:
        raise UnsupportedSchemaVersionError(
            f"unsupported schema version: {version}"
        )
    return version


print(require_schema_version({"schema_version": 1}))
```

Output:

```text
1
```

**Failure case:**

```python
require_schema_version({"schema_version": 99})
```

Debe terminar con `UnsupportedSchemaVersionError`. No se interpreta 99 como la versión “más cercana”.

### 16.4 Rechazar puede ser más seguro que adivinar

Una versión desconocida puede incluir keys con otra semántica. Fail fast conserva el record para una herramienta que sí conozca la migración. El mensaje no necesita imprimir todo el payload.

### Predice

¿JSON sintácticamente válido implica schema compatible?

### Explica

¿Por qué actualizar la aplicación de 0.0a a 0.0b no obliga por sí solo a incrementar `schema_version`?

### Detecta el bug

¿Qué riesgo crea `record["schema_version"] = 1` al cargar cualquier versión?

---

## 17. JSONL: un record JSON por línea

### 17.1 Modelo mental

```text
línea 1 → objeto JSON
línea 2 → objeto JSON
línea 3 → objeto JSON
```

Cada línea completa es JSON independiente. Un newline dentro de un string se representa escapado como `\n`, no como un salto físico adicional producido por indentación.

JSONL favorece:

- append de nuevos records;
- procesamiento incremental;
- replay en orden de archivo;
- ubicación de corrupción por línea;
- datasets que no necesitan parsearse como un array único.

No ofrece:

- transactions reales entre múltiples records;
- constraints relacionales;
- queries sofisticadas o índices persistentes;
- coordinación segura de writers concurrentes;
- reemplazo para PostgreSQL cuando aparezcan esas necesidades.

### 17.2 Append mínimo

**Ejemplo ejecutable:**

```python
import json
from pathlib import Path
from tempfile import TemporaryDirectory


def append_record(path: Path, record: dict[str, object]) -> None:
    line = json.dumps(
        record,
        ensure_ascii=False,
        separators=(",", ":"),
    )
    with path.open("a", encoding="utf-8", newline="\n") as file:
        file.write(line)
        file.write("\n")


with TemporaryDirectory() as directory:
    path = Path(directory) / "journal.jsonl"
    append_record(path, {"schema_version": 1, "event_id": "evt-001"})
    append_record(path, {"schema_version": 1, "event_id": "evt-002"})

    lines = path.read_text(encoding="utf-8").splitlines()
    assert len(lines) == 2
    assert json.loads(lines[1])["event_id"] == "evt-002"

print("JSONL append: PASS")
```

Output:

```text
JSONL append: PASS
```

`newline="\n"` hace explícito el delimitador lógico escrito. El journal asume un único writer educativo. No afirma que múltiples llamadas concurrentes formen una transaction.

### 17.3 Leer con número de línea

**Ejemplo ejecutable:**

```python
import json
from json import JSONDecodeError
from pathlib import Path
from tempfile import TemporaryDirectory


class JournalCorruptionError(Exception):
    pass


def load_records(path: Path) -> list[dict[str, object]]:
    records: list[dict[str, object]] = []

    with path.open("r", encoding="utf-8") as file:
        for line_number, raw_line in enumerate(file, start=1):
            try:
                value = json.loads(raw_line)
            except JSONDecodeError as exc:
                raise JournalCorruptionError(
                    f"invalid JSON at line {line_number}"
                ) from exc

            if not isinstance(value, dict):
                raise JournalCorruptionError(
                    f"expected object at line {line_number}"
                )
            records.append(value)

    return records


with TemporaryDirectory() as directory:
    path = Path(directory) / "journal.jsonl"
    path.write_text(
        '{"event_id":"evt-001"}\n{"event_id":"evt-002"}\n',
        encoding="utf-8",
    )
    assert len(load_records(path)) == 2

print("JSONL load: PASS")
```

Output:

```text
JSONL load: PASS
```

Esta función itera línea por línea pero devuelve una list porque el ejemplo P0 es pequeño. Un replay grande puede procesar cada record dentro del `with` sin conservarlos todos; no debe devolver un iterator ligado a un file ya cerrado.

### Predice

¿Qué número reportará una tercera línea corrupta?

### Modifica

Agrega un record con Unicode y verifica que el archivo sigue teniendo una línea por record.

### Explica

¿Qué propiedad de JSONL ayuda a localizar corrupción y cuál no resuelve concurrencia?

### 17.4 Checksum de integridad: detectar, no autenticar

EIDOLON 0.0a también debe detectar un record cuyo contenido ya no coincide con el checksum registrado. El checksum se calcula sobre una representación canónica del record **sin** el propio field `checksum`; de otro modo el cálculo sería recursivo. `sort_keys=True` y separators explícitos evitan que distinto orden o whitespace cambien el input del hash.

**Ejemplo ejecutable:**

```python
import hashlib
import json


class JournalChecksumError(Exception):
    pass


def canonical_record_bytes(record: dict[str, object]) -> bytes:
    return json.dumps(
        record,
        ensure_ascii=False,
        sort_keys=True,
        separators=(",", ":"),
    ).encode("utf-8")


def seal_record(record: dict[str, object]) -> dict[str, object]:
    if "checksum" in record:
        raise ValueError("record already contains checksum")
    sealed = dict(record)
    sealed["checksum"] = hashlib.sha256(
        canonical_record_bytes(record)
    ).hexdigest()
    return sealed


def verify_record(record: dict[str, object]) -> dict[str, object]:
    payload = dict(record)
    expected = payload.pop("checksum", None)
    if not isinstance(expected, str):
        raise JournalChecksumError("missing checksum")

    actual = hashlib.sha256(canonical_record_bytes(payload)).hexdigest()
    if actual != expected:
        raise JournalChecksumError("checksum mismatch")
    return payload


sealed = seal_record(
    {
        "schema_version": 2,
        "record_type": "event",
        "event_id": "evt-001",
        "text": "Llegué a casa 🏠",
    }
)
assert verify_record(sealed)["event_id"] == "evt-001"

tampered = dict(sealed)
tampered["text"] = "texto cambiado"
try:
    verify_record(tampered)
except JournalChecksumError:
    print("checksum mismatch detected")
```

Output:

```text
checksum mismatch detected
```

SHA-256 aquí detecta cambios accidentales bajo este contrato; **no autentica** el origen. Quien pueda cambiar el record y recalcular el checksum puede producir otra pareja válida. Firmas, keys y threat modeling profundo pertenecen a D12. El loader debe reportar ubicación y conservar source/quarantine igual que ante JSON o schema inválido.

### Predice

¿Cambiar solo el orden de keys antes de `seal_record` cambia el checksum? Explica el papel de `sort_keys=True`.

### Detecta el bug

Una implementación calcula el hash después de agregar `checksum` y luego intenta verificar incluyendo ese field. Identifica el contrato circular.

---

## 18. Corrupción localizada, fail fast y quarantine

### 18.1 Failure case reproducible

**Failure case ejecutable — usa `load_records` anterior:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as directory:
    path = Path(directory) / "journal.jsonl"
    path.write_text(
        '{"event_id":"evt-001"}\n'
        '{"event_id":\n'
        '{"event_id":"evt-003"}\n',
        encoding="utf-8",
    )
    load_records(path)
```

Debe terminar con `JournalCorruptionError: invalid JSON at line 2`, cuya causa es `JSONDecodeError`. La línea 3 no se procesa bajo política fail fast.

Qué ocurrió: una línea no forma JSON completo.  
Por qué: puede existir edición manual, truncamiento o escritura parcial.  
Evidencia: source intacto, número de línea, causa, path y bytes/texto bajo acceso controlado.  
Capa adecuada: parser localiza; application policy decide detener o quarantine.

### 18.2 Tres políticas

| Política | Comportamiento | Tradeoff |
|---|---|---|
| fail fast | detiene al primer record inválido | no produce una vista incompleta fingiendo éxito |
| quarantine | preserva la línea no interpretada aparte y emite receipt | permite continuar, pero exige que la ausencia quede visible |
| reporte | reúne ubicaciones sin mutar source | puede requerir recorrer más y definir qué errores son recuperables |

Para EIDOLON se prioriza preservation + evidence. “Saltar la línea y seguir” sin receipt convierte corrupción en olvido.

### 18.3 Receipt no sensible

Un receipt mínimo puede contener:

```json
{
  "source_name": "journal.jsonl",
  "line_number": 27,
  "error_type": "JSONDecodeError",
  "action": "quarantined"
}
```

Evita imprimir indiscriminadamente el payload privado. El contenido bruto, si debe conservarse, va a un artefacto de quarantine con acceso y lifecycle definidos; el receipt identifica sin duplicarlo.

### 18.4 Blank lines requieren política

`json.loads("")` falla. Puedes declarar blank line como corrupción o permitirla explícitamente, pero no debe desaparecer por un `if not raw_line.strip(): continue` sin documentar la decisión.

### Predice

¿Fail fast devuelve los records anteriores como si la carga hubiera sido exitosa?

### Diseña el contrato

Elige política para una CLI `verify` y otra para `replay`. ¿Deben ser iguales?

### Repara

Diseña un receipt que conserve ubicación y causa sin copiar el payload completo.

---

## 19. Append, overwrite y escritura parcial

### 19.1 Failure case: `"w"` destruye el contenido anterior

**Failure case ejecutable:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as directory:
    path = Path(directory) / "journal.jsonl"
    path.write_text('{"event_id":"evt-001"}\n', encoding="utf-8")

    with path.open("w", encoding="utf-8") as file:
        file.write('{"event_id":"evt-002"}\n')

    print(path.read_text(encoding="utf-8").strip())
```

Output:

```text
{"event_id":"evt-002"}
```

Qué ocurrió: abrir en `"w"` truncó antes de escribir.  
Evidencia perdida: source anterior, salvo backup externo.  
Capa adecuada: journal append usa `"a"`; export/migration escribe otro destino de forma segura.

### 19.2 `"a"` tampoco garantiza todo

Append conserva bytes anteriores en el caso normal de un solo writer, pero:

- un proceso puede fallar a mitad de línea;
- dos writers pueden interferir según OS/filesystem y tamaño;
- varios records no forman una transaction;
- close no garantiza durabilidad física absoluta.

PF-M6 limita el ejemplo a un writer y detecta una última línea truncada durante replay.

### 19.3 `try/except` no vuelve atómica una escritura

```text
abrir destino
↓
escribir una parte
↓
fallo
↓
destino incompleto
```

Capturar `OSError` puede reportar el fallo, pero no restaura automáticamente los bytes anteriores.

**Código incorrecto:**

```python
try:
    destination.write_text(new_text, encoding="utf-8")
except OSError:
    print("write failed")
```

El destino pudo truncarse antes del error. “Se capturó” no significa “se recuperó”.

### Predice

¿En qué momento `"w"` destruye el contenido: al primer `write` o al abrir?

### Detecta el bug

¿Por qué un `except` alrededor de `write_text` no implementa rollback?

### Decide

¿Usarías append para modificar una línea anterior? Explica por qué Correction agrega otro record.

---

## 20. Archivo temporal + replace: atomicidad práctica

### 20.1 Patrón

```text
datos nuevos
↓
temporal en el mismo directory
↓
escritura completa
↓
flush / close
↓
validación
↓
replace del destino
```

El temporal se crea junto al destino para evitar cruzar filesystems durante replace.

### 20.2 Implementación con standard library

**Ejemplo ejecutable:**

```python
import os
from pathlib import Path
from tempfile import NamedTemporaryFile, TemporaryDirectory


def write_text_safely(destination: Path, text: str) -> None:
    destination.parent.mkdir(parents=True, exist_ok=True)
    temporary_path: Path | None = None

    try:
        with NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            dir=destination.parent,
            prefix=f".{destination.name}.",
            suffix=".tmp",
            delete=False,
        ) as file:
            temporary_path = Path(file.name)
            file.write(text)
            file.flush()
            os.fsync(file.fileno())

        temporary_path.replace(destination)
    finally:
        if temporary_path is not None and temporary_path.exists():
            temporary_path.unlink()


with TemporaryDirectory() as directory:
    destination = Path(directory) / "export.json"
    write_text_safely(destination, '{"status":"válido"}\n')
    assert destination.read_text(encoding="utf-8") == '{"status":"válido"}\n'

print("safe replace: PASS")
```

Output:

```text
safe replace: PASS
```

El file temporal se cierra antes de `replace`, lo cual también evita conflictos de handles abiertos en Windows. `finally` limpia un temporal restante sin ocultar la exception original; si cleanup también falla, una implementación de producción necesitaría una política explícita y evidencia adicional.

### 20.3 Qué garantiza y qué no

`Path.replace` usa la operación de reemplazo del OS. En filesystems locales comunes y dentro del mismo filesystem, reduce la ventana en que el destino tendría contenido parcial: los readers suelen observar el destino anterior o el nuevo.

No promete:

- durabilidad absoluta ante power loss;
- que directory metadata ya esté físicamente persistida;
- atomicidad en cualquier filesystem remoto o no convencional;
- transacción de varios archivos;
- ausencia de races entre writers;
- recuperación ante cualquier fallo de hardware.

`fsync` del file solicita persistir sus datos antes del replace; no cubre por sí solo todas las capas ni el directory. La formulación correcta es “reduce destinos parcialmente reemplazados bajo las garantías del OS/filesystem”, no “nunca se pierden datos”.

### 20.4 Failure injection conceptual

Si una función falla antes de `replace`:

- el destino anterior debe permanecer;
- el temporal debe limpiarse o reportarse;
- la exception debe propagarse;
- no debe anunciarse éxito.

PF-L10 diseñará el context manager propio y una inyección de fallo más rigurosa. PF-M6 usa la función explícita anterior.

### Predice

Si `file.write` falla antes de `replace`, ¿qué contenido conserva un destino preexistente?

### Explica

¿Por qué el temporal debe residir en el mismo filesystem?

### Detecta la promesa falsa

Corrige: “`fsync` + `replace` garantiza que ningún crash ni fallo de hardware puede perder datos”.

### Modifica

Escribe primero `"old"` en el destino y después promueve `"new"`. Comprueba solo el resultado normal; el lab posterior inyectará el fallo.

---

## 21. Source data, derived data y migración

### 21.1 Dos roles que no deben confundirse

```text
source original
      ↓ parse
validated / derived representation
```

Source data conserva lo recibido o registrado bajo su contrato. Derived data es una interpretación, normalización, índice, exportación o migración producida desde esa fuente.

Si una transformación:

- normaliza texto;
- cambia keys;
- agrega defaults;
- convierte una versión;
- descarta una línea;

su output no se vuelve source original por tener una forma más cómoda.

### 21.2 Failure case: migración destructiva

**Código incorrecto:**

```python
records = load_records(source_path)
migrated = [migrate_v1_to_v2(record) for record in records]
write_records(source_path, migrated)
```

Qué ocurrió: el único source se reemplaza.  
Por qué importa: un bug de migración, record omitido o nueva semántica ya no puede compararse con el original.  
Evidencia perdida: bytes, orden, versión y errores originales.  
Capa adecuada: la application escribe un destino distinto, valida y solo después decide promoción bajo una política explícita.

### 21.3 Migración pura primero

**Ejemplo ejecutable:**

```python
class UnsupportedSchemaVersionError(Exception):
    pass


def require_non_empty_string(
    record: dict[str, object],
    key: str,
) -> str:
    value = record.get(key)
    if not isinstance(value, str) or not value:
        raise ValueError(f"{key} must be a non-empty string")
    return value


def migrate_event_v1_to_v2(
    source_record: dict[str, object],
) -> dict[str, object]:
    if source_record.get("schema_version") != 1:
        raise UnsupportedSchemaVersionError("expected schema version 1")
    if source_record.get("record_type") != "event":
        raise ValueError("expected event record")

    return {
        "schema_version": 2,
        "record_type": "event",
        "event_id": require_non_empty_string(source_record, "id"),
        "text": require_non_empty_string(source_record, "text"),
        "valid_time": require_non_empty_string(
            source_record,
            "happened_at",
        ),
        "recorded_at": require_non_empty_string(
            source_record,
            "recorded_at",
        ),
        "source_id": require_non_empty_string(source_record, "source_id"),
        "status": "active",
        "tags": [],
    }


v1 = {
    "schema_version": 1,
    "record_type": "event",
    "id": "evt-001",
    "text": "Llegué a casa",
    "happened_at": "2026-08-26T18:30:00+00:00",
    "recorded_at": "2026-08-26T18:31:00+00:00",
    "source_id": "src-001",
}

v2 = migrate_event_v1_to_v2(v1)

assert v1["schema_version"] == 1
assert "id" in v1
assert v2["schema_version"] == 2
assert v2["event_id"] == "evt-001"
print("pure migration: PASS")
```

Output:

```text
pure migration: PASS
```

La function no abre archivos ni muta `source_record`. Puede probarse con distintos records antes de diseñar la promoción.

### 21.4 Pipeline seguro de migración

```text
leer source
↓
parsear y validar con línea
↓
migrar mediante pure functions
↓
escribir temporal del destino
↓
releer y validar temporal
↓
promover al destination
↓
conservar source
```

“Destino” no debe ser el único source. Una política posterior puede archivar o reemplazar una copia después de validación y rollback preparado; PF-M6 no borra el original.

### Predice

Después de `migrate_event_v1_to_v2(v1)`, ¿qué keys conserva `v1`?

### Repara

Cambia el antipatrón para escribir `journal.v2.jsonl` sin tocar `journal.v1.jsonl`.

### Refactoriza

Separa una función que lee, migra y escribe en tres partes: I/O de entrada, pure migration e I/O de salida.

### Explica

¿Por qué un record migrado es derived data hasta que una política explícita lo promueve?

---

## 22. Taxonomía mínima, filesystem failures y datos sensibles

### 22.1 Cinco categorías útiles

| Categoría | Ejemplo | Acción posible |
|---|---|---|
| invalid input | ID vacío, key con tipo incorrecto | rechazar cerca de la frontera |
| domain conflict | duplicate ID, Correction sin target | reportar conflicto; no inventar resolución |
| serialization/parsing | malformed JSON, schema desconocido | localizar, preservar y migrar/rechazar |
| filesystem/I/O | no existe, permiso, storage error | recuperar solo si la política lo permite |
| programming bug | key equivocada interna, cálculo imposible | propagar y corregir código |

La clasificación no exige una class por fila. Sirve para decidir quién puede manejar el fallo y qué evidencia conservar.

### 22.2 `FileNotFoundError`

**Failure case ejecutable:**

```python
from pathlib import Path
from tempfile import TemporaryDirectory


with TemporaryDirectory() as directory:
    missing = Path(directory) / "missing.jsonl"
    missing.read_text(encoding="utf-8")
```

Termina con `FileNotFoundError`. Una command `init` podría tratar ausencia como señal para crear; una command `verify` sobre un path requerido debería fallar. El significado lo define el caller, no la exception sola.

### 22.3 `PermissionError` es ambiental

**Fragmento — dependencia ambiental:**

```python
try:
    text = path.read_text(encoding="utf-8")
except PermissionError as exc:
    raise JournalReadError(
        f"permission denied while reading {path.name}"
    ) from exc
```

Para reproducirlo se necesita un path existente que el proceso actual no pueda leer. ACLs, permisos POSIX, sandboxing, usuario elevado y Windows cambian cómo se crea ese escenario. No uses `/root` ni otro path sensible como ejercicio. En PF-L09 se prepara un directory temporal con permisos controlados o un fallo inyectado por el harness del lab.

La propiedad estable es capturar `PermissionError` cuando la operación real lo produce. Un pre-check `exists` no lo evita.

### 22.4 `OSError` como familia exterior

`FileNotFoundError` y `PermissionError` son casos de `OSError`. Otros fallos de filesystem pueden variar por plataforma. Captura `OSError` solo en una frontera que pueda añadir contexto o presentar un fallo técnico; no lo conviertas en journal vacío.

### 22.5 Mensajes accionables sin payload privado

Preferible:

```text
Invalid JSON at line 27 in journal.jsonl
```

Riesgoso:

```text
Invalid payload: {contenido autobiográfico completo...}
```

Un traceback técnico puede contener paths. Una UI o receipt público debe minimizar datos; el diagnóstico protegido conserva la causa y una referencia al source. PF-M6 introduce awareness, no diseña todavía el Security/Privacy Track completo.

### Clasifica

Ubica: key ausente obligatoria, duplicate Event ID, `JSONDecodeError`, `PermissionError` y `AttributeError` por typo interno.

### Explica

¿Por qué la misma `FileNotFoundError` puede ser recuperable en `init` y fatal en `verify`?

### Diseña el mensaje

Escribe un mensaje para línea corrupta que permita actuar sin copiar el payload.

---

## 23. Separación de responsabilidades

### 23.1 Flujo pequeño

```text
domain objects
      ↓
serialization functions
      ↓
journal functions
      ↓
filesystem
```

Responsabilidades:

- `model.py`: invariantes de Event, Claim, Correction y SourceRef;
- `serialization.py`: domain object ↔ JSON-safe record;
- `journal.py`: append, lectura con línea y escritura segura;
- una frontera exterior futura: elegir path, traducir errores y presentar resultado.

### 23.2 Mega-class que mezcla todo

**Código incorrecto — fragmento:**

```python
class JournalManager:
    def load_validate_migrate_save_and_print(self, path):
        text = path.read_text()
        records = json.loads(text)
        # valida dominio, modifica records, guarda, imprime y captura todo
```

Qué ocurrió: parsing, dominio, filesystem y UI comparten control flow.  
Impacto: no puedes probar migración sin escribir; una captura amplia oculta la capa; source y derived pueden confundirse.  
Corrección incremental: extrae primero pure serialization/migration, después ownership del file y finalmente presentación.

### 23.3 Functions pequeñas, no layers vacías

```python
event_to_record(event)
event_from_record(record)
append_record(path, record)
load_records(path)
```

Cada frontera tiene input, output, errores y efectos identificables. No hace falta una class si no posee estado o lifecycle. Tampoco se introduce repository persistente como arquitectura: el archivo JSONL es un adapter educativo transitorio.

### Refactoriza

Elige el primer bloque que extraerías de `JournalManager` y explica por qué reduce el riesgo.

### Decide

¿`event_to_record` debe imprimir un mensaje? ¿Qué consumer futuro se beneficiaría de mantenerla pura?

### Detecta el bug

Si `load_records` corrige keys y luego sobrescribe el mismo archivo, ¿qué dos responsabilidades y roles de datos mezcló?

---

## 24. Caso progresivo integrado: journal EIDOLON pequeño

### 24.1 Layout

```text
eidolon-journal/
├── pyproject.toml
├── src/
│   └── eidolon/
│       ├── __init__.py
│       ├── errors.py
│       ├── model.py
│       ├── serialization.py
│       └── journal.py
└── checks/
    └── smoke.py
```

Reutiliza package, venv y `pyproject.toml` de PF-M4. No agrega dependencias runtime.

### 24.2 `errors.py`

**Archivo completo:**

```python
class JournalError(Exception):
    pass


class RecordValidationError(JournalError):
    pass


class UnsupportedSchemaVersionError(JournalError):
    pass


class JournalCorruptionError(JournalError):
    pass


class JournalReadError(JournalError):
    pass


class JournalWriteError(JournalError):
    pass
```

La taxonomía permanece pequeña. `RecordValidationError` expresa forma persistida inválida; las dataclasses todavía usan `ValueError` para invariantes de dominio.

### 24.3 `model.py`

**Archivo completo:**

```python
from dataclasses import dataclass
from datetime import datetime
from enum import Enum


def require_aware(value: datetime, field_name: str) -> None:
    if not isinstance(value, datetime):
        raise ValueError(f"{field_name} must be a datetime")
    if value.tzinfo is None or value.utcoffset() is None:
        raise ValueError(f"{field_name} must be timezone-aware")


def require_text(value: str, field_name: str) -> None:
    if type(value) is not str or not value:
        raise ValueError(f"{field_name} must be a non-empty str")


@dataclass(frozen=True)
class SourceRef:
    source_id: str

    def __post_init__(self) -> None:
        require_text(self.source_id, "source_id")


class EventStatus(Enum):
    ACTIVE = "active"
    ARCHIVED = "archived"


class CorrectionTarget(Enum):
    EVENT = "event"
    CLAIM = "claim"


@dataclass(frozen=True)
class Event:
    event_id: str
    text: str
    valid_time: datetime
    recorded_at: datetime
    source: SourceRef
    status: EventStatus = EventStatus.ACTIVE
    tags: tuple[str, ...] = ()

    def __post_init__(self) -> None:
        require_text(self.event_id, "event_id")
        require_text(self.text, "text")
        require_aware(self.valid_time, "valid_time")
        require_aware(self.recorded_at, "recorded_at")
        if not isinstance(self.source, SourceRef):
            raise ValueError("source must be a SourceRef")
        if not isinstance(self.status, EventStatus):
            raise ValueError("status must be an EventStatus")
        if type(self.tags) is not tuple:
            raise ValueError("tags must be a tuple")
        if any(type(tag) is not str or not tag for tag in self.tags):
            raise ValueError("tags must be non-empty strings")


@dataclass(frozen=True)
class Claim:
    claim_id: str
    about_event_id: str
    statement: str
    recorded_at: datetime
    source: SourceRef

    def __post_init__(self) -> None:
        require_text(self.claim_id, "claim_id")
        require_text(self.about_event_id, "about_event_id")
        require_text(self.statement, "statement")
        require_aware(self.recorded_at, "recorded_at")
        if not isinstance(self.source, SourceRef):
            raise ValueError("source must be a SourceRef")


@dataclass(frozen=True)
class Correction:
    correction_id: str
    target_kind: CorrectionTarget
    target_id: str
    replacement_text: str
    recorded_at: datetime
    source: SourceRef

    def __post_init__(self) -> None:
        require_text(self.correction_id, "correction_id")
        require_text(self.target_id, "target_id")
        require_text(self.replacement_text, "replacement_text")
        require_aware(self.recorded_at, "recorded_at")
        if not isinstance(self.target_kind, CorrectionTarget):
            raise ValueError("target_kind must be a CorrectionTarget")
        if not isinstance(self.source, SourceRef):
            raise ValueError("source must be a SourceRef")


JournalEntry = Event | Claim | Correction
```

`recorded_at` conserva cuándo se registró cada entrada; `valid_time` conserva cuándo se afirma que ocurrió el Event. Correction sigue siendo un objeto nuevo.

### 24.4 `serialization.py`

**Archivo completo:**

```python
from datetime import datetime

from eidolon.errors import (
    RecordValidationError,
    UnsupportedSchemaVersionError,
)
from eidolon.model import (
    Claim,
    Correction,
    CorrectionTarget,
    Event,
    EventStatus,
    JournalEntry,
    SourceRef,
)


SCHEMA_VERSION = 2


def require_string(record: dict[str, object], key: str) -> str:
    value = record.get(key)
    if not isinstance(value, str) or not value:
        raise RecordValidationError(
            f"{key} must be a non-empty string"
        )
    return value


def require_aware_datetime(
    record: dict[str, object],
    key: str,
) -> datetime:
    text = require_string(record, key)
    try:
        value = datetime.fromisoformat(text)
    except ValueError as exc:
        raise RecordValidationError(
            f"{key} must be an ISO 8601 datetime"
        ) from exc
    if value.tzinfo is None or value.utcoffset() is None:
        raise RecordValidationError(
            f"{key} must be timezone-aware"
        )
    return value


def require_string_tuple(
    record: dict[str, object],
    key: str,
) -> tuple[str, ...]:
    value = record.get(key)
    if not isinstance(value, list):
        raise RecordValidationError(f"{key} must be a JSON array")
    if any(not isinstance(item, str) or not item for item in value):
        raise RecordValidationError(
            f"{key} must contain non-empty strings"
        )
    return tuple(value)


def require_current_schema(record: dict[str, object]) -> None:
    version = record.get("schema_version")
    if type(version) is not int:
        raise RecordValidationError("schema_version must be an int")
    if version != SCHEMA_VERSION:
        raise UnsupportedSchemaVersionError(
            f"unsupported schema version: {version}"
        )


def entry_to_record(entry: JournalEntry) -> dict[str, object]:
    common: dict[str, object] = {
        "schema_version": SCHEMA_VERSION,
        "recorded_at": entry.recorded_at.isoformat(),
        "source_id": entry.source.source_id,
    }

    if isinstance(entry, Event):
        return common | {
            "record_type": "event",
            "event_id": entry.event_id,
            "text": entry.text,
            "valid_time": entry.valid_time.isoformat(),
            "status": entry.status.value,
            "tags": list(entry.tags),
        }
    if isinstance(entry, Claim):
        return common | {
            "record_type": "claim",
            "claim_id": entry.claim_id,
            "about_event_id": entry.about_event_id,
            "statement": entry.statement,
        }
    if isinstance(entry, Correction):
        return common | {
            "record_type": "correction",
            "correction_id": entry.correction_id,
            "target_kind": entry.target_kind.value,
            "target_id": entry.target_id,
            "replacement_text": entry.replacement_text,
        }
    raise TypeError(f"unsupported entry type: {type(entry).__name__}")


def entry_from_record(record: dict[str, object]) -> JournalEntry:
    require_current_schema(record)
    record_type = require_string(record, "record_type")
    source = SourceRef(require_string(record, "source_id"))
    recorded_at = require_aware_datetime(record, "recorded_at")

    if record_type == "event":
        status_text = require_string(record, "status")
        try:
            status = EventStatus(status_text)
        except ValueError as exc:
            raise RecordValidationError(
                f"unknown event status: {status_text}"
            ) from exc
        return Event(
            event_id=require_string(record, "event_id"),
            text=require_string(record, "text"),
            valid_time=require_aware_datetime(record, "valid_time"),
            recorded_at=recorded_at,
            source=source,
            status=status,
            tags=require_string_tuple(record, "tags"),
        )

    if record_type == "claim":
        return Claim(
            claim_id=require_string(record, "claim_id"),
            about_event_id=require_string(record, "about_event_id"),
            statement=require_string(record, "statement"),
            recorded_at=recorded_at,
            source=source,
        )

    if record_type == "correction":
        target_text = require_string(record, "target_kind")
        try:
            target_kind = CorrectionTarget(target_text)
        except ValueError as exc:
            raise RecordValidationError(
                f"unknown correction target: {target_text}"
            ) from exc
        return Correction(
            correction_id=require_string(record, "correction_id"),
            target_kind=target_kind,
            target_id=require_string(record, "target_id"),
            replacement_text=require_string(
                record,
                "replacement_text",
            ),
            recorded_at=recorded_at,
            source=source,
        )

    raise RecordValidationError(
        f"unknown record_type: {record_type}"
    )
```

La conversion es deliberada: no usa `asdict` ni `default=str`. Un type hint no valida el record; cada frontera hace narrowing runtime.

### 24.5 `journal.py`

**Archivo completo:**

```python
import json
import os
from json import JSONDecodeError
from pathlib import Path
from tempfile import NamedTemporaryFile

from eidolon.errors import (
    JournalCorruptionError,
    JournalReadError,
    JournalWriteError,
    RecordValidationError,
    UnsupportedSchemaVersionError,
)
from eidolon.model import JournalEntry
from eidolon.serialization import entry_from_record, entry_to_record


def append_entry(path: Path, entry: JournalEntry) -> None:
    record = entry_to_record(entry)
    line = json.dumps(
        record,
        ensure_ascii=False,
        separators=(",", ":"),
    )
    try:
        with path.open("a", encoding="utf-8", newline="\n") as file:
            file.write(line)
            file.write("\n")
    except OSError as exc:
        raise JournalWriteError(
            f"could not append to {path.name}"
        ) from exc


def load_entries(path: Path) -> list[JournalEntry]:
    entries: list[JournalEntry] = []
    try:
        with path.open("r", encoding="utf-8") as file:
            for line_number, raw_line in enumerate(file, start=1):
                try:
                    value = json.loads(raw_line)
                except JSONDecodeError as exc:
                    raise JournalCorruptionError(
                        f"invalid JSON at line {line_number}"
                    ) from exc

                if not isinstance(value, dict):
                    raise JournalCorruptionError(
                        f"expected object at line {line_number}"
                    )

                try:
                    entry = entry_from_record(value)
                except UnsupportedSchemaVersionError as exc:
                    raise UnsupportedSchemaVersionError(
                        f"{exc} at line {line_number}"
                    ) from exc
                except (RecordValidationError, ValueError) as exc:
                    raise JournalCorruptionError(
                        f"invalid record at line {line_number}"
                    ) from exc
                entries.append(entry)
    except OSError as exc:
        raise JournalReadError(
            f"could not read {path.name}"
        ) from exc

    return entries


def write_entries_safely(
    destination: Path,
    entries: list[JournalEntry],
) -> None:
    temporary_path: Path | None = None

    try:
        destination.parent.mkdir(parents=True, exist_ok=True)
        with NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            dir=destination.parent,
            prefix=f".{destination.name}.",
            suffix=".tmp",
            delete=False,
            newline="\n",
        ) as file:
            temporary_path = Path(file.name)
            for entry in entries:
                record = entry_to_record(entry)
                json.dump(
                    record,
                    file,
                    ensure_ascii=False,
                    separators=(",", ":"),
                )
                file.write("\n")
            file.flush()
            os.fsync(file.fileno())

        load_entries(temporary_path)
        temporary_path.replace(destination)
    except OSError as exc:
        raise JournalWriteError(
            f"could not replace {destination.name}"
        ) from exc
    finally:
        if temporary_path is not None and temporary_path.exists():
            temporary_path.unlink()
```

Serialization exceptions se dejan visibles porque representan bugs o contratos de record antes de tocar el file. I/O se traduce con chaining. La validación del temporal ocurre después de close y antes de replace.

Hay una sutileza: si `unlink` del `finally` falla durante otra exception, puede reemplazar la evidencia primaria. Una implementación profesional registraría o agruparía ambos fallos. PF-M9 y el lab profundizarán failure injection; el ejemplo mantiene la mecánica principal visible.

### 24.6 `checks/smoke.py`

**Archivo completo:**

```python
from datetime import UTC, datetime
from pathlib import Path
from tempfile import TemporaryDirectory

from eidolon.journal import (
    append_entry,
    load_entries,
    write_entries_safely,
)
from eidolon.model import (
    Claim,
    Correction,
    CorrectionTarget,
    Event,
    SourceRef,
)


recorded_at = datetime(2026, 8, 26, 18, 31, tzinfo=UTC)
source = SourceRef("src-001")
event = Event(
    event_id="evt-001",
    text="Llegué a casa 🏠",
    valid_time=datetime(2026, 8, 26, 18, 30, tzinfo=UTC),
    recorded_at=recorded_at,
    source=source,
    tags=("home", "arrival"),
)
claim = Claim(
    claim_id="clm-001",
    about_event_id=event.event_id,
    statement="La llegada ocurrió antes de las 19:00",
    recorded_at=recorded_at,
    source=source,
)
correction = Correction(
    correction_id="cor-001",
    target_kind=CorrectionTarget.EVENT,
    target_id=event.event_id,
    replacement_text="Llegué al edificio",
    recorded_at=recorded_at,
    source=SourceRef("src-002"),
)

with TemporaryDirectory() as directory:
    root = Path(directory)
    journal_path = root / "journal.jsonl"

    append_entry(journal_path, event)
    append_entry(journal_path, claim)
    append_entry(journal_path, correction)

    restored = load_entries(journal_path)
    assert restored == [event, claim, correction]
    assert event.text == "Llegué a casa 🏠"
    assert correction.replacement_text == "Llegué al edificio"

    export_path = root / "export.jsonl"
    write_entries_safely(export_path, restored)
    assert load_entries(export_path) == restored

print("PF-M6 journal smoke: PASS")
```

Output:

```text
PF-M6 journal smoke: PASS
```

El journal agrega Event, Claim y Correction en orden. Correction no modifica ni reemplaza el Event: replay puede observar ambos records.

### 24.7 Límites del ejemplo

- no verifica todavía existencia cross-object de cada target;
- no implementa locks ni writers concurrentes;
- no conecta todavía el checksum de §17.4 con cada variante de `JournalEntry`; el mini challenge sí debe hacerlo;
- no implementa database, transaction, query engine ni repository general;
- no borra source ni convierte una proyección en source of truth;
- no implementa custom context manager.

### Predice

¿Qué tres tipos reconstruye `load_entries` y en qué orden?

### Round trip

Cambia source, Unicode y offset temporal. Comprueba equality de los domain objects.

### Detecta el bug

¿Qué ocurriría si `entry_to_record` retornara un `datetime` sin convertir?

### Explica

¿Por qué `load_entries` captura `OSError` solo al abrir, pero deja que un bug de `entry_from_record` conserve su categoría?

---

## 25. Catálogo razonado de failure cases

La tabla resume los fallos de mayor riesgo. Cada uno debe reproducirse o defenderse antes del mini challenge.

| Failure case | Qué ocurrió | Evidencia | Capa que decide |
|---|---|---|---|
| `except Exception: pass` | fallo desaparece | traceback y causa, si aún existen | retirar captura o frontera exterior |
| causa suprimida | error exterior sin origen visible | `__cause__`/context técnico | adapter con `raise ... from exc` |
| source sobrescrito | original truncado/reemplazado | backup o source restante | migration workflow |
| encoding implícito | contrato depende del environment | encoding y bytes | I/O adapter |
| `FileNotFoundError` | path requerido no existe | path y operation | command/application |
| `PermissionError` | OS niega operación | path minimizado y errno | filesystem boundary |
| malformed JSON | syntax inválida | línea/posición y source | parser/policy |
| `datetime` directo | encoder no conoce tipo | tipo y field | serialization |
| Decimal → float | semántica exacta degradada | valor original y conversión | serialization contract |
| schema desconocido | consumer no conoce forma | version y ubicación | migration/application |
| línea JSONL corrupta | record no interpretable | line number, cause, source | replay/quarantine policy |
| file cerrado | operación fuera de lifecycle | `file.closed` y traceback | owner del resource |
| `exists` como garantía | estado cambió después del check | operation real y cause | I/O boundary |
| escritura parcial | destination incompleto | source/destination/temp | safe-write workflow |
| mega-function | parse/domain/I/O/UI acoplados | rutas y efectos mezclados | refactor incremental |
| migración destructiva | derived reemplaza source | source original si sobrevive | migration workflow |

### Failure case adicional: exception demasiado amplia oculta un bug

**Código incorrecto:**

```python
try:
    event_id = record["evnet_id"]  # typo
except Exception:
    event_id = "unknown"
```

El `KeyError` por typo se convierte en un ID inventado. No toda exception “esperable” por tipo representa un fallo de datos; el alcance del `try` y el contrato ayudan a distinguirlo.

### Clasifica

Elige cinco filas y escribe si el programa debe recuperar, abortar o producir quarantine.

### Repara

Transforma el typo en una validación de key obligatoria sin inventar `"unknown"`.

### Explica

¿Qué failure cases pueden dejar el programa con output aparentemente válido?

---

## 26. Ejercicios guiados con solución razonada

### Ejercicio guiado 1 — Event a record JSON-safe

**Objetivo:** serializar sin `asdict` automático.

**Input:** Event con `datetime` aware, SourceRef y tags tuple.

**Antes de ejecutar:** predice qué fields no acepta `json.dumps(event)`.

**Solución:**

```python
def event_to_record(event: Event) -> dict[str, object]:
    return {
        "schema_version": 1,
        "record_type": "event",
        "event_id": event.event_id,
        "text": event.text,
        "valid_time": event.valid_time.isoformat(),
        "source_id": event.source.source_id,
        "tags": list(event.tags),
    }
```

**Razonamiento:** cada conversión declara una decisión: datetime → ISO text, SourceRef → ID y tuple → JSON array. El criterio no es solo que `json.dumps` deje de fallar; el record debe tener schema identificable y no contener objetos de dominio.

**Variación:** agrega Enum y serializa `.value`.

### Ejercicio guiado 2 — Record a Event

**Objetivo:** reconstruir tipos e invariantes, no solo desempaquetar un dict.

**Antes de ejecutar:** explica por qué `Event(**record)` no sirve con el record anterior.

**Solución parcial ejecutable:**

```python
def parse_aware_datetime(text: str) -> datetime:
    value = datetime.fromisoformat(text)
    if value.tzinfo is None or value.utcoffset() is None:
        raise ValueError("datetime must be timezone-aware")
    return value


def event_from_record(record: dict[str, object]) -> Event:
    if record.get("schema_version") != 1:
        raise UnsupportedSchemaVersionError("unsupported schema")
    tags = record.get("tags")
    if not isinstance(tags, list):
        raise RecordValidationError("tags must be an array")
    if any(not isinstance(tag, str) or not tag for tag in tags):
        raise RecordValidationError("invalid tag")

    return Event(
        event_id=require_string(record, "event_id"),
        text=require_string(record, "text"),
        valid_time=parse_aware_datetime(
            require_string(record, "valid_time")
        ),
        source=SourceRef(require_string(record, "source_id")),
        tags=tuple(tags),
    )
```

**Razonamiento:** persisted names y domain constructor no coinciden; además hay runtime narrowing. El criterio es recuperar `datetime`, SourceRef y tuple válidos, no solo construir algo.

### Ejercicio guiado 3 — Preservar tiempo

**Objetivo:** demostrar round trip aware.

**Solución ejecutable:**

```python
from datetime import datetime, timedelta, timezone


offset = timezone(timedelta(hours=-6))
original = datetime(2026, 8, 26, 12, 15, tzinfo=offset)
restored = datetime.fromisoformat(original.isoformat())

assert restored == original
assert restored.utcoffset() == timedelta(hours=-6)
```

**Razonamiento:** el assert compara la semántica y el offset. Un string sin offset debe rechazarse por el contrato de EIDOLON.

**Variación:** normaliza a UTC antes de serializar y documenta que ya no conservas el offset textual original.

### Ejercicio guiado 4 — Rechazar schema desconocido

**Objetivo:** no adivinar compatibilidad.

**Solución ejecutable:**

```python
class UnsupportedSchemaVersionError(Exception):
    pass


def accept_v2(record: dict[str, object]) -> None:
    version = record.get("schema_version")
    if version != 2:
        raise UnsupportedSchemaVersionError(
            f"unsupported schema version: {version}"
        )


accept_v2({"schema_version": 2})
```

**Razonamiento:** un tipo específico permite que una command de migración actúe, mientras replay normal aborta. El criterio es que 1, 3 y ausencia no se conviertan silenciosamente en 2.

### Ejercicio guiado 5 — Leer JSONL por línea

**Objetivo:** conservar ubicación.

**Solución:**

```python
def read_jsonl(path: Path) -> list[dict[str, object]]:
    records: list[dict[str, object]] = []
    with path.open("r", encoding="utf-8") as file:
        for number, line in enumerate(file, start=1):
            value = parse_line(line, number)
            records.append(value)
    return records
```

**Razonamiento:** el owner mantiene el handle dentro de `with`; `parse_line` puede probarse sin filesystem. El criterio es que la línea 1 tenga número 1 y que el file esté cerrado al retornar.

### Ejercicio guiado 6 — Reportar la línea dañada

**Objetivo:** chaining y evidencia no sensible.

**Solución:**

```python
def parse_line(raw_line: str, line_number: int) -> dict[str, object]:
    try:
        value = json.loads(raw_line)
    except JSONDecodeError as exc:
        raise JournalCorruptionError(
            f"invalid JSON at line {line_number}"
        ) from exc
    if not isinstance(value, dict):
        raise JournalCorruptionError(
            f"expected object at line {line_number}"
        )
    return value
```

**Razonamiento:** la exception exterior no imprime el payload, pero la causa conserva el diagnóstico del parser. El criterio es `JournalCorruptionError` con `JSONDecodeError` en `__cause__` para syntax inválida.

### Ejercicio guiado 7 — Separar parsing de I/O

**Objetivo:** aislar pure logic.

**Código inicial incorrecto:** una function abre, parsea, migra y escribe.

**Solución conceptual:**

```python
raw_lines = read_lines(source_path)          # effect
records = parse_lines(raw_lines)             # pure respecto a filesystem
migrated = migrate_records(records)          # pure
write_records_safely(destination, migrated)  # effect
```

**Razonamiento:** ahora una migración incorrecta puede reproducirse con values en memoria. `parse_lines` puede incluir número de línea como input explícito. El criterio es que `migrate_records` no importe `pathlib` ni abra files.

### Ejercicio guiado 8 — Exception chaining en escritura

**Objetivo:** traducir I/O sin perder causa.

**Solución:**

```python
def save_text(path: Path, text: str) -> None:
    try:
        path.write_text(text, encoding="utf-8")
    except OSError as exc:
        raise JournalWriteError(
            f"could not write {path.name}"
        ) from exc
```

**Razonamiento:** el caller reconoce JournalWriteError; el traceback técnico conserva `OSError`. El criterio es `exc.__cause__`, no comparar el mensaje variable del OS.

### Ejercicio guiado 9 — Migrar v1 a v2 sin tocar source

**Objetivo:** source preservation.

**Solución:**

```python
source_records = load_records(source_path)
migrated_records = [
    migrate_event_v1_to_v2(record)
    for record in source_records
]
write_records_safely(destination_path, migrated_records)

assert source_path != destination_path
assert load_records(source_path) == source_records
```

**Razonamiento:** la migration function produce nuevos dicts y el destination es distinto. El criterio no es solo que exista v2: source v1 debe conservar bytes y semántica.

### Ejercicio guiado 10 — Temporal, validación y promoción

**Objetivo:** evitar un destination parcialmente reemplazado.

**Solución razonada:**

1. crea `NamedTemporaryFile(delete=False)` en `destination.parent`;
2. escribe UTF-8 y termina cada record con newline;
3. ejecuta `flush`, cierra y, si la política lo exige, `fsync`;
4. relee y valida el temporal;
5. ejecuta `temporary_path.replace(destination)`;
6. limpia el temporal restante en `finally`;
7. deja propagar cualquier fallo con su causa.

La implementación completa está en la sección 24.5. El criterio es que `replace` nunca ocurra antes de validar y que no se prometa durabilidad absoluta.

---

## 27. Ejercicios independientes

1. Convierte una function que retorna `False` para cualquier fallo en un contrato con ausencia válida y excepciones específicas.
2. Dibuja el call stack de `cli → load_entries → entry_from_record → require_string` ante una key inválida.
3. Reproduce `raise` y demuestra qué líneas posteriores no se ejecutan.
4. Reduce un `try` de ocho líneas a la única operación que puede producir la exception capturada.
5. Escribe ramas específicas para `FileNotFoundError` y `PermissionError`; justifica si deben capturarse.
6. Usa `else` para que un bug posterior al parsing no quede dentro del `try`.
7. Usa `finally` para marcar cleanup y demuestra su ejecución con éxito y fallo.
8. Construye un error exterior con `from exc` y comprueba `__cause__`.
9. Reproduce cómo `from None` cambia el traceback visible.
10. Diseña una taxonomía con máximo cuatro exceptions para un importador.
11. Compara LBYL/EAFP para key opcional, key obligatoria y apertura.
12. Construye un path mediante `/` y explica cada componente.
13. Ejecuta un script desde dos working directories y diagnostica el path relativo.
14. Crea directories anidados con una política deliberada para `exist_ok`.
15. Demuestra `FileNotFoundError` usando un directory temporal.
16. Describe cómo preparar `PermissionError` sin usar un path sensible; documenta la dependencia ambiental.
17. Compara `"r"`, `"w"` y `"a"` con un archivo temporal.
18. Haz round trip UTF-8 con español, emoji y salto de línea.
19. Provoca `UnicodeDecodeError` leyendo como UTF-8 bytes inválidos.
20. Demuestra que leer un file cerrado produce `ValueError`.
21. Compara `read` y loop por líneas con datos sintéticos grandes.
22. Serializa values JSON básicos y anota tipos reconstruidos.
23. Provoca `JSONDecodeError` y registra ubicación sin imprimir todo el payload.
24. Demuestra que tuple vuelve como list después de round trip JSON.
25. Provoca `TypeError` con datetime, Decimal, Enum, dataclass y set.
26. Escribe serializers explícitos para SourceRef y Event.
27. Reconstruye Event validando timezone awareness.
28. Diseña una política de Decimal y justifica string frente a number.
29. Distingue application version y schema version con dos cambios.
30. Rechaza una schema version desconocida con exception específica.
31. Escribe tres records JSONL y comprueba una línea física por record.
32. Inserta corrupción en la segunda línea y comprueba line number y chaining.
33. Diseña políticas diferentes para `verify` y `replay`.
34. Crea un quarantine receipt sin payload.
35. Reproduce truncamiento por `"w"` y explica cuándo ocurre.
36. Simula conceptualmente un fallo después de medio record en append.
37. Implementa temp + replace y comprueba que no quedan temporales tras éxito.
38. Enumera qué fallos todavía no cubre `fsync` del file.
39. Migra un dict v1 a v2 sin mutar el input.
40. Migra un JSONL a un destination distinto y comprueba que source no cambió.
41. Separa una mega-function en parsing, validation, migration e I/O.
42. Redacta mensajes para malformed JSON, schema desconocido y permission denied sin payload.
43. Agrega Claim al serializer integrado y prueba round trip.
44. Agrega Correction y demuestra que Event conserva su texto.
45. Defiende cuándo JSONL deja de ser apropiado ante concurrencia y constraints cross-record.

---

## 28. Preguntas conceptuales

1. ¿Cuándo debes capturar una exception y cuándo dejarla propagarse?
2. ¿Qué diferencia existe entre ausencia válida, error esperado y bug?
3. ¿Por qué `except Exception` puede destruir evidencia aunque imprima?
4. ¿Qué conserva un traceback que un boolean pierde?
5. ¿Qué cambia entre `raise`, `raise NewError(...) from exc` y `from None`?
6. ¿Por qué `else` reduce el alcance accidental de captura?
7. ¿Cuándo `finally` añade claridad y cuándo `with` es mejor?
8. ¿Por qué dominio no debería imprimir un `PermissionError`?
9. ¿Qué tradeoff existe entre LBYL y EAFP?
10. ¿Por qué `path.exists()` no garantiza `open()`?
11. ¿Qué diferencia existe entre working directory y module directory?
12. ¿Por qué un path relativo puede romper reproducibilidad?
13. ¿Qué representa un file object y qué recurso controla?
14. ¿Por qué garbage collection no sustituye cierre explícito?
15. ¿Qué diferencia existe entre `flush`, `close` y persistencia física?
16. ¿Cuándo conviene leer todo y cuándo línea por línea?
17. ¿Qué diferencia existe entre serialización y persistencia?
18. ¿Por qué una dataclass no es un formato persistente?
19. ¿Qué pierde JSON respecto a tuple, set, Enum, Decimal y datetime?
20. ¿Por qué `default=str` puede ocultar un contrato incompleto?
21. ¿Por qué `datetime` necesita política explícita y validación aware?
22. ¿Qué diferencia hay entre application version y schema version?
23. ¿Por qué una versión desconocida no debe convertirse automáticamente?
24. ¿Qué ventaja ofrece JSONL a un journal append-only?
25. ¿Qué no ofrece JSONL?
26. ¿Por qué append no garantiza una línea completa ante crash?
27. ¿Por qué temp + replace es más seguro que sobrescribir?
28. ¿Por qué atomic replace no implica durabilidad absoluta?
29. ¿Por qué el temporal debe estar en el mismo filesystem?
30. ¿Qué diferencia existe entre source data y derived representation?
31. ¿Por qué una migration pure es más fácil de verificar?
32. ¿Por qué una migración no debe destruir el único source?
33. ¿Qué evidencia mínima necesita una línea en quarantine?
34. ¿Por qué un receipt no debería copiar payload privado?
35. ¿Cómo preserva Correction que el Event original no cambia?
36. ¿Cuándo JSONL debe ceder lugar a una database transaccional?

---

## 29. Mini challenge — Journal JSONL versionado y migración segura

### 29.1 Objetivo

Construye una persistencia local educativa para los domain objects de PF-M5. Debe demostrar exceptions, UTF-8, JSONL, schema explícito, round trip, corrupción localizada, source preservation y promoción segura. Prepara PF-L09, PF-L10, PF-L11 y PF-MP2.

### 29.2 Árbol requerido

```text
eidolon-pf-m6/
├── pyproject.toml
├── src/
│   └── eidolon/
│       ├── __init__.py
│       ├── errors.py
│       ├── model.py
│       ├── serialization.py
│       ├── journal.py
│       └── migration.py
└── checks/
    ├── smoke.py
    ├── corrupt_line.py
    ├── unknown_schema.py
    ├── source_preserved.py
    └── partial_write_policy.md
```

Reutiliza el package instalado y environment de PF-M4. `dependencies = []`: standard library basta.

### 29.3 Modelos

Incluye:

1. frozen `SourceRef` con ID no vacío;
2. `SourceRecord` con ID, Unicode text, `recorded_at` aware y SourceRef;
3. `Event` con ID, Unicode text, `valid_time` aware, `recorded_at` aware, SourceRef y tuple de tags;
4. `Claim` con ID, Event ID, statement, `recorded_at` y SourceRef;
5. `Correction` con ID, target kind, target ID, replacement text, `recorded_at` y SourceRef;
6. Enum para conjuntos cerrados;
7. invariantes runtime mínimas.

No agregues fields de EIDOLON 1.0.

### 29.4 Taxonomía

Define, como máximo:

- `JournalError`;
- `RecordValidationError`;
- `UnsupportedSchemaVersionError`;
- `JournalCorruptionError`;
- `JournalChecksumError`;
- `JournalReadError`;
- `JournalWriteError`.

Usa chaining al traducir `JSONDecodeError` y `OSError`. No captures bugs de programación como JournalError.

### 29.5 Serialization contract

Implementa:

```python
def entry_to_record(entry: JournalEntry) -> dict[str, object]:
    ...


def entry_from_record(record: dict[str, object]) -> JournalEntry:
    ...
```

El schema actual es 2 e incluye `schema_version`, `record_type`, ID estable, `source_id`, `recorded_at`, fields específicos, timestamps ISO con offset, tags como JSON array y checksum calculado según §17.4. Incluye un `record_type` distinto para SourceRecord.

No uses `pickle`, `asdict` como contrato completo ni `default=str`.

### 29.6 Journal append-only

Implementa:

```python
def append_entry(path: Path, entry: JournalEntry) -> None:
    ...


def load_entries(path: Path) -> list[JournalEntry]:
    ...
```

Requisitos:

1. UTF-8 explícito;
2. una línea compacta por entry;
3. iteración línea por línea;
4. números desde 1;
5. malformed JSON produce `JournalCorruptionError` con `JSONDecodeError` como causa;
6. schema desconocido produce `UnsupportedSchemaVersionError` y conserva línea;
7. no se saltan blank/corrupt lines silenciosamente;
8. una Correction se agrega después del Event y no lo reemplaza.
9. checksum ausente o inconsistente produce `JournalChecksumError` con line number y conserva el source.

### 29.7 Round trip

`checks/smoke.py` crea SourceRecord, Event, Claim y Correction sintéticos con acentos y emoji. Debe comprobar:

```python
assert load_entries(path) == [source_record, event, claim, correction]
assert event.text == "Llegué a casa 🏠"
assert correction.target_id == event.event_id
assert correction.replacement_text == "Llegué al edificio"
assert event.valid_time.tzinfo is not None
```

Output final:

```text
PF-M6 journal challenge: PASS
```

### 29.8 Corrupción y evidence

`checks/corrupt_line.py` crea tres líneas y daña la segunda. Comprueba capturando solo para el check:

```python
try:
    load_entries(path)
except JournalCorruptionError as exc:
    assert "line 2" in str(exc)
    assert isinstance(exc.__cause__, JSONDecodeError)
else:
    raise AssertionError("corruption was not detected")
```

El source no se modifica. Diseña además un receipt con line number y error type, sin payload. No es obligatorio continuar replay después de la corrupción.

### 29.9 Schema v1 → v2

`migration.py` contiene pure functions:

```python
def migrate_record_v1_to_v2(
    record: dict[str, object],
) -> dict[str, object]:
    ...


def migrate_records_v1_to_v2(
    records: list[dict[str, object]],
) -> list[dict[str, object]]:
    ...
```

La versión 1 usa `id` y `happened_at` para Event. La versión 2 usa `event_id` y `valid_time`, y agrega `record_type`, `status` y `tags`. La migration:

1. acepta solo v1 conocido;
2. valida fields obligatorios;
3. crea dicts nuevos;
4. no abre files;
5. no muta inputs;
6. conserva IDs, Unicode, source y tiempos;
7. rechaza record types no implementados en vez de adivinar.

### 29.10 Escritura segura del resultado

Implementa:

```python
def write_records_safely(
    destination: Path,
    records: list[dict[str, object]],
) -> None:
    ...
```

Debe:

1. crear temporal en `destination.parent`;
2. escribir JSONL UTF-8 completo;
3. flush, cerrar y opcionalmente solicitar `fsync` según política documentada;
4. releer y validar el temporal;
5. reemplazar destination solo después;
6. limpiar temporal restante;
7. propagar error con chaining cuando corresponda;
8. no prometer durabilidad absoluta.

`source_path` y `destination` deben ser distintos. `checks/source_preserved.py` guarda el texto original, migra y comprueba:

```python
assert source_path.read_text(encoding="utf-8") == original_text
assert load_records(destination)[0]["schema_version"] == 2
```

### 29.11 Failure cases obligatorios

Demuestra y explica:

1. `FileNotFoundError` en source requerido;
2. malformed JSON en línea 2;
3. schema 99;
4. naive datetime;
5. `datetime` enviado directamente a `json.dumps`;
6. destination preexistente que no se reemplaza hasta completar temporal;
7. intento rechazado de usar source como destination;
8. Correction que deja Event intacto;
9. checksum inconsistente en una línea preservada.

En `partial_write_policy.md` responde:

- qué conserva el algoritmo si falla antes de replace;
- qué temporal podría quedar;
- por qué `try/except` solo no hace rollback;
- qué garantías dependen del filesystem;
- por qué close/`fsync` no garantizan cualquier crash.

### 29.12 Comprobaciones ejecutables

Desde un venv limpio:

```bash
python -m compileall -q src checks
python checks/smoke.py
python checks/corrupt_line.py
python checks/unknown_schema.py
python checks/source_preserved.py
```

Todos los scripts capturan únicamente el failure esperado para verificarlo y deben terminar exitosamente. No necesitan pytest.

### 29.13 Criterio de aprobación

El challenge se resuelve si:

- models conservan invariantes de PF-M5;
- serializers son explícitos y JSON-safe;
- SourceRecord/Event/Claim/Correction pasan round trip;
- JSONL tiene una entry por línea y UTF-8;
- error corrupto conserva line number y cause;
- checksum inconsistente se detecta sin presentar el hash como autenticación;
- schema desconocido no se migra implícitamente;
- migration es pure y no muta records;
- source bytes/text permanecen idénticos;
- destination se valida antes de replace;
- Correction no reescribe Event;
- no existe captura amplia silenciosa;
- límites de atomicidad/durabilidad están escritos con precisión.

### 29.14 Límites

No implementes custom decorators, custom context managers, async, pytest avanzado, database, ORM, API, Pydantic, FastAPI, Docker, AI, locks distribuidos ni un query engine sobre JSONL.

---

## 30. Resumen

- Una exception representa que una operación no completó su contrato normal.
- Resultado válido, ausencia válida, fallo esperado y bug necesitan contratos distintos.
- `raise` interrumpe la ruta normal y la exception se propaga hasta que una capa la maneja.
- El traceback conserva tipo, mensaje, punto del fallo y callers.
- Un `try` pequeño y captures específicas evitan esconder bugs.
- `else` separa la ruta exitosa; `finally` asegura cleanup, pero ninguno es obligatorio.
- Captura solo cuando la capa sabe recuperar, traducir o añadir contexto útil.
- `raise ... from exc` conserva la causa técnica bajo un error superior.
- LBYL y EAFP son decisiones bajo races y contratos, no reglas absolutas.
- `Path` expresa componentes, pero un relative path depende del working directory.
- `"w"` trunca, `"a"` agrega y ninguno crea transactions.
- UTF-8 explícito evita que el formato dependa del environment.
- `with` delimita lifecycle; garbage collection no sustituye cierre.
- Leer línea por línea reduce materialización y localiza fallos tardíos.
- JSON es texto interoperable; no conserva tipos Python complejos automáticamente.
- `dumps/loads` transforman values/text; `dump/load` trabajan con file objects.
- `asdict` no define schema, compatibilidad ni políticas de tipos.
- Domain object y persisted record son contratos diferentes.
- `datetime` aware y Decimal requieren representaciones deliberadas.
- `schema_version` describe el record, no la release completa.
- JSONL ayuda con append, replay y corrupción localizada, pero no es database.
- Un checksum sobre JSON canónico detecta cambios bajo ese contrato, pero no autentica el origen.
- Saltar una línea corrupta sin receipt destruye evidencia.
- Append puede dejar una última línea parcial y no coordina writers.
- Temporal + validation + replace reduce destinos parciales, no promete durabilidad absoluta.
- Source data no debe sobrescribirse con una derived representation.
- Migration pure separa semántica del filesystem.
- SourceRecord, Event, Claim y Correction se serializan como tipos distintos.
- Correction agrega evidencia y no reescribe silenciosamente el Event.

---

## 31. Checklist de dominio

- [ ] Puedo distinguir resultado, ausencia, fallo esperado y bug.
- [ ] Puedo explicar propagación y leer un traceback práctico.
- [ ] Puedo usar `raise` para rechazar una violación contractual.
- [ ] Puedo mantener pequeño un `try`.
- [ ] Puedo capturar exceptions específicas en orden correcto.
- [ ] Puedo detectar `except Exception: pass` y explicar evidencia perdida.
- [ ] Puedo usar `else` y `finally` solo cuando agregan claridad.
- [ ] Puedo decidir capturar o propagar según la capa.
- [ ] Puedo implementar chaining y comprobar `__cause__`.
- [ ] Puedo diseñar una taxonomía pequeña.
- [ ] Puedo comparar LBYL y EAFP.
- [ ] Puedo explicar por qué `exists` no garantiza la operación posterior.
- [ ] Puedo construir paths con `Path`.
- [ ] Puedo distinguir path relativo, absoluto, working y module directory.
- [ ] Puedo usar operaciones prácticas de `Path` deliberadamente.
- [ ] Puedo elegir `r`, `w`, `a`, texto y binario.
- [ ] Puedo declarar UTF-8 en fronteras textuales.
- [ ] Puedo explicar `str → bytes → str`.
- [ ] Puedo delimitar un file handle con `with`.
- [ ] Puedo detectar uso de un resource cerrado.
- [ ] Puedo distinguir buffer, flush, close y durabilidad.
- [ ] Puedo elegir lectura completa o línea por línea.
- [ ] Puedo usar `json.dumps/loads` y `dump/load`.
- [ ] Puedo provocar y clasificar `JSONDecodeError`.
- [ ] Puedo explicar los tipos que JSON no conserva.
- [ ] Puedo evitar `default=str` como sustituto de schema.
- [ ] Puedo separar domain object y persisted record.
- [ ] Puedo implementar Event → record → Event.
- [ ] Puedo conservar Unicode y datetime aware.
- [ ] Puedo justificar una política Decimal.
- [ ] Puedo definir y validar `schema_version`.
- [ ] Puedo distinguir schema y application version.
- [ ] Puedo escribir JSONL append-only de un solo writer.
- [ ] Puedo calcular y verificar un checksum canónico sin confundir integridad con autenticidad.
- [ ] Puedo reportar línea corrupta con chaining.
- [ ] Puedo elegir fail fast, quarantine o reporte.
- [ ] Puedo diseñar un receipt sin payload sensible.
- [ ] Puedo explicar por qué `"w"` destruye y `"a"` no es transaction.
- [ ] Puedo explicar una escritura parcial.
- [ ] Puedo implementar temporal + close + validation + replace.
- [ ] Puedo declarar límites de atomicidad y durabilidad.
- [ ] Puedo distinguir source y derived data.
- [ ] Puedo migrar records con una pure function.
- [ ] Puedo escribir migrated data sin tocar el único source.
- [ ] Puedo serializar Event, Claim y Correction por contratos distintos.
- [ ] Puedo demostrar que Correction no muta Event.
- [ ] Puedo completar el mini challenge solo con PF-M1–PF-M6.

---

## 32. Preparación para labs y EIDOLON 0.0a

### Lab principal: PF-L09 — Taxonomía de errores

PF-M6 prepara categorías input, domain, parsing/schema, I/O y bug; captura específica; propagación; chaining y traducción exterior sin destruir el traceback técnico.

La CLI del lab decidirá presentación y exit status; el módulo no desarrolla argparse ni logging avanzado.

### PF-L10 — Exportación atómica

PF-M6 prepara ownership del file handle, temporal en el mismo directory, close/flush/`fsync` al nivel conceptual, validación antes de replace, cleanup y límites de garantía.

PF-L10 implementará y probará un custom context manager; ese mecanismo pertenece principalmente a PF-M7.

### PF-L11 — Repositorio JSONL versionado

PF-M6 prepara append/load/replay, `schema_version`, round trip explícito, line number, quarantine policy y los domain objects. “Repository” es el nombre del lab; PF-M6 no crea una arquitectura persistente universal ni una database.

### PF-MP2 — Migrador de schema JSONL

PF-M6 prepara pure migration, source preservation, destination temporal, validation y promotion. Golden files, failure injection extensa, rollback y ADR pertenecen al mini proyecto/labs y PF-M9.

### Evidencia antes de avanzar

1. propagation dibujada desde parser hasta frontera exterior;
2. chaining demostrado mediante `__cause__`;
3. UTF-8 y timestamp aware con round trip;
4. SourceRecord/Event/Claim/Correction serializados explícitamente;
5. JSONL con line number ante corrupción y checksum inconsistente;
6. source v1 idéntico después de migrar;
7. destination v2 validado antes de replace;
8. Correction presente junto al Event original;
9. nota de límites de JSONL, atomic replace y durabilidad;
10. mini challenge ejecutado desde environment limpio.

Este módulo refuerza **CHECKPOINT PF-C2 — Diseño y lifecycle** y aporta la primera persistencia local auditable de EIDOLON 0.0a.

---

## 33. Recursos de ampliación

La explicación fundamental está contenida aquí. Los recursos canónicos del track permanecen en [PF.11 Recursos recomendados](../../02_curriculum/01_programming_foundations.md#pf11-recursos-recomendados), especialmente Python 3.14 sobre exceptions, `pathlib`, `json`, `datetime` y `tempfile`.

Consulta documentación oficial para verificar flags o garantías de una plataforma concreta. No uses una receta de “atomic write” como prueba de durabilidad sin leer el contrato del OS y filesystem.

---

## 34. Límite del módulo

PF-M6 termina en exceptions, file resources, paths, UTF-8, JSON/JSONL, schema version, round trip, corruption evidence, source preservation y atomic replace práctico con standard library.

PF-M7 enseñará decorators y custom context managers como políticas; PF-M8, async/await y ownership bajo cancellation; PF-M9, pytest avanzado, failure injection, debugging y logging.

No se introducen Pydantic, SQLAlchemy, PostgreSQL, FastAPI, Docker, Ollama, embeddings, LLMs, databases ni ORMs. JSONL permanece como persistencia educativa transitoria; no se convierte en una database casera.
