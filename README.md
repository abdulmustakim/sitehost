# FreeHost — Full Stack Static Site Hosting Platform

A complete browser-based static site hosting platform with integrated code editor, live preview, deployment pipeline, and admin dashboard.

---

## 🚀 Quick Start (Demo)

Open `index.html` directly in any browser — no build step.

**Demo Accounts:**
| Email | Password | Role |
|-------|----------|------|
| user@demo.com | demo1234 | User |
| Click "Sign in as Admin" | — | Admin |

---

## 📁 Project Structure (Full Stack)

```
freehost/
├── frontend/
│   ├── index.html         # Landing page
│   ├── src/
│   │   ├── main.jsx       # React entry
│   │   ├── App.jsx        # Router
│   │   ├── pages/
│   │   │   ├── Landing.jsx
│   │   │   ├── Auth.jsx
│   │   │   ├── Dashboard.jsx
│   │   │   ├── Editor.jsx
│   │   │   └── Admin.jsx
│   │   ├── components/
│   │   │   ├── Editor/
│   │   │   │   ├── CodeEditor.jsx   # Monaco wrapper
│   │   │   │   ├── FileTree.jsx
│   │   │   │   └── Preview.jsx
│   │   │   ├── Dashboard/
│   │   │   │   ├── SiteCard.jsx
│   │   │   │   └── DeployLog.jsx
│   │   │   └── Admin/
│   │   │       ├── UserTable.jsx
│   │   │       ├── SiteTable.jsx
│   │   │       └── Charts.jsx
│   │   └── lib/
│   │       ├── supabase.js   # Supabase client
│   │       ├── api.js        # API calls
│   │       └── storage.js    # File operations
│   ├── package.json
│   └── vite.config.js
│
├── backend/
│   ├── server.js          # Express entry
│   ├── routes/
│   │   ├── auth.js        # /api/auth/*
│   │   ├── sites.js       # /api/sites/*
│   │   ├── files.js       # /api/files/*
│   │   └── admin.js       # /api/admin/*
│   ├── middleware/
│   │   ├── auth.js        # JWT verification
│   │   └── admin.js       # Admin role check
│   ├── lib/
│   │   ├── supabase.js    # Supabase admin client
│   │   └── storage.js     # File system utils
│   └── package.json
│
├── supabase/
│   ├── migrations/
│   │   └── 001_init.sql   # Database schema
│   └── seed.js            # Admin seed script
│
├── nginx/
│   └── freehost.conf      # Wildcard subdomain routing
│
├── .env.example
└── README.md
```

---

## 🗄️ Database Schema (Supabase / PostgreSQL)

```sql
-- supabase/migrations/001_init.sql

-- Extend the auth.users table with a profile
CREATE TABLE public.profiles (
  id          UUID REFERENCES auth.users(id) ON DELETE CASCADE PRIMARY KEY,
  username    TEXT UNIQUE NOT NULL,
  role        TEXT NOT NULL DEFAULT 'user' CHECK (role IN ('user','admin')),
  status      TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active','suspended')),
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

-- Row Level Security
ALTER TABLE public.profiles ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users can read own profile" ON public.profiles FOR SELECT USING (auth.uid() = id);
CREATE POLICY "Admins can read all profiles" ON public.profiles FOR SELECT USING (
  (SELECT role FROM public.profiles WHERE id = auth.uid()) = 'admin'
);

-- Sites table
CREATE TABLE public.sites (
  id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  owner_id    UUID REFERENCES public.profiles(id) ON DELETE CASCADE NOT NULL,
  subdomain   TEXT UNIQUE NOT NULL CHECK (subdomain ~ '^[a-z0-9-]{3,30}$'),
  custom_domain TEXT,
  status      TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft','live','suspended')),
  storage_bytes BIGINT DEFAULT 0,
  last_deploy TIMESTAMPTZ,
  created_at  TIMESTAMPTZ DEFAULT NOW()
);

ALTER TABLE public.sites ENABLE ROW LEVEL SECURITY;
CREATE POLICY "Users see own sites" ON public.sites FOR ALL USING (owner_id = auth.uid());
CREATE POLICY "Admins see all sites" ON public.sites FOR ALL USING (
  (SELECT role FROM public.profiles WHERE id = auth.uid()) = 'admin'
);

-- Site files (metadata; actual content in Supabase Storage)
CREATE TABLE public.site_files (
  id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  site_id     UUID REFERENCES public.sites(id) ON DELETE CASCADE NOT NULL,
  path        TEXT NOT NULL,  -- e.g. "index.html", "assets/logo.png"
  size_bytes  INT DEFAULT 0,
  mime_type   TEXT,
  updated_at  TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(site_id, path)
);

-- Deployments / logs
CREATE TABLE public.deployments (
  id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  site_id     UUID REFERENCES public.sites(id) ON DELETE CASCADE NOT NULL,
  status      TEXT NOT NULL CHECK (status IN ('pending','running','success','failed')),
  log         TEXT,
  deployed_at TIMESTAMPTZ DEFAULT NOW()
);

-- Analytics (aggregated daily)
CREATE TABLE public.analytics (
  id          UUID DEFAULT gen_random_uuid() PRIMARY KEY,
  site_id     UUID REFERENCES public.sites(id) ON DELETE CASCADE NOT NULL,
  date        DATE NOT NULL,
  visits      INT DEFAULT 0,
  bandwidth   BIGINT DEFAULT 0,
  UNIQUE(site_id, date)
);

-- System settings (admin-only)
CREATE TABLE public.system_settings (
  key   TEXT PRIMARY KEY,
  value JSONB
);
INSERT INTO public.system_settings VALUES
  ('max_storage_mb', '100'),
  ('max_bandwidth_gb', '10'),
  ('max_sites_per_user', '5'),
  ('public_registration', 'true'),
  ('maintenance_mode', 'false');
```

