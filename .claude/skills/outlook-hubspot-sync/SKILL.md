---
name: outlook-hubspot-sync
description: Lee las reuniones nuevas del calendario Outlook de <EJECUTIVA> y, para las que tienen invitados externos, busca o crea cuenta y contacto en HubSpot y crea el negocio en "Primera Reunión" si es la primera reunión con ese contacto. Usar al pedir sincronizar reuniones con HubSpot.
argument-hint: "[dry-run] [filtro opcional: texto del asunto o dominio]"
---

# Sincronizar reuniones de Outlook con HubSpot

Toda la configuración (owner, pipeline, etapa, dominios internos) y las reglas de negocio están en `CLAUDE.md` del proyecto. Respetarlas al pie de la letra.

Argumentos: `$ARGUMENTS`
- Si incluye `dry-run`: **no escribir nada en HubSpot ni en el estado**; solo mostrar el plan de acciones.
- Cualquier otro texto se usa como filtro (asunto o dominio) para procesar solo esas reuniones.

## Procedimiento

### 0. Verificaciones
- `Microsoft_365__get_me` → debe ser `<EMAIL_EJECUTIVA>`. Si no, frenar.
- Leer `state/processed.json` (`lastRun`, `processedEventIds`).

### 1. Leer reuniones
- `outlook_calendar_search` en el calendario propio (sin `calendarOwnerEmail`), usando **siempre un rango de fechas** (`afterDateTime`/`beforeDateTime`) porque así devuelve organizador e invitados. Rango: **hoy 00:00 ART hasta el momento actual** (salvo que el usuario pida otra fecha al ejecutarlo a mano). Nunca reuniones futuras. Paginar con `offset`.
- Si hay filtro en los argumentos, pasarlo como `query`.
- Nota: `read_resource` puede fallar con IDs que contienen `/`; usar los datos que ya trae la búsqueda.

### 2. Filtrar
- Saltar eventos cuyo `id` esté en `processedEventIds`, los cancelados, los que no tengan invitados y los que **todavía no terminaron** (`end` > ahora).
- Para cada evento, separar invitados (y organizador si no es la ejecutiva) en internos/externos según los dominios internos de `CLAUDE.md`. Si no hay externos → ignorar (reportar "solo internos").
- Ignorar eventos/webinars y buzones genéricos según la regla 1 de `CLAUDE.md`.
- Agrupar los externos por dominio.

### 3. Por cada dominio externo → Cuenta
- `search_crm_objects` COMPANY con `domain EQ <dominio>`; traer `name, domain, hubspot_owner_id`.
- No existe → crear COMPANY: `name` (nombre de la empresa derivado del dominio, capitalizado), `domain`, `hubspot_owner_id` = owner de la ejecutiva.
- Existe → no modificar. Anotar su owner para el reporte.

### 4. Por cada invitado externo → Contacto
- `search_crm_objects` CONTACT con `email EQ <email en minúsculas>`.
- No existe → crear CONTACT con `email`, `firstname`, `lastname`, `hubspot_owner_id` = la ejecutiva, asociado a la cuenta.
  - Email y nombre: validarlos según la regla 7 de `CLAUDE.md`, **empezando por el servidor de correo del contacto**: (a) `dig +short MX <dominio>` debe devolver registros; (b) `outlook_email_search` con `sender` = email → si hay correos suyos, el email está validado y el nombre sale de `sender.name` (`read_resource` del mensaje). Recién después Apollo (búsqueda gratis; `apollo_people_match` cuesta 1 crédito: solo con aprobación). Si no se valida, no crear el contacto y pedir confirmación en el reporte.
- Existe → no modificar sus propiedades; si no está asociado a la cuenta, agregar solo la asociación.

