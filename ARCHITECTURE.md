# CloudVault — Architecture

## Overview

CloudVault is a self-hosted cloud storage platform running on a Framework Desktop
with external HDD storage via a UGREEN dual-bay USB dock. It provides authenticated
file storage with per-user quotas, AI-powered photo tagging, and natural language
search. Accessible externally via Cloudflare Tunnel.

## System Diagram

```
Internet
    ↓
Cloudflare Tunnel (cloudflared)
    ↓
Caddy (reverse proxy, localhost:443)
    ↓
┌─────────────────────────────────────────┐
│  FastAPI (localhost:8000)                │
│  ├── /api/auth/*      Auth + invite     │
│  ├── /api/files/*     Upload/download   │
│  ├── /api/search/*    Smart search      │
│  ├── /api/tags/*      AI tagging        │
│  └── /api/admin/*     Quota management  │
└─────────────────────────────────────────┘
    ↓               ↓               ↓
 SQLite          Storage         AI Workers
 (metadata)     /mnt/storage/    (background)
                ├── 1tb/
                └── 8tb/
```

## Tech Stack

| Layer        | Technology                     |
|-------------|--------------------------------|
| Frontend    | React 18 + Vite + Tailwind CSS |
| Backend     | FastAPI (Python 3.12+)         |
| Database    | SQLite (via SQLAlchemy + alembic) |
| Auth        | JWT (access + refresh tokens)  |
| File store  | Direct filesystem (ext4)       |
| AI/ML       | CLIP (tagging), sentence-transformers (search) |
| Thumbnails  | Pillow (images), ffmpeg (video)|
| Proxy       | Caddy                          |
| Tunnel      | cloudflared                    |
| Task queue  | Background tasks (FastAPI) → arq later |

## Storage Layout

```
/mnt/storage/
├── 1tb/                          # Toshiba 1TB
│   └── users/
│       └── {user_id}/
│           ├── files/            # Original uploads
│           └── thumbnails/       # Generated previews
└── 8tb/                          # WD Red Plus 8TB (when ready)
    └── users/
        └── {user_id}/
            ├── files/
            └── thumbnails/
```

Files are stored as `{uuid}.{ext}` on disk. The SQLite database maps
UUIDs to original filenames, user ownership, tags, and metadata.

## Auth Flow

1. Admin (you) generates invite codes via CLI or admin panel
2. User visits /register, enters invite code + creates username/password
3. Password hashed with bcrypt, stored in SQLite
4. Login returns JWT access token (15min) + refresh token (7d)
5. All API requests include Bearer token
6. /request-access page lets people request an invite from you

## AI Pipeline

Runs as background tasks after file upload:

1. **Upload completes** → file metadata written to DB
2. **Thumbnail generation** → Pillow for images, ffmpeg for video frames
3. **CLIP embedding** → generate 512-dim vector from image/thumbnail
4. **Auto-tagging** → classify against label set (objects, scenes, etc.)
5. **Embedding stored** → SQLite (or numpy mmap for perf later)
6. **Smart search** → user query → CLIP text encoder → cosine similarity

## Per-User Quotas

- Each user has a `quota_bytes` field (set by admin)
- `used_bytes` tracked and updated on upload/delete
- Upload rejected if `used_bytes + file_size > quota_bytes`
- Admin can view and adjust quotas via dashboard

## API Routes

```
POST   /api/auth/register          # Redeem invite code + create account
POST   /api/auth/login             # Get JWT tokens
POST   /api/auth/refresh           # Refresh access token
POST   /api/auth/request-access    # Request an invite

GET    /api/files/                  # List user's files (paginated)
POST   /api/files/upload           # Upload file(s)
GET    /api/files/{id}             # File metadata
GET    /api/files/{id}/download    # Download file
GET    /api/files/{id}/thumbnail   # Get thumbnail
DELETE /api/files/{id}             # Delete file

GET    /api/search?q=...           # Natural language search
GET    /api/tags/                  # List all tags for user
GET    /api/tags/{tag}/files       # Files with specific tag

GET    /api/admin/users            # List users + quota usage
PUT    /api/admin/users/{id}/quota # Update user quota
POST   /api/admin/invite-codes    # Generate invite codes
GET    /api/admin/invite-codes    # List active codes
```