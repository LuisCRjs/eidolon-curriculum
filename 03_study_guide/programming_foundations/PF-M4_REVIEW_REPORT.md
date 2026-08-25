# PF-M4 — Reporte de revisión técnica, pedagógica, curricular y editorial

**Artefacto revisado:** `PF-M4_modules_packages_dependency_management.md`  
**Curriculum source:** [PF-M4](../../02_curriculum/01_programming_foundations.md#pf-m4--módulos-paquetes-y-dependency-management)  
**Revisión:** v0.1  
**Fecha:** 2026-08-25  
**Estado del reporte:** completado  
**Decisión después de correcciones:** PASS

## 1. Alcance y método

La revisión utilizó únicamente las fuentes canónicas necesarias:

- `00_master_prompt/EIDOLON_MASTER_PROMPT.md`;
- `01_blueprint/EIDOLON_CURRICULUM_BLUEPRINT_v0.2.md`;
- `02_curriculum/01_programming_foundations.md`;
- `03_study_guide/STUDY_GUIDE_EDITORIAL_STANDARD_v0.1.md`;
- la versión `review candidate` de `PF-M4_modules_packages_dependency_management.md`.

No fue necesario consultar PF-M1–PF-M3: PF-M4 declara y reutiliza sus prerequisites sin reabrir su contenido.

Se evaluaron cuatro dimensiones:

1. **Técnica:** módulos, namespaces, resolución y estilos de import, packages, circularidad, shadowing, venv, pip, `pyproject.toml`, entry points, dependencias y reproducibilidad.
2. **Pedagógica:** progresión problema → modelo mental → mecanismo → ejemplo → fallo → corrección → aplicación EIDOLON, con práctica distribuida.
3. **Curricular:** cobertura completa de PF-M4 y PF-L06 sin desarrollar PF-M5–PF-M9, Backend, databases, Docker o Security Track.
4. **Editorial:** metadata, jerarquía, clasificación de bloques, outputs, failure cases, ejercicios, challenge, checklist, links y no duplicación.

La validación se ejecutó con Python 3.14.7 sobre Linux. Los comandos PowerShell y Command Prompt se revisaron por sintaxis, rutas y equivalencia semántica; no se ejecutaron en Windows.

## 2. Resumen de hallazgos

| Severidad | Encontrados | Corregidos | Pendientes para el gate |
|---|---:|---:|---:|
| CRITICAL | 0 | 0 | 0 |
| IMPORTANT | 3 | 3 | 0 |
| MINOR | 2 | 2 | 0 |
| OPTIONAL | 0 | 0 | 0 |

Las correcciones fueron localizadas. Se preservaron la estructura, los resultados de aprendizaje, la práctica, los ejercicios, el challenge y los límites del módulo.

## 3. Hallazgos IMPORTANT

### I-01 — El ejemplo de import cache no era autónomo ni mostraba su output real

**Ubicación:** sección 3.2, “Import cache al nivel necesario”.

**Problema:** el bloque estaba etiquetado como ejemplo ejecutable aunque dependía del `event_rules.py` anterior. Además, los comentarios solo anticipaban dos valores booleanos y omitían `registrando reglas`, que realmente se imprime al primer import.

**Impacto:** contradecía la clasificación de código del estándar y podía hacer creer que recuperar el módulo desde cache elimina también el side effect del primer import.

**Corrección realizada:** el bloque se clasificó como continuación, se añadió el output completo `registrando reglas / True / True` y se explicó que el segundo import reutiliza el objeto sin repetir el side effect.

### I-02 — Retirar `__init__.py` se presentaba como failure case garantizado

**Ubicación:** sección 14.10.

**Problema:** Python moderno puede tratar el subdirectorio sin `__init__.py` como namespace package. Retirar el archivo incumple el contrato de package regular elegido, pero el import no tiene por qué fallar.

**Impacto:** pedía al estudiante observar una excepción que no está garantizada y confundía package regular con la capacidad general de import de namespace packages.

**Corrección realizada:** la sección distingue ahora esa variación de los failure cases reales, exige registrar el comportamiento observado y prohíbe inventar una excepción.

### I-03 — El starter del mini challenge no era una entrada reproducible

**Ubicación:** sección 20.2.

**Problema:** se describían cuatro archivos y sus defectos, pero no se proporcionaba su contenido. El estudiante debía inventar el circular import y el side effect antes de poder diagnosticarlos.

**Impacto:** incumplía el contrato editorial de entrada reproducible y hacía que dos estudiantes pudieran resolver challenges materialmente distintos.

**Corrección realizada:** se añadieron los cuatro archivos completos (`event_rules.py`, `summary.py`, `eidolon.py`, `check.py`), el comando de reproducción y la secuencia para aislar circular import y side effect.

## 4. Hallazgos MINOR

### M-01 — El diagnóstico de shadowing quedaba ligado a un solo nombre

**Ubicación:** sección 4.5.

**Problema:** `random.py` demostraba correctamente el mecanismo, pero faltaba transferirlo a colisiones frecuentes como `typing.py` y `json.py`.

**Impacto:** el estudiante podía memorizar el ejemplo sin generalizar que el origen debe verificarse para cualquier nombre local que coincida con standard library o third-party code.

**Corrección realizada:** se añadieron ambos contrastes y se explicó que el síntoma puede aparecer indirectamente en una herramienta que importe el módulo ocultado.

### M-02 — La primera recreación del venv adelantaba comandos aún no explicados

**Ubicación:** sección 8.6.

**Problema:** `pip install .` aparecía antes de las secciones de pip y `pyproject.toml` sin indicar cuándo debía ejecutarse.

**Impacto:** creaba un salto pedagógico y podía provocar un fallo ambiental antes de que el estudiante dispusiera del proyecto instalable.

**Corrección realizada:** se marcó la secuencia como objetivo operativo para ejecutar después de las secciones 10 y 14, con referencia a la explicación de pip de la sección 9.

## 5. Evidencia técnica

| Verificación | Resultado |
|---|---|
| Fences Markdown | 135 bloques cerrados |
| Bloques Python | 41/41 compilan sintácticamente |
| Bloques TOML | 6/6 parsean correctamente |
| `pyproject.toml` integrado | metadata, package discovery, extra y script válidos |
| `__name__` y main guard | outputs de import y ejecución coinciden |
| Import cache | módulo ejecutado una vez y reutilizado desde `sys.modules` |
| Circular import del starter | `ImportError` reproducido durante inicialización |
| Circular import corregido | grafo `application → domain`; output `event:evt-001` |
| Module shadowing | origen local confirmado y `AttributeError` reproducido |
| Relative import sin package context | `ImportError` reproducido; `python -m` funciona |
| `ModuleNotFoundError` preinstalación | reproducido desde la raíz del `src` layout |
| Instalación editable | PASS en venv nuevo |
| Instalación no editable | PASS en segundo venv nuevo |
| Smoke check y `pip check` | PASS en ambos environments |
| Entry point `eidolon` | PASS mediante script instalado y `python -m` |
| Ejecución desde otra ubicación | PASS; instalación no editable carga desde `site-packages` |
| Entry point con atributo inexistente | `AttributeError` reproducido |
| `__init__.py` retirado en subpackage | import puede continuar como namespace package; texto corregido |
| Links, anchors, numeración y archivos locales | PASS |

El proyecto de referencia no tiene runtime dependencies externas. La instalación obtiene únicamente el build backend declarado. El módulo distingue build dependency, runtime dependency y development dependency, y no presenta `pip freeze` como sustituto de intención, resolución o lock.

## 6. Gate pedagógico y curricular

- El capítulo parte del crecimiento de `eidolon.py` y explica cohesión, responsabilidades y namespaces antes del tooling.
- Los imports se modelan como bindings y flechas de dependencia; los ciclos se corrigen por ownership, no mediante imports locales arbitrarios.
- Módulo, package y distribución instalable se distinguen por namespace, agrupación y metadata de instalación.
- Venv se introduce desde el problema de estado global y se verifica con executable, prefix y pip del mismo Python.
- Instalación, declaración, resolución/lock y environment instalado permanecen como contratos distintos.
- `pyproject.toml` separa `[build-system]`, `[project]`, extras, scripts y configuración del backend concreto.
- La neutralidad de tooling se conserva: setuptools hace ejecutable el ejemplo, pero Poetry, uv, PDM y otros gestores no se convierten en fundamento.
- Existen 15 bloques distribuidos de cada tipo: `Predice`, `Explica`, `Detecta el bug`, `Modifica` y `Comprueba en terminal`.
- El módulo contiene 10 ejercicios guiados, 28 independientes y 28 preguntas conceptuales.
- El mini challenge usa solo funciones, colecciones, módulos, packages, imports, venv, pip y TOML disponibles en PF-M1–PF-M4.
- OOP, dataclasses, type hints, files/JSON, decorators, context managers, async y testing avanzado permanecen diferidos y señalizados.

## 7. Gate EIDOLON y dependencias

- `domain`, `application` y `cli` representan responsabilidades del journal, no una arquitectura universal.
- No se crean microservices, dependency injection frameworks, repositories, FastAPI, databases, Docker ni AI.
- Los datos son sintéticos e in-memory; no se introduce persistencia ni se confunde un índice derivado con source of truth.
- EIDOLON P0 conserva `dependencies = []`; agregar un paquete exige problema, provenance, mantenimiento, constraint, actualización y estrategia de retiro.
- Typosquatting, abandono y transitivos se enseñan como awareness inicial. Threat modeling, SBOM, firmas y response a vulnerabilidades permanecen en D12.
- La evidencia final sigue la secuencia: obtener código → crear environment limpio → instalar → ejecutar checks → usar `eidolon` desde otra ubicación.

## 8. Decisión

No quedan hallazgos CRITICAL ni IMPORTANT abiertos. PF-M4 cubre el resultado curricular, prepara PF-L06 y cumple el estándar editorial sin invadir módulos posteriores.

`PF-M4 GATE: PASS`

`PF-M4 EDITORIAL GATE: PASS`

PF-M4 cambia de `review candidate` a `approved`. PF-M5 puede comenzar como siguiente módulo, pero no se genera durante este gate.
