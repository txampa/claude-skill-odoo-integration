---
name: odoo-integration
version: "2.0.0"
description: "Use this skill when the user needs to connect to Odoo via XML-RPC or the JSON-2 API (Odoo 19+), migrate an integration from XML-RPC to JSON-2, sync Odoo data to external systems (Google Sheets, databases, APIs), read or write Odoo models (stock.picking, sale.order, purchase.order, product.product, etc.), set up cron-based sync jobs, or deploy Odoo integrations on Railway/Render/Fly.io. Also use when the user mentions albaranes, pedidos, stock, ERP sync, or Odoo automation. Do NOT trigger for generic REST API tasks or non-Odoo ERP systems."
license: MIT. See LICENSE.txt
---

# Odoo Integration Skill

You are an expert in Odoo integrations using Node.js, over XML-RPC (Odoo 8 to 19) and the External JSON-2 API (Odoo 19+). You know the quirks of each version, which field names changed and when, key model names and their states, and how to sync data reliably to external systems.

**First question on any Odoo task: which version?** Field names, states and even the API depend on it (see "Version compatibility" in Common Gotchas). If the user doesn't know, read it from `/web/version` or from the XML-RPC `version()` call before writing code.

---

## Core Architecture

Every Odoo integration follows this structure:

```
odoo-sync/
├── index.js              # Entry point + cron scheduler
├── config/
│   ├── odoo.js           # OdooClient class (XML-RPC, Odoo 8–19)
│   └── odoo-json2.js     # OdooJson2Client class (JSON-2, Odoo 19+), same methods
├── utils/
│   ├── sync.js           # SyncManager (business logic)
│   └── sheets.js         # Output client (Sheets, DB, etc.)
├── test-odoo.js          # Test raw Odoo connection
├── test-sync.js          # Test full sync pipeline
├── debug-states.js       # Inspect model states in prod
├── .env                  # Credentials (never commit)
├── Procfile              # Railway: "web: node index.js"
└── package.json
```

---

## Which API: XML-RPC or JSON-2

| Odoo version | API to use | Notes |
|---|---|---|
| 8 – 18 | XML-RPC (`OdooClient`) | The only external API. JSON-RPC (`/jsonrpc`) also exists but adds nothing for integrations. |
| 19+ | **JSON-2** (`OdooJson2Client`) for new work | XML-RPC still works but is deprecated since 19.0. |
| Odoo 22 (fall 2028) · Odoo Online 21.1 (winter 2027) | JSON-2 only | `/xmlrpc`, `/xmlrpc/2` and `/jsonrpc` are scheduled for removal. |

- **Odoo Online (SaaS):** external API access, old and new, is only available on the **Custom** pricing plan.
  Check the plan before promising an integration.
- **Migrating an existing integration:** both clients below expose the same methods (`authenticate`,
  `searchRead`, `read`, `create`, `write`, `search`). Swap the client, switch the env vars from login/password
  to an API key, re-check field names for the new version, and the business logic stays as it is.
- **JSON-2 runs each call in its own transaction.** Two calls are two commits: if the second fails, the first
  is not rolled back. For multi-step writes, call one server method that does everything (an `action_*`
  method, or a custom one in a small module).

---

## OdooClient — XML-RPC Base Class

Always use this class (or `OdooJson2Client` on Odoo 19+) as the foundation. Never call xmlrpc or fetch directly from business logic.

