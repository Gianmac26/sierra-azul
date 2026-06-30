# Sierra Azul — WhatsApp Integration & Full Platform Design

**Fecha:** 2026-06-29  
**Estado:** Aprobado  
**Proyecto:** Sierra Azul — agua ozonizada, Huanta, Ayacucho

---

## 1. Contexto y alcance

El prototipo actual es un `index.html` autocontenido (Tailwind CDN + JS vanilla, sin backend). Este diseño lo convierte en una plataforma real con:

- **WhatsApp Business Cloud API** integrada (Meta) — mensajes entrantes y salientes, campañas de remarketing
- **Backend** en Vercel Serverless Functions + Vercel Cron
- **Base de datos** en Supabase (PostgreSQL)
- **Cuatro vistas funcionales:** Inbox (existente), Pedidos, Analytics, Agentes
- **Multi-número** desde el inicio (soporte para agregar Huamanga u otras zonas sin cambiar código)
- **Etiquetas configurables** — Braulio puede crear/editar etiquetas desde la UI sin tocar código

---

## 2. Arquitectura general

```
sierra-azul/
├── index.html                  ← frontend (HTML + Tailwind CDN + JS vanilla)
├── vercel.json                 ← rutas, cron jobs, headers
├── package.json                ← @supabase/supabase-js (única dependencia)
└── api/
    ├── webhook.js              ← GET (verificación) + POST (mensajes entrantes)
    ├── messages.js             ← GET  — polling mensajes nuevos cada 3s
    ├── send.js                 ← POST — enviar mensaje individual
    ├── campaign.js             ← POST — encolar campaña masiva
    ├── campaign-worker.js      ← GET  — Vercel Cron, procesa cola en batches
    ├── pedidos.js              ← GET / POST / PATCH
    ├── agentes.js              ← GET / POST / PATCH / DELETE
    └── analytics.js            ← GET — métricas agregadas
```

### Flujo mensaje entrante
```
Cliente WhatsApp → Meta Cloud API → POST /api/webhook
  → validar X-Hub-Signature-256
  → upsert contacto + chat
  → INSERT mensajes ON CONFLICT (wamid) DO NOTHING
  → actualizar ventana_expira_at = NOW() + 24h
  → retornar 200
  ← frontend polling /api/messages cada 3s → bandeja actualizada
```

### Flujo mensaje saliente
```
Agente escribe → POST /api/send { chat_id, texto }
  → buscar teléfono + phone_number_id en Supabase
  → POST graph.facebook.com/v19.0/{phone_number_id}/messages
  → guardar mensaje saliente con wamid de respuesta de Meta
```

### Flujo campaña
```
Braulio configura campaña → POST /api/campaign
  → INSERT campanas + campana_destinatarios (estado='pendiente')
  → retorna 200 inmediatamente

Vercel Cron (cada minuto) → GET /api/campaign-worker
  → toma campaña pendiente/procesando
  → procesa batch de 20 destinatarios con 50ms entre envíos
  → actualiza contadores; si completa → estado='completada'
```

---

## 3. Variables de entorno (Vercel)

| Variable | Descripción |
|---|---|
| `META_ACCESS_TOKEN` | System User token de Meta (válido para todos los números) |
| `META_APP_SECRET` | Para validar `X-Hub-Signature-256` en webhook |
| `META_WEBHOOK_VERIFY_TOKEN` | String propio configurado en Meta Dashboard |
| `SUPABASE_URL` | URL del proyecto Supabase |
| `SUPABASE_SERVICE_ROLE_KEY` | Service role key (nunca la anon key en server-side) |

El `phone_number_id` de cada número vive en la tabla `numeros` en Supabase, no como env var.

---

## 4. Esquema de base de datos (Supabase PostgreSQL)

