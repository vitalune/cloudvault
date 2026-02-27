# CloudVault

Self-hosted cloud storage with AI-powered tagging and search.

## Quick Start

### 1. Mount your drives

```bash
# Create mount points
sudo mkdir -p /mnt/storage/{1tb,8tb}

# Get UUIDs
blkid /dev/sda1 /dev/sdb1

# Add to /etc/fstab (use your actual UUIDs)
# UUID=xxxx  /mnt/storage/1tb  ext4  defaults,nofail  0  2
# UUID=xxxx  /mnt/storage/8tb  ext4  defaults,nofail  0  2

sudo mount -a
```

### 2. Backend setup

```bash
cd backend

# Create virtualenv
python -m venv .venv
source .venv/bin/activate

# Install dependencies
pip install -r requirements.txt

# Configure
cp .env.example .env
# Edit .env — at minimum, change CLOUDVAULT_SECRET_KEY

# Initialize database and create admin account
python -m scripts.bootstrap setup

# Run the API server
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

### 3. Frontend setup

```bash
cd frontend

npm install
npm run dev
```

### 4. Cloudflare Tunnel (external access)

```bash
# Install cloudflared (Arch)
pacman -S cloudflared

# Authenticate
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create cloudvault

# Configure tunnel to point to Caddy
# ~/.cloudflared/config.yml:
#   tunnel: <tunnel-id>
#   credentials-file: ~/.cloudflared/<tunnel-id>.json
#   ingress:
#     - hostname: vault.yourdomain.com
#       service: http://localhost:8080
#     - service: http_status:404

# Add DNS record
cloudflared tunnel route dns cloudvault vault.yourdomain.com

# Run
cloudflared tunnel run cloudvault
```

### 5. Caddy (reverse proxy)

```bash
caddy run --config config/Caddyfile
```

## CLI Commands

```bash
# Create admin account
python -m scripts.bootstrap setup

# Generate invite code
python -m scripts.bootstrap invite

# Generate invite with custom quota
python -m scripts.bootstrap invite --quota 10GB
```

## Architecture

See [ARCHITECTURE.md](./ARCHITECTURE.md) for full system design.