# qa-automation-pipeline (plugin de Claude Code)

Pipeline de QA automation empaquetado como plugin, para instalarlo una vez y usarlo en cualquier producto/repo sin volver a copiar skills a mano.

## Qué hace

Convierte una user story / documento de criterios de aceptación en tests de Playwright automatizados, validados y reportados — pero **no fuerza automatizar todo**: si una historia es muy temprana, ambigua, o el usuario solo pide el análisis, el pipeline se detiene ahí y deja documentados los casos de prueba y las preguntas abiertas, sin escribir código.

```
inbox/<historia>.md
   │  qa-analyze          → decide readiness: ready-to-automate | analysis-only | needs-clarification
   ▼
analysis/<slug>.analysis.md
   │  qa-automate (solo si ready-to-automate)
   ▼
automation/<slug>.automation.md   +   apps/<app-slug>/{pages,fixtures,test-data,tests}/
   │  qa-validate (solo si hubo automation)
   ▼
validation/<slug>.validation.md
   │  qa-report (siempre — cierra el ciclo pase lo que pase)
   ▼
reports/<slug>.report.md   +   inbox/<historia>.md → processed/<historia>.md
```

Orquestador manual: `qa-pipeline` revisa `pipeline/inbox/` y corre lo que corresponda para la historia más antigua pendiente (o todas, con `--all`). Si no hay nada pendiente, lo dice y no hace nada.

## Instalación (una vez por máquina/equipo)

Este repo es la fuente del plugin. Para instalarlo en Claude Code:

```
/plugin marketplace add C:\Users\Pc\qa-automation-pipeline
/plugin install qa-automation-pipeline
```

(o `/plugin marketplace add <url-del-repo-git>` si se publica en un git remoto compartido, para que el equipo lo instale igual sin la ruta local).

Una vez instalado, las skills quedan disponibles en **cualquier repo** que abras con Claude Code, invocables como `/qa-automation-pipeline:qa-pipeline`, `/qa-automation-pipeline:qa-analyze`, etc.

## Cómo se usa en un producto

El plugin no asume nada sobre el producto — la primera vez que se invoca en un repo nuevo, `qa-analyze`/`qa-automate` bootstrapean ahí mismo la carpeta `pipeline/`, el toolchain de Playwright (`package.json`, `tsconfig.json`, `playwright.config.ts`, `shared/BasePage.ts`) y, por cada app que aparezca en una historia, su carpeta `apps/<app-slug>/`. Ese contenido generado **vive en el repo del producto**, no acá — este repo es solo el proceso.

1. En el repo del producto, escribí la historia en `pipeline/inbox/<nombre>.md` (incluí la URL/app objetivo si la sabés).
2. Corré `/qa-automation-pipeline:qa-pipeline`.
3. Revisá `pipeline/reports/<slug>.report.md`.

## Actualizar el plugin

Cualquier cambio a una skill se edita acá (`skills/<nombre>/SKILL.md`), se sube el commit, y el equipo actualiza con `/plugin update qa-automation-pipeline` (o reinstalando). Los repos de producto no necesitan tocarse — consumen la versión del plugin que tengan instalada.