```sql
-- NÚMEROS WHATSAPP
CREATE TABLE numeros (
  id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  phone_number_id  TEXT NOT NULL UNIQUE,
  display_name     TEXT NOT NULL,
  numero_display   TEXT NOT NULL,
  activo           BOOLEAN DEFAULT true,
  created_at       TIMESTAMPTZ DEFAULT NOW()
);

-- ETIQUETAS CONFIGURABLES (reemplaza el array TAGS hardcodeado)
CREATE TABLE etiquetas (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  numero_id  UUID REFERENCES numeros(id) ON DELETE CASCADE,
  nombre     TEXT NOT NULL,
  color_hex  TEXT NOT NULL DEFAULT '#004a7c',
  orden      INT DEFAULT 0
);

-- AGENTES
CREATE TABLE agentes (
  id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  numero_id  UUID REFERENCES numeros(id) ON DELETE CASCADE,  -- NULL = global
  nombre     TEXT NOT NULL,
  email      TEXT UNIQUE,
  avatar     TEXT,
  rol        TEXT NOT NULL DEFAULT 'agente'
               CHECK (rol IN ('admin', 'supervisor', 'agente')),
  activo     BOOLEAN DEFAULT true,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

-- CONTACTOS
CREATE TABLE contactos (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  numero_id   UUID NOT NULL REFERENCES numeros(id),
  telefono    TEXT NOT NULL,
  nombre      TEXT NOT NULL,
  avatar      TEXT,
  notas       TEXT,
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (numero_id, telefono)
);

-- CHATS
CREATE TABLE chats (
  id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  contacto_id       UUID NOT NULL REFERENCES contactos(id) ON DELETE CASCADE,
  agente_id         UUID REFERENCES agentes(id) ON DELETE SET NULL,
  etiqueta_id       UUID REFERENCES etiquetas(id) ON DELETE SET NULL,
  ultimo_mensaje    TEXT,
  ultimo_mensaje_at TIMESTAMPTZ,
  no_leidos         INT DEFAULT 0,
  ia_activa         BOOLEAN DEFAULT true,
  ventana_expira_at TIMESTAMPTZ,
  created_at        TIMESTAMPTZ DEFAULT NOW()
);

-- MENSAJES
CREATE TABLE mensajes (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  chat_id        UUID NOT NULL REFERENCES chats(id) ON DELETE CASCADE,
  wamid          TEXT UNIQUE,         -- clave de idempotencia; NULL para salientes antes de confirmación
  direccion      TEXT NOT NULL CHECK (direccion IN ('entrante', 'saliente')),
  texto          TEXT NOT NULL,
  enviado_por_ia BOOLEAN DEFAULT false,
  agente_id      UUID REFERENCES agentes(id) ON DELETE SET NULL,
  leido          BOOLEAN DEFAULT false,
  enviado_at     TIMESTAMPTZ DEFAULT NOW()
);

-- PEDIDOS
CREATE TABLE pedidos (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  contacto_id UUID NOT NULL REFERENCES contactos(id),
  chat_id     UUID REFERENCES chats(id) ON DELETE SET NULL,
  cantidad    INT NOT NULL DEFAULT 1,
  direccion   TEXT,
  notas       TEXT,
  estado      TEXT NOT NULL DEFAULT 'pendiente'
                CHECK (estado IN ('pendiente', 'en_ruta', 'entregado', 'cancelado')),
  total_soles NUMERIC(8,2),
  created_at  TIMESTAMPTZ DEFAULT NOW(),
  updated_at  TIMESTAMPTZ DEFAULT NOW()
);

-- CAMPAÑAS
CREATE TABLE campanas (
  id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  numero_id           UUID NOT NULL REFERENCES numeros(id),
  nombre              TEXT NOT NULL,
  plantilla_nombre    TEXT NOT NULL,
  plantilla_params    JSONB,
  estado              TEXT NOT NULL DEFAULT 'pendiente'
                        CHECK (estado IN ('pendiente','procesando','completada','fallida')),
  total_destinatarios INT DEFAULT 0,
  enviados            INT DEFAULT 0,
  fallidos            INT DEFAULT 0,
  created_at          TIMESTAMPTZ DEFAULT NOW(),
  started_at          TIMESTAMPTZ,
  completed_at        TIMESTAMPTZ
);

-- COLA DE DESTINATARIOS DE CAMPAÑA
CREATE TABLE campana_destinatarios (
  id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  campana_id   UUID NOT NULL REFERENCES campanas(id) ON DELETE CASCADE,
  telefono     TEXT NOT NULL,
  nombre       TEXT,
  contacto_id  UUID REFERENCES contactos(id) ON DELETE SET NULL,
  estado       TEXT NOT NULL DEFAULT 'pendiente'
                 CHECK (estado IN ('pendiente','enviado','fallido')),
  wamid        TEXT,
  error        TEXT,
  procesado_at TIMESTAMPTZ
);

-- PLANTILLAS HSM
CREATE TABLE plantillas_hsm (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  numero_id     UUID NOT NULL REFERENCES numeros(id) ON DELETE CASCADE,
  nombre        TEXT NOT NULL,
  nombre_meta   TEXT NOT NULL,
  texto_preview TEXT NOT NULL,
  idioma        TEXT NOT NULL DEFAULT 'es',
  categoria     TEXT NOT NULL DEFAULT 'MARKETING',
  activa        BOOLEAN DEFAULT true,
  UNIQUE (numero_id, nombre_meta)
);

-- ÍNDICES
CREATE INDEX ON mensajes (chat_id, enviado_at DESC);
CREATE INDEX ON chats (contacto_id);
CREATE INDEX ON campana_destinatarios (campana_id, estado);
CREATE INDEX ON pedidos (contacto_id, created_at DESC);
```

