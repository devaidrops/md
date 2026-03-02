## План перенастройки coinexplorers.tw1.su → coinexplorers.com

**Цель**: по этому документу можно по шагам, копируя команды, перенастроить сервер с тестового домена `coinexplorers.tw1.su` на продакшен `coinexplorers.com`.

Ниже предполагается, что:

- проект развернут в директории `/srv/coinexplorers.com`;
- фронтенд (Next.js) слушает `127.0.0.1:3000`;
- Strapi слушает `127.0.0.1:1337`;
- API (отдельный сервис) слушает `127.0.0.1:8000`;
- используется `pm2` с конфигом `ecosystem.config.js`;
- в nginx уже есть рабочий конфиг для `coinexplorers.tw1.su`.

Если что‑то отличается — скорректируйте команды под свои пути/порты.

---

## 1. Предварительная проверка и бэкапы

**1.1. Проверить, что вы на нужном сервере**

```bash
hostname
ls -la /srv/coinexplorers.com
```

Убедитесь, что вывод соответствует прод-серверу и проекту.

**1.2. Проверить состояние nginx и pm2**

```bash
sudo systemctl status nginx
pm2 status
```

**1.3. Сделать бэкап ключевых конфигов**

```bash
mkdir -p ~/backup-coinexplorers

sudo cp /etc/nginx/sites-available/coinexplorers.tw1.su.conf \
        ~/backup-coinexplorers/coinexplorers.tw1.su.conf.$(date +%F-%H%M%S)

sudo cp /etc/nginx/sites-available/coinexplorers.twc1.net.conf \
        ~/backup-coinexplorers/coinexplorers.twc1.net.conf.$(date +%F-%H%M%S)

cp /srv/coinexplorers.com/client/.env \
   ~/backup-coinexplorers/client.env.$(date +%F-%H%M%S)

cp /srv/coinexplorers.com/ecosystem.config.js \
   ~/backup-coinexplorers/ecosystem.config.js.$(date +%F-%H%M%S)

cp /srv/coinexplorers.com/server/config/server.js \
   ~/backup-coinexplorers/strapi.server.js.$(date +%F-%H%M%S)
```

---

## 2. Выпуск SSL‑сертификатов для coinexplorers.com

Если сертификаты Let's Encrypt для `coinexplorers.com` ещё не выпущены, выполните:

**2.1. Остановить nginx (если нужен standalone/temporay webroot вариант)**

Обычно достаточно webroot, так как в конфиге уже есть location для `/.well-known/acme-challenge/`. Ниже пример для нового домена с тем же webroot.

**2.2. Выдать сертификат через certbot (webroot)**

```bash
sudo certbot certonly --webroot \
  -w /var/www/_acme-challenge/coinexplorers.tw1.su \
  -d coinexplorers.com \
  -d www.coinexplorers.com
```

> Если webroot для нового домена будет другим, укажите корректный путь.

Проверьте, что сертификаты появились:

```bash
sudo ls -la /etc/letsencrypt/live/coinexplorers.com
```

Должны быть файлы `fullchain.pem` и `privkey.pem`.

---

## 3. Настройка nginx для coinexplorers.com

Сейчас есть рабочий конфиг `/etc/nginx/sites-available/coinexplorers.tw1.su.conf`, который:

- перенаправляет HTTP → HTTPS;
- принудительно убирает `www`;
- проксирует:
  - `/` и `/news` → фронтенд `127.0.0.1:3000` (c basic auth);
  - `/strapi/` → Strapi `127.0.0.1:1337`;
  - `/api/` → API `127.0.0.1:8000`.

Сделаем новый конфиг для `coinexplorers.com`, максимально повторяя логику.

**3.1. Создать конфиг coinexplorers.com на основе tw1.su**

```bash
sudo cp /etc/nginx/sites-available/coinexplorers.tw1.su.conf \
        /etc/nginx/sites-available/coinexplorers.com.conf
```

**3.2. Отредактировать новый конфиг**

