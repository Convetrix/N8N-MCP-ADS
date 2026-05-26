# Setup Guide — N8N MCP Ads Conversion Tracking

## Overview

Workflow ini terdiri dari 2 komponen utama:
1. **`kommo-leads-win-conversion.json`** — Workflow otomatis yang mengirim conversion API saat lead Win
2. **`mcp-ads-tools.json`** — MCP Server yang bisa dipakai AI agent untuk manage & query ads

---

## Langkah 1: Setup Kommo CRM

### 1.1 Buat Custom Fields

Buka Kommo > Settings > Custom Fields > Leads, buat field sesuai `config/kommo-custom-fields.json`.

### 1.2 Setup Webhook

1. Kommo > Settings > Integrations > Webhooks
2. Tambah webhook baru:
   - **URL**: `https://your-n8n.domain/webhook/kommo-leads-win`
   - **Events**: ✅ Lead status changed
3. Simpan

### 1.3 Generate API Token

Kommo > Settings > Integrations > Private Integrations > Buat baru, copy Long-lived token.

### 1.4 Capture UTM Parameters di Landing Page

Pastikan landing page Anda menyimpan UTM & click ID ke form/cookie, lalu kirim ke Kommo saat lead masuk. Contoh script:

```javascript
// Tambahkan di landing page Anda
(function() {
  const params = new URLSearchParams(window.location.search);
  const fields = ['utm_source','utm_medium','utm_campaign','utm_content','utm_term','gclid','ttclid','li_fat_id'];
  fields.forEach(f => {
    if (params.get(f)) sessionStorage.setItem(f, params.get(f));
  });
  // Simpan fbp/fbc dari cookie
  document.cookie.split(';').forEach(c => {
    const [k,v] = c.trim().split('=');
    if (k === '_fbp' || k === '_fbc') sessionStorage.setItem(k.replace('_',''), v);
  });
})();

// Saat submit form, tambahkan ke hidden fields
function getTrackingData() {
  return {
    utm_source:   sessionStorage.getItem('utm_source')   || 'organic',
    utm_medium:   sessionStorage.getItem('utm_medium')   || '',
    utm_campaign: sessionStorage.getItem('utm_campaign') || '',
    gclid:        sessionStorage.getItem('gclid')        || '',
    ttclid:       sessionStorage.getItem('ttclid')       || '',
    li_fat_id:    sessionStorage.getItem('li_fat_id')    || '',
    fbp:          sessionStorage.getItem('fbp')          || '',
    fbc:          sessionStorage.getItem('fbc')          || '',
    ip_address:   '', // server-side
    user_agent:   navigator.userAgent
  };
}
```

---

## Langkah 2: Setup n8n

### 2.1 Install n8n

```bash
npm install -g n8n
# atau pakai Docker:
docker run -it --rm --name n8n -p 5678:5678 -v ~/.n8n:/home/node/.n8n docker.n8n.io/n8nio/n8n
```

### 2.2 Set Environment Variables

Di n8n Settings > Variables, tambahkan semua variabel dari `config/credentials-template.json`:

```
KOMMO_SUBDOMAIN=yourcompany
META_PIXEL_ID=1234567890
META_ACCESS_TOKEN=EAAxxxxx...
GOOGLE_ADS_CUSTOMER_ID=123-456-7890
GOOGLE_ADS_DEVELOPER_TOKEN=xxxxx
GOOGLE_ADS_CONVERSION_ACTION_ID=123456789
TIKTOK_PIXEL_CODE=ABCDEFG
TIKTOK_ACCESS_TOKEN=xxxxx
LINKEDIN_ACCESS_TOKEN=xxxxx
LINKEDIN_CONVERSION_ID=123456789
WEBSITE_URL=https://yoursite.com
```

### 2.3 Setup Credentials di n8n

#### Kommo API Token
1. n8n > Credentials > New > HTTP Header Auth
2. Name: `Kommo API Token`
3. Header Name: `Authorization`, Value: `Bearer YOUR_KOMMO_TOKEN`

#### Google Ads OAuth2
1. n8n > Credentials > New > OAuth2 API
2. Ikuti panduan Google Cloud Console untuk setup OAuth2
3. Scopes: `https://www.googleapis.com/auth/adwords`

### 2.4 Import Workflows

1. Buka n8n > Workflows > Import from File
2. Import `workflows/kommo-leads-win-conversion.json`
3. Import `workflows/mcp-ads-tools.json`
4. Assign credentials yang sudah dibuat ke masing-masing node
5. Aktifkan kedua workflow

---

## Langkah 3: Setup Masing-Masing Platform

### Meta Ads (Facebook/Instagram)

1. **Business Manager** > Events Manager > Add Event Source > Website
2. Copy **Pixel ID**
3. **Meta Business Manager** > System Users > Create System User (Admin)
4. Generate token dengan scope: `ads_management`, `business_management`
5. Set `META_PIXEL_ID` dan `META_ACCESS_TOKEN`

### Google Ads

1. **Google Ads API Center** > Apply for Developer Token
2. **Google Cloud Console** > Enable Google Ads API > Create OAuth2 credentials
3. **Google Ads** > Tools > Measurement > Conversions > New Conversion Action > Import > CRM
4. Copy Conversion Action ID dari URL
5. Set semua `GOOGLE_ADS_*` variables

### TikTok Ads

1. **TikTok Events Manager** > Web Events > Add Events > API
2. Copy **Pixel Code**
3. Di bagian Access Token, generate token
4. Set `TIKTOK_PIXEL_CODE` dan `TIKTOK_ACCESS_TOKEN`

### LinkedIn Ads

1. **Campaign Manager** > Analyze > Conversion Tracking > Create Conversion
2. Pilih type: **Online** > API
3. Copy **Conversion ID** dari URL
4. **Developer Portal** > Create App > Request `rw_conversions` scope
5. Set `LINKEDIN_ACCESS_TOKEN` dan `LINKEDIN_CONVERSION_ID`

---

## Langkah 4: Test

### Test Meta

Set `META_TEST_EVENT_CODE` ke kode test dari Events Manager, lalu buka tab **Test Events** di Meta Events Manager.

### Test Google

Gunakan Google Ads API **Test Account** atau set `partialFailure: true` dan periksa response untuk errors.

### Test TikTok

Set `TIKTOK_TEST_EVENT_CODE` dari TikTok Events Manager > Test Events.

### Test Workflow End-to-End

1. Buat lead test di Kommo dengan UTM source yang sesuai
2. Pindahkan lead ke stage **Leads-Win**
3. Periksa n8n execution log
4. Verifikasi event di masing-masing platform dashboard

---

## Menggunakan MCP Ads Tools

Setelah `mcp-ads-tools.json` aktif, Anda bisa menghubungkannya ke AI agent (seperti n8n AI Agent node atau Claude).

**MCP Server URL**: `https://your-n8n.domain/mcp/mcp-ads`

Contoh pertanyaan yang bisa ditanyakan ke AI:
- "Tampilkan performa campaign Meta bulan ini"
- "Berapa leads Win dari Google Ads minggu ini?"
- "Kirim conversion event untuk lead ID 123456"
- "Bandingkan CPL antara TikTok dan Meta bulan lalu"

---

## Troubleshooting

| Error | Penyebab | Solusi |
|-------|----------|--------|
| `400 Bad Request` dari Meta | Pixel ID salah / token expired | Cek META_PIXEL_ID dan regenerate token |
| `401 Unauthorized` dari Google | OAuth token expired | Re-authenticate di n8n credentials |
| `40105` dari TikTok | Access token tidak valid | Generate ulang access token |
| Stage check tidak match | Nama stage berbeda | Sesuaikan nama stage di node "Cek Stage" |
| Contact tidak ditemukan | Lead tidak punya kontak | Pastikan kontak terhubung ke lead di Kommo |
