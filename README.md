# n8n + Ollama LAN Architecture

A complete deployment guide for running **n8n in Docker on a Mac Mini M4**, storing n8n data on an external HDD, and using a **Windows machine with an NVIDIA GPU to run Ollama and Qwen3 8B** for LLM inference over the local LAN.

This setup separates workflow automation from LLM inference:

* **Mac Mini M4:** Docker Desktop + n8n + persistent n8n data
* **Windows:** Ollama + Qwen3 8B + NVIDIA GPU
* **LAN:** Communication between n8n and Ollama
* **External HDD:** Persistent n8n application data

---

# 1. Architecture Overview

```text
                              PRIVATE LAN
                       SAME LOCAL NETWORK
                              │
             ┌────────────────┴─────────────────┐
             │                                  │
             │                                  │
     ┌───────▼───────────┐              ┌───────▼───────────┐
     │    MAC MINI M4    │              │ WINDOWS MACHINE   │
     │                   │              │                   │
     │  macOS            │              │ Windows           │
     │       │           │              │       │           │
     │       ▼           │              │       ▼           │
     │ Docker Desktop    │              │     Ollama        │
     │       │           │              │       │           │
     │       ▼           │              │       ▼           │
     │      n8n           │   HTTP LAN   │   Qwen3 8B        │
     │     :5678          │─────────────►│   :11434          │
     │       │            │              │       │           │
     │       ▼            │              │       ▼           │
     │ External HDD       │              │ NVIDIA GPU        │
     │ /Docker/n8n/data   │              │ RTX GPU           │
     └───────────────────┘              └───────────────────┘
```

## Communication Flow

```text
User
 │
 │ Browser
 ▼
http://192.xx.xx.xx:5678
 │
 ▼
n8n
 │
 │ HTTP Request / Ollama Node
 │
 │ http://192.xx.xx.xx:11434
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
 │
 ▼
Generated response
 │
 ▼
Ollama API
 │
 ▼
n8n
 │
 ▼
Workflow output
```

---

# 2. Machine Responsibilities

| Machine            | Component        | Responsibility                 |
| ------------------ | ---------------- | ------------------------------ |
| Mac Mini M4        | macOS            | Main automation host           |
| Mac Mini M4        | Docker Desktop   | Container runtime              |
| Mac Mini M4        | n8n              | Workflow automation            |
| External HDD       | n8n data         | Persistent application storage |
| Windows            | Ollama           | Local LLM runtime              |
| Windows            | Qwen3 8B         | Language model                 |
| Windows NVIDIA GPU | GPU acceleration | Model inference                |
| LAN                | TCP/HTTP         | Mac ↔ Windows communication    |

The main architectural principle is:

```text
n8n = orchestration
Ollama = inference
Qwen3 = model
GPU = computation
LAN = communication
```

---

# 3. Prerequisites

## Mac Mini

Required:

* macOS
* Docker Desktop
* Docker Compose
* External HDD mounted
* Mac and Windows connected to the same LAN
* Port `5678` available

Verify Docker:

```bash
docker --version
docker compose version
```

## Windows

Required:

* Windows 10/11
* Ollama
* NVIDIA GPU
* NVIDIA driver
* PowerShell
* Port `11434` available
* Mac and Windows connected to the same LAN

Verify NVIDIA:

```powershell
nvidia-smi
```

Verify Ollama:

```powershell
ollama --version
```

---

# 4. Network Design

Both machines must be reachable from each other.

Example:

```text
Router
 │
 ├── Mac Mini
 │    IP: 192.xx.xx.xx
 │
 └── Windows
      IP: 192.xx.xx.xx
```

Services:

```text
Mac n8n
192.xx.xx.xx:5678

Windows Ollama
192.xx.xx.xx:11434
```

The traffic path is:

```text
192.xx.xx.xx:5678
        │
        │ HTTP
        ▼
192.xx.xx.xx:11434
```

## Important Network Requirement

The Mac and Windows systems should be on the same trusted/private network.

