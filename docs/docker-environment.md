# The Docker Environment

A walkthrough of how a project in this family runs in Docker: what the containers
are, what each file in `.docker/` is for, how to start it, and how to point the
same setup at a different application.

There are two halves to the story. The development half — a bind-mounted working
copy, php-fpm, nginx, Postgres, Redis — is what most of these repos have running
today. The production half is a three-stage build that throws most of itself
away before shipping. Both are covered, because the second is where you end up
once a project stops being local-only.

Screenshots live in `docs/images/`. The filenames below are fixed, so dropping a
matching file in place is all it takes for the image to appear. Until then the
placeholder line stays visible on purpose.

---

## 1. Placeholders in this guide

This guide is written to be copied. Anywhere you see one of these, substitute
your own value:

| Placeholder | Means | Example |
| --- | --- | --- |
| `{project}` | Your project slug, usually the directory name. Used as the container-name prefix. | `uplb-sims`, `uplb-tks` |
| `{project}.local` | The development hostname nginx answers on. | `uplb-ams.local` |
| `{registry}` | Your container registry, if you push images. | `myregistry.azurecr.io` |

So `{project}-php` means the PHP container — `uplb-sims-php` in one checkout,
`uplb-tks-php` in another. The names have to differ per project, and section 10
explains why that bites sooner than you would expect.

---

## 2. How the project should be laid out

