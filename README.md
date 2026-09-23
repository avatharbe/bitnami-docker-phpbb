# phpBB 3.0.14 LAMP dev container

This spins up a small, self-contained web server on your own computer so you
can run and test an old (2015-era) phpBB 3.0.14 forum, without installing PHP,
Apache, or MySQL directly on your machine. Everything runs inside **Docker
containers** — think of a container as a lightweight, disposable virtual
machine that already has everything set up correctly inside it. If you delete
it, your actual computer is untouched.

## What you need first

- **Docker Desktop** installed and running. Download it from
  [docker.com](https://www.docker.com/products/docker-desktop/) if you don't
  have it yet (free for personal use). On Windows, install it with the
  **WSL2** backend when the installer asks (this is the default and
  recommended option).
- A copy of the **phpBB 3.0.14 source files** you want to run (a folder
  containing `install/`, `adm/`, etc.). This project doesn't include the
  forum software itself — you provide it.
- A terminal: **Terminal** on macOS, **PowerShell** on Windows.

You do **not** need to install PHP, Apache, or MySQL yourself — the container
provides all of that.

## What gets started

Running this brings up three separate containers that talk to each other:

| Service      | What it's for                          | Open in your browser at |
|--------------|-----------------------------------------|--------------------------|
| `php`        | The web server running the forum        | http://localhost:8014    |
| `phpmyadmin` | A web UI for looking at/editing the database | http://localhost:8015 |
| `db`         | The MySQL database (no browser UI)      | —                         |

> **Apple Silicon Macs (M1/M2/M3/M4):** `php` and `db` are built for the
> older `amd64` chip architecture and run through an emulation layer on your
> `arm64` Mac. This is expected and only really noticeable on the very first
> build (a few minutes) — after that it runs fine for everyday dev use. On
> Windows or an Intel Mac this emulation doesn't happen at all, so it'll be
> faster.

## Step-by-step setup

1. Open a terminal in this project folder and run:

   ```
   docker compose up -d --build
   ```

   The first time you run this, Docker needs to download and build images —
   this can take several minutes (longer on Apple Silicon, see note above).
   Subsequent runs are fast. You'll know it worked when the terminal returns
   you to a normal prompt with no errors.

2. Copy your phpBB 3.0.14 source files into the `php/` folder in this
   project, so that `php/install/` exists (i.e. the `install` folder sits
   directly inside `php/`).

3. Open **http://localhost:8014/install/** in your browser and follow the
   installation wizard. When it asks for database details, enter exactly:

   | Field | Value |
   |---|---|
   | Database type | MySQL with MySQLi Extension |
   | Database server hostname | `db` (not `localhost` — that's the name of the database container) |
   | Database server port | `3306` |
   | Database name | `phpbb` |
   | Database username | `phpbb` |
   | Database password | `phpbb` |

4. Once the wizard finishes, it will tell you to delete or rename the
   `install/` folder for security. Do that inside `php/install/`.

5. Your forum is now live at http://localhost:8014/. To make changes to the
   forum's files, edit them directly inside the `php/` folder on your
   computer — the container picks up changes immediately, just refresh your
   browser (no restart needed).

6. To look at or edit the database directly, go to
   http://localhost:8015 and log in with username `phpbb` / password
   `phpbb` (or `root` / `root` for full admin access).

## Stopping / resetting

- **Stop everything, keep your data** (forum + database stay intact for next
  time):
  ```
  docker compose down
  ```

- **Wipe the database and start completely fresh** (your `php/` forum files
  are untouched, only the database is deleted):

  macOS / Linux / Git Bash:
  ```
  docker compose down && rm -rf mysql/*
  ```
  Windows PowerShell:
  ```
  docker compose down; Remove-Item mysql\* -Recurse -Force
  ```

## Troubleshooting

- **"Port is already allocated" error on startup** — something else on your
  computer is already using port 8014, 8015, or 6034. Either stop that other
  program, or edit the port numbers on the left side of the `ports:` entries
  in `docker-compose.yml` (e.g. change `"8014:80"` to `"9000:80"` and use
  that port instead).
- **MySQL container won't start / crashes on boot** — this is sometimes a
  file-permission quirk with Docker Desktop's file sharing. See the note at
  the bottom of `docker-compose.yml`'s `db` service, or ask for help
  switching it to a named volume instead of a bind-mount.
- **Page looks broken / blank after editing a file** — do a hard refresh in
  your browser (Cmd+Shift+R on Mac, Ctrl+Shift+R on Windows) to rule out
  cached files.
- **First `docker compose up` is very slow** — expected on the first run
  (downloading + building images), and extra slow on Apple Silicon Macs
  (see the emulation note above). Every run after that is fast.

## Behind the scenes (not required reading)

- `php:5.6-apache` and `mysql:5.7` are deliberately old, no-longer-updated
  ("end of life") versions — that's intentional, because this container
  exists to match phpBB 3.0.14's original 2015-era environment, not to be
  a modern production setup.
- The Dockerfile repoints Debian's package manager at an archive mirror,
  because the Debian version this PHP image is built on is itself end of
  life and its normal package servers no longer serve it.
- `phpmyadmin` is pinned to version `5.2.3` (rather than always using
  whatever is newest) so the environment doesn't unexpectedly change out
  from under you.