Avoid exposing Ollama directly to the public internet.

Do not configure:

```text
0.0.0.0:11434
```

on an internet-facing interface without additional network controls.

---

# PART A — MAC MINI

# 5. Create n8n Directory

```bash
mkdir -p "/Volumes/Apple Storage HDD/Docker/n8n"
cd "/Volumes/Apple Storage HDD/Docker/n8n"
```

Verify:

```bash
pwd
```

Expected:

```text
/Volumes/Apple Storage HDD/Docker/n8n
```

---

# 6. Create Persistent Storage

```bash
mkdir -p "/Volumes/Apple Storage HDD/Docker/n8n/data"
```

Verify:

```bash
ls -lah "/Volumes/Apple Storage HDD/Docker/n8n"
```

Expected structure:

```text
n8n/
├── data/
└── docker-compose.yml
```

---

# 7. Why Persistent Storage Is Required

The n8n container itself is disposable.

Without persistent storage, n8n's application data could be lost if the container is removed.

With the volume:

```text
Mac HDD
   │
   ▼
/Volumes/Apple Storage HDD/Docker/n8n/data
   │
   │ Docker mount
   ▼
/home/node/.n8n
   │
   ▼
n8n container
```

Docker stores n8n's persistent data outside the container.

This allows:

```bash
docker compose down
docker compose up -d
```

without intentionally removing the persistent application data.

---

# 8. Create Docker Compose Configuration

```bash
nano docker-compose.yml
```

Use:

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

---

# 9. Explanation of Docker Compose

## Image

```yaml
image: docker.n8n.io/n8nio/n8n:latest
```

Downloads the official n8n container image.

## Container Name

```yaml
container_name: n8n
```

Allows commands such as:

```bash
docker logs n8n
docker restart n8n
docker inspect n8n
```

## Restart Policy

```yaml
restart: unless-stopped
```

Docker automatically restarts n8n after:

* container crashes
* Docker restart
* Mac restart

unless the container was intentionally stopped.

## Port Mapping

```yaml
ports:
  - "5678:5678"
```

The mapping means:

```text
Mac port 5678
      │
      ▼
Container port 5678
```

Therefore:

```text
http://localhost:5678
```

accesses n8n locally.

The Mac LAN address becomes:

```text
http://192.xx.xx.xx:5678
```

## Volume

```yaml
volumes:
  - "/Volumes/Apple Storage HDD/Docker/n8n/data:/home/node/.n8n"
```

Maps the external HDD storage to n8n's persistent application directory.

---

# 10. Validate Docker Compose

```bash
cd "/Volumes/Apple Storage HDD/Docker/n8n"
docker compose config
```

Also check:

```bash
docker compose config --services
```

Expected:

```text
n8n
```

---

# 11. Pull n8n Image

```bash
docker compose pull
```

Check downloaded images:

```bash
docker images
```

---

# 12. Start n8n

```bash
docker compose up -d
```

Verify:

```bash
docker ps
```

Expected port mapping:

```text
0.0.0.0:5678->5678/tcp
```

---

# 13. Check Container Health

```bash
docker ps
```

Detailed information:

```bash
docker inspect n8n
```

Check restart configuration:

```bash
docker inspect n8n --format '{{.HostConfig.RestartPolicy.Name}}'
```

Expected:

```text
unless-stopped
```

---

# 14. Check n8n Logs

```bash
docker logs n8n
```

For live logs:

```bash
docker logs -f n8n
```

Stop viewing logs:

```text
CTRL + C
```

---

# 15. Test n8n Locally

Open:

```text
http://localhost:5678
```

Confirm that the n8n interface loads.

---

# 16. Find Mac LAN IP

For Wi-Fi:

```bash
ipconfig getifaddr en0
```

Example output format:

```text
192.xx.xx.xx
```

Check all interfaces:

```bash
ifconfig
```

---

# 17. Test Windows → Mac

On Windows:

```powershell
curl http://192.xx.xx.xx:5678
```

You can also test from the browser:

```text
http://192.xx.xx.xx:5678
```

