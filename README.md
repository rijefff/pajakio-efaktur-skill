# Pajak.io e-Faktur — Agent Skill

An **Agent Skill** that teaches AI agents (Claude Code, Cursor, and other
agentic tools) how to create and manage Indonesian tax invoices (**Faktur
Pajak**) through the [Pajak.io](https://pajak.io) Open API for CTAS / Coretax.

With this skill loaded, a Pajak.io customer can do things like:

> "Buatkan faktur penjualan ke PT Kongsi Tirta, 1 unit nailer Rp3.763.500,
> PPN 11%, masa pajak November 2024."

…and the agent will verify the buyer's NPWP, build the payload, create a **draft**
VAT Output, and ask for confirmation before uploading it to DGT.

## What's inside

| File | Purpose |
|------|---------|
| [`SKILL.md`](./SKILL.md) | The agent skill — auth, endpoints, safety rules, payloads |
| [`examples/create-vat-output.json`](./examples/create-vat-output.json) | Sample Create VAT Output request body |

## Safety first

Creating a Faktur Pajak is a real tax filing to the Directorate General of Taxes.
The skill is built to be safe by default:

- Creates **drafts** (`autoUploadDjp: false`) unless told otherwise.
- Requires explicit confirmation before **upload**, **cancel**, or **delete**.
- Never prints the API token or the digital-certificate passphrase.

## Quick start

1. Get a Pajak.io Open API token from `my.pajak.io` (or support).
2. Base64-encode it and expose it as an environment variable:
   ```bash
   export PAJAKIO_TOKEN="<base64-encoded-token>"
   export PAJAKIO_BASE_URL="https://sandbox-openapi.pajak.io"   # sandbox first
   ```
3. Point your agent at `SKILL.md` and ask it to create a VAT Output.

## Environments

| Environment | Base URL |
|-------------|----------|
| Sandbox     | `https://sandbox-openapi.pajak.io` |
| Production  | `https://openapi.pajak.io` |

## Onboarding

You must register at [my.pajak.io](https://my.pajak.io) and contact the Pajak.io
H2H team at [pajak.io/h2h-solution](https://pajak.io/h2h-solution) to enable Open
API access and receive sandbox + production tokens.

## License

MIT — see [LICENSE](./LICENSE).

---

Based on Pajak.io Open API Documentation v3.0.5.
