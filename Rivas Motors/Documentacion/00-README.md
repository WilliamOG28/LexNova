# Documentación — Automatización Rivas Motors / Bajaj Sureste (n8n)

Este directorio documenta el sistema de automatización conversacional (chatbot comercial vía WhatsApp/Chatwoot) construido en **n8n**, que atiende dos marcas/canales sobre la misma infraestructura: **Rivas Motors** y **Bajaj Sureste**.

## Índice

1. [01-Arquitectura-Clean-Architecture.md](01-Arquitectura-Clean-Architecture.md) — Cómo se mapea el proyecto n8n a capas de Clean Architecture, regla de dependencia, stack de infraestructura.
2. [02-Workflow-MAIN.md](02-Workflow-MAIN.md) — Desglose fase por fase del orquestador principal (`🔶 MAIN · Rivas Motors`), con diagramas de flujo.
3. [03-Catalogo-Subworkflows.md](03-Catalogo-Subworkflows.md) — Catálogo de los 22 sub-workflows: propósito, trigger, inputs/outputs, dependencias.
4. [04-Diagramas-UML.md](04-Diagramas-UML.md) — Diagramas UML consolidados: componentes, secuencia (casos clave), estados del lead, y "clase" de las entidades principales.
5. [05-Diagrama-Archify-Arquitectura.html](05-Diagrama-Archify-Arquitectura.html) — Diagrama de arquitectura interactivo (HTML standalone, generado con el skill `archify`): abrir en el navegador para pan/zoom, tema claro/oscuro y trazado de relaciones. Es una vista simplificada (agrupada por dominio) del mismo sistema que detalla el archivo 01; para el detalle nodo-por-nodo, usar los `.md`.

## Cómo se generó esta documentación

Se extrajo la estructura de los 23 archivos JSON exportados de n8n (`Workflow/🔶 MAIN · Rivas Motors.json` y `Workflow/Sub-Worflow/*.json`) mediante un script que listó nodos, tipos, parámetros clave y conexiones (source → target por tipo de salida), evitando cargar el contenido íntegro de nodos `code` muy extensos. Sobre esa base estructural se reconstruyeron los flujos y se redactó la narrativa de negocio.

## Alcance y limitaciones

- No se ejecutó ningún workflow; el análisis es estático, basado en la definición JSON.
- El contenido completo de los prompts de sistema del agente LLM (~50k–73k caracteres) no se transcribe aquí; se referencia su propósito y las herramientas que expone.
- Nombres de archivos tienen inconsistencias de tipeo en el proyecto original (p. ej. "Bajaja" vs "Bajaj", "Bjaja" vs "Bajaj"); se preservan tal cual para que la documentación sea trazable 1:1 a los archivos reales.
