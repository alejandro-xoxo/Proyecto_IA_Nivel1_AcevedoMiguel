# Documentación Técnica — AgendaBot

## 1. Visión General

AgendaBot es un bot conversacional que corre sobre **n8n Community Edition**, recibe mensajes vía **Telegram Bot API** y persiste todos los datos en una hoja de cálculo de **Google Sheets** (`AgendaBot_DB`). No depende de ninguna plataforma de pago, API con tarjeta de crédito, ni modelos de IA entrenados/embeddings/RAG.

## 2. Arquitectura del Workflow

### 2.1 Flujo de entrada

```
Recibir Mensaje de Telegram (Entrada)
  → Extraer Datos del Mensaje (Entrada)
  → Leer Sesión Guardada (Sesión)
  → Armar Objeto de Sesión (Sesión)
  → Leer Usuarios (Permisos)
  → Verificar Permiso Usuario (Permisos)
  → Procesar Lógica y Generar Respuesta (Núcleo)
```

El nodo **Procesar Lógica y Generar Respuesta (Núcleo)** es el corazón del bot: un único nodo Code (JavaScript) que implementa una máquina de estados mediante `switch (pantalla)`, donde `pantalla` representa la "ubicación" del usuario dentro de los menús (ej. `MENU_PRINCIPAL`, `CITA_WIZARD`, `MENU_LISTAS`, etc.) y `paso` representa en qué punto de un wizard multi-paso se encuentra.

Este nodo, en paralelo, dispara tres cosas:
1. **Guardar Sesión Actualizada** — persiste `pantalla_actual`, `paso_actual` y `datos_parciales` (JSON serializado) en la hoja `Sessions`.
2. **Registrar Log de Interacción** — guarda cada interacción en la hoja `Logs`.
3. **Enviar Respuesta al Usuario** — responde inmediatamente al usuario por Telegram con el texto generado.
4. **Enrutar Acción por Módulo** — un nodo Switch con ~25 ramas (una por cada `accion` posible: `GUARDAR_CITA`, `VER_TAREAS`, `ACTUALIZAR_HABITO`, etc.) que ejecuta en segundo plano las operaciones reales sobre Google Sheets.

### 2.2 Patrón de "acción diferida"

Como el nodo Core no puede hacer llamadas a Google Sheets de forma síncrona dentro de su propio código, cada vez que necesita persistir o consultar datos, retorna un campo `accion` (ej. `"GUARDAR_TAREA"`). El Switch enruta esa acción hacia una cadena de nodos: `Preparar Datos → Google Sheets (append/update/read) → [Filtrar/Formatear] → [Enviar mensaje Telegram]`.

Esto separa claramente:
- **Lo que el usuario ve de inmediato** (generado por el nodo Core, enviado por "Enviar Respuesta al Usuario").
- **Lo que pasa en segundo plano** (guardar en Sheets, o —para consultas— leer y mandar un segundo mensaje con los resultados).

### 2.3 Control de acceso (Verificar Permiso Usuario)

Antes de que cualquier mensaje llegue al núcleo de lógica, se lee la hoja `Usuarios` completa y se busca la fila cuyo `telegram_user` coincida con el remitente. Si no existe la fila, o `permitido` no es `TRUE`, el flujo corta ahí mismo con un mensaje de "Acceso restringido" y nunca llega a ejecutar ningún menú.

El campo `rol` se normaliza a minúsculas y se acepta como administrador cualquier valor que **empiece por** `"admin"` (cubre `admin`, `Admin`, `Administrador`, `ADMINISTRADOR`, etc.), evitando fallos por diferencias de mayúsculas/minúsculas.

### 2.4 Validación de doble reserva

Al confirmar una cita nueva (paso final del wizard de Agenda), el núcleo **no guarda directamente**: dispara la acción `VERIFICAR_Y_GUARDAR_CITA`, que:
1. Lee todas las citas existentes.
2. Busca si el mismo usuario ya tiene una cita activa (`estado != cancelada`) en la misma fecha y hora.
3. Si hay conflicto → envía un aviso y **no guarda** la nueva cita.
4. Si no hay conflicto → guarda la cita y confirma por Telegram.

### 2.5 Resumen diario automático

Un **Schedule Trigger** independiente corre cada hora. En cada ejecución:
1. Lee todos los usuarios permitidos.
2. Para cada uno, compara su `horario_recordatorios` configurado contra la hora actual.
3. Si coincide, calcula sus citas de hoy y tareas pendientes.
4. Si tiene algo pendiente, le envía un resumen por Telegram; si no tiene nada, no le envía nada (para no generar ruido innecesario).

## 3. Modelo de Datos

Ver tabla completa en `README.md`. Notas relevantes:

- **`Sessions`**: una fila por usuario, se sobrescribe en cada interacción. Guarda el estado exacto de la conversación para que el bot recuerde en qué paso del wizard se quedó el usuario.
- **`Logs`**: append-only, nunca se borra ni actualiza, sirve como evidencia de auditoría.
- **`Usuarios.horario_recordatorios`**: formato `HH:00` (hora en punto), usado tanto por el resumen diario como por la opción de Configuración.

## 4. Principios Conversacionales Aplicados

- Todas las opciones se seleccionan escribiendo un número.
- Toda pantalla ofrece una opción de salida/cancelación (normalmente `9`).
- El bot siempre sugiere una opción recomendada en el menú principal.
- Ninguna opción inválida se procesa silenciosamente: siempre hay un mensaje de "opción no existe" que repite las opciones válidas.
- Toda acción de guardado pasa por un paso de confirmación explícita antes de escribir en Sheets.
- Los menús usan viñetas (`-`) en vez de listas numeradas de Markdown, para evitar que Telegram renumere automáticamente las opciones (por ejemplo, que la opción "9. Volver" aparezca visualmente como "6.").

## 5. Decisiones de Diseño

- **Un solo nodo Code central** en vez de un nodo por cada pantalla: facilita mantener toda la máquina de estados en un solo lugar, a costa de que el archivo de código sea largo. Se consideró aceptable dado el tamaño del proyecto.
- **Acciones diferidas vía Switch**: permite que el nodo Core siga siendo puramente síncrono (sin llamadas a APIs externas dentro de su lógica), delegando toda I/O a nodos nativos de n8n.
- **Reutilización de nodos de guardado**: por ejemplo, el nodo que guarda una cita nueva (`Guardar Cita en Sheet`) se reutiliza tanto en el flujo normal como en el flujo con verificación de disponibilidad, evitando duplicar la definición de columnas.

## 6. Limitaciones Conocidas

- El Schedule Trigger de resumen diario corre cada hora en punto; si el usuario configura un horario que no sea una hora exacta (ej. `07:30`), no lo recibirá correctamente — está pensado para horas en punto.
- La validación de doble reserva compara solo por el mismo usuario (`creado_por`), no bloquea si dos usuarios distintos agendan la misma hora (no aplica si el bot es de uso personal, pero sí importaría en un caso de agenda compartida).
- Las opciones "Reportes" (bajo demanda) y "Resumen diario" (automático) son mecanismos separados y complementarios, no deben confundirse.
