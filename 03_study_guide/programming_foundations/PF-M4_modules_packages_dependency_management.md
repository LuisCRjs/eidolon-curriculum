# PF-M4 — Módulos, paquetes y dependency management

**Track:** Programming Foundations  
**Competencias:** D1.1, D1.2; refuerza D1.3 y D12.3  
**Fase:** P0  
**Nivel objetivo:** Profesional para el alcance del módulo  
**Prerequisites:** PF-M1, PF-M2, PF-M3  
**Build:** EIDOLON 0.0a  
**Curriculum source:** [PF-M4](../../02_curriculum/01_programming_foundations.md#pf-m4--módulos-paquetes-y-dependency-management)  
**Status:** approved

Un archivo `eidolon.py` basta mientras el programa tiene pocas reglas. Después aparecen validación, índices, presentación en terminal y comprobaciones. Si todo permanece junto, cada cambio obliga a entender el archivo completo; si se separa sin criterio, aparecen imports circulares y el programa solo funciona desde una carpeta concreta. Además, instalar paquetes globalmente puede ocultar dependencias que faltarán en otra computadora.

PF-M4 enseña a transformar scripts en un proyecto organizado, instalable y reproducible. El orden importa: primero delimitar responsabilidades; después crear módulos y paquetes; por último declarar cómo se construye e instala el proyecto. El tooling sirve a ese contrato, no lo reemplaza.

Ya aprobaste PF-M1–PF-M3. Puedes razonar sobre nombres, namespaces, scopes, funciones, contratos, efectos, colecciones e iteración. Este módulo reutiliza esas capacidades. No introduce OOP, dataclasses, type hints, persistencia, JSON, decorators, context managers, async, bases de datos, backend, Docker ni frameworks de dependency injection.

## Resultados de aprendizaje

Al terminar deberías poder:

- explicar por qué un script monolítico deja de ser una frontera útil;
- separar código por cohesión y responsabilidad sin crear capas vacías;
- distinguir scope de función, namespace de módulo y namespace de paquete;
- crear e importar módulos con `import`, `from ... import ...` y aliases deliberados;
- predecir qué código top-level se ejecuta durante un import y qué reutiliza el cache;
- usar `__name__` y un main guard sin esconder efectos de import;
- diagnosticar conceptualmente `ModuleNotFoundError` antes de instalar o modificar rutas;
- inspeccionar el intérprete, `sys.path` y el origen de un módulo;
- detectar shadowing accidental de módulos como `random.py`;
- distinguir módulo, paquete de import y distribución instalable;
- crear packages regulares con `__init__.py` e imports absolutos o relativos justificados;
- reproducir un circular import y corregir la dirección de dependencias;
- justificar un flat layout o un `src` layout según el riesgo que se quiere detectar;
- crear, activar, verificar y recrear un virtual environment;
- usar `python -m pip` para instalar, consultar, comprobar y desinstalar distribuciones;
- distinguir dependencia directa de transitiva y declarar toda dependencia usada directamente;
- explicar por qué instalar no equivale a declarar ni a bloquear una resolución;
- definir un `pyproject.toml` pequeño con build system, metadata, dependencias y entry point;
- comparar version constraints, lock y environment instalado sin confundir sus funciones;
- evaluar el costo operativo y de supply chain antes de añadir una dependencia;
- instalar EIDOLON desde un environment limpio y ejecutarlo desde otra ubicación.

## Cómo estudiar este módulo

PF-M4 combina Python, estructura de directorios, TOML y terminal. Para cada bloque:

1. identifica desde qué directorio se ejecuta el comando;
2. anota qué intérprete y environment deberían intervenir;
3. predice qué módulo se importará y qué código top-level correrá;
4. ejecuta el ejemplo en un directorio temporal o copia de práctica;
5. verifica el origen real del módulo y la distribución instalada;
6. repite la comprobación desde otra ubicación.

No experimentes dentro de un repositorio con datos importantes. Usa eventos sintéticos y un proyecto de práctica.

### Convenciones del código y la terminal

- **Ejemplo ejecutable:** bloque autónomo dentro del árbol declarado.
- **Continuación:** depende solo del bloque o árbol inmediatamente anterior.
- **Código incorrecto:** antipatrón deliberado cuyo síntoma se explica.
- **Failure case:** comando o import que debe fallar de una forma indicada.
- **Fragmento:** muestra sintaxis o estructura y no se ofrece como proyecto completo.
- **Solución parcial:** corrige el concepto local, pero deja decisiones al estudiante.
- **Terminal POSIX:** comandos para Linux/macOS en un shell compatible.
- **PowerShell:** comandos para Windows PowerShell.
- **Command Prompt:** comandos para `cmd.exe` cuando la activación difiere.

Los outputs con rutas, versiones o mensajes dependientes del sistema se describen mediante propiedades estables. Los ejemplos se validan con Python 3.14. La instalación requiere una distribución de Python que incluya `venv` y permita bootstrap de `pip`; algunas distribuciones del sistema separan esos componentes en paquetes del sistema operativo.

### Sintaxis de apoyo

- `sys` e `importlib.util.find_spec` se usan como instrumentos de diagnóstico; import hooks avanzados quedan fuera;
- `python -c` ejecuta una expresión breve para comprobar el environment;
- `assert` expresa smoke checks pequeños; la estrategia de testing llega en PF-M9;
- TOML se explica solo en las tablas necesarias de `pyproject.toml`;
- los comandos de eliminación no son necesarios para completar el módulo: puedes crear un segundo venv limpio y retirar el anterior después de verificarlo.

---

## 1. De un script a un programa organizado

### 1.1 El problema del archivo único

**Ejemplo ejecutable — script pequeño que ya mezcla responsabilidades:**

```python
events = [
    {"id": "evt-001", "person_id": "person-007", "active": True},
    {"id": "evt-002", "person_id": "person-008", "active": False},
]


def is_valid_event(event):
    return "id" in event and "person_id" in event and bool(event["id"])


def build_active_ids(source_events):
    return [
        event["id"]
        for event in source_events
        if is_valid_event(event) and event["active"]
    ]


active_ids = build_active_ids(events)
for event_id in active_ids:
    print(event_id)
```

Output:

```text
evt-001
```

El código funciona. El problema aparece al crecer:

- la validación cambia por reglas del dominio;
- la construcción de índices cambia por casos de uso;
- la impresión cambia por necesidades de interfaz;
- el arranque cambia por instalación y comandos.

Si todo comparte un archivo, esas razones de cambio se mezclan. Separar no reduce automáticamente la complejidad: debe crear fronteras con propósito.

### 1.2 Cohesión y responsabilidad

**Cohesión** describe cuánto pertenecen juntas las partes de una unidad. Un módulo cohesivo agrupa comportamiento que responde a una responsabilidad reconocible.

Para el journal pequeño:

| Responsabilidad | Pregunta |
|---|---|
| reglas de events | ¿este event sintético cumple el contrato? |
| aplicación | ¿qué resumen o índice se construye? |
| terminal | ¿qué se muestra y cuál es el punto de entrada? |

“Una función por archivo” suele producir baja cohesión y demasiados imports. “Todo en un archivo” mezcla responsabilidades. La frontera adecuada permite nombrar una decisión y cambiarla sin crear un grafo innecesario.

### 1.3 Namespaces y scopes

PF-M2 enseñó LEGB dentro de funciones. Un módulo añade otro namespace: los nombres asignados en su nivel superior pertenecen al namespace global **de ese módulo**, no a un global universal del programa.

```text
eidolon.domain.events       → namespace de un módulo
eidolon.application.summary → otro namespace de módulo
```

Dos módulos pueden definir `build_summary` sin que los nombres choquen, porque se accede a ellos mediante rutas distintas. El import enlaza un nombre en el módulo que importa; no fusiona todos los namespaces.

### 1.4 Separar no es diseñar arquitectura empresarial

En PF-M4, `domain`, `application` y `cli` son nombres útiles para tres responsabilidades observables. No implican microservices, repositories, ports/adapters completos ni dependency injection. Si el proyecto solo tiene dos módulos cohesivos, no necesita cinco directorios vacíos.

### Predice

Si `domain/events.py` y `cli.py` definen un nombre `main`, ¿son el mismo binding? Explica qué namespace contiene cada uno.

### Explica

Separa “cantidad de líneas” de “cantidad de razones para cambiar”. ¿Cuál justifica mejor extraer un módulo?

### Detecta el bug

Un proyecto crea `validators.py`, `helpers.py`, `utils.py`, `managers.py` y `services.py`, cada uno con una función sin relación clara. Detecta por qué tener más archivos no garantiza cohesión.

### Modifica

Marca en el script inicial qué líneas pertenecen a reglas, aplicación y terminal. No muevas código todavía.

### Comprueba en terminal

Guarda una copia del script como `eidolon.py`, ejecútala y registra el directorio actual:

```bash
pwd
python3.14 eidolon.py
```

En PowerShell:

```powershell
Get-Location
py -3.14 .\eidolon.py
```

La comprobación debe producir `evt-001`. La ruta exacta de `pwd` o `Get-Location` depende de tu sistema y no se fija como output.

---

## 2. Módulos e import styles

### 2.1 Qué es un módulo

En el caso habitual de PF-M4, un archivo `.py` importable define un **módulo**. Al importarlo, Python crea un objeto módulo con un namespace y enlaza los nombres definidos en su nivel superior.

Árbol mínimo:

```text
journal_demo/
├── events.py
└── report.py
```

**Ejemplo ejecutable — `events.py`:**

```python
def is_active(event):
    return event["active"]
```

**Ejemplo ejecutable — `report.py`:**

```python
import events


event = {"id": "evt-001", "active": True}
print(events.is_active(event))
```

Desde `journal_demo/`:

```bash
python3.14 report.py
```

Output:

```text
True
```

`import events` enlaza el nombre `events` con el objeto módulo. La función permanece calificada como `events.is_active`, lo que hace visible su procedencia.

### 2.2 `from module import name`

**Ejemplo ejecutable — variante de `report.py`:**

```python
from events import is_active


event = {"id": "evt-001", "active": True}
print(is_active(event))
```

Esta forma enlaza `is_active` directamente en el namespace de `report`. Es legible cuando el nombre es específico y su procedencia permanece clara. En archivos con muchos imports o nombres genéricos, `events.is_active` puede comunicar mejor ownership.

### 2.3 Aliases con `as`

**Ejemplo ejecutable — alias sobre el `events.py` del árbol de la sección 2.1:**

```python
import events as event_rules


event = {"id": "evt-001", "active": True}
print(event_rules.is_active(event))
```

Un alias es apropiado para evitar un choque o adoptar una convención estable. No lo uses para ocultar un nombre claro detrás de una abreviatura privada que nadie reconoce.

También puede aplicarse a un nombre:

```python
from events import is_active as is_active_event
```

### 2.4 Elegir estilo

| Estilo | Ventaja | Riesgo |
|---|---|---|
| `import module` | procedencia visible | nombre más largo |
| `from module import name` | uso conciso | colisiones y procedencia menos visible |
| `import module as alias` | resuelve nombre largo/choque | alias críptico |
| `from module import name as alias` | contrato local explícito | demasiados renombres |

Evita `from module import *`: vuelve implícitos los bindings y dificulta descubrir su origen.

### 2.5 Imports son bindings

Conecta con PF-M1 y PF-M2:

```python
import events
```

no copia el archivo dentro de `report`. Ejecuta o recupera el módulo y enlaza `events` en el namespace actual. Esta operación sigue las reglas de nombres y objetos ya aprendidas.

### Predice

Si `report.py` hace `import events as rules`, ¿existe allí el nombre `events`, el nombre `rules` o ambos?

### Explica

¿Cuándo `events.is_active` comunica mejor que `is_active`?

### Detecta el bug

```python
from events import *

is_active = "yes"
```

Explica por qué el segundo binding vuelve ambiguo qué significa `is_active` y por qué el wildcard dificulta review.

### Modifica

Cambia `report.py` entre los tres estilos enseñados. Conserva el mismo output y justifica cuál deja más clara la procedencia.

### Comprueba en terminal

Desde `journal_demo/`:

```bash
python3.14 -c "import events; print(events.__name__)"
```

Output:

```text
events
```

---

## 3. Ejecución durante import, cache y `__name__`

### 3.1 Importar ejecuta el nivel superior

**Código incorrecto — `event_rules.py` con side effect de import:**

```python
print("registrando reglas")


def is_valid_event(event):
    return bool(event["id"])
```

**Ejemplo ejecutable — `use_rules.py`:**

```python
import event_rules


print(event_rules.is_valid_event({"id": "evt-001"}))
```

Output:

```text
registrando reglas
True
```

La definición de la función necesita ejecutarse para crear el objeto función. El `print` también se ejecuta porque está en top-level. Imports que escriben en console, consultan red, abren archivos o mutan estado externo hacen que obtener nombres produzca efectos sorpresivos. PF-M4 evita esos efectos; PF-M6 estudiará lifecycle de archivos y errores.

### 3.2 Import cache al nivel necesario

Dentro de un proceso, un módulo importado normalmente queda registrado en `sys.modules`. Otro import del mismo nombre reutiliza ese objeto en vez de ejecutar de nuevo todo su top-level.

**Continuación — usa el `event_rules.py` de la sección 3.1:**

```python
import sys
import event_rules
import event_rules as same_rules


print(event_rules is same_rules)
print("event_rules" in sys.modules)
```

Output:

```text
registrando reglas
True
True
```

El side effect aparece una sola vez en este proceso: el segundo import recupera el mismo objeto módulo desde el cache.

Esto no vuelve aceptables los side effects. “Suele ocurrir una vez por proceso” sigue siendo un contrato frágil para inicialización, tests, tools y reloads. Tampoco significa que editar el archivo cambie un proceso que ya lo importó; normalmente debes reiniciar el proceso de práctica.

No necesitas import locks, loaders, finders personalizados ni internals de bytecode en PF-M4.

### 3.3 `__name__`

Cada módulo recibe un nombre especial `__name__`:

- al importarse, suele ser su nombre importable, por ejemplo `event_rules`;
- cuando se ejecuta como programa principal, vale `"__main__"`.

**Ejemplo ejecutable — `show_name.py`:**

```python
print(__name__)
```

```bash
python3.14 show_name.py
python3.14 -c "import show_name"
```

Outputs, en orden:

```text
__main__
show_name
```

### 3.4 Main guard

El main guard separa definiciones importables del arranque directo.

**Ejemplo ejecutable — `journal_cli.py`:**

```python
def main():
    print("EIDOLON journal")


if __name__ == "__main__":
    main()
```

Al ejecutar el archivo imprime el título. Al importarlo, define `main` sin llamarla.

El guard no corrige imports rotos, no vuelve instalable un proyecto y no sustituye un entry point. Solo decide si se llama al arranque por ejecución directa.

### 3.5 Qué pertenece en top-level

Top-level apropiado:

- definiciones de funciones;
- constantes inmutables pequeñas;
- imports sin efectos de dominio;
- metadata del módulo.

Top-level que exige cautela:

- imprimir;
- leer environment o clock para fijar reglas;
- construir grandes índices mutables;
- abrir recursos;
- ejecutar la aplicación.

PF-M2 ya explicó que los efectos deben ser visibles. PF-M4 añade: no deben dispararse solo por importar una definición.

### Predice

Predice cuántas veces imprime `registrando reglas` un proceso que ejecuta dos sentencias `import event_rules`. Después explica por qué esa predicción no justifica el side effect.

### Explica

¿Qué diferencia existe entre “definir `main`” y “llamar `main`” durante un import?

### Detecta el bug

```python
events_by_id = {}
events_by_id["evt-001"] = {"id": "evt-001"}
print("índice listo")
```

Si esto ocurre al importar `index.py`, cualquier consumidor dispara mutación y output. Mueve la construcción a una función explícita.

### Modifica

Convierte `event_rules.py` para que no imprima al importarse. Conserva un `main()` opcional para una demostración manual.

### Comprueba en terminal

```bash
python3.14 -c "import journal_cli; print('import terminado')"
python3.14 journal_cli.py
```

El primer comando solo imprime `import terminado`; el segundo imprime `EIDOLON journal`.

---

## 4. Cómo Python encuentra módulos

### 4.1 Modelo práctico del import system

Al resolver `import eidolon`, Python consulta su sistema de importación. Algunas partes de la standard library están integradas o congeladas; otras y los módulos del proyecto se localizan mediante rutas de búsqueda. `sys.path` permite inspeccionar las ubicaciones relevantes para el proceso actual.

En la práctica pueden intervenir:

1. el contexto de ejecución: directorio del script o current working directory según la forma de invocación;
2. ubicaciones de la instalación de Python y su standard library;
3. `site-packages` del environment activo, donde viven third-party distributions instaladas;
4. el proyecto propio cuando está instalado o expuesto deliberadamente.

El orden y las rutas exactas dependen del intérprete, plataforma, environment e invocación. No memorices una lista de directorios de otra computadora.

### 4.2 Inspeccionar `sys.path`

**Ejemplo ejecutable:**

```python
import sys


for position, location in enumerate(sys.path):
    print(position, repr(location))
```

No fijes el output: las rutas son ambientales. Busca qué entrada representa el proyecto, la standard library y `site-packages`.

Modificar `sys.path` dentro del programa para “hacer que funcione” suele ocultar un proyecto no instalado, una invocación incorrecta o un layout defectuoso. Diagnostica antes de mutar la ruta.

### 4.3 `ModuleNotFoundError`

**Failure case — módulo inexistente:**

```bash
python3.14 -c "import eidolon_missing"
```

Debe terminar con `ModuleNotFoundError`. El mensaje exacto puede variar; conserva:

- el nombre importado;
- `sys.executable`;
- el directorio actual;
- `sys.path`;
- el comando de instalación usado.

Secuencia de diagnóstico:

1. verifica spelling y nombre de import;
2. verifica qué Python ejecuta el comando;
3. verifica que `pip` pertenezca al mismo Python;
4. pregunta si el proyecto está instalado en ese environment;
5. inspecciona si el package forma parte de la distribución;
6. revisa desde dónde y cómo se ejecutó el programa.

No respondas automáticamente con `pip install <nombre>`: el nombre podría ser incorrecto, malicioso o distinto del nombre de distribución esperado.

### 4.4 Encontrar una especificación sin importar

`importlib.util.find_spec` permite preguntar si el import system encuentra un nombre top-level sin ejecutar el módulo objetivo. Con nombres punteados como `package.submodule`, la búsqueda puede importar el parent package para localizar el hijo; por eso la sonda tampoco autoriza `__init__.py` con side effects.

**Ejemplo ejecutable:**

```python
import importlib.util


spec = importlib.util.find_spec("collections")
print(spec is not None)
# True
```

Para un módulo ausente devuelve `None` en el caso habitual:

```python
import importlib.util


spec = importlib.util.find_spec("eidolon_missing")
print(spec is None)
# True
```

No profundices en `ModuleSpec`, loaders ni hooks; aquí es una sonda de diagnóstico.

### 4.5 Shadowing accidental: `random.py`

El contexto del proyecto suele buscarse antes que muchas ubicaciones instaladas. Un archivo local con el mismo nombre que un módulo esperado puede ocultarlo.

Árbol incorrecto:

```text
shadow_demo/
├── random.py
└── use_random.py
```

**Código incorrecto — `random.py`:**

```python
DEFAULT_SEED = 7
```

**Failure case — `use_random.py`:**

```python
import random


print(random.randint(1, 10))
```

El import puede resolver el archivo local y después fallar con `AttributeError` porque ese módulo no define `randint`. Diagnóstico:

```bash
python3.14 -c "import random; print(random.__file__)"
```

La ruta debe señalar qué archivo ganó. Corrige renombrando el módulo local con un nombre de dominio, actualizando imports y retirando cache generado si todavía interfiere. No borres archivos sin confirmar primero su ruta.

El mismo riesgo existe con nombres locales como `typing.py` o `json.py`: pueden ocultar módulos de la standard library que importa tu programa o una herramienta. El síntoma puede aparecer lejos del archivo que produjo el shadowing; por eso se verifica el origen del módulo, no solo el último nombre del traceback.

### Predice

¿Qué cambia en `sys.path` al usar otro interpreter o environment aunque el código fuente sea idéntico?

### Explica

¿Por qué `ModuleNotFoundError` no demuestra por sí solo que “falta instalar un paquete de internet”?

### Detecta el bug

Un desarrollador agrega `sys.path.append("../..")` hasta que el import funciona. Explica qué información del layout y la invocación quedó escondida.

### Modifica

Renombra el ejemplo `random.py` a `journal_defaults.py` y corrige su import sin tocar `sys.path`.

### Comprueba en terminal

```bash
python3.14 -c "import sys; print(sys.executable)"
python3.14 -c "import random; print(random.__file__)"
python3.14 -c "import importlib.util; print(importlib.util.find_spec('eidolon_missing'))"
```

La última línea debe imprimir `None`. Las rutas anteriores dependen del sistema.

---

## 5. Packages e imports internos

### 5.1 Qué es un package

Un **package** agrupa módulos bajo un nombre importable y puede contener subpackages.

```text
journal_package/
└── eidolon/
    ├── __init__.py
    ├── domain/
    │   ├── __init__.py
    │   └── events.py
    └── report.py
```

En este módulo usamos packages regulares: cada directorio importable contiene `__init__.py`. Python también admite namespace packages sin ese archivo, pero no los necesitas para EIDOLON P0.

### 5.2 `__init__.py`

`__init__.py` marca el package regular y su top-level se ejecuta cuando se inicializa el package. Manténlo pequeño y sin efectos.

**Ejemplo ejecutable — `eidolon/__init__.py`:**

```python
"""EIDOLON journal package para PF-M4."""
```

No necesitas reexportar cada función. Un `__init__.py` que importa todo puede aumentar acoplamiento y provocar ciclos difíciles de ver.

### 5.3 Absolute imports

**Ejemplo ejecutable — `eidolon/domain/events.py`:**

```python
def is_valid_event(event):
    return "id" in event and bool(event["id"])
```

**Ejemplo ejecutable — `eidolon/report.py`:**

```python
from eidolon.domain.events import is_valid_event


event = {"id": "evt-001"}
print(is_valid_event(event))
```

El absolute import muestra la ruta completa desde el package superior. Suele ser preferible entre responsabilidades distintas porque deja visible la dirección de dependencia.

### 5.4 Relative imports

Dentro de un package, un punto representa el package actual y dos puntos el parent package.

**Fragmento — alternativa dentro de `eidolon/domain/labels.py`:**

```python
from .events import is_valid_event
```

**Fragmento — desde `eidolon/application/summary.py`:**

```python
from ..domain.events import is_valid_event
```

Un relative import corto entre módulos estrechamente relacionados puede ser razonable. Al cruzar responsabilidades, el absolute import suele comunicar mejor el grafo. No mezcles estilos arbitrariamente.

Los relative imports necesitan package context. Ejecutar directamente:

```bash
python3.14 eidolon/application/summary.py
```

puede fallar con “attempted relative import with no known parent package”. Ejecuta desde el package instalado:

```bash
python3.14 -m eidolon.application.summary
```

si el módulo ofrece un main guard, o usa el entry point del proyecto.

### 5.5 Módulo, package y distribución

| Concepto | Qué representa | Ejemplo |
|---|---|---|
| módulo | unidad con namespace importable | `eidolon.cli` |
| package | módulo que agrupa submódulos | `eidolon` |
| distribución instalable | proyecto versionado que instala packages y metadata | `eidolon-journal-pf4` |

El nombre de distribución que consulta `pip` puede diferir del nombre importable. Instalar `eidolon-journal-pf4` puede habilitar `import eidolon`. No adivines uno desde el otro; consulta `pyproject.toml` y metadata instalada.

### 5.6 Organización por responsabilidad

Un package no obliga a crear una carpeta por verbo. Agrupa módulos que cambian por razones relacionadas. Para PF-M4:

```text
eidolon.domain       → reglas puras sobre events sintéticos
eidolon.application  → coordinación y derivados en memoria
eidolon.cli          → efecto de console y entry point
```

`adapters` puede aparecer cuando PF-M6 introduzca fronteras reales de archivos. Crear ahora un directorio vacío no mejora el diseño.

### Predice

¿Qué `__name__` esperas dentro de `eidolon.domain.events` al importarlo?

### Explica

¿Por qué `eidolon` puede ser package mientras `eidolon-journal-pf4` es nombre de distribución?

### Detecta el bug

```bash
python3.14 src/eidolon/application/summary.py
```

Si el archivo usa relative imports, explica por qué ejecutarlo como archivo elimina el package context esperado.

### Modifica

Cambia un relative import entre `application` y `domain` por un absolute import. Dibuja si la dirección del grafo cambió o solo su escritura.

### Comprueba en terminal

Después de instalar el proyecto de práctica:

```bash
python -c "import eidolon; print(eidolon.__name__)"
python -c "import eidolon.domain.events as events; print(events.__name__)"
```

Outputs:

```text
eidolon
eidolon.domain.events
```

---

## 6. Imports circulares y dirección de dependencias

### 6.1 Cómo aparece un ciclo

Un dependency graph usa módulos como nodos e imports como flechas. Este grafo contiene un ciclo:

```text
eidolon.domain.events ──→ eidolon.application.summary
          ↑                         │
          └─────────────────────────┘
```

Árbol:

```text
cycle_demo/
└── eidolon/
    ├── __init__.py
    ├── domain/
    │   ├── __init__.py
    │   └── events.py
    └── application/
        ├── __init__.py
        └── summary.py
```

**Código incorrecto — `domain/events.py`:**

```python
from eidolon.application.summary import event_label


def is_valid_event(event):
    return bool(event["id"])


def describe_event(event):
    return event_label(event)
```

**Código incorrecto — `application/summary.py`:**

```python
from eidolon.domain.events import is_valid_event


def event_label(event):
    if is_valid_event(event):
        return "event:" + event["id"]
    return "invalid"
```

Al importar `summary`, Python empieza a ejecutar un módulo que importa `events`; `events` intenta obtener `event_label` desde `summary`, que todavía no terminó de inicializarse. El resultado habitual es `ImportError` desde un módulo parcialmente inicializado.

### 6.2 El ciclo revela responsabilidades invertidas

`domain` no debería depender de una presentación de `application`. La regla `is_valid_event` es más estable y pequeña; `application` puede depender de ella.

Grafo corregido:

```text
eidolon.cli ──→ eidolon.application.summary ──→ eidolon.domain.events
```

**Ejemplo ejecutable — `domain/events.py` corregido:**

```python
def is_valid_event(event):
    return bool(event["id"])
```

**Ejemplo ejecutable — `application/summary.py` corregido:**

```python
from eidolon.domain.events import is_valid_event


def event_label(event):
    if is_valid_event(event):
        return "event:" + event["id"]
    return "invalid"
```

Si `describe_event` solo reenviaba a `event_label`, no pertenece al dominio. Elimínala o ubica esa coordinación en application.

### 6.3 Extraer una responsabilidad compartida

Otro ciclo puede indicar que dos módulos necesitan un tercer concepto estable. Extraerlo es válido si posee responsabilidad propia, no como depósito `utils.py`.

```text
module_a ──→ shared_rule ←── module_b
```

La extracción correcta elimina las flechas inversas. No basta mover funciones hasta que el error desaparezca; el nombre y contrato del tercer módulo deben ser coherentes.

### 6.4 Import local: alivio táctico, no refactor automático

Mover un import dentro de una función retrasa su ejecución:

**Fragmento:**

```python
def describe_event(event):
    from eidolon.application.summary import event_label

    return event_label(event)
```

Puede evitar el fallo durante inicialización, pero el grafo conceptual sigue siendo bidireccional y el error puede aparecer al llamar. Los imports locales son razonables para dependencias opcionales o costos medidos, no como cura universal de un diseño circular.

### 6.5 Diagnóstico reproducible

Conserva:

- el traceback completo;
- los dos módulos y sus imports top-level;
- el primer comando que reproduce el ciclo;
- el grafo antes y después;
- la responsabilidad que se movió.

No “corrijas” renombrando aleatoriamente o añadiendo `try/except ImportError`; eso oculta la dirección defectuosa.

### Predice

En el ciclo mostrado, ¿qué nombre intenta obtener `events.py` antes de que `summary.py` termine de definirlo?

### Explica

¿Por qué un circular import suele ser un problema de ownership y no solo de orden de líneas?

### Detecta el bug

Un equipo mueve los dos imports problemáticos dentro de funciones y declara resuelto el ciclo. Dibuja el grafo conceptual y demuestra que las flechas inversas siguen allí.

### Modifica

Refactoriza el ejemplo para que solo `application` importe `domain`. No agregues un módulo `utils`.

### Comprueba en terminal

Con el package expuesto en el environment, el caso incorrecto debe fallar:

```bash
python -c "from eidolon.application.summary import event_label"
```

Después de la refactorización, comprueba:

```bash
python -c "from eidolon.application.summary import event_label; print(event_label({'id': 'evt-001'}))"
```

Output:

```text
event:evt-001
```

---

## 7. Layout del proyecto y `src` layout

### 7.1 Qué problema resuelve un layout

Un layout decide dónde viven metadata, código importable, documentación y comprobaciones. Debe ayudar a que una instalación limpia encuentre exactamente lo que se publicaría o instalaría.

Partimos de:

```text
eidolon.py
```

Después de separar responsabilidades, un árbol posible es:

```text
eidolon-journal/
├── pyproject.toml
├── README.md
├── src/
│   └── eidolon/
│       ├── __init__.py
│       ├── domain/
│       │   ├── __init__.py
│       │   └── events.py
│       ├── application/
│       │   ├── __init__.py
│       │   └── summary.py
│       └── cli.py
└── tests/
    └── smoke.py
```

Razón de cada parte:

| Parte | Responsabilidad actual |
|---|---|
| `pyproject.toml` | contrato de build, metadata, instalación y tooling |
| `README.md` | instrucciones mínimas para otra persona |
| `src/` | separa source importable de la raíz del repositorio |
| `eidolon/` | package importable |
| `domain/` | reglas puras ya existentes |
| `application/` | coordinación e índices derivados |
| `cli.py` | console y entry point |
| `tests/smoke.py` | comprobaciones mínimas; PF-M9 diseñará la suite real |

### 7.2 Flat layout frente a `src` layout

Flat layout:

```text
project/
├── pyproject.toml
└── eidolon/
```

Al ejecutar Python desde `project/`, la raíz suele estar disponible para imports. `import eidolon` puede funcionar aunque el proyecto no esté instalado o aunque la configuración de packaging omita archivos.

`src` layout:

```text
project/
├── pyproject.toml
└── src/
    └── eidolon/
```

La raíz no contiene directamente `eidolon`. Para importarlo de forma normal debes instalar la distribución o modificar rutas deliberadamente. Esa fricción detecta errores de packaging antes de otra máquina.

### 7.3 No es la única estructura válida

Un script de una sola finalidad puede permanecer como archivo. Un package interno pequeño puede usar flat layout. PF-M4 elige `src` para PF-L06 porque el objetivo explícito es comprobar instalación limpia y evitar imports accidentales desde la raíz.

No crees `domain/application/cli` por ceremonia. El hilo del journal ya posee esas tres responsabilidades; por eso aparecen.

### 7.4 Ejecutar desde la raíz no equivale a instalar

**Failure case — antes de instalar un `src` layout:**

```bash
python3.14 -c "import eidolon"
```

Desde la raíz del proyecto debe fallar en un environment limpio si `src` no se añadió por otro medio. El fallo es útil: demuestra que no estás importando accidentalmente el source tree.

No lo “arregles” con `export PYTHONPATH=src` como instrucción principal. Instala el proyecto en el environment.

### Predice

¿Por qué `import eidolon` puede funcionar desde la raíz de un flat layout aunque `pip` no conozca la distribución?

### Explica

¿Qué clase de error detecta `src` layout antes de empaquetar o copiar el proyecto?

### Detecta el bug

El README dice “ejecuta siempre desde la raíz porque de otro modo fallan los imports”. Explica por qué esa precondición revela que el proyecto no está correctamente instalado o importado.

### Modifica

Convierte un flat layout de práctica a `src/` sin cambiar el nombre importable `eidolon`. Actualiza solo la configuración necesaria.

### Comprueba en terminal

Antes y después de la instalación, desde la raíz:

```bash
python -c "import importlib.util; print(importlib.util.find_spec('eidolon'))"
python -m pip install -e .
python -c "import importlib.util; print(importlib.util.find_spec('eidolon') is not None)"
```

En un venv limpio, la primera comprobación debe imprimir `None`; la última, `True`.

---

## 8. Environments y virtual environments

### 8.1 El problema de las dependencias globales

Si varios proyectos comparten el mismo conjunto global de paquetes:

- una actualización para uno puede romper otro;
- una dependencia “fantasma” puede estar instalada sin declararse;
- `pip list` no revela qué proyecto pidió cada paquete;
- otra máquina no reproduce el estado.

Un **environment** reúne, al nivel práctico, un intérprete y las distribuciones disponibles para él, además del contexto desde el que se ejecuta. Un virtual environment (venv) crea un entorno aislado de paquetes respecto al Python base.

El venv no aísla sistema operativo, red, archivos, secrets o variables de entorno. Tampoco vuelve reproducible por sí solo qué versiones instalarás mañana.

### 8.2 Crear un venv

Desde la raíz del proyecto.

**Terminal POSIX — Linux/macOS:**

```bash
python3.14 -m venv .venv
. .venv/bin/activate
```

**PowerShell:**

```powershell
py -3.14 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

**Command Prompt:**

```bat
py -3.14 -m venv .venv
.venv\Scripts\activate.bat
```

La activación antepone ejecutables del venv a `PATH` para esa terminal. No cambia el sistema de forma mágica ni es obligatoria: también puedes llamar `.venv/bin/python` o `.venv\Scripts\python.exe` directamente.

Una política de ejecución de PowerShell puede impedir scripts de activación. No desactives controles globales sin entenderlos; usa el Python del venv directamente o aplica la política aprobada de tu entorno.

### 8.3 Verificar Python y pip

Después de activar:

```bash
python -c "import sys; print(sys.executable)"
python -c "import sys; print(sys.prefix != sys.base_prefix)"
python -m pip --version
```

La segunda comprobación debe imprimir:

```text
True
```

`sys.executable` y `pip --version` deben señalar `.venv`. La ruta exacta varía.

En POSIX puedes añadir:

```bash
command -v python
```

En Windows:

```powershell
where.exe python
```

Prefiere `python -m pip` a `pip`: fuerza a ejecutar el módulo `pip` del Python que acabas de verificar.

### 8.4 Cuando el venv no contiene pip

Algunas instalaciones de Python separan `venv` o `ensurepip`. Primero corrige la instalación de Python según tu sistema. Cuando `ensurepip` está disponible:

```bash
python -m ensurepip --upgrade
```

No descargues y ejecutes scripts arbitrarios solo porque un mensaje sugiera “instalar pip”. Verifica origen y documentación del runtime.

### 8.5 El venv no es source code

`.venv/` contiene ejecutables, rutas y paquetes específicos del environment. No debe versionarse ni copiarse como mecanismo de instalación. Decláralo en `.gitignore`, documenta cómo recrearlo y conserva intención/versiones en archivos de proyecto o lock apropiado.

Un venv tampoco debe considerarse relocatable. Si cambia la ruta o el Python base, recréalo.

### 8.6 Recrear sin destruir primero

Puedes demostrar reproducibilidad creando otro environment:

La secuencia siguiente es el objetivo operativo. Ejecútala después de crear el `pyproject.toml` de la sección 10 y el proyecto integrado de la sección 14; la sección 9 explica cada operación de `pip`.

**POSIX:**

```bash
python3.14 -m venv .venv-clean
.venv-clean/bin/python -m pip install .
.venv-clean/bin/eidolon
```

**PowerShell:**

```powershell
py -3.14 -m venv .venv-clean
.\.venv-clean\Scripts\python.exe -m pip install .
.\.venv-clean\Scripts\eidolon.exe
```

Cuando `.venv-clean` pasa las comprobaciones, puedes retirar el environment anterior mediante una operación deliberada y local. No uses un target ambiguo o una variable no verificada en comandos recursivos.

### Predice

Después de activar `.venv`, ¿qué propiedad cambia: el archivo fuente, la variable `PATH` de la terminal o ambos?

### Explica

¿Por qué un venv aislado no es todavía una especificación reproducible de versiones?

### Detecta el bug

```bash
pip install example-package
python3.14 app.py
```

Si `pip` y `python3.14` pertenecen a environments distintos, la instalación no afecta al proceso. Corrige verificando `sys.executable` y usando `python -m pip`.

### Modifica

Reescribe todos tus comandos de `pip` para que estén asociados al Python verificado.

### Comprueba en terminal

Crea `.venv-clean` sin activar y demuestra:

```bash
.venv-clean/bin/python -c "import sys; print(sys.prefix != sys.base_prefix)"
.venv-clean/bin/python -m pip --version
```

En PowerShell usa las rutas bajo `.venv-clean\Scripts\`.

---

## 9. `pip`: instalar no es declarar

### 9.1 Operaciones esenciales

Dentro del venv verificado:

```bash
python -m pip install .
python -m pip install -e .
python -m pip list
python -m pip show eidolon-journal-pf4
python -m pip check
python -m pip uninstall eidolon-journal-pf4
```

- `install .` construye e instala desde el proyecto local;
- `install -e .` crea una instalación editable útil durante desarrollo;
- `list` inventaría distribuciones instaladas;
- `show` consulta metadata de una distribución;
- `check` detecta requirements instalados incompatibles o ausentes;
- `uninstall` retira la distribución del environment actual.

Editable no significa “sin instalación” ni simula una release. El código fuente queda enlazado al environment, por lo que cambios se reflejan sin reinstalar; metadata o entry points pueden requerir reinstalación.

### 9.2 Consultar versiones

```bash
python -m pip show pip
python -m pip list
```

No fijes el output del entorno del autor. Registra al menos nombre, versión y ubicación para diagnosticar.

### 9.3 `pip install` cambia estado, no intención

Ejecutar:

```bash
python -m pip install some-package
```

modifica el environment actual. No añade automáticamente esa dependencia a `[project.dependencies]`, no explica por qué existe y no garantiza que otro environment resuelva la misma versión.

Reproducibilidad requiere declarar intención y aplicar una política de resolución/locking adecuada; después se comprueba en un environment vacío.

### 9.4 Direct dependency y transitive dependency

```text
EIDOLON ──declara──→ package-a ──declara──→ package-b
```

- `package-a` es dependencia directa de EIDOLON;
- `package-b` llega transitivamente mediante `package-a`.

Si el código de EIDOLON hace `import package_b`, entonces `package-b` es una dependencia directa en términos de contrato y debe declararse directamente. Depender de que `package-a` continúe instalándola es frágil.

### 9.5 Dependencia global fantasma

Un proyecto puede funcionar en la computadora del autor porque una distribución quedó instalada globalmente. En un venv limpio aparece `ModuleNotFoundError`. La corrección es decidir:

1. si la standard library resuelve el problema;
2. si la dependencia externa es necesaria;
3. si se usa directamente y debe declararse;
4. qué constraint y evidencia la acompañan.

No copies todo lo que aparece en `pip list` a `pyproject.toml`; el environment puede contener tooling y transitive dependencies ajenas al runtime.

### 9.6 `pip freeze`

`python -m pip freeze` describe gran parte del estado instalado en un formato de requirements. Puede servir como evidencia o input de algunos workflows, pero no distingue intención directa de transitiva y no sustituye por sí solo metadata del proyecto ni un lock multiplataforma deliberado.

### Predice

Si EIDOLON declara `package-a` pero importa directamente `package-b`, ¿qué ocurre si una versión futura de `package-a` deja de depender de `package-b`?

### Explica

¿Por qué `pip list` y `[project.dependencies]` responden preguntas distintas?

### Detecta el bug

“Funciona en mi máquina; ejecuté `pip install` hace meses” no es una especificación. Enumera la evidencia que falta.

### Modifica

Clasifica una lista de paquetes instalada en: runtime directo, development directo, transitivo y no relacionado. No declares los transitivos como directos salvo que tu código los use.

### Comprueba en terminal

```bash
python -m pip --version
python -m pip list
python -m pip check
```

Confirma que todas las rutas pertenecen al venv esperado.

---

## 10. `pyproject.toml`: contrato moderno del proyecto

### 10.1 Propósito

`pyproject.toml` centraliza metadata del proyecto, elección de build backend, dependencias declaradas, entry points y configuración namespaced de herramientas. No es un lock file y no describe todo el environment instalado.

TOML usa tablas como `[project]`, strings, lists y keys. PF-M4 enseña solo lo necesario para un proyecto local instalable.

### 10.2 Ejemplo mínimo ejecutable

**Ejemplo ejecutable — `pyproject.toml`:**

```toml
[build-system]
requires = ["setuptools>=80"]
build-backend = "setuptools.build_meta"

[project]
name = "eidolon-journal-pf4"
version = "0.1.0"
description = "Journal sintético instalable para PF-M4"
readme = "README.md"
requires-python = ">=3.14"
dependencies = []

[project.optional-dependencies]
dev = []

[project.scripts]
eidolon = "eidolon.cli:main"

[tool.setuptools.packages.find]
where = ["src"]
```

La elección concreta de setuptools permite ejecutar el ejemplo; no es una declaración de que todo proyecto deba usarlo. Otra herramienta usaría su build backend y su tabla `[tool.<name>]`, mientras los conceptos de metadata, build, dependencies y entry points permanecen.

### 10.3 `[build-system]`

- `requires` declara lo necesario para construir la distribución;
- `build-backend` identifica la implementación que responde al frontend de build/install.

Estas son build dependencies, no runtime dependencies de EIDOLON. El constraint del ejemplo exige una versión reciente compatible con el runtime declarado, pero no produce un lock exacto.

En una primera instalación, el frontend puede necesitar obtener ese build requirement desde el índice configurado. En un entorno offline debe existir previamente en un cache o repositorio aprobado. Un fallo al preparar el backend se diagnostica como problema de build/environment; no demuestra por sí solo que los imports del source sean incorrectos.

### 10.4 `[project]`

- `name` es el nombre de distribución;
- `version` identifica esta release del proyecto;
- `requires-python` restringe runtimes compatibles;
- `dependencies` declara requirements runtime directos;
- `description` y `readme` alimentan metadata y documentación.

El package importable sigue llamándose `eidolon`; la distribución se llama `eidolon-journal-pf4`.

### 10.5 Dependencias opcionales/development

`[project.optional-dependencies]` define extras instalables. El ejemplo deja `dev = []` deliberadamente: PF-M4 demuestra separación sin agregar tooling innecesario. Cuando PF-M9 elija herramientas de testing, podrán declararse con constraints revisados e instalarse mediante:

```bash
python -m pip install -e ".[dev]"
```

No todos los gestores modelan development groups de la misma forma. Distingue el concepto —tooling no requerido en runtime— de la sintaxis concreta del workflow elegido.

### 10.6 Entry points

```toml
[project.scripts]
eidolon = "eidolon.cli:main"
```

Al instalar, el tooling crea un launcher `eidolon` que importa `eidolon.cli`, obtiene `main` y la llama. `main` debe recibir cero argumentos en este contrato y devolver `None` o un código de salida apropiado. PF-M4 no desarrolla parsing de argumentos.

### 10.7 Configuración de herramientas

Las tablas `[tool.<name>]` pertenecen a cada herramienta. En el ejemplo:

```toml
[tool.setuptools.packages.find]
where = ["src"]
```

indica al backend concreto dónde descubrir packages. No agregues configuración copiada si no entiendes qué herramienta la consume.

### 10.8 Lo que `pyproject.toml` no resuelve solo

- no instala el proyecto hasta ejecutar una operación de instalación;
- no prueba que los imports sean correctos;
- no bloquea necesariamente todas las versiones transitivas;
- no garantiza compatibilidad entre plataformas;
- no sustituye README, checks ni un environment limpio.

### Predice

¿Qué nombre usa `pip show`: `eidolon` o `eidolon-journal-pf4`? ¿Qué nombre usa `import`?

### Explica

Separa build dependency, runtime dependency y development dependency con un ejemplo de cada categoría conceptual.

### Detecta el bug

```toml
[project]
name = "eidolon-journal-pf4"
dependencies = ["everything-i-found-in-pip-list"]
```

Explica por qué copiar el environment no identifica intención, provenance ni uso directo.

### Modifica

Cambia el nombre del comando de `eidolon` a `eidolon-pf4` sin cambiar el import path. Reinstala y comprueba qué launcher aparece.

### Comprueba en terminal

```bash
python -m pip install -e ".[dev]"
python -m pip show eidolon-journal-pf4
eidolon
```

La instalación debe crear el comando y `pip show` debe reportar la distribución, versión `0.1.0` y ubicación del environment.

---

## 11. Version constraints, locking y reproducibilidad

### 11.1 Tres estados distintos

```text
declaración → resolución/lock → environment instalado
```

- **declaración:** qué acepta el proyecto, por ejemplo un rango de versiones;
- **resolución/lock:** selección concreta de versiones directas y transitivas bajo una plataforma/política;
- **environment:** lo que realmente está instalado aquí y ahora.

Confundirlos produce el falso argumento “está en mi venv, por tanto es reproducible”.

### 11.2 Version constraints

Ejemplos conceptuales:

```text
package-a>=2
package-a>=2,<3
package-a==2.4.1
```

- un mínimo abierto permite recibir cambios futuros amplios;
- un rango compatible limita cambios mayores, pero todavía puede resolver distinto mañana;
- un pin exacto selecciona una versión, pero no garantiza por sí solo artifacts, plataforma o transitivos.

No existe una regla universal de “siempre pin” o “nunca pin”. Una library y una application pueden publicar contratos distintos. EIDOLON P0 necesita una resolución reproducible para su environment de desarrollo/build, sin fingir que un archivo sirve igual para todas las plataformas.

### 11.3 Lock files

Un lock file registra una resolución concreta según el gestor y workflow. Puede incluir transitivos, hashes, markers o plataformas. No existe un único formato universal usado de igual manera por todas las herramientas Python.

PF-M4 exige comprender el propósito y aplicar el mecanismo elegido por el proyecto. No centra el aprendizaje en Poetry, uv, PDM, pip-tools ni otro producto. Si se elige uno después, documenta:

- cuál archivo es fuente de intención;
- cómo se actualiza el lock;
- cómo se instala sin resolver de nuevo;
- qué plataformas cubre;
- cómo se revisa el diff.

### 11.4 Actualizaciones controladas

Una actualización profesional:

1. parte de intención declarada;
2. cambia constraints o regenera el lock deliberadamente;
3. revisa versión, provenance, mantenimiento y cambios relevantes;
4. instala en un environment nuevo;
5. ejecuta checks;
6. conserva un camino de rollback.

No ejecutes upgrades indiscriminados y entregues el environment resultante sin registrar qué cambió.

### 11.5 Reproducibilidad proporcional

En PF-M4, evidencia suficiente:

- `pyproject.toml` válido;
- mecanismo de resolución/locking descrito, aunque el ejemplo sin runtime dependencies no necesite un lock voluminoso;
- install desde venv vacío;
- import y entry point desde otra carpeta;
- `pip check` exitoso;
- instrucciones que no dependan de paquetes globales.

La reproducibilidad hermética de build, artifacts y supply chain se profundiza en D12 y fases posteriores.

### Predice

Dos personas instalan mañana `package-a>=2` sin lock. ¿Por qué podrían obtener versiones diferentes sin que ninguna ignore la declaración?

### Explica

¿Qué pregunta responde un constraint y cuál responde un lock?

### Detecta el bug

Un README dice “reproducible” porque incluye output de `pip freeze` de una laptop. Identifica plataforma, intención y proceso de actualización que siguen sin contrato.

### Modifica

Escribe una política de actualización de cinco pasos para una dependencia directa. No elijas una herramienta específica.

### Comprueba en terminal

En dos venvs nuevos instala el mismo proyecto y compara:

```bash
.venv/bin/python -m pip list --format=freeze
.venv-clean/bin/python -m pip list --format=freeze
```

Explica diferencias antes de concluir que existe una falla. `pip`, build tooling y editable installs pueden variar si no forman parte del mismo contrato.

---

## 12. Dependency hygiene y supply chain básica

### 12.1 Cada dependencia añade código externo

Una dependencia puede ahorrar trabajo y aportar mantenimiento especializado. También añade:

- código que se ejecuta durante build o runtime;
- releases y constraints que debes seguir;
- transitivos;
- superficie de fallos y supply chain;
- costo de actualización, licencia y abandono.

Antes de instalar pregunta:

> ¿Python o el código pequeño ya existente resuelve esto suficientemente bien?

No significa rechazar paquetes; exige que el beneficio supere el costo.

### 12.2 Provenance y nombre

Verifica de dónde proviene el paquete, quién lo mantiene, qué nombre de distribución es correcto y qué índice/configuración usa el installer. Un nombre parecido no demuestra identidad.

**Typosquatting** consiste en publicar un paquete con nombre parecido a uno legítimo para provocar instalaciones por error. Por eso `pip install` no debe ser una reacción automática a un import fallido.

### 12.3 Mantenimiento y abandono

Señales que requieren evaluación:

- releases inexistentes o incompatibles con Python requerido;
- issues de seguridad/mantenimiento sin respuesta;
- ownership o provenance confusos;
- documentación que recomienda comandos opacos;
- transitivos desproporcionados para una función pequeña.

Ausencia de releases recientes no prueba abandono por sí sola: un paquete estable puede cambiar poco. Evalúa contexto y necesidad.

### 12.4 Dependencias pequeñas en EIDOLON P0

El journal de PF-M4 usa solo standard library en runtime. `dependencies = []` es una decisión, no un hueco. Añadir una library para imprimir tres líneas o deduplicar IDs no mejora el MVP.

Cuando una dependencia sea necesaria:

1. documenta el problema;
2. compara standard library/código local;
3. verifica provenance y mantenimiento;
4. declara uso directo y constraint;
5. actualiza resolución/lock;
6. prueba en limpio;
7. define cómo retirarla.

Security profunda, SBOM, firmas, índices privados y response a vulnerabilidades pertenecen a D12. PF-M4 solo evita instalaciones arbitrarias y conserva provenance mínima.

### Predice

¿Una dependencia con cero runtime imports puede ejecutar código durante build? Relaciónalo con `[build-system].requires`.

### Explica

¿Por qué “tiene muchas descargas” no basta como evaluación de provenance o fitness?

### Detecta el bug

Un tutorial dice `pip install eidloon` sin verificar spelling ni publisher. Identifica el riesgo antes de ejecutar.

### Modifica

Toma una dependencia propuesta y escribe una alternativa con standard library. Decide con criterios de mantenimiento, código y claridad, no por moda.

### Comprueba en terminal

```bash
python -m pip show eidolon-journal-pf4
python -m pip check
```

Verifica que la distribución consultada sea la local esperada y que sus requirements estén satisfechos.

---

## 13. Entry points y comando `eidolon`

### 13.1 Del módulo a un comando instalado

**Ejemplo ejecutable — `src/eidolon/cli.py`:**

```python
from eidolon.application.summary import build_summary


def sample_events():
    return [
        {"id": "evt-002", "person_id": "person-008", "active": False},
        {"id": "evt-001", "person_id": "person-007", "active": True},
    ]


def main():
    summary = build_summary(sample_events())
    print("EIDOLON journal")
    for event_id in summary["active_ids"]:
        print(event_id)


if __name__ == "__main__":
    main()
```

`sample_events` crea datos nuevos por llamada y evita un global mutable compartido. `main` concentra el efecto de console; application permanece pura.

Con:

```toml
[project.scripts]
eidolon = "eidolon.cli:main"
```

la instalación genera un launcher. No necesitas escribir un shell script ni modificar `PATH` global: la activación del venv expone su directorio de ejecutables.

### 13.2 Dos formas de ejecutar

Después de instalar:

```bash
eidolon
python -m eidolon.cli
```

Ambas deben producir:

```text
EIDOLON journal
evt-001
```

El entry point llama `main` mediante metadata. `python -m` ejecuta el módulo con package context y activa su main guard.

### 13.3 Sin argparse todavía

PF-M4 solo necesita demostrar un comando instalado. Añadir subcommands, help complejo, parsing y errores de usuario distraería del packaging. El journal futuro podrá usar `argparse` cuando su interfaz tenga requisitos claros.

### 13.4 Entry point roto

**Failure case — metadata apunta a un nombre inexistente:**

```toml
[project.scripts]
eidolon = "eidolon.cli:run"
```

Si `cli.py` solo define `main`, el launcher fallará al cargar el atributo. Corrige uno de los dos contratos y reinstala. Conserva `pyproject.toml`, el módulo instalado y el comando usado.

### Predice

¿Se ejecuta el bloque del main guard cuando el launcher importa `eidolon.cli` para obtener `main`?

### Explica

¿Qué responsabilidad tiene `[project.scripts]` y cuál conserva `main()`?

### Detecta el bug

`main()` devuelve el dict completo de summary. Explica por qué un console launcher puede interpretarlo como resultado de salida no apropiado; imprime dentro del borde y devuelve `None` en este contrato.

### Modifica

Cambia el output de `main` sin modificar `domain` ni `application`. Comprueba que las funciones puras mantienen sus asserts.

### Comprueba en terminal

```bash
python -m pip install -e .
eidolon
python -m eidolon.cli
```

Ambas ejecuciones deben coincidir.

---

## 14. Caso progresivo integrado: journal instalable

Esta es la versión mínima completa usada por el resto del módulo.

### 14.1 Árbol

```text
eidolon-journal/
├── pyproject.toml
├── README.md
├── src/
│   └── eidolon/
│       ├── __init__.py
│       ├── domain/
│       │   ├── __init__.py
│       │   └── events.py
│       ├── application/
│       │   ├── __init__.py
│       │   └── summary.py
│       └── cli.py
└── tests/
    └── smoke.py
```

### 14.2 Package markers

**Fragmento ejecutable — contenido válido para cada uno de los tres `__init__.py`:**

```python
"""Package marker sin side effects."""
```

Cada archivo puede usar una docstring específica. No importa submódulos automáticamente.

### 14.3 Dominio

**Ejemplo ejecutable — `src/eidolon/domain/events.py`:**

```python
def is_valid_event(event):
    required_keys = {"id", "person_id", "active"}
    if not required_keys <= event.keys():
        return False
    if type(event["id"]) is not str or not event["id"]:
        return False
    if type(event["person_id"]) is not str or not event["person_id"]:
        return False
    return type(event["active"]) is bool
```

Solo usa contratos ya estudiados. No introduce classes ni type hints.

### 14.4 Aplicación

**Ejemplo ejecutable — `src/eidolon/application/summary.py`:**

```python
from eidolon.domain.events import is_valid_event


def build_summary(source_events):
    valid_events = [event for event in source_events if is_valid_event(event)]
    active_ids = sorted(
        event["id"]
        for event in valid_events
        if event["active"]
    )
    person_ids = sorted({event["person_id"] for event in valid_events})

    return {
        "valid_count": len(valid_events),
        "active_ids": active_ids,
        "person_ids": person_ids,
    }
```

La flecha es `application → domain`. El módulo no imprime ni conoce entry points.

### 14.5 CLI

Usa el `cli.py` de la sección 13. Su flecha es `cli → application`.

### 14.6 Smoke check

**Ejemplo ejecutable — `tests/smoke.py`:**

```python
from eidolon.application.summary import build_summary


events = [
    {"id": "evt-002", "person_id": "person-008", "active": False},
    {"id": "evt-001", "person_id": "person-007", "active": True},
    {"id": "", "person_id": "person-009", "active": True},
]

summary = build_summary(events)

assert summary == {
    "valid_count": 2,
    "active_ids": ["evt-001"],
    "person_ids": ["person-007", "person-008"],
}

print("smoke check: PASS")
```

Este archivo demuestra import e invariantes mínimas después de instalar. No sustituye el diseño de tests de PF-M9.

### 14.7 README mínimo

El README debe declarar:

- Python requerido;
- cómo crear el venv;
- cómo instalar;
- cómo ejecutar `eidolon`;
- cómo ejecutar `python tests/smoke.py`;
- cómo verificar desde otra ubicación;
- que los datos son sintéticos y no existe persistencia.

### 14.8 Qué ocurre al instalar

1. `pip` lee `pyproject.toml`;
2. el frontend prepara el build backend declarado;
3. el backend descubre packages bajo `src`;
4. se instala metadata y código/enlace editable;
5. se crea el entry point;
6. imports resuelven desde el environment, no por casualidad desde la raíz.

No necesitas construir wheels manualmente ni publicar en PyPI en PF-M4.

### 14.9 Comprobación completa

**Terminal POSIX:**

```bash
python3.14 -m venv .venv
. .venv/bin/activate
python -m pip install -e ".[dev]"
python -m compileall -q src
python tests/smoke.py
python -m pip check
eidolon
```

**PowerShell:**

```powershell
py -3.14 -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install -e ".[dev]"
python -m compileall -q src
python .\tests\smoke.py
python -m pip check
eidolon
```

Output estable relevante:

```text
smoke check: PASS
EIDOLON journal
evt-001
```

`pip` y `compileall` pueden producir output adicional dependiente del environment.

### 14.10 Variación de package y failure cases integrados

Provoca de uno en uno:

1. retira temporalmente `src/eidolon/application/__init__.py` y observa que Python moderno puede tratar el directorio como namespace package: incumple el contrato de package regular elegido, pero **no es un fallo garantizado**;
2. cambia el entry point a una función inexistente;
3. invierte la dependencia para crear un ciclo;
4. desinstala la distribución e intenta importar desde una carpeta ajena;
5. ejecuta con el Python base en vez del venv;
6. agrega un import de una distribución global no declarada;
7. renombra un módulo local como `random.py` y verifica su origen.

En la primera variación registra el contrato observado sin inventar una excepción. Para los seis failure cases restantes registra predicción, comando, síntoma estable, causa, corrección y datos de diagnóstico. Restaura la versión válida antes del siguiente caso.

---

## 15. Reproducibilidad desde otra ubicación

### 15.1 Por qué cambiar de directorio

Un proyecto puede depender accidentalmente del current working directory. Ejecutarlo desde otra ubicación comprueba que el import y el launcher provienen del environment instalado.

### 15.2 POSIX

Desde la raíz del proyecto, con `.venv` creado:

```bash
project_dir="$PWD"
cd /tmp
"$project_dir/.venv/bin/python" -c "import eidolon; print(eidolon.__name__)"
"$project_dir/.venv/bin/eidolon"
```

Outputs relevantes:

```text
eidolon
EIDOLON journal
evt-001
```

No conviertas `project_dir` en una variable global del sistema; solo vive en esa shell de comprobación.

### 15.3 PowerShell

```powershell
$ProjectDir = (Get-Location).Path
Set-Location $env:TEMP
& "$ProjectDir\.venv\Scripts\python.exe" -c "import eidolon; print(eidolon.__name__)"
& "$ProjectDir\.venv\Scripts\eidolon.exe"
```

Vuelve después con:

```powershell
Set-Location $ProjectDir
```

### 15.4 Qué demuestra y qué no

Demuestra:

- package instalado;
- entry point disponible mediante el venv;
- ausencia de dependencia directa del repositorio como current directory.

No demuestra todavía:

- reproducibilidad en otra plataforma;
- persistencia correcta;
- tests completos;
- seguridad de todas las dependencias;
- build hermético.

### 15.5 Proyecto que solo funciona desde una carpeta

Causas frecuentes:

- import de un módulo suelto en la raíz;
- package no instalado;
- `PYTHONPATH` manual no documentado;
- ejecución directa de un archivo con relative imports;
- recursos abiertos mediante rutas relativas al current working directory, tema que PF-M6 tratará.

No corrijas rutas de recursos todavía; solo identifica esa frontera futura.

### Predice

Si `eidolon` funciona desde `/tmp`, ¿qué evidencia aporta sobre `src` layout y la instalación?

### Explica

¿Por qué cambiar de cwd es una comprobación distinta de abrir otra terminal en la raíz?

### Detecta el bug

El smoke check hace `sys.path.insert(0, "src")`. Explica por qué invalida la prueba de instalación.

### Modifica

Elimina cualquier hack de path y ejecuta el check con el Python del venv que tiene el proyecto instalado.

### Comprueba en terminal

Ejecuta la secuencia POSIX o PowerShell anterior y conserva los comandos exactos, no las rutas privadas completas, como evidencia reproducible.

---

## 16. Diagnóstico de fallos frecuentes

### 16.1 Matriz de síntomas

| Síntoma | Hipótesis inicial | Evidencia útil | Corrección probable |
|---|---|---|---|
| `ModuleNotFoundError` | environment/package/invocación incorrectos | executable, cwd, spec, instalación | instalar proyecto correcto o corregir invocación |
| import parcialmente inicializado | circular import | traceback y grafo | redirigir responsabilidades |
| `random` sin API esperada | shadowing local | `random.__file__` | renombrar módulo local |
| funciona solo en raíz | import accidental desde cwd | repetir desde otra carpeta | instalar package; retirar path hacks |
| `pip show` no encuentra distribución | pip distinto o nombre de distribución errado | `python -m pip --version`, `[project].name` | usar el mismo Python/nombre correcto |
| dependency falta en limpio | global fantasma o transitiva usada directamente | pyproject + imports + venv limpio | declarar directa o eliminarla |
| launcher no encuentra función | entry point desalineado | `[project.scripts]`, módulo instalado | corregir target y reinstalar |

La tabla genera hipótesis, no diagnósticos automáticos.

### 16.2 Orden de diagnóstico

```text
comando → cwd → interpreter → environment → import spec/origin
        → metadata instalada → grafo de imports → corrección
```

Cambiar cinco cosas a la vez destruye evidencia. Reproduce con un caso mínimo, cambia una frontera y vuelve a comprobar.

### 16.3 Ejecutar desde ubicación incorrecta

Si estás un nivel arriba o dentro de `src/eidolon`, comandos relativos como `pip install .` apuntan al directorio equivocado. Verifica que el cwd contiene el `pyproject.toml` esperado:

**POSIX:**

```bash
pwd
test -f pyproject.toml
```

**PowerShell:**

```powershell
Get-Location
Test-Path .\pyproject.toml
```

No busques recursivamente y elijas el primer archivo sin confirmar el proyecto.

### 16.4 Python y pip distintos

Compara:

```bash
python -c "import sys; print(sys.executable)"
python -m pip --version
```

Ambos deben señalar el mismo venv. Un prompt con `(.venv)` es una pista visual, no evidencia suficiente.

### 16.5 Cache después de renombrar

Tras corregir shadowing o imports, reinicia el proceso. Los módulos ya cargados permanecen en `sys.modules`. Directorios `__pycache__` son derivados y no source; si debes retirarlos, confirma primero la ruta exacta dentro del proyecto. No empieces con borrados amplios.

### Predice

Si `python -m pip show` señala `.venv` pero `sys.executable` señala el Python base, ¿pueden provenir del mismo comando `python`? Revisa tu evidencia.

### Explica

¿Por qué el cwd pertenece al diagnóstico aunque el package esté instalado?

### Detecta el bug

Un estudiante cambia `sys.path`, reinstala globalmente y renombra archivos antes de volver a ejecutar. Explica qué hipótesis ya no puede confirmar.

### Modifica

Convierte un relato “no importa” en una reproducción mínima con comando, cwd, executable, spec y traceback.

### Comprueba en terminal

Ejecuta la cadena de diagnóstico sobre el proyecto válido y guarda solo propiedades no sensibles:

```bash
python -c "import sys; print(sys.prefix != sys.base_prefix)"
python -c "import importlib.util; print(importlib.util.find_spec('eidolon') is not None)"
python -m pip show eidolon-journal-pf4
python -m pip check
```

---

## 17. Ejercicios guiados con solución razonada

### Ejercicio guiado 1 — Separar por responsabilidad

**Objetivo.** Convertir un script sin inventar capas.

**Código inicial:**

```python
def is_valid_event(event):
    return bool(event["id"])


def active_ids(events):
    return [event["id"] for event in events if is_valid_event(event) and event["active"]]


print(active_ids([{"id": "evt-001", "active": True}]))
```

**Predice antes de resolver:** ¿qué función pertenece a domain, cuál a application y qué línea pertenece al borde de console?

**Solución razonada:**

```text
eidolon/domain/events.py       → is_valid_event
eidolon/application/summary.py → active_ids e import de domain
eidolon/cli.py                 → print/main e import de application
```

La dirección es `cli → application → domain`. La solución es correcta si cada módulo puede describirse con una responsabilidad y no existe flecha inversa.

**Comprobación ejecutable:** instala el package y ejecuta `python -m eidolon.cli`; debe imprimir la misma lista que el script inicial.

**Variación.** Si solo hubiera una función y ningún crecimiento previsto, defiende mantener un script.

### Ejercicio guiado 2 — Import sin side effect

**Objetivo.** Separar definición de arranque.

**Código incorrecto:**

```python
print("iniciando")


def main():
    print("journal")


main()
```

**Predice antes de ejecutar:** ¿qué dos líneas se imprimen al importar este archivo y cuál de ellas es un efecto de arranque?

**Solución ejecutable:**

```python
def main():
    print("journal")


if __name__ == "__main__":
    main()
```

Al importar no imprime; al ejecutar imprime `journal`. El main guard corrige el arranque, no otros efectos top-level.

**Criterio:** `python -c "import <módulo>"` no produce output y ejecutar el módulo produce exactamente `journal`.

**Variación.** Agrega una constante inmutable y explica por qué no es equivalente a llamar `print`.

### Ejercicio guiado 3 — Diagnosticar shadowing

**Objetivo.** Confirmar el origen antes de renombrar.

**Input:** un proyecto contiene `random.py`; `import random; random.randint(1, 3)` falla.

**Predice antes de ejecutar:** ¿qué ruta esperas ver si el proyecto está ocultando la standard library?

**Solución:**

```bash
python -c "import random; print(random.__file__)"
```

Si la ruta apunta al proyecto, renombra el archivo local, actualiza imports y reinicia el proceso. El criterio correcto es el origen observado, no asumir que la standard library está rota.

**Comprobación ejecutable:** en un proceso nuevo, `python -c "import random; print(hasattr(random, 'randint'))"` debe imprimir `True` y `random.__file__` ya no debe señalar el antiguo archivo local.

**Variación.** Repite con un nombre que no choque y comprueba el origen.

### Ejercicio guiado 4 — Corregir un circular import

**Objetivo.** Cambiar ownership, no retrasar el error.

**Input:** `domain.events` importa `application.summary`, que importa `domain.events`.

**Predice antes de ejecutar:** ¿qué módulo estará parcialmente inicializado y qué nombre intentará obtener el otro?

**Solución razonada:** conserva validación en domain; application la importa y construye el label; elimina el import de application desde domain.

```text
antes:   domain ↔ application
después: application → domain
```

Comprueba import y llamada en un proceso nuevo. La solución no cuenta si solo movió ambos imports dentro de funciones.

**Criterio:** el grafo final no tiene ciclo y `event_label({"id": "evt-001"})` devuelve `event:evt-001`.

### Ejercicio guiado 5 — Verificar el environment

**Objetivo.** Asociar installer e interpreter.

**Predice antes de ejecutar:** anota qué ruta debería aparecer para Python, qué ruta debería reportar pip y qué booleano confirma el venv.

**Solución POSIX/PowerShell:**

```bash
python -c "import sys; print(sys.executable)"
python -m pip --version
python -c "import sys; print(sys.prefix != sys.base_prefix)"
```

Las dos rutas deben pertenecer al venv y la última propiedad debe ser `True`. Un prompt activado sin estas propiedades no satisface el criterio.

**Variación.** Ejecuta el Python del venv por ruta sin activación.

### Ejercicio guiado 6 — Distinguir import y distribución

**Objetivo.** Consultar el nombre correcto.

**Predice antes de ejecutar:** ¿cuál comando consulta el package importable y cuál consulta metadata de la distribución?

**Solución:**

```bash
python -c "import eidolon; print(eidolon.__name__)"
python -m pip show eidolon-journal-pf4
```

`eidolon` es import package; `eidolon-journal-pf4`, distribution name. Ambos contratos proceden del mismo proyecto, pero no son aliases universales.

**Criterio:** el import produce `eidolon` y `pip show` encuentra nombre y versión de `eidolon-journal-pf4` en el mismo venv.

### Ejercicio guiado 7 — Declarar el proyecto mínimo

**Objetivo.** Relacionar cada tabla TOML con su consumidor.

**Predice antes de modificar:** para cada tabla del ejemplo, anota qué operación dejaría de funcionar o qué metadata se perdería si la retiras.

**Solución:** usa el `pyproject.toml` de la sección 10 y explica:

- frontend/build backend para `[build-system]`;
- installer/metadata para `[project]`;
- launcher para `[project.scripts]`;
- setuptools para `[tool.setuptools.packages.find]`.

La solución accidental solo copia TOML; la correcta puede retirar una tabla y predecir qué contrato se rompe.

**Comprobación ejecutable:** restaura el archivo válido, ejecuta `python -m pip install -e .`, `python -m pip show eidolon-journal-pf4` y `eidolon`.

### Ejercicio guiado 8 — Detectar una dependencia transitiva usada directamente

**Objetivo.** Corregir intención declarada.

**Input:** EIDOLON declara `package-a`, pero su código importa `package-b`, instalado transitivamente.

**Predice antes de resolver:** ¿qué import puede romperse si `package-a` deja de requerir `package-b`, aunque la API de `package-a` siga siendo compatible?

**Solución razonada:** si el import directo es necesario, declara `package-b` como dependencia directa con constraint deliberado; si no, usa solo la API pública de `package-a`. Después reinstala en limpio.

La solución no consiste en copiar todas las transitivas a dependencies.

**Criterio:** un venv limpio instala desde la declaración y el código no depende de una transitiva accidental.

### Ejercicio guiado 9 — Comprobar desde otra carpeta

**Objetivo.** Detectar dependencia del cwd.

**Predice antes de ejecutar:** ¿qué debería ocurrir con el import y el launcher después de cambiar a `/tmp` o `$env:TEMP`?

**Solución POSIX:**

```bash
project_dir="$PWD"
cd /tmp
"$project_dir/.venv/bin/python" -c "import eidolon"
"$project_dir/.venv/bin/eidolon"
```

**Solución PowerShell:** usa `$ProjectDir`, `$env:TEMP` y los executables bajo `.venv\Scripts` como en la sección 15.

El criterio es que no se añada `src` a `sys.path` manualmente.

**Variación.** Desinstala la distribución del venv y repite el import desde otra carpeta para observar el contraste.

### Ejercicio guiado 10 — Recrear el environment

**Objetivo.** Probar instrucciones, no conservar estado.

**Predice antes de ejecutar:** enumera qué outputs estables deben coincidir y qué metadata ambiental puede diferir entre los dos venvs.

**Solución:** crea `.venv-clean`, instala `.[dev]`, ejecuta compileall, smoke check, `pip check` y entry point. Compara outputs con `.venv`.

No elimines el environment anterior hasta completar la comparación. La evidencia es una secuencia repetible desde un directorio que solo contiene el proyecto declarado.

**Criterio:** la instalación no editable de `.venv-clean` pasa smoke check, `pip check` y entry point desde otra ubicación, sin reutilizar packages de `.venv`.

---

## 18. Ejercicios independientes

1. **Namespaces.** Crea dos módulos con una función `normalize`. Importa ambos sin colisión y explica los bindings.
2. **Import styles.** Implementa tres variantes del mismo import y defiende una según procedencia y claridad.
3. **Top-level.** Detecta cinco efectos de import en un módulo y mueve cada uno a una función o borde explícito.
4. **Cache.** Importa dos veces un módulo con una observación controlada. Explica `sys.modules` sin usar reload.
5. **`__name__`.** Predice y comprueba ejecución como archivo, import y `python -m`.
6. **ModuleNotFoundError.** Produce un fallo por spelling y otro por environment incorrecto. Conserva evidencia distinta.
7. **Shadowing.** Reproduce `random.py`, confirma `__file__`, corrige y reinicia.
8. **Package.** Convierte tres módulos sueltos en package regular con `__init__.py` sin side effects.
9. **Absolute/relative.** Implementa ambos estilos dentro de un subpackage y explica cuándo falla la ejecución directa.
10. **Tres nombres.** Elige package name, module name y distribution name distintos pero claros; muestra cómo consultarlos.
11. **Circular import.** Construye un ciclo mínimo, dibuja el grafo y elimínalo por ownership.
12. **Import local.** Retrasa un import circular dentro de una función y demuestra por qué el diseño sigue acoplado.
13. **Layout.** Compara import antes de instalar en flat y `src` layout.
14. **Venv.** Crea dos environments, instala el proyecto en uno y explica el fallo en el otro.
15. **Python/pip.** Provoca una confusión de environments sin instalar nada global; diagnostica con rutas.
16. **Pip.** Instala editable, consulta, comprueba y desinstala la distribución local.
17. **Global fantasma.** Describe cómo un paquete preinstalado puede ocultar una declaración ausente y compruébalo con venv limpio.
18. **Directa/transitiva.** Dibuja tres dependencias y clasifica cada flecha desde el punto de vista de EIDOLON.
19. **Pyproject.** Explica cada key del ejemplo y elimina una para observar un failure case controlado.
20. **Entry point.** Cambia el target a un nombre inválido, reinstala, diagnostica y restaura.
21. **Constraints.** Compara mínimo, rango y pin exacto sin afirmar reproducibilidad completa.
22. **Locking.** Escribe el contrato que exigirías a una herramienta de lock sin elegir una por popularidad.
23. **Update.** Diseña una actualización con environment nuevo, checks y rollback.
24. **Dependency hygiene.** Evalúa una dependencia que duplica una operación de standard library.
25. **Typosquatting.** Diseña una checklist previa a instalar un nombre recibido en un tutorial.
26. **Otra ubicación.** Ejecuta import, smoke check y entry point desde un directorio temporal.
27. **Otro clon/copia.** Copia solo archivos fuente declarados, crea un venv nuevo y reproduce el proyecto.
28. **Review.** Revisa un README que depende de `PYTHONPATH`, packages globales y cwd fijo; propón cambios mínimos.

---

## 19. Preguntas conceptuales

1. ¿Qué diferencia existe entre function scope y module namespace?
2. ¿Por qué separar un archivo puede disminuir cohesión en vez de aumentarla?
3. ¿Qué bindings crea `import module` frente a `from module import name`?
4. ¿Qué código ejecuta Python al importar un módulo por primera vez en un proceso?
5. ¿Qué garantiza y qué no garantiza el import cache?
6. ¿Por qué un main guard no vuelve instalable un proyecto?
7. ¿Qué evidencia necesitas antes de diagnosticar `ModuleNotFoundError`?
8. ¿Por qué modificar `sys.path` puede ocultar el problema real?
9. ¿Cómo demuestra `random.__file__` un caso de shadowing?
10. ¿Qué diferencia existe entre módulo, package y distribución?
11. ¿Cuándo un relative import mejora claridad y cuándo la reduce?
12. ¿Por qué ejecutar un archivo interno directamente puede perder package context?
13. ¿Qué relación existe entre circular import y dirección de dependencias?
14. ¿Cuándo un import local es razonable y por qué no es una cura general?
15. ¿Qué error accidental ayuda a descubrir un `src` layout?
16. ¿Por qué activation solo es una conveniencia sobre `PATH`?
17. ¿Qué aislamiento no ofrece un venv?
18. ¿Por qué `python -m pip` reduce ambigüedad?
19. ¿Qué diferencia existe entre estado instalado e intención declarada?
20. ¿Cuándo una transitive dependency debe declararse directa?
21. ¿Qué contratos pertenecen a `[build-system]` y `[project]`?
22. ¿Por qué distribution name e import package pueden diferir?
23. ¿Qué crea `[project.scripts]` y qué función sigue perteneciendo al código?
24. ¿Qué diferencia existe entre constraint y lock?
25. ¿Por qué un pin exacto no garantiza por sí solo reproducibilidad completa?
26. ¿Qué costo añade una dependencia aunque no se importe en runtime?
27. ¿Qué es typosquatting y qué hábito reduce el riesgo?
28. ¿Qué prueba ejecutar desde otra ubicación y qué deja sin demostrar?

---

## 20. Mini challenge — Journal instalable y reproducible

### 20.1 Objetivo

Convierte un pequeño journal EIDOLON de varios archivos sueltos en una distribución local instalable con `src` layout, imports dirigidos, venv limpio, `pyproject.toml` y comando `eidolon`.

El challenge prepara **PF-L06 — Paquete instalable local** y la reproducibilidad de EIDOLON 0.0a.

### 20.2 Punto de partida

```text
starter/
├── eidolon.py
├── event_rules.py
├── summary.py
└── check.py
```

Problemas deliberados:

- `eidolon.py` importa módulos sueltos y ejecuta al importarse;
- `event_rules.py` importa `summary.py` para construir labels;
- `summary.py` importa `event_rules.py`, creando un ciclo;
- `check.py` inserta el cwd en `sys.path`;
- el proyecto carece de metadata, venv documentado y entry point;
- funciona solo al ejecutarse desde `starter/`.

**Código incorrecto — `event_rules.py`:**

```python
from summary import event_label


def is_valid_event(event):
    return bool(event["id"])


def describe_event(event):
    return event_label(event)
```

**Código incorrecto — `summary.py`:**

```python
from event_rules import is_valid_event


def active_ids(events):
    return sorted(
        event["id"]
        for event in events
        if is_valid_event(event) and event["active"]
    )


def event_label(event):
    return "event:" + event["id"]
```

**Código incorrecto — `eidolon.py`:**

```python
from summary import active_ids


events = [
    {"id": "evt-002", "active": False},
    {"id": "evt-001", "active": True},
]

print("EIDOLON journal")
for event_id in active_ids(events):
    print(event_id)
```

**Código incorrecto — `check.py`:**

```python
import sys


sys.path.insert(0, ".")
import eidolon
```

Desde `starter/`, `python eidolon.py` reproduce el circular import antes de imprimir el journal: `summary` intenta importar `is_valid_event`, mientras `event_rules` intenta obtener `event_label` desde un `summary` parcialmente inicializado. Una vez eliminado el ciclo, importar `eidolon.py` revela el segundo defecto: ejecuta el output de aplicación durante el import. Provoca y corrige cada fallo por separado para conservar evidencia causal.

### 20.3 Contrato funcional

La versión final usa events sintéticos y produce:

```text
EIDOLON journal
evt-001
```

`domain` valida events; `application` produce IDs activos ordenados; `cli` imprime. No existe persistencia.

### 20.4 Estructura requerida

```text
eidolon-journal/
├── pyproject.toml
├── README.md
├── src/
│   └── eidolon/
│       ├── __init__.py
│       ├── domain/
│       │   ├── __init__.py
│       │   └── events.py
│       ├── application/
│       │   ├── __init__.py
│       │   └── summary.py
│       └── cli.py
└── tests/
    └── smoke.py
```

Puedes proponer un layout más pequeño si satisface los mismos contratos y justificas cómo detecta imports accidentales. Para la ruta principal usa `src`.

### 20.5 Requisitos numerados

1. Mueve validación pura a `eidolon.domain.events`.
2. Mueve summary a `eidolon.application.summary`.
3. Mantén console en `eidolon.cli.main`.
4. Elimina la flecha `domain → application`; conserva `cli → application → domain`.
5. Usa packages regulares con `__init__.py` sin side effects.
6. Prefiere absolute imports entre responsabilidades.
7. No modifiques `sys.path` ni exijas `PYTHONPATH`.
8. Usa el `pyproject.toml` mínimo de este módulo, con distribution name `eidolon-journal-pf4`.
9. Declara Python `>=3.14`, runtime dependencies vacías y extra `dev` vacío.
10. Define `eidolon = "eidolon.cli:main"`.
11. Crea `.venv` y verifica interpreter/pip.
12. Instala editable para desarrollo y ejecuta smoke check.
13. Crea `.venv-clean` e instala de forma no editable.
14. Ejecuta import y entry point desde otra ubicación.
15. Documenta comandos POSIX y Windows en README.
16. No dependas de una distribución global o transitiva no declarada.

### 20.6 Comprobaciones obligatorias

Desde `.venv`:

```bash
python -m compileall -q src
python tests/smoke.py
python -m pip check
python -c "import eidolon; print(eidolon.__name__)"
python -c "from eidolon.application.summary import build_summary; print(callable(build_summary))"
eidolon
```

Outputs estables:

```text
smoke check: PASS
eidolon
True
EIDOLON journal
evt-001
```

Desde `.venv-clean`, repite smoke check, `pip check` y `eidolon`.

### 20.7 Comprobación desde otra ubicación

Usa los comandos de la sección 15. No copies `src` al directorio temporal ni agregues path hacks. El import debe resolverse mediante la instalación.

### 20.8 Failure cases obligatorios

Provoca y documenta por separado:

1. `ModuleNotFoundError` antes de instalar el `src` layout;
2. circular import del starter;
3. side effect al importar el antiguo `eidolon.py`;
4. shadowing mediante un `random.py` local;
5. ejecución directa de un módulo con relative import y sin package context;
6. pip e interpreter de environments distintos;
7. distribución instalada en `.venv` pero ausente en `.venv-clean` antes de instalar;
8. entry point que apunta a una función inexistente;
9. dependencia global ficticia no declarada;
10. uso directo de una transitive dependency no declarada;
11. proyecto que funciona desde la raíz pero falla desde otra carpeta.

Para cada uno registra:

- predicción;
- comando y cwd;
- síntoma estable;
- causa;
- corrección;
- `sys.executable`, origen/spec o metadata relevante.

### 20.9 Artefactos esperados

- árbol final;
- source code sin clases ni type hints;
- `pyproject.toml`;
- README reproducible;
- `.gitignore` que excluye `.venv/`, `.venv-clean/`, `__pycache__/` y metadata de build local;
- smoke check con asserts;
- diagrama de dependencias antes/después;
- tabla de direct/transitive/development dependencies;
- registro de los once failure cases.

### 20.10 Criterio de aprobación

Apruebas si otra persona puede copiar solo los archivos declarados, crear un venv vacío, instalar, ejecutar checks y usar `eidolon` desde otra carpeta. Debes explicar por qué cada import apunta en esa dirección y por qué `pyproject.toml` no equivale a un lock.

### 20.11 Límites

No agregues OOP, dataclasses, type hints, repositories, archivos de datos, JSON, decorators, context managers, async, database, backend, Docker, publicación a PyPI ni frameworks. No conviertas el smoke check en una suite avanzada.

---

## 21. Resumen

- Un módulo aporta un namespace y una frontera de responsabilidad; más archivos no garantizan cohesión.
- `import module` enlaza el módulo; `from module import name` enlaza un nombre suyo; aliases deben aclarar.
- Python ejecuta top-level al inicializar un módulo y normalmente reutiliza el objeto desde `sys.modules`.
- Los imports no deben disparar efectos de aplicación.
- `__name__ == "__main__"` separa import de arranque directo, pero no reemplaza packaging.
- `sys.path` depende del interpreter, environment e invocación; no se parchea antes de diagnosticar.
- `ModuleNotFoundError` puede indicar spelling, environment, instalación, layout o invocación.
- Un archivo local como `random.py` puede ocultar un módulo esperado; verifica su origen.
- Un package agrupa módulos; una distribución instala packages y metadata; sus nombres pueden diferir.
- `__init__.py` debe permanecer pequeño y sin side effects en el package regular de PF-M4.
- Absolute imports suelen mostrar mejor dependencias entre responsabilidades; relative imports pueden ser claros localmente.
- Los circular imports suelen revelar ownership bidireccional; retrasar imports no elimina el grafo.
- `src` layout evita que la raíz haga importable el package por accidente y prueba la instalación real.
- Un venv aísla paquetes, pero no declara versiones ni aísla el sistema completo.
- Activation modifica `PATH`; usar el executable del venv directamente es equivalente para el proceso.
- `python -m pip` asocia installer con el interpreter verificado.
- Instalar modifica un environment; declarar expresa intención; un lock registra una resolución.
- Una dependencia transitiva usada directamente debe convertirse en directa o dejar de importarse.
- `pyproject.toml` declara build system, metadata, requirements, extras, entry points y tool config.
- `[project.scripts]` convierte una función sin argumentos en un comando instalado.
- Cada dependencia agrega código y mantenimiento; provenance y spelling se verifican antes de instalar.
- Ejecutar desde otra ubicación detecta dependencia accidental del cwd.

---

## 22. Checklist de dominio

- [ ] Puedo explicar cuándo extraer un módulo por cohesión y responsabilidad.
- [ ] Puedo distinguir scope local y namespace de módulo.
- [ ] Puedo elegir entre `import module`, `from ... import ...` y aliases.
- [ ] Puedo predecir efectos top-level durante import.
- [ ] Puedo explicar el import cache sin depender de reload internals.
- [ ] Puedo usar `__name__` y un main guard correctamente.
- [ ] Puedo eliminar side effects de import.
- [ ] Puedo inspeccionar `sys.executable`, `sys.path` y module spec/origin.
- [ ] Puedo diagnosticar `ModuleNotFoundError` sin instalar nombres arbitrarios.
- [ ] Puedo detectar shadowing de standard library.
- [ ] Puedo distinguir módulo, package y distribución.
- [ ] Puedo crear packages regulares con `__init__.py` mínimos.
- [ ] Puedo justificar absolute o relative imports.
- [ ] Puedo detectar ejecución sin package context.
- [ ] Puedo dibujar y reproducir un circular import.
- [ ] Puedo refactorizar un ciclo cambiando ownership.
- [ ] Puedo explicar por qué un import local no siempre resuelve el diseño.
- [ ] Puedo justificar flat o `src` layout.
- [ ] Puedo crear y verificar venvs en POSIX y Windows.
- [ ] Puedo ejecutar el Python del venv sin activación.
- [ ] Puedo recrear un environment sin copiar `.venv`.
- [ ] Puedo usar `python -m pip` para instalar, listar, mostrar, comprobar y desinstalar.
- [ ] Puedo distinguir instalación editable de instalación regular.
- [ ] Puedo distinguir dependencia directa, transitiva y development.
- [ ] Puedo detectar una dependencia global fantasma.
- [ ] Puedo explicar por qué `pip install` y `pip freeze` no bastan como contrato.
- [ ] Puedo escribir un `pyproject.toml` mínimo y explicar cada tabla.
- [ ] Puedo distinguir import name de distribution name.
- [ ] Puedo definir y comprobar un console entry point.
- [ ] Puedo distinguir constraint, lock y environment instalado.
- [ ] Puedo proponer una actualización controlada con rollback.
- [ ] Puedo evaluar costo, provenance, mantenimiento y typosquatting.
- [ ] Puedo mantener EIDOLON P0 sin dependencias runtime innecesarias.
- [ ] Puedo ejecutar smoke checks desde otra ubicación sin path hacks.
- [ ] Puedo completar el mini challenge solo con PF-M1–PF-M4.

---

## 23. Preparación para labs y EIDOLON 0.0a

Después de dominar PF-M4 puedes comenzar:

- **PF-L06 — Paquete instalable local:** es el laboratorio principal. Debes entregar `src` layout, entry point, `pyproject.toml`, venv limpio y separación de dependencias.
- **PF-L03 — Funciones sin estado oculto:** ahora sus funciones puras pueden ubicarse en módulos con responsabilidades explícitas.
- **PF-L04 — Índice de entidades:** los índices in-memory pueden vivir en application sin convertirse en source of truth.
- **PF-L05 — Stream de eventos:** PF-M4 aporta package/environment; aún faltan archivos, JSONL y lifecycle de PF-M6.
- **PF-L14 — Pytest de fronteras:** el layout y extra dev quedan preparados, pero PF-M9 enseñará la estrategia de testing.
- **EIDOLON 0.0a:** el proyecto ya puede instalarse y exponer un comando. Aún faltan modelos de PF-M5, persistencia/errores de PF-M6 y testing profesional de PF-M9.

### Evidencia antes de avanzar a PF-M5

1. mini challenge reproducido en `.venv` y `.venv-clean`;
2. árbol final y grafo de imports defendidos;
3. diez ejercicios guiados ejecutados;
4. al menos doce ejercicios independientes, incluidos 3, 6, 7, 11, 14, 15, 18, 19, 20, 24, 26 y 27;
5. outputs de `sys.executable`, `pip --version`, `pip show` y `pip check` sin rutas sensibles en la entrega;
6. circular import reproducido y corregido por ownership;
7. shadowing reproducido y diagnosticado por origen;
8. instalación y entry point ejecutados desde otra ubicación;
9. nota titulada **“Qué dependencia no añadiría y por qué”**;
10. explicación oral de cinco minutos o nota equivalente sobre declaración, lock y environment.

Este módulo satisface la preparación directa de **PF-L06** y refuerza el **CHECKPOINT PF-C2 — Diseño y lifecycle** en su parte de package/environment reproducible.

---

## 24. Recursos de ampliación

La explicación fundamental está contenida en este módulo. Para verificar sintaxis y comportamiento consulta selectivamente la documentación oficial de Python 3.14 sobre modules, packages, `venv`, import system y `sys.path`, además de Python Packaging User Guide para `pyproject.toml` e instalación en virtual environments.

Los recursos canónicos del track permanecen en [`PF.11 Recursos recomendados`](../../02_curriculum/01_programming_foundations.md#pf11-recursos-recomendados). Úsalos para ampliar, no para sustituir este capítulo.

---

## 25. Límite del módulo

PF-M4 termina aquí. **PF-M5** estudiará OOP, dataclasses y type hints; **PF-M6**, excepciones, archivos, JSON y lifecycle; **PF-M7**, decorators y context managers; **PF-M8**, async/await; y **PF-M9**, testing, debugging y logging avanzados.

No se desarrollan import hooks, custom loaders, namespace packages distribuidos, publicación a PyPI, construcción manual de wheels, monorepos, containers ni CI/CD. Locking se enseña como contrato y no se impone una herramienta por moda.

Tampoco se crean repositories reales, adapters de persistencia, bases de datos, backend, AI ni arquitectura empresarial. La frontera lograda es verificable: módulos cohesivos, package instalable, imports dirigidos, environment aislado, dependencies declaradas y entry point reproducible.
