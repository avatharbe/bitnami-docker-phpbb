# phpBB 3.0.14 LAMP dev container — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Stand up a Dockerized PHP 5.6 + Apache + MySQL 5.7 + phpMyAdmin stack for running and testing legacy phpBB 3.0.14 and its MODs.

**Architecture:** Three Compose services on one bridge network: `php` (built from `php:5.6-apache`, live bind-mount of the board tree), `db` (`mysql:5.7`, bind-mounted data dir), `phpmyadmin`. Board source is user-provided and dropped into `./php` after bring-up. Verification is command-driven (`docker compose` + HTTP/DB probes), not a unit-test suite.

**Tech Stack:** Docker Compose v2, `php:5.6-apache` (amd64 via emulation), `mysql:5.7` (amd64 via emulation), `phpmyadmin:latest` (native arm64), Apache 2.4, bash.

## Global Constraints

- Host is **Apple Silicon (arm64)**; `php` and `db` services MUST set `platform: linux/amd64`. `phpmyadmin` runs native (multi-arch).
- Ports MUST avoid the sibling `phpbb329` setup: board `8014:80`, phpMyAdmin `8015:80`, MySQL `6034:3306`.
- DB credentials: root `root`; db `phpbb`, user `phpbb`, password `phpbb`.
- Service names are DNS names on the shared network; board connects to DB host `db` port `3306`.
- MySQL runs with `--sql-mode=""` and `--default-authentication-plugin=mysql_native_password`.
- All config files (compose, Dockerfile, README, .gitignore) are versioned; `mysql/` data and the phpBB board source under `php/` (everything except `php/Dockerfile`) are gitignored.
- Project root: `/Users/Andreas/development/docker/phpbb3014/`.

---

### Task 1: Project scaffold, .gitignore, README skeleton

**Files:**
- Create: `.gitignore`
- Create: `README.md`
- Create: `php/.gitkeep`, `mysql/.gitkeep`

**Interfaces:**
- Consumes: nothing.
- Produces: the directory layout (`php/`, `mysql/`) and ignore rules later tasks rely on. `php/Dockerfile` (Task 2) and `docker-compose.yml` (Task 3) live at paths this task establishes.

- [ ] **Step 1: Create `.gitignore`**

```gitignore
# MySQL data directory (runtime state)
/mysql/*
!/mysql/.gitkeep

# phpBB board source is user-provided and bulky — version only the Dockerfile
/php/*
!/php/Dockerfile
!/php/.gitkeep

# macOS
.DS_Store
```

- [ ] **Step 2: Create placeholder keeps so empty dirs exist**

Create `php/.gitkeep` and `mysql/.gitkeep` as empty files.

- [ ] **Step 3: Create `README.md` skeleton**

````markdown
# phpBB 3.0.14 LAMP dev container

PHP 5.6 + Apache + MySQL 5.7 + phpMyAdmin for running/testing legacy phpBB 3.0.14 and MODs.

## Services & ports

| Service      | URL / port                | Notes                          |
|--------------|---------------------------|--------------------------------|
| Board (php)  | http://localhost:8014     | Apache + PHP 5.6               |
| phpMyAdmin   | http://localhost:8015     | DB inspection / SQL import     |
| MySQL (db)   | localhost:6034            | user/pass/db all `phpbb`; root `root` |

> Apple Silicon: `php` and `db` run under `linux/amd64` emulation (slower, expected).

## Usage

(Filled in by Task 4 — bring-up, install, teardown.)
````

- [ ] **Step 4: Verify layout**

Run: `cd /Users/Andreas/development/docker/phpbb3014 && ls -a php mysql && cat .gitignore`
Expected: both dirs contain `.gitkeep`; `.gitignore` prints.

- [ ] **Step 5: Commit**

```bash
cd /Users/Andreas/development/docker/phpbb3014
git add .gitignore README.md php/.gitkeep mysql/.gitkeep
git commit -m "chore: scaffold phpbb3014 project layout and gitignore"
```

---

### Task 2: PHP 5.6 + Apache Dockerfile

**Files:**
- Create: `php/Dockerfile`