Откройте файл:

```bash
sudo nano /etc/nginx/sites-available/coinexplorers.com.conf
```

Замените в нём все вхождения `coinexplorers.tw1.su` на `coinexplorers.com`:

- `server_name coinexplorers.com www.coinexplorers.com;`
- `return 301 https://coinexplorers.com$request_uri;`
- `ssl_certificate     /etc/letsencrypt/live/coinexplorers.com/fullchain.pem;`
- `ssl_certificate_key /etc/letsencrypt/live/coinexplorers.com/privkey.pem;`
- `root /var/www/_acme-challenge/coinexplorers.com;` (если решите сделать отдельный webroot; можно оставить старый, но лучше разнести по доменам).

Примерно структура файла должна остаться такой же, как в `coinexplorers.tw1.su.conf`:

- первый `server {}` — HTTP → HTTPS redirect;
- второй `server {}` — HTTPS `www` → non‑`www`;
- третий `server {}` — основной HTTPS‑виртуалхост с proxy на 3000/1337/8000.

**3.3. Включить сайт и проверить конфиг**

Создать symlink в `sites-enabled` (если используется схема enabled/available):

```bash
sudo ln -s /etc/nginx/sites-available/coinexplorers.com.conf \
           /etc/nginx/sites-enabled/coinexplorers.com.conf
```

Проверить конфигурацию nginx:

```bash
sudo nginx -t
```

Если ошибок нет — перезагрузить nginx:

```bash
sudo systemctl reload nginx
```

После этого nginx должен обслуживать **оба** домена: `coinexplorers.tw1.su` и `coinexplorers.com`.

---

## 4. Обновление фронтенд `.env` под coinexplorers.com

Текущий файл `/srv/coinexplorers.com/client/.env` содержит:

- `NEXT_PUBLIC_API_ENDPOINT=https://coinexplorers.tw1.su/strapi`
- `NEXT_PUBLIC_APP_URL=https://coinexplorers.tw1.su`
- `NEXT_PUBLIC_APP_HOST=https://coinexplorers.tw1.su`
- `NEXT_PUBLIC_STRAPI_URL=https://coinexplorers.tw1.su`

**4.1. Открыть и отредактировать `.env`**

```bash
cd /srv/coinexplorers.com/client
nano .env
```

Замените `coinexplorers.tw1.su` на `coinexplorers.com`:

```bash
NEXT_PUBLIC_API_ENDPOINT=https://coinexplorers.com/strapi
NEXT_PUBLIC_APP_URL=https://coinexplorers.com
NEXT_PUBLIC_APP_HOST=https://coinexplorers.com
NEXT_PUBLIC_STRAPI_URL=https://coinexplorers.com
NEXT_PUBLIC_POSTS_PRIORITY=0.6
NEXT_PUBLIC_REVIEWS_PRIORITY=0.8
```

Сохраните файл.

---

## 5. Обновление Strapi-конфига `server.js`

Файл: `/srv/coinexplorers.com/server/config/server.js`

Сейчас там в проде выставлен URL по умолчанию на тестовый домен:

```js
url: env("PUBLIC_URL", "https://coinexplorers.tw1.su"),
```

**5.1. Отредактировать конфиг**

```bash
cd /srv/coinexplorers.com/server
nano config/server.js
```

Замените тестовый домен на прод:

```js
url: env("PUBLIC_URL", "https://coinexplorers.com"),
```

Остальные настройки (порт 1337, admin по `http://127.0.0.1:1337/admin`) менять не нужно.

---

## 6. Обновление `ecosystem.config.js` (pm2)

Файл: `/srv/coinexplorers.com/ecosystem.config.js`

Сейчас:

```js
env: {
  HOST: "127.0.0.1",
  PORT: "1337",
  NODE_ENV: "production",
  PUBLIC_CLIENT_URL: "https://coinexplorers.tw1.su"
}
```

**6.1. Отредактировать PUBLIC_CLIENT_URL**

