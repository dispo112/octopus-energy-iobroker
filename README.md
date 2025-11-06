# 🐙 Octopus Energy × ioBroker – Smart-Meter via GraphQL + WhatsApp Alerts

Automatisches Auslesen der **Octopus Energy GraphQL API (DE)** in ioBroker – inklusive
Token‑Auto‑Refresh (KT‑CT‑1124/1143), 15‑Min/1h‑Werten, Tageskosten (00–05 Uhr Nachtpreis)
und **WhatsApp‑Benachrichtigungen** mit Emojis, Debounce, Cooldown & Test‑Ping.

> **Hinweis:** Keine geheimen Zugangsdaten im Repo – im Code sind **Platzhalter** gesetzt.
> Echte Werte bitte im ioBroker unter `0_userdata.0.Octopus.Config.*` pflegen.

---

## ✅ Features
- 🔐 **Auto-Login & Token-Refresh** (auch bei *expired*/KT‑CT‑1124/1143/401)
- 🧭 **Kein Redirect-Hopping** – fester GraphQL‑Endpoint (mit `/` am Ende)
- ⏱️ **RAW (15‑Min)** oder **HOUR (1h)** frei wählbar
- 💶 **Kostenberechnung** (z. B. 19 ct/kWh 00–05 Uhr, 31 ct/kWh tagsüber – anpassbar)
- 🗂️ Saubere **State‑Struktur**: `0_userdata.0.Octopus.*`
- 📲 **WhatsApp‑Alerts** bei Änderungen (gruppiert, mit Emojis) – inkl. Testmeldung beim Start
- 🧰 Robuste Logs & sanfte Selbstheilung (Retries, Backoff)

---

## 📦 Struktur

```
octopus-iobroker/
├── README.md
├── LICENSE
├── .gitignore
├── scripts/
│   ├── octopus_main.js          # Octopus GraphQL + Token + Kosten + States
│   └── whatsapp_notify.js       # WhatsApp-Änderungsmonitor
└── examples/
    └── (optional) screenshots / exports
```

---

## ⚙️ Installation (ioBroker)
1. ioBroker Admin → **Skripte** → **Neu** → **JavaScript**.
2. Inhalt von `scripts/octopus_main.js` einfügen, speichern, starten.
3. Inhalt von `scripts/whatsapp_notify.js` einfügen, speichern, starten.
4. **Zugangsdaten** eintragen (Platzhalter im Code oder über States):
   - `0_userdata.0.Octopus.Config.Email`
   - `0_userdata.0.Octopus.Config.Password`
   - `0_userdata.0.Octopus.Config.AccountNumber`
5. Für WhatsApp den Adapter `whatsapp-cmb` einrichten und die Instanz in `whatsapp_notify.js` prüfen.

---

## 🔧 Wichtige Parameter (Defaults im Code/State)
| State/Variable                               | Default        | Zweck |
|----------------------------------------------|----------------|------|
| `…Config.ReadingFrequency`                   | `RAW_INTERVAL` | 15‑Min‑Werte (`HOUR_INTERVAL` = 1h) |
| `…Config.NightStartHour` / `…NightEndHour`   | `0` / `5`      | Nachtfenster für Kosten |
| `…Config.NightPrice_ctkWh` / `…DayPrice_ctkWh`| `19.0` / `31.0`| ct/kWh (brutto) |
| `…Config.PollMinutes`                        | `15`           | Abfrageintervall |
| `…Auth.MaxAgeHours`                          | `20`           | Proaktischer Re‑Login |
| `Notify.Enabled` / `DebounceMs` / `CooldownMin` | `true` / `12000` / `10` | WhatsApp‑Monitor |

---

## 📁 Datenpunkte (Auszug)
```
0_userdata.0.Octopus.
├─ Info.LastRun / Status / RequestsTotal / RetriesTotal
├─ Config.(Email|Password|AccountNumber|ReadingFrequency|NightStartHour|…)
├─ Auth.Token / TokenIssuedAt / MaxAgeHours
├─ Meters.ActivePropertyId
├─ Raw.Today_JSON / Raw.Yesterday_JSON
├─ Totals.Today_kWh / Totals.Yesterday_kWh
└─ Cost.Today_EUR   / Cost.Yesterday_EUR
```