---

## 5. API endpoints

| Método | Endpoint | Descripción |
|---|---|---|
| GET | `/api/webhook` | Verificación de Meta (challenge) |
| POST | `/api/webhook` | Recibir mensajes entrantes de WhatsApp |
| GET | `/api/messages` | Polling — mensajes nuevos `?chat_id=&after=` |
| POST | `/api/send` | Enviar mensaje individual `{ chat_id, texto }` |
| POST | `/api/campaign` | Encolar campaña masiva |
| GET | `/api/campaign-worker` | Vercel Cron — procesa cola de campañas |
| GET | `/api/pedidos` | Listar pedidos con filtros |
| POST | `/api/pedidos` | Crear pedido |
| PATCH | `/api/pedidos/:id` | Actualizar estado / campos |
| GET | `/api/agentes` | Listar agentes |
| POST | `/api/agentes` | Crear agente |
| PATCH | `/api/agentes/:id` | Editar agente |
| DELETE | `/api/agentes/:id` | Soft delete — set `activo = false`, no borra el registro |
| GET | `/api/analytics` | Métricas agregadas `?periodo=hoy|semana|mes&numero_id=` |

---

## 6. Integración WhatsApp — detalles críticos

### Validación de firma (webhook)
```js
const expected = 'sha256=' + crypto
  .createHmac('sha256', process.env.META_APP_SECRET)
  .update(rawBody)
  .digest('hex');
if (req.headers['x-hub-signature-256'] !== expected) return res.status(403).end();
```

### Idempotencia
```sql
INSERT INTO mensajes (chat_id, wamid, direccion, texto, enviado_at)
VALUES (?, ?, 'entrante', ?, ?)
ON CONFLICT (wamid) DO NOTHING;
```

### Rate limiting en campañas (Vercel Cron)
- Batch de 20 destinatarios por ejecución
- 50ms entre envíos (~20 msg/s, bajo el límite de Meta de 80/s)
- 500 contactos = 25 ejecuciones = ~25 minutos total
- Reanudable: el cron retoma donde quedó si falla a mitad

### Deuda técnica conocida
- **Polling → Supabase Realtime:** cuando escalen a múltiples agentes simultáneos, migrar `/api/messages` por suscripción Realtime de Supabase. El frontend no cambia su forma de datos, solo el mecanismo de entrega.

