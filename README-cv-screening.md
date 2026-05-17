# 🤖 Automated CV Screening Workflow

![n8n](https://img.shields.io/badge/n8n-workflow-orange?logo=n8n&logoColor=white)
![Claude](https://img.shields.io/badge/Claude-Sonnet_4.5-8B5CF6?logo=anthropic&logoColor=white)
![Airtable](https://img.shields.io/badge/Airtable-database-18BFFF?logo=airtable&logoColor=white)
![Slack](https://img.shields.io/badge/Slack-notifications-4A154B?logo=slack&logoColor=white)
![Gmail](https://img.shields.io/badge/Gmail-email-EA4335?logo=gmail&logoColor=white)
![License](https://img.shields.io/badge/license-All%20Rights%20Reserved-red)

Pipeline de screening de CVs construido en **n8n** que procesa postulaciones desde Airtable, analiza el CV con **Claude (Anthropic)**, notifica al recruiter por **Slack** y marca el registro como procesado automáticamente.

> 👤 **Creado por [Ana Ferreira](https://www.linkedin.com/in/anaferreirabezerra)** — HR Operations Analyst · People Analytics · HR Automation Specialist · 📩 [ani.fb95@gmail.com](mailto:ani.fb95@gmail.com) · [WhatsApp](https://wa.me/5491135077374)



---

## 📋 Descripción

Este workflow corre **cada minuto** y procesa todas las postulaciones nuevas en Airtable. Por cada candidato, descarga el CV, lo analiza con IA, genera un score y envía un resumen al canal de Slack del recruiter. Si la vacante está cerrada, notifica al candidato por email.

**Flujo principal:**
```
Trigger → Leer → Filtrar → Separar Jobs → Validar Vacante
  → [Activa]  → Analizar CV → Slack
  → [Cerrada] → Email al candidato
```

---

## 🗺️ Diagrama del Workflow

<img width="1568" height="696" alt="image" src="https://github.com/user-attachments/assets/7afccb2d-e7e5-4672-926f-3fb4901f8013" />

---

## ⚙️ Nodos del Workflow

| Nodo | Tipo | Descripción |
|------|------|-------------|
| **Disparador Cada Minuto** | Schedule Trigger | Dispara el workflow automáticamente cada 1 minuto |
| **Leer Postulaciones** | Airtable – Search Records | Trae todos los registros de la tabla `Applicants` con `Procesado = false` |
| **Nueva Postulacion** | IF / Filter | Si todas están procesadas, detiene el workflow |
| **Separar Jobs** | Split In Batches | Si el candidato aplicó a múltiples vacantes, crea un item por cada job para procesarlos individualmente |
| **Obtener Vacante** | Airtable – Get Record | Fetch del registro completo de la vacante en la tabla `Jobs` usando el linked record ID |
| **Descargar CV** | HTTP Request | Descarga el PDF del CV desde la URL del adjunto en Airtable (`continueOnFail` activo) |
| **¿Vacante Activa?** | IF | `Status = Open` → analizar CV / `Status ≠ Open` → email de vacante cerrada |
| **Extraer Texto CV** | Extract From PDF | Extrae el contenido de texto del PDF para enviarlo al prompt de IA |
| **Preparar Prompt** | Code / Set | Combina datos del candidato + criterios de la vacante y construye el prompt para Claude |
| **Claude API** | HTTP Request | POST a `api.anthropic.com/v1/messages` con el prompt. Usa `claude-sonnet-4-5` |
| **Parsear Respuesta Claude** | Code | Extrae y valida el JSON devuelto por Claude con el score, status y detalles del análisis. Reintenta si falla |
| **Preparar Mensaje Slack** | Set | Formatea el análisis con emojis, score, skills match y link directo al registro en Airtable |
| **Enviar Slack** | Slack – Post Message | Publica el análisis en el canal del recruiter definido en el campo `Slack_Channel` de la vacante |
| **Marcar Como Procesado** | Airtable – Update Record | PATCH a la API de Airtable seteando `Procesado = true` para evitar reprocesar el registro |
| **Preparar Email Vacante Cerrada** | Code / Set | Crea un email HTML notificando al candidato que la vacante no está activa |
| **Enviar Email Vacante Cerrada** | Gmail – Send Message | Envía el email al candidato via Gmail OAuth |
| **Marcar Procesado – Inactiva** | Airtable – Update Record | También marca `Procesado = true` para postulaciones a vacantes cerradas |

---

## 🗄️ Estructura de Airtable

### Tabla: `Jobs`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `Job_ID` | Single line text | Identificador único (ej: JOB001) |
| `Job` | Single line text | Nombre del puesto |
| `Department` | Single select | Engineering, Data, Design, Product |
| `Description` | Long text | Descripción del rol |
| `Required_Skills` | Long text | Skills requeridas |
| `Nice_to_Have` | Long text | Skills deseables |
| `Seniority` | Single select | Junior, Mid, Senior |
| `Min_Years_Experience` | Number | Años mínimos requeridos |
| `Location` | Single line text | Ubicación |
| `Employment_Type` | Single select | Full-time, Part-time, Contract |
| `Recruiter` | Single line text | Nombre del recruiter responsable |
| `Slack_Channel` | Single line text | Canal de Slack donde llegan las notificaciones |
| `Status` | Single select | Open, Closed, On Hold |

### Tabla: `Applicants`

| Campo | Tipo | Descripción |
|-------|------|-------------|
| `Applicant_ID` | Single line text | Identificador único |
| `Full_Name` | Single line text | Nombre completo |
| `Email` | Email | Email del candidato |
| `Phone` | Phone number | Teléfono |
| `Job` | Link → Jobs | Vacante a la que aplica |
| `CV` | Attachment | PDF del CV |
| `LinkedIn` | URL | Perfil de LinkedIn |
| `Years_Experience` | Number | Años de experiencia |
| `Current_Role` | Single line text | Puesto actual |
| `Location` | Single line text | Ubicación del candidato |
| `Source` | Single select | LinkedIn, Referral, Web, Other |
| `Stage` | Single select | New, Screening, Interview, Offer, Hired, Rejected |
| `AI_Score` | Number | Score asignado por Claude (0–100) |
| `AI_Notes` | Long text | Análisis detallado de Claude |
| `Applied_At` | Date | Fecha de postulación |
| `Procesado` | Checkbox | Flag para evitar reprocesar |

---

## 🔑 Credenciales requeridas

| Servicio | Tipo | Dónde configurar en n8n |
|----------|------|--------------------------|
| **Airtable** | Personal Access Token (PAT) | n8n → Credentials → Airtable |
| **Anthropic (Claude)** | API Key | n8n → Credentials → Header Auth (`x-api-key`) |
| **Slack** | Bot Token (`xoxb-...`) | n8n → Credentials → Slack |
| **Gmail** | OAuth2 | n8n → Credentials → Gmail OAuth2 |

> ⚠️ El API key de Anthropic es **independiente** de la cuenta Claude.ai. Se obtiene en [console.anthropic.com](https://console.anthropic.com).

### Permisos mínimos requeridos

**Airtable PAT scopes:**
- `data.records:read`
- `data.records:write`

**Slack Bot scopes:**
- `chat:write`
- `chat:write.public`

---

## 🚀 Setup

1. Importar el workflow JSON en n8n (`Settings → Import Workflow`)
2. Configurar las 4 credenciales listadas arriba
3. En el nodo **Leer Postulaciones**, actualizar el `Base ID` y el nombre de la tabla
4. En el nodo **Obtener Vacante**, actualizar el `Base ID` de la tabla `Jobs`
5. En el nodo **Claude API**, verificar que el modelo sea `claude-sonnet-4-5-20251001` (o el deseado)
6. Activar el workflow

---

## 🧠 Prompt de Claude

El prompt enviado a Claude incluye:

- Datos del candidato (nombre, años de experiencia, rol actual, ubicación)
- Descripción completa de la vacante y skills requeridas
- Texto completo del CV extraído del PDF

Claude devuelve un JSON con:

```json
{
  "score": 85,
  "status": "Recomendado",
  "skills_match": ["React", "TypeScript", "Git"],
  "skills_missing": ["Docker"],
  "summary": "Candidato sólido con 4 años en frontend...",
  "recommendation": "Avanzar a entrevista técnica"
}
```

---

## 🚨 Workflow de Manejo de Errores

Workflow secundario (`CV Screening — Manejo de Errores`) que se activa automáticamente cuando **cualquier nodo del workflow principal falla**. Captura el error, lo formatea y alerta al equipo en el canal `#ats-errores` de Slack.

Se configura en los settings del workflow principal como `errorWorkflow`.

<img width="1459" height="812" alt="image" src="https://github.com/user-attachments/assets/181d8869-3edd-4408-b8ee-e8bf48a91745" />

| Nodo | Tipo | Descripción |
|------|------|-------------|
| **Error Trigger** | Error Trigger | Se dispara automáticamente cuando el workflow principal lanza un error en cualquier nodo. Recibe `execution`, `workflow` y contexto del fallo |
| **Formatear Error** | Code / Set | Extrae del contexto: nombre del nodo que falló, mensaje de error, nombre del workflow y link a la ejecución en n8n |
| **Slack Error** | Slack – Post Message | Envía el mensaje de error al canal `#ats-errores` con el detalle del fallo y link para investigar la ejecución |

### Configuración

En los settings del workflow principal (⚙️ → Settings → Error Workflow), seleccionar `CV Screening — Manejo de Errores`.

---

## 📁 Estructura del repositorio

```
cv-screening-n8n/
├── README.md
├── workflow.json
└── workflow-error.json
```

---

## 📌 Notas técnicas

- El nodo **Descargar CV** usa `continueOnFail: true` para no bloquear el flujo si el adjunto no existe.
- El patrón `.first().json` se usa en los nodos de Code para manejar correctamente flows de un único candidato por ejecución.
- El nodo **Parsear Respuesta Claude** incluye reintento con fallback en caso de que el JSON devuelto esté malformado.
- El campo `Slack_Channel` en la tabla `Jobs` permite rutear notificaciones a distintos canales por equipo/recruiter.

---

## 🛠️ Stack

| Herramienta | Rol |
|-------------|-----|
| [n8n](https://n8n.io) | Automatización del workflow |
| [Anthropic Claude](https://console.anthropic.com) | Análisis de CVs con IA |
| [Airtable](https://airtable.com) | Base de datos de jobs y candidatos |
| [Slack](https://slack.com) | Notificaciones al recruiter |
| [Gmail](https://mail.google.com) | Emails a candidatos |

---

## 👤 Autor

**Ana Ferreira**  
HR Operations Analyst · People Analytics · HR Automation Specialist · BI & Data Analyst  
SQL · Power BI · Looker Studio · n8n · Make · SAP SuccessFactors

📩 **Contacto:** [ani.fb95@gmail.com](mailto:ani.fb95@gmail.com) · [LinkedIn](https://www.linkedin.com/in/anaferreirabezerra) · [WhatsApp](https://wa.me/5491135077374)

---

## ⚠️ Licencia

© 2025 Ana Ferreira. Todos los derechos reservados.

Este proyecto es de **uso personal y educativo**. Queda prohibida su reproducción, distribución o uso comercial sin autorización expresa de la autora.
