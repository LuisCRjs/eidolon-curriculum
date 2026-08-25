# PF-M3 — Colecciones, comprehensions e iteración

**Track:** Programming Foundations  
**Competencias:** D1.1, D2.1; refuerza D3.1 y D11.1  
**Fase:** P0  
**Nivel objetivo:** Profesional para el alcance del módulo  
**Prerequisites:** PF-M1, PF-M2  
**Build:** EIDOLON 0.0a  
**Curriculum source:** [PF-M3](../../02_curriculum/01_programming_foundations.md#pf-m3--colecciones-comprehensions-e-iteración)  
**Status:** approved

Un evento aislado rara vez basta. En cuanto un programa debe conservar una timeline, localizar una entidad por ID, deduplicar tags o recorrer datos sin cargarlos todos, necesita representar grupos de valores y elegir cómo procesarlos. La colección elegida comunica un contrato: qué orden importa, qué puede cambiar, cómo se accede y qué operaciones dominan.

Ya aprobaste PF-M1 y PF-M2. Por tanto puedes razonar sobre nombres, referencias, identidad, igualdad, mutabilidad, truthiness, funciones, contratos, scopes, pureza y efectos. PF-M3 usa esos fundamentos para elegir y transformar colecciones. No introduce clases, dataclasses, type hints, módulos propios, archivos, JSON, decorators, async, frameworks ni persistencia.

Los costos se describen de manera aproximada para tomar decisiones locales. **CS-M1** y **CS-M2** formalizarán complejidad, análisis y estructuras de datos; PF-M3 no sustituye esos módulos.

## Resultados de aprendizaje

Al terminar deberías poder:

- distinguir un valor, una secuencia, una asociación clave → valor y un conjunto de elementos únicos;
- elegir entre `list`, `tuple`, `dict`, `set` y `frozenset` según orden, mutabilidad, acceso, asociación y unicidad;
- predecir cómo referencias y aliasing afectan colecciones mutables y anidadas;
- usar operaciones esenciales de `list` y explicar sus costos aproximados;
- usar `tuple` para expresar una estructura posicional estable y aplicar unpacking;
- construir, consultar, actualizar e iterar diccionarios sin confundir keys con values;
- explicar hashability al nivel necesario para elegir keys y elementos de sets;
- usar sets para membership, deduplicación y operaciones de conjuntos;
- justificar cuándo una búsqueda lineal es suficiente y cuándo conviene un índice `dict` o `set`;
- distinguir iterable de iterator y predecir consumo, reutilización y `StopIteration`;
- usar `range`, `enumerate` y `zip`, incluido `strict=True`, según el contrato;
- convertir un loop claro en una comprehension equivalente y reconocer cuándo no hacerlo;
- evitar side effects y complejidad escondida dentro de comprehensions;
- distinguir materialización eager de procesamiento lazy;
- usar una generator expression y una función generadora mínima sin asumir que pueden recorrerse dos veces;
- aplicar `any`, `all`, `sum`, `min`, `max` y `sorted` a problemas concretos;
- construir índices derivados de eventos sintéticos sin tratarlos como source of truth.

## Cómo estudiar este módulo

Para cada bloque importante:

1. identifica el contrato de la colección: orden, duplicados, acceso y mutabilidad;
2. predice el output o el estado antes de ejecutar;
3. ejecuta el ejemplo;
4. modifica un dato o una operación;
5. explica el costo dominante de manera cualitativa;
6. pregunta si el resultado debe materializarse o puede consumirse una sola vez.

### Convenciones del código

- **Ejemplo ejecutable:** bloque autónomo que corre sin preparación adicional.
- **Continuación:** depende solo del bloque inmediatamente anterior y se indica de forma explícita.
- **Código incorrecto:** antipatrón deliberado; observa el síntoma y explica la causa.
- **Failure case:** provoca el error indicado. El manejo formal de excepciones pertenece a PF-M6.
- **Fragmento:** ilustra sintaxis o una decisión; no se ofrece como programa autónomo.
- **Solución parcial:** deja decisiones al estudiante y declara qué falta.

Los outputs corresponden a Python 3.14. No se fija el orden impreso de un `set`, porque ese orden no forma parte de su contrato. Cuando importa un output determinista se usa `sorted` o un `assert` sobre propiedades.

### Sintaxis de apoyo

- `try`/`except` se usa solo para observar un error deliberado; la taxonomía y recuperación llegan en PF-M6;
- `raise` puede aparecer en un contrato de validación ya presentado en PF-M2, pero no se exige diseñar excepciones nuevas;
- `assert` comprueba propiedades pequeñas; testing profesional llega en PF-M9;
- `yield` se explica localmente para una función generadora mínima; generators avanzados y ownership de archivos quedan fuera;
- `lambda` no es necesaria en este módulo: cuando `sorted` necesita una key, preferimos una función nombrada de PF-M2.

---

## 1. Modelo mental de colección

### 1.1 El problema que resuelven las colecciones

Una variable puede enlazarse con un único valor:

```python
event_id = "evt-001"
```

Pero una timeline contiene varios eventos, un índice relaciona IDs con entidades y un conjunto de tags necesita eliminar duplicados. Guardar cada elemento bajo un nombre independiente (`event_1`, `event_2`, `event_3`) no ofrece una forma general de recorrer, filtrar o buscar.

Una **colección** representa varios elementos bajo un contrato común. Antes de elegir sintaxis, pregunta qué relación existe entre esos elementos.

### 1.2 Cuatro formas conceptuales

| Forma | Pregunta principal | Ejemplo pequeño |
|---|---|---|
| Un único valor | ¿Cuál es este dato? | `event_id = "evt-001"` |
| Secuencia | ¿Qué elementos aparecen y en qué orden? | timeline de eventos |
| Asociación clave → valor | ¿Qué valor corresponde a esta clave? | `entity_id -> entity` |
| Conjunto | ¿Qué elementos únicos pertenecen? | IDs ya vistos |

Estas formas no son intercambiables. Una secuencia puede contener duplicados porque dos posiciones pueden guardar valores iguales. Una asociación necesita keys. Un conjunto expresa pertenencia y unicidad, no posición.

### 1.3 Las preguntas de diseño

Antes de elegir una colección, responde:

1. ¿El orden tiene significado o solo facilita presentación?
2. ¿Necesito acceder por posición, por key o solo preguntar membership?
3. ¿Se permiten duplicados?
4. ¿La estructura debe mutar después de construirse?
5. ¿Qué operación se repetirá más: recorrer, buscar, insertar, eliminar o combinar?
6. ¿Necesito todos los resultados ahora o puedo procesarlos uno por uno?

No existe una colección universalmente “mejor”. Existe una colección coherente o incoherente con el contrato.

### 1.4 Conexión con PF-M1: referencias, identidad y aliasing

Una colección también es un objeto. Asignar otro nombre no la copia.

**Ejemplo ejecutable:**

```python
active_tags = ["familia", "viaje"]
shared_tags = active_tags

shared_tags.append("prioritario")

print(active_tags)
# ['familia', 'viaje', 'prioritario']
print(active_tags is shared_tags)
# True
```

`active_tags` y `shared_tags` son referencias al mismo objeto mutable. La elección de `list` no elimina las reglas de PF-M1; las vuelve más importantes porque una mutación puede ser visible desde varios nombres.

### Predice

Sin ejecutar, predice `timeline` después del cambio:

```python
timeline = ["evt-001", "evt-002"]
visible_events = timeline
visible_events[0] = "evt-099"
```

Explica qué nombre fue reasignado y qué objeto fue mutado.

### Modifica

Cambia el ejemplo anterior para que `visible_events` sea una colección exterior independiente mediante una copia superficial. Comprueba con `is` y `==` que los contenedores son distintos pero inicialmente iguales.

### Explica

¿Por qué “guardar varios valores” no basta para decidir entre `list`, `dict` y `set`? Responde usando al menos dos operaciones dominantes.

### Detecta el bug

```python
seen_source_ids = []

# El contrato exige IDs únicos, pero nada impide repetirlos.
seen_source_ids.append("src-001")
seen_source_ids.append("src-001")
```

El bug no es que `list` funcione mal. La estructura elegida no expresa la invariante de unicidad.

---

## 2. `list`: secuencia mutable y ordenada

### 2.1 Cuándo expresa bien el problema

Una `list` es apropiada cuando:

- el orden forma parte del resultado o del proceso;
- pueden existir duplicados;
- necesitas acceder por posición;
- la colección exterior debe crecer, reducirse o reemplazar elementos.

Una timeline pequeña en memoria puede ser una list. Eso no significa que una list sea una base de datos ni que su posición sea una identidad estable.

### 2.2 Creación, índice e índices negativos

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002", "evt-003"]
empty_events = []

print(event_ids[0])
# evt-001
print(event_ids[-1])
# evt-003
print(len(empty_events))
# 0
```

Los índices empiezan en `0`. Un índice negativo cuenta desde el final: `-1` es el último elemento. Acceder fuera de los límites produce `IndexError`; PF-M6 estudiará la política de manejo, pero aquí debes poder predecir la causa.

**Failure case — índice fuera de rango:**

```python
event_ids = ["evt-001"]
print(event_ids[1])  # IndexError
```

Antes de ejecutar, predice el tipo de error. El índice `1` no existe en una list de longitud `1`; corrige el límite o itera los elementos si la posición no forma parte del contrato. Para diagnosticar conserva la list y el índice solicitado.

### 2.3 Slicing

Un slice selecciona una región mediante `start:stop:step`. `stop` no se incluye.

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002", "evt-003", "evt-004"]

print(event_ids[1:3])
# ['evt-002', 'evt-003']
print(event_ids[:2])
# ['evt-001', 'evt-002']
print(event_ids[::2])
# ['evt-001', 'evt-003']
print(event_ids[::-1])
# ['evt-004', 'evt-003', 'evt-002', 'evt-001']
```

Un slice de list crea una list exterior nueva. Si los elementos son objetos mutables, esos objetos siguen compartidos: es una copia superficial.

### 2.4 Mutabilidad y operaciones de crecimiento

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001"]

event_ids.append("evt-002")
event_ids.extend(["evt-003", "evt-004"])
event_ids.insert(1, "evt-001b")

print(event_ids)
# ['evt-001', 'evt-001b', 'evt-002', 'evt-003', 'evt-004']
```

- `append(value)` añade **un** elemento al final;
- `extend(iterable)` recorre otro iterable y añade cada elemento;
- `insert(index, value)` inserta antes de la posición indicada y desplaza elementos posteriores.

Confundir `append` y `extend` cambia la forma de la colección.

**Ejemplo ejecutable — contraste:**

```python
first = ["evt-001"]
second = ["evt-001"]

first.append(["evt-002", "evt-003"])
second.extend(["evt-002", "evt-003"])

print(first)
# ['evt-001', ['evt-002', 'evt-003']]
print(second)
# ['evt-001', 'evt-002', 'evt-003']
```

### 2.5 Eliminación

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002", "evt-003", "evt-002"]

event_ids.remove("evt-002")  # elimina la primera coincidencia
last_id = event_ids.pop()     # elimina y devuelve el último
del event_ids[0]              # elimina por posición

print(event_ids)
# ['evt-003']
print(last_id)
# evt-002
```

`remove` busca por igualdad y falla con `ValueError` si no encuentra el valor. `pop(index)` elimina por posición y devuelve el elemento; sin argumento usa el final. `del` es una sentencia y no devuelve el elemento.

### 2.6 Membership e iteración

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002", "evt-003"]

print("evt-002" in event_ids)
# True

for event_id in event_ids:
    print(event_id)
```

Output adicional:

```text
evt-001
evt-002
evt-003
```

En una list, `in` examina elementos hasta encontrar una igualdad o terminar. Es una búsqueda lineal (linear scan). Para una colección pequeña o una consulta ocasional puede ser la decisión más clara. Para miles de consultas repetidas por ID, un índice `dict` o `set` suele expresar mejor el problema.

### 2.7 Costos aproximados que cambian decisiones

Sin formalizar Big O todavía:

| Operación en `list` | Intuición de costo |
|---|---|
| acceder por índice | directo, normalmente barato |
| `append` al final | normalmente barato; costo amortizado |
| `pop()` al final | normalmente barato |
| buscar con `in` o `remove` | puede recorrer gran parte de la list |
| insertar/eliminar al inicio o en medio | desplaza elementos posteriores |
| crear un slice | materializa una list nueva con las referencias seleccionadas |

“Amortizado” significa que una operación ocasional puede requerir reorganizar almacenamiento, pero el costo promedio de muchas inserciones al final sigue siendo bajo. **CS-M1** y **CS-M2** expresarán y analizarán estas relaciones formalmente.

### 2.8 Copia y listas anidadas

**Ejemplo ejecutable — copia superficial:**

```python
original = [["tag-a"], ["tag-b"]]
copied = original.copy()

copied.append(["tag-c"])
copied[0].append("shared")

print(original)
# [['tag-a', 'shared'], ['tag-b']]
print(copied)
# [['tag-a', 'shared'], ['tag-b'], ['tag-c']]
```

La list exterior se separó; la list interior en la posición `0` sigue siendo el mismo objeto. Esto es exactamente el grafo de referencias de PF-M1 aplicado a colecciones anidadas.

### 2.9 Aliasing accidental por repetición

**Código incorrecto — filas compartidas:**

```python
tag_slots = [[]] * 3
tag_slots[0].append("viaje")

print(tag_slots)
# [['viaje'], ['viaje'], ['viaje']]
```

La repetición copió tres referencias a la misma list interior. Si se necesitan filas independientes, constrúyelas una por una:

**Ejemplo ejecutable — corrección:**

```python
tag_slots = [[] for _ in range(3)]
tag_slots[0].append("viaje")

print(tag_slots)
# [['viaje'], [], []]
```

La comprehension se estudiará en la sección 12. Aquí importa la propiedad: se crea una list interior nueva en cada iteración.

### Predice

```python
events = ["evt-001", "evt-002"]
snapshot = events[:]
snapshot.append("evt-003")

print(events)
print(snapshot)
```

Predice ambos outputs y explica por qué este caso no contiene elementos interiores mutables.

### Modifica

Parte de `event_ids = ["evt-001", "evt-003"]`. Inserta `"evt-002"` en su posición temporal, elimina `"evt-003"` obteniendo el valor eliminado y comprueba el estado final con un `assert`.

### Explica

¿Por qué `append` suele ser mejor que `insert(0, value)` para acumular elementos? No uses notación Big O; describe qué elementos deben moverse.

### Detecta el bug

```python
events = [{"id": "evt-001"}, {"id": "evt-002"}]
backup = events.copy()
backup[0]["id"] = "evt-099"
```

Si se esperaba que `events[0]["id"]` siguiera siendo `"evt-001"`, localiza la referencia compartida. No propongas `deepcopy` automáticamente: explica primero si conviene reconstruir el dato derivado.

---

## 3. `tuple`: estructura posicional estable

### 3.1 Más que una list que no cambia

Una `tuple` es una secuencia ordenada cuya **estructura** no permite añadir, eliminar ni reemplazar posiciones. Además de inmutabilidad, suele comunicar que las posiciones forman un registro pequeño o un contrato fijo.

Ejemplos de contratos razonables:

- `(event_id, person_id)` como par;
- `(minimum_score, maximum_score)` como límites;
- el resultado de `enumerate`, que produce pares `(index, value)`.

Una list suele decir “colección homogénea que puede crecer”; una tuple puede decir “estas posiciones tienen roles estables”. El significado debe quedar claro por el contexto y los nombres usados al hacer unpacking.

### 3.2 Creación y la coma de una tuple de un elemento

**Ejemplo ejecutable:**

```python
pair = ("evt-001", "person-007")
single = ("evt-001",)
not_a_tuple = ("evt-001")

print(len(pair))
# 2
print(type(single).__name__)
# tuple
print(type(not_a_tuple).__name__)
# str
```

Los paréntesis ayudan a leer; la coma crea la tuple de un elemento.

### 3.3 Unpacking y múltiples valores

**Ejemplo ejecutable:**

```python
event_ref = ("evt-001", "person-007")
event_id, person_id = event_ref

print(event_id)
# evt-001
print(person_id)
# person-007
```

El número de targets debe coincidir con los elementos, salvo formas de unpacking extendido que no son necesarias aquí.

Python también empaqueta varios valores devueltos en una tuple.

**Ejemplo ejecutable:**

```python
def bounds(scores):
    return min(scores), max(scores)


lowest, highest = bounds([0.2, 0.8, 0.5])

print(lowest)
# 0.2
print(highest)
# 0.8
```

PF-M2 ya estableció que una función debe tener un contrato claro. Devolver dos valores es apropiado si forman un resultado pequeño y estable. Si el caller debe recordar cinco posiciones ambiguas, el contrato necesita otro diseño; PF-M5 introducirá dataclasses.

### 3.4 Estructura inmutable, contenido potencialmente mutable

**Ejemplo ejecutable:**

```python
event_summary = ("evt-001", ["viaje"])
event_summary[1].append("familia")

print(event_summary)
# ('evt-001', ['viaje', 'familia'])
```

La tuple aún contiene las mismas dos referencias. La list interior sí pudo mutar. “Tuple inmutable” no significa “todo el grafo alcanzable es inmutable”.

**Failure case — reemplazar una posición:**

```python
event_summary = ("evt-001", ["viaje"])
event_summary[0] = "evt-002"  # TypeError
```

Antes de ejecutar, predice el tipo de error. La posición exterior no puede reemplazarse; construye una tuple nueva si el contrato necesita otro valor. Conserva la tuple y la posición que se intentó cambiar para explicar el fallo.

### 3.5 Cuándo no usarla

No elijas tuple solo para impedir `append`. Si los elementos forman una colección de longitud variable, una list puede expresar mejor el contrato aunque el caller no deba mutarla. Tampoco uses posiciones crípticas para simular un objeto de dominio complejo; PF-M5 ofrecerá estructuras nombradas.

### Predice

```python
payload = ("evt-001", {"status": "draft"})
payload[1]["status"] = "accepted"
print(payload)
```

Predice el output y separa la mutabilidad de la tuple exterior de la del dict interior.

### Modifica

Escribe una función `split_event_ref` que reciba `"evt-001:person-007"` ya separado como dos argumentos y devuelva una tuple. Haz unpacking del resultado sin usar índices.

### Explica

¿Qué comunica mejor `(minimum, maximum)` que `[minimum, maximum]` cuando el contrato siempre devuelve exactamente dos límites?

### Detecta el bug

```python
event_ref = ("evt-001", "person-007")
event_id, person_id, source_id = event_ref
```

El síntoma es `ValueError`. La corrección no es agregar un dato inventado: alinea el unpacking con el contrato real.

---

## 4. `dict`: asociación clave → valor

### 4.1 El problema del lookup por ID

Con una list de eventos, localizar repetidamente uno por ID exige recorrer hasta encontrarlo:

**Ejemplo ejecutable — búsqueda lineal:**

```python
events = [
    {"id": "evt-001", "text": "Llegada"},
    {"id": "evt-002", "text": "Reunión"},
    {"id": "evt-003", "text": "Regreso"},
]

found = None
for event in events:
    if event["id"] == "evt-002":
        found = event
        break

print(found)
# {'id': 'evt-002', 'text': 'Reunión'}
```

Para una consulta, este loop puede ser suficiente. Si el programa consulta muchas veces por ID, conviene construir una asociación:

```text
entity_id → entity
```

Un `dict` expresa esa relación de manera directa.

### 4.2 Creación, acceso y actualización

**Ejemplo ejecutable:**

```python
events_by_id = {
    "evt-001": {"id": "evt-001", "text": "Llegada"},
    "evt-002": {"id": "evt-002", "text": "Reunión"},
}

print(events_by_id["evt-002"])
# {'id': 'evt-002', 'text': 'Reunión'}

events_by_id["evt-003"] = {"id": "evt-003", "text": "Regreso"}
events_by_id["evt-002"] = {"id": "evt-002", "text": "Reunión corregida"}

print(len(events_by_id))
# 3
print(events_by_id["evt-002"]["text"])
# Reunión corregida
```

Asignar una key nueva agrega una asociación. Asignar una key existente reemplaza su value; no crea una segunda key duplicada.

### 4.3 `KeyError` y `.get`

El acceso `mapping[key]` exige que la key exista.

**Failure case — key ausente:**

```python
events_by_id = {"evt-001": {"id": "evt-001"}}
print(events_by_id["evt-999"])  # KeyError
```

Antes de ejecutar, predice la key reportada. Si la ausencia es esperable, usa membership o `.get` según el contrato; si no lo es, no ocultes el `KeyError`. Para diagnosticar conserva la key solicitada y las keys disponibles.

`.get` permite expresar ausencia sin provocar `KeyError`:

**Ejemplo ejecutable:**

```python
events_by_id = {"evt-001": {"id": "evt-001"}}

print(events_by_id.get("evt-999"))
# None
print(events_by_id.get("evt-999", {"status": "missing"}))
# {'status': 'missing'}
```

No uses `.get` automáticamente. Si `None` es un value válido, `mapping.get(key)` no distingue “key ausente” de “key presente con value `None`”. En ese contrato usa membership primero o un default que no pueda confundirse con datos válidos.

### 4.4 Membership pregunta por keys

**Ejemplo ejecutable:**

```python
status_by_id = {
    "evt-001": "draft",
    "evt-002": "accepted",
}

print("evt-001" in status_by_id)
# True
print("accepted" in status_by_id)
# False
print("accepted" in status_by_id.values())
# True
```

`value in mapping` examina **keys**, no values. Buscar repetidamente por values puede requerir un scan; si el caso de uso dominante es inverso, quizá necesitas otro índice derivado.

### 4.5 `.keys`, `.values` y `.items`

**Ejemplo ejecutable:**

```python
status_by_id = {
    "evt-001": "draft",
    "evt-002": "accepted",
}

print(list(status_by_id.keys()))
# ['evt-001', 'evt-002']
print(list(status_by_id.values()))
# ['draft', 'accepted']
print(list(status_by_id.items()))
# [('evt-001', 'draft'), ('evt-002', 'accepted')]
```

Estas llamadas devuelven vistas dinámicas del dict, no snapshots list independientes. Aquí se materializan con `list(...)` solo para mostrar el output. Si el dict cambia, una vista ya creada refleja el cambio.

**Ejemplo ejecutable — vista dinámica:**

```python
status_by_id = {"evt-001": "draft"}
keys_view = status_by_id.keys()

status_by_id["evt-002"] = "accepted"

print(list(keys_view))
# ['evt-001', 'evt-002']
```

### 4.6 Iteración

Iterar un dict directamente recorre sus keys. Usa `.items()` cuando necesitas ambos componentes.

**Ejemplo ejecutable:**

```python
status_by_id = {
    "evt-001": "draft",
    "evt-002": "accepted",
}

for event_id in status_by_id:
    print(event_id)

for event_id, status in status_by_id.items():
    print(event_id, status)
```

Output:

```text
evt-001
evt-002
evt-001 draft
evt-002 accepted
```

### 4.7 Orden de inserción: contrato práctico

En Python moderno, un dict conserva el orden de inserción. Actualizar el value de una key existente no la mueve; eliminarla y volverla a insertar la coloca al final.

**Ejemplo ejecutable:**

```python
status_by_id = {}
status_by_id["evt-002"] = "draft"
status_by_id["evt-001"] = "accepted"
status_by_id["evt-002"] = "accepted"

print(list(status_by_id))
# ['evt-002', 'evt-001']
```

El orden de inserción no equivale a orden temporal, alfabético ni prioridad. Si el contrato exige otro orden, ordénalo de forma explícita o usa datos que expresen esa dimensión.

### 4.8 Keys hashable

Un dict necesita localizar una key mediante su hash y confirmar igualdad. Por eso las keys deben ser **hashable**: ofrecen un hash estable durante su uso y compatible con la igualdad. Si `a == b`, entonces debe cumplirse `hash(a) == hash(b)`; compartir hash no implica por sí solo que dos objetos sean iguales.

En este nivel:

- `str`, `int` y tuples compuestas solo por elementos hashable suelen servir como keys;
- `list`, `dict` y `set` son mutables y no son hashable;
- una tuple que contiene una list tampoco es hashable.

**Ejemplo ejecutable:**

```python
events_by_key = {
    ("person-007", "evt-001"): "Llegada",
}

print(events_by_key[("person-007", "evt-001")])
# Llegada
```

**Failure case — list como key:**

```python
events_by_key = {}
events_by_key[["person-007", "evt-001"]] = "Llegada"  # TypeError: unhashable type
```

Antes de ejecutar, predice el tipo de error. Corrige la key solo si existe una representación hashable con la misma semántica, como la tuple del ejemplo anterior; no conviertas una list a tuple de manera automática. Conserva el tipo y el valor de la key propuesta.

No necesitas implementar tablas hash ni estudiar colisiones en PF-M3. El modelo suficiente es: el hash permite localizar una zona candidata y la igualdad confirma la key. **CS-M2** profundizará estructuras y costos.

### 4.9 Lookup conceptual y costo aproximado

En condiciones normales, consultar, insertar o comprobar membership por key en un dict suele ser directo y barato en promedio. No implica recorrer todas las keys como una list. Sin embargo:

- construir el índice cuesta tiempo y memoria;
- el costo “promedio” no es una promesa de tiempo fijo universal;
- la calidad y estabilidad de las keys importan;
- para una sola consulta sobre tres elementos, el loop más claro puede ganar.

La decisión profesional compara el costo de construir/mantener el índice con la frecuencia de consultas.

### 4.10 Diccionarios anidados

**Ejemplo ejecutable:**

```python
event = {
    "id": "evt-001",
    "entity": {
        "id": "person-007",
        "display_name": "Ada",
    },
    "tags": ["viaje", "familia"],
}

print(event["entity"]["display_name"])
# Ada
```

Cada nivel tiene su propio contrato. El anidamiento no elimina `KeyError`, aliasing ni mutabilidad. Si dos eventos comparten el mismo dict interior, mutarlo desde uno afecta al otro.

### 4.11 EIDOLON: índice derivado, no source of truth

**Ejemplo ejecutable:**

```python
source_events = [
    {"id": "evt-001", "text": "Llegada"},
    {"id": "evt-002", "text": "Reunión"},
]

events_by_id = {}
for event in source_events:
    events_by_id[event["id"]] = event

print(events_by_id["evt-002"]["text"])
# Reunión
```

`events_by_id` es un **índice derivado en memoria**: acelera el lookup y puede reconstruirse desde `source_events`. No es una fuente persistente ni autoriza a descartar datos originales. En EIDOLON futuro, la persistencia tendrá contratos propios; PF-M6 introducirá archivos/JSON y fases posteriores tratarán bases de datos.

Este ejemplo también comparte las referencias a cada event. Si el contrato exige no mutar acontecimientos fuente, las funciones deben tratarlos como read-only y construir resultados derivados nuevos. Que una operación sea técnicamente posible no la vuelve válida para el dominio.

### Predice

```python
status_by_id = {"evt-001": None}

print(status_by_id.get("evt-001"))
print(status_by_id.get("evt-999"))
print("evt-001" in status_by_id)
print("evt-999" in status_by_id)
```

Predice los cuatro outputs y explica cuál operación distingue presencia de ausencia.

### Modifica

Construye `events_by_id` desde una list de tres events mediante un loop. Decide una política explícita para IDs duplicados: conservar el primero o conservar el último. Provoca un duplicado y demuestra tu política con un `assert`.

### Explica

¿Por qué el orden de inserción de un dict no convierte sus keys en una timeline confiable? ¿Qué dato debería expresar el orden temporal?

### Detecta el bug

```python
events_by_id = {
    "evt-001": {"status": "draft"},
    "evt-002": {"status": "accepted"},
}

if "accepted" in events_by_id:
    print("Hay un evento aceptado")
```

No imprime nada porque membership examina keys. Corrige el objetivo inmediato con `.values()` y luego explica cuándo convendría un índice separado por status.

---

## 5. `set` y `frozenset`: pertenencia y unicidad

### 5.1 El problema de los duplicados silenciosos

Una list puede deduplicarse manualmente, pero no expresa unicidad por sí misma. Un `set` almacena elementos únicos y está diseñado para membership y operaciones de conjuntos.

**Ejemplo ejecutable:**

```python
seen_source_ids = {"src-001", "src-002"}
seen_source_ids.add("src-002")
seen_source_ids.add("src-003")

print(len(seen_source_ids))
# 3
print("src-003" in seen_source_ids)
# True
```

Añadir un elemento igual a uno existente no crea un duplicado.

### 5.2 Creación: el set vacío

`{}` crea un dict vacío. Usa `set()` para un set vacío.

**Ejemplo ejecutable:**

```python
empty_set = set()
empty_dict = {}

print(type(empty_set).__name__)
# set
print(type(empty_dict).__name__)
# dict
```

### 5.3 El orden no es el contrato

Un set no ofrece acceso por índice ni promete un orden de iteración útil para el dominio. No fijes como output su representación directa. Si necesitas presentar sus elementos de forma determinista, crea una vista ordenada:

**Ejemplo ejecutable:**

```python
active_tags = {"viaje", "familia", "trabajo"}

print(sorted(active_tags))
# ['familia', 'trabajo', 'viaje']
```

`sorted` devuelve una list nueva; el set original sigue siendo un conjunto sin orden de dominio.

### 5.4 Operaciones de conjuntos

**Ejemplo ejecutable:**

```python
event_a_tags = {"familia", "viaje", "prioritario"}
event_b_tags = {"viaje", "trabajo"}

print(sorted(event_a_tags | event_b_tags))
# ['familia', 'prioritario', 'trabajo', 'viaje']
print(sorted(event_a_tags & event_b_tags))
# ['viaje']
print(sorted(event_a_tags - event_b_tags))
# ['familia', 'prioritario']
```

- union (`|` o `.union`) reúne elementos de ambos;
- intersection (`&` o `.intersection`) conserva los comunes;
- difference (`-` o `.difference`) conserva los del lado izquierdo que no están en el derecho.

### 5.5 Subset y superset

**Ejemplo ejecutable:**

```python
required_tags = {"viaje", "familia"}
event_tags = {"viaje", "familia", "prioritario"}

print(required_tags <= event_tags)
# True
print(event_tags >= required_tags)
# True
print(required_tags < event_tags)
# True
```

`<=` pregunta subset; `>=`, superset. Las versiones estrictas `<` y `>` además exigen que los conjuntos no sean iguales.

### 5.6 Elementos hashable

Al igual que las keys de dict, los elementos de un set deben ser hashable.

**Failure case — list dentro de set:**

```python
tag_groups = {["viaje", "familia"]}  # TypeError: unhashable type
```

Antes de ejecutar, predice el tipo de error. Decide si el grupo debe ser una tuple ordenada o un `frozenset` sin orden; la corrección depende del contrato, no solo de hacer el valor hashable. Conserva el elemento rechazado y la relación que pretendía representar.

Una tuple de strings sí puede ser elemento, aunque debes preguntarte si el par representa realmente una unidad con igualdad significativa.

### 5.7 `frozenset`

`frozenset` representa un conjunto que no puede mutarse después de crearse. Mantiene unicidad y operaciones de conjuntos; no tiene `.add` ni `.remove`. Al ser hashable cuando sus elementos lo permiten, puede usarse como key o como elemento de otro set.

**Ejemplo ejecutable:**

```python
tag_signature = frozenset(["viaje", "familia", "viaje"])
descriptions = {tag_signature: "evento familiar de viaje"}

print(len(tag_signature))
# 2
print(descriptions[frozenset(["familia", "viaje"])])
# evento familiar de viaje
```

El orden de entrada no afecta la igualdad del conjunto.

### 5.8 `set` frente a `list`

| Pregunta | `list` | `set` |
|---|---|---|
| ¿conserva orden y posición? | sí | no como contrato |
| ¿permite duplicados? | sí | no |
| ¿acceso por índice? | sí | no |
| ¿membership repetido? | scan lineal | normalmente directo y barato en promedio |
| ¿elementos no hashable? | sí | no |

No conviertas toda list a set “por velocidad”. Perder duplicados u orden puede destruir información. Convierte solo si la semántica del resultado es un conjunto.

### 5.9 EIDOLON: tags, IDs vistos y deduplicación

**Ejemplo ejecutable:**

```python
events = [
    {"id": "evt-001", "tags": ["viaje", "familia"]},
    {"id": "evt-002", "tags": ["trabajo", "viaje"]},
]

active_tags = set()
seen_source_ids = set()

for event in events:
    seen_source_ids.add(event["id"])
    active_tags.update(event["tags"])

print(sorted(seen_source_ids))
# ['evt-001', 'evt-002']
print(sorted(active_tags))
# ['familia', 'trabajo', 'viaje']
```

`update` añade todos los elementos de un iterable. Los sets son derivados útiles; la list de tags original puede conservar orden, repeticiones o evidencia que un set descartaría.

### Predice

```python
left = {"evt-001", "evt-002"}
right = {"evt-002", "evt-003"}

print(sorted(left & right))
print(sorted(left - right))
print(left <= right)
```

### Modifica

Agrega una tercera colección de tags y calcula los tags presentes en cualquiera de las tres y los presentes en las tres. Muestra outputs deterministas.

### Explica

¿Por qué deduplicar una timeline con `set(timeline)` puede ser pérdida de información aunque todos los IDs repetidos parezcan accidentales?

### Detecta el bug

```python
seen_source_ids = []

for source_id in ["src-001", "src-002", "src-001"]:
    if source_id not in seen_source_ids:
        seen_source_ids.append(source_id)
```

El resultado es único, pero cada membership puede escanear la list. Sustituye la estructura de membership por un set. Si también necesitas conservar el primer orden observado, justifica mantener además una list; cada colección cumpliría una responsabilidad distinta.

---

## 6. Elegir una colección por el contrato

### 6.1 Razonamiento antes que tabla

Imagina cuatro necesidades:

1. **Timeline pequeña:** importa el orden y dos eventos podrían tener el mismo texto. La operación principal es recorrer. Una `list` expresa el contrato.
2. **Referencia `(event_id, person_id)`:** hay dos posiciones fijas con roles estables. Una `tuple` puede expresar el par.
3. **Buscar una entidad por ID muchas veces:** domina el lookup por key. Un `dict` expresa `entity_id -> entity`.
4. **Detectar IDs ya vistos:** domina membership y la unicidad es una invariante. Un `set` expresa esa pertenencia.

La pregunta no es “¿qué tipo es más rápido?”. Es “¿qué propiedades necesito preservar y qué operaciones dominan?”. El costo solo decide entre alternativas que respetan la semántica.

### 6.2 Tabla de decisión

| Necesidad | Colección probable | Razón principal | Advertencia |
|---|---|---|---|
| secuencia ordenada, mutable y con duplicados | `list` | posición y recorrido | lookup por ID repetido exige scan |
| estructura posicional fija | `tuple` | contrato de posiciones estable | contenido mutable puede seguir mutando |
| key → value | `dict` | lookup y asociación explícita | keys únicas y hashable |
| pertenencia y unicidad mutable | `set` | membership y operaciones de conjuntos | no conserva posición ni duplicados |
| pertenencia y unicidad inmutable | `frozenset` | conjunto estable y hashable | no permite `.add` |
| resultado completo para reutilizar | colección materializada | múltiples recorridos | ocupa memoria proporcional al resultado |
| pipeline de una pasada | iterator/generator | produce bajo demanda | se consume y puede fallar tarde |

Esta tabla resume decisiones; no sustituye el razonamiento.

### 6.3 Una estructura puede complementar a otra

No siempre eliges una sola. Una list puede conservar el orden fuente y un dict puede acelerar lookups:

```text
source_events: list en orden de entrada
events_by_id: dict derivado para lookup
seen_source_ids: set derivado para duplicados
```

La condición profesional es mantener claras las responsabilidades. Si los derivados divergen de la fuente, debes reconstruirlos o actualizarlos bajo una política explícita. La persistencia y replay completos llegan después.

### 6.4 Cuándo no crear un índice

Construir un dict para buscar una vez entre cinco elementos puede añadir estado y complejidad sin beneficio. Crea un índice cuando:

- las consultas son repetidas;
- la key representa una identidad estable;
- la política de duplicados está definida;
- puedes reconstruir o mantener el índice sin volverlo una segunda fuente contradictoria.

### Predice

Elige una colección para cada contrato y escribe una frase de justificación: a) orden de pasos con repetición; b) roles de usuario únicos; c) configuración por nombre; d) coordenada fija `(x, y)`.

### Modifica

Toma una list de events y añade dos derivados: `events_by_id` para consulta y `active_tags` para pertenencia. Conserva la list sin mutarla.

### Explica

¿Qué costo de consistencia aparece cuando mantienes una list fuente y dos índices derivados? ¿Por qué “derivado” es una propiedad arquitectónica y no solo un nombre de variable?

### Detecta el bug

```python
events = [
    {"id": "evt-001", "text": "A"},
    {"id": "evt-002", "text": "B"},
]

for requested_id in ["evt-002", "evt-001", "evt-002"]:
    for event in events:
        if event["id"] == requested_id:
            print(event["text"])
```

El resultado es correcto, pero el lookup lineal se repite. Construye una vez `events_by_id` y úsalo para las consultas. Explica cuándo el código original seguiría siendo aceptable.

---

## 7. Iteración: iterable, iterator y consumo

### 7.1 `for` responde “dame el siguiente”

Un `for` no necesita saber si los elementos están en list, tuple, dict, set, range o se producen bajo demanda. Necesita un **iterable**: un objeto del que puede obtener un iterator.

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002"]

for event_id in event_ids:
    print(event_id)
```

Output:

```text
evt-001
evt-002
```

Conceptualmente, `for`:

1. llama `iter(iterable)` para obtener un iterator;
2. llama repetidamente `next(iterator)`;
3. asigna cada elemento al nombre del loop;
4. termina cuando `next` señala `StopIteration`.

No necesitas implementar el protocolo en una clase dentro de PF-M3.

### 7.2 `iter`, `next` y `StopIteration`

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002"]
iterator = iter(event_ids)

print(next(iterator))
# evt-001
print(next(iterator))
# evt-002
```

Una tercera llamada provoca `StopIteration`.

**Failure case — iterator agotado:**

```python
iterator = iter(["evt-001"])
print(next(iterator))
print(next(iterator))  # StopIteration
```

Antes de ejecutar, predice qué llamada falla. Si necesitas otra pasada, crea un iterator nuevo desde un iterable reutilizable o materializa deliberadamente; no intentes “reiniciar” el objeto agotado. Conserva la fuente y cuántos elementos se consumieron.

Normalmente no capturas `StopIteration` al escribir un `for`; el loop aplica ese contrato por ti.

### 7.3 Iterable no es lo mismo que iterator

- Un **iterable** puede producir un iterator. Una list es iterable y puede recorrerse de nuevo.
- Un **iterator** conserva una posición de consumo y entrega el siguiente elemento. `iter(list)` devuelve uno.
- `iter(iterator) is iterator` suele ser `True`: el iterator se entrega a sí mismo y no reinicia su estado.

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002"]
iterator = iter(event_ids)

print(list(iterator))
# ['evt-001', 'evt-002']
print(list(iterator))
# []
print(list(event_ids))
# ['evt-001', 'evt-002']
```

`list(iterator)` consumió lo restante. La list original sigue siendo iterable y reutilizable.

### 7.4 Modificar una colección durante su iteración

Cambiar la estructura recorrida puede saltar elementos, repetir trabajo o producir un error según el tipo de colección.

**Código incorrecto — eliminar de una list mientras se recorre:**

```python
event_ids = ["evt-001", "drop-001", "drop-002", "evt-002"]

for event_id in event_ids:
    if event_id.startswith("drop-"):
        event_ids.remove(event_id)

print(event_ids)
# ['evt-001', 'drop-002', 'evt-002']
```

Al eliminar `drop-001`, los elementos se desplazan y el loop avanza; `drop-002` queda saltado. La corrección más clara es construir una colección nueva:

**Ejemplo ejecutable — reconstrucción:**

```python
event_ids = ["evt-001", "drop-001", "drop-002", "evt-002"]
kept_ids = []

for event_id in event_ids:
    if not event_id.startswith("drop-"):
        kept_ids.append(event_id)

print(kept_ids)
# ['evt-001', 'evt-002']
```

Iterar sobre una copia puede ser válido si necesitas mutar la original, pero separar fuente y resultado suele hacer el contrato más visible.

**Failure case — cambiar tamaño de dict durante iteración:**

```python
status_by_id = {"evt-001": "draft", "evt-002": "accepted"}

for event_id in status_by_id:
    status_by_id[event_id + "-copy"] = "derived"  # RuntimeError
```

Conserva el estado inicial y la operación que produjo el fallo para diagnosticarlo; no suprimas el error y continúes con un índice parcial.

### 7.5 Side effects del cuerpo

Un loop explícito puede contener efectos, pero PF-M2 exige que sean visibles y deliberados. Para transformaciones de datos, prefiere devolver una colección nueva. Para I/O futuro, separa la obtención de datos de la regla que los filtra.

### Predice

```python
iterator = iter([10, 20, 30])
print(next(iterator))
print(list(iterator))
print(list(iterator))
```

Predice los tres outputs y dibuja la posición del iterator después de cada operación.

### Modifica

Reescribe el ejemplo de eliminación para conservar solo IDs que empiezan con `"evt-"`, sin mutar la list original. Comprueba ambas listas.

### Explica

¿Por qué una list es iterable pero el iterator obtenido de ella no es una “segunda list”? Incluye estado de consumo y reutilización.

### Detecta el bug

```python
events = iter(["evt-001", "evt-002"])
first_pass = list(events)
second_pass = list(events)
```

Si el contrato esperaba dos listas iguales, el bug es reutilizar un iterator consumido. Conserva el iterable reutilizable o crea un iterator nuevo por pasada.

---

## 8. `range`: una progresión compacta

### 8.1 Qué representa

`range` representa una secuencia inmutable de enteros definida por límites. No construye una list con todos ellos. Calcula los valores según se necesitan y puede recorrerse más de una vez porque es iterable, no un iterator consumible por sí solo.

Formas principales:

```text
range(stop)
range(start, stop)
range(start, stop, step)
```

`stop` se excluye.

### 8.2 Ejemplos

**Ejemplo ejecutable:**

```python
print(list(range(4)))
# [0, 1, 2, 3]
print(list(range(2, 6)))
# [2, 3, 4, 5]
print(list(range(10, 4, -2)))
# [10, 8, 6]
```

`list(...)` se usa para observar los valores; el objeto `range` original no era esa list.

### 8.3 Límites y step

Un step positivo no produce valores si `start >= stop`; uno negativo no produce valores si `start <= stop`. `step=0` es inválido.

**Ejemplo ejecutable:**

```python
print(list(range(5, 2)))
# []
print(list(range(2, 5, -1)))
# []
```

**Failure case — step cero:**

```python
numbers = range(0, 10, 0)  # ValueError
```

Antes de ejecutar, predice el tipo de error. Un step cero nunca avanza hacia `stop`; elige un step no nulo cuya dirección coincida con los límites. Para diagnosticar conserva `start`, `stop` y `step`.

### 8.4 Cuándo no usarlo

No uses `range(len(events))` solo para recuperar cada elemento por índice. Si necesitas los elementos, itera directamente. Si necesitas posición y elemento, usa `enumerate`.

### Predice

Predice `list(range(7, 1, -3))` y explica por qué `1` no aparece.

### Modifica

Escribe un `range` que produzca `2, 5, 8, 11` y otro que produzca `5, 3, 1`.

### Explica

¿Por qué “no es una list materializada” no significa “es un iterator de una sola pasada”?

### Detecta el bug

```python
event_ids = ["evt-001", "evt-002", "evt-003"]

for index in range(1, len(event_ids)):
    print(event_ids[index])
```

Si se pretendía imprimir todos los IDs, se omitió el índice `0`. Corrige el límite o, mejor, itera los valores directamente.

---

## 9. `enumerate`: posición y elemento sin contador manual

### 9.1 El problema del contador separado

**Código incorrecto — estado manual innecesario:**

```python
event_ids = ["evt-001", "evt-002", "evt-003"]
position = 0

for event_id in event_ids:
    print(position, event_id)
    position += 1
```

Funciona, pero `position` es estado mutable separado que puede olvidarse o actualizarse en la rama equivocada.

### 9.2 Solución con `enumerate`

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002", "evt-003"]

for position, event_id in enumerate(event_ids, start=1):
    print(position, event_id)
```

Output:

```text
1 evt-001
2 evt-002
3 evt-003
```

`enumerate` produce pares `(index, value)` bajo demanda. `start` cambia la etiqueta inicial, no la posición real dentro de la list.

### 9.3 Cuándo no usarlo

Si la posición no participa en el contrato, no la agregues. `for event in events` comunica mejor que el índice carece de importancia.

### Predice

¿Qué imprime `enumerate(["a", "b"], start=10)`? ¿Qué valor sigue ocupando la posición `0` de la list?

### Modifica

Numera una timeline desde `1`, pero imprime solo eventos cuyo ID termina en un número impar. Explica si la numeración representa posición original o contador de resultados filtrados.

### Explica

¿Qué estado elimina `enumerate` frente a un contador manual y qué bug plausible evita?

### Detecta el bug

```python
events = ["evt-001", "evt-002"]

for event_id, position in enumerate(events, start=1):
    print(position, event_id)
```

Los nombres están invertidos respecto al par `(index, value)`. El código corre, pero comunica y produce datos equivocados.

---

## 10. `zip`: iteración paralela y contratos de longitud

### 10.1 Iterar dos fuentes en paralelo

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002"]
statuses = ["draft", "accepted"]

for event_id, status in zip(event_ids, statuses):
    print(event_id, status)
```

Output:

```text
evt-001 draft
evt-002 accepted
```

`zip` produce tuples con un elemento de cada iterable.

### 10.2 Truncamiento predeterminado

Por default, `zip` termina cuando se agota el iterable más corto.

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002", "evt-003"]
statuses = ["draft", "accepted"]

pairs = list(zip(event_ids, statuses))

print(pairs)
# [('evt-001', 'draft'), ('evt-002', 'accepted')]
```

`"evt-003"` desapareció del resultado sin error. A veces ese truncamiento es el contrato correcto; muchas veces oculta datos faltantes.

### 10.3 `strict=True`

Cuando longitudes distintas indican un bug, usa `strict=True`.

**Failure case — longitudes distintas:**

```python
event_ids = ["evt-001", "evt-002", "evt-003"]
statuses = ["draft", "accepted"]

print(list(zip(event_ids, statuses, strict=True)))  # ValueError
```

Antes de ejecutar, predice cuándo se detecta el mismatch. Corrige la fuente que perdió un elemento o declara explícitamente que el truncamiento es válido; no elimines `strict=True` solo para silenciar el síntoma. Conserva ambos inputs y los pares ya consumidos.

El error puede aparecer después de haber producido pares previos si consumes el zip elemento por elemento. Esto anticipa la idea de error tardío en procesamiento lazy.

### 10.4 `zip` también es un iterator

**Ejemplo ejecutable:**

```python
pairs = zip(["evt-001", "evt-002"], ["draft", "accepted"])

print(list(pairs))
# [('evt-001', 'draft'), ('evt-002', 'accepted')]
print(list(pairs))
# []
```

No asumas que `zip` exige longitudes iguales. Sin `strict=True`, su contrato es truncar.

### Predice

Predice `list(zip([1, 2, 3], ["a"], [True, False]))` sin ejecutar. Identifica cuál iterable determina la longitud.

### Modifica

Convierte dos listas iguales de `event_ids` y `statuses` en un dict mediante `dict(zip(..., strict=True))`. Explica qué ocurriría con IDs duplicados.

### Explica

¿Cuándo es útil el truncamiento y cuándo debe considerarse pérdida silenciosa?

### Detecta el bug

```python
event_ids = ["evt-001", "evt-002"]
statuses = ["draft"]

for event_id, status in zip(event_ids, statuses):
    print(event_id, status)
```

Si cada event necesita status, el código debe usar `strict=True`. El bug no es creer que `zip` siempre falla; es asumir una validación que no se solicitó.

---

## 11. Transformaciones explícitas antes de comprehensions

### 11.1 Separar selección, transformación y acumulación

Una transformación común tiene tres decisiones:

1. de dónde salen los elementos;
2. cuáles pasan un filtro;
3. qué valor se agrega al resultado.

**Ejemplo ejecutable — loop explícito:**

```python
events = [
    {"id": "evt-001", "active": True},
    {"id": "evt-002", "active": False},
    {"id": "evt-003", "active": True},
]

active_event_ids = []

for event in events:
    if event["active"]:
        active_event_ids.append(event["id"])

print(active_event_ids)
# ['evt-001', 'evt-003']
```

Antes de compactar este código, debes poder explicarlo. El loop explícito es la referencia de claridad y la forma correcta cuando hay varias ramas, efectos o estados intermedios significativos.

### 11.2 Funciones nombradas para condiciones de dominio

PF-M2 enseñó a nombrar reglas. Si una condición es compleja, extráela antes de considerar una comprehension.

**Ejemplo ejecutable:**

```python
def is_active_for_person(event, person_id):
    return event["active"] and event["person_id"] == person_id


events = [
    {"id": "evt-001", "person_id": "person-007", "active": True},
    {"id": "evt-002", "person_id": "person-008", "active": True},
    {"id": "evt-003", "person_id": "person-007", "active": False},
]

events_for_person = []
for event in events:
    if is_active_for_person(event, "person-007"):
        events_for_person.append(event)

print([event["id"] for event in events_for_person])
# ['evt-001']
```

La última línea usa una list comprehension pequeña solo para mostrar IDs; la siguiente sección la desarrolla.

### Predice

Añade un cuarto evento activo para `person-007`. Predice el orden del resultado y explica de dónde proviene.

### Modifica

Extiende el loop para producir etiquetas `"event:<id>"` de eventos activos, sin comprehension todavía.

### Explica

Señala las líneas que representan fuente, filtro, transformación y acumulación.

### Detecta el bug

**Fragmento para diagnosticar — reutiliza `events` del ejemplo ejecutable anterior:**

```python
active_event_ids = []

for event in events:
    active_event_ids.append(event["id"])
    if not event["active"]:
        active_event_ids.remove(event["id"])
```

Aunque puede producir el resultado esperado en casos simples, acumula antes de decidir. Mueve la mutación dentro de la rama positiva para que el control flow exprese el contrato.

---

## 12. Comprehensions legibles

### 12.1 De loop a transformación sencilla

Una list comprehension crea una list nueva al evaluar una expresión por cada elemento.

**Ejemplo ejecutable — equivalencia:**

```python
event_ids = ["evt-001", "evt-002", "evt-003"]

labels_loop = []
for event_id in event_ids:
    labels_loop.append("event:" + event_id)

labels_comprehension = ["event:" + event_id for event_id in event_ids]

print(labels_loop)
# ['event:evt-001', 'event:evt-002', 'event:evt-003']
print(labels_loop == labels_comprehension)
# True
```

Léela de izquierda a derecha:

```text
[expresión por cada elemento en iterable]
```

### 12.2 Agregar un filtro

**Ejemplo ejecutable:**

```python
events = [
    {"id": "evt-001", "active": True},
    {"id": "evt-002", "active": False},
    {"id": "evt-003", "active": True},
]

active_events = [event for event in events if event["active"]]

print([event["id"] for event in active_events])
# ['evt-001', 'evt-003']
```

Ahora la forma es:

```text
[expresión por cada elemento en iterable si condición]
```

### 12.3 Transformación + filtro

**Ejemplo ejecutable:**

```python
events = [
    {"id": "evt-001", "active": True},
    {"id": "evt-002", "active": False},
    {"id": "evt-003", "active": True},
]

active_event_ids = [event["id"] for event in events if event["active"]]

print(active_event_ids)
# ['evt-001', 'evt-003']
```

La expresión `event["id"]` transforma; el `if` final filtra.

### 12.4 Set comprehension

Una set comprehension materializa elementos únicos.

**Ejemplo ejecutable:**

```python
events = [
    {"id": "evt-001", "person_id": "person-007"},
    {"id": "evt-002", "person_id": "person-008"},
    {"id": "evt-003", "person_id": "person-007"},
]

person_ids = {event["person_id"] for event in events}

print(sorted(person_ids))
# ['person-007', 'person-008']
```

### 12.5 Dict comprehension

Una dict comprehension necesita expresión de key y de value.

**Ejemplo ejecutable:**

```python
events = [
    {"id": "evt-001", "status": "draft"},
    {"id": "evt-002", "status": "accepted"},
]

status_by_id = {event["id"]: event["status"] for event in events}

print(status_by_id)
# {'evt-001': 'draft', 'evt-002': 'accepted'}
```

Si dos elementos producen la misma key, el último value reemplaza al anterior. Esa política puede ser válida o puede ocultar un duplicate ID. Una comprehension no ofrece un lugar claro para detectar el duplicado; usa un loop explícito cuando la política necesita diagnóstico.

### 12.6 Cuándo mejora la claridad

Una comprehension suele ayudar cuando:

- produce una colección nueva;
- tiene una transformación pequeña;
- tiene cero o un filtro legible;
- no produce efectos externos;
- su política de duplicados es segura y evidente.

### 12.7 Cuándo la destruye

Vuelve a loops o funciones nombradas cuando:

- hay varias ramas o condiciones de dominio;
- necesitas registrar por qué un elemento fue rechazado;
- debes detectar duplicados y conservar evidencia;
- la expresión requiere releerla para descubrir el orden;
- combina mutación externa, llamadas con efectos o I/O;
- oculta trabajo costoso dentro de cada elemento.

**Fragmento — comprehension sintácticamente válida cuya densidad oculta el contrato:**

```python
result = [
    event["id"].upper()
    for event in events
    if event.get("active") and "private" not in event.get("tags", []) and event.get("score", 0) > 0.7
]
```

Puede ser sintácticamente válida, pero mezcla varias reglas y defaults implícitos. Nombra el predicado, decide el contrato de keys ausentes y luego conserva una comprehension corta si todavía ayuda.

### Predice

```python
numbers = [1, 2, 3, 4, 5]
result = [number * 10 for number in numbers if number % 2 == 1]
print(result)
```

Identifica primero el filtro y después la transformación.

### Modifica

Escribe el ejemplo anterior como loop explícito. Después cambia ambos para producir un set de resultados y explica qué propiedad nueva aparece.

### Explica

¿Por qué una dict comprehension puede ocultar duplicate IDs aunque su output parezca correcto?

### Detecta el bug

```python
events = [
    {"id": "evt-001", "status": "draft"},
    {"id": "evt-001", "status": "accepted"},
]

events_by_id = {event["id"]: event for event in events}
```

Si el contrato exige rechazar duplicados, la comprehension es insuficiente: sobrescribe. Reescribe con un loop, un set de IDs vistos y una política observable.

---

## 13. Nested comprehensions: solo cuando el orden sigue visible

### 13.1 Aplanar una estructura pequeña

**Ejemplo ejecutable — primero loops:**

```python
event_tag_groups = [
    ["viaje", "familia"],
    ["trabajo"],
]

all_tags_loop = []
for tag_group in event_tag_groups:
    for tag in tag_group:
        all_tags_loop.append(tag)

all_tags_comprehension = [
    tag
    for tag_group in event_tag_groups
    for tag in tag_group
]

print(all_tags_loop)
# ['viaje', 'familia', 'trabajo']
print(all_tags_loop == all_tags_comprehension)
# True
```

El orden de los `for` en la comprehension coincide con el orden de los loops explícitos: primero cada grupo; dentro, cada tag.

### 13.2 Dedupe al aplanar

**Ejemplo ejecutable:**

```python
event_tag_groups = [
    ["viaje", "familia"],
    ["trabajo", "viaje"],
]

active_tags = {
    tag
    for tag_group in event_tag_groups
    for tag in tag_group
}

print(sorted(active_tags))
# ['familia', 'trabajo', 'viaje']
```

### 13.3 Límite de legibilidad

Una nested comprehension deja de ayudar cuando agrega filtros en varios niveles, transformaciones largas o más nesting. Si necesitas simular mentalmente el control flow, vuelve al loop y nombra estados intermedios.

**Fragmento — comprehension sintácticamente válida con demasiadas decisiones compactadas:**

```python
result = [
    tag.strip().lower()
    for event in events
    if event.get("active")
    for tag in event.get("tags", [])
    if tag.strip() and not tag.startswith("internal:")
]
```

Separa al menos la política de validez/normalización de tags en una función de PF-M2. Si también necesitas informar tags rechazadas, usa loops explícitos.

### Predice

Predice el orden de `[letter for word in ["ab", "cd"] for letter in word]`. Reescríbelo como dos loops antes de responder.

### Modifica

Convierte el aplanamiento de tags en set comprehension y luego en loops explícitos con `set.add`. Comprueba igualdad, no orden.

### Explica

¿Qué señal concreta te haría abandonar una nested comprehension aunque ocupe menos líneas?

### Detecta el bug

```python
event_tag_groups = [["viaje"], ["familia"]]
wrong = [tag_group for tag in tag_group for tag_group in event_tag_groups]
```

Los `for` están invertidos y `tag_group` se usa antes de enlazarse en ese scope. Deriva el orden desde los loops explícitos.

---

## 14. Side effects en comprehensions

### 14.1 Una comprehension debe construir un valor

La intención natural de una comprehension es producir una colección. Usarla para modificar una estructura externa, imprimir o ejecutar I/O mezcla resultado con efectos.

**Código incorrecto — mutación externa:**

```python
seen_ids = set()
events = [{"id": "evt-001"}, {"id": "evt-002"}]

results = [seen_ids.add(event["id"]) for event in events]

print(results)
# [None, None]
```

`set.add` muta y devuelve `None`, por eso se materializa una list inútil. El efecto queda escondido dentro de una forma que promete transformación.

**Ejemplo ejecutable — intención explícita:**

```python
seen_ids = set()
events = [{"id": "evt-001"}, {"id": "evt-002"}]

for event in events:
    seen_ids.add(event["id"])

print(sorted(seen_ids))
# ['evt-001', 'evt-002']
```

Si solo necesitas el resultado y no una mutación incremental externa, una set comprehension es aún más directa:

**Ejemplo ejecutable — transformación pura:**

```python
events = [{"id": "evt-001"}, {"id": "evt-002"}]
seen_ids = {event["id"] for event in events}

print(sorted(seen_ids))
# ['evt-001', 'evt-002']
```

### 14.2 `print` e I/O

**Código incorrecto — imprimir por side effect:**

```python
event_ids = ["evt-001", "evt-002"]
printed = [print(event_id) for event_id in event_ids]

print(printed)
# [None, None]
```

Usa un loop si imprimir es realmente la responsabilidad del borde. Para archivos, red o bases de datos futuras, la razón es aún más fuerte: los efectos pueden fallar a mitad del consumo y dejar estado parcial.

### 14.3 Conexión con funciones puras de PF-M2

Una comprehension con una expresión y un predicado puros es fácil de razonar: el resultado depende de la entrada. Una llamada impura dentro de ella conserva su impureza; la sintaxis compacta no convierte el efecto en transformación.

### Predice

Predice el valor de `[set().add(number) for number in [1, 2]]` y explica por qué cada set creado se pierde.

### Modifica

Reescribe `[print(event["id"]) for event in events]` como una función impura `print_event_ids(events)` con loop explícito y retorno `None` deliberado.

### Explica

¿Por qué la list de `None` es una pista de que se usó una comprehension para efectos?

### Detecta el bug

```python
events_by_id = {}
events = [{"id": "evt-001"}, {"id": "evt-002"}]

discarded = [events_by_id.update({event["id"]: event}) for event in events]
```

La mutación funciona, pero el nombre `discarded` recibe una list de `None`. Usa un loop para política de duplicados o una dict comprehension solo si sobrescribir está permitido.

---

## 15. Eager y lazy: materializar o producir bajo demanda

### 15.1 Materialización eager

Una list comprehension es **eager**: calcula y almacena todos sus resultados al crear la list.

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002", "evt-003"]
labels = ["event:" + event_id for event_id in event_ids]

print(labels)
# ['event:evt-001', 'event:evt-002', 'event:evt-003']
```

Ventajas:

- puedes recorrer el resultado varias veces;
- puedes usar `len`, índices y slicing;
- los errores de transformación ocurren durante la construcción;
- para datos pequeños, el contrato suele ser directo.

Costo: memoria proporcional a los resultados materializados, además del trabajo completo aunque solo consumas el primero.

### 15.2 Procesamiento lazy

Un procesamiento **lazy** produce elementos cuando el consumidor los solicita. Puede reducir memoria y evitar trabajo innecesario, pero introduce:

- consumo de una sola pasada para iterators/generators;
- errores que pueden aparecer tarde;
- ausencia de `len` o índice general;
- necesidad de mantener válidas las dependencias durante el consumo.

Lazy no significa automáticamente más rápido. Cambia cuándo ocurre el trabajo y cuánta memoria se retiene.

### 15.3 Una sola pasada

**Ejemplo ejecutable:**

```python
labels = ("event:" + event_id for event_id in ["evt-001", "evt-002"])

print(next(labels))
# event:evt-001
print(list(labels))
# ['event:evt-002']
print(list(labels))
# []
```

La generator expression es un iterator. Después de consumir un elemento, solo queda el resto.

### 15.4 Error tardío

**Failure case — el error aparece al consumir:**

```python
raw_scores = ["10", "bad", "30"]
scores = (int(raw_score) for raw_score in raw_scores)

print(next(scores))
# 10
print(next(scores))  # ValueError
```

Crear `scores` no convirtió aún todos los textos. La evidencia útil incluye el valor que se intentaba procesar y cuántos elementos se consumieron; la política formal de errores llega en PF-M6.

### 15.5 Decisión tiempo/memoria

Materializa cuando necesitas reutilización, índices, longitud o una snapshot estable. Procesa lazy cuando la secuencia puede ser grande, basta una pasada y puedes mantener explícito el ownership de sus dependencias.

En PF-M3 los datos siguen en memoria. PF-L05 aplicará el modelo a un stream, pero archivos y lifecycle pertenecen a PF-M6.

### Predice

```python
numbers = (number * 2 for number in range(4))
print(next(numbers))
print(sum(numbers))
print(list(numbers))
```

Predice cómo `sum` consume lo restante.

### Modifica

Cambia una list comprehension de 10 etiquetas por una generator expression. Consume solo las primeras dos con `next` y explica qué trabajo no ocurrió todavía.

### Explica

¿Qué tradeoff introduce materializar una generator expression con `list(...)`? Incluye memoria, reutilización y momento de los errores.

### Detecta el bug

```python
events = [
    {"id": "evt-001", "active": True},
    {"id": "evt-002", "active": False},
]

active_ids = (event["id"] for event in events if event["active"])

print("Primera vista:", list(active_ids))
print("Segunda vista:", list(active_ids))
```

Si ambas vistas debían ser iguales, conserva una list materializada o crea una generator expression nueva para cada pasada.

---

## 16. Generators al nivel de PF-M3

Generators son contenido **SHOULD** del track. Aquí se estudian para elegir entre materialización y una pasada, no para implementar pipelines de archivos, delegación con `yield from`, `send`, `throw`, cierre manual ni async generators.

### 16.1 Generator expression frente a list comprehension

La diferencia visual principal son corchetes frente a paréntesis:

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002", "evt-003"]

eager_labels = ["event:" + event_id for event_id in event_ids]
lazy_labels = ("event:" + event_id for event_id in event_ids)

print(type(eager_labels).__name__)
# list
print(type(lazy_labels).__name__)
# generator
print(list(lazy_labels))
# ['event:evt-001', 'event:evt-002', 'event:evt-003']
```

La forma interna de comprensión es la misma; el contrato de consumo cambia.

### 16.2 Función generadora mínima con `yield`

Una función que contiene `yield` devuelve un generator al llamarse. Su cuerpo empieza a ejecutarse cuando se solicita el primer elemento y se pausa en cada `yield` conservando su estado local.

**Ejemplo ejecutable:**

```python
def iter_active_event_ids(events):
    for event in events:
        if event["active"]:
            yield event["id"]


events = [
    {"id": "evt-001", "active": True},
    {"id": "evt-002", "active": False},
    {"id": "evt-003", "active": True},
]

active_ids = iter_active_event_ids(events)

print(next(active_ids))
# evt-001
print(list(active_ids))
# ['evt-003']
```

`yield` entrega un valor sin terminar permanentemente la función; la siguiente solicitud continúa después de ese punto.

### 16.3 Cuándo ayuda

- el input o output puede ser grande;
- basta una sola pasada;
- el consumidor puede detenerse temprano;
- la transformación por elemento es independiente y legible.

No ayuda si el caller necesita índices, `len`, varias pasadas o una snapshot estable. Tampoco uses generator para esconder una función corta que solo devuelve tres elementos conocidos.

### 16.4 Dependencias y datos mutables

Como el trabajo ocurre después, mutar los datos fuente entre la creación y el consumo puede cambiar el resultado.

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002"]
labels = ("event:" + event_id for event_id in event_ids)

event_ids.append("evt-003")

print(list(labels))
# ['event:evt-001', 'event:evt-002', 'event:evt-003']
```

No es un bug de Python: es una consecuencia del procesamiento tardío sobre una fuente mutable. Si necesitas snapshot, materializa o copia deliberadamente bajo un contrato claro.

### Predice

¿Cuándo se ejecuta el cuerpo de una función generadora: al llamar la función o al pedir el primer elemento? Diseña un ejemplo con `print` para comprobarlo, pero mantén `print` como instrumento de observación, no como diseño final.

### Modifica

Agrega un parámetro `person_id` a `iter_active_event_ids` y produce solo IDs activos de esa persona. Conserva la función sin side effects.

### Explica

¿Por qué un generator puede reducir memoria sin reducir necesariamente el trabajo total?

### Detecta el bug

```python
def iter_active_event_ids(events):
    for event in events:
        if event["active"]:
            yield event["id"]


events = [
    {"id": "evt-001", "active": True},
    {"id": "evt-002", "active": True},
]

active_ids = iter_active_event_ids(events)
count = sum(1 for _ in active_ids)
ids = list(active_ids)
```

`count` consume el generator; `ids` queda vacío. Si necesitas ambos resultados, acumúlalos en una pasada o materializa una vez y deriva el count con `len`.

---

## 17. Operaciones comunes guiadas por problemas

### 17.1 ¿Existe alguno? `any`

**Ejemplo ejecutable:**

```python
events = [
    {"id": "evt-001", "active": False},
    {"id": "evt-002", "active": True},
]

has_active_event = any(event["active"] for event in events)

print(has_active_event)
# True
```

`any` se detiene al encontrar un valor truthy. Con input vacío devuelve `False`.

### 17.2 ¿Cumplen todos? `all`

**Ejemplo ejecutable:**

```python
events = [
    {"id": "evt-001", "valid": True},
    {"id": "evt-002", "valid": True},
]

print(all(event["valid"] for event in events))
# True
print(all([]))
# True
```

`all` se detiene al encontrar un falsy. Para una colección vacía devuelve `True`: no existe contraejemplo. El contrato de dominio puede requerir comprobar además que haya elementos.

### 17.3 ¿Cuál es el total? `sum`

**Ejemplo ejecutable:**

```python
events = [
    {"id": "evt-001", "weight": 2},
    {"id": "evt-002", "weight": 3},
]

total_weight = sum(event["weight"] for event in events)

print(total_weight)
# 5
```

`sum` es para valores numéricos. No lo uses para concatenar strings o lists.

### 17.4 Extremos: `min` y `max`

**Ejemplo ejecutable:**

```python
timeline_orders = [30, 10, 20]

print(min(timeline_orders))
# 10
print(max(timeline_orders))
# 30
```

Sobre una colección vacía, `min` y `max` fallan salvo que se proporcione un `default` compatible con el contrato.

### 17.5 Vista ordenada: `sorted`

`sorted(iterable)` materializa una list nueva y no muta el input.

**Ejemplo ejecutable:**

```python
def timeline_key(event):
    return event["timeline_order"], event["id"]


events = [
    {"id": "evt-002", "timeline_order": 20},
    {"id": "evt-003", "timeline_order": 10},
    {"id": "evt-001", "timeline_order": 10},
]

ordered_events = sorted(events, key=timeline_key)

print([event["id"] for event in ordered_events])
# ['evt-001', 'evt-003', 'evt-002']
print([event["id"] for event in events])
# ['evt-002', 'evt-003', 'evt-001']
```

La key devuelve una tuple: primero decide `timeline_order`; `id` rompe empates para un resumen determinista. Ordenar tiene costo y materializa; no lo hagas dentro de cada iteración sin necesidad.

### 17.6 No convertir built-ins en inventario

Usa estas operaciones cuando expresan una pregunta completa. Un loop explícito sigue siendo mejor si necesitas explicar por qué falló cada elemento, acumular varios resultados relacionados o conservar evidencia de rechazos.

### Predice

Predice `any([])`, `all([])`, `sum([])` y explica por qué no tienen todos la misma semántica de “vacío”.

### Modifica

Calcula en una sola pasada explícita el número de eventos activos y el conjunto de personas activas. Compara la legibilidad con dos expresiones separadas que recorrerían dos veces.

### Explica

¿Por qué `sorted(events, key=...)` devuelve una list aunque el input sea un set o un generator?

### Detecta el bug

```python
events = []
if all(event["valid"] for event in events):
    print("Dataset válido")
```

Si el contrato exige al menos un evento, `all` no basta. Comprueba presencia y después validez.

---

## 18. Caso progresivo integrado: timeline e índices derivados de EIDOLON

Este caso usa solo estructuras y funciones básicas. Los events son datos sintéticos en memoria; no son dataclasses ni registros persistidos.

### 18.1 Fuente sintética

**Ejemplo ejecutable:**

```python
source_events = [
    {
        "id": "evt-003",
        "person_id": "person-007",
        "timeline_order": 30,
        "active": True,
        "tags": ["viaje", "familia"],
    },
    {
        "id": "evt-001",
        "person_id": "person-008",
        "timeline_order": 10,
        "active": False,
        "tags": ["trabajo"],
    },
    {
        "id": "evt-002",
        "person_id": "person-007",
        "timeline_order": 20,
        "active": True,
        "tags": ["viaje", "prioritario"],
    },
]

print(len(source_events))
# 3
```

La list conserva el orden de entrada, que no coincide necesariamente con `timeline_order`.

### 18.2 Índice `events_by_id` con política de duplicados

**Ejemplo ejecutable:**

```python
def build_events_by_id(events):
    events_by_id = {}
    duplicate_ids = set()

    for event in events:
        event_id = event["id"]
        if event_id in events_by_id:
            duplicate_ids.add(event_id)
        else:
            events_by_id[event_id] = event

    return events_by_id, duplicate_ids


source_events = [
    {"id": "evt-001", "text": "original"},
    {"id": "evt-002", "text": "second"},
    {"id": "evt-001", "text": "duplicate"},
]

events_by_id, duplicate_ids = build_events_by_id(source_events)

print(events_by_id["evt-001"]["text"])
# original
print(sorted(duplicate_ids))
# ['evt-001']
```

La política conserva el primer evento y registra los IDs duplicados. No borra la fuente ni finge que el duplicado nunca existió.

### 18.3 Índice por persona

**Ejemplo ejecutable:**

```python
def build_events_by_person(events):
    events_by_person = {}

    for event in events:
        person_id = event["person_id"]
        if person_id not in events_by_person:
            events_by_person[person_id] = []
        events_by_person[person_id].append(event)

    return events_by_person


events = [
    {"id": "evt-001", "person_id": "person-007"},
    {"id": "evt-002", "person_id": "person-008"},
    {"id": "evt-003", "person_id": "person-007"},
]

events_by_person = build_events_by_person(events)

print([event["id"] for event in events_by_person["person-007"]])
# ['evt-001', 'evt-003']
```

Cada person key necesita su propia list. Reutilizar una sola list para varias keys produciría aliasing.

### 18.4 Filtro y tags activos

**Ejemplo ejecutable:**

```python
events = [
    {"id": "evt-001", "active": True, "tags": ["viaje", "familia"]},
    {"id": "evt-002", "active": False, "tags": ["trabajo"]},
    {"id": "evt-003", "active": True, "tags": ["viaje", "prioritario"]},
]

active_events = [event for event in events if event["active"]]
active_tags = {
    tag
    for event in active_events
    for tag in event["tags"]
}

print([event["id"] for event in active_events])
# ['evt-001', 'evt-003']
print(sorted(active_tags))
# ['familia', 'prioritario', 'viaje']
```

La comprehension de filtro es pequeña; el set anidado conserva una lectura directa. Si se añadieran validación, normalización y razones de rechazo, convendrían funciones y loops.

### 18.5 Timeline determinista

**Ejemplo ejecutable:**

```python
def timeline_key(event):
    return event["timeline_order"], event["id"]


events = [
    {"id": "evt-003", "timeline_order": 30},
    {"id": "evt-001", "timeline_order": 10},
    {"id": "evt-002", "timeline_order": 20},
]

timeline = sorted(events, key=timeline_key)
timeline_ids = [event["id"] for event in timeline]

print(timeline_ids)
# ['evt-001', 'evt-002', 'evt-003']
```

`timeline` es una vista materializada nueva. La list fuente no fue reordenada.

### 18.6 Romperlo deliberadamente

Provoca y explica estos fallos antes de los ejercicios:

1. usa una list como key de `events_by_id` y observa `TypeError`;
2. accede a un ID ausente con `[]` y observa `KeyError`;
3. construye `events_by_person` usando una única list compartida para todas las personas;
4. usa una dict comprehension con duplicate IDs y observa la sobrescritura;
5. consume dos veces una generator expression de IDs activos;
6. combina IDs y status de longitudes distintas con `zip` sin `strict=True`;
7. elimina events de la list mientras la recorres y observa elementos saltados.

Para cada caso conserva el input sintético, el resultado o tipo de error y la regla que permite predecirlo. No diseñes todavía una arquitectura de excepciones o logging.

---

## 19. Ejercicios guiados con solución razonada

### Ejercicio guiado 1 — Elegir por operaciones dominantes

**Objetivo.** Justificar una colección sin memorizar la tabla.

Debes conservar una timeline pequeña en orden, consultar eventos muchas veces por ID y detectar source IDs repetidos.

**Predice antes de resolver:** ¿una sola colección satisface los tres contratos sin perder claridad?

**Solución ejecutable:**

```python
source_events = [
    {"id": "evt-001", "source_id": "src-001"},
    {"id": "evt-002", "source_id": "src-002"},
]

events_by_id = {}
seen_source_ids = set()

for event in source_events:
    events_by_id[event["id"]] = event
    seen_source_ids.add(event["source_id"])

assert [event["id"] for event in source_events] == ["evt-001", "evt-002"]
assert events_by_id["evt-002"]["source_id"] == "src-002"
assert seen_source_ids == {"src-001", "src-002"}
```

**Razonamiento.** La list conserva el orden fuente; el dict añade lookup por ID; el set expresa unicidad/membership. Ningún derivado sustituye la fuente. La solución es correcta por sus contratos, no porque use más estructuras.

**Variación.** Añade un duplicate `source_id` y detecta la repetición antes de `add`.

### Ejercicio guiado 2 — Corregir aliasing anidado

**Objetivo.** Construir lists interiores independientes.

**Código inicial — predice antes de ejecutar:**

```python
tags_by_day = [[]] * 3
tags_by_day[1].append("viaje")
```

**Solución ejecutable:**

```python
tags_by_day = [[] for _ in range(3)]
tags_by_day[1].append("viaje")

assert tags_by_day == [[], ["viaje"], []]
assert tags_by_day[0] is not tags_by_day[1]
```

**Razonamiento.** La repetition repite referencias; la comprehension evalúa `[]` en cada iteración. El segundo assert comprueba independencia, no solo un output accidental.

**Variación.** Usa un loop explícito con `append([])` y comprueba la misma propiedad.

### Ejercicio guiado 3 — Distinguir key ausente de value `None`

**Objetivo.** Elegir membership frente a `.get`.

**Código inicial:**

```python
status_by_id = {"evt-001": None}
```

**Solución ejecutable:**

```python
status_by_id = {"evt-001": None}

assert status_by_id.get("evt-001") is None
assert status_by_id.get("evt-999") is None
assert "evt-001" in status_by_id
assert "evt-999" not in status_by_id
```

**Razonamiento.** `.get` sin default colapsa dos estados en `None`. Membership responde la pregunta de presencia. La solución sería accidental si solo comparara ambos `.get` con `None`.

**Variación.** Usa `.get(key, "missing")` solo si `"missing"` no puede ser un value válido.

### Ejercicio guiado 4 — Detectar duplicate IDs sin sobrescribir

**Objetivo.** Construir un índice derivado con política explícita.

**Predice:** ¿qué event conserva una dict comprehension si la misma key aparece dos veces?

**Solución ejecutable:**

```python
events = [
    {"id": "evt-001", "text": "first"},
    {"id": "evt-001", "text": "duplicate"},
    {"id": "evt-002", "text": "second"},
]

events_by_id = {}
duplicate_ids = set()

for event in events:
    event_id = event["id"]
    if event_id in events_by_id:
        duplicate_ids.add(event_id)
    else:
        events_by_id[event_id] = event

assert events_by_id["evt-001"]["text"] == "first"
assert duplicate_ids == {"evt-001"}
assert len(events) == 3
```

**Razonamiento.** El loop hace visible la decisión “first wins” y conserva la fuente con el duplicado para diagnóstico. Una coincidencia accidental sería terminar con dos keys sin comprobar qué versión se conservó.

**Variación.** Implementa “last wins”, pero registra igualmente el ID duplicado.

### Ejercicio guiado 5 — No mutar durante iteración

**Objetivo.** Filtrar sin saltar elementos.

**Código incorrecto — predice:**

```python
event_ids = ["keep-1", "drop-1", "drop-2", "keep-2"]
for event_id in event_ids:
    if event_id.startswith("drop-"):
        event_ids.remove(event_id)
```

**Solución ejecutable:**

```python
event_ids = ["keep-1", "drop-1", "drop-2", "keep-2"]
kept_ids = [
    event_id
    for event_id in event_ids
    if not event_id.startswith("drop-")
]

assert event_ids == ["keep-1", "drop-1", "drop-2", "keep-2"]
assert kept_ids == ["keep-1", "keep-2"]
```

**Razonamiento.** Fuente y resultado son objetos separados. La comprehension contiene un filtro único y legible. El primer assert evita que una solución “correcta” haya mutado silenciosamente la fuente.

**Variación.** Reescribe la solución con loop explícito.

### Ejercicio guiado 6 — `zip` con contrato estricto

**Objetivo.** Evitar truncamiento silencioso.

**Predice:** ¿qué devuelve `list(zip([1, 2], ["a"]))`?

**Solución ejecutable:**

```python
event_ids = ["evt-001", "evt-002"]
statuses = ["draft", "accepted"]

pairs = list(zip(event_ids, statuses, strict=True))

assert pairs == [("evt-001", "draft"), ("evt-002", "accepted")]
```

**Razonamiento.** `strict=True` expresa que cada ID debe tener status. La solución no depende de que las longitudes “casualmente” coincidan; el failure case con una longitud menor debe producir `ValueError`.

**Variación.** Agrega un tercer ID sin status y provoca el error.

### Ejercicio guiado 7 — Convertir loop en comprehension

**Objetivo.** Mantener equivalencia y legibilidad.

**Código inicial:**

```python
events = [
    {"id": "evt-001", "active": True},
    {"id": "evt-002", "active": False},
]
active_ids = []
for event in events:
    if event["active"]:
        active_ids.append(event["id"])
```

**Solución ejecutable:**

```python
events = [
    {"id": "evt-001", "active": True},
    {"id": "evt-002", "active": False},
]

active_ids = [event["id"] for event in events if event["active"]]

assert active_ids == ["evt-001"]
```

**Razonamiento.** Una fuente, una transformación y un filtro caben sin ocultar una política. Si hubiera que explicar rechazos, el loop sería preferible.

**Variación.** Extrae un predicado nombrado antes de agregar una segunda condición.

### Ejercicio guiado 8 — Aplanar tags sin side effects

**Objetivo.** Usar set comprehension para una colección derivada.

**Solución ejecutable:**

```python
events = [
    {"id": "evt-001", "tags": ["viaje", "familia"]},
    {"id": "evt-002", "tags": ["viaje", "trabajo"]},
]

active_tags = {
    tag
    for event in events
    for tag in event["tags"]
}

assert active_tags == {"viaje", "familia", "trabajo"}
```

**Razonamiento.** El resultado es conceptualmente un conjunto; la comprehension no imprime ni modifica un contenedor externo. La igualdad de sets evita depender de su orden.

**Variación.** Conserva además el primer orden observado usando un loop, una list de salida y un set de membership.

### Ejercicio guiado 9 — Diagnosticar un iterator consumido

**Objetivo.** Elegir una pasada o materialización.

**Código inicial — predice:**

```python
active_ids = (event_id for event_id in ["evt-001", "evt-002"])
first = list(active_ids)
second = list(active_ids)
```

**Solución ejecutable:**

```python
active_ids_iterator = (event_id for event_id in ["evt-001", "evt-002"])
active_ids = list(active_ids_iterator)

first = list(active_ids)
second = list(active_ids)

assert first == second == ["evt-001", "evt-002"]
```

**Razonamiento.** Se materializa una vez porque el contrato exige reutilización. Si bastara una sola pasada, materializar sería innecesario.

**Variación.** Mantén lazy el pipeline y calcula dos resultados en una sola pasada explícita.

### Ejercicio guiado 10 — Resumen determinista

**Objetivo.** Ordenar por una key explícita y no mutar la fuente.

**Solución ejecutable:**

```python
def timeline_key(event):
    return event["timeline_order"], event["id"]


events = [
    {"id": "evt-002", "timeline_order": 20},
    {"id": "evt-003", "timeline_order": 10},
    {"id": "evt-001", "timeline_order": 10},
]

ordered = sorted(events, key=timeline_key)
ordered_ids = [event["id"] for event in ordered]

assert ordered_ids == ["evt-001", "evt-003", "evt-002"]
assert [event["id"] for event in events] == ["evt-002", "evt-003", "evt-001"]
```

**Razonamiento.** La tuple de key crea un desempate estable y deliberado. El segundo assert comprueba que `sorted` produjo una vista nueva.

**Variación.** Cambia dos IDs manteniendo el mismo `timeline_order` y predice el desempate.

---

## 20. Ejercicios independientes

No consultes una solución antes de escribir una predicción y un criterio verificable.

1. **Índices y slices.** Dada una list de cinco IDs, produce mediante slicing: los tres primeros, los dos últimos y el orden inverso. Explica `stop` exclusivo.
2. **Operaciones de list.** Simula una cola pequeña con `append` y `pop(0)`. Describe el desplazamiento y señala que estructuras de cola más adecuadas se estudiarán en Computer Science; no importes otra API todavía.
3. **Shallow copy.** Diseña una list anidada donde copiar con `[:]` separe el exterior pero comparta un dict interior. Demuestra ambas propiedades.
4. **Tuple como contrato.** Escribe una función que devuelva `(valid_count, invalid_count)` y usa unpacking. Justifica por qué dos posiciones son manejables y cinco serían frágiles.
5. **Contenido mutable en tuple.** Provoca una mutación interior válida y una reasignación de posición inválida. Predice el tipo de error.
6. **KeyError.** Compara `mapping[key]`, `.get(key)` y membership para un ID ausente. Explica qué forma comunica mejor tres contratos distintos.
7. **Membership de dict.** Corrige un programa que busca un status entre keys. Después diseña un índice inverso `status -> list de IDs` sin compartir accidentalmente las lists.
8. **Hashability.** Clasifica como key válida o inválida: string, int, tuple de strings, list de strings, tuple que contiene una list, frozenset de strings. Comprueba tus predicciones.
9. **Orden de dict.** Inserta, actualiza, elimina y reinserta una key. Predice el orden final y explica por qué no equivale a tiempo de dominio.
10. **Set operations.** Con tags requeridas, observadas y prohibidas, calcula faltantes, compartidas y unión. Muestra output determinista.
11. **Dedupe con orden.** Conserva el primer orden de IDs y elimina duplicados usando una list de resultado más un set de membership. Justifica ambas estructuras.
12. **Lookup repetido.** Implementa primero tres consultas lineales sobre events; después construye `events_by_id`. Explica el tradeoff de tiempo y memoria sin notación formal.
13. **Modificación durante iteración.** Reproduce un elemento saltado en list y un `RuntimeError` en dict. Corrige ambos sin ocultar el fallo.
14. **Iterator consumido.** Consume un elemento con `next`, el resto con `list` y vuelve a consumir. Dibuja el estado tras cada paso.
15. **Range.** Produce cuatro progresiones, incluida una descendente. Para cada una explica por qué `stop` se incluye o excluye.
16. **Enumerate.** Numera events desde `1` y conserva la posición original después de filtrar. Contrasta con numerar solo el resultado filtrado.
17. **Zip estricto.** Combina tres iterables de igual longitud y provoca después un mismatch. Explica cuándo aparece el error si consumes con `next`.
18. **Comprehension progresiva.** Escribe una transformación con loop, luego list comprehension, luego filtro y finalmente transformación + filtro. Compara claridad.
19. **Duplicate key.** Demuestra cómo una dict comprehension sobrescribe una key. Reescribe con loop para detectar y conservar evidencia del duplicado.
20. **Nested comprehension.** Aplana dos niveles de tags. Agrega una condición; si deja de ser legible, vuelve a loops y explica la decisión.
21. **Side effects.** Corrige tres antipatterns: `print`, `set.add` y `dict.update` dentro de comprehensions.
22. **Lazy vs eager.** Procesa 100 números con list comprehension y generator expression. No midas performance; compara tipo, `len`, reutilización y consumo.
23. **Función generadora.** Implementa `iter_events_for_person(events, person_id)` con `yield`. Demuestra single pass.
24. **Error tardío.** Crea una generator expression que convierta textos a int con un dato inválido en tercer lugar. Registra qué valores ya se consumieron antes del error.
25. **Built-ins.** Responde con código a cinco preguntas: ¿hay alguno activo?, ¿son todos válidos?, ¿peso total?, ¿orden mínimo/máximo?, ¿timeline ordenada?
26. **Índice derivado.** Construye `events_by_id`, `events_by_person` y `active_tags` desde una list fuente. Demuestra que puedes reconstruirlos y que no sustituyen la fuente.

---

## 21. Preguntas conceptuales

1. ¿Qué propiedad pierde una list al convertirla en set y cuándo esa pérdida es semánticamente incorrecta?
2. ¿Por qué acceso por índice y lookup por ID no son la misma operación?
3. ¿Cómo se conectan aliasing y shallow copy con una list de dicts?
4. ¿Qué expresa una tuple como contrato además de impedir cambios estructurales?
5. ¿Por qué una tuple puede ser inmutable y aun así observar cambios internos?
6. ¿Qué diferencia existe entre key ausente y key presente con value `None`?
7. ¿Qué recorre `for key in mapping` y cuándo usarías `.items()`?
8. ¿Por qué las keys de dict y los elementos de set necesitan hashability?
9. ¿Qué significa que dict conserve orden de inserción y qué no significa?
10. ¿Cuándo compensa construir `events_by_id` y cuándo basta un scan?
11. ¿Por qué un índice in-memory de EIDOLON es derived data y no source of truth?
12. ¿Qué diferencia existe entre iterable e iterator? Da un ejemplo reutilizable y uno consumible.
13. ¿Cómo sabe un `for` que terminó un iterator?
14. ¿Por qué modificar una list durante su iteración puede saltar elementos?
15. ¿Por qué `range` no es list ni iterator de una sola pasada?
16. ¿Qué estado manual evita `enumerate`?
17. ¿Qué contrato aplica `zip` sin `strict=True` y qué cambia al activarlo?
18. ¿Cuándo una comprehension comunica mejor que un loop y cuándo oculta decisiones?
19. ¿Por qué los side effects dentro de comprehensions contradicen su intención de construcción de valores?
20. ¿Qué relación existe entre funciones puras de PF-M2 y comprehensions legibles?
21. ¿Qué cambia entre list comprehension y generator expression: sintaxis, memoria, momento de ejecución y reutilización?
22. ¿Cómo puede un error aparecer tarde en un pipeline lazy?
23. ¿Por qué reducir memoria no garantiza reducir trabajo total?
24. ¿Por qué `all([])` es `True` y qué validación adicional puede exigir el dominio?
25. ¿Qué desempate hace determinista una timeline con valores de orden iguales?
26. ¿Qué parte de estos costos profundizarán CS-M1 y CS-M2 y por qué PF-M3 se detiene antes?

---

## 22. Mini challenge — Pipeline de eventos e índices derivados

### 22.1 Objetivo

Construye un script en memoria que reciba eventos sintéticos básicos y produzca:

- eventos válidos sin duplicate IDs;
- IDs únicos;
- un índice `events_by_id`;
- un índice `events_by_person`;
- tags utilizadas por eventos activos;
- una timeline y un resumen deterministas.

Usa exclusivamente PF-M1, PF-M2 y PF-M3.

### 22.2 Entrada obligatoria

**Ejemplo ejecutable — datos de entrada:**

```python
source_events = [
    {
        "id": "evt-003",
        "source_id": "src-003",
        "person_id": "person-007",
        "timeline_order": 30,
        "active": True,
        "tags": ["viaje", "familia", "viaje"],
    },
    {
        "id": "evt-001",
        "source_id": "src-001",
        "person_id": "person-008",
        "timeline_order": 10,
        "active": False,
        "tags": ["trabajo"],
    },
    {
        "id": "evt-002",
        "source_id": "src-002",
        "person_id": "person-007",
        "timeline_order": 20,
        "active": True,
        "tags": ["prioritario", "viaje"],
    },
    {
        "id": "evt-002",
        "source_id": "src-099",
        "person_id": "person-999",
        "timeline_order": 99,
        "active": True,
        "tags": ["duplicado"],
    },
    {
        "id": "",
        "source_id": "src-004",
        "person_id": "person-007",
        "timeline_order": 40,
        "active": True,
        "tags": ["invalido"],
    },
]
```

Todos los datos son sintéticos. No agregues campos, archivos ni formatos externos.

### 22.3 Política obligatoria

1. Un evento es válido si contiene las keys `id`, `source_id`, `person_id`, `timeline_order`, `active` y `tags`, y si `id`, `source_id` y `person_id` no son strings vacíos.
2. Para este challenge, `timeline_order` debe ser un `int`, `active` un `bool` y `tags` una `list`; cada tag presente debe ser un `str` no vacío.
3. La validación no debe mutar el evento.
4. Entre eventos válidos con el mismo `id`, conserva el primero y registra el ID en `duplicate_ids`.
5. No elimines ni sobrescribas `source_events`; es la fuente sintética del ejercicio.
6. Los índices son derivados y deben poder reconstruirse desde `valid_events`.
7. `events_by_person[person_id]` contiene una list distinta por persona y conserva el orden de `valid_events`.
8. `active_tags` es un set deduplicado de tags de eventos activos.
9. La timeline se ordena por `(timeline_order, id)` sin mutar `valid_events`.
10. El resumen expone listas ordenadas de IDs y tags para ser determinista.

La comprobación con `type(value) is int` y `type(value) is bool` es deliberada en este challenge: `bool` es un subtipo de `int`, por lo que `isinstance(True, int)` no expresaría este contrato exacto. No generalices esta decisión sin revisar el dominio.

### 22.4 Funciones requeridas

Implementa funciones pequeñas con estos contratos:

```text
is_valid_event(event) -> bool
deduplicate_valid_events(source_events) -> (valid_events, duplicate_ids)
build_events_by_id(valid_events) -> dict
build_events_by_person(valid_events) -> dict
collect_active_tags(valid_events) -> set
build_timeline(valid_events) -> list
build_summary(valid_events, duplicate_ids, active_tags, timeline) -> dict
process_events(source_events) -> dict
```

`process_events` compone las demás. Las funciones de cálculo no imprimen ni modifican `source_events`.

### 22.5 Requisitos de implementación

1. Usa una `list` para `valid_events` y para cada grupo por persona.
2. Usa `dict` para ambos índices y para el resultado compuesto.
3. Usa `set` para IDs vistos, `duplicate_ids` y `active_tags`.
4. Usa funciones y retornos explícitos de PF-M2.
5. Usa loops donde haya política de validación, deduplicación o acumulación múltiple.
6. Usa al menos una comprehension donde la transformación sea corta y sin side effects.
7. Usa `sorted` con una función de key nombrada para la timeline.
8. No uses side effects dentro de comprehensions.
9. No consumas un iterator dos veces; si introduces una generator expression opcional, demuestra su ownership.
10. No uses `try`/`except` para convertir datos inválidos: la validación debe clasificarlos sin conceptos de PF-M6.

### 22.6 Contrato mínimo de salida

`process_events(source_events)` devuelve un dict con estas keys:

```text
valid_events
duplicate_ids
events_by_id
events_by_person
active_tags
timeline
summary
```

`summary` contiene al menos:

```text
valid_count
unique_ids
duplicate_ids
person_ids
active_tags
timeline_ids
```

Las colecciones de `summary` deben ser deterministas: usa lists ordenadas. Los índices conservan sus tipos útiles y no necesitan convertirse a texto.

### 22.7 Comprobaciones mínimas

**Fragmento de comprobación — ejecútalo después de tu implementación:**

```python
source_snapshot = []
for event in source_events:
    event_snapshot = event.copy()
    event_snapshot["tags"] = event["tags"].copy()
    source_snapshot.append(event_snapshot)

result = process_events(source_events)

assert source_events == source_snapshot
assert len(source_events) == 5
assert source_events[0]["tags"] is not source_snapshot[0]["tags"]

assert [event["id"] for event in result["valid_events"]] == [
    "evt-003",
    "evt-001",
    "evt-002",
]
assert result["duplicate_ids"] == {"evt-002"}
assert set(result["events_by_id"]) == {"evt-001", "evt-002", "evt-003"}

assert [
    event["id"]
    for event in result["events_by_person"]["person-007"]
] == ["evt-003", "evt-002"]
assert result["events_by_person"]["person-007"] is not result["events_by_person"]["person-008"]

assert result["active_tags"] == {"viaje", "familia", "prioritario"}
assert [event["id"] for event in result["timeline"]] == [
    "evt-001",
    "evt-002",
    "evt-003",
]

assert result["summary"] == {
    "valid_count": 3,
    "unique_ids": ["evt-001", "evt-002", "evt-003"],
    "duplicate_ids": ["evt-002"],
    "person_ids": ["person-007", "person-008"],
    "active_tags": ["familia", "prioritario", "viaje"],
    "timeline_ids": ["evt-001", "evt-002", "evt-003"],
}
```

Cada dict y su list interior de tags se copian de manera deliberada para que el snapshot detecte mutaciones en ambos niveles usados por este challenge. No generalices esta copia manual a grafos arbitrarios: PF-M1 ya explicó los límites de shallow y deep copy.

### 22.8 Failure cases obligatorios

Provoca, diagnostica y corrige por separado:

1. dos duplicate IDs no consecutivos;
2. un event sin key `tags` y, por separado, otro con un tag que no sea `str` no vacío;
3. un `id` vacío;
4. un `timeline_order` que sea `True` en vez de un int exacto;
5. dos personas que comparten accidentalmente la misma list en `events_by_person`;
6. una dict comprehension que sobrescribe el duplicate ID sin registrarlo;
7. una list usada para membership repetido de IDs vistos;
8. un `zip` opcional de IDs/status con longitudes distintas sin `strict=True`;
9. un iterator de IDs activos consumido durante el count y reutilizado después;
10. una comprehension que modifica `events_by_id` y produce una list de `None`.

Para cada caso entrega: input mínimo, predicción, síntoma, causa, corrección y dato que conservarías para diagnóstico. No implementes logging ni excepciones de dominio todavía.

### 22.9 Criterio de aprobación

Apruebas el challenge si:

- todos los asserts pasan;
- otra persona puede reconstruir los índices solo desde `valid_events`;
- puedes justificar cada colección por su contrato y operación dominante;
- la política de duplicados es visible y no pierde la fuente;
- ninguna comprehension contiene side effects ni reglas difíciles de leer;
- puedes demostrar qué resultados son eager y cuáles serían de una sola pasada;
- los diez failure cases se reproducen y explican;
- la solución no usa conceptos posteriores.

### 22.10 Límites explícitos

No agregues classes, dataclasses, type hints, archivos, JSON, repositories, bases de datos, decorators, context managers, async, frameworks, embeddings ni graph databases. Tampoco conviertas el challenge en el build completo de EIDOLON. PF-M3 demuestra selección e iteración de colecciones en memoria.

---

## 23. Resumen

- Una colección expresa relaciones entre varios elementos, no solo capacidad de almacenamiento.
- `list` representa una secuencia ordenada, mutable y con duplicados; el índice no es una identidad de dominio.
- Los slices materializan una list exterior nueva y comparten referencias interiores.
- `append`, `extend` e `insert` tienen contratos distintos; insertar o eliminar en medio desplaza elementos.
- Una `tuple` expresa una estructura posicional estable; su contenido alcanzable puede seguir siendo mutable.
- `dict` expresa key → value, conserva orden de inserción y consulta membership sobre keys.
- `[]` produce `KeyError` para una key ausente; `.get` puede ocultar la diferencia entre ausencia y `None`.
- Keys de dict y elementos de set deben ser hashable.
- `set` expresa unicidad y membership; `frozenset` conserva esas propiedades sin mutación.
- Un set no debe usarse cuando orden o duplicados son información.
- Lookup de dict y membership de set suelen ser baratos en promedio; list membership es un scan lineal.
- Esos costos son intuiciones de PF-M3; CS-M1 y CS-M2 los formalizarán.
- Un índice in-memory es derived data reconstruible, no source of truth persistente.
- Un iterable produce un iterator; un iterator conserva estado de consumo y puede agotarse.
- `for` obtiene elementos hasta `StopIteration`.
- Modificar la colección recorrida puede saltar elementos o fallar; construir un resultado nuevo suele ser más claro.
- `range` representa una progresión compacta con `stop` exclusivo y no es una list materializada.
- `enumerate` enlaza posición y valor sin contador manual.
- `zip` trunca al iterable más corto salvo `strict=True`.
- Una comprehension ayuda con una transformación y filtro legibles; no debe esconder políticas, complejidad ni efectos.
- Nested comprehensions se conservan solo mientras el orden mental resulte evidente.
- List/set/dict comprehensions son eager; una generator expression produce bajo demanda.
- Iterators y generators suelen ser de una sola pasada y pueden producir errores tardíos.
- `any`, `all`, `sum`, `min`, `max` y `sorted` expresan preguntas concretas; no sustituyen un loop cuando se necesita evidencia detallada.

---

## 24. Checklist de dominio

- [ ] Puedo distinguir valor, secuencia, asociación y conjunto.
- [ ] Puedo elegir una colección según orden, mutabilidad, acceso, asociación y unicidad.
- [ ] Puedo justificar la operación dominante y un costo aproximado sin fingir análisis formal.
- [ ] Puedo usar índices negativos y slicing, y explicar `stop` exclusivo.
- [ ] Puedo distinguir `append`, `extend`, `insert`, `remove`, `pop` y `del`.
- [ ] Puedo predecir aliasing en una list y en lists anidadas.
- [ ] Puedo detectar filas compartidas creadas con `[[]] * n`.
- [ ] Puedo justificar una shallow copy sin asumir independencia interior.
- [ ] Puedo usar tuple y unpacking para un contrato posicional pequeño.
- [ ] Puedo explicar por qué una tuple puede contener objetos mutables.
- [ ] Puedo crear, consultar, actualizar e iterar un dict.
- [ ] Puedo distinguir membership de keys y búsqueda entre values.
- [ ] Puedo decidir entre `[]`, `.get` y membership ante ausencia.
- [ ] Puedo provocar y diagnosticar `KeyError`.
- [ ] Puedo explicar hashability al elegir keys o elementos de set.
- [ ] Puedo detectar una key no hashable.
- [ ] Puedo explicar el orden de inserción de dict sin confundirlo con orden temporal.
- [ ] Puedo construir dicts anidados sin aliasing accidental.
- [ ] Puedo usar set para unicidad, membership, union, intersection, difference, subset y superset.
- [ ] Puedo elegir entre set, frozenset y list sin perder orden o duplicados necesarios.
- [ ] Puedo construir `events_by_id`, `events_by_person`, `seen_source_ids` y `active_tags` como derivados.
- [ ] Puedo explicar por qué un dict in-memory no es source of truth persistente.
- [ ] Puedo distinguir iterable de iterator.
- [ ] Puedo usar `iter` y `next` y explicar `StopIteration`.
- [ ] Puedo detectar el consumo y reutilización incorrecta de un iterator.
- [ ] Puedo filtrar sin modificar la colección durante iteración.
- [ ] Puedo usar `range` con start, stop y step correctos.
- [ ] Puedo preferir `enumerate` a un contador manual cuando necesito posición.
- [ ] Puedo usar `zip(..., strict=True)` cuando longitudes distintas son un bug.
- [ ] Puedo traducir un loop sencillo a list, set o dict comprehension.
- [ ] Puedo detectar una comprehension ilegible o con side effects.
- [ ] Puedo decidir cuándo volver a loops o funciones nombradas.
- [ ] Puedo explicar eager, lazy, materialización, memoria y single pass.
- [ ] Puedo usar generator expression y una función generadora mínima.
- [ ] Puedo diagnosticar un error tardío y un generator consumido.
- [ ] Puedo aplicar `any`, `all`, `sum`, `min`, `max` y `sorted` a preguntas concretas.
- [ ] Puedo completar el mini challenge y defender cada colección y transformación.

---

## 25. Preparación para labs y EIDOLON 0.0a

Después de dominar PF-M3 puedes comenzar:

- **PF-L04 — Índice de entidades:** es el laboratorio principal. Ya puedes construir `dict`/`set` derivados, detectar duplicados y explicar el tradeoff frente a búsqueda lineal.
- **PF-L05 — Stream de eventos:** PF-M3 prepara generator expressions, single pass y errores tardíos. Aún faltan archivos, JSONL y lifecycle de PF-M6 para completar el lab.
- **PF-L03 — Funciones sin estado oculto:** ahora puedes refactorizar filtros y acumulaciones usando colecciones con contratos explícitos.
- **PF-L01 — Diagnóstico reproducible:** puedes añadir failure cases de aliasing, `KeyError`, hashability, mutación durante iteración y consumo.
- **EIDOLON 0.0a:** ya puedes mantener índices derivados en memoria para ID, persona y tags, y producir una timeline determinista. Aún faltan packaging, modelos, persistencia, errores y testing profesional.

### Evidencia antes de avanzar a PF-M4

1. el script del mini challenge con todos los asserts;
2. una tabla de decisiones para sus colecciones y operaciones dominantes;
3. los diez ejercicios guiados ejecutados con predicciones previas;
4. al menos diez ejercicios independientes, incluidos 3, 7, 11, 12, 13, 17, 19, 21, 24 y 26;
5. siete failure cases reproducibles: aliasing anidado, `KeyError`, key no hashable, mutación durante iteración, overwrite de duplicate ID, `zip` truncado e iterator consumido;
6. una comparación escrita de una búsqueda lineal frente a `events_by_id`, sin formalismo de Big O;
7. una nota breve titulada **“Qué materializaría y qué consumiría una sola vez”**;
8. una explicación oral de cinco minutos o nota equivalente sobre por qué los índices derivados no son source of truth.

Este módulo contribuye al **CHECKPOINT PF-C1 — Código determinista**: debes explicar colecciones sin documentación e implementar primero un filtro claro antes de comparar alternativas.

---

## 26. Recursos de ampliación

La explicación fundamental está contenida en este módulo. Para verificar sintaxis y profundizar selectivamente consulta la [documentación oficial de Python 3.14](https://docs.python.org/3/contents.html), en especial el Tutorial sobre data structures y control flow, y la referencia de tipos integrados e iteración.

Los libros y recursos compartidos permanecen en [`PF.11 Recursos recomendados`](../../02_curriculum/01_programming_foundations.md#pf11-recursos-recomendados). Úsalos para ampliar, no para sustituir los modelos y ejercicios de este capítulo.

---

## 27. Límite del módulo

PF-M3 termina aquí. **PF-M4** estudiará módulos, packages y dependency management; **PF-M5**, POO, dataclasses y type hints; **PF-M6**, excepciones, archivos, JSON y lifecycle; **PF-M7**, decorators y context managers; **PF-M8**, async/await; y **PF-M9**, testing y debugging avanzado.

Generators avanzados, iterators personalizados, pipelines de archivos, async generators y backpressure no pertenecen a PF-M3. **CS-M1** y **CS-M2** formalizarán complejidad, análisis y estructuras; este módulo solo establece intuiciones suficientes para elegir colecciones en programas pequeños.

Tampoco se introducen repositories, databases, JSONL, embeddings, graph databases, backend, frameworks ni AI. La frontera lograda es concreta: colecciones correctas, iteración predecible, transformaciones legibles e índices derivados en memoria.
