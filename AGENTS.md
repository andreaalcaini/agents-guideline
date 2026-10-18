# AGENTS.md

Linee guida operative per agenti AI (Claude Code, Antigravity, Cursor, Codex) e sviluppatori in questo repository.
La mappa dei file, architettura, porte e componenti è descritta in [STRUCTURE.md](STRUCTURE.md): leggilo per primo per orientarti.

---

## 🚨 REGOLA AUREA: Cloudflare Tunnel Mesh (`cloudflared-mesh`)

Ogni volta che crei o modifichi un file `docker-compose.yml` per aggiungere un nuovo servizio, devi:
1. Collegare il servizio alla rete esterna `cloudflared-mesh`:
   ```yaml
   services:
     <nome-servizio>:
       # ... altre configurazioni del servizio ...
       networks:
         - default
         - cloudflared-mesh

   networks:
     default:
     cloudflared-mesh:
       external: true
       name: cloudflared-mesh
   ```
2. Specificare l'alias per la mesh sotto il servizio:
   ```yaml
       networks:
         cloudflared-mesh:
           aliases:
             - <nomeservizio>.mesh
   ```
3. ⚠️ **CHIEDERE SEMPRE ALL'UTENTE:**
   > *"Quale alias desideri assegnare a questo servizio per la rete Cloudflare (ad es. `<nomeservizio>.mesh`)?"*
   Non inventare o finalizzare mai l'alias senza aver chiesto e ricevuto conferma dall'utente.

---

## ⚡ DOCKER MULTI-ARCH PERFORMANCE REQUIREMENT (`--platform=$BUILDPLATFORM`)

### Il problema QEMU:
I runner GitHub Actions girano su architettura AMD64 (`linux/amd64`). Quando si compila per ARM64 (es. Raspberry Pi, Mac Apple Silicon), l'emulazione QEMU durante `npm install` o la build del codice è lentissima (freeze e timeout oltre 90 minuti).

### Lo standard:
Negli stage di build e installazione dipendenze usa SEMPRE `--platform=$BUILDPLATFORM` per sfruttare la velocità nativa della CPU del runner (build in ~30 secondi). Nello stage finale (`runner`) usa l'architettura target copiando gli asset già compilati.

### Template universali per framework:

#### Next.js (Standalone)
*Richiede `output: 'standalone'` in `next.config.ts`*
```dockerfile
# 1. Dipendenze su architettura nativa veloce
FROM --platform=$BUILDPLATFORM node:20-alpine AS deps
RUN apk add --no-cache libc6-compat
WORKDIR /app
COPY package.json package-lock.json* pnpm-lock.yaml* yarn.lock* ./
RUN npm ci

# 2. Builder nativo x86_64
FROM --platform=$BUILDPLATFORM node:20-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
ENV NEXT_TELEMETRY_DISABLED=1
RUN npm run build

# 3. Runner finale target
FROM node:20-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
ENV NEXT_TELEMETRY_DISABLED=1
ENV PORT=3000
ENV HOSTNAME="0.0.0.0"
RUN addgroup --system --gid 1001 nodejs && adduser --system --uid 1001 nextjs
RUN mkdir -p /app/data && chown -R nextjs:nodejs /app/data
COPY --from=builder /app/public ./public
COPY --from=builder --chown=nextjs:nodejs /app/.next/standalone ./
COPY --from=builder --chown=nextjs:nodejs /app/.next/static ./.next/static
USER nextjs
EXPOSE 3000
CMD ["node", "server.js"]
```

