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

### Data Dictionary (MANDATORY — NEVER SKIP)
- The `data_dictionary` table at `warehouse.public.data_dictionary` is the
  authoritative source for resolving all codes to human-readable labels.
- **NEVER present raw numeric or coded values to the user.** Every code must be
  replaced with its `deskripsi` label before showing the answer. This is
  non-negotiable.
- Table structure:
  - `sumber` — source scope (e.g. 'ERBA dan ERLA')
  - `kategori` — category name (e.g. 'SKALA_INDUSTRI dan SKALA_INDUSTRI_ID',
    'JENIS_PERMOHONAN', 'KATEGORI_DOKUMEN')
  - `kode` — the code value (e.g. '1')
  - `deskripsi` — the human-readable label (e.g. 'Mikro')

#### Resolution workflow (MANDATORY after EVERY SQL pair execution)

After executing any SQL pair, check the result columns against this table.
If a column contains coded values, you MUST execute a follow-up query to
data_dictionary and replace codes with labels BEFORE presenting the answer.

| SQL pair | Column with codes | Data dictionary kategori | Special handling |
|---|---|---|---|
| `tren_btp_skala_industri` | `skala_industri_nama` (1,2,3,4,NULL) | `SKALA_INDUSTRI dan SKALA_INDUSTRI_ID` | NULL/empty → "Importir" |
| `jumlah_izin_edar_skala_industri` | `SKALA_INDUSTRI_ID` (1,2,3,4) | `SKALA_INDUSTRI dan SKALA_INDUSTRI_ID` | NULL/empty → "Importir" |
| `jumlah_izin_edar_skala_usaha` | `skala_industri_id` (1,2,3,4) | `SKALA_INDUSTRI dan SKALA_INDUSTRI_ID` | NULL/empty → "Importir" |
| `jumlah_permohonan_per_jenis` | `jenis_permohonan` (301-305) | `JENIS_PERMOHONAN` | — |
| `tren_izin_edar_tahun_daerah_pabrik` | `daerah_pabrik` (integer codes) | `DAERAH_TRADER, DAERAH_PABRIK, DAERAH_PRODUSEN, PROVINSI_ID, KOTAKAB_ID` | Convert: `(daerah_pabrik / 100)::text` to match kode format |
| `produk_berdasarkan_nama_daerah_pabrik` | `daerah_pabrik` (integer codes) | `DAERAH_TRADER, DAERAH_PABRIK, DAERAH_PRODUSEN, PROVINSI_ID, KOTAKAB_ID` | Convert: `(daerah_pabrik / 100)::text` to match kode format |

**SQL pairs that already resolve internally (no follow-up needed):**
- `tren_amdk_skala_industri` — already JOINs data_dictionary

**Follow-up query template:**
```sql
SELECT kode, deskripsi
FROM warehouse.public.data_dictionary
WHERE kategori = '<relevant_category>'
ORDER BY kode
```

**For regional codes (daerah_pabrik), use this conversion:**
Product tables store codes as integers (e.g. `7271`) but `data_dictionary.kode`
uses decimal format (e.g. `72.71`). When resolving, apply:
`kode = (daerah_pabrik::numeric / 100)::text`
Example: 7271 / 100 = '72.71' which matches data_dictionary.kode.

**For skala industri, handle edge cases:**
- `NULL` or empty string or single space `' '` → label as **Importir**
- `TRIM()` the code before matching to handle trailing spaces

**All data_dictionary categories (complete reference):**
When ANY of these column names or values appear in query results, resolve them
using the corresponding kategori.

- `SKALA_INDUSTRI dan SKALA_INDUSTRI_ID` → 1=Mikro, 2=Kecil, 3=Menengah, 4=Besar
  Columns: `skala_industri_id`, `skala_industri`
- `JENIS_PERMOHONAN` → 301=Permohonan Baru, 302=Perubahan Mayor, 303=Perubahan Minor, 304=Daftar Ulang, 305=Baru Notifikasi
  Columns: `jenis_permohonan`
- `KATEGORI_DOKUMEN` → 301=Tinggi, 302=Menengah Tinggi, 303=Menengah Rendah, 304=Tinggi Notifikasi
  Columns: `kategori_dokumen`
- `STATUS_KOMITMEN` → 0=Draft, 1=Proses Penilaian Kembali, 2=Verifikasi, 4=Disetujui, 5=Dibatalkan, 7=Disetujui Dengan Catatan, 8=Validasi Pembatalan, 9=Variasi Komitmen
  Columns: `status_komitmen`
- `DAERAH_TRADER, DAERAH_PABRIK, DAERAH_PRODUSEN, PROVINSI_ID, KOTAKAB_ID` → 512 regional codes (kabupaten/kota)
  Columns: `daerah_pabrik`, `daerah_trader`, `daerah_produsen`, `provinsi_id`, `kotakab_id`
  Requires conversion: `kode = (column_value::numeric / 100)::text`
