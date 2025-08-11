# 🛡️ Fictitious App - Secrets Management Lab (Local + CI/CD Concepts)

## 📌 Overview
This lab simulates a **full CI/CD pipeline** for a fictitious application with **secrets management**, using **only open-source tools**.  
It is designed for **self-learning** or **teaching purposes** in DevSecOps, Secrets Management, and Application Security.

The lab includes:
- **FastAPI** application
- **PostgreSQL** database
- **HashiCorp Vault** for secrets management
- **SOPS + age** for encrypted config files
- **Gitleaks** for secrets detection
- **Pre-commit hooks**
- **Docker Compose** for local orchestration

---

## 🧰 Tools & Technologies
- **FastAPI** (Python 3.12)
- **PostgreSQL 16**
- **HashiCorp Vault OSS** (KV v2 engine)
- **SOPS + age** (optional encrypted config)
- **Gitleaks** (secrets detection)
- **Docker + Docker Compose**
- **Pre-commit**

---

## 📋 Requirements
- [Docker](https://docs.docker.com/get-docker/) + Docker Compose
- [Python 3.12+](https://www.python.org/downloads/)
- [Git](https://git-scm.com/downloads)
- Optional: [age](https://github.com/FiloSottile/age) and [SOPS](https://github.com/getsops/sops)

---

## 🏗️ 1. Lab Setup Scripts

You can create the lab in **two ways**:

---

### **A) Script: Empty Structure (`create_lab_structure.sh`)**
Use this if you want **only the folder structure and empty files**.

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT_NAME="fictitious-app"

echo "[+] Creating project structure: $PROJECT_NAME"
mkdir -p $PROJECT_NAME/{app/tests,ops/{vault,scripts},.github/workflows,secrets}

# Basic files
cat > $PROJECT_NAME/app/main.py <<'EOF'
# FastAPI app placeholder
EOF

cat > $PROJECT_NAME/app/requirements.txt <<'EOF'
# Python dependencies placeholder
EOF

touch $PROJECT_NAME/app/tests/__init__.py
touch $PROJECT_NAME/ops/docker-compose.dev.yml
touch $PROJECT_NAME/ops/docker-compose.staging.yml
touch $PROJECT_NAME/ops/vault/config.hcl
touch $PROJECT_NAME/ops/vault/policies.hcl
touch $PROJECT_NAME/ops/scripts/init_vault.sh
touch $PROJECT_NAME/ops/scripts/render_env.py
touch $PROJECT_NAME/.github/workflows/ci.yml
touch $PROJECT_NAME/.github/workflows/cd.yml
touch $PROJECT_NAME/.gitleaks.toml
touch $PROJECT_NAME/.pre-commit-config.yaml
touch $PROJECT_NAME/.sops.yaml
touch $PROJECT_NAME/secrets/app.enc.yaml
touch $PROJECT_NAME/secrets/README.md
touch $PROJECT_NAME/README.md

echo "[✓] Empty structure created."
```
Run:
```bash
chmod +x create_lab_structure.sh
./create_lab_structure.sh
```

### **B) Script: Fully Populated Lab (`create_local_lab.sh`)**
Use this if you want the ready-to-run local lab with FastAPI, Vault, PostgreSQL, secrets, and all configs populated.

```bash
#!/usr/bin/env bash
set -euo pipefail

PROJECT_NAME="fictitious-app"

echo "[+] Creating local lab: $PROJECT_NAME"
mkdir -p $PROJECT_NAME/{app/tests,ops/{vault,scripts},.github/workflows,secrets}

# --- FastAPI App ---
cat > $PROJECT_NAME/app/main.py <<'EOF'
from fastapi import FastAPI
import os
import psycopg2

app = FastAPI()

@app.get("/health")
def health():
    return {"ok": True}

@app.get("/info")
def info():
    return {
        "env": os.getenv("APP_ENV", "dev"),
        "db_host": os.getenv("DB_HOST"),
        "uses_vault": True
    }

@app.get("/db-check")
def db_check():
    conn = psycopg2.connect(
        host=os.getenv("DB_HOST","db"),
        user=os.getenv("DB_USER","app"),
        password=os.getenv("DB_PASSWORD","app"),
        dbname=os.getenv("DB_NAME","appdb"),
        port=int(os.getenv("DB_PORT","5432"))
    )
    with conn.cursor() as cur:
        cur.execute("SELECT 1;")
        row = cur.fetchone()
    conn.close()
    return {"db_ok": row == 1}
EOF

cat > $PROJECT_NAME/app/requirements.txt <<'EOF'
fastapi==0.111.0
uvicorn==0.30.0
psycopg2-binary==2.9.9
EOF
touch $PROJECT_NAME/app/tests/__init__.py

# --- Dockerfile ---
cat > $PROJECT_NAME/Dockerfile <<'EOF'
FROM python:3.12-slim
WORKDIR /app
COPY app/requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app/ /app/app
EXPOSE 8000
CMD ["uvicorn","app.main:app","--host","0.0.0.0","--port","8000"]
EOF

# --- docker-compose.dev.yml ---
cat > $PROJECT_NAME/ops/docker-compose.dev.yml <<'EOF'
version: "3.8"

services:
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: appdb
      POSTGRES_USER: app
      POSTGRES_PASSWORD: app
    ports:
      - "5432:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d appdb"]
      interval: 5s
      timeout: 5s
      retries: 10

  api:
    build:
      context: ..
      dockerfile: Dockerfile
    environment:
      APP_ENV: dev
      DB_HOST: db
      DB_PORT: "5432"
      DB_NAME: appdb
      DB_USER: app
      DB_PASSWORD: app
    depends_on:
      db:
        condition: service_healthy
    ports:
      - "8000:8000"
    command: ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000", "--reload"]

  vault:
    image: hashicorp/vault:1.16
    container_name: vault
    cap_add:
      - IPC_LOCK
    environment:
      VAULT_DEV_ROOT_TOKEN_ID: root
      VAULT_DEV_LISTEN_ADDRESS: "0.0.0.0:8200"
    ports:
      - "8200:8200"
    entrypoint: /bin/sh
    command: -c "
      vault server -dev -dev-root-token-id=root -dev-listen-address=0.0.0.0:8200 &
      sleep 5 &&
      export VAULT_ADDR=http://127.0.0.1:8200 &&
      export VAULT_TOKEN=root &&
      vault secrets enable -path=secret kv-v2 || true &&
      vault kv put secret/app/db DB_HOST='db' DB_PORT='5432' DB_NAME='appdb' DB_USER='app' DB_PASSWORD='superS3cret!' &&
      vault kv put secret/app/jwt JWT_SECRET='dev-jwt-very-secret' &&
      echo 'path \"secret/data/app/*\" { capabilities = [\"read\"] }' > /tmp/app-read.hcl &&
      vault policy write app-read /tmp/app-read.hcl &&
      vault auth enable approle || true &&
      ROLE_ID=$(vault write -f -format=json auth/approle/role/app-role policies='app-read' token_ttl='30m' token_max_ttl='2h' secret_id_ttl='30m' | jq -r '.data.role_id') &&
      SECRET_ID=$(vault write -f -format=json auth/approle/role/app-role/secret-id | jq -r '.data.secret_id') &&
      echo 'ROLE_ID=' $ROLE_ID &&
      echo 'SECRET_ID=' $SECRET_ID &&
      tail -f /dev/null
    "
EOF

# --- docker-compose.staging.yml ---
cat > $PROJECT_NAME/ops/docker-compose.staging.yml <<'EOF'
version: "3.8"
services:
  api:
    image: fictitious-app:latest
    env_file:
      - ../.env.staging
    ports: [ "8080:8000" ]
    depends_on:
      db:
        condition: service_healthy
  db:
    image: postgres:16
    environment:
      POSTGRES_DB: ${DB_NAME:-appdb}
      POSTGRES_USER: ${DB_USER:-app}
      POSTGRES_PASSWORD: ${DB_PASSWORD:-app}
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-app} -d ${DB_NAME:-appdb}"]
      interval: 5s
      timeout: 5s
      retries: 10
EOF

# --- render_env.py ---
cat > $PROJECT_NAME/ops/scripts/render_env.py <<'EOF'
import os

secrets = {
    "APP_ENV": os.getenv("APP_ENV", "staging"),
    "DB_HOST": os.getenv("DB_HOST"),
    "DB_PORT": os.getenv("DB_PORT"),
    "DB_NAME": os.getenv("DB_NAME"),
    "DB_USER": os.getenv("DB_USER"),
    "DB_PASSWORD": os.getenv("DB_PASSWORD"),
    "JWT_SECRET": os.getenv("JWT_SECRET")
}

with open(".env.staging", "w") as f:
    for k, v in secrets.items():
        f.write(f"{k}={v}\n")

print(".env.staging file created")
EOF

# --- Configs ---
cat > $PROJECT_NAME/.gitleaks.toml <<'EOF'
title = "gitleaks config"
[allowlist]
description = "allow local dev passwords"
regexes = [
  '''postgres://app:app@db:5432/appdb''',
]
EOF

cat > $PROJECT_NAME/.pre-commit-config.yaml <<'EOF'
repos:
  - repo: https://github.com/gitleaks/gitleaks
    rev: v8.18.2
    hooks:
      - id: gitleaks
        args: ["protect", "--staged", "--redact"]
EOF

cat > $PROJECT_NAME/secrets/app.enc.yaml <<'EOF'
data:
  feature_flag_x: "on"
  analytics_sample_rate: "0.2"
EOF

cat > $PROJECT_NAME/secrets/README.md <<'EOF'
# Secrets folder
Contains files encrypted with SOPS. No plain-text secrets should be stored here.
EOF

cat > $PROJECT_NAME/README.md <<'EOF'
# Fictitious App Lab - Local
Run:
docker compose -f ops/docker-compose.dev.yml up -d
EOF

echo "[✓] Local lab created successfully."
```
Run:
```bash
chmod +x create_local_lab.sh
./create_local_lab.sh
```

---
## 🚀 2. Running the Lab
After running `create_local_lab.sh`:
```bash
cd fictitious-app
docker compose -f ops/docker-compose.dev.yml up -d
```
API → http://127.0.0.1:8000

Vault → http://127.0.0.1:8200 (token: `root`)

---
## 🧪 3. Testing
```bash
curl http://127.0.0.1:8000/health
```
Expected:
```json
{"ok": true}
```

---
## 🎓 4. Classroom Exercises
- Rotate secrets in Vault and redeploy
- Commit a fake `.env` file and detect it with Gitleaks
- Create a new secret in Vault and use it in the API
- Restrict Vault policies and see failure
- Add SOPS encryption for configs