**Interfaces:**
- Consumes: base image `php:5.6-apache`.
- Produces: an image the `php` service (Task 3) builds via `build: ./php`. Serves `/var/www/html` on container port 80 with `mysqli`, `mysql`, `gd`, `mbstring`, `zip` loaded and `mod_rewrite` + `AllowOverride All` active.

- [ ] **Step 1: Write `php/Dockerfile`**

```dockerfile
FROM php:5.6-apache

# Build deps for gd (png/jpeg) and zip
RUN apt-get update && apt-get install -y --no-install-recommends \
        libpng-dev \
        libjpeg-dev \
        libzip-dev \
        zlib1g-dev \
    && docker-php-ext-configure gd --with-jpeg-dir=/usr/include/ \
    && docker-php-ext-install mysqli mysql gd mbstring zip \
    && rm -rf /var/lib/apt/lists/*

# phpBB needs URL rewriting and .htaccess overrides
RUN a2enmod rewrite \
    && sed -ri 's/AllowOverride None/AllowOverride All/g' /etc/apache2/apache2.conf

EXPOSE 80
```

Note: if `libzip-dev` is too new for the bundled `zip` ext on the 5.6 image and the build fails, drop `zip` from the `docker-php-ext-install` line and the `libzip-dev` package — phpBB 3.0.14 does not require it. Re-run Step 2 after editing.

- [ ] **Step 2: Verify the image builds**

Run: `cd /Users/Andreas/development/docker/phpbb3014 && docker build --platform linux/amd64 -t phpbb3014-php ./php`
Expected: build completes; final line `naming to ... phpbb3014-php`.

- [ ] **Step 3: Verify PHP version and extensions inside the image**

Run: `docker run --rm --platform linux/amd64 phpbb3014-php sh -c "php -v && php -m | grep -E '^(mysqli|mysql|gd|mbstring)$'"`
Expected: `PHP 5.6.x`, and lines `mysqli`, `mysql`, `gd`, `mbstring` present.

- [ ] **Step 4: Commit**

```bash
cd /Users/Andreas/development/docker/phpbb3014
git add php/Dockerfile
git commit -m "feat: PHP 5.6 + Apache image with phpBB extensions"
```

---

### Task 3: docker-compose.yml (php + db + phpmyadmin)

**Files:**
- Create: `docker-compose.yml`

**Interfaces:**
- Consumes: `php/Dockerfile` (Task 2); base images `mysql:5.7`, `phpmyadmin:latest`.
- Produces: three named services (`php`, `db`, `phpmyadmin`) on network `phpbb3014`, mounting `./php` and `./mysql`, exposing 8014/8015/6034. Task 4 brings this up.

- [ ] **Step 1: Write `docker-compose.yml`**

```yaml
networks:
  phpbb3014:

services:
  php:
    build:
      context: ./php
      dockerfile: Dockerfile
    platform: linux/amd64
    container_name: phpbb3014-php
    networks: [phpbb3014]
    depends_on: [db]
    volumes:
      - ./php:/var/www/html
    ports:
      - "8014:80"
    restart: unless-stopped

  db:
    image: mysql:5.7
    platform: linux/amd64
    container_name: phpbb3014-db
    command: --sql-mode="" --default-authentication-plugin=mysql_native_password
    networks: [phpbb3014]
    volumes:
      - ./mysql:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: phpbb
      MYSQL_USER: phpbb
      MYSQL_PASSWORD: phpbb
    ports:
      - "6034:3306"
    restart: unless-stopped

  phpmyadmin:
    image: phpmyadmin:latest
    container_name: phpbb3014-pma
    networks: [phpbb3014]
    depends_on: [db]
    environment:
      PMA_HOST: db
      PMA_PORT: 3306
      UPLOAD_LIMIT: 256M
    ports:
      - "8015:80"
    restart: unless-stopped
```

- [ ] **Step 2: Validate compose config**

Run: `cd /Users/Andreas/development/docker/phpbb3014 && docker compose config`
Expected: parsed config prints with no error; three services listed.

- [ ] **Step 3: Commit**

```bash
cd /Users/Andreas/development/docker/phpbb3014
git add docker-compose.yml
git commit -m "feat: compose stack — php 5.6, mysql 5.7, phpmyadmin"
```

