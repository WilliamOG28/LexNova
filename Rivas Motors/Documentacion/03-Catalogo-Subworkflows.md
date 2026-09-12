# Catálogo de Sub-Workflows

Todos los archivos viven en `Workflow/Sub-Worflow/`. Se agrupan por dominio funcional. El trigger `n8n-nodes-base.executeWorkflowTrigger` ("When Executed by Another Workflow") indica que el subworkflow **solo se invoca internamente** (desde MAIN u otro subworkflow), no tiene entrada pública propia.

## A. Hidratación de contexto (invocados por mensaje entrante)

### 💦 SUB · Lead Hydrator
- **Trigger**: `executeWorkflowTrigger` — inputs: `user_id_canal`, `channel`, `sender_name`, `conversation_id`, ...
- **Propósito**: obtener (o crear) el lead en Airtable, con cache-aside en Redis (`lead:{user_id_canal}`, TTL implícito por invalidaciones explícitas en otros subworkflows), y anexar la lista de financieras activas (también cacheada, `financieras:activas`).
- **Salida**: objeto lead completo (`lead_existe`, `lead_id`, `cliente_nombre_registrado`, `lead_score`, `etapa_pipeline`, `modelo_interes`, `financiera_elegida`, `sucursal_asignada`, etc.) + `financieras_disponibles`.
- **Patrón**: Cache-Aside — `If · Cache Hit?` intenta Redis primero; si falla, busca/crea en Airtable y repuebla la cache.
- **Dependencias de infraestructura**: Airtable (`Leads`, `Financieras`), Redis.

### 📍🗺️ SUB · Geo Router
- **Trigger**: inputs `message_text`, `user_id_canal`, `ciudad_lead`.
- **Propósito**: resolver a qué sucursal pertenece el lead según ciudad. Primero intenta un **match determinístico por código** (`Code · Geo Match`); si es ambiguo, cae a un **LLM** (`gpt-5.4-mini` + `outputParserStructured`) para inferir la ciudad desde el texto libre del cliente.
- **Patrón**: *fallback en cascada* reglas-primero, LLM-como-respaldo (mismo patrón que FAQ Lookup y Financiera Inbound).
- **Dependencias**: Airtable (`Config`, `Sucursales`, `Leads`), OpenAI.

### 🎟️ SUB · Promos Vigentes
- **Trigger**: input `user_id_canal`.
- **Propósito**: devolver promociones activas, cacheadas en Redis (`promos:__vigentes__`).
- **Dependencias**: Airtable (`Promociones`), Redis.

## B. Herramientas del Agente Comercial (tool-calling desde el LLM)

### 🗃️ SUB · Inventory Lookup
- **Trigger**: inputs `modelo`, `user_id_canal`.
- **Propósito**: dado un modelo de moto, resolver la familia/modelo exacto y sumar stock disponible entre sucursales + precio + cilindrada + categoría. Cache por modelo (`inventario:{modelo}`).
- **Dependencias**: Airtable (`Modelos`, `Inventario`, `Sucursales`), Redis.

### 🗂️ SUB · Inventory Overview
- **Trigger**: input `user_id_canal`.
- **Propósito**: snapshot de disponibilidad de **todos** los modelos de un jalón (para "qué motos manejan"). Cache global (`inventario:__overview__`).
- **Dependencias**: Airtable (`Modelos`, `Inventario`, `Sucursales`), Redis.

### 🤔❓ SUB · FAQ Lookup
- **Trigger**: input `tema`.
- **Propósito**: consulta la tabla `FAQs` en Airtable y arma una respuesta de respaldo cuando la pregunta no está cubierta por el conocimiento del agente. Es el subworkflow más simple del sistema (2 nodos de lógica).
- **Dependencias**: Airtable (`FAQs`).

### 🏦 SUB · Calcular Precotización
- **Trigger**: inputs `user_id_canal`, `modelo`, `enganche`, `plazo_meses`, ...
- **Propósito**: calcular la mensualidad estimada de un crédito para una moto, y marcar en el CRM que el lead fue "enviado a financiera".
- **Detalle notable**: el workflow contiene **dos ramas paralelas equivalentes** de la misma lógica; la primera (nodos sin sufijo) está **completamente deshabilitada** (`[DISABLED]` en todos sus nodos) y solo la segunda (`Financiera1`, `Modelo1`, `Precotización1`, etc.) está activa — probablemente un remanente de una iteración anterior dejado como referencia/rollback.
- **Dependencias**: Airtable (`Financieras`, `Modelos`, `Leads`), Redis (invalidación de cache de lead).

