# PF-M5 — POO, dataclasses y type hints

**Track:** Programming Foundations  
**Competencias:** D1.1; refuerza D2.3, D3.1 y D11.1  
**Fase:** P0  
**Nivel objetivo:** Profesional para el alcance del módulo  
**Prerequisites:** PF-M1, PF-M2, PF-M3, PF-M4  
**Build:** EIDOLON 0.0a  
**Curriculum source:** [PF-M5](../../02_curriculum/01_programming_foundations.md#pf-m5--poo-dataclasses-y-type-hints)  
**Status:** review candidate

Un diccionario puede representar un event pequeño. El problema aparece cuando distintas partes del programa conocen sus keys, inventan variantes incompatibles y cambian estado sin conservar las reglas que lo hacían válido. Crear una clase tampoco resuelve eso automáticamente: una clase sin responsabilidad solo cambia corchetes por puntos.

PF-M5 estudia la programación orientada a objetos (object-oriented programming, OOP) como una herramienta para agrupar estado, comportamiento relacionado e invariantes. Después usa dataclasses para retirar boilerplate, type hints para hacer visibles contratos estáticos y composición (composition) para construir modelos pequeños sin una jerarquía anticipatoria.

El hilo conductor es:

```text
datos sueltos → estado con significado → invariantes → objetos pequeños
              → contratos tipados → composición
```

PF-M1–PF-M4 ya aportan objetos, identidad, mutabilidad, aliasing, funciones, contratos, colecciones, módulos y packages. Aquí se reutilizan; no se vuelven a enseñar. El módulo usa `ValueError` de forma mínima para hacer observable una precondición inválida, pero PF-M6 desarrollará la taxonomía y el manejo de excepciones. No hay archivos, JSON, persistencia, decorators generales, async, databases ni frameworks.

## Resultados de aprendizaje

Al terminar deberías poder:

- explicar cuándo estado y comportamiento comparten una responsabilidad y cuándo una función sigue siendo suficiente;
- distinguir class, instance, atributo de instancia, atributo de clase y method;
- predecir identidad, mutabilidad y aliasing entre referencias a una instancia;
- usar `__init__` para establecer un estado válido sin atribuirle validación automática;
- diseñar una interfaz pública pequeña y usar encapsulation para proteger invariantes, no para esconder todo;
- formular invariantes locales y distinguirlas de reglas que requieren otros objetos;
- detectar mutables compartidos accidentalmente como class attributes;
- elegir entre instance method, `classmethod`, `staticmethod`, property y función de módulo;
- explicar los costos de inheritance, overriding y `super()`;
- elegir composition o inheritance bajo relaciones `has-a` e `is-a` justificadas;
- distinguir value object, entity, identidad de runtime, identidad de dominio y equality;
- crear dataclasses con fields, defaults, `default_factory`, `field`, `__post_init__` y `frozen=True` deliberados;
- explicar qué generan las dataclasses y qué no validan;
- demostrar por qué `frozen=True` no produce inmutabilidad transitiva;
- escribir type hints modernos para funciones y colecciones sin anotar ruido innecesario;
- distinguir `X | None`, ausencia normal y fallo contractual;
- ejecutar un type checker y separar su diagnóstico del comportamiento runtime;
- distinguir type correctness, data validity y domain validity;
- reconocer cómo `Any` propaga pérdida de información;
- definir un `Protocol` pequeño y satisfacerlo mediante structural typing sin herencia explícita;
- modelar `SourceRef`, `Event`, `Claim` y `Correction` mediante dataclasses, Enum y composition;
- producir una Correction sin mutar ni sustituir silenciosamente el Event original.

## Cómo estudiar este módulo

Para cada ejemplo:

1. identifica qué objeto posee el estado;
2. escribe la invariante antes de mirar el constructor;
3. predice qué alias puede observar una mutación;
4. separa lo que comprueba Python runtime de lo que comprueba un type checker;
5. pregunta si una función de módulo expresaría mejor la operación;
6. ejecuta el ejemplo y conserva solo outputs estables;
7. cambia una regla y verifica que el objeto no pueda quedar en estado imposible.

No copies el modelo integrado como plantilla universal. Su propósito es enseñar decisiones pequeñas con datos sintéticos.

### Convenciones de código

- **Ejemplo ejecutable:** bloque autónomo o archivo completo dentro del árbol declarado.
- **Continuación:** reutiliza únicamente el bloque o árbol inmediatamente anterior.
- **Código incorrecto:** antipatrón deliberado cuyo defecto se explica.
- **Failure case:** debe producir el error o estado incorrecto indicado.
- **Fragmento:** muestra una firma o decisión con contexto omitido explícito.
- **Solución parcial:** resuelve el concepto local, pero no representa el modelo integrado completo.
- **Diagnóstico estático:** archivo destinado al type checker; puede ejecutar en runtime aunque contenga errores de tipos estáticos.

Los ejemplos se validan con Python 3.14. Los mensajes completos de excepciones y type checkers pueden variar; se fija el tipo de fallo o propiedad estable. `assert` se usa para comprobaciones pequeñas, no como introducción a pytest. Los decorators específicos `@dataclass`, `@classmethod`, `@staticmethod` y `@property` se explican por su contrato local; PF-M7 estudiará decorators como mecanismo general.

### Sintaxis de apoyo

- `ValueError` comunica una entrada que no satisface una invariante simple; no se enseña todavía recuperación con `try/except`;
- `datetime` y timestamps aware se reutilizan desde PF-M1 sin reabrir su teoría;
- `Enum`, `dataclass`, `field`, `Protocol` y `Any` pertenecen a la standard library;
- un type checker es development tooling y debe declararse en el extra de desarrollo enseñado en PF-M4;
- ellipsis (`...`) dentro de un método de `Protocol` declara la forma estática y no se ofrece como implementación runtime.

---

## 1. De datos sueltos a estado con significado

### 1.1 El punto de partida funciona

**Ejemplo ejecutable:**

```python
event = {
    "id": "evt-001",
    "text": "Llegué a casa",
    "status": "active",
}


def event_label(source_event):
    return source_event["id"] + ": " + source_event["text"]


print(event_label(event))
```

Output:

```text
evt-001: Llegué a casa
```

No hay un defecto por usar un dict. Para una transformación local y pequeña, la representación puede ser suficiente.

### 1.2 El costo aparece al dispersar la forma

Supón que varios módulos conocen las keys:

```text
cli.py         conoce id, text, status
summary.py     conoce id, status
correction.py  conoce id, text y otra key llamada state
```

Entonces pueden aparecer cuatro problemas:

- la representación interna se vuelve contrato de todo el programa;
- cualquier caller puede producir `{"id": "", "status": "deleted", "active": True}`;
- las reglas relacionadas quedan repartidas entre funciones y módulos;
- estructuras casi iguales usan `id`, `event_id`, `status` o `state` con significados distintos.

Una clase puede crear una frontera con nombre. No es automáticamente mejor: aporta valor cuando existe estado con significado, comportamiento estrechamente relacionado o invariantes que deben conservarse.

### 1.3 El modelo mental

```text
class     → define cómo se construyen y operan objetos de una categoría
instance  → objeto concreto creado a partir de esa class
attribute → nombre accesible a través del objeto o la class
method    → función definida en la class y enlazada a una instance o class
```

OOP agrupa potencialmente:

```text
estado + comportamiento relacionado + invariantes
```

No convierte cada sustantivo en class ni cada función en method.

### Predice

Si dos módulos reciben el mismo dict `event`, ¿crear una class impediría por sí sola que ambos compartan una referencia mutable?

### Explica

¿Qué problema resuelve nombrar una representación como `Event` que no resuelve añadir otra key al dict?

### Decide

Para normalizar un tag aislado sin estado, ¿elegirías una función o una class `TagNormalizer`? Justifica qué invariante poseería la class, si alguna.

### Modifica

Agrega una función que cambie `status` en el dict. Lista qué callers deben conocer la key y qué estado imposible podrían crear.

---

## 2. Classes, instances, attributes y methods

### 2.1 Una class mínima

**Ejemplo ejecutable:**

```python
class Event:
    def __init__(self, event_id, text, status):
        self.event_id = event_id
        self.text = text
        self.status = status

    def label(self):
        return self.event_id + ": " + self.text


event = Event("evt-001", "Llegué a casa", "active")
print(event.event_id)
print(event.label())
```

Output:

```text
evt-001
evt-001: Llegué a casa
```

`Event` es la class; `event` es una instance. Llamar `Event(...)` crea una instancia y después ejecuta `__init__` sobre ella. En este ejemplo, `__init__` enlaza tres atributos de instancia.

`self` no es una keyword especial: es el nombre convencional del primer parámetro de un instance method. Cuando escribes `event.label()`, Python enlaza la instance y el efecto conceptual es `Event.label(event)`.

**Ejemplo ejecutable — equivalencia observable:**

```python
class Event:
    def label(self):
        return self.event_id


event = Event()
event.event_id = "evt-001"

print(event.label())
print(Event.label(event))
```

Output:

```text
evt-001
evt-001
```

La asignación posterior es legal, pero deja claro por qué un constructor resulta útil: centraliza el estado inicial esperado.

### 2.2 Class e instance no son lo mismo

La class también es un objeto. El nombre `Event` apunta a ese objeto class; `Event(...)` produce otra instancia. Cada instancia puede poseer atributos distintos aunque comparta los métodos definidos en la class.

**Ejemplo ejecutable:**

```python
class Event:
    def __init__(self, event_id):
        self.event_id = event_id


first = Event("evt-001")
second = Event("evt-002")

print(first is second)
print(first.event_id)
print(second.event_id)
```

Output:

```text
False
evt-001
evt-002
```

### 2.3 Aliasing sigue existiendo

PF-M1 enseñó que asignar enlaza otro nombre; no copia. Eso también se aplica a instances.

**Ejemplo ejecutable:**

```python
class Event:
    def __init__(self, status):
        self.status = status


event = Event("active")
alias = event

alias.status = "corrected"

print(event is alias)
print(event.status)
```

Output:

```text
True
corrected
```

La class no elimina identidad, referencias ni mutabilidad. Solo puede ofrecer operaciones que hagan esas decisiones explícitas.

### Predice

¿Qué imprime el ejemplo si se cambia `alias = event` por `alias = Event(event.status)`? Separa identidad de igualdad de atributos.

### Explica

¿Qué hace `self` al llamar un method? ¿Por qué dos instances usan el mismo código de `label` pero leen atributos diferentes?

### Detecta el bug

Una función recibe `event` y `backup`, pero ambos nombres apuntan a la misma instance. Cambia `backup.status` esperando conservar un snapshot. Explica la falsa suposición.

### Comprueba

Imprime `id(event)` e `id(alias)` solo para confirmar el experimento local; no fijes esos números como output contractual.

---

## 3. Función, method o class

### 3.1 Una regla pura puede seguir siendo una función

**Ejemplo ejecutable:**

```python
def is_valid_transition(status, new_status):
    allowed = {
        "active": {"corrected", "archived"},
        "corrected": {"archived"},
        "archived": set(),
    }
    return new_status in allowed[status]


print(is_valid_transition("active", "corrected"))
print(is_valid_transition("archived", "active"))
```

Output:

```text
True
False
```

La función es explícita, pura y reutilizable. No necesita una instance.

### 3.2 Un method puede poseer la transición

**Ejemplo ejecutable:**

```python
class Event:
    def __init__(self, event_id, status):
        self.event_id = event_id
        self.status = status

    def can_transition_to(self, new_status):
        allowed = {
            "active": {"corrected", "archived"},
            "corrected": {"archived"},
            "archived": set(),
        }
        return new_status in allowed[self.status]


event = Event("evt-001", "active")
print(event.can_transition_to("corrected"))
```

Output:

```text
True
```

Esta forma comunica que la consulta pertenece al estado actual del Event. Todavía no muta nada.

### 3.3 Tradeoff real

Prefiere considerar una class cuando:

- varias operaciones comparten un estado con nombre;
- existe una invariante que debe mantenerse entre operaciones;
- ocultar un detalle reduce acoplamiento real;
- múltiples callers necesitan el mismo contrato de objeto.

Prefiere una función cuando:

- transforma inputs en output sin poseer estado;
- la regla es reutilizable fuera de una entidad;
- convertirla en method solo crea ceremony;
- una class existiría únicamente para llamar un método una vez.

`is_valid_transition(status, new_status)` facilita probar la regla aislada. `event.can_transition_to(new_status)` comunica ownership. También puede existir una combinación: una función pura define la política y el method delega en ella si eso mejora reutilización sin duplicación.

### Decide

¿`normalize_tag(text)` debería ser función, instance method de `Event` o class propia? Responde según el estado que usa y la invariante que protege.

### Modifica

Extrae el dict `allowed` a una constante de módulo y conserva ambos estilos sin duplicar la regla.

### Explica

¿Qué información añade `event.can_transition_to(...)` y qué acoplamiento añade respecto a la función pura?

---

## 4. Encapsulation: proteger reglas, no esconder todo

### 4.1 Interfaz pública y detalle interno

Encapsulación (encapsulation) significa que los callers dependen de una interfaz pequeña y que el objeto conserva sus invariantes detrás de ella. No significa “hacer privado cada atributo”.

En Python:

- `event.status` comunica una parte pública del contrato;
- `event._status` comunica por convención que es un detalle interno;
- `event.__status` activa name mangling, no privacidad estricta.

**Ejemplo ejecutable:**

```python
class Event:
    def __init__(self, event_id):
        self.event_id = event_id
        self._status = "active"

    def archive(self):
        self._status = "archived"

    def status_label(self):
        return self._status.upper()


event = Event("evt-001")
event.archive()
print(event.status_label())
```

Output:

```text
ARCHIVED
```

El underscore pide a los callers usar la interfaz, pero `event._status = "invented"` sigue siendo técnicamente posible. La protección real proviene de contratos claros, diseño, type checking y comprobaciones; no de una barrera de seguridad.

### 4.2 Name mangling al nivel necesario

**Ejemplo ejecutable:**

```python
class Event:
    def __init__(self):
        self.__status = "active"

    def status(self):
        return self.__status


event = Event()
print(event.status())
print(hasattr(event, "__status"))
print(hasattr(event, "_Event__status"))
```

Output:

```text
active
False
True
```

El nombre se transforma para reducir colisiones accidentales, especialmente con subclasses. No protege secrets ni vuelve imposible el acceso. Usarlo para imitar `private` de Java añade complejidad sin garantizar una invariante.

### 4.3 Evitar getters y setters triviales

**Código innecesario:**

```python
class Event:
    def get_text(self):
        return self._text

    def set_text(self, text):
        self._text = text
```

Si no existe cálculo, validación, compatibilidad o política, `event.text` comunica mejor. Una operación nombrada aporta valor cuando expresa dominio: `archive()`, `correct_with(...)` o `can_transition_to(...)`.

### Predice

¿Puede código externo asignar `event._status`? ¿Qué cambia la convención y qué no cambia el runtime?

### Explica

¿Por qué encapsulation reduce acoplamiento aunque Python no tenga privacidad estricta?

### Detecta el diseño deficiente

Una class contiene veinte getters y setters que solo leen o asignan fields. Identifica qué invariantes protegen realmente y elimina ceremony donde no exista ninguna.

### Decide

¿Un method `set_status("archived")` comunica mejor que `archive()`? Considera qué estados debería aceptar cada interfaz.

---

## 5. Invariantes: el centro del modelo

### 5.1 Qué es una invariante

Una **invariante** es una condición que debe permanecer verdadera para que el objeto represente un estado válido.

```text
constructor → objeto válido → operación permitida → objeto sigue válido
```

Ejemplos locales para EIDOLON:

- un ID es un string no vacío;
- un status pertenece al conjunto permitido;
- un timestamp temporal es aware;
- `active` y `deleted` no pueden ser flags independientes que resulten ambos verdaderos.

Ejemplos que necesitan contexto externo:

- una Correction referencia un Event o Claim que existe;
- un ID no está duplicado en el store;
- una transición es autorizada por una policy externa.

Un constructor puede asegurar invariantes locales. No puede probar por sí solo todo el sistema sin recibir las dependencias necesarias.

### 5.2 Construir solo estados locales válidos

**Ejemplo ejecutable:**

```python
class Event:
    VALID_STATUSES = {"active", "corrected", "archived"}

    def __init__(self, event_id, status):
        if type(event_id) is not str or not event_id:
            raise ValueError("event_id must be a non-empty str")
        if status not in self.VALID_STATUSES:
            raise ValueError("unsupported status")

        self.event_id = event_id
        self._status = status

    def archive(self):
        self._status = "archived"


event = Event("evt-001", "active")
event.archive()
print(event._status)
```

Output:

```text
archived
```

El constructor establece un estado válido y `archive` conserva el conjunto permitido. El ejemplo usa `ValueError` únicamente como señal observable; PF-M6 enseñará cómo clasificar, propagar y traducir errores.

**Failure case — ID vacío:**

```python
class Event:
    def __init__(self, event_id):
        if type(event_id) is not str or not event_id:
            raise ValueError("event_id must be a non-empty str")
        self.event_id = event_id


Event("")
```

Debe terminar con `ValueError`. La causa estable es que el input no pertenece al conjunto de IDs válidos; la corrección es proporcionar un ID válido o rechazarlo en la frontera que crea el objeto, no construir y reparar después.

### 5.3 Flags que permiten combinaciones imposibles

**Código incorrecto:**

```python
class Event:
    def __init__(self, active, deleted):
        self.active = active
        self.deleted = deleted


event = Event(active=True, deleted=True)
```

La representación admite cuatro combinaciones aunque el dominio quizá permita solo `active`, `corrected` y `archived`. Un único estado enumerado puede reducir el espacio de estados posibles. Enum se introduce en la sección 21.

### 5.4 Validación local frente a consistencia entre objetos

Una `Correction(target_id="evt-999")` puede tener un target ID no vacío y seguir refiriendo algo inexistente. El objeto conoce su forma local; el store o la operación que reúne ambos objetos debe comprobar existencia. Meter todos los Events globales dentro del constructor de Correction acoplaría el value object al estado completo de la aplicación.

### Predice

¿Qué pasa si `archive()` asigna `"deleted"` aunque el constructor lo prohíba? ¿Dónde se rompió la cadena de invariantes?

### Explica

Separa una invariante comprobable dentro de `Event` de una regla que necesita un índice externo.

### Detecta el bug

Una class valida `status` en `__init__`, pero publica `event.status` mutable. Muestra cómo un caller puede invalidarla después.

### Modifica

Haz que el status se lea mediante una property sin setter y que las transiciones ocurran mediante operaciones nombradas.

---

## 6. Atributos de class y de instance

### 6.1 Lookup conceptual

Un atributo asignado mediante `self.name` suele vivir en la instance. Un atributo escrito en el cuerpo de la class vive en la class y puede ser visible a sus instances mediante attribute lookup.

**Ejemplo ejecutable:**

```python
class Event:
    kind = "event"

    def __init__(self, event_id):
        self.event_id = event_id


first = Event("evt-001")
second = Event("evt-002")

print(first.kind)
print(second.kind)
print(first.event_id)
```

Output:

```text
event
event
evt-001
```

El lookup busca primero un atributo de instance apropiado y luego puede encontrarlo en la class y sus bases. Asignar `first.kind = "special"` crea un atributo de instance que oculta el de class para `first`; no cambia `second.kind` ni `Event.kind`.

### 6.2 Failure case: mutable compartido en la class

**Código incorrecto ejecutable:**

```python
class Event:
    tags = []

    def __init__(self, event_id):
        self.event_id = event_id


first = Event("evt-001")
second = Event("evt-002")

first.tags.append("home")

print(first.tags)
print(second.tags)
print(first.tags is second.tags)
```

Output:

```text
['home']
['home']
True
```

Ninguna instance creó su propia list; ambas resolvieron `tags` en la class. Es el mismo modelo de referencia y mutabilidad de PF-M1.

**Ejemplo ejecutable — atributo por instance:**

```python
class Event:
    def __init__(self, event_id):
        self.event_id = event_id
        self.tags = []


first = Event("evt-001")
second = Event("evt-002")
first.tags.append("home")

print(first.tags)
print(second.tags)
print(first.tags is second.tags)
```

Output:

```text
['home']
[]
False
```

### Predice

Después de `first.kind = "special"`, predice `first.kind`, `second.kind` y `Event.kind`.

### Explica

¿Por qué una constante string en la class es razonable mientras una list mutable suele ser peligrosa?

### Detecta el bug

Una class usa `seen_ids = set()` para “ahorrar memoria”. Explica qué tests pueden contaminarse entre instances.

### Modifica

Mueve `seen_ids` a `__init__` y demuestra con `is` que cada instance posee un set distinto.

---

## 7. Instance methods, classmethods y staticmethods

### 7.1 Instance method

Un instance method necesita el estado de una instance o realiza una operación que le pertenece.

**Ejemplo ejecutable:**

```python
class SourceRef:
    def __init__(self, source_id):
        self.source_id = source_id

    def label(self):
        return "source:" + self.source_id


print(SourceRef("src-001").label())
```

Output:

```text
source:src-001
```

### 7.2 `@classmethod`: construcción ligada a la class

`classmethod` recibe la class como primer parámetro convencional `cls`. Un uso común es un constructor alternativo que conserva subclasses posibles.

**Ejemplo ejecutable:**

```python
class SourceRef:
    def __init__(self, source_id):
        if not source_id:
            raise ValueError("source_id must not be empty")
        self.source_id = source_id

    @classmethod
    def from_namespace(cls, namespace, local_id):
        return cls(namespace + ":" + local_id)


source = SourceRef.from_namespace("journal", "src-001")
print(source.source_id)
```

Output:

```text
journal:src-001
```

No uses `classmethod` solo para mover una función dentro de la class. Debe existir una relación real con construcción o estado de class.

### 7.3 `@staticmethod`: función namespaced sin `self` ni `cls`

**Ejemplo ejecutable:**

```python
class SourceRef:
    @staticmethod
    def has_supported_prefix(source_id):
        return source_id.startswith("src-")


print(SourceRef.has_supported_prefix("src-001"))
```

Output:

```text
True
```

Esta función podría ser perfectamente una función de módulo. `staticmethod` solo es mejor si el namespace de la class ayuda a descubrir el contrato. No es un requisito estilístico.

### Decide

Clasifica cada operación: `event.label()`, `Event.from_dict(...)`, `is_supported_event_id(text)`. ¿Instance method, classmethod, staticmethod o función de módulo? No uses `from_dict` todavía: files/JSON y validación externa llegan después.

### Explica

¿Qué binding automático recibe cada tipo de method?

### Detecta el diseño innecesario

Una class contiene diez staticmethods sin estado y nunca se instancia. Decide si una module boundary expresaría mejor la cohesión.

---

## 8. Properties sin ceremonia de getters/setters

### 8.1 Valor derivado

Una property permite exponer una operación sin argumentos mediante acceso de atributo. Es apropiada para un valor derivado barato y sin side effects sorprendentes.

**Ejemplo ejecutable:**

```python
class Event:
    def __init__(self, event_id, text):
        self.event_id = event_id
        self.text = text

    @property
    def label(self):
        return self.event_id + ": " + self.text


event = Event("evt-001", "Llegué a casa")
print(event.label)
```

Output:

```text
evt-001: Llegué a casa
```

### 8.2 Lectura estable, mutación controlada

**Ejemplo ejecutable:**

```python
class Event:
    def __init__(self):
        self._status = "active"

    @property
    def status(self):
        return self._status

    def archive(self):
        self._status = "archived"


event = Event()
event.archive()
print(event.status)
```

Output:

```text
archived
```

No definir setter obliga a usar operaciones nombradas para cambios normales. No crea una frontera de seguridad: `_status` sigue accesible por convención.

### 8.3 Cuándo no usar property

Evítala cuando:

- realiza I/O, espera, muta estado o puede ser muy costosa;
- oculta un error complejo que un method haría visible;
- solo recrea `get_name`/`set_name` sin política;
- el valor necesita parámetros.

Una property puede mantener una API estable si un atributo almacenado pasa a ser derivado, pero no justifica ocultar trabajo sorprendente.

### Predice

¿Qué ocurre con `event.status = "invented"` cuando la property no tiene setter? Identifica el tipo general de fallo sin memorizar el mensaje.

### Explica

¿Por qué `event.label` puede ser property mientras `event.transition_to(new_status)` debe ser method?

### Decide

¿`event.age_in_days` debería ser property si depende del clock actual? Considera determinismo y dependencia oculta.

---

## 9. Inheritance: relación `is-a` y acoplamiento

### 9.1 Base class, subclass y overriding

Herencia (inheritance) permite que una subclass reutilice y especialice comportamiento de una base class. Expresa una relación `is-a` cuando la subclass puede usarse donde se espera la base sin romper su contrato.

**Ejemplo ejecutable:**

```python
class Record:
    def __init__(self, record_id):
        self.record_id = record_id

    def label(self):
        return "record:" + self.record_id


class Event(Record):
    def __init__(self, record_id, text):
        super().__init__(record_id)
        self.text = text

    def label(self):
        return "event:" + self.record_id + ":" + self.text


record = Event("evt-001", "Llegué")
print(record.label())
print(isinstance(record, Record))
```

Output:

```text
event:evt-001:Llegué
True
```

`super()` delega en el comportamiento cooperativo de la base según el method resolution order. PF-M5 solo necesita la cadena simple mostrada; multiple inheritance y MRO profundo quedan fuera.

### 9.2 El costo

Una subclass depende de:

- el contrato público de la base;
- assumptions sobre inicialización y methods que puede override;
- cambios futuros de la jerarquía;
- la promesa de sustitución `is-a`.

Reutilizar tres líneas no basta para justificar ese acoplamiento.

### 9.3 Jerarquía rígida

**Código deficiente:**

```python
class JournalItem:
    def publish(self):
        return "published"


class Event(JournalItem):
    pass


class Correction(JournalItem):
    pass
```

Si `Correction` no debe “publicarse” igual que Event, la base impuso comportamiento por conveniencia. Añadir overrides que lanzan errores revela que la relación `is-a` era falsa.

### 9.4 Cuándo inheritance puede ser clara

Puede ser razonable si:

- existe un contrato estable y pequeño;
- la relación `is-a` es verdadera para todo el comportamiento público;
- sustituir subclass por base conserva expectativas;
- el shared behavior no cambia por combinaciones independientes.

No es necesario prohibir inheritance; es necesario pagar su acoplamiento solo cuando modela mejor el dominio.

### Predice

¿Qué method se ejecuta en `record.label()` y qué aporta `super().__init__`?

### Explica

¿Por qué “Event reutiliza código de Record” es una justificación más débil que “Event satisface el contrato de Record”?

### Detecta el bug de diseño

Una subclass overridea cinco methods solo para lanzar `NotImplementedError`. Explica qué dice eso sobre sustitución.

### Decide

¿`Correction` es un tipo de `Event`, o referencia un Event? Dibuja `is-a` y `has-a` antes de elegir.

---

## 10. Composition: relación `has-a`

### 10.1 Construir con objetos pequeños

Composición (composition) expresa que un objeto **tiene** otro objeto o comportamiento. No hereda toda su interfaz.

**Ejemplo ejecutable:**

```python
class SourceRef:
    def __init__(self, source_id):
        self.source_id = source_id


class Event:
    def __init__(self, event_id, text, source):
        self.event_id = event_id
        self.text = text
        self.source = source


source = SourceRef("src-001")
event = Event("evt-001", "Llegué", source)

print(event.source.source_id)
```

Output:

```text
src-001
```

Event **has-a** SourceRef. Decir que Event **is-a** SourceRef no tendría sentido.

### 10.2 Componer comportamiento intercambiable

**Ejemplo ejecutable:**

```python
class KeepOriginalText:
    def normalize(self, text):
        return text


class LowercaseText:
    def normalize(self, text):
        return text.lower()


class EventLabeler:
    def __init__(self, text_policy):
        self.text_policy = text_policy

    def label(self, event_id, text):
        normalized = self.text_policy.normalize(text)
        return event_id + ": " + normalized


labeler = EventLabeler(LowercaseText())
print(labeler.label("evt-001", "Llegué A Casa"))
```

Output:

```text
evt-001: llegué a casa
```

El comportamiento cambia al reemplazar un componente, sin crear `LowercaseEventLabeler`, `KeepOriginalEventLabeler` y futuras combinaciones. Todavía debes preguntar si dos funciones serían más simples; composition no exige objetos para cada estrategia.

### 10.3 Composition frente a inheritance

| Pregunta | Composition | Inheritance |
|---|---|---|
| Relación principal | `has-a` / delega | `is-a` / sustituye |
| Reemplazo de comportamiento | cambia componente | crea/elige subclass |
| Acoplamiento | interfaz del componente | contrato y estructura de la base |
| Riesgo típico | demasiados objetos diminutos | jerarquía rígida |
| Cuándo ayuda | capacidades combinables | taxonomía estable y sustituible |

“Composition suele evolucionar mejor” es una heurística, no una ley. Una composition con seis wrappers sin responsabilidad también puede ser peor que una base class pequeña.

### Predice

¿Qué cambia si `EventLabeler` recibe `KeepOriginalText()`? ¿Debe modificarse la class `EventLabeler`?

### Explica

¿Por qué SourceRef es composición de Event y no inheritance?

### Decide

Para agregar una policy de redacción opcional, compara una subclass por combinación contra un objeto policy pequeño o una función.

### Modifica

Implementa `StripText` con `normalize` y úsalo sin modificar `EventLabeler`.

---

## 11. Value objects, entities e identidad

### 11.1 Value object

Un **value object** se distingue principalmente por su valor. Dos referencias con los mismos componentes representan el mismo valor bajo el contrato elegido.

Ejemplos potenciales:

- `SourceRef` compuesto por namespace e ID;
- un identificador tipado;
- un rango temporal pequeño.

Suele ser buen candidato para equality por fields y mutabilidad restringida.

### 11.2 Entity

Una **entity** conserva identidad de dominio aunque otros atributos cambien. Un Event con `event_id="evt-001"` puede recibir una interpretación o estado posterior y seguir refiriendo el mismo acontecimiento de dominio.

No confundas:

```text
identidad de runtime → ¿es la misma instance? (`is`)
equality             → ¿son iguales según `==`?
identidad de dominio → ¿representan la misma entidad? (por ejemplo, mismo event_id)
```

**Ejemplo ejecutable:**

```python
class Event:
    def __init__(self, event_id, text):
        self.event_id = event_id
        self.text = text


first_snapshot = Event("evt-001", "Llegué")
second_snapshot = Event("evt-001", "Llegué a casa")

print(first_snapshot is second_snapshot)
print(first_snapshot.event_id == second_snapshot.event_id)
```

Output:

```text
False
True
```

La igualdad default de una class normal sigue siendo identidad de runtime a menos que la redefinas. Una dataclass genera otra política por default: compara fields. Ninguna de las dos decide automáticamente identidad de dominio.

### 11.3 No es DDD avanzado

PF-M5 usa value object y entity para razonar sobre equality, mutabilidad e identidad. No introduce aggregates, bounded contexts, domain services ni repositories persistentes.

### Predice

Dos dataclasses Event tienen el mismo `event_id` y distinto `text`. ¿Podrían ser la misma entity de dominio aunque `==` devuelva `False`?

### Explica

¿Por qué `is` no debe usarse para decidir que dos snapshots representan el mismo Event?

### Decide

¿SourceRef se entiende mejor como value object o entity? Escribe qué cambio preservaría o alteraría su significado.

---
## 12. Dataclasses: retirar boilerplate sin retirar decisiones

### 12.1 El problema que resuelven

Una class usada principalmente para conservar fields suele repetir inicialización, representación y equality.

**Ejemplo ejecutable — implementación manual:**

```python
class SourceRef:
    def __init__(self, source_id):
        self.source_id = source_id

    def __repr__(self):
        return f"SourceRef(source_id={self.source_id!r})"

    def __eq__(self, other):
        return (
            type(other) is SourceRef
            and self.source_id == other.source_id
        )


first = SourceRef("src-001")
second = SourceRef("src-001")

print(first)
print(first == second)
```

Output:

```text
SourceRef(source_id='src-001')
True
```

Ese código puede ser correcto. El problema es repetirlo para objetos cuyo contrato ya se expresa mediante fields.

### 12.2 La versión dataclass

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass
class SourceRef:
    source_id: str


first = SourceRef("src-001")
second = SourceRef("src-001")

print(first)
print(first == second)
```

Output:

```text
SourceRef(source_id='src-001')
True
```

`@dataclass` inspecciona los fields anotados y, según sus opciones, genera methods como `__init__`, `__repr__` y `__eq__`. La anotación `source_id: str` participa en ese contrato y en static typing.

No genera automáticamente:

- validación runtime del tipo o del dominio;
- copia defensiva de objetos mutables;
- identidad de dominio;
- persistencia o parsing;
- una arquitectura adecuada.

### 12.3 Equality generada

Por default, dos instances de la misma dataclass comparan los valores de todos sus fields marcados para comparación. Esto suele encajar con value objects. Para entities puede comparar snapshots completos cuando la identidad de dominio depende solo del ID.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass
class Event:
    event_id: str
    text: str


before = Event("evt-001", "Llegué")
after = Event("evt-001", "Llegué a casa")

print(before == after)
print(before.event_id == after.event_id)
```

Output:

```text
False
True
```

No cambies `__eq__` hasta escribir la pregunta que debe responder. Para este módulo, compara IDs explícitamente cuando la pregunta sea identidad de dominio.

### Predice

¿Qué fields aparecen en el `repr` generado y cuáles participan en `==` por default?

### Explica

¿Qué boilerplate retira `@dataclass` y qué decisiones de modelado siguen siendo tuyas?

### Detecta el error conceptual

Un equipo afirma que `source_id: str` rechazará automáticamente un entero. Explica qué componente podría detectarlo y qué hace Python runtime.

### Modifica

Agrega `namespace: str = "journal"` después del field sin default y observa la firma generada.

---

## 13. Fields, defaults y `default_factory`

### 13.1 Defaults por valor

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass
class Claim:
    claim_id: str
    statement: str
    confidence: float = 1.0


claim = Claim("clm-001", "La puerta estaba abierta")
print(claim.confidence)
```

Output:

```text
1.0
```

Como en las funciones, un field sin default debe aparecer antes que uno con default en la firma generada.

### 13.2 Failure case: mutable default

**Failure case — definición rechazada:**

```python
from dataclasses import dataclass


@dataclass
class Event:
    event_id: str
    tags: list[str] = []
```

Python moderno rechaza esta definición con `ValueError` porque la list no debe usarse como default compartido de un field. El mensaje recomienda `default_factory`.

La regla no es magia de dataclasses. Una list creada una sola vez y reutilizada por todas las instances produciría el mismo aliasing visto en los class attributes.

### 13.3 Una fábrica por instance

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass, field


@dataclass
class Event:
    event_id: str
    tags: list[str] = field(default_factory=list)


first = Event("evt-001")
second = Event("evt-002")
first.tags.append("home")

print(first.tags)
print(second.tags)
print(first.tags is second.tags)
```

Output:

```text
['home']
[]
False
```

`default_factory=list` conserva la función `list` y la llama sin argumentos al crear cada instance. El resultado es una list nueva por objeto.

No escribas `default_factory=list()`; eso ejecutaría la fábrica durante la definición y pasaría una list, no un callable.

### 13.4 `field(...)` como configuración local

`field` permite configurar el field generado. Opciones comunes:

- `default` o `default_factory`;
- `repr=False` para omitir un field del `repr` generado;
- `compare=False` para excluirlo de equality/order;
- `init=False` para no recibirlo en el constructor generado.

Cada opción cambia un contrato. Ocultar un field del `repr` no lo cifra ni protege; excluirlo de equality puede ser incorrecto si afecta el valor.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass, field


@dataclass
class EventSummary:
    event_id: str
    cached_label: str = field(default="", repr=False, compare=False)


summary = EventSummary("evt-001", "computed")
print(summary)
```

Output:

```text
EventSummary(event_id='evt-001')
```

El cache no participa en el valor lógico elegido para este ejemplo. Si esa decisión no puede justificarse, conserva el default de dataclass.

### Predice

¿Cuántas lists crea `default_factory=list` al construir tres Events? ¿Cuándo se llama la fábrica?

### Explica

Conecta el failure case con evaluación de defaults y aliasing ya vistos en PF-M1/PF-M2.

### Detecta el bug

```python
field(default_factory=list())
```

¿Qué objeto recibe `default_factory` y por qué no cumple su contrato?

### Decide

¿Un `tags: tuple[str, ...] = ()` necesita default factory? Razona desde mutabilidad y reutilización del objeto.

---

## 14. `frozen=True`: asignación restringida, no inmutabilidad perfecta

### 14.1 Qué ofrece

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class SourceRef:
    source_id: str

    def __post_init__(self) -> None:
        if type(self.source_id) is not str or not self.source_id:
            raise ValueError("source_id must be a non-empty str")


source = SourceRef("src-001")
print(source.source_id)
```

Output:

```text
src-001
```

Una frozen dataclass impide la asignación normal y el borrado de sus fields después de construirla. Esto comunica que el valor no debe cambiar mediante su API ordinaria y puede encajar con un value object.

**Failure case — reasignación:**

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class SourceRef:
    source_id: str


source = SourceRef("src-001")
source.source_id = "src-002"
```

Debe terminar con `FrozenInstanceError`, una forma específica de `AttributeError` de dataclasses. No hace falta capturarla en PF-M5; el ejemplo demuestra el contrato.

### 14.2 Lo que no ofrece

`frozen=True` no vuelve transitivamente inmutables los objetos contenidos.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass, field


@dataclass(frozen=True)
class FrozenLabelSet:
    labels: list[str] = field(default_factory=list)


labels = FrozenLabelSet()
labels.labels.append("home")
print(labels.labels)
```

Output:

```text
['home']
```

El field no se reasignó; la list referenciada cambió internamente. Si el valor exige inmutabilidad transitiva para este nivel, usa componentes inmutables como `tuple[str, ...]` y evita exponer mutables internos.

### 14.3 Hashing no es automático para cualquier contenido

Una frozen dataclass puede recibir `__hash__` generado según sus opciones, pero hacer hash sigue requiriendo que los fields participantes sean hashable. Una list interior puede permitir la construcción y luego causar `TypeError` al ejecutar `hash(instance)`.

No marques una dataclass frozen solo para usarla como dict key. Empieza por el contrato de valor e identifica mutabilidad y hashability de cada componente.

### 14.4 Tradeoffs

`frozen=True` ayuda cuando:

- un value object debe conservar su valor;
- reemplazar el objeto completo es más claro que mutarlo;
- equality por fields describe el contrato.

Puede estorbar cuando:

- una entity posee lifecycle mutable deliberado;
- cada cambio exige copiar estructuras grandes sin beneficio;
- contiene mutables que vuelven engañosa la promesa.

### Predice

¿Por qué `labels.labels.append(...)` funciona pero `labels.labels = []` no?

### Explica

Define “inmutabilidad transitiva” con un grafo de dos objetos.

### Detecta el bug

Una frozen dataclass contiene `metadata: dict[str, str]` y se anuncia como “completamente inmutable”. Construye un contraejemplo.

### Decide

¿Event debería ser frozen en un modelo que representa snapshots? ¿Y en uno que modela una entity mutable? Expón ambos contratos.

---

## 15. `__post_init__` e invariantes simples

### 15.1 Validar después del `__init__` generado

Dataclass genera `__init__` y después llama `__post_init__`, si existe. Es un lugar apropiado para comprobar invariantes locales o derivar un field sencillo.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class SourceRef:
    source_id: str

    def __post_init__(self):
        if type(self.source_id) is not str or not self.source_id:
            raise ValueError("source_id must be a non-empty str")


source = SourceRef("src-001")
print(source)
```

Output:

```text
SourceRef(source_id='src-001')
```

**Failure case:**

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class SourceRef:
    source_id: str

    def __post_init__(self):
        if type(self.source_id) is not str or not self.source_id:
            raise ValueError("source_id must be a non-empty str")


SourceRef("")
```

Debe terminar con `ValueError`. La dataclass no produjo esa regla; tú la escribiste.

### 15.2 Frozen y fields derivados

Dentro de `__post_init__`, una frozen dataclass no puede asignar normalmente. `object.__setattr__` permite establecer un field derivado durante construcción, pero úsalo con moderación: evita que `frozen=True` sea una promesa falsa.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass, field


@dataclass(frozen=True)
class SourceRef:
    source_id: str
    label: str = field(init=False)

    def __post_init__(self):
        if not self.source_id:
            raise ValueError("source_id must not be empty")
        object.__setattr__(self, "label", "source:" + self.source_id)


source = SourceRef("src-001")
print(source.label)
```

Output:

```text
source:src-001
```

Si el valor se calcula barato y siempre puede derivarse, una property puede evitar almacenar estado duplicado.

### 15.3 Validación externa sigue fuera

`__post_init__` recibe valores ya proporcionados a Python. No prueba que un payload externo tenga schema correcto, no convierte formatos y no conserva evidencia de parsing. PF-M6 tratará esas fronteras con errores y datos serializados.

### Predice

¿En qué orden ocurren asignación de fields y `__post_init__` en la construcción generada?

### Explica

¿Por qué una anotación y una comprobación en `__post_init__` tienen responsabilidades distintas?

### Decide

Para `label = "source:" + source_id`, compara field derivado con property. ¿Qué estado duplicado aparece?

---

## 16. Ordering, slots y otras opciones con moderación

### 16.1 `order=True` necesita significado

`@dataclass(order=True)` genera comparaciones lexicográficas por fields, en su orden. Eso no demuestra que el dominio tenga un orden total útil.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(order=True, frozen=True)
class SequencePosition:
    value: int


positions = [SequencePosition(3), SequencePosition(1)]
print(sorted(positions))
```

Output:

```text
[SequencePosition(value=1), SequencePosition(value=3)]
```

Aquí el orden numérico es parte del value object. En Event, ordenar por `event_id` y luego `text` solo porque ese es el orden de fields puede ser una política accidental. Usa una `key=` explícita cuando la pregunta de orden sea externa.

### 16.2 `slots=True` al nivel necesario

`@dataclass(slots=True)` genera slots para los fields en vez de depender del `__dict__` de instance habitual. Puede reducir espacio y evita agregar atributos arbitrarios no declarados en el caso simple.

No valida valores, no vuelve frozen al objeto y tiene interacciones avanzadas con inheritance que quedan fuera de PF-M5. Úsalo solo con una razón medida o un contrato de shape claro; no es requisito del challenge.

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(slots=True)
class EventLabel:
    event_id: str
    text: str


label = EventLabel("evt-001", "Llegué")
print(label.text)
```

Output:

```text
Llegué
```

### 16.3 No activar opciones por catálogo

Cada opción (`frozen`, `order`, `slots`, `eq`) modifica semántica. Primero escribe:

- qué puede cambiar;
- qué significa equality;
- si existe orden natural;
- qué evidencia de memoria/shape justifica slots.

Después elige opciones. `@dataclass(frozen=True, order=True, slots=True)` no es automáticamente “más profesional”.

### Decide

¿Event posee un orden natural universal? Compara ordenar una timeline por `valid_time` con ordenar IDs para un resumen determinista.

### Explica

¿Por qué `slots=True` no protege invariantes de `text`?

### Detecta el diseño accidental

Una dataclass Event usa `order=True` y el equipo cambia el orden de fields. Explica por qué puede cambiar sorting sin cambiar requisitos.

---

## 17. Type hints: contratos para humanos y herramientas

### 17.1 El problema antes de la sintaxis

**Ejemplo ejecutable sin annotations:**

```python
def find_event(events, event_id):
    for event in events:
        if event.event_id == event_id:
            return event
    return None
```

La función puede ser correcta, pero un lector debe inferir:

- qué contiene `events`;
- qué tipo tiene `event_id`;
- si ausencia es normal;
- qué retorna cuando encuentra algo.

Los type hints vuelven ese contrato visible para personas, IDEs, linters, type checkers y refactoring. Python runtime conserva su comportamiento dinámico salvo que tu código añada comprobaciones.

### 17.2 Firma tipada

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Event:
    event_id: str


def find_event(events: list[Event], event_id: str) -> Event | None:
    for event in events:
        if event.event_id == event_id:
            return event
    return None


events = [Event("evt-001"), Event("evt-002")]
found = find_event(events, "evt-002")
print(found)
print(find_event(events, "evt-999"))
```

Output:

```text
Event(event_id='evt-002')
None
```

`Event | None` declara dos resultados normales. No dice “siempre existe” ni “la función falla”. Si el contrato fuera “debe existir”, podrías diseñar otra función que produzca un error cuando falta; PF-M6 profundizará esa decisión.

### 17.3 Tipos comunes necesarios

**Ejemplo sintáctico:**

```python
event_id: str = "evt-001"
revision: int = 2
confidence: float = 0.8
active: bool = True
missing: None = None

tags: list[str] = ["home", "arrival"]
events_by_id: dict[str, Event] = {event_id: Event(event_id)}
seen_ids: set[str] = {event_id}
position: tuple[int, str] = (1, event_id)
identifier: str | int = event_id
maybe_event: Event | None = None
```

La union `A | B` significa que ambos tipos forman parte del contrato. `X | None` expresa opcionalidad; no uses `None` para mezclar silenciosamente ausencia, fallo y valor inválido.

### 17.4 Inferencia local

No todo binding necesita annotation:

```python
count = 0
label = "event"
active_ids = ["evt-001"]
```

Un type checker suele inferir tipos evidentes. Anota interfaces públicas, estructuras vacías ambiguas y puntos donde la intención no es obvia. Repetir cada tipo local puede reducir legibilidad sin agregar contrato.

### Predice

¿Qué resultados normales permite `Event | None`? ¿Permite un string según el contrato estático?

### Explica

Separa “puede no existir” de “debe existir y fallar si no”.

### Detecta el ruido

Un bloque anota cada entero temporal y cada string literal. Conserva solo annotations que cambien comprensión o análisis.

### Modifica

Cambia `find_event` para aceptar `tuple[Event, ...]`. Después explica por qué aceptar cualquier iterable sería un contrato distinto que no necesitas introducir aquí.

---

## 18. Runtime no es static type checking

### 18.1 Las annotations no interceptan llamadas

**Ejemplo ejecutable:**

```python
def echo_event_id(event_id: str) -> str:
    return event_id


result = echo_event_id(42)
print(result)
print(type(result).__name__)
```

Output:

```text
42
int
```

Python runtime ejecuta la función. La annotation no convirtió, rechazó ni validó `42`. Un type checker puede señalar la llamada antes de ejecutar.

### 18.2 Dataclass tipada tampoco valida el tipo

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass
class Event:
    event_id: str


event = Event(42)
print(event.event_id)
print(type(event.event_id).__name__)
```

Output:

```text
42
int
```

Si `__post_init__` solo comprueba `if not event_id`, el entero `42` seguiría pasando. Escribe la validación runtime que corresponda a una frontera real; no supongas que la annotation la creó.

### 18.3 Tres preguntas distintas

```text
type correctness   → ¿los tipos estáticos son compatibles?
data validity      → ¿los valores runtime tienen forma/rango aceptable?
domain validity    → ¿representan un estado permitido por las reglas?
```

**Ejemplo conceptual:**

```python
event_id: str = ""
```

El valor tiene el tipo estático `str`, pero puede ser inválido como ID de dominio. Un type checker general no demuestra que el string sea no vacío.

### Predice

¿Qué hace runtime con `Event(42)` en el ejemplo? ¿Qué debería señalar un checker?

### Explica

Da un ejemplo type-correct pero domain-invalid y otro type-incorrect que runtime acepta inicialmente.

### Detecta la falsa garantía

Un README afirma “todos los inputs están validados porque el proyecto tiene 100% de annotations”. Identifica las dos garantías que faltan.

---

## 19. Ejecutar un type checker

### 19.1 Concepto frente a herramienta

Un **type checker estático** analiza código y annotations sin usar cada ruta runtime posible como prueba. PF-M5 usa mypy como herramienta concreta para el laboratorio; el concepto no depende de mypy. Pyright u otro checker puede tener configuración y diagnósticos distintos.

El checker es development tooling, no runtime dependency de EIDOLON. En el `pyproject.toml` enseñado por PF-M4, un ejemplo deliberadamente amplio es:

```toml
[project.optional-dependencies]
dev = ["mypy>=1,<2"]

[tool.mypy]
python_version = "3.14"
strict = true
mypy_path = "src"
```

El rango demuestra declaración, no locking. El proyecto debe probar y bloquear una resolución compatible según su workflow antes de llamarla reproducible.

Instalación y ejecución desde el venv:

```bash
python -m pip install -e ".[dev]"
python -m mypy src tests/type_failures.py
```

En PowerShell y Command Prompt, estos dos comandos son iguales después de activar el venv; solo cambia la activación enseñada en PF-M4.

### 19.2 Errores detectables estáticamente

Árbol conceptual:

```text
project/
├── pyproject.toml
├── src/
│   └── eidolon/
│       ├── __init__.py
│       └── model.py
└── tests/type_failures.py
```

**Diagnóstico estático — `src/eidolon/model.py`:**

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Event:
    event_id: str
    tags: tuple[str, ...] = ()


def event_label(event: Event) -> str:
    return "event:" + event.event_id
```

**Diagnóstico estático — `tests/type_failures.py`:**

```python
from eidolon.model import Event, event_label


event = Event(event_id=42)
tags: list[str] = [1]
label = event_label("evt-001")
```

El checker debe señalar tres incompatibilidades estables:

1. `int` donde `Event.event_id` espera `str`;
2. elementos `int` donde la list declara `str`;
3. `str` donde `event_label` espera `Event`.

El texto exacto y las columnas pueden variar por versión. El objetivo es explicar el contrato roto, no memorizar output de una herramienta.

### 19.3 Lo que el checker no demuestra

**Diagnóstico estático que puede pasar:**

```python
from eidolon.model import Event


event = Event(event_id="", tags=("",))
```

Los tipos son compatibles. Sin invariantes runtime, el ID y el tag vacío siguen siendo domain-invalid. PF-L08 exige incluir fallos de ambos grupos.

### 19.4 Checkers no sustituyen ejecución

```text
type checker PASS ≠ programa correcto
runtime smoke PASS ≠ contratos estáticos correctos
```

Necesitas ambos tipos de evidencia, más invariantes de dominio. PF-M9 diseñará la estrategia completa de tests y debugging.

### Predice

Clasifica los cuatro casos anteriores: ¿checker, runtime validation o domain rule?

### Explica

¿Por qué mypy pertenece a dev y no a `[project.dependencies]` del journal?

### Modifica

Corrige dos de los tres errores estáticos y deja uno deliberado. Ejecuta de nuevo el checker y confirma que la cantidad de diagnósticos relevantes disminuye.

### Comprueba

Ejecuta el archivo type-correct con `event_id=""`. Confirma que un checker exitoso no agrega una excepción runtime inexistente.

---

## 20. `Any` y aliases: precisión con moderación

### 20.1 `Any` como escape hatch

`Any` indica al checker que acepte prácticamente cualquier operación y compatibilidad en ese punto. Es útil en una frontera dinámica no tipada todavía, pero propaga pérdida de información.

**Fragmento deliberadamente impreciso:**

```python
from typing import Any


def extract_event_id(payload: Any) -> str:
    return payload["id"]
```

El checker no puede proteger el acceso, la key ni el tipo de retorno dentro de ese flujo. Runtime todavía puede fallar o devolver un valor no string.

No prohíbas `Any` absolutamente. Aísla su alcance, conviértelo pronto a un modelo validado y documenta por qué existe. No uses `Any` para silenciar un diseño que sí conoces.

### 20.2 `object` no es lo mismo

`object` significa “algún objeto, pero debes estrechar o comprobar antes de operar”. `Any` desactiva gran parte de ese análisis. Elegir `object` conserva la obligación de justificar operaciones.

### 20.3 Type aliases

Un alias puede nombrar una forma compleja repetida:

**Ejemplo sintáctico de Python 3.14:**

```python
type EventsByPerson = dict[str, list[Event]]
```

Esto mejora una firma que repite el dict completo. No crea una nueva class runtime ni distingue dos IDs basados en `str`.

**Ejemplo:**

```python
type EventId = str
type SourceId = str
```

Estos aliases documentan intención, pero un checker normalmente sigue aceptando un SourceId donde espera EventId porque ambos son aliases de `str`. Un identificador realmente distinto requiere otra técnica o value object; no se desarrolla aquí una taxonomía avanzada de IDs.

No crees aliases para `str`, `int` y cada list local si no reducen ambigüedad.

### 20.4 Generics avanzados quedan fuera

PF-M5 no desarrolla `TypeVar` complejo, covariance, contravariance, `ParamSpec` ni variadic generics. El estudiante solo necesita leer containers parametrizados y definir contratos concretos de EIDOLON.

### Predice

¿Qué información pierde el checker después de que un valor se convierte en `Any`?

### Explica

¿Por qué `EventId = str` no valida formato ni crea identidad de dominio?

### Decide

¿`EventsByPerson` aclara una firma pública? ¿Un alias `Count = int` aclara el mismo grado? Justifica densidad, no cuota.

---

## 21. Enum para estados cerrados

### 21.1 Strings mágicos

**Código incorrecto:**

```python
status = "activ"

if status == "active":
    print("visible")
```

El typo es un string válido y puede atravesar el programa hasta que una comparación no coincide. Repetir literales dispersa el conjunto permitido.

### 21.2 Un conjunto de miembros con nombre

**Ejemplo ejecutable:**

```python
from enum import Enum


class EventStatus(Enum):
    ACTIVE = "active"
    CORRECTED = "corrected"
    ARCHIVED = "archived"


status = EventStatus.ACTIVE
print(status is EventStatus.ACTIVE)
print(status.value)
```

Output:

```text
True
active
```

Enum concentra miembros permitidos y ofrece nombres que un checker puede analizar. No valida por sí solo un string externo ni decide transiciones.

### 21.3 Enum dentro de dataclass

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass
from enum import Enum


class EventStatus(Enum):
    ACTIVE = "active"
    CORRECTED = "corrected"
    ARCHIVED = "archived"


@dataclass(frozen=True)
class Event:
    event_id: str
    status: EventStatus = EventStatus.ACTIVE


event = Event("evt-001")
print(event.status.name)
```

Output:

```text
ACTIVE
```

El constructor runtime aún acepta `Event("evt-001", "active")` si no añades validación, aunque un checker lo marque. Type hint y Enum tampoco sustituyen la frontera runtime.

### 21.4 Alternativa a flags imposibles

Un único `EventStatus` evita representar simultáneamente `active=True` y `archived=True`. Si dos propiedades son realmente independientes, dos flags pueden ser correctos. Enum sirve cuando el dominio define alternativas mutuamente excluyentes.

### Predice

¿Qué ocurre runtime al pasar el string `"active"` a un field anotado `EventStatus` sin `__post_init__`?

### Explica

¿Qué estado imposible elimina Enum respecto a dos booleans y qué reglas siguen faltando?

### Decide

¿Tags deben ser Enum? Considera si el conjunto es cerrado y controlado o si crece con datos del usuario.

---

## 22. Protocol y structural typing

### 22.1 El problema: depender de una capacidad, no de una class concreta

Una función necesita consultar Events por ID. Si anota directamente `InMemoryEventStore`, queda acoplada a una implementación aunque solo use `get`.

```text
función de aplicación → capacidad get(event_id)
                     ≠ todos los detalles del dict interno
```

`Protocol` permite describir esa forma estática mediante structural typing.

### 22.2 Protocol pequeño

**Fragmento de contrato estático:**

```python
from typing import Protocol


class EventReader(Protocol):
    def get(self, event_id: str) -> Event | None:
        ...
```

El contrato dice: cualquier objeto con un `get` compatible puede usarse como EventReader para static typing.

### 22.3 Implementación in-memory sin herencia explícita

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Event:
    event_id: str
    text: str


class EventReader(Protocol):
    def get(self, event_id: str) -> Event | None:
        ...


class InMemoryEventStore:
    def __init__(self) -> None:
        self._events: dict[str, Event] = {}

    def add(self, event: Event) -> None:
        self._events[event.event_id] = event

    def get(self, event_id: str) -> Event | None:
        return self._events.get(event_id)


def find_label(reader: EventReader, event_id: str) -> str | None:
    event = reader.get(event_id)
    if event is None:
        return None
    return event.event_id + ": " + event.text


store = InMemoryEventStore()
store.add(Event("evt-001", "Llegué"))
print(find_label(store, "evt-001"))
print(find_label(store, "evt-999"))
```

Output:

```text
evt-001: Llegué
None
```

`InMemoryEventStore` no hereda de `EventReader`. Un checker comprueba compatibilidad estructural por los members usados. Runtime no hace esa comprobación automáticamente en esta forma.

### 22.4 Nominal frente a structural

```text
nominal subtype       → relación declarada por inheritance
structural compatible → posee la forma requerida por Protocol
```

Protocol ayuda a que application dependa de un contrato pequeño y estable, conectando con dependency direction de PF-M4. No exige crear una interfaz para cada función.

### 22.5 Un Protocol demasiado grande

**Código de diseño deficiente:**

```python
class EverythingStore(Protocol):
    def get(self, event_id: str) -> Event | None: ...
    def add(self, event: Event) -> None: ...
    def delete(self, event_id: str) -> None: ...
    def export(self) -> str: ...
    def connect(self) -> None: ...
```

Una función que solo lee no debería depender de escritura, export o conexión. Interfaces pequeñas siguen las necesidades del consumer, no una lista anticipada de capacidades.

### 22.6 ABCs brevemente

Una Abstract Base Class (ABC) expresa una relación nominal y puede incluir implementación compartida o impedir instanciar una class que no implemente abstract methods. Usa inheritance explícita y decorators como `@abstractmethod`.

Protocol favorece compatibilidad estructural y análisis estático sin obligar a heredar. ABC puede ser apropiada cuando el contrato runtime y shared behavior de una familia nominal son parte del diseño. PF-M5 no convierte ninguna en default ni profundiza metaclasses/registro virtual.

### Predice

¿Necesita `InMemoryEventStore(EventReader)` en su declaración para satisfacer el Protocol ante un checker?

### Explica

¿Qué dependencia elimina `EventReader` de la firma de `find_label` y cuál conserva?

### Detecta el acoplamiento

Una función anotada con `InMemoryEventStore` solo llama `get`. Reduce su contrato al member necesario.

### Modifica

Crea `SingleEventReader` con solo un field y un method `get`. Pásalo a `find_label` sin heredar de EventReader.

---

## 23. Modelo EIDOLON progresivo

### 23.1 Fronteras del modelo

El modelo educativo representa únicamente:

- una referencia de procedencia (`SourceRef`);
- un acontecimiento sintético (`Event`);
- una afirmación sobre un Event (`Claim`);
- una corrección que referencia un objeto anterior (`Correction`);
- un store in-memory para consultar Events.

No interpreta psicología, no persiste, no parsea JSON y no define el modelo final de EIDOLON 1.0.

### 23.2 `SourceRef` como value object

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class SourceRef:
    source_id: str

    def __post_init__(self) -> None:
        if type(self.source_id) is not str or not self.source_id:
            raise ValueError("source_id must be a non-empty str")


first = SourceRef("src-001")
second = SourceRef("src-001")

print(first == second)
print(first is second)
```

Output:

```text
True
False
```

Equality por valor encaja; identity de runtime sigue siendo distinta.

### 23.3 `Event` como snapshot pequeño

**Ejemplo ejecutable:**

```python
from dataclasses import dataclass
from datetime import UTC, datetime
from enum import Enum


class EventStatus(Enum):
    ACTIVE = "active"
    CORRECTED = "corrected"
    ARCHIVED = "archived"


@dataclass(frozen=True)
class SourceRef:
    source_id: str

    def __post_init__(self) -> None:
        if type(self.source_id) is not str or not self.source_id:
            raise ValueError("source_id must be a non-empty str")


@dataclass(frozen=True)
class Event:
    event_id: str
    text: str
    valid_time: datetime
    source: SourceRef
    status: EventStatus = EventStatus.ACTIVE
    tags: tuple[str, ...] = ()

    def __post_init__(self) -> None:
        if type(self.event_id) is not str or not self.event_id:
            raise ValueError("event_id must be a non-empty str")
        if type(self.text) is not str or not self.text:
            raise ValueError("text must be a non-empty str")
        if not isinstance(self.valid_time, datetime):
            raise ValueError("valid_time must be a datetime")
        if self.valid_time.tzinfo is None or self.valid_time.utcoffset() is None:
            raise ValueError("valid_time must be timezone-aware")
        if not isinstance(self.source, SourceRef):
            raise ValueError("source must be a SourceRef")
        if not isinstance(self.status, EventStatus):
            raise ValueError("status must be an EventStatus")
        if type(self.tags) is not tuple:
            raise ValueError("tags must be a tuple")
        if any(type(tag) is not str or not tag for tag in self.tags):
            raise ValueError("tags must be non-empty strings")


event = Event(
    event_id="evt-001",
    text="Llegué a casa",
    valid_time=datetime(2026, 8, 26, 18, 30, tzinfo=UTC),
    source=SourceRef("src-001"),
    tags=("home", "arrival"),
)

print(event.event_id)
print(event.status.value)
print(event.source.source_id)
print(event.tags)
```

Output:

```text
evt-001
active
src-001
('home', 'arrival')
```

El ejemplo usa frozen snapshots: cambiar interpretación crea otro objeto en vez de mutar este. Esa es una decisión pedagógica, no una afirmación de que toda entity deba ser frozen.

### 23.4 `Claim` no es `Event`

Event representa un acontecimiento registrado. Claim representa una afirmación sobre algo, aquí un Event. No convertir Claim en subclass de Event evita afirmar una relación `is-a` falsa.

**Continuación — agrega al módulo anterior:**

```python
@dataclass(frozen=True)
class Claim:
    claim_id: str
    about_event_id: str
    statement: str
    source: SourceRef

    def __post_init__(self) -> None:
        values = (self.claim_id, self.about_event_id, self.statement)
        if any(type(value) is not str or not value for value in values):
            raise ValueError("claim strings must be non-empty")
        if not isinstance(self.source, SourceRef):
            raise ValueError("source must be a SourceRef")


claim = Claim(
    claim_id="clm-001",
    about_event_id=event.event_id,
    statement="La llegada ocurrió antes de las 19:00",
    source=SourceRef("src-002"),
)

print(claim.about_event_id)
print(claim.statement)
```

Output:

```text
evt-001
La llegada ocurrió antes de las 19:00
```

La class no decide si el Claim es verdadero ni mide confidence. Ese modelado epistemológico pertenece a fases posteriores.

### 23.5 Correction referencia; no reescribe

**Continuación — agrega al módulo anterior:**

```python
class CorrectionTarget(Enum):
    EVENT = "event"
    CLAIM = "claim"


@dataclass(frozen=True)
class Correction:
    correction_id: str
    target_kind: CorrectionTarget
    target_id: str
    replacement_text: str
    source: SourceRef

    def __post_init__(self) -> None:
        values = (self.correction_id, self.target_id, self.replacement_text)
        if any(type(value) is not str or not value for value in values):
            raise ValueError("correction strings must be non-empty")
        if not isinstance(self.target_kind, CorrectionTarget):
            raise ValueError("target_kind must be a CorrectionTarget")
        if not isinstance(self.source, SourceRef):
            raise ValueError("source must be a SourceRef")


correction = Correction(
    correction_id="cor-001",
    target_kind=CorrectionTarget.EVENT,
    target_id=event.event_id,
    replacement_text="Llegué al edificio",
    source=SourceRef("src-003"),
)

print(event.text)
print(correction.target_id)
print(correction.replacement_text)
```

Output:

```text
Llegué a casa
evt-001
Llegué al edificio
```

El original conserva `Llegué a casa`. La Correction es evidencia nueva que referencia el ID anterior. La existencia real del target requiere un índice o store; no puede probarla `Correction.__post_init__` por sí sola.

### 23.6 Diagrama conceptual

```text
SourceRef ──has-a──▶ Event
SourceRef ──has-a──▶ Claim ──references──▶ Event ID
SourceRef ──has-a──▶ Correction ──references──▶ Event/Claim ID
```

No hay base class `JournalObject`. Los objetos comparten SourceRef mediante composition y conservan contratos distintos.

### Predice

Después de crear Correction, ¿qué valor conserva `event.text` y por qué frozen ayuda a demostrarlo?

### Explica

¿Por qué Claim referencia Event en vez de heredar de Event?

### Detecta la invariante externa

`Correction(target_id="evt-999")` puede construirse con strings no vacíos. ¿Qué objeto necesita consultar para comprobar existencia?

### Decide

¿`CorrectionTarget` justifica Enum? Compáralo con tags, cuyo conjunto puede ser abierto.

---

## 24. Store in-memory y creación de corrections

### 24.1 Un contrato pequeño para lectura

**Ejemplo ejecutable integrado:**

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Event:
    event_id: str
    text: str

    def __post_init__(self) -> None:
        values = (self.event_id, self.text)
        if any(type(value) is not str or not value for value in values):
            raise ValueError("event fields must be non-empty strings")


@dataclass(frozen=True)
class Correction:
    correction_id: str
    target_id: str
    replacement_text: str

    def __post_init__(self) -> None:
        values = (self.correction_id, self.target_id, self.replacement_text)
        if any(type(value) is not str or not value for value in values):
            raise ValueError("correction fields must be non-empty strings")


class EventReader(Protocol):
    def get(self, event_id: str) -> Event | None:
        ...


class InMemoryEventStore:
    def __init__(self) -> None:
        self._events: dict[str, Event] = {}

    def add(self, event: Event) -> None:
        if event.event_id in self._events:
            raise ValueError("duplicate event_id")
        self._events[event.event_id] = event

    def get(self, event_id: str) -> Event | None:
        return self._events.get(event_id)


def create_correction(
    reader: EventReader,
    correction_id: str,
    target_id: str,
    replacement_text: str,
) -> Correction:
    if reader.get(target_id) is None:
        raise ValueError("correction target does not exist")
    return Correction(correction_id, target_id, replacement_text)


store = InMemoryEventStore()
original = Event("evt-001", "Llegué a casa")
store.add(original)

correction = create_correction(
    store,
    correction_id="cor-001",
    target_id="evt-001",
    replacement_text="Llegué al edificio",
)

print(original.text)
print(correction)
```

Output:

```text
Llegué a casa
Correction(correction_id='cor-001', target_id='evt-001', replacement_text='Llegué al edificio')
```

La regla local vive en dataclasses; la regla cross-object consulta EventReader. `InMemoryEventStore` es un índice en memoria, no source of truth persistente ni database.

### 24.2 Duplicate ID es una regla del store

**Failure case — usa las definiciones anteriores:**

```python
store = InMemoryEventStore()
store.add(Event("evt-001", "Llegué"))
store.add(Event("evt-001", "Salí"))
```

Debe terminar con `ValueError`. El segundo Event puede ser localmente válido; el conjunto administrado por el store viola unicidad.

### 24.3 Target ausente

**Failure case — usa las definiciones anteriores:**

```python
store = InMemoryEventStore()
create_correction(store, "cor-001", "evt-999", "Texto nuevo")
```

Debe terminar con `ValueError`. Correction no se construye hasta confirmar existencia conceptual en este store.

### Predice

¿Qué invariantes comprueba Event y cuáles comprueba InMemoryEventStore?

### Explica

¿Por qué `create_correction` acepta EventReader en vez de InMemoryEventStore?

### Modifica

Implementa `SingleEventReader` y demuestra que satisface el contrato de `create_correction` sin herencia.

### Decide

¿La verificación de target debe vivir en `Correction.__post_init__`? Enumera qué dependencia tendría que recibir.

---

## 25. Diseños deficientes y refactor incremental

### 25.1 God class: usar classes no garantiza diseño

**Fragmento de diseño deficiente — no se implementa en PF-M5:**

```python
class EidolonJournal:
    def load_files(self): ...
    def validate_events(self): ...
    def save_json(self): ...
    def print_report(self): ...
    def calculate_claims(self): ...
    def write_logs(self): ...
    def mutate_everything(self): ...
```

Esta class cambia por filesystem, formato, console, reglas de dominio, cálculo y observabilidad. Tiene baja cohesión aunque todo esté “encapsulado”. Files, JSON y logging son además contenido posterior.

Refactor incremental para el alcance actual:

1. conserva `SourceRef`, `Event`, `Claim` y `Correction` como modelos pequeños;
2. conserva cálculos puros como funciones de módulo;
3. usa un store in-memory con responsabilidad explícita;
4. deja I/O y logging fuera hasta que existan sus contratos.

No es necesario crear siete services vacíos.

### 25.2 Class sin estado ni invariante

**Código innecesario:**

```python
class TagNormalizer:
    def normalize(self, tag: str) -> str:
        return tag.strip().lower()
```

**Alternativa ejecutable:**

```python
def normalize_tag(tag: str) -> str:
    return tag.strip().lower()


print(normalize_tag("  Home "))
```

Output:

```text
home
```

La class no posee estado, policy reemplazable ni lifecycle. Una función comunica el contrato con menos ceremony.

### 25.3 Getters/setters automáticos

`get_name()`/`set_name()` no son más encapsulados si solo reflejan el field. Usa atributo público cuando sea parte del contrato, property cuando exista cálculo/compatibilidad simple y operation nombrada cuando se conserve una invariante.

### 25.4 Jerarquía por tipo de dato

**Código deficiente:**

```python
class JournalObject:
    pass


class SourceRef(JournalObject):
    pass


class Event(SourceRef):
    pass


class Claim(Event):
    pass
```

La jerarquía afirma `Event is-a SourceRef` y `Claim is-a Event`, relaciones falsas. Composition expresa el grafo real sin heredar behavior irrelevante.

### 25.5 Estados imposibles a pesar de tener type hints

**Código incorrecto:**

```python
from dataclasses import dataclass


@dataclass
class Event:
    event_id: str
    active: bool
    archived: bool


event = Event(event_id="", active=True, archived=True)
```

Todo es type-correct. El modelo permite ID vacío y flags incompatibles. Type hints no compensan una representación débil.

### Predice

¿Cuántas responsabilidades de la God class cambian por razones independientes?

### Explica

¿Por qué menos classes puede producir un modelo más orientado a objetos en el sentido de invariantes claras?

### Refactoriza

Reemplaza la jerarquía incorrecta con dataclasses que contienen `SourceRef`.

### Decide

Para normalización configurable, compara función, callable guardado y objeto policy. Elige solo lo que el requisito actual necesita.

---

## 26. Caso progresivo integrado: paquete de dominio pequeño

### 26.1 Layout

PF-M4 ya enseñó packages. Un layout pedagógico posible:

```text
eidolon-domain/
├── pyproject.toml
├── src/
│   └── eidolon/
│       ├── __init__.py
│       ├── model.py
│       └── store.py
└── checks/
    ├── smoke.py
    └── type_failures.py
```

`model.py` contiene valores y entities pequeñas. `store.py` contiene Protocol e implementación in-memory. No crea `repositories/`, adapters persistentes ni layers vacías.

### 26.2 Dirección de dependencias

```text
checks → store → model
          │
          └── EventReader Protocol
```

`model` no importa store. Las invariantes locales no dependen de infraestructura.

### 26.3 Smoke check mínimo

**Ejemplo ejecutable — después de instalar el package:**

```python
from datetime import UTC, datetime

from eidolon.model import Event, SourceRef
from eidolon.store import InMemoryEventStore


event = Event(
    event_id="evt-001",
    text="Llegué a casa",
    valid_time=datetime(2026, 8, 26, 18, 30, tzinfo=UTC),
    source=SourceRef("src-001"),
)
store = InMemoryEventStore()
store.add(event)

assert store.get("evt-001") is event
assert store.get("evt-999") is None

print("domain smoke: PASS")
```

Output:

```text
domain smoke: PASS
```

El `is` del primer assert pregunta deliberadamente si el store devuelve la misma instance agregada. No se usa para identidad de dominio general.

### 26.4 Cuándo detenerse

El modelo ya demuestra:

- invariantes locales;
- composition;
- equality de value objects;
- identity de entity por ID;
- type hints;
- Protocol estructural;
- índice in-memory.

Agregar serialization, persistence, schemas externos, services, factories complejas o dependency injection no mejora PF-M5. Esas fronteras esperan requisitos y prerequisites posteriores.

---

## 27. Ejercicios guiados con solución razonada

### Ejercicio guiado 1 — Del dict a un objeto con propósito

**Objetivo.** Convertir representación dispersa sin añadir behavior inventado.

**Input:**

```python
event = {"id": "evt-001", "text": "Llegué", "status": "active"}
```

**Predice antes de resolver:** ¿qué estados inválidos permite y qué regla debe pertenecer al objeto?

**Solución parcial ejecutable:**

```python
from dataclasses import dataclass
from enum import Enum


class EventStatus(Enum):
    ACTIVE = "active"
    ARCHIVED = "archived"


@dataclass(frozen=True)
class Event:
    event_id: str
    text: str
    status: EventStatus = EventStatus.ACTIVE

    def __post_init__(self) -> None:
        values = (self.event_id, self.text)
        if any(type(value) is not str or not value for value in values):
            raise ValueError("event fields must be non-empty strings")
```

La solución nombra el estado, cierra status y rechaza fields vacíos. No añade methods hasta existir behavior relacionado. Es correcta si el dict deja de ser parte del contrato público y no aparecen layers nuevas.

### Ejercicio guiado 2 — Introducir una invariante temporal

**Objetivo.** Rechazar datetime naive sin enseñar manejo de excepciones.

**Predice:** ¿un type checker distingue siempre datetime aware de naive usando solo `datetime`?

**Solución:** en `__post_init__` comprueba `tzinfo is not None` y `utcoffset() is not None`, como en la sección 23. Construye un Event aware y ejecuta un archivo separado con datetime naive que debe terminar en `ValueError`.

El checker ve ambos como `datetime`; la invariante pertenece a runtime/domain.

### Ejercicio guiado 3 — Extraer SourceRef mediante composition

**Objetivo.** Sustituir strings paralelos por un value object.

**Input:** Event y Claim contienen `source_id: str` y `source_namespace: str` por separado.

**Predice:** ¿qué fields deben moverse juntos para preservar significado?

**Solución:** crea una frozen dataclass `SourceRef(namespace: str, source_id: str)` y usa `source: SourceRef` en ambos. Equality por fields describe el valor; Event y Claim no heredan de SourceRef.

**Criterio:** cambiar SourceRef no exige duplicar validación en Event y Claim.

### Ejercicio guiado 4 — Tipar `find_event`

**Objetivo.** Hacer explícita la ausencia normal.

**Solución ejecutable:**

```python
def find_event(events: list[Event], event_id: str) -> Event | None:
    for event in events:
        if event.event_id == event_id:
            return event
    return None
```

`Event | None` obliga al consumer estático a considerar ausencia. No captura errores ni valida IDs. El criterio es que la firma responda qué recibe y qué resultados normales produce.

### Ejercicio guiado 5 — Reemplazar flags por Enum

**Objetivo.** Reducir estados imposibles.

**Input:** `active: bool`, `corrected: bool`, `archived: bool`.

**Solución razonada:** define `EventStatus` con tres members y un solo field. Conserva las transiciones como regla aparte o method explícito; Enum cierra alternativas, pero no decide flechas permitidas.

**Variación:** si “visible” es independiente del lifecycle, puede permanecer como boolean separado.

### Ejercicio guiado 6 — Frozen superficial

**Objetivo.** Demostrar mutabilidad contenida.

**Input:** frozen dataclass con `tags: list[str]`.

**Predice:** ¿qué operación falla: reasignar la list o hacer `append`?

**Solución:** cambia el contrato a `tags: tuple[str, ...] = ()` si los tags del snapshot no deben mutar. La corrección no es “usar frozen más fuerte”; es elegir componentes compatibles con la promesa.

### Ejercicio guiado 7 — Protocol pequeño

**Objetivo.** Desacoplar una consulta del store concreto.

**Input:** `def find_label(store: InMemoryEventStore, ...)` solo llama `get`.

**Solución:** define EventReader con `get` y anota el parámetro. InMemoryEventStore lo satisface estructuralmente. Una implementación con solo un Event también puede satisfacerlo.

**Criterio:** el consumer no requiere `add`, dict interno ni inheritance explícita.

### Ejercicio guiado 8 — Cinco fallos estáticos y dos de dominio

**Objetivo.** Preparar PF-L08.

**Solución razonada:** crea un archivo con cinco incompatibilidades de tipos: ID entero, list de ints, argumento string donde espera Event, retorno incompatible y Optional usado sin estrechar. En otros dos archivos usa valores type-correct pero inválidos: ID vacío y datetime naive.

Ejecuta el checker sobre el primer archivo y runtime sobre los dos últimos. La solución es correcta si no llama “runtime validation” al checker ni “type error” al ID vacío.

### Ejercicio guiado 9 — Correction sin mutar Event

**Objetivo.** Conservar evidencia original.

**Solución:** Event es frozen snapshot; Correction contiene `target_id` y replacement text. `create_correction` consulta EventReader antes de construir. Comprueba `original.text` después.

**Criterio:** no se asigna `event.text`, no se reemplaza el objeto en el store y el target ausente se rechaza antes de crear Correction.

### Ejercicio guiado 10 — Retirar una jerarquía falsa

**Objetivo.** Elegir composition o función.

**Input:** `Claim(Event)` y `Correction(Event)` heredan fields que no representan.

**Solución razonada:** separa tres dataclasses y compón SourceRef; usa referencias por ID. Conserva una función pura para labels si no necesita estado.

**Criterio:** cada class puede describir su invariante sin overrides que deshabilitan behavior heredado.

---

## 28. Ejercicios independientes

1. **Class/instance.** Crea dos instances y demuestra qué attributes comparten y cuáles no.
2. **Aliasing.** Usa dos nombres para una instance mutable; predice y comprueba una mutación.
3. **Función o method.** Implementa una transición en ambos estilos y compara ownership/testabilidad.
4. **Encapsulation.** Reemplaza un setter genérico por una operación de dominio.
5. **Name mangling.** Inspecciona el nombre transformado y explica por qué no es seguridad.
6. **Invariante local.** Rechaza ID vacío en construcción.
7. **Invariante externa.** Identifica qué collaborator necesita una Correction para comprobar target.
8. **Class attribute.** Reproduce contaminación con una list compartida y corrígela.
9. **Method types.** Clasifica cinco operaciones entre instance/class/static/module function.
10. **Property.** Expón un label derivado sin esconder I/O ni mutación.
11. **Inheritance.** Diseña una relación `is-a` válida y prueba sustitución conceptual.
12. **Jerarquía rígida.** Encuentra behavior heredado que una subclass deba desactivar.
13. **Composition.** Reemplaza dos subclasses de policy con un componente intercambiable.
14. **Value object.** Diseña SourceRef con equality y mutabilidad deliberadas.
15. **Entity.** Compara dos snapshots con mismo ID y distinto estado.
16. **Dataclass.** Convierte una class manual y enumera methods generados.
17. **Default factory.** Crea tres instances y demuestra ausencia de aliasing.
18. **Field options.** Justifica un único `repr=False` o `compare=False`; retíralo si no puedes.
19. **Frozen.** Demuestra reasignación rechazada y mutación interior permitida.
20. **Ordering.** Ordena positions con `order=True` y Events con `key=` explícita.
21. **Slots.** Usa slots en un ejemplo aislado y explica qué no valida.
22. **Type hints.** Anota una función con list/dict/set/tuple y union solo donde sea útil.
23. **Optional.** Obliga al consumer a tratar `None` antes de acceder a Event.
24. **Runtime.** Ejecuta una llamada type-incorrect aceptada inicialmente por Python.
25. **Domain validity.** Escribe tres strings type-correct pero inválidos según dominio.
26. **Any.** Aísla un Any en una frontera y conviértelo a un objeto validado lo antes posible.
27. **Alias.** Decide si un alias de container mejora una firma pública.
28. **Enum.** Sustituye strings mágicos y conserva una regla de transición separada.
29. **Protocol.** Crea dos implementaciones estructurales de EventReader.
30. **Review.** Reduce una God class sin crear services vacíos.

---

## 29. Preguntas conceptuales

1. ¿Qué diferencia existe entre class e instance?
2. ¿Por qué `self` permite que el mismo method opere sobre instances distintas?
3. ¿Qué diferencia hay entre identidad de objeto e identidad de dominio?
4. ¿Cuándo una class aporta algo que una función no aporta?
5. ¿Cuándo una función pura comunica mejor que un method?
6. ¿Por qué encapsulation no significa hacer todos los attributes privados?
7. ¿Qué garantiza un underscore y qué no garantiza?
8. ¿Qué problema resuelve name mangling y cuál no?
9. ¿Qué es una invariante y quién debe conservarla después del constructor?
10. ¿Por qué algunas invariantes requieren otros objetos?
11. ¿Cómo aparece aliasing mediante un mutable class attribute?
12. ¿Cuándo un classmethod es mejor que una función?
13. ¿Cuándo staticmethod solo añade ceremony?
14. ¿Qué trabajo debería evitar una property?
15. ¿Qué significa una relación `is-a` sustituible?
16. ¿Qué acoplamiento introduce inheritance?
17. ¿Cuándo inheritance sería más clara que composition?
18. ¿Qué expresa `has-a` en el modelo Event/SourceRef?
19. ¿Qué diferencia un value object de una entity en este alcance?
20. ¿Qué methods genera normalmente una dataclass?
21. ¿Por qué una dataclass no valida datos externos?
22. ¿Por qué existe `default_factory`?
23. ¿Qué significa que `frozen=True` no produzca inmutabilidad transitiva?
24. ¿Cuándo `order=True` expresa dominio y cuándo crea política accidental?
25. ¿Qué diferencia existe entre type correctness y domain validity?
26. ¿Qué significa `Event | None` para el caller?
27. ¿Por qué runtime no aplica automáticamente type hints?
28. ¿Qué costo propaga `Any`?
29. ¿Por qué Protocol puede reducir acoplamiento?
30. ¿Por qué Correction debe agregar evidencia en vez de reescribir Event?

---

## 30. Mini challenge — Dominio tipado in-memory

### 30.1 Objetivo

Construye una pequeña capa de dominio EIDOLON con dataclasses, invariantes, composition, Enum, type hints, un Protocol y una implementación in-memory. Debe preparar **PF-L07 — Modelo Event/Claim** y **PF-L08 — Type-check failure lab**.

### 30.2 Árbol requerido

```text
eidolon-domain/
├── pyproject.toml
├── src/
│   └── eidolon/
│       ├── __init__.py
│       ├── model.py
│       └── store.py
└── checks/
    ├── smoke.py
    ├── type_failures.py
    ├── invalid_empty_id.py
    └── invalid_naive_time.py
```

Reutiliza el `src` layout y venv de PF-M4. El extra dev declara el checker concreto, por ejemplo `mypy>=1,<2`, y el workflow conserva una resolución probada según PF-M4.

### 30.3 Datos sintéticos de entrada

```python
from datetime import UTC, datetime


valid_time = datetime(2026, 8, 26, 18, 30, tzinfo=UTC)
source = SourceRef("src-001")
```

No existen archivos ni payloads externos.

### 30.4 Modelos requeridos

1. `SourceRef`: frozen value object con `source_id` no vacío.
2. `EventStatus`: Enum con `ACTIVE`, `CORRECTED`, `ARCHIVED`.
3. `Event`: frozen snapshot con ID, text, aware `valid_time`, SourceRef, status y tuple de tags.
4. `Claim`: frozen object con ID, `about_event_id`, statement y SourceRef.
5. `CorrectionTarget`: Enum que distingue Event de Claim.
6. `Correction`: frozen object con ID, target kind, target ID, replacement text y SourceRef.

Todos los strings de identidad/texto obligatorios son no vacíos. Los tags son strings no vacíos. Las annotations por sí solas no cuentan como validación.

### 30.5 Contrato de store

Define un `EventReader(Protocol)` con:

```python
def get(self, event_id: str) -> Event | None:
    ...
```

Implementa `InMemoryEventStore` sin heredar explícitamente del Protocol:

- `add(event: Event) -> None` rechaza duplicate IDs;
- `get(event_id: str) -> Event | None` consulta un dict privado;
- el dict es índice in-memory, no persistencia.

### 30.6 Operación de Correction

Implementa una función tipada que reciba EventReader y produzca Correction solo si el target Event existe. Debe:

- devolver un objeto nuevo;
- conservar el Event original sin mutación;
- rechazar target ausente con `ValueError` mínimo;
- no reemplazar el Event dentro del store.

### 30.7 Smoke contract

`checks/smoke.py` debe construir un Event y un Claim, agregarlos donde corresponda, crear Correction y comprobar:

```python
assert store.get("evt-001") is event
assert event.text == "Llegué a casa"
assert correction.target_id == event.event_id
assert correction.replacement_text == "Llegué al edificio"
assert claim.about_event_id == event.event_id
assert event.tags == ("home", "arrival")
```

Output final:

```text
PF-M5 domain challenge: PASS
```

### 30.8 Cinco fallos estáticos

`checks/type_failures.py` contiene deliberadamente:

1. `event_id` entero donde espera string;
2. tuple con un tag entero;
3. string donde una función espera Event;
4. función declarada `-> Event` que retorna `None`;
5. acceso a `.text` sobre `Event | None` sin comprobar ausencia.

Ejecuta:

```bash
python -m mypy src checks/type_failures.py
```

Registra cinco categorías de incompatibilidad. El texto exacto puede variar; no edites configuración para silenciarlas.

### 30.9 Dos fallos que el checker no resuelve

- `invalid_empty_id.py` usa `event_id=""`: type-correct, domain-invalid, debe terminar con `ValueError`.
- `invalid_naive_time.py` usa `datetime(2026, 8, 26, 18, 30)`: type-correct, temporalmente inválido, debe terminar con `ValueError`.

Ejecuta cada archivo por separado. No necesitas escribir manejo de excepciones para demostrar el fallo.

### 30.10 Comprobaciones ejecutables

Desde un venv limpio y después de instalar `.[dev]`:

```bash
python -m compileall -q src checks
python checks/smoke.py
python -m mypy src
python -m mypy src checks/type_failures.py
python checks/invalid_empty_id.py
python checks/invalid_naive_time.py
```

Los primeros tres comandos deben terminar exitosamente. Los últimos tres fallan deliberadamente: el checker encuentra incompatibilidades y los scripts runtime rechazan invariantes.

### 30.11 Preguntas de defensa

1. ¿Qué objects son value objects y cuál representa una entity snapshot?
2. ¿Qué igualdad genera cada dataclass y cómo respondes identidad de dominio?
3. ¿Por qué SourceRef se compone en vez de heredarse?
4. ¿Qué estado imposible evita EventStatus?
5. ¿Qué comprueba `__post_init__` que el checker no demuestra?
6. ¿Por qué InMemoryEventStore satisface EventReader sin inheritance?
7. ¿Por qué Correction no muta Event?
8. ¿Qué cambiaría para persistir y por qué queda fuera de PF-M5?

### 30.12 Criterio de aprobación

El challenge se resuelve si:

- `src` pasa el checker sin errores;
- el archivo deliberadamente incorrecto produce cinco diagnósticos relevantes;
- los dos domain failures se reproducen por separado;
- smoke imprime el output esperado;
- Event original permanece intacto;
- no se usa `Any` para silenciar el checker;
- no se crean base classes, getters/setters o layers sin invariante;
- todo se instala y ejecuta con el environment de PF-M4.

### 30.13 Límites

No agregues files, JSON, custom exception hierarchy, decorators generales, context managers, async, database, backend, framework, Pydantic, SQLAlchemy, FastAPI, Docker, Ollama, LLMs ni embeddings.

---

## 31. Resumen

- OOP agrupa estado, comportamiento relacionado e invariantes cuando esa frontera aporta claridad.
- Class define un contrato de construcción/comportamiento; instance es un objeto concreto.
- Names siguen apuntando a objetos: aliasing, identity y mutabilidad no desaparecen con classes.
- `self` recibe la instance enlazada; `cls` recibe la class en un classmethod.
- Una función pura sigue siendo mejor cuando no existe estado o invariante que poseer.
- Encapsulation protege reglas mediante una interfaz pequeña; underscore es convención, no seguridad.
- Una invariante debe ser verdadera después de construcción y de cada operación pública.
- Class attributes mutables pueden compartir estado accidentalmente entre instances.
- Property sirve para valores derivados o compatibilidad simple, no para esconder I/O o ceremony.
- Inheritance expresa `is-a` y acopla subclasses al contrato de la base.
- Composition expresa `has-a` y permite combinar componentes, pero también puede sobredimensionarse.
- Value object se define principalmente por valor; entity conserva identidad de dominio.
- Dataclass genera boilerplate según fields/opciones; no decide identidad ni valida datos externos.
- `default_factory` crea un mutable nuevo por instance.
- `frozen=True` restringe reasignación de fields; no congela transitivamente objetos contenidos.
- `__post_init__` permite invariantes locales después del constructor generado.
- `order=True` y `slots=True` requieren una razón; no son indicadores automáticos de calidad.
- Type hints documentan contratos para humanos y static tooling; runtime no los aplica por sí solo.
- Type correctness, data validity y domain validity responden preguntas diferentes.
- `Any` desactiva información estática y debe aislarse.
- Enum ayuda con conjuntos cerrados y estados mutuamente excluyentes.
- Protocol describe compatibilidad estructural sin exigir herencia explícita.
- Event, Claim y Correction conservan contratos distintos y comparten SourceRef mediante composition.
- Una Correction agrega evidencia y referencia el original; no lo reescribe silenciosamente.

---

## 32. Checklist de dominio

- [ ] Puedo explicar cuándo una class aporta más que una función.
- [ ] Puedo distinguir class, instance, attribute y method.
- [ ] Puedo predecir aliasing entre nombres que apuntan a la misma instance.
- [ ] Puedo usar `__init__` para establecer estado válido.
- [ ] Puedo diseñar una interfaz pública pequeña sin simular privacidad estricta.
- [ ] Puedo distinguir underscore y name mangling.
- [ ] Puedo formular invariantes locales y cross-object.
- [ ] Puedo impedir que operaciones públicas rompan una invariante.
- [ ] Puedo detectar mutables compartidos como class attributes.
- [ ] Puedo elegir instance method, classmethod, staticmethod o función.
- [ ] Puedo usar property sin recrear getters/setters triviales.
- [ ] Puedo explicar `is-a`, overriding y `super()` al nivel del módulo.
- [ ] Puedo detectar una jerarquía que rompe sustitución.
- [ ] Puedo elegir composition o inheritance bajo tradeoffs explícitos.
- [ ] Puedo distinguir value object y entity sin introducir DDD avanzado.
- [ ] Puedo separar identidad de runtime, equality e identidad de dominio.
- [ ] Puedo crear una dataclass y explicar los methods generados.
- [ ] Puedo usar defaults y `default_factory` sin aliasing.
- [ ] Puedo justificar opciones de `field(...)`.
- [ ] Puedo explicar `frozen=True` y demostrar su límite transitivo.
- [ ] Puedo usar `__post_init__` para invariantes simples.
- [ ] Puedo rechazar ordering/slots cuando no tienen contrato.
- [ ] Puedo escribir hints para scalars, collections, tuples, unions y Optional.
- [ ] Puedo decidir cuándo la inferencia local es suficiente.
- [ ] Puedo ejecutar un type checker declarado como dev tooling.
- [ ] Puedo separar static diagnosis de runtime behavior.
- [ ] Puedo distinguir type correctness, data validity y domain validity.
- [ ] Puedo reconocer el costo de `Any`.
- [ ] Puedo usar aliases solo cuando aclaran una forma.
- [ ] Puedo modelar estados cerrados con Enum cuando corresponde.
- [ ] Puedo definir un Protocol pequeño desde las necesidades del consumer.
- [ ] Puedo satisfacer un Protocol sin inheritance explícita.
- [ ] Puedo explicar cuándo una ABC sería una alternativa nominal.
- [ ] Puedo modelar SourceRef, Event, Claim y Correction como objetos pequeños.
- [ ] Puedo producir Correction sin mutar Event.
- [ ] Puedo mantener un store in-memory sin llamarlo persistencia.
- [ ] Puedo completar el mini challenge solo con PF-M1–PF-M5.

---

## 33. Preparación para labs y EIDOLON 0.0a

### Lab principal: PF-L07 — Modelo Event/Claim

PF-M5 prepara:

- dataclasses separadas para Event y Claim;
- SourceRef mediante composition;
- Enum/IDs e invariantes locales;
- distinction entre equality e identidad de dominio;
- Correction como evidencia adicional.

PF-L07 debe decidir y defender el contrato exacto sin agregar persistencia.

### Lab complementario: PF-L08 — Type-check failure lab

PF-M5 prepara:

- annotations modernas;
- checker como dev tooling;
- cinco fallos estáticos deliberados;
- dos fallos type-correct que requieren runtime/domain evidence;
- distinción checker/runtime/tests.

PF-M9 ampliará la estrategia de tests; PF-L08 aquí solo demuestra la frontera.

### Fundamento para PF-MP1 — Journal CLI

El package de PF-M4 ya puede instalarse; PF-M5 aporta sus modelos tipados. PF-MP1 todavía necesita errores, files/JSON y tests de módulos posteriores antes de completarse.

### Evidencia antes de avanzar

1. mini challenge instalado desde venv limpio;
2. `src` sin diagnósticos del checker;
3. cinco errores estáticos reproducidos y explicados;
4. dos estados type-correct/domain-invalid rechazados en runtime;
5. grafo composition/inheritance defendido;
6. demo de aliasing de instance y mutable class attribute;
7. demo de frozen superficial;
8. Correction creada sin mutar Event;
9. explicación de por qué Protocol no requiere inheritance;
10. nota breve: **“Qué class eliminaría y por qué”**.

Este módulo refuerza **CHECKPOINT PF-C2 — Diseño y lifecycle** en su parte de Event/Claim/Correction con composition y estados válidos.

---

## 34. Recursos de ampliación

La explicación fundamental está contenida aquí. Los recursos canónicos del track permanecen en [`PF.11 Recursos recomendados`](../../02_curriculum/01_programming_foundations.md#pf11-recursos-recomendados), especialmente la documentación oficial de Python 3.14 sobre data model, dataclasses y typing.

Consulta documentación para verificar una API concreta; no la uses como sustituto del razonamiento sobre invariantes, equality, dependency direction y validez.

---

## 35. Límite del módulo

PF-M5 termina en objetos pequeños, invariantes locales/cross-object, dataclasses, type hints, Enum, composition y Protocol con almacenamiento in-memory.

PF-M6 enseñará taxonomía de excepciones, files, JSON y lifecycle de recursos; PF-M7, decorators y context managers como mecanismo general; PF-M8, async/await; PF-M9, testing, debugging y logging avanzados.

No se introducen Pydantic, SQLAlchemy, FastAPI, PostgreSQL, Docker, Ollama, LLMs, embeddings, database, repositories persistentes ni dependency injection frameworks. Tampoco se desarrollan generics avanzados, descriptors, metaclasses, multiple inheritance o DDD completo.

La frontera alcanzada es verificable: datos sueltos se convierten en modelos con significado, los estados inválidos se rechazan donde existe información suficiente, los contratos estáticos se comprueban sin confundirse con runtime y composition mantiene el modelo EIDOLON pequeño.
