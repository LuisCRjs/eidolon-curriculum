# EIDOLON Engineering Study Guide — Estándar editorial v0.1

**Artifact:** Engineering Study Guide editorial standard  
**Version:** v0.1  
**Applies to:** todos los módulos del Study Guide  
**Authority:** deriva del Curriculum y de la revisión del primer módulo aprobado  
**Status:** active

## 1. Propósito

Este estándar define cómo convertir una especificación curricular en material principal de estudio.

> **Curriculum:** qué aprender.  
> **Study Guide:** cómo aprenderlo.  
> **Labs:** demostrarlo.  
> **EIDOLON builds:** integrarlo.

Un módulo del Study Guide no es un syllabus ampliado, una lista de definiciones ni documentación de API. Debe permitir que un estudiante con los prerequisites declarados construya un modelo mental correcto, observe el comportamiento, cometa errores controlados, practique y demuestre dominio.

El estándar no autoriza cambiar el Curriculum. Si la enseñanza revela una ambigüedad o un error en la especificación, se documenta y se escala; no se reinterpreta silenciosamente.

## 2. Jerarquía y trazabilidad

Cada módulo se deriva de un único ID de módulo del Curriculum y respeta:

- competencias asociadas;
- fase y gate;
- nivel objetivo;
- prerequisites;
- prioridad `[MUST]`, `[SHOULD]`, `[NICE]` o `[LATER]`;
- criterios de dominio;
- labs, mini proyectos y builds relacionados;
- profundidad matemática declarada.

El Study Guide puede explicar, ordenar localmente y añadir práctica. No puede renumerar IDs, cambiar prioridades, introducir un gate nuevo o presentar como requisito algo que el Curriculum reserva para después.

### 2.1 Metadata obligatoria

Todo módulo comienza con metadata breve:

```markdown
# <MODULE-ID> — <Título>

**Track:** <track>  
**Competencias:** <IDs>  
**Fase:** <P0–P11>  
**Nivel objetivo:** <nivel para el alcance>  
**Prerequisites:** <IDs o ninguno>  
**Build:** <build relacionado>  
**Curriculum source:** [<MODULE-ID>](<enlace relativo al heading canónico>)  
**Status:** draft | review candidate | approved | deprecated
```

Los prerequisites se referencian por ID. No se copian sus explicaciones dentro del módulo actual.

### 2.2 Estados

- `draft`: desarrollo incompleto; no debe usarse como patrón.
- `review candidate`: contenido completo pendiente de auditoría técnica, pedagógica y editorial.
- `approved`: pasó el gate editorial y sus ejemplos fueron verificados.
- `deprecated`: conserva trazabilidad, pero fue sustituido; debe enlazar al reemplazo.

Solo un módulo `approved` puede servir como patrón de producción.

## 3. Contrato de alcance

Antes de redactar, el autor crea una nota de alcance de trabajo —no tiene que publicarse— con cuatro listas:

1. conceptos que el Curriculum exige enseñar;
2. demostraciones que el estudiante debe producir;
3. prerequisites disponibles;
4. temas vecinos que deben diferirse.

Cada concepto incluido debe cumplir al menos una de estas condiciones:

- aparece en el módulo del Curriculum;
- es un paso estrictamente necesario para explicar uno de sus objetivos;
- es una mención de contexto claramente marcada como futura.

Una API o sintaxis futura puede usarse como instrumento si evita complejidad y recibe una explicación mínima local. Debe indicarse dónde se estudiará formalmente. Su dominio no puede convertirse en criterio de aprobación del módulo actual.

### 3.1 Profundidad

La profundidad se decide por la competencia demostrable, no por todo lo que conoce el autor.

Incluye:

- teoría suficiente para predecir comportamiento;
- detalles internos que cambien decisiones o permitan diagnosticar errores;
- tradeoffs que el estudiante deba justificar;
- vocabulario profesional necesario para consultar documentación y comunicarse.

Excluye:

- internals de una implementación que no afecten el contrato enseñado;
- historia extensa sin función pedagógica;
- variantes avanzadas que pertenecen a otro módulo;
- arquitectura de producción antes de contar con sus prerequisites;
- exhaustividad enciclopédica de APIs.

