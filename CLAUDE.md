# Asistente Outlook → HubSpot (AI Day Flock)

Cuando hay una reunión nueva en Outlook con un invitado externo, el asistente crea la oportunidad (Deal) en HubSpot, busca o crea la cuenta (Company) y el contacto, y los asocia.
Se ejecuta con el skill `/outlook-hubspot-sync` (ver `.claude/skills/outlook-hubspot-sync/SKILL.md`).

## Configuración

Los valores reales de los marcadores (`<EJECUTIVA>`, `<OWNER_ID_EJECUTIVA>`, etc.) están en `config.local.md` (no se sube al repo; plantilla en `config.example.md`):

@config.local.md

| Clave | Valor |
|---|---|
| Calendario | Solo el de <EJECUTIVA> — `<EMAIL_EJECUTIVA>` |
| Owner HubSpot de la ejecutiva | `<OWNER_ID_EJECUTIVA>` |
| Portal HubSpot | `<PORTAL_ID>` |
| Pipeline de negocios | `<PIPELINE_NOMBRE>` — id `<PIPELINE_ID>` |
| Etapa inicial | `Primera Reunión` — id `<STAGE_ID_PRIMERA_REUNION>` |
| Dominios internos | Cualquier dominio que contenga `flockit` (`flockit.com.ar`, `flockit.es`), `acciona` (`acciona-it.com`) o `a-factory` (`a-factory.es`) |
| Nombre del negocio | `<Nombre de la empresa> – <Nombre y apellido del contacto>` (si hay varios contactos nuevos, el primero en la lista de invitados) |
| Estado | `state/processed.json` |

## Propiedades del negocio

| Propiedad (label) | Nombre interno | Valor |
|---|---|---|
| Fecha de cierre | `closedate` | Fecha de la reunión + **3 meses** |
| Forecast probability | `hs_forecast_probability` | `0.05` (5%) en etapa Primera Reunión |
| Tipo de servicio | `tipo_de_servicio` | Siempre `Servicios IT` |
| Origen | `origen` | Según el organizador de la reunión — ver **Regla de Origen** |
| Tipo de cliente | `tipo_de_cliente` | `Existente` si la cuenta tiene `lifecyclestage = customer` o algún negocio en etapa de cierre ganado; si no, `Nuevo`. Contrastar con información pública de la empresa |
| País | `pais` | País de la empresa: `country` de la cuenta en HubSpot, validado con información pública (Apollo / web / TLD del dominio). Opciones: Argentina, Chile, España, EEUU, Otro |
| Modo de Presentación | `modo_de_presentacion` | **Vacío** (no enviar) |

## Regla de Origen

La existencia de cuenta y contacto se evalúa **antes** de que el asistente cree nada. Se aplica la primera regla que coincida:

| # | Organizador de la reunión | Cuenta | Contacto | `origen` (valor interno) |
|---|---|---|---|---|
| 1 | `<SDR_1>` o `<SDR_2>` (SDRs; emails y owners en la configuración) | — | — | `SDR - Argentina` |
| 2 | Otra persona interna (dominio Flock / acciona / a-factory) que no es la ejecutiva | — | — | `Contacto generado por otro ejecutivo (Cuenta Nueva)` |
| 3 | la ejecutiva o el invitado externo | No existía | — | `Campaña Ejecutivo (Cuenta Nueva)` |
| 4 | la ejecutiva o el invitado externo | Existía | No existía | `Nuevo contacto (Cliente Existente)` |
| 5 | la ejecutiva o el invitado externo | Existía | Existía | `Contacto conocido (Cliente Existente)` |

- Identificar al organizador por email (comparación sin distinguir mayúsculas/minúsculas).
- Si en un mismo negocio hay varios contactos y alguno ya existía, usar la regla 5 solo si **todos** existían; si no, la 4.

## Reglas de negocio

1. **Reunión que califica**: **ya sucedió** (su hora de fin es anterior al momento de la corrida; las reuniones futuras nunca generan nada, ni cuenta, ni contacto, ni negocio) y tiene al menos un invitado con dominio externo. Si además hay invitados internos (Flock / acciona / a-factory), igual califica; los internos simplemente se ignoran. Reuniones solo con internos se ignoran. También se ignoran las canceladas.
   - **Se ignoran eventos, webinars y similares** (ferias, conferencias, "Live", summits, meetups, lanzamientos, capacitaciones abiertas): señales = asunto/cuerpo con esas palabras, ubicación en hotel/auditorio, invitación masiva o enviada por plataforma de marketing. Se listan en el reporte como "evento ignorado".
   - **Se ignoran buzones genéricos** como invitados externos: direcciones tipo `marketing@`, `mkt.` en el dominio, `events@`/`eventos@`, `noreply@`/`no-reply@`, `info@`, `newsletter@`, `notifications@`, `fieldmarketing`. No generan cuenta, contacto ni negocio.