---

## 🧪 Troubleshooting
- **KT‑CT‑1124/1143, „expired“, 401** → Token wird automatisch erneuert. Notfalls `…Auth.Token` manuell leeren.
- **HTTP 301/302** → Wir nutzen bewusst einen **fixen Endpoint mit Slash**, daher keine Redirect‑Probleme.
- **Werte = 0** → Viele iMSys liefern Vortageswerte **gebündelt** → siehe `…Raw.Yesterday_JSON`.

---

## 📝 Lizenz
Dieses Projekt steht unter der **MIT‑Lizenz** (siehe `LICENSE`). Nutzung auf eigene Gefahr, ohne Gewähr.

---

## 💻 Skripte

### `scripts/octopus_main.js`
```javascript
// Octopus GraphQL (DE) – NO REDIRECT HOPS + Auto-Refresh
// Stand: 05.11.2025

const EMAIL_INIT          = 'deine@mail.de';
const PASSWORD_INIT       = 'deinPasswort';
const ACCOUNT_NUMBER_INIT = 'A-XXXXXXX'; // z.B. "A-12345678"

const ENDPOINT_FIXED_DEFAULT = 'https://api.oeg-kraken.energy/v1/graphql/';

const DEFAULT_READING_FREQUENCY = 'RAW_INTERVAL'; // oder 'HOUR_INTERVAL'
const DEFAULT_NIGHT_START = 0;
const DEFAULT_NIGHT_END   = 5;
const DEFAULT_PRICE_NIGHT = 19.0; // ct/kWh brutto
const DEFAULT_PRICE_DAY   = 31.0; // ct/kWh brutto
const DEFAULT_POLL_MIN    = 15;
const DEFAULT_TOKEN_MAX_H = 20;   // proaktiver Re-Login nach 20h

const HTTP_TIMEOUT_MS = 15000;
const MAX_RETRIES     = 3;

const https = require('https');
const BASE  = '0_userdata.0.Octopus';

async function ensureState(id, common) {
  if (!getObject(id)) {
    await createStateAsync(id, null, Object.assign({ read: true, write: true, role: 'state' }, common || {}));
  }
}
async function ensureString(id, name){ await ensureState(id, { type:'string', role:'text',  name:name||id }); }
async function ensureNumber(id, name, unit){ await ensureState(id, { type:'number', role:'value', unit, name:name||id }); }

function getStateSilent(id){
  if (!getObject(id)) return null;
  return getState(id);
}
async function w(id, val){
  if (!getObject(id)) {
    const isNum = typeof val === 'number';
    await ensureState(id, { type: isNum ? 'number' : 'string', role: isNum ? 'value' : 'text' });
  }
  await setStateAsync(id, val, true);
}
async function incr(id){
  const s = getStateSilent(id);
  const v = s && typeof s.val === 'number' ? s.val : 0;
  await w(id, v + 1);
}
function sleep(ms){ return new Promise(r=>setTimeout(r,ms)); }

function parseOrigin(u){
  const m = /^https?:\/\/[^/]+/i.exec(u);
  return m ? m[0] : '';
}
function buildHeaders(payload, token, url){
  const origin = parseOrigin(url);
  const headers = {
    'Content-Type': 'application/json',
    'Accept': 'application/json, text/plain, */*',
    'Content-Length': Buffer.byteLength(payload),
    'User-Agent': 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ioBroker-Octopus/1.5',
    'Origin': origin || undefined,
    'Referer': origin ? origin + '/' : undefined
  };
  if (token) headers['Authorization'] = token; // kein "Bearer", laut DE-Anleitung
  return headers;
}

function httpPostJson(url, payload, headers){
  return new Promise((resolve, reject) => {
    const req = https.request(url, { method:'POST', headers, timeout: HTTP_TIMEOUT_MS }, (res) => {
      let data = '';
      res.on('data', c => data += c);
      res.on('end', () => resolve({ status: res.statusCode, body: data, headers: res.headers || {} }));
    });
    req.on('timeout', () => req.destroy(new Error('HTTP timeout')));
    req.on('error', reject);
    req.write(payload);
    req.end();
  });
}
function readEndpoint(){
  const o = getStateSilent(`${BASE}.Config.Endpoint`);
  const v = o && typeof o.val === 'string' ? o.val.trim() : '';
  let url = (v || ENDPOINT_FIXED_DEFAULT).replace(/\/+$/,'') + '/';
  return url;
}
async function gqlRequest(bodyObj, token, purpose){
  const url = readEndpoint();
  const payload = JSON.stringify(bodyObj);
  const headers = buildHeaders(payload, token, url);

  const res = await httpPostJson(url, payload, headers);
  if (res.status < 200 || res.status >= 300) {
    throw new Error(`HTTP ${res.status}: ${String(res.body).slice(0,200)}…`);
  }
  const json = JSON.parse(res.body || '{}');
  if (json.errors) throw new Error('GraphQL errors: ' + JSON.stringify(json.errors));
  return json.data;
}
async function gqlWithRetry(body, tokenIn, purpose){
  let attempt = 0;
  let token = tokenIn;
  let reauthedOnce = false;

  while (true){
    try{
      attempt++;
      await incr(`${BASE}.Info.RequestsTotal`);
      return await gqlRequest(body, token, purpose);

    }catch(e){
      const msg = String(e && e.message || e || '');
      log(`[Octopus] ${purpose} fehlgeschlagen (Versuch ${attempt}/${MAX_RETRIES}): ${msg}`, 'warn');

      if (!reauthedOnce && /(KT-CT-1143|KT-CT-1124|AUTHORIZATION|Authorization header|expired|token.*expir|401|unauthor)/i.test(msg)){
        try{
          await w(`${BASE}.Auth.Token`, '');
          const cfg = readCfg();
          token = await obtainToken(cfg.email, cfg.password);
          reauthedOnce = true;
          continue;
        }catch(re){
          log(`[Octopus] Re-Auth fehlgeschlagen: ${String(re.message || re)}`, 'error');
        }
      }

      if (attempt >= MAX_RETRIES) throw e;
      await incr(`${BASE}.Info.RetriesTotal`);
      await sleep(500 * Math.pow(2, attempt - 1));
    }
  }
}

const MUTATION_TOKEN = `
mutation krakenTokenAuthentication($email: String!, $password: String!) {
  obtainKrakenToken(input: {email: $email, password: $password}) {
    token
  }
}
`;
const QUERY_PROPERTIES = `
query getPropertyIds($accountNumber: String!) {
  account(accountNumber: $accountNumber) {
    properties {
      id
      occupancyPeriods { effectiveFrom effectiveTo }
    }
  }
}
`;
function buildMeasurementsQuery(freq, first){
  return `