The full layout, development and production together. Not every project needs
every file — the third column says what happens if it is absent.

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
│   ├── docker-environment.md       this file
│   └── images/                     screenshots referenced from the docs
├── composer.json
├── package.json
└── ...application code...
```

| File | If missing |
| --- | --- |
| `php/Dockerfile.dev` | No development image; nothing to build |
| `php/Dockerfile` | No production image — you would be shipping the dev one, source mount and build tools included |
| `php/php.ini` | Stock PHP defaults: `memory_limit 128M`, `upload_max_filesize 2M` |
| `php/docker.conf` | fpm logs go somewhere you cannot see them; `catch_workers_output` off |
| `php/www.conf` | Default pool sizing, usually fine until it is not |
| `php/supervisord.conf` | You need separate containers for the queue worker and scheduler |
| `php/entrypoint.sh` | `storage/` permission errors on first boot |
| `nginx/*.conf` | nginx serves its default welcome page instead of your app |
| `.dockerignore` | Slow builds, and `.env` plus `.git` baked into the image |
| `docker-compose.prod.yml` | Two full compose files to keep in sync instead of one plus an override |

The important line in that table is `.dockerignore`. A dev-only setup gets away
without it because `Dockerfile.dev` copies no application code at all — the
project arrives by bind mount. The moment you add the production Dockerfile with
its `COPY . /var/www`, a missing `.dockerignore` means `node_modules`, `vendor`,
`.git` and `.env` all land in the image. Add the file first:

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

## 3. The shape of it

Five containers sharing one network. The habit to build early: inside Docker,
containers find each other by *service name*, never by `localhost`. Postgres is
at `pg_db`, Redis at `redis`, php-fpm at `php`. Write `127.0.0.1` in `.env` and
you are telling the PHP container to look inside itself.

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

The bind mount is the whole trick of the development setup. Nothing about the
application is baked into the image — the image is just PHP with the right
extensions, and your working copy is mounted over `/var/www`. Edit a file on the
host and the container sees it immediately, no rebuild.


---

## 4. The services

| Service | Container | Image or build | Host port | What it does |
| --- | --- | --- | --- | --- |
| `php` | `{project}-php` | built from `.docker/php/Dockerfile.dev` | `5173` for Vite | php-fpm 8.4 on port 9000; also where you run artisan, composer, npm |
| `nginx` | `{project}-nginx` | `nginx:latest` | `80` | Serves `public/`, hands `.php` to `php:9000` |
| `pg_db` | `{project}-pgdb` | `postgres:15` | `5432` | The database; data lives in the `postgres_data` volume |
| `adminer` | `{project}-adminer` | `adminer:latest` | `8080` | Browser database client, already pointed at `pg_db` |
| `redis` | `{project}-redis` | `redis/redis-stack-server:latest` | `6379` | Cache, and the full-text search index |

That last one matters more than it looks. Where a project uses RediSearch for
full-text search, it has to be Redis *Stack*, not plain `redis` — the search code
issues `FT.*` commands and the vanilla image does not ship the module. Swap in
`redis:alpine` to save a few megabytes and search silently stops working.

---

## 5. What is in `.docker/`, and what is actually wired up

A recurring trap in these repos: `.docker/php/` carries seven files, but only
some of them are connected to anything. Configuration files do nothing unless the
Dockerfile `COPY`s them or Compose mounts them, and nothing warns you.

| File | Wired how | What it is for |
| --- | --- | --- |
| `php/Dockerfile.dev` | `build.dockerfile` in compose | `php:8.4-fpm` plus extensions and tooling |
| `nginx/default.conf` | Mounted to `/etc/nginx/conf.d/default.conf` | The vhost: root `/var/www/public`, 100M uploads, 3600s FastCGI timeouts |
| `nginx/nginx.conf` | Mounted to `/etc/nginx/nginx.conf` | Global nginx settings, gzip for JSON, `server_tokens off` |
| `php/php.ini` | `COPY` in the Dockerfile, or a mount | `memory_limit 2048M`, `upload_max_filesize 50M`, `post_max_size 100M`, `max_execution_time 300` |
| `php/docker.conf` | `COPY` to `/usr/local/etc/php-fpm.d/` | Logs to stdout and stderr, `clear_env = no`, `catch_workers_output = yes` |
| `php/entrypoint.sh` | `COPY` plus `ENTRYPOINT`, or a mount | Creates `storage/framework/{cache,views,sessions}`, fixes ownership at boot |
| `php/.bashrc` | `COPY` to the container home | Shell aliases, plus a `phpd` alias running PHP with Xdebug at `host.docker.internal:9001` |

**Check which of the bottom four your project actually applies.** Where a repo
has trimmed the `COPY` lines out of `Dockerfile.dev`, the files sit there
implying settings that are not in effect — `php.ini` says `memory_limit 2048M`
while the container runs on 128M, and `upload_max_filesize 50M` while PHP rejects
anything over 2M. nginx will happily accept a 100M upload and PHP will refuse it,
which is a confusing way to spend an afternoon.

The quickest way to check:

```bash
docker exec -it {project}-php php -i | grep -E "memory_limit|upload_max_filesize"
```

If those come back as `128M` and `2M`, the files are inert. Mount them:

```yaml
    volumes:
      - .:/var/www:cached
      - .docker/php/php.ini:/usr/local/etc/php/conf.d/zzz-app.ini
      - .docker/php/docker.conf:/usr/local/etc/php-fpm.d/zzz-docker.conf
      - .docker/php/entrypoint.sh:/usr/local/bin/app-entrypoint.sh
    entrypoint: ["sh", "/usr/local/bin/app-entrypoint.sh"]
    command: ["php-fpm"]
```

One catch before wiring the entrypoint. Some copies of `entrypoint.sh` in this
family end after the `chmod` and never hand off to the main process — used as an
entrypoint as-is, the container fixes permissions and exits. Confirm yours ends
with:

```sh
# Hand off to the container's main process (php-fpm, from the Dockerfile CMD)
exec "$@"
```

### Why the inert files are there at all

They are inherited. `.docker/` gets copied from project to project, and the
fuller ancestor version does the wiring that trimmed copies dropped:

```dockerfile
COPY php.ini /usr/local/etc/php/
COPY docker.conf /usr/local/etc/php-fpm.d/docker.conf
COPY entrypoint.sh /usr/local/bin/entrypoint.sh
RUN chmod +x /usr/local/bin/entrypoint.sh
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]
CMD ["php-fpm"]
```

It also clones `seebi/dircolors-solarized` into `/root`, which is what the
`eval $(dircolors ~/dircolors-solarized/...)` line in `.bashrc` is looking for —
without the clone, that line errors on every shell login.

So these are copy-forward leftovers, not unfinished features. Either wire them up
or delete them; leaving them in place implying settings that are not active is
the worst of the three options.

### What the dev PHP image gives you

Compiled in: `sockets`, `zip`, `pdo_mysql`, `pdo_pgsql`, `pgsql`, `intl`,
`pcntl`, and `gd` with freetype and jpeg. From PECL: `imagick` and `rdkafka`.
On the system side: ImageMagick, Ghostscript for rasterising PDFs, git, vim,
unzip, procps, node and npm, and the MySQL client. Composer is copied in from the
official `composer:latest` image.

Both MySQL and Postgres drivers are present, which is why the same image drops
into a MySQL project without edits — only `DB_CONNECTION` and the database
service change.

---

## 6. Getting it running

```bash
# first time, builds the PHP image
docker compose up -d --build

# every time after that
docker compose up -d
docker compose ps
docker compose logs -f php
```

Before that first `up`, make sure `.env` exists. Compose reads
`${DB_DATABASE}`, `${DB_USERNAME}` and `${DB_PASSWORD}` straight out of it to
create the Postgres user, and if the file is missing the database initialises
with empty credentials. Worse, those credentials are only applied the first time
the data volume is created, so fixing `.env` afterwards does not help — you have
to `docker compose down -v` and start over, which takes the database with it.

Everything PHP-side runs inside the container:

```bash
docker compose exec php php artisan migrate
docker compose exec php php artisan test --compact
docker compose exec php composer install
docker compose exec php vendor/bin/pint --dirty
docker compose exec php php artisan tinker
```

`docker compose exec php` is worth preferring over `docker exec -it {project}-php`
— it addresses the service rather than the container name, so the same command
works in every project without substituting anything.

Running those from the host fails, and the error is worth recognising on sight:
`could not translate host name "pg_db"`. The host has no idea what `pg_db` is —
that name only exists on the Compose network. Same for `redis`.

The `.env` values that matter for this topology:

```dotenv
DB_CONNECTION=pgsql
DB_HOST=pg_db          # the service name, not 127.0.0.1
DB_PORT=5432
REDIS_HOST=redis
REDIS_SEARCH_HOST=redis
```

<!-- IMAGE PLACEHOLDER -->
![Terminal after a successful docker compose up](images/compose-up.png)
<!-- Suggested capture: the tail of `docker compose up -d --build`, with all
     five services created. -->

---

## 7. The applications it serves

One codebase, several front doors, all through the same two containers.

| Application | Where | What serves it |
| --- | --- | --- |
| App panel, for end users | `http://{project}.local/app` | nginx to php; `AppPanelProvider`, resources in `app/Filament/App/Resources` |
| Admin panel | `http://{project}.local/admin` | nginx to php; `AdminPanelProvider`, resources in `app/Filament/Resources` |
| API and mobile endpoints | `http://{project}.local/api/...` | nginx to php; Sanctum-authenticated routes in `routes/` |
| Adminer | `http://localhost:8080` | the adminer container; server `pg_db`, credentials from `.env` |
| Vite dev server | `http://localhost:5173` | the php container, started by hand |

nginx answers on whatever `server_name` is set in `.docker/nginx/default.conf`,
so you want a matching hosts entry:

```
127.0.0.1  {project}.local
```

Check the checked-in value before assuming — a copied `default.conf` often still
carries the *previous* project's hostname, which is harmless until it confuses
someone. `http://localhost` works too, since there is only one server block and
it ends up being the default, but any absolute URL Laravel generates follows
`APP_URL`. Pick one and keep the two in sync, or you get mixed-host redirects.


### Front-end assets

```bash
# one-off build
docker compose exec php npm run build

# watch mode
docker compose exec php npm run dev -- --host 0.0.0.0
```

That `--host 0.0.0.0` is not optional. Unless `vite.config.js` sets
`server.host`, Vite binds to localhost *inside* the container, and the published
port 5173 looks dead from your browser even though the process is clearly
running. Either pass the flag or add `server.host: '0.0.0.0'` to the config.

---

## 8. Things the code may expect that Compose does not start

**Kafka.** The image installs `rdkafka`, and a project with a `config/kafka.php`
will have topics defined, but there is usually no broker in Compose. Point
`KAFKA_BROKERS` at an external one, or add a service:

```yaml
  kafka:
    image: bitnami/kafka:latest
    container_name: {project}-kafka
    environment:
      KAFKA_CFG_NODE_ID: 0
      KAFKA_CFG_PROCESS_ROLES: controller,broker
      KAFKA_CFG_LISTENERS: PLAINTEXT://:9092,CONTROLLER://:9093
      KAFKA_CFG_CONTROLLER_QUORUM_VOTERS: 0@kafka:9093
      KAFKA_CFG_CONTROLLER_LISTENER_NAMES: CONTROLLER
    ports:
      - "9092:9092"
```

Then `KAFKA_BROKERS=kafka:9092`.

**Queue worker and scheduler.** No containers for these in the dev setup, so run
them by hand:

```bash
docker compose exec php php artisan queue:listen --tries=1 --timeout=0
docker compose exec php php artisan schedule:work
```

In production these are what `supervisord.conf` is for — one container running
fpm and the worker together, rather than remembering to start them.

**Mail.** Nothing catches outbound mail. Use `MAIL_MAILER=log`, or add a Mailpit
service if you need to look at rendered messages.

**Stale volumes.** Watch for volumes declared in `docker-compose.yml` that no
service mounts — `phpmyadmin_sessions` and `db_data` are common leftovers from a
MySQL-era version of the file. Safe to delete.

---

## 9. The other half: building for production

Everything above is a development setup, and it leans on things you do not want
in production: a bind-mounted working copy, build tools left in the image, a root
user, no compiled assets in the image at all.

The production pattern is a three-stage build. The idea is that the image you
ship is assembled by throwing most of the build away:

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

Stage one is `node:22-alpine`: `npm ci`, `npm run build`, done. Stage two is
`php:8.4-fpm-alpine` with all the `-dev` headers and compilers, where extensions
get built and `composer install --no-dev --optimize-autoloader` runs. Stage three
starts from a clean `php:8.4-fpm-alpine` and copies in only the compiled
extensions, the config, and the application — no compilers, no headers, no npm.

Four things that buys you: a much smaller image, no build toolchain in
production, assets built once and shared between the PHP and nginx images, and
layer caching that actually holds because the volatile steps come last.

The runtime stage also does two things that are easy to skip and awkward to
retrofit. It drops privileges before the entrypoint, and it strips setuid bits to
shrink the attack surface:

```dockerfile
RUN find /usr/local -type f -perm /6000 -exec chmod a-s {} + || true
USER www-data
```

The nginx side gets the same treatment as its own small image: `nginx:alpine`,
config copied in, and `public/` pulled from the node stage, so the web server
serves built assets without ever mounting your source tree.

### Overriding Compose for production

Keep the dev `docker-compose.yml` as the base and layer a production file on top,
rather than maintaining two full copies:

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

Two details worth copying while you are in there. Give Postgres a real
healthcheck instead of relying on `depends_on`, which only waits for the
container to start, not for the database to accept connections:

```yaml
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
```

And bound Redis memory, because an unbounded cache container will happily eat the
host:

```yaml
    command: >
      redis-server
      --appendonly yes
      --maxmemory 256mb
      --maxmemory-policy allkeys-lru
```

If `docker-compose.yml` declares no explicit network, everything lands on the
Compose default. That works, but naming one makes the topology obvious and stops
a stray container joining by accident.

### Pushing images

```bash
docker tag {project}-php {registry}/{project}-php:latest
docker push {registry}/{project}-php:latest
```

---

## 10. Pointing this at a different application

Nothing application-specific is baked into the dev image, so moving the setup to
another Laravel project is mostly renaming. This is not hypothetical — the same
recipe runs several of these projects side by side on one machine, each with its
own copy of `.docker/`.

Copy `.docker/` and `docker-compose.yml` across, then work through these five.

**Rename the containers.** This is the one that bites first. `container_name:`
values are fixed strings, so two projects both claiming `{project}-php` cannot
run at the same time — the second `up` fails with a name conflict. Three ways
out, best last:

```yaml
    container_name: newproject-php        # rename every prefix by hand
```
```yaml
    # drop container_name entirely; Compose names it <dir>-php-1
```
```yaml
    container_name: ${APP_SLUG}-php       # one .env value drives all five
```

The third is what makes the template genuinely reusable: set `APP_SLUG` in
`.env` and the compose file itself never needs editing again.

**Move the host ports.** 80, 5432, 6379, 8080 and 5173 are all
single-occupancy. Shift the host side for the second project — `8081:80`,
`5433:5432`, and so on. The container side stays put, so no application config
changes. Same trick applies here: `"${HTTP_PORT:-80}:80"` beats hardcoding.

**Change `server_name`** in `.docker/nginx/default.conf` to `{project}.local`,
and add the matching hosts entry. This is the step people forget, because the
site still loads on `http://localhost` and the stale hostname goes unnoticed.

**Swap the database if the app is MySQL.** `pdo_mysql` and the MySQL client are
already in the image, so it is just replacing the `pg_db` service with `mysql:8`
and setting `DB_CONNECTION=mysql`, `DB_HOST=mysql`.

**Drop what you do not need.** Redis Stack is only there for RediSearch — plain
`redis:alpine` is lighter if the app just caches. Adminer goes if you use a
desktop client.

For a non-Laravel PHP app the only extra change is the nginx `root`, since
`/var/www/public` assumes Laravel's layout.

<!-- IMAGE PLACEHOLDER -->
![Two projects running side by side on different host ports](images/multi-project-ports.png)
<!-- Suggested capture: `docker compose ps` for two projects at once, showing
     the remapped host ports. -->

### Where to add your own things

Additions have to land in the right stage. System packages needed only to
*compile* something go in the build stage; anything the running app needs at
runtime has to be repeated in the runtime stage, or the extension loads against a
library that is not there. That duplication looks redundant and is not.

Built-in PHP extensions go through `docker-php-ext-install`; PECL ones through
`pecl install X && docker-php-ext-enable X`. New services join Compose with a
service name, a container name, and the shared network.

---

## 11. When it breaks

| What you see | What it usually is | What to do |
| --- | --- | --- |
| `502 Bad Gateway` | php container down, or fpm not listening | `docker compose logs php`; check the pool is on `0.0.0.0:9000` |
| `could not translate host name "pg_db"` | Command run on the host, not in the container | Prefix it with `docker compose exec php` |
| Name conflict on `up` | Another project using the same `container_name` | See section 10 |
| Postgres rejects the password after an `.env` edit | Credentials only apply when the volume is first created | `docker compose down -v` and up again, which destroys the database |
| `413 Request Entity Too Large` | Upload over a limit | nginx allows 100M; PHP may still be on the stock 2M, see section 5 |
| `Unable to locate file in Vite manifest` | Assets never built | `docker compose exec php npm run build` |
| `localhost:5173` refuses to connect | Vite bound inside the container | Add `--host 0.0.0.0`, see section 7 |
| `storage` permission errors | `entrypoint.sh` not wired up | Wire it, or `docker compose exec php chown -R www-data:www-data storage bootstrap/cache` |
| Search returns nothing | Wrong Redis image, or index never built | Confirm `redis/redis-stack-server`, then rebuild the index |
| Config changes have no effect | The file is not mounted or copied | Check with `php -i`, see section 5 |

Resetting, in increasing order of regret:

```bash
docker compose down                    # stop, keep the data
docker compose down -v                 # stop and delete volumes, database included
docker compose build --no-cache php    # rebuild after editing Dockerfile.dev
```


---

## 12. Command reference

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
