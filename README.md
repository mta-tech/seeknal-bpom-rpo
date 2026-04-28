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

## License

Proprietary — Badan Pengawas Obat dan Makanan (BPOM)