```javascript
// config/odoo.js
const xmlrpc = require('xmlrpc');

class OdooClient {
  constructor(config = {}) {
    this.host     = config.host     || process.env.ODOO_HOST;
    this.database = config.database || process.env.ODOO_DATABASE;
    this.login    = config.login    || process.env.ODOO_LOGIN;
    this.password = config.password || process.env.ODOO_PASSWORD;
    this.port     = config.port     || 443;
    this.uid      = null;
    this.authenticated = false;

    const clientConfig = { host: this.host, port: this.port };

    this.commonClient = xmlrpc.createSecureClient({
      ...clientConfig, path: '/xmlrpc/2/common'
    });
    this.objectClient = xmlrpc.createSecureClient({
      ...clientConfig, path: '/xmlrpc/2/object'
    });
  }

  async authenticate() {
    return new Promise((resolve, reject) => {
      this.commonClient.methodCall(
        'authenticate',
        [this.database, this.login, this.password, {}],
        (error, uid) => {
          if (error || !uid) {
            reject(error || new Error('Authentication failed — check credentials and database name'));
            return;
          }
          this.uid = uid;
          this.authenticated = true;
          resolve(uid);
        }
      );
    });
  }

  async searchRead(model, domain = [], fields = [], options = {}) {
    if (!this.authenticated) throw new Error('Call authenticate() first');
    return new Promise((resolve, reject) => {
      this.objectClient.methodCall('execute_kw', [
        this.database, this.uid, this.password,
        model, 'search_read', [domain],
        {
          fields: fields.length > 0 ? fields : ['id', 'name'],
          limit: options.limit || 100,
          order: options.order || 'id desc',
          offset: options.offset || 0,
        }
      ], (error, result) => {
        if (error) reject(error);
        else resolve(result || []);
      });
    });
  }

  async read(model, id, fields = []) {
    if (!this.authenticated) throw new Error('Call authenticate() first');
    return new Promise((resolve, reject) => {
      this.objectClient.methodCall('execute_kw', [
        this.database, this.uid, this.password,
        model, 'read', [[id]],
        { fields: fields.length > 0 ? fields : [] }
      ], (error, result) => {
        if (error) reject(error);
        else resolve(result && result.length > 0 ? result[0] : null);
      });
    });
  }

  async create(model, values) {
    if (!this.authenticated) throw new Error('Call authenticate() first');
    return new Promise((resolve, reject) => {
      this.objectClient.methodCall('execute_kw', [
        this.database, this.uid, this.password,
        model, 'create', [values]
      ], (error, result) => {
        if (error) reject(error);
        else resolve(result); // returns new record ID
      });
    });
  }

  async write(model, ids, values) {
    if (!this.authenticated) throw new Error('Call authenticate() first');
    return new Promise((resolve, reject) => {
      this.objectClient.methodCall('execute_kw', [
        this.database, this.uid, this.password,
        model, 'write', [ids, values]
      ], (error, result) => {
        if (error) reject(error);
        else resolve(result); // returns true
      });
    });
  }

  async search(model, domain = [], options = {}) {
    if (!this.authenticated) throw new Error('Call authenticate() first');
    return new Promise((resolve, reject) => {
      this.objectClient.methodCall('execute_kw', [
        this.database, this.uid, this.password,
        model, 'search', [domain],
        { limit: options.limit || 100 }
      ], (error, result) => {
        if (error) reject(error);
        else resolve(result || []); // returns array of IDs
      });
    });
  }

  // { field_name: { type, store, ... } } — use it to pick field names for the server's version
  async fieldsGet(model, attributes = ['type', 'store', 'string']) {
    if (!this.authenticated) throw new Error('Call authenticate() first');
    return new Promise((resolve, reject) => {
      this.objectClient.methodCall('execute_kw', [
        this.database, this.uid, this.password,
        model, 'fields_get', [],
        { attributes }
      ], (error, result) => {
        if (error) reject(error);
        else resolve(result);
      });
    });
  }
}

module.exports = OdooClient;
```

---

## OdooJson2Client — JSON-2 API (Odoo 19+)

Same methods as `OdooClient`, no dependencies (Node 18+ `fetch`). Every call is
`POST /json/2/<model>/<method>` with the API key as a bearer token and the method's arguments **by name**
(no positional arguments: the names must match the Python signature, e.g. `vals_list` for `create`).

