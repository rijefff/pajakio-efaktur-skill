# Pajak.io e-Faktur — Agent Skill

Pajak.io is an official partner of the Directorate General of Taxes (DGT/DJP) of
Indonesia. This skill lets an AI agent create and manage **Faktur Pajak**
(Indonesian VAT invoices) on the CTAS / Coretax system through the Pajak.io Open
API — VAT Output, VAT Output Return, VAT Input, Other Documents, and taxpayer
status verification (VSWP).

Version: based on Pajak.io Open API Documentation **v3.0.5**.

---

## ⚠️ Safety rules (read first)

Creating a Faktur Pajak is a **legal tax filing to DGT**, not a casual action.
A wrong or accidental upload can trigger real tax/penalty consequences. The agent
**MUST**:

1. **Default to draft.** Always create with `"autoUploadDjp": false` unless the
   user explicitly says "upload to DJP now". A draft is safe and reversible.
2. **Confirm before irreversible steps.** Always ask for explicit user
   confirmation before calling **upload**, **cancel** (`batal`), or **delete**
   (`hapus`). State clearly what will happen.
3. **Never reveal secrets.** Do not print the raw API token or the digital
   certificate passphrase (`penandatangan.passphrase`) in any output or log.
4. **Verify amounts.** Echo back DPP, PPN, and totals to the user before sending.
5. **Production = real.** Treat the production base URL as live filings. When in
   doubt, use sandbox.

---

## Authentication

Pajak.io authenticates with an **API Key** in the `Authorization` header.

1. Get your **user token** from the Pajak.io dashboard (`my.pajak.io`) or support.
2. **Base64-encode** the raw token.
3. Send the encoded value in the `Authorization` header.

```
Authorization: <base64-encoded-user-token>
Content-Type: application/json
```

Recommended: provide the token via an environment variable (e.g.
`PAJAKIO_TOKEN`), never hard-code it in prompts.

### Environments

| Environment | Base URL                          | Use for                         |
|-------------|-----------------------------------|---------------------------------|
| Sandbox     | `https://sandbox-openapi.pajak.io`| development & testing (default) |
| Production  | `https://openapi.pajak.io`        | live filings to DGT             |

All endpoints below use `{base_url}` = one of the above.

---

## Standard response shape

Every endpoint returns JSON in this form:

```json
{ "code": 200, "data": { }, "status": "OK", "message": "..." }
```

- `code` — HTTP status code (`200` OK, `400` BAD_REQUEST, etc.)
- `status` — `"OK"` on success, otherwise an error label.
- `data` — payload (may be `null`).
- `message` — human-readable explanation (often in Indonesian).

If `code != 200`, show `message` to the user and stop — do not blindly retry.

---

## Endpoint reference

### A. VAT Output (Faktur Keluaran) — `/efaktur/v3/penjualan`

| # | Action            | Method | Path                                  |
|---|-------------------|--------|---------------------------------------|
| 1 | Create            | POST   | `/efaktur/v3/penjualan`               |
| 2 | Create multiple   | POST   | `/efaktur/v3/penjualan/multiple`      |
| 3 | Update            | PUT    | `/efaktur/v3/penjualan`               |
| 4 | Get detail        | POST   | `/efaktur/v3/penjualan/{transactionId}` |
| 5 | Get PDF (base64)  | POST   | `/efaktur/v3/penjualan/get-pdf`       |
| 6 | Get list          | POST   | `/efaktur/v3/penjualan/list?page=1&size=10&sortBy=createdAt&sortOrder=ASC` |
| 7 | Upload to DGT     | POST   | `/efaktur/v3/penjualan/upload`        |
| 8 | Cancel (batal)    | POST   | `/efaktur/v3/penjualan/batal`         |
| 9 | Delete (hapus)    | POST   | `/efaktur/v3/penjualan/hapus`         |

Endpoints 5, 7, 8, 9 take a body of just `{ "transactionId": "..." }`.

