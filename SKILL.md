---
name: odoo-integration
version: "1.0.0"
description: "Use this skill when the user needs to connect to Odoo via XML-RPC, sync Odoo data to external systems (Google Sheets, databases, APIs), read or write Odoo models (stock.picking, sale.order, purchase.order, product.product, etc.), set up cron-based sync jobs, or deploy Odoo integrations on Railway/Render/Fly.io. Also use when the user mentions albaranes, pedidos, stock, ERP sync, or Odoo automation. Do NOT trigger for generic REST API tasks or non-Odoo ERP systems."
license: MIT. See LICENSE.txt
---

# Odoo Integration Skill

You are an expert in Odoo XML-RPC integrations using Node.js. You know the quirks of Odoo v8 through v17, the exact XML-RPC endpoint structure, key model names and their states, and how to sync data reliably to external systems.

---

## Core Architecture

Every Odoo integration follows this structure:

```
odoo-sync/
├── index.js              # Entry point + cron scheduler
├── config/
│   └── odoo.js           # OdooClient class (XML-RPC)
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

## OdooClient — XML-RPC Base Class

Always use this class as the foundation. Never call xmlrpc directly from business logic.

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
}

module.exports = OdooClient;
```

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

**Key fields:**
```javascript
['id', 'name', 'state', 'date', 'move_lines', 'origin',
 'partner_id', 'picking_type_id', 'scheduled_date']
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
['id', 'product_id', 'state', 'product_uom_qty',
 'quantity_done', 'name', 'picking_id']
```

**To get lines for a picking:**
```javascript
const moveLines = await odoo.searchRead(
  'stock.move',
  [['id', 'in', picking.move_lines]],
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
| `done` | Bloqueado |
| `cancel` | Cancelado |

**Key fields:**
```javascript
['id', 'name', 'state', 'date_order', 'partner_id',
 'amount_total', 'order_line', 'picking_ids']
```

---

### purchase.order — Pedidos de Compra

**States:** `draft`, `sent`, `to approve`, `purchase`, `done`, `cancel`

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

---

### res.partner — Clientes/Proveedores

```javascript
['id', 'name', 'email', 'phone', 'street', 'city',
 'country_id', 'customer_rank', 'supplier_rank']
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

    // 4. Fetch from Odoo
    const records = await this.odoo.searchRead(
      'stock.picking', domain,
      ['id', 'name', 'state', 'date', 'move_lines', 'origin'],
      { limit: 1000, order: 'date asc' }
    );

    // 5. Process and deduplicate
    const rows = [];
    for (const record of records) {
      const key = extractKey(record); // e.g. procurement group
      if (existingKeys.has(key)) continue;

      const detail = await this.odoo.read(
        'stock.picking', record.id,
        ['id', 'name', 'date', 'move_lines', 'origin']
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
# Odoo
ODOO_HOST=erp.yourcompany.com
ODOO_DATABASE=production_db
ODOO_LOGIN=sync_user@company.com
ODOO_PASSWORD=your_password
ODOO_PORT=443

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
    ['id', 'name', 'move_lines', 'origin']
  );
  console.log('Picking:', picking.name);

  const lines = await odoo.searchRead(
    'stock.move',
    [['id', 'in', picking.move_lines]],
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

The XML-RPC protocol is identical across all Odoo versions (v8–v17). Only a handful of field names changed:

| Field | v8–v12 | v13+ | Safe fallback |
|-------|--------|-------|---------------|
| Transfer lines | `move_lines` | `move_ids` | `picking.move_ids \|\| picking.move_lines \|\| []` |
| Scheduled date | `min_date` | `scheduled_date` | `picking.scheduled_date \|\| picking.min_date` |
| Detailed ops | _(not used)_ | `move_line_ids` (v14+) | check before using |
| Invoice model | `account.invoice` | `account.move` (v13+) | ask user which version |

Always use safe fallbacks when writing code that should work across versions:

```javascript
// v8–v17 safe patterns
const lines    = picking.move_ids      || picking.move_lines || [];
const date     = picking.scheduled_date || picking.min_date;
const invoice  = 'account.move'; // v13+, or 'account.invoice' for v8–v12
```

When the user mentions their Odoo version, use the exact field name. When unknown, use the safe fallback and add a comment.

### `date` field format
Odoo returns dates as `"2026-03-25 14:30:00"`. To get only the date part:
```javascript
const dateOnly = picking.date.split(' ')[0]; // "2026-03-25"
```

### searchRead limit
Default limit in OdooClient is 100. For production syncs always set `limit: 1000` (or use pagination with `offset`).

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

### Fields not returned unless explicitly requested
Odoo XML-RPC never returns all fields by default. Always specify fields:
```javascript
// Wrong — returns only id and name:
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
