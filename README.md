# Setup LiveKit Server di LAN dengan Self-Signed Certificate

Panduan lengkap untuk menjalankan LiveKit Server di jaringan LAN menggunakan Docker, dengan TLS self-signed certificate untuk koneksi WebSocket Secure (WSS).

## 📋 Prasyarat

- OS: Ubuntu 22.04 (atau setara)
- Docker Engine + Docker Compose v2
- Firewall dapat dikonfigurasi (UFW/iptables)
- IP server tetap (contoh: `172.15.1.156`)
- Akses shell root/sudo

## 🚀 Instalasi

### 1. Pastikan Snapd Aktif

Snapd umumnya sudah terinstall dan aktif di Ubuntu 22.04:

```bash
sudo systemctl enable --now snapd
```

### 2. Install Docker dari Snap

```bash
sudo snap install docker
```

Snap akan menginstall:
- Docker Engine
- Docker Compose v2
- Docker daemon (`dockerd`)

### 3. Verifikasi Instalasi

Cek status service Docker:

```bash
snap services docker
```

Output yang diharapkan:
```
Service               Startup  Current  Notes
docker.dockerd        enabled  active   -
```

### 4. Verifikasi Versi

```bash
docker --version
docker compose version
```

Output contoh:
```
Docker version 24.0.x, build xxxxxxx
Docker Compose version v2.x.x
```

## 📁 1. Struktur Direktori

Buat direktori untuk LiveKit menggunakan path aman untuk Docker Snap:

```bash
sudo mkdir -p /var/snap/docker/common/livekit/certs
cd /var/snap/docker/common/livekit
```

## 🔐 2. Generate Self-Signed Certificate

**Ganti IP di bawah dengan IP server LAN Anda** (wajib cocok dengan yang akan diakses klien):

```bash
cd /var/snap/docker/common/livekit/certs
sudo openssl req -x509 -newkey rsa:4096 -sha256 -days 365 -nodes \
  -keyout livekit.key -out livekit.crt \
  -subj "/CN=172.15.1.156" \
  -addext "subjectAltName=IP:172.15.1.156"
```

Verifikasi SAN berisi IP yang benar:

```bash
openssl x509 -in livekit.crt -noout -subject -ext subjectAltName
```

## ⚙️ 3. File Konfigurasi

### 3.1 livekit.yaml

Buat file `/var/snap/docker/common/livekit/livekit.yaml`:

```yaml
port: 7880
log_level: info

keys:
  APIFMFFtgcCFtjV: YXDalHCztxmao2JawfdDPthWEpByoso5yts3y5GVH3H

# aktifkan hanya jika menjalankan service "redis" di compose
redis:
  address: 127.0.0.1:6379

rtc:
  port_range_start: 50000
  port_range_end: 60000
  tcp_port: 7881
  use_external_ip: false
```

### 3.2 Caddyfile

Buat file `/var/snap/docker/common/livekit/Caddyfile`:

```
https://172.15.1.156:443 {
  tls /livekit_certs/livekit.crt /livekit_certs/livekit.key

  @ws {
    header Connection *Upgrade*
    header Upgrade websocket
  }

  reverse_proxy @ws 127.0.0.1:7880
  reverse_proxy /  127.0.0.1:7880
}
```

> **Catatan**: Versi minimal tanpa `@ws` juga bisa digunakan: `reverse_proxy 127.0.0.1:7880` — Caddy otomatis handle WebSocket.

### 3.3 docker-compose.yml

Buat file `/var/snap/docker/common/livekit/docker-compose.yml`:

```yaml
services:
  livekit:
    image: livekit/livekit-server:latest
    command: ["--config", "/livekit/livekit.yaml"]
    network_mode: "host"
    restart: unless-stopped
    volumes:
      - ./livekit.yaml:/livekit/livekit.yaml:ro

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    network_mode: "host"

  caddy:
    image: caddy:2
    network_mode: "host"
    restart: unless-stopped
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./certs:/livekit_certs:ro
      - caddy_data:/data
      - caddy_config:/config

volumes:
  caddy_data:
  caddy_config:
```

> **Catatan**: Jika tidak ingin menggunakan Redis, hapus service `redis` dan blok `redis:` di `livekit.yaml`.

## 🔥 4. Konfigurasi Firewall

Buka port yang diperlukan:

```bash
sudo ufw allow 443/tcp           # Caddy WSS
sudo ufw allow 7880/tcp          # LiveKit HTTP/WS
sudo ufw allow 7881/tcp          # LiveKit TCP fallback
sudo ufw allow 50000:60000/udp   # WebRTC media
sudo ufw status
```

## 🚀 5. Jalankan Stack

```bash
cd /var/snap/docker/common/livekit
sudo docker compose down
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100 livekit
sudo docker compose logs --tail=100 caddy
```

