# Master Linux Admin Control UI — Zorin 18 Setup

> **Hardware:** Laptop · i7-13620H · RTX 5060 8GB · 16GB RAM  
> **OS:** Zorin OS 18 · GNOME · Wayland  
> **Modell:** Gemma-4-12B Q6 via Ollama · MiniMax API als Fallback  
> **Ziel:** Vollständiges lokales Admin+AI-Dashboard mit linux-assistant (Systemkontrolle) + Odysseus (AI-Workspace)

---

## Architektur-Übersicht

```
┌─────────────────────────────────────────────────────┐
│                    Zorin OS 18 Host                  │
│                                                     │
│  ┌──────────────────┐    ┌────────────────────────┐ │
│  │  linux-assistant │    │  Ollama (systemd)       │ │
│  │  (nativ .deb)    │    │  Port: 11434            │ │
│  │  GNOME Wayland   │    │  Gemma-4-12B Q6 (GPU)  │ │
│  │  polkit / apt    │    │  all-minilm:l6-v2 (CPU)│ │
│  └──────────────────┘    └────────────┬───────────┘ │
│                                       │ host.docker.internal
│  ┌────────────────────────────────────▼───────────┐ │
│  │              Docker Compose                     │ │
│  │                                                 │ │
│  │  odysseus :7000  ──►  searxng  :8080            │ │
│  │                  ──►  chromadb :8100            │ │
│  │                  ──►  ntfy     :8091            │ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

| Komponente | Deployment | Port | Zweck |
|---|---|---|---|
| linux-assistant | Nativ (.deb) | — | Systemkontrolle, Routinen, GNOME-Shortcuts |
| Ollama | Nativ (systemd) | 11434 | Gemma-4-12B Q6 lokal, Embeddings |
| Odysseus | Docker Compose | 7000 | AI-Chat, Agents, Research, Docs, Email, Notes |
| SearXNG | Docker (via Odysseus) | 8080 | Web-Search für Deep Research |
| ChromaDB | Docker (via Odysseus) | 8100 | RAG / Vector Store |
| ntfy | Docker (via Odysseus) | 8091 | Push-Notifications für Agent-Tasks |

---

## PHASE 0 — Voraussetzungen prüfen

```bash
# NVIDIA-Treiber prüfen (muss 570+ sein für Blackwell/RTX 5060)
nvidia-smi

# Falls kein Output oder Treiber veraltet:
sudo ubuntu-drivers install nvidia:570
sudo reboot
# Danach nochmals: nvidia-smi
```

```bash
# Ollama prüfen
ollama --version
systemctl status ollama

# Falls nicht installiert:
curl -fsSL https://ollama.com/install.sh | sh
```

---

## PHASE 1 — Docker (offiziell, nicht snap)

```bash
# Schritt 1: Alte Docker-Versionen entfernen
sudo apt remove -y docker docker.io docker-doc docker-compose \
  podman-docker containerd runc 2>/dev/null || true

# Schritt 2: Abhängigkeiten
sudo apt update && sudo apt install -y ca-certificates curl gnupg

# Schritt 3: Docker GPG-Key
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Schritt 4: Docker Repository (Ubuntu 24.04 noble = Zorin 18 Base)
echo "deb [arch=$(dpkg --print-architecture) \
  signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu noble stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list

# Schritt 5: Installieren
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin

# Schritt 6: User zur Docker-Gruppe (kein sudo mehr nötig)
sudo usermod -aG docker $USER
newgrp docker

# Schritt 7: Test
docker run hello-world
```

---

## PHASE 2 — NVIDIA Container Toolkit

```bash
# GPG-Key
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | \
  sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

# Repository
curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
  sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
  sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# Installieren
sudo apt update && sudo apt install -y nvidia-container-toolkit

# Docker Runtime konfigurieren
sudo nvidia-ctk runtime configure --runtime=docker

# Überprüfen (nvidia muss drin stehen)
cat /etc/docker/daemon.json

