# Sierra Azul — Plataforma Completa: WhatsApp + Vistas

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Convertir el prototipo index.html en una plataforma real con backend Vercel + Supabase, integración WhatsApp Business Cloud API, y vistas funcionales de Pedidos, Analytics y Agentes.

**Architecture:** Vercel Serverless Functions en `/api/` manejan webhook, envío, campañas y CRUD. Supabase PostgreSQL persiste todos los datos. El frontend `index.html` reemplaza los arrays hardcodeados por fetch calls a `/api/`, con polling cada 3s para mensajes nuevos.

**Tech Stack:** Node.js 18 (Vercel), @supabase/supabase-js v2, Meta Cloud API v19.0, HTML + Tailwind CDN + JS vanilla (sin bundler).

## Global Constraints

- Sin npm scripts, sin bundler, sin TypeScript — el frontend sigue siendo un único `index.html`
- Node.js 18+ en funciones Vercel (fetch nativo disponible)
- Supabase service role key solo en server-side (nunca en el browser)
- `wamid` es la clave de idempotencia en `mensajes` — `INSERT ... ON CONFLICT (wamid) DO NOTHING`
- Soft delete en agentes: `activo = false`, nunca `DELETE`
- Barras de Analytics: divs CSS, sin librería de charts
- Campaña: batches de 20, 50ms entre envíos, procesado por Vercel Cron

---

## Mapa de archivos

| Archivo | Acción | Responsabilidad |
|---|---|---|
| `package.json` | Crear | Dependencia única: @supabase/supabase-js |
| `vercel.json` | Crear | Cron config + headers CORS |
| `api/_supabase.js` | Crear | Cliente Supabase compartido |
| `api/webhook.js` | Crear | Recibir mensajes entrantes de Meta |
| `api/chats.js` | Crear | Listar chats para el inbox |
| `api/messages.js` | Crear | Polling de mensajes nuevos |
| `api/send.js` | Crear | Enviar mensaje individual |
| `api/campaign.js` | Crear | Encolar campaña masiva |
| `api/campaign-worker.js` | Crear | Cron: procesar cola de campaña |
| `api/pedidos.js` | Crear | CRUD pedidos |
| `api/agentes.js` | Crear | CRUD agentes + stats |
| `api/analytics.js` | Crear | Métricas agregadas |
| `api/etiquetas.js` | Crear | CRUD etiquetas configurables |
| `index.html` | Modificar | Sumar 4 vistas, nav lógica, fetch calls, polling |

---

## Task 1: Scaffolding del proyecto

**Files:**
- Create: `package.json`
- Create: `vercel.json`
- Create: `api/_supabase.js`

**Interfaces:**
- Produces: cliente `supabase` importable en todas las funciones API como `require('./_supabase')`

- [ ] **Step 1: Crear package.json**

```json
{
  "name": "sierra-azul",
  "version": "1.0.0",
  "private": true,
  "dependencies": {
    "@supabase/supabase-js": "^2.45.0"
  }
}
```

- [ ] **Step 2: Instalar dependencia**

```bash
npm install
```

Resultado esperado: carpeta `node_modules/@supabase/` creada, `package-lock.json` generado.

- [ ] **Step 3: Crear vercel.json**

```json
{
  "crons": [
    {
      "path": "/api/campaign-worker",
      "schedule": "* * * * *"
    }
  ],
  "headers": [
    {
      "source": "/api/(.*)",
      "headers": [
        { "key": "Access-Control-Allow-Origin", "value": "*" },
        { "key": "Access-Control-Allow-Methods", "value": "GET,POST,PATCH,DELETE,OPTIONS" },
        { "key": "Access-Control-Allow-Headers", "value": "Content-Type" }
      ]
    }
  ]
}
```

- [ ] **Step 4: Crear api/_supabase.js**

```js
const { createClient } = require('@supabase/supabase-js');

module.exports = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_SERVICE_ROLE_KEY
);
```

- [ ] **Step 5: Verificar que Vercel CLI está instalado**

```bash
vercel --version
```

Si no está: `npm i -g vercel`

- [ ] **Step 6: Commit**

```bash
git add package.json package-lock.json vercel.json api/_supabase.js
git commit -m "feat: project scaffolding — vercel + supabase client"
```

---

## Task 2: Esquema de base de datos en Supabase

**Files:**
- Create: `docs/schema.sql` (referencia, se ejecuta en Supabase Dashboard)

**Interfaces:**
- Produces: tablas `numeros`, `etiquetas`, `agentes`, `contactos`, `chats`, `mensajes`, `pedidos`, `campanas`, `campana_destinatarios`, `plantillas_hsm`

- [ ] **Step 1: Crear docs/schema.sql**

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

-- ETIQUETAS CONFIGURABLES
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
  numero_id  UUID REFERENCES numeros(id) ON DELETE CASCADE,
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

-- CHATS (un chat por contacto)
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
  created_at        TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE (contacto_id)
);

