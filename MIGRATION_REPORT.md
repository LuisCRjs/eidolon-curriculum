# Migration Report — v0.2 DOCX a arquitectura Markdown v0.3.0

**Fecha:** 2026-08-25  
**Tipo de cambio:** migración estructural y nuevo contrato editorial  
**Resultado:** contenido curricular preservado; primer Study Guide añadido como artefacto separado

## Fuentes canónicas utilizadas

- `EIDOLON_MASTER_PROMPT.md`
- `EIDOLON_CURRICULUM_BLUEPRINT_v0.2.docx`
- `EIDOLON_ENGINEERING_CURRICULUM_v0.2.docx`

## Mapeo de migración

| Origen | Destino | Tratamiento |
|---|---|---|
| Master Prompt Markdown | `00_master_prompt/EIDOLON_MASTER_PROMPT.md` | Copia byte a byte |
| Blueprint v0.2 DOCX | `01_blueprint/EIDOLON_CURRICULUM_BLUEPRINT_v0.2.md` | Conversión estructural a GFM; tabla e imagen preservadas |
| Engineering Curriculum v0.2, inicio–Parte I y puente de Foundation Tracks | `02_curriculum/00_overview.md` | Conversión y corte por encabezado |
| Programming Foundations | `02_curriculum/01_programming_foundations.md` | Bloque completo desde `Programming Foundations` hasta antes de `Computer Science Foundations` |
| Computer Science Foundations | `02_curriculum/02_computer_science_foundations.md` | Bloque completo hasta el final de v0.2 |
| Especificación `PF-M1` | `03_study_guide/programming_foundations/PF-M1_execution_and_data_model.md` | Desarrollo educativo nuevo; no reemplaza la especificación |

## Cambios deliberados

- Se propuso **v0.3.0** para la arquitectura documental.
- Se agregaron README, metadata de trazabilidad, changelog e índices de directorio.
- Las rutas de imágenes se hicieron relativas al repositorio.
- Las expresiones visuales propias de Word se representaron como Markdown equivalente; el contenido textual no se reescribió.

## Elementos que no cambiaron

- 51 competencias `D1.1`–`D17.3`.
- Fases `P0`–`P11` y sus builds.
- Gates conceptuales y de fase.
- Prioridades `LEARN NOW`, `LEARN SOON`, `LEARN LATER`, `ONLY IF NEEDED` y `DO NOT PRIORITIZE`.
- IDs `PF-M1`–`PF-M9`, `CS-M1`–`CS-M10`, labs, mini proyectos y checkpoints.
- Recursos y estimaciones Minimum, Recommended y Deep mastery.

## Límites de esta migración

- No se generaron DOCX ni PDF.
- No se desarrollaron Backend Engineering ni Database Engineering.
- No se extrajeron todavía labs o proyectos individuales: sus especificaciones permanecen canónicas dentro de cada track y los directorios nuevos solo las referencian.
- No se avanzó a PF-M2. El único módulo educativo creado es PF-M1.

## Validación ejecutada

Resultado: **aprobado**.

- El Master Prompt migrado coincide byte a byte con la fuente (`SHA-256 26e43691a2e825e75ea0c7f528f2bc49e7ed052c2d36c7054c6edbfdd5774d57`).
- Los tres segmentos del Engineering Curriculum coinciden exactamente con la conversión Markdown completa, salvo metadata añadida y rutas relativas de imágenes.
- Se preservaron 51 competencias, 12 fases, 9 módulos PF, 10 módulos CS, 16 labs PF y 18 labs CS.
- Se compararon también IDs de mini proyectos, checkpoints y builds sin diferencias.
- Las tres imágenes coinciden byte a byte con los assets extraídos de las fuentes.
- Todos los enlaces locales resuelven a archivos existentes.
- Compilaron sintácticamente 97 bloques Python de PF-M1.
- El caso integrado del mini challenge pasó sus assertions con Unicode descompuesto, `Decimal("149.90")`, confidence `0.0` y conversión `America/Merida → UTC`.
- No existen archivos de Backend Engineering ni Database Engineering.

### Hashes de las fuentes DOCX

- Blueprint v0.2: `fac20e669b0852ba52c6dc0d3ab9813ba5b5bf15db65a89a1fd576dff8c3d02e`
- Engineering Curriculum v0.2: `4e2d36627bb3a81ae56135f879d58480f421721f8254394c1d31b2f95e8dfe54`
