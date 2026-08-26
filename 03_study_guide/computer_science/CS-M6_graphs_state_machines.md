# CS-M6 — Graphs, lógica, conjuntos y máquinas de estado

**Track:** Computer Science Foundations  
**Competencias:** D2.2; soporte D2.1, D2.3  
**Fase:** P0  
**Nivel objetivo:** Aplicado  
**Prerequisites:** PF-M1, PF-M2, PF-M3, PF-M5, PF-M9, CS-M1, CS-M2, CS-M3, CS-M4, CS-M5  
**Build:** EIDOLON 0.0b  
**Curriculum source:** [CS-M6](../../02_curriculum/02_computer_science_foundations.md#cs-m6--graphs-lógica-conjuntos-y-máquinas-de-estado)  
**Status:** approved

Una persona puede conectarse con muchas otras por múltiples caminos; una entidad puede cambiar de state, pero no mediante cualquier transition. Una secuencia o un tree no expresan esas reglas generales por sí solos.

```text
relación o lifecycle
↓
modelo explícito
↓
invariantes
↓
representación
↓
traversal / transition
↓
correctness y complejidad
↓
EIDOLON derived view + provenance
```

CS-M6 construye adjacency lists, BFS/DFS y finite state machines pequeñas. No introduce graph databases, algoritmos weighted avanzados, OS, concurrency, networking, backend ni AI.

## Resultados de aprendizaje

Al terminar podrás:

- distinguir vertex identity de edge semantics;
- modelar directed, undirected y weighted edges bajo políticas explícitas;
- validar IDs, referencias, symmetry, duplicates, self-loops y metadata;
- elegir adjacency list o matrix desde density y queries;
- calcular degree, indegree y outdegree;
- explicar path, reachability, component y cycle;
- implementar BFS con `deque`, visited y neighbor order determinista;
- reconstruir un shortest path por cantidad de edges en graph no ponderado;
- implementar DFS recursivo e iterativo sin repetir cycles;
- derivar O(V + E) time y O(V) auxiliary space bajo adjacency list;
- encontrar connected components en undirected graph;
- detectar un cycle undirected con parent y uno directed con visiting/completed;
- preservar edge type/provenance sin promover inferencias a hechos;
- definir states, actions y transition table con `Enum`;
- rechazar invalid transitions antes de cambiar state;
- separar transition pure function de effects;
- reconstruir current derived state desde ordered append-only history;
- verificar deterministic replay bajo rules/version estables;
- detectar reachable/unreachable states y cycles permitidos por policy;
- mantener graph, current state e índices como derivados de source/history.

## Cómo estudiar este módulo

1. Define primero si cada edge es directed, symmetric, typed y/o weighted.
2. Declara qué referencias, duplicates y self-loops acepta el graph.
3. Traza queue/stack y visited antes de ejecutar.
4. Ordena neighbors solo si el output necesita determinismo y contabiliza el costo.
5. En FSM, escribe la transition table antes del código.
6. Valida una transition antes de append/mutation.
7. Separa source/history de adjacency/current derived state.

### Convenciones

- **Ejemplo ejecutable:** bloque autónomo con output estable o `assert`.
- **Failure case ejecutable:** falla de forma controlada; nunca se deja un loop infinito.
- **Modelo educativo:** implementación in-memory para aprender, no arquitectura persistente.
- **Benchmark ejecutable:** fija propiedades, no tiempos ambientales.
- **Fragmento:** requiere contexto omitido.
- **Continuación:** depende del bloque inmediatamente anterior o del challenge del estudiante.

Baseline: Python 3.14 y standard library.

---

## 1. Relaciones generales y terminología

```text
Alice ─── Bob
  │       │
  │       Diego
  │       │
  └── Carla
```

Alice y Diego están conectados por más de un path. Diego tiene más de un neighbor que podría verse como “parent” bajo una jerarquía artificial. Un graph permite esta conectividad general.

- **vertex/node:** entidad con identidad estable;
- **edge:** relación entre vertices;
- **path:** secuencia de vertices unida por edges válidos;
- **cycle:** path que regresa a su origen sin reutilizar arbitrariamente el mismo paso;
- **reachable:** existe algún path desde source a target;
- **connected component:** grupo maximal conectado en un undirected graph;
- **degree:** cantidad de incident edges/neighbors en un simple undirected graph;
- **indegree/outdegree:** incoming/outgoing edges en directed graph.

Usa `person_id`, `event_id` o `claim_id` como identity. Un display name puede cambiar o colisionar.

### Modela

¿“A conoce B” es necesariamente symmetric? Define la semántica antes de escoger directed/undirected.

---

## 2. Directed, undirected, weighted y edge policy

```text
undirected: A — B       implica ambos sentidos bajo esa relación
directed:   A → B       no implica B → A
weighted:   A -3.5→ B  añade cost/weight al edge
```

Weights pueden representar distance, latency o relevance sintética. CS-M6 no implementa Dijkstra, A* ni otros shortest paths weighted.

Un graph model también debe decidir:

- ¿permite self-loop `A → A`?
- ¿colapsa duplicate edge?
- ¿permite parallel edges con metadata distinta?
- ¿una edge puede referenciar vertex inexistente?
- ¿qué edge types son válidos?

Este módulo usa simple graphs: adjacency `set` colapsa duplicate endpoints, self-loops se rechazan y toda referencia debe existir. Si repeated evidence importa, se conserva en source/provenance; no se pierde silenciosamente dentro de un set.

### Representa

Dos records sustentan la misma relation derivada. ¿Es un duplicate edge o dos evidencias? Separa estructura algorítmica de source evidence.

---

## 3. Graph invariants y adjacency list

Una adjacency list con `dict[str, set[str]]` es natural para sparse graphs:

```python
graph = {
    "A": {"B", "C"},
    "B": {"A"},
    "C": {"A"},
}
```

Para el contrato undirected simple:

1. cada vertex ID es único;
2. todo neighbor existe como vertex;
3. `v in graph[u]` implica `u in graph[v]`;
4. no hay self-loops;
5. sets colapsan duplicate endpoints bajo policy explícita.

**Modelo educativo ejecutable:**

```python
class UndirectedGraph:
    def __init__(self) -> None:
        self._adjacency: dict[str, set[str]] = {}

    def add_vertex(self, vertex: str) -> None:
        if vertex in self._adjacency:
            raise ValueError(f"duplicate vertex: {vertex}")
        self._adjacency[vertex] = set()

    def add_edge(self, left: str, right: str) -> None:
        if left == right:
            raise ValueError("self-loop rejected")
        if left not in self._adjacency or right not in self._adjacency:
            raise KeyError("edge references missing vertex")
        if right in self._adjacency[left]:
            raise ValueError("duplicate edge")
        self._adjacency[left].add(right)
        self._adjacency[right].add(left)

    def neighbors(self, vertex: str) -> tuple[str, ...]:
        if vertex not in self._adjacency:
            raise KeyError(vertex)
        return tuple(sorted(self._adjacency[vertex]))

    def validate(self) -> bool:
        for vertex, neighbors in self._adjacency.items():
            if vertex in neighbors:
                return False
            for neighbor in neighbors:
                if neighbor not in self._adjacency:
                    return False
                if vertex not in self._adjacency[neighbor]:
                    return False
        return True


graph = UndirectedGraph()
for vertex in ("A", "B", "C"):
    graph.add_vertex(vertex)
graph.add_edge("A", "B")
graph.add_edge("A", "C")

print(graph.neighbors("A"))
print(graph.validate())
```

Output:

```text
('B', 'C')
True
```

Add vertex/edge valida antes de modificar ambas direcciones. Remove vertex tendría que limpiar todas las incoming references; borrarlo solo del `dict` deja dangling neighbors.

### Invariante

Si falla la segunda inserción de una undirected edge, ¿qué rollback lógico necesitas para no conservar una mitad?

---

## 4. Adjacency matrix, density y degree

Para vertices `[A,B,C]`, una matrix undirected puede ser:

```text
    A B C
A   0 1 1
B   1 0 0
C   1 0 0
```

La matrix ocupa O(V²) cells aunque existan pocas edges. Edge existence es O(1); iterar neighbors inspecciona O(V). Una adjacency list típica ocupa O(V + E); con neighbor `set`, edge membership es expected O(1) bajo hashing y neighbor iteration es proporcional al degree.

**Ejemplo ejecutable — degree dirigido:**

```python
directed = {
    "A": {"B", "C"},
    "B": {"C"},
    "C": set(),
}

outdegree = {vertex: len(neighbors) for vertex, neighbors in directed.items()}
indegree = {vertex: 0 for vertex in directed}
for neighbors in directed.values():
    for neighbor in neighbors:
        indegree[neighbor] += 1

print(outdegree)
print(indegree)
```

Output:

```text
{'A': 2, 'B': 1, 'C': 0}
{'A': 0, 'B': 1, 'C': 2}
```

Un sparse graph tiene E muy por debajo del máximo posible; uno dense se acerca. Relationship graphs iniciales de EIDOLON probablemente sean sparse, pero eso es hipótesis a medir, no arquitectura definitiva.

### Representa

El workload consulta millones de posibles pairs y casi todos tienen edge. ¿Qué ventaja de matrix empieza a importar y qué costo permanece?

---

## 5. Path, cycle y tree como caso especial

```text
A → B → C → A    cycle dirigido
A → D            otro path
```

Direct neighbor no equivale a reachable: D puede estar a varios edges. Path length cuenta edges.

Un undirected tree es connected y acyclic; entre dos vertices hay un path único. Un directed rooted tree añade orientación jerárquica según contrato. General graph puede tener cycles, múltiples paths y múltiples incoming edges.

Un cycle no es automáticamente bug: puede representar mutual relationship o lifecycle `ACTIVE → PAUSED → ACTIVE`. En dependency imports suele ser indeseable. La semántica decide.

### Cycle

Clasifica un cycle de friendship, uno de module imports y uno de state machine. ¿Qué invariant cambia?

---

## 6. Traversal y el `visited` set

Pregunta central:

> ¿Qué vertices son reachable desde A?

Sin `visited`, un cycle `A—B—C—A` vuelve a encolar/apilar vertices indefinidamente. Multiple paths también duplican trabajo. Regla:

```text
descubrir vertex
→ marcar visited
→ agregarlo a pending una sola vez
```

Marcar al descubrir, no al retirar, evita que dos parents agreguen el mismo neighbor.

Si adjacency usa sets, iteration order no es contrato. Para output/replay determinista, estos ejemplos usan `sorted(neighbors)`, añadiendo sorting local. Otra representación ordered puede ser mejor si el dominio exige ese orden frecuentemente.

### Visited

En diamond `A→B, A→C, B→D, C→D`, ¿cuándo debe marcarse D para no entrar dos veces en queue?

---

## 7. BFS: queue, levels y shortest path no ponderado

Breadth-First Search explora distance 0, luego 1, luego 2. Usa queue FIFO.

```text
graph: A—B, A—C, B—D, C—D

step current queue_after visited
0    -       [A]         {A}
1    A       [B,C]       {A,B,C}
2    B       [C,D]       {A,B,C,D}
3    C       [D]         {A,B,C,D}
4    D       []          {A,B,C,D}
```

**Ejemplo ejecutable — orden y parent map:**

```python
from collections import deque


def bfs(
    graph: dict[str, set[str]],
    start: str,
) -> tuple[list[str], dict[str, str | None]]:
    if start not in graph:
        raise KeyError(start)

    order: list[str] = []
    parents: dict[str, str | None] = {start: None}
    queue = deque([start])

    while queue:
        current = queue.popleft()
        order.append(current)
        for neighbor in sorted(graph[current]):
            if neighbor not in graph:
                raise ValueError(f"dangling vertex: {neighbor}")
            if neighbor not in parents:
                parents[neighbor] = current
                queue.append(neighbor)
    return order, parents


graph = {
    "A": {"B", "C"},
    "B": {"A", "D"},
    "C": {"A", "D"},
    "D": {"B", "C"},
}
order, parents = bfs(graph, "A")
print(order)
print(parents)
```

Output:

```text
['A', 'B', 'C', 'D']
{'A': None, 'B': 'A', 'C': 'A', 'D': 'B'}
```

En un unweighted graph, el primer descubrimiento ocurre por un path con mínimo número de edges. Esto no resuelve weights.

### Traza

Cambia neighbor policy a reverse sorted. ¿Puede cambiar el parent elegido para D sin cambiar shortest distance?

---

## 8. Reconstrucción de path y reachability

Un parent map guarda cómo se descubrió cada vertex. Se reconstruye desde target hacia source y luego se invierte.

**Ejemplo ejecutable:**

```python
from collections import deque


def shortest_path(
    graph: dict[str, set[str]], start: str, target: str
) -> list[str] | None:
    if start not in graph or target not in graph:
        raise KeyError("start or target missing")
    parents: dict[str, str | None] = {start: None}
    queue = deque([start])

    while queue:
        current = queue.popleft()
        if current == target:
            path: list[str] = []
            cursor: str | None = target
            while cursor is not None:
                path.append(cursor)
                cursor = parents[cursor]
            return list(reversed(path))
        for neighbor in sorted(graph[current]):
            if neighbor not in parents:
                parents[neighbor] = current
                queue.append(neighbor)
    return None


graph = {"A": {"B", "C"}, "B": {"D"}, "C": {"D"}, "D": set(), "X": set()}
print(shortest_path(graph, "A", "D"))
print(shortest_path(graph, "A", "X"))
```

Output:

```text
['A', 'B', 'D']
None
```

Reachability puede terminar temprano al descubrir/encontrar target; worst case sigue O(V + E).

### Reachability

¿A puede llegar a X? ¿X puede llegar a A en este directed graph?

---

## 9. DFS recursivo e iterativo

DFS avanza por un path y retrocede. Puede usar call stack o stack explícito.

**Ejemplo ejecutable:**

```python
def dfs_recursive(graph: dict[str, set[str]], start: str) -> list[str]:
    if start not in graph:
        raise KeyError(start)
    visited: set[str] = set()
    order: list[str] = []

    def visit(current: str) -> None:
        visited.add(current)
        order.append(current)
        for neighbor in sorted(graph[current]):
            if neighbor not in visited:
                visit(neighbor)

    visit(start)
    return order


def dfs_iterative(graph: dict[str, set[str]], start: str) -> list[str]:
    if start not in graph:
        raise KeyError(start)
    visited: set[str] = set()
    order: list[str] = []
    stack = [start]
    while stack:
        current = stack.pop()
        if current in visited:
            continue
        visited.add(current)
        order.append(current)
        stack.extend(
            neighbor
            for neighbor in sorted(graph[current], reverse=True)
            if neighbor not in visited
        )
    return order


graph = {"A": {"B", "C"}, "B": {"A", "D"}, "C": {"A"}, "D": {"B"}}
print(dfs_recursive(graph, "A"))
print(dfs_iterative(graph, "A"))
```

Output:

```text
['A', 'B', 'D', 'C']
['A', 'B', 'D', 'C']
```

Recursive DFS puede alcanzar recursion depth en un graph profundo. Iterative DFS hace pending explícito. Ninguno es universalmente superior.

### Determinismo

¿Por qué se hace `reverse=True` al extender un LIFO stack si queremos el neighbor menor primero?

---

## 10. Components en undirected graph

Recorre todos los vertices. Cada no visitado inicia un traversal que descubre un connected component.

**Ejemplo ejecutable:**

```python
def connected_components(graph: dict[str, set[str]]) -> list[tuple[str, ...]]:
    visited: set[str] = set()
    components: list[tuple[str, ...]] = []

    for start in sorted(graph):
        if start in visited:
            continue
        component: list[str] = []
        stack = [start]
        visited.add(start)
        while stack:
            current = stack.pop()
            component.append(current)
            for neighbor in sorted(graph[current], reverse=True):
                if neighbor not in visited:
                    visited.add(neighbor)
                    stack.append(neighbor)
        components.append(tuple(sorted(component)))
    return components


graph = {"A": {"B"}, "B": {"A"}, "C": set(), "D": {"E"}, "E": {"D"}}
print(connected_components(graph))
```

Output:

```text
[('A', 'B'), ('C',), ('D', 'E')]
```

No generalices esta definición a strongly connected components de directed graphs; queda fuera del módulo.

### Complejidad

¿Por qué iniciar varios traversals no multiplica a O(V·(V+E)) cuando `visited` es global?

---

## 11. Cycle detection y dependency graph

En undirected DFS, volver a un visited neighbor distinto del parent revela cycle. En directed DFS necesitamos distinguir:

```text
unvisited
visiting  (en current recursion path)
completed
```

Una edge hacia `visiting` es back edge/cycle.

**Ejemplo ejecutable — directed cycle:**

```python
def has_directed_cycle(graph: dict[str, set[str]]) -> bool:
    visiting: set[str] = set()
    completed: set[str] = set()

    def visit(vertex: str) -> bool:
        if vertex in visiting:
            return True
        if vertex in completed:
            return False
        visiting.add(vertex)
        for neighbor in graph[vertex]:
            if neighbor not in graph:
                raise ValueError("dangling vertex")
            if visit(neighbor):
                return True
        visiting.remove(vertex)
        completed.add(vertex)
        return False

    return any(visit(vertex) for vertex in graph if vertex not in completed)


acyclic = {"A": {"B"}, "B": {"C"}, "C": set()}
circular_imports = {"A": {"B"}, "B": {"C"}, "C": {"A"}}
print(has_directed_cycle(acyclic))
print(has_directed_cycle(circular_imports))
```

Output:

```text
False
True
```

`A imports B`, `B imports C`, `C imports A` conecta con PF-M4. El graph hace visible el cycle; resolverlo exige redirigir responsabilidades, no esconder imports por reflejo.

### Cycle

¿Por qué un edge hacia `completed` no prueba cycle en directed DFS?

---

## 12. Mutation, typed edges y provenance

Edge identity y semantics son distintas. Una adjacency list de endpoints no conserva type ni evidence.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Edge:
    source_id: str
    target_id: str
    relation: str
    provenance_id: str


allowed_relations = {"supports", "refutes", "mentions", "derived_from"}
vertices = {"claim-A", "claim-B", "evidence-1"}
edge = Edge("claim-A", "evidence-1", "supports", "record-007")

assert edge.source_id in vertices and edge.target_id in vertices
assert edge.relation in allowed_relations
assert edge.provenance_id
print(edge.relation, edge.provenance_id)
```

Output:

```text
supports record-007
```

Un derived edge no reemplaza `record-007`. Rebuild vuelve a leer source/evidence y reconstruye adjacency/metadata. Remove vertex debe limpiar outgoing/incoming derived edges o rechazar la operación.

### Provenance

¿Qué se pierde si adjacency conserva solo target ID y descarta relation/provenance?

---

## 13. Complejidad de representación y traversal

Con adjacency list:

- space O(V + E) para directed; undirected suele almacenar cada edge dos veces, mismo orden asintótico;
- BFS/DFS O(V + E) sobre la parte alcanzable o todo graph;
- visited y queue/stack O(V) worst case;
- iterate neighbors O(degree);
- edge membership expected O(1) si neighbors es `set`, bajo hashing.

Con matrix:

- space O(V²);
- edge check O(1);
- neighbor scan O(V), incluso con degree pequeño.

Sorting neighbors para determinismo añade trabajo. Una bound útil es la suma de `d(v) log d(v)` sobre vertices visitados; no escondas ese costo dentro de O(V+E) si domina.

### Complejidad

Define V y E para un undirected graph almacenado simétricamente. ¿Contarás logical edges una vez o adjacency entries dos veces? Sé consistente.

---

## 14. Benchmark controlado de sparse/dense representations

Este experimento pequeño compara edge checks y space lógico; no decide arquitectura universal.

**Benchmark ejecutable:**

```python
from random import Random
from time import perf_counter


def build_graph(size: int, edge_count: int) -> tuple[list[set[int]], list[list[bool]]]:
    adjacency = [set() for _ in range(size)]
    matrix = [[False] * size for _ in range(size)]
    rng = Random(size + edge_count)
    added = 0
    while added < edge_count:
        source = rng.randrange(size)
        target = rng.randrange(size)
        if source == target or target in adjacency[source]:
            continue
        adjacency[source].add(target)
        matrix[source][target] = True
        added += 1
    return adjacency, matrix


def measure(size: int, edge_count: int) -> tuple[float, float, int, int]:
    adjacency, matrix = build_graph(size, edge_count)
    pairs = [(index % size, (index * 7 + 3) % size) for index in range(500)]

    started = perf_counter()
    list_hits = sum(target in adjacency[source] for source, target in pairs)
    list_seconds = perf_counter() - started

    started = perf_counter()
    matrix_hits = sum(matrix[source][target] for source, target in pairs)
    matrix_seconds = perf_counter() - started

    assert list_hits == matrix_hits
    adjacency_slots = size + sum(len(neighbors) for neighbors in adjacency)
    matrix_slots = size * size
    return list_seconds, matrix_seconds, adjacency_slots, matrix_slots


rows = [measure(80, edges) for edges in (160, 3_000)]
times = [value for row in rows for value in row[:2]]
print(f"measurements={len(times)}")
print(f"times_non_negative={all(value >= 0 for value in times)}")
print(rows[0][2] < rows[0][3])
```

Output estable:

```text
measurements=4
times_non_negative=True
True
```

`matrix_slots` cuenta booleans lógicos, no bytes reales de Python. Usa `tracemalloc` si la pregunta es peak allocation; conserva esa limitación.

### Representa

¿Por qué la matrix sigue reservando V² en ambos rows aunque E cambie?

---

## 15. Experimento EIDOLON: relationship graph derived

```text
source relationship records
↓ validate IDs/type/provenance
derived adjacency + edges_by_pair
↓
BFS/DFS, reachability, components
```

Person IDs son vertices; display names son attributes aparte. El graph solo representa relaciones sustentadas por source records. No infiere psicología ni convierte path en causalidad.

Una edge `Claim A → refutes → Claim B` es directed. Su existencia no convierte ninguna claim en hecho. Rebuild debe producir la misma adjacency bajo la misma ordered source y rules/version.

### Determinismo

¿Qué outputs deben ser sets de reachability y cuáles requieren order reproducible? No impongas sorting si el contrato solo pide pertenencia.

---

## 16. Del graph al lifecycle explícito

Un objeto puede estar en un state discreto, pero no toda asignación es válida:

```text
PENDING → VALIDATING → READY → IMPORTED
              └──────→ REJECTED
```

Este import lifecycle es **sintético**. Enseña mecanismo, no estados canónicos persistentes de EIDOLON.

Una transition combina:

```text
current state + action → next state
```

Una deterministic FSM práctica ofrece como máximo un next state para cada pair `(state, action)`. No estudiamos languages/automata theory.

Representar mutually exclusive state con `Enum` evita combinations como `active=True, rejected=True, deleted=True`. FSM no es universal: flags independientes siguen siendo correctas si el dominio permite combinarlas.

### State

¿`retry_count` es state de la FSM o data auxiliar? Decide si altera qué transitions son válidas.

---

## 17. Transition table y pure function

Una tabla centraliza reglas que múltiples `if/elif` dispersos esconderían.

**Ejemplo ejecutable:**

```python
from enum import Enum


class ImportState(Enum):
    PENDING = "pending"
    VALIDATING = "validating"
    READY = "ready"
    IMPORTED = "imported"
    REJECTED = "rejected"


class ImportAction(Enum):
    START_VALIDATION = "start_validation"
    ACCEPT = "accept"
    REJECT = "reject"
    IMPORT = "import"


TRANSITIONS: dict[tuple[ImportState, ImportAction], ImportState] = {
    (ImportState.PENDING, ImportAction.START_VALIDATION): ImportState.VALIDATING,
    (ImportState.VALIDATING, ImportAction.ACCEPT): ImportState.READY,
    (ImportState.VALIDATING, ImportAction.REJECT): ImportState.REJECTED,
    (ImportState.READY, ImportAction.IMPORT): ImportState.IMPORTED,
}


def next_state(current: ImportState, action: ImportAction) -> ImportState:
    key = (current, action)
    if key not in TRANSITIONS:
        raise ValueError(f"invalid transition: {current.value} + {action.value}")
    return TRANSITIONS[key]


state = ImportState.PENDING
for action in (
    ImportAction.START_VALIDATION,
    ImportAction.ACCEPT,
    ImportAction.IMPORT,
):
    state = next_state(state, action)
print(state.value)

try:
    next_state(ImportState.PENDING, ImportAction.IMPORT)
except ValueError as error:
    print(error)
```

Output:

```text
imported
invalid transition: pending + import
```

`next_state` hace lookup/validation y no I/O. El caller aplica effects después de obtener una transition válida. Invalid transition se rechaza antes de mutar state o history.

Con dict lookup, la transición es expected O(1) bajo hashing. Para FSM pequeñas, correctness/testability importa más que micro-optimizar.

### State

¿Qué terminal states tiene esta policy? Aquí `IMPORTED` y `REJECTED` no tienen outgoing entries y son terminales por tabla, no por su nombre.

---

## 18. Invariantes, terminal states y transition graph

Invariantes:

- current pertenece al `Enum`;
- `(current, action)` existe;
- terminal no tiene outgoing transitions bajo policy;
- cada history record encadena `from_state` con el `to_state` anterior;
- action y result coinciden con la tabla/rules version.

La tabla también forma un directed graph:

```text
PENDING --start_validation--> VALIDATING
VALIDATING --accept---------> READY
VALIDATING --reject---------> REJECTED
READY --import--------------> IMPORTED
```

Graph traversal pregunta qué states son reachable. FSM execution pregunta qué transition permite el pair actual/action. Mismo shape dirigido, semántica distinta.

### State

¿Agregar `REJECTED → PENDING` crea un bug? Solo la lifecycle policy puede decidir si retry cycle es legal.

---

## 19. Append-only history y deterministic replay

Current state puede derivarse de ordered transition history:

```text
initial + ordered actions + rules/version → current
```

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass
from enum import Enum


class State(Enum):
    PENDING = "pending"
    VALIDATING = "validating"
    READY = "ready"
    IMPORTED = "imported"
    REJECTED = "rejected"


class Action(Enum):
    START = "start"
    ACCEPT = "accept"
    REJECT = "reject"
    IMPORT = "import"


RULES: dict[tuple[State, Action], State] = {
    (State.PENDING, Action.START): State.VALIDATING,
    (State.VALIDATING, Action.ACCEPT): State.READY,
    (State.VALIDATING, Action.REJECT): State.REJECTED,
    (State.READY, Action.IMPORT): State.IMPORTED,
}


@dataclass(frozen=True)
class TransitionRecord:
    from_state: State
    action: Action
    to_state: State
    rules_version: int


def apply_action(current: State, action: Action) -> tuple[State, TransitionRecord]:
    target = RULES.get((current, action))
    if target is None:
        raise ValueError("invalid transition")
    return target, TransitionRecord(current, action, target, rules_version=1)


def replay(initial: State, history: list[TransitionRecord]) -> State:
    current = initial
    for record in history:
        if record.rules_version != 1:
            raise ValueError("unsupported rules version")
        expected = RULES.get((current, record.action))
        if record.from_state != current or record.to_state != expected:
            raise ValueError("inconsistent history")
        current = record.to_state
    return current


state = State.PENDING
history: list[TransitionRecord] = []
for action in (Action.START, Action.ACCEPT, Action.IMPORT):
    state, record = apply_action(state, action)
    history.append(record)

print(state.value)
print(replay(State.PENDING, history).value)
print(replay(State.PENDING, history) == replay(State.PENDING, history))
```

Output:

```text
imported
imported
True
```

History es append-only source para este modelo sintético; `state` es projection. Mismo initial, ordered history y rules version produce mismo result. Procesar history desde `set` rompe order contractual. Si rules cambian sin version, replay histórico puede cambiar: sistemas reales necesitan version/migration policy, fuera del detalle de CS-M6.

### Replay

Intercambia `ACCEPT` e `IMPORT`. ¿Dónde se rechaza y por qué no debe appendearse el segundo record?

---

## 20. Reachable y unreachable states

La transition table puede proyectarse a adjacency por states y recorrerse.

**Ejemplo ejecutable:**

```python
from collections import deque


states = {"PENDING", "VALIDATING", "READY", "IMPORTED", "REJECTED", "ORPHAN"}
transition_graph = {
    "PENDING": {"VALIDATING"},
    "VALIDATING": {"READY", "REJECTED"},
    "READY": {"IMPORTED"},
    "IMPORTED": set(),
    "REJECTED": set(),
    "ORPHAN": set(),
}


def reachable(graph: dict[str, set[str]], initial: str) -> set[str]:
    visited = {initial}
    queue = deque([initial])
    while queue:
        current = queue.popleft()
        for neighbor in graph[current]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
    return visited


reachable_states = reachable(transition_graph, "PENDING")
print(sorted(reachable_states))
print(sorted(states - reachable_states))
```

Output:

```text
['IMPORTED', 'PENDING', 'READY', 'REJECTED', 'VALIDATING']
['ORPHAN']
```

Un unreachable state puede ser diseño innecesario, missing transition o bug. No hacemos automata minimization.

### State

Si `ORPHAN` es un initial state alternativo válido, ¿sigue siendo unreachable? Reachability siempre depende del source set.

---

## 21. MemoryCandidate canónico y correction sintética

El Curriculum define para **MemoryCandidate** transitions `proposed → approved/rejected → superseded/purged`. La tabla concreta debe aclarar qué branches se permiten; el lab CS-L11 fijará y probará la policy sin inventar promotions silenciosas.

```text
PROPOSED → APPROVED → SUPERSEDED → PURGED
    └────→ REJECTED ────────────→ PURGED   (solo si policy lo declara)
```

Este diagrama es una **policy educativa candidata**, no una modificación silenciosa del modelo canónico. `purged` no significa borrar source evidence: puede describir lifecycle de una projection mientras provenance/history se conserva según reglas superiores.

Un correction lifecycle `PROPOSED → APPLIED → SUPERSEDED` puede usarse solo como ejemplo sintético; no se declara canónico aquí.

### Provenance

¿Qué debe conservarse aunque current candidate state sea `superseded` o `purged`?

---

## 22. Giant `if/elif`, side effects y boolean combinations

**Código incorrecto:**

```python
if status == "pending" and action == "start":
    status = "validating"
    print("started")
elif status == "validating" and action == "accept":
    save_to_database()
    status = "ready"
# reglas continúan dispersas...
```

Problemas: magic strings, validation mezclada con I/O, mutation parcial si effect falla y reglas difíciles de enumerar. Refactor:

```text
pure next_state(current, action)
↓
append valid transition / update projection
↓
effects explícitos en otra frontera
```

No desarrollamos persistence/transactions. La separación permite unit tests de todas las pairs.

### Determinismo

¿Qué I/O o clock access dentro de `next_state` haría replay dependiente del ambiente?

---

## 23. Failure cases de graph

1. **Traversal sin visited.** Un cycle repite indefinidamente. Reproduce con step limit, nunca cuelgues el test.
2. **Set order como output.** Puede variar; sort neighbors o define ordered representation.
3. **Adjacency undirected incompleta.** `A→B` sin `B→A` viola symmetry del contrato.
4. **Dangling vertex.** Decide reject/auto-create/permit; aquí se rechaza.
5. **Tree algorithm sobre graph.** Multiple parents/cycles rompen path único y base assumptions.
6. **BFS con `list.pop(0)`.** Produce shifting O(n) repetido; usa `deque.popleft()`.
7. **Recursive DFS profundo.** Puede alcanzar recursion limit; usa stack explícito cuando depth lo exige.
8. **Duplicate edge colapsada sin analizar evidence.** Set no sustituye provenance records.
9. **Derived adjacency como source.** Pierde rebuild/audit; source records conservan autoridad.
10. **BFS shortest path weighted.** Minimiza edge count, no total weight.

### Visited

Diseña un failure case con límite de diez pops que demuestre repetición sin dejar un proceso colgado.

---

## 24. Failure cases de state machine

1. **Invalid transition aceptada.** Valida antes de mutation/append.
2. **Múltiples booleans contradictorios.** Usa Enum si states son mutually exclusive.
3. **Replay desde order incidental.** History exige sequence explícita.
4. **Transition function con I/O.** Pierde aislamiento y determinismo.
5. **Current state como único source.** Pierde history/provenance bajo modelo replayable.
6. **Rules drift sin version.** Mismo history puede producir otro result.
7. **Terminal por nombre.** Terminal es ausencia de outgoing transitions según policy.
8. **Todo cycle declarado bug.** Pause/resume puede ser válido.
9. **Graph traversal confundido con execution.** Reachable no significa action actualmente permitida.

### State

Una invalid transition lanza después de mutar `current`. ¿Qué evidencia muestra partial state corruption?

---

## 25. Dos experimentos integrados, modelos separados

### A. Relationship graph

```text
source records → validate → adjacency/edge metadata
                           → BFS/DFS/path/components
```

### B. Lifecycle FSM

```text
ordered history + rules/version → current state
                                → reachable-state audit
```

Una synthetic entity puede tener relationship edges y lifecycle state, pero no se mezclan en un mismo “mega-model”. Adjacency responde connectivity; FSM responde valid transition. Ambos son reconstructible derived state.

### Provenance

Lista cuatro artifacts: source records, transition history, adjacency index, current state. ¿Cuáles son source y cuáles derived en este experimento?

---

## 26. Testing de invariantes

Graph tests cubren empty, isolated vertex, missing start, diamond, cycle, components, dangling reference y deterministic order. FSM tests cubren cada valid pair, todas las invalid pairs, terminals, inconsistent history, rules version y replay idempotente sobre el mismo input.

**Ejemplo ejecutable — transition table exhaustiva:**

```python
from enum import Enum


class State(Enum):
    NEW = "new"
    ACTIVE = "active"
    CLOSED = "closed"


class Action(Enum):
    ACTIVATE = "activate"
    CLOSE = "close"


RULES = {
    (State.NEW, Action.ACTIVATE): State.ACTIVE,
    (State.ACTIVE, Action.CLOSE): State.CLOSED,
}

valid_count = 0
invalid_count = 0
for state in State:
    for action in Action:
        if (state, action) in RULES:
            valid_count += 1
        else:
            invalid_count += 1

assert valid_count + invalid_count == len(State) * len(Action)
assert all(source != State.CLOSED for source, _ in RULES)
print(valid_count, invalid_count)
```

Output:

```text
2 4
```

Property-based testing de PF-M9 puede generar action histories: replay acepta exactamente prefixes válidos y rechaza el primer invalid action sin append parcial.

### State

¿Por qué probar solo happy path no demuestra terminal policy?

---

## 27. Ejercicios guiados con solución razonada

### 27.1 Modelar un graph

**Input/decisión.** A follows B; B no necesariamente follows A.  
**Solución.** Directed edge `A→B`; IDs únicos.  
**Invariante/costo.** No symmetry; adjacency list O(V+E).  
**Criterio.** No añade reverse edge.  
**Variación.** Cambia a “is sibling of”.

### 27.2 Adjacency list

**Input.** `A→B,A→C,B→C`.  
**Solución.** `{"A":{"B","C"},"B":{"C"},"C":set()}`.  
**Invariante.** Todos endpoints existen.  
**Criterio.** C aparece aunque outdegree cero.  
**Variación.** Calcula indegrees.

### 27.3 Directed vs undirected

**Input.** `A—B`.  
**Solución.** Undirected adjacency almacena B en A y A en B.  
**Costo.** Dos entries por logical edge, O(E).  
**Criterio.** Validator comprueba symmetry.  
**Variación.** Self-loop policy.

### 27.4 Add edge con invariants

**Input.** Missing B.  
**Solución.** Rechaza antes de mutation; no auto-create bajo policy.  
**Criterio.** Graph queda idéntico tras error.  
**Variación.** Duplicate edge.

### 27.5 BFS

**Input.** Diamond de sección 7.  
**Solución.** Queue trace `A; B,C; C,D; D`; order `A,B,C,D`.  
**Invariante/costo.** Mark-on-discovery, O(V+E).  
**Criterio.** D se encola una vez.  
**Variación.** Reverse neighbor order.

### 27.6 DFS recursivo

**Input.** Mismo graph.  
**Solución.** Sorted policy `A,B,D,C`.  
**Invariante/costo.** Visited antes de expandir; O(V+E), stack depth variable.  
**Criterio.** Cycle termina.  
**Variación.** Vertex aislado.

### 27.7 DFS iterativo

**Input.** Mismo graph.  
**Solución.** Push reverse-sorted para retirar menor primero.  
**Invariante/costo.** Stack=pending; visited evita reprocess.  
**Criterio.** Set reachable coincide con recursive.  
**Variación.** Compara orders, no solo sets.

### 27.8 Reachability

**Input.** `A→B`, X isolated.  
**Solución.** A reaches B, no X; X no reaches A.  
**Costo.** Early exit posible, worst O(V+E).  
**Criterio.** Respeta direction.  
**Variación.** `start==target`.

### 27.9 Reconstruct shortest path

**Input.** `A→B→D` y `A→C→D`.  
**Solución.** BFS parent policy sorted elige `A,B,D`; length 2.  
**Invariante.** Primer discovery fija minimum edge distance.  
**Criterio.** No promete minimum weight.  
**Variación.** Missing path.

### 27.10 Components

**Input.** `{A—B},{C},{D—E}`.  
**Solución.** Tres components como sección 10.  
**Costo.** Global visited mantiene O(V+E).  
**Criterio.** Solo undirected semantics.  
**Variación.** Agrega B—D.

### 27.11 Detect cycle

**Input.** `A→B→C→A`.  
**Solución.** DFS encuentra edge C→A hacia visiting.  
**Costo.** O(V+E).  
**Criterio.** Edge hacia completed no se marca.  
**Variación.** Undirected parent tracking.

### 27.12 State Enum

**Input.** PENDING/VALIDATING/READY/IMPORTED/REJECTED.  
**Solución.** `Enum` de sección 17.  
**Invariante.** Un current state, no booleans incompatibles.  
**Criterio.** Nombres no definen transitions.  
**Variación.** Añade state unreachable.

### 27.13 Transition table

**Input.** Lifecycle sintético.  
**Solución.** `dict[(state,action)]→state`.  
**Costo.** Expected O(1) lookup.  
**Criterio.** Cada pair tiene máximo un target.  
**Variación.** Enumera terminals.

### 27.14 Validate transition

**Input.** PENDING+START.  
**Solución.** VALIDATING sin effects.  
**Invariante.** Pair existe antes de return.  
**Criterio.** Pure function.  
**Variación.** VALIDATING+REJECT.

### 27.15 Reject invalid transition

**Input.** PENDING+IMPORT.  
**Solución.** `ValueError`; state/history intactos.  
**Criterio.** Validation precede mutation.  
**Variación.** Action desde terminal.

### 27.16 Transition history

**Input.** START,ACCEPT,IMPORT.  
**Solución.** Tres immutable records encadenados.  
**Invariante.** `to[i]==from[i+1]`, version soportada.  
**Criterio.** Ordered append-only list.  
**Variación.** Corrompe middle record.

### 27.17 Replay

**Input.** Initial PENDING + history anterior.  
**Solución.** IMPORTED; dos replays iguales.  
**Costo.** O(T) transitions.  
**Criterio.** Revalida table/version.  
**Variación.** Empty history.

### 27.18 Reachable states

**Input.** Transition graph con ORPHAN.  
**Solución.** ORPHAN unreachable desde PENDING.  
**Costo.** O(S+T).  
**Criterio.** Initial explícito.  
**Variación.** Retry cycle válido.

### 27.19 Unreachable-state audit

**Input.** Enum states menos reachable set.  
**Solución.** Diferencia de sets.  
**Invariante.** Graph incluye terminals con empty adjacency.  
**Criterio.** No borra state automáticamente.  
**Variación.** Multiple initials.

### 27.20 Benchmark representation

**Input.** Mismos V/E y query pairs para list/matrix.  
**Solución.** Usa sección 14, repite y reporta logical slots/times.  
**Criterio.** Sparse/dense separados; output validated.  
**Variación.** Neighbor iteration benchmark.

---

## 28. Ejercicios independientes

1. Etiqueta vertices/edges/path/cycle en un graph ASCII.
2. Modela directed y undirected variants del mismo dominio.
3. Añade weighted metadata sin implementar shortest path.
4. Define self-loop/duplicate policy.
5. Implementa/valida adjacency list simple.
6. Representa el mismo graph en matrix.
7. Calcula degree/indegree/outdegree.
8. Compara space con V/E concretos.
9. Traza BFS con queue/visited.
10. Implementa BFS con missing-start contract.
11. Reconstruye shortest unweighted path.
12. Implementa DFS recursivo con depth risk documentado.
13. Implementa DFS iterativo con deterministic policy.
14. Compara reachable sets y traversal orders.
15. Encuentra components undirected.
16. Detecta cycle undirected con parent.
17. Detecta cycle directed con visiting/completed.
18. Modela circular imports como dependency graph.
19. Implementa remove vertex limpiando references.
20. Conserva typed edge metadata/provenance.
21. Demuestra set order no contractual.
22. Mide cost de sorting neighbors.
23. Construye FSM con Enum/table.
24. Enumera todas valid/invalid pairs.
25. Rechaza invalid antes de mutation.
26. Define terminal policy explícita.
27. Genera append-only records.
28. Replay y valida chain/version.
29. Detecta state unreachable.
30. Modela un cycle FSM permitido.
31. Reproduce replay desde order incorrecto.
32. Separa transition logic de I/O.
33. Construye relationship graph derived/rebuildable.
34. Distingue source evidence de edge.
35. Benchmarkea sparse/dense representations.
36. Documenta Complexity Budget con V/E.
37. Explica cuándo no adoptar graph database.
38. Integra entity graph y lifecycle sin mezclarlos.

---

## 29. Preguntas conceptuales

1. ¿Qué modela graph que tree no?
2. ¿Qué diferencia directed/undirected?
3. ¿Qué aporta weight y qué no resuelve BFS?
4. ¿Cuándo list supera matrix?
5. ¿Por qué matrix usa O(V²)?
6. ¿Qué significan degree/indegree/outdegree?
7. ¿Qué diferencia neighbor/reachable?
8. ¿Por qué BFS usa queue?
9. ¿Por qué DFS usa stack/call stack?
10. ¿Por qué visited es central?
11. ¿Por qué mark-on-discovery evita duplicates?
12. ¿Por qué BFS da minimum edges?
13. ¿Qué significa O(V+E)?
14. ¿Cómo neighbor order cambia output?
15. ¿Por qué components aquí son undirected?
16. ¿Cuándo cycle es válido/bug?
17. ¿Cómo tree es caso especial?
18. ¿Qué invariant rompe dangling edge?
19. ¿Por qué duplicate adjacency no equivale a duplicate evidence?
20. ¿Qué hace explícita transition table?
21. ¿Cómo FSM evita booleans incompatibles?
22. ¿Qué significa deterministic FSM aquí?
23. ¿Qué define terminal state?
24. ¿Por qué invalid se valida primero?
25. ¿Qué diferencia traversal de FSM execution?
26. ¿Qué significa deterministic replay?
27. ¿Por qué history puede ser source y current derived?
28. ¿Qué riesgo introduce rules drift?
29. ¿Cómo detectas unreachable state?
30. ¿Por qué current graph no reemplaza evidence?
31. ¿Cómo rebuild protege source authority?
32. ¿Qué separa graph/FSM en la integración?

---

## 30. Mini challenge — Relationship graph y lifecycle replayable

### Objetivo

Construye dos modelos separados, in-memory y sintéticos, resolubles solo con PF + CS-M1–CS-M6.

### A. Graph

1. Define stable entity IDs y source relationship records con relation/provenance.
2. Elige directed/undirected por contrato.
3. Construye derived adjacency y valida endpoints, duplicates, self-loops y symmetry aplicable.
4. Implementa deterministic BFS y DFS con visited.
5. Implementa reachability y shortest unweighted path con parent map.
6. Encuentra components si tu graph es undirected.
7. Detecta/rechaza cycle según una policy explícita.
8. Documenta V, E, O(V+E) y space.

### B. State machine

9. Define `Enum` states/actions y transition table.
10. Declara terminal states por policy.
11. Implementa pure `next_state` y rechaza invalid pairs.
12. Append transition record solo después de validation.
13. Replay ordered history y valida chain/rules version.
14. Proyecta transition graph y detecta unreachable states.
15. Prueba un cycle permitido o explica por qué ninguno lo es.

### C. Integration y evidence discipline

16. Una synthetic entity puede tener edges y lifecycle, pero APIs/data quedan separados.
17. Source records e history se preservan; adjacency/current son derived/rebuildable.
18. Una edge typed siempre conserva provenance.
19. Replay del mismo ordered input produce mismo graph/state.

### Comprobaciones contractuales

**Continuación — adapta nombres:**

```python
source_snapshot = tuple(source_records)
graph_a = build_graph(source_records)
graph_b = build_graph(source_records)

assert validate_graph(graph_a)
assert graph_a == graph_b
assert bfs(graph_a, start) == bfs(graph_b, start)
assert set(bfs(graph_a, start)) == set(dfs(graph_a, start))
assert tuple(source_records) == source_snapshot

history_snapshot = tuple(history)
state_a = replay(initial_state, history, rules_version=1)
state_b = replay(initial_state, history, rules_version=1)
assert state_a == state_b
assert tuple(history) == history_snapshot

try:
    apply_invalid_without_append(current_state, invalid_action, history)
except ValueError:
    pass
else:
    raise AssertionError("invalid transition accepted")
assert tuple(history) == history_snapshot
```

### Failure cases

Incluye: cycle sin visited con step limit; set order usado como contrato; asymmetric undirected adjacency; dangling vertex; BFS weighted mal interpretado; invalid transition parcial; history desordenado; unsupported rules version; derived graph tratado como source.

### Criterio de aprobación

- invariantes/semántica directed quedan explícitas;
- traversals terminan ante cycles y son deterministas bajo policy;
- shortest path se limita a edge count;
- FSM rechaza antes de mutation;
- replay conserva ordered history/version;
- source/provenance no se pierden;
- complejidad usa V/E/T definidos;
- no aparecen weighted shortest path, SCC/topological avanzado, graph DB, OS, concurrency, networking, backend ni AI.

---

## 31. Resumen

- Graph representa vertices y typed/directed/undirected edges; identity no es edge semantics.
- Adjacency list usa O(V+E); matrix, O(V²).
- Visited evita cycles y duplicate work.
- BFS usa queue y minimiza edges en unweighted graph; DFS usa stack/recursion.
- Con adjacency list, ambos cuestan O(V+E) y O(V) auxiliary worst case.
- Deterministic traversal requiere neighbor-order policy, no set order incidental.
- Components aquí son undirected; directed SCC queda fuera.
- Graph invariants incluyen endpoints, symmetry, duplicate/self-loop y metadata policies.
- FSM explicita state + action → next state mediante table.
- Invalid transition se rechaza antes de mutation/effects.
- Terminal/cycle semantics dependen de lifecycle policy.
- Ordered history + stable rules/version permite deterministic replay.
- Relationship graph/current state son derived; source evidence/history conservan autoridad.

---

## 32. Checklist de dominio

- [ ] Puedo modelar directed/undirected/weighted-intro edges.
- [ ] Puedo distinguir identity, relation y provenance.
- [ ] Puedo validar graph invariants.
- [ ] Puedo elegir adjacency list/matrix por V/E/workload.
- [ ] Puedo calcular degree/indegree/outdegree.
- [ ] Puedo explicar path/cycle/component/reachability.
- [ ] Puedo implementar BFS con deque/visited.
- [ ] Puedo reconstruir shortest unweighted path.
- [ ] Puedo implementar DFS recursivo/iterativo.
- [ ] Puedo derivar O(V+E) y O(V) space.
- [ ] Puedo asegurar deterministic neighbor order.
- [ ] Puedo detectar directed cycle sin colgarme.
- [ ] Puedo preservar typed edge provenance.
- [ ] Puedo definir Enum/table de transitions.
- [ ] Puedo rechazar invalid antes de mutation.
- [ ] Puedo separar pure transition de I/O.
- [ ] Puedo definir terminal/cycle policy.
- [ ] Puedo append/replay ordered history.
- [ ] Puedo validar rules version y chain.
- [ ] Puedo encontrar unreachable states.
- [ ] Puedo distinguir traversal de FSM execution.
- [ ] Puedo reconstruir derived graph/current state.
- [ ] Puedo completar challenge con PF + CS-M1–CS-M6.

---

## 33. Preparación para labs y EIDOLON 0.0b

- **CS-L10 — Evidence graph:** adjacency list typed, `supports/refutes` edges, BFS path y provenance obligatorio.
- **CS-L11 — Memory state machine:** `proposed/approved/rejected/superseded/purged` con property tests.
- **CS-MP3 — Relationship & Evidence Graph:** graph dirigido de people/events/claims/hypotheses, typed edges y hypothesis state machine.

| Concepto | Secciones | Evidencia | Lab/proyecto |
|---|---:|---|---|
| Adjacency/invariants | 1–6, 12 | Guiados 1–4 | CS-L10 |
| BFS/DFS/path | 7–10 | Guiados 5–10 | CS-L10 |
| Cycles | 11, 23 | Guiado 11 | CS-L10, CS-MP3 |
| Transition table | 16–18 | Guiados 12–15 | CS-L11 |
| History/replay | 19 | Guiados 16–17 | CS-L11 |
| Reachable states | 20–21 | Guiados 18–19 | CS-L11, CS-MP3 |
| Benchmark V/E | 13–14 | Guiado 20 | EIDOLON 0.0b |

CS-M6 aporta relationship/dependency graphs, explicit invariants, cycle awareness, state transitions, replay y derived indexes al build 0.0b. No implementa el build completo ni graph persistence.

---

## 34. Recursos de ampliación

El módulo es autocontenido. Usa los recursos canónicos de [CS.11 Recursos recomendados](../../02_curriculum/02_computer_science_foundations.md#cs11-recursos-recomendados) para profundizar y la documentación oficial de `collections.deque`, `Enum` y dataclasses para verificar APIs.

---

## 35. Límite explícito del módulo

CS-M6 termina en graph models/invariants, adjacency list/matrix, BFS/DFS, unweighted path, components, cycle detection aplicada, typed/provenance edges, FSM tables, terminal/cycle policy, ordered history y deterministic replay.

No desarrolla Dijkstra, Bellman-Ford, Floyd-Warshall, A*, MST, SCC, topological sorting avanzado, graph databases, automata language recognition, OS/filesystem, processes/threads, networking, backend ni AI.

El siguiente paso permitido es revisar CS-M6 como `review candidate`. **No se crea ni se desarrolla CS-M7 en esta entrega.**
