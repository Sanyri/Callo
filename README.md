# Callo

Callo — браузерный сервис 1:1 аудио- и видеозвонков без email, пароля и обязательной регистрации.

## Стек

- React + TypeScript + Vite
- Tailwind CSS + Lucide
- Cloudflare Workers + Workers Static Assets
- Cloudflare D1
- Durable Objects + WebSocket Hibernation
- WebRTC
- R2 для аватарок

## Как устроено

```text
Browser
  │
  ├── HTTPS ──> Worker API ──> D1
  │                 │
  │                 ├── UserSession DO ── presence / incoming calls
  │                 └── CallRoom DO ───── offer / answer / ICE
  │
  └──────────── WebRTC audio/video ────────────> другой Browser
```

Worker и Durable Objects не получают и не записывают медиапоток звонка. Они передают только signaling. TURN подключается отдельно через secrets, если нужен relay.

Cloudflare рекомендует Vite + Cloudflare Vite plugin для React-приложений на Workers и Workers Static Assets для раздачи клиентской сборки. Для WebSocket-серверов в Durable Objects рекомендуется Hibernation API. citeturn385552search0turn385552search1turn385552search6

## Самый простой способ: GitHub → Cloudflare

1. Распакуй проект и загрузи папку `callo` в GitHub repository.
2. В Cloudflare открой **Workers & Pages → Create → Import a repository** и выбери репозиторий.
3. Root directory: `/`.
4. Build command: `npm run build`.
5. Deploy command: `npm run deploy:cloudflare`.
6. Build variables не нужны.
7. Runtime secret `CALLO_SECRET` добавь отдельно в **Settings → Variables & Secrets → Secrets**.

Workers Builds запускает build command, затем deploy command; стандартный deploy — `npx wrangler deploy`. citeturn907784search2

## Первый запуск инфраструктуры

Обычный способ один раз создать D1/R2 и записать UUID D1 в `wrangler.jsonc`:

```bash
npm install
npx wrangler login
npm run cf:setup
npx wrangler secret put CALLO_SECRET
npm run deploy
```

`cf:setup`:

- создаёт или находит D1 `callo-db`;
- автоматически записывает `database_id` в `wrangler.jsonc`;
- пытается создать R2 `callo-profile-images`.

После этого `wrangler.jsonc` можно закоммитить в GitHub. Идентификатор D1 не является секретом. Секретом является только `CALLO_SECRET` и TURN credentials.

D1 migrations хранятся в `migrations/` и применяются Wrangler-командой `d1 migrations apply`. citeturn907784search0

## Что указывать в Cloudflare Workers Builds

```text
Root directory: /
Build command: npm run build
Deploy command: npm run deploy:cloudflare
```

Важный момент: `npm run deploy:cloudflare` применяет только неприменённые D1 migrations и затем запускает `wrangler deploy`.

Preview deployments можно не использовать для production-проверки звонков: Cloudflare отмечает особенности preview для Workers с Durable Objects. Для production используй обычный production deploy. citeturn907784search2

## Секреты

Обязательный:

```bash
npx wrangler secret put CALLO_SECRET
```

Опциональный TURN:

```bash
npx wrangler secret put TURN_URLS
npx wrangler secret put TURN_USERNAME
npx wrangler secret put TURN_CREDENTIAL
```

Пример:

```text
TURN_URLS=turn:turn.example.com:3478,turns:turn.example.com:5349
```

Не помещай эти значения во frontend, `.env`, `VITE_*` или клиентский JavaScript.

## Локальная разработка

```bash
npm install
cp .dev.vars.example .dev.vars
npm run dev
```

Проверка:

```bash
npm run check
npm test
npm run build
```

Preview Worker runtime:

```bash
npx wrangler dev
```

## Cloudflare resources

### D1

Имя:

```text
callo-db
```

Binding:

```text
DB
```

### Durable Objects

Bindings:

```text
CALL_ROOM   → CallRoom
USER_SESSION → UserSession
RATE_LIMITER → RateLimiter
```

### R2

Bucket:

```text
callo-profile-images
```

Binding:

```text
PROFILE_IMAGES
```

## Домен

После production deploy:

**Workers & Pages → callo → Settings → Domains & Routes → Add Custom Domain**.

Для браузерных камеры/микрофона production-домен должен работать по HTTPS; WebSocket signaling будет использовать WSS.

## WebRTC

По умолчанию Worker отдаёт STUN:

```text
stun:stun.cloudflare.com:3478
stun:stun.l.google.com:19302
```

Для сложных NAT/Firewall-сетей добавляй собственный TURN через secrets.

## Функциональность

- анонимный серверный bootstrap;
- уникальный `CALL-123-456`;
- локальная сессия в HttpOnly/Secure cookie;
- имя, описание, цвет аватарки, изображение;
- поиск пользователя по номеру;
- друзья;
- блокировка;
- минимальная история звонков;
- входящие вызовы через WebSocket;
- аудиозвонки;
- видеозвонки;
- mute/unmute;
- camera on/off;
- switch camera;
- screen share;
- громкость;
- статус подключения;
- сообщения о запрете камеры/микрофона и занятых устройствах;
- dark mode;
- recovery-link;
- public profile;
- rate limiting.

## Проверка сценариев

1. Открыть Callo в чистом браузере → должен создаться профиль и номер.
2. Указать имя.
3. Открыть настройки → изменить имя/цвет/описание.
4. Загрузить аватар → проверить R2.
5. На втором браузере получить другой `CALL-...` номер.
6. Найти второго пользователя.
7. Добавить в друзья.
8. Позвонить аудио.
9. Позвонить видео.
10. Проверить принятие/отклонение.
11. Проверить завершение звонка и history.
12. Заблокировать пользователя → новый звонок должен получить отказ.
13. Создать recovery-link → открыть его в другом браузере → получить прежний профиль.

## Ограничения браузера

Звуковое автопроигрывание входящего звонка может зависеть от autoplay-policy браузера. Сам входящий вызов приходит через WebSocket; пользовательское действие «Принять» всегда может инициировать воспроизведение удалённого audio.

Не очищай browser storage/cookies без recovery-link: профиль намеренно остаётся анонимным и без обязательного аккаунта.