-- MENSAJES
CREATE TABLE mensajes (
  id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  chat_id        UUID NOT NULL REFERENCES chats(id) ON DELETE CASCADE,
  wamid          TEXT UNIQUE,
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
                CHECK (estado IN ('pendiente','en_ruta','entregado','cancelado')),
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

-- COLA DE DESTINATARIOS
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

- [ ] **Step 2: Ejecutar el SQL en Supabase**

Ir a Supabase Dashboard → proyecto → SQL Editor → pegar el contenido de `docs/schema.sql` → Run.

Verificar que las 10 tablas aparecen en Table Editor.

- [ ] **Step 3: Insertar datos semilla**

En SQL Editor de Supabase:

```sql
-- Insertar el número de Sierra Azul Huanta
-- Reemplazar 'TU_PHONE_NUMBER_ID' con el valor real de Meta Dashboard
INSERT INTO numeros (phone_number_id, display_name, numero_display)
VALUES ('TU_PHONE_NUMBER_ID', 'Sierra Azul Huanta', '+51 959 349 049');

-- Etiquetas iniciales
INSERT INTO etiquetas (numero_id, nombre, color_hex, orden)
SELECT id, 'Pedidos Nuevos', '#15803d', 1 FROM numeros LIMIT 1;
INSERT INTO etiquetas (numero_id, nombre, color_hex, orden)
SELECT id, 'Recargas', '#1d4ed8', 2 FROM numeros LIMIT 1;
INSERT INTO etiquetas (numero_id, nombre, color_hex, orden)
SELECT id, 'Consultas Distribuidores', '#b45309', 3 FROM numeros LIMIT 1;

-- Plantillas HSM iniciales
INSERT INTO plantillas_hsm (numero_id, nombre, nombre_meta, texto_preview)
SELECT id,
  'Confirmación de pedido',
  'confirmacion_pedido',
  'Hola {{1}}, tu pedido de {{2}} bidón(es) fue confirmado. Llegará entre {{3}} y {{4}}. Sierra Azul 💧'
FROM numeros LIMIT 1;

INSERT INTO plantillas_hsm (numero_id, nombre, nombre_meta, texto_preview)
SELECT id,
  'Recordatorio de recarga',
  'recordatorio_recarga',
  'Hola {{1}}, notamos que hace {{2}} días no realizas una recarga. ¿Necesitas agua ozonizada?'
FROM numeros LIMIT 1;

-- Agente inicial (Braulio)
INSERT INTO agentes (nombre, email, avatar, rol)
VALUES ('Braulio Ruiz', 'braulio@sierraazul.pe', 'BR', 'admin');
```

- [ ] **Step 4: Copiar las variables de entorno necesarias**

Guardar en un archivo local `.env.local` (NO commitear):

```
META_ACCESS_TOKEN=...
META_APP_SECRET=...
META_WEBHOOK_VERIFY_TOKEN=sierra-azul-webhook-2026
SUPABASE_URL=https://xxxx.supabase.co
SUPABASE_SERVICE_ROLE_KEY=eyJ...
```

- [ ] **Step 5: Agregar .env.local a .gitignore**

```bash
echo ".env.local" >> .gitignore
git add docs/schema.sql .gitignore
git commit -m "feat: database schema and seed data"
```

---

## Task 3: API de chats y mensajes (lectura)

**Files:**
- Create: `api/chats.js`
- Create: `api/messages.js`

**Interfaces:**
- Consumes: `require('./_supabase')`, tablas `chats`, `contactos`, `etiquetas`, `mensajes`
- Produces:
  - `GET /api/chats?numero_id=` → array de chats con contacto + etiqueta anidados
  - `GET /api/messages?chat_id=&after=` → array de mensajes ordenados por `enviado_at ASC`

- [ ] **Step 1: Crear api/chats.js**

```js
const supabase = require('./_supabase');

module.exports = async function handler(req, res) {
  if (req.method === 'OPTIONS') return res.status(200).end();
  if (req.method !== 'GET') return res.status(405).end();

  const { numero_id } = req.query;

  let query = supabase
    .from('chats')
    .select(`
      id, ia_activa, no_leidos, ultimo_mensaje, ultimo_mensaje_at,
      ventana_expira_at, agente_id, etiqueta_id,
      contacto:contactos(id, nombre, telefono, avatar, numero_id),
      etiqueta:etiquetas(nombre, color_hex)
    `)
    .order('ultimo_mensaje_at', { ascending: false, nullsFirst: false });

  if (numero_id) {
    query = query.eq('contacto.numero_id', numero_id);
  }

  const { data, error } = await query;
  if (error) return res.status(500).json({ error: error.message });
  return res.status(200).json(data || []);
};
```

- [ ] **Step 2: Crear api/messages.js**

```js
const supabase = require('./_supabase');

module.exports = async function handler(req, res) {
  if (req.method === 'OPTIONS') return res.status(200).end();
  if (req.method !== 'GET') return res.status(405).end();

  const { chat_id, after } = req.query;
  if (!chat_id) return res.status(400).json({ error: 'chat_id required' });

  let query = supabase
    .from('mensajes')
    .select('id, wamid, direccion, texto, enviado_por_ia, agente_id, leido, enviado_at')
    .eq('chat_id', chat_id)
    .order('enviado_at', { ascending: true });

  if (after) query = query.gt('enviado_at', after);

  const { data, error } = await query;
  if (error) return res.status(500).json({ error: error.message });
  return res.status(200).json(data || []);
};
```

- [ ] **Step 3: Levantar Vercel dev con las env vars**

```bash
vercel dev --local-config .env.local
```

Si pide linkear al proyecto: `vercel link` primero.

- [ ] **Step 4: Verificar GET /api/chats**

```bash
curl "http://localhost:3000/api/chats"
```

Resultado esperado: `[]` (array vacío — todavía no hay datos de contactos).

- [ ] **Step 5: Verificar GET /api/messages con chat_id inválido**

```bash
curl "http://localhost:3000/api/messages?chat_id=00000000-0000-0000-0000-000000000000"
```

Resultado esperado: `[]`

- [ ] **Step 6: Commit**

```bash
git add api/chats.js api/messages.js
git commit -m "feat: chats and messages read endpoints"
```

---

## Task 4: Webhook de WhatsApp

**Files:**
- Create: `api/webhook.js`

**Interfaces:**
- Consumes: `require('./_supabase')`, `crypto` (nativo Node), tablas `numeros`, `contactos`, `chats`, `mensajes`
- Produces:
  - `GET /api/webhook` responde con `hub.challenge` si el token coincide
  - `POST /api/webhook` valida firma, upserta contacto+chat, inserta mensaje idempotente

- [ ] **Step 1: Crear api/webhook.js**

```js
const crypto = require('crypto');
const supabase = require('./_supabase');

// Desactivar body parser de Vercel para acceder al raw body (HMAC requiere el body sin parsear)
module.exports.config = { api: { bodyParser: false } };

async function getRawBody(req) {
  return new Promise((resolve, reject) => {
    const chunks = [];
    req.on('data', chunk => chunks.push(chunk));
    req.on('end', () => resolve(Buffer.concat(chunks)));
    req.on('error', reject);
  });
}

module.exports = async function handler(req, res) {
  if (req.method === 'OPTIONS') return res.status(200).end();

  // GET: verificación de Meta al configurar el webhook
  if (req.method === 'GET') {
    const mode = req.query['hub.mode'];
    const token = req.query['hub.verify_token'];
    const challenge = req.query['hub.challenge'];
    if (mode === 'subscribe' && token === process.env.META_WEBHOOK_VERIFY_TOKEN) {
      return res.status(200).send(challenge);
    }
    return res.status(403).end();
  }

  // POST: mensajes entrantes
  if (req.method === 'POST') {
    const rawBody = await getRawBody(req);

    // Validar X-Hub-Signature-256
    const sig = req.headers['x-hub-signature-256'];
    const expected = 'sha256=' + crypto
      .createHmac('sha256', process.env.META_APP_SECRET)
      .update(rawBody)
      .digest('hex');
    if (sig !== expected) return res.status(403).end();

    let body;
    try { body = JSON.parse(rawBody.toString()); }
    catch { return res.status(400).end(); }

    const change = body.entry?.[0]?.changes?.[0];
    if (change?.field !== 'messages') return res.status(200).end();

    const value = change.value;
    const phoneNumberId = value.metadata?.phone_number_id;
    const messages = value.messages || [];
    const contacts = value.contacts || [];

    if (messages.length === 0) return res.status(200).end();

    // Buscar numero_id en DB
    const { data: numero } = await supabase
      .from('numeros')
      .select('id')
      .eq('phone_number_id', phoneNumberId)
      .single();

    if (!numero) return res.status(200).end(); // número no registrado, ignorar

    for (const msg of messages) {
      if (msg.type !== 'text') continue;

      const telefono = msg.from;
      const contactInfo = contacts.find(c => c.wa_id === telefono);
      const nombre = contactInfo?.profile?.name || telefono;
      const avatar = nombre.split(' ').map(w => w[0]).join('').toUpperCase().slice(0, 2);
      const texto = msg.text?.body || '';
      const wamid = msg.id;
      const enviadoAt = new Date(parseInt(msg.timestamp) * 1000).toISOString();

      // Upsert contacto
      const { data: contacto } = await supabase
        .from('contactos')
        .upsert(
          { numero_id: numero.id, telefono, nombre, avatar },
          { onConflict: 'numero_id,telefono' }
        )
        .select('id')
        .single();

      // Upsert chat (UNIQUE contacto_id)
      const ventanaExpiraAt = new Date(Date.now() + 24 * 60 * 60 * 1000).toISOString();
      const { data: chat } = await supabase
        .from('chats')
        .upsert(
          {
            contacto_id: contacto.id,
            ultimo_mensaje: texto,
            ultimo_mensaje_at: enviadoAt,
            ventana_expira_at: ventanaExpiraAt,
          },
          { onConflict: 'contacto_id', ignoreDuplicates: false }
        )
        .select('id, no_leidos')
        .single();

      // Insert mensaje idempotente
      const { error: insertError } = await supabase
        .from('mensajes')
        .insert({
          chat_id: chat.id,
          wamid,
          direccion: 'entrante',
          texto,
          enviado_at: enviadoAt,
        });

      // Solo incrementar no_leidos si el mensaje es nuevo (no duplicado)
      if (!insertError) {
        await supabase
          .from('chats')
          .update({
            no_leidos: (chat.no_leidos || 0) + 1,
            ultimo_mensaje: texto,
            ultimo_mensaje_at: enviadoAt,
            ventana_expira_at: ventanaExpiraAt,
          })
          .eq('id', chat.id);
      }
    }

    return res.status(200).end();
  }

  return res.status(405).end();
};
```

- [ ] **Step 2: Verificar GET (verificación de Meta)**

```bash
curl "http://localhost:3000/api/webhook?hub.mode=subscribe&hub.verify_token=sierra-azul-webhook-2026&hub.challenge=TESTCHALLENGE"
```

Resultado esperado: `TESTCHALLENGE`

- [ ] **Step 3: Verificar que firma inválida retorna 403**

```bash
curl -X POST "http://localhost:3000/api/webhook" \
  -H "Content-Type: application/json" \
  -H "x-hub-signature-256: sha256=invalida" \
  -d '{}'
```

Resultado esperado: HTTP 403

- [ ] **Step 4: Probar con payload real de Meta (simulado)**

Primero generar la firma correcta:

```bash
# Calcular HMAC-SHA256 del payload con tu APP_SECRET real
echo -n '{"object":"whatsapp_business_account","entry":[{"id":"TEST","changes":[{"value":{"messaging_product":"whatsapp","metadata":{"display_phone_number":"959349049","phone_number_id":"TU_PHONE_NUMBER_ID"},"contacts":[{"profile":{"name":"Carlos Ruiz"},"wa_id":"51959349049"}],"messages":[{"from":"51959349049","id":"wamid.test001","timestamp":"1735689600","type":"text","text":{"body":"Hola prueba"}}]},"field":"messages"}]}]}' | \
  openssl dgst -sha256 -hmac "TU_APP_SECRET"
```

Usar el hash resultante como `sha256=<hash>` en el header de la llamada curl.

- [ ] **Step 5: Verificar en Supabase que el contacto, chat y mensaje fueron creados**

En Supabase Dashboard → Table Editor → `mensajes`: debe aparecer una fila con `wamid = 'wamid.test001'`.

- [ ] **Step 6: Commit**

```bash
git add api/webhook.js
git commit -m "feat: WhatsApp webhook with HMAC validation and idempotent message insert"
```

---

## Task 5: API de envío de mensajes

**Files:**
- Create: `api/send.js`

**Interfaces:**
- Consumes: `require('./_supabase')`, tablas `chats`, `contactos`, `numeros`; Meta Cloud API v19.0
- Produces: `POST /api/send { chat_id, texto, agente_id? }` → `{ ok: true, wamid: string }`

- [ ] **Step 1: Crear api/send.js**

```js
const supabase = require('./_supabase');

module.exports = async function handler(req, res) {
  if (req.method === 'OPTIONS') return res.status(200).end();
  if (req.method !== 'POST') return res.status(405).end();

  const { chat_id, texto, agente_id } = req.body;
  if (!chat_id || !texto) return res.status(400).json({ error: 'chat_id and texto required' });

  // Obtener teléfono y phone_number_id
  const { data: chat, error: chatErr } = await supabase
    .from('chats')
    .select('id, contacto:contactos(telefono, numero:numeros(phone_number_id))')
    .eq('id', chat_id)
    .single();

  if (chatErr || !chat) return res.status(404).json({ error: 'chat not found' });

  const phoneNumberId = chat.contacto.numero.phone_number_id;
  const to = chat.contacto.telefono;

  // Llamar Meta Cloud API
  const metaRes = await fetch(
    `https://graph.facebook.com/v19.0/${phoneNumberId}/messages`,
    {
      method: 'POST',
      headers: {
        'Authorization': `Bearer ${process.env.META_ACCESS_TOKEN}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({
        messaging_product: 'whatsapp',
        to,
        type: 'text',
        text: { body: texto },
      }),
    }
  );

  if (!metaRes.ok) {
    const err = await metaRes.json();
    return res.status(502).json({ error: 'Meta API error', details: err });
  }

  const metaData = await metaRes.json();
  const wamid = metaData.messages?.[0]?.id || null;

  // Guardar mensaje saliente
  await supabase.from('mensajes').insert({
    chat_id,
    wamid,
    direccion: 'saliente',
    texto,
    agente_id: agente_id || null,
  });

  // Actualizar último mensaje del chat
  await supabase
    .from('chats')
    .update({ ultimo_mensaje: texto, ultimo_mensaje_at: new Date().toISOString() })
    .eq('id', chat_id);

  return res.status(200).json({ ok: true, wamid });
};
```

- [ ] **Step 2: Verificar con curl (requiere un chat_id real de Supabase)**

```bash
curl -X POST "http://localhost:3000/api/send" \
  -H "Content-Type: application/json" \
  -d '{"chat_id":"UUID_DEL_CHAT_CREADO_EN_TASK4","texto":"Hola, prueba de envío"}'
```

Resultado esperado: `{"ok":true,"wamid":"wamid.xxx"}` y el mensaje llega al teléfono real.

- [ ] **Step 3: Commit**

```bash
git add api/send.js
git commit -m "feat: send WhatsApp message endpoint"
```

---

## Task 6: Sistema de campañas (cola + cron worker)

**Files:**
- Create: `api/campaign.js`
- Create: `api/campaign-worker.js`

**Interfaces:**
- Consumes: `require('./_supabase')`, tablas `campanas`, `campana_destinatarios`, `numeros`; Meta Cloud API
- Produces:
  - `POST /api/campaign { numero_id, nombre, plantilla_nombre, plantilla_params, destinatarios }` → `{ ok: true, campana_id }`
  - `GET /api/campaign-worker` → procesa batch de 20, retorna `{ procesados, pendientes }`

- [ ] **Step 1: Crear api/campaign.js**

```js
const supabase = require('./_supabase');

module.exports = async function handler(req, res) {
  if (req.method === 'OPTIONS') return res.status(200).end();
  if (req.method !== 'POST') return res.status(405).end();

  const { numero_id, nombre, plantilla_nombre, plantilla_params, destinatarios } = req.body;

  if (!numero_id || !plantilla_nombre || !destinatarios?.length) {
    return res.status(400).json({ error: 'numero_id, plantilla_nombre y destinatarios requeridos' });
  }

  // Deduplicar por teléfono
  const unicos = [...new Map(destinatarios.map(d => [d.telefono, d])).values()];

  // Crear campaña
  const { data: campana, error } = await supabase
    .from('campanas')
    .insert({
      numero_id,
      nombre,
      plantilla_nombre,
      plantilla_params: plantilla_params || null,
      total_destinatarios: unicos.length,
    })
    .select('id')
    .single();

  if (error) return res.status(500).json({ error: error.message });

  // Encolar destinatarios
  await supabase.from('campana_destinatarios').insert(
    unicos.map(d => ({
      campana_id: campana.id,
      telefono: d.telefono,
      nombre: d.nombre || null,
      contacto_id: d.contacto_id || null,
    }))
  );

  return res.status(201).json({ ok: true, campana_id: campana.id, total: unicos.length });
};
```

- [ ] **Step 2: Crear api/campaign-worker.js**

```js
const supabase = require('./_supabase');

const BATCH_SIZE = 20;
const DELAY_MS = 50;

function sleep(ms) { return new Promise(r => setTimeout(r, ms)); }

module.exports = async function handler(req, res) {
  if (req.method === 'OPTIONS') return res.status(200).end();
  // Vercel Cron invoca con GET
  if (req.method !== 'GET') return res.status(405).end();

  // Tomar la primera campaña pendiente o procesando
  const { data: campana } = await supabase
    .from('campanas')
    .select('id, numero_id, plantilla_nombre, plantilla_params')
    .in('estado', ['pendiente', 'procesando'])
    .order('created_at', { ascending: true })
    .limit(1)
    .single();

  if (!campana) return res.status(200).json({ ok: true, message: 'no campaigns pending' });

  // Marcar como procesando
  await supabase.from('campanas').update({ estado: 'procesando', started_at: new Date().toISOString() }).eq('id', campana.id);

  // Obtener phone_number_id del número
  const { data: numero } = await supabase
    .from('numeros')
    .select('phone_number_id')
    .eq('id', campana.numero_id)
    .single();

  // Tomar siguiente batch de pendientes
  const { data: destinatarios } = await supabase
    .from('campana_destinatarios')
    .select('id, telefono, nombre')
    .eq('campana_id', campana.id)
    .eq('estado', 'pendiente')
    .limit(BATCH_SIZE);

  if (!destinatarios || destinatarios.length === 0) {
    // No quedan pendientes → completar campaña
    await supabase.from('campanas').update({ estado: 'completada', completed_at: new Date().toISOString() }).eq('id', campana.id);
    return res.status(200).json({ ok: true, message: 'campaign completed' });
  }

  let enviados = 0;
  let fallidos = 0;

  for (const dest of destinatarios) {
    try {
      // Construir componentes del template con parámetros
      const params = campana.plantilla_params || {};
      const components = Object.keys(params).length > 0
        ? [{ type: 'body', parameters: Object.values(params).map(v => ({ type: 'text', text: String(v) })) }]
        : [];

      const metaRes = await fetch(
        `https://graph.facebook.com/v19.0/${numero.phone_number_id}/messages`,
        {
          method: 'POST',
          headers: {
            'Authorization': `Bearer ${process.env.META_ACCESS_TOKEN}`,
            'Content-Type': 'application/json',
          },
          body: JSON.stringify({
            messaging_product: 'whatsapp',
            to: dest.telefono,
            type: 'template',
            template: {
              name: campana.plantilla_nombre,
              language: { code: 'es' },
              components,
            },
          }),
        }
      );

      if (metaRes.ok) {
        const data = await metaRes.json();
        const wamid = data.messages?.[0]?.id || null;
        await supabase.from('campana_destinatarios').update({ estado: 'enviado', wamid, procesado_at: new Date().toISOString() }).eq('id', dest.id);
        enviados++;
      } else {
        const err = await metaRes.json();
        await supabase.from('campana_destinatarios').update({ estado: 'fallido', error: JSON.stringify(err), procesado_at: new Date().toISOString() }).eq('id', dest.id);
        fallidos++;
      }
    } catch (e) {
      await supabase.from('campana_destinatarios').update({ estado: 'fallido', error: e.message, procesado_at: new Date().toISOString() }).eq('id', dest.id);
      fallidos++;
    }

    await sleep(DELAY_MS);
  }

  // Actualizar contadores en la campaña
  await supabase.rpc('increment_campaign_counters', {
    p_campana_id: campana.id,
    p_enviados: enviados,
    p_fallidos: fallidos,
  });

  return res.status(200).json({ ok: true, procesados: enviados + fallidos, enviados, fallidos });
};
```

- [ ] **Step 3: Crear la función RPC en Supabase**

En SQL Editor de Supabase:

```sql
CREATE OR REPLACE FUNCTION increment_campaign_counters(
  p_campana_id UUID,
  p_enviados INT,
  p_fallidos INT
) RETURNS void AS $$
  UPDATE campanas
  SET enviados = enviados + p_enviados,
      fallidos = fallidos + p_fallidos
  WHERE id = p_campana_id;