If this fails, investigate:

```text
Mac IP
   ↓
LAN connectivity
   ↓
Docker port 5678
   ↓
Mac firewall
   ↓
n8n container
```

---

# 18. Verify HDD Storage

```bash
ls -lah "/Volumes/Apple Storage HDD/Docker/n8n/data"
```

After n8n starts and is configured, the directory should contain n8n application data.

---

# 19. Verify Docker Mount

```bash
docker inspect n8n --format '{{range .Mounts}}{{println .Source "->" .Destination}}{{end}}'
```

Expected:

```text
/Volumes/Apple Storage HDD/Docker/n8n/data -> /home/node/.n8n
```

---

# PART B — WINDOWS MACHINE

# 20. Install Ollama

Install Ollama for Windows.

After installation:

```powershell
ollama --version
```

---

# 21. Verify NVIDIA GPU

```powershell
nvidia-smi
```

Verify:

* GPU name
* VRAM
* driver version
* GPU utilization

---

# 22. Pull Qwen3 8B

Start with one model:

```powershell
ollama pull qwen3:8b
```

---

# 23. Verify Model

```powershell
ollama list
```

Expected:

```text
qwen3:8b
```

---

# 24. Run Qwen3 Locally

```powershell
ollama run qwen3:8b
```

Test:

```text
Explain Docker in one sentence.
```

Exit:

```text
/bye
```

---

# 25. Test Ollama REST API

```powershell
curl http://localhost:11434/api/tags
```

The response should contain the installed model.

---

# PART C — OLLAMA LAN CONFIGURATION

# 26. Why LAN Access Is Required

n8n is running on:

```text
Mac
```

Ollama is running on:

```text
Windows
```

Therefore Windows Ollama must accept requests from the Mac.

Desired communication:

```text
Mac
 │
 │ TCP
 ▼
192.xx.xx.xx:11434
 │
 ▼
Ollama
```

---

# 27. Configure OLLAMA_HOST

Run:

```powershell
[Environment]::SetEnvironmentVariable(
    "OLLAMA_HOST",
    "0.0.0.0:11434",
    "User"
)
```

Verify in a new PowerShell session:

```powershell
$env:OLLAMA_HOST
```

---

# 28. Restart Ollama

Completely exit Ollama.

If Ollama is visible in the Windows system tray:

```text
Ollama Tray Icon
        ↓
Exit
```

Start Ollama again.

---

# 29. Find Windows IP

```powershell
ipconfig
```

Locate the active adapter.

Example format:

```text
IPv4 Address:
192.xx.xx.xx
```

Use the address belonging to the same network as the Mac.

---

# 30. Configure Windows Firewall

Open PowerShell as Administrator:

```powershell
New-NetFirewallRule `
    -DisplayName "Ollama LAN 11434" `
    -Direction Inbound `
    -Protocol TCP `
    -LocalPort 11434 `
    -Action Allow `
    -Profile Private
```

---

# 31. Verify Firewall Rule

```powershell
Get-NetFirewallRule -DisplayName "Ollama LAN 11434"
```

---

# 32. Verify Ollama Listening Address

```powershell
netstat -ano | findstr :11434
```

Desired result should show Ollama listening on a suitable interface, such as:

```text
0.0.0.0:11434
```

If it only listens on:

```text
127.0.0.1:11434
```

the Mac will not be able to connect.

---

# 33. Test Ollama Through Windows LAN Address

```powershell
curl http://192.xx.xx.xx:11434/api/tags
```

---

# PART D — MAC → WINDOWS TESTING

# 34. Basic Connectivity Test

From Mac:

```bash
ping 192.xx.xx.xx
```

Stop:

```text
CTRL + C
```

---

# 35. Test Port 11434

From Mac:

```bash
nc -vz 192.xx.xx.xx 11434
```

A successful connection indicates that TCP connectivity exists.

---

# 36. Test Ollama API

From Mac:

```bash
curl http://192.xx.xx.xx:11434/api/tags
```

Expected:

```text
JSON
```

containing installed models.

---

# 37. Test Model Generation

```bash
curl http://192.xx.xx.xx:11434/api/generate \
  -H "Content-Type: application/json" \
  -d '{
    "model": "qwen3:8b",
    "prompt": "Say hello in one sentence.",
    "stream": false
  }'
```

Successful output confirms:

```text
Mac
 ↓
LAN
 ↓
Windows
 ↓
Ollama
 ↓
Qwen3
 ↓
GPU inference
 ↓
Response
```

---

# PART E — n8n + OLLAMA

# 38. Open n8n

From Windows browser:

```text
http://192.xx.xx.xx:5678
```

---

# 39. Configure Ollama in n8n

Use the Ollama integration or Ollama Chat Model node.

Configure the Ollama Base URL as:

```text
http://192.xx.xx.xx:11434
```

Do not use:

```text
http://localhost:11434
```

inside n8n for this architecture.

Why?

Because `localhost` from inside the n8n container refers to the container itself, not the Windows machine.

Correct:

```text
n8n container
     │
     │ LAN
     ▼
192.xx.xx.xx:11434
```

---

# 40. Select Qwen3

Use:

```text
qwen3:8b
```

The model name in n8n must match the model installed in Ollama.

Verify:

```powershell
ollama list
```

---

# 41. Recommended Test Workflow

```text
Manual Trigger
       │
       ▼
Ollama Chat Model
       │
       ▼
AI Agent / Basic Prompt
       │
       ▼
Output
```

Example prompt:

```text
Explain what Docker is in three sentences.
```

Execute the workflow.

---

# 42. Testing Layers

Test in this order:

### Layer 1 — Ollama

```powershell
curl http://localhost:11434/api/tags
```

### Layer 2 — LAN

```bash
curl http://192.xx.xx.xx:11434/api/tags
```

### Layer 3 — n8n

```text
http://192.xx.xx.xx:5678
```

### Layer 4 — n8n → Ollama

Execute the Ollama workflow.

---

# PART F — TROUBLESHOOTING

# 43. n8n Does Not Open

Check:

```bash
docker ps
docker logs n8n
docker port n8n
```

Expected:

```text
5678/tcp -> 0.0.0.0:5678
```

---

# 44. Windows Cannot Reach n8n

Test:

```powershell
ping 192.xx.xx.xx
```

Then:

```powershell
curl http://192.xx.xx.xx:5678
```

Check on Mac:

```bash
docker ps
```

---

# 45. Mac Cannot Reach Ollama

Test:

```bash
ping 192.xx.xx.xx
```

Then:

```bash
nc -vz 192.xx.xx.xx 11434
```

Then:

```bash
curl http://192.xx.xx.xx:11434/api/tags
```

Check Windows:

```powershell
netstat -ano | findstr :11434
```

---

# 46. Ollama Only Shows 127.0.0.1

If:

```text
127.0.0.1:11434
```

is the only listening address, remote clients cannot access it.

Check:

```powershell
$env:OLLAMA_HOST
```

Restart Ollama.

Verify again:

```powershell
netstat -ano | findstr :11434
```

---

# 47. n8n Cannot Find Model

Verify:

```powershell
ollama list
```

Expected:

```text
qwen3:8b
```

Verify n8n Base URL:

```text
http://192.xx.xx.xx:11434
```

Then:

```bash
curl http://192.xx.xx.xx:11434/api/tags
```

---

# 48. GPU Is Not Being Used

Run:

```powershell
nvidia-smi
```

Run the model:

```powershell
ollama run qwen3:8b
```

In another PowerShell window:

```powershell
nvidia-smi
```

Observe GPU utilization and memory usage while the model is generating.

---

# 49. HDD Is Not Mounted on Mac

Check:

```bash
ls "/Volumes/Apple Storage HDD"
```

If the directory does not exist, the external HDD may not currently be mounted.

The Docker volume points to:

```text
/Volumes/Apple Storage HDD/Docker/n8n/data
```

