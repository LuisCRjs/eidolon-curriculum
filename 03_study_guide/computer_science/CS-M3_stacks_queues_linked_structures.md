# CS-M3 — Stacks, queues, deques y linked lists

**Track:** Computer Science Foundations  
**Competencias:** D2.1; soporte D2.3, D3.2  
**Fase:** P0  
**Nivel objetivo:** Aplicado  
**Prerequisites:** PF-M1, PF-M3, PF-M5, PF-M8, CS-M1, CS-M2  
**Build:** EIDOLON 0.0b  
**Curriculum source:** [CS-M3](../../02_curriculum/02_computer_science_foundations.md#cs-m3--stacks-queues-deques-y-linked-lists)  
**Status:** approved

Deshacer la última acción, procesar trabajos en orden de llegada e insertar un elemento después de una posición ya conocida son problemas distintos. Una sola colección puede ejecutarlos, pero no necesariamente comunica el orden correcto ni ofrece el costo apropiado.

CS-M3 sigue esta cadena:

```text
orden del problema
↓
abstract data type
↓
representación concreta
↓
operaciones dominantes
↓
invariantes
↓
failure cases
↓
medición y tradeoff
```

El objetivo no es implementar linked lists para reemplazar `list` o `deque`. Es aprender a separar contrato de representación, razonar sobre referencias y justificar una estructura desde el workload.

No se desarrollan recursion, searching/sorting formal, trees, heaps, graphs, concurrencia, OS internals ni networking. CS-M4–CS-M10 cubren esos temas.

## Resultados de aprendizaje

Al terminar podrás:

- distinguir un abstract data type (ADT) de su implementación;
- explicar LIFO/FIFO y predecir el orden de salida;
- implementar un stack idiomático con `list` y justificar sus costos;
- implementar una queue síncrona con `deque` y evitar `list.pop(0)` repetido;
- usar ambos extremos de `deque` sin convertirlo en inventario de APIs;
- modelar una queue bounded sin reabrir async/backpressure de PF-M8;
- dibujar nodes, `head`, `tail` y referencias `next`;
- recorrer y buscar en una singly linked list con costo O(n);
- justificar inserción/eliminación O(1) solo cuando ya posee la referencia apropiada;
- derivar search O(n) + relink O(1) = O(n);
- implementar una singly linked list educativa con invariantes comprobables;
- detectar stale tail, lost chain, external mutation y cycle sin colgar la ejecución;
- explicar ownership lógico de nodes y encapsulación mínima;
- comparar dynamic array, deque y linked list por tiempo, memoria, locality y simplicidad;
- medir stacks, queues y operaciones enlazadas sin universalizar resultados locales;
- aplicar stack/queue como proyecciones derivadas y auditables de EIDOLON.

## Cómo estudiar este módulo

1. Nombra primero el orden requerido: LIFO, FIFO, ambos extremos o posición enlazada.
2. Escribe el ADT antes de elegir `list`, `deque` o nodes.
3. Dibuja cada referencia antes de modificar `next`.
4. Separa encontrar una posición de modificarla.
5. Comprueba invariantes en vacío, un elemento y varios elementos.
6. Protege todo ejemplo con cycle mediante visited IDs o límite.
7. Formula la hipótesis de benchmark antes de medir.

### Convenciones

- **Ejemplo ejecutable:** bloque autónomo con output estable o asserts.
- **Benchmark ejecutable:** produce tiempos ambientales; comprueba propiedades estables.
- **Modelo educativo:** estructura manual para observar referencias/invariantes, no default de producción.
- **Failure case ejecutable:** reproduce un fallo sin loop infinito.
- **Código incorrecto:** antipatrón deliberado cuya causa/corrección se explica.
- **Fragmento:** requiere contexto omitido y no se ofrece como programa autónomo.

Baseline recomendado: Python 3.14 y standard library.

---

## 1. La estructura refleja el orden del problema

Compara tres preguntas:

```text
deshacer la última acción
→ debe salir lo más reciente

procesar trabajos por llegada
→ debe salir lo más antiguo pendiente

insertar tras un elemento ya localizado
→ debe cambiar una relación entre posiciones
```

De ellas surgen tres modelos:

- **LIFO** (*Last In, First Out*): stack;
- **FIFO** (*First In, First Out*): queue;
- **linked structure**: nodes conectados mediante referencias.

El orden no es un detalle de implementación. Procesar una queue como stack cambia comportamiento aunque todos los elementos sigan presentes.

### Identifica

Clasifica: undo, pending imports, historial que se recorre por posición y relink después de un node conocido. Explica la propiedad, no solo el nombre.

### Predice

Si entran A, B, C, ¿qué sale primero bajo LIFO y bajo FIFO?

---

## 2. Abstract data type frente a implementación

Un **abstract data type (ADT)** define operaciones y comportamiento observable. No obliga una representación concreta.

```text
Stack ADT
- push(value)
- pop()
- peek()
- is_empty()

Queue ADT
- enqueue(value)
- dequeue()
- front()
- is_empty()
```

El Stack ADT podría implementarse con `list`, `deque` o nodes. La elección de representación decide costos y failure surface; el ADT conserva la semántica LIFO.

```text
abstracción: qué promete
representación: cómo conserva estado
```

Una API puede decidir si pop/dequeue vacío retorna `None` o falla. Lo esencial es documentar y probar una política, no mezclar ambas accidentalmente.

### Explica

¿Por qué “queue = deque” confunde contrato e implementación? Da otra representación posible y su tradeoff.

### Elige

Para una queue profesional síncrona en Python, ¿por qué `deque` suele ser mejor default que una linked list manual aun cuando ambas puedan ofrecer extremos O(1)?

---

## 3. Stack: LIFO y operaciones

### 3.1 Modelo mental

```text
push A
push B
push C

top
 ↓
[C]
[B]
[A]

pop → C
pop → B
```

Operaciones:

- `push`: añade al top;
- `pop`: retira/devuelve el top;
- `peek`: observa top sin retirar;
- `is_empty`: comprueba ausencia;
- `size`: cuenta elementos si el contrato lo necesita.

### 3.2 Stack idiomático con `list`

**Ejemplo ejecutable:**

```python
stack = []
stack.append("action-A")
stack.append("action-B")
stack.append("action-C")

print(stack[-1])
print(stack.pop())
print(stack.pop())
print(len(stack))
```

Output:

```text
action-C
action-C
action-B
1
```

Según CS-M2:

- `append`: O(1) amortizado;
- `pop()` al final: O(1) amortizado/práctico;
- `stack[-1]`: O(1);
- `not stack`: O(1).

La `list` ya es una implementación madura y clara para un stack. Una linked stack no gana automáticamente.

### 3.3 Empty contract

**Ejemplo ejecutable:**

```python
def pop_or_none(stack):
    if not stack:
        return None
    return stack.pop()


print(pop_or_none([]))
print(pop_or_none(["action-A"]))
```

Output:

```text
None
action-A
```

Aquí `None` significa empty y los values válidos son strings. Si `None` pudiera ser un item válido, el contrato necesitaría otra señal.

### Predice

Después de push A, B, pop, push C, ¿qué devuelve peek y qué contiene el stack?

### Costo

¿Por qué implementar push con `insert(0, value)` funciona semánticamente pero compra O(n) shifts innecesarios?

---

## 4. Call stack y undo: intuiciones controladas

### 4.1 Call stack

```text
function A calls B
            B calls C

active top: C
return C → continúa B
return B → continúa A
```

El retorno en orden inverso ilustra LIFO. Recursion y límites del call stack pertenecen a CS-M4; aquí no se implementan.

### 4.2 Undo sobre una vista derivada

EIDOLON no borra el source journal. Un undo didáctico puede revertir cambios en una vista temporal:

**Ejemplo ejecutable:**

```python
source_tags = ("home",)
visible_tags = list(source_tags)
undo_stack = []

undo_stack.append(visible_tags.copy())
visible_tags.append("arrival")

visible_tags = undo_stack.pop()

print(source_tags)
print(visible_tags)
```

Output:

```text
('home',)
['home']
```

El stack conserva un estado inverso/snapshot pequeño de la vista. No reescribe el source. CS-L04 exigirá límites y empty semantics.

### Invariante

¿Qué debe seguir siendo cierto sobre `source_tags` después de push, cambio y undo?

---

## 5. Queue: FIFO y el problema de `list.pop(0)`

### 5.1 Modelo mental

```text
enqueue A
enqueue B
enqueue C

front → [A][B][C] ← back

dequeue → A
dequeue → B
```

Operaciones:

- `enqueue`: añade en back;
- `dequeue`: retira/devuelve front;
- `front`: observa primero sin retirar;
- `is_empty` y `size`: estado observable.

### 5.2 Queue con list: correcta pero costosa

**Ejemplo ejecutable:**

```python
queue = []
queue.append("job-A")
queue.append("job-B")
queue.append("job-C")

print(queue.pop(0))
print(queue.pop(0))
```

Output:

```text
job-A
job-B
```

La semántica es FIFO, pero cada `pop(0)` desplaza las referencias restantes: O(n). Repetirlo para vaciar `n` items acumula trabajo cuadrático aproximado.

### Detecta el bug de costo

Un review dice: “`pop(0)` es una llamada, entonces O(1)”. Distingue líneas de operaciones y describe el shift.

---

## 6. `collections.deque`: ambos extremos

`deque` significa **double-ended queue**. Su contrato práctico ofrece append/pop eficientes en ambos extremos sin depender de un array que desplaza todo al retirar el frente.

```text
← front [ ... ] back →
```

**Ejemplo ejecutable:**

```python
from collections import deque


items = deque(["B"])
items.appendleft("A")
items.append("C")

print(items.popleft())
print(items.pop())
print(list(items))
```

Output:

```text
A
C
['B']
```

Operaciones en extremos (`append`, `appendleft`, `pop`, `popleft`) son O(1) práctico/amortizado bajo el contrato de `deque`. Acceso arbitrario en medio no es su fortaleza; no la uses como reemplazo universal de `list`.

### 6.1 Queue idiomática

**Ejemplo ejecutable:**

```python
from collections import deque


pending = deque()
pending.append("job-A")
pending.append("job-B")

first = pending.popleft() if pending else None
second = pending.popleft() if pending else None
third = pending.popleft() if pending else None

print(first, second, third)
```

Output:

```text
job-A job-B None
```

### Elige

¿Usarías `list` o `deque` para acceso repetido por índice? ¿Y para 100,000 `popleft`? Justifica por operación dominante.

---

## 7. Queue de trabajos síncrona y capacidad

Una queue de `pending_imports` representa orden, no concurrencia:

**Ejemplo ejecutable:**

```python
from collections import deque


class BoundedWorkQueue:
    def __init__(self, capacity):
        if capacity <= 0:
            raise ValueError("capacity must be positive")
        self._capacity = capacity
        self._items = deque()

    def enqueue(self, job):
        if len(self._items) >= self._capacity:
            raise OverflowError("queue capacity reached")
        self._items.append(job)

    def dequeue(self):
        if not self._items:
            return None
        return self._items.popleft()

    def __len__(self):
        return len(self._items)


queue = BoundedWorkQueue(capacity=2)
queue.enqueue({"job_id": "job-001", "source_id": "src-a"})
queue.enqueue({"job_id": "job-002", "source_id": "src-b"})

print(queue.dequeue()["job_id"])
print(queue.dequeue()["job_id"])
print(queue.dequeue())
```

Output:

```text
job-001
job-002
None
```

La capacidad evita crecimiento ilimitado y hace visible overflow. No espera, no ejecuta concurrentemente y no resuelve retries/idempotency. PF-M8 enseñó espera cooperativa/backpressure; CS-L05 añadirá job IDs, retry count y simulación acotada.

No uses `deque(maxlen=k)` sin entender su contrato: al llenarse, un append puede descartar silenciosamente items del extremo opuesto, algo inaceptable para una work queue si perder jobs viola invariantes.

### Edge case

Provoca el tercer enqueue antes de dequeue y verifica `OverflowError` sin fijar el mensaje completo.

### Invariante

Escribe: `0 ≤ len(queue) ≤ capacity` y FIFO para todos los jobs aceptados.

---

## 8. Stack, queue, deque y priority preview

| Semántica requerida | ADT | Implementación Python probable |
|---|---|---|
| último agregado sale primero | stack | `list` o `deque` |
| primero agregado sale primero | queue | `deque` |
| operaciones en ambos extremos | deque | `collections.deque` |
| más urgente sale primero | priority queue | preview; heaps en CS-M5 |

“Más urgente” no es FIFO. Implementar prioridad mediante sorting en cada dequeue cambia costo y pertenece a otro módulo.

### Identifica

¿Una pila de undo es queue porque contiene tareas? ¿Una queue de imports debe procesar el job más nuevo primero? Responde desde orden contractual.

---

## 9. Linked structures y `Node`

Una **linked structure** conserva relaciones mediante referencias entre objects:

```text
node = value + reference to next
```

No necesita posiciones contiguas conceptualmente. Cada referencia sigue las reglas de PF-M1: asignar crea alias; mutar `next` cambia el object compartido; `None` marca ausencia de siguiente node.

### 9.1 Node mínimo

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(eq=False)
class Node:
    value: str
    next: "Node | None" = None


third = Node("C")
second = Node("B", third)
head = Node("A", second)

print(head.value)
print(head.next.value)
print(head.next.next.value)
print(head.next.next.next)
```

Output:

```text
A
B
C
None
```

`eq=False` conserva equality por identity, apropiada para razonar sobre nodes individuales y visited IDs. La string annotation permite referirse a `Node` mientras la class se está definiendo.

### 9.2 Singly linked list

```text
head
 ↓
[A | •] → [B | •] → [C | None]
```

Cada node conoce solo `next`. `head` apunta al primero o es `None`. `tail` es opcional y añade otra referencia/invariante.

### Dibuja referencias

Si `alias = second` y después `alias.next = None`, ¿qué cadena queda alcanzable desde `head`? ¿Qué node sigue existiendo bajo el nombre `third`?

---

## 10. Traversal y búsqueda enlazada

No existe cálculo `base + i × size`. Para llegar al tercero debes seguir referencias.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(eq=False)
class Node:
    value: str
    next: "Node | None" = None


head = Node("A", Node("B", Node("C")))

values = []
current = head
while current is not None:
    values.append(current.value)
    current = current.next

print(values)
```

Output:

```text
['A', 'B', 'C']
```

Traversal de `n` nodes cuesta Θ(n). Buscar por value tiene best Θ(1) en `head` y worst Θ(n) al final/ausente. Una linked list no ofrece indexed access O(1).

### Costo

Para obtener el node en posición `i` desde `head`, ¿qué referencias deben seguirse? ¿Qué cambia si ya posees una referencia directa a ese node?

---

## 11. Prepend, insert after y remove

### 11.1 Prepend O(1)

Orden correcto:

```text
new.next = head
head = new
```

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(eq=False)
class Node:
    value: str
    next: "Node | None" = None


head = Node("B", Node("C"))
new = Node("A")

new.next = head
head = new

print(head.value, head.next.value, head.next.next.value)
```

Output:

```text
A B C
```

Con referencia a `head`, son asignaciones constantes. No se desplazan otros nodes.

### 11.2 Insert after un node conocido

```text
antes:  current → old_next

new.next = current.next
current.next = new

después: current → new → old_next
```

El relink es O(1) **si ya posees `current`**. Si la operación recibe solo `head` y un value, debe buscar primero: O(n) + O(1) = O(n).

### 11.3 Remove after conocido

Con precondición `current.next is not None`:

```text
removed = current.next
current.next = removed.next
removed.next = None
```

El node retirado deja de ser alcanzable desde esa chain; puede seguir vivo si otro nombre lo referencia. Python gestiona reclamación de objetos; memory management profundo pertenece a CS-M7.

### 11.4 Remove by value

Buscar previous/current cuesta O(n) worst case. Reenlazar cuesta O(1). El total sigue O(n).

### Costo

Corrige: “insertar `X` después del primer node cuyo value es target cuesta O(1)”. Declara input, search y mutation.

### Dibuja referencias

Dibuja A→B→C, inserta X después de B y retira C. Marca qué referencias cambian y cuáles permanecen.

---

## 12. Tail, size e invariantes

Conservar `tail` permite append O(1):

```text
tail.next = new
tail = new
```

Pero añade estado que debe permanecer coherente.

### 12.1 Invariantes centrales

- empty: `head is None`, `tail is None`, `size == 0`;
- non-empty: `head` y `tail` son nodes alcanzables;
- `tail.next is None`;
- seguir `next` desde `head` llega exactamente a `size` nodes;
- no aparece cycle bajo el contrato de list lineal.

### 12.2 One-element edge case

```text
head ─┐
      ▼
     [A|None]
      ▲
tail ─┘
```

Al retirar A:

```text
head = None
tail = None
size = 0
```

Dejar `tail` apuntando a A crea stale state.

### Invariante

¿Qué debe pasar con `tail` al retirar el último node? ¿Y al hacer prepend sobre empty?

---

## 13. Cycles, lost chains y aliasing

### 13.1 Cycle accidental

```text
A → B → C
    ↑   ↓
    └───┘
```

Un traversal ingenuo no termina. La validación debe protegerse.

**Failure case ejecutable — cycle detectado:**

```python
from dataclasses import dataclass


@dataclass(eq=False)
class Node:
    value: str
    next: "Node | None" = None


first = Node("A")
second = Node("B")
third = Node("C")
first.next = second
second.next = third
third.next = second

seen_ids = set()
current = first

try:
    while current is not None:
        marker = id(current)
        if marker in seen_ids:
            raise ValueError("cycle detected")
        seen_ids.add(marker)
        current = current.next
except ValueError:
    print("cycle_detected=True")
```

Output estable:

```text
cycle_detected=True
```

`id` solo identifica objects durante esta ejecución; no se persiste. Floyd cycle detection queda como ampliación, no requisito.

### 13.2 Lost chain por orden incorrecto

**Código incorrecto ejecutable:**

```python
from dataclasses import dataclass


@dataclass(eq=False)
class Node:
    value: str
    next: "Node | None" = None


second = Node("B", Node("C"))
head = Node("A", second)
new = Node("X")

head.next = new
new.next = head.next

print(head.next is new)
print(new.next is new)
print(second.value)
```

Output:

```text
True
True
B
```

La primera asignación perdió desde `head` la referencia a B. La segunda lee el `head.next` ya modificado y crea un self-cycle X→X. `second` aún existe porque ese nombre lo conserva, pero ya no pertenece a la chain desde `head`.

Corrección: primero `new.next = head.next`; después `head.next = new`.

### 13.3 Aliasing externo

Dos nombres pueden apuntar al mismo node. Si código externo recibe un node y cambia `node.next`, modifica la estructura. Ownership lógico debe indicar quién puede mutar enlaces.

### Detecta el bug

¿Qué línea del lost-chain example debe ejecutarse primero y por qué? Responde leyendo los valores de las referencias antes/después.

---

## 14. Ownership y encapsulación mínima

Ownership aquí no significa `malloc/free`. Significa: ¿qué componente es responsable de preservar `head`, `tail`, `size` y `next`?

Una implementación educativa puede mantener nodes detrás de métodos y devolver values, no referencias mutables. Python no impone privacidad estricta; el underscore comunica frontera y los tests protegen invariantes.

```text
caller
  │ append/find/remove values
  ▼
SinglyLinkedList
  │ único owner lógico de relinks
  ▼
Node chain
```

Exponer nodes puede ser necesario para enseñar insert-after-known, pero una API de producción debe justificar esa capacidad porque el caller podría crear cycles o stale tail.

### Elige

¿Devolverías `Node` o su `value` desde `find`? Compara flexibilidad con capacidad de romper invariantes.

---

## 15. `SinglyLinkedList` educativa

La implementación concreta usa strings para evitar generics avanzados. Incluye append, prepend, find, remove, iteration e invariant validation.

**Modelo educativo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(eq=False)
class Node:
    value: str
    next: "Node | None" = None


class SinglyLinkedList:
    def __init__(self):
        self._head = None
        self._tail = None
        self._size = 0

    def prepend(self, value):
        new = Node(value, self._head)
        self._head = new
        if self._tail is None:
            self._tail = new
        self._size += 1

    def append(self, value):
        new = Node(value)
        if self._tail is None:
            self._head = self._tail = new
        else:
            self._tail.next = new
            self._tail = new
        self._size += 1

    def find(self, value):
        current = self._head
        while current is not None:
            if current.value == value:
                return current.value
            current = current.next
        return None

    def remove_first(self, value):
        previous = None
        current = self._head

        while current is not None:
            if current.value == value:
                if previous is None:
                    self._head = current.next
                else:
                    previous.next = current.next

                if current is self._tail:
                    self._tail = previous

                current.next = None
                self._size -= 1
                return True

            previous = current
            current = current.next

        return False

    def __iter__(self):
        current = self._head
        while current is not None:
            yield current.value
            current = current.next

    def __len__(self):
        return self._size

    def validate(self):
        if self._size == 0:
            assert self._head is None
            assert self._tail is None
            return True

        assert self._head is not None
        assert self._tail is not None

        seen_ids = set()
        current = self._head
        count = 0
        last = None

        while current is not None:
            marker = id(current)
            if marker in seen_ids:
                raise ValueError("cycle detected")
            seen_ids.add(marker)
            count += 1
            last = current
            current = current.next

        assert count == self._size
        assert last is self._tail
        assert self._tail.next is None
        return True


values = SinglyLinkedList()
values.append("B")
values.prepend("A")
values.append("C")

assert values.validate()
assert values.find("B") == "B"
assert values.remove_first("A")
assert values.remove_first("C")
assert values.validate()

print(list(values))
print(len(values))
```

Output:

```text
['B']
1
```

### 15.1 Costos

| Operación | Costo | Precondición/razón |
|---|---|---|
| prepend | O(1) | `head` conocido |
| append | O(1) | `tail` mantenido |
| find | O(n) worst | traversal |
| remove by value | O(n) worst | search + O(1) relink |
| iteration | Θ(n) | visita nodes alcanzables |
| validate | O(n) time/space | visited set y conteo |

`validate` es instrumentation educativa y testing; no tiene que ejecutarse tras cada operación de producción.

### 15.2 Edge cases cubiertos

- append/prepend sobre empty establecen head/tail;
- remove head cambia head;
- remove tail cambia tail;
- remove único deja ambos en `None`;
- missing retorna `False`;
- validate detecta cycle/size/tail incoherente.

### Modifica

Agrega `peek_first` que retorne value o `None` sin exponer Node. Escribe tests para empty y one-element.

---

## 16. `__iter__` como puente

PF-M3 enseñó iterable/iterator. `__iter__` permite que una instancia produzca values uno por uno:

```text
for value in linked_list
↓
head, next, next, ... hasta None
```

El generator conserva una referencia `current`; no materializa otra list salvo que el caller llame `list(linked_list)`. Una estructura corrupta con cycle haría que el iterator ingenuo no termine; valida antes de iterar datos no confiables en este modelo educativo.

No se reabre el protocolo completo ni generators avanzados.

### Predice

¿Qué memoria adicional crea `list(linked_list)` para `n` values? ¿Qué cambia si consumes el iterator uno por uno?

---

## 17. Doubly linked list y sentinel: conceptos

### 17.1 Doubly linked list

```text
None ← [A] ⇄ [B] ⇄ [C] → None
```

Cada node conserva `next` y `previous`. Con referencia directa al node, puede reenlazarse con vecinos sin buscar previous. El costo es más memoria y más invariantes:

- `node.next.previous is node` cuando existe next;
- `node.previous.next is node` cuando existe previous;
- head.previous y tail.next son `None`;
- cada insert/delete actualiza enlaces en ambos sentidos.

No se implementa completa: el boilerplate no añade una decisión nueva en CS-M3.

### 17.2 Sentinel node

```text
sentinel → first real node → ... → None
```

Un sentinel artificial puede reducir branches al insertar/eliminar en head. Añade un node que no pertenece a los datos y una convención que todos deben respetar. Es una opción, no requisito.

### Tradeoff

¿Qué edge case simplifica un sentinel? ¿Qué confusión introduce al contar/iterar values reales?

---

## 18. Linked list frente a dynamic array

| Dimensión | Python `list` | Singly linked list manual |
|---|---|---|
| indexed access | O(1) | O(n) |
| append | O(1) amortizado | O(1) con tail |
| prepend | O(n) shifts | O(1) |
| insert tras posición conocida | O(n) shifts | O(1) relink |
| buscar posición/value | O(n) | O(n) |
| locality | referencias contiguas | nodes separados |
| overhead | container de referencias | object + `next` por node |
| implementación | standard library madura | invariantes manuales |

La ventaja O(1) enlazada exige poseer la referencia apropiada. Si primero buscas, el total es O(n).

En Python, `list` y `deque` suelen ser mejores defaults: están optimizadas, probadas y reducen failure surface. Una linked list manual es valiosa para aprender referencias, o para un workload muy específico demostrado; “nunca” y “siempre” son conclusiones demasiado fuertes.

### Elige

Workload: 90% indexed access, 10% insertion tras buscar por value. ¿Qué estructura candidatiza y por qué “linked insert O(1)” no resuelve la búsqueda?

### Locality

¿Qué objetos adicionales crea una linked list de `n` strings frente a una list de las mismas referencias? No inventes bytes; formula una medición.

---

## 19. Stack y queue enlazados: modelos educativos

### 19.1 Linked stack

Top funciona como head. Push es prepend; pop retira head.

**Modelo educativo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(eq=False)
class Node:
    value: str
    next: "Node | None" = None


class LinkedStack:
    def __init__(self):
        self._top = None
        self._size = 0

    def push(self, value):
        self._top = Node(value, self._top)
        self._size += 1

    def pop(self):
        if self._top is None:
            return None
        removed = self._top
        self._top = removed.next
        removed.next = None
        self._size -= 1
        return removed.value


stack = LinkedStack()
stack.push("A")
stack.push("B")

print(stack.pop())
print(stack.pop())
print(stack.pop())
```

Output:

```text
B
A
None
```

Push/pop son O(1), pero cada push asigna un Node. `list.append/pop` ofrece el mismo ADT con menos código y suele ser la opción normal en Python.

### 19.2 Linked queue

Head + tail permiten enqueue al final y dequeue al frente en O(1).

**Modelo educativo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(eq=False)
class Node:
    value: str
    next: "Node | None" = None


class LinkedQueue:
    def __init__(self):
        self._head = None
        self._tail = None

    def enqueue(self, value):
        new = Node(value)
        if self._tail is None:
            self._head = self._tail = new
        else:
            self._tail.next = new
            self._tail = new

    def dequeue(self):
        if self._head is None:
            return None
        removed = self._head
        self._head = removed.next
        if self._head is None:
            self._tail = None
        removed.next = None
        return removed.value

    def is_consistent(self):
        return (self._head is None) == (self._tail is None)


queue = LinkedQueue()
queue.enqueue("A")
queue.enqueue("B")

print(queue.dequeue())
print(queue.dequeue())
print(queue.dequeue())
print(queue.is_consistent())
```

Output:

```text
A
B
None
True
```

Actualizar tail al vaciar es obligatorio. `deque` ofrece el mismo patrón con una implementación estándar; el modelo enlazado existe para entender referencias e invariantes.

### Elige

¿Qué ganaría una linked queue manual frente a `deque` en tu workload concreto? Si no puedes medir o nombrar una capacidad necesaria, conserva `deque`.

---

## 20. Caso EIDOLON: tres órdenes, ninguna autoridad

```text
source journal (authority)
       │ replay
       ├──▶ undo_stack de una vista temporal
       ├──▶ pending_imports queue
       └──▶ educational_linked_chain para practicar invariantes
```

### 20.1 Undo stack

Última transformación de la vista se revierte primero. El source permanece auditable.

### 20.2 Pending imports queue

El primer job aceptado se procesa primero. Esto no implica concurrent workers ni async.

### 20.3 Replay worklist

`deque` puede conservar records pendientes para un loop síncrono. Que una worklist se use más adelante en BFS no convierte este ejemplo en graph traversal; CS-M6 lo hará.

**Ejemplo ejecutable:**

```python
from collections import deque


source_records = ("record-001", "record-002", "record-003")
worklist = deque(source_records)
processed = []

while worklist:
    processed.append(worklist.popleft())

print(processed)
print(source_records)
```

Output:

```text
['record-001', 'record-002', 'record-003']
('record-001', 'record-002', 'record-003')
```

La tuple source no cambia; worklist y output son derivados.

### Source of truth

Si el proceso termina con jobs aún en la queue in-memory, ¿puedes afirmar que están durablemente registrados? Separa orden de procesamiento de persistencia.

---

## 21. Empty y one-element edge cases

Todo contrato debe decidir:

- pop de stack empty;
- dequeue de queue empty;
- linked list empty;
- remove head;
- remove único node;
- actualización de tail al vaciar.

Una matriz pequeña ayuda:

| Estado | Operación | Estado posterior |
|---|---|---|
| empty list | append A | head=tail=A, size=1 |
| one node A | remove A | head=tail=None, size=0 |
| A→B | remove A | head=B, tail=B, size=1 |
| empty queue | dequeue | `None` bajo este contrato |

### Edge case

Ejecuta `SinglyLinkedList`: append A, remove A, validate, append B, validate. Esto detecta un tail stale que un test con varios nodes puede ocultar.

---

## 22. Failure cases de referencias y elección

### 22.1 Queue con `list.pop(0)`

Es FIFO correcto pero shift O(n) por dequeue. Corrección: `deque.popleft()` para queues largas/repetidas.

### 22.2 Insert O(1) sin node conocido

Si recibe head + target value, primero recorre O(n). Corrección: reportar search + relink y aclarar la precondición de referencia directa.

### 22.3 Stale tail

**Código incorrecto ejecutable:**

```python
from dataclasses import dataclass


@dataclass(eq=False)
class Node:
    value: str
    next: "Node | None" = None


class BrokenQueue:
    def __init__(self):
        node = Node("only")
        self.head = node
        self.tail = node

    def dequeue(self):
        removed = self.head
        self.head = removed.next
        return removed.value


queue = BrokenQueue()
print(queue.dequeue())
print(queue.head is None)
print(queue.tail is None)
```

Output:

```text
only
True
False
```

La queue parece empty por head, pero tail conserva el node. Corrección: si el nuevo head es `None`, asignar también tail `None`.

### 22.4 Cycle accidental

El ejemplo de la sección 13 demuestra cycle con visited IDs. Nunca uses `while current` sin protección sobre una chain deliberadamente corrupta durante testing.

### 22.5 Lost chain

Sobrescribir `current.next` antes de conservar `old_next` puede perder el resto o crear self-cycle. Corrección: leer/enlazar `new.next` antes de cambiar `current.next`.

### 22.6 External node mutation

**Failure case ejecutable:**

```python
from dataclasses import dataclass


@dataclass(eq=False)
class Node:
    value: str
    next: "Node | None" = None


head = Node("A", Node("B"))
exposed = head.next
exposed.next = exposed

print(exposed.next is exposed)
```

Output:

```text
True
```

Un caller con referencia mutable creó un cycle. Corrección de diseño: devolver values/iterators o documentar ownership y validar mutations controladas.

### 22.7 Linked list por slogan

“Insert O(1)” no justifica una linked list si 90% del workload es indexed lookup, cada insert busca posición y `deque/list` ya cubren el contrato. Corrección: modelo de costo + benchmark + maintenance risk.

### Diagnostica

Para cinco fallos, registra diagrama antes/después, síntoma, referencia incorrecta, invariante rota y corrección mínima.

---

## 23. Benchmark de stack y queue

El benchmark siguiente mide workloads completos y equivalentes. Los tiempos son locales; solo el formato se fija.

**Benchmark ejecutable:**

```python
from collections import deque
from statistics import median
from timeit import repeat


def run_list_stack(n):
    stack = []
    for value in range(n):
        stack.append(value)
    while stack:
        stack.pop()


def run_deque_stack(n):
    stack = deque()
    for value in range(n):
        stack.append(value)
    while stack:
        stack.pop()


def run_list_queue(n):
    queue = []
    for value in range(n):
        queue.append(value)
    while queue:
        queue.pop(0)


def run_deque_queue(n):
    queue = deque()
    for value in range(n):
        queue.append(value)
    while queue:
        queue.popleft()


def measured(action):
    return median(repeat(action, repeat=5, number=1))


n = 2_000
rows = {
    "list_stack": measured(lambda: run_list_stack(n)),
    "deque_stack": measured(lambda: run_deque_stack(n)),
    "list_queue": measured(lambda: run_list_queue(n)),
    "deque_queue": measured(lambda: run_deque_queue(n)),
}

assert all(seconds >= 0.0 for seconds in rows.values())
print("implementations=4")
print("all_measurements_non_negative=True")
```

Output estable:

```text
implementations=4
all_measurements_non_negative=True
```

Cada callable incluye creación, fill y drain. Responde end-to-end para ese workload. Si preguntas solo por dequeue sobre una estructura ya preparada, diseña batches equivalentes sin reutilizar una queue agotada.

Ejecuta tamaños crecientes. Espera que `list.pop(0)` muestre peor tendencia; no declares que un tiempo local prueba una complejidad ni que deque siempre gana en cualquier stack pequeño.

### Benchmark

¿Qué resultado refutaría tu hipótesis práctica? ¿Qué datos debes conservar: `n`, versión, samples, mediana y carga?

---

## 24. Benchmark de operaciones enlazadas

Separamos prepend batch, lookup y relink tras node conocido.

**Benchmark ejecutable:**

```python
from dataclasses import dataclass
from statistics import median
from timeit import repeat


@dataclass(eq=False)
class Node:
    value: int
    next: "Node | None" = None


def build_by_prepend(n):
    head = None
    for value in range(n):
        head = Node(value, head)
    return head


def build_list_by_prepend(n):
    values = []
    for value in range(n):
        values.insert(0, value)
    return values


def find_value(head, target):
    current = head
    while current is not None:
        if current.value == target:
            return current
        current = current.next
    return None


def insert_after_known(current, value):
    new = Node(value, current.next)
    current.next = new
    return new


def measured(action, number=1):
    samples = repeat(action, repeat=5, number=number)
    return median(samples) / number


n = 2_000
head = build_by_prepend(n)
array_values = build_list_by_prepend(n)
known = head

linked_prepend_seconds = measured(lambda: build_by_prepend(n))
list_prepend_seconds = measured(lambda: build_list_by_prepend(n))
linked_lookup_seconds = measured(lambda: find_value(head, 0), number=100)
list_lookup_seconds = measured(lambda: 0 in array_values, number=100)
linked_relink_seconds = measured(
    lambda: insert_after_known(known, -1), number=100
)
list_insert_seconds = measured(
    lambda: array_values.insert(1, -1), number=100
)

assert find_value(head, 0).value == 0
assert array_values[-1] == 0
assert min(
    linked_prepend_seconds,
    list_prepend_seconds,
    linked_lookup_seconds,
    list_lookup_seconds,
    linked_relink_seconds,
    list_insert_seconds,
) >= 0.0
print("measurements=6")
print("measurements_non_negative=True")
```

Output estable:

```text
measurements=6
measurements_non_negative=True
```

Los benchmarks de insert mutan repetidamente estructuras que crecen: linked relink toca un número constante de referencias; `list.insert(1, ...)` desplaza la cola creciente. Prepend/lookup usan el mismo orden lógico `n-1 ... 0`. Las implementaciones también difieren en cuánto trabajo ocurre en Python frente a código nativo, por lo que la medición complementa el modelo y no lo reemplaza.

`find_value(head, 0)` recorre hasta el final porque prepend produce `n-1 ... 0`; por eso modela worst-case lookup.

### Interpreta

¿Por qué un relink medido rápido no justifica la estructura si el caller primero llama `find_value`?

---

## 25. Memoria y locality: comparación prudente

Una linked list crea un object Node y referencia `next` por value. `deque` conserva su propia estructura por bloques; `list` un array de referencias. `sys.getsizeof` superficial no suma el object graph.

**Benchmark ejecutable:**

```python
from collections import deque
from dataclasses import dataclass
import tracemalloc


@dataclass(eq=False)
class Node:
    value: int
    next: "Node | None" = None


def build_linked(values):
    head = None
    for value in reversed(values):
        head = Node(value, head)
    return head


def traced_build(builder, values):
    tracemalloc.start()
    before, _ = tracemalloc.get_traced_memory()
    result = builder(values)
    current, peak = tracemalloc.get_traced_memory()
    tracemalloc.stop()
    assert result is not None
    return current - before, peak - before


values = tuple(range(2_000))
measurements = {
    "list": traced_build(list, values),
    "deque": traced_build(deque, values),
    "linked": traced_build(build_linked, values),
}

assert all(peak >= retained >= 0 for retained, peak in measurements.values())
print("structures=3")
print("memory_relations_valid=True")
```

Output estable:

```text
structures=3
memory_relations_valid=True
```

El input tuple se crea antes de tracing. La medición aproxima allocations Python nuevas de cada container; no es RSS ni cifra universal. Ejecuta cada build aislado, conserva Python/hardware y no asumas que menor bytes implica mejor semántica.

### Tradeoff

¿Qué compra cada referencia `next`? ¿Qué pierde en locality y maintenance frente a una list estándar?

---

## 26. Selección guiada por workload

Antes de mirar la matriz, responde orden, acceso, extremos, búsqueda, frecuencia y ownership.

| Operación dominante | Candidato | Motivo / costo oculto |
|---|---|---|
| indexed access | `list` | O(1); inserts internos desplazan |
| LIFO | `list` / `deque` | extremos O(1) amortizado/práctico |
| FIFO | `deque` | append + popleft |
| membership | `set` | hashing de CS-M2 |
| key lookup | `dict` | derived index de CS-M2 |
| pointer manipulation educativa | linked list | nodes/invariantes/overhead |
| urgent-first | futuro heap | CS-M5, no FIFO |

Una queue no resuelve durability, retries, idempotency ni concurrency. Una stack no vuelve una acción reversible. Una linked list no entrega random access rápido.

### Elige

1. undo local acotado; 2. imports FIFO; 3. timeline con indexed reads; 4. ejercicio para detectar stale tail. Elige estructura y alternativa descartada.

---

## 27. Ejercicios guiados con solución razonada

Predice y dibuja antes de leer la solución. Cada solución separa semántica, referencias, costo e invariante.

### Ejercicio guiado 1 — Stack con list

**Objetivo:** demostrar LIFO.  
**Input:** push A/B/C, dos pops.  
**Predice:** C, B.  
**Solución razonada:** append coloca cada value al final; pop retira ese extremo; el último push sale primero; ambos son O(1) amortizado/práctico.  
**Criterio:** no usar índice 0.

### Ejercicio guiado 2 — Queue incorrecta con list

**Objetivo:** detectar costo oculto.  
**Input:** append + `pop(0)` repetido.  
**Predice:** FIFO correcto, O(n) por front removal.  
**Solución razonada:** la primera referencia sale; las restantes se desplazan; vaciar acumula `(n-1)+...+1`; el problema es representación, no semántica.  
**Criterio:** distinguir output correcto de costo.

### Ejercicio guiado 3 — Refactor a deque

**Objetivo:** conservar FIFO y cambiar costo.  
**Input:** reemplazar list/pop(0) por deque/popleft.  
**Predice:** mismo orden.  
**Solución razonada:** append añade back; popleft retira front; no desplaza todo el container; extremos O(1) práctico.  
**Criterio:** asserts comparan secuencia completa.

### Ejercicio guiado 4 — Primer Node

**Objetivo:** modelar referencias.  
**Input:** A→B→None.  
**Predice:** `head.next.value`.  
**Solución razonada:** head referencia A; A.next referencia B; B.next es None; asignar alias a B no copia.  
**Criterio:** diagrama coincide con objects.

### Ejercicio guiado 5 — Traversal

**Objetivo:** recorrer hasta None.  
**Input:** A→B→C.  
**Predice:** tres iterations.  
**Solución razonada:** current empieza en head y avanza por next; cada node se visita una vez; Θ(n) time, O(1) auxiliary sin cycle detection.  
**Criterio:** no usar indexed access inexistente.

### Ejercicio guiado 6 — Prepend

**Objetivo:** conservar chain.  
**Input:** B→C y new A.  
**Predice:** A→B→C.  
**Solución razonada:** primero new.next=head conserva B; después head=new publica A; dos relinks O(1).  
**Criterio:** el old head sigue alcanzable.

### Ejercicio guiado 7 — Append con tail

**Objetivo:** añadir sin traversal.  
**Input:** head/tail existentes y new C.  
**Predice:** old tail.next=C, tail=C.  
**Solución razonada:** ambas asignaciones son constantes; empty debe establecer head=tail=new; tail.next queda None.  
**Criterio:** prueba empty/one/many.

### Ejercicio guiado 8 — Find

**Objetivo:** demostrar búsqueda lineal.  
**Input:** target al final/ausente.  
**Predice:** hasta n comparisons.  
**Solución razonada:** no existe mapa posición→node; se siguen links desde head; worst Θ(n).  
**Criterio:** early return no cambia worst.

### Ejercicio guiado 9 — Remove first

**Objetivo:** separar search/relink.  
**Input:** quitar B de A→B→C.  
**Predice:** A→C.  
**Solución razonada:** search encuentra previous A/current B en O(n); A.next=C en O(1); detach B; total O(n).  
**Criterio:** size/tail/chain válidos.

### Ejercicio guiado 10 — One-element

**Objetivo:** evitar stale tail.  
**Input:** head=tail=A, remove A.  
**Predice:** ambos None.  
**Solución razonada:** head toma A.next=None; current es tail, entonces tail=previous=None; size pasa 0.  
**Criterio:** validate y append posterior pasan.

### Ejercicio guiado 11 — Cycle

**Objetivo:** diagnosticar sin colgar.  
**Input:** A→B→C→B.  
**Predice:** B aparece por segunda vez.  
**Solución razonada:** visited IDs detecta identidad repetida antes de avanzar indefinidamente; falla controlada.  
**Criterio:** test termina y reporta cycle.

### Ejercicio guiado 12 — Lost node

**Objetivo:** ordenar asignaciones.  
**Input:** insertar X después A.  
**Predice:** leer old next antes de sobrescribir.  
**Solución razonada:** `X.next=A.next`; luego `A.next=X`; invertirlas puede perder B/crear self-cycle.  
**Criterio:** B sigue alcanzable.

### Ejercicio guiado 13 — Linked stack

**Objetivo:** mapear stack a head.  
**Input:** push/pop.  
**Predice:** push=new→top; pop=top→top.next.  
**Solución razonada:** ambos relinks O(1); empty usa top None; cada push asigna Node.  
**Criterio:** compara con list y justifica cuál usar.

### Ejercicio guiado 14 — Linked queue

**Objetivo:** mapear FIFO a head/tail.  
**Input:** enqueue A/B, dequeue dos.  
**Predice:** A/B y luego head=tail=None.  
**Solución razonada:** enqueue enlaza tail; dequeue mueve head; al vaciar actualiza tail; extremos O(1).  
**Criterio:** invariant empty equivalente.

### Ejercicio guiado 15 — Standard library

**Objetivo:** evitar implementación manual por dogma.  
**Input:** pending queue de producto.  
**Predice:** `deque`.  
**Solución razonada:** ofrece semántica/costos requeridos, menos code y menos invariants manuales; linked queue queda educativa salvo trigger concreto.  
**Criterio:** decisión desde workload/maintenance.

### Ejercicio guiado 16 — Benchmark

**Objetivo:** medir operación correcta.  
**Input:** list queue vs deque queue.  
**Predice:** curvas distintas al crecer n.  
**Solución razonada:** mismos values, fill/drain, tamaños/repeticiones; conserva samples; no mezcla targets ni universaliza cruce.  
**Criterio:** hipótesis escrita y resultado semántico igual.

---

## 28. Ejercicios independientes

1. Clasifica seis workloads como LIFO/FIFO/indexed/linked.
2. Escribe Stack ADT sin nombrar representación.
3. Escribe Queue ADT con empty policy.
4. Implementa stack con list y prueba empty/one/many.
5. Predice una secuencia intercalada de push/pop.
6. Implementa queue con list y deriva costo de drain.
7. Refactoriza a deque conservando outputs.
8. Usa appendleft/pop y explica qué ADT resulta.
9. Implementa bounded queue síncrona con overflow explícito.
10. Explica por qué deque(maxlen) podría perder jobs.
11. Dibuja call stack A→B→C sin recursion.
12. Diseña undo sobre una vista derivada, no source.
13. Modela Node A→B→C con aliases.
14. Recorre una chain y cuenta nodes.
15. Busca target inicio/final/ausente.
16. Implementa prepend y prueba old head alcanzable.
17. Inserta tras node conocido y cuenta relinks.
18. Repite recibiendo target value y añade search cost.
19. Elimina después de node conocido con preconditions.
20. Elimina por value y separa fases.
21. Agrega tail y append O(1).
22. Escribe invariantes empty/non-empty.
23. Prueba one-element remove y append posterior.
24. Introduce stale tail y haz fallar validate.
25. Introduce cycle y detecta con visited IDs.
26. Reproduce lost chain sin loop infinito.
27. Corrige el orden de asignaciones.
28. Demuestra external node mutation.
29. Refactoriza find para devolver value, no Node.
30. Implementa `__iter__` y materialización opcional.
31. Dibuja doubly linked nodes e invariantes bidireccionales.
32. Diseña sentinel y enumera edge cases simplificados.
33. Implementa linked stack y compárala con list.
34. Implementa linked queue y compárala con deque.
35. Mide stack list/deque en tamaños crecientes.
36. Mide queue list/deque en tamaños crecientes.
37. Mide linked prepend/lookup/relink separado.
38. Compara memoria list/deque/nodes con límites.
39. Analiza workload 90% indexed / 10% insert.
40. Analiza workload con referencia directa y prepends dominantes.
41. Documenta cuándo no usar linked list en Python.
42. Construye replay worklist FIFO desde source inmutable.
43. Añade job IDs/retry count sin async.
44. Escribe tests de equivalencia entre implementaciones ADT.
45. Define trigger cuantitativo para revisar una queue/estructura.

---

## 29. Preguntas conceptuales

1. ¿Qué diferencia existe entre ADT e implementación?
2. ¿Qué propiedad define LIFO?
3. ¿Qué propiedad define FIFO?
4. ¿Por qué un stack basado en list es razonable?
5. ¿Qué significa pop empty bajo tu contrato?
6. ¿Por qué `list.pop(0)` cuesta O(n)?
7. ¿Qué problema resuelve `deque`?
8. ¿Cuándo list sigue siendo mejor que deque?
9. ¿Por qué urgent-first no es FIFO?
10. ¿Qué representa `next` en un Node?
11. ¿Por qué una linked list no ofrece indexed O(1)?
12. ¿Cuándo prepend es realmente O(1)?
13. ¿Cuándo insert-after es realmente O(1)?
14. ¿Por qué buscar + insertar sigue O(n)?
15. ¿Qué ocurre con un removed node referenciado por otro nombre?
16. ¿Qué invariante añade tail?
17. ¿Qué debe ocurrir al remover el único node?
18. ¿Cómo se forma un cycle accidental?
19. ¿Cómo detectas cycle sin colgar el test?
20. ¿Cómo se pierde una chain por assignment order?
21. ¿Qué riesgo introduce exponer Node?
22. ¿Qué significa ownership lógico de enlaces?
23. ¿Qué comprueba size almacenado?
24. ¿Para qué sirve `__iter__` aquí?
25. ¿Qué añade una doubly linked list?
26. ¿Qué edge cases simplifica un sentinel?
27. ¿Por qué linked nodes suelen tener peor locality?
28. ¿Qué overhead añade cada Node?
29. ¿Por qué Big O no basta para elegir linked list?
30. ¿Por qué una implementación manual puede ser pedagógica pero no default?
31. ¿Qué estructura elegirías para undo local?
32. ¿Qué estructura elegirías para pending FIFO?
33. ¿Qué no resuelve una queue acerca de durability/idempotency?
34. ¿Cómo separar setup de operación en un benchmark mutable?
35. ¿Por qué stack/queue in-memory no son source of truth?

---

## 30. Mini challenge — Coordinador síncrono de estructuras

### 30.1 Objetivo

Construye tres componentes pequeños y separados:

```text
UndoStack             → LIFO sobre vista derivada
PendingImports        → FIFO bounded con deque
EducationalLinkedList → referencias e invariantes
```

Debe resolverse con Programming Foundations + CS-M1–CS-M3.

### 30.2 UndoStack

- `push(action)`;
- `pop_or_none()`;
- `peek_or_none()`;
- `is_empty()`;
- límite configurable y policy explícita cuando se llena;
- acciones sintéticas que revierten solo una vista derivada.

Prueba empty, one y multiple. Source data no cambia.

### 30.3 PendingImports

- usa `deque`;
- `enqueue(job)` y `dequeue_or_none()`;
- FIFO estable;
- capacity positiva;
- overflow explícito, sin descarte silencioso;
- cada job tiene `job_id`, `source_id`, `retry_count`;
- no hay async, workers ni threads.

Prueba orden, empty, capacity y duplicate job policy elegida.

### 30.4 EducationalLinkedList

Implementa:

- `Node`;
- head, tail, size;
- append/prepend;
- find por value;
- remove first;
- `__iter__`;
- `validate` con cycle detection;
- ownership documentado.

No necesitas generics, doubly links ni sentinel.

### 30.5 Tests obligatorios

```text
empty
one element
multiple elements
remove head
remove tail
remove only
missing
stale tail injection
cycle injection con visited IDs
lost-node regression
external mutation diagnosis
```

El test de cycle debe terminar con error controlado; nunca con traversal infinito.

### 30.6 Análisis

Documenta:

- stack operations O(1) amortizado/práctico;
- deque extremes O(1) práctico;
- linked append/prepend O(1) con head/tail;
- linked find/remove by value O(n);
- search + relink;
- memory/locality tradeoff;
- por qué `deque/list` son alternativas estándar.

### 30.7 Benchmarks

1. stack list vs deque;
2. queue list `pop(0)` vs deque `popleft`;
3. linked prepend;
4. linked worst lookup;
5. relink after known node;
6. memory list/deque/linked.
7. linked list frente a list sobre al menos un workload equivalente y definido.

Usa tamaños crecientes, múltiples repeticiones y resultados semánticos equivalentes. Separa build cuando la pregunta lo requiera.

### 30.8 EIDOLON

Explica:

- source journal es authority;
- undo opera una vista/simulación;
- pending queue es work state efímero, no durability;
- linked chain es educativa;
- reiniciar proceso no debe presentarse como preservar esas estructuras.

### 30.9 Failure cases

1. queue con `pop(0)` sin analizar;
2. insert O(1) que oculta search;
3. stale tail;
4. cycle;
5. lost chain;
6. external node mutation;
7. benchmark que mide inputs distintos;
8. linked list elegida solo por slogan.

### 30.10 Criterio de aprobación

- LIFO/FIFO correctos;
- empty/capacity policies probadas;
- head/tail/size invariants pasan;
- one-element no deja tail stale;
- cycle test termina;
- lost-chain regression conserva todos los nodes;
- costos declaran preconditions;
- benchmarks miden workloads comparables;
- elección profesional favorece standard library salvo evidencia;
- ninguna estructura se vuelve source of truth;
- no requiere CS-M4+.

Output final:

```text
CS-M3 coordinator challenge: PASS
```

---

## 31. Resumen

- Stack/queue son ADTs; list/deque/nodes son representaciones.
- Stack es LIFO; queue es FIFO.
- `list.append/pop()` implementa un stack idiomático.
- `list.pop(0)` desplaza y cuesta O(n).
- `deque.append/popleft()` expresa una queue eficiente en extremos.
- Capacity explícita evita work queues ilimitadas; no añade concurrency.
- Node enlaza value con referencia `next`.
- Traversal y arbitrary search enlazados son O(n).
- Prepend/relink son O(1) solo con referencia apropiada.
- Search O(n) + relink O(1) sigue O(n).
- Tail permite append O(1) y añade invariantes.
- Remove del único node debe limpiar head/tail.
- Assignment order incorrecto puede perder chain o crear cycle.
- Visited IDs detectan cycle sin loop infinito.
- Exponer Node aumenta capacidad de romper invariantes.
- Doubly links añaden dirección, memoria e invariantes.
- Sentinel puede simplificar edges y añade convención.
- Linked lists manuales rara vez son default en Python.
- Locality, allocations y maintenance importan además de Big O.
- Benchmarks deben medir operaciones equivalentes y separar setup.
- Undo/work queues in-memory son proyecciones, no authority.

---

## 32. Checklist de dominio

- [ ] Puedo distinguir ADT de representación.
- [ ] Puedo predecir LIFO y FIFO.
- [ ] Puedo implementar stack con list.
- [ ] Puedo explicar empty policy.
- [ ] Puedo derivar `pop(0)` O(n).
- [ ] Puedo usar deque para FIFO.
- [ ] Puedo usar ambos extremos de deque deliberadamente.
- [ ] Puedo modelar capacity síncrona sin async.
- [ ] Puedo dibujar Node/head/tail/next.
- [ ] Puedo recorrer una chain O(n).
- [ ] Puedo explicar búsqueda enlazada O(n).
- [ ] Puedo hacer prepend O(1).
- [ ] Puedo declarar precondición de insert-after O(1).
- [ ] Puedo sumar search + relink.
- [ ] Puedo mantener head/tail/size.
- [ ] Puedo resolver one-element correctamente.
- [ ] Puedo detectar stale tail.
- [ ] Puedo detectar cycle con protección.
- [ ] Puedo corregir lost chain.
- [ ] Puedo diagnosticar external node mutation.
- [ ] Puedo explicar ownership lógico.
- [ ] Puedo implementar `SinglyLinkedList` educativa.
- [ ] Puedo iterar sin reabrir protocolo completo.
- [ ] Puedo explicar doubly links y sentinel.
- [ ] Puedo comparar linked list con list/deque.
- [ ] Puedo justificar standard library como default.
- [ ] Puedo benchmarkear stack/queue/linked operations.
- [ ] Puedo interpretar memory measurements prudentemente.
- [ ] Puedo preservar source authority.
- [ ] Puedo completar el mini challenge con PF + CS-M1–CS-M3.

---

## 33. Preparación para labs y EIDOLON 0.0b

### CS-L04 — Undo stack

Secciones 3–4 y ejercicios 1/13 preparan LIFO, empty semantics, límite y undo sobre una vista sin borrar Events.

### CS-L05 — Consolidation queue

Secciones 5–8/19–20 y ejercicios 3/14/15 preparan deque bounded, FIFO, job IDs, retry count conceptual y overflow. Idempotency/backpressure completo se integra con PF-M8; este módulo no añade concurrency.

### CS-L06 — Linked list reality check

Secciones 9–18/22–25 y ejercicios 4–12/16 preparan nodes, invariants, traversal, memory/locality benchmark y justificación explícita de por qué no usar una linked list manual.

### EIDOLON 0.0b

CS-M3 aporta orden de work y fallas de referencias. No agrega persistence ni cambia source authority. CS-MP2 requerirá también CS-M5 para prioridades y módulos posteriores para concurrencia/cancelación.

### Evidencia antes de CS-M4

1. dieciséis ejercicios guiados reproducidos;
2. al menos veinte independientes, incluidos 1–10, 13–29, 33–38, 42–45;
3. stack/queue standard con empty/capacity tests;
4. linked list con validate/cycle/stale-tail tests;
5. benchmark stack/queue/linked/memory;
6. explicación search + relink;
7. decisión documentada standard vs manual;
8. demostración source/projection.

---

## 34. Recursos de ampliación

El módulo es autocontenido para revisión. Para ampliar usa los recursos canónicos de [`CS.11`](../../02_curriculum/02_computer_science_foundations.md#cs11-recursos-recomendados): MIT OCW 6.006 y CLRS para ADTs/linked structures; *The Algorithm Design Manual* para elección por workload.

La documentación oficial de Python sobre `list` y `collections.deque` sirve para comprobar contratos. No sustituye medir el workload ni probar invariantes.

---

## 35. Límite explícito del módulo

CS-M3 termina en stack, queue, deque, nodes, singly linked list educativa, doubly/sentinel conceptuales, invariantes, failure cases y benchmarks.

Quedan fuera:

- recursion, binary search y sorting: CS-M4;
- trees, heaps y priority queues: CS-M5;
- graphs, BFS/DFS y state machines: CS-M6;
- memory/filesystem/OS internals: CS-M7;
- processes/threads/races: CS-M8;
- networking: CS-M9;
- architecture/cache lines: CS-M10.

No se introducen databases, backend, frontend, AI, concurrency ni linked collections industriales/genéricas.

El siguiente paso permitido es revisar CS-M3 como `review candidate`. **No se crea ni se desarrolla CS-M4 en esta entrega.**