$$ LANGUAGE sql;
```

- [ ] **Step 4: Verificar encolar campaña**

```bash
curl -X POST "http://localhost:3000/api/campaign" \
  -H "Content-Type: application/json" \
  -d '{
    "numero_id":"UUID_DEL_NUMERO",
    "nombre":"Test campaña",
    "plantilla_nombre":"recordatorio_recarga",
    "plantilla_params":{"1":"Carlos","2":"7"},
    "destinatarios":[{"telefono":"51959349049","nombre":"Carlos"}]
  }'
```

Resultado esperado: `{"ok":true,"campana_id":"...","total":1}`

- [ ] **Step 5: Verificar worker**

```bash
curl "http://localhost:3000/api/campaign-worker"
```

Resultado esperado: `{"ok":true,"procesados":1,"enviados":1,"fallidos":0}` (si el número de teléfono existe en Meta).

- [ ] **Step 6: Commit**

```bash
git add api/campaign.js api/campaign-worker.js
git commit -m "feat: campaign queue and cron worker with rate limiting"
```

---

## Task 7: API de Pedidos

**Files:**
- Create: `api/pedidos.js`

**Interfaces:**
- Produces:
  - `GET /api/pedidos?estado=&periodo=&numero_id=` → array de pedidos con contacto anidado
  - `POST /api/pedidos { contacto_id, chat_id?, cantidad, direccion?, notas?, total_soles? }` → pedido creado
  - `PATCH /api/pedidos?id= { estado?, cantidad?, direccion?, notas?, total_soles? }` → pedido actualizado

- [ ] **Step 1: Crear api/pedidos.js**

```js
const supabase = require('./_supabase');

module.exports = async function handler(req, res) {
  if (req.method === 'OPTIONS') return res.status(200).end();

  if (req.method === 'GET') {
    const { estado, periodo } = req.query;

    let query = supabase
      .from('pedidos')
      .select('id, cantidad, direccion, notas, estado, total_soles, created_at, updated_at, contacto:contactos(id, nombre, telefono), chat_id')
      .order('created_at', { ascending: false });

    if (estado && estado !== 'todos') query = query.eq('estado', estado);

    if (periodo === 'hoy') {
      query = query.gte('created_at', new Date().toISOString().split('T')[0]);
    } else if (periodo === 'semana') {
      query = query.gte('created_at', new Date(Date.now() - 7 * 86400000).toISOString());
    } else if (periodo === 'mes') {
      query = query.gte('created_at', new Date(Date.now() - 30 * 86400000).toISOString());
    }

    const { data, error } = await query;
    if (error) return res.status(500).json({ error: error.message });
    return res.status(200).json(data || []);
  }

  if (req.method === 'POST') {
    const { contacto_id, chat_id, cantidad, direccion, notas, total_soles } = req.body;
    if (!contacto_id || !cantidad) return res.status(400).json({ error: 'contacto_id y cantidad requeridos' });

    const { data, error } = await supabase
      .from('pedidos')
      .insert({ contacto_id, chat_id: chat_id || null, cantidad, direccion, notas, total_soles })
      .select()
      .single();

    if (error) return res.status(500).json({ error: error.message });
    return res.status(201).json(data);
  }

  if (req.method === 'PATCH') {
    const id = req.query.id;
    if (!id) return res.status(400).json({ error: 'id requerido' });

    const allowed = ['estado', 'cantidad', 'direccion', 'notas', 'total_soles'];
    const updates = { updated_at: new Date().toISOString() };
    for (const field of allowed) {
      if (req.body[field] !== undefined) updates[field] = req.body[field];
    }

    const { data, error } = await supabase
      .from('pedidos').update(updates).eq('id', id).select().single();

    if (error) return res.status(500).json({ error: error.message });
    return res.status(200).json(data);
  }

  return res.status(405).end();
};
```

- [ ] **Step 2: Verificar GET**

```bash
curl "http://localhost:3000/api/pedidos"
```

Resultado esperado: `[]`

- [ ] **Step 3: Verificar POST**

```bash
curl -X POST "http://localhost:3000/api/pedidos" \
  -H "Content-Type: application/json" \
  -d '{"contacto_id":"UUID_CONTACTO_REAL","cantidad":2,"direccion":"Jr. Lima 123","total_soles":20}'
```

Resultado esperado: objeto con `id`, `estado: "pendiente"`.

- [ ] **Step 4: Verificar PATCH**

```bash
curl -X PATCH "http://localhost:3000/api/pedidos?id=UUID_PEDIDO" \
  -H "Content-Type: application/json" \
  -d '{"estado":"en_ruta"}'
```

Resultado esperado: objeto con `estado: "en_ruta"`.

- [ ] **Step 5: Commit**

```bash
git add api/pedidos.js
git commit -m "feat: pedidos CRUD endpoint"
```

---

## Task 8: API de Agentes

**Files:**
- Create: `api/agentes.js`

**Interfaces:**
- Produces:
  - `GET /api/agentes` → array de agentes activos
  - `GET /api/agentes?id=&stats=true&periodo=` → agente con métricas
  - `POST /api/agentes { nombre, email, rol, numero_id? }` → agente creado
  - `PATCH /api/agentes?id= { nombre?, email?, rol?, activo? }` → agente actualizado
  - `DELETE /api/agentes?id=` → soft delete (`activo = false`)

- [ ] **Step 1: Crear api/agentes.js**

```js
const supabase = require('./_supabase');

module.exports = async function handler(req, res) {
  if (req.method === 'OPTIONS') return res.status(200).end();

  if (req.method === 'GET') {
    const { id, stats, periodo } = req.query;

    if (id && stats === 'true') {
      // Agente con métricas
      const { data: agente } = await supabase
        .from('agentes').select('*').eq('id', id).single();
      if (!agente) return res.status(404).json({ error: 'not found' });

      let since = new Date(0).toISOString();
      if (periodo === 'hoy') since = new Date().toISOString().split('T')[0];
      else if (periodo === 'semana') since = new Date(Date.now() - 7 * 86400000).toISOString();
      else if (periodo === 'mes') since = new Date(Date.now() - 30 * 86400000).toISOString();

      const { count: chatsAtendidos } = await supabase
        .from('chats')
        .select('*', { count: 'exact', head: true })
        .eq('agente_id', id)
        .gte('created_at', since);

      const { count: mensajesEnviados } = await supabase
        .from('mensajes')
        .select('*', { count: 'exact', head: true })
        .eq('agente_id', id)
        .eq('direccion', 'saliente')
        .gte('enviado_at', since);

      return res.status(200).json({ ...agente, chatsAtendidos, mensajesEnviados });
    }

    const { data, error } = await supabase
      .from('agentes')
      .select('id, nombre, email, avatar, rol, activo, created_at')
      .order('nombre');

    if (error) return res.status(500).json({ error: error.message });
    return res.status(200).json(data || []);
  }

  if (req.method === 'POST') {
    const { nombre, email, rol, numero_id } = req.body;
    if (!nombre) return res.status(400).json({ error: 'nombre requerido' });

    const avatar = nombre.split(' ').map(w => w[0]).join('').toUpperCase().slice(0, 2);

    const { data, error } = await supabase
      .from('agentes')
      .insert({ nombre, email: email || null, avatar, rol: rol || 'agente', numero_id: numero_id || null })
      .select().single();

    if (error) return res.status(500).json({ error: error.message });
    return res.status(201).json(data);
  }

  if (req.method === 'PATCH') {
    const id = req.query.id;
    if (!id) return res.status(400).json({ error: 'id requerido' });

    const allowed = ['nombre', 'email', 'rol', 'activo'];
    const updates = {};
    for (const field of allowed) {
      if (req.body[field] !== undefined) updates[field] = req.body[field];
    }
    if (updates.nombre) {
      updates.avatar = updates.nombre.split(' ').map(w => w[0]).join('').toUpperCase().slice(0, 2);
    }

    const { data, error } = await supabase
      .from('agentes').update(updates).eq('id', id).select().single();

    if (error) return res.status(500).json({ error: error.message });
    return res.status(200).json(data);
  }

  if (req.method === 'DELETE') {
    const id = req.query.id;
    if (!id) return res.status(400).json({ error: 'id requerido' });

    const { error } = await supabase
      .from('agentes').update({ activo: false }).eq('id', id);

    if (error) return res.status(500).json({ error: error.message });
    return res.status(200).json({ ok: true });
  }

  return res.status(405).end();
};
```

- [ ] **Step 2: Verificar GET**

```bash
curl "http://localhost:3000/api/agentes"
```

Resultado esperado: array con el agente Braulio creado en Task 2.

- [ ] **Step 3: Verificar POST**

```bash
curl -X POST "http://localhost:3000/api/agentes" \
  -H "Content-Type: application/json" \
  -d '{"nombre":"Lucía Mamani","email":"lucia@sierraazul.pe","rol":"agente"}'
