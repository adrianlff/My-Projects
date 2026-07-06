# Sistema de Captación IA — Nicho: Clínicas Estéticas

Diseño técnico del motor de adquisición: scraping por nicho/ciudad (Apify) → análisis y scoring con IA (Claude) → CRM propio con pipeline de estados → automatizaciones por estado (email de propuesta, agenda, agente 24/7) → web propia como entrada adicional de leads.

Este sistema es la implementación concreta del "Motor de adquisición" del plan general de la agencia (`plan-agencia-ia.md`), aplicado al primer nicho vertical: clínicas estéticas.

---

## 1. Arquitectura general

```
┌─────────────┐     ┌──────────────────┐     ┌─────────────────────┐
│   Apify      │────▶│  n8n (orquesta)  │────▶│   CRM propio         │
│ (scraping)   │     │  + Claude API    │◀───▶│  Next.js + Supabase │
└─────────────┘     └──────────────────┘     └─────────────────────┘
                              │                          ▲
                              ▼                          │
                     ┌──────────────────┐                │
                     │ Gmail / Calendar │                │
                     │ Twilio WhatsApp  │                │
                     └──────────────────┘                │
                                                          │
                     ┌──────────────────┐                │
                     │  Web propia       │────────────────┘
                     │ (formulario+chat) │
                     └──────────────────┘
```

- **Apify**: extrae clínicas de un nicho+ciudad (Google Maps + datos de su web si existe).
- **n8n**: orquesta todo el flujo (dispara scraping, llama a Claude para el análisis, envía emails, agenda citas). Es el "pegamento" entre servicios. Autohospedado (Docker/VPS) para no depender de límites de ejecución de la nube y controlar costes a volumen alto.
- **Claude (Anthropic API)**: analiza cada clínica (con o sin web), genera el score y la propuesta personalizada. También es el cerebro del agente conversacional 24/7.
- **CRM propio**: base de datos + interfaz Kanban de estados, construida desde cero (no un CRM de terceros).
- **Web propia**: landing con formulario + chat del agente, entra leads inbound al mismo CRM.

---

## 2. Flujo end-to-end

1. **Selección de nicho/ciudad**: desde el CRM, el usuario pulsa "Lanzar scraping", elige nicho ("clínica estética") y ciudad.
2. **Scraping (Apify)**: n8n llama al actor de Apify (Google Maps Scraper) con esos parámetros. Devuelve: nombre, dirección, teléfono, web (si existe), rating, nº reseñas, categoría.
3. **Ingesta**: n8n inserta cada resultado en el CRM (Supabase) como lead en estado `Nuevo`.
4. **Análisis IA**:
   - Si tiene web → Apify (Website Content Crawler) extrae el contenido de la web → se pasa a Claude con un prompt de auditoría (SEO on-page, velocidad/UX aparente, presencia de formulario de contacto, CTA claros, si hay chat/reservas online, coherencia de marca) → Claude devuelve: **score 0-100**, **diagnóstico** y **propuesta de mejora personalizada** (borrador en texto).
   - Si no tiene web → Claude genera una **propuesta de sitio web nuevo** basada en categoría de negocio, reseñas y competencia típica del sector.
5. **Cualificación automática**: el lead pasa a `Lead Potencial - Web a Mejorar` o `Lead Potencial - Sin Web` con el score y la propuesta guardados en su ficha. Si el score es muy alto (ya tiene web+CRM+chat competentes) pasa a `Descartado`.
6. **Revisión humana**: el usuario revisa la ficha del lead, edita si quiere la propuesta generada, y decide enviarla.
7. **Envío de propuesta**: al mover el lead a `Propuesta Enviada`, se dispara automáticamente el email con la propuesta adjunta en PDF.
8. **Seguimiento**: el lead avanza por el resto de estados según su respuesta (negociación, presupuesto, reunión, ganado/perdido).
9. **Web propia + Agente 24/7**: en paralelo, la web de la agencia capta leads propios (formulario + chat del agente), que entran directamente al mismo CRM. El agente (Claude + Twilio WhatsApp/web widget) responde dudas, cualifica y agenda citas en Google Calendar de forma autónoma.

---

## 3. Conectores necesarios

### Ya disponibles en esta sesión de Claude

| Conector | Uso en el sistema |
|---|---|
| **Supabase** (`mcp__f13fe09b...`) | Base de datos del CRM (tablas leads, estados, propuestas, timeline), autenticación del panel, Edge Functions para lógica de negocio (ej. calcular score, generar PDF). |
| **GitHub** | Repositorio del CRM (frontend Next.js), de la web de captación y de los workflows de n8n exportados como JSON (control de versiones). |
| **Airtable** (`mcp__eacf77ad...`) | Alternativa rápida si en algún momento se quiere un MVP del CRM sin backend propio (no es el plan principal, pero sirve como prototipo en días en lugar de semanas). |

