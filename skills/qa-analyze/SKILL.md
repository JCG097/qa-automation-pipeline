---
name: qa-analyze
description: Fase 1 del pipeline de QA automation. Lee un user story / documento de criterios de aceptación desde pipeline/inbox/ y produce un documento de escenarios de prueba en pipeline/analysis/, marcando cuáles son candidatos a automatizar y detectando si la historia todavía no está lista para eso. Úsala cuando el usuario pida analizar una historia, sacar casos de prueba de un AC, revisar una historia temprana, o invoque /qa-analyze.
---

# qa-analyze

Fase 1 del pipeline de QA automation (`qa-analyze` → `qa-automate` → `qa-validate` → `qa-report`, orquestadas por `qa-pipeline`). **No todas las historias llegan hasta el final** — esta fase decide qué tan lejos tiene sentido llegar.

## Entrada

Un argumento con la ruta o el nombre del archivo en `pipeline/inbox/` (p. ej. `checkout-happy-path.md` o `pipeline/inbox/checkout-happy-path.md`). Si no se pasa nada, listar `pipeline/inbox/*.md` (ignorando archivos que empiecen con `_` o `.`) y pedir cuál procesar, o si hay uno solo, usar ese.

El **slug** de la historia es el nombre de archivo sin extensión, en kebab-case (si el archivo no está en kebab-case, normalizarlo para nombrar los artefactos de salida, pero no renombrar el original en inbox/).

Si el usuario pide explícitamente "solo análisis" / "no automatices todavía" / "quiero dudas sobre esta historia", respetarlo aunque el contenido de la historia diera para más — eso fija `readiness: analysis-only` sin importar lo demás.

## Bootstrap (primera vez en un producto nuevo)

Si `pipeline/inbox/` no existe en el directorio donde se invoca esto, es un producto/repo nuevo que todavía no tiene el pipeline montado. Antes de seguir, crear:

```
pipeline/{inbox,analysis,automation,validation,reports,processed}/
shared/BasePage.ts   (ver plantilla en qa-automate/SKILL.md — solo si tampoco existe)
```

Avisar al usuario que se acaba de inicializar el pipeline en este repo, y seguir con el resto del flujo normalmente (si además pidió analizar un archivo, seguir con ese archivo; si no, preguntar cuál).

## Qué hacer

1. Leer el archivo completo de `pipeline/inbox/<slug>.md`.
2. Extraer:
   - **Título** de la historia.
   - **target_app**: si el documento menciona una URL, dominio o nombre de aplicación explícito, usarlo tal cual. Si NO lo menciona, dejar `target_app: TBD`.
   - Todos los **criterios de aceptación**, explícitos e implícitos.
3. Revisar la calidad de la historia **antes** de forzar una tabla de escenarios. Buscar activamente:
   - Criterios ambiguos, contradictorios o sin resultado esperado claro.
   - Casos negativos/edge obvios que la historia no menciona (¿qué pasa si falla?, ¿límites de datos?, ¿permisos?).
   - Términos de negocio sin definir, dependencias no aclaradas (¿integra con otro sistema? ¿hay un diseño/mock de referencia?).
   - Falta total de un sistema/UI real contra el cual probar (historia puramente conceptual, feature todavía no construida).
   Cada hallazgo de este tipo va a la sección **Preguntas abiertas y sugerencias de mejora a la documentación** — esta sección se escribe siempre que haya algo que reportar, incluso si la historia sí alcanza para automatizar.
4. Derivar la lista de **tareas de QA** (qué hay que verificar en total).
5. Construir la tabla de **escenarios de prueba** con el mejor esfuerzo posible dado lo que hay. Para cada uno:
   - `id`: `SC-01`, `SC-02`, ... (correlativo, estable entre re-análisis del mismo archivo si es posible).
   - `type`: `happy-path` | `alternative` | `edge` | `negative`.
   - `title`: descripción breve y accionable.
   - `steps`: pasos a alto nivel (sin selectores ni código todavía).
   - `expected`: resultado esperado.
   - `automate`: Sí/No con justificación de una línea (ver criterio abajo).
