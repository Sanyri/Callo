# Callo: запуск на Cloudflare

Самый надёжный вариант — один раз создать инфраструктуру через Wrangler, затем подключить GitHub к Workers Builds.

## 1. Один раз на своём компьютере

Требуется Node.js 20+.

```bash
cd callo
npm install
npx wrangler login
npm run cf:setup
npx wrangler secret put CALLO_SECRET
npm run deploy
```

`cf:setup` создаст или найдёт D1 `callo-db`, запишет его UUID в `wrangler.jsonc` и создаст R2 bucket `callo-profile-images`.

Для D1 скрипт использует европейскую локацию/юрисдикцию (`eeur`/`eu`). При необходимости это можно изменить в `scripts/cloudflare-setup.mjs` до создания базы.

## 2. GitHub

Закоммить всю папку `callo`, включая уже заполненный `wrangler.jsonc`.

Не коммить:

```text
.dev.vars
.env
.wrangler/
node_modules/
```

## 3. Cloudflare Workers Builds

Cloudflare Dashboard → Workers & Pages → Create/Import repository.

Параметры:

```text
Root directory: /
Build command: npm run build
Deploy command: npm run deploy:cloudflare
```

`npm install` Cloudflare выполняет как часть установки зависимостей сборки.

## 4. Secrets

В Cloudflare Dashboard открой Worker → Settings → Variables & Secrets.

Добавь secret:

```text
CALLO_SECRET
```

Значение должно быть длинным случайным секретом.

TURN, если используется:

```text
TURN_URLS
TURN_USERNAME
TURN_CREDENTIAL
```

Опционально:

```text
APP_ORIGIN=https://callo.example.com
```

## 5. Что получится

После deployment:

```text
https://callo.<subdomain>.workers.dev
```

Можно добавить custom domain.

## 6. Последующие обновления

После изменения исходников:

```bash
git add .
git commit -m "Update Callo"
git push
```

Cloudflare Workers Builds повторит build/deploy автоматически.

## 7. D1 migrations

Новая миграция добавляется в `migrations/`, например:

```text
migrations/0002_add_something.sql
```

Production deploy command уже делает:

```bash
wrangler d1 migrations apply callo-db --remote
wrangler deploy
```

Поэтому применяются только ещё не применённые миграции.