```javascript
// config/odoo-json2.js
class OdooJson2Client {
  constructor(config = {}) {
    this.baseUrl  = (config.baseUrl || process.env.ODOO_URL || '').replace(/\/+$/, ''); // https://erp.example.com
    this.apiKey   = config.apiKey   || process.env.ODOO_API_KEY;
    // Only needed when several databases share the same host and dbfilter can't pick one
    this.database = config.database || process.env.ODOO_DATABASE;
    this.uid = null;
    this.authenticated = false;
    if (!this.baseUrl || !this.apiKey) throw new Error('ODOO_URL and ODOO_API_KEY are required');
  }

  async call(model, method, body = {}) {
    const headers = {
      'Authorization': `bearer ${this.apiKey}`,
      'Content-Type': 'application/json; charset=utf-8',
      'User-Agent': 'odoo-sync',
    };
    if (this.database) headers['X-Odoo-Database'] = this.database;

    const res = await fetch(`${this.baseUrl}/json/2/${model}/${method}`, {
      method: 'POST', headers, body: JSON.stringify(body),
    });
    const text = await res.text();
    let data;
    try {
      data = text ? JSON.parse(text) : null;
    } catch {
      throw new Error(`Odoo ${model}.${method}: HTTP ${res.status}, response is not JSON: ${text.slice(0, 200)}`);
    }
    if (!res.ok) {
      // Odoo sends { name, message, arguments, context, debug } on 4xx/5xx
      const err = new Error(`Odoo ${model}.${method}: HTTP ${res.status} ${data?.name ?? ''}: ${data?.message ?? text.slice(0, 200)}`);
      err.status = res.status;
      err.odoo = data;
      throw err;
    }
    return data;
  }

  // JSON-2 has no login step: the API key identifies the user. This checks the key and stores the uid.
  async authenticate() {
    const ctx = await this.call('res.users', 'context_get');
    this.uid = ctx.uid;
    this.authenticated = true;
    return this.uid;
  }

  searchRead(model, domain = [], fields = [], options = {}) {
    return this.call(model, 'search_read', {
      domain,
      fields: fields.length > 0 ? fields : ['id', 'name'],
      limit: options.limit || 100,
      order: options.order || 'id desc',
      offset: options.offset || 0,
    });
  }

  async read(model, id, fields = []) {
    const rows = await this.call(model, 'read', { ids: [id], fields: fields.length > 0 ? fields : null });
    return rows.length > 0 ? rows[0] : null;
  }

  async create(model, values) {
    const ids = await this.call(model, 'create', { vals_list: [values] });
    return ids[0]; // methods that return records come back as a list of ids
  }

  write(model, ids, values) {
    return this.call(model, 'write', { ids, vals: values }); // returns true
  }

  search(model, domain = [], options = {}) {
    return this.call(model, 'search', { domain, limit: options.limit || 100 }); // returns array of IDs
  }

  fieldsGet(model, attributes = ['type', 'store', 'string']) {
    return this.call(model, 'fields_get', { attributes });
  }
}

module.exports = OdooJson2Client;
```

**API keys** (Preferences → Account Security → New API Key, or `res.users.apikeys.generate` from code):
- Last **3 months at most**. Put the expiry date in your calendar or rotate them from code.
- Shown only once. Create one per integration, on a dedicated user with the minimum access rights.
- A wrong or expired key returns `HTTP 401 werkzeug.exceptions.Unauthorized: Invalid apikey`.

---

## Key Odoo Models

### stock.picking — Albaranes (Delivery/Transfer Orders)

**States:**
| Value | Label |
|-------|-------|
| `draft` | Borrador |
| `waiting` | En espera (waiting for another operation) |
| `confirmed` | Esperando disponibilidad (waiting for stock) |
| `assigned` | Listo para transferir |
| `done` | Hecho |
| `cancel` | Cancelado |

