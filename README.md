# sql-api

REST API napisan u NestJS + TypeScript za upravljanje poslovnim entitetima (auth, korisnici, CRUD i dr.). Ovaj repozitorij sadrži kod backend servisa, dokumentaciju endpointa i instrukcije za lokalno pokretanje i deploy.

## Osnovno (kratko)

- Tehnologije: NestJS, TypeScript, Node.js
- DB: PostgreSQL (ili MS SQL po potrebi)
- Autentifikacija: JWT
- Dokumentacija API‑ja: Swagger / OpenAPI
- Deploy: Docker Compose, Nginx kao reverse proxy, Certbot za TLS

## Sadržaj repozitorijuma

- `/src` — izvorni TypeScript kod aplikacije (controllers, services, modules)
- `/sql` ili `/sql-scripts` — SQL skripte / migracije / stored procedures (ako postoje)
- `docker/` — Docker Compose i Dockerfile primeri za lokalni razvoj i deploy
- `.env.example` — primer environment promenljivih
- `README.md` — ovaj fajl

## Brzi start — lokalno (developer)

1. Kloniraj repozitorij:

```bash
git clone https://github.com/Hogar77/sql-api.git
cd sql-api
```

2. Napravi `.env` fajl na osnovu `.env.example`:

```bash
cp .env.example .env
# Uredi .env i unesi prave vrednosti (ne commituj .env)
```

Primer `.env.example`:

# App
PORT=4000
NODE_ENV=development

# Frontend / public config (ako koristiš Next.js ili slično)
NEXT_PUBLIC_API_URL=http://localhost:4000
NEXT_PUBLIC_BASE_PATH=
NEXT_PUBLIC_ROLE=user
NEXT_PUBLIC_ADMIN_ROLE=admin
ADMIN_ROLE=admin
DEFAULT_USER_ROLE=user

# Database (MSSQL u docker-compose.yml primeru)
# Ako koristiš Postgres zameni image/port i promenljive POSTGRES_...
DATABASE_HOST=sql
DATABASE_PORT=1433
DATABASE_USERNAME=sa
DATABASE_PASSWORD=change_me_password
DATABASE_NAME=sql_api_db

# (Opcionalno za Postgres docker image)
POSTGRES_DB=sql_api_db
POSTGRES_USER=postgres
POSTGRES_PASSWORD=change_me_password

# JWT / auth
JWT_SECRET=change_this_to_a_strong_secret
JWT_EXPIRATION=3600   # u sekundama ili string kao "1h"

# SMTP (email)
SMTP_HOST=smtp.example.com
SMTP_PORT=587
SMTP_SECURE=false     # true za 465
SMTP_USER=smtp_user@example.com
SMTP_PASS=smtp_password
SMTP_CODE_EXPIRATION=300000   # u ms (npr. 5 minuta)

# Role / admin (primer)
ROLE_ADMIN=admin_role_name

# Seller / fakturisanje (opciono)
SELLER_NAME="My Company DOO"
SELLER_ADDRESS="Ulica 1"
SELLER_CITY=Beograd
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

# MinIO root (ako pokrećeš minio servis)
MINIO_ROOT_USER=minio
MINIO_ROOT_PASSWORD=minio_password

# Certbot / nginx (primer e-mail u docker-compose certbot service)
# Ostaviti prazno ili podesiti svoj kontakt e-mail (ne stavljati tajne)
CERTBOT_EMAIL=safi@example.com

# Dodatne (ako su potrebne)
# REDIS_URL=redis://redis:6379
# OTHER_SERVICE_URL=

3. Instaliraj zavisnosti i pokreni:

```bash
npm install
npm run build
npm run start:dev    # ili `npm run start` za produkciju (ili `npm run start:prod`)
```

API će po defaultu biti dostupan na: `http://localhost:3000` (ili PORT iz `.env`).

## Pokretanje sa Docker Compose

U direktorijumu `docker/` se nalazi primer `docker-compose.yml`. Osnovni primer:

```yaml
version: '3.9'

services:
  api:
    build: .
    ports:
      - '4000:4000'
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
      context: ../front-b2b # prilagodi putanju ako treba
    ports:
      - '3000:3000'
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
      - '${DATABASE_PORT}:1433'
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
      - '9000:9000'
      - '9001:9001'
    volumes:
      - minio-data:/data
    networks:
      - safi-sql
    restart: unless-stopped

  nginx:
    image: nginx:stable
    ports:
      - '80:80'
      - '443:443'
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/certs:/etc/nginx/certs:ro
      - ./nginx/www:/var/www/certbot:ro # Webroot za ACME challenge
      - ./certbot/conf:/etc/letsencrypt:ro # Let's Encrypt sertifikati (ro, jer ih kreira Certbot)
      - ./nginx/log:/var/log/nginx
    networks:
      - safi-sql
    depends_on:
      - frontend
      - api
      # Nije striktno neophodno, ali može biti korisno ako nginx treba da sačeka Certbot (nakon inicijalnog pokretanja)
      # - certbot
    restart: unless-stopped

  # NOVI Certbot servis
  certbot:
    image: certbot/certbot
    volumes:
      # Ova putanja MORA da odgovara putanji u Nginx servisu za sertifikate
      - ./certbot/conf:/etc/letsencrypt
      # Ova putanja MORA da odgovara putanji za ACME challenge u Nginx-u
      - ./nginx/www:/var/www/certbot
    # Koristi profiles za ručno pokretanje ili za inicijalnu instalaciju sertifikata
    profiles: ['ssl']
    command: >-
      certonly --webroot
      --webroot-path=/var/www/certbot
      --email safi@safi.rs
      --agree-tos --no-eff-email
      -d b2b.safi.rs
    depends_on:
      - nginx # Čeka da Nginx bude pokrenut da bi mogao da izvrši validaciju
    # Nema restart: unless-stopped, jer se pokreće samo po potrebi (profil 'ssl')

networks:
  safi-sql:
    driver: bridge

volumes:
  mssql-data:
  minio-data:
# Zakomentarisano/Uklonjeno: nginx-cert-data: jer nije korišćeno u servisima
```