No tengo un conector MCP de n8n activo en esta sesión. Como n8n es autohospedado, la forma de trabajar es: yo te genero los workflows (JSON) y la lógica de cada nodo, tú los importas en tu instancia de n8n (o me das la URL + API key de tu instancia y los despliego vía su API REST).

### A conectar en n8n (nodos nativos o vía HTTP Request)

| Conector | Para qué | Dónde se contrata/conecta |
|---|---|---|
| **Apify** | Actor "Google Maps Scraper" (datos de negocios) + "Website Content Crawler" (contenido de webs existentes) | Cuenta Apify (plan ~$49/mes); n8n tiene nodo nativo de Apify |
| **Anthropic (Claude API)** | Scoring, generación de propuestas, agente conversacional 24/7 | API key de Anthropic, vía nodo HTTP Request en n8n (o el nodo comunitario de Anthropic) |
| **Gmail / Google Workspace** | Envío automático de emails con propuesta adjunta | Conexión OAuth2 con nodo nativo de Gmail en n8n |
| **Google Calendar** | Agendar citas que cualifica el agente 24/7 | Conexión OAuth2 con nodo nativo de Google Calendar en n8n |
| **Twilio (WhatsApp Business)** | Canal del agente 24/7 para las clínicas y para tus propios leads | Cuenta Twilio + WhatsApp Sender aprobado; n8n tiene nodo nativo de Twilio |
| **Stripe** (opcional, fase posterior) | Cobro de setup/mantenimiento a clientes cerrados | Nodo nativo de Stripe en n8n |

**Nota práctica**: n8n necesita estar autohospedado (Docker en un VPS, o n8n Cloud) para que todo esto se ejecute 24/7 de forma autónoma, igual que antes con Make. La diferencia es que aquí tienes control total del coste (no pagas por operación) y puedes meter código JS/Python propio en cualquier nodo — útil, por ejemplo, para lógica de scoring adicional antes o después de la llamada a Claude.

---

## 4. Estados del CRM y automatizaciones

| # | Estado | Cómo se llega | Automatización disparada al entrar |
|---|---|---|---|
| 1 | **Nuevo / Sin Cualificar** | Resultado crudo del scraping de Apify | Dispara el análisis IA (paso 4 del flujo) |
| 2 | **Descartado en Cualificación** | Claude determina que ya tiene web/CRM/IA competentes, o no es el nicho correcto | Se archiva, no se notifica (opcional: revisión mensual en bloque) |
| 3 | **Lead Potencial - Sin Web** | Claude confirma que no tiene sitio web | Se genera y guarda la propuesta de "web nueva" en la ficha; notificación al usuario (Telegram/email) de "nuevo lead cualificado" |
| 4 | **Lead Potencial - Web a Mejorar** | Claude confirma que tiene web con oportunidades | Se genera y guarda la propuesta de mejora + score; misma notificación |
| 5 | **Contactado (Primer contacto)** | El usuario marca que hizo una llamada/mensaje previo informal | Se registra la fecha/nota en el timeline del lead |
| 6 | **Propuesta Enviada** | El usuario mueve el lead tras revisar la propuesta | **Se genera el PDF final de la propuesta (con el análisis de Claude) y se envía automáticamente por email adjunto**, usando la plantilla de marca. Se registra el envío en el timeline. |
| 7 | **Propuesta Vista** *(opcional)* | Tracking de apertura del email | Notificación al usuario de que el lead abrió la propuesta |
| 8 | **En Negociación / Seguimiento** | El lead responde con dudas o interés | Se activa una secuencia de seguimiento (recordatorio automático a los 3 y 7 días si no hay respuesta) |
| 9 | **Presupuesto Enviado** | Se acuerda alcance y se envía cotización formal | Envío automático del documento de presupuesto por email; recordatorio a los 5 días si no hay respuesta |
| 10 | **Reunión Agendada** | El agente 24/7 o el usuario agenda una llamada | Evento creado en Google Calendar + email de confirmación al lead + recordatorio 1h antes |
| 11 | **Ganado - Cliente Nuevo** | El lead firma/acepta | Se crea automáticamente la ficha de "cliente" (checklist de onboarding), y opcionalmente el enlace de pago (Stripe) |
| 12 | **Perdido / Rechazado** | El lead rechaza o no responde tras la secuencia de seguimiento | Se archiva; entra en una lista de "recontacto en 6 meses" |
| 13 | **Cliente Activo (Onboarding)** | Tras ganar, mientras se implementa el servicio | Checklist de tareas de implementación (web/CRM/agente) visibles en el CRM |
| 14 | **Cliente Recurrente** | Onboarding completado, factura mensual activa | Recordatorio mensual de facturación + revisión trimestral de resultados |