**Key fields** (asking for a field that doesn't exist in that version makes the whole call fail):
```javascript
// Odoo 16+
['id', 'name', 'state', 'date', 'move_ids', 'origin',
 'partner_id', 'picking_type_id', 'scheduled_date']
// Odoo 11–15: 'move_lines' instead of 'move_ids'
// Odoo 8–10:  'move_lines' and 'min_date' instead of 'scheduled_date'
```

**Typical domain — "Esperando Disponibilidad":**
```javascript
[['state', 'in', ['waiting', 'confirmed']]]
```

**Note on `origin` field:** Contains the procurement group reference, e.g. `"WH/OUT/259023: Sale Order SO-1234"`. To extract the group ID: `origin.split(':')[0].trim()`.

---

### stock.move — Líneas de movimiento

**States:**
| Value | Label |
|-------|-------|
| `draft` | Nuevo |
| `waiting` | En espera (waiting for move) |
| `confirmed` | Esperando disponibilidad |
| `assigned` | Disponible |
| `done` | Hecho |
| `cancel` | Cancelado |

**Key fields:**
```javascript
// Odoo 17+
['id', 'product_id', 'state', 'product_uom_qty',
 'quantity', 'name', 'picking_id']
// Odoo 8–16: 'quantity_done' instead of 'quantity'
```

**To get lines for a picking:**
```javascript
const moveLines = await odoo.searchRead(
  'stock.move',
  [['picking_id', '=', picking.id]], // works in every version, no need to know move_ids vs move_lines
  ['id', 'product_id', 'state', 'product_uom_qty', 'name']
);
// Filter by state:
const waitingLines = moveLines.filter(l => l.state === 'confirmed');
```

**Product name cleanup (remove SKU codes like `[SKU-123]`):**
```javascript
const cleanName = product.replace(/^\s*\[.*?\]\s*/, '').trim();
```

---

### sale.order — Pedidos de Venta

**States:**
| Value | Label |
|-------|-------|
| `draft` | Presupuesto |
| `sent` | Presupuesto enviado |
| `sale` | Pedido de venta |
| `done` | Bloqueado (**Odoo 8–16 only**) |
| `cancel` | Cancelado |

**Odoo 17+:** there is no `done` state. A locked order stays in `sale` with the boolean `locked = true`.

**Key fields:**
```javascript
['id', 'name', 'state', 'date_order', 'partner_id',
 'amount_total', 'order_line', 'picking_ids']
// Odoo 17+: add 'locked' if you need to know whether it's locked
```

---

### purchase.order — Pedidos de Compra

**States:** `draft`, `sent`, `to approve`, `purchase`, `done`, `cancel`

**Odoo 19+:** `done` is gone, same change as sale.order: locked orders stay in `purchase` with `locked = true`.

**Key fields:**
```javascript
['id', 'name', 'state', 'date_order', 'partner_id',
 'amount_total', 'order_line', 'picking_ids']
```

---

### product.product / product.template

```javascript
// product.product = variant (has stock)
// product.template = template (groups variants)
const products = await odoo.searchRead(
  'product.product',
  [['active', '=', true]],
  ['id', 'name', 'default_code', 'list_price', 'qty_available']
);
```

**Stockable products changed in Odoo 18:**
```javascript
// Odoo 8–17: stockable = type 'product'
[['type', '=', 'product']]
// Odoo 18+: type is 'consu' | 'service' | 'combo', stock tracking is a separate boolean
[['type', '=', 'consu'], ['is_storable', '=', true]]
```

---

### res.partner — Clientes/Proveedores

```javascript
// Odoo 13+
['id', 'name', 'email', 'phone', 'street', 'city',
 'country_id', 'customer_rank', 'supplier_rank']
// Odoo 8–12: booleans 'customer' and 'supplier' instead of the *_rank fields
// Odoo 19+: 'mobile' no longer exists (only 'phone')
```

---

## SyncManager — Incremental Sync with Deduplication

The canonical pattern for reliable, idempotent syncs:

```javascript
// utils/sync.js
class SyncManager {
  constructor() {
    this.odoo   = new OdooClient();
    this.output = new OutputClient(); // Sheets, DB, etc.
  }

  async sync() {
    await this.odoo.authenticate();
    await this.output.authenticate();

    // 1. Load already-synced keys (deduplication)
    const existingKeys = await this.output.getExistingKeys();

    // 2. Get last synced date (incremental — avoids full scans)
    const lastDate = await this.output.getLastDate();

    // 3. Build domain dynamically
    const domain = [['state', 'in', ['waiting', 'confirmed']]];
    if (lastDate) {
      domain.push(['date', '>=', lastDate]);
    }

    // 4. Fetch from Odoo — field name depends on the version (move_ids since 16)
    const pickingFields = await this.odoo.fieldsGet('stock.picking');
    const LINES = 'move_ids' in pickingFields ? 'move_ids' : 'move_lines';
    const LIMIT = 1000;
    const records = await this.odoo.searchRead(
      'stock.picking', domain,
      ['id', 'name', 'state', 'date', LINES, 'origin'],
      { limit: LIMIT, order: 'date asc' }
    );
    if (records.length === LIMIT) {
      // More records than the limit: with 'date asc' the newest ones were left out. Page with offset or narrow the domain.
      console.warn(`stock.picking: got exactly ${LIMIT} records, some were not read`);
    }

    // 5. Process and deduplicate
    const rows = [];
    for (const record of records) {
      const key = extractKey(record); // e.g. procurement group
      if (existingKeys.has(key)) continue;

      const detail = await this.odoo.read(
        'stock.picking', record.id,
        ['id', 'name', 'date', LINES, 'origin']
      );
      if (!detail) continue;

      // ... build rows ...
      rows.push(buildRow(detail));
    }

    // 6. Write output
    if (rows.length > 0) {
      await this.output.appendRows(rows);
    }

    return { success: true, rowsAdded: rows.length };
  }
}
```

---

## Google Sheets Output Client

```javascript
// utils/sheets.js
const { google } = require('googleapis');

class SheetsClient {
  constructor() {
    this.spreadsheetId = process.env.GOOGLE_SHEETS_ID;
    this.sheetName     = process.env.GOOGLE_SHEET_NAME || 'Hoja1';
    this.sheets        = null;
  }

  async authenticate() {
    // Supports both: local credentials.json file OR GOOGLE_CREDENTIALS env var (Railway)
    let auth;
    if (process.env.GOOGLE_CREDENTIALS) {
      const credentials = JSON.parse(process.env.GOOGLE_CREDENTIALS);
      auth = new google.auth.GoogleAuth({
        credentials,
        scopes: ['https://www.googleapis.com/auth/spreadsheets']
      });
    } else {
      auth = new google.auth.GoogleAuth({
        keyFile: './credentials.json',
        scopes: ['https://www.googleapis.com/auth/spreadsheets']
      });
    }
    this.sheets = google.sheets({ version: 'v4', auth });
  }

  async appendRows(rows) {
    // Read column A to find true last row (works even with active filters)
    const col = await this.sheets.spreadsheets.values.get({
      spreadsheetId: this.spreadsheetId,
      range: `${this.sheetName}!A:A`
    });
    const lastRow = (col.data.values || []).length;
    const range   = `${this.sheetName}!A${lastRow + 1}:Z${lastRow + rows.length}`;

    await this.sheets.spreadsheets.values.update({
      spreadsheetId: this.spreadsheetId,
      range,
      valueInputOption: 'RAW',
      resource: { values: rows }
    });
  }

  async getExistingKeys() {
    // Column B = deduplication key (e.g. procurement group)
    const data = await this._readColumn('B');
    const keys = new Set();
    data.slice(1).forEach(row => { if (row[0]) keys.add(row[0].toString()); });
    return keys;
  }

  async getLastDate() {
    // Column A = date. Walk backwards for last non-empty value.
    const data = await this._readColumn('A');
    for (let i = data.length - 1; i >= 1; i--) {
      if (data[i] && data[i][0]) return data[i][0];
    }
    return null;
  }

  async _readColumn(col) {
    const res = await this.sheets.spreadsheets.values.get({
      spreadsheetId: this.spreadsheetId,
      range: `${this.sheetName}!${col}:${col}`
    });
    return res.data.values || [];
  }
}

module.exports = SheetsClient;
```

---

## Cron Scheduler (index.js)

```javascript
const cron = require('node-cron');
const SyncManager = require('./utils/sync');

const syncManager = new SyncManager();

// Run on start
(async () => {
  console.log('Running initial sync...');
  await syncManager.sync();
})();

// Schedule: every 15 min, Mon-Fri, 6am-10pm
// Adjust HOURS_START / HOURS_END via env vars if needed
const start = process.env.HOURS_START || 6;
const end   = process.env.HOURS_END   || 22;

cron.schedule(`*/15 ${start}-${end} * * 1-5`, async () => {
  console.log(`[${new Date().toLocaleString()}] Running sync...`);
  await syncManager.sync();
});

process.on('SIGTERM', () => process.exit(0));
```

**Cron patterns reference:**
```
*/15 6-22 * * *      → every 15 min, 6am–10pm, all days
0    8-17 * * 1-5    → hourly, 8am–5pm, Mon–Fri
30   6-15 * * *      → at :30 of each hour, 6am–3pm
0,30 6-15 * * *      → every 30 min, 6am–3pm
```

---

## Environment Variables

```env
# Odoo — XML-RPC (OdooClient)
ODOO_HOST=erp.yourcompany.com
ODOO_DATABASE=production_db
ODOO_LOGIN=sync_user@company.com
ODOO_PASSWORD=your_password   # an API key also works here instead of the password
ODOO_PORT=443

# Odoo — JSON-2, Odoo 19+ (OdooJson2Client)
ODOO_URL=https://erp.yourcompany.com
ODOO_API_KEY=your_api_key     # expires in 3 months at most
# ODOO_DATABASE only if several databases share the host

# Google Sheets
GOOGLE_SHEETS_ID=1abc...xyz
GOOGLE_SHEET_NAME=Marzo 2026
# On Railway: paste full credentials.json content as single line
GOOGLE_CREDENTIALS={"type":"service_account","project_id":"..."}

# Scheduler
HOURS_START=6
HOURS_END=22
```

**Monthly sheet rotation:** Just change `GOOGLE_SHEET_NAME` in `.env` (or Railway env vars) at the start of each month. No code changes needed.

---

## Railway Deployment

**Procfile:**
```
web: node index.js
```

**package.json engines:**
```json
{
  "engines": { "node": ">=18.0.0" }
}
```

**Steps:**
1. Push to GitHub
2. Connect repo in Railway
3. Set env vars in Railway dashboard (Settings → Variables)
4. For `GOOGLE_CREDENTIALS`: copy the full contents of `credentials.json`, minify to single line, paste as value
5. Deploy — Railway auto-restarts on crash

**NEVER commit:** `.env`, `credentials.json`

---

## Debug Scripts

Always include these in the project. Run them locally to diagnose production issues.

### test-odoo.js — Verify raw connection
```javascript
require('dotenv').config();
const OdooClient = require('./config/odoo');

(async () => {
  const odoo = new OdooClient();
  await odoo.authenticate();
  console.log('UID:', odoo.uid);

  const pickings = await odoo.searchRead(
    'stock.picking', [],
    ['id', 'name', 'state', 'date'],
    { limit: 5 }
  );
  console.log('Sample pickings:', JSON.stringify(pickings, null, 2));
})();
```

### debug-states.js — Discover real state values in your instance
```javascript
require('dotenv').config();
const OdooClient = require('./config/odoo');

(async () => {
  const odoo = new OdooClient();
  await odoo.authenticate();

  const records = await odoo.searchRead(
    'stock.picking', [],
    ['id', 'name', 'state', 'date'],
    { limit: 100 }
  );

  const states = {};
  records.forEach(r => {
    if (!states[r.state]) states[r.state] = 0;
    states[r.state]++;
  });
  console.log('States found:', states);
})();
```

### debug-move-lines.js — Inspect lines inside a specific picking
```javascript
require('dotenv').config();
const OdooClient = require('./config/odoo');

const PICKING_ID = 1341658; // replace with target ID

(async () => {
  const odoo = new OdooClient();
  await odoo.authenticate();

  const picking = await odoo.read(
    'stock.picking', PICKING_ID,
    ['id', 'name', 'origin']
  );
  console.log('Picking:', picking.name);

  const lines = await odoo.searchRead(
    'stock.move',
    [['picking_id', '=', PICKING_ID]], // same in every version
    ['id', 'product_id', 'state', 'product_uom_qty', 'name']
  );

  const byState = {};
  lines.forEach(l => {
    if (!byState[l.state]) byState[l.state] = [];
    byState[l.state].push(l.product_id[1] || l.name);
  });
  console.log('Lines by state:', JSON.stringify(byState, null, 2));
})();
```

---

## Common Gotchas

### Many2one fields return `[id, name]` arrays
```javascript
// product_id = [42, "Bike Model XR"]
const name = line.product_id[1]; // ✅
const id   = line.product_id[0]; // ✅
const name = line.product_id;    // ❌ returns array, not string
```

### Version compatibility — field name differences

The XML-RPC protocol is the same from Odoo 8 to 19. What changes is field names, states and models
(checked against the Odoo source of each branch):

| What | Before | After | Since |
|------|--------|-------|-------|
| Picking lines | `stock.picking.move_lines` | `move_ids` | **16** |
| Picking scheduled date | `min_date` | `scheduled_date` | **11** |
| Detailed operations | `pack_operation_ids` (`stock.pack.operation`) | `move_line_ids` (`stock.move.line`) | **11** |
| Done quantity on a move | `stock.move.quantity_done` | `quantity` | **17** |
| Stockable product | `type = 'product'` | `type = 'consu'` + `is_storable = true` | **18** |
| Invoices | `account.invoice` | `account.move` | **13** |
| Customer / supplier flag | booleans `customer`, `supplier` | `customer_rank`, `supplier_rank` | **13** |
| Locked sale order | state `done` | state `sale` + `locked = true` | **17** |
| Locked purchase order | state `done` | state `purchase` + `locked = true` | **19** |
| `stock.picking.move_ids_without_package` | exists (12–18) | removed | **19** |
| `res.partner.mobile` | exists | removed | **19** |

**Don't "fall back" by asking for both names.** Requesting a field that doesn't exist in that version
makes the whole call fail. Ask the server which fields it has, once, and build the field list from that:

```javascript
const pickingFields = await odoo.fieldsGet('stock.picking');
const LINES = 'move_ids' in pickingFields ? 'move_ids' : 'move_lines';
const DATE  = 'scheduled_date' in pickingFields ? 'scheduled_date' : 'min_date';

const pickings = await odoo.searchRead('stock.picking', domain, ['id', 'name', 'state', LINES, DATE]);
```

When the user tells you their version, use the exact names from the table and skip the detection.

### Dates and datetimes are UTC strings
Odoo returns datetimes as naive **UTC** strings (`"2026-03-25 14:30:00"`), never in the user's timezone.
Compare `write_date` in UTC, and convert external timestamps to UTC before writing them.
```javascript
const dateOnly = picking.date.split(' ')[0];                    // "2026-03-25" (UTC date)
const asDate   = new Date(picking.date.replace(' ', 'T') + 'Z'); // real Date object
```

### Empty values are `false`, not `null`
An empty many2one, char or date comes back as `false`. Coerce explicitly (`line.product_id ? line.product_id[0] : null`,
`partner.email || ''`) and drop non-integer values from id lists before using them in an `in` domain.

### searchRead limit can silently cut the newest records
Default limit in OdooClient is 100. With `order: 'date asc'` and more matching records than the limit, the ones
you lose are the newest, and nothing warns you. For production syncs either page through everything
(`order: 'id'`, `offset` += `limit`, until a page comes back shorter than `limit`) or log a warning when
`result.length === limit`. Pages of 1,000–2,000 records per call work well.

### `read` fails where `search_read` stays silent
If a record rule hides one of the ids, `read(ids)` raises an AccessError for the whole batch, while
`search_read([['id', 'in', ids]])` just leaves that record out. Pick one on purpose, and log when fewer records
come back than you asked for.

### A domain on a non-stored computed field can be ignored
In older versions (seen on Odoo 8 with `stock.picking.sale_id`), filtering on a computed field that isn't stored
and has no search method returns unfiltered results instead of an error. Check `fieldsGet(model)[field].store`
before filtering on it, and filter on a stored field instead (for pickings of a sale order: `group_id`).

### Creating records over the API skips the UI
- **Onchanges don't run.** Whatever the form fills in when you pick a customer or a product (payment terms,
  fiscal position, taxes, UoM…) must be sent explicitly. Read the record back after `create` and compare.