### B. VAT Output Return (Retur Keluaran)

| Action          | Method | Path                                                 |
|-----------------|--------|------------------------------------------------------|
| Prepopulate     | POST   | `/efaktur/v3/penjualan/retur/prepopulated`           |
| Save to pajak.io| POST   | `/efaktur/v3/penjualan/retur/save`                   |
| Cancel          | POST   | `/efaktur/v3/penjualan/retur/cancel/:transactionId`  |
| Check status    | POST   | `/efaktur/v3/penjualan/retur/status/:transactionId`  |
| Get detail      | POST   | `/efaktur/v3/penjualan/retur/detail/:transactionId`  |

### C. VAT Input (Faktur Masukan) — `/efaktur/v3/pembelian`

| Action                       | Method | Path                                         |
|------------------------------|--------|----------------------------------------------|
| Prepopulate period           | POST   | `/efaktur/v3/pembelian/prepopulated/masa`    |
| Prepopulate detail (by no.)  | POST   | `/efaktur/v3/pembelian/prepopulated/detail`  |
| Upload credit                | POST   | `/efaktur/v3/pembelian/upload`               |
| Upload invalid               | POST   | `/efaktur/v3/pembelian/invalid`              |
| Confirmation: cancelation    | POST   | `/efaktur/v3/pembelian/konfirmasi/batal`     |

### D. Output Other Document — `/efaktur/v3/dokumen/keluaran`

| Action     | Method | Path                                               |
|------------|--------|----------------------------------------------------|
| Create     | POST   | `/efaktur/v3/dokumen/keluaran`                     |
| Update     | PUT    | `/efaktur/v3/dokumen/keluaran`                     |
| Get status | POST   | `/efaktur/v3/dokumen/keluaran/status/{transactionId}` |
| Get detail | POST   | `/efaktur/v3/dokumen/keluaran/detail/{transactionId}` |
| Upload     | POST   | `/efaktur/v3/dokumen/keluaran/upload/{transactionId}` |
| Delete     | POST   | `/efaktur/v3/dokumen/keluaran/delete/{transactionId}` |
| Get list   | POST   | `/efaktur/v3/dokumen/keluaran/list`                |

### E. VSWP — Taxpayer Status Verification

| Action          | Method | Path                      |
|-----------------|--------|---------------------------|
| Verify by NPWP  | POST   | `/vswp/v2/verify/npwp`    |

Body: `{ "npwp": "...", "tujuan": "..." }`. Returns `nama`, `alamat`,
`statusWp` (e.g. `AKTIF`), `statusSpt`. **Use this to validate a counterparty's
NPWP before creating a VAT Output.**

---

## Primary workflow: create a VAT Output

1. *(Recommended)* Verify the buyer's NPWP via **VSWP** (`/vswp/v2/verify/npwp`).
2. Build the `barangJasa[]` line items; compute `dpp`, `ppn`, and the totals.
3. POST to `/efaktur/v3/penjualan` with `"autoUploadDjp": false` (draft).
4. Read `data.transactionId` from the response and keep it.
5. Show the draft summary to the user and ask whether to upload.
6. If confirmed, POST `transactionId` to `/efaktur/v3/penjualan/upload`.

### Request body (Create VAT Output)