---

## 🌱 Seed Script (First Admin)

```javascript
// supabase/seed.js
import { createClient } from '@supabase/supabase-js';
import dotenv from 'dotenv';
dotenv.config();

const supabase = createClient(
  process.env.SUPABASE_URL,
  process.env.SUPABASE_SERVICE_ROLE_KEY  // service role bypasses RLS
);

async function seedAdmin() {
  const { data: user, error } = await supabase.auth.admin.createUser({
    email: 'admin@freehost.com',
    password: 'Admin$ecure123!',
    email_confirm: true,
  });
  if (error) { console.error('Error creating admin user:', error); return; }

  const { error: profileError } = await supabase.from('profiles').insert({
    id: user.user.id,
    username: 'admin',
    role: 'admin',
    status: 'active',
  });
  if (profileError) console.error('Error creating profile:', profileError);
  else console.log('✅ Admin created: admin@freehost.com / Admin$ecure123!');
}

seedAdmin();
```

Run with: `node supabase/seed.js`

---

## ⚙️ Backend API (Express)

### `backend/server.js`
```javascript
import express from 'express';
import cors from 'cors';
import multer from 'multer';
import { authRouter } from './routes/auth.js';
import { sitesRouter } from './routes/sites.js';
import { filesRouter } from './routes/files.js';
import { adminRouter } from './routes/admin.js';
import { subdomainRouter } from './routes/subdomain.js';

const app = express();
app.use(cors({ origin: process.env.FRONTEND_URL }));
app.use(express.json());

// API routes
app.use('/api/auth',  authRouter);
app.use('/api/sites', sitesRouter);
app.use('/api/files', filesRouter);
app.use('/api/admin', adminRouter);

// Subdomain file serving (for *.freehost.com)
app.use(subdomainRouter);

app.listen(3000, () => console.log('FreeHost API running on :3000'));
```

### Auth Routes (`/api/auth`)
```
POST /api/auth/signup   → create account + profile row
POST /api/auth/login    → Supabase auth → return JWT
POST /api/auth/logout   → invalidate session
GET  /api/auth/me       → return current user + profile
```

### Sites Routes (`/api/sites`)
```
GET    /api/sites              → list user's sites
POST   /api/sites              → create site (check subdomain availability)
GET    /api/sites/:id          → get site details
DELETE /api/sites/:id          → delete site + all files
POST   /api/sites/:id/deploy   → trigger deployment
GET    /api/sites/:id/deploys  → deployment history
GET    /api/sites/:id/analytics → visit/bandwidth stats
```

### Files Routes (`/api/files/:siteId`)
```
GET    /api/files/:siteId           → list all files
GET    /api/files/:siteId/*         → get file content
PUT    /api/files/:siteId/*         → upload/update file
DELETE /api/files/:siteId/*         → delete file
POST   /api/files/:siteId/bulk      → batch upload (ZIP)
POST   /api/files/:siteId/extract   → extract ZIP
```

