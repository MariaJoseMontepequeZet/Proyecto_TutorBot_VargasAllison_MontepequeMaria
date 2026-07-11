# TutorBot

**Sistema automatizado de coordinación de tutorías académicas**

Autoras: **Allison Vargas** y **Maria Jose Montepeque**

---

## Descripción del proyecto

TutorBot es un sistema de coordinación de tutorías académicas desarrollado como proyecto de clase utilizando **n8n** como plataforma de automatización de flujos de trabajo. El sistema permite a estudiantes solicitar, consultar y cancelar tutorías conversando por **Telegram** con un agente de inteligencia artificial, el cual gestiona automáticamente la disponibilidad de los tutores, el registro de tutorías, las notificaciones y los recordatorios, utilizando **Google Sheets** como base de datos.

> **Nota:** Este proyecto fue desarrollado con fines académicos y no está destinado a un entorno de producción.

---

## Arquitectura general

El workflow de n8n está dividido en dos sub-flujos:

### 1. Flujo conversacional (estudiante ⇄ AI Agent)

```
Telegram Trigger
  → Code - Marcar Duplicados
  → Filtrar Duplicados
  → AI Agent (+ Memoria de Conversación, + OpenAI Chat Model, + 12 Tools de Google Sheets/Telegram)
  → Code - Parsear Salida IA
  → Code in JavaScript1
  → HTTP Request1 (POST a la API de Telegram con teclado inline dinámico)
```

### 2. Flujo de recordatorios (disparado por Webhook externo, ej. Cron)

```
Webhook - Disparador de Recordatorios (path: /tutorbot-recordatorios)
  → Sheets - Tutorias del Dia
  → Filtrar Fecha Hoy
      → Telegram - Recordatorio Estudiante
      → Sheets - Buscar Chat Tutor → Telegram - Recordatorio Tutor
  → Respuesta Webhook
```

---

## Nodos utilizados

| Categoría | Nodos en el workflow |
|---|---|
| **Telegram** | `Telegram Trigger`, `Telegram - Recordatorio Tutor`, `Telegram - Recordatorio Estudiante`, `Tool - Notificar Tutor Asignacion` (envía mensajes desde el agente), `HTTP Request1` (POST directo a la Bot API para teclados inline dinámicos) |
| **Google Sheets** | `Sheets - Tutorias del Dia`, `Sheets - Buscar Chat Tutor`, y 9 Sheets Tool nodes conectados al AI Agent (ver tabla de herramientas) |
| **Modelo de lenguaje** | `OpenAI Chat Model1` (`gpt-4o-mini`), conectado al AI Agent como `ai_languageModel` |
| **Chatbot / AI Agent** | `AI Agent` (LangChain Agent) — orquesta el wizard conversacional y todas las herramientas |
| **Memoria** | `Memoria de Conversación` (Window Buffer Memory), conectada al AI Agent como `ai_memory`, con `chat_id` como clave de sesión |
| **Webhook** | `Webhook - Disparador de Recordatorios` (dispara el flujo de recordatorios diarios) y `Respuesta Webhook` (respuesta final de ese flujo) |
| **Code (JavaScript)** | `Code - Marcar Duplicados`, `Filtrar Duplicados` (evitan procesar el mismo mensaje de Telegram dos veces), `Code - Parsear Salida IA` (parsea el JSON de salida del agente), `Code in JavaScript1` (arma el payload para el envío a Telegram) |

---

## Herramientas (tools) conectadas al AI Agent

El agente tiene acceso a 12 herramientas para interactuar con la base de datos y con Telegram:

| Herramienta | Propósito |
|---|---|
| `Tool - Buscar o Registrar Estudiante` | Resuelve el `chat_id` de Telegram al `id_estudiante` real (formato `ESTxxxx`); si no existe, lo crea |
| `Tool - Buscar Materias` | Lista las materias disponibles según los tutores activos |
| `Tool - Buscar Tutor Disponible` | Busca un tutor libre para una materia y día específicos |
| `Tool - Obtener Siguiente ID Tutoria` | Calcula el siguiente `id_tutoria` consecutivo (formato `TUTxxxxxxx`) |
| `Tool - Registrar Tutoria` | Crea el registro de la tutoría en la hoja TUTORIAS |
| `Tool - Bloquear Disponibilidad` | Marca el horario del tutor como "Ocupado" tras agendar |
| `Tool - Buscar Telegram Tutor` | Obtiene el `telegram_user` del tutor asignado |
| `Tool - Notificar Tutor Asignacion` | Envía al tutor un mensaje de Telegram avisando la nueva asignación |
| `Tool - Consultar Estado` | Devuelve las tutorías del estudiante, incluyendo su `id_tutoria` exacto |
| `Tool - Cancelar Tutoria` | Marca una tutoría como "Cancelada" |
| `Tool - Liberar Disponibilidad` | Regresa el horario del tutor a "Libre" tras una cancelación |
| `Tool - Registrar Sesion` | Guarda trazabilidad del paso actual del wizard por conversación |