2. **Cuenta**: se busca por `domain` del invitado externo. Si no existe se crea con owner la ejecutiva. Si existe, **no se modifica** (ni owner ni campos), solo se usa su ID.
3. **Contacto**: se busca por `email`. Si no existe se crea con owner la ejecutiva. Si existe, **no se modifica**. Se asocia a la cuenta si falta esa asociación.
4. **Primera reunión vs. seguimiento**: antes de crear un negocio se verifica si ya hay un negocio **de la ejecutiva** (`hubspot_owner_id = <OWNER_ID_EJECUTIVA>`) en el pipeline **<PIPELINE_NOMBRE>** asociado a esa cuenta o a ese contacto. Si hay al menos uno **abierto** → **seguimiento**, no se crea nada. Si solo hay negocios **cerrados** (ganados o perdidos) → **caso dudoso**: no se crea ni se descarta nada para esa cuenta (ni contactos nuevos ni negocio) y se le pregunta a la ejecutiva si crear un negocio nuevo o tomarlo como seguimiento. Negocios de **otros owners o de otros pipelines no cuentan**: se ignoran y se crea el negocio igual.
5. Varios contactos nuevos del mismo dominio en una reunión → **un solo negocio** por cuenta asociado a todos ellos.
6. **Owner**: todo lo que crea el asistente (cuenta, contacto, negocio) es **siempre de la ejecutiva** (`<OWNER_ID_EJECUTIVA>`); nunca se asigna a otro usuario, aunque la cuenta tenga otro owner o la reunión la haya organizado otra persona. HubSpot tiene una automatización que copia el owner de la cuenta al contacto: **después de crear, releer el owner** de cada objeto creado y, si no es la ejecutiva, volver a ponerlo en `<OWNER_ID_EJECUTIVA>` y reportarlo.
7. **Validación del contacto (email y nombre)** antes de crearlo. Primero siempre el **servidor de correo del propio contacto**:
   1. **Dominio con correo activo**: el dominio del email debe tener registros MX (`dig +short MX <dominio>`). Sin MX → no crear, reportar.
   2. **Email confirmado por su servidor**: buscar correos enviados **desde** ese email a la ejecutiva (`outlook_email_search` con `sender` = email; ej. la respuesta "Aceptada: …" a la invitación). Si su servidor emitió correo desde esa dirección, el email existe. De ese mismo mensaje se toma el **nombre** (`sender.name` vía `read_resource`), que lo pone su propio servidor.
   3. Si no hay correos suyos: Apollo `apollo_mixed_people_api_search` con el dominio y nombre/apellido como keyword (gratis).
   4. Apollo `apollo_people_match` por email: cuesta 1 crédito → solo con aprobación explícita de la ejecutiva.
   5. LinkedIn no tiene conector: si hay dudas, se informa para validar a mano.
   Si el email no se pudo validar en su servidor (pasos 1–2) o el nombre no es confiable, se marca "email sin validar" en el reporte y se pide confirmación antes de crear. No inventar datos.
8. Nunca editar ni borrar registros que el asistente no creó.

## Ejecución

- Rutina en la nube `<ROUTINE_ID>` (prompt en `routine/prompt.md`): **todos los días a las 17:00 ART** (cron UTC `0 20 * * *`), Sonnet 5.5, conectores Microsoft-365 + HubSpot + Apollo-io + Slack. Activa desde el 09/10/2026.
- Solo procesa las reuniones **de ese día** que ya terminaron (00:00 ART → hora de la corrida). Reuniones del mismo día que terminan después de las 17:00 no se procesan.
- **Casos dudosos** (solo hay negocios de la ejecutiva cerrados en el pipeline, contacto sin validar, nombre dudoso, varias cuentas con el mismo dominio, empresa del asunto distinta al dominio, evento vs. reunión comercial, owner que HubSpot vuelve a cambiar, errores): no se escribe nada para ese caso y se le consulta a la ejecutiva en el mensaje diario de Slack.
- **Mensaje diario por Slack** (conector Slack, cuenta `<SLACK_CUENTA>`, DM a sí misma `channel_id <SLACK_USER_ID>`; nunca a otro canal ni persona): **siempre** al final de cada corrida, un solo mensaje con lo cargado ese día (negocios, cuentas y contactos con links), seguimientos, ignoradas y la sección "⚠️ Necesito que me digas qué hacer" con los casos dudosos. Slack no notifica los mensajes a uno mismo: La ejecutiva lo revisa en su DM. Solo si Slack falla → notificación push + detalle en el reporte de la sesión.
  - Descartados: Teams (el conector no tiene `ChatMessage.Send`) y webhook a `#hubspot-asistente` (el entorno de la nube bloquea `hooks.slack.com` con 403).
- La ejecución manual sigue disponible con `/outlook-hubspot-sync`.
- La rutina no ve estos archivos: su prompt es autocontenido. **Si cambia una regla acá, actualizar también `routine/prompt.md` y la rutina.**