- **Defaults are not what users expect.** Send every field you care about; don't rely on defaults.
- **Automated actions and crons still fire** on records created over the API (emails, workflows, auto-confirm).
  List `ir.cron` and the automated actions before the first write.
- **Totals may differ by a cent.** Check whether taxes are included in prices (`account.tax.price_include`) and the
  company's `tax_calculation_rounding_method` before comparing totals with the source system.

### Idempotent creates
Before creating, search for the record by an external reference you control (`origin`, `ref` or a deterministic
`name`) in **any** state, with context `{ active_test: false }` so archived records are found too. Never run two
writers on the same model at the same time.

### Incremental sync by `write_date`
Save the last run time, subtract a safety overlap (e.g. 15 minutes) and query `write_date >= since`. Stock moves
update `stock.quant`, not the product: to catch stock changes, query both. Keep a list of failed pushes to retry,
because a failure on the other system doesn't change anything in Odoo and the next run won't see it.

### Check permissions before the first run
On Odoo 8–19, `check_access_rights(operation, raise_exception=False)` on each model tells you whether the
integration user can `read`/`create`/`write` before anything breaks halfway (Odoo 18+ also has `has_access`).

### Retries and proxies
- Retry **reads** on network errors with backoff. Never retry **writes** automatically: you may create duplicates.
- Some reverse proxies drop requests with non-browser or empty User-Agents. If calls hang or close without a
  response, set an explicit User-Agent.