### 5. ¿Primera reunión o seguimiento?
- Por cada cuenta: `search_crm_objects` DEAL con filtros `hubspot_owner_id EQ <OWNER_ID_EJECUTIVA>` y `pipeline EQ <PIPELINE_ID>`, en dos filterGroups (OR): `associatedWith` cuenta EQUAL id / `associatedWith` contacto IN ids de los contactos externos.
- Traer `hs_is_closed`, `hs_is_closed_won`, `dealstage`, `closedate`. Hacer este chequeo **antes** de crear contactos nuevos de una cuenta que ya existía.
- Hay al menos uno **abierto** → **seguimiento**: no crear negocio.
- Solo hay negocios **cerrados** (ganados o perdidos) → **caso dudoso**: no crear negocio ni contactos de esa cuenta, ni descartar la reunión; preguntarle a la ejecutiva (mostrar los negocios cerrados con link) si crear un negocio nuevo o tomarlo como seguimiento.
- Ninguno → **primera reunión**. Los negocios de otros owners u otros pipelines se ignoran.
- Un único negocio por cuenta, asociado a todos los contactos externos de ese dominio.

### 6. Crear negocio
- DEAL con:
  - `dealname`: `<Nombre de la empresa> – <Nombre y apellido del contacto>` (si el negocio agrupa varios contactos, el primero de la lista de invitados)
  - `pipeline`, `dealstage`: los de `CLAUDE.md` (Primera Reunión)
  - `hubspot_owner_id`: owner de la ejecutiva (siempre)
  - `closedate`: fecha de la reunión + 3 meses (ISO)
  - `hs_forecast_probability`: `0.05`
  - `tipo_de_servicio`: `Servicios IT`
  - `origen`: según la **Regla de Origen** de `CLAUDE.md` (organizador del evento + si la cuenta y el contacto existían antes de esta corrida — anotarlo en los pasos 3 y 4)
  - `tipo_de_cliente`: `Nuevo` / `Existente` según la regla de `CLAUDE.md` (lifecyclestage de la cuenta y negocios ganados; contrastar con info pública)
  - `pais`: país de la empresa (`country` de la cuenta en HubSpot validado con información pública; si no está en la lista → `Otro`)
  - `modo_de_presentacion`: no enviar. **Después de crear**, releer el negocio: HubSpot lo autocompleta con "Presentación presencial"; si tiene valor, actualizarlo a `""` y volver a verificar.
  - `description`: `Creado por asistente Outlook→HubSpot. Reunión: <asunto> — <fecha y hora>. Outlook event id: <id>`
  - Asociaciones: COMPANY + cada CONTACT de ese grupo.
- Máximo 10 objetos por llamada a `manage_crm_objects`.

### 6b. Verificar owner
- Releer cada cuenta, contacto y negocio creados en esta corrida (`get_crm_objects` con `hubspot_owner_id`). Si alguno no tiene `<OWNER_ID_EJECUTIVA>`, actualizarlo a `<OWNER_ID_EJECUTIVA>`, volver a verificar y reportarlo.

### 7. Estado
- Fuera de dry-run: agregar el id de cada evento procesado (incluidos seguimiento e ignorados) a `processedEventIds` y actualizar `lastRun` con la fecha/hora actual.

### 7b. Casos dudosos
- Si hay casos que no se pueden resolver con las reglas (ver `CLAUDE.md` → Ejecución), no escribir nada para esos casos y preguntarle a la ejecutiva (en ejecución manual, directamente en la conversación; en la rutina, en el mensaje diario de Slack).

### 8. Reporte
Tabla por reunión: fecha, asunto, resultado (negocio creado / seguimiento / ignorada y motivo), cuenta (creada o existente + owner), contactos (creados o existentes), y links de HubSpot:
- Cuenta: `https://app.hubspot.com/contacts/<PORTAL_ID>/record/0-2/<id>`
- Contacto: `https://app.hubspot.com/contacts/<PORTAL_ID>/record/0-1/<id>`
- Negocio: `https://app.hubspot.com/contacts/<PORTAL_ID>/record/0-3/<id>`
