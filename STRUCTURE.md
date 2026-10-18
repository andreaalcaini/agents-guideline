# Project Structure

Descrive l'architettura dei file, la topologia di rete, i container e le variabili d'ambiente del progetto.
Mantenere aggiornato questo file ad ogni modifica strutturale o aggiunta di endpoint.

---

## 🗂️ Albero del Repository

```text
repo/
├── .github/
│   └── workflows/
│       └── deploy.yml        # CI/CD: build multi-arch (amd64/arm64) e push su GHCR
├── src/                      # Codice sorgente applicativo
├── data/                     # Dati persistenti (DB / JSON) mappati nel volume Docker
├── public/                   # Asset statici
├── .dockerignore             # File esclusi dal build context Docker
├── docker-compose.yml        # Configurazione container di produzione e rete mesh
├── Dockerfile                # Build multi-stage ottimizzata con BUILDPLATFORM
└── .env.example              # Template variabili d'ambiente documentate
```

---

## 🐳 Docker & Topologia di Rete

### Container di Produzione:
- **Immagine:** `ghcr.io/<username>/<repo>:latest`
- **Porte esposte:** `3000:3000` (o porta specifica del servizio)
- **Volume persistente:** `./data:/app/data` (salvataggio dati duraturo)
- **Architetture supportate:** `linux/amd64` (Server Intel/AMD) e `linux/arm64` (Raspberry Pi, Mac)

### Rete Cloudflare Tunnel (`cloudflared-mesh`):
Il container è connesso alla rete bridge esterna `cloudflared-mesh` creata da Cloudflared sul server host.
- **Rete esterna:** `cloudflared-mesh`
- **Alias DNS interno:** `<nomeservizio>.mesh` (confermato dall'utente)
- **Instradamento:** Cloudflared tunnel instrada il traffico HTTPS pubblico verso `http://<nomeservizio>.mesh:<porta>`

---

## ⚙️ Variabili d'Ambiente

| Variabile | Default | Descrizione |
|---|---|---|
| `PORT` | `3000` | Porta interna su cui ascolta il server |
| `NODE_ENV` | `production` | Ambiente di esecuzione |