```bash
cd /srv/coinexplorers.com
nano ecosystem.config.js
```

Замените:

```js
PUBLIC_CLIENT_URL: "https://coinexplorers.com"
```

Остальные поля (`HOST`, `PORT`, `NODE_ENV`) оставьте как есть.

---

## 7. Перезапуск процессов pm2 с новыми настройками

После правок `.env`, `server.js` и `ecosystem.config.js` нужно перезапустить процессы под управлением `pm2`.

**7.1. Загрузить обновлённый конфиг в pm2**

```bash
cd /srv/coinexplorers.com
pm2 startOrReload ecosystem.config.js
```

**7.2. Проверить статус процессов**

```bash
pm2 status
pm2 logs --lines 50
```

Убедитесь, что процессы `strapi`, `api`, `frontend` запущены без ошибок.

---

## 8. Проверка работы сайта по новому домену

**8.1. Локальная проверка c сервера**

```bash
curl -I https://coinexplorers.com
curl -I https://coinexplorers.com/api/health || curl -I https://coinexplorers.com/api
curl -I https://coinexplorers.com/strapi
```

Должны приходить коды `200`/`301` (в зависимости от маршрута), без 4xx/5xx.

**8.2. Проверка из браузера**

Откройте в браузере:

- `https://coinexplorers.com`
- `https://www.coinexplorers.com` (должно редиректить на без `www`)
- разделы сайта, критичные для SEO (категории, страницы, поиск).

---

## 9. Плавное отключение тестового домена (опционально)

После того как убедились, что `coinexplorers.com` работает стабильно и DNS полностью перенесён:

### Вариант A: Оставить tw1.su с редиректом на prod

Оставить текущий nginx-конфиг `coinexplorers.tw1.su.conf` как есть — трафик по старому домену будет продолжать работать (например, для внутренних тестов).

### Вариант B: Делать 301‑редирект на coinexplorers.com

Можно изменить конфиг `coinexplorers.tw1.su.conf`, чтобы он всегда делал 301‑редирект на новый домен:

- в HTTP‑сервере:

```nginx
return 301 https://coinexplorers.com$request_uri;
```

- в HTTPS‑сервере:

```nginx
return 301 https://coinexplorers.com$request_uri;
```

После изменений:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### Вариант C: Полностью отключить tw1.su

Когда тестовый домен совсем не нужен:

```bash
sudo rm /etc/nginx/sites-enabled/coinexplorers.tw1.su.conf
sudo nginx -t
sudo systemctl reload nginx
```

> Конфиг в `sites-available` и сертификаты можно удалить позже, когда будете уверены, что откат не потребуется.

---

## 10. Чек-лист перед завершением работ

- **DNS**: A/AAAA‑записи доменов `coinexplorers.com` и `www.coinexplorers.com` указывают на этот сервер;
- **Сертификаты**: в `/etc/letsencrypt/live/coinexplorers.com` есть актуальные `fullchain.pem` и `privkey.pem`;
- **nginx**:
  - есть рабочий конфиг `/etc/nginx/sites-available/coinexplorers.com.conf`;
  - есть symlink в `sites-enabled`;
  - `nginx -t` проходит без ошибок;
  - редиректы HTTP→HTTPS и www→non-www работают корректно;
- **Backend**:
  - `ecosystem.config.js` содержит `PUBLIC_CLIENT_URL=https://coinexplorers.com`;
  - `server/config/server.js` использует прод‑домен;
  - процессы `strapi`, `api`, `frontend` в `pm2 status` **online**;
- **Frontend**:
  - `.env` в `client` ссылается на `https://coinexplorers.com` и `/strapi`;
  - критичные страницы и SEO‑маршруты (`/`, категории, поиск, sitemap) открываются и отдают корректные коды;
- **Старый домен**:
  - либо корректно редиректит на новый домен (301);
  - либо отключён согласно вашей политике.

Если все пункты выполнены — перенастройка с `coinexplorers.tw1.su` на `coinexplorers.com` завершена.