- Put every write behind one switch in the client so a dry run is the default.

### Authentication returns `false` (not an error)
When `uid === false`, credentials are wrong or the database name is incorrect. The `xmlrpc` library will NOT throw — it returns `false`. Always check `if (!uid)`.

### HTTPS vs HTTP
Most hosted Odoo instances use port 443 with HTTPS. Local/dev instances often use port 8069 with HTTP:
```javascript
// HTTPS (production):
xmlrpc.createSecureClient({ host, port: 443, path })

// HTTP (local dev):
xmlrpc.createClient({ host, port: 8069, path })
```

### Always pass the field list
Called with an empty field list, Odoo returns **every** field of the model (slow and heavy). These clients
default to `['id', 'name']` instead, so always pass the fields you need:
```javascript
// Wrong — the client only returns id and name:
await odoo.searchRead('stock.picking', [], []);

// Correct:
await odoo.searchRead('stock.picking', [], ['id', 'name', 'state', 'date', 'origin']);
```

---

## packages.json dependencies

```json
{
  "dependencies": {
    "googleapis": "^118.0.0",
    "node-cron": "^3.0.2",
    "xmlrpc": "^1.3.2",
    "dotenv": "^16.0.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  },
  "engines": {
    "node": ">=18.0.0"
  }
}
```