- `BENTUK_SEDIAAN` → 101=Cair/Pasta, 102=Serbuk, 103=Bahan Penolong, 104=Gas, 105=Padat
  Columns: `bentuk_sediaan`
- `JENIS_BTP` → 29 BTP type codes (e.g. 13=Pengkarbonasi, 40=Pengemulsi, 43=Penguat Rasa)
  Columns: `jenis_btp`
- `JENIS_DOKUMEN` → 000=Belum Dikategorikan
  Columns: `jenis_dokumen`
- `JENIS_PRODUK_BTP` → 301=Tunggal, 302=Campuran, 303=Perisa, 304=Bahan Penolong
  Columns: `jenis_produk_btp`
- `KEMASAN_ID` → 15 packaging codes (e.g. 1=Kaca/Keramik, 2=Plastik tunggal, 5=Logam)
  Columns: `kemasan_id`
- `KLASIFIKASI_ID` → 13 classification codes (e.g. 301=Makanan, 302=Minuman, 303=BTP)
  Columns: `klasifikasi_id`
- `KODE_KBLI` → 95 business activity codes
  Columns: `kode_kbli`
- `NEGARA_PABRIK dan NEGARA_PRODUSEN` → 102 country codes (e.g. AE=Uni Emirat Arab, CN=Tiongkok, ID=Indonesia)
  Columns: `negara_pabrik`, `negara_produsen`
- `PERUNTUKAN` → 0000=Pangan Umum, 0201=Pangan Peruntukan
  Columns: `peruntukan`
- `STATUS` → 89 status codes (e.g. 0999=Berlaku, 0906=Berlaku, 0099=Tidak Berlaku, 0000=Dihapus)
  Columns: `status`
- `STATUS_PRODUK` → 302=Impor, 306=Single MD Induk, 307=Single MD Anak
  Columns: `status_produk`
- `STATUS_USAHA` → 31=Produsen, 33=Importir
  Columns: `status_usaha`
- `SUB_KEMASAN_ID` → 37 sub-packaging codes (e.g. 101=Kaca, 201=PET, 301=Kertas)
  Columns: `sub_kemasan_id`

#### Fallback resolution for unmapped coded values

If a query result contains coded values that are **NOT listed in the mapping table
above**, do NOT show the raw codes to the user. Instead, follow these steps:

1. **Discover the correct category** — run the `data_dictionary_categories` SQL pair
   to list all available categories:
   ```sql
   SELECT DISTINCT kategori
   FROM warehouse.public.data_dictionary
   ORDER BY kategori
   ```
2. **Identify the matching category** — pick the category that semantically matches
   the coded column (e.g. a column `status_usaha` likely maps to `STATUS_USAHA`).
3. **Resolve the codes** — run the `data_dictionary_by_category` SQL pair with the
   discovered category:
   ```sql
   SELECT kode, deskripsi
   FROM warehouse.public.data_dictionary
   WHERE kategori = '<discovered_category>'
   ORDER BY kode
   ```
4. **Replace codes with labels** in the final answer.

This fallback applies to ALL future SQL pairs, ad-hoc queries, and any new
data_dictionary categories that have not yet been added to the mapping table.

### Direct data_dictionary lookups

When the user asks a **simple reference question** (e.g. "apa saja bentuk peruntukan?",
"apa saja status yang ada?", "berapa jenis BTP?", "apa saja kemasan?"), do NOT
probe schemas or build complex ad-hoc queries. Instead, execute a single direct
query to data_dictionary:

```sql
SELECT kode, deskripsi
FROM warehouse.public.data_dictionary
WHERE kategori = '<kategori>'
ORDER BY kode
```

Common question-term to kategori mapping:

| User asks about | Use kategori |
|---|---|
| bentuk sediaan, bentuk produk | `BENTUK_SEDIAAN` |
| peruntukan | `PERUNTUKAN` |
| status, status izin | `STATUS` |
| status komitmen | `STATUS_KOMITMEN` |
| status usaha | `STATUS_USAHA` |
| status produk | `STATUS_PRODUK` |
| skala industri, skala usaha | `SKALA_INDUSTRI dan SKALA_INDUSTRI_ID` |
| jenis permohonan | `JENIS_PERMOHONAN` |
| jenis BTP, jenis bahan tambahan pangan | `JENIS_BTP` |
| jenis produk BTP | `JENIS_PRODUK_BTP` |
| kategori dokumen, kategori risiko | `KATEGORI_DOKUMEN` |
| kemasan | `KEMASAN_ID` |
| sub kemasan | `SUB_KEMASAN_ID` |
| klasifikasi | `KLASIFIKASI_ID` |
| negara pabrik, negara produsen | `NEGARA_PABRIK dan NEGARA_PRODUSEN` |
| daerah, kabupaten, kota, provinsi | `DAERAH_TRADER, DAERAH_PABRIK, DAERAH_PRODUSEN, PROVINSI_ID, KOTAKAB_ID` |
| KBLI, kode KBLI | `KODE_KBLI` |
| jenis dokumen | `JENIS_DOKUMEN` |
| semua kategori, daftar kategori | Run `SELECT DISTINCT kategori FROM warehouse.public.data_dictionary ORDER BY kategori` |