Cuando un detalle sea dependiente de implementación, plataforma, versión o datos ambientales, debe decirse explícitamente.

## 4. Arquitectura pedagógica obligatoria

Cada concepto nuevo sigue esta progresión. Se pueden agrupar subsecciones cercanas para evitar encabezados artificiales, pero ninguna función pedagógica puede desaparecer.

### 4.1 Por qué existe

Presenta un problema concreto que el concepto resuelve. El estudiante debe poder entender el costo del error antes de conocer la definición.

### 4.2 Modelo mental

Construye una intuición simple y técnicamente correcta. Declara los límites de cualquier analogía; ninguna metáfora debe sustituir el comportamiento real.

### 4.3 Teoría

Explica relaciones y consecuencias, no solo términos. Debe permitir responder “¿por qué ocurre?” y predecir un caso nuevo.

### 4.4 Sintaxis o mecanismo

Muestra cómo se expresa el concepto en el lenguaje o herramienta. Toda sintaxis que no figure en los prerequisites recibe una explicación mínima o una referencia futura.

### 4.5 Ejemplo mínimo

Aísla una sola idea. Su output relevante se muestra o se comprueba con un `assert`.

### 4.6 Ejemplo progresivo

Añade una decisión, una frontera o una interacción realista. No introduce simultáneamente varias abstracciones nuevas.

### 4.7 Qué ocurre internamente

Se incluye cuando mejora la capacidad de predecir, depurar o elegir. Se detiene en el nivel requerido por el Curriculum.

### 4.8 Errores comunes

Muestra código o razonamiento incorrecto, el síntoma observable, la causa y la corrección. “No hagas esto” sin explicación no es suficiente.

### 4.9 Aplicación en EIDOLON

Conecta el concepto con un problema auténtico del proyecto usando el modelo más pequeño que funcione. Debe respetar las invariantes vigentes sin adelantar arquitectura futura.

### 4.10 Cuándo no utilizarlo

Explica tradeoffs y alternativas. Una técnica no debe presentarse como universal solo porque sea el tema de la sección.

### 4.11 Práctica inmediata

Después de cada bloque conceptual importante se incluye una predicción, una modificación breve o una referencia al ejercicio guiado correspondiente. En capítulos largos, la primera práctica no puede posponerse hasta el final.

## 5. Orden obligatorio del módulo

El módulo completo usa, como mínimo, esta secuencia:

1. título y metadata;
2. propósito y límites de prerequisites;
3. resultados de aprendizaje observables;
4. instrucciones de estudio y convenciones de código;
5. desarrollo conceptual con la progresión de la sección 4;
6. caso progresivo integrado, cuando el Curriculum lo justifique;
7. ejercicios guiados con solución razonada;
8. ejercicios independientes;
9. preguntas conceptuales;
10. mini challenge;
11. resumen;
12. checklist de dominio;
13. preparación para labs y builds;
14. recursos de ampliación;
15. límite explícito del módulo.

Los números de sección deben ser consecutivos. Los encabezados principales no alternan sin razón entre formas nominales, preguntas y verbos.

## 6. Resultados de aprendizaje

Cada resultado usa un verbo observable: explicar, predecir, implementar, detectar, comparar, justificar, diagnosticar o demostrar.

Evita resultados como “conocer”, “familiarizarse” o “entender” sin evidencia observable.

Todo resultado debe aparecer al menos en uno de estos lugares:

- ejemplo con predicción;
- ejercicio guiado;
- ejercicio independiente;
- pregunta conceptual;
- mini challenge;
- checklist.

No es necesario repetir el mismo resultado en todos.

## 7. Código

### 7.1 Reglas generales

El código debe:

- usar Python moderno compatible con la versión declarada por el proyecto;
- evitar dependencias externas si la biblioteca estándar basta;
- ser ejecutable en el contexto que declara;
- usar nombres de dominio comprensibles;
- mantener los ejemplos pequeños antes de integrarlos;
- incluir outputs o asserts cuando el comportamiento no sea evidente;
- evitar datos sensibles y efectos externos innecesarios;
- distinguir un comportamiento garantizado de uno dependiente de implementación.

No se publica código que solo “parece correcto”. Todo bloque se compila y los ejemplos completos se ejecutan durante la revisión.

### 7.2 Convenciones visibles

