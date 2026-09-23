# phpBB 3.0.14 LAMP dev container — Design

**Date:** 2026-07-29
**Status:** Approved (pending spec review)
**Goal:** A Dockerized LAMP stack (PHP 5.6 + Apache + MySQL 5.7) for running and testing legacy phpBB 3.0.14 and its MODs (e.g. Raidtracker, Raidplanner) on an Apple Silicon Mac.

## Context

- Host: macOS (darwin 24.6), **arm64 / Apple Silicon**, Docker Desktop 28.5 (`docker compose` v2).
- An existing sibling setup at `development/docker/phpbb329` (PHP 7.2 + MySQL 8.0) is the structural template.
- The official `php:5.6-apache` and `mysql:5.7` images are **amd64-only** → both services must declare `platform: linux/amd64` and run under emulation. This is expected and acceptable (slower cold start, fine for dev).
- The phpBB 3.0.14 board source is **user-provided** — dropped into `./php` after bring-up. This spec does not fetch it.

## Layout

```
development/docker/phpbb3014/
├── docker-compose.yml
├── .gitignore
├── README.md
├── php/
│   └── Dockerfile        # php:5.6-apache + extensions + apache config
│   # (user drops the 3.0.14 board tree here; bind-mounted live)
└── mysql/                # MySQL 5.7 data dir (bind-mount, gitignored)
```

The project directory is its own git repo. Versioned: compose file, Dockerfile, README, .gitignore, this spec. Ignored: `mysql/` (DB data) and the bulky phpBB board source under `php/` (everything except `php/Dockerfile`).

## Services

### 1. `php` (Apache + PHP 5.6)
- `build: ./php`, image base `php:5.6-apache`, `platform: linux/amd64`.
- `container_name: phpbb3014-php`.
- Dockerfile steps:
  - `docker-php-ext-install mysqli mysql gd mbstring zip`
    - Both `mysqli` **and** the legacy `mysql` driver — some 3.0-era MODs use `mysql_*` directly. Both are available in PHP 5.6.
    - `gd` for CAPTCHA / avatar image handling; `mbstring` for Unicode; `zip` for the extension/MOD tooling.
    - gd may need `apt-get install` of `libpng-dev` / `libjpeg-dev` + `docker-php-ext-configure gd` before install.
  - `a2enmod rewrite` and set `AllowOverride All` on `/var/www/html` so phpBB's `.htaccess` (URL rewriting, security rules) is honored.
- Volume: `./php:/var/www/html`.
- Port: `8014:80` (avoids phpbb329's 8000).
- `depends_on: [db]`.

### 2. `db` (MySQL 5.7)
- `image: mysql:5.7`, `platform: linux/amd64`.
- `container_name: phpbb3014-db`.
- `command: --sql-mode="" --default-authentication-plugin=mysql_native_password` — relax MySQL 5.7 strict mode for 2015-era SQL; native password auth for the old mysqli client.
- Volume: `./mysql:/var/lib/mysql`.
- Env: `MYSQL_ROOT_PASSWORD=root`, `MYSQL_DATABASE=phpbb`, `MYSQL_USER=phpbb`, `MYSQL_PASSWORD=phpbb`.
- Port: `6034:3306` (avoids phpbb329's 6033).
- `restart: unless-stopped`.

### 3. `phpmyadmin`
- `image: phpmyadmin:latest` (multi-arch, runs native on arm64).
- `container_name: phpbb3014-pma`.
- Env: `PMA_HOST=db`, `PMA_PORT=3306`, `UPLOAD_LIMIT=256M` (for importing SQL dumps).
- Port: `8015:80`.
- `depends_on: [db]`.

All three services share a user-defined bridge network (`phpbb3014`) so they resolve each other by service name (`db`, etc.).

## Usage flow (documented in README)

1. `cd development/docker/phpbb3014 && docker compose up -d --build`
2. Drop the phpBB 3.0.14 tree into `./php` (so `install/` sits at `php/install/`).
3. Browse `http://localhost:8014/install/`; in the installer set **DB host = `db`**, DB name/user/pass = `phpbb`/`phpbb`/`phpbb`, DB port `3306`.
4. After install, delete/rename `php/install/` as phpBB requires.
5. MOD testing: edit files in `./php` on the host (live bind-mount) → refresh browser. Inspect/modify DB at `http://localhost:8015`.
6. Teardown: `docker compose down` (keeps data) / `docker compose down -v` + clear `./mysql` for a clean slate.

## Non-goals / YAGNI

- No auto-download of the phpBB source (user provides it).
- No mail/SMTP service (phpBB email testing out of scope).
- No HTTPS / reverse proxy — plain HTTP on localhost is sufficient for dev.
- No seeding of a pre-built board or MODs — clean install driven by the web wizard.

## Risks & mitigations

- **amd64 emulation flakiness:** if `php:5.6-apache` fails to pull/run, fall back to a community arm64 PHP 5.6 image or `php:5.6-fpm` + separate httpd. Mitigation deferred unless it actually fails.
- **MySQL 5.7 auth vs old mysqli:** handled by `mysql_native_password`.
- **`./mysql` bind-mount permissions** under Docker Desktop file sharing: if MySQL refuses to init, switch `db` to a named volume. Note in README.