If the user's term is not listed above, match it to the closest kategori name
semantically. These are **single-query questions** — no schema probing needed.

### Permohonan
- Counts permohonan using 'produk_id' coloumn
- Common user terms: permohonan, permohonan izin edar, jumlah permohonan, permohonan registrasi, registrasi
- **Tanggal column for permohonan: ALWAYS use `tanggal_bayar` (payment date). NEVER use `tanggal_aju` (submission date).**
  - `tanggal_bayar` = tanggal pembayaran, used for counting permohonan (when the application is officially registered/paid).
  - `tanggal_aju` = tanggal pengajuan, submission date only. Do NOT use this for permohonan counts.
  - `tanggal` = tanggal terbit izin edar (NIE issue date), used for NIE counts only.
  - `tanggal_berkas`, `tanggal_diambil`, `tanggal_exp`, etc. = other process dates, not for counting.
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

## Data Quality & Query Rules

### Data exclusions (apply to ALL queries)
- **Year exclusion**: Exclude all data with year 1900 or 1970. Add filter:
  `EXTRACT(YEAR FROM tanggal) NOT IN (1900, 1970)` or
  `EXTRACT(YEAR FROM tanggal_bayar) NOT IN (1900, 1970)` as appropriate.
- **Test accounts (ERBA)**: Exclude records where `trader_id IN (5, 17, 50, 85)`.
- **Test accounts (ERLA)**: Exclude records where `trader_id = 3384`.

### Scale vs Status (do not confuse)
- **Skala industri / skala usaha** refers to `skala_industri_id` or `skala_industri`.
- **Status usaha** refers strictly to the `status_usaha` column.
- These are NOT interchangeable.

### UMKM Classification
- UMKM includes: Mikro (1), Kecil (2), Menengah (3).
- Importir: any record where `skala_industri_id` or `skala_industri` is NULL or empty.

### Commitment details for Medium-Low Risk (kategori_dokumen '303')
- The `status_komitmen` column applies ONLY to products with `kategori_dokumen = '303'`.
- Code `'9'` = Variation commitment.
- Code `'1'` = Unevaluated / re-evaluation of commitment process (ERBA only).
- Code `'5'` = Cancelled commitment.

### Jenis Permohonan details
- `302` (Perubahan Mayor) = changes in Composition.
- `303` (Perubahan Minor) = administrative/document updates.

### Makloon products (status_produksi '304')
- For products with `status_produksi = '304'`, use columns:
  `produsen_id`, `nama_produsen`, `alamat_produsen`, `daerah_produsen`, `negara_produsen`.

### Query behavior
- **SQL transparency**: If the user requests to see the query, display the SQL alongside the result.
- **Unmatched regional codes**: If a daerah code does not match any data_dictionary entry, display the original code as-is. Do not guess.
- **NIE re-registration**: Re-registration is carried out 5 years after the NIE is issued.
- **Default result limit**: If the user does not specify a maximum, limit output to top 10 results.
- **Brand specificity**: Use exact matches for brand names. Ignore partial matches; focus on the specific brand requested.

## Required SQL patterns

`seeknal/sql_pairs/*.yml` contains authoritative prompt-to-SQL definitions that
the Ask agent MUST follow. Each pair maps a business question to a verified SQL
query. When a SQL pair matches the user's question (even partially or by
semantic similarity), the agent MUST execute the pair's SQL exactly as written.

### Rules

1. **Never rewrite a matched SQL pair.** Do not replace columns, add/remove
   filters, change grouping, or swap table references. Execute the SQL from
   the pair verbatim.
2. **Match by intent, not just exact text.** If the user asks "top 10 kategori
   pangan" or "kategori pangan terbanyak" and a SQL pair covers that topic,
   treat it as a match and use the pair's SQL.
3. **Use the pair's column choices.** For example, the pair
   `top_10_izin_edar_kategori_pangan` uses `kategori_pangan` with
   `LEFT(kategori_pangan, 2)` — do NOT substitute `nama_kategori` or remove the
   `LEFT()` grouping.
4. **Preserve all filters.** SQL pairs include filters like
   `tanggal IS NOT NULL` and `jenis_permohonan IN (...)` for correct business
   logic. Do not drop them.
5. **Only generate ad-hoc SQL when no SQL pair matches.** If the question has
   no corresponding pair, the agent may write new SQL, but should follow the
   patterns and conventions established in existing pairs.
6. **Always resolve coded values after execution.** After running a SQL pair,
   check the result for any columns containing numeric codes (skala_industri,
   jenis_permohonan, daerah_pabrik, kategori_dokumen, status_komitmen, etc.).
   If coded values are found, run a follow-up query to `data_dictionary` to
   resolve them to human-readable `deskripsi` labels before presenting the
   answer. See the "Data Dictionary" section for the full mapping table.

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