query getSmartMeterUsage($accountNumber: String!, $propertyId: ID!, $date: Date!) {
  account(accountNumber: $accountNumber) {
    property(id: $propertyId) {
      measurements(
        utilityFilters: {electricityFilters: {readingFrequencyType: ${freq}, readingQuality: COMBINED}}
        startOn: $date
        first: ${first}
      ) {
        edges { node { ... on IntervalMeasurementType { startAt endAt unit value } } }
      }
    }
  }
}
`;}
function isoLocalDate(d){ return `${d.getFullYear()}-${String(d.getMonth()+1).padStart(2,'0')}-${String(d.getDate()).padStart(2,'0')}`; }
function toLocal(s){ return new Date(s); }
function roundR(n,d=3){ const p=10**d; return Math.round(n*p)/p; }
function priceCentsFor(dt, ns, ne, nct, dct){ const h=dt.getHours(); return (h>=ns && h<ne) ? nct : dct; }
function summarizeEdges(edges, cfg){
  let kWh=0, eur=0; const out=[];
  for (const e of edges||[]){
    const n=e?.node; if(!n) continue;
    const v=Number(n.value)||0; kWh+=v;
    const ref=toLocal(n.startAt||n.endAt);
    eur += v * (priceCentsFor(ref, cfg.nightStart, cfg.nightEnd, cfg.priceNight, cfg.priceDay)/100);
    out.push({ startAt:n.startAt, endAt:n.endAt, unit:n.unit, value:v });
  }
  return { kWh: roundR(kWh), eur: roundR(eur,3), list: out };
}
function pickPropertyIdForDate(props, dateISO){
  if(!Array.isArray(props)) return null;
  const ts=new Date(dateISO+'T12:00:00').getTime();
  for (const p of props){
    for (const per of (p?.occupancyPeriods||[])){
      const from=per?.effectiveFrom?new Date(per.effectiveFrom).getTime():-Infinity;
      const to  =per?.effectiveTo  ?new Date(per.effectiveTo).getTime(): Infinity;
      if (ts>=from && ts<=to) return p.id;
    }
  }
  return props[0]?.id||null;
}

async function initStates(){
  await ensureString(`${BASE}.Info.LastRun`, 'Letzter Erfolg');
  await ensureString(`${BASE}.Info.Status`, 'Status');
  await ensureNumber(`${BASE}.Info.RequestsTotal`, 'API Requests total');
  await ensureNumber(`${BASE}.Info.RetriesTotal`, 'API Retries total');

  await ensureString(`${BASE}.Config.Email`, 'E-Mail');
  await ensureString(`${BASE}.Config.Password`, 'Passwort');
  await ensureString(`${BASE}.Config.AccountNumber`, 'AccountNumber');
  await ensureString(`${BASE}.Config.ReadingFrequency`, 'RAW_INTERVAL/HOUR_INTERVAL');
  await ensureNumber(`${BASE}.Config.NightStartHour`, 'Nacht Start (h)');
  await ensureNumber(`${BASE}.Config.NightEndHour`, 'Nacht Ende (h)');
  await ensureNumber(`${BASE}.Config.NightPrice_ctkWh`, 'Nacht ct/kWh','ct/kWh');
  await ensureNumber(`${BASE}.Config.DayPrice_ctkWh`, 'Tag ct/kWh','ct/kWh');
  await ensureNumber(`${BASE}.Config.PollMinutes`, 'Poll-Minuten','min');
  await ensureString(`${BASE}.Config.Endpoint`, 'Endpoint-Override (optional, mit Slash)');

  await ensureString(`${BASE}.Auth.Token`, 'Token');
  await ensureNumber(`${BASE}.Auth.TokenIssuedAt`, 'Token ausgegeben (ms)','ms');
  await ensureNumber(`${BASE}.Auth.MaxAgeHours`, 'Max Token-Alter (h)','h');

  await ensureString(`${BASE}.Meters.ActivePropertyId`, 'Aktive PropertyId');

  await ensureString(`${BASE}.Raw.Today_JSON`, 'Heute raw JSON');
  await ensureString(`${BASE}.Raw.Yesterday_JSON`, 'Gestern raw JSON');

  await ensureNumber(`${BASE}.Totals.Today_kWh`, 'Heute kWh','kWh');
  await ensureNumber(`${BASE}.Totals.Yesterday_kWh`, 'Gestern kWh','kWh');
  await ensureNumber(`${BASE}.Cost.Today_EUR`, 'Heute €','€');
  await ensureNumber(`${BASE}.Cost.Yesterday_EUR`, 'Gestern €','€');

  if ((getStateSilent(`${BASE}.Config.Email`)?.val ?? '') === '')         await w(`${BASE}.Config.Email`, EMAIL_INIT);
  if ((getStateSilent(`${BASE}.Config.Password`)?.val ?? '') === '')      await w(`${BASE}.Config.Password`, PASSWORD_INIT);
  if ((getStateSilent(`${BASE}.Config.AccountNumber`)?.val ?? '') === '') await w(`${BASE}.Config.AccountNumber`, ACCOUNT_NUMBER_INIT);
  if ((getStateSilent(`${BASE}.Config.ReadingFrequency`)?.val ?? '') === '') await w(`${BASE}.Config.ReadingFrequency`, DEFAULT_READING_FREQUENCY);
  if ((getStateSilent(`${BASE}.Config.NightStartHour`)?.val ?? '') === '') await w(`${BASE}.Config.NightStartHour`, DEFAULT_NIGHT_START);
  if ((getStateSilent(`${BASE}.Config.NightEndHour`)?.val ?? '') === '')   await w(`${BASE}.Config.NightEndHour`, DEFAULT_NIGHT_END);
  if ((getStateSilent(`${BASE}.Config.NightPrice_ctkWh`)?.val ?? '') === '') await w(`${BASE}.Config.NightPrice_ctkWh`, DEFAULT_PRICE_NIGHT);
  if ((getStateSilent(`${BASE}.Config.DayPrice_ctkWh`)?.val ?? '') === '')   await w(`${BASE}.Config.DayPrice_ctkWh`, DEFAULT_PRICE_DAY);
  if ((getStateSilent(`${BASE}.Config.PollMinutes`)?.val ?? '') === '')      await w(`${BASE}.Config.PollMinutes`, DEFAULT_POLL_MIN);

  if ((getStateSilent(`${BASE}.Auth.MaxAgeHours`)?.val ?? '') === '')       await w(`${BASE}.Auth.MaxAgeHours`, DEFAULT_TOKEN_MAX_H);

  const ep = String(getStateSilent(`${BASE}.Config.Endpoint`)?.val || '');
  if (!ep) await w(`${BASE}.Config.Endpoint`, ENDPOINT_FIXED_DEFAULT);
}

async function obtainToken(email, password){
  const data = await gqlWithRetry(
    { query: MUTATION_TOKEN, variables: { email, password } },
    null, 'Login'
  );
  const token = data?.obtainKrakenToken?.token;
  if (!token) throw new Error('Kein Token erhalten');
  await w(`${BASE}.Auth.Token`, token);
  await w(`${BASE}.Auth.TokenIssuedAt`, Date.now());
  return token;
}
function readCfg(){
  return {
    email:       String(getStateSilent(`${BASE}.Config.Email`)?.val || EMAIL_INIT),
    password:    String(getStateSilent(`${BASE}.Config.Password`)?.val || PASSWORD_INIT),
    account:     String(getStateSilent(`${BASE}.Config.AccountNumber`)?.val || ACCOUNT_NUMBER_INIT),
    freq:        String(getStateSilent(`${BASE}.Config.ReadingFrequency`)?.val || DEFAULT_READING_FREQUENCY),
    nightStart:  Number(getStateSilent(`${BASE}.Config.NightStartHour`)?.val ?? DEFAULT_NIGHT_START),
    nightEnd:    Number(getStateSilent(`${BASE}.Config.NightEndHour`)?.val ?? DEFAULT_NIGHT_END),
    priceNight:  Number(getStateSilent(`${BASE}.Config.NightPrice_ctkWh`)?.val ?? DEFAULT_PRICE_NIGHT),
    priceDay:    Number(getStateSilent(`${BASE}.Config.DayPrice_ctkWh`)?.val ?? DEFAULT_PRICE_DAY),
    pollMinutes: Number(getStateSilent(`${BASE}.Config.PollMinutes`)?.val ?? DEFAULT_POLL_MIN)
  };
}

function buildMeasurementsQuery(freq, first){
  return `
query getSmartMeterUsage($accountNumber: String!, $propertyId: ID!, $date: Date!) {
  account(accountNumber: $accountNumber) {
    property(id: $propertyId) {
      measurements(
        utilityFilters: {electricityFilters: {readingFrequencyType: ${freq}, readingQuality: COMBINED}}
        startOn: $date
        first: ${first}
      ) {
        edges { node { ... on IntervalMeasurementType { startAt endAt unit value } } }
      }
    }
  }
}
`; }
function buildMeasBody(freq, first, account, propertyId, dateISO){
  return { query: buildMeasurementsQuery(freq, first), variables: { accountNumber: account, propertyId, date: dateISO } };
}
async function fetchDay(dateISO, cfg, token){
  const propsRes = await gqlWithRetry({ query: QUERY_PROPERTIES, variables: { accountNumber: cfg.account } }, token, 'Properties');
  const props = propsRes?.account?.properties || [];
  if (!props.length) throw new Error('Keine Properties gefunden');

  const propertyId = pickPropertyIdForDate(props, dateISO) || props[0].id;
  await w(`${BASE}.Meters.ActivePropertyId`, propertyId);

  const firstCount = cfg.freq === 'RAW_INTERVAL' ? 96 : 24;
  const measRes = await gqlWithRetry(buildMeasBody(cfg.freq, firstCount, cfg.account, propertyId, dateISO), token, `Measurements ${dateISO}`);
  const edges = measRes?.account?.property?.measurements?.edges || [];
  return { edges, propertyId };
}

async function pollOnce(){
  try{
    await w(`${BASE}.Info.Status`, 'running…');
    const cfg = readCfg();

    const maxH   = Number(getStateSilent(`${BASE}.Auth.MaxAgeHours`)?.val ?? DEFAULT_TOKEN_MAX_H);
    const issued = Number(getStateSilent(`${BASE}.Auth.TokenIssuedAt`)?.val || 0);
    if (issued && (Date.now() - issued) > maxH*3600*1000) {
      await w(`${BASE}.Auth.Token`, '');
    }

    let token = String(getStateSilent(`${BASE}.Auth.Token`)?.val || '');
    if (!token) token = await obtainToken(cfg.email, cfg.password);

    const now = new Date();
    const todayISO = isoLocalDate(now);
    const yestISO  = isoLocalDate(new Date(now.getTime() - 24*3600*1000));

    const t = await fetchDay(todayISO, cfg, token);
    const sumT = summarizeEdges(t.edges, cfg);
    await w(`${BASE}.Raw.Today_JSON`, JSON.stringify(sumT.list));
    await w(`${BASE}.Totals.Today_kWh`, sumT.kWh);
    await w(`${BASE}.Cost.Today_EUR`, sumT.eur);

    const y = await fetchDay(yestISO, cfg, token);
    const sumY = summarizeEdges(y.edges, cfg);
    await w(`${BASE}.Raw.Yesterday_JSON`, JSON.stringify(sumY.list));
    await w(`${BASE}.Totals.Yesterday_kWh`, sumY.kWh);
    await w(`${BASE}.Cost.Yesterday_EUR`, sumY.eur);

    await w(`${BASE}.Info.LastRun`, new Date().toISOString());
    await w(`${BASE}.Info.Status`, 'ok');
    log('[Octopus] Update ok', 'info');

  }catch(err){
    const msg = `[Octopus] Fehler: ${err.message}`;
    log(msg, 'error');
    await w(`${BASE}.Info.Status`, msg);

    if (/(401|unauthor|KT-CT-1143|KT-CT-1124|AUTHORIZATION|Authorization header|expired|token.*expir)/i.test(String(err))) {
      await w(`${BASE}.Auth.Token`, '');
    }
  }
}

(async () => {
  await initStates();
  log('[Octopus] Initialisiert (no-redirect). Starte erste Abfrage…', 'info');
  await pollOnce();

  const cfgMin = Number(getStateSilent(`${BASE}.Config.PollMinutes`)?.val ?? DEFAULT_POLL_MIN);
  const pollMin = Math.max(1, Math.floor(cfgMin));
  schedule(`*/${pollMin} * * * *`, () => pollOnce());
})();

```

