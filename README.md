# selfhosted-qdrant

Self-hosted [Qdrant](https://qdrant.tech) vector search engine deployed on [Dokploy](https://dokploy.com) via Docker Compose. Used as the vector database for [pdf-search](https://github.com/anla-124/pdf-search).

## Stack

- **Qdrant** `v1.14.1` — vector search engine
- **Dokploy** — Docker Compose host, Traefik reverse proxy, domain management
- **Cloudflare** — DNS + TLS termination (Flexible SSL)

## Deployment (Dokploy)

### 1. Prerequisites

- Dokploy instance running with `dokploy-network` external Docker network
- Domain pointed to your server via Cloudflare (e.g. `qdrant.yourdomain.com`)

### 2. Create the Compose service

In Dokploy UI:
1. New service → **Compose**
2. Provider: **GitHub** → select this repo, branch `main`, path `./docker-compose.yml`

### 3. Set the API key

In the service's **Environment** tab, add:

```
QDRANT_API_KEY=<output of: openssl rand -hex 32>
```

> The container runs unauthenticated if this is empty — always set a strong key.

### 4. Deploy

Click **Deploy**. Confirm the container is healthy in the logs.

### 5. Add domain

In the service's **Domains** tab:

| Field | Value |
|---|---|
| Service Name | `qdrant` |
| Host | `qdrant.yourdomain.com` |
| Container Port | `6333` |
| HTTPS | **off** (Cloudflare handles TLS) |

### 6. Redeploy

Click **Deploy** again. The deploy log must show:

```
Container <app>-qdrant-1 Recreated
```

If it shows "Started" instead of "Recreated", click **Stop** then **Deploy** again.

### 7. Verify

```bash
curl https://qdrant.yourdomain.com/healthz -H "api-key: YOUR_KEY"
# Expected: {"title":"qdrant - vector search engine","version":"1.14.1",...}
```

---

## Accessing the Dashboard

Open in your browser:

```
https://qdrant.yourdomain.com/dashboard
```

When prompted, enter your `QDRANT_API_KEY`. The dashboard lets you:
- Browse and create collections
- Inspect and query points
- View cluster/node info
- Trigger and download snapshots

> The dashboard is served on the same port as the REST API (6333). It is not auth-gated at the network level — anyone who knows the URL can reach the login screen. Keep your API key secret.

---

## Connecting from an Application

Add to your app's `.env`:

```env
QDRANT_URL=https://qdrant.yourdomain.com
QDRANT_API_KEY=<your key>
QDRANT_COLLECTION_NAME=documents
```

### JavaScript / TypeScript

```bash
npm install @qdrant/js-client-rest
```

```ts
import { QdrantClient } from '@qdrant/js-client-rest';

const client = new QdrantClient({
  url: process.env.QDRANT_URL,
  apiKey: process.env.QDRANT_API_KEY,
});
```

> The API key is sent as the `api-key` header — **not** `Authorization: Bearer`. The JS client handles this automatically.

### Python

```bash
pip install qdrant-client
```

```python
from qdrant_client import QdrantClient

client = QdrantClient(
    url=os.environ["QDRANT_URL"],
    api_key=os.environ["QDRANT_API_KEY"],
)
```

---

## Environment Variables

| Variable | Required | Description |
|---|---|---|
| `QDRANT_API_KEY` | Yes | API key for authenticating requests. Set in Dokploy UI. |

---

## Volumes

| Volume | Mount path | Purpose |
|---|---|---|
| `qdrant_storage` | `/qdrant/storage` | Collections, vectors, indexes |
| `qdrant_snapshots` | `/qdrant/snapshots` | Snapshot files for backup/restore |

Snapshots can be triggered via the REST API:

```bash
curl -X POST https://qdrant.yourdomain.com/collections/{collection_name}/snapshots \
  -H "api-key: YOUR_KEY"
```

---

## Notes

- gRPC (port 6334) is not exposed — not needed for HTTP clients
- Telemetry is disabled (`QDRANT__TELEMETRY_DISABLED=true`)
- Recovery mode is enabled (`QDRANT_ALLOW_RECOVERY_MODE=true`) to allow startup after crashes
- TLS is terminated by Cloudflare/Traefik — the container runs plain HTTP internally
