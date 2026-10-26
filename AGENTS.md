# Base44 sandbox notes

## What this repo is

A single static page: `admin` (no file extension) — a barbershop ("Barbearia") admin
panel in Brazilian Portuguese, plain HTML with inline CSS and vanilla JS. No build step,
no dependencies, no package manager, no backend.

## How it runs here

`docker compose -f docker-compose.base44.yml up -d --build` starts one service, `web`:
`nginx:alpine` serving the bind-mounted repo at `/usr/share/nginx/html` on host port 3000
(the preview entry point).

The nginx config lives in `.base44/nginx/default.conf`. Two non-obvious bits in it:
`listen 3000` (the preview port, not nginx's default 80) and `default_type text/html`
(the page is called `admin`, with no `.html` extension, so nginx would otherwise send
`application/octet-stream` and the browser would download the file). `index admin` serves
it at `/`; `try_files $uri $uri/ /admin` serves it for any other path. Caching is off.

There is no watch/dev-server step: nginx reads the file from disk, so a source edit shows
on the next page load (use `reload_preview` to force it).

### Two quirks that will bite you if you touch the compose file

1. The sandbox checks this repo out with mode **0700**, and nginx's worker runs as the
   unprivileged `nginx` user, so it cannot traverse the mount — every request answers
   `403`/`500` with `stat() ... failed (13: Permission denied)` in the logs. The service
   therefore runs `chmod 755` on the mounted repo root before starting nginx, which is also
   why the source mount is not `:ro` (a read-only mount rejects the `chmod`). The chmod only
   adds traverse/read for others, never write, and re-runs on every container start.
2. `default_type text/html` is load-bearing: the page has no extension, so without it nginx
   serves `application/octet-stream` and the browser downloads the file instead of rendering it.

## Credentials / external services

None. Nothing is needed from `.env` or the secrets store, and there is nothing to ask the
user for:

- Login is client-side and hardcoded in `admin` — user `admin`, password `1234`, with the
  flag kept in `sessionStorage.adminLogado`.
- Bookings live in `localStorage.agendamentos` as `[{nome, data, hora, servico, status}]`
  with status `pendente | confirmado | recusado`; today's prices are the `precos` object in
  the same file. All data is per-browser — nothing is persisted server-side.

## Verifying it works

- `docker compose -f docker-compose.base44.yml ps` → `web` is `healthy`.
- `curl -s http://localhost:3000/ | grep -i "Login Admin"` matches. The served page must be
  this repo's `admin` file.
- The compose healthcheck is the same check (greps the served page for `Login Admin`).
- Login (`admin` / `1234`) and the Confirmar/Recusar actions were exercised end to end in the
  preview: confirming a `Corte` booking moved "Clientes hoje" 0 → 1 and "Faturamento hoje"
  to R$ 30, with no console errors.

## Gotcha

The client booking page that writes `localStorage.agendamentos` is **not** in this repo —
only `admin` is, so on its own the panel lists "Nenhum agendamento.".
