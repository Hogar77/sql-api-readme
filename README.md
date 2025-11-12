# sql-api

A REST API written in NestJS + TypeScript for managing business entities (auth, users, CRUD, etc.). This repository contains the backend service code, endpoint documentation and instructions for local running and deployment.

## Basics (short)

- Technologies: NestJS, TypeScript, Node.js
- DB: PostgreSQL (or MS SQL if needed)
- Authentication: JWT
- API documentation: Swagger / OpenAPI
- Deploy: Docker Compose, Nginx as reverse proxy, Certbot for TLS

## Repository structure

- `/src` — application TypeScript source code (controllers, services, modules)
- `/sql` or `/sql-scripts` — SQL scripts / migrations / stored procedures (if present)
- `docker/` — example Docker Compose and Dockerfile for local development and deployment
- `.env.example` — example environment variables
- `README.md` — this file

## Quick start — local (developer)

1. Clone the repository:

```bash
git clone https://github.com/Hogar77/sql-api.git
cd sql-api
```

2. Create a `.env` file from `.env.example`:

```bash
cp .env.example .env
# Edit .env and fill in real values (do not commit .env)
```

Example `.env.example`:

```env
# App
PORT=4000
NODE_ENV=development

# Frontend / public config (if you use Next.js or similar)
NEXT_PUBLIC_API_URL=http://localhost:4000
NEXT_PUBLIC_BASE_PATH=
NEXT_PUBLIC_ROLE=user
NEXT_PUBLIC_ADMIN_ROLE=admin
ADMIN_ROLE=admin
DEFAULT_USER_ROLE=user

# Database (MSSQL in the docker-compose.yml example)
# If you use Postgres replace the image/port and POSTGRES_... variables
DATABASE_HOST=sql
DATABASE_PORT=1433
DATABASE_USERNAME=sa
DATABASE_PASSWORD=change_me_password
DATABASE_NAME=sql_api_db

# (Optional for Postgres docker image)
POSTGRES_DB=sql_api_db
POSTGRES_USER=postgres
POSTGRES_PASSWORD=change_me_password

# JWT / auth
JWT_SECRET=change_this_to_a_strong_secret
JWT_EXPIRATION=3600   # in seconds or a string like "1h"

# SMTP (email)
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_SECURE=false     # true for 465
SMTP_USER=smtp_user@example.com
SMTP_PASS=smtp_password
SMTP_CODE_EXPIRATION=300000   # in ms (e.g. 5 minutes)

# Role / admin (example)
ROLE_ADMIN=admin_role_name

# Seller / invoicing (optional)
SELLER_NAME="My Company DOO"
SELLER_ADDRESS="Street 1"
SELLER_CITY=Belgrade
SELLER_PIB=123456789
SELLER_MAT_BROJ=12345678
SELLER_BANK_ACCOUNT=RS35123456789000000000
SELLER_EMAIL=office@example.com
SELLER_PHONE="+381600000000"

# S3 / MinIO
S3_REGION=us-east-1
S3_BUCKET=my-bucket
S3_ENDPOINT=http://minio:9000
AWS_ACCESS_KEY_ID=minio_access_key
AWS_SECRET_ACCESS_KEY=minio_secret_key
S3_FORCE_PATH_STYLE=true
S3_CDN_URL=

# MinIO root (if you run the minio service)
MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=minio_password

# Certbot / nginx (example contact email used in docker-compose certbot service)
# Leave empty or set your contact e‑mail (do not put secrets)
CERTBOT_EMAIL=safi@example.com

# Additional (if needed)
# REDIS_URL=redis://redis:6379
# OTHER_SERVICE_URL=
```

3. Install dependencies and run:

```bash
npm install
npm run build
npm run start:dev    # or `npm run start` for production (or `npm run start:prod`)
```

By default the API will be available at: `http://localhost:3000` (or the PORT from `.env`).

## Running with Docker Compose

There is an example `docker-compose.yml` in the `docker/` directory. Basic example:

