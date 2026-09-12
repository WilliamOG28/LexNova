# Documentación técnica — Automatización Rivas Motors / Bajaj Sureste (n8n)

Documentación profesional del sistema de automatización conversacional (chatbot comercial vía WhatsApp/Chatwoot) construido en **n8n**, que atiende dos marcas/canales sobre la misma infraestructura: **Rivas Motors** y **Bajaj Sureste**.

**Total de workflows documentados: 26** (1 orquestador `MAIN` + 25 sub-workflows).

## Cómo leer esta documentación

Cada workflow tiene su propio archivo en [`workflows/`](workflows/), con una plantilla fija:

1. Metadata (archivo fuente, ID n8n, estado, categoría)
2. Resumen de negocio
3. Trigger
4. Contrato de datos (inputs/outputs)
5. Flujo del proceso (narrativa paso a paso)
6. Diagrama de flujo UML (Mermaid — fiel a las conexiones reales del JSON)
7. Integraciones externas
8. Relación con otros workflows
9. Reglas de negocio y notas técnicas (incluye deuda técnica real cuando existe)

Las convenciones completas de esta documentación (y cómo mantenerla) están en [`/CLAUDE.md`](../../CLAUDE.md) en la raíz del repositorio.

## Índice de workflows

### Orquestador
| Workflow | Archivo |
|---|---|
| 🔶 MAIN · Rivas Motors | [workflows/00-MAIN.md](workflows/00-MAIN.md) |

### Dominio — Hidratación de contexto
| Workflow | Archivo |
|---|---|
| 💦 Lead Hydrator | [workflows/01-lead-hydrator.md](workflows/01-lead-hydrator.md) |
| 📍 Geo Router | [workflows/02-geo-router.md](workflows/02-geo-router.md) |
| 🎟️ Promos Vigentes | [workflows/03-promos-vigentes.md](workflows/03-promos-vigentes.md) |

### Dominio — Tools del Agente Comercial
| Workflow | Archivo |
|---|---|
| 🗃️ Inventory Lookup | [workflows/04-inventory-lookup.md](workflows/04-inventory-lookup.md) |
| 🗂️ Inventory Overview | [workflows/05-inventory-overview.md](workflows/05-inventory-overview.md) |
| 🤔 FAQ Lookup | [workflows/06-faq-lookup.md](workflows/06-faq-lookup.md) |
| 🏦 Calcular Precotización | [workflows/07-calcular-precotizacion.md](workflows/07-calcular-precotizacion.md) |
| 🗓️ Agenda Manager | [workflows/08-agenda-manager.md](workflows/08-agenda-manager.md) |
| 📸 Media Sender | [workflows/09-media-sender.md](workflows/09-media-sender.md) |
| 👨‍💻 Human Handoff | [workflows/10-human-handoff.md](workflows/10-human-handoff.md) |
| 📊 CRM Updater | [workflows/11-crm-updater.md](workflows/11-crm-updater.md) |

### Dominio — Post-proceso
| Workflow | Archivo |
|---|---|
| 🥇 Lead Scorer | [workflows/12-lead-scorer.md](workflows/12-lead-scorer.md) |
| 🪞 CRM Sucursal Mirror | [workflows/13-crm-sucursal-mirror.md](workflows/13-crm-sucursal-mirror.md) |
| 🪞 CRM Region Mirror | [workflows/14-crm-region-mirror.md](workflows/14-crm-region-mirror.md) |
| 👤 Auto-Asignar Asesor | [workflows/15-auto-asignar-asesor.md](workflows/15-auto-asignar-asesor.md) |

### Dominio — Integración externa
| Workflow | Archivo |
|---|---|
| 🏦 Financiera Inbound | [workflows/16-financiera-inbound.md](workflows/16-financiera-inbound.md) |

### Webhooks independientes
| Workflow | Archivo |
|---|---|
| 🏷️ Chatwoot Label Control | [workflows/17-chatwoot-label-control.md](workflows/17-chatwoot-label-control.md) |
| 🔄 Chatwoot Reactivar Bot | [workflows/18-chatwoot-reactivar-bot.md](workflows/18-chatwoot-reactivar-bot.md) |

### Programados (Cron)
| Workflow | Archivo |
|---|---|
| ⏰ CRON · Nurturing 14d (inactivo) | [workflows/19-cron-nurturing-14d.md](workflows/19-cron-nurturing-14d.md) |
| ⏰ CRON · Recordatorio de Cita | [workflows/20-cron-recordatorio-cita.md](workflows/20-cron-recordatorio-cita.md) |
| ⏰ CRON · Resumen Diario (inactivo) | [workflows/21-cron-resumen-diario.md](workflows/21-cron-resumen-diario.md) |
| SYNC · Monday → Airtable Inventario | [workflows/22-sync-monday-airtable.md](workflows/22-sync-monday-airtable.md) |
| SYNC · Asesores → Redis | [workflows/23-sync-asesores-redis.md](workflows/23-sync-asesores-redis.md) |

### Utilidades
| Workflow | Archivo |
|---|---|
| 🧹 Reset Lead (inactivo) | [workflows/24-reset-lead.md](workflows/24-reset-lead.md) |
| ⚠️ Error Handler Global | [workflows/25-error-handler-global.md](workflows/25-error-handler-global.md) |

## Vista de arquitectura complementaria

[05-Diagrama-Archify-Arquitectura.html](05-Diagrama-Archify-Arquitectura.html) — diagrama de arquitectura interactivo (HTML standalone, generado con el skill `archify`), con una vista **simplificada y agrupada por dominio** del sistema completo. Útil como mapa mental de alto nivel; para el detalle nodo-por-nodo de cada workflow, usar los archivos en `workflows/`.

## Deuda técnica detectada (resumen)

Durante el análisis se identificaron los siguientes puntos que ameritan limpieza, documentados con detalle en el archivo del workflow correspondiente:

| Workflow | Hallazgo |
|---|---|
| 🔶 MAIN | Duplicación 1:1 de ~68 nodos por canal (Rivas Motors / Bajaj Sureste); nodos `WhatsApp · Typing` deshabilitados; longitud de system prompt inconsistente entre canales |
| 🏦 Calcular Precotización | Rama completa duplicada y deshabilitada (código muerto) |
| 📸 Media Sender | Rama duplicada no alcanzable desde el trigger (código muerto) |
| 👨‍💻 Human Handoff | Nodos HTTP deshabilitados reemplazados por versiones activas sin limpiar; lógica de asignación duplicada con Auto-Asignar Asesor |
| 👤 Auto-Asignar Asesor | Tres variantes coexistentes del mismo guard/picker de asesor, con un nodo `If` inicial sin nombre descriptivo |
| 🪞 CRM Region Mirror | Asimetría respecto a CRM Sucursal Mirror (falta el paso de completar `sucursal_interes` en el maestro) |

## Aviso de seguridad

Este repositorio está conectado a un remoto de GitHub **público** con auto-sync del entorno. Ver detalle en `/CLAUDE.md`.