Cada bloque pertenece a una de estas categorías y el texto debe hacerlo inequívoco:

- **Ejemplo ejecutable:** programa o snippet autónomo.
- **Continuación:** depende del bloque inmediatamente anterior y lo declara.
- **Código incorrecto:** falla o expresa un antipatrón de manera deliberada.
- **Failure case:** provoca una excepción o resultado incorrecto específico y declara cuál.
- **Fragmento:** usa nombres genéricos, `...` o contexto omitido; no se ofrece como programa completo.
- **Solución parcial:** deja decisiones al estudiante y declara qué falta.

Un bloque no debe depender de nombres creados varias páginas antes. Si la repetición mínima vuelve el ejemplo autónomo, se prefiere esa repetición.

### 7.3 Outputs

Muestra el output cuando confirma el modelo mental. No fijes literalmente:

- direcciones o identidades de memoria;
- rutas temporales;
- mensajes completos que varían por plataforma o versión;
- orden no garantizado por el contrato;
- información dependiente del reloj o de la zona local.

En esos casos describe la propiedad estable o compruébala con un `assert`.

### 7.4 Errores deliberados

Un error pedagógico incluye:

1. etiqueta clara;
2. predicción solicitada;
3. síntoma esperado;
4. explicación causal;
5. corrección o política adecuada;
6. indicación de los datos que deben conservarse para diagnosticar.

No uses un error cuyo comportamiento dependa de una optimización no garantizada, salvo que precisamente se enseñe esa falta de garantía.

### 7.5 Precondiciones ambientales

Si un ejemplo depende de versión, sistema operativo, locale, zona horaria, servicio o dataset:

- declara la precondición junto al primer uso;
- ofrece una comprobación mínima;
- explica el fallo esperable cuando falta;
- no sustituye silenciosamente la semántica por un fallback incorrecto.

## 8. Ejemplos

La progresión recomendada es:

1. un caso mínimo que aísla la regla;
2. un contraste que rompe una intuición común;
3. un caso progresivo con una decisión de diseño;
4. una aplicación EIDOLON;
5. una variación que el estudiante debe predecir.

Los ejemplos no deben reutilizar siempre el mismo dato si eso permite memorizar el output. Cambia valores, formas de entrada y condiciones sin introducir otro concepto.

Un ejemplo EIDOLON enseña el concepto; no es una excusa para añadir clases, servicios, persistencia o AI antes de tiempo.

## 9. Integración EIDOLON

Toda integración respeta, cuando apliquen, estas invariantes:

- los acontecimientos originales son inmutables; las interpretaciones pueden cambiar;
- source data y derived data tienen nombres y roles distintos;
- una transformación no se presenta como fuente;
- la provenance mínima necesaria se conserva o se señala como requisito posterior;
- identidad de dominio no se confunde con identidad del runtime;
- tiempo válido, tiempo de registro y precisión de la fuente no se colapsan;
- una inferencia no se convierte en hecho por cambiar su tipo o representación;
- los errores no producen pérdida silenciosa de evidencia.

Si aplicar una invariante completa exige contenido posterior, el módulo enseña solo la frontera actual y enlaza el ID futuro. No inventa una arquitectura provisional incompatible.

## 10. Errores comunes

La sección de errores no es una lista de prohibiciones. Cada error significativo debe responder:

- ¿qué escribió o pensó el estudiante?;
- ¿qué observó?;
- ¿por qué ocurrió?;
- ¿qué regla permite predecirlo?;
- ¿cuál es la corrección?;
- ¿cuándo podría ser válida una variante parecida?

Prioriza errores plausibles para los prerequisites declarados. No colecciones rarezas de entrevistas o casos de implementación sin impacto en el dominio.

## 11. Ejercicios guiados

Cada módulo incluye ejercicios guiados suficientes para cubrir sus modelos mentales principales antes del mini challenge.

Un ejercicio guiado contiene:

1. objetivo observable;
2. input o código inicial;
3. instrucción de predecir o explicar antes de ejecutar;
4. solución ejecutable;
5. razonamiento paso a paso;
6. criterio que distingue una solución correcta de una coincidencia accidental;
7. una variación opcional cuando ayude a transferir el concepto.

