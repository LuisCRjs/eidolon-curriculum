# CS-M1 — Complejidad, medición y modelos de costo

**Track:** Computer Science Foundations  
**Competencias:** D2.1; soporte D2.3, D3.1  
**Fase:** P0  
**Nivel objetivo:** Aplicado  
**Prerequisites:** PF-M1, PF-M2, PF-M3, PF-M9  
**Build:** EIDOLON 0.0b  
**Curriculum source:** [CS-M1](../../02_curriculum/02_computer_science_foundations.md#cs-m1--complejidad-medición-y-modelos-de-costo)  
**Status:** approved

Una solución correcta puede dejar de ser viable cuando crecen los datos o se repite una operación. También puede ocurrir lo contrario: una estructura con mejor crecimiento asintótico puede perder frente a un scan pequeño por el costo de construirla, asignar memoria y calcular hashes.

CS-M1 enseña a sostener ambas ideas sin contradicción:

```text
algoritmo correcto ≠ algoritmo suficientemente escalable
mejor Big O ≠ mejor programa para cualquier tamaño y contexto
```

El trabajo del módulo sigue esta cadena:

```text
problema
↓
definir n y la operación
↓
predecir tiempo y memoria
↓
formular una hipótesis
↓
medir un workload representativo
↓
interpretar el punto de cruce
↓
decidir y documentar un trigger
```

No se implementan todavía arrays dinámicos, hash maps, binary search, sorting algorithms, heaps ni graph traversals. CS-M2–CS-M6 desarrollarán esas estructuras y algoritmos. Aquí se utilizan operaciones de Python ya conocidas para aprender a razonar sobre costo.

## Resultados de aprendizaje

Al terminar podrás:

- definir el tamaño relevante de una entrada y distinguir dimensiones como `n`, `m`, `q` y `k`;
- identificar operaciones dominantes y derivar costos secuenciales, anidados y condicionales;
- explicar O, Ω y Θ con cotas asintóticas técnicamente correctas;
- comparar O(1), O(log n), O(n), O(n log n), O(n²) y O(nm);
- distinguir best, average y worst case sin inventar una distribución;
- distinguir average case de amortized cost;
- estimar auxiliary space, materialización y time-space tradeoffs;
- reconocer trabajo oculto en built-ins, comprehensions y generators;
- convertir un análisis asintótico en una hipótesis comprobable;
- medir duración con `perf_counter` y `timeit` sin mezclar setup irrelevante;
- comparar memoria Python con `tracemalloc` y explicar los límites de `getsizeof`;
- interpretar una curva y un break-even point sin universalizar resultados locales;
- documentar un Complexity Budget y un migration trigger cuantitativo;
- justificar cuándo conservar un scan y cuándo construir un índice derivado de EIDOLON.

## Cómo estudiar este módulo

1. Antes de ejecutar código, escribe qué significa cada variable de tamaño.
2. Cuenta operaciones como función del tamaño; no cuentes líneas de código.
3. Separa corrección, análisis y medición. Una no sustituye a las otras.
4. Ejecuta benchmarks con el equipo libre de carga innecesaria y conserva la versión de Python.
5. No copies tiempos de este capítulo como expectativas universales: vuelve a medir.
6. Resuelve la práctica inmediata antes de leer los ejercicios guiados.

### Convenciones

- **Ejemplo ejecutable:** bloque autónomo con output estable o asserts.
- **Benchmark ejecutable:** produce cifras variables; se fija el formato y las propiedades, no los tiempos.
- **Derivación:** razonamiento matemático que declara sus tamaños y supuestos.
- **Failure case:** análisis o medición incorrectos de forma deliberada.
- **Fragmento:** requiere contexto y no se ofrece como programa autónomo.

Baseline recomendado: Python 3.14 y standard library. Los tamaños marcados como opcionales dependen de RAM, CPU y carga del equipo.

---

## 1. Por qué necesitamos un modelo de costo

**Fragmento — búsqueda secuencial:**

```python
def contains_id(events, target_id):
    for event in events:
        if event["id"] == target_id:
            return True
    return False
```

**Fragmento — lookup sobre un índice ya construido:**

```python
target_id in events_by_id
```

Ambas expresiones pueden responder correctamente. Decir que la segunda “es más rápida” omite decisiones esenciales:

- ¿ya existía el índice o hay que construirlo?
- ¿habrá una consulta o cien mil?
- ¿cuántos eventos contiene el journal?
- ¿cuánta memoria adicional usa el índice?
- ¿cambian los datos entre consultas?
- ¿qué costo domina fuera de la colección: I/O, parsing o logging?

Un **modelo de costo** selecciona recursos y operaciones relevantes para comparar estrategias. En CS-M1 modelaremos principalmente:

```text
tiempo ≈ cantidad de operaciones dominantes
espacio ≈ memoria adicional que crece con la entrada
```

El modelo es una simplificación deliberada. Ayuda a predecir crecimiento; no simula cada instrucción, allocation o cache miss.

### Aplicación EIDOLON

El journal sigue siendo source of truth. Un `dict` por ID es una proyección reconstruible:

```text
journal append-only ──replay──▶ events_by_id
        source                    derived index
```

Construir el índice cuesta tiempo y memoria. Consultarlo repetidamente puede ahorrar scans. La pregunta correcta no es “¿dict o list?”, sino “¿qué workload y qué evidencia justifican mantener esta proyección?”.

### Define `n`

En `contains_id`, ¿`n` es el número de líneas de Python, el número de eventos o la longitud del ID? Defiende una respuesta para la operación analizada.

### Decide

Si el journal contiene 40 eventos y una command hace una consulta, ¿construirías un índice solo por su lookup esperado O(1)? Enumera la evidencia que te falta.

---

## 2. Definir el tamaño de entrada

### 2.1 `n` no significa “cantidad de líneas”

`n` representa una dimensión relevante de la entrada. Su significado pertenece al análisis y debe escribirse:

| Operación | Tamaño posible |
|---|---|
| buscar Event por ID en una list | `n = número de eventos` |
| comprobar tags prohibidos | `t = número de tags del Event` |
| normalizar un texto | `c = número de code points del texto` |
| ejecutar búsquedas repetidas | `q = número de consultas` |
| reconstruir journal | `r = número de records` |

Una misma función puede analizarse respecto de dimensiones distintas si cambia la pregunta. Comparar IDs de longitud no acotada tiene un costo relacionado con sus caracteres; en un modelo introductorio puede tratarse la comparación de cada ID como unidad si el contrato limita su longitud. El supuesto debe quedar visible.

### 2.2 Una operación puede depender de más de una dimensión

**Ejemplo ejecutable:**

```python
def events_with_any_tag(events, requested_tags):
    matches = []
    for event in events:
        for requested in requested_tags:
            if requested in event["tags"]:
                matches.append(event)
                break
    return matches


events = [
    {"id": "evt-001", "tags": {"home", "arrival"}},
    {"id": "evt-002", "tags": {"work"}},
]

result = events_with_any_tag(events, ["travel", "home"])
print([event["id"] for event in result])
```

Output:

```text
['evt-001']
```

Si `n` es el número de events y `m` el número de requested tags, el par de loops puede hacer hasta `n × m` membership checks. Bajo el supuesto práctico de membership esperado O(1) en cada set de tags, el costo exterior es O(nm). No debe renombrarse automáticamente O(n²): `n` y `m` pueden crecer de forma independiente.

Si los tags de cada Event fueran una list, aparecería otra dimensión: la cantidad de tags internos examinados por membership. El tipo de colección cambia el modelo.

### Define `n`

**Fragmento para analizar:** define las dimensiones para:

```python
def answer_queries(events, target_ids):
    return [contains_id(events, target) for target in target_ids]
```

### Corrige

Un análisis dice: “Hay dos loops, por tanto O(n²)”. Reescríbelo declarando `n` y `m`.

---

## 3. Qué describen O, Ω y Θ

### 3.1 Big O: cota superior asintótica

Decimos que `f(n)` pertenece a O(`g(n)`) si existen constantes positivas `c` y `n₀` tales que, para todo `n ≥ n₀`:

```text
f(n) ≤ c · g(n)
```

Esto significa que, desde cierto tamaño, `g` multiplicada por una constante acota el crecimiento de `f` por arriba. No significa segundos, hardware concreto ni que la cota sea ajustada.

Por ejemplo, `3n + 20` pertenece a O(n). También pertenece a O(n²), pero esta segunda cota es válida y poco informativa. Para describir crecimiento ajustado necesitamos Θ.

### 3.2 Big Omega: cota inferior asintótica

`f(n)` pertenece a Ω(`g(n)`) si existen `c > 0` y `n₀` tales que:

```text
f(n) ≥ c · g(n)    para n ≥ n₀
```

Expresa que `f` no crece asintóticamente más lento que esa cota inferior.

### 3.3 Big Theta: cota ajustada

`f(n)` pertenece a Θ(`g(n)`) cuando está acotada por arriba y por abajo por múltiplos de `g(n)` desde algún tamaño:

```text
c₁ · g(n) ≤ f(n) ≤ c₂ · g(n)
```

Por eso `3n + 20` pertenece a Θ(n).

### 3.4 Uso profesional de “Big O”

En conversaciones técnicas se usa “Big O” coloquialmente para nombrar el orden de crecimiento esperado, incluso cuando se pretende Θ. Conviene entender esa convención y escribir con precisión cuando la distinción afecta el argumento.

Big O tampoco significa automáticamente worst case. La notación acota una función de costo; primero debes decir qué función analizas: best, expected o worst-case cost.

### Deriva

Para `f(n) = 7n + 50`, explica por qué Θ(n) es más informativo que O(n²). No necesitas calcular `n₀` mínimo.

### Explica

¿Por qué “esta función tarda O(n) segundos” mezcla dos conceptos distintos?

---

## 4. Familias de crecimiento

### 4.1 Comparación relativa

La tabla muestra unidades de trabajo relativas, no tiempo real:

| `n` | O(1) | O(log₂ n) aprox. | O(n) | O(n log₂ n) aprox. | O(n²) |
|---:|---:|---:|---:|---:|---:|
| 10 | 1 | 3.3 | 10 | 33 | 100 |
| 100 | 1 | 6.6 | 100 | 664 | 10,000 |
| 1,000 | 1 | 10.0 | 1,000 | 9,966 | 1,000,000 |
| 1,000,000 | 1 | 19.9 | 1,000,000 | 19,931,569 | 1,000,000,000,000 |

Las columnas no comparten una constante real. Una “unidad” de hashing puede costar más que una comparación; por eso la tabla enseña ritmo de crecimiento, no predice cuál gana para `n = 10`.

### 4.2 O(1): costo que no crece con `n`

Acceder a `events[index]` es O(1) bajo el modelo de list de Python: localizar la referencia no requiere recorrer los `n` elementos. O(1) no significa una instrucción, cero allocations ni instantáneo.

El lookup de `dict` se tratará en este módulo como expected average O(1), bajo supuestos razonables de hashing y distribución. Puede degradarse teóricamente; CS-M2 explicará buckets, colisiones y resize.

### 4.3 O(log n): reducir por un factor

Si cada paso reduce aproximadamente a la mitad el espacio pendiente:

```text
n → n/2 → n/4 → ... → 1
```

la cantidad de pasos es aproximadamente `log₂(n)`. Duplicar `n` agrega cerca de un paso. Binary search usa esta idea, pero su implementación y precondición de orden pertenecen a CS-M4.

La base del logaritmo no cambia la clase asintótica: `log₂(n)` y `log₁₀(n)` difieren por un factor constante.

### 4.4 O(n): trabajo proporcional a la entrada

Un scan completo hace una cantidad de comparaciones proporcional al número de elementos. Si duplicas `n`, esperas aproximadamente duplicar el trabajo dominante, no necesariamente el tiempo exacto.

### 4.5 O(n log n): patrón frecuente de sorting

Muchos algoritmos de comparación eficientes tienen crecimiento O(n log n). `sorted(events, key=...)` sirve como contexto profesional; CS-M4 estudiará sorting, estabilidad y la implementación de Python con la profundidad adecuada.

### 4.6 O(n²): pares de elementos

**Ejemplo ejecutable:**

```python
def ordered_pairs(items):
    pairs = []
    for first in items:
        for second in items:
            pairs.append((first, second))
    return pairs


result = ordered_pairs(["a", "b", "c"])
print(len(result))
```

Output:

```text
9
```

Con `n` items, ambos loops recorren la misma entrada: `n × n = n²` pares. La list resultante también ocupa Θ(n²) espacio; no es solo un costo temporal.

### Predice

Si `n` pasa de 1,000 a 2,000, ¿por qué un modelo cuadrático predice cerca de cuatro veces el trabajo dominante y no dos?

### Explica

¿Qué parte de la tabla cambiaría si cada operación O(1) costara mil veces más? ¿Qué parte no cambiaría?

---

## 5. Derivar costos de código

### 5.1 Operaciones secuenciales: sumar

**Ejemplo ejecutable:**

```python
def active_ids_and_tags(events):
    active_ids = []
    for event in events:
        if event["active"]:
            active_ids.append(event["id"])

    tags = set()
    for event in events:
        tags.update(event["tags"])

    return active_ids, tags


events = [
    {"id": "evt-001", "active": True, "tags": ["home"]},
    {"id": "evt-002", "active": False, "tags": ["work"]},
]

active_ids, tags = active_ids_and_tags(events)
print(active_ids)
print(sorted(tags))
```

Output:

```text
['evt-001']
['home', 'work']
```

Si cada Event tiene un número acotado de tags, hay dos recorridos lineales:

```text
an + bn + c
= (a + b)n + c
∈ Θ(n)
```

O(n) + O(n) se simplifica a O(n), no a O(n²), porque los trabajos ocurren uno después del otro.

Si después aparece una etapa Θ(n²):

```text
Θ(n) + Θ(n²) = Θ(n²)
```

El término cuadrático domina asintóticamente. Esto no borra el costo lineal para datos pequeños; solo describe crecimiento.

### 5.2 Trabajo anidado: multiplicar

Cuando por cada uno de `n` elementos se realizan hasta `m` operaciones:

```text
n × m = nm
```

Si `m = n`, obtenemos n². Si `m` está limitado a 5 por contrato, `5n` pertenece a Θ(n).

### 5.3 Loop triangular y sumatoria sencilla

**Ejemplo ejecutable:**

```python
def unordered_index_pairs(items):
    pairs = []
    for left in range(len(items)):
        for right in range(left + 1, len(items)):
            pairs.append((left, right))
    return pairs


print(len(unordered_index_pairs(["a", "b", "c", "d"])))
```

Output:

```text
6
```

El inner loop hace `n-1`, luego `n-2`, hasta 0 iteraciones:

```text
(n - 1) + (n - 2) + ... + 1 + 0
= n(n - 1) / 2
∈ Θ(n²)
```

Hacer aproximadamente la mitad de `n²` sigue siendo crecimiento cuadrático; el factor `1/2` es constante.

### 5.4 Condicionales

**Fragmento para analizar:**

```python
if use_index:
    build_index(events)
else:
    scan(events)
```

no se suman automáticamente ambos costos porque solo una branch se ejecuta. La respuesta depende de la pregunta:

- worst case: máximo costo entre ramas para el input permitido;
- una precondición conocida: costo de la rama que realmente puede ejecutarse;
- expected behavior: requiere supuestos sobre frecuencia/distribución de ramas.

Sin distribución, “average de ambas ramas” no tiene fundamento.

### Deriva

**Fragmento para analizar:**

```python
for event in events:
    validate(event)

for query in queries:
    for event in events:
        compare(query, event)
```

Usa `n = events` y `q = queries`.

### Corrige

Un reviewer afirma que el loop triangular es O(n) porque “cada índice se visita una vez como `left`”. Señala el trabajo omitido.

---

## 6. Best, average y worst case

### 6.1 Linear search

**Ejemplo ejecutable:**

```python
def find_event(events, event_id):
    for event in events:
        if event["id"] == event_id:
            return event
    return None


events = [
    {"id": "evt-001"},
    {"id": "evt-002"},
    {"id": "evt-003"},
]

assert find_event(events, "evt-001") == {"id": "evt-001"}
assert find_event(events, "missing") is None
print("linear search: PASS")
```

Output:

```text
linear search: PASS
```

Con `n = número de events` y comparaciones de ID tratadas como unidad:

- best case: target en primera posición, Θ(1);
- worst case: target al final o ausente, Θ(n);
- average case: no se puede afirmar sin un modelo de entradas.

Si asumimos que una búsqueda exitosa tiene la misma probabilidad de apuntar a cada posición, las comparaciones esperadas son:

```text
(1 + 2 + ... + n) / n
= n(n + 1) / (2n)
= (n + 1) / 2
∈ Θ(n)
```

Ese resultado depende del supuesto uniforme. Si 99% de las consultas busca el primer elemento, el número esperado cambia, aunque el worst case siga lineal.

### 6.2 Early return no cambia el worst case

`return` evita trabajo después de encontrar el target. Mejora casos concretos y constantes; no convierte el worst-case scan en O(1), porque existe una entrada válida que exige revisar los `n` elementos.

### Predice

¿Qué caso representa buscar un ID ausente? ¿Qué cambia si la función sabe por contrato que siempre existe?

### Explica

Escribe el supuesto necesario antes de decir “la búsqueda promedio revisa la mitad”.

---

## 7. Constantes, términos dominantes y límites de Big O

### 7.1 Ignorar constantes ayuda a comparar escalamiento

Compara:

```text
f(n) = 1000n
g(n) = n²
```

Asintóticamente, `f ∈ Θ(n)` y `g ∈ Θ(n²)`. Para `n < 1000`, `n²` puede ser menor que `1000n`; después aparece un punto de cruce. Big O identifica qué crecimiento terminará dominando, no dónde ocurre el cruce real.

### 7.2 Un mejor orden puede perder para `n` pequeño

Un scan lineal sobre diez referencias contiguas puede ser más rápido que construir un dict y calcular hashes. Esto no refuta el análisis: compara costos completos y constantes para un tamaño concreto.

Factores omitidos por el modelo asintótico pueden incluir:

- costo de construir la estructura;
- hashing y equality;
- allocations y object overhead;
- cache locality;
- interpreter overhead;
- I/O, parsing, logging y contention;
- distribución real de consultas.

Cache locality se menciona aquí solo como ejemplo: acceso cercano suele aprovechar mejor la jerarquía de memoria. CS-M10 lo desarrollará.

### Failure case — O(1) significa instantáneo

Una función que espera diez segundos y devuelve el primer elemento sigue siendo O(1) respecto de `n`, pero es lenta. La notación describe crecimiento en la dimensión elegida, no experiencia del usuario.

### Explica

¿Por qué medir un índice sobre `n = 20` no invalida que su lookup tenga mejor crecimiento esperado?

---

## 8. Complejidad espacial y time-space tradeoff

### 8.1 Input, output y auxiliary space

Conviene declarar qué memoria se cuenta:

- **input space:** objetos que la función recibe y ya existen;
- **output space:** resultado que el contrato exige conservar;
- **auxiliary space:** memoria adicional usada para ejecutar la estrategia.

Un scan puede usar O(1) auxiliary space si conserva solo una referencia actual. Construir `events_by_id` usa O(n) espacio adicional para el índice, aunque los Event ya existan.

### 8.2 Materialización

**Ejemplo ejecutable:**

```python
def active_ids_list(events):
    return [event["id"] for event in events if event["active"]]


def active_ids_iter(events):
    return (
        event["id"]
        for event in events
        if event["active"]
    )


events = [
    {"id": "evt-001", "active": True},
    {"id": "evt-002", "active": False},
]

eager = active_ids_list(events)
lazy = active_ids_iter(events)

print(eager)
print(list(lazy))
```

Output:

```text
['evt-001']
['evt-001']
```

Ambas estrategias recorren hasta `n` Events: tiempo Θ(n) para consumir el resultado completo. La generator expression evita materializar inmediatamente la list de IDs y puede mantener auxiliary space O(1) si el consumidor procesa uno por uno.

No elimina el trabajo, puede retrasar errores y normalmente permite una sola pasada. Si el consumidor llama `list(...)`, la materialización reaparece.

### 8.3 Índice derivado

```text
scan
time por consulta: Θ(n) worst case
auxiliary space: Θ(1)

índice
build time: Θ(n) expected
lookup: O(1) expected average
auxiliary space: Θ(n)
```

El índice intercambia memoria y build/maintenance work por consultas esperadas más baratas. Sigue siendo derived data reconstruible por replay.

### Space

¿Qué memoria nueva crea `events[:]`? Si el slice tiene `k` elementos, razona sobre tiempo y espacio en función de `k`, no de las líneas de código.

### Decide

¿Aceptarías duplicar memoria para acelerar una command ejecutada una vez al día? Define qué workload y límite de RAM necesitas conocer.

---

## 9. Costo amortizado

### 9.1 El problema

Una list dinámica reserva capacidad. Algunas llamadas a `append` caben en la capacidad actual; ocasionalmente se necesita una expansión más costosa. No todas las llamadas cuestan exactamente lo mismo.

Si una secuencia de `n` appends tiene costo total O(n), el costo amortizado por append es:

```text
O(n) / n = O(1) amortized
```

La intuición es repartir expansiones ocasionales entre muchas operaciones baratas. No depende de afirmar el factor exacto de crecimiento de CPython, que puede cambiar entre versiones y pertenece a CS-M2.

### 9.2 Amortized no es average case probabilístico

- **average case:** promedia sobre una distribución de inputs u operaciones; debe declararse.
- **amortized analysis:** acota el costo promedio por operación sobre una secuencia, incluso sin probabilidad.

Un adversario no “tiene mala suerte” para causar resize: la secuencia incluye operaciones caras, pero el costo total se distribuye.

### Predice

¿Un append individual puede costar más que O(1) aunque se describa append como O(1) amortized?

### Explica

¿Qué información probabilística necesitas para una afirmación amortizada? La respuesta debe distinguirla de average case.

---

## 10. Costos de operaciones Python conocidas

Esta tabla es un punto de razonamiento, no un inventario para memorizar. Los supuestos se profundizan en CS-M2 y CS-M4.

| Operación | Costo práctico | Condición relevante |
|---|---|---|
| `items[i]` | O(1) | índice válido en list |
| `items.append(x)` | O(1) amortized | expansión ocasional más cara |
| `target in items` | O(n) worst case | list; early exit posible |
| `mapping[key]` | O(1) expected average | hashing/equality razonables |
| `target in seen` | O(1) expected average | set; mismos supuestos hash |
| `items[a:b]` | O(k) tiempo y espacio | crea list con `k` referencias |
| `items.copy()` | O(n) tiempo y espacio | copia superficial |
| `sorted(items)` | O(n log n) worst case | crea nueva list; sorting formal en CS-M4 |

### 10.1 Trabajo oculto

**Fragmento — trabajo oculto en operaciones conocidas:**

```python
if target in list_of_ids:
    ...

ordered = sorted(events, key=lambda event: event["recorded_at"])
```

Contar líneas clasificaría ambas como una operación. El modelo inspecciona el contrato del operador o built-in.

### 10.2 Comprehensions

**Fragmento:**

```python
[transform(item) for item in items if predicate(item)]
```

La syntax compacta sigue iterando. Si `predicate` y `transform` son O(1), el recorrido es O(n), más O(k) output para `k` resultados. Si dentro existe membership lineal sobre otra collection, el costo cambia.

### 10.3 Hidden quadratic work

**Ejemplo ejecutable:**

```python
def duplicate_ids_slow(ids):
    return {
        event_id
        for event_id in ids
        if ids.count(event_id) > 1
    }


print(sorted(duplicate_ids_slow(["a", "b", "a"])))
```

Output:

```text
['a']
```

La comprehension recorre `n` IDs. Cada `ids.count` vuelve a escanear la list: hasta `n × n` comparaciones, Θ(n²). El hecho de usar set comprehension no elimina el trabajo interno.

### Deriva

Analiza `[x for x in items if x in allowed]` en dos casos: `allowed` es list de tamaño `m` y `allowed` es set ya construido.

---

## 11. Caso EIDOLON: scan e índice derivado

### 11.1 Estrategia A — scan

**Fragmento:**

```python
def find_event(events, event_id):
    for event in events:
        if event["id"] == event_id:
            return event
    return None
```

Para una consulta:

- best case Θ(1);
- worst case Θ(n);
- space adicional Θ(1).

Para `q` consultas independientes, el worst-case total es Θ(qn).

### 11.2 Estrategia B — índice

**Ejemplo ejecutable:**

```python
def build_event_index(events):
    events_by_id = {}
    for event in events:
        event_id = event["id"]
        if event_id in events_by_id:
            raise ValueError("duplicate event id")
        events_by_id[event_id] = event
    return events_by_id


events = [
    {"id": "evt-001", "text": "Llegué"},
    {"id": "evt-002", "text": "Salí"},
]

events_by_id = build_event_index(events)
print(events_by_id["evt-002"]["text"])
```

Output:

```text
Salí
```

Bajo el modelo esperado de dict:

```text
build: Θ(n) expected
q lookups: O(q) expected
total: Θ(n + q) expected
auxiliary space: Θ(n)
```

El peor caso teórico del hash lookup puede degradarse; no se utiliza aquí para diseñar internals. También existen costos de validar duplicates y reconstruir cuando cambia el journal.

### 11.3 Break-even intuition

Incluye constantes desconocidas:

```text
scan total      ≈ q · a · n
index total     ≈ b · n + q · c
```

- `a`: costo efectivo por elemento escaneado;
- `b`: costo efectivo de indexar un elemento;
- `c`: costo efectivo de un lookup;
- `q`: número de consultas sobre el mismo snapshot de `n` eventos.

El índice compensa en tiempo cuando:

```text
b·n + q·c < q·a·n
```

Si `a·n > c`, puede despejarse una orientación:

```text
q > b·n / (a·n - c)
```

No es una fórmula universal: las constantes y el workload se miden. Si el índice se reconstruye antes de cada consulta, su costo cambia a Θ(qn) y puede perder su ventaja.

### 11.4 Cambios y mantenimiento

Un índice puede:

- construirse una vez por replay;
- actualizarse con cada append bajo invariantes explícitas;
- invalidarse y reconstruirse;
- divergir si se trata como una segunda autoridad.

CS-M1 solo compara build + queries sobre un snapshot. CS-M2 profundizará el diseño del índice.

### Hipótesis

Formula una hipótesis distinta para `q = 1` y `q = 1,000`. Incluye tiempo de construcción y memoria.

### Decide

¿Qué observación demostraría que el índice no compensa aunque sus lookups aislados sean más rápidos?

---

## 12. Sorting, top-k y graph cost como contexto

### 12.1 Timeline: ordenar una vez o en cada consulta

Si `n` Events se ordenan una vez:

```text
build ordered view: O(n log n)
q recorridos posteriores: O(qn), si cada uno consume toda la timeline
```

Si cada consulta vuelve a ordenar:

```text
O(q · n log n)
```

La key de orden y los tie-breakers deterministas son parte del contrato, pero sorting formal pertenece a CS-M4.

### 12.2 Top-k

Seleccionar los `k` mejores candidatos podría resolverse ordenando todos: O(n log n). Para `2 <= k <= n`, una estructura futura de tamaño `k` puede orientar un costo O(n log k); `k = 1` conserva un scan O(n), no “cero trabajo” porque `log(1) = 0`. No se implementa todavía: heaps pertenecen a CS-M5.

El punto pedagógico es formular dimensiones:

```text
n = candidatos
k = resultados requeridos
```

Si `k` está cerca de `n`, las constantes y el contrato pueden favorecer sorting. Si `k` es pequeño y `n` grande, medir una estructura especializada tiene sentido.

### 12.3 Graphs

Un recorrido futuro puede depender de:

```text
V = vertices
E = edges
```

Por eso se expresa O(V + E), no O(n) sin significado. BFS/DFS, dirección y visited sets pertenecen a CS-M6.

### Define `n`

Para un top-10 entre un millón de candidatos, define `n` y `k`. ¿Qué dimensión permanece fija en ese workload?

---

## 13. Del análisis a una hipótesis de benchmark

Big O produce una predicción de crecimiento. Un benchmark responde una pregunta empírica más limitada:

> ¿Qué ocurre con estos datos, esta implementación, esta versión de Python, este hardware y esta carga?

Un benchmark no prueba complejidad matemática. Una curva compatible con O(n) en cinco tamaños puede ocultar rangos o efectos no observados. Aun así, puede refutar una expectativa práctica o encontrar el break-even relevante.

### 13.1 Pregunta antes que números

Pregunta útil:

```text
¿A partir de qué combinación de n y q el tiempo de build + q lookups
es menor que q scans, y cuánta memoria adicional requiere?
```

Hipótesis útil:

```text
Para q = 1 y n pequeño, el scan puede ganar.
Al crecer q sobre el mismo snapshot, el índice debe recuperar su build cost.
La memoria adicional del índice crecerá aproximadamente con n.
```

“Ejecutaré ambos a ver qué sale” carece de variable, operación y criterio de decisión.

### Benchmark

¿Qué debes mantener igual para atribuir una diferencia a scan vs index?

---

## 14. Medición temporal con `perf_counter`

`time.perf_counter()` ofrece un reloj de alta resolución apropiado para duraciones. Se resta inicio de fin; su valor absoluto no tiene significado contractual.

### 14.1 Medir una operación suficientemente grande

**Benchmark ejecutable:**

```python
from time import perf_counter


def find_event(events, event_id):
    for event in events:
        if event["id"] == event_id:
            return event
    return None


events = [{"id": f"evt-{index:06d}"} for index in range(10_000)]
targets = ["evt-009999"] * 100

started = perf_counter()
for target in targets:
    assert find_event(events, target) is not None
elapsed = perf_counter() - started

assert elapsed >= 0.0
print("queries=100")
print("elapsed_non_negative=True")
```

Output estable:

```text
queries=100
elapsed_non_negative=True
```

El tiempo numérico se omite porque depende del entorno. Generar el dataset ocurre fuera de la región medida. Agrupar 100 consultas evita depender de una única operación demasiado corta.

### 14.2 Repetir

Una ejecución puede coincidir con scheduling, background work, allocation o caches. Repite el experimento y resume valores; no elijas solo el mejor sin explicar por qué.

### Modifica

Mueve la creación de `events` dentro de la región medida. Explica qué pregunta distinta respondería ese benchmark.

---

## 15. Microbenchmarks con `timeit`

`timeit` repite una callable y reduce parte de la ceremonia. Sigue siendo responsabilidad del autor separar setup y elegir un workload válido.

### 15.1 `repeat`, `number` y mediana

**Benchmark ejecutable:**

```python
from statistics import median
from timeit import repeat


values = list(range(1_000))
target = 999

totals = repeat(
    lambda: target in values,
    repeat=5,
    number=1_000,
)
seconds_per_operation = [total / 1_000 for total in totals]

assert len(seconds_per_operation) == 5
assert median(seconds_per_operation) >= 0.0
print("repetitions=5")
print("samples_non_negative=True")
```

Output estable:

```text
repetitions=5
samples_non_negative=True
```

`repeat` devuelve el tiempo total de cada repetición. Dividir por `number` estima tiempo por operación dentro de ese microbenchmark. La mediana reduce la influencia de algún outlier; no sustituye estadística formal.

Por ejemplo, en `1.0, 1.1, 1.2, 1.3, 20.0`, la media queda arrastrada por la muestra extrema, mientras la mediana sigue siendo `1.2`. Esto no autoriza borrar `20.0`: conserva las muestras e investiga si representa ruido o un comportamiento real del workload.

`timeit` desactiva temporalmente el garbage collector durante cada medición. Esto hace más comparables muchos microbenchmarks, pero también excluye un costo que puede pertenecer a un workload real con muchas allocations. Si esa diferencia afecta la decisión, formula y mide por separado una prueba que incluya el comportamiento de garbage collection; no cambies el setup sin documentarlo.

### 15.2 Warmup prudente

Primeras ejecuciones pueden incluir imports, allocations, carga de datos o caches. Separa setup cuando no pertenece a la operación y realiza una llamada previa si el contrato requiere inicializar estado.

No atribuyas la diferencia a un JIT de JVM: el runtime objetivo es Python/CPython 3.14 y este módulo no presupone un JIT. “Warmup” aquí significa controlar setup y efectos de primeras ejecuciones observables, no inventar internals.

### 15.3 Microbenchmark no es workload completo

Medir `mapping[key]` aislado no incluye construir el mapping, leer JSONL ni mantenerlo. Sirve para una pregunta pequeña; no autoriza por sí solo cambiar arquitectura.

### Interpreta

Si el lookup aislado gana por 100×, pero build + una consulta pierde, ¿qué costo omitió el microbenchmark?

---

## 16. Diseñar un benchmark representativo

### 16.1 Variables controladas

Un benchmark comparable usa:

- mismos Events e IDs objetivo;
- misma semántica de éxito/ausencia;
- tamaños crecientes;
- distribución declarada de targets;
- misma versión de Python y entorno;
- múltiples repeticiones;
- dataset generation fuera de la región si no se mide ingestion;
- I/O y logging fuera si se quiere aislar CPU/collection work.

### 16.2 Tamaños

Una secuencia útil puede ser:

```text
10², 10³, 10⁴, 10⁵ y 10⁶ si el equipo lo permite
```

No fuerces 10⁶ si causa swapping, agota RAM o vuelve impráctica la sesión. Documenta el máximo ejecutado. Un dataset enorme artificial tampoco es mejor si no representa el producto.

### 16.3 Distribución de consultas

Separar al menos:

- target al inicio;
- target al final;
- target ausente;
- targets repartidos de manera determinista;
- consultas repetidas sobre el mismo snapshot.

Cambiar distribución puede cambiar resultados del scan sin cambiar su worst case.

### 16.4 Tabla antes que gráfica

```text
n | q | scan_total | index_build | indexed_queries | extra_peak_bytes
```

Una tabla permite detectar tendencia y punto de cruce sin añadir matplotlib, pandas o NumPy.

### Hipótesis

¿Qué forma esperas para `scan_total` si mantienes `q` fijo? ¿Y si `q` crece junto con `n`?

---

## 17. Medir memoria con prudencia

### 17.1 `sys.getsizeof` es superficial

**Ejemplo ejecutable:**

```python
from sys import getsizeof


events = [{"id": "evt-001"}, {"id": "evt-002"}]

outer_size = getsizeof(events)
first_event_size = getsizeof(events[0])

assert outer_size > 0
assert first_event_size > 0
print("shallow_sizes_positive=True")
```

Output estable:

```text
shallow_sizes_positive=True
```

`getsizeof(events)` informa el tamaño superficial de la list según esa implementación. No recorre dicts, strings ni objetos compartidos. Sumar recursivamente también exige decidir cómo evitar contar dos veces objetos compartidos. No uses ese número como memoria total universal.

### 17.2 `tracemalloc`: allocations Python observadas

**Benchmark ejecutable:**

```python
import tracemalloc


def build_event_index(events):
    return {event["id"]: event for event in events}


events = [{"id": f"evt-{index:05d}"} for index in range(5_000)]

tracemalloc.start()
before_current, _ = tracemalloc.get_traced_memory()
events_by_id = build_event_index(events)
after_current, peak = tracemalloc.get_traced_memory()
tracemalloc.stop()

retained_delta = after_current - before_current
peak_delta = peak - before_current

assert len(events_by_id) == len(events)
assert peak_delta >= retained_delta
print("index_complete=True")
print("peak_covers_retained=True")
```

Output estable:

```text
index_complete=True
peak_covers_retained=True
```

El input se crea antes de iniciar tracing para orientar la comparación hacia allocations del índice. `tracemalloc` rastrea memoria asignada por Python dentro de su alcance; no equivale automáticamente a RSS total, memoria del kernel o todos los buffers nativos. `peak_delta` y `retained_delta` responden preguntas distintas.

Para comparar estrategias, ejecuta cada una en una medición aislada y conserva el resultado mientras lees `current` si deseas observar memoria retenida.

### Space

¿Por qué medir ambos índices en la misma sesión de tracing puede contaminar la comparación?

---

## 18. Tiempo real, I/O y locality

### 18.1 I/O puede dominar

Si cada benchmark vuelve a leer y parsear un archivo, la latencia de filesystem/JSON puede ocultar la diferencia entre list membership y dict lookup. Eso puede ser correcto si la pregunta es end-to-end, pero incorrecto si se pretende aislar la estructura.

Declara la frontera:

```text
microbenchmark: events ya están en memoria
integration benchmark: incluye load/replay
```

Ambos aportan evidencia distinta.

### 18.2 Cache locality

Una list recorre referencias de manera secuencial; un hash lookup sigue otra ruta de memoria. La jerarquía de memoria, object layout y caches afectan constantes. CS-M10 explicará locality y CPU/cache/I/O; CS-M1 solo reconoce que Big O no captura todo el tiempo real.

### 18.3 Hardware y runtime

**Ejemplo ejecutable — registra como mínimo:**

```python
import platform
import sys

print(sys.version.split()[0])
print(platform.python_implementation())
```

No fijes esos outputs en una solución universal. Ayudan a reproducir, no explican por sí solos una diferencia.

### Decide

¿Qué benchmark usarías para decidir la experiencia de una command: micro o end-to-end? ¿Cuál usarías para atribuir el bottleneck a lookup?

---

## 19. Benchmarks engañosos

### 19.1 Medir setup en vez de la operación

**Código incorrecto — mezcla generación, printing y lookup:**

```python
started = perf_counter()
events = make_events(n)
print(events)
result = find_event(events, target)
elapsed = perf_counter() - started
```

El resultado no aísla lookup. Puede ser un benchmark end-to-end solo si generación y printing pertenecen al workload real; aun así, imprimir todo suele distorsionar la operación de producto.

### 19.2 Dead code o resultado no comprobado

Python normalmente ejecutará la llamada aunque ignores el resultado, pero un benchmark sin assert puede estar midiendo una operación incorrecta, una excepción ocultada o un target que nunca existe. Verifica semántica fuera o dentro de la medición con costo conocido.

### 19.3 Dataset demasiado pequeño

Con `n = 3`, ruido y constantes pueden ocultar la tendencia. Eso no vuelve inválido el caso pequeño; lo vuelve insuficiente para extrapolar escalamiento.

### 19.4 Dataset enorme irrelevante

Un millón de Events puede causar un comportamiento que EIDOLON P0 nunca enfrentará. Usa rangos que incluyan el workload esperado y algunos tamaños de stress justificados.

### 19.5 Una sola ejecución

Una muestra no muestra variabilidad. Repite, registra todas las muestras o un resumen sencillo y no elimines outliers sin causa.

### 19.6 Cambiar varias variables

Comparar algoritmo A con IDs cortos y `n=1_000` contra algoritmo B con IDs largos y `n=100_000` no permite atribuir la diferencia a la estrategia.

### 19.7 Benchmark sin hipótesis

Una tabla sin pregunta puede acumular números y no apoyar ninguna decisión. Escribe primero qué patrón o cruce esperas.

### Corrige

Diseña una región medida que compare scan e indexed lookup sobre exactamente los mismos targets, dejando dataset/build fuera cuando la pregunta sea lookup aislado.

---

## 20. Benchmarking, profiling y optimización

### 20.1 Diferencia

```text
benchmark
→ compara tiempo/memoria de una operación o estrategia definida

profiling
→ localiza dónde se consume tiempo o recursos dentro de una ejecución
```

`cProfile` existe como herramienta `[NICE]`, pero profiling detallado no pertenece a CS-M1. Primero formula una pregunta; después elige herramienta.

### 20.2 Optimización prematura con rigor

“No optimices prematuramente” no significa ignorar un nested scan obvio sobre datos que crecerán. Tampoco justifica reemplazar código claro por una estructura compleja sin riesgo observable.

```text
solución correcta y clara
↓
modelo de costo
↓
riesgo cuantificado
↓
benchmark o profile apropiado
↓
bottleneck confirmado
↓
cambio mínimo
↓
regression/performance evidence
```

La complejidad de mantenimiento también cuesta. Una optimización puede aumentar estados, invalidation paths y surface de bugs.

### Failure case — optimizar antes de medir

Un equipo introduce una cache persistente y synchronization para 200 Events sin medir scans. El cambio agrega otra fuente potencial de divergencia. La corrección no es “nunca usar cache”: es definir workload, umbral y rebuild/invalidation contract.

### Decide

¿Qué evidencia mínima pedirías antes de reemplazar el índice in-memory por una tecnología externa? No elijas la tecnología.

---

## 21. Migration triggers y Complexity Budget

### 21.1 Trigger cuantitativo

Un trigger contiene:

- workload: operación y distribución;
- tamaño: `n`, `q` y otras dimensiones;
- métrica: latencia, peak memory, throughput o error budget;
- percentil/resumen cuando aplique;
- environment objetivo;
- umbral;
- ventana o frecuencia de medición;
- acción de reevaluación, no tecnología predeterminada.

Ejemplo de forma, sin valor universal:

> Mantendremos scan lineal mientras el 95% de las consultas por ID sobre 100k Events termine bajo `X ms` en el hardware objetivo y el replay permanezca bajo `Y MiB`. Si cualquiera falla en tres mediciones comparables, evaluaremos un índice o estrategia distinta.

`X` y `Y` deben provenir del producto y la evidencia, no de este texto.

### 21.2 Complexity Budget

| Operación | `n` / workload | Estrategia | Tiempo estimado | Espacio auxiliar | Medición | Trigger |
|---|---|---|---|---|---|---|
| `show event_id` | `n` Events, `q` queries | scan | worst Θ(qn) | Θ(1) | por medir | latencia p95 > X |
| rebuild | `n` records | dict derivado | expected Θ(n) | Θ(n) | por medir | peak > Y MiB |
| timeline export | `n` Events | `sorted` | O(n log n) | O(n) output | por medir | export > Z s |

Este artefacto hace visibles supuestos y evita adoptar tecnología por moda. Se actualizará para EIDOLON 0.0b conforme CS-M2–CS-M10 introduzcan estructuras y sistemas.

### Modifica

Completa una fila para `q = 1` y otra para `q = 10_000`. ¿Por qué el mismo `n` puede producir decisiones distintas?

---

## 22. Caso progresivo integrado: medir scan frente a índice

El siguiente programa separa dataset, build, queries y memoria. No fija un ganador universal.

### 22.1 Implementación

**Benchmark ejecutable:**

```python
from statistics import median
from timeit import repeat
import tracemalloc


def make_events(n):
    return [
        {
            "id": f"evt-{index:07d}",
            "person_id": f"person-{index % 100:03d}",
            "tags": ("synthetic", f"group-{index % 10}"),
        }
        for index in range(n)
    ]


def find_event(events, event_id):
    for event in events:
        if event["id"] == event_id:
            return event
    return None


def build_index(events):
    result = {}
    for event in events:
        if event["id"] in result:
            raise ValueError("duplicate event id")
        result[event["id"]] = event
    return result


def query_by_scan(events, targets):
    return [find_event(events, target) for target in targets]


def query_by_index(events_by_id, targets):
    return [events_by_id.get(target) for target in targets]


def median_seconds(action, repeats=5):
    samples = repeat(action, repeat=repeats, number=1)
    return median(samples)


def index_peak_bytes(events):
    tracemalloc.start()
    before, _ = tracemalloc.get_traced_memory()
    events_by_id = build_index(events)
    after, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    assert len(events_by_id) == len(events)
    return after - before, peak - before


def benchmark_case(n, q):
    events = make_events(n)
    targets = [f"evt-{((index * 7_919) % n):07d}" for index in range(q)]

    expected = query_by_scan(events, targets)
    events_by_id = build_index(events)
    assert query_by_index(events_by_id, targets) == expected

    scan_seconds = median_seconds(
        lambda: query_by_scan(events, targets)
    )
    build_seconds = median_seconds(
        lambda: build_index(events)
    )
    lookup_seconds = median_seconds(
        lambda: query_by_index(events_by_id, targets)
    )
    retained_bytes, peak_bytes = index_peak_bytes(events)

    return {
        "n": n,
        "q": q,
        "scan_seconds": scan_seconds,
        "index_build_seconds": build_seconds,
        "indexed_queries_seconds": lookup_seconds,
        "index_retained_bytes": retained_bytes,
        "index_peak_bytes": peak_bytes,
    }


row = benchmark_case(n=1_000, q=100)

assert row["n"] == 1_000
assert row["q"] == 100
assert row["scan_seconds"] >= 0.0
assert row["index_build_seconds"] >= 0.0
assert row["indexed_queries_seconds"] >= 0.0
assert row["index_peak_bytes"] >= row["index_retained_bytes"]

print("columns=7")
print("semantic_results_equal=True")
print("memory_relation_valid=True")
```

Output estable:

```text
columns=7
semantic_results_equal=True
memory_relation_valid=True
```

Los tiempos y bytes se conservan en `row`, pero no se imprimen como output universal. El benchmark:

- crea Events y targets fuera de las regiones temporales;
- valida igualdad semántica antes de comparar;
- mide build separado de indexed queries;
- incluye memoria adicional aproximada del índice;
- usa IDs sintéticos y distribución determinista;
- no incluye I/O ni logging.

### 22.2 Interpretación correcta

Para comparar una operación completa:

```text
scan_total = scan_seconds
indexed_total = index_build_seconds + indexed_queries_seconds
```

Si el índice ya existe por otra necesidad, la pregunta puede excluir build. Si debe reconstruirse para cada command, debe incluirlo. El contexto decide la frontera.

Para encontrar break-even observado, ejecuta varios `q` con el mismo `n` y busca cuándo `indexed_total` queda consistentemente debajo de `scan_total`. No elijas un cruce por una sola muestra ruidosa.

### 22.3 Curva escalonada

**Fragmento — ejecútalo con tamaños adecuados a tu equipo:**

```python
rows = []
for n in (100, 1_000, 10_000, 100_000):
    rows.append(benchmark_case(n=n, q=100))
```

Para `10⁶`, ejecútalo solo si RAM/tiempo lo permiten. Registra si omitiste el tamaño y por qué.

### Interpreta

Si lookup aislado permanece casi plano pero `index_peak_bytes` crece, ¿qué tradeoff confirma?

### Decide

¿Usarías `indexed_queries_seconds` o `index_build_seconds + indexed_queries_seconds` para una command que crea y descarta el índice? Justifica la frontera.

---

## 23. Catálogo de failure cases

### 23.1 Big O sin definir `n`

> “La función es O(n).”

Incompleto: ¿`n` es Events, queries, tags o caracteres? Corrección: declarar tamaño, operación, caso y supuestos.

### 23.2 Contar líneas

> “`sorted(items)` es O(1) porque ocupa una línea.”

Síntoma: el benchmark crece con `n`. Causa: la llamada oculta sorting y materialización. Corrección: analizar el contrato de la operación.

### 23.3 Nested loops siempre O(n²)

Un outer loop de `n` y un inner loop de `m` es O(nm); si el inner recorre cinco estados fijos, es O(n). Corrección: declarar dimensiones y límites.

### 23.4 O(1) igual a instantáneo

Una operación constante puede tener una constante grande, I/O o hashing costoso. Corrección: separar crecimiento de tiempo observado.

### 23.5 Big O ignora memoria

Un índice puede reducir consultas y agotar el budget de RAM. Corrección: documentar time y auxiliary space.

### 23.6 Average sin distribución

Promediar best y worst no define average case. Corrección: declarar modelo de posiciones, éxitos/ausencias y frecuencia.

### 23.7 Amortized como “probablemente rápido”

Amortización no expresa probabilidad. Corrección: razonar sobre costo total de una secuencia.

### 23.8 Generator como ahorro de tiempo automático

Procesar todos los elementos sigue siendo Θ(n). Corrección: distinguir materialización/memoria de trabajo temporal.

### 23.9 Benchmark sin hipótesis

Números sin pregunta no producen decisión. Corrección: formular variable, patrón esperado y criterio de refutación.

### 23.10 Benchmark con variables mezcladas

Datasets, targets o environments diferentes impiden atribuir el cambio. Corrección: controlar lo no investigado.

### 23.11 Setup accidental

Medir generation/printing cuando se afirma medir lookup cambia la pregunta. Corrección: delimitar región.

### 23.12 Una sola ejecución

Una muestra puede ser ruido. Corrección: repetir y usar resumen sencillo sin ocultar variabilidad.

### 23.13 Optimización antes de medir

Una estructura compleja puede aumentar bugs y mantenimiento sin cruzar el trigger. Corrección: solución clara → modelo → evidencia → cambio.

### 23.14 Índice como source of truth

Si el dict derivado se vuelve la única autoridad y diverge del journal, la optimización rompe provenance/replay. Corrección: reconstrucción comprobable y ownership explícito.

### Diagnostica

Elige cuatro failure cases y escribe para cada uno: síntoma observable, causa, evidencia que conservarías y cambio mínimo.

---

## 24. Ejercicios guiados con solución razonada

En cada ejercicio escribe primero tu predicción. Después contrástala con la derivación; ejecutar código sirve para confirmar propiedades, no para adivinar la notación.

### Ejercicio guiado 1 — Loop lineal

**Objetivo:** definir tamaño y derivar crecimiento.  
**Input:** `sum(1 for event in events if event["active"])`.  
**Predice:** costo respecto de `n = Events`.  
**Solución razonada:** (1) la generator considera los `n` Events; (2) leer `active` y acumular se modela O(1) por Event; (3) `n · O(1)` da Θ(n) tiempo; (4) `sum` consume un valor a la vez, por lo que el auxiliary space es O(1).  
**Criterio:** no afirmar O(1) por usar `sum`.  
**Variación:** si `active` se consulta mediante una función O(m), el total puede ser O(nm).

### Ejercicio guiado 2 — Dos etapas secuenciales

**Objetivo:** sumar etapas y separar output space.  
**Input:** un loop valida `n` Events y otro construye `n` labels.  
**Predice:** O(n) + O(n).  
**Solución razonada:** (1) la validación cuesta `an`; (2) construir labels cuesta `bn`; (3) son etapas sucesivas, así que se suman: `(a+b)n`; (4) el tiempo es Θ(n) y conservar `n` labels exige Θ(n) output space.  
**Criterio:** separar tiempo de espacio.

### Ejercicio guiado 3 — Nested loops

**Objetivo:** multiplicar trabajo dependiente.  
**Input:** comparar cada Event con cada Event.  
**Predice:** número de comparaciones y memoria si se guardan todos los pares.  
**Solución razonada:** (1) el outer loop elige `n` Events; (2) para cada uno, el inner recorre `n`; (3) hay `n × n = n²` comparaciones; (4) materializar cada par produce también Θ(n²) output space.  
**Criterio:** explicar ambas dimensiones del resultado.

### Ejercicio guiado 4 — `n` y `m`

**Objetivo:** conservar dimensiones independientes.  
**Input:** por cada Event recorrer `requested_tags`.  
**Predice:** costo para `n` Events y `m` requested tags.  
**Solución razonada:** (1) el outer loop aporta `n`; (2) el inner aporta `m`; (3) el producto es O(nm); (4) solo puede expresarse O(n²) bajo el supuesto adicional `m ∈ Θ(n)`.  
**Criterio:** no ocultar la segunda entrada.

### Ejercicio guiado 5 — Space complexity

**Objetivo:** distinguir auxiliary y output space.  
**Input:** scan que conserva el mejor Event frente a comprehension de todos los matches.  
**Predice:** tiempo y memoria adicional con `n` Events y `k` matches.  
**Solución razonada:** (1) ambas versiones inspeccionan hasta `n` Events, por lo que usan Θ(n) tiempo; (2) el scan conserva una referencia y usa O(1) auxiliary space; (3) la comprehension materializa `k` referencias y exige Θ(k) output space.  
**Criterio:** distinguir output obligatorio de memoria auxiliar.

### Ejercicio guiado 6 — Append amortized

**Objetivo:** explicar amortización sin probabilidades.  
**Input:** `for item in items: result.append(item)`.  
**Predice:** costo total de la secuencia y costo amortizado por operación.  
**Solución razonada:** (1) muchos appends usan capacidad disponible; (2) algunas expansiones copian referencias y son más caras; (3) bajo el modelo de dynamic array, el costo acumulado de `n` appends es O(n); (4) dividir por `n` da O(1) amortized por append.  
**Criterio:** no afirmar que cada append individual cuesta exactamente lo mismo.

### Ejercicio guiado 7 — Scan repetido

**Objetivo:** incorporar cantidad de consultas.  
**Input:** `q` IDs, cada uno buscado en `n` Events.  
**Predice:** worst-case time y space bajo dos contratos de resultado.  
**Solución razonada:** (1) cada consulta puede recorrer `n`; (2) repetirla `q` veces da Θ(qn) worst-case time; (3) guardar las `q` respuestas usa Θ(q) output space; (4) procesarlas una por una puede mantener O(1) auxiliary space.  
**Criterio:** incluir `q`.

### Ejercicio guiado 8 — Índice derivado

**Objetivo:** modelar build, lookup, memoria y autoridad.  
**Input:** comprehension `event_id → event`.  
**Predice:** costo de construir y consultar; señala qué ocurre con IDs duplicados.  
**Solución razonada:** (1) build visita `n` Events; (2) cada inserción se modela expected average O(1), por lo que build es expected Θ(n); (3) el índice conserva `n` entradas, Θ(n) space; (4) lookup es expected average O(1); (5) una comprehension sobrescribiría duplicates, así que un build seguro necesita policy explícita. El dict sigue siendo derived data, no source persistente.  
**Criterio:** incluir build, memoria, expected y duplicate policy.

### Ejercicio guiado 9 — Build + consultas

**Objetivo:** comparar estrategias completas.  
**Input:** una estrategia construye índice una vez y responde `q` queries.  
**Predice:** costo total indexado frente a scans repetidos.  
**Solución razonada:** (1) build expected Θ(n); (2) `q` lookups expected O(q); (3) la suma es expected Θ(n+q); (4) `q` scans tienen worst Θ(qn); (5) con `q=1`, build, hashing y allocations pueden hacer perder al índice pese a la diferencia asintótica del lookup.  
**Criterio:** no comparar solo lookups.

### Ejercicio guiado 10 — Benchmark temporal

**Objetivo:** delimitar la región medida.  
**Input:** dataset y targets generados dentro del timer.  
**Predice:** qué operación describe el tiempo obtenido.  
**Solución razonada:** (1) el timer incluye generación y lookup; (2) por tanto no aísla lookup; (3) mueve dataset/targets fuera si esa es la pregunta; (4) consérvalos dentro solo para una medición end-to-end que los incluya por contrato; (5) valida resultados y repite.  
**Criterio:** la región responde la pregunta declarada.

### Ejercicio guiado 11 — Tamaños crecientes

**Objetivo:** diseñar una curva comparable.  
**Input:** 10², 10³, 10⁴, 10⁵.  
**Predice:** qué variables deben permanecer fijas.  
**Solución razonada:** (1) cambia solo `n`; (2) conserva `q`, distribución de targets, semántica y environment; (3) repite y registra mediana junto con muestras; (4) busca tendencia, no igualdad exacta con una fórmula; (5) no fuerces 10⁶ si provoca swapping o una sesión impráctica.  
**Criterio:** misma operación y environment.

### Ejercicio guiado 12 — Memoria

**Objetivo:** elegir una medición espacial defendible.  
**Input:** `getsizeof(index)`.  
**Predice:** qué incluye y qué omite el número.  
**Solución razonada:** (1) `getsizeof(index)` mide el contenedor superficial; (2) no recorre keys, values ni objetos compartidos; (3) usa una sesión aislada de `tracemalloc` alrededor del build para comparar allocations Python; (4) conserva current/peak y declara que no son RSS universal.  
**Criterio:** no presentar bytes como grafo transitivo/RSS universal.

### Ejercicio guiado 13 — Interpretar el cruce

**Objetivo:** interpretar evidencia sin universalizarla.  
**Input:** lookup indexado gana aislado; total indexado gana desde `q=20`.  
**Predice:** qué resultado determina la estrategia completa.  
**Solución razonada:** (1) lookup aislado omite build; (2) compara `scan_total` con `build + indexed_queries`; (3) el primer cruce consistente alrededor de `q=20` es local a ese `n`, dataset, distribución y equipo; (4) repite antes de convertirlo en decisión.  
**Criterio:** no universalizar `q=20`.

### Ejercicio guiado 14 — Migration trigger

**Objetivo:** convertir el riesgo en un criterio falsable.  
**Input:** scan actual sin SLO.  
**Predice:** qué datos faltan para decidir un cambio.  
**Solución razonada:** (1) nombra operation y distribución; (2) fija `n/q` objetivo; (3) elige métrica y environment; (4) define umbral y frecuencia; (5) especifica que cruzarlo activa reevaluación. No prescribe una tecnología específica.  
**Criterio:** trigger falsable y medible.

---

## 25. Ejercicios independientes

No consultes soluciones hasta conservar tu derivación y mediciones.

1. Define `n` para normalizar un texto y para filtrar un journal.
2. Encuentra una función con dos dimensiones y nómbralas.
3. Deriva un loop de un solo recorrido con early return.
4. Deriva dos loops secuenciales sobre la misma list.
5. Deriva nested loops sobre `n` Events y `m` tags.
6. Analiza un inner loop limitado a siete estados.
7. Deriva un loop triangular mediante sumatoria.
8. Distingue best/worst de target al inicio, final y ausente.
9. Escribe un supuesto que produzca average Θ(n) para linear search.
10. Da una cota O correcta pero floja y una Θ ajustada.
11. Explica por qué la base del log no cambia O(log n).
12. Compara crecimiento al duplicar `n` para lineal y cuadrático.
13. Encuentra hidden work en tres built-ins.
14. Analiza comprehension con membership en list de tamaño `m`.
15. Repite el análisis cuando membership usa set preconstruido.
16. Señala build/space omitidos al recomendar dict.
17. Distingue average de amortized con ejemplos propios.
18. Explica por qué un resize caro no contradice append amortized O(1).
19. Calcula auxiliary space de scan, list output e index dict.
20. Convierte una list comprehension a generator y separa tiempo/memoria.
21. Demuestra consumo único del generator sin llamarlo “más rápido”.
22. Diseña targets best, worst y absent para scan.
23. Compara una query frente a 1,000 sobre el mismo snapshot.
24. Mide `n = 10²..10⁵` con `perf_counter` en batches.
25. Repite con `timeit.repeat` y conserva todas las muestras.
26. Compara media y mediana cuando una muestra es un outlier.
27. Introduce printing en el benchmark, observa y explica la contaminación.
28. Mide build separado de lookup.
29. Mide total `build + q lookups` frente a `q scans`.
30. Usa `tracemalloc` para el índice con input creado antes.
31. Explica por qué `getsizeof(events)` no mide todos sus dicts.
32. Ejecuta dos mediciones de memoria aisladas, no acumuladas.
33. Construye una tabla con `n`, `q`, tiempos y peak memory.
34. Encuentra un break-even observado para un `n` concreto.
35. Cambia distribución de targets y explica el nuevo cruce.
36. Cambia longitud de IDs y registra el supuesto afectado.
37. Diseña un benchmark end-to-end que incluya replay, sin confundirlo con lookup.
38. Escribe un Complexity Budget de tres operaciones.
39. Define un trigger de latencia y otro de memoria.
40. Revisa una propuesta de servicio de índice externo sin benchmark y enumera la evidencia faltante, sin elegir tecnología.
41. Explica por qué ordenar en cada consulta cambia el costo total.
42. Define `n` y `k` para top-k sin implementar heap.
43. Define `V` y `E` para un graph futuro sin implementar recorrido.
44. Documenta Python version, hardware relevante y carga durante una corrida.
45. Identifica qué resultado invalidaría tu hipótesis inicial.

---

## 26. Preguntas conceptuales

1. ¿Qué debe definirse antes de afirmar O(n)?
2. ¿Qué significa realmente O(1)?
3. ¿Por qué O(1) no implica una instrucción ni tiempo instantáneo?
4. ¿Por qué Big O no predice segundos?
5. ¿Qué diferencia existe entre O, Ω y Θ?
6. ¿Por qué una cota correcta puede ser poco informativa?
7. ¿Big O significa automáticamente worst case?
8. ¿Cuándo dos nested loops no implican O(n²)?
9. ¿Por qué dos loops secuenciales no se multiplican?
10. ¿Qué muestra la sumatoria triangular?
11. ¿Por qué average case necesita una distribución?
12. ¿Qué diferencia existe entre average y amortized?
13. ¿Cómo puede un append individual ser caro y el costo amortized O(1)?
14. ¿Qué diferencia existe entre input, output y auxiliary space?
15. ¿Por qué una generator expression no reduce automáticamente el tiempo total?
16. ¿Qué trabajo oculta `target in ids` cuando `ids` es list?
17. ¿Qué cambia si `ids` es set ya construido?
18. ¿Qué tradeoff introduce un índice dict?
19. ¿Por qué el índice no debe ser source of truth?
20. ¿Qué costo se omite al comparar solo lookup indexado con scan?
21. ¿Por qué una solución O(n) puede ganar a O(1) expected para inputs pequeños?
22. ¿Qué significa break-even point en este contexto?
23. ¿Por qué un benchmark no demuestra una clase asintótica?
24. ¿Qué diferencia existe entre microbenchmark e integration benchmark?
25. ¿Cuándo debe el setup quedar fuera del timer?
26. ¿Por qué una sola ejecución es evidencia débil?
27. ¿Qué ventaja práctica ofrece la mediana ante outliers?
28. ¿Qué mide `getsizeof` y qué omite?
29. ¿Qué rastrea `tracemalloc` y qué no debe afirmarse a partir de él?
30. ¿Qué diferencia existe entre benchmark y profiling?
31. ¿Por qué “optimización prematura” no autoriza ignorar costos obvios?
32. ¿Qué información necesita un migration trigger?
33. ¿Cuándo conservarías scan lineal en EIDOLON?
34. ¿Qué evidencia justificaría construir un índice derivado?
35. ¿Qué observation te haría revertir una optimización?

---

## 27. Mini challenge — Scan vs índice con evidencia

### 27.1 Objetivo

Construye un experimento reproducible que compare búsquedas repetidas por `event_id` mediante:

- **Estrategia A:** scan lineal;
- **Estrategia B:** índice `dict` derivado en memoria.

Debes producir análisis, medición y decisión. No basta con imprimir cuál ganó una corrida.

### 27.2 Artefacto

```text
cs-m1-cost-model/
├── README.md
├── analysis.md
├── benchmark.py
├── checks.py
├── complexity_budget.md
└── results/
    └── sample_table.md
```

No necesitas packaging nuevo si ejecutas desde el environment aprobado de Programming Foundations. Todos los datos son sintéticos.

### 27.3 Dataset

`make_events(n)` crea IDs únicos deterministas, `person_id` y tags acotados. Usa tamaños:

```text
100, 1_000, 10_000, 100_000
```

`1_000_000` es opcional y solo se ejecuta si el equipo lo permite. Registra tamaños omitidos.

Para cada `n`, prueba `q` en al menos:

```text
1, 10, 100, 1_000
```

Reduce combinaciones si una corrida excede un límite razonable documentado.

### 27.4 Análisis previo

Antes de medir, `analysis.md` debe:

1. definir `n` y `q`;
2. declarar que comparar IDs se modela O(1) por longitud acotada;
3. derivar scan worst Θ(qn), auxiliary space Θ(1) sin materializar respuestas;
4. derivar index build expected Θ(n), queries expected O(q), space Θ(n);
5. distinguir expected lookup de worst case teórico;
6. formular una hipótesis de break-even;
7. predecir cómo crecerá peak memory;
8. declarar qué observación refutaría la hipótesis.

### 27.5 Corrección antes de rendimiento

**Fragmento — comprobaciones que debe contener `checks.py`:**

```python
assert query_by_scan(events, targets) == query_by_index(index, targets)
assert len(index) == len(events)
assert build_index(events) == index
```

Agrega un duplicate ID y exige `ValueError`. Reconstruye el índice desde la list fuente; no serialices el índice ni lo conviertas en authority.

### 27.6 Medición temporal

Separa:

- dataset generation;
- scan queries;
- index build;
- indexed queries;
- `index build + indexed queries`.

Usa `timeit.repeat` o `perf_counter` con batches y múltiples repeticiones. Conserva mediana y muestras suficientes para detectar ruido básico. No incluyas printing/logging en la región medida.

### 27.7 Medición espacial

Usa `tracemalloc` en una corrida aislada del build. Reporta:

- retained delta aproximado;
- peak delta aproximado;
- limitación: allocations Python rastreadas, no memoria total del sistema.

Puedes mostrar `getsizeof(index)` solo como shallow size claramente etiquetado.

### 27.8 Tabla

Cada fila contiene:

```text
n | q | scan median | build median | indexed-query median
  | indexed total | retained bytes | peak bytes
```

No redondees hasta volver indistinguibles valores pequeños; tampoco presentes más precisión que la medición justifica.

### 27.9 Interpretación

Responde:

1. ¿Dónde apareció el primer cruce consistente?
2. ¿Cambió con `n` o distribución de targets?
3. ¿El lookup aislado y la estrategia completa cuentan la misma historia?
4. ¿Qué constantes o efectos explican discrepancias con la intuición?
5. ¿Qué costo espacial compra la mejora temporal?
6. ¿Qué no puede concluirse fuera de este environment?

### 27.10 Complexity Budget

Documenta al menos:

| operation | expected `n/q` | strategy | time | auxiliary space | measured result | migration trigger |
|---|---|---|---|---|---|---|
| single show | por medir | scan/index | derivado | derivado | local | cuantitativo |
| batch lookup | por medir | scan/index | derivado | derivado | local | cuantitativo |
| rebuild | `n` records | dict | expected Θ(n) | Θ(n) | local | memory/latency |

### 27.11 Migration trigger

Define workload, métrica, umbral `X/Y`, environment y frecuencia. El trigger pide reevaluar estrategia; no prescribe una tecnología.

### 27.12 Failure cases obligatorios

Demuestra o diagnostica:

1. benchmark que incluye dataset generation accidentalmente;
2. una sola ejecución ruidosa;
3. scan con targets al inicio comparado contra index con targets distintos;
4. índice medido sin build y presentado como estrategia total;
5. `getsizeof` presentado incorrectamente como memoria transitiva;
6. duplicate ID que sobrescribe sin policy;
7. índice tratado como source of truth;
8. afirmación “dict siempre gana”.

### 27.13 Criterio de aprobación

- análisis define todas las dimensiones;
- ambas estrategias producen resultados equivalentes;
- derivaciones temporales y espaciales son correctas;
- average/amortized no se confunden;
- benchmark controla dataset, targets y setup;
- tamaños crecen y cada punto usa múltiples repeticiones;
- build y lookup están separados y también sumados;
- `tracemalloc` se interpreta prudentemente;
- break-even se reporta como observación local;
- Complexity Budget y trigger son cuantitativos;
- journal/list de Events sigue siendo source del experimento;
- no se implementan internals de hash map ni conceptos de CS-M2+.

Output final después de checks y tabla:

```text
CS-M1 cost-model challenge: PASS
```

---

## 28. Resumen

- Un modelo de costo declara recurso, operación, tamaño y supuestos.
- `n` es el tamaño relevante de la entrada, no líneas de código.
- Varias dimensiones deben conservar nombres distintos.
- O es cota superior; Ω inferior; Θ ajusta por ambos lados.
- Big O describe crecimiento, no segundos ni hardware.
- O(1) no significa instantáneo.
- O(log n) aparece al reducir el problema por un factor.
- Loops secuenciales suman; trabajo anidado multiplica dimensiones.
- El término dominante guía crecimiento sin borrar constantes prácticas.
- Best, average y worst case son funciones distintas.
- Average case requiere una distribución; amortized distribuye costo de una secuencia.
- Space complexity incluye materialización y estructuras auxiliares.
- Generators pueden reducir materialización, no el trabajo de consumir todos los elementos.
- Built-ins y comprehensions pueden ocultar scans o sorting.
- Un índice cambia build/memoria por lookups expected average O(1).
- El índice EIDOLON es derived y reconstruible; el journal conserva authority.
- Big O formula hipótesis; benchmark aporta evidencia local.
- `perf_counter` mide duración; `timeit` ayuda con repetición controlada.
- Setup, distribución, tamaños y environment forman parte del experimento.
- `getsizeof` es shallow; `tracemalloc` rastrea allocations Python con límites.
- Mediana reduce influencia de outliers sin sustituir estadística formal.
- Benchmark compara; profiling localiza bottlenecks.
- Optimizar requiere corrección, riesgo, evidencia y un cambio defendible.
- Complexity Budget y migration trigger convierten medición en decisión.

---

## 29. Checklist de dominio

- [ ] Puedo definir `n` antes de usar notación asintótica.
- [ ] Puedo reconocer una segunda dimensión `m`, `q` o `k`.
- [ ] Puedo explicar O, Ω y Θ sin confundirlas.
- [ ] Puedo explicar por qué Big O no produce segundos.
- [ ] Puedo comparar O(1), O(log n), O(n), O(n log n) y O(n²).
- [ ] Puedo sumar etapas secuenciales y multiplicar trabajo anidado.
- [ ] Puedo derivar una sumatoria triangular sencilla.
- [ ] Puedo distinguir best, average y worst case.
- [ ] Puedo declarar el supuesto de un average case.
- [ ] Puedo distinguir average de amortized.
- [ ] Puedo explicar append O(1) amortized sin depender del factor de CPython.
- [ ] Puedo detectar hidden work en membership, slicing, copying y sorting.
- [ ] Puedo analizar una comprehension sin confundir compactación con menor costo.
- [ ] Puedo separar time y auxiliary space.
- [ ] Puedo explicar qué materializa una list y qué difiere un generator.
- [ ] Puedo justificar el time-space tradeoff de un índice.
- [ ] Puedo derivar `q` scans frente a build + `q` lookups.
- [ ] Puedo formular una hipótesis de break-even.
- [ ] Puedo usar `perf_counter` sobre una región deliberada.
- [ ] Puedo usar `timeit.repeat` y resumir con mediana.
- [ ] Puedo separar setup de operación cuando la pregunta lo requiere.
- [ ] Puedo diseñar tamaños y targets comparables.
- [ ] Puedo explicar por qué una muestra no basta.
- [ ] Puedo usar `getsizeof` solo como shallow size.
- [ ] Puedo medir allocations Python con `tracemalloc` y declarar límites.
- [ ] Puedo distinguir microbenchmark, integration benchmark y profiling.
- [ ] Puedo interpretar un cruce como resultado local, no universal.
- [ ] Puedo conservar journal/source e índice derived separados.
- [ ] Puedo escribir un Complexity Budget.
- [ ] Puedo definir un migration trigger cuantitativo sin elegir tecnología prematuramente.
- [ ] Puedo completar el mini challenge con Programming Foundations + CS-M1.

---

## 30. Preparación para labs y EIDOLON 0.0b

### CS-L01 — Curvas de crecimiento

Es el lab principal. CS-M1 prepara:

- tamaños 10²–10⁶ condicionados por el equipo;
- curvas O(n), contexto O(n log n) y lookup hash esperado;
- repetición, mediana, tablas y control de setup;
- interpretación de constantes y break-even;
- medición prudente de memoria.

Evidencia previa: challenge con tabla, hypothesis y Complexity Budget.

### CS-L02 — List vs dict entity lookup

CS-M1 aporta el cost model, benchmark y time-space tradeoff. CS-M2 añadirá internals de arrays/hash maps, colisiones, load factor y análisis más preciso. No inicies la implementación didáctica del hash map solo con este módulo.

### CS-MP1 — In-memory Event Index

Quedan preparados:

- baseline scan;
- costo de build y queries;
- memoria del índice;
- rebuild desde source;
- threshold de cambio.

Faltan CS-M2 para justificar internals de list/dict/set, CS-M4 para búsquedas/orden temporal y CS-M5 para top-k.

### EIDOLON 0.0b

CS-M1 aporta el primer `Complexity Budget` y triggers de migración. No cambia EIDOLON 0.0a ni agrega tecnología. Su contribución es hacer explícitos los costos que CS-M2–CS-M10 comprobarán con estructuras y sistemas.

### Evidencia antes de CS-M2

1. catorce ejercicios guiados reproducidos;
2. al menos veinte ejercicios independientes, incluidos 4, 7, 9, 14, 16, 17, 19, 23, 24, 28–35, 38, 39 y 45;
3. benchmark en dos valores de `n` y dos de `q` como mínimo;
4. memoria medida con límites escritos;
5. Complexity Budget de tres operaciones;
6. trigger cuantitativo defendido;
7. explicación oral o escrita de average vs amortized;
8. demostración de que el índice se reconstruye desde source.

---

## 31. Recursos de ampliación

El módulo es autocontenido para su gate futuro. Para ampliar, usa selectivamente los recursos canónicos de [`CS.11`](../../02_curriculum/02_computer_science_foundations.md#cs11-recursos-recomendados): MIT OCW 6.006 para modelos y growth, CLRS para notación/cotas y Algorithm Design Manual para decisiones aplicadas.

La documentación oficial de Python sobre `time`, `timeit`, `tracemalloc`, `statistics`, complejidad práctica de built-ins y `sys.getsizeof` sirve para verificar APIs. No sustituye definir workload ni interpretar evidencia.

---

## 32. Límite explícito del módulo

CS-M1 termina en análisis asintótico elemental, space complexity, amortización intuitiva, benchmark de tiempo/memoria, break-even, Complexity Budget y migration triggers.

Quedan fuera:

- internals de arrays/hash maps/sets y colisiones: CS-M2;
- stacks, queues, deques y linked lists: CS-M3;
- recursion, binary search y sorting algorithms: CS-M4;
- trees, heaps y top-k implementado: CS-M5;
- BFS/DFS, graphs y state machines: CS-M6;
- OS, virtual memory y crash consistency profunda: CS-M7;
- processes, threads, locks y races: CS-M8;
- DNS, TCP, TLS y HTTP: CS-M9;
- CPU/cache/I/O y architecture detallada: CS-M10.

No se introducen backend, databases, vector stores, AI, frameworks ni servicios externos. Tampoco se desarrolla teoría de complejidad computacional, master theorem, cálculo de límites, pruebas formales extensas o profiling avanzado.

El siguiente paso permitido es revisar CS-M1 como `review candidate`. **No se crea ni se desarrolla CS-M2 en esta entrega.**
