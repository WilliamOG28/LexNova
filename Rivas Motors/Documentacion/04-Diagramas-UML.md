# Diagramas UML

## 1. Diagrama de Componentes (vista de sistema completo)

```mermaid
flowchart TB
    subgraph EXT["Sistemas Externos"]
        WA[WhatsApp\nMeta Cloud API]
        CW[Chatwoot]
        MON[Monday.com]
        FIN_EXT[Bot Financiera\nexterna]
    end

    subgraph N8N["n8n — Rivas Motors / Bajaj Sureste"]
        MAIN(("🔶 MAIN"))
        subgraph WEBHOOKS["Webhooks independientes"]
            LBL["🏷️ Chatwoot Label Control"]
            RES["🔄 Chatwoot Reactivar Bot"]
        end
        subgraph CRONS["Programados"]
            NUR["⏰ Nurturing 14d"]
            REC["⏰ Recordatorio de Cita"]
            RESU["⏰ Resumen Diario"]
            SYNCA["SYNC Asesores→Redis"]
            SYNCM["SYNC Monday→Airtable"]
        end
        subgraph DOM["Dominio"]
            LH["💦 Lead Hydrator"]
            GR["📍 Geo Router"]
            PV["🎟️ Promos Vigentes"]
            IL["🗃️ Inventory Lookup"]
            IO["🗂️ Inventory Overview"]
            FQ["🤔 FAQ Lookup"]
            PR["🏦 Calcular Precotización"]
            AG["🗓️ Agenda Manager"]
            MS["📸 Media Sender"]
            HH["👨‍💻 Human Handoff"]
            CU["📊 CRM Updater"]
            LS["🥇 Lead Scorer"]
            MSU["🪞 CRM Sucursal Mirror"]
            MRE["🪞 CRM Region Mirror"]
            AA["👤 Auto-Asignar Asesor"]
            FI["🏦 Financiera Inbound"]
        end
        ERR["⚠️ Error Handler Global"]
    end

    subgraph INFRA["Infraestructura"]
        AT[(Airtable\nCRM Rivas Motors\n+ N bases sucursal/región)]
        RD[(Redis)]
        SL[Slack]
        AI[OpenAI\nGPT-5.4-mini + Whisper]
        PG[(Postgres\nMemoria conversacional)]
    end

    WA <--> CW <--> MAIN
    CW --> LBL
    CW --> RES
    MON --> SYNCM
    FIN_EXT <--> FI

    MAIN --> LH & GR & PV & IL & IO & FQ & PR & AG & MS & HH & CU & LS & MSU & MRE & AA
    FI --> HH
    HH -.pausa/asigna.-> AA
    NUR & REC & RESU --> AT
    SYNCA --> RD
    SYNCM --> AT

    DOM --> AT
    DOM --> RD
    HH & AG & LS & AA & FI & SYNCA --> SL
    HH & MS & FI --> CW
    MAIN --> AI
    MAIN --> PG

    N8N -. errores no manejados .-> ERR
    ERR --> SL
    ERR --> CW
```

## 2. Diagrama de Secuencia — Mensaje entrante estándar (consulta comercial)

```mermaid
sequenceDiagram
    actor Cliente
    participant WA as WhatsApp/Chatwoot
    participant MAIN as MAIN (n8n)
    participant Redis
    participant LH as Lead Hydrator
    participant GR as Geo Router
    participant Router as Router LLM
    participant Agent as Agent Comercial (LLM)
    participant Tool as Tool (ej. Inventory Lookup)
    participant AT as Airtable

    Cliente->>WA: Envía mensaje(s)
    WA->>MAIN: Webhook POST
    MAIN->>Redis: Check Dedupe(message_id)
    Redis-->>MAIN: no procesado
    MAIN->>Redis: Push to Buffer + Debounce 8s
    MAIN->>Redis: Get Buffer (última ejecución)
    MAIN->>MAIN: Concat mensajes del buffer
    par Hidratación de contexto
        MAIN->>LH: Execute(user_id_canal, ...)
        LH->>Redis: Get lead cache
        alt cache miss
            LH->>AT: Search/Create Lead
            LH->>Redis: Save lead cache
        end
        LH-->>MAIN: lead + financieras
    and
        MAIN->>GR: Execute(message_text, ciudad_lead)
        GR-->>MAIN: sucursal/ciudad resuelta
    end
    MAIN->>Router: Classify Intent(mensaje + contexto)
    Router-->>MAIN: intención = "comercial"
    MAIN->>Agent: Invoke con historial (Postgres memory)
    Agent->>Tool: tool_call: Consultar Inventario(modelo)
    Tool->>Redis: Get inventory cache
    Tool-->>Agent: stock, precio, cilindrada
    Agent-->>MAIN: respuesta(s) generadas
    loop por cada mensaje de respuesta
        MAIN->>WA: Wait typing + Send message
    end
    MAIN->>MAIN: Execute Lead Scorer / CRM Mirrors / Auto-Asignar Asesor (async)
    WA-->>Cliente: Respuesta(s) del asesor virtual
```

## 3. Diagrama de Secuencia — Agendar cita

