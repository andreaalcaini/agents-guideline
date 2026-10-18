# CLAUDE.md

@AGENTS.md
@STRUCTURE.md

## 🚨 REGOLE PRIORITARIE PER CLAUDE

1. **Cloudflare Tunnel Mesh:**
   Ogni volta che crei o modifichi un `docker-compose.yml`, devi:
   - Inserire il collegamento alla rete esterna `cloudflared-mesh`.
   - Inserire l'alias `aliases: - <nomeservizio>.mesh`.
   - **Chiedere SEMPRE all'utente prima di finalizzare:** *"Quale nome desideri utilizzare per l'alias `<nomeservizio>.mesh` nella rete Cloudflare?"*

2. **Docker Multi-Arch Nativo:**
   Usa sempre `--platform=$BUILDPLATFORM` negli stage di build/installazione per evitare il freeze QEMU su ARM64 (Raspberry Pi/Mac).

3. **GitHub CI/CD & GHCR:**
   Solo Classic PAT (`ghp_...`) con scope `write:packages` e `repo`. Non usare mai Fine-Grained tokens (`github_pat_...`).
