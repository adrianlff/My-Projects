# Contexto del proyecto: Agencia de Soluciones IA

Este repositorio documenta el lanzamiento de una agencia de soluciones IA (webs, CRM, automatización, agentes IA), fundada por Adrián, con 13+ años de experiencia en:

- Creación de sitios WordPress (Koru Entrenamiento, Empodera tu Cuerpo, Cali Asesores, Ran4Capital, Pizzas Print Vallecas, Laser Arganda, Poma Social).
- Automatizaciones en Make y n8n.
- CRMs en HubSpot y Go High Level.
- Dashboards de estadísticas de negocio para clientes.

## Archivos de este repo

- `plan-agencia-ia.md` — plan general de lanzamiento de la agencia: posicionamiento, paquetes de servicio, prueba social, estructura legal, motor de adquisición de clientes, roadmap de 90 días.
- `sistema-captacion-clinicas-esteticas.md` — diseño técnico del primer sistema de captación: scraping con Apify de un nicho (clínicas estéticas) en una ciudad determinada, scoring y generación de propuestas con Claude, CRM propio con pipeline de estados y automatizaciones, agente IA 24/7 por WhatsApp. Incluye precios y paquetes de servicio para el nicho.

## Decisiones ya tomadas (no las reabras sin motivo)

- **Orquestación de automatizaciones: n8n**, no Make. Motivo: sin coste por operación, control total con código propio.
- **n8n autohospedado**: ya existe una instancia corriendo en un VPS de Hostinger, en `https://n8n.koruentrenamiento.com/` — originalmente creada para flujos del CRM de Koru Entrenamiento (cliente de gimnasios). Se reutilizará el mismo VPS para los flujos de la agencia (no se comprará un VPS nuevo), usando prefijos de nombre (`[IA Agency] ...`) y credenciales separadas de las de Koru para no mezclar proyectos.
- **Canal de WhatsApp: Twilio** (como BSP), no la Cloud API de Meta directamente. Motivo: el modelo es multi-cliente (varias clínicas), y Twilio permite gestionar varios números de WhatsApp desde una única cuenta maestra, evitando que cada clínica tenga que pasar su propia verificación de Meta Business.
- **Backend del CRM: Supabase**, con un **proyecto nuevo y separado** del proyecto Supabase ya existente (`koru`, usado por otro proyecto de Adrián) — no se debe tocar ni reutilizar ese proyecto existente.
- **Frontend del CRM**: Next.js (pendiente de iniciar).

## Estado actual / pendiente

- No se ha creado aún el proyecto Supabase nuevo para el CRM de la agencia.
- No se ha desplegado aún ningún workflow de n8n para este proyecto (scraping, análisis IA, envío de propuestas).
- Pendiente: credenciales necesarias para conectar y empezar a construir:
  - Supabase: `SUPABASE_URL` y `SUPABASE_SERVICE_ROLE_KEY` del proyecto nuevo (aún por crear).
  - n8n: URL de la instancia (`https://n8n.koruentrenamiento.com`) + una API key nueva y dedicada (generar en Settings → n8n API → Create API key).
  - Apify: cuenta + API token (para los actores "Google Maps Scraper" y "Website Content Crawler").
  - Twilio: cuenta + credenciales (Account SID, Auth Token) + número de WhatsApp Business aprobado.
  - Anthropic (Claude API): API key para las llamadas de scoring/generación de propuestas desde n8n.
- **Importante de seguridad**: ninguna de estas claves debe subirse nunca a git. El repo ya tiene `.gitignore` con `.env` excluido. Guarda las credenciales en un `.env` local (no versionado) y, si se comparten con Claude, usa claves nuevas y dedicadas a este proyecto para poder revocarlas fácilmente.

## Roadmap técnico (orden recomendado)

1. Modelo de datos en Supabase (tabla `leads` con estado, score, propuesta, timeline) + primer workflow de n8n que hace scraping (Apify) e inserta leads en estado `Nuevo`.
2. Workflow de análisis IA (Claude) que cualifica el lead (con/sin web) y genera la propuesta personalizada.
3. Panel Kanban mínimo en Next.js para mover leads entre los 14 estados definidos en `sistema-captacion-clinicas-esteticas.md`.
4. Automatización de envío de propuesta por email (PDF adjunto) al mover el lead a `Propuesta Enviada`.
5. Web propia de la agencia (landing + formulario) y agente IA 24/7 vía Twilio WhatsApp, conectado al mismo CRM.

## Cómo continuar esta conversación en local

Este `.md` resume todo el contexto necesario para retomar el trabajo con Claude Code en el ordenador. Al abrir el proyecto, indícale a Claude que lea este archivo (`CLAUDE.md`) antes de continuar, y decide junto a él si seguimos por el punto 1 del roadmap (modelo de datos + primer workflow de n8n) una vez tengas las credenciales listas.
