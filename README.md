# Calendar → CSV

App web que migra **todos** tus eventos de Google Calendar a un archivo **CSV** listo
para importar a tu base de datos. Todo el procesamiento ocurre en tu navegador: tus
datos nunca se suben a ningún servidor.

Dos formas de cargar los datos, en la misma app:

- **A) Archivo `.ics` / `.zip`** — sin configuración, funciona offline. (Recomendado)
- **B) Conexión en vivo con Google (OAuth)** — baja los eventos directo por la API.

El CSV usa fechas **ISO 8601** y separador **coma** por defecto, y un **selector de
columnas** para que elijas exactamente qué campos de cada evento incluir.

---

## Uso rápido — método A (.ics), sin configuración

1. Abrí `index.html` haciendo doble clic (o arrastralo al navegador).
2. En `calendar.google.com` → **⚙ Configuración → Importar y exportar → Exportar**.
   Se descarga un `.zip` con un `.ics` por cada calendario.
3. Arrastrá ese `.zip` (o los `.ics`) al recuadro de la app.
4. Elegí las columnas → **Descargar CSV**.

Eso es todo. No hace falta cuenta de Google Cloud ni servidor.

---

## Uso — método B (conexión OAuth en vivo)

La conexión OAuth necesita que la página se sirva en `http://localhost` (no funciona
con `file://`). Es una limitación de seguridad de Google, no de la app.

### 1. Servir la página localmente

Con **Python** (viene en muchas máquinas):

```powershell
cd C:\FL\calendar-to-csv
python -m http.server 8000
```

O con **Node**:

```powershell
cd C:\FL\calendar-to-csv
npx http-server -p 8000
```

Luego abrí `http://localhost:8000` en el navegador.

### 2. Crear el OAuth Client ID (configuración única, ~10 min)

1. Entrá a `console.cloud.google.com` y creá/elegí un proyecto.
2. **APIs y servicios → Biblioteca** → buscá **Google Calendar API** → **Habilitar**.
3. **Pantalla de consentimiento OAuth** → tipo **Externo**. Completá nombre y tu email.
   En **Usuarios de prueba** agregá tu propia cuenta de Google.
4. **Credenciales → Crear credenciales → ID de cliente de OAuth** → **Aplicación web**.
5. En **Orígenes autorizados de JavaScript** agregá exactamente: `http://localhost:8000`
   (debe coincidir con el puerto que usás al servir).
6. Copiá el **Client ID** (termina en `.apps.googleusercontent.com`).

### 3. Conectar

En la pestaña **Conectar con Google**, pegá el Client ID, tocá **Conectar**, autorizá
en la ventana de Google y la app baja todos los eventos de todos tus calendarios.

---

## Columnas del CSV

Se incluyen sólo las columnas que tengan datos. Las que podés elegir:

| Columna        | Descripción                                             |
|----------------|---------------------------------------------------------|
| `calendar`     | Nombre del calendario de origen                         |
| `title`        | Título del evento (SUMMARY)                             |
| `start`        | Inicio en ISO 8601 (`2026-06-08T14:30:00`)              |
| `end`          | Fin en ISO 8601                                         |
| `all_day`      | `true`/`false` si es de día completo                    |
| `timezone`     | Zona horaria del evento (`America/Argentina/Buenos_Aires`, `UTC`…) |
| `location`     | Ubicación                                               |
| `description`  | Descripción / notas                                     |
| `status`       | `confirmed` / `tentative` / `cancelled`                 |
| `visibility`   | `public` / `private` / `default`                        |
| `transparency` | `opaque` (ocupado) / `transparent` (libre)              |
| `organizer`    | Organizador                                             |
| `attendees`    | Invitados (separados por `; `)                          |
| `recurrence`   | Regla de repetición (RRULE/EXDATE…)                     |
| `categories`   | Categorías/etiquetas                                    |
| `created`      | Fecha de creación                                       |
| `updated`      | Última modificación                                     |
| `uid`          | Identificador único del evento                          |
| `html_link`    | Enlace al evento (sólo método Google)                   |
| `color`        | Color/colorId                                           |
| `source`       | `ics` o `google`                                        |

---

## Notas sobre los datos

- **Zonas horarias:** para no alterar tus datos, el inicio/fin se guardan como la
  hora "de pared" original y la zona va aparte en la columna `timezone`. En tu base
  podés combinar `start` + `timezone` en un `timestamptz`. Las horas marcadas con
  `UTC` ya están en UTC.
- **Eventos recurrentes:**
  - Método **Google (OAuth):** se expanden en instancias individuales
    (`singleEvents=true`), una fila por ocurrencia.
  - Método **.ics:** se guarda el evento maestro con su `recurrence` (RRULE). No se
    expanden las ocurrencias.
- **Eventos de día completo (.ics):** el `DTEND` de Google es *exclusivo* (el día
  siguiente). Se exporta tal cual viene en el archivo.

---

## Importar a tu base de datos (ejemplo PostgreSQL)

```sql
CREATE TABLE eventos (
  calendar text, title text, start_at text, end_at text,
  all_day boolean, timezone text, location text, description text
);

\copy eventos FROM 'google-calendar-export.csv' WITH (FORMAT csv, HEADER true);
```

Si exportaste con **BOM UTF-8** activado (útil para Excel) y tu importador se queja,
volvé a exportar destildando esa opción.