Therefore the drive should be available before starting n8n.

---

# 50. Docker Container Restart

Restart only n8n:

```bash
docker restart n8n
```

Restart through Compose:

```bash
cd "/Volumes/Apple Storage HDD/Docker/n8n"
docker compose restart
```

Stop and recreate:

```bash
docker compose down
docker compose up -d
```

Do not use:

```bash
docker compose down -v
```

unless you specifically intend to remove Docker-managed volumes.

---

# PART G — SECURITY CONSIDERATIONS

# 51. Keep Ollama on the Private LAN

The intended architecture is:

```text
Private LAN
   │
   ├── Mac
   │
   └── Windows
```

Avoid exposing port `11434` to the internet through:

* router port forwarding
* public reverse proxy
* unrestricted cloud firewall rules

Ollama is intended here as an internal inference service.

---

# 52. Restrict Windows Firewall

For a controlled environment, restrict the firewall rule to the private network profile.

You can further restrict the source IP to the Mac if required.

Example:

```text
Mac
192.xx.xx.xx
      │
      ▼
Windows
192.xx.xx.xx:11434
```

This provides tighter access control than allowing every device on the LAN.

---

# 53. Static or Reserved IP Addresses

Because n8n depends on the Windows Ollama endpoint, changing the Windows IP can break the connection.

Recommended:

```text
Mac       → Reserved IP
Windows   → Reserved IP
```

Example:

```text
Mac       192.xx.xx.xx
Windows   192.xx.xx.xx
```

Prefer DHCP reservations on the router rather than manually configuring static IPs unless there is a specific networking requirement.

---

# PART H — DAILY OPERATIONS

# 54. Start n8n

```bash
cd "/Volumes/Apple Storage HDD/Docker/n8n"
docker compose up -d
```

---

# 55. Stop n8n

```bash
cd "/Volumes/Apple Storage HDD/Docker/n8n"
docker compose down
```

---

# 56. Restart n8n

```bash
cd "/Volumes/Apple Storage HDD/Docker/n8n"
docker compose restart
```

---

# 57. Check n8n Containers

```bash
docker ps
```

All containers:

```bash
docker ps -a
```

---

# 58. View Logs

```bash
docker logs -f n8n
```

Last 100 lines:

```bash
docker logs --tail 100 n8n
```

---

# 59. Check Ollama Models

```powershell
ollama list
```

---

# 60. Update Qwen3

```powershell
ollama pull qwen3:8b
```

---

# 61. Check GPU

```powershell
nvidia-smi
```

For continuous monitoring:

```powershell
nvidia-smi -l 2
```

Stop:

```text
CTRL + C
```

---

# 62. Check Ollama API

Windows:

```powershell
curl http://localhost:11434/api/tags
```

Mac:

```bash
curl http://192.xx.xx.xx:11434/api/tags
```

---

# PART I — SAFE SHUTDOWN

# 63. Stop n8n Before Shutting Down Mac

```bash
cd "/Volumes/Apple Storage HDD/Docker/n8n"
docker compose down
```

Confirm:

```bash
docker ps
```

Then quit Docker Desktop if required.

---

# 64. Stop Windows Ollama

Normally exit Ollama from the Windows tray before shutting down Windows.

Before shutdown, verify that no important inference or workflow is still running.

---

# PART J — BACKUP

# 65. n8n Backup

Critical n8n data is located at:

```text
/Volumes/Apple Storage HDD/Docker/n8n/data
```

Example backup:

```bash
cp -R \
"/Volumes/Apple Storage HDD/Docker/n8n/data" \
"/Volumes/Backup/n8n-data-backup"
```

For production use, use a proper backup strategy rather than relying on manual copies.

---

# 66. Docker Compose Backup

Keep:

```text
docker-compose.yml
```

under version control.

This allows the environment to be recreated:

```text
docker-compose.yml
        │
        ▼
docker compose up -d
        │
        ▼
n8n
```

The application configuration and persistent data are separate.

---

# 67. Recommended Repository Structure

