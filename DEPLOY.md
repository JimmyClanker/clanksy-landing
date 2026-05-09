# Deploy clanksy.ai — istruzioni

Ultimo deploy verificato da Codex: 2026-05-09, Cloudflare Pages project
`clanksy-landing`, branch `main`, preview
`https://b73a457e.clanksy-landing.pages.dev`.

Il token locale in `.env` ha permesso Pages deploy al 2026-05-09. Non stampare
mai il valore del token e non committare `.env` o copie tipo `.env.save`.

## Opzione A — Cloudflare Pages (raccomandata)

Tutto su Cloudflare: niente terzi, dominio già gestito da loro, deploy istantaneo.

### 1. API token richiesto

Se il token locale scade o viene ruotato, crea un nuovo API token con permission
Pages Edit.

https://dash.cloudflare.com/profile/api-tokens → "Create Token" → custom token con:
- **Account permissions**: `Cloudflare Pages` → `Edit`
- **Zone permissions**: `clanksy.ai` → `DNS` → `Edit`
- **Zone permissions**: `clanksy.ai` → `Workers Routes` → `Edit` (per redirect dopo)

Copia il token. Sostituiscilo in `.env`:
```bash
# .env
CLOUDFLARE_API_TOKEN=<nuovo-token>
```

### 2. Deploy

Da terminale:
```bash
cd /Users/b00lish/Documents/Clawnkers/whatsapp-automation
export CLOUDFLARE_API_TOKEN=$(grep CLOUDFLARE_API_TOKEN .env | cut -d= -f2)
export CLOUDFLARE_ACCOUNT_ID=bf2583052d46a3a9acc807bc069c9dda

# Crea il project solo se manca
npx wrangler pages project create clanksy-landing --production-branch=main

# Deploy
npx wrangler pages deploy landing --project-name=clanksy-landing --branch=main
```

Output atteso: URL temporaneo `https://<hash>.clanksy-landing.pages.dev`.
Verifica poi `https://clanksy.ai/`, `https://clanksy.ai/es` e
`https://clanksy.ai/en`.

### 3. Connetti dominio custom clanksy.ai

```bash
# Aggiungi il dominio al project
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects/clanksy-landing/domains" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"clanksy.ai"}'

# Aggiungi anche www.clanksy.ai
curl -X POST "https://api.cloudflare.com/client/v4/accounts/$CLOUDFLARE_ACCOUNT_ID/pages/projects/clanksy-landing/domains" \
  -H "Authorization: Bearer $CLOUDFLARE_API_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"name":"www.clanksy.ai"}'
```

Cloudflare auto-configura DNS + SSL (certificati Let's Encrypt). Tempo di propagazione: ~30 secondi.

### 4. Verifica

Apri https://clanksy.ai — deve mostrare la landing italiana. https://clanksy.ai/es.html spagnolo, https://clanksy.ai/en.html inglese.

---

## Opzione B — GitHub Pages + Cloudflare DNS

Se preferisci un repo GitHub dedicato.

### 1. Crea repo GitHub

```bash
gh repo create JimmyClanker/clanksy-landing --public --description "Clanksy landing page"
```

### 2. Copia files + push

```bash
cd /tmp
git clone git@github.com:JimmyClanker/clanksy-landing.git
cp -r /Users/b00lish/Documents/Clawnkers/whatsapp-automation/landing/* /tmp/clanksy-landing/
cd /tmp/clanksy-landing
echo "clanksy.ai" > CNAME
git add . && git commit -m "Initial landing"
git push
```

### 3. Abilita GitHub Pages

Settings repo → Pages → Source: `main` branch / root → Save.

### 4. DNS Cloudflare

Apri dashboard CF → clanksy.ai → DNS → aggiungi:
- `CNAME` `clanksy.ai` → `jimmyclanker.github.io` (proxied OFF inizialmente, poi ON dopo verifica)
- `CNAME` `www` → `jimmyclanker.github.io`

GitHub auto-emette certificato (~10 min).

---

## Opzione C — Vercel (full-managed, gratis)

Per chi non vuole pensieri.

```bash
npx vercel --cwd /Users/b00lish/Documents/Clawnkers/whatsapp-automation/landing/
```

Domande interactive Vercel: nome progetto, git, etc. Al termine output URL. Da dashboard Vercel aggiungi dominio custom `clanksy.ai`. Vercel ti dice quale CNAME aggiungere su Cloudflare DNS.

---

## Form demo: webhook backend

Il form della landing punta a:
```
https://api.clanksy.ai/webhook/landing-demo-request
```

L'endpoint puo' essere sovrascritto a runtime creando `landing/config.js`
prima del deploy Pages. Il file e' ignorato da git per non versionare URL
temporanei:

```js
window.CLANKSY_FORM_ENDPOINT = 'https://<tunnel>.trycloudflare.com/webhook/landing-demo-request';
```

`api.clanksy.ai` resta il default corretto per produzione stabile. Non puntarlo
con CNAME proxied a `trycloudflare.com`: Cloudflare risponde con errore 1014
cross-user CNAME. Per il dev live temporaneo usa `landing/config.js`; per la
produzione usare VPS/Caddy o Cloudflare Tunnel nominato con hostname custom.

Il workflow n8n `41_landing_demo_request` è da creare. Stub veloce: webhook POST → INSERT in nuova tabella `demo_requests` → notifica Andrea via WhatsApp.

## Asset mancanti (TODO)

- **Logo Clanksy**: il favicon attuale è un wordmark "C" generato via SVG inline. Quando hai il logo definitivo (3 giorni), sostituisci `link rel="icon"` nei 3 HTML con il file reale + aggiungi anche `og:image` per share preview social.
- **Open Graph image**: niente foto sociali per ora. Aggiungere `landing/assets/og-image.jpg` 1200×630 quando hai screenshot dashboard / logo.
- **Privacy policy**: link in footer punta a `/privacy.html` non ancora creata. Da redigere (Iubenda potrebbe aiutare).
- **Logo MCP per design futuro**: valuta installare `Figma MCP` o `shadcn MCP` per iterazioni più rapide su component design.