#### Vue / React / Vite / SPA (Nginx)
```dockerfile
# 1. Compilazione nativa x86_64
FROM --platform=$BUILDPLATFORM node:20-alpine AS builder
WORKDIR /app
COPY package.json package-lock.json* ./
RUN npm ci
COPY . .
RUN npm run build

# 2. Web server Nginx minimale
FROM nginx:alpine AS runner
COPY --from=builder /app/dist /usr/share/nginx/html
RUN echo 'server { \
    listen 80; \
    location / { \
        root /usr/share/nginx/html; \
        index index.html index.htm; \
        try_files $uri $uri/ /index.html; \
    } \
}' > /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### Python (FastAPI / Flask)
```dockerfile
# 1. Builder dipendenze
FROM --platform=$BUILDPLATFORM python:3.11-slim AS builder
WORKDIR /app
RUN apt-get update && apt-get install -y --no-install-recommends gcc && rm -rf /var/lib/apt/lists/*
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# 2. Runner finale
FROM python:3.11-slim AS runner
WORKDIR /app
ENV PYTHONUNBUFFERED=1
COPY --from=builder /root/.local /root/.local
ENV PATH=/root/.local/bin:$PATH
COPY . .
EXPOSE 8000
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

#### File `.dockerignore` Universale
```
node_modules
.next
dist
build
.git
.github
*.local
.env*.local
data/*
!data/.gitkeep
npm-debug.log*
```

---

## 🚀 PIPELINE CI/CD & GITHUB PACKAGES (GHCR)

### File `.github/workflows/deploy.yml`:
- Trigger su push verso `main`/`master` e `workflow_dispatch`.
- `concurrency: cancel-in-progress: true` per interrompere build obsolete.
- QEMU + Docker Buildx per architetture `linux/amd64,linux/arm64`.
- Autenticazione con fallback `${{ secrets.GHCR_PAT || secrets.GITHUB_TOKEN }}`.
- Cache GitHub Actions (`cache-from/to: type=gha`).

```yaml
name: Build and Push to GHCR

on:
  push:
    branches: [ "master", "main" ]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Set up QEMU (ARM64)
        uses: docker/setup-qemu-action@v3

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Log in to GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GHCR_PAT || secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=raw,value=latest
            type=sha,format=short

      - name: Build and push multi-arch Docker image
        uses: docker/build-push-action@v5
        with:
          context: .
          platforms: linux/arm64,linux/amd64
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

### Regola d'oro Token GitHub (PAT):
- ⚠️ **NON USARE Fine-Grained Tokens (`github_pat_...`)**: non sono supportati da GHCR per namespace utente e causano l'errore `denied: permission_denied: The token provided does not match expected scopes`.
- ✅ **USARE SEMPRE Personal Access Token (Classic) (`ghp_...`)**:
  - Permessi obbligatori: `write:packages` (include `read:packages`), `repo`, `workflow`.
  - Salvare in `Settings -> Secrets and variables -> Actions` come secret `GHCR_PAT`.
- **Visibilità Package:** Impostare il package su **Public** in GitHub Packages così i server remoti possono fare `docker compose pull` senza autenticazione o problemi di token scaduti.

---

## 🔄 WORKFLOW DI SVILUPPO & REGOLE OPERATIVE

1. **Pianificazione prima delle modifiche non banali:**
   Per nuove feature, refactoring o modifiche multi-file: analizza il codice, presenta il piano (file toccati, logica, trade-off) e attendi approvazione prima di modificare.
2. **Usa i dev tools durante lo sviluppo:**
   Usa `npm run dev` / dev server con hot reload per testare le modifiche. Compila l'immagine Docker solo per verificare il rilascio o il container finale.
3. **Commits convenzionali a step:**
   Fai commit piccoli e frequenti (`feat:`, `fix:`, `refactor:`, `chore:`). Non fare un unico commit gigante alla fine.
4. **Nessun effetto collaterale non autorizzato:**
   Non inviare email, pagamenti o messaggi a utenti reali. I test usano mock o ambienti sandbox.
5. **Persistenza e cartella dati:**
   Tutti i dati persistenti (SQLite, upload, JSON) risiedono in `./data` (mappato nel container in `/app/data`). Includi `.gitignore` per escludere `data/*` tranne `!data/.gitkeep`.
6. **Allinea la documentazione:**
   Quando aggiungi o modifichi file, endpoint o variabili d'ambiente, aggiorna `STRUCTURE.md` e `.env.example` contestualmente.

---

## ⚠️ KNOWN PITFALLS & RISOLUZIONE ERRORI

- **Errore `error from registry: denied` sul server remoto:**
  Causato da vecchie credenziali memorizzate in `~/.docker/config.json`.
  - Se il package è pubblico: esegui `docker logout ghcr.io && docker compose pull`.
  - Se il package è privato: riesegui `echo "ghp_TUO_TOKEN" | docker login ghcr.io -u USER -p --password-stdin`.
- **Git Bash su Windows riscrive i path:**
  Le variabili d'ambiente con percorsi tipo `/data` possono essere trasformate da Git Bash in `C:/Program Files/Git/data`. Anteponi `MSYS2_ENV_CONV_EXCL='*'` o usa file `.env`.
- **Windows file locking:**
  Su Windows i binari in esecuzione bloccano il file su disco: ferma il processo prima di ricompilare.
