# PF-M9 — Testing, debugging, logging y revisión

**Track:** Programming Foundations  
**Competencias:** D1.1, D1.2, D1.3, D3.1, D3.2, D11.1  
**Fase:** P0  
**Nivel objetivo:** profesional para evidencia reproducible dentro del alcance de Programming Foundations  
**Prerequisites:** PF-M1, PF-M2, PF-M3, PF-M4, PF-M5, PF-M6, PF-M7, PF-M8  
**Build:** EIDOLON 0.0a  
**Curriculum source:** [PF-M9](../../02_curriculum/01_programming_foundations.md#pf-m9--testing-debugging-logging-y-revisión)  
**Status:** approved

`Funcionó una vez` no significa que un comportamiento esté protegido. Un cambio pequeño puede romper Unicode, replay o cleanup sin producir un síntoma inmediato. Prints dispersos y cambios simultáneos agregan ruido, no necesariamente evidencia.

PF-M9 cierra la secuencia de Programming Foundations con una disciplina:

```text
comportamiento esperado
→ test
→ falla reproducible
→ hipótesis y evidencia
→ fix mínimo
→ regression test
→ logging seguro
→ review
→ rollback/documentación
```

Un test es evidencia limitada sobre un contrato específico, no prueba absoluta de ausencia de bugs. PF-M1–PF-M8 aportan el sistema que aquí se protege: modelo de datos, funciones, colecciones, packages, dataclasses/types, JSONL, lifecycle, policies y async.

No se realiza la auditoría global del track, no se cambian estados de módulos anteriores y no comienza Computer Science Foundations.

## Resultados de aprendizaje

Al terminar deberías poder:

- formular un test desde preconditions y behavior observable;
- distinguir behavior contract de implementation detail;
- escribir tests positivos, negativos, de frontera e invariantes;
- estructurar tests con Arrange–Act–Assert cuando aclare intención;
- usar discovery, `assert`, `pytest.raises` y parametrization de pytest;
- diseñar fixtures explícitas con cleanup;
- probar filesystem con `tmp_path`;
- controlar environment y clock mediante contratos o `monkeypatch`;
- elegir entre real implementation, fake, stub y mock con cautela;
- distinguir unit, integration y end-to-end tests sin dogmatismo;
- formular una property independiente y probarla con Hypothesis;
- interpretar coverage como señal, no como objetivo;
- convertir un bug reproducible en regression test;
- reducir una falla y separar symptom de root cause;
- leer traceback y usar `breakpoint()`/debugger;
- elegir print o logging según la pregunta;
- emitir logs con level, context, operation ID y redaction;
- usar `logger.exception` sin duplicar failures en cada layer;
- diagnosticar coroutine no esperada, forgotten Task y flaky async order;
- diseñar tests async acotados sin plugin innecesario;
- usar commits pequeños como unidades revisables y revertibles;
- revisar un diff por correctness, contracts, privacy, tests y rollback;
- redactar una PR autocontenida y un ADR corto;
- usar `git bisect` para reducir una regresión histórica;
- distinguir `git revert` de history rewriting destructivo;
- defender EIDOLON 0.0a con evidencia reproducible.

## Cómo estudiar este módulo

Para cada falla:

1. conserva input, environment y traceback;
2. escribe el behavior esperado sin describir internals;
3. reduce al caso mínimo;
4. formula dos hipótesis distinguibles;
5. obtiene una observación concreta;
6. crea un test que falla por la razón correcta;
7. aplica un cambio mínimo;
8. ejecuta el test y la suite relevante;
9. revisa diff, datos expuestos y rollback;
10. registra decisiones duraderas.

### Convenciones

- **Ejemplo ejecutable:** programa autónomo.
- **Archivo ejecutable:** contenido válido dentro del filename indicado.
- **Fragmento:** depende del contexto explicado.
- **Código incorrecto:** antipatrón deliberado.
- **Failure case:** debe fallar bajo el contrato indicado.
- **Comando:** se ejecuta en terminal desde el directorio señalado.
- **Output:** contiene solo propiedades estables.

Baseline de ejemplos: Python 3.14, pytest 9 y Hypothesis 6. pytest/Hypothesis son dependencies de desarrollo, no runtime requirements de EIDOLON. Decláralos en `pyproject.toml` y usa el mecanismo de resolución/locking elegido por el proyecto según PF-M4.

---

## 1. Por qué probar software

Esta normalización funciona para el ejemplo inicial:

```python
def normalize_tag(tag):
    return tag.strip().lower()
```

Después alguien agrega `replace(" ", "-")` y rompe un consumidor que preservaba espacios internos. Haber visto output correcto una vez no conserva el contrato.

```text
expectation no escrita
→ cambio
→ regresión silenciosa

expectation observable
→ test
→ cambio incompatible produce evidencia
```

Un test reduce incertidumbre sobre inputs concretos o una propiedad. No cubre paths que nunca ejercita, environment desconocido ni assertions que no formuló.

### Diseña el test

¿Qué behavior protegerías si tags deben recortar extremos y conservar espacios internos?

### Explica

¿Por qué diez tests que solo usan `"work"` siguen ofreciendo evidencia limitada?

---

## 2. Modelo mental: precondition → operation → observation

```text
Arrange: input/precondition
        ↓
Act: una operación relevante
        ↓
Assert: behavior observable
```

AAA ayuda a leer, no exige comentarios ceremoniales. Si Act requiere abrir cinco resources, mutar globals y ejecutar CLI, el test revela un problema de diseño: la operation no tiene una frontera clara.

### 2.1 Behavior, no implementation incidental

**Código frágil:**

```python
def test_index_uses_dict(journal):
    assert isinstance(journal._events_by_id, dict)
```

El test impide reemplazar un dict interno por otra representación que conserve lookup semantics.

**Contrato observable:**

```python
def test_find_returns_event_for_existing_id(journal, event):
    journal.add(event)

    found = journal.find(event.event_id)

    assert found == event
```

Conecta con PF-M2: inputs, output, effects y errors forman el contrato. Private representation no, salvo que sea precisamente el behavior público prometido.

### Detecta el test débil

¿Qué refactor correcto rompe el primer test pero no el segundo?

### Review

Un test afirma cantidad exacta de llamadas a una helper privada. ¿Protege behavior o estructura actual?

---

## 3. De `assert` a pytest

Python ya permite assertions:

```python
assert normalize_tag("  Work  ") == "work"
```

pytest agrega discovery, reports, exception assertions, parametrization, fixtures, monkeypatch y un ecosistema extensible. No decide qué vale la pena probar.

### 3.1 Proyecto mínimo

```text
project/
├── pyproject.toml
├── src/
│   └── eidolon/
│       └── tags.py
└── tests/
    └── test_tags.py
```

**`src/eidolon/tags.py`:**

```python
def normalize_tag(tag: str) -> str:
    return tag.strip().lower()
```

**`tests/test_tags.py`:**

```python
from eidolon.tags import normalize_tag


def test_normalize_tag_trims_and_lowercases():
    result = normalize_tag("  Work  ")

    assert result == "work"
```

**`pyproject.toml` — fragmento:**

```toml
[project.optional-dependencies]
test = [
  "pytest>=9,<10",
  "hypothesis>=6,<7",
]

[tool.pytest.ini_options]
testpaths = ["tests"]
```

Comandos:

```bash
python -m pip install -e '.[test]'
python -m pytest -q
```

`python -m pytest` hace explícito qué interpreter/environment ejecuta, útil para PF-M4. Discovery encuentra `test_*.py` y functions `test_*`.

Output estable del ejemplo:

```text
1 passed
```

No declares solo `pip install pytest` en una máquina y llames a eso reproducibilidad: instalación, declaración y locking siguen siendo contratos distintos.

### Comprueba en terminal

¿Qué Python ejecuta pytest y dónde está instalado `eidolon`?

### Detecta el bug

El test se llama `check_normalize` y pytest reporta `no tests ran`. ¿Qué convención falta?

---

## 4. Positive, negative, boundary e invariant tests

Estas categorías responden preguntas diferentes:

| Tipo | Pregunta | Ejemplo EIDOLON |
|---|---|---|
| positive | ¿funciona una entrada válida? | Event válido round trip |
| negative | ¿rechaza exactamente lo inválido? | schema no soportado |
| boundary | ¿qué ocurre en la frontera? | texto vacío, Unicode, una línea corrupta |
| invariant | ¿qué siempre debe conservarse? | Correction no muta Event original |

### 4.1 Positive y negative

```python
def test_parse_status_accepts_active():
    assert parse_status("active") == Status.ACTIVE


def test_parse_status_rejects_unknown():
    with pytest.raises(ValueError, match="unsupported status"):
        parse_status("archived-forever")
```

El negative test no afirma “ocurrió algo”. Especifica error type y, cuando el texto es contrato estable, una parte del message.

### 4.2 Boundary partitions

Para una collection: cero, uno, varios, duplicado.  
Para text: vacío, whitespace, acento, emoji, combining character.  
Para time: aware UTC, offset, frontera DST relevante.  
Para JSONL: file vacío, última newline ausente, línea corrupta, schema futuro.  
Para async: success inmediato, timeout, cancellation durante await.

No pruebes cada valor posible; divide input space en clases donde esperas comportamiento distinto.

### 4.3 Invariant

```python
def test_correction_preserves_original_event():
    original = Event("evt-1", "texto original")

    correction = correct(original, "texto corregido")

    assert original.text == "texto original"
    assert correction.target_id == original.event_id
    assert correction.replacement_text == "texto corregido"
```

Otros invariants de P0: IDs únicos, replay equivalente, migration preserva source, cancellation limpia staged state e index derivado puede reconstruirse.

### Boundary

Diseña partitions para lookup por ID y justifica por qué cada una puede revelar behavior distinto.

### Invariante

¿Qué assertion demuestra que migration no destruyó source? Evita comparar solo file size.

---

## 5. Exceptions con `pytest.raises`

**Archivo ejecutable:**

```python
import pytest


def parse_version(value):
    if value not in {1, 2}:
        raise ValueError(f"unsupported version: {value}")
    return value


def test_parse_version_rejects_future_version():
    with pytest.raises(ValueError, match="unsupported version") as captured:
        parse_version(99)

    assert captured.value.args[0] == "unsupported version: 99"
```

`pytest.raises(Exception)` es casi siempre demasiado amplio: un `NameError` accidental también haría pasar el test. Captura la clase contractual más específica.

El match usa regular expression. No amarres el test a punctuation cambiante si el message completo no es API.

### Predice

¿Pasaría el test si `parse_version` contiene un typo que produce `NameError`?

### Detecta el test débil

```python
with pytest.raises(Exception):
    parse_version(99)
```

¿Qué bug podría aceptar?

---

## 6. Parametrization: una clase coherente de casos

Tres tests casi idénticos añaden ruido:

```python
def test_empty_tag(): ...
def test_spaces_tag(): ...
def test_uppercase_tag(): ...
```

**Archivo ejecutable:**

```python
import pytest


def normalize_tag(tag):
    return tag.strip().lower()


@pytest.mark.parametrize(
    ("raw", "expected"),
    [
        ("Work", "work"),
        ("  work  ", "work"),
        ("ÁRBOL", "árbol"),
    ],
    ids=["case", "outer-space", "unicode"],
)
def test_normalize_tag(raw, expected):
    assert normalize_tag(raw) == expected
```

Parametriza cuando los casos comparten contract y assertion. Separa un caso si necesita una historia, setup o failure policy distinta. Una matriz gigante puede ocultar intención.

### Parametriza

¿`None` y string vacío pertenecen a la misma clase si uno produce `TypeError` y otro `ValueError`? Justifica.

---

## 7. Fixtures, cleanup y `tmp_path`

Una fixture nombra setup reutilizable. Debe hacer visible lo importante.

```python
import pytest


@pytest.fixture
def event():
    return Event("evt-1", "Llegué a casa")
```

Evita una fixture `whole_application` que crea package, clock, files, logger, 40 Events y mocks invisibles.

El default scope `function` crea una instance por test y suele maximizar aislamiento. Scopes más amplios (`module`, `session`) pueden reducir setup cost, pero comparten lifecycle y aumentan riesgo de state leak; úsalos solo con un resource realmente costoso y contract explícito.

### 7.1 Yield fixture

```python
@pytest.fixture
def open_journal(tmp_path):
    path = tmp_path / "journal.jsonl"
    handle = path.open("w+", encoding="utf-8")
    yield handle
    handle.close()
```

Antes de `yield` ocurre setup; después, teardown incluso si el test falla. Conecta con PF-M7 sin repetir context manager internals.

### 7.2 `tmp_path`

**Archivo ejecutable:**

```python
import json


def append_record(path, record):
    with path.open("a", encoding="utf-8") as handle:
        handle.write(json.dumps(record, ensure_ascii=False) + "\n")


def test_append_record_preserves_unicode(tmp_path):
    journal = tmp_path / "journal.jsonl"

    append_record(journal, {"text": "pingüino 🐧"})

    assert "pingüino 🐧" in journal.read_text(encoding="utf-8")
```

`tmp_path` da un directory único por test. No escribe en Desktop, no depende del home real y pytest administra cleanup general. El test todavía posee cada file contract que crea.

Úsalo para JSONL, migration, atomic export y corrupted-line fixtures.

### Detecta el bug

Dos tests escriben `/tmp/journal.jsonl` con el mismo nombre global. Enumera dos fuentes de flakiness.

### Cleanup

¿Qué resource posee la fixture y qué resource administra `tmp_path`?

---

## 8. Controlar no determinismo: clock y environment

Tests flaky suelen depender de una frontera no controlada: current time, random, environment, scheduling o filesystem compartido.

### 8.1 Primero: contract explícito

```python
from datetime import datetime, timezone


def make_receipt(record_id, now):
    return {"record_id": record_id, "recorded_at": now().isoformat()}


def test_make_receipt_uses_supplied_clock():
    fixed = datetime(2026, 1, 2, 3, 4, tzinfo=timezone.utc)

    receipt = make_receipt("rec-1", lambda: fixed)

    assert receipt["recorded_at"] == "2026-01-02T03:04:00+00:00"
```

La dependency aparece en el function contract. No requiere framework de dependency injection ni monkeypatch.

### 8.2 Monkeypatch de environment

**Archivo ejecutable:**

```python
import os


def journal_mode():
    return os.getenv("EIDOLON_MODE", "safe")


def test_journal_mode_reads_environment(monkeypatch):
    monkeypatch.setenv("EIDOLON_MODE", "preview")

    assert journal_mode() == "preview"


def test_journal_mode_uses_default(monkeypatch):
    monkeypatch.delenv("EIDOLON_MODE", raising=False)

    assert journal_mode() == "safe"
```

pytest restaura environment al terminar cada test. Eliminar explícitamente prueba default aun si la shell del autor tiene la variable.

### 8.3 Monkeypatch de boundary pequeña

```python
def test_receipt_uses_module_clock(monkeypatch):
    monkeypatch.setattr(receipts, "current_time", lambda: fixed_time)

    receipt = receipts.make_receipt("rec-1")

    assert receipt.recorded_at == fixed_time
```

Parchea el nombre que el code under test realmente consulta. Si todo necesita monkeypatch de internals, la frontera está demasiado acoplada.

### Decide

¿Preferirías parámetro `now` o monkeypatch para una pure function nueva? ¿Y para una legacy module boundary pequeña?

### Detecta el test débil

Un test afirma que `datetime.now()` ocurrió “hace menos de 10 ms”. ¿Por qué puede fallar sin regresión?

---

## 9. Test doubles sin confianza falsa

Taxonomía práctica, no rígida:

- **fake:** implementación funcional simplificada, como un in-memory store;
- **stub:** retorna respuestas predefinidas;
- **mock:** permite verificar interactions esperadas.

```python
class FakeEventStore:
    def __init__(self):
        self.events = {}

    def add(self, event):
        self.events[event.event_id] = event

    def get(self, event_id):
        return self.events.get(event_id)
```

El fake ejecuta behavior real pequeño. Un stub clock puede devolver un instant fijo. Un mock podría verificar que un notification boundary recibió un ID, si esa interaction es el contract.

### 9.1 Overmocking

**Código incorrecto:**

```python
def test_journal(mock_json, mock_open, mock_serializer, mock_path):
    mock_serializer.return_value = "record"
    save_event(mock_path, event)
    mock_open.assert_called_once()
```

El test reemplaza serialization y filesystem: no demuestra que JSONL sea válido, UTF-8 se preserve ni file contenga record. Un integration test con `tmp_path` es más simple y fiel.

> No mockees lo que puedes probar de forma simple y real.

### Review

¿Qué pregunta real responde el test overmocked? ¿Qué bugs deja intactos?

### Decide

Elige real/fake/stub/mock para: clock; JSON serializer estándar; in-memory lookup; future email boundary que no debe enviarse.

---

## 10. Unit, integration y end-to-end

### 10.1 Unit test

Protege behavior pequeño/cohesivo con feedback rápido y dependencies mínimas. Unidad no significa automáticamente “una clase”. Una pure function o un pequeño object collaboration puede ser unidad.

### 10.2 Integration test

```text
Event
↓ serializer real
JSONL real
↓ tmp_path
load real
↓
Event semánticamente equivalente
```

No necesita database para integrar piezas.

### 10.3 End-to-end introductorio

```text
CLI arguments
↓ command
journal file
↓ another command
CLI output
```

Cruza fronteras exteriores y ofrece mayor fidelity, pero suele ser más lento y difícil de diagnosticar.

| Alcance | Velocidad típica | Fidelidad | Diagnóstico |
|---|---|---|---|
| unit | alta | acotada | directo |
| integration | media | varias piezas reales | moderado |
| end-to-end | menor | frontera amplia | más costoso |

“Muchos rápidos, menos amplios” es una heurística, no ley. El riesgo del producto determina la combinación.

### Clasifica

Clasifica: `normalize_tag` aislada; Event→JSONL→load con tmp_path; CLI create→list.

### Explica

¿Por qué un integration test puede descubrir un bug que cinco unit tests mockeados no ven?

---

## 11. Property-based testing con Hypothesis

Ejemplos manuales protegen puntos. Una property describe un invariante para una clase amplia de inputs.

```text
decode(encode(text)) == text
```

**Archivo ejecutable:**

```python
from hypothesis import given, strategies as st


@given(st.text())
def test_utf8_round_trip(text):
    encoded = text.encode("utf-8")

    decoded = encoded.decode("utf-8")

    assert decoded == text
```

Hypothesis genera examples y, si encuentra failure, intenta **shrinking**: reduce hacia un caso pequeño que todavía falla. Eso mejora reproducción; no prueba todos los strings posibles.

### 11.1 Una property independiente

**Mala property:**

```python
assert normalize_tag(value) == value.strip().lower()
```

Si el test repite el mismo algoritmo, puede duplicar el mismo error conceptual.

Properties más independientes:

- normalizar dos veces equivale a una vez (idempotence);
- correction preserva snapshot del original;
- serialize/deserialize conserva fields semánticos;
- replay reconstruye el mismo index derivado desde el mismo journal.

```python
@given(st.text())
def test_normalize_tag_is_idempotent(text):
    once = normalize_tag(text)

    assert normalize_tag(once) == once
```

### 11.2 Datetime y domain limits

Usa strategies restringidas a valores que el domain contract acepta. Generar naive datetimes para una API que exige aware no prueba round trip válido; prueba rejection si esa es la intención.

No se profundiza en composite strategies, stateful testing ni custom shrinkers.

### Diseña la property

Formula una property para migration que no compare simplemente output con otra implementación idéntica.

### Detecta el test débil

¿Por qué `assert deserialize(serialize(event)) == deserialize(serialize(event))` es tautológico?

---

## 12. Coverage es una señal

Coverage responde “¿qué lines/branches ejecutó la suite?”, no “¿el behavior es correcto?”.

```text
100% coverage
≠
100% correctness
```

Este test ejecuta la función pero no protege nada:

```python
def test_parse_runs():
    parse_record('{"id": "evt-1"}')
```

Una weak assertion, un input uniforme y una branch con behavior incorrecto pueden coexistir con coverage alto. No fijes porcentaje universal; usa gaps para formular riesgos y tests significativos.

### Detecta el test débil

¿Qué assertion o failure mode falta en `test_parse_runs`?

---

## 13. Regression test y debugging científico

Un regression test nace de una falla real o un riesgo específico:

```text
bug reproducible
↓
test falla por la causa observada
↓
fix mínimo
↓
test pasa
↓
suite relevante sigue pasando
```

Escribe el test antes o junto al fix cuando sea práctico. Si pasa antes del fix, no reproduce la regresión.

### 13.1 Ciclo de debugging

```text
observar symptom
↓ conservar evidencia
reproducir
↓ reducir input/environment
formular hipótesis
↓ medir una diferencia
confirmar o descartar
↓
corregir causa
↓ regression test
```

Cambiar cinco cosas simultáneamente impide saber cuál hipótesis era correcta.

### Reproduce

Un journal de 20 000 líneas falla una vez. ¿Qué artifacts conservarías antes de “limpiarlo”?

### Hipótesis

Dos hipótesis: línea truncada o encoding incorrecto. ¿Qué observación distingue ambas?

---

## 14. Reproducción mínima y tracebacks

Reduce a:

- dataset mínimo;
- una operation;
- interpreter/environment conocidos;
- input exacto;
- config/seed explícitos;
- command reproducible.

**Traceback pedagógico:**

```text
Traceback (most recent call last):
  File "cli.py", line 18, in main
    records = load(path)
  File "journal.py", line 42, in load
    records.append(parse_line(line))
  File "journal.py", line 17, in parse_line
    return json.loads(line)
json.decoder.JSONDecodeError: Expecting value: line 1 column 12 (char 11)
```

Lee de abajo hacia arriba para type/message y frame inmediato; después recorre call stack para descubrir cómo llegó ese input. `json.loads` es lugar del symptom. Root cause podría ser writer parcial, file equivocado, corruption o schema handling.

Exception chaining de PF-M6 puede conservar una causa técnica detrás de un error de domain. No ignores `The above exception was the direct cause...`.

### Traceback

¿Qué línea detecta el problema? ¿Qué evidencia necesitas antes de culpar al parser?

### Root cause

Si writer truncó la línea, ¿por qué “hacer parser más permisivo” ataca el symptom?

---

## 15. Debugger y `breakpoint()`

Inserta temporalmente `breakpoint()` en la reproducción mínima:

```python
def replay(records):
    state = {}
    for record in records:
        breakpoint()
        state[record["entity_id"]] = record
    return state
```

Comandos conceptuales de pdb:

| Acción | Comando común |
|---|---|
| inspeccionar expression | `p expression` |
| siguiente línea del frame | `n` |
| entrar a llamada | `s` |
| continuar | `c` |
| stack | `where` |
| salir | `q` |

El objetivo es contestar una pregunta: “¿qué record cambia esta key?”; no caminar cada línea sin hipótesis.

IDE debugger ofrece equivalentes visuales. Retira breakpoints accidentales antes del commit.

### Hipótesis

¿Qué breakpoint y expression usarías para distinguir duplicate ID de sort incorrecto?

### Decide

¿Debugger, test, print o log para inspeccionar una pure function de cinco líneas? Justifica la opción menos costosa que aporte evidencia.

---

## 16. Print debugging y logging

`print` sirve para exploración local rápida: confirmar que una branch corre o inspeccionar un value sintético. Se queda corto cuando necesitas levels, timestamps, context, múltiples components, configurable destination o redaction policy.

Logging no es “print profesional”. Es evidencia operacional con contrato de datos.

```text
qué operation
cuándo
qué outcome
qué synthetic ID
qué duración
qué error type
```

No uses logs como source of truth ni permitas que logging cambie domain behavior.

### Decide

¿Cuándo retirarías un print y cuándo lo convertirías en log?

---

## 17. `logging`, levels y traceback

**Ejemplo ejecutable:**

```python
import logging


logger = logging.getLogger("eidolon.replay")


def replay(record_count):
    logger.info(
        "replay completed",
        extra={"operation_id": "op-001", "record_count": record_count},
    )
    return record_count


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO, format="%(levelname)s %(message)s")
    replay(3)
```

Output estable relevante:

```text
INFO replay completed
```

- DEBUG: diagnóstico detallado activable;
- INFO: operación normal significativa;
- WARNING: condición inesperada recuperable;
- ERROR: operación falló, no necesariamente proceso completo;
- CRITICAL: falla de alcance extremo; úsalo raramente.

Logger emite records; handler decide destination y formatter presentation. PF-M9 solo necesita reconocer esas responsabilidades.

### 17.1 `logger.exception`

```python
try:
    load_journal(path)
except JournalCorruptionError:
    logger.exception("journal load failed", extra={"operation_id": operation_id})
    raise
```

Dentro de `except`, `logger.exception` incluye traceback. `logger.error(str(error))` normalmente pierde stack context. No loguees el mismo error en cada layer: elige la boundary que tiene context y ownership para reportarlo.

Un traceback técnico puede revelar local paths o values incluidos en exception messages. En una frontera real, controla destination/access y redacta messages sensibles; PF-M9 no diseña el Security Track.

### Log seguro

¿Qué level usarías para una línea corrupta puesta en quarantine bajo policy esperada? Explica según contrato, no por receta universal.

### Detecta el bug

Una layer captura, loguea y relanza; tres layers superiores hacen lo mismo. ¿Qué problema operacional produce?

---

## 18. Structured logging, correlation y privacidad

Structured logging trata un record como campos estables:

```text
operation_id=op-001
action=replay
record_count=42
schema_version=2
duration_ms=8.4
status=ok
error_type=null
```

No requiere una library externa. `extra`, un formatter propio posterior o una representación simple bastan para aprender el concepto.

### 18.1 Redaction por default

**Código incorrecto:**

```python
logger.info("message=%s token=%s", user_message, api_token)
```

Alternativa:

```python
logger.info(
    "message accepted",
    extra={
        "operation_id": operation_id,
        "synthetic_message_id": message_id,
        "message_length": len(user_message),
    },
)
```

No registres conversations completas, secrets, tokens, personal files ni sensitive payloads por default. Hashing no vuelve automáticamente anónimo un identificador sensible.

### 18.2 Operation ID

Un ID sintético relaciona:

```text
op-001 import started
op-001 validation completed
op-001 write completed
op-001 import completed
```

Permite seguir una operation sin distributed tracing ni payload.

### 18.3 Probar redaction con `caplog`

`caplog` es una fixture integrada de pytest para capturar log records durante un test:

```python
def observe_replay(operation_id, record_count, event_text):
    logger.info(
        "replay completed",
        extra={"operation_id": operation_id, "record_count": record_count},
    )


def test_replay_log_omits_payload(caplog):
    with caplog.at_level(logging.INFO):
        observe_replay(
            operation_id="op-1",
            record_count=2,
            event_text="SECRET-PAYLOAD",
        )

    assert "SECRET-PAYLOAD" not in caplog.text
    assert caplog.records[0].operation_id == "op-1"
```

La assertion positiva debe comprobar que el expected record sí ocurrió; solo afirmar que un secret no aparece podría pasar si no se emitió ningún log.

### Log seguro

Diseña fields para replay failure sin event text ni path personal completo.

### Review

¿`record_id` real está permitido siempre? Define una policy, no una suposición.

---

## 19. Testing y debugging de async

PF-M8 ya enseñó lifecycle. Aquí se obtiene evidencia sin dependency adicional:

**Archivo ejecutable:**

```python
import asyncio


async def fetch_synthetic():
    await asyncio.sleep(0)
    return "evt-1"


def test_fetch_synthetic_returns_id():
    assert asyncio.run(fetch_synthetic()) == "evt-1"
```

`asyncio.run` basta para tests sync que poseen un scenario async autónomo. Un plugin async puede justificarse más adelante si la suite necesita async fixtures/integration; no se agrega por moda.

### 19.1 Timeout

```python
async def scenario():
    try:
        async with asyncio.timeout(0.01):
            await asyncio.sleep(10)
    except TimeoutError:
        return "timeout"


def test_timeout_has_visible_policy():
    assert asyncio.run(scenario()) == "timeout"
```

No asserts duration exacta. El behavior es outcome timeout.

### 19.2 Cancellation cleanup

```python
def test_cancellation_cleans_staged_state():
    async def scenario():
        staged = set()

        async def operation():
            staged.add("src-1")
            try:
                await asyncio.sleep(10)
            finally:
                staged.discard("src-1")

        task = asyncio.create_task(operation())
        await asyncio.sleep(0)
        task.cancel()
        with pytest.raises(asyncio.CancelledError):
            await task
        assert staged == set()

    asyncio.run(scenario())
```

El owner crea, cancela y espera. Para un scenario acotado, al salir de `asyncio.run` no queda child work propio sin observar.

### 19.3 Fallas típicas

- `RuntimeWarning: coroutine was never awaited`: encuentra dónde se creó y abandonó coroutine.
- `Task exception was never retrieved`: identifica Task sin owner/await.
- timeout: separa slow operation de incorrect timeout policy.
- cancellation: comprueba propagación y cleanup.
- nondeterministic order: reconstruye output contract; no agregues sleeps.

### Detecta el bug

Un test hace `asyncio.create_task(operation())` y termina sin await. ¿Qué evidencia falta?

### Refactoriza

Sustituye `await asyncio.sleep(0.1)` usado como sincronización por await directo, TaskGroup, Queue o señal del contract ya enseñada.

---

## 20. Flaky tests e independencia

Flaky significa que el mismo código/input/environment aparente alterna pass/fail. Es un bug de suite o producto, no invitación a rerun hasta verde.

Fuentes comunes:

- wall clock y timezone no controlados;
- random/seed ocultos;
- order incidental de set/task;
- filesystem compartido;
- global state leak;
- test order dependency;
- network externa;
- timeout demasiado ligado a performance.

**Failure case:**

```python
cache = []


def test_a_populates_cache():
    cache.append("evt-1")


def test_b_reads_cache():
    assert cache == ["evt-1"]
```

`test_b` pasa solo después de A. Cada test debe arreglar su propia precondition y limpiar lo que posee.

Nunca dependas de Internet en la suite P0. Usa fake/stub local o tmp resources.

### Reproduce

Ejecuta B aislado y después suite completa. ¿Qué hipótesis confirma la diferencia?

### Detecta el bug

Un timeout test pasa en laptop y falla en una máquina lenta. ¿Qué property contractual debería afirmar?

---

## 21. Git como evidencia de cambio

PF-M9 no repite Git básico. Lo conecta con debugging:

```text
cambio coherente
↓ tests
commit pequeño
↓ review
revert posible
```

### 21.1 Commit atómico

Atómico significa una intención coherente, no una sola línea. Debe ser suficientemente pequeño para comprender, probar y revertir sin arrastrar un objetivo no relacionado.

Mejor:

```text
Preserve source file during schema migration
```

Débil:

```text
changes
```

Separa “añadir regression test” y “fix mínimo” si el workflow permite demostrar red/green claramente; mantenerlos juntos también puede ser coherente cuando no conviene integrar un test deliberadamente rojo. Explica la decisión.

### Review

Un commit mezcla rename masivo, feature y fix de corruption. ¿Por qué dificulta bisect y revert?

### Mensaje

Escribe un subject orientado a intención para cancellation cleanup.

---

## 22. Leer un diff y hacer code review

Review busca riesgo y claridad, no juzga a la persona.

Preguntas en orden útil:

1. ¿qué contract cambió?;
2. ¿es behavior correcto?;
3. ¿qué failure puede introducir?;
4. ¿qué test aporta evidencia?;
5. ¿cambia effects, error boundary o privacy?;
6. ¿naming/complexity permiten comprender?;
7. ¿crece coupling sin necesidad?;
8. ¿documentation y rollback son suficientes?

No discutas estilo menor mientras existe pérdida de datos. No exijas abstraction sin un problema real.

### 22.1 Review incremental

Un diff coherente suele revisarse mejor que miles de líneas con objetivos mezclados. No hay límite numérico universal. Reduce por intención, no para satisfacer una cuota.

### 22.2 Comentario accionable

Vago:

```text
Esto está mal.
```

Accionable:

```text
Si write_record falla después de truncar target, se pierde el journal anterior.
¿Podemos escribir a un temporary y promover solo después de success, con un failure test?
```

Señala location, risk, scenario y dirección verificable.

### Review

Prioriza estos hallazgos: variable mal nombrada; token en log; source overwrite; espacio extra.

---

## 23. Pull Request como artefacto técnico

Una PR autocontenida funciona incluso sin UI específica:

```markdown
## Problema
Replay cambia el orden de records con timestamps iguales.

## Cambio
Agrega tie-break por record_id y regression test.

## Cómo probar
python -m pytest -q tests/test_replay.py

## Riesgos
El orden observable cambia para journals con empate.

## Rollback
Revertir el commit; no existe migration de datos.
```

Incluye screenshots/logs solo si aportan evidencia, nunca payload sensible. Una PR personal sigue siendo un buen paquete de reasoning.

### Review

¿Qué falta si una PR dice solo “fix replay” y muestra suite verde?

---

## 24. ADR: registrar decisiones duraderas

Architecture Decision Record pequeño:

```markdown
# ADR-0001 — Usar JSONL append-only en EIDOLON 0.0a

## Status
Accepted

## Context
P0 necesita journal local inspeccionable y replay determinista.

## Decision
Usar JSONL UTF-8 versionado; derived indexes se reconstruyen.

## Consequences
Append y recovery son simples; queries complejas no son objetivo.

## Alternatives considered
Un único JSON array; database embebida.
```

Escribe ADR cuando una decision afecta estructura, impone constraint futuro, elige tradeoff significativo o será difícil reconstruir meses después. No para renombrar variable.

Status debe distinguir proposed/accepted/superseded según convention del repositorio. Un ADR nuevo puede supersede uno anterior sin borrar historia.

### ADR

¿La elección de orden estable de replay merece ADR o solo test/documentation? Justifica su alcance.

---

## 25. `git bisect`: reducir historia

Necesitas un known good commit y un known bad commit:

```bash
git bisect start
git bisect bad <bad-commit>
git bisect good <good-commit>
```

Git selecciona un midpoint. Ejecutas una reproducción y clasificas:

```bash
python -m pytest -q tests/test_replay_regression.py
git bisect good   # si el commit probado no tiene la regresión
git bisect bad    # si la reproduce
```

Al localizar el primer bad commit:

```bash
git bisect reset
```

`reset` aquí es el subcommand específico que termina la sesión de bisect y vuelve al estado previo; no equivale a recomendar `git reset --hard`.

Si el test es automatizable y exit code 0=good, nonzero=bad:

```bash
git bisect run python -m pytest -q tests/test_replay_regression.py
```

Un test que falla por dependency ausente clasifica mal commits. Primero valida environment y reproduction.

### Bisect

¿Qué condiciones debe cumplir el command para usar `git bisect run` con confianza?

---

## 26. `git revert`, `reset` y rollback

`git revert <commit>` crea un commit nuevo que aplica el inverse patch. Conserva historia y suele ser la opción segura para un cambio ya compartido.

`git reset` mueve una reference local a otro commit y puede cambiar index/worktree según mode. Reescribir published history rompe assumptions de collaborators. No uses `reset --hard` indiscriminadamente: puede destruir trabajo no guardado.

```text
shared bad commit
↓
git revert
↓
new auditable inverse commit
```

Rollback es también diseño:

- commits coherentes;
- source preserved;
- derived outputs reemplazables;
- changes sin irreversible side effects cuando sea posible;
- tests de state antes/después;
- ADR/PR que documentan consecuencias.

### Rollback

Un fix ya está en branch compartida y produce corruption. ¿Por qué revert gana frente a history rewrite?

### Decide

¿Qué evidencia ejecutarías después del revert?

---

## 27. Debugging de environment, filesystem, Unicode y time

### 27.1 Environment/packaging

Antes de cambiar imports, recoge:

```bash
python -c "import sys; print(sys.executable)"
python -m pip show eidolon
python -c "import eidolon; print(eidolon.__file__)"
pwd
```

Comprueba venv, version, package location, current working directory, missing dependency y module shadowing. No modifiques `sys.path` como reflejo.

### 27.2 Filesystem

Preguntas: ¿path resuelto?, ¿permissions?, ¿temporary existe?, ¿write parcial?, ¿source==target?, ¿derived data stale? Reproduce bajo `tmp_path`.

### 27.3 Unicode/time

Convierte edge case en regression input: combining characters, emoji, UTF-8, naive/aware mismatch, offset o DST. No “normalices todo” ni quites timezone para hacer pasar el test.

### Evidencia

Un import funciona globalmente pero falla en venv limpio. ¿Qué commands distinguen code bug de dependency declaration ausente?

---

## 28. Failure injection

Provoca deliberadamente failure en una frontera para comprobar recovery.

```python
def export_records(records, target, write_line):
    temporary = target.with_suffix(".tmp")
    try:
        with temporary.open("w", encoding="utf-8") as handle:
            for record in records:
                write_line(handle, record)
        temporary.replace(target)
    finally:
        temporary.unlink(missing_ok=True)
```

Un injected writer puede fallar después de N records:

```python
def fail_after_one():
    calls = 0

    def write_line(handle, record):
        nonlocal calls
        calls += 1
        if calls > 1:
            raise OSError("synthetic write failure")
        handle.write(record + "\n")

    return write_line
```

El test debe comprobar target anterior preservado, temporary ausente y exception visible. Otros injections: corrupt line, fixed clock y async timeout. No construyas framework de fault injection.

### Failure injection

¿Qué failure point distingue cleanup correcto de happy path únicamente?

---

## 29. Anti-patterns de evidencia

### Testing

- test sin assertion relevante;
- assertion sobre implementation incidental;
- sleeps para sincronizar;
- Internet o personal data reales;
- test-order dependency;
- overmocking;
- snapshot enorme sin intención;
- coverage como única meta;
- `pytest.raises(Exception)` que acepta cualquier failure.

### Debugging

- cambiar múltiples variables a la vez;
- borrar evidence antes de conservarla;
- asumir root cause por intuición;
- ignorar traceback;
- agregar sleeps;
- desactivar tests;
- capturar exceptions para “hacerlo funcionar”;
- rewrite masivo antes de reproducir.

### Logging

- secrets/private payload;
- exception duplicada en cada layer;
- records sin operation/context;
- DEBUG permanente para todo;
- logs como source of truth;
- domain mutation desde logging.

### Review

- estilo menor antes que data loss;
- abstractions sin problem;
- aprobar sin ejecutar evidence;
- comentarios vagos;
- feature y refactor enorme mezclados;
- ignorar rollback.

### Revisión rápida

Elige un anti-pattern de cada grupo y escribe: risk observable, reproduction y correction mínima.

---

## 30. Suite pedagógica de EIDOLON 0.0a

El journal P0 integra objetos, JSONL, replay, migrations, atomic export y coordinator async. Cada test existe por un riesgo:

| Test | Riesgo protegido | Alcance probable |
|---|---|---|
| Unicode round trip | pérdida/cambio de texto | integration/property |
| aware timezone round trip | instant ambiguo | unit/integration |
| SourceRecord permanece separado | source reinterpretado/destruido | invariant/integration |
| Event serialization | field/schema drift | unit |
| checksum verification | modificación no detectada | negative integration |
| replay determinista | state/order diferente | integration |
| Correction preserva Event | source mutation | invariant unit |
| duplicate IDs | silent overwrite | negative/invariant |
| v1→v2 migration | source/data loss | integration |
| corrupted JSONL | failure sin línea/policy | negative integration |
| atomic export failure | target parcial | failure injection |
| cancellation cleanup | staged/temporary huérfano | async integration |
| deterministic summary | scheduler filtra output | async invariant |
| CLI instalada | entry point/subcommand depende del cwd | smoke/end-to-end local |

No todo necesita unit + integration + property. Selecciona la capa que observa el risk con menor costo y suficiente fidelity.

### 30.1 Integration round trip

```python
def test_journal_round_trip_preserves_semantics(tmp_path):
    path = tmp_path / "journal.jsonl"
    event = Event("evt-1", "México 🦋", aware_time)

    append_event(path, event)
    loaded = load_events(path)

    assert loaded == [event]
```

No assertions sobre whitespace interno de JSON si canonical formatting no es contract.

### 30.2 Corrupted line

```python
def test_load_identifies_corrupted_line(tmp_path):
    path = tmp_path / "journal.jsonl"
    path.write_text('{"id":"evt-1"}\n{broken}\n', encoding="utf-8")

    with pytest.raises(JournalCorruptionError) as captured:
        load_events(path)

    assert captured.value.line_number == 2
```

Verifica domain error y ubicación; no acepta cualquier parse failure.

### Diseña el test

¿Qué test demuestra que derived index se reconstruye y no se trata como source of truth?

### Invariante

¿Qué tres properties deben mantenerse durante schema migration?

---

## 31. Caso integrado de debugging: replay cambia de orden

**Symptom:** después de migration, dos ejecuciones producen distinto order para records con el mismo timestamp.

No reescribas replay. Sigue el proceso.

### 31.1 Reproduce

Reduce a tres records sintéticos:

```python
records = [
    {"record_id": "rec-b", "recorded_at": "2026-01-01T00:00:00+00:00"},
    {"record_id": "rec-a", "recorded_at": "2026-01-01T00:00:00+00:00"},
    {"record_id": "rec-c", "recorded_at": "2026-01-02T00:00:00+00:00"},
]
```

Ejecuta desde dos input orders. Conserva output exacto.

### 31.2 Observaciones

Registra sin payload:

- input position;
- record ID sintético;
- parsed timestamp;
- sort key;
- output position.

### 31.3 Hipótesis

H1: migration cambia timestamp.  
H2: sort usa timestamp sin tie-break.  
H3: replay parte de un set/dict construido desde input no estable.

Para cada una, define una observación que la falsaría.

### 31.4 Regression test antes del fix

```python
def test_replay_order_breaks_timestamp_ties_by_record_id():
    first = replay([record_b, record_a])
    second = replay([record_a, record_b])

    assert ids(first) == ["rec-a", "rec-b"]
    assert ids(second) == ["rec-a", "rec-b"]
```

Debe fallar en la implementación afectada. Si el product contract no define tie-break, primero hay que decidir/documentarlo.

### 31.5 Fix mínimo razonado

Después de confirmar H2, cambia key de `recorded_at` a `(recorded_at, record_id)`. Ejecuta regression test, replay suite y migration integration. No alteres serializer ni modelo temporal.

### 31.6 Evidencia final

- reproducción roja antes;
- diff acotado;
- test verde después;
- input source idéntico;
- ADR o contract note si order público cambió.

### Hipótesis

¿Qué output descartaría H1 sin cambiar código?

### Review

¿Por qué ordenar por input position resolvería una ejecución pero no el contract entre permutaciones?

---

## 32. Caso integrado de logging seguro

Objetivo: investigar replay/import sin registrar Event text.

```python
logger.info(
    "replay completed",
    extra={
        "operation_id": "op-replay-001",
        "record_count": 42,
        "schema_version": 2,
        "duration_ms": 8.4,
        "status": "ok",
        "error_type": None,
    },
)
```

Failure boundary:

```python
def replay_with_observation(path, operation_id):
    try:
        return replay(path)
    except JournalCorruptionError as error:
        logger.exception(
            "replay failed",
            extra={
                "operation_id": operation_id,
                "status": "failed",
                "error_type": type(error).__name__,
                "line_number": error.line_number,
            },
        )
        raise
```

Line number en synthetic/local test no es payload. En producción futura, privacy policy deberá revisar también metadata y paths.

### Debugging con logs

Pregunta: “¿la falla ocurre antes o después de migration?” Agrega un record en cada boundary con same operation ID y status; no 50 logs arbitrarios.

### Log seguro

¿Qué field añadirías para distinguir schema v1/v2? ¿Qué field rechazarías aunque facilite debugging?

---

## 33. Caso integrado de review incremental

**Código problemático:**

```python
def migrate_and_export(source, target):
    records = []
    for line in source.read_text().splitlines():
        data = json.loads(line)
        if data["schema_version"] == 1:
            data["schema_version"] = 2
        records.append(data)
    target.write_text("\n".join(json.dumps(item) for item in records))
    print(records)
    logger.info("records=%s", records)
```

Risks:

- parse/validation/migration/export/printing/logging mezclados;
- default encoding implícito;
- no line context;
- target puede quedar parcial;
- payload completo en print/log;
- source==target no se rechaza;
- difícil inyectar write failure;
- no rollback story.

### 33.1 Refactor incremental

1. agrega tests de source preservation, Unicode y corrupt line;
2. extrae pure `migrate_record`;
3. separa `load_records` con line-number error;
4. reutiliza atomic export de PF-M6/PF-M7;
5. sustituye payload log por operation metadata;
6. elimina print;
7. ejecuta suite en cada paso.

No se reescribe todo de una vez. Cada commit tiene intención y rollback local.

### 33.2 Fronteras resultantes

```text
load_records(source) → records
migrate_record(record) → new record
serialize(records) → text
atomic_export(target, text) → effect
observe(operation metadata) → log
```

### Review

Ordena los risks por impacto. ¿Qué regression test escribirías primero?

### Rollback

¿Qué commit sería seguro revertir sin devolver el payload logging?

---

## 34. Ejercicios guiados

Cada ejercicio exige predecir antes de ejecutar. Las soluciones privilegian contrato y evidencia sobre syntax trivia.

### 34.1 Primer test pytest

**Objetivo:** proteger normalización observable.  
**Input:** `normalize_tag("  Work  ")`.  
**Predice:** ¿qué nombre debe tener el file/function para discovery?

**Solución — `tests/test_tags.py`:**

```python
from eidolon.tags import normalize_tag


def test_normalize_tag_trims_outer_space():
    assert normalize_tag("  Work  ") == "work"
```

**Razonamiento:** el nombre describe behavior; assertion no mira helper privada.  
**Criterio:** `python -m pytest -q tests/test_tags.py` reporta un pass.  
**Variación:** preserva espacio interno y agrega un caso que lo demuestre.

### 34.2 Positive y negative

**Objetivo:** distinguir success de rejection contractual.  
**Input:** status `active` y `unknown`.  
**Predice:** ¿por qué “cualquier exception” es insuficiente?

**Solución:**

```python
def test_parse_status_accepts_active():
    assert parse_status("active") is Status.ACTIVE


def test_parse_status_rejects_unknown():
    with pytest.raises(ValueError, match="unsupported status"):
        parse_status("unknown")
```

**Razonamiento:** cada test tiene un outcome; negative especifica error class.  
**Criterio:** cambiar a `NameError` hace fallar el negative.  
**Variación:** agrega whitespace solo si el contract decide aceptarlo/rechazarlo.

### 34.3 Boundary cases

**Objetivo:** particionar lookup.  
**Input:** index vacío, un ID presente, ID ausente, duplicate.  
**Predice:** ¿duplicate se rechaza o reemplaza? Consulta contract antes de testear.

**Solución de diseño:**

```python
def test_find_in_empty_index_returns_none(): ...
def test_find_existing_id_returns_event(): ...
def test_find_missing_id_returns_none(): ...
def test_build_index_rejects_duplicate_id(): ...
```

**Razonamiento:** las partitions cambian control flow o invariant.  
**Criterio:** cada test puede fallar por una behavior distinta y documentada.  
**Variación:** transfiere la partición a tags sin coincidencias/una/varias.

### 34.4 `pytest.raises` específico

**Objetivo:** probar schema futuro.  
**Input:** `schema_version=99`.  
**Predice:** ¿qué ocurre si parser produce `KeyError` accidental?

**Solución:**

```python
def test_future_schema_reports_unsupported_version():
    with pytest.raises(UnsupportedSchemaError) as captured:
        parse_record({"schema_version": 99})

    assert captured.value.version == 99
```

**Razonamiento:** domain error y attribute son más estables que cualquier exception.  
**Criterio:** `KeyError` no hace pasar el test.  
**Variación:** usa `match` solo si message es parte del contract.

### 34.5 Parametrization coherente

**Objetivo:** consolidar Unicode cases con una assertion común.  
**Input:** case, accents y emoji.  
**Predice:** ¿todos esperan round trip exacto?

**Solución:**

```python
@pytest.mark.parametrize("text", ["hello", "pingüino", "🦋"], ids=["ascii", "accent", "emoji"])
def test_text_round_trip(text):
    assert decode_text(encode_text(text)) == text
```

**Razonamiento:** inputs forman una clase de round trip.  
**Criterio:** pytest reporta ID útil al fallar.  
**Variación:** separa invalid bytes porque contract/outcome cambia.

### 34.6 Fixture explícita

**Objetivo:** compartir un Event pequeño sin hidden application.  
**Input:** dos tests necesitan mismo valid Event.  
**Predice:** ¿qué setup pertenece a fixture y cuál al test?

**Solución:**

```python
@pytest.fixture
def event():
    return Event("evt-1", "synthetic text")


def test_event_id(event):
    assert event.event_id == "evt-1"
```

**Razonamiento:** fixture expresa una precondition pequeña.  
**Criterio:** leer test revela fields relevantes sin navegar cinco fixtures.  
**Variación:** inline Event si solo un test lo usa.

### 34.7 `tmp_path` y cleanup

**Objetivo:** probar file effect aislado.  
**Input:** un record Unicode.  
**Predice:** ¿por qué no usar `~/journal.jsonl`?

**Solución:**

```python
def test_export_writes_utf8(tmp_path):
    target = tmp_path / "export.jsonl"

    export_records([{"text": "México"}], target)

    assert "México" in target.read_text(encoding="utf-8")
```

**Razonamiento:** file es real, location aislada.  
**Criterio:** test no lee/escribe fuera de `tmp_path`.  
**Variación:** inyecta failure y comprueba temporary cleanup.

### 34.8 Monkeypatch de environment

**Objetivo:** aislar default y configured mode.  
**Input:** `EIDOLON_MODE` puede existir en la shell.  
**Predice:** ¿qué test falla si variable externa contamina suite?

**Solución:**

```python
def test_mode_default(monkeypatch):
    monkeypatch.delenv("EIDOLON_MODE", raising=False)
    assert journal_mode() == "safe"


def test_mode_configured(monkeypatch):
    monkeypatch.setenv("EIDOLON_MODE", "preview")
    assert journal_mode() == "preview"
```

**Razonamiento:** cada test controla su environment.  
**Criterio:** funciona igual con variable previamente presente/ausente.  
**Variación:** prueba invalid value según contract.

### 34.9 Clock controlado

**Objetivo:** remover wall-clock flakiness.  
**Input:** receipt necesita recorded time.  
**Predice:** ¿qué diseño hace dependency visible?

**Solución:**

```python
def test_receipt_uses_clock():
    fixed = datetime(2026, 2, 3, tzinfo=timezone.utc)

    receipt = make_receipt("rec-1", now=lambda: fixed)

    assert receipt.recorded_at == fixed
```

**Razonamiento:** explicit parameter supera patch de internals para code nuevo.  
**Criterio:** no usa tolerance temporal.  
**Variación:** monkeypatch una legacy module boundary y compara coupling.

### 34.10 Property introductoria

**Objetivo:** explorar Unicode idempotence.  
**Input:** strings generados por Hypothesis.  
**Predice:** ¿qué minimal failing example podría descubrir?

**Solución:**

```python
from hypothesis import given, strategies as st


@given(st.text())
def test_normalization_is_idempotent(text):
    once = normalize_for_search(text)
    assert normalize_for_search(once) == once
```

**Razonamiento:** property no replica el algoritmo.  
**Criterio:** ejecuta múltiples generated examples y preserva invariant.  
**Variación:** restringe strategy si domain prohíbe control characters y prueba rejection aparte.

### 34.11 Failure injection

**Objetivo:** proteger target previo ante write failure.  
**Input:** target `old` y writer que falla después de una línea.  
**Predice:** ¿qué artifacts deben quedar?

**Solución:**

```python
def test_export_failure_preserves_target(tmp_path):
    target = tmp_path / "journal.jsonl"
    target.write_text("old\n", encoding="utf-8")

    with pytest.raises(OSError, match="synthetic"):
        export_records(["a", "b"], target, write_line=fail_after_one())

    assert target.read_text(encoding="utf-8") == "old\n"
    assert not target.with_suffix(".tmp").exists()
```

**Razonamiento:** assertions cubren error, preservation y cleanup.  
**Criterio:** happy path alone no puede satisfacerlo.  
**Variación:** falla antes de primera write.

### 34.12 Reproducir un bug

**Objetivo:** reducir nondeterministic replay a dos records.  
**Input:** timestamp empatado e IDs b/a.  
**Predice:** lista hipótesis antes de tocar sort.

**Solución de reproducción:**

```python
def test_replay_tie_order_is_stable():
    left = replay([record_b, record_a])
    right = replay([record_a, record_b])
    assert ids(left) == ids(right) == ["rec-a", "rec-b"]
```

**Razonamiento:** cambia solo input order y fija expected contract.  
**Criterio:** falla antes del fix por la regresión observada.  
**Variación:** agrega tercer timestamp no empatado para proteger primary key.

### 34.13 Debugger dirigido

**Objetivo:** descubrir qué sort key difiere.  
**Input:** reproducción anterior.  
**Predice:** ¿qué variables inspeccionarás?

**Solución práctica:**

```bash
python -m pdb -m pytest -q tests/test_replay.py -x
```

Coloca `breakpoint()` temporal junto a construcción de key; usa `p record["record_id"]`, `p key`, `where` y `c`.

**Razonamiento:** breakpoint responde H2, no inspecciona todo el programa.  
**Criterio:** registra evidencia de key empatada y retira breakpoint antes de commit.  
**Variación:** usa IDE debugger con mismas observaciones.

### 34.14 Structured/redacted logging

**Objetivo:** observar replay sin payload.  
**Input:** success de 3 records.  
**Predice:** ¿qué fields bastan para correlación?

**Solución:**

```python
logger.info(
    "replay completed",
    extra={
        "operation_id": "op-1",
        "record_count": 3,
        "schema_version": 2,
        "status": "ok",
    },
)
```

**Razonamiento:** metadata responde operation/outcome sin text.  
**Criterio:** no contiene record payload, token ni personal path.  
**Variación:** agrega failure record con `error_type` y `logger.exception` en una sola boundary.

### 34.15 Regression test

**Objetivo:** proteger una corrupt last line previamente aceptada.  
**Input:** valid line seguida de fragmento JSON.  
**Predice:** ¿qué error/line debe observar caller?

**Solución:**

```python
def test_truncated_last_line_reports_line_number(tmp_path):
    path = tmp_path / "journal.jsonl"
    path.write_text('{"id":"ok"}\n{"id":', encoding="utf-8")

    with pytest.raises(JournalCorruptionError) as captured:
        load_events(path)

    assert captured.value.line_number == 2
```

**Razonamiento:** conserva minimal input que reprodujo bug.  
**Criterio:** test falla en old behavior y pasa solo con policy correcta.  
**Variación:** diferencia empty trailing line permitida de truncated JSON.

### 34.16 Revisar un diff

**Objetivo:** priorizar correctness sobre style.  
**Input:** diff cambia UTF-8 a default encoding y renombra variable.  
**Predice:** ¿qué finding bloquea merge?

**Solución razonada:** comenta primero portability/data preservation, pide Unicode regression test y explicit encoding. El rename es non-blocking si claro.

**Criterio:** comentario incluye location, risk, reproducible scenario y evidence solicitada.  
**Variación:** agrega token en log y reordena prioridades.

### 34.17 Escribir ADR

**Objetivo:** documentar JSONL append-only en P0.  
**Input:** alternatives JSON array y embedded database.  
**Predice:** ¿qué future constraint introduce?

**Solución:** usa Title/Status/Context/Decision/Consequences/Alternatives; limita decisión a EIDOLON 0.0a y declara derived indexes reconstruibles.

**Razonamiento:** la elección afecta persistence structure y merece history.  
**Criterio:** un lector entiende why, tradeoffs y scope sin conversación original.  
**Variación:** redacta un superseding ADR sin editar el accepted original.

### 34.18 Simular `git bisect`

**Objetivo:** localizar primer bad commit con test estable.  
**Input:** repo didáctico con tag `good` y HEAD bad.  
**Predice:** ¿qué significa exit 0/1?

**Solución:**

```bash
git bisect start
git bisect bad HEAD
git bisect good good
git bisect run python -m pytest -q tests/test_replay_regression.py
git bisect reset
```

**Razonamiento:** binary search reduce commits; test clasifica behavior, no fecha.  
**Criterio:** identifica first bad commit y restaura checkout.  
**Variación:** documenta qué hacer si un midpoint no puede probarse (`git bisect skip`) sin abusar.

### 34.19 Revertir un cambio

**Objetivo:** rollback auditable en history compartida.  
**Input:** bad commit aislado y clean worktree.  
**Predice:** ¿qué history queda después?

**Solución:**

```bash
git revert <bad-commit>
python -m pytest -q
git log --oneline -3
```

**Razonamiento:** nuevo commit conserva bad decision e inverse; no reescribe collaborators.  
**Criterio:** regression desaparece, suite pasa y revert commit es visible.  
**Variación:** si revert tiene conflict, resuélvelo entendiendo current contract; no uses reset destructivo.

---

## 35. Ejercicios independientes

### Diseño de tests

1. Reescribe un test de private dict como lookup behavior.
2. Diseña positive/negative para Event status.
3. Particiona un list filter en cero/uno/varios matches.
4. Añade boundaries Unicode: accent, emoji y combining sequence.
5. Prueba aware datetime y rejection de naive según contract.
6. Formula invariant “Correction preserva Event”.
7. Demuestra duplicate IDs no causan silent overwrite.
8. Diseña replay equivalence desde mismo journal.

### pytest

9. Crea proyecto `src/` mínimo con dependency test declarada.
10. Ejecuta un test aislado por node ID.
11. Sustituye `raises(Exception)` por domain error específico.
12. Inspecciona un error attribute con `pytest.raises`.
13. Parametriza una clase de valid IDs con IDs legibles.
14. Separa de la tabla un case con contract distinto.
15. Diseña fixture pequeña y después elimina otra fixture innecesaria.
16. Implementa yield fixture con cleanup verificable.
17. Migra un filesystem test global a `tmp_path`.
18. Comprueba source/target distintos en atomic export.

### No determinismo y doubles

19. Inyecta clock explícito a receipt function.
20. Prueba default/configured environment con monkeypatch.
21. Parchea una module clock boundary y explica dónde lookup ocurre.
22. Reemplaza mock de JSON/filesystem por integration test real.
23. Implementa FakeEventStore y prueba behavior, no calls.
24. Encuentra un overmocked test y enumera paths que no ejecuta.
25. Reproduce test-order dependency ejecutando test aislado.
26. Elimina sleep de sincronización de un async test.

### Property y coverage

27. Formula Unicode encode/decode property.
28. Formula normalization idempotence.
29. Crea strategy de aware datetimes aceptados.
30. Conserva semantic fields en Event round trip.
31. Explica shrinking desde un failing example observado.
32. Detecta una property tautológica.
33. Encuentra line con coverage pero assertion ausente.
34. Propón test significativo sin perseguir porcentaje.

### Debugging

35. Reduce corrupted JSONL de 1 000 líneas a una línea.
36. Escribe dos hipótesis para `JSONDecodeError`.
37. Lee traceback y separa detecting frame/root-cause boundary.
38. Usa `breakpoint()` para inspeccionar sort key.
39. Usa `where` para explicar call path.
40. Convierte el minimal reproduction en regression test.
41. Inyecta writer failure después de N records.
42. Diagnostica package presente globalmente y ausente en venv.
43. Convierte un DST/Unicode symptom en fixed input.

### Logging y privacidad

44. Elige levels para success, quarantine y failed operation.
45. Usa `logger.exception` en una boundary y evita duplicates.
46. Diseña fields para import con operation ID.
47. Redacta message text y personal path.
48. Usa logs para distinguir validation/write boundary.
49. Revisa si synthetic IDs pueden considerarse no sensibles bajo contract.
50. Demuestra que logging failure no debe cambiar domain result.

### Git y review

51. Divide un diff mezclado en commits por intención.
52. Escribe tres commit subjects orientados a behavior.
53. Revisa un diff por data loss antes que style.
54. Redacta comentario con risk/reproduction/evidence.
55. Escribe PR con problema/cambio/test/risk/rollback.
56. Decide si una choice merece ADR.
57. Escribe ADR y una superseding decision.
58. Prepara un regression test apto para bisect.
59. Simula bisect manual en historia pequeña.
60. Revierte un commit y ejecuta suite relevante.

---

## 36. Preguntas de comprensión

1. ¿Por qué un test es evidencia limitada y no garantía absoluta?
2. ¿Qué diferencia existe entre behavior e implementation detail?
3. ¿Cuándo AAA revela una operation mal diseñada?
4. ¿Qué distingue positive, negative, boundary e invariant tests?
5. ¿Por qué un negative test debe especificar failure contract?
6. ¿Cómo eliges partitions sin enumerar todos los inputs?
7. ¿Por qué `pytest.raises(Exception)` puede ocultar un bug?
8. ¿Cuándo parametrization mejora intención y cuándo la oculta?
9. ¿Qué hace que una fixture sea explícita?
10. ¿Qué ownership tiene una yield fixture?
11. ¿Por qué `tmp_path` mejora aislamiento?
12. ¿Cuándo inyectar clock supera monkeypatch?
13. ¿Qué parchea realmente `monkeypatch.setattr`?
14. ¿Qué diferencia práctica existe entre fake, stub y mock?
15. ¿Por qué overmocking produce confianza falsa?
16. ¿Qué pregunta distinta responden unit e integration tests?
17. ¿Qué tradeoff añade end-to-end?
18. ¿Qué ventaja aporta property-based testing?
19. ¿Qué hace independiente a una property?
20. ¿Qué es shrinking y qué no garantiza?
21. ¿Por qué 100% coverage no implica correctness?
22. ¿Qué hace útil a un regression test?
23. ¿Cuál es el primer objetivo ante una falla desconocida?
24. ¿Cómo distingues symptom y root cause?
25. ¿Qué evidencia vuelve mínima una reproducción?
26. ¿Cómo lees traceback sin culpar al último frame automáticamente?
27. ¿Cuándo debugger aporta más que logs?
28. ¿Cuándo print es suficiente?
29. ¿Por qué logging no es source of truth?
30. ¿Qué información no debe aparecer en logs EIDOLON?
31. ¿Qué aporta operation ID sin distributed tracing?
32. ¿Por qué `logger.exception` se usa dentro de `except`?
33. ¿Cómo pruebas cancellation cleanup sin sleep arbitrario?
34. ¿Por qué un flaky test debe tratarse como bug?
35. ¿Qué hace independiente a un test?
36. ¿Qué hace que un commit sea atómico/revisable?
37. ¿Qué preguntas priorizas al leer un diff?
38. ¿Qué debe contener una PR autocontenida?
39. ¿Cuándo una decisión merece ADR?
40. ¿Qué preconditions necesita `git bisect`?
41. ¿Por qué un test ambientalmente roto clasifica mal bisect?
42. ¿Por qué `git revert` suele ser más seguro en history compartida?
43. ¿Qué diferencia práctica existe entre revert y reset?
44. ¿Cómo influye rollback en diseño antes del deployment?

---

## 37. Mini challenge — Evidence-driven EIDOLON 0.0a

Este challenge final integra únicamente PF-M1–PF-M9. Parte de un package sintético deliberadamente defectuoso.

### 37.1 Artefacto

```text
eidolon-evidence-challenge/
├── pyproject.toml
├── README.md
├── CHANGELOG.md
├── THREAT_NOTES.md
├── ADR/
├── src/eidolon/
│   ├── model.py
│   ├── journal.py
│   ├── migration.py
│   ├── replay.py
│   ├── import_coordinator.py
│   ├── observation.py
│   └── cli.py
├── tests/
│   ├── test_model.py
│   ├── test_journal.py
│   ├── test_migration.py
│   ├── test_replay.py
│   ├── test_import_coordinator.py
│   └── test_observation.py
└── history/
    └── README.md
```

Declara pytest/Hypothesis como optional development dependencies. Runtime usa standard library.

### 37.2 Contrato acumulado de EIDOLON 0.0a

Antes de depurar, la baseline debe hacer visible el contrato construido durante PF-M1–PF-M8:

- `SourceRecord`, `Event`, `Claim` y `Correction` son dataclasses distintas;
- una Correction se agrega y nunca reescribe silenciosamente su target;
- JSONL append-only incluye `schema_version`, IDs estables, `recorded_at`, `valid_time` aware y checksum verificable;
- replay reconstruye índices in-memory derivados por ID, persona, rango temporal y tags;
- una línea truncada, JSON inválido o checksum inconsistente produce evidence localizada y una policy de quarantine/reporte;
- la CLI instalada despacha `init`, `append`, `list`, `show`, `correct`, `export`, `verify` y `replay` mediante funciones pequeñas;
- export/migration preserva source y usa temporal + validation + replace dentro de las garantías enseñadas;
- el coordinator async opera solo sobre I/O sintético y conserva ownership/cancellation.

No necesitas implementar arquitectura posterior. Este inventario evita que el challenge final pruebe únicamente utilities aisladas mientras omite el build canónico.

### 37.3 Bugs deliberados

La baseline debe contener:

1. serialization que pierde un Unicode edge case;
2. naive/aware comparison incorrecta;
3. replay sin stable tie-break;
4. duplicate ID que sobrescribe silenciosamente;
5. loader que no informa corrupted JSONL line;
6. migration que puede sobrescribir source;
7. atomic export que deja temporary/target parcial bajo injected failure;
8. async cancellation que deja staged ID;
9. log que expone Event text;
10. test dependiente de order/environment;
11. historia Git preparada con una replay regression;
12. checksum que no detecta un record modificado.

No arregles todos antes de reproducirlos.

### 37.4 Baseline reproducible

```bash
python -m pip install -e '.[test]'
python -m pytest -q
```

Guarda failed test names y tracebacks. El README enumera environment, Python version y exact commands; no copia personal paths ni payload.

### 37.5 Test design requerido

Implementa y justifica:

- positive Event round trip;
- SourceRecord separado de su Event derivado;
- negative unsupported schema;
- negative checksum mismatch con line number;
- boundaries empty/one/duplicate/Unicode/time;
- Correction/source invariant;
- parametrization de una clase coherente;
- fixture pequeña;
- `tmp_path` para files;
- monkeypatch de clock o environment;
- Hypothesis property introductoria;
- integration replay/migration;
- failure injection;
- async timeout/cancellation cleanup;
- deterministic summary.
- smoke test de los ocho subcommands desde el entry point instalado.

### 37.6 Debugging protocol

Para dos bugs:

1. reproduce con minimal input;
2. conserva traceback/output;
3. escribe al menos dos hipótesis;
4. usa breakpoint/debugger o logs dirigidos;
5. identifica root cause;
6. agrega regression test rojo;
7. aplica fix mínimo;
8. ejecuta suite relevante y completa.

### 37.7 Logging policy

Emite records con:

- `operation_id`;
- action/status;
- record count;
- schema version;
- duration;
- error type cuando aplique.

Prohíbe Event/Claim text, tokens, secrets y personal paths. Prueba con `caplog` o handler in-memory que payload sintético marcado como secret no aparece.

### 37.8 Git, documentación y supply-chain evidence

- commits pequeños por intención;
- commit messages orientados a behavior;
- diff review escrito con findings priorizados;
- PR local/autocontenida;
- ADR corto para stable replay order o JSONL policy;
- `git bisect` manual/automatizado sobre historia preparada;
- `git revert` de un bad commit en branch de práctica;
- suite después del revert;
- nada de history rewrite destructivo.

El README documenta instalación limpia, comandos y límites. `CHANGELOG.md` registra los fixes relevantes. `THREAT_NOTES.md` contiene únicamente controles P0: datos sintéticos, archivos que no deben versionarse, payloads prohibidos en logs, dependencies directas/dev y provenance mínima. Conserva también el comando y resultado de un secret scan local sobre archivos tracked; un resultado limpio es evidencia acotada, no prueba ausencia universal de secrets. SBOM, firmas y CI de supply chain se profundizan en D12.

### 37.9 Failure checks mínimos

`unicode_time.py`: preserva Unicode y aware instant.  
`corruption.py`: reporta línea corrupta.  
`checksum.py`: detecta contenido modificado y conserva la línea fuente.  
`migration_failure.py`: source idéntico, target anterior seguro, temporary limpio.  
`replay_regression.py`: mismas IDs/order desde input permutations.  
`cancellation.py`: caller observa cancellation y staged queda vacío.  
`redaction.py`: logs no contienen sentinel secret payload.  
`independence.py`: tests pasan aislados y en suite.

### 37.10 Comprobaciones

```bash
python -m compileall -q src tests
python -m pytest -q
python -m pytest -q tests/test_replay.py
python -m pytest -q tests/test_import_coordinator.py
eidolon --help
git status --short
git log --oneline --decorate -10
```

No fija coverage universal. Si ejecutas coverage, úsalo para investigar gaps, no como único PASS.

### 37.11 Output contractual

```text
PF-M9 evidence challenge: PASS
```

Solo se imprime después de suite verde, redaction/secret-hygiene checks, bisect evidence, ADR y rollback verification.

### 37.12 Criterio de aprobación

- cada fix tiene reproducción/regression test;
- positive/negative/boundary/invariant están representados;
- pytest/parametrization/fixtures/tmp_path/monkeypatch son correctos;
- property no replica implementation;
- corrupted line y failure injection tienen policy visible;
- SourceRecord permanece separado de Event y checksum mismatch se detecta;
- los ocho subcommands llegan a una función de aplicación sin convertir la CLI en dominio;
- source no se destruye;
- async work tiene owner y cancellation cleanup;
- summary es determinista;
- logs están redactados/correlacionados;
- commits son revisables/revertibles;
- PR explica test/risk/rollback;
- ADR registra tradeoff significativo;
- README, CHANGELOG y threat notes P0 son reproducibles y no contienen datos reales;
- bisect localiza first bad commit;
- revert conserva history y restaura behavior;
- project funciona desde environment limpio.

### 37.13 Límites

No uses backend, database, frontend, networking real, Docker obligatorio, CI/CD avanzado, distributed tracing, production observability, fuzzing avanzado, mutation testing, LLMs ni embeddings.

---

## 38. Resumen

- Tests protegen behavior/invariants concretos; no garantizan ausencia de bugs.
- AAA hace visible precondition, operation y observation.
- Implementation-detail tests frenan refactors correctos.
- pytest aporta discovery y herramientas; no sustituye test design.
- Positive, negative, boundary e invariant responden preguntas distintas.
- `pytest.raises` debe especificar error contractual.
- Parametrization agrupa una clase coherente de casos.
- Fixtures aclaran setup compartido cuando permanecen pequeñas.
- Yield fixture posee teardown; `tmp_path` aísla filesystem.
- Controla clock/environment en la frontera.
- Monkeypatch es herramienta temporal, no arquitectura.
- Fake/stub/mock tienen costos; overmocking reduce fidelity.
- Unit/integration/e2e equilibran velocidad, fidelidad y diagnóstico.
- Property-based testing explora una clase de inputs; shrinking reduce failures.
- Coverage es señal de ejecución, no correctness.
- Regression test sigue bug → reproducción roja → fix → verde.
- Debugging formula hipótesis y busca evidencia, no cambios aleatorios.
- Traceback muestra call path; último frame puede ser symptom.
- Debugger debe responder preguntas específicas.
- Print sirve localmente; logging ofrece records configurables/contextuales.
- Logs de EIDOLON usan metadata redactada y operation IDs.
- `logger.exception` conserva traceback en una error boundary.
- Async tests esperan/cancelan Tasks y no dependen de sleeps.
- Flaky tests e order dependencies son bugs.
- Commits pequeños mejoran review, bisect y revert.
- Review prioriza correctness, data/privacy y rollback.
- PR conserva problem/change/test/risk/rollback.
- ADR registra decisiones duraderas, no trivia.
- Bisect reduce la historia entre known good/bad.
- Revert crea inverse commit seguro para history compartida.
- Rollback se diseña antes de necesitarlo.

---

## 39. Checklist de dominio

- [ ] Puedo explicar límites de la evidencia de un test.
- [ ] Puedo probar behavior sin acoplarme a internals.
- [ ] Puedo usar AAA cuando mejora claridad.
- [ ] Puedo diseñar positive, negative, boundary e invariant tests.
- [ ] Puedo demostrar que SourceRecord, checksum, replay e índices derivados conservan sus contratos integrados.
- [ ] Puedo ejecutar pytest desde el interpreter correcto.
- [ ] Puedo usar `pytest.raises` con error específico.
- [ ] Puedo parametrizar una clase coherente.
- [ ] Puedo diseñar fixture explícita y teardown.
- [ ] Puedo usar `tmp_path` sin tocar user files.
- [ ] Puedo controlar clock/environment.
- [ ] Puedo reconocer monkeypatch excesivo.
- [ ] Puedo elegir real, fake, stub o mock.
- [ ] Puedo detectar overmocking.
- [ ] Puedo distinguir unit/integration/end-to-end.
- [ ] Puedo formular una property independiente.
- [ ] Puedo explicar shrinking.
- [ ] Puedo interpretar coverage sin perseguir cifra universal.
- [ ] Puedo reproducir/reducir una falla.
- [ ] Puedo separar symptom/root cause.
- [ ] Puedo leer traceback/cause chain.
- [ ] Puedo usar debugger para una hipótesis.
- [ ] Puedo crear regression test antes/junto al fix.
- [ ] Puedo aplicar failure injection acotada.
- [ ] Puedo probar timeout/cancellation cleanup.
- [ ] Puedo eliminar flakiness/order dependency.
- [ ] Puedo configurar logging básico.
- [ ] Puedo elegir levels por contract.
- [ ] Puedo usar `logger.exception` sin duplicación.
- [ ] Puedo diseñar structured/redacted fields.
- [ ] Puedo entregar README, CHANGELOG, ADR y threat notes P0 con evidencia de secret hygiene acotada.
- [ ] Puedo correlacionar una operation sin payload.
- [ ] Puedo dividir cambios en commits coherentes.
- [ ] Puedo revisar diff por risk/correctness.
- [ ] Puedo redactar comentario accionable.
- [ ] Puedo escribir PR autocontenida.
- [ ] Puedo decidir/escribir ADR.
- [ ] Puedo ejecutar `git bisect` con test confiable.
- [ ] Puedo usar `git revert` y explicar `reset` conservadoramente.
- [ ] Puedo diseñar rollback y verificarlo.
- [ ] Puedo defender EIDOLON 0.0a con evidencia reproducible.

---

## 40. Preparación para labs y builds

### PF-L14 — Pytest de fronteras

Evidencia: fixtures, parametrization, monkeypatch de clock/environment, `tmp_path` y property introductoria con boundaries Unicode/time.

### PF-L15 — Debugging con evidencia

Evidencia: corruption reproducible, minimal input, traceback, pdb/logs dirigidos, regression test y first bad commit con bisect.

### PF-L16 — Review y rollback

Evidencia: branch, commits coherentes, PR autocontenida, checklist por riesgo, ADR y revert sin history rewrite compartido.

### Consolidación

PF-M9 aporta defensa para PF-L01–PF-L13, PF-MP1 Journal CLI, PF-MP2 schema migrator y PF-MP3 Import Coordinator. La integración final es EIDOLON 0.0a reproducible, probada y defendible; no “perfecta”.

### Evidencia antes de la auditoría global

1. suite limpia en environment nuevo;
2. boundaries Unicode/time/files/async;
3. property independiente;
4. failure injection/recovery;
5. debugging record con hipótesis;
6. logs redactados;
7. commits y PR revisables;
8. ADR significativo;
9. bisect reproducible;
10. revert verificado.

Esto no aprueba todavía Programming Foundations. La auditoría acumulativa PF-M1–PF-M9 es una tarea separada.

---

## 41. Qué debes poder hacer después de PF-M1–PF-M9

Sin sustituir el futuro gate global, la secuencia pretende que puedas:

- razonar sobre nombres, objetos, identidad y mutabilidad;
- escribir funciones con contratos/effects explícitos;
- elegir colecciones y transformaciones legibles;
- organizar/installar un package reproducible;
- modelar Event/Claim/Correction con dataclasses/types/invariants;
- manejar errors, JSONL y local resource lifecycle;
- usar decorators/context managers solo cuando expresen policy/lifecycle;
- coordinar async work con ownership, cancellation y backpressure;
- probar boundaries e invariants;
- depurar mediante reproducción/hipótesis/evidencia;
- observar sin exponer payload;
- revisar, documentar y revertir cambios.

En conjunto, estas capacidades permiten construir y defender EIDOLON 0.0a: journal local determinista, source-preserving, instalable y observable con synthetic data.

El siguiente bloque curricular solo puede comenzar después de la auditoría/gate separado. Este capítulo no declara el track aprobado ni inicia Computer Science Foundations.

---

## 42. Recursos de ampliación

La explicación esencial está contenida aquí. Consulta [PF.11 Recursos recomendados](../../02_curriculum/01_programming_foundations.md#pf11-recursos-recomendados) y documentación oficial de pytest, Hypothesis, Python `pdb`/`logging` y Git para syntax/version details.

Prioriza primary documentation y reproduce examples en un repository sintético. No sustituyas test design por recipes ni review por checklists automáticos.

---

## 43. Límite del módulo

PF-M9 termina en test design, pytest, fixtures/tmp_path/monkeypatch, property-based testing introductorio, coverage crítico, debugging científico, tracebacks/debugger, logging redactado, async evidence, Git bisect/revert, code review, PR, ADR y rollback.

No desarrolla Computer Science Foundations formal, backend/API/database/frontend testing, web e2e, CI/CD avanzado, distributed tracing, production observability, security testing formal, mutation/load/fuzz testing avanzado, LLM/RAG evaluation, FastAPI, PostgreSQL, React, Docker obligatorio, Ollama, embeddings ni LLMs.

No audita/corrige PF-M1–PF-M8, no cambia estados anteriores, no genera review report global, no aprueba Programming Foundations y no comienza el siguiente track.