---

## 7. Vistas de UI

### Navegación
Los `nav-item` del sidebar llaman a `navegarA('inbox'|'pedidos'|'analytics'|'agentes')`. La función oculta las columnas del inbox y muestra el panel correspondiente dentro del área derecha.

### Vista Pedidos
- Barra de stats: contadores por estado (pendiente / en_ruta / entregado / cancelado)
- Tabla con columnas: Cliente, Bidones, Dirección, Estado (select inline), Acciones
- Selector de estado hace `PATCH /api/pedidos/:id` al cambiar
- Botón [💬] abre el chat asociado en el inbox
- Modal "Nuevo pedido": buscar cliente existente por nombre/teléfono, o número nuevo; cantidad, dirección, notas, precio
- Filtros: por estado + por período (Hoy / Semana / Mes)

### Vista Analytics
- Selector de período: Hoy / Esta semana / Este mes
- **Sección Mensajes:** total chats, tiempo promedio de primera respuesta, ratio IA vs humano, barras por etiqueta (divs CSS, sin librería de charts)
- **Sección Ventas:** bidones entregados, ingresos estimados, pedidos por estado, top clientes por volumen de pedidos
- Datos de `GET /api/analytics?periodo=&numero_id=`

### Vista Agentes
- Grid de tarjetas: nombre, rol badge, estado activo, métricas (chats atendidos, tiempo promedio de respuesta)
- Modal crear/editar: nombre, email, rol, activo toggle; avatar generado automáticamente con las 2 iniciales
- Métricas de `GET /api/agentes/:id/stats`
- Asignación de agente a chat: select en el header del inbox → `PATCH /api/chats/:id { agente_id }`

### Etiquetas configurables
- Accesibles desde Configuración (nav item existente)
- CRUD de etiquetas: nombre + color_hex + orden
- Los filter chips del inbox se generan consultando la tabla `etiquetas`, no el array hardcodeado

---

## 8. Campañas de remarketing

El botón "Campañas de remarketing" del sidebar abre un modal con dos pasos:

**Paso 1 — Destinatarios:**
- Opción A: seleccionar de contactos existentes (con filtro por etiqueta)
- Opción B: subir CSV con columnas `telefono,nombre`
- Opción C: ambos (se combinan, se deduplicar por teléfono)

**Paso 2 — Plantilla:**
- Selector de plantilla HSM
- Preview del mensaje con variables resueltas
- Botón "Enviar campaña" → `POST /api/campaign` → confirmación con total de destinatarios encolados

---

## 9. Seguridad (alcance MVP)

Los endpoints `/api/` no tienen autenticación de usuario en esta etapa — son accesibles por cualquiera que conozca la URL. Esto es aceptable para un equipo pequeño y cerrado en Huanta. Cuando el equipo crezca o la URL sea pública, agregar un header `Authorization: Bearer INTERNAL_API_KEY` validado en cada función de Vercel (una env var adicional).

El webhook de Meta sí tiene seguridad real: `X-Hub-Signature-256` garantiza que solo Meta puede llamar ese endpoint.

---

## 10. Decisiones de diseño

| Decisión | Razón |
|---|---|
| Multi-número desde el inicio | Agregar Huamanga = INSERT en `numeros`, sin cambiar código |
| `phone_number_id` en DB, no env var | Escala a N números con un solo token de acceso |
| `etiqueta_id` en `chats`, no en `contactos` | Un cliente puede tener chats de distintas categorías |
| `agentes.numero_id` nullable | Admin global; agentes operativos por número |
| Cron worker para campañas | Evita timeout de Vercel en envíos masivos; reanudable |
| Barras CSS para Analytics | Mantiene zero-dependency en el frontend |
| Polling 3s migrable a Realtime | Misma forma de datos; migración = 10 líneas cuando escalen |