6. Determinar `readiness` — esto es lo que decide si el pipeline sigue a `qa-automate` o se detiene acá:
   - **`ready-to-automate`**: `target_app` está definido, los criterios son suficientemente claros, y hay al menos un escenario marcado `automate: Sí`.
   - **`analysis-only`**: la historia está clara y los escenarios están bien definidos, pero automatizar no corresponde todavía (feature en fase muy temprana, ambiente/UI aún no existe, o el usuario pidió explícitamente solo el análisis). El valor de esta fase es la tabla de casos de prueba en sí, para que el equipo la tenga documentada.
   - **`needs-clarification`**: la historia tiene ambigüedades o huecos importantes — la sección de preguntas abiertas es el entregable principal, y los escenarios (si se llegan a escribir) son tentativos. Recomendar explícitamente no avanzar a automatizar hasta resolver las dudas.
   - Nunca fuerces `ready-to-automate` solo porque hay escenarios de tipo happy-path — si `target_app` es `TBD` o hay dudas de fondo sin resolver, no es `ready-to-automate`.
7. Criterio para `automate: Sí/No` en cada escenario (aplica independientemente del `readiness` general — un escenario puede ser un buen candidato aunque la historia entera todavía no esté lista):
   - Sí: happy paths y flujos alternativos estables, repetibles, de valor de regresión.
   - No: juicio visual/subjetivo, depende de datos o sistemas externos no disponibles en pruebas, o el costo de automatizar supera el valor.
8. Escribir el resultado en `pipeline/analysis/<slug>.analysis.md` con exactamente esta estructura:

```markdown
---
slug: <slug>
source_file: pipeline/inbox/<archivo-original>.md
target_app: <URL o "TBD">
readiness: ready-to-automate | analysis-only | needs-clarification
generated: <fecha ISO>
---

# Análisis: <Título de la historia>

## Criterios de aceptación identificados
- ...

## Preguntas abiertas y sugerencias de mejora a la documentación
<Ambigüedades, casos no cubiertos, términos sin definir, sugerencias de reescritura. Si no hay nada que señalar, escribir "Ninguna — el criterio de aceptación es suficientemente claro.">

## Tareas de QA
- ...

## Escenarios de prueba

| ID | Tipo | Descripción | Pasos | Resultado esperado | Automatizar |
|----|------|-------------|-------|---------------------|-------------|
| SC-01 | happy-path | ... | 1. ...<br>2. ... | ... | Sí — <justificación> |
| SC-02 | negative | ... | ... | ... | No — <justificación> |

## Automation Candidates
<Lista exacta de IDs que pasarían a la fase 2 si `readiness` es `ready-to-automate`. Si `readiness` no es `ready-to-automate`, igual completar esta lista (son los candidatos "para cuando corresponda"), pero dejar explícito que el pipeline no los va a tomar todavía.>

## Bloqueantes
<Si target_app quedó en TBD, o `readiness` no es `ready-to-automate`, explicar exactamente qué falta para que lo sea.>
```

9. Terminar la respuesta al usuario con un resumen de 2-3 líneas: `readiness` resultante, cuántos escenarios y candidatos hay, y si hay preguntas abiertas destacar la más importante. Si `readiness` es `needs-clarification`, priorizar mostrar las preguntas por sobre la tabla de escenarios en la respuesta al usuario.

## No hacer en esta fase

- No escribir código, page objects ni specs.
- No tocar `pipeline/inbox/` ni moverlo — eso lo hace `qa-report` al cerrar el ciclo completo.
- No asumir la app objetivo si no está explícita en el documento.
- No marcar `ready-to-automate` para forzar que el pipeline avance — es preferible frenar en `analysis-only`/`needs-clarification` y que un humano decida, a automatizar algo que no correspondía todavía.
