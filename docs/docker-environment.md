# The Docker Environment

How a project in this family runs in Docker: what the containers are, what each
file in `.docker/` does, how to start it, and how to reuse the setup on another
app. If you just want it running, go to [section 7](#7-step-by-step-setup).

Two setups are covered. Development is a bind-mounted working copy with php-fpm,
nginx, Postgres and Redis — that's what most of these repos run today. Production
is a multi-stage build that ships without the build tools.

---

## Contents

- [1. Placeholders](#1-placeholders)
- [2. What to install](#2-what-to-install) — [Windows](#windows) · [macOS](#macos) · [Linux](#linux)
  - [What you don't need on the host](#what-you-dont-need-on-the-host)
- [3. Project layout](#3-project-layout)
- [4. The shape of it](#4-the-shape-of-it)
- [5. The services](#5-the-services)
- [6. What's in `.docker/`, and what's actually wired up](#6-whats-in-docker-and-whats-actually-wired-up)
  - [Why the inert files are there](#why-the-inert-files-are-there)
  - [What the dev PHP image gives you](#what-the-dev-php-image-gives-you)
- [7. Step-by-step setup](#7-step-by-step-setup)
  - [Step 1: Check the prerequisites](#step-1-check-the-prerequisites)
  - [Step 2: Get the code](#step-2-get-the-code)
  - [Step 3: Write `.env` before anything starts](#step-3-write-env-before-anything-starts)
  - [Step 4: Point the hostname at your machine](#step-4-point-the-hostname-at-your-machine)
  - [Step 5: Build and start](#step-5-build-and-start)
  - [Step 6: Check all five containers are up](#step-6-check-all-five-containers-are-up)
  - [Step 7: Install dependencies and generate the app key](#step-7-install-dependencies-and-generate-the-app-key)
  - [Step 8: Create the schema](#step-8-create-the-schema)
  - [Step 9: Build the front-end assets](#step-9-build-the-front-end-assets)
  - [Step 10: Check the PHP config is actually applied](#step-10-check-the-php-config-is-actually-applied)
  - [Step 11: Open it](#step-11-open-it)
  - [Step 12: Start the optional processes](#step-12-start-the-optional-processes)
  - [After the first time](#after-the-first-time)
- [8. What it serves](#8-what-it-serves)
  - [Front-end assets](#front-end-assets)
- [9. What the code may expect that Compose doesn't start](#9-what-the-code-may-expect-that-compose-doesnt-start)
- [10. Building for production](#10-building-for-production)
  - [Overriding Compose for production](#overriding-compose-for-production)
- [11. Reusing this on another app](#11-reusing-this-on-another-app)
  - [Where to add your own things](#where-to-add-your-own-things)
- [12. When it breaks](#12-when-it-breaks)
- [13. Command reference](#13-command-reference)

---

## 1. Placeholders

Substitute your own values wherever you see these:

| Placeholder | Means | Example |
| --- | --- | --- |
| `{project}` | Your project slug, usually the directory name. Used as the container-name prefix. | `uplb-sims`, `uplb-tks` |
| `{project}.local` | The development hostname nginx answers on. | `uplb-ams.local` |
| `{repository-url}` | The git remote you clone from. | `git@github.com:org/uplb-sims.git` |

So `{project}-php` is `uplb-sims-php` in one checkout and `uplb-tks-php` in
another. The names have to differ per project — section 11 covers why.

---

## 2. What to install

Only Docker and git go on your machine. PHP, Composer, Node, Postgres and Redis
all live in the containers.

### Windows

| Install | Why | How |
| --- | --- | --- |
| WSL 2 | Docker Desktop's backend, and the shell you'll run everything in | `wsl --install` in an Administrator PowerShell, then reboot |
| Docker Desktop | Docker Engine plus Compose v2 | `winget install Docker.DockerDesktop`, or download from docker.com |
| Git | Cloning the project | `winget install Git.Git`, or `sudo apt install git` inside WSL |
| Windows Terminal | Sane terminal for the WSL shell | `winget install Microsoft.WindowsTerminal` |

Four Windows-specific things worth doing once:

- In Docker Desktop, turn on **Settings → Resources → WSL Integration** for your
  distro. Without it, `docker` isn't on the PATH inside WSL.
- Keep the project inside the WSL filesystem (`/home/you/projects/...`), not
  `/mnt/c/...`. Bind mounts across the Windows drive are slow enough to notice on
  every request.
- Set `git config --global core.autocrlf input`. A CRLF `entrypoint.sh` fails in
  the container with a confusing `\r: not found` error.
- The hosts file is `C:\Windows\System32\drivers\etc\hosts`, and you have to
  open the editor as Administrator to save it.

### macOS

| Install | Why | How |
| --- | --- | --- |
| Docker Desktop | Docker Engine plus Compose v2 | `brew install --cask docker`, or download from docker.com |
| Git | Cloning the project | `xcode-select --install`, or `brew install git` |
| Homebrew | Optional, but it's what makes the above one-liners | `/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"` |

Give Docker Desktop at least 4 GB of RAM under **Settings → Resources**;
composer and the asset build are the parts that feel it. Apple silicon and Intel
both work — the images used here have arm64 builds. If you'd rather not run
Docker Desktop, OrbStack is a lighter drop-in and every command in this guide
still applies.

### Linux

Docker Engine plus the `docker-compose-plugin` package from Docker's own repo
(the distro `docker.io` package is usually too old for Compose v2), and git. Add
yourself to the `docker` group so you're not typing `sudo` all day.

### What you don't need on the host

PHP, Composer, Node, npm, Postgres and Redis. Installing them locally is
harmless, but nothing in this setup uses them — and a host Postgres already
listening on 5432 will collide with the container, which is the first thing
step 1 checks for.

---

## 3. Project layout

Development and production together. Not every project needs every file; the
second table says what happens if one is missing.

```
project-root/
├── .docker/
│   ├── php/
│   │   ├── Dockerfile.dev          dev image: extensions + tooling, no app code
│   │   ├── Dockerfile              prod image: three-stage build
│   │   ├── php.ini                 PHP overrides (memory, upload sizes)
│   │   ├── docker.conf             php-fpm pool: logging, clear_env
│   │   ├── www.conf                php-fpm pool sizing (prod)
│   │   ├── supervisord.conf        runs fpm + queue worker in one container (prod)
│   │   └── entrypoint.sh           creates storage dirs, fixes ownership, execs CMD
│   ├── nginx/
│   │   ├── nginx.conf              global nginx settings
│   │   ├── default.conf            dev vhost, proxies to php:9000
│   │   ├── prod-default.conf       prod vhost, serves baked-in assets
│   │   └── Dockerfile              prod nginx image with assets copied in
│   └── db/
│       └── my.cnf                  only if you run MySQL and need overrides
├── docker-compose.yml              development orchestration
├── docker-compose.prod.yml         production overrides, layered on the above
├── .dockerignore                   REQUIRED once anything does `COPY . /var/www`
├── docs/
│   └── docker-environment.md       this file
├── composer.json
├── package.json
└── ...application code...
```

| File | If missing |
| --- | --- |
| `php/Dockerfile.dev` | No development image; nothing to build |
| `php/Dockerfile` | No production image — you'd ship the dev one, source mount and build tools included |
| `php/php.ini` | Stock PHP defaults: `memory_limit 128M`, `upload_max_filesize 2M` |
| `php/docker.conf` | fpm logs go somewhere you can't see them; `catch_workers_output` off |
| `php/www.conf` | Default pool sizing |
| `php/supervisord.conf` | You need separate containers for the queue worker and scheduler |
| `php/entrypoint.sh` | `storage/` permission errors on first boot |
| `nginx/*.conf` | nginx serves its default welcome page instead of your app |
| `.dockerignore` | Slow builds, and `.env` plus `.git` baked into the image |
| `docker-compose.prod.yml` | Two full compose files to keep in sync instead of one plus an override |

`.dockerignore` is the one to watch. A dev-only setup gets away without it
because `Dockerfile.dev` copies no application code — the project arrives by bind
mount. Add the production Dockerfile with its `COPY . /var/www` and suddenly
`node_modules`, `vendor`, `.git` and `.env` all land in the image. Add the file
first:

```
.git
.gitignore
.github
node_modules
vendor
.env
.env.*
!.env.example
.vscode
.idea
*.swp
dist
build
coverage
.DS_Store
.dockerignore
Dockerfile*
docker-compose*.yml
```

---

## 4. The shape of it

Five containers on one network. Inside Docker, containers reach each other by
service name, not `localhost`. Postgres is `pg_db`, Redis is `redis`, php-fpm is
`php`. Put `127.0.0.1` in `.env` and the PHP container looks inside itself.

```
                    host machine
   :80        :8080        :5432       :6379      :5173
    |           |            |           |          |
+---v----+  +---v-----+  +---v-----+ +---v----+    |
| nginx  |  | adminer |  |  pg_db  | | redis  |    |
+---+----+  +----+----+  +----^----+ +---^----+    |
    | fastcgi     | pg_db      |         |         |
    | php:9000    +------------+         |         |
+---v------------------------------------------+   |
| php (php-fpm 8.4) - also runs artisan, vite -+---+
+----------------------------------------------+
                    | bind mount
              . -> /var/www  (whole project, live)
```

The bind mount is the point of the dev setup. Nothing app-specific is baked into
the image — it's just PHP with the right extensions, and your working copy is
mounted over `/var/www`. Edit a file on the host and the container sees it, no
rebuild.

---

## 5. The services

| Service | Container | Image or build | Host port | What it does |
| --- | --- | --- | --- | --- |
| `php` | `{project}-php` | built from `.docker/php/Dockerfile.dev` | `5173` for Vite | php-fpm 8.4 on port 9000; also where you run artisan, composer, npm |
| `nginx` | `{project}-nginx` | `nginx:latest` | `80` | Serves `public/`, hands `.php` to `php:9000` |
| `pg_db` | `{project}-pgdb` | `postgres:15` | `5432` | The database; data lives in the `postgres_data` volume |
| `adminer` | `{project}-adminer` | `adminer:latest` | `8080` | Browser database client, already pointed at `pg_db` |
| `redis` | `{project}-redis` | `redis/redis-stack-server:latest` | `6379` | Cache, and the full-text search index |

If the project uses RediSearch, it has to be Redis *Stack*, not plain `redis` —
the search code sends `FT.*` commands and the vanilla image doesn't ship the
module. Swap in `redis:alpine` to save a few megabytes and search stops working
without saying so.

---

## 6. What's in `.docker/`, and what's actually wired up

Common trap: `.docker/php/` carries seven files, but only some are connected to
anything. Config files do nothing unless the Dockerfile `COPY`s them or Compose
mounts them, and nothing warns you.

| File | Wired how | What it is for |
| --- | --- | --- |
| `php/Dockerfile.dev` | `build.dockerfile` in compose | `php:8.4-fpm` plus extensions and tooling |
| `nginx/default.conf` | Mounted to `/etc/nginx/conf.d/default.conf` | The vhost: root `/var/www/public`, 100M uploads, 3600s FastCGI timeouts |
| `nginx/nginx.conf` | Mounted to `/etc/nginx/nginx.conf` | Global nginx settings, gzip for JSON, `server_tokens off` |
| `php/php.ini` | `COPY` in the Dockerfile, or a mount | `memory_limit 2048M`, `upload_max_filesize 50M`, `post_max_size 100M`, `max_execution_time 300` |
| `php/docker.conf` | `COPY` to `/usr/local/etc/php-fpm.d/` | Logs to stdout and stderr, `clear_env = no`, `catch_workers_output = yes` |
| `php/entrypoint.sh` | `COPY` plus `ENTRYPOINT`, or a mount | Creates `storage/framework/{cache,views,sessions}`, fixes ownership at boot |
| `php/.bashrc` | `COPY` to the container home | Shell aliases, plus a `phpd` alias running PHP with Xdebug at `host.docker.internal:9001` |

**Check the bottom four in your project.** Where the `COPY` lines got trimmed out
of `Dockerfile.dev`, those files sit there implying settings that aren't in
effect — `php.ini` says `memory_limit 2048M` while the container runs on 128M,
and `upload_max_filesize 50M` while PHP rejects anything over 2M. nginx accepts a
100M upload and PHP refuses it.

Check with:

```bash
docker compose exec php php -i | grep -E "memory_limit|upload_max_filesize"
```

`128M` and `2M` mean the files are inert. Mount them:

```yaml
    volumes:
      - .:/var/www:cached
      - .docker/php/php.ini:/usr/local/etc/php/conf.d/zzz-app.ini
      - .docker/php/docker.conf:/usr/local/etc/php-fpm.d/zzz-docker.conf
      - .docker/php/entrypoint.sh:/usr/local/bin/app-entrypoint.sh
    entrypoint: ["sh", "/usr/local/bin/app-entrypoint.sh"]
    command: ["php-fpm"]
```

One catch before wiring the entrypoint. Some copies of `entrypoint.sh` end after
the `chmod` and never hand off to the main process, so the container fixes
permissions and exits. Make sure yours ends with:

```sh
# Hand off to the container's main process (php-fpm, from the Dockerfile CMD)
exec "$@"
```

### Why the inert files are there

They're inherited. `.docker/` gets copied from project to project, and the fuller
ancestor version does the wiring the trimmed copies dropped:

```dockerfile
COPY php.ini /usr/local/etc/php/
COPY docker.conf /usr/local/etc/php-fpm.d/docker.conf
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["php-fpm"]
```

It also clones `seebi/dircolors-solarized` into `/root`, which is what the
`eval $(dircolors ~/dircolors-solarized/...)` line in `.bashrc` looks for.
Without the clone, that line errors on every shell login.

They're leftovers, not unfinished features. Wire them up or delete them.

### What the dev PHP image gives you

Compiled in: `sockets`, `zip`, `pdo_mysql`, `pdo_pgsql`, `pgsql`, `intl`,
`pcntl`, and `gd` with freetype and jpeg. From PECL: `imagick`. System side:
ImageMagick, Ghostscript for rasterising PDFs, git, vim, unzip, procps, node and
npm, and the MySQL client. Composer is copied in from the official
`composer:latest` image.

Both MySQL and Postgres drivers are there, so the same image drops into a MySQL
project — only `DB_CONNECTION` and the database service change.

---

## 7. Step-by-step setup

The first run, in order. Steps 1 to 6 can't be reordered, step 3 especially — it's
only cheap to get right before the database volume exists.

### Step 1: Check the prerequisites

Docker and git, installed as section 2 describes. Compose has to be v2, since
every command here is `docker compose`, not `docker-compose`:

```bash
docker --version
docker compose version
git --version
```

Then check the five host ports are free — 80, 5432, 6379, 8080 and 5173.
Anything already listening on one makes step 5 fail with a bind error:

```bash
# macOS / Linux
lsof -nP -iTCP -sTCP:LISTEN | grep -E ':(80|5432|6379|8080|5173) '
```

Usual culprits: a local Apache, a Homebrew Postgres, or another project from this
same family. Stop it, or move the host side of the port as section 11 describes.

### Step 2: Get the code

```bash
git clone {repository-url} {project}
cd {project}
```

Check that `.docker/` and `docker-compose.yml` came with the checkout. If
`.docker/php/Dockerfile.dev` is missing there's nothing to build — section 3 has
the expected layout.

### Step 3: Write `.env` before anything starts

Compose reads `${DB_DATABASE}`, `${DB_USERNAME}` and `${DB_PASSWORD}` out of
`.env` to create the Postgres role, and Postgres only honours them the first time
the data volume is created. Fix them later and nothing happens — you'd have to
`docker compose down -v` and start over, which destroys the database.

```bash
cp .env.example .env
```

Set at least these:

```dotenv
APP_URL=http://{project}.local
DB_CONNECTION=pgsql
DB_HOST=pg_db          # the service name, not 127.0.0.1
DB_PORT=5432
DB_DATABASE={project}
DB_USERNAME={project}
DB_PASSWORD=<pick one>
REDIS_HOST=redis
REDIS_SEARCH_HOST=redis
```

`DB_HOST=pg_db` is the line people get wrong. Containers find each other by
service name; `127.0.0.1` points the PHP container at itself.

### Step 4: Point the hostname at your machine

nginx answers on whatever `server_name` is set in `.docker/nginx/default.conf`,
so add a matching hosts entry:

```bash
# macOS / Linux
sudo sh -c 'echo "127.0.0.1  {project}.local" >> /etc/hosts'

# Windows: edit C:\Windows\System32\drivers\etc\hosts as Administrator
```

Read the checked-in `server_name` first — a copied `default.conf` often still
carries the previous project's hostname.

### Step 5: Build and start

```bash
docker compose up -d --build
```

The first run pulls the base images and compiles the PHP extensions, so give it a
few minutes. After that, `--build` is only needed when `Dockerfile.dev` changes.

### Step 6: Check all five containers are up

```bash
docker compose ps
```

You want `php`, `nginx`, `pg_db`, `adminer` and `redis`, all `Up`. If one is in a
restart loop, read its log before going further:

```bash
docker compose logs -f php
```

### Step 7: Install dependencies and generate the app key

```bash
docker compose exec php composer install
docker compose exec php php artisan key:generate
```

Everything PHP-side runs inside the container. Prefer `docker compose exec php`
over `docker exec -it {project}-php` — it addresses the service instead of the
container name, so the same command works in every project. `vendor/` still shows
up in your working copy, because the bind mount goes both ways.

### Step 8: Create the schema

```bash
docker compose exec php php artisan migrate

# or, if the project ships seeders
docker compose exec php php artisan migrate --seed
```

`could not translate host name "pg_db"` means the command ran on the host instead
of in the container. That name only exists on the Compose network, same for
`redis`.

### Step 9: Build the front-end assets

```bash
docker compose exec php npm install
docker compose exec php npm run build
```

Skip this and the app still boots, but the first page load fails with
`Unable to locate file in Vite manifest`.

### Step 10: Check the PHP config is actually applied

```bash
docker compose exec php php -i | grep -E "memory_limit|upload_max_filesize"
```

`2048M` and `50M` mean `php.ini` is wired up. `128M` and `2M` mean it's inert and
PHP is on stock defaults — see section 6.

### Step 11: Open it

| What | Where |
| --- | --- |
| App panel | `http://{project}.local/app` |
| Admin panel | `http://{project}.local/admin` |
| Adminer | `http://localhost:8080` — system PostgreSQL, server `pg_db`, credentials from `.env` |

### Step 12: Start the optional processes

None of these run on their own in development. Each wants its own terminal:

```bash
docker compose exec php php artisan queue:listen --tries=1 --timeout=0
docker compose exec php php artisan schedule:work
docker compose exec php npm run dev -- --host 0.0.0.0
```

### After the first time

```bash
docker compose up -d          # start
docker compose ps             # what is running
docker compose logs -f php    # follow one service
docker compose down           # stop, keep the data
```

---

## 8. What it serves

One codebase, several front doors, all through the same two containers.

| Application | Where | What serves it |
| --- | --- | --- |
| App panel, for end users | `http://{project}.local/app` | nginx to php; `AppPanelProvider`, resources in `app/Filament/App/Resources` |
| Admin panel | `http://{project}.local/admin` | nginx to php; `AdminPanelProvider`, resources in `app/Filament/Resources` |
| API and mobile endpoints | `http://{project}.local/api/...` | nginx to php; Sanctum-authenticated routes in `routes/` |
| Adminer | `http://localhost:8080` | the adminer container; server `pg_db`, credentials from `.env` |
| Vite dev server | `http://localhost:5173` | the php container, started by hand |

The hostname comes from `server_name` in `.docker/nginx/default.conf`, matched by
the hosts entry from step 4. `http://localhost` works too, since there's only one
server block and it ends up being the default — but absolute URLs Laravel
generates follow `APP_URL`. Pick one host and keep the two in sync, or you get
mixed-host redirects.

### Front-end assets

```bash
# one-off build
docker compose exec php npm run build

# watch mode
docker compose exec php npm run dev -- --host 0.0.0.0
```

`--host 0.0.0.0` isn't optional. Unless `vite.config.js` sets `server.host`, Vite
binds to localhost inside the container and port 5173 looks dead from your
browser even though the process is running. Pass the flag, or add
`server.host: '0.0.0.0'` to the config.

---

## 9. What the code may expect that Compose doesn't start

**Queue worker and scheduler.** No containers for these in development, so run
them by hand:

```bash
docker compose exec php php artisan queue:listen --tries=1 --timeout=0
docker compose exec php php artisan schedule:work
```

In production that's what `supervisord.conf` is for — one container running fpm
and the worker together.

**Mail.** Nothing catches outbound mail. Use `MAIL_MAILER=log`, or add a Mailpit
service if you need to see rendered messages.

**Stale volumes.** Watch for volumes declared in `docker-compose.yml` that no
service mounts. `phpmyadmin_sessions` and `db_data` are common leftovers from a
MySQL-era version of the file. Safe to delete.

---

## 10. Building for production

The development setup leans on things you don't want in production: a
bind-mounted working copy, build tools in the image, a root user, no compiled
assets.

Production is a three-stage build, where most of the build gets thrown away
before shipping:

```
+-----------------+
|  Stage 1: Node  |   builds CSS and JS
+--------+--------+
         |
    +----+-----------------------+
    |                            |
+---v--------------+     +-------v----------+
| Stage 2: PHP     |     | Nginx runtime    |
| build & compile  |     |                  |
+---+--------------+     +-------+----------+
    |                            |
+---v--------------+     +-------v----------+
| Stage 3: PHP     |     | final nginx      |
| runtime image    |     | image            |
+------------------+     +------------------+
```

Stage one is `node:22-alpine`: `npm ci`, `npm run build`. Stage two is
`php:8.4-fpm-alpine` with the `-dev` headers and compilers, where extensions get
built and `composer install --no-dev --optimize-autoloader` runs. Stage three
starts from a clean `php:8.4-fpm-alpine` and copies in only the compiled
extensions, the config and the application — no compilers, no headers, no npm.

You get a much smaller image, no build toolchain in production, assets built once
and shared between the PHP and nginx images, and layer caching that holds because
the volatile steps come last.

The runtime stage also drops privileges and strips setuid bits:

```dockerfile
RUN find /usr/local -type f -perm /6000 -exec chmod a-s {} + || true
USER www-data
```

nginx gets its own small image: `nginx:alpine`, config copied in, and `public/`
pulled from the node stage, so it serves built assets without mounting your
source tree.

### Overriding Compose for production

Keep the dev `docker-compose.yml` as the base and layer a production file on top
instead of maintaining two full copies:

```yaml
# docker-compose.prod.yml
services:
  php:
    build:
      context: .
      dockerfile: .docker/php/Dockerfile   # the multi-stage one
    restart: always
    environment:
      APP_ENV: production
      APP_DEBUG: "false"

  nginx:
    restart: always
```

```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml build
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

Two details worth copying. Give Postgres a real healthcheck — `depends_on` only
waits for the container to start, not for the database to accept connections:

```yaml
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
```

And bound Redis memory, or the cache container will happily eat the host:

```yaml
    command: >
      redis-server
      --appendonly yes
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
```

If `docker-compose.yml` declares no network, everything lands on the Compose
default. That works, but naming one makes the topology obvious and stops a stray
container joining by accident.

---

## 11. Reusing this on another app

Nothing app-specific is baked into the dev image, so moving the setup to another
Laravel project is mostly renaming. The same recipe runs several of these
projects side by side on one machine, each with its own copy of `.docker/`.

Copy `.docker/` and `docker-compose.yml` across, then work through these five.

**Rename the containers.** `container_name:` values are fixed strings, so two
projects both claiming `{project}-php` can't run at once — the second `up` fails
with a name conflict. Three ways out, best last:

```yaml
    container_name: newproject-php        # rename every prefix by hand
```
```yaml
    # drop container_name entirely; Compose names it <dir>-php-1
```
```yaml
    container_name: ${APP_SLUG}-php       # one .env value drives all five
```

The third makes the template properly reusable: set `APP_SLUG` in `.env` and the
compose file never needs editing again.

**Move the host ports.** 80, 5432, 6379, 8080 and 5173 are single-occupancy.
Shift the host side for the second project — `8081:80`, `5433:5432`, and so on.
The container side stays put, so no application config changes. Same trick here:
`"${HTTP_PORT:-80}:80"` beats hardcoding.

**Change `server_name`** in `.docker/nginx/default.conf` to `{project}.local` and
add the matching hosts entry. Easy to forget, because the site still loads on
`http://localhost`.

**Swap the database if the app is MySQL.** `pdo_mysql` and the MySQL client are
already in the image, so it's just replacing the `pg_db` service with `mysql:8`
and setting `DB_CONNECTION=mysql`, `DB_HOST=mysql`.

**Drop what you don't need.** Redis Stack is only there for RediSearch — plain
`redis:alpine` is lighter if the app just caches. Adminer goes if you use a
desktop client.

For a non-Laravel PHP app, the only extra change is the nginx `root`, since
`/var/www/public` assumes Laravel's layout.

### Where to add your own things

Additions have to land in the right stage. Packages needed only to compile
something go in the build stage; anything the running app needs has to be
repeated in the runtime stage, or the extension loads against a library that
isn't there. The duplication looks redundant but isn't.

Built-in PHP extensions go through `docker-php-ext-install`, PECL ones through
`pecl install X && docker-php-ext-enable X`. New services join Compose with a
service name, a container name, and the shared network.

---

## 12. When it breaks

| What you see | What it usually is | What to do |
| --- | --- | --- |
| `502 Bad Gateway` | php container down, or fpm not listening | `docker compose logs php`; check the pool is on `0.0.0.0:9000` |
| `could not translate host name "pg_db"` | Command run on the host, not in the container | Prefix it with `docker compose exec php` |
| Name conflict on `up` | Another project using the same `container_name` | See section 11 |
| Postgres rejects the password after an `.env` edit | Credentials only apply when the volume is first created | `docker compose down -v` and up again, which destroys the database |
| `413 Request Entity Too Large` | Upload over a limit | nginx allows 100M; PHP may still be on the stock 2M, see section 6 |
| `Unable to locate file in Vite manifest` | Assets never built | `docker compose exec php npm run build` |
| `localhost:5173` refuses to connect | Vite bound inside the container | Add `--host 0.0.0.0`, see section 8 |
| `storage` permission errors | `entrypoint.sh` not wired up | Wire it, or `docker compose exec php chown -R www-data:www-data storage bootstrap/cache` |
| Search returns nothing | Wrong Redis image, or index never built | Confirm `redis/redis-stack-server`, then rebuild the index |
| Config changes have no effect | The file isn't mounted or copied | Check with `php -i`, see section 6 |

Resetting:

```bash
docker compose down                    # stop, keep the data
docker compose down -v                 # stop and delete volumes, database included
docker compose build --no-cache php    # rebuild after editing Dockerfile.dev
```

---

## 13. Command reference

```bash
docker compose up -d --build            # build and start
docker compose ps                       # what is running
docker compose logs -f php              # follow one service
docker compose down                     # stop

docker compose exec php php artisan migrate
docker compose exec php php artisan test --compact
docker compose exec php php artisan tinker
docker compose exec php composer install
docker compose exec php vendor/bin/pint --dirty
docker compose exec php npm run build
docker compose exec php npm run dev -- --host 0.0.0.0

# production
docker compose -f docker-compose.yml -f docker-compose.prod.yml build
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```
