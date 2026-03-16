# stavby-template

> 🇨🇿 [Česky](#-česky--onboarding-nové-firmy) | 🇬🇧 [English](#-english--onboarding-a-new-company)

---

## 🇨🇿 Česky — Onboarding nové firmy

### Co je stavby-template

Tento repozitář je šablona pro aplikaci Stavby. Každá firma dostane vlastní fork (kopii) tohoto repozitáře s vlastní databází a vlastním hostingem. Kód aplikace je identický — liší se pouze konfigurační soubor `.env`.

```
stavby-template  ← tento repozitář (nikdy se nenasazuje přímo)
    ├── stavby-znojmo    ← fork pro Firmu 1 (ostrá produkce)
    ├── stavby-brno      ← fork pro Firmu 2
    └── stavby-[nazev]   ← fork pro každou další firmu
```

### Architektura jedné instance

Každá firma má **dvě prostředí** — produkci a staging:

| Prostředí | Větev  | Supabase projekt | URL                        |
|-----------|--------|------------------|----------------------------|
| Produkce  | `main` | produkční DB     | stavby-firma.pages.dev     |
| Staging   | `staging` | testovací DB  | stavby-firma-staging.pages.dev |

---

### Krok za krokem: přidání nové firmy

#### 1. Forkni repozitář

1. Přejdi na `github.com/doco1971/stavby-template`
2. Klikni na **Fork** → pojmenuj ho `stavby-[nazev-firmy]` (např. `stavby-brno`)
3. Nastav ho jako **Private** pokud data firmy nesmí být veřejná

#### 2. Vytvoř Supabase projekty

Pro každou firmu vytvoř **dva** Supabase projekty (každý vyžaduje vlastní Supabase účet nebo jiný název):

- `stavby-[firma]-prod` — ostrá produkce
- `stavby-[firma]-staging` — testovací prostředí

**V každém projektu spusť v SQL editoru tyto migrace:**

```sql
-- Tabulky
CREATE TABLE IF NOT EXISTS stavby (
  id BIGSERIAL PRIMARY KEY,
  firma TEXT, cislo_stavby TEXT, nazev_stavby TEXT,
  ps_i NUMERIC, snk_i NUMERIC, bo_i NUMERIC,
  ps_ii NUMERIC, bo_ii NUMERIC, poruch NUMERIC,
  vyfakturovano NUMERIC, ukonceni TEXT, zrealizovano NUMERIC,
  sod TEXT, ze_dne TEXT, objednatel TEXT, stavbyvedouci TEXT,
  nabidkova_cena NUMERIC, cislo_faktury TEXT, castka_bez_dph NUMERIC,
  splatna TEXT, poznamka TEXT,
  cislo_faktury_2 TEXT, castka_bez_dph_2 NUMERIC, splatna_2 TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS ciselniky (
  id BIGSERIAL PRIMARY KEY,
  typ TEXT, hodnota TEXT, barva TEXT
);

CREATE TABLE IF NOT EXISTS uzivatele (
  id BIGSERIAL PRIMARY KEY,
  email TEXT UNIQUE, heslo TEXT, role TEXT, name TEXT
);

CREATE TABLE IF NOT EXISTS log_aktivit (
  id BIGSERIAL PRIMARY KEY,
  akce TEXT, detail TEXT, uzivatel TEXT,
  cas TIMESTAMPTZ DEFAULT NOW()
);

CREATE TABLE IF NOT EXISTS nastaveni (
  id BIGSERIAL PRIMARY KEY,
  klic TEXT UNIQUE, hodnota TEXT
);

-- RLS políčky
ALTER TABLE stavby ENABLE ROW LEVEL SECURITY;
ALTER TABLE ciselniky ENABLE ROW LEVEL SECURITY;
ALTER TABLE uzivatele ENABLE ROW LEVEL SECURITY;
ALTER TABLE log_aktivit ENABLE ROW LEVEL SECURITY;
ALTER TABLE nastaveni ENABLE ROW LEVEL SECURITY;

CREATE POLICY "allow_all" ON stavby USING (true) WITH CHECK (true);
CREATE POLICY "allow_all" ON ciselniky USING (true) WITH CHECK (true);
CREATE POLICY "allow_all" ON uzivatele USING (true) WITH CHECK (true);
CREATE POLICY "allow_all" ON nastaveni USING (true) WITH CHECK (true);
CREATE POLICY "admin_read_all" ON log_aktivit FOR SELECT USING (true);
CREATE POLICY "allow_insert" ON log_aktivit FOR INSERT WITH CHECK (true);

-- První superadmin uživatel (změň email a heslo!)
INSERT INTO uzivatele (email, heslo, role, name)
VALUES ('admin@firma.cz', 'ZMEN_HESLO', 'superadmin', 'Admin')
ON CONFLICT (email) DO NOTHING;
```

#### 3. Nastav .env soubor

Zkopíruj `.env.template` na `.env` a vyplň hodnoty z Supabase dashboardu:

```bash
cp .env.template .env
```

```env
VITE_SB_URL=https://TVUJ-PROJEKT.supabase.co
VITE_SB_KEY=tvuj-anon-klic
```

> ⚠️ Nikdy nepushuj `.env` do Gitu — obsahuje tajné klíče. Soubor je v `.gitignore`.

#### 4. Vytvoř větev staging

```bash
git checkout -b staging
git push origin staging
```

#### 5. Nasaď na Cloudflare Pages

**Pro produkci (větev `main`):**
1. Přejdi na [pages.cloudflare.com](https://pages.cloudflare.com)
2. Create a project → Connect to Git → vyber `stavby-[firma]`
3. Nastavení buildu:
   - Framework preset: **Vite**
   - Build command: `npm run build`
   - Build output directory: `dist`
4. Environment Variables → přidej `VITE_SB_URL` a `VITE_SB_KEY` (produkční hodnoty)
5. Deploy!

**Pro staging (větev `staging`):**
- Totéž, ale jako druhý projekt v Cloudflare Pages
- Environment Variables → přidej staging hodnoty (`VITE_SB_URL_STAGING`, `VITE_SB_KEY_STAGING`)

#### 6. Nastav GitHub Actions Secrets

Aby heartbeat workflow fungoval, přidej secrets do GitHub repozitáře:

`Settings → Secrets and variables → Actions → New repository secret`

| Secret name       | Hodnota                              |
|-------------------|--------------------------------------|
| `VITE_SB_URL`     | URL produkčního Supabase projektu    |
| `VITE_SB_KEY`     | Anon klíč produkčního projektu       |

Pokud máš staging, přidej taky:

| Secret name              | Hodnota                           |
|--------------------------|-----------------------------------|
| `VITE_SB_URL_STAGING`    | URL stagingového projektu         |
| `VITE_SB_KEY_STAGING`    | Anon klíč stagingového projektu   |

Heartbeat se spustí automaticky každé pondělí a čtvrtek — Supabase Free projekt se nikdy neuspí.

#### 7. Nastav Resend pro emaily (volitelné)

Pokud firma chce denní email notifikace o prošlých termínech:

1. Vytvoř účet na [resend.com](https://resend.com)
2. Ověř doménu firmy (DNS záznamy — nutná pomoc IT správce)
3. V Supabase → Edge Functions → Manage secrets přidej:
   - `RESEND_API_KEY` — API klíč z Resend
   - `FROM_EMAIL` — odesílací adresa (např. `stavby@firma.cz`)
4. Nasaď Edge Function `send-deadline-emails` (viz složka `supabase/functions/`)
5. Nastav pg_cron job:
   ```sql
   SELECT cron.schedule('deadline-emails', '0 5 * * *', $$
     SELECT net.http_post(
       url := current_setting('app.supabase_url') || '/functions/v1/send-deadline-emails',
       headers := '{"Authorization": "Bearer ' || current_setting('app.service_role_key') || '"}'::jsonb
     );
   $$);
   ```

---

### Šíření oprav kódu do všech forků

Když opravíš chybu nebo přidáš novou funkci v `stavby-template`, propagace do forků:

```bash
# V repozitáři každé firmy (jednou nastavit upstream):
git remote add upstream https://github.com/doco1971/stavby-template.git

# Při každé aktualizaci:
git fetch upstream
git merge upstream/main
git push origin main
```

Pokud firma nechce Git → udělej to za ně (5 minut práce, stačí přístup ke GitHub repozitáři).

---

### Přehled instancí

| Firma | GitHub repo | Produkce | Staging | Supabase |
|-------|-------------|----------|---------|----------|
| Znojmo | stavby-znojmo | ✅ | ⬜ připravit | ✅ prod |
| _(další)_ | stavby-[název] | ⬜ | ⬜ | ⬜ |

---

## 🇬🇧 English — Onboarding a new company

### What is stavby-template

This repository is the base template for the Stavby (Construction Projects) management app. Each company gets their own fork with their own database and hosting. The application code is identical — only the `.env` configuration file differs.

```
stavby-template  ← this repo (never deployed directly)
    ├── stavby-znojmo    ← fork for Company 1 (production)
    ├── stavby-brno      ← fork for Company 2
    └── stavby-[name]    ← fork for each additional company
```

### One instance — two environments

| Environment | Branch    | Supabase project | URL                         |
|-------------|-----------|------------------|-----------------------------|
| Production  | `main`    | production DB    | stavby-company.pages.dev    |
| Staging     | `staging` | test DB          | stavby-company-staging.pages.dev |

---

### Step by step: adding a new company

#### 1. Fork the repository

1. Go to `github.com/doco1971/stavby-template`
2. Click **Fork** → name it `stavby-[company-name]` (e.g. `stavby-brno`)
3. Set it to **Private** if the company's data must not be public

#### 2. Create Supabase projects

Create **two** Supabase projects per company:
- `stavby-[company]-prod` — live production
- `stavby-[company]-staging` — test environment

Run the SQL migrations from the Czech section above in each project's SQL editor.

#### 3. Configure .env

```bash
cp .env.template .env
# Fill in VITE_SB_URL and VITE_SB_KEY from the Supabase dashboard
```

> ⚠️ Never commit `.env` to Git — it contains secret keys. It's already in `.gitignore`.

#### 4. Create staging branch

```bash
git checkout -b staging
git push origin staging
```

#### 5. Deploy to Cloudflare Pages

1. Go to [pages.cloudflare.com](https://pages.cloudflare.com)
2. Create a project → Connect to Git → select `stavby-[company]`
3. Build settings:
   - Framework preset: **Vite**
   - Build command: `npm run build`
   - Build output directory: `dist`
4. Add Environment Variables: `VITE_SB_URL` and `VITE_SB_KEY`
5. Deploy!

Repeat for the staging branch as a second Cloudflare Pages project with staging `.env` values.

#### 6. Set GitHub Actions Secrets

`Settings → Secrets and variables → Actions → New repository secret`

| Secret name    | Value                              |
|----------------|------------------------------------|
| `VITE_SB_URL`  | Production Supabase project URL    |
| `VITE_SB_KEY`  | Production Supabase anon key       |

The heartbeat workflow (`.github/workflows/supabase-heartbeat.yml`) will automatically ping Supabase every Monday and Thursday so the Free tier project never gets paused.

#### 7. Propagating code fixes to all forks

```bash
# Set upstream once per fork:
git remote add upstream https://github.com/doco1971/stavby-template.git

# Pull updates:
git fetch upstream
git merge upstream/main
git push origin main
```

---

### Quick reference

| Task | Where |
|------|-------|
| Supabase URL + key | Supabase Dashboard → Project Settings → API |
| Supabase Edge Function secrets | Edge Functions → Manage secrets |
| Cloudflare Pages env vars | Pages project → Settings → Environment variables |
| GitHub Actions secrets | Repo → Settings → Secrets and variables → Actions |
| Heartbeat schedule | `.github/workflows/supabase-heartbeat.yml` |
| App config (emails, company name) | App → Nastavení → Aplikace |
