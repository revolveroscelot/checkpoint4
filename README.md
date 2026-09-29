# Checkpoint 4 — Integraciones Avanzadas e Interconexión de Sistemas

Cuarta entrega del proyecto integrador: se le suman al Manager (que ya tenía memoria y arquitectura multi-agente de los checkpoints anteriores) integraciones reales con **Gmail**, **HubSpot (CRM)** y **Slack**, vía OAuth2, para procesar emails entrantes a una casilla de soporte.

Este circuito convive en el mismo archivo que el resto del proyecto, pero es **independiente**: se dispara con su propio trigger (un email entrante), no con el chat del Manager de los módulos anteriores.

## Arquitectura

```
Gmail Trigger (casilla de soporte)
        │
        ▼
① IF - ¿Es Auto-Reply? ──Sí──▶ Cortar Bucle (fin)
        │ No
        ▼
AI Agent - Clasifica y Redacta (borrador de respuesta)
        │
        ▼
④ Set - Limpia y Valida el Payload  (deja solo From, Subject, body_text)
        │
        ▼
   ¿Email Válido? ──No──▶ Descartar (Email Vacío)
        │ Sí
        ▼
② Look up - ¿Contacto ya existe en HubSpot?
        │
        ▼
Actualizar o Crear Contacto  (operación Create or Update / upsert)
        │
        ▼
③ Create Draft (Human-in-the-Loop)  — Gmail, nunca envía directo
        │
        ▼
Limpiar Payload para Slack  (solo texto liviano)
        │
        ▼
Slack - Notificar Equipo de Operaciones
```

## Los 4 guardrails que evalúa la rúbrica

- **① IF anti auto-reply**: escanea `Subject` (Auto-reply / Out of Office / Undeliverable) y `From` (no-reply@ / noreply@) para cortar el bucle infinito de auto-respuestas antes de llegar al agente.
- **② Look up antes de Create**: busca el contacto por email en HubSpot antes de escribir, dejando evidencia visual del paso que previene duplicados.
- **③ Create Draft**: el nodo de salida de Gmail usa exclusivamente la operación de crear borrador — nunca envía un email sin que un humano lo revise y apruebe primero (Human-in-the-Loop).
- **④ Set de limpieza de payload**: dos instancias — una antes de tocar el CRM (deja solo `From`, `Subject`, `body_text`) y otra antes de Slack (solo remitente, asunto y un resumen truncado), evitando volcar HTML o payloads pesados en ningún sistema externo.

## Detalles técnicos a tener en cuenta

- El nodo **Gmail Trigger** en modo simple devuelve los campos con mayúscula inicial (`From`, `Subject`), no en minúscula — las expresiones de todo el circuito están ajustadas a eso.
- El campo `From` llega como texto plano (`Nombre <email@dominio.com>`), no como objeto — se extrae el email con una expresión regular en el nodo `④ Set`.
- **HubSpot** usa la operación **"Create or Update"** (upsert) en un solo nodo: internamente decide si crea o actualiza según el email, evitando el Error 409 sin necesitar un IF adicional después del Look up.
- Las credenciales de Gmail apuntan a una casilla de pruebas dedicada, separada de la casilla personal, para no saturarla durante los tests.

## Principio de Mínimo Privilegio

La credencial de HubSpot está autorizada únicamente con los scopes de lectura y escritura de **contactos** — sin acceso a negocios, tickets, ni configuración de cuenta.

## Cómo probarlo

1. Importar el `.json` en n8n y reconectar las 4 credenciales (Gmail, HubSpot, Slack, OpenRouter) con tus propias cuentas.
2. Mandar un email de prueba a la casilla de soporte configurada en el Gmail Trigger:
   - Un email nuevo → debería crear un contacto en HubSpot y un borrador en Gmail.
   - Un segundo email del mismo remitente → debería actualizar el contacto existente, sin duplicarlo.
   - Un email con asunto "Out of Office" → debería cortarse en el IF ①, sin llegar al agente.
3. Confirmar que llega la notificación liviana a Slack en los casos que sí se procesan.
4. Revisar el panel de ejecución de n8n: todo el recorrido debe pintarse en verde de punta a punta.

## Próximos pasos del proyecto

Este circuito de integraciones se sigue ampliando en los módulos siguientes hasta el Proyecto Final Integrador (M11).# checkpoint4