```text
n8n-mac-windows-ollama-setup/
│
├── README.md
│
├── docker-compose.yml
│
└── docs/
    ├── network.md
    ├── troubleshooting.md
    └── backup.md
```

Do not commit sensitive information such as:

```text
API keys
Passwords
n8n credentials
Private tokens
SSH keys
.env files containing secrets
```

---

# PART K — COMPLETE VALIDATION CHECKLIST

## Mac

```bash
docker --version
docker compose version
docker ps
docker inspect n8n
docker logs --tail 50 n8n
ipconfig getifaddr en0
```

## Windows

```powershell
ollama --version
ollama list
nvidia-smi
ipconfig
netstat -ano | findstr :11434
curl http://localhost:11434/api/tags
```

## Mac → Windows

```bash
ping 192.xx.xx.xx
nc -vz 192.xx.xx.xx 11434
curl http://192.xx.xx.xx:11434/api/tags
```

## Windows → Mac

```powershell
ping 192.xx.xx.xx
curl http://192.xx.xx.xx:5678
```

## n8n → Ollama

```text
Ollama Base URL:
http://192.xx.xx.xx:11434

Model:
qwen3:8b
```

---

# PART L — FINAL COMMUNICATION MAP

```text
                              USER
                               │
                               │ Browser
                               ▼
                  http://192.xx.xx.xx:5678
                               │
                               ▼
                    ┌──────────────────┐
                    │     MAC MINI     │
                    │                  │
                    │ Docker Desktop   │
                    │       │          │
                    │       ▼          │
                    │      n8n         │
                    │     :5678        │
                    └────────┬─────────┘
                             │
                             │ HTTP LAN
                             │ TCP 11434
                             ▼
                    ┌──────────────────┐
                    │     WINDOWS      │
                    │                  │
                    │     Ollama       │
                    │     :11434       │
                    │        │         │
                    │        ▼         │
                    │   Qwen3 8B       │
                    │        │         │
                    │        ▼         │
                    │   NVIDIA GPU     │
                    └──────────────────┘
```

---

# 68. Final Directory Structure

```text
/Volumes/Apple Storage HDD/Docker/n8n/
│
├── docker-compose.yml
│
└── data/
    ├── database.sqlite
    ├── config
    ├── binaryData/
    └── other n8n application data
```

---

# 69. Final IP and Port Configuration

| Service  | Machine     |  Port | Endpoint                    |
| -------- | ----------- | ----: | --------------------------- |
| n8n      | Mac Mini M4 |  5678 | `http://192.xx.xx.xx:5678`  |
| Ollama   | Windows     | 11434 | `http://192.xx.xx.xx:11434` |
| Qwen3 8B | Windows     |     — | `qwen3:8b`                  |

Example configuration:

```text
Mac:
192.xx.xx.xx:5678

Windows:
192.xx.xx.xx:11434
```

n8n should therefore use:

```text
http://192.xx.xx.xx:11434
```

as its Ollama endpoint.

---

# 70. Final Recommended Architecture

```text
MAC MINI M4
│
├── Docker Desktop
│   │
│   └── n8n
│       │
│       ├── Port 5678
│       │
│       └── Persistent data
│           └── /Volumes/Apple Storage HDD/Docker/n8n/data
│
└──────────── PRIVATE LAN ────────────┐
                                      │
                                      ▼
                              WINDOWS MACHINE
                                      │
                                      ├── Ollama
                                      │
                                      ├── Port 11434
                                      │
                                      └── Qwen3 8B
                                           │
                                           ▼
                                       NVIDIA GPU
```

## Operational Principle

```text
MAC
  n8n
   │
   │ Workflow execution
   ▼
WINDOWS
  Ollama
   │
   │ Model inference
   ▼
Qwen3 8B
   │
   │ GPU acceleration
   ▼
Response
   │
   ▼
n8n
   │
   ▼
Workflow result
```

This design keeps the **automation layer**, **persistent workflow data**, and **LLM inference layer** separated while allowing both machines to operate together as a single local AI automation environment.
