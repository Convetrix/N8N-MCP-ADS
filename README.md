# N8N MCP Ads — Multi-Platform Conversion Tracking

Workflow n8n yang otomatis mengirim Conversion API ke **Meta Ads, Google Ads, TikTok Ads, dan LinkedIn Ads** ketika lead di Kommo CRM berpindah stage ke **"Leads-Win"**.

## Arsitektur Workflow

```
Kommo Webhook
    │
    ▼
Cek Stage = "Leads-Win"
    │
    ▼
Ambil Detail Lead (Kommo API)
    │
    ▼
Hash PII (email, phone, name)
    │
    ├──► Meta Conversions API   (jika utm_source = facebook/instagram/meta)
    ├──► Google Ads Conversion   (jika utm_source = google / ada gclid)
    ├──► TikTok Events API       (jika utm_source = tiktok / ada ttclid)
    └──► LinkedIn Conversions    (jika utm_source = linkedin / ada li_fat_id)
```

## File Workflow

| File | Deskripsi |
|------|-----------|
| `workflows/kommo-leads-win-conversion.json` | Workflow utama |
| `workflows/mcp-ads-tools.json` | MCP Tools wrapper untuk semua platform |
| `config/credentials-template.json` | Template konfigurasi credentials |
| `docs/setup-guide.md` | Panduan setup lengkap |

## Quick Start

1. Import `workflows/kommo-leads-win-conversion.json` ke n8n
2. Import `workflows/mcp-ads-tools.json` ke n8n
3. Isi semua credentials sesuai `config/credentials-template.json`
4. Setup webhook di Kommo ke URL n8n webhook
5. Aktifkan workflow

## Custom Fields Kommo yang Dibutuhkan

Pastikan leads di Kommo memiliki field berikut:
- `utm_source` — sumber traffic (facebook, google, tiktok, linkedin)
- `utm_medium` — medium iklan
- `utm_campaign` — nama campaign
- `gclid` — Google Click ID (auto-capture dari landing page)
- `ttclid` — TikTok Click ID
- `li_fat_id` — LinkedIn First-Party Ad Tracking ID
- `fbp` / `fbc` — Facebook Pixel / Click ID
- `email` — email lead
- `phone` — nomor telepon lead
- `conversion_value` — nilai konversi (opsional)

## Dokumentasi Lengkap

Lihat `docs/setup-guide.md` untuk panduan lengkap.