La solución no introduce una abstracción nueva solo para ser elegante. Si existen dos soluciones válidas con contratos distintos, se explican los tradeoffs.

Los checkpoints distribuidos pueden enlazar a estos ejercicios para evitar duplicar soluciones.

## 12. Ejercicios independientes

Los ejercicios independientes no incluyen la solución inmediata. Deben mezclar:

- predicción;
- implementación;
- depuración;
- comparación de alternativas;
- explicación escrita;
- transferencia a un dato diferente.

La dificultad progresa. Los primeros ejercicios aíslan una idea; los últimos combinan dos o tres ideas ya enseñadas. Ninguno requiere contenido no incluido en prerequisites o en el módulo.

La cantidad se decide por cobertura, no por una cuota fija. Todo concepto de riesgo alto necesita al menos una práctica independiente o un failure case reproducible.

## 13. Preguntas conceptuales

Las preguntas verifican modelos mentales, no memoria literal. Deben poder responderse sin ejecutar; después puede pedirse un experimento mínimo para comprobar la respuesta.

Incluye preguntas de:

- distinción entre conceptos cercanos;
- causalidad;
- predicción;
- elección bajo un contrato;
- límites y tradeoffs;
- aplicación de invariantes EIDOLON.

Evita preguntas cuya respuesta sea solo una definición copiada del encabezado.

## 14. Mini challenge

El mini challenge integra únicamente conceptos ya enseñados. Contiene:

- objetivo y artefacto esperado;
- entrada reproducible;
- requisitos numerados;
- contrato de salida;
- comprobaciones ejecutables;
- failure cases deliberados;
- criterio de aprobación;
- límites explícitos sobre lo que no debe implementarse todavía.

Los asserts comprueban propiedades del contrato, no detalles accidentales de implementación. Los failure cases deben pedir diagnóstico y política, no necesariamente una arquitectura completa de recuperación si ese contenido pertenece a otro módulo.

El challenge debe preparar un lab o build identificado. No sustituye el lab.

## 15. Evaluación y evidencia

La evaluación combina:

- **predicción:** anticipar outputs o estados;
- **ejecución:** producir un artefacto que corre;
- **diagnóstico:** explicar un fallo reproducible;
- **decisión:** justificar una alternativa bajo un contrato;
- **transferencia:** aplicar el concepto a un caso no idéntico;
- **comunicación:** explicar el modelo sin leer una definición.

Los criterios de aprobación son observables. Evita medidas ambiguas como “respuesta completa” sin describir qué evidencia cuenta.

## 16. Checklist de dominio

El checklist usa afirmaciones verificables en primera persona:

```markdown
- [ ] Puedo explicar...
- [ ] Puedo predecir...
- [ ] Puedo implementar...
- [ ] Puedo detectar...
- [ ] Puedo justificar...
```

Debe cubrir resultados de aprendizaje, errores de alto riesgo y límites del módulo. No repite cada detalle del índice ni introduce requisitos nuevos.

## 17. Labs y builds

La sección final indica:

- qué labs puede comenzar el estudiante;
- cuál es el lab principal del módulo;
- qué parte de cada lab ya está preparada;
- qué prerequisites faltan para los labs todavía no ejecutables;
- qué aporta el módulo al build EIDOLON asociado;
- qué evidencia debe entregar antes de avanzar.

Usa IDs canónicos. No copies las especificaciones completas del lab o build.

## 18. Recursos

El módulo es autocontenido hasta donde sea razonable. Los recursos externos amplían, verifican o profundizan; no sustituyen la explicación fundamental.

Prioridad de fuentes:

1. documentación oficial y estándares;
2. libros o cursos canónicos definidos por el Curriculum;
3. artículos técnicos primarios o mantenidos por responsables de la tecnología;
4. recursos secundarios solo cuando aporten una explicación pedagógica distintiva.

Cada enlace debe tener un propósito claro. No se incluye una lista larga no curada. Los recursos compartidos del track se referencian por enlace en vez de copiarse.

Durante el gate editorial se verifican enlaces locales, anchors internos y URLs críticas.

## 19. Lenguaje y terminología

Todos los documentos educativos se escriben en español latinoamericano claro, técnico y natural.

Mantén en inglés:

- keywords, APIs, comandos y código;
- nombres oficiales de tecnologías;
- términos cuyo uso profesional habitual sea inglés.

