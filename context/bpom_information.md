This file contains detailed information regarding the business data processes of the Direktorat Registrasi Pangan Olahan


### NIE (Nomor Izin Edar)
- NIE is the product license number, stored in the `nomor` column.
- Counts NIE using 'tanggal' coloumn
- Common user terms: izin edar, NIE, izin terbit.
- Product data for NIE queries spans `t_produk_3_erba` and `t_produk_3_rilis_erla` 
- Food addictives for NIE queries spans `t_btp_3_erba` and `t_btp_3_erla`

### Permohonan
- Counts permohonan using 'produk_id' coloumn
- Common user terms: permohonan, permohonan izin edar, jumlah permohonan, permohonan registrasi, registrasi
- Counts permohonan using 'tanggal_bayar' coloumn
- Product data for permohonan queries spans `t_produk_3_erba` and `t_produk_3_rilis_erla` 
- Food addictives for NIE queries spans `t_btp_3_erba` and `t_btp_3_erla`

## Temporal & Reliability Filters
- Year Exclusion: Exclude all data associated with the years 1900 and 1970.
- Test Accounts (ERBA): Exclude records where trader_id is 5, 17, 50, or 85.
- Test Accounts (ERLA): Exclude records where trader_id is 3384.
- Brand Specificity: Use exact matches for brand names. If multiple similar brands exist, ignore partial matches and focus strictly on the specific brand requested.

## Industry scale vs status scale
- Scale vs. Status: "Skala industri" or "Skala usaha" refers to "skala_industri_id" or "skala_industri". "Status usaha" refers strictly to the 'status_usaha' column. These are not interchangeable.
- UMKM Classification: Includes records labeled as Mikro, Kecil, and Menengah. Importers: Any record where skala_industri_id or skala_industri is NULL or empty is categorized as an "Importir".

## Specialized Document
- Medium-Low Risk (Code '303'): The status_komitmen column applies only to products with kategori_dokumen '303'.
  - Variation Commitment: Use code '9'.
  - Unevaluated Commitments: Use code '1' in ERBA Table (Re-evaluation of commitment process).
  - Cancelled Commitments: Use code '5' (Commitment cancelled). Identify the most frequent reason for cancellation when requested.

  ## Changes
- Variations/Changes:
- Perubahan mayor ('302') in 'jenis_permohonan' coloumn: Refers to changes in Composition.
- Perubahan minor ('303') in 'jenis_permohonan' coloumn: Refers to administrative/document updates.

## Product Attributes
- Product with contract status (Makloon): For products with status_produksi '304', use the following columns: produsen_id, nama_produsen, alamat_produsen, daerah_produsen, and negara_produsen.

## SQL Transparancy
- If the user requests to see the query used to generate an answer, the system must display the SQL query alongside the result for transparency and verification.

## Query Matching
- If the user’s prompt matches exactly with a file name or a specific task defined in the `seeknal/sql_pairs/*.yml` , the system must prioritize and execute the query provided in that folder as the primary source of truth.

## UnMatched Regional Code
- If a regional code (kode daerah) does not match any entry in the data_dictionary, display the original code as is. Do not attempt to guess or match it with other regional codes.

## NIE Re-Registration
- The re-registration process is carried out 5 years after the marketing authorization (NIE) is issued.

## Default Result Limitation
- If a request does not specify a maximum number of records, the system must limit the output to the top 10 results by default.