```mermaid
sequenceDiagram
    actor Cliente
    participant Agent as Agent Comercial
    participant AG as Agenda Manager
    participant AT as Airtable
    participant Redis
    participant Slack

    Cliente->>Agent: "Sí, el jueves a las 4pm para prueba de manejo"
    Agent->>AG: tool_call Agendar Cita(tipo, fecha, hora, sucursal_id)
    AG->>AG: Validar datos
    alt datos inválidos
        AG-->>Agent: {ok:false, mensaje_agente:"faltan datos"}
    else datos válidos
        AG->>AT: Get Sucursal
        AG->>AT: Check Conflicts (tabla Citas)
        alt horario ocupado
            AG-->>Agent: {ok:false, alternativas}
        else horario libre
            AG->>AT: Get Lead / Get Asesor
            AG->>AT: Create Cita
            AG->>AT: Update Lead Etapa
            AG->>Redis: Invalidate lead cache
            AG->>Slack: Notify Cita (canal sucursal)
            opt asesor tiene Slack
                AG->>Slack: DM al asesor
            end
            AG-->>Agent: {ok:true, mensaje_agente:"cita confirmada"}
        end
    end
    Agent-->>Cliente: Confirmación / solicitud de datos adicionales
```

## 4. Diagrama de Secuencia — Resultado de financiera externa

```mermaid
sequenceDiagram
    participant FinBot as Bot Financiera Externa
    participant WA as WhatsApp/Chatwoot
    participant MAIN
    participant FI as Financiera Inbound
    participant LLM as OpenAI (fallback parse)
    participant AT as Airtable
    participant HH as Human Handoff
    participant Cliente
    participant Slack

    FinBot->>WA: "Solicitud aprobada, cliente Juan Pérez, tel 999..."
    WA->>MAIN: Webhook (canal identificado como financiera)
    MAIN->>FI: Execute(financiera_user_id, message_text, ...)
    FI->>FI: Parse determinístico
    alt parseo ambiguo
        FI->>LLM: Interpretar mensaje
        LLM-->>FI: {nombre, telefono, estado}
    end
    FI->>AT: Get Leads Enviados a Financiera
    FI->>FI: Match Lead (por teléfono)
    alt no hay match
        FI->>Slack: Match Manual (intervención humana)
    else match encontrado
        FI->>AT: Update Lead Financiera (estado)
        alt aprobado
            FI->>WA: Msg Aprobado al Lead
            FI->>HH: Execute Human Handoff
            HH-->>Cliente: Asesor asignado da seguimiento
        else rechazado
            FI->>WA: Msg Rechazado al Lead
            FI->>Slack: Aviso Rechazo
        end
    end
```

## 5. Diagrama de Estados — Ciclo de vida del Lead (inferido)

> Los nombres exactos de `etapa_pipeline` no están enumerados explícitamente en un solo lugar del JSON (se arman dinámicamente en código); este diagrama refleja las **transiciones observables** a través de qué subworkflow las provoca.

```mermaid
stateDiagram-v2
    [*] --> Nuevo: Lead Hydrator crea el registro\n(primer contacto)
    Nuevo --> EnConversacion: Agent Comercial interactúa
    EnConversacion --> Calificado: Lead Scorer cruza umbral\n(hot lead)
    EnConversacion --> CitaAgendada: Agenda Manager crea Cita
    CitaAgendada --> EnConversacion: Recordatorio de Cita\n(refuerzo pre-cita)
    EnConversacion --> EnviadoAFinanciera: Calcular Precotización\nmarca "enviado a financiera"
    EnviadoAFinanciera --> AprobadoFinanciera: Financiera Inbound (aprobado)
    EnviadoAFinanciera --> RechazadoFinanciera: Financiera Inbound (rechazado)
    AprobadoFinanciera --> ConAsesor: Human Handoff asigna asesor
    Calificado --> ConAsesor: Auto-Asignar Asesor / Handoff Asesor
    EnConversacion --> PausadoBot: Asesor humano escribe\no etiqueta de pausa en Chatwoot
    ConAsesor --> PausadoBot: Handoff pausa el bot
    PausadoBot --> EnConversacion: Chatwoot Reactivar Bot\n(conversación resuelta)\no etiqueta de reactivación
    EnConversacion --> Frio: sin actividad
    Frio --> EnConversacion: Nurturing 14d reengancha
    RechazadoFinanciera --> [*]
```

## 6. "Diagrama de Clases" — Entidades principales (derivadas de los campos que fijan los nodos `Set`)

> n8n no tiene clases; estas son las **formas de datos** (shape) que viajan entre workflows, reconstruidas a partir de los campos asignados en nodos `Set` / `Code` de salida. Se documentan como clase para dar una referencia de "contrato de datos" entre MAIN y los subworkflows.

```mermaid
classDiagram
    class Lead {
        +string lead_id
        +bool lead_existe
        +string cliente_nombre_registrado
        +int lead_score
        +string etapa_pipeline
        +string modelo_interes
        +string categoria_interes
        +string uso_principal
        +string forma_pago_preferida
        +string financiera_estado
        +string financiera_elegida
        +string ciudad
        +string colonia
        +string sucursal_asignada
        +string ultima_intencion
        +string lead_summary
        +string[] signals_acumuladas
        +bool calificado
        +bool bot_paused
        +int score_maximo
        +int followups_count
        +bool _from_cache
    }

    class ContextoMensaje {
        +string user_id_canal
        +string conversation_id
        +string sender_name
        +string channel
        +string message
        +datetime received_at
    }

    class RespuestaHerramienta {
        +bool ok
        +string mensaje_agente
        +string link
    }

    class Cita {
        +string tipo_cita
        +date fecha
        +time hora
        +string sucursal_id
        +string lead_id
        +string asesor_id
    }

    class InventarioItem {
        +string modelo
        +int stock_total
        +float precio
        +int cilindrada
        +string categoria
    }

    ContextoMensaje "1" --> "1" Lead : hidrata (Lead Hydrator)
    Lead "1" --> "*" Cita : agenda (Agenda Manager)
    RespuestaHerramienta <|-- InventarioItem : especializa
    RespuestaHerramienta <|-- Cita : especializa
```
