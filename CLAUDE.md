# CLAUDE.md — Piscinas Vazquez (auditoría de WhatsApp vía Kommo, Zaiko)

> Hablarle SIEMPRE en español rioplatense al usuario, corto y sin tecnicismos (no es técnico).

**Este repositorio es SOLO de Piscinas Vazquez** (`hotel_id = 'piscinas-vazquez'`, embudo de Kommo `14468572`).
Guarda lo propio de este cliente: su configuración (`cliente.json`), sus tableros de Grafana (`grafana/`),
su relevamiento y su estado (`ESTADO.md`).

## Lo que NO está acá (y por qué)
Piscinas Vazquez funciona dentro del **sistema compartido de hoteles**, junto con Caliu, Oculto Hotel Boutique,
Universo Liv y Piscinas Vazquez:
- **Misma base de datos** (Supabase, datasource de Grafana `bfz14hr25v6kgd`). Los datos de este cliente son
  las filas con `hotel_id = 'piscinas-vazquez'` en las tablas `wa_*`. Decisión del usuario (17/11/2026): los hoteles
  siguen en la misma base.
- **Mismos 7 flujos de n8n** ("Kuro - WhatsApp Auditor - …", carpeta "Kuro - Auditoría WhatsApp"). NO se
  copian por cliente: el control de ritmo de pedidos a Kommo es uno solo para toda la cuenta; dos copias se
  pisarían y Kommo nos podría bloquear.
- El código de esos flujos, el esquema de la base, las migraciones y TODA la documentación técnica están en el
  repo `UserMZ2026/Zaiko-Flujos_para_auditar_hoteles` (rama más nueva al 17/11/2026:
  `claude/zen-dijkstra-cp9srr`). **Antes de tocar flujos o la base, agregar ese repo a la sesión y leer su
  CLAUDE.md y `whatsapp-auditoria-kommo/05_README_INSTALACION.md`.**

## Reglas (no negociables, resumen del repo compartido)
1. Kommo es SOLO LECTURA: nunca POST/PATCH/PUT/DELETE contra Kommo.
2. Toda llamada a Kommo pasa por el sub-flujo "Kommo GET Controlado" (`8Gx97d31oH6SmJfX`).
3. Un cambio en los flujos compartidos afecta a TODOS los clientes: probarlo pensando en los cinco.
4. Toda consulta o panel de este cliente filtra `hotel_id = 'piscinas-vazquez'`. Nunca mostrar datos de otro cliente.
5. Nunca inventar datos: la IA solo extrae lo explícito; si no está, va vacío.
6. Publicar un tablero con link público lo hace el usuario a mano (Share externally en Grafana).
7. Nunca commitear tokens, contraseñas ni mails de clientes.

## Datos útiles
- Grafana: https://jadecrepe1761.grafana.net (token en la variable `GRAFANA_TOKEN`).
- n8n: https://vps-5158417-x.dattaweb.com (API `/api/v1`, header `X-N8N-API-KEY: $N8N_API_KEY`).
- Si se cambia un tablero de este cliente en Grafana, bajar el JSON a `grafana/` en el mismo commit.
- Si cambia un link público, actualizar `wa_hotels` (`grafana_demanda_url`, `grafana_gestion_url`): el reporte
  diario lee los links de ahí.
