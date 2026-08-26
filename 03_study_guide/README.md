# EIDOLON Engineering Study Guide

El Study Guide es el material principal de enseñanza. Su trabajo es convertir cada especificación del Curriculum en una secuencia que permita comprender, predecir, implementar, romper, corregir y justificar el comportamiento estudiado.

> Curriculum = qué aprender.  
> Study Guide = cómo aprenderlo.  
> Labs = demostrarlo.  
> EIDOLON builds = integrarlo.

## Qué contiene un módulo

Cada módulo parte del ID y el alcance definidos por el Curriculum. Cuando un concepto es nuevo, el módulo desarrolla, según corresponda:

1. por qué existe;
2. un modelo mental técnicamente correcto;
3. teoría detallada;
4. sintaxis o mecanismo;
5. un ejemplo mínimo;
6. un ejemplo progresivo;
7. qué ocurre internamente al nivel necesario;
8. errores comunes con código roto;
9. aplicación en EIDOLON;
10. cuándo no utilizar la técnica;
11. ejercicios guiados con solución razonada;
12. ejercicios independientes;
13. preguntas conceptuales;
14. un mini challenge;
15. resumen y checklist;
16. preparación para labs.

No todos los conceptos requieren exactamente la misma longitud o una repetición artificial de los 16 apartados. La progresión debe estar presente y ser verificable, pero el texto se organiza para enseñar con naturalidad.

## Autocontención y fuentes externas

El módulo explica sus fundamentos sin exigir que el estudiante abandone el documento para entenderlos. La documentación oficial, libros, cursos y papers sirven para ampliar o profundizar, no para sustituir la explicación esencial.

Si otro módulo ya enseñó un concepto, se enlaza por ID en vez de copiarlo. Si un concepto todavía no ha sido enseñado, el texto no puede asumirlo como prerequisite oculto.

## Estado editorial

| Módulo | Estado | Nota |
|---|---|---|
| [`PF-M1`](programming_foundations/PF-M1_execution_and_data_model.md) | approved | Patrón editorial aprobado |
| [`PF-M2`](programming_foundations/PF-M2_functions_contracts_scopes.md) | approved | Prerequisite aprobado para PF-M3 |
| [`PF-M3`](programming_foundations/PF-M3_collections_comprehensions_iteration.md) | approved | Gate técnico, pedagógico y editorial aprobado |
| [`PF-M4`](programming_foundations/PF-M4_modules_packages_dependency_management.md) | approved | Gate técnico, pedagógico, curricular y editorial aprobado |
| [`PF-M5`](programming_foundations/PF-M5_oop_dataclasses_type_hints.md) | approved | Gate global acumulativo aprobado |
| [`PF-M6`](programming_foundations/PF-M6_exceptions_files_json_resource_lifecycle.md) | approved | Gate global acumulativo aprobado |
| [`PF-M7`](programming_foundations/PF-M7_decorators_context_managers.md) | approved | Gate global acumulativo aprobado |
| [`PF-M8`](programming_foundations/PF-M8_async_await_tasks_cancellation_backpressure.md) | approved | Gate global acumulativo aprobado |
| [`PF-M9`](programming_foundations/PF-M9_testing_debugging_logging_review.md) | approved | Gate global acumulativo aprobado |

Programming Foundations superó el gate acumulativo. PF-M1–PF-M9 están aprobados y Computer Science Foundations es el track activo.

## Computer Science Foundations

| Módulo | Estado | Nota |
|---|---|---|
| [`CS-M1`](computer_science/CS-M1_complexity_measurement_cost_models.md) | review candidate | Complejidad, medición y modelos de costo; pendiente de auditoría acumulativa del track |
| [`CS-M2`](computer_science/CS-M2_arrays_hashmaps_sets.md) | review candidate | Arrays, hash maps y sets; pendiente de auditoría acumulativa del track |
| [`CS-M3`](computer_science/CS-M3_stacks_queues_linked_structures.md) | review candidate | Stacks, queues y estructuras enlazadas; pendiente de auditoría acumulativa del track |
| [`CS-M4`](computer_science/CS-M4_recursion_searching_sorting.md) | review candidate | Recursión, búsqueda y ordenamiento; pendiente de auditoría acumulativa del track |
| [`CS-M5`](computer_science/CS-M5_trees_heaps_priority_queues.md) | review candidate | Árboles, heaps y priority queues; pendiente de auditoría acumulativa del track |
| [`CS-M6`](computer_science/CS-M6_graphs_state_machines.md) | review candidate | Grafos y máquinas de estado; pendiente de auditoría acumulativa del track |
| [`CS-M7`](computer_science/CS-M7_operating_systems_memory_filesystems.md) | review candidate | Sistemas operativos, memoria y filesystems; pendiente de auditoría acumulativa del track |
| [`CS-M8`](computer_science/CS-M8_processes_threads_concurrency_synchronization.md) | review candidate | Procesos, threads, concurrencia y sincronización; pendiente de auditoría acumulativa del track |
| [`CS-M9`](computer_science/CS-M9_networking_http_transport_foundations.md) | review candidate | Networking, transporte y fundamentos de HTTP; pendiente de auditoría acumulativa del track |
| [`CS-M10`](computer_science/CS-M10_computer_architecture_memory_hierarchy_performance.md) | review candidate | Arquitectura de computadoras para programadores; pendiente de auditoría acumulativa del track |
