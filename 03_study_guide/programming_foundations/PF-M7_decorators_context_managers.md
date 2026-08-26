# PF-M7 — Decorators y context managers como políticas

**Track:** Programming Foundations  
**Competencias:** D1.1; refuerza D1.2, D3.1 y D11.1  
**Fase:** P0  
**Nivel objetivo:** Profesional para el alcance del módulo  
**Prerequisites:** PF-M1, PF-M2, PF-M3, PF-M4, PF-M5, PF-M6  
**Build:** EIDOLON 0.0a  
**Curriculum source:** [PF-M7](../../02_curriculum/01_programming_foundations.md#pf-m7--decorators-y-context-managers-como-políticas)  
**Status:** approved

Una operación de dominio debería explicar su regla principal. Cuando varias operaciones repiten timing, tracing o cleanup, copiar esa política alrededor de cada una crea inconsistencias. Ocultarla detrás de sintaxis sofisticada tampoco ayuda. PF-M7 estudia dos mecanismos para extraerla sin volver invisible el control flow:

- un **decorator** transforma un callable en otro callable;
- un **context manager** controla entrada y salida de una región `with`.

El hilo conductor es:

```text
comportamiento explícito
        ↓
aparece una preocupación repetida
        ↓
¿conviene extraerla?
        ↓
decorator o context manager
        ↓
contrato visible
        ↓
cleanup / metadata / errores preservados
```

PF-M2 ya enseñó funciones como objetos, closures, scopes, `*args` y `**kwargs`. PF-M5 introdujo decorators built-in. PF-M6 enseñó exceptions, `try/finally`, `with`, resources y temporal + replace. Aquí se recupera solo lo indispensable.

No se estudian async decorators/context managers, typing avanzado, descriptors, metaclasses, concurrency, database transactions ni logging avanzado.

## Resultados de aprendizaje

Al terminar deberías poder:

- explicar una higher-order function sin convertir PF-M7 en programación funcional avanzada;
- construir un wrapper y trazar qué nombres captura su closure;
- desazucarar `@decorator` como una asignación explícita;
- preservar retorno, argumentos y exception propagation;
- demostrar el daño de un wrapper que retorna `None` o silencia errores;
- usar `functools.wraps` y comprobar metadata relevante;
- construir y desazucarar un decorator parametrizado;
- predecir aplicación y ejecución de decorators apilados;
- detectar estado mutable oculto dentro de un closure;
- envolver un method sin perder `self`;
- elegir decorator o función explícita según el control flow;
- medir elapsed time mediante `time.perf_counter()`;
- registrar metadata sintética sin payload sensible;
- explicar `with` como entrada, valor opcional, salida y exception policy;
- implementar `__enter__` y `__exit__`;
- predecir los tres argumentos de exception recibidos por `__exit__`;
- distinguir retorno falsy y truthy de `__exit__`;
- garantizar cleanup sin suprimir exceptions accidentalmente;
- construir un manager generator-based con `@contextmanager` y un solo `yield`;
- elegir class o `@contextmanager` según estado y comportamiento;
- hacer visible resource ownership;
- predecir orden inverso de cleanup en managers anidados;
- implementar una exportación temporal derivada que promueve solo en éxito;
- explicar límites de atomic replace y durabilidad;
- preservar source data y separar dominio, observabilidad y lifecycle.

## Cómo estudiar este módulo

Para cada decorator:

1. escribe primero la llamada sin `@`;
2. identifica original function, decorator y wrapper;
3. predice retorno, exception, metadata y side effects;
4. desazucara la pila completa;
5. pregunta si una función explícita sería más legible.

Para cada context manager:

1. identifica quién adquiere el resource;
2. indica qué retorna la entrada;
3. provoca éxito y excepción;
4. comprueba cleanup en ambas rutas;
5. verifica si la exception continúa;
6. declara qué garantía no ofrece el manager.

### Convenciones de código

- **Ejemplo ejecutable:** bloque autónomo o archivo completo.
- **Continuación:** depende solo del bloque inmediatamente anterior.
- **Código incorrecto:** antipatrón deliberado.
- **Failure case:** provoca el fallo o contrato roto indicado.
- **Fragmento:** omite contexto explícitamente.
- **Solución parcial:** resuelve una política local, no el caso integrado.

Los ejemplos se validan con Python 3.14. Duraciones reales no se fijan como output; se comprueba que sean no negativas. `assert` verifica contratos locales sin adelantar la estrategia de PF-M9.

### Sintaxis de apoyo

- `functools.wraps` es un decorator de la standard library;
- `time.perf_counter()` mide duración transcurrida, no tiempo de dominio;
- `contextlib.contextmanager` convierte un generator de un `yield` en context manager;
- `NamedTemporaryFile` y `Path.replace` se reutilizan desde PF-M6;
- los tipos precisos de decorators genéricos con `ParamSpec` quedan fuera.

---

## 1. El problema: una política repetida rodea la operación

### 1.1 Duplicación alrededor de varias functions

**Código repetido:**

```python
from time import perf_counter


started = perf_counter()
first_result = build_summary()
first_elapsed = perf_counter() - started
print("build_summary", first_elapsed)

started = perf_counter()
second_result = export_records()
second_elapsed = perf_counter() - started
print("export_records", second_elapsed)
```

La lógica se repite, puede medir con clocks distintos, olvidar exceptions o registrar payload accidentalmente. Timing es una preocupación transversal: rodea varias operaciones sin ser su regla de dominio.

### 1.2 Extraer no significa esconder

Antes de `@`, expresa la operación como value:

```python
def run_with_notice(operation):
    print("before")
    result = operation()
    print("after")
    return result
```

La function recibe otra function. El caller todavía ve la composición:

```python
result = run_with_notice(build_summary)
```

Un decorator automatiza una transformación parecida al definir el callable. Solo mejora el diseño si la política y sus efectos siguen siendo comprensibles.

### Predice

¿Qué ocurre con `"after"` si `operation()` raises y no existe `finally`?

### Explica

¿Qué parte es dominio y cuál es observabilidad en el primer bloque?

### Refactoriza

Escribe una function explícita que mida dos operaciones sin usar todavía `@`.

---

## 2. Higher-order functions al nivel necesario

### 2.1 Recibir una function

**Ejemplo ejecutable:**

```python
def apply_twice(func, value):
    return func(func(value))


def add_one(value):
    return value + 1


print(apply_twice(add_one, 3))
```

Output:

```text
5
```

Una **higher-order function** recibe o devuelve otra function. Python enlaza `func` al mismo object function que referencia `add_one`.

### 2.2 Devolver una function

**Ejemplo ejecutable:**

```python
def make_prefixer(label_text):
    def add_prefix(value):
        return prefix_text + value

    prefix_text = label_text + ": "
    return add_prefix


label = make_prefixer("event")
print(label("evt-001"))
```

Output:

```text
event: evt-001
```

La inner function conserva bindings del enclosing scope. PF-M2 desarrolló closures y LEGB; PF-M7 usa esa capacidad para capturar original function y configuración.

### Predice

¿Qué objects referencian `label` y `add_prefix` después de retornar `make_prefixer`?

### Modifica

Crea `apply_three_times` sin recursion ni estado global.

### Explica

¿Por qué “higher-order” describe inputs/outputs y no una jerarquía de classes?

---

## 3. El wrapper básico

### 3.1 Construcción línea por línea

**Ejemplo ejecutable:**

```python
def trace(func):
    def wrapper():
        print("before")
        result = func()
        print("after")
        return result

    return wrapper


def run():
    print("body")
    return "done"


traced_run = trace(run)
print(traced_run())
```

Output:

```text
before
body
after
done
```

Modelo:

```text
trace recibe original run
wrapper captura func mediante closure
trace retorna wrapper
traced_run referencia wrapper
wrapper llama original run
```

`func` se resuelve en el enclosing scope de `trace`. `result` es local a cada llamada del wrapper.

### 3.2 Dos identities

**Continuación:**

```python
print(traced_run is run)
print(traced_run.__name__)
```

Output:

```text
False
wrapper
```

Sin metadata preservation, el callable público tiene identity y nombre del wrapper.

### Predice

¿Qué function imprime `"body"` y cuál imprime `"before"`?

### Desazucara

Todavía no existe `@`: dibuja `run → trace(run) → wrapper`.

### Metadata

¿Por qué `traced_run.__name__` no describe la operación original?

---

## 4. `@decorator` es syntactic sugar

### 4.1 Dos formas equivalentes

**Ejemplo ejecutable:**

```python
def trace(func):
    def wrapper():
        print("trace")
        return func()

    return wrapper


@trace
def run():
    return "ok"


print(run())
```

Conceptualmente equivale a:

```python
def run():
    return "ok"


run = trace(run)
```

Output:

```text
trace
ok
```

La asignación ocurre al ejecutar la definición de la function, normalmente durante import del módulo. Después, el nombre `run` referencia el wrapper retornado.

### 4.2 El original sigue capturado

Aunque `run` ya apunta al wrapper, el closure conserva la original function como `func`. No existe búsqueda recursiva del nombre global `run` para ejecutar el body.

### Predice

Después de decorar, ¿`run` referencia la original function o el wrapper?

### Desazucara

Convierte:

```python
@trace
def export():
    return 3
```

en definición + asignación explícita.

### Explica

¿En qué momento se llama `trace(run)` y en qué momento se llama `run()`?

---

## 5. Preservar retorno y exception propagation

### 5.1 Failure case: retorno perdido

**Código incorrecto ejecutable:**

```python
def broken_trace(func):
    def wrapper():
        result = func()
        print("observed")
        # Falta return result.

    return wrapper


@broken_trace
def event_count():
    return 3


print(event_count())
```

Output:

```text
observed
None
```

El decorator cambió silenciosamente `int` por `None`. Ejecutar la original no basta: el wrapper posee el contrato público.

### 5.2 Corrección

```python
def trace(func):
    def wrapper():
        result = func()
        print("observed")
        return result

    return wrapper
```

### 5.3 Failure case: exception silenciada

**Código incorrecto:**

```python
def hide_failures(func):
    def wrapper():
        try:
            return func()
        except Exception:
            return None

    return wrapper
```

Un `ValueError` esperado y un `NameError` por bug se vuelven indistinguibles de un retorno válido `None`. Se pierden tipo, traceback y causa, justo la evidencia enseñada en PF-M6.

Un wrapper de observabilidad normalmente deja propagar. Si observa, usa `raise` para relanzar la misma exception.

### Predice

¿Qué recibe el caller de `event_count` bajo `broken_trace`?

### Detecta el bug

¿Qué contratos mezcla `hide_failures` y por qué el nombre no basta para hacerlo seguro?

### Repara

Haz que un wrapper agregue `"failure"` a una lista y vuelva a lanzar la exception original.

---

## 6. `*args` y `**kwargs`

### 6.1 Envolver firmas distintas

**Ejemplo ejecutable:**

```python
def trace(func):
    def wrapper(*args, **kwargs):
        print(f"calling {func.__name__}")
        return func(*args, **kwargs)

    return wrapper


@trace
def event_label(event_id, *, prefix="event"):
    return f"{prefix}:{event_id}"


print(event_label("evt-001", prefix="E"))
```

Output:

```text
calling event_label
E:evt-001
```

El wrapper reenvía positional y keyword arguments. No los inspecciona ni modifica.

### 6.2 Flexibilidad no significa contrato preciso

`*args, **kwargs` permite envolver varias firmas en runtime, pero la firma escrita del wrapper es genérica. `wraps` ayudará a introspection, no resuelve por sí solo typing estático preciso. `ParamSpec` y typing avanzado quedan para profundización posterior.

### Predice

¿Qué ocurre si el wrapper llama `func(args, kwargs)` en lugar de desempaquetar?

### Modifica

Agrega un segundo positional argument y comprueba que se reenvía.

### Explica

¿Por qué el wrapper no debería normalizar `kwargs` silenciosamente?

---

## 7. Metadata y `functools.wraps`

### 7.1 El problema observable

**Ejemplo ejecutable:**

```python
def trace(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)

    return wrapper


@trace
def load_event(event_id):
    """Load one synthetic event."""
    return event_id


print(load_event.__name__)
print(load_event.__doc__)
```

Output:

```text
wrapper
None
```

Debugging, documentation generators, introspection y tooling ven metadata del wrapper.

### 7.2 Preservarla

**Ejemplo ejecutable:**

```python
from functools import wraps


def trace(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)

    return wrapper


@trace
def load_event(event_id):
    """Load one synthetic event."""
    return event_id


print(load_event.__name__)
print(load_event.__doc__)
print(load_event.__wrapped__.__name__)
```

Output:

```text
load_event
Load one synthetic event.
load_event
```

`wraps(func)` copia metadata seleccionada y establece `__wrapped__`. No hace que wrapper y original tengan la misma identity ni garantiza que preserven comportamiento.

### 7.3 Metadata no corrige semántica

Un wrapper con `@wraps` todavía puede perder retorno, mutar argumentos o suprimir exceptions. Metadata preservation es necesaria para un decorator transparente, no suficiente.

### Metadata

¿Qué cambia en `__name__`, `__doc__` y `__wrapped__`?

### Predice

¿`load_event is load_event.__wrapped__` será verdadero?

### Detecta el bug

Un decorator usa `wraps` pero retorna siempre `True`. ¿Qué preservó y qué rompió?

---

## 8. Decorators parametrizados

### 8.1 Tres capas

**Ejemplo ejecutable:**

```python
from functools import wraps


def announce(label):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            print(label)
            return func(*args, **kwargs)

        return wrapper

    return decorator


@announce("journal")
def entry_count(entries):
    return len(entries)


print(entry_count(["event", "claim"]))
```

Output:

```text
journal
2
```

```text
announce("journal") → decorator
decorator(original)  → wrapper
wrapper(arguments)   → original result
```

### 8.2 Desazucarado

```python
def entry_count(entries):
    return len(entries)


entry_count = announce("journal")(entry_count)
```

`label` y `func` quedan capturados por closures diferentes. La configuración se evalúa al definir.

### 8.3 Retry solo como forma mental

`@retry_limit(3)` tendría la misma estructura de tres capas, pero una política real de retry necesita idempotencia, clasificación de errores, delay y límites. PF-M7 no enseña network retries mediante un ejemplo engañosamente pequeño.

### Predice

¿Cuántas functions se crean o retornan antes de llamar `entry_count(...)`?

### Desazucara

Expande `@announce("export")` sin `@`.

### Explica

¿Por qué `announce("journal")` se ejecuta al definir y wrapper al llamar?

---

## 9. Closures y estado oculto

### 9.1 Configuración pequeña

Capturar un label inmutable o un callable original expresa la configuración del decorator. El closure mantiene esos bindings mientras el wrapper exista.

### 9.2 Failure case: contador persistente sorpresivo

**Ejemplo ejecutable de riesgo:**

```python
from functools import wraps


def count_calls(func):
    calls = 0

    @wraps(func)
    def wrapper(*args, **kwargs):
        nonlocal calls
        calls += 1
        print(f"calls={calls}")
        return func(*args, **kwargs)

    return wrapper


@count_calls
def ping():
    return "pong"


ping()
ping()
```

Output:

```text
calls=1
calls=2
```

El estado vive tanto como wrapper y es compartido por todos sus callers. Tests dependen del orden; futuras llamadas concurrentes crearían preguntas no cubiertas aquí. Si el decorator necesita mucho estado, lifecycle o coordinación, reconsidera el diseño.

### Predice

¿Crear otra function decorada comparte el mismo `calls` o crea otro closure?

### Explica

¿Por qué un label capturado es menos riesgoso que un dict mutable de estado?

### Refactoriza

Haz explícito el contador como objeto recibido por la function en vez de estado invisible.

---

## 10. Orden de múltiples decorators

### 10.1 Aplicación bottom-up

```python
@a
@b
def operation():
    ...
```

equivale a:

```python
def operation():
    ...


operation = a(b(operation))
```

`b` recibe primero la original; `a` recibe el resultado de `b`.

### 10.2 Ejecución outside-in

**Ejemplo ejecutable:**

```python
from functools import wraps


def mark(label):
    def decorator(func):
        print(f"apply {label}")

        @wraps(func)
        def wrapper(*args, **kwargs):
            print(f"enter {label}")
            result = func(*args, **kwargs)
            print(f"exit {label}")
            return result

        return wrapper

    return decorator


@mark("A")
@mark("B")
def operation():
    print("body")


operation()
```

Output:

```text
apply B
apply A
enter A
enter B
body
exit B
exit A
```

Las decorator expressions se evalúan de arriba hacia abajo durante la definición; los decorators resultantes se aplican bottom-up. Al llamar, el outer wrapper A entra primero y sale último.

### Orden

Desazucara la pila y señala cuál wrapper captura a cuál.

### Predice

Invierte `@mark("A")` y `@mark("B")`. Escribe el output completo antes de ejecutar.

### Explica

¿Por qué “el de arriba corre primero” es una regla incompleta?

---

## 11. Methods y decorators ya conocidos

### 11.1 Un wrapper recibe `self` entre args

**Ejemplo ejecutable:**

```python
from functools import wraps


def announce_call(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        print(func.__name__)
        return func(*args, **kwargs)

    return wrapper


class Journal:
    def __init__(self, name):
        self.name = name

    @announce_call
    def label(self, prefix):
        return f"{prefix}:{self.name}"


print(Journal("synthetic").label("journal"))
```

Output:

```text
label
journal:synthetic
```

`self` llega como primer positional argument y se reenvía. No hace falta profundizar en descriptors o method binding para usar este contrato.

### 11.2 Puente desde PF-M5

Ya utilizaste:

- `@property`;
- `@classmethod`;
- `@staticmethod`;
- `@dataclass`.

Son decorators con contratos específicos. Reconocer la sintaxis no implica que debas recrear sus internals.

### Predice

¿Qué ocurre si wrapper descarta `args` al envolver `label`?

### Explica

¿Por qué `@dataclass` demuestra que un decorator puede recibir una class, no solo una function?

---

## 12. Cuándo un decorator aporta claridad

### 12.1 Criterios

Puede ser apropiado cuando:

- la preocupación realmente rodea varias operaciones;
- el contrato añadido es pequeño y nombrable;
- retorno y exceptions permanecen comprensibles;
- los side effects están documentados;
- el order en una pila es defendible;
- el caller no necesita conocer detalles internos para usar la function.

Ejemplos razonables: timing, métricas, tracing, instrumentation, caching bajo contrato claro y autorización declarativa futura.

### 12.2 Pregunta de lectura

> ¿El comportamiento sigue siendo comprensible leyendo la function y el nombre del decorator?

Si necesitas abrir cuatro archivos para descubrir que un decorator modifica payload, escribe, reintenta y suprime errores, la abstracción ocultó control flow crítico.

### Decide

Clasifica: timing uniforme; transition de Event; normalización de un único tag; autorización futura; corrección que referencia source.

### Explica

¿Por qué una regla central de dominio suele merecer una llamada explícita?

---

## 13. Cuándo no usar decorator

### 13.1 Control flow oculto

Evita esconder:

- lógica de dominio central;
- mutación de argumentos;
- cambio de tipo o semántica de retorno;
- filesystem o network I/O inesperado;
- recuperación de exceptions no declarada;
- estado global;
- múltiples side effects bajo un nombre vago.

### 13.2 Function explícita

**Código difícil de defender:**

```python
@normalize_event
@save_event
def apply_correction(event, correction):
    return corrected_event(event, correction)
```

¿`normalize_event` muta? ¿`save_event` escribe aunque la corrección falle? ¿En qué orden? Una composición explícita muestra decisiones:

```python
corrected = corrected_event(event, correction)
record = event_to_record(corrected)
append_record(journal_path, record)
```

La segunda forma puede ser más larga y aun así más segura.

### 13.3 Side effects y función pura

Una pure domain function no debe convertirse accidentalmente en filesystem + mutation + instrumentation por una pila invisible. Un decorator puede añadir side effects; nombre, documentación y ubicación deben hacerlo visible.

### Detecta el bug

¿Qué operación del ejemplo decorado podría ocurrir antes de validar la Correction?

### Refactoriza

Convierte una pila que normaliza, guarda e imprime en llamadas explícitas.

### Decide

¿Un cache simple es correcto si la function lee el reloj? Explica qué contrato faltaría.

---

## 14. Observabilidad EIDOLON sin payload sensible

### 14.1 Medir elapsed time

`datetime.now()` expresa wall-clock time. `time.perf_counter()` usa un clock apropiado para medir intervalos; su valor absoluto no es timestamp de dominio.

```text
valid_time / recorded_at → tiempo del dominio
perf_counter delta       → duración de una operación
```

### 14.2 Decorator parametrizado con sink explícito

**Ejemplo ejecutable:**

```python
from functools import wraps
from time import perf_counter


observations: list[dict[str, object]] = []


def observe_operation(operation_id, sink):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            started = perf_counter()
            outcome = "interrupted"
            try:
                result = func(*args, **kwargs)
            except Exception:
                outcome = "failure"
                raise
            else:
                outcome = "success"
                return result
            finally:
                sink.append(
                    {
                        "operation_id": operation_id,
                        "operation": func.__name__,
                        "outcome": outcome,
                        "elapsed_seconds": perf_counter() - started,
                    }
                )

        return wrapper

    return decorator


@observe_operation("op-001", observations)
def count_events(events):
    return len(events)


assert count_events(["evt-001", "evt-002"]) == 2
assert observations[0]["operation_id"] == "op-001"
assert observations[0]["operation"] == "count_events"
assert observations[0]["outcome"] == "success"
assert observations[0]["elapsed_seconds"] >= 0

print(
    observations[0]["operation_id"],
    observations[0]["operation"],
    observations[0]["outcome"],
)
```

Output:

```text
op-001 count_events success
```

No registra `args`, `kwargs`, retorno ni payload. El sink se pasa como dependencia de ejemplo; no es un logger global de producción.

### 14.3 Exception observada y propagada

**Ejemplo ejecutable:**

```python
from functools import wraps


events = []


def observe(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except Exception:
            events.append(("failure", func.__name__))
            raise

    return wrapper


@observe
def fail():
    raise ValueError("synthetic failure")


try:
    fail()
except ValueError as exc:
    print(type(exc).__name__, events[0])
```

Output:

```text
ValueError ('failure', 'fail')
```

`raise` sin argumento conserva la misma exception y traceback. Transformarla solo sería correcto si observabilidad poseyera explícitamente ese contrato, lo cual aquí no ocurre.

### Predice

¿Qué outcome registra el primer decorator si `count_events` raises `ValueError`?

### Metadata

Comprueba `count_events.__name__` y explica por qué importa para la observation.

### Privacy

Enumera tres datos que el decorator evita registrar y por qué.

---

## 15. Decorators, typing y demasiada magia

### 15.1 Firma runtime e información estática

Un wrapper `(*args, **kwargs)` acepta muchas formas, pero su annotation precisa no aparece automáticamente. `wraps` ayuda a runtime metadata e introspection; no es un sistema de typing genérico.

PF-M7 no introduce `ParamSpec`, `Concatenate`, variance ni advanced generics. Para decorators simples, mantén el body transparente y verifica con el checker qué información conserva la herramienta elegida.

### 15.2 Cinco políticas invisibles

```python
@authorize
@retry
@cache
@trace
@normalize
def operation(payload):
    ...
```

No existe un máximo numérico universal. El problema es si order, errors, mutation y side effects ya no pueden explicarse localmente. Una function explícita o un pequeño orchestrator puede ser más legible.

### 15.3 Estado incidental

Capturar counters, caches o sinks mutables cambia lifetime y testing. Si ese estado tiene identidad, methods o cleanup, una class explícita puede expresar ownership mejor que closure.

### Explica

¿Qué preserva `wraps` y qué no prueba sobre typing?

### Orden

¿Qué preguntas debes responder antes de aceptar una pila con cache y retry?

### Refactoriza

Reescribe una pila de tres políticas como tres llamadas con nombres.

---

## 16. De `try/finally` a context manager

### 16.1 Lifecycle repetido

```python
resource = acquire()
try:
    use(resource)
finally:
    release(resource)
```

El patrón es correcto si `acquire` completa y `release` corresponde al mismo resource. Repetirlo en varios callers puede olvidar cleanup o mezclar ownership.

### 16.2 Modelo mental

```text
entrar
↓
adquirir / preparar
↓
bloque with
↓
salir
↓
cleanup
```

Un context manager encapsula la política de entrada/salida. La lógica del bloque sigue visible.

### Predice

Si `use` raises, ¿qué parte del patrón explícito se ejecuta?

### Ownership

¿Quién adquiere y quién libera en el código anterior?

### Explica

¿Por qué context manager no es sinónimo de “abrir archivo”?

---

## 17. Qué significa `with`

### 17.1 Cuatro responsabilidades

```python
with resource_manager() as resource:
    use(resource)
```

Conceptualmente:

1. evalúa el manager;
2. entra y obtiene un valor opcional;
3. ejecuta el bloque;
4. sale con información sobre éxito o exception;
5. decide cleanup y si la exception continúa.

`as resource` recibe el valor retornado por entrada, no necesariamente el manager.

### 17.2 Frontera temporal

El resource suele ser válido dentro de la región. Devolver un file cerrado o temporary path ya eliminado rompe ownership.

### Predice

Si `__enter__` retorna `"token"`, ¿qué nombre enlaza `as value`?

### Explica

¿Qué control conserva el context manager cuando el bloque raises?

---

## 18. Protocolo `__enter__` / `__exit__`

### 18.1 Class mínima

**Ejemplo ejecutable:**

```python
class ManagedResource:
    def __init__(self, events):
        self.events = events
        self.active = False

    def __enter__(self):
        self.active = True
        self.events.append("enter")
        return "resource-value"

    def __exit__(self, exc_type, exc, tb):
        self.active = False
        name = None if exc_type is None else exc_type.__name__
        self.events.append(("exit", name))
        return False


events = []
manager = ManagedResource(events)

with manager as value:
    events.append(("body", value, manager.active))

print(events)
print(manager.active)
```

Output:

```text
['enter', ('body', 'resource-value', True), ('exit', None)]
False
```

`__enter__` devuelve el valor para `as`. En salida normal, `exc_type`, `exc` y `tb` son `None`. `return False` significa que una exception, si existe, no se suprime.

### 18.2 Fallo durante entrada

Si `__enter__` raises, el bloque no comienza y Python no llama `__exit__` porque la entrada no terminó. Si `__enter__` adquirió parcialmente varios resources, debe limpiarlos antes de propagar. Por eso conviene que la adquisición sea pequeña o use managers internos ya seguros.

### 18.3 Exception dentro del bloque

**Ejemplo ejecutable:**

```python
class InspectExit:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc, tb):
        print(exc_type.__name__, str(exc), tb is not None)
        return False


try:
    with InspectExit():
        raise ValueError("synthetic")
except ValueError:
    print("propagated")
```

Output:

```text
ValueError synthetic True
propagated
```

`__exit__` recibe tipo, instance y traceback. No necesita capturar alrededor del bloque para observarlos.

### Predice

¿Qué recibe `__exit__` bajo salida normal?

### Exception flow

¿Por qué aparece `"propagated"` después de `return False`?

### Modifica

Haz que `__enter__` retorne un dict y comprueba que `as` recibe ese dict, no el manager.

---

## 19. Exception suppression: truthy y falsy

### 19.1 Contrato exacto

- retorno falsy de `__exit__`: exception continúa;
- retorno truthy: exception se considera manejada y se suprime;
- en salida normal, el retorno no cambia un fallo inexistente.

`None` es falsy y suele ser el retorno implícito apropiado para cleanup que no maneja errors.

### 19.2 Failure case: bug oculto

**Código incorrecto ejecutable:**

```python
class HideEverything:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc, tb):
        return True


with HideEverything():
    missing_name

print("continued")
```

Output:

```text
continued
```

El `NameError` desapareció. El programa continúa desde un estado cuya operación no terminó. Retornar `True` no es cleanup; es una decisión de error handling.

### 19.3 Supresión explícita y estrecha

Puede ser válida si ignorar un tipo concreto forma parte del contrato, pero debe verificarse el `exc_type` exacto y documentarse. `contextlib.suppress` hace visible esa intención para casos acotados; no debe envolver bloques amplios.

### Predice

¿Qué ocurre si `__exit__` retorna `0`? ¿Y `"handled"`?

### Detecta el bug

¿Qué evidencia perdió `HideEverything`?

### Repara

Haz que cleanup retorne `False` y comprueba que `NameError` se propaga.

---

## 20. Cleanup garantizado

### 20.1 Éxito y exception

**Ejemplo ejecutable:**

```python
class Lifecycle:
    def __init__(self, events):
        self.events = events

    def __enter__(self):
        self.events.append("setup")
        return "resource"

    def __exit__(self, exc_type, exc, tb):
        self.events.append("cleanup")
        return False


normal = []
with Lifecycle(normal):
    normal.append("body")

failed = []
try:
    with Lifecycle(failed):
        failed.append("body")
        raise ValueError("failure")
except ValueError:
    failed.append("observed")

print(normal)
print(failed)
```

Output:

```text
['setup', 'body', 'cleanup']
['setup', 'body', 'cleanup', 'observed']
```

Cleanup ocurre antes de que el outer caller observe la propagated exception.

### 20.2 Ejemplos de resources

- file handle;
- temporary file/directory;
- lock conceptual;
- transaction futura.

PF-M7 no enseña locks ni database transactions. Los menciona para mostrar que context manager modela lifecycle, no un tipo de resource específico.

### Ownership

¿Qué object posee la obligación de agregar `"cleanup"`?

### Predice

¿En qué orden aparecen cleanup y observed?

---

## 21. `contextlib.contextmanager`

### 21.1 Del protocolo a una function generator-based

**Ejemplo ejecutable:**

```python
from contextlib import contextmanager


@contextmanager
def managed_resource(events):
    events.append("setup")
    try:
        yield "resource-value"
    finally:
        events.append("cleanup")


events = []
with managed_resource(events) as value:
    events.append(("body", value))

print(events)
```

Output:

```text
['setup', ('body', 'resource-value'), 'cleanup']
```

`@contextmanager` es un decorator que transforma una generator function con un `yield` en factory de context managers.

### 21.2 Mapa del `yield`

```text
antes del yield  → enter / setup
valor del yield  → valor para as
bloque with      → se ejecuta mientras generator está suspendido
después del yield→ exit / teardown
```

Debe producir exactamente un valor. No se enseña aquí la teoría general de generators.

### Desazucara

Identifica decorator, original generator function y manager producido al llamar.

### Predice

¿Qué recibe `value` y cuándo se agrega `cleanup`?

### Explica

¿Cómo combina `@contextmanager` los dos temas del módulo?

---

## 22. Exceptions alrededor de `yield`

### 22.1 El bloque falla

**Ejemplo ejecutable:**

```python
from contextlib import contextmanager


@contextmanager
def lifecycle(events):
    events.append("setup")
    try:
        yield
    finally:
        events.append("cleanup")


events = []
try:
    with lifecycle(events):
        events.append("body")
        raise ValueError("synthetic")
except ValueError:
    events.append("propagated")

print(events)
```

Output:

```text
['setup', 'body', 'cleanup', 'propagated']
```

La exception se inyecta en el generator en el punto de `yield`. `finally` ejecuta cleanup y, como no se suprime, continúa al caller.

### 22.2 Failure case: capturar demasiado

**Código incorrecto:**

```python
@contextmanager
def hide_errors():
    try:
        yield
    except Exception:
        pass
```

El generator termina normalmente después de capturar; `contextmanager` interpreta la exception como manejada. Es equivalente en riesgo a un `__exit__` truthy indiscriminado.

### Exception flow

¿Dónde “reaparece” la exception del bloque dentro del generator?

### Detecta el bug

¿Por qué `except Exception: pass` cambia el contrato del `with`?

### Repara

Usa `finally` para cleanup sin capturar el error.

---

## 23. Class o `@contextmanager`

### 23.1 Class puede expresar una abstracción real

Elige class cuando:

- existe estado significativo antes/después;
- hay múltiples methods públicos;
- el lifecycle pertenece a un object con identidad;
- callers necesitan inspeccionar estado deliberadamente.

### 23.2 Generator-based puede reducir ceremony

Elige `@contextmanager` cuando:

- setup y teardown son cortos;
- un valor se entrega al bloque;
- no hacen falta otros methods;
- `try/finally` expresa todo el lifecycle.

### 23.3 No son equivalentes a cualquier function

Si no existe region de lifecycle, una function regular es más directa. No conviertas cada `try/finally` local en abstraction reusable.

### Decide

Elige para: temporary export; connection pool complejo futuro; normalizar tag; medir una única expression.

### Explica

¿Qué estado justificaría una class `AtomicExport`?

---

## 24. Context manager class para exportación temporal

### 24.1 Contrato

```text
enter   → crea temporal y devuelve su Path
body    → caller escribe derived output
success → close ya ocurrió en caller; manager promueve
failure → no promueve
exit    → limpia temporal restante y propaga errors
```

### 24.2 Implementación ejecutable

**Ejemplo ejecutable:**

```python
from pathlib import Path
from tempfile import NamedTemporaryFile, TemporaryDirectory


class AtomicExport:
    def __init__(self, target, *, protected_source=None):
        self.target = Path(target)
        self.protected_source = (
            None
            if protected_source is None
            else Path(protected_source)
        )
        self.temporary_path = None

    def __enter__(self):
        if (
            self.protected_source is not None
            and self.target.resolve() == self.protected_source.resolve()
        ):
            raise ValueError("target must differ from protected source")

        self.target.parent.mkdir(parents=True, exist_ok=True)
        file = NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            dir=self.target.parent,
            prefix=f".{self.target.name}.",
            suffix=".tmp",
            delete=False,
        )
        self.temporary_path = Path(file.name)
        file.close()
        return self.temporary_path

    def __exit__(self, exc_type, exc, tb):
        try:
            if exc_type is None:
                if self.temporary_path is None:
                    raise RuntimeError("temporary path was not created")
                self.temporary_path.replace(self.target)
        finally:
            if (
                self.temporary_path is not None
                and self.temporary_path.exists()
            ):
                self.temporary_path.unlink()
        return False


with TemporaryDirectory() as directory:
    root = Path(directory)
    target = root / "export.jsonl"

    with AtomicExport(target) as temporary:
        temporary.write_text('{"event_id":"evt-001"}\n', encoding="utf-8")

    assert target.read_text(encoding="utf-8") == (
        '{"event_id":"evt-001"}\n'
    )

print("class export: PASS")
```

Output:

```text
class export: PASS
```

El manager crea y posee el temporary path. El caller posee cada file handle que abre mediante `write_text` o un nested `with`. La promoción ocurre solo si el body sale normalmente.

### 24.3 Failure path

**Ejemplo ejecutable:**

```python
from pathlib import Path
from tempfile import NamedTemporaryFile, TemporaryDirectory


class AtomicExport:
    def __init__(self, target):
        self.target = Path(target)
        self.temporary_path = None

    def __enter__(self):
        file = NamedTemporaryFile(
            mode="w",
            encoding="utf-8",
            dir=self.target.parent,
            delete=False,
        )
        self.temporary_path = Path(file.name)
        file.close()
        return self.temporary_path

    def __exit__(self, exc_type, exc, tb):
        try:
            if exc_type is None:
                self.temporary_path.replace(self.target)
        finally:
            if self.temporary_path.exists():
                self.temporary_path.unlink()
        return False


with TemporaryDirectory() as directory:
    root = Path(directory)
    target = root / "export.jsonl"
    target.write_text("old\n", encoding="utf-8")
    temporary = None

    try:
        with AtomicExport(target) as temporary:
            temporary.write_text("partial\n", encoding="utf-8")
            raise ValueError("synthetic failure")
    except ValueError:
        pass

    assert target.read_text(encoding="utf-8") == "old\n"
    assert temporary is not None
    assert not temporary.exists()

print("failed export cleanup: PASS")
```

Output:

```text
failed export cleanup: PASS
```

### 24.4 Garantía limitada

El manager expresa lifecycle y usa replace en el mismo directory. No realiza validación de contenido, `fsync` del file/directory, locking ni transacción de múltiples archivos. Reduce el riesgo de target parcial bajo garantías del OS/filesystem; no promete durabilidad absoluta.

### Ownership

Cleanup también puede fallar. Si `unlink` raises mientras ya se propagaba otro error, esa nueva exception puede reemplazar la evidencia visible. Un diseño de producción necesita una política para conservar ambos fallos; PF-M9 profundizará failure injection y diagnóstico.

¿Quién limpia temporary y quién cierra el file usado para escribirlo?

### Exception flow

¿Por qué el `ValueError` sigue visible?

### Detecta la promesa falsa

Corrige: “Usar `with AtomicExport` convierte cualquier export en transaction durable”.

---

## 25. Variante con `@contextmanager`

### 25.1 Setup/teardown simple

**Ejemplo ejecutable:**

```python
from contextlib import contextmanager
from pathlib import Path
from tempfile import NamedTemporaryFile, TemporaryDirectory


@contextmanager
def atomic_export(target):
    target = Path(target)
    target.parent.mkdir(parents=True, exist_ok=True)
    file = NamedTemporaryFile(
        mode="w",
        encoding="utf-8",
        dir=target.parent,
        prefix=f".{target.name}.",
        suffix=".tmp",
        delete=False,
    )
    temporary = Path(file.name)
    file.close()

    try:
        yield temporary
        temporary.replace(target)
    finally:
        if temporary.exists():
            temporary.unlink()


with TemporaryDirectory() as directory:
    target = Path(directory) / "export.txt"
    with atomic_export(target) as temporary:
        temporary.write_text("derived\n", encoding="utf-8")

    assert target.read_text(encoding="utf-8") == "derived\n"

print("generator export: PASS")
```

Output:

```text
generator export: PASS
```

Si el body raises, la línea después de `yield` no se alcanza; `finally` limpia. Si `replace` falla, `finally` también limpia y el fallo se propaga.

### 25.2 Tradeoff con class

La variante es compacta. La class puede exponer estado deliberado o aceptar una validator policy con methods claros. No elijas por menor número de líneas únicamente.

### Predice

¿Qué statement se omite si el body raises antes de retornar?

### Desazucara

Señala la función decorada por `contextmanager` y el manager que produce al llamarla.

---

## 26. Source, derived output y ownership

### 26.1 Tres roles

```text
source data      → input preservado
temporary output → resource intermedio poseído por manager
derived export   → target promovido bajo política
```

Un context manager no autoriza sobrescribir source. El caller debe elegir target y declarar qué source está protegido.

### 26.2 Failure case: target igual al source

`AtomicExport(target, protected_source=source)` rechaza igualdad antes de crear temporary. Esta comprobación es una política local; no resuelve aliases, symlinks hostiles o races de seguridad profundas.

### 26.3 Ownership visible

Pregunta siempre:

> ¿Quién adquiere el resource y quién es responsable de liberarlo?

- quien abre file handle suele cerrarlo;
- quien crea temporary debe limpiarlo;
- manager promueve solo el resource que creó;
- no retornes un handle cuyo lifecycle ya terminó;
- source owner conserva el original.

### Ownership

Si una helper crea temporary pero retorna solo un open file y olvida quién lo borra, ¿qué contrato falta?

### Explica

¿Por qué protected source y target son decisiones del caller, no de la pure domain function?

---

## 27. Managers anidados y orden de cleanup

### 27.1 Entrada en orden, salida inversa

**Ejemplo ejecutable:**

```python
from contextlib import contextmanager


@contextmanager
def layer(name, events):
    events.append(f"enter {name}")
    try:
        yield name
    finally:
        events.append(f"exit {name}")


events = []
with layer("A", events) as a, layer("B", events) as b:
    events.append(f"body {a} {b}")

print(events)
```

Output:

```text
['enter A', 'enter B', 'body A B', 'exit B', 'exit A']
```

Los resources se adquieren de izquierda a derecha y se liberan en orden inverso. Conceptualmente, el último adquirido es el primero en cleanup.

### 27.2 Nesting explícito equivalente

```python
with layer("A", events) as a:
    with layer("B", events) as b:
        ...
```

Si B falla al entrar, A ya fue adquirido y debe salir. Python conserva esa estructura de cleanup.

### Predice

Agrega C y escribe el orden completo.

### Ownership

¿Qué manager posee cada cleanup?

### Explica

¿Por qué liberar A antes de B podría romper una dependencia de B sobre A?

---

## 28. `contextlib` útil pero limitado

La standard library también incluye:

- `closing`: llama `close` sobre un object que no ofrece context protocol;
- `nullcontext`: región opcional sin setup/teardown real;
- `suppress`: suprime exceptions específicas bajo contrato explícito;
- `ExitStack`: administra un número dinámico de context managers.

PF-M7 no los convierte en inventario. `ExitStack` es [NICE] para recursos dinámicos y se estudia cuando exista ese problema.

Especial cuidado:

```python
with suppress(FileNotFoundError):
    cache_path.unlink()
```

puede ser correcto si “cache ya ausente” equivale al resultado deseado. No uses `suppress(Exception)` ni envuelvas lógica de dominio.

### Decide

¿Es válido suprimir `FileNotFoundError` al eliminar un cache derivado? ¿Y al leer el único source journal?

### Explica

¿Qué problema distinto resuelve `ExitStack`?

---

## 29. Anti-patterns de decorators y context managers

### 29.1 Decorator que muta argumentos

**Código incorrecto:**

```python
def add_default_tag(func):
    def wrapper(event, *args, **kwargs):
        event["tags"].append("default")
        return func(event, *args, **kwargs)

    return wrapper
```

El caller observa aliasing y mutación no declarada. Repetir la llamada agrega tags. La corrección es una transformation explícita que retorna una nueva representation o una operación de dominio nombrada.

### 29.2 Decorator que cambia retorno

**Código incorrecto:**

```python
def stringify(func):
    def wrapper(*args, **kwargs):
        return str(func(*args, **kwargs))

    return wrapper
```

Una function `-> int` ahora produce `str`. Si cambiar semántica es realmente el objetivo, el nombre y type contract deben hacerlo explícito; una function normal suele ser más legible.

### 29.3 Swallowing en ambos mecanismos

```python
def wrapper(*args, **kwargs):
    try:
        return func(*args, **kwargs)
    except Exception:
        pass
```

```python
def __exit__(self, exc_type, exc, tb):
    return True
```

Ambos pueden ocultar bugs y pérdida de datos. Observabilidad y cleanup no implican error recovery.

### 29.4 Context manager gigante

**Código incorrecto — fragmento:**

```python
class EverythingManager:
    def __enter__(self):
        self.file = open(...)
        self.events = parse_validate_and_mutate(self.file)
        print("loaded")
        return self.events

    def __exit__(self, exc_type, exc, tb):
        save_domain_changes()
        write_logs_with_payload()
        decide_business_rules()
        self.file.close()
        return True
```

Lifecycle, domain, parsing, UI, logging y suppression quedan acoplados. Extrae pure transformations y deja al manager adquirir/liberar el resource.

### 29.5 Magia acumulada

Una pila de decorators y un giant manager pueden esconder más que la duplicación original. Compara siempre con composición explícita.

### Detecta el bug

¿Qué alias observa la mutación de `event["tags"]`?

### Refactoriza

Separa `EverythingManager` en una pure function, una operación de I/O y un manager pequeño.

### Exception flow

¿Qué dos snippets suprimen exceptions?

---

## 30. Caso progresivo integrado EIDOLON

### 30.1 Objetivo y fronteras

```text
synthetic Event tuple
      ↓ pure export_records
@observe_operation
      ↓ safe metadata only
with atomic_export
      ↓ temporary owned by manager
derived JSONL target
```

La pure function no conoce decorator, sink, filesystem ni temporary. El decorator no conoce payload. El manager no valida reglas de Event.

### 30.2 Ejemplo ejecutable integrado

**Archivo completo:**

```python
import json
from contextlib import contextmanager
from dataclasses import dataclass
from functools import wraps
from pathlib import Path
from tempfile import NamedTemporaryFile, TemporaryDirectory
from time import perf_counter


@dataclass(frozen=True)
class Event:
    event_id: str
    text: str


def event_to_record(event):
    return {
        "schema_version": 1,
        "record_type": "event",
        "event_id": event.event_id,
        "text": event.text,
    }


def export_records(events):
    return [event_to_record(event) for event in events]


def observe_operation(operation_id, sink):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            started = perf_counter()
            outcome = "interrupted"
            try:
                result = func(*args, **kwargs)
            except Exception:
                outcome = "failure"
                raise
            else:
                outcome = "success"
                return result
            finally:
                sink.append(
                    {
                        "operation_id": operation_id,
                        "operation": func.__name__,
                        "outcome": outcome,
                        "elapsed_seconds": perf_counter() - started,
                    }
                )

        return wrapper

    return decorator


@contextmanager
def atomic_export(target, *, protected_source=None):
    target = Path(target)
    if (
        protected_source is not None
        and target.resolve() == Path(protected_source).resolve()
    ):
        raise ValueError("target must differ from protected source")

    target.parent.mkdir(parents=True, exist_ok=True)
    file = NamedTemporaryFile(
        mode="w",
        encoding="utf-8",
        dir=target.parent,
        prefix=f".{target.name}.",
        suffix=".tmp",
        delete=False,
    )
    temporary = Path(file.name)
    file.close()

    try:
        yield temporary
        temporary.replace(target)
    finally:
        if temporary.exists():
            temporary.unlink()


observations = []
observed_export_records = observe_operation(
    "op-export-001",
    observations,
)(export_records)

source_events = (
    Event("evt-001", "Llegué a casa 🏠"),
    Event("evt-002", "Escuché música 🎵"),
)

with TemporaryDirectory() as directory:
    root = Path(directory)
    source_path = root / "source.jsonl"
    target_path = root / "derived-export.jsonl"
    source_text = "source remains unchanged\n"
    source_path.write_text(source_text, encoding="utf-8")

    records = observed_export_records(source_events)
    with atomic_export(
        target_path,
        protected_source=source_path,
    ) as temporary:
        lines = [
            json.dumps(record, ensure_ascii=False)
            for record in records
        ]
        temporary.write_text(
            "\n".join(lines) + "\n",
            encoding="utf-8",
        )

    assert source_path.read_text(encoding="utf-8") == source_text
    assert source_events[0].text == "Llegué a casa 🏠"
    assert target_path.exists()
    assert len(target_path.read_text(encoding="utf-8").splitlines()) == 2
    assert observations[0]["operation"] == "export_records"
    assert observations[0]["outcome"] == "success"
    assert observations[0]["elapsed_seconds"] >= 0
    assert observed_export_records.__name__ == "export_records"

print("PF-M7 integrated policy: PASS")
```

Output:

```text
PF-M7 integrated policy: PASS
```

### 30.3 Qué no demuestra

- no valida schema JSONL completo;
- no garantiza durabilidad absoluta;
- no coordina writers;
- no registra payload, paths completos ni secrets;
- no convierte JSONL en database;
- no implementa async o logging de producción.

### Predice

¿Qué artifacts cambian si `export_records` falla antes de entrar al manager?

### Metadata

¿Qué prueba el assert de `__name__` y qué no prueba?

### Ownership

Identifica owner de source, target, temporary y file handles.

### Refactoriza

Reescribe el flujo sin decorator y compara visibilidad.

---

## 31. Ejercicios guiados con solución razonada

### Ejercicio guiado 1 — Decorator simple

**Objetivo:** transformar un callable sin `@`.

**Antes de ejecutar:** identifica original y wrapper.

**Solución:**

```python
def announce(func):
    def wrapper():
        print("before")
        return func()

    return wrapper


def operation():
    return "ok"


operation = announce(operation)
assert operation() == "ok"
```

**Razonamiento:** la asignación sustituye el binding público; closure conserva original. El criterio es preservar el retorno, no solo imprimir.

### Ejercicio guiado 2 — Retorno y exception

**Objetivo:** mantener contrato observable.

**Solución:**

```python
def transparent(func):
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)

    return wrapper
```

No agregues `try/except` si no existe recovery. El criterio es mismo result en éxito y misma exception visible en fallo.

### Ejercicio guiado 3 — Argumentos arbitrarios

**Objetivo:** reenviar positional/keyword arguments.

```python
@transparent
def label(event_id, *, prefix):
    return f"{prefix}:{event_id}"


assert label("evt-001", prefix="event") == "event:evt-001"
```

La coincidencia accidental sería aceptar solo cero argumentos. La variación usa dos formas.

### Ejercicio guiado 4 — Metadata con `wraps`

**Objetivo:** preservar nombre, docstring y `__wrapped__`.

```python
from functools import wraps


def transparent(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)

    return wrapper
```

Comprueba metadata además de resultado. `wraps` no corrige un wrapper semánticamente incorrecto.

### Ejercicio guiado 5 — Decorator parametrizado

**Objetivo:** distinguir factory, decorator y wrapper.

```python
def with_label(label):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            print(label)
            return func(*args, **kwargs)

        return wrapper

    return decorator
```

Desazucarado: `operation = with_label("x")(operation)`. El criterio es explicar cuándo se evalúa cada capa.

### Ejercicio guiado 6 — Pila de dos decorators

**Objetivo:** predecir order.

```python
@with_label("outer")
@with_label("inner")
def operation():
    return "ok"
```

Aplicación: inner, luego outer. Ejecución: outer imprime antes que inner. El criterio es derivarlo de `outer(inner(original))`.

### Ejercicio guiado 7 — Class context manager

**Objetivo:** implementar entrada, valor y salida.

```python
class Managed:
    def __enter__(self):
        self.active = True
        return "value"

    def __exit__(self, exc_type, exc, tb):
        self.active = False
        return False
```

Comprueba active dentro/fuera y valor `as`. El criterio incluye exception propagation, no solo ruta normal.

### Ejercicio guiado 8 — Cleanup ante exception

**Objetivo:** demostrar salida antes de propagación.

```python
manager = Managed()
try:
    with manager:
        raise ValueError("synthetic")
except ValueError:
    pass

assert manager.active is False
```

`return False` conserva el error. Cambiarlo a True rompería el criterio.

### Ejercicio guiado 9 — Convertir a `@contextmanager`

**Objetivo:** expresar setup/yield/cleanup.

```python
from contextlib import contextmanager


@contextmanager
def managed():
    active = {"value": True}
    try:
        yield active
    finally:
        active["value"] = False
```

La dict solo hace observable el estado educativo. El criterio es un `yield` y cleanup bajo éxito/fallo.

### Ejercicio guiado 10 — Export manager

**Objetivo:** promover solo tras salida normal.

Usa `AtomicExport` de 24.2. Prueba target previo bajo éxito y fallo. No fijes paths temporales ni afirmes durabilidad física.

El criterio es:

```text
success → target nuevo, temporary ausente
failure → target anterior, temporary ausente, exception visible
```

### Ejercicio guiado 11 — Sacar dominio del decorator

**Código inicial incorrecto:**

```python
@validate_and_apply_correction
def receive(event, correction):
    return event
```

**Solución conceptual:**

```python
validated = validate_correction(event, correction)
corrected = apply_correction(event, validated)
observed_export(corrected)
```

**Razonamiento:** las reglas se vuelven llamadas explícitas y la instrumentation puede seguir transversal. El criterio es encontrar la transición leyendo el caller.

---

## 32. Ejercicios independientes

1. Implementa `apply_twice` con otra transformation.
2. Escribe una higher-order function que elija entre dos callables.
3. Dibuja bindings de original, wrapper y nombre decorado.
4. Desazucara tres decorators simples.
5. Reproduce un wrapper que pierde retorno y corrígelo.
6. Reproduce un wrapper que oculta `ValueError` y conserva la exception.
7. Reenvía positional, keyword-only y default arguments.
8. Compara metadata con y sin `wraps`.
9. Comprueba `__wrapped__` sin asumir identity.
10. Implementa un decorator parametrizado con label inmutable.
11. Explica cuándo se evalúa la factory.
12. Predice output de dos decorators con before/after.
13. Agrega prints de application y diferencia definition/call.
14. Decora un method y conserva `self`.
15. Detecta estado mutable oculto en closure.
16. Refactoriza un counter closure a object explícito.
17. Diseña observabilidad que no registre args/return.
18. Mide elapsed con `perf_counter` sin fijar duración.
19. Observa una exception y relánzala con `raise`.
20. Compara `raise` con retornar `None`.
21. Justifica si caching de una pure function es decorator apropiado.
22. Refactoriza una pila de side effects a llamadas explícitas.
23. Escribe `try/finally` y conviértelo a manager.
24. Implementa `__enter__` que retorne algo distinto de `self`.
25. Imprime argumentos de `__exit__` en éxito y fallo.
26. Demuestra falsy propagation y truthy suppression.
27. Corrige un manager que suprime `NameError`.
28. Garantiza cleanup con success/failure.
29. Convierte una class simple a `@contextmanager`.
30. Explica el punto donde reaparece exception alrededor de `yield`.
31. Provoca un error antes de `yield` y otro después.
32. Elige class vs generator manager para tres escenarios.
33. Implementa temporary cleanup sin promotion en fallo.
34. Conserva target anterior ante body failure.
35. Rechaza target igual a protected source.
36. Anida tres managers y predice cleanup inverso.
37. Evalúa un uso estrecho de `suppress(FileNotFoundError)`.
38. Reduce un giant manager a lifecycle únicamente.
39. Detecta decorator que muta list/dict argument.
40. Integra pure domain function, observation y derived export.
41. Comprueba que observation no contiene payload.
42. Comprueba metadata del callable integrado.
43. Induce fallo y verifica source/target/temporary.
44. Escribe límites de replace, `fsync` y durability.
45. Defiende cuándo función explícita supera a decorator/manager.

---

## 33. Preguntas conceptuales

1. ¿Qué transformación representa realmente `@decorator`?
2. ¿Qué diferencia hay entre decorator y wrapper?
3. ¿Por qué wrapper debe preservar retorno?
4. ¿Por qué observar exception no autoriza suprimirla?
5. ¿Qué problema resuelve `functools.wraps`?
6. ¿Qué no garantiza `wraps`?
7. ¿Cómo reenvía `*args, **kwargs` y qué información puede ocultar?
8. ¿Qué tres capas tiene un decorator parametrizado?
9. ¿Qué diferencia existe entre capturar configuración y estado mutable?
10. ¿Cuánto vive el estado de closure?
11. ¿Cómo se aplica `@a @b` y cómo se ejecutan sus wrappers?
12. ¿Por qué múltiples decorators pueden ocultar control flow?
13. ¿Cuándo una función explícita es mejor?
14. ¿Por qué `perf_counter` no es timestamp de Event?
15. ¿Qué metadata de observability es segura en el ejemplo?
16. ¿Qué diferencia conceptual existe entre decorator y context manager?
17. ¿Qué fases controla `with`?
18. ¿Qué devuelve `__enter__` y qué recibe `as`?
19. ¿Qué recibe `__exit__` bajo exception?
20. ¿Qué significa retorno truthy de `__exit__`?
21. ¿Por qué suppressing accidental es peligroso?
22. ¿Cómo se relaciona context manager con `try/finally`?
23. ¿Cómo se relaciona `@contextmanager` con decorator y generator?
24. ¿Qué ocurre antes, durante y después de `yield`?
25. ¿Cuándo class manager supera a generator-based?
26. ¿Quién debe ser owner de temporary?
27. ¿Por qué no debes devolver un resource ya cerrado?
28. ¿En qué orden salen managers anidados?
29. ¿Por qué context manager no equivale a durable transaction?
30. ¿Qué diferencia hay entre source, temporary y derived target?
31. ¿Qué problemas EIDOLON son políticas transversales?
32. ¿Qué reglas deben permanecer en dominio?
33. ¿Qué límites tiene atomic replace?
34. ¿Por qué un decorator de observability no debe registrar payload?

---

## 34. Mini challenge — Políticas visibles para exportación derivada

### 34.1 Objetivo

Construye un flujo PF-M1–PF-M7:

```text
synthetic Events
      ↓
pure transformation
      ↓
@observe_operation
      ↓
with AtomicExport(...)
      ↓
derived JSONL export
```

Debe preparar PF-L12 y reforzar PF-L10.

### 34.2 Árbol requerido

```text
eidolon-pf-m7/
├── pyproject.toml
├── src/
│   └── eidolon/
│       ├── __init__.py
│       ├── model.py
│       ├── observation.py
│       ├── export.py
│       └── application.py
└── checks/
    ├── success.py
    ├── decorator_failure.py
    ├── export_failure.py
    ├── metadata.py
    └── source_preserved.py
```

No agregues dependencies runtime.

### 34.3 Pure domain transformation

Define frozen Event pequeño y:

```python
def events_to_records(
    events: tuple[Event, ...],
) -> list[dict[str, object]]:
    ...
```

No muta events, no lee/escribe filesystem y no conoce observation.

### 34.4 Decorator de observability

Implementa:

```python
def observe_operation(operation_id, sink):
    ...
```

Debe:

1. usar `functools.wraps`;
2. aceptar/reenvíar `*args, **kwargs`;
3. preservar retorno;
4. medir con `perf_counter`;
5. registrar operation ID sintético, `func.__name__`, outcome y elapsed;
6. no registrar args, kwargs, return, Event text, path completo ni secrets;
7. observar `Exception` y relanzarla con bare `raise`;
8. no capturar `BaseException`;
9. preservar `__name__`, `__doc__` y `__wrapped__`.

### 34.5 Custom class context manager

Implementa `AtomicExport` con:

```python
with AtomicExport(
    target_path,
    protected_source=source_path,
) as temporary_path:
    ...
```

Contrato:

1. crea temporary en target parent;
2. `__enter__` retorna temporary Path;
3. manager posee cleanup del temporary;
4. body posee file handles que abre;
5. éxito promueve mediante replace;
6. exception no promueve;
7. cleanup ocurre en ambas rutas;
8. `__exit__` retorna falsy;
9. source==target se rechaza antes de crear;
10. no promete durabilidad absoluta.

### 34.6 Variante `@contextmanager`

Implementa una segunda versión pequeña:

```python
@contextmanager
def atomic_export(target, *, protected_source=None):
    ...
```

Documenta setup, yield, promotion y finally cleanup. Usa una de las dos versiones en application y compara tradeoff en cinco líneas.

### 34.7 Application flow

1. crea tuple de Events sintéticos;
2. conserva snapshot/equality del source;
3. ejecuta decorated pure transformation;
4. serializa records a JSONL UTF-8;
5. escribe al temporary dentro de `with`;
6. permite que manager promueva;
7. comprueba target;
8. comprueba source sin cambios.

Output:

```text
PF-M7 policy challenge: PASS
```

### 34.8 Failure checks

`decorator_failure.py`:

- decorated function raises `ValueError`;
- sink registra failure sin payload;
- caller recibe `ValueError`;
- metadata permanece.

`export_failure.py`:

- target contiene `"old"`;
- body escribe partial al temporary y raises;
- caller recibe la exception;
- target sigue `"old"`;
- temporary no existe.

`source_preserved.py`:

- source text/bytes son idénticos;
- target es distinto;
- Events originales no mutan;
- protected source no puede ser target.

### 34.9 Stacking y ownership

Agrega un decorator pequeño `@announce` únicamente en un check. Desazucara:

```text
observed = announce(observe_operation(...)(operation))
```

Predice application/execution order. No agregues una pila al production flow si reduce claridad.

Escribe ownership:

```text
source path      → caller
Event tuple      → caller/domain input
observation sink → caller del decorator factory
temporary Path  → AtomicExport
write handle     → body/nested with
target export    → caller/application
```

### 34.10 Comprobaciones

```bash
python -m compileall -q src checks
python checks/success.py
python checks/decorator_failure.py
python checks/export_failure.py
python checks/metadata.py
python checks/source_preserved.py
```

Todos deben terminar exitosamente capturando solo failures deliberados.

### 34.11 Criterio de aprobación

- pure transformation permanece pura;
- decorator conserva args, return, exceptions y metadata;
- observation no contiene payload;
- duration usa performance clock;
- class manager implementa protocolo correcto;
- generator manager usa setup/yield/finally;
- success promueve y failure limpia;
- source y target tienen roles distintos;
- source permanece idéntico;
- exception suppression accidental no aparece;
- límites de replace/durability se documentan;
- el diseño puede explicarse sin “magia”.

### 34.12 Límites

No uses async, threads, database, ORM, FastAPI, SQLAlchemy, PostgreSQL, Docker, Ollama, LLMs, dependency injection, logging avanzado, `ParamSpec` ni pytest avanzado.

---

## 35. Resumen

- Higher-order function recibe o devuelve callables.
- Wrapper captura original function y configuración mediante closure.
- `@decorator` equivale a reasignar el nombre al resultado del decorator.
- Wrapper público debe preservar retorno y exception contract.
- `*args, **kwargs` reenvía formas distintas, pero typing preciso requiere más trabajo.
- `wraps` preserva metadata seleccionada y `__wrapped__`, no semántica.
- Decorator parametrizado tiene factory, decorator y wrapper.
- Decorators apilados se aplican inside-out y ejecutan outside-in.
- Estado mutable capturado persiste y puede sorprender.
- Methods reciben `self` dentro de args.
- Timing usa `perf_counter`, no timestamp de dominio.
- Observability segura registra metadata, no payload.
- Function explícita gana cuando domain/control flow quedaría oculto.
- Context manager encapsula entrada/salida de una región.
- `__enter__` retorna el valor de `as`.
- `__exit__` recibe información de exception.
- Falsy propaga; truthy suprime.
- Cleanup no implica suppression.
- `@contextmanager` transforma generator de un yield.
- Antes/yield/después corresponden setup/value/teardown.
- Exception del body reaparece en el punto de yield.
- Class sirve para estado/methods; generator manager para lifecycle simple.
- Nested managers limpian en orden inverso.
- Temporary manager posee cleanup; body posee handles que abre.
- Promotion solo en éxito preserva target anterior ante body failure.
- Context manager no crea transaction durable.
- Source, temporary y derived target permanecen separados.

---

## 36. Checklist de dominio

- [ ] Puedo explicar higher-order function con un ejemplo.
- [ ] Puedo trazar original, decorator y wrapper.
- [ ] Puedo desazucarar `@decorator`.
- [ ] Puedo preservar retorno y exception propagation.
- [ ] Puedo detectar retorno perdido y swallowing.
- [ ] Puedo reenviar `*args, **kwargs`.
- [ ] Puedo usar `functools.wraps`.
- [ ] Puedo comprobar `__name__`, `__doc__` y `__wrapped__`.
- [ ] Puedo explicar tres capas de decorator parametrizado.
- [ ] Puedo distinguir configuration closure de mutable state.
- [ ] Puedo predecir order de decorators apilados.
- [ ] Puedo decorar method sin perder `self`.
- [ ] Puedo elegir decorator o function explícita.
- [ ] Puedo medir elapsed con `perf_counter`.
- [ ] Puedo observar success/failure sin payload.
- [ ] Puedo explicar límites de typing del wrapper.
- [ ] Puedo traducir `try/finally` a context manager.
- [ ] Puedo explicar las fases de `with`.
- [ ] Puedo implementar `__enter__` y `__exit__`.
- [ ] Puedo predecir el valor de `as`.
- [ ] Puedo inspeccionar argumentos de exception en `__exit__`.
- [ ] Puedo distinguir falsy propagation y truthy suppression.
- [ ] Puedo garantizar cleanup bajo success/failure.
- [ ] Puedo implementar `@contextmanager` con un yield.
- [ ] Puedo explicar exception alrededor de yield.
- [ ] Puedo elegir class o generator manager.
- [ ] Puedo identificar resource ownership.
- [ ] Puedo predecir cleanup inverso.
- [ ] Puedo usar `suppress` solo bajo contrato estrecho.
- [ ] Puedo detectar giant manager y decorator con mutation.
- [ ] Puedo implementar temporary export y promotion.
- [ ] Puedo conservar target anterior ante body failure.
- [ ] Puedo declarar límites de replace/durability.
- [ ] Puedo separar source, temporary y derived target.
- [ ] Puedo demostrar que Event source no muta.
- [ ] Puedo completar challenge solo con PF-M1–PF-M7.

---

## 37. Preparación para labs y EIDOLON 0.0a

### Lab principal: PF-L12 — Decorator de observabilidad

PF-M7 prepara:

- wrapper transparente con `wraps`;
- metadata y operation ID sintéticos;
- elapsed con `perf_counter`;
- success/failure sin payload;
- exception propagation;
- decisión decorator vs composición explícita.

PF-L12 ampliará checks y la política de datos; logging avanzado pertenece a PF-M9.

### Refuerzo: PF-L10 — Exportación atómica

PF-M6 aportó temporal + replace y límites. PF-M7 añade:

- custom class context manager;
- variant con `@contextmanager`;
- cleanup bajo exception;
- ownership explícito;
- suppression semantics;
- nested lifecycle.

El lab debe inyectar fallos y verificar cleanup; PF-M7 no promete una transaction durable.

### Evidencia antes de avanzar

1. decorator desazucarado;
2. retorno/exception/metadata preservados;
3. pila de dos decorators predicha;
4. observation sin payload;
5. class manager success/failure;
6. truthy suppression reproducida como antipattern;
7. generator manager con cleanup;
8. nested cleanup inverso;
9. source/target/temporary ownership escrito;
10. mini challenge en environment limpio.

Este módulo refuerza **CHECKPOINT PF-C2 — Diseño y lifecycle** y deja políticas pequeñas disponibles para EIDOLON 0.0a sin ocultar dominio.

---

## 38. Recursos de ampliación

La explicación fundamental está contenida aquí. Consulta [PF.11 Recursos recomendados](../../02_curriculum/01_programming_foundations.md#pf11-recursos-recomendados), especialmente Python 3.14 sobre `functools`, data model y `contextlib`.

Usa documentación oficial para verificar metadata copiada por `wraps` o exception semantics de `contextmanager`. No sustituyas el modelo mental por recipes.

---

## 39. Límite del módulo

PF-M7 termina en decorators sync pequeños, metadata, stacking, side effects visibles, context managers sync, ownership, cleanup y exportación temporal derivada.

PF-M8 enseñará async/await, tasks, cancellation y backpressure; PF-M9, pytest, debugging, logging y review avanzados.

No se introducen async decorators/context managers, locks, database transactions, dependency injection frameworks, FastAPI, SQLAlchemy, PostgreSQL, Docker, Ollama, LLMs, descriptors, metaclasses, `ParamSpec`, advanced generics, monkey patching ni aspect-oriented programming formal.
