# seeknal-bpom-rpo-1 Ask Context

This file is loaded into Seeknal Ask sessions. Keep it concise, durable, and
business-specific. It should teach the Ask agent how to reason about this
project without hardcoding project logic into Seeknal itself.

## Business purpose

This project connects to a BPOM (Badan Pengawas Obat dan Makanan) read-only
database containing product registration data, including processed food products
and food additives (BTP). Users are BPOM analysts who ask questions about:
- Number of issued product licenses (NIE / Izin Edar)
- Application trends (permohonan)
- Product categories, factory regions, industrial scale
- Specific product types (AMDK, Garam Beryodium)

## Vocabulary and business definitions

### NIE (Nomor Izin Edar)
- NIE is the product license number, stored in the `nomor` column.
- Counts NIE using 'tanggal' coloumn
- Common user terms: izin edar, NIE, izin terbit.
- Product data for NIE queries spans `t_produk_3_erba` and `t_produk_3_rilis_erla` 
- Food addictives for NIE queries spans `t_btp_3_erba` and `t_btp_3_erla`

### Data Dictionary
- Before generating any response, the system must consult the "data_dictionary" table. This ensures all acronyms, terms, and field definitions are interpreted correctly according to the official documentation.

### Permohonan
- Counts permohonan using 'produk_id' coloumn
- Common user terms: permohonan, permohonan izin edar, jumlah permohonan, permohonan registrasi, registrasi
- Counts permohonan using 'tanggal_bayar' coloumn
- Product data for permohonan queries spans `t_produk_3_erba` and `t_produk_3_rilis_erla` 
- Food addictives for NIE queries spans `t_btp_3_erba` and `t_btp_3_erla`

### Risk Categories (Kategori Dokumen)
- Only tracked in 't_produk_3_erba'
- `301`, `304` = Risiko Tinggi (T / High Risk)
- `302` = Risiko Menengah Tinggi (MT / Medium-High Risk)
- `303` = Risiko Menengah Rendah (MR / Medium-Low Risk)

### Status Codes
- Valid/approved statuses: `'0999'`, `'0906'`, `'9999'`
- ERLA additionally uses: `'0099'`

### Jenis Permohonan (Application Type)
- `301` = Baru (New)
- `302`, `303`, `304`, `305` = Various renewal/change types
- For NIE counts, typically filter to `('301', '305')` for ERBA and
  `('301', '304', '305')` for ERLA.

### Commitment Status (Status Komitmen)
- Only tracked in `t_produk_3_erba`; ERLA does not have this column.
- `4`, `7` = Disetujui (Approved)
- `5` = Dibatalkan (Cancelled)

### AMDK (Air Minum Dalam Kemasan)
- Bottled drinking water products.
- ERBA: `jenis_pangan = '1401'`
- ERLA: `jenis_pangan IN ('651', '652', '655')`

### Garam Beryodium
- Iodized salt products.
- ERBA: `kategori_pangan = '120101000001'`
- ERLA: `kategori_pangan = '12010103'`

### BTP (Bahan Tambahan Pangan)
- Food additives, stored in `t_btp_3_erba` and `t_btp_3_erla`.

### Skala Industri
- ERBA references `m_trader_rba.skala_industri_id`.
- ERLA references `m_trader_rla.skala_industri`.

## Data Architecture

- ERBA tables : `t_produk_3_erba`, `t_btp_3_erba`, `m_trader_rba`
- ERLA tables : `t_produk_3_rilis_erla`, `t_btp_3_erla`, `m_trader_rla`
- No `t_produk_3_erla` or unified `t_produk_3` table exists.
- For queries covering all products, UNION ERBA and ERLA tables.
- For commitment-related queries, use only ERBA tables.

## Knowledge Base
- Before providing any responses, you are required to thoroughly read and reference the context provided in the file located at context/bpom_information.md. Use the technical definitions, database schemas, and project constraints defined there as the primary source of truth for all your answers.

##

## Required SQL patterns
Use `seeknal/sql_pairs/*.yml` for reusable prompt-to-SQL examples. Each pair
should include:

```yaml
name: example_metric
prompt: Natural-language question users ask
intent: What business definition this query implements
sql: |
  SELECT ...
notes:
  - Explain filters, date fields, deduplication keys, and caveats.
```

When a SQL pair directly matches a user question, the Ask agent should execute
that pair as the authoritative business definition before trying ad-hoc SQL.

## Source context workflow

1. Configure sources with `seeknal source connect`.
2. Refresh generated database context with `seeknal source sync <source> --project .`.
3. Keep generated files under `.seeknal/context/sources/` as derived metadata.
4. Put human business context in this file or under `context/*.md`.

## Ask QA workflow

Use `seeknal/tests/*.yml` for executable regression cases. A good test captures:

- user prompt
- expected SQL or SQL result
- expected values when the business answer is known
- notes explaining why the query is correct

Run:

```bash
seeknal ask test --project . --sql-only
seeknal ask test --project .
```

## Guardrails

- Never save passwords, DSNs, API keys, bearer tokens, or private credentials.
- Keep one-off filters in the chat turn; save only reusable business logic.
- Prefer SQL pairs and source context before broad table discovery.
- For unknown or advanced questions, the agent may use `execute_sql`,
  `execute_python`, statistics, and modeling, but conclusions must cite data.
- 