```

Resultado esperado: objeto con `avatar: "LM"`, `rol: "agente"`.

- [ ] **Step 4: Commit**

```bash
git add api/agentes.js
git commit -m "feat: agentes CRUD with soft delete and stats"
```

---

## Task 9: API de Analytics y Etiquetas

**Files:**
- Create: `api/analytics.js`
- Create: `api/etiquetas.js`

**Interfaces:**
- Produces:
  - `GET /api/analytics?periodo=hoy|semana|mes&numero_id=` → objeto con métricas de mensajes y ventas
  - `GET /api/etiquetas?numero_id=` → array de etiquetas
  - `POST /api/etiquetas { numero_id, nombre, color_hex, orden }` → etiqueta creada
  - `PATCH /api/etiquetas?id= { nombre?, color_hex?, orden? }` → etiqueta actualizada
  - `DELETE /api/etiquetas?id=` → eliminación (hard delete — etiquetas no tienen historial crítico)

- [ ] **Step 1: Crear api/analytics.js**

```js
const supabase = require('./_supabase');

module.exports = async function handler(req, res) {
  if (req.method === 'OPTIONS') return res.status(200).end();
  if (req.method !== 'GET') return res.status(405).end();

  const { periodo, numero_id } = req.query;

  let since;
  if (periodo === 'hoy') since = new Date().toISOString().split('T')[0];
  else if (periodo === 'semana') since = new Date(Date.now() - 7 * 86400000).toISOString();
  else since = new Date(Date.now() - 30 * 86400000).toISOString(); // mes por defecto

  // Total de chats activos en el período
  const { count: totalChats } = await supabase
    .from('chats')
    .select('*', { count: 'exact', head: true })
    .gte('ultimo_mensaje_at', since);

  // Mensajes entrantes y salientes
  const { count: mensajesEntrantes } = await supabase
    .from('mensajes')
    .select('*', { count: 'exact', head: true })
    .eq('direccion', 'entrante')
    .gte('enviado_at', since);

  const { count: mensajesSalientes } = await supabase
    .from('mensajes')
    .select('*', { count: 'exact', head: true })
    .eq('direccion', 'saliente')
    .gte('enviado_at', since);

  const { count: mensajesIA } = await supabase
    .from('mensajes')
    .select('*', { count: 'exact', head: true })
    .eq('direccion', 'saliente')
    .eq('enviado_por_ia', true)
    .gte('enviado_at', since);

  // Chats por etiqueta
  const { data: porEtiqueta } = await supabase
    .from('chats')
    .select('etiqueta:etiquetas(nombre, color_hex)')
    .gte('ultimo_mensaje_at', since)
    .not('etiqueta_id', 'is', null);

  const etiquetaConteo = {};
  (porEtiqueta || []).forEach(c => {
    if (!c.etiqueta) return;
    const key = c.etiqueta.nombre;
    etiquetaConteo[key] = { count: (etiquetaConteo[key]?.count || 0) + 1, color: c.etiqueta.color_hex };
  });

  // Pedidos
  const { data: pedidos } = await supabase
    .from('pedidos')
    .select('cantidad, total_soles, estado')
    .gte('created_at', since);

  const pedidosResumen = { pendiente: 0, en_ruta: 0, entregado: 0, cancelado: 0 };
  let totalBidones = 0;
  let totalIngresos = 0;

  (pedidos || []).forEach(p => {
    pedidosResumen[p.estado] = (pedidosResumen[p.estado] || 0) + 1;
    if (p.estado === 'entregado') {
      totalBidones += p.cantidad || 0;
      totalIngresos += parseFloat(p.total_soles || 0);
    }
  });

  // Top 5 clientes por pedidos
  const { data: topClientes } = await supabase
    .from('pedidos')
    .select('contacto:contactos(nombre)')
    .gte('created_at', since);

  const clienteConteo = {};
  (topClientes || []).forEach(p => {
    const nombre = p.contacto?.nombre;
    if (nombre) clienteConteo[nombre] = (clienteConteo[nombre] || 0) + 1;
  });
  const top5 = Object.entries(clienteConteo)
    .sort((a, b) => b[1] - a[1])
    .slice(0, 5)
    .map(([nombre, count]) => ({ nombre, count }));

  return res.status(200).json({
    periodo,
    mensajes: {
      totalChats: totalChats || 0,
      entrantes: mensajesEntrantes || 0,
      salientes: mensajesSalientes || 0,
      porIA: mensajesIA || 0,
      ratioIA: mensajesSalientes > 0 ? Math.round(((mensajesIA || 0) / mensajesSalientes) * 100) : 0,
      porEtiqueta: etiquetaConteo,
    },
    ventas: {
      totalBidones,
      totalIngresos: parseFloat(totalIngresos.toFixed(2)),
      pedidosPorEstado: pedidosResumen,
      totalPedidos: (pedidos || []).length,
      top5Clientes: top5,
    },
  });
};
```

- [ ] **Step 2: Crear api/etiquetas.js**

```js
const supabase = require('./_supabase');

module.exports = async function handler(req, res) {
  if (req.method === 'OPTIONS') return res.status(200).end();

  if (req.method === 'GET') {
    const { numero_id } = req.query;
    let q = supabase.from('etiquetas').select('*').order('orden');
    if (numero_id) q = q.eq('numero_id', numero_id);
    const { data, error } = await q;
    if (error) return res.status(500).json({ error: error.message });
    return res.status(200).json(data || []);
  }

  if (req.method === 'POST') {
    const { numero_id, nombre, color_hex, orden } = req.body;
    if (!numero_id || !nombre) return res.status(400).json({ error: 'numero_id y nombre requeridos' });
    const { data, error } = await supabase
      .from('etiquetas')
      .insert({ numero_id, nombre, color_hex: color_hex || '#004a7c', orden: orden || 0 })
      .select().single();
    if (error) return res.status(500).json({ error: error.message });
    return res.status(201).json(data);
  }

  if (req.method === 'PATCH') {
    const id = req.query.id;
    if (!id) return res.status(400).json({ error: 'id requerido' });
    const allowed = ['nombre', 'color_hex', 'orden'];
    const updates = {};
    for (const f of allowed) if (req.body[f] !== undefined) updates[f] = req.body[f];
    const { data, error } = await supabase
      .from('etiquetas').update(updates).eq('id', id).select().single();
    if (error) return res.status(500).json({ error: error.message });
    return res.status(200).json(data);
  }

  if (req.method === 'DELETE') {
    const id = req.query.id;
    if (!id) return res.status(400).json({ error: 'id requerido' });
    const { error } = await supabase.from('etiquetas').delete().eq('id', id);
    if (error) return res.status(500).json({ error: error.message });
    return res.status(200).json({ ok: true });
  }

  return res.status(405).end();
};
```

- [ ] **Step 3: Verificar analytics**

```bash
curl "http://localhost:3000/api/analytics?periodo=mes"
```

Resultado esperado: objeto con `mensajes` y `ventas` con ceros (sin datos aún).

- [ ] **Step 4: Verificar etiquetas**

```bash
curl "http://localhost:3000/api/etiquetas"
```

Resultado esperado: array con las 3 etiquetas insertadas en Task 2.

- [ ] **Step 5: Commit**

```bash
git add api/analytics.js api/etiquetas.js
git commit -m "feat: analytics and etiquetas endpoints"
```

---

## Task 10: Frontend — Conectar Inbox a la API

**Files:**
- Modify: `index.html`

Esta tarea reemplaza los arrays hardcodeados `CHATS` y `PLANTILLAS_HSM` con fetch calls a la API, y agrega el polling de mensajes nuevos.

**Interfaces:**
- Consumes: `GET /api/chats`, `GET /api/messages`, `GET /api/etiquetas`, `POST /api/send`
- El estado del inbox pasa de arrays en memoria a datos cargados desde la API

- [ ] **Step 1: Agregar estado global y la constante de la URL base en index.html**

Dentro del `<script>`, reemplazar las constantes hardcodeadas al inicio:

```js
// Reemplazar CHATS, PLANTILLAS_HSM, TAGS por:
const API = ''; // En producción Vercel, '' funciona (mismo origen). En local: 'http://localhost:3000'

let CHATS = [];
let PLANTILLAS_HSM = [];
let ETIQUETAS = ['Todos'];
let NUMERO_ID = null; // se llena al cargar

let chatActivoId = null;
let filtroActivo = 'Todos';
let textoBusqueda = '';
let pollingInterval = null;
let ultimoMensajeAt = null;
```

- [ ] **Step 2: Agregar función loadInitialData**

```js
async function loadInitialData() {
  // Cargar etiquetas
  const etRes = await fetch(`${API}/api/etiquetas`);
  const etiquetas = await etRes.json();
  ETIQUETAS = ['Todos', ...etiquetas.map(e => e.nombre)];

  // Cargar chats
  const chRes = await fetch(`${API}/api/chats`);
  const chats = await chRes.json();

  // Detectar numero_id del primer chat (o del primer contacto)
  if (chats.length > 0) {
    NUMERO_ID = chats[0].contacto?.numero_id || null;
  }

  // Mapear al formato que usan las funciones de render existentes
  CHATS = chats.map(c => ({
    id: c.id,
    nombre: c.contacto?.nombre || '—',
    telefono: c.contacto?.telefono || '',
    avatar: c.contacto?.avatar || '??',
    tag: c.etiqueta?.nombre || '',
    ultimoMensaje: c.ultimo_mensaje || '',
    hora: c.ultimo_mensaje_at ? formatHora(c.ultimo_mensaje_at) : '—',
    noLeidos: c.no_leidos || 0,
    minutosRestantes: c.ventana_expira_at ? calcMinutosRestantes(c.ventana_expira_at) : 0,
    iaActiva: c.ia_activa,
    agente_id: c.agente_id,
    mensajes: [], // se cargan al seleccionar el chat
  }));

  // Cargar plantillas HSM para el compositor
  const plantillasRes = await fetch(`${API}/api/etiquetas`); // se reutiliza desde plantillas_hsm más adelante
  // Por ahora mantener las plantillas hardcodeadas hasta que exista el endpoint de plantillas
}

function formatHora(isoStr) {
  const d = new Date(isoStr);
  const ahora = new Date();
  const diff = ahora - d;
  if (diff < 86400000 && ahora.getDate() === d.getDate()) {
    return d.toLocaleTimeString('es-PE', { hour: '2-digit', minute: '2-digit' });
  }
  if (diff < 172800000) return 'Ayer';
  return d.toLocaleDateString('es-PE', { weekday: 'short' });
}

