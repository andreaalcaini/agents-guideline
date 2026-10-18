# Agent Guidelines: Docker Multi-Arch, GHCR & Cloudflare Mesh

Template universale di linee guida per agenti di codifica AI (**Claude Code**, **Antigravity**, **Cursor**, **Codex**) pronto da clonare o importare in qualsiasi nuovo progetto (Next.js, Vue, React, Vite, Python, Go, ecc.).

---

## 📦 Struttura dei File

| File | Scopo |
|---|---|
| [`CLAUDE.md`](CLAUDE.md) | Entrypoint principale per Claude Code. Importa `@AGENTS.md` e `@STRUCTURE.md` ed elenca le regole prioritarie immediate. |
| [`AGENTS.md`](AGENTS.md) | Regole operative e standard tecnici: Cloudflare Tunnel Mesh, Docker Multi-Arch veloce (`--platform=$BUILDPLATFORM`), GitHub Actions con GHCR, gestione token Classic PAT, workflow e risoluzione errori. |
| [`STRUCTURE.md`](STRUCTURE.md) | Mappa strutturale del repository: albero cartelle, container, topologia di rete Cloudflare, porte e variabili d'ambiente. |

---

## 🚀 Come usarlo in un nuovo progetto

Esegui questo comando dalla root del tuo nuovo progetto per scaricare istantaneamente i 3 file:

### Su Linux / macOS / Git Bash:
```bash
curl -fsSL --remote-name-all "https://raw.githubusercontent.com/<TUO-USERNAME>/agents-guideline/main/{CLAUDE,AGENTS,STRUCTURE}.md"
```

### Su Windows (PowerShell):
```powershell
@("CLAUDE.md", "AGENTS.md", "STRUCTURE.md") | ForEach-Object {
    Invoke-WebRequest -Uri "https://raw.githubusercontent.com/<TUO-USERNAME>/agents-guideline/main/$_" -OutFile $_
}
```

---

## ⚡ Caratteristiche Chiave Incluse

1. **Regola Aurea Cloudflare Tunnel Mesh:**
   Ogni servizio aggiunto in `docker-compose.yml` viene automaticamente configurato sulla rete `cloudflared-mesh` e l'agente chiede sempre la conferma dell'alias `<nomeservizio>.mesh`.
2. **Build Docker Multi-Arch Veloci (No Freeze QEMU):**
   Utilizza `--platform=$BUILDPLATFORM` per compilare a velocità nativa dell'host x86_64 producendo immagini compatibili sia con `linux/amd64` sia con `linux/arm64` (Raspberry Pi/Apple Silicon) in meno di 2 minuti.
3. **Pipeline CI/CD GHCR Garantita:**
   Include la pipeline GitHub Actions con gestione della cache e avverte sui limiti dei token Fine-Grained, guidando all'uso del Classic PAT (`write:packages`).
