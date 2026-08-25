# Changelog

Todas las modificaciones relevantes de la arquitectura documental se registran aquí.

## v0.3.3 — 2026-08-25

### Cambió

- `PF-M4_modules_packages_dependency_management.md` supera el gate técnico, pedagógico, curricular y editorial.
- El estado de PF-M4 cambia de `review candidate` a `approved`.
- El ejemplo de import cache se clasifica como continuación y muestra su output completo, incluido el único side effect de import.
- La retirada de `__init__.py` se corrige como variación que puede producir un namespace package, no como excepción garantizada.
- El mini challenge incorpora el starter reproducible de cuatro archivos y conserva separados el circular import y el side effect de import.
- Se amplía el diagnóstico de module shadowing a nombres locales como `typing.py` y `json.py`.
- Se explicita que la recreación del environment se ejecuta después de disponer de `pyproject.toml` y del proyecto integrado.
- Se crea `PF-M4_REVIEW_REPORT.md` con hallazgos, correcciones y evidencia ejecutable del gate.
- Los índices del Study Guide registran PF-M4 como `approved` y habilitan el inicio posterior de PF-M5.

### No cambió

- PF-M4 conserva su ID, competencias, fase P0, build, prerequisites, alcance curricular y profundidad objetivo.
- El proyecto de referencia conserva runtime dependencies vacías y tooling neutral.
- No se cambian decisiones del Master Prompt, Blueprint o Engineering Curriculum.
- No se genera ni desarrolla PF-M5.

### Gate

`PF-M4 GATE: PASS`

`PF-M4 EDITORIAL GATE: PASS`

## v0.3.2 — 2026-08-25

### Cambió

- `PF-M3_collections_comprehensions_iteration.md` supera el gate técnico, pedagógico, de alcance, EIDOLON y editorial.
- El estado de PF-M3 cambia de `review candidate` a `approved`.
- El contrato del mini challenge valida que cada tag sea un string no vacío antes de construir sets derivados.
- El snapshot del mini challenge separa los dicts exteriores y sus lists de tags para detectar mutación anidada.
- Se precisan el contrato de hashability, la clasificación de fragmentos y la evidencia de los failure cases.
- Se crea `PF-M3_REVIEW_REPORT.md` con hallazgos, resoluciones y evidencia ejecutable del gate.
- El índice del Study Guide registra PF-M3 como `approved` y habilita el inicio posterior de PF-M4.

### No cambió

- PF-M3 conserva su ID, competencias, fase P0, build, prerequisites, alcance curricular y profundidad objetivo.
- No se cambian decisiones del Master Prompt, Blueprint o Engineering Curriculum.
- No se genera ni desarrolla PF-M4.

### Gate

`PF-M3 EDITORIAL GATE: PASS`

## v0.3.1 — 2026-08-25

### Cambió

- `PF-M1_execution_and_data_model.md` supera una revisión técnica, pedagógica, de alcance, EIDOLON y editorial.
- El estado de PF-M1 cambia de `review candidate` a `approved`.
- Se corrige el caso integrado para conservar `valid_time_utc` sin derivar incorrectamente `recorded_at` desde el tiempo válido.
- Se precisan aliasing, comparaciones con `is`, awareness temporal, disponibilidad de datos IANA y los límites de exactitud de `Decimal`.
- Se incorporan convenciones de código y checkpoints de práctica distribuida.
- Se crea `PF-M1_REVIEW_REPORT.md` con hallazgos, resoluciones y evidencia del gate.
- Se crea `STUDY_GUIDE_EDITORIAL_STANDARD_v0.1.md` como estándar obligatorio para futuros módulos.

### No cambió

- PF-M1 conserva su ID, competencias, fase P0, build, prerequisites, alcance curricular y profundidad objetivo.
- No se cambian decisiones del Master Prompt, Blueprint o Engineering Curriculum.
- No se genera ni desarrolla PF-M2.

### Gate

`PF-M1 EDITORIAL GATE: PASS`

## v0.3.0 — 2026-08-25

### Cambió

- Markdown (`.md`) pasa a ser la fuente de verdad documental.
- El monolito `EIDOLON_ENGINEERING_CURRICULUM_v0.2.docx` se distribuye en overview y tracks desacoplados.
- Se distingue formalmente entre Curriculum Blueprint, Engineering Curriculum, Engineering Study Guide y Labs & Projects.
- Se agrega metadata breve y trazable a los tracks y módulos.
- Se crea el primer módulo educativo: `PF-M1_execution_and_data_model.md`.
- Se documenta una regla de carga mínima para reducir duplicación y consumo de contexto.

### No cambió

- El Curriculum Blueprint conserva su versión v0.2.
- No se renumeraron competencias, fases P0–P11, gates, módulos, laboratorios, mini proyectos ni builds.
- No se modificaron prioridades técnicas, niveles de dominio, estimaciones de esfuerzo ni criterios de aprobación de v0.2.
- No se desarrollaron Backend Engineering, Database Engineering ni tracks posteriores.

### Motivo

El Curriculum existente funciona como una especificación rigurosa de aprendizaje, pero no como material autocontenido de enseñanza. Separar ambos artefactos permite editar y versionar módulos pequeños, mantener trazabilidad por IDs y desarrollar explicaciones profundas sin inflar o duplicar la especificación.
