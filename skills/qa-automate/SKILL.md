---
name: qa-automate
description: Fase 2 del pipeline de QA automation. Toma el documento de pipeline/analysis/<slug>.analysis.md, inspecciona la app objetivo real y escribe los Page Objects y specs de Playwright para los escenarios marcados como candidatos a automatizar. Úsala cuando el usuario pida automatizar los casos de un análisis ya hecho, o invoque /qa-automate.
---

# qa-automate

Fase 2 del pipeline de QA automation. Recibe el trabajo de `qa-analyze` y produce tests reales de Playwright. Solo corre si el análisis marcó la historia como lista para eso.

## Entrada

Ruta o slug de un archivo en `pipeline/analysis/<slug>.analysis.md`. Si no se pasa nada, listar los `.analysis.md` disponibles y preguntar cuál.

## Antes de escribir una sola línea de test

1. Leer `pipeline/analysis/<slug>.analysis.md` completo.
2. Revisar `readiness`. Si es `analysis-only` o `needs-clarification`, **detenerse** — esta fase no corre todavía. Avisar al usuario que el análisis marcó esta historia como no lista para automatizar (y por qué) y sugerir `qa-report` para cerrar el ciclo en la fase de análisis, en vez de forzar la automatización.
3. Si `readiness` es `ready-to-automate`: tomar la sección **Automation Candidates** — solo esos IDs se automatizan.
4. Si `target_app` es `TBD`: **detenerse y preguntar al usuario** cuál es la URL/app objetivo. No adivinar, no elegir una app de ejemplo.
5. Derivar el **app-slug** (kebab-case del nombre/dominio de la app, ej. `saucedemo` para `https://www.saucedemo.com`).

## Bootstrap del toolchain (primera vez que se automatiza algo en este repo)

Si no existe `package.json` en la raíz del repo, este es el primer escenario que se automatiza acá — crear antes que nada:

**`package.json`**
```json
{
  "name": "qa-automation",
  "version": "1.0.0",
  "description": "QA automation generado por el pipeline qa-analyze/qa-automate/qa-validate/qa-report.",
  "scripts": {
    "test": "playwright test",
    "test:headed": "playwright test --headed",
    "test:ui": "playwright test --ui",
    "report": "playwright show-report reports/html"
  },
  "devDependencies": {
    "@playwright/test": "^1.63.0",
    "@types/node": "^22.20.3",
    "typescript": "^7.0.2"
  }
}
```

**`tsconfig.json`**
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "node16",
    "moduleResolution": "node16",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "types": ["node"]
  },
  "include": ["**/*.ts"],
  "exclude": ["node_modules"]
}
```

**`playwright.config.ts`**
```ts
import { defineConfig, devices } from '@playwright/test';

// Un project por app onboardeada (agregado acá mismo la primera vez que
// una historia la menciona) — comparten un único toolchain/node_modules.
export default defineConfig({
  timeout: 30_000,
  expect: { timeout: 10_000 },
  fullyParallel: false,
  retries: 1,
  reporter: [
    ['html', { outputFolder: 'reports/html', open: 'never' }],
    ['list'],
  ],
  use: {
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
    trace: 'on-first-retry',
    headless: true,
  },
  projects: [],
  outputDir: 'reports/test-results',
});
```

**`shared/BasePage.ts`**
```ts
import { Page } from '@playwright/test';

export abstract class BasePage {
  constructor(readonly page: Page) {}

  async navigate(path: string): Promise<void> {
    await this.page.goto(path);
  }

  async waitForPageLoad(): Promise<void> {
    await this.page.waitForLoadState('networkidle');
  }
}
```

**`.gitignore`**
```
node_modules/
reports/
playwright-report/
test-results/
```

Después de crear estos archivos, correr `npm install` y `npx playwright install chromium` antes de seguir.

## Onboarding de la app (nueva o existente)

Si `apps/<app-slug>/` no existe, escafoldarlo:
```
apps/<app-slug>/
  pages/
  fixtures/pages.ts   (test.extend con los page objects que se vayan creando)
  test-data/
  tests/
