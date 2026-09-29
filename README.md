Checkpoint 4 — Integraciones Avanzadas e Interconexión de Sistemas

Cuarta entrega del proyecto integrador. Sobre el Manager de los checkpoints anteriores (arquitectura multi-agente + memoria persistente en Google Sheets) se agrega un circuito nuevo e independiente: integraciones reales con Gmail, HubSpot (CRM) y Slack, vía OAuth2, para procesar emails entrantes a una casilla de soporte.

Estructura del repositorio
/checkpoint4_almendra_andres.json   → Manager completo (chat + memoria + email/CRM/Slack)
/workers/worker1_tech_support.json  → Worker de Soporte Técnico (sub-workflow)
/workers/worker2_billing.json       → Worker de Facturación (sub-workflow)

Los Workers deben existir como workflows independientes en la misma instancia de n8n para que los nodos Execute Workflow del Manager puedan invocarlos. Al importar, hay que reconectar esos nodos a los Workers ya importados (los IDs de workflow son específicos de cada instancia).

Dos circuitos independientes, un solo proyecto

El Manager tiene dos puntos de entrada distintos que conviven en el mismo lienzo sin conectarse entre sí:

Circuito 1 — Chat + memoria + multi-agente (Módulos 1 a 3)

Chat Trigger → Google Sheets (Buscar Memoria) → IF ¿Tiene Historial?
    → Limpiar Payload → Router de Triaje → Delegación a Workers
    → Unificar Respuesta → IF ¿Supera 5 Intercambios? → Resumen / Progreso → Slack (Log)

Circuito 2 — Integraciones de negocio (Módulo 4, nuevo)

Gmail Trigger (casilla de soporte)
    → ① IF ¿Es Auto-Reply? ──Sí──▶ Cortar Bucle (fin)
    → AI Agent (clasifica y redacta borrador)
    → ④ Set (limpia y valida el payload: From, Subject, body_text)
    → ¿Email Válido? ──No──▶ Descartar (Email Vacío)
    → ② Look up (¿el contacto ya existe en HubSpot?)
    → Actualizar o Crear Contacto (operación Create or Update / upsert)
    → ③ Create Draft (Human-in-the-Loop, nunca envía directo)
    → Limpiar Payload para Slack (solo texto liviano)
    → Slack (Notificar Equipo de Operaciones)
Los 4 guardrails que evalúa la rúbrica del Módulo 4
① IF anti auto-reply: escanea Subject (Auto-reply / Out of Office / Undeliverable) y From (no-reply@ / noreply@) para cortar el bucle infinito de auto-respuestas antes de llegar al agente.
② Look up antes de Create: busca el contacto por email en HubSpot antes de escribir, dejando evidencia visual del paso que previene duplicados (Error 409).
③ Create Draft: el nodo de salida de Gmail usa exclusivamente la operación de crear borrador — nunca envía un email sin que un humano lo revise y apruebe primero.
④ Set de limpieza de payload: dos instancias — antes del CRM (solo From, Subject, body_text) y antes de Slack (remitente, asunto y resumen truncado) — nunca se vuelca HTML ni payloads pesados en sistemas externos.
Memoria y contexto (heredado del Módulo 3, corregido en esta entrega)

El agente de Soporte Técnico recibe user_name y last_summary como variables separadas, mapeadas desde las columnas Nombre de Usuario y Resumen Consolidado de Google Sheets, e inyectadas en su System Prompt con delimitadores rígidos:

[INICIO DE CONTEXTO COMPARTIDO]
CONFIGURACION DE IDENTIDAD Y MEMORIA DE LARGO PLAZO
El usuario se llama {{ $json.user_name }}. El contexto de vuestra ultima charla es: {{ $json.last_summary }}.
[FIN DEL CONTEXTO COMPARTIDO]

Estas variables se propagan de punta a punta: Google Sheets → Preparar Contexto Recuperado/Usuario Nuevo → Limpiar Payload → Delegar - Worker Soporte_tecnico → Execute Workflow Trigger del Worker → AI Agent.

Detalles técnicos a tener en cuenta
El Gmail Trigger en modo simple devuelve los campos con mayúscula inicial (From, Subject), no en minúscula, y From llega como texto plano (Nombre <email@dominio.com>), no como objeto — se extrae el email con una expresión regular en el nodo ④ Set.
HubSpot usa la operación "Create or Update" (upsert) en un solo nodo: decide internamente si crea o actualiza según el email, evitando el Error 409 sin necesitar un IF adicional después del Look up.
La credencial de Gmail apunta a una casilla de pruebas dedicada, separada de la personal, para no saturarla durante los tests.
Principio de Mínimo Privilegio

La credencial de HubSpot está autorizada únicamente con los scopes de lectura y escritura de contactos — sin acceso a negocios, tickets, ni configuración de cuenta.

Cómo probarlo
Importar los 3 archivos como workflows separados en n8n y reconectar los nodos "Delegar - Worker..." del Manager a los Workers propios.
Reconectar las 4 credenciales (Gmail, HubSpot, Slack, OpenRouter) con cuentas propias.
Circuito de chat: probar un mensaje de soporte técnico, uno de facturación, y uno ambiguo — confirmando que la memoria (user_name/last_summary) llega correctamente al Worker en sesiones nuevas y recurrentes.
Circuito de email: mandar un email nuevo (crea contacto + borrador), repetir desde el mismo remitente (actualiza, no duplica), y mandar uno con asunto "Out of Office" (se corta en el IF ①).
Confirmar en el panel de ejecución que todo el recorrido pinta en verde de punta a punta, en ambos circuitos.
