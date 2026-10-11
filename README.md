# Cherry Nails 💅

Web y sistema de turnos online de **Cherry Nails**, un estudio de uñas en Corrientes.

Las clientas ven los servicios y precios, eligen día y horario, y reservan su turno desde el celular en menos de un minuto. La administradora recibe el aviso por WhatsApp y gestiona todo desde un panel: confirma turnos, carga turnos a mano, edita servicios, precios y horarios, y manda recordatorios.

Está pensado **mobile-first**: la mayoría de las clientas llega desde Instagram o WhatsApp.

---

## Índice

- [Qué incluye](#qué-incluye)
- [Tecnología](#tecnología)
- [Cómo reserva una clienta](#cómo-reserva-una-clienta)
- [Cómo se calculan los horarios libres](#cómo-se-calculan-los-horarios-libres)
- [Estados de un turno](#estados-de-un-turno)
- [Panel de administración](#panel-de-administración)
- [Mensajes de WhatsApp](#mensajes-de-whatsapp)
- [Seguridad](#seguridad)
- [Instalación y uso local](#instalación-y-uso-local)
- [Configuración inicial](#configuración-inicial)
- [Fotos](#fotos)
- [Estructura del proyecto](#estructura-del-proyecto)
- [API](#api)
- [Publicarlo](#publicarlo)
- [Limitaciones](#limitaciones)

---

## Qué incluye

| Parte | Dirección | Para quién | Qué hace |
|---|---|---|---|
| **Web pública** | `/` | Clientas | Presentación, servicios con precio y duración, carrusel de trabajos, contacto y ubicación. Barra inferior fija: Inicio · Servicios · Trabajos · Turnos · WhatsApp. |
| **Reserva de turno** | `/reservar` | Clientas | Servicio → fecha → horario disponible → nombre y WhatsApp → confirmación. |
| **Panel** | `/admin/` | Administradora | Dashboard del día, turnos próximos y pendientes, calendario, servicios y precios, horarios, días bloqueados, configuración y mensajes de WhatsApp. |

## Tecnología

- **Node.js** 22.13 o superior.
- **Express 5**: servidor web y API. Es la única dependencia.
- **SQLite**: usa el módulo `node:sqlite` que ya viene con Node, así que no hay que instalar una base de datos aparte. Todos los datos quedan en un solo archivo (`data/mercenails.db`).
- **Frontend en HTML, CSS y JavaScript puros**, sin frameworks ni paso de compilación. Express sirve directamente la carpeta `public/`.

## Cómo reserva una clienta

1. **Elige un servicio.** Cada servicio tiene precio y **duración**. La duración define cuánto tiempo ocupa el turno.
2. **Elige un día.** Se muestran los próximos días (30 por defecto). Los días sin lugar para ese servicio aparecen deshabilitados.
3. **Elige un horario.** Solo se muestran los horarios que están realmente libres.
4. **Carga su nombre y su WhatsApp** y confirma.
5. El servidor **vuelve a verificar** que el horario siga libre, por si otra clienta lo tomó mientras tanto. Si está libre, **guarda el turno en la base de datos** con estado *pendiente*.
6. Recién después se abre WhatsApp con el mensaje armado para el negocio:
   ```
   💅 Nuevo turno — Cherry Nails
   Cliente: Camila Rodríguez
   Servicio: Capping
   Fecha: 05/10/2026
   Hora: 17:00
   WhatsApp: 3794123456
   Turno N° 12
   ```
7. El turno aparece en el panel como *pendiente* hasta que la administradora lo confirma.

> **WhatsApp es el aviso, no el registro.** Si la clienta no llega a mandar el mensaje, el turno ya está guardado y el horario queda ocupado.

Para evitar abusos, cada persona puede hacer como máximo **6 reservas cada 15 minutos**.

## Cómo se calculan los horarios libres

La lógica está en `src/slots.js`. No se trabaja con franjas continuas sino con **horarios fijos de inicio** para cada día de la semana. Estos son los de ejemplo, y se editan desde el panel:

| Día | Horarios |
|---|---|
| Lunes a viernes | 14:00 · 17:00 · 20:00 |
| Sábado | 10:00 · 14:00 · 17:00 · 20:00 |
| Domingo | Sin atención |

Un horario se ofrece a la clienta solo si cumple todo esto:

- **El día no está bloqueado** (feriados, vacaciones).
- **Respeta la anticipación mínima**: 2 horas por defecto, para que nadie reserve para dentro de 10 minutos.
- **No supera el máximo de días hacia adelante** (30 por defecto).
- **No se superpone con otro turno** *pendiente* o *confirmado*, teniendo en cuenta la duración de los dos servicios. Por ejemplo, un Soft Acrílico de 150 min a las 14:00 ocupa hasta las 16:30, así que el horario de las 17:00 sigue libre.

Los turnos *cancelados* o *realizados* no ocupan lugar. Todo se calcula en **hora de Argentina** (`America/Argentina/Buenos_Aires`), sin importar dónde esté alojado el servidor.

## Estados de un turno

```
pendiente ──► confirmado ──► realizado
    │              │
    └──────────────┴──► cancelado
```

| Estado | Significado |
|---|---|
| **Pendiente** | La clienta reservó desde la web y falta que el negocio lo confirme. |
| **Confirmado** | El negocio lo aceptó. Los turnos cargados a mano entran directamente así. |
| **Realizado** | La clienta ya vino. |
| **Cancelado** | Se canceló y el horario vuelve a quedar libre. |

Cada turno guarda **una copia del nombre, el precio y la duración del servicio** al momento de reservar. Si después cambiás un precio, los turnos viejos no se modifican.

## Panel de administración

Se entra en `/admin/` con usuario y contraseña. Secciones:

- **Dashboard**: saludo, cantidad de turnos de hoy (total, pendientes y confirmados), aviso de turnos por confirmar, agenda del día y próximos turnos.
- **Turnos de hoy**: la agenda del día.
- **Próximos turnos**: filtros por Activos, Pendientes, Confirmados, Cancelados e Historial (últimos 90 días).
- **Turnos pendientes**: los que faltan confirmar. El menú muestra cuántos hay.
- **Calendario**: turnos por fecha.
- **Servicios**: crear, editar, ordenar, desactivar o borrar servicios (nombre, descripción, precio, duración e ícono).
- **Horarios disponibles**: los horarios de inicio de cada día de la semana y los **días bloqueados**, con un motivo opcional.
- **Configuración**: nombre del negocio, tu nombre (el del saludo del panel), usuario, WhatsApp del negocio, dirección, link de Google Maps, Instagram, anticipación mínima, días hacia adelante y cambio de contraseña.
- **WhatsApp**: plantillas de los mensajes de confirmación y de recordatorio, con vista previa.

Al abrir un turno se puede:

- **Confirmarlo y avisar por WhatsApp**: abre el chat con la clienta y el mensaje de confirmación listo.
- **Mandar un recordatorio** por WhatsApp.
- **Reprogramarlo**: solo ofrece horarios libres.
- Marcarlo como realizado o cancelado.
- Agregar notas internas.

**Nuevo turno manual**: sirve para clientas que escriben por Instagram, por teléfono o en persona. Entra como *confirmado* y no tiene el límite de anticipación ni de días hacia adelante. Igual controla que no se superponga con otro turno.

## Mensajes de WhatsApp

Los mensajes se arman con plantillas editables desde el panel. Variables disponibles:

| Variable | Ejemplo |
|---|---|
| `{nombre}` | Camila |
| `{servicio}` | Capping |
| `{dia}` | jueves |
| `{fecha}` | 05/10/2026 |
| `{hora}` | 17:00 |

Los números de teléfono se normalizan al formato argentino para `wa.me` (`549` + característica + número). Las clientas pueden escribirlos con o sin `0`, `15` o `+54`.

## Seguridad

- **Contraseña** guardada con hash *scrypt* y sal aleatoria. Nunca se guarda en texto plano.
- **Sesión** con cookie `HttpOnly` y `SameSite=Strict` que dura 30 días. En producción la cookie también es `Secure`.
- **Límite de intentos de login**: 10 cada 15 minutos.
- **Validación en el servidor** de todo lo que llega: fechas, horarios, teléfonos y largo de los textos.
- **Encabezados de seguridad** (`nosniff`, `X-Frame-Options`, `Referrer-Policy`), y la API del panel no se cachea.

## Instalación y uso local

Requisito: **Node.js 22.13 o superior**.

```bash
npm install
cp .env.example .env      # y poné una ADMIN_PASSWORD
npm start                 # o npm run dev para que se reinicie solo con cada cambio
```

- Web: http://localhost:3000
- Panel: http://localhost:3000/admin/ (usuario `admin` y la contraseña de `ADMIN_PASSWORD`)

### Variables de entorno (`.env`)

| Variable | Para qué sirve |
|---|---|
| `PORT` | Puerto del servidor (por defecto `3000`). |
| `ADMIN_PASSWORD` | Contraseña inicial del panel. Solo se usa la primera vez; si falta, se genera una y se muestra en consola. |
| `DATA_DIR` | Carpeta de la base de datos (por defecto `./data`). |
| `NODE_ENV` | En `production` activa la cookie segura y el soporte para proxy HTTPS. |

### Olvidé la contraseña

```bash
npm run set-password -- "nueva-clave"
```

## Configuración inicial

La primera vez que arranca, se cargan servicios, precios y horarios de ejemplo:

| Servicio | Precio | Duración |
|---|---|---|
| Soft Gel | $18.000 | 120 min |
| Soft Acrílico | $20.000 | 150 min |
| Capping | $15.000 | 90 min |
| Semipermanente | $10.000 | 60 min |
| Dipping | $16.000 | 90 min |
| Nail Art (por uña) | $1.500 | 30 min |

**Revisalos y editalos desde el panel** antes de publicar, junto con los horarios, el número de WhatsApp, la dirección y el Instagram en Configuración.

## Fotos

- **Carrusel "Mis trabajos"**: las fotos están en `public/img/trabajos/`. Para agregar o cambiar fotos, ponelas en esa carpeta y sumalas a la lista `GALERIA` en `public/script.js`.
- **Portada**: `public/img/hero.jpg`.

## Estructura del proyecto

```
server.js              servidor Express (seguridad, rutas, archivos estáticos)
src/db.js              base de datos: tablas, configuración y datos iniciales
src/slots.js           cálculo de horarios libres (hora de Argentina)
src/auth.js            login, contraseña (scrypt) y sesiones con cookie
src/whatsapp.js        links wa.me y mensaje de nuevo turno
src/util.js            validaciones, errores y límite de pedidos
src/routes/public.js   API pública: servicios, días, horarios, crear turno
src/routes/admin.js    API del panel
scripts/set-password.js  cambiar la contraseña desde la terminal
public/                web de clientas (index.html), reserva (reservar.html) y panel (admin/)
data/                  base de datos (se crea sola y no se sube a git)
```

### Base de datos

| Tabla | Contenido |
|---|---|
| `services` | Servicios: nombre, descripción, precio, duración, ícono, activo y orden. |
| `appointments` | Turnos: servicio, precio, duración, fecha, hora, clienta, teléfono, estado, notas y origen (`web` o `admin`). |
| `schedule_slots` | Horarios de inicio por día de la semana. |
| `blocked_dates` | Días sin atención, con motivo. |
| `settings` | Configuración del negocio, plantillas de WhatsApp y hash de la contraseña. |
| `sessions` | Sesiones activas del panel. |

## API

**Pública** (`/api`):

| Método | Ruta | Descripción |
|---|---|---|
| GET | `/config` | Nombre, WhatsApp, dirección, mapa e Instagram del negocio. |
| GET | `/services` | Servicios activos. |
| GET | `/days?service_id=` | Próximos días y si hay lugar para ese servicio. |
| GET | `/slots?service_id=&date=` | Horarios libres de un día. |
| POST | `/appointments` | Crear un turno. Devuelve el turno y el link de WhatsApp. |

**Panel** (`/api/admin`, requiere sesión salvo el login):

| Método | Ruta | Descripción |
|---|---|---|
| POST | `/login` · `/logout` | Iniciar y cerrar sesión. |
| GET | `/me` | Datos de la administradora y fecha de hoy. |
| PUT | `/password` | Cambiar la contraseña. |
| GET · POST | `/appointments` | Listar turnos (filtros `from`, `to` y `status`) y crear un turno manual. |
| PATCH | `/appointments/:id` | Cambiar estado, notas, fecha u hora. |
| GET | `/slots` | Horarios libres (sin límites de anticipación; `exclude` para reprogramar). |
| GET · POST · PUT · DELETE | `/services` | Gestionar servicios. |
| GET · PUT | `/schedule` | Horarios semanales y días bloqueados. |
| POST · DELETE | `/blocked` | Bloquear y desbloquear días. |
| GET · PUT | `/settings` | Configuración del negocio. |

## Publicarlo

Necesita un hosting que corra Node y tenga **disco persistente** para la carpeta `data/`, por ejemplo Railway o Render con disco, o un VPS. No sirve un hosting solo estático (Netlify, GitHub Pages) ni uno sin disco persistente, porque se perderían los turnos.

- Configurá `NODE_ENV=production`, `ADMIN_PASSWORD` y, si el disco se monta en otra ruta, `DATA_DIR`.
- Usá **HTTPS**.
- **Backup**: todo está en un solo archivo, `data/mercenails.db`. Conviene copiarlo periódicamente.

## Limitaciones

- **Los WhatsApp no se envían solos.** Se abre un link `wa.me` y la persona toca "enviar". Para el envío automático hace falta la API de WhatsApp Business (Meta Cloud API), que es paga y requiere verificar el negocio.
- **Los recordatorios son manuales.** Se mandan desde cada turno en el panel.
- **Hay una sola cuenta de administración.**
- **Pagos y señas** no están integrados.
