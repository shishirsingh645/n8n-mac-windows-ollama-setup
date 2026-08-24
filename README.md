# n8n + Ollama LAN Architecture

## 1. Architecture

```text
                         SAME LOCAL NETWORK
                                │
                ┌───────────────┴────────────────┐
                │                                │
        ┌───────▼────────┐              ┌────────▼─────────┐
        │    MAC MINI    │              │  WINDOWS MACHINE  │
        │                │              │                   │
        │ Docker Desktop │              │ Ollama Native     │
        │       │        │              │       │           │
        │       ▼        │              │       ▼           │
        │     n8n         │   HTTP LAN   │   Qwen3 8B       │
        │    :5678        │─────────────►│   :11434         │
        │       │        │              │       │           │
        │       │        │              │       ▼           │
        │ HDD storage     │              │ NVIDIA GPU        │
        └────────────────┘              └───────────────────┘

Mac n8n URL:
http://MAC-IP:5678

Windows Ollama API:
http://WINDOWS-IP:11434
```

## 2. Roles

| Machine | Component | Purpose |
|---|---|---|
| Mac Mini M4 | Docker Desktop | Container runtime |
| Mac Mini M4 | n8n | Workflow automation |
| Mac Mini HDD | n8n data | Persistent n8n data |
| Windows | Ollama | LLM inference |
| Windows | Qwen3 8B | Lightweight LLM |
| Windows NVIDIA GPU | GPU acceleration | Model inference |
| LAN | HTTP | Mac ↔ Windows communication |

---

# PART A — MAC MINI

## 3. Verify Docker

```bash
docker --version
docker compose version
```

## 4. Create n8n directory

```bash
mkdir -p "/Volumes/Apple Storage HDD/Docker/n8n"
cd "/Volumes/Apple Storage HDD/Docker/n8n"
```

## 5. Create persistent data directory

```bash
mkdir -p "/Volumes/Apple Storage HDD/Docker/n8n/data"
```

## 6. Create Docker Compose file

```bash
nano docker-compose.yml
```

Paste:

```yaml
services:
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped

    ports:
      - "5678:5678"

    environment:
      - TZ=Asia/Kolkata
      - GENERIC_TIMEZONE=Asia/Kolkata
      - N8N_SECURE_COOKIE=false

    volumes:
      - "/Volumes/Apple Storage HDD/Docker/n8n/data:/home/node/.n8n"
```

Save:

```text
CTRL + O
ENTER
CTRL + X
```

## 7. Validate Docker Compose

```bash
cd "/Volumes/Apple Storage HDD/Docker/n8n"
docker compose config
```

## 8. Pull n8n image

```bash
docker compose pull
```

## 9. Start n8n

```bash
docker compose up -d
```

## 10. Check container

```bash
docker ps
```

Expected port mapping:

```text
0.0.0.0:5678->5678/tcp
```

## 11. Check n8n logs

```bash
docker logs -f n8n
```

Stop log output:

```text
CTRL + C
```

## 12. Test n8n locally on Mac

Open:

```text
http://localhost:5678
```

## 13. Find Mac LAN IP

```bash
ipconfig getifaddr en0
```

Example:

```text
192.168.1.20
```

If Wi-Fi uses another interface, check:

```bash
ifconfig
```

## 14. Test n8n from Windows

From the Windows browser:

```text
http://MAC-IP:5678
```

Example:

```text
http://192.168.1.20:5678
```

## 15. Verify n8n data is stored on HDD

```bash
ls -lah "/Volumes/Apple Storage HDD/Docker/n8n/data"
```

## 16. Verify Docker mount