Cada transición de estado se implementa como: **trigger** (cambio de estado detectado vía webhook de base de datos en Supabase) → **workflow de n8n correspondiente** → **acción** (email, calendario, notificación, generación de documento).

---

## 5. Servicios y precios (paquetes para las clínicas)

El "análisis IA" gratuito es el gancho: cualificas al lead sin coste para ti (es automático) y se lo ofreces como auditoría gratuita, lo cual justifica el primer contacto.

| Paquete | Incluye | Setup | Mensualidad | Para quién |
|---|---|---|---|---|
| **Auditoría Gratuita** | Análisis automático de su presencia digital (o ausencia) + informe con puntuación y 3 recomendaciones clave | 0 € | — | Gancho de entrada para todos los leads cualificados |
| **Web Nueva** | Sitio WordPress optimizado a conversión (SEO local, formulario, integración CRM) | 900 – 1.500 € | 40 – 60 €/mes (hosting + mantenimiento) | Clínicas sin web |
| **Mejora de Web** | Rediseño/optimización de la web existente según el diagnóstico de la auditoría (velocidad, SEO, CTAs, formulario) | 400 – 900 € (según alcance) | 40 €/mes | Clínicas con web mejorable |
| **Agente IA 24/7** | Chat en web + WhatsApp (Twilio) que responde dudas, cualifica interesados y agenda citas en su calendario | 300 – 500 € | 80 – 150 €/mes (incluye coste de API de Claude y Twilio) | Cualquier clínica con volumen de consultas |
| **CRM a Medida** | Pipeline de pacientes/leads (consulta → presupuesto → cita → tratamiento → seguimiento), en GHL/HubSpot o CRM propio ligero | 500 – 1.200 € | 60 – 100 €/mes | Clínicas sin sistema de seguimiento de leads |
| **Full Stack** | Web (nueva o mejorada) + CRM + Agente IA + automatización de todo el flujo | 1.800 – 3.000 € (con descuento vs. suma individual) | 200 – 350 €/mes | Clínicas con presupuesto medio-alto que quieren resolverlo todo de una vez |

**Coste operativo interno estimado** (para que sepas tu margen):

- Apify: ~49 €/mes (plan básico, suficiente para varias ciudades/mes).
- n8n: VPS pequeño (~5-15 €/mes) o n8n Cloud (~20 €/mes) — sin coste por operación, a diferencia de Make.
- Claude API: coste variable por análisis/propuesta (del orden de céntimos por lead analizado; unos pocos euros/mes para volúmenes de cientos de leads).
- Supabase: gratis hasta cierto volumen, luego ~25 €/mes.
- Twilio WhatsApp: coste por conversación (categoría marketing/utility/servicio de Meta + margen de Twilio), estimar 0,02-0,06 €/conversación iniciada por el negocio, más el alquiler del número (~1 €/mes).

Con esto, tu coste fijo mensual del sistema ronda 90-130 €/mes para cientos de leads cualificados automáticamente — el margen sobre cualquier cliente cerrado es alto, y con n8n autohospedado ese coste no crece con el volumen de ejecuciones (solo con el uso de Apify/Claude/Twilio).

---

## Próximo paso técnico

Recomiendo construirlo en este orden (evita montar todo a la vez):

1. Modelo de datos en Supabase (tabla `leads` con estado, score, propuesta, timeline) + workflow de n8n que hace scraping + inserta leads `Nuevo`.
2. Workflow de análisis IA (Claude) que cualifica y genera la propuesta.
3. Panel Kanban mínimo en Next.js para mover leads entre estados.
4. Automatización de envío de propuesta al pasar a `Propuesta Enviada`.
5. Web propia + agente 24/7 con Twilio WhatsApp (última fase, una vez el pipeline de captación ya genera leads cualificados).

Para el paso 1 necesito que tengas (o montemos) una instancia de n8n accesible — dime si ya tienes una corriendo (Docker/VPS/n8n Cloud) o si la levantamos como parte de este trabajo. Dime también si quieres que empecemos por el modelo de datos de Supabase y el primer workflow (scraping → CRM), o prefieres primero el panel del CRM.
