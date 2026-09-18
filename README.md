# qa-automation-pipeline (plugin de Claude Code)

Pipeline de QA automation empaquetado como plugin, para instalarlo una vez y usarlo en cualquier producto/repositorio sin volver a copiar skills a mano.

## Qué hace

Convierte una user story o un documento de criterios de aceptación en tests de Playwright automatizados, validados y reportados — pero **no fuerza automatizar todo**: si una historia es muy temprana, ambigua, o el usuario solo pide el análisis, el pipeline se detiene ahí y deja documentados los casos de prueba y las preguntas abiertas, sin escribir código.

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

Orquestador manual: `qa-pipeline` revisa `pipeline/inbox/` y ejecuta lo que corresponda para la historia más antigua pendiente (o todas, con `--all`). Si no hay nada pendiente, lo indica y no hace nada.

## Instalación (una vez por equipo)

Este repositorio funciona como marketplace y como plugin al mismo tiempo. Para instalarlo en Claude Code:

```
/plugin marketplace add JCG097/qa-automation-pipeline
/plugin install qa-automation-pipeline@qa-automation-pipeline
```

En la extensión de VS Code, el equivalente es escribir `/plugins` en el chat y usar el diálogo para agregar el marketplace (`JCG097/qa-automation-pipeline`) e instalar el plugin desde ahí.

Una vez instalado, las skills quedan disponibles en **cualquier repositorio** que se abra con Claude Code, invocables como `/qa-automation-pipeline:qa-pipeline`, `/qa-automation-pipeline:qa-analyze`, etc.

## Cómo se usa en un producto

El plugin no asume nada sobre el producto — la primera vez que se invoca en un repositorio nuevo, `qa-analyze`/`qa-automate` crean ahí mismo la carpeta `pipeline/`, el toolchain de Playwright (`package.json`, `tsconfig.json`, `playwright.config.ts`, `shared/BasePage.ts`) y, por cada app que aparezca en una historia, su carpeta `apps/<app-slug>/`. Ese contenido generado **vive en el repositorio del producto**, no en este — este repositorio contiene únicamente el proceso.

1. En el repositorio del producto, escribir la historia en `pipeline/inbox/<nombre>.md` (incluir la URL o app objetivo si se conoce).
2. Ejecutar `/qa-automation-pipeline:qa-pipeline`.
3. Revisar `pipeline/reports/<slug>.report.md`.

## Actualizar el plugin

Cualquier cambio a una skill se edita aquí (`skills/<nombre>/SKILL.md`), se sube el commit, y el equipo actualiza con `/plugin update qa-automation-pipeline` (o reinstalando). Los repositorios de producto no necesitan modificarse — consumen la versión del plugin que tengan instalada.