### 🗓️ SUB · Agenda Manager
- **Trigger**: inputs `user_id_canal`, `lead_id`, `tipo_cita`, `fecha`, `hora`, `sucursal_id`, ...
- **Propósito**: agendar una cita (prueba de manejo, visita, etc.) validando disponibilidad.
- **Flujo**: valida datos → resuelve sucursal → chequea conflictos de horario (`Airtable · Check Conflicts` contra tabla `Citas`) → si libre, crea la cita, actualiza la etapa del lead, invalida cache, y notifica por Slack tanto al canal general de la sucursal como (si tiene Slack vinculado) por DM al asesor asignado.
- **Dependencias**: Airtable (`Sucursales`, `Citas`, `Leads`, `Asesores`), Redis, Slack.

### 📸 SUB · Media Sender
- **Trigger**: inputs `modelo`, `conversation_id`, `tipo` (`imagenes` | `ficha` | `promo` | flyer financiera).
- **Propósito**: enviar material multimedia (fotos de modelo, ficha técnica PDF, flyer de promo o de financiera) al cliente vía Chatwoot. Descarga el asset desde Airtable/URL externa y lo reenvía como adjunto.
- **Detalle notable**: igual que Precotización, tiene dos ramas duplicadas de la misma lógica (una sin sufijo, otra con sufijo `1`); a diferencia de Precotización aquí **ambas están activas** — la rama sin sufijo maneja menos tipos de asset (no incluye flyer de financiera) que la rama `1`, que sí lo soporta. Revisar cuál rama es la realmente conectada al trigger (`When Executed by Another Workflow` conecta a `Switch · Tipo Asset1`, es decir, la rama `1` es la que efectivamente se ejecuta; la rama sin sufijo parece código muerto/no alcanzable).
- **Dependencias**: Airtable (`Modelos`, `Promociones`, `Config`, `Financieras`), Chatwoot HTTP.

### 👨🏻‍💻 SUB · Human Handoff
- **Trigger**: inputs `user_id_canal`, `conversation_id`, `sucursal_id`, ...
- **Propósito**: escalar la conversación a un asesor humano. Si el lead no tiene asesor, elige uno por **round-robin ponderado** (`Redis · RR Counter` + `Code · Pick Asesor & Build Payload`), lo asigna en Chatwoot y Airtable, pausa al bot (`Redis · Set Pause`), notifica al canal de Slack del equipo y por DM al asesor si tiene Slack vinculado, y avisa al cliente que fue transferido.
- **Dependencias**: Airtable (`Leads`, `Sucursales`, `Asesores`), Redis, Slack, Chatwoot HTTP.

### 📊📉 SUB · CRM Updater
- **Trigger**: inputs `user_id_canal`, `campo`, `valor`, `sucursal_id`.
- **Propósito**: permitir que el agente LLM guarde "en silencio" un dato revelado en la conversación (nombre, ciudad, colonia, presupuesto, etc.), un campo a la vez, con validación de qué campos son escribibles.
- **Dependencias**: Airtable (`Leads`, `Sucursales`), Redis.

## C. Post-proceso tras cada interacción

### 🥇🥈🏅 SUB · Lead Scorer
- **Trigger**: inputs `user_id_canal`, `lead_id`, `current_score`, `new_signal`(s).
- **Propósito**: recalcular el score del lead con un sistema de **pesos por señal** (ej. `pide_prueba_manejo: 20`, `pide_cita_visita: 18`, `quiere_apartar_con_enganche: ...`). Si el nuevo score cruza el umbral de "hot lead", alerta al canal de Slack de ventas.
- **Dependencias**: Airtable (`Leads`), Redis, Slack.

### 🪞 SUB · CRM Sucursal Mirror / 🪞 SUB · CRM Region Mirror
- **Trigger**: inputs `user_id_canal`, `sucursal_id_hint`, `ciudad_hint`.
- **Propósito**: arquitectura de **CRM maestro + espejos**. El maestro (`CRM Rivas Motors`) es la fuente de verdad; estos subworkflows resuelven a qué base de Airtable de sucursal/región corresponde el lead y **replican** (`upsert`) tanto el lead como sus citas hacia esa base. Si el lead cambió de sucursal/región respecto al espejo anterior, desactivan el registro en la base anterior. Guardan el estado del espejo en Redis (`mirror:lead:{id}` / `mirror:region:lead:{id}`) para detectar cambios en la siguiente ejecución.
- **Dependencias**: Airtable (múltiples bases: maestro + N sucursales/regiones), Redis.

