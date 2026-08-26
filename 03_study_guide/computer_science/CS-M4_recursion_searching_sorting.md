# CS-M4 — Recursión, búsqueda y ordenamiento

**Track:** Computer Science Foundations  
**Competencias:** D2.1; soporte D2.3  
**Fase:** P0  
**Nivel objetivo:** Aplicado  
**Prerequisites:** PF-M2, PF-M3, PF-M9, CS-M1, CS-M2, CS-M3  
**Build:** EIDOLON 0.0b  
**Curriculum source:** [CS-M4](../../02_curriculum/02_computer_science_foundations.md#cs-m4--recursion-searching-y-sorting)  
**Status:** review candidate

Dos programas pueden producir el mismo resultado y, aun así, exigir cantidades muy distintas de tiempo, memoria o preprocesamiento. También pueden parecer correctos mientras violan una precondition: binary search sobre datos desordenados puede devolver una respuesta falsa sin lanzar ninguna excepción.

CS-M4 sigue esta cadena:

```text
problema
↓
estructura y preconditions
↓
invariante
↓
algoritmo
↓
complejidad temporal y espacial
↓
medición
↓
tradeoff para el workload
```

Implementaremos algoritmos clásicos para observar sus invariantes, no para reemplazar la biblioteca estándar en producción. No se desarrollan trees, heaps, graphs, OS internals, concurrency, networking ni arquitectura. Esos temas comienzan en CS-M5 y módulos posteriores.

## Resultados de aprendizaje

Al terminar podrás:

- trazar una función recursiva y distinguir base case, recursive case y progreso;
- explicar qué conserva cada frame conceptual del call stack;
- detectar recursion sin progreso y elegir iteración cuando la profundidad sea riesgosa;
- implementar linear search con semántica explícita de ausencia;
- derivar sus costos best y worst case sin inventar un average case;
- declarar y comprobar la precondition de orden de binary search;
- implementar binary search iterativo y recursivo para empty, one, first, last y missing;
- explicar por qué dividir repetidamente entre dos produce O(log n);
- distinguir coincidencia cualquiera, primera frontera e insertion point;
- elegir entre scan, binary search e índice `dict` según preprocessing y cantidad de queries;
- definir una política de orden mediante `key`, dirección, stability y tie-breakers;
- distinguir sorting in-place de una vista derivada out-of-place;
- implementar y analizar selection sort, insertion sort y merge sort;
- justificar O(n²), O(n log n) y auxiliary space con modelos simples;
- comprobar correctness y stability con edge cases y properties;
- medir search/sort sin mezclar setup ni usar inputs distintos;
- construir una timeline determinista sin mutar el source de EIDOLON.

## Cómo estudiar este módulo

1. Antes del código, escribe precondition, contrato de ausencia e invariante.
2. Traza inputs pequeños a mano; luego ejecuta.
3. Separa correctness, complejidad y medición: ninguna reemplaza a las otras.
4. En benchmarks, separa preprocessing de query cost y reutiliza exactamente los mismos datos.
5. Trata cada timeline ordenada y cada índice como derived data reconstruible.
6. Resuelve la práctica inmediata antes de avanzar a los ejercicios guiados.

### Convenciones

- **Ejemplo ejecutable:** bloque autónomo con output estable o `assert`.
- **Failure case ejecutable:** reproduce un resultado incorrecto o una excepción controlada.
- **Código incorrecto:** antipatrón deliberado que no debe ejecutarse sin el límite indicado.
- **Benchmark ejecutable:** los tiempos son ambientales; solo se afirma una propiedad estable.
- **Fragmento:** omite contexto y no se ofrece como programa autónomo.
- **Implementación educativa:** existe para observar invariantes; la recomendación de producción se declara aparte.

Baseline recomendado: Python 3.14. Los ejemplos principales usan standard library. La property con Hypothesis reutiliza la dependency de desarrollo ya presentada en PF-M9.

---

## 1. Por qué estudiar algoritmos clásicos

Supón dos formas de localizar un evento:

```text
scan de una list sin ordenar
índice dict ya construido
```

Ambas pueden devolver el mismo evento. La decisión cambia con el workload:

- un scan único puede evitar el costo de construir y mantener un índice;
- muchas consultas por ID estable pueden amortizar el índice;
- ordenar por tiempo no acelera automáticamente lookup por ID;
- una mejor clase asintótica puede perder para inputs pequeños o setup dominante.

Implementar algoritmos clásicos permite observar:

- qué precondition hace válida una estrategia;
- qué parte del estado ya está correcta —su **invariante**—;
- cómo un off-by-one rompe un resultado;
- qué recurso crece con `n`;
- dónde debe comenzar y terminar una medición.

En producción, `sorted()`, `list.sort()`, `bisect` y `dict` suelen ser preferibles a una implementación educativa cuando satisfacen el contrato. “Lo escribí yo” no es un requisito de producto.

### Decide

Tienes 30 eventos no ordenados y una sola consulta. ¿Qué costo nuevo introduciría construir un índice? ¿Qué dato podría cambiar tu decisión?

---

## 2. De iteración a recursión

Primero observa un problema conocido: sumar los enteros de `1` a `n`, con `n >= 0`.

**Ejemplo ejecutable — estado explícito en un loop:**

```python
def sum_to_iterative(n: int) -> int:
    total = 0
    for value in range(1, n + 1):
        total += value
    return total


print(sum_to_iterative(4))
print(sum_to_iterative(0))
```

Output:

```text
10
0
```

El loop conserva explícitamente `total` y `value`. La versión recursiva expresa el mismo problema como una instancia pequeña del propio problema:

```text
sum_to(n) = n + sum_to(n - 1)
sum_to(0) = 0
```

**Ejemplo ejecutable — estado distribuido entre llamadas:**

```python
def sum_to_recursive(n: int) -> int:
    if n == 0:
        return 0
    return n + sum_to_recursive(n - 1)


print(sum_to_recursive(4))
print(sum_to_recursive(0))
```

Output:

```text
10
0
```

Ambas versiones requieren la precondition `n >= 0`. Una firma tipada no impide `sum_to_recursive(-1)`; type correctness no equivale a domain validity, como estudiaste en PF-M5.

### Traza

Sin ejecutar, escribe las llamadas producidas por `sum_to_recursive(3)` y el orden en que se realizan las sumas al retornar.

### Modifica

Añade una validación mínima que rechace `n < 0` con `ValueError`. PF-M6 ya enseñó este mecanismo; aquí la decisión importante es preservar la precondition.

---

## 3. Modelo mental: base case, recursive case y progreso

Una recursion correcta necesita tres piezas:

```text
base case
+
recursive case
+
progreso hacia el base case
```

Para `sum_to_recursive(3)` el descenso es:

```text
sum_to_recursive(3)
└── 3 + sum_to_recursive(2)
    └── 2 + sum_to_recursive(1)
        └── 1 + sum_to_recursive(0)
            └── base: 0
```

Después ocurre el **unwind**, cuando cada llamada pendiente recibe un resultado:

```text
0
↑ 1 + 0 = 1
↑ 2 + 1 = 3
↑ 3 + 3 = 6
```

El base case detiene la reducción. El recursive case resuelve una parte y delega una versión menor. El progreso es una propiedad de la transición: `n - 1` acerca un entero no negativo a cero.

**Ejemplo ejecutable — traza observable:**

```python
def trace_countdown(n: int) -> None:
    print(f"enter {n}")
    if n == 0:
        print("base")
        return
    trace_countdown(n - 1)
    print(f"exit {n}")


trace_countdown(3)
```

Output:

```text
enter 3
enter 2
enter 1
enter 0
base
exit 1
exit 2
exit 3
```

La impresión posterior a la llamada ocurre durante el unwind, no durante el descenso.

### Base case

¿Qué cambia si el caso base es `n == 1`? Razona por separado para inputs `3`, `1` y `0`.

### Progress

Para un input no negativo, ¿por qué `n // 2` progresa hacia cero más rápido que `n - 1`? Esto no basta para declarar complejidad: también importa el trabajo hecho por llamada.

---

## 4. Call stack, profundidad y failure cases

Conecta con el stack de CS-M3. Cada llamada activa conserva un **frame conceptual** con:

- argumentos;
- variables locales;
- punto al que debe retornar;
- resultado pendiente de llamadas internas.

No necesitamos internals de memoria para predecir la consecuencia: una cadena de profundidad `d` mantiene aproximadamente `d` llamadas activas. Python impone un límite práctico y puede lanzar `RecursionError` antes de agotar toda la memoria del proceso.

### 4.1 Failure case: existe base case, pero no hay progreso

**Código incorrecto — no ejecutar sin un límite externo:**

```python
def countdown(n: int) -> None:
    if n == 0:
        return
    countdown(n + 1)  # se aleja de cero para n positivo
```

Síntoma: termina en `RecursionError`, no en el caso base. Causa: la transición viola el argumento de progreso. Corrección: usar `n - 1` bajo `n >= 0`, o elegir un loop.

### 4.2 Failure case: base case incompleto

**Failure case ejecutable:**

```python
def broken_sum(n: int) -> int:
    if n == 1:
        return 1
    return n + broken_sum(n - 1)


try:
    broken_sum(0)
except RecursionError:
    print("RecursionError: el caso 0 quedó fuera del contrato")
```

Output estable:

```text
RecursionError: el caso 0 quedó fuera del contrato
```

No se recomienda aumentar arbitrariamente `sys.setrecursionlimit` como corrección normal. Eso desplaza el límite sin corregir falta de progreso ni vuelve apropiado un traversal lineal profundo.

### 4.3 Recursión frente a iteración

| Pregunta | Recursión | Iteración |
|---|---|---|
| ¿Dónde vive el estado? | En argumentos, locals y retornos de frames | En variables y estructuras explícitas |
| ¿Usa call stack por paso? | Sí | No para cada iteración |
| ¿Tiene riesgo por depth en Python? | Sí | Normalmente menor |
| ¿Puede reflejar la estructura del problema? | Sí, especialmente divide and conquer o nesting acotado | Sí, a veces con stack explícito |
| ¿Es automáticamente más clara? | No | No |

Python no realiza tail-call optimization general. Reescribir una función como tail recursion no elimina su crecimiento de call stack.

La recursión surge naturalmente en estructuras nested, divide and conquer y los trees que estudiará CS-M5. Para millones de elementos lineales, un loop suele ofrecer un contrato de profundidad más seguro.

### 4.4 Sustituir depth implícita por un stack explícito

Una estructura nested cuyas hojas son strings puede recorrerse sin una llamada por nivel. El stack de CS-M3 hace visible el trabajo pendiente:

**Ejemplo ejecutable:**

```python
def flatten_strings(values: list[object]) -> list[str]:
    pending = list(reversed(values))
    output: list[str] = []

    while pending:
        value = pending.pop()
        if isinstance(value, list):
            pending.extend(reversed(value))
        else:
            assert isinstance(value, str)
            output.append(value)

    return output


print(flatten_strings(["A", ["B", ["C"]], "D"]))
```

Output:

```text
['A', 'B', 'C', 'D']
```

La estructura puede seguir siendo muy grande, pero la depth ya no consume call frames. El stack explícito también permite imponer un límite, inspeccionar pendientes o pausar el traversal. No se introducen trees ni graphs; el ejemplo solo transfiere la idea a lists nested.

### Detecta el bug

Una función tiene `if not items: return 0`, pero llama recursivamente con `items` en vez de `items[1:]`. ¿Qué pieza existe y cuál falta?

### Decide

Debes procesar una lista lineal de diez millones de IDs. ¿Qué riesgo específico introduce una llamada por elemento aunque el código recursivo sea corto?

---

## 5. Linear search: contrato, invariante y costo

Linear search examina elementos en secuencia. Su precondition mínima es poder iterar y comparar cada elemento con el target; no exige orden.

Primero decide el contrato. Esta versión devuelve el índice de la primera coincidencia o `None`:

**Ejemplo ejecutable:**

```python
def linear_search(items: list[str], target: str) -> int | None:
    for index, item in enumerate(items):
        if item == target:
            return index
    return None


ids = ["evt-003", "evt-001", "evt-002", "evt-001"]
print(linear_search(ids, "evt-001"))
print(linear_search(ids, "evt-999"))
print(linear_search([], "evt-001"))
```

Output:

```text
1
None
None
```

La invariante antes de examinar posición `i` es:

> El target no apareció en las posiciones `0` a `i - 1`.

Si `items[i] == target`, ese índice es la primera coincidencia. Si el loop termina, la invariante cubre toda la colección y justifica `None`.

Costos para `n = len(items)`:

- best case: O(1), target en la primera posición;
- worst case: O(n), target al final o ausente;
- auxiliary space: O(1) para esta implementación;
- average/expected: no se afirma sin una distribución sobre posiciones y ausencia.

Linear search suele ser suficiente con input pequeño, una consulta, datos no ordenados o cuando construir otra vista no se amortiza.

### Predice

¿Cuántas comparaciones realiza para buscar `evt-001` y `evt-999` en el ejemplo? Distingue resultado de costo.

### Modifica

Cambia el contrato para retornar todos los índices coincidentes. ¿Cómo cambia la posibilidad de terminar temprano?

---

## 6. Binary search empieza por una precondition

Si los datos están ordenados por la misma relación usada en la consulta, una comparación con el elemento medio permite descartar aproximadamente la mitad.

```text
items = [10, 20, 30, 40, 50, 60, 70]
target = 60

low=0, high=6, mid=3 → items[3]=40 < 60
descartar 0..3

low=4, high=6, mid=5 → items[5]=60
found
```

La precondition crítica es:

> `items` está ordenada de forma no decreciente bajo la misma comparación que guía el algoritmo.

Un type hint como `list[int]` no expresa ni verifica esa propiedad. Un array desordenado puede producir un resultado incorrecto sin excepción.

### 6.1 Failure case: “a veces funciona”

**Failure case ejecutable:**

```python
def binary_search_unchecked(items: list[int], target: int) -> int | None:
    low = 0
    high = len(items) - 1
    while low <= high:
        mid = (low + high) // 2
        if items[mid] == target:
            return mid
        if items[mid] < target:
            low = mid + 1
        else:
            high = mid - 1
    return None


unordered = [40, 10, 30, 20, 50]
print(binary_search_unchecked(unordered, 30))
print(binary_search_unchecked(unordered, 10))
```

Output:

```text
2
None
```

Encontrar `30` accidentalmente no valida el input ni el algoritmo. `10` existe, pero queda en una mitad descartada bajo una inferencia inválida.

### Precondition

Una timeline está ordenada por `valid_time`. ¿Es válido aplicar binary search por `event_id`? Explica qué relación falta.

---

## 7. Binary search iterativo

Usaremos un intervalo cerrado `[low, high]`: ambos extremos todavía pueden contener el target.

**Ejemplo ejecutable:**

```python
def binary_search(items: list[int], target: int) -> int | None:
    low = 0
    high = len(items) - 1

    while low <= high:
        mid = (low + high) // 2
        value = items[mid]

        if value == target:
            return mid
        if value < target:
            low = mid + 1
        else:
            high = mid - 1

    return None


cases = [
    ([], 5, None),
    ([5], 5, 0),
    ([5], 4, None),
    ([1, 3, 5, 7, 9], 1, 0),
    ([1, 3, 5, 7, 9], 9, 4),
    ([1, 3, 5, 7, 9], 4, None),
]

for items, target, expected in cases:
    assert binary_search(items, target) == expected

print(f"cases={len(cases)}")
```

Output:

```text
cases=6
```

### 7.1 Invariante y progreso

Al inicio de cada iteración:

> Si el target existe, está dentro del intervalo cerrado `[low, high]`.

Cuando `value < target`, las posiciones hasta `mid` no pueden contenerlo; por eso `low = mid + 1`. Usar `low = mid` puede dejar el mismo midpoint y no progresar.

`mid = (low + high) // 2` es seguro para enteros de Python en este contexto: no sufren overflow de ancho fijo. Otras plataformas pueden usar otra fórmula, pero ese detalle no cambia la invariante.

### 7.2 Failure cases off-by-one

**Código incorrecto:**

```python
low = 0
high = len(items)       # incorrecto para intervalo cerrado
while low < high:       # excluye el caso low == high
    mid = (low + high) // 2
    ...
```

Hay dos convenciones válidas comunes: intervalo cerrado `[low, high]` e intervalo semiabierto `[low, high)`. El bug aparece al mezclar sus inicializaciones, condiciones y updates.

### Traza

Traza `target=9` en `[1, 3, 5, 7, 9]`. Anota `low`, `high`, `mid` y el intervalo conservado después de cada comparación.

### Detecta el bug

¿Qué input mínimo revela una versión con `while low < high` que solo comprueba `items[mid]` dentro del loop?

---

## 8. Binary search recursivo y O(log n)

La idea algorítmica puede expresarse con llamadas recursivas. Los límites deben ser argumentos para que cada llamada represente un subproblema menor.

**Ejemplo ejecutable:**

```python
def binary_search_recursive(items: list[int], target: int) -> int | None:
    def search(low: int, high: int) -> int | None:
        if low > high:
            return None

        mid = (low + high) // 2
        if items[mid] == target:
            return mid
        if items[mid] < target:
            return search(mid + 1, high)
        return search(low, mid - 1)

    return search(0, len(items) - 1)


items = [2, 4, 6, 8, 10, 12]
assert binary_search_recursive(items, 2) == 0
assert binary_search_recursive(items, 12) == 5
assert binary_search_recursive(items, 7) is None
assert binary_search_recursive([], 7) is None
print("recursive_cases=PASS")
```

Output:

```text
recursive_cases=PASS
```

Cada llamada conserva una de estas partes:

```text
n
→ aproximadamente n/2
→ aproximadamente n/4
→ aproximadamente n/8
→ ...
→ 1
```

El número de divisiones hasta llegar a uno es proporcional a `log2(n)`. Por eso el tiempo worst case es O(log n). La versión iterativa usa O(1) auxiliary space; la recursiva añade O(log n) frames conceptuales.

No hay contradicción con el límite práctico de recursion: `log2(n)` crece lentamente. Aun así, la versión iterativa evita overhead de llamadas y suele ser el default claro en Python para este algoritmo.

### Complejidad

Para `n = 1_024`, ¿cuántas reducciones a la mitad hacen falta aproximadamente? ¿Por qué la respuesta no es 1_024?

### Decide

Si ambas versiones pasan los mismos tests, ¿qué evidencia usarías para elegir? Incluye claridad, stack usage y convenciones del proyecto.

---

## 9. Duplicates, fronteras y `bisect`

La implementación básica promete **alguna coincidencia**. Con duplicates no promete primera ni última:

```text
[1, 2, 2, 2, 3]
       ↑
una coincidencia válida, no necesariamente una frontera
```

Una variante útil busca la primera posición cuyo valor sea `>= target`, también llamada insertion point izquierdo. Python ya ofrece esa operación en `bisect`.

**Ejemplo ejecutable:**

```python
from bisect import bisect_left, bisect_right


items = [1, 2, 2, 2, 4]
left = bisect_left(items, 2)
right = bisect_right(items, 2)

print(left, right)
print(items[left:right])
```

Output:

```text
1 4
[2, 2, 2]
```

`bisect_left` y `bisect_right` requieren la misma precondition de orden. Localizar una frontera cuesta O(log n), pero insertar después en una `list` cuesta O(n) por desplazamiento de referencias. `bisect.insort` no convierte el costo completo de inserción en O(log n).

### Precondition

¿Qué debe ser cierto sobre `items` antes y después de cada inserción si seguirás usando `bisect`?

### Modifica

Usa las dos fronteras para contar ocurrencias de `2` sin recorrer todo el slice. ¿Qué resta expresa el conteo?

---

## 10. Elegir una estrategia de búsqueda

La estructura sale del workload, no de una jerarquía de algoritmos “mejores”.

| Workload | Estrategia probable | Razón y costo omitido que debes revisar |
|---|---|---|
| Una consulta sobre datos pequeños no ordenados | Linear search | O(n), sin preprocessing ni memoria derivada |
| Muchas equality queries por ID estable | Índice `dict` | Lookup esperado O(1), pero build O(n), memoria O(n) y mantenimiento |
| Muchas queries por una key ordenada | Binary search / `bisect` | Query O(log n), requiere vista ordenada y relación consistente |
| Una consulta sobre datos no ordenados | A menudo scan | Ordenar O(n log n) solo para buscar una vez puede perder |
| Query por rango temporal | Fronteras sobre timeline temporal ordenada | No sustituye índice por ID; puede requerir keys auxiliares |

Para `q` consultas sobre `n` eventos:

```text
q scans                    ≈ O(qn)
sort una vez + q búsquedas ≈ O(n log n + q log n)
build dict + q lookups     ≈ O(n + q) esperado
```

Estos modelos omiten constantes, key cost y actualizaciones. CS-M1 ya enseñó que el benchmark representativo decide el break-even local.

### Caso EIDOLON: dos derived views

```text
source events
├── timeline ordenada por tiempo → range queries
└── events_by_id dict            → equality lookup por ID
```

Ninguna vista es source of truth. Ambas deben poder reconstruirse. Ordenar por `valid_time` no vuelve eficiente buscar `event_id`; el orden responde a otra consulta.

### Decide

Recibes un batch inmutable, haces 10 000 consultas por ID y 20 consultas por rango temporal. Justifica si conservarías una o dos vistas derivadas.

---

## 11. Qué significa ordenar

Sorting produce una secuencia bajo una relación de orden. Una política completa responde:

- ¿cuál es la `key`?
- ¿ascending o descending?
- ¿qué ocurre con keys iguales?
- ¿se necesita stability?
- ¿qué tie-breaker hace determinista el resultado?
- ¿se modifica el input o se crea una vista?

Selection, insertion, merge y el sorting general de Python pertenecen aquí al modelo **comparison-based**: toman decisiones desde comparaciones de keys. Este módulo no desarrolla algoritmos especializados por dígitos, buckets u otras propiedades del dominio.

Un **stable sort** preserva el orden relativo previo de elementos cuyas keys comparan iguales.

**Ejemplo ejecutable — stability observable:**

```python
events = [
    {"id": "evt-B", "day": 2, "arrival": 0},
    {"id": "evt-A", "day": 1, "arrival": 1},
    {"id": "evt-C", "day": 2, "arrival": 2},
]

ordered = sorted(events, key=lambda event: event["day"])
print([event["id"] for event in ordered])
```

Output:

```text
['evt-A', 'evt-B', 'evt-C']
```

`evt-B` y `evt-C` tienen la misma key y conservan su orden relativo de entrada.

### Stability no inventa determinismo

Si el input llega cada vez en distinto orden, preservar ese orden distinto produce outputs distintos. Para replay reproducible, una política puede usar una key total explícita:

**Fragmento:**

```python
key=lambda event: (event["day"], event["id"])
```

El tie-breaker debe tener semántica justificada. No se añade solo para esconder incertidumbre temporal.

### Stability

Si primero ordenas por `event_id` y luego realizas un stable sort por `day`, ¿qué orden actúa como desempate dentro de un mismo día?

### Source vs derived

¿Es seguro llamar `.sort()` sobre una lista cuyo orden de llegada conserva evidencia? Responde desde el contrato, no desde la sintaxis.

---

## 12. `sorted()`, `.sort()`, key cost y datos ordenables

`sorted(iterable, key=..., reverse=...)` materializa una nueva `list`. `list.sort(...)` modifica la list existente y devuelve `None`. Ambas operaciones son stable y usan el sorting optimizado de Python; no llaman nuestros algoritmos educativos.

**Ejemplo ejecutable:**

```python
source = [3, 1, 2]
derived = sorted(source)

print(source)
print(derived)

result = source.sort()
print(source)
print(result)
```

Output:

```text
[3, 1, 2]
[1, 2, 3]
[1, 2, 3]
None
```

Para EIDOLON, una timeline suele ser una vista derivada:

**Fragmento:**

```python
timeline = sorted(source_events, key=timeline_key)
```

Esto evita alterar silenciosamente el orden significativo del source.

### 12.1 El costo de `key`

Python calcula la key una vez por elemento durante una operación de sorting, pero ese trabajo sigue formando parte del costo total. Una key que realiza un scan, parsing costoso o I/O no es O(1) por decreto.

```text
costo total ≈ calcular n keys + ordenar referencias
```

Una intuición de decorate-sort-undecorate es materializar pares `(key, value)`, ordenar por key y recuperar values. `key=` expresa normalmente esa intención sin ceremonia manual.

### 12.2 Strings no significan collation lingüística

**Ejemplo ejecutable:**

```python
words = ["árbol", "apple", "Zulu"]
print(sorted(words))
```

Output:

```text
['Zulu', 'apple', 'árbol']
```

Python compara strings lexicográficamente por valores Unicode, no por reglas completas de un idioma o locale. No llames “alfabético para español” a ese contrato.

### 12.3 Datetimes compatibles

**Ejemplo ejecutable:**

```python
from datetime import UTC, datetime


times = [
    datetime(2026, 8, 26, 11, 0, tzinfo=UTC),
    datetime(2026, 8, 26, 9, 0, tzinfo=UTC),
]

print([value.hour for value in sorted(times)])
```

Output:

```text
[9, 11]
```

Usa timezone-aware datetimes compatibles. Mezclar naive y aware al ordenar lanza `TypeError`, como ya demostró PF-M1.

### Complejidad

No asumas que `sorted()` es O(n). La cota general que usamos es O(n log n); Python puede aprovechar patrones del input, pero no dependas de un best case interno no especificado para tu decisión.

### Detecta el bug

¿Por qué `ordered = events.sort(key=...)` deja `ordered` en `None`? ¿Qué objeto cambió?

---

## 13. Selection sort: mínimo restante

Selection sort mantiene un prefijo terminado y busca el mínimo en el sufijo restante:

```text
[4, 2, 3, 1]
 ^ buscar mínimo 1 y colocarlo en posición 0
[1 | 2, 3, 4]
     buscar mínimo 2
[1, 2 | 3, 4]
        ...
```

**Implementación educativa ejecutable:**

```python
def selection_sort(items: list[int]) -> list[int]:
    result = items.copy()

    for start in range(len(result)):
        min_index = start
        for index in range(start + 1, len(result)):
            if result[index] < result[min_index]:
                min_index = index
        result[start], result[min_index] = result[min_index], result[start]

    return result


source = [4, 2, 3, 1]
ordered = selection_sort(source)
print(ordered)
print(source)
```

Output:

```text
[1, 2, 3, 4]
[4, 2, 3, 1]
```

La invariante al inicio de cada iteración externa es:

> El prefijo anterior a `start` contiene los elementos menores ya colocados en su posición final.

El número de comparaciones es:

```text
(n - 1) + (n - 2) + ... + 1
= n(n - 1) / 2
∈ O(n²)
```

La implementación hace relativamente pocos swaps, pero sigue comparando cuadráticamente aun con input ordenado. Copiar el input añade O(n) espacio; el core de swaps podría operar in-place.

### 13.1 Esta versión no es stable

**Ejemplo ejecutable:**

```python
def selection_sort_records(items: list[tuple[int, str]]) -> list[tuple[int, str]]:
    result = items.copy()
    for start in range(len(result)):
        min_index = start
        for index in range(start + 1, len(result)):
            if result[index][0] < result[min_index][0]:
                min_index = index
        result[start], result[min_index] = result[min_index], result[start]
    return result


records = [(2, "A"), (2, "B"), (1, "C")]
print(selection_sort_records(records))
```

Output:

```text
[(1, 'C'), (2, 'B'), (2, 'A')]
```

El swap del mínimo `C` invierte el orden relativo de `A` y `B`, que compartían key `2`. No afirmes stability para un nombre de algoritmo sin revisar la implementación exacta.

### Invariante

Después de colocar la posición `start`, ¿qué sabes del prefijo y qué aún no sabes del sufijo?

### Decide

¿Que selection sort sea corto justifica usarlo para ordenar 100 000 eventos en producción? Compara legibilidad con la existencia de `sorted()` y el crecimiento O(n²).

---

## 14. Insertion sort: prefijo ordenado

Insertion sort conserva un prefijo ordenado e inserta el siguiente elemento en su posición:

```text
[2 | 4, 3, 1]
[2, 4 | 3, 1]  → insertar 3 entre 2 y 4
[2, 3, 4 | 1]  → insertar 1 al inicio
[1, 2, 3, 4]
```

**Implementación educativa ejecutable y stable:**

```python
def insertion_sort(items: list[int]) -> list[int]:
    result = items.copy()

    for index in range(1, len(result)):
        current = result[index]
        position = index

        while position > 0 and result[position - 1] > current:
            result[position] = result[position - 1]
            position -= 1

        result[position] = current

    return result


for sample in ([], [4], [1, 2, 3], [4, 3, 2, 1], [3, 1, 3, 2]):
    assert insertion_sort(sample) == sorted(sample)

print("insertion_cases=PASS")
```

Output:

```text
insertion_cases=PASS
```

Invariante:

> Antes de procesar `result[index]`, el prefijo `result[:index]` ya está ordenado y contiene los mismos elementos que originalmente ocupaban ese prefijo.

Costos:

- best case O(n): cada elemento ya está en orden y el `while` no desplaza;
- worst case O(n²): input inverso desplaza todo el prefijo;
- esta función añade O(n) por la copia; una variante in-place puede usar O(1) auxiliary space.

La comparación es estricta (`>`). Un elemento con key igual no cruza a otro anterior; esa decisión preserva stability.

### Stability

¿Qué cambiaría si el `while` usara `>=` para registros comparados solo por key?

### Decide

Insertion sort puede ser razonable como algoritmo educativo y en segmentos pequeños o casi ordenados. ¿Qué benchmark necesitarías antes de reemplazar `sorted()` en producto?

---

## 15. Divide and conquer y la operación `merge`

Divide and conquer separa un problema, resuelve piezas menores y combina resultados:

```text
divide
↓
solve left + solve right
↓
combine
```

Antes de merge sort necesitamos su operación central. `merge` recibe **dos listas ya ordenadas**; esa es su precondition.

**Ejemplo ejecutable:**

```python
def merge(left: list[int], right: list[int]) -> list[int]:
    merged: list[int] = []
    left_index = 0
    right_index = 0

    while left_index < len(left) and right_index < len(right):
        if left[left_index] <= right[right_index]:
            merged.append(left[left_index])
            left_index += 1
        else:
            merged.append(right[right_index])
            right_index += 1

    merged.extend(left[left_index:])
    merged.extend(right[right_index:])
    return merged


print(merge([1, 4, 7], [2, 2, 9]))
print(merge([], [3, 5]))
```

Output:

```text
[1, 2, 2, 4, 7, 9]
[3, 5]
```

Invariante del loop:

> `merged` está ordenada y contiene exactamente los elementos consumidos de `left` y `right`.

Elegir el lado izquierdo con `<=` en un empate es la pieza que permite preservar stability cuando cada mitad ya conserva su orden relativo. Después del loop, una mitad está agotada; sus remanentes ya están ordenados.

El merge completo examina y copia O(n + m) elementos para longitudes `n` y `m`, con O(n + m) espacio en esta implementación.

### Precondition

Predice el resultado de `merge([3, 1], [2, 4])`. Si no queda ordenado, ¿falló el loop o el caller violó el contrato?

### Invariante

Después de agregar un elemento a `merged`, ¿por qué ningún elemento no consumido puede ser menor que el último agregado bajo la precondition?

---

## 16. Merge sort: recursion con trabajo real

Merge sort usa recursion para ordenar las dos mitades y `merge` para combinarlas:

```text
[4, 1, 3, 2]
├── [4, 1]
│   ├── [4]
│   └── [1]
│   └── merge → [1, 4]
└── [3, 2]
    ├── [3]
    └── [2]
    └── merge → [2, 3]
merge → [1, 2, 3, 4]
```

El base case `len(items) <= 1` es correcto porque una secuencia vacía o de un elemento ya está ordenada. Cada slice tiene longitud menor cuando `len(items) > 1`, así que existe progreso.

**Implementación educativa ejecutable:**

```python
def merge(left: list[int], right: list[int]) -> list[int]:
    merged: list[int] = []
    i = 0
    j = 0

    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            merged.append(left[i])
            i += 1
        else:
            merged.append(right[j])
            j += 1

    merged.extend(left[i:])
    merged.extend(right[j:])
    return merged


def merge_sort(items: list[int]) -> list[int]:
    if len(items) <= 1:
        return items.copy()

    midpoint = len(items) // 2
    left = merge_sort(items[:midpoint])
    right = merge_sort(items[midpoint:])
    return merge(left, right)


samples = [[], [8], [4, 1, 3, 2], [2, 2, 1], [5, 4, 3, 2, 1]]
for sample in samples:
    assert merge_sort(sample) == sorted(sample)

print(f"merge_cases={len(samples)}")
```

Output:

```text
merge_cases=5
```

### 16.1 Tiempo: O(n log n)

La recursion tree tiene aproximadamente `log2(n)` niveles de división. En cada nivel, los merges procesan en conjunto `n` elementos:

```text
log n niveles
×
n trabajo por nivel
=
O(n log n)
```

La recurrence simple que describe la idea es `T(n) = 2T(n/2) + O(n)`: dos mitades y un merge lineal. No la resolvemos con Master Theorem; agrupar el trabajo por niveles basta para este alcance.

### 16.2 Espacio

Esta versión crea slices y listas de merge. Su auxiliary space crece O(n), además de O(log n) frames en una ejecución balanceada. El detalle de asignaciones puede ser mayor por los slices temporales, pero la clase espacial relevante permanece lineal.

### 16.3 Stability

Para registros, la variante debe comparar keys y elegir el elemento izquierdo en empate. Cambiar `<=` por `<` selecciona primero el lado derecho cuando las keys son iguales y puede romper stability.

### Traza

Dibuja el call tree para `[3, 1, 2]`. Marca cada base case y el orden de los merges durante el unwind.

### Complejidad

¿Por qué no multiplicamos `n` por `n` si existen muchas llamadas? Agrupa el trabajo por nivel.

---

## 17. Comparar algoritmos desde workloads

| Estrategia | Tiempo relevante | Auxiliary space de la versión mostrada | Stable | Uso razonable |
|---|---:|---:|---|---|
| Selection sort | O(n²) best/worst | O(n) por copiar; core in-place posible | No | Aprender invariantes y swaps |
| Insertion sort | O(n) best, O(n²) worst | O(n) por copiar; core in-place posible | Sí | Inputs pequeños/casi ordenados; aprendizaje |
| Merge sort | O(n log n) | O(n) más call stack O(log n) | Sí con empate hacia izquierda | Divide and conquer; aprendizaje |
| `sorted()` | O(n log n) como cota general de trabajo comparativo | Nueva list O(n) | Sí | Default general de producción |
| `.sort()` | O(n log n) como cota general | Modifica list; usa memoria interna dependiente del input | Sí | Cuando mutar esa list es parte del contrato |

Python utiliza Timsort, un algoritmo híbrido stable y altamente optimizado que aprovecha orden existente. CS-M4 no estudia sus internals. La conclusión profesional es más importante:

> Los algoritmos manuales enseñan modelos; la biblioteca estándar suele implementar el trabajo de producción.

No uses la tabla como receta aislada. Evalúa source ownership, key cost, tamaño, patrón de orden previo, memoria y cantidad de operaciones.

### Decide

Tienes 40 elementos casi ordenados dentro de una práctica educativa. Luego tienes 2 millones de eventos en producto. ¿Qué cambia en la recomendación y por qué?

---

## 18. Sorting, indexing y costo de reordenar

Ordenar e indexar resuelven preguntas diferentes:

```text
timeline por (valid_time, event_id)
→ orden, rangos y replay determinista

events_by_id
→ equality lookup por ID
```

Pueden coexistir como derived views. Ninguna debe convertirse en autoridad persistente solo por estar optimizada.

Si llega un elemento nuevo, hacer `append + sort all` cuesta hasta O(n log n) por batch. `bisect` encuentra posición en O(log n), pero insertar en una `list` desplaza O(n). CS-M5 presentará estructuras ordered y heaps para otros workloads; aquí solo diagnosticamos el costo.

### Pipeline que exige una pregunta

```text
source
↓ filter O(n)
filtered
↓ sort O(m log m)
ordered
↓ binary search O(log m)
```

Si necesitas una sola coincidencia, quizá un scan con filtro O(n) sea más barato que ordenar. Si necesitas miles de range queries sobre la misma vista, el preprocessing puede amortizarse.

### Decide

¿Cuántas veces se reutiliza la vista ordenada? Sin esa respuesta, ¿qué comparación de costos queda incompleta?

---

## 19. Failure cases que parecen algoritmos válidos

Los fallos algorítmicos peligrosos no siempre lanzan una excepción. Conserva input, precondition, resultado observado y traza de límites al diagnosticarlos.

### 19.1 Binary search sobre datos desordenados

- **Síntoma:** encuentra algunos targets y omite otros presentes.
- **Causa:** usa una inferencia de orden que el input no satisface.
- **Corrección:** garantizar la vista ordenada por la misma key o elegir scan/índice.
- **Evidencia:** input original, target y secuencia `(low, high, mid)`.

### 19.2 Límites que no progresan

**Código incorrecto:**

```python
if items[mid] < target:
    low = mid  # puede repetir mid indefinidamente
```

- **Síntoma:** loop infinito para ciertos intervalos de dos elementos.
- **Causa:** el nuevo intervalo no es estrictamente menor.
- **Corrección:** con intervalo cerrado, `low = mid + 1`.
- **Evidencia:** convención de intervalo y traza de bounds.

### 19.3 Recursion sin progreso

- **Síntoma:** `RecursionError`.
- **Causa:** el argumento no se acerca al base case o el caso base excluye un boundary.
- **Corrección:** declarar una medida que disminuye; preferir loop si la depth lineal es grande.
- **Evidencia:** últimos argumentos distintos, no un traceback completo sin contexto.

### 19.4 Mutar el source al ordenar

**Código incorrecto para una fuente cuyo orden de llegada es evidencia:**

```python
events.sort(key=lambda event: event["valid_time"])
```

- **Síntoma:** se pierde el orden histórico original.
- **Causa:** `.sort()` modifica el mismo objeto referenciado por aliases.
- **Corrección:** `timeline = sorted(events, key=...)` y nombre explícito de derived view.
- **Evidencia:** source antes/después y ownership del objeto.

### 19.5 Benchmark con inputs distintos

Comparar insertion sort con una lista casi ordenada y selection sort con otra aleatoria no aísla el algoritmo. Genera un dataset base reproducible y entrega copias equivalentes a cada estrategia.

### 19.6 Selection sort “porque es simple”

La simplicidad importa, pero la standard library ya ofrece una interfaz más corta, probada y con mejor scaling. El algoritmo educativo solo gana si el objetivo explícito es aprender o estudiar una restricción especial demostrada.

### 19.7 Suponer `sorted()` O(n)

Recorrer `n` outputs no prueba que el trabajo total sea lineal. Usa O(n log n) como cota general aplicada; cualquier adaptación a runs del input es una propiedad de Timsort que no se asume sin medir/documentar.

### 19.8 Ignorar key cost

Si `expensive_key` hace O(n) trabajo por elemento, las `n` evaluaciones pueden contribuir O(n²) antes o además de las comparaciones. Aísla y mide esa función.

### 19.9 Stable input no determinista

Stability conserva el orden relativo recibido. Si ese orden cambia, el resultado empatado también. Define un tie-breaker de dominio o conserva una posición de source explícita; no atribuyas determinismo al algoritmo por sí solo.

### Detecta el bug

Clasifica cada caso anterior como violación de precondition, falta de progreso, error de ownership o benchmark inválido. Algunos pertenecen a más de una categoría; justifica.

---

## 20. Caso EIDOLON: timeline determinista y rango temporal

Usaremos eventos sintéticos pequeños. `source_events` conserva orden de llegada; `timeline` es una vista derivada ordenada por `(valid_time, event_id)`. Los datetimes son aware y compatibles.

**Ejemplo ejecutable:**

```python
from bisect import bisect_left, bisect_right
from datetime import UTC, datetime


source_events = [
    {
        "event_id": "evt-C",
        "valid_time": datetime(2026, 8, 26, 10, 0, tzinfo=UTC),
        "text": "third arrival",
    },
    {
        "event_id": "evt-B",
        "valid_time": datetime(2026, 8, 26, 9, 0, tzinfo=UTC),
        "text": "first at nine",
    },
    {
        "event_id": "evt-A",
        "valid_time": datetime(2026, 8, 26, 9, 0, tzinfo=UTC),
        "text": "second at nine",
    },
]

source_ids_before = [event["event_id"] for event in source_events]
timeline = sorted(
    source_events,
    key=lambda event: (event["valid_time"], event["event_id"]),
)

timeline_times = [event["valid_time"] for event in timeline]
start = datetime(2026, 8, 26, 9, 0, tzinfo=UTC)
end = datetime(2026, 8, 26, 9, 30, tzinfo=UTC)

left = bisect_left(timeline_times, start)
right = bisect_right(timeline_times, end)
window = timeline[left:right]

print(source_ids_before)
print([event["event_id"] for event in timeline])
print([event["event_id"] for event in window])
print(source_ids_before == [event["event_id"] for event in source_events])
```

Output:

```text
['evt-C', 'evt-B', 'evt-A']
['evt-A', 'evt-B', 'evt-C']
['evt-A', 'evt-B']
True
```

### 20.1 Qué demuestra

- la vista usa tiempo como primary key y `event_id` como tie-breaker;
- `sorted()` no cambia la list source;
- las fronteras temporales usan una lista de keys bajo el mismo orden;
- el rango es inclusivo en ambos extremos porque combina `bisect_left(start)` y `bisect_right(end)`;
- `timeline_times` también es derived data y cuesta O(n) materializarla en este ejemplo.

Un sistema que hace muchas range queries puede conservar keys alineadas con la timeline. Debe reconstruir ambas juntas y probar que no divergen. No introducimos database indexes ni estructuras ordered de CS-M5.

### 20.2 Lookup por ID sigue siendo otra pregunta

**Fragmento:**

```python
events_by_id = {event["event_id"]: event for event in source_events}
```

Buscar `evt-B` en ese índice es equality lookup esperado O(1). Hacer binary search por ID en una timeline ordenada por tiempo viola la precondition relativa a la key.

### Source vs derived

Si cambia la política de tie-break, ¿qué objeto se reconstruye y cuál no debe reescribirse?

### Precondition

¿Por qué `timeline_times` debe conservar exactamente el mismo orden y longitud que `timeline`?

---

## 21. Testing algorítmico por contratos y properties

Un ejemplo feliz no basta. Diseña casos desde fronteras y propiedades.

### 21.1 Matriz mínima

| Operación | Casos esenciales |
|---|---|
| Linear search | empty, one, first, last, missing, duplicates |
| Binary search | empty, one found/missing, first, last, middle, missing, duplicates bajo contrato “alguna” |
| Sorting | empty, one, duplicates, already sorted, reverse sorted, Unicode cuando la key lo requiera |
| Stable sorting | registros con key igual e identidades distintas |
| Timeline | timestamps aware iguales, tie-breaker, source sin mutar, replay repetido |

**Ejemplo ejecutable — tests de propiedades con stdlib:**

```python
from collections import Counter


def insertion_sort(items: list[int]) -> list[int]:
    result = items.copy()
    for index in range(1, len(result)):
        current = result[index]
        position = index
        while position > 0 and result[position - 1] > current:
            result[position] = result[position - 1]
            position -= 1
        result[position] = current
    return result


samples = [[], [1], [2, 1], [3, 1, 3, 2], [-1, 0, -1]]
for sample in samples:
    ordered = insertion_sort(sample)
    assert len(ordered) == len(sample)
    assert Counter(ordered) == Counter(sample)
    assert all(left <= right for left, right in zip(ordered, ordered[1:]))
    assert insertion_sort(ordered) == ordered

print("sorting_properties=PASS")
```

Output:

```text
sorting_properties=PASS
```

Las properties son independientes del detalle principal:

- misma longitud;
- mismo multiset, medido con `Counter`;
- output no decreciente;
- idempotence.

Comparar únicamente con otra copia del mismo algoritmo puede duplicar el mismo bug.

### 21.2 Property-based testing con Hypothesis

PF-M9 ya introdujo Hypothesis. Si está declarado en las optional development dependencies del proyecto, una prueba acotada puede cubrir muchos inputs:

**Archivo de test ejecutable con esa precondition ambiental:**

```python
from collections import Counter

from hypothesis import given, strategies as st


@given(st.lists(st.integers(), max_size=100))
def test_merge_sort_properties(items: list[int]) -> None:
    ordered = merge_sort(items)

    assert Counter(ordered) == Counter(items)
    assert all(a <= b for a, b in zip(ordered, ordered[1:]))
    assert merge_sort(ordered) == ordered
```

Este archivo reutiliza `merge_sort` del código bajo prueba. Hypothesis genera ejemplos y reduce failures; no demuestra formalmente la ausencia de bugs.

### 21.3 Property de binary search

**Ejemplo ejecutable:**

```python
def binary_search(items: list[int], target: int) -> int | None:
    low = 0
    high = len(items) - 1
    while low <= high:
        mid = (low + high) // 2
        if items[mid] == target:
            return mid
        if items[mid] < target:
            low = mid + 1
        else:
            high = mid - 1
    return None


items = [-3, -1, 0, 0, 4, 8]
for target in range(-5, 11):
    index = binary_search(items, target)
    if target in items:
        assert index is not None and items[index] == target
    else:
        assert index is None

print("search_properties=PASS")
```

Output:

```text
search_properties=PASS
```

La property no exige cuál índice retornar entre duplicates; respeta el contrato “alguna coincidencia”.

### Diseña la property

Formula una prueba de stability para registros `(key, arrival_id)` sin limitarte a comparar con la misma implementación.

### Detecta el test débil

¿Por qué probar solo que `len(sort(items)) == len(items)` permite perder un valor y duplicar otro?

---

## 22. Benchmark de estrategias de búsqueda

Separa cuatro cantidades:

```text
generación del dataset
preprocessing
query cost
memoria derivada
```

El siguiente benchmark mide preprocessing y queries en regiones distintas. Los tiempos dependen de la máquina; solo el número de mediciones y su no negatividad son outputs estables.

**Benchmark ejecutable:**

```python
from bisect import bisect_left
from random import Random
from time import perf_counter


def linear_contains(items: list[int], target: int) -> bool:
    return any(item == target for item in items)


def binary_contains(items: list[int], target: int) -> bool:
    index = bisect_left(items, target)
    return index < len(items) and items[index] == target


def measure_search(n: int, query_count: int) -> dict[str, float]:
    rng = Random(20260826 + n)
    source = list(range(n))
    rng.shuffle(source)
    targets = [rng.randrange(n * 2) for _ in range(query_count)]

    started = perf_counter()
    ordered = sorted(source)
    sort_seconds = perf_counter() - started

    started = perf_counter()
    index = {value: True for value in source}
    index_seconds = perf_counter() - started

    expected = [target in index for target in targets]

    started = perf_counter()
    linear_results = [linear_contains(source, target) for target in targets]
    linear_seconds = perf_counter() - started

    started = perf_counter()
    binary_results = [binary_contains(ordered, target) for target in targets]
    binary_seconds = perf_counter() - started

    started = perf_counter()
    dict_results = [target in index for target in targets]
    dict_seconds = perf_counter() - started

    assert linear_results == binary_results == dict_results == expected
    return {
        "sort_preprocessing": sort_seconds,
        "dict_preprocessing": index_seconds,
        "linear_queries": linear_seconds,
        "binary_queries": binary_seconds,
        "dict_queries": dict_seconds,
    }


rows = [measure_search(n, query_count=200) for n in (100, 1_000, 10_000)]
measurements = [value for row in rows for value in row.values()]
print(f"measurements={len(measurements)}")
print(f"all_non_negative={all(value >= 0 for value in measurements)}")
```

Output estable:

```text
measurements=15
all_non_negative=True
```

No sumes preprocessing a cada query si la vista se reutiliza. Tampoco lo omitas si la pregunta end-to-end es “obtener código source no preparado y contestar una sola consulta”. Ejecuta múltiples repeticiones antes de comparar cifras pequeñas, como enseñó CS-M1.

### Benchmark

Modifica `query_count` con `1`, `10`, `100` y `10_000`. ¿Dónde aparece un break-even local? No generalices el valor a otra máquina o distribución.

### Decide

¿Qué estrategia elegirías si source cambia después de cada consulta? Añade el costo de reconstruir o mantener cada vista.

---

## 23. Benchmark de sorting con inputs equivalentes

Cada algoritmo recibe el mismo dataset lógico. Las copias se realizan fuera de la región si la función ya copia internamente; documenta esa convención. Los tamaños son deliberadamente modestos para no convertir O(n²) en una espera innecesaria.

**Benchmark ejecutable:**

```python
from random import Random
from time import perf_counter


def selection_sort(items: list[int]) -> list[int]:
    result = items.copy()
    for start in range(len(result)):
        min_index = start
        for index in range(start + 1, len(result)):
            if result[index] < result[min_index]:
                min_index = index
        result[start], result[min_index] = result[min_index], result[start]
    return result


def insertion_sort(items: list[int]) -> list[int]:
    result = items.copy()
    for index in range(1, len(result)):
        current = result[index]
        position = index
        while position > 0 and result[position - 1] > current:
            result[position] = result[position - 1]
            position -= 1
        result[position] = current
    return result


def merge_sort(items: list[int]) -> list[int]:
    if len(items) <= 1:
        return items.copy()
    midpoint = len(items) // 2
    left = merge_sort(items[:midpoint])
    right = merge_sort(items[midpoint:])
    merged: list[int] = []
    i = 0
    j = 0
    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            merged.append(left[i])
            i += 1
        else:
            merged.append(right[j])
            j += 1
    merged.extend(left[i:])
    merged.extend(right[j:])
    return merged


algorithms = {
    "selection": selection_sort,
    "insertion": insertion_sort,
    "merge": merge_sort,
    "built_in": sorted,
}

rng = Random(42)
measurements: list[float] = []
for n in (20, 100, 400):
    source = [rng.randrange(n * 2) for _ in range(n)]
    source_snapshot = source.copy()
    expected = sorted(source)
    for algorithm in algorithms.values():
        started = perf_counter()
        result = algorithm(source)
        measurements.append(perf_counter() - started)
        assert result == expected
        assert source == source_snapshot

print(f"measurements={len(measurements)}")
print(f"all_non_negative={all(value >= 0 for value in measurements)}")
```

Output estable:

```text
measurements=12
all_non_negative=True
```

Una sola medición no establece una curva. Repite, conserva muestras/mediana y prueba familias de input: aleatorio, ordenado, inverso y casi ordenado. Interpreta crecimiento, no una victoria por microsegundos.

### Benchmark

Formula antes de medir: ¿qué algoritmo debería mejorar relativamente al pasar de `n=100` a `n=1_000`? ¿Qué patrón debería favorecer a insertion sort?

### Detecta el benchmark inválido

Un estudiante ejecuta `.sort()` sobre la misma list cuatro veces seguidas para comparar algoritmos. ¿Qué algoritmo recibe un input distinto a partir de la segunda ejecución?

---

## 24. Caso progresivo integrado: search + sort sin perder source

La decisión completa combina preguntas distintas:

```text
source_events
├── linear search ocasional por text/tag
├── events_by_id para muchas equality queries
└── timeline para orden/rangos temporales
```

Contrato mínimo:

1. `source_events` conserva orden y objetos originales.
2. `events_by_id` falla al construirse si hay IDs duplicados; no sobrescribe evidencia silenciosamente.
3. `timeline` usa datetimes aware y tie-breaker explícito.
4. `timeline_times` se alinea uno a uno con la timeline.
5. cualquier range query documenta inclusión/exclusión de endpoints.
6. replay del mismo source produce los mismos IDs ordenados.

**Ejemplo ejecutable:**

```python
from bisect import bisect_left, bisect_right
from dataclasses import dataclass
from datetime import UTC, datetime


@dataclass(frozen=True)
class Event:
    event_id: str
    valid_time: datetime


def build_index(events: list[Event]) -> dict[str, Event]:
    index: dict[str, Event] = {}
    for event in events:
        if event.event_id in index:
            raise ValueError(f"duplicate event_id: {event.event_id}")
        index[event.event_id] = event
    return index


def build_timeline(events: list[Event]) -> list[Event]:
    return sorted(
        events,
        key=lambda event: (event.valid_time, event.event_id),
    )


events = [
    Event("evt-2", datetime(2026, 8, 26, 12, tzinfo=UTC)),
    Event("evt-1", datetime(2026, 8, 26, 11, tzinfo=UTC)),
    Event("evt-3", datetime(2026, 8, 26, 12, tzinfo=UTC)),
]
source_order = [event.event_id for event in events]
index = build_index(events)
timeline = build_timeline(events)
times = [event.valid_time for event in timeline]

start = datetime(2026, 8, 26, 12, tzinfo=UTC)
left = bisect_left(times, start)
right = bisect_right(times, start)

assert index["evt-1"] is events[1]
assert [event.event_id for event in timeline[left:right]] == ["evt-2", "evt-3"]
assert [event.event_id for event in events] == source_order
assert build_timeline(events) == timeline
print("derived_views=PASS")
```

Output:

```text
derived_views=PASS
```

Los asserts verifican identidad compartida de los registros sin presentar el índice como source. En una fase futura, persistencia y snapshots requerirán contratos adicionales; no se introducen aquí.

### Explica

¿Por qué `index["evt-1"] is events[1]` no convierte el `dict` en source of truth? ¿Qué riesgo de aliasing sí debes reconocer?

### Modifica

Cambia la range query a intervalo semiabierto `[start, end)`. Decide qué combinación de `bisect_left`/`bisect_right` expresa exactamente ese contrato.

---

## 25. Ejercicios guiados con solución razonada

Resuelve cada predicción antes de leer la solución. Los criterios distinguen correctness de coincidencias accidentales.

### 25.1 Trazar recursion

**Objetivo.** Separar descenso y unwind.  
**Input.** `product_to(3)`, donde el base case `product_to(0)` retorna `1` y el recursive case retorna `n * product_to(n - 1)`.  
**Predice.** Lista llamadas y multiplicaciones.

**Solución ejecutable:**

```python
def product_to(n: int) -> int:
    if n == 0:
        return 1
    return n * product_to(n - 1)


assert product_to(3) == 6
assert product_to(0) == 1
```

**Razonamiento.** Desciende `3 → 2 → 1 → 0`; retorna `1 → 1 → 2 → 6`. El uno es identidad multiplicativa.  
**Criterio.** La traza debe mostrar que el cálculo final ocurre durante unwind.  
**Variación.** Predice `product_to(1)`.

### 25.2 Corregir un base case

**Objetivo.** Cubrir el boundary vacío.  
**Input.** Una función que retorna el primer elemento o llama con `items[1:]`, pero solo detiene en `len(items) == 1`.  
**Predice.** ¿Qué ocurre con `[]`?

**Solución ejecutable:**

```python
def count_items(items: list[object]) -> int:
    if not items:
        return 0
    return 1 + count_items(items[1:])


assert count_items([]) == 0
assert count_items(["a", "b"]) == 2
```

**Razonamiento.** Empty es el estado terminal natural; cada slice reduce longitud. Esta versión es pedagógica: `items[1:]` copia en cada llamada y acumula trabajo/allocations cuadráticos. El ejercicio siguiente usa un índice para aislar el costo de call stack.  
**Criterio.** Deben demostrarse base case y progreso, no solo el output para dos elementos.  
**Variación.** Compara su espacio con `len(items)` y con un loop.

### 25.3 Convertir loop a recursion y comparar

**Objetivo.** No confundir equivalencia de resultado con equivalencia de recursos.  
**Input.** Contar apariciones de un tag.  
**Predice.** ¿Cuál versión soporta una list mucho más profunda sin call stack por elemento?

**Solución ejecutable:**

```python
def count_tag_recursive(tags: list[str], target: str, index: int = 0) -> int:
    if index == len(tags):
        return 0
    found = 1 if tags[index] == target else 0
    return found + count_tag_recursive(tags, target, index + 1)


def count_tag_iterative(tags: list[str], target: str) -> int:
    count = 0
    for tag in tags:
        if tag == target:
            count += 1
    return count


tags = ["home", "work", "home"]
assert count_tag_recursive(tags, "home") == 2
assert count_tag_iterative(tags, "home") == 2
```

**Razonamiento.** Ambas hacen O(n) comparaciones; la recursiva añade O(n) frames.  
**Criterio.** La recomendación iterativa debe justificarse por depth, no porque recursion sea “mala”.  
**Variación.** Traza el índice para input vacío.

### 25.4 Implementar linear search

**Objetivo.** Hacer explícita la ausencia.  
**Input.** IDs con duplicates.  
**Predice.** ¿Qué índice representa “primera coincidencia”?

**Solución ejecutable:**

```python
def find_first(items: list[str], target: str) -> int | None:
    for index, item in enumerate(items):
        if item == target:
            return index
    return None


assert find_first(["a", "b", "a"], "a") == 0
assert find_first(["a"], "z") is None
```

**Razonamiento.** El retorno temprano preserva el primer índice; terminar el loop justifica ausencia.  
**Criterio.** Debe diferenciar índice `0` de `None`; usar truthiness sobre el resultado sería incorrecto.  
**Variación.** Retorna el objeto en vez del índice y documenta el nuevo contrato.

### 25.5 Derivar el costo del scan

**Objetivo.** Separar best/worst de average.  
**Input.** `n` elementos y target primero, último o ausente.  
**Predice.** Comparaciones exactas.

**Solución razonada:**

```text
primero  → 1 comparación → O(1)
último   → n comparaciones → O(n)
ausente  → n comparaciones → O(n)
```

No hay average único sin probabilidades de presencia y posición.  
**Criterio.** El argumento nombra operación y supuesto.  
**Variación.** ¿Qué cambia si se buscan todas las coincidencias?

### 25.6 Trazar binary search

**Objetivo.** Mantener un intervalo cerrado.  
**Input.** `[2, 4, 6, 8, 10, 12, 14]`, target `12`.  
**Predice.** Bounds y midpoints.

**Solución razonada:**

```text
low=0 high=6 mid=3 value=8  → low=4
low=4 high=6 mid=5 value=12 → found index 5
```

**Criterio.** La traza descarta `mid` junto con la mitad imposible.  
**Variación.** Traza target `13` hasta `low > high`.

### 25.7 Corregir off-by-one

**Objetivo.** Alinear condición e intervalo.  
**Input.** `high = len(items) - 1` con `while low < high`.  
**Predice.** Qué boundary queda sin examinar.

**Solución ejecutable:**

```python
def contains_sorted(items: list[int], target: int) -> bool:
    low = 0
    high = len(items) - 1
    while low <= high:
        mid = (low + high) // 2
        if items[mid] == target:
            return True
        if items[mid] < target:
            low = mid + 1
        else:
            high = mid - 1
    return False


assert contains_sorted([7], 7)
assert not contains_sorted([], 7)
```

**Razonamiento.** `low == high` representa un candidato pendiente en un intervalo cerrado.  
**Criterio.** Empty y one-element deben pasar.  
**Variación.** Diseña conscientemente una versión semiabierta `[low, high)`.

### 25.8 Implementar selection sort

**Objetivo.** Preservar el prefijo final.  
**Input.** `[3, 1, 2]`.  
**Predice.** `min_index` en cada iteración externa.

**Solución.** Usa `selection_sort` de la sección 13 y comprueba.

**Fragmento — requiere esa implementación:**

```python
source = [3, 1, 2]
assert selection_sort(source) == [1, 2, 3]
assert source == [3, 1, 2]
```

**Razonamiento.** Buscar mínimo y hacer un swap extiende el prefijo correcto en uno.  
**Criterio.** Resultado ordenado y multiset conservado; un único ejemplo no basta en la suite final.  
**Variación.** Cuenta comparaciones para `n=3` y `n=4`.

### 25.9 Implementar insertion sort

**Objetivo.** Mantener un prefijo ordenado.  
**Input.** `[1, 3, 2, 4]`.  
**Predice.** Cuántos desplazamientos ocurren.

**Solución.** Usa `insertion_sort` de la sección 14; solo `3` se desplaza al insertar `2`.

**Fragmento — requiere esa implementación:**

```python
assert insertion_sort([1, 3, 2, 4]) == [1, 2, 3, 4]
```

**Razonamiento.** La comparación estricta detiene al encontrar un elemento `<= current`.  
**Criterio.** Debe explicarse por qué el prefijo sigue ordenado.  
**Variación.** Mide desplazamientos en input inverso.

### 25.10 Merge de dos listas

**Objetivo.** Aplicar la invariante del output parcial.  
**Input.** `[1, 5, 8]` y `[2, 3, 9]`.  
**Predice.** Orden de consumo de lados.

**Fragmento — requiere `merge` de la sección 15:**

```python
assert merge([1, 5, 8], [2, 3, 9]) == [1, 2, 3, 5, 8, 9]
```

**Razonamiento.** Cada front es el mínimo restante de su propia lista; el menor de ambos es el mínimo global restante.  
**Criterio.** También debe funcionar con un lado vacío.  
**Variación.** Usa duplicates y explica el empate izquierdo.

### 25.11 Implementar merge sort

**Objetivo.** Conectar base, progreso y combine.  
**Input.** `[4, 1, 3, 2]`.  
**Predice.** Leaves y orden de merges.

**Solución.** Usa `merge_sort` de la sección 16.

**Fragmento — requiere esa implementación:**

```python
source = [4, 1, 3, 2]
assert merge_sort(source) == [1, 2, 3, 4]
assert source == [4, 1, 3, 2]
```

**Razonamiento.** Slices menores garantizan progreso; `merge` combina resultados bajo su precondition.  
**Criterio.** Empty, one y duplicates también pasan.  
**Variación.** Cuenta niveles para ocho elementos.

### 25.12 Demostrar stability

**Objetivo.** Observar identidad relativa con keys iguales.  
**Input.** `[(2, "first"), (1, "middle"), (2, "last")]`.  
**Predice.** Orden de las etiquetas tras stable sort por la primera posición.

**Solución ejecutable:**

```python
records = [(2, "first"), (1, "middle"), (2, "last")]
ordered = sorted(records, key=lambda record: record[0])
assert ordered == [(1, "middle"), (2, "first"), (2, "last")]
```

**Razonamiento.** Los registros con key `2` conservan `first` antes de `last`.  
**Criterio.** Comparar solo las keys `[1, 2, 2]` no demostraría stability.  
**Variación.** Ejecuta la selection sort no stable de la sección 13.

### 25.13 Comparar algoritmos

**Objetivo.** Elegir desde input y contrato.  
**Input.** 50 valores casi ordenados frente a 50 000 aleatorios.  
**Predice.** Qué algoritmos educativos podrían mostrar diferencias de crecimiento.

**Solución razonada.** Insertion puede aprovechar el primer patrón; selection mantiene O(n²); merge permanece O(n log n). En producción, empieza con `sorted()` para ambos y mide solo si existe un problema demostrado.  
**Criterio.** La respuesta separa aprendizaje de recomendación de producto.  
**Variación.** Añade restricción severa de memoria y vuelve a discutir, sin inventar otra estructura.

### 25.14 Usar `sorted(key=...)`

**Objetivo.** Definir orden total reproducible.  
**Input.** Eventos con `valid_time` empatado.  
**Predice.** Resultado con key temporal sola y con tuple `(time, id)`.

**Solución ejecutable:**

```python
events = [
    {"id": "B", "time": 1},
    {"id": "A", "time": 1},
    {"id": "C", "time": 2},
]
ordered = sorted(events, key=lambda event: (event["time"], event["id"]))
assert [event["id"] for event in ordered] == ["A", "B", "C"]
```

**Razonamiento.** Tuple comparison aplica primary y secondary keys en orden.  
**Criterio.** El source sigue en orden `B, A, C`.  
**Variación.** Ordena descending solo por tiempo y documenta el desempate esperado.

### 25.15 Benchmark de search

**Objetivo.** Separar setup de queries.  
**Input.** `n=10_000`, `q=1` y `q=10_000`.  
**Predice.** Qué costo domina en cada escenario.

**Solución razonada.** Usa `measure_search` de la sección 22. Reporta `sort_preprocessing`, `dict_preprocessing` y cada lote de queries por separado; luego calcula total según cuántas veces se reutiliza la vista.  
**Criterio.** No compara solo binary query con scan mientras oculta sorting para una consulta.  
**Variación.** Incluye memoria derivada como otra métrica.

### 25.16 Benchmark de sorting

**Objetivo.** Mantener inputs equivalentes y múltiples tamaños.  
**Input.** Aleatorio, ordenado e inverso para `n=20, 100, 400`.  
**Predice.** Patrón de insertion y selection.

**Solución razonada.** Genera una base por caso, pasa la misma secuencia lógica a todas las funciones, valida cada output fuera o dentro de la región con criterio documentado y repite. Selection no mejora asintóticamente con orden previo; insertion sí reduce desplazamientos/comparaciones del `while`.  
**Criterio.** Conserva muestras, Python/version y setup.  
**Variación.** Añade input con muchos duplicates.

### 25.17 Documentar el tradeoff

**Objetivo.** Convertir evidencia en decisión revisable.  
**Input.** 100 eventos, una query hoy; 100 000 eventos y 20 000 queries proyectadas.  
**Predice.** Estrategia inicial y migration trigger.

**Solución razonada:**

```text
decisión actual: linear scan por simplicidad para n≈100, q≈1
trigger: medir índice cuando q·n exceda el budget observado
alternativa: dict por ID; timeline separada solo si aparecen range queries
evidencia faltante: frecuencia real, memoria y actualización
```

**Criterio.** La decisión declara workload, costos incluidos y condición para revisarla.  
**Variación.** Añade source que cambia después de cada operación.

---

## 26. Ejercicios independientes

No consultes una solución mientras trabajas. Conserva código, trazas, hipótesis y evidencia.

### Fundamentos

1. Traza descenso y unwind de una suma recursiva sobre `[4, 2, 1]`.
2. Corrige una recursion cuyo base case existe pero cuya medida no disminuye.
3. Diseña un base case para encontrar el máximo de una secuencia no vacía; declara esa precondition.
4. Convierte una recursion lineal de conteo en loop y compara auxiliary space.
5. Explica por qué tail recursion no elimina depth en Python.
6. Provoca y captura `RecursionError` con un ejemplo acotado; no cambies el recursion limit.
7. Dibuja frames conceptuales para tres llamadas y marca sus return points.

### Search

8. Implementa linear search que retorne la primera coincidencia por `event_id`.
9. Cambia el contrato para retornar todas las coincidencias y deriva el costo.
10. Implementa binary search iterativo con intervalo cerrado.
11. Implementa conscientemente una versión semiabierta y documenta cada update.
12. Escribe tests para empty, one, first, last y missing.
13. Encuentra el input mínimo que rompe un `while low < high` mal diseñado.
14. Demuestra un target presente que binary search omite en datos desordenados.
15. Implementa binary search recursivo y mide su depth lógica.
16. Prueba duplicates sin exigir un índice específico cuando el contrato dice “alguna”.
17. Usa `bisect_left`/`bisect_right` para contar un valor.
18. Explica por qué localizar e insertar en `list` sigue siendo O(n) completo.
19. Compara una query mediante scan, sort+binary y dict con preprocessing incluido.

### Sorting

20. Traza selection sort sobre cuatro valores y verifica el prefijo final.
21. Cuenta sus comparaciones y deriva `n(n-1)/2`.
22. Construye otro input que demuestre que la implementación mostrada no es stable.
23. Traza insertion sort sobre input casi ordenado y cuenta desplazamientos.
24. Cambia `>` por `>=` para registros con key y reproduce la pérdida de stability.
25. Implementa `merge` y prueba ambos lados vacíos por separado.
26. Traza el recursion tree de merge sort con cinco elementos.
27. Explica `n` trabajo por nivel aun cuando los subarrays cambian de tamaño.
28. Mide auxiliary space de forma prudente; distingue retained de peak allocations.
29. Verifica misma longitud, multiset, orden e idempotence para los tres sorts.
30. Escribe una property Hypothesis para merge sort con `max_size` acotado.
31. Diseña una prueba de stability con identidades visibles.
32. Compara `sorted()` y `.sort()` en presencia de un alias a la list.
33. Ordena strings y explica por qué el resultado no afirma collation española.
34. Reproduce el `TypeError` al mezclar naive y aware datetimes y corrige el contrato.
35. Diseña una key costosa, aísla su costo y propone precálculo si la semántica lo permite.

### EIDOLON y medición

36. Construye una timeline derivada con `(valid_time, event_id)` sin mutar source.
37. Cambia el orden de llegada y demuestra que el tie-breaker conserva replay determinista.
38. Implementa un rango inclusivo y otro semiabierto con `bisect`.
39. Construye `events_by_id` y rechaza IDs duplicados sin overwrite silencioso.
40. Demuestra por qué la timeline temporal no soporta binary search por ID.
41. Mide scan, binary y dict en `n` y `q` crecientes; separa preprocessing.
42. Mide selection, insertion, merge y built-in con inputs equivalentes.
43. Repite cada benchmark y reporta mediana, no solo la mejor muestra.
44. Introduce accidentalmente generación dentro de la región medida; diagnostica el cambio de pregunta.
45. Documenta un break-even local y una limitación que impida generalizarlo.
46. Decide entre source order y derived order cuando ambos tienen significado distinto.
47. Propón una estrategia para batch estático y otra para updates frecuentes sin introducir CS-M5.
48. Revisa un diseño que ordena una sola vez para una sola query y cuantifica el trabajo evitable.

---

## 27. Preguntas conceptuales

1. ¿Qué tres elementos necesita una recursion correcta y por qué ninguno sustituye a los otros?
2. ¿Cómo puede una función tener base case y aun así no terminar?
3. ¿Qué información conceptual conserva cada frame del call stack?
4. ¿Cuándo preferirías iteración aunque la definición recursiva sea compacta?
5. ¿Por qué cambiar el recursion limit no corrige una transición sin progreso?
6. ¿Qué contrato de ausencia usa tu linear search y por qué importa?
7. ¿Qué invariante justifica retornar la primera coincidencia?
8. ¿Por qué no existe un average case universal para linear search?
9. ¿Qué precondition necesita binary search?
10. ¿Por qué ordenar por timestamp no permite binary search por ID?
11. ¿Qué significa que `[low, high]` sea un intervalo cerrado?
12. ¿Cómo un off-by-one puede omitir el único elemento pendiente?
13. ¿Por qué binary search es O(log n)?
14. ¿Qué space tradeoff distingue las versiones iterativa y recursiva?
15. ¿Por qué una coincidencia con duplicates no identifica necesariamente la primera?
16. ¿Qué encuentra `bisect_left` y qué costo no elimina al insertar en list?
17. ¿Cuándo linear search supera conceptualmente a sort+binary search?
18. ¿Por qué un `dict` por ID y una timeline ordenada pueden coexistir?
19. ¿Qué significa que un sorting sea stable?
20. ¿Por qué stability no garantiza deterministic output por sí sola?
21. ¿Qué diferencia existe entre in-place y out-of-place bajo aliasing?
22. ¿Qué invariante mantiene selection sort?
23. ¿Por qué su número de comparaciones es O(n²) incluso sobre input ordenado?
24. ¿Qué comparación hace stable a la insertion sort mostrada?
25. ¿Qué precondition recibe `merge`?
26. ¿Por qué elegir el lado izquierdo en empate conserva stability?
27. ¿Por qué merge sort usa O(n log n) time y O(n) auxiliary space en esta versión?
28. ¿Cuándo insertion sort puede ser razonable sin reemplazar el built-in por defecto?
29. ¿Por qué selection sort normalmente no pertenece a producción?
30. ¿Qué significa que Python use un sort stable optimizado sin estudiar Timsort internals?
31. ¿Cómo puede el costo de `key` dominar el sorting?
32. ¿Por qué code-point order no equivale a collation lingüística?
33. ¿Qué evidencia destruyes al ordenar destructivamente un source order significativo?
34. ¿Cómo separamos preprocessing cost de query cost?
35. ¿Por qué inputs distintos invalidan una comparación de algoritmos?
36. ¿Qué properties de sorting detectan pérdida, duplicación y desorden?
37. ¿Qué property de binary search respeta el contrato “alguna coincidencia”?
38. ¿Cómo combinarías timeline ordenada e índice `dict` en EIDOLON?

---

## 28. Mini challenge — Timeline search reproducible

### Objetivo y artefacto

Construye un pequeño paquete o script autocontenido que compare search y sorting sobre eventos sintéticos sin mutar la fuente. Debe ser resoluble únicamente con Programming Foundations + CS-M1–CS-M4.

Entrega:

```text
timeline_challenge/
├── algorithms.py
├── timeline.py
├── benchmark.py
├── test_timeline.py
└── decision.md
```

Puedes mantenerlo en un solo archivo si el objetivo del laboratorio no requiere packaging; PF-M4 ya permite separarlo sin introducir arquitectura nueva.

### Input reproducible

Usa al menos 12 eventos con:

- `event_id` único;
- `valid_time` timezone-aware;
- dos o más timestamps empatados;
- orden de llegada diferente del temporal;
- una key secundaria determinista;
- un target presente y uno ausente.

No uses archivos, database ni red: los eventos viven in-memory.

### Requisitos A — Search

1. Implementa linear search por una key y define ausencia.
2. Implementa binary search iterativo sobre una vista ordenada por esa misma key.
3. Construye un índice `dict` por `event_id` sin overwrite silencioso.
4. Compara preprocessing + una query y preprocessing + muchas queries.
5. Explica por qué binary search sobre la timeline temporal no reemplaza al índice por ID.

### Requisitos B — Sorting

6. Implementa selection sort, insertion sort y merge sort educativos.
7. No muta el input en ninguno de los tres contratos.
8. Comprueba empty, one, duplicates, sorted y reverse sorted.
9. Verifica stability solo para las implementaciones que realmente la prometen.
10. Compara correctness con `sorted()` sin usarlo dentro de tus algoritmos.

### Requisitos C — Recursion

11. Usa recursion dentro de merge sort.
12. Dibuja el call tree para cuatro eventos.
13. Identifica base case, progreso, depth O(log n) y trabajo por nivel.
14. Explica por qué no usarías recursion lineal para millones de eventos.

### Requisitos D — Timeline determinista

15. Conserva `source_events` sin mutar.
16. Construye una vista con key `(valid_time, event_id)` o un tie-breaker igualmente explícito y justificado.
17. Implementa una range query con fronteras y endpoints documentados.
18. Ejecuta el build de timeline dos veces desde el mismo source y compara resultados.
19. Reordena el input de forma reproducible y demuestra qué parte del tie-breaker mantiene el mismo orden total.

### Requisitos E — Medición y decisión

20. Benchmarkea linear, binary y dict con datos/targets idénticos.
21. Reporta preprocessing y query cost por separado.
22. Benchmarkea selection, insertion, merge y `sorted()` con tamaños modestos y múltiples repeticiones.
23. Explica time y auxiliary space de cada estrategia.
24. En `decision.md`, elige estrategias para:
    - una consulta sobre un batch pequeño;
    - muchas consultas por ID;
    - muchas range queries sobre un batch estable;
    - updates frecuentes que obligan a reconstruir vistas.

### Contrato de comprobación

Adapta nombres si es necesario, pero conserva propiedades equivalentes.

**Continuación contractual — depende de tus implementaciones y del modelo `Event` ya aprendido en PF-M5:**

```python
from collections import Counter


source_snapshot = tuple(event.event_id for event in source_events)
timeline_a = build_timeline(source_events)
timeline_b = build_timeline(source_events)

assert tuple(event.event_id for event in source_events) == source_snapshot
assert [event.event_id for event in timeline_a] == [
    event.event_id for event in timeline_b
]
assert all(
    (left.valid_time, left.event_id) <= (right.valid_time, right.event_id)
    for left, right in zip(timeline_a, timeline_a[1:])
)

for algorithm in (selection_sort, insertion_sort, merge_sort):
    sample = [4, 1, 3, 1, 2]
    output = algorithm(sample)
    assert output == sorted(sample)
    assert Counter(output) == Counter(sample)
    assert sample == [4, 1, 3, 1, 2]

for event in timeline_a:
    index = binary_search_by_time(timeline_a, event.valid_time)
    assert index is not None
    assert timeline_a[index].valid_time == event.valid_time

assert linear_search_by_id(source_events, "missing") is None
assert binary_search_by_time([], target_time) is None
```

### Failure cases deliberados

Diagnostica y corrige:

1. binary search sobre `source_events` desordenado;
2. `low = mid` que deja de progresar;
3. merge que elige primero el lado derecho en empate y rompe stability;
4. `.sort()` aplicado al source;
5. ID duplicado que el `dict` sobrescribe;
6. benchmark de algoritmos con datasets distintos;
7. resultado stable pero no determinista por falta de tie-breaker;
8. mezcla de datetime naive y aware.

### Criterio de aprobación

El challenge queda completo cuando:

- todos los asserts y edge cases pasan;
- las trazas justifican base/progreso/invariantes;
- source y derived views quedan diferenciados;
- la timeline es determinista bajo la política declarada;
- los benchmarks separan setup y queries y usan inputs equivalentes;
- `decision.md` relaciona workload, time, space, preconditions y evidencia;
- no se introducen trees, heaps, graphs, concurrency, database, backend ni AI.

---

## 29. Resumen

- Recursion correcta requiere base case, recursive case y progreso.
- Cada llamada añade un frame conceptual; Python tiene depth limitada y no optimiza tail calls de forma general.
- Linear search no exige orden: best O(1), worst O(n).
- Binary search exige orden por la misma key y reduce el espacio a la mitad: O(log n).
- Duplicates exigen distinguir cualquier coincidencia de fronteras.
- `bisect` localiza en O(log n), pero insertar en `list` sigue costando O(n).
- Selection sort mostrado es O(n²) y no stable.
- Insertion sort mostrado es stable, O(n) best y O(n²) worst.
- Merge sort usa divide and conquer, O(n log n) time y O(n) auxiliary space en la versión mostrada.
- Stability conserva orden relativo; un tie-breaker explícito aporta determinismo cuando el source order no basta.
- `sorted()` crea una list; `.sort()` muta la existente. Ambas son stable y normalmente preferibles en producción.
- Sorting, índice por ID y source son responsabilidades distintas.
- Un benchmark válido separa preprocessing de query cost y compara inputs equivalentes.

---

## 30. Checklist de dominio

- [ ] Puedo identificar base case, recursive case y medida de progreso.
- [ ] Puedo trazar descenso y unwind sin ejecutar.
- [ ] Puedo explicar call stack y riesgo de recursion depth.
- [ ] Puedo elegir iteración sin afirmar que recursion siempre sea inferior.
- [ ] Puedo implementar linear search con ausencia explícita.
- [ ] Puedo derivar best/worst sin inventar average case.
- [ ] Puedo declarar la precondition de binary search.
- [ ] Puedo implementar y probar binary search iterativo y recursivo.
- [ ] Puedo detectar bounds mezclados y updates sin progreso.
- [ ] Puedo explicar O(log n) mediante reducciones a la mitad.
- [ ] Puedo distinguir coincidencia, frontera e insertion point.
- [ ] Puedo explicar el costo completo de insertar con `bisect` en una list.
- [ ] Puedo elegir scan, binary o dict desde `n`, `q`, updates y memoria.
- [ ] Puedo definir key, dirección, stability y tie-breaker.
- [ ] Puedo distinguir in-place de out-of-place bajo aliasing.
- [ ] Puedo implementar selection, insertion, merge y merge sort.
- [ ] Puedo declarar invariantes y complejidades de esos algoritmos.
- [ ] Puedo demostrar stability con identidades, no solo con keys.
- [ ] Puedo explicar O(n log n) y auxiliary space de merge sort.
- [ ] Puedo usar `sorted()`/`.sort()` de acuerdo con ownership.
- [ ] Puedo evitar confundir code-point ordering con collation lingüística.
- [ ] Puedo ordenar timezone-aware datetimes compatibles.
- [ ] Puedo escribir properties de sorting y search no tautológicas.
- [ ] Puedo comparar algoritmos con datos equivalentes y múltiples tamaños.
- [ ] Puedo separar preprocessing, query cost y memoria derivada.
- [ ] Puedo construir una timeline determinista sin mutar source.
- [ ] Puedo completar el mini challenge con PF + CS-M1–CS-M4.

---

## 31. Preparación para labs y EIDOLON 0.0b

### Lab principal

**CS-L07 — Timeline search** es el lab directamente preparado por este módulo:

```text
stable sort + tie-breaker
↓
timeline con timestamps aware
↓
bisect para fronteras de rango
↓
source preservado
↓
tests de determinismo y boundaries
```

### Labs previos que aportan evidencia

- **CS-L01 — Curvas de crecimiento:** reutiliza el diseño de tamaños, repeticiones y punto de cruce para search/sort.
- **CS-L02 — List vs dict entity lookup:** aporta el contraste entre scan e índice derivado.

El Curriculum no define un lab separado de recursion o de cada sorting clásico. Esos conceptos se demuestran en los ejercicios, merge sort y el mini challenge; no se inventa un ID.

### Trazabilidad

| Concepto | Sección | Evidencia | Lab |
|---|---:|---|---|
| Base/progreso/call stack | 2–4, 16 | Guiados 1–3 y 11; call tree del challenge | Fundamento interno de CS-L07 |
| Linear vs binary vs dict | 5–10 | Guiados 4–7 y 15; benchmark del challenge | CS-L01, CS-L02, CS-L07 |
| Stability y tie-breaker | 11–17 | Guiados 12–14; replay determinista | CS-L07 |
| Range boundaries con `bisect` | 9, 20 | Ejercicios 17 y 38; challenge | CS-L07 |
| Benchmark search/sort | 22–23 | Guiados 15–17; `decision.md` | CS-L01 y CS-L07 |

### Evidencia antes de avanzar

Entrega:

1. implementaciones educativas y edge-case suite;
2. traza de recursion y binary search;
3. timeline derivada con tie-breaker y source intacto;
4. benchmark con preprocessing separado;
5. decisión escrita por workload;
6. comprobaciones de stability y determinismo.

CS-M4 prepara divide-and-conquer reasoning y ordered workloads que CS-M5 usará. No autoriza comenzar trees/heaps dentro de este módulo.

---

## 32. Recursos de ampliación

El módulo es autocontenido. Para profundizar sin cambiar el alcance, consulta los recursos canónicos del track en [CS.11 Recursos recomendados](../../02_curriculum/02_computer_science_foundations.md#cs11-recursos-recomendados) y la documentación oficial de Python sobre `sorted`, `list.sort`, `bisect`, `timeit` y `RecursionError`.

Úsalos para verificar APIs y explorar variantes; no sustituyen la traza, las preconditions ni el benchmark propio.

---

## 33. Límite explícito del módulo

CS-M4 termina en recursive thinking aplicado, call stack conceptual, linear/binary search, fronteras con `bisect`, selection/insertion/merge sort, stability, deterministic ordering, auxiliary space, testing y benchmarking de search/sort.

No se implementan trees, binary search trees, heaps, top-k, graphs, state machines, OS/memory internals, processes, threads, networking, databases, backend ni AI. CS-M5–CS-M10 desarrollarán los temas correspondientes.

El siguiente paso permitido es revisar CS-M4 como `review candidate`. **No se crea ni se desarrolla CS-M5 en esta entrega.**