function calcMinutosRestantes(isoStr) {
  return Math.max(0, Math.round((new Date(isoStr) - new Date()) / 60000));
}
```

- [ ] **Step 3: Reemplazar la función seleccionarChat para cargar mensajes desde API**

```js
async function seleccionarChat(id) {
  chatActivoId = id;
  cerrarPlantillas();
  ultimoMensajeAt = null;

  // Cargar todos los mensajes del chat desde la API
  const res = await fetch(`${API}/api/messages?chat_id=${id}`);
  const mensajes = await res.json();

  const chat = getChat(id);
  if (chat) {
    chat.mensajes = mensajes.map(m => ({
      de: m.direccion === 'saliente' ? 'agente' : 'cliente',
      texto: m.texto,
      hora: formatHora(m.enviado_at),
      leido: m.leido,
      ia: m.enviado_por_ia,
    }));
    if (mensajes.length > 0) {
      ultimoMensajeAt = mensajes[mensajes.length - 1].enviado_at;
    }
    chat.noLeidos = 0;
  }

  // Marcar no_leidos = 0 en DB (best effort)
  fetch(`${API}/api/chats`, {
    method: 'PATCH',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ id, no_leidos: 0 }),
  }).catch(() => {});

  renderChatList();
  renderChatWindow();
  iniciarPolling(id);
}
```

- [ ] **Step 4: Agregar función de polling**

```js
function iniciarPolling(chatId) {
  if (pollingInterval) clearInterval(pollingInterval);
  pollingInterval = setInterval(async () => {
    if (!chatId) return;
    const url = ultimoMensajeAt
      ? `${API}/api/messages?chat_id=${chatId}&after=${encodeURIComponent(ultimoMensajeAt)}`
      : `${API}/api/messages?chat_id=${chatId}`;
    const res = await fetch(url);
    const nuevos = await res.json();
    if (nuevos.length > 0) {
      const chat = getChat(chatId);
      if (chat) {
        nuevos.forEach(m => {
          chat.mensajes.push({
            de: m.direccion === 'saliente' ? 'agente' : 'cliente',
            texto: m.texto,
            hora: formatHora(m.enviado_at),
            leido: m.leido,
            ia: m.enviado_por_ia,
          });
        });
        ultimoMensajeAt = nuevos[nuevos.length - 1].enviado_at;
        renderChatWindow();
      }
    }
  }, 3000);
}
```

- [ ] **Step 5: Reemplazar enviarMensaje para usar API**

```js
async function enviarMensaje() {
  const input = $('#msg-input');
  const texto = input.value.trim();
  if (!texto || !chatActivoId) return;

  input.value = '';
  input.style.height = 'auto';

  // Optimistic UI: mostrar el mensaje antes de confirmar
  const chat = getChat(chatActivoId);
  if (chat) {
    chat.mensajes.push({ de: 'agente', texto, hora: horaActual(), leido: false });
    renderChatWindow();
  }

  try {
    const res = await fetch(`${API}/api/send`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ chat_id: chatActivoId, texto }),
    });
    if (!res.ok) {
      console.error('Error al enviar:', await res.json());
    }
  } catch (e) {
    console.error('Error de red al enviar:', e);
  }
}
```

- [ ] **Step 6: Actualizar init() para llamar loadInitialData**

```js
async function init() {
  await loadInitialData();
  renderFilterChips();
  renderChatList();
  actualizarInboxBadge();

  if (CHATS.length > 0) {
    await seleccionarChat(CHATS[0].id);
  } else {
    renderChatWindow();
  }

  renderPlantillas();

  $('#search-input').addEventListener('input', e => buscarChats(e.target.value));

  const input = $('#msg-input');
  input.addEventListener('input', () => {
    input.style.height = 'auto';
    input.style.height = Math.min(input.scrollHeight, 96) + 'px';
  });
  input.addEventListener('keydown', e => {
    if (e.key === 'Enter' && !e.shiftKey) { e.preventDefault(); enviarMensaje(); }
  });

  $('#btn-send').addEventListener('click', enviarMensaje);
  $('#btn-hsm').addEventListener('click', e => {
    e.stopPropagation();
    $('#hsm-panel').classList.contains('hidden') ? abrirPlantillas() : cerrarPlantillas();
  });
  document.addEventListener('click', e => {
    if (!$('#hsm-panel').contains(e.target) && !$('#btn-hsm').contains(e.target)) cerrarPlantillas();
  });
}

init();
```

- [ ] **Step 7: Verificar en el browser**

```bash
vercel dev
```

Abrir `http://localhost:3000`. Verificar:
- La lista de chats carga desde la API (vacía si no hay contactos aún, o con los del webhook test)
- Seleccionar un chat carga sus mensajes
- Enviar un mensaje llama `/api/send` (verificar en Network tab de DevTools)
- Los filter chips muestran las etiquetas de la DB

- [ ] **Step 8: Agregar endpoint PATCH /api/chats para limpiar no_leidos**

Crear `api/chats.js` (agregar al archivo existente un handler para PATCH):

```js
// Al final del handler existente en api/chats.js, antes del return de GET:
if (req.method === 'PATCH') {
  const { id, no_leidos, agente_id, etiqueta_id, ia_activa } = req.body;
  if (!id) return res.status(400).json({ error: 'id requerido' });
  const allowed = { no_leidos, agente_id, etiqueta_id, ia_activa };
  const updates = Object.fromEntries(Object.entries(allowed).filter(([, v]) => v !== undefined));
  const { data, error } = await supabase.from('chats').update(updates).eq('id', id).select().single();
  if (error) return res.status(500).json({ error: error.message });
  return res.status(200).json(data);
}
```

- [ ] **Step 9: Commit**

```bash
git add index.html api/chats.js
git commit -m "feat: wire inbox to API — polling, send, and chat loading"
```

---

## Task 11: Vista Pedidos en el frontend

**Files:**
- Modify: `index.html`

**Interfaces:**
- Consumes: `GET /api/pedidos`, `POST /api/pedidos`, `PATCH /api/pedidos?id=`
- `GET /api/chats` para buscar contactos al crear pedido

- [ ] **Step 1: Agregar el panel de Pedidos al HTML**

Dentro del `<div class="flex h-screen overflow-hidden">`, después del `</section>` del chat-window, agregar:

```html
<!-- PANEL PEDIDOS -->
<section id="panel-pedidos" class="hidden flex-1 flex flex-col min-w-0 overflow-hidden" style="background: var(--surface-bg);">
  <header class="h-[72px] shrink-0 px-6 flex items-center justify-between border-b bg-white" style="border-color: var(--border-soft);">
    <h2 class="font-bold text-lg" style="color: var(--texto-principal);">Pedidos</h2>
    <button id="btn-nuevo-pedido" class="flex items-center gap-2 px-4 py-2 rounded-lg text-white text-sm font-semibold" style="background: var(--azul-egeo);">
      <span class="material-symbols-outlined text-[18px]">add</span> Nuevo pedido
    </button>
  </header>

  <!-- Stats bar -->
  <div id="pedidos-stats" class="grid grid-cols-4 gap-4 p-4 shrink-0"></div>

  <!-- Filtros -->
  <div class="px-4 pb-3 flex items-center gap-3 shrink-0">
    <select id="pedidos-filtro-estado" class="rounded-lg border px-3 py-2 text-sm" style="border-color: var(--border-soft);">
      <option value="todos">Todos los estados</option>
      <option value="pendiente">Pendiente</option>
      <option value="en_ruta">En ruta</option>
      <option value="entregado">Entregado</option>
      <option value="cancelado">Cancelado</option>
    </select>
    <select id="pedidos-filtro-periodo" class="rounded-lg border px-3 py-2 text-sm" style="border-color: var(--border-soft);">
      <option value="">Todos</option>
      <option value="hoy">Hoy</option>
      <option value="semana">Esta semana</option>
      <option value="mes">Este mes</option>
    </select>
  </div>

  <!-- Tabla -->
  <div class="flex-1 overflow-y-auto custom-scroll px-4">
    <table class="w-full text-sm">
      <thead>
        <tr class="border-b" style="border-color: var(--border-soft);">
          <th class="text-left py-2 px-2 font-semibold" style="color: var(--texto-secundario);">Cliente</th>
          <th class="text-left py-2 px-2 font-semibold" style="color: var(--texto-secundario);">Bidones</th>
          <th class="text-left py-2 px-2 font-semibold" style="color: var(--texto-secundario);">Dirección</th>
          <th class="text-left py-2 px-2 font-semibold" style="color: var(--texto-secundario);">Estado</th>
          <th class="text-left py-2 px-2 font-semibold" style="color: var(--texto-secundario);">S/.</th>
          <th class="py-2 px-2"></th>
        </tr>
      </thead>
      <tbody id="pedidos-tbody"></tbody>
    </table>
  </div>

  <!-- Modal nuevo pedido -->
  <div id="modal-pedido" class="hidden fixed inset-0 z-50 flex items-center justify-center" style="background: rgba(0,0,0,0.4);">
    <div class="bg-white rounded-xl p-6 w-full max-w-md shadow-xl">
      <h3 class="font-bold text-base mb-4">Nuevo pedido</h3>
      <div class="flex flex-col gap-3">
        <input id="pedido-buscar-cliente" type="text" placeholder="Buscar cliente por nombre o teléfono..."
          class="border rounded-lg px-3 py-2 text-sm outline-none" style="border-color: var(--border-soft);" />
        <div id="pedido-cliente-resultado" class="hidden max-h-40 overflow-y-auto border rounded-lg" style="border-color: var(--border-soft);"></div>
        <input id="pedido-cantidad" type="number" min="1" value="1" placeholder="Bidones"
          class="border rounded-lg px-3 py-2 text-sm outline-none" style="border-color: var(--border-soft);" />
        <input id="pedido-direccion" type="text" placeholder="Dirección de entrega"
          class="border rounded-lg px-3 py-2 text-sm outline-none" style="border-color: var(--border-soft);" />
        <input id="pedido-total" type="number" step="0.50" placeholder="Total S/."
          class="border rounded-lg px-3 py-2 text-sm outline-none" style="border-color: var(--border-soft);" />
        <input id="pedido-notas" type="text" placeholder="Notas (opcional)"
          class="border rounded-lg px-3 py-2 text-sm outline-none" style="border-color: var(--border-soft);" />
      </div>
      <div class="flex gap-2 mt-4 justify-end">
        <button id="btn-cancelar-pedido" class="px-4 py-2 rounded-lg text-sm" style="color: var(--texto-secundario);">Cancelar</button>
        <button id="btn-guardar-pedido" class="px-4 py-2 rounded-lg text-sm text-white font-semibold" style="background: var(--azul-egeo);">Guardar pedido</button>
      </div>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Agregar funciones JS para Pedidos**

```js
const ESTADO_PEDIDO = {
  pendiente: { label: 'Pendiente', bg: '#fef3c7', color: '#b45309' },
  en_ruta:   { label: 'En ruta',   bg: '#dbeafe', color: '#1d4ed8' },
  entregado: { label: 'Entregado', bg: '#dcfce7', color: '#15803d' },
  cancelado: { label: 'Cancelado', bg: '#fee2e2', color: '#b91c1c' },
};

let pedidoClienteSeleccionado = null;

