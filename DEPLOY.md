# Deploy clanksy.ai — istruzioni

Ultimo stato verificato da Codex: 2026-05-10. Il dominio pubblico è in
fallback temporaneo su GitHub Pages repository `JimmyClanker/clanksy-landing`,
con HTTPS attivo. Cloudflare gestisce ancora il DNS, ma Cloudflare Pages è
bypassato perché `clanksy-landing.pages.dev` andava in timeout.

Stato live atteso durante il fallback:
- `https://clanksy.ai/` risponde da GitHub Pages.
- `http://clanksy.ai/` redirige a `https://clanksy.ai/`.
- GitHub Pages ha certificato approvato per `clanksy.ai` e `www.clanksy.ai`;
  `Enforce HTTPS` è attivo.
- Il form demo usa di default `https://api.clanksy.ai/webhook/landing-demo-request`.
  Al 2026-05-10 punta al tunnel nominato Cloudflare verso n8n locale.

Il token locale in `.env` ha permesso Pages deploy al 2026-05-09. Non stampare
mai il valore del token e non committare `.env` o copie tipo `.env.save`.

## Opzione A — Cloudflare Pages

Tutto su Cloudflare: niente terzi, dominio già gestito da loro, deploy istantaneo.
Al 2026-05-09 è stata disattivata come strada live perché il project Pages
timeoutava anche dopo rollback statico. Prima di riusarla, testare su staging
o preview senza spostare DNS production.

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

Fallback attualmente usato in produzione.

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
- record apex `A` DNS-only verso gli IP GitHub Pages:
  `185.199.108.153`, `185.199.109.153`, `185.199.110.153`, `185.199.111.153`
- `CNAME` `www` → `jimmyclanker.github.io` DNS-only

GitHub auto-emette il certificato. Non attivare `Enforce HTTPS` finché
l'API GitHub Pages non mostra un certificato disponibile.

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

### Stato attuale `api.clanksy.ai`

Al 2026-05-10 `api.clanksy.ai` e' operativo come Cloudflare Tunnel nominato
verso n8n locale:

```text
api.clanksy.ai -> <tunnel_id>.cfargotunnel.com -> http://127.0.0.1:5678
```

Sul Mac di sviluppo il token e' locale/non versionato:

```text
.env -> CLOUDFLARE_TUNNEL_TOKEN
~/Library/Application Support/Clanksy/cloudflare_tunnel_token
```

Il tunnel e' tenuto vivo dal LaunchAgent utente:

```bash
launchctl print "gui/$(id -u)/com.clanksy.api-tunnel"
```

Nota sicurezza: non avviare `cloudflared` con `--token <valore>` in command
line, perche' il token compare in `ps`. Usare la variabile env `TUNNEL_TOKEN`
o il LaunchAgent locale.

Il workflow n8n `41_landing_demo_request` esiste e inserisce in `demo_requests`.
Per produzione stabile, configurare `api.clanksy.ai` su VPS/Caddy:

```env
N8N_HOST=api.clanksy.ai
PUBLIC_HOSTS=api.clanksy.ai
```

Poi creare record DNS `A api.clanksy.ai -> <IP_VPS>` e verificare:

```bash
curl -i https://api.clanksy.ai/webhook/landing-demo-request
```

Una risposta `404`/method-not-allowed su GET è accettabile per smoke test;
il POST del form va testato con payload controllato.

## Asset mancanti (TODO)

- **Logo Clanksy**: il favicon attuale è un wordmark "C" generato via SVG inline. Quando hai il logo definitivo (3 giorni), sostituisci `link rel="icon"` nei 3 HTML con il file reale + aggiungi anche `og:image` per share preview social.
- **Open Graph image**: niente foto sociali per ora. Aggiungere `landing/assets/og-image.jpg` 1200×630 quando hai screenshot dashboard / logo.
- **Privacy policy**: link in footer punta a `/privacy.html` non ancora creata. Da redigere (Iubenda potrebbe aiutare).
- **Logo MCP per design futuro**: valuta installare `Figma MCP` o `shadcn MCP` per iterazioni più rapide su component design.