En la primera aparición de un concepto importante, usa cuando ayude:

> explicación en español (technical term in English)

Después conserva una forma consistente. No alternes sin motivo entre traducciones diferentes.

Reglas de estilo:

- dirigirse al estudiante con claridad, sin tono condescendiente;
- preferir verbos y ejemplos concretos a sustantivos abstractos;
- definir un término antes de usarlo para razonar;
- evitar anglicismos cuando exista una forma española natural, pero enseñar el término de búsqueda profesional;
- no usar “obviamente”, “simplemente” o “trivial” para ocultar un paso;
- no atribuir intención humana a una herramienta cuando una explicación causal sea más precisa;
- separar hechos garantizados, recomendaciones e invariantes del proyecto.

## 20. Edición y no duplicación

Antes de repetir contenido, pregunta si basta con un enlace por ID. Repite solo el contexto mínimo necesario para que el módulo sea estudiable con sus prerequisites directos.

Se elimina o combina contenido cuando:

- expresa la misma regla sin añadir ejemplo, contraste o consecuencia;
- contradice una formulación anterior;
- pertenece por completo a otro módulo;
- repite metadata canónica que puede referenciarse.

No se reduce una explicación útil solo para acortar el archivo. La medida es densidad pedagógica, no longitud.

## 21. Validación técnica obligatoria

Antes de cambiar el estado a `approved`:

1. extraer todos los bloques de código del lenguaje principal;
2. compilar todos los bloques sintácticamente completos;
3. ejecutar cada ejemplo autónomo;
4. ejecutar las secuencias declaradas en orden;
5. comprobar outputs estables;
6. provocar los errores deliberados y verificar su tipo o propiedad;
7. ejecutar una solución de referencia del mini challenge;
8. ejecutar sus asserts y failure cases;
9. revisar afirmaciones sensibles contra fuentes primarias;
10. comprobar enlaces y anchors locales.

El reporte distingue:

- error accidental;
- fallo deliberado;
- fragmento no autónomo;
- dependencia ambiental;
- comportamiento no garantizado.

## 22. Gate editorial

Un módulo obtiene `PASS` solo si:

- no quedan hallazgos `CRITICAL` ni `IMPORTANT` abiertos;
- los `MINOR` pendientes están documentados y no impiden estudiar;
- cumple todo el alcance `[MUST]` y la profundidad declarada;
- no invade un módulo posterior sin señalización;
- un estudiante con los prerequisites puede seguir la progresión;
- los ejemplos y comprobaciones ejecutables pasan;
- las integraciones respetan las invariantes EIDOLON;
- metadata, IDs, enlaces y terminología son consistentes;
- el checklist y el challenge evalúan lo enseñado;
- el reporte de revisión registra qué cambió y qué se preservó.

La decisión se expresa exactamente como:

```text
<MODULE-ID> EDITORIAL GATE: PASS
```

o:

```text
<MODULE-ID> EDITORIAL GATE: FAIL
```

Un `FAIL` enumera los defectos bloqueantes y mantiene el estado `review candidate`. Un `PASS` cambia el estado a `approved` y se registra en `CHANGELOG.md`.

## 23. Definition of done para un módulo nuevo

- [ ] Metadata y trazabilidad correctas.
- [ ] Alcance comparado con el Curriculum y prerequisites.
- [ ] Resultados observables cubiertos por evidencia.
- [ ] Cada concepto pasa por problema, intuición, teoría, mecanismo y práctica.
- [ ] Código clasificado, compilado y ejecutado según su contrato.
- [ ] Outputs y failure cases verificados.
- [ ] Práctica distribuida antes del mini challenge.
- [ ] Ejercicios guiados con solución razonada.
- [ ] Ejercicios independientes sin solución inmediata.
- [ ] Preguntas conceptuales de causalidad y decisión.
- [ ] Mini challenge reproducible y alineado con un lab o build.
- [ ] Invariantes EIDOLON respetadas.
- [ ] Recursos curados y enlaces comprobados.
- [ ] Límites hacia módulos futuros explícitos.
- [ ] Auditoría técnica, pedagógica, de alcance y editorial completada.
- [ ] Gate registrado y `Status` coherente.