```json
{
  "autoUploadDjp": false,
  "pengganti": false,
  "nofaDiganti": "",
  "kdJenisTransaksi": "TD.00301",
  "keteranganTambahan": { "kode": "", "nomorDokumenPendukung": "" },
  "masaPajak": "11",
  "tahunPajak": "2024",
  "tanggalFaktur": "2024-11-16",
  "noInvoice": "INV-2024-001",
  "barangJasa": [
    {
      "jenis": "BARANG",
      "nama": "Cordless Brad Nailer INGCO CBNLI2028",
      "kode": "845900",
      "jumlah": 1,
      "kodeSatuan": "UM.0021",
      "harga": 3763500,
      "totalHarga": "3763500",
      "diskon": 0,
      "tarifPpn": 11,
      "dpp": 3763500,
      "cekDppLain": false,
      "dppLain": 3763500,
      "ppn": 413985,
      "tarifPpnbm": 0,
      "ppnbm": 0
    }
  ],
  "lawanTransaksi": {
    "identityType": "NPWP",
    "identityValue": "1091031210911629",
    "nitku": "1091031210911629000000",
    "nama": "Kongsi Tirta",
    "telp": "0218720712",
    "kodeNegara": "IDN",
    "alamatJalan": "Gatot Subroto 23, Jakarta Selatan, DKI Jakarta 12990",
    "kota": "DKI Jakarta",
    "email": ""
  },
  "terminPembayaran": { "type": "NORMAL", "dpp": 0, "ppn": 0, "ppnbm": 0, "nofa": "" },
  "totalDpp": 3763500,
  "totalDppLain": 3763500,
  "totalPpn": 413985,
  "totalPpnBm": 0,
  "penandatangan": {
    "npwp": "1305202311840002",
    "nama": "FERRY IRAWAN",
    "jabatan": "CEO",
    "kota": "DKI Jakarta",
    "passphrase": "<digital-cert-passphrase>"
  },
  "pembuatFaktur": { "npwp": "1305202311840002", "nama": "FERRY IRAWAN" },
  "nitkuPkp": { "nitku": "1091031210910416000001", "nama": "Cabang Jakarta" }
}
```

### Success response

```json
{
  "code": 200,
  "data": {
    "transactionId": "6fb7e0fa-92cd-4d27-863e-b9433b37e81e",
    "noInvoice": "INV-2024-001"
  },
  "message": "SUCCESS SENDING REQUEST TO CREATE VAT OUTPUT",
  "status": "OK"
}
```

### Key field notes

- `autoUploadDjp` — `false` keeps it a draft (safe); `true` auto-uploads to DGT.
- `pengganti` — `true` for a revised invoice; then `nofaDiganti` (the replaced
  tax-invoice number) is required.
- `kdJenisTransaksi` — DGT transaction-type code, e.g. `TD.00301` (to non-VAT
  collector). See the DGT reference for the full list.
- `terminPembayaran.type` — `NORMAL`, `UANG_MUKA` (down payment), or `PELUNASAN`
  (repayment). For `NORMAL`, set its `dpp`/`ppn`/`ppnbm` to `0` and `nofa` to `""`.
- `penandatangan` — optional; if present, all its sub-fields are mandatory.
  `passphrase` is the digital-certificate passphrase — handle as a secret.

---

## Example: create a draft via curl

```bash
curl -X POST "$PAJAKIO_BASE_URL/efaktur/v3/penjualan" \
  -H "Authorization: $PAJAKIO_TOKEN" \
  -H "Content-Type: application/json" \
  -d @create-vat-output.json
```

Then upload the returned draft:

```bash
curl -X POST "$PAJAKIO_BASE_URL/efaktur/v3/penjualan/upload" \
  -H "Authorization: $PAJAKIO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{ "transactionId": "6fb7e0fa-92cd-4d27-863e-b9433b37e81e" }'
```

---

## Common error messages

| code | message (example)                                                      | meaning                                  |
|------|------------------------------------------------------------------------|------------------------------------------|
| 400  | `Hanya faktur yang berstatus DRAFT atau DITOLAK yang dapat di upload`   | only DRAFT/rejected invoices can upload  |
| 400  | `Tidak bisa membatalkan Faktur yang berstatus APPROVAL_SUKSES`         | approved invoices can't be cancelled here |

---

## Onboarding prerequisites

Before any of this works the company must:
1. Register at `https://my.pajak.io`.
2. Contact Pajak.io support / H2H team (`https://pajak.io/h2h-solution`) to enable
   Open API access and obtain a sandbox + production token.