### 👤 SUB · Auto-Asignar Asesor
- **Trigger**: inputs `user_id_canal`, `conversation_id`.
- **Propósito**: asignar automáticamente un asesor a un lead que aún no tiene uno, usando la misma mecánica de *weighted round-robin con overflow multi-sucursal* que Human Handoff, pero disparado proactivamente (post-mirror) en vez de por una solicitud explícita del cliente.
- **Detalle notable**: el workflow tiene **tres variantes casi idénticas** del guard "¿toca asignar?" y del picker de asesor (sufijos vacío, `1`, `3`), encadenadas por un nodo `If` inicial — indicio de iteraciones sucesivas del mismo mecanismo conviviendo en el mismo workflow.
- **Dependencias**: Airtable (`Leads`, `Sucursales`, `Asesores`), Redis (cache de asesores + contador round-robin), Slack, Chatwoot HTTP.

## D. Integración con financiera externa

### 🏦 SUB · Financiera Inbound
- **Trigger**: inputs `financiera_user_id`, `message_text`, `conversation_id_financiera`, ...
- **Propósito**: parsear los mensajes que el bot/agente de la financiera externa manda de vuelta (aprobación/rechazo de crédito), hacer *match* contra los leads enviados a financiera, y notificar al cliente final del resultado — vía Chatwoot si fue aprobado (con handoff a asesor humano) o rechazado (con aviso a Slack).
- **Patrón**: igual que Geo Router, usa **parseo determinístico primero, LLM como respaldo** (`If · ¿Necesita LLM?`) para interpretar mensajes con formato variable.
- **Match manual**: si no logra emparejar automáticamente el mensaje con un lead, notifica a Slack para intervención manual (`Slack · Match Manual`).
- **Dependencias**: Airtable (`Leads`, `Sucursales`), OpenAI, Slack, Chatwoot HTTP, ejecuta `👨🏻‍💻 SUB · Human Handoff` en caso de aprobación.

## E. Webhooks independientes de eventos Chatwoot

### 🏷️ SUB · Chatwoot Label Control
- **Trigger**: `webhook` propio — `POST /rivasmotors-chatwoot-label`.
- **Propósito**: escucha el evento de Chatwoot cuando un agente humano **agrega o quita una etiqueta** a una conversación, y usa eso como control manual de pausa/reactivación del bot (etiqueta de pausa → pausa Redis+Airtable; etiqueta de reactivación → limpia pausa).
- **Dependencias**: Airtable (`Leads`), Redis.

### 🔄 SUB · Chatwoot Reactivar Bot
- **Trigger**: `webhook` propio — `POST /rivasmotors-chatwoot-resolve`.
- **Propósito**: escucha `conversation_status_changed` de Chatwoot; cuando un asesor **resuelve** la conversación, reactiva el lead (decide nuevos campos de reactivación según la etapa actual) y limpia la pausa/caché en Redis. Además, marca la conversación con una etiqueta "IA" en Chatwoot para indicar que el bot retomó el control.
- **Dependencias**: Airtable (`Leads`), Redis, Chatwoot HTTP.

## F. Procesos programados (Cron)

### ⏰ CRON · Nurturing 14d
- **Trigger**: `scheduleTrigger` cada 30 min · **estado: inactivo**.
- **Propósito**: reenganchar leads fríos con mensajes de seguimiento automáticos, dentro de una ventana horaria (9–21h hora de Mérida) para no enviar de madrugada. Itera lead por lead (`splitInBatches`) con rate-limit de 2s entre envíos.
- **Dependencias**: Airtable (`Leads`), Chatwoot HTTP.

### ⏰ CRON · Recordatorio de Cita
- **Trigger**: `scheduleTrigger` diario a las 10h · **estado: activo**.
- **Propósito**: recordar a clientes con citas para el día siguiente, y avisar por Slack al asesor asignado (si tiene Slack vinculado).
- **Dependencias**: Airtable (`Citas`, `Leads`, `Sucursales`, `Asesores`), Chatwoot HTTP, Slack.

