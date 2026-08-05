# AgendaBot

Bot conversacional en Telegram para agendar citas, gestionar tareas, hábitos, recordatorios y listas, construido con **n8n Community Edition** y **Google Sheets** como base de datos. Sin costos de plataforma ni APIs que requieran tarjeta de crédito.

## Stack Tecnológico

- **Telegram** — interfaz conversacional (Bot API)
- **n8n Community Edition** — automatización, lógica de negocio y router
- **Google Sheets** — almacenamiento de datos (hoja `AgendaBot_DB`)

## Estructura del Repositorio

```
Proyecto_IA1_ApellidoNombre/
├── README.md
├── docs/
│   └── AgendaBot.md        # Documentación técnica detallada
├── workflows/
│   └── AgendaBot_Modular_CRUD_Completo.json
└── evidencias/
    ├── 00 - Bienvenida/
    ├── 01 - Agenda/
    ├── 02 - Tareas/
    ├── 03 - Recordatorios/
    ├── 04 - Habitos/
    ├── 05 - Listas/
    ├── 06 - Reportes/
    ├── 07 - Configuracion/
    ├── 08 - Administrador/
    ├── 09 - Errores/
    ├── 10 - Logs/
    └── 11 - Productividad Usuario/
```

## Funcionalidades

![Menú de Bienvenida](evidencias/00%20-%20Bienvenida/00%20-%20menu_bienvenida.png)

- **Agenda (Citas):** crear, consultar, reprogramar, cancelar y completar citas. Valida fecha/hora, no permite agendar en el pasado y evita doble reserva (si el usuario ya tiene una cita activa en la misma fecha/hora, el bot avisa y no la guarda).
  <br>![Menú Agenda](evidencias/01%20-%20Agenda/00%20-%20Menu%20Agenda.png)
- **Tareas:** crear, consultar y cambiar estado (pendiente, en progreso, completada, cancelada).
  <br>![Menú Tareas](evidencias/02%20-%20Tareas/00%20-%20Menu%20Tareas.png)
- **Hábitos:** crear, consultar, activar/desactivar con recordatorio diario.
  <br>![Crear Hábito](evidencias/04%20-%20Habitos/01%20-%20Crear%20Habito.png)
- **Recordatorios:** crear y consultar recordatorios puntuales.
- **Listas:** crear una lista y agregar varios ítems en un solo flujo continuo; consultar listas e ítems; marcar ítems como completados.
  <br>![Crear Lista](evidencias/05%20-%20Listas/01%20-%20Crear%20Lista%20(1).png)
- **Reportes:** menú con 6 opciones de análisis (ver detalle abajo).
- **Configuración:** cambiar nombre de usuario y horario preferido de recordatorios.
- **Administrador:** ver usuarios, registrar nuevos usuarios, permitir/bloquear acceso por rol.
- **Resumen diario automático:** cada usuario recibe, a la hora que configuró, un resumen de sus citas y tareas pendientes del día (vía trigger de Cron, sin intervención manual).
- **Logs:** cada interacción queda registrada (usuario, pantalla, opción elegida, resultado).

### Menú de Reportes

Al escribir `6` desde el menú principal se despliegan 6 opciones:

```
📊 Vamos con tus reportes.

¿Qué quieres consultar?

1. Reporte diario
2. Reporte semanal
3. Productividad (tareas)
4. Hábitos (cumplimiento)
5. Agenda (estado de citas)
6. Productividad por usuario (reporte completo)
9. Volver al menú principal
```

1. **Reporte diario / Reporte semanal:** listado de las citas del usuario en ese rango de fechas.
2. **Productividad (tareas):** conteo de tareas del usuario por estado (completadas, en progreso, pendientes, canceladas) y por prioridad.
3. **Hábitos (cumplimiento):** listado de los hábitos del usuario con su estado (activo/inactivo).
4. **Agenda (estado de citas):** conteo de citas del usuario agrupadas por estado.
5. **Productividad por usuario (reporte completo):** el único reporte que cruza datos de **todos** los usuarios del bot (no solo el que pregunta). Ver detalle abajo.

![Menú de Reportes](evidencias/06%20-%20Reportes/00%20-%20Menu%20Reportes%20v3.png)

### Reporte de Productividad por Usuario (opción 6)

Esta opción lee las hojas `Citas`, `Tareas` y `Logs`, agrupa la información por `telegram_user` y genera un reporte con:

**Por cada usuario:**
- `total_citas`, `citas_completadas`, `citas_canceladas`
- `total_tareas`, `tareas_completadas`, `tareas_pendientes`
- Número de interacciones registradas en `Logs`

**Resumen general:**
- Usuario más activo (el de más interacciones)
- Total de citas registradas
- Total de tareas registradas
- Total de interacciones con el bot

**Reglas aplicadas:**
- Un usuario sin citas o sin tareas igual aparece en el listado, con esos valores en **0** (no se omite).
- Una fila de `Citas`, `Tareas` o `Logs` se considera **incompleta y se ignora** únicamente si le falta el campo de agrupación (`creado_por` o `telegram_user` vacío). No se descartan filas por otros campos vacíos.
- El listado se ordena **alfabéticamente** por el identificador del usuario.
- Como Telegram no comparte el `@username` real a través de la API que usa el bot (solo entrega el ID numérico), el alias que se muestra (`@nombre`) se arma a partir del campo `nombre` que el usuario tiene registrado en la hoja `Usuarios`, en minúsculas y sin espacios. Si el usuario no tiene `nombre` registrado, se muestra `@` + su ID numérico como respaldo.

Ejemplo de salida:

```
📊 Reporte de productividad (AgendaBot)

Resumen general
- Usuario más activo: @ana
- Total citas registradas: 12
- Total tareas registradas: 20
- Total interacciones con el bot: 65

Detalle por usuario
1) @ana
   - Citas: 2 (Completadas: 1 | Canceladas: 0)
   - Tareas: 5 (Hechas: 2 | Pendientes: 3)
   - Interacciones: 8

2) @pedrofp
   - Citas: 5 (Completadas: 3 | Canceladas: 1)
   - Tareas: 7 (Hechas: 4 | Pendientes: 3)
   - Interacciones: 18

¿Qué deseas hacer ahora?
1. Volver al menú Reportes
2. Volver al menú principal
```

![Reporte de Productividad](evidencias/06%20-%20Reportes/03%20-%20Reporte%20completo.png)

## Modelo de Datos (Google Sheets — hoja `AgendaBot_DB`)

| Hoja | Columnas |
|---|---|
| `Citas` | id_cita, fecha, hora, nombre, motivo, canal, estado, creado_por, timestamp_creacion |
| `Tareas` | id_tarea, titulo, prioridad, estado, fecha_objetivo, creado_por |
| `Habitos` | id_habito, nombre, frecuencia, hora_recordatorio, estado |
| `Recordatorios` | id_recordatorio, mensaje, fecha, hora, creado_por |
| `Listas` | id_lista, nombre_lista, tipo, creado_por |
| `Items_Lista` | id_item, id_lista, item, estado |
| `Usuarios` | telegram_user, nombre, rol, permitido, horario_recordatorios |
| `Logs` | timestamp, telegram_user, pantalla, opcion_elegida, resultado |
| `Sessions` | telegram_user, pantalla_actual, paso_actual, datos_parciales, timestamp_ultima_interaccion |

Más detalle de la arquitectura y las decisiones de diseño en [`docs/AgendaBot.md`](docs/AgendaBot.md).

## Cómo correrlo

1. Importar `workflows/AgendaBot_Modular_CRUD_Completo.json` en una instancia de n8n (Community Edition o n8n Cloud).
2. Crear una copia de la hoja de cálculo con las pestañas y columnas descritas arriba, nombrada `AgendaBot_DB`.
3. Conectar las credenciales de:
   - **Telegram** (Bot API, creado vía BotFather) en todos los nodos de tipo Telegram.
   - **Google Sheets OAuth2** en todos los nodos de tipo Google Sheets.
4. Agregar al menos un usuario administrador en la hoja `Usuarios` (`permitido = TRUE`, `rol = admin`).
5. Activar (Publish) el workflow.
6. Escribir `/start` al bot en Telegram.

## Control de Acceso

Solo los usuarios registrados en la hoja `Usuarios` con `permitido = TRUE` pueden usar el bot. El rol `admin` (o cualquier variante que empiece por "admin") desbloquea el menú de Administrador.

![Sin permiso](evidencias/09%20-%20Errores/00%20-%20Sin%20permiso.png)
![Menú Administrador](evidencias/08%20-%20Administrador/00%20-%20Menu%20Administrador.png)

## Autor

Miguel Alejandro Acevedo