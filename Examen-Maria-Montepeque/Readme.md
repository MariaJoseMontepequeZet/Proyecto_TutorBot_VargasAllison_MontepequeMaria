# Resumen de Tutorías por Materia — Sub-flujo n8n

Sub-flujo para TutorBot que genera un reporte semanal de tutorías agrupadas por materia y lo envía por Telegram, restringido al usuario coordinador.

## Contenido

`resumen_tutorias_por_materia.json` — 7 nodos:

| Nodo | Función |
|---|---|
| Menú: Resumen Semanal | Punto de entrada (conectar desde el Switch de Coordinación) |
| Config Coordinador | Guarda el `telegram_user_id` autorizado |
| IF - Validar Coordinador | Compara el usuario que escribe contra el coordinador configurado |
| Google Sheets - Leer TUTORIAS | Lee todos los registros de la hoja `TUTORIAS` |
| Code - Agrupar por Materia | Filtra por semana actual (lunes–domingo) y cuenta por materia |
| Set - Mensaje Resumen | Arma el texto final con el formato acordado |
| Telegram - Enviar Resumen / Acceso Denegado | Responde al chat original según el resultado del IF |

## Requisitos previos

- Instancia n8n con credenciales de **Google Sheets (OAuth2)** y **Telegram API** ya configuradas.
- Hoja `TUTORIAS` con al menos las columnas `fecha` y `materia`.
- Flujo principal de TutorBot con un nodo Telegram Trigger ya existente (necesario para leer el chat/usuario, ver sección siguiente).

## Instalación

1. En n8n: **Import from File** → seleccionar `resumen_tutorias_por_materia.json`.
2. Reemplazar los siguientes placeholders:

   | Placeholder | Dónde | Reemplazar por |
   |---|---|---|
   | `PLACEHOLDER_COORDINADOR_TELEGRAM_ID` | Nodo `Config Coordinador` | ID numérico de Telegram del coordinador |
   | `PLACEHOLDER_SPREADSHEET_ID` | Nodo `Google Sheets - Leer TUTORIAS` | ID de tu Google Sheet |
   | `PLACEHOLDER_GOOGLE_SHEETS_CREDENTIAL_ID` | Mismo nodo | ID de tu credencial de Google Sheets en n8n |
   | `PLACEHOLDER_TELEGRAM_CREDENTIAL_ID` | Nodos `Telegram - Enviar Resumen` y `Telegram - Acceso Denegado` | ID de tu credencial de Telegram en n8n |

3. Ajustar las referencias a `'Telegram Trigger'` (ver siguiente sección) si tu nodo trigger tiene otro nombre.
4. Conectar la salida del botón **"📊 Resumen Semanal de Tutorías"** del menú de Coordinación al nodo **"Menú: Resumen Semanal"**, en lugar de dejarlo como entrada suelta.
5. Activar el flujo.

## Sobre la referencia al Telegram Trigger

Este sub-flujo no genera por sí solo el ID del usuario ni del chat — los toma del nodo Trigger de Telegram del flujo principal, usando:

- `$('Telegram Trigger').item.json.message.from.id` → para validar **quién** escribe (usado en el IF de Coordinador)
- `$('Telegram Trigger').item.json.message.chat.id` → para **responder** al chat correcto (usado en ambos nodos Telegram de salida)

Si tu nodo trigger principal tiene otro nombre, o si ya existe un nodo intermedio que extrae `chat_id`/`user_id` a un campo propio del wizard, reemplaza las 3 expresiones anteriores por la referencia correcta a ese nodo.

## Formato del mensaje final

```
📚 Resumen de Tutorías por Materia
----------------------------
🧮 Matemáticas: [Cantidad]
💻 Programación: [Cantidad]
🧪 Química: [Cantidad]
📈 TOTAL SEMANA: [Suma_Total]
```

## Notas y extensiones

- El Code node calcula la semana actual como lunes 00:00 a domingo 23:59, según la fecha del servidor donde corre n8n.
- Si `TUTORIAS` tiene materias fuera de las 3 contempladas en el formato fijo, quedan disponibles en `conteoCompleto` dentro del item de salida del nodo Code, por si luego quieres extender el mensaje.
- El nombre de la columna `fecha` está hardcodeado en el Code node — ajustarlo si en tu hoja se llama distinto.