### `scripts/whatsapp_notify.js`
```javascript
// Octopus → WhatsApp Monitor (pretty msgs, grouped, emojis)
// Stand: 05.11.2025

const WHATSAPP_INSTANCE = 'whatsapp-cmb.0';
const WHATSAPP_TITLE_DEFAULT = '🐙⚡ Octopus Monitor';

const OBASE = '0_userdata.0.Octopus';

const WATCH = [
  { id: `${OBASE}.Totals.Today_kWh`,       label: 'Heute',    fmt: vKWh,      emoji: '📊', group: 'Verbrauch' },
  { id: `${OBASE}.Totals.Yesterday_kWh`,   label: 'Gestern',  fmt: vKWh,      emoji: '📊', group: 'Verbrauch', priority: true },
  { id: `${OBASE}.Cost.Today_EUR`,         label: 'Heute',    fmt: vMoney,    emoji: '💶', group: 'Kosten'    },
  { id: `${OBASE}.Cost.Yesterday_EUR`,     label: 'Gestern',  fmt: vMoney,    emoji: '💶', group: 'Kosten'    },
  { id: `${OBASE}.Raw.Today_JSON`,         label: 'Heute',    fmt: vIntervals,emoji: '⏱️', group: 'Intervalle' },
  { id: `${OBASE}.Raw.Yesterday_JSON`,     label: 'Gestern',  fmt: vIntervals,emoji: '⏱️', group: 'Intervalle', priority: true },
];

const NBASE = `${OBASE}.Notify`;
const DEFAULTS = {
  enabled:     true,
  debounceMs:  12000,
  cooldownMin: 10,
  title:       WHATSAPP_TITLE_DEFAULT,
};

let debounceTimer = null;
let pendingMsgs   = [];
let lastSendTs    = 0;

function vKWh(v){ const n = toNumber(v); return n == null ? '–' : `${round(n,3)} kWh`; }
function vMoney(v){ const n = toNumber(v); return n == null ? '–' : `${round(n,2)} €`; }
function vText(v){ if (v == null) return '–'; return String(v).slice(0,200); }
function vIntervals(v){
  try{
    if (!v) return '0 Intervalle';
    const arr = typeof v === 'string' ? JSON.parse(v) : v;
    const len = Array.isArray(arr) ? arr.length : 0;
    return `${len} Intervalle`;
  }catch(e){ return '0 Intervalle'; }
}
function toNumber(v){
  if (v === null || v === undefined || v === '' || v === 'unavailable' || v === 'unknown') return null;
  if (typeof v === 'number' && isFinite(v)) return v;
  if (typeof v === 'string'){ const n = parseFloat(v.replace(',', '.')); return isFinite(n) ? n : null; }
  return null;
}
function round(n,d=3){ const p=10**d; return Math.round(n*p)/p; }
function nowMs(){ return Date.now(); }
function tsHuman(){ try { return new Date().toLocaleString('de-DE', { hour12:false }); } catch { return new Date().toISOString(); } }

async function ensureState(id, common) {
  if (!getObject(id)) {
    await createStateAsync(id, null, Object.assign({ read: true, write: true, role: 'state' }, common || {}));
  }
}
async function ensureString(id, name){ await ensureState(id, { type:'string', role:'text',  name:name||id }); }
async function ensureNumber(id, name, unit){ await ensureState(id, { type:'number', role:'value', unit, name:name||id }); }

function cfgEnabled(){ return !!getState(`${NBASE}.Enabled`)?.val; }
function cfgDebounceMs(){ return Number(getState(`${NBASE}.DebounceMs`)?.val ?? DEFAULTS.debounceMs); }
function cfgCooldown(){ return Number(getState(`${NBASE}.CooldownMin`)?.val ?? DEFAULTS.cooldownMin); }
function cfgTitle(){ return String(getState(`${NBASE}.Title`)?.val ?? DEFAULTS.title); }

async function ensureNotifyStates(){
  if (!getObject(`${NBASE}.Enabled`))     await createStateAsync(`${NBASE}.Enabled`, DEFAULTS.enabled,     { type:'boolean', name:'Monitor aktiv' });
  if (!getObject(`${NBASE}.DebounceMs`))  await createStateAsync(`${NBASE}.DebounceMs`, DEFAULTS.debounceMs,{ type:'number',  name:'Debounce (ms)' });
  if (!getObject(`${NBASE}.CooldownMin`)) await createStateAsync(`${NBASE}.CooldownMin`, DEFAULTS.cooldownMin,{ type:'number', name:'Cooldown (Minuten)' });
  if (!getObject(`${NBASE}.Title`))       await createStateAsync(`${NBASE}.Title`, DEFAULTS.title,         { type:'string',  name:'WhatsApp Titel' });
  if (!getObject(`${NBASE}.LastSendTs`))  await createStateAsync(`${NBASE}.LastSendTs`, 0,                 { type:'number',  name:'Letzter Versand (ms)' });
  if (!getObject(`${NBASE}.LastMessage`)) await createStateAsync(`${NBASE}.LastMessage`, '',               { type:'string',  name:'Letzte Meldung' });
}

function sendToWhatsApp(msg, title){
  try {
    sendTo(WHATSAPP_INSTANCE, { text: msg, title: title || cfgTitle() });
    return true;
  } catch (e) {
    log(`[Octopus Monitor] sendTo(${WHATSAPP_INSTANCE}) fehlgeschlagen: ${e.message}`, 'error');
    return false;
  }
}

function sectionIcon(name){
  switch ((name||'').toLowerCase()){
    case 'verbrauch':  return '📊';
    case 'kosten':     return '💶';
    case 'intervalle': return '⏱️';
    case 'system':     return '🛠️';
    default:           return '📬';
  }
}
function groupBy(arr, keyFn){
  return arr.reduce((acc, item)=>{
    const k = keyFn(item);
    (acc[k] = acc[k] || []).push(item);
    return acc;
  }, {});
}

function sendWhatsappPretty(changes, { priority=false } = {}){
  if (!cfgEnabled()) return;

  const now  = nowMs();
  const cdMs = cfgCooldown() * 60 * 1000;
  if (!priority && lastSendTs && (now - lastSendTs) < cdMs){
    log(`[Octopus Monitor] Cooldown ${cfgCooldown()} min aktiv – Nachricht verworfen.`, 'info');
    return;
  }

  const title  = cfgTitle();
  const header = `${title}\n${'━'.repeat(18)}\n🕒 ${tsHuman()}`;
  const sections = groupBy(changes, c => c.group || 'Änderungen');

  const order = ['Verbrauch','Kosten','Intervalle','System','Änderungen'];
  const sortedKeys = Object.keys(sections).sort((a,b)=> order.indexOf(a) - order.indexOf(b));

  const lines = [header];
  for (const key of sortedKeys){
    lines.push(`\n${sectionIcon(key)}  *${key}*`);
    for (const c of sections[key]){
      const delta = (c.oldFmt && c.oldFmt !== c.newFmt) ? ` (alt ${c.oldFmt})` : '';
      lines.push(`${c.emoji || '•'} ${c.label}: ${c.newFmt}${delta}`);
    }
  }
  const msg = lines.join('\n');

  if (sendToWhatsApp(msg, title)) {
    lastSendTs = now;
    setState(`${NBASE}.LastSendTs`, lastSendTs, true);
    setState(`${NBASE}.LastMessage`, msg, true);
    log(`[Octopus Monitor] WhatsApp gesendet (${changes.length} Änderungen)`, 'info');
  }
}

function vKWhFmt(v){ const n = toNumber(v); return n == null ? '–' : `${round(n,3)} kWh`; }

function handleChange(obj, cfgEntry){
  try{
    const id     = obj.id;
    const oldVal = obj.oldState ? obj.oldState.val : undefined;
    const newVal = obj.state ? obj.state.val : undefined;

    const label  = cfgEntry.label || id.split('.').slice(-1)[0];
    const fmt    = cfgEntry.fmt || ((v)=>String(v));
    const newFmt = fmt(newVal);
    const oldFmt = (oldVal === undefined) ? null : fmt(oldVal);

    pendingMsgs.push({
      id, label,
      newFmt, oldFmt,
      emoji: cfgEntry.emoji, group: cfgEntry.group,
      priority: !!cfgEntry.priority
    });

    if (debounceTimer) clearTimeout(debounceTimer);
    debounceTimer = setTimeout(flushBundle, cfgDebounceMs());
  }catch(e){
    log(`[Octopus Monitor] handleChange-Fehler: ${e.message}`, 'warn');
  }
}

function flushBundle(){
  debounceTimer = null;
  if (!pendingMsgs.length) return;

  const hadPriority = pendingMsgs.some(m => m.priority);
  const bundle = pendingMsgs.slice();
  pendingMsgs = [];

  sendWhatsappPretty(bundle, { priority: hadPriority });
}

function registerWatchers(){
  WATCH.forEach(cfgEntry => {
    if (!cfgEntry.id) return;
    on({ id: cfgEntry.id, change: 'ne' }, obj => handleChange(obj, cfgEntry));
  });
}

(async () => {
  await ensureNotifyStates();
  registerWatchers();

  // hübsche Test-Nachricht beim Start
  sendWhatsappPretty([
    { label:'Monitor',   newFmt:'aktiv',                oldFmt:null, emoji:'✅', group:'System' },
    { label:'Debounce',  newFmt:`${cfgDebounceMs()} ms`,oldFmt:null, emoji:'⏱️', group:'System' },
    { label:'Cooldown',  newFmt:`${cfgCooldown()} min`, oldFmt:null, emoji:'🧊', group:'System' },
  ], { priority:true });

  log(`[Octopus Monitor] läuft. Beobachtet ${WATCH.length} Datenpunkte.`, 'info');
})();

```

---

Viel Freude beim Automatisieren! ⚡🐙
Feedback/PRs willkommen.
