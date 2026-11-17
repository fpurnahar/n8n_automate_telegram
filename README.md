# Telegram AI Agent dengan n8n, Gemini, dan Memory

Project ini menjalankan bot Telegram berbasis AI melalui n8n lokal. Pesan Telegram diterima oleh `Telegram Trigger`, diproses oleh Google Gemini dengan memory per chat, lalu dikirim kembali ke pengguna Telegram.

## Alur

```text
Telegram → Cloudflare Quick Tunnel → n8n → Gemini → n8n → Telegram
```

Workflow utama: `workflows/telegram-ai-gemini-memory-n8n.json`.

## Prasyarat

- Docker dan Docker Compose.
- Bot Telegram dari BotFather beserta token barunya.
- Gemini API key dari Google AI Studio.
- Koneksi internet untuk Telegram, Cloudflare Tunnel, dan Gemini API.

## Menjalankan n8n dan tunnel

1. Salin `.env.example` menjadi `.env`, lalu ganti seluruh placeholder secret.
2. Jalankan n8n dan Cloudflare Quick Tunnel:

```powershell
docker-compose -f docker-compose-n8n.yml up -d n8n cloudflared
docker-compose -f docker-compose-n8n.yml logs -f cloudflared
```

3. Salin URL `https://....trycloudflare.com` dari log `cloudflared` ke `.env`:

```env
WEBHOOK_URL=https://URL-ASLI.trycloudflare.com/
```

4. Recreate hanya n8n agar tunnel tidak memperoleh URL baru:

```powershell
docker-compose -f docker-compose-n8n.yml up -d --force-recreate n8n
```

5. Buka n8n melalui URL `trycloudflare.com` tersebut.

Quick Tunnel tidak memerlukan akun Cloudflare, tetapi URL-nya berubah ketika container `cloudflared` dibuat ulang. Jika berubah, perbarui `WEBHOOK_URL`, recreate n8n, dan aktifkan ulang workflow.

## Impor dan konfigurasi workflow

1. Di n8n, pilih **Import from File** dan impor `workflows/telegram-ai-gemini-memory-n8n.json`.
2. Pada `Telegram Trigger`, buat atau pilih credential **Telegram Bot API**.
3. Gunakan credential Telegram yang sama pada `Send Telegram Reply`.
4. Pada `Google Gemini Chat Model`, buat credential **Google Gemini (PaLM) API** dan masukkan Gemini API key.
5. Pastikan model yang dipilih adalah `gemini-2.5-flash` atau model Gemini yang tersedia pada akun.
6. Simpan lalu **Publish/Activate** workflow.
7. Kirim pesan pribadi ke bot Telegram.

Credential sengaja tidak disimpan dalam workflow JSON. Jangan menaruh token Telegram atau Gemini API key di repository.

## Test dan production webhook

- Workflow aktif memakai path `/webhook/...`.
- Workflow nonaktif yang sedang menunggu test event memakai `/webhook-test/...`.
- Telegram Trigger tidak dapat mendengarkan test dan production secara bersamaan. Unpublish workflow sebelum memakai **Execute workflow** dalam test mode.

Untuk pengujian normal, biarkan workflow aktif dan kirim pesan langsung ke bot Telegram.

## Diagnostik

```powershell
docker-compose -f docker-compose-n8n.yml ps
docker-compose -f docker-compose-n8n.yml logs -f n8n
docker-compose -f docker-compose-n8n.yml logs -f cloudflared
```

Periksa hal berikut jika bot tidak merespons:

- `WEBHOOK_URL` sama dengan URL tunnel yang sedang aktif.
- Workflow sudah Published/Active.
- Trigger dan reply memakai credential bot Telegram yang sama.
- Credential Gemini valid dan model tersedia.
- Pesan dikirim melalui private chat dan berisi teks.
- Execution n8n tidak menunjukkan error quota, credential, atau koneksi.

## Service WhatsApp lama

`docker-compose-n8n.yml` masih memuat Evolution API, PostgreSQL, dan Redis untuk workflow WhatsApp lama. Service tersebut tidak diperlukan oleh bot Telegram dan tidak ikut dijalankan jika perintah hanya menyebut `n8n cloudflared`.

## Keamanan

- Jangan commit `.env` atau credential API.
- Revoke token yang pernah terlihat di chat, log, atau repository.
- Quick Tunnel cocok untuk pengembangan, bukan endpoint production permanen.
- Jangan menjalankan `docker-compose down -v` jika tidak ingin menghapus volume/data lokal.