```
Y agregar un project nuevo al array `projects` de `playwright.config.ts` (no duplicar si ya existe uno con ese `name`):
```ts
{
  name: '<app-slug>',
  testDir: './apps/<app-slug>/tests',
  use: { ...devices['Desktop Chrome'], baseURL: '<target_app>' },
}
```
(agregar el import de `devices` si el archivo todavía no lo tiene).

Si `apps/<app-slug>/` ya existe (historia previa sobre la misma app), reutilizar sus page objects existentes en vez de recrearlos — leerlos primero.

## Inspeccionar la app real antes de escribir selectores

**No inventar selectores.** Antes de codificar cada escenario, confirmar la estructura real de la página.

**Dónde guardar lo temporal — esto es estricto, no una sugerencia:**
- **Nunca** crear archivos ni carpetas de sondeo dentro del repositorio del proyecto (nada de `.scratch/`, `scratch/`, `tmp/`, `probe-*.js`/`.txt` sueltos en la raíz o en `apps/`, etc.), sin importar el nombre que se les ocurra ponerles.
- Si el entorno expone un directorio de scratchpad propio (fuera del repo, ej. una ruta de temp de la sesión), usar ese.
- Si no hay uno expuesto, usar el directorio temporal del sistema operativo — nunca el working directory del proyecto: en Node, `require('os').tmpdir()`; en shell, `mktemp -d` (Unix) o `%TEMP%` (Windows).
- Borrar cada archivo de sondeo **inmediatamente después de leer lo que necesitabas de él** — no dejarlo "para el final". Si se generan varios (uno por escenario, por ejemplo), se borra cada uno apenas se usó, no se acumulan.

**Cómo sondear:**
- Si la página es server-rendered (HTML estático), usar `curl` (con la salida en el directorio temporal de arriba) o el tool `Grep`/`Read` sobre el HTML para confirmar ids/clases/atributos reales.
- Si la página es una SPA (React/Vue/etc. — se nota porque el HTML crudo trae solo un `<div id="root">` vacío y un bundle JS), `curl` no sirve. Escribir un script Node desechable (en el directorio temporal de arriba, nunca en el repo) que use `require('@playwright/test').chromium`, navegue con un browser real, ejecute las acciones del flujo, y vuelque `outerHTML`/atributos de los elementos relevantes a un archivo. Como el script vive fuera del proyecto, `require('@playwright/test')` no lo resuelve solo desde ahí — `node`, al ejecutar un archivo, busca `node_modules` a partir de la ubicación del archivo, no del directorio de trabajo. Ejecutarlo así: `NODE_PATH="<ruta-absoluta-al-proyecto>/node_modules" node <script-temporal>` (verificado: sin `NODE_PATH` falla con `Cannot find module '@playwright/test'` aunque se invoque `node` desde la raíz del proyecto). Si una acción dispara ruteo cliente-side (URL cambia sin recarga real), no confiar en `waitForLoadState('networkidle')` para sincronizar — esperar explícitamente el selector o la URL resultante (`page.waitForURL(...)`, `locator.waitFor()`).

**Verificación final antes de cerrar esta fase:** listar el árbol del repo (o `git status` si hay repo git) y confirmar que no quedó ningún archivo de sondeo, script `_probe*`/`probe-*`, ni carpeta `.scratch`/`scratch`/`tmp` dentro del proyecto. Si aparece alguno, borrarlo antes de escribir `pipeline/automation/<slug>.automation.md`.

## Convenciones de código a seguir

- Page Objects extienden `shared/BasePage.ts` (`navigate`, `waitForPageLoad`).
- Un fixture por app en `apps/<app-slug>/fixtures/pages.ts`, siguiendo el patrón `test.extend<...>()`.
- Cada escenario automatizado es **un test**, en un `describe` nombrado `<Feature> - Happy Path` o `<Feature> - Alternative` según el `type` del escenario en el análisis.
- El nombre del test debe permitir rastrear el ID del escenario (ej. comentario `// SC-01` justo antes del `test(...)`, o incluirlo en el título).
- Datos de prueba (usuarios, productos, payloads) van en `apps/<app-slug>/test-data/`, nunca hardcodeados dentro del test si son reutilizables.
- No automatizar pagos reales ni acciones irreversibles contra sistemas de producción de terceros — si un escenario candidato lo requiere, marcarlo como bloqueado acá (no en el análisis) y explicar la limitación.

## Al terminar

1. Confirmar que no quedó ningún archivo/carpeta de sondeo en el repo (ver "Verificación final" arriba). Borrar lo que haya quedado.
2. Verificar que compila: `npx tsc --noEmit`.
3. Escribir `pipeline/automation/<slug>.automation.md`:
   ```markdown
   ---
   slug: <slug>
   analysis_file: pipeline/analysis/<slug>.analysis.md
   app_slug: <app-slug>
   target_app: <URL>
   generated: <fecha ISO>
   ---

   # Automatización: <Título de la historia>

   ## Escenarios automatizados
   | ID | Test | Archivo |
   |----|------|---------|
   | SC-01 | should ... | apps/<app-slug>/tests/<archivo>.spec.ts |

   ## Escenarios NO automatizados (aunque eran candidatos)
   | ID | Motivo |
   |----|--------|

   ## Playwright project
   name: <app-slug>  (comando para correr solo esto: `npx playwright test --project=<app-slug>`)
   ```
4. Resumen de 2-3 líneas al usuario: cuántos escenarios se automatizaron, en qué archivos, y si algo quedó bloqueado.

No corras la suite completa en esta fase más allá de una verificación rápida de sintaxis/compilación — la ejecución real y su validación es trabajo de `qa-validate`.