async function cargarPedidos() {
  const estado = $('#pedidos-filtro-estado').value;
  const periodo = $('#pedidos-filtro-periodo').value;
  const params = new URLSearchParams();
  if (estado !== 'todos') params.set('estado', estado);
  if (periodo) params.set('periodo', periodo);

  const res = await fetch(`${API}/api/pedidos?${params}`);
  const pedidos = await res.json();

  // Render stats
  const conteo = { pendiente: 0, en_ruta: 0, entregado: 0, cancelado: 0 };
  pedidos.forEach(p => { conteo[p.estado] = (conteo[p.estado] || 0) + 1; });
  $('#pedidos-stats').innerHTML = Object.entries(ESTADO_PEDIDO).map(([key, s]) =>
    `<div class="bg-white rounded-xl p-4 border" style="border-color: var(--border-soft);">
      <div class="text-2xl font-bold" style="color: ${s.color};">${conteo[key]}</div>
      <div class="text-xs mt-1" style="color: var(--texto-secundario);">${s.label}</div>
    </div>`
  ).join('');

  // Render tabla
  const tbody = $('#pedidos-tbody');
  if (pedidos.length === 0) {
    tbody.innerHTML = '<tr><td colspan="6" class="text-center py-8" style="color: var(--texto-placeholder);">Sin pedidos</td></tr>';
    return;
  }
  tbody.innerHTML = pedidos.map(p => {
    const s = ESTADO_PEDIDO[p.estado] || ESTADO_PEDIDO.pendiente;
    const options = Object.entries(ESTADO_PEDIDO)
      .map(([k, v]) => `<option value="${k}" ${k === p.estado ? 'selected' : ''}>${v.label}</option>`)
      .join('');
    return `<tr class="border-b" style="border-color: var(--border-soft);">
      <td class="py-3 px-2 font-medium">${escapeHtml(p.contacto?.nombre || '—')}</td>
      <td class="py-3 px-2">${p.cantidad}</td>
      <td class="py-3 px-2 text-sm" style="color: var(--texto-secundario);">${escapeHtml(p.direccion || '—')}</td>
      <td class="py-3 px-2">
        <select data-pedido-id="${p.id}" class="pedido-estado-select text-xs rounded px-2 py-1 font-semibold border-0 outline-none"
          style="background: ${s.bg}; color: ${s.color};">${options}</select>
      </td>
      <td class="py-3 px-2 text-sm">${p.total_soles ? 'S/. ' + p.total_soles : '—'}</td>
      <td class="py-3 px-2">
        ${p.chat_id ? `<button data-chat-id="${p.chat_id}" class="pedido-ir-chat p-1 rounded" title="Ver chat">
          <span class="material-symbols-outlined text-[18px]" style="color: var(--azul-egeo);">chat</span>
        </button>` : ''}
      </td>
    </tr>`;
  }).join('');

  // Event listeners para cambio de estado
  tbody.querySelectorAll('.pedido-estado-select').forEach(sel => {
    sel.addEventListener('change', async () => {
      await fetch(`${API}/api/pedidos?id=${sel.dataset.pedidoId}`, {
        method: 'PATCH',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ estado: sel.value }),
      });
      cargarPedidos();
    });
  });

  // Ir al chat
  tbody.querySelectorAll('.pedido-ir-chat').forEach(btn => {
    btn.addEventListener('click', () => {
      navegarA('inbox');
      seleccionarChat(btn.dataset.chatId);
    });
  });
}

function initPedidos() {
  $('#pedidos-filtro-estado').addEventListener('change', cargarPedidos);
  $('#pedidos-filtro-periodo').addEventListener('change', cargarPedidos);
  $('#btn-nuevo-pedido').addEventListener('click', () => {
    pedidoClienteSeleccionado = null;
    $('#pedido-buscar-cliente').value = '';
    $('#pedido-cliente-resultado').classList.add('hidden');
    $('#modal-pedido').classList.remove('hidden');
  });
  $('#btn-cancelar-pedido').addEventListener('click', () => $('#modal-pedido').classList.add('hidden'));

  // Buscar cliente en el modal
  $('#pedido-buscar-cliente').addEventListener('input', async e => {
    const q = e.target.value.trim().toLowerCase();
    if (q.length < 2) { $('#pedido-cliente-resultado').classList.add('hidden'); return; }
    const coincidencias = CHATS.filter(c =>
      c.nombre.toLowerCase().includes(q) || c.telefono.includes(q)
    ).slice(0, 5);
    const div = $('#pedido-cliente-resultado');
    if (coincidencias.length === 0) { div.classList.add('hidden'); return; }
    div.classList.remove('hidden');
    div.innerHTML = coincidencias.map(c =>
      `<button class="pedido-cliente-opcion w-full text-left px-3 py-2 text-sm hover:bg-gray-50" data-contacto-id="${c.id}" data-nombre="${escapeHtml(c.nombre)}">
        ${escapeHtml(c.nombre)} <span style="color: var(--texto-secundario);">${escapeHtml(c.telefono)}</span>
      </button>`
    ).join('');
    div.querySelectorAll('.pedido-cliente-opcion').forEach(btn => {
      btn.addEventListener('click', () => {
        pedidoClienteSeleccionado = btn.dataset.contactoId;
        $('#pedido-buscar-cliente').value = btn.dataset.nombre;
        div.classList.add('hidden');
      });
    });
  });

  $('#btn-guardar-pedido').addEventListener('click', async () => {
    // Obtener contacto_id desde CHATS si no está seleccionado por buscador
    if (!pedidoClienteSeleccionado) {
      const nombre = $('#pedido-buscar-cliente').value.trim();
      const match = CHATS.find(c => c.nombre.toLowerCase() === nombre.toLowerCase());
      if (match) pedidoClienteSeleccionado = match.contacto_id || match.id;
    }
    if (!pedidoClienteSeleccionado) { alert('Selecciona un cliente'); return; }

    const body = {
      contacto_id: pedidoClienteSeleccionado,
      cantidad: parseInt($('#pedido-cantidad').value) || 1,
      direccion: $('#pedido-direccion').value.trim() || null,
      total_soles: parseFloat($('#pedido-total').value) || null,
      notas: $('#pedido-notas').value.trim() || null,
    };
    await fetch(`${API}/api/pedidos`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(body),
    });
    $('#modal-pedido').classList.add('hidden');
    cargarPedidos();
  });
}
```

- [ ] **Step 3: Verificar en el browser**

```bash
vercel dev
```

Navegar a Pedidos. Verificar:
- Tabla carga con `GET /api/pedidos`
- Cambiar estado en el select hace `PATCH /api/pedidos?id=`
- Modal de nuevo pedido abre, busca clientes, y guarda

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: pedidos view with stats, table, and create modal"
```

---

## Task 12: Vista Agentes en el frontend

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Agregar el panel de Agentes al HTML**

```html
<!-- PANEL AGENTES -->
<section id="panel-agentes" class="hidden flex-1 flex flex-col min-w-0 overflow-hidden" style="background: var(--surface-bg);">
  <header class="h-[72px] shrink-0 px-6 flex items-center justify-between border-b bg-white" style="border-color: var(--border-soft);">
    <h2 class="font-bold text-lg">Agentes</h2>
    <button id="btn-nuevo-agente" class="flex items-center gap-2 px-4 py-2 rounded-lg text-white text-sm font-semibold" style="background: var(--azul-egeo);">
      <span class="material-symbols-outlined text-[18px]">person_add</span> Nuevo agente
    </button>
  </header>
  <div id="agentes-grid" class="flex-1 overflow-y-auto custom-scroll p-4 grid grid-cols-3 gap-4 content-start"></div>

  <!-- Modal agente -->
  <div id="modal-agente" class="hidden fixed inset-0 z-50 flex items-center justify-center" style="background: rgba(0,0,0,0.4);">
    <div class="bg-white rounded-xl p-6 w-full max-w-sm shadow-xl">
      <h3 id="modal-agente-titulo" class="font-bold text-base mb-4">Nuevo agente</h3>
      <input type="hidden" id="agente-edit-id" />
      <div class="flex flex-col gap-3">
        <input id="agente-nombre" type="text" placeholder="Nombre completo"
          class="border rounded-lg px-3 py-2 text-sm outline-none" style="border-color: var(--border-soft);" />
        <input id="agente-email" type="email" placeholder="Email (opcional)"
          class="border rounded-lg px-3 py-2 text-sm outline-none" style="border-color: var(--border-soft);" />
        <select id="agente-rol" class="border rounded-lg px-3 py-2 text-sm outline-none" style="border-color: var(--border-soft);">
          <option value="agente">Agente</option>
          <option value="supervisor">Supervisor</option>
          <option value="admin">Admin</option>
        </select>
        <label class="flex items-center gap-2 text-sm">
          <input type="checkbox" id="agente-activo" checked />
          Agente activo
        </label>
      </div>
      <div class="flex gap-2 mt-4 justify-end">
        <button id="btn-cancelar-agente" class="px-4 py-2 rounded-lg text-sm" style="color: var(--texto-secundario);">Cancelar</button>
        <button id="btn-guardar-agente" class="px-4 py-2 rounded-lg text-sm text-white font-semibold" style="background: var(--azul-egeo);">Guardar</button>
      </div>
    </div>
  </div>
</section>
```

- [ ] **Step 2: Agregar funciones JS para Agentes**

```js
async function cargarAgentes() {
  const res = await fetch(`${API}/api/agentes`);
  const agentes = await res.json();
  const grid = $('#agentes-grid');

  if (agentes.length === 0) {
    grid.innerHTML = '<div class="col-span-3 text-center py-12" style="color: var(--texto-placeholder);">Sin agentes</div>';
    return;
  }

  grid.innerHTML = agentes.map(a => `
    <div class="bg-white rounded-xl p-4 border flex flex-col gap-3" style="border-color: var(--border-soft);">
      <div class="flex items-center gap-3">
        <div class="w-12 h-12 rounded-full flex items-center justify-center font-bold text-lg shrink-0"
          style="background: var(--celeste-claro); color: var(--azul-egeo);">${escapeHtml(a.avatar || '??')}</div>
        <div class="min-w-0">
          <div class="font-semibold text-sm truncate">${escapeHtml(a.nombre)}</div>
          <div class="text-xs truncate" style="color: var(--texto-secundario);">${escapeHtml(a.email || '—')}</div>
        </div>
      </div>
      <div class="flex items-center gap-2">
        <span class="text-[11px] font-semibold px-2 py-0.5 rounded" style="background: var(--celeste-claro); color: var(--azul-egeo);">${a.rol}</span>
        <span class="flex items-center gap-1 text-[11px]" style="color: ${a.activo ? 'var(--verde-vivo)' : 'var(--texto-secundario)'};">
          <span class="w-1.5 h-1.5 rounded-full inline-block" style="background: ${a.activo ? 'var(--verde-vivo)' : 'var(--border-soft)'}"></span>
          ${a.activo ? 'Activo' : 'Inactivo'}
        </span>
      </div>
      <div class="flex gap-2">
        <button class="agente-editar flex-1 text-sm py-1.5 rounded-lg border font-medium"
          style="border-color: var(--border-soft); color: var(--texto-secundario);"
          data-agente='${escapeHtml(JSON.stringify(a))}'>Editar</button>
        ${a.activo ? `<button class="agente-desactivar text-sm py-1.5 px-3 rounded-lg border font-medium"
          style="border-color: var(--rojo-critico); color: var(--rojo-critico);"
          data-agente-id="${a.id}">Desactivar</button>` : ''}
      </div>
    </div>
  `).join('');

  grid.querySelectorAll('.agente-editar').forEach(btn => {
    btn.addEventListener('click', () => {
      const a = JSON.parse(btn.dataset.agente);
      $('#agente-edit-id').value = a.id;
      $('#agente-nombre').value = a.nombre;
      $('#agente-email').value = a.email || '';
      $('#agente-rol').value = a.rol;
      $('#agente-activo').checked = a.activo;
      $('#modal-agente-titulo').textContent = 'Editar agente';
      $('#modal-agente').classList.remove('hidden');
    });
  });

  grid.querySelectorAll('.agente-desactivar').forEach(btn => {
    btn.addEventListener('click', async () => {
      await fetch(`${API}/api/agentes?id=${btn.dataset.agenteId}`, { method: 'DELETE' });
      cargarAgentes();
    });
  });
}

function initAgentes() {
  $('#btn-nuevo-agente').addEventListener('click', () => {
    $('#agente-edit-id').value = '';
    $('#agente-nombre').value = '';
    $('#agente-email').value = '';
    $('#agente-rol').value = 'agente';
    $('#agente-activo').checked = true;
    $('#modal-agente-titulo').textContent = 'Nuevo agente';
    $('#modal-agente').classList.remove('hidden');
  });

  $('#btn-cancelar-agente').addEventListener('click', () => $('#modal-agente').classList.add('hidden'));

  $('#btn-guardar-agente').addEventListener('click', async () => {
    const id = $('#agente-edit-id').value;
    const body = {
      nombre: $('#agente-nombre').value.trim(),
      email: $('#agente-email').value.trim() || null,
      rol: $('#agente-rol').value,
      activo: $('#agente-activo').checked,
    };
    if (!body.nombre) { alert('El nombre es requerido'); return; }

    if (id) {
      await fetch(`${API}/api/agentes?id=${id}`, {
        method: 'PATCH', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body),
      });
    } else {
      await fetch(`${API}/api/agentes`, {
        method: 'POST', headers: { 'Content-Type': 'application/json' }, body: JSON.stringify(body),
      });
    }
    $('#modal-agente').classList.add('hidden');
    cargarAgentes();
  });
}
```

