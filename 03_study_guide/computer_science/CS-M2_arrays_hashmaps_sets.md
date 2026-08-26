# CS-M2 — Arrays, hash maps y sets

**Track:** Computer Science Foundations  
**Competencias:** D2.1; soporte D2.3, D3.1  
**Fase:** P0  
**Nivel objetivo:** Aplicado  
**Prerequisites:** PF-M1, PF-M3, PF-M5, PF-M9, CS-M1  
**Build:** EIDOLON 0.0b  
**Curriculum source:** [CS-M2](../../02_curriculum/02_computer_science_foundations.md#cs-m2--arrays-hash-maps-y-sets)  
**Status:** review candidate

Programming Foundations enseñó a utilizar `list`, `dict` y `set`. CS-M1 enseñó a definir `n`, estimar tiempo y memoria y medir una hipótesis. Ahora abriremos la caja conceptual lo suficiente para justificar por qué esas estructuras favorecen operaciones distintas.

El hilo del módulo es:

```text
workload
↓
operación dominante
↓
modelo de almacenamiento
↓
costo y supuestos
↓
failure cases
↓
medición
↓
estructura defendible
```

No basta memorizar “list es O(n)” o “dict es O(1)”. Una `list` ofrece acceso por posición O(1) y append amortizado O(1); un `dict` necesita build, memoria, claves estables y un modelo expected, no una garantía absoluta. Para una consulta pequeña, el scan puede seguir siendo la mejor decisión.

Este módulo no desarrolla linked lists, stacks, queues, búsqueda/ordenamiento formal, trees, heaps, graphs ni arquitectura de cache. Esos temas pertenecen a CS-M3–CS-M10.

## Resultados de aprendizaje

Al terminar podrás:

- explicar un array como posiciones indexables y una `list` de Python como dynamic array de referencias;
- justificar acceso por índice O(1), slicing O(k), append O(1) amortizado e inserción/eliminación con desplazamientos;
- separar el costo de buscar un valor del costo de mover referencias;
- explicar length, capacity y locality al nivel práctico;
- modelar una hash table mediante hash, buckets, entries y collision resolution;
- distinguir hashing de encryption y de identidad persistente;
- aplicar el contrato entre equality y hash y reconocer keys inestables;
- implementar una hash map didáctica con separate chaining sin confundirla con `dict`;
- explicar expected practical O(1), worst O(n), load factor y resizing conceptual;
- usar la garantía de insertion order de `dict` sin confundirla con sorted order;
- detectar duplicate keys antes de que un índice EIDOLON sobrescriba evidencia;
- justificar `set` y `frozenset` por membership, unicidad e inmutabilidad;
- producir output determinista sin depender del orden de un `set`;
- reconstruir list/dict/set in-memory desde una fuente autoritativa;
- medir build, una consulta, consultas repetidas y memoria antes de adoptar un índice;
- documentar el time-space tradeoff y un migration trigger cuantitativo.

## Cómo estudiar este módulo

1. Define la operación antes de nombrar la estructura.
2. Dibuja referencias y posiciones; no imagines valores crudos dentro de una `list`.
3. Para hashing, separa hash, bucket, collision, equality y entry.
4. Predice el costo antes de ejecutar cada ejemplo.
5. Verifica primero corrección y duplicate policy; mide después.
6. Incluye build y memoria cuando compares una proyección con un scan.
7. No copies tiempos locales como límites universales.

### Convenciones

- **Ejemplo ejecutable:** bloque autónomo con output estable o asserts.
- **Benchmark ejecutable:** produce cifras ambientales; comprueba formato y propiedades, no tiempos exactos.
- **Modelo educativo:** implementación pequeña para observar una idea; no representa internals exactos de CPython.
- **Código incorrecto:** rompe deliberadamente un contrato.
- **Failure case:** provoca el síntoma indicado y explica causa/corrección.
- **Fragmento:** requiere contexto omitido y no se ofrece como programa autónomo.

Baseline recomendado: Python 3.14 y standard library. Los tamaños grandes son ajustables a RAM, CPU y carga del equipo.

---

## 1. De usar estructuras a comprenderlas

Estas tres expresiones ya son conocidas:

**Fragmento:**

```python
timeline.append(event)
event = events_by_id[event_id]
already_seen = source_id in seen_source_ids
```

La sintaxis no explica por qué las operaciones cuestan distinto. Cada estructura promete una organización:

| Necesidad | Organización inicial |
|---|---|
| conservar secuencia, posiciones y duplicates | array dinámico / `list` |
| asociar stable key con value | hash map / `dict` |
| expresar membership y unicidad | hash set / `set` |

La tabla no decide sola. Debes preguntar:

- ¿predomina traversal, acceso por posición o lookup por key?;
- ¿el orden y los duplicates son parte del contrato?;
- ¿cuántas consultas reutilizarán la estructura?;
- ¿qué build y memoria adicional compra la mejora?;
- ¿qué estructura es autoridad y cuál puede reconstruirse?

### Estructura

Una command recibe 30 eventos ordenados, los muestra una vez y termina. ¿Qué operación domina? ¿Qué evidencia necesitarías antes de añadir un `dict`?

### Predice el costo

Para `q` búsquedas lineales sobre `n` Events, deriva el worst-case time. Conecta tu respuesta con CS-M1 sin reabrir Big O formal.

---

## 2. Array: posiciones indexables

### 2.1 Modelo mental

Un **array** representa una secuencia de posiciones numeradas:

```text
índice      0       1       2       3
posición  [   ]   [   ]   [   ]   [   ]
elemento    A       B       C       D
length = 4
```

- el **índice** identifica una posición;
- la **posición** es un slot de la secuencia;
- el **elemento** es lo recuperado desde ese slot;
- la **longitud** es la cantidad de posiciones usadas.

En un array contiguo de elementos con tamaño fijo, la ubicación conceptual de `i` puede calcularse:

```text
base_address + i × element_size
```

La fórmula es una intuición para O(1): llegar a `i` no exige visitar `0..i-1`. No es una invitación a manipular memoria ni una descripción completa de todos los objetos Python.

### 2.2 Python `list` almacena referencias

A nivel útil, una `list` es un dynamic array contiguo de **referencias**:

```text
list slots
┌─────┬─────┬─────┐
│ ref │ ref │ ref │
└──┬──┴──┬──┴──┬──┘
   │     │     │
   ▼     ▼     ▼
 object object object
```

Los slots tienen tamaño uniforme porque guardan referencias; los objetos apuntados pueden tener tipos y tamaños distintos. La contigüidad del array de referencias no implica que todos los objetos vivan contiguos.

### 2.3 Aliasing sigue existiendo

**Ejemplo ejecutable:**

```python
shared_event = {"id": "evt-001", "tags": ["home"]}
timeline = [shared_event, shared_event]

timeline[0]["tags"].append("arrival")

print(timeline[1]["tags"])
print(timeline[0] is timeline[1])
```

Output:

```text
['home', 'arrival']
True
```

Dos slots contienen la misma referencia. Acceso O(1) no copia el objeto ni elimina mutabilidad, identity o aliasing de PF-M1.

### Predice el costo

¿Qué parte cuesta O(1) en `timeline[0]["tags"]`: localizar el slot, copiar el Event o recorrer todos los tags? Separa las operaciones.

### Detecta el bug

Un equipo cree que poner el mismo Event en dos timelines crea dos snapshots. Dibuja slots y referencias y explica qué mutación refuta esa idea.

---

## 3. Acceso por índice, negativos y slicing

### 3.1 Índice O(1)

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002", "evt-003", "evt-004"]

print(event_ids[2])
print(event_ids[-1])
```

Output:

```text
evt-003
evt-004
```

`items[i]` calcula una posición y recupera una referencia: O(1) respecto de `n = length`. Un índice negativo se traduce respecto del final y conserva costo O(1). Validar límites y recuperar la referencia tienen constantes, por lo que O(1) no significa instantáneo.

### 3.2 Slicing materializa

**Ejemplo ejecutable:**

```python
event_ids = ["evt-001", "evt-002", "evt-003", "evt-004"]
window = event_ids[1:3]

print(window)
print(window is event_ids)
```

Output:

```text
['evt-002', 'evt-003']
False
```

El slice crea otra `list` y copia `k` referencias: O(k) tiempo y O(k) espacio para `k` elementos. Es una shallow copy: no duplica transitivamente los objetos referidos.

### Predice el costo

Compara `events[-1]`, `events[:]` y `events[-100:]` en función de `n` y del tamaño `k` del resultado.

### Modifica

Cambia el ejemplo para que los elementos sean dicts mutables. Muta un dict mediante `window` y comprueba que el contenedor exterior es nuevo, pero sus elementos siguen compartidos.

---

## 4. Dynamic array: length, capacity y append

### 4.1 El problema de crecer

Un array de tamaño fijo no puede añadir una posición sin disponer de espacio. Un **dynamic array** separa:

```text
length   = slots ocupados por elementos lógicos
capacity = slots reservados antes de necesitar otra expansión
```

Puede reservar capacidad extra:

```text
length = 3, capacity conceptual = 5
┌─────┬─────┬─────┬─────┬─────┐
│ ref │ ref │ ref │     │     │
└─────┴─────┴─────┴─────┴─────┘
```

La capacidad no forma parte de la semántica pública habitual de `list`; es un modelo para entender crecimiento. No necesitas ni debes depender de un factor exacto de CPython.

### 4.2 Append amortizado

Cuando hay capacidad, `append` escribe una referencia al final. Cuando no la hay, el runtime puede reservar un bloque mayor y copiar referencias antes de añadir.

```text
muchos append baratos
+
resize ocasional O(n)
↓
O(1) amortizado por append sobre una secuencia
```

No significa que cada llamada individual sea worst-case O(1). Significa que una secuencia de `n` appends tiene costo total O(n) bajo el modelo de dynamic array.

**Ejemplo ejecutable:**

```python
timeline = []
for index in range(4):
    timeline.append(f"evt-{index:03d}")

assert len(timeline) == 4
print(timeline)
```

Output:

```text
['evt-000', 'evt-001', 'evt-002', 'evt-003']
```

El output no revela qué append causó un resize ni cuál fue la capacity. Esas decisiones dependen de implementación y versión; el contrato curricular es la amortización.

### Predice el costo

¿Puede una llamada concreta a `append` mover O(n) referencias? ¿Por qué eso no contradice O(1) amortizado?

### Explica

¿Por qué reservar exactamente un slot nuevo para cada append produciría un costo total desfavorable?

---

## 5. Insertar, eliminar y buscar en una `list`

### 5.1 Inserción desplaza referencias

```text
antes:    A B C D
insert X:     ↑
después:  A B X C D
```

Insertar en `0` puede desplazar `n` referencias: O(n). En medio, el costo depende de cuántos elementos quedan a la derecha y es O(n) worst case. Para añadir al final, `append` expresa mejor la intención y ofrece O(1) amortizado.

### 5.2 Eliminación separa dos costos

- `pop()` del final recupera y retira el último elemento en O(1) amortizado bajo el modelo de dynamic array;
- `pop(i)` o `del items[i]` puede desplazar referencias: O(n) worst case;
- `remove(value)` primero busca por equality y luego desplaza: O(n) + O(n), que se simplifica O(n).

No digas “remove es O(n) porque mueve”: puede pagar scan y movimiento. Nombrar las fases permite elegir otra estructura si una domina.

### 5.3 Membership es linear scan

**Ejemplo ejecutable:**

```python
source_ids = ["src-001", "src-002", "src-003"]

print("src-002" in source_ids)
print("src-999" in source_ids)
```

Output:

```text
True
False
```

`target in items` compara mediante equality hasta encontrar match o agotar la `list`. Best case Θ(1); worst case Θ(n). Si comparar elementos no es O(1), ese costo también importa.

### 5.4 Locality controlada

Recorrer referencias contiguas suele favorecer locality frente a perseguir enlaces dispersos. Pero los objetos referidos pueden estar en otros lugares, y Big O no captura caches, object overhead ni interpreter cost. CS-M10 profundizará hardware; CS-M3 comparará estructuras enlazadas.

### Predice el costo

Analiza por separado `items.remove(target)` cuando el target está primero, último y ausente. ¿Qué parte es scan y qué parte es shift?

### Tradeoff

Si insertas una vez al inicio de una timeline de 20 Events y luego la recorres cien veces, ¿el O(n) de inserción obliga a cambiar la estructura? ¿Qué medirías?

---

## 6. EIDOLON: una timeline pequeña

Una `list` expresa bien una timeline in-memory cuando predominan orden, append y traversal:

**Ejemplo ejecutable:**

```python
recent_events = [
    {"event_id": "evt-001", "text": "Llegué"},
    {"event_id": "evt-002", "text": "Salí"},
]

recent_events.append({"event_id": "evt-003", "text": "Volví"})

print([event["event_id"] for event in recent_events])
```

Output:

```text
['evt-001', 'evt-002', 'evt-003']
```

La posición no es identidad de dominio: insertar otro Event cambia posiciones sin cambiar `event_id`. Si las consultas repetidas por ID dominan, construir un índice puede ser razonable. Si hay una sola consulta sobre 30 elementos, mantener la list puede ser más claro y barato.

### Estructura

Elige `list` o `dict` para: “mostrar en orden todos los Events recientes”. Después elige para: “resolver 50,000 lookups por `event_id` sobre el mismo snapshot”. Justifica desde la operación.

---

## 7. Del scan al hash map

### 7.1 El problema

**Fragmento — scan conocido:**

```python
def find_event(events, event_id):
    for event in events:
        if event["event_id"] == event_id:
            return event
    return None
```

Para `q` consultas worst case, el costo es Θ(qn). Un **hash map** intenta ubicar una entry desde una key sin recorrer necesariamente todas:

```text
key
↓ hash function
hash value
↓ selección de región
bucket / ubicación candidata
↓ equality
entry key → value
```

El hash no produce un índice exacto y único. Keys distintas pueden llegar a la misma región; por eso equality y collision resolution son parte del mecanismo.

### 7.2 Hash function

Una hash function transforma una key en un entero. Para una tabla práctica interesa:

- coherencia con equality;
- estabilidad mientras la key participa en la tabla;
- distribución razonable;
- costo de cálculo razonable.

### 7.3 Hash no es encryption

El hash usado para elegir una ubicación:

- no cifra la key;
- no oculta secretos;
- no demuestra autenticidad;
- no es un ID persistente.

Criptografía no pertenece a CS-M2. Aquí `hash` es una herramienta de indexación in-memory.

### Hashability

¿Por qué `event_id` string puede ser key mientras una `list` de tags no? Responde desde estabilidad y mutabilidad, no desde memorizar tipos.

---

## 8. Hashability, equality y stable keys

### 8.1 El contrato esencial

Las keys de `dict` y los elementos de `set` deben ser hashable. La regla que no puede romperse es:

```text
a == b  ⇒  hash(a) == hash(b)
```

El inverso no se exige: dos objetos con el mismo hash pueden ser distintos; eso es una collision. La tabla debe confirmar equality después de localizar candidatos.

### 8.2 Un value object estable

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class SourceKey:
    namespace: str
    source_id: str


first = SourceKey("journal", "src-001")
second = SourceKey("journal", "src-001")

print(first == second)
print(hash(first) == hash(second))
print({first: "seen"}[second])
```

Output:

```text
True
True
seen
```

La dataclass frozen con fields hashable ofrece value equality y un hash compatible. `frozen=True` no vuelve transitivamente inmutables objetos mutables contenidos; PF-M5 ya estableció ese límite.

### 8.3 Mutabilidad problemática

**Failure case ejecutable — una `list` no es hashable:**

```python
tags = ["home", "arrival"]

try:
    {tags: "event"}
except TypeError:
    print("mutable list rejected as key")
```

Output estable:

```text
mutable list rejected as key
```

Si una key cambiara de manera que alterara hash/equality después de insertarse, un lookup posterior buscaría bajo otra semántica. Python rechaza built-ins mutables comunes como `list`, `dict` y `set` para impedir esa clase de problema.

### 8.4 Contrato incoherente

**Código incorrecto ejecutable — solo para diagnóstico:**

```python
class BadKey:
    def __init__(self, code):
        self.code = code

    def __eq__(self, other):
        return isinstance(other, BadKey)

    def __hash__(self):
        return self.code


left = BadKey(1)
right = BadKey(2)

print(left == right)
print(hash(left) == hash(right))
```

Output:

```text
True
False
```

`BadKey` afirma que todos sus objetos son iguales pero devuelve hashes distintos. La class viola el contrato antes de entrar en una colección; el comportamiento de lookup ya no es fiable y no debe fijarse un output accidental de `dict`. Corrección: derivar equality y hash de los mismos fields estables o no hacer la class hashable.

### Stable key

Elige entre `event_id`, display name, texto completo y timestamp local ambiguo. ¿Qué propiedades necesita la identidad elegida?

---

## 9. Buckets y collisions

### 9.1 Bucket como modelo

Un **bucket** es una región conceptual donde la tabla busca entries candidatas. Una representación didáctica con cuatro buckets:

```text
0 → []
1 → [("evt-001", event_a)]
2 → [("evt-002", event_b), ("evt-010", event_c)]  ← collision
3 → []
```

Dos keys diferentes pueden compartir bucket. Eso no implica perder una: collision resolution conserva varias entries y equality distingue la correcta.

### 9.2 Separate chaining

En **separate chaining**, cada bucket contiene una pequeña secuencia de entries. Lookup:

1. calcula bucket;
2. recorre la chain;
3. compara keys por equality;
4. devuelve el value o informa ausencia.

Esta estrategia es pedagógica. **No presupongas que Python `dict` usa separate chaining ni que su layout coincide con nuestros buckets.** El contrato de `dict` importa más que sus internals variables.

### Collision

Si dos keys caen en el mismo bucket, ¿qué dato adicional necesita la tabla para elegir la entry correcta? ¿Por qué el hash solo no basta?

---

## 10. Hash map didáctico con separate chaining

El siguiente modelo acepta keys `str`, usa una hash function determinista deliberadamente pequeña y no implementa delete ni resizing. Su propósito es observar insert, update, get, collision y missing key.

**Modelo educativo ejecutable:**

```python
class SimpleHashMap:
    def __init__(self, bucket_count=4, hash_function=None):
        if bucket_count <= 0:
            raise ValueError("bucket_count must be positive")
        self._buckets = [[] for _ in range(bucket_count)]
        self._hash_function = hash_function or self._educational_hash
        self._size = 0

    @staticmethod
    def _educational_hash(key):
        if not isinstance(key, str):
            raise TypeError("SimpleHashMap keys must be str")
        return sum(ord(character) for character in key)

    def _bucket(self, key):
        hash_value = self._hash_function(key)
        return self._buckets[hash_value % len(self._buckets)]

    def put(self, key, value):
        bucket = self._bucket(key)
        for index, (stored_key, _) in enumerate(bucket):
            if stored_key == key:
                bucket[index] = (key, value)
                return
        bucket.append((key, value))
        self._size += 1

    def get(self, key, default=None):
        bucket = self._bucket(key)
        for stored_key, value in bucket:
            if stored_key == key:
                return value
        return default

    def __len__(self):
        return self._size

    def bucket_sizes(self):
        return [len(bucket) for bucket in self._buckets]


events = SimpleHashMap(bucket_count=2)
events.put("ab", "first")
events.put("ba", "second")  # same character sum: deliberate collision
events.put("ab", "updated")

print(events.get("ab"))
print(events.get("ba"))
print(events.get("missing"))
print(len(events))
print(sorted(events.bucket_sizes()))
```

Output:

```text
updated
second
None
2
[0, 2]
```

`"ab"` y `"ba"` producen la misma suma y llegan al mismo bucket, pero ambas entries sobreviven. Actualizar `"ab"` no aumenta size. `None` representa missing solo porque ese es el contrato elegido; una API real debe decidir si `None` puede ser value válido.

### Qué ocurre internamente

Con `b` buckets y una chain de longitud `c`, lookup inspecciona hasta `c` entries. Si las entries se distribuyen bien, `c` suele mantenerse pequeño. Si todas caen en un bucket, `c = n` y lookup se vuelve lineal.

### Modifica

Cambia `bucket_count` a 1. Predice `bucket_sizes()` y explica qué costo se degrada sin que la tabla deje de ser correcta.

### Collision

Agrega una tercera key con la misma suma de caracteres. Comprueba que `len` aumenta y que todas siguen recuperables.

---

## 11. Expected O(1), worst O(n) y mala distribución

Decir “hash lookup es O(1)” sin calificar oculta supuestos.

```text
expected practical case
hashing razonable + distribución + resizing
→ chain/probes acotados en promedio
→ expected O(1)

worst case
muchas collisions en la misma región
→ revisar hasta n entries
→ O(n)
```

El expected case depende del modelo de inputs y de la implementación. No se obtiene promediando arbitrariamente best y worst. CS-M1 ya estableció esa disciplina.

### Mala hash function

**Modelo educativo ejecutable:**

```python
def always_zero(_key):
    return 0


buckets = [[] for _ in range(8)]
for index in range(5):
    key = f"evt-{index}"
    bucket_index = always_zero(key) % len(buckets)
    buckets[bucket_index].append((key, index))

found = None
for stored_key, value in buckets[always_zero("evt-4")]:
    if stored_key == "evt-4":
        found = value
        break

print([len(bucket) for bucket in buckets])
print(found)
```

Output:

```text
[5, 0, 0, 0, 0, 0, 0, 0]
4
```

La tabla conserva corrección gracias a chaining, pero perdió la ventaja de dispersión. Este resultado describe `SimpleHashMap`, no internals de `dict`.

### Predice el costo

Con `n` entries y `always_zero`, deriva lookup de una key ausente. ¿Qué operación domina?

---

## 12. Load factor y resizing

### 12.1 Load factor

En un modelo con `n` entries y `b` buckets:

```text
load factor α = n / b
```

Es una razón práctica, no una promesa de chain exacta. Si `α` crece demasiado, aumentan candidatos, collisions o probes según la estrategia. Implementaciones reales administran capacidad para conservar rendimiento esperado.

### 12.2 Resizing conceptual

```text
tabla pequeña
↓ crecen entries
allocate tabla mayor
↓
redistribuir/reinsertar entries
↓
tabla con más capacidad
```

Redistribuir `n` entries cuesta O(n) ocasionalmente. Con una política adecuada, muchas inserciones baratas absorben esos eventos y la inserción puede conservar expected amortized O(1). No se fija threshold, factor de crecimiento ni algoritmo de CPython.

`SimpleHashMap` no redimensiona: esa omisión permite observar degradación, pero impide usarla como estructura profesional.

### Predice el costo

Si duplicas entries sin cambiar buckets, ¿qué ocurre con `α`? ¿Por qué `α` no predice por sí solo el worst case de una hash function adversarial?

### Explica

¿Cómo puede una inserción ocasional O(n) coexistir con inserción expected amortized O(1)?

---

## 13. Python `dict`: contrato profesional

`dict` implementa una asociación key → value con hashing administrado por Python. Usa el modelo anterior para razonar, no para adivinar su layout.

**Ejemplo ejecutable:**

```python
events_by_id = {}
events_by_id["evt-002"] = {"text": "Salí"}
events_by_id["evt-001"] = {"text": "Llegué"}
events_by_id["evt-002"] = {"text": "Salí de casa"}

print(events_by_id["evt-002"]["text"])
print("evt-001" in events_by_id)
print(list(events_by_id))
```

Output:

```text
Salí de casa
True
['evt-002', 'evt-001']
```

Semántica y costo práctico:

- insertion/update por key: expected amortized O(1);
- lookup y membership de keys: expected practical O(1), worst O(n);
- iteration: O(n);
- memoria: O(n) adicional para entries/capacity.

Estos costos tratan hash/equality de cada key como trabajo acotado. Si el tamaño de la key también crece, debes declarar esa dimensión y añadir su costo, como enseñó CS-M1.

### 13.1 Insertion order no es sorted order

Python moderno garantiza iteration en insertion order. Actualizar una key existente conserva su posición. Eso no ordena lexicográficamente keys: arriba, `evt-002` aparece antes de `evt-001`.

Si necesitas orden por key, `sorted(events_by_id)` materializa y ordena keys con costo adicional; sorting formal pertenece a CS-M4. No llames “ordenado por key” a un dict solo porque su iteration sea estable.

### Predice

¿Qué orden esperas después de borrar una key y volver a insertarla? Compruébalo, pero distingue esa semántica de una estructura sorted.

---

## 14. Dict como índice derivado EIDOLON

### 14.1 Build explícito y duplicate policy

Una comprehension es compacta, pero una duplicate key conserva silenciosamente el último value. Si duplicate `event_id` viola el contrato, el build debe detectarlo.

**Ejemplo ejecutable:**

```python
def build_event_index(events):
    events_by_id = {}
    for event in events:
        event_id = event["event_id"]
        if event_id in events_by_id:
            raise ValueError(f"duplicate event_id: {event_id}")
        events_by_id[event_id] = event
    return events_by_id


events = [
    {"event_id": "evt-001", "text": "Llegué"},
    {"event_id": "evt-002", "text": "Salí"},
]

events_by_id = build_event_index(events)

assert events_by_id["evt-001"] is events[0]
print(events_by_id["evt-002"]["text"])
```

Output:

```text
Salí
```

Build visita `n` Events: expected Θ(n). El dict usa Θ(n) espacio adicional y lookup expected O(1). Conserva referencias a Events; no duplica todos sus objetos.

### 14.2 Failure case: overwrite silencioso

**Código incorrecto ejecutable:**

```python
events = [
    {"event_id": "evt-001", "text": "original"},
    {"event_id": "evt-001", "text": "replacement"},
]

unsafe_index = {event["event_id"]: event for event in events}

print(len(unsafe_index))
print(unsafe_index["evt-001"]["text"])
```

Output:

```text
1
replacement
```

No falló `dict`; falló la política. El síntoma es pérdida silenciosa del primer record. La corrección es validar antes de insertar y conservar el duplicate para diagnosticar source, posición y ID.

### Detecta corrupción

Escribe un test que exija `ValueError` para duplicate `event_id` y compruebe que el input original no fue modificado.

---

## 15. Diseñar stable keys y composite keys

Una key debe representar identidad estable durante la vida relevante del índice.

| Candidato | Riesgo |
|---|---|
| `event_id` explícito | apropiado si es único, estable y validado |
| `source_id` explícito | apropiado para identidad de source bajo contrato |
| display name | puede cambiar y no ser único |
| texto completo | mutable, voluminoso y no necesariamente único |
| timestamp local ambiguo | collisions semánticas/interpretación |
| `id(object)` | identidad runtime, no persistente |
| `hash(text)` | valor runtime, no domain ID estable |

Cuando la identidad requiere varios componentes, una tuple hashable puede expresarla:

**Ejemplo ejecutable:**

```python
events_by_person_and_id = {
    ("person-001", "evt-001"): "arrival",
    ("person-002", "evt-001"): "departure",
}

print(events_by_person_and_id[("person-002", "evt-001")])
```

Output:

```text
departure
```

El composite key no introduce database design. Solo declara que ambos componentes participan en la identidad in-memory.

### Stable key

Un Event cambia de title. Si el índice usa title, ¿qué debe actualizarse y qué bug aparece si se olvida? Compara con `event_id` inmutable.

---

## 16. `set`: membership y unicidad

### 16.1 El problema que expresa

Un `set` responde:

```text
¿ya vi este ID?
¿pertenece este tag?
¿qué valores únicos existen?
```

No expresa posición ni duplicates. Membership comparte el modelo de hashing de `dict`: expected practical O(1), worst O(n) bajo degradación.

**Ejemplo ejecutable:**

```python
seen_source_ids = set()
for source_id in ["src-001", "src-002", "src-001"]:
    seen_source_ids.add(source_id)

print("src-002" in seen_source_ids)
print(sorted(seen_source_ids))
```

Output:

```text
True
['src-001', 'src-002']
```

`sorted` se usa únicamente para presentar output determinista. El orden de iteration de `set` no es contrato de negocio.

### 16.2 Operaciones de conjuntos

**Ejemplo ejecutable:**

```python
active_tags = {"home", "private", "arrival"}
requested_tags = {"home", "work"}

print(sorted(active_tags | requested_tags))
print(sorted(active_tags & requested_tags))
print(sorted(active_tags - requested_tags))
print({"home"} <= active_tags)
```

Output:

```text
['arrival', 'home', 'private', 'work']
['home']
['arrival', 'private']
True
```

Union reúne; intersection conserva elementos comunes; difference quita los presentes en el otro conjunto; subset comprueba inclusión. El análisis formal de lógica/conjuntos llega en CS-M6.

### 16.3 `frozenset`

`frozenset` ofrece semántica set-like inmutable. Si todos sus elementos son hashable, también puede ser key:

**Ejemplo ejecutable:**

```python
normalized_tags = frozenset({"home", "arrival"})
labels = {normalized_tags: "arrival-at-home"}

print(labels[frozenset({"arrival", "home"})])
```

Output:

```text
arrival-at-home
```

Úsalo cuando el conjunto como valor estable mejora el modelo, no para reemplazar cada `set` mutable.

### Hashability

¿Puede un `set` contener una `list`, otro `set` o un `frozenset` de strings? Explica cada respuesta.

### Source of truth

Si `seen_source_ids` se pierde al terminar el proceso, ¿qué fuente permite reconstruirlo?

---

## 17. Elegir entre `list`, `dict` y `set`

Primero razona; después consulta la tabla.

1. ¿Necesitas posición/orden, asociación o solo membership?
2. ¿Deben preservarse duplicates?
3. ¿Cuántas veces se repetirá la operación?
4. ¿la estructura ya existe o hay que construirla?
5. ¿qué memoria y maintenance agrega?
6. ¿qué fuente reconstruye la proyección?

| Necesidad dominante | Estructura probable | Costo comprado |
|---|---|---|
| traversal ordenado, duplicates, acceso por posición | `list` | membership lineal; shifts |
| stable key → value y lookups repetidos | `dict` | build, memoria, key/invariant maintenance |
| membership y unicidad | `set` | build, memoria, sin orden contractual |
| conjunto inmutable/hashable | `frozenset` | nueva materialización; no mutación incremental |

### 17.1 List puede seguir ganando

Para `n=8`, una consulta y orden requerido, convertir a set o construir dict añade trabajo y pierde semántica. Mejor expected Big O de lookup no decide el costo total.

### 17.2 Dict no reemplaza la timeline

Muchas veces conviven:

```text
ordered_events: list    → orden/traversal
events_by_id: dict      → lookup por identity
```

Son dos vistas, no dos autoridades.

### Estructura

Elige para: tags con duplicates significativos; tags únicos sin orden; Event por ID; timeline con orden. Explica qué información perdería la alternativa.

### Tradeoff

¿Construirías un set para una sola prueba de membership sobre diez elementos? ¿Qué cambia con un millón de pruebas sobre el mismo snapshot?

---

## 18. Una autoridad y proyecciones reconstruibles

### 18.1 Rebuild

```text
source journal
↓ replay
ordered_events: list
events_by_id: dict
seen_source_ids: set
```

En P0, el journal/source es autoridad. Las tres estructuras viven en memoria, desaparecen con el proceso y deben poder descartarse y reconstruirse.

**Ejemplo ejecutable:**

```python
def rebuild_indexes(source_events):
    ordered_events = []
    events_by_id = {}
    seen_source_ids = set()

    for source_event in source_events:
        event_id = source_event["event_id"]
        source_id = source_event["source_id"]

        if event_id in events_by_id:
            raise ValueError(f"duplicate event_id: {event_id}")

        event = source_event.copy()
        ordered_events.append(event)
        events_by_id[event_id] = event
        seen_source_ids.add(source_id)

    return ordered_events, events_by_id, seen_source_ids


source_events = [
    {"event_id": "evt-001", "source_id": "src-001"},
    {"event_id": "evt-002", "source_id": "src-002"},
]

ordered, by_id, seen = rebuild_indexes(source_events)

assert ordered[0] is by_id["evt-001"]
assert ordered[0] is not source_events[0]
print([event["event_id"] for event in ordered])
print(sorted(seen))
```

Output:

```text
['evt-001', 'evt-002']
['src-001', 'src-002']
```

La copia superficial evita mutar los dicts exteriores del source en este modelo. Los índices apuntan a los mismos Event derivados para no crear dos estados divergentes dentro de la proyección.

### 18.2 Maintenance cost

Cada nuevo índice añade:

- build y memoria;
- actualización ante nuevos Events;
- duplicate/invariant checks;
- riesgo de stale data;
- tests de rebuild/equivalencia.

No crees un índice por cada consulta imaginable. Usa query pattern → modelo de costo → benchmark → trigger.

### Source of truth

Si `ordered_events` y `events_by_id` discrepan, ¿cuál corriges manualmente? La respuesta correcta debe partir de reconstruir ambos desde source, no escoger una proyección como autoridad accidental.

---

## 19. Memory overhead y medición prudente

Una `list` guarda un array de referencias. `dict` y `set` reservan metadata/capacidad para hashing además de referencias. Por eso pueden consumir considerablemente más memoria; no existe una cifra universal por elemento.

`sys.getsizeof(container)` informa tamaño superficial de ese objeto, no todo el object graph. `tracemalloc` permite comparar allocations Python dentro de una región, pero tampoco equivale automáticamente a RSS o memoria de todos los buffers nativos. CS-M1 ya explicó estas fronteras.

### Tradeoff

Un índice reduce lookup y aumenta working set. ¿Qué medirías para decidir si conservarlo durante todo el proceso o reconstruirlo solo para un batch?

---

## 20. Benchmark: list scan frente a dict lookup

La comparación útil separa:

```text
dataset generation
scan de una consulta
q scans
dict build
una consulta indexada
q consultas indexadas
memoria adicional del índice
```

**Benchmark ejecutable:**

```python
from statistics import median
from timeit import repeat
import tracemalloc


def make_events(n):
    return [
        {"event_id": f"evt-{index:07d}", "source_id": f"src-{index:07d}"}
        for index in range(n)
    ]


def find_event(events, event_id):
    for event in events:
        if event["event_id"] == event_id:
            return event
    return None


def build_event_index(events):
    result = {}
    for event in events:
        event_id = event["event_id"]
        if event_id in result:
            raise ValueError(f"duplicate event_id: {event_id}")
        result[event_id] = event
    return result


def median_seconds(action):
    return median(repeat(action, repeat=5, number=1))


def index_memory(events):
    tracemalloc.start()
    before, _ = tracemalloc.get_traced_memory()
    index = build_event_index(events)
    current, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    assert len(index) == len(events)
    return current - before, peak - before


def benchmark_lookup(n, q):
    events = make_events(n)
    targets = [f"evt-{((position * 7_919) % n):07d}" for position in range(q)]
    events_by_id = build_event_index(events)

    scan_results = lambda: [find_event(events, target) for target in targets]
    dict_results = lambda: [events_by_id.get(target) for target in targets]
    assert scan_results() == dict_results()

    scan_seconds = median_seconds(scan_results)
    build_seconds = median_seconds(lambda: build_event_index(events))
    lookup_seconds = median_seconds(dict_results)
    retained, peak = index_memory(events)

    return {
        "n": n,
        "q": q,
        "scan_seconds": scan_seconds,
        "build_seconds": build_seconds,
        "lookup_seconds": lookup_seconds,
        "indexed_total": build_seconds + lookup_seconds,
        "retained_bytes": retained,
        "peak_bytes": peak,
    }


row = benchmark_lookup(n=1_000, q=100)

assert row["scan_seconds"] >= 0.0
assert row["indexed_total"] >= row["lookup_seconds"]
assert row["peak_bytes"] >= row["retained_bytes"] >= 0
print("lookup_results_equal=True")
print("build_included=True")
print("memory_relation_valid=True")
```

Output estable:

```text
lookup_results_equal=True
build_included=True
memory_relation_valid=True
```

Los valores ambientales permanecen en `row`. Para una sola query compara scan con `build + lookup` si el índice nace y muere en esa operación. Si el índice ya existe por un workload mayor, mide lookup aislado como otra pregunta. Ejecuta tamaños 10²–10⁵ y 10⁶ solo si el equipo lo permite.

### Benchmark

¿Qué variables deben permanecer iguales entre scan y dict: targets, distribución, Events, Python y región medida? ¿Qué observación define el break-even local?

### Interpreta

Si lookup aislado gana desde `n=100`, pero `build + lookup` pierde cuando `q=1`, ¿qué afirmación sería correcta y cuál exageraría la evidencia?

---

## 21. Benchmark: list membership frente a set membership

Convertir una list a set también tiene build O(n) expected y espacio O(n). Una consulta no amortiza automáticamente ese costo.

**Benchmark ejecutable:**

```python
from statistics import median
from timeit import repeat


def median_seconds(action):
    return median(repeat(action, repeat=5, number=1))


def benchmark_membership(n, q):
    source_ids = [f"src-{index:07d}" for index in range(n)]
    targets = [source_ids[-1]] * q

    list_action = lambda: [target in source_ids for target in targets]
    seen_ids = set(source_ids)
    set_action = lambda: [target in seen_ids for target in targets]

    assert list_action() == set_action()

    list_seconds = median_seconds(list_action)
    build_seconds = median_seconds(lambda: set(source_ids))
    set_seconds = median_seconds(set_action)

    return list_seconds, build_seconds, set_seconds


for q in (1, 100):
    list_seconds, build_seconds, set_seconds = benchmark_membership(1_000, q)
    assert list_seconds >= 0.0
    assert build_seconds >= 0.0
    assert set_seconds >= 0.0

print("query_counts_checked=[1, 100]")
print("membership_results_equal=True")
```

Output estable:

```text
query_counts_checked=[1, 100]
membership_results_equal=True
```

No se imprime un ganador. El propósito es comparar `q=1` y repetición. La distribución elegida usa un target al final, worst case para la list; también debes medir inicio y ausencia antes de generalizar.

### Benchmark

¿Por qué construir `seen_ids` fuera del timer de membership aislado y medir build por separado produce dos preguntas útiles?

### Tradeoff

Si el contrato necesita duplicates y orden para output, ¿puedes reemplazar la list por set? Diseña cómo podrían convivir sin crear dos autoridades.

---

## 22. Benchmark didáctico de collisions

El tiempo local puede ser ruidoso. Para observar el mecanismo, primero cuenta comparaciones dentro de la chain.

**Modelo educativo ejecutable:**

```python
class MeasuredHashMap:
    def __init__(self, bucket_count, hash_function):
        self._buckets = [[] for _ in range(bucket_count)]
        self._hash_function = hash_function

    def _bucket(self, key):
        index = self._hash_function(key) % len(self._buckets)
        return self._buckets[index]

    def put(self, key, value):
        bucket = self._bucket(key)
        for index, (stored_key, _) in enumerate(bucket):
            if stored_key == key:
                bucket[index] = (key, value)
                return
        bucket.append((key, value))

    def get_with_steps(self, key, default=None):
        bucket = self._bucket(key)
        steps = 0
        for stored_key, value in bucket:
            steps += 1
            if stored_key == key:
                return value, steps
        return default, steps


def numeric_suffix(key):
    return int(key.split("-")[1])


def always_zero(_key):
    return 0


distributed = MeasuredHashMap(16, numeric_suffix)
colliding = MeasuredHashMap(16, always_zero)

for index in range(16):
    key = f"evt-{index}"
    distributed.put(key, index)
    colliding.put(key, index)

distributed_value, distributed_steps = distributed.get_with_steps("evt-15")
colliding_value, colliding_steps = colliding.get_with_steps("evt-15")

assert distributed_value == colliding_value == 15
print(distributed_steps)
print(colliding_steps)
```

Output:

```text
1
16
```

Con este dataset controlado, una hash dispersa entrega una entry por bucket; `always_zero` obliga a recorrer 16. El experimento valida el modelo educativo, no demuestra cómo colisiona `dict` ni permite inferir thresholds de CPython.

### Collision

Busca una key ausente en ambas tablas. Predice steps y explica por qué el worst case de la tabla colisionada es O(n).

---

## 23. Hash randomization, persistencia y cache

### 23.1 `hash()` no es un ID persistente

Para tipos como `str` y `bytes`, Python puede randomizar hashes entre procesos. No fijes outputs de `hash("evt-001")` ni los guardes como identidad estable.

**Código incorrecto — no ejecutes como diseño:**

```python
persistent_event_id = hash(event_text)
```

El valor puede cambiar entre ejecuciones, puede colisionar y su propósito es runtime hashing. Usa un domain ID explícito y validado.

### 23.2 Hash map no es persistence

Un `dict`:

- vive en memoria;
- desaparece cuando termina el proceso;
- no ofrece durability ni crash recovery;
- no reemplaza el source journal.

Storage persistente futuro es otro problema y queda fuera de CS-M2.

### 23.3 Índice no equivale automáticamente a cache

Un derived index organiza el mismo snapshot para consultas. Llamarlo cache introduce preguntas de freshness, invalidation y ownership. Esas policies no se desarrollan aquí. Nombra la estructura según el contrato real.

### Explica

Contrasta `event_id`, `id(event)` y `hash(event_id)` respecto de estabilidad, identidad de dominio y vida del proceso.

---

## 24. Catálogo de failure cases

### 24.1 List para lookup repetido

`q` scans sobre `n` Events cuestan worst Θ(qn). Corrección posible: medir build Θ(n), lookups expected O(q) y memoria Θ(n) de un índice.

### 24.2 Dict para una sola consulta

Build + lookup puede hacer más trabajo y usar más memoria que un scan. Corrección: incluir el costo completo y el workload real.

### 24.3 Key mutable o derivada de un campo cambiante

El índice conserva la key antigua o rompe la relación hash/equality. Corrección: stable domain ID y rebuild/update explícito.

### 24.4 Duplicate key silenciosa

Una comprehension retiene el último value. Corrección: comprobar membership antes de insertar, rechazar el duplicate y conservar evidencia diagnóstica.

### 24.5 `hash()` como persistent ID

Confunde runtime hashing con identity. Corrección: ID explícito independiente de proceso.

### 24.6 Confiar en set order

**Failure case ejecutable — el orden mostrado no se fija:**

```python
seen = {"src-003", "src-001", "src-002"}
print(seen)
```

El proceso imprime los tres IDs, pero su orden no forma parte del contrato y puede variar entre ejecuciones/configuraciones. Corrección: `sorted(seen)` o una regla determinista explícita en la frontera de presentación. Conserva Python/version y `PYTHONHASHSEED` si diagnosticas una diferencia; no conviertas el orden observado en requisito.

### 24.7 Hash table sin collision handling

**Código incorrecto ejecutable — una entry por bucket:**

```python
class BrokenHashMap:
    def __init__(self, bucket_count=2):
        self._slots = [None] * bucket_count

    @staticmethod
    def _hash(key):
        return sum(ord(character) for character in key)

    def put(self, key, value):
        index = self._hash(key) % len(self._slots)
        self._slots[index] = (key, value)

    def get(self, key):
        index = self._hash(key) % len(self._slots)
        entry = self._slots[index]
        if entry is not None and entry[0] == key:
            return entry[1]
        return None


broken = BrokenHashMap()
broken.put("ab", "first")
broken.put("ba", "second")

print(broken.get("ab"))
print(broken.get("ba"))
```

Output:

```text
None
second
```

`"ab"` y `"ba"` colisionan; la segunda inserción sobrescribe el único slot y pierde la primera entry. Corrección: chaining u otra política que conserve varias entries candidatas y compare sus keys. `SimpleHashMap` ya muestra esa corrección.

### 24.8 Mala hash function

`return 0` conserva corrección con chaining, pero degrada lookup a O(n). Corrección: función/distribución apropiadas; en producción usa `dict`, no una hash casera.

### 24.9 Equality/hash incoherentes

Objetos iguales con hashes distintos pueden aparecer como entries separadas. Corrección: ambos contratos derivan de los mismos fields estables.

### 24.10 Dict insertion order como sorted order

Iteration estable no significa orden por key. Corrección: declarar orden requerido y pagar la operación correspondiente.

### 24.11 Dos fuentes de verdad

Mutar list e índice por paths diferentes produce divergencia. Corrección: source autoritativo, una función de rebuild y tests de equivalencia.

### 24.12 Medir lookup sin build

Presentar lookup aislado como costo de una command que crea el índice exagera la mejora. Corrección: reportar aislado y total con fronteras explícitas.

### Diagnostica

Elige cinco casos y documenta: input, síntoma, causa, propiedad violada, evidencia que conservarías y corrección mínima.

---

## 25. Caso progresivo integrado: Event Index in-memory

El caso combina orden, lookup y unicidad sin convertir las proyecciones en source.

### 25.1 Contrato

- `source_events` permanece sin mutar;
- `ordered_events` preserva orden de source;
- `events_by_id` rechaza duplicate `event_id`;
- `seen_source_ids` deduplica source IDs para membership;
- todas las proyecciones se reconstruyen desde source;
- list y dict apuntan al mismo Event derivado por ID.

**Ejemplo ejecutable:**

```python
def rebuild_event_state(source_events):
    ordered_events = []
    events_by_id = {}
    seen_source_ids = set()
    repeated_source_ids = set()

    for source_position, source_event in enumerate(source_events):
        event_id = source_event["event_id"]
        source_id = source_event["source_id"]

        if event_id in events_by_id:
            raise ValueError(
                f"duplicate event_id at position {source_position}: {event_id}"
            )
        if source_id in seen_source_ids:
            repeated_source_ids.add(source_id)

        event = source_event.copy()
        ordered_events.append(event)
        events_by_id[event_id] = event
        seen_source_ids.add(source_id)

    return {
        "ordered_events": ordered_events,
        "events_by_id": events_by_id,
        "seen_source_ids": seen_source_ids,
        "repeated_source_ids": sorted(repeated_source_ids),
    }


source_events = [
    {"event_id": "evt-001", "source_id": "src-a", "text": "Llegué"},
    {"event_id": "evt-002", "source_id": "src-a", "text": "Salí"},
    {"event_id": "evt-003", "source_id": "src-b", "text": "Volví"},
]

state = rebuild_event_state(source_events)

assert [event["event_id"] for event in state["ordered_events"]] == [
    "evt-001", "evt-002", "evt-003"
]
assert state["events_by_id"]["evt-002"]["text"] == "Salí"
assert state["ordered_events"][1] is state["events_by_id"]["evt-002"]
assert state["ordered_events"][0] is not source_events[0]
assert state["seen_source_ids"] == {"src-a", "src-b"}
assert state["repeated_source_ids"] == ["src-a"]

print("ordered=True")
print("lookup=True")
print("source_unchanged=True")
print("seen_source_ids=" + repr(sorted(state["seen_source_ids"])))
print("repeated_source_ids=" + repr(state["repeated_source_ids"]))
```

Output:

```text
ordered=True
lookup=True
source_unchanged=True
seen_source_ids=['src-a', 'src-b']
repeated_source_ids=['src-a']
```

Repetir `source_id` es válido en este contrato: varios Events pueden provenir de la misma source; el set registra sources únicas y el resumen demuestra que la repetición fue detectada. Duplicate `event_id` es corrupción y aborta rebuild. Una política diferente debe declararse, no inferirse del tipo `set`.

### 25.2 Rebuild determinista

Ejecutar la función dos veces sobre el mismo source produce estructuras exteriores nuevas con values iguales y orden equivalente. El test no compara identity entre rebuilds, salvo para confirmar que no reutilizan estado accidental.

### 25.3 Costo

Con expected hashing:

```text
rebuild time: expected Θ(n)
projections space: Θ(n)
lookup by ID: expected O(1)
source membership: expected O(1)
ordered traversal: Θ(n)
```

### Source of truth

Diseña un test que descarte `state`, lo reconstruya y obtenga los mismos IDs, sources y lookup results desde `source_events`.

---

## 26. Ejercicios guiados con solución razonada

Escribe primero tu predicción. Después sigue cada derivación paso por paso y ejecuta el ejemplo asociado cuando corresponda.

### Ejercicio guiado 1 — Access, append e insert

**Objetivo:** separar tres operaciones del dynamic array.  
**Input:** `items[i]`, `items.append(x)`, `items.insert(0, x)`.  
**Predice:** costo respecto de `n`.  
**Solución razonada:** (1) índice calcula un slot: O(1); (2) append usa capacity o resize ocasional: O(1) amortizado; (3) insertar al inicio desplaza `n` referencias: O(n).  
**Criterio:** no describir toda `list` con un único costo.

### Ejercicio guiado 2 — Membership en list

**Objetivo:** relacionar membership con equality.  
**Input:** target primero, último y ausente.  
**Predice:** best/worst.  
**Solución razonada:** (1) primero requiere una comparación, Θ(1); (2) último/ausente requieren hasta `n`, Θ(n); (3) si equality cuesta O(m), el worst puede ser O(nm).  
**Criterio:** declarar posición/distribución y costo de equality.

### Ejercicio guiado 3 — Construir índice dict

**Objetivo:** derivar build, lookup y space.  
**Input:** `n` Events con IDs únicos.  
**Predice:** costo expected.  
**Solución razonada:** (1) visitar `n`: Θ(n); (2) cada insert expected amortized O(1); (3) build expected Θ(n); (4) lookup expected O(1); (5) índice usa Θ(n) space.  
**Criterio:** incluir build y memoria.

### Ejercicio guiado 4 — Duplicate keys

**Objetivo:** evitar overwrite silencioso.  
**Input:** dos Events con `event_id="evt-001"`.  
**Predice:** resultado de comprehension.  
**Solución razonada:** (1) la segunda asignación usa la misma key; (2) reemplaza el value; (3) length queda 1; (4) el build seguro debe comprobar membership antes de insertar y fallar con posición/ID.  
**Criterio:** el test debe observar el duplicate, no solo length final.

### Ejercicio guiado 5 — Stable key

**Objetivo:** separar identity de display data.  
**Input:** `event_id` estable y title editable.  
**Predice:** qué índice queda stale tras cambiar title.  
**Solución razonada:** (1) title participa en su key; (2) al mutar, la entry sigue bajo el valor anterior salvo update; (3) `event_id` evita acoplar lookup a presentation; (4) valida unicidad/estabilidad del ID.  
**Criterio:** justificar desde dominio, no solo hashability.

### Ejercicio guiado 6 — Hash map didáctico

**Objetivo:** trazar put/get.  
**Input:** `SimpleHashMap(4)` y key `"ab"`.  
**Predice:** hash, bucket conceptual y comparación.  
**Solución razonada:** (1) suma code points; (2) aplica módulo 4; (3) recorre chain; (4) compara key por equality; (5) inserta o devuelve.  
**Criterio:** no llamar al hash “ubicación única”.

### Ejercicio guiado 7 — Provocar collision

**Objetivo:** distinguir hash de equality.  
**Input:** `"ab"` y `"ba"`.  
**Predice:** mismo hash educativo, keys distintas.  
**Solución razonada:** (1) ambas sumas coinciden; (2) caen en el mismo bucket; (3) equality es false; (4) chaining conserva dos entries.  
**Criterio:** recuperar ambos values.

### Ejercicio guiado 8 — Resolver collision

**Objetivo:** explicar separate chaining.  
**Input:** bucket con tres entries.  
**Predice:** lookup de segunda y ausente.  
**Solución razonada:** (1) hash selecciona bucket; (2) lookup compara en orden hasta match; (3) la segunda requiere dos comparaciones; (4) ausente revisa toda la chain.  
**Criterio:** collision no produce overwrite.

### Ejercicio guiado 9 — Mala hash

**Objetivo:** derivar degradación.  
**Input:** `always_zero`, `n` keys.  
**Predice:** bucket sizes y lookup ausente.  
**Solución razonada:** (1) todas van a bucket 0; (2) chain length es `n`; (3) lookup ausente compara `n`; (4) worst time Θ(n), aunque la tabla siga correcta.  
**Criterio:** no extrapolar a `dict` internals.

### Ejercicio guiado 10 — Set para deduplicación

**Objetivo:** elegir semántica.  
**Input:** source IDs con repeats válidos.  
**Predice:** contenido único.  
**Solución razonada:** (1) add calcula hash; (2) equality evita otra entry equivalente; (3) membership expected O(1); (4) output determinista requiere `sorted`.  
**Criterio:** no depender de set order.

### Ejercicio guiado 11 — List vs set membership

**Objetivo:** incluir conversion cost.  
**Input:** `n` IDs y `q` queries.  
**Predice:** worst list O(qn) frente a build expected Θ(n) + queries expected O(q).  
**Solución razonada:** (1) lista escanea por query; (2) set visita `n` al construir; (3) luego hace `q` lookups expected; (4) `q=1` puede favorecer list; (5) `q` grande puede amortizar build.  
**Criterio:** comparar estrategia total.

### Ejercicio guiado 12 — Rebuild desde source

**Objetivo:** preservar autoridad.  
**Input:** source Events y proyecciones descartadas.  
**Predice:** qué datos deben reproducirse.  
**Solución razonada:** (1) recorre source en orden; (2) valida duplicates; (3) reconstruye list/dict/set; (4) compara IDs, lookups y seen set; (5) ninguna proyección se lee como source.  
**Criterio:** rebuild funciona desde estado vacío.

### Ejercicio guiado 13 — Tiempo y memoria

**Objetivo:** diseñar comparación defendible.  
**Input:** scan vs dict para `n/q` crecientes.  
**Predice:** tendencia y memory tradeoff.  
**Solución razonada:** (1) genera dataset fuera; (2) usa mismos targets; (3) mide scan, build y indexed queries separados; (4) suma total; (5) usa múltiples repeticiones; (6) mide allocations del índice aisladamente.  
**Criterio:** reportar resultados locales y límites.

### Ejercicio guiado 14 — Migration trigger

**Objetivo:** convertir medición en decisión.  
**Input:** scan actual sin umbral.  
**Predice:** información faltante.  
**Solución razonada:** (1) define operation, `n/q`, distribución y environment; (2) elige latencia/memoria; (3) fija umbral derivado del producto; (4) exige repeticiones; (5) cruzarlo activa reevaluación, no tecnología predeterminada.  
**Criterio:** trigger cuantitativo y falsable.

---

## 27. Ejercicios independientes

Conserva derivaciones, código y mediciones. No hay soluciones inmediatas.

1. Dibuja slots/referencias de una list con dos aliases al mismo dict.
2. Explica por qué `items[500]` no recorre 500 posiciones.
3. Analiza un índice negativo.
4. Deriva tiempo/espacio de un slice de `k` elementos.
5. Demuestra que slice es shallow con objetos mutables.
6. Distingue length y capacity sin intentar leer un threshold interno.
7. Explica una secuencia de appends y un resize ocasional.
8. Compara insert en inicio, medio y final.
9. Separa search y shift en `remove(value)`.
10. Analiza `pop()` y `pop(0)` según elementos movidos.
11. Mide membership en list para inicio, final y ausencia.
12. Define `n` y costo cuando equality recorre strings largos.
13. Diseña una timeline donde list sea la mejor elección.
14. Identifica cuándo esa timeline necesita un índice por ID.
15. Traza key → hash → bucket → equality → value.
16. Explica por qué hashing no protege un secreto.
17. Clasifica cinco objetos como hashable/no hashable y justifica.
18. Construye una `SourceKey` frozen con fields hashable.
19. Rompe deliberadamente equality/hash y escribe un test diagnóstico.
20. Implementa `SimpleHashMap` sin copiar internals de dict.
21. Agrega update y comprueba que size no crece.
22. Provoca tres collisions deterministas.
23. Busca una key ausente en una chain.
24. Repite con `always_zero` y cuenta comparaciones.
25. Calcula load factor para tres tablas conceptuales.
26. Explica por qué load factor no basta ante input adversarial.
27. Diseña resizing conceptual sin fijar threshold de Python.
28. Demuestra dict insertion order y no sorted order.
29. Detecta duplicate IDs antes de insertarlos.
30. Diseña una stable key y rechaza dos alternativas mutables.
31. Usa tuple como composite key y documenta su identidad.
32. Deduplica IDs con set y presenta output con `sorted`.
33. Aplica union/intersection/difference a tags.
34. Usa `frozenset` solo donde su inmutabilidad aporte contrato.
35. Compara list/set para `q=1`, `100` y `10_000`.
36. Compara scan/dict incluyendo build y memoria.
37. Reconstruye list/dict/set desde source vacío de projections.
38. Inyecta duplicate event ID y conserva posición/ID en el diagnóstico.
39. Verifica que dos rebuilds no compartan contenedores exteriores.
40. Escribe un migration trigger de latencia y memoria para retirar o mantener el índice.

---

## 28. Preguntas conceptuales

1. ¿Por qué acceso por índice en un array es O(1)?
2. ¿Qué almacena conceptualmente una `list` de Python?
3. ¿Por qué contigüidad de referencias no implica contigüidad de objetos?
4. ¿Qué diferencia existe entre length y capacity?
5. ¿Cómo puede append ser O(1) amortizado si un resize cuesta O(n)?
6. ¿Por qué insertar al inicio suele ser O(n)?
7. ¿Qué costos distintos combina `remove(value)`?
8. ¿Por qué list membership depende de equality?
9. ¿Cuándo locality puede hacer relevante una constante sin cambiar Big O?
10. ¿Qué problema resuelve un hash map?
11. ¿Por qué el hash no es una ubicación única?
12. ¿Qué diferencia existe entre hash y encryption?
13. ¿Qué implica `a == b` para sus hashes?
14. ¿Por qué el inverso no es obligatorio?
15. ¿Por qué una key mutable amenaza lookup estable?
16. ¿Qué es un bucket dentro del modelo educativo?
17. ¿Qué es una collision?
18. ¿Por qué collision resolution conserva corrección?
19. ¿Cómo funciona separate chaining?
20. ¿Por qué `SimpleHashMap` no describe internals de `dict`?
21. ¿Qué significa expected practical O(1)?
22. ¿Cómo aparece worst O(n) en hashing?
23. ¿Qué expresa load factor y qué no garantiza?
24. ¿Qué costo ocasional introduce resizing?
25. ¿Qué garantiza insertion order de `dict`?
26. ¿Por qué insertion order no significa sorted order?
27. ¿Cómo puede una comprehension ocultar duplicate keys?
28. ¿Por qué `hash()` no sirve como domain ID persistente?
29. ¿Cuándo list puede ser mejor que set o dict?
30. ¿Por qué un índice derivado no debe convertirse en source of truth?
31. ¿Qué invariantes agrega mantener list y dict simultáneamente?
32. ¿Cómo demuestra un rebuild que la proyección es descartable?
33. ¿Qué costo espacial compra expected lookup más barato?
34. ¿Qué debe incluir un benchmark para comparar estrategias completas?
35. ¿Qué evidencia justificaría crear, mantener o retirar un índice EIDOLON?

---

## 29. Mini challenge — In-memory Event Index y collisions

### 29.1 Objetivo

Construye un experimento reproducible con Events sintéticos y dos artefactos separados:

1. proyecciones profesionales con `list`, `dict` y `set`;
2. hash map educativa para demostrar collisions.

No uses la tabla didáctica como índice EIDOLON profesional.

### 29.2 Estructuras requeridas

```text
source_events      → autoridad de entrada del experimento
ordered_events     → list derivada
events_by_id       → dict derivado
seen_source_ids    → set derivado
```

### 29.3 Política

- preservar el orden de source;
- no mutar los dicts del input;
- duplicate `event_id`: `ValueError` con ID y posición;
- duplicate `source_id`: permitido y detectado; guardar una sola membership entry y contar repeats en un resumen separado;
- lookup ausente retorna `None`;
- output de sets se ordena para presentación;
- rebuild parte siempre de proyecciones vacías.

### 29.4 API mínima

**Solución parcial — faltan implementación y decisiones de benchmark:**

```python
def rebuild_state(source_events):
    ...


def find_by_scan(ordered_events, event_id):
    ...


def find_by_index(events_by_id, event_id):
    ...
```

`rebuild_state` devuelve las tres estructuras y un resumen determinista de duplicate source IDs. La función no conserva globals ni muta input.

### 29.5 Comprobaciones

Debes demostrar:

**Fragmento — adapta los nombres a tu implementación:**

```python
assert [event["event_id"] for event in ordered_events] == expected_order
assert events_by_id["evt-002"] == ordered_events[1]
assert ordered_events[1] is events_by_id["evt-002"]
assert ordered_events[0] is not source_events[0]
assert find_by_scan(ordered_events, "evt-002") == find_by_index(
    events_by_id, "evt-002"
)
assert seen_source_ids == {"src-a", "src-b"}
```

Agrega duplicate `event_id` y comprueba `ValueError`. Agrega duplicate `source_id` y comprueba la policy sin confundirlo con corrupción de Event identity.

### 29.6 Rebuild

Ejecuta dos rebuilds desde el mismo source. Comprueba:

- igualdad de orden, keys y seen IDs;
- contenedores exteriores distintos;
- ninguna lectura desde proyecciones anteriores;
- source sin cambios.

### 29.7 Benchmark scan vs dict

Para `n = 10², 10³, 10⁴, 10⁵` y `q = 1, 10, 100, 1_000` cuando sea razonable:

- mismos Events y targets;
- targets al inicio, final y ausentes en grupos declarados;
- múltiples repeticiones y mediana;
- build separado;
- lookup separado;
- total `build + indexed lookups`;
- allocations del índice con `tracemalloc` y límites escritos.

`10⁶` es opcional si el equipo lo permite.

### 29.8 Benchmark list vs set

Compara membership sobre source IDs para una y muchas consultas. Incluye build del set cuando la estrategia lo requiera. No reemplaces la ordered list si su orden/duplicates siguen siendo parte del contrato.

### 29.9 Hash map didáctico separado

Implementa `SimpleHashMap` con:

- insert;
- update;
- get/missing;
- separate chaining;
- collision determinista;
- mala hash `always_zero`;
- contador de comparaciones para distributed vs colliding input.

Incluye en el README del challenge la advertencia: no replica internals de Python `dict` y no se usa como estructura de producción.

### 29.10 Time-space tradeoff y trigger

Entrega una tabla:

```text
n | q | scan median | dict build | dict lookup | indexed total
  | dict retained bytes | dict peak bytes | list/set membership totals
```

Documenta:

- primer break-even consistente observado;
- distribución de targets;
- costo de memoria;
- qué resultado refutó o confirmó la hipótesis;
- trigger cuantitativo para crear/mantener/retirar el índice;
- por qué el source sigue siendo authority.

### 29.11 Failure cases obligatorios

1. duplicate event ID sobrescrito silenciosamente;
2. mutable/unhashable key;
3. `hash()` usado como persistent ID;
4. set order usado como output contractual;
5. hash table sin collision handling;
6. `always_zero` y lookup lineal;
7. benchmark de dict que omite build;
8. source y projection divergentes.

### 29.12 Criterio de aprobación

- list/dict/set responden operaciones distintas;
- duplicate policies son explícitas y probadas;
- source no se modifica;
- rebuild funciona desde estado vacío;
- ambas estrategias de lookup devuelven lo mismo;
- benchmark controla inputs y mide build/memoria;
- results no se universalizan;
- SimpleHashMap conserva colliding entries;
- expected y worst case se distinguen;
- trigger y time-space tradeoff son cuantitativos;
- no se requiere CS-M3+.

Output final después de todas las comprobaciones:

```text
CS-M2 structures challenge: PASS
```

---

## 30. Resumen

- Un array ofrece posiciones indexables; `list` es un dynamic array de referencias.
- El índice permite acceso O(1); no copia el objeto recuperado.
- Un slice materializa `k` referencias y cuesta O(k) tiempo/espacio.
- Append es O(1) amortizado; un resize individual puede costar O(n).
- Inserción/eliminación arbitraria desplaza referencias.
- List membership es linear scan guiado por equality.
- Locality afecta constantes y se profundiza en CS-M10.
- Un hash map usa hash para reducir candidatos y equality para confirmar key.
- Hashing no es encryption, autenticidad ni identidad persistente.
- Objetos iguales deben tener hashes compatibles.
- Keys deben permanecer estables durante su vida en la tabla.
- Collisions son normales; collision resolution conserva entries.
- Separate chaining es nuestro modelo didáctico, no el layout de `dict`.
- Hash lookup es expected practical O(1) y worst O(n).
- Load factor relaciona entries/capacity; resizing redistribuye con costo ocasional.
- `dict` conserva insertion order, no sorted order.
- Duplicate keys requieren policy; overwrite puede ocultar corrupción.
- `set` expresa membership/unicidad y no ofrece orden contractual.
- `frozenset` expresa un conjunto inmutable y puede ser hashable.
- List, dict y set pueden convivir como proyecciones de una fuente.
- Cada índice compra tiempo con build, memoria, maintenance e invariants.
- Un benchmark defendible incluye build, consultas, distribución y memoria.
- El journal/source sigue siendo autoridad; índices in-memory son rebuildable.

---

## 31. Checklist de dominio

- [ ] Puedo explicar array, índice, elemento, length y capacity.
- [ ] Puedo dibujar una list como slots de referencias.
- [ ] Puedo predecir aliasing entre elementos de list.
- [ ] Puedo justificar acceso por índice O(1).
- [ ] Puedo derivar slicing O(k) y explicar shallow copy.
- [ ] Puedo explicar append O(1) amortizado sin negar resize O(n).
- [ ] Puedo separar search y shift en insertion/deletion.
- [ ] Puedo justificar list membership worst O(n).
- [ ] Puedo explicar locality sin adelantar CS-M10.
- [ ] Puedo trazar key → hash → bucket → equality → entry.
- [ ] Puedo explicar por qué hash no es encryption ni ID.
- [ ] Puedo aplicar el contrato equality/hash.
- [ ] Puedo detectar una key mutable o inestable.
- [ ] Puedo explicar bucket, collision y chaining.
- [ ] Puedo implementar y probar `SimpleHashMap` educativa.
- [ ] Puedo demostrar una collision sin perder entries.
- [ ] Puedo provocar degradación O(n) con mala hash.
- [ ] Puedo distinguir expected O(1) de worst O(n).
- [ ] Puedo calcular load factor intuitivo.
- [ ] Puedo explicar resizing sin thresholds inventados.
- [ ] Puedo usar insertion order sin llamarlo sorted order.
- [ ] Puedo rechazar duplicate event IDs antes de overwrite.
- [ ] Puedo diseñar una stable/composite key.
- [ ] Puedo usar set para membership/unicidad.
- [ ] Puedo producir output determinista desde set.
- [ ] Puedo justificar cuándo `frozenset` aporta un contrato.
- [ ] Puedo elegir list/dict/set desde el workload.
- [ ] Puedo mantener una autoridad y proyecciones rebuildable.
- [ ] Puedo medir scan/dict y list/set incluyendo build.
- [ ] Puedo interpretar memoria con `tracemalloc` prudentemente.
- [ ] Puedo documentar un time-space tradeoff y migration trigger.
- [ ] Puedo completar el challenge con PF + CS-M1 + CS-M2.

---

## 32. Preparación para labs y EIDOLON 0.0b

### CS-L02 — List vs dict entity lookup

Es el lab principal de decisión profesional. CS-M2 prepara:

- baseline scan y expected dict lookup;
- build y duplicate detection;
- distribución de targets;
- memoria del índice;
- test de rebuild desde source;
- break-even y trigger.

### CS-L03 — Hash collisions

CS-M2 aporta `SimpleHashMap`, separate chaining, collision determinista, mala hash y contador de comparaciones. El lab debe demostrar correctitud bajo collisions y worst-case degradation sin confundir el modelo con CPython.

### CS-MP1 — In-memory Event Index

Quedan preparados ordered Events, index by ID, seen source IDs, stable keys, rebuild, duplicate policy y medición. Range index y ordenamiento formal esperan CS-M4; top-k espera CS-M5.

### EIDOLON 0.0b

CS-M2 añade proyecciones reconstruibles y una justificación cuantitativa. No añade persistence, services ni otra tecnología.

### Evidencia antes de CS-M3

1. catorce ejercicios guiados reproducidos;
2. al menos veinte independientes, incluidos 1, 4, 7–12, 18–24, 28–30, 35–40;
3. `SimpleHashMap` con collision/update/missing verificados;
4. explicación escrita de equality/hash y expected/worst;
5. rebuild desde source vacío con duplicate test;
6. benchmark scan/dict y list/set con build/memoria;
7. output de set determinista;
8. trigger cuantitativo defendido.

---

## 33. Recursos de ampliación

El módulo es autocontenido para su revisión. Para ampliar, usa los recursos canónicos de [`CS.11`](../../02_curriculum/02_computer_science_foundations.md#cs11-recursos-recomendados): MIT OCW 6.006 para arrays/hashing, CLRS para modelos formales y *The Algorithm Design Manual* para selección aplicada.

La documentación oficial de Python sobre `list`, `dict`, `set`, `frozenset`, `hash`, data model y `tracemalloc` sirve para comprobar contratos de API. No sustituye modelar workload ni medir el proyecto.

---

## 34. Límite explícito del módulo

CS-M2 termina en arrays dinámicos de referencias, hashing aplicado, collision resolution didáctica, `dict`/`set`, stable keys, derived indexes, rebuild y benchmarks de tiempo/memoria.

Quedan fuera:

- stacks, queues, deques y linked lists: CS-M3;
- binary search, recursion y sorting algorithms: CS-M4;
- trees, heaps y top-k: CS-M5;
- graphs, lógica/conjuntos formal y state machines: CS-M6;
- filesystem/memory internals: CS-M7;
- concurrency: CS-M8;
- networking: CS-M9;
- cache lines y arquitectura detallada: CS-M10.

No se implementan persistence, databases, backend, AI, services ni custom production hash tables. No se enseñan universal hashing, proofs probabilísticos, cryptographic hashing ni internals detallados de CPython.

El siguiente paso permitido es revisar CS-M2 como `review candidate`. **No se crea ni se desarrolla CS-M3 en esta entrega.**
