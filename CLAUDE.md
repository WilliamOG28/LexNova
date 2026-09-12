# CLAUDE.md

Guía para trabajar en este repositorio con Claude Code.

## Qué es este proyecto

Este repositorio contiene la **automatización conversacional de Rivas Motors / Bajaj Sureste**: un chatbot comercial construido en **n8n** que atiende WhatsApp (vía Chatwoot) para dos marcas/canales, califica leads, agenda citas, calcula precotizaciones de crédito, escala a asesores humanos y mantiene sincronizado un CRM distribuido en Airtable.

No es un proyecto de código tradicional: es un **conjunto de workflows de n8n exportados como JSON**, más su documentación técnica.

## Estructura del repositorio

```
Rivas Motors/
├── Workflow/
│   ├── 🔶 MAIN · Rivas Motors.json        # Orquestador principal (único webhook de entrada)
│   └── Sub-Worflow/                        # 25 sub-workflows invocados por MAIN o entre sí
├── Documentacion/
│   ├── 00-README.md                        # Índice general + arquitectura resumida
│   ├── workflows/                          # Un .md por cada uno de los 26 workflows (MAIN + 25 subs)
│   └── 05-Diagrama-Archify-Arquitectura.html  # Diagrama de arquitectura interactivo (skill archify)
├── Plan de trabajo/
└── Reportes/
```

**Total de workflows: 26** (1 MAIN + 25 sub-workflows). Si vas a documentar o auditar "todos los workflows", verifica siempre el conteo real con `ls "Rivas Motors/Workflow/Sub-Worflow" | wc -l` antes de dar un número — este proyecto ha tenido conteos erróneos en documentación anterior.

## Cómo analizar un workflow de n8n sin reventar el contexto

Los JSON de n8n son grandes (hasta ~400 KB, con nodos `code` de miles de caracteres). **Nunca leas un archivo completo con `Read` para entender su estructura.** En su lugar:

1. Extrae solo nodos + conexiones con Python (`json.load` + iterar `nodes`/`connections`), nunca el `parameters` completo de nodos `code`.
2. Genera el diagrama de flujo (Mermaid) programáticamente a partir de `connections` (source → target por tipo/índice de salida), no a mano — así se garantiza fidelidad 1:1 con el JSON real.
3. Solo entra al detalle de un nodo `code`/`set` puntual cuando necesites entender una regla de negocio específica.

## Convenciones de documentación de workflows

Cada archivo en `Documentacion/workflows/` sigue esta plantilla fija — mantenla al agregar o actualizar un workflow:

1. **Metadata** (archivo fuente, ID n8n, estado activo/inactivo, categoría)
2. **Resumen** — propósito de negocio en 1-2 párrafos
3. **Trigger** — cómo se dispara
4. **Contrato de datos** — inputs esperados / outputs devueltos
5. **Flujo del proceso** — narrativa paso a paso
6. **Diagrama de flujo (Mermaid)** — fiel a los nodos/conexiones reales del JSON, no una simplificación inventada
7. **Integraciones externas** — tabla de sistemas tocados (Airtable, Redis, Slack, Chatwoot, OpenAI, Postgres, Monday.com)
8. **Relación con otros workflows** — quién lo invoca / a quién invoca
9. **Reglas de negocio y notas técnicas** — incluye deuda técnica real si se detecta (ramas duplicadas, nodos deshabilitados, código muerto) — no omitirla por quedar "más limpio"

## Reglas de honestidad para esta documentación

- Todo diagrama de flujo debe derivarse de las conexiones reales del JSON, no de una versión idealizada del proceso.
- Si un workflow tiene ramas duplicadas, nodos deshabilitados o lógica muerta, **documentarlo explícitamente** — es información valiosa para quien mantenga el sistema.
- No inventar nombres de campos, tablas de Airtable o valores de enum que no aparezcan evidenciados en el JSON o en los prompts del agente.

## Aviso de seguridad activo

Este repositorio está conectado a un remoto de GitHub **público** (`WilliamOG28/LexNova`) con **auto-commit/auto-push del entorno** (no controlado por Claude Code). Los JSON de workflows exponen paths de webhooks reales, IDs de bases/tablas de Airtable, canales de Slack y la URL de la instancia de Chatwoot. Antes de agregar credenciales, tokens o cualquier dato de cliente real a un archivo nuevo, ten en cuenta que se publicará automáticamente.

## Herramientas disponibles en este entorno

- **Skill `archify`**: genera diagramas de arquitectura/secuencia/flujo interactivos en HTML. Usar `node bin/archify.mjs validate` y `deliver` desde `~/.claude/skills/archify` antes de dar por bueno un diagrama — no asumir que un JSON de especificación es correcto sin validarlo.
- **MCP `codebase-memory-mcp`**: indexa código fuente vía tree-sitter. **No aporta valor en este repo** porque el contenido son exports JSON de n8n, no código con funciones/clases parseables — no perder tiempo indexándolo para tareas de documentación de workflows.
