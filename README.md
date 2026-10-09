# Asistente Outlook → HubSpot

Proyecto del AI Day de Flock. Un asistente hecho con [Claude Code](https://claude.com/claude-code) que, todos los días, lee las reuniones del calendario de Outlook de una ejecutiva comercial y carga en HubSpot la cuenta, el contacto y el negocio de cada primera reunión con un cliente externo. Al terminar, le manda un informe por Slack.

No hay código: el comportamiento está definido en lenguaje natural y Claude lo ejecuta usando los conectores de Microsoft 365, HubSpot, Apollo y Slack.

## Flujo

```
 17:00 ART, todos los días (rutina en la nube)
          │
          ▼
 ┌────────────────────────────────────┐
 │ 1. Leer el calendario de Outlook   │  Solo reuniones de HOY que ya terminaron
 └────────────────────────────────────┘
          │
          ▼
 ┌────────────────────────────────────┐
 │ 2. Filtrar                         │  Ignora: canceladas, solo internos,
 │                                    │  eventos/webinars, buzones genéricos
 └────────────────────────────────────┘
          │ (reuniones con al menos un externo, agrupadas por dominio)
          ▼
 ┌────────────────────────────────────┐
 │ 3. Cuenta (por dominio)            │  Existe → la usa sin modificarla
 │                                    │  No existe → la crea (owner: la ejecutiva)
 │                                    │  Varias con el mismo dominio → CONSULTA
 └────────────────────────────────────┘
          │
          ▼
 ┌────────────────────────────────────┐
 │ 4. ¿Ya hay negocio de la ejecutiva │  Abierto → SEGUIMIENTO (no crea nada)
 │    en el pipeline?                 │  Solo cerrados → CONSULTA
 │                                    │  Ninguno → sigue
 └────────────────────────────────────┘
          │
          ▼
 ┌────────────────────────────────────┐
 │ 5. Contacto (por invitado)         │  Existe → lo usa sin modificarlo
 │                                    │  No existe → valida el email y el nombre
 │                                    │    No se puede validar → CONSULTA
 └────────────────────────────────────┘
          │
          ▼
 ┌────────────────────────────────────┐
 │ 6. Crear el negocio                │  En "Primera Reunión", con Origen,
 │                                    │  Tipo de cliente, País, etc.
 └────────────────────────────────────┘
          │
          ▼
 ┌────────────────────────────────────┐
 │ 7. Verificar el owner              │  Si HubSpot lo cambió → lo corrige
 │                                    │  Si se repite → CONSULTA
 └────────────────────────────────────┘
          │
          ▼
 ┌────────────────────────────────────┐
 │ 8. Informe diario por Slack        │  Cargado hoy · Seguimientos ·
 │                                    │  Ignoradas · ⚠️ Consultas
 └────────────────────────────────────┘
```

### Validación de contactos

Antes de crear un contacto, el asistente comprueba que el email exista:

1. El dominio tiene registros MX (correo activo).
2. Busca correos enviados desde ese email a la ejecutiva, por ejemplo el "Aceptada: …" de la invitación. Si su servidor emitió correo, el email existe, y el nombre se toma de ese mensaje.
3. Si no hay correos suyos, hace una búsqueda gratuita en Apollo.
4. Si no se puede validar, no lo crea y lo consulta.

### Origen del negocio

| Organizador de la reunión | Situación | Origen |
|---|---|---|
| SDR | — | SDR - Argentina |
| Otro ejecutivo interno | — | Contacto generado por otro ejecutivo (Cuenta Nueva) |
| La ejecutiva o el externo | Cuenta nueva | Campaña Ejecutivo (Cuenta Nueva) |
| La ejecutiva o el externo | Cuenta existente, contacto nuevo | Nuevo contacto (Cliente Existente) |
| La ejecutiva o el externo | Cuenta y contacto existentes | Contacto conocido (Cliente Existente) |

### Casos en los que consulta (y no carga nada para ese caso)

- La cuenta solo tiene negocios cerrados de la ejecutiva.
- Un contacto que no se pudo validar o tiene un nombre dudoso.
- Varias cuentas con el mismo dominio.
- La empresa del asunto no coincide con el dominio del invitado.
- No está claro si es un evento o una reunión comercial.
- HubSpot vuelve a cambiar el owner.
- Errores de las herramientas.

### Lo que nunca hace

Crear registros a nombre de otro usuario, modificar o borrar registros que no creó, tocar otros pipelines, usar créditos de Apollo, enviar correos o escribirle a otra persona.

## Estructura

| Archivo | Qué es |
|---|---|
| `CLAUDE.md` | Reglas de negocio y configuración. Claude Code lo lee en cada sesión |
| `.claude/skills/outlook-hubspot-sync/SKILL.md` | Skill para correrlo a mano: `/outlook-hubspot-sync [dry-run] [filtro]` |
| `routine/prompt.md` | Prompt autocontenido de la rutina diaria en la nube |
| `config.example.md` | Plantilla de los valores reales (IDs de HubSpot, emails, Slack) |

Los datos reales (emails, IDs de HubSpot y Slack) no están en el repo: se usan marcadores como `<OWNER_ID_EJECUTIVA>`, cuyos valores van en `config.local.md`, que está en `.gitignore`.

## Correrlo en local

### Requisitos

- [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) instalado, con sesión iniciada con una cuenta de claude.ai.
- En esa cuenta, conectores activos en https://claude.ai/customize/connectors: **Microsoft 365**, **HubSpot**, **Apollo** y **Slack**. Claude Code los toma automáticamente; se verifican con `/mcp`.
- `dig` disponible en la terminal (viene en macOS y en la mayoría de los Linux), para validar los dominios de correo.

### Variables

Copiar la plantilla y completar los valores:

```bash
cp config.example.md config.local.md
```

`CLAUDE.md` importa `config.local.md`, así que Claude reemplaza cada marcador por su valor. Ese archivo está en `.gitignore` y no se sube.

| Variable | Qué es | Dónde se obtiene |
|---|---|---|
| `<EJECUTIVA>` | Nombre de la ejecutiva dueña del calendario | — |
| `<EMAIL_EJECUTIVA>` | Su email de Outlook / Microsoft 365. Solo se procesa este calendario | — |
| `<OWNER_ID_EJECUTIVA>` | Su id de owner en HubSpot. Todo lo que crea el asistente queda a su nombre | HubSpot → Settings → Users & Teams, o pidiéndole a Claude "buscá mi owner id en HubSpot" |
| `<PORTAL_ID>` | Id de la cuenta (portal) de HubSpot, para armar los links | Menú de la cuenta (arriba a la derecha) o el número en la URL `app.hubspot.com/contacts/<PORTAL_ID>/...` |
| `<PIPELINE_NOMBRE>` | Nombre del pipeline de negocios donde se crean los deals | HubSpot → Settings → Objects → Deals → Pipelines |
| `<PIPELINE_ID>` | Id interno de ese pipeline | Mismo lugar, o pidiéndole a Claude "listá los pipelines de deals con sus ids" |
| `<STAGE_ID_PRIMERA_REUNION>` | Id de la etapa inicial ("Primera Reunión") | Mismo lugar que el pipeline |
| `<SDR_1>`, `<SDR_2>` | Nombre, email y owner id de los SDRs. Sus reuniones llevan Origen "SDR - Argentina" | Emails corporativos; owner id como el de la ejecutiva |
| `<EMAIL_SDR_1>`, `<EMAIL_SDR_2>` | Emails de los SDRs (se comparan con el organizador de la reunión) | — |
| `<SLACK_CUENTA>` | Cuenta de Slack conectada al conector de Claude | — |
| `<SLACK_USER_ID>` | Id de usuario de Slack de la ejecutiva. El informe diario se manda a su DM consigo misma | Slack → tu perfil → ⋮ → **Copiar ID de miembro** (empieza con `U`) |
| `<ROUTINE_ID>` | Id de la rutina diaria en la nube (solo como referencia) | URL de la rutina en https://claude.ai/code/routines |

Configuración fija que también se puede adaptar en `CLAUDE.md`: los **dominios internos** (hoy, cualquiera que contenga `flockit`, `acciona` o `a-factory`) y los **valores de propiedades** del negocio (`tipo_de_servicio`, `origen`, `pais`, etc.), que deben coincidir con las opciones de tu HubSpot.

### Ejecutar

```bash
git clone https://github.com/Fatimabottero/asistente-opp-outlook-hubspot.git
cd asistente-opp-outlook-hubspot
claude
```

Dentro de Claude Code:

- `/outlook-hubspot-sync dry-run`: muestra qué haría, sin escribir nada en HubSpot. Conviene empezar por acá.
- `/outlook-hubspot-sync`: procesa las reuniones de hoy que ya terminaron.
- `/outlook-hubspot-sync <texto>`: procesa solo las reuniones cuyo asunto o dominio coincide con el texto.

El asistente guarda localmente en `state/processed.json` los eventos ya procesados (también en `.gitignore`).

## Rutina diaria en la nube

1. En https://claude.ai/code/routines crear una rutina con cron `0 20 * * *` (17:00 ART), modelo Sonnet, y los conectores **Microsoft 365**, **HubSpot**, **Apollo** y **Slack**.
2. Pegar como prompt el contenido de `routine/prompt.md` (desde la línea `---`), con los marcadores reemplazados por los valores de `config.local.md`.
3. Probarla con **Run now** un día con reuniones ya terminadas y revisar el informe en Slack.

La rutina no ve los archivos del repo. Si cambia una regla en `CLAUDE.md`, hay que actualizar también `routine/prompt.md` y la rutina.