### Admin Routes (`/api/admin`)
```
GET    /api/admin/users             → all users
PATCH  /api/admin/users/:id/status  → suspend/reactivate
DELETE /api/admin/users/:id         → delete user + sites
GET    /api/admin/sites             → all sites
DELETE /api/admin/sites/:id         → delete site
GET    /api/admin/stats             → global analytics
GET    /api/admin/logs              → server activity logs
GET    /api/admin/settings          → system_settings table
PUT    /api/admin/settings          → update settings
```

---

## 🚀 Deployment Pipeline Implementation

```javascript
// backend/routes/sites.js — deploy endpoint
router.post('/:id/deploy', requireAuth, async (req, res) => {
  const { id } = req.params;
  const site = await getSiteByIdAndOwner(id, req.user.id);
  if (!site) return res.status(404).json({ error: 'Site not found' });

  // Create deployment record
  const { data: deploy } = await supabase.from('deployments').insert({
    site_id: id, status: 'running'
  }).select().single();

  // Async deployment pipeline
  deployPipeline(site, deploy.id, req.user.id).catch(console.error);

  res.json({ deploymentId: deploy.id, status: 'running' });
});

async function deployPipeline(site, deployId, userId) {
  const log = [];
  const addLog = async (msg) => {
    log.push(msg);
    await supabase.from('deployments').update({ log: log.join('\n') }).eq('id', deployId);
  };

  try {
    await addLog('▶ Starting deployment...');
    
    // List files in storage bucket
    const { data: files } = await supabase.storage.from('sites').list(`${userId}/${site.id}`);
    await addLog(`  Found ${files.length} files`);

    // Calculate total size
    const totalSize = files.reduce((sum, f) => sum + (f.metadata?.size || 0), 0);
    await addLog(`  Total size: ${(totalSize/1024).toFixed(1)} KB`);

    // Update site record
    await supabase.from('sites').update({
      status: 'live',
      storage_bytes: totalSize,
      last_deploy: new Date().toISOString()
    }).eq('id', site.id);

    await addLog('  Purging CDN cache...');
    // In production: call CDN API (Cloudflare, etc.)
    await new Promise(r => setTimeout(r, 500));

    await addLog(`✔ Deployed to ${site.subdomain}.freehost.com`);
    await supabase.from('deployments').update({ status: 'success', log: log.join('\n') }).eq('id', deployId);
  } catch (err) {
    await addLog(`✖ Error: ${err.message}`);
    await supabase.from('deployments').update({ status: 'failed', log: log.join('\n') }).eq('id', deployId);
  }
}
```

---

## 🌐 NGINX Wildcard Subdomain Config

```nginx
# nginx/freehost.conf

# Redirect HTTP → HTTPS
server {
    listen 80;
    server_name freehost.com *.freehost.com;
    return 301 https://$host$request_uri;
}

# Main platform (app.freehost.com / freehost.com)
server {
    listen 443 ssl http2;
    server_name freehost.com www.freehost.com app.freehost.com;

    ssl_certificate     /etc/letsencrypt/live/freehost.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/freehost.com/privkey.pem;
    include             /etc/letsencrypt/options-ssl-nginx.conf;

    location /api {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_read_timeout 60s;
    }

    location / {
        root /var/www/freehost/frontend/dist;
        try_files $uri /index.html;
        add_header Cache-Control "no-cache";
    }
}

# Wildcard — user sites (*.freehost.com)
server {
    listen 443 ssl http2;
    server_name ~^(?<subdomain>[a-z0-9-]+)\.freehost\.com$;

    ssl_certificate     /etc/letsencrypt/live/freehost.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/freehost.com/privkey.pem;

    # Security headers
    add_header X-Frame-Options SAMEORIGIN;
    add_header X-Content-Type-Options nosniff;
    add_header Content-Security-Policy "default-src 'self' 'unsafe-inline' 'unsafe-eval' data: blob:;";

    # Proxy to Node.js which reads from Supabase Storage
    location / {
        proxy_pass http://localhost:3000/_sites/$subdomain$request_uri;
        proxy_set_header X-Subdomain $subdomain;
    }
}
```