```yaml
version: "3.9"

services:
  api:
    build: .
    ports:
      - "4000:4000"
    env_file:
      - .env
    environment:
      - DATABASE_HOST=sql
      - DATABASE_PORT=${DATABASE_PORT}
      - DATABASE_USERNAME=${DATABASE_USERNAME}
      - DATABASE_PASSWORD=${DATABASE_PASSWORD}
      - DATABASE_NAME=${DATABASE_NAME}
      - JWT_SECRET=${JWT_SECRET}
      - JWT_EXPIRATION=${JWT_EXPIRATION}
      - PORT=${PORT}
      - SMTP_HOST=${SMTP_HOST}
      - SMTP_PORT=${SMTP_PORT}
      - SMTP_SECURE=${SMTP_SECURE}
      - SMTP_USER=${SMTP_USER}
      - SMTP_PASS=${SMTP_PASS}
      - SMTP_CODE_EXPIRATION=${SMTP_CODE_EXPIRATION}
      - ROLE_ADMIN=${ROLE_ADMIN}
      - DEFAULT_USER_ROLE=${DEFAULT_USER_ROLE}
      - SELLER_NAME=${SELLER_NAME}
      - SELLER_ADDRESS=${SELLER_ADDRESS}
      - SELLER_CITY=${SELLER_CITY}
      - SELLER_PIB=${SELLER_PIB}
      - SELLER_MAT_BROJ=${SELLER_MAT_BROJ}
      - SELLER_BANK_ACCOUNT=${SELLER_BANK_ACCOUNT}
      - SELLER_EMAIL=${SELLER_EMAIL}
      - SELLER_PHONE=${SELLER_PHONE}
      - S3_REGION=${S3_REGION}
      - S3_BUCKET=${S3_BUCKET}
      - S3_ENDPOINT=${S3_ENDPOINT}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - S3_FORCE_PATH_STYLE=${S3_FORCE_PATH_STYLE}
      - S3_CDN_URL=${S3_CDN_URL}
    networks:
      - safi-sql
    depends_on:
      - sql
      - minio
    restart: unless-stopped

  frontend:
    build:
      context: ../front-b2b # adjust path if needed
    ports:
      - "3000:3000"
    env_file:
      - .env
    environment:
      - PORT=3000
      - NEXT_PUBLIC_API_URL=${NEXT_PUBLIC_API_URL}
      - NEXT_PUBLIC_ROLE=${NEXT_PUBLIC_ROLE}
      - NEXT_PUBLIC_ADMIN_ROLE=${NEXT_PUBLIC_ADMIN_ROLE}
      - NEXT_PUBLIC_BASE_PATH=${NEXT_PUBLIC_BASE_PATH}
      - ADMIN_ROLE=${ADMIN_ROLE}
    networks:
      - safi-sql
    depends_on:
      - api
    restart: unless-stopped

  sql:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=${DATABASE_PASSWORD}
    ports:
      - "${DATABASE_PORT}:1433"
    volumes:
      - mssql-data:/var/opt/mssql
    networks:
      - safi-sql
    restart: unless-stopped

  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      - MINIO_ROOT_USER=${AWS_ACCESS_KEY_ID}
      - MINIO_ROOT_PASSWORD=${AWS_SECRET_ACCESS_KEY}
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio-data:/data
    networks:
      - safi-sql
    restart: unless-stopped

  nginx:
    image: nginx:stable
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/certs:/etc/nginx/certs:ro
      - ./nginx/www:/var/www/certbot:ro # Webroot for ACME challenge
      - ./certbot/conf:/etc/letsencrypt:ro # Let's Encrypt certs (ro, created by Certbot)
      - ./nginx/log:/var/log/nginx
    networks:
      - safi-sql
    depends_on:
      - frontend
      - api
      # Not strictly necessary but can help if nginx should wait for Certbot (after initial start)
      # - certbot
    restart: unless-stopped

  # NEW Certbot service
  certbot:
    image: certbot/certbot
    volumes:
      # This path MUST match the certs path in the Nginx service
      - ./certbot/conf:/etc/letsencrypt
      # This path MUST match the ACME challenge webroot in Nginx
      - ./nginx/www:/var/www/certbot
    # Use profiles for manual run or initial certificate issuance
    profiles: ["ssl"]
    command: >-
      certonly --webroot
      --webroot-path=/var/www/certbot
      --email safi@safi.rs
      --agree-tos --no-eff-email
      -d b2b.safi.rs
    depends_on:
      - nginx # Waits for Nginx to be up for validation
    # No restart: unless-stopped, since it runs only as needed (profile 'ssl')

networks:
  safi-sql:
    driver: bridge

volumes:
  mssql-data:
  minio-data:
# Commented/Removed: nginx-cert-data as it was not used in services
```