---

### Task 4: Bring up the stack, verify end-to-end, finalize README

**Files:**
- Modify: `README.md` (fill in Usage section)
- Temp: `php/_probe.php` (created then deleted — never committed)

**Interfaces:**
- Consumes: everything from Tasks 1–3.
- Produces: a running, verified stack and complete operator docs. No new code artifacts.

- [ ] **Step 1: Build and start the stack**

Run: `cd /Users/Andreas/development/docker/phpbb3014 && docker compose up -d --build`
Expected: `php`, `db`, `phpmyadmin` reach `Started`/`Running`. (First run pulls amd64 images — may be slow.)

- [ ] **Step 2: Wait for MySQL to finish initializing, then confirm it is healthy**

Run: `docker compose exec db sh -c 'until mysqladmin ping -h localhost --silent; do sleep 2; done; mysql -uphpbb -pphpbb -e "SELECT VERSION();" phpbb'`
Expected: prints a `5.7.x` version string with no access error (confirms `phpbb` user + DB exist and auth works).

- [ ] **Step 3: Verify Apache/PHP serves and PHP is 5.6 with the DB drivers**

Create `php/_probe.php`:

```php
<?php
echo 'PHP ' . PHP_VERSION . "\n";
echo extension_loaded('mysqli') ? "mysqli OK\n" : "mysqli MISSING\n";
var_dump(@mysqli_connect('db', 'phpbb', 'phpbb', 'phpbb') !== false);
```

Run: `curl -s http://localhost:8014/_probe.php`
Expected: `PHP 5.6.x`, `mysqli OK`, `bool(true)` (PHP container reaches the DB by service name).

- [ ] **Step 4: Verify phpMyAdmin is up and bound to the db host**

Run: `curl -s -o /dev/null -w '%{http_code}\n' http://localhost:8015/`
Expected: `200`.

- [ ] **Step 5: Remove the probe (must not be committed)**

Run: `rm /Users/Andreas/development/docker/phpbb3014/php/_probe.php`
Expected: file gone. (It is gitignored anyway, but delete it so it is not served.)

- [ ] **Step 6: Fill in the README Usage section**

Replace the `## Usage` placeholder with:

````markdown
## Usage

1. Start: `docker compose up -d --build` (first run pulls amd64 images — slow).
2. Drop your phpBB 3.0.14 tree into `./php` so `install/` sits at `php/install/`.
3. Open http://localhost:8014/install/ and run the wizard:
   - Database type: MySQL with MySQLi Extension
   - Database server hostname: `db`   (NOT localhost)
   - Database server port: `3306`
   - Database name: `phpbb`
   - Database username: `phpbb`
   - Database password: `phpbb`
4. After install, delete or rename `php/install/` as phpBB requires.
5. Edit files in `./php` on the host — changes are live (bind-mount). Refresh the browser.
6. Inspect / import the DB at http://localhost:8015 (login `phpbb` / `phpbb`, or `root` / `root`).

## Teardown

- Stop, keep data: `docker compose down`
- Wipe everything (fresh install): `docker compose down && rm -rf mysql/* && :`
````

- [ ] **Step 7: Commit**

```bash
cd /Users/Andreas/development/docker/phpbb3014
git add README.md
git commit -m "docs: verified bring-up and finalize usage instructions"
```

---

## Self-Review

- **Spec coverage:** layout (Task 1), PHP image + extensions + apache rewrite (Task 2), three services / ports / platform / sql-mode / credentials (Task 3), bring-up + end-to-end verification + usage docs (Task 4). Non-goals (no source download, no mail, no HTTPS, no seeding) are respected. Risk mitigations (native password auth, gd/zip build notes, bind-mount fallback) are folded into the relevant tasks/README.
- **Placeholder scan:** README `## Usage` placeholder in Task 1 is intentionally and explicitly filled in Task 4 Step 6; no other TBDs.
- **Type/name consistency:** service names `php`/`db`/`phpmyadmin`, container names `phpbb3014-*`, ports 8014/8015/6034, and credentials `phpbb`/`root` are identical across Tasks 3 and 4 and the README.
```
