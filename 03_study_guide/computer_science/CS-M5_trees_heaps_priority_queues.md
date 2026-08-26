# CS-M5 — Árboles, heaps y priority queues

**Track:** Computer Science Foundations  
**Competencias:** D2.1; soporte D2.2, D2.3  
**Fase:** P0  
**Nivel objetivo:** Aplicado  
**Prerequisites:** PF-M1, PF-M3, PF-M5, PF-M9, CS-M1, CS-M2, CS-M3, CS-M4  
**Build:** EIDOLON 0.0b  
**Curriculum source:** [CS-M5](../../02_curriculum/02_computer_science_foundations.md#cs-m5--trees-heaps-y-top-k)  
**Status:** review candidate

Una secuencia comunica “antes/después”; un `dict`, “key → value”. Ninguna expresa por sí sola parent/child ni mantiene accesible el candidato prioritario mientras llegan nuevos elementos.

CS-M5 sigue esta cadena:

```text
problema
↓
jerarquía o prioridad
↓
invariante estructural
↓
representación y operaciones
↓
altura / costo
↓
medición
↓
tradeoff para EIDOLON
```

Los trees y heaps manuales son modelos educativos. No se convierten en arquitectura canónica ni reemplazan `dict`, `sorted()`, `bisect`, `heapq` o futuros índices persistentes sin un workload medido. No se desarrollan general graphs, state machines, database indexes, OS, concurrency, networking, backend ni AI.

## Resultados de aprendizaje

Al terminar podrás:

- identificar root, parent, child, sibling, leaf, ancestor, descendant y subtree en un diagrama;
- calcular depth y height con una convención de edges explícita;
- explicar por qué un tree es recursivo sin exigir implementación recursiva;
- implementar preorder, inorder y postorder y justificar su semántica;
- sustituir call stack por stack explícito y recorrer por levels con `deque` en un tree;
- derivar traversal O(n) y space O(h) sin suponer balance;
- distinguir binary tree de binary search tree (BST);
- implementar search/insert O(h) con política de duplicates explícita;
- validar la invariante global de un BST;
- demostrar degradación skewed a O(n) y evitar prometer O(log n) sin balance;
- distinguir BST, heap, sorted list e índice `dict` por sus operaciones;
- derivar la representación de complete binary tree en una list;
- trazar y ejecutar sift up, sift down y heapify;
- implementar un min-heap educativo con invariant comprobable;
- usar `heapq` como min-heap y las APIs max-heap del baseline Python 3.14;
- distinguir priority queue ADT de heap implementation y queue FIFO;
- diseñar tie-breakers que eviten comparaciones incompatibles y orden no determinista;
- mantener top-k en O(n log k) time y O(k) space;
- comparar top-k con full sort incluyendo `n`, `k`, output order y constantes;
- preservar source data mientras trees/heaps actúan como derived synthetic views.

## Cómo estudiar este módulo

1. Dibuja la estructura y nombra su invariante antes de programar.
2. Declara si height/depth cuentan edges o nodes; aquí cuentan edges.
3. Traza empty, one, balanced-ish y skewed.
4. En BST, revisa toda la subtree, no solo children inmediatos.
5. En heap, comprueba parent/children después de cada mutation.
6. Separa el ADT de la representación concreta.
7. Formula la hipótesis de benchmark antes de medir.

### Convenciones

- **Ejemplo ejecutable:** bloque autónomo con output estable o `assert`.
- **Modelo educativo:** implementación manual para observar invariantes, no recomendación de producción.
- **Failure case ejecutable:** reproduce un fallo y explica causa/corrección.
- **Benchmark ejecutable:** los tiempos son ambientales; solo se fijan propiedades estables.
- **Fragmento:** requiere contexto omitido.
- **Continuación:** depende explícitamente del bloque inmediatamente anterior.

Baseline: Python 3.14 y standard library.

---

## 1. El problema jerárquico

Una jerarquía impone una relación distinta de la secuencia:

```text
                  journal
                 /       \
           personal       work
           /      \          \
       health    travel      projects
```

Usaremos esta convención: la **depth** de un node es la cantidad de edges desde root; la **height** de un node es la mayor cantidad de edges desde ese node hasta una leaf. Por tanto:

- `journal` es root, depth 0 y height 2;
- `personal` y `work` son children de root y siblings entre sí;
- `health`, `travel` y `projects` son leaves, height 0;
- `personal` es parent de `health` y `travel`;
- `personal` con sus descendants forma una subtree;
- `journal` es ancestor de `travel`; `travel` es descendant de `journal`.

Un tree no es “cualquier dato conectado”. Bajo el modelo estricto usado aquí:

- existe un root;
- cada node salvo root tiene un solo parent;
- hay un camino único desde root a cada node;
- no existen cycles.

General graphs permiten múltiples caminos, arbitrary edges y cycles; CS-M6 los desarrolla.

### Dibuja

Añade `urgent` debajo de `projects`. Calcula depth de `urgent` y las nuevas heights de `work` y `journal`.

### Detecta la confusión

Si una categoría aparece como child de dos parents, ¿sigue siendo el tree estricto de este módulo? ¿Qué premisa se rompió?

---

## 2. Tree como estructura recursiva y referencias

Un tree puede definirse recursivamente:

```text
tree = node + zero or more child subtrees
```

En un binary tree cada node tiene, como máximo, `left` y `right`. “Binary” describe cantidad de children, no orden de keys.

**Ejemplo ejecutable — representación mínima:**

```python
from __future__ import annotations

from dataclasses import dataclass


@dataclass
class TreeNode:
    value: int
    left: TreeNode | None = None
    right: TreeNode | None = None


root = TreeNode(
    8,
    left=TreeNode(3, TreeNode(1), TreeNode(6)),
    right=TreeNode(10, right=TreeNode(14)),
)

print(root.value)
print(root.left.right.value if root.left and root.left.right else None)
```

Output:

```text
8
6
```

Los fields contienen referencias. Dos paths pueden apuntar accidentalmente al mismo node, o un child volver a un ancestor. Eso viola el contrato de tree aunque el type hint siga siendo correcto.

### Predice

Si `alias = root.left`, ¿qué observa `root.left.value` después de `alias.value = 99`? Conecta con identidad y mutabilidad de PF-M1.

### Invariante

¿Qué restricciones adicionales al tipo `TreeNode | None` hacen que el objeto conectado sea realmente un tree?

---

## 3. Tres órdenes de depth-first traversal

Un tree no tiene un único orden lineal natural. El orden depende de cuándo se procesa el node respecto de sus subtrees.

Para este árbol:

```text
      A
     / \
    B   C
   / \
  D   E
```

los tres depth-first traversals son:

```text
preorder:  node, left, right → A B D E C
inorder:   left, node, right → D B E A C
postorder: left, right, node → D E B C A
```

- **Preorder** procesa parent antes de children: inspección, copia estructural o serialización conceptual.
- **Inorder** coloca node entre subtrees; solo produce keys ordenadas si el tree cumple la invariante BST.
- **Postorder** procesa children antes de parent: aggregates, evaluación o cleanup conceptual.

**Ejemplo ejecutable — tres variantes recursivas:**

```python
from __future__ import annotations

from dataclasses import dataclass


@dataclass
class TreeNode:
    value: str
    left: TreeNode | None = None
    right: TreeNode | None = None


def preorder(node: TreeNode | None) -> list[str]:
    output: list[str] = []

    def visit(current: TreeNode | None) -> None:
        if current is None:
            return
        output.append(current.value)
        visit(current.left)
        visit(current.right)

    visit(node)
    return output


def inorder(node: TreeNode | None) -> list[str]:
    output: list[str] = []

    def visit(current: TreeNode | None) -> None:
        if current is None:
            return
        visit(current.left)
        output.append(current.value)
        visit(current.right)

    visit(node)
    return output


def postorder(node: TreeNode | None) -> list[str]:
    output: list[str] = []

    def visit(current: TreeNode | None) -> None:
        if current is None:
            return
        visit(current.left)
        visit(current.right)
        output.append(current.value)

    visit(node)
    return output


tree = TreeNode("A", TreeNode("B", TreeNode("D"), TreeNode("E")), TreeNode("C"))
print(" ".join(preorder(tree)))
print(" ".join(inorder(tree)))
print(" ".join(postorder(tree)))
print(preorder(None))
```

Output:

```text
A B D E C
D B E A C
D E B C A
[]
```

En cada función el base case es `None`. El recursive case conserva el mismo trabajo pero cambia dónde aparece `[node.value]`.

### Traza

Marca cuándo se agrega `B` en cada función. No memorices la lista: deriva desde la posición de “node”.

### Modifica

Cambia los returns para contar nodes. ¿Qué traversal differences dejan de ser visibles cuando la operación es solo sumar uno?

---

## 4. Traversal iterativo, por levels y costo

El call stack implícito puede sustituirse por un stack explícito de CS-M3.

**Ejemplo ejecutable — preorder iterativo:**

```python
from __future__ import annotations

from dataclasses import dataclass


@dataclass
class TreeNode:
    value: str
    left: TreeNode | None = None
    right: TreeNode | None = None


def preorder_iterative(root: TreeNode | None) -> list[str]:
    if root is None:
        return []

    output: list[str] = []
    stack = [root]
    while stack:
        node = stack.pop()
        output.append(node.value)
        if node.right is not None:
            stack.append(node.right)
        if node.left is not None:
            stack.append(node.left)
    return output


tree = TreeNode("A", TreeNode("B", TreeNode("D"), TreeNode("E")), TreeNode("C"))
print(preorder_iterative(tree))
```

Output:

```text
['A', 'B', 'D', 'E', 'C']
```

Se agrega right antes de left porque `list.pop()` retira lo último. El resultado sigue procesando left primero.

Una queue permite **level-order traversal** sobre un tree. Esta es la intuición breadth-first necesaria para CS-L09; general BFS sobre graphs, con visited/path reconstruction, pertenece a CS-M6.

**Ejemplo ejecutable — levels con `deque`:**

```python
from __future__ import annotations

from collections import deque
from dataclasses import dataclass


@dataclass
class TreeNode:
    value: str
    left: TreeNode | None = None
    right: TreeNode | None = None


def level_order(root: TreeNode | None) -> list[str]:
    if root is None:
        return []
    pending = deque([root])
    output: list[str] = []
    while pending:
        node = pending.popleft()
        output.append(node.value)
        if node.left is not None:
            pending.append(node.left)
        if node.right is not None:
            pending.append(node.right)
    return output


tree = TreeNode("A", TreeNode("B", TreeNode("D"), TreeNode("E")), TreeNode("C"))
print(level_order(tree))
```

Output:

```text
['A', 'B', 'C', 'D', 'E']
```

Si cada node se visita una vez, time es O(n). Space depende de shape:

- recursion o explicit DFS stack: O(h) en el modelo ideal, donde `h` es height; puede ser O(n) en tree skewed;
- level-order queue: O(w), donde `w` es el máximo width; puede ser O(n).

Las versiones recursivas anteriores usan un accumulator: cada node agrega una vez. Concatenar listas de subresultados en cada return podría añadir copias y distorsionar el costo del patrón.

### Altura

¿Qué estructura auxiliar crece más en un tree ancho de dos levels? ¿Y en una cadena de n nodes?

### Traza

Escribe el stack después de procesar `A` y `B` en preorder iterativo.

---

## 5. Height, shape y riesgo de depth

Con la convención de edges, empty tree tiene height `-1` y una leaf tiene height `0`.

**Ejemplo ejecutable:**

```python
from __future__ import annotations

from dataclasses import dataclass


@dataclass
class TreeNode:
    value: int
    left: TreeNode | None = None
    right: TreeNode | None = None


def height(node: TreeNode | None) -> int:
    if node is None:
        return -1
    return 1 + max(height(node.left), height(node.right))


balanced_ish = TreeNode(4, TreeNode(2, TreeNode(1), TreeNode(3)), TreeNode(6))
skewed = TreeNode(1, right=TreeNode(2, right=TreeNode(3, right=TreeNode(4))))

print(height(None))
print(height(TreeNode(9)))
print(height(balanced_ish))
print(height(skewed))
```

Output:

```text
-1
0
2
3
```

Para `n` nodes, un binary tree balanced-ish tiene height típica O(log n); una cadena completamente skewed tiene height `n - 1`, O(n). “Balanced-ish” describe shape observada o mantenida por alguna política; no es garantía automática.

Una recursion sobre tree skewed profundo puede alcanzar `RecursionError`. El traversal iterativo hace la depth explícita y suele ser más robusto; sigue usando memoria para pendientes.

### Predice

¿Qué height produce insertar children siempre por `right`? Exprésala en función de `n`.

### Detecta la confusión

En el diagrama de la sección 1, `travel` tiene depth 2 y height 0. Explica por qué esos números miden direcciones distintas.

---

## 6. Binary tree no significa BST

Este es un binary tree válido:

```text
      10
     /  \
   20    5
```

Tiene como máximo dos children por node, pero viola el orden de un binary search tree. Inorder produce `[20, 10, 5]`, no una secuencia ordenada.

Un **binary search tree (BST)** añade una invariante global. Con la política sin duplicates usada aquí, para cada node:

```text
todas las keys de left subtree  < node.key
todas las keys de right subtree > node.key
```

“Global” importa: comprobar solo children inmediatos puede aceptar un descendant mal ubicado.

Binary search de CS-M4 es un algoritmo sobre una secuencia ordenada e indexable. BST es una estructura enlazada que mantiene una relación de orden. Comparten la idea de descartar por comparación, no representación ni preconditions idénticas.

### Invariante

¿Por qué `8` como right child de `5`, dentro de la left subtree de root `7`, viola la invariante aunque `8 > 5`?

---

## 7. BST educativo: contrato y duplicates

Para IDs estables, insertar un duplicate silenciosamente perdería identidad o escondería un conflicto. El modelo educativo rechazará duplicate key con `ValueError`.

**Modelo educativo ejecutable:**

```python
from __future__ import annotations

from dataclasses import dataclass


@dataclass
class BSTNode:
    key: int
    left: BSTNode | None = None
    right: BSTNode | None = None


class EducationalBST:
    def __init__(self) -> None:
        self.root: BSTNode | None = None

    def insert(self, key: int) -> None:
        if self.root is None:
            self.root = BSTNode(key)
            return

        current = self.root
        while True:
            if key == current.key:
                raise ValueError(f"duplicate key: {key}")
            if key < current.key:
                if current.left is None:
                    current.left = BSTNode(key)
                    return
                current = current.left
            else:
                if current.right is None:
                    current.right = BSTNode(key)
                    return
                current = current.right

    def search(self, target: int) -> BSTNode | None:
        current = self.root
        while current is not None:
            if target == current.key:
                return current
            current = current.left if target < current.key else current.right
        return None

    def inorder(self) -> list[int]:
        output: list[int] = []

        def visit(node: BSTNode | None) -> None:
            if node is None:
                return
            visit(node.left)
            output.append(node.key)
            visit(node.right)

        visit(self.root)
        return output


tree = EducationalBST()
for key in (8, 3, 10, 1, 6, 14):
    tree.insert(key)

print(tree.inorder())
print(tree.search(6).key if tree.search(6) else None)
print(tree.search(99))

try:
    tree.insert(6)
except ValueError as error:
    print(error)
```

Output:

```text
[1, 3, 6, 8, 10, 14]
6
None
duplicate key: 6
```

Search e insert siguen un solo path. Su costo es O(h), no O(log n) incondicional.

### Traza

Traza search de `6` y `7`. ¿Qué comparación elimina cada subtree?

### Modifica

Implementa `insert` recursivo como ejercicio. Mantén exactamente la misma política de duplicates.

---

## 8. Validar la invariante global del BST

Cada node hereda bounds de todos sus ancestors. Esta validación usa intervalos abiertos porque duplicates están prohibidos.

**Ejemplo ejecutable:**

```python
from __future__ import annotations

from dataclasses import dataclass


@dataclass
class BSTNode:
    key: int
    left: BSTNode | None = None
    right: BSTNode | None = None


def is_valid_bst(
    node: BSTNode | None,
    lower: int | None = None,
    upper: int | None = None,
) -> bool:
    if node is None:
        return True
    if lower is not None and node.key <= lower:
        return False
    if upper is not None and node.key >= upper:
        return False
    return is_valid_bst(node.left, lower, node.key) and is_valid_bst(
        node.right, node.key, upper
    )


valid = BSTNode(7, BSTNode(4, BSTNode(2), BSTNode(6)), BSTNode(9))
invalid = BSTNode(7, BSTNode(4, right=BSTNode(8)), BSTNode(9))

print(is_valid_bst(valid))
print(is_valid_bst(invalid))
```

Output:

```text
True
False
```

El `8` es mayor que su parent `4`, pero sigue dentro de la left subtree de `7` y debe ser `< 7`. Un check local lo habría aceptado.

### Invariante

¿Qué bounds recibe el node `6` del árbol válido? ¿Cuáles provienen de parent y ancestor?

---

## 9. Insertion order, skew y costo O(h)

Insertar `1, 2, 3, 4, 5` en un BST simple produce:

```text
1
 \
  2
   \
    3
     \
      4
       \
        5
```

Height es 4 y search de `5` visita cinco nodes. Con `n` keys crecientes, height es `n - 1`; search/insert worst case son O(n).

Un insertion order como `4, 2, 6, 1, 3, 5, 7` produce shape balanced-ish para esas keys y paths O(log n). No convierte el BST simple en self-balancing: otra secuencia lo degrada.

**Ejemplo ejecutable — pasos observables:**

```python
def bst_path(keys: list[int], target: int) -> list[int]:
    tree: dict[int, list[int | None]] = {}
    root: int | None = None

    for key in keys:
        if root is None:
            root = key
            tree[key] = [None, None]
            continue
        current = root
        while True:
            side = 0 if key < current else 1
            child = tree[current][side]
            if child is None:
                tree[current][side] = key
                tree[key] = [None, None]
                break
            current = child

    path: list[int] = []
    current = root
    while current is not None:
        path.append(current)
        if target == current:
            break
        current = tree[current][0 if target < current else 1]
    return path


print(bst_path([4, 2, 6, 1, 3, 5, 7], 7))
print(bst_path([1, 2, 3, 4, 5, 6, 7], 7))
```

Output:

```text
[4, 6, 7]
[1, 2, 3, 4, 5, 6, 7]
```

AVL y Red-Black trees mantienen altura mediante invariantes y reorganizaciones. Solo se mencionan: implementarlas y probarlas correctamente excede CS-M5.

### Predice

¿Qué shape produce insertion order descendente? ¿Cambia la clase worst-case?

### Altura

Explica por qué el costo es O(h) primero y solo O(log n) bajo una garantía/supuesto de altura.

---

## 10. Por qué Python no necesita un BST general como default

Python ofrece herramientas distintas para workloads distintos:

- `dict`: equality lookup esperado O(1);
- `list` + `bisect`: boundaries sobre batch ordenado; insertion O(n);
- `sorted()`: orden total batch;
- `heapq`: prioridad root y top-k;
- estructuras especializadas externas o índices futuros: solo con necesidad demostrada.

Esto no significa “BST no sirve”. Significa que una estructura enlazada ordered, con policy de balance, duplicates y mutation, no tiene un único contrato general ideal para todos los programas Python.

No uses el BST educativo como “database index”. B-trees, transactions, persistence y database query planners pertenecen a otro track.

### Workload

Para lookup exacto repetido por `event_id`, ¿qué ofrece un `dict` que el BST simple no garantiza?

---

## 11. Tree EIDOLON controlado y frontera con graph

Una taxonomía sintética permite practicar sin decidir arquitectura:

```text
memory
├── event
│   ├── observation
│   └── activity
└── claim
    ├── supported
    └── disputed
```

Esta jerarquía es derived/synthetic. No convierte interpretaciones en hechos ni reemplaza provenance. Se adopta solo si queries parent/child y traversal justifican su costo.

Si `supported` necesita pertenecer a múltiples parents, aparecen multiple paths o cross-links. Esa estructura deja de ser el tree estricto; CS-M6 enseñará graph modeling, visited sets y edge semantics.

### Failure case: referencia repetida o cycle

Un traversal que asume tree podría visitar dos veces un shared child o no terminar con un cycle. Esta comprobación defensiva rechaza cualquier identidad repetida; no intenta interpretar un graph.

**Failure case ejecutable:**

```python
from __future__ import annotations

from dataclasses import dataclass


@dataclass
class TreeNode:
    value: str
    children: list[TreeNode]


def preorder_checked(root: TreeNode) -> list[str]:
    output: list[str] = []
    pending = [root]
    seen: set[int] = set()
    while pending:
        node = pending.pop()
        identity = id(node)
        if identity in seen:
            raise ValueError("input is not a strict tree")
        seen.add(identity)
        output.append(node.value)
        pending.extend(reversed(node.children))
    return output


root = TreeNode("root", [])
child = TreeNode("child", [])
root.children.append(child)
child.children.append(root)

try:
    preorder_checked(root)
except ValueError as error:
    print(error)
```

Output:

```text
input is not a strict tree
```

### Decide

¿Rechazarías un shared child, lo copiarías o cambiarías el modelo a graph? La estructura no decide la semántica por ti.

---

## 12. De orden total a prioridad

Ahora cambia el problema:

```text
tengo n candidatos
no necesito ordenar todos
necesito el menor, el mayor o los k mejores
```

Un full sort entrega más información: posición relativa de todos los elementos. Si solo necesitas el mejor o `k << n`, ese trabajo puede ser innecesario.

Un **heap** mantiene una propiedad parcial de orden:

```text
min-heap: parent <= children
max-heap: parent >= children
```

No exige que siblings estén ordenados ni que el array completo sea monotónico.

### Workload

¿Necesitas todos los candidatos ordenados o solo acceso repetido al siguiente prioritario? Esa pregunta precede la elección.

---

## 13. Complete binary tree y representación en array

Un binary heap clásico mantiene shape de **complete binary tree**:

- todos los levels salvo quizá el último están llenos;
- el último se llena de izquierda a derecha.

```text
array:  [1, 3, 2, 7, 8, 4]
index:   0  1  2  3  4  5

            1 (0)
          /       \
      3 (1)       2 (2)
      /   \       /
   7 (3) 8 (4) 4 (5)
```

Como no hay huecos estructurales, no se necesitan referencias `left/right`. Para zero-based indexing:

```text
left(i)   = 2*i + 1
right(i)  = 2*i + 2
parent(i) = (i - 1) // 2, para i > 0
```

La fórmula sale de los indices del diagrama, no de memorización ciega.

**Ejemplo ejecutable:**

```python
def children(index: int, size: int) -> tuple[int | None, int | None]:
    left = 2 * index + 1
    right = 2 * index + 2
    return (
        left if left < size else None,
        right if right < size else None,
    )


print(children(0, 6))
print(children(2, 6))
print((5 - 1) // 2)
```

Output:

```text
(1, 2)
(5, None)
2
```

### Index math

Para index `4`, deriva parent y posibles children en un heap de size `7`.

### Heap check

¿El array del diagrama cumple min-heap property? Comprueba cada parent, no orden global.

---

## 14. Root, sift up y sift down

En min-heap, `heap[0]` es el mínimo; en max-heap, el máximo. Consultarlo cuesta O(1). Extraerlo cambia shape y requiere reparación.

### 14.1 Sift up después de push

```text
[2, 5, 4, 9, 7]
append 1
[2, 5, 4, 9, 7, 1]
             ↑ parent index 2, value 4

swap → [2, 5, 1, 9, 7, 4]
swap → [1, 5, 2, 9, 7, 4]
```

Cada swap sube un level. En height O(log n), push cuesta O(log n).

**Ejemplo ejecutable — traza de sift up:**

```python
def push_with_trace(heap: list[int], value: int) -> list[list[int]]:
    heap.append(value)
    states = [heap.copy()]
    index = len(heap) - 1
    while index > 0:
        parent = (index - 1) // 2
        if heap[parent] <= heap[index]:
            break
        heap[parent], heap[index] = heap[index], heap[parent]
        index = parent
        states.append(heap.copy())
    return states


print(push_with_trace([2, 5, 4, 9, 7], 1))
```

Output:

```text
[[2, 5, 4, 9, 7, 1], [2, 5, 1, 9, 7, 4], [1, 5, 2, 9, 7, 4]]
```

### 14.2 Sift down después de pop

Para remover root:

```text
mover last a root
↓
elegir el menor child
↓
swap mientras child < parent
```

Elegir el menor child es esencial: intercambiar con uno mayor puede dejar al otro child violando la propiedad.

**Ejemplo ejecutable — traza de sift down:**

```python
def pop_with_trace(heap: list[int]) -> tuple[int, list[list[int]]]:
    if not heap:
        raise IndexError("pop from empty heap")
    root = heap[0]
    last = heap.pop()
    states: list[list[int]] = []
    if heap:
        heap[0] = last
        states.append(heap.copy())
        index = 0
        while True:
            left = 2 * index + 1
            right = 2 * index + 2
            if left >= len(heap):
                break
            smaller = left
            if right < len(heap) and heap[right] < heap[left]:
                smaller = right
            if heap[index] <= heap[smaller]:
                break
            heap[index], heap[smaller] = heap[smaller], heap[index]
            index = smaller
            states.append(heap.copy())
    return root, states


removed, states = pop_with_trace([1, 3, 2, 7, 8, 4])
print(removed)
print(states)
```

Output:

```text
1
[[4, 3, 2, 7, 8], [2, 3, 4, 7, 8]]
```

Pop recorre como máximo height O(log n); peek no reorganiza y es O(1).

### Sift

¿Qué swaps produce insertar `0` en `[1, 3, 2, 7, 8, 4]`?

---

## 15. Min-heap educativo e invariante

La implementación siguiente demuestra push, peek, pop y las dos reparaciones. No replica internals de CPython ni reemplaza `heapq`.

**Modelo educativo ejecutable:**

```python
class EducationalMinHeap:
    def __init__(self) -> None:
        self._items: list[int] = []

    def __len__(self) -> int:
        return len(self._items)

    def peek(self) -> int:
        if not self._items:
            raise IndexError("peek from empty heap")
        return self._items[0]

    def push(self, value: int) -> None:
        self._items.append(value)
        self._sift_up(len(self._items) - 1)

    def pop(self) -> int:
        if not self._items:
            raise IndexError("pop from empty heap")
        root = self._items[0]
        last = self._items.pop()
        if self._items:
            self._items[0] = last
            self._sift_down(0)
        return root

    def _sift_up(self, index: int) -> None:
        while index > 0:
            parent = (index - 1) // 2
            if self._items[parent] <= self._items[index]:
                return
            self._items[parent], self._items[index] = (
                self._items[index],
                self._items[parent],
            )
            index = parent

    def _sift_down(self, index: int) -> None:
        while True:
            left = 2 * index + 1
            right = 2 * index + 2
            if left >= len(self._items):
                return
            smaller = left
            if right < len(self._items) and self._items[right] < self._items[left]:
                smaller = right
            if self._items[index] <= self._items[smaller]:
                return
            self._items[index], self._items[smaller] = (
                self._items[smaller],
                self._items[index],
            )
            index = smaller

    def snapshot(self) -> tuple[int, ...]:
        return tuple(self._items)


def is_min_heap(items: list[int] | tuple[int, ...]) -> bool:
    for child in range(1, len(items)):
        parent = (child - 1) // 2
        if items[parent] > items[child]:
            return False
    return True


heap = EducationalMinHeap()
for value in (5, 1, 4, 1, 3):
    heap.push(value)
    assert is_min_heap(heap.snapshot())

print(heap.peek())
removed = [heap.pop() for _ in range(len(heap))]
print(removed)
print(is_min_heap(heap.snapshot()))

try:
    heap.pop()
except IndexError as error:
    print(error)
```

Output:

```text
1
[1, 1, 3, 4, 5]
True
pop from empty heap
```

Duplicates son válidos en el heap de valores: la heap property usa `<=`. Esta política no contradice el rechazo de duplicate IDs en el BST; son contratos distintos.

### Heap check

Después de cada mutation, ¿qué pairs parent/child debe verificar `is_min_heap`?

### Modifica

Añade `heapify(values)` a la clase usando sift down desde el último parent. No lo implementes como n llamadas a `push` si quieres estudiar el algoritmo lineal.

---

## 16. Heap no es BST ni sorted list

El array:

```text
[1, 3, 2, 7, 8, 4]
```

es min-heap válido y no está globalmente sorted: `3 > 2`. La propiedad solo conecta cada parent con children.

| Pregunta | Heap | BST simple | Sorted list | `dict` |
|---|---|---|---|---|
| Consultar mínimo | O(1) en min-heap | O(h) siguiendo left | O(1) en index 0 | No representa orden |
| Push/insert | O(log n) heap | O(h) | O(n) por desplazamiento | Esperado O(1) |
| Arbitrary equality lookup | O(n) | O(h) por key | O(log n) binary search | Esperado O(1) |
| Todos en orden | Pop repetido O(n log n) | Inorder O(n) si válido | Ya disponibles | No |
| Shape | Complete | Depende de insertion/balance | Contiguous sequence | Hash table |

Heap tampoco ofrece inorder sorted traversal. Su árbol conceptual existe para mantener priority root, no ordered search arbitrario.

### Workload

¿Qué estructura usarías para lookup exacto por `event_id`? ¿Por qué min access O(1) no responde esa query?

---

## 17. `heapq`, `heapify` y max-heap en Python 3.14

`heapq` implementa heaps sobre una `list`. Sus funciones mutan esa list; conserva ownership claro.

**Ejemplo ejecutable — min-heap:**

```python
from heapq import heapify, heappop, heappush


values = [7, 2, 5, 1, 9]
heapify(values)
assert values[0] == 1
heappush(values, 3)
ordered = [heappop(values) for _ in range(len(values))]
print(ordered)
```

Output:

```text
[1, 2, 3, 5, 7, 9]
```

No fijamos el array intermedio como contrato; distintas representaciones pueden satisfacer la heap property. Solo root y secuencia de pops tienen semántica relevante.

### 17.1 `heapify` es O(n)

El algoritmo adecuado repara subtrees desde los últimos parents hacia root. La intuición:

- muchas leaves no bajan ningún level;
- muchos nodes cercanos a leaves bajan poco;
- pocos nodes altos pueden bajar mucho.

Por eso el trabajo agregado es O(n), mejor que describirlo como “n pushes O(n log n)”. Ambos producen un heap, pero no ejecutan exactamente el mismo algoritmo ni costo.

### 17.2 Max-heap del baseline

Python 3.14 añade APIs max-heap simétricas. Este ejemplo declara esa precondition de versión:

**Ejemplo ejecutable — requiere Python 3.14+:**

```python
from heapq import heapify_max, heappop_max, heappush_max


values = [7, 2, 5, 1, 9]
heapify_max(values)
heappush_max(values, 8)
print([heappop_max(values) for _ in range(len(values))])
```

Output:

```text
[9, 8, 7, 5, 2, 1]
```

En Python anteriores, la técnica tradicional para prioridades numéricas era negar valores dentro de un min-heap. No mezcles ambas representaciones dentro del mismo heap sin un contrato explícito.

`heappushpop` y `heapreplace` combinan operaciones con semánticas distintas sobre el root; consúltalas cuando reduzcan trabajo real. No son inventario obligatorio de CS-M5.

### Predice

Después de `heapify`, ¿qué única posición puedes interpretar como mínimo sin ordenar el resto?

---

## 18. Priority queue ADT y tie-breaking

Una **priority queue** es una abstracción:

- insertar item con priority;
- inspeccionar el de mayor/menor prioridad;
- remover el siguiente por prioridad.

Un heap es una implementación común. Una queue FIFO de CS-M3 decide por arrival; priority queue decide por priority. Equal priority no implica FIFO automáticamente.

El patrón `(priority, sequence, item)` ofrece:

- priority como criterio primario;
- sequence creciente como FIFO entre ties;
- item queda fuera de la comparación mientras `(priority, sequence)` sea único.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass
from heapq import heappop, heappush


@dataclass(frozen=True)
class Job:
    job_id: str


pending: list[tuple[int, int, Job]] = []
sequence = 0
for priority, job_id in ((2, "job-B"), (1, "job-A"), (1, "job-C")):
    heappush(pending, (priority, sequence, Job(job_id)))
    sequence += 1

print([heappop(pending)[2].job_id for _ in range(len(pending))])
```

Output:

```text
['job-A', 'job-C', 'job-B']
```

Menor número significa mayor urgencia en este contrato. `job-A` sale antes que `job-C` por sequence FIFO.

### Tie

Si replay debe ser independiente del arrival no determinista, sustituye sequence de runtime por una secondary key estable de dominio. ¿Qué propiedad debe cumplir?

---

## 19. Failure cases de entries y priorities mutables

### 19.1 Equal priorities comparan objetos incompatibles

Sin sequence, tuples empatadas intentan comparar `Job`.

**Failure case ejecutable:**

```python
from dataclasses import dataclass
from heapq import heappush


@dataclass
class Job:
    job_id: str


pending: list[tuple[int, Job]] = []
heappush(pending, (1, Job("A")))
try:
    heappush(pending, (1, Job("B")))
except TypeError as error:
    print(type(error).__name__)
```

Output estable:

```text
TypeError
```

Corrección: `(priority, sequence, item)` o una comparable secondary key única.

### 19.2 Mutation in-place rompe la propiedad lógica

**Failure case ejecutable:**

```python
from heapq import heapify, heappop


entries = [[1, "A"], [2, "B"], [3, "C"]]
heapify(entries)
entries[2][0] = 0  # cambia prioridad sin reparar
print(entries[0])
print(heappop(entries))
```

Output:

```text
[1, 'A']
[1, 'A']
```

`C` debería ser nuevo mínimo, pero el heap no observa mutations arbitrarias. Prefiere immutable entries y reinserción controlada.

`heapq` tampoco ofrece decrease-key general por item arbitrario. Un patrón futuro inserta la nueva entry y marca la anterior stale, descartándola al extraer; se menciona sin construir scheduler ni lifecycle complejo.

### Invariante

¿Qué datos necesitas conservar para decidir si una entry extraída sigue vigente?

---

## 20. Top-k con heap bounded

Para `n` candidatos y `k << n`:

```text
full sort       → O(n log n), output completo
heap de size k  → O(n log k), auxiliary space O(k)
```

Mantenemos un min-heap de los mejores vistos. Root es el peor dentro del top-k; un candidato mejor lo reemplaza. La policy de tie es explícita: mayor `candidate_id` gana cuando scores empatan, solo como regla sintética reproducible.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass
from heapq import heapreplace, heappush


@dataclass(frozen=True)
class Candidate:
    candidate_id: str
    score: float


def top_k(candidates: list[Candidate], k: int) -> list[Candidate]:
    if k <= 0:
        return []
    if len({candidate.candidate_id for candidate in candidates}) != len(candidates):
        raise ValueError("duplicate candidate_id")

    heap: list[tuple[float, str, Candidate]] = []
    for candidate in candidates:
        entry = (candidate.score, candidate.candidate_id, candidate)
        if len(heap) < k:
            heappush(heap, entry)
        elif entry[:2] > heap[0][:2]:
            heapreplace(heap, entry)

    return [
        entry[2]
        for entry in sorted(heap, key=lambda entry: (entry[0], entry[1]), reverse=True)
    ]


candidates = [
    Candidate("cand-A", 0.7),
    Candidate("cand-B", 0.9),
    Candidate("cand-C", 0.9),
    Candidate("cand-D", 0.6),
]

print([(item.candidate_id, item.score) for item in top_k(candidates, 2)])
print(top_k(candidates, 0))
```

Output:

```text
[('cand-C', 0.9), ('cand-B', 0.9)]
[]
```

El heap bounded selecciona; el `sorted()` final ordena solo `k` outputs, O(k log k). Por tanto una expresión más precisa es O(n log k + k log k), que se resume O(n log k) cuando `k <= n` y domina el scan.

Scores son datos sintéticos: no sustituyen provenance, evidence policy ni domain validity. Retrieval real se estudia mucho después.

### Workload

¿Qué cambia si `k` está cerca de `n` o si necesitas todos los resultados ordenados?

---

## 21. Full sort frente a heap

Un heap no gana por definición.

| Workload | Estrategia probable | Razón |
|---|---|---|
| `n` pequeño | `sorted()` | Menos código y constantes excelentes |
| `k` cercano a `n` | `sorted()` | Necesitas casi todo; el heap aporta poco |
| Todos los resultados ordenados | `sorted()` | Heap requeriría pop repetido y más ceremonia |
| `k << n`, batch grande | Heap bounded / `nlargest` | Limita trabajo y auxiliary state a k |
| Stream incremental, siguiente priority | Heap | Push/pop sin re-sort completo |

`heapq.nlargest(k, items, key=...)` y `nsmallest` expresan top-k de standard library. La implementación manual anterior existe para mostrar la invariante y tie policy.

### Decide

Para `n=100`, `k=80`, ¿qué evidencia faltaría antes de afirmar que heap es mejor?

---

## 22. Lookup arbitrario, deletion y dos invariantes

Buscar un item cualquiera en heap es O(n). Remover por ID tampoco es eficiente con `heapq` por sí solo.

Combinar:

```text
heap por priority
+
dict por ID
```

puede acelerar lookup, pero introduce dos invariantes:

1. toda entry vigente del heap corresponde a la versión esperada del dict;
2. toda entidad activa del dict tiene la entry necesaria en heap.

Eso añade memory, update complexity y failure modes. No lo adoptes por reflejo ni conviertas ninguna vista derivada en source of truth.

Actualizar priorities puede requerir reinsertion/stale entries o una estructura distinta. CS-M5 solo define la frontera; no implementa una priority queue mutable avanzada.

### Invariante

Si una operación actualiza el dict pero falla antes de actualizar el heap, ¿qué query devuelve un resultado incoherente?

---

## 23. Tres experimentos EIDOLON, no una arquitectura

### A. Tree traversal

Taxonomía sintética para preorder/postorder, height, traversal iterativo y rechazo de repeated identity.

### B. BST educativo

Índice ordered experimental para insert/search/inorder y degradación skewed. No storage ni decisión de arquitectura.

### C. Priority ranking

Candidatos sintéticos `(candidate_id, score)` para heapq, tie-breaker, top-k y benchmark vs full sort. No embeddings ni AI real.

Las tres estructuras son derived views reconstruibles. Source authority, provenance y evidencia no cambian.

### Source discipline

¿Qué input mínimo conservarías para reconstruir cada experimento? ¿Qué mutation evitarías sobre source?

---

## 24. Testing por invariantes y edge cases

Una suite debe comprobar propiedades, no arrays internos accidentales.

| Estructura | Casos mínimos |
|---|---|
| Tree traversal | empty, leaf, left/right only, balanced-ish, skewed, repeated identity |
| BST | empty search, root, missing, duplicate, inorder, violation profunda, skew |
| Heap | empty peek/pop, one, duplicates, ascending/descending input, invariant tras cada op |
| Priority queue | equal priorities, non-orderable items, deterministic tie, mutation policy |
| Top-k | `k<=0`, `k=1`, `k<n`, `k>=n`, ties, duplicate IDs, source intacto |

**Ejemplo ejecutable — property del heap:**

```python
from heapq import heapify, heappop, heappush


def is_min_heap(items: list[int]) -> bool:
    return all(items[(child - 1) // 2] <= items[child] for child in range(1, len(items)))


for sample in ([], [1], [3, 1, 2, 1], list(range(10)), list(range(9, -1, -1))):
    heap = sample.copy()
    heapify(heap)
    assert is_min_heap(heap)
    heappush(heap, -1)
    assert is_min_heap(heap)
    popped = [heappop(heap) for _ in range(len(heap))]
    assert popped == sorted([*sample, -1])

print("heap_properties=PASS")
```

Output:

```text
heap_properties=PASS
```

Para traversal, una property útil es que el output contenga exactamente cada node identity una vez bajo la precondition de tree. Para BST válido, inorder debe ser estrictamente creciente con la policy sin duplicates.

### Diseña la property

¿Cómo comprobarías que `top_k(candidates, k)` devuelve exactamente `min(k, n)` IDs únicos y que ningún candidato excluido tiene rank mejor que el peor incluido?

---

## 25. Benchmark de shape BST

El costo O(h) debe observarse con el mismo conjunto de keys y distintos insertion orders. Medimos search, no generación.

**Benchmark ejecutable:**

```python
from time import perf_counter


def build_bst(keys: list[int]) -> tuple[int, dict[int, list[int | None]]]:
    root = keys[0]
    links: dict[int, list[int | None]] = {root: [None, None]}
    for key in keys[1:]:
        current = root
        while True:
            side = 0 if key < current else 1
            child = links[current][side]
            if child is None:
                links[current][side] = key
                links[key] = [None, None]
                break
            current = child
    return root, links


def search_steps(root: int, links: dict[int, list[int | None]], target: int) -> int:
    steps = 0
    current: int | None = root
    while current is not None:
        steps += 1
        if current == target:
            return steps
        current = links[current][0 if target < current else 1]
    return steps


def midpoint_order(low: int, high: int) -> list[int]:
    if low > high:
        return []
    mid = (low + high) // 2
    return [mid, *midpoint_order(low, mid - 1), *midpoint_order(mid + 1, high)]


n = 255
balanced_root, balanced_links = build_bst(midpoint_order(0, n - 1))
skewed_root, skewed_links = build_bst(list(range(n)))

started = perf_counter()
balanced_steps = search_steps(balanced_root, balanced_links, n - 1)
balanced_seconds = perf_counter() - started

started = perf_counter()
skewed_steps = search_steps(skewed_root, skewed_links, n - 1)
skewed_seconds = perf_counter() - started

print(balanced_steps, skewed_steps)
print(balanced_seconds >= 0 and skewed_seconds >= 0)
```

Output estable:

```text
8 255
True
```

Los tiempos individuales no son universales; los steps verifican la causa estructural. Repite con varios `n` antes de interpretar growth.

### Benchmark

¿Por qué comparar dos conjuntos de keys distintos confundiría shape con distribución de targets?

---

## 26. Benchmark de full sort y top-k

El benchmark debe variar `n` y `k`, usar candidates idénticos y validar outputs antes de interpretar tiempos.

**Benchmark ejecutable:**

```python
from heapq import heapreplace, heappush
from random import Random
from time import perf_counter


def top_k_heap(values: list[int], k: int) -> list[int]:
    if k <= 0:
        return []
    heap: list[int] = []
    for value in values:
        if len(heap) < k:
            heappush(heap, value)
        elif value > heap[0]:
            heapreplace(heap, value)
    return sorted(heap, reverse=True)


def measure(n: int, k: int) -> tuple[float, float]:
    rng = Random(2026 + n)
    values = [rng.randrange(n * 10) for _ in range(n)]

    started = perf_counter()
    by_sort = sorted(values, reverse=True)[:k]
    sort_seconds = perf_counter() - started

    started = perf_counter()
    by_heap = top_k_heap(values, k)
    heap_seconds = perf_counter() - started

    assert by_heap == by_sort
    return sort_seconds, heap_seconds


rows = [measure(n, k) for n, k in ((100, 5), (1_000, 10), (1_000, 800))]
measurements = [value for row in rows for value in row]
print(f"measurements={len(measurements)}")
print(f"all_non_negative={all(value >= 0 for value in measurements)}")
```

Output estable:

```text
measurements=6
all_non_negative=True
```

Un failure case común crea un `Random` nuevo por elemento y repite el primer valor: correctness puede pasar, pero el workload carece de diversidad.

**Código incorrecto:**

```python
values = [Random(2026 + n).randrange(n * 10) for _ in range(n)]
```

La versión ejecutable crea un solo `rng` antes de la comprehension. Un benchmark puede ejecutar sin ser conceptualmente válido.

### Workload

Predice qué fila favorece menos al heap. Después mide con múltiples repeticiones y conserva la mediana, como en CS-M1.

---

## 27. Failure cases y diagnóstico

1. **Binary tree = BST.** Síntoma: inorder no queda ordenado. Causa: “máximo dos children” no implica order invariant. Corrección: validar bounds globales.
2. **BST siempre O(log n).** Síntoma: paths crecen como n. Causa: insertion order skewed sin balancing. Corrección: declarar O(h), medir shape o usar otra estructura.
3. **Duplicates sin policy.** Síntoma: identidad perdida o search inconsistente. Corrección: reject/count/side policy explícita; IDs duplicados se rechazan aquí.
4. **Depth = height.** Síntoma: números correctos solo para root/leaf especiales. Corrección: dibujar dirección y convención de edges.
5. **Heap globalmente sorted.** Síntoma: iterar la list no entrega prioridades. Corrección: interpretar root/property o usar pops/sort según contrato.
6. **Priority mutable in-place.** Síntoma: root deja de ser mejor. Corrección: immutable entry y reinsertion controlada.
7. **Tie compara objects.** Síntoma: `TypeError` solo cuando priorities empatan. Corrección: comparable secondary key antes del item.
8. **Full sort para stream top-k por reflejo.** Puede repetir O(n log n) trabajo. Compara heap incremental; no prohíbas sort sin medir.
9. **Heap cuando necesitas todo sorted.** Pop repetido añade complejidad; `sorted()` suele expresar mejor el contrato.
10. **BST como database index.** Omite persistence, pages, concurrency y recovery. No es una decisión válida desde este modelo educativo.
11. **Arbitrary heap lookup O(1).** Root access no generaliza; scan sigue O(n).
12. **Heap + dict sin atomicidad lógica.** Una partial update rompe coherencia entre derived views.
13. **Traversal recursivo sobre skew extremo.** Puede lanzar `RecursionError`; usa stack explícito o limita/rechaza depth según contrato.
14. **Repeated child/cycle tratado como tree.** Puede duplicar trabajo o no terminar; valida identidad y escala el modelo a CS-M6.
15. **Benchmark con generator defectuoso.** Inputs sin diversidad no representan workload; inspecciona dataset antes de medir.

### Detecta el bug

Para cada caso, conserva estructura/input, operación, invariant esperada y output/exception. Clasifica si falló model, implementation o measurement.

---

## 28. Ejercicios guiados con solución razonada

Predice o dibuja antes de leer cada solución.

### 28.1 Terminología

**Objetivo/input.** Usa el diagrama de la sección 1; identifica root, leaves, siblings y subtree `personal`.  
**Solución.** Root=`journal`; leaves=`health, travel, projects`; siblings=`personal/work` y `health/travel`; subtree contiene `personal, health, travel`.  
**Por qué/costo.** Son relaciones, no operaciones costosas aún.  
**Criterio.** No llama sibling a parent/child.  
**Variación.** Añade un único child a `health`.

### 28.2 Depth y height

**Objetivo/input.** Calcula ambos para `personal` y `travel`.  
**Solución.** Con edges: `personal` depth 1, height 1; `travel` depth 2, height 0.  
**Por qué.** Depth parte de root; height baja a la leaf más distante.  
**Criterio.** Declara convención.  
**Variación.** Calcula height de empty tree.

### 28.3 Construir `TreeNode`

**Objetivo/input.** Representa `2` con children `1` y `3`.  
**Solución ejecutable:**

```python
from __future__ import annotations
from dataclasses import dataclass


@dataclass
class TreeNode:
    value: int
    left: TreeNode | None = None
    right: TreeNode | None = None


root = TreeNode(2, TreeNode(1), TreeNode(3))
assert root.left and root.left.value == 1
assert root.right and root.right.value == 3
```

**Por qué.** Fields guardan references opcionales.  
**Criterio.** No comparte accidentalmente el mismo child.  
**Variación.** Construye una leaf.

### 28.4 Preorder

**Objetivo/input.** Traza `A(B(D,E),C)`.  
**Solución.** `A B D E C`: node aparece antes de llamadas left/right.  
**Invariante/costo.** Cada subtree produce preorder exactamente una vez; O(n).  
**Criterio.** Parent precede descendants.  
**Variación.** Cambia B por leaf.

### 28.5 Inorder

**Objetivo/input.** Usa el mismo tree.  
**Solución.** `D B E A C`. Solo sería sorted si keys cumplen BST.  
**Invariante/costo.** Left completo, node, right completo; O(n).  
**Criterio.** No afirma orden para cualquier binary tree.  
**Variación.** Usa el binary tree inválido de sección 6.

### 28.6 Postorder

**Objetivo/input.** Usa el mismo tree.  
**Solución.** `D E B C A`.  
**Invariante/costo.** Children se procesan antes de parent; O(n).  
**Criterio.** Root es último.  
**Variación.** Formula un aggregate de subtree.

### 28.7 Traversal iterativo

**Objetivo/input.** Reproduce preorder con stack.  
**Solución.** Push right antes de left; usa `preorder_iterative` de sección 4.  
**Invariante/costo.** Stack contiene roots de subtrees pendientes; O(n) time, O(h) en este DFS estructurado.  
**Criterio.** Output coincide con recursive.  
**Variación.** Registra máximo stack size.

### 28.8 BST search

**Objetivo/input.** Busca `6` en keys `8,3,10,1,6,14`.  
**Solución.** Path `8 → 3 → 6`.  
**Invariante/costo.** Target, si existe, permanece en subtree elegida; O(h).  
**Criterio.** Explica cada descarte.  
**Variación.** Busca `7`.

### 28.9 BST insert

**Objetivo/input.** Inserta `4` en ese BST.  
**Solución.** `8 left → 3 right → 6 left`; crea leaf `4`.  
**Invariante/costo.** Bounds `3 < 4 < 6`; O(h).  
**Criterio.** Inorder queda `[1,3,4,6,8,10,14]`.  
**Variación.** Intenta duplicate `4`.

### 28.10 Demostrar skew

**Objetivo/input.** Inserta `1..7`.  
**Solución.** Cadena right, height 6, search `7` visita 7 nodes.  
**Costo.** O(h)=O(n).  
**Criterio.** No escribe O(log n).  
**Variación.** Compara midpoint order.

### 28.11 Validar BST

**Objetivo/input.** Root 7, left 4, right child de 4 igual 8.  
**Solución.** Inválido: el bound upper heredado es 7.  
**Invariante/costo.** `is_valid_bst` visita O(n), stack O(h).  
**Criterio.** Un check parent-only no cuenta.  
**Variación.** Añade duplicate 7 profundo.

### 28.12 Heap como array

**Objetivo/input.** Dibuja `[1,3,2,7,8,4]`.  
**Solución.** Edges `0→1,2`; `1→3,4`; `2→5`.  
**Invariante.** Cada parent `<=` children.  
**Criterio.** No exige `3<=2`.  
**Variación.** Añade index 6.

### 28.13 Derivar indices

**Objetivo/input.** Para `i=3`, size 9.  
**Solución.** parent 1, left 7, right 8.  
**Por qué.** Fórmulas zero-based de sección 13.  
**Criterio.** No calcula parent para root sin condición.  
**Variación.** Repite con `i=8`.

### 28.14 Sift up

**Objetivo/input.** Inserta `1` en `[2,5,4,9,7]`.  
**Solución.** Swaps indices `5↔2`, luego `2↔0`; final `[1,5,2,9,7,4]`.  
**Invariante/costo.** Solo ancestors pueden violar; O(log n).  
**Criterio.** Comprueba heap property final.  
**Variación.** Inserta 6 y explica cero swaps.

### 28.15 Sift down

**Objetivo/input.** Pop de `[1,3,2,7,8,4]`.  
**Solución.** Mueve 4 a root, elige child 2; final `[2,3,4,7,8]`.  
**Invariante/costo.** Subtrees ya eran heaps; repara un path O(log n).  
**Criterio.** Elige menor child.  
**Variación.** Caso de un elemento.

### 28.16 Min-heap educativo

**Objetivo/input.** Push `5,1,4,1,3` y pop todo.  
**Solución.** Usa sección 15; pops `[1,1,3,4,5]`.  
**Invariante/costo.** Check tras cada mutation; push/pop O(log n), peek O(1).  
**Criterio.** Empty produce policy explícita.  
**Variación.** Implementa `from_values` con heapify.

### 28.17 Usar `heapq`

**Objetivo/input.** Heapifica `[7,2,5,1,9]`.  
**Solución.** `heap[0]==1`; pops producen `[1,2,5,7,9]`.  
**Invariante/costo.** heapify O(n), pops O(log n).  
**Criterio.** No fija array interno exacto.  
**Variación.** Usa max-heap de Python 3.14.

### 28.18 Priority queue con tie-breaker

**Objetivo/input.** Jobs A/B con priority 1 y C con 0.  
**Solución.** Entries `(priority, sequence, job)`; C sale primero, luego A/B por sequence.  
**Invariante/costo.** Tuple comparable antes de item; push/pop O(log n).  
**Criterio.** Equal priority no compara Job.  
**Variación.** Sustituye sequence por stable ID para replay.

### 28.19 Top-k

**Objetivo/input.** Scores `5,1,9,7`, k=2.  
**Solución.** Heap bounded termina con `7,9`; output ordered `[9,7]`.  
**Invariante/costo.** Root es peor incluido; O(n log k), O(k) space.  
**Criterio.** `k=0` y `k>=n` están definidos.  
**Variación.** Añade ties e ID secondary.

### 28.20 Benchmark sort vs heap

**Objetivo/input.** Mismos candidates, `n` creciente, `k=5` y `k≈n`.  
**Solución.** Repite sección 26 con un solo seeded RNG, múltiples muestras y validation contra full sort.  
**Invariante/costo.** Setup fuera de región; O(n log n) vs O(n log k).  
**Criterio.** Interpreta curves, no un tiempo aislado.  
**Variación.** Mide peak memory.

---

## 29. Ejercicios independientes

1. Etiqueta terminología completa en un tree de tres levels.
2. Calcula depth/height con edges y luego traduce a convención por nodes.
3. Construye un binary tree no-BST y demuestra inorder no ordenado.
4. Implementa preorder/postorder con accumulator para evitar concatenaciones.
5. Implementa inorder iterativo con stack.
6. Implementa level-order y mide máximo width.
7. Rechaza repeated identity sin recorrer indefinidamente.
8. Convierte height recursiva a iterativa.
9. Implementa BST search recursivo y compara space.
10. Implementa insert iterativo con duplicate rejection.
11. Prueba invariante global con violations profundas.
12. Construye balanced-ish y skewed con mismas keys.
13. Mide steps de search en ambos shapes.
14. Explica por qué AVL/Red-Black quedan fuera.
15. Deriva indices parent/children para cinco posiciones.
16. Traza sift up con cero, uno y varios swaps.
17. Traza sift down cuando solo existe left child.
18. Implementa min-heap educativo con empty policy.
19. Añade `is_min_heap` y ejecútalo tras cada mutation.
20. Implementa heapify bottom-up y compáralo con n pushes.
21. Usa `heapq` sin depender del array intermedio.
22. Usa APIs max-heap bajo check Python 3.14.
23. Reproduce el `TypeError` de ties sin sequence.
24. Diseña policy FIFO y otra replay-deterministic.
25. Reproduce mutable priority y corrige por reinsertion.
26. Demuestra arbitrary lookup O(n) contando comparaciones.
27. Diseña invariantes de heap + dict sin implementar scheduler.
28. Implementa top-k con IDs únicos y ties.
29. Prueba `k<=0`, `k=1`, `k=n`, `k>n`.
30. Compara con `heapq.nlargest` y `sorted()`.
31. Benchmarkea `n` y `k` crecientes con inputs idénticos.
32. Mide/cuenta search paths BST además de tiempo.
33. Detecta un generador aleatorio sin diversidad.
34. Construye taxonomía derived y preserva source.
35. Explica cuándo esa taxonomía deja de ser tree.
36. Documenta un workload donde `dict` supera BST.
37. Documenta otro donde full sort supera heap.
38. Define migration trigger cuantitativo sin elegir database.

---

## 30. Preguntas conceptuales

1. ¿Qué diferencia existe entre depth y height?
2. ¿Por qué un tree es naturalmente recursivo?
3. ¿Qué restricciones distinguen tree estricto de graph general?
4. ¿Qué distingue binary tree de BST?
5. ¿Por qué inorder solo ordena un BST válido?
6. ¿Qué parte del traversal cambia entre preorder/inorder/postorder?
7. ¿Por qué traversal es O(n)?
8. ¿Por qué space no es siempre O(log n)?
9. ¿De qué depende realmente BST search?
10. ¿Cómo insertion order produce skew?
11. ¿Por qué validar children inmediatos no basta?
12. ¿Qué policy usa este módulo para duplicates y por qué?
13. ¿Por qué no implementamos AVL/Red-Black?
14. ¿Por qué Python no ofrece BST como default universal?
15. ¿Qué propiedad define min-heap?
16. ¿Por qué heap no está totalmente ordered?
17. ¿Cómo complete shape permite usar list?
18. ¿Por qué root access es O(1) y pop O(log n)?
19. ¿Por qué heapify puede ser O(n)?
20. ¿Qué diferencia existe entre FIFO queue y priority queue?
21. ¿Por qué un heap es implementación y no el ADT mismo?
22. ¿Por qué necesitas tie-breaker?
23. ¿Cómo sequence evita comparar items incompatibles?
24. ¿Qué rompe una priority mutable in-place?
25. ¿Por qué arbitrary heap search es O(n)?
26. ¿Cuándo top-k heap mejora full sort?
27. ¿Cuándo `sorted()` sigue siendo mejor?
28. ¿Qué costo espacial tiene heap bounded?
29. ¿Qué riesgo añade heap + dict?
30. ¿Qué usarías para lookup exacto por event_id?
31. ¿Cómo mantienes tree/heap como derived views?
32. ¿Qué evidencia exige convertir la taxonomía en arquitectura?

---

## 31. Mini challenge — Jerarquía e índice de prioridad derivados

### Objetivo y artefacto

Construye tres experimentos in-memory resolubles solo con Programming Foundations + CS-M1–CS-M5:

```text
cs_m5_challenge/
├── trees.py
├── bst.py
├── ranking.py
├── benchmark.py
├── test_structures.py
└── decision.md
```

### A. Tree inspection

1. Crea una jerarquía sintética de al menos nueve nodes.
2. Calcula height con convención de edges.
3. Implementa preorder, inorder y postorder recursivos sobre un binary tree.
4. Implementa al menos preorder iterativo con stack.
5. Implementa level-order solo para tree con `deque`.
6. Comprueba empty, leaf, balanced-ish y skewed.
7. Rechaza repeated identity/cycle en una entrada que viola tree.

### B. BST educativo

8. Implementa insert/search/inorder con numeric keys.
9. Rechaza duplicate key.
10. Valida bounds globales.
11. Construye las mismas keys con midpoint order y sorted order.
12. Compara height/search steps y benchmark de ambos shapes.
13. Declara explícitamente que no es storage ni architecture decision.

### C. Priority ranking

14. Modela candidates sintéticos con `candidate_id` único y `score`.
15. Usa `heapq` y deterministic secondary key.
16. Implementa top-k con heap bounded.
17. Compara el resultado con full sort.
18. Prueba ties, `k<=0`, `k=1`, `k>=n` y duplicate IDs.
19. Benchmarkea al menos tres pares `(n, k)` con múltiples repeticiones.
20. Explica O(n log n), O(n log k), O(k) space y output ordering.

### D. Source discipline

21. Conserva inputs source sin mutar.
22. Trata trees, BST y heap como derived synthetic views reconstruibles.
23. No atribuyas significado epistemológico a `score`.
24. En `decision.md`, elige estructura para hierarchy traversal, equality lookup, priority root, top-k y full ordering.

### Comprobaciones contractuales

**Continuación — adapta nombres a tus implementaciones:**

```python
source_keys = [4, 2, 6, 1, 3, 5, 7]
source_snapshot = tuple(source_keys)

balanced = build_bst(source_keys)
skewed = build_bst(sorted(source_keys))

assert inorder(balanced) == sorted(source_keys)
assert inorder(skewed) == sorted(source_keys)
assert validate_bst(balanced)
assert validate_bst(skewed)
assert height(balanced) < height(skewed)
assert tuple(source_keys) == source_snapshot

try:
    insert(balanced, 4)
except ValueError:
    pass
else:
    raise AssertionError("duplicate key accepted")

for root in (None, one_node, hierarchy_root):
    assert preorder_recursive(root) == preorder_iterative(root)

source_candidates_snapshot = tuple(source_candidates)
for k in (0, 1, 3, len(source_candidates), len(source_candidates) + 2):
    assert top_k_heap(source_candidates, k) == top_k_sort(source_candidates, k)
assert tuple(source_candidates) == source_candidates_snapshot
```

### Failure cases deliberados

Diagnostica y corrige:

1. binary tree tratado como BST;
2. validator que solo compara parent/child;
3. sorted insertion que degrada height;
4. duplicate key insertado silenciosamente;
5. traversal recursivo sobre skew extremo;
6. heap interpretado como sorted list;
7. equal priorities que comparan objects;
8. priority mutada in-place;
9. top-k sin tie-breaker;
10. benchmark con RNG reiniciado por elemento;
11. datasets distintos para sort y heap;
12. heap + dict propuesto sin invariantes de coherencia.

### Criterio de aprobación

- traversals y orders coinciden con diagramas;
- depth/height y space usan la convención declarada;
- BST search/insert/validation preservan policy;
- skew demuestra O(h)=O(n), sin promesa falsa de O(log n);
- heap invariant pasa después de cada push/pop;
- top-k coincide con full sort bajo la misma policy;
- benchmark separa setup, usa inputs idénticos y reporta `n/k`;
- source queda intacto y derived views son reconstruibles;
- no aparecen general graphs, database, backend, concurrency, networking ni AI real.

---

## 32. Resumen

- Tree expresa parent/child, path único y ausencia de cycles.
- Depth mide edges desde root; height, edges hacia la leaf más profunda.
- Preorder/inorder/postorder cambian cuándo se procesa node.
- Traversal visita O(n); auxiliary space depende de height o width.
- Binary tree limita children; BST añade order invariant global.
- BST search/insert son O(h): balanced-ish puede dar O(log n), skewed O(n).
- Duplicates requieren policy; IDs duplicados se rechazan en el modelo educativo.
- Heap combina complete shape con parent priority, no orden total.
- Root access es O(1); push/pop con sift son O(log n); heapify es O(n).
- `heapq` es la opción práctica; Python 3.14 añade APIs max-heap.
- Priority queue es ADT; heap es una implementación común.
- `(priority, sequence, item)` hace ties comparables/FIFO; replay puede requerir stable key distinta.
- Mutable priorities pueden romper la heap property.
- Top-k bounded cuesta O(n log k) y O(k) space, pero full sort puede ser mejor según workload.
- Trees/heaps EIDOLON de este módulo son experiments derived, no source ni arquitectura.

---

## 33. Checklist de dominio

- [ ] Puedo etiquetar terminología de un tree y calcular depth/height.
- [ ] Puedo explicar el modelo recursivo y sus límites.
- [ ] Puedo implementar y trazar preorder/inorder/postorder.
- [ ] Puedo usar stack explícito y level-order sobre tree.
- [ ] Puedo derivar O(n) traversal y space dependiente de shape.
- [ ] Puedo distinguir binary tree, BST, heap y graph.
- [ ] Puedo implementar BST search/insert O(h).
- [ ] Puedo rechazar duplicates y validar bounds globales.
- [ ] Puedo demostrar balanced-ish frente a skewed.
- [ ] Puedo explicar por qué BST no garantiza O(log n).
- [ ] Puedo derivar parent/children indices de heap.
- [ ] Puedo trazar sift up/down.
- [ ] Puedo implementar y verificar min-heap educativo.
- [ ] Puedo explicar heapify O(n) intuitivamente.
- [ ] Puedo usar `heapq` sin depender del array interno.
- [ ] Puedo usar max-heap solo bajo baseline compatible.
- [ ] Puedo distinguir priority queue de FIFO queue.
- [ ] Puedo diseñar tie-breaker determinista y comparable.
- [ ] Puedo detectar priorities mutadas y arbitrary lookup O(n).
- [ ] Puedo implementar top-k O(n log k), O(k) space.
- [ ] Puedo decidir entre full sort y heap mediante benchmark.
- [ ] Puedo mantener tree/heap como derived views.
- [ ] Puedo completar el challenge con PF + CS-M1–CS-M5.

---

## 34. Preparación para labs y EIDOLON 0.0b

### Labs principales

- **CS-L08 — Top-k evidence:** heap de k candidates con score/tie-breaker y comparación contra full sort.
- **CS-L09 — Tree traversal:** taxonomía sintética, DFS/BFS de tree, depth limit y entrada cíclica rechazada.

CS-L09 usa “DFS/BFS” como evidencia canónica. Este módulo prepara preorder/inorder/postorder, stack explícito y level-order sobre tree; CS-M6 generalizará visited/path semantics a graphs.

### Lab previo de medición

- **CS-L01 — Curvas de crecimiento:** aporta sizes, repetitions, curvas y break-even para los benchmarks.

### Trazabilidad

| Concepto | Secciones | Evidencia | Lab |
|---|---:|---|---|
| Terminología, height, traversal | 1–5 | Guiados 1–7; challenge A | CS-L09 |
| Tree inválido/cycle | 11, 27 | Ejercicio 7; failure case | CS-L09 |
| BST shape O(h) | 6–10, 25 | Guiados 8–11; challenge B | Fundamento de CS-L09 |
| Heap invariant/sifts | 12–17 | Guiados 12–17; challenge C | CS-L08 |
| Tie-breaking/top-k | 18–22 | Guiados 18–19; challenge C | CS-L08 |
| Benchmark sort vs heap | 26 | Guiado 20; `decision.md` | CS-L01, CS-L08 |

El Curriculum no define un lab BST separado; no se inventa un ID. Antes de avanzar entrega código, edge cases, invariant checks, trazas, benchmark y decisión por workload.

---

## 35. Recursos de ampliación

El módulo es autocontenido. Para ampliar, usa los recursos canónicos de [CS.11 Recursos recomendados](../../02_curriculum/02_computer_science_foundations.md#cs11-recursos-recomendados) y documentación oficial de Python 3.14 para `heapq`.

La documentación verifica API/versiones; no reemplaza dibujar invariantes, declarar tie policy ni medir el workload.

---

## 36. Límite explícito del módulo

CS-M5 termina en tree terminology, binary tree/BST educativo, traversals de tree, shape/height, complete binary heap, sift/heapify, `heapq`, priority queue, deterministic ties, top-k y benchmark.

No implementa AVL, Red-Black, B/B+ trees, tries, segment/interval trees, Fibonacci/binomial heaps, general graph traversal, state machines, database indexes, backend, processes, threads, networking ni AI.

El siguiente paso permitido es revisar CS-M5 como `review candidate`. **No se crea ni se desarrolla CS-M6 en esta entrega.**