### Subdomain file-serving route (Node.js)
```javascript
// backend/routes/subdomain.js
router.get('/_sites/:subdomain/*', async (req, res) => {
  const { subdomain } = req.params;
  const filePath = req.params[0] || 'index.html';

  const site = await supabase.from('sites').select('id,owner_id,status').eq('subdomain', subdomain).single();
  if (!site.data || site.data.status !== 'live') return res.status(404).send('Site not found');

  const { data, error } = await supabase.storage
    .from('sites')
    .download(`${site.data.owner_id}/${site.data.id}/${filePath}`);
  
  if (error) return res.status(404).send('File not found');
  
  const mimeTypes = { html:'text/html', css:'text/css', js:'application/javascript', png:'image/png', svg:'image/svg+xml', ico:'image/x-icon' };
  const ext = filePath.split('.').pop();
  res.setHeader('Content-Type', mimeTypes[ext] || 'text/plain');
  res.send(Buffer.from(await data.arrayBuffer()));
});
```

---

## 🔒 Auth Middleware

```javascript
// backend/middleware/auth.js
import { supabase } from '../lib/supabase.js';

export async function requireAuth(req, res, next) {
  const token = req.headers.authorization?.split(' ')[1];
  if (!token) return res.status(401).json({ error: 'Not authenticated' });

  const { data: { user }, error } = await supabase.auth.getUser(token);
  if (error || !user) return res.status(401).json({ error: 'Invalid token' });

  const { data: profile } = await supabase.from('profiles').select('*').eq('id', user.id).single();
  if (profile?.status === 'suspended') return res.status(403).json({ error: 'Account suspended' });

  req.user = { ...user, ...profile };
  next();
}

export async function requireAdmin(req, res, next) {
  await requireAuth(req, res, async () => {
    if (req.user.role !== 'admin') return res.status(403).json({ error: 'Admin access required' });
    next();
  });
}
```

---

## 🖥️ Frontend Monaco Editor Integration

```jsx
// frontend/src/components/Editor/CodeEditor.jsx
import Editor from '@monaco-editor/react';

export function CodeEditor({ value, language, onChange, readOnly = false }) {
  return (
    <Editor
      height="100%"
      language={language}
      value={value}
      theme="vs-dark"
      onChange={onChange}
      options={{
        readOnly,
        minimap: { enabled: false },
        fontSize: 13,
        lineHeight: 1.7,
        fontFamily: 'Geist Mono, Fira Code, monospace',
        tabSize: 2,
        wordWrap: 'on',
        scrollBeyondLastLine: false,
        renderLineHighlight: 'gutter',
        cursorBlinking: 'smooth',
        formatOnPaste: true,
        formatOnType: true,
      }}
    />
  );
}
```

---

## 📦 Environment Variables

```bash
# .env.example

# Supabase
VITE_SUPABASE_URL=https://your-project.supabase.co
VITE_SUPABASE_ANON_KEY=your-anon-key
SUPABASE_SERVICE_ROLE_KEY=your-service-role-key

# Backend
PORT=3000
FRONTEND_URL=https://freehost.com
JWT_SECRET=your-jwt-secret-change-this

# Storage
SUPABASE_STORAGE_BUCKET=sites

# Domain
BASE_DOMAIN=freehost.com
```

---

## 📋 Local Development Setup

```bash
# 1. Clone and install
git clone https://github.com/yourname/freehost
cd freehost

# 2. Install frontend deps
cd frontend && npm install

# 3. Install backend deps
cd ../backend && npm install

# 4. Set up Supabase
#    - Create project at supabase.com
#    - Run SQL from supabase/migrations/001_init.sql in the SQL editor
#    - Create a "sites" Storage bucket (public: false)

# 5. Configure environment
cp .env.example .env
# Fill in your Supabase URL, keys, etc.

# 6. Seed admin user
node supabase/seed.js

# 7. Start development
# Terminal 1:
cd frontend && npm run dev   # Vite on :5173

# Terminal 2:
cd backend && npm run dev    # Express on :3000
```

---

## ⭐ Bonus Features (Implementation Notes)

### ZIP Export
```javascript
router.get('/api/sites/:id/export', requireAuth, async (req, res) => {
  const archiver = require('archiver');
  const archive = archiver('zip');
  res.setHeader('Content-Type', 'application/zip');
  res.setHeader('Content-Disposition', `attachment; filename="${site.subdomain}.zip"`);
  archive.pipe(res);
  for (const file of files) {
    const content = await downloadFile(file.path);
    archive.append(content, { name: file.path });
  }
  archive.finalize();
});
```

### Let's Encrypt Auto-SSL for Custom Domains
```bash
# Using certbot with DNS challenge per custom domain
certbot certonly --webroot -w /var/www/freehost/certbot \
  -d mycustomdomain.com --non-interactive --agree-tos
```

---

## 📝 License
MIT — Free to use, modify, deploy.
