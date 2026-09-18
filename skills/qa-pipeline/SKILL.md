---
name: qa-pipeline
description: Orquestador manual del pipeline de QA automation. Revisa pipeline/inbox/ por historias pendientes, corre qa-analyze, y según qué tan lista esté la historia decide si sigue a automatizar/validar/reportar o se detiene en el análisis (historias tempranas, con dudas, o donde el usuario solo pidió casos de prueba). Si no hay nada pendiente, lo informa y no hace nada. Úsalo cuando el usuario quiera procesar lo que haya en inbox/, o invoque /qa-pipeline.
---

# qa-pipeline

Orquestador de hasta 4 fases: `qa-analyze` → (`qa-automate` → `qa-validate`)? → `qa-report`.

**No todas las historias llegan a automatizarse.** `qa-analyze` decide el `readiness` (`ready-to-automate` | `analysis-only` | `needs-clarification`) y este skill respeta esa decisión — no fuerza el pipeline completo solo porque existe.

Disparo manual (no hay loop/cron todavía) — se corre cada vez que el usuario quiere procesar lo que haya en `pipeline/inbox/`.

## Qué hacer

1. Listar `pipeline/inbox/*.md`, ignorando archivos que empiecen con `_` o `.`.
2. **Si no hay ninguno**: responder que no hay historias pendientes y que se puede volver a correr `/qa-pipeline` cuando llegue una nueva. No hacer nada más.
3. **Si hay uno o más**: si el usuario no especificó cuál, tomar el más antiguo y avisar cuántos más quedan en cola. Si pidió `--all`, procesar todos en orden.
4. Para la historia elegida:
   1. Correr `qa-analyze` sobre el archivo de inbox.
   2. Leer el `readiness` que devolvió:
      - **`ready-to-automate`** → seguir con `qa-automate` sobre el análisis, luego `qa-validate` sobre esa automatización, luego `qa-report`.
      - **`analysis-only`** → **no correr `qa-automate` ni `qa-validate`**. Ir directo a `qa-report`, que debe generar el reporte reflejando que el ciclo se cerró en la fase de análisis (ver nota en `qa-report/SKILL.md` sobre cómo manejar fases ausentes).
      - **`needs-clarification`** → **no correr `qa-automate` ni `qa-validate`**. Mostrarle al usuario la sección de preguntas abiertas del análisis de forma destacada (no enterrada en un resumen), preguntar si quiere resolverlas ahora mismo o dejarlo así. Luego ir a `qa-report` para dejar constancia y cerrar el ciclo — la historia puede reabrirse después con un archivo nuevo en `inbox/` una vez aclarada.
   3. Excepción: si `target_app` quedó en `TBD` dentro de un análisis que de otro modo sería `ready-to-automate`, preguntar al usuario cuál es la app antes de decidir si se puede pasar a automatizar. No adivinar.
5. Al terminar (una historia o todas las pedidas), volver a listar `pipeline/inbox/*.md`. Si quedan pendientes y no se pidió `--all`, decir cuántas quedan. Si no queda ninguna, decirlo explícitamente ("no hay historias pendientes, en espera de las próximas").

## Notas

- Cada fase ya sabe escribir su propio artefacto en `pipeline/<fase>/`; este skill no reescribe esos formatos, solo encadena las invocaciones y aplica la lógica de "¿sigo o freno acá?".
- Si en cualquier fase el usuario necesita intervenir (target_app indefinido, dudas por resolver, una falla real de la app que requiere decisión de negocio), pausar ahí y no seguir de largo.