# Docker neu starten
sudo systemctl daemon-reload && sudo systemctl restart docker

# VERIFY — GPU in Container sichtbar?
docker run --rm --gpus all \
  nvidia/cuda:12.6.0-base-ubuntu24.04 nvidia-smi
# ✅ Muss deine RTX 5060 mit 8GB zeigen
```

---

## PHASE 3 — Ollama für Docker konfigurieren

```bash
# systemd Override erstellen
sudo systemctl edit ollama
```

Einfügen zwischen die Kommentarzeilen:

```ini
[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"
Environment="OLLAMA_NUM_PARALLEL=1"
Environment="OLLAMA_MAX_LOADED_MODELS=1"
```

```bash
sudo systemctl daemon-reload && sudo systemctl restart ollama

# Embedding-Modell laden (läuft auf CPU, kein VRAM-Konflikt)
ollama pull all-minilm:l6-v2

# Test: von Docker aus erreichbar?
curl http://localhost:11434/api/tags
# ✅ Muss JSON mit verfügbaren Modellen zurückgeben
```

> **Hinweis VRAM:** Gemma-4-12B Q6 ≈ 9.8 GB benötigt — mit 8 GB VRAM macht Ollama  
> automatisch GPU+CPU-Splitting. Das ist normal und erklärt die etwas langsamere Geschwindigkeit.  
> `OLLAMA_MAX_LOADED_MODELS=1` verhindert OOM wenn mehrere Modelle gleichzeitig  
> angefordert werden.

---

## PHASE 4 — Odysseus aufsetzen

```bash
# Repo klonen (dev = neuester Stand)
git clone https://github.com/Toqsick/odysseus.git
cd odysseus

# .env erstellen
cp .env.example .env
```

### .env Konfiguration (komplett für dein Setup)

```bash
# Docker GID ermitteln
getent group docker | cut -d: -f3
# Diesen Wert unten als DOCKER_GID eintragen
```

`.env` bearbeiten:

```env
# ── GPU ─────────────────────────────────────────────
COMPOSE_FILE=docker-compose.yml:docker/gpu.nvidia.yml
DOCKER_GID=999  # ← Ersetzen mit Ausgabe von: getent group docker | cut -d: -f3

# ── Ollama (Host → Docker) ───────────────────────────
OLLAMA_BASE_URL=http://host.docker.internal:11434/v1

# ── Embeddings via Ollama (CPU, kein VRAM-Verbrauch) ─
EMBEDDING_URL=http://host.docker.internal:11434/v1/embeddings
EMBEDDING_MODEL=all-minilm:l6-v2

# ── MiniMax API (OpenAI-kompatibel) ──────────────────
# In Odysseus UI unter Settings → Providers eintragen:
# Base URL: https://api.minimax.chat/v1
# API Key: dein MiniMax Key
OPENAI_API_KEY=dein_minimax_key_hier

# ── Auth & Security ──────────────────────────────────
AUTH_ENABLED=true
APP_BIND=127.0.0.1
APP_PORT=7000
ODYSSEUS_ADMIN_PASSWORD=sicheres_passwort_hier_setzen

# ── Upload-Größen (für lokale Nutzung erhöht) ────────
ODYSSEUS_CHAT_UPLOAD_MAX_BYTES=52428800
ODYSSEUS_PERSONAL_UPLOAD_MAX_BYTES=52428800

# ── User IDs (Standard Zorin OS) ─────────────────────
PUID=1000
PGID=1000
```

```bash
# Starten
docker compose up -d --build

# Logs verfolgen (erster Build dauert 3-5 Min)
docker compose logs -f odysseus

# ✅ Warten bis: "Odysseus is running on http://0.0.0.0:7000"
```

Öffnen: **http://localhost:7000**  
Admin-Passwort steht in der `.env` (ODYSSEUS_ADMIN_PASSWORD).

### MiniMax als Provider in Odysseus eintragen

1. Settings → Providers → "Add Provider"
2. Type: `OpenAI Compatible`
3. Base URL: `https://api.minimax.chat/v1`
4. API Key: dein MiniMax Key
5. Model Name: `MiniMax-Text-01` oder `abab6.5s-chat`

---

## PHASE 5 — linux-assistant (Systemkontrolle)

```bash
# Abhängigkeiten
sudo apt install -y libkeybinder-3.0-0 libkeybinder-3.0-dev wmctrl

# Snap für Flutter
sudo rm -f /etc/apt/preferences.d/nosnap.pref  # Falls Linux Mint Reste
sudo apt install -y snapd git
sudo snap install flutter --classic

# Repo (dein Fork)
git clone https://github.com/Toqsick/linux-assistant.git
cd linux-assistant

# .deb bauen
bash ./build-deb.sh

# Installieren
bash install.sh
# oder: sudo apt install ./linux-assistant_*_amd64.deb
```

> **Wayland-Hinweis:** `wmctrl` hat unter purem Wayland eingeschränkte Funktionen.  
> Zorin 18 hat XWayland standardmäßig aktiv — die meisten Features funktionieren.  
> Falls Fenster-Fokus-Probleme auftreten: `sudo apt install ydotool` als Ersatz.

Keyboard-Shortcut entfernen/ändern:  
*Einstellungen → Tastatur → Eigene Tastenkombinationen*

---

## PHASE 6 — Verify & First-Run Checklist

```bash
# Alle Container laufen?
docker compose ps
# Erwartet: odysseus, searxng, chromadb, ntfy → alle "running"

# Ollama erreichbar vom Container?
docker exec $(docker compose ps -q odysseus) \
  curl -s http://host.docker.internal:11434/api/tags | python3 -m json.tool

# GPU im Container aktiv?
docker exec $(docker compose ps -q odysseus) nvidia-smi
```

### In der Odysseus UI testen:

- [ ] Chat mit Gemma-4-12B (Ollama) → Antwort kommt
- [ ] Chat mit MiniMax → schnelle Antwort
- [ ] Deep Research → SearXNG-Suche funktioniert
- [ ] Documents → Datei hochladen und AI-Edit
- [ ] Memory/RAG → ChromaDB-Verbindung grün
- [ ] linux-assistant → Shortcut funktioniert, Systeminfo sichtbar

---

## Nützliche Befehle

```bash
# Odysseus stoppen/starten
cd ~/odysseus
docker compose down
docker compose up -d

# Logs live
docker compose logs -f

# Modell in Ollama wechseln
ollama list
ollama run gemma4:12b-instruct-q6_K

# linux-assistant deinstallieren (sauber)
sudo apt remove linux-assistant
rm -rf ~/.config/linux-assistant ~/.cache/linux-assistant
# Shortcut manuell unter Einstellungen → Tastatur entfernen

# Odysseus Update
cd ~/odysseus
git pull origin dev
docker compose up -d --build
```

---

## Bekannte Einschränkungen

| Problem | Ursache | Workaround |
|---|---|---|
| Gemma-4 langsam (~5 tok/s) | 9.8GB Modell auf 8GB VRAM → GPU+CPU Split | MiniMax für schnelle Tasks nutzen |
| wmctrl unter Wayland | XWayland nötig | Ist auf Zorin 18 standard aktiv |
| Odysseus dev-Branch instabil | Aktive Entwicklung | Nach Update: `docker compose logs` prüfen |
| 16GB RAM eng mit Docker | SearXNG+ChromaDB+Odysseus ≈ 2-3GB | `--memory=2g` Limit in compose bei Bedarf |

---

## Ressourcen

- [Odysseus Repo (Toqsick Fork)](https://github.com/Toqsick/odysseus)
- [linux-assistant Repo (Toqsick Fork)](https://github.com/Toqsick/linux-assistant)
- [Odysseus Roadmap](https://github.com/Toqsick/odysseus/blob/dev/ROADMAP.md)
- [Ollama Dokumentation](https://ollama.com/docs)
- [NVIDIA Container Toolkit Docs](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