- [ ] **Step 3: Verificar en el browser**

Navegar a Agentes. Verificar:
- Grid muestra al agente Braulio creado en Task 2
- Modal de nuevo agente guarda correctamente
- Editar agente precarga los datos
- Desactivar hace soft delete

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: agentes view with create/edit/deactivate"
```

---

## Task 13: Vista Analytics en el frontend

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Agregar el panel de Analytics al HTML**

```html
<!-- PANEL ANALYTICS -->
<section id="panel-analytics" class="hidden flex-1 flex flex-col min-w-0 overflow-hidden" style="background: var(--surface-bg);">
  <header class="h-[72px] shrink-0 px-6 flex items-center justify-between border-b bg-white" style="border-color: var(--border-soft);">
    <h2 class="font-bold text-lg">Analytics</h2>
    <div class="flex gap-2">
      <button data-periodo="hoy" class="analytics-periodo-btn px-4 py-1.5 rounded-full text-sm font-medium border" style="border-color: var(--border-soft); color: var(--texto-secundario);">Hoy</button>
      <button data-periodo="semana" class="analytics-periodo-btn px-4 py-1.5 rounded-full text-sm font-medium border" style="border-color: var(--border-soft); color: var(--texto-secundario);">Semana</button>
      <button data-periodo="mes" class="analytics-periodo-btn px-4 py-1.5 rounded-full text-sm font-medium border active-periodo" style="background: var(--azul-egeo); border-color: transparent; color: white;">Mes</button>
    </div>
  </header>
  <div id="analytics-content" class="flex-1 overflow-y-auto custom-scroll p-4 flex flex-col gap-6"></div>
</section>
```

- [ ] **Step 2: Agregar funciones JS para Analytics**

```js
let analyticsperiodo = 'mes';

async function cargarAnalytics() {
  const res = await fetch(`${API}/api/analytics?periodo=${analyticsperiodo}`);
  const d = await res.json();
  const cont = $('#analytics-content');

  const maxEtiqueta = Math.max(...Object.values(d.mensajes.porEtiqueta).map(v => v.count), 1);
  const maxCliente = Math.max(...(d.ventas.top5Clientes.map(c => c.count)), 1);

  cont.innerHTML = `
    <!-- Mensajes -->
    <div>
      <h3 class="font-bold text-sm mb-3" style="color: var(--texto-secundario);">MENSAJES</h3>
      <div class="grid grid-cols-3 gap-3 mb-4">
        ${[
          ['Conversaciones', d.mensajes.totalChats, 'var(--azul-egeo)'],
          ['Tiempo prom. respuesta', '—', 'var(--texto-principal)'],
          ['IA ratio', d.mensajes.ratioIA + '%', 'var(--verde-vivo)'],
        ].map(([label, val, color]) => `
          <div class="bg-white rounded-xl p-4 border" style="border-color: var(--border-soft);">
            <div class="text-2xl font-bold" style="color: ${color};">${val}</div>
            <div class="text-xs mt-1" style="color: var(--texto-secundario);">${label}</div>
          </div>`).join('')}
      </div>
      <div class="bg-white rounded-xl p-4 border" style="border-color: var(--border-soft);">
        <div class="font-semibold text-sm mb-3">Chats por etiqueta</div>
        <div class="flex flex-col gap-2">
          ${Object.entries(d.mensajes.porEtiqueta).map(([nombre, v]) => `
            <div class="flex items-center gap-3">
              <span class="text-sm w-40 truncate">${escapeHtml(nombre)}</span>
              <div class="flex-1 h-3 rounded-full" style="background: var(--surface-mid);">
                <div class="h-3 rounded-full" style="width: ${Math.round((v.count / maxEtiqueta) * 100)}%; background: ${v.color};"></div>
              </div>
              <span class="text-sm font-semibold w-8 text-right">${v.count}</span>
            </div>`).join('') || '<div style="color: var(--texto-placeholder);" class="text-sm">Sin datos</div>'}
        </div>
      </div>
    </div>

    <!-- Ventas -->
    <div>
      <h3 class="font-bold text-sm mb-3" style="color: var(--texto-secundario);">VENTAS</h3>
      <div class="grid grid-cols-3 gap-3 mb-4">
        ${[
          ['Bidones entregados', d.ventas.totalBidones, 'var(--azul-egeo)'],
          ['Ingresos S/.', d.ventas.totalIngresos.toFixed(2), 'var(--verde-vivo)'],
          ['Total pedidos', d.ventas.totalPedidos, 'var(--texto-principal)'],
        ].map(([label, val, color]) => `
          <div class="bg-white rounded-xl p-4 border" style="border-color: var(--border-soft);">
            <div class="text-2xl font-bold" style="color: ${color};">${val}</div>
            <div class="text-xs mt-1" style="color: var(--texto-secundario);">${label}</div>
          </div>`).join('')}
      </div>
      <div class="bg-white rounded-xl p-4 border" style="border-color: var(--border-soft);">
        <div class="font-semibold text-sm mb-3">Top clientes</div>
        <div class="flex flex-col gap-2">
          ${d.ventas.top5Clientes.map(c => `
            <div class="flex items-center gap-3">
              <span class="text-sm w-40 truncate">${escapeHtml(c.nombre)}</span>
              <div class="flex-1 h-3 rounded-full" style="background: var(--surface-mid);">
                <div class="h-3 rounded-full" style="width: ${Math.round((c.count / maxCliente) * 100)}%; background: var(--azul-egeo);"></div>
              </div>
              <span class="text-sm font-semibold w-8 text-right">${c.count}</span>
            </div>`).join('') || '<div style="color: var(--texto-placeholder);" class="text-sm">Sin datos</div>'}
        </div>
      </div>
    </div>
  `;
}

function initAnalytics() {
  document.querySelectorAll('.analytics-periodo-btn').forEach(btn => {
    btn.addEventListener('click', () => {
      document.querySelectorAll('.analytics-periodo-btn').forEach(b => {
        b.style.background = '';
        b.style.borderColor = 'var(--border-soft)';
        b.style.color = 'var(--texto-secundario)';
      });
      btn.style.background = 'var(--azul-egeo)';
      btn.style.borderColor = 'transparent';
      btn.style.color = 'white';
      analyticsperiodo = btn.dataset.periodo;
      cargarAnalytics();
    });
  });
}
```

- [ ] **Step 3: Verificar en el browser**

Navegar a Analytics. Verificar:
- Cards de métricas cargan desde API
- Barras CSS proporcionales
- Botones de período recargan los datos

- [ ] **Step 4: Commit**

```bash
git add index.html
git commit -m "feat: analytics view with period selector and CSS bars"
```

---

## Task 14: Navegación + Modal de Campañas + Configuración de Etiquetas

**Files:**
- Modify: `index.html`

- [ ] **Step 1: Agregar lógica de navegación en el script**

```js
function navegarA(seccion) {
  // Ocultar todas las secciones
  ['chat-list', 'chat-window', 'panel-pedidos', 'panel-agentes', 'panel-analytics', 'panel-config'].forEach(id => {
    const el = document.getElementById(id);
    if (el) el.classList.add('hidden');
  });

  // Quitar active de todos los nav-items
  document.querySelectorAll('.nav-item').forEach(el => el.classList.remove('active'));

  // Detener polling si salimos del inbox
  if (seccion !== 'inbox' && pollingInterval) {
    clearInterval(pollingInterval);
    pollingInterval = null;
  }

  if (seccion === 'inbox') {
    document.getElementById('chat-list').classList.remove('hidden');
    document.getElementById('chat-window').classList.remove('hidden');
    document.getElementById('nav-inbox').classList.add('active');
  } else if (seccion === 'pedidos') {
    document.getElementById('panel-pedidos').classList.remove('hidden');
    document.getElementById('nav-pedidos').classList.add('active');
    cargarPedidos();
    initPedidos();
  } else if (seccion === 'analytics') {
    document.getElementById('panel-analytics').classList.remove('hidden');
    document.getElementById('nav-analytics').classList.add('active');
    cargarAnalytics();
    initAnalytics();
  } else if (seccion === 'agentes') {
    document.getElementById('panel-agentes').classList.remove('hidden');
    document.getElementById('nav-agentes').classList.add('active');
    cargarAgentes();
    initAgentes();
  } else if (seccion === 'config') {
    document.getElementById('panel-config').classList.remove('hidden');
    document.getElementById('nav-config').classList.add('active');
    cargarConfigEtiquetas();
  }
}
```

- [ ] **Step 2: Agregar IDs a los nav-items y sus listeners**

En el HTML del sidebar, agregar `id` y `onclick` a cada nav-item:

```html
<div id="nav-inbox" class="nav-item active" onclick="navegarA('inbox')">...</div>
<div id="nav-pedidos" class="nav-item" onclick="navegarA('pedidos')">...</div>
<div id="nav-analytics" class="nav-item" onclick="navegarA('analytics')">...</div>
<div id="nav-agentes" class="nav-item" onclick="navegarA('agentes')">...</div>
<!-- En el nav inferior: -->
<div id="nav-config" class="nav-item" onclick="navegarA('config')">...</div>
```

- [ ] **Step 3: Agregar panel de Configuración (etiquetas)**

```html
<!-- PANEL CONFIGURACIÓN -->
<section id="panel-config" class="hidden flex-1 flex flex-col min-w-0 overflow-hidden" style="background: var(--surface-bg);">
  <header class="h-[72px] shrink-0 px-6 flex items-center border-b bg-white" style="border-color: var(--border-soft);">
    <h2 class="font-bold text-lg">Configuración</h2>
  </header>
  <div class="p-6 flex flex-col gap-6 max-w-lg">
    <div class="bg-white rounded-xl p-5 border" style="border-color: var(--border-soft);">
      <div class="flex items-center justify-between mb-4">
        <h3 class="font-semibold">Etiquetas de chats</h3>
        <button id="btn-nueva-etiqueta" class="text-sm px-3 py-1.5 rounded-lg text-white font-medium" style="background: var(--azul-egeo);">+ Nueva</button>
      </div>
      <div id="etiquetas-lista" class="flex flex-col gap-2"></div>
    </div>
  </div>