---

## Security

### .gitignore — mandatory
Every project must have this before the first `git add`:
```
.env
credentials.json
*.env*
```
Never put credentials in documentation files (.md, .txt, .doc). Use `.env.example` with placeholder values only.

### Odoo user — principle of least privilege
Create a dedicated sync user in Odoo with read-only access restricted to the models the integration needs. Never use an admin account for automated scripts.

```
Settings → Users → New User
- Access Rights: set to minimum needed (e.g. Inventory / Read only)
- Do NOT use admin credentials
```

If the credentials are ever compromised, a read-only restricted user limits the blast radius to data exposure — not data modification or deletion.

Prefer an **API key** over the user's password, on XML-RPC too (it goes in the password parameter). A key can be
revoked without touching the user's login, and on JSON-2 it is the only option.

### Google Service Account — restrict scope
The service account should only have `Editor` access to the specific spreadsheet, not to the entire Google Drive. Share the sheet with the service account email directly, not at folder/Drive level.

### Railway / deployment secrets
- Use platform env vars (Railway, Render, Fly.io) — never commit secrets
- Rotate credentials if they were ever committed to git, even briefly
- `GOOGLE_CREDENTIALS` should be the minified single-line JSON of the service account — check no newlines crept in

### What this integration does NOT expose
- No HTTP server or public endpoints — the attack surface is zero from the internet
- No user input flows into Odoo queries — XML-RPC injection is not a realistic vector
- No `eval()`, no `exec()`, no dynamic code execution

---

## Checklist for every new Odoo integration

- [ ] Odoo version known (`/web/version`), and field names checked against the version table or `fieldsGet`
- [ ] Odoo 19+: new integrations use JSON-2; existing XML-RPC ones have a migration date before Odoo 22 / Online 21.1
- [ ] API key expiry date written down (3 months max)
- [ ] Odoo Online: the customer is on the Custom plan (no external API otherwise)
- [ ] Crons and automated actions that fire on create/write reviewed before the first write
- [ ] `.gitignore` exists and covers `.env`, `credentials.json`, `*.env*`
- [ ] No real credentials or secrets in any `.md` or documentation file
- [ ] Odoo sync user is read-only and restricted to needed models only
- [ ] Google service account has access only to the specific spreadsheet
- [ ] `node test-odoo.js` authenticates and returns real data
- [ ] Run `debug-states.js` to confirm actual state values before writing domain filters
- [ ] `Procfile` exists for Railway deploy
- [ ] `GOOGLE_SHEET_NAME` is configurable via env var (not hardcoded)
- [ ] Deduplication key is chosen and persisted in a queryable column
- [ ] `limit` in searchRead is set high enough for production volume
- [ ] Error handling in sync loop uses `continue` (one bad record should not abort the batch)
