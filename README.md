# EIDOLON Engineering Curriculum

**Arquitectura documental:** v0.3.0  
**Curriculum Blueprint:** v0.2  
**Idioma:** español latinoamericano técnico  
**Estado:** Foundation Tracks activos; tracks posteriores pendientes

Este repositorio separa cuatro responsabilidades que antes convivían en un solo documento:

1. **Curriculum Blueprint:** fija la arquitectura curricular, las competencias, dependencias, fases, gates, prioridades, niveles de dominio y decisiones pedagógicas.
2. **Engineering Curriculum:** especifica qué aprender, en qué orden, con qué profundidad y qué evidencia demuestra dominio.
3. **Engineering Study Guide:** enseña paso a paso el contenido especificado por cada módulo del Curriculum.
4. **Labs & Projects:** exige demostrar e integrar lo aprendido mediante trabajo ejecutable.

La relación operativa es:

> Curriculum = qué aprender.  
> Study Guide = cómo aprenderlo.  
> Labs = demostrarlo.  
> EIDOLON builds = integrarlo.

## Fuente de verdad

Markdown (`.md`) es el formato canónico a partir de v0.3.0. DOCX y PDF son artefactos de publicación y solo deben generarse para milestones, releases o entregas finales explícitas.

Las fuentes se interpretan en este orden:

1. [`00_master_prompt/EIDOLON_MASTER_PROMPT.md`](00_master_prompt/EIDOLON_MASTER_PROMPT.md): intención educativa, alcance y criterios generales.
2. [`01_blueprint/EIDOLON_CURRICULUM_BLUEPRINT_v0.2.md`](01_blueprint/EIDOLON_CURRICULUM_BLUEPRINT_v0.2.md): decisiones curriculares corregidas y canónicas.
3. [`02_curriculum/`](02_curriculum/): especificaciones ejecutables de overview y tracks.
4. [`03_study_guide/`](03_study_guide/): enseñanza correspondiente a módulos concretos.

Una discrepancia no se resuelve silenciosamente. Debe registrarse mediante un Architecture Decision Record (ADR) curricular o una nueva versión del documento que gobierna la decisión.

## Regla de carga mínima

Para trabajar en un módulo futuro, carga únicamente:

- el Master Prompt;
- el Blueprint;
- el track correspondiente;
- sus prerequisites directos;
- el módulo actual del Study Guide.

No copies bloques canónicos completos entre archivos. Usa IDs estables y enlaces relativos.

## Estructura actual

```text
eidolon-curriculum/
├── 00_master_prompt/
│   └── EIDOLON_MASTER_PROMPT.md
├── 01_blueprint/
│   ├── assets/
│   └── EIDOLON_CURRICULUM_BLUEPRINT_v0.2.md
├── 02_curriculum/
│   ├── assets/
│   ├── README.md
│   ├── 00_overview.md
│   ├── 01_programming_foundations.md
│   └── 02_computer_science_foundations.md
├── 03_study_guide/
│   ├── README.md
│   ├── programming_foundations/
│   │   └── PF-M1_execution_and_data_model.md
│   └── computer_science/
├── 04_labs/
│   ├── README.md
│   ├── programming_foundations/
│   └── computer_science/
├── 05_projects/
│   └── README.md
├── 06_certification/
│   └── README.md
├── 07_research/
│   └── README.md
├── CHANGELOG.md
└── MIGRATION_REPORT.md
```

Los archivos de Backend Engineering, Database Engineering y tracks posteriores no se crean todavía: su ausencia es deliberada y conserva el gate indicado por el Blueprint.

## Convenciones

- Los IDs canónicos (`Dn.m`, `Pn`, `PF-Mn`, `CS-Mn`, `PF-Lnn`, `CS-Lnn`) no se renumeran.
- Las etiquetas `[MUST]`, `[SHOULD]`, `[NICE]` y `[LATER]` conservan el significado del Curriculum.
- Cada módulo del Study Guide declara su `Curriculum source` y sus prerequisites.
- Los labs permanecen especificados dentro del track hasta que su tamaño justifique extraerlos a `04_labs/`.
- Un módulo no se considera aprobado por tiempo invertido: necesita la evidencia y el criterio de dominio del Curriculum.

# eidolon-curriculum
