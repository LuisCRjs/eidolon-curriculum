# PF-M2 — Funciones, contratos y scopes

**Track:** Programming Foundations  
**Competencias:** D1.1; refuerza D3.1 y D11.1  
**Fase:** P0  
**Nivel objetivo:** Profesional para el alcance del módulo  
**Prerequisites:** PF-M1  
**Build:** EIDOLON 0.0a  
**Curriculum source:** [PF-M2](../../02_curriculum/01_programming_foundations.md#pf-m2--funciones-contratos-y-scopes)  
**Status:** approved

Un programa pequeño puede sobrevivir con instrucciones escritas una detrás de otra. El problema aparece cuando una regla cambia, debe reutilizarse o necesita probarse sin ejecutar todo el programa. PF-M2 enseña a convertir comportamiento en funciones con fronteras visibles: qué reciben, qué devuelven, qué pueden rechazar y qué efectos producen.

Ya aprobaste PF-M1. Por tanto puedes razonar sobre nombres, objetos, asignación, referencias, mutabilidad, `None`, truthiness, texto, números y tiempo. Este capítulo utiliza `list` y `dict` de forma básica, pero su selección y complejidad pertenecen a PF-M3. No usaremos clases, type hints, archivos, JSON, decorators, async ni frameworks.

## Resultados de aprendizaje

Al terminar deberías poder:

- definir y llamar funciones pequeñas con parámetros, argumentos y retornos claros;
- distinguir imprimir de devolver y detectar un retorno `None` accidental;
- elegir entre argumentos posicionales, keyword y keyword-only según el contrato;
- usar defaults seguros y explicar cuándo se evalúan;
- emplear `*args` y `**kwargs` solo cuando el conjunto variable forma parte del contrato;
- describir una función mediante entradas, precondiciones, salida, errores, efectos e invariantes;
- predecir la resolución de nombres con LEGB;
- distinguir scope, binding y lifetime;
- detectar shadowing, `UnboundLocalError` y estado global oculto;
- explicar y usar `global` y `nonlocal`, además de justificar cuándo evitarlos;
- reproducir y corregir un bug de mutable default argument;
- separar lógica pura de console, filesystem, clock, environment, random y estado global;
- hacer explícitas dependencias básicas mediante datos, configuración, clocks y comportamientos recibidos;
- construir y explicar una closure sencilla sin usarla para esconder estado complejo;
- refactorizar reglas sintéticas de EIDOLON en funciones deterministas y fáciles de probar.

## Cómo estudiar este módulo

Para cada ejemplo:

1. identifica los nombres que entran a la función;
2. predice el retorno y cualquier efecto observable;
3. ejecuta el bloque;
4. cambia un argumento sin cambiar la función;
5. explica qué parte del contrato aceptó o rechazó el caso;
6. pregunta si podrías probar la regla sin console, clock o estado global.

### Convenciones del código

- **Ejemplo ejecutable:** bloque autónomo que debe correr sin preparación adicional.
- **Continuación:** reutiliza definiciones del bloque inmediatamente anterior.
- **Código incorrecto:** antipatrón ejecutable cuyo resultado debe observarse y explicarse.
- **Failure case:** provoca deliberadamente un error indicado por el texto.
- **Fragmento de sintaxis:** ilustra una firma o forma válida, pero no constituye un programa completo.
- **Solución parcial:** resuelve el concepto local, no una arquitectura completa.

Los outputs mostrados corresponden a Python 3.14. Cuando el resultado estable es una propiedad y no una representación concreta, se comprueba con `assert`.

### Sintaxis de apoyo

Usaremos unas pocas construcciones sin desarrollar todavía sus módulos completos:

- `raise ValueError(...)` hace visible una precondición incumplida; la taxonomía y recuperación de excepciones llegan en PF-M6;
- `try`/`except` permite observar failure cases sin detener el ejemplo; también pertenece a PF-M6;
- `assert` expresa una comprobación ejecutable pequeña; pytest y estrategia de testing llegan en PF-M9;
- `dict` y `list` sirven como datos sintéticos; sus tradeoffs se estudian en PF-M3;
- imports puntuales de la biblioteca estándar permiten observar clock y environment; módulos y packages llegan en PF-M4.

No necesitas dominar esas construcciones para seguir PF-M2.

---

## 1. Por qué existen las funciones

### 1.1 El problema de una regla dispersa

Supón que EIDOLON acepta el cambio de estado `draft → accepted`. Si la regla se escribe directamente en tres lugares, una modificación futura puede actualizar dos y olvidar el tercero.

**Código incorrecto — regla duplicada:**

```python
current_status = "draft"
target_status = "accepted"

if current_status == "draft" and target_status == "accepted":
    print("Transición válida")

# En otra parte del programa aparece otra copia de la misma regla.
if current_status == "draft" and target_status == "accepted":
    print("Puede persistirse")
```

El problema no es solo repetir caracteres. Ahora existen dos lugares que afirman saber qué transición es válida. Si divergen, el programa contiene dos versiones del dominio.

### 1.2 Modelo mental: una función es una frontera de comportamiento

Una función:

- recibe información mediante argumentos;
- ejecuta una regla o transformación;
- devuelve un resultado;
- puede producir efectos observables;
- al terminar, sus nombres locales dejan de ser accesibles por una llamada normal; objetos devueltos o bindings capturados por una closure pueden seguir vivos.

Puedes pensarla como una pregunta nombrada:

```text
entradas → regla nombrada → resultado
```

El nombre debe comunicar la decisión. `is_valid_transition` expresa mejor el contrato que `process_data`.

### 1.3 Definición y llamada

**Ejemplo ejecutable:**

```python
def is_valid_transition(current_status, target_status):
    return current_status == "draft" and target_status == "accepted"


result = is_valid_transition("draft", "accepted")

print(result)
# True
```

La sentencia `def` crea un objeto función y enlaza el nombre `is_valid_transition` con él. El cuerpo no se ejecuta en ese momento. Este **Fragmento de llamada** muestra la expresión:

```python
is_valid_transition("draft", "accepted")
```

realiza una llamada. Python enlaza los argumentos con los parámetros, ejecuta el cuerpo y entrega el valor indicado por `return`.

En el ejemplo:

- `current_status` y `target_status` son **parámetros** de la definición;
- `"draft"` y `"accepted"` son **argumentos** de esa llamada;
- `True` es el resultado devuelto.

### 1.4 Definir no es llamar

**Ejemplo ejecutable — predice el orden:**

```python
print("antes de definir")


def announce():
    print("dentro de la función")


print("después de definir")
announce()
print("después de llamar")
```

Output:

```text
antes de definir
después de definir
dentro de la función
después de llamar
```

`def` crea la función; `announce()` ejecuta su cuerpo.

### 1.5 Qué ocurre durante una llamada

A nivel práctico, cada llamada crea un contexto local nuevo para sus parámetros y nombres locales. Dos llamadas consecutivas no comparten automáticamente esos bindings.

**Ejemplo ejecutable:**

```python
def describe_event(event_id):
    label = "event:" + event_id
    return label


first = describe_event("evt-001")
second = describe_event("evt-002")

print(first)   # event:evt-001
print(second)  # event:evt-002
```

Cada llamada tuvo su propio `event_id` y `label`. Los strings devueltos pueden sobrevivir porque `first` y `second` quedaron enlazados con ellos; los nombres locales de la llamada dejan de estar disponibles al terminar.

No necesitas estudiar frames, stack internals ni bytecode en PF-M2. Necesitas predecir qué bindings pertenecen a cada llamada.

### 1.6 Cuándo no crear una función

No conviertas cada línea en una función. Extraer comportamiento ayuda cuando existe al menos una razón concreta:

- la regla necesita un nombre;
- se usa en más de un lugar;
- debe probarse de manera aislada;
- mezcla una decisión con un efecto que conviene separar;
- el bloque tiene más de una razón para cambiar;
- reducirlo permite explicar mejor su contrato.

Una operación evidente usada una sola vez puede permanecer en el flujo principal. El objetivo es claridad y control del cambio, no maximizar el número de funciones.

> **Antes de continuar:** modifica `is_valid_transition` para aceptar también `accepted → archived`. Predice tres llamadas: una válida por cada regla y una inválida. No uses estado global.

---

## 2. `return`, resultados y caminos de salida

### 2.1 Imprimir no es devolver

`print` escribe texto en la console: es un efecto externo. `return` entrega un objeto a quien llamó la función.

**Código incorrecto — confundir output visual con resultado:**

```python
def build_event_label(event_id):
    print("event:" + event_id)


label = build_event_label("evt-001")

print(label)
```

Output:

```text
event:evt-001
None
```

La función imprimió una etiqueta, pero no la devolvió. Al terminar sin `return`, Python devuelve `None` implícitamente.

### 2.2 Devolver permite componer

**Ejemplo ejecutable:**

```python
def build_event_label(event_id):
    return "event:" + event_id


label = build_event_label("evt-001")
message = "Procesando " + label

print(message)
# Procesando event:evt-001
```

Ahora quien llama decide si imprime, almacena o compara el valor. La función no impone una salida a console.

### 2.3 `return` detiene la llamada actual

**Ejemplo ejecutable — predice qué líneas aparecen:**

```python
def classify_confidence(value):
    if value is None:
        return "missing"

    if value == 0.0:
        return "zero"

    return "present"


print(classify_confidence(None))  # missing
print(classify_confidence(0.0))   # zero
print(classify_confidence(0.8))   # present
```

Solo se ejecuta el primer `return` alcanzado en cada llamada. Tener varios caminos de retorno no es incorrecto por sí mismo: puede hacer visibles decisiones distintas. El riesgo aparece cuando algún camino viola el contrato.

### 2.4 Retorno implícito accidental

**Código incorrecto:**

```python
def classify_status(status):
    if status == "accepted":
        return "final"


print(classify_status("accepted"))  # final
print(classify_status("draft"))     # None
```

Si el contrato prometía siempre un `str`, el segundo resultado es un bug. Las alternativas dependen del dominio:

- devolver una categoría válida para `draft`;
- rechazar el estado mediante un error;
- declarar que `None` significa “sin clasificación”.

No añadas un `return` arbitrario solo para eliminar `None`; primero decide la semántica.

### 2.5 `return` sin expresión

**Ejemplo ejecutable:**

```python
def stop_if_empty(text):
    if text == "":
        return

    return text.casefold()


print(stop_if_empty(""))        # None
print(stop_if_empty("Memoria")) # memoria
```

`return` sin expresión también devuelve `None`. Puede ser correcto si el contrato lo declara, pero no debe ser accidental.

### 2.6 Responsabilidad única sin dogma

“Una función hace una sola cosa” no significa “una función contiene una sola línea”. Significa que su comportamiento puede describirse como una responsabilidad coherente y cambia por una razón principal.

Una función `normalize_tag` puede eliminar espacios, normalizar Unicode y aplicar `casefold()` si las tres operaciones forman una política de normalización. Separarlas en tres funciones públicas no aporta claridad por sí solo.

En cambio, una función que normaliza, consulta el clock, imprime y modifica una lista global tiene varias razones independientes para cambiar.

> **Antes de continuar:** toma el código incorrecto de `build_event_label`. Escribe una versión que devuelva el dato y deja `print` fuera. Después reemplaza `print` por un `assert` sin modificar la función.

---

## 3. Parámetros y argumentos: hacer visible el contrato de llamada

### 3.1 El problema

Este **Fragmento de llamada** no comunica qué representa cada valor:

```python
create_receipt("evt-001", "accepted", True, 30)
```

¿`True` significa `verified`, `private` o `dry_run`? ¿`30` son segundos, intentos o días? La firma y la forma de llamada deben reducir ambigüedad.

### 3.2 Positional arguments

Los argumentos posicionales se enlazan por orden.

**Ejemplo ejecutable:**

```python
def build_transition(current_status, target_status):
    return current_status + " -> " + target_status


print(build_transition("draft", "accepted"))
# draft -> accepted
```

El primer argumento se enlaza con `current_status`; el segundo, con `target_status`.

Son adecuados cuando:

- hay pocos argumentos;
- el orden es natural y estable;
- los valores se distinguen con facilidad.

Dos strings del mismo tipo pueden intercambiarse sin que Python detecte el error:

**Código incorrecto — orden semántico invertido:**

```python
def build_transition(current_status, target_status):
    return current_status + " -> " + target_status


print(build_transition("accepted", "draft"))
# accepted -> draft
```

Python cumplió el binding posicional. El contrato del dominio era responsabilidad del código.

### 3.3 Keyword arguments

Los argumentos keyword se enlazan por nombre.

**Ejemplo ejecutable:**

```python
def build_transition(current_status, target_status):
    return current_status + " -> " + target_status


result = build_transition(
    target_status="accepted",
    current_status="draft",
)

print(result)
# draft -> accepted
```

El orden escrito dejó de importar porque los nombres hacen explícita la intención.

No uses keywords para volver ceremoniosa una llamada evidente. Úsalos cuando evitan confusión, documentan unidades u ofrecen opciones.

### 3.4 Default arguments

Un default declara qué valor se usa cuando el caller omite ese argumento.

**Ejemplo ejecutable:**

```python
def build_label(event_id, prefix="event"):
    return prefix + ":" + event_id


print(build_label("evt-001"))
print(build_label("evt-001", prefix="memory"))
```

Output:

```text
event:evt-001
memory:evt-001
```

El default forma parte del contrato público. Cambiarlo puede cambiar el comportamiento de todos los callers que lo omiten.

Los parámetros sin default deben aparecer antes que los parámetros con default dentro del mismo grupo posicional. Esta firma es inválida:

**Failure case — `SyntaxError` al compilar:**

```python
# No ejecutes este fragmento dentro de otro archivo.
# def build_label(prefix="event", event_id):
#     return prefix + ":" + event_id
```

El código está comentado para que el capítulo completo pueda compilarse; si eliminas los comentarios, Python rechaza la definición porque un parámetro requerido sigue a uno con default.

### 3.5 Keyword-only arguments

Algunos valores deben nombrarse porque su posición no comunica suficiente información. Un `*` en la firma hace keyword-only los parámetros posteriores.

**Ejemplo ejecutable:**

```python
def filter_events(events, *, status=None, limit=None):
    selected = []

    for event in events:
        if status is not None and event["status"] != status:
            continue

        selected.append(event)

        if limit is not None and len(selected) == limit:
            break

    return selected


events = [
    {"id": "evt-001", "status": "draft"},
    {"id": "evt-002", "status": "accepted"},
]

result = filter_events(events, status="accepted", limit=1)
print(result)
# [{'id': 'evt-002', 'status': 'accepted'}]
```

`status` y `limit` no pueden pasarse accidentalmente como tercer y cuarto valor sin nombre.

**Failure case — argumento keyword-only pasado por posición:**

```python
def filter_events(events, *, status=None):
    return events


try:
    filter_events([], "accepted")
except TypeError as error:
    print(type(error).__name__)
# TypeError
```

El diseño es útil cuando los argumentos son opciones, flags, unidades o valores del mismo tipo que sería fácil intercambiar.

### 3.6 Positional-only arguments

Una `/` marca como positional-only los parámetros anteriores. Es pertinente cuando el nombre del parámetro no debe formar parte del contrato de llamada.

**Ejemplo ejecutable:**

```python
def clamp(value, /, *, minimum=0, maximum=100):
    if value < minimum:
        return minimum
    if value > maximum:
        return maximum
    return value


print(clamp(120, maximum=90))  # 90
```

`value` se pasa por posición; `minimum` y `maximum`, por keyword. La firma expresa dos decisiones distintas.

**Failure case — positional-only pasado por keyword:**

```python
def clamp(value, /, *, minimum=0, maximum=100):
    return value


try:
    clamp(value=20)
except TypeError as error:
    print(type(error).__name__)
# TypeError
```

No conviertas todos los parámetros en positional-only. En código de aplicación, los nombres keyword suelen mejorar legibilidad y estabilidad. `/` es más común en APIs de bajo nivel o cuando el nombre interno debe poder cambiar sin romper callers.

### 3.7 `*args`: cantidad variable de argumentos posicionales

`*args` agrupa los argumentos posicionales adicionales en una tupla.

**Ejemplo ejecutable:**

```python
def all_statuses_match(expected_status, *statuses):
    for status in statuses:
        if status != expected_status:
            return False
    return True


print(all_statuses_match("accepted", "accepted", "accepted"))  # True
print(all_statuses_match("accepted", "accepted", "draft"))     # False
```

Aquí la cantidad variable forma parte de la pregunta: “¿todos estos estados coinciden?”.

Con cero valores adicionales la función devuelve `True`: no encontró un estado que contradijera la condición. Si el dominio exige al menos uno, esa precondición debe validarse de forma explícita.

`*args` oculta el contrato cuando en realidad existen tres roles diferentes. Este es un **Fragmento de llamada** con ese problema:

```python
create_event("evt-1", "texto", "draft", "America/Merida")
```

Si cada posición tiene un significado único, parámetros nombrados suelen ser mejores.

### 3.8 `**kwargs`: cantidad variable de argumentos keyword

`**kwargs` agrupa keywords adicionales en un diccionario.

**Ejemplo ejecutable:**

```python
def build_metadata(**fields):
    return fields


metadata = build_metadata(source="journal", language="es")

print(metadata)
# {'source': 'journal', 'language': 'es'}
```

Es válido si el contrato realmente acepta campos abiertos. Es peligroso cuando se usa para evitar diseñar una firma:

**Código incorrecto — contrato oculto:**

```python
def create_event(**options):
    event_id = options["id"]
    status = options["status"]
    return {"id": event_id, "status": status}


event = create_event(id="evt-001", status="draft")
print(event)
```

La función requiere `id` y `status`, pero su firma no lo declara. Un typo se descubre tarde mediante acceso al diccionario. Si los campos son conocidos, escríbelos como parámetros.

### 3.9 Orden mental de una firma

No necesitas memorizar toda la gramática, pero sí leer esta secuencia:

**Fragmento de sintaxis:**

```python
def example(
    positional_only, /,
    positional_or_keyword,
    *extra_positional,
    keyword_only,
    **extra_keyword,
):
    ...
```

No toda función necesita todas las categorías. Una firma pequeña es preferible cuando expresa el contrato completo.

> **Antes de continuar:** cambia `filter_events` para que `limit=0` devuelva una lista vacía y no se confunda con ausencia. Prueba `limit=None`, `limit=0` y `limit=1`. Después explica por qué `limit` debe ser keyword-only.

---

## 4. Contratos de funciones

### 4.1 Por qué una firma no basta

Este **Fragmento de sintaxis** revela nombres, pero todavía deja preguntas:

```python
def normalize_tag(tag):
    ...
```

- ¿`tag` debe ser `str`?
- ¿acepta texto vacío?
- ¿modifica algún objeto externo?
- ¿devuelve el original o una representación derivada?
- ¿qué ocurre con input inválido?

El contrato es el conjunto de acuerdos observables entre la función y quien la llama.

### 4.2 Las partes del contrato

Para PF-M2 describe al menos:

| Parte | Pregunta |
|---|---|
| Entradas | ¿Qué objetos recibe y qué representa cada uno? |
| Tipo esperado | ¿Qué operaciones debe soportar el objeto? |
| Dato válido | ¿Qué valores del tipo pertenecen al dominio? |
| Precondición | ¿Qué debe ser cierto antes de llamar? |
| Resultado | ¿Qué devuelve y qué significa? |
| Error | ¿Cómo comunica que no puede cumplir? |
| Efectos | ¿Qué cambia u observa fuera de sus valores locales? |
| Invariantes | ¿Qué propiedad debe conservarse? |

La profundidad matemática es **Level 0 — intuition**. Usamos precondiciones y postcondiciones como acuerdos razonables, no formal methods.

### 4.3 Tipo esperado no equivale a dato válido

`""` es un `str`, pero puede ser un ID inválido. `-3` es un `int`, pero puede ser un límite inválido.

**Ejemplo ejecutable:**

```python
def build_event_key(event_id):
    if not isinstance(event_id, str):
        raise TypeError("event_id debe ser str")
    if event_id == "":
        raise ValueError("event_id no puede estar vacío")

    return "event:" + event_id


print(build_event_key("evt-001"))
# event:evt-001
```

- `str` es el tipo esperado;
- texto no vacío es una regla de validez;
- “el caller proporciona un ID no vacío” es una precondición;
- el string prefijado es el resultado;
- `TypeError` o `ValueError` hacen visible el incumplimiento.

`isinstance(object, type)` devuelve `True` cuando el objeto es una instancia del tipo indicado —o de un subtipo—. También acepta una tupla de tipos. Comprueba una relación de tipos en runtime; no demuestra que el valor satisfaga las reglas del dominio.

La selección completa de excepciones y la recuperación se estudian en PF-M6. Aquí solo evitamos un estado imposible silencioso.

### 4.4 Failure cases del contrato

**Ejemplo ejecutable:**

```python
def build_event_key(event_id):
    if not isinstance(event_id, str):
        raise TypeError("event_id debe ser str")
    if event_id == "":
        raise ValueError("event_id no puede estar vacío")
    return "event:" + event_id


for invalid_value in (42, ""):
    try:
        build_event_key(invalid_value)
    except (TypeError, ValueError) as error:
        print(type(error).__name__)
```

Output:

```text
TypeError
ValueError
```

El `try`/`except` pertenece a la observación del ejemplo, no a `build_event_key`. La función no atrapa su propio error para imprimir y continuar con un resultado inventado.

### 4.5 Precondición, excepción y resultado son cosas distintas

Considera `take_first(items, count)`:

- **precondición:** `count >= 0`;
- **excepción:** `ValueError` si `count < 0`;
- **resultado:** una lista nueva con hasta `count` elementos;
- **invariante:** `items` no se modifica;
- **efectos:** ninguno fuera del resultado.

La precondición describe lo que debe ser cierto. La excepción es un mecanismo posible para comunicar que no lo fue. El resultado describe el caso exitoso.

### 4.6 Docstring como contrato cercano

Una docstring útil registra comportamiento que la firma no expresa. No necesita repetir cada línea.

**Ejemplo ejecutable:**

```python
def take_first(items, count):
    """Devuelve una lista nueva con hasta `count` elementos.

    Requiere `count >= 0` y no modifica `items`.
    Lanza ValueError cuando `count` es negativo.
    """
    if count < 0:
        raise ValueError("count no puede ser negativo")

    result = []
    index = 0

    while index < count and index < len(items):
        result.append(items[index])
        index += 1

    return result


source = ["evt-1", "evt-2", "evt-3"]
selected = take_first(source, 2)

assert selected == ["evt-1", "evt-2"]
assert source == ["evt-1", "evt-2", "evt-3"]
```

El algoritmo de colección no es el objetivo; lo importante es que contrato, implementación y asserts cuentan la misma historia.

### 4.7 Invariantes en EIDOLON

Una función que aplica una `Correction` no debe mutar el evento fuente. Esa propiedad puede expresarse como contrato y test.

**Ejemplo ejecutable:**

```python
def apply_correction(event, correction):
    """Devuelve una vista corregida sin modificar el evento fuente."""
    if correction["target_event_id"] != event["id"]:
        raise ValueError("la corrección apunta a otro evento")

    corrected = event.copy()
    corrected["text"] = correction["replacement_text"]
    corrected["correction_id"] = correction["id"]
    return corrected


event = {"id": "evt-1", "text": "lunes"}
correction = {
    "id": "cor-1",
    "target_event_id": "evt-1",
    "replacement_text": "martes",
}

view = apply_correction(event, correction)

assert event == {"id": "evt-1", "text": "lunes"}
assert view["text"] == "martes"
assert view is not event
```

La shallow copy es suficiente para los campos planos de este ejemplo. PF-M1 ya enseñó que no sería suficiente si mutáramos estructuras anidadas compartidas.

### 4.8 Cuándo no llenar todo de validaciones

No toda función interna necesita repetir las mismas comprobaciones. Valida en una frontera que posea el contrato y conserva invariantes dentro del núcleo. Duplicar defensas puede oscurecer el flujo y producir mensajes contradictorios.

En PF-M2 no diseñamos todavía schemas externos ni una taxonomía completa de errores. Aprendemos a declarar qué acepta la función y a no fabricar resultados válidos ante datos imposibles.

> **Antes de continuar:** escribe el contrato de `is_valid_transition` sin código: entradas, valores válidos, resultado, errores, efectos e invariante. Luego implementa la versión mínima y crea un assert positivo y otro negativo.

---

## 5. Composición y tamaño de funciones

### 5.1 Del bloque monolítico a decisiones nombradas

Una función grande suele mezclar pasos que cambian por razones distintas.

**Código incorrecto — transformación, clock y console mezclados:**

```python
from datetime import UTC, datetime


def ingest_event(raw_event):
    normalized_text = raw_event["text"].strip().casefold()
    recorded_at = datetime.now(UTC).isoformat()
    event = {
        "id": raw_event["id"],
        "source_text": raw_event["text"],
        "search_text": normalized_text,
        "recorded_at": recorded_at,
    }
    print("Evento creado:", event["id"])
    return event


ingest_event({"id": "evt-1", "text": "  Mérida  "})
```

El código puede funcionar, pero probar la transformación obliga a aceptar un clock real y output de console.

### 5.2 Extraer por responsabilidad

**Ejemplo ejecutable:**

```python
from datetime import UTC, datetime


def build_search_text(source_text):
    return source_text.strip().casefold()


def build_event(raw_event, recorded_at):
    return {
        "id": raw_event["id"],
        "source_text": raw_event["text"],
        "search_text": build_search_text(raw_event["text"]),
        "recorded_at": recorded_at,
    }


def now_utc_text():
    return datetime.now(UTC).isoformat()


raw_event = {"id": "evt-1", "text": "  Mérida  "}
event = build_event(raw_event, recorded_at=now_utc_text())
print("Evento creado:", event["id"])
```

Ahora:

- `build_search_text` transforma texto;
- `build_event` construye datos a partir de entradas explícitas;
- `now_utc_text` consulta el clock;
- el flujo principal decide imprimir.

`strip()` crea un string sin whitespace al inicio y al final; no elimina espacios internos. PF-M1 ya enseñó que los strings son inmutables, así que la fuente no cambia por llamar este método.

Esto no es todavía una arquitectura de ports/adapters. Es una separación local suficiente para probar el dominio sin activar efectos irrelevantes.

### 5.3 Composición

Componer funciones significa usar el resultado de una como entrada de otra.

**Ejemplo ejecutable:**

```python
def clean_tag(tag):
    return tag.strip().casefold()


def build_tag_key(tag):
    cleaned = clean_tag(tag)
    return "tag:" + cleaned


print(build_tag_key("  Importante "))
# tag:importante
```

La composición es útil si cada paso tiene significado propio. Crear `remove_left_space`, `remove_right_space` y `lowercase_tag` solo para componer tres líneas no mejora necesariamente el diseño.

### 5.4 Predicados

Una función que responde una pregunta booleana es un predicado. Un nombre como `is_`, `has_` o `can_` comunica que el resultado esperado es booleano.

**Ejemplo ejecutable:**

```python
def has_source_text(event):
    return event["source_text"] != ""


print(has_source_text({"source_text": "Recuerdo"}))  # True
print(has_source_text({"source_text": ""}))          # False
```

Evita predicados que imprimen, cambian el evento o consultan estado global. Una pregunta reutilizable debe poder responderse sin efectos sorpresa.

### 5.5 Cuándo no fragmentar

Mantén juntas operaciones que forman una sola política y se comprenden mejor en secuencia. Extrae cuando el nombre revela una regla, reduce acoplamiento o permite probar una decisión.

La longitud es una señal, no un criterio absoluto. Una función de quince líneas con un contrato coherente puede ser mejor que cinco funciones de tres líneas que obligan a saltar entre nombres vagos.

> **Antes de continuar:** toma `ingest_event` y escribe tres asserts que solo deberían probar la transformación. Comprueba por qué la primera versión dificulta fijar `recorded_at`, y por qué recibirlo como dato vuelve reproducible el resultado.

---

## 6. Scope y resolución de nombres: LEGB

### 6.1 Por qué importa

Cuando una función usa `status`, Python debe decidir a qué binding se refiere. Si el programador y el runtime eligen bindings distintos, aparecen bugs que parecen cambios “mágicos” de estado.

El alcance (scope) es la región donde un binding puede encontrarse directamente por su nombre. La resolución sigue, de forma práctica, LEGB:

1. **Local:** el bloque de la función actual;
2. **Enclosing:** funciones exteriores que contienen a la actual;
3. **Global:** el namespace del módulo;
4. **Builtins:** nombres incorporados como `len`, `print` o `str`.

LEGB describe búsqueda de nombres. No describe búsquedas de atributos como `event.copy`; esas siguen otras reglas y quedan fuera de esta sección.

### 6.2 Local scope

**Ejemplo ejecutable:**

```python
status = "global"


def show_status():
    status = "local"
    return status


print(show_status())  # local
print(status)         # global
```

La asignación dentro de `show_status` crea un binding local. No cambia el binding global.

Los parámetros también son nombres locales:

**Ejemplo ejecutable:**

```python
status = "global"


def echo_status(status):
    return status


print(echo_status("argument"))  # argument
print(status)                    # global
```

### 6.3 Enclosing scope

Una función interna puede leer nombres de la función exterior.

**Ejemplo ejecutable:**

```python
label = "global"


def outer():
    label = "enclosing"

    def inner():
        return label

    return inner()


print(outer())  # enclosing
```

`inner` no encuentra `label` local, así que continúa hacia el scope enclosing de `outer`. No llega al global.

### 6.4 Global scope

Una función puede leer un nombre global sin declarar `global`.

**Ejemplo ejecutable:**

```python
DEFAULT_STATUS = "draft"


def default_status_message():
    return "Estado: " + DEFAULT_STATUS


print(default_status_message())
# Estado: draft
```

La lectura funciona, pero sigue siendo una dependencia implícita: el resultado no depende solo de los argumentos visibles. Una constante estable puede ser aceptable; configuración cambiante y estado de negocio suelen ser mejores como entradas explícitas.

### 6.5 Builtins

Si un nombre no aparece en L, E ni G, Python busca en builtins.

**Ejemplo ejecutable:**

```python
def count_events(events):
    return len(events)


print(count_events(["evt-1", "evt-2"]))  # 2
```

`len` se resuelve en builtins.

### 6.6 Predicción completa de LEGB

**Ejemplo ejecutable — predice antes de correr:**

```python
name = "global"


def outer():
    name = "enclosing"

    def inner(argument):
        name = "local"
        return name + " / " + argument + " / " + str(len(argument))

    return inner("value")


print(outer())
# local / value / 5
```

- `name` se encuentra en Local;
- `argument` es un parámetro local;
- `str` y `len` llegan a Builtins;
- el `name` enclosing y el global quedan ocultos por shadowing.

### 6.7 Shadowing

Shadowing ocurre cuando un binding más cercano oculta otro con el mismo nombre.

**Código incorrecto — ocultar un builtin:**

```python
def count_events(events):
    len = 100
    return len(events)


try:
    count_events(["evt-1"])
except TypeError as error:
    print(type(error).__name__)
# TypeError
```

El nombre local `len` contiene un entero. LEGB lo encuentra antes que al builtin `len`, y `100(...)` no es una llamada válida.

Shadowing no siempre es un error. Un parámetro local puede usar un nombre significativo aunque exista otro global. Es peligroso cuando oculta una dependencia o un builtin y vuelve ambigua la lectura.

### 6.8 Scope no es lifetime

Scope responde “¿dónde puede resolverse este nombre?”. Lifetime responde “¿durante cuánto tiempo existe el binding u objeto durante la ejecución?”.

**Ejemplo ejecutable:**

```python
def build_payload():
    local_payload = {"id": "evt-1"}
    return local_payload


payload = build_payload()
print(payload)  # {'id': 'evt-1'}
```

El nombre local `local_payload` ya no puede usarse fuera de la función, pero el objeto diccionario continúa vivo porque `payload` lo referencia. PF-M1 ya enseñó que nombres y objetos son cosas distintas.

### 6.9 Los bloques de control no crean function scope

En Python, un `if`, `for` o `while` dentro de una función no crea otro local scope de función.

**Ejemplo ejecutable:**

```python
def choose_label(accepted):
    if accepted:
        label = "accepted"
    else:
        label = "draft"

    return label


print(choose_label(True))   # accepted
print(choose_label(False))  # draft
```

`label` pertenece al local scope de `choose_label`, no a un scope separado del `if`.

Las comprehensions tienen una regla de scope propia en Python moderno, pero se estudian en PF-M3.

> **Antes de continuar:** para el ejemplo de `outer`/`inner`, cambia primero el `name` local, luego elimínalo y finalmente elimina el enclosing. Predice en cada versión qué binding gana y anota la letra de LEGB que lo explica.

---

## 7. Asignación local, `global` y `nonlocal`

### 7.1 La regla estática que produce `UnboundLocalError`

Si un nombre recibe una asignación en una función, Python lo trata como local en ese bloque, salvo una declaración `global` o `nonlocal`. La decisión se toma al analizar el bloque completo, no línea por línea durante la ejecución.

**Failure case intencional:**

```python
status = "global"


def broken_status():
    print(status)
    status = "local"


try:
    broken_status()
except UnboundLocalError as error:
    print(type(error).__name__)
# UnboundLocalError
```

Aunque la asignación aparece después de `print`, hace que `status` sea local para toda la función. Cuando `print(status)` intenta leerlo, ese binding local todavía no tiene valor.

Mover líneas al azar puede ocultar el síntoma, pero la pregunta correcta es: ¿la función necesita leer un dato, producir uno nuevo o modificar estado externo?

### 7.2 Solución habitual: recibir y devolver

**Ejemplo ejecutable:**

```python
status = "global"


def advance_status(current_status):
    print(current_status)
    return "accepted"


new_status = advance_status(status)

print(status)      # global
print(new_status)  # accepted
```

El estado entra como argumento y el nuevo estado sale como retorno. No existe una mutación global oculta.

### 7.3 `global`

`global name` indica que los bindings de ese nombre dentro del bloque pertenecen al namespace global del módulo.

**Ejemplo ejecutable:**

```python
processed_count = 0


def mark_processed():
    global processed_count
    processed_count += 1


mark_processed()
mark_processed()

print(processed_count)  # 2
```

La declaración no crea una variable “universal” compartida por todo sistema; selecciona el binding global de ese módulo para las asignaciones de esa función.

Este código funciona, pero su contrato visible es pobre:

- la firma no muestra que cambia `processed_count`;
- el orden de llamadas afecta resultados futuros;
- un test debe restaurar el global;
- dos operaciones no pueden mantener contadores independientes.

### 7.4 Evitar `global` con datos explícitos

**Ejemplo ejecutable:**

```python
def increment_count(current_count):
    return current_count + 1


processed_count = 0
processed_count = increment_count(processed_count)
processed_count = increment_count(processed_count)

print(processed_count)  # 2
```

La función es reutilizable para cualquier contador y cada llamada puede probarse con un input conocido.

`global` no está prohibido. Puede ser razonable en scripts muy pequeños o al enlazar APIs que exigen cierto estado de módulo. Debe tratarse como un efecto explícitamente documentado, no como atajo normal para devolver varios datos.

### 7.5 `nonlocal`

`nonlocal name` permite reasignar el binding correspondiente en el scope enclosing más cercano de una función exterior.

**Ejemplo ejecutable:**

```python
def make_counter():
    count = 0

    def next_count():
        nonlocal count
        count += 1
        return count

    return next_count


counter = make_counter()

print(counter())  # 1
print(counter())  # 2
```

`count` no es local de `next_count` ni global: pertenece a la llamada de `make_counter` que creó esa función interna. La closure mantiene accesible ese binding después de que `make_counter` terminó.

`nonlocal` solo puede referirse a un binding existente en una función enclosing. No crea uno nuevo y no selecciona el global.

### 7.6 Failure case de `nonlocal`

Una declaración sin binding enclosing es un error de sintaxis.

**Fragmento deliberadamente inválido, mantenido como texto:**

```text
def broken():
    nonlocal missing_name
```

Python produce `SyntaxError: no binding for nonlocal 'missing_name' found` al compilar. No se incluye como bloque Python porque no forma un programa sintácticamente válido.

### 7.7 Elegir entre retorno, `nonlocal` y objeto futuro

Usa un retorno cuando el nuevo estado debe ser visible para el caller. Una closure con `nonlocal` puede ser útil cuando:

- el estado es pequeño;
- pertenece claramente a una sola función creada;
- existe una API mínima para observarlo;
- los tests pueden construir una instancia nueva;
- ocultarlo no daña provenance ni replay.

Si el estado tiene varias operaciones, invariantes complejas o identidad de dominio, probablemente merezca un objeto explícito. Ese modelado se estudia en PF-M5; no lo adelantaremos con clases improvisadas.

> **Antes de continuar:** reproduce `UnboundLocalError` y explica por qué ocurre antes de cambiar el código. Después implementa la solución recibir/devolver y demuestra que dos contadores independientes no comparten estado.

---

## 8. Mutable default arguments

### 8.1 El bug

Queremos que cada llamada sin `tags` comience con una lista vacía.

**Código incorrecto — predice ambos resultados:**

```python
def add_tag(tag, tags=[]):
    tags.append(tag)
    return tags


first = add_tag("personal")
second = add_tag("urgent")

print(first)
print(second)
print(first is second)
```

Output:

```text
['personal', 'urgent']
['personal', 'urgent']
True
```

El segundo llamado no recibió una lista nueva. Ambos usaron el mismo objeto default.

### 8.2 Cuándo se evalúan los defaults

Las expresiones default se evalúan cuando se ejecuta la definición de la función, no cada vez que se llama. El objeto resultante se conserva en el objeto función y se reutiliza cuando el caller omite ese argumento.

Modelo práctico:

```text
se ejecuta def
    ↓
se crea una lista default
    ↓
llamada 1 sin tags ─┐
llamada 2 sin tags ─┴─→ misma lista
```

PF-M1 ya proporciona la causa: los parámetros son nombres enlazados con objetos; asignar no copia y una lista es mutable.

### 8.3 El patrón con `None`

**Ejemplo ejecutable:**

```python
def add_tag(tag, tags=None):
    if tags is None:
        tags = []

    tags.append(tag)
    return tags


first = add_tag("personal")
second = add_tag("urgent")

print(first)             # ['personal']
print(second)            # ['urgent']
print(first is second)   # False
```

`None` es inmutable y representa “el caller no proporcionó una colección”. La lista se crea dentro de cada llamada que la necesita.

No uses `tags = tags or []` si una lista vacía proporcionada por el caller debe conservar su identidad. Truthiness mezclaría ausencia con colección vacía.

### 8.4 El caller sí puede compartir deliberadamente

**Ejemplo ejecutable:**

```python
def add_tag(tag, tags=None):
    if tags is None:
        tags = []

    tags.append(tag)
    return tags


shared_tags = []

first = add_tag("personal", shared_tags)
second = add_tag("urgent", shared_tags)

print(shared_tags)       # ['personal', 'urgent']
print(first is second)   # True
```

Aquí compartir no es un default accidental: el caller pasó explícitamente el mismo objeto mutable. Todavía debes decidir si la función tiene permiso para mutarlo.

### 8.5 Mejor contrato: no mutar la entrada

Si el nombre `add_tag` debe producir una lista derivada sin cambiar la fuente, crea un contenedor exterior nuevo.

**Ejemplo ejecutable:**

```python
def with_tag(tags, tag):
    result = tags.copy()
    result.append(tag)
    return result


source_tags = ["personal"]
derived_tags = with_tag(source_tags, "urgent")

print(source_tags)   # ['personal']
print(derived_tags)  # ['personal', 'urgent']
```

El patrón correcto depende del contrato:

- `append_tag_in_place` podría declarar mutación;
- `with_tag` podría declarar una lista nueva;
- un acumulador privado podría compartir el default deliberadamente, pero ocultarlo en la firma es una API frágil.

### 8.6 Defaults inmutables también se evalúan una vez

La regla no es “los mutables se evalúan una vez”. Todos los defaults se evalúan al definir. Los inmutables suelen ser seguros porque no pueden cambiar de estado.

**Ejemplo ejecutable:**

```python
def build_label(event_id, prefix="event"):
    return prefix + ":" + event_id


print(build_label("evt-1"))
print(build_label("evt-2"))
```

El mismo string default puede reutilizarse sin filtrar mutaciones.

### 8.7 Default calculado desde clock o environment

No uses una llamada al clock como default si necesitas el tiempo de cada invocación.

**Código incorrecto — tiempo congelado en la definición:**

```python
from datetime import UTC, datetime


def build_receipt(recorded_at=datetime.now(UTC)):
    return recorded_at


first = build_receipt()
second = build_receipt()

print(first == second)  # True
print(first is second)  # True
```

Ambas llamadas reutilizan el mismo objeto `datetime` default. Para tiempo actual, recibe el valor o el clock explícitamente; veremos ambos patrones en la sección 10.

> **Antes de continuar:** ejecuta el bug de `add_tag`, dibuja el único objeto lista y sus aliases, y corrígelo con `None`. Luego pasa una lista explícita y explica por qué compartirla sí es una decisión del caller.

---

## 9. Funciones puras, determinismo y side effects

### 9.1 El problema de los efectos invisibles

Una función puede parecer que recibe todo lo necesario, pero consultar datos externos dentro de su cuerpo.

**Código incorrecto — dependencia global y console:**

```python
minimum_score = 0.7


def is_relevant(score):
    result = score >= minimum_score
    print("evaluated", score)
    return result


print(is_relevant(0.8))
```

La firma muestra solo `score`, pero el resultado depende también de `minimum_score` y la llamada imprime. Un test que cambie el global puede afectar a otros tests.

### 9.2 Modelo mental

Para este módulo, una **función pura (pure function)** cumple dos propiedades prácticas:

1. su resultado depende solo de sus entradas explícitas;
2. no cambia estado observable fuera de la función.

Una función es determinista si las mismas entradas bajo el mismo contrato producen el mismo resultado. Una función que siempre imprime el mismo mensaje puede ser determinista, pero no pura porque produce un efecto de console.

Pureza y determinismo son propiedades útiles, no medallas morales.

### 9.3 Versión pura

**Ejemplo ejecutable:**

```python
def is_relevant(score, minimum_score):
    return score >= minimum_score


assert is_relevant(0.8, 0.7) is True
assert is_relevant(0.6, 0.7) is False
```

No hay que configurar globals ni capturar output. La regla puede usarse en CLI, tests o un adaptador futuro.

### 9.4 Catálogo práctico de efectos

Estas operaciones observan o cambian algo fuera de los argumentos y retornos locales:

| Efecto | Ejemplo | Por qué afecta reproducibilidad |
|---|---|---|
| Console | `print`, `input` | Depende de o modifica interacción externa. |
| Filesystem | leer o escribir un archivo | El contenido y los permisos cambian fuera del proceso. |
| Clock | `datetime.now(...)` | Dos llamadas pueden producir valores distintos. |
| Environment | `os.environ.get(...)` | Depende de configuración externa al argumento. |
| Random | `random.random()` | Produce una elección no determinada solo por la firma. |
| Estado global mutable | modificar una lista o contador global | El orden de llamadas altera resultados posteriores. |

Network y bases de datos también son efectos, pero se estudiarán en tracks posteriores.

Leer del filesystem también es un efecto aunque no modifique el archivo: el retorno depende de bytes, permisos y estado externos a la firma. Escribir cambia ese estado. PF-M6 enseñará las APIs y el lifecycle de archivos; PF-M2 solo exige que la regla del dominio no los consulte de forma oculta.

Random merece el mismo razonamiento. Una función puede recibir los mismos argumentos y elegir otro elemento porque consulta el estado de un generador.

**Ejemplo ejecutable — efecto no determinista controlado solo por una propiedad:**

```python
import random


def choose_event(events):
    return random.choice(events)


events = ["evt-1", "evt-2", "evt-3"]
selected = choose_event(events)

assert selected in events
```

El assert comprueba el contrato estable —el resultado pertenece a la entrada— sin fijar cuál debe salir. Si el resultado exacto importa para replay, la elección o el generador deben llegar desde una frontera controlada.

### 9.5 Los efectos son inevitables en los bordes

Un programa útil debe leer datos, conocer el tiempo, mostrar resultados o persistirlos. El objetivo no es eliminar efectos, sino evitar que atraviesen cada regla del dominio.

```text
CLI / filesystem / clock
          ↓
       adapter
          ↓
   pure domain function
```

La flecha representa flujo de datos, no una arquitectura completa. En PF-M2 basta con:

1. obtener el dato externo en un borde pequeño;
2. pasarlo a la lógica como argumento;
3. devolver un resultado;
4. ejecutar el siguiente efecto fuera de la regla.

### 9.6 Efecto en la frontera, regla en el centro

**Ejemplo ejecutable:**

```python
import os


def should_show_private(event, allow_private):
    if event["private"]:
        return allow_private
    return True


allow_private_text = os.environ.get("EIDOLON_ALLOW_PRIVATE", "false")
allow_private = allow_private_text == "true"

event = {"id": "evt-1", "private": True}
visible = should_show_private(event, allow_private)

print(visible)
```

El acceso al environment sigue siendo un efecto, pero `should_show_private` ya no lo oculta. Para probar la regla puedes pasar `True` o `False` directamente.

La gestión completa de variables de entorno, configuración y secretos se estudia en PF-M4 y Security. Este ejemplo solo marca la frontera.

### 9.7 Source data y derived data

Una transformación pura no debe disfrazar el derivado como fuente.

**Ejemplo ejecutable:**

```python
def build_search_text(source_text):
    return source_text.strip().casefold()


source_text = "  Canción en Mérida  "
search_text = build_search_text(source_text)

assert source_text == "  Canción en Mérida  "
assert search_text == "canción en mérida"
```

La pureza ayuda a preservar la fuente porque la función produce un valor nuevo. El nombre `search_text` mantiene visible que es derivado.

### 9.8 Cuándo una función impura es correcta

Una función `read_source`, `save_receipt` o `print_report` tiene un efecto como responsabilidad principal. Ocultarlo sería peor que admitirlo. Una buena función impura:

- tiene un nombre que revela el efecto;
- limita el número de cosas externas que toca;
- recibe configuración explícita cuando es razonable;
- devuelve información útil o deja un resultado observable definido;
- no mezcla reglas del dominio que podrían probarse por separado.

> **Antes de continuar:** clasifica estas operaciones como pura, determinista pero impura, o no determinista: sumar dos números; imprimir una suma; leer `os.environ`; recibir configuración como argumento; llamar `datetime.now(UTC)`; devolver un mensaje sin imprimirlo. Justifica cada respuesta.

---

## 10. Dependency boundaries básicas

### 10.1 De dependencia implícita a entrada explícita

Una dependencia es algo que una función necesita para cumplir su trabajo. Puede ser:

- un dato;
- configuración;
- un clock;
- un comportamiento.

En PF-M2 no usamos frameworks de dependency injection. Solo pasamos lo necesario mediante parámetros.

### 10.2 Recibir datos

La forma más simple de aislar un efecto es obtener el dato antes y pasarlo a una función pura.

**Ejemplo ejecutable:**

```python
def build_receipt(event_id, recorded_at):
    return {
        "event_id": event_id,
        "recorded_at": recorded_at,
    }


receipt = build_receipt(
    "evt-1",
    recorded_at="2026-08-25T20:00:00+00:00",
)

assert receipt["recorded_at"].endswith("+00:00")
```

`build_receipt` no sabe de dónde salió la hora. Esa ignorancia deliberada vuelve la regla reproducible.

### 10.3 Recibir un clock como comportamiento

A veces una función de borde es responsable de capturar el momento. Puede recibir una función `clock` y llamarla.

**Ejemplo ejecutable:**

```python
from datetime import UTC, datetime


def system_clock():
    return datetime.now(UTC)


def capture_receipt(event_id, *, clock):
    recorded_at = clock()
    return {
        "event_id": event_id,
        "recorded_at": recorded_at.isoformat(),
    }


def fixed_clock():
    return datetime(2026, 8, 25, 20, 0, tzinfo=UTC)


receipt = capture_receipt("evt-1", clock=fixed_clock)

print(receipt["recorded_at"])
# 2026-08-25T20:00:00+00:00
```

`capture_receipt` no es pura: llama a un comportamiento que puede consultar el tiempo. Sin embargo, su dependencia es visible y controlable. En producción podría recibir `system_clock`; en el ejemplo recibe `fixed_clock`.

Pasamos la función `fixed_clock`, no el resultado `fixed_clock()`. El parámetro `clock` queda enlazado con un objeto función y `clock()` lo llama después.

### 10.4 Recibir configuración específica

**Ejemplo ejecutable:**

```python
def filter_by_minimum_score(events, *, minimum_score):
    selected = []

    for event in events:
        if event["score"] >= minimum_score:
            selected.append(event)

    return selected


events = [
    {"id": "evt-1", "score": 0.5},
    {"id": "evt-2", "score": 0.9},
]

selected = filter_by_minimum_score(events, minimum_score=0.7)
assert selected == [{"id": "evt-2", "score": 0.9}]
```

Preferir `minimum_score` sobre un diccionario gigante de configuración reduce el acoplamiento. Pasa el objeto completo solo cuando la función realmente necesita ese contrato completo.

### 10.5 Recibir comportamiento

Una función también puede recibir otra función para decidir una política.

**Ejemplo ejecutable:**

```python
def is_accepted(event):
    return event["status"] == "accepted"


def filter_events(events, *, predicate):
    selected = []

    for event in events:
        if predicate(event):
            selected.append(event)

    return selected


events = [
    {"id": "evt-1", "status": "draft"},
    {"id": "evt-2", "status": "accepted"},
]

assert filter_events(events, predicate=is_accepted) == [events[1]]
```

Este patrón es útil cuando la variación real es comportamiento. No pases funciones por todas partes solo para parecer flexible. Una condición directa es mejor si no existe una política intercambiable.

### 10.6 Console como dependencia de un adaptador

La lógica puede devolver un mensaje y el borde decidir cómo emitirlo.

**Ejemplo ejecutable:**

```python
def build_summary(event):
    return event["id"] + " | " + event["status"]


def print_summary(event):
    message = build_summary(event)
    print(message)


print_summary({"id": "evt-1", "status": "accepted"})
# evt-1 | accepted
```

`build_summary` es pura. `print_summary` es un adaptador impuro deliberado.

### 10.7 Random: elegir afuera o recibir el generador

Si el dominio solo necesita un ID ya elegido, pásalo como dato. Si el borde es responsable de generarlo, recibe un comportamiento controlable.

**Ejemplo ejecutable:**

```python
def build_event(event_id, source_text):
    return {"id": event_id, "source_text": source_text}


event = build_event("evt-fixed-001", "Texto sintético")
assert event["id"] == "evt-fixed-001"
```

La generación aleatoria real no mejora esta regla. El diseño y seguridad de IDs se profundizarán cuando el modelo de dominio exista.

### 10.8 No es un framework

Recibir dependencias no obliga a construir contenedores, interfaces o factories. En PF-M2 una dependencia explícita puede ser solo un parámetro con buen nombre.

La regla práctica es:

> si una función lee algo que puede cambiar entre ejecuciones, considera obtenerlo en el borde y pasarlo como dato o comportamiento.

> **Antes de continuar:** cambia el ejemplo de `capture_receipt` para contar cuántas veces llama al clock sin usar globals. El contrato debe exigir exactamente una llamada por receipt.

---

## 11. Closures

### 11.1 Por qué existen

A veces quieres crear una función configurada sin pasar la misma opción en cada llamada.

**Ejemplo ejecutable:**

```python
def make_labeler(prefix):
    def build_label(event_id):
        return prefix + ":" + event_id

    return build_label


event_labeler = make_labeler("event")
memory_labeler = make_labeler("memory")

print(event_labeler("001"))   # event:001
print(memory_labeler("001"))  # memory:001
```

`build_label` usa `prefix` desde el scope enclosing. Cuando `make_labeler` termina, la función devuelta conserva acceso a ese binding. Esa combinación es una closure.

### 11.2 Modelo mental correcto

Una closure no copia necesariamente el valor visible en una fotografía del momento. Conserva acceso a bindings de scopes enclosing que necesita su cuerpo.

En el ejemplo, cada llamada a `make_labeler` crea un binding `prefix` diferente. Por eso las dos funciones devueltas se comportan de manera independiente.

### 11.3 Closure sin mutación

**Ejemplo ejecutable:**

```python
def make_minimum_filter(minimum_score):
    def is_relevant(event):
        return event["score"] >= minimum_score

    return is_relevant


strict_filter = make_minimum_filter(0.8)

print(strict_filter({"score": 0.9}))  # True
print(strict_filter({"score": 0.7}))  # False
```

El estado capturado es pequeño y estable. La función creada puede pasarse como `predicate` al filtro de la sección anterior.

### 11.4 Closure con estado mutable mediante `nonlocal`

**Ejemplo ejecutable:**

```python
def make_call_counter():
    count = 0

    def count_call():
        nonlocal count
        count += 1
        return count

    return count_call


first_counter = make_call_counter()
second_counter = make_call_counter()

print(first_counter())   # 1
print(first_counter())   # 2
print(second_counter())  # 1
```

Cada closure tiene su propio binding `count`. Es mejor que un único global si realmente necesitas contadores aislados, pero el cambio sigue siendo estado oculto para el caller.

### 11.5 Late binding

Las closures resuelven el binding cuando se ejecutan. Esto sorprende al crear varias funciones dentro de un loop.

**Código incorrecto — predice los tres resultados:**

```python
def make_multipliers():
    functions = []

    for factor in (1, 2, 3):
        def multiply(value):
            return value * factor

        functions.append(multiply)

    return functions


multipliers = make_multipliers()

print(multipliers[0](10))
print(multipliers[1](10))
print(multipliers[2](10))
```

Output:

```text
30
30
30
```

Las tres closures consultan el mismo binding `factor` de `make_multipliers` cuando se llaman; al terminar el loop contiene `3`.

Una corrección local puede capturar el valor actual mediante un default evaluado al ejecutar cada `def`:

**Ejemplo ejecutable:**

```python
def make_multipliers():
    functions = []

    for factor in (1, 2, 3):
        def multiply(value, factor=factor):
            return value * factor

        functions.append(multiply)

    return functions


multipliers = make_multipliers()

print(multipliers[0](10))  # 10
print(multipliers[1](10))  # 20
print(multipliers[2](10))  # 30
```

Aquí el default es un `int` inmutable y cada ejecución de `def` evalúa el `factor` visible en esa iteración.

No necesitas producir closures en loops con frecuencia en PF-M2. El ejemplo demuestra que capturan bindings, no una intuición vaga de “guardar variables”.

### 11.6 Cuándo no usar closures

Evítalas cuando:

- esconden estado de negocio que necesita inspección o provenance;
- acumulan varias operaciones y reglas;
- requieren `nonlocal` para muchos bindings;
- su configuración no puede observarse con facilidad;
- una función con argumentos explícitos sería más clara.

Este mecanismo volverá a aparecer cuando estudiemos decorators en PF-M7. No desarrollaremos decorators aquí.

> **Antes de continuar:** crea dos filtros mediante `make_minimum_filter`, uno con `0.5` y otro con `0.9`. Prueba el mismo evento con ambos. Luego explica qué binding conserva cada closure y por qué no necesita `global`.

---

## 12. Caso progresivo integrado: núcleo funcional pequeño de EIDOLON

El objetivo es aplicar PF-M1 y PF-M2 sin clases, archivos, JSON ni frameworks. Los `dict` y `list` son datos sintéticos; PF-M3 decidirá estructuras con mayor rigor.

### 12.1 Entrada sintética

**Ejemplo ejecutable:**

```python
raw_event = {
    "id": "evt-001",
    "source_text": "  Canción en Mérida 🎵  ",
    "tags": [" Personal ", "MÚSICA", "personal"],
    "status": "draft",
}
```

La fuente debe preservarse. `source_text` y `tags` originales no se sobrescriben para facilitar búsqueda.

### 12.2 `normalize_tags`: derivar sin mutar

**Continuación:** reutiliza `raw_event` de 12.1.

```python
import unicodedata


def normalize_tags(tags):
    """Devuelve tags normalizados y sin duplicados; no modifica la fuente."""
    normalized_tags = []

    for tag in tags:
        normalized = unicodedata.normalize("NFC", tag).strip().casefold()
        if normalized != "" and normalized not in normalized_tags:
            normalized_tags.append(normalized)

    return normalized_tags


search_tags = normalize_tags(raw_event["tags"])

print(search_tags)
# ['personal', 'música']
print(raw_event["tags"])
# [' Personal ', 'MÚSICA', 'personal']
```

La búsqueda `normalized not in normalized_tags` es suficiente para pocos datos sintéticos. PF-M3 enseñará cuándo otra colección reduce costo.

### 12.3 `is_valid_transition`: decisión pura

**Ejemplo ejecutable:**

```python
def is_valid_transition(current_status, target_status):
    if current_status == "draft":
        return target_status in ("accepted", "rejected")

    if current_status == "accepted":
        return target_status == "archived"

    return False


assert is_valid_transition("draft", "accepted") is True
assert is_valid_transition("accepted", "draft") is False
```

La función no cambia el evento. Responde una pregunta reusable.

### 12.4 `filter_events`: opciones keyword-only

**Ejemplo ejecutable:**

```python
def filter_events(events, *, status=None, required_tag=None):
    selected = []

    for event in events:
        if status is not None and event["status"] != status:
            continue
        if required_tag is not None and required_tag not in event["search_tags"]:
            continue
        selected.append(event)

    return selected


events = [
    {"id": "evt-1", "status": "draft", "search_tags": ["personal"]},
    {"id": "evt-2", "status": "accepted", "search_tags": ["música"]},
    {"id": "evt-3", "status": "accepted", "search_tags": ["personal"]},
]

result = filter_events(events, status="accepted", required_tag="personal")
assert result == [events[2]]
```

Los keywords evitan dos strings posicionales difíciles de distinguir. `None` significa “no aplicar este filtro”; una string vacía no se confunde con ausencia.

La lista de salida es nueva, pero contiene referencias a los mismos eventos seleccionados. El contrato solo promete filtrar sin mutar; no promete copiar cada evento.

### 12.5 `apply_correction`: fuente inmutable, vista derivada

**Ejemplo ejecutable:**

```python
def apply_correction(event, correction):
    if correction["target_event_id"] != event["id"]:
        raise ValueError("correction target no coincide")

    corrected_view = event.copy()
    corrected_view["display_text"] = correction["replacement_text"]
    corrected_view["correction_id"] = correction["id"]
    return corrected_view


event = {
    "id": "evt-1",
    "source_text": "La reunión fue el lunes",
    "display_text": "La reunión fue el lunes",
}
correction = {
    "id": "cor-1",
    "target_event_id": "evt-1",
    "replacement_text": "La reunión fue el martes",
}

corrected_view = apply_correction(event, correction)

assert event["display_text"] == "La reunión fue el lunes"
assert corrected_view["display_text"] == "La reunión fue el martes"
assert corrected_view is not event
```

La `Correction` no reescribe `source_text`. El modelo formal de entidades llega en PF-M5.

### 12.6 Construcción pura y clock en el borde

**Continuación:** este bloque reutiliza `normalize_tags` de 12.2.

```python
from datetime import UTC, datetime


def build_event(raw_event, *, recorded_at):
    return {
        "id": raw_event["id"],
        "source_text": raw_event["source_text"],
        "search_text": raw_event["source_text"].strip().casefold(),
        "source_tags": raw_event["tags"].copy(),
        "search_tags": normalize_tags(raw_event["tags"]),
        "status": raw_event["status"],
        "recorded_at": recorded_at.isoformat(),
    }


def fixed_clock():
    return datetime(2026, 8, 25, 20, 0, tzinfo=UTC)


raw_event = {
    "id": "evt-001",
    "source_text": "  Canción en Mérida 🎵  ",
    "tags": [" Personal ", "MÚSICA", "personal"],
    "status": "draft",
}

event = build_event(raw_event, recorded_at=fixed_clock())

assert event["source_text"] == "  Canción en Mérida 🎵  "
assert event["search_text"] == "canción en mérida 🎵"
assert event["source_tags"] == raw_event["tags"]
assert event["search_tags"] == ["personal", "música"]
assert event["recorded_at"] == "2026-08-25T20:00:00+00:00"
```

Si guardas este caso como script independiente, incluye `normalize_tags` antes de `build_event`.

La lógica no inventa `valid_time` y no confunde el clock de ingesta con el tiempo afirmado por la fuente. Solo construye `recorded_at` a partir de un dato aware proporcionado.

### 12.7 Adaptador mínimo de console

**Ejemplo ejecutable:**

```python
def build_event_summary(event):
    return event["id"] + " | " + event["status"]


def print_event_summary(event):
    print(build_event_summary(event))


event = {"id": "evt-001", "status": "draft"}
print_event_summary(event)
# evt-001 | draft
```

La console está aislada en `print_event_summary`; la representación se prueba con `build_event_summary`.

### 12.8 Qué no resuelve este caso

Todavía no incluye:

- selección rigurosa de colecciones o análisis de complejidad: PF-M3;
- módulos, configuración reproducible o environment management: PF-M4;
- dataclasses, entidades y type hints: PF-M5;
- parsing, excepciones de dominio, archivos o JSON: PF-M6;
- decorators: PF-M7;
- async: PF-M8;
- pytest avanzado, fixtures o monkeypatch: PF-M9.

La separación lograda es deliberadamente pequeña: efectos en bordes visibles y reglas del dominio en funciones deterministas.

### 12.9 Romperlo deliberadamente

Haz un cambio por vez y predice la propiedad que falla:

1. mueve `datetime.now(UTC)` dentro de `build_event`;
2. reemplaza `recorded_at` por un default `datetime.now(UTC)` en la firma;
3. usa `tags=[]` como default y muta la lista;
4. normaliza `raw_event["source_text"]` in place;
5. modifica `event` directamente en `apply_correction`;
6. lee `minimum_score` desde un global dentro de un predicado;
7. imprime dentro de `is_valid_transition`;
8. reemplaza la firma de `filter_events` por `**kwargs` y oculta campos requeridos.

Para cada bug identifica: contrato roto, efecto introducido y test mínimo que lo detectaría.

---

## 13. Ejercicios guiados con solución razonada

### Ejercicio guiado 1 — Imprimir frente a devolver

Repara esta función para poder comparar su resultado sin capturar la console:

**Código incorrecto:**

```python
def normalize_status(status):
    print(status.strip().casefold())


result = normalize_status(" Accepted ")
assert result == "accepted"
```

#### Solución razonada

**Ejemplo ejecutable:**

```python
def normalize_status(status):
    return status.strip().casefold()


result = normalize_status(" Accepted ")
assert result == "accepted"
```

`print` enviaba información a un efecto externo y la función devolvía `None`. `return` entrega el string al caller; imprimirlo sigue siendo posible fuera de la función.

**Variación:** devuelve `None` solo cuando el input sea `None`, pero conserva `""` como string vacío. Explica por qué truthiness no expresa ese contrato.

### Ejercicio guiado 2 — Hacer una opción keyword-only

La llamada `select_events(events, "accepted", 2)` no explica el significado de `2`. Cambia la firma para que `status` y `limit` deban nombrarse y conserva `limit=0`.

#### Solución razonada

**Ejemplo ejecutable:**

```python
def select_events(events, *, status=None, limit=None):
    if limit is not None and limit < 0:
        raise ValueError("limit no puede ser negativo")

    selected = []

    for event in events:
        if status is not None and event["status"] != status:
            continue
        if limit is not None and len(selected) == limit:
            break
        selected.append(event)

    return selected


events = [
    {"id": "evt-1", "status": "accepted"},
    {"id": "evt-2", "status": "accepted"},
]

assert select_events(events, status="accepted", limit=0) == []
assert select_events(events, status="accepted", limit=1) == [events[0]]
```

La comparación explícita con `None` preserva `0`. El `*` hace visibles las dos opciones en cada llamada.

### Ejercicio guiado 3 — Escribir un contrato verificable

Diseña `is_valid_confidence(value)` con este contrato:

- recibe `int` o `float`, pero `bool` no cuenta como medida válida;
- devuelve `True` si el valor está entre `0.0` y `1.0`, inclusive;
- devuelve `False` para otro valor o tipo;
- no imprime ni modifica estado.

#### Solución razonada

**Ejemplo ejecutable:**

```python
def is_valid_confidence(value):
    if isinstance(value, bool):
        return False
    if not isinstance(value, (int, float)):
        return False
    return 0.0 <= value <= 1.0


assert is_valid_confidence(0.0) is True
assert is_valid_confidence(1) is True
assert is_valid_confidence(True) is False
assert is_valid_confidence(-0.1) is False
assert is_valid_confidence("0.5") is False
```

`bool` es un subtipo de `int` en Python, por eso se descarta antes de comprobar `(int, float)`. Este detalle importa porque el contrato de dominio distingue una medida de un valor lógico. La función elige devolver `False` ante input inválido; otro contrato podría lanzar un error, pero debe declararlo.

### Ejercicio guiado 4 — Resolver LEGB

Predice el output y anota la letra que resuelve cada nombre usado por `inner`.

**Ejemplo ejecutable:**

```python
label = "global"


def outer(prefix):
    label = "enclosing"

    def inner(value):
        suffix = "local"
        return prefix + ":" + label + ":" + suffix + ":" + str(value)

    return inner(3)


print(outer("argument"))
```

#### Solución razonada

Output:

```text
argument:enclosing:local:3
```

- `value` y `suffix`: Local de `inner`;
- `prefix` y `label`: Enclosing de `outer`;
- `str`: Builtins;
- `label` global: oculto por el binding enclosing.

No aparece un nombre resuelto en Global dentro de la expresión final.

### Ejercicio guiado 5 — Corregir `UnboundLocalError`

Repara sin `global`:

**Failure case intencional:**

```python
processed = 0


def increment():
    processed = processed + 1
    return processed


try:
    increment()
except UnboundLocalError as error:
    print(type(error).__name__)
```

#### Solución razonada

**Ejemplo ejecutable:**

```python
def increment(current_value):
    return current_value + 1


processed = 0
processed = increment(processed)
processed = increment(processed)

assert processed == 2
```

La asignación de la versión rota hacía local a `processed` y trataba de leerlo antes de enlazarlo. Recibir y devolver revela el flujo de estado y permite probar `increment(10)` sin preparar nada externo.

### Ejercicio guiado 6 — Corregir un mutable default

Repara `append_error` para que dos llamadas sin lista no compartan estado y una lista explícita sí pueda usarse.

**Código incorrecto:**

```python
def append_error(message, errors=[]):
    errors.append(message)
    return errors
```

#### Solución razonada

**Ejemplo ejecutable:**

```python
def append_error(message, errors=None):
    if errors is None:
        errors = []

    errors.append(message)
    return errors


first = append_error("missing id")
second = append_error("missing status")
shared = []
append_error("one", shared)
append_error("two", shared)

assert first == ["missing id"]
assert second == ["missing status"]
assert first is not second
assert shared == ["one", "two"]
```

El patrón evita un default persistente. Mutar `shared` sigue siendo una decisión explícita del caller y debería formar parte del contrato.

### Ejercicio guiado 7 — Aislar environment

Refactoriza una regla que lee el environment para que su lógica pueda probarse sin cambiar el proceso.

**Código incorrecto:**

```python
import os


def can_export(event):
    mode = os.environ.get("EIDOLON_MODE", "safe")
    return mode == "full" and event["private"] is False
```

#### Solución razonada

**Ejemplo ejecutable:**

```python
def can_export(event, *, mode):
    return mode == "full" and event["private"] is False


public_event = {"id": "evt-1", "private": False}
private_event = {"id": "evt-2", "private": True}

assert can_export(public_event, mode="full") is True
assert can_export(private_event, mode="full") is False
assert can_export(public_event, mode="safe") is False
```

El adaptador futuro puede leer `os.environ` una vez y pasar `mode`. La política ya depende de entradas explícitas.

### Ejercicio guiado 8 — Inyectar un clock y verificar una llamada

Construye un clock controlado que registre cuántas veces fue llamado sin usar `global`.

#### Solución razonada

**Ejemplo ejecutable:**

```python
from datetime import UTC, datetime


def make_counted_clock(fixed_time):
    calls = 0

    def clock():
        nonlocal calls
        calls += 1
        return fixed_time

    def call_count():
        return calls

    return clock, call_count


def capture_receipt(event_id, *, clock):
    recorded_at = clock()
    return {"event_id": event_id, "recorded_at": recorded_at.isoformat()}


fixed_time = datetime(2026, 8, 25, 20, 0, tzinfo=UTC)
clock, call_count = make_counted_clock(fixed_time)
receipt = capture_receipt("evt-1", clock=clock)

assert receipt["recorded_at"] == "2026-08-25T20:00:00+00:00"
assert call_count() == 1
```

La closure encapsula estado pequeño de observación para el ejercicio. En PF-M9 aprenderás herramientas de testing más completas; aquí no necesitamos mocks ni frameworks.

### Ejercicio guiado 9 — Captura tardía en closures

Explica por qué todas las funciones devuelven el mismo prefijo y corrige el bug.

**Código incorrecto:**

```python
def make_labelers():
    labelers = []

    for prefix in ("event", "claim", "memory"):
        def label(value):
            return prefix + ":" + value

        labelers.append(label)

    return labelers


labelers = make_labelers()
```

#### Solución razonada

**Ejemplo ejecutable:**

```python
def make_labelers():
    labelers = []

    for prefix in ("event", "claim", "memory"):
        def label(value, prefix=prefix):
            return prefix + ":" + value

        labelers.append(label)

    return labelers


labelers = make_labelers()


assert labelers[0]("001") == "event:001"
assert labelers[1]("001") == "claim:001"
assert labelers[2]("001") == "memory:001"
```

La versión rota resolvía el único binding `prefix` al llamar, después del loop. La corrección evalúa un default inmutable en cada ejecución de `def`.

### Ejercicio guiado 10 — Aplicar una Correction sin mutar la fuente

Implementa una función que rechace una corrección dirigida a otro evento y produzca una vista nueva.

#### Solución razonada

**Ejemplo ejecutable:**

```python
def apply_correction(event, correction):
    if event["id"] != correction["target_event_id"]:
        raise ValueError("target_event_id no coincide")

    result = event.copy()
    result["display_text"] = correction["replacement_text"]
    result["applied_correction_id"] = correction["id"]
    return result


event = {
    "id": "evt-1",
    "source_text": "lunes",
    "display_text": "lunes",
}
correction = {
    "id": "cor-1",
    "target_event_id": "evt-1",
    "replacement_text": "martes",
}

result = apply_correction(event, correction)

assert result is not event
assert event["display_text"] == "lunes"
assert result["display_text"] == "martes"
assert result["source_text"] == "lunes"
```

El contrato conserva el acontecimiento fuente y deja visible la identidad de la corrección. No crea aún clases ni un event store.

---

## 14. Ejercicios independientes

No consultes los ejercicios guiados mientras resuelves. Antes de ejecutar, escribe la predicción y la parte del contrato que estás comprobando.

1. Escribe `format_event_id(number, *, width=4)` para devolver `evt-0007`. Haz `width` keyword-only y rechaza valores menores que `1`.
2. Implementa una función que devuelva `"missing"`, `"empty"` o `"present"` sin confundir `None` con `""`.
3. Construye una función con tres caminos de retorno y demuestra que ninguno cae accidentalmente en `None`.
4. Diseña una llamada donde dos argumentos posicionales sean fáciles de invertir. Corrige la firma o la llamada usando keywords.
5. Escribe una función con un positional-only razonable y defiende por qué el nombre de ese parámetro no debe formar parte del contrato. Después explica por qué no usarías `/` en una función de dominio común.
6. Crea `join_labels(separator, *labels)` y decide qué debe devolver cuando no recibe labels. Documenta esa decisión.
7. Reemplaza una función `**kwargs` con campos obligatorios ocultos por una firma explícita. Provoca el typo de un keyword en ambas versiones y compara los errores.
8. Redacta el contrato completo de `normalize_tags`: tipo esperado, valores válidos, precondiciones, resultado, errores, efectos e invariantes.
9. Implementa `is_valid_transition` con al menos cuatro casos positivos y cuatro negativos, sin cambiar estado.
10. Dibuja los scopes de tres funciones anidadas. Coloca un nombre distinto en cada nivel y predice cuál gana al eliminar bindings de adentro hacia afuera.
11. Produce un `UnboundLocalError` sin usar un nombre inexistente. Explica la decisión estática que lo causa.
12. Refactoriza un contador global para que reciba y devuelva estado. Demuestra dos flujos independientes.
13. Reproduce un bug de default mutable con un diccionario. Corrígelo con `None` y demuestra identidades distintas.
14. Muestra un caso donde una colección explícita compartida entre llamadas sea intencional. Nombra la función para revelar que muta in place.
15. Clasifica cinco funciones propias como puras o impuras. Para cada impura identifica console, filesystem, clock, environment, random o global state.
16. Separa una función que llama `datetime.now(UTC)` y normaliza texto en un borde impuro y una transformación pura.
17. Recibe un `clock` como keyword-only. Usa dos clocks fijos distintos y demuestra que el dominio produce receipts reproducibles.
18. Construye una función que reciba un `predicate` y filtre eventos. Prueba dos predicados sin modificar el filtro.
19. Crea una closure configurada con un prefijo inmutable. Después crea dos instancias y demuestra que no comparten configuración.
20. Reproduce late binding con tres closures en un loop y corrígelo. Explica por qué el default usado en la corrección no tiene el bug de lista mutable.
21. Audita un script anterior: enumera globals, lecturas de environment, clocks, prints y mutaciones de argumentos. Propón una refactorización incremental, no una reescritura.
22. Implementa `apply_correction` con datos anidados y demuestra si una shallow copy basta para tu contrato. Si no basta, reconstruye explícitamente solo las ramas modificadas usando PF-M1.

---

## 15. Preguntas conceptuales

Responde primero sin ejecutar. Después diseña el experimento mínimo que podría refutar tu respuesta.

1. ¿Qué diferencia existe entre definir una función y llamarla?
2. ¿Qué diferencia existe entre parámetro y argumento?
3. ¿Por qué `print` no sustituye a `return`?
4. ¿Qué devuelve Python cuando una llamada termina sin ejecutar `return`?
5. ¿Cuándo varios caminos de retorno mejoran claridad y cuándo esconden un `None` accidental?
6. ¿Por qué responsabilidad única no equivale a una línea por función?
7. ¿Cuándo un positional argument comunica suficiente intención?
8. ¿Qué problema resuelve un keyword-only argument?
9. ¿Cuándo un positional-only argument puede proteger la evolución de una API?
10. ¿Cuándo `*args` expresa una cantidad variable real y cuándo oculta roles distintos?
11. ¿Por qué `**kwargs` puede retrasar la detección de un campo requerido?
12. ¿Qué diferencia existe entre tipo esperado y dato válido?
13. ¿Qué relación existe entre una precondición y la excepción que comunica su incumplimiento?
14. ¿Qué invariante de EIDOLON debe preservar `apply_correction`?
15. ¿Qué orden representa LEGB?
16. ¿Por qué leer un global no requiere `global`, pero reasignarlo sí?
17. ¿Por qué una asignación posterior puede causar `UnboundLocalError` en una lectura anterior?
18. ¿Qué diferencia existe entre scope y lifetime?
19. ¿Por qué un objeto devuelto puede sobrevivir al nombre local que lo referenciaba?
20. ¿Qué cambia exactamente una declaración `global`?
21. ¿Qué binding selecciona `nonlocal` y qué requisito debe cumplir?
22. ¿Cuándo se evalúa una expresión default?
23. ¿Por qué `None` evita el bug de default mutable?
24. ¿Por qué una lista explícita pasada por el caller todavía puede compartirse?
25. ¿Qué diferencia existe entre una función pura y una función determinista con efectos?
26. ¿Por qué console, filesystem, clock, environment y random son efectos externos?
27. ¿Por qué un programa real no puede eliminar todos los efectos?
28. ¿Qué ventaja aporta pasar `recorded_at` como dato?
29. ¿Qué cambia si recibes un `clock` en lugar del valor del tiempo?
30. ¿Cuándo recibir comportamiento es más claro que usar una condición directa?
31. ¿Qué conserva una closure: una “foto” universal de valores o acceso a bindings concretos?
32. ¿Por qué aparece late binding en funciones creadas dentro de un loop?
33. ¿Cuándo una closure pequeña es adecuada y cuándo oculta demasiado estado?
34. ¿Cómo ayuda la lógica pura al replay determinista de EIDOLON?
35. ¿Qué conceptos usados aquí deben esperar a PF-M3–PF-M9 para su tratamiento completo?

---

## 16. Mini challenge — Pipeline funcional de eventos sintéticos

### 16.1 Objetivo

Construye un script ejecutable llamado `pf_m2_event_pipeline.py`. Debe transformar, filtrar y corregir eventos sintéticos mediante funciones pequeñas, contratos visibles y ausencia de estado global.

El challenge integra PF-M1 y PF-M2. No requiere clases, type hints, archivos, JSON, decorators, async, bases de datos ni frameworks.

### 16.2 Entrada obligatoria

**Ejemplo ejecutable — datos de entrada:**

```python
from datetime import UTC, datetime


raw_events = [
    {
        "id": "evt-0001",
        "source_text": "  Canción en Mérida 🎵  ",
        "tags": [" Personal ", "MÚSICA", "personal"],
        "status": "draft",
        "confidence": 0.0,
    },
    {
        "id": "evt-0002",
        "source_text": "Reunión del martes",
        "tags": ["Trabajo"],
        "status": "accepted",
        "confidence": None,
    },
]

correction = {
    "id": "cor-0001",
    "target_event_id": "evt-0002",
    "replacement_text": "Reunión del miércoles",
}


def fixed_clock():
    return datetime(2026, 8, 25, 20, 0, tzinfo=UTC)
```

### 16.3 Funciones requeridas

Implementa, como mínimo:

1. `normalize_tags(tags)`  
   Devuelve una lista nueva, aplica NFC, `strip()` y `casefold()`, elimina strings vacíos y duplicados, y no muta la fuente.

2. `is_valid_transition(current_status, target_status)`  
   Implementa `draft → accepted|rejected` y `accepted → archived`. Cualquier otra transición devuelve `False`.

3. `build_event(raw_event, *, recorded_at, normalizer)`  
   `recorded_at` y `normalizer` son keyword-only. Conserva source data, crea derivados explícitos y serializa el aware datetime recibido. Rechaza ID vacío y datetime naive mediante `ValueError`. No consulta el clock y entrega una copia de los tags al normalizer para proteger la fuente incluso si la dependencia está defectuosa.

4. `capture_event(raw_event, *, clock, normalizer)`  
   Es un adaptador pequeño: llama al clock exactamente una vez y delega la construcción a `build_event`. Su efecto debe quedar documentado.

5. `filter_events(events, *, status=None, required_tag=None)`  
   Aplica solo los filtros no `None` y devuelve una lista nueva.

6. `apply_correction(event, correction)`  
   Devuelve una vista nueva, conserva `source_text`, agrega `display_text` y el ID de corrección, y rechaza targets incompatibles. No muta el evento ni la corrección.

7. `build_summary(event)`  
   Devuelve un string y no imprime.

8. `print_summary(event)`  
   Es el único adaptador de console del challenge y usa `build_summary`.

### 16.4 Contratos obligatorios

- ninguna función usa `global`;
- ningún parámetro usa `[]` o `{}` como default;
- `None` se distingue de `0.0`, `""` y listas vacías;
- source data no se sobrescribe con derivados;
- toda función que muta una entrada debe rechazarse en code review;
- los errores de precondición se comunican, no se convierten en datos inventados;
- el clock y el normalizer son dependencias explícitas;
- las funciones puras pueden probarse sin console ni clock real.

### 16.5 Contrato mínimo de salida

Cada evento construido debe contener:

```text
id
source_text
search_text
source_tags
search_tags
status
confidence
recorded_at
```

La vista corregida añade:

```text
display_text
applied_correction_id
```

### 16.6 Comprobaciones mínimas

**Fragmento de asserts — ejecútalo después de tu implementación:**

```python
events = []

for raw_event in raw_events:
    event = capture_event(
        raw_event,
        clock=fixed_clock,
        normalizer=normalize_tags,
    )
    events.append(event)

assert events[0]["source_text"] == "  Canción en Mérida 🎵  "
assert events[0]["search_text"] == "canción en mérida 🎵"
assert events[0]["source_tags"] == [" Personal ", "MÚSICA", "personal"]
assert events[0]["source_tags"] is not raw_events[0]["tags"]
assert events[0]["search_tags"] == ["personal", "música"]
assert events[0]["confidence"] == 0.0
assert events[1]["confidence"] is None
assert events[0]["recorded_at"] == "2026-08-25T20:00:00+00:00"

accepted = filter_events(events, status="accepted")
assert accepted == [events[1]]

corrected = apply_correction(events[1], correction)
assert corrected is not events[1]
assert events[1]["source_text"] == "Reunión del martes"
assert corrected["source_text"] == "Reunión del martes"
assert corrected["display_text"] == "Reunión del miércoles"
assert corrected["applied_correction_id"] == "cor-0001"

assert is_valid_transition("draft", "accepted") is True
assert is_valid_transition("accepted", "draft") is False
assert build_summary(events[0]) == "evt-0001 | draft"


def make_counted_clock(fixed_time):
    calls = 0

    def clock():
        nonlocal calls
        calls += 1
        return fixed_time

    def call_count():
        return calls

    return clock, call_count


counted_clock, call_count = make_counted_clock(fixed_clock())
capture_event(
    raw_events[0],
    clock=counted_clock,
    normalizer=normalize_tags,
)
assert call_count() == 1
```

### 16.7 Failure cases obligatorios

Provoca y documenta:

1. `raw_event["id"] = ""`;
2. un naive datetime en `recorded_at`;
3. una `Correction` dirigida a otro evento;
4. un default mutable introducido deliberadamente en una copia rota;
5. una función que lea configuración desde un global;
6. un `normalizer` que modifique la lista fuente;
7. una versión que llame al clock dos veces para un solo evento;
8. una versión que imprima dentro de `build_summary` y por ello devuelva `None`.

Para cada caso escribe:

- síntoma observado;
- contrato roto;
- corrección mínima;
- assert que evita la regresión.

No necesitas construir una jerarquía de excepciones ni capturar todos los errores. Eso corresponde a PF-M6.

### 16.8 Criterio de aprobación

Apruebas el challenge si:

- el script puede ejecutarse desde cero;
- los asserts mínimos pasan;
- cada función tiene una responsabilidad explicable;
- no existe estado global mutable;
- puedes clasificar cada función como pura o impura y justificarlo;
- el clock se controla sin monkeypatch ni framework;
- los eventos fuente permanecen sin mutaciones;
- otra persona puede cambiar una regla de transición sin tocar console o captura de tiempo;
- puedes reproducir y explicar los ocho failure cases.

---

## 17. Resumen

- `def` crea una función; llamar ejecuta su cuerpo con bindings locales nuevos.
- Los parámetros pertenecen a la definición; los argumentos pertenecen a una llamada.
- `return` entrega un resultado; `print` produce un efecto de console.
- Una función sin retorno explícito devuelve `None`.
- Los caminos de retorno deben preservar el contrato, no solo evitar errores sintácticos.
- Positional, keyword, positional-only y keyword-only expresan distintos contratos de llamada.
- `*args` y `**kwargs` son apropiados cuando la variabilidad es real, no para evitar diseñar una firma.
- Un contrato distingue tipo esperado, dato válido, precondición, resultado, errores, efectos e invariantes.
- LEGB busca nombres en Local, Enclosing, Global y Builtins.
- Una asignación dentro de una función vuelve local al nombre salvo `global` o `nonlocal`.
- Scope y lifetime no son sinónimos; un objeto puede sobrevivir al binding local que lo produjo.
- `global` modifica un binding del módulo; `nonlocal` modifica un binding de una función enclosing.
- Los defaults se evalúan al ejecutar `def`, no en cada llamada.
- Un default mutable puede filtrar estado entre llamadas; `None` permite crear el objeto por llamada.
- Una función pura depende de entradas explícitas y no produce efectos observables externos.
- Determinismo no implica pureza: imprimir siempre el mismo mensaje sigue siendo un efecto.
- Console, filesystem, clock, environment, random y estado global son efectos que conviene aislar.
- Recibir datos, configuración, clocks o comportamientos hace visibles dependencias básicas.
- Una closure conserva acceso a bindings enclosing; `nonlocal` permite cambiar uno.
- Las closures en loops pueden mostrar late binding.
- EIDOLON necesita reglas deterministas para replay y adaptadores pequeños para efectos.
- Una `Correction` produce una vista derivada; no reescribe el acontecimiento fuente.

---

## 18. Checklist de dominio

- [ ] Puedo explicar qué ocurre al definir y al llamar una función.
- [ ] Puedo distinguir parámetros, argumentos, efectos y retorno.
- [ ] Puedo detectar una función que imprime cuando debería devolver.
- [ ] Puedo predecir un `None` implícito en todos los caminos de ejecución.
- [ ] Puedo justificar la responsabilidad de una función sin medirla solo por líneas.
- [ ] Puedo elegir positional o keyword arguments según claridad y estabilidad.
- [ ] Puedo implementar un keyword-only argument que proteja el contrato.
- [ ] Puedo explicar cuándo positional-only es pertinente y cuándo reduce legibilidad.
- [ ] Puedo usar `*args` y `**kwargs` sin ocultar campos obligatorios.
- [ ] Puedo escribir un contrato con entradas, validez, precondiciones, resultado, errores, efectos e invariantes.
- [ ] Puedo distinguir tipo correcto de dato válido.
- [ ] Puedo escribir una docstring que agregue información y coincida con los asserts.
- [ ] Puedo descomponer una regla por responsabilidades sin fragmentación artificial.
- [ ] Puedo resolver nombres paso a paso mediante LEGB.
- [ ] Puedo detectar shadowing de globals y builtins.
- [ ] Puedo explicar la diferencia entre scope y lifetime.
- [ ] Puedo provocar y corregir `UnboundLocalError`.
- [ ] Puedo explicar qué bindings seleccionan `global` y `nonlocal`.
- [ ] Puedo reemplazar estado global por entradas y retornos explícitos.
- [ ] Puedo reproducir el bug de mutable default argument.
- [ ] Puedo corregir un default mutable con `None` sin confundir una colección vacía explícita.
- [ ] Puedo distinguir compartir deliberadamente un objeto de compartirlo por un default accidental.
- [ ] Puedo clasificar una función como pura, determinista impura o no determinista.
- [ ] Puedo separar domain logic de console, clock y environment.
- [ ] Puedo recibir datos, configuración, un clock o un predicate mediante parámetros simples.
- [ ] Puedo explicar qué parte continúa siendo impura al recibir un clock.
- [ ] Puedo construir y probar una closure con estado pequeño y estable.
- [ ] Puedo explicar late binding y corregir un caso reproducible.
- [ ] Puedo detectar cuándo una closure oculta estado que debería ser explícito.
- [ ] Puedo implementar `normalize_tags`, `is_valid_transition`, `filter_events` y `apply_correction` con contratos claros.
- [ ] Puedo preservar source data y producir derived data sin mutaciones silenciosas.
- [ ] Puedo completar el mini challenge y defender cada frontera.

---

## 19. Preparación para labs y EIDOLON 0.0a

Después de dominar PF-M2 debes poder comenzar:

- **PF-L03 — Funciones sin estado oculto:** es el laboratorio principal. Debes refactorizar un script global en funciones puras y un adaptador CLI pequeño, y demostrar bugs de defaults y scopes.
- **PF-L01 — Diagnóstico reproducible:** ahora puedes convertir una especificación pequeña en contratos y evidencia aislada.
- **PF-L02 — Unicode y tiempo hostiles:** PF-M2 permite mover normalización y conversión temporal a funciones deterministas sin destruir la fuente.
- **PF-L04 — Índice de entidades:** puedes crear predicados y filtros claros; PF-M3 enseñará la colección y el costo adecuados.
- **PF-L14 — Pytest de fronteras:** todavía no debes iniciarlo completo. PF-M2 prepara clocks y environment explícitos; PF-M9 enseñará fixtures, parametrización y monkeypatch.
- **EIDOLON 0.0a:** ya puedes separar reglas de transición, filtros, normalización y correcciones de CLI, filesystem y clock. Aún faltan colecciones, packaging, modelado, errores, persistencia y testing profesional.

### Evidencia antes de avanzar a PF-M3

1. el script del mini challenge con todos los asserts;
2. una tabla que clasifique sus funciones como puras o impuras y enumere dependencias;
3. los diez ejercicios guiados ejecutados con predicciones previas;
4. al menos ocho ejercicios independientes, incluidos 10, 13, 16, 18, 20 y 22;
5. un diagrama LEGB explicado con un failure case de `UnboundLocalError`;
6. una nota breve titulada **“Qué estado no escondería y por qué”**;
7. una explicación oral de cinco minutos o nota equivalente sobre cómo PF-M2 prepara replay determinista.

Este módulo contribuye al **CHECKPOINT PF-C1 — Código determinista**: debes poder explicar scopes y funciones sin documentación y construir el mismo filtro de forma clara antes de medir alternativas.

---

## 20. Recursos de ampliación

La explicación fundamental está contenida en este módulo. Para verificar sintaxis y profundizar selectivamente consulta:

- [Python Tutorial — Defining Functions](https://docs.python.org/3/tutorial/controlflow.html#defining-functions), para llamadas, defaults, keywords y parámetros especiales;
- [Python Reference — Execution model](https://docs.python.org/3/reference/executionmodel.html#naming-and-binding), para binding y resolución de nombres;
- [Python Reference — Function definitions](https://docs.python.org/3/reference/compound_stmts.html#function-definitions), para el contrato exacto de la sintaxis;
- [`global` y `nonlocal`](https://docs.python.org/3/reference/simple_stmts.html#the-global-statement), para sus reglas formales.

Los libros y recursos compartidos permanecen en [`PF.11 Recursos recomendados`](../../02_curriculum/01_programming_foundations.md#pf11-recursos-recomendados). Úsalos para ampliar, no para sustituir los modelos y ejercicios de este capítulo.

---

## 21. Límite del módulo

PF-M2 termina aquí. La recursión no se desarrolla como tema central; se estudiará con algoritmos en Computer Science Foundations. Las colecciones y comprehensions pertenecen a **PF-M3**; módulos y packages a **PF-M4**; POO, dataclasses y type hints a **PF-M5**; excepciones, archivos y JSON a **PF-M6**; decorators y context managers a **PF-M7**; async/await a **PF-M8**; y pytest avanzado, debugging y logging a **PF-M9**.

Tampoco se introducen backend, bases de datos, AI ni una arquitectura completa de ports/adapters. La frontera lograda es más pequeña y comprobable: funciones con contratos claros, scopes predecibles y efectos visibles.
