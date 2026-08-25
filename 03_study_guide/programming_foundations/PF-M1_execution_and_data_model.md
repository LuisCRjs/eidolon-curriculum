# PF-M1 — Modelo de ejecución y datos de Python

**Track:** Programming Foundations  
**Competencias:** D1.1, D2.3; refuerza D3.1 y D11.1  
**Fase:** P0  
**Nivel objetivo:** Profesional para el alcance del módulo  
**Prerequisites:** ninguno  
**Build:** EIDOLON 0.0a  
**Curriculum source:** [PF-M1](../../02_curriculum/01_programming_foundations.md#pf-m1--modelo-de-ejecución-y-datos-de-python)  
**Status:** approved

Este módulo enseña a predecir qué datos existen en un programa Python, qué nombres apuntan a ellos y qué operaciones pueden cambiar el estado compartido. También establece fronteras correctas para texto, bytes, números y tiempo. Son fundamentos pequeños en apariencia, pero un error aquí puede convertir un evento válido en texto corrupto, una cantidad exacta en una aproximación o un timestamp ambiguo.

No necesitas conocer clases, decorators, async, frameworks ni bases de datos. Usaremos `list` y `dict` de manera básica para observar referencias; su selección y complejidad se estudian en PF-M3 y CS-M2.

## Resultados de aprendizaje

Al terminar deberías poder:

- explicar que Python enlaza nombres con objetos y que asignar no copia;
- predecir cuándo dos nombres observan el mismo objeto;
- distinguir identidad (`is`) de igualdad (`==`);
- localizar y corregir un bug de aliasing;
- justificar una shallow copy, una deep copy o ninguna copia;
- distinguir ausencia (`None`) de valores falsy válidos;
- separar `str` de `bytes` y controlar la frontera de encoding;
- preservar texto Unicode sin normalizar destructivamente la fuente;
- elegir entre `int`, `float` y `Decimal` según el contrato;
- distinguir `date`, `datetime`, timestamp, naive datetime y timezone-aware datetime;
- comparar y serializar tiempo sin depender de la zona local de la computadora;
- aplicar estas decisiones a un evento sintético de EIDOLON.

## Cómo estudiar este módulo

Ejecuta cada bloque que el texto presente como ejemplo completo y luego modifícalo. Antes de correrlo, escribe tu predicción. No ejecutes de forma aislada un bloque que el texto identifique como continuación, código incorrecto o fragmento. Cuando aparezca código incorrecto, no memorices solo la corrección: explica qué modelo mental produjo el error.

### Convenciones del código

- **Ejemplo ejecutable:** funciona como bloque independiente, salvo que el texto indique que continúa el bloque anterior.
- **Continuación:** reutiliza nombres creados inmediatamente antes; ejecuta ambos bloques en la misma sesión.
- **Código incorrecto:** contiene un fallo deliberado para observarlo o razonarlo; no es una recomendación.
- **Fragmento:** ilustra sintaxis y puede usar nombres genéricos o `...`; no es un programa completo.

Los comentarios `# output` y los bloques `text` muestran el resultado esperado relevante. Un output que incluya identidades de objetos, rutas o mensajes dependientes del sistema se describe en vez de fijarse literalmente.

### Sintaxis de apoyo

PF-M1 usa unas pocas construcciones antes de estudiarlas formalmente:

- `import` hace disponible un nombre de la biblioteca estándar; módulos y packaging llegan en PF-M4;
- `try`/`except` se usa solo para observar un error esperado sin detener el ejemplo; el manejo de errores llega en PF-M6;
- `assert` comprueba una condición local durante los ejercicios; la estrategia de testing llega en PF-M9;
- una list comprehension crea una lista evaluando una expresión por elemento; se estudia en PF-M3.

No necesitas dominar todavía esas construcciones. En este módulo sirven como instrumentos de observación.

La secuencia recomendada es:

1. leer una sección;
2. predecir sus outputs;
3. ejecutar el código;
4. cambiar un dato o una operación;
5. resolver el ejercicio breve;
6. anotar una regla propia con un contraejemplo.

---

## 1. Modelo de ejecución: nombres, objetos y asignación

### 1.1 Por qué existe este modelo

Observa este programa:

```python
tags_a = ["familia", "viaje"]
tags_b = tags_a
tags_b.append("importante")

print(tags_a)
```

Una intuición común dice: “copié `tags_a` en `tags_b`; solo modifiqué `tags_b`”. Python imprime:

```text
['familia', 'viaje', 'importante']
```

El programa no falló. Falló la predicción. Para programar con seguridad necesitas un modelo que te permita anticipar este comportamiento antes de ejecutar.

### 1.2 Modelo mental

Piensa en tres elementos distintos:

- un **nombre** es una etiqueta disponible dentro de un espacio de nombres (namespace);
- un **objeto** es un valor existente durante la ejecución, con tipo, identidad y estado;
- una **asignación** enlaza un nombre con un objeto.

La sentencia:

```python
event_id = 42
```

no debe imaginarse como “una caja llamada `event_id` contiene físicamente 42”. Para razonar en Python es más útil leerla así:

> el nombre `event_id` queda enlazado al objeto entero cuyo valor es 42.

Cuando haces:

```python
event_id = 42
copy_of_id = event_id
```

el segundo nombre queda enlazado al mismo objeto. La asignación no solicita una copia.

### 1.3 Teoría

Python evalúa expresiones y produce objetos. Los nombres permiten recuperar esos objetos dentro de un alcance. Cada objeto posee, al menos para nuestro modelo práctico:

- **tipo:** determina las operaciones válidas;
- **identidad:** permite saber si es exactamente el mismo objeto;
- **valor o estado observable:** lo que representa y, si es mutable, lo que puede cambiar.

Una asignación simple ocurre en dos pasos conceptuales:

1. se evalúa la expresión a la derecha;
2. el nombre de la izquierda se enlaza con el objeto resultante.

```python
subtotal = 120 + 30
```

Primero se evalúa `120 + 30`; después `subtotal` queda enlazado al resultado.

Reasignar un nombre cambia el enlace, no transforma necesariamente el objeto anterior:

```python
status = "draft"
status = "accepted"
```

La segunda línea hace que `status` apunte a otro objeto `str`. El objeto que representa `"draft"` no fue convertido en `"accepted"`.

### 1.4 Sintaxis y mecanismos esenciales

**Fragmento de sintaxis:**

```python
name = expression        # asignación
first = second = []      # ambos nombres reciben el mismo objeto
left, right = (10, 20)   # desempaquetado: cada nombre recibe un elemento
```

La asignación encadenada con un mutable merece atención:

```python
primary_tags = secondary_tags = []
primary_tags.append("private")

print(secondary_tags)
# ['private']
```

Solo se creó una lista.

### 1.5 Ejemplo mínimo

```python
original = ["evento-1"]
alias = original

print(alias is original)  # True

alias.append("evento-2")
print(original)           # ['evento-1', 'evento-2']
```

`original` y `alias` son dos nombres. Ambos recuperan el mismo objeto lista; por eso una mutación observada mediante un nombre también es visible mediante el otro.

### 1.6 Ejemplo progresivo

```python
event = {
    "id": "evt-001",
    "tags": ["viaje"],
}

candidate = event
candidate["tags"].append("relevante")

print(event)
# {'id': 'evt-001', 'tags': ['viaje', 'relevante']}

candidate = {"id": "evt-002", "tags": []}

print(event)
# {'id': 'evt-001', 'tags': ['viaje', 'relevante']}
```

La mutación de la lista ocurrió mientras ambos nombres apuntaban al mismo diccionario. La reasignación posterior de `candidate` no borra esa mutación: solo cambia a qué objeto apunta `candidate`.

### 1.7 Qué ocurre internamente, al nivel que importa

Python administra la memoria y la vida de los objetos. No necesitas estudiar todavía el allocator de CPython ni el bytecode. Lo importante es distinguir:

- crear u obtener un objeto;
- enlazar uno o más nombres con ese objeto;
- mutar el objeto, si su tipo lo permite;
- reasignar un nombre a otro objeto.

`id(obj)` devuelve un entero que identifica al objeto durante su vida. Dos objetos cuyas vidas se solapan tienen identidades distintas; después de que un objeto desaparece, ese entero puede reutilizarse. Es útil para experimentar, pero no es un ID de dominio y no debe persistirse.

```python
items = []
same_items = items

print(id(items) == id(same_items))  # True
```

Para EIDOLON, un ID estable como `"evt-001"` pertenece al modelo de datos. `id(event)` pertenece únicamente a esa ejecución de Python.

### 1.8 Errores comunes

**Error: creer que cambiar el alias crea una copia retroactiva.**

```python
source = [1, 2]
alias = source
alias.append(3)
alias = [99]

print(source)  # [1, 2, 3]
```

Reasignar `alias` no deshace la mutación anterior.

**Error: utilizar nombres como si garantizaran propiedad.**

Llamar a una variable `original` no impide que otro nombre apunte al mismo objeto. La propiedad es una decisión de diseño, no una característica inferida del nombre.

### 1.9 Aplicación en EIDOLON

Un `Event` y una `Correction` deben ser entidades conceptualmente separadas. Si una operación “crea una corrección” pero en realidad modifica el mismo diccionario del evento, destruye la capacidad de replay y provenance.

```python
event = {"id": "evt-001", "text": "Llegué el lunes"}
correction = event  # diseño incorrecto: es un alias
correction["text"] = "Llegué el martes"

print(event["text"])
# Llegué el martes
```

El error no se resuelve renombrando variables. Se necesita crear un objeto distinto y conservar el vínculo explícito con el original. El modelado formal llegará en PF-M5; aquí debes reconocer el riesgo.

### 1.10 Cuándo no sobredimensionarlo

No necesitas dibujar un grafo de objetos para cada línea. Usa este modelo cuando exista mutación, una frontera entre componentes, un bug de estado compartido o una decisión de copia. Para enteros y strings en código directo suele bastar con recordar que la reasignación enlaza a otro objeto.

> **Antes de continuar:** resuelve el ejercicio guiado 1 sin mirar su solución. Dibuja solo los enlaces que cambian y distingue mutación de reasignación.

---

## 2. Identidad, igualdad, `is` y `==`

### 2.1 El problema

Estas preguntas no son equivalentes:

1. ¿Son exactamente el mismo objeto?
2. ¿Representan valores equivalentes?

Python ofrece operadores distintos porque ambas preguntas importan.

### 2.2 Modelo mental

- `a is b` pregunta por **identidad**: “¿son el mismo objeto?”.
- `a == b` pregunta por **igualdad**: “¿sus valores se consideran equivalentes?”.

Dos objetos diferentes pueden ser iguales:

```python
left = ["evidence"]
right = ["evidence"]

print(left == right)  # True
print(left is right)  # False
```

### 2.3 Teoría y sintaxis

**Fragmento de sintaxis:**

```python
same_object = a is b
different_object = a is not b
equal_value = a == b
different_value = a != b
```

`is` no consulta el contenido. `==` delega la comparación al comportamiento de los tipos involucrados. Más adelante podrás definir igualdad para objetos propios; en PF-M1 basta con comprender que igualdad no implica identidad.

### 2.4 Ejemplo progresivo

```python
stored_id = "evt-001"
requested_id = "".join(["evt", "-", "001"])

print(stored_id == requested_id)  # True
print(stored_id is requested_id)  # No debe formar parte del contrato
```

Aunque en alguna ejecución `is` pareciera devolver `True` para strings o enteros iguales, Python puede reutilizar ciertos objetos como optimización. Ese detalle no constituye una garantía para comparar valores.

### 2.5 Regla práctica para `None`

`None` es un singleton: existe un único objeto que representa ausencia. Por eso la forma idiomática es:

```python
valid_time = None

if valid_time is None:
    print("El evento no declara valid_time")
```

Evita:

```python
valid_time = None

if valid_time == None:  # funciona en casos simples, pero no expresa la intención correcta
    ...
```

### 2.6 Error común: comparar IDs con `is`

```python
event_id = "evt-001"
query_id = "".join(("evt", "-", "001"))

if event_id is query_id:  # incorrecto
    print("Encontrado")
```

`query_id` se construye durante la ejecución para no depender de que el compilador pliegue dos literales. Aun así, el resultado concreto de `is` no debe convertirse en contrato: el operador responde la pregunta equivocada.

Un ID de dominio se compara por valor:

```python
event_id = "evt-001"
query_id = "".join(("evt", "-", "001"))

if event_id == query_id:
    print("Encontrado")
```

### 2.7 Aplicación en EIDOLON

- Usa `==` para comparar IDs, textos, estados y valores definidos por el dominio.
- Usa `is None` para detectar ausencia.
- Usa `is` entre objetos solo cuando la identidad del objeto en memoria sea realmente la pregunta, algo poco frecuente en el dominio de EIDOLON.

### 2.8 Cuándo no usar `is`

No uses `is` para números, strings, fechas, bytes, listas o IDs por el hecho de que “a veces funciona”. Tampoco persistas `id(obj)` para identificar eventos. La identidad de ejecución y la identidad del dominio son conceptos diferentes.

> **Antes de continuar:** predice `left == right` y `left is right` para dos listas creadas por separado con el mismo contenido. Explica qué pregunta responde cada operador antes de ejecutar.

---

## 3. Mutabilidad, inmutabilidad y aliasing

### 3.1 Por qué importa

Un programa longitudinal conserva datos que luego se vuelven evidencia. Si dos componentes comparten accidentalmente una estructura mutable, uno puede cambiar lo que el otro considera histórico sin dejar una operación explícita de corrección.

### 3.2 Modelo mental

La mutabilidad (mutability) responde:

> ¿puede cambiar el estado observable de este objeto sin reemplazarlo por otro?

Ejemplos habituales:

| Tipo | Comportamiento práctico |
|---|---|
| `list`, `dict`, `set`, `bytearray` | Mutables |
| `int`, `float`, `Decimal`, `str`, `bytes`, `tuple`, `date`, `datetime` | Inmutables |

Una tupla es inmutable como contenedor, pero puede referenciar objetos mutables:

```python
container = (["original"], "evt-001")
container[0].append("derived")

print(container)
# (['original', 'derived'], 'evt-001')
```

No cambió qué objetos contiene la tupla; sí cambió el estado de la lista contenida.

### 3.3 Mutación frente a reasignación

```python
tags = ["uno"]
alias = tags

tags.append("dos")  # muta la lista compartida
print(alias)         # ['uno', 'dos']

tags = ["nuevo"]    # reasigna solo el nombre tags
print(alias)         # ['uno', 'dos']
```

### 3.4 Aliasing

Aliasing ocurre cuando varios nombres o rutas de acceso conducen al mismo objeto. Puede existir también con objetos inmutables; se vuelve especialmente importante cuando el objeto es mutable porque una ruta puede cambiar el estado que observan las demás.

```python
event = {"metadata": {"source": "journal"}}
metadata = event["metadata"]

metadata["reviewed"] = True

print(event)
# {'metadata': {'source': 'journal', 'reviewed': True}}
```

El aliasing no es siempre un error. Compartir estado puede ser deliberado y eficiente. El problema aparece cuando la propiedad, el permiso para mutar o el ciclo de vida (lifecycle) no están claros.

### 3.5 Ejemplo clásico: repetición de listas

Código incorrecto:

```python
rows = [[0] * 3] * 3
rows[0][0] = 1

print(rows)
# [[1, 0, 0], [1, 0, 0], [1, 0, 0]]
```

`* 3` repitió tres referencias a la misma lista interna. Para crear listas independientes:

```python
rows = [[0] * 3 for _ in range(3)]
rows[0][0] = 1

print(rows)
# [[1, 0, 0], [0, 0, 0], [0, 0, 0]]
```

La comprehension se estudia formalmente en PF-M3. Por ahora observa la diferencia: la expresión interna se evalúa una vez por iteración y crea tres listas.

### 3.6 Copia superficial (shallow copy)

Una shallow copy crea un contenedor exterior nuevo, pero conserva referencias a los objetos anidados.

```python
import copy

event = {
    "id": "evt-001",
    "metadata": {"tags": ["viaje"]},
}

shallow = copy.copy(event)

print(shallow is event)                     # False
print(shallow["metadata"] is event["metadata"])  # True

shallow["id"] = "evt-002"
shallow["metadata"]["tags"].append("2026")

print(event)
# {'id': 'evt-001', 'metadata': {'tags': ['viaje', '2026']}}
```

La reasignación de `shallow["id"]` solo afectó el contenedor nuevo. La lista anidada continuó compartida.

Para listas y diccionarios también existen formas específicas. Este es un **fragmento de sintaxis**:

```python
list_copy = original_list.copy()
dict_copy = original_dict.copy()
```

Siguen siendo copias superficiales.

### 3.7 Copia profunda (deep copy)

`copy.deepcopy` intenta copiar recursivamente el grafo de objetos:

```python
import copy

event = {
    "id": "evt-001",
    "metadata": {"tags": ["viaje"]},
}

deep = copy.deepcopy(event)
deep["metadata"]["tags"].append("2026")

print(event["metadata"]["tags"])  # ['viaje']
print(deep["metadata"]["tags"])   # ['viaje', '2026']
```

`deepcopy` mantiene un registro interno de objetos ya copiados para manejar referencias repetidas y ciclos. Aun así, no conoce la semántica de tu dominio. Puede copiar demasiado, ser costosa o producir una réplica inválida de recursos como archivos, conexiones o locks.

### 3.8 Elegir entre compartir, copiar y reconstruir

Usa esta secuencia de preguntas:

1. ¿El objeto será mutado?
2. Si se muta, ¿todos los observadores deben ver el cambio?
3. ¿Solo necesito un contenedor exterior independiente?
4. ¿Los objetos anidados también necesitan independencia?
5. ¿Una construcción explícita del nuevo objeto expresa mejor la intención?

Para una `Correction`, una construcción semántica suele ser mejor que un `deepcopy` ciego:

```python
event = {"id": "evt-001", "text": "lunes"}

correction = {
    "id": "cor-001",
    "target_event_id": event["id"],
    "replacement_text": "martes",
}
```

Ahora el modelo comunica que la corrección no es otro evento copiado, sino una operación relacionada.

### 3.9 Errores comunes

- Copiar “por seguridad” en cada capa y perder costo, claridad y relaciones intencionales.
- Usar shallow copy creyendo que duplica objetos anidados.
- Usar `deepcopy` como sustituto de un modelo de dominio explícito.
- Exponer una lista interna y asumir que quien la recibe no la mutará.
- Confundir inmutabilidad del contenedor con inmutabilidad transitiva de todo el grafo.

### 3.10 Aplicación en EIDOLON

Los registros fuente requieren una política clara de ownership. Una vista derivada puede compartir objetos inmutables sin riesgo, pero no debería recibir una lista mutable interna si puede modificarla sin dejar evidencia. Las correcciones deben ser objetos nuevos vinculados al evento, no copias profundas transformadas que oculten la operación.

### 3.11 Cuándo no copiar

No copies estructuras grandes automáticamente “por si acaso”. Si el código solo lee un objeto, una copia puede ser desperdicio. Si necesitas una vista inmutable, diseña esa frontera de forma explícita en módulos posteriores. Copia cuando la independencia sea parte del contrato y prueba exactamente qué nivel debe ser independiente.

> **Antes de continuar:** resuelve el ejercicio guiado 2 y predice qué niveles comparten `original` y `candidate` antes de ejecutar. Después decide si el contrato exige shallow copy, deep copy o reconstrucción explícita.

---

## 4. `None` y truthiness

### 4.1 El problema de mezclar ausencia con falsedad

Supón que `confidence = 0.0` significa “existe una estimación y su valor es cero”, mientras `confidence = None` significa “no existe estimación”. Este código destruye la diferencia:

```python
confidence = 0.0

if not confidence:
    print("No hay estimación")  # mensaje incorrecto
```

### 4.2 Modelo mental

`None` representa ausencia de valor. Truthiness es la regla mediante la cual Python decide si un objeto se comporta como verdadero o falso en un contexto booleano.

Son falsy, entre otros:

- `None`;
- `False`;
- ceros numéricos como `0`, `0.0` y `Decimal("0")`;
- secuencias y colecciones vacías como `""`, `b""`, `[]`, `{}` y `set()`.

La mayoría de los demás objetos son truthy.

### 4.3 Sintaxis correcta según la pregunta

Pregunta por ausencia:

```python
valid_time = None

if valid_time is None:
    print("No se proporcionó una fecha")
```

Pregunta por contenido vacío:

```python
text = ""

if text == "":
    print("Se proporcionó texto, pero está vacío")
```

Pregunta por comportamiento booleano general:

```python
tags = ["importante"]

if tags:
    print("Existe al menos un tag")
```

### 4.4 `and` y `or` devuelven operandos

No siempre producen `True` o `False`:

```python
label = "" or "sin etiqueta"
print(label)  # sin etiqueta

value = "evento" and 42
print(value)  # 42
```

Esto permite defaults compactos, pero puede borrar valores falsy válidos:

```python
provided_count = 0
count = provided_count or 10

print(count)  # 10; quizá era un bug
```

Si `0` es válido y solo `None` significa ausencia:

```python
provided_count = 0
count = 10 if provided_count is None else provided_count
```

### 4.5 Aplicación en EIDOLON

No colapses estados distintos:

- `None`: no se conoce el valor;
- `""`: el usuario proporcionó texto vacío o el parsing lo produjo;
- `0.0`: existe una medida cuyo valor es cero;
- `[]`: existe una colección y actualmente está vacía.

Estos estados pueden requerir políticas y explicaciones diferentes.

### 4.6 Cuándo usar truthiness

Úsala cuando todos los valores falsy tengan el mismo significado para el contrato, por ejemplo “no hay tags para iterar”. Evítala cuando debas distinguir ausencia, cero, vacío y `False`.

> **Antes de continuar:** resuelve el ejercicio guiado 3. Prueba por separado `None`, `0`, `False` y `""`; no aceptes una solución que borre sus diferencias semánticas.

---

## 5. `str`, Unicode y code points

### 5.1 Por qué existe Unicode

El texto humano no cabe en un alfabeto de 128 caracteres. EIDOLON debe conservar español, nombres, emojis y escrituras diferentes sin depender del sistema operativo o del idioma de la computadora.

Python usa `str` para texto Unicode. Esto permite representar:

```python
text = "Canción en Mérida 🎵"
print(text)
```

### 5.2 Modelo mental

Separa cuatro ideas:

1. **texto abstracto:** lo que una persona interpreta como caracteres;
2. **code point:** número asignado por Unicode a una unidad de texto;
3. **`str`:** secuencia de code points en el modelo del lenguaje Python;
4. **bytes:** representación concreta usada al guardar o transmitir.

Un code point suele escribirse como `U+` seguido de hexadecimal. Por ejemplo, `A` es `U+0041` y `ñ` es `U+00F1`.

### 5.3 Mecanismos prácticos: `ord`, `chr` y escapes

```python
print(ord("A"))          # 65
print(hex(ord("ñ")))     # 0xf1
print(chr(0x1F3B5))      # 🎵
print("\N{MUSICAL NOTE}")  # ♪
```

`ord` convierte un carácter de un solo code point en entero. `chr` hace la operación inversa para un code point válido.

### 5.4 `len` no siempre cuenta lo que una persona ve

Estos dos textos pueden verse iguales:

```python
composed = "é"          # U+00E9
decomposed = "e\u0301" # U+0065 + U+0301

print(composed)                 # é
print(decomposed)               # é
print(len(composed))            # 1
print(len(decomposed))          # 2
print(composed == decomposed)   # False
```

El segundo usa `e` más un acento combinante. Python cuenta code points, no grafemas percibidos por una persona. Emojis con modificadores o secuencias unidas también pueden ocupar varios code points.

No necesitas implementar segmentación de grafemas en PF-M1. Sí necesitas evitar la afirmación incorrecta de que `len(text)` siempre equivale a “número de caracteres visibles”.

### 5.5 Normalización Unicode

Unicode puede representar texto visualmente equivalente de varias formas. `unicodedata.normalize` permite normalizar para un propósito concreto:

```python
import unicodedata

composed = "é"
decomposed = "e\u0301"

normalized_left = unicodedata.normalize("NFC", composed)
normalized_right = unicodedata.normalize("NFC", decomposed)

print(normalized_left == normalized_right)  # True
```

Formas frecuentes:

- `NFC`: compone cuando existe una forma canónica; buena opción general para comparación controlada;
- `NFD`: descompone canónicamente;
- `NFKC` y `NFKD`: aplican además equivalencias de compatibilidad y pueden perder distinciones visuales o semánticas relevantes.

### 5.6 Preservar el original y derivar una vista

En EIDOLON, normalizar destructivamente el registro fuente daña provenance. Conserva el texto original y crea una representación derivada para búsqueda:

```python
import unicodedata

source_text = "Cancio\u0301n en Mérida 🎵"
search_text = unicodedata.normalize("NFC", source_text).casefold()

print(source_text)
print(search_text)
```

`casefold()` es una transformación más adecuada que `lower()` para comparación Unicode sin distinción de mayúsculas. Tampoco debe reemplazar silenciosamente el original.

### 5.7 Errores comunes

**Error: normalizar sin declarar el objetivo.**

Una normalización de compatibilidad puede unir símbolos que el dominio necesitaba distinguir. Primero define si buscas igualdad canónica, búsqueda tolerante, presentación o preservación forense.

**Error: usar índices como posiciones visuales.**

```python
text = "e\u0301"
print(text[0])  # e, sin el acento combinante
```

El slicing puede separar una secuencia que el usuario percibe como una unidad.

**Error: eliminar acentos en el registro fuente para facilitar búsqueda.**

Genera un campo derivado. No mutiles el dato original.

### 5.8 Aplicación en EIDOLON

EIDOLON puede necesitar simultáneamente:

- `source_text`: exactamente lo recibido;
- `display_text`: texto aprobado para mostrar;
- `search_text`: forma normalizada y quizá casefolded;
- bytes originales o su hash, cuando la política de ingesta lo requiera.

La representación derivada debe registrar qué transformación y versión la produjo en fases posteriores.

En PF-M1 esta lista ilustra la separación entre fuente y derivados; no define todavía el schema, la política de hashes ni la arquitectura de ingesta.

### 5.9 Cuándo no normalizar

No normalices automáticamente contraseñas, firmas, checksums, identificadores opacos ni texto cuyo valor legal o forense dependa de los code points exactos. Tampoco asumas que NFC resuelve reglas lingüísticas, transliteración o búsqueda semántica.

> **Antes de continuar:** resuelve el ejercicio guiado 4 y conserva ambas fuentes. Luego ejecuta el ejercicio independiente 6 para inspeccionar los code points, no solo el resultado de `==`.

---

## 6. `bytes`, encoding, decoding y UTF-8

### 6.1 El problema de la frontera

Los archivos, sockets y APIs intercambian bytes. Las personas trabajan con texto. Un programa seguro debe indicar dónde ocurre la conversión.

```text
str --encode--> bytes --decode--> str
```

Encoding y decoding no son formatos decorativos: son contratos de interpretación.

### 6.2 Modelo mental

- `str` representa texto Unicode.
- `bytes` representa una secuencia inmutable de enteros de 0 a 255.
- un **encoding** define cómo convertir code points en bytes;
- **decoding** aplica esa convención en sentido inverso.

UTF-8 representa cada code point usando de uno a cuatro bytes. El texto ASCII es también UTF-8 válido con los mismos bytes para sus primeros 128 caracteres.

### 6.3 Sintaxis

```python
text = "Mérida 🎵"
payload = text.encode("utf-8")
restored = payload.decode("utf-8")

print(payload)
print(restored)
print(restored == text)  # True
```

Output representativo:

```text
b'M\xc3\xa9rida \xf0\x9f\x8e\xb5'
Mérida 🎵
True
```

La representación `b'...'` muestra escapes para bytes no ASCII. No significa que el texto se haya corrompido.

### 6.4 Un code point puede ocupar varios bytes

```python
text = "é🎵"
payload = text.encode("utf-8")

print(len(text))     # 2 code points
print(len(payload))  # 6 bytes: 2 para é y 4 para 🎵
```

Por eso no puedes usar `len(text)` como tamaño de almacenamiento o límite de bytes.

### 6.5 Errores de encoding

Un encoding limitado no puede representar todo Unicode:

```python
text = "Mérida 🎵"

try:
    text.encode("ascii")
except UnicodeEncodeError as error:
    print(type(error).__name__)
# UnicodeEncodeError
```

El error es información útil: el contrato `ascii` era incompatible con el dato.

### 6.6 Errores de decoding

Los mismos bytes producen resultados distintos o errores según el encoding elegido:

```python
payload = b"M\xc3\xa9rida"

print(payload.decode("utf-8"))    # Mérida
print(payload.decode("latin-1"))  # MÃ©rida: mojibake
```

`latin-1` puede asignar cada byte a un code point, así que no falla; produce texto incorrecto. La ausencia de excepción no demuestra que el encoding sea correcto.

Bytes UTF-8 truncados sí generan un error estricto:

```python
broken = b"\xf0\x9f\x8e"  # falta un byte del emoji

try:
    broken.decode("utf-8")
except UnicodeDecodeError as error:
    print(type(error).__name__)
# UnicodeDecodeError
```

### 6.7 Políticas de error

```python
broken = b"texto:\xff"

print(broken.decode("utf-8", errors="replace"))
# texto:�

print(broken.decode("utf-8", errors="ignore"))
# texto:
```

- `strict` es el default y falla explícitamente;
- `replace` conserva la posición del daño mediante `�`, pero pierde el byte original en el texto resultante;
- `ignore` elimina datos silenciosamente y rara vez es aceptable para registros canónicos.

Una ingesta responsable puede conservar los bytes originales, enviar el registro a quarantine y producir un receipt sin datos sensibles. No debe inventar texto válido.

### 6.8 Vista previa de una frontera de archivo (PF-M6)

El patrón seguro para texto es:

```python
from pathlib import Path

path = Path("evento.txt")
path.write_text("Mérida 🎵", encoding="utf-8")
restored = path.read_text(encoding="utf-8")

print(restored)
```

No necesitas dominar `Path` ni archivos en PF-M1. El ciclo de vida de archivos y sus errores se profundizan en PF-M6. Aquí importa una sola regla: una frontera de texto declara `encoding="utf-8"` en vez de depender del default del sistema.

### 6.9 Aplicación en EIDOLON

En un importador:

1. recibe bytes;
2. conoce o detecta mediante una política explícita el encoding permitido;
3. decodifica con errores estrictos;
4. conserva provenance;
5. normaliza solo una representación derivada;
6. pone en quarantine el dato que no satisface el contrato.

### 6.10 Cuándo no convertir

No decodifiques archivos binarios como si fueran texto. No hagas `str(payload)` para “convertir” bytes:

```python
payload = "Mérida".encode("utf-8")
print(str(payload))
# b'M\xc3\xa9rida'
```

Eso produce la representación textual del objeto `bytes`, no el contenido decodificado. Usa `payload.decode("utf-8")` cuando el contrato sea UTF-8.

> **Antes de continuar:** resuelve el ejercicio guiado 5 y después provoca los dos fallos del ejercicio independiente 9. Conserva los bytes que no pudieron decodificarse.

---

## 7. `int`, `float` y `Decimal`

### 7.1 El problema: “número” no es un contrato suficiente

Estas cantidades tienen necesidades distintas:

- número de eventos: debe ser entero;
- score aproximado de similitud: tolera error de punto flotante;
- cantidad monetaria: normalmente requiere aritmética decimal explícita;
- ID escrito con dígitos: quizá no sea un número y deba conservar ceros iniciales.

Elegir el tipo comunica qué operaciones y errores son aceptables.

### 7.2 `int`: enteros exactos

En Python, `int` representa enteros con precisión arbitraria limitada por la memoria disponible.

```python
event_count = 3
next_count = event_count + 1

print(next_count)  # 4
```

No existe overflow de ancho fijo como en muchos tipos enteros de otros lenguajes, aunque números enormes consumen tiempo y memoria.

Usa `int` para conteos, offsets, longitudes y magnitudes discretas. No conviertas un identificador a `int` solo porque contiene dígitos:

```python
source_id = "000123"
print(int(source_id))  # 123: se perdió la representación original
```

### 7.3 `float`: aproximación binaria

En las plataformas habituales, incluido CPython, `float` sigue el modelo IEEE 754 de doble precisión. Puede representar exactamente muchos valores, pero no todos los decimales finitos.

```python
result = 0.1 + 0.2

print(result)
# 0.30000000000000004

print(result == 0.3)
# False
```

No ocurrió una suma “mal hecha”. `0.1`, `0.2` y `0.3` no tienen una representación binaria finita exacta; Python opera con aproximaciones cercanas.

### 7.4 Qué está ocurriendo internamente

Una fracción como $1/2$ tiene representación binaria finita: `0.1₂`. Una fracción como $1/10$ se repite en base 2, de manera semejante a cómo $1/3$ se repite en base 10. Con una cantidad finita de bits se almacena el valor representable más cercano.

La profundidad matemática objetivo es **Level 1 — practical formulas**. Debes comprender precisión finita, no derivar el estándar IEEE 754.

### 7.5 Comparación aproximada

Para resultados numéricos aproximados usa una tolerancia justificada:

```python
import math

result = 0.1 + 0.2

print(math.isclose(result, 0.3))
# True
```

`math.isclose(a, b)` considera tolerancia relativa y absoluta. Para valores cercanos a cero, una tolerancia absoluta puede ser necesaria:

```python
import math

print(math.isclose(1e-12, 0.0, abs_tol=1e-9))
# True
```

No elijas tolerancias al azar para “hacer pasar el test”. Deben surgir del error aceptable del dominio.

### 7.6 `Decimal`: aritmética decimal controlada

Cuando el contrato exige comportamiento decimal, usa `Decimal` y constrúyelo desde texto:

```python
from decimal import Decimal

subtotal = Decimal("0.1") + Decimal("0.2")

print(subtotal)                # 0.3
print(subtotal == Decimal("0.3"))  # True
```

Evita construirlo desde un `float` ya aproximado:

```python
from decimal import Decimal

print(Decimal(0.1))
# 0.1000000000000000055511151231257827021181583404541015625
```

`Decimal` conserva exactamente el valor binario aproximado recibido. No puede adivinar que la intención humana era el texto `"0.1"`.

`Decimal` tampoco vuelve exacta toda operación. Los valores decimales de entrada pueden representarse exactamente, pero un resultado infinito como $1/3$ se redondea según la precisión y el modo del contexto decimal:

```python
from decimal import Decimal

print(Decimal("1") / Decimal("3"))
# 0.3333333333333333333333333333 con el contexto predeterminado
```

La cantidad de dígitos mostrada depende del contexto activo. En PF-M1 basta con reconocer que el tipo permite controlar esa política; configurar contextos avanzados queda fuera del alcance.

### 7.7 Redondeo explícito

```python
from decimal import Decimal, ROUND_HALF_UP

amount = Decimal("12.345")
rounded = amount.quantize(Decimal("0.01"), rounding=ROUND_HALF_UP)

print(rounded)  # 12.35
```

El modo de redondeo pertenece al contrato. Finanzas, estadísticas y presentación pueden requerir reglas distintas.

### 7.8 Ejemplo progresivo: score y cantidad exacta

```python
from decimal import Decimal
import math

retrieval_score = 0.1 + 0.2
expected_score = 0.3
amount = Decimal("149.90")

print(math.isclose(retrieval_score, expected_score, rel_tol=1e-9))
# True

print(amount + Decimal("0.10"))
# 150.00
```

Un score de ranking suele ser aproximado y compararse con tolerancia. Una cantidad monetaria puede requerir decimales exactos y política de redondeo. Usar `Decimal` para todo no mejora automáticamente el sistema.

### 7.9 Serialización segura

Cuando serialices un valor exacto, declara el contrato. Muchos formatos JSON no distinguen un decimal exacto de un número binario aproximado. Una estrategia simple en P0 es serializar el decimal como texto más una unidad o schema conocido:

```python
from decimal import Decimal

amount = Decimal("149.90")
serialized_amount = str(amount)

print(serialized_amount)  # '149.90'
```

La validación y serialización JSON completa pertenecen a PF-M6. La idea esencial es no convertir a `float` solo para facilitar la salida.

### 7.10 Errores comunes

- Comparar resultados `float` con `==` cuando el cálculo introduce aproximación.
- Reemplazar una comparación exacta por `isclose` sin que el dominio tolere error.
- Construir `Decimal` desde `float` y creer que recupera el decimal original.
- Mezclar `Decimal` y `float` sin una frontera explícita.
- Convertir IDs numéricos a `int` y perder ceros o formato.
- Usar `float` para dinero por comodidad.

### 7.11 Aplicación en EIDOLON

Un `confidence_score` experimental puede ser `float` si su definición acepta aproximación y sus comparaciones usan tolerancias o rangos. Una cantidad exacta citada por el usuario podría conservarse como texto original y, si se calcula, como `Decimal`. El número de eventos es `int`. Ninguno de estos tipos convierte una inferencia en hecho: solo representa el dato bajo un contrato.

### 7.12 Cuándo no usar `Decimal`

No lo uses automáticamente para embeddings, trigonometría, modelos numéricos o bibliotecas que esperan `float`. `Decimal` suele ser más lento y no elimina errores de medición ni incertidumbre. Úsalo cuando la semántica decimal y el redondeo controlado sean requisitos reales.

> **Antes de continuar:** resuelve el ejercicio guiado 6. Para cada solución escribe primero si el contrato permite aproximación o exige semántica decimal; el test no decide el tipo por sí solo.

---

## 8. `date`, `datetime`, timestamps y zonas horarias

### 8.1 Por qué el tiempo exige un contrato

“25 de agosto a las 8” puede significar:

- una fecha sin hora;
- una hora local en Mérida;
- un instante global;
- una hora registrada por una computadora con zona incorrecta;
- una expresión aproximada del usuario;
- el momento en que ocurrió algo o el momento en que se registró.

Un solo string no resuelve esas diferencias. PF-M1 se concentra en representar y serializar tiempo de forma no ambigua; el modelado temporal profundo llega después.

### 8.2 `date`: fecha civil sin hora

```python
from datetime import date

birthday = date(2026, 5, 12)

print(birthday)             # 2026-05-12
print(birthday.isoformat()) # 2026-05-12
```

Usa `date` cuando el contrato solo necesita año, mes y día. No inventes medianoche ni una zona horaria para almacenar un cumpleaños.

### 8.3 `datetime`: fecha y hora

```python
from datetime import datetime

meeting = datetime(2026, 8, 25, 14, 30)
print(meeting)
# 2026-08-25 14:30:00
```

Este objeto es un **naive datetime**: `tzinfo` es `None`. Contiene campos de calendario y reloj, pero no identifica por sí mismo un instante global.

### 8.4 Naive y timezone-aware datetime

Un **timezone-aware datetime** incluye información suficiente para calcular su offset respecto de UTC. En términos prácticos, `tzinfo` no es `None` y `utcoffset()` tampoco devuelve `None`:

```python
from datetime import UTC, datetime

recorded_at = datetime.now(UTC)

is_aware = recorded_at.tzinfo is not None and recorded_at.utcoffset() is not None

print(is_aware)                # True
print(recorded_at.utcoffset()) # 0:00:00
```

Para representar un instante registrado por EIDOLON, UTC es un default seguro:

```python
from datetime import UTC, datetime

recorded_at = datetime(2026, 8, 25, 20, 0, tzinfo=UTC)
print(recorded_at.isoformat())
# 2026-08-25T20:00:00+00:00
```

El sufijo `+00:00` elimina la ambigüedad del offset.

### 8.5 Zonas horarias reales con `zoneinfo`

Un offset fijo no contiene las reglas históricas y futuras de una región. Para una hora civil usa un nombre de zona IANA:

```python
from datetime import datetime
from zoneinfo import ZoneInfo

merida = ZoneInfo("America/Merida")
local_time = datetime(2026, 8, 25, 14, 0, tzinfo=merida)

print(local_time.isoformat())
print(local_time.astimezone(ZoneInfo("UTC")).isoformat())
```

El output UTC dependerá de las reglas de zona vigentes para esa fecha. No fijes manualmente `-06:00` si el contrato necesita reglas regionales a través del tiempo.

`ZoneInfo` necesita una base de zonas IANA. Busca primero los datos del sistema y, si no existen, puede usar el paquete oficial `tzdata`. En algunos entornos —especialmente Windows— debes declarar esa dependencia; de lo contrario, una clave válida puede producir `ZoneInfoNotFoundError`. La gestión reproducible del ambiente se estudia en PF-M4.

### 8.6 Convertir zona no cambia el instante

```python
from datetime import datetime
from zoneinfo import ZoneInfo

merida = ZoneInfo("America/Merida")
mexico_city = ZoneInfo("America/Mexico_City")

local_time = datetime(2026, 8, 25, 14, 0, tzinfo=merida)
other_view = local_time.astimezone(mexico_city)

print(local_time.timestamp() == other_view.timestamp())
# True
```

Son dos representaciones civiles del mismo instante.

### 8.7 Timestamp

En este módulo, timestamp significa tiempo Unix: segundos desde el epoch `1970-01-01T00:00:00Z`, sin contar leap seconds según el modelo habitual del sistema.

```python
from datetime import UTC, datetime

instant = datetime(2026, 8, 25, 20, 0, tzinfo=UTC)
timestamp_seconds = instant.timestamp()
restored = datetime.fromtimestamp(timestamp_seconds, tz=UTC)

print(restored == instant)  # True
```

Un timestamp representa un instante, no una zona. Para mostrar la hora civil original quizá necesites conservar también el nombre de zona o el texto fuente.

Declara siempre la unidad. `seconds`, `milliseconds` y `microseconds` pueden confundirse por varios órdenes de magnitud.

### 8.8 Error: depender de la zona local implícita

```python
from datetime import datetime

timestamp_seconds = 1787688000
local_interpretation = datetime.fromtimestamp(timestamp_seconds)
```

Sin `tz`, el resultado depende de la configuración local del equipo. Dos máquinas pueden producir horas diferentes. Para un instante canónico:

```python
from datetime import UTC, datetime

timestamp_seconds = 1787688000
canonical = datetime.fromtimestamp(timestamp_seconds, tz=UTC)
```

### 8.9 Comparar naive y aware

Python impide ordenar un naive datetime y uno aware porque no existe una interpretación segura:

```python
from datetime import UTC, datetime

naive = datetime(2026, 8, 25, 14, 0)
aware = datetime(2026, 8, 25, 14, 0, tzinfo=UTC)

try:
    print(naive < aware)
except TypeError as error:
    print(type(error).__name__)
# TypeError
```

La igualdad se comporta de otra manera: un naive datetime y uno aware no son iguales, pero esa comparación no lanza `TypeError`. No generalices el ejemplo de ordenación a todos los operadores.

No “arregles” el error eliminando `tzinfo`:

```python
from datetime import UTC, datetime

aware = datetime(2026, 8, 25, 14, 0, tzinfo=UTC)
unsafe = aware.replace(tzinfo=None)
```

`replace(tzinfo=None)` descarta la información de zona sin convertir la hora. Solo es correcto si el contrato realmente quiere una hora civil sin zona.

### 8.10 Asignar zona frente a convertir zona

Estas operaciones responden preguntas distintas:

```python
from datetime import UTC, datetime
from zoneinfo import ZoneInfo

naive_local = datetime(2026, 8, 25, 14, 0)
merida = ZoneInfo("America/Merida")

# Interpretación: esos campos de reloj pertenecen a Mérida.
localized = naive_local.replace(tzinfo=merida)

# Conversión: representa el mismo instante en UTC.
converted = localized.astimezone(UTC)

print(localized.isoformat())
print(converted.isoformat())
```

`replace(tzinfo=...)` no convierte; adjunta una interpretación. Debe usarse solo cuando sabes qué zona corresponde a los campos naive. Para zonas con cambios de horario, horas ambiguas o inexistentes requieren política adicional. `zoneinfo` no valida por sí solo toda hora civil construida manualmente.

### 8.11 Horas ambiguas y `fold`

Cuando un reloj se atrasa por una transición de zona, una misma hora civil puede ocurrir dos veces. `datetime` dispone del atributo `fold` (`0` o `1`) para distinguirlas.

No necesitas dominar calendarios históricos en PF-M1, pero sí reconocer esta regla:

> una hora local y un nombre de zona pueden no bastar durante una transición; la fuente o la política deben resolver la ambigüedad.

Nunca inventes `fold` si la fuente no permite saber cuál instante fue.

### 8.12 Serialización segura

Para `date`:

```python
from datetime import date

value = date(2026, 8, 25)
serialized = value.isoformat()
restored = date.fromisoformat(serialized)

print(serialized)         # 2026-08-25
print(restored == value)  # True
```

Para un instante:

```python
from datetime import UTC, datetime

value = datetime(2026, 8, 25, 20, 0, 0, 123456, tzinfo=UTC)
serialized = value.isoformat(timespec="microseconds")
restored = datetime.fromisoformat(serialized)

print(serialized)
# 2026-08-25T20:00:00.123456+00:00
print(restored == value)
# True
```

Un contrato de EIDOLON debería exigir offset en cada datetime serializado. Puedes almacenar UTC para comparación y conservar por separado la zona o expresión original cuando sea relevante.

### 8.13 `recorded_at` no es `valid_time`

Considera el mensaje escrito el 25 de agosto:

> “La conversación ocurrió el 20 de agosto.”

- `recorded_at`: cuándo EIDOLON recibió o registró el dato;
- `valid_time`: cuándo afirma la fuente que ocurrió el evento.

En PF-M1 solo representamos ambos correctamente. El modelado bitemporal completo pertenece a fases posteriores.

```python
from datetime import UTC, date, datetime

recorded_at = datetime(2026, 8, 25, 20, 0, tzinfo=UTC)
valid_date = date(2026, 8, 20)

print(recorded_at.isoformat())  # instante preciso
print(valid_date.isoformat())   # solo fecha conocida
```

No inventes las `00:00:00` del 20 de agosto si la fuente solo declaró una fecha.

### 8.14 Errores comunes

- Usar naive datetime para instantes persistidos.
- Mezclar naive y aware y eliminar `tzinfo` para callar el error.
- Confundir adjuntar una zona con convertir a otra zona.
- Serializar una hora sin offset y asumir que todos interpretarán la misma zona.
- Confundir timestamp en segundos con milisegundos.
- Usar timestamp cuando también necesitas la zona civil original.
- Convertir una fecha parcial en una precisión que la fuente no proporcionó.
- Tratar `recorded_at` y `valid_time` como el mismo dato.

### 8.15 Aplicación en EIDOLON

- genera `recorded_at` como aware UTC desde un clock controlado;
- conserva `valid_time` con la precisión realmente conocida;
- serializa datetimes con ISO 8601 y offset explícito;
- conserva zona o expresión fuente cuando importen para interpretación;
- rechaza o pone en quarantine timestamps ambiguos según una política explícita;
- compara instantes después de normalizarlos a una referencia común, normalmente UTC.

### 8.16 Cuándo no usar `datetime`

Usa `date` para una fecha sin hora. Usa una duración (`timedelta`) para intervalos, no un datetime inventado. Conserva texto y metadata de precisión cuando la expresión temporal sea incierta (“a principios de mayo”). Un `datetime` exacto no vuelve exacta a la evidencia.

> **Antes de continuar:** resuelve el ejercicio guiado 7. Después comprueba con `.timestamp()` que la vista local y la vista UTC representan el mismo instante y explica por qué aún conviene conservar la zona fuente.

---

## 9. Caso progresivo integrado: una frontera de datos para EIDOLON

Construiremos un registro sintético sin clases ni framework. El objetivo no es definir todavía el modelo final, sino aplicar decisiones correctas de representación.

### 9.1 Entrada

```python
raw_text = "Cancio\u0301n en Mérida 🎵".encode("utf-8")
amount_text = "149.90"
local_time_text = "2026-08-25T14:00:00"
source_zone_name = "America/Merida"
```

La frontera declara:

- `raw_text` contiene UTF-8;
- `amount_text` es decimal exacto expresado como texto;
- `local_time_text` contiene hora civil naive;
- `source_zone_name` aporta la interpretación regional.

### 9.2 Decodificación y derivación para búsqueda

**Continuación:** ejecuta este bloque en la misma sesión que la entrada de 9.1.

```python
import unicodedata

source_text = raw_text.decode("utf-8")
search_text = unicodedata.normalize("NFC", source_text).casefold()

print(source_text == search_text)  # False: cumplen contratos distintos
```

No reemplazamos `source_text` con `search_text`.

### 9.3 Cantidad decimal

**Continuación:** reutiliza `amount_text` de 9.1.

```python
from decimal import Decimal

amount = Decimal(amount_text)
serialized_amount = str(amount)

print(serialized_amount)  # 149.90
```

### 9.4 Tiempo válido local y UTC

**Continuación:** reutiliza `local_time_text` y `source_zone_name` de 9.1.

```python
from datetime import UTC, datetime
from zoneinfo import ZoneInfo

naive_local = datetime.fromisoformat(local_time_text)
source_zone = ZoneInfo(source_zone_name)
aware_local = naive_local.replace(tzinfo=source_zone)
valid_instant_utc = aware_local.astimezone(UTC)

print(aware_local.isoformat())
print(valid_instant_utc.isoformat())
```

La asignación de zona es válida aquí porque el contrato declaró que los campos pertenecen a `America/Merida`. `valid_instant_utc` es otra representación del tiempo válido, no el momento de registro. Un sistema real debe decidir qué hacer con horas ambiguas o inexistentes.

### 9.5 Registro resultante

**Continuación:** reúne los valores derivados en 9.2–9.4.

```python
event = {
    "id": "evt-001",
    "source_text": source_text,
    "search_text": search_text,
    "amount_decimal": serialized_amount,
    "valid_time": aware_local.isoformat(),
    "valid_time_zone": source_zone_name,
    "valid_time_utc": valid_instant_utc.isoformat(),
}

print(event["source_text"])
print(event["valid_time_utc"])
```

El ejemplo no inventa `recorded_at`: esa marca debe provenir por separado del clock de ingesta cuando EIDOLON reciba el dato. Derivarla de `valid_time` haría que dos campos con significados distintos representaran el mismo hecho temporal.

Este ejemplo ya evita varios errores:

- no confunde bytes con texto;
- conserva Unicode original;
- separa representación derivada para búsqueda;
- no usa `float` para una cantidad decimal exacta;
- no persiste un datetime naive;
- conserva la zona civil y una vista UTC del tiempo válido;
- no confunde el tiempo válido con el momento de registro;
- usa un ID de dominio, no `id(event)`.

Todavía no resuelve schema validation, errores de parsing, persistencia JSONL, dataclasses ni provenance completa. Esos temas pertenecen a módulos posteriores.

### 9.6 Romperlo deliberadamente

Prueba estos cambios uno por uno y explica la falla antes de ejecutar:

1. reemplaza `.decode("utf-8")` por `str(raw_text)`;
2. elimina `source_text` y conserva solo `search_text`;
3. construye `Decimal(float(amount_text))`;
4. serializa `naive_local.isoformat()` como si fuera un instante;
5. usa `event_copy = event` y modifica `event_copy["id"]`;
6. compara el ID con `is`.

El objetivo es conectar cada bug con el modelo correspondiente, no solo restaurar el código original.

---

## 10. Ejercicios guiados con solución razonada

### Ejercicio guiado 1 — Predecir enlaces y mutación

Predice cada output:

```python
a = [1, 2]
b = a
c = a.copy()

b.append(3)
c.append(4)

print(a)
print(b)
print(c)
print(a is b)
print(a is c)
print(a == c)
```

#### Solución razonada

`b = a` crea un alias. `c = a.copy()` crea una lista exterior nueva con los mismos elementos, que aquí son enteros inmutables.

```text
[1, 2, 3]
[1, 2, 3]
[1, 2, 4]
True
False
False
```

`a` y `b` son el mismo objeto. `c` es distinto. Al final tampoco tiene el mismo valor porque recibió `4` en vez de `3`.

### Ejercicio guiado 2 — Detectar shallow copy insuficiente

```python
original = {"metadata": {"tags": ["a"]}}
candidate = original.copy()
candidate["metadata"]["tags"].append("b")

print(original)
```

#### Solución razonada

La shallow copy separó los diccionarios exteriores, pero `metadata` y `tags` siguen compartidos. El output incluye `"b"` en `original`.

Si la independencia completa es parte del contrato y la estructura solo contiene datos copiables:

```python
import copy

original = {"metadata": {"tags": ["a"]}}
candidate = copy.deepcopy(original)
candidate["metadata"]["tags"].append("b")

print(original)
# {'metadata': {'tags': ['a']}}
```

Para una operación de dominio, considera construir explícitamente el nuevo objeto en vez de duplicar todo el grafo.

### Ejercicio guiado 3 — Ausencia frente a cero

Repara el código para que `None` use el default y `0` se conserve:

```python
provided_limit = 0
limit = provided_limit or 20
print(limit)
```

#### Solución razonada

`or` selecciona el segundo operando cuando el primero es falsy; por eso reemplaza `0`. La pregunta correcta es por ausencia:

```python
provided_limit = 0
limit = 20 if provided_limit is None else provided_limit

print(limit)  # 0
```

### Ejercicio guiado 4 — Igualdad Unicode

Haz que dos variantes canónicamente equivalentes comparen igual sin destruir las fuentes:

```python
left = "Canción"
right = "Cancio\u0301n"
```

#### Solución razonada

Conserva `left` y `right`; deriva claves normalizadas:

```python
import unicodedata

left = "Canción"
right = "Cancio\u0301n"
left_key = unicodedata.normalize("NFC", left)
right_key = unicodedata.normalize("NFC", right)

print(left_key == right_key)  # True
print(left)                   # fuente intacta
print(right)                  # fuente intacta
```

### Ejercicio guiado 5 — Recuperar texto desde bytes

```python
payload = b"Tuxtla Guti\xc3\xa9rrez"
```

¿Por qué `str(payload)` no es la respuesta y cuál es la operación correcta?

#### Solución razonada

`str(payload)` crea una representación del objeto, incluyendo el prefijo `b` y escapes. El contrato declara UTF-8, así que debes decodificar:

```python
payload = b"Tuxtla Guti\xc3\xa9rrez"
text = payload.decode("utf-8")

print(text)  # Tuxtla Gutiérrez
```

### Ejercicio guiado 6 — Comparación numérica

El siguiente test falla:

```python
score = 0.1 + 0.2
assert score == 0.3
```

#### Solución razonada

Si `score` es una medida aproximada, usa una tolerancia justificada:

```python
import math

score = 0.1 + 0.2
assert math.isclose(score, 0.3, rel_tol=1e-9, abs_tol=0.0)
```

Si el dominio exige aritmética decimal exacta, cambia el tipo en la frontera:

```python
from decimal import Decimal

score = Decimal("0.1") + Decimal("0.2")
assert score == Decimal("0.3")
```

No elijas la segunda solución solo para hacer pasar el test; el contrato decide.

### Ejercicio guiado 7 — Instante reproducible

Convierte una hora civil de Mérida a UTC y serialízala con offset:

```python
local_text = "2026-08-25T14:00:00"
```

#### Solución razonada

```python
from datetime import UTC, datetime
from zoneinfo import ZoneInfo

local_text = "2026-08-25T14:00:00"
naive_local = datetime.fromisoformat(local_text)
aware_local = naive_local.replace(tzinfo=ZoneInfo("America/Merida"))
utc_instant = aware_local.astimezone(UTC)
serialized = utc_instant.isoformat()

assert utc_instant.tzinfo is not None and utc_instant.utcoffset() is not None
assert serialized.endswith("+00:00")
print(serialized)
```

La solución interpreta explícitamente los campos naive como hora civil de una zona conocida y luego convierte el instante a UTC. Si la fuente no declara la zona, no puedes deducirla con seguridad.

### Ejercicio guiado 8 — Correction sin mutar el evento

Repara este diseño:

```python
event = {"id": "evt-1", "text": "lunes"}
correction = event
correction["text"] = "martes"
```

#### Solución razonada

Una corrección es un objeto distinto y referencia el ID del evento:

```python
event = {"id": "evt-1", "text": "lunes"}
correction = {
    "id": "cor-1",
    "target_event_id": event["id"],
    "replacement_text": "martes",
}

assert event["text"] == "lunes"
assert correction["target_event_id"] == "evt-1"
```

Más adelante se formalizarán invariantes y tipos. Aquí demostramos que la operación no comparte el objeto mutable equivocado.

---

## 11. Ejercicios independientes

No consultes la solución de los ejercicios guiados hasta haber escrito una predicción y una prueba propia.

1. Crea tres nombres: dos deben apuntar a la misma lista y uno a una shallow copy. Diseña mutaciones que demuestren las identidades y valores finales.
2. Construye un diccionario con dos niveles anidados. Demuestra con `is` exactamente qué nivel comparte una shallow copy.
3. Modela un evento y una corrección con diccionarios separados. Agrega una lista de tags a cada uno sin aliasing accidental.
4. Escribe cinco valores falsy distintos y explica por qué un contrato podría necesitar distinguirlos.
5. Diseña una expresión que use un default solo cuando el valor sea `None`, conservando `0`, `False` y `""`.
6. Encuentra dos strings visualmente equivalentes con representaciones Unicode diferentes. Imprime sus code points y compáralos después de NFC.
7. Elige un emoji compuesto y compara su apariencia con `len()`. Explica por qué no implementarías un contador de “caracteres visibles” solo con `len`.
8. Codifica un texto español con emoji a UTF-8, imprime sus bytes y recupéralo. Verifica el round trip con `==`.
9. Produce deliberadamente un `UnicodeDecodeError` con bytes truncados. Captura el error sin descartar los bytes originales.
10. Compara `errors="strict"`, `"replace"` e `"ignore"` sobre el mismo payload. Escribe qué política usarías para un registro canónico.
11. Demuestra la diferencia entre `Decimal("0.1")` y `Decimal(0.1)`. Explica el origen, no solo el output.
12. Crea un cálculo con `float` que requiera `math.isclose`. Justifica `rel_tol` y `abs_tol` con una unidad concreta.
13. Representa por separado un cumpleaños, una hora civil local y un instante UTC usando los tipos apropiados.
14. Convierte un aware datetime entre `America/Merida` y UTC. Comprueba que ambos tienen el mismo `.timestamp()`.
15. Intenta ordenar un naive datetime y uno aware. Explica por qué eliminar `tzinfo` no es una corrección general.
16. Serializa y restaura un aware datetime con microsegundos. Verifica igualdad tras el round trip.
17. Construye un ejemplo en el que solo conoces la fecha del evento. Demuestra por qué no debes inventar medianoche.
18. Audita un script propio anterior y encuentra al menos una frontera donde el tipo usado no expresa completamente el contrato.

---

## 12. Preguntas conceptuales

Responde sin ejecutar código. Después usa un experimento mínimo para comprobar tu respuesta.

1. ¿Qué significa afirmar que un nombre está enlazado a un objeto?
2. ¿Cuál es la diferencia observable entre mutar un objeto y reasignar un nombre?
3. ¿Por qué `a = b` no constituye una copia?
4. ¿Qué pregunta responde `is` y qué pregunta responde `==`?
5. ¿Por qué el interning de strings o enteros no permite usar `is` para comparar valores?
6. ¿Qué es aliasing y cuándo puede ser deliberado?
7. ¿Qué separa una shallow copy y qué continúa compartiendo?
8. ¿Por qué `deepcopy` no conoce las invariantes de EIDOLON?
9. ¿Qué diferencia semántica existe entre `None`, `0`, `False` y una colección vacía?
10. ¿Cuándo es apropiado usar truthiness y cuándo destruye información?
11. ¿Por qué dos strings visualmente iguales pueden comparar diferente?
12. ¿Qué cuenta `len(str)` y por qué no siempre coincide con grafemas visibles?
13. ¿Cuál es la dirección de `encode` y cuál la de `decode`?
14. ¿Por qué un decoding que no lanza excepción todavía puede ser incorrecto?
15. ¿Por qué `errors="ignore"` es peligroso en una fuente canónica?
16. ¿Por qué `0.1 + 0.2 != 0.3` con `float`?
17. ¿En qué caso `math.isclose` sería una solución incorrecta?
18. ¿Por qué debe construirse `Decimal` desde texto cuando el input es decimal?
19. ¿Qué información representa `date` que no debería inflarse a `datetime`?
20. ¿Qué falta en un naive datetime para identificar un instante?
21. ¿Qué diferencia existe entre asignar una zona y convertir una zona?
22. ¿Qué representa un timestamp y qué información civil no conserva?
23. ¿Por qué `recorded_at` y `valid_time` no son intercambiables?
24. ¿Qué datos conservarías además de UTC para reconstruir cómo expresó el usuario una hora local?
25. ¿Qué decisiones de PF-M1 protegen el principio “los acontecimientos originales son inmutables; las interpretaciones pueden cambiar”?

---

## 13. Mini challenge — Ingesta segura de un evento sintético

### Objetivo

Construye un script ejecutable llamado `pf_m1_event_boundary.py`. No necesitas funciones ni clases; PF-M2 enseñará a refactorizarlo. El script debe transformar una entrada sintética en un registro seguro sin destruir la fuente.

Antes de comenzar, confirma que tu entorno puede cargar `ZoneInfo("America/Merida")`. Si una clave IANA válida produce `ZoneInfoNotFoundError`, documenta la precondición ambiental y configura datos de zona mediante el sistema o `tzdata`; no reemplaces la zona regional por un offset fijo inventado.

### Entrada obligatoria

```python
raw_text = b"Cancio\xcc\x81n en M\xc3\xa9rida \xf0\x9f\x8e\xb5"
amount_text = "149.90"
valid_time_text = "2026-08-25T14:00:00"
valid_time_zone = "America/Merida"
event_id = "evt-0001"
optional_confidence = 0.0
```

### Requisitos

1. Decodifica `raw_text` como UTF-8 con política estricta.
2. Conserva el texto fuente sin normalización destructiva.
3. Crea una clave de búsqueda con NFC y `casefold()`.
4. Convierte la cantidad a `Decimal` desde el texto y serialízala nuevamente como string.
5. Interpreta la hora civil en la zona indicada y produce una representación UTC con offset explícito.
6. Conserva también el nombre de zona original.
7. Conserva `optional_confidence = 0.0`; no lo reemplaces por un default debido a truthiness.
8. Construye un diccionario nuevo llamado `event`.
9. Construye una shallow copy llamada `view`, modifica solo un campo inmutable del contenedor exterior y demuestra que `event` no cambió en ese campo.
10. No uses `id(event)` como identificador ni `is` para comparar `event_id`.

### Contrato de salida

El diccionario debe contener como mínimo:

```text
id
source_text
search_text
amount_decimal
valid_time
valid_time_zone
valid_time_utc
confidence
```

### Comprobaciones mínimas

Incluye asserts equivalentes a:

```python
assert event["id"] == "evt-0001"
assert event["source_text"] == "Cancio\u0301n en Mérida 🎵"
assert event["search_text"] == "canción en mérida 🎵"
assert event["amount_decimal"] == "149.90"
assert event["confidence"] == 0.0
assert event["valid_time_zone"] == "America/Merida"
assert event["valid_time_utc"].endswith("+00:00")
assert view is not event
```

### Failure cases que debes provocar

Documenta el error observado y la política elegida para:

- un byte final truncado;
- `amount_text = "ciento cuarenta"`;
- una zona inexistente;
- una hora sin zona declarada;
- comparación del ID con `is`;
- un alias mutable creado deliberadamente.

No necesitas resolver todos los errores con una arquitectura de excepciones; eso corresponde a PF-M6. Sí debes evitar pérdida silenciosa y explicar qué dato conservarías para diagnosticar cada caso.

### Criterio de aprobación

Apruebas el challenge si otra persona puede ejecutar el script, obtener los asserts exitosos, reproducir los seis failure cases y escuchar una explicación correcta de cada frontera sin que leas una definición.

---

## 14. Resumen

- Python enlaza nombres con objetos; asignar no copia.
- Reasignar un nombre y mutar un objeto son operaciones diferentes.
- `is` compara identidad; `==` compara igualdad.
- `None` se compara con `is None` y no equivale semánticamente a todos los valores falsy.
- Aliasing comparte estado; puede ser intencional o un bug según ownership e invariantes.
- Una shallow copy separa el contenedor exterior; una deep copy intenta separar el grafo, pero no sustituye el diseño de dominio.
- `str` representa texto Unicode; `bytes` representa octetos.
- Encoding convierte `str` a `bytes`; decoding convierte `bytes` a `str`.
- UTF-8 es el default explícito recomendado para las fronteras de texto de este currículo.
- El texto original se conserva; las formas normalizadas son derivados con un propósito.
- `int` es exacto para enteros; `float` es aproximado; `Decimal` expresa aritmética decimal controlada.
- `date` representa una fecha; un aware `datetime` puede representar un instante; un naive datetime no basta para ello.
- UTC facilita comparación y persistencia, pero no sustituye la zona civil ni la expresión original cuando importan.
- La representación precisa no debe inventar precisión que la fuente no proporcionó.

---

## 15. Checklist de dominio

- [ ] Puedo explicar nombres, objetos, asignación y referencias sin recurrir a la metáfora de cajas como si fuera literal.
- [ ] Puedo predecir el resultado de una mutación cuando existen dos aliases.
- [ ] Puedo distinguir una mutación de una reasignación.
- [ ] Puedo explicar por qué `is` y `==` responden preguntas diferentes.
- [ ] Puedo detectar una comparación de valor implementada incorrectamente con `is`.
- [ ] Puedo explicar qué separa una shallow copy y qué conserva compartido.
- [ ] Puedo justificar cuándo `deepcopy` es apropiado y cuándo oculta un problema de diseño.
- [ ] Puedo distinguir `None`, cero, `False`, texto vacío y colección vacía cuando el contrato lo exige.
- [ ] Puedo evitar que `or` reemplace un valor falsy válido.
- [ ] Puedo explicar la diferencia entre code point, grafema, `str` y `bytes` a nivel práctico.
- [ ] Puedo demostrar dos representaciones Unicode canónicamente equivalentes.
- [ ] Puedo preservar el texto original y crear una vista normalizada para búsqueda.
- [ ] Puedo ejecutar un round trip `str → bytes → str` con UTF-8.
- [ ] Puedo provocar y diagnosticar `UnicodeEncodeError` y `UnicodeDecodeError`.
- [ ] Puedo justificar por qué `errors="ignore"` no es una política segura para datos canónicos.
- [ ] Puedo elegir `int`, `float` o `Decimal` según la semántica del dato.
- [ ] Puedo explicar el error de precisión de `float` sin decir simplemente “Python se equivoca”.
- [ ] Puedo usar `math.isclose` con una tolerancia justificada.
- [ ] Puedo construir `Decimal` desde texto y aplicar un redondeo explícito.
- [ ] Puedo distinguir `date`, naive datetime, aware datetime y timestamp.
- [ ] Puedo asignar una zona conocida y convertir el instante a UTC sin confundir ambas operaciones.
- [ ] Puedo serializar y restaurar un aware datetime con offset.
- [ ] Puedo detectar la mezcla insegura de naive y aware datetimes.
- [ ] Puedo justificar por qué `recorded_at` y `valid_time` deben separarse.
- [ ] Puedo implementar el mini challenge y explicar cada decisión.

---

## 16. Preparación para labs

Después de dominar PF-M1 debes poder comenzar:

- **PF-L02 — Unicode y tiempo hostiles:** es el laboratorio principal de este módulo. Debes conservar el original, producir derivados explícitos y probar acentos, emojis y timestamps aware.
- **PF-L01 — Diagnóstico reproducible:** PF-M1 aporta casos de predicción, debugging y explicación para tu baseline personal.
- **PF-L04 — Índice de entidades:** podrás reconocer identidad de dominio frente a identidad de objetos; la selección formal de `dict` y `set` llega en PF-M3.
- **PF-L11 — Repositorio JSONL versionado:** todavía no debes iniciarlo completo. PF-M1 prepara Unicode, tiempo y números; PF-M6 enseñará archivos, JSON, errores y lifecycle.
- **EIDOLON 0.0a:** este módulo cubre las fronteras de Unicode, timestamps, identidad y datos que el build deberá probar, pero no basta por sí solo para implementarlo.

Antes de avanzar a PF-M2, entrega como evidencia:

1. el script del mini challenge;
2. un archivo de predicciones y resultados para los ocho ejercicios guiados;
3. al menos seis ejercicios independientes resueltos;
4. una explicación oral de cinco minutos o una nota escrita equivalente sobre aliasing, Unicode y tiempo;
5. una nota breve titulada **“Qué no copiaría y por qué”** aplicada a un Event y una Correction.

## 17. Recursos de ampliación

La explicación fundamental está contenida en este módulo. Para profundizar, consulta selectivamente la [documentación oficial de Python 3.14](https://docs.python.org/3/contents.html), especialmente *Data Model*, tipos integrados, `copy`, `unicodedata`, `decimal`, `datetime` y `zoneinfo`. Los demás recursos del track se mantienen en [`PF.11 Recursos recomendados`](../../02_curriculum/01_programming_foundations.md#pf11-recursos-recomendados).

## 18. Límite del módulo

PF-M1 termina aquí. Funciones, contratos, LEGB y scopes pertenecen a **PF-M2** y no se desarrollan todavía. Tampoco se introducen dataclasses, validación de schemas, JSONL, pytest, backend, bases de datos ni AI.
