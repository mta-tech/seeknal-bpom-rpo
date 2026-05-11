# seeknal-bpom-rpo-1

BPOM (Badan Pengawas Obat dan Makanan) analytics project powered by [Seeknal](https://github.com/seeknal/seeknal). Connects to a read-only PostgreSQL database to answer business questions about product registrations, licenses, and applications.

## Overview

This project operates in **tap-in / read-only connected-source analyst** mode. It connects to an existing BPOM database and uses Seeknal Ask to translate natural-language questions into SQL queries.

Key capabilities:
- Count issued product licenses (NIE / Nomor Izin Edar) by risk category
- Analyze application trends (permohonan) over time
- Query product categories, factory regions, and industrial scale
- Track specific product types (AMDK, Garam Beryodium, BTP)

## Data Sources

| Source | Type | Tables |
|--------|------|--------|
| `warehouse` | PostgreSQL (read-only) | 16 tables, 548 columns |

### Key Tables

**Product Registration (ERBA - Domestic)**
- `warehouse.public.t_produk_3_erba` — Processed food products
- `warehouse.public.t_btp_3_erba` — Food additives (Bahan Tambahan Pangan)
- `warehouse.public.m_trader_rba` — Trader references

**Product Registration (ERLA - Imported)**
- `warehouse.public.t_produk_3_rilis_erla` — Imported processed food products
- `warehouse.public.t_btp_3_erla` — Imported food additives
- `warehouse.public.m_trader_rla` — Imported trader references

**Views & Star Schema**
- `warehouse.public.vw_pemeriksaan_bcc`
- `warehouse.public.vw_pengujian_bcc`
- `star.dim_balai`, `star.dim_komoditi`, `star.dim_lokasi`, `star.dim_waktu`
- `star.fact_pemeriksaan`, `star.fact_pengujian`

## Quick Start

### Prerequisites

- Python 3.11+
- Seeknal CLI installed
- Access to the BPOM PostgreSQL database

### Setup

1. **Clone and enter the project**
   ```bash
   cd seeknal-bpom-rpo-1
   ```

2. **Configure environment variables**
   ```bash
   cp .env.example .env
   # Edit .env and set WAREHOUSE_URL to your PostgreSQL DSN
   ```

3. **Connect the data source**
   ```bash
   seeknal source connect warehouse --connector postgresql --dsn-env WAREHOUSE_URL --project .
   ```

4. **Sync schema context**
   ```bash
   seeknal source sync warehouse --project .
   ```

5. **Test the connection**
   ```bash
   seeknal source test warehouse --project .
   ```

6. **Run regression tests**
   ```bash
   seeknal ask test --project . --sql-only
   ```

7. **Start asking questions**
   ```bash
   seeknal ask chat --project .
   ```

## Common Commands

```bash
# List connected sources
seeknal source list --project .

# Inspect a specific source
seeknal source inspect warehouse --project .

# Sync source context after schema changes
seeknal source sync warehouse --project .

# Run SQL-only tests
seeknal ask test --project . --sql-only

# Run full agent tests (requires LLM configuration)
seeknal ask test --project .

# Interactive chat mode
seeknal ask chat --project .
```

## Project Structure

```
.
├── .env                      # Local secrets (gitignored)
├── .env.example              # Safe placeholder variables
├── seeknal_agent.yml         # Ask mode config & source registry
├── seeknal_project.yml       # Project manifest
├── SEEKNAL_ASK.md            # Business context & vocabulary
├── profiles.yml              # Pipeline credentials (gitignored)
│
├── seeknal/
│   ├── sql_pairs/            # Prompt-to-SQL examples
│   │   ├── bpom_nie.yml      # NIE (license) queries
│   │   ├── bpom_permohonan.yml  # Application queries
│   │   └── bpom_tren.yml     # Trend & product queries
│   └── tests/                # Executable QA tests
│       ├── nie_mr.yml
│       ├── nie_mt.yml
│       ├── nie_t.yml
│       ├── jumlah_permohonan.yml
│       ├── tren_amdk.yml
│       └── ...
│
├── context/                  # User-taught project memory
├── .seeknal/                 # Generated source context
│   └── context/
│       └── sources/
│           └── warehouse/
│               ├── SOURCE.md
│               ├── tables/
│               └── relationships.md
│
└── target/                   # Generated pipeline outputs
```

## Business Vocabulary

| Term | Definition |
|------|------------|
| **NIE** | Nomor Izin Edar — product license number (column: `nomor`) |
| **ERBA** | Domestic product registration |
| **ERLA** | Imported product registration |
| **BTP** | Bahan Tambahan Pangan — food additives |
| **AMDK** | Air Minum Dalam Kemasan — bottled drinking water |
| **Kategori Dokumen** | Risk category: `301`/`304` = Tinggi, `302` = Menengah Tinggi, `303` = Menengah Rendah |
| **Jenis Permohonan** | Application type: `301` = Baru (new), `305` = renewal |
| **Status** | Valid statuses: `0999`, `0906`, `9999` (ERLA also uses `0099`) |

## Example Questions

The following questions have curated SQL pairs and regression tests:

- *Jumlah Izin Edar Pangan Olahan Risiko Menengah Rendah?*
- *Jumlah Izin Edar Pangan Olahan Risiko Menengah Tinggi?*
- *Jumlah Izin Edar Pangan Olahan Risiko Tinggi?*
- *Berapa jumlah permohonan?*
- *Tren Jumlah Izin Edar pangan olahan per tahun?*
- *Tren Produk Air Minum Dalam Kemasan (AMDK)?*
- *Top 10 jumlah izin edar berdasarkan kategori pangan?*

## Testing

Run the SQL-only test suite to verify expected queries execute correctly against the database:

```bash
seeknal ask test --project . --sql-only
```

All 8 tests should pass:
- `jumlah_nie_risiko_menengah_rendah`
- `jumlah_nie_risiko_menengah_tinggi`
- `jumlah_nie_risiko_tinggi`
- `jumlah_nie_mr_disetujui_komitmen`
- `jumlah_nie_mr_dibatalkan_komitmen`
- `jumlah_permohonan_total`
- `tren_izin_edar_per_tahun`
- `tren_amdk`

## Configuration

### LLM Provider (for Ask mode)

Edit `.env` to configure the LLM provider for natural-language-to-SQL translation:

```bash
SEEKNAL_ASK_LLM_PROVIDER=openai
SEEKNAL_ASK_MODEL=gpt-4o-mini
OPENAI_API_KEY=sk-...
```

### Adding New SQL Pairs

Add reusable prompt-to-SQL examples to `seeknal/sql_pairs/<category>.yml`:

```yaml
- name: my_query
  prompt: Natural language question
  intent: What this query implements
  sql: |
    SELECT ...
  notes:
    - Explain filters and caveats
```

### Adding New Tests

Add regression tests to `seeknal/tests/<name>.yml`:

```yaml
name: my_test
prompt: Natural language question
sql: |
  SELECT ...
assertions:
  answer_contains:
    - "expected phrase"
```

## Safety & Notes

- The database connection is **read-only**. No writes or mutations are permitted.
- Never commit `.env` or `profiles.yml` — they contain secrets.
- Product data is split across ERBA (domestic) and ERLA (imported) tables. Queries covering all products must `UNION ALL` both sets.
- Commitment status (`status_komitmen`) is only tracked in ERBA tables.

## Deployment (Docker Worker)

Worker berjalan sebagai Docker container yang mem-poll gateway (kcenter-v2) untuk menerima pertanyaan, menjalankan SQL terhadap warehouse, dan mengembalikan jawaban.

### Arsitektur

```
User (Keycenter UI)
  │
  ▼
iba-api-gateway (Spring Cloud Gateway, port 9000)
  │
  ▼ route: /kcenter-v2/**
kcenter-v2 service
  │
  ▼ work-stream (HTTP long-polling)
seeknal-worker (Docker container)
  │
  ▼ read-only SQL
PostgreSQL (warehouse BPOM)
```

### Prasyarat

- Docker + Docker Compose
- Akses ke jaringan tempat gateway berjalan (`serving_network`)
- Database PostgreSQL BPOM sudah terisi
- LLM API key (Google / OpenAI)

### Langkah Deploy

#### 1. Clone repo ke server

```bash
cd /data/deployment/docker
git clone <repo-url> seeknal-bpom-rpo
cd seeknal-bpom-rpo
```

#### 2. Buat file secrets

**`.env`** — koneksi database dan LLM:

```bash
cp .env.example .env
```

Isi dengan:

```bash
WAREHOUSE_URL=postgresql://<user>:<password>@<db_host>:<port>/<database>?sslmode=disable
GOOGLE_API_KEY=<your-google-api-key>
```

> Jika DB mendukung SSL, gunakan `sslmode=require`. Jika tidak, gunakan `sslmode=disable`.

**`.env.worker`** — koneksi ke gateway:

```bash
cp .env.worker.example .env.worker   # atau buat manual
```

Isi dengan:

```bash
SEEKNAL_WORKER_IMAGE=ghcr.io/mta-tech/seeknal-worker:2.9.1
SEEKNAL_GATEWAY_URL=http://iba-api-gateway:9000/kcenter-v2
# Atau jika worker di server berbeda:
# SEEKNAL_GATEWAY_URL=https://alpha.keycenter.ai/_api/kcenter-v2/services/kcenter/v6
SEEKNAL_API_TOKEN=<token-dari-kcenter>
SEEKNAL_WORKER_TRANSPORT=http
SEEKNAL_WORKER_TASK_QUEUE=seeknal-ask
```

> - Jika worker dan gateway di **server yang sama** dan dalam satu Docker network, gunakan internal URL (`http://iba-api-gateway:9000/kcenter-v2`).
> - Jika worker di **server berbeda**, gunakan public URL (`https://alpha.keycenter.ai/...`).

#### 3. Pastikan Docker network ada

```bash
docker network inspect serving_network
```

Jika belum ada:

```bash
docker network create serving_network
```

#### 4. Sync source context

```bash
docker run --rm \
  --env-file .env \
  -v $(pwd):/app/project \
  ghcr.io/mta-tech/seeknal-worker:2.9.1 \
  seeknal source sync warehouse --project /app/project
```

Ini menghasilkan file di `.seeknal/context/sources/warehouse/` yang berisi schema database. Langkah ini hanya perlu diulang jika schema database berubah.

#### 5. Start worker

```bash
docker compose -f docker-compose.worker.yml up -d
```

#### 6. Verifikasi

```bash
# Cek container running
docker ps | grep seeknal-worker

# Cek logs
docker logs -f seeknal-worker-bpom-rpo

# Cek DB connection
docker exec seeknal-worker-bpom-rpo python -c \
  "import os, psycopg2; conn = psycopg2.connect(os.environ['WAREHOUSE_URL']); print('DB OK')"

# Cek env vars
docker exec seeknal-worker-bpom-rpo printenv WAREHOUSE_URL
docker exec seeknal-worker-bpom-rpo printenv GOOGLE_API_KEY

# Cek project files ter-mount
docker exec seeknal-worker-bpom-rpo ls /app/project/seeknal/sql_pairs/
```

### Update / Redeploy

```bash
cd /data/deployment/docker/seeknal-bpom-rpo
git pull
docker compose -f docker-compose.worker.yml pull
docker compose -f docker-compose.worker.yml up -d --force-recreate
```

Jika schema database berubah, ulangi step 4 (sync source context).

### Resync source context setelah schema berubah

```bash
docker exec seeknal-worker-bpom-rpo \
  seeknal source sync warehouse --project /app/project
```

### Troubleshooting

| Masalah | Penyebab | Solusi |
|---------|----------|--------|
| `502 Bad Gateway` | kcenter-v2 service down | Cek status upstream service di gateway |
| `sslmode=require` error | DB tidak support SSL | Ganti `sslmode=disable` di `WAREHOUSE_URL` |
| Agent bilang "belum ada data" | Project files tidak ter-mount | Cek volume mount, pastikan `seeknal/` dan `SEEKNAL_ASK.md` ada di `/app/project/` |
| Agent bilang "belum ada data" | Source context belum sync | Jalankan `seeknal source sync warehouse` |
| Container tidak re-read `.env` setelah edit | `docker restart` tidak re-read env | Gunakan `docker compose up -d --force-recreate` |
| Worker tidak terima work | Task queue salah | Pastikan `SEEKNAL_WORKER_TASK_QUEUE` sesuai routing di gateway |

### Multi-Org Deployment

Jika 2 org ID mengakses database dan konteks yang sama, cukup **1 worker**. Jika ingin isolasi per org:

```yaml
services:
  seeknal-worker-org1:
    image: ghcr.io/mta-tech/seeknal-worker:2.9.1
    container_name: seeknal-worker-org1
    env_file: .env.worker.org1
    environment:
      SEEKNAL_PROJECT_PATH: /app/project
      SEEKNAL_WORKER_TRANSPORT: http
    volumes:
      - /data/deployment/docker/seeknal-bpom-rpo:/app/project:ro
    networks:
      - serving_network

  seeknal-worker-org2:
    image: ghcr.io/mta-tech/seeknal-worker:2.9.1
    container_name: seeknal-worker-org2
    env_file: .env.worker.org2
    environment:
      SEEKNAL_PROJECT_PATH: /app/project
      SEEKNAL_WORKER_TRANSPORT: http
    volumes:
      - /data/deployment/docker/seeknal-bpom-rpo:/app/project:ro
    networks:
      - serving_network

networks:
  serving_network:
    external: true
```

Buat `.env.worker.org1` dan `.env.worker.org2` dengan `SEEKNAL_API_TOKEN` dan/atau `SEEKNAL_WORKER_TASK_QUEUE` yang berbeda sesuai routing per org.

### Gateway Config (iba-api-gateway)

Pastikan route berikut ada di Spring Cloud Gateway:

```yaml
- id: kcenter-v2
  uri: ${KCENTER_V2_SERVICE_URL}
  predicates:
    - Path=/kcenter-v2/**
  filters:
    - RewritePath=/kcenter-v2/(?<segment>.*), /$\{segment}
```

Dan endpoint worker di-whitelist di `service.open.authentication`:

```yaml
service:
  open:
    authentication:
      - /kcenter-v2/v6/internal/worker/work-stream
      - /kcenter-v2/services/kcenter/v6/internal/worker/work-stream
```

> `KCENTER_V2_SERVICE_URL` **wajib** di-set (tidak ada default value). Jika tidak, gateway akan return 502.

### Database: Buat Read-Only User

```sql
CREATE USER seeknal_reader WITH PASSWORD '<password>';
GRANT CONNECT ON DATABASE rpo_v2 TO seeknal_reader;
GRANT USAGE ON SCHEMA public TO seeknal_reader;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO seeknal_reader;
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT ON TABLES TO seeknal_reader;
```

Gunakan user ini di `WAREHOUSE_URL`.

## License

Proprietary — Badan Pengawas Obat dan Makanan (BPOM)
