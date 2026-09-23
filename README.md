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
- Wipe everything (fresh install): `docker compose down && rm -rf mysql/*`

## Notes

- `php` and `db` are amd64 images running under emulation on Apple Silicon — expect a slow first build/boot; steady-state is fine for dev.
- The Dockerfile repoints apt at `archive.debian.org` because the `php:5.6-apache` base (Debian stretch) is EOL.
- If MySQL ever refuses to initialize against the `./mysql` bind-mount (Docker Desktop file-sharing permissions), switch the `db` volume to a named volume in `docker-compose.yml`.
