# N8N + WAHA Automation Stack

WhatsApp AI chatbot automation using N8N (workflow engine), WAHA (WhatsApp HTTP API),
Redis (AI Agent chat memory), and Postgres (N8N database).

Based on the tutorial: https://www.youtube.com/watch?v=KkKlfAb3TSI

---

## Option A — Docker Compose (local dev)

```bash
cd apps/n8n-waha-automation/docker-compose
cp .env.example .env        # edit credentials if desired
docker compose up -d
```

| Service        | URL                                |
|----------------|------------------------------------|
| N8N            | http://localhost:5678              |
| WAHA Dashboard | http://localhost:3000/dashboard    |

### Post-startup steps (Docker Compose)

1. **N8N first run** — create your admin account at http://localhost:5678
2. **Install WAHA community node** — go to Settings → Community Nodes → install `@devlikeapro/n8n-nodes-waha`
3. Restart N8N after node install: `docker compose restart n8n`
4. **Scan WAHA QR code** — http://localhost:3000/dashboard → start a session → scan QR
5. Continue with the [workflow setup](#workflow-setup) below

---

## Option B — Kubernetes via ArgoCD (girus cluster)

The ArgoCD Application is already registered in `app-of-apps/values.yaml` with
`n8nWahaAutomation.enabled: true`. After pushing this repo, ArgoCD will:

1. Create the `automation` namespace
2. Deploy Postgres, Redis, WAHA, and N8N with PVCs backed by `local-path`
3. Run an init container on N8N to pre-install `@devlikeapro/n8n-nodes-waha`

### Port access

Socat systemd services are already **enabled and running** on the host:

| Service | Host port | NodePort | Command to check              |
|---------|-----------|----------|-------------------------------|
| N8N     | 5678      | 30567    | `systemctl --user status n8n-port-forward` |
| WAHA    | 3030      | 30030    | `systemctl --user status waha-port-forward` |

| Service        | URL                                |
|----------------|------------------------------------|
| N8N            | http://localhost:5678              |
| WAHA Dashboard | http://localhost:3030/dashboard    |

### Monitor rollout

```bash
kubectl get pods -n automation -w
kubectl logs -n automation deployment/n8n -f
kubectl logs -n automation deployment/waha -f
```

### Post-deployment steps (Kubernetes)

1. **N8N first run** — create your admin account at http://localhost:5678
   - The WAHA community node was pre-installed by the init container; no manual install needed
2. **Scan WAHA QR code** — http://localhost:3030/dashboard → start a session → scan QR
3. Continue with the [workflow setup](#workflow-setup) below

---

## Workflow Setup

Build this flow in N8N (matching the tutorial):

```
Webhook (POST /webhook/webhook)
  └─ Set  (extract: chatId, message, sessionName)
      └─ Switch  (filter: event == "message")
          └─ AI Agent  (Google Gemini model + Redis chat memory)
              └─ WAHA: sendSeen  (mark message as read)
                  └─ WAHA: sendText  (reply with AI response)
```

### Step-by-step

1. **Webhook node** — path: `webhook`, method: POST
2. **Set node** — map fields from `{{ $json.payload }}`:
   - `chatId` → `{{ $json.payload.from }}`
   - `message` → `{{ $json.payload.body }}`
   - `sessionName` → `{{ $json.payload.sessionName }}`
3. **Switch node** — route on `{{ $json.event }}` == `"message"`
4. **AI Agent node**
   - Model: Google Gemini (add credentials: API key from https://aistudio.google.com/app/apikey)
   - Memory: Redis Chat Memory → host `redis`, port `6379`, password `default` (K8s) or as set in `.env` (Docker Compose)
   - System prompt: define your bot's persona
5. **WAHA: Mark as seen** — session: `{{ $json.sessionName }}`, chatId: `{{ $json.chatId }}`
6. **WAHA: Send text** — session: `{{ $json.sessionName }}`, chatId: `{{ $json.chatId }}`, text: `{{ $json.output }}`

### WAHA credentials in N8N

| Field    | Docker Compose          | Kubernetes              |
|----------|-------------------------|-------------------------|
| Base URL | `http://waha:3000`      | `http://waha:3000`      |
| API Key  | *(leave empty)*         | *(leave empty)*         |

---

## Updating passwords

The passwords are set to `default` for simplicity (matching the tutorial).
To change them for a production deployment:

**Docker Compose** — edit `.env` and recreate:
```bash
docker compose down -v   # WARNING: destroys volumes
docker compose up -d
```

**Kubernetes** — edit `base/secrets.yaml` (values must be base64-encoded):
```bash
echo -n "mynewpassword" | base64
```
Then update the secret and roll the deployments:
```bash
kubectl rollout restart deployment -n automation
```