---

## Lógica del wizard conversacional

El `AI Agent` sigue un flujo guiado, un paso a la vez, sin inventar datos:

1. **Menú principal**: solicitar tutoría, consultar estado o cancelar tutoría.
2. **Solicitar tutoría**: elegir materia → indicar fecha → el agente busca tutor disponible ese día → el usuario confirma → se registra la tutoría, se bloquea el horario y se notifica al tutor por Telegram.
3. **Consultar estado**: muestra las tutorías del estudiante con su `id_tutoria` exacto.
4. **Cancelar tutoría**: cancela usando el `id_tutoria` exacto devuelto por la consulta, y libera el horario correspondiente.

La salida del agente siempre es un JSON estricto `{"mensaje": ..., "opciones": [...]}`, que luego se transforma en botones interactivos de Telegram mediante los nodos de código y el `HTTP Request1`.

---

## Base de datos (Google Sheets)

El sistema utiliza una hoja de cálculo estructurada en cinco hojas:

- **TUTORES** – Tutores registrados, con `telegram_user`, materias y estado (Activo/Inactivo).
- **DISPONIBILIDAD** – Bloques de horario por tutor, identificados con `id_dispo`, con estado Libre/Ocupado.
- **TUTORIAS** – Tutorías agendadas, identificadas con `id_tutoria`, vinculadas a `id_estudiante`, `id_tutor` e `id_dispo`.
- **ESTUDIANTES** – Estudiantes registrados, con `id_estudiante` y `telegram_user`.
- **SESSIONS** – Historial de pasos del wizard por conversación, para trazabilidad.

---

## Decisiones técnicas relevantes

- **AI Agent con múltiples tools (LangChain)** en lugar de un flujo de nodos condicionales tipo wizard clásico, lo que permite conversación en lenguaje natural sin perder control estricto del flujo (el system prompt fuerza el orden de pasos y prohíbe inventar datos).
- **Resolución de identidad**: el `chat_id` de Telegram nunca se usa directamente como `id_estudiante`; el agente siempre lo resuelve primero contra la hoja ESTUDIANTES.
- **Envío de teclados interactivos dinámicos** vía `HTTP Request1` haciendo POST directo a la API de Telegram (`sendMessage`), en lugar del nodo nativo de Telegram, que no soporta bien arreglos dinámicos de botones.
- **Deduplicación de actualizaciones de Telegram** (`Code - Marcar Duplicados` + `Filtrar Duplicados`) antes de llegar al AI Agent, para evitar procesar el mismo mensaje más de una vez.
- **Parseo de salida estructurada mediante Code node** (`Code - Parsear Salida IA`) en lugar del Structured Output Parser nativo, dado que la salida del agente debe ser JSON estricto sin backticks ni texto adicional.
- **Flujo de recordatorios independiente**, disparado por un Webhook externo (pensado para dispararse por Cron), que revisa las tutorías del día y notifica tanto a estudiantes como a tutores.

---

## Stack tecnológico

- **n8n** – Plataforma de automatización de flujos de trabajo
- **Telegram Bot API** – Interfaz de mensajería con estudiantes y tutores
- **Google Sheets API** – Base de datos del sistema
- **OpenAI `gpt-4o-mini`** – Modelo de lenguaje del AI Agent
- **LangChain AI Agent** (Window Buffer Memory, `chat_id` como clave de sesión)

---

## Entregables del proyecto

- Archivo JSON del workflow de n8n (`TutorBot.json`)
- Hoja de Google Sheets compartida (base de datos)
- Repositorio de GitHub con el workflow y este README

---

## Autoras

- **Allison Vargas**
- **Maria Jose Montepeque**