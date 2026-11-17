# WhatsApp → Evolution API → n8n AI Agent + OpenAI + Memory → WhatsApp

Workflow: `workflows/wa-ai-chatgpt-memory.json`. Kompatibel dengan stack Docker Compose sebelumnya (`docker-compose.yml`), menggunakan instance Evolution API `bot-local`.

## 1. Jalankan service

```powershell
docker compose up -d
docker compose ps
```

Buka `http://localhost:5678` untuk n8n dan `http://localhost:8080/manager` untuk Evolution Manager. Pair instance `bot-local` via QR. PostgreSQL dan Redis dalam stack ini untuk Evolution API. n8n menggunakan volume lokal dan database default SQLite.

## 2. Import workflow dan atur credentials

- n8n → New workflow → menu (tiga titik) → Import from File → pilih `workflows/wa-ai-chatgpt-memory.json`.
- Klik **OpenAI Chat Model** → **Credential for OpenAI** → Create new → isi API key dari `https://platform.openai.com/api-keys`. Jangan masukkan token langsung ke JSON workflow. Pastikan akun API memiliki billing/credit; ChatGPT Plus tidak termasuk API usage.
- Klik **Send WhatsApp Reply** → **Credential for Header Auth** → Create new. Isi **Name** = `apikey` dan **Value** = nilai `EVOLUTION_API_KEY` dari `.env` (jangan dibagikan). Ini sengaja menggunakan credential n8n dan **tidak** membaca `$env` untuk menghindari pembatasan environment variable di n8n terbaru.
- Di node **OpenAI Chat Model**, pilih model yang tersedia pada akun jika `gpt-4.1-mini` tidak muncul. Pada versi n8n lain, nama field atau node type version bisa berubah; pastikan koneksi AI Agent model dan memory terpasang.
- Klik Save lalu Publish/Activate workflow.

## 3. Konfigurasi webhook instance

**URL internal dari Evolution container:** `http://n8n:5678/webhook/wa-ai-chatgpt-local`

Event: `MESSAGES_UPSERT`; enabled `true`; Webhook by Events `false`.

Via Evolution Manager: pilih instance `bot-local` → webhook → isi URL dan event di atas.

Alternatif menggunakan PowerShell di folder ini:

```powershell
$key = ((Get-Content .env | Where-Object { $_ -like 'EVOLUTION_API_KEY=*' }) -split '=', 2)[1]
$headers = @{ apikey = $key }
$body = @{ webhook = @{ enabled = $true; url = 'http://n8n:5678/webhook/wa-ai-chatgpt-local'; webhookByEvents = $false; events = @('MESSAGES_UPSERT') } } | ConvertTo-Json -Depth 6
Invoke-RestMethod -Method POST -Uri 'http://localhost:8080/webhook/set/bot-local' -Headers $headers -ContentType 'application/json' -Body $body
```

Jika path berbeda pada image Evolution yang terpasang, ikuti endpoint yang tercantum di API/Manager versi tersebut.

## 4. Uji

1. Kirim `Halo, kamu bisa bantu apa?` dari nomor WhatsApp **lain** ke nomor yang sudah dipairing.
2. Periksa n8n → Executions. Alur: Webhook → Normalize Incoming Message → AI Agent → Send WhatsApp Reply.
3. Kirim pesan lanjutan dari nomor yang sama: `Terus apa bedanya dengan chatbot biasa?` Memory menggunakan `bot-local:nomor@...` sebagai session key.
4. Jika AI Agent tidak menghasilkan balasan, lihat error pada OpenAI credential, kredit, akses model, atau log.

## Troubleshooting

```powershell
docker compose logs -f n8n
docker compose logs -f evolution-api
```

- Webhook tidak terpanggil: pastikan workflow telah Publish/Activate dan Evolution webhook menunjuk **production** `/webhook/...` bukan `/webhook-test/...`.
- Error 401 saat mengirim pesan: credential Header Auth perlu `Name=apikey`, Value persis dengan `EVOLUTION_API_KEY`.
- `@lid` WhatsApp: beberapa pesan memakai privacy JID. Workflow ini hanya membalas jika ada JID normal di `remoteJid` atau `sender`; yang lain dilewati untuk menghindari salah tujuan. Jika perlu, kembangkan resolusi LID → nomor berdasarkan fitur Evolution API dan uji dengan contoh webhook asli.
- Pesan dari bot sendiri (`fromMe=true`), grup, dan pesan tanpa teks dibuang.
- Simple Memory adalah memory demo, **tidak menjamin persistensi lintas restart atau pada queue/multi-worker**; produksi sebaiknya gunakan Redis Chat Memory atau Postgres Chat Memory yang terkonfigurasi khusus di n8n.
- Tidak ada penyimpanan deduplikasi event: jika Evolution melakukan retry pada event yang sama, bot berpotensi membalas dua kali. Untuk produksi tambahkan deduplikasi berdasarkan `messageId`, rate limiting, dan validasi webhook.
- Model OpenAI memerlukan koneksi internet **keluar** dari container n8n ke API OpenAI. Local-only di sini berarti WhatsApp gateway dan webhook berjalan dalam Docker network, bukan AI-nya offline.

## Keamanan

Evolution Baileys bukan API WhatsApp resmi. Gunakan nomor uji, jangan untuk spam, dan jangan ekspos port tanpa autentikasi/HTTPS. Credential tidak dibundel dalam workflow karena harus diisi melalui n8n UI. Jaga isi `.env` tetap privat.
