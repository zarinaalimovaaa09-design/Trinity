/**
 * Trinity — Economic Case Championship
 * Express backend
 *
 * Routes:
 *   GET  /api/stats           Public event stats (registration count, event date)
 *   POST /api/register        Create a new delegate registration
 *   POST /api/admin/login     Verify an admin key
 *   GET  /api/admin/users     List all registrations (requires x-admin-key header)
 *
 * Storage: a flat JSON file on disk (data/registrations.json). This is intentionally
 * simple so the project needs no external database to deploy. NOTE: Render's free-tier
 * filesystem is ephemeral — data will reset on redeploy or when the instance restarts/spins
 * down. Swap `readData`/`writeData` for a real database (Postgres, Mongo, etc.) before
 * relying on this for a production event with data you can't afford to lose.
 */

const express = require('express');
const bodyParser = require('body-parser');
const cors = require('cors');
const path = require('path');
const fs = require('fs/promises');
const crypto = require('crypto');

const app = express();
const PORT = process.env.PORT || 3000;
const ADMIN_KEY = process.env.ADMIN_KEY || 'trinity-admin-2026';
const EVENT_DATE_ISO = '2026-09-27T10:00:00+05:00'; // Fergana time, GMT+5

const DATA_DIR = path.join(__dirname, 'data');
const DATA_FILE = path.join(DATA_DIR, 'registrations.json');

if (process.env.ADMIN_KEY === undefined) {
  console.warn(
    '[trinity] ADMIN_KEY is not set — falling back to a default admin key. ' +
    'Set the ADMIN_KEY environment variable in Render before going live.'
  );
}

// ---------- storage helpers ----------

async function ensureDataFile() {
  await fs.mkdir(DATA_DIR, { recursive: true });
  try {
    await fs.access(DATA_FILE);
  } catch {
    await fs.writeFile(DATA_FILE, '[]', 'utf8');
  }
}

async function readData() {
  await ensureDataFile();
  const raw = await fs.readFile(DATA_FILE, 'utf8');
  try {
    return JSON.parse(raw);
  } catch {
    return [];
  }
}

async function writeData(records) {
  await ensureDataFile();
  await fs.writeFile(DATA_FILE, JSON.stringify(records, null, 2), 'utf8');
}

// ---------- middleware ----------

app.use(cors());
app.use(bodyParser.json());
app.use(express.static(path.join(__dirname, 'public')));

function requireAdmin(req, res, next) {
  const key = req.headers['x-admin-key'];
  if (!key || key !== ADMIN_KEY) {
    return res.status(401).json({ ok: false, error: 'Invalid or missing admin key.' });
  }
  next();
}

// ---------- validation ----------

const VALID_TRACKS = ['Team Leader', 'Delegate', 'Individual Analyst'];

function validateRegistration(body) {
  const errors = [];
  const fullName = (body.fullName || '').toString().trim();
  const telegram = (body.telegram || '').toString().trim();
  const track = (body.track || '').toString().trim();
  const organization = (body.organization || '').toString().trim();

  if (fullName.length < 2 || fullName.length > 100) {
    errors.push('Full name must be between 2 and 100 characters.');
  }
  if (!/^@?[a-zA-Z0-9_]{5,32}$/.test(telegram)) {
    errors.push('Enter a valid Telegram username (5–32 characters, letters/numbers/underscore).');
  }
  if (!VALID_TRACKS.includes(track)) {
    errors.push('Select a valid track.');
  }
  if (organization.length < 2 || organization.length > 150) {
    errors.push('Organization must be between 2 and 150 characters.');
  }

  return {
    errors,
    clean: {
      fullName,
      telegram: telegram.startsWith('@') ? telegram.slice(1) : telegram,
      track,
      organization,
    },
  };
}

// ---------- routes ----------

app.get('/api/stats', async (req, res) => {
  const records = await readData();
  res.json({
    ok: true,
    eventDate: EVENT_DATE_ISO,
    totalRegistrations: records.length,
  });
});

app.post('/api/register', async (req, res) => {
  const { errors, clean } = validateRegistration(req.body || {});
  if (errors.length) {
    return res.status(400).json({ ok: false, error: errors.join(' ') });
  }

  const records = await readData();

  const duplicate = records.find(
    (r) => r.telegram.toLowerCase() === clean.telegram.toLowerCase()
  );
  if (duplicate) {
    return res.status(409).json({
      ok: false,
      error: 'This Telegram username is already registered.',
    });
  }

  const record = {
    id: crypto.randomUUID(),
    ...clean,
    registeredAt: new Date().toISOString(),
  };

  records.push(record);
  await writeData(records);

  res.status(201).json({ ok: true, registration: record, total: records.length });
});

app.post('/api/admin/login', (req, res) => {
  const key = (req.body && req.body.key) || '';
  if (key !== ADMIN_KEY) {
    return res.status(401).json({ ok: false, error: 'Incorrect admin key.' });
  }
  res.json({ ok: true });
});

app.get('/api/admin/users', requireAdmin, async (req, res) => {
  const records = await readData();
  const sorted = [...records].sort(
    (a, b) => new Date(b.registeredAt) - new Date(a.registeredAt)
  );
  res.json({ ok: true, total: sorted.length, users: sorted });
});

app.get('/healthz', (req, res) => res.json({ ok: true }));

app.use((req, res) => {
  // /admin, /portal, etc. aren't real server routes — they're sections the
  // front-end JS shows/hides after index.html loads. Serve the page for any
  // other GET request so direct links and refreshes don't hit a bare JSON 404.
  if (req.method === 'GET' && !req.path.startsWith('/api/')) {
    return res.sendFile(path.join(__dirname, 'public', 'index.html'), (err) => {
      if (err) res.status(404).json({ ok: false, error: 'Not found.' });
    });
  }
  res.status(404).json({ ok: false, error: 'Not found.' });
});

app.listen(PORT, () => {
  console.log(`[trinity] server listening on port ${PORT}`);
});