```bash
docker inspect n8n --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Expected:

```text
/Volumes/Apple Storage HDD/Docker/n8n/data -> /home/node/.n8n
```

---

# PART B — WINDOWS MACHINE

## 17. Install Ollama

Install Ollama for Windows.

After installation, open PowerShell.

## 18. Verify Ollama

```powershell
ollama --version
```

## 19. Check NVIDIA GPU

```powershell
nvidia-smi
```

Verify that the RTX GPU is detected.

## 20. Pull the initial model

Use only one model initially:

```powershell
ollama pull qwen3:8b
```

## 21. Test the model locally

```powershell
ollama run qwen3:8b
```

Test a prompt.

Exit:

```text
/bye
```

## 22. Check installed models

```powershell
ollama list
```

Expected:

```text
qwen3:8b
```

## 23. Check Ollama locally

```powershell
curl http://localhost:11434/api/tags
```

The response should contain `qwen3:8b`.

---

# PART C — WINDOWS OLLAMA LAN ACCESS

## 24. Configure Ollama to listen on LAN

Set the Windows environment variable:

```powershell
[Environment]::SetEnvironmentVariable("OLLAMA_HOST", "0.0.0.0:11434", "User")
```

## 25. Restart Ollama

Close Ollama completely and start it again.

If Ollama is running as a tray application, exit it and reopen it.

## 26. Check Windows IP

```powershell
ipconfig
```

Find the active adapter's IPv4 address.

Example:

```text
192.168.1.50
```

## 27. Allow Ollama through Windows Firewall

Open PowerShell as Administrator:

```powershell
New-NetFirewallRule -DisplayName "Ollama LAN 11434" -Direction Inbound -Protocol TCP -LocalPort 11434 -Action Allow -Profile Private
```

## 28. Verify Ollama is listening

```powershell
netstat -ano | findstr :11434
```

Expected:

```text
0.0.0.0:11434
```

## 29. Test Ollama from Windows itself

```powershell
curl http://WINDOWS-IP:11434/api/tags
```

Example:

```powershell
curl http://192.168.1.50:11434/api/tags
```

---

# PART D — TEST MAC → WINDOWS OLLAMA

## 30. Test from Mac

On the Mac:

```bash
curl http://WINDOWS-IP:11434/api/tags
```

Example:

```bash
curl http://192.168.1.50:11434/api/tags
```

Expected result:

```text
JSON response containing qwen3:8b
```

## 31. Test Ollama generation from Mac

```bash
curl http://WINDOWS-IP:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3:8b",
    "prompt": "Say hello in one sentence.",
    "stream": false
  }'
```

Expected:

```text
JSON response containing the generated answer.
```

---

# PART E — CONNECT n8n TO WINDOWS OLLAMA

## 32. Open n8n

From Windows:

```text
http://MAC-IP:5678
```

Example:

```text
http://192.168.1.20:5678
```

## 33. Create Ollama credential in n8n

Use the Ollama integration/node.

Set the Ollama Base URL to:

```text
http://WINDOWS-IP:11434
```

Example:

```text
http://192.168.1.50:11434
```

## 34. Select model

```text
qwen3:8b
```

## 35. Test n8n → Ollama

Create a simple workflow:

```text
Manual Trigger
      ↓
Ollama Chat Model
      ↓
Output
```

Run the workflow.

## 36. Final communication path

```text
Windows Browser
      │
      │ HTTP
      ▼
Mac Mini
      │
      │ :5678
      ▼
Docker
      │
      ▼
n8n
      │
      │ HTTP LAN
      │ :11434
      ▼
Windows
      │
      ▼
Ollama
      │
      ▼
Qwen3 8B
      │
      ▼
NVIDIA GPU
```

---

# PART F — DAILY MANAGEMENT

## 37. Start n8n

### Mac

```bash
cd "/Volumes/Apple Storage HDD/Docker/n8n"
docker compose up -d
```

### Windows

Start Ollama normally.

---

## 38. Stop n8n

### Mac

```bash
cd "/Volumes/Apple Storage HDD/Docker/n8n"
docker compose down
```

### Windows

Exit Ollama if required.

---

## 39. Restart n8n

### Mac

```bash
cd "/Volumes/Apple Storage HDD/Docker/n8n"
docker compose restart
```

### Windows

Restart Ollama.

---

## 40. Check n8n status

### Mac

```bash
docker ps
```

## 41. Check n8n logs

### Mac

```bash
docker logs -f n8n
```

## 42. Check Ollama models

### Windows

```powershell
ollama list
```

## 43. Check Ollama GPU usage

### Windows

```powershell
nvidia-smi
```

## 44. Test Mac → Windows connectivity

### Mac

```bash
curl http://WINDOWS-IP:11434/api/tags
```

## 45. Test Windows → Mac connectivity

### Windows

```powershell
curl http://MAC-IP:5678
```

---

# 46. Final Directory Structure

```text
/Volumes/Apple Storage HDD/Docker/n8n/
│
├── docker-compose.yml
│
└── data/
    ├── database.sqlite
    ├── config
    ├── binaryData/
    └── other n8n data
```

# 47. Final IP/Port Configuration

| Service | Machine | Port | URL |
|---|---|---:|---|
| n8n | Mac Mini | 5678 | `http://MAC-IP:5678` |
| Ollama | Windows | 11434 | `http://WINDOWS-IP:11434` |
| Qwen3 8B | Windows | — | `qwen3:8b` |

# 48. Final Recommended Setup

```text
MAC MINI M4
└── Docker
    └── n8n
        └── /Volumes/Apple Storage HDD/Docker/n8n/data


WINDOWS
└── Ollama
    └── qwen3:8b
        └── NVIDIA GPU


LAN
└── n8n → http://WINDOWS-IP:11434 → Ollama
```