Pokretanje:

```bash
cd docker
docker compose up --build
```

## Baza podataka i migracije

- Ako koristiš ORM (Prisma/TypeORM), definiši migracije u folderu `prisma` ili koristeći CLI ORM‑a:
  - Prisma:
    ```bash
    npx prisma migrate dev --name init
    npx prisma generate
    ```
  - TypeORM (primer):
    ```bash
    npm run typeorm migration:run
    ```
- Ako koristiš SQL skripte, izvrši ih na bazi u ispravnom redosledu (fajlovi u `/sql-scripts`).

## Dokumetacija API‑ja (Swagger)

Swagger je dostupan u developmentu na ruti:

```
GET /api/docs
```

(ili `/api` u zavisnosti od konfiguracije). Koristi Swagger da pogledaš rute, šeme i testiraš endpoint‑e.

## Primeri zahteva (curl)

- Login (dobijanje JWT):

```bash
curl -X POST "http://localhost:3000/auth/login" \
  -H "Content-Type: application/json" \
  -d '{"username":"demo","password":"demo"}'
```

- Poziv zaštićenog endpointa:

```bash
curl -X GET "http://localhost:3000/users/me" \
  -H "Authorization: Bearer <JWT_TOKEN>"
```

- Kreiranje entiteta:

```bash
curl -X POST "http://localhost:3000/items" \
  -H "Authorization: Bearer <JWT_TOKEN>" \
  -H "Content-Type: application/json" \
  -d '{"name":"Item 1","price":123.45}'
```

Prilagodi rute prema implementaciji u `src/controllers`.

## Arhitektura i dizajn (kratko)

- Modularna NestJS struktura (feature modules: auth, users, items, sync itd.)
- Services sadrže poslovnu logiku; controllers izlažu REST rute
- DTO‑ovi i validation pipe za validaciju input‑a
- Error handling i global filters za uniformne odgovore

## Sinhronizacija i integracije

- Ako je potrebno povezivanje sa lokalnim ERP‑om (bez fiksne IP) — preporučujemo:
  - Cloudflare Tunnel (cloudflared) ili
  - autossh reverse tunnel ka VPS + Nginx reverse proxy ili
  - WireGuard VPN
- Koristi idempotency i retry mehanizme u sync worker‑ima; zadrži logove i queue za greške.

## Deployment (produkcija)

Preporučeni stack:

- VPS / cloud instance (DigitalOcean, Hetzner, AWS EC2)
- Docker Compose ili Kubernetes za orkestraciju
- Nginx kao reverse proxy i terminator TLS (Certbot za Let's Encrypt)
- Postavljanje systemd servisa / docker restart policy za resilijentnost
- CI/CD: GitHub Actions za build i deploy pipeline (uvođenje secrets u Actions Secrets)

Nginx primer konfiguracije (reverse proxy):

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

## Sigurnost i tajne

- Ne commituj stvarne tajne (.env, private keys). Dodaj `.env` u `.gitignore`.
- Koristi secret menadžere na deployment platformi (GitHub Secrets, Vault, cloud provider secrets).
- Rotiraj ključeve ako su ikada bili izloženi.

## Testovi

- Unit testovi i integracioni testovi (Jest ili drugi test runner):

```bash
npm run test
npm run test:cov
```

- Pokrij kritične delove poslovne logike testovima (auth, sync, idempotency).

## Logging & monitoring

- Struktuirano logovanje (JSON ili leveled logs) sa konfigurisanim transportima
- Health endpoint: `/health` ili `/healthz` koji vraća stanje servisa i DB konekcije
- Integracija sa monitoring alatima (Prometheus, Grafana, Sentry za error tracking) po potrebi

## Troubleshooting

- Proveri env varijable i DB konekciju ako servis ne startuje
- Proveri logs: `docker logs <container>` ili systemd unit logs
- Proveri migracije i verzije šeme baze

## Contributing

- Otvori issue za bug ili feature request
- Napravi branch `feature/<kratak-opis>` i pošalji PR sa opisom i testovima
- Drži commit poruke jasnim i atomarnim

## License

- Dodaj license fajl (`LICENSE`) prema potrebi (npr. MIT).

## Korisni linkovi

- Repo frontend demo: https://github.com/Hogar77/front-b2b
- ERP sync repo: https://github.com/Hogar77/erp-api-sync
- Live demo / API: https://b2b.safi.rs/api

---

Ako želiš, mogu:

- prilagoditi README sa tačnim komandama iz package.json (pogledam `package.json` i ubacim tačne npm skripte),
- generisati primer `docker-compose.yml` u `docker/` folderu i napraviti PR,
- ili dodati gotov Swagger/OpenAPI snapshot u repozitorij.

Reci šta da uradim dalje.