**Verifikasi**:
- LiveKit: `starting LiveKit server {"portHttp": 7880, ...}`
- Caddy: berjalan tanpa error membaca certificate

## 🔑 6. Token Server (Python)

### 6.1 Environment Variables

Buat file `/var/snap/docker/common/livekit/development.env`:

```bash
export LIVEKIT_API_KEY=APIFMFFtgcCFtjV
export LIVEKIT_API_SECRET=YXDalHCztxmao2JawfdDPthWEpByoso5yts3y5GVH3H
```

Load environment:

```bash
cd /var/snap/docker/common/livekit
source development.env
```

### 6.2 Token Server (Flask)

Buat file `/var/snap/docker/common/livekit/server.py`:

```python
import os
import datetime
from livekit import api
from flask import Flask, request, jsonify

app = Flask(__name__)
API_KEY = os.getenv("LIVEKIT_API_KEY")
API_SECRET = os.getenv("LIVEKIT_API_SECRET")

@app.route("/getToken")
def get_token():
    room = request.args.get("room", "demo")
    identity = request.args.get("user", "user")
    ttl_sec = int(request.args.get("ttl", "600"))
    ttl = datetime.timedelta(seconds=ttl_sec)

    token = (
        api.AccessToken(API_KEY, API_SECRET)
        .with_identity(identity)
        .with_grants(api.VideoGrants(
            room_join=True,
            room=room,
            can_publish=True,
            can_subscribe=True,
        ))
        .with_ttl(ttl)
    )
    return jsonify({"token": token.to_jwt()})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=3000)
```

### 6.3 Install Dependencies

```bash
pip install livekit-api Flask
```

### 6.4 Jalankan Token Server

```bash
source development.env
python3 /var/snap/docker/common/livekit/server.py
```

**Uji dari klien LAN**:

```bash
curl "http://172.15.1.156:3000/getToken?room=demo&user=alice&ttl=600"
# Output: {"token":"<JWT>"}
```


## 🔧 7. Systemd Service (Auto-Start saat Boot)

### 7.1 Unit untuk Docker Compose Stack (LiveKit + Caddy + Redis)

Buat file `/etc/systemd/system/livekit-stack.service`:

```ini
[Unit]
Description=LiveKit + Caddy (+ Redis) via Docker Compose
Requires=network-online.target
Wants=network-online.target docker.service snap.docker.dockerd.service
After=network-online.target docker.service snap.docker.dockerd.service

[Service]
Type=oneshot
# Direktori kerja tempat docker-compose.yml berada
WorkingDirectory=/var/snap/docker/common/livekit
# Pastikan compose membaca file yang benar
Environment=COMPOSE_FILE=/var/snap/docker/common/livekit/docker-compose.yml
# Start stack (detached)
ExecStart=/usr/bin/docker compose up -d
# Stop stack
ExecStop=/usr/bin/docker compose down
# Agar unit dianggap "aktif" setelah ExecStart selesai
RemainAfterExit=yes
TimeoutStartSec=0

# (Opsional) kalau ingin bisa reload untuk apply perubahan minor:
# ExecReload=/usr/bin/docker compose up -d

[Install]
WantedBy=multi-user.target
```

**Catatan:**
- `Wants/After` menyebut `docker.service` dan `snap.docker.dockerd.service` supaya aman untuk instalasi Docker biasa maupun Snap. Bila salah satu tidak ada, systemd akan tetap jalan (hanya warning).
- `RemainAfterExit=yes` dipakai karena ini oneshot yang hanya menjalankan `docker compose up -d` lalu selesai, tapi kontainer tetap berjalan di background.

### 7.2 Unit untuk Token Server (Flask)

Buat file `/etc/systemd/system/livekit-token.service`:

```ini
[Unit]
Description=LiveKit Token Server (Flask)
After=network-online.target
Wants=network-online.target

[Service]
# Ganti 'me' dengan user Linux kamu (yang biasa pakai pip/virtualenv). 
# Bisa juga dipakai 'root' jika perlu (tidak direkomendasikan).
User=me
Group=me
WorkingDirectory=/var/snap/docker/common/livekit
EnvironmentFile=/var/snap/docker/common/livekit/development.env
ExecStart=/usr/bin/python3 /var/snap/docker/common/livekit/server.py
Restart=always
RestartSec=3
# Logging ke journal
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

**Buka port 3000** jika ingin klien LAN lain bisa akses token server:

```bash
sudo ufw allow 3000/tcp
```

### 7.3 Daftarkan & Jalankan Service

```bash
# Reload systemd setelah membuat file unit
sudo systemctl daemon-reload

# Aktifkan saat boot + langsung jalankan sekarang
sudo systemctl enable --now livekit-stack.service
sudo systemctl enable --now livekit-token.service

# Restart service
sudo systemctl restart livekit-stack.service
sudo systemctl restart livekit-token.service

```