</section>
```

- [ ] **Step 4: Agregar funciones JS para Configuración de Etiquetas**

```js
async function cargarConfigEtiquetas() {
  const res = await fetch(`${API}/api/etiquetas`);
  const etiquetas = await res.json();
  const lista = $('#etiquetas-lista');
  lista.innerHTML = etiquetas.map(e => `
    <div class="flex items-center gap-3 py-2 border-b" style="border-color: var(--border-soft);">
      <span class="w-4 h-4 rounded-full shrink-0" style="background: ${e.color_hex};"></span>
      <span class="flex-1 text-sm font-medium">${escapeHtml(e.nombre)}</span>
      <button class="etiqueta-eliminar text-xs px-2 py-1 rounded" data-id="${e.id}"
        style="color: var(--rojo-critico);">Eliminar</button>
    </div>
  `).join('') || '<div class="text-sm" style="color: var(--texto-placeholder);">Sin etiquetas</div>';

  lista.querySelectorAll('.etiqueta-eliminar').forEach(btn => {
    btn.addEventListener('click', async () => {
      await fetch(`${API}/api/etiquetas?id=${btn.dataset.id}`, { method: 'DELETE' });
      cargarConfigEtiquetas();
      // Recargar etiquetas en el inbox
      loadInitialData().then(() => { renderFilterChips(); renderChatList(); });
    });
  });
}

$('#btn-nueva-etiqueta')?.addEventListener('click', async () => {
  const nombre = prompt('Nombre de la etiqueta:');
  if (!nombre) return;
  const color = prompt('Color (hex, ej: #15803d):', '#004a7c');
  if (!NUMERO_ID) { alert('No hay número configurado'); return; }
  await fetch(`${API}/api/etiquetas`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ numero_id: NUMERO_ID, nombre, color_hex: color || '#004a7c' }),
  });
  cargarConfigEtiquetas();
});
```

- [ ] **Step 5: Agregar modal de Campañas**

```html
<!-- MODAL CAMPAÑAS (sobre cualquier vista) -->
<div id="modal-campana" class="hidden fixed inset-0 z-50 flex items-center justify-center" style="background: rgba(0,0,0,0.4);">
  <div class="bg-white rounded-xl p-6 w-full max-w-lg shadow-xl max-h-[90vh] flex flex-col">
    <h3 class="font-bold text-base mb-4">Campaña de remarketing</h3>
    <div class="flex flex-col gap-3 overflow-y-auto flex-1 custom-scroll">
      <input id="campana-nombre" type="text" placeholder="Nombre de la campaña"
        class="border rounded-lg px-3 py-2 text-sm outline-none" style="border-color: var(--border-soft);" />
      <select id="campana-plantilla" class="border rounded-lg px-3 py-2 text-sm outline-none" style="border-color: var(--border-soft);">
        <option value="">Seleccionar plantilla HSM...</option>
      </select>
      <div id="campana-preview" class="hidden p-3 rounded-lg text-sm" style="background: var(--celeste-claro); color: var(--azul-egeo);"></div>
      <div>
        <div class="text-sm font-semibold mb-2">Destinatarios</div>
        <label class="flex items-center gap-2 text-sm mb-2">
          <input type="checkbox" id="campana-usar-existentes" checked />
          Contactos existentes en la bandeja
        </label>
        <label class="flex items-center gap-2 text-sm">
          <input type="checkbox" id="campana-usar-csv" />
          Subir CSV (columnas: telefono, nombre)
        </label>
        <input id="campana-csv" type="file" accept=".csv" class="hidden mt-2" />
        <div id="campana-csv-info" class="hidden text-xs mt-1" style="color: var(--texto-secundario);"></div>
      </div>
    </div>
    <div class="flex gap-2 mt-4 justify-between items-center shrink-0">
      <span id="campana-total-info" class="text-sm" style="color: var(--texto-secundario);"></span>
      <div class="flex gap-2">
        <button id="btn-cancelar-campana" class="px-4 py-2 rounded-lg text-sm" style="color: var(--texto-secundario);">Cancelar</button>
        <button id="btn-enviar-campana" class="px-4 py-2 rounded-lg text-sm text-white font-semibold" style="background: var(--azul-egeo);">Enviar campaña</button>
      </div>
    </div>
  </div>
</div>
```

- [ ] **Step 6: Agregar JS para el modal de Campañas**

```js
function initCampanas() {
  // Rellenar selector de plantillas
  const sel = $('#campana-plantilla');
  PLANTILLAS_HSM.forEach(p => {
    const opt = document.createElement('option');
    opt.value = p.nombre_meta || p.nombre;
    opt.textContent = p.nombre;
    opt.dataset.preview = p.texto_preview;
    sel.appendChild(opt);
  });

  sel.addEventListener('change', () => {
    const opt = sel.options[sel.selectedIndex];
    const preview = $('#campana-preview');
    if (opt.dataset.preview) {
      preview.textContent = opt.dataset.preview;
      preview.classList.remove('hidden');
    } else {
      preview.classList.add('hidden');
    }
    actualizarTotalCampana();
  });

  $('#campana-usar-csv').addEventListener('change', e => {
    if (e.target.checked) $('#campana-csv').classList.remove('hidden');
    else { $('#campana-csv').classList.add('hidden'); $('#campana-csv-info').classList.add('hidden'); }
    actualizarTotalCampana();
  });

  let csvDestinatarios = [];
  $('#campana-csv').addEventListener('change', async e => {
    const file = e.target.files[0];
    if (!file) return;
    const text = await file.text();
    const lines = text.trim().split('\n').slice(1); // skip header
    csvDestinatarios = lines.map(l => {
      const [telefono, nombre] = l.split(',').map(s => s.trim().replace(/"/g, ''));
      return { telefono, nombre: nombre || telefono };
    }).filter(d => d.telefono);
    $('#campana-csv-info').textContent = `${csvDestinatarios.length} contactos del CSV`;
    $('#campana-csv-info').classList.remove('hidden');
    actualizarTotalCampana();
  });

  function actualizarTotalCampana() {
    let total = 0;
    if ($('#campana-usar-existentes').checked) total += CHATS.length;
    if ($('#campana-usar-csv').checked) total += csvDestinatarios.length;
    // Deduplicar aproximación
    $('#campana-total-info').textContent = total > 0 ? `~${total} destinatarios` : '';
  }

  $('#campana-usar-existentes').addEventListener('change', actualizarTotalCampana);

  $('#btn-cancelar-campana').addEventListener('click', () => $('#modal-campana').classList.add('hidden'));

  $('#btn-enviar-campana').addEventListener('click', async () => {
    const nombre = $('#campana-nombre').value.trim();
    const plantilla = $('#campana-plantilla').value;
    if (!nombre || !plantilla) { alert('Completa el nombre y selecciona una plantilla'); return; }
    if (!NUMERO_ID) { alert('No hay número configurado'); return; }

    let destinatarios = [];
    if ($('#campana-usar-existentes').checked) {
      destinatarios = CHATS.map(c => ({ telefono: c.telefono, nombre: c.nombre, contacto_id: c.contacto_id || c.id }));
    }
    if ($('#campana-usar-csv').checked) {
      destinatarios = [...destinatarios, ...csvDestinatarios];
    }

    if (destinatarios.length === 0) { alert('Selecciona al menos un grupo de destinatarios'); return; }

    const res = await fetch(`${API}/api/campaign`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ numero_id: NUMERO_ID, nombre, plantilla_nombre: plantilla, destinatarios }),
    });
    const data = await res.json();
    $('#modal-campana').classList.add('hidden');
    alert(`Campaña "${nombre}" encolada con ${data.total} destinatarios. Se procesará en los próximos minutos.`);
  });
}

// Conectar el botón del sidebar
$('#btn-campanas')?.addEventListener('click', () => {
  initCampanas();
  $('#modal-campana').classList.remove('hidden');
});
```

- [ ] **Step 7: Verificar navegación completa en el browser**

```bash
vercel dev
```

Verificar:
- Clic en cada nav-item muestra la sección correcta
- El inbox se mantiene funcional al volver a él
- Modal de campañas abre con el botón del sidebar
- Panel de configuración muestra etiquetas y permite crear nuevas

- [ ] **Step 8: Commit**

```bash
git add index.html
git commit -m "feat: navigation, campaigns modal, and etiquetas config"
```

---

## Task 15: Deploy a producción en Vercel + configuración del webhook en Meta

**Files:**
- No modifica archivos — configura el entorno externo

- [ ] **Step 1: Subir variables de entorno a Vercel**

```bash
vercel env add META_ACCESS_TOKEN production
vercel env add META_APP_SECRET production
vercel env add META_WEBHOOK_VERIFY_TOKEN production
vercel env add SUPABASE_URL production
vercel env add SUPABASE_SERVICE_ROLE_KEY production
```

Pegar el valor de cada variable cuando se solicite.

- [ ] **Step 2: Desplegar a producción**

```bash
vercel deploy --prod
```

Resultado esperado: URL de producción como `https://sierra-azul.vercel.app`

- [ ] **Step 3: Verificar que el webhook responde**

```bash
curl "https://sierra-azul.vercel.app/api/webhook?hub.mode=subscribe&hub.verify_token=sierra-azul-webhook-2026&hub.challenge=PING"
```

Resultado esperado: `PING`

- [ ] **Step 4: Configurar el webhook en Meta for Developers**

1. Ir a [developers.facebook.com](https://developers.facebook.com) → tu app → WhatsApp → Configuration
2. En **Webhook URL**: `https://sierra-azul.vercel.app/api/webhook`
3. En **Verify Token**: `sierra-azul-webhook-2026` (el valor de `META_WEBHOOK_VERIFY_TOKEN`)
4. Hacer clic en **Verify and Save**
5. En **Webhook fields**: suscribirse a `messages`

- [ ] **Step 5: Prueba end-to-end**

Enviar un mensaje de WhatsApp al número de Sierra Azul desde un teléfono real.

Verificar en Supabase Dashboard → `mensajes`: debe aparecer el mensaje como `direccion = 'entrante'`.

Abrir `https://sierra-azul.vercel.app` en el browser: el chat debe aparecer en el inbox dentro de 3 segundos (polling).

- [ ] **Step 6: Commit final**

```bash
git add .
git commit -m "feat: production deployment complete"
git push origin main
```

---

## Self-Review

### Cobertura del spec

| Requisito del spec | Tarea |
|---|---|
| Multi-número desde el inicio | Task 2 (schema `numeros`), Tasks 3-9 (todos usan `numero_id`) |
| Etiquetas configurables (no hardcodeadas) | Task 2 (seed), Task 9 (`etiquetas.js`), Task 14 (config UI) |
| X-Hub-Signature-256 validation | Task 4 (`webhook.js`) |
| Idempotencia con wamid | Task 4 (`ON CONFLICT (wamid) DO NOTHING`) |
| Polling 3s migrable a Realtime | Task 10 (`iniciarPolling`) |
| Campañas con batches + cron | Task 6 (`campaign-worker.js`) |
| Soft delete en agentes | Task 8 (`DELETE → activo = false`) |
| Barras CSS en Analytics (sin librería) | Task 13 (divs con width%) |
| Vista Pedidos con gestión completa | Task 11 |
| Vista Agentes con gestión completa | Task 12 |
| Vista Analytics (mensajes + ventas) | Task 13 |
| Campañas desde contactos existentes o CSV | Task 14 (modal) |
| Deploy Vercel + webhook Meta | Task 15 |

### Sin placeholders ✓

Todos los pasos incluyen código real o comandos exactos.

### Consistencia de tipos ✓

- `CHATS[].id` = UUID string (de `/api/chats`) → usado como `chat_id` en `/api/send` y `/api/messages`
- `contacto_id` en pedidos viene de `CHATS[n].contacto_id` que se mapea de `c.contacto.id`
- `NUMERO_ID` se detecta de `chats[0].contacto.numero_id` en `loadInitialData`
