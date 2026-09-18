---
name: qa-validate
description: Fase 3 del pipeline de QA automation. Corre los tests generados por qa-automate para una historia, diagnostica fallas reales vs. tests mal escritos, corrige lo que corresponda corregir en el test, y deja constancia del resultado en pipeline/validation/. Úsala cuando el usuario pida validar o correr los tests de una historia ya automatizada, o invoque /qa-validate.
---

# qa-validate

Fase 3 de 4 del pipeline de QA automation. Ejecuta de verdad lo que `qa-automate` escribió y confirma que funciona.

## Entrada

Ruta o slug de un archivo en `pipeline/automation/<slug>.automation.md`. Si no se pasa nada, listar los `.automation.md` disponibles y preguntar cuál.

## Qué hacer

1. Leer `pipeline/automation/<slug>.automation.md` para saber `app_slug` y qué archivos de test corresponden a esta historia.
2. Correr **solo esos tests** (no toda la suite del repo, salvo que el usuario pida explícitamente una corrida completa de regresión):
   ```
   npx playwright test --project=<app_slug> <ruta-o-patrón-de-los-specs-de-esta-historia>
   ```
3. Si algo falla, diagnosticar antes de tocar nada:
   - Leer el `error-context.md` / trace que Playwright genera en `reports/test-results/`.
   - Si el fallo es porque el test asume algo incorrecto sobre la app real (selector equivocado, timing, aserción mal planteada) → corregir el test/page object y volver a correr.
   - Si el fallo revela que la app realmente no cumple el criterio de aceptación → **no lo arregles silenciosamente en el test para que pase**. Repórtalo tal cual como una falla real de la app en `pipeline/validation/<slug>.validation.md`, con evidencia (el error, el escenario, el criterio de aceptación que no se cumple).
   - Si el fallo es intermitente/externo (red, timing de un sitio real de terceros) documentarlo como flaky con la evidencia, no como bug de producto ni de test.
4. Reintentar como máximo una vez más tras cada corrección. Si sigue fallando después de eso, se reporta como está — no forzar un verde artificial.
5. Escribir `pipeline/validation/<slug>.validation.md`:
   ```markdown
   ---
   slug: <slug>
   automation_file: pipeline/automation/<slug>.automation.md
   app_slug: <app-slug>
   generated: <fecha ISO>
   ---

   # Validación: <Título de la historia>

   ## Resultado de la corrida
   | ID | Test | Estado | Notas |
   |----|------|--------|-------|
   | SC-01 | should ... | ✅ Pass | |
   | SC-02 | should ... | ❌ Fail | <causa raíz> |
   | SC-03 | should ... | ⚠️ Flaky | <evidencia> |

   ## Fallas reales de la app (no del test)
   <Si las hay, describir con evidencia. Si no hay ninguna, decir "Ninguna".>

   ## Correcciones aplicadas a los tests durante esta fase
   <Qué se corrigió y por qué, si aplica.>
   ```
6. Resumen de 2-3 líneas al usuario: cuántos pasaron, cuántos fallaron y por qué (test mal escrito vs. bug real de la app), cuántos son flaky.

## No hacer

- No modificar el análisis (`pipeline/analysis/`) ni la lista de escenarios — si un escenario resulta imposible de automatizar de verdad, eso se anota aquí como hallazgo, no se reescribe la fase 1.
- No maquillar resultados: si algo no pasa, se reporta como no-pasa.
