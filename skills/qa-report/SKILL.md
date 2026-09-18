---
name: qa-report
description: Fase final del pipeline de QA automation. Junta el análisis (y, si existen, la automatización y validación) de una historia en un resumen ejecutivo, y cierra el ciclo moviendo la historia de pipeline/inbox/ a pipeline/processed/. Sabe reportar también historias que se quedaron solo en análisis (readiness analysis-only o needs-clarification). Úsala cuando el usuario pida el reporte final de una historia, o invoque /qa-report.
---

# qa-report

Fase final del pipeline de QA automation. Cierra el ciclo de una historia — con o sin automatización.

## Entrada

Slug de la historia (o ruta a cualquiera de sus artefactos previos: `pipeline/analysis/<slug>.analysis.md`, y si existen `pipeline/automation/<slug>.automation.md`, `pipeline/validation/<slug>.validation.md`).

## Qué hacer

1. Leer `pipeline/analysis/<slug>.analysis.md`. Es el único artefacto obligatorio — si no existe, avisar y pedir que se corra `qa-analyze` primero, no seguir.
2. Mirar su campo `readiness`:
   - **`ready-to-automate`**: `pipeline/automation/<slug>.automation.md` y `pipeline/validation/<slug>.validation.md` deberían existir. Si falta alguno, avisar cuál falta y no inventar su contenido — pedir que se corra esa fase primero, no cerrar el ciclo todavía.
   - **`analysis-only`** o **`needs-clarification`**: es esperable que `automation`/`validation` NO existan. No lo reportes como un error ni pidas que se generen — el reporte debe reflejar que el ciclo se cerró intencionalmente en la fase de análisis.
3. Escribir `pipeline/reports/<slug>.report.md`. Estructura base (omitir/adaptar las secciones de automatización y validación cuando no aplican, en vez de dejarlas vacías o inventadas):

   ```markdown
   ---
   slug: <slug>
   readiness: <el mismo valor que trae el análisis>
   generated: <fecha ISO>
   ---

   # Reporte: <Título de la historia>

   ## Resumen ejecutivo
   <3-5 líneas. Si es ready-to-automate: qué se pidió, qué se automatizó, el resultado.
   Si es analysis-only: qué se pidió, por qué no se automatizó todavía (ej. "feature en fase temprana, sin ambiente disponible"), y que los casos de prueba quedan documentados para cuando corresponda.
   Si es needs-clarification: qué dudas quedaron abiertas y que el ciclo se cierra a la espera de esas respuestas.>

   ## Preguntas abiertas y sugerencias de mejora a la documentación
   <Copiar/resumir la sección equivalente del análisis. Si el análisis dijo "Ninguna", repetirlo acá.>

   ## Cobertura de automatización
   - Escenarios identificados: <N>
   - Automatizados: <N o "0 — no se automatizó en este ciclo (<motivo>)">
   - No automatizados (manual/exploratorio o pendiente): <N> (<lista de IDs + motivo breve>)

   <Si NO es ready-to-automate, omitir por completo la sección "Resultado de la validación" (no hay nada que correr todavía) en vez de mostrarla vacía.>
   ## Resultado de la validación
   | ID | Test | Estado |
   |----|------|--------|
   ...

   ## Hallazgos
   <Bugs reales encontrados, si los hay. Si no hay ninguno, decir "Ninguno".>

   ## Artefactos
   - Análisis: pipeline/analysis/<slug>.analysis.md
   <Solo si existen:>
   - Automatización: pipeline/automation/<slug>.automation.md
   - Validación: pipeline/validation/<slug>.validation.md
   - Tests: apps/<app-slug>/tests/...

   ## Recomendaciones / seguimiento
   <Si es analysis-only o needs-clarification: qué tiene que pasar para que esta historia pueda pasar a automatizarse (ej. "definir target_app", "resolver las preguntas de la sección anterior", "esperar a que el feature esté deployado"). Si es ready-to-automate: deuda técnica, escenarios a reconsiderar, bugs a reportar formalmente.>
   ```

4. Mover el archivo original de `pipeline/inbox/<archivo>.md` a `pipeline/processed/<archivo>.md` (moverlo, no copiarlo). Esto aplica **también** cuando el ciclo se cerró en `analysis-only`/`needs-clarification` — "procesada" significa "se generó un artefacto para esta historia", no "quedó automatizada". Si más adelante llega la aclaración o el feature está listo, entra como un archivo nuevo a `inbox/` (puede referenciar el reporte anterior).
5. Responder al usuario con el resumen ejecutivo (3-5 líneas) y la ruta al reporte completo. Si `readiness` no era `ready-to-automate`, decir explícitamente qué falta para la próxima vuelta. No pegues tablas completas en el chat si ya están en el archivo — decí dónde está.

## No hacer

- No tratar `analysis-only`/`needs-clarification` como un error o una fase incompleta — es un resultado válido del pipeline.
- No cerrar el ciclo de una historia `ready-to-automate` si la validación reportó fallas sin explicar — el reporte puede documentar fallas y aun así cerrarse, pero nunca sin que quede constancia clara de ellas.