### ⏰ CRON · Resumen Diario
- **Trigger**: dos `scheduleTrigger` (diario 21h y semanal lunes 9h) · **estado: inactivo (todos los nodos deshabilitados)**.
- **Propósito**: publicar en Slack un resumen ejecutivo (leads nuevos, citas, calificados, hot leads sin atender, detalle por asesor), en modo diario o semanal según el trigger.
- **Dependencias**: Airtable (multi-tabla), Slack.

### SUB · Sync Monday to Airtable Inventario
- **Trigger**: `scheduleTrigger` cada 10h · **estado: activo**.
- **Propósito**: sincronizar el inventario de motos desde un tablero de **Monday.com** hacia la tabla `Inventario` de Airtable, con cache de 2 minutos en Redis para evitar syncs redundantes si se dispara manualmente en paralelo.
- **Dependencias**: Monday.com, Airtable, Redis.

### SYNC · Asesores → Redis
- **Trigger**: `manualTrigger` (para desarrollo) + `scheduleTrigger` mensual (día 1, 3am).
- **Propósito**: reconstruir la cache de asesores agrupados por sucursal (`asesores:cache`, TTL 35 días) que usan `Auto-Asignar Asesor` y `Human Handoff` para el round-robin ponderado, y notifica por Slack cuando termina.
- **Dependencias**: Airtable (`Asesores`, `Sucursales`), Redis, Slack.

## G. Utilidades y resiliencia

### 🧹 SUB · Reset Lead
- **Trigger**: `executeWorkflowTrigger` · **estado: inactivo**.
- **Propósito**: herramienta de mantenimiento/testing — borra todas las llaves de Redis de un lead, opcionalmente su memoria conversacional en Postgres (`n8n_chat_memory_bajajsureste`) y opcionalmente resetea su registro en Airtable. Pensado para reiniciar pruebas end-to-end con un mismo número de teléfono.
- **Dependencias**: Redis, Postgres, Airtable.

### ⚠️ Error Handler Global
- **Trigger**: `errorTrigger` (workflow de manejo de errores de n8n, se asocia como *Error Workflow* de los demás).
- **Propósito**: capturar cualquier error no manejado en cualquier workflow del proyecto, clasificar severidad, alertar al canal de Slack del equipo, e intentar (best-effort) avisar al cliente final vía Chatwoot si se puede extraer un `conversation_id` del error/stack.
- **Dependencias**: Slack, Chatwoot HTTP.

## Tabla resumen de dependencias externas por subworkflow

| Subworkflow | Airtable | Redis | Slack | Chatwoot HTTP | OpenAI | Postgres | Monday.com |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| Lead Hydrator | ✔ | ✔ | | | | | |
| Geo Router | ✔ | | | | ✔ | | |
| Promos Vigentes | ✔ | ✔ | | | | | |
| Inventory Lookup | ✔ | ✔ | | | | | |
| Inventory Overview | ✔ | ✔ | | | | | |
| FAQ Lookup | ✔ | | | | | | |
| Calcular Precotización | ✔ | ✔ | | | | | |
| Agenda Manager | ✔ | ✔ | ✔ | | | | |
| Media Sender | ✔ | | | ✔ | | | |
| Human Handoff | ✔ | ✔ | ✔ | ✔ | | | |
| CRM Updater | ✔ | ✔ | | | | | |
| Lead Scorer | ✔ | ✔ | ✔ | | | | |
| CRM Sucursal Mirror | ✔ | ✔ | | | | | |
| CRM Region Mirror | ✔ | ✔ | | | | | |
| Auto-Asignar Asesor | ✔ | ✔ | ✔ | ✔ | | | |
| Financiera Inbound | ✔ | | ✔ | ✔ | ✔ | | |
| Chatwoot Label Control | ✔ | ✔ | | | | | |
| Chatwoot Reactivar Bot | ✔ | ✔ | | ✔ | | | |
| Nurturing 14d | ✔ | | | ✔ | | | |
| Recordatorio de Cita | ✔ | | ✔ | ✔ | | | |
| Resumen Diario | ✔ | | ✔ | | | | |
| Sync Monday→Airtable | ✔ | ✔ | | | | | ✔ |
| Sync Asesores→Redis | ✔ | ✔ | ✔ | | | | |
| Reset Lead | ✔ | ✔ | | | | ✔ | |
| Error Handler Global | | | ✔ | ✔ | | | |