Run:

```bash
cd docker
docker compose up --build
```

## Database and migrations

- If you use an ORM (Prisma/TypeORM), define migrations in the `prisma` folder or use the ORM CLI:
  - Prisma:
    ```bash
    npx prisma migrate dev --name init
    npx prisma generate
    ```
  - TypeORM (example):
    ```bash
    npm run typeorm migration:run
    ```
- If you use raw SQL scripts, execute them against the database in the correct order (files in `/sql-scripts`).

## API documentation (Swagger)

Swagger is available in development at:

```
GET /api/docs
```

(or `/api` depending on configuration). Use Swagger to view routes, schemas and to test endpoints.

## Example requests (curl)

- Login (obtain JWT):

```bash
curl -X POST "http://localhost:3000/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"demo"}'
```

- Call a protected endpoint:

```bash
curl -X GET "http://localhost:3000/users/me" \
  -H "Authorization: Bearer <JWT_TOKEN>"
```

- Create an entity:

```bash
curl -X POST "http://localhost:3000/items" \
  -H "Authorization: Bearer <JWT_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Item 1","price":123.45}'
```

Adjust routes according to implementation in `src/controllers`.

## Architecture and design (short)

- Modular NestJS structure (feature modules: auth, users, items, sync etc.)
- Services contain business logic; controllers expose REST routes
- DTOs and validation pipe for input validation
- Error handling and global filters for uniform responses

## Synchronization and integrations

- If you need to connect to a local ERP (without a fixed IP) we recommend:
  - Cloudflare Tunnel (cloudflared) or
  - autossh reverse tunnel to a VPS + Nginx reverse proxy or
  - WireGuard VPN
- Use idempotency and retry mechanisms in sync workers; keep logs and a queue for errors.

## Deployment (production)

Recommended stack:

- VPS / cloud instance (DigitalOcean, Hetzner, AWS EC2)
- Docker Compose or Kubernetes for orchestration
- Nginx as reverse proxy and TLS terminator (Certbot for Let's Encrypt)
- Set up systemd services / docker restart policy for resilience
- CI/CD: GitHub Actions for build and deploy pipeline (store secrets in Actions Secrets)

Nginx example configuration (reverse proxy):

```nginx
server {
  listen 80;
  server_name api.example.com;
  location / {
    proxy_pass http://127.0.0.1:3000;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
  }
}
```

## Security and secrets

- Do not commit real secrets (.env, private keys). Add `.env` to `.gitignore`.
- Use secret managers on the deployment platform (GitHub Secrets, Vault, cloud provider secrets).
- Rotate keys if they were ever exposed.

## Tests

- Unit and integration tests (Jest or another test runner):

```bash
npm run test
npm run test:cov
```

- Cover critical business logic with tests (auth, sync, idempotency).

## Logging & monitoring

- Structured logging (JSON or leveled logs) with configured transports
- Health endpoint: `/health` or `/healthz` that returns service and DB connection status
- Integrate with monitoring tools (Prometheus, Grafana, Sentry for error tracking) as needed

## Troubleshooting

- Check env variables and DB connection if the service does not start
- Check logs: `docker logs <container>` or systemd unit logs
- Check migrations and schema versions

## Contributing

- Open an issue for bugs or feature requests
- Create a branch `feature/<short-description>` and submit a PR with description and tests
- Keep commit messages clear and atomic

## License

- Add a license file (`LICENSE`) as needed (e.g. MIT).

## Useful links

- Frontend demo repo: https://github.com/Hogar77/front-b2b
- ERP sync repo: https://github.com/Hogar77/erp-api-sync
- Live demo / API: https://b2b.safi.